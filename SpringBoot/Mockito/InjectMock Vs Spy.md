
# Mockito Annotations: `@Mock`, `@Spy`, and `@InjectMocks`

This document explains the Mockito annotations `@Mock`, `@Spy`, and `@InjectMocks` with clear examples, test cases, and log outputs.

## Annotations Overview

### 1. `@Mock`
- **Purpose**: Creates a mock object, replacing the actual dependency with a controlled version.
- **Use Case**: Useful for isolating the class under test by mocking its dependencies.

**Code Example:**
```java
import static org.mockito.Mockito.*;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

class UserServiceTest {

    @Mock
    private UserRepository userRepository; // Mocking the dependency

    private UserService userService; // Class under test

    @Test
    void testGetUserById() {
        MockitoAnnotations.openMocks(this);
        userService = new UserService(userRepository);

        // Log: Mocking repository behavior
        System.out.println("Mocking userRepository.findById()");

        when(userRepository.findById(1L)).thenReturn(new User(1L, "John"));

        User user = userService.getUserById(1L);

        // Log: Verifying interactions
        System.out.println("Verifying userRepository.findById(1L)");
        verify(userRepository).findById(1L);

        System.out.println("Asserting user name is John");
        assertEquals("John", user.getName());
    }
}
```

**Expected Output:**
```
Mocking userRepository.findById()
Verifying userRepository.findById(1L)
Asserting user name is John
```

---

### 2. `@Spy`
- **Purpose**: Partially mocks an object, allowing you to call real methods unless they are explicitly stubbed.
- **Use Case**: Useful when you want to test specific methods of a class while spying on the rest.

**Code Example:**
```java
import static org.mockito.Mockito.*;
import org.junit.jupiter.api.Test;
import org.mockito.Spy;
import org.mockito.MockitoAnnotations;

class CalculatorTest {

    @Spy
    private Calculator calculator = new Calculator(); // Partially mocked object

    @Test
    void testAddAndMultiply() {
        MockitoAnnotations.openMocks(this);

        // Log: Stubbing add method
        System.out.println("Stubbing calculator.add()");
        doReturn(10).when(calculator).add(3, 7);

        int result = calculator.addAndMultiply(3, 7, 2);

        // Log: Verifying interactions
        System.out.println("Verifying calculator.add(3, 7)");
        verify(calculator).add(3, 7);

        System.out.println("Verifying calculator.multiply(10, 2)");
        verify(calculator).multiply(10, 2);

        System.out.println("Asserting result is 20");
        assertEquals(20, result); // 10 (stubbed add) * 2
    }
}

class Calculator {
    public int add(int a, int b) { return a + b; }
    public int multiply(int a, int b) { return a * b; }
    public int addAndMultiply(int a, int b, int c) {
        return multiply(add(a, b), c);
    }
}
```

**Expected Output:**
```
Stubbing calculator.add()
Verifying calculator.add(3, 7)
Verifying calculator.multiply(10, 2)
Asserting result is 20
```

---

### 3. `@InjectMocks`
- **Purpose**: Automatically injects mocks and spies into the class under test.
- **Use Case**: Simplifies the setup when a class has multiple dependencies.

**Code Example:**
```java
import static org.mockito.Mockito.*;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

class OrderServiceTest {

    @Mock
    private PaymentService paymentService; // Mocked dependency

    @Mock
    private InventoryService inventoryService; // Mocked dependency

    @InjectMocks
    private OrderService orderService; // Class under test with injected mocks

    @Test
    void testPlaceOrder() {
        MockitoAnnotations.openMocks(this);

        // Log: Mocking dependencies
        System.out.println("Mocking paymentService and inventoryService");

        when(paymentService.processPayment(any())).thenReturn(true);
        when(inventoryService.checkStock(any())).thenReturn(true);

        boolean result = orderService.placeOrder(new Order());

        // Log: Verifying interactions
        System.out.println("Verifying paymentService.processPayment()");
        verify(paymentService).processPayment(any());

        System.out.println("Verifying inventoryService.checkStock()");
        verify(inventoryService).checkStock(any());

        System.out.println("Asserting order placement result is true");
        assertTrue(result);
    }
}

class OrderService {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;

    public OrderService(PaymentService paymentService, InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }

    public boolean placeOrder(Order order) {
        return paymentService.processPayment(order) && inventoryService.checkStock(order);
    }
}
```

**Expected Output:**
```
Mocking paymentService and inventoryService
Verifying paymentService.processPayment()
Verifying inventoryService.checkStock()
Asserting order placement result is true
```

---

## Summary of Annotations:
- **`@Mock`**: Replaces dependencies with fake implementations.
- **`@Spy`**: Partially mocks an object, using real methods unless stubbed.
- **`@InjectMocks`**: Automatically injects mocks and spies into the class under test.


