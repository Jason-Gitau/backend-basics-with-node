# Node.js Architecture: Visual Decision Trees & Diagrams

## Diagram 1: The Event Loop Cycle

```
┌─────────────────────────────────────────────────┐
│         START: JavaScript Execution             │
│      (Main code, module initialization)         │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│    TIMERS PHASE                                 │
│  Execute setTimeout/setInterval callbacks       │
│  (Callbacks from timers that have expired)      │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│    I/O PHASE                                    │
│  Execute file system, network, DB callbacks     │
│  (Delayed callbacks from I/O operations)        │
│  This is where most app work happens            │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│    MICROTASKS (Between every phase)             │
│  Process .then(), .catch(), async operations    │
│  Run ALL microtasks before moving on            │
│  (Higher priority than regular callbacks)       │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│    CHECK PHASE                                  │
│  Execute setImmediate() callbacks               │
│  (These run AFTER I/O, not before timers)       │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│    CLOSE PHASE                                  │
│  Handle connection closures and cleanup         │
└────────────────┬────────────────────────────────┘
                 │
          Are there more │
          tasks pending?  │
                 │
        Yes ─────┴─────── No
         │               │
         ▼               ▼
      [Loop back]   [Exit process]
       (TIMERS)
```

**What this means:** The event loop continuously circles through these phases. In each phase, it processes all queued callbacks. Only when a phase is empty does it move to the next.

---

## Diagram 2: Blocking vs Non-Blocking

### Blocking Code (❌ WRONG)

```
Time  Thread Status                  Network
──────────────────────────────────────────────
0ms   ┌──────────────────────────────┐
      │ Req1: Process                │
      │ (CPU intensive loop)         │
      └──────────────────────────────┘
            Req2 arrives ← (can't accept, blocked)
            Req3 arrives ← (can't accept, blocked)

2000ms ┌──────────┐
       │ Req1 Done│  Send response
       └──────────┘
            │
            ▼
       ┌──────────────────────────────┐
       │ Req2: Process                │
       │ (finally starting!)          │
       └──────────────────────────────┘
       
4000ms ┌──────────┐
       │ Req2 Done│  Send response
       └──────────┘

All users except Req1 wait 2+ seconds. Terrible!
```

### Non-Blocking Code (✅ CORRECT)

```
Time  Thread Status              OS Work        Network
──────────────────────────────────────────────────────
0ms   ┌──────────┐               ┌──────────┐
      │ Req1:    │  Submits query│ Database │
      │ Delegate │────────────→  │ processing
      │ DB work  │               │          │
      └──────────┘               └──────────┘
            │
            ▼
       ┌──────────┐
       │ Req2:    │
       │ Delegate │  Database still processing
       │ DB work  │
       └──────────┘
            │
            ▼
       ┌──────────┐
       │ Req3:    │
       │ Process  │  Database still processing
       └──────────┘
            │
            ▼
       ┌──────────┐
       │ Req4:    │
       │ Delegate │
       │ File I/O │
       └──────────┘

100ms  (Thread waits here, can still process new requests)

100ms         Req1 complete ←  ┘
       ┌──────────┐
       │ Execute  │  Callback runs
       │ Req1's   │
       │ callback │
       └──────────┘

All users respond in ~100ms. Excellent!
```

---

## Diagram 3: CPU-Bound vs I/O-Bound

```
                          CPU-Bound Work
                          (Calculations)
                          │
                          ▼
        ┌─────────────────────────────────────────┐
        │  Can be offloaded to Worker Threads      │
        │  Does NOT benefit from non-blocking      │
        │  MUST use Worker for responsiveness      │
        └─────────────────────────────────────────┘


                          I/O-Bound Work
                          (Network, File, DB)
                          │
                          ▼
        ┌─────────────────────────────────────────┐
        │  Naturally concurrent with Event Loop    │
        │  Non-blocking callbacks work perfectly   │
        │  Does NOT need Worker Threads           │
        └─────────────────────────────────────────┘


           ┌─────────────────────────┐
           │    Data Processing      │
           │  (Image, Video, Crypto) │
           │                         │
           │ Both CPU AND I/O        │
           │                         │
           │ CPU work on workers,    │
           │ I/O operations stay     │
           │ on main thread          │
           └─────────────────────────┘
```

---

## Diagram 4: Stream Processing with Backpressure

### Without Proper Backpressure Handling (❌ PROBLEM)

```
Source (Fast)        Destination (Slow)
┌────────────────┐   ┌────────────────┐
│ Reading from   │   │ Writing to     │
│ disk: 100MB/s  │──→│ network: 1MB/s │
└────────────────┘   └────────────────┘
                          │
                          ▼
                  ┌──────────────────┐
                  │ Chunk Buffer:    │
                  │ Gets huge!       │
                  │ Memory explodes! │
                  └──────────────────┘
```

### With Proper Backpressure Handling (✅ CORRECT)

```
Source (Fast)        Destination (Slow)
┌────────────────┐   ┌────────────────┐
│ Read chunk     │   │ Network full?  │
│ from disk      │──→│ (backpressure)│
└────────────────┘   └────────────────┘
                          │
                          ▼ If network returns false:
                    ┌──────────────────┐
                    │ PAUSE reading    │
                    │ Buffer stays     │
                    │ manageable       │
                    └──────────────────┘
                          │
                    Network drains...
                          │
                          ▼
                    ┌──────────────────┐
                    │ 'drain' event    │
                    │ RESUME reading   │
                    └──────────────────┘
```

---

## Diagram 5: Worker Thread Architecture

### Single-Threaded (Blocking)

```
                    Main Thread
                    ───────────
                    
    Req1: Heavy      User blocked!
    calculation ←────(waiting for result)
    ════════════ (2 seconds)
    
    Req2 arrives
    (can't handle yet)
    
    Req1 done
    
    Req2: Heavy
    calculation ────→User 2 blocked!
    ════════════
```

### With Worker Threads (Parallel)

```
    Main Thread            Worker Pool
    ───────────            ───────────
    
    Req1 → ┐
           ├─→ [Worker 1] Calculating...
    Req2 → ┐       │
           ├─→ [Worker 2] Calculating...
    Req3 → ┐       │
           ├─→ [Worker 3] Calculating...
    Req4 ──┘       │
    (accepts)      │
    
    Req5 ──┐       ▼
           → [Worker 1 free] ← [Result sent to Req1]
    
All users get responses quickly!
Requests don't pile up!
```

---

## Diagram 6: Concurrency vs Parallelism

### Concurrency (Node.js Default)

```
         Timeline
         ────────
         
0ms  ┌─ Req1 ──────┐ (waiting for I/O)
     │             │
1ms  │ Req2 ────┐  │
     │          │  │ (still waiting)
2ms  │ Req3 ──┐ │  │
     │        │ │  │
3ms  └────────┼─┼──┘ (I/O completes)
              │ │
              ▼ ▼  
         ┌────────────┐
         │ Run Req1's │
         │ callback   │
         └────────────┘
              │
              ▼
         ┌────────────┐
         │ Run Req2's │
         │ callback   │
         └────────────┘


One thread, many interleaved tasks
```

### Parallelism (Worker Threads)

```
         CPU Core 1      CPU Core 2       CPU Core 3
         ──────────      ──────────       ──────────
         
0ms      Req1 job     │  Req2 job      │  Req3 job
         processing   │  processing    │  processing
         
1ms      ││           │  ││            │  ││
         ││  (all     │  ││  (all      │  ││  (all
         ││   running │  ││   running  │  ││   running
         ││   truly   │  ││   truly    │  ││   truly
         ││   parallel)   ││  parallel) │  ││   parallel)
         
3ms      ▼            │  ▼             │  ▼
         Results ready simultaneously


Multiple threads, actual simultaneous execution
```

---

## Diagram 7: Code Review Decision Tree

```
                    ┌─ Found slow code?
                    │
                    ▼
          Is it I/O related?
          (DB, API, File, Network)
                    │
          ┌─────────┴─────────┐
          │                   │
         YES                 NO
          │                   │
          ▼                   ▼
    Using callbacks/  Is it a loop or
    promises/async?   calculation?
          │                   │
          ▼                   ▼
         YES                 YES
          │                   │
          ▼                   ▼
    ✅ Correct         Will it take
    Pattern           >50ms?
    
                           │
                    ┌──────┴──────┐
                    │             │
                   YES            NO
                    │             │
                    ▼             ▼
              Using Worker    ✅ Probably OK
              Threads?        (depends on
                   │          load)
                   │
              ┌────┴────┐
              │         │
             YES        NO
              │         │
              ▼         ▼
         ✅ Good    ❌ BUG!
                    (Blocks
                     thread)


     Any 'Sync' methods used
     in request handler?
     (readFileSync, etc)
              │
          ┌───┴───┐
          │       │
         YES     NO
          │       │
          ▼       ▼
         ❌ BUG  ✅ Good
         (blocks)
```

---

## Diagram 8: The Memory Flow with Streams

### Naive Approach (Load Everything)

```
File on Disk (1GB)
        │
        ▼
    ┌─────────┐
    │ Node    │ Loads entire file into memory
    │ Memory  │ (1GB RAM consumed!)
    └─────────┘
        │
        ▼
    Process  ← All 1GB in memory
        │
        ▼
    Send to user
        
    Problem: Memory spike, GC pauses, crashes under load
```

### Stream Approach (Chunk by Chunk)

```
File on Disk (1GB)
        │
    ┌───┴───┬────┬────┬────┬────┐
    │ Ch1  │Ch2 │Ch3 │Ch4 │... │
    │ 64KB │64KB│64KB│64KB│    │
    └───┬───┴────┴────┴────┴────┘
        │
        ▼
    ┌─────────┐
    │ Stream  │ Only 64KB in memory at a time
    │ Buffer  │
    └─────────┘
        │
        ▼
    Process chunk → Send → Request next chunk
        │
        ▼
    Repeat
    
    Benefit: Constant memory usage, no GC pauses, infinite file support
```

---

## Diagram 9: Callback Hell vs Promise Flow

### Callback Hell (Nested, Hard to Follow)

```
getUser(userId, function(err, user) {
    if (err) {
        getBackupUser(userId, function(err2, backupUser) {
            if (err2) {
                handleError(err2);
            } else {
                processUser(backupUser, function(err3, result) {
                    if (err3) {
                        handleError(err3);
                    } else {
                        sendResponse(result, function(err4) {
                            if (err4) {
                                log(err4);
                            }
                        });
                    }
                });
            }
        });
    } else {
        processUser(user, function(err3, result) {
            // ... more nesting
        });
    }
});

Visual:
┌─────────────────────────────────────┐
│ getUser                             │
│  ┌──────────────────────────────┐   │
│  │ getBackupUser                │   │
│  │  ┌──────────────────────────┐│   │
│  │  │ processUser              ││   │
│  │  │  ┌──────────────────────┐││   │
│  │  │  │ sendResponse         │││   │
│  │  │  └──────────────────────┘││   │
│  │  └──────────────────────────┘│   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘

"Pyramid of Doom" - Hard to read, hard to debug
```

### Promise Chain (Linear, Clear Flow)

```
getUser(userId)
    .then(user => processUser(user))
    .then(result => sendResponse(result))
    .catch(err => handleError(err));

Visual:
┌──────────────┐
│  getUser     │
└────┬─────────┘
     │
     ▼
┌──────────────┐
│ processUser  │
└────┬─────────┘
     │
     ▼
┌──────────────┐
│ sendResponse │
└────┬─────────┘
     │
     ▼
┌──────────────┐
│ handleError  │
└──────────────┘

Left-to-right, top-to-bottom, much clearer!
```

---

## Diagram 10: The Mental Model - Restaurant Analogy

```
Scenario: Restaurant with 1 waiter, 100 customers

THE PROBLEM (Blocking Model):
────────────────────────────
Waiter: "I'll take an order, wait for food, serve it, repeat."

Customer 1: Orders steak
Waiter: (goes to kitchen, stands there for 30 min) [BLOCKED]
Customers 2-100: Waiting outside, can't even order

Customer 1: Food arrives
Waiter: Serves food, clears table
Customer 2: NOW can order

This waiter can serve maybe 2-3 customers per hour.


THE SOLUTION (Non-Blocking Model):
──────────────────────────────────
Waiter: "I'll take orders from EVERYONE, then coordinate with the kitchen."

Customer 1: Orders steak
Waiter: Shouts order to kitchen, walks immediately to next customer [NON-BLOCKING]
Customer 2: Orders fish
Waiter: Shouts order to kitchen, next customer [FAST]
Customers 3-100: Quickly order while waiter takes orders
Kitchen: (preparing 100 meals in parallel)

Customer 1's order ready: Waiter delivers
Customer 2's order ready: Waiter delivers
Customer 15's order ready: Waiter delivers

This waiter can serve 100+ customers per hour!

The key: Waiter never *waits*. Kitchen does the waiting (I/O).
```

---

## Summary Decision: Should I Use...?

```
❓ Callbacks
   → Use when: Simple async operations
   → Avoid: Multiple nested async operations

❓ Promises
   → Use when: Chaining multiple operations
   → Good for: Error handling across chains

❓ Async/Await
   → Use when: Complex async flows
   → Read like: Synchronous code
   → Best for: Most modern code

❓ Worker Threads
   → Use when: CPU-intensive work >50ms
   → NOT for: I/O operations
   → Goal: Keep main thread responsive

❓ Streams
   → Use when: Handling large data
   → NOT for: Small data sets
   → Always: Handle backpressure

❓ setTimeout vs setImmediate
   → setTimeout: "Do this later" (after I/O phase)
   → setImmediate: "Do this sooner" (after I/O phase, before next timers)
   → Confusing! Usually doesn't matter, avoid unless optimization is needed
```
