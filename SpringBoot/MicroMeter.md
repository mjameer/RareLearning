
### Tracking Metrics with Micrometer in Spring Boot: Errors, Timings, and Resources

#### Why Use Micrometer?

Micrometer is a powerful metrics instrumentation library that enables developers to monitor application behavior and performance. It provides a standardized way to collect and publish metrics, which can be integrated with monitoring tools like Prometheus, Grafana, and CloudWatch. The main use cases for Micrometer include:

- **Tracking Error Occurrences**: Helps in identifying and resolving application issues by counting errors.
- **Measuring Execution Times**: Monitors performance to ensure that services are operating within expected time limits.
- **Monitoring System Resource Usage**: Provides insights into resource utilization for better scalability and efficiency.

---

### Step 1: Configuration of Metrics

Create a configuration class to define your custom metrics. This ensures a centralized and reusable setup for metrics like counters and timers.

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsConfig {

    private static final String TAG = "appName";

    // Counter to track AWS API errors
    @Bean("trackerServiceApiCounter")
    public Counter gettrackerServiceApiCounter(MeterRegistry registry) {
        return registry.counter("tracker_service_api_error_count", TAG, "tracker api");
    }

    // Timer to measure execution time of methods
    @Bean("methodExecutionTimer")
    public Timer getMethodExecutionTimer(MeterRegistry registry) {
        return registry.timer("method_execution_time", TAG, "performance");
    }
}
```

- **Tags**: Tags add context to metrics (e.g., `appName` and `tracker api` for error tracking).
- **Counters**: Track occurrences (e.g., errors).
- **Timers**: Measure execution time of methods.

---

### Step 2: Tracking Error Occurrences

Use counters to track errors in your service methods.

```java
import io.micrometer.core.instrument.Counter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class ErrorTrackingService {

    private final Counter errorCounter;

    public ErrorTrackingService(@Qualifier("trackerServiceApiCounter") Counter errorCounter) {
        this.errorCounter = errorCounter;
    }

    public void processRequest() {
        try {
            // Simulate a task that might fail
            performTask();
        } catch (Exception ex) {
            errorCounter.increment(); // Increment counter on error
            log.error("Error occurred: {}", ex.getMessage());
        }
    }

    private void performTask() {
        throw new RuntimeException("Simulated exception");
    }
}
```

**Key Points**:

- Inject the counter using `@Qualifier`.
- Increment the counter in the `catch` block to record errors.

---

### Step 3: Measuring Execution Times

Timers are used to measure how long specific methods or blocks of code take to execute.

#### Declarative Timing with `@Timed`

```java
import io.micrometer.core.annotation.Timed;
import org.springframework.stereotype.Service;

@Service
public class TimedService {

    @Timed(value = "timed_method_execution", description = "Execution time of timedMethod")
    public void timedMethod() throws InterruptedException {
        // Simulate processing
        Thread.sleep(500);
    }
}
```

#### Manual Timing with `Timer`

```java
import io.micrometer.core.instrument.Timer;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class ManualTimingService {

    private final Timer executionTimer;

    public ManualTimingService(@Qualifier("methodExecutionTimer") Timer executionTimer) {
        this.executionTimer = executionTimer;
    }

    public void processRequest() {
        executionTimer.record(() -> {
            try {
                performTask();
            } catch (Exception e) {
                // Handle exception
            }
        });
    }

    private void performTask() throws InterruptedException {
        Thread.sleep(300);
    }
}
```

#### Output Example for Execution Times

1. **`timed_method_execution`**:

   - Measures the execution time of a method annotated with `@Timed`.
   - Output example from `/actuator/metrics/timed_method_execution`:
     ```json
     {
       "name": "timed_method_execution",
       "description": "Execution time of timedMethod",
       "baseUnit": "seconds",
       "measurements": [
         { "statistic": "COUNT", "value": 5 },
         { "statistic": "TOTAL_TIME", "value": 2.5 },
         { "statistic": "MAX", "value": 0.6 }
       ],
       "availableTags": []
     }
     ```

2. **`method_execution_time`**:

   - Measures the execution time recorded manually.
   - Output example from `/actuator/metrics/method_execution_time`:
     ```json
     {
       "name": "method_execution_time",
       "description": "Timer for manually tracked method execution",
       "baseUnit": "seconds",
       "measurements": [
         { "statistic": "COUNT", "value": 10 },
         { "statistic": "TOTAL_TIME", "value": 5.0 },
         { "statistic": "MAX", "value": 0.7 }
       ],
       "availableTags": []
     }
     ```

---

### Step 4: Monitoring System Resource Usage

Micrometer provides built-in support to monitor JVM and system metrics, such as memory usage, garbage collection, thread pool sizes, etc.

Add the following dependencies for system metrics:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Enable system metrics in your application:

```java
import io.micrometer.core.instrument.binder.system.DiskSpaceMetrics;
import io.micrometer.core.instrument.binder.system.ProcessorMetrics;
import io.micrometer.core.instrument.Gauge;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.io.File;

@Configuration
public class SystemMetricsConfig {

    @Bean
    public DiskSpaceMetrics diskSpaceMetrics() {
        return new DiskSpaceMetrics(new File("."));
    }

    @Bean
    public ProcessorMetrics processorMetrics() {
        return new ProcessorMetrics();
    }

    @Bean
    public Gauge activeThreadGauge(MeterRegistry registry) {
        return Gauge.builder("jvm.threads.active", Thread::activeCount)
                    .description("Number of active threads in the JVM")
                    .register(registry);
    }

    @Bean
    public Gauge customHeapUsageGauge(MeterRegistry registry) {
        return Gauge.builder("jvm.memory.heap.usage", () -> Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory())
                    .description("Heap memory usage")
                    .baseUnit("bytes")
                    .register(registry);
    }
}
```

#### Output Examples

1. **`jvm.threads.active`**:

   - Tracks the number of active threads in the JVM.
   - Output example from `/actuator/metrics/jvm.threads.active`:
     ```json
     {
       "name": "jvm.threads.active",
       "description": "Number of active threads in the JVM",
       "baseUnit": null,
       "measurements": [
         { "statistic": "VALUE", "value": 15 }
       ],
       "availableTags": []
     }
     ```

2. **`jvm.memory.heap.usage`**:

   - Tracks the current heap memory usage in bytes.
   - Output example from `/actuator/metrics/jvm.memory.heap.usage`:
     ```json
     {
       "name": "jvm.memory.heap.usage",
       "description": "Heap memory usage",
       "baseUnit": "bytes",
       "measurements": [
         { "statistic": "VALUE", "value": 52428800 }
       ],
       "availableTags": []
     }
     ```

---

### Accessing Metrics via Swagger or Actuator

Spring Boot Actuator exposes metrics endpoints for monitoring:

1. Add Actuator dependency:

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   ```

2. Configure Actuator in `application.properties`:

   ```properties
   management.endpoints.web.exposure.include=metrics,health,info
   ```

3. Access metrics via `/actuator/metrics` endpoint:

   - Example: `http://localhost:8080/actuator/metrics/tracker_service_api_error_count`

---

### Summary

Micrometer provides a flexible and powerful way to monitor your application by tracking:

- **Error Occurrences**: Use `Counter` to monitor and alert on error counts.
- **Execution Times**: Use `Timer` or `@Timed` for performance insights.
- **System Resource Usage**: Leverage built-in metrics and custom gauges for JVM and system monitoring.

This setup ensures improved observability, better debugging, and proactive performance tuning, all of which can be visualized using tools like Prometheus, Grafana, or CloudWatch.
