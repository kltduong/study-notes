# Callback Problems & HTTP

**TL;DR:** Callbacks break down when async operations depend on each other — nested chains are hard to follow, error handling is duplicated at every level, and parallel coordination requires manual bookkeeping. `XMLHttpRequest` (XHR) was the original browser HTTP API and a prime source of callback hell. These pain points motivated promises.

## Why Callbacks Break Down

A single async callback is fine. The problems surface when operations **depend on each other** — you need the result of one to start the next:

```js
getUser(id, (err, user) => {
  if (err) return handleError(err);
  getPosts(user.id, (err, posts) => {
    if (err) return handleError(err);
    getComments(posts[0].id, (err, comments) => {
      if (err) return handleError(err);
      render(user, posts, comments);
    });
  });
});
```

This is **callback hell** (the "pyramid of doom"). Three structural problems, none of which are about indentation:

### 1. Composition is inside-out

Each step nests inside the previous one's callback. The logic reads right-to-left and inside-out. Extracting named functions flattens the visual nesting but doesn't change the coupling — you still have to jump between definitions to trace the flow.

### 2. Error handling is scattered

Every level needs its own `if (err)` check. Miss one and errors silently vanish. There's no way to catch all errors in one place — no equivalent of a single `try/catch` wrapping the whole sequence.

### 3. Inversion of control

You hand your callback to someone else's function and trust it to:

- Call it exactly once
- Pass the right arguments
- Call it at the right time

Nothing enforces that contract. The receiving function could call it twice, never call it, or call it synchronously when you expected async. You have no control.

### Parallel coordination is manual

Two independent async operations that both need to finish before continuing:

```js
let user = null;
let notifications = null;

function tryRender() {
  if (user && notifications) render(user, notifications);
}

getUser(id, (err, u) => {
  user = u;
  tryRender();
});
getNotifications(id, (err, n) => {
  notifications = n;
  tryRender();
});
```

Shared mutable state, a manual gate function, and error handling bolted on separately for each call. It works, but it's bookkeeping you write and maintain yourself.

## XMLHttpRequest (XHR)

The original browser API for HTTP requests. Entirely callback-based:

```js
const xhr = new XMLHttpRequest();
xhr.open("GET", "https://api.example.com/users/1");

xhr.onload = function () {
  if (xhr.status === 200) {
    const user = JSON.parse(xhr.responseText);
    console.log(user);
  }
};

xhr.onerror = function () {
  console.error("Network error");
};

xhr.send();
```

Key details:

- Create an object, configure it, attach callbacks, call `send()`.
- Success and error are **separate callbacks** (`onload` vs `onerror`).
- `onload` fires for **any completed HTTP response** — 200, 404, 500. You must check `xhr.status` yourself.
- `onerror` fires only for **network-level failures** (DNS failure, connection refused, no internet).
- Despite the name, not limited to XML — handles any HTTP request.

`fetch()` eventually replaced XHR with a promise-based API, eliminating most of this ceremony.

## Callback Hell with XHR

Sequential XHR calls — get user → get posts → get comments — produce the classic pyramid:

```js
function getUser(id, callback) {
  const xhr = new XMLHttpRequest();
  xhr.open("GET", `/api/users/${id}`);
  xhr.onload = function () {
    if (xhr.status === 200) {
      callback(null, JSON.parse(xhr.responseText));
    } else {
      callback(new Error(`User fetch failed: ${xhr.status}`));
    }
  };
  xhr.onerror = () => callback(new Error("Network error"));
  xhr.send();
}

// Same pattern repeated for getPosts, getComments...

getUser(1, (err, user) => {
  if (err) return console.error(err);
  getPosts(user.id, (err, posts) => {
    if (err) return console.error(err);
    getComments(posts[0].id, (err, comments) => {
      if (err) return console.error(err);
      render(user, posts, comments);
    });
  });
});
```

Each wrapper function duplicates the XHR boilerplate. Each consumer level duplicates error handling. Adding retry logic or a fourth step makes it exponentially worse. This is the exact pain point that motivated promises: **composable, trustable async with centralized error handling**.
