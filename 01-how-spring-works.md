"Explain how Spring Boot works."

Spring Boot builds on the Spring Framework and uses an **IoC container** to manage application objects, which Spring calls **beans**.

The **ApplicationContext** acts as the container and handles bean creation, configuration and dependency injection. Components such as `@Service`, `@Repository` and `@RestController` are discovered through **component scanning** and registered as beans.

Spring Boot adds **conventions**, **starters** and **auto-configuration** to reduce the amount of manual configuration required.

Configuration can be supplied through properties or **YAML files**, **profiles** and **environment variables**.

---

```
IoC
 ↓
DI
 ↓
Bean
 ↓
ApplicationContext
 ↓
Configuration
 ↓
Auto-configuration
```

| Concept                | Think                                                      |
| ---------------------- | ---------------------------------------------------------- |
| **IoC**                | Spring takes control                                       |
| **DI**                 | Spring gives objects their dependencies                    |
| **Bean**               | Object managed by Spring                                   |
| **ApplicationContext** | Spring's container                                         |
| **Configuration**      | How we tell the application what/how to run                |
| **Auto-configuration** | Spring Boot automatically configures common infrastructure |

---
