# Transaction Propagation Types in Spring Framework

## 1. **Propagation.REQUIRED**  
### Use Case:  
Default and most commonly used propagation type. Use when all related operations must succeed or fail as a single unit.  

### Example:
- **E-commerce Order Processing**: Placing an order involves:
  - Saving order details.
  - Updating inventory stock.
  - Processing payment.
  
  If any of these operations fail, the entire transaction should roll back to maintain consistency.

---

## 2. **Propagation.REQUIRES_NEW**  
### Use Case:  
Use when an operation must execute in a new, independent transaction, regardless of the outer transaction's state.  

### Example:
- **Audit Logging**: Log failed order attempts to an audit table, even if the order transaction itself rolls back.

---

## 3. **Propagation.MANDATORY**  
### Use Case:  
Use when the method must always run within an existing transaction, and it doesn’t make sense to proceed without one.  

### Example:
- **Payment Validation**: Ensuring payment is valid and linked to an ongoing order placement transaction.
- **Inventory Check**: Validating stock levels before proceeding with an order.

---

## 4. **Propagation.NEVER**  
### Use Case:  
Use when a method must not run in a transaction and should explicitly fail if one exists.  

### Example:
- **Sending Notifications**: Sending a "Thank You" email after placing an order should not be part of the transaction to avoid rollback issues.
- **Cache Update**: Updating a product recommendation cache after an order is placed, independent of transactional concerns.

---

## 5. **Propagation.NOT_SUPPORTED**  
### Use Case:  
Use when the method should never run within a transaction but doesn't need to throw an error if one exists—it just suspends it.  

### Example:
- **Fetching Read-Only Reports**: Generating reports or recommendations during checkout without impacting the transaction.
- **Analytics**: Logging user behavior for analytics purposes, which doesn’t require a transaction.

---

## 6. **Propagation.SUPPORTS**  
### Use Case:  
Use when the method can optionally join an existing transaction but doesn't require one to run.  

### Example:
- **Fetching Customer Details**: Retrieving customer information during order placement. If the outer transaction exists, it participates; otherwise, it runs independently.
- **Fetching Product Reviews**: Showing product reviews while adding items to a cart.

---

## 7. **Propagation.NESTED**  
### Use Case:  
Use when you want to isolate a part of a transaction so it can roll back independently without affecting the main transaction.  

### Example:
- **Partial Rollback in Complex Operations**:
  - In a system, updating a profile involves two steps:
    1. Updating user account.
    2. Updating address table.
  - If updating user account fails, you want to roll back just that step without affecting the address update.

---

## Summary Table  

| **Propagation Type**  | **Use Case**                                                                                         |  
|------------------------|-----------------------------------------------------------------------------------------------------|  
| **REQUIRED**           | Group operations (e.g., order placement with inventory and payment updates).                        |  
| **REQUIRES_NEW**       | Audit logging.                                                           |  
| **MANDATORY**          | Payment or stock validation tied to an existing transaction.                                        |  
| **NEVER**              | Sending email or updating cache outside of transactions.                                            |  
| **NOT_SUPPORTED**      | Fetching read-only data (e.g., reports, recommendations).                                           |  
| **SUPPORTS**           | Optional participation in transactions (e.g., customer or product details retrieval).               |  
| **NESTED**             | Isolated rollback for child operations (e.g., updating user profile and address details ).                          |
