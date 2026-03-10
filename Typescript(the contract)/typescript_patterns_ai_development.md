# TypeScript Patterns for AI-Assisted Development
## Practical Strategies and Real-World Examples

---

## Part 1: The Type-First Development Workflow

### The Old Way (JavaScript)

```
1. Rough idea in your head
2. Write implementation code
3. Test and find bugs
4. Debug for hours
5. Maybe it works
6. Repeat when requirements change
```

### The New Way (TypeScript + AI)

```
1. Define the data structures (Interfaces)
2. Define the transformations (Function signatures)
3. Ask AI to implement the logic
4. Review for business logic, not syntax
5. Compiler ensures type safety
6. Changes to types cascade through code automatically
```

### Real Example: Building a Payment System

**Step 1: Define the Contract**

```typescript
// Payment types
interface Money {
    amount: number;     // in cents to avoid floating point issues
    currency: 'USD' | 'EUR' | 'GBP' | 'KES';  // Kenyan Shilling for JKUATHub!
}

interface PaymentMethod {
    id: string;
    type: 'card' | 'mobile_money' | 'bank_transfer';
    lastFour?: string;
    provider: 'mpesa' | 'stripe' | 'wise';  // M-Pesa for Kenya
}

interface PaymentRequest {
    amount: Money;
    paymentMethod: PaymentMethod;
    description: string;
    idempotencyKey: string;  // Prevent duplicate charges
}

interface PaymentResponse {
    transactionId: string;
    status: 'pending' | 'completed' | 'failed' | 'refunded';
    amount: Money;
    timestamp: Date;
    metadata?: Record<string, any>;
}

interface PaymentError {
    code: 'INSUFFICIENT_FUNDS' | 'INVALID_METHOD' | 'NETWORK_ERROR' | 'DUPLICATE_REQUEST';
    message: string;
    retryable: boolean;
}

type PaymentResult = 
    | { success: true; data: PaymentResponse }
    | { success: false; error: PaymentError };
```

**Step 2: Ask AI to Implement**

```
You: "Given these types, implement the processPayment function that:
1. Validates the payment amount is positive
2. Checks idempotency (same key = same result)
3. Routes to the correct payment provider
4. Returns properly typed result"

// AI can now write:
const paymentCache = new Map<string, PaymentResponse>();

async function processPayment(request: PaymentRequest): Promise<PaymentResult> {
    // Check cache first (idempotency)
    const cached = paymentCache.get(request.idempotencyKey);
    if (cached) {
        return { success: true, data: cached };
    }
    
    // Validate amount
    if (request.amount.amount <= 0) {
        return {
            success: false,
            error: {
                code: 'INVALID_METHOD',
                message: 'Amount must be positive',
                retryable: false
            }
        };
    }
    
    try {
        const response = await routeToProvider(request);
        paymentCache.set(request.idempotencyKey, response);
        return { success: true, data: response };
    } catch (error) {
        return {
            success: false,
            error: {
                code: 'NETWORK_ERROR',
                message: error.message,
                retryable: true
            }
        };
    }
}
```

**Result:** AI wrote the correct implementation because the type contract made requirements explicit.

---

## Part 2: Building APIs with Type Safety

### Pattern: API Request/Response Typing

This pattern ensures your frontend and backend never disagree on data shapes.

```typescript
// ============ SHARED TYPES (shared/types.ts) ============
// Available to both frontend and backend

// User domain
interface User {
    id: string;
    email: string;
    name: string;
    role: 'student' | 'instructor' | 'admin';
    enrolledCourses: string[];  // Course IDs
}

interface CreateUserPayload {
    email: string;
    name: string;
    password: string;  // Only in request, never in response
}

// Course domain (for JKUATHub)
interface Course {
    id: string;
    code: string;      // e.g., "CS101"
    name: string;
    department: string;
    semester: number;
    materials: CourseMaterial[];
}

interface CourseMaterial {
    id: string;
    type: 'lecture' | 'assignment' | 'exam' | 'notes';
    title: string;
    url: string;
    uploadedAt: Date;
    uploadedBy: string;  // User ID
}

// API Endpoints
interface ApiEndpoint<Req, Res> {
    request: Req;
    response: Res;
}

// ============ BACKEND ============
// handlers/users.ts

async function handleCreateUser(payload: CreateUserPayload): Promise<User> {
    // AI knows:
    // - payload has email, name, password
    // - Must return User (with id)
    // - Must NOT return password in response
    
    const user = await db.users.create({
        email: payload.email,
        name: payload.name,
        passwordHash: await hashPassword(payload.password)
    });
    
    return {
        id: user.id,
        email: user.email,
        name: user.name,
        role: 'student',
        enrolledCourses: []
    };
}

// ============ FRONTEND ============
// api/users.ts

async function createUser(payload: CreateUserPayload): Promise<User> {
    const response = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
    });
    
    const data: User = await response.json();
    return data;
    // TypeScript guarantees data is User shape
    // Can't access non-existent fields
}

// ============ RESULT ============
// Frontend and backend both use SAME types
// Impossible to have shape mismatches
// Both can evolve safely (compiler warns if incompatible change)
```

### Benefits for AI:

1. **No Guessing:** AI sees exact request/response shapes
2. **No Duplication:** One definition, used everywhere
3. **Safe Refactoring:** Change type → compiler shows all affected code
4. **Documentation:** Types ARE the API documentation

---

## Part 3: Domain-Driven Design with Types

This approach is powerful for organizing complex systems like JKUATHub.

```typescript
// ============ USER DOMAIN ============
namespace UserDomain {
    export interface User {
        id: string;
        email: string;
        verified: boolean;
    }
    
    export interface CreateUserCommand {
        email: string;
        name: string;
    }
    
    export interface UserCreatedEvent {
        userId: string;
        email: string;
        createdAt: Date;
    }
}

// ============ COURSE DOMAIN ============
namespace CourseDomain {
    export interface Course {
        id: string;
        code: string;
        name: string;
        instructors: string[];  // User IDs
    }
    
    export interface EnrollmentCommand {
        userId: string;
        courseId: string;
    }
    
    export interface EnrollmentConfirmedEvent {
        userId: string;
        courseId: string;
        enrolledAt: Date;
    }
}

// ============ SHARED LOGIC ============
namespace EventBus {
    export interface Event {
        type: string;
        timestamp: Date;
    }
    
    export interface EventHandler<T extends Event> {
        handle(event: T): Promise<void>;
    }
    
    export interface EventStore {
        publish<T extends Event>(event: T): Promise<void>;
        subscribe<T extends Event>(
            eventType: new (...args: any[]) => T,
            handler: EventHandler<T>
        ): void;
    }
}

// ============ USING THE DOMAINS ============

// When user enrolls in course, emit event
async function enrollUserInCourse(command: CourseDomain.EnrollmentCommand): Promise<void> {
    const event: CourseDomain.EnrollmentConfirmedEvent = {
        userId: command.userId,
        courseId: command.courseId,
        enrolledAt: new Date()
    };
    
    await eventBus.publish(event);
}

// Another service listens for enrollment
class EnrollmentNotificationService implements EventBus.EventHandler<CourseDomain.EnrollmentConfirmedEvent> {
    async handle(event: CourseDomain.EnrollmentConfirmedEvent): Promise<void> {
        // AI knows event has userId, courseId, enrolledAt
        const user = await users.getById(event.userId);
        const course = await courses.getById(event.courseId);
        
        await email.send({
            to: user.email,
            subject: `Enrolled in ${course.name}`,
            body: `You're now enrolled in ${course.name}`
        });
    }
}
```

**Why This Works:**
- Each domain is self-contained (low coupling)
- Types make boundaries explicit
- AI can implement handlers knowing exact event shape
- Prevents accidental cross-domain dependencies

---

## Part 4: Repository Pattern for Data Access

This pattern is crucial for keeping data access type-safe.

```typescript
// Generic repository interface
interface IRepository<T extends BaseEntity> {
    create(entity: T): Promise<T>;
    findById(id: string): Promise<T | null>;
    findAll(): Promise<T[]>;
    findBy(criteria: Partial<T>): Promise<T[]>;
    update(id: string, updates: Partial<T>): Promise<T>;
    delete(id: string): Promise<boolean>;
}

// Base entity all entities inherit from
interface BaseEntity {
    id: string;
    createdAt: Date;
    updatedAt: Date;
}

// Course entity
interface Course extends BaseEntity {
    code: string;
    name: string;
    department: string;
    materials: CourseMaterial[];
}

// Repository implementation
class CourseRepository implements IRepository<Course> {
    async create(course: Course): Promise<Course> {
        // AI knows:
        // - Must save all Course properties
        // - Must set createdAt, updatedAt
        // - Must return Course type
        const now = new Date();
        return db.courses.insertOne({
            ...course,
            createdAt: now,
            updatedAt: now
        });
    }
    
    async findById(id: string): Promise<Course | null> {
        // Return Course or null, nothing else
        return db.courses.findOne({ id });
    }
    
    async findBy(criteria: Partial<Course>): Promise<Course[]> {
        // Only filter by Course properties
        return db.courses.find(criteria);
    }
    
    async update(id: string, updates: Partial<Course>): Promise<Course> {
        // AI knows updates can only contain Course properties
        // createdAt never updated (business rule enforced by type)
        return db.courses.updateOne(
            { id },
            {
                ...updates,
                updatedAt: new Date()
            }
        );
    }
    
    async delete(id: string): Promise<boolean> {
        return db.courses.deleteOne({ id }).then(r => r.deletedCount > 0);
    }
}

// Usage
const courseRepo = new CourseRepository();

// Get all courses
const courses = await courseRepo.findAll();
// TypeScript knows this is Course[]
courses.forEach(course => {
    console.log(course.code);  // ✓ Known property
    console.log(course.invalid); // ✗ Compiler error
});

// Find courses by criteria
const csCourses = await courseRepo.findBy({ department: 'Computer Science' });
// TypeScript only allows searching by valid Course properties
```

**For AI Review:**
- Repository methods should only allow valid properties
- Updates should be `Partial<T>` to allow partial updates
- All methods should respect the generic type T

---

## Part 5: State Management with Types

For complex applications like JKUATHub.

```typescript
// ============ STATE STRUCTURE ============
interface AppState {
    user: UserState;
    courses: CourseState;
    ui: UIState;
}

interface UserState {
    current: User | null;
    loading: boolean;
    error: string | null;
}

interface CourseState {
    enrolled: Course[];
    available: Course[];
    loading: boolean;
}

interface UIState {
    sidebarOpen: boolean;
    selectedCourseId: string | null;
    notificationQueue: Notification[];
}

// ============ ACTIONS ============
type UserAction = 
    | { type: 'LOGIN_REQUEST' }
    | { type: 'LOGIN_SUCCESS'; payload: User }
    | { type: 'LOGIN_FAILURE'; payload: string }
    | { type: 'LOGOUT' };

type CourseAction = 
    | { type: 'FETCH_COURSES_REQUEST' }
    | { type: 'FETCH_COURSES_SUCCESS'; payload: Course[] }
    | { type: 'FETCH_COURSES_FAILURE'; payload: string }
    | { type: 'ENROLL'; payload: string };  // courseId

type UIAction = 
    | { type: 'TOGGLE_SIDEBAR' }
    | { type: 'SELECT_COURSE'; payload: string }
    | { type: 'ADD_NOTIFICATION'; payload: Notification }
    | { type: 'REMOVE_NOTIFICATION'; payload: string };

type AppAction = UserAction | CourseAction | UIAction;

// ============ REDUCERS ============
function userReducer(state: UserState, action: UserAction): UserState {
    switch (action.type) {
        case 'LOGIN_REQUEST':
            return { ...state, loading: true, error: null };
        
        case 'LOGIN_SUCCESS':
            return { ...state, current: action.payload, loading: false };
        
        case 'LOGIN_FAILURE':
            return { ...state, error: action.payload, loading: false };
        
        case 'LOGOUT':
            return { current: null, loading: false, error: null };
        
        default:
            return state;
    }
}

function courseReducer(state: CourseState, action: CourseAction): CourseState {
    switch (action.type) {
        case 'FETCH_COURSES_SUCCESS':
            // AI knows action.payload is Course[]
            return { ...state, available: action.payload, loading: false };
        
        case 'ENROLL':
            // AI knows action.payload is courseId (string)
            const course = state.available.find(c => c.id === action.payload);
            if (!course) return state;
            
            return {
                ...state,
                enrolled: [...state.enrolled, course],
                available: state.available.filter(c => c.id !== action.payload)
            };
        
        default:
            return state;
    }
}

// ============ USAGE ============
// Dispatch with type safety
dispatch({ type: 'LOGIN_REQUEST' });  // ✓ Valid action
dispatch({ type: 'INVALID_ACTION' });  // ✗ Compiler error

dispatch({ type: 'LOGIN_SUCCESS', payload: userData });  // ✓ Correct
dispatch({ type: 'LOGIN_SUCCESS', payload: 'string' });  // ✗ Compiler error - must be User
```

**Benefits:**
- AI can't dispatch invalid actions
- Payload types are enforced
- Reducers always return correct state type
- Type-safe state management

---

## Part 6: Error Handling with Types

```typescript
// ============ ERROR HIERARCHY ============
abstract class AppError extends Error {
    constructor(
        public code: string,
        public message: string,
        public statusCode: number = 500,
        public retryable: boolean = false
    ) {
        super(message);
    }
}

class NotFoundError extends AppError {
    constructor(resource: string) {
        super('NOT_FOUND', `${resource} not found`, 404, false);
    }
}

class ValidationError extends AppError {
    constructor(public field: string, message: string) {
        super('VALIDATION_ERROR', message, 400, false);
    }
}

class NetworkError extends AppError {
    constructor(message: string) {
        super('NETWORK_ERROR', message, 503, true);  // Retryable
    }
}

// ============ RESULT TYPE ============
type Result<T, E extends AppError = AppError> = 
    | { ok: true; value: T }
    | { ok: false; error: E };

// ============ USAGE ============
async function getUserEmail(userId: string): Promise<Result<string>> {
    try {
        // Validate ID format
        if (!userId.match(/^[a-f0-9]{24}$/)) {
            return {
                ok: false,
                error: new ValidationError('userId', 'Invalid user ID format')
            };
        }
        
        const user = await db.users.findById(userId);
        if (!user) {
            return {
                ok: false,
                error: new NotFoundError('User')
            };
        }
        
        return { ok: true, value: user.email };
        
    } catch (error) {
        return {
            ok: false,
            error: new NetworkError('Database connection failed')
        };
    }
}

// ============ HANDLING RESULTS ============
const result = await getUserEmail(id);

if (result.ok) {
    // TypeScript knows result.value is string
    console.log('Email:', result.value);
} else {
    // TypeScript knows result.error is AppError
    console.error(`Error [${result.error.code}]: ${result.error.message}`);
    
    if (result.error.retryable) {
        // Retry logic
    }
}
```

**Why This Works:**
- Errors are typed
- Can't forget to handle errors
- Error codes prevent misspellings
- Retryable flag is explicit in type

---

## Part 7: Type Guards and Type Predicates

Help AI understand how to safely narrow types.

```typescript
// ============ BASIC TYPE GUARD ============
function isUser(value: any): value is User {
    return (
        typeof value === 'object' &&
        value !== null &&
        'id' in value &&
        'email' in value &&
        typeof value.email === 'string'
    );
}

// ============ USAGE ============
function processData(data: unknown) {
    if (isUser(data)) {
        // TypeScript narrows type to User
        console.log(data.email);  // ✓ Known property
    } else {
        // data is not User
        console.log('Data is not a user');
    }
}

// ============ DISCRIMINATED UNIONS ============
interface SuccessResponse<T> {
    type: 'success';
    data: T;
}

interface ErrorResponse {
    type: 'error';
    error: AppError;
}

type Response<T> = SuccessResponse<T> | ErrorResponse;

// Type guard is automatic with discriminated unions
function handleResponse<T>(response: Response<T>) {
    if (response.type === 'success') {
        // TypeScript narrows to SuccessResponse<T>
        console.log(response.data);  // ✓ data property exists
    } else {
        // TypeScript narrows to ErrorResponse
        console.log(response.error);  // ✓ error property exists
    }
}

// ============ GENERIC CONSTRAINTS ============
function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
    // AI knows:
    // - T must be an object
    // - K must be a valid key in T
    // - Return type is T[K] (the type of that property)
    return obj[key];
}

// Usage
const user: User = { id: '1', email: 'test@example.com', ... };
const email = getProperty(user, 'email');  // Type is string
const invalid = getProperty(user, 'invalid');  // ✗ Compiler error
```

---

## Part 8: Testing with Types

Types make testing more reliable.

```typescript
// ============ MOCK DATA WITH TYPES ============
const mockUser: User = {
    id: 'test-1',
    email: 'test@example.com',
    name: 'Test User',
    role: 'student',
    enrolledCourses: []
};

const mockCourse: Course = {
    id: 'cs101',
    code: 'CS101',
    name: 'Intro to CS',
    department: 'Computer Science',
    materials: [],
    createdAt: new Date(),
    updatedAt: new Date()
};

// ============ TESTING REPOSITORIES ============
describe('CourseRepository', () => {
    let repo: IRepository<Course>;
    
    beforeEach(() => {
        repo = new CourseRepository(mockDatabase);
    });
    
    it('should create a course', async () => {
        const created = await repo.create(mockCourse);
        
        expect(created).toEqual(mockCourse);
        expect(created.id).toBeDefined();
        expect(created.createdAt).toBeDefined();
    });
    
    it('should find course by id', async () => {
        // Type safe: result is Course | null
        const found = await repo.findById(mockCourse.id);
        
        if (found) {
            expect(found.code).toBe('CS101');
        }
    });
});

// ============ TYPE ASSERTIONS IN TESTS ============
// Sometimes need to override types for testing
const testUser = mockUser as User;  // Assert testUser is User
const anyValue = someFunction() as unknown;  // Loosen type for testing
```

---

## Quick Reference: Type Patterns

```typescript
// Request/Response pair
interface CreateXRequest { }
interface CreateXResponse { }

// Entity with timestamps
interface Entity extends BaseEntity { }

// Repository pattern
interface IRepository<T extends BaseEntity> { }

// State management
interface State { }
type Action = { type: 'X' } | { type: 'Y' };

// Error handling
abstract class DomainError extends Error { }
type Result<T> = | { ok: true; value: T } | { ok: false; error: DomainError };

// Discriminated unions
type Response<T> = 
    | { type: 'success'; data: T }
    | { type: 'error'; error: string };

// Generics with constraints
function fn<T extends object, K extends keyof T>(obj: T, key: K): T[K] { }

// Type guards
function isX(value: unknown): value is X { }
```

---

## Summary

These patterns work best with AI because:

1. **They're explicit** - AI can't misunderstand the structure
2. **They're reusable** - AI can use the same pattern everywhere
3. **They're testable** - Mock types enforce correct shape
4. **They're safe** - Compiler catches violations

When you give AI these patterns, it will:
- Write consistent code
- Follow the patterns correctly
- Make fewer mistakes
- Produce code that's easier to review
