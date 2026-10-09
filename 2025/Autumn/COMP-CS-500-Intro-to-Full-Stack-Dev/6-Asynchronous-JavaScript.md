
JavaScript is **single-threaded**, but the Event Loop allows executing operations **without blocking** the main thread. Asynchronicity = the ability to do other things while waiting for a result (like sending an email and continuing work without waiting for a reply).

---

## Event Loop - The Heart of Asynchronicity

### Architecture (5 components):

```
┌─────────────────────────────────────┐
│ 1. Call Stack                       │ ← Only place for execution
│    (synchronous code)               │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│ 2. Web APIs / Node APIs             │ ← Background operations (C++)
│    setTimeout, fetch, fs.readFile   │    Work in parallel!
└─────────────────────────────────────┘
          ↓ callback ready
┌─────────────────────────────────────┐
│ 3. Microtask Queue (priority!)      │
│    • Promise.then()                 │
│    • async/await                    │
│    • queueMicrotask()               │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│ 4. Task Queue (Macrotasks)          │
│    • setTimeout()                   │
│    • setInterval()                  │
│    • I/O, UI events                 │
└─────────────────────────────────────┘
```

### Event Loop Algorithm:

```javascript
while (true) {
    // 1. Execute EVERYTHING from Call Stack
    executeCallStack();
    
    // 2. Execute ALL Microtasks (even new ones added during execution!)
    while (microtaskQueue.length > 0) {
        executeMicrotask();
    }
    
    // 3. Execute ONE Macrotask
    if (macrotaskQueue.length > 0) {
        executeMacrotask();
    }
    
    // 4. Render (in browser)
    if (needsRender()) render();
    
    // 5. Repeat
}
```

### Key Insights:

**Why do Microtasks have priority?**

- For **state predictability**
- Promise chains must execute **atomically** (without interruptions)
- Otherwise other tasks might sneak in between `.then()` calls

**Why only ONE Macrotask?**

- Balance between performance and responsiveness
- Allows handling UI events between tasks
- If all executed at once → UI would freeze

**What if a Microtask hangs?**

- Everything blocks! UI stops responding
- Macrotasks won't execute (stuck in queue)
- **Solution:** break heavy operations into chunks with `setTimeout`

---

## Callbacks

Callback = function passed as a parameter, called later.

A callback itself is NOT "synchronous" or "asynchronous"! **The difference is in WHO and WHEN calls it:**

#### Synchronous callback (JavaScript calls immediately):

```javascript
[1, 2, 3].forEach(num => console.log(num));
// forEach calls callback DIRECTLY, in Call Stack
```

#### Asynchronous callback (Browser/Node API calls via queue):

```javascript
setTimeout(() => console.log("Later"), 1000);
// setTimeout REGISTERS callback in Web API
// Does NOT call it! Returns immediately
// After 1s callback goes to Task Queue
```

### How does code "distinguish"?

**JavaScript does NOT distinguish!** The **environment** (Browser/Node) distinguishes:

```javascript
// Synchronous: YOU call callback directly
array.forEach(cb);  // ← JS executes cb immediately

// Asynchronous: Browser API registers and will call later
setTimeout(cb, 0);  // ← Browser API registers
                     // ← setTimeout RETURNS immediately
                     // ← cb will go to queue
```

**Magic:** `setTimeout`, `fetch`, `fs.readFile` are **NOT JavaScript**! They're Browser/Node APIs written in C++ that work in **background threads**!

### Callback Hell (Pyramid of Doom):

```javascript
getUser(1, (err, user) => {
    if (err) return handleError(err);
    getPosts(user.id, (err, posts) => {
        if (err) return handleError(err);
        getComments(posts[0].id, (err, comments) => {
            if (err) return handleError(err);
            // 4 levels of nesting!
        });
    });
});
```

**Problems:**

- Code "goes right"
- Repetitive error handling
- Results "stuck" inside

---

## Promises

Promise = object representing the **future result** of an asynchronous operation.

### Three states:

```
pending (waiting) 
    ↓
    ├→ fulfilled (success)
    └→ rejected (error)
```

### Creation:

```javascript
new Promise((resolve, reject) => {
    // Asynchronous work
    if (success) resolve(result);
    else reject(error);
});
```

### Key features:

**1. `.then()` ALWAYS returns a new Promise:**

```javascript
Promise.resolve(5)
    .then(x => x * 2)    // ← returns Promise(10)
    .then(x => x + 3)    // ← returns Promise(13)
    .then(x => console.log(x)); // 13
```

**2. Chains = flat structure (NO nesting):**

```javascript
getUser(1)
    .then(user => getPosts(user.id))
    .then(posts => getComments(posts[0].id))
    .then(comments => console.log(comments))
    .catch(error => console.error(error)); // ← single error handler!
```

**3. Callbacks go to Microtask Queue:**

```javascript
console.log("1");
Promise.resolve().then(() => console.log("3"));
console.log("2");
// Output: 1, 2, 3
// Promise.then went to Microtask Queue!
```

### Promise API:

```javascript
Promise.all([p1, p2, p3])      // Waits for ALL, rejects if any fails
Promise.allSettled([p1, p2])   // Waits for all, doesn't reject
Promise.race([p1, p2])         // First to complete (success or fail)
Promise.any([p1, p2])          // First successful
```

### Thenable - what is it?

**Thenable** = any object with a `.then()` method

```javascript
const thenable = {
    then: function(onFulfilled) {
        setTimeout(() => onFulfilled("data"), 100);
    }
};

// Can be used as Promise!
Promise.resolve(thenable).then(data => console.log(data));
```

**Why?**

- Compatibility with old libraries (jQuery, Bluebird)
- Can be converted to full Promise: `Promise.resolve(thenable)`

**Rule:** Every Promise is a Thenable, but not every Thenable is a Promise!

---

## async/await

**Syntactic sugar over Promises**, making asynchronous code look synchronous.

### Two keywords:

**`async`** — makes function asynchronous (returns Promise):

```javascript
async function test() {
    return 42;
}
// Equivalent to:
function test() {
    return Promise.resolve(42);
}
```

**`await`** — waits for Promise result (pauses function):

```javascript
async function getData() {
    const response = await fetch('/api/data'); // ← wait
    const data = await response.json();        // ← wait
    return data; // ← returned as Promise!
}
```

### Critical rules:

**1. `await` ONLY inside `async`:**

```javascript
// Error!
function regular() {
    await Promise.resolve(); // SyntaxError!
}

// Correct
async function proper() {
    await Promise.resolve(); // OK
}
```

**2. `async` creates chain reaction:**

```javascript
async function step1() { /* ... */ }
async function step2() { await step1(); /* ... */ }  // ← must be async
async function step3() { await step2(); /* ... */ }  // ← must be async
```

**3. Under the hood - Promise:**

```javascript
async function f(x) {
    return x + 1;
}

// Transforms to:
function f(x) {
    return new Promise((resolve, reject) => {
        try {
            resolve(x + 1);
        } catch (e) {
            reject(e);
        }
    });
}
```

### IIFE (Immediately Invoked Function Expression):

**Problem:** `await` not allowed at top level (in regular scripts)

**Solution:** Create async block on the spot:

```javascript
(async () => {
    const data = await fetch('/api/data');
    console.log(data);
})();  // ← invoked immediately!
```

---

## Performance: Sequential vs Parallel

### BAD (sequential - slow):

```javascript
async function getData() {
    const user = await getUser();        // 1s
    const posts = await getPosts();      // 1s (waits for user!)
    const comments = await getComments(); // 1s (waits for posts!)
    // TOTAL: 3 seconds
}
```

**Visualization:**

```
0s ──┬─ getUser() ──┬─ 1s
     │              ↓
     │         getPosts() ──┬─ 2s
     │                      ↓
     │              getComments() ──── 3s
```

### GOOD (parallel - fast):

```javascript
async function getData() {
    // Start ALL at once!
    const [user, posts, comments] = await Promise.all([
        getUser(),
        getPosts(),
        getComments()
    ]);
    // TOTAL: 1 second (longest one)
}
```

**Visualization:**

```
0s ──┬─ getUser() ────┬─ 1s
     ├─ getPosts() ───┤
     └─ getComments() ┘
```

### When to use what?

**Sequential (`await` one after another):**

- Second request **DEPENDS** on first:

```javascript
const user = await getUser(123);
const posts = await getPosts(user.id); // ← needs user.id!
```

**Parallel (`Promise.all`):**

- Requests are **INDEPENDENT**:

```javascript
const [weather, news, stocks] = await Promise.all([
    getWeather(),  // don't depend on each other
    getNews(),
    getStocks()
]);
```

---

## Additional Concepts

### Hoisting:

**Function Declaration** — hoisted:

```javascript
foo(); // Works!
function foo() { return "Hi"; }
```

**Function Expression** — NOT hoisted:

```javascript
bar(); // ReferenceError!
const bar = () => "Hi";
```

**Why?** Declarations are hoisted, initializations are not.

---

### Closures:

**Closure** = function that "remembers" variables from outer scope.

```javascript
function makeCounter() {
    let count = 0; // ← private!
    
    return {
        increment: () => ++count,
        value: () => count
    };
}

const counter = makeCounter();
counter.increment(); // 1
counter.value();     // 1
counter.count;       // undefined ← not accessible outside!
```

**Use cases:**

- Private variables
- Event handlers with data
- Callbacks in loops

---

### Arity:

**Arity** = number of function parameters.

```javascript
function add(a, b) { }       // 2-arity (binary)
function sum(a, b, c) { }    // 3-arity (ternary)

console.log(add.length);  // 2
console.log(sum.length);  // 3
```

**Why know this?** To implement `curry()` you need to know how many arguments the function expects!

---

### Currying vs Callback Hell:

**Visually similar, but DIFFERENT!**

**Currying** (synchronous):

```javascript
const add = (a) => (b) => (c) => a + b + c;
add(1)(2)(3); // ← executes IMMEDIATELY
```

**Callback Hell** (asynchronous):

```javascript
getUser(1, (user) => {
    getPosts(user.id, (posts) => {
        // ← executes LATER via queues
    });
});
```

**Differences:**

- Currying: synchronous, returns functions, you control timing
- Callbacks: asynchronous, system controls timing, result inside callback

---

## Evolution of Asynchronicity

```
1995: Callbacks
      ↓
      ❌ Pyramid of Doom
      ❌ Results inside callbacks
      
2015: Promises (ES2015)
      ↓
      ✅ Flat chains
      ✅ Single error handling
      ❌ Still not on "next line"
      
2017: async/await (ES2017)
      ↓
      ✅ Looks like synchronous code
      ✅ try/catch works
      ✅ All variables accessible
```

---

## Practical Rules

### DO:

- Use **async/await** for new code
- Use **Promise.all()** for independent operations
- Wrap async code in **try/catch**
- Use **IIFE** for await at top level
- Remember **Microtask priority**

### DON'T:

- Don't use callbacks in new code (only for legacy)
- Don't do sequential `await` for independent operations
- Don't forget `async` when using `await`
- Don't create infinite Microtask loops (will block UI)

---

## Key Insights

1. **JavaScript is single-threaded**, but Browser/Node are multi-threaded
2. **setTimeout/fetch/fs are NOT JavaScript**, they're Browser/Node APIs (C++)
3. **Microtasks ALWAYS have priority** over macrotasks
4. **Only ONE macrotask per cycle** (balance performance/responsiveness)
5. **Callback itself isn't async** — WHO calls it matters
6. **async/await = sugar over Promises** — need to understand both
7. **Promise.all() is critical** for independent operations performance
8. **Heavy Microtask blocks EVERYTHING** — break into chunks

---

## Summary Table

|Concept|When to use|Where it goes|
|---|---|---|
|**Callbacks**|Legacy code, simple cases|Task Queue (if async)|
|**Promises**|Modern code, need `.all()/.race()`|Microtask Queue|
|**async/await**|New code, need readability|Microtask Queue|
|**Promise.all()**|Independent parallel operations|Microtask Queue|

**Golden rule:** Use async/await + Promise.all() = best combination!