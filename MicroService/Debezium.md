

## 1. Overview of Change Data Capture (CDC)


<img width="690" height="373" alt="Screen Shot 2025-11-28 at 11 10 53 PM" src="https://github.com/user-attachments/assets/0009aab6-8e11-46e3-80aa-03fa37b1b8f9" />


**Change Data Capture (CDC)** is a pattern and technology used to identify, capture, and track data changes in a database and deliver those changes as a stream of events.

* **Core Process:** The general CDC architecture involves a **Source Database** (PostgreSQL), a **CDC Tool** (**Debezium**), a **Message Broker** (**Kafka**), and various **Consumers** (Microservices or other data systems).
* **Mechanism:** CDC tools like Debezium avoid resource-intensive polling by reading the database's internal transaction log (the **Write-Ahead Log, or WAL** in PostgreSQL). This allows for low-latency, real-time capture of changes.
* **Event Format:** When a change occurs (Insert, Update, or Delete), Debezium converts the change into a structured **JSON message** and sends it to a Kafka topic. This message typically contains:
    * The **operation type** (`op`: `c` for create/insert, `u` for update, `d` for delete).
    * The **`before`** state of the row (for updates/deletes).
    * The **`after`** state of the row (for inserts/updates).

***

## 2. Core Technology Stack: PostgreSQL, Debezium, and Kafka

### PostgreSQL Configuration (Source Database)

To enable CDC, PostgreSQL requires specific configuration changes and permissions:

* **WAL Level:** The `wal_level` parameter in the PostgreSQL configuration must be set to **'logical'**. This is crucial as the **WAL** is the source of all change events.
* **Replication User & Slot:**
    * The database user connecting with Debezium must have **replication permissions**.
    * A **logical replication slot** is created for Debezium (e.g., `debezium_slot` using the `pgoutput` plugin). This slot acts as a **bookmark** to ensure Debezium knows where to resume reading the WAL if it stops.

### Debezium and Kafka Connect

* **Kafka Connect:** This is a framework for integrating Kafka with other systems. It runs connectors that either stream data **into** Kafka (Source Connectors) or **out of** Kafka (Sink Connectors).
* **Debezium:** Debezium is a suite of **Source Connectors** (like the PostgreSQL Connector) that run inside the Kafka Connect framework.
* **Deployment:** The Debezium Connector is **registered** with the Kafka Connect service (typically running in a separate container) and establishes a connection to the PostgreSQL database. Once registered, it starts reading the WAL and publishing change events to Kafka.

<img width="661" height="338" alt="Screen Shot 2025-11-28 at 11 12 45 PM" src="https://github.com/user-attachments/assets/17b12a2f-7c65-4691-b882-b92047e4bd37" />


## 3. Consuming Change Events (Microservices Perspective)

In a microservices architecture, a consumer service connects to Kafka to read and process the change events in real-time.

* **Consumer Setup:** A typical consumer (often a **Spring Boot** application) needs to be configured with the Kafka bootstrap servers and a consumer group ID.
* **Event Processing:** The consumer uses a **`@KafkaListener`** or similar mechanism to poll the designated Kafka topic. The core logic involves:
    1.  **Deserialization:** Converting the raw Kafka message back into a usable data structure (like a JSON object).
    2.  **Operation Check:** Examining the `op` (operation type) field in the event payload to determine if the change was an Insert, Update, or Delete.
    3.  **Business Logic:** Applying the necessary business logic based on the type of change and the data in the `before` or `after` fields.

***

## 4. CDC in Microservices Architecture (Pros & Cons)

CDC is presented as a strong solution for solving the **distributed data problem** in microservices.

### Benefits of Using CDC
| Benefit | Description |
| :--- | :--- |
| **Loose Coupling** | Services are decoupled because they do not share a database directly or communicate synchronously via APIs. |
| **Polyglot Persistence** | Different microservices can use different database technologies (e.g., one uses Postgres, another uses MongoDB) and still share data through the unified event stream. |
| **Real-Time Views** | Allows for the creation and maintenance of **real-time materialized views** or search indexes by subscribing to the change events. |
| **Asynchronous Communication** | Replaces synchronous API calls with an **event-driven architecture**, improving resilience and performance. |

### Drawbacks of Using CDC
| Drawback | Description |
| :--- | :--- |
| **Operational Overhead** | Requires maintenance and monitoring of new components, namely the **Kafka event platform** and **Kafka Connect/Debezium**. |
| **Complexity** | Introduces complexity in handling **schema evolution** (changes to table structure) and ensuring **transactional consistency** (e.g., using the **Transactional Outbox Pattern** for reliability). |
| **Not a Silver Bullet** | It's a powerful pattern, but not suitable for every scenario; it's best for systems requiring **loose coupling** and **real-time data synchronization**. |

***

## 5. GitHub Repository Summary

The **`irtiza07/postgres_debezium_cdc`** GitHub repository serves as a **minimal, working example** for setting up the CDC pipeline. Its primary purpose is to provide a practical, ready-to-run environment for users to quickly experiment with the concepts covered in the videos.

* **Technology:** Uses **Docker Compose** to deploy the entire stack: PostgreSQL, Zookeeper, Kafka, and Kafka Connect with the Debezium PostgreSQL Connector.
* **Purpose:** To demonstrate how Debezium is configured to read from PostgreSQL's WAL and publish the change events to Kafka topics, allowing users to test the data flow immediately.

***

## References

1.  **PostgreSQL Change Data Capture with Debezium, Kafka, and Microservices**
    * [https://www.youtube.com/watch?v=fwlAFJxRcok](https://www.youtube.com/watch?v=fwlAFJxRcok)
2.  **Change Data Capture (CDC) - The Best Practice for Microservices?**
    * [https://www.youtube.com/watch?v=Uoas9E8Luo8](https://www.youtube.com/watch?v=Uoas9E8Luo8)
3.  **Change Data Capture with Kafka Connect and Debezium**
    * [https://www.youtube.com/watch?v=6VbRlQ0rL3I](https://www.youtube.com/watch?v=6VbRlQ0rL3I)
4.  **CDC with Debezium and PostgreSQL | Setup Debezium for PostgreSQL (Part 3)**
    * [https://www.youtube.com/watch?v=YZRHqRznO-o&list=PLPnOUbNhPEkzd9-VVX7IjnBqsg9O6axOI&index=3](https://www.youtube.com/watch?v=YZRHqRznO-o&list=PLPnOUbNhPEkzd9-VVX7IjnBqsg9O6axOI&index=3)
5.  **Consume Debezium Events with a Simple Kafka Consumer (Part 19)**
    * [https://www.youtube.com/watch?v=0d_I2aQm4LE&list=PLPnOUbNhPEkzd9-VVX7IjnBqsg9O6axOI&index=19](https://www.youtube.com/watch?v=0d_I2aQm4LE&list=PLPnOUbNhPEkzd9-VVX7IjnBqsg9O6axOI&index=19)
6.  **GitHub Repository: irtiza07/postgres\_debezium\_cdc**
    * [https://github.com/irtiza07/postgres_debezium_cdc](https://github.com/irtiza07/postgres_debezium_cdc)



## Missed Schema Registry

That's an excellent point! The **Schema Registry** is indeed a critical component for building robust, production-grade Change Data Capture (CDC) pipelines with Debezium and Kafka. My initial summary focused on the core data flow but omitted this crucial element of **data governance** and **compatibility**.

Here is a summary of the role of the Schema Registry in this architecture:

## 🧭 The Role of the Schema Registry

The **Schema Registry** acts as a centralized repository for managing, validating, and versioning the **schemas** used by Kafka messages. It is essential when using structured serialization formats like **Apache Avro**, **JSON Schema**, or **Protobuf**, which are far more efficient and safer than plain JSON.

### 1. The Core Problem It Solves

Kafka messages are fundamentally just byte arrays. Without a defined contract (a schema), producers (like Debezium) and consumers cannot be guaranteed to understand the data. As your database schema **evolves** (e.g., adding a new column, changing a data type), downstream consumers can easily break if they are suddenly sent messages in a new, incompatible format.

The Schema Registry solves this by maintaining a **versioned history** of schemas and enforcing **compatibility rules** [1.2, 3.5].

### 2. How Debezium and Kafka Use It

* **Serialization and Compression:** When Debezium (running inside Kafka Connect) captures a change from PostgreSQL, it uses a **Converter** (like `AvroConverter`) to serialize the data. Instead of sending the full schema with every message, the producer (Debezium) registers the schema with the Registry and receives a unique **Schema ID** [1.3].
    * The Debezium message payload is then sent to Kafka with only the **Schema ID** prepended to the data bytes [1.2]. This significantly optimizes the payload size and transfer speed.
* **Consumer Contract:** When a consumer microservice receives the message, it extracts the Schema ID, fetches the corresponding schema from the Registry, and uses that schema to accurately **deserialize** the data [1.2].
* ****

### 3. Key Benefits for CDC Pipelines

| Feature | Description |
| :--- | :--- |
| **Schema Evolution** | The Registry enforces **compatibility rules** (e.g., `BACKWARD`, `FORWARD`, `FULL`) when a new version of a schema is registered (when a table changes) [3.1, 4.1]. This prevents incompatible changes from breaking downstream applications and allows for a controlled, safe evolution of the database structure. |
| **Data Governance** | It serves as the single source of truth for all data contracts, improving **data quality** and providing visibility into data lineage [1.1]. |
| **Data Validation** | Producers validate data against the registered schema before sending it, catching potential errors early in the pipeline [1.2]. |
| **Loose Coupling** | By providing an explicit contract, it further decouples services, as consumers know exactly what structure to expect, even if the database changes in a backward-compatible way [1.3]. |

---

## Git
https://github.com/irtiza07/postgres_debezium_cdc/tree/master


