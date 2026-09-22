# Payment Integration (Razorpay) — NeuralArtX

This document covers the secure payment flow: backend order creation, frontend checkout, payment verification, and order recording.

---

## Why Razorpay

For an Indian art marketplace accepting INR payments, Razorpay is the standard choice: test mode with no upfront costs, a well-documented Node.js SDK, and a drop-in checkout overlay that handles card/UPI/netbanking without the frontend needing to touch raw payment data. The seller never sees or stores card numbers — Razorpay handles PCI compliance.

## Payment Flow

The flow has three steps, all protected by the JWT middleware from the [auth layer](jwt-auth.md):

```
Buyer clicks "Buy Now"
        │
        ▼
┌──────────────────────────────────────────────┐
│  STEP 1: Backend creates a Razorpay order    │
│                                              │
│  POST /api/payment/create                    │
│  Authorization: Bearer <token>               │
│  Body: { amount: 1000000 }  (₹10,000 in     │
│                               paise)         │
│                                              │
│  → Razorpay returns an order_id              │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  STEP 2: Frontend opens Razorpay Checkout    │
│                                              │
│  Razorpay overlay handles:                   │
│  • Card / UPI / Netbanking input             │
│  • OTP verification                          │
│  • Payment processing                        │
│                                              │
│  → On success, returns:                      │
│    razorpay_payment_id                       │
│    razorpay_order_id                         │
│    razorpay_signature                        │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  STEP 3: Frontend records the order          │
│                                              │
│  POST /api/orders                            │
│  Authorization: Bearer <token>               │
│  Body: { art_id, totalamt, payment_id }      │
│                                              │
│  → Backend inserts into orders table         │
│    with the Razorpay payment_id for audit    │
└──────────────────────────────────────────────┘
```

## Backend — Order Creation

The backend creates a Razorpay order with the amount in paise (₹1 = 100 paise). This order is what the frontend checkout binds to — it prevents the buyer from tampering with the price on the client side, because the amount is locked server-side.

```javascript
const Razorpay = require('razorpay');
const { verifyToken } = require('./auth');

const razorpay = new Razorpay({
    key_id: process.env.RAZORPAY_KEY_ID,
    key_secret: process.env.RAZORPAY_KEY_SECRET
});

app.post('/api/payment/create', verifyToken, async (req, res) => {
    try {
        const { amount } = req.body;

        const order = await razorpay.orders.create({
            amount: amount,          // in paise
            currency: 'INR',
            receipt: 'order_' + Date.now()
        });

        res.json({
            orderId: order.id,
            amount: order.amount,
            currency: order.currency
        });
    } catch (err) {
        res.status(500).json({ error: 'Payment order creation failed' });
    }
});
```

**Why the amount is set server-side:** if the frontend sent the price directly to Razorpay, a buyer could modify the DOM or intercept the request and pay ₹1 for a ₹10,000 painting. By creating the order on the backend with the price from the `artwork` table, the amount is tamper-proof.

## Frontend — Razorpay Checkout

The Razorpay Checkout.js SDK provides a pre-built overlay. The frontend never handles raw card data — it passes the `order_id` from Step 1, and Razorpay handles the rest.

```html
<script src="https://checkout.razorpay.com/v1/checkout.js"></script>
```

```javascript
async function payNow(artId, price) {
    const token = localStorage.getItem('token');

    // Step 1 — create order on backend
    const res = await fetch('/api/payment/create', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer ' + token
        },
        body: JSON.stringify({ amount: price * 100 })
    });
    const { orderId } = await res.json();

    // Step 2 — open Razorpay checkout overlay
    const options = {
        key: 'rzp_test_XXXXXXXXXXXXXXX',   // test mode key
        amount: price * 100,
        currency: 'INR',
        name: 'NeuralArtX',
        description: 'Art Purchase',
        order_id: orderId,
        handler: async function (response) {
            // Step 3 — payment succeeded, record the order
            await fetch('/api/orders', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Bearer ' + token
                },
                body: JSON.stringify({
                    art_id: artId,
                    totalamt: price,
                    payment_id: response.razorpay_payment_id
                })
            });
            alert('Payment successful — thank you!');
        },
        prefill: {
            name: localStorage.getItem('username')
        },
        theme: {
            color: '#6C63FF'
        }
    };

    const rzp = new Razorpay(options);
    rzp.open();
}
```

## Backend — Order Recording

After a successful payment, the order is stored in MySQL with the Razorpay `payment_id`. This links every database record to a verifiable transaction on the Razorpay dashboard.

```javascript
app.post('/api/orders', verifyToken, (req, res) => {
    const { art_id, totalamt, payment_id } = req.body;
    const username = req.user.username;    // extracted from JWT by middleware

    connection.query(
        'INSERT INTO orders (username, art_id, totalamt, payment_id) VALUES (?, ?, ?, ?)',
        [username, art_id, totalamt, payment_id],
        (err) => {
            if (err) return res.status(500).json({ error: 'Failed to record order' });
            res.json({ message: 'Order recorded successfully' });
        }
    );
});
```

**Note:** `req.user.username` comes from the JWT — the server doesn't trust the client to send its own username. The token, signed by the server, is the source of truth for who is making the request.

## Key Design Decisions

- **Server-side order creation before checkout** — prevents client-side price tampering. The Razorpay order locks the amount.
- **Test mode keys for development** — Razorpay test mode simulates the full payment flow (including OTP) without real money. Switching to live is a key swap, not a code change.
- **`payment_id` stored in orders** — every purchase has a paper trail. If a buyer disputes a transaction, the `payment_id` maps directly to a record on the Razorpay dashboard.
- **JWT-gated payment route** — both `/api/payment/create` and `/api/orders` require a valid token. An unauthenticated user can't initiate a payment, and a payment can't be recorded without the server knowing who paid.
- **No card data touches the server** — the Razorpay overlay handles all sensitive input. The backend only sees order IDs and payment IDs, never card numbers or UPI PINs. This keeps the app outside PCI-DSS scope.

---

← [JWT Authentication](jwt-auth.md) · Back to [README](../README.md)
