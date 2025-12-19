# Quick Start - ChainPe Payment Gateway

## 🚀 Current Setup Status

✅ Backend API (FastAPI) - Complete
✅ PostgreSQL Database Configuration
✅ Checkout Page with QR Code - **Created!**
✅ Frontend Integration Guide - **Created!**

## ⚠️ Action Required

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

This will install `stellar-sdk` and all other required packages.

### 2. Start the Server

```bash
# Terminal 1: Start API server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

```bash
# Terminal 2: Start Stellar payment listener (REQUIRED!)
python -m app.services.stellar_listener
```

## 📄 Key Pages & Features

### 1. **Checkout Page (QR Code Page)**
**URL:** `http://localhost:8000/checkout/{session_id}`

**Features:**
- ✅ Beautiful purple gradient design
- ✅ QR code for Stellar wallet scanning
- ✅ Real-time payment status updates
- ✅ 15-minute countdown timer
- ✅ Copy-paste merchant address
- ✅ Mobile responsive
- ✅ Auto-redirect on success/cancel

**Location:** [app/templates/checkout.html](app/templates/checkout.html)

**When is it shown?**
When a merchant creates a payment session, they get a `checkout_url` like:
```
http://localhost:8000/checkout/pay_abc123def456
```

Redirect your customer to this URL to show the payment page with QR code!

### 2. **API Documentation**
- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`

### 3. **Health Check**
```bash
curl http://localhost:8000/health
```

## 🔄 Complete Payment Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   1. Customer Checkout                          │
│  Customer clicks "Pay Now" on your website/app                 │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│               2. Create Payment Session                         │
│  Your backend calls ChainPe API:                                │
│  POST /api/merchant/payment                                     │
│  Returns: checkout_url                                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│           3. Redirect to ChainPe Checkout Page                  │
│  Customer sees:                                                 │
│  • Amount to pay (USD + USDC)                                   │
│  • QR code ← SCAN HERE!                                         │
│  • Merchant Stellar address (copy-paste)                        │
│  • 15-minute countdown timer                                    │
│  • Real-time status updates                                     │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              4. Customer Scans QR Code                          │
│  Opens Stellar wallet app (Lobstr, Freighter, etc.)            │
│  Scans QR code → Wallet auto-fills:                            │
│    - Destination address                                        │
│    - USDC amount                                                │
│    - Payment memo                                               │
│  Customer confirms payment                                      │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│          5. Stellar Listener Detects Payment                    │
│  python -m app.services.stellar_listener                        │
│  Monitors blockchain every 5 seconds                            │
│  Validates payment amount + memo                                │
│  Updates payment_sessions table: status = 'paid'                │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             6. Checkout Page Updates                            │
│  JavaScript polls /api/public/verify/{session_id}               │
│  Sees status = 'paid'                                           │
│  Shows "✅ Payment successful!"                                  │
│  Auto-redirects to success_url after 2 seconds                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│               7. Webhook Notification                           │
│  ChainPe sends POST to your webhook_url:                        │
│  {                                                              │
│    "session_id": "pay_abc123",                                  │
│    "status": "paid",                                            │
│    "tx_hash": "a1b2c3...",                                      │
│    "amount_usdc": "50.00"                                       │
│  }                                                              │
│  Your backend marks order as paid, sends confirmation email    │
└─────────────────────────────────────────────────────────────────┘
```

## 📱 How QR Code Payment Works

### Step 1: Customer Sees Checkout Page
When merchant creates payment session and redirects customer to:
```
http://localhost:8000/checkout/pay_abc123def456
```

They see a beautiful page with:

```
┌──────────────────────────────────────┐
│    🔒 ChainPe Payment                │
│    Your Store Name                   │
├──────────────────────────────────────┤
│                                      │
│         USD 50.00                    │
│         ≈ 50.00 USDC                 │
│                                      │
│    ⏳ Waiting for payment...         │
│    Expires in: 14:59                 │
│                                      │
│    ┌────────────────────┐            │
│    │                    │            │
│    │   [QR CODE HERE]   │ ← Scan!   │
│    │                    │            │
│    └────────────────────┘            │
│                                      │
│  Scan with Stellar wallet app        │
│                                      │
│  Destination: GCX7N...  [Copy]       │
│  Amount: 50.00 USDC                  │
│  Network: Stellar Testnet            │
│                                      │
│      [Cancel Payment]                │
└──────────────────────────────────────┘
```

### Step 2: Customer Scans QR Code
QR code contains Stellar payment URI:
```
web+stellar:pay?
  destination=GCXXXXXXX&
  amount=50.00&
  memo=pay_abc123def456&
  asset_code=USDC&
  asset_issuer=GBBD47IF...
```

### Step 3: Wallet Auto-Fills
Customer's Stellar wallet (Lobstr, Freighter, etc.) automatically fills:
- **To:** Merchant's Stellar address
- **Amount:** 50.00 USDC
- **Memo:** pay_abc123def456 (payment ID)
- **Asset:** USDC (not XLM)

Customer just clicks "Confirm"!

### Step 4: Real-Time Status Update
Checkout page JavaScript polls every 3 seconds:
```javascript
// Checks payment status
GET /api/public/verify/pay_abc123def456

// When paid:
{
  "status": "paid",
  "tx_hash": "a1b2c3d4e5f6..."
}
```

Page shows "✅ Payment successful!" and redirects to merchant's success URL.

## 🛠️ Testing the Complete Flow

### Test Scenario 1: Create Payment & View QR Code

```bash
# 1. Register a merchant
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Store",
    "email": "test@store.com",
    "password": "SecurePass123",
    "stellar_address": "GAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "webhook_url": "https://webhook.site/your-uuid"
  }'

# Save the access_token from response

# 2. Create payment session
curl -X POST http://localhost:8000/api/merchant/payment \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_fiat": "25.50",
    "fiat_currency": "USD",
    "success_url": "https://mystore.com/success",
    "cancel_url": "https://mystore.com/cancel"
  }'

# Response includes checkout_url
# Visit that URL in browser to see QR code!
```

### Test Scenario 2: Simulate Complete Payment

```bash
# 1. Create payment session (as above)
# 2. Open checkout_url in browser
# 3. Open Stellar testnet wallet (Lobstr/Freighter)
# 4. Scan QR code
# 5. Confirm payment in wallet
# 6. Watch checkout page update in real-time
# 7. Auto-redirect to success URL
# 8. Check your webhook endpoint for notification
```

## 📚 Documentation Files

| File | Purpose |
|------|---------|
| [README.md](README.md) | Main project overview and architecture |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Deployment guide (local + production) |
| [API_REFERENCE.md](API_REFERENCE.md) | All 25 API endpoints documented |
| **[FRONTEND_GUIDE.md](FRONTEND_GUIDE.md)** | **How to build your frontend/app** ← NEW! |
| [QUICK_START.md](QUICK_START.md) | This file |

## 🎨 Frontend Integration

See [FRONTEND_GUIDE.md](FRONTEND_GUIDE.md) for:
- ✅ React, Vue, Next.js examples
- ✅ React Native mobile app integration
- ✅ Complete webhook handler code
- ✅ Security best practices
- ✅ Error handling
- ✅ Testing guides

### Quick Frontend Example (React)

```jsx
// Your checkout page component
function CheckoutButton({ amount, orderId }) {
  const handlePayment = async () => {
    // 1. Call YOUR backend
    const response = await fetch('/api/create-payment', {
      method: 'POST',
      body: JSON.stringify({ amount, orderId })
    });
    
    const { checkout_url } = await response.json();
    
    // 2. Redirect to ChainPe checkout page (shows QR code)
    window.location.href = checkout_url;
  };
  
  return (
    <button onClick={handlePayment}>
      Pay ${amount} with USDC
    </button>
  );
}
```

## 🔧 Troubleshooting

### Issue: "ModuleNotFoundError: No module named 'stellar_sdk'"
**Solution:**
```bash
pip install -r requirements.txt
```

### Issue: "No such table: merchants"
**Solution:**
```bash
python init_db.py
```
Or just start the server - it auto-creates tables on startup!

### Issue: Payments not being detected
**Solution:**
Make sure Stellar listener is running:
```bash
python -m app.services.stellar_listener
```

### Issue: Checkout page shows "Not Found"
**Solution:**
Check the session_id is correct and server is running:
```bash
curl http://localhost:8000/checkout/pay_abc123def456
```

## 🌐 API Endpoints Quick Reference

### Public (No Auth)
- `GET /checkout/{session_id}` - **QR code payment page**
- `GET /api/public/verify/{session_id}` - Check payment status
- `GET /api/public/stats` - Gateway statistics

### Merchant (Auth Required)
- `POST /auth/register` - Register merchant
- `POST /auth/login` - Login
- `POST /api/merchant/payment` - **Create payment session**
- `GET /api/merchant/payments` - List all payments
- `GET /api/merchant/stats` - Revenue statistics

### Admin
- `POST /admin/auth/login` - Admin login
- `GET /admin/merchants` - List all merchants
- `GET /admin/payments` - All payment sessions
- `POST /admin/webhooks/retry/{session_id}` - Retry webhook

Full docs: http://localhost:8000/docs

## ✅ Next Steps

1. ✅ Install dependencies: `pip install -r requirements.txt`
2. ✅ Start API server: `uvicorn app.main:app --reload`
3. ✅ Start Stellar listener: `python -m app.services.stellar_listener`
4. ✅ Register a merchant account
5. ✅ Create a payment session
6. ✅ Visit checkout URL to see QR code
7. ✅ Build your frontend (see FRONTEND_GUIDE.md)
8. 🚀 Go live!

---

**Questions?** Check:
- API Docs: http://localhost:8000/docs
- Frontend Guide: [FRONTEND_GUIDE.md](FRONTEND_GUIDE.md)
- Deployment: [DEPLOYMENT.md](DEPLOYMENT.md)
