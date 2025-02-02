
<img width="709" alt="Screen Shot 2024-11-24 at 9 32 15 PM" src="https://github.com/user-attachments/assets/22580f8f-d646-4826-8ba4-901e7195df02">



<img width="886" alt="Screen Shot 2024-11-24 at 9 32 00 PM" src="https://github.com/user-attachments/assets/a019d0df-0713-4448-ad0d-f240b58a0d5b">

# Hidden Gems in Lombok

Lombok provides powerful annotations to simplify Java development. Here are some lesser-known but incredibly useful Lombok annotations:

## Annotations Overview

### `@SneakyThrows`
- **Purpose**: Suppresses checked exceptions without `try-catch`.
- **Example**:
  ```java
  @SneakyThrows
  public void readFile() {
      Files.readAllBytes(Paths.get("file.txt"));
  }
  ```

### `@Cleanup`
- **Purpose**: Ensures resources are properly closed.
- **Example**:
  ```java
  @Cleanup
  InputStream inputStream = new FileInputStream("file.txt");
  ```

### `@Synchronized`
- **Purpose**: Thread-safe methods using private locks.
- **Example**:
  ```java
  @Synchronized
  public void safeMethod() {
      // Thread-safe code
  }
  ```

### `@Slf4j`
- **Purpose**: Adds an SLF4J logger to your class.
- **Example**:
  ```java
  @Slf4j
  public class MyClass {
      public void logSomething() {
          log.info("Logging with Lombok!");
      }
  }
  ```

### `@Delegate`
- **Purpose**: Delegates methods to another object.
- **Example**:
  ```java
  @Delegate
  private final List<String> myList = new ArrayList<>();
  ```

### `@SuperBuilder`
- **Purpose**: Builds complex objects with inheritance support.
- **Example**:
  ```java
  @SuperBuilder
  public class Parent {
      private String name;
  }

  @SuperBuilder
  public class Child extends Parent {
      private int age;
  }
  ```

### `@UtilityClass`
- **Purpose**: Marks a class as a utility class with static methods.
- **Example**:
  ```java
  @UtilityClass
  public class Utils {
      public void doSomething() {
          // Static utility method
      }
  }
  ```

### `@ExtensionMethod`
- **Purpose**: Enables extension methods for existing classes.
- **Example**:
  ```java
  @ExtensionMethod(Math.class)
  public class MyClass {
      public void example() {
          double sqrt = 4.0.sqrt(); // Calls Math.sqrt(4.0)
      }
  }
  ```


### Lombok: Generate Modern Getters and Setters

#### Question
Is there a way for Lombok to generate modern-style getters and setters, such as:
- Getter: `name()` instead of `getName()`
- Setter: `name(String name)` instead of `setName(String name)`

#### Solution
Yes! Lombok supports this using `@Accessors(fluent = true)`. This allows you to use Java 14+ record-style accessors.

```java
import lombok.Getter;
import lombok.Setter;
import lombok.experimental.Accessors;

@Getter
@Setter
@Accessors(fluent = true) // Enables name() and name(String name)
public class Person {
    private String name;
}
```

#### **Generated Methods**
- **Getter:** `public String name()`
- **Setter:** `public void name(String name)`


#### **Explanation**
- `@Accessors(fluent = true)` removes the `get` and `set` prefixes, making accessors look like Java records or Kotlin-style functions.

#### **Enforce This Style Globally?**
You can configure this in your Lombok settings (`lombok.config`) to enforce it across your project:
```properties
lombok.accessors.fluent = true
```
