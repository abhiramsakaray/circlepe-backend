# 🏗️ SYSTEM ARCHITECTURE

## Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        STELLAR PAYMENT GATEWAY                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────┐         ┌──────────────────────────────────┐
│   Merchant  │────────>│      FastAPI Backend             │
│   Dashboard │         │  (http://localhost:8000)         │
└─────────────┘         │                                  │
                        │  ┌────────────────────────────┐  │
┌─────────────┐         │  │   Authentication           │  │
│  End User   │────────>│  │   - JWT Tokens            │  │
│  (Buyer)    │         │  │   - Password Hashing      │  │
└─────────────┘         │  └────────────────────────────┘  │
                        │                                  │
┌─────────────┐         │  ┌────────────────────────────┐  │
│   Admin     │────────>│  │   Payment Sessions         │  │
│  Dashboard  │         │  │   - Create Session        │  │
└─────────────┘         │  │   - Track Status          │  │
                        │  └────────────────────────────┘  │
                        │                                  │
                        │  ┌────────────────────────────┐  │
                        │  │   Hosted Checkout          │  │
                        │  │   - QR Code Generation    │  │
                        │  │   - Real-time Polling     │  │
                        │  └────────────────────────────┘  │
                        └──────────────────────────────────┘
                                        │
                                        ▼
                        ┌──────────────────────────────────┐
                        │        Database (SQLite)         │
                        │  ┌────────────┬──────────────┐  │
                        │  │ merchants  │payment_sessions │
                        │  │  admins    │              │  │
                        │  └────────────┴──────────────┘  │
                        └──────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                   Stellar Listener (Background Worker)               │
│                   (python -m app.services.stellar_listener)         │
│                                                                      │
│   ┌───────────────────────────────────────────────────────────┐   │
│   │  1. Connect to Stellar Horizon API                        │   │
│   │  2. Stream transactions in real-time                      │   │
│   │  3. Filter USDC payments                                  │   │
│   │  4. Validate memo, amount, destination                    │   │
│   │  5. Update payment session status                         │   │
│   │  6. Trigger webhook notification                          │   │
│   └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                        ┌──────────────────────────────────┐
                        │   Stellar Blockchain (Testnet)   │
                        │   - USDC Payments                │
                        │   - Transaction Memos            │
                        │   - Real-time Confirmation       │
                        └──────────────────────────────────┘
                                        ▲
                                        │
                        ┌──────────────────────────────────┐
                        │   User Wallet (Freighter)        │
                        │   - Scan QR Code                 │
                        │   - Send USDC                    │
                        │   - Include Memo                 │
                        └──────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                        Webhook Delivery                              │
│                                                                      │
│   Backend ──HTTP POST──> Merchant Webhook URL                      │
│   {                                                                  │
│     "event": "payment.success",                                     │
│     "session_id": "pay_xxx",                                       │
│     "amount": "30.12",                                             │
│     "tx_hash": "abc123..."                                         │
│   }                                                                  │
│                                                                      │
│   Retry on failure (up to 3 times)                                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Component Interactions

### 1. Payment Session Creation Flow

```
Merchant App
    │
    │ POST /v1/payment_sessions
    │ {amount: 1999, currency: "INR"}
    ▼
FastAPI Backend
    │
    │ 1. Validate merchant
    │ 2. Check Stellar address exists
    │ 3. Generate session_id (pay_xxx)
    │ 4. Convert fiat to USDC
    │ 5. Save to database
    │ 6. Generate checkout_url
    ▼
Response
    {
      session_id: "pay_abc123",
      checkout_url: "http://app.com/checkout/pay_abc123"
    }
```

### 2. Checkout Flow

```
User Browser
    │
    │ GET /checkout/pay_abc123
    ▼
FastAPI Backend
    │
    │ 1. Fetch session from DB
    │ 2. Check expiry
    │ 3. Generate QR code
    │ 4. Render HTML page
    ▼
Checkout Page
    │
    │ - Display QR code
    │ - Show payment details
    │ - Poll /v1/payment_sessions/pay_abc123 every 2s
    │
    │ User scans QR with wallet
    │ User sends USDC payment
    │
    │ JavaScript polling detects status change
    │
    │ status === "paid"
    ▼
Auto Redirect to success_url
```

### 3. Payment Detection Flow

```
Stellar Listener
    │
    │ Connect to Horizon API
    │ Stream transactions
    ▼
New Transaction Detected
    │
    │ 1. Check asset == USDC
    │ 2. Extract memo (session_id)
    │ 3. Find session in DB
    │ 4. Validate destination address
    │ 5. Validate amount
    │ 6. Check session not already paid
    ▼
All Valid ✓
    │
    │ Update database:
    │   status = "paid"
    │   tx_hash = "abc123..."
    │   paid_at = now()
    ▼
Trigger Webhook
    │
    │ POST merchant.webhook_url
    │ Retry up to 3 times on failure
    ▼
Complete ✓
```

### 4. Authentication Flow

```
Client
    │
    │ POST /auth/login
    │ {email, password}
    ▼
FastAPI Backend
    │
    │ 1. Check if admin email
    │ 2. Verify password hash
    │ 3. Generate JWT token
    │    {sub: user_id, role: "merchant/admin"}
    ▼
Response
    {
      access_token: "eyJhbGci...",
      token_type: "bearer"
    }
    │
    │ Client stores token
    │
    │ Subsequent requests:
    │ Authorization: Bearer <token>
    ▼
Backend Middleware
    │
    │ 1. Decode JWT
    │ 2. Verify signature
    │ 3. Check expiry
    │ 4. Extract user_id and role
    │ 5. Verify role permissions
    ▼
Access Granted ✓
```

---

## Database Schema Visualization

```
┌─────────────────────────────────────┐
│           merchants                 │
├─────────────────────────────────────┤
│ id (UUID) PK                       │
│ name (String)                      │
│ email (String) UNIQUE              │
│ password_hash (String)             │
│ stellar_address (String) NULLABLE  │
│ webhook_url (String) NULLABLE      │
│ is_active (Boolean)                │
│ created_at (DateTime)              │
└─────────────────────────────────────┘
            │
            │ 1:N
            ▼
┌─────────────────────────────────────┐
│       payment_sessions              │
├─────────────────────────────────────┤
│ id (String) PK "pay_xxx"           │
│ merchant_id (UUID) FK              │
│ amount_fiat (Numeric)              │
│ fiat_currency (String)             │
│ amount_usdc (String)               │
│ status (Enum) created/paid/expired │
│ success_url (String)               │
│ cancel_url (String)                │
│ tx_hash (String) NULLABLE          │
│ created_at (DateTime)              │
│ paid_at (DateTime) NULLABLE        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│             admins                  │
├─────────────────────────────────────┤
│ id (UUID) PK                       │
│ email (String) UNIQUE              │
│ password_hash (String)             │
│ created_at (DateTime)              │
└─────────────────────────────────────┘
```

---

## File Organization Map

```
app/
│
├── core/                   # Core Infrastructure
│   ├── config.py          # Environment & Settings
│   ├── database.py        # SQLAlchemy Setup
│   └── security.py        # Auth & Hashing
│
├── models/                 # Data Models
│   └── models.py          # Merchant, PaymentSession, Admin
│
├── schemas/                # API Contracts
│   └── schemas.py         # Request/Response Schemas
│
├── routes/                 # API Endpoints
│   ├── auth.py            # /auth/*
│   ├── merchant.py        # /merchant/*
│   ├── payments.py        # /v1/payment_sessions/*
│   ├── checkout.py        # /checkout/*
│   └── admin.py           # /admin/*
│
├── services/               # Business Logic
│   ├── payment_utils.py   # Helper Functions
│   ├── stellar_listener.py # Blockchain Monitor
│   └── webhook_service.py # Webhook Delivery
│
└── main.py                # FastAPI App Entry Point
```

---

## Technology Stack Layers

```
┌─────────────────────────────────────────────────┐
│           Presentation Layer                    │
│  - Hosted Checkout HTML                         │
│  - QR Code Generation                           │
│  - Auto-generated API Docs (Swagger/ReDoc)      │
└─────────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────────┐
│           Application Layer (FastAPI)           │
│  - REST API Endpoints                           │
│  - Request Validation (Pydantic)                │
│  - JWT Authentication                           │
│  - CORS Middleware                              │
│  - Error Handling                               │
└─────────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────────┐
│           Business Logic Layer                  │
│  - Payment Session Management                   │
│  - Fiat-to-USDC Conversion                     │
│  - Payment Validation                           │
│  - Webhook Delivery                             │
└─────────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────────┐
│           Data Access Layer (SQLAlchemy)        │
│  - ORM Models                                   │
│  - Database Sessions                            │
│  - Relationships                                │
│  - Migrations                                   │
└─────────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────────┐
│           Database Layer                        │
│  - PostgreSQL (Production)                      │
│  - SQLite (Development)                         │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│         External Integration Layer              │
│  - Stellar Blockchain (via Horizon API)         │
│  - Merchant Webhooks (HTTP)                     │
└─────────────────────────────────────────────────┘
```

---

## Security Architecture

```
┌─────────────────────────────────────────────────┐
│              Security Layers                    │
└─────────────────────────────────────────────────┘

1. Network Layer
   ├── HTTPS (in production)
   ├── CORS Configuration
   └── Firewall Rules

2. Authentication Layer
   ├── JWT Tokens (HS256)
   ├── Token Expiry (24h)
   └── Role-Based Access

3. Authorization Layer
   ├── Merchant Role
   ├── Admin Role
   └── Endpoint Protection

4. Data Layer
   ├── Password Hashing (bcrypt)
   ├── Input Validation (Pydantic)
   ├── SQL Injection Prevention (ORM)
   └── No Sensitive Data Storage

5. Application Layer
   ├── Error Handling
   ├── Logging (no sensitive data)
   ├── Request Validation
   └── Environment Variables

6. Blockchain Layer
   ├── Read-Only Access
   ├── No Private Keys
   ├── Payment Verification Only
   └── Non-Custodial
```

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────┐
│            Production Setup                     │
└─────────────────────────────────────────────────┘

Load Balancer (nginx)
        │
        ├──> Web Server Instance 1 (FastAPI)
        │         │
        │         └──> PostgreSQL Database
        │
        ├──> Web Server Instance 2 (FastAPI)
        │         │
        │         └──> PostgreSQL Database
        │
        └──> Worker Instance (Stellar Listener)
                  │
                  └──> Stellar Horizon API

Optional:
├── Redis (Caching)
├── Sentry (Error Tracking)
├── DataDog (Monitoring)
└── S3 (Backups)
```

---

## Data Flow Summary

```
1. Merchant Registration
   Client → FastAPI → Database

2. Payment Session Creation
   Merchant App → FastAPI → Database → Response

3. Checkout Display
   User Browser → FastAPI → Database → HTML

4. Payment Execution
   User Wallet → Stellar Network

5. Payment Detection
   Stellar Network → Listener → Database

6. Webhook Notification
   Listener → Merchant Webhook

7. User Redirect
   Checkout Page → Poll API → Success URL
```

---

## Error Handling Flow

```
Request
    │
    ▼
Input Validation (Pydantic)
    │
    ├─ Invalid ──> 422 Unprocessable Entity
    │
    ▼ Valid
Authentication Check
    │
    ├─ No Token ──> 401 Unauthorized
    ├─ Invalid Token ──> 401 Unauthorized
    │
    ▼ Authenticated
Authorization Check
    │
    ├─ Wrong Role ──> 403 Forbidden
    │
    ▼ Authorized
Business Logic
    │
    ├─ Business Error ──> 400 Bad Request
    ├─ Not Found ──> 404 Not Found
    ├─ Server Error ──> 500 Internal Error
    │
    ▼ Success
Response (200/201)
```

---

This architecture ensures:
✅ Scalability
✅ Security
✅ Maintainability
✅ Real-time Performance
✅ Reliability
