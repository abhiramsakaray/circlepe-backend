# 🛍️ Merchant Integration Guide

Complete guide for adding "Pay with ChainPe" button to your e-commerce store.

---

## 📋 Table of Contents

1. [Quick Start](#quick-start)
2. [Shopify Integration](#shopify-integration)
3. [WooCommerce Integration](#woocommerce-integration)
4. [Custom Website Integration](#custom-website-integration)
5. [Payment Flow](#payment-flow)
6. [Webhook Configuration](#webhook-configuration)
7. [Testing](#testing)

---

## 🚀 Quick Start

### Step 1: Get Your API Key

1. Sign up at ChainPe Dashboard
2. Navigate to **Settings** → **API Keys**
3. Copy your **Live API Key** (starts with `pk_live_`)

### Step 2: Add the Button Script

Add this script to your website's `<head>` section:

```html
<script src="https://your-chainpe-domain.com/chainpe-button.js"></script>
```

### Step 3: Add the Button

Add this HTML where you want the payment button to appear:

```html
<button id="chainpe-payment-button"></button>
```

### Step 4: Initialize ChainPe

Add this JavaScript code:

```javascript
<script>
  ChainPe.init({
    apiKey: 'your_api_key_here',
    amount: 50.00,  // Amount in USDC
    orderId: 'ORDER-12345',
    successUrl: 'https://yourstore.com/success',
    cancelUrl: 'https://yourstore.com/cart',
    metadata: {
      customerEmail: 'customer@example.com',
      customerName: 'John Doe'
    }
  });
</script>
```

---

## 🛒 Shopify Integration

### Method 1: Using Custom Checkout Button

1. **Go to Shopify Admin** → **Online Store** → **Themes**
2. **Click "Customize"** on your active theme
3. **Add Custom HTML Block** to your checkout page
4. **Paste this code:**

```html
<!-- ChainPe Payment Button -->
<div id="chainpe-container">
  <button id="chainpe-payment-button"></button>
</div>

<script src="https://your-chainpe-domain.com/chainpe-button.js"></script>
<script>
  // Get order details from Shopify Liquid variables
  const orderTotal = {{ checkout.total_price | money_without_currency }};
  const orderId = '{{ checkout.order_id }}';
  
  ChainPe.init({
    apiKey: '{{ settings.chainpe_api_key }}',  // Set in theme settings
    amount: parseFloat(orderTotal),
    orderId: orderId,
    successUrl: 'https://{{ shop.domain }}/pages/payment-success',
    cancelUrl: '{{ checkout.cancel_url }}',
    metadata: {
      customerEmail: '{{ checkout.email }}',
      customerName: '{{ checkout.shipping_address.name }}',
      shopifyOrderId: orderId
    }
  });
</script>
```

### Method 2: Using Shopify App

**Coming Soon:** We're building a native Shopify app for easier integration!

---

## 🔌 WooCommerce Integration

### Step 1: Install Code Snippets Plugin

1. Install **Code Snippets** plugin from WordPress
2. Go to **Snippets** → **Add New**

### Step 2: Add Payment Gateway

Create a new snippet with this code:

```php
<?php
/**
 * ChainPe Payment Gateway for WooCommerce
 */

add_action('plugins_loaded', 'init_chainpe_gateway');

function init_chainpe_gateway() {
    class WC_Gateway_ChainPe extends WC_Payment_Gateway {
        public function __construct() {
            $this->id = 'chainpe';
            $this->method_title = 'ChainPe';
            $this->method_description = 'Accept cryptocurrency payments via Stellar network';
            $this->has_fields = false;
            
            $this->init_form_fields();
            $this->init_settings();
            
            $this->title = $this->get_option('title');
            $this->description = $this->get_option('description');
            $this->api_key = $this->get_option('api_key');
            $this->api_url = 'http://localhost:8000';  // Update to production
            
            add_action('woocommerce_update_options_payment_gateways_' . $this->id, 
                array($this, 'process_admin_options'));
            add_action('woocommerce_api_chainpe_callback', 
                array($this, 'handle_callback'));
        }
        
        public function init_form_fields() {
            $this->form_fields = array(
                'enabled' => array(
                    'title' => 'Enable/Disable',
                    'type' => 'checkbox',
                    'label' => 'Enable ChainPe Payment',
                    'default' => 'yes'
                ),
                'title' => array(
                    'title' => 'Title',
                    'type' => 'text',
                    'default' => 'Cryptocurrency Payment (USDC/XLM)',
                    'desc_tip' => true
                ),
                'description' => array(
                    'title' => 'Description',
                    'type' => 'textarea',
                    'default' => 'Pay with USDC or XLM via Stellar network',
                    'desc_tip' => true
                ),
                'api_key' => array(
                    'title' => 'API Key',
                    'type' => 'text',
                    'description' => 'Get your API key from ChainPe dashboard',
                    'desc_tip' => true
                )
            );
        }
        
        public function process_payment($order_id) {
            $order = wc_get_order($order_id);
            
            // Create ChainPe payment session
            $response = wp_remote_post($this->api_url . '/api/sessions/create', array(
                'headers' => array(
                    'Content-Type' => 'application/json',
                    'X-API-Key' => $this->api_key
                ),
                'body' => json_encode(array(
                    'amount_usdc' => floatval($order->get_total()),
                    'order_id' => $order->get_order_number(),
                    'success_url' => $this->get_return_url($order),
                    'cancel_url' => wc_get_checkout_url(),
                    'metadata' => array(
                        'woocommerce_order_id' => $order_id,
                        'customer_email' => $order->get_billing_email(),
                        'customer_name' => $order->get_billing_first_name() . ' ' . $order->get_billing_last_name()
                    )
                ))
            ));
            
            if (is_wp_error($response)) {
                wc_add_notice('Payment error: ' . $response->get_error_message(), 'error');
                return;
            }
            
            $body = json_decode(wp_remote_retrieve_body($response), true);
            
            if (!isset($body['checkout_url'])) {
                wc_add_notice('Payment error: Unable to create payment session', 'error');
                return;
            }
            
            // Store session ID
            $order->update_meta_data('_chainpe_session_id', $body['session_id']);
            $order->save();
            
            // Redirect to ChainPe checkout
            return array(
                'result' => 'success',
                'redirect' => $body['checkout_url']
            );
        }
        
        public function handle_callback() {
            // Handle webhook callback
            $payload = file_get_contents('php://input');
            $data = json_decode($payload, true);
            
            if (isset($data['metadata']['woocommerce_order_id'])) {
                $order_id = $data['metadata']['woocommerce_order_id'];
                $order = wc_get_order($order_id);
                
                if ($data['status'] === 'paid') {
                    $order->payment_complete($data['tx_hash']);
                    $order->add_order_note('ChainPe payment received. TX: ' . $data['tx_hash']);
                }
            }
            
            http_response_code(200);
            exit;
        }
    }
    
    // Register the gateway
    add_filter('woocommerce_payment_gateways', function($gateways) {
        $gateways[] = 'WC_Gateway_ChainPe';
        return $gateways;
    });
}
?>
```

### Step 3: Configure Settings

1. Go to **WooCommerce** → **Settings** → **Payments**
2. Enable **ChainPe**
3. Enter your **API Key**
4. Save changes

---

## 🌐 Custom Website Integration

### Simple HTML Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>Checkout - My Store</title>
    <script src="https://your-chainpe-domain.com/chainpe-button.js"></script>
</head>
<body>
    <div class="checkout-container">
        <h1>Complete Your Order</h1>
        
        <div class="order-summary">
            <p>Total: $50.00 USDC</p>
        </div>
        
        <!-- ChainPe Payment Button -->
        <button id="chainpe-payment-button"></button>
        
        <!-- Or traditional payment methods -->
        <button onclick="payWithCard()">Pay with Credit Card</button>
    </div>
    
    <script>
        ChainPe.init({
            apiKey: 'pk_live_xxxxxxxxxxxx',
            amount: 50.00,
            orderId: 'ORDER-' + Date.now(),
            successUrl: window.location.origin + '/success.html',
            cancelUrl: window.location.origin + '/cart.html',
            metadata: {
                customerEmail: localStorage.getItem('userEmail'),
                items: JSON.parse(localStorage.getItem('cartItems'))
            }
        });
    </script>
</body>
</html>
```

### Success Page (success.html)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Payment Success</title>
    <script src="https://your-chainpe-domain.com/chainpe-button.js"></script>
</head>
<body>
    <div class="success-container">
        <h1>✅ Payment Successful!</h1>
        <p id="payment-status">Verifying payment...</p>
    </div>
    
    <script>
        // Get session ID from URL parameter
        const urlParams = new URLSearchParams(window.location.search);
        const sessionId = urlParams.get('session_id');
        
        if (sessionId) {
            ChainPe.init({
                apiKey: 'pk_live_xxxxxxxxxxxx'
            });
            
            // Verify payment
            ChainPe.verifyPayment(sessionId)
                .then(session => {
                    document.getElementById('payment-status').innerHTML = `
                        <h2>Order Confirmed!</h2>
                        <p>Transaction Hash: ${session.tx_hash}</p>
                        <p>Amount: ${session.amount_usdc} USDC</p>
                        <p>Order ID: ${session.order_id}</p>
                    `;
                })
                .catch(error => {
                    document.getElementById('payment-status').innerHTML = `
                        <h2>❌ Verification Failed</h2>
                        <p>Please contact support.</p>
                    `;
                });
        }
    </script>
</body>
</html>
```

---

## 🔄 Payment Flow

### Complete User Journey

```
┌─────────────────────────────────────────────────────────────────┐
│                      MERCHANT WEBSITE                           │
│                                                                 │
│  1. Customer clicks "Pay with ChainPe" button                  │
│  2. JavaScript SDK creates payment session via API             │
│  3. Customer redirected to ChainPe checkout page               │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CHAINPE CHECKOUT PAGE                        │
│                                                                 │
│  4. Customer sees order details and QR codes                   │
│  5. Customer chooses USDC or XLM payment method                │
│  6. Customer scans QR with Freighter/Lobstr/Solar wallet       │
│  7. Customer confirms payment in wallet app                    │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STELLAR BLOCKCHAIN                            │
│                                                                 │
│  8. Payment submitted to Stellar network                       │
│  9. ChainPe listener detects payment                           │
│  10. Payment validated and marked as PAID                      │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    WEBHOOK NOTIFICATION                         │
│                                                                 │
│  11. ChainPe sends webhook to merchant's server                │
│  12. Merchant verifies payment and fulfills order              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   REDIRECT TO SUCCESS URL                       │
│                                                                 │
│  13. Customer redirected to merchant's success page            │
│  14. Merchant shows order confirmation                         │
│  15. Journey complete! ✅                                       │
└─────────────────────────────────────────────────────────────────┘
```

### URL Parameters on Redirect

When customer returns to your `success_url`, ChainPe appends:

```
https://yourstore.com/success?session_id=sess_xxxx&status=paid
```

**Parameters:**
- `session_id`: Payment session identifier
- `status`: Payment status (`paid`, `pending`, `expired`)

---

## 🎣 Webhook Configuration

### Set Up Webhook Endpoint

Create an endpoint on your server to receive payment notifications:

```javascript
// Express.js example
app.post('/webhooks/chainpe', express.raw({type: 'application/json'}), (req, res) => {
    const payload = req.body;
    
    // Verify webhook signature (recommended for production)
    // const signature = req.headers['x-chainpe-signature'];
    
    const event = JSON.parse(payload);
    
    switch (event.type) {
        case 'payment.succeeded':
            // Payment successful
            const orderId = event.data.order_id;
            const txHash = event.data.tx_hash;
            
            // Update your database
            updateOrderStatus(orderId, 'paid', txHash);
            
            // Send confirmation email
            sendConfirmationEmail(event.data.metadata.customerEmail);
            break;
            
        case 'payment.expired':
            // Payment session expired
            updateOrderStatus(event.data.order_id, 'expired');
            break;
    }
    
    res.json({received: true});
});
```

### Configure Webhook URL

1. Go to **ChainPe Dashboard** → **Settings** → **Webhooks**
2. Add your webhook URL: `https://yourstore.com/webhooks/chainpe`
3. Select events to receive:
   - ✅ `payment.succeeded`
   - ✅ `payment.expired`
   - ✅ `payment.refunded`

---

## 🧪 Testing

### Test Mode

Use test API keys for development:

```javascript
ChainPe.init({
    apiKey: 'pk_test_xxxxxxxxxxxx',  // Test key
    // ... rest of config
});
```

### Test with Stellar Testnet

1. **Get Test XLM:**
   - Visit: https://laboratory.stellar.org/#account-creator
   - Create test account and fund it

2. **Get Test USDC:**
   ```javascript
   // Add USDC trustline on testnet
   // USDC Issuer: GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5
   ```

3. **Test Payment:**
   - Click "Pay with ChainPe" button
   - Scan QR with Freighter wallet (set to testnet)
   - Confirm payment
   - Verify webhook received

---

## 🔐 Security Best Practices

1. **Never expose API keys in client-side code**
   - Create server-side endpoint to generate sessions
   - Pass only session data to frontend

2. **Validate webhook signatures**
   ```javascript
   const crypto = require('crypto');
   
   function verifyWebhook(payload, signature, secret) {
       const expectedSignature = crypto
           .createHmac('sha256', secret)
           .update(payload)
           .digest('hex');
       
       return crypto.timingSafeEqual(
           Buffer.from(signature),
           Buffer.from(expectedSignature)
       );
   }
   ```

3. **Use HTTPS only in production**

4. **Verify payment status on your server**
   - Don't trust client-side verification alone
   - Always verify via webhook or API call

---

## 📞 Support

- **Documentation:** https://docs.chainpe.io
- **API Reference:** https://docs.chainpe.io/api
- **Email:** support@chainpe.io
- **Discord:** https://discord.gg/chainpe

---

## 🎯 Next Steps

1. ✅ Get your API key
2. ✅ Add button to your store
3. ✅ Configure webhook
4. ✅ Test with testnet
5. ✅ Go live!

**Happy selling! 🚀**
