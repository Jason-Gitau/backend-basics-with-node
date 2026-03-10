# TypeScript for AI Code Generation
## Designing Types That Guide AI to Correct Implementations

---

## The Problem: AI Without Clear Types

When you ask AI to write code without clear type definitions:

```typescript
// ❌ VAGUE REQUEST (AI must guess)
"Create a function that handles user data"

// What AI might write:
function handleUser(data: any): any {
    // AI guesses:
    // - What fields does data have?
    // - What should it return?
    // - What validation is needed?
    // Results in defensive, fragile code
    
    if (data.email) {
        // Maybe process email?
        return { user: data };
    }
}

// ✓ CLEAR REQUEST WITH TYPES (AI knows exactly)
interface UserInput {
    email: string;
    name: string;
    dateOfBirth: Date;
}

interface UserOutput {
    id: string;
    email: string;
    name: string;
    age: number;
    createdAt: Date;
}

"Implement this function that accepts UserInput and returns UserOutput"

function transformUser(input: UserInput): UserOutput {
    // AI KNOWS:
    // - input has email, name, dateOfBirth (guaranteed)
    // - Must return object with specific fields
    // - Can calculate age from dateOfBirth
    // - Must generate id
    
    const birthDate = new Date(input.dateOfBirth);
    const today = new Date();
    const age = today.getFullYear() - birthDate.getFullYear();
    
    return {
        id: generateId(),
        email: input.email,
        name: input.name,
        age,
        createdAt: new Date()
    };
}
```

---

## Principle 1: Be Explicit About What Fields Are Required

### The Mistake AI Makes

```typescript
// ❌ VAGUE
interface User {
    email: string;
    phone?: string;
    address?: string;
    preferences?: any;  // Too vague
}

// AI might write validation:
function validateUser(user: User): boolean {
    return !!user.email;  // Assumes only email matters
}

// But what if preferences has structure?
// AI can't know, so it guesses wrong
```

### How to Guide AI Correctly

```typescript
// ✓ EXPLICIT
interface User {
    id: string;
    email: string;           // Required, core data
    firstName: string;       // Required, core data
    lastName: string;        // Required, core data
    phone?: string;          // Optional, non-core
    preferences: {           // Required but can be empty
        emailNotifications: boolean;
        smsNotifications: boolean;
        darkMode: boolean;
    };
    metadata?: {             // Optional metadata
        lastLoginAt?: Date;
        loginCount?: number;
    };
}

// Now AI writes correct validation:
function validateUser(user: User): ValidationResult {
    const errors: string[] = [];
    
    // Core fields must exist and be valid
    if (!user.email || !user.email.includes('@')) {
        errors.push('Invalid email');
    }
    if (!user.firstName || user.firstName.trim().length === 0) {
        errors.push('First name is required');
    }
    
    return {
        valid: errors.length === 0,
        errors
    };
}
```

**What AI Learned:**
- Core fields need validation
- Optional fields are genuinely optional
- Nested objects have structure
- Can't use shorthand like `!!user.email` for complex requirements

---

## Principle 2: Use Literal Types to Constrain Choices

### The Mistake AI Makes

```typescript
// ❌ VAGUE
interface Payment {
    status: string;  // What values are valid?
    method: string;  // What values are valid?
}

// AI might write:
function updatePaymentStatus(payment: Payment, newStatus: string) {
    payment.status = newStatus;  // Could be anything!
    // What if newStatus is a typo? No validation possible
}
```

### How to Guide AI Correctly

```typescript
// ✓ EXPLICIT - Literal types constrain to exact values
interface Payment {
    id: string;
    amount: number;
    status: 'pending' | 'processing' | 'completed' | 'failed' | 'refunded';
    method: 'card' | 'mobile_money' | 'bank_transfer';
    
    // Business rules encoded in type
    // If status is 'refunded', refundDate must exist
    refundDate: Date | null;
}

// AI now writes correct logic:
function updatePaymentStatus(
    payment: Payment,
    newStatus: Payment['status']  // Only valid statuses allowed
): Payment {
    let refundDate = payment.refundDate;
    
    if (newStatus === 'refunded' && !payment.refundDate) {
        refundDate = new Date();  // Auto-set refund date
    }
    
    if (newStatus !== 'refunded' && payment.refundDate) {
        refundDate = null;  // Clear refund date if not refunding
    }
    
    return {
        ...payment,
        status: newStatus,
        refundDate
    };
}

// Usage:
let payment = { /* ... */, status: 'pending', refundDate: null };

payment = updatePaymentStatus(payment, 'processing');  // ✓ Valid
payment = updatePaymentStatus(payment, 'invalid');     // ✗ Compiler error
```

**What AI Learned:**
- Only specific values are allowed
- Can't create invalid states
- Business rules are encoded in types
- No defensive `if (status === 'something')` needed for unknown values

---

## Principle 3: Use Optional Fields Strategically

### The Mistake AI Makes

```typescript
// ❌ VAGUE - Everything optional
interface Product {
    name?: string;
    price?: number;
    description?: string;
    category?: string;
    sku?: string;
}

// AI doesn't know what's required for basic functionality
function getProduct(id: string): Product {
    // What if price is undefined? What if name is undefined?
    // Can't implement correctly
    return {};
}
```

### How to Guide AI Correctly

```typescript
// ✓ EXPLICIT - Clear requirements
interface Product {
    // These are REQUIRED - every product must have these
    id: string;
    sku: string;
    name: string;
    price: Money;
    category: string;
    
    // These are OPTIONAL - nice to have but not required
    description?: string;
    imageUrl?: string;
    
    // Timestamps - included in all entities
    createdAt: Date;
    updatedAt: Date;
}

// AI now knows what's guaranteed to exist:
function getProduct(id: string): Promise<Product> {
    const product = await db.products.findById(id);
    
    if (!product) {
        throw new ProductNotFoundError(id);
    }
    
    // AI knows these ALWAYS exist:
    const displayName = `${product.sku}: ${product.name}`;
    const formattedPrice = formatMoney(product.price);
    
    // AI handles optional fields carefully:
    const fullDescription = product.description || `Premium ${product.category}`;
    const image = product.imageUrl || getDefaultImage(product.category);
    
    return product;
}
```

**What AI Learned:**
- Required fields can be used without checking
- Optional fields need null checks
- Can provide sensible defaults for optional fields
- Less defensive code needed

---

## Principle 4: Use Branded Types for Domain-Specific Data

### The Mistake AI Makes

```typescript
// ❌ VAGUE - These are all strings, but different domains
interface Order {
    id: string;        // What if passed a user ID by accident?
    userId: string;    // Same type, different meaning
    sku: string;       // Different kind of string
}

// AI might make mistakes:
function findOrder(userId: string): Order {
    // Easy to accidentally pass wrong string type
    return { id: userId, userId: '???', sku: '' };  // WRONG!
}
```

### How to Guide AI Correctly

```typescript
// ✓ EXPLICIT - Branded types prevent mixing
// Branding: A way to make different strings distinguishable at compile time

type OrderId = string & { readonly __brand: 'OrderId' };
type UserId = string & { readonly __brand: 'UserId' };
type ProductSku = string & { readonly __brand: 'ProductSku' };

// Helper functions to create branded types
const createOrderId = (id: string): OrderId => id as OrderId;
const createUserId = (id: string): UserId => id as UserId;
const createProductSku = (sku: string): ProductSku => sku as ProductSku;

interface Order {
    id: OrderId;        // Specifically an OrderId
    userId: UserId;     // Specifically a UserId
    productSku: ProductSku;  // Specifically a ProductSku
}

// AI can't mix them up:
function findOrderByUser(userId: UserId): Promise<Order> {
    // Can only pass UserId here
    const order = await db.findOrder({ userId });
    return order;
}

// Usage:
const userId = createUserId('user-123');
const orderId = createOrderId('order-456');

const order = findOrderByUser(userId);        // ✓ Correct type
const order = findOrderByUser(orderId);       // ✗ Type mismatch - compiler error
```

**What AI Learned:**
- Different domains have different types
- Can't accidentally pass wrong ID type
- Compiler prevents mixing up IDs
- More maintainable for complex systems

---

## Principle 5: Use Generics to Avoid Duplication

### The Mistake AI Makes

```typescript
// ❌ DUPLICATED CODE - One for each type
async function getUser(id: string): Promise<User> {
    return db.users.findById(id);
}

async function getProduct(id: string): Promise<Product> {
    return db.products.findById(id);
}

async function getOrder(id: string): Promise<Order> {
    return db.orders.findById(id);
}

// Same function 3 times, boring!
```

### How to Guide AI Correctly

```typescript
// ✓ GENERIC - Write once, use everywhere

// Define the pattern
interface Entity {
    id: string;
    createdAt: Date;
    updatedAt: Date;
}

interface Repository<T extends Entity> {
    findById(id: string): Promise<T | null>;
    findAll(): Promise<T[]>;
    save(entity: T): Promise<T>;
    delete(id: string): Promise<boolean>;
}

// Implement once with generics
class GenericRepository<T extends Entity> implements Repository<T> {
    constructor(private collection: Collection<T>) {}
    
    async findById(id: string): Promise<T | null> {
        // AI knows T is Entity with id, createdAt, updatedAt
        return this.collection.findOne({ id } as any);
    }
    
    async findAll(): Promise<T[]> {
        return this.collection.find({});
    }
    
    async save(entity: T): Promise<T> {
        const now = new Date();
        return this.collection.updateOne(
            { id: entity.id } as any,
            {
                ...entity,
                updatedAt: now
            } as any
        );
    }
    
    async delete(id: string): Promise<boolean> {
        const result = await this.collection.deleteOne({ id } as any);
        return result.deletedCount > 0;
    }
}

// Use with any entity type:
const userRepo = new GenericRepository<User>(db.users);
const productRepo = new GenericRepository<Product>(db.products);
const orderRepo = new GenericRepository<Order>(db.orders);

// Same interface, works with all types
const user = await userRepo.findById('123');        // User | null
const product = await productRepo.findById('456');  // Product | null
const order = await orderRepo.findById('789');      // Order | null
```

**What AI Learned:**
- Generic pattern avoids duplication
- Works with any type that matches constraints
- Consistency across domains
- Less code to maintain

---

## Principle 6: Model Business Rules in Types

### The Mistake AI Makes

```typescript
// ❌ VAGUE - Rules in comments only
interface Account {
    balance: number;  // Must be >= 0, but how does AI know?
    monthlySpend: number;  // Should be reset monthly, but no enforcement
    accountType: string;  // What types exist?
}

// AI might not enforce rules:
function deposit(account: Account, amount: number): Account {
    // Forgot to validate amount > 0
    // Didn't cap max balance
    // Didn't handle different account types
    return { ...account, balance: account.balance + amount };
}
```

### How to Guide AI Correctly

```typescript
// ✓ EXPLICIT - Business rules in types

// Use branded types for amounts
type PositiveAmount = number & { readonly __brand: 'PositiveAmount' };
type Money = number;  // In cents

const createPositiveAmount = (amount: number): PositiveAmount => {
    if (amount <= 0) throw new Error('Amount must be positive');
    return amount as PositiveAmount;
};

// Model account types with constraints
type RegularAccountType = 'savings' | 'checking';
type PremiumAccountType = 'investment' | 'business';
type AccountType = RegularAccountType | PremiumAccountType;

interface Account {
    id: string;
    accountType: AccountType;
    balance: Money;      // In cents, can be any value
    monthlySpend: Money; // In cents
    monthlySpendReset: Date;  // When it resets
    maxBalance: Money;   // Account type determines this
}

// Business rules encoded in functions
function getMaxBalance(accountType: AccountType): Money {
    if (accountType === 'checking') return 100000 * 100;    // $100k
    if (accountType === 'savings') return 500000 * 100;     // $500k
    if (accountType === 'investment') return 5000000 * 100; // $5M
    if (accountType === 'business') return 50000000 * 100;  // $50M
}

// Deposit enforces business rules
function deposit(account: Account, amount: PositiveAmount): Account {
    const maxBalance = getMaxBalance(account.accountType);
    const newBalance = account.balance + amount;
    
    if (newBalance > maxBalance) {
        throw new DepositExceedsLimitError(maxBalance, newBalance);
    }
    
    return {
        ...account,
        balance: newBalance
    };
}

// Monthly spend must be reset
function resetMonthlySpend(account: Account): Account {
    const now = new Date();
    const needsReset = account.monthlySpendReset < now;
    
    if (needsReset) {
        return {
            ...account,
            monthlySpend: 0,
            monthlySpendReset: getNextMonthDate(now)
        };
    }
    
    return account;
}
```

**What AI Learned:**
- Business rules are explicit in types
- Can't violate constraints (compiler prevents it)
- Account types have different limits
- Operations understand the business logic

---

## Principle 7: Design for Evolution

### The Mistake AI Makes

```typescript
// ❌ TIGHT COUPLING - Hard to evolve
interface CreateOrderRequest {
    userId: string;
    items: { productId: string; quantity: number }[];
    shippingAddress: string;
    paymentMethod: string;
}

// If you need to add a field, must change everywhere
// If you need different payment validation, break backward compatibility
```

### How to Guide AI Correctly

```typescript
// ✓ DESIGNED FOR EVOLUTION

// Use extension/composition, not inheritance
interface CreateOrderRequestBase {
    userId: string;
    items: { productId: string; quantity: number }[];
}

interface CreateOrderRequestV1 extends CreateOrderRequestBase {
    shippingAddress: string;
    paymentMethod: string;
}

interface CreateOrderRequestV2 extends CreateOrderRequestBase {
    shipping: {
        address: string;
        method: 'standard' | 'express' | 'overnight';
        estimatedDays: number;
    };
    payment: {
        method: 'card' | 'mobile_money' | 'bank_transfer';
        cardLastFour?: string;
        mobileProvider?: string;
    };
}

// Handle both versions:
function createOrder(request: CreateOrderRequestV1 | CreateOrderRequestV2): Promise<Order> {
    // Normalize to internal format
    const shipping = 'shipping' in request
        ? request.shipping
        : {
            address: request.shippingAddress,
            method: 'standard',
            estimatedDays: 5
        };
    
    const payment = 'payment' in request
        ? request.payment
        : { method: request.paymentMethod };
    
    // Proceed with normalized data
    return createOrderInternal(request.userId, request.items, shipping, payment);
}

// Alternative: Use discriminated union
type CreateOrderRequest = 
    | { version: 1; userId: string; items: any[]; shippingAddress: string; paymentMethod: string }
    | { version: 2; userId: string; items: any[]; shipping: ShippingInfo; payment: PaymentInfo };

function createOrder(request: CreateOrderRequest): Promise<Order> {
    if (request.version === 1) {
        // Handle V1
    } else if (request.version === 2) {
        // Handle V2
    }
}
```

**What AI Learned:**
- Different versions can coexist
- Can gradually migrate
- Backward compatibility is explicit
- Changes don't break existing code

---

## Quick Checklist: Type Design for AI

When defining types for AI to use:

- [ ] **Are required fields clear?** (Not everything optional)
- [ ] **Are constraints explicit?** (Literal types, branded types, not just comments)
- [ ] **Are optional fields truly optional?** (Not "maybe" fields)
- [ ] **Are business rules encoded?** (Not in comments)
- [ ] **Is the pattern reusable?** (Generics instead of duplication)
- [ ] **Is it extensible?** (Can evolve without breaking)
- [ ] **Would an AI understand it?** (No ambiguity)

---

## The AI-Friendly Type Manifesto

```typescript
// ✓ DO THIS

interface Clear {
    required: string;
    also required: number;
    optional?: string;
    constrained: 'option1' | 'option2' | 'option3';
    branded: UserId;
}

function withSignatures<T extends SomeConstraint>(param: T): ResultType {
    // Clear what goes in, clear what comes out
}

// ❌ DON'T DO THIS

interface Vague {
    anything: any;
    maybe?: string;
    probably?: string;
    data?: any;
    metadata?: any;
}

function unclear(param) {
    // What goes in? What comes out?
    // AI must guess
}
```

---

## Real-World Example: JKUATHub Course Types

Let's apply these principles to a real system you're building:

```typescript
// Branded types prevent mixing IDs
type UserId = string & { readonly __brand: 'UserId' };
type CourseId = string & { readonly __brand: 'CourseId' };
type MaterialId = string & { readonly __brand: 'MaterialId' };

// Core entities
interface Course {
    id: CourseId;
    code: string;  // CS101, CS102, etc.
    name: string;
    department: string;
    semester: number;
    instructor: UserId;
    capacity: number;
    enrolled: number;
    materials: Material[];
    createdAt: Date;
    updatedAt: Date;
}

interface Material {
    id: MaterialId;
    courseId: CourseId;
    type: 'lecture' | 'assignment' | 'exam' | 'notes' | 'reading';
    title: string;
    description?: string;
    fileUrl: string;
    uploadedBy: UserId;
    uploadedAt: Date;
    downloadCount: number;
}

// API contracts
interface EnrollCourseRequest {
    courseId: CourseId;
}

interface EnrollCourseResponse {
    success: boolean;
    message: string;
    courseData?: Course;
}

// Repository pattern
interface CourseRepository {
    findById(id: CourseId): Promise<Course | null>;
    search(criteria: { code?: string; department?: string }): Promise<Course[]>;
    getEnrolledCourses(userId: UserId): Promise<Course[]>;
    enroll(userId: UserId, courseId: CourseId): Promise<Course>;
}

// Now AI can implement these correctly!
// Types guide every implementation
```

This is your superpower in the AI era: **type design.**
