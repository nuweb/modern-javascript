# Closures & Scope

Closures are one of the most powerful and frequently misunderstood concepts in JavaScript. Mastering closures is essential for senior-level interviews and writing elegant, maintainable code.

## Table of Contents

1. [What is a Closure?](#1-what-is-a-closure)
2. [Lexical Scoping](#2-lexical-scoping)
3. [Real-World Applications](#3-real-world-applications)
4. [Memory Management](#4-how-closures-affect-memory-management)
5. [Common Pitfalls](#5-common-closure-pitfalls)
6. [Quick Interview Reference](#quick-interview-reference)

---

## 1. What is a Closure?

A **closure** is a function that "remembers" the variables from its outer (enclosing) scope, even after the outer function has finished executing.

### Basic Example

```javascript
function createGreeter(greeting) {
  // `greeting` is in the outer scope
  
  return function(name) {
    // This inner function "closes over" `greeting`
    return `${greeting}, ${name}!`;
  };
}

const sayHello = createGreeter('Hello');
const sayHola = createGreeter('Hola');

console.log(sayHello('Alice')); // "Hello, Alice!"
console.log(sayHola('Bob'));    // "Hola, Bob!"

// `createGreeter` has finished executing, but the returned
// functions still have access to their respective `greeting` values
```

### Visualizing the Closure

```
┌─────────────────────────────────────────────────────────┐
│  createGreeter('Hello') execution context              │
│  ┌───────────────────────────────────────────────────┐ │
│  │  greeting = 'Hello'                               │ │
│  │                                                   │ │
│  │  ┌─────────────────────────────────────────────┐ │ │
│  │  │  returned function (closure)                │ │ │
│  │  │  - has reference to `greeting`              │ │ │
│  │  │  - "closes over" the outer scope            │ │ │
│  │  └─────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
         │
         ▼ After createGreeter returns...
         
┌─────────────────────────────────────────────────────────┐
│  sayHello (the closure)                                 │
│  ┌───────────────────────────────────────────────────┐ │
│  │  [[Environment]] → { greeting: 'Hello' }         │ │
│  │  (keeps the outer scope alive)                   │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### The Three Components of a Closure

1. **A function** defined inside another function
2. **Free variables** from the outer scope that the inner function references
3. **The environment** where those variables were defined (kept alive)

```javascript
function outer() {
  let count = 0;  // Free variable
  
  function inner() {  // Inner function
    count++;  // References free variable
    return count;
  }
  
  return inner;  // Returns the closure
}

const counter = outer();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
// `count` persists between calls!
```

---

## 2. Lexical Scoping

**Lexical scoping** (also called static scoping) means that a function's scope is determined by where it's **written** in the code, not where it's **called**.

### Understanding Scope Chain

```javascript
const global = 'I am global';

function outer() {
  const outerVar = 'I am outer';
  
  function middle() {
    const middleVar = 'I am middle';
    
    function inner() {
      const innerVar = 'I am inner';
      
      // inner can access ALL outer scopes
      console.log(innerVar);   // ✓ Own scope
      console.log(middleVar);  // ✓ Parent scope
      console.log(outerVar);   // ✓ Grandparent scope
      console.log(global);     // ✓ Global scope
    }
    
    inner();
  }
  
  middle();
}

outer();
```

### Scope Chain Visualization

```
┌─────────────────────────────────────────────────────────────┐
│  GLOBAL SCOPE                                               │
│  └── global = 'I am global'                                 │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  outer() SCOPE                                      │   │
│  │  └── outerVar = 'I am outer'                        │   │
│  │      │                                              │   │
│  │      ▼                                              │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  middle() SCOPE                             │   │   │
│  │  │  └── middleVar = 'I am middle'              │   │   │
│  │  │      │                                      │   │   │
│  │  │      ▼                                      │   │   │
│  │  │  ┌─────────────────────────────────────┐   │   │   │
│  │  │  │  inner() SCOPE                      │   │   │   │
│  │  │  │  └── innerVar = 'I am inner'        │   │   │   │
│  │  │  │                                     │   │   │   │
│  │  │  │  Scope chain: inner → middle →      │   │   │   │
│  │  │  │               outer → global        │   │   │   │
│  │  │  └─────────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Lexical vs Dynamic Scoping

JavaScript uses **lexical scoping**. This matters when functions are passed around:

```javascript
const value = 'global';

function showValue() {
  console.log(value);  // Where is `value` looked up?
}

function wrapper() {
  const value = 'wrapper';  // This does NOT affect showValue
  showValue();
}

wrapper();  // Prints "global", not "wrapper"

// Why? Because showValue's scope was determined when it was DEFINED,
// not when it was CALLED. It looks for `value` in its lexical environment.
```

### Implications in Large Codebases

#### 1. Module Pattern & Encapsulation

```javascript
// Each module has its own scope - variables don't leak
const UserModule = (function() {
  // Private - not accessible outside
  let users = [];
  const API_KEY = 'secret123';
  
  // Public interface
  return {
    addUser(user) {
      users.push(user);
    },
    getUsers() {
      return [...users];  // Return copy, not reference
    }
  };
})();

UserModule.addUser({ name: 'Alice' });
console.log(UserModule.getUsers());  // [{ name: 'Alice' }]
console.log(UserModule.users);       // undefined (private!)
console.log(UserModule.API_KEY);     // undefined (private!)
```

#### 2. Avoiding Global Namespace Pollution

```javascript
// BAD: Pollutes global scope
var config = {};
var utils = {};
var helpers = {};
// Risk of naming collisions with other scripts!

// GOOD: Use closures to create namespaces
const MyApp = (function() {
  const config = {};
  const utils = {};
  const helpers = {};
  
  return {
    init() { /* ... */ },
    destroy() { /* ... */ }
  };
})();
```

#### 3. Predictable Variable Resolution

```javascript
// In large codebases, lexical scoping makes code predictable
// You can always trace where a variable comes from by looking UP in the code

function createAPI(baseURL) {
  // Anyone reading `request` knows `baseURL` comes from createAPI's parameter
  async function request(endpoint) {
    return fetch(`${baseURL}${endpoint}`);
  }
  
  return {
    getUsers: () => request('/users'),
    getPosts: () => request('/posts')
  };
}
```

---

## 3. Real-World Applications

### 1. Data Privacy / Encapsulation

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;  // Private!
  
  return {
    deposit(amount) {
      if (amount > 0) {
        balance += amount;
        return balance;
      }
      throw new Error('Invalid amount');
    },
    withdraw(amount) {
      if (amount > 0 && amount <= balance) {
        balance -= amount;
        return balance;
      }
      throw new Error('Invalid withdrawal');
    },
    getBalance() {
      return balance;
    }
  };
}

const account = createBankAccount(100);
account.deposit(50);   // 150
account.withdraw(30);  // 120
account.getBalance();  // 120

// Cannot directly access or modify balance:
console.log(account.balance);  // undefined
account.balance = 1000000;     // Does nothing
account.getBalance();          // Still 120
```

### 2. Function Factories

```javascript
// Create specialized functions
function createMultiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);
const tenX = createMultiplier(10);

console.log(double(5));  // 10
console.log(triple(5));  // 15
console.log(tenX(5));    // 50
```

### 3. Memoization / Caching

```javascript
function memoize(fn) {
  const cache = {};  // Closed over by the returned function
  
  return function(...args) {
    const key = JSON.stringify(args);
    
    if (key in cache) {
      console.log('Cache hit!');
      return cache[key];
    }
    
    console.log('Computing...');
    const result = fn.apply(this, args);
    cache[key] = result;
    return result;
  };
}

function expensiveCalculation(n) {
  // Simulate expensive work
  let result = 0;
  for (let i = 0; i < n * 1000000; i++) {
    result += i;
  }
  return result;
}

const memoizedCalc = memoize(expensiveCalculation);

memoizedCalc(10);  // Computing... (slow)
memoizedCalc(10);  // Cache hit! (instant)
memoizedCalc(20);  // Computing... (slow)
memoizedCalc(10);  // Cache hit! (instant)
```

### 4. Event Handlers with State

```javascript
function createToggleButton(elementId) {
  let isOn = false;  // Private state
  const button = document.getElementById(elementId);
  
  button.addEventListener('click', function() {
    isOn = !isOn;  // Closure accesses `isOn`
    button.textContent = isOn ? 'ON' : 'OFF';
    button.classList.toggle('active', isOn);
  });
  
  // Public API to check state
  return {
    getState: () => isOn,
    setState: (state) => {
      isOn = state;
      button.textContent = isOn ? 'ON' : 'OFF';
      button.classList.toggle('active', isOn);
    }
  };
}

const toggle = createToggleButton('myButton');
```

### 5. Partial Application / Currying

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
}

function add(a, b, c) {
  return a + b + c;
}

const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3));     // 6
console.log(curriedAdd(1, 2)(3));     // 6
console.log(curriedAdd(1)(2, 3));     // 6
console.log(curriedAdd(1, 2, 3));     // 6

// Practical example: creating specialized loggers
const log = curry((level, timestamp, message) => {
  console.log(`[${level}] ${timestamp}: ${message}`);
});

const errorLog = log('ERROR');
const errorLogNow = errorLog(new Date().toISOString());

errorLogNow('Something went wrong');  // [ERROR] 2025-02-02T...: Something went wrong
```

### 6. Debounce & Throttle

```javascript
function debounce(fn, delay) {
  let timeoutId;  // Closed over
  
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

function throttle(fn, limit) {
  let inThrottle;  // Closed over
  
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Usage
const debouncedSearch = debounce(searchAPI, 300);
const throttledScroll = throttle(updateScrollPosition, 100);
```

---

## 4. How Closures Affect Memory Management

### Memory Retention

Closures keep their outer scope variables alive in memory. This is powerful but can cause memory issues if not handled carefully.

```javascript
function createLargeDataProcessor() {
  const hugeArray = new Array(1000000).fill('data');  // ~8MB
  
  return function process(index) {
    return hugeArray[index];
  };
}

const processor = createLargeDataProcessor();
// `hugeArray` stays in memory as long as `processor` exists!
// Even if you only need one element, the entire array is retained.

processor = null;  // Now hugeArray can be garbage collected
```

### Visualizing Memory Retention

```
┌─────────────────────────────────────────────────────────────┐
│  Closure Memory Model                                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  processor (function)                                       │
│       │                                                     │
│       │  [[Environment]]                                    │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Closure Scope (kept alive)                         │   │
│  │  ┌───────────────────────────────────────────────┐ │   │
│  │  │  hugeArray: [data, data, data, ... x1000000]  │ │   │
│  │  │  (8MB - cannot be garbage collected)          │ │   │
│  │  └───────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  When processor = null:                                     │
│  - No references to closure                                 │
│  - Environment can be garbage collected                     │
│  - hugeArray memory is freed                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Memory Leak Patterns

#### 1. Accidental Global References

```javascript
function leakyFunction() {
  const largeData = new Array(1000000).fill('x');
  
  // Accidentally creates global reference
  globalCallback = function() {
    console.log(largeData.length);
  };
}

leakyFunction();
// `largeData` is never freed because `globalCallback` references it
```

#### 2. Forgotten Event Listeners

```javascript
function setupHandler() {
  const hugeData = loadHugeDataset();
  
  document.getElementById('button').addEventListener('click', function() {
    processData(hugeData);
  });
  
  // Even if setupHandler completes, hugeData stays in memory
  // because the event listener's closure references it
}

// Fix: Remove listener when done, or use WeakRef
function setupHandlerFixed() {
  const hugeData = loadHugeDataset();
  const button = document.getElementById('button');
  
  function handler() {
    processData(hugeData);
  }
  
  button.addEventListener('click', handler);
  
  // Return cleanup function
  return function cleanup() {
    button.removeEventListener('click', handler);
  };
}
```

#### 3. Circular References in Closures

```javascript
function createCircularLeak() {
  const obj = {};
  
  obj.closure = function() {
    // References obj, which references this closure
    console.log(obj);
  };
  
  return obj;
}

// Modern JS engines handle this, but it can cause issues
// with older engines or when DOM elements are involved
```

### Best Practices for Memory

```javascript
// 1. Only capture what you need
function goodClosure() {
  const hugeData = loadHugeDataset();
  const summary = computeSummary(hugeData);  // Extract what's needed
  
  return function() {
    return summary;  // Only closes over `summary`, not `hugeData`
  };
}

// 2. Nullify references when done
function managedClosure() {
  let data = loadData();
  
  return {
    process() {
      if (data) {
        const result = transform(data);
        return result;
      }
      throw new Error('Already disposed');
    },
    dispose() {
      data = null;  // Allow garbage collection
    }
  };
}

// 3. Use WeakMap for object keys (allows GC when key is unreferenced)
const cache = new WeakMap();

function memoizeByObject(fn) {
  return function(obj) {
    if (cache.has(obj)) {
      return cache.get(obj);
    }
    const result = fn(obj);
    cache.set(obj, result);
    return result;
  };
}
```

---

## 5. Common Closure Pitfalls

### The Classic Loop Problem (var vs let)

```javascript
// PROBLEM: var is function-scoped, not block-scoped
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 100);
}
// Output: 3, 3, 3
// Why? All callbacks share the SAME `i`, which is 3 after the loop

// SOLUTION 1: Use let (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 100);
}
// Output: 0, 1, 2
// Each iteration gets its own `i`

// SOLUTION 2: Create new scope with IIFE
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(function() {
      console.log(j);
    }, 100);
  })(i);
}
// Output: 0, 1, 2

// SOLUTION 3: Use bind
for (var i = 0; i < 3; i++) {
  setTimeout(function(j) {
    console.log(j);
  }.bind(null, i), 100);
}
// Output: 0, 1, 2
```

### Visual Explanation: var vs let in Loops

```
WITH var:
┌─────────────────────────────────────────────────┐
│  Function Scope                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  i = 3  (shared by all callbacks)         │ │
│  │         ↑      ↑      ↑                   │ │
│  │         │      │      │                   │ │
│  │    callback  callback  callback           │ │
│  │    (prints 3) (prints 3) (prints 3)       │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘

WITH let:
┌─────────────────────────────────────────────────┐
│  Loop - Each iteration has its own scope        │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │ i = 0       │ │ i = 1       │ │ i = 2     │ │
│  │     ↑       │ │     ↑       │ │     ↑     │ │
│  │ callback    │ │ callback    │ │ callback  │ │
│  │ (prints 0)  │ │ (prints 1)  │ │ (prints 2)│ │
│  └─────────────┘ └─────────────┘ └───────────┘ │
└─────────────────────────────────────────────────┘
```

### Pitfall: Unintended Shared State

```javascript
// PROBLEM: All buttons share the same `count`
function setupButtons(buttonIds) {
  let count = 0;  // Shared!
  
  buttonIds.forEach(id => {
    document.getElementById(id).addEventListener('click', () => {
      count++;
      console.log(`Button clicked, count: ${count}`);
    });
  });
}

setupButtons(['btn1', 'btn2', 'btn3']);
// Clicking any button increments the same counter

// SOLUTION: Create separate state for each
function setupButtonsSeparate(buttonIds) {
  buttonIds.forEach(id => {
    let count = 0;  // Each button gets its own count
    
    document.getElementById(id).addEventListener('click', () => {
      count++;
      console.log(`${id} clicked, count: ${count}`);
    });
  });
}
```

### Pitfall: this Binding Confusion

```javascript
const obj = {
  name: 'MyObject',
  
  // PROBLEM: Regular function loses `this` in callback
  delayedGreetBroken() {
    setTimeout(function() {
      console.log(`Hello from ${this.name}`);  // `this` is undefined or window!
    }, 100);
  },
  
  // SOLUTION 1: Arrow function (lexical `this`)
  delayedGreetArrow() {
    setTimeout(() => {
      console.log(`Hello from ${this.name}`);  // Works! Arrow uses outer `this`
    }, 100);
  },
  
  // SOLUTION 2: Bind
  delayedGreetBind() {
    setTimeout(function() {
      console.log(`Hello from ${this.name}`);
    }.bind(this), 100);
  },
  
  // SOLUTION 3: Store reference
  delayedGreetRef() {
    const self = this;
    setTimeout(function() {
      console.log(`Hello from ${self.name}`);
    }, 100);
  }
};
```

### Pitfall: Stale Closure in React/Hooks

```javascript
// PROBLEM: Closure captures initial value
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      // This `count` is always 0! Stale closure.
      console.log('Count:', count);
      setCount(count + 1);  // Always sets to 1
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);  // Empty deps = closure captures initial count
  
  return <div>{count}</div>;
}

// SOLUTION 1: Use functional update
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prevCount => prevCount + 1);  // No closure issue
  }, 1000);
  
  return () => clearInterval(interval);
}, []);

// SOLUTION 2: Use ref for latest value
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const interval = setInterval(() => {
    console.log('Count:', countRef.current);  // Always current
  }, 1000);
  
  return () => clearInterval(interval);
}, []);
```

---

## Quick Interview Reference

### "What is a closure?"

> "A closure is a function that retains access to variables from its outer (enclosing) scope, even after the outer function has finished executing. The inner function 'closes over' the variables it needs."

### "How is lexical scoping different from dynamic scoping?"

> "Lexical scoping determines variable scope based on where the function is written in the code. Dynamic scoping would determine scope based on where the function is called. JavaScript uses lexical scoping, so you can always trace a variable's origin by looking up through the nested code structure."

### "Name three practical uses of closures"

> "1) Data privacy/encapsulation - hiding implementation details behind a public API. 2) Function factories - creating specialized functions like multipliers or formatters. 3) Memoization - caching expensive computation results in a closure."

### "What's the var vs let loop problem?"

> "`var` is function-scoped, so all loop iterations share the same variable. When callbacks execute later, they all see the final value. `let` is block-scoped, creating a new binding for each iteration, so each callback captures its own value."

### "How can closures cause memory leaks?"

> "Closures keep their outer scope variables alive in memory. If a closure references large data structures and the closure itself is never dereferenced (e.g., in a long-lived event listener), that memory cannot be garbage collected."

---

## Practice Questions

1. What does this code output?
```javascript
function createFunctions() {
  const funcs = [];
  for (var i = 0; i < 3; i++) {
    funcs.push(function() { return i; });
  }
  return funcs;
}
const fns = createFunctions();
console.log(fns[0](), fns[1](), fns[2]());
```

2. Implement a `once` function that ensures a function can only be called once.

3. Create a `privateCounter` with `increment`, `decrement`, and `getValue` methods where the count cannot be directly modified.

4. Explain why this creates a memory leak and how to fix it:
```javascript
function attachHandler() {
  const data = fetchLargeDataset();
  element.onclick = () => process(data);
}
```

5. What is the difference between these two?
```javascript
const obj = {
  method: function() { return this; },
  arrow: () => this
};
```
