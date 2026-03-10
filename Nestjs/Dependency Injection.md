# Dependency Injection in NestJS: Complete Deep Dive

## What This Guide Does
This guide teaches you to master **Dependency Injection (DI)** so you can:
- Write code that's automatically testable
- Understand who "owns" data and when to pass it around
- Direct AI to generate code that doesn't need refactoring for testing
- Debug DI issues by understanding the dependency graph

---

## Part 1: The Core Problem DI Solves

### 1.1 Without Dependency Injection (The Problem)

Let's start with code that doesn't use DI. This is what happens when classes create their own dependencies:

```typescript
// ❌ BAD: No DI - Class creates its own dependencies

export class OrderService {
  // Problem 1: This service creates the database connection itself
  private db = new DatabaseConnection({
    host: 'localhost',
    port: 5432,
    database: 'myapp'
  });
  
  // Problem 2: Creates its own repository
  private orderRepository = new OrderRepository(this.db);
  
  // Problem 3: Creates its own external service
  private stripePaymentGateway = new StripeGateway('sk_live_...');
  
  createOrder(userId: string, items: any[]) {
    // Implementation uses dependencies it created
    const order = this.orderRepository.save({ userId, items });
    await this.stripePaymentGateway.charge(...);
    return order;
  }
}

// Now try to use it:
const orderService = new OrderService();
orderService.createOrder('user123', []);
```

**What's the problem?**

```
Issue 1: TESTING IS IMPOSSIBLE
├─ You can't test this without:
│  ├─ An actual database running (slow, flaky)
│  ├─ Stripe API being available (slow, costs money)
│  └─ All real dependencies present (can't isolate)
└─ You can't test "what if Stripe returns an error?" 
   without making Stripe actually fail

Issue 2: HARD TO CHANGE
├─ Want to switch from Stripe to PayPal?
│  └─ Can't, it's hardcoded inside the class
├─ Want to use a test database for testing?
│  └─ Can't, it's hardcoded inside the class
└─ Want to use a mock Stripe for a feature branch?
   └─ Can't, it's hardcoded inside the class

Issue 3: HARD TO UNDERSTAND
├─ You can't tell from the class definition what it depends on
└─ You have to read the constructor to find out
   (And even then, you might miss something created inside methods)

Issue 4: CANNOT BE COMPOSED
├─ You can't use OrderService in different contexts
│  ├─ One context might want to use PostgreSQL
│  ├─ Another context might want to use MongoDB
│  └─ One class can only do one thing
└─ Fails the "dependency composition" principle
```

### 1.2 The Real-World Impact

Imagine you write the code above. Now here's what happens in the real world:

```
Day 1: Tests pass, everything works locally ✅

Day 2: New developer joins
├─ Clones repo
├─ Tries to run tests
├─ Tests fail: "Can't connect to database"
├─ Spends 2 hours setting up PostgreSQL
└─ Still fails because Stripe credentials are hardcoded

Day 3: You need to test payment failure scenario
├─ You want to test: "What if Stripe returns an error?"
├─ Can't. You'd need to break Stripe API or mock it somehow
├─ Instead: You test manually against real Stripe
└─ Test is slow (API call takes time, uses real credits)

Day 4: You want to refactor OrderService to use async/await
├─ You make the change
├─ Now you need to test the new logic
├─ But you can't run tests without all dependencies
├─ You're blocked

Week 2: Product says "use PayPal instead of Stripe"
├─ You need to change OrderService
├─ But OrderService creates StripeGateway internally
├─ You have to rewrite the whole class
├─ Risk of introducing bugs
└─ Testing is still a nightmare
```

### 1.3 With Dependency Injection (The Solution)

```typescript
// ✅ GOOD: Using Dependency Injection - Class receives dependencies

export class OrderService {
  // Dependencies are declared, not created
  constructor(
    private orderRepository: OrderRepository,
    private paymentGateway: PaymentGateway  // Note: it's an interface, not StripeGateway
  ) {}
  
  createOrder(userId: string, items: any[]) {
    // Implementation uses injected dependencies
    const order = this.orderRepository.save({ userId, items });
    await this.paymentGateway.charge(...);
    return order;
  }
}

// Now using it is different:
const orderService = new OrderService(
  new OrderRepository(realDatabase),
  new StripeGateway('sk_live_...')
);
orderService.createOrder('user123', []);

// And testing it is easy:
const mockRepository = { save: jest.fn() };
const mockPaymentGateway = { charge: jest.fn() };

const orderService = new OrderService(
  mockRepository,
  mockPaymentGateway
);

// Now you can test specific scenarios:
mockPaymentGateway.charge.mockRejectedValue(new Error('Declined'));
const result = await orderService.createOrder('user123', []);
// Test: how does the service handle payment failure?
```

**What changed?**

```
✅ TESTING IS TRIVIAL
├─ No database needed, use mock
├─ No Stripe needed, use mock
├─ Can test payment failure without breaking anything
└─ Tests run instantly (no I/O)

✅ EASY TO CHANGE
├─ Switch Stripe to PayPal? Pass a different paymentGateway
├─ Use different database? Pass different repository
├─ Test with test database? Pass test repository
└─ All without changing OrderService code

✅ CLEAR DEPENDENCIES
├─ The constructor explicitly shows what this class needs
├─ No hidden dependencies created inside methods
├─ Easy to understand the class at a glance
└─ IDE can help you autocomplete

✅ COMPOSABLE
├─ OrderService works the same way in any context
├─ Can be used with Stripe or PayPal
├─ Can be used with PostgreSQL or MongoDB
├─ Can be used with real or fake implementations
└─ This is flexibility
```

---

## Part 2: Understanding the DI Principle

### 2.1 The Golden Rule of Dependency Injection

```
A class should NOT create its dependencies.
A class should RECEIVE its dependencies.

More specifically:

❌ WRONG: OrderService creates OrderRepository
✅ RIGHT: OrderService receives OrderRepository from outside

The "outside" is usually:
1. The constructor (most common)
2. A setter method (less common)
3. A factory (advanced, rarely needed)
```

### 2.2 Why This Matters: The Ownership Question

**Key Question for every dependency: "Who owns this data?"**

This question is the heart of DI.

```typescript
// Example: OrderService needs to access orders

export class OrderService {
  constructor(private orderRepository: OrderRepository) {}
  
  getOrder(id: string) {
    return this.orderRepository.findById(id);
  }
}

// Question: Who owns the OrderRepository?
// 
// Answer: NOT OrderService!
// 
// Why? Because:
// 1. OrderRepository is needed by multiple services
//    (OrderService, OrderController, OrderAnalytics, etc.)
// 2. OrderRepository manages the connection to the database
// 3. The database is a shared resource
// 4. OrderService shouldn't control when the repository is created/destroyed
//
// Instead: A higher-level component (NestJS container) owns it
// This component creates one OrderRepository
// And gives it to everyone who needs it
```

Let's illustrate this with a real example:

```typescript
// Without DI: Each service creates its own repository (BAD)

export class OrderService {
  private orderRepository = new OrderRepository();  // Creates its own
}

export class OrderController {
  private orderRepository = new OrderRepository();  // Creates another one!
}

export class OrderAnalytics {
  private orderRepository = new OrderRepository();  // Creates yet another!
}

// Problem: Three separate OrderRepository instances!
// - Each has its own database connection
// - They're not synchronized
// - Wasting resources
// - If one modifies data, the others don't know
// - Tests are completely unreliable
```

```typescript
// With DI: One repository is created and shared (GOOD)

// First, the DI container creates it once:
const orderRepository = new OrderRepository();

// Then, it shares it with everyone who needs it:
const orderService = new OrderService(orderRepository);
const orderController = new OrderController(orderService);
const orderAnalytics = new OrderAnalytics(orderRepository);

// Result: Everyone uses the SAME OrderRepository instance
// - Single database connection
// - Everyone sees the same data
// - Efficient
// - Testable
```

### 2.3 Levels of Ownership

There are different levels of who "owns" a dependency. Understanding this is crucial:

#### **Level 1: Primitive Data (No Ownership)**

```typescript
// Primitives are passed in, not owned

export class OrderService {
  createOrder(userId: string, total: number) {
    //           ↑ primitive       ↑ primitive
    // These are passed in. OrderService doesn't "own" them.
    // They're just values the service uses.
  }
}

// Ownership: The caller owns userId and total
// They came from the request, or from the database, or from somewhere else
```

#### **Level 2: Object Dependencies (Received, Not Owned)**

```typescript
// Most services fall here

export class OrderService {
  constructor(
    private orderRepository: OrderRepository  // ← Received, not owned
  ) {}
}

// Ownership: Some higher component (the DI container) owns OrderRepository
// OrderService receives it, uses it, but doesn't create or destroy it
// When OrderService is destroyed, OrderRepository is not destroyed
//    (it might still be used by other services)
```

#### **Level 3: Data Owned by a Service (Created and Managed)**

```typescript
// Sometimes a service creates temporary data structures it owns

export class OrderService {
  private createdAt: Date;  // ← Service creates and owns this
  
  constructor(private orderRepository: OrderRepository) {
    this.createdAt = new Date();  // Service creates it
  }
  
  // When service is destroyed, createdAt is destroyed too
  // No one else uses createdAt, so service can manage it
}

// Ownership: OrderService creates and owns createdAt
// No one else needs it
// Service can safely manage its lifecycle
```

```typescript
// ❌ WRONG: Service owns something it shouldn't

export class OrderService {
  private database: Database;  // ← Service creates it
  
  constructor() {
    this.database = new Database({...});  // ← Service creates and manages
  }
}

// Problem: Multiple services might need the same database connection
// But each creates its own
// This defeats the purpose of DI
// The database is a RESOURCE that should be owned at a higher level
```

### 2.4 The DI Ownership Hierarchy

Understanding this hierarchy is key to good architecture:

```
┌─────────────────────────────────────────────────────┐
│                    DI Container                       │
│  (owns all services and infrastructure)              │
└─────────────────────────────────────────────────────┘
           ↓ creates and manages ↓
      ┌────────────────────┬────────────────────┐
      │                    │                    │
  ┌───▼────────┐   ┌───────▼──────┐   ┌────────▼───┐
  │   Database │   │   Logger     │   │   Config   │
  │ Connection │   │  (Singleton) │   │            │
  └────────────┘   └──────────────┘   └────────────┘
           ↓ shared with ↓
  ┌────────────────────────────────┐
  │  Services (OrderService,        │
  │  UserService, PaymentService)   │
  └────────────────────────────────┘
           ↓ uses ↓
  ┌────────────────────────────────┐
  │  Controllers (OrderController,  │
  │  UserController, ...)           │
  └────────────────────────────────┘
           ↓ provides ↓
  ┌────────────────────────────────┐
  │  HTTP Responses to Client       │
  └────────────────────────────────┘

Key insight:
- Container owns infrastructure (database, logger, config)
- Container owns services (they're singletons by default)
- Services are injected into controllers
- Controllers use services to handle requests
- Controllers are created fresh for each request
```

---

## Part 3: Dependency Injection Patterns

### 3.1 Constructor Injection (Best Practice)

```typescript
// ✅ BEST: Constructor Injection - Dependencies visible, immutable

export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly logger: Logger
  ) {}
  
  async createOrder(userId: string, items: Item[]) {
    this.logger.log('Creating order...');
    const order = await this.orderRepository.save({ userId, items });
    await this.paymentGateway.charge(...);
    return order;
  }
}

// Why this is best:
// ✅ Clear contract: You see exactly what this service needs
// ✅ Immutable: Dependencies can be readonly
// ✅ Circular dependency detection: DI container catches it
// ✅ IDE support: Autocomplete works
// ✅ Type safe: TypeScript validates all dependencies
// ✅ Easy to test: Just pass mocks in constructor
```

### 3.2 Property Injection (Avoid, but understand)

```typescript
// ⚠️ OKAY in some frameworks, but not recommended in NestJS

export class OrderService {
  @Inject(OrderRepository)
  private orderRepository: OrderRepository;  // Injected after construction
  
  createOrder(userId: string, items: Item[]) {
    // This might be undefined if injection fails!
    return this.orderRepository.save({ userId, items });
  }
}

// Why not great:
// ❌ Not clear from constructor what's needed
// ❌ Dependency might be undefined at runtime
// ❌ Harder to test (have to manually set properties)
// ❌ Order of property injection is not guaranteed
// ❌ TypeScript can't catch missing dependencies
```

### 3.3 Method Injection (Rare, specific use cases)

```typescript
// ⚠️ Very rare, only for specific scenarios

export class OrderService {
  createOrder(
    userId: string,
    items: Item[],
    repository: OrderRepository  // Injected as method parameter
  ) {
    return repository.save({ userId, items });
  }
}

// When to use:
// - Different calls to the same method need different dependencies
// - Example: Processing a job queue where each job brings its own database connection
// - Very uncommon in web services

// When NOT to use:
// - Normal business logic (use constructor injection)
// - It obscures the service's real dependencies
```

### 3.4 The NestJS Way: Decorator-Based Injection

```typescript
// This is how NestJS does it (under the hood):

@Injectable()  // ← This marks the service as injectable
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,    // Injected automatically
    private paymentGateway: PaymentGateway       // by the DI container
  ) {}
}

// NestJS uses TypeScript metadata and decorators to:
// 1. Detect that OrderService needs OrderRepository and PaymentGateway
// 2. Create instances of those dependencies
// 3. Pass them to the constructor
// 4. Handle the entire dependency graph
```

---

## Part 4: Why DI Makes Testing Trivial

This is the most important benefit. Let me show you exactly why:

### 4.1 Testing Without DI (Nightmare)

```typescript
// ❌ BAD: Service without DI
export class PaymentService {
  async processPayment(userId: string, amount: number) {
    const stripe = new Stripe({ apiKey: 'sk_live_...' });  // Creates Stripe
    const userDb = new Database({ url: 'postgresql://...' });  // Creates DB
    const user = await userDb.query('SELECT * FROM users WHERE id = ?', [userId]);
    const result = await stripe.charge(user.card, amount);
    return result;
  }
}

// Try to test it:
describe('PaymentService', () => {
  it('should process payment', async () => {
    const service = new PaymentService();
    const result = await service.processPayment('user123', 100);
    expect(result.success).toBe(true);
    
    // What happens?
    // ❌ Connects to real database
    // ❌ Calls real Stripe API
    // ❌ Takes 5+ seconds to run
    // ❌ Uses real money/credits
    // ❌ Fails if either service is down
    // ❌ Can't test error scenarios easily
    // ❌ Can't run tests offline
    // This is NOT a unit test, it's an integration test!
  });
});

// Trying to test error scenarios:
describe('PaymentService - Error Cases', () => {
  it('should handle declined card', async () => {
    const service = new PaymentService();
    const result = await service.processPayment('user_declined', 100);
    
    // How do you make this test a "declined card" scenario?
    // ❌ You can't, unless you:
    //   1. Have test credit card numbers (Stripe provides these)
    //   2. Use a test database
    //   3. Set up the test database with specific test data
    //   4. Hope nothing else modifies the test database while running
    // This is fragile and unreliable
  });
});
```

### 4.2 Testing With DI (Easy)

```typescript
// ✅ GOOD: Service with DI

export interface StripeGateway {
  charge(card: any, amount: number): Promise<ChargeResult>;
}

export interface UserRepository {
  findById(id: string): Promise<User>;
}

export class PaymentService {
  constructor(
    private stripe: StripeGateway,      // Receives interface, not concrete class
    private userDb: UserRepository      // Receives interface, not concrete class
  ) {}
  
  async processPayment(userId: string, amount: number) {
    const user = await this.userDb.findById(userId);
    const result = await this.stripe.charge(user.card, amount);
    return result;
  }
}

// Now testing is trivial:

describe('PaymentService', () => {
  let service: PaymentService;
  let mockStripe: StripeGateway;
  let mockUserDb: UserRepository;
  
  beforeEach(() => {
    // Create mock implementations
    mockStripe = {
      charge: jest.fn().mockResolvedValue({ success: true, transactionId: '123' })
    };
    
    mockUserDb = {
      findById: jest.fn().mockResolvedValue({ id: 'user123', card: '4242...' })
    };
    
    // Inject the mocks
    service = new PaymentService(mockStripe, mockUserDb);
  });
  
  // Test 1: Normal flow
  it('should process payment successfully', async () => {
    const result = await service.processPayment('user123', 100);
    
    expect(result.success).toBe(true);
    expect(mockStripe.charge).toHaveBeenCalledWith('4242...', 100);
    expect(mockUserDb.findById).toHaveBeenCalledWith('user123');
  });
  
  // Test 2: Error scenario - payment declined
  it('should handle declined card', async () => {
    // Change the mock behavior for this test
    mockStripe.charge.mockRejectedValue(new Error('Card declined'));
    
    // Now the test will use this rejection
    try {
      await service.processPayment('user123', 100);
      fail('Should have thrown');
    } catch (error) {
      expect(error.message).toBe('Card declined');
    }
  });
  
  // Test 3: Error scenario - user not found
  it('should handle user not found', async () => {
    mockUserDb.findById.mockRejectedValue(new Error('User not found'));
    
    try {
      await service.processPayment('nonexistent', 100);
      fail('Should have thrown');
    } catch (error) {
      expect(error.message).toBe('User not found');
    }
  });
  
  // Test 4: Verify correct call to dependencies
  it('should call Stripe with correct amount', async () => {
    await service.processPayment('user123', 500);
    
    expect(mockStripe.charge).toHaveBeenCalledWith('4242...', 500);
  });
});

// Benefits:
// ✅ Tests run in milliseconds (no I/O)
// ✅ No need for real database, Stripe, etc.
// ✅ Can test any scenario (success, decline, errors)
// ✅ Tests are deterministic (always same result)
// ✅ Can run offline
// ✅ No risk of breaking real systems
// ✅ Can verify service called dependencies correctly
// ✅ Easy to change mock behavior between tests
```

### 4.3 The Testing Principle: Isolation

DI enables **test isolation**:

```
Without DI:          With DI:
┌──────────────┐    ┌──────────────┐
│   Service    │    │   Service    │
│              │    │              │
├──────────────┤    ├──────────────┤
│  ❌ Database │    │  ✅ Mock DB   │
│  ❌ API      │    │  ✅ Mock API  │
│  ❌ Cache    │    │  ✅ Mock      │
│  ❌ Logger   │    │     Cache     │
│  ❌ ...      │    │  ✅ Mock      │
│     (all     │    │     Logger    │
│    real)     │    │  ✅ ...       │
└──────────────┘    └──────────────┘

You test:           You test:
- Service logic +   - ONLY service logic
- Database          - Everything else is
- API               controlled by mocks
- Cache
- Logger
- Network
- ...

Not a unit test!    Pure unit test!
```

### 4.4 Why AI Loves DI Code

When you write code with DI:

```typescript
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private paymentGateway: PaymentGateway
  ) {}
  
  async createOrder(userId: string, items: Item[]) {
    // Logic here
  }
}
```

AI can **automatically generate unit tests** because:

```
1. Clear dependencies
   → AI knows what to mock (orderRepository, paymentGateway)

2. Constructor injection
   → AI knows how to instantiate the service

3. No hidden dependencies
   → AI won't miss anything when writing tests

4. Testable architecture
   → AI can generate meaningful tests

Example: Ask AI to "write unit tests for OrderService"
AI will generate:
✅ Tests for each method
✅ Proper mocks for all dependencies
✅ Test cases for success and failure paths
✅ Verification that dependencies are called correctly

Without DI, AI would generate:
❌ Integration tests (calling real database)
❌ Tests that can't run (missing credentials)
❌ Flaky tests (depending on external services)
❌ Tests that are slow (waiting for I/O)
```

---

## Part 5: Ownership Patterns & Data Flow

### 5.1 The Ownership Question in Different Scenarios

**Scenario 1: Database Connections**

```typescript
// Question: Who owns the database connection?

❌ WRONG: Each service creates it
export class UserService {
  private db = new Database();  // Service owns database
}
export class OrderService {
  private db = new Database();  // Another database!
}

✅ RIGHT: The container owns it
const db = new Database();  // Created once at app startup
const userService = new UserService(db);
const orderService = new OrderService(db);

Ownership: Container
Why: Database is a shared resource. Multiple services need it.
    Should be created once, shared by all.
    Service doesn't manage its lifecycle.
```

**Scenario 2: External API Clients (Stripe, SendGrid, etc.)**

```typescript
// Question: Who owns the API client?

❌ WRONG: Each service creates it
export class PaymentService {
  private stripe = new Stripe({ apiKey: '...' });
}
export class NotificationService {
  private stripe = new Stripe({ apiKey: '...' });  // Different instance!
}

✅ RIGHT: The container owns it (usually as a "gateway" interface)
const stripeGateway = new StripePaymentGateway({ apiKey: '...' });
const paymentService = new PaymentService(stripeGateway);

// Other modules can also use it:
const refundService = new RefundService(stripeGateway);

Ownership: Container
Why: External APIs are expensive to create. Single instance, shared.
    Credentials are app-level concerns.
    Services don't manage the API client.
```

**Scenario 3: Request-Specific Data (User ID from JWT, Request Object, etc.)**

```typescript
// Question: Who owns request-specific data?

❌ WRONG: Service looks it up globally
export class OrderService {
  createOrder(items: Item[]) {
    const userId = CurrentContext.get('userId');  // Global lookup! Bad!
    // ...
  }
}

✅ RIGHT: Controller passes it to the service
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}
  
  @Post()
  @UseGuards(AuthGuard('jwt'))
  createOrder(@Body() dto: CreateOrderDto, @Req() req: any) {
    // Extract from request
    const userId = req.user.id;
    
    // Pass to service
    return this.orderService.createOrder(userId, dto.items);
  }
}

export class OrderService {
  createOrder(userId: string, items: Item[]) {
    // Service receives it, doesn't look it up
    // ...
  }
}

Ownership: Request context
Why: This data is specific to a single request.
    Service shouldn't know HOW to get it.
    Controller knows about requests, so controller extracts and passes it.
```

**Scenario 4: Calculated/Derived Data (Values computed from other data)**

```typescript
// Question: Who owns calculated data?

❌ WRONG: Create a service just to store it
const total = 100;
const tax = 10;
const withTax = 110;

// Creating three separate services for these? No.

✅ RIGHT: Service calculates it when needed
export class OrderService {
  calculateTotal(items: Item[]): OrderTotal {
    const subtotal = items.reduce((sum, item) => sum + item.price, 0);
    const tax = subtotal * 0.1;
    const total = subtotal + tax;
    
    return { subtotal, tax, total };
  }
}

// Or if it's needed in multiple places:
export class PricingService {  // New service for pricing logic
  calculateTotal(items: Item[]): OrderTotal {
    // Calculation logic
  }
}

// Now both OrderService and RefundService can use it:
export class OrderService {
  constructor(private pricingService: PricingService) {}
  
  createOrder(items: Item[]) {
    const total = this.pricingService.calculateTotal(items);
    // ...
  }
}

export class RefundService {
  constructor(private pricingService: PricingService) {}
  
  refundOrder(order: Order) {
    const total = this.pricingService.calculateTotal(order.items);
    // ...
  }
}

Ownership: Service that calculated it
Why: Calculated data doesn't "belong" anywhere. It's computed on demand.
    If multiple services need the same calculation, create a shared service.
```

### 5.2 The Ownership Decision Tree

Use this to decide who owns each piece of data:

```
Question: Should this data be injected?

1. Is it a resource that multiple services need?
   → YES: Create once, inject to everyone
      Examples: Database, Logger, Config, API Clients
   → NO: Go to question 2

2. Is this data needed throughout the service's lifetime?
   → YES: Inject it in the constructor
      Examples: Repository, External service client, Logger
   → NO: Go to question 3

3. Is this data request-specific?
   → YES: Pass it as a method parameter
      Examples: userId from JWT, request object, user input
   → NO: Go to question 4

4. Is this data calculated from other data?
   → YES: Calculate it in the method, OR
      Create a utility service if multiple services need it
      Examples: Total price, tax calculation, formatted date
   → NO: Go to question 5

5. Is this data something only this service needs?
   → YES: Create it in the service, store as instance variable
      Examples: Internal state, caches, temporary data
   → NO: You've found a design issue. Refactor.

Result: You've placed the data correctly in the ownership hierarchy.
```

### 5.3 Common Ownership Mistakes

**Mistake 1: Service owns something the container should own**

```typescript
// ❌ BAD: Service creates its own logger
export class OrderService {
  private logger = new Logger('OrderService');  // Wrong ownership
}

// ✅ GOOD: Service receives logger from container
export class OrderService {
  constructor(private logger: Logger) {}  // Container owns logger
}

// Why: Logger is configured at app level. Should be single instance.
// Multiple services logging to same logger. Service doesn't manage lifecycle.
```

**Mistake 2: Forgetting request-specific data should be passed**

```typescript
// ❌ BAD: Service looks up user from some global context
export class OrderService {
  createOrder(items: Item[]) {
    const user = UserContext.current();  // Global lookup!
    // ...
  }
}

// ✅ GOOD: Controller passes user
export class OrderController {
  @Post()
  createOrder(@Req() req: any, @Body() dto: CreateOrderDto) {
    return this.orderService.createOrder(req.user.id, dto.items);
  }
}

export class OrderService {
  createOrder(userId: string, items: Item[]) {  // Receive it, don't look it up
    // ...
  }
}

// Why: userId is request-specific. Service doesn't own it. Controller does.
```

**Mistake 3: Service owns data that should be request-level**

```typescript
// ❌ BAD: Service stores request data as instance variable
export class OrderService {
  private currentUserId: string;  // Changes per request!
  
  setCurrentUser(userId: string) {
    this.currentUserId = userId;  // Thread-unsafe in Node.js!
  }
  
  createOrder(items: Item[]) {
    // Uses currentUserId
  }
}

// In Node.js, the same OrderService instance is used for all requests!
// So this.currentUserId gets overwritten by different requests.
// Leading to race conditions.

// ✅ GOOD: Pass request data as parameters
export class OrderService {
  createOrder(userId: string, items: Item[]) {
    // Each request gets its own parameters
  }
}
```

**Mistake 4: Circular ownership (A owns B, B owns A)**

```typescript
// ❌ BAD: Circular dependency
export class UserService {
  constructor(private orderService: OrderService) {}  // ← Depends on Order
}

export class OrderService {
  constructor(private userService: UserService) {}    // ← Depends on User
}

// NestJS will throw an error when starting up.
// Symptom: Services depending on each other to function.
// Cause: Both think they "own" or control the other's responsibility.

// ✅ GOOD: Break the cycle with events
export class UserService {
  constructor(
    private events: EventEmitter,
    private userRepository: UserRepository
  ) {}
  
  registerUser(data: any) {
    const user = this.userRepository.save(data);
    this.events.emit('user.registered', { userId: user.id });
    return user;
  }
}

export class OrderService {
  constructor(
    private events: EventEmitter,
    private orderRepository: OrderRepository
  ) {}
  
  constructor() {
    // Listen for events, don't depend on UserService
    this.events.on('user.registered', (data) => {
      this.initializeUserOrders(data.userId);
    });
  }
}
```

---

## Part 6: Dependency Injection in NestJS (Practical Implementation)

### 6.1 How NestJS Implements DI

NestJS uses **TypeScript decorators and metadata** to implement DI:

```typescript
// Step 1: Mark service as injectable
@Injectable()
export class OrderRepository {
  constructor(private db: Database) {}
  
  save(order: any) {
    // ...
  }
}

// Step 2: Another service that depends on the first
@Injectable()
export class OrderService {
  constructor(private orderRepository: OrderRepository) {}
  
  createOrder(userId: string, items: Item[]) {
    return this.orderRepository.save({ userId, items });
  }
}

// Step 3: Declare them in a module
@Module({
  providers: [OrderRepository, OrderService],  // Declare here
  exports: [OrderService],                      // Export what others need
})
export class OrderModule {}

// Step 4: NestJS does the magic
// When you import OrderModule into another module:
@Module({
  imports: [OrderModule],  // NestJS registers OrderRepository and OrderService
  providers: [SomeService],
})
export class AnotherModule {}

// When SomeService needs OrderService:
@Injectable()
export class SomeService {
  constructor(private orderService: OrderService) {}  // ← NestJS provides it
}

// What NestJS does behind the scenes:
// 1. Reads the constructor parameters using TypeScript metadata
// 2. Looks up those types in the DI container
// 3. Creates instances if needed (or returns singleton)
// 4. Passes them to the constructor
```

### 6.2 NestJS Provider Types

NestJS offers different ways to provide dependencies:

#### **Type 1: Class Provider (Most Common)**

```typescript
@Module({
  providers: [OrderService],
  // Shorthand for:
  // providers: [
  //   { provide: OrderService, useClass: OrderService }
  // ]
})
export class OrderModule {}

// NestJS will:
// 1. Create an instance of OrderService
// 2. Inject it wherever needed
```

#### **Type 2: Value Provider (Constants, Configuration)**

```typescript
const CONFIG = {
  database_url: 'postgresql://...',
  stripe_key: 'sk_live_...',
};

@Module({
  providers: [
    { provide: 'CONFIG', useValue: CONFIG },  // Provide the value
    OrderService,
  ],
})
export class OrderModule {}

// Use it in a service:
@Injectable()
export class OrderService {
  constructor(
    @Inject('CONFIG') private config: any  // Inject with token
  ) {}
}

// Why use this:
// - For configuration objects
// - For constants
// - When you can't use a class
```

#### **Type 3: Factory Provider (Dynamic Creation)**

```typescript
const databaseFactory = {
  provide: 'DATABASE',
  useFactory: async (configService: ConfigService) => {
    const config = configService.get('database');
    const db = new Database(config);
    await db.connect();
    return db;
  },
  inject: [ConfigService],  // Tell NestJS what to inject
};

@Module({
  providers: [databaseFactory, ConfigService],
})
export class DatabaseModule {}

// Use it:
@Injectable()
export class UserRepository {
  constructor(@Inject('DATABASE') private db: Database) {}
}

// Why use this:
// - Need to run code before providing the dependency
// - Dependencies have complex initialization
// - Need to choose between multiple implementations
```

#### **Type 4: Class Alias (Strategy Pattern)**

```typescript
interface PaymentGateway {
  charge(amount: number): Promise<void>;
}

class StripeGateway implements PaymentGateway {
  async charge(amount: number) { /* ... */ }
}

class PayPalGateway implements PaymentGateway {
  async charge(amount: number) { /* ... */ }
}

// Production: use Stripe
const productionProvider = {
  provide: 'PaymentGateway',
  useClass: StripeGateway,
};

// Testing: use PayPal
const testingProvider = {
  provide: 'PaymentGateway',
  useClass: PayPalGateway,
};

// Use it:
@Module({
  providers: [
    process.env.NODE_ENV === 'production' 
      ? productionProvider 
      : testingProvider,
  ],
})
export class PaymentModule {}
```

### 6.3 Scope of Providers

NestJS providers have different lifetimes:

#### **Singleton (Default)**

```typescript
@Injectable()  // ← By default, singleton
export class OrderService {}

// What happens:
// 1. First request to OrderService: Create instance
// 2. All subsequent requests: Reuse the same instance
// 3. Service lives for entire app lifetime

// Good for:
// - Services (stateless, shared)
// - Repositories (connection pooling)
// - Loggers
// - Most services
```

#### **Transient**

```typescript
@Injectable({ scope: Scope.TRANSIENT })
export class OrderService {}

// What happens:
// 1. Every time something injects OrderService: Create new instance
// 2. Each injection gets its own instance
// 3. Service doesn't persist

// Good for:
// - Services that need per-request state
// - Rare in practice (usually controller handles per-request)
```

#### **Request**

```typescript
@Injectable({ scope: Scope.REQUEST })
export class OrderService {}

// What happens:
// 1. Each HTTP request: Create instance
// 2. All services in that request share the same instance
// 3. New instance for new request

// Good for:
// - Services that need request context
// - Example: storing user from JWT
```

---

## Part 7: Testing Patterns with DI

### 7.1 Unit Test Structure with Mocks

```typescript
describe('OrderService', () => {
  let service: OrderService;
  let mockRepository: MockOrderRepository;
  let mockPayment: MockPaymentGateway;
  
  beforeEach(() => {
    // Create mocks implementing the interface
    mockRepository = {
      save: jest.fn(),
      findById: jest.fn(),
    };
    
    mockPayment = {
      charge: jest.fn(),
    };
    
    // Inject mocks
    service = new OrderService(mockRepository, mockPayment);
  });
  
  describe('createOrder', () => {
    it('should save order and process payment', async () => {
      // Arrange
      mockRepository.save.mockResolvedValue({ id: '123', total: 100 });
      mockPayment.charge.mockResolvedValue({ transactionId: 'tx_123' });
      
      // Act
      const result = await service.createOrder('user123', []);
      
      // Assert
      expect(result.id).toBe('123');
      expect(mockRepository.save).toHaveBeenCalled();
      expect(mockPayment.charge).toHaveBeenCalledWith(100);
    });
    
    it('should handle payment failure', async () => {
      // Arrange
      mockPayment.charge.mockRejectedValue(new Error('Card declined'));
      
      // Act & Assert
      await expect(
        service.createOrder('user123', [])
      ).rejects.toThrow('Card declined');
    });
  });
});
```

### 7.2 Integration Test with NestJS Test Module

```typescript
describe('OrderService Integration', () => {
  let app: INestApplication;
  let orderService: OrderService;
  let orderRepository: OrderRepository;
  
  beforeAll(async () => {
    // Create a test module with real implementations
    const moduleFixture = await Test.createTestingModule({
      imports: [OrderModule, DatabaseModule],  // Real modules
    })
      .overrideProvider(OrderRepository)
      .useValue(mockRepository)  // Override with mock
      .compile();
    
    app = moduleFixture.createNestApplication();
    await app.init();
    
    orderService = moduleFixture.get<OrderService>(OrderService);
    orderRepository = moduleFixture.get<OrderRepository>(OrderRepository);
  });
  
  afterAll(async () => {
    await app.close();
  });
  
  it('should process order with all dependencies', async () => {
    const result = await orderService.createOrder('user123', []);
    expect(result).toBeDefined();
  });
});
```

---

## Part 8: Directing AI to Write DI Code

### 8.1 Good Prompts (Lead to Testable Code)

```
✅ GOOD: Clear about dependencies and testing

"Create an OrderService that:
1. Depends on OrderRepository and PaymentGateway (both injected via constructor)
2. Should have methods: createOrder(userId, items), getOrder(id)
3. Constructor should be private for dependency injection
4. Imports both dependencies as interfaces, not concrete classes
5. Should be easily testable with mocks"
```

```
✅ GOOD: Explicit about what to export

"Create an OrderModule that:
1. Provides OrderService (exported)
2. Provides OrderRepository (NOT exported, internal)
3. Only exports OrderService
4. When another module imports this, it should only access OrderService"
```

```
✅ GOOD: Focuses on domain responsibility

"Create a PricingService that:
1. Handles all price calculations (subtotal, tax, total)
2. Is stateless (no internal state)
3. Can be injected into OrderService and RefundService
4. Has clear, testable methods for each calculation"
```

### 8.2 Bad Prompts (Lead to Untestable Code)

```
❌ BAD: Vague about dependencies

"Create an OrderService with all the order stuff"
→ AI will create dependencies everywhere
→ Will be hard to test
→ Coupling between services
```

```
❌ BAD: Doesn't mention testing

"Create a PaymentService"
→ AI might hardcode external dependencies
→ Will create its own Stripe client
→ Won't be testable
```

```
❌ BAD: Doesn't specify what's internal vs exported

"Create a UserModule"
→ AI might export everything
→ Other modules will depend on internals
→ Architecture becomes spaghetti
```

### 8.3 Advanced Prompt: "Write Tests for X"

When you ask AI to write tests, it will only generate good tests if the code has DI:

```
"Write unit tests for OrderService. 
The service has:
- constructor(orderRepository: OrderRepository, paymentGateway: PaymentGateway)
- createOrder(userId: string, items: Item[]): Promise<Order>
- getOrder(id: string): Promise<Order>

Test cases:
1. createOrder should save order via repository
2. createOrder should charge payment via gateway
3. If payment fails, order should not be saved
4. getOrder should return correct order

Use Jest and mock the dependencies."
```

Because OrderService has DI, AI can easily:
- Create mocks for OrderRepository and PaymentGateway
- Instantiate OrderService with mocks
- Write meaningful test cases
- Verify dependencies were called correctly

---

## Part 9: Ownership Checklist

Use this when designing services:

```
Before writing a service, answer these:

□ What does this service depend on?
  List: ______________________________

□ For each dependency, ask: Who owns it?
  - Container (shared resource)?
  - Service itself (temporary data)?
  - Caller (request-specific)?
  
□ Will the service be testable?
  - Can I mock all dependencies?
  - Can I test without external services?

□ Is ownership clear?
  - Constructor shows all required dependencies?
  - No hidden dependencies?
  - No global lookups?

□ Can multiple requests use this service safely?
  - No per-request state stored in service?
  - Thread-safe (Node.js single thread, but event loop)?
  - Safe for concurrent use?

□ Is the public interface clear?
  - What methods does this service expose?
  - Do they take only the parameters they need?
  - Do they return what callers expect?
```

---

## Part 10: Common DI Mistakes & How to Avoid Them

### Mistake 1: Service Creates Its Own Dependencies

```typescript
// ❌ BAD
@Injectable()
export class UserService {
  private userRepository = new UserRepository(
    new Database('postgresql://...')  // ← Service creates database!
  );
}

// ✅ GOOD
@Injectable()
export class UserService {
  constructor(private userRepository: UserRepository) {}
  // Repository is injected, not created
}
```

### Mistake 2: Circular Dependencies

```typescript
// ❌ BAD
@Injectable()
export class UserService {
  constructor(private orderService: OrderService) {}  // ← Depends on Order
}

@Injectable()
export class OrderService {
  constructor(private userService: UserService) {}    // ← Depends on User
}
// This will crash NestJS at startup

// ✅ GOOD: Break cycle with events
// UserService emits 'user.created'
// OrderService listens and responds
// No circular dependency
```

### Mistake 3: Service Depends on Everything

```typescript
// ❌ BAD
@Injectable()
export class UserService {
  constructor(
    private userRepository: UserRepository,
    private orderService: OrderService,
    private paymentService: PaymentService,
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService,
    // ... 5 more services
  ) {}
}
// This service knows too much. It's doing too much.

// ✅ GOOD: Minimal dependencies
@Injectable()
export class UserService {
  constructor(
    private userRepository: UserRepository,
    private passwordHasher: PasswordHasher
  ) {}
  // Only what it needs
  // Everything else handles itself
}
```

### Mistake 4: Not Using Constructor Injection

```typescript
// ❌ BAD: Property injection
@Injectable()
export class UserService {
  @Inject(UserRepository)
  private userRepository: UserRepository;  // ← Might be undefined
  
  getUser(id: string) {
    return this.userRepository.findById(id);  // ← Could throw!
  }
}

// ✅ GOOD: Constructor injection
@Injectable()
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  getUser(id: string) {
    return this.userRepository.findById(id);  // ← Guaranteed to exist
  }
}
```

### Mistake 5: Mixing Concerns (Storage, Business Logic, Infrastructure)

```typescript
// ❌ BAD: Everything in one service
@Injectable()
export class OrderService {
  // Storage concern
  async save(order: any) {
    const connection = await Database.connect();
    const result = await connection.query('INSERT...');
    await connection.close();
    return result;
  }
  
  // Business logic
  calculateTotal(items: Item[]) {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
  
  // Infrastructure concern
  logOrder(order: Order) {
    console.log(JSON.stringify(order));
  }
}

// ✅ GOOD: Separated concerns
@Injectable()
export class OrderRepository {  // Storage
  constructor(private db: Database) {}
  
  save(order: any) {
    return this.db.query('INSERT...', order);
  }
}

@Injectable()
export class OrderService {  // Business logic
  constructor(private orderRepository: OrderRepository) {}
  
  async createOrder(userId: string, items: Item[]) {
    const total = this.calculateTotal(items);
    return this.orderRepository.save({ userId, items, total });
  }
  
  private calculateTotal(items: Item[]) {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
}

@Injectable()
export class OrderLogger {  // Infrastructure
  constructor(private logger: Logger) {}
  
  logOrder(order: Order) {
    this.logger.log(JSON.stringify(order));
  }
}
```

---

## Part 11: The DI Ownership Hierarchy (Summary)

```
┌─────────────────────────────────────────────────────────┐
│                 APPLICATION CONTAINER                    │
│         (NestJS, owns all singleton instances)           │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   [Infrastructure]  [Services]      [Repositories]
   - Logger          - UserService   - UserRepository
   - Database        - OrderService  - OrderRepository
   - Config          - PaymentSvc    - ProductRepository
   - Cache           - ...           - ...
   
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    [Controllers]   [Request Handlers]  [Guards/Middleware]
    - HTTP         - Process requests  - Authorization
    - WebSocket    - Extract params    - Validation
    - gRPC         - Call services     - Logging

Key Principle:
- Upper layers own lower layers
- Lower layers never depend on upper layers
- Data flows down (injection)
- Results flow up (HTTP responses)
- Communication across same level through events or service calls
```

---

## Part 12: Quick Reference for Directing AI

### When You Want Testable Code

```
Tell AI:

"Create [Service] with:
1. Constructor dependency injection
2. All dependencies as parameters
3. No hardcoded connections/APIs
4. Constructor signature:
   constructor(dep1: Type1, dep2: Type2)
5. Mark with @Injectable()

This service will be tested by injecting mocks,
so make sure nothing is created inside the service."
```

### When You Want Multiple Implementations

```
Tell AI:

"Create an interface [Name] with methods [list].
Then create two implementations:
- [Name]Dev for development
- [Name]Prod for production

In the module, use a factory provider to choose
based on environment variable."
```

### When You Want to Review AI Code for DI

```
Check list:
□ Does constructor inject all dependencies?
□ Are there any 'new' keywords inside methods? (Should only be in constructors)
□ Are dependencies immutable? (readonly keyword?)
□ Can I easily pass mocks in tests?
□ Are there any hardcoded values/credentials?
□ Does the service do only one thing?
```

---

## Summary: DI in One Sentence

**A class should receive its dependencies rather than create them, making the code testable, flexible, and easy to understand.**

The ownership question "Who owns this data?" is the key to getting it right:
- Shared resources → Container owns
- Service needs for its lifetime → Service receives
- Request-specific → Controller/Caller passes
- Temporary/calculated → Service creates

Master this, and you'll be able to architect systems and direct AI to build them cleanly.
