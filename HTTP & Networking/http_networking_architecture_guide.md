# HTTP & Networking: Production-Grade Architecture
## Deep Principles for Building Secure, Fast, Reliable Systems

---

## Part 1: Understanding HTTP as a Protocol

### What Is HTTP Really?

HTTP (HyperText Transfer Protocol) is a **communication contract** between two machines:
- **Client** (browser, mobile app, another server)
- **Server** (your backend)

**Key insight:** HTTP is stateless and text-based. Every request is independent. No permanent connection by default.

### The Request/Response Cycle

Let's trace exactly what happens when you access a website:

```
STEP 1: CLIENT INITIATES
────────────────────────
Browser: "I want /api/users/123"
(Prepares a REQUEST)

REQUEST (what the browser sends):
┌──────────────────────────────────────┐
│ GET /api/users/123 HTTP/1.1          │ ← Method, Path, Protocol
│ Host: api.example.com                │ ← Where to send it
│ Accept: application/json             │ ← What format I want
│ Authorization: Bearer token...       │ ← Authentication
│ Content-Type: application/json       │ ← Body format (if POST)
│ User-Agent: Mozilla/5.0              │ ← Browser info
│                                      │
│ (Optional body for POST/PUT)         │ ← JSON, form data, etc.
│ { "data": "..." }                    │
└──────────────────────────────────────┘


STEP 2: NETWORK JOURNEY
────────────────────────
Browser → [Internet] → Your Server
(Thousands of hops across the internet)

Client-side processing time: ~10-50ms (network latency)


STEP 3: SERVER PROCESSES
────────────────────────
Server: "Received GET /api/users/123"
Server: "Validating Authorization header..."
Server: "Querying database for user 123..."
Server: "Preparing response..."

Processing time: Variable (1-1000ms depending on operation)


STEP 4: SERVER RESPONDS
────────────────────────
RESPONSE (what the server sends back):
┌──────────────────────────────────────┐
│ HTTP/1.1 200 OK                      │ ← Status code
│ Content-Type: application/json       │ ← What format
│ Content-Length: 256                  │ ← Size of body
│ Cache-Control: max-age=3600          │ ← Caching instructions
│ Set-Cookie: session=abc123; Path=/   │ ← Store session
│ X-Security-Token: xyz789             │ ← Custom headers
│                                      │
│ { "id": 123, "name": "John", ... }   │ ← Response body
└──────────────────────────────────────┘

Server processing time: Usually 50-500ms


STEP 5: NETWORK RETURN JOURNEY
────────────────────────────────
Server → [Internet] → Browser

Return time: ~10-50ms


STEP 6: BROWSER PROCESSES
────────────────────────
Browser: "Status 200 = Success"
Browser: "Content-Type is JSON, parse it"
Browser: "Store the session cookie"
Browser: "Cache this response for 1 hour"
Browser: "Update the page"
Browser: "Done"

Total time: 50-150ms for simple requests, 1-5s for complex operations
```

### Why This Matters for Production

Every part of this cycle affects:
- **Security**: Headers determine if request is authenticated
- **Performance**: Caching headers determine if response is reused
- **Reliability**: Status codes tell client if something failed
- **User experience**: Total time affects perceived performance

---

## Part 2: HTTP Methods — Semantics Matter

### Understanding Method Semantics

HTTP methods tell the server **what you're trying to do**, not just for organization but for **semantic meaning** that affects caching, security, and client behavior.

```
GET: Retrieve data (SAFE, IDEMPOTENT)
─────────────────────────────────────
• Don't modify anything
• Can be cached
• Can be repeated safely (same result each time)
• Safe for prefetching
• Should never change server state

Examples:
GET /api/users           → Get list of users
GET /api/users/123       → Get specific user
GET /api/health          → Check if server is alive

Browser behavior:
• Can be cached by browser
• Browser can auto-retry on failure
• Browser can prefetch (for performance)
• Can include query parameters

❌ DON'T DO THIS:
GET /api/users/123/delete  ← Using GET to modify is WRONG
  → Violates HTTP semantics
  → Browser might prefetch it (accidentally deleting!)
  → Can't be cached safely


POST: Create new resource (NOT SAFE, NOT IDEMPOTENT)
──────────────────────────────────────────────────
• Creates something new
• Each request might create a duplicate (idempotency problem)
• Cannot be cached (by default)
• Includes a body (data to create)

Examples:
POST /api/users          → Create new user
POST /api/login          → Authenticate user
POST /api/orders         → Create new order

Idempotency problem:
  Request 1: POST /api/orders { "items": [...] }
    ↓ Creates order #1001
    ✗ Network timeout before response
  
  Request 2: POST /api/orders { "items": [...] }  (retry)
    ↓ Creates order #1002
    ✗ Now user has 2 orders for 1 action!

Solution: Idempotency keys (will discuss later)


PUT: Replace entire resource (IDEMPOTENT if used correctly)
────────────────────────────────────────────────────────
• Replace the ENTIRE resource
• Can be safely repeated (same result)
• Use for complete replacement, not partial updates

Example:
PUT /api/users/123 { "name": "John", "email": "john@example.com", "age": 30 }
  ↓ Replaces entire user object
  ↓ If called 3 times, result is same (idempotent)

⚠️ Gotcha:
• If body is missing required fields, resource becomes incomplete
• Use PATCH for partial updates instead


PATCH: Partially update resource (IDEMPOTENT)
──────────────────────────────────────────────
• Update ONLY the fields provided
• Leave other fields unchanged
• Can be safely repeated

Example:
PATCH /api/users/123 { "name": "John" }
  ↓ Only updates name, leaves email/age unchanged
  ↓ Safe to repeat


DELETE: Remove resource (IDEMPOTENT)
────────────────────────────────────
• Removes the resource
• Can be safely repeated (deleting twice = same result)
• No body (usually)

Example:
DELETE /api/users/123
  ↓ Deletes user 123
  DELETE /api/users/123 (again)
  ↓ Still deleted (idempotent)


HEAD: Like GET but no body
──────────────────────────
• Get headers and status code
• No response body (saves bandwidth)
• Used for checking if resource exists or getting metadata

Example:
HEAD /api/users/123
  Response: 200 OK (headers only, no body)
  ↓ Know resource exists without transferring data


OPTIONS: Describe communication options
───────────────────────────────────────
• Returns what methods are allowed
• Used for CORS preflight (will discuss)

Example:
OPTIONS /api/users
  Response: Allow: GET, POST, PUT, DELETE
```

### Idempotency — A Critical Concept

**Idempotent = Safe to repeat with same result**

This is CRUCIAL for building reliable systems:

```
IDEMPOTENT (Safe to retry):
──────────────────────────
GET /api/users/123
  ↓ Returns user 123
  ↓ Retry → Still returns user 123
  ✓ Safe

PUT /api/users/123 { "name": "John" }
  ↓ Sets name to "John"
  ↓ Retry → Still sets name to "John"
  ✓ Safe

DELETE /api/users/123
  ↓ Deletes user 123
  ↓ Retry → Still deleted
  ✓ Safe


NOT IDEMPOTENT (Dangerous to retry):
───────────────────────────────────
POST /api/orders { "amount": 100 }
  ↓ Creates order #1001 for $100
  ↓ Retry (network error, client retries)
  ↓ Creates order #1002 for $100
  ✗ Dangerous! User charged twice!
  
POST /api/payments { "amount": 100 }
  ↓ Charges credit card $100
  ↓ Retry → Charges $100 again
  ✗ Catastrophic!


Why This Matters in Production:
─────────────────────────────
• Network can be unreliable
• Client might not get response, retries request
• If method is not idempotent, retry causes duplicate action
• Especially bad for payments, orders, transactions

Solution: Use idempotency keys (explained later)
```

---

## Part 3: Status Codes — The Server's Answer

Status codes tell the client: "Did I successfully do what you asked?"

### Understanding Status Code Categories

```
1xx (Informational): "I'm processing, hold on"
─────────────────────────────────────────────
100 Continue: "Send the request body"
  (Used when client checks before sending large body)

⚠️ Rare in modern APIs, mostly unused


2xx (Success): "I did what you asked"
─────────────────────
200 OK: "Success, here's data"
  GET /api/users → 200 { users: [...] }
  PUT /api/users/123 → 200 { id: 123, ... }
  DELETE /api/users/123 → 200 { deleted: true }

201 Created: "I created a new resource"
  POST /api/users → 201 { id: new_id, name: "John" }
  Location: /api/users/123 ← Newly created resource location
  
  ⚠️ Important: Return the newly created resource
  ⚠️ Include Location header with new resource URL

202 Accepted: "I accepted it, processing asynchronously"
  POST /api/bulk-process → 202
  Location: /api/jobs/job-123 ← Check status here
  
  Used for long operations:
  - Bulk imports
  - Video encoding
  - Report generation
  
  Client must poll the Job endpoint to check progress

204 No Content: "Success, no body to return"
  DELETE /api/users/123 → 204
  (No response body, just headers)
  
  Rare in modern APIs, most return 200 with empty body


3xx (Redirect): "Look somewhere else"
──────────────────────────────────────
301 Moved Permanently: "Resource moved, update your bookmarks"
  GET /api/users (old path) → 301 Moved Permanently
  Location: /api/v2/users (new path)
  
  Browsers/clients cache this decision
  Used for API versioning migration

302 Found: "Temporarily redirected"
  Similar to 301 but temporary
  Not cached

304 Not Modified: "You already have the latest version"
  GET /api/users (with If-None-Match header)
  → 304 Not Modified (no body)
  
  Browser uses cached copy
  Saves bandwidth
  Critical for performance


4xx (Client Error): "You did something wrong"
──────────────────────────────────────────────
400 Bad Request: "Request malformed or invalid"
  POST /api/users { invalid JSON }
  → 400 Bad Request
  
  { "error": "Invalid JSON syntax" }
  
  Client must fix request and retry

401 Unauthorized: "You must authenticate"
  GET /api/private → No auth header
  → 401 Unauthorized
  
  { "error": "Missing Authorization header" }
  
  Client should redirect to login

403 Forbidden: "You're authenticated but not authorized"
  GET /api/admin → Authenticated as regular user
  → 403 Forbidden
  
  { "error": "Insufficient permissions" }
  
  Different from 401: User is authenticated, just lacks permissions

404 Not Found: "Resource doesn't exist"
  GET /api/users/999 (user doesn't exist)
  → 404 Not Found
  
  { "error": "User not found" }

409 Conflict: "Can't fulfill request due to conflict"
  PUT /api/users/123 { version: 5 } but current version is 10
  → 409 Conflict
  
  { "error": "Version mismatch, refetch and retry" }
  
  Used for optimistic locking (will discuss)

429 Too Many Requests: "Rate limited, back off"
  60 requests in 1 minute to rate-limited endpoint
  → 429 Too Many Requests
  
  { "error": "Rate limit exceeded" }
  X-RateLimit-Reset: 2025-03-10T10:00:00Z
  
  Critical for production: Protects against abuse


5xx (Server Error): "I (server) screwed up"
──────────────────────────────────────────
500 Internal Server Error: "Unexpected error, not your fault"
  Server code threw an unhandled exception
  → 500 Internal Server Error
  
  { "error": "Internal server error" }
  
  Client can retry (might be transient)

502 Bad Gateway: "Server upstream is down"
  Your server tried calling another service, it's down
  → 502 Bad Gateway
  
  Client should retry

503 Service Unavailable: "Server is overloaded or down"
  Server is restarting, traffic too high
  → 503 Service Unavailable
  
  Retry-After: 60 (seconds to wait before retry)
  
  Client should back off and retry
```

### Practical Status Code Rules for AI Review

```typescript
// ✓ CORRECT - Uses right status codes
function handleCreateUser(req: Request, res: Response) {
    const user = createUser(req.body);
    res.status(201).json({        // ← 201 Created
        id: user.id,
        ...user
    });
    res.set('Location', `/api/users/${user.id}`);  // ← Include location
}

// ❌ WRONG - Wrong status codes
function handleCreateUser(req: Request, res: Response) {
    const user = createUser(req.body);
    res.status(200).json(user);   // ← Should be 201, not 200
}

// ✓ CORRECT - Proper error handling
function handleGetUser(req: Request, res: Response) {
    const user = User.findById(req.params.id);
    
    if (!user) {
        return res.status(404).json({
            error: 'User not found'
        });
    }
    
    res.status(200).json(user);
}

// ❌ WRONG - Returns success even when not found
function handleGetUser(req: Request, res: Response) {
    const user = User.findById(req.params.id);
    res.status(200).json(user);   // ← If user is null, still 200
}

// ✓ CORRECT - Handles rate limiting
function handleRequest(req: Request, res: Response) {
    if (isRateLimited(req)) {
        res.status(429)
            .set('Retry-After', '60')
            .json({ error: 'Rate limited' });
        return;
    }
    
    // Process request...
}

// ✓ CORRECT - Proper auth error distinction
function handlePrivateRequest(req: Request, res: Response) {
    if (!req.headers.authorization) {
        return res.status(401).json({
            error: 'Authentication required'
        });
    }
    
    const user = verifyToken(req.headers.authorization);
    if (!user.hasPermission('admin')) {
        return res.status(403).json({
            error: 'Insufficient permissions'
        });
    }
    
    // Process request...
}
```

---

## Part 4: Headers — The Metadata Layer

Headers are **metadata about the request/response**. They tell both client and server how to handle the body.

### Request Headers (Client → Server)

```
Host: api.example.com
  Required: Tells server which domain the request is for
  Critical for virtual hosting (multiple domains on one server)

Accept: application/json
  Tells server: "I want response in JSON format"
  Server might support multiple formats (JSON, XML, HTML)
  Server picks the best match

Content-Type: application/json
  "The body I'm sending is JSON"
  Server knows how to parse the body

Authorization: Bearer <token>
  "Here's proof I'm authenticated"
  Critical for security
  Always use HTTPS with Authorization header

User-Agent: Mozilla/5.0 (Chrome/120)
  What browser/client is making the request
  Used for analytics, debugging
  Can be spoofed, don't trust for security

X-Request-ID: abc-123-xyz
  Unique ID for this request
  Helps with logging and tracing
  Correlates request across multiple services

Referer: https://example.com/page
  What page the request came from
  Can be spoofed, use for analytics only

X-Forwarded-For: 192.168.1.1
  Client's real IP (when behind proxy/load balancer)
  Used to identify client IP
  Can be spoofed by client, verify with proxy

If-None-Match: "abc123def"
  Conditional request: "Only send if body changed"
  Browser caches this ETag from previous response
  Server responds 304 if unchanged (saves bandwidth)

Cookie: session=xyz789
  Browser automatically sends cookies
  Used for session management (old method)
  Now replaced by Authorization headers (tokens)
```

### Response Headers (Server → Client)

```
Content-Type: application/json
  "The body I'm sending is JSON"
  Browser knows how to parse it

Content-Length: 1024
  "The body is 1024 bytes"
  Browser knows when download is complete
  Used for progress bars

Cache-Control: max-age=3600, public
  "Cache this response for 3600 seconds (1 hour)"
  Browser stores response locally
  Next request for same URL returns cached copy
  Critical for performance

  Directives:
  • max-age=N: Cache for N seconds
  • no-cache: Revalidate before use (check with server if changed)
  • no-store: Don't cache at all
  • public: Anyone can cache (CDN, proxy, browser)
  • private: Only browser can cache (not CDN)

ETag: "abc123def"
  "Hash of the response body"
  Browser stores this with cached response
  Next request: Browser sends If-None-Match with this value
  Server compares: If same, responds 304 (not modified)
  If different, sends full response (200)
  
  Saves bandwidth: Don't resend body if unchanged

Set-Cookie: session=xyz789; Path=/; HttpOnly; Secure
  "Store this cookie on the browser"
  Browser automatically sends it with future requests
  
  Flags:
  • HttpOnly: Only sent to server, not accessible to JavaScript
    (Prevents XSS attacks)
  • Secure: Only sent over HTTPS
    (Prevents man-in-the-middle)
  • SameSite=Strict: Only sent to same domain
    (Prevents CSRF attacks)
  • Path=/api: Only sent to /api/* paths
  • Domain=example.com: Sent to all subdomains

Location: /api/users/123
  "Newly created resource is here"
  Used with 201 Created or 3xx redirects
  Browser might auto-follow (for 3xx)

Retry-After: 60
  "Wait 60 seconds before retrying"
  Used with 429 (rate limited) or 503 (service unavailable)
  Client backs off, then retries

X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 2025-03-10T10:00:00Z
  "You have 1000 requests/hour, 999 remaining, resets at 10:00"
  Helps client know how many requests left
  Client can optimize or warn user

Access-Control-Allow-Origin: https://example.com
  "Allow requests from example.com domain"
  CORS header (will discuss later)

Strict-Transport-Security: max-age=31536000
  "Only use HTTPS for all future requests to this domain"
  Browser enforces HTTPS, not HTTP
  Critical for security

X-Content-Type-Options: nosniff
  "Don't guess the content type, trust my Content-Type header"
  Prevents browser from executing scripts if misidentified as HTML
  Security header

X-Frame-Options: DENY
  "Don't allow this page to be loaded in an iframe"
  Prevents clickjacking attacks
  Security header

Content-Security-Policy: "default-src 'self'; script-src 'self' cdn.example.com"
  "Only load resources from my domain or trusted CDN"
  Prevents XSS attacks
  Critical security header
```

### Which Headers Matter Most for Production

```
CRITICAL (Always include these):
────────────────────────────────
✓ Authorization header for auth endpoints
✓ Content-Type: application/json
✓ Cache-Control for performance
✓ X-Content-Type-Options: nosniff
✓ Strict-Transport-Security for HTTPS
✓ Content-Security-Policy for security

IMPORTANT (Include for better systems):
────────────────────────────────────────
✓ X-Request-ID for tracing
✓ Set-Cookie with HttpOnly flag
✓ ETag for caching efficiency
✓ Retry-After for rate limiting
✓ Access-Control headers for CORS

NICE TO HAVE (Analytics and debugging):
─────────────────────────────────────
• X-RateLimit-* headers
• X-Response-Time header
• Server header (what server software)
```

---

## Part 5: Cookies vs. Tokens — Authentication Methods

This is one of the most misunderstood aspects of HTTP. Let's dive deep.

### Cookies: Session-Based Authentication

```
HOW COOKIES WORK:
─────────────────

CLIENT                                    SERVER
──────                                    ──────
1. Login request:
   POST /login { email, password }
                  ───────────────────────>
                                          Validate credentials
                                          Create session
                                          Store session in DB/Redis
                                          Generate session ID

                                    Response: 200 OK
                  <─────────────────────────
   Set-Cookie: session=xyz789; HttpOnly; Secure

2. Browser automatically stores cookie

3. Future requests:
   GET /api/private
   (Browser auto-adds Cookie header)
   Cookie: session=xyz789
                  ───────────────────────>
                                          Verify session exists
                                          Check permissions
                                          Return protected data
                                    <─────────────────────────
   Response: 200 { data: ... }


STATE ON SERVER:
────────────────
Session store (Redis/DB):
  session:xyz789 → {
    userId: 123,
    email: "user@example.com",
    permissions: ["read", "write"],
    createdAt: 2025-03-10T09:00:00Z,
    expiresAt: 2025-03-11T09:00:00Z
  }

Server maintains state for each session.


KEY CHARACTERISTICS:
───────────────────
• Stateful: Server stores session data
• Server-side invalidation: Server deletes session = user logged out
• Automatic: Browser includes cookie automatically
• Domain-scoped: Only sent to same domain (by default)
• Revocable: Server can immediately revoke access
• Session fixation: Cookie value is just an ID, attacker can't forge


LOGOUT:
───────
POST /logout
                  ───────────────────────>
                                          Delete session from store
                                          Send Set-Cookie with max-age=0
                                    <─────────────────────────────
   Set-Cookie: session=xyz789; max-age=0
   
   Browser deletes cookie
   Access immediately revoked
```

### Tokens: Stateless Authentication

```
HOW TOKENS WORK (JWT - JSON Web Token):
───────────────────────────────────────

CLIENT                                    SERVER
──────                                    ──────
1. Login request:
   POST /login { email, password }
                  ───────────────────────>
                                          Validate credentials
                                          Create JWT token
                                          Sign with secret key
                                          Return token (no storage)

                                    Response: 200 OK
                  <─────────────────────────
   { "token": "eyJhbGciOiJIUzI1NiIs..." }

2. Browser stores token (localStorage, sessionStorage, or cookie)
   JavaScript: localStorage.setItem('token', token)

3. Future requests:
   GET /api/private
   Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
                  ───────────────────────────────>
                                          Verify JWT signature
                                          Check expiration
                                          Extract user ID
                                          Check permissions
                                          Return protected data
                                    <─────────────────────────────
   Response: 200 { data: ... }


STATE ON SERVER:
────────────────
NOTHING! Server doesn't store anything.

Token contains all info:
  Header: { "alg": "HS256", "typ": "JWT" }
  Payload: { "userId": 123, "email": "...", "permissions": [...] }
  Signature: HMACSHA256(header + payload + secret)

Server only needs the secret key to verify.


KEY CHARACTERISTICS:
───────────────────
• Stateless: Server stores no session data
• No server-side invalidation: Token valid until expiration
• Manual: Must include Authorization header (JavaScript)
• Not domain-scoped: Can be used across domains/subdomains
• Not immediately revocable: Token valid until expiration
• Forgeable: If attacker knows secret, can create fake tokens


LOGOUT PROBLEM:
───────────────
Token expires in 1 hour.
User clicks logout.
Token still valid for 59 minutes!

Solutions:
1. Use short expiration (15 min) + refresh tokens (24 hour)
2. Keep blacklist of revoked tokens (defeats "stateless" purpose)
3. Accept that logout isn't immediate
```

### Cookies vs. Tokens: Comprehensive Comparison

```
ASPECT              COOKIES                 TOKENS (JWT)
──────────────────────────────────────────────────────
Architecture        Stateful (server)       Stateless (no server)

Auto-inclusion      Yes (browser)           No (manual, header)

Domain scope        Domain-specific         Not domain-specific

Revocation          Immediate               Delayed (until expiry)

Storage             HTTPOnly (secure)       localStorage (vulnerable)

Replay attack       Session ID can't        Token can be replayed
                    be forged               if not HTTPS

XSS vulnerability   HTTPOnly prevents it    JavaScript access
                                            makes it vulnerable

CSRF vulnerability  Possible (auto-sent)    Protected (header-based)

Multi-domain        Difficult               Easy

Mobile apps         Difficult               Easy

API usage           Common                  Standard

Scaling             Need shared session     No coordination needed
                    store (Redis)           (stateless)

Instant logout      Yes                     No (need refresh token)


WHEN TO USE COOKIES:
────────────────────
• Traditional web apps (browser-based)
• Single domain/app
• Need immediate logout
• User-friendly (auto-sent)
• Trust browser security

✓ JKUATHub web frontend: Use HttpOnly cookies


WHEN TO USE TOKENS:
──────────────────
• Modern SPAs (Single Page Apps)
• Multiple domains/services
• Mobile apps
• API-first architecture
• Cross-origin requests
• Microservices

✓ JKUATHub mobile app: Use JWT tokens
✓ JKUATHub API: Support both (flexibility)
```

### Hybrid Approach: Refresh Tokens

This solves both worlds:

```
STRATEGY: Short-lived access token + long-lived refresh token
─────────────────────────────────────────────────────────────

LOGIN:
────────
POST /auth/login { email, password }
  ↓
Server creates:
  • Access Token: expires in 15 minutes
  • Refresh Token: expires in 7 days (stored in secure cookie or DB)

Response:
{
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",  ← Short-lived
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",  ← Long-lived
    "expiresIn": 900  ← 15 minutes in seconds
}


NORMAL REQUEST (within 15 min):
──────────────────────────────
Authorization: Bearer {accessToken}
  ↓
Server: Token valid, return data


REQUEST AFTER 15 MINUTES:
─────────────────────────
Authorization: Bearer {expiredAccessToken}
  ↓
Server: Token expired (401 Unauthorized)
Client: Token expired, need new one

POST /auth/refresh { refreshToken: ... }
  ↓
Server: Verify refresh token valid
Response: { "accessToken": "new_token", "expiresIn": 900 }

Client: Store new access token
Authorization: Bearer {newAccessToken}
  ↓
Server: Token valid, return data


AFTER 7 DAYS (refresh token expires):
──────────────────────────────────────
User has to login again
GET new access + refresh tokens


LOGOUT:
───────
POST /logout { refreshToken: ... }
  ↓
Server: Blacklist/invalidate refresh token
User: Can't get new access tokens anymore


ADVANTAGES:
───────────
• Access token is short-lived (small window if compromised)
• Refresh token can be revoked (immediate logout)
• Stateless access checks (no DB query for every request)
• Can rotate refresh tokens for extra security
• Mobile-friendly (can store tokens securely)


IMPLEMENTATION:
───────────────
// In Authorization header middleware
function verifyAccessToken(token) {
    try {
        return jwt.verify(token, SECRET_KEY);  // 15 min expiry
    } catch (err) {
        if (err.name === 'TokenExpiredError') {
            throw new TokenExpiredError('Access token expired');
        }
        throw new InvalidTokenError('Invalid token');
    }
}

// On token expiry, client calls refresh endpoint
app.post('/auth/refresh', (req, res) => {
    const { refreshToken } = req.body;
    
    try {
        const payload = jwt.verify(refreshToken, REFRESH_SECRET_KEY);
        
        // Check if refresh token is blacklisted
        if (isBlacklisted(refreshToken)) {
            return res.status(401).json({ error: 'Token revoked' });
        }
        
        // Generate new access token
        const newAccessToken = jwt.sign(
            { userId: payload.userId },
            SECRET_KEY,
            { expiresIn: '15m' }
        );
        
        res.json({
            accessToken: newAccessToken,
            expiresIn: 900
        });
    } catch (err) {
        res.status(401).json({ error: 'Invalid refresh token' });
    }
});
```

---

## Part 6: Caching — Performance at Scale

Caching is one of the most complex and misunderstood aspects of HTTP. It directly impacts performance and user experience.

### HTTP Caching: Three Layers

```
USER BROWSER                CDN / PROXY              YOUR SERVER
────────────               ──────────────           ─────────────

GET /api/data (first request)
                    ────────────────────────────────────>
                                                    Return data
                                                    Cache-Control: max-age=3600
                    <──────────────────────────────────
Store in:
- Browser cache
- CDN cache


GET /api/data (within 1 hour)
Store browser cache: Has it? Return immediately
  ↓ No server request needed
  ↓ No network latency
  ↓ Instant response


GET /api/data (after 1 hour)
Browser cache expired, but might still be fresh
Send request with:
If-None-Match: "abc123"
  (ETag from cached response)
                    ────────────────────────────────>
                                                    Check: Has body changed?
                                                    No change detected
                                                    Response: 304 Not Modified
                    <──────────────────────────────
Browser: Use cached body (from previous response)
  ↓ Network request happened but no body transferred
  ↓ Bandwidth saved
  ↓ Still fast


SECOND VISITOR TO SAME DATA:
───────────────────────────
GET /api/data
              Check CDN cache: Has it?
              → Yes, return from CDN
              ↓ No need to reach origin server
              ↓ Ultra-fast (CDN is geographically close)


VISITOR IN DIFFERENT COUNTRY:
───────────────────────────
GET /api/data (same URL)
              Check CDN cache (different CDN node)
              → Yes, has it from previous visitor's request
              ↓ Origin server not hit
              ↓ Data served from nearest CDN
              ↓ Global performance improvement
```

### Cache-Control Directives

```
DIRECTIVE           MEANING                     USE CASE
──────────────────────────────────────────────────────────
public              Anyone can cache            Public data
                    (browser, CDN, proxy)

private             Only browser caches         Personal data
                    (not CDN, not proxy)        (logged-in user data)

no-cache            Don't use cached copy       Data might change
                    without validating          but can check freshness
                    (check ETag/Last-Modified)

no-store            Don't cache at all          Sensitive data
                                                (passwords, tokens)

max-age=N           Cache for N seconds         Static files: 86400
                                                API responses: 60-3600

immutable           Response never changes      Versioned assets
                    Safe to cache forever       (app.abc123.js)

must-revalidate     Can't use stale copy        Important data
                    Must revalidate when        (account balance)
                    cache expires

proxy-revalidate    Like must-revalidate        For CDN/proxy only
                    but only for proxies        (private to browser)

s-maxage=N          Proxy cache duration        Different duration
                    (overrides max-age          for CDN vs browser
                    for proxies)


EXAMPLES:
─────────

Static assets (won't change):
    Cache-Control: public, max-age=31536000, immutable
    (1 year, anyone can cache, never changes)

Public data (might change):
    Cache-Control: public, max-age=3600
    (1 hour, anyone can cache, might become stale)

Personal data (logged-in user):
    Cache-Control: private, max-age=300
    (5 minutes, only browser caches, not CDN)

Sensitive data (never cache):
    Cache-Control: no-store
    (Don't cache this data)

Data that might change often:
    Cache-Control: public, max-age=0, must-revalidate
    (Always validate with server before use)
```

### ETag and Conditional Requests

```
PROBLEM:
────────
User requests /api/users a million times today.
Each request transfers the entire 100KB JSON response.
That's 100GB transferred!

Most of the time, the data hasn't changed.


SOLUTION: ETag (Entity Tag)
───────────────────────────

FIRST REQUEST:
Server: "Here's the user data"
Response: { users: [...] }
ETag: "abc123" (hash of response body)

Browser caches both response AND ETag

SECOND REQUEST (5 minutes later):
Browser: "I have your data from before (ETag: abc123)"
Request: GET /api/users
         If-None-Match: "abc123"

Server: "Let me check if the body has changed..."
        Hash current data...
        Current hash: "abc123" (same as before!)
        
Response: 304 Not Modified
        (No body, just headers)

Browser: "Okay, my cached copy is still fresh, use that"


BENEFITS:
─────────
• Request still made (but smaller)
• Response body not transferred (saves bandwidth)
• Can detect changes without transferring full body
• Works even if cache header says "might be stale"


IMPLEMENTATION:
───────────────
// Server generates ETag
function getUsers(req, res) {
    const users = fetchUsersFromDB();
    const data = JSON.stringify(users);
    const etag = crypto.createHash('md5').update(data).digest('hex');
    
    res.set('ETag', etag);
    
    // If client has same ETag, no need to send body
    if (req.headers['if-none-match'] === etag) {
        return res.status(304).end();  // No body
    }
    
    res.json(users);
}

// Or use Last-Modified (older approach)
res.set('Last-Modified', new Date(users.updatedAt).toUTCString());

if (new Date(req.headers['if-modified-since']) >= new Date(users.updatedAt)) {
    return res.status(304).end();
}

res.json(users);
```

---

## Part 7: CORS — Cross-Origin Requests

This is critical if your backend serves multiple frontends or external clients.

### CORS Problem

```
SCENARIO: You have a frontend and API on different domains

Frontend: https://app.example.com
API:      https://api.example.com

Browser security: "Cross-origin requests are dangerous!"

Frontend JavaScript tries:
    fetch('https://api.example.com/users')

Browser blocks it:
    ❌ Error: Cross-Origin Request Blocked
    (The request is from app.example.com, going to api.example.com)


WHY THE RESTRICTION?
────────────────────
Imagine attacker.com makes a request to your bank:
    fetch('https://bank.example.com/transfer', {
        method: 'POST',
        body: { to: 'attacker', amount: 1000000 }
    })

Browser has your bank login cookie.
Request includes it automatically.
Your money is transferred to attacker!

That's CSRF (Cross-Site Request Forgery).

Browser prevents this by blocking cross-origin requests.


THE SOLUTION: CORS
──────────────────
Server explicitly says: "I allow requests from these domains"

Response headers:
    Access-Control-Allow-Origin: https://app.example.com
    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
    Access-Control-Allow-Headers: Authorization, Content-Type
```

### CORS Mechanics

```
SIMPLE REQUEST (GET, POST, HEAD with simple headers):
─────────────────────────────────────────────────────

Browser request:
    GET /api/users HTTP/1.1
    Origin: https://app.example.com

Server response:
    200 OK
    Access-Control-Allow-Origin: https://app.example.com

Browser: "Origin is in allowed list, request succeeded"


COMPLEX REQUEST (PUT, DELETE, custom headers, custom content-type):
─────────────────────────────────────────────────────────────────

Browser first sends OPTIONS "preflight" request:
    OPTIONS /api/users HTTP/1.1
    Origin: https://app.example.com
    Access-Control-Request-Method: DELETE
    Access-Control-Request-Headers: Authorization

Server responds to OPTIONS:
    200 OK
    Access-Control-Allow-Origin: https://app.example.com
    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
    Access-Control-Allow-Headers: Authorization, Content-Type
    Access-Control-Max-Age: 86400  (cache for 24 hours)

Browser: "Server allows DELETE and Authorization header"

Now browser sends actual request:
    DELETE /api/users/123 HTTP/1.1
    Authorization: Bearer token...
    Origin: https://app.example.com

Server responds:
    204 No Content
    Access-Control-Allow-Origin: https://app.example.com

Browser: "Request succeeded"


KEY POINT:
──────────
Preflight happens automatically before complex requests.
Developers don't see it, but it adds latency.
Caching the preflight (Access-Control-Max-Age) helps.
```

### Implementing CORS Safely

```typescript
// ❌ DANGEROUS - Allows any origin
app.use((req, res, next) => {
    res.set('Access-Control-Allow-Origin', '*');
    next();
});

// This allows anyone to make requests from their site
// Better than nothing, but less secure


// ✓ CORRECT - Whitelist trusted origins
const allowedOrigins = [
    'https://app.example.com',
    'https://www.example.com',
    'https://admin.example.com'
];

app.use((req, res, next) => {
    const origin = req.headers.origin;
    
    if (allowedOrigins.includes(origin)) {
        res.set('Access-Control-Allow-Origin', origin);
        res.set('Access-Control-Allow-Credentials', 'true');
    }
    
    if (req.method === 'OPTIONS') {
        res.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
        res.set('Access-Control-Allow-Headers', 'Authorization, Content-Type');
        res.set('Access-Control-Max-Age', '86400');
        res.status(200).end();
    }
    
    next();
});


// ✓ OR use library (Express)
import cors from 'cors';

const corsOptions = {
    origin: function(origin, callback) {
        if (!origin || allowedOrigins.includes(origin)) {
            callback(null, true);
        } else {
            callback(new Error('Not allowed by CORS'));
        }
    },
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Authorization', 'Content-Type']
};

app.use(cors(corsOptions));
```

---

## Part 8: Security Headers — Production Requirements

These headers protect your users and your system.

```
HEADER                              PURPOSE
────────────────────────────────────────────────────────
Strict-Transport-Security           Force HTTPS everywhere
X-Content-Type-Options: nosniff     Prevent content-type sniffing
X-Frame-Options: DENY               Prevent clickjacking
Content-Security-Policy             Prevent XSS attacks
X-XSS-Protection                    Legacy XSS protection

Set-Cookie: ...; HttpOnly            Prevent XSS cookie theft
Set-Cookie: ...; Secure              Only HTTPS
Set-Cookie: ...; SameSite=Strict     Prevent CSRF


IMPLEMENTATION:
───────────────

// ✓ CORRECT - All security headers
app.use((req, res, next) => {
    // Force HTTPS for next 1 year
    res.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    
    // Don't let browser guess content type
    res.set('X-Content-Type-Options', 'nosniff');
    
    // Don't allow iframe embedding
    res.set('X-Frame-Options', 'DENY');
    
    // Prevent inline scripts (only from same origin)
    res.set('Content-Security-Policy', "default-src 'self'; script-src 'self' cdn.example.com");
    
    next();
});

// Or use helmet library
import helmet from 'helmet';
app.use(helmet());  // Adds all security headers automatically
```

---

## Part 9: Rate Limiting — Protecting Your System

Rate limiting prevents abuse and protects your infrastructure.

```
STRATEGY 1: Token Bucket
────────────────────────
Bucket capacity: 100 tokens (requests)
Refill rate: 10 tokens/minute

User makes 5 requests:
    Tokens left: 95

Wait 1 minute:
    Tokens refill: 10 added
    Tokens total: 105 (capped at 100)

Make 150 requests in burst:
    After 100: Bucket empty
    Requests 101-150: Rejected (429 Too Many Requests)

Wait 5 minutes:
    Tokens refilled: 50
    Back to normal


STRATEGY 2: Sliding Window
──────────────────────────
Limit: 100 requests per hour

Request 1-100: Accepted (within hour)
Request 101: Rejected (would exceed 100/hour)

After 1 hour 1 minute:
Request 101: Accepted (oldest request fell out of window)


IMPLEMENTATION:
───────────────

// Simple in-memory rate limiter
const requestCounts = new Map();

function isRateLimited(userId, limit = 100, windowMs = 60000) {
    const key = `user:${userId}`;
    const now = Date.now();
    
    if (!requestCounts.has(key)) {
        requestCounts.set(key, []);
    }
    
    // Remove requests older than window
    let requests = requestCounts.get(key)
        .filter(t => now - t < windowMs);
    
    if (requests.length >= limit) {
        return true;  // Rate limited
    }
    
    // Add current request
    requests.push(now);
    requestCounts.set(key, requests);
    return false;  // Not rate limited
}

// Middleware
app.use((req, res, next) => {
    const userId = req.user?.id || req.ip;
    
    if (isRateLimited(userId, 100, 60000)) {  // 100 per minute
        return res.status(429)
            .set('Retry-After', '60')
            .json({ error: 'Rate limited' });
    }
    
    next();
});


RESPONSE WITH RATE LIMIT INFO:
────────────────────────────
Response: 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1615383600

Client can check remaining requests before hitting limit.
```

---

## Summary: Production Checklist

When building production APIs, ensure:

```
✓ Use correct HTTP methods (GET, POST, PUT, DELETE)
✓ Return correct status codes (200, 201, 400, 401, 403, 404, 429, 500)
✓ Include required headers (Content-Type, Cache-Control, Authorization)
✓ Implement authentication (Cookies or Tokens)
✓ Use ETag/Last-Modified for caching
✓ Implement CORS correctly (whitelist origins)
✓ Add all security headers (HSTS, CSP, X-Frame-Options, etc.)
✓ Implement rate limiting (protect against abuse)
✓ Use HTTPS everywhere (Strict-Transport-Security)
✓ Log requests/responses (X-Request-ID for tracing)
✓ Handle idempotency (Idempotency-Key for POST)
✓ Provide Retry-After headers (for 429, 503)
✓ Document all endpoints (what methods, what auth, what status codes)
```

These aren't optional. These are what separates hobby projects from production systems.
