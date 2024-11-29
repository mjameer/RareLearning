
### CompletableFuture

CompletableFuture provides a way to compose asynchronous tasks and handle results in a non-blocking manner. It allows you to run tasks asynchronously and chain operations once the tasks complete.

```java
CompletableFuture.supplyAsync() can return value of any type.
CompletableFuture.runAsync() has no return type or void return type.
```

```java
@Component
@Slf4j
public class ProductASyncFacade {

    @Autowired
    private ProductService productService;
    @Autowired
    private InventoryService inventoryService;
    @Autowired
    private PriceService priceService;

    public CompletableFuture<Product> getProductById(long productId) {
        return CompletableFuture
                .supplyAsync(() -> productService.findById(productId));
    }

    public CompletableFuture<Price> getPriceByProductById(long productId) {
        return CompletableFuture
                .supplyAsync(() -> priceService.getPriceByProductId(productId));
    }

    public CompletableFuture<Inventory> getInventoryByProductId(long productId) {
        return CompletableFuture
                .supplyAsync(() -> inventoryService.getInventoryByProductId(productId));
    }

    public ProductDetailDTO getProductDetails(long productId) {
        //fetch all async
        CompletableFuture<Product> productFuture = getProductById(productId);
        CompletableFuture<Price> priceFuture = getPriceByProductById(productId);
        CompletableFuture<Inventory> inventoryFuture = getInventoryByProductId(productId);

        //wait for all futures to complete
        CompletableFuture.allOf(priceFuture, productFuture, inventoryFuture);

        //combine the result
        Product product = productFuture.join();
        Price price = priceFuture.join();
        Inventory inventory = inventoryFuture.join();

        //build and return

        return new ProductDetailDTO(productId, product.getCategory().getName(),
                product.getName(), product.getDescription(),
                inventory.getAvailableQuantity(), price.getPrice(),
                inventory.getStatus());

    }
}

```


### ForkJoinPool

ForkJoinPool is a specialized implementation of the ExecutorService in Java that is designed to handle parallelism more efficiently, especially for tasks that can be recursively divided into smaller tasks. 

### Best example of Combining CompletableFuture and FrokJoinPool 

```java
package org.example;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.IntStream;

import java.time.Duration;
import java.time.Instant;

import java.util.concurrent.ForkJoinPool;

public class CustomParallelStream {

    public static void main(String[] args) {
        System.out.println("no of cores : " + Runtime.getRuntime().availableProcessors());
        CustomParallelStream c = new CustomParallelStream();
        // Start the stopwatch
        Instant  start = Instant.now();
        System.out.println("Start time: " + start);

        inMethodFork(c);

        // Stop the stopwatch
        Instant end = Instant.now();
        System.out.println("End time: " + end);

        // Calculate the duration
        Duration duration = Duration.between(start, end);
        System.out.println("Total time taken: " + duration.toMillis() + " milliseconds");
    }

    // Best Performer 240 ms - By Overriding ForkJoinPool at each method Implementation level
    private static void inMethodFork(CustomParallelStream c) {
        List <Integer> numbers = IntStream.range(1, 100).boxed().toList();
        CompletableFuture<Void> voidCompletableFuture = CompletableFuture.runAsync(() -> c.forkJoinPoolImplementation(numbers));
        CompletableFuture<Void> voidCompletableFuture2 = CompletableFuture.runAsync(() -> c.forkJoinPoolImplementation(numbers));
        CompletableFuture<Void> voidCompletableFuture3 = CompletableFuture.runAsync(() -> c.forkJoinPoolImplementation(numbers));

        CompletableFuture<Void> voidCompletableFuture1 = CompletableFuture.allOf(voidCompletableFuture, voidCompletableFuture2, voidCompletableFuture3);
        voidCompletableFuture1.join();
    }

    // Ok Performer 650 ms - By Overriding ForkJoinPool at base method
    private static void inlineFork(CustomParallelStream c) {
        int parallelismLevel = 50; // Set your desired level of parallelism
        ForkJoinPool customPool = new ForkJoinPool(parallelismLevel);

        List <Integer> numbers = IntStream.range(1, 100).boxed().toList();
        CompletableFuture<Void> voidCompletableFuture = CompletableFuture.runAsync(() -> c.forkJoinPool(numbers), customPool);
        CompletableFuture<Void> voidCompletableFuture2 = CompletableFuture.runAsync(() -> c.forkJoinPool(numbers), customPool);
        CompletableFuture<Void> voidCompletableFuture3 = CompletableFuture.runAsync(() -> c.forkJoinPool(numbers), customPool);

        CompletableFuture<Void> voidCompletableFuture1 = CompletableFuture.allOf(voidCompletableFuture, voidCompletableFuture2, voidCompletableFuture3);
        voidCompletableFuture1.join();

        customPool.shutdown();
    }

    // Worst Performer 3000 ms - By Overriding Executors at base method
    private static void inExecutor(CustomParallelStream c) {
        ExecutorService executorService = Executors.newFixedThreadPool(50);

        List <Integer> numbers = IntStream.range(1, 100).boxed().toList();
        CompletableFuture<Void> voidCompletableFuture = CompletableFuture.runAsync(() -> c.forkJoinPool(numbers), executorService);
        CompletableFuture<Void> voidCompletableFuture2 = CompletableFuture.runAsync(() -> c.forkJoinPool(numbers), executorService);
        CompletableFuture<Void> voidCompletableFuture3 = CompletableFuture.runAsync(() -> c.forkJoinPool(numbers), executorService);

        CompletableFuture<Void> voidCompletableFuture1 = CompletableFuture.allOf(voidCompletableFuture, voidCompletableFuture2, voidCompletableFuture3);
        voidCompletableFuture1.join();

        executorService.shutdown();
    }

    private  void forkJoinPool(List<Integer> numbers) {
        List<Integer> list = getData(numbers);
    }

    // This gives better yield
    private  void forkJoinPoolImplementation(List<Integer> numbers) {
        int parallelismLevel = 100; // Set your desired level of parallelism
        ForkJoinPool customPool = new ForkJoinPool(parallelismLevel);
        customPool.submit( () -> getData(numbers)).join() ;
        customPool.shutdown();
    }

    private List<Integer> getData(List<Integer> numbers) {
        return numbers.parallelStream()
                .map(this::processRecord)
                .toList();
    }

    // Dummy processing function
    private  Integer processRecord(Integer record) {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println("Processing number: " + record + " by " + Thread.currentThread().getName());
        return record * 2;
    }
}

```





