# Spring Transaction Isolation Levels

This tutorial explains the **transaction isolation levels** in Spring and their role in ensuring data consistency across transactions in a real-world application. It provides theoretical insights along with practical examples for each isolation level.

---

## What is Transaction Isolation?

Transaction isolation determines the **visibility of changes** made by one transaction to others. It helps maintain **data consistency** in scenarios where multiple transactions interact with the same data.

---

## Isolation Levels in Spring

### 1. **Default**
- Relies on the database's default isolation level (e.g., `Repeatable Read` in MySQL).
- Can be overridden using a query or configuration.

---

### 2. **Read Uncommitted**
- **Allows Dirty Reads**: Transactions can see **uncommitted changes** made by others.
- **Use Case**:
  - Example: Movie ticket booking where tickets appear unavailable due to a pending transaction.
- **Risk**: High chance of **data inconsistency**.

---

### 3. **Read Committed**
- **Prevents Dirty Reads**: Transactions can only see **committed changes**.
- **Use Case**:
  - Example: Stock values are always fetched as the **last committed value**.
- **Benefit**: Ensures consistency for committed data.

---

### 4. **Repeatable Read**
- **Prevents Dirty Reads and Non-Repeatable Reads**:
  - A transaction always sees **consistent data** even if another transaction modifies the data.
- **Allows Phantom Reads**: New rows added by other transactions may be visible.
- **Use Case**:
  - Example: Stock values remain consistent before and after updates during a transaction.

---

### 5. **Serializable**
- Ensures transactions run **sequentially** (no concurrent execution).
- **Prevents Dirty Reads, Non-Repeatable Reads, and Phantom Reads**.
- **Use Case**:
  - Example: Movie ticket booking ensures complete consistency.
- **Trade-off**: May cause performance issues due to the locking mechanism.

---

## Practical Implementation

- Isolation levels can be set using the `@Transactional` annotation in Spring.
- Example:
  ```java
  @Transactional(isolation = Isolation.READ_COMMITTED)
  public void updateStock() {
      // Your code here
  }
```


<img width="782" alt="image" src="https://github.com/user-attachments/assets/c420ea9a-80a6-4e1b-93b9-7e86220114b3" />




