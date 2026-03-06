# API Conventions

## Overview

This document defines REST API design standards for all services in the rkamradt-platform. Consistent API design improves developer experience and reduces integration complexity.

## API Design Principles

### 1. RESTful Resource Design

- APIs are resource-oriented (nouns, not verbs)
- Use HTTP methods semantically: GET, POST, PUT, PATCH, DELETE
- Resources are identified by URIs
- Stateless communication

### 2. API-First Development

- Define OpenAPI/Swagger spec before implementation
- Use spec for contract testing
- Generate client SDKs from spec
- Keep spec in sync with implementation

### 3. Backward Compatibility

- Never break existing API contracts
- Version APIs when breaking changes are unavoidable
- Deprecate gracefully with advance notice
- Support deprecated versions for reasonable period

## URL Structure

### Base Path

```
{protocol}://{host}:{port}/api/{resource}
```

**Examples**:
- `http://localhost:8080/api/bom`
- `http://localhost:8080/api/orders`
- `http://localhost:8080/api/work-schedules`

### Resource Naming

- Use **plural nouns** for collections: `/api/boms`, `/api/orders`
- Use **lowercase** with hyphens for multi-word resources: `/api/work-schedules`
- Avoid verbs in resource names (use HTTP methods instead)

**Good**:
```
GET    /api/boms          # List BOMs
POST   /api/boms          # Create BOM
GET    /api/boms/{id}     # Get BOM by ID
PUT    /api/boms/{id}     # Update BOM
DELETE /api/boms/{id}     # Delete BOM
```

**Avoid**:
```
GET    /api/getBoms       # Verb in URL
POST   /api/createBom     # Verb in URL
GET    /api/bom           # Singular (unless truly singleton resource)
```

### Nested Resources

Use nesting to show relationships, but limit to 2 levels:

**Good**:
```
GET /api/boms/{bomId}/parts              # Parts of a BOM
POST /api/boms/{bomId}/parts             # Add part to BOM
```

**Avoid** (too deep):
```
GET /api/boms/{bomId}/parts/{partId}/suppliers/{supplierId}/addresses
```

### Query Parameters

Use for filtering, sorting, pagination, and field selection:

```
GET /api/boms?status=APPROVED&sort=createdAt:desc&page=1&size=20
```

## HTTP Methods

| Method | Purpose | Idempotent | Safe | Request Body | Response Body |
|--------|---------|------------|------|--------------|---------------|
| GET | Retrieve resource(s) | Yes | Yes | No | Yes |
| POST | Create resource | No | No | Yes | Yes |
| PUT | Replace resource | Yes | No | Yes | Yes |
| PATCH | Partial update | No | No | Yes | Yes |
| DELETE | Remove resource | Yes | No | No | Optional |

### GET - Retrieve Resources

**Collection**:
```http
GET /api/boms
Response: 200 OK
[
  {"id": "123", "productId": "PROD-001", ...},
  {"id": "456", "productId": "PROD-002", ...}
]
```

**Single Resource**:
```http
GET /api/boms/123
Response: 200 OK
{"id": "123", "productId": "PROD-001", ...}
```

**Not Found**:
```http
GET /api/boms/999
Response: 404 Not Found
{"error": "BOM not found with id: 999"}
```

### POST - Create Resource

```http
POST /api/boms
Content-Type: application/json

{
  "productId": "PROD-001",
  "productName": "Widget",
  "parts": [...]
}

Response: 201 Created
Location: /api/boms/123
{
  "id": "123",
  "productId": "PROD-001",
  ...
}
```

### PUT - Replace Resource

```http
PUT /api/boms/123
Content-Type: application/json

{
  "productId": "PROD-001",
  "productName": "Widget Deluxe",
  "parts": [...]
}

Response: 200 OK
{
  "id": "123",
  "productId": "PROD-001",
  "productName": "Widget Deluxe",
  ...
}
```

### PATCH - Partial Update

```http
PATCH /api/boms/123
Content-Type: application/json

{
  "status": "APPROVED"
}

Response: 200 OK
{
  "id": "123",
  "status": "APPROVED",
  ...
}
```

### DELETE - Remove Resource

```http
DELETE /api/boms/123

Response: 204 No Content
```

Or with response body:
```http
DELETE /api/boms/123

Response: 200 OK
{
  "message": "BOM deleted successfully",
  "id": "123"
}
```

## Status Codes

Use appropriate HTTP status codes:

### Success (2xx)

| Code | Meaning | Use Case |
|------|---------|----------|
| 200 | OK | Successful GET, PUT, PATCH, or DELETE with body |
| 201 | Created | Successful POST creating new resource |
| 204 | No Content | Successful DELETE or update with no response body |

### Client Errors (4xx)

| Code | Meaning | Use Case |
|------|---------|----------|
| 400 | Bad Request | Invalid request payload or parameters |
| 401 | Unauthorized | Authentication required or failed |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Request conflicts with current state (e.g., duplicate) |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |

### Server Errors (5xx)

| Code | Meaning | Use Case |
|------|---------|----------|
| 500 | Internal Server Error | Unexpected server error |
| 503 | Service Unavailable | Service temporarily down or overloaded |

## Request/Response Format

### Content Type

- **Default**: `application/json`
- Support JSON for all APIs
- Consider `application/xml` only if required by clients

### JSON Naming Convention

Use **camelCase** for JSON fields:

```json
{
  "bomId": "123",
  "productId": "PROD-001",
  "estimatedCost": 100.50,
  "createdAt": "2026-03-05T14:30:00Z"
}
```

### Date/Time Format

Use **ISO-8601** format in UTC:

```json
{
  "createdAt": "2026-03-05T14:30:00Z",
  "scheduledDate": "2026-03-15"
}
```

### Enums

Use **UPPER_CASE** for enum values:

```json
{
  "status": "APPROVED",
  "priority": "HIGH"
}
```

## Error Responses

### Standard Error Format

```json
{
  "error": "Resource not found",
  "message": "BOM not found with id: 999",
  "status": 404,
  "timestamp": "2026-03-05T14:30:00Z",
  "path": "/api/boms/999"
}
```

### Validation Errors

```json
{
  "error": "Validation failed",
  "status": 422,
  "timestamp": "2026-03-05T14:30:00Z",
  "path": "/api/boms",
  "validationErrors": [
    {
      "field": "productId",
      "message": "productId is required"
    },
    {
      "field": "parts",
      "message": "parts list must not be empty"
    }
  ]
}
```

### Implementation

```java
public record ErrorResponse(
    String error,
    String message,
    int status,
    Instant timestamp,
    String path
) {}

public record ValidationErrorResponse(
    String error,
    int status,
    Instant timestamp,
    String path,
    List<FieldError> validationErrors
) {}

public record FieldError(
    String field,
    String message
) {}
```

## Pagination

### Query Parameters

```
GET /api/boms?page=1&size=20
```

- `page`: Page number (1-indexed)
- `size`: Items per page (default: 20, max: 100)

### Response Format

```json
{
  "content": [
    {"id": "123", ...},
    {"id": "456", ...}
  ],
  "page": {
    "number": 1,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  }
}
```

### Spring Data Page

```java
@GetMapping
public Page<BomResponse> findAll(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) {
    Pageable pageable = PageRequest.of(page, size);
    return bomService.findAll(pageable);
}
```

## Filtering

### Query Parameter Filtering

```
GET /api/boms?status=APPROVED&productId=PROD-001
```

### Advanced Filtering

For complex queries, consider query DSL or specification pattern:

```
GET /api/boms?filter=status:APPROVED AND estimatedCost:>100
```

## Sorting

```
GET /api/boms?sort=createdAt:desc
GET /api/boms?sort=productId:asc,createdAt:desc
```

**Spring Implementation**:
```java
@GetMapping
public Page<BomResponse> findAll(
    @PageableDefault(sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable
) {
    return bomService.findAll(pageable);
}
```

## Versioning

### URI Versioning (Recommended)

```
GET /api/v1/boms
GET /api/v2/boms
```

- Clear and explicit
- Easy to route in API gateway
- Simple for clients

### Header Versioning (Alternative)

```http
GET /api/boms
Accept: application/vnd.kamradtfamily.v1+json
```

### When to Version

- Breaking changes to request/response schema
- Removed fields
- Changed field semantics
- Changed business logic significantly

### Deprecation

Announce deprecation with headers:

```http
GET /api/v1/boms
Response: 200 OK
Deprecation: true
Sunset: Wed, 11 Nov 2026 11:11:11 GMT
Link: </api/v2/boms>; rel="successor-version"
```

## Authentication & Authorization

### JWT Bearer Token

```http
GET /api/boms
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### API Keys (Service-to-Service)

```http
GET /api/boms
X-API-Key: your-api-key-here
```

### Security Headers

Always include:

```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

## Rate Limiting

### Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1617123456
```

### Rate Limit Exceeded

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600

{
  "error": "Rate limit exceeded",
  "message": "Too many requests. Try again in 3600 seconds."
}
```

## HATEOAS (Optional)

For advanced API maturity, include links to related resources:

```json
{
  "id": "123",
  "productId": "PROD-001",
  "_links": {
    "self": {"href": "/api/boms/123"},
    "parts": {"href": "/api/boms/123/parts"},
    "orders": {"href": "/api/boms/123/orders"}
  }
}
```

## OpenAPI Documentation

### Annotations

```java
@RestController
@RequestMapping("/api/boms")
@Tag(name = "BOM", description = "Bill of Materials API")
public class BomController {

    @Operation(summary = "Create a new BOM")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "BOM created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid request"),
        @ApiResponse(responseCode = "422", description = "Validation failed")
    })
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public BomResponse create(
        @Valid @RequestBody
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "BOM creation request"
        )
        CreateBomRequest request
    ) {
        return bomService.createBom(request);
    }
}
```

### Swagger UI

Expose at `/swagger-ui.html` or `/api-docs`

## Testing

### Integration Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
class BomControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldCreateBom() throws Exception {
        CreateBomRequest request = new CreateBomRequest(...);

        mockMvc.perform(post("/api/boms")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(header().exists("Location"));
    }

    @Test
    void shouldReturn404WhenBomNotFound() throws Exception {
        mockMvc.perform(get("/api/boms/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("Resource not found"));
    }
}
```

### Contract Testing

Use Spring Cloud Contract or Pact for consumer-driven contract testing.

## Best Practices

### DO

✅ Use plural nouns for collections
✅ Use HTTP methods semantically
✅ Return appropriate status codes
✅ Use ISO-8601 for dates
✅ Support pagination for collections
✅ Include validation errors in response
✅ Version APIs for breaking changes
✅ Document with OpenAPI/Swagger
✅ Use camelCase for JSON fields
✅ Validate input thoroughly

### DON'T

❌ Use verbs in resource URLs
❌ Expose internal implementation details
❌ Return 200 for errors
❌ Skip input validation
❌ Break backward compatibility without versioning
❌ Return unbounded collections
❌ Include sensitive data in responses (passwords, tokens)
❌ Use GET requests to modify state
❌ Ignore security headers

## Performance

### Caching

Use HTTP caching headers:

```http
GET /api/boms/123
Response: 200 OK
Cache-Control: max-age=3600
ETag: "abc123"
```

Conditional requests:
```http
GET /api/boms/123
If-None-Match: "abc123"
Response: 304 Not Modified
```

### Compression

Enable GZIP compression for responses > 1KB

### Field Selection

Allow clients to request specific fields:

```
GET /api/boms?fields=id,productId,status
```

## References

- [Java Conventions](java-conventions.md) - Java coding standards
- [Kafka Conventions](kafka-conventions.md) - Event-driven patterns
- [Service Map](../architecture/service-map.md) - Service endpoints
