# 📚 COMPLETE API REFERENCE

## Base URL
```
http://localhost:8000
```

## 🔐 Authentication
All protected endpoints require JWT token in the Authorization header:
```
Authorization: Bearer <your_jwt_token>
```

---

## 🌐 PUBLIC ENDPOINTS (No Auth Required)

### Get Public Stats
Get gateway statistics without authentication.

```http
GET /public/stats
```

**Response:**
```json
{
  "gateway": "Stellar Payment Gateway",
  "network": "testnet",
  "stats": {
    "total_transactions": 150,
    "successful_payments": 120,
    "active_merchants": 25,
    "last_24h_payments": 15
  },
  "status": "operational"
}
```

### Verify Payment Session
Check if a payment session exists and get minimal info.

```http
GET /public/session/{session_id}/verify
```

**Response:**
```json
{
  "session_id": "pay_abc123",
  "exists": true,
  "status": "created",
  "amount_usdc": "30.12",
  "merchant_name": "Demo Store",
  "is_expired": false,
  "created_at": "2025-12-19T10:30:00"
}
```

### Root
```http
GET /
```

### Health Check
```http
GET /health
```

---

## 👤 AUTHENTICATION ENDPOINTS

### Register Merchant
Create a new merchant account.

```http
POST /auth/register
Content-Type: application/json

{
  "name": "My Store",
  "email": "merchant@store.com",
  "password": "securepassword123"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

### Login
Login as merchant or admin.

```http
POST /auth/login
Content-Type: application/json

{
  "email": "merchant@store.com",
  "password": "securepassword123"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

---

## 🏪 MERCHANT PROFILE ENDPOINTS

### Get Profile
```http
GET /merchant/profile
Authorization: Bearer <token>
```

**Response:**
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "My Store",
  "email": "merchant@store.com",
  "stellar_address": "GABC123...",
  "webhook_url": "https://mystore.com/webhook",
  "is_active": true,
  "created_at": "2025-12-19T10:00:00"
}
```

### Update Profile
```http
PUT /merchant/profile
Authorization: Bearer <token>
Content-Type: application/json

{
  "stellar_address": "GABC123...",
  "webhook_url": "https://mystore.com/webhook"
}
```

---

## 💳 PAYMENT SESSION ENDPOINTS

### Create Payment Session
```http
POST /v1/payment_sessions
Authorization: Bearer <token>
Content-Type: application/json

{
  "amount": 1999,
  "currency": "INR",
  "success_url": "https://mystore.com/success",
  "cancel_url": "https://mystore.com/cancel"
}
```

**Response:**
```json
{
  "session_id": "pay_abc123",
  "checkout_url": "http://localhost:8000/checkout/pay_abc123"
}
```

### Get Payment Status (Public)
```http
GET /v1/payment_sessions/{session_id}
```

**Response:**
```json
{
  "session_id": "pay_abc123",
  "status": "paid",
  "amount_usdc": "30.12",
  "tx_hash": "abc123xyz...",
  "created_at": "2025-12-19T10:30:00",
  "paid_at": "2025-12-19T10:35:00"
}
```

---

## 📊 MERCHANT PAYMENTS ENDPOINTS (NEW!)

### Get All My Payment Sessions
Get all payment sessions for authenticated merchant with filtering.

```http
GET /merchant/payments?status=paid&limit=50&offset=0
Authorization: Bearer <token>
```

**Query Parameters:**
- `status` (optional): Filter by status (created, paid, expired)
- `limit` (optional): Max 100, default 50
- `offset` (optional): For pagination, default 0

**Response:**
```json
[
  {
    "id": "pay_abc123",
    "merchant_id": "123e4567...",
    "merchant_name": "My Store",
    "amount_fiat": 1999.00,
    "fiat_currency": "INR",
    "amount_usdc": "30.12",
    "status": "paid",
    "tx_hash": "abc123...",
    "created_at": "2025-12-19T10:30:00",
    "paid_at": "2025-12-19T10:35:00"
  }
]
```

### Get Payment Statistics
Get comprehensive stats for merchant account.

```http
GET /merchant/payments/stats
Authorization: Bearer <token>
```

**Response:**
```json
{
  "total_sessions": 100,
  "sessions_by_status": {
    "paid": 75,
    "pending": 15,
    "expired": 10
  },
  "revenue": {
    "total_usdc": 2500.50,
    "currency": "USDC"
  },
  "recent": {
    "today": 5,
    "this_week": 25
  },
  "success_rate": 75.00
}
```

### Get Recent Payments
Get most recent payment sessions.

```http
GET /merchant/payments/recent?limit=10
Authorization: Bearer <token>
```

**Query Parameters:**
- `limit` (optional): Max 50, default 10

### Get Specific Payment Detail
Get detailed info about a specific session.

```http
GET /merchant/payments/{session_id}
Authorization: Bearer <token>
```

**Response:**
```json
{
  "session_id": "pay_abc123",
  "status": "paid",
  "amount_usdc": "30.12",
  "tx_hash": "abc123...",
  "created_at": "2025-12-19T10:30:00",
  "paid_at": "2025-12-19T10:35:00"
}
```

### Cancel Payment Session
Manually cancel/expire a pending session.

```http
POST /merchant/payments/{session_id}/cancel
Authorization: Bearer <token>
```

**Response:**
```json
{
  "message": "Payment session cancelled successfully",
  "session_id": "pay_abc123",
  "status": "expired"
}
```

---

## 🎨 CHECKOUT ENDPOINTS

### Hosted Checkout Page (HTML)
```http
GET /checkout/{session_id}
```

Returns beautiful HTML checkout page with QR code.

### Get Checkout Details (JSON)
```http
GET /checkout/api/{session_id}
```

**Response:**
```json
{
  "id": "pay_abc123",
  "merchant_name": "My Store",
  "merchant_stellar_address": "GABC123...",
  "amount_fiat": 1999.00,
  "fiat_currency": "INR",
  "amount_usdc": "30.12",
  "status": "created",
  "success_url": "https://mystore.com/success",
  "cancel_url": "https://mystore.com/cancel",
  "tx_hash": null,
  "created_at": "2025-12-19T10:30:00",
  "paid_at": null
}
```

---

## 👑 ADMIN ENDPOINTS

### List All Merchants
```http
GET /admin/merchants?skip=0&limit=100
Authorization: Bearer <admin_token>
```

### List All Payments
```http
GET /admin/payments?skip=0&limit=100
Authorization: Bearer <admin_token>
```

### Gateway Health
```http
GET /admin/health
Authorization: Bearer <admin_token>
```

**Response:**
```json
{
  "status": "healthy",
  "merchants": {
    "total": 50,
    "active": 45,
    "inactive": 5
  },
  "payments": {
    "total": 500,
    "paid": 400,
    "pending": 75,
    "expired": 25
  }
}
```

### Disable/Enable Merchant
```http
PATCH /admin/merchants/{merchant_id}/disable
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "is_active": false
}
```

---

## 🔔 ADMIN WEBHOOK ENDPOINTS (NEW!)

### Retry Webhook
Manually retry webhook delivery for a paid session.

```http
POST /admin/webhooks/retry/{session_id}
Authorization: Bearer <admin_token>
```

**Response:**
```json
{
  "message": "Webhook retry initiated",
  "session_id": "pay_abc123",
  "webhook_url": "https://merchant.com/webhook"
}
```

### Test Webhook
Send a test webhook to merchant.

```http
GET /admin/webhooks/test/{merchant_id}
Authorization: Bearer <admin_token>
```

**Response:**
```json
{
  "success": true,
  "webhook_url": "https://merchant.com/webhook",
  "status_code": 200,
  "response": "OK"
}
```

---

## 📋 COMPLETE ENDPOINT SUMMARY

### Public (No Auth) - 4 endpoints
- `GET /` - Root
- `GET /health` - Health check
- `GET /public/stats` - Public statistics
- `GET /public/session/{id}/verify` - Verify session

### Authentication - 2 endpoints
- `POST /auth/register` - Register merchant
- `POST /auth/login` - Login

### Merchant Profile - 2 endpoints
- `GET /merchant/profile` - Get profile
- `PUT /merchant/profile` - Update profile

### Payment Sessions - 2 endpoints
- `POST /v1/payment_sessions` - Create session
- `GET /v1/payment_sessions/{id}` - Get status

### Merchant Payments (NEW) - 5 endpoints
- `GET /merchant/payments` - List all sessions
- `GET /merchant/payments/stats` - Get statistics
- `GET /merchant/payments/recent` - Recent payments
- `GET /merchant/payments/{id}` - Get specific session
- `POST /merchant/payments/{id}/cancel` - Cancel session

### Checkout - 2 endpoints
- `GET /checkout/{id}` - Hosted page (HTML)
- `GET /checkout/api/{id}` - Checkout details (JSON)

### Admin - 4 endpoints
- `GET /admin/merchants` - List merchants
- `GET /admin/payments` - List payments
- `GET /admin/health` - Health stats
- `PATCH /admin/merchants/{id}/disable` - Toggle merchant

### Admin Webhooks (NEW) - 2 endpoints
- `POST /admin/webhooks/retry/{id}` - Retry webhook
- `GET /admin/webhooks/test/{id}` - Test webhook

**Total: 25 Endpoints** (up from 14!)

---

## 🔄 Common Response Codes

- `200 OK` - Success
- `201 Created` - Resource created
- `400 Bad Request` - Invalid input
- `401 Unauthorized` - Missing or invalid token
- `403 Forbidden` - Insufficient permissions
- `404 Not Found` - Resource not found
- `500 Internal Server Error` - Server error

---

## 💡 Usage Examples

### Complete Merchant Flow

```bash
# 1. Register
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"My Store","email":"test@store.com","password":"pass123"}'

# Save the token from response

# 2. Update profile
curl -X PUT http://localhost:8000/merchant/profile \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"stellar_address":"GABC...","webhook_url":"https://webhook.site/xxx"}'

# 3. Create payment
curl -X POST http://localhost:8000/v1/payment_sessions \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount":100,"currency":"USD","success_url":"https://store.com/success","cancel_url":"https://store.com/cancel"}'

# 4. Get statistics
curl http://localhost:8000/merchant/payments/stats \
  -H "Authorization: Bearer YOUR_TOKEN"

# 5. List all payments
curl "http://localhost:8000/merchant/payments?status=paid&limit=10" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## 🎉 New Features Added

✅ **Merchant Payment Management**
- List all payment sessions with filtering
- Get detailed statistics
- View recent payments
- Cancel pending sessions

✅ **Public API**
- Public statistics endpoint
- Session verification endpoint

✅ **Admin Webhook Tools**
- Manually retry failed webhooks
- Test webhook connectivity

✅ **Auto Database Initialization**
- Database tables created automatically on startup
- Admin account created automatically
- No need to run init_db.py manually!

---

**Access the interactive API docs:**
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc
