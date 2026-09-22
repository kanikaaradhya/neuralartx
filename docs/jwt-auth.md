# JWT Authentication — NeuralArtX

This document walks through the full authentication flow: registration, login, token issuance, middleware verification, and frontend token handling.

---

## How It Fits Together

JWT doesn't replace database authentication — it layers on top of it. The flow has two distinct phases:

1. **Identity verification (DB layer):** the user sends credentials, the backend checks them against the MySQL `users` table via a parameterised query. This is the same pattern as traditional session-based auth.
2. **Session management (JWT layer):** once identity is confirmed, the server signs a JWT and sends it to the client. Every subsequent request carries this token — the server validates the signature without querying the database again.

```
Register                          Login                           Protected Request
────────                          ─────                           ─────────────────
                                                                  
Client sends                      Client sends                    Client sends request
username, email,                  username +                      with header:
phone, password                   password                        Authorization: Bearer <token>
       │                                │                                │
       ▼                                ▼                                ▼
┌─────────────┐                  ┌─────────────┐                 ┌──────────────┐
│  Hash pwd   │                  │  Query DB   │                 │ verifyToken  │
│  (bcrypt)   │                  │  for user   │                 │  middleware   │
└──────┬──────┘                  └──────┬──────┘                 └──────┬───────┘
       │                                │                                │
       ▼                                ▼                                │ valid?
┌─────────────┐                  ┌─────────────┐                        │
│  INSERT     │                  │  bcrypt     │                   yes ──┤── no
│  into users │                  │  compare    │                   │         │
└──────┬──────┘                  └──────┬──────┘                   ▼         ▼
       │                                │                        Route    403
       ▼                           match?                       handler  Forbidden
  201 Created                     │         │
                              yes ─┘         └─ no
                               │                 │
                               ▼                 ▼
                        ┌─────────────┐      401
                        │  Sign JWT   │   Unauthorized
                        │  (24h exp)  │
                        └──────┬──────┘
                               │
                               ▼
                        { token: "eyJ..." }
```

## Registration

Password is hashed before it ever touches the database. bcrypt with a cost factor of 10 (standard) means even if the DB is compromised, passwords can't be reversed.

```javascript
const bcrypt = require('bcryptjs');

app.post('/api/register', async (req, res) => {
    const { username, email, phno, password } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);

    connection.query(
        'INSERT INTO users (username, password, phno) VALUES (?, ?, ?)',
        [username, hashedPassword, phno],
        (err, results) => {
            if (err) return res.status(400).json({ error: 'Username already exists' });
            res.status(201).json({ message: 'Account created' });
        }
    );
});
```

**Why bcrypt over plain SHA-256:** SHA-256 is fast by design, which makes brute-forcing easy. bcrypt is deliberately slow (configurable via the cost factor) and salts automatically, so two users with the same password produce different hashes.

## Login — Where Both Layers Meet

This is the critical route. The DB layer (credential check) and the JWT layer (token issuance) happen in sequence:

```javascript
const jwt = require('jsonwebtoken');

app.post('/api/login', (req, res) => {
    const { username, password } = req.body;

    // STEP 1: DB layer — retrieve user and verify credentials
    connection.query(
        'SELECT * FROM users WHERE username = ?',
        [username],
        async (err, results) => {
            if (err || results.length === 0)
                return res.status(401).json({ error: 'Invalid credentials' });

            const user = results[0];
            const match = await bcrypt.compare(password, user.password);
            if (!match)
                return res.status(401).json({ error: 'Invalid credentials' });

            // STEP 2: JWT layer — identity confirmed, issue token
            const token = jwt.sign(
                { username: user.username },
                process.env.JWT_SECRET,
                { expiresIn: '24h' }
            );

            res.json({ token, username: user.username });
        }
    );
});
```

**What's inside the JWT payload:** just `{ username, iat, exp }` — the username for identification, issued-at and expiry timestamps. No sensitive data (password, phone) goes into the token since JWTs are base64-encoded, not encrypted — anyone can decode the payload; the signature only guarantees it wasn't tampered with.

## Token Verification Middleware

Every protected route passes through this middleware. It checks the token's signature and expiry without hitting the database — that's the performance gain over session-based auth.

```javascript
function verifyToken(req, res, next) {
    const header = req.headers['authorization'];
    if (!header)
        return res.status(401).json({ error: 'No token provided' });

    const token = header.split(' ')[1]; // "Bearer <token>"

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;  // makes username available to route handlers
        next();
    } catch (err) {
        return res.status(403).json({ error: 'Invalid or expired token' });
    }
}
```

**Usage on routes:**

```javascript
const { verifyToken } = require('./auth');

// These routes are inaccessible without a valid JWT
app.post('/api/orders', verifyToken, orderHandler);
app.post('/api/payment/create', verifyToken, paymentHandler);
```

## Frontend Token Handling

On login, the token is stored in `localStorage`. On every protected request, it's sent in the `Authorization` header.

```javascript
// After successful login
function handleLoginResponse(data) {
    if (data.token) {
        localStorage.setItem('token', data.token);
        localStorage.setItem('username', data.username);
        // redirect to home / update UI
    }
}

// On any protected request
async function protectedFetch(url, body) {
    const token = localStorage.getItem('token');
    return fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer ' + token
        },
        body: JSON.stringify(body)
    });
}

// Logout — clear the token
function logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('username');
}
```

## Key Design Decisions

- **24-hour expiry** — long enough that a user browsing art doesn't get logged out mid-session, short enough that a stolen token has a limited window.
- **No refresh tokens** — for a storefront with low-sensitivity data, a single access token with 24h expiry is sufficient. A banking app would need refresh tokens; an art marketplace doesn't.
- **`localStorage` over cookies** — the frontend is vanilla JS making explicit fetch calls, not a server-rendered app. `localStorage` is simpler and avoids CSRF concerns that come with cookie-based auth. The trade-off is XSS vulnerability, which is mitigated by not storing sensitive data in the token payload.
- **Generic error messages** — both "user not found" and "wrong password" return the same `Invalid credentials` message. This prevents username enumeration attacks.
- **Parameterised queries** — `?` placeholders in every SQL query prevent SQL injection regardless of input.

---

← Back to [README](../README.md) · Next: [Payment Integration →](payment-integration.md)
