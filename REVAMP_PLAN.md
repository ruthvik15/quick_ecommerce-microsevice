# Architecture Design Document: Quick Ecommerce Revamp

## 1. High-Level Overview
A distributed microservices system designed for high availability and strict contract enforcement. The system mimics a real-world fintech/retail environment by prioritizing data consistency (Sagas) and fault tolerance (Circuit Breakers) over simple CRUD operations.

- **Architecture Style**: Microservices (Single-Node Cluster).
- **Orchestration**: Docker Compose.
- **Communication**: gRPC (Internal), REST (External).
- **Consistency Model**: Eventual Consistency (via Sagas).

## 2. The Tech Stack & Rationale

| Component | Choice | Why? |
| :--- | :--- | :--- |
| **Language** | Go (Golang) | High concurrency, low memory footprint, and native gRPC support. |
| **Framework** | Clean Arch | Separation of concerns (Handler, Service, Repo) for testability. |
| **Communication** | gRPC + Protobuf | Strict schema validation prevents integration bugs; faster than JSON/HTTP. |
| **Dependency Injection** | Google Wire | Compile-time dependency injection to manage initialization complexity. |
| **Resilience** | Hystrix-Go | Circuit Breaker pattern to prevent cascading failures when a service is down. |
| **Database** | Postgres | ACID compliance for Order/Payment data. |
| **Cache** | Redis | Geospatial indexing for Rider tracking and caching product catalog. |

## 3. Core Design Patterns

### A. The "Orchestrated Saga" (Data Consistency)
An Orchestrator (Order Service) manages the lifecycle of a request to handle distributed transactions.

**The Workflow:**
1.  **Order Service**: Creates `PENDING` order.
2.  **Order Service**: Calls `Inventory.DeductStock()`.
3.  **Order Service**: Calls `Payment.Charge()`.
4.  **Order Service**: Finalizes order (`CONFIRMED`).

**Failure Handling (The Rollback):**
*   If `Payment.Charge()` fails:
    1.  Order Service catches the error.
    2.  Order Service immediately calls `Inventory.AddStock()` (The Compensating Transaction).
    3.  Order Service marks order as `FAILED`.

### B. The "Circuit Breaker" (Fault Tolerance)
Uses `hystrix-go` to wrap gRPC calls to prevent cascading failures.

*   **Closed State (Normal)**: Traffic flows to Inventory.
*   **Open State (Failure)**: If Inventory fails 5 times in a row, the Breaker "Trips" (Opens).
*   **Fast Fail**: Subsequent requests fail instantly (0ms) with error "Circuit Open".
*   **Half-Open**: After 5 seconds, 1 request is allowed through to test if Inventory is back up.

## 4. Failure Scenarios & Recovery Strategy

| Scenario | What Happens? | Our Solution |
| :--- | :--- | :--- |
| **Service Crash** | Order Service cannot reach Inventory. | Circuit Breaker opens immediately. Gateway returns "Service Unavailable". |
| **Network Blip** | "Release Stock" fails during rollback. | **Idempotent Retry**: Order Service retries the release call 3 times. |
| **Payment Fail** | User has no money. | **Saga Rollback**: We return the item to inventory. |
| **Zombie Data** | Rollback fails 100% (Inventory DB deleted). | **Dead Letter Queue**: Log failure for manual reconciliation. |

## 5. Tradeoffs (Pros & Cons)

### Pros
*   **Robustness**: The system is designed to fail gracefully. Circuit breakers prevent one down service from taking down the whole system.
*   **Data Integrity**: Sagas ensure that even in distributed scenarios, data remains consistent eventually.
*   **Scalability**: Microservices allow specific components (like Inventory) to be scaled independently if load increases.
*   **Strict Contracts**: gRPC/Protobuf ensures that service communication is type-safe and performant.

### Cons
*   **Complexity**: Significantly more complex to set up and debug than a monolithic REST API. Tracing request flows across services requires good observability.
*   **Development Speed**: Adding a field requires updating Proto definitions, regenerating code, and potentially updating multiple services.
*   **Latency**: Service-to-service network hops add latency compared to in-memory function calls in a monolith.
*   **Consistency Lag**: The system is eventually consistent, not immediately consistent. The UI must handle states like `PENDING` gracefully.

## 6. Plan of Action

### Phase 1: Infrastructure & Shared Contracts
1.  **Repo Setup**: Initialize Git, Go modules, and directory structure.
2.  **Proto Definitions**: Define `order.proto` and `inventory.proto` with gRPC service definitions.
3.  **Docker Environment**: Create `docker-compose.yml` for Postgres and Redis.

### Phase 2: Internal Services (The Core)
1.  **Inventory Service**: Implement `DeductStock` and `AddStock` (compensation) with Postgres storage.
2.  **Payment Service**: Implement a mock `Charge` method.
3.  **Order Service (Orchestrator)**: Implement the Saga logic—create order, call inventory, call payment, handle rollbacks.

### Phase 3: Resilience & Public API
1.  **Hystrix Integration**: Wrap gRPC clients in the Order Service with Circuit Breakers.
2.  **API Gateway**: Build a Gin-based HTTP server to expose endpoints to the frontend/user.
3.  **Dependency Injection**: Use Google Wire to wire up all services and repositories.

### Phase 4: Verification
1.  **Unit Tests**: Test core logic (handlers/services) using mocks.
2.  **Integration Tests**: Run the full stack in Docker and simulate success/failure flows (e.g., payment failure triggers inventory rollback).
