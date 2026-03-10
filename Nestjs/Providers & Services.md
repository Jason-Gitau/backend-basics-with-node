# Providers & Services in NestJS: Complete Deep Dive

## What This Guide Does

This guide teaches you to master **Providers & Services** so you can:
- Understand the difference between Providers and Services
- Design clean service interfaces (method signatures) that are easy to test
- Know exactly what should go in a Service vs Controller
- Direct AI to implement service logic by giving it a method signature
- Review AI-generated services for proper abstraction and testability
- See how Providers + DI + Modules work together as one system

---

## Part 0: How This Connects to What You Already Know

Before we dive in, let's see how Providers, Services, DI, and Modules form one cohesive system:

### The Architecture Stack

```
┌──────────────────────────────────────────┐
│         MODULES (Part 1)                 │
│  ├─ Define boundaries                    │
│  ├─ Group related logic                  │
│  └─ Declare what's public (exports)      │
└──────────────────────────────────────────┘
              │
┌──────────────────────────────────────────┐
│      PROVIDERS (Part 3, this guide)      │
│  ├─ Registered in modules                │
│  ├─ Services are a type of provider      │
│  ├─ Can be classes, values, factories    │
│  └─ Managed by DI container              │
└──────────────────────────────────────────┘
              │
┌──────────────────────────────────────────┐
│    DEPENDENCY INJECTION (Part 2)         │
│  ├─ Injects providers into consumers     │
│  ├─ Makes everything testable            │
│  ├─ Manages provider lifecycle           │
│  └─ Resolves the dependency graph        │
└──────────────────────────────────────────┘
              │
┌──────────────────────────────────────────┐
│      SERVICES (Part 3, this guide)       │
│  ├─ Implement business logic             │
│  ├─ Are providers (most common type)     │
│  ├─ Injected into controllers            │
│  └─ Injected into other services         │
└──────────────────────────────────────────┘

Flow:
1. Define module boundaries (MODULES)
2. Create providers that implement logic (SERVICES)
3. Register providers in modules (MODULES + PROVIDERS)
4. DI injects them where needed (DEPENDENCY INJECTION)
5. Controllers use services via injection (SERVICES → CONTROLLERS)
```

**Example:** How they work together

```typescript
// Step 1: Define a service (Provider)
@Injectable()
export class OrderService {
  constructor(private orderRepository: OrderRepository) {}
  
  createOrder(userId: string, items: Item[]) {
    return this.orderRepository.save({ userId, items });
  }
}

// Step 2: Register in module with other providers
@Module({
  providers: [OrderService, OrderRepository],  // ← Providers registered here
  exports: [OrderService],                      // ← Public API
})
export class OrderModule {}

// Step 3: DI injects it where needed
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}  // ← Injected!
  
  @Post()
  create(@Body() dto: CreateOrderDto) {
    return this.orderService.createOrder(dto.userId, dto.items);
  }
}

// Step 4: Entire flow
// - OrderModule declares OrderService as provider
// - DI container instantiates OrderService
// - OrderController receives OrderService via constructor
// - Controller calls service method
// - Service uses repository (also injected)
// - Everything is testable because everything is injected
```

---

## Part 1: Providers vs Services (What's the Difference?)

### 1.1 The Confusing Terminology

In NestJS, **Service** and **Provider** are often used interchangeably, but they're technically different concepts:

```
PROVIDER: An umbrella term for anything that can be injected
├─ Services (classes with business logic) ← Most common
├─ Values (constants, config objects)
├─ Factories (functions that create things)
└─ Other injectable things (guards, filters, etc.)

SERVICE: A specific type of provider
└─ A class marked with @Injectable()
    that implements business logic
```

### 1.2 Visualizing It

```
"Provider" is the general concept
         │
    ┌────┴────┬──────────┬─────────┐
    │         │          │         │
  Service   Value    Factory     Custom
  ├─────┐   ├──────┐  ├──────┐
  │Class│   │Const│  │Function
  │@Inj │   │Config    
  
In 90% of code, "Provider" = "Service"
But technically, Service is one type of Provider
```

### 1.3 Examples of Each Type

#### **Type 1: Service Provider (Class)**

```typescript
// The most common provider

@Injectable()  // ← Marks this as injectable
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  getUser(id: string) {
    return this.userRepository.findById(id);
  }
  
  createUser(data: CreateUserDto) {
    return this.userRepository.save(data);
  }
}

// Register it:
@Module({
  providers: [UserService, UserRepository],
})
export class UserModule {}

// Use it:
@Controller('users')
export class UserController {
  constructor(private userService: UserService) {}  // ← Service injected
}
```

#### **Type 2: Value Provider (Constants)**

```typescript
// Use when you want to inject configuration or constants

const DATABASE_CONFIG = {
  host: 'localhost',
  port: 5432,
  database: 'myapp',
};

@Module({
  providers: [
    { provide: 'DB_CONFIG', useValue: DATABASE_CONFIG },  // ← Value provider
    UserService,
  ],
})
export class UserModule {}

// Use it:
@Injectable()
export class UserService {
  constructor(
    @Inject('DB_CONFIG') private dbConfig: any  // ← Inject the value
  ) {}
  
  getConnectionString() {
    return `postgresql://${this.dbConfig.host}:${this.dbConfig.port}...`;
  }
}
```

#### **Type 3: Factory Provider (Dynamic Creation)**

```typescript
// Use when you need to create a provider dynamically (e.g., based on env)

const databaseFactory = {
  provide: 'DATABASE',
  useFactory: (configService: ConfigService) => {
    const config = configService.get('database');
    
    if (process.env.NODE_ENV === 'production') {
      return new PostgresDatabase(config);
    } else {
      return new InMemoryDatabase();  // Use in-memory for testing
    }
  },
  inject: [ConfigService],  // ← Tell DI to inject ConfigService
};

@Module({
  providers: [databaseFactory, ConfigService],
})
export class DatabaseModule {}

// Now every service gets the correct database implementation
// without having to know about the environment
```

#### **Type 4: Class Provider with Alias (Strategy Pattern)**

```typescript
// Use when you want to provide different implementations

interface PaymentGateway {
  charge(amount: number): Promise<ChargeResult>;
}

export class StripeGateway implements PaymentGateway {
  async charge(amount: number) { /* Stripe implementation */ }
}

export class PayPalGateway implements PaymentGateway {
  async charge(amount: number) { /* PayPal implementation */ }
}

@Module({
  providers: [
    {
      provide: 'PaymentGateway',
      useClass: process.env.PAYMENT_PROVIDER === 'paypal' 
        ? PayPalGateway 
        : StripeGateway,
    },
  ],
})
export class PaymentModule {}

// Use it:
@Injectable()
export class OrderService {
  constructor(
    @Inject('PaymentGateway') private gateway: PaymentGateway
  ) {}
  
  async processOrder(amount: number) {
    return this.gateway.charge(amount);  // Works with any implementation
  }
}
```

---

## Part 2: Services - Where Business Logic Lives

### 2.1 The Rule: Logic Goes in Services, Not Controllers

This is fundamental. Controllers should be thin. Services should be thick.

#### **❌ BAD: Logic in Controller**

```typescript
@Controller('orders')
export class OrderController {
  constructor(
    private db: Database,
    private stripe: StripeClient,
    private email: EmailService,
    private logger: Logger
  ) {}
  
  @Post()
  async createOrder(@Body() dto: CreateOrderDto) {
    // ❌ All logic is here!
    
    const items = dto.items;
    let total = 0;
    
    for (const item of items) {
      const product = await this.db.query(
        'SELECT * FROM products WHERE id = ?',
        [item.productId]
      );
      if (!product) throw new Error('Product not found');
      if (product.stock < item.quantity) throw new Error('Out of stock');
      
      total += product.price * item.quantity;
    }
    
    const tax = total * 0.1;
    const finalTotal = total + tax;
    
    const order = {
      userId: dto.userId,
      items: dto.items,
      total: finalTotal,
      createdAt: new Date(),
      status: 'pending',
    };
    
    await this.db.query(
      'INSERT INTO orders (user_id, items, total, created_at, status) VALUES (?, ?, ?, ?, ?)',
      [order.userId, JSON.stringify(order.items), order.total, order.createdAt, order.status]
    );
    
    const paymentResult = await this.stripe.charge(
      dto.paymentMethodId,
      finalTotal
    );
    
    if (!paymentResult.success) {
      await this.db.query('UPDATE orders SET status = ? WHERE id = ?', 
        ['failed', order.id]);
      throw new Error('Payment failed');
    }
    
    await this.email.send(dto.email, 'Order confirmed', {
      orderId: order.id,
      total: finalTotal,
    });
    
    this.logger.log(`Order created: ${order.id}`);
    
    return order;
  }
}

// Problems:
// ❌ Controller is 80 lines
// ❌ Impossible to test (can't isolate logic)
// ❌ Hard to reuse (logic is locked in controller)
// ❌ Hard to change (touched by many issues)
// ❌ Can't test payment failure, stock checks separately
// ❌ Can't test what happens if email fails
```

#### **✅ GOOD: Logic in Service**

```typescript
// Service 1: Handle product checks and pricing
@Injectable()
export class ProductService {
  constructor(private productRepository: ProductRepository) {}
  
  async validateAndPrice(items: Item[]) {
    const priced = [];
    
    for (const item of items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) throw new NotFoundException('Product not found');
      if (product.stock < item.quantity) throw new BadRequestException('Out of stock');
      
      priced.push({
        ...item,
        price: product.price,
        lineTotal: product.price * item.quantity,
      });
    }
    
    return priced;
  }
}

// Service 2: Handle pricing calculations
@Injectable()
export class PricingService {
  calculateTotal(items: any[]) {
    const subtotal = items.reduce((sum, item) => sum + item.lineTotal, 0);
    const tax = subtotal * 0.1;
    return { subtotal, tax, total: subtotal + tax };
  }
}

// Service 3: Handle order creation and payment
@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private productService: ProductService,
    private pricingService: PricingService,
    private paymentGateway: PaymentGateway,
    private notificationService: NotificationService
  ) {}
  
  async createOrder(userId: string, items: Item[], paymentMethodId: string) {
    // Validate products and get pricing
    const pricedItems = await this.productService.validateAndPrice(items);
    
    // Calculate total
    const pricing = this.pricingService.calculateTotal(pricedItems);
    
    // Create order record
    const order = await this.orderRepository.save({
      userId,
      items: pricedItems,
      ...pricing,
      status: 'pending',
    });
    
    // Process payment
    const paymentResult = await this.paymentGateway.charge(
      paymentMethodId,
      pricing.total
    );
    
    if (!paymentResult.success) {
      await this.orderRepository.updateStatus(order.id, 'failed');
      throw new PaymentFailedException('Payment was declined');
    }
    
    // Update order status
    await this.orderRepository.updateStatus(order.id, 'confirmed');
    
    // Notify customer
    await this.notificationService.sendOrderConfirmation(userId, order);
    
    return order;
  }
}

// Controller: Thin and clear
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}
  
  @Post()
  async createOrder(@Body() dto: CreateOrderDto, @Req() req: any) {
    return this.orderService.createOrder(
      req.user.id,
      dto.items,
      dto.paymentMethodId
    );
  }
}

// Benefits:
// ✅ Controller is 5 lines (clear intent)
// ✅ Each service is testable in isolation
// ✅ Logic can be reused (OrderService works anywhere)
// ✅ Easy to change (modify service, not controller)
// ✅ Can test payment failure, stock issues separately
// ✅ Each service does one thing well
```

### 2.2 The Line: What Goes in Controllers vs Services

```
┌─────────────────────────────────────────────────────────┐
│ CONTROLLER (HTTP Request Handler)                       │
├─────────────────────────────────────────────────────────┤
│ ✅ DO:                                                  │
│ - Extract parameters from HTTP request                 │
│ - Validate HTTP request format (guards, pipes)         │
│ - Extract user from JWT/session                        │
│ - Call service method with extracted data              │
│ - Handle HTTP response formatting                      │
│ - Handle HTTP errors (convert to HTTP status codes)    │
│                                                         │
│ ❌ DON'T:                                               │
│ - Implement business logic                             │
│ - Query database directly                              │
│ - Call external APIs directly                          │
│ - Calculate complex values                             │
│ - Manage data transformations                          │
│ - Handle multiple concerns                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ SERVICE (Business Logic)                                │
├─────────────────────────────────────────────────────────┤
│ ✅ DO:                                                  │
│ - Implement the actual business logic                  │
│ - Use repositories to access data                      │
│ - Coordinate with other services                       │
│ - Calculate values                                     │
│ - Validate business rules                              │
│ - Return domain objects                                │
│                                                         │
│ ❌ DON'T:                                               │
│ - Know about HTTP (no Express req, res)                │
│ - Handle HTTP status codes                             │
│ - Format JSON for HTTP responses                       │
│ - Know about routing                                   │
│ - Extract request parameters                           │
└─────────────────────────────────────────────────────────┘
```

### 2.3 The Responsibility Question

For each piece of logic, ask: **"Does the Controller need to know this?"**

```
Question: Should the controller know about calculating tax?
Answer: NO
Why: Tax calculation is a business rule, not HTTP-related
Where: PricingService

Question: Should the controller know about validating JWT?
Answer: NO (guards handle this)
Where: Guards/middleware

Question: Should the controller know about validating product stock?
Answer: NO
Why: That's a business rule
Where: ProductService

Question: Should the controller know about formatting a date?
Answer: NO
Why: That's data transformation
Where: Service or DTO transformer

Question: Should the controller know about checking if user has permission?
Answer: NO (guards handle this)
Where: Guards/Authorization

Question: Should the controller extract userId from request?
Answer: YES
Why: That's part of handling the HTTP request
Where: Controller (but pass to service)

Question: Should the controller decide what to return in the response?
Answer: YES
Why: That's formatting for HTTP
Where: Controller
```

---

## Part 3: Service Design - Method Signatures (Your Job as an Architect)

This is crucial: **Your job is to design the method signature. Let AI implement the body.**

### 3.1 Signature = Contract

A method signature is a **contract** between the controller and the service.

```typescript
// Good signature:
async createOrder(
  userId: string,
  items: CreateOrderItemDto[],
  paymentMethodId: string
): Promise<OrderDto>

// What it says:
// - Input: User ID (string), items (array of objects), payment method
// - Output: Promise that resolves to an OrderDto
// - Clear: Everything needed is a parameter
// - Testable: Can call with any values, easy to mock

// Bad signature:
async createOrder() {
  // What does it do?
  // Where do it get data?
  // What does it return?
  // Can't test because unclear what to pass
}

// Worse signature:
async process(data: any): Promise<any> {
  // Super vague
  // Can't type-check anything
  // Can't generate tests
}
```

### 3.2 Designing Good Signatures

The signature should express:
1. **What data the service needs** (parameters)
2. **What the service guarantees** (return type)
3. **What can go wrong** (thrown exceptions)

#### **Example 1: Order Creation**

```typescript
// ❌ VAGUE
async createOrder(data: any) {
  // What's in data?
  // What's returned?
  // Unclear
}

// ✅ CLEAR
async createOrder(
  userId: string,
  items: CreateOrderItemDto[],
  paymentMethodId: string
): Promise<OrderCreatedResponseDto> {
  // Clear input: userId (string), items (array), paymentMethodId
  // Clear output: OrderCreatedResponseDto
  // Implementation can be complex, but interface is clear
  
  // Can throw:
  // - UserNotFoundException (user doesn't exist)
  // - ProductOutOfStockException (product unavailable)
  // - PaymentFailedException (payment declined)
  // - InvalidPaymentMethodException (invalid method)
}

// DTO types define the shape:
export class CreateOrderItemDto {
  productId: string;
  quantity: number;
}

export class OrderCreatedResponseDto {
  id: string;
  userId: string;
  items: CreateOrderItemDto[];
  total: number;
  tax: number;
  status: 'confirmed' | 'failed';
  createdAt: Date;
}
```

#### **Example 2: User Registration**

```typescript
// ❌ VAGUE
async register(body: any) {
  // Unclear what's in body
}

// ✅ CLEAR
async registerUser(
  email: string,
  password: string,
  firstName: string,
  lastName: string
): Promise<UserRegistrationResponseDto> {
  // Clear what's needed
  // Clear what's returned
  // Implementation: validate email, hash password, save to DB
  
  // Can throw:
  // - EmailAlreadyExistsException
  // - WeakPasswordException
  // - InvalidEmailFormatException
}

export class UserRegistrationResponseDto {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  createdAt: Date;
}
```

#### **Example 3: List with Pagination**

```typescript
// ❌ VAGUE
async getOrders(filters: any) {
  // What filters?
  // How is pagination handled?
}

// ✅ CLEAR
async getUserOrders(
  userId: string,
  page: number = 1,
  limit: number = 10,
  status?: 'pending' | 'confirmed' | 'failed'
): Promise<PaginatedOrdersResponseDto> {
  // Clear parameters
  // Clear return type with pagination info
  
  // Can throw:
  // - UserNotFoundException
  // - InvalidPaginationException
}

export class PaginatedOrdersResponseDto {
  data: OrderDto[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

### 3.3 The Signature-Driven Workflow (How to Direct AI)

This is your workflow as an architect:

```
Step 1: Design the signature (YOU do this)
├─ What parameters are needed?
├─ What's the return type?
├─ What exceptions can be thrown?
└─ Document in DTOs

Step 2: Ask AI to implement (AI does this)
├─ Give AI the signature
├─ Give AI any business rules or constraints
├─ Let AI implement the body

Step 3: Review the implementation (YOU do this)
├─ Does it follow the signature?
├─ Are there any side effects?
├─ Is the logic testable?
└─ Does it match your design intent?

Example Prompt to AI:

"Implement this service method:

@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private productService: ProductService,
    private pricingService: PricingService,
    private paymentGateway: PaymentGateway
  ) {}
  
  async createOrder(
    userId: string,
    items: CreateOrderItemDto[],
    paymentMethodId: string
  ): Promise<OrderCreatedResponseDto> {
    // Your implementation here
    
    // Requirements:
    // 1. Validate that all items exist and have stock
    // 2. Calculate total with tax
    // 3. Create order record with status 'pending'
    // 4. Charge the payment method
    // 5. If payment succeeds, update status to 'confirmed'
    // 6. If payment fails, update status to 'failed' and throw PaymentFailedException
    // 7. Return the created order
    
    // Throw:
    // - UserNotFoundException if user doesn't exist
    // - ProductNotFoundException if product doesn't exist
    // - OutOfStockException if not enough stock
    // - PaymentFailedException if payment fails
  }
}

Do not change the method signature. Implement the body according to requirements."

This prompt gives AI:
✅ Clear method signature (can't change)
✅ Clear requirements (business rules)
✅ Clear error cases (what to throw)
✅ Dependencies (already injected)
```

---

## Part 4: Service Granularity - How Big Should Services Be?

### 4.1 The Goldilocks Principle

```
TOO SMALL:                    JUST RIGHT:                TOO BIG:
└─ God Services             ├─ Single Responsibility   ├─ Super Services
  ├─ Do one thing,          ├─ Clear purpose           ├─ Do everything
    extremely narrow         ├─ Cohesive methods        ├─ Multiple reasons
  ├─ One method             ├─ Few dependencies           to change
  ├─ Lots of services       ├─ Easy to test            ├─ Hard to test
  ├─ Hard to navigate       ├─ Easy to understand      ├─ Hard to understand
  └─ Too much ceremony      └─ Reusable                └─ Tight coupling

Example: Too Small           Example: Just Right        Example: Too Big
UserValidationService        UserService               UserService
├─ validateEmail()           ├─ registerUser()         ├─ registerUser()
                             ├─ getUser()              ├─ getUser()
UserPasswordService          ├─ updateProfile()        ├─ updateProfile()
├─ hashPassword()            ├─ changePassword()       ├─ changePassword()
                             └─ deleteUser()           ├─ resetPassword()
UserProfileService                                    ├─ verifyEmail()
├─ updateBio()              Single service with        ├─ sendWelcomeEmail()
                             related methods!          ├─ createNotification()
These are utilities,                                  ├─ chargeSubscription()
not services.                                         ├─ sendInvoice()
Too much overhead.                                    ├─ logAnalytics()
                                                       └─ 10 more...
```

### 4.2 The Test: Can You Describe It in One Sentence?

```
✅ GOOD: Single responsibility
┌─────────────────────────────────────┐
│ UserService: Manages user accounts  │
│ (registration, profiles, passwords) │
└─────────────────────────────────────┘

❌ BAD: Multiple responsibilities
┌──────────────────────────────────────┐
│ UserService: Manages users... and    │
│ sends notifications... and charges   │
│ subscriptions... and logs events...  │
└──────────────────────────────────────┘
```

### 4.3 Grouping Methods into Services

Ask: **"Do these methods have the same reason to change?"**

```
Methods that change together → Same service
Methods that change separately → Different services

Example: E-Commerce

❌ BAD Grouping:
UserService
├─ registerUser()
├─ getUser()
├─ updateProfile()
├─ processRefund()        ← Why is refund here?
├─ sendNotification()     ← Why is notification here?
├─ chargePayment()        ← Why is payment here?

Better Grouping:
UserService
├─ registerUser()
├─ getUser()
├─ updateProfile()
(Only user-related methods)

RefundService
├─ processRefund()        ← Refund-specific logic

NotificationService
├─ sendEmail()
├─ sendSMS()

PaymentService
├─ chargePayment()
├─ refundPayment()
```

### 4.4 Cohesion in Methods

All methods in a service should work together toward one goal:

```
✅ COHESIVE:
export class OrderService {
  // All these methods are about creating and managing orders
  
  async createOrder(userId: string, items: Item[]): Promise<Order>
  async getOrder(orderId: string): Promise<Order>
  async listUserOrders(userId: string): Promise<Order[]>
  async cancelOrder(orderId: string): Promise<void>
  async updateOrderStatus(orderId: string, status: string): Promise<void>
  
  // They all use the same repository
  // They all change when "how we manage orders" changes
  // They all need the same dependencies
}

❌ INCOHERENT:
export class OrderService {
  async createOrder(userId: string, items: Item[]): Promise<Order>
  async getOrder(orderId: string): Promise<Order>
  async sendEmail(to: string, subject: string): Promise<void>  // ← Email?!
  async processPayment(amount: number): Promise<PaymentResult>  // ← Payment?!
  async logEvent(event: string): Promise<void>  // ← Logging?!
  
  // These are different concerns
  // They don't all change together
  // They have different dependencies
  // Should be different services
}
```

### 4.5 The Size Metrics

```
Guidelines (not hard rules):

Lines of code:         50-300 lines is typical
Methods:              3-10 methods is typical
Dependencies:         2-4 dependencies is typical
Reasons to change:    1 (that's the point!)

If your service is:
<50 lines, 1-2 methods   → Probably too small (combine with related)
50-300 lines, 3-10 meth  → Good size
>500 lines, >15 methods  → Probably too big (split up)

If your service has:
<2 dependencies          → Good (few external concerns)
2-4 dependencies         → Normal
5+ dependencies          → Red flag (doing too much)
```

---

## Part 5: Common Service Patterns

### 5.1 The Repository Pattern (Data Access)

Services use repositories to access data. Repositories are simple providers:

```typescript
// ❌ BAD: Service queries database directly
@Injectable()
export class UserService {
  constructor(private db: Database) {}
  
  async getUser(id: string) {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    return result;
  }
}

// ✅ GOOD: Service uses repository
@Injectable()
export class UserRepository {
  constructor(private db: Database) {}
  
  async findById(id: string): Promise<User | null> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    return result ? new User(result) : null;
  }
}

@Injectable()
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async getUser(id: string): Promise<User> {
    const user = await this.userRepository.findById(id);
    if (!user) throw new NotFoundException('User not found');
    return user;
  }
}

Why this matters:
✅ Service doesn't know HOW data is stored (SQL, MongoDB, cache, etc.)
✅ Repository is reusable across services
✅ Easy to switch databases (change repository, not service)
✅ Easy to mock in tests (mock repository)
✅ Service logic is separate from data access logic
```

### 5.2 The Service Composition Pattern (Services Using Services)

Services can depend on other services:

```typescript
// UserService: Handles user-specific logic
@Injectable()
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async getUser(id: string): Promise<User> {
    return this.userRepository.findById(id);
  }
}

// PricingService: Handles pricing calculations
@Injectable()
export class PricingService {
  calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
}

// OrderService: Uses both
@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private userService: UserService,        // ← Uses UserService
    private pricingService: PricingService   // ← Uses PricingService
  ) {}
  
  async createOrder(userId: string, items: Item[]) {
    // Verify user exists
    const user = await this.userService.getUser(userId);
    
    // Calculate pricing
    const total = this.pricingService.calculateTotal(items);
    
    // Create order
    return this.orderRepository.save({
      userId: user.id,
      items,
      total,
    });
  }
}

// Module registers all of them
@Module({
  providers: [UserService, OrderService, PricingService, UserRepository, OrderRepository],
  exports: [OrderService],
})
export class OrderModule {}

Guidelines for service composition:
✅ Small services that do one thing
✅ Larger services that compose smaller ones
✅ Avoid circular dependencies (use events instead)
✅ Each level adds a layer of abstraction
✅ Easy to test each layer independently
```

### 5.3 The Event-Driven Pattern (Decoupling Services)

When services need to communicate but shouldn't depend on each other:

```typescript
// ❌ WRONG: Services depend on each other
@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private notificationService: NotificationService  // ← Tight coupling
  ) {}
  
  async createOrder(userId: string, items: Item[]) {
    const order = await this.orderRepository.save({ userId, items });
    
    // OrderService knows about NotificationService
    // If NotificationService changes, OrderService might break
    await this.notificationService.sendOrderConfirmation(userId, order);
    
    return order;
  }
}

// ✅ RIGHT: Services are decoupled via events
@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private eventEmitter: EventEmitter  // ← Generic event emitter
  ) {}
  
  async createOrder(userId: string, items: Item[]) {
    const order = await this.orderRepository.save({ userId, items });
    
    // Just emit an event
    // OrderService doesn't care who listens
    this.eventEmitter.emit('order:created', { orderId: order.id, userId });
    
    return order;
  }
}

@Injectable()
export class NotificationService {
  constructor(private eventEmitter: EventEmitter) {}
  
  constructor() {
    // Listen for the event
    // NotificationService doesn't depend on OrderService
    this.eventEmitter.on('order:created', (data) => {
      this.sendOrderConfirmation(data.userId, data.orderId);
    });
  }
  
  private async sendOrderConfirmation(userId: string, orderId: string) {
    // Send email/SMS
  }
}

Benefits:
✅ OrderService doesn't know about NotificationService
✅ NotificationService doesn't know about OrderService
✅ Can add more listeners without changing OrderService
✅ Can disable notifications without changing OrderService
✅ Services are independently testable
```

---

## Part 6: Designing Services for AI Implementation

This is where your skill as an architect shines:

### 6.1 The Signature-First Workflow

You design the interface. AI implements the body.

#### **Step 1: Think About the Domain**

```
Question: What does this service do?
Order Management

Question: What are the main operations?
- Create orders
- Get order details
- List user's orders
- Update order status
- Cancel orders

Question: What data does each operation need?
Create: userId, items, payment method
Get: orderId
List: userId, pagination params
Update: orderId, new status
Cancel: orderId, reason

Question: What can go wrong?
Create: user doesn't exist, product out of stock, payment fails
Get: order doesn't exist
Cancel: order is already shipped, can't cancel
```

#### **Step 2: Design the Signatures**

```typescript
export class CreateOrderDto {
  @IsString() userId: string;
  @IsArray() items: OrderItemDto[];
  @IsString() paymentMethodId: string;
}

export class OrderItemDto {
  @IsString() productId: string;
  @IsNumber() quantity: number;
}

export class OrderResponseDto {
  id: string;
  userId: string;
  items: OrderItemDto[];
  total: number;
  status: OrderStatus;
  createdAt: Date;
  updatedAt: Date;
}

@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private productService: ProductService,
    private pricingService: PricingService,
    private paymentGateway: PaymentGateway,
    private eventEmitter: EventEmitter
  ) {}
  
  // Signature 1: Create order
  async createOrder(
    userId: string,
    items: OrderItemDto[],
    paymentMethodId: string
  ): Promise<OrderResponseDto> {
    // Implementation
  }
  
  // Signature 2: Get order
  async getOrder(orderId: string): Promise<OrderResponseDto> {
    // Implementation
  }
  
  // Signature 3: List user's orders
  async listUserOrders(
    userId: string,
    page: number = 1,
    limit: number = 10
  ): Promise<PaginatedResponseDto<OrderResponseDto>> {
    // Implementation
  }
  
  // Signature 4: Update status
  async updateOrderStatus(
    orderId: string,
    status: OrderStatus
  ): Promise<OrderResponseDto> {
    // Implementation
  }
  
  // Signature 5: Cancel order
  async cancelOrder(
    orderId: string,
    reason: string
  ): Promise<void> {
    // Implementation
  }
}
```

#### **Step 3: Write Requirements as Comments**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private productService: ProductService,
    private pricingService: PricingService,
    private paymentGateway: PaymentGateway
  ) {}
  
  /**
   * Creates a new order
   * 
   * @param userId - The user creating the order
   * @param items - Array of items to order
   * @param paymentMethodId - Payment method to charge
   * @returns The created order
   * 
   * @throws UserNotFoundException - if user doesn't exist
   * @throws ProductNotFoundException - if product doesn't exist
   * @throws OutOfStockException - if product doesn't have enough stock
   * @throws PaymentFailedException - if payment is declined
   * 
   * Requirements:
   * 1. Validate user exists using userService
   * 2. Validate each product exists and has sufficient stock
   * 3. Calculate total price using pricingService
   * 4. Create order record with status 'pending'
   * 5. Attempt payment via paymentGateway
   * 6. If payment succeeds:
   *    - Update order status to 'confirmed'
   *    - Emit 'order:created' event with orderId
   * 7. If payment fails:
   *    - Update order status to 'failed'
   *    - Throw PaymentFailedException
   * 8. Return the order
   */
  async createOrder(
    userId: string,
    items: OrderItemDto[],
    paymentMethodId: string
  ): Promise<OrderResponseDto> {
    // AI implements here
  }
}
```

#### **Step 4: Ask AI to Implement**

```
PROMPT TO AI:

"You are implementing the OrderService. 

The method signatures are already defined. DO NOT CHANGE THEM.

Here's the service skeleton with one method to implement:

[Paste the class with signatures]

Implement ONLY the createOrder method according to these requirements:

1. Validate user exists (throw UserNotFoundException if not)
2. For each item:
   - Validate product exists (throw ProductNotFoundException if not)
   - Check stock availability (throw OutOfStockException if not enough)
3. Calculate total using pricingService.calculateTotal(items)
4. Create order record:
   - Use orderRepository.save() with status: 'pending'
   - Include all item details and calculated total
5. Process payment:
   - Call paymentGateway.charge(paymentMethodId, total)
   - This returns { success: boolean, transactionId: string }
6. Handle payment result:
   - If success:
     * Update order status to 'confirmed' using orderRepository.updateStatus()
     * Emit 'order:created' event with { orderId: order.id, userId }
     * Return the order
   - If fails:
     * Update order status to 'failed'
     * Throw PaymentFailedException('Payment was declined')

Use async/await. Return the order as OrderResponseDto.

Requirements:
- Use the injected dependencies
- Follow the exception types specified
- Don't create new services or change signatures
- Make it testable (no hidden dependencies)"
```

AI will generate something like:

```typescript
async createOrder(
  userId: string,
  items: OrderItemDto[],
  paymentMethodId: string
): Promise<OrderResponseDto> {
  // Validate user exists
  const user = await this.userService.getUser(userId);
  if (!user) {
    throw new UserNotFoundException(`User ${userId} not found`);
  }
  
  // Validate products and get pricing
  const pricedItems = [];
  for (const item of items) {
    const product = await this.productService.getProduct(item.productId);
    if (!product) {
      throw new ProductNotFoundException(`Product ${item.productId} not found`);
    }
    if (product.stock < item.quantity) {
      throw new OutOfStockException(`Product ${item.productId} out of stock`);
    }
    pricedItems.push({
      ...item,
      price: product.price,
      lineTotal: product.price * item.quantity,
    });
  }
  
  // Calculate total
  const { subtotal, tax, total } = this.pricingService.calculateTotal(pricedItems);
  
  // Create order
  const order = await this.orderRepository.save({
    userId,
    items: pricedItems,
    subtotal,
    tax,
    total,
    status: 'pending',
  });
  
  // Process payment
  const paymentResult = await this.paymentGateway.charge(paymentMethodId, total);
  
  if (paymentResult.success) {
    await this.orderRepository.updateStatus(order.id, 'confirmed');
    this.eventEmitter.emit('order:created', { orderId: order.id, userId });
    return { ...order, status: 'confirmed' };
  } else {
    await this.orderRepository.updateStatus(order.id, 'failed');
    throw new PaymentFailedException('Payment was declined');
  }
}
```

#### **Step 5: Review**

Check:
- ✅ Does it follow the signature?
- ✅ Does it implement all requirements?
- ✅ Are error cases handled correctly?
- ✅ Is it testable (uses injected dependencies)?
- ✅ Is the logic clear and maintainable?
- ✅ Any issues? Ask AI to refine

---

## Part 7: Testing Services (Why DI + Good Design = Easy Testing)

This brings everything together: Modules → DI → Services → Testing

### 7.1 Unit Test Structure

```typescript
describe('OrderService', () => {
  let service: OrderService;
  let mockOrderRepository: jest.Mocked<OrderRepository>;
  let mockProductService: jest.Mocked<ProductService>;
  let mockPricingService: jest.Mocked<PricingService>;
  let mockPaymentGateway: jest.Mocked<PaymentGateway>;
  let mockEventEmitter: jest.Mocked<EventEmitter>;
  
  beforeEach(() => {
    // Create mocks for all dependencies
    mockOrderRepository = {
      save: jest.fn(),
      updateStatus: jest.fn(),
      findById: jest.fn(),
    };
    
    mockProductService = {
      getProduct: jest.fn(),
    };
    
    mockPricingService = {
      calculateTotal: jest.fn(),
    };
    
    mockPaymentGateway = {
      charge: jest.fn(),
    };
    
    mockEventEmitter = {
      emit: jest.fn(),
      on: jest.fn(),
    };
    
    // Inject mocks
    service = new OrderService(
      mockOrderRepository,
      mockProductService,
      mockPricingService,
      mockPaymentGateway,
      mockEventEmitter
    );
  });
  
  describe('createOrder', () => {
    it('should create order successfully with valid data', async () => {
      // Arrange
      const userId = 'user123';
      const items = [{ productId: 'prod1', quantity: 2 }];
      const paymentMethodId = 'pm_123';
      
      mockProductService.getProduct.mockResolvedValue({
        id: 'prod1',
        price: 50,
        stock: 10,
      });
      
      mockPricingService.calculateTotal.mockReturnValue({
        subtotal: 100,
        tax: 10,
        total: 110,
      });
      
      mockOrderRepository.save.mockResolvedValue({
        id: 'order123',
        userId,
        items,
        subtotal: 100,
        tax: 10,
        total: 110,
        status: 'pending',
      });
      
      mockPaymentGateway.charge.mockResolvedValue({
        success: true,
        transactionId: 'tx123',
      });
      
      mockOrderRepository.updateStatus.mockResolvedValue({
        id: 'order123',
        status: 'confirmed',
      });
      
      // Act
      const result = await service.createOrder(userId, items, paymentMethodId);
      
      // Assert
      expect(result.status).toBe('confirmed');
      expect(mockOrderRepository.save).toHaveBeenCalledWith(
        expect.objectContaining({ userId, status: 'pending' })
      );
      expect(mockPaymentGateway.charge).toHaveBeenCalledWith(paymentMethodId, 110);
      expect(mockEventEmitter.emit).toHaveBeenCalledWith(
        'order:created',
        expect.objectContaining({ orderId: 'order123' })
      );
    });
    
    it('should throw when payment fails', async () => {
      // Arrange
      mockProductService.getProduct.mockResolvedValue({
        id: 'prod1',
        price: 50,
        stock: 10,
      });
      
      mockPricingService.calculateTotal.mockReturnValue({
        subtotal: 100,
        tax: 10,
        total: 110,
      });
      
      mockOrderRepository.save.mockResolvedValue({
        id: 'order123',
        status: 'pending',
      });
      
      mockPaymentGateway.charge.mockResolvedValue({
        success: false,  // ← Payment fails
        error: 'Card declined',
      });
      
      // Act & Assert
      await expect(
        service.createOrder('user123', [{productId: 'prod1', quantity: 2}], 'pm_123')
      ).rejects.toThrow(PaymentFailedException);
      
      expect(mockOrderRepository.updateStatus).toHaveBeenCalledWith('order123', 'failed');
      expect(mockEventEmitter.emit).not.toHaveBeenCalled();
    });
    
    it('should throw when product is out of stock', async () => {
      // Arrange
      mockProductService.getProduct.mockResolvedValue({
        id: 'prod1',
        price: 50,
        stock: 1,  // ← Not enough stock
      });
      
      // Act & Assert
      await expect(
        service.createOrder('user123', [{productId: 'prod1', quantity: 5}], 'pm_123')
      ).rejects.toThrow(OutOfStockException);
      
      // Should not save or charge
      expect(mockOrderRepository.save).not.toHaveBeenCalled();
      expect(mockPaymentGateway.charge).not.toHaveBeenCalled();
    });
  });
});

// Why this works:
// ✅ Service is injectable (constructor takes all dependencies)
// ✅ All dependencies can be mocked
// ✅ Each test case is isolated
// ✅ Tests run instantly (no I/O)
// ✅ Can test success and failure paths
// ✅ Can verify service called dependencies correctly
```

---

## Part 8: Common Service Mistakes & Red Flags

### Mistake 1: Service Creating Its Own Dependencies

```typescript
// ❌ BAD
@Injectable()
export class OrderService {
  private orderRepository = new OrderRepository(  // ← Creates its own!
    new Database()  // ← Creates database!
  );
  
  async createOrder(userId: string, items: Item[]) {
    return this.orderRepository.save({ userId, items });
  }
}

// ✅ GOOD
@Injectable()
export class OrderService {
  constructor(private orderRepository: OrderRepository) {}  // ← Injected
  
  async createOrder(userId: string, items: Item[]) {
    return this.orderRepository.save({ userId, items });
  }
}
```

### Mistake 2: Service Doing Too Much

```typescript
// ❌ BAD: Service is 500+ lines, does 15 different things
@Injectable()
export class OrderService {
  async createOrder() { /* ... */ }
  async sendNotification() { /* ... */ }
  async processPayment() { /* ... */ }
  async updateInventory() { /* ... */ }
  async generateInvoice() { /* ... */ }
  async updateAnalytics() { /* ... */ }
  // ... 9 more methods
}

// ✅ GOOD: Small focused services
@Injectable()
export class OrderService {
  async createOrder() { /* ... */ }
  async getOrder() { /* ... */ }
  async listOrders() { /* ... */ }
}

@Injectable()
export class PaymentService {
  async processPayment() { /* ... */ }
}

@Injectable()
export class NotificationService {
  async sendNotification() { /* ... */ }
}

// Composed in controllers or orchestration services
```

### Mistake 3: Exposing Internal Details

```typescript
// ❌ BAD: Service exposes repository
export class OrderService {
  constructor(private orderRepository: OrderRepository) {}
  
  // Why expose this?
  getRepository(): OrderRepository {
    return this.orderRepository;
  }
}

// Usage: Others bypass service and use repository directly
const order = orderService.getRepository().findById('123');

// ✅ GOOD: Keep repository private
export class OrderService {
  constructor(private readonly orderRepository: OrderRepository) {}
  
  async getOrder(id: string): Promise<Order> {
    return this.orderRepository.findById(id);
  }
}

// Others must use service method
const order = await orderService.getOrder('123');
```

### Mistake 4: Service Depending on HTTP

```typescript
// ❌ BAD: Service knows about Express
import { Request, Response } from 'express';

@Injectable()
export class OrderService {
  async createOrder(req: Request, res: Response) {
    // Service shouldn't know about HTTP!
    res.json({ success: true });
  }
}

// ✅ GOOD: Service is framework-agnostic
@Injectable()
export class OrderService {
  async createOrder(userId: string, items: Item[]): Promise<OrderResponseDto> {
    // Service only knows about business logic
    return { id: '123', userId, items, total: 100 };
  }
}

// Controller handles HTTP
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}
  
  @Post()
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    const order = await this.orderService.createOrder(req.user.id, dto.items);
    return order;  // NestJS handles HTTP response
  }
}
```

### Mistake 5: Returning Database Objects Instead of DTOs

```typescript
// ❌ BAD: Exposing database schema
@Injectable()
export class UserService {
  async getUser(id: string): Promise<UserEntity> {  // ← Raw entity
    return this.userRepository.findById(id);
  }
}

// Problems:
// - Exposes internal implementation (UserEntity)
// - If you add database fields, they're exposed
// - Can't change database schema without affecting API

// ✅ GOOD: Return DTO
export class UserResponseDto {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
}

@Injectable()
export class UserService {
  async getUser(id: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(id);
    if (!user) throw new NotFoundException();
    
    return {
      id: user.id,
      email: user.email,
      firstName: user.firstName,
      lastName: user.lastName,
      // password, salt, etc. are NOT exposed
    };
  }
}

// Benefits:
// ✅ API contract is explicit
// ✅ Can change database schema without affecting API
// ✅ Don't expose sensitive fields (passwords, salt, etc.)
```

---

## Part 9: Module Review Checklist (For Reviewing AI Code)

When AI generates a module with services, use this checklist:

```
□ SERVICE DESIGN
  □ Each service has a single responsibility
  □ Service name clearly describes what it does
  □ All methods are related (same reason to change)
  □ Service doesn't do too much (not 500+ lines)

□ DEPENDENCIES
  □ All dependencies injected via constructor
  □ No 'new' keywords inside methods
  □ No global singletons (no CurrentUser.get(), etc.)
  □ Dependencies are immutable (readonly keywords)

□ METHOD SIGNATURES
  □ Parameters are explicit (not 'data: any')
  □ Return types are specified (not 'any')
  □ DTOs are used for complex input/output
  □ No HTTP-specific types (Request, Response)

□ EXPORTS
  □ Only services exported, not repositories
  □ Internal utilities are not exported
  □ Only the main service exported (if applicable)

□ TESTING
  □ Constructor takes all dependencies
  □ Easy to mock dependencies
  □ No hardcoded values/credentials
  □ Can test without external services

□ INTEGRATION WITH MODULES
  □ Service registered in module providers
  □ Service exported if needed by other modules
  □ No circular dependencies between services

□ ERROR HANDLING
  □ Throws appropriate exceptions (NotFoundException, etc.)
  □ Exception types are documented in JSDoc
  □ Error messages are helpful
```

---

## Part 10: Directing AI - Prompt Templates

### Template 1: Generate a Service from Scratch

```
"Create a [ServiceName] service with these responsibilities:
- [Responsibility 1]
- [Responsibility 2]
- [Responsibility 3]

The service should:
1. Be marked with @Injectable()
2. Use constructor dependency injection
3. Inject these dependencies: [list]
4. Export these methods (signatures below):
   
   async method1(param: Type): Promise<ReturnType>
   async method2(param: Type): Promise<ReturnType>

Do not:
- Create any dependencies (they're injected)
- Directly access HTTP request/response
- Hardcode any configuration
- Exceed 300 lines (split into smaller services if needed)

Implement each method according to these requirements:
[Detail requirements for each method]"
```

### Template 2: Generate Unit Tests

```
"Generate unit tests for [ServiceName] using Jest.

Requirements:
1. Test each public method
2. Mock all dependencies
3. Test success paths
4. Test error scenarios
5. Verify service calls dependencies correctly
6. Use describe() blocks to organize

The service has these dependencies:
- [Dependency1]: [what it does]
- [Dependency2]: [what it does]

Public methods:
- [Method signature]
- [Method signature]

Test these scenarios:
1. [Success scenario]
2. [Error scenario]
3. [Edge case]
"
```

### Template 3: Review Service Code

```
"Review this service code for:
1. Single responsibility principle
2. Proper dependency injection
3. Testability
4. Adherence to module boundaries
5. Error handling

[Paste code]

For any issues found, suggest specific improvements."
```

---

## Part 11: How It All Connects

Let me show you the complete picture of how Modules, DI, and Services work together:

### 11.1 The Architecture Flow

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: Define Module Boundaries (Your Job)                │
│ - UserModule handles user logic                             │
│ - OrderModule handles order logic                           │
│ - PaymentModule handles payment logic                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: Create Services in Each Module (Design Signatures) │
│ - UserService: registerUser(), getUser(), updateProfile()  │
│ - OrderService: createOrder(), getOrder(), listOrders()    │
│ - PaymentService: charge(), refund()                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: Design Dependencies (DI Architecture)              │
│ - UserService depends on UserRepository                     │
│ - OrderService depends on OrderRepository, UserService     │
│ - PaymentService depends on PaymentGateway                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: Register in Modules (Binding Providers)            │
│ @Module({                                                   │
│   providers: [UserService, UserRepository],                 │
│   exports: [UserService]                                    │
│ })                                                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: DI Container Resolves (Automatic Injection)        │
│ - Creates singleton instances                              │
│ - Injects dependencies where needed                         │
│ - Manages lifecycle                                         │
│ - Detects circular dependencies                            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 6: Controllers Use Services (HTTP Handling)           │
│ @Controller('orders')                                       │
│ export class OrderController {                              │
│   constructor(private orderService: OrderService) {}       │
│   @Post()                                                   │
│   create(@Body() dto: CreateOrderDto) {                    │
│     return this.orderService.createOrder(...)              │
│   }                                                         │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 7: Testing (Everything is Testable)                   │
│ - Services are injectable → easy to mock                    │
│ - All dependencies are explicit → nothing hidden           │
│ - Signatures are clear → know what to test                 │
│ - Tests run in milliseconds → no external I/O              │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 Code Example Showing It All Together

```typescript
// ===== STEP 1 & 2: Module Boundaries + Service Signatures =====

// UserModule
@Module({
  providers: [UserService, UserRepository],
  exports: [UserService],
})
export class UserModule {}

@Injectable()
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async registerUser(email: string, password: string): Promise<UserResponseDto> {
    // Implementation
  }
  
  async getUser(id: string): Promise<UserResponseDto> {
    // Implementation
  }
}

// OrderModule
@Module({
  imports: [UserModule],  // ← Imports UserModule, gets access to UserService
  providers: [OrderService, OrderRepository],
  exports: [OrderService],
})
export class OrderModule {}

@Injectable()
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private userService: UserService  // ← From UserModule
  ) {}
  
  async createOrder(userId: string, items: Item[]): Promise<OrderResponseDto> {
    // Implementation: Uses UserService and OrderRepository
  }
}

// ===== STEP 3 & 4: DI Setup =====
// NestJS automatically:
// 1. Creates UserService with UserRepository injected
// 2. Creates OrderService with OrderRepository and UserService injected
// 3. Manages singleton instances
// 4. Detects circular dependencies

// ===== STEP 5 & 6: Controllers Use Services =====
@Controller('orders')
export class OrderController {
  constructor(private orderService: OrderService) {}  // ← Injected by DI
  
  @Post()
  async create(@Body() dto: CreateOrderDto, @Req() req: any) {
    // Controller: Handle HTTP (extract user from JWT)
    // Service: Business logic
    return this.orderService.createOrder(req.user.id, dto.items);
  }
}

// ===== STEP 7: Testing =====
describe('OrderService', () => {
  let service: OrderService;
  let mockOrderRepository: jest.Mocked<OrderRepository>;
  let mockUserService: jest.Mocked<UserService>;
  
  beforeEach(() => {
    mockOrderRepository = { save: jest.fn(), /* ... */ };
    mockUserService = { getUser: jest.fn(), /* ... */ };
    
    // Inject mocks
    service = new OrderService(mockOrderRepository, mockUserService);
  });
  
  it('should create order with valid user and items', async () => {
    mockUserService.getUser.mockResolvedValue({ id: 'user123', email: '...' });
    // ... rest of test
  });
});
```

---

## Part 12: Quick Reference - Services Checklist

Before writing or asking AI to generate a service, check:

```
BEFORE DESIGN:
□ What is the single responsibility of this service?
□ Can I describe it in one sentence?
□ What are the main operations?
□ What data does each operation need?
□ What can go wrong?

DURING DESIGN:
□ Create clear method signatures (input/output)
□ Create DTOs for complex types
□ Document exceptions in JSDoc
□ List all dependencies needed
□ Verify no circular dependencies

WHEN ASKING AI:
□ Provide method signatures (don't let AI decide)
□ Explain business requirements clearly
□ Specify exception types to throw
□ List injected dependencies
□ Ask for testable code (no hardcoding)

WHEN REVIEWING:
□ Do all methods belong together?
□ Are all dependencies injected?
□ Does the service expose internal details?
□ Can I easily mock and test it?
□ Are return types DTOs, not raw entities?
□ Does it know about HTTP (bad sign)?
```

---

## Summary: Providers & Services

**What They Are:**
- **Providers**: Any injectable thing (Services, Values, Factories)
- **Services**: Classes marked with @Injectable() that implement business logic

**Your Job:**
1. Design the method signatures (input/output)
2. Define the DTOs (shape of data)
3. List the dependencies
4. Document the business rules

**AI's Job:**
1. Implement the method bodies
2. Call dependencies correctly
3. Handle error cases
4. Write unit tests

**Why It Matters:**
- Clear signatures → Easy to understand
- Clear signatures → Easy for AI to implement
- DI everywhere → Everything is testable
- Small focused services → Easy to maintain
- Services in modules → Code is organized

**Connected to Previous Topics:**
- **Modules**: Services are grouped in modules (boundaries)
- **DI**: Services are injected (testability)
- **Tests**: Services can be unit tested (clean code)

Master this, and you become the architect directing AI to build production-grade systems.
