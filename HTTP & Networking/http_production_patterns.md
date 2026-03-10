# HTTP Patterns for Production APIs
## Building Secure, Fast, and Reliable Endpoints

---

## Part 1: API Design Patterns with HTTP

### Pattern 1: Resource-Oriented REST

REST = Representational State Transfer. Design your API around **resources**, not actions.

```typescript
// ❌ BAD - Action-oriented (Not REST)
GET /api/getUserById?id=123
POST /api/createUser
POST /api/deleteUser?id=123
POST /api/updateUserName?id=123&name=John

// Problem:
// - Multiple URLs for same resource
// - Doesn't use HTTP methods semantically
// - Hard to cache
// - Non-standard


// ✓ GOOD - Resource-oriented (REST)
GET /api/users/123              // Get user resource
POST /api/users                 // Create new user resource
PUT /api/users/123              // Replace user resource
PATCH /api/users/123            // Partially update user
DELETE /api/users/123           // Delete user resource
GET /api/users                  // List all user resources
GET /api/users?page=2&limit=20  // List with pagination


BENEFITS:
─────────
• Consistent URL structure
• HTTP methods are semantic
• Easy to cache
• Standard across industry
```

### Pattern 2: Proper Request/Response Structure

```typescript
// REQUEST STRUCTURE
─────────────────

// Create user
POST /api/users
Content-Type: application/json
Authorization: Bearer token...

{
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "password": "secure_password"  // Only in POST, never in response
}


// Partial update
PATCH /api/users/123
{
    "firstName": "Jane"  // Only changed fields
}


// RESPONSE STRUCTURE
───────────────────

// Success with resource (201 Created)
Status: 201 Created
Content-Type: application/json
Location: /api/users/123

{
    "id": "123",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "createdAt": "2025-03-10T09:00:00Z",
    "updatedAt": "2025-03-10T09:00:00Z"
}


// Success without body (204 No Content)
Status: 204 No Content


// Error response
Status: 400 Bad Request
Content-Type: application/json

{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid email format",
        "fields": {
            "email": "Must be valid email address"
        }
    }
}


IMPLEMENTATION:
───────────────

interface ApiResponse<T> {
    success: boolean;
    data?: T;
    error?: {
        code: string;
        message: string;
        details?: any;
    };
}

// Endpoint handler
async function createUser(req: Request, res: Response) {
    try {
        // Validate
        const { error, value } = validateUserInput(req.body);
        if (error) {
            return res.status(400).json({
                error: {
                    code: 'VALIDATION_ERROR',
                    message: error.message,
                    fields: error.details
                }
            });
        }
        
        // Create
        const user = await User.create(value);
        
        // Response
        res.status(201)
            .set('Location', `/api/users/${user.id}`)
            .json({
                success: true,
                data: user
            });
            
    } catch (err) {
        res.status(500).json({
            error: {
                code: 'INTERNAL_ERROR',
                message: 'Failed to create user'
            }
        });
    }
}
```

### Pattern 3: Proper Error Handling

```typescript
// ERROR RESPONSE STRUCTURE
──────────────────────────

// ✓ CORRECT
Status: 400 Bad Request
{
    "error": {
        "code": "INVALID_INPUT",
        "message": "Email is required",
        "timestamp": "2025-03-10T09:00:00Z",
        "requestId": "req-123-abc"
    }
}

// Client can:
// - Show user message: error.message
// - Log issue: error.requestId
// - Handle programmatically: error.code
// - Retry decision: Check if error.retryable


// DIFFERENT ERROR CODES
──────────────────────

// 400: Bad Request - Client error
{
    "error": {
        "code": "INVALID_EMAIL",
        "message": "Email format is invalid",
        "field": "email"
    }
}

// 401: Unauthorized - Missing auth
{
    "error": {
        "code": "MISSING_AUTH",
        "message": "Authorization header required"
    }
}

// 403: Forbidden - Insufficient permissions
{
    "error": {
        "code": "INSUFFICIENT_PERMISSIONS",
        "message": "You don't have permission to delete this user"
    }
}

// 404: Not Found
{
    "error": {
        "code": "NOT_FOUND",
        "message": "User 999 not found"
    }
}

// 409: Conflict - Duplicate or version mismatch
{
    "error": {
        "code": "EMAIL_ALREADY_EXISTS",
        "message": "User with this email already exists"
    }
}

// 429: Rate Limited
{
    "error": {
        "code": "RATE_LIMITED",
        "message": "Too many requests",
        "retryAfter": 60
    }
}

// 500: Internal Server Error
{
    "error": {
        "code": "INTERNAL_ERROR",
        "message": "Internal server error",
        "requestId": "req-123-abc"
    }
}


// CLIENT-SIDE HANDLING
──────────────────────

async function fetchUsers() {
    try {
        const response = await fetch('/api/users');
        const data = await response.json();
        
        if (!response.ok) {
            // Server returned error
            const error = data.error;
            
            switch (error.code) {
                case 'RATE_LIMITED':
                    // Show: "Too many requests, try again later"
                    // Schedule retry after delay
                    break;
                    
                case 'MISSING_AUTH':
                    // Redirect to login
                    window.location.href = '/login';
                    break;
                    
                case 'INSUFFICIENT_PERMISSIONS':
                    // Show: "You don't have permission"
                    break;
                    
                case 'INTERNAL_ERROR':
                    // Log: error.requestId for support
                    // Show: "Something went wrong, try again"
                    break;
                    
                default:
                    // Generic error handling
                    break;
            }
            
            throw error;
        }
        
        return data.data;  // Success case
        
    } catch (err) {
        console.error('Failed to fetch users:', err);
        throw err;
    }
}
```

---

## Part 2: Authentication & Authorization Patterns

### Pattern 1: JWT Token Implementation

```typescript
// ============ TOKEN GENERATION ============

interface TokenPayload {
    userId: string;
    email: string;
    role: 'user' | 'admin';
    permissions: string[];
}

interface TokenResponse {
    accessToken: string;
    refreshToken: string;
    expiresIn: number;  // seconds
}

function generateTokens(user: User): TokenResponse {
    // Access token: short-lived (15 minutes)
    const accessToken = jwt.sign(
        {
            userId: user.id,
            email: user.email,
            role: user.role,
            permissions: user.permissions
        } as TokenPayload,
        process.env.JWT_SECRET!,
        { expiresIn: '15m' }
    );
    
    // Refresh token: long-lived (7 days)
    const refreshToken = jwt.sign(
        { userId: user.id },
        process.env.JWT_REFRESH_SECRET!,
        { expiresIn: '7d' }
    );
    
    // Store refresh token for revocation
    saveRefreshToken(user.id, refreshToken);
    
    return {
        accessToken,
        refreshToken,
        expiresIn: 15 * 60  // 15 minutes in seconds
    };
}


// ============ LOGIN ENDPOINT ============

app.post('/auth/login', async (req: Request, res: Response) => {
    try {
        const { email, password } = req.body;
        
        // Validate credentials
        const user = await User.findByEmail(email);
        if (!user || !await user.verifyPassword(password)) {
            return res.status(401).json({
                error: {
                    code: 'INVALID_CREDENTIALS',
                    message: 'Email or password is incorrect'
                }
            });
        }
        
        // Generate tokens
        const tokens = generateTokens(user);
        
        // Response
        res.status(200).json({
            success: true,
            data: {
                accessToken: tokens.accessToken,
                refreshToken: tokens.refreshToken,
                expiresIn: tokens.expiresIn,
                user: {
                    id: user.id,
                    email: user.email,
                    role: user.role
                }
            }
        });
        
    } catch (err) {
        res.status(500).json({
            error: { code: 'INTERNAL_ERROR', message: 'Login failed' }
        });
    }
});


// ============ REFRESH TOKEN ENDPOINT ============

app.post('/auth/refresh', (req: Request, res: Response) => {
    try {
        const { refreshToken } = req.body;
        
        if (!refreshToken) {
            return res.status(401).json({
                error: { code: 'MISSING_REFRESH_TOKEN', message: 'Refresh token required' }
            });
        }
        
        // Verify refresh token
        const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET!) as { userId: string };
        
        // Check if token is blacklisted (revoked)
        if (isTokenBlacklisted(refreshToken)) {
            return res.status(401).json({
                error: { code: 'TOKEN_REVOKED', message: 'Refresh token has been revoked' }
            });
        }
        
        // Get user and generate new access token
        const user = await User.findById(payload.userId);
        if (!user) {
            return res.status(404).json({
                error: { code: 'USER_NOT_FOUND', message: 'User not found' }
            });
        }
        
        const newAccessToken = jwt.sign(
            {
                userId: user.id,
                email: user.email,
                role: user.role,
                permissions: user.permissions
            },
            process.env.JWT_SECRET!,
            { expiresIn: '15m' }
        );
        
        res.json({
            success: true,
            data: {
                accessToken: newAccessToken,
                expiresIn: 15 * 60
            }
        });
        
    } catch (err) {
        if (err instanceof jwt.TokenExpiredError) {
            return res.status(401).json({
                error: { code: 'REFRESH_TOKEN_EXPIRED', message: 'Refresh token expired' }
            });
        }
        
        res.status(401).json({
            error: { code: 'INVALID_REFRESH_TOKEN', message: 'Invalid refresh token' }
        });
    }
});


// ============ LOGOUT ENDPOINT ============

app.post('/auth/logout', auth, (req: Request, res: Response) => {
    try {
        const { refreshToken } = req.body;
        
        // Blacklist refresh token (invalidate it)
        blacklistToken(refreshToken);
        
        res.status(200).json({
            success: true,
            message: 'Logged out successfully'
        });
        
    } catch (err) {
        res.status(500).json({
            error: { code: 'INTERNAL_ERROR', message: 'Logout failed' }
        });
    }
});


// ============ PROTECTED ENDPOINT MIDDLEWARE ============

function authMiddleware(req: Request, res: Response, next: NextFunction) {
    try {
        const authHeader = req.headers.authorization;
        
        if (!authHeader || !authHeader.startsWith('Bearer ')) {
            return res.status(401).json({
                error: { code: 'MISSING_AUTH', message: 'Authorization header required' }
            });
        }
        
        const token = authHeader.slice(7);  // Remove "Bearer "
        
        // Verify token
        const payload = jwt.verify(token, process.env.JWT_SECRET!) as TokenPayload;
        
        // Attach to request
        req.user = payload;
        next();
        
    } catch (err) {
        if (err instanceof jwt.TokenExpiredError) {
            return res.status(401).json({
                error: { code: 'TOKEN_EXPIRED', message: 'Access token expired, refresh it' }
            });
        }
        
        res.status(401).json({
            error: { code: 'INVALID_TOKEN', message: 'Invalid access token' }
        });
    }
}

// Usage
app.get('/api/private', authMiddleware, (req: Request, res: Response) => {
    // req.user is now available
    res.json({
        success: true,
        data: { message: `Hello ${req.user.email}` }
    });
});
```

### Pattern 2: Permission-Based Authorization

```typescript
// ============ PERMISSION CHECKING ============

type Permission = 
    | 'users.read'
    | 'users.create'
    | 'users.update'
    | 'users.delete'
    | 'courses.manage'
    | 'materials.upload';

interface User {
    id: string;
    role: 'student' | 'instructor' | 'admin';
    permissions: Permission[];
}

// Define what permissions each role has
const rolePermissions: Record<string, Permission[]> = {
    student: [
        'users.read',
        'courses.read',
        'materials.read'
    ],
    instructor: [
        'users.read',
        'courses.read',
        'courses.manage',
        'materials.read',
        'materials.upload'
    ],
    admin: [
        'users.read',
        'users.create',
        'users.update',
        'users.delete',
        'courses.manage',
        'materials.upload'
    ]
};

// Middleware to check permissions
function requirePermission(...permissions: Permission[]) {
    return (req: Request, res: Response, next: NextFunction) => {
        const user = req.user;
        
        const hasPermission = permissions.some(p => user.permissions.includes(p));
        
        if (!hasPermission) {
            return res.status(403).json({
                error: {
                    code: 'INSUFFICIENT_PERMISSIONS',
                    message: `Requires one of: ${permissions.join(', ')}`
                }
            });
        }
        
        next();
    };
}

// Usage
app.delete('/api/users/:id', 
    authMiddleware,
    requirePermission('users.delete'),
    (req: Request, res: Response) => {
        // Only authenticated users with 'users.delete' permission
        User.deleteById(req.params.id);
        res.status(204).end();
    }
);
```

---

## Part 3: Caching Strategies

### Pattern 1: HTTP Caching for APIs

```typescript
// ============ STATIC DATA - Cache forever ============

app.get('/api/departments', (req: Request, res: Response) => {
    const departments = Departments.getAll();
    
    res.set('Cache-Control', 'public, max-age=86400, immutable');
    // Cache for 1 day, anyone can cache, never changes
    
    res.json({
        success: true,
        data: departments
    });
});


// ============ FREQUENTLY ACCESSED BUT CHANGEABLE DATA ============

app.get('/api/courses', (req: Request, res: Response) => {
    const courses = Course.getAll();
    
    // Generate ETag based on data
    const etag = crypto
        .createHash('md5')
        .update(JSON.stringify(courses))
        .digest('hex');
    
    // Check If-None-Match header
    if (req.headers['if-none-match'] === etag) {
        return res.status(304).end();  // Not modified
    }
    
    res.set('Cache-Control', 'public, max-age=300, must-revalidate');
    // Cache for 5 minutes, always revalidate after
    res.set('ETag', etag);
    
    res.json({
        success: true,
        data: courses
    });
});


// ============ USER-SPECIFIC DATA - Don't cache broadly ============

app.get('/api/my-courses', authMiddleware, (req: Request, res: Response) => {
    const courses = Course.findByUserId(req.user.userId);
    
    res.set('Cache-Control', 'private, max-age=60, must-revalidate');
    // Cache only in browser, not CDN
    // Revalidate frequently (1 minute)
    
    res.json({
        success: true,
        data: courses
    });
});


// ============ SENSITIVE DATA - Don't cache ============

app.get('/api/account', authMiddleware, (req: Request, res: Response) => {
    const account = User.findById(req.user.userId);
    
    res.set('Cache-Control', 'no-store');
    // Don't cache at all
    
    res.json({
        success: true,
        data: account
    });
});
```

---

## Part 4: Idempotency Pattern

```typescript
// ============ IDEMPOTENCY KEY ============

// Problem: POST requests are not idempotent
// Solution: Use idempotency key to track duplicate requests

interface IdempotencyRequest {
    idempotencyKey: string;
    // ... other fields
}

// Store completed requests
const completedRequests = new Map<string, any>();

// Middleware to handle idempotency
function idempotencyMiddleware(req: Request, res: Response, next: NextFunction) {
    const idempotencyKey = req.headers['idempotency-key'];
    
    // Only for POST, PATCH, DELETE (not GET)
    if (!['POST', 'PATCH', 'DELETE'].includes(req.method)) {
        return next();
    }
    
    if (!idempotencyKey) {
        return res.status(400).json({
            error: {
                code: 'MISSING_IDEMPOTENCY_KEY',
                message: 'Idempotency-Key header required for this operation'
            }
        });
    }
    
    // Check if this request was already processed
    const cached = completedRequests.get(idempotencyKey as string);
    if (cached) {
        // Return same response as before
        return res.status(cached.status).json(cached.body);
    }
    
    // Store original res.json
    const originalJson = res.json.bind(res);
    
    // Intercept response
    res.json = function(body: any) {
        // Cache the response
        completedRequests.set(idempotencyKey as string, {
            status: res.statusCode,
            body
        });
        
        // Clean up old entries (keep last 1000)
        if (completedRequests.size > 1000) {
            const firstKey = completedRequests.keys().next().value;
            completedRequests.delete(firstKey);
        }
        
        return originalJson(body);
    };
    
    next();
}

app.use(idempotencyMiddleware);


// ============ CLIENT USAGE ============

async function createPayment(amount: number) {
    const idempotencyKey = generateUUID();  // Unique per request
    
    const response = await fetch('/api/payments', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Idempotency-Key': idempotencyKey  // ← Send this
        },
        body: JSON.stringify({ amount })
    });
    
    const data = await response.json();
    
    // If network fails, retry with SAME idempotencyKey
    // Server returns same response as first attempt
    // No duplicate payment!
}
```

---

## Part 5: Rate Limiting Pattern

```typescript
// ============ RATE LIMITING MIDDLEWARE ============

import Redis from 'ioredis';

const redis = new Redis();

interface RateLimitOptions {
    windowMs: number;      // Time window in milliseconds
    maxRequests: number;   // Max requests per window
}

function rateLimitMiddleware(options: RateLimitOptions) {
    return async (req: Request, res: Response, next: NextFunction) => {
        try {
            // Identify user/IP
            const identifier = req.user?.userId || req.ip;
            const key = `ratelimit:${identifier}`;
            
            // Get current count
            const current = await redis.incr(key);
            
            // Set expiration on first request
            if (current === 1) {
                await redis.expire(key, Math.ceil(options.windowMs / 1000));
            }
            
            // Set headers
            res.set('X-RateLimit-Limit', options.maxRequests.toString());
            res.set('X-RateLimit-Remaining', Math.max(0, options.maxRequests - current).toString());
            
            const resetTime = await redis.ttl(key);
            res.set('X-RateLimit-Reset', (Date.now() + resetTime * 1000).toString());
            
            // Check if rate limited
            if (current > options.maxRequests) {
                return res.status(429).json({
                    error: {
                        code: 'RATE_LIMITED',
                        message: 'Too many requests',
                        retryAfter: resetTime
                    }
                });
            }
            
            next();
            
        } catch (err) {
            // If Redis fails, allow request (fail open)
            next();
        }
    };
}

// Usage
app.use(rateLimitMiddleware({ windowMs: 60000, maxRequests: 100 }));  // 100/min

// Different limits for different endpoints
app.post('/auth/login',
    rateLimitMiddleware({ windowMs: 3600000, maxRequests: 5 }),  // 5 per hour
    handleLogin
);

app.get('/api/search',
    rateLimitMiddleware({ windowMs: 60000, maxRequests: 10 }),  // 10 per minute
    handleSearch
);
```

---

## Part 6: Request Logging & Tracing

```typescript
// ============ REQUEST LOGGING ============

function loggingMiddleware(req: Request, res: Response, next: NextFunction) {
    const requestId = req.headers['x-request-id'] || generateUUID();
    const startTime = Date.now();
    
    // Attach to request for later use
    req.requestId = requestId;
    
    // Log request
    console.log({
        level: 'info',
        message: 'Incoming request',
        requestId,
        method: req.method,
        path: req.path,
        timestamp: new Date().toISOString()
    });
    
    // Intercept response
    const originalJson = res.json.bind(res);
    res.json = function(body: any) {
        const duration = Date.now() - startTime;
        
        // Log response
        console.log({
            level: res.statusCode >= 400 ? 'warn' : 'info',
            message: 'Outgoing response',
            requestId,
            method: req.method,
            path: req.path,
            status: res.statusCode,
            duration: `${duration}ms`,
            timestamp: new Date().toISOString()
        });
        
        // Add requestId to response
        body.requestId = requestId;
        
        return originalJson(body);
    };
    
    next();
}

app.use(loggingMiddleware);


// ============ STRUCTURED LOGGING WITH REQUEST ID ============

// All errors in response include requestId
// Client can provide it to support: "Your request ID: abc-123"
// Support can use ID to find logs: grep 'abc-123' logs.txt
// Correlates all services that handled the request
```

---

## Quick Reference: API Patterns Checklist

```
✓ Resource-oriented URLs (/api/users, not /api/getUsers)
✓ Correct HTTP methods (GET, POST, PUT, PATCH, DELETE)
✓ Correct status codes (200, 201, 400, 401, 403, 404, 429, 500)
✓ Proper error responses (with error code, message, details)
✓ Request ID in responses (for tracing)
✓ Caching headers (Cache-Control, ETag)
✓ Authentication (Authorization header with Bearer token)
✓ Authorization (Permission checks)
✓ Rate limiting (429 with Retry-After)
✓ CORS headers (if needed)
✓ Security headers (HSTS, CSP, X-Frame-Options)
✓ Idempotency keys (for POST/PATCH/DELETE)
✓ Proper pagination (page, limit, total)
✓ Versioning strategy (/api/v1/users vs /api/v2/users)
```
