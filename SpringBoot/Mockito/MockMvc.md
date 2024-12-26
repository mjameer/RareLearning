
# Spring Testing: `@SpringBootTest` vs `@WebMvcTest`

This document provides a comprehensive overview of the differences, use cases, and configurations for `@SpringBootTest`, `@WebMvcTest`, and `@AutoConfigureMockMvc` in Spring Boot testing.

## Overview

Spring provides powerful testing annotations to test your application effectively. Two key annotations, `@SpringBootTest` and `@WebMvcTest`, are used for testing different aspects of the application. Here's a summary of their features, use cases, and configurations:

---

### **@SpringBootTest**
- **Scope**: Loads the complete application context, including controllers, services, repositories, and other beans.
- **Purpose**: Suitable for **integration testing** involving multiple layers of the application.
- **Real Logic Execution**: Executes the actual logic in controllers, services, and repositories.
- **Dependency Management**: All beans are real unless explicitly mocked.
- **Performance**: Slower due to the full context load.
- **Common Use Case**:
  - Testing end-to-end interactions between controllers, services, and repositories.

#### With `@AutoConfigureMockMvc`
- Enables and configures `MockMvc` to simulate HTTP requests and verify responses.
- Allows integration testing of controllers without starting a real web server.
- Ensures real beans are used for services and repositories.

---

### **@WebMvcTest**
- **Scope**: Loads only the web layer (controllers, `@ControllerAdvice`, Jackson converters, etc.).
- **Purpose**: Suitable for **unit testing** controllers in isolation.
- **Real Logic Execution**: Executes the actual logic in controllers only.
- **Dependency Management**: Service and repository dependencies must be mocked using `@MockBean`.
- **Performance**: Faster due to the limited context load.
- **Common Use Case**:
  - Testing the behavior, validation, and response handling of controllers.

#### Built-in MockMvc Configuration
- `@WebMvcTest` automatically configures `MockMvc`.
- No need for `@AutoConfigureMockMvc`.

---

## Choosing Between `@SpringBootTest` and `@WebMvcTest`

| Feature                  | @SpringBootTest                          | @WebMvcTest                              |
|--------------------------|-------------------------------------------|------------------------------------------|
| **Scope**               | Full application context                 | Only the web layer                       |
| **Real Dependencies**   | Yes (unless mocked)                      | No (requires mocking dependencies)       |
| **Controller Logic**    | Executes real logic                      | Executes real logic                      |
| **Performance**         | Slower                                   | Faster                                   |
| **Use Case**            | Integration testing                      | Unit testing for controllers             |

---

## Key Takeaways

- Use `@SpringBootTest` for broader integration tests involving multiple layers of the application.
- Use `@WebMvcTest` for focused unit tests of controller logic.
- Leverage `@AutoConfigureMockMvc` with `@SpringBootTest` to test controllers using `MockMvc` without starting a real web server.
- Mock dependencies as needed to keep tests isolated and efficient.

---

## Example Code Snippets

### Using `@SpringBootTest` with `MockMvc`

```java
@SpringBootTest
@AutoConfigureMockMvc
public class MyControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void shouldReturnExpectedResponse() throws Exception {
        mockMvc.perform(get("/api/resource"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.key").value("expectedValue"));
    }
}
```

### Using `@WebMvcTest`

```java
@WebMvcTest(MyController.class)
public class MyControllerUnitTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private MyService myService;

    @Test
    public void shouldReturnExpectedResponse() throws Exception {
        when(myService.getData()).thenReturn("mockedValue");

        mockMvc.perform(get("/api/resource"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.key").value("mockedValue"));
    }
}
```

### Sample Controller 

```java

@RestController
public class ExampleController {

	private final RestClient restClient;

	public ExampleController() {
		this.restClient = RestClient.builder()
			.baseUrl("https://jsonplaceholder.typicode.com")
			.build();
	}

	@GetMapping("/api/resource")
	public Object getExample() {
		return restClient.get()
			.uri("/posts/1") // Specify the endpoint
			.retrieve()
			.body(Object.class);
	}
}
```
---
