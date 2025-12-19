# 🚀 ChainPe Backend - React Frontend Integration Guide

Complete guide to integrate ChainPe payment gateway backend with your React frontend.

---

## 📋 Table of Contents

1. [Backend Setup](#backend-setup)
2. [Frontend Setup](#frontend-setup)
3. [API Endpoints Reference](#api-endpoints-reference)
4. [React Integration Examples](#react-integration-examples)
5. [Advanced Features](#advanced-features)
6. [Production Deployment](#production-deployment)

---

## 🔧 Backend Setup

### Prerequisites

- Python 3.10+
- PostgreSQL 12+
- Node.js 16+ (for frontend)

### 1. Install Dependencies

```bash
cd backend
pip install -r requirements.txt
```

### 2. Configure Environment Variables

Create `.env` file:

```bash
# Database
DATABASE_URL=postgresql://username:password@localhost:5432/chainpe

# Stellar Configuration
STELLAR_NETWORK=testnet
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
USDC_ISSUER=GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5

# Application
APP_URL=http://localhost:8000
SECRET_KEY=your-secret-key-here
PAYMENT_EXPIRY_MINUTES=30

# CORS (allow your React app)
CORS_ORIGINS=http://localhost:3000,http://localhost:5173

# Admin
ADMIN_EMAIL=admin@chainpe.io
ADMIN_PASSWORD=admin123
```

### 3. Initialize Database

```bash
# Create database
psql -U postgres -c "CREATE DATABASE chainpe;"

# Run migrations
python -m app.core.init_db
```

### 4. Start Backend Server

```bash
# Development
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Production
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

### 5. Start Stellar Payment Listener

```bash
# In a separate terminal
python -m app.services.stellar_listener
```

Backend is now running at: **http://localhost:8000**

API Documentation: **http://localhost:8000/docs**

---

## ⚛️ Frontend Setup

### 1. Create React App

```bash
# Using Vite (recommended)
npm create vite@latest my-store -- --template react
cd my-store
npm install

# Or using Create React App
npx create-react-app my-store
cd my-store
```

### 2. Install Required Packages

```bash
npm install axios react-router-dom @tanstack/react-query
```

### 3. Configure Environment Variables

Create `.env` file in React project root:

```bash
VITE_API_URL=http://localhost:8000
VITE_CHAINPE_API_KEY=your_merchant_api_key_here
```

For Create React App, use `REACT_APP_` prefix:
```bash
REACT_APP_API_URL=http://localhost:8000
REACT_APP_CHAINPE_API_KEY=your_merchant_api_key_here
```

---

## 📡 API Endpoints Reference

### Base URL
```
http://localhost:8000
```

### Authentication
Most endpoints require API key in header:
```
X-API-Key: your_api_key_here
```

---

### **Public Endpoints (No Auth Required)**

#### 1. Health Check
```http
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "network": "testnet"
}
```

---

### **Merchant Authentication**

#### 1. Register Merchant
```http
POST /auth/register
Content-Type: application/json

{
  "name": "My Store",
  "email": "merchant@example.com",
  "password": "SecurePassword123"
}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer"
}
```

#### 2. Login Merchant
```http
POST /auth/login
Content-Type: application/json

{
  "email": "merchant@example.com",
  "password": "SecurePassword123"
}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer"
}
```

---

### **Merchant Profile Management**

#### 1. Get Merchant Profile
```http
GET /merchant/profile
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "My Store",
  "email": "merchant@example.com",
  "stellar_address": "GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "webhook_url": "https://mystore.com/webhooks/chainpe",
  "api_key": "pk_live_xxxxxxxxxxxx",
  "is_active": true,
  "created_at": "2025-01-01T00:00:00"
}
```

#### 2. Update Merchant Profile
```http
PUT /merchant/profile
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "stellar_address": "GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "webhook_url": "https://mystore.com/webhooks/chainpe"
}
```

---

### **Payment Session Management**

#### 1. Create Payment Session (Public API)
```http
POST /api/sessions/create
X-API-Key: your_api_key
Content-Type: application/json

{
  "amount_usdc": 50.00,
  "order_id": "ORDER-12345",
  "success_url": "https://mystore.com/success",
  "cancel_url": "https://mystore.com/cart",
  "metadata": {
    "customer_email": "customer@example.com",
    "customer_name": "John Doe",
    "items": [
      {"id": "item1", "name": "Product A", "quantity": 2}
    ]
  }
}
```

**Response:**
```json
{
  "session_id": "sess_abc123xyz",
  "checkout_url": "http://localhost:8000/checkout/sess_abc123xyz",
  "amount_usdc": 50.00,
  "order_id": "ORDER-12345",
  "expires_at": "2025-12-19T12:30:00",
  "status": "created",
  "success_url": "https://mystore.com/success",
  "cancel_url": "https://mystore.com/cart"
}
```

#### 2. Get Payment Session Status
```http
GET /api/sessions/{session_id}
X-API-Key: your_api_key
```

**Response:**
```json
{
  "session_id": "sess_abc123xyz",
  "status": "paid",
  "amount_usdc": "50.00",
  "order_id": "ORDER-12345",
  "tx_hash": "abc123...xyz789",
  "created_at": "2025-12-19T12:00:00",
  "paid_at": "2025-12-19T12:05:00",
  "expires_at": "2025-12-19T12:30:00",
  "metadata": {
    "customer_email": "customer@example.com"
  }
}
```

#### 3. List Payment Sessions (Merchant Dashboard)
```http
GET /merchant/payments?limit=10&offset=0
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "total": 100,
  "payments": [
    {
      "id": "sess_abc123xyz",
      "amount_usdc": "50.00",
      "status": "paid",
      "order_id": "ORDER-12345",
      "tx_hash": "abc123...xyz789",
      "created_at": "2025-12-19T12:00:00",
      "paid_at": "2025-12-19T12:05:00"
    }
  ]
}
```

---

### **Checkout Page**

#### Get Hosted Checkout Page
```http
GET /checkout/{session_id}
```

Returns HTML page with QR codes and payment UI.

---

## ⚛️ React Integration Examples

### Project Structure

```
src/
├── components/
│   ├── ChainPeButton.jsx
│   ├── PaymentStatus.jsx
│   └── PaymentHistory.jsx
├── services/
│   ├── api.js
│   └── chainpe.js
├── hooks/
│   ├── useChainPe.js
│   └── usePaymentStatus.js
├── pages/
│   ├── Checkout.jsx
│   ├── Success.jsx
│   └── Dashboard.jsx
└── App.jsx
```

---

### 1. API Service Setup

**`src/services/api.js`**

```javascript
import axios from 'axios';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000';

// Create axios instance
export const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add API key to requests
api.interceptors.request.use((config) => {
  const apiKey = import.meta.env.VITE_CHAINPE_API_KEY;
  if (apiKey) {
    config.headers['X-API-Key'] = apiKey;
  }
  
  // Add JWT token if available
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers['Authorization'] = `Bearer ${token}`;
  }
  
  return config;
});

// Handle errors
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('access_token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

---

### 2. ChainPe Service

**`src/services/chainpe.js`**

```javascript
import api from './api';

export const chainpeService = {
  // Create payment session
  createSession: async (paymentData) => {
    const response = await api.post('/api/sessions/create', paymentData);
    return response.data;
  },

  // Get session status
  getSessionStatus: async (sessionId) => {
    const response = await api.get(`/api/sessions/${sessionId}`);
    return response.data;
  },

  // Get merchant profile
  getMerchantProfile: async () => {
    const response = await api.get('/merchant/profile');
    return response.data;
  },

  // Update merchant profile
  updateMerchantProfile: async (data) => {
    const response = await api.put('/merchant/profile', data);
    return response.data;
  },

  // Get payment history
  getPaymentHistory: async (params = {}) => {
    const response = await api.get('/merchant/payments', { params });
    return response.data;
  },

  // Merchant auth
  register: async (data) => {
    const response = await api.post('/auth/register', data);
    return response.data;
  },

  login: async (data) => {
    const response = await api.post('/auth/login', data);
    return response.data;
  },
};
```

---

### 3. Custom React Hook

**`src/hooks/useChainPe.js`**

```javascript
import { useState } from 'react';
import { chainpeService } from '../services/chainpe';

export const useChainPe = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const createPayment = async (paymentData) => {
    setLoading(true);
    setError(null);

    try {
      const session = await chainpeService.createSession(paymentData);
      
      // Redirect to checkout
      window.location.href = session.checkout_url;
      
      return session;
    } catch (err) {
      setError(err.response?.data?.detail || 'Failed to create payment');
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return {
    createPayment,
    loading,
    error,
  };
};
```

**`src/hooks/usePaymentStatus.js`**

```javascript
import { useQuery } from '@tanstack/react-query';
import { chainpeService } from '../services/chainpe';

export const usePaymentStatus = (sessionId, options = {}) => {
  return useQuery({
    queryKey: ['payment', sessionId],
    queryFn: () => chainpeService.getSessionStatus(sessionId),
    enabled: !!sessionId,
    refetchInterval: (data) => {
      // Poll every 3 seconds if payment is still pending
      if (data?.status === 'created') {
        return 3000;
      }
      return false;
    },
    ...options,
  });
};
```

---

### 4. ChainPe Payment Button Component

**`src/components/ChainPeButton.jsx`**

```jsx
import React from 'react';
import { useChainPe } from '../hooks/useChainPe';
import './ChainPeButton.css';

export const ChainPeButton = ({ 
  amount, 
  orderId, 
  metadata = {},
  onSuccess,
  onError 
}) => {
  const { createPayment, loading, error } = useChainPe();

  const handlePayment = async () => {
    try {
      await createPayment({
        amount_usdc: amount,
        order_id: orderId,
        success_url: `${window.location.origin}/success`,
        cancel_url: window.location.href,
        metadata,
      });
      
      onSuccess?.();
    } catch (err) {
      onError?.(err);
    }
  };

  return (
    <div className="chainpe-button-container">
      <button
        onClick={handlePayment}
        disabled={loading}
        className="chainpe-button"
      >
        {loading ? (
          <>
            <span className="spinner" />
            Processing...
          </>
        ) : (
          <>
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
              <path d="M12 2L2 7L12 12L22 7L12 2Z" fill="white"/>
              <path d="M2 17L12 22L22 17" stroke="white" strokeWidth="2"/>
              <path d="M2 12L12 17L22 12" stroke="white" strokeWidth="2"/>
            </svg>
            Pay with ChainPe
          </>
        )}
      </button>
      
      {error && (
        <div className="error-message">{error}</div>
      )}
    </div>
  );
};
```

**`src/components/ChainPeButton.css`**

```css
.chainpe-button {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  padding: 14px 28px;
  font-size: 16px;
  font-weight: 600;
  border-radius: 8px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 10px;
  transition: all 0.2s;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.chainpe-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
}

.chainpe-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error-message {
  margin-top: 10px;
  padding: 10px;
  background: #fee;
  color: #c33;
  border-radius: 4px;
  font-size: 14px;
}
```

---

### 5. Payment Status Component

**`src/components/PaymentStatus.jsx`**

```jsx
import React from 'react';
import { usePaymentStatus } from '../hooks/usePaymentStatus';

export const PaymentStatus = ({ sessionId }) => {
  const { data: payment, isLoading, error } = usePaymentStatus(sessionId);

  if (isLoading) {
    return (
      <div className="payment-status loading">
        <div className="spinner-large" />
        <p>Verifying payment...</p>
      </div>
    );
  }

  if (error) {
    return (
      <div className="payment-status error">
        <h2>❌ Verification Failed</h2>
        <p>Unable to verify payment status.</p>
      </div>
    );
  }

  if (payment?.status === 'paid') {
    return (
      <div className="payment-status success">
        <div className="success-icon">✅</div>
        <h2>Payment Successful!</h2>
        <div className="payment-details">
          <div className="detail-row">
            <span>Order ID:</span>
            <strong>{payment.order_id}</strong>
          </div>
          <div className="detail-row">
            <span>Amount:</span>
            <strong>{payment.amount_usdc} USDC</strong>
          </div>
          <div className="detail-row">
            <span>Transaction:</span>
            <code className="tx-hash">{payment.tx_hash}</code>
          </div>
        </div>
      </div>
    );
  }

  if (payment?.status === 'created') {
    return (
      <div className="payment-status pending">
        <h2>⏳ Awaiting Payment</h2>
        <p>Please complete payment on the checkout page.</p>
      </div>
    );
  }

  return (
    <div className="payment-status">
      <p>Status: {payment?.status}</p>
    </div>
  );
};
```

---

### 6. Complete Page Examples

**`src/pages/Checkout.jsx`**

```jsx
import React, { useState } from 'react';
import { ChainPeButton } from '../components/ChainPeButton';

export const Checkout = () => {
  const [cart] = useState([
    { id: '1', name: 'Premium Headset', price: 50.00, quantity: 1 },
  ]);

  const total = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  const orderId = `ORDER-${Date.now()}`;

  return (
    <div className="checkout-page">
      <h1>Checkout</h1>
      
      <div className="cart-items">
        {cart.map((item) => (
          <div key={item.id} className="cart-item">
            <span>{item.name}</span>
            <span>${item.price} × {item.quantity}</span>
          </div>
        ))}
      </div>

      <div className="total">
        <strong>Total: ${total.toFixed(2)} USDC</strong>
      </div>

      <ChainPeButton
        amount={total}
        orderId={orderId}
        metadata={{
          customerEmail: 'customer@example.com',
          items: cart,
        }}
        onSuccess={() => console.log('Payment initiated')}
        onError={(err) => console.error('Payment failed', err)}
      />
    </div>
  );
};
```

**`src/pages/Success.jsx`**

```jsx
import React, { useEffect } from 'react';
import { useSearchParams } from 'react-router-dom';
import { PaymentStatus } from '../components/PaymentStatus';

export const Success = () => {
  const [searchParams] = useSearchParams();
  const sessionId = searchParams.get('session_id');

  useEffect(() => {
    // Track conversion
    if (sessionId) {
      console.log('Payment session:', sessionId);
      // Add analytics tracking here
    }
  }, [sessionId]);

  return (
    <div className="success-page">
      <PaymentStatus sessionId={sessionId} />
      
      <div className="actions">
        <a href="/" className="button">Continue Shopping</a>
      </div>
    </div>
  );
};
```

**`src/pages/Dashboard.jsx`**

```jsx
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { chainpeService } from '../services/chainpe';

export const Dashboard = () => {
  const { data: payments, isLoading } = useQuery({
    queryKey: ['payments'],
    queryFn: () => chainpeService.getPaymentHistory({ limit: 50 }),
  });

  const { data: profile } = useQuery({
    queryKey: ['profile'],
    queryFn: chainpeService.getMerchantProfile,
  });

  return (
    <div className="dashboard">
      <h1>Payment Dashboard</h1>
      
      <div className="profile-card">
        <h2>{profile?.name}</h2>
        <p>API Key: <code>{profile?.api_key}</code></p>
      </div>

      <div className="payments-table">
        <h2>Recent Payments</h2>
        {isLoading ? (
          <p>Loading...</p>
        ) : (
          <table>
            <thead>
              <tr>
                <th>Order ID</th>
                <th>Amount</th>
                <th>Status</th>
                <th>Date</th>
              </tr>
            </thead>
            <tbody>
              {payments?.payments?.map((payment) => (
                <tr key={payment.id}>
                  <td>{payment.order_id}</td>
                  <td>{payment.amount_usdc} USDC</td>
                  <td>
                    <span className={`status ${payment.status}`}>
                      {payment.status}
                    </span>
                  </td>
                  <td>{new Date(payment.created_at).toLocaleDateString()}</td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
};
```

---

### 7. Main App Setup

**`src/App.jsx`**

```jsx
import React from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Checkout } from './pages/Checkout';
import { Success } from './pages/Success';
import { Dashboard } from './pages/Dashboard';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<Checkout />} />
          <Route path="/success" element={<Success />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  );
}

export default App;
```

---

## 🔥 Advanced Features

### Real-time Payment Updates with Polling

```jsx
import { useEffect, useState } from 'react';
import { chainpeService } from '../services/chainpe';

export const usePaymentPolling = (sessionId, interval = 3000) => {
  const [payment, setPayment] = useState(null);
  const [isPolling, setIsPolling] = useState(true);

  useEffect(() => {
    if (!sessionId || !isPolling) return;

    const poll = async () => {
      try {
        const data = await chainpeService.getSessionStatus(sessionId);
        setPayment(data);

        // Stop polling if paid or expired
        if (data.status === 'paid' || data.status === 'expired') {
          setIsPolling(false);
        }
      } catch (error) {
        console.error('Polling error:', error);
      }
    };

    poll(); // Initial call
    const intervalId = setInterval(poll, interval);

    return () => clearInterval(intervalId);
  }, [sessionId, interval, isPolling]);

  return { payment, isPolling };
};
```

---

### Webhook Listener (Backend Integration)

If your React app has a backend (Next.js, Node.js, etc.):

```javascript
// Express.js example
app.post('/api/webhooks/chainpe', express.json(), (req, res) => {
  const payload = req.body;
  
  // Verify webhook (recommended)
  // const signature = req.headers['x-chainpe-signature'];
  
  switch (payload.type) {
    case 'payment.succeeded':
      // Update your database
      console.log('Payment succeeded:', payload.data);
      // Send confirmation email, update inventory, etc.
      break;
      
    case 'payment.expired':
      console.log('Payment expired:', payload.data);
      break;
  }
  
  res.json({ received: true });
});
```

---

## 🚀 Production Deployment

### Backend (Railway/Heroku/DigitalOcean)

```bash
# Environment variables
DATABASE_URL=postgresql://...
STELLAR_NETWORK=public
STELLAR_HORIZON_URL=https://horizon.stellar.org
CORS_ORIGINS=https://yourstore.com
APP_URL=https://api.yourstore.com
```

### Frontend (Vercel/Netlify)

```bash
# Environment variables
VITE_API_URL=https://api.yourstore.com
VITE_CHAINPE_API_KEY=pk_live_xxxxx
```

### CORS Configuration

Ensure backend `.env` includes your production frontend URL:
```bash
CORS_ORIGINS=https://yourstore.com,https://www.yourstore.com
```

---

## 📊 Testing Checklist

- [ ] Backend running on `localhost:8000`
- [ ] Frontend running on `localhost:3000`
- [ ] CORS configured correctly
- [ ] API key obtained from merchant dashboard
- [ ] Environment variables set
- [ ] Create payment session works
- [ ] Redirect to checkout works
- [ ] QR code displays correctly
- [ ] Payment detection works
- [ ] Webhook received (if configured)
- [ ] Success redirect works
- [ ] Payment verification works

---

## 🆘 Troubleshooting

### CORS Errors
```bash
# Update backend .env
CORS_ORIGINS=http://localhost:3000,http://localhost:5173
```

### API Key Not Working
```bash
# Check header format
X-API-Key: pk_live_xxxxx  # Correct
Api-Key: pk_live_xxxxx     # Wrong
```

### Payment Not Detected
```bash
# Check stellar listener is running
python -m app.services.stellar_listener
```

---

## 📚 Additional Resources

- **API Documentation:** http://localhost:8000/docs
- **Merchant Integration Guide:** [MERCHANT_INTEGRATION.md](./MERCHANT_INTEGRATION.md)
- **Frontend Guide:** [FRONTEND_GUIDE.md](./FRONTEND_GUIDE.md)

---

## 🎉 You're Ready!

Your React app can now accept cryptocurrency payments via ChainPe! 🚀

**Quick Start:**
```bash
# Terminal 1: Start backend
uvicorn app.main:app --reload

# Terminal 2: Start listener
python -m app.services.stellar_listener

# Terminal 3: Start React app
npm run dev
```

Happy coding! 💜
