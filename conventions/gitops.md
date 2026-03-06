# GitOps Conventions

## Overview

This document defines CI/CD, deployment, and infrastructure-as-code conventions for the rkamradt-platform using GitOps principles.

## GitOps Principles

1. **Declarative**: System state described declaratively (YAML manifests)
2. **Versioned**: All configuration in Git as single source of truth
3. **Automated**: Changes automatically applied from Git
4. **Continuously Reconciled**: System state continuously synced with Git

## Repository Structure

### Service Repositories (Polyrepo)

Each service repository contains:

```
service-repo/
├── src/                          # Source code
├── pom.xml or build.gradle       # Build configuration
├── Dockerfile                    # Container image definition
├── .github/
│   └── workflows/
│       ├── build.yml             # Build and test
│       ├── release.yml           # Build and publish container
│       └── deploy.yml            # Update deployment manifests
└── k8s/                          # Kubernetes manifests
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── dev/
        │   ├── kustomization.yaml
        │   └── patches/
        ├── staging/
        │   └── kustomization.yaml
        └── prod/
            └── kustomization.yaml
```

### GitOps Repository (Optional)

For centralized manifest management:

```
gitops-repo/
├── applications/
│   ├── manufacturing-service/
│   │   ├── base/
│   │   └── overlays/
│   └── vehicleevent-api/
│       ├── base/
│       └── overlays/
└── infrastructure/
    ├── kafka/
    ├── postgres/
    └── monitoring/
```

## GitHub Actions Workflows

### Build and Test Pipeline

**Trigger**: Pull request, push to main

```yaml
# .github/workflows/build.yml
name: Build and Test

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
          cache: 'maven'

      - name: Build with Maven
        run: mvn clean verify

      - name: Run tests
        run: mvn test

      - name: Generate coverage report
        run: mvn jacoco:report

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./target/site/jacoco/jacoco.xml
```

### Release Pipeline

**Trigger**: Push to main (after successful build)

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
          cache: 'maven'

      - name: Build application
        run: mvn clean package -DskipTests

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      - name: Update deployment manifest
        run: |
          cd k8s/overlays/dev
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "chore: update image to ${{ steps.meta.outputs.version }}"
          git push
```

### Deployment Pipeline (ArgoCD Sync)

ArgoCD watches Git repository and automatically syncs changes to Kubernetes.

```yaml
# .github/workflows/deploy.yml (optional - for manual deployment)
name: Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Install ArgoCD CLI
        run: |
          curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          chmod +x /usr/local/bin/argocd

      - name: Login to ArgoCD
        run: argocd login ${{ secrets.ARGOCD_SERVER }} --username admin --password ${{ secrets.ARGOCD_PASSWORD }}

      - name: Sync application
        run: argocd app sync manufacturing-service-${{ github.event.inputs.environment }}

      - name: Wait for sync
        run: argocd app wait manufacturing-service-${{ github.event.inputs.environment }} --sync
```

## Kubernetes Manifests

### Deployment

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: manufacturing-service
  labels:
    app: manufacturing-service
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: manufacturing-service
  template:
    metadata:
      labels:
        app: manufacturing-service
        version: v1
    spec:
      containers:
        - name: manufacturing-service
          image: ghcr.io/rkamradt/manufacturing-service:main
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "kafka:9092"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
```

### Service

```yaml
# k8s/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: manufacturing-service
  labels:
    app: manufacturing-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: manufacturing-service
```

### ConfigMap

```yaml
# k8s/base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: manufacturing-service-config
data:
  application.yml: |
    spring:
      application:
        name: manufacturing-service
      kafka:
        bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
    server:
      port: 8080
```

### Kustomization

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: manufacturing-service
  managed-by: kustomize
```

### Environment Overlays

**Development**:
```yaml
# k8s/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: dev

replicas:
  - name: manufacturing-service
    count: 1

images:
  - name: ghcr.io/rkamradt/manufacturing-service
    newTag: main-abc123
```

**Production**:
```yaml
# k8s/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: prod

replicas:
  - name: manufacturing-service
    count: 3

images:
  - name: ghcr.io/rkamradt/manufacturing-service
    newTag: v1.2.3

patches:
  - path: resource-limits.yaml
```

## ArgoCD Application

```yaml
# argocd/manufacturing-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: manufacturing-service-prod
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/rkamradt/manufacturing-service.git
    targetRevision: main
    path: k8s/overlays/prod

  destination:
    server: https://kubernetes.default.svc
    namespace: prod

  syncPolicy:
    automated:
      prune: true       # Remove resources deleted from Git
      selfHeal: true    # Sync when cluster state differs from Git
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## Docker Best Practices

### Multi-Stage Build

```dockerfile
# Dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### .dockerignore

```
# .dockerignore
target/
.git/
.github/
*.md
.gitignore
.mvn/
*.iml
.idea/
```

## GitHub Packages

### Publishing

Images automatically published to `ghcr.io/{owner}/{repo}:{tag}`:

```
ghcr.io/rkamradt/manufacturing-service:main
ghcr.io/rkamradt/manufacturing-service:main-abc123
ghcr.io/rkamradt/manufacturing-service:v1.2.3
```

### Pulling Images

```bash
# Authenticate
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Pull image
docker pull ghcr.io/rkamradt/manufacturing-service:main
```

## Secrets Management

### GitHub Secrets

Store in repository settings → Secrets and variables → Actions:

- `ARGOCD_SERVER`
- `ARGOCD_PASSWORD`
- `DB_PASSWORD` (for testing)

### Kubernetes Secrets

```yaml
# k8s/base/secret.yaml (stored separately, not in Git!)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
stringData:
  username: postgres
  password: super-secret-password
```

### Sealed Secrets (Recommended)

Encrypt secrets before committing to Git:

```bash
# Install kubeseal
brew install kubeseal

# Encrypt secret
kubectl create secret generic postgres-credentials \
  --from-literal=username=postgres \
  --from-literal=password=super-secret \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to Git
```

## Environment Strategy

### Environments

| Environment | Branch | Auto-Deploy | Approval Required |
|-------------|--------|-------------|-------------------|
| Development | main | Yes | No |
| Staging | release/* | Yes | No |
| Production | v* tags | No | Yes (manual) |

### Branch Protection

**Main branch**:
- Require pull request reviews
- Require status checks to pass
- Require linear history
- No force pushes

## Monitoring Deployment

### ArgoCD Dashboard

Monitor application sync status:

```bash
argocd app list
argocd app get manufacturing-service-prod
argocd app sync manufacturing-service-prod --prune
```

### Rollback

```bash
# Rollback to previous version
argocd app rollback manufacturing-service-prod

# Rollback to specific version
argocd app rollback manufacturing-service-prod 5
```

## Database Migrations

### Flyway Integration

```yaml
# src/main/resources/db/migration/V1__initial_schema.sql
CREATE TABLE bom (
    id UUID PRIMARY KEY,
    product_id VARCHAR(255) NOT NULL,
    version VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

### Kubernetes Job for Migrations

```yaml
# k8s/base/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: manufacturing-service-migration
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: ghcr.io/rkamradt/manufacturing-service:main
          command: ["java", "-jar", "app.jar", "--spring.flyway.enabled=true"]
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
      restartPolicy: OnFailure
```

## Best Practices

### DO

✅ Keep manifests in Git
✅ Use Kustomize for environment-specific config
✅ Automate build, test, deploy pipeline
✅ Use multi-stage Docker builds
✅ Tag images with Git SHA
✅ Implement health checks (liveness/readiness)
✅ Set resource limits
✅ Use secrets for sensitive data
✅ Enable auto-sync in ArgoCD
✅ Monitor deployment status

### DON'T

❌ Commit secrets to Git (use Sealed Secrets)
❌ Use `latest` tag in production
❌ Skip tests in CI pipeline
❌ Deploy directly to production
❌ Run containers as root
❌ Ignore resource limits
❌ Deploy without health checks
❌ Skip database migration testing

## Troubleshooting

### ArgoCD Out of Sync

```bash
# Check difference
argocd app diff manufacturing-service-prod

# Force sync
argocd app sync manufacturing-service-prod --force

# Refresh cache
argocd app refresh manufacturing-service-prod
```

### Failed Deployment

```bash
# Check pod status
kubectl get pods -n prod

# View logs
kubectl logs -n prod deployment/manufacturing-service --tail=100

# Describe pod for events
kubectl describe pod -n prod <pod-name>

# Rollback
argocd app rollback manufacturing-service-prod
```

## References

- [Java Conventions](java-conventions.md) - Build and packaging standards
- [Architecture Overview](../architecture/overview.md) - Infrastructure architecture
- [Service Map](../architecture/service-map.md) - Service dependencies
