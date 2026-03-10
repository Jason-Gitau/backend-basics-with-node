# NestJS Modules & Module Boundaries: Deep Dive

## What This Guide Does
This guide teaches you **how to think about modules** so you can architect systems, direct AI to scaffold them cleanly, and review generated code for tight coupling and architectural debt.

---

## Part 1: The Core Concept of Modules

### 1.1 What is a Module, Really?

In NestJS, a **module is a container that groups related domain logic** under a single boundary. Think of it as a shipping container:
- Everything inside serves a single **purpose**
- Has **clearly defined entry and exit points** (what it exports, what it imports)
- Can be tested, replaced, or understood **independently**

```typescript
// This is the structure of every NestJS module:
@Module({
  imports: [],      // What this module needs from outside
  controllers: [],  // Entry points (HTTP, WebSocket, etc.)
  providers: [],    // The actual business logic (services, repos, etc.)
  exports: [],      // What this module shares with others
})
export class UserModule {}
```

**Key insight for AI direction:**
When you ask AI to "scaffold a module," you're asking it to:
1. Create a logical boundary
2. Place only related logic inside
3. Define clear import/export contracts
4. Avoid leaking internal details

### 1.2 Why Modules Matter (The Spaghetti Code Problem)

Without proper module boundaries, here's what happens:

```typescript
// ❌ BAD: UserService knows about OrderService knows about PaymentService knows about...
// This is spaghetti code—everything depends on everything

export class UserService {
  constructor(
    private orderService: OrderService,
    private paymentService: PaymentService,
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService,
    private emailService: EmailService
  ) {}
  
  registerUser(data: any) {
    // User logic tangled with order, payment, notification, analytics...
    // Can't test this in isolation
    // Can't reuse in a different context
    // Hard to understand what "registering a user" actually means
  }
}
```

When you have tight coupling like this:
- **Testing is hard**: You need to mock 5 services to test one simple action
- **Reusability is low**: The User logic is locked into this specific ecosystem
- **Debugging is painful**: A bug could be in User, Order, Payment, Notification...
- **Changes ripple everywhere**: Modify one service, break code across the codebase
- **AI generates spaghetti too**: If you don't give AI a clear boundary, it will create dependencies everywhere

---

## Part 2: Understanding Module Boundaries

### 2.1 The Boundary Question: What Belongs Together?

A module boundary should group things that:

#### ✅ **Have the Same Reason to Change**
This is the core principle. If two pieces of code change for the same business reason, they belong in the same module.

```typescript
// ✅ GOOD: UserModule
// All these change when "how we manage users" changes

@Module({
  providers: [
    UserService,           // Logic for user operations
    UserRepository,        // How we persist users
    UserValidator,         // How we validate user data
    UserMapper,           // How we transform user DTOs
  ],
  exports: [UserService],
})
export class UserModule {}

// If tomorrow someone says "we need to change how users are stored,
// validated, and transformed" — it's ALL in UserModule. Good!
```

#### ✅ **Share the Same Domain Language**
Members of a module should think of themselves as part of one "team" solving one problem.

```typescript
// ✅ GOOD: OrderModule
// Everything here speaks "Order" language

export class OrderService {
  // Methods like:
  // - createOrder()
  // - calculateTotal()
  // - applyDiscount()
  // - shipOrder()
}

export class OrderRepository {
  // Methods like:
  // - saveOrder()
  // - findOrderById()
  // - updateOrderStatus()
}

// NOT in OrderModule: UserService, PaymentService, NotificationService
// Those are different domains with different language
```

#### ✅ **Form a Cohesive Unit**
You should be able to describe the module in **one sentence**:

```typescript
// ✅ GOOD descriptions:
// "UserModule manages user registration, authentication, and profiles"
// "OrderModule handles order creation, status tracking, and fulfillment"
// "PaymentModule processes transactions and manages payment methods"

// ❌ BAD descriptions:
// "This module does... user stuff... and orders... and sometimes payments..."
// (If you can't describe it in one sentence, it's too broad)
```

### 2.2 Anti-Pattern: Tight Coupling

**Tight coupling** happens when modules can't exist independently. Here's what it looks like:

```typescript
// ❌ SPAGHETTI: UserModule is tightly coupled to OrderModule

// Inside users/user.service.ts
export class UserService {
  constructor(
    private userRepository: UserRepository,
    private orderService: OrderService  // ← Problem! UserService needs OrderService
  ) {}
  
  getUserWithOrders(userId: string) {
    const user = this.userRepository.findById(userId);
    const orders = this.orderService.getUserOrders(userId); // ← Tight coupling
    return { user, orders };
  }
}

// Problem:
// - You can't use UserService without OrderService
// - You can't test UserService without mocking OrderService
// - They're interdependent in confusing ways
// - If Order logic changes, User logic might break
```

**Why AI generates this:**
When you ask AI "build me a user service," and you haven't clearly scoped what UserService should do, AI will create dependencies everywhere because it "makes sense to fetch orders when getting a user."

### 2.3 Good Design: Loose Coupling

**Loose coupling** means modules can exist independently and communicate through well-defined contracts:

```typescript
// ✅ GOOD: Loose coupling

// In users/user.service.ts
export class UserService {
  constructor(
    private userRepository: UserRepository  // ← Only what User needs
  ) {}
  
  getUser(userId: string) {
    return this.userRepository.findById(userId);
  }
  
  // If orders are needed, that's handled at a HIGHER level
  // NOT in UserService
}

// In orders/order.service.ts
export class OrderService {
  constructor(
    private orderRepository: OrderRepository  // ← Only what Order needs
  ) {}
  
  getUserOrders(userId: string) {
    return this.orderRepository.findByUserId(userId);
  }
}

// At a higher level (like a controller), you compose them:
@Controller('users')
export class UserController {
  constructor(
    private userService: UserService,
    private orderService: OrderService  // ← Composed at the boundary
  ) {}
  
  @Get(':id/profile')
  async getUserWithOrders(@Param('id') userId: string) {
    const user = await this.userService.getUser(userId);
    const orders = await this.orderService.getUserOrders(userId);
    return { user, orders };  // ← Composition happens here
  }
}

// Result:
// - UserService can be tested alone
// - OrderService can be tested alone
// - They don't know about each other
// - Either can be replaced independently
// - Easy to understand what each does
```

---

## Part 3: Recognizing and Avoiding Coupling

### 3.1 The Coupling Spectrum

```
Loose ←————————————————→ Tight

[Independent] → [Imported] → [Depends on] → [Circular] → [Spaghetti]

✅ Good                              ❌ Bad
```

### 3.2 Types of Coupling to Watch For

#### **1. Direct Service Injection (Watch This!)**

```typescript
// ⚠️ MEDIUM COUPLING: OrderService directly depends on UserService

@Module({
  providers: [OrderService, OrderRepository, UserService],
})
export class OrderModule {}

export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private userService: UserService  // ← Direct dependency
  ) {}
  
  createOrder(userId: string, items: Item[]) {
    const user = this.userService.getUser(userId); // ← Service calls service
    // ... create order
  }
}

// Problem: OrderModule now depends on UserService
// This means: you can't use OrderService without UserService being available
// For unit testing, you need to mock UserService
// If UserService changes, OrderService might break
```

**When this is OK:**
- When the dependency is **truly fundamental** to the domain
- Example: `OrderService` depending on `OrderRepository` is fine (same module)

**When this is NOT OK:**
- When importing a service from **another module** just to call it
- That's a sign the modules should talk through **events**, **commands**, or **queries** instead

#### **2. Repository Leakage (Common AI Mistake)**

```typescript
// ❌ BAD: UserRepository exposed, OrderService uses it directly

// users/user.repository.ts
export class UserRepository {
  findById(id: string) { /* ... */ }
}

// users/user.module.ts
@Module({
  providers: [UserService, UserRepository],
  exports: [UserRepository],  // ← Exporting repository! Bad!
})
export class UserModule {}

// orders/order.service.ts
export class OrderService {
  constructor(
    private userRepository: UserRepository  // ← Using another module's repository!
  ) {}
}

// Problems:
// - OrderService is now coupled to how users are stored (database, cache, etc.)
// - If user storage changes, OrderService breaks
// - OrderService shouldn't care HOW users are stored, just needs user data
// - This is like calling someone's internal plumbing instead of using their door
```

**The fix:**

```typescript
// ✅ GOOD: Only export services, not repositories

// users/user.module.ts
@Module({
  providers: [UserService, UserRepository],
  exports: [UserService],  // ← Only export the service!
})
export class UserModule {}

// orders/order.service.ts
export class OrderService {
  constructor(
    private userService: UserService  // ← Call the service
  ) {}
}
```

#### **3. Circular Dependencies (The Spaghetti Symptom)**

```typescript
// ❌ VERY BAD: UserModule depends on OrderModule depends on UserModule

// users/user.service.ts
export class UserService {
  constructor(private orderService: OrderService) {} // ← Points to OrderModule
}

// orders/order.service.ts
export class OrderService {
  constructor(private userService: UserService) {} // ← Points back to UserModule!
}

// Result: Can't instantiate either. NestJS will throw an error.
// This is a sign your modules are fighting over the same responsibility.
```

**Root cause of circular dependencies:**
- Two modules think they own the same logic
- Example: User module has a service that creates orders, Order module has a service that validates users

**Solution:**
- Extract the shared logic into a **third module**
- Or use **events** to decouple them

```typescript
// ✅ GOOD: Extract shared responsibility

// shared/event-emitter.service.ts
export class EventEmitterService {
  // Neutral module that others can publish/subscribe to
}

// users/user.service.ts
export class UserService {
  constructor(private events: EventEmitterService) {}
  
  registerUser(data) {
    // ... register
    this.events.emit('user.registered', { userId: user.id });
  }
}

// orders/order.service.ts
export class OrderService {
  constructor(private events: EventEmitterService) {}
  
  constructor() {
    // Listen for user registration, don't depend on UserService
    this.events.on('user.registered', (data) => {
      // ... initialize user's orders
    });
  }
}

// No circular dependency. They communicate through events.
```

---

## Part 4: How to Design Module Boundaries (The Process)

### 4.1 The Module Design Checklist

When designing modules (or asking AI to scaffold them), ask these questions:

#### **1. Cohesion: Does Everything Belong Together?**

```
Questions to ask:
□ Can I describe this module in one sentence?
□ Do all classes change for the same reason?
□ Do they use the same domain language?
□ If I remove one class, does the module still make sense?
□ Can a new developer understand "what this module does" quickly?
```

Example:
```typescript
// ✅ GOOD: UserModule
// "Manages user registration, authentication, profile updates, and password resets"
// Everything here changes when "how we handle users" changes

// ❌ BAD: UserModule
// "Manages users... and sends emails... and creates orders... and processes payments"
// These are different domains with different reasons to change
```

#### **2. Coupling: Are Dependencies Minimal and Clear?**

```
Questions to ask:
□ Does this module depend on other modules? (List them)
□ Are these dependencies necessary or accidental?
□ Could I replace this dependency with something else?
□ Are there circular dependencies?
□ Can I test this module in isolation?
```

Example:
```typescript
// ✅ GOOD: Clear dependencies
// OrderModule imports:
//   - UserModule (to validate users exist)
//   - ProductModule (to check product availability)
// These are necessary domain dependencies

// ❌ BAD: Too many dependencies
// OrderModule imports:
//   - UserModule, ProductModule, PaymentModule, NotificationModule,
//     AnalyticsModule, LoggingModule, CacheModule, ...
// Suggests OrderModule is doing too much
```

#### **3. Public API: What Does This Module Export?**

```
Questions to ask:
□ What is the minimal set of exports?
□ Should I export services? Yes. Repositories? No. Controllers? No.
□ What would someone importing this module expect?
□ Are there internal details leaking out?
```

Example:
```typescript
// ✅ GOOD: Minimal, clear exports
@Module({
  providers: [UserService, UserRepository, UserValidator, PasswordHasher],
  exports: [UserService],  // Only the main service
})
export class UserModule {}

// Importing module knows: "I can use UserService"
// It doesn't care about: Repository, Validator, PasswordHasher (internal)

// ❌ BAD: Leaking internals
@Module({
  providers: [UserService, UserRepository, UserValidator, PasswordHasher],
  exports: [UserService, UserRepository, UserValidator, PasswordHasher],
  // ↑ Everything exposed! Others will start depending on UserRepository directly!
})
export class UserModule {}
```

### 4.2 Real-World Module Design Examples

#### **Example 1: E-Commerce System**

```
Bad Module Boundaries:
├── UserModule (registers, logs in, orders, pays, gets notifications)
├── ProductModule (creates, searches, recommends, analyzes)
└── AdminModule (does everything for admins)

Problem: UserModule does too much. UserModule, ProductModule, AdminModule
overlap. Hard to test. Hard to reason about.
```

```
Good Module Boundaries:
├── UserModule (authentication, profiles, registration)
├── OrderModule (creates, tracks, manages orders)
├── ProductModule (catalog, search, inventory)
├── PaymentModule (processes payments, manages payment methods)
├── NotificationModule (sends emails, SMS, push notifications)
├── SharedModule (utilities, common DTOs, filters)
└── AppModule (orchestrates all of them)

Each module:
- Does one thing well
- Can be tested independently
- Has clear exports
- Minimizes dependencies
```

#### **Example 2: Building a Module From Scratch (What to Tell AI)**

**Bad prompt (leads to spaghetti):**
```
"Create a UserModule with all the user stuff"
```
AI response: Creates a service that depends on Orders, Payments, Notifications, Analytics...

**Good prompt (clear boundaries):**
```
"Create a UserModule that handles:
- User registration with email validation
- User profile management (update name, bio, avatar)
- Password reset
- User retrieval by ID

The module should:
- Export only UserService
- NOT depend on any domain modules (OrderModule, PaymentModule, etc.)
- Use UserRepository internally (not exported)
- Provide a clean interface for other modules to call UserService"
```

AI response: Clean, focused, testable module.

---

## Part 5: Evaluating AI-Generated Modules (Code Review Checklist)

When AI generates a module, here's how to review it for coupling and architectural debt:

### 5.1 The Review Checklist

```typescript
// AI Generated Module - Time to Review

@Module({
  imports: [/*...*/],
  providers: [/*...*/],
  exports: [/*...*/],
})
export class GeneratedModule {}
```

**Check 1: Are Imports Necessary?**
```typescript
@Module({
  imports: [UserModule, OrderModule, PaymentModule, NotificationModule],
  //       ↑ Stop. Is this module really supposed to depend on all four?
  //       If yes, maybe it's doing too much
  //       If no, ask AI to remove unnecessary imports
})
```

**Check 2: Are Exports Minimal?**
```typescript
@Module({
  exports: [SomeService, SomeRepository, SomeValidator, SomeHelper],
  //       ↑ Too many! Only export SomeService. Keep internals private.
})
```

**Check 3: Do Provider Names Match Their Purpose?**
```typescript
// ✅ GOOD: Names are clear
providers: [
  UserService,        // I know this handles user logic
  UserRepository,     // I know this handles persistence
  UserValidator,      // I know this validates users
  UserMapper,         // I know this transforms user data
]

// ❌ BAD: Generic names
providers: [
  Service,           // What service?
  Manager,           // What does it manage?
  Helper,            // What does it help with?
  Utils,             // Vague
]
```

**Check 4: Look for Circular Dependencies**
```typescript
// Ask yourself:
// - Does UserModule import OrderModule?
// - Does OrderModule import UserModule?
// If yes to both: ❌ RED FLAG. Ask AI to refactor using events.
```

**Check 5: Can You Understand It in 30 Seconds?**
```typescript
// If you read the module definition, can you say in one sentence what it does?
// If not, the boundaries are unclear. Ask AI to clarify or break it up.
```

### 5.2 Common AI Mistakes to Look For

**Mistake 1: Service-to-Service Calls Across Modules**

```typescript
// AI Generated OrderService
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private userService: UserService,        // ← Importing from UserModule!
    private productService: ProductService,  // ← Importing from ProductModule!
    private paymentService: PaymentService   // ← Importing from PaymentModule!
  ) {}
  
  createOrder(userId, items) {
    // Calls UserService.getUser()
    // Calls ProductService.getProduct() multiple times
    // Calls PaymentService.processPayment()
    // This is tight coupling!
  }
}
```

**How to fix (ask AI):**
```
"The OrderService is doing too much. For a basic order:
1. OrderService should only validate the order itself (items, quantities)
2. Validation that user/product/payment is correct should happen
   at the controller/handler level, not in OrderService
3. If OrderService needs to call other modules, use events instead:
   - Publish 'order.created' event
   - Other modules listen and react (process payment, send notification)
4. Refactor OrderService to be independent"
```

**Mistake 2: Exposing Internal Repositories**

```typescript
// AI Generated UserModule
@Module({
  providers: [UserService, UserRepository],
  exports: [UserService, UserRepository],  // ← Oops! Exposed repository
})
export class UserModule {}

// Now other modules will use UserRepository directly
// This breaks encapsulation
```

**How to fix (ask AI):**
```
"Remove UserRepository from exports. Only export UserService.
If other modules need user data, they call UserService, not UserRepository."
```

**Mistake 3: Creating a "God Module"**

```typescript
// AI Generated: AppService in AppModule
export class AppService {
  constructor(
    private userService: UserService,
    private orderService: OrderService,
    private productService: ProductService,
    private paymentService: PaymentService,
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService,
    private cacheService: CacheService,
    // ... 10 more services
  ) {}
  
  // Does everything in one service
}

// This is spaghetti. This is what you want to avoid.
```

**How to fix (ask AI):**
```
"Break AppService into smaller, focused services. Each should handle
one piece of the application flow. Use orchestration patterns like:
- Commands: AppService sends commands to OrderService, UserService, etc.
- Events: Services emit events, other services react
- Sagas: Complex workflows managed by a saga orchestrator
Never have one service that depends on and calls all other services."
```

---

## Part 6: Practical Example: Designing a Real Module

Let's design a **PaymentModule** from scratch. Here's how to think about it:

### 6.1 Step 1: Define the Domain Boundary

```
What is PaymentModule responsible for?
- Processing credit card payments
- Managing payment methods
- Tracking payment status
- Refunds

What is it NOT responsible for?
- Sending notifications (NotificationModule does that)
- Logging analytics (AnalyticsModule does that)
- Managing user accounts (UserModule does that)
- Creating orders (OrderModule does that)
```

### 6.2 Step 2: Define the Public Interface

```typescript
// This is what other modules will use
export class PaymentService {
  processPayment(paymentData: {
    userId: string,
    amount: number,
    paymentMethodId: string,
  }): Promise<PaymentResult> {
    // ... implementation
  }
  
  refundPayment(paymentId: string): Promise<RefundResult> {
    // ... implementation
  }
  
  getPaymentStatus(paymentId: string): Promise<PaymentStatus> {
    // ... implementation
  }
  
  // No other module should care about these:
  // getPaymentRepository()  ← Don't expose this
  // validateCardNumber()    ← Don't expose this
  // encryptPaymentMethod()  ← Don't expose this
}
```

### 6.3 Step 3: Define Internal Providers (Hidden)

```typescript
// These are internal to PaymentModule, never exported

export class PaymentRepository {
  // Database operations
  // Other modules don't need to know this exists
}

export class CardValidator {
  // Validates credit cards
  // Internal implementation detail
}

export class PaymentGatewayAdapter {
  // Handles Stripe, PayPal, etc.
  // Other modules don't care which gateway we use
}

export class EncryptionService {
  // Encrypts sensitive payment data
  // Internal security concern
}
```

### 6.4 Step 4: Define Dependencies

```typescript
// What does PaymentModule actually need?
// - A database (handled by repository)
// - A payment gateway (Stripe, PayPal, etc.)
// - Maybe a configuration module
// - Maybe a logging module
// That's it. No OrderModule, UserModule, NotificationModule, etc.

// If OrderModule needs to know payment was successful,
// PaymentModule emits an event: 'payment.completed'
// OrderModule listens and reacts
```

### 6.5 Step 5: Define the Module

```typescript
@Module({
  imports: [ConfigModule],  // Needs config for API keys
  providers: [
    PaymentService,         // Public API
    PaymentRepository,      // Internal
    CardValidator,          // Internal
    PaymentGatewayAdapter,  // Internal
    EncryptionService,      // Internal
  ],
  exports: [PaymentService], // Only export the service!
})
export class PaymentModule {}
```

### 6.6 Step 7: How Other Modules Use It

```typescript
// OrderModule wants to process payment for an order
// It doesn't have tight coupling. It just uses PaymentService.

@Module({
  imports: [PaymentModule],  // Import the module
  providers: [OrderService, OrderRepository],
})
export class OrderModule {}

// OrderService
export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private paymentService: PaymentService  // ← Use it
  ) {}
  
  async createOrder(userId: string, items: Item[]) {
    const order = await this.orderRepository.create({
      userId,
      items,
      status: 'pending_payment'
    });
    
    const paymentResult = await this.paymentService.processPayment({
      userId,
      amount: order.total,
      paymentMethodId: 'user_card_123',
    });
    
    if (paymentResult.success) {
      await this.orderRepository.updateStatus(order.id, 'completed');
      // Emit event so other modules know
      this.eventEmitter.emit('order.completed', { orderId: order.id });
    }
    
    return order;
  }
}
```

---

## Part 7: Summary: The Rules for Module Design

### Key Principles

```
1. COHESION: Everything in a module should change together
   ✅ If something would change independently, move it to another module
   
2. MINIMAL COUPLING: Modules should be as independent as possible
   ✅ Minimize imports from other modules
   ✅ Use events to decouple instead of direct service calls
   
3. CLEAR EXPORTS: Hide internals, expose only what others need
   ✅ Export services, not repositories
   ✅ Export interfaces, not implementations
   
4. SINGLE RESPONSIBILITY: One module, one reason to change
   ✅ If you're describing it with "and", split it
   
5. OBSERVABILITY: You should understand it in 30 seconds
   ✅ If not, the boundaries are unclear
```

### Directing AI for Good Modules

**Good prompt:**
```
"Create a [DomainName]Module that:
1. Handles [specific responsibility]
2. Exports only [ServiceName]
3. Does NOT import [specific other modules] — use events if coordination needed
4. Provides these methods: [list public methods]
5. Keeps [internal detail] private"
```

**Bad prompt:**
```
"Create a [DomainName]Module with all the [domain] stuff"
```

---

## Part 8: Exercises to Solidify Your Understanding

### Exercise 1: Identify the Spaghetti

```typescript
// Look at this module. What's wrong?

export class UserService {
  constructor(
    private userRepository: UserRepository,
    private orderService: OrderService,
    private paymentService: PaymentService,
    private emailService: EmailService,
    private smsService: SmsService,
    private analyticsService: AnalyticsService,
  ) {}
  
  registerUser(data) {
    // Creates user
    // Creates initial order
    // Processes trial payment
    // Sends welcome email
    // Sends SMS
    // Tracks analytics
  }
}
```

**What's wrong?**
- [ ] Too many dependencies
- [ ] Does too much
- [ ] Changes for multiple reasons
- [ ] All of the above

**The fix:**
```
UserService should ONLY register the user. That's it.
Other modules listen for 'user.registered' event:
- OrderModule creates initial order
- PaymentModule processes trial payment
- NotificationModule sends email
- SmsModule sends SMS
- AnalyticsModule tracks signup
```

### Exercise 2: Design a Module From Scratch

**Scenario:** Build a `NotificationModule` for your app.

**Questions to answer:**
1. What is the single responsibility?
2. What is the public API (exported)?
3. What are internal details (not exported)?
4. What other modules does it need to depend on?
5. What other modules might depend on it?
6. Write the module definition

**Answer template:**
```typescript
// Single responsibility: Sending notifications (email, SMS, push) to users

// Public API: NotificationService with methods like sendEmail(), sendSMS(), sendPush()

// Internal: MailProvider, SmsProvider, PushProvider (users don't need to know about these)

// Dependencies: ConfigModule (for API keys), maybe LoggingModule (optional)

// Dependents: UserModule, OrderModule, PaymentModule (they all send notifications)

@Module({
  imports: [ConfigModule],
  providers: [
    NotificationService,
    MailProvider,
    SmsProvider,
    PushProvider,
  ],
  exports: [NotificationService],
})
export class NotificationModule {}
```

---

## Part 9: Red Flags & When You Know Boundaries Are Wrong

If you see these, your modules need redesign:

```
🚩 Circular dependencies (Module A imports B, B imports A)
🚩 One module depends on many others (more than 2-3)
🚩 Hard to test a service in isolation
🚩 You keep finding yourself reaching into another module's internals
🚩 Changes in one module break many other modules
🚩 You can't describe a module in one sentence
🚩 Repository leakage (modules using other modules' repositories)
🚩 God services that do everything
🚩 Naming that's vague (Manager, Helper, Service without context)
🚩 Every service depends on every other service
```

---

## Next Steps

Now that you understand module boundaries, here's what to study next:

1. **Dependency Injection (DI)** - How NestJS instantiates and manages services
2. **Providers** - Different types (Services, Factories, Values)
3. **Custom Decorators** - Metadata to guide AI
4. **Middleware & Guards** - Boundaries at the HTTP level
5. **Events** - How to decouple modules (publish/subscribe)

This foundation will let you:
- ✅ Design systems that stay clean as they grow
- ✅ Direct AI to scaffold good code
- ✅ Review AI code for architectural debt
- ✅ Debug issues knowing where to look
- ✅ Build production-grade systems

---

## Quick Reference: Module Checklist

Before telling AI to generate a module, have answers to these:

```
[ ] What is the single reason this module exists?
[ ] What is its public API? (What do other modules call?)
[ ] What is internal? (What does it hide?)
[ ] Which modules does it depend on? (Should be ≤3)
[ ] Can it be tested without mocking other domains?
[ ] Can I describe it in one sentence?
[ ] Do all classes change together?
[ ] Are any circular dependencies possible?
```

If you answer all of these clearly, AI will generate good code.
```
