What's the difference between @Component, @Service, and @Repository

They all register classes as Spring beans through component scanning. @Service and @Repository are semantic specializations intended for service and persistence layers. @Repository also participates in Spring's persistence exception translation mechanism.

---

Which injection style do you prefer?

Constructor injection, because dependencies are explicit, fields can be final, the object is easier to test, and required dependencies are enforced when the object is constructed.

@Autowired tells Spring:
Use this constructor to inject dependencies.

---

```
@Component
     ↓
"Spring, discover this class."

@Bean
     ↓
"Spring, use this method to create the bean."
```

### Bean lifecycle

```
Spring starts
     ↓
Create bean
     ↓
Inject dependencies
     ↓
Initialization
     ↓
Bean ready
     ↓
Application runs
     ↓
Spring shuts down
     ↓
Destroy bean
```

- By default, Spring beans have singleton scope.
- becomes important with concurrency.
- the service is normally singleton-scoped, multiple HTTP requests may use the same object concurrently
- Spring services should be stateless.

---

### The complete startup process

```
             Spring Boot starts
                    │
                    ▼
          Create ApplicationContext
                    │
                    ▼
           Read configuration
                    │
                    ▼
          Perform component scan
                    │
                    ▼
        Find @Component / @Service /
        @Repository / @Controller
                    │
                    ▼
             Create beans
                    │
                    ▼
       Resolve dependencies
                    │
                    ▼
        Perform dependency injection
                    │
                    ▼
          Initialize beans
                    │
                    ▼
             Application ready
```

Basic

What is a Spring bean?
An object whose lifecycle and configuration are managed by the Spring IoC container.

What is dependency injection?
A mechanism where Spring supplies an object's dependencies rather than the object creating them itself.

What is ApplicationContext?
The Spring IoC container that manages beans, their dependencies, configuration, and lifecycle.

What happens when Spring starts?
It creates the ApplicationContext, processes configuration, scans for components, creates beans, resolves dependencies, initializes them, and starts the application infrastructure.

How does @Transactional work?
Spring typically uses proxies/AOP around the bean to intercept the method invocation and manage the transaction.
