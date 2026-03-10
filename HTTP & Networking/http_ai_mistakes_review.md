# HTTP & Networking: AI Mistakes & Code Review
## What AI Gets Wrong and How to Catch It

---

## Mistake 1: Wrong Status Codes

### What AI Often Does

```typescript
// ❌ WRONG
app.post('/api/users', (req, res) => {
    const user = User.create(req.body);
    
    // AI returns 200 for everything
    res.status(200).json(user);  // ← Should be 201
});

app.get('/api/users/:id', (req, res) => {
    const user = User.findById(req.params.id);
    
    // AI doesn't check if user exists
    res.status(200).json(user);  // ← Returns 200 with null, should be 404
});

app.delete('/api/users/:id', (req, res) => {
    User.deleteById(req.params.id);
    
    // AI returns full data, should be 204
    res.status(200).json(user);  // ← Should be 204 No Content
});
```

### Why It's Wrong

- 200 means "success, here's data"
- 201 means "created, here's the new resource"
- 404 means "resource doesn't exist"
- 204 means "success, no data to return"

Clients rely on status codes to know if operation succeeded!

### How to Catch It

```
RED FLAGS when reviewing:
─────────────────────────
❌ POST endpoint returns 200 (should be 201)
❌ GET returns 200 for non-existent resource (should be 404)
❌ DELETE returns 200 with body (should be 204)
❌ Update returns 201 (should be 200 or 204)
❌ No handling for "not found" case
```

### What You Should Write

```typescript
// ✓ CORRECT
app.post('/api/users', (req, res) => {
    const user = User.create(req.body);
    
    res.status(201)  // ← Created
        .set('Location', `/api/users/${user.id}`)
        .json(user);
});

app.get('/api/users/:id', (req, res) => {
    const user = User.findById(req.params.id);
    
    if (!user) {
        return res.status(404).json({  // ← Not Found
            error: { code: 'NOT_FOUND', message: 'User not found' }
        });
    }
    
    res.status(200).json(user);
});

app.delete('/api/users/:id', (req, res) => {
    User.deleteById(req.params.id);
    
    res.status(204).end();  // ← No Content, no body
});
```

---

## Mistake 2: Missing or Wrong Content-Type Headers

### What AI Often Does

```typescript
// ❌ WRONG
app.get('/api/users', (req, res) => {
    const users = User.getAll();
    
    // AI sends JSON without declaring it
    res.json(users);  // ← Express sets Content-Type automatically
    
    // But what if using raw Node?
    res.write(JSON.stringify(users));
    res.end();  // ← No Content-Type header!
});

// ❌ WRONG - Returning wrong content type
app.get('/api/file', (req, res) => {
    const buffer = fs.readFileSync('file.pdf');
    
    // AI forgets to set Content-Type
    res.send(buffer);
    
    // Browser doesn't know it's PDF, might try to display as text
});
```

### Why It's Wrong

- Browser doesn't know how to parse the response
- Might display binary data as text
- File might not download/open correctly
- JSON responses might be treated as HTML

### How to Catch It

```
RED FLAGS:
──────────
❌ res.json() without explicit Content-Type
   (Express does this, but be explicit)
❌ res.send() without Content-Type
❌ Custom headers without Content-Type
❌ Returning file without application/pdf or application/octet-stream
❌ Returning CSV without text/csv
```

### What You Should Write

```typescript
// ✓ CORRECT
app.get('/api/users', (req, res) => {
    const users = User.getAll();
    
    res.set('Content-Type', 'application/json');
    res.json(users);
});

app.get('/api/file.pdf', (req, res) => {
    const buffer = fs.readFileSync('file.pdf');
    
    res.set('Content-Type', 'application/pdf');
    res.set('Content-Length', buffer.length.toString());
    res.end(buffer);
});

app.get('/api/export.csv', (req, res) => {
    const csv = generateCSV(data);
    
    res.set('Content-Type', 'text/csv');
    res.set('Content-Disposition', 'attachment; filename="export.csv"');
    res.send(csv);
});
```

---

## Mistake 3: Missing Authentication/Authorization Checks

### What AI Often Does

```typescript
// ❌ DANGEROUS
app.get('/api/admin/users', (req, res) => {
    // AI doesn't check authentication!
    const users = User.getAll();
    res.json(users);
});

// ❌ DANGEROUS - Auth check but no proper error
app.delete('/api/users/:id', (req, res) => {
    if (!req.headers.authorization) {
        res.json({ error: 'Not authorized' });  // ← 200 OK with error message
        return;
    }
    
    User.deleteById(req.params.id);
    res.json({ success: true });
});

// ❌ WRONG - Returns 403 for missing auth (should be 401)
app.get('/api/private', (req, res) => {
    if (!req.headers.authorization) {
        return res.status(403).json({  // ← Should be 401
            error: 'Unauthorized'
        });
    }
    
    res.json({ data: 'secret' });
});

// ❌ DANGEROUS - No permission check
app.post('/api/admin/create-user', auth, (req, res) => {
    // Only checks authentication, not admin permission!
    User.create(req.body);
    res.status(201).json(user);
});
```

### Why It's Wrong

- Exposes private/admin data to unauthenticated users
- Returns success (200) when should return error (401)
- Doesn't distinguish between "not authenticated" (401) and "not authorized" (403)
- Doesn't verify user has permission for the action

### How to Catch It

```
RED FLAGS:
──────────
❌ No auth check on private endpoints
❌ No error response for missing auth
❌ 200 OK status with error message in body
❌ 403 used for missing auth (should be 401)
❌ Auth check but no permission check
❌ Admin endpoints have no admin verification
❌ Deletes/updates user data without auth
```

### What You Should Write

```typescript
// ✓ CORRECT - Auth middleware
function auth(req: Request, res: Response, next: NextFunction) {
    const token = req.headers.authorization?.slice(7);  // Remove "Bearer "
    
    if (!token) {
        return res.status(401).json({  // ← 401 Not Authenticated
            error: { code: 'MISSING_AUTH', message: 'Authorization required' }
        });
    }
    
    try {
        const payload = jwt.verify(token, SECRET);
        req.user = payload;
        next();
    } catch {
        return res.status(401).json({
            error: { code: 'INVALID_TOKEN', message: 'Invalid token' }
        });
    }
}

// ✓ CORRECT - Authorization middleware
function requireRole(...roles: string[]) {
    return (req: Request, res: Response, next: NextFunction) => {
        if (!roles.includes(req.user?.role)) {
            return res.status(403).json({  // ← 403 Not Authorized
                error: { 
                    code: 'INSUFFICIENT_PERMISSIONS', 
                    message: `Requires one of: ${roles.join(', ')}` 
                }
            });
        }
        next();
    };
}

// ✓ CORRECT - Protected endpoint
app.delete(
    '/api/users/:id',
    auth,  // Check authenticated
    requireRole('admin'),  // Check authorized
    (req: Request, res: Response) => {
        User.deleteById(req.params.id);
        res.status(204).end();
    }
);
```

---

## Mistake 4: Wrong HTTP Method Usage

### What AI Often Does

```typescript
// ❌ WRONG - Using POST for retrieval
app.post('/api/users/search', (req, res) => {
    const users = User.search(req.body.query);
    res.json(users);
});

// Problem: Can't be cached, can't be bookmarked, non-standard


// ❌ WRONG - Using GET to modify data
app.get('/api/users/:id/delete', (req, res) => {
    User.deleteById(req.params.id);
    res.json({ deleted: true });
});

// Problem: Browser might prefetch it! Accidentally deletes data!


// ❌ WRONG - Using POST for partial update
app.post('/api/users/:id/update-name', (req, res) => {
    User.updateName(req.params.id, req.body.name);
    res.json(user);
});

// Problem: Multiple endpoints for same resource, non-standard


// ❌ WRONG - Using PUT but treating like PATCH
app.put('/api/users/:id', (req, res) => {
    // Only updates fields in body, doesn't replace entire resource
    User.updatePartial(req.params.id, req.body);
    res.json(user);
});

// Problem: PUT should replace entire resource, this is PATCH behavior
```

### Why It's Wrong

- HTTP methods have semantic meaning
- GET shouldn't modify data (browser can prefetch it)
- POST should be idempotent-aware (has duplicate problem)
- PUT is for full replacement, PATCH for partial
- Non-standard endpoints confuse clients and APIs

### How to Catch It

```
RED FLAGS:
──────────
❌ POST /api/users/get (should be GET)
❌ GET /api/users/delete (should be DELETE)
❌ POST /api/users/update (should be PUT or PATCH)
❌ /api/search-users (should be GET /api/users?q=...)
❌ Multiple endpoints for same resource
```

### What You Should Write

```typescript
// ✓ CORRECT
// Create
app.post('/api/users', (req, res) => {
    const user = User.create(req.body);
    res.status(201).json(user);
});

// Read
app.get('/api/users/:id', (req, res) => {
    const user = User.findById(req.params.id);
    res.json(user);
});

// Search (GET with query)
app.get('/api/users', (req, res) => {
    const users = User.search(req.query.q);
    res.json(users);
});

// Full replace (PUT)
app.put('/api/users/:id', (req, res) => {
    const user = User.replace(req.params.id, req.body);
    res.json(user);
});

// Partial update (PATCH)
app.patch('/api/users/:id', (req, res) => {
    const user = User.updatePartial(req.params.id, req.body);
    res.json(user);
});

// Delete
app.delete('/api/users/:id', (req, res) => {
    User.deleteById(req.params.id);
    res.status(204).end();
});
```

---

## Mistake 5: Forgetting Security Headers

### What AI Often Does

```typescript
// ❌ WRONG - No security headers
app.get('/api/data', (req, res) => {
    const data = getData();
    res.json(data);
    
    // Missing security headers!
    // No HTTPS enforcement
    // No XSS protection
    // No CSRF protection
});

// ❌ WRONG - Returns data over HTTP (should enforce HTTPS)
// No Strict-Transport-Security header
```

### Why It's Wrong

- Exposes users to man-in-the-middle attacks
- Vulnerable to XSS (Cross-Site Scripting)
- Vulnerable to CSRF (Cross-Site Request Forgery)
- Vulnerable to clickjacking
- No protection against MIME type sniffing

### How to Catch It

```
RED FLAGS:
──────────
❌ No Strict-Transport-Security header
❌ No X-Content-Type-Options header
❌ No X-Frame-Options header
❌ No Content-Security-Policy header
❌ Endpoints accessible over HTTP (should be HTTPS only)
```

### What You Should Write

```typescript
// ✓ CORRECT - Add security headers
app.use((req, res, next) => {
    // Force HTTPS
    res.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    
    // Prevent MIME sniffing
    res.set('X-Content-Type-Options', 'nosniff');
    
    // Prevent clickjacking
    res.set('X-Frame-Options', 'DENY');
    
    // Prevent XSS
    res.set('Content-Security-Policy', "default-src 'self'");
    
    next();
});

// OR use helmet library (recommended)
import helmet from 'helmet';
app.use(helmet());
```

---

## Mistake 6: No Caching Headers

### What AI Often Does

```typescript
// ❌ WRONG - No cache headers
app.get('/api/users', (req, res) => {
    const users = User.getAll();
    res.json(users);
    
    // No Cache-Control header!
    // Browser caches anyway (browser defaults)
    // CDN can't cache
    // Client doesn't know how long to cache
});

// ❌ WRONG - All endpoints cached equally
app.use((req, res, next) => {
    res.set('Cache-Control', 'max-age=3600');  // Everything cached 1 hour
    next();
});

// Problem: Personal data cached like public data!
```

### Why It's Wrong

- Personal data shouldn't be cached broadly (private, not public)
- No ETag means full response redownloaded even if unchanged
- Makes system slower (no caching)
- Makes system less scalable (more load)

### How to Catch It

```
RED FLAGS:
──────────
❌ No Cache-Control header
❌ Same cache duration for all endpoints
❌ Personal data marked as "public"
❌ No ETag on cached responses
❌ max-age=0 (disables all caching)
```

### What You Should Write

```typescript
// ✓ CORRECT

// Public data - cache broadly
app.get('/api/departments', (req, res) => {
    const depts = Department.getAll();
    
    res.set('Cache-Control', 'public, max-age=86400');
    res.json(depts);
});

// Personal data - cache only in browser
app.get('/api/my-profile', auth, (req, res) => {
    const user = User.findById(req.user.id);
    
    res.set('Cache-Control', 'private, max-age=300');
    res.json(user);
});

// Sensitive data - don't cache
app.get('/api/account', auth, (req, res) => {
    const account = Account.findById(req.user.id);
    
    res.set('Cache-Control', 'no-store');
    res.json(account);
});

// With ETag for efficiency
app.get('/api/courses', (req, res) => {
    const courses = Course.getAll();
    const etag = crypto.createHash('md5')
        .update(JSON.stringify(courses))
        .digest('hex');
    
    if (req.headers['if-none-match'] === etag) {
        return res.status(304).end();
    }
    
    res.set('Cache-Control', 'public, max-age=3600, must-revalidate');
    res.set('ETag', etag);
    res.json(courses);
});
```

---

## Mistake 7: Poor Error Responses

### What AI Often Does

```typescript
// ❌ WRONG - No error code
app.get('/api/users/:id', (req, res) => {
    try {
        const user = User.findById(req.params.id);
        res.json(user);
    } catch (err) {
        res.status(500).json({ error: err.message });
        // Client has no error code to handle programmatically
    }
});

// ❌ WRONG - Inconsistent error format
app.post('/api/users', (req, res) => {
    if (!req.body.email) {
        res.status(400).json({ error: 'Email required' });
    } else if (!req.body.name) {
        res.status(400).json({ message: 'Name required' });  // Different key!
    }
});

// ❌ WRONG - Doesn't distinguish error types
app.get('/api/users/:id', (req, res) => {
    try {
        const user = User.findById(req.params.id);
        res.json(user);
    } catch (err) {
        // Could be "not found" or "database error"
        // Client doesn't know
        res.status(500).json({ error: 'Failed to get user' });
    }
});
```

### Why It's Wrong

- Client can't handle errors programmatically
- Inconsistent error structure
- Can't distinguish between different types of failures
- No way for support to track errors (no request ID)

### How to Catch It

```
RED FLAGS:
──────────
❌ Error response has no error code
❌ Error structure varies between endpoints
❌ No distinction between 404 (not found) and 500 (server error)
❌ Error message doesn't help client know what to do
❌ No request ID in error responses
```

### What You Should Write

```typescript
// ✓ CORRECT

interface ErrorResponse {
    error: {
        code: string;        // Programmatic error code
        message: string;     // Human-readable message
        details?: any;       // Additional details (validation errors, etc)
    };
    requestId: string;       // For support
}

// Consistent error structure
app.get('/api/users/:id', (req, res) => {
    try {
        const user = User.findById(req.params.id);
        
        if (!user) {
            return res.status(404).json({
                error: {
                    code: 'USER_NOT_FOUND',
                    message: 'User not found'
                },
                requestId: req.requestId
            });
        }
        
        res.json(user);
        
    } catch (err) {
        res.status(500).json({
            error: {
                code: 'INTERNAL_ERROR',
                message: 'Failed to retrieve user'
            },
            requestId: req.requestId
        });
    }
});

// Validation errors with details
app.post('/api/users', (req, res) => {
    const errors: Record<string, string> = {};
    
    if (!req.body.email) {
        errors.email = 'Email is required';
    }
    if (!req.body.name) {
        errors.name = 'Name is required';
    }
    
    if (Object.keys(errors).length > 0) {
        return res.status(400).json({
            error: {
                code: 'VALIDATION_ERROR',
                message: 'Validation failed',
                details: { fields: errors }
            },
            requestId: req.requestId
        });
    }
    
    const user = User.create(req.body);
    res.status(201).json(user);
});
```

---

## Mistake 8: Not Handling Idempotency

### What AI Often Does

```typescript
// ❌ DANGEROUS - Not idempotent
app.post('/api/orders', (req, res) => {
    const order = Order.create(req.body);
    res.status(201).json(order);
    
    // No idempotency key!
    // Network fails, client retries
    // Creates duplicate order!
});

// ❌ WRONG - Doesn't require idempotency key
app.post('/api/payments', (req, res) => {
    const payment = Payment.process(req.body);
    res.status(200).json(payment);
    
    // Should require idempotency key for payments!
});
```

### Why It's Wrong

- POST requests can be retried on network failure
- Without idempotency keys, duplicates are created
- Critical for payments, orders, financial transactions
- Causes customer complaints (charged twice!)

### How to Catch It

```
RED FLAGS:
──────────
❌ POST endpoint without idempotency key support
❌ Financial operations (payments, transfers) without idempotency
❌ No Idempotency-Key header requirement
❌ Endpoint description doesn't mention idempotency
```

### What You Should Write

```typescript
// ✓ CORRECT

const processedRequests = new Map();

app.post('/api/orders', (req, res) => {
    const idempotencyKey = req.headers['idempotency-key'];
    
    if (!idempotencyKey) {
        return res.status(400).json({
            error: {
                code: 'MISSING_IDEMPOTENCY_KEY',
                message: 'Idempotency-Key header required'
            }
        });
    }
    
    // Check if already processed
    if (processedRequests.has(idempotencyKey)) {
        const result = processedRequests.get(idempotencyKey);
        return res.status(201).json(result);  // Return same response
    }
    
    // Process
    const order = Order.create(req.body);
    
    // Cache result
    processedRequests.set(idempotencyKey, order);
    
    res.status(201).json(order);
});
```

---

## Code Review Checklist

Use this when reviewing API code AI writes:

```
STATUS CODES
─────────────
☐ POST returns 201 (not 200)
☐ GET non-existent returns 404 (not 200)
☐ DELETE returns 204 (not 200)
☐ Missing auth returns 401 (not 403)
☐ Insufficient permission returns 403
☐ Rate limited returns 429

HEADERS
───────
☐ Content-Type set correctly
☐ Cache-Control set appropriately
☐ ETag on cacheable responses
☐ Security headers present (HSTS, X-Frame-Options, etc)
☐ CORS headers if cross-origin
☐ Location header on 201 Created

AUTHENTICATION & AUTHORIZATION
──────────────────────────────
☐ Auth check on private endpoints
☐ Authorization check on restricted endpoints
☐ Distinguishes 401 (auth) from 403 (permissions)
☐ Tokens verified before use
☐ No sensitive data in logs/errors

CACHING
────────
☐ Public data marked as "public"
☐ Personal data marked as "private"
☐ Sensitive data has "no-store"
☐ Appropriate max-age values
☐ ETag for validation

ERRORS
───────
☐ Consistent error structure
☐ Error responses have code + message
☐ Meaningful error messages
☐ Request ID included
☐ Different codes for different errors (NOT_FOUND vs VALIDATION_ERROR)

IDEMPOTENCY
───────────
☐ POST/PATCH/DELETE require Idempotency-Key if critical
☐ Financial operations have idempotency support
☐ Duplicate requests return same response

SECURITY
─────────
☐ No sensitive data in URLs
☐ HTTPS enforced (Strict-Transport-Security)
☐ CORS whitelist (not *)
☐ Rate limiting on auth endpoints
☐ Validation on all inputs
```

When you use this checklist, you'll catch what AI consistently misses.
