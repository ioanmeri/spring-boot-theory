Absolutely. These are exactly the questions you should be able to answer before moving to JPA. I'll give you the **interview answer first**, then the explanation where needed.

---

# Basic

## 1. What does `@RestController` do?

### Interview answer

> `@RestController` marks a class as a controller whose methods handle HTTP requests and whose return values are written directly to the HTTP response body, typically serialized as JSON. It is effectively `@Controller` + `@ResponseBody`.

Example:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable Long id) {
        return new UserDto(1L, "John");
    }
}
```

The returned `UserDto` is serialized to JSON:

```json
{
  "id": 1,
  "name": "John"
}
```

### Important distinction

`@RestController` does **not** mean "this is a REST API server" by itself.

It tells Spring that this class contains request-handling methods whose results should normally become the response body.

---

# 2. What's the difference between `@Controller` and `@RestController`?

### Interview answer

> `@Controller` is traditionally used for MVC controllers and can return view names. `@RestController` is intended for REST APIs and combines `@Controller` with `@ResponseBody`, so return values are written directly to the HTTP response body.

For example:

### `@Controller`

```java
@Controller
public class UserController {

    @GetMapping("/users")
    public String users() {
        return "users";
    }
}
```

Here `"users"` can be interpreted as a **view name**.

Think:

```text
Controller
   ↓
View
   ↓
HTML
```

Whereas:

### `@RestController`

```java
@RestController
public class UserController {

    @GetMapping("/users")
    public List<UserDto> users() {
        return users;
    }
}
```

The list becomes JSON:

```text
Java object
    ↓
Jackson
    ↓
JSON
    ↓
HTTP response
```

### Remember

```text
@Controller
      ↓
usually MVC / views

@RestController
      ↓
REST / response body
```

---

# 3. What does `@RequestMapping` do?

### Interview answer

> `@RequestMapping` maps HTTP requests to a controller class or method based on URL, HTTP method, parameters, headers, or other request conditions.

Most commonly you'll see it defining a base path:

```java
@RestController
@RequestMapping("/users")
public class UserController {
}
```

Then:

```java
@GetMapping("/{id}")
public UserDto getUser(@PathVariable Long id) {
}
```

creates:

```text
GET /users/42
```

You can also use:

```java
@RequestMapping(
    path = "/users",
    method = RequestMethod.GET
)
```

but these specialized annotations are preferred:

```java
@GetMapping
@PostMapping
@PutMapping
@PatchMapping
@DeleteMapping
```

### Interview tip

`@RequestMapping` isn't just about URLs. It can also specify:

```text
method
params
headers
consumes
produces
```

For example:

```java
@GetMapping(
    value = "/users",
    produces = "application/json"
)
```

---

# 4. Difference between `@GetMapping` and `@PostMapping`?

### Interview answer

> `@GetMapping` maps HTTP GET requests, normally used to retrieve resources. `@PostMapping` maps HTTP POST requests, commonly used to create resources or trigger operations.

Example:

```java
@GetMapping("/{id}")
public UserDto getUser(@PathVariable Long id) {
    ...
}
```

Handles:

```http
GET /users/42
```

Whereas:

```java
@PostMapping
public UserDto createUser(@RequestBody CreateUserRequest request) {
    ...
}
```

handles:

```http
POST /users
```

with:

```json
{
  "name": "John",
  "email": "john@example.com"
}
```

### REST semantics

Generally:

```text
GET      → retrieve
POST     → create / submit operation
PUT      → replace/update
PATCH    → partial update
DELETE   → delete
```

One important interview detail:

**GET should be safe and generally have no server-side side effects.**

---

# 5. What is `@PathVariable`?

### Interview answer

> `@PathVariable` extracts a value from the URI path and binds it to a method parameter.

Example:

```java
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id) {
    ...
}
```

Request:

```text
GET /users/42
```

Spring extracts:

```text
42
```

and gives it to:

```java
Long id
```

So conceptually:

```text
/users/42
   ↓
{id}
   ↓
@PathVariable Long id
```

You can explicitly specify the name:

```java
@PathVariable("id") Long userId
```

---

# 6. What is `@RequestParam`?

### Interview answer

> `@RequestParam` extracts values from the query string of an HTTP request.

For:

```text
GET /users?name=John&active=true
```

you can write:

```java
@GetMapping("/users")
public List<UserDto> getUsers(
        @RequestParam String name,
        @RequestParam boolean active) {

    ...
}
```

Result:

```text
name   → "John"
active → true
```

### Path vs query parameter

This distinction is important:

```text
/users/42
    ↑
@PathVariable
```

versus:

```text
/users?id=42
       ↑
@RequestParam
```

Typically:

```text
@PathVariable
    ↓
identifies a resource

@RequestParam
    ↓
filters/configures the request
```

For example:

```text
GET /users/42
GET /users?role=admin&page=2
```

---

# 7. What is `@RequestBody`?

### Interview answer

> `@RequestBody` tells Spring to deserialize the HTTP request body into a Java object, using an HTTP message converter such as Jackson for JSON.

Client sends:

```json
{
  "name": "John",
  "email": "john@example.com"
}
```

Controller:

```java
@PostMapping
public UserDto createUser(
        @RequestBody CreateUserRequest request) {

    ...
}
```

Spring effectively performs:

```text
JSON
 ↓
Jackson
 ↓
CreateUserRequest
```

So you get:

```java
request.name()
request.email()
```

---

# Intermediate

# 8. How does Spring convert JSON into a Java object?

This is an important one.

### Interview answer

> Spring MVC uses HTTP message converters to convert request and response bodies. For JSON, Spring Boot commonly configures Jackson's `ObjectMapper` as the JSON converter. When a request has `Content-Type: application/json` and the controller parameter is annotated with `@RequestBody`, Jackson deserializes the JSON into the target Java type.

The flow is:

```text
HTTP request
     │
     │ Content-Type: application/json
     ▼
Spring MVC
     │
     ▼
HttpMessageConverter
     │
     ▼
Jackson
     │
     ▼
Java object
```

For example:

```json
{
  "name": "John",
  "age": 35
}
```

becomes:

```java
CreateUserRequest
```

The reverse happens for responses:

```text
Java object
     ↓
Jackson
     ↓
JSON
     ↓
HTTP response
```

---

# 9. What is Jackson?

### Interview answer

> Jackson is a Java library used primarily for converting Java objects to JSON and JSON to Java objects.

The two terms to remember are:

### Serialization

```text
Java → JSON
```

### Deserialization

```text
JSON → Java
```

Example:

```java
User user = new User("John");
```

serialization:

```json
{
  "name": "John"
}
```

Deserialization:

```text
JSON
 ↓
User
```

Spring Boot commonly configures Jackson automatically when using its web starter.

---

# 10. Why use DTOs instead of entities?

This is a **very important senior-level question**.

### Interview answer

> DTOs separate the external API contract from the internal persistence model. This prevents exposing database implementation details, gives us control over what data clients can read or modify, avoids accidental serialization of relationships, and allows the API and database models to evolve independently.

Imagine your entity:

```java
@Entity
public class User {

    private Long id;
    private String name;
    private String email;
    private String passwordHash;
    private LocalDateTime createdAt;
}
```

You definitely don't want:

```json
{
  "id": 1,
  "name": "John",
  "email": "...",
  "passwordHash": "...",
  "createdAt": "..."
}
```

Instead:

```java
public record UserDto(
    Long id,
    String name,
    String email
) {}
```

Now:

```text
Database model
      ≠
API model
```

That's good architecture.

### DTOs also help prevent:

- accidentally updating fields the client shouldn't control
- leaking internal fields
- exposing relationships unnecessarily
- API/database coupling
- serialization problems involving lazy JPA relationships

---

# 11. What is `ResponseEntity`?

### Interview answer

> `ResponseEntity` represents the complete HTTP response and allows us to control the response body, HTTP status code, and headers.

For example:

```java
return ResponseEntity.ok(user);
```

produces:

```text
200 OK
```

with the user as the body.

Creation:

```java
return ResponseEntity
        .status(HttpStatus.CREATED)
        .body(user);
```

produces:

```text
201 Created
```

You can also set headers:

```java
return ResponseEntity
        .ok()
        .header("X-Custom", "value")
        .body(user);
```

### Why use it?

Because sometimes simply returning:

```java
UserDto
```

isn't enough.

You may need to control:

```text
status
headers
body
```

---

# 12. How do you validate incoming requests?

Usually with **Jakarta Bean Validation**.

Example:

```java
public record CreateUserRequest(

        @NotBlank
        String name,

        @Email
        @NotBlank
        String email

) {}
```

Then:

```java
@PostMapping
public UserDto create(
        @Valid @RequestBody CreateUserRequest request) {

    return service.create(request);
}
```

The important annotations include:

```text
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@Email
@Pattern
@Positive
@PositiveOrZero
```

Validation occurs before your business logic executes.

Conceptually:

```text
HTTP request
     ↓
JSON deserialization
     ↓
Validation
     ↓
Controller
     ↓
Service
```

---

# 13. What does `@Valid` do?

### Interview answer

> `@Valid` triggers Bean Validation on the annotated object, causing its validation constraints to be evaluated before the controller method proceeds.

For:

```java
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email
) {}
```

and:

```java
public UserDto create(
        @Valid @RequestBody CreateUserRequest request) {
}
```

Spring validates:

```text
name
email
```

If validation fails, the controller method normally isn't executed.

Instead, Spring produces a validation-related error response, which you can customize using global exception handling.

---

# 14. How do you handle exceptions globally?

### Interview answer

> I would use `@RestControllerAdvice` with `@ExceptionHandler` methods to centralize exception-to-HTTP-response mapping.

Example:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(
            UserNotFoundException ex) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(
                        404,
                        ex.getMessage()
                ));
    }
}
```

Now any controller can throw:

```java
throw new UserNotFoundException(id);
```

and the centralized handler converts it to:

```text
404 Not Found
```

This gives you:

```text
Controller A ─┐
Controller B ─┼──→ @RestControllerAdvice
Controller C ─┘            │
                           ▼
                     ErrorResponse
```

Instead of duplicating `try/catch` logic throughout your controllers.

---

# Senior

Now we get to the questions where the interviewer is testing whether you can **design systems**, rather than just use annotations.

---

# 15. How would you design a consistent error response across all APIs?

### Strong interview answer

> I would define a standard error response model and use a centralized `@RestControllerAdvice` to map application exceptions and validation errors to that model. I would include information such as HTTP status, application-specific error code, human-readable message, timestamp, and possibly a correlation/request ID. I would avoid exposing stack traces or internal implementation details to clients.

For example:

```json
{
  "timestamp": "2026-09-02T15:30:00Z",
  "status": 404,
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "traceId": "abc-123"
}
```

Architecture:

```text
Exception
    ↓
@RestControllerAdvice
    ↓
Exception → ErrorCode
    ↓
Standard ErrorResponse
    ↓
JSON
```

You should also distinguish:

```text
validation errors
business errors
authentication errors
authorization errors
unexpected errors
```

Don't return:

```json
{
  "error": "NullPointerException at UserService.java:52"
}
```

to a client.

---

# 16. Where should business logic live?

### Interview answer

> Business logic should primarily live in the service/domain layer rather than controllers or repositories. Controllers should handle HTTP concerns, while repositories should focus on persistence.

Think:

```text
Controller
    │
    │ HTTP concerns
    ▼
Service
    │
    │ business rules
    ▼
Repository
    │
    │ data access
    ▼
Database
```

For example, this belongs in the service:

```java
if (account.getBalance() < amount) {
    throw new InsufficientFundsException();
}
```

Not in the controller.

The controller should be relatively thin:

```java
@PostMapping("/transfer")
public void transfer(@RequestBody TransferRequest request) {
    service.transfer(request);
}
```

---

# 17. How would you implement pagination/filtering?

For a simple API:

```text
GET /users?page=0&size=20&name=John
```

Spring Data provides convenient support for pagination using `Pageable`.

Conceptually:

```java
@GetMapping
public Page<UserDto> getUsers(
        Pageable pageable) {

    return service.getUsers(pageable);
}
```

The client can specify:

```text
?page=0&size=20
```

and potentially sorting:

```text
?page=0&size=20&sort=name,asc
```

### But the senior answer goes further.

Don't blindly allow:

```text
sort=anything
```

You should:

- whitelist sortable fields
- enforce reasonable maximum page size
- validate filters
- avoid expensive queries
- use database indexes
- consider cursor/keyset pagination for very large datasets

For example:

```text
Offset pagination
    ↓
page=10000
    ↓
potentially expensive

Keyset/cursor pagination
    ↓
afterId=123456
    ↓
often better for large datasets
```

---

# 18. How would you version an API?

There are several approaches.

The most straightforward:

```text
/api/v1/users
/api/v2/users
```

For example:

```java
@RequestMapping("/api/v1/users")
```

Then later:

```java
@RequestMapping("/api/v2/users")
```

Other approaches include:

```text
URL versioning
/api/v1/users

Header versioning
Accept: application/vnd.myapp.v2+json

Query parameter
/users?version=2
```

### What I'd say in an interview

> For most public REST APIs I'd favor explicit versioning, commonly URL-based such as `/api/v1`, because it's easy for clients and operational tooling to understand. However, the important part is maintaining compatibility and having a clear deprecation strategy rather than simply creating a new version whenever the API changes.

---

# 19. How would you handle idempotency for POST requests?

This is a **good senior question**.

Normally:

```text
POST /payments
```

can create a new payment every time it's called.

Imagine the client sends the request, but the network times out.

The client doesn't know whether the payment succeeded.

It retries.

Now you might have:

```text
Payment #123 → €100
Payment #124 → €100
```

That's bad.

### Solution: Idempotency key

Client sends:

```http
POST /payments
Idempotency-Key: abc123
```

Server stores the result associated with:

```text
abc123
```

If the same request arrives again:

```text
Idempotency-Key: abc123
```

the server returns the original result instead of performing the operation again.

Conceptually:

```text
Request
   │
   ▼
Idempotency key
   │
   ▼
Redis / Database
   │
   ├── exists → return previous result
   │
   └── doesn't exist
           ↓
       process
           ↓
       store result
```

This is especially important for:

- payments
- orders
- financial operations
- resource creation

---

# 20. How would you prevent clients from accessing fields they shouldn't see?

First line of defense:

> **DTOs.**

Don't expose your entity directly.

For example:

```java
public record UserResponse(
    Long id,
    String name,
    String email
) {}
```

instead of returning:

```java
UserEntity
```

containing:

```text
passwordHash
internalFlags
securityTokens
audit fields
```

Then security controls access to **resources/actions**:

```text
Authentication
     ↓
Who are you?

Authorization
     ↓
What are you allowed to do?
```

For example:

```text
GET /users/42
```

may require:

```text
USER_READ
```

while:

```text
DELETE /users/42
```

may require:

```text
USER_ADMIN
```

So:

```text
DTO
 +
Authentication
 +
Authorization
```

protects the API.

---

# 21. How would you design a REST API for millions of requests?

This is where your existing architecture experience becomes very useful.

Don't answer:

> "I'll use Spring Boot."

Spring Boot isn't what makes an architecture scale.

I'd answer something like:

> I would keep the application stateless so instances can scale horizontally behind a load balancer. I'd optimize database access and indexing, introduce caching where appropriate, use connection pooling, and avoid blocking operations where they become bottlenecks. I'd use asynchronous messaging for workloads that don't need synchronous responses. I'd add rate limiting, observability, health checks, and autoscaling, and I'd load-test the system to identify actual bottlenecks.

Architecture:

```text
                 Load Balancer
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Spring Boot   Spring Boot   Spring Boot
          │            │            │
          └────────────┼────────────┘
                       │
                 ┌─────┴─────┐
                 ↓           ↓
               Redis       PostgreSQL
                             │
                          replicas
```

And potentially:

```text
Spring Boot
     │
     ▼
Kafka
     │
     ▼
Async workers
```

Important concepts:

```text
horizontal scaling
stateless services
caching
database optimization
connection pooling
load balancing
asynchronous processing
rate limiting
observability
autoscaling
```

You already know most of these from your cloud/microservice experience.

---

# 22. How would you handle backwards compatibility?

### Interview answer

> I would treat the API contract as a compatibility boundary. Additive changes such as new optional response fields are generally safer than removing or changing existing fields. Breaking changes should be versioned or introduced through a controlled migration/deprecation process.

For example, this is usually safer:

```json
{
  "id": 1,
  "name": "John",
  "email": "...",
  "phone": "..."
}
```

than changing:

```json
"name"
```

to:

```json
"fullName"
```

because existing clients may depend on `name`.

Generally:

### Safer

```text
add optional field
add new endpoint
accept additional optional input
```

### Potentially breaking

```text
remove field
rename field
change data type
change semantics
make optional field required
```

You should also have:

```text
deprecation period
documentation
monitoring
migration plan
```

---

# 23. How would you distinguish `401` from `403`?

This is **very commonly asked**.

### `401 Unauthorized`

Means:

> The client has not successfully authenticated.

Examples:

```text
No token
Invalid token
Expired token
```

Conceptually:

```text
"Who are you?"
       ↓
I don't know.
       ↓
401
```

### `403 Forbidden`

Means:

> The client is authenticated, but doesn't have permission to perform the operation.

For example:

```text
User authenticated
        ↓
Role = USER
        ↓
DELETE /users/42
        ↓
Requires ADMIN
        ↓
403
```

So remember:

```text
401 → Authentication problem
403 → Authorization problem
```

A simple interview phrase:

> **401 = I don't know who you are. 403 = I know who you are, but you're not allowed to do this.**

---

# 24. What happens inside Spring when an HTTP request arrives?

This is probably the **most important senior question in this section**.

You should understand the flow.

Suppose:

```http
GET /users/42
```

### Step 1 — Request reaches the server

Spring Boot commonly runs with an embedded server such as Tomcat.

```text
HTTP request
     ↓
Tomcat
```

### Step 2 — Spring's web infrastructure receives it

The request enters Spring MVC.

A central component is:

> `DispatcherServlet`

Conceptually:

```text
Tomcat
   ↓
DispatcherServlet
```

### Step 3 — Find the appropriate controller

Spring determines which controller method matches:

```text
GET /users/42
```

For example:

```java
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id)
```

The `DispatcherServlet` uses handler mappings to find the appropriate handler.

### Step 4 — Resolve method arguments

Spring extracts:

```text
42
```

for:

```java
@PathVariable Long id
```

If there's:

```java
@RequestParam
```

Spring extracts query parameters.

If there's:

```java
@RequestBody
```

Spring uses an HTTP message converter/Jackson to deserialize the body.

### Step 5 — Invoke the controller

Spring calls:

```java
getUser(42L)
```

### Step 6 — Controller calls service

```text
Controller
    ↓
Service
```

### Step 7 — Service calls repository

```text
Service
    ↓
Repository
    ↓
Database
```

### Step 8 — Result returns

Suppose:

```java
UserDto user
```

comes back.

Spring uses an HTTP message converter, commonly Jackson, to serialize:

```text
UserDto
   ↓
JSON
```

### Step 9 — HTTP response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "id": 42,
  "name": "John"
}
```

---

# The complete request lifecycle

This is worth memorizing conceptually:

```text
                    HTTP Request
                         │
                         ▼
                    Web Server
                    (Tomcat)
                         │
                         ▼
                 DispatcherServlet
                         │
                         ▼
                  Handler Mapping
                         │
                         ▼
                Controller Method
                         │
              ┌──────────┴──────────┐
              │                     │
       @PathVariable          @RequestBody
       @RequestParam          + Jackson
              │                     │
              └──────────┬──────────┘
                         ▼
                       Service
                         │
                         ▼
                     Repository
                         │
                         ▼
                      Database
                         │
                         ▼
                     Java object
                         │
                         ▼
                      Jackson
                         │
                         ▼
                   JSON response
                         │
                         ▼
                       Client
```

There are also things that can happen **before the controller**:

```text
HTTP
 ↓
Security filters
 ↓
Other filters/interceptors
 ↓
DispatcherServlet
 ↓
Controller
```

We'll cover the security filter chain later.

---

# What I want you to retain for the interview

Don't try to memorize all the Spring annotations independently. Build this model:

```text
                       HTTP
                        │
                        ▼
                DispatcherServlet
                        │
                        ▼
                   Controller
                        │
                ┌───────┴────────┐
                │                │
        @PathVariable       @RequestBody
        @RequestParam          ↓
                │           Jackson
                └───────┬────────┘
                        ▼
                     Service
                        │
                  Business Logic
                        │
                        ▼
                   Repository
                        │
                        ▼
                    Database
```

And remember these mappings:

| HTTP concept             | Spring                     |
| ------------------------ | -------------------------- |
| `/users/42`              | `@PathVariable`            |
| `?page=2`                | `@RequestParam`            |
| JSON request body        | `@RequestBody`             |
| JSON serialization       | Jackson                    |
| HTTP endpoint            | `@RestController`          |
| URL mapping              | `@RequestMapping`          |
| GET                      | `@GetMapping`              |
| POST                     | `@PostMapping`             |
| Validation               | `@Valid` + Bean Validation |
| Global errors            | `@RestControllerAdvice`    |
| HTTP status/body/headers | `ResponseEntity`           |

### The senior-level picture

And this is the architecture I'd expect you to be able to explain on a whiteboard:

```text
             Client
                │
                ▼
         Load Balancer
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
   Spring Boot Spring Boot Spring Boot
       │        │        │
       └────────┼────────┘
                │
       ┌────────┼─────────┐
       ▼        ▼         ▼
     Redis   PostgreSQL  Kafka
```

with:

```text
Security
Validation
Exception handling
Logging
Metrics
Tracing
Rate limiting
```

around the API layer.

---

## One important correction to keep in mind

Since you're coming from Express, don't think of Spring as:

> "Express but with Java annotations."

The important difference is that Spring has a **large managed runtime/container model** underneath the API:

```text
Spring Container
      │
      ├── Beans
      ├── Dependency Injection
      ├── Proxies/AOP
      ├── Transactions
      ├── Security
      ├── Configuration
      └── Web infrastructure
```

The REST controller is just one part of that ecosystem.

---

### Next: JPA, Hibernate & Spring Data

This is the next topic I'd move to.

And I'd recommend we spend **considerably more time on it than the REST annotations**, because interviews for experienced Spring Boot developers very frequently go into:

**JPA vs Hibernate vs Spring Data JPA → Entity lifecycle → relationships → LAZY/EAGER → N+1 → transactions → JPQL → repository queries → pagination → optimistic/pessimistic locking.**

Those are the areas where a candidate who has only "used Spring Boot" versus someone who actually understands Spring Boot tend to separate.
