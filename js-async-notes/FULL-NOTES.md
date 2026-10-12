# JavaScript Async + Node/Express Notes (Full, with examples)

Topics marked ⭐ are the ones that needed the most explanation. Read those twice.

Contents
1. Functions and arrow functions ⭐
2. `.map`, `.forEach`, `.filter` ⭐
3. Sync vs async callbacks, `setTimeout`
4. Why callbacks cause trouble
5. Promises ⭐
6. `async` / `await` ⭐
7. The event loop
8. Web API, language, or Node? ⭐
9. Express controllers and request data
10. `res.render` and EJS ⭐
11. Mongoose, sessions, favourites
12. Production checklist

---

## 1. Functions and arrow functions ⭐

A function is a saved set of instructions with a name.

```js
function add(a, b) { return a + b; }              // declaration
const add2 = function (a, b) { return a + b; };   // expression
const add3 = (a, b) => a + b;                     // arrow
```

```js
add        // the function itself (the recipe)
add(2, 3)  // RUN it now (cooking it)
```

### ⭐ The `return` rule

`=>` separates the inputs from the body.

```
(n) => n * 2
 ^      ^
inputs  body
```

| Style | Returns? | Example |
|---|---|---|
| No braces | The single expression is returned automatically (implicit return) | `(n) => n * 2` |
| With braces | Nothing, unless you write `return` | `(n) => { return n * 2; }` |

```js
const double  = (n) => n * 2;            // double(5) -> 10
const double2 = (n) => { return n * 2; };// double2(5) -> 10
const bad     = (n) => { n * 2; };       // bad(5) -> undefined (calculated, never returned)
```

**Calculating a value is not the same as returning it.**

Returning an object with the short form needs brackets:

```js
const wrong = (name) => { houseName: name };    // JS reads { } as a body -> undefined
const right = (name) => ({ houseName: name });  // returns { houseName: name }
```

### Hoisting

```js
console.log(add(2, 3));   // 5, declarations are available before their line
function add(a, b) { return a + b; }

console.log(mul(2, 3));   // ReferenceError: Cannot access 'mul' before initialization
const mul = (a, b) => a * b;
```

Declarations are hoisted. `const` and arrow functions are not, so define them before use.

---

## 2. `.map`, `.forEach`, `.filter` ⭐

**Callback** = a function passed to another function so that the other function can run it.

```js
function sayHi() { console.log("Hi!"); }
function doTwice(task) { task(); task(); }
doTwice(sayHi);   // no brackets after sayHi: pass the function, do not run it
```

```js
const nums = [1, 2, 3];

nums.map((n) => n * 2);       // [2, 4, 6]   NEW array of returned values
nums.filter((n) => n > 1);    // [2, 3]      keeps items where the callback returns true
nums.forEach((n) => n * 2);   // undefined   return values are ignored
```

### ⭐ `.map`

- Calls the callback once per item.
- Builds a **new array** from whatever the callback **returns**.
- The original array is **not changed**.
- The result is **lost** unless you store or use it.

```js
const original = [1, 2, 3];
const doubled = original.map((n) => n * 2);
console.log(doubled);    // [2, 4, 6]
console.log(original);   // [1, 2, 3] unchanged

[1, 2, 3].map((n) => n * 2);   // new array created, then thrown away (not stored)
```

### ⭐ `.forEach`

- Runs the callback per item and returns `undefined`.
- Use it for actions (print, save, send).

```js
const r = [1, 2, 3].forEach((n) => n * 2);
console.log(r);                                  // undefined
[1, 2, 3].forEach((n) => console.log(n * 2));    // prints 2, 4, 6
```

**Use `map` to build a new list. Use `forEach` to do something per item.**

### ⭐ Mistakes to remember

```js
[1, 2, 3].map((n) => n + 10);          // [11, 12, 13]
[1, 2, 3].map((n) => { n + 10; });     // [undefined, undefined, undefined]  (no return)
```

- `[ ]` is an **array** (a list). `{ }` is an **object** (key and value pairs).
- `.map` always gives an array, so `[11, 12, 13]`, three separate numbers.

### `.filter` with objects

```js
const homes = [
  { name: "Beach Villa", price: 5000 },
  { name: "City Flat",   price: 2500 },
  { name: "Hill Cabin",  price: 1800 },
];
homes.filter((h) => h.price < 3000);  // [City Flat, Hill Cabin]
homes.map((h) => h.name);             // ["Beach Villa", "City Flat", "Hill Cabin"]
```

---

## 3. Sync vs async callbacks, and `setTimeout`

| Kind | When it runs | Example |
|---|---|---|
| Synchronous callback | Right now, before the outer function finishes | `.map`, `.forEach` |
| Asynchronous callback | Later, after the current code finishes | `setTimeout` |

```js
console.log("A");
[1, 2].forEach((n) => console.log("item", n));
setTimeout(() => console.log("B"), 0);
console.log("C");
// A, item 1, item 2, C, B
```

### One thread, one call stack

JavaScript does one thing at a time. Slow work (timers, network, disk, database) is waited on **outside** the stack. When it finishes, its callback is queued and runs when the stack is empty.

```js
console.log("A");
setTimeout(() => console.log("B"), 1000);
console.log("C");
```

```
0 ms      prints A
0 ms      setTimeout handed to the environment, stack is free
0 ms      prints C
1000 ms   timer done -> callback queued -> stack empty -> prints B
```

- Slow things finish last, even if written in the middle.
- `setTimeout(fn, 0)` means "after the current code", not "now".
- The delay is a **minimum**. A busy stack makes it later.

```js
console.log("start");
setTimeout(() => console.log("timer"), 0);
const end = Date.now() + 2000;
while (Date.now() < end) {}     // blocks for 2 seconds
console.log("end");
// start, (2 s freeze) end, timer
```

### Blocking in a server

```js
app.get("/slow", (req, res) => {
  const end = Date.now() + 5000;
  while (Date.now() < end) {}    // blocks everyone
  res.send("done");
});
app.get("/fast", (req, res) => res.send("hi"));   // waits ~5 s if /slow was hit first
```

Never run a long synchronous loop in a request handler. In Node it freezes all users.

### Cancel and repeat

```js
const id = setTimeout(() => console.log("never"), 1000);
clearTimeout(id);

let count = 0;
const timer = setInterval(() => {
  count++;
  console.log("tick", count);
  if (count === 3) clearInterval(timer);
}, 500);
```

---

## 4. Why callbacks cause trouble

```js
getUser(userId, (err, user) => {
  if (err) return next(err);
  getFavourites(user, (err, favs) => {
    if (err) return next(err);
    getHomes(favs, (err, homes) => {
      if (err) return next(err);
      res.render("favourites", { homes });
    });
  });
});
```

1. **Callback hell:** nesting grows to the right.
2. The error check is repeated in every step.
3. `try/catch` cannot catch an error thrown later inside a callback.
4. It is hard to combine steps or run them in parallel.

```js
try {
  setTimeout(() => { throw new Error("boom"); }, 100);
} catch (e) {
  console.log("caught");   // never runs, the try block ended long before
}
```

Node's error-first convention: `callback(error, result)`. Check `err` first, then `return`.

```js
fs.readFile("homes.json", "utf8", (err, data) => {
  if (err) { console.log("failed:", err.code); return; }   // e.g. ENOENT
  console.log(data.length);
});
```

---

## 5. Promises ⭐

### ⭐ Why Promises exist

The same waiting, written flat, with one error handler:

```js
getUser(id)
  .then((user) => getFavourites(user))
  .then((favs) => getHomes(favs))
  .then((homes) => res.render("favourites", { homes }))
  .catch((err) => next(err));     // one catch covers every step above
```

### What a Promise is

An object meaning "the result is not ready yet, but it will be" (like a food-court buzzer).

| State | Meaning |
|---|---|
| pending | still waiting |
| fulfilled | worked, has a value |
| rejected | failed, has an error |

Fulfilled and rejected are together called **settled**. Once settled, a promise never changes.

- `.then(cb)` runs on success
- `.catch(cb)` runs on failure
- `.finally(cb)` runs either way

### ⭐ Chaining rule

`.then` returns a **new** promise. Inside a `.then` callback:

| You do | Next step gets |
|---|---|
| return a value | that value |
| return a promise | the chain waits, then gets its result |
| throw | jumps to the next `.catch` |
| return nothing | `undefined` |

```js
Promise.resolve(5)
  .then((n) => n * 2)            // 10
  .then((n) => n + 1)            // 11
  .then((n) => console.log(n));  // prints 11
```

### ⭐ Is the function inside `.then(...)` a callback? Yes.

A callback is any function passed in to be called later. `.then((response) => ...)` takes one. Promises do not remove callbacks. They make them flat and chainable.

### ⭐ `new Promise` and `resolve`

```js
new Promise((resolve, reject) => {
  // start slow work
  // call resolve(value) when done   -> fulfilled
  // call reject(error) if it fails  -> rejected
});
```

A promise starts **pending** (an empty box). `resolve` fills it.

```js
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```

Long form, same thing:

```js
function wait(ms) {
  return new Promise((resolve) => {
    setTimeout(() => { resolve(); }, ms);
  });
}
```

```js
async function demo() {
  console.log("start");
  await wait(1000);
  console.log("1 second later");
}
demo();
console.log("outside");
// start, outside, (after 1 second) 1 second later
```

- `setTimeout` is still callback-based. The Promise is a wrapper so we can `await` it.
- `outside` prints before `1 second later` because `await` pauses only `demo()`, not the whole program.

### Running promises together

```js
await Promise.all([a(), b()]);          // all succeed, or rejects at the first failure
await Promise.allSettled([a(), b()]);   // waits for all, never rejects, gives each result
```

---

## 6. `async` / `await` ⭐

```js
exports.getFavouriteList = async (req, res, next) => {
  const userId = req.session.user._id;                                 // stores the id
  const user = await User.findById(userId).populate("favourites");     // waits, then stores the user
  res.render("store/favourite-list", { favouriteHomes: user.favourites });
};
```

### ⭐ `async` does not store anything

`async` is a **label** that allows `await` inside the function. The storing is done by the `const` lines.

- `await promise` pauses only this function until the promise settles.
- It gives the fulfilled value, or **throws** if rejected.
- The rest of the program keeps running.
- It is syntax over promises, not a different mechanism.
- An `async` function **always returns a Promise**: `return x` fulfills with `x`, `throw e` rejects with `e`.

Same code, two styles:

```js
// .then style
Home.find().then((homes) => { console.log(homes); });

// await style
async function show() {
  const homes = await Home.find();
  console.log(homes);
}
```

Errors with `await`:

```js
try {
  const user = await User.findById(id);
} catch (err) {
  next(err);
}
```

`try/catch` only works if you `await` the promise.

---

## 7. The event loop

Order of execution:

1. Run all synchronous code until the call stack is empty.
2. Run **all** microtasks (`.then` callbacks, code after `await`).
3. Run **one** task (a `setTimeout` or I/O callback).
4. Repeat from step 2.

Microtasks always finish before the next task.

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
// 1, 4, 3, 2
```

```js
async function run() {
  console.log("A");
  await null;
  console.log("B");
}
run();
console.log("C");
// A, C, B   (the code after await is a microtask, so it waits until C has run)
```

Parts of the diagram:

- **Call stack:** where code runs, one function at a time.
- **Web APIs:** browser features that wait for you (timers, `fetch`, DOM events).
- **Task queue:** callbacks from timers and events.
- **Microtask queue:** promise callbacks, run first.
- **Event loop:** "stack empty? then take the next waiting callback."

---

## 8. Web API, language, or Node? ⭐

### ⭐ Is `fetch` or `setTimeout` a promise function or an API?

Both are **Web APIs**. They differ in how they hand you the result.

| | Provider | Style | Returns a Promise? |
|---|---|---|---|
| `setTimeout` | browser / Node | callback | No (returns a timer id) |
| `fetch` | browser / Node | promise | Yes |
| Geolocation | browser | callback | No |
| `Promise` | the language | not an API | It is the promise object |

- **API** = the feature (timer, network request, location).
- **Callback or Promise** = how the feature hands you its result.
- Callback-based: you pass the callback **into** the call.
- Promise-based: you get a promise back, then use `.then` or `await`.

Queues: a `setTimeout` callback goes to the **task** queue. A `.then` or `await` continuation goes to the **microtask** queue, which runs first.

### Who provides what

| Group | Examples |
|---|---|
| Language (everywhere) | `Promise`, `async/await`, `Array`, `Object`, `Map`, `Math`, `Date`, `JSON` |
| Browser APIs | DOM, `fetch`, `setTimeout`, `localStorage`, `WebSocket`, Geolocation, Notification, Clipboard, `URL`, `FormData`, `console` |
| Node APIs | `fs`, `http`, `path`, `os`, `crypto`, `events`, `process`, `require` |
| Libraries (npm) | Express, Mongoose, EJS, body-parser |

- Needs `window` or `document`: browser.
- Needs `require()` or `npm install`: Node or a library.
- Anything **async** comes from the environment, not the language.

### `fetch` pitfall

```js
const res = await fetch(url);
if (!res.ok) throw new Error("HTTP " + res.status);   // REQUIRED
const data = await res.json();                         // json() is also a promise
```

`fetch` only rejects on network failure. A 404 or 500 still **fulfills**, so you must check `res.ok`.

---

## 9. Express controllers and request data

Request flow:

```
Browser -> URL -> ROUTER -> CONTROLLER -> MODEL (data) -> VIEW (.ejs) -> HTML -> Browser
```

```js
exports.getHomes = (req, res, next) => { /* ... */ };
```

- `exports.x = ...` makes it importable from other files.
- `require("../models/home")` imports; `..` means one folder up.
- `req` = the request, `res` = the response tool, `next` = pass control on (`next(err)` goes to the error handler).

### Dynamic routes

```js
storeRouter.get("/homes/:homeId", storeController.getHomeDetails);
// URL /homes/123  ->  req.params.homeId === "123"   (always a string)
```

| Source | Where | Example |
|---|---|---|
| `req.params.homeId` | URL placeholder | `/homes/123` |
| `req.query.sort` | query string | `/homes?sort=price` |
| `req.body.id` | POST form body | needs `express.urlencoded({ extended: true })` |
| `req.session.user` | remembered per visitor | cookie holds only the session id |

### Pitfalls

- **Route order matters.** Put specific routes before `/homes/:homeId`.
- The 404 handler goes **last**.
- An invalid id makes `findById` return `null` or throw. Check it.
- `body-parser deprecated ... extended` is only a warning. Add `{ extended: true }`.
- `[nodemon] app crashed`: read the error **above** it, fix the file, save.

---

## 10. `res.render` and EJS ⭐

```js
res.render("store/favourite-list", { pageTitle: "My Favourites", homes: ["Villa", "Flat"] });
//         ^ template file                ^ each key becomes a variable in the template
```

```html
<!-- views/store/favourite-list.ejs -->
<h1><%= pageTitle %></h1>
<% homes.forEach((home) => { %>
  <p><%= home %></p>
<% }) %>
```

```html
<!-- HTML sent to the browser -->
<h1>My Favourites</h1>
<p>Villa</p>
<p>Flat</p>
```

- The controller is the kitchen with the ingredients. The `.ejs` file is the plate with blanks. `render` fills the blanks and sends the finished HTML.
- `<%= x %>` prints, **HTML-escaped** (safe, the default).
- `<%- x %>` prints raw HTML (dangerous with user input).
- `<% code %>` runs JavaScript and prints nothing.
- A variable missing from the data object causes a render error.
- `await` must come **before** `render`, or the data is not ready.

---

## 11. Mongoose, sessions, favourites

| Call | Meaning |
|---|---|
| `Home.find()` | all documents (promise) |
| `Home.findById(id)` | one document, or `null` |
| `user.save()` | write changes (needs `await`) |
| `.populate("favourites")` | replace stored ids with full documents |
| `_id` | MongoDB's unique id |

```js
// add favourite
if (!user.favourites.includes(homeId)) {     // avoids duplicates
  user.favourites.push(homeId);
  await user.save();
}
res.redirect("/favourites");                 // POST, then redirect

// remove favourite
user.favourites = user.favourites.filter((f) => f != homeId);   // keep all except homeId
```

A **session** is data the server remembers per visitor. The browser holds only a cookie with the session id, and `req.session.user` is looked up from it.

Weak spots in that code:

- No `try/catch`, so a database error hangs the request (Express 4).
- `req.session.user` is `undefined` when logged out, so `._id` crashes.
- `user` can be `null`.
- Better: protect routes with auth middleware, and set `isLoggedIn` / `user` once in `res.locals`.

---

## 12. Production checklist

1. **Forgot `await`:** `const user = User.findById(id)` makes `user` a Promise, not a user.
2. **`await` in `forEach` does not work.** Use `for...of`, or `await Promise.all(arr.map(...))`.
3. **Independent awaits run one after another.** Run them together with `Promise.all`.
4. **Express 4 async handlers:** use `try { } catch (err) { next(err); }`. (Express 5 forwards rejected promises automatically.)
5. **`fetch`:** check `res.ok`.
6. **`.map`** needs `return` when you use braces.
7. **Never block the stack** in a server.
8. **`try/catch`** only catches awaited promises, not errors thrown later in callbacks.
9. **Null checks:** `findById` can return `null`, `session.user` can be missing.
