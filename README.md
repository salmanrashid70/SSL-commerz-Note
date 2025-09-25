
# SSLCommerz Payment Integration

This document describes the **Backend architecture flow** for integrating the **SSLCommerz payment gateway** into an Express.js REST API for an e-commerce platform. It focuses on **Architectural clarity, Backend logic, Database interactions, and Real-time updates** using Socket.IO.  

---

## 🎯 Goals

- Reliable and auditable payment processing.
- Clear separation of **UX routes** (success/fail/cancel) vs **Truth routes** (IPN).
- Idempotent and race-safe order state transitions.
- Real-time status updates to frontend via **Socket.IO**.
- Extensible for retries, risk handling, and multi-currency support.

---

## 🔹 Key Principles

- **IPN (Instant Payment Notification)** = **single source of truth** for payment status.  
- **Success/Fail/Cancel** = UX-only redirects. No DB finalization.  
- **Redis**: temporary order sessions + idempotency keys.  
- **Socket.IO**: push real-time status updates to frontend.  
- **Database (MongoDB)**: Orders are the source of truth for the business.  
- **External API Calls**: triggered only after successful payment validation.  

---

## 🔹 Payment Lifecycle

`PENDING → VALIDATED → SUCCESS | FAILED | CANCELLED | SYNC_PENDING`

- **PENDING**: After `/payment/init`.  
- **VALIDATED**: IPN received, SSLCommerz validation passed.  
- **SUCCESS**: Payment validated + external API (e.g., license/key provisioning) succeeded.  
- **SYNC_PENDING**: Payment validated, but external API failed → retry later.  
- **FAILED / CANCELLED**: Transaction rejected or canceled.  

---

## 🔹 Flow Overview

### 1. Initiation
- Client requests `/payment/init`.  
- API creates an **Order (PENDING)** in DB.  
- Temporary session stored in **Redis** (`order-session:{sessionId}`).  
- API requests a payment session from SSLCommerz.  
- Response → client receives `GatewayPageURL` for redirection.  

### 2. Payment
- User completes payment on SSLCommerz hosted page.  

### 3. IPN (Source of Truth)
- SSLCommerz **server → server** calls `/payment/ipn/:sessionId`.  
- Backend validates payment using **SSL validator API**.  
- Updates **Order** in DB to `SUCCESS`, `FAILED`, or `CANCELLED`.  
- On success, calls **external API** synchronously (e.g., provision service).  
- Emits `paymentStatusUpdate` event to frontend via Socket.IO.  

### 4. UX Redirects
- SSLCommerz redirects user to:  
  - `/payment/success/:sessionId`  
  - `/payment/fail/:sessionId`  
  - `/payment/cancel/:sessionId`  
- These **do not finalize** the order — only redirect user to frontend pages.  

### 5. Frontend Status Check
- Frontend connects to **Socket.IO** and joins `sessionId` room.  
- Listens for `paymentStatusUpdate`.  
- Optionally calls `/payment/status/:sessionId` (polling fallback).  

---

## 🔹 Routes
| Method | Path                          | Purpose                           | Notes                  |
|--------|-------------------------------|-----------------------------------|------------------------|
| POST   | `/payment/init`               | Create order, start SSL session   | Auth required          |
| POST   | `/payment/success/:sessionId` | UX redirect only                  | No DB writes           |
| POST   | `/payment/fail/:sessionId`    | UX redirect only                  | No DB writes           |
| POST   | `/payment/cancel/:sessionId`  | UX redirect only                  | No DB writes           |
| POST   | `/payment/ipn/:sessionId`     | **Finalize via IPN**              | Validates & updates DB |
| GET    | `/payment/status/:sessionId`  | Polling fallback                  | Returns latest order   |

---

## 🔹 Backend Components

### 1. **Controllers**
- Handle route requests. 
- Delegate business logic to services. 
- Emit socket events on status changes. 

### 2. **Services**
- `SSLService`: Payment session + validation API. 
- `PaymentService`: Idempotency, state transitions, external sync. 
- `OrderService`: CRUD for orders. 
- `ExternalSyncService`: Calls external system after success. 
- `RedisService`: Session caching + deduplication. 

### 3. **Database Models**
- **Order**: Source of truth for each payment. 
  - `tran_id`, `val_id`, `amount`, `currency`, `sessionId`, `status`. 
  - `customer`, `items`, `chassisNo`, `productKey`. 
  - `payment_info` (raw validator response). 
  - `externalApiResponse`. 

### 4. **Redis Keys**
- `order-session:{sessionId}` → temp session (TTL ~15min). 
- `idempotency:val:{val_id}` → prevent double IPN handling. 
- `idempotency:tran:{tran_id}` → prevent race conditions. 

### 5. **Socket.IO**
- Clients join room = `sessionId`. 
- Server emits `paymentStatusUpdate` with `{ sessionId, tran_id, status }`. 

---

## 🔹 Sequence Diagrams

  ### Case 1: Initiate → Pay → IPN (preferred path)

    Client           API (Express)           SSLCommerz           MongoDB            Socket.IO
      |  POST /payment/init  ----------------->  |                     |                 |
      |                                          | create Order(PENDING)                 |
      |                                          |------ insert Order ------------------>|
      |                                          |                                       |
      |<---- 200 + GatewayPageURL ---------------|                                       |
      |---------------- redirect ----------------> SSL hosted page                       |
      |                                          |                                       |
      |                          (after pay)     |--- POST IPN --> /payment/ipn -------->|
      |                                          |                                       |
      |                                          |   validate with SSL validator         |
      |                                          |-------------- GET validate ---------->|
      |                                          |<----- VALIDATED ----------------------|
      |                                          | update: VALIDATED/SUCCESS             |
      |                                          |------- update Order ----------------->|
      |                                          | emit socket event                     |
      |                                          |-------------------------- emit ------>|

  ### Case 2: Success-first edge case (User redirect hits before IPN)
      Client       API (Express)           SSLCommerz           MongoDB            Socket.IO
      |  card paid    |                        |                    |                  |
      |<-- redirect --| POST /payment/success  |                    |                  |
      |  show "Processing..." & socket join    |                    |                  |
      |                                          (IPN arrives a moment later)          |
      |                                          |--- POST /payment/ipn -------------->|
      |                                          | validate + update + emit            |
      |                                          |-------------- emit ---------------->|
      |<------------------------- real-time status via socket ------------------------>|


## ✅ Best Practices
- Always rely on **IPN** for final payment status (not success redirect).
- Use **Redis** for idempotency to avoid duplicate order processing.
- Store **all validation responses** in DB for auditing.
- Keep **Success/Fail routes** only for frontend redirection & UX.
