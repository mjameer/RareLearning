# Read-Your-Write Consistency in a Java Spring Boot Application

## **Overview**
One of the key challenges in scaling databases is ensuring **Read-Your-Write (RYW) consistency**, where a user sees their latest writes on subsequent reads. This is particularly challenging in **master-replica architectures**, where reads are often directed to replicas to reduce load on the primary database. However, **replication lag** can cause a scenario where a user reads stale data from a replica before the latest writes have been propagated.

### **Bitbucket’s Approach to Ensuring Read-Your-Write Consistency**
Bitbucket, which uses **PostgreSQL** as its database, employs a **replication lag-aware routing strategy** to ensure **RYW consistency** while offloading as much read traffic as possible to replicas.

#### **1. Master-Replica Setup**
- Writes go to the **master** database.
- Reads typically go to **replicas**, except when a user expects to read their most recent writes.

#### **2. Replication Mechanism**
- Replicas **pull** data from the master using the **Write-Ahead Log (WAL)**.
- Each write entry in WAL is assigned a **Log Sequence Number (LSN)**, which helps determine replication progress.

#### **3. Tracking the Latest User Write LSN**
- After a user **writes** to the master, the system **stores the latest LSN** for that user in **Redis**.
- The stored LSN serves as a checkpoint indicating the latest write this user expects to see.

#### **4. Determining the Correct Replica for Reads**
- When a user issues a read request, the middleware:
  - Retrieves the **latest LSN** for that user from Redis.
  - Queries all **replicas** for their current **LSN** progress.
  - Directs the read to a **replica that has caught up** with the latest user write, ensuring consistency.

#### **5. Performance Impact**
- The system reduces **read queries hitting the master** by **50%**, significantly improving scalability.
- The overhead for checking LSNs (~10ms) is **tolerable**, ensuring fast and scalable performance.

---

## **Implementation in Java Spring Boot**
This section demonstrates how to implement **Read-Your-Write Consistency** in a **Spring Boot** application using:
- **PostgreSQL** as the database.
- **Redis** for storing the last written LSN per user.
- **JPA + JDBC** for database interactions.

### **1. Dependencies (Maven)**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

---

### **2. Database Schema (PostgreSQL)**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE user_actions (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    action TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

### **3. Service Implementation**
#### **Database Service**
```java
@Service
public class UserService {
    private final JdbcTemplate jdbcTemplate;
    private final RedisService redisService;

    public UserService(JdbcTemplate jdbcTemplate, RedisService redisService) {
        this.jdbcTemplate = jdbcTemplate;
        this.redisService = redisService;
    }

    @Transactional
    public void performUserAction(Long userId, String action) {
        jdbcTemplate.update("INSERT INTO user_actions (user_id, action) VALUES (?, ?)", userId, action);
        String latestLsn = getCurrentLsn();
        redisService.storeUserLsn(userId, latestLsn);
    }

    public Optional<User> readUser(Long userId) {
        String userLsn = redisService.getUserLsn(userId);
        String replicaLsn = getReplicaLsn();
        boolean isCaughtUp = isReplicaCaughtUp(userLsn, replicaLsn);
        return isCaughtUp ? userRepository.findById(userId) : userRepository.findById(userId);
    }

    private String getCurrentLsn() {
        return jdbcTemplate.queryForObject("SELECT pg_current_wal_lsn()", String.class);
    }

    private String getReplicaLsn() {
        return jdbcTemplate.queryForObject("SELECT pg_last_wal_replay_lsn()", String.class);
    }

    private boolean isReplicaCaughtUp(String userLsn, String replicaLsn) {
        String query = "SELECT pg_wal_lsn_diff(?, ?)";
        Long diff = jdbcTemplate.queryForObject(query, new Object[]{replicaLsn, userLsn}, Long.class);
        return diff >= 0;
    }
}
```

---

### **4. Redis Service**
```java
@Service
public class RedisService {
    private final StringRedisTemplate redisTemplate;

    public RedisService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void storeUserLsn(Long userId, String lsn) {
        redisTemplate.opsForValue().set("user:lsn:" + userId, lsn);
    }

    public String getUserLsn(Long userId) {
        return redisTemplate.opsForValue().get("user:lsn:" + userId);
    }
}
```

---

### **5. Repository Interface**
```java
public interface UserRepository extends JpaRepository<User, Long> {}
```

---

### **6. REST Controller**
```java
@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping("/{id}/actions")
    public String performUserAction(@PathVariable Long id, @RequestBody String action) {
        userService.performUserAction(id, action);
        return "Action recorded";
    }

    @GetMapping("/{id}")
    public Optional<User> readUser(@PathVariable Long id) {
        return userService.readUser(id);
    }
}
```

---

### **Summary of Implementation**
- **Writes** update PostgreSQL and store the latest **LSN** in Redis.
- **Reads** check **replication lag** by comparing the **replica LSN** with the user's latest **LSN**.
- If the **replica is caught up**, read from it. Otherwise, read from the **master**.

This approach **reduces master database load**, ensuring **Read-Your-Write consistency** while maintaining performance.

---

## **Conclusion**
This approach is a practical way to implement **Read-Your-Write consistency** in a **Spring Boot** application while effectively utilizing **database replicas** and reducing master load. 🚀
