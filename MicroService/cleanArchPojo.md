# Clean Architecture: Entities, DTOs, and POJOs in Spring Boot

This document outlines the best practices for structuring data models in a layered Spring Boot application to achieve **low coupling** and **separation of concerns** between the persistence (database) and presentation (API) layers.

---

## 🔑 Key Principles

The fundamental goal is to **never expose the database model (Entity) directly to the client**,  as Entities are designed for persistence, not for serialization. 
Overloading a single class to handle both roles muddies its purpose.

| Data Object | Primary Responsibility | Primary Layer | Core Rule |
| :--- | :--- | :--- | :--- |
| **Entity** | Database Persistence | Repository | **Tightly coupled to the DB schema.** Must be kept **private** to the persistence layer. |
| **DTO** | API Contract | Controller (API) | **Tightly coupled to the client contract.** Represents exactly what is sent/received over the network. |
| **POJO** | Business Logic | Service | **Independent of both DB and API.** Used for internal calculations and intermediate data structures. |

---

## 🏗️ Recommended Data Flow

To maintain a secure and decoupled architecture, objects must be converted between layers:

### 1. Request Flow (Client $\to$ Database)

1.  **Client $\to$ Controller:** Receives data as a **DTO** (e.g., `UserDTO`).
2.  **Service Layer:** Converts the incoming **DTO** into an **Entity**.
3.  **Repository Layer:** Persists the **Entity** to the database.

### 2. Response Flow (Database $\to$ Client)

1.  **Repository Layer:** Retrieves the data as an **Entity**.
2.  **Service Layer:** Converts the **Entity** into a **DTO**, often selectively removing sensitive fields like `hashedPassword` or `role`.
3.  **Controller $\to$ Client:** Returns the **DTO** as the API response.

---

## 🚫 Critical Mistakes to Avoid

| Mistake | Why It's Bad | Solution |
| :--- | :--- | :--- |
| **Exposing Entities** | **Security Leak:** Reveals internal fields (passwords, roles) and back-end encryption methods. **High Coupling:** Database schema changes break the client's API contract. | Use a **DTO** as the public API contract. Entity objects must not leave the Service layer boundary. |
| **Manual Mapping** | Writing boilerplate `.set()` and `.get()` code for conversions is time-consuming and error-prone. | Use a specialized mapping tool like **MapStruct** to automate Entity-to-DTO and DTO-to-Entity conversions. |
| **Confusing POJO & DTO** | Using a DTO for internal business logic (e.g., calculating a total cart value) blurs responsibility and can lead to unnecessary framework coupling on internal models. | Use generic, non-annotated **POJOs** for all intermediate business calculations and analysis models. |

**In short:** DTOs are for the **network** (client contract), Entities are for the **database** (persistence), and POJOs are for **everything else** (internal business logic).
