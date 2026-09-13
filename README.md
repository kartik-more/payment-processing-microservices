# 💳 Distributed Payment Processing Microservices System
[![Java](https://img.shields.io/badge/Java-17-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.1.12-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Spring Data JPA](https://img.shields.io/badge/Spring%20Data-JPA-blue.svg)](https://spring.io/projects/spring-data-jpa)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-blue.svg)](https://www.mysql.com/)
A production-grade, distributed **Payment Processing System** built using **Java 17, Spring Boot 3, Spring Data JPA, Hibernate, and MySQL**. The system comprises two decoupled microservices executing real-time payment validation, lifecycle state-machine transitions, and ACID audit logging.
---
└─────────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                        Synchronous HTTP POST (RestTemplate Client)
                                          │
                                          ▼ (http://localhost:8082/v1/payments)
┌─────────────────────────────────────────────────────────────────────────────────┐
│  MICROSERVICE 2: payment-processing-service-v2 (Port 8082)                      │
│                                                                                 │
│  [PaymentController] ──> [PaymentServiceImpl]                                   │
│                               ├── Creates UUID Reference                        │
│                               ├── Enforces State Machine Lifecycle              │
│                               └── Logs Audit Trail                              │
│                                         │                                       │
│                                         ▼ (JPA)                                 │
│                   ┌─────────────────────┴─────────────────────┐                 │
│                   ▼                                           ▼                 │
│       [TransactionRepository]                    [TransactionLogRepository]     │
│                   │                                           │                 │
│                   ▼                                           ▼                 │
│       MySQL DB: `payments`                        MySQL DB: `payments`          │
│       Table: `transactions`                       Table: `transaction_log`      │
└─────────────────────────────────────────────────────────────────────────────────┘
```
---
## 📖 Deep-Dive Code & Component Walkthrough
### 1. Microservice 1: `payment-validation-service-v2` (Port 8081)
* **Goal**: Acts as the API Gateway/Ingress validator for all incoming merchant payment requests.
* **3-Stage Validation Pipeline**:
  1. `AmountValidator`: Validates that `amount > 0.0`. Throws `InvalidAmountException` (Mapped to HTTP 400).
  2. `DuplicatePaymentValidator`: Attempts to insert `merchantTxnRef` into MySQL database `validations.merchant_payment_request`. If duplicate reference is supplied, MySQL raises `DataIntegrityViolationException`, which is caught and re-thrown as `DuplicatePaymentException` (Mapped to HTTP 400).
  3. `ThresholdValidator`: Executes JPA derived query `countByUserIdAndCreatedAtAfter(userId, timeLimit)` to verify the user has not exceeded maximum payment attempts (Max 3 attempts within 60 minutes). Throws `ThresholdExceededException` (Mapped to HTTP 400).
* **Inter-Service Communication**:
  - `ProcessingServiceClient`: Uses `RestTemplate` to forward validated payloads to `payment-processing-service-v2` at `http://localhost:8082/v1/payments`. If the processing service is offline, it catches `ResourceAccessException` and throws `ServiceUnavailableException` (Mapped to HTTP 503 Service Unavailable).
* **Global Exception Handler (`@RestControllerAdvice`)**:
  - `GlobalExceptionHandler.java`: Intercepts all custom runtime exceptions (`InvalidAmountException`, `DuplicatePaymentException`, `ThresholdExceededException`, `ServiceUnavailableException`, `@Valid` `MethodArgumentNotValidException`) and converts them into standardized JSON error responses containing specific `ErrorCode` enums (e.g., `10001`, `10002`, `10003`, `10004`).

### 2. Microservice 2: `payment-processing-service-v2` (Port 8082)
* **Goal**: Core transaction ledger and state machine lifecycle manager.
* **State Machine Lifecycle**:
  - `CREATED` (Phase 1): Initialized upon payment creation. Generates server-side UUID `txnReference`.
  - `INITIATED` (Phase 2): Triggered when user opens the payment gateway page (`/initiate`).
  - `SUCCESS` / `FAILED` (Phase 3): Terminal states updated via payment callback / webhook (`/status`).
* **Dual-Write Persistence & `@Transactional` Control**:
  - Service methods (`createPayment`, `initiatePayment`, `updateStatus`) are annotated with `@Transactional`.
  - Every state transition performs a **dual-write**:
    1. Saves/Updates transaction entity in `payments.transactions` via `TransactionRepository`.
    2. Writes immutable historical log entry in `payments.transaction_log` via `TransactionLogRepository`.
  - If any error occurs during logging, `@Transactional` triggers an automatic database rollback to preserve ACID consistency.
   ```sql
CREATE TABLE payments.transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    userId VARCHAR(100) NOT NULL,
    paymentMethod VARCHAR(50) NOT NULL,
    provider VARCHAR(50) NOT NULL,
    paymentType VARCHAR(50) NOT NULL,
    amount DOUBLE NOT NULL,
    currency VARCHAR(10) NOT NULL,
    txnStatus VARCHAR(20) NOT NULL,
    merchantTxnRef VARCHAR(100) NOT NULL,
    txnReference VARCHAR(100) NOT NULL UNIQUE
);
CREATE TABLE payments.transaction_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    txnReference VARCHAR(100) NOT NULL,
    fromStatus VARCHAR(20) NOT NULL,
    toStatus VARCHAR(20) NOT NULL
);
```
---

* **Request Body**:
```json
{
  "txnStatus": "SUCCESS"
}
```
* **Success Response (`200 OK`)**:
```json
{
  "txnReference": "e4d3c2b1-5555-4abc-9999-123456789abc",
  "txnStatus": "SUCCESS",
  "errorCode": null,
  "errorMessage": null
}
```
---
## 🛠️ How to Run Locally
### 1. Prerequisites
* **Java 17** installed (`java -version`)
* **MySQL 8** running on `localhost:3306` (username: `root`, password: `root`)
### 2. Execution Steps
1. **Clone the repository**:
   ```bash
   git clone https://github.com/your-username/payment-processing-microservices.git
   ```
2. **Start Processing Service**:
   Run `PaymentProcessingApp.java` in IDE (Port `8082`).
