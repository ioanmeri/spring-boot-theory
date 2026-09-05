# 26. Spring Boot + Microservices ⭐⭐⭐⭐⭐

For a **mid-level / senior interview**, you don't need to memorize every microservices technology. What matters is that you can explain **how services communicate, what can go wrong, and how you make communication resilient**.

The core mental model:

```text
                 API Gateway
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      Order       Payment     User
      Service     Service     Service
          │          │
          └────── HTTP ────────┘
                    │
             failures happen
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     timeout      retry     circuit breaker
```

---

# 1. Service-to-service communication ⭐⭐⭐⭐⭐

Suppose:

```text
Order Service
      │
      │ create payment
      ▼
Payment Service
```

The Order Service needs to communicate with the Payment Service.

There are two broad approaches:

### Synchronous

```text
Order → HTTP → Payment → response → Order
```

The caller waits.

Common example:

```text
REST / HTTP
```

### Asynchronous

```text
Order → Message Broker → Payment
```

The caller doesn't necessarily wait for an immediate response.

Common technologies:

```text
Kafka
RabbitMQ
```

For this topic, focus mainly on **synchronous REST communication**, but understand that asynchronous messaging is often preferable when you don't need an immediate response.

### Interview question

**"How would two Spring Boot microservices communicate?"**

Good answer:

> They can communicate synchronously using HTTP/REST, typically through a REST client such as RestClient, WebClient, or OpenFeign. For asynchronous communication, they can use messaging systems such as Kafka or RabbitMQ. For synchronous calls, I would also consider timeouts, retries, circuit breakers, and observability.

That's a strong answer.

---

# 2. REST clients ⭐⭐⭐⭐⭐

Your service needs to call another service.

For example:

```text
OrderService
     │
     │ GET /users/42
     ▼
UserService
```

Spring provides several ways to make the HTTP call.

The ones you listed are:

```text
RestClient
WebClient
OpenFeign
```

You should understand **why each exists**, rather than memorizing APIs.

---

# 3. `RestClient` ⭐⭐⭐⭐⭐

`RestClient` is Spring's modern **synchronous HTTP client**.

Conceptually:

```java
User user = restClient
    .get()
    .uri("/users/{id}", id)
    .retrieve()
    .body(User.class);
```

The important characteristic:

> **Synchronous / blocking HTTP communication.**

Meaning:

```text
Thread
  │
  ▼
HTTP request
  │
  │ waiting...
  ▼
response
  │
  ▼
continue
```

For typical Spring MVC applications, `RestClient` is often the straightforward choice.

### Interview answer

> `RestClient` is Spring's synchronous HTTP client for making REST calls. It's a good choice when the application follows the traditional blocking Spring MVC model.

---

# 4. WebClient ⭐⭐⭐⭐

`WebClient` is Spring's **reactive HTTP client**.

It belongs to Spring WebFlux.

Conceptually:

```text
Thread
  │
  ▼
send request
  │
  └───────► response later
                  │
                  ▼
             continue processing
```

The important distinction:

```text
RestClient → synchronous/blocking
WebClient  → reactive/non-blocking
```

Don't get trapped into thinking:

> "WebClient is always better."

It isn't.

If you're building a normal Spring MVC application, `RestClient` is often simpler.

If you're already using a reactive architecture/WebFlux and need non-blocking I/O, `WebClient` makes more sense.

---

# 5. OpenFeign ⭐⭐⭐⭐⭐

OpenFeign provides a **declarative HTTP client**.

Instead of manually constructing HTTP requests:

```java
restClient
    .get()
    .uri("/users/{id}", id)
    ...
```

you define an interface:

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
}
```

Then:

```java
User user = userClient.getUser(42L);
```

The implementation is generated for you.

So the mental model is:

```text
RestClient
    ↓
"You manually make the HTTP call"

Feign
    ↓
"You describe the HTTP API as an interface"
```

### Interview answer

> OpenFeign is a declarative HTTP client. You define an interface describing the remote API, and the framework generates the client implementation. It reduces boilerplate compared with manually constructing HTTP requests.

---

# 6. RestClient vs WebClient vs Feign

This table is enough for most interviews:

|                  | RestClient | WebClient             | OpenFeign                      |
| ---------------- | ---------- | --------------------- | ------------------------------ |
| Style            | Imperative | Reactive              | Declarative                    |
| Blocking         | Yes        | Non-blocking/reactive | Typically blocking             |
| Main use         | MVC apps   | WebFlux/reactive      | Simple service-to-service APIs |
| Boilerplate      | Low        | Medium                | Very low                       |
| Define interface | No         | No                    | Yes                            |

The important interview distinction is:

> **RestClient = synchronous HTTP API**
> **WebClient = reactive/non-blocking HTTP API**
> **Feign = declarative HTTP API**

---

# 7. Timeouts ⭐⭐⭐⭐⭐

This is **very important in microservices**.

Imagine:

```text
Order Service
      │
      │ request
      ▼
Payment Service
      │
      │
      X  hanging
```

Without a timeout:

```text
Order Service
    │
    └── waiting...
           │
           └── waiting...
                  │
                  └── resources exhausted
```

Eventually your application can become unhealthy because requests are waiting on downstream services.

A timeout says:

> "If the dependency doesn't respond within X seconds, stop waiting."

There are commonly two concepts:

### Connection timeout

How long to establish the connection.

### Read/response timeout

How long to wait for the response.

For interviews:

> **Every remote call should have sensible timeouts. Never allow an external dependency to block indefinitely.**

---

# 8. Retries ⭐⭐⭐⭐⭐

Suppose:

```text
Order → Payment
         ↓
      temporary failure
```

A retry can attempt the request again:

```text
attempt 1 → failure
attempt 2 → failure
attempt 3 → success
```

This can help with **transient failures**.

But retries are dangerous.

Imagine 100 requests arrive:

```text
100 requests
    ↓
each retries 3 times
    ↓
300 requests
    ↓
overloaded Payment Service
```

You can make the outage worse.

### Good retry strategy

Usually:

* limited number of retries
* exponential backoff
* possibly jitter
* only retry appropriate failures

Example:

```text
attempt 1
   ↓
wait 100ms
   ↓
attempt 2
   ↓
wait 300ms
   ↓
attempt 3
```

### Critical interview point

**Don't blindly retry every operation.**

For example:

```text
GET /users/42
```

is generally safer to retry than:

```text
POST /payments
```

because repeating a payment operation could potentially create duplicate effects.

This connects directly to **idempotency**, which you studied in REST API design.

---

# 9. Circuit Breaker ⭐⭐⭐⭐⭐

This is one of the most important microservices patterns.

Suppose:

```text
Order Service
      │
      ▼
Payment Service
      X
   failing
```

If Order Service keeps calling Payment Service:

```text
request
request
request
request
request
request
...
```

you're wasting resources and potentially making the situation worse.

A circuit breaker detects repeated failures and **stops making calls temporarily**.

Conceptually:

```text
              ┌─────────────┐
              │   CLOSED    │
              │ normal      │
              └──────┬──────┘
                     │ failures
                     ▼
              ┌─────────────┐
              │    OPEN     │
              │ fail fast   │
              └──────┬──────┘
                     │ after timeout
                     ▼
              ┌─────────────┐
              │ HALF-OPEN   │
              │ test request│
              └─────────────┘
```

### CLOSED

Normal operation.

```text
requests → service
```

### OPEN

Too many failures.

```text
requests → fail immediately
```

The downstream service isn't called.

### HALF-OPEN

After some time, allow a small number of test requests.

If successful:

```text
HALF-OPEN → CLOSED
```

If they fail:

```text
HALF-OPEN → OPEN
```

### Interview answer

> A circuit breaker prevents repeated calls to an unhealthy dependency. After a configured failure threshold, it opens and fails fast, giving the downstream service time to recover. It later enters half-open state to test whether the dependency has recovered.

---

# 10. Bulkheads ⭐⭐⭐⭐

The name comes from ships.

A ship has separate compartments so that if one compartment floods, the entire ship doesn't sink.

Same idea in microservices.

Imagine:

```text
Application
│
├── Payment calls
│
├── User calls
│
└── Notification calls
```

Without isolation:

```text
Payment becomes slow
      ↓
all threads become occupied
      ↓
entire application suffers
```

With bulkheads:

```text
Payment → limited resources
User    → separate resources
Email   → separate resources
```

So:

> **Bulkhead isolation prevents one failing or slow dependency from consuming all application resources.**

This is less commonly asked than circuit breakers, but you should know the concept.

---

# 11. Resilience4j ⭐⭐⭐⭐⭐

Resilience4j is a library commonly used in Spring applications for resilience patterns.

It provides things such as:

```text
Circuit Breaker
Retry
Rate Limiter
Bulkhead
Time Limiter
```

So you can think:

```text
Spring Boot
    │
    ▼
Resilience4j
    │
    ├── Retry
    ├── Circuit Breaker
    ├── Bulkhead
    └── Rate Limiter
```

You don't need to memorize its configuration syntax for a normal interview.

Know **what problem each pattern solves**.

---

# 12. How these patterns work together ⭐⭐⭐⭐⭐

This is a very good interview scenario.

Suppose:

```text
Order Service → Payment Service
```

A production call might have:

```text
                 Order Service
                       │
                       ▼
                Circuit Breaker
                       │
                       ▼
                    Retry
                       │
                       ▼
                    Timeout
                       │
                       ▼
                Payment Service
```

And potentially:

```text
Bulkhead
    ↓
limits resources used by Payment calls
```

The purpose is resilience.

If Payment is temporarily slow:

```text
timeout
```

If the failure might be transient:

```text
retry
```

If it keeps failing:

```text
circuit breaker opens
```

If Payment consumes too many resources:

```text
bulkhead isolation
```

This is exactly the kind of reasoning interviewers like.

---

# 13. Service Discovery ⭐⭐⭐⭐⭐

Now imagine you have:

```text
Order Service
Payment Service
User Service
```

Where is Payment Service?

You don't want to hardcode:

```text
http://10.0.2.17:8080
```

because microservice instances can change.

You need **service discovery**.

Conceptually:

```text
              Service Registry
             ┌───────────────┐
             │ payment → ... │
             │ user → ...    │
             │ order → ...   │
             └───────────────┘
                ▲       ▲
                │       │
          register    lookup
                │       │
             Services
```

A service registers itself:

```text
Payment Service
      ↓
"payment-service is available at X"
```

Another service can discover it:

```text
Order Service
      ↓
"Where is payment-service?"
      ↓
Service Registry
      ↓
instance address
```

### Technologies

Historically, you may hear:

* Eureka
* Consul
* ZooKeeper

In modern cloud/Kubernetes environments, service discovery is often provided by **Kubernetes/DNS or cloud infrastructure**, so don't assume Eureka is mandatory.

### Interview answer

> Service discovery allows services to find dynamically changing service instances without hardcoding their network locations. In Kubernetes, this is commonly handled by Kubernetes Services and DNS; dedicated registries such as Eureka or Consul are another approach.

---

# 14. API Gateway ⭐⭐⭐⭐⭐

Now imagine clients directly call every service:

```text
Frontend
 ├──→ User Service
 ├──→ Order Service
 ├──→ Payment Service
 ├──→ Product Service
 └──→ Notification Service
```

This becomes messy.

Instead:

```text
             Frontend
                 │
                 ▼
            API Gateway
          /      |      \
         /       |       \
       User    Order    Product
```

The gateway is the **single entry point** for external clients.

It can handle things such as:

* routing
* authentication
* authorization-related enforcement
* rate limiting
* request filtering
* TLS termination
* sometimes aggregation

Example:

```text
GET /api/orders/123
        ↓
API Gateway
        ↓
Order Service
```

### Important distinction

Don't say:

> "The API Gateway is where all business logic goes."

That's bad architecture.

The gateway should generally handle **cross-cutting edge concerns**, not domain business logic.

---

# 15. API Gateway vs Service Discovery

These are often confused.

### API Gateway

Answers:

> **"How do external clients enter my system?"**

```text
Client → Gateway → Services
```

### Service Discovery

Answers:

> **"How do services find each other?"**

```text
Service A → Discovery → Service B
```

They solve different problems.

---

# 16. Configuration management ⭐⭐⭐⭐⭐

You already learned Spring Boot configuration, so this should connect directly.

You don't want:

```java
String paymentUrl =
    "http://10.0.2.17:8080";
```

Instead:

```yaml
payment:
  service:
    url: ${PAYMENT_SERVICE_URL}
```

Then each environment can provide a different value.

```text
Development
    ↓
payment-service-dev

Production
    ↓
payment-service-prod
```

The principle:

> **Keep environment-specific configuration outside the application code.**

For microservices, configuration commonly includes:

* service URLs
* timeouts
* retry configuration
* feature flags
* database configuration
* credentials/secrets

Secrets should be handled separately through a secrets-management mechanism rather than committed to source control.

---

# 17. Distributed tracing ⭐⭐⭐⭐⭐

This becomes critical when one request travels through multiple services.

Imagine:

```text
Frontend
   │
   ▼
Gateway
   │
   ▼
Order Service
   │
   ├──→ User Service
   │
   └──→ Payment Service
             │
             └──→ Bank API
```

Something fails.

Where?

You need to follow the request across services.

That's what distributed tracing helps with.

---

## Trace

A **trace** represents the entire request.

```text
Trace ID: abc123

Gateway
   │
   └── Order Service
          │
          ├── User Service
          │
          └── Payment Service
```

## Span

Each operation is a **span**.

```text
Trace
│
├── Gateway span
├── Order span
├── User span
├── Payment span
└── Bank API span
```

So:

> **Trace = complete journey**
> **Span = individual operation within that journey**

Modern Spring applications commonly integrate distributed tracing through **Micrometer Tracing / OpenTelemetry-compatible tooling**.

You don't need to memorize the implementation details unless the job specifically asks for them.

---

# 18. Why tracing + logging + metrics work together

You learned observability earlier.

These three complement each other:

```text
Metrics
   ↓
"Payment Service has 10% error rate."

Logs
   ↓
"Payment failed because timeout."

Trace
   ↓
"Which request path caused the timeout?"
```

Together:

```text
Metrics → WHAT is happening?
Logs    → WHAT happened / WHY?
Tracing → WHERE did it happen?
```

That's a very good interview-level explanation.

---

# 19. A complete microservices example

Imagine an e-commerce system:

```text
                       Client
                         │
                         ▼
                    API Gateway
                         │
                ┌────────┴────────┐
                ▼                 ▼
          Order Service      Product Service
                │
                ▼
         Payment Service
```

Order → Payment might use:

```text
             Order Service
                   │
                   ▼
             HTTP Client
                   │
                   ▼
             Timeout
                   │
                   ▼
               Retry
                   │
                   ▼
          Circuit Breaker
                   │
                   ▼
          Payment Service
```

And infrastructure provides:

```text
Service Discovery
Configuration
Distributed Tracing
Metrics
Logging
```

This is the **mental architecture** you should be able to explain in an interview.

---

# 20. The interview questions I would expect

### "How do microservices communicate?"

> Through synchronous protocols such as HTTP/REST or asynchronous messaging such as Kafka. For synchronous calls, I would consider appropriate timeouts, retries, circuit breakers and observability.

### "RestClient vs WebClient?"

> RestClient is synchronous and blocking, while WebClient is reactive and designed for non-blocking communication.

### "Why use OpenFeign?"

> It provides a declarative HTTP client where the remote API is represented as an interface, reducing boilerplate.

### "Why are timeouts important?"

> Without timeouts, a slow dependency can cause requests and threads to wait indefinitely, potentially exhausting resources and causing cascading failures.

### "Should you always retry?"

> No. Retries should be limited to appropriate transient failures, use backoff, and consider idempotency because retrying non-idempotent operations can cause duplicate side effects.

### "What is a circuit breaker?"

> It stops calls to a repeatedly failing dependency and fails fast until the dependency has a chance to recover.

### "What is a bulkhead?"

> It isolates resources used by different operations or dependencies so that failure or slowness in one area doesn't exhaust resources for the entire application.

### "What is service discovery?"

> It allows services to dynamically locate service instances without hardcoding their network addresses.

### "What does an API Gateway do?"

> It provides a single entry point for external clients and can handle routing and cross-cutting concerns such as authentication, rate limiting and request filtering.

### "What is distributed tracing?"

> It tracks a request across multiple services using a trace composed of multiple spans, making it possible to identify where latency or failures occur.

---

# What I would prioritize for your interview

### 🔥 Must know very well

1. **Service-to-service communication**
2. **RestClient**
3. **OpenFeign**
4. **Timeouts**
5. **Retries**
6. **Circuit breakers**
7. **Resilience4j**
8. **Service discovery**
9. **API Gateway**
10. **Distributed tracing**

### Know conceptually

* WebClient
* Bulkheads
* configuration management
* synchronous vs asynchronous communication

### Don't spend much time on

* detailed Resilience4j configuration syntax
* Eureka internals
* implementing a service registry
* implementing an API Gateway from scratch
* low-level WebFlux/reactive internals
* tracing implementation details

---

## One thing I'd especially memorize

For a **senior/mid-level interview**, if they ask:

> **"How would you make a microservice call resilient?"**

A strong answer is:

> "I'd configure sensible connection and response timeouts, use limited retries with exponential backoff for transient failures, make sure the operation is safe to retry, and use a circuit breaker to prevent repeated calls to an unhealthy dependency. For resource isolation I could use bulkheads, and I'd add metrics, logs and distributed tracing so failures are observable."

That answer demonstrates that you understand **microservices as a distributed-systems problem**, rather than simply knowing how to make an HTTP request.
