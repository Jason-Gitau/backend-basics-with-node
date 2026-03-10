# Node.js Code Review: Real-World Scenarios

When reviewing AI-generated code, you'll encounter patterns. Here's how to spot architectural problems.

---

## Scenario 1: The Accidental Blocker

### What AI Might Write:

```javascript
// Express route handler
app.get('/process-image', (req, res) => {
    const imageBuffer = req.body.image; // From upload
    
    // AI thought: "I'll process the image and return it"
    let output = Buffer.alloc(imageBuffer.length);
    for (let i = 0; i < imageBuffer.length; i++) {
        output[i] = applyFilter(imageBuffer[i]); // Complex calculation
    }
    
    res.json({ processed: output });
});
```

### The Problem:
- Image processing is pure CPU work (no I/O)
- For a 10MB image, this loop runs 10 million iterations
- The entire thread freezes during processing
- While this one user's image processes, 1000 other users' requests get no attention

### What You Should Ask:
1. "Is this a CPU-bound operation?" → YES (calculations, not I/O)
2. "Could it block?" → YES (large loop, no async breaks)
3. "How long does it take?" → Estimate: If 1000 iterations/ms, then 10MB = 10,000ms = 10 seconds
4. **Verdict: ARCHITECTURAL PROBLEM** → Need worker threads

### The Fix They Should Have Done:

```javascript
import { Worker } from 'worker_threads';

app.get('/process-image', (req, res) => {
    const imageBuffer = req.body.image;
    
    // Offload to worker, don't block main thread
    const worker = new Worker('./image-processor.js');
    
    worker.on('message', (processedImage) => {
        res.json({ processed: processedImage });
        worker.terminate();
    });
    
    worker.postMessage({ image: imageBuffer });
});
```

### Key Principle:
**CPU work that takes >50ms should use workers. Estimate the timing — if you're unsure, ask.**

---

## Scenario 2: The Forgotten Await

### What AI Might Write:

```javascript
async function saveUserAndSendEmail(userData, email) {
    // AI thought: "I'll do both things"
    
    const user = await db.save(userData); // Takes 100ms
    
    sendEmail(email); // Missing 'await'!
    
    return user;
}

// Usage
app.post('/register', async (req, res) => {
    await saveUserAndSendEmail(req.body, req.body.email);
    res.json({ message: 'User created' });
});
```

### The Problem:
```
Timeline:
0ms:   Start saving to database
0ms:   Start sending email (but don't wait for it)
100ms: Database finishes, return response
       ^ User gets response immediately
       
Meanwhile:
       Email sending still happening...
       But what if it fails? We already sent response!
       What if network is slow? Email never fully sends!
```

### The Architecture Issue:
- "Fire and forget" is dangerous for critical operations
- The email might never reach the user
- No error handling for the email failure
- Race condition: Response sends before email even starts

### What You Should Ask:
1. "Is there an 'await' before every async operation?" → NO
2. "What if the email fails?" → No error handling
3. "Is the response sent before the operation completes?" → YES

**Verdict: CORRECTNESS PROBLEM** (not just architecture, but related)

### The Fix:

```javascript
async function saveUserAndSendEmail(userData, email) {
    const user = await db.save(userData);
    
    // WAIT for email to complete
    await sendEmail(email);
    
    return user;
}

// Or, if email should be async (don't block response):
async function saveUserAndSendEmail(userData, email) {
    const user = await db.save(userData);
    
    // Queue email to be sent (don't wait, don't block)
    // But handle failures separately
    sendEmail(email).catch(err => {
        logger.error('Email failed', err);
        // Maybe send to retry queue?
    });
    
    return user;
}
```

### Key Principle:
**Every async operation needs an 'await' or explicit error handling. If it's missing, the operation runs in the background with no guarantees.**

---

## Scenario 3: The Stream That Doesn't Handle Backpressure

### What AI Might Write:

```javascript
app.get('/download-large-file', (req, res) => {
    const readStream = fs.createReadStream('huge-file.zip'); // 5GB file
    
    readStream.on('data', (chunk) => {
        // AI thought: "Just send each chunk as it comes"
        res.write(chunk);
    });
    
    readStream.on('end', () => {
        res.end();
    });
});
```

### The Problem:
```
Disk: Reading 5GB file rapidly (100MB/s)
         ↓
      Buffer: Chunks pile up
         ↓
Network: Can only send 1MB/s
         ↓
Result: Memory buffer grows uncontrollably
        - Chunks accumulate in memory: 1GB, 2GB, 3GB...
        - Node process crashes or system runs out of RAM
        - Other requests slow to crawl
```

### The Architecture Issue:
- No backpressure handling
- Assumes source and destination have same speed
- Allows unbounded memory growth
- One slow connection can crash the server

### What You Should Ask:
1. "Does the code call `res.write()` without checking return value?" → YES (PROBLEM)
2. "Is there a `pause()` call when buffer fills?" → NO
3. "Is there a `drain` event listener?" → NO

**Verdict: MEMORY AND STABILITY PROBLEM**

### The Fix:

```javascript
app.get('/download-large-file', (req, res) => {
    const readStream = fs.createReadStream('huge-file.zip');
    
    readStream.on('data', (chunk) => {
        // Check if write succeeded
        const canContinue = res.write(chunk);
        
        if (!canContinue) {
            // Network buffer is full, pause reading
            readStream.pause();
        }
    });
    
    // When network drains, resume reading
    res.on('drain', () => {
        readStream.resume();
    });
    
    readStream.on('end', () => {
        res.end();
    });
});

// OR, use pipe (handles all this automatically)
app.get('/download-large-file', (req, res) => {
    fs.createReadStream('huge-file.zip').pipe(res);
    // pipe() handles backpressure automatically!
});
```

### Key Principle:
**Streaming code MUST handle backpressure. Either manually pause/resume or use pipe() which does it automatically.**

---

## Scenario 4: The Synchronous Database Call

### What AI Might Write:

```javascript
const express = require('express');
const Database = require('sqlite3').Database;

const db = new Database(':memory:');

app.get('/user/:id', (req, res) => {
    // AI saw synchronous method and used it
    const user = db.get(`SELECT * FROM users WHERE id = ?`, [req.params.id]);
    
    if (user) {
        res.json(user);
    } else {
        res.status(404).json({ error: 'Not found' });
    }
});
```

### The Problem:
```
User 1 requests /user/1
Node thread: "Let me get that user... blocking for 100ms"
Users 2-10000: "Can't you hear me??"
(They can't make requests until User 1 gets their response)
```

### The Architecture Issue:
- Using synchronous database methods (if they even exist)
- Thread blocked on I/O
- Every user must wait for previous user's DB query
- Throughput drops dramatically

### What You Should Ask:
1. "Are there any 'Sync' method names?" → YES (`db.get`, `db.run`, etc — if synchronous)
2. "Is this I/O operation blocking?" → YES
3. "Is there an async version?" → YES

**Verdict: CONCURRENCY PROBLEM** → Can't handle multiple users efficiently

### The Fix:

```javascript
app.get('/user/:id', async (req, res) => {
    try {
        // Use callback-based or Promise-based API
        const user = await db.query(
            `SELECT * FROM users WHERE id = ?`, 
            [req.params.id]
        );
        
        if (user) {
            res.json(user);
        } else {
            res.status(404).json({ error: 'Not found' });
        }
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

### Key Principle:
**Any method name containing "Sync" is a red flag. Most modern libraries have async alternatives (Promises, callbacks). Use them.**

---

## Scenario 5: The Unhandled Promise Rejection

### What AI Might Write:

```javascript
async function processOrder(orderId) {
    const order = await getOrder(orderId);
    const payment = await processPayment(order.amount);
    
    // AI forgot to handle what happens if this fails
    publishToQueue('order-processed', {
        orderId,
        paymentId: payment.id
    });
    
    return { success: true };
}

// Used in route handler
app.post('/order/:id/checkout', (req, res) => {
    processOrder(req.params.id);  // No await, no error handling
    res.json({ message: 'Processing...' });
});
```

### The Problem:
```
Scenario 1 (Unhandled rejection):
1. Payment fails with error
2. publishToQueue() never executes
3. No one knows the order failed
4. User's response was already sent ("Processing...")
5. User thinks order succeeded but it actually failed
6. Silent corruption of business logic

Scenario 2 (Forgotten await):
1. Order is async
2. Response sends immediately
3. Order processing continues in background
4. If order fails, no one handles it
5. Node might crash with "Unhandled Promise Rejection"
```

### The Architecture Issue:
- No error handling for async operations
- Missing await means fire-and-forget
- Can lead to data corruption or crashes
- Silent failures

### What You Should Ask:
1. "Is this an async function being called?" → YES
2. "Is there an await?" → NO
3. "Is there a .catch() or try/catch?" → NO

**Verdict: RELIABILITY AND ERROR HANDLING PROBLEM**

### The Fix:

```javascript
async function processOrder(orderId) {
    try {
        const order = await getOrder(orderId);
        const payment = await processPayment(order.amount);
        
        await publishToQueue('order-processed', {
            orderId,
            paymentId: payment.id
        });
        
        return { success: true };
    } catch (error) {
        // Handle error properly
        logger.error('Order processing failed', { orderId, error });
        // Maybe queue a retry? Alert admin? 
        throw error;
    }
}

// In route handler
app.post('/order/:id/checkout', async (req, res) => {
    try {
        const result = await processOrder(req.params.id);
        res.json(result);
    } catch (error) {
        res.status(500).json({ 
            error: 'Order processing failed',
            message: error.message 
        });
    }
});
```

### Key Principle:
**Always await async operations and always handle errors. Fire-and-forget is only acceptable for non-critical background tasks with proper error handling.**

---

## Scenario 6: The CPU-Heavy Loop in Middleware

### What AI Might Write:

```javascript
// Middleware that processes every request
app.use((req, res, next) => {
    // AI thought: "I'll validate the data by hashing it"
    let hash = 0;
    for (let i = 0; i < req.body.length; i++) {
        // Simulate complex hash calculation
        for (let j = 0; j < 10000; j++) {
            hash += complexCalculation(req.body[i], j);
        }
    }
    
    req.hash = hash;
    next();
});
```

### The Problem:
```
Request 1 arrives:
  Middleware: Processing (2 seconds of CPU calculation)
  
Request 2 arrives at 0.5s:
  (Still waiting for Request 1's middleware)
  
Request 3 arrives at 1.0s:
  (Still waiting...)
  
Request 1's middleware done (2s):
  Now Request 2's middleware starts
  
Request 2's middleware done (4s):
  Now Request 3's middleware starts

Sequential requests that should take 0.5s each
now take 2+ seconds each because of shared middleware
```

### The Architecture Issue:
- CPU-intensive code in middleware
- Runs on every request
- Blocks all subsequent requests
- Affects overall server responsiveness

### What You Should Ask:
1. "Is there CPU-intensive work in middleware?" → YES
2. "Does it run on every request?" → YES
3. "Could it be offloaded or optimized?" → YES

**Verdict: PERFORMANCE PROBLEM** → Impacts all routes

### The Fix:

```javascript
import { Worker } from 'worker_threads';

// Create a pool of workers for hashing
const hashWorkerPool = [
    new Worker('./hash-worker.js'),
    new Worker('./hash-worker.js'),
    new Worker('./hash-worker.js'),
];

let currentWorker = 0;

app.use((req, res, next) => {
    const worker = hashWorkerPool[currentWorker];
    currentWorker = (currentWorker + 1) % hashWorkerPool.length;
    
    worker.once('message', (hash) => {
        req.hash = hash;
        next();
    });
    
    worker.postMessage(req.body);
});

// OR, cache the hash if it's the same data:
const hashCache = new Map();

app.use((req, res, next) => {
    const key = JSON.stringify(req.body);
    
    if (hashCache.has(key)) {
        req.hash = hashCache.get(key);
        next();
    } else {
        // Do expensive calculation
        const hash = calculateHash(req.body);
        hashCache.set(key, hash);
        req.hash = hash;
        next();
    }
});
```

### Key Principle:
**CPU-intensive work in middleware affects all routes. Use workers or caching to avoid blocking.**

---

## Scenario 7: The Memory Leak Pattern

### What AI Might Write:

```javascript
const cache = {}; // Global cache

app.get('/user/:id', async (req, res) => {
    const userId = req.params.id;
    
    if (cache[userId]) {
        return res.json(cache[userId]);
    }
    
    const user = await db.getUser(userId);
    
    // AI thought: "I'll cache this forever"
    cache[userId] = user;
    
    res.json(user);
});
```

### The Problem:
```
After 1000 requests with different user IDs:
  cache = {
    user1: {...},
    user2: {...},
    user3: {...},
    ... (1000 entries)
  }
  
  Memory used: ~1MB

After 1 million requests:
  Memory used: ~1GB (and growing)
  
After a week of production:
  Memory used: Growing constantly
  Node process uses all available RAM
  Application crashes
```

### The Architecture Issue:
- Unbounded cache growth
- No eviction strategy
- Leaks memory over time
- Can crash under load

### What You Should Ask:
1. "Is data being cached indefinitely?" → YES
2. "Is there an eviction strategy?" → NO (LRU, TTL, etc.)
3. "Could cache grow unboundedly?" → YES

**Verdict: MEMORY LEAK** → Will eventually crash

### The Fix:

```javascript
import LRU from 'lru-cache';

// Create a bounded cache with max 1000 entries
const cache = new LRU({
    max: 1000, // Maximum 1000 entries
    ttl: 1000 * 60 * 5 // Entries expire after 5 minutes
});

app.get('/user/:id', async (req, res) => {
    const userId = req.params.id;
    
    const cached = cache.get(userId);
    if (cached) {
        return res.json(cached);
    }
    
    const user = await db.getUser(userId);
    cache.set(userId, user); // Auto-evicts old entries
    
    res.json(user);
});
```

### Key Principle:
**Unbounded data structures (caches, queues, arrays) will eventually crash your server. Always add limits.**

---

## Quick Reference: Red Flags Checklist

When reviewing AI code, scan for these:

### 🚩 Blocking Red Flags
- [ ] Synchronous file operations (`fs.readFileSync`)
- [ ] CPU-intensive loops in request handlers
- [ ] Long calculations without worker threads
- [ ] Any method name containing "Sync"

### 🚩 Concurrency Red Flags
- [ ] Async function called without `await`
- [ ] No error handling for async operations
- [ ] Missing `.catch()` on promises
- [ ] Fire-and-forget without proper handling

### 🚩 Memory Red Flags
- [ ] Unbounded caches or global data structures
- [ ] No streaming for large files
- [ ] Buffer accumulation without backpressure
- [ ] Data being loaded entirely into memory

### 🚩 Stream Red Flags
- [ ] No backpressure handling in stream code
- [ ] Not using `.pipe()` when available
- [ ] Manual buffer management
- [ ] No pause/resume logic

### 🚩 General Red Flags
- [ ] Promise rejections not handled
- [ ] No try/catch in async functions
- [ ] Mixing callbacks and promises
- [ ] No timeout handling on long operations

---

## Summary: The Questions to Always Ask

1. **Is this blocking?** (CPU work without workers, Sync methods)
2. **Is this async properly?** (await? error handling?)
3. **Does this scale?** (unbounded growth? backpressure?)
4. **Could this crash?** (memory leaks? unhandled errors?)
5. **Does it match the architecture?** (I/O concurrency, CPU parallelism)

If you can answer these questions clearly, you can review any Node.js code effectively.
