# Practice questions

Try each one before opening the answer. A wrong guess is useful, because it shows which note to reread.

## Set 1: Functions, `.map`, callbacks

1. What does `result` hold?
   ```js
   const result = [1, 2, 3].map((n) => n + 10);
   ```
2. What does `result` hold?
   ```js
   const result = [1, 2, 3].map((n) => { n + 10; });
   ```
3. What does this print?
   ```js
   console.log("X");
   setTimeout(() => console.log("Y"), 0);
   [1, 2].forEach((n) => console.log("item", n));
   console.log("Z");
   ```
4. What does this return?
   ```js
   [1, 2, 3].forEach((n) => n * 2);
   ```
5. What is wrong here?
   ```js
   const makeHome = (name) => { houseName: name };
   ```

<details><summary>Answers</summary>

1. `[11, 12, 13]`
2. `[undefined, undefined, undefined]`. There are braces and no `return`.
3. `X`, `item 1`, `item 2`, `Z`, `Y`. `forEach` is synchronous, `setTimeout` waits.
4. `undefined`. `forEach` ignores returned values.
5. JS reads `{ }` as a function body, so it returns `undefined`. Fix: `(name) => ({ houseName: name })`.
</details>

## Set 2: Async, timers, errors

6. What does this print, and why?
   ```js
   console.log("A");
   setTimeout(() => console.log("B"), 0);
   const end = Date.now() + 1000;
   while (Date.now() < end) {}
   console.log("C");
   ```
7. Why does the `catch` never run?
   ```js
   try {
     setTimeout(() => { throw new Error("boom"); }, 100);
   } catch (e) {
     console.log("caught");
   }
   ```
8. Why is a long synchronous loop worse in a Node server than in a one-off script?

<details><summary>Answers</summary>

6. `A`, then a 1 second freeze, `C`, then `B`. The timer was ready after 0 ms, but its callback cannot run until the stack is empty.
7. The callback runs later, after the `try` block has already finished, so nothing is around to catch the error. The process crashes instead.
8. One call stack serves every request. While the loop runs, all other users wait.
</details>

## Set 3: Promises and `async/await`

9. What does `.then()` return, and why does it matter?
10. What is printed?
    ```js
    Promise.resolve(5)
      .then((n) => n * 2)
      .then((n) => n + 1)
      .then((n) => console.log(n));
    ```
11. What is printed, and in what order?
    ```js
    async function run() {
      console.log("A");
      await null;
      console.log("B");
    }
    run();
    console.log("C");
    ```
12. What is the bug?
    ```js
    exports.getX = async (req, res) => {
      const user = User.findById(req.session.user._id);
      res.render("x", { favourites: user.favourites });
    };
    ```
13. Two `await` lines do not depend on each other. How do you speed them up?
14. In `const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));`, what does `resolve` do, and is `setTimeout` still callback-based?
15. What does `async` do? Does it store anything?

<details><summary>Answers</summary>

9. A **new promise**. That is why chaining works: each `.then` returns something you can attach another `.then` to.
10. `11`.
11. `A`, `C`, `B`. Code after `await` is a microtask, so it waits until the current synchronous code has finished.
12. `await` is missing. `user` is a Promise, so `user.favourites` is `undefined`.
13. `const [a, b] = await Promise.all([getA(), getB()]);`
14. `resolve` moves the promise from pending to fulfilled, which lets whoever is awaiting it continue. `setTimeout` is still callback-based. The Promise is a wrapper around it.
15. `async` is only a label that allows `await` inside the function, and it makes the function return a Promise. It stores nothing. The `const` lines store values.
</details>

## Set 4: Web APIs and `fetch`

16. Is `fetch` a Web API or a Promise function?
17. Does `fetch("/missing-page")` reject on a 404? What must you do?
18. Which queue does a `setTimeout` callback go to, and which does a `.then` callback go to? Which runs first?
19. Is the arrow function inside `.then(...)` a callback?

<details><summary>Answers</summary>

16. It is a **Web API** (provided by the browser or Node) that is **promise-based**, so it returns a Promise. `Promise` itself is part of the language.
17. No. `fetch` only rejects on network failure. A 404 still fulfills, so check `res.ok`.
18. `setTimeout` goes to the task queue, `.then` goes to the microtask queue. Microtasks run first.
19. Yes. A callback is any function passed in to be called later.
</details>

## Set 5: Express

20. In `/homes/:homeId`, where does `homeId` end up in the controller, and what type is it?
21. Name two things in `getFavouriteList` that can crash it in production.
22. What is the difference between `<%= %>` and `<%- %>`? Which is safer?
23. Why does route order matter?

<details><summary>Answers</summary>

20. `req.params.homeId`, and it is always a **string**.
21. Any two of: no `try/catch` (a database error hangs the request), `req.session.user` is `undefined` when logged out, `user` can be `null`.
22. `<%= %>` prints HTML-escaped text (safe). `<%- %>` prints raw HTML (dangerous with user input). `<%= %>` is safer.
23. Express matches top to bottom. A dynamic route like `/homes/:homeId` placed before `/homes/add` would treat `add` as an id.
</details>
