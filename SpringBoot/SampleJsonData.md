### Sample Json Data Consumption via RestClient

[https://jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com)


```java
@RestController
public class ExampleController {

	private final RestClient restClient;

	public ExampleController() {
		this.restClient = RestClient.builder()
			.baseUrl("https://jsonplaceholder.typicode.com")
			.build();
	}

	@GetMapping("/api/example")
	public Object getExample() {
        return restClient.get()
            .uri("/posts/1") // Specify the endpoint
            .retrieve()
            .body(Object.class);
	}
} ```
