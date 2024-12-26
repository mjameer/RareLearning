
# Mockito Verify Example

This project demonstrates how to use Mockito's `verify` method to validate interactions between a class under test and its dependencies. 

## Purpose of Mockito `verify`
The `verify` method is used to:
1. Ensure specific methods on mock objects are called as expected.
2. Validate that the correct parameters are passed to these methods.
3. Test methods with no return value (void) by asserting on their side effects.

## Example Scenario

**Real-World Analogy**:  
A ticket booking system involves the following steps:
1. A user books a ticket.
2. The system:
   - Updates the database.
   - Sends a confirmation email to the user.

The method under test doesn’t return any value but interacts with dependent services. Using `verify`, we can ensure:
- The database update method is called.
- The email service sends a confirmation.

## Code Implementation

### Classes to Test
```java
// Collaborator 1: Database Service
public class DatabaseService {
    public void saveBooking(String userId, String bookingDetails) {
        // Save booking in the database
    }
}

// Collaborator 2: Email Service
public class EmailService {
    public void sendConfirmation(String userId) {
        // Send confirmation email
    }
}

// Class to Test: BookingManager
public class BookingManager {
    private final DatabaseService databaseService;
    private final EmailService emailService;

    public BookingManager(DatabaseService databaseService, EmailService emailService) {
        this.databaseService = databaseService;
        this.emailService = emailService;
    }

    public void bookTicket(String userId, String bookingDetails) {
        databaseService.saveBooking(userId, bookingDetails);  // Save booking
        emailService.sendConfirmation(userId);               // Notify user
    }
}
```

### Test Case with Mockito
```java
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;

public class BookingManagerTest {
    @Test
    public void testBookTicket() {
        // Arrange: Create mock collaborators
        DatabaseService databaseServiceMock = mock(DatabaseService.class);
        EmailService emailServiceMock = mock(EmailService.class);

        // Create the class under test with mocked dependencies
        BookingManager bookingManager = new BookingManager(databaseServiceMock, emailServiceMock);

        // Act: Call the method to test
        String userId = "user123";
        String bookingDetails = "Flight 123";
        bookingManager.bookTicket(userId, bookingDetails);

        // Assert: Verify interactions with mocks
        verify(databaseServiceMock).saveBooking(userId, bookingDetails);  // Ensure DB update
        verify(emailServiceMock).sendConfirmation(userId);               // Ensure email sent
    }
}
```

## Key Benefits of Using `verify`
1. **Interaction Validation**: Ensures that dependencies are called correctly.
2. **Parameter Checks**: Confirms that the correct arguments are passed.
3. **Improved Test Coverage**: Allows testing of void methods by asserting their side effects.
4. **Debugging**: Provides clarity when interactions or parameters are incorrect.

---
