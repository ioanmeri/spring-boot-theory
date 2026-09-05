## 25. Spring AOP ⭐⭐⭐⭐⭐

This is **very important for interviews**, especially because you just learned `@Transactional`.

The key idea is:

> **AOP allows Spring to add behavior around method calls without putting that behavior directly inside your business code.**

Typical cross-cutting concerns:

* transactions
* security
* caching
* logging
* auditing
* async execution

---

# 1. What problem does AOP solve?

Imagine you have:

```java
public void transferMoney() {
    // start transaction

    validateTransfer();
    updateAccount();
    saveTransaction();

    // commit transaction
}
```

You might need the same transaction logic around hundreds of methods.

Without AOP, you'd end up with:

```java
beginTransaction();

try {
    doBusinessLogic();
    commit();
} catch (Exception e) {
    rollback();
}
```

everywhere.

That's undesirable because transaction management is **not the business logic**.

AOP separates these concerns:

```text
                ┌─────────────────────┐
                │   Your Business      │
                │      Method          │
                └─────────────────────┘
                         ▲
                         │
              ┌─────────────────────┐
              │    Spring Proxy     │
              │                     │
              │  transaction       │
              │  security          │
              │  caching            │
              │  logging            │
              └─────────────────────┘
```

The proxy intercepts the call and performs additional behavior.

---

# 2. The most important concept: Proxy-based AOP

This is probably the **single most important thing to understand**.

Suppose you have:

```java
@Service
public class PaymentService {

    @Transactional
    public void pay() {
        // business logic
    }
}
```

You might imagine:

```text
Controller
    │
    ▼
PaymentService.pay()
```

But conceptually Spring does something more like:

```text
Controller
    │
    ▼
PaymentService PROXY
    │
    ├── start transaction
    │
    ▼
real PaymentService
    │
    └── pay()
```

So when you call:

```java
paymentService.pay();
```

the object you receive from Spring may actually be a **proxy**.

The proxy intercepts the method call.

For `@Transactional`, it effectively does:

```text
method call
     ↓
proxy
     ↓
start transaction
     ↓
real method
     ↓
commit / rollback
```

This is why understanding AOP helps you understand `@Transactional`.

---

# 3. Why does Spring use proxies?

Because Spring can add behavior **without modifying your class**.

For example:

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        ...
    }
}
```

Spring doesn't need you to write transaction-management code.

Instead:

```text
Your class
    +
Spring proxy
    =
Transactional behavior
```

This same mechanism is used by many Spring features.

---

# 4. JDK Dynamic Proxy vs CGLIB

This is an interview favorite.

There are two important proxy mechanisms.

## JDK Dynamic Proxy

Works through **interfaces**.

For example:

```java
public interface PaymentService {
    void pay();
}
```

```java
@Service
public class PaymentServiceImpl implements PaymentService {

    @Transactional
    public void pay() {
        ...
    }
}
```

Spring can create something conceptually like:

```text
PaymentService interface
        ↑
   JDK Proxy
        │
        ▼
PaymentServiceImpl
```

The proxy implements the interface.

---

# 5. CGLIB proxy

CGLIB works by creating a **subclass** of your class.

Conceptually:

```text
PaymentService
      ↑
      │ extends
      │
PaymentService$$Proxy
```

The proxy overrides methods and intercepts calls.

So:

```java
public class PaymentService {
    public void pay() {
    }
}
```

can conceptually become:

```text
PaymentServiceProxy extends PaymentService
```

This is particularly useful when there isn't an interface.

### Interview-level distinction

| JDK Proxy                         | CGLIB                      |
| --------------------------------- | -------------------------- |
| Uses interface                    | Uses subclassing           |
| Proxy implements interface        | Proxy extends class        |
| Interface-based                   | Class-based                |
| Cannot proxy without an interface | Can proxy concrete classes |

Modern Spring commonly uses class-based proxies by default, but the important interview concept is **interface proxy vs subclass proxy**.

---

# 6. Join Point

A **join point** is a point in program execution where additional behavior can be applied.

In Spring AOP, the important practical join points are:

> **method executions**

For example:

```java
createOrder()
cancelOrder()
deleteOrder()
```

Each method execution can be a join point.

Don't overcomplicate this for a Spring interview.

---

# 7. Pointcut

A **pointcut** defines **which join points should be intercepted**.

For example:

```java
execution(* com.example.service.*.*(..))
```

means approximately:

> Intercept methods in the service package.

Think:

```text
Join points:
    createOrder()
    cancelOrder()
    deleteOrder()
    calculatePrice()

Pointcut:
    "Which of these should I intercept?"
```

So:

**Join point = possible interception location**

**Pointcut = rule selecting locations**

---

# 8. Advice

Advice is:

> **What should happen when the selected method is intercepted?**

For example:

```text
@Before
    ↓
execute method
    ↓
@After
```

Or:

```text
@Around
    ↓
before
    ↓
execute method
    ↓
after
```

The major types you should know:

| Advice            | Meaning                 |
| ----------------- | ----------------------- |
| `@Before`         | Before method           |
| `@After`          | After method            |
| `@AfterReturning` | After successful return |
| `@AfterThrowing`  | After exception         |
| `@Around`         | Wraps the entire method |

For your interview, the first three you absolutely need are:

**`@Before`, `@After`, `@Around`**

---

# 9. `@Aspect`

An aspect contains cross-cutting behavior.

Example:

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() {
        System.out.println("Method called");
    }
}
```

The important pieces are:

```java
@Aspect
```

→ this class contains AOP behavior.

```java
@Component
```

→ Spring manages the aspect as a bean.

```java
@Before(...)
```

→ defines when the advice runs.

---

# 10. `@Before`

Runs before the target method.

```java
@Before("execution(* com.example.service.*.*(..))")
public void beforeMethod() {
    System.out.println("Before method");
}
```

Conceptually:

```text
proxy
  ↓
@Before
  ↓
actual method
```

Good use cases:

* logging
* authorization checks
* auditing

---

# 11. `@After`

Runs after the method completes, whether normally or exceptionally.

```java
@After("execution(* com.example.service.*.*(..))")
public void afterMethod() {
    System.out.println("After method");
}
```

Conceptually:

```text
method
   ↓
completed
   ↓
@After
```

If you specifically need to know whether the method **returned successfully**, use:

```java
@AfterReturning
```

If you specifically want to react to an exception:

```java
@AfterThrowing
```

---

# 12. `@Around` ⭐⭐⭐⭐⭐

This is the most powerful advice type.

It wraps the method.

Example:

```java
@Around("execution(* com.example.service.*.*(..))")
public Object around(ProceedingJoinPoint joinPoint) throws Throwable {

    long start = System.currentTimeMillis();

    Object result = joinPoint.proceed();

    long duration = System.currentTimeMillis() - start;

    System.out.println("Duration: " + duration);

    return result;
}
```

The key line is:

```java
joinPoint.proceed();
```

That means:

> **Execute the original method.**

Conceptually:

```text
@Around
   │
   ├── before logic
   │
   ├── joinPoint.proceed()
   │       │
   │       ▼
   │   actual method
   │
   └── after logic
```

And because `@Around` controls the call, it can even:

* modify arguments
* modify return values
* prevent the method from executing
* catch exceptions
* measure execution time

That's why it is powerful.

---

# 13. Example: Logging with AOP

Suppose you have:

```java
@Service
public class OrderService {

    public void createOrder() {
        System.out.println("Creating order");
    }
}
```

Instead of:

```java
public void createOrder() {
    log("starting");
    ...
    log("finished");
}
```

you can have:

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object log(ProceedingJoinPoint joinPoint) throws Throwable {

        System.out.println("Starting: " +
                joinPoint.getSignature().getName());

        Object result = joinPoint.proceed();

        System.out.println("Finished");

        return result;
    }
}
```

Now all matching service methods automatically get logging.

---

# 14. Self-invocation ⭐⭐⭐⭐⭐

This is **extremely important** for Spring interviews.

Consider:

```java
@Service
public class OrderService {

    public void createOrder() {
        calculatePrice();
    }

    @Transactional
    public void calculatePrice() {
        ...
    }
}
```

You might expect:

```text
createOrder()
      ↓
@Transactional proxy
      ↓
calculatePrice()
```

But that's **not what happens**.

The call is:

```text
OrderService
   │
   └── this.calculatePrice()
```

The method calls another method **inside the same object**.

It doesn't go through the Spring proxy.

Therefore:

```java
@Transactional
```

on `calculatePrice()` may **not be applied**.

---

# 15. Why?

Remember:

```text
external caller
      ↓
    PROXY
      ↓
 actual object
```

But inside the object:

```java
this.calculatePrice();
```

is effectively:

```text
actual object
      ↓
this.calculatePrice()
```

The proxy is bypassed.

Therefore:

> **Spring proxy-based AOP only works when the method call goes through the proxy.**

This is one of the most important interview statements to memorize.

---

# 16. Common example with `@Transactional`

```java
@Service
public class UserService {

    public void register() {
        saveUser();
    }

    @Transactional
    public void saveUser() {
        ...
    }
}
```

Calling:

```java
userService.register();
```

does **not necessarily activate the transaction on `saveUser()`**, because the call is internal.

Better design:

```java
@Service
public class RegistrationService {

    private final UserService userService;

    public void register() {
        userService.saveUser();
    }
}
```

Now:

```text
RegistrationService
        ↓
UserService proxy
        ↓
@Transactional
        ↓
saveUser()
```

The proxy is involved.

---

# 17. Proxy limitations

You should know these for interviews.

### 1. Self-invocation

```java
this.someMethod();
```

bypasses the proxy.

### 2. Private methods

Spring proxy-based AOP generally cannot intercept private methods.

```java
private void doSomething() {
}
```

Don't put `@Transactional` on a private method expecting proxy interception.

### 3. Final methods

Subclass-based proxies cannot override final methods.

```java
public final void doSomething() {
}
```

So CGLIB-style interception cannot work normally there.

### 4. Final classes

A subclass proxy cannot extend a final class.

```java
public final class MyService {
}
```

This matters particularly for class-based proxies.

---

# 18. How this connects to `@Transactional`

Now you can understand something that is often asked in interviews:

### Is `@Transactional` implemented using AOP?

**Yes.**

Conceptually:

```text
@Transactional
       ↓
Spring detects transactional bean
       ↓
Spring creates proxy
       ↓
method call
       ↓
proxy intercepts
       ↓
transaction starts
       ↓
target method executes
       ↓
commit / rollback
```

You don't manually write the transaction handling.

---

# 19. Other Spring features using the same idea

This is exactly why your topic list includes these:

### `@Transactional`

```text
method
  ↓
transaction interceptor
  ↓
method
```

### `@Cacheable`

```text
method call
    ↓
cache interceptor
    ↓
is value cached?
   / \
 yes  no
 ↓     ↓
return  execute method
        ↓
       cache
```

### `@Async`

```text
method call
    ↓
proxy
    ↓
submit to executor
    ↓
return
```

The method can execute asynchronously.

### `@PreAuthorize`

```text
method call
    ↓
security interceptor
    ↓
authorized?
   / \
 yes  no
 ↓     ↓
execute exception
```

So a very useful mental model is:

> **Spring annotations such as `@Transactional`, `@Cacheable`, `@Async`, and `@PreAuthorize` can trigger behavior through Spring's proxy/interceptor mechanism.**

---

# 20. The complete AOP vocabulary

This is worth memorizing:

```text
AOP
 │
 ├── Aspect
 │      → cross-cutting concern
 │
 ├── Join Point
 │      → possible interception point
 │
 ├── Pointcut
 │      → selects which join points
 │
 └── Advice
        → what code to execute
```

And:

```text
Advice types:

@Before
@After
@AfterReturning
@AfterThrowing
@Around
```

---

# 21. Interview questions you should expect

### Q: What is AOP?

Good answer:

> AOP is a programming technique for separating cross-cutting concerns such as transactions, logging, security, and caching from business logic. In Spring, AOP is primarily implemented using proxies that intercept method calls and execute additional behavior.

---

### Q: How does `@Transactional` work?

> Spring creates a proxy around the transactional bean. When a method is called through that proxy, the transaction interceptor starts a transaction, invokes the target method, and then commits or rolls back depending on the outcome.

---

### Q: What is a pointcut?

> A pointcut is an expression that defines which method executions should be intercepted by an aspect.

---

### Q: What is advice?

> Advice is the code that executes at a selected join point, such as before, after, or around a method execution.

---

### Q: Difference between `@Before` and `@Around`?

> `@Before` executes before the target method, while `@Around` wraps the method and gives control over whether and how the target method executes through `ProceedingJoinPoint.proceed()`.

---

### Q: What is self-invocation?

> Self-invocation occurs when a method calls another method on the same object, typically through `this`. Because the call bypasses the Spring proxy, proxy-based AOP advice such as `@Transactional` may not be applied.

---

### Q: JDK proxy vs CGLIB?

> JDK dynamic proxies are interface-based, while CGLIB uses subclassing to create class-based proxies.

---

# What you actually need to know for your interview

### ⭐⭐⭐⭐⭐ Absolutely know

* What AOP is
* Why cross-cutting concerns exist
* Spring's proxy-based AOP
* How `@Transactional` uses the proxy
* `@Aspect`
* `@Before`
* `@Around`
* join point vs pointcut vs advice
* self-invocation
* basic proxy limitations
* JDK proxy vs CGLIB
* `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize` as examples

### ⭐⭐⭐ Know conceptually

* `@After`
* `@AfterReturning`
* `@AfterThrowing`
* `ProceedingJoinPoint`

### ⭐ Don't spend much time on

* AOP implementation internals
* AspectJ bytecode weaving
* custom `BeanPostProcessor` implementations
* advanced pointcut expression syntax
* obscure proxy configuration

---

## The mental model to remember

If you remember only one diagram, remember this:

```text
                YOUR CODE
                   │
                   │ method call
                   ▼
          ┌─────────────────┐
          │  SPRING PROXY   │
          │                 │
          │ @Transactional  │
          │ @Cacheable      │
          │ @PreAuthorize   │
          │ @Async          │
          │ @Aspect         │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │  REAL OBJECT    │
          │                 │
          │ business logic  │
          └─────────────────┘
```

And the **golden rule**:

> **If the call goes through the Spring proxy, AOP advice can run. If the call bypasses the proxy—most notably through self-invocation—the advice won't be applied.**

That one concept explains a surprisingly large number of Spring interview questions.
