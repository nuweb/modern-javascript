# Event Loop & Asynchronous JavaScript

This is the **most critical** topic for senior JavaScript interviews. Interviewers expect deep understanding of how JavaScript handles asynchronous operations despite being single-threaded.

## Table of Contents

1. [The Event Loop Architecture](#1-the-event-loop-architecture)
2. [Execution Order: Microtasks vs Macrotasks](#2-execution-order-microtasks-vs-macrotasks)
3. [Preventing Race Conditions](#3-preventing-race-conditions-in-async-code)
4. [async/await vs Promises](#4-asyncawait-vs-raw-promises-error-handling-differences)
5. [Quick Interview Reference](#quick-interview-reference)

---

## 1. The Event Loop Architecture

JavaScript is **single-threaded** but handles asynchronous operations through the **event loop**. Here's how the components work together:

```
┌─────────────────────────────────────────────────────────────┐
│                        CALL STACK                          │
│              (Executes synchronous code)                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                       EVENT LOOP                           │
│    Checks: Is call stack empty? → Process queues           │
└─────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
┌─────────────────────────┐   ┌─────────────────────────────┐
│    MICROTASK QUEUE      │   │     MACROTASK QUEUE         │
│    (Higher priority)    │   │     (Lower priority)        │
│                         │   │                             │
│  • Promise.then/catch   │   │  • setTimeout               │
│  • queueMicrotask()     │   │  • setInterval              │
│  • MutationObserver     │   │  • setImmediate (Node)      │
│  • process.nextTick*    │   │  • I/O callbacks            │
│                         │   │  • UI rendering events      │
└─────────────────────────┘   └─────────────────────────────┘
```

### Call Stack

The call stack tracks function execution. Functions are **pushed** when called, **popped** when they return:

```javascript
function multiply(a, b) {
  return a * b;
}

function square(n) {
  return multiply(n, n);
}

function printSquare(n) {
  const result = square(n);
  console.log(result);
}

printSquare(4);
```

#### Visual Step-by-Step Execution

```
Step 1: printSquare(4) is called
┌─────────────────┐
│  printSquare    │  ← pushed onto stack
└─────────────────┘

Step 2: printSquare calls square(4)
┌─────────────────┐
│  square         │  ← pushed onto stack
├─────────────────┤
│  printSquare    │
└─────────────────┘

Step 3: square calls multiply(4, 4)
┌─────────────────┐
│  multiply       │  ← pushed onto stack
├─────────────────┤
│  square         │
├─────────────────┤
│  printSquare    │
└─────────────────┘

Step 4: multiply returns 16
┌─────────────────┐
│  square         │  ← multiply popped (returned)
├─────────────────┤
│  printSquare    │
└─────────────────┘

Step 5: square returns 16
┌─────────────────┐
│  printSquare    │  ← square popped (returned)
└─────────────────┘

Step 6: printSquare logs 16 and returns
┌─────────────────┐
│     (empty)     │  ← printSquare popped (returned)
└─────────────────┘
```

#### Reading the Execution Flow

The execution flow can be summarized as:

```
printSquare → square → multiply → (returns) → square → (returns) → printSquare → (returns)
     │           │          │         │           │         │            │           │
     │           │          │         │           │         │            │           └── printSquare finishes
     │           │          │         │           │         │            └── back in printSquare
     │           │          │         │           │         └── square finishes
     │           │          │         │           └── back in square (with result 16)
     │           │          │         └── multiply finishes, returns 16
     │           │          └── multiply starts (top of stack)
     │           └── square starts
     └── printSquare starts
```

#### Key Stack Concepts

| Term | Meaning |
|------|---------|
| **Push** | Function is called → added to top of stack |
| **Pop** | Function returns → removed from top of stack |
| **LIFO** | Last In, First Out — most recent function finishes first |
| **Stack Overflow** | Too many nested calls exceed stack size limit |

### Microtask vs Macrotask Queue

**Critical rule**: After each macrotask completes, the event loop **drains the entire microtask queue** before moving to the next macrotask.

```javascript
console.log('1: Script start');

setTimeout(() => {
  console.log('2: setTimeout (macrotask)');
}, 0);

Promise.resolve()
  .then(() => console.log('3: Promise 1 (microtask)'))
  .then(() => console.log('4: Promise 2 (microtask)'));

queueMicrotask(() => {
  console.log('5: queueMicrotask (microtask)');
});

console.log('6: Script end');
```

**Output:**
```
1: Script start
6: Script end
3: Promise 1 (microtask)
5: queueMicrotask (microtask)
4: Promise 2 (microtask)
2: setTimeout (macrotask)
```

**Why this order?**
1. Synchronous code runs first (`1`, `6`)
2. Script completion is a macrotask — all microtasks drain (`3`, `5`, `4`)
3. Next macrotask executes (`2`)

---

## 2. Execution Order: Microtasks vs Macrotasks

### Complex Example

```javascript
console.log('A: Sync 1');

setTimeout(() => console.log('B: setTimeout 1'), 0);
setTimeout(() => console.log('C: setTimeout 2'), 0);

Promise.resolve()
  .then(() => {
    console.log('D: Promise 1');
    queueMicrotask(() => console.log('E: Nested queueMicrotask'));
  })
  .then(() => console.log('F: Promise 2'));

queueMicrotask(() => {
  console.log('G: queueMicrotask 1');
  Promise.resolve().then(() => console.log('H: Promise inside queueMicrotask'));
});

console.log('I: Sync 2');
```

**Output:**
```
A: Sync 1
I: Sync 2
D: Promise 1
G: queueMicrotask 1
E: Nested queueMicrotask
H: Promise inside queueMicrotask
F: Promise 2
B: setTimeout 1
C: setTimeout 2
```

**Key insight**: Microtasks can spawn more microtasks, and ALL of them run before any macrotask.

### Priority Order (Highest to Lowest)

| Priority | Type | Examples |
|----------|------|----------|
| 1 | Synchronous | Regular code execution |
| 2 | Microtasks | `Promise.then`, `queueMicrotask`, `MutationObserver` |
| 3 | Macrotasks | `setTimeout`, `setInterval`, I/O, UI events |

---

## 3. Preventing Race Conditions in Async Code

### Problem: Race Condition Example

```javascript
let latestRequestId = 0;

// BAD: Race condition - responses may arrive out of order
async function searchUsers(query) {
  const response = await fetch(`/api/users?q=${query}`);
  const users = await response.json();
  renderUsers(users); // May render stale results!
}

// User types: "a" → "ab" → "abc"
// Request for "a" might return AFTER "abc"
```

### Solution 1: Request ID Tracking

```javascript
let latestRequestId = 0;

async function searchUsers(query) {
  const requestId = ++latestRequestId;
  
  const response = await fetch(`/api/users?q=${query}`);
  const users = await response.json();
  
  // Only render if this is still the latest request
  if (requestId === latestRequestId) {
    renderUsers(users);
  }
}
```

### Solution 2: AbortController (Preferred)

```javascript
let abortController = null;

async function searchUsers(query) {
  // Cancel previous request
  if (abortController) {
    abortController.abort();
  }
  
  abortController = new AbortController();
  
  try {
    const response = await fetch(`/api/users?q=${query}`, {
      signal: abortController.signal
    });
    const users = await response.json();
    renderUsers(users);
  } catch (error) {
    if (error.name !== 'AbortError') {
      throw error; // Re-throw non-abort errors
    }
    // Aborted requests are expected, don't treat as error
  }
}
```

### Solution 3: Mutex/Lock Pattern

```javascript
function createMutex() {
  let locked = false;
  const queue = [];
  
  return {
    async acquire() {
      if (locked) {
        await new Promise(resolve => queue.push(resolve));
      }
      locked = true;
    },
    release() {
      if (queue.length > 0) {
        queue.shift()(); // Resolve next waiting promise
      } else {
        locked = false;
      }
    }
  };
}

const mutex = createMutex();

async function criticalSection() {
  await mutex.acquire();
  try {
    // Only one execution at a time
    await performSensitiveOperation();
  } finally {
    mutex.release();
  }
}
```

### Solution 4: Debouncing for User Input

```javascript
function debounce(fn, delay) {
  let timeoutId;
  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

const debouncedSearch = debounce(searchUsers, 300);
input.addEventListener('input', (e) => debouncedSearch(e.target.value));
```

---

## 4. async/await vs Raw Promises: Error Handling Differences

### Raw Promises

```javascript
function fetchUserData(userId) {
  return fetch(`/api/users/${userId}`)
    .then(response => {
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      return response.json();
    })
    .then(user => {
      return fetch(`/api/posts?userId=${user.id}`);
    })
    .then(response => response.json())
    .catch(error => {
      // Catches ANY error in the chain
      console.error('Error:', error);
      return { user: null, posts: [] }; // Fallback
    })
    .finally(() => {
      hideLoadingSpinner();
    });
}
```

**Promise characteristics:**
- Errors propagate down the chain until caught
- Single `.catch()` handles all upstream errors
- Can't use `try/catch`
- Harder to do conditional logic mid-chain

### async/await

```javascript
async function fetchUserData(userId) {
  try {
    const userResponse = await fetch(`/api/users/${userId}`);
    if (!userResponse.ok) {
      throw new Error(`HTTP ${userResponse.status}`);
    }
    const user = await userResponse.json();
    
    const postsResponse = await fetch(`/api/posts?userId=${user.id}`);
    const posts = await postsResponse.json();
    
    return { user, posts };
  } catch (error) {
    // Catches ANY error above
    console.error('Error:', error);
    return { user: null, posts: [] };
  } finally {
    hideLoadingSpinner();
  }
}
```

### Key Differences

| Aspect | Raw Promises | async/await |
|--------|--------------|-------------|
| Syntax | Chain-based | Sequential, like sync code |
| Error handling | `.catch()` | `try/catch` |
| Debugging | Harder stack traces | Clear stack traces |
| Conditional logic | Awkward nesting | Natural `if/else` |
| Parallel execution | `Promise.all()` | Same, but clearer |

### Granular Error Handling with async/await

```javascript
async function processOrder(orderId) {
  let user, inventory, payment;
  
  // Handle each operation's errors separately
  try {
    user = await fetchUser(orderId);
  } catch (error) {
    throw new Error(`User fetch failed: ${error.message}`);
  }
  
  try {
    inventory = await checkInventory(orderId);
  } catch (error) {
    // Graceful degradation
    console.warn('Inventory check failed, proceeding anyway');
    inventory = { available: true }; // Assume available
  }
  
  try {
    payment = await processPayment(user, orderId);
  } catch (error) {
    // Critical error - must handle
    await rollbackOrder(orderId);
    throw new Error(`Payment failed: ${error.message}`);
  }
  
  return { user, inventory, payment };
}
```

### Common Pitfall: Unhandled Rejections

```javascript
// BAD: Unhandled rejection
async function bad() {
  const data = await fetchData(); // If this throws, unhandled!
}
bad(); // No .catch(), no try/catch wrapping the call

// GOOD: Always handle
async function good() {
  try {
    const data = await fetchData();
  } catch (error) {
    handleError(error);
  }
}

// OR at call site
bad().catch(handleError);
```

### Parallel Execution Comparison

```javascript
// SEQUENTIAL (slow) - each await blocks
async function sequential() {
  const user = await fetchUser();      // 200ms
  const posts = await fetchPosts();    // 300ms
  const comments = await fetchComments(); // 250ms
  // Total: ~750ms
}

// PARALLEL (fast) - all requests at once
async function parallel() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),      // 200ms ─┐
    fetchPosts(),     // 300ms ─┼── All run simultaneously
    fetchComments()   // 250ms ─┘
  ]);
  // Total: ~300ms (slowest request)
}

// PARALLEL with individual error handling
async function parallelWithHandling() {
  const results = await Promise.allSettled([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ]);
  
  // results[n].status === 'fulfilled' | 'rejected'
  // results[n].value (if fulfilled) or results[n].reason (if rejected)
}
```

---

## Quick Interview Reference

### "Explain the event loop in 30 seconds"

> "JavaScript has a single-threaded call stack. When async operations complete, their callbacks go into queues. The event loop continuously checks if the stack is empty, then processes all microtasks (Promises, queueMicrotask), then one macrotask (setTimeout, I/O), then microtasks again, repeating this cycle."

### "Why do Promises execute before setTimeout(0)?"

> "Promise callbacks go to the microtask queue, which has higher priority. The event loop drains all microtasks before processing the next macrotask."

### "What's the difference between microtasks and macrotasks?"

> "Microtasks (Promises, queueMicrotask) run immediately after the current script and before any macrotasks. Macrotasks (setTimeout, I/O) run one at a time, with the microtask queue draining completely between each macrotask."

### "How would you prevent a race condition in a search autocomplete?"

> "Use AbortController to cancel in-flight requests when a new search starts, or track request IDs and only render results from the most recent request."

---

## Practice Questions

1. What will this code output and why?
```javascript
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('sync');
```

2. How does `process.nextTick` differ from `queueMicrotask` in Node.js?

3. Write a function that limits concurrent API requests to N at a time.

4. Explain why this creates an infinite loop:
```javascript
function bad() {
  Promise.resolve().then(bad);
}
bad();
```
