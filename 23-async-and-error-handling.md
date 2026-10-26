# Async TypeScript and Error Handling

> **Session 23 of 33** | Part 6: Real Programs | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Typing async Functions](#typing-async-functions)
- [Forgetting await](#forgetting-await)
- [Promises Nobody Waits For](#promises-nobody-waits-for)
- [Creating Promises](#creating-promises)
- [Errors in Async Code](#errors-in-async-code)
- [Running Tasks at the Same Time](#running-tasks-at-the-same-time)
- [Awaited: Unwrapping a Promise Type](#awaited-unwrapping-a-promise-type)
- [Wrapping Errors with cause](#wrapping-errors-with-cause)
- [Typing fetch Results](#typing-fetch-results)
- [Mini Example 1: Retry with Backoff](#mini-example-1-retry-with-backoff)
- [Mini Example 2: An Order Page with a Time Limit](#mini-example-2-an-order-page-with-a-time-limit)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `23-async` inside `ts-practice`, with the one-line `tsconfig.json`. Files run directly with `node file.ts` again; only Session 22's decorators needed a build step

23-async/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

This session builds on promises and `async`/`await` from the JavaScript course ([Session 24](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/24-promises.md) and [Session 25](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/25-async-await-and-fetch.md)), and on `catch (err)` being `unknown` (Session 04).

---

## Typing async Functions

An `async` function always returns a promise. Its return type says what the promise will hold, with `Promise<T>`

orders.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";

interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

const orders: Order[] = [
  { id: 1001, customer: "Asha", totalPaise: 249900 },
  { id: 1002, customer: "Ravi", totalPaise: 79900 }
];

async function findOrder(id: number): Promise<Order | undefined> {
  await sleep(50);
  return orders.find((order) => order.id === id);
}

async function isInStock(productId: number): Promise<boolean> {
  await sleep(20);
  return productId !== 2;
}

const order = await findOrder(1001);
console.log(order);
console.log(await findOrder(9999));
console.log(await isInStock(2));
```

```bash
npx tsc --noEmit
node orders.ts
```

```text
{ id: 1001, customer: 'Asha', totalPaise: 249900 }
undefined
false
```

- `sleep` is the promise version of `setTimeout` from `node:timers/promises` (JavaScript course, Session 24). It stands in for a database or another service.
- `findOrder` returns `Promise<Order | undefined>`: the order may not exist, and the type says so (Session 11).
- `await` takes the value out of the promise: `order` is an `Order | undefined`, not a promise. The file is an ES module, so `await` works at the top level.

The return type of an `async` function must be a `Promise`. Add a function that forgets that, at the end

```typescript
async function countOrders(): number {
  await sleep(10);
  return orders.length;
}
```

```text
orders.ts:29:31 - error TS1064: The return type of an async function or method must be the global Promise<T> type. Did you mean to write 'Promise<number>'?

29 async function countOrders(): number {
                                 ~~~~~~


Found 1 error in orders.ts:29
```

`tsc` even says what to write. Inside the function you still `return orders.length`, a plain number: `async` wraps it in the promise for you.

![An async function is drawn as a box that hands out a sealed Promise<Order | undefined> envelope; await opens the envelope and takes out the Order itself](images/23-async-and-error-handling/promise-type.gif)

---

## Forgetting await

Without `await`, you hold the promise, not the value. TypeScript notices as soon as you use it. Replace `countOrders` with a `checkout` function that forgets both `await`s

```typescript
async function checkout(orderId: number): Promise<string> {
  const found = findOrder(orderId);
  if (isInStock(1)) {
    return `${found.customer} pays ${found.totalPaise} paise`;
  }
  return "out of stock";
}
```

```text
orders.ts:31:7 - error TS2801: This condition will always return true since this 'Promise<boolean>' is always defined.

31   if (isInStock(1)) {
         ~~~~~~~~~~~~

  orders.ts:31:7 - Did you forget to use 'await'?
    31   if (isInStock(1)) {
             ~~~~~~~~~~~~

orders.ts:32:21 - error TS2339: Property 'customer' does not exist on type 'Promise<Order | undefined>'.

32     return `${found.customer} pays ${found.totalPaise} paise`;
                       ~~~~~~~~

orders.ts:32:44 - error TS2339: Property 'totalPaise' does not exist on type 'Promise<Order | undefined>'.

32     return `${found.customer} pays ${found.totalPaise} paise`;
                                              ~~~~~~~~~~


Found 3 errors in the same file, starting at: orders.ts:31
```

The first error is the dangerous one in plain JavaScript: a promise is an object, and an object is always truthy, so `if (isInStock(1))` would **always** take the first branch, even for a product that is out of stock. `tsc` calls it out and asks "Did you forget to use 'await'?". The other two errors say the same thing in another way: `found` is a `Promise<Order | undefined>`, which has no `customer`.

With both `await`s, TypeScript also reminds you that the order may be missing (Session 11), so the working version checks it

```typescript
async function checkout(orderId: number): Promise<string> {
  const found = await findOrder(orderId);
  if (found === undefined) {
    return `order ${orderId} not found`;
  }
  const inStock = await isInStock(1);
  if (!inStock) {
    return "out of stock";
  }
  return `${found.customer} pays ${found.totalPaise} paise`;
}

console.log(await checkout(1002));
console.log(await checkout(1003));
```

```text
{ id: 1001, customer: 'Asha', totalPaise: 249900 }
undefined
false
Ravi pays 79900 paise
order 1003 not found
```

![Two calls without await hand back sealed promises: using the promise in an if is always true, and reading customer from it fails; tsc marks both and suggests await, and with await the real values come out](images/23-async-and-error-handling/forgot-await.gif)

---

## Promises Nobody Waits For

`tsc` catches a forgotten `await` when you **use** the value. When you do not use it at all, it has nothing to check

payment.ts

```typescript
async function chargeCard(amountPaise: number): Promise<string> {
  if (amountPaise > 100000) {
    throw new Error(`card declined for ${amountPaise} paise`);
  }
  return `charged ${amountPaise} paise`;
}

chargeCard(249900);
console.log("order saved");
```

```bash
npx tsc --noEmit
node payment.ts
```

```text
order saved
file:///D:/ts-practice/23-async/payment.ts:3
    throw new Error(`card declined for ${amountPaise} paise`);
          ^

Error: card declined for 249900 paise
    at chargeCard (file:///D:/ts-practice/23-async/payment.ts:3:11)
    at file:///D:/ts-practice/23-async/payment.ts:8:1
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` is quiet, the program says "order saved" for an order that was never paid, and then Node.js stops with an unhandled rejection (JavaScript course, Session 24). A promise that nobody awaits is called a **floating promise**. The same happens with `forEach` and an `async` callback

receipts.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";

async function sendReceipt(orderId: number): Promise<void> {
  await sleep(50);
  console.log(`receipt sent for ${orderId}`);
}

const orderIds = [1001, 1002];
orderIds.forEach(async (id) => {
  await sendReceipt(id);
});
console.log("all receipts sent");
```

```text
all receipts sent
receipt sent for 1001
receipt sent for 1002
```

`forEach` calls the callback and throws away the promise it returns, so "all receipts sent" comes first. A `for...of` loop waits

```typescript
for (const id of orderIds) {
  await sendReceipt(id);
}
```

```text
receipt sent for 1001
receipt sent for 1002
all receipts sent
```

TypeScript itself has no check for floating promises. A lint rule does it (`no-floating-promises`, Session 29). Until then, the habit: every call to an `async` function gets an `await`, a `return`, or a `.catch()`.

![A timeline: chargeCard starts but nobody waits for it; order saved is printed at once; a moment later the rejected promise has no handler and Node.js stops the program](images/23-async-and-error-handling/floating-promise.gif)

---

## Creating Promises

Most promises come from `async` functions and from libraries. To turn an old callback-style function into a promise, use `new Promise` (JavaScript course, Session 24). Here Node.js's `randomBytes`, which takes a callback, becomes a function that returns `Promise<string>`

token.ts

```typescript
import { randomBytes } from "node:crypto";

function makeToken(bytes: number): Promise<string> {
  return new Promise((resolve, reject) => {
    randomBytes(bytes, (err, buffer) => {
      if (err) {
        reject(err);
        return;
      }
      resolve(buffer.toString("hex"));
    });
  });
}

const token = await makeToken(8);
console.log(token.length, /^[0-9a-f]+$/.test(token));
```

```text
16 true
```

`makeToken` is not `async`, but it returns a promise, so its return type is `Promise<string>`. From that, TypeScript knows that `resolve` takes a `string`. Resolve with something else

```typescript
      resolve(buffer.length);
```

```text
token.ts:10:15 - error TS2345: Argument of type 'number' is not assignable to parameter of type 'string | PromiseLike<string>'.

10       resolve(buffer.length);
                 ~~~~~~~~~~~~~


Found 1 error in token.ts:10
```

Two more things to know about `new Promise`:

- Without a return type or a type argument, nothing tells TypeScript what the promise holds, and it becomes `Promise<unknown>`. Write `new Promise<string>(...)` when there is no return type to go by.
- A promise that holds nothing is `Promise<void>`. `new Promise<number>((resolve) => ...)` with a plain `resolve()` gives `error TS2794: Expected 1 arguments, but got 0. Did you forget to include 'void' in your type argument to 'Promise'?`

![The callback function randomBytes is wrapped in new Promise: its error goes to reject and its buffer, turned into text, goes to resolve, so the caller gets a Promise<string> and can await it](images/23-async-and-error-handling/new-promise.gif)

---

## Errors in Async Code

With `await`, errors from a promise arrive in a normal `try...catch`. As in Session 04, `err` in `catch` is `unknown`, so use the `errorMessage` helper from that session. Put it in its own file, so that every file in this session can import it

errors.ts

```typescript
export function errorMessage(err: unknown): string {
  if (err instanceof Error) {
    return err.message;
  }
  if (typeof err === "string") {
    return err;
  }
  return "Unknown error";
}
```

place-order.ts

```typescript
import { errorMessage } from "./errors.ts";

async function chargeCard(amountPaise: number): Promise<string> {
  if (amountPaise > 100000) {
    throw new Error(`card declined for ${amountPaise} paise`);
  }
  return `charged ${amountPaise} paise`;
}

async function placeOrder(amountPaise: number): Promise<string> {
  try {
    const receipt = await chargeCard(amountPaise);
    return `order placed, ${receipt}`;
  } catch (err) {
    return `order failed: ${errorMessage(err)}`;
  }
}

console.log(await placeOrder(49900));
console.log(await placeOrder(249900));
```

```text
order placed, charged 49900 paise
order failed: card declined for 249900 paise
```

Note what `Promise<string>` does **not** say: which errors the promise can reject with. A promise can reject with anything, so TypeScript has no type for it, and the safe type for an error is always `unknown`.

There is one place where TypeScript is less careful. Add a `.catch()` call at the end, with a typo

```typescript
chargeCard(249900).catch((err) => console.log("caught:", err.mesage));
```

```text
order placed, charged 49900 paise
order failed: card declined for 249900 paise
caught: undefined
```

`tsc` is quiet, and the message is lost. TypeScript's own type for `.catch()` gives the callback's parameter the type `any`, not `unknown`. (The `reason` of a rejected result in `Promise.allSettled`, in the next section, is `any` too.) Give it the type yourself

```typescript
chargeCard(249900).catch((err: unknown) => console.log("caught:", err.mesage));
```

```text
place-order.ts:22:67 - error TS18046: 'err' is of type 'unknown'.

22 chargeCard(249900).catch((err: unknown) => console.log("caught:", err.mesage));
                                                                     ~~~


Found 1 error in place-order.ts:22
```

```typescript
chargeCard(249900).catch((err: unknown) => console.log("caught:", errorMessage(err)));
```

```text
order placed, charged 49900 paise
order failed: card declined for 249900 paise
caught: card declined for 249900 paise
```

![A thrown error reaches two kinds of handlers: catch (err) in try...catch receives it as unknown and must be checked, while the callback of .catch() receives it as any, so a typo like err.mesage slips through until the parameter is written as err: unknown](images/23-async-and-error-handling/catch-types.gif)

---

## Running Tasks at the Same Time

`Promise.all` runs several promises at the same time and waits for all of them (JavaScript course, Session 24). TypeScript keeps track of each result's type, in order

parallel.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";

interface Customer {
  id: number;
  name: string;
}

async function getCustomer(id: number): Promise<Customer> {
  await sleep(100);
  return { id, name: "Asha" };
}

async function countOrders(customerId: number): Promise<number> {
  await sleep(150);
  return customerId === 7 ? 3 : 0;
}

async function getCity(customerId: number): Promise<string> {
  await sleep(120);
  return customerId === 7 ? "Pune" : "unknown";
}

const started = Date.now();
const [customer, orderCount, city] = await Promise.all([getCustomer(7), countOrders(7), getCity(7)]);
const took = Date.now() - started;

console.log(`${customer.name} from ${city} has ${orderCount} orders`);
console.log(took < 250 ? "ran at the same time" : "ran one after another");
```

```text
Asha from Pune has 3 orders
ran at the same time
```

The result of `Promise.all` is a **tuple** (Session 06): `[Customer, number, string]`. So `customer` is a `Customer`, `orderCount` a `number` and `city` a `string`, each in its own place. Use one the wrong way, at the end

```typescript
console.log(orderCount.toUpperCase());
```

```text
parallel.ts:29:24 - error TS2339: Property 'toUpperCase' does not exist on type 'number'.

29 console.log(orderCount.toUpperCase());
                          ~~~~~~~~~~~


Found 1 error in parallel.ts:29
```

`Promise.all` rejects as soon as one promise rejects. When you want every result, also the failures, use `Promise.allSettled`. Each result is a discriminated union (Session 10) on `status`

settled.ts

```typescript
import { errorMessage } from "./errors.ts";

async function chargeCard(amountPaise: number): Promise<string> {
  if (amountPaise > 100000) {
    throw new Error(`card declined for ${amountPaise} paise`);
  }
  return `charged ${amountPaise} paise`;
}

const results = await Promise.allSettled([chargeCard(49900), chargeCard(249900), chargeCard(19900)]);

for (const result of results) {
  if (result.status === "fulfilled") {
    console.log("ok:", result.value);
  } else {
    console.log("failed:", errorMessage(result.reason));
  }
}
```

```text
ok: charged 49900 paise
failed: card declined for 249900 paise
ok: charged 19900 paise
```

A fulfilled result has a `value`, a rejected one a `reason`. Only after the `status` check can you use the right one. Add at the end

```typescript
console.log(results[0]?.value);
```

```text
settled.ts:20:25 - error TS2339: Property 'value' does not exist on type 'PromiseSettledResult<string>'.
  Property 'value' does not exist on type 'PromiseRejectedResult'.

20 console.log(results[0]?.value);
                           ~~~~~


Found 1 error in settled.ts:20
```

`reason` is `any`, like the `.catch()` parameter, so it goes through `errorMessage`.

![Three tasks start at the same time as bars of different lengths; Promise.all waits for the longest one and returns a tuple whose three places keep their own types: Customer, number and string](images/23-async-and-error-handling/promise-all.gif)

---

## Awaited: Unwrapping a Promise Type

Session 17 promised one more utility type. `ReturnType` (Session 17) of an `async` function gives the promise; `Awaited` takes the value out of it, the type-level version of `await`. It is useful when a function's result type has no name, which is common in code you did not write

dashboard.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";

async function loadDashboard(userId: number) {
  await sleep(10);
  return { userId, unread: 3, plan: "pro" as const };
}

type Dashboard = Awaited<ReturnType<typeof loadDashboard>>;

function renderDashboard(data: Dashboard): string {
  return `user ${data.userId}: ${data.unread} unread (${data.plan})`;
}

console.log(renderDashboard(await loadDashboard(7)));
```

```text
user 7: 3 unread (pro)
```

`loadDashboard` has no return type on purpose here, to play the part of a function from a library. Use the "surely wrong value" trick from Session 18 to see the types. Add at the end

```typescript
const a: ReturnType<typeof loadDashboard> = "x";
const b: Awaited<ReturnType<typeof loadDashboard>> = "x";
const c: Awaited<Promise<Promise<number>>> = "x";
```

```text
dashboard.ts:16:7 - error TS2322: Type 'string' is not assignable to type 'Promise<{ userId: number; unread: number; plan: "pro"; }>'.

16 const a: ReturnType<typeof loadDashboard> = "x";
         ~

dashboard.ts:17:7 - error TS2322: Type 'string' is not assignable to type '{ userId: number; unread: number; plan: "pro"; }'.

17 const b: Awaited<ReturnType<typeof loadDashboard>> = "x";
         ~

dashboard.ts:18:7 - error TS2322: Type 'string' is not assignable to type 'number'.

18 const c: Awaited<Promise<Promise<number>>> = "x";
         ~


Found 3 errors in the same file, starting at: dashboard.ts:16
```

`ReturnType` alone is the promise; `Awaited` is the object inside. It unwraps nested promises too, just like `await` does. Without `Awaited`, `Dashboard` would be the promise, and every `data.userId` in `renderDashboard` would give `error TS2339` with "Did you forget to use 'await'?".

![Awaited peels a type like layers: Promise of Promise of number loses one Promise layer, then the next, until only number is left; the same happens to the return type of loadDashboard](images/23-async-and-error-handling/awaited-type.gif)

---

## Wrapping Errors with cause

A low-level error like "file not found" is too technical for the caller, but it is exactly what you need when you debug. The answer is to throw a clearer error and keep the original one as its **cause**. `Error` takes it as a second argument: `new Error(message, { cause })`. Add `AppError` (Session 21) to the top of `errors.ts`, with that option passed through to `super`

```typescript
export class AppError extends Error {
  readonly statusCode: number;

  constructor(message: string, statusCode: number, options?: ErrorOptions) {
    super(message, options);
    this.name = "AppError";
    this.statusCode = statusCode;
  }
}
```

`ErrorOptions` is a type built into TypeScript: `{ cause?: unknown }`. And add a helper at the end of `errors.ts` that follows the chain of causes

```typescript
export function errorChain(err: unknown): string {
  const messages: string[] = [];
  let current: unknown = err;
  while (current !== undefined) {
    messages.push(errorMessage(current));
    current = current instanceof Error ? current.cause : undefined;
  }
  return messages.join(" <- ");
}
```

A settings loader can now fail with clear messages and keep the details. It reads files with `readFile` from `node:fs/promises` (Node.js course). Create two files next to it: `settings.json` with `{ "port": 3000, "currency": "INR" }`, and `broken.json` with `{ "port": 3000, }` (a trailing comma is not allowed in JSON)

settings.ts

```typescript
import { readFile } from "node:fs/promises";
import { AppError, errorChain } from "./errors.ts";

async function loadSettings(path: string): Promise<unknown> {
  let text: string;
  try {
    text = await readFile(path, "utf8");
  } catch (err) {
    throw new AppError(`could not read ${path}`, 500, { cause: err });
  }
  try {
    return JSON.parse(text);
  } catch (err) {
    throw new AppError(`${path} is not valid JSON`, 500, { cause: err });
  }
}

for (const path of ["settings.json", "broken.json", "missing.json"]) {
  try {
    console.log(path, await loadSettings(path));
  } catch (err) {
    console.log(errorChain(err));
  }
}
```

```text
settings.json { port: 3000, currency: 'INR' }
broken.json is not valid JSON <- Expected double-quoted property name in JSON at position 16 (line 1 column 17)
could not read missing.json <- ENOENT: no such file or directory, open 'D:\ts-practice\23-async\missing.json'
```

Each line starts with the message for the caller, and ends with the technical cause. `loadSettings` returns `Promise<unknown>`, because the settings come from a file: Session 26 checks them properly. And `let text: string;` without a value is fine here: `tsc` sees that the first `catch` always throws, so `text` has a value after it.

`cause` is `unknown`, for the same reason as `err` in `catch`: anything can be a cause. Replace the `errorChain` line with a direct use

```typescript
    console.log(err instanceof AppError ? err.cause.message : err);
```

```text
settings.ts:22:43 - error TS18046: 'err.cause' is of type 'unknown'.

22     console.log(err instanceof AppError ? err.cause.message : err);
                                             ~~~~~~~~~


Found 1 error in settings.ts:22
```

Node.js prints the cause too. Put `errorChain` back, and add an uncaught call at the end

```typescript
await loadSettings("missing.json");
```

```text
settings.json { port: 3000, currency: 'INR' }
broken.json is not valid JSON <- Expected double-quoted property name in JSON at position 16 (line 1 column 17)
could not read missing.json <- ENOENT: no such file or directory, open 'D:\ts-practice\23-async\missing.json'
file:///D:/ts-practice/23-async/settings.ts:9
    throw new AppError(`could not read ${path}`, 500, { cause: err });
          ^

AppError: could not read missing.json
    at loadSettings (file:///D:/ts-practice/23-async/settings.ts:9:11)
    at async file:///D:/ts-practice/23-async/settings.ts:26:1 {
  statusCode: 500,
  [cause]: Error: ENOENT: no such file or directory, open 'D:\ts-practice\23-async\missing.json'
      at async open (node:internal/fs/promises:697:25)
      at async readFile (node:internal/fs/promises:1348:14)
      at async loadSettings (file:///D:/ts-practice/23-async/settings.ts:7:12)
      at async file:///D:/ts-practice/23-async/settings.ts:26:1 {
    errno: -4058,
    code: 'ENOENT',
    syscall: 'open',
    path: 'D:\\ts-practice\\23-async\\missing.json'
  }
}

Node.js v24.19.0
```

The `[cause]` part keeps the original error with its own stack: you can see that it came from `readFile`.

![An ENOENT error from readFile is caught and wrapped in an AppError could not read missing.json, which keeps it as its cause; errorChain walks the chain from the outer error to the inner one and joins the messages](images/23-async-and-error-handling/error-cause.gif)

---

## Typing fetch Results

`fetch` (JavaScript course, Session 25) returns `Promise<Response>`, and `response.json()` returns `Promise<any>`. That `any` is a hole in your types. To try it without a server, give the function the fetch function as a parameter (dependency injection, Session 21), and pass a fake one that answers with a real `Response` object

products.ts

```typescript
import { AppError, errorChain } from "./errors.ts";

type Fetcher = (url: string) => Promise<Response>;

interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

async function getProduct(id: number, fetcher: Fetcher): Promise<Product> {
  const res = await fetcher(`http://localhost:4000/products/${id}`);
  return await res.json();
}

const fakeFetch: Fetcher = async (url) => {
  if (url.endsWith("/1")) {
    return new Response(JSON.stringify({ id: 1, name: "Keyboard", pricePaise: 149900 }));
  }
  if (url.endsWith("/2")) {
    return new Response(JSON.stringify({ id: 2, name: "Mouse", price: 499 }));
  }
  return new Response(JSON.stringify({ error: "not found" }), { status: 404 });
};

const mouse = await getProduct(2, fakeFetch);
console.log(mouse.name, (mouse.pricePaise / 100).toFixed(2));
```

```text
Mouse NaN
```

`tsc` was quiet. `res.json()` is `any`, so returning it as a `Product` is allowed, although the API sent `price` instead of `pricePaise`. The same happens with every API that changes, has a bug, or answers with an error page. Treat the body as `unknown`, check `res.ok` (JavaScript course, Session 25), and check the data with a type predicate (Session 10) before you call it a `Product`. Replace `getProduct` and the last two lines

```typescript
function isProduct(value: unknown): value is Product {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value && typeof value.id === "number" &&
    "name" in value && typeof value.name === "string" &&
    "pricePaise" in value && typeof value.pricePaise === "number"
  );
}

async function getProduct(id: number, fetcher: Fetcher): Promise<Product> {
  const res = await fetcher(`http://localhost:4000/products/${id}`);
  if (!res.ok) {
    throw new AppError(`product ${id}: the API answered ${res.status}`, res.status);
  }
  const data: unknown = await res.json();
  if (!isProduct(data)) {
    throw new AppError(`product ${id}: unexpected data from the API`, 502);
  }
  return data;
}
```

```typescript
for (const id of [1, 2, 3]) {
  try {
    const product = await getProduct(id, fakeFetch);
    console.log(`${product.name}: Rs ${(product.pricePaise / 100).toFixed(2)}`);
  } catch (err) {
    console.log(errorChain(err));
  }
}
```

```text
Keyboard: Rs 1499.00
product 2: unexpected data from the API
product 3: the API answered 404
```

`const data: unknown = await res.json()` turns the `any` into `unknown` at the moment it enters your code. After that, nothing reaches the rest of the program unchecked. Writing such checks by hand gets long; Session 26 does the same with Joi.

The real `fetch` fits the `Fetcher` type, so `getProduct(1, fetch)` works against a real API, such as the JavaScript course's practice API on port 4000. Tests keep using a fake one. Session 29 does exactly that with Vitest.

![Two paths from res.json(): as any, the wrong body flows straight into a Product and prints NaN; as unknown, the body must pass the isProduct check, and the wrong body is stopped with unexpected data from the API](images/23-async-and-error-handling/typed-fetch.gif)

---

## Mini Example 1: Retry with Backoff

Other services fail now and then: a database is busy, a payment provider times out. A common answer is to try again, waiting a little longer each time (**backoff**). The JavaScript course retried with a loop (Session 25); here the retry is a typed, generic helper that works for any task

retry.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import { AppError, errorChain, errorMessage } from "./errors.ts";

interface RetryOptions {
  attempts: number;
  delayMs: number;
}

async function retry<T>(task: () => Promise<T>, { attempts, delayMs }: RetryOptions): Promise<T> {
  let lastError: unknown;
  for (let attempt = 1; attempt <= attempts; attempt++) {
    try {
      return await task();
    } catch (err) {
      lastError = err;
      const wait = delayMs * 2 ** (attempt - 1);
      console.log(`  attempt ${attempt} failed: ${errorMessage(err)}`);
      if (attempt < attempts) {
        console.log(`  waiting ${wait} ms`);
        await sleep(wait);
      }
    }
  }
  throw new AppError(`gave up after ${attempts} attempts`, 503, { cause: lastError });
}

let inventoryCalls = 0;

async function reserveStock(sku: string, quantity: number): Promise<string> {
  inventoryCalls += 1;
  await sleep(10);
  if (inventoryCalls < 3) {
    throw new Error(`inventory service busy (call ${inventoryCalls})`);
  }
  return `reserved ${quantity} x ${sku}`;
}

async function chargeCard(amountPaise: number): Promise<string> {
  await sleep(10);
  throw new Error(`card declined for ${amountPaise} paise`);
}

console.log("reserving stock:");
const reservation = await retry(() => reserveStock("KB-101", 2), { attempts: 4, delayMs: 100 });
console.log(reservation);

console.log("charging the card:");
try {
  await retry(() => chargeCard(249900), { attempts: 3, delayMs: 50 });
} catch (err) {
  console.log(errorChain(err));
}
```

```bash
npx tsc --noEmit
node retry.ts
```

```text
reserving stock:
  attempt 1 failed: inventory service busy (call 1)
  waiting 100 ms
  attempt 2 failed: inventory service busy (call 2)
  waiting 200 ms
reserved 2 x KB-101
charging the card:
  attempt 1 failed: card declined for 249900 paise
  waiting 50 ms
  attempt 2 failed: card declined for 249900 paise
  waiting 100 ms
  attempt 3 failed: card declined for 249900 paise
gave up after 3 attempts <- card declined for 249900 paise
```

How the types work:

- `task: () => Promise<T>` is a function that starts the work. It must be a function, not a promise: a promise has already started, and cannot be started again.
- `T` comes from the task: for `reserveStock` it is `string`, so `reservation` is a `string`. The helper works for any result type.
- `lastError` is `unknown`, like every error, and becomes the `cause` of the final `AppError`. `errorChain` then shows both messages.
- `2 ** (attempt - 1)` doubles the wait each time: 100, 200, 400 ms.

`return await task()` needs the `await`: without it, the promise is returned while still pending, its rejection happens outside the `try`, and nothing is retried (see [Mistake 6](#mistake-6-returning-a-promise-from-try-without-await)). And the options are checked like any object. Add at the end

```typescript
const count: number = await retry(() => reserveStock("KB-101", 1), { attempts: 2, delayMs: 10 });
await retry(() => chargeCard(100), { attempts: 2 });
```

```text
retry.ts:54:7 - error TS2322: Type 'string' is not assignable to type 'number'.

54 const count: number = await retry(() => reserveStock("KB-101", 1), { attempts: 2, delayMs: 10 });
         ~~~~~

retry.ts:55:36 - error TS2741: Property 'delayMs' is missing in type '{ attempts: number; }' but required in type 'RetryOptions'.

55 await retry(() => chargeCard(100), { attempts: 2 });
                                      ~~~~~~~~~~~~~~~

  retry.ts:6:3 - 'delayMs' is declared here.
    6   delayMs: number;
        ~~~~~~~


Found 2 errors in the same file, starting at: retry.ts:54
```

![A timeline of three attempts to reserve stock: the first two fail with inventory service busy, and the waits between them double from 100 to 200 ms; the third attempt succeeds; for the card, all three attempts fail and the final AppError keeps the last error as its cause](images/23-async-and-error-handling/retry.gif)

---

## Mini Example 2: An Order Page with a Time Limit

An order page needs the customer and their orders, and would like some product suggestions. The first two are required. The suggestions come from a slow service, and the page should not wait for it for more than 200 ms

order-page.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import { AppError, errorChain } from "./errors.ts";

interface Customer {
  id: number;
  name: string;
}

interface OrderSummary {
  id: number;
  totalPaise: number;
}

async function getCustomer(id: number): Promise<Customer> {
  await sleep(40);
  if (id !== 7) {
    throw new AppError(`customer ${id} not found`, 404);
  }
  return { id, name: "Asha" };
}

async function getOrders(customerId: number): Promise<OrderSummary[]> {
  await sleep(60);
  return [
    { id: 1001, totalPaise: 249900 },
    { id: 1002, totalPaise: 79900 }
  ];
}

async function getRecommendations(customerId: number): Promise<string[]> {
  await sleep(500);
  return ["Mouse pad", "USB hub"];
}

function withTimeout<T>(task: Promise<T>, ms: number, fallback: T): Promise<T> {
  return Promise.race([task, sleep(ms, fallback)]);
}

async function loadOrderPage(customerId: number) {
  const [customer, orders, recommendations] = await Promise.all([
    getCustomer(customerId),
    getOrders(customerId),
    withTimeout(getRecommendations(customerId), 200, [])
  ]);
  return { customer, orders, recommendations };
}

type OrderPage = Awaited<ReturnType<typeof loadOrderPage>>;

function renderPage(page: OrderPage): string {
  const totalPaise = page.orders.reduce((sum, order) => sum + order.totalPaise, 0);
  const extras = page.recommendations.length > 0 ? page.recommendations.join(", ") : "none (too slow)";
  return `${page.customer.name}: ${page.orders.length} orders, Rs ${totalPaise / 100}, suggestions: ${extras}`;
}

const started = Date.now();
console.log(renderPage(await loadOrderPage(7)));
console.log(Date.now() - started < 400 ? "page ready before the slow service" : "page waited for the slow service");

try {
  await loadOrderPage(8);
} catch (err) {
  console.log(errorChain(err));
}
```

```bash
npx tsc --noEmit
node order-page.ts
```

```text
Asha: 2 orders, Rs 3298, suggestions: none (too slow)
page ready before the slow service
customer 8 not found
```

What each part does:

- `withTimeout` races the task against `sleep(ms, fallback)`, which resolves with the fallback after `ms` milliseconds. Whichever settles first wins. Both promises hold a `T`, so the result is a `Promise<T>`.
- The `fallback` must have the task's type: `[]` for `string[]`. A text like `"none"` gives `error TS2345: Argument of type 'string' is not assignable to parameter of type 'string[]'.`
- `Promise.all` waits for all three at the same time, so the page is ready after about 200 ms, the time of its slowest part: the timeout.
- `loadOrderPage` has no return type annotation; `OrderPage` is built from it with `Awaited<ReturnType<...>>`, so `renderPage` always matches what the loader returns.
- A missing customer rejects `Promise.all` at once, and the error reaches the `catch`.

One limit: the timeout only stops the **waiting**. The slow request keeps running until it finishes; that is why the program ends about half a second after it starts. Cancelling the work itself needs an `AbortSignal`, which `fetch` accepts.

![Three requests start together as bars: customer and orders finish quickly, recommendations would take 500 ms but a 200 ms timer wins the race and supplies an empty list, so the page renders at 200 ms](images/23-async-and-error-handling/order-page.gif)

---

## Beginner Mistakes

### Mistake 1: Giving an async function a plain return type

`async function countOrders(): number` gives `error TS1064: The return type of an async function or method must be the global Promise<T> type. Did you mean to write 'Promise<number>'?` Write `Promise<number>`.

---

### Mistake 2: Forgetting await

`if (isInStock(1))` gives `error TS2801: This condition will always return true since this 'Promise<boolean>' is always defined.` Reading a property of a promise gives `error TS2339`. Add `await`.

---

### Mistake 3: Calling an async function without waiting for it

`chargeCard(249900);` on its own line passes `tsc`, and a rejection stops Node.js later. `await` it, `return` it, or add a `.catch()`. A lint rule finds these (Session 29).

---

### Mistake 4: Using forEach with an async callback

`forEach` does not wait for the promises its callback returns, so code after it runs too early. Use `for...of` with `await` for one after another, or `Promise.all(items.map(...))` for all at once.

---

### Mistake 5: Trusting the any in .catch() and allSettled

The parameter of a `.catch()` callback and the `reason` of a rejected `allSettled` result are `any`, so `err.mesage` passes `tsc`. Write `(err: unknown) =>`, and use a helper like `errorMessage`.

---

### Mistake 6: Returning a promise from try without await

`try { return task(); } catch { ... }` returns the promise while it is still pending. If it rejects later, the `catch` never sees it: in Mini Example 1, the first failure stopped the program at once instead of retrying. Write `return await task();` inside `try`.

---

### Mistake 7: Using res.json() as your type

`res.json()` is `Promise<any>`, so `return await res.json()` passes as a `Product` even when the data is wrong. Store it as `unknown` and check it first.

---

### Mistake 8: Reading cause without a check

`err.cause.message` gives `error TS18046: 'err.cause' is of type 'unknown'.` Check it with `instanceof Error`, or use a helper like `errorChain`.

---

### Mistake 9: Passing a promise where a task is expected

`retry(reserveStock("KB-101", 2), ...)` passes a promise that has already started, and it cannot be started again. `tsc` refuses it with `error TS2345: Argument of type 'Promise<string>' is not assignable to parameter of type '() => Promise<unknown>'.` Pass `() => reserveStock("KB-101", 2)`.

---

### Mistake 10: A resolve() without a value in Promise<number>

`new Promise<number>((resolve) => resolve())` gives `error TS2794`. Resolve with a number, or make it a `Promise<void>`.

---

### Mistake 11: Expecting a timeout to cancel the work

`Promise.race` with a timer stops your code from waiting, but the slow task still runs to the end. Use an `AbortSignal` when the work itself must stop.

---

## Practice Exercises

### Exercise 1

Write `async function getStock(sku: string): Promise<number>` with a `Map` of stock levels and a `sleep` of 30 ms. Then write `canOrder(sku, quantity)` that uses it. Remove one `await` on purpose and read the error.

### Exercise 2

Wrap `setTimeout` yourself: write `wait(ms: number): Promise<void>` with `new Promise`. Then write `waitFor<T>(ms: number, value: T): Promise<T>` and test it with a string and an object.

### Exercise 3

Turn the callback function `scrypt` from `node:crypto` into `hashPassword(password: string, salt: string): Promise<string>` (use a key length of 32 and return the key as hex). Check that the same password and salt give the same hash.

### Exercise 4

With `Promise.allSettled`, charge five cards with different amounts and print a summary: how many succeeded, how many failed, and the total amount charged.

### Exercise 5

Write `firstSuccessful<T>(tasks: (() => Promise<T>)[]): Promise<T>` that tries the tasks one after another and returns the first result that does not reject. If all fail, throw an `AppError` whose `cause` is the last error.

### Exercise 6

Add `getReviews(productId)` to `products.ts` with its own type guard `isReview`, and a fake fetcher that returns a list of reviews. Make the guard reject a list that contains one bad review.

### Exercise 7

In Mini Example 1, add an option `shouldRetry: (err: unknown) => boolean`, so that a declined card (a permanent error) is not retried, but a busy service is.

### Exercise 8

In Mini Example 2, give `getOrders` its own timeout of 300 ms with an empty list as the fallback, and add a field `partial: boolean` to the page that is true when any part used its fallback.

---

## Interview Questions

### How do you type the return value of an async function?

With `Promise<T>`, where `T` is the type of the value it returns: `async function findOrder(id: number): Promise<Order | undefined>`. A plain type like `number` gives TS1064.

### What does TypeScript do when you forget await?

When you use the value, it reports an error, often with "Did you forget to use 'await'?": TS2801 for a promise in a condition, TS2339 for a missing property. When you do not use the value at all, nothing is reported.

### What is a floating promise?

A promise that nobody awaits, returns or handles with `.catch()`. Its errors become unhandled rejections. TypeScript does not report it; the lint rule `no-floating-promises` does.

### Why is there no error type in Promise<T>?

A promise can reject with any value, and TypeScript cannot know which. So errors are always `unknown` in `catch`, and must be checked.

### What type does the err parameter of .catch() have?

`any`, because of how TypeScript's built-in types describe `.catch()`. Write `(err: unknown) =>` yourself to get the same safety as in a `catch` clause.

### What does Promise.all return in TypeScript?

A promise of a tuple, with each place keeping the type of its promise: `Promise.all([getCustomer(7), countOrders(7)])` gives `[Customer, number]`.

### How do you use the results of Promise.allSettled?

Each result is a union on `status`: check `status === "fulfilled"` before reading `value`, and use `reason` (which is `any`) otherwise.

### What does Awaited<T> do?

It unwraps a promise type, like `await` does for values, also through nested promises. `Awaited<ReturnType<typeof loadDashboard>>` is the type of the loaded data.

### What is the cause of an error?

The original error, kept inside a new, clearer one: `new AppError("could not read settings", 500, { cause: err })`. Its type is `unknown`, and Node.js prints it with the error.

### Why is res.json() dangerous in TypeScript?

It returns `Promise<any>`, so wrong data passes as any type you like. Store it as `unknown` and check it, with a type predicate or a validation library.

### Why should a retry helper take a function instead of a promise?

A promise starts its work as soon as it is created, and cannot be started again. A function `() => Promise<T>` can be called once per attempt.

### Why is return await needed inside try?

Without `await`, the function returns the pending promise, and a later rejection happens outside the `try`, so the `catch` never runs.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [22. Decorators](22-decorators.md) | [Course outline](README.md#course-outline) | [24. Modules in TypeScript](24-modules.md) |
