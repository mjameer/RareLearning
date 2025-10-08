# Microservices Anti-Patterns: From Distributed Monolith to True Microservices

This document outlines the six most common mistakes developers make when building microservices, which often results in a tightly coupled **Distributed Monolith**. It provides the fix for each anti-pattern, with relevance to modern backend stacks like Java/Spring Boot.

---

## 1. Shared Database

| Anti-Pattern (Mistake) | The Impact (Why it Fails) | The Fix (True Microservice) |
| :--- | :--- | :--- |
| **Shared Database:** Services connect to the same physical database or schema (e.g., `OrderService` querying `Customer` tables directly). | Creates **tight coupling**. A schema change in one service breaks others. Violates the principle of a microservice being a self-contained unit. | **Database per Service:** Each microservice must own its data. Data sharing must occur only via well-defined **REST APIs** (synchronous) or **Asynchronous Messaging** (e.g., Kafka, RabbitMQ). |

## 2. Excessive Synchronous Communication

| Anti-Pattern (Mistake) | The Impact (Why it Fails) | The Fix (True Microservice) |
| :--- | :--- | :--- |
| **Synchronous Chains:** Over-reliance on blocking HTTP calls (Service A -> B -> C). | Creates a **Service Dependency Chain**. A failure or slowdown in Service C blocks the entire upstream chain, resulting in a system-wide failure. | **Implement Resilience:** <br> 1. Use **Asynchronous Messaging** for non-critical or eventual consistency tasks (e.g., sending notifications). <br> 2. Implement the **Circuit Breaker** pattern (e.g., Spring Cloud Resilience4j) for critical sync calls to prevent infinite blocking and enable immediate **fallbacks**. |

## 3. Sharing DTOs and Contracts

| Anti-Pattern (Mistake) | The Impact (Why it Fails) | The Fix (True Microservice) |
| :--- | :--- | :--- |
| **Shared DTOs/Utilities:** Creating a common-shared library (JAR) for business contracts (DTOs like `UserDTO`) or utility code. | Introduces **tight coupling** and prevents **independent versioning**. A change to the shared DTO by one team causes compilation errors for all other services using it. | **Internal Contracts Only:** <br> **Duplicate DTOs** across services. While this increases boilerplate, it ensures each service can change its API contract without affecting others. <br> For true technical reuse, use a **Utility JAR** dependency, but **never** for business contracts (DTOs). |

## 4. Centralized Security (SPOF)

| Anti-Pattern (Mistake) | The Impact (Why it Fails) | The Fix (True Microservice) |
| :--- | :--- | :--- |
| **Centralized Auth Service:** Requiring every microservice to call a single `AuthService` for authentication/authorization on every request. | Creates a **Single Point of Failure (SPOF)**. If the Auth service is down, the entire application shuts down. | **Decentralize Token Validation:** Use stateless **JWT (JSON Web Tokens)**. After token generation, each microservice should perform **offline validation** (checking the signature and claims) independently without needing to call the central Auth service for every request. |

## 5. Lack of Bounded Context (Technical Splitting)

| Anti-Pattern (Mistake) | The Impact (Why it Fails) | The Fix (True Microservice) |
| :--- | :--- | :--- |
| **Technical Service Splitting:** Creating services based on technical functions instead of business domains (e.g., a centralized `NotificationService` that handles all emails for all other services). | Violates **Domain-Driven Design (DDD)**. Leads to cross-domain dependencies and creates another SPOF for a critical business function. | **Domain-Driven Design (DDD):** Services must be built around **business capabilities** (e.g., `OrderService`, `PaymentService`, `ShipmentService`). The `OrderService` must be capable of sending its own order confirmation email (using an internal utility JAR, not a separate microservice). |

## 6. Coupled Deployment and Scaling

| Anti-Pattern (Mistake) | The Impact (Why it Fails) | The Fix (True Microservice) |
| :--- | :--- | :--- |
| **Coupled Deployment:** Using a single CI/CD pipeline, the same load balancer, or scaling all services together. | Negates the core benefit of microservices: **independent evolution** and **resource efficiency**. An under-utilized service is unnecessarily scaled alongside a high-traffic service. | **Independent Pipelines & Scaling:** Every microservice requires its own dedicated: <br> 1. **CI/CD Pipeline** (e.g., Jenkins/ArgoCD). <br> 2. **Versioning** scheme. <br> 3. **Scaling Strategy** (e.g., Kubernetes HPA), scaled only based on its unique traffic requirements. |

---

> **The Goal:** Build true microservices that can scale, fail gracefully, and evolve **independently**. If you must rebuild or redeploy Service A when changing Service B, you have a Distributed Monolith.
