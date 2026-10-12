# Write in your copy (short version)

Only what to handwrite. The full explanations and examples are in `FULL-NOTES.md`.
⭐ = a topic that needed extra care, so write it neatly and add the example.

---

## Page 1: Functions ⭐

```
Function = saved instructions.  add -> the function.  add(2,3) -> RUN it.

Arrow: (n) => n * 2

⭐ RETURN RULE
  no { }   -> returns automatically        (n) => n * 2
  with { } -> must write return            (n) => { return n * 2; }
  { } without return -> undefined          <- the trap

Object with short form:  (x) => ({ key: x })

Declarations are hoisted. const/arrow functions are not.
```

## Page 2: map / forEach / filter ⭐

```
Callback = a function passed to another function, to be run by it.
doTwice(sayHi)   <- no brackets: pass it, don't run it

⭐ .map(cb)     -> NEW array of what cb RETURNS. Original unchanged.
                  Store it or it is lost.
   .forEach(cb) -> just runs cb, returns undefined. Use for actions.
   .filter(cb)  -> keeps items where cb returns true.

map = BUILD a list.   forEach = DO something per item.

Example:  [1,2,3].map(n => n + 10)        -> [11, 12, 13]
          [1,2,3].map(n => { n + 10; })   -> [undefined x3]
[ ] = array     { } = object
```

## Page 3: Sync, async, setTimeout

```
Sync callback:  runs NOW        (.map)
Async callback: runs LATER      (setTimeout)

JS = ONE call stack. Slow work is waited on OUTSIDE the stack.
Callback is queued, runs when the stack is empty.

Slow things finish LAST.
setTimeout(fn, 0) = after current code, not now.
Delay is a MINIMUM.

Example:  log("1"); setTimeout(log "2", 1000); log("3")   -> 1, 3, 2

clearTimeout(id), clearInterval(id)
Blocking the stack in a server freezes ALL users.
```

## Page 4: Callback problems

```
1. Callback hell (nesting)
2. Error check repeated every step
3. try/catch can't catch errors thrown later in a callback
4. Hard to combine / run in parallel

Node callback: (err, result). Check err first, then return.
```

## Page 5: Promises ⭐

```
⭐ WHY: same waiting, written FLAT, ONE .catch for all steps.

  getUser(id).then(...).then(...).then(...).catch(...)

Promise = object meaning "result not ready yet, but will be."
States: pending -> fulfilled (value)  |  rejected (error)
.then(cb) success   .catch(cb) failure   .finally(cb) always

⭐ .then returns a NEW promise. In a .then callback:
   return value   -> next step gets it
   return promise -> chain waits
   throw          -> jumps to .catch
   return nothing -> undefined

⭐ The function inside .then IS a callback.
   Promises don't remove callbacks. They flatten them.

⭐ new Promise((resolve, reject) => { slow work; resolve(value) })
   starts pending. resolve() fills it -> fulfilled.

   const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
   setTimeout is still callback-based. The Promise is a WRAPPER so we can await it.

Promise.all([...])        all succeed, or rejects at first failure
Promise.allSettled([...]) waits for all, never rejects
```

## Page 6: async / await ⭐

```
⭐ async does NOT store anything. It is a LABEL allowing await inside.
   The const lines do the storing.

await promise:
  - pauses ONLY this function
  - gives the value, or THROWS if rejected
  - rest of program keeps running
async function ALWAYS returns a Promise.

  const user = await User.findById(id);
  try { ... await ... } catch (err) { next(err); }

try/catch only works if you AWAIT.
```

## Page 7: Event loop

```
1. Run all sync code until the stack is empty
2. Run ALL microtasks (.then, code after await)
3. Run ONE task (setTimeout callback)
4. Repeat from 2

log 1; setTimeout(log 2, 0); Promise.then(log 3); log 4   -> 1, 4, 3, 2

Call stack | Web APIs | Task queue | Microtask queue (first) | Event loop
```

## Page 8: Web API vs language vs Node ⭐

```
⭐ setTimeout: Web API, callback-based, NO promise
   fetch:      Web API, promise-based, returns a Promise
   Promise:    LANGUAGE object, not an API

API = the feature.   Callback / Promise = how it gives you the result.

Language: Promise, async/await, Array, Object, Map, Math, Date, JSON
Browser:  DOM, fetch, setTimeout, localStorage, WebSocket, Geolocation
Node:     fs, http, path, os, crypto, events, process, require
Libraries: Express, Mongoose, EJS

Async things come from the ENVIRONMENT, not the language.

fetch only rejects on network failure.
404 / 500 still fulfil -> check res.ok.
```

## Page 9: Express

```
Browser -> URL -> ROUTER -> CONTROLLER -> MODEL -> VIEW -> HTML -> Browser

req = request   res = response   next = pass on / next(err)

/homes/:homeId   -> req.params.homeId   (string)
?sort=price      -> req.query.sort
POST form        -> req.body.id         (needs express.urlencoded)
logged-in user   -> req.session.user

Route ORDER matters. 404 handler LAST.
Check findById for null.
```

## Page 10: res.render / EJS ⭐

```
⭐ res.render("store/page", { title: "X", homes })
            ^ template file   ^ keys become variables in the template

controller = kitchen (ingredients)   .ejs = plate with blanks
render fills the blanks, sends finished HTML.

<%= x %>  print, escaped (safe)
<%- x %>  print raw HTML (dangerous)
<%  code %>  run JS, print nothing

await BEFORE render.
```

## Page 11: Mongoose and favourites

```
find() all   findById(id) one or null   save() needs await
populate("favourites") replaces ids with full documents

add:    if (!favs.includes(id)) { push(id); await save() }
remove: favs = favs.filter(f => f != id)
POST then redirect.

Session = server remembers visitor; browser holds only a cookie id.
```

## Page 12: Production checklist

```
1. Forgot await -> you get a Promise, not the value
2. await in forEach doesn't work -> for...of or Promise.all(map)
3. Independent awaits -> Promise.all
4. Express 4 async -> try/catch + next(err)
5. fetch -> check res.ok
6. .map with { } needs return
7. Never block the stack in a server
8. try/catch only catches awaited promises
9. Null checks (findById, session.user)
```
