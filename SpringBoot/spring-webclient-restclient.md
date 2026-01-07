# Complete Spring HTTP Interface Examples (Reactive & Sync)

**Spring Boot Version**: 3.2+ (Framework 6.1+). Uses `@HttpExchange` for declarative clients. Reactive via WebClient (non-blocking, Mono/Flux). Sync via RestClient (blocking).[1][2]

## Maven Dependencies (pom.xml)
```xml
<dependencies>
    <!-- For Sync (RestClient) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- For Reactive (WebClient) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    
    <!-- JSON handling -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```
*Notes*: Webflux enables both modes. Use `spring-boot-starter-webflux` alone for pure reactive apps.[1]

## Shared Data Model
```java
// src/main/java/com/example/model/User.java
public record User(Long id, String name, String email) {}
```
*Notes*: Records for simplicity (Java 17+). Use POJOs if needed.

## 1. Synchronous Client (RestClient - Blocking)
### Interface
```java
// src/main/java/com/example/client/SyncUserClient.java
package com.example.client;

import com.example.model.User;
import org.springframework.web.service.annotation.*;
import java.util.List;

@HttpExchange("http://localhost:8080/api/users")  // Base URL
public interface SyncUserClient {
    
    @GetExchange("/{id}")
    User getById(@PathVariable Long id);  // Returns synchronously (blocks)
    
    @GetExchange
    List<User> getAll();
    
    @PostExchange
    User create(@RequestBody User user);
    
    @PutExchange("/{id}")
    User update(@PathVariable Long id, @RequestBody User user);
    
    @DeleteExchange("/{id}")
    void delete(@PathVariable Long id);  // Void for no response
}
```
### Configuration
```java
// src/main/java/com/example/config/ClientConfig.java
package com.example.config;

import com.example.client.SyncUserClient;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.service.annotation.ImportHttpServices;

@Configuration
@ImportHttpServices({SyncUserClient.class})  // Registers all clients
public class ClientConfig {}
```
*Notes*: `@HttpExchange` generates RestClient proxy. Methods block until response. Ideal for servlet-based apps.[1]

### Usage in Service/Controller
```java
// src/main/java/com/example/service/UserService.java
package com.example.service;

import com.example.client.SyncUserClient;
import com.example.model.User;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class UserService {
    private final SyncUserClient client;

    public UserService(SyncUserClient client) {
        this.client = client;
    }
    
    public User createUser(User user) {
        return client.create(user);  // Blocks here
    }
    
    public List<User> getAllUsers() {
        return client.getAll();
    }
}
```

## 2. Reactive Client (WebClient - Non-Blocking)
### Interface
```java
// src/main/java/com/example/client/ReactiveUserClient.java
package com.example.client;

import com.example.model.User;
import org.springframework.web.service.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@HttpExchange("http://localhost:8080/api/users")
public interface ReactiveUserClient {
    
    @GetExchange("/{id}")
    Mono<User> getById(@PathVariable Long id);  // Reactive single item
    
    @GetExchange
    Flux<User> getAll();  // Reactive stream
    
    @PostExchange
    Mono<User> create(@RequestBody User user);
    
    @PutExchange("/{id}")
    Mono<User> update(@PathVariable Long id, @RequestBody User user);
    
    @DeleteExchange("/{id}")
    Mono<Void> delete(@PathVariable Long id);
}
```
### Configuration
Update `ClientConfig.java`:
```java
@Configuration
@ImportHttpServices(basePackages = "com.example.client")  // Scans all interfaces
public class ClientConfig {}
```
*Notes*: Returns `Mono` (0-1 items) or `Flux` (0-N items). Auto-configures WebClient builder.[2][1]

### Usage in Reactive Service/Controller
```java
// src/main/java/com/example/service/ReactiveUserService.java
package com.example.service;

import com.example.client.ReactiveUserClient;
import com.example.model.User;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Service
public class ReactiveUserService {
    private final ReactiveUserClient client;

    public ReactiveUserService(ReactiveUserClient client) {
        this.client = client;
    }
    
    public Mono<User> createUser(User user) {
        return client.create(user);  // Non-blocking
    }
    
    public Flux<User> getAllUsers() {
        return client.getAll();
    }
    
    // Example: Chain operations
    public Mono<User> getOrCreate(Long id, User fallback) {
        return client.getById(id)
                .switchIfEmpty(client.create(fallback));
    }
}
```
### Reactive Controller Example
```java
// src/main/java/com/example/controller/UserController.java
package com.example.controller;

import com.example.model.User;
import com.example.service.ReactiveUserService;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/users")
public class UserController {
    private final ReactiveUserService service;

    public UserController(ReactiveUserService service) {
        this.service = service;
    }
    
    @GetMapping
    public Flux<User> getAll() {
        return service.getAllUsers();
    }
    
    @PostMapping
    public Mono<User> create(@RequestBody User user) {
        return service.createUser(user);
    }
}
```
*Notes*: Use in WebFlux apps (no servlet). Subscribe or return directly in controllers. Avoid `.block()` for backpressure handling.[1]

## Key Differences & Tips
| Feature          | Sync (RestClient) | Reactive (WebClient) |
|------------------|-------------------|----------------------|
| Return Types    | POJO, List       | Mono<T>, Flux<T>    |
| Threading       | Blocking         | Non-blocking        |
| Scalability     | Lower            | High (handles 1000s reqs) |
| Error Handling  | Try-catch        | onErrorMap/doOnError |

**Tips**:
- Base URL supports variables: `@HttpExchange("${api.base-url}")`.
- Headers: `@HttpExchange(header = "Authorization=Bearer ${token}")`.
- Test sync with MockRestServiceServer; reactive with WebTestClient.
- Prod: Add timeouts, retries via WebClient.Builder customization.
