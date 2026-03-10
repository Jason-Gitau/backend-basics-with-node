# Guards, Interceptors, & Filters in NestJS: Complete Deep Dive

## What This Guide Does

This guide teaches you to master **cross-cutting concerns** so you can:
- Understand the exact order NestJS executes Guards, Interceptors, and Filters
- Create reusable Guards for authentication and authorization
- Use Interceptors to transform requests/responses and handle timing
- Use Filters to centralize error handling
- Know the difference between each and when to use them
- Direct AI to create these components instead of repeating code in controllers
- Debug issues by understanding which layer is responsible

---

## Part 0: How This Completes Your NestJS Mastery

You've learned:

```
Modules        ← Organizational boundaries
    ↓
DI             ← Injectability, testability
    ↓
Services       ← Business logic implementation
    ↓
Controllers    ← HTTP handlers
    ↓
Guards, Interceptors, Filters  ← THIS GUIDE
                              Cross-cutting concerns
```

Together, these form the complete NestJS architecture:

```
REQUEST comes in
    ↓
[Middleware] ← Earliest
    ↓
[Guards] ← Authentication/Authorization
    ↓
[Pipes] ← Validation/Transformation
    ↓
[Controller] ← Extract params, call service
    ↓
[Service] ← Business logic
    ↓
[Interceptor - After] ← Transform response
    ↓
[Filter] ← Catch exceptions
    ↓
RESPONSE goes out

This is the NestJS request/response pipeline.
You need to understand every layer.
```

---

## Part 1: The Request/Response Pipeline (Low-Level Understanding)

### 1.1 The Complete Execution Order

This is critical. Understanding the order lets you know where each piece of logic should go.

```
1. Middleware (Express middleware runs first)
   ├─ Your custom middleware
   ├─ CORS, body-parser, etc.
   └─ This is the absolute earliest point

2. Guards (Authentication/Authorization checks)
   ├─ @UseGuards(JwtGuard)
   ├─ Runs BEFORE the controller
   ├─ Can prevent controller from running
   ├─ Asks: "Can this request proceed?"
   └─ Returns true (proceed) or false (reject)

3. Pipes (Data validation and transformation)
   ├─ @Body(ValidationPipe)
   ├─ Validates DTOs
   ├─ Transforms strings to numbers, etc.
   ├─ Throws BadRequestException if invalid
   └─ Can't continue if validation fails

4. Controller Method Executes
   ├─ Extracts parameters
   ├─ Calls service
   └─ Returns result

5. Interceptor - BEFORE (runs after controller, before service)
   ├─ @UseInterceptors(LoggingInterceptor)
   ├─ Can modify the request before it hits service
   ├─ Can intercept before service call
   └─ Receives the result

6. Interceptor - AFTER (response transformation)
   ├─ Can transform the response
   ├─ Can log the response
   ├─ Can measure execution time
   └─ Can add metadata to response

7. Filter (Exception handling)
   ├─ @UseFilters(ExceptionFilter)
   ├─ Catches ANY exception thrown
   ├─ If controller throws, filter catches it
   ├─ If service throws, filter catches it
   ├─ Formats the error response
   └─ This is the LAST chance to handle errors

8. Response is sent to client
```

### 1.2 Visualizing the Pipeline

```
REQUEST
  │
  ├─→ [Middleware] - CORS, auth headers, etc.
  │
  ├─→ [Guards] - Can I proceed?
  │     │
  │     └─ If guard returns false:
  │        └─→ ForbiddenException
  │            └─→ Filter catches it
  │
  ├─→ [Pipes] - Validate input
  │     │
  │     └─ If validation fails:
  │        └─→ BadRequestException
  │            └─→ Filter catches it
  │
  ├─→ [Controller] - Extract params, call service
  │     │
  │     ├─→ [Service] - Business logic
  │     │     │
  │     │     └─ If service throws:
  │     │        └─→ Filter catches it
  │     │
  │     └─ Returns response
  │
  ├─→ [Interceptor AFTER] - Transform response
  │
  ├─→ [Filter] - Last resort for errors
  │
  └─→ RESPONSE sent to client


Example: Request with invalid data

POST /orders { invalid: data }

ORDER OF EXECUTION:
1. Middleware runs ✓
2. Guards check (JWT valid?) ✓
3. Pipes validate (BadRequestException thrown) ✗
   └─→ Filter catches it
   └─→ Returns { statusCode: 400, message: "..." }

Controller and Service never run!
```

### 1.3 Request Context - The Request Object Flows Through Everything

```typescript
// The same request context flows through the entire pipeline

// Middleware
app.use((req, res, next) => {
  req.user = jwt.decode(req.headers.authorization);
  req.requestId = generateId();
  next();
});

// Guard
@Injectable()
export class JwtGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    // req.user is available (set by middleware)
    // req.requestId is available
    return !!req.user;
  }
}

// Interceptor
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const req = context.switchToHttp().getRequest();
    console.log(`Request ${req.requestId} started at ${Date.now()}`);
    return next.handle();
  }
}

// Filter
@Catch(HttpException)
export class ExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const req = ctx.getRequest();
    const res = ctx.getResponse();
    // req.requestId is available
    // Can log which request had an error
  }
}

// Key insight:
// The request object flows through EVERY layer
// You can add data at any layer and access it at later layers
// This is how you avoid repeating code
```

---

## Part 2: Guards - The Authentication/Authorization Layer

### 2.1 What Guards Do

A Guard is a class that implements `CanActivate`. It answers one question: **"Can this request proceed?"**

```typescript
export interface CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean>;
}

// Returns true  → Proceed to next step
// Returns false → Throw ForbiddenException
```

### 2.2 Types of Guards (Authentication vs Authorization)

#### **Type 1: Authentication Guard (Is the user who they claim to be?)**

```typescript
// ❌ BAD: Authentication logic in every controller
@Controller('orders')
export class OrderController {
  @Post()
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    // ❌ Check JWT every time
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) throw new UnauthorizedException('No token');
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = decoded;
    } catch (e) {
      throw new UnauthorizedException('Invalid token');
    }
    
    return this.orderService.createOrder(req.user.id, dto.items);
  }
}

// ✅ GOOD: Authentication guard (reusable)
@Injectable()
export class JwtGuard implements CanActivate {
  constructor(private configService: ConfigService) {}
  
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      throw new UnauthorizedException('No token provided');
    }
    
    try {
      const secret = this.configService.get('JWT_SECRET');
      const decoded = jwt.verify(token, secret);
      request.user = decoded;  // ← Attach user to request
      return true;  // Proceed
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}

// Usage: Apply to any route
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}
  
  @Post()
  @UseGuards(JwtGuard)  // ← One line replaces 15 lines of auth logic
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    // req.user is guaranteed to exist (guard ensures it)
    return this.orderService.createOrder(req.user.id, dto.items);
  }
}

// Can also apply globally:
// app.useGlobalGuards(new JwtGuard());
// All routes protected automatically
```

**Why This Matters:**
- ❌ Without guard: 15 lines of auth logic in every controller
- ✅ With guard: 1 line in the controller decorator
- ✅ Guard is testable (can mock it)
- ✅ Guard can be injected with dependencies (ConfigService, JwtService, etc.)
- ✅ Single place to change auth logic (affects entire app)

#### **Type 2: Authorization Guard (Does the user have permission for this action?)**

```typescript
// ❌ BAD: Role check in every controller
@Controller('admin')
export class AdminController {
  @Post('ban-user')
  async banUser(@Param('id') userId: string, @Req() req: any) {
    // ❌ Check permission every time
    if (!req.user || req.user.role !== 'admin') {
      throw new ForbiddenException('Only admins can ban users');
    }
    // ... ban user logic
  }
  
  @Post('create-report')
  async createReport(@Body() dto: any, @Req() req: any) {
    // ❌ Check permission every time
    if (!req.user || req.user.role !== 'admin') {
      throw new ForbiddenException('Only admins can create reports');
    }
    // ... create report logic
  }
}

// ✅ GOOD: Role-based guard (reusable)
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    if (!user) {
      throw new UnauthorizedException('User not authenticated');
    }
    
    // Check if route requires specific roles
    const reflector = new Reflector();
    const requiredRoles = reflector.get<string[]>('roles', context.getHandler());
    
    if (!requiredRoles) {
      return true;  // No role requirement
    }
    
    // Check if user has one of the required roles
    const hasRole = requiredRoles.some(role => user.role === role);
    
    if (!hasRole) {
      throw new ForbiddenException(
        `This action requires one of these roles: ${requiredRoles.join(', ')}`
      );
    }
    
    return true;
  }
}

// Custom decorator to specify required roles
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Usage: Simple one-liner on routes
@Controller('admin')
export class AdminController {
  @Post('ban-user')
  @UseGuards(RolesGuard)
  @Roles('admin')  // ← Specify required role
  async banUser(@Param('id') userId: string) {
    // Guard ensures user is admin
    // No role check needed here
    // ... ban user logic
  }
  
  @Post('create-report')
  @UseGuards(RolesGuard)
  @Roles('admin', 'moderator')  // ← Multiple allowed roles
  async createReport(@Body() dto: CreateReportDto) {
    // Guard ensures user is admin or moderator
    // ... create report logic
  }
}

// Benefits:
// ✅ Authorization logic in one place
// ✅ Reusable across multiple routes
// ✅ Single point to change permission rules
// ✅ Easy to test (can mock the decorator)
// ✅ Clean controller code
```

#### **Type 3: Composable Guards (Multiple Guards)**

```typescript
// Often you need multiple guards on one route

@Controller('orders')
export class OrderController {
  @Post()
  @UseGuards(JwtGuard, RolesGuard)  // ← Apply multiple guards
  @Roles('customer', 'admin')
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    // First: JwtGuard validates token and sets req.user
    // Second: RolesGuard checks if user has required role
    // Both must pass for controller to execute
    return this.orderService.createOrder(req.user.id, dto.items);
  }
}

// Execution order:
// 1. JwtGuard.canActivate() runs
// 2. If true, RolesGuard.canActivate() runs
// 3. If both true, controller runs
// 4. If any false, ForbiddenException thrown
```

### 2.3 Advanced: Custom Parameter Decorators with Guards

```typescript
// Often you want to extract the user from request easily

// ❌ BAD: Accessing request directly
@Post()
@UseGuards(JwtGuard)
async create(@Req() req: any, @Body() dto: CreateOrderDto) {
  const userId = req.user.id;  // Have to know request structure
  return this.orderService.createOrder(userId, dto.items);
}

// ✅ GOOD: Custom decorator extracts user
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  }
);

// Usage:
@Post()
@UseGuards(JwtGuard)
async create(@CurrentUser() user: UserPayload, @Body() dto: CreateOrderDto) {
  // user is extracted automatically
  // Type-safe (UserPayload type)
  // No need to access req.user
  return this.orderService.createOrder(user.id, dto.items);
}

// Can be even more specific:
export const UserId = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user?.id;
  }
);

// Usage:
@Post()
@UseGuards(JwtGuard)
async create(@UserId() userId: string, @Body() dto: CreateOrderDto) {
  // Just get the userId directly
  return this.orderService.createOrder(userId, dto.items);
}
```

### 2.4 Testing Guards

```typescript
describe('JwtGuard', () => {
  let guard: JwtGuard;
  let configService: ConfigService;
  
  beforeEach(() => {
    configService = {
      get: jest.fn().mockReturnValue('secret'),
    } as any;
    
    guard = new JwtGuard(configService);
  });
  
  it('should return true if valid JWT is provided', () => {
    const mockRequest = {
      headers: {
        authorization: 'Bearer valid.jwt.token',
      },
    };
    
    const context = {
      switchToHttp: () => ({
        getRequest: () => mockRequest,
      }),
    } as any;
    
    // Mock jwt.verify
    jest.spyOn(jwt, 'verify').mockReturnValue({ id: 'user123' } as any);
    
    expect(guard.canActivate(context)).toBe(true);
    expect(mockRequest.user).toEqual({ id: 'user123' });
  });
  
  it('should throw UnauthorizedException if no token', () => {
    const mockRequest = {
      headers: {},
    };
    
    const context = {
      switchToHttp: () => ({
        getRequest: () => mockRequest,
      }),
    } as any;
    
    expect(() => guard.canActivate(context)).toThrow(UnauthorizedException);
  });
  
  it('should throw UnauthorizedException if token is invalid', () => {
    const mockRequest = {
      headers: {
        authorization: 'Bearer invalid.token',
      },
    };
    
    const context = {
      switchToHttp: () => ({
        getRequest: () => mockRequest,
      }),
    } as any;
    
    jest.spyOn(jwt, 'verify').mockImplementation(() => {
      throw new Error('Invalid token');
    });
    
    expect(() => guard.canActivate(context)).toThrow(UnauthorizedException);
  });
});
```

### 2.5 Common Guard Patterns

#### **Pattern 1: Optional Authentication (Some routes public, some private)**

```typescript
// Guard that doesn't throw, just sets user if valid
@Injectable()
export class OptionalJwtGuard implements CanActivate {
  constructor(private configService: ConfigService) {}
  
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace('Bearer ', '');
    
    // If no token, just continue (don't throw)
    if (!token) {
      return true;
    }
    
    // If token provided, validate it
    try {
      const secret = this.configService.get('JWT_SECRET');
      const decoded = jwt.verify(token, secret);
      request.user = decoded;
    } catch (error) {
      // Invalid token, but don't throw (still allow request)
      // Just don't set req.user
    }
    
    return true;  // Always return true (optional)
  }
}

// Usage: Applied globally or selectively
@Controller('products')
export class ProductController {
  @Get()
  @UseGuards(OptionalJwtGuard)
  async list(@Req() req: any) {
    // If user is authenticated, show personalized products
    // If not authenticated, show public products
    
    if (req.user) {
      return this.productService.getPersonalizedProducts(req.user.id);
    }
    return this.productService.getPublicProducts();
  }
}
```

#### **Pattern 2: Ownership Guard (Check if user owns the resource)**

```typescript
// Ensure user can only access their own data

@Injectable()
export class OwnershipGuard implements CanActivate {
  constructor(private orderService: OrderService) {}
  
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    if (!user) {
      throw new UnauthorizedException('User not authenticated');
    }
    
    // Get the resource ID from route params
    const orderId = request.params.id;
    
    // Check if user owns this order
    const order = await this.orderService.getOrder(orderId);
    
    if (order.userId !== user.id) {
      throw new ForbiddenException('You can only access your own orders');
    }
    
    return true;
  }
}

// Usage:
@Controller('orders')
export class OrderController {
  @Get(':id')
  @UseGuards(JwtGuard, OwnershipGuard)  // Check auth AND ownership
  async getOrder(@Param('id') orderId: string) {
    return this.orderService.getOrder(orderId);
  }
}
```

#### **Pattern 3: API Key Guard (For service-to-service auth)**

```typescript
@Injectable()
export class ApiKeyGuard implements CanActivate {
  constructor(private configService: ConfigService) {}
  
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];
    
    if (!apiKey) {
      throw new UnauthorizedException('No API key provided');
    }
    
    const validApiKey = this.configService.get('API_KEY');
    
    if (apiKey !== validApiKey) {
      throw new UnauthorizedException('Invalid API key');
    }
    
    return true;
  }
}

// Usage: For internal APIs, webhooks, etc.
@Controller('webhooks')
export class WebhookController {
  @Post('stripe')
  @UseGuards(ApiKeyGuard)  // Verify API key instead of JWT
  async handleStripeWebhook(@Body() event: any) {
    // This endpoint can be called by Stripe
    // Requires valid API key
    return this.orderService.handleStripeEvent(event);
  }
}
```

---

## Part 3: Interceptors - Request/Response Transformation & Timing

### 3.1 What Interceptors Do

Interceptors implement `NestInterceptor`. They wrap the request/response:

```typescript
export interface NestInterceptor<T = any, R = any> {
  intercept(context: ExecutionContext, next: CallHandler<T>): Observable<R>;
}

// Key points:
// - Can run BEFORE controller
// - Can run AFTER controller
// - Returns Observable (RxJS)
// - Can transform request before controller
// - Can transform response after controller
// - Can measure timing
// - Can add metadata
```

### 3.2 Interceptor Execution Pattern

```typescript
// The typical interceptor pattern:

@Injectable()
export class TimingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    
    // BEFORE controller runs
    console.log('Request started at', now);
    
    return next.handle().pipe(
      tap(() => {
        // AFTER controller runs
        const duration = Date.now() - now;
        console.log(`Request took ${duration}ms`);
      }),
      // Can also use other RxJS operators:
      // catchError() - handle errors
      // map() - transform response
      // timeout() - limit execution time
    );
  }
}

// Execution flow:
// 1. BEFORE: Interceptor runs (console.log 'Request started')
// 2. → Controller runs
// 3. → Service runs
// 4. AFTER: tap() runs (console.log duration)
// 5. → Response returned
```

### 3.3 Common Interceptor Patterns

#### **Pattern 1: Logging Interceptor (Track all requests)**

```typescript
// ❌ BAD: Log in every controller
@Controller('orders')
export class OrderController {
  @Post()
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    console.log('Creating order', dto);
    const result = await this.orderService.createOrder(dto);
    console.log('Order created:', result);
    return result;
  }
  
  @Get(':id')
  async getOrder(@Param('id') id: string) {
    console.log('Getting order', id);
    const result = await this.orderService.getOrder(id);
    console.log('Order retrieved:', result);
    return result;
  }
}

// ✅ GOOD: Logging interceptor (reusable)
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const method = request.method;
    const url = request.url;
    const now = Date.now();
    
    // Log request
    this.logger.log(`[${method}] ${url}`);
    
    return next.handle().pipe(
      tap((data) => {
        // Log response
        const duration = Date.now() - now;
        this.logger.log(
          `[${method}] ${url} - ${duration}ms - Response: ${JSON.stringify(data)}`
        );
      }),
      catchError((error) => {
        // Log error
        const duration = Date.now() - now;
        this.logger.error(
          `[${method}] ${url} - ${duration}ms - Error: ${error.message}`
        );
        throw error;
      })
    );
  }
}

// Global usage: Apply to all routes
// In main.ts:
app.useGlobalInterceptors(new LoggingInterceptor(logger));

// Benefits:
// ✅ All requests/responses logged automatically
// ✅ No logging code in controllers
// ✅ Single place to change logging format
// ✅ Can disable for specific routes
```

#### **Pattern 2: Response Transformation Interceptor (Wrap responses)**

```typescript
// Often you want to wrap all responses in a consistent format

// ❌ BAD: Every controller returns different format
@Controller('orders')
export class OrderController {
  @Get(':id')
  async getOrder(@Param('id') id: string) {
    const order = await this.orderService.getOrder(id);
    // One format: just data
    return order;
  }
}

@Controller('products')
export class ProductController {
  @Get(':id')
  async getProduct(@Param('id') id: string) {
    const product = await this.productService.getProduct(id);
    // Another format: { success: true, data }
    return { success: true, data: product };
  }
}

// ✅ GOOD: Interceptor wraps all responses
@Injectable()
export class ResponseInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data: data,
        timestamp: new Date().toISOString(),
      }))
    );
  }
}

// Global usage:
app.useGlobalInterceptors(new ResponseInterceptor());

// All responses automatically formatted:
// GET /orders/123 returns:
// {
//   "success": true,
//   "data": { id: "123", total: 100, ... },
//   "timestamp": "2024-03-10T10:30:00.000Z"
// }

// Front-end always knows the format
// No inconsistencies
```

#### **Pattern 3: Error Handling Interceptor (Timing, retries, etc.)**

```typescript
// Handle transient errors with retry logic

@Injectable()
export class RetryInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      // Retry up to 3 times on failure
      retry(3),
      // With exponential backoff
      retryWhen((errors) =>
        errors.pipe(
          mergeMap((error, index) => {
            if (index >= 3) {
              return throwError(() => error);
            }
            const delayMs = Math.pow(2, index) * 100;  // 100ms, 200ms, 400ms
            return timer(delayMs);
          })
        )
      ),
      // Timeout after 5 seconds
      timeout(5000)
    );
  }
}

// Useful for:
// - Database connection timeouts (retry)
// - External API calls (retry with backoff)
// - Ensuring request completes within time limit
```

#### **Pattern 4: Caching Interceptor (Cache responses)**

```typescript
// Cache GET requests to avoid hitting database repeatedly

@Injectable()
export class CachingInterceptor implements NestInterceptor {
  private cache = new Map<string, any>();
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    
    // Only cache GET requests
    if (request.method !== 'GET') {
      return next.handle();
    }
    
    const cacheKey = `${request.method}:${request.url}`;
    
    // Check if cached
    if (this.cache.has(cacheKey)) {
      return of(this.cache.get(cacheKey));  // Return cached value
    }
    
    // Not cached, execute and cache result
    return next.handle().pipe(
      tap((data) => {
        this.cache.set(cacheKey, data);
        // Optionally set TTL
        setTimeout(() => this.cache.delete(cacheKey), 60000);  // 1 minute
      })
    );
  }
}

// GET /products/123 - First call: hits database
// GET /products/123 - Second call (within 1 min): returns cached
```

#### **Pattern 5: Transformation Interceptor (Add metadata)**

```typescript
// Add metadata to every response (user info, request ID, etc.)

@Injectable()
export class MetadataInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();
    
    return next.handle().pipe(
      map((data) => ({
        data,
        meta: {
          timestamp: new Date().toISOString(),
          requestId: request.headers['x-request-id'] || generateId(),
          userId: request.user?.id || null,
          duration: Date.now() - request.startTime,  // Set by middleware
        },
      }))
    );
  }
}

// Response:
// {
//   "data": { ... },
//   "meta": {
//     "timestamp": "2024-03-10T10:30:00.000Z",
//     "requestId": "req-xyz-123",
//     "userId": "user-456",
//     "duration": 145
//   }
// }
```

### 3.4 Testing Interceptors

```typescript
describe('LoggingInterceptor', () => {
  let interceptor: LoggingInterceptor;
  let logger: Logger;
  
  beforeEach(() => {
    logger = { log: jest.fn(), error: jest.fn() } as any;
    interceptor = new LoggingInterceptor(logger);
  });
  
  it('should log request and response', (done) => {
    const context = {
      switchToHttp: () => ({
        getRequest: () => ({
          method: 'GET',
          url: '/orders/123',
        }),
      }),
    } as any;
    
    const handler = {
      handle: () => of({ id: '123', total: 100 }),
    };
    
    interceptor.intercept(context, handler).subscribe(() => {
      expect(logger.log).toHaveBeenCalledWith('[GET] /orders/123');
      expect(logger.log).toHaveBeenCalledWith(
        expect.stringContaining('[GET] /orders/123')
      );
      done();
    });
  });
  
  it('should log errors', (done) => {
    const context = {
      switchToHttp: () => ({
        getRequest: () => ({
          method: 'POST',
          url: '/orders',
        }),
      }),
    } as any;
    
    const handler = {
      handle: () => throwError(() => new Error('Database error')),
    };
    
    interceptor.intercept(context, handler).subscribe(
      () => {},
      (error) => {
        expect(logger.error).toHaveBeenCalledWith(
          expect.stringContaining('Database error')
        );
        done();
      }
    );
  });
});
```

---

## Part 4: Filters - Exception Handling & Error Response Formatting

### 4.1 What Filters Do

Filters implement `ExceptionFilter`. They catch exceptions and transform them into proper HTTP responses:

```typescript
export interface ExceptionFilter<T = any> {
  catch(exception: T, host: ArgumentsHost): void;
}

// Filters:
// - Are the LAST step in the pipeline
// - Catch any exception thrown anywhere
// - Can transform exception into HTTP response
// - Can log the exception
// - Can format error message for client
```

### 4.2 The Problem Without Filters

```typescript
// Without filters, errors are handled inconsistently

@Injectable()
export class OrderService {
  async getOrder(id: string) {
    const order = await this.orderRepository.findById(id);
    
    // Different services throw different errors
    if (!order) {
      throw new Error('Order not found');  // Generic Error
    }
    
    return order;
  }
}

@Injectable()
export class ProductService {
  async getProduct(id: string) {
    const product = await this.productRepository.findById(id);
    
    // Different exception type
    if (!product) {
      throw new NotFoundException('Product not found');  // Specific NestJS error
    }
    
    return product;
  }
}

// Client receives inconsistent error responses:
// GET /orders/invalid
// Response: 500 Internal Server Error
// Body: Error message not shown

// GET /products/invalid
// Response: 404 Not Found
// Body: { statusCode: 404, message: "Product not found", error: "Not Found" }

// Problem: No consistency, no proper HTTP status codes
```

### 4.3 Solution: Exception Filters

```typescript
// ✅ GOOD: Centralized exception handling

// Define custom exceptions
export class OrderNotFoundException extends Error {
  constructor(orderId: string) {
    super(`Order ${orderId} not found`);
    this.name = 'OrderNotFoundException';
  }
}

export class InsufficientStockException extends Error {
  constructor(productId: string, requested: number, available: number) {
    super(
      `Product ${productId}: requested ${requested}, available ${available}`
    );
    this.name = 'InsufficientStockException';
  }
}

// Create filter to catch custom exceptions
@Catch(OrderNotFoundException)
export class OrderNotFoundFilter implements ExceptionFilter {
  catch(exception: OrderNotFoundException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    
    response.status(404).json({
      statusCode: 404,
      message: exception.message,
      error: 'Order Not Found',
      timestamp: new Date().toISOString(),
    });
  }
}

// Filter for business logic errors
@Catch(InsufficientStockException)
export class InsufficientStockFilter implements ExceptionFilter {
  catch(exception: InsufficientStockException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    
    response.status(400).json({
      statusCode: 400,
      message: exception.message,
      error: 'Insufficient Stock',
      timestamp: new Date().toISOString(),
    });
  }
}

// Global filter to catch all exceptions
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private logger: Logger) {}
  
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    
    let status = 500;
    let message = 'Internal Server Error';
    
    // Check if it's a known HttpException
    if (exception instanceof HttpException) {
      status = exception.getStatus();
      message = exception.message;
    }
    
    // Log the error
    this.logger.error(
      `[${request.method}] ${request.url} - ${status} - ${message}`,
      exception instanceof Error ? exception.stack : String(exception)
    );
    
    response.status(status).json({
      statusCode: status,
      message: message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// Usage: Apply filters
@Controller('orders')
@UseFilters(OrderNotFoundFilter, InsufficientStockFilter)
export class OrderController {
  @Get(':id')
  async getOrder(@Param('id') id: string) {
    // If service throws OrderNotFoundException
    // → OrderNotFoundFilter catches it
    // → Returns 404 with proper format
    return this.orderService.getOrder(id);
  }
  
  @Post()
  async create(@Body() dto: CreateOrderDto) {
    // If service throws InsufficientStockException
    // → InsufficientStockFilter catches it
    // → Returns 400 with proper format
    return this.orderService.createOrder(dto);
  }
}

// Global usage: Applied to entire app
app.useGlobalFilters(new GlobalExceptionFilter(logger));
```

### 4.4 Custom Exception Hierarchy

For better organization, create a custom exception hierarchy:

```typescript
// Base exception with common properties
export abstract class BaseException extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly message: string,
    public readonly code?: string
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

// Domain-specific exceptions
export class OrderNotFoundException extends BaseException {
  constructor(orderId: string) {
    super(404, `Order ${orderId} not found`, 'ORDER_NOT_FOUND');
  }
}

export class PaymentFailedException extends BaseException {
  constructor(reason: string) {
    super(402, `Payment failed: ${reason}`, 'PAYMENT_FAILED');
  }
}

export class InsufficientPermissionException extends BaseException {
  constructor(resource: string) {
    super(403, `Insufficient permission for ${resource}`, 'INSUFFICIENT_PERMISSION');
  }
}

// Generic filter that handles all custom exceptions
@Catch(BaseException)
export class CustomExceptionFilter implements ExceptionFilter {
  constructor(private logger: Logger) {}
  
  catch(exception: BaseException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    
    this.logger.error(
      `[${request.method}] ${request.url} - ${exception.statusCode} - ${exception.message}`,
      exception.stack
    );
    
    response.status(exception.statusCode).json({
      statusCode: exception.statusCode,
      message: exception.message,
      code: exception.code,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// Benefits:
// ✅ All custom exceptions handled consistently
// ✅ Proper HTTP status codes
// ✅ Proper error codes for client handling
// ✅ Automatic logging
// ✅ No try/catch blocks scattered everywhere
```

### 4.5 Testing Filters

```typescript
describe('OrderNotFoundFilter', () => {
  let filter: OrderNotFoundFilter;
  
  beforeEach(() => {
    filter = new OrderNotFoundFilter();
  });
  
  it('should catch OrderNotFoundException and return 404', () => {
    const mockResponse = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn(),
    };
    
    const mockRequest = {
      method: 'GET',
      url: '/orders/invalid',
    };
    
    const host = {
      switchToHttp: () => ({
        getResponse: () => mockResponse,
        getRequest: () => mockRequest,
      }),
    } as any;
    
    const exception = new OrderNotFoundException('invalid-id');
    
    filter.catch(exception, host);
    
    expect(mockResponse.status).toHaveBeenCalledWith(404);
    expect(mockResponse.json).toHaveBeenCalledWith(
      expect.objectContaining({
        statusCode: 404,
        message: expect.stringContaining('Order'),
        error: 'Order Not Found',
      })
    );
  });
});
```

---

## Part 5: Pipes - Validation & Transformation

Pipes aren't Filters/Guards/Interceptors, but they're in the pipeline and important:

### 5.1 What Pipes Do

Pipes implement `PipeTransform`. They validate and transform input data:

```typescript
export interface PipeTransform<T = any, R = any> {
  transform(value: T, metadata: ArgumentMetadata): R;
}

// Pipes:
// - Transform input data (string to number, string to Date)
// - Validate input data (DTO validation)
// - Throw BadRequestException if invalid
// - Run before controller
```

### 5.2 Built-in Pipes

```typescript
import { ValidationPipe, ParseIntPipe, ParseUUIDPipe } from '@nestjs/common';

@Controller('orders')
export class OrderController {
  // Validate DTO
  @Post()
  async create(
    @Body(new ValidationPipe()) dto: CreateOrderDto
  ) {
    // ValidationPipe validates dto against CreateOrderDto schema
    // If invalid, throws BadRequestException
    // Controller only runs if validation passes
    return this.orderService.createOrder(dto);
  }
  
  // Parse string to number
  @Get(':id')
  async getOrder(
    @Param('id', new ParseIntPipe()) id: number  // ← id is number now
  ) {
    return this.orderService.getOrder(id);
  }
  
  // Validate UUID format
  @Get(':id')
  async getOrderByUuid(
    @Param('id', new ParseUUIDPipe()) id: string
  ) {
    // id must be valid UUID
    return this.orderService.getOrder(id);
  }
}
```

### 5.3 Global Validation Pipe

```typescript
// Usually apply validation globally in main.ts

import { ValidationPipe } from '@nestjs/common';
import { ClassSerializerInterceptor, ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,         // Remove unknown properties
      forbidNonWhitelisted: true,  // Throw error if unknown properties
      transform: true,         // Auto-transform to DTO class
      transformOptions: {
        enableImplicitConversion: true,  // Convert types automatically
      },
    })
  );
  
  await app.listen(3000);
}

bootstrap();

// Usage: No need to specify pipe on every route
@Controller('orders')
export class OrderController {
  @Post()
  async create(@Body() dto: CreateOrderDto) {
    // ValidationPipe automatically applied
    // dto is validated and transformed
    return this.orderService.createOrder(dto);
  }
}
```

### 5.4 Custom Pipes

```typescript
// Create a custom pipe for business logic validation

@Injectable()
export class OrderStatusPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    const validStatuses = ['pending', 'processing', 'completed', 'cancelled'];
    
    if (!validStatuses.includes(value)) {
      throw new BadRequestException(
        `Invalid status. Must be one of: ${validStatuses.join(', ')}`
      );
    }
    
    return value;
  }
}

// Usage:
@Controller('orders')
export class OrderController {
  @Patch(':id')
  async updateStatus(
    @Param('status', new OrderStatusPipe()) status: string
  ) {
    // status is validated before controller runs
    return this.orderService.updateStatus(status);
  }
}
```

---

## Part 6: Middleware - Request Processing at Entry Point

Middleware is technically not part of Filters/Guards/Interceptors, but it's the first entry point:

### 6.1 What Middleware Does

Middleware runs at the Express level, before anything else:

```typescript
// Create custom middleware
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Generate or get request ID
    const requestId = req.headers['x-request-id'] as string || generateId();
    
    // Attach to request
    req.requestId = requestId;
    
    // Pass to next middleware
    next();
  }
}

// Usage: Apply to module
@Module({
  controllers: [AppController],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(RequestIdMiddleware)
      .forRoutes('*');  // Apply to all routes
  }
}

// Or functional middleware:
app.use((req, res, next) => {
  req.requestId = generateId();
  next();
});
```

### 6.2 Common Middleware Uses

```typescript
// CORS handling
app.use(cors());

// Body parsing
app.use(express.json({ limit: '10mb' }));

// Request logging at Express level
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// Authentication (set user on request)
app.use((req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (token) {
    try {
      req.user = jwt.verify(token, process.env.JWT_SECRET);
    } catch (e) {
      // Not authenticated, but don't throw
      // Guards will handle authentication later
    }
  }
  next();
});
```

---

## Part 7: The Complete Pipeline - Putting It All Together

### 7.1 Request Flow with Everything

```
REQUEST: POST /orders with JWT token

1. [Middleware] RequestIdMiddleware
   └─ Add requestId to request

2. [Middleware] Authentication Middleware
   └─ Decode JWT, set req.user

3. [Guard] JwtGuard
   └─ Verify JWT (req.user exists)
   └─ If false: throw UnauthorizedException
      → Filter catches it → 401 response

4. [Guard] RolesGuard
   └─ Check user has 'customer' role
   └─ If false: throw ForbiddenException
      → Filter catches it → 403 response

5. [Pipe] ValidationPipe
   └─ Validate request body (CreateOrderDto)
   └─ If invalid: throw BadRequestException
      → Filter catches it → 400 response

6. [Controller] OrderController.create()
   └─ Extract parameters
   └─ Call OrderService.createOrder()

7. [Service] OrderService.createOrder()
   └─ Business logic
   └─ If error: throw PaymentFailedException
      → Filter catches it → 402 response

8. [Interceptor] LoggingInterceptor (AFTER)
   └─ Log request/response
   └─ Log duration

9. [Interceptor] ResponseInterceptor
   └─ Wrap response in { success: true, data: ... }

10. [Filter] GlobalExceptionFilter
    └─ No exception, so doesn't do anything
    └─ Request completed successfully

11. RESPONSE: 200 OK
    └─ Body: { success: true, data: { id, items, total, ... } }
```

### 7.2 Example: Complete Order Creation Flow

```typescript
// 1. Middleware
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req.requestId = generateId();
    next();
  }
}

// 2. Guard
@Injectable()
export class JwtGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace('Bearer ', '');
    
    if (!token) throw new UnauthorizedException('No token');
    
    try {
      request.user = jwt.verify(token, process.env.JWT_SECRET);
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }
}

// 3. Interceptor
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const start = Date.now();
    
    this.logger.log(`[${request.requestId}] Request: ${request.method} ${request.url}`);
    
    return next.handle().pipe(
      tap((data) => {
        const duration = Date.now() - start;
        this.logger.log(
          `[${request.requestId}] Response: ${duration}ms`
        );
      }),
      catchError((error) => {
        const duration = Date.now() - start;
        this.logger.error(
          `[${request.requestId}] Error: ${error.message} (${duration}ms)`
        );
        throw error;
      })
    );
  }
}

// 4. Filter
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private logger: Logger) {}
  
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    
    let status = 500;
    let message = 'Internal Server Error';
    
    if (exception instanceof HttpException) {
      status = exception.getStatus();
      message = exception.message;
    }
    
    this.logger.error(
      `[${request.requestId}] Exception: ${status} - ${message}`
    );
    
    response.status(status).json({
      statusCode: status,
      message: message,
      requestId: request.requestId,
      timestamp: new Date().toISOString(),
    });
  }
}

// 5. Service
@Injectable()
export class OrderService {
  async createOrder(userId: string, items: Item[]) {
    // Business logic
    const order = await this.orderRepository.save({ userId, items });
    return order;
  }
}

// 6. Controller
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}
  
  @Post()
  @UseGuards(JwtGuard)
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    return this.orderService.createOrder(req.user.id, dto.items);
  }
}

// Request flow:
// POST /orders { items: [...] }
// Authorization: Bearer token...
//
// → Middleware: requestId = "req-123"
// → Guard: User authenticated
// → Pipe: DTO validated
// → Controller: Call service
// → Service: Create order in database
// → Interceptor: Log response
// → Response: { statusCode: 201, data: {...}, requestId: "req-123" }
```

---

## Part 8: Common Mistakes & Patterns

### Mistake 1: Business Logic in Guard

```typescript
// ❌ BAD: Guard has business logic
@Injectable()
export class OwnershipGuard implements CanActivate {
  constructor(private orderService: OrderService) {}
  
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const orderId = request.params.id;
    
    // ❌ Calling orderService here is inefficient
    // If guard returns false, we've wasted the service call
    // Business logic should be in service
    const order = await this.orderService.getOrder(orderId);
    
    if (order.userId !== request.user.id) {
      throw new ForbiddenException();
    }
    
    return true;
  }
}

// ✅ GOOD: Lightweight guard, business logic in service
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    
    // Just check: is user authenticated?
    return !!request.user;
  }
}

// Business logic in service:
async getOrder(id: string, userId: string) {
  const order = await this.orderRepository.findById(id);
  
  if (!order) throw new NotFoundException();
  
  // Check ownership here, in business logic
  if (order.userId !== userId) {
    throw new ForbiddenException('You can only access your own orders');
  }
  
  return order;
}

// Usage:
@Get(':id')
@UseGuards(AuthGuard)  // Just check auth
async getOrder(@Param('id') id: string, @Req() req: any) {
  // Service handles ownership check
  return this.orderService.getOrder(id, req.user.id);
}
```

### Mistake 2: Duplicated Guards on Every Route

```typescript
// ❌ BAD: Apply guard to every route
@Controller('orders')
export class OrderController {
  @Get()
  @UseGuards(JwtGuard)
  list() { }
  
  @Get(':id')
  @UseGuards(JwtGuard)
  getOne() { }
  
  @Post()
  @UseGuards(JwtGuard)
  create() { }
  
  @Patch(':id')
  @UseGuards(JwtGuard)
  update() { }
  
  @Delete(':id')
  @UseGuards(JwtGuard)
  delete() { }
}

// ✅ GOOD: Apply guard to controller
@Controller('orders')
@UseGuards(JwtGuard)  // All routes in this controller use JwtGuard
export class OrderController {
  @Get()
  list() { }
  
  @Get(':id')
  getOne() { }
  
  @Post()
  create() { }
  
  @Patch(':id')
  update() { }
  
  @Delete(':id')
  delete() { }
}

// ✅ EVEN BETTER: Apply globally
// In main.ts:
app.useGlobalGuards(new JwtGuard());
// All routes protected by default

// Exceptions:
@Get('/public')
@SkipGuards(JwtGuard)  // Skip guard for this route
getPublic() { }
```

### Mistake 3: Too Much in Interceptors

```typescript
// ❌ BAD: Interceptor has complex logic
@Injectable()
export class ComplexInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      // Validate business logic
      tap((data) => {
        if (data.total > 10000) {
          // Send approval email
          // Create audit log
          // Update analytics
          // Call webhook
        }
      })
    );
  }
}

// ✅ GOOD: Interceptor is simple, business logic in service
@Injectable()
export class SimpleLoggingInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    
    return next.handle().pipe(
      tap(() => {
        // Just log
        this.logger.log(`Request took ${Date.now() - start}ms`);
      })
    );
  }
}

// Business logic stays in service:
async createOrder(userId: string, items: Item[]) {
  const order = await this.orderRepository.save({ userId, items });
  
  if (order.total > 10000) {
    await this.emailService.sendApprovalRequest(order);
    await this.auditService.log('large_order', order);
    await this.analyticsService.track('large_order_created', order);
  }
  
  return order;
}
```

### Mistake 4: Not Testing Guards/Filters/Interceptors

```typescript
// Test your cross-cutting concerns!

describe('JwtGuard', () => {
  let guard: JwtGuard;
  
  beforeEach(() => {
    guard = new JwtGuard();
  });
  
  it('should return true with valid token', () => {
    const context = createMockContext({
      headers: {
        authorization: 'Bearer valid.token',
      },
    });
    
    jest.spyOn(jwt, 'verify').mockReturnValue({ id: 'user123' } as any);
    
    expect(guard.canActivate(context)).toBe(true);
  });
  
  it('should throw with invalid token', () => {
    const context = createMockContext({
      headers: {
        authorization: 'Bearer invalid.token',
      },
    });
    
    jest.spyOn(jwt, 'verify').mockImplementation(() => {
      throw new Error('Invalid token');
    });
    
    expect(() => guard.canActivate(context)).toThrow(UnauthorizedException);
  });
});
```

---

## Part 9: Directing AI to Create These Components

### 9.1 Creating Guards

```
PROMPT TO AI:

"Create a JWT authentication guard with these requirements:

1. Class: JwtGuard implements CanActivate
2. Inject: ConfigService (to get JWT_SECRET)
3. Extract token from Authorization header
4. Parse as Bearer token (remove 'Bearer ' prefix)
5. Verify token using jwt.verify()
6. Attach decoded token to req.user
7. Return true if valid
8. Throw UnauthorizedException if:
   - No token provided
   - Token is invalid
   - Token is expired

Do not:
- Use hardcoded secrets
- Assume token format is different
- Catch errors silently

Include JSDoc comments explaining each step."
```

### 9.2 Creating Interceptors

```
PROMPT TO AI:

"Create a logging interceptor with these requirements:

1. Class: LoggingInterceptor implements NestInterceptor
2. Inject: Logger service
3. Extract method and URL from request
4. Log the request
5. Measure execution time (Date.now())
6. Use RxJS operators:
   - tap() to log successful response
   - catchError() to log errors
7. Log format: [METHOD] URL - DURATIONms
8. Also log if an error occurs

Use RxJS operators (tap, catchError).
Do not use async/await in intercept method.
Return Observable."
```

### 9.3 Creating Filters

```
PROMPT TO AI:

"Create an exception filter for custom business exceptions:

1. Class: BusinessExceptionFilter implements ExceptionFilter
2. Catch: Any exception
3. Extract from ExecutionContext:
   - Request (method, URL, headers)
   - Response object
4. Check exception type and determine:
   - HTTP status code
   - Error message
   - Error code
5. Log the error with:
   - Method and URL
   - Status code
   - Error message
   - Request ID (from headers or generate)
6. Return JSON response with:
   - statusCode
   - message
   - code (error code)
   - requestId
   - timestamp

For unknown exceptions: return 500 Internal Server Error

Do not:
- Expose stack traces to client
- Expose database details
- Swallow errors silently"
```

---

## Part 10: Integration Checklist

When designing cross-cutting concerns, use this checklist:

```
BEFORE IMPLEMENTATION:
□ Where in the pipeline does this logic belong?
  □ Middleware? (Express level, auth headers)
  □ Guard? (Allow/deny entry)
  □ Pipe? (Validate/transform input)
  □ Interceptor? (Wrap request/response)
  □ Filter? (Handle exceptions)

□ Should it be applied globally or per-route?
  □ Global: app.useGlobal...()
  □ Per-controller: @Controller() @Use...()
  □ Per-route: @Get() @Use...()

□ What dependencies does it need?
  □ Logger? → Inject Logger
  □ ConfigService? → Inject ConfigService
  □ Custom service? → Inject it

□ Is it testable?
  □ Can mock dependencies?
  □ Can test with mock ExecutionContext?

DURING IMPLEMENTATION:
□ Guard: Returns boolean or throws exception
□ Interceptor: Returns Observable (use RxJS)
□ Filter: Calls response.status().json()
□ Pipe: Returns transformed value or throws

WHEN INTEGRATING:
□ Order matters:
  □ Middleware → Guard → Pipe → Controller → Service → Interceptor → Filter
  □ Exception handling: Guard can throw, Interceptor can throw, Service can throw
  □ All exceptions caught by Filter

□ Request context flows through:
  □ Middleware can set req properties
  □ Guard can access req properties
  □ Service doesn't know about req
  □ Interceptor can access req

□ No duplication:
  □ Auth logic: Use Guard, not repeated in service
  □ Logging: Use Interceptor, not in controllers
  □ Error formatting: Use Filter, not in services
  □ Validation: Use Pipe, not in service
```

---

## Summary: Guards, Interceptors, Filters

**Guards** - Answer "Can this request proceed?"
- Authentication (is user who they claim?)
- Authorization (does user have permission?)
- Returns true/false or throws exception

**Interceptors** - Wrap request/response
- Logging (all requests/responses)
- Transformation (format responses)
- Timing (measure duration)
- Caching (avoid repeated work)

**Filters** - Handle exceptions
- Catch exceptions from anywhere
- Format error response
- Log errors
- Transform exceptions to HTTP responses

**Pipes** - Validate/transform input
- Validate DTOs
- Transform types (string to number)
- Throw BadRequestException if invalid

**Middleware** - Entry point
- CORS, body-parser
- Request ID generation
- Early authentication (set user object)

**Complete Pipeline:**
Middleware → Guard → Pipe → Controller → Service → Interceptor → Filter → Response

**Your Job as Architect:**
1. Identify which layer each concern belongs in
2. Create reusable components (guards, filters, interceptors)
3. Apply them globally or per-route
4. Avoid repeating code in controllers/services
5. Direct AI to create these instead of business logic in controllers

---

## Connecting to Previous Parts

This completes your NestJS architecture knowledge:

```
Part 1: Modules
├─ Organize code into boundaries
└─ DI works within module boundaries

Part 2: Dependency Injection
├─ Everything is injectable
└─ Makes testing possible

Part 3: Providers & Services
├─ Services implement business logic
└─ Clean separation of concerns

Part 4: Guards, Interceptors, Filters (THIS PART)
├─ Cross-cutting concerns
├─ Authentication/Authorization
├─ Request/Response transformation
├─ Error handling
└─ Completes the architecture

Together:
✅ Modules define boundaries
✅ DI makes everything testable
✅ Services implement logic
✅ Guards/Interceptors/Filters handle concerns
✅ Controllers are thin HTTP handlers
✅ Clean, maintainable, production-grade
```

You now have the complete mental model of NestJS. You can design any system, direct AI to implement it, review the code, and maintain it effectively.
