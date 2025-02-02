
<img width="709" alt="Screen Shot 2024-11-24 at 9 32 15 PM" src="https://github.com/user-attachments/assets/22580f8f-d646-4826-8ba4-901e7195df02">



<img width="886" alt="Screen Shot 2024-11-24 at 9 32 00 PM" src="https://github.com/user-attachments/assets/a019d0df-0713-4448-ad0d-f240b58a0d5b">


# Lombok: Generate Modern Getters and Setters

## Question
Is there a way for Lombok to generate modern-style getters and setters, such as:
- Getter: `name()` instead of `getName()`
- Setter: `name(String name)` instead of `setName(String name)`

## Solution
Yes! Lombok supports this using `@Accessors(fluent = true)`. This allows you to use Java 14+ record-style accessors.

### **Usage**

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

### **Generated Methods**
- **Getter:** `public String name()`
- **Setter:** `public void name(String name)`

### **Example Usage**
```java
Person p = new Person();
p.name("Alice"); // Fluent setter
System.out.println(p.name()); // Fluent getter
```

### **Explanation**
- `@Accessors(fluent = true)` removes the `get` and `set` prefixes, making accessors look like Java records or Kotlin-style functions.

### **Enforce This Style Globally?**
You can configure this in your Lombok settings (`lombok.config`) to enforce it across your project:
```properties
lombok.accessors.fluent = true
```
