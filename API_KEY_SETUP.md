# 🔑 Getting Your API Key - Quick Guide

## The Problem

You're seeing these errors:
```
POST http://localhost:8000/api/sessions/create 401 (Unauthorized)
GET http://localhost:8000/merchant/payments?limit=50 403 (Forbidden)
```

This happens because you haven't set up your API key yet.

---

## Solution: Get Your API Key

### Step 1: Login to Dashboard

You've already registered and logged in ✅

### Step 2: Get Your Profile to See API Key

There are two ways:

#### Option A: Via Settings Page (Easiest)

1. Go to Settings page in dashboard
2. Scroll to "API Keys" section
3. Copy your API Key (click the eye icon to reveal it)

#### Option B: Via Browser Console (Quick)

1. Open browser DevTools (F12)
2. Go to Console tab
3. Run this command:
```javascript
fetch('http://localhost:8000/merchant/profile', {
  headers: {
    'Authorization': 'Bearer ' + localStorage.getItem('merchant_token')
  }
})
.then(r => r.json())
.then(data => {
  console.log('Your API Key:', data.api_key);
  console.log('Copy this key to your .env file');
});
```

### Step 3: Update .env File

1. Open your frontend `.env` file
2. Replace the API key line:
```bash
VITE_CHAINPE_API_KEY=your_actual_api_key_here
```

Replace `your_actual_api_key_here` with your real API key.

### Step 4: Restart Dev Server

**Important:** Vite requires restart after .env changes!

```bash
# In your frontend terminal, press Ctrl+C
# Then restart:
npm run dev
```

### Step 5: Test Payment Creation

1. Refresh browser (Ctrl+R)
2. Go to Dashboard → Create Payment
3. Enter amount and submit
4. Should work now! ✅

---

## Why This Happens

The ChainPe API has two types of endpoints:

| Endpoint | Auth Type | Header |
|----------|-----------|--------|
| `/api/sessions/create` | API Key | `X-API-Key: pk_live_xxx` |
| `/merchant/payments` | JWT Token | `Authorization: Bearer eyJ...` |
| `/merchant/profile` | JWT Token | `Authorization: Bearer eyJ...` |

- **API Key** = Used to create payment sessions (public-ish endpoint)
- **JWT Token** = Used to access your merchant data (private endpoints)

You have the JWT token (from login), but need to set the API key for payment creation.

---

## Backend: Generating API Keys for Merchants

### Automatic API Key Generation

API keys are automatically generated when a merchant registers. Check [app/routes/auth.py](app/routes/auth.py#L30-L40).

### Manual API Key Generation

If a merchant doesn't have an API key, run this Python script:

```python
# generate_api_key.py
import secrets
from app.core.database import SessionLocal
from app.models import Merchant

def generate_api_key_for_merchant(merchant_email: str):
    db = SessionLocal()
    try:
        merchant = db.query(Merchant).filter(Merchant.email == merchant_email).first()
        if not merchant:
            print(f"❌ Merchant {merchant_email} not found")
            return
        
        # Generate new API key
        api_key = f"pk_live_{secrets.token_urlsafe(32)}"
        merchant.api_key = api_key
        db.commit()
        
        print(f"✅ API Key generated for {merchant_email}")
        print(f"API Key: {api_key}")
        print(f"\nAdd this to your .env file:")
        print(f"VITE_CHAINPE_API_KEY={api_key}")
        
    finally:
        db.close()

if __name__ == "__main__":
    email = input("Enter merchant email: ")
    generate_api_key_for_merchant(email)
```

Run it:
```bash
python generate_api_key.py
```

---

## Quick Fix Commands

Run these in order:

### 1. Get your API key (Browser Console):
```javascript
localStorage.getItem('merchant_token') && 
fetch('http://localhost:8000/merchant/profile', {
  headers: {'Authorization': 'Bearer ' + localStorage.getItem('merchant_token')}
})
.then(r => r.json())
.then(d => console.log('API Key:', d.api_key));
```

### 2. Update .env:
```bash
# Replace with your actual key
VITE_CHAINPE_API_KEY=pk_live_your_key_here
```

### 3. Restart:
```bash
# Ctrl+C then:
npm run dev
```

---

## Troubleshooting

### "403 Forbidden on merchant/profile"

Your JWT token might be invalid. Try:
1. Logout
2. Login again
3. Retry getting API key

### "Still getting 401 on create payment"

Check:
1. API key is correctly copied (no extra spaces)
2. Dev server was restarted
3. Browser was refreshed
4. Header name is exactly `X-API-Key` (case-sensitive)

### "Cannot read api_key"

Your merchant account might not have an API key yet. Possible causes:
1. Old account created before API key feature
2. Database migration issue

**Fix:** Run the `generate_api_key.py` script above.

### "CORS Error"

Make sure backend `.env` includes your frontend URL:
```bash
CORS_ORIGINS=http://localhost:3000,http://localhost:5173,http://localhost:8000
```

Restart backend after changing this.

---

## Expected Flow After Fix

✅ Login → Get JWT Token  
✅ Access Settings → See API Key  
✅ Update .env → Set API Key  
✅ Restart Server → Load new env  
✅ Create Payment → Works!  
✅ View Payments → Works!  

---

## Security Best Practices

### Development
- ✅ Use `.env` file (never commit to git)
- ✅ Add `.env` to `.gitignore`
- ✅ Use test API keys (`pk_test_xxx`)

### Production
- ✅ Use environment variables (not files)
- ✅ Rotate API keys regularly
- ✅ Use live API keys (`pk_live_xxx`)
- ✅ Never expose API keys in client-side code
- ✅ Create server-side endpoint to proxy payment creation

### Example Production Setup

**Bad (client exposes API key):**
```javascript
// ❌ Don't do this in production
const apiKey = 'pk_live_xxx'; // Exposed in browser
```

**Good (server proxies request):**
```javascript
// ✅ Do this instead
// Frontend calls your backend
fetch('/api/create-payment', {
  method: 'POST',
  body: JSON.stringify({ amount: 50, orderId: 'ORDER-123' })
});

// Your backend endpoint
app.post('/api/create-payment', async (req, res) => {
  // API key stored securely on server
  const response = await fetch('http://localhost:8000/api/sessions/create', {
    headers: { 'X-API-Key': process.env.CHAINPE_API_KEY }
  });
  res.json(await response.json());
});
```

---

## Alternative: Hardcode Temporarily (Dev Only)

**For testing only**, you can hardcode the API key:

1. Open `src/services/api.ts` (or `.js`)
2. Find the request interceptor
3. Temporarily hardcode:
```typescript
api.interceptors.request.use((config) => {
  const apiKey = import.meta.env.VITE_CHAINPE_API_KEY;
  
  if (apiKey && apiKey !== 'your_merchant_api_key_here') {
    config.headers['X-API-Key'] = apiKey;
  } else {
    // ⚠️ TEMPORARY - Replace with your actual key
    config.headers['X-API-Key'] = 'pk_live_paste_your_key_here';
  }
  
  return config;
});
```

**⚠️ Remember to remove this before committing!**

---

## Need Help?

1. Check backend is running: http://localhost:8000/docs
2. Check API key exists: Run browser console command above
3. Check .env file is loaded: `console.log(import.meta.env.VITE_CHAINPE_API_KEY)`
4. Check network tab: Look at request headers (should have `X-API-Key`)

Still stuck? Check the [README_REACT_INTEGRATION.md](README_REACT_INTEGRATION.md) for complete setup guide.
