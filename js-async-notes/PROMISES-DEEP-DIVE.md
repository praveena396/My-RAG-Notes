# Promises Deep Dive: every way to create and use them

All outputs in this file were run in Node 22 and checked.
A **Write in your copy** section is at the very end.

Contents
1. The mental model
2. Creating promises (5 ways)
3. Using promises: `.then`, `.catch`, `.finally`, `await`
4. Chaining in detail
5. Error handling in detail
6. Sequential vs parallel
7. `Promise.all`, `allSettled`, `race`, `any`
8. Promises inside loops
9. The same task written three ways
10. `async` functions in every shape
11. Pitfalls with `new Promise`
12. Patterns you will actually use (wait, timeout, retry, promisify)
13. Ordering puzzles
14. Using promises in Express
15. "I want to... use..." cheat sheet

Helpers used in the examples:

```js
// resolves with value v after ms milliseconds
const wait = (ms, v) => new Promise((resolve) => setTimeout(() => resolve(v), ms));
// rejects with an Error after ms milliseconds
const fail = (ms, msg) => new Promise((_, reject) => setTimeout(() => reject(new Error(msg)), ms));
```

---

## 1. The mental model

A promise is an object that stands for a result that is not ready yet.

```
pending ──resolve(value)──> fulfilled (has a value)
   │
   └─────reject(error)────> rejected  (has an error)
```

- **Settled** means fulfilled or rejected. A settled promise never changes again.
- There are only two jobs: **create** a promise (someone produces a result later) and **use** a promise (someone reacts when it settles).
- `.then` and `await` are two ways to do the second job. They are the same mechanism with different syntax.

---

## 2. Creating promises (5 ways)

### 2.1 `new Promise((resolve, reject) => { ... })`

Use it to wrap code that gives results through callbacks (timers, old Node APIs).

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 1000);   // after 1 s the promise is fulfilled with "done"
});
```

- `resolve(value)` makes it fulfilled.
- `reject(error)` makes it rejected. Always reject with an `Error` object, not a string.

### 2.2 `Promise.resolve(value)` and `Promise.reject(error)`

An already-settled promise. Useful in tests, defaults, and when a function must always return a promise.

```js
await Promise.resolve(7);            // 7
Promise.resolve(7).then((x) => x + 1);   // fulfills with 8

try {
  await Promise.reject(new Error("no"));
} catch (e) {
  console.log("rejected:", e.message);   // rejected: no
}
```

### 2.3 Any `async` function

An `async` function **always returns a promise**.

```js
async function f() { return 1; }
console.log(f());          // Promise { 1 }
f().then(console.log);     // 1

async function g() { throw new Error("bad"); }
g().catch((e) => console.log("g caught:", e.message));   // g caught: bad
```

- `return x` fulfills the promise with `x`.
- `throw err` rejects the promise with `err`.

### 2.4 APIs that already return promises

`fetch`, Mongoose (`Home.find()`, `findById`, `save`), `fs.promises.readFile`, and so on. You do not create these, you just use them.

```js
const homes = await Home.find();             // Mongoose query, works with await
const data = await require("fs").promises.readFile("homes.json", "utf8");
```

### 2.5 `util.promisify(callbackFunction)`

Converts an error-first callback function into a promise-returning function.

```js
const { promisify } = require("util");
const readFile = promisify(require("fs").readFile);
const text = await readFile("homes.json", "utf8");
```

---

## 3. Using promises: `.then`, `.catch`, `.finally`, `await`

### The four tools

| Tool | Runs when | Notes |
|---|---|---|
| `.then(onFulfilled)` | the promise fulfills | gets the value, returns a **new** promise |
| `.catch(onRejected)` | the promise rejects | gets the error, returns a **new** promise |
| `.finally(fn)` | either way | gets **no** argument, used for cleanup |
| `await p` | pauses until `p` settles | gives the value, or **throws** the error |

### The same task in both styles

```js
// .then style
Home.findById(id)
  .then((home) => {
    console.log(home.houseName);
  })
  .catch((err) => {
    console.log("failed", err.message);
  });

// await style
async function show(id) {
  try {
    const home = await Home.findById(id);
    console.log(home.houseName);
  } catch (err) {
    console.log("failed", err.message);
  }
}
```

### Every form of `.then`

```js
p.then((value) => { /* success */ });                       // success only
p.then((value) => { /* success */ }, (err) => { /* fail */ });   // two callbacks (rare, see 5.4)
p.then(null, (err) => { /* fail */ });                      // same as .catch
p.catch((err) => { /* fail */ });                           // failure only
p.finally(() => { /* cleanup */ });                         // always
```

### Every form of `await`

```js
const value = await p;                       // wait for one promise
const [a, b] = await Promise.all([p1, p2]);  // wait for several at once
await p;                                     // wait, ignore the value
const v = await 42;                          // works on non-promises too, gives 42
```

`await` also works on anything with a `.then` method (a "thenable"):

```js
console.log(await { then(resolve) { resolve("thenable!"); } });   // thenable!
```

(You rarely need this, but it explains why `await Home.find()` works on a Mongoose query.)

### Rules for `await`

- It can only be used inside an `async` function (or at the top level of an ES module, see 10.5).
- It pauses **only that function**. The rest of the program keeps running.
- Code after `await` resumes through the **microtask queue**.

---

## 4. Chaining in detail

`.then` returns a **new promise**. What that new promise becomes depends on what your callback does.

### 4.1 Return a value: next step gets the value

```js
Promise.resolve(5)
  .then((n) => n * 2)             // 10
  .then((n) => n + 1)             // 11
  .then((n) => console.log(n));   // 11
```

### 4.2 Return a promise: the chain waits for it

```js
Promise.resolve(1)
  .then((n) => wait(50, n + 100))   // chain pauses until this settles
  .then((v) => console.log(v));     // 101
```

This is how dependent async steps are written flat.

```js
getUser(id)
  .then((user) => getFavourites(user))   // returns a promise, so the chain waits
  .then((favs) => getHomes(favs))
  .then((homes) => res.render("favourites", { homes }))
  .catch(next);
```

### 4.3 Return nothing: next step gets `undefined`

```js
Promise.resolve(1)
  .then((n) => { n + 1; })              // calculated, never returned
  .then((v) => console.log(v));         // undefined
```

This is the same trap as `.map` with braces and no `return`.

### 4.4 Throw: jump to the next `.catch`

```js
Promise.resolve(1)
  .then(() => { throw new Error("x"); })
  .then(() => console.log("skipped"))      // never runs
  .catch((e) => console.log(e.message));   // x
```

### 4.5 Summary table

| Inside a `.then` callback you... | The next step receives |
|---|---|
| return a value | that value |
| return a promise | the chain waits, then gets that promise's result |
| return nothing | `undefined` |
| throw | skips to the next `.catch` |

### 4.6 Common chaining bugs

**Bug 1: forgetting to return the inner promise.** The chain does not wait.

```js
// WRONG: render runs before getHomes finishes
getUser(id).then((user) => {
  getHomes(user);                          // started, but not returned
}).then((homes) => res.render("x", { homes }));   // homes is undefined

// RIGHT
getUser(id).then((user) => {
  return getHomes(user);
}).then((homes) => res.render("x", { homes }));
```

**Bug 2: nesting instead of chaining.** It works, but it becomes callback hell again.

```js
// avoid
getUser(id).then((user) => {
  getFavourites(user).then((favs) => {
    getHomes(favs).then((homes) => { /* ... */ });
  });
});

// prefer a flat chain (see 4.2) or await
```

**Bug 3: a promise with no `.catch` and no `await`.** See 5.6.

---

## 5. Error handling in detail

### 5.1 One `.catch` handles every step above it

```js
step1().then(step2).then(step3).catch((err) => console.log("something failed", err.message));
```

### 5.2 `.catch` can recover

If your `.catch` callback returns a value, the chain becomes **fulfilled** again.

```js
fail(10, "boom")
  .catch(() => "default")
  .then((v) => console.log("recovered ->", v));   // recovered -> default
```

### 5.3 Rethrow to pass the error on

Log it, then throw it again so later `.catch` handlers still see it.

```js
Promise.resolve(1)
  .then(() => { throw new Error("inner"); })
  .catch((e) => { console.log("log:", e.message); throw e; })
  .catch((e) => console.log("rethrown reached:", e.message));
// log: inner
// rethrown reached: inner
```

### 5.4 `.then(a, b)` is not the same as `.then(a).catch(b)`

```js
Promise.resolve(1)
  .then(
    () => { throw new Error("in then"); },     // a
    (e) => console.log("NOT reached")          // b: only handles errors from BEFORE this .then
  )
  .catch((e) => console.log("caught later:", e.message));   // caught later: in then
```

With `.then(a, b)`, `b` does **not** catch errors thrown inside `a`. Prefer `.then(a).catch(b)`.

### 5.5 `.finally`

Runs on success or failure, for cleanup (hide a spinner, close a connection). It gets no argument and does **not** change the value.

```js
fail(10, "z").catch(() => {}).finally(() => console.log("finally runs"));

const v = await Promise.resolve("val").finally(() => "ignored");
console.log(v);   // val
```

### 5.6 `try / catch / finally` with `await`

```js
async function getFavs(userId) {
  try {
    const user = await User.findById(userId);
    return user.favourites;
  } catch (err) {
    console.log("db problem:", err.message);
    throw err;                     // or return a default
  } finally {
    console.log("done");           // always runs
  }
}
```

`try/catch` only catches a rejection if you **await** the promise.

### 5.7 `return await` inside `try`

```js
async function noAwait() {
  try { return fail(10, "e1"); }            // returns the promise without waiting
  catch (e) { return "caught inside"; }
}
async function withAwait() {
  try { return await fail(10, "e2"); }      // waits here, so the catch can see it
  catch (e) { return "caught inside"; }
}

noAwait().then(console.log, (e) => console.log("escaped ->", e.message));   // escaped -> e1
withAwait().then(console.log);                                              // caught inside
```

Inside a `try` block, write `return await somePromise` if you want the `catch` to handle its failure.

### 5.8 Unhandled rejections

A rejected promise with no handler is an **unhandled rejection**. In Node 15 and later, this **crashes the process** by default.

```js
Promise.reject(new Error("nobody handles me"));   // crash
```

Every promise should end with `await` inside `try/catch`, a `.catch`, or be returned to a caller who handles it.

---

## 6. Sequential vs parallel

### 6.1 Sequential (one after another): slow when the jobs are independent

```js
const a = await wait(300, "A");
const b = await wait(300, "B");
// total ≈ 600 ms
```

### 6.2 Parallel with `Promise.all`

```js
const [a, b] = await Promise.all([wait(300, "A"), wait(300, "B")]);
// total ≈ 300 ms
```

### 6.3 Parallel by starting first, awaiting later

```js
const p1 = wait(300, "A");   // starts now
const p2 = wait(300, "B");   // starts now
const a = await p1;
const b = await p2;
// total ≈ 300 ms
```

### 6.4 When to use which

| Situation | Use |
|---|---|
| Step 2 needs the result of step 1 | sequential `await` |
| Steps are independent | `Promise.all` |
| Independent, but you want to handle each result separately | start first, await later |

Real example, in your favourites page:

```js
// slow: user lookup and homes lookup do not depend on each other
const user = await User.findById(userId);
const allHomes = await Home.find();

// fast
const [user, allHomes] = await Promise.all([User.findById(userId), Home.find()]);
```

---

## 7. `Promise.all`, `allSettled`, `race`, `any`

| Method | Fulfills when | Rejects when | Result |
|---|---|---|---|
| `Promise.all([...])` | **all** fulfill | **any** rejects (immediately) | array of values, in **input order** |
| `Promise.allSettled([...])` | **all** settle | never | array of `{ status, value }` or `{ status, reason }` |
| `Promise.race([...])` | the first one **settles** | the first one settles as rejected | that one result |
| `Promise.any([...])` | the first one **fulfills** | **all** reject (`AggregateError`) | that one value |

```js
// all: order matches the input, not the finishing order
await Promise.all([wait(200, "slow"), wait(50, "fast")]);   // [ 'slow', 'fast' ]

// all: one failure rejects everything
try {
  await Promise.all([wait(200, "ok"), fail(50, "one failed")]);
} catch (e) {
  console.log("all rejected:", e.message);                  // all rejected: one failed
}

// allSettled: never rejects, you inspect each result
const results = await Promise.allSettled([wait(10, "ok"), fail(10, "bad")]);
// [ { status: 'fulfilled', value: 'ok' },
//   { status: 'rejected',  reason: Error: bad } ]

// race: first to settle wins
await Promise.race([wait(200, "slow"), wait(50, "fast")]);  // 'fast'

// any: first to FULFILL wins, failures are ignored
await Promise.any([fail(10, "x"), wait(100, "second ok")]); // 'second ok'

// any: all fail
try {
  await Promise.any([fail(10, "x"), fail(20, "y")]);
} catch (e) {
  console.log(e.constructor.name);                          // AggregateError
}
```

When to use which:

- `all`: you need every result and one failure should stop everything.
- `allSettled`: you want to try everything and report which parts failed.
- `race`: timeouts (see 12.2).
- `any`: the first successful source wins (for example, several mirrors).

---

## 8. Promises inside loops

```js
const ids = [1, 2, 3];
const save = (id) => wait(100 * (4 - id)).then(() => console.log("saved", id));
// id 1 takes 300 ms, id 2 takes 200 ms, id 3 takes 100 ms
```

### 8.1 `forEach` with `async` does not wait

```js
ids.forEach(async (id) => { await save(id); });
console.log("after forEach (nothing waited)");
// after forEach (nothing waited)   <- prints FIRST
// saved 3
// saved 2
// saved 1
```

`forEach` ignores the promises returned by the callback, so nothing waits for them.

### 8.2 `for...of` with `await`: sequential

```js
for (const id of ids) { await save(id); }
console.log("for...of done (sequential)");
// saved 1
// saved 2
// saved 3
// for...of done (sequential)       (total ≈ 600 ms)
```

Use it when order matters or each step needs the one before.

### 8.3 `map` + `Promise.all`: parallel

```js
await Promise.all(ids.map((id) => save(id)));
console.log("map+all done (parallel)");
// saved 3
// saved 2
// saved 1
// map+all done (parallel)          (total ≈ 300 ms)
```

Note that the work **finished** in a different order than the list. `Promise.all` still returns its results in the input order, but side effects happen in finishing order.

### 8.4 Which to use

| Goal | Use |
|---|---|
| Order matters, or step depends on the previous | `for...of` + `await` |
| Independent jobs, speed matters | `Promise.all(arr.map(...))` |
| Thousands of jobs | process in batches (too many at once can overload the database or API) |

---

## 9. The same task written three ways

Task: for a logged-in user, load their favourite homes, then render the page.

### 9.1 Callbacks

(Shown for comparison. Modern Mongoose, version 7 and later, no longer accepts callbacks, so you will only meet this style in older code or Node's built-in APIs.)

```js
exports.getFavouriteList = (req, res, next) => {
  User.findById(req.session.user._id, (err, user) => {
    if (err) return next(err);
    Home.find({ _id: { $in: user.favourites } }, (err, homes) => {
      if (err) return next(err);
      res.render("store/favourite-list", { favouriteHomes: homes });
    });
  });
};
```

### 9.2 Promises with `.then`

```js
exports.getFavouriteList = (req, res, next) => {
  User.findById(req.session.user._id)
    .then((user) => Home.find({ _id: { $in: user.favourites } }))   // returned promise: chain waits
    .then((homes) => res.render("store/favourite-list", { favouriteHomes: homes }))
    .catch(next);                                                   // one handler for everything
};
```

### 9.3 `async/await`

```js
exports.getFavouriteList = async (req, res, next) => {
  try {
    const user = await User.findById(req.session.user._id);
    const homes = await Home.find({ _id: { $in: user.favourites } });
    res.render("store/favourite-list", { favouriteHomes: homes });
  } catch (err) {
    next(err);
  }
};
```

All three do the same thing. Notice how the nesting disappears and error handling collapses into one place.

---

## 10. `async` functions in every shape

### 10.1 Declaration

```js
async function load() { return await Home.find(); }
```

### 10.2 Expression

```js
const load = async function () { return await Home.find(); };
```

### 10.3 Arrow (the one you see in controllers)

```js
const load = async () => await Home.find();
exports.getHomes = async (req, res, next) => { /* ... */ };
```

### 10.4 Method in a class or object

```js
class HomeService {
  async all() { return await Home.find(); }
}
const service = { async all() { return await Home.find(); } };
```

### 10.5 Immediately invoked, and top-level `await`

```js
(async () => {
  const homes = await Home.find();
  console.log(homes.length);
})();
```

At the top level of a file, `await` works only in **ES modules** (`.mjs`, or `"type": "module"` in `package.json`). Your project uses `require` (CommonJS), so wrap awaits in an async function.

### 10.6 Always returns a promise

```js
async function f() { return 1; }
f();               // Promise { 1 }   NOT 1
await f();         // 1
```

If you call an `async` function without `await` or `.then`, you get a promise, not the value. This is the most common bug.

---

## 11. Pitfalls with `new Promise`

### 11.1 Resolving or rejecting twice: only the first counts

```js
const result = await new Promise((resolve, reject) => {
  resolve("first");
  resolve("second");           // ignored
  reject(new Error("late"));   // ignored
});
console.log(result);           // first
```

### 11.2 Do not wrap something that already returns a promise

```js
// WRONG: pointless wrapper, and errors can get lost
const getHomes = () => new Promise((resolve, reject) => {
  Home.find().then(resolve).catch(reject);
});

// RIGHT
const getHomes = () => Home.find();
```

Use `new Promise` only to wrap callback-based code (timers, old APIs).

### 11.3 Forgetting `reject` for the failure case

```js
// WRONG: if readFile fails, the promise stays pending forever
new Promise((resolve) => {
  fs.readFile("homes.json", "utf8", (err, data) => { resolve(data); });
});

// RIGHT
new Promise((resolve, reject) => {
  fs.readFile("homes.json", "utf8", (err, data) => {
    if (err) return reject(err);
    resolve(data);
  });
});
```

---

## 12. Patterns you will actually use

### 12.1 `wait`

```js
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
await wait(1000);
```

### 12.2 Timeout (`race`)

```js
const timeout = (promise, ms) =>
  Promise.race([
    promise,
    new Promise((_, reject) => setTimeout(() => reject(new Error("timeout")), ms)),
  ]);

try {
  await timeout(wait(500), 100);              // too slow
} catch (e) {
  console.log(e.message);                     // timeout
}
console.log(await timeout(wait(50, "in time"), 200));   // in time
```

### 12.3 Retry

```js
async function retry(fn, times) {
  for (let i = 1; i <= times; i++) {
    try {
      return await fn();
    } catch (err) {
      if (i === times) throw err;             // out of attempts
      console.log("retrying after:", err.message);
    }
  }
}

let n = 0;
const flaky = async () => { n++; if (n < 3) throw new Error("try " + n); return "ok after " + n; };
console.log(await retry(flaky, 5));
// retrying after: try 1
// retrying after: try 2
// ok after 3
```

### 12.4 `promisify`

See 2.5.

---

## 13. Ordering puzzles

### Puzzle 1

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3")).then(() => console.log("4"));
console.log("5");
// 1, 5, 3, 4, 2
```

Why: `1` and `5` are synchronous. Then **all** microtasks run (`3`, then `4`, which was queued when `3` finished). Only after that does the timer task run (`2`).

### Puzzle 2

```js
async function run() {
  console.log("A");
  await null;
  console.log("B");
}
run();
console.log("C");
// A, C, B
```

Why: `run()` starts synchronously and prints `A`, then pauses at `await`. The caller continues and prints `C`. `B` is a microtask that runs afterwards.

### Rule for all of them

```
1. Run all synchronous code
2. Run ALL microtasks (.then callbacks, code after await)
3. Run ONE task (setTimeout, I/O callback)
4. Repeat from 2
```

---

## 14. Using promises in Express

### 14.1 With `.then`

```js
exports.getHomeDetails = (req, res, next) => {
  Home.findById(req.params.homeId)
    .then((home) => {
      if (!home) return res.redirect("/homes");
      res.render("store/home-detail", { home });
    })
    .catch(next);
};
```

### 14.2 With `async/await`

```js
exports.getHomeDetails = async (req, res, next) => {
  try {
    const home = await Home.findById(req.params.homeId);
    if (!home) return res.redirect("/homes");
    res.render("store/home-detail", { home });
  } catch (err) {
    next(err);
  }
};
```

### 14.3 A wrapper so you do not repeat `try/catch`

```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

router.get("/favourites", asyncHandler(storeController.getFavouriteList));
```

Now any error thrown or rejected inside the controller is passed to your error middleware.

### 14.4 Express 4 vs Express 5

- **Express 4** does not catch rejected promises from async handlers. Without `try/catch` or a wrapper, the request hangs.
- **Express 5** forwards rejected promises from handlers to the error middleware automatically.

### 14.5 Mongoose

Queries are "thenables", so `await Home.find()` works. If you need a real promise, call `.exec()`: `await Home.find().exec()`.

---

## 15. "I want to... use..." cheat sheet

| I want to | Use |
|---|---|
| Wait for one result | `const x = await promise` |
| Run steps in order, each needing the last | sequential `await`, or a flat `.then` chain |
| Run independent jobs at the same time | `await Promise.all([...])` |
| Try several jobs and report which failed | `Promise.allSettled([...])` |
| Take the first result of several | `Promise.race` (settles first) or `Promise.any` (fulfils first) |
| Add a timeout | `Promise.race([job, rejectAfter(ms)])` |
| Handle errors in one place | `try/catch` around awaits, or one `.catch` at the end |
| Clean up either way | `finally` |
| Provide a fallback on failure | `.catch(() => fallback)` |
| Wrap a callback API | `new Promise` with `resolve` and `reject` |
| Wrap a Node error-first function | `util.promisify` |
| Loop in order | `for...of` with `await` |
| Loop in parallel | `await Promise.all(arr.map(...))` |
| Make a function return a promise | mark it `async` |

---

# Write in your copy

```
CREATE a promise
  new Promise((resolve, reject) => {...})   wrap callback code (timers, old APIs)
  Promise.resolve(v) / Promise.reject(e)    already settled
  any async function                         always returns a promise
  fetch, Mongoose, fs.promises               already return promises
  util.promisify(fn)                         error-first callback -> promise

USE a promise
  .then(cb)     success, returns a NEW promise
  .catch(cb)    failure, returns a NEW promise
  .finally(cb)  always, no argument, doesn't change the value
  await p       pauses ONLY this function, gives value or THROWS
  await works only inside async (or top-level in ES modules)
  Forgetting await = you get a Promise, not the value

CHAINING: inside .then
  return value   -> next gets value
  return promise -> chain WAITS for it
  return nothing -> next gets undefined   (bug: forgot return)
  throw          -> skips to next .catch

ERRORS
  one .catch (or try/catch) covers every step above
  .catch can RECOVER: .catch(() => "default")
  rethrow to pass on: catch(e){ log; throw e; }
  .then(a,b) does NOT catch errors thrown in a. Use .then(a).catch(b)
  try/catch only catches AWAITED promises
  inside try, write: return await p   (so catch sees the failure)
  unhandled rejection crashes Node 15+

SEQUENTIAL vs PARALLEL
  await a(); await b();                  600 ms  (b waits for a)
  await Promise.all([a(), b()]);         300 ms  (independent jobs)

COMBINATORS
  all        all succeed, or rejects at first failure. Results in INPUT order
  allSettled never rejects, gives {status, value|reason} for each
  race       first to SETTLE wins
  any        first to FULFIL wins (AggregateError if all fail)

LOOPS
  forEach(async ...)       does NOT wait  (bug)
  for (x of arr) await f   sequential
  await Promise.all(arr.map(f))   parallel (finishes in finishing order)

new Promise PITFALLS
  resolve/reject twice: only the first counts
  don't wrap something that already returns a promise
  always call reject on the failure path or it hangs forever

EXPRESS
  Express 4: async handler needs try/catch + next(err), or an asyncHandler wrapper
  Express 5: forwards rejected promises automatically
  const asyncHandler = (fn) => (req,res,next) =>
        Promise.resolve(fn(req,res,next)).catch(next);

ORDER: sync code -> ALL microtasks (.then, after await) -> ONE task (setTimeout) -> repeat
  log 1; setTimeout(log 2, 0); then(log 3).then(log 4); log 5   ->  1, 5, 3, 4, 2
```

---

# Try these

1. What does `total` take, and why?
   ```js
   const a = await wait(200);
   const b = await wait(200);
   const c = await wait(200);
   ```
   How would you cut it to about 200 ms?
2. What does this print?
   ```js
   Promise.resolve(2)
     .then((n) => { n * 10; })
     .then((v) => console.log(v));
   ```
3. What does `r` hold?
   ```js
   const r = await Promise.allSettled([Promise.resolve("a"), Promise.reject(new Error("b"))]);
   ```
4. Why does the `catch` not run here?
   ```js
   async function f() {
     try { return fail(10, "oops"); }
     catch (e) { return "caught"; }
   }
   ```
5. Why does `console.log("done")` print before the saves finish?
   ```js
   items.forEach(async (item) => { await save(item); });
   console.log("done");
   ```

<details><summary>Answers</summary>

1. About 600 ms, because each `await` waits for the previous one. Run them together: `await Promise.all([wait(200), wait(200), wait(200)])` takes about 200 ms.
2. `undefined`. The callback has braces and no `return`.
3. `[ { status: 'fulfilled', value: 'a' }, { status: 'rejected', reason: Error: b } ]`
4. `return fail(...)` returns the promise without awaiting it, so the rejection happens after the function has left the `try` block. Write `return await fail(...)`.
5. `forEach` ignores the promises returned by the `async` callback, so nothing waits for them. Use `for...of` with `await`, or `await Promise.all(items.map(...))`.
</details>
