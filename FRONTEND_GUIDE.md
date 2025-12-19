# Frontend Integration Guide - ChainPe Payment Gateway

## Overview

This guide explains how to build a frontend (web or mobile) that integrates with the ChainPe Stellar payment gateway.

## Architecture Flow

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│  Your Frontend  │────1───▶│  Your Backend    │────2───▶│  ChainPe API    │
│  (Checkout UI)  │         │  (Merchant API)  │         │  (Our Gateway)  │
└─────────────────┘         └──────────────────┘         └─────────────────┘
         │                           │                            │
         │                           │                            │
         └──────────3────────────────┴─────────────4──────────────┘
                                     │
                                     ▼
                         ┌──────────────────────┐
                         │   ChainPe Checkout   │
                         │   (Hosted Payment)   │
                         │   QR Code + Status   │
                         └──────────────────────┘
```

### Flow Steps:

1. **User initiates checkout** on your frontend
2. **Your backend** creates payment session via ChainPe API
3. **Redirect user** to ChainPe hosted checkout page
4. **User pays** using Stellar wallet (scans QR code)
5. **ChainPe detects** payment on blockchain
6. **User redirected** back to your success/cancel URL
7. **Webhook notification** sent to your backend

## Integration Steps

### Step 1: Merchant Registration

First, register your merchant account:

**Endpoint:** `POST /auth/register`

```bash
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My E-Commerce Store",
    "email": "merchant@mystore.com",
    "password": "SecurePassword123!",
    "stellar_address": "GCXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "webhook_url": "https://mystore.com/api/payment-webhook"
  }'
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "merchant": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "My E-Commerce Store",
    "email": "merchant@mystore.com",
    "stellar_address": "GCXX...",
    "is_active": true
  }
}
```

**Save the `access_token` securely in your backend!**

### Step 2: Create Payment Session (Backend)

When user clicks "Pay Now" on your checkout page, your backend should create a payment session:

**Endpoint:** `POST /api/merchant/payment`

```javascript
// Example: Node.js/Express
app.post('/checkout', async (req, res) => {
  const { amount, currency } = req.body;
  
  const response = await fetch('http://localhost:8000/api/merchant/payment', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${MERCHANT_TOKEN}` // Your saved token
    },
    body: JSON.stringify({
      amount_fiat: amount,
      fiat_currency: currency,
      success_url: 'https://mystore.com/payment/success',
      cancel_url: 'https://mystore.com/payment/cancel'
    })
  });
  
  const data = await response.json();
  
  // Redirect user to ChainPe checkout page
  res.redirect(data.checkout_url);
});
```

**Response:**
```json
{
  "session_id": "pay_abc123def456",
  "amount_usdc": "50.00",
  "checkout_url": "http://localhost:8000/checkout/pay_abc123def456",
  "expires_at": "2025-12-19T10:15:00Z",
  "status": "created"
}
```

### Step 3: Redirect to ChainPe Checkout

Redirect the user to `checkout_url`. This is our **hosted payment page** that shows:

- ✅ Payment amount (Fiat + USDC equivalent)
- ✅ QR code for Stellar wallet
- ✅ Merchant's Stellar address (copyable)
- ✅ Real-time payment status updates
- ✅ Countdown timer (15 minutes)
- ✅ Cancel button

**User scans QR code** with their Stellar wallet app (like Lobstr, Freighter, etc.) and sends USDC.

### Step 4: Handle Webhook Notifications (Backend)

ChainPe will send webhook notifications to your `webhook_url` when payment status changes:

```javascript
// Example: Express webhook endpoint
app.post('/api/payment-webhook', express.json(), (req, res) => {
  const { session_id, status, amount_usdc, tx_hash, paid_at } = req.body;
  
  // Verify webhook signature (optional but recommended)
  // const signature = req.headers['x-chainpe-signature'];
  
  if (status === 'paid') {
    // Payment successful!
    // 1. Mark order as paid in your database
    // 2. Send confirmation email to customer
    // 3. Trigger fulfillment process
    
    console.log(`Payment ${session_id} successful!`);
    console.log(`Transaction hash: ${tx_hash}`);
    
    // Update your database
    await Order.update({ 
      paymentStatus: 'paid',
      txHash: tx_hash
    }, { 
      where: { sessionId: session_id } 
    });
  }
  
  // Always respond 200 OK
  res.status(200).json({ received: true });
});
```

**Webhook payload:**
```json
{
  "session_id": "pay_abc123def456",
  "merchant_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "paid",
  "amount_fiat": "50.00",
  "fiat_currency": "USD",
  "amount_usdc": "50.00",
  "tx_hash": "a1b2c3d4e5f6...",
  "paid_at": "2025-12-19T10:05:32Z"
}
```

### Step 5: Handle Success/Cancel Redirects

After payment (or cancellation), user is redirected back to your URLs:

**Success Page (`success_url`):**
```html
<!-- https://mystore.com/payment/success?session_id=pay_abc123 -->
<!DOCTYPE html>
<html>
<head>
    <title>Payment Successful</title>
</head>
<body>
    <h1>✅ Payment Successful!</h1>
    <p>Thank you for your payment.</p>
    <p>Order ID: <span id="orderId"></span></p>
    
    <script>
        const urlParams = new URLSearchParams(window.location.search);
        const sessionId = urlParams.get('session_id');
        
        // Verify payment on your backend
        fetch(`/api/verify-payment/${sessionId}`)
            .then(res => res.json())
            .then(data => {
                document.getElementById('orderId').textContent = data.order_id;
            });
    </script>
</body>
</html>
```

**Cancel Page (`cancel_url`):**
```html
<!-- https://mystore.com/payment/cancel?session_id=pay_abc123 -->
<!DOCTYPE html>
<html>
<head>
    <title>Payment Cancelled</title>
</head>
<body>
    <h1>❌ Payment Cancelled</h1>
    <p>Your payment was cancelled or expired.</p>
    <a href="/checkout">Try again</a>
</body>
</html>
```

## Frontend Examples

### React Example

```jsx
// components/CheckoutButton.jsx
import React from 'react';

export default function CheckoutButton({ amount, currency, orderId }) {
  const handleCheckout = async () => {
    try {
      // Call your backend to create payment session
      const response = await fetch('/api/create-payment', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          amount,
          currency,
          orderId
        })
      });
      
      const { checkout_url } = await response.json();
      
      // Redirect to ChainPe checkout
      window.location.href = checkout_url;
    } catch (error) {
      console.error('Checkout error:', error);
      alert('Failed to initiate payment');
    }
  };
  
  return (
    <button 
      onClick={handleCheckout}
      className="bg-purple-600 text-white px-6 py-3 rounded-lg"
    >
      Pay ${amount} with USDC
    </button>
  );
}
```

### Vue Example

```vue
<!-- components/CheckoutButton.vue -->
<template>
  <button @click="handleCheckout" class="checkout-btn">
    Pay {{ amount }} {{ currency }} with USDC
  </button>
</template>

<script>
export default {
  props: ['amount', 'currency', 'orderId'],
  methods: {
    async handleCheckout() {
      try {
        const response = await fetch('/api/create-payment', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            amount: this.amount,
            currency: this.currency,
            orderId: this.orderId
          })
        });
        
        const { checkout_url } = await response.json();
        window.location.href = checkout_url;
      } catch (error) {
        console.error('Checkout error:', error);
      }
    }
  }
}
</script>
```

### Next.js Example

```typescript
// app/checkout/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function CheckoutPage() {
  const router = useRouter();
  const [loading, setLoading] = useState(false);
  
  const handlePayment = async () => {
    setLoading(true);
    
    try {
      const response = await fetch('/api/create-payment', {
        method: 'POST',
        body: JSON.stringify({
          amount: 100.00,
          currency: 'USD'
        })
      });
      
      const { checkout_url } = await response.json();
      window.location.href = checkout_url;
    } catch (error) {
      console.error(error);
      setLoading(false);
    }
  };
  
  return (
    <div className="container mx-auto p-8">
      <h1 className="text-3xl font-bold mb-4">Checkout</h1>
      <div className="bg-white shadow-md rounded p-6">
        <p className="text-xl mb-4">Total: $100.00 USD</p>
        <button
          onClick={handlePayment}
          disabled={loading}
          className="bg-purple-600 text-white px-8 py-3 rounded"
        >
          {loading ? 'Processing...' : 'Pay with USDC'}
        </button>
      </div>
    </div>
  );
}
```

## Mobile App Integration

### React Native Example

```jsx
import React from 'react';
import { Button, Linking } from 'react-native';

export default function CheckoutScreen({ amount, currency }) {
  const handleCheckout = async () => {
    try {
      const response = await fetch('https://mybackend.com/api/create-payment', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount, currency })
      });
      
      const { checkout_url } = await response.json();
      
      // Open in default browser
      await Linking.openURL(checkout_url);
      
      // Or use in-app browser
      // import { WebView } from 'react-native-webview';
      // navigation.navigate('Payment', { url: checkout_url });
    } catch (error) {
      console.error(error);
    }
  };
  
  return (
    <Button 
      title={`Pay $${amount} with USDC`}
      onPress={handleCheckout}
    />
  );
}
```

## API Reference Summary

### Key Endpoints for Frontend Integration

| Endpoint | Method | Purpose | Auth Required |
|----------|--------|---------|---------------|
| `/auth/register` | POST | Register merchant | No |
| `/auth/login` | POST | Get access token | No |
| `/api/merchant/payment` | POST | Create payment session | Yes (Merchant) |
| `/api/public/verify/{session_id}` | GET | Check payment status | No |
| `/checkout/{session_id}` | GET | Hosted checkout page | No |

Full API documentation: [API_REFERENCE.md](./API_REFERENCE.md)

## Testing

### Test Cards / Wallets

For testnet testing:

1. **Get test USDC:**
   - Use Stellar testnet friendbot: https://friendbot.stellar.org
   - Fund your testnet account with test XLM
   - Add USDC trustline for issuer: `GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5`

2. **Test Stellar wallets:**
   - **Lobstr** (Mobile): https://lobstr.co
   - **Freighter** (Browser extension): https://freighter.app
   - **Solar Wallet** (Desktop/Mobile): https://solarwallet.io

3. **Test payment flow:**
```bash
# 1. Create payment session
curl -X POST http://localhost:8000/api/merchant/payment \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_fiat": "10.00",
    "fiat_currency": "USD",
    "success_url": "http://localhost:3000/success",
    "cancel_url": "http://localhost:3000/cancel"
  }'

# 2. Visit checkout URL
# 3. Scan QR code with Stellar testnet wallet
# 4. Send exact USDC amount shown
```

## Security Best Practices

### 1. **Never expose merchant tokens in frontend:**
```javascript
// ❌ BAD - Don't do this
const MERCHANT_TOKEN = 'eyJhbGc...'; // Exposed in browser

// ✅ GOOD - Call your backend
fetch('/api/create-payment', { ... });
```

### 2. **Verify webhook signatures:**
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const hash = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');
  
  return hash === signature;
}
```

### 3. **Validate session on success page:**
```javascript
// Don't trust URL parameters alone
app.get('/payment/success', async (req, res) => {
  const { session_id } = req.query;
  
  // Verify with ChainPe API or your database
  const payment = await verifyPaymentSession(session_id);
  
  if (payment.status !== 'paid') {
    return res.redirect('/payment/failed');
  }
  
  res.render('success', { payment });
});
```

## UI/UX Best Practices

### 1. **Loading States**
Always show loading indicators during API calls:
```jsx
{loading ? <Spinner /> : <CheckoutButton />}
```

### 2. **Error Handling**
Display user-friendly error messages:
```jsx
{error && (
  <Alert type="error">
    Payment failed: {error.message}
    <button onClick={retry}>Try Again</button>
  </Alert>
)}
```

### 3. **Payment Status Updates**
Show real-time updates on checkout page:
- Waiting for payment ⏳
- Payment detected ⚡
- Confirming... 🔄
- Success! ✅

### 4. **Mobile Responsiveness**
Ensure checkout works on all devices, especially mobile (QR codes)

## Common Issues & Solutions

### Issue 1: CORS Errors
```javascript
// Add your frontend URL to .env
CORS_ORIGINS=http://localhost:3000,https://myapp.com
```

### Issue 2: Webhook Not Received
- Check firewall settings
- Use ngrok for local testing: `ngrok http 3000`
- Verify webhook URL in merchant settings

### Issue 3: QR Code Not Scanning
- Ensure proper lighting
- Try manual copy-paste of Stellar address
- Check wallet supports USDC on Stellar

## Example Projects

Check out complete example integrations:

- **React + Express:** [Link to example repo]
- **Next.js:** [Link to example repo]
- **Vue + Node:** [Link to example repo]
- **React Native:** [Link to example repo]

## Support

For integration help:
- 📧 Email: support@chainpe.com
- 📚 Documentation: http://localhost:8000/docs
- 💬 Discord: [Your Discord]
- 🐛 Issues: [GitHub Issues]

## Next Steps

1. ✅ Register your merchant account
2. ✅ Test payment flow on testnet
3. ✅ Implement webhook handler
4. ✅ Build checkout UI
5. ✅ Test end-to-end
6. 🚀 Go live on mainnet

---

**Happy coding! 🚀**
