# Enterprise Order System 

A production-grade Spring Boot application combining **User, Order, Payment, Inventory, and Gateway** services into one deployable JAR.
Runs on **port 8080** with a single H2 in-memory database — no external dependencies required.

---

## Architecture

```
Single Spring Boot Application (port 8080)
├── User Domain       → /api/users/**
├── Order Domain      → /api/orders/**
├── Payment Domain    → /api/payments/**
├── Inventory Domain  → /api/inventory/**
└── Gateway Domain    → /api/gateway/**
```

All domains are logically separated into packages but share one datasource, one JVM, and one application context.

---

## Project Structure

```
order-system-monolith/
├── pom.xml
├── README.md
├── OrderSystemMonolith.postman_collection.json
└── src/main/
    ├── java/com/enterprise/ordersystem/
    │   ├── OrderSystemApplication.java         ← Main class
    │   ├── config/
    │   │   ├── KafkaConfig.java
    │   │   ├── KafkaEventBus.java              ← Shared event publisher
    │   │   ├── OpenApiConfig.java
    │   │   ├── GlobalExceptionHandler.java
    │   │   └── DataInitializer.java            ← Seeds all domains on startup
    │   ├── user/
    │   │   ├── model/User.java
    │   │   ├── dto/UserDto.java
    │   │   ├── repository/UserRepository.java
    │   │   ├── service/UserService.java
    │   │   ├── controller/UserController.java
    │   │   └── exception/UserNotFoundException.java
    │   ├── order/
    │   │   ├── model/Order.java
    │   │   ├── dto/OrderDto.java + OrderEvent.java
    │   │   ├── repository/OrderRepository.java
    │   │   ├── service/OrderService.java       ← Idempotency + retry
    │   │   ├── service/OrderSagaService.java   ← Saga choreography
    │   │   └── controller/OrderController.java
    │   ├── payment/
    │   │   ├── model/Payment.java
    │   │   ├── dto/PaymentDto.java
    │   │   ├── repository/PaymentRepository.java
    │   │   ├── service/PaymentService.java
    │   │   └── controller/PaymentController.java
    │   ├── inventory/
    │   │   ├── model/Inventory.java + InventoryReservation.java
    │   │   ├── dto/InventoryDto.java
    │   │   ├── repository/InventoryRepository.java + InventoryReservationRepository.java
    │   │   ├── service/InventoryService.java   ← Optimistic locking
    │   │   └── controller/InventoryController.java
    │   └── gateway/
    │       └── controller/GatewayController.java
    └── resources/
        └── application.yml
```

---

## Prerequisites

| Tool    | Version |
|---------|---------|
| Java    | 17+     |
| Maven   | 3.8+    |
| Eclipse | 2023+   |

---

## Eclipse Import (Step by Step)

1. **File → Import → Maven → Existing Maven Projects**
2. Browse to the `order-system-monolith/` folder
3. One project will appear — check it → **Finish**
4. Wait for Maven to resolve dependencies (~1–2 min first time)
5. Right-click `OrderSystemApplication.java` → **Run As → Spring Boot App**

> Eclipse tip: If you see red markers, right-click project → **Maven → Update Project** → OK.

---

## Running from Terminal

```bash
cd order-system-monolith
mvn clean install
mvn spring-boot:run
```

---

## All URLs (port 8080)

| URL | Purpose |
|-----|---------|
| http://localhost:8080/swagger-ui.html | Swagger UI — all APIs |
| http://localhost:8080/api-docs | OpenAPI JSON |
| http://localhost:8080/h2-console | H2 DB console |
| http://localhost:8080/actuator/health | Actuator health |
| http://localhost:8080/api/gateway/info | Route map + system info |

**H2 Console Settings:**
- JDBC URL: `jdbc:h2:mem:orderSystemDb`
- Username: `sa`
- Password: *(leave blank)*

---

## API Reference

### User API — `/api/users`

| Method | URL | Description |
|--------|-----|-------------|
| POST | `/api/users` | Create user |
| GET | `/api/users` | Get all users |
| GET | `/api/users/{id}` | Get user by ID |
| PUT | `/api/users/{id}` | Update name/phone |
| DELETE | `/api/users/{id}` | Delete user |

```json
POST /api/users
{
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "phone": "9876543210"
}
```

---

### Order API — `/api/orders`

| Method | URL | Description |
|--------|-----|-------------|
| POST | `/api/orders` | Create order (idempotent) |
| GET | `/api/orders` | Get all orders |
| GET | `/api/orders/{id}` | Get order by ID |
| GET | `/api/orders/user/{userId}` | Orders by user |
| PUT | `/api/orders/{id}/cancel` | Cancel order |

```json
POST /api/orders
{
  "userId": 1,
  "productId": "PROD-001",
  "quantity": 2,
  "totalAmount": 299.99,
  "idempotencyKey": "unique-key-abc123"
}
```

> Sending the same `idempotencyKey` twice returns the existing order — no duplicate created.

---

### Payment API — `/api/payments`

| Method | URL | Description |
|--------|-----|-------------|
| POST | `/api/payments` | Process payment |
| GET | `/api/payments` | Get all payments |
| GET | `/api/payments/{id}` | Get by ID |
| GET | `/api/payments/order/{orderId}` | Get by order ID |

```json
POST /api/payments
{
  "orderId": 5,
  "userId": 1,
  "amount": 299.99
}
```

---

### Inventory API — `/api/inventory`

| Method | URL | Description |
|--------|-----|-------------|
| POST | `/api/inventory` | Add product |
| GET | `/api/inventory` | Get all products |
| GET | `/api/inventory/{productId}` | Get product stock |
| POST | `/api/inventory/reserve` | Reserve stock |
| POST | `/api/inventory/{productId}/restock?quantity=N` | Add stock |
| DELETE | `/api/inventory/release/{orderId}` | Release reservation |

---

## Saga Flow (Automatic on Order Create)

When you create an order via `POST /api/orders`, the Saga runs **automatically**:

```
POST /api/orders
   │
   ▼
Order saved (status=CONFIRMED)
   │
   ▼ OrderSagaService.onOrderCreated()
   │
   ├── [kafka.enabled=false]  ──▶ Runs synchronously in-process:
   │       │
   │       ├── PaymentService.processPayment() → 90% SUCCESS
   │       │       ├── SUCCESS → inventoryService.reserveStock()
   │       │       │       ├── Stock OK  → Order status = COMPLETED ✓
   │       │       │       └── No stock  → Refund + SAGA_ROLLED_BACK ✗
   │       │       └── FAILED → Order status = SAGA_ROLLED_BACK ✗
   │       │
   └── [kafka.enabled=true]   ──▶ Publishes ORDER_CREATED to Kafka
           └── (Kafka consumers drive the same flow asynchronously)
```

---

## Seed Data (Auto-loaded on Startup)

**Users:**
| ID | Name | Email |
|----|------|-------|
| 1 | Alice Johnson | alice@example.com |
| 2 | Bob Smith | bob@example.com |
| 3 | Charlie Brown | charlie@example.com |
| 4 | Diana Prince | diana@example.com |

**Inventory:**
| Product ID | Name | Available |
|------------|------|-----------|
| PROD-001 | Laptop Pro 15 | 100 |
| PROD-002 | Wireless Mouse | 500 |
| PROD-003 | USB-C Hub | 250 |
| PROD-004 | Mechanical Keyboard | 150 |
| PROD-005 | 4K Monitor | 75 |

**Orders:** 3 seeded orders (COMPLETED, PAYMENT_PROCESSING, PENDING)

**Payments:** 2 seeded payments (SUCCESS, PROCESSING)

---

## Design Patterns

| Pattern | Where Used |
|---------|-----------|
| Saga Choreography | `OrderSagaService` — synced or via Kafka |
| Idempotency | `OrderService` — `idempotencyKey` unique constraint |
| Optimistic Locking | `Inventory` entity — `@Version` field |
| Retry + Exponential Backoff | `OrderService.triggerSaga()` — `@Retryable` |
| Global Exception Handling | `GlobalExceptionHandler` — covers all domains |
| Repository Pattern | All domain repositories |
| DTO Pattern | Request/Response separation in all domains |

---

## Enabling Kafka (Optional)

By default `kafka.enabled=false` — Saga runs synchronously in-process.

To enable Kafka:
1. Start Kafka on `localhost:9092`
2. In `application.yml` set:
   ```yaml
   kafka:
     enabled: true
   ```
3. Create topics: `order-events`, `payment-results`, `inventory-results`

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | Spring Boot 3.2.0 |
| Database | H2 In-Memory |
| ORM | Spring Data JPA / Hibernate |
| Validation | Jakarta Validation |
| API Docs | SpringDoc OpenAPI 2.3.0 |
| Messaging | Spring Kafka (disabled by default) |
| Retry | Spring Retry |
| Boilerplate | Lombok |
| Build | Maven |
| Java | 17 |
