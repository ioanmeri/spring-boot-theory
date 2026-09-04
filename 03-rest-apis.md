```
HTTP Request
     │
     ▼
Controller
     │
     ▼
Service
     │
     ▼
Repository
     │
     ▼
Database
```

Then the response travels back:

```
PostgreSQL
    ↓
Repository
    ↓
Service
    ↓
DTO
    ↓
Controller
    ↓
JSON
```

### Controller

```java
@RestController
@RequestMapping("/users") // defines the base URL.
public class UserController {

    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable Long id) {
        ...
    }
}
```

Mappings:

```
@GetMapping
@PostMapping
@PutMapping
@PatchMapping
@DeleteMapping
```

This class handles HTTP requests and its methods return data directly as the HTTP response.

---

database entity and your API contract are different concerns.

```
HTTP
 ↓
CreateUserRequest DTO
 ↓
Service
 ↓
User Entity
 ↓
Database
```
