# Linting, Testing and Debugging

> **Session 29 of 33** | Part 6: Real Programs | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Four Tools, Four Questions](#four-tools-four-questions)
- [Your First Test with Vitest](#your-first-test-with-vitest)
- [A Test That Finds a Bug](#a-test-that-finds-a-bug)
- [Vitest Does Not Check Types](#vitest-does-not-check-types)
- [Testing Errors, and a Wrong Value on Purpose](#testing-errors-and-a-wrong-value-on-purpose)
- [Testing Async Code with a Fake Fetcher](#testing-async-code-with-a-fake-fetcher)
- [Fake Functions with vi.fn](#fake-functions-with-vifn)
- [ESLint and TypeScript 7](#eslint-and-typescript-7)
- [Your First Lint Run](#your-first-lint-run)
- [Fixing What ESLint Found](#fixing-what-eslint-found)
- [Reading a Stack Trace](#reading-a-stack-trace)
- [Built Code and Source Maps](#built-code-and-source-maps)
- [Pausing the Program in VS Code](#pausing-the-program-in-vs-code)
- [The Debugger in the Terminal: node inspect](#the-debugger-in-the-terminal-node-inspect)
- [Mini Example 1: The Bug from Session 01](#mini-example-1-the-bug-from-session-01)
- [Mini Example 2: From a Bug Report to a Test](#mini-example-2-from-a-bug-report-to-a-test)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

This session's folder is a small project with its own `package.json`, like Session 28. Vitest and ESLint are installed per project, and ESLint needs a special setup (you will see why), so it should not change the `ts-practice` project. Inside `ts-practice`

```bash
mkdir 29-quality
cd 29-quality
npm init -y
npm pkg set type=module
npm install --save-dev typescript@7.0.2 @types/node@24
npm install joi
```

Then copy three files you already have

| Copy                | From                           | It contains                                                  |
| ------------------- | ------------------------------ | ------------------------------------------------------------ |
| `tsconfig.json`     | `27-api` (Session 27)          | The settings for a Node.js API: `src/` in, `dist/` out, strict, source maps |
| `src/errors.ts`     | `23-async/errors.ts` (Session 23) | `AppError`, `errorMessage` and `errorChain`              |
| `src/validation.ts` | `26-validation/validation.ts` (Session 26) | `InvalidDataError` and `validate()`             |

Every command in this session runs inside `29-quality`. The versions we tested: vitest 5.0.5, eslint 10.14.0, typescript-eslint 8.72.1.

---

## Four Tools, Four Questions

Since Session 01, `tsc` has answered one question: do the types fit? Session 01 also showed a bug it can not see: `sendEmail("Welcome to the shop!", "riya@example.com")` had the arguments the wrong way round, and both are strings. Real projects use three more tools, and each one answers a different question

| Tool                | Question                                   | Finds, for example                                 |
| ------------------- | ------------------------------------------ | -------------------------------------------------- |
| `tsc`               | Do the types fit?                          | A string passed where a number belongs             |
| Tests (Vitest)      | Does the code give the right result?       | A price of 588.82 paise where 589 is right         |
| A linter (ESLint)   | Does the code look like a known mistake?   | A promise that nobody waits for (Session 23)       |
| A debugger          | What is really in this variable right now? | A quantity that is the text `"1"`, not the number 1 |

![Four tools side by side: tsc answers whether the types fit, Vitest whether the result is right, ESLint whether the code is a known mistake, and the debugger what is really in a variable, each with a mistake from this session that only it catches](images/29-linting-testing-debugging/four-tools.gif)

---

## Your First Test with Vitest

A **test** is a small program that calls your code with inputs you chose and compares the result with the answer you expect. A **test runner** finds the tests, runs them and reports what failed. This course uses **Vitest**: it runs TypeScript files directly and works with ES modules without any settings

```bash
npm install --save-dev vitest
```

```text
added 35 packages, and audited 48 packages in 7s

12 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

Something to test: a function that adds GST (goods and services tax) to a price in paise

src/prices.ts

```typescript
export function withGst(pricePaise: number, gstPercent: number): number {
  return pricePaise + (pricePaise * gstPercent) / 100;
}
```

The test goes into a file next to it, with `.test.ts` at the end of the name

src/prices.test.ts

```typescript
import { describe, expect, it } from "vitest";
import { withGst } from "./prices.ts";

describe("withGst", () => {
  it("adds 18% GST to a price", () => {
    expect(withGst(149900, 18)).toBe(176882);
  });
});
```

| Part                                   | Meaning                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| `import { ... } from "vitest"`         | The test functions, with their types. Nothing is global      |
| `describe("withGst", ...)`             | A group of tests with a name, usually the function's name    |
| `it("adds 18% GST to a price", ...)`   | One test. The name and `it` read like a sentence. `test` is the same function |
| `expect(value)`                        | Starts a check on a value                                    |
| `.toBe(176882)`                        | The value must be exactly this (for numbers, strings, booleans) |

149900 paise is Rs. 1499; 18% of it is 26982 paise, so the answer is 176882. Run it

```bash
npx vitest run
```

```text
 RUN  v5.0.5 D:/ts-practice/29-quality


 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  16:45:49
   Duration  376ms (transform 53%, import 29%, worker 10%, tests 7%)
```

Vitest found the file on its own: it runs every file whose name ends with `.test.ts` (or `.spec.ts`), in every folder except `node_modules`. The times are different on every run.

The summary is all you see when everything passes. To see every test by name, add `--reporter=verbose`

```bash
npx vitest run --reporter=verbose
```

```text
 RUN  v5.0.5 D:/ts-practice/29-quality

 ✓ src/prices.test.ts > withGst > adds 18% GST to a price 2ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  16:45:51
   Duration  187ms (import 50%, transform 25%, worker 14%, tests 10%, environment 1%)
```

Put both checks into `package.json`. Replace the `"scripts"` part with

```json
  "scripts": {
    "check": "tsc --noEmit",
    "test": "vitest run --reporter=verbose"
  },
```

| Command          | What it does                                                         |
| ---------------- | -------------------------------------------------------------------- |
| `npm run check`  | `tsc` checks the types (Session 02)                                  |
| `npm test`       | Vitest runs all tests once. `npm test` is short for `npm run test`   |
| `npx vitest`     | Without `run`: **watch mode**. Vitest stays open and runs the tests again whenever you save a file. Stop it with Ctrl+C |

---

## A Test That Finds a Bug

One test with one easy number proves little. Add a second test inside `describe`, with a cheap product of Rs. 4.99

```typescript
  it("returns whole paise", () => {
    expect(withGst(499, 18)).toBe(589);
  });
```

```bash
npm test
```

```text
> 29-quality@1.0.0 test
> vitest run --reporter=verbose


 RUN  v5.0.5 D:/ts-practice/29-quality

 ✓ src/prices.test.ts > withGst > adds 18% GST to a price 1ms
 × src/prices.test.ts > withGst > returns whole paise 5ms
   → expected 588.8199999999999 to be 589 // Object.is equality

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/prices.test.ts > withGst > returns whole paise
AssertionError: expected 588.8199999999999 to be 589 // Object.is equality

- Expected
+ Received

- 589
+ 588.8199999999999

 ❯ src/prices.test.ts:10:30
      8|
      9|   it("returns whole paise", () => {
     10|     expect(withGst(499, 18)).toBe(589);
       |                              ^
     11|   });
     12| });

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


 Test Files  1 failed (1)
      Tests  1 failed | 1 passed (2)
   Start at  16:47:47
   Duration  191ms (import 38%, transform 27%, tests 22%, worker 12%, environment 1%)
```

How to read it

| Part of the output                         | Meaning                                                    |
| ------------------------------------------ | ---------------------------------------------------------- |
| The check mark and the cross               | Passed and failed tests, with the `describe` and `it` names |
| `- Expected` / `+ Received`                | What the test wanted (589) and what the code returned      |
| `src/prices.test.ts:10:30` and the code    | The exact line of the failed check                         |
| `Tests  1 failed \| 1 passed (2)`          | The totals. `npm test` ends with exit code 1, so scripts and CI servers notice |

18% of 499 is 89.82, so the function returned 588.82 paise, and a computer's decimal arithmetic adds the long tail `588.8199999999999` (JavaScript course, Session 09). Paise must be whole numbers. Round in `withGst`

src/prices.ts

```typescript
export function withGst(pricePaise: number, gstPercent: number): number {
  return Math.round(pricePaise + (pricePaise * gstPercent) / 100);
}
```

```bash
npm test
```

```text
 ✓ src/prices.test.ts > withGst > adds 18% GST to a price 1ms
 ✓ src/prices.test.ts > withGst > returns whole paise 0ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

Look at what found the bug. `tsc` was happy the whole time: 588.82 is a `number`, exactly what the return type promises. Only a check with a known answer can tell that a value of the right type is wrong.

![The test file with two tests next to npm test: the first run shows one green and one red test, expected 588.82 to be 589; after Math.round is added to withGst, both tests are green](images/29-linting-testing-debugging/test-run.gif)

---

## Vitest Does Not Check Types

Add this test to the `describe`, the way a value from a form would arrive

```typescript
  it("works with a GST percent from a form", () => {
    expect(withGst(149900, "18")).toBe(176882);
  });
```

```bash
npm test
```

```text
 ✓ src/prices.test.ts > withGst > adds 18% GST to a price 2ms
 ✓ src/prices.test.ts > withGst > returns whole paise 0ms
 ✓ src/prices.test.ts > withGst > works with a GST percent from a form 0ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

All green. Vitest does what `node file.ts` does (Session 02): it removes the types and runs the JavaScript. `149900 * "18"` happens to work in JavaScript, because `*` turns text into a number. The type check is `tsc`'s job

```bash
npm run check
```

```text
> 29-quality@1.0.0 check
> tsc --noEmit

src/prices.test.ts:14:28 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

14     expect(withGst(149900, "18")).toBe(176882);
                              ~~~~


Found 1 error in src/prices.test.ts:14
```

The test files are inside `src`, so `tsc` checks them like every other file. A project is only green when **both** are green: `npm run check` and `npm test`. Remove the test with `"18"`.

Vitest also has a way to test a **type**: `expectTypeOf`. Add it to the import, and add a test

```typescript
import { describe, expect, expectTypeOf, it } from "vitest";
```

```typescript
  it("returns a number", () => {
    expectTypeOf(withGst).returns.toEqualTypeOf<number>();
  });
```

It reads: "the return type of `withGst` equals `number`". This line is checked by `tsc`, not by Vitest: at run time it does nothing. To see it fail, write `toEqualTypeOf<string>()` for a moment. `npm test` still shows `returns a number` as passed, and `npm run check` says

```text
src/prices.test.ts:14:49 - error TS2344: Type 'string' does not satisfy the constraint '"Expected: string, Actual: number"'.

14     expectTypeOf(withGst).returns.toEqualTypeOf<string>();
                                                   ~~~~~~
```

The message is unusual, but the important part is in the quotes: `Expected: string, Actual: number`. Put `number` back. Type tests are most useful for the generic and mapped types of Sessions 14 to 19, where a small change can quietly change a result type.

![One test file and two checks: npm test removes the types and runs the code, and the test with the text "18" passes; npm run check reads the types and runs nothing, and reports TS2345 for the same line](images/29-linting-testing-debugging/types-vs-values.gif)

---

## Testing Errors, and a Wrong Value on Purpose

Session 10 ended a `switch` with a `default` that assigns the value to `never`. Here it is in a shipping function

src/shipping.ts

```typescript
export type Zone = "local" | "national";

export function shippingPaise(zone: Zone, cartPaise: number): number {
  if (cartPaise >= 50000) {
    return 0;
  }
  switch (zone) {
    case "local":
      return 4000;
    case "national":
      return 9000;
    default: {
      const unknownZone: never = zone;
      throw new Error(`unknown zone: ${unknownZone}`);
    }
  }
}
```

The `default` can still run: data from a JavaScript caller, or data that skipped validation, may hold any text. A test for it

src/shipping.test.ts

```typescript
import { describe, expect, it } from "vitest";
import { shippingPaise } from "./shipping.ts";

describe("shippingPaise", () => {
  it("charges by zone", () => {
    expect(shippingPaise("local", 19900)).toBe(4000);
    expect(shippingPaise("national", 19900)).toBe(9000);
  });

  it("is free from Rs. 500", () => {
    expect(shippingPaise("national", 50000)).toBe(0);
  });

  it("refuses a zone that is not in the type", () => {
    expect(() => shippingPaise("moon", 19900)).toThrow("unknown zone: moon");
  });
});
```

`toThrow` needs a **function** to call: it calls it, catches the error, and compares the message. That is why the call is wrapped in `() => ...` (Beginner Mistakes shows what happens without it). Now check the types

```text
src/shipping.test.ts:15:32 - error TS2345: Argument of type '"moon"' is not assignable to parameter of type 'Zone'.

15     expect(() => shippingPaise("moon", 19900)).toThrow("unknown zone: moon");
                                  ~~~~~~


Found 1 error in src/shipping.test.ts:15
```

`tsc` is right, and that is exactly the point of this test: the wrong value is passed **on purpose**. Session 05 promised that this is the place for `@ts-expect-error`. Put it on the line above, with a reason

```typescript
  it("refuses a zone that is not in the type", () => {
    // @ts-expect-error a zone from outside data, which skipped validation
    expect(() => shippingPaise("moon", 19900)).toThrow("unknown zone: moon");
  });
```

`npm run check` is quiet now. To run only this file, put its name after `--`: npm passes everything after `--` on to Vitest, and Vitest runs only the test files whose path contains that text

```bash
npm test -- src/shipping.test.ts
```

```text
 ✓ src/shipping.test.ts > shippingPaise > charges by zone 2ms
 ✓ src/shipping.test.ts > shippingPaise > is free from Rs. 500 0ms
 ✓ src/shipping.test.ts > shippingPaise > refuses a zone that is not in the type 1ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

If someone later adds `"moon"` to `Zone`, the expected error disappears and `tsc` reports TS2578 "Unused '@ts-expect-error' directive" (Session 28), so the test is updated instead of silently testing nothing.

---

## Testing Async Code with a Fake Fetcher

Session 23 gave `getProduct` a `fetcher` parameter, so tests can pass a fake instead of the real `fetch`. Session 26 added Joi. Put that code into its own file, with `export`

src/products.ts

```typescript
import Joi from "joi";
import { AppError } from "./errors.ts";
import { validate } from "./validation.ts";

export type Fetcher = (url: string) => Promise<Response>;

export interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

const productSchema = Joi.object<Product, true>({
  id: Joi.number().integer().required(),
  name: Joi.string().required(),
  pricePaise: Joi.number().integer().min(0).required()
});

export async function getProduct(id: number, fetcher: Fetcher): Promise<Product> {
  const res = await fetcher(`http://localhost:4000/products/${id}`);
  if (!res.ok) {
    throw new AppError(`product ${id}: the API answered ${res.status}`, res.status);
  }
  const data: unknown = await res.json();
  try {
    return validate(productSchema, data, `product ${id}`);
  } catch (err) {
    throw new AppError(`product ${id}: unexpected data from the API`, 502, { cause: err });
  }
}
```

The test uses Session 26's fake fetcher: one good answer, one in the wrong shape, one 404

src/products.test.ts

```typescript
import { describe, expect, it } from "vitest";
import { getProduct, type Fetcher } from "./products.ts";

const fakeFetch: Fetcher = async (url) => {
  if (url.endsWith("/1")) {
    return new Response(JSON.stringify({ id: 1, name: "Keyboard", pricePaise: 149900, stock: 12 }));
  }
  if (url.endsWith("/2")) {
    return new Response(JSON.stringify({ id: 2, name: "Mouse", price: 499 }));
  }
  return new Response(JSON.stringify({ error: "not found" }), { status: 404 });
};

describe("getProduct", () => {
  it("returns the product without the extra fields", async () => {
    const product = await getProduct(1, fakeFetch);
    expect(product).toEqual({ id: 1, name: "Keyboard", pricePaise: 149900 });
  });

  it("turns data in the wrong shape into a 502", async () => {
    await expect(getProduct(2, fakeFetch)).rejects.toThrow("unexpected data from the API");
    await expect(getProduct(2, fakeFetch)).rejects.toMatchObject({ statusCode: 502 });
    await expect(getProduct(2, fakeFetch)).rejects.toHaveProperty("cause.problems", ["pricePaise is required"]);
  });

  it("passes the 404 of the API on", async () => {
    await expect(getProduct(3, fakeFetch)).rejects.toMatchObject({ statusCode: 404 });
  });
});
```

```bash
npm test -- src/products.test.ts
```

```text
 ✓ src/products.test.ts > getProduct > returns the product without the extra fields 23ms
 ✓ src/products.test.ts > getProduct > turns data in the wrong shape into a 502 5ms
 ✓ src/products.test.ts > getProduct > passes the 404 of the API on 0ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

| Part                                          | Meaning                                                      |
| --------------------------------------------- | ------------------------------------------------------------ |
| `async () => { ... }`                         | An async test: Vitest waits for it to finish                 |
| `toEqual({ ... })`                            | Compares the **contents** of objects and arrays. `toBe` would compare two different objects and fail |
| `await expect(promise).rejects`               | The promise must reject; the checks after `.rejects` look at the error |
| `.toThrow("unexpected data from the API")`    | The error message contains this text                         |
| `.toMatchObject({ statusCode: 502 })`         | The error has at least these properties                      |
| `.toHaveProperty("cause.problems", [...])`    | Follows a path: the `cause` (Session 23) is the `InvalidDataError` with its `problems` |

The fake fetcher is why these tests are good tests: no server has to run, nothing goes over the network, every run takes milliseconds and gives the same answers, and the error cases are as easy to test as the good one. With a real API, a 404 or a broken answer is hard to get on demand.

Every `expect(...).rejects` and `expect(...).resolves` needs an `await`. Without it, Vitest 5 fails the test

```text
 × src/products.test.ts > getProduct > passes the 404 of the API on 10ms
   → Promise returned by `expect(actual).rejects.toMatchObject(expected)` was not awaited. This assertion is asynchronous and must be awaited; otherwise, it is not guaranteed to complete before the test finishes:
```

![The fake fetcher answers three addresses: a keyboard with an extra stock field, a mouse with price instead of pricePaise, and a 404; each answer leads to one green test, with toEqual for the clean product and rejects for the 502 and the 404](images/29-linting-testing-debugging/fake-fetcher.gif)

---

## Fake Functions with vi.fn

Sometimes the question is not "what did my function return?" but "what did it **call**, and with what?". `vi.fn()` makes a fake function, a **mock**, that remembers every call. Add `vi` to the import

```typescript
import { describe, expect, it, vi } from "vitest";
```

and this test at the end of the `describe`

```typescript
  it("asks the right address", async () => {
    const fetcher = vi.fn<Fetcher>();
    fetcher.mockResolvedValue(new Response(JSON.stringify({ id: 7, name: "Cable", pricePaise: 19900 })));
    await getProduct(7, fetcher);
    expect(fetcher).toHaveBeenCalledWith("http://localhost:4000/products/7");
    expect(fetcher).toHaveBeenCalledTimes(1);
  });
```

| Part                                   | Meaning                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| `vi.fn<Fetcher>()`                     | A mock with the type `Fetcher` (a generic, Session 14), so it fits the `fetcher` parameter |
| `.mockResolvedValue(response)`         | Every call returns a promise that resolves with this value  |
| `toHaveBeenCalledWith(...)`            | Some call had exactly these arguments                        |
| `toHaveBeenCalledTimes(1)`             | It was called exactly once                                   |

```bash
npm test -- src/products.test.ts
```

```text
 ✓ src/products.test.ts > getProduct > returns the product without the extra fields 25ms
 ✓ src/products.test.ts > getProduct > turns data in the wrong shape into a 502 6ms
 ✓ src/products.test.ts > getProduct > passes the 404 of the API on 0ms
 ✓ src/products.test.ts > getProduct > asks the right address 1ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
```

The type argument keeps the mock honest. A fake answer that a real fetcher could never give is an error

```typescript
fetcher.mockResolvedValue({ id: 7 });
```

```text
src/products.test.ts:40:33 - error TS2353: Object literal may only specify known properties, and 'id' does not exist in type 'Response'.
```

One part is **not** checked by types: the arguments of `toHaveBeenCalledWith`. `expect(fetcher).toHaveBeenCalledWith(7)` passes `tsc`, and only the test run finds it

```text
 × src/products.test.ts > getProduct > asks the right address 6ms
   → expected "vi.fn()" to be called with arguments: [ 7 ]
```

![A test with vi.fn of Fetcher: the mock records the call with the product address and passes the toHaveBeenCalledWith check; a fake answer without a Response gives TS2353, while toHaveBeenCalledWith(7) passes tsc and fails only when the test runs](images/29-linting-testing-debugging/typed-mock.gif)

---

## ESLint and TypeScript 7

Some mistakes pass both `tsc` and the tests, because nobody wrote a test for them. Session 23 showed two. Write them into a file, together with two smaller problems

src/payments.ts

```typescript
import { setTimeout as sleep } from "node:timers/promises";

export interface Payment {
  orderId: number;
  amountPaise: number;
}

export async function chargeCard(payment: Payment): Promise<string> {
  await sleep(50);
  if (payment.amountPaise > 100000) {
    throw new Error(`card declined for order ${payment.orderId}`);
  }
  return `charged ${payment.amountPaise} paise for order ${payment.orderId}`;
}

export async function sendReceipt(orderId: number): Promise<void> {
  await sleep(50);
  console.log(`receipt sent for ${orderId}`);
}
```

src/orders.ts

```typescript
import { errorMessage } from "./errors.ts";
import { chargeCard, sendReceipt } from "./payments.ts";

export async function placeOrder(orderId: number, amountPaise: number): Promise<void> {
  chargeCard({ orderId, amountPaise });
  let message = `order ${orderId} saved`;
  console.log(message);
}

export function sendReceipts(orderIds: number[]): void {
  orderIds.forEach(async (id) => {
    await sendReceipt(id);
  });
}
```

`npm run check` is quiet. A **linter** reads your code and compares it with **rules**: patterns that are almost always mistakes, like a promise nobody waits for. **ESLint** is the standard linter for JavaScript, and **typescript-eslint** adds rules for TypeScript. Some of its rules read the **types** too: to know that `chargeCard(...)` returns a promise, a rule has to ask TypeScript. That is called **type-aware linting**. Install it

```bash
npm install --save-dev eslint @eslint/js typescript-eslint
```

```text
npm error code ERESOLVE
npm error ERESOLVE unable to resolve dependency tree
npm error
npm error While resolving: 29-quality@1.0.0
npm error Found: typescript@7.0.2
npm error node_modules/typescript
npm error   dev typescript@"^7.0.2" from the root project
npm error
npm error Could not resolve dependency:
npm error peer typescript@">=4.8.4 <6.1.0" from typescript-eslint@8.72.1
npm error node_modules/typescript-eslint
npm error   dev typescript-eslint@"*" from the root project
```

Read it the way Session 05 taught you to read errors: typescript-eslint needs a **peer** package `typescript` between 4.8.4 and 6.1.0, and this project has 7.0.2. The reason: TypeScript 7.0 is a new, native program (Session 01), and the TypeScript team's announcement says it does not ship with a JavaScript **API** yet, the functions other tools call to ask TypeScript about types. A new API is planned for 7.1. Until then, tools like typescript-eslint use TypeScript 6.

The announcement also gives the fix: install both versions, each under its own name

```bash
npm install --save-dev @typescript/native@npm:typescript@7.0.2 typescript@npm:@typescript/typescript6
npm install --save-dev eslint @eslint/js typescript-eslint
```

```text
added 91 packages, and audited 141 packages in 6s

40 packages are looking for funding
  run `npm fund` for details

11 moderate severity vulnerabilities

To address all issues, run:
  npm audit fix

Run `npm audit` for details.
```

`name@npm:package` installs a package under another name (an **alias**). The `devDependencies` in `package.json` now read

```json
  "devDependencies": {
    "@eslint/js": "^10.0.1",
    "@types/node": "^24.19.2",
    "@typescript/native": "npm:typescript@^7.0.2",
    "eslint": "^10.14.0",
    "typescript": "npm:@typescript/typescript6@^6.0.2",
    "typescript-eslint": "^8.72.1",
    "vitest": "^5.0.5"
  },
```

| Name in `node_modules` | Really is                  | Used by                                                    |
| ---------------------- | -------------------------- | ---------------------------------------------------------- |
| `@typescript/native`   | TypeScript 7.0.2           | The `tsc` command: `npm run check`, builds                 |
| `typescript`           | TypeScript 6, for its API  | typescript-eslint, which imports `"typescript"`. It also has a `tsc6` command |

```bash
npx tsc -v
npx tsc6 -v
```

```text
Version 7.0.2
Version 6.0.3
```

Your checks and builds still use TypeScript 7. Only the lint rules see TypeScript 6, and the types in this project mean the same in both. When typescript-eslint supports the new API, the alias goes away.

About the 11 "moderate severity vulnerabilities": `npm audit` explains that they are in a package used inside ESLint, and that the fix would be `Will install eslint@4.0.0, which is a breaking change`, an ESLint from 2017. Do **not** run `npm audit fix --force` here. The Node.js course met the same warning with Jest: these tools only run on your computer, never on the server.

![npm install of typescript-eslint fails with ERESOLVE next to TypeScript 7; with two aliases in package.json, npx tsc runs TypeScript 7.0.2 and typescript-eslint imports TypeScript 6.0.3 for its types](images/29-linting-testing-debugging/side-by-side.gif)

---

## Your First Lint Run

ESLint reads its settings from `eslint.config.js` in the project folder. It is a JavaScript file, so `// @ts-check` (Session 28) lets your editor check it

eslint.config.js

```javascript
// @ts-check
import js from "@eslint/js";
import { defineConfig } from "eslint/config";
import tseslint from "typescript-eslint";

export default defineConfig(
  js.configs.recommended,
  tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true }
    }
  }
);
```

| Part                                    | Meaning                                                      |
| --------------------------------------- | ------------------------------------------------------------ |
| `js.configs.recommended`                | ESLint's recommended rules for JavaScript                    |
| `tseslint.configs.recommendedTypeChecked` | typescript-eslint's recommended rules, including the ones that read types |
| `projectService: true`                  | Ask TypeScript for the types, using the nearest `tsconfig.json` |
| `defineConfig(...)`                     | Combines the parts; later parts can change earlier ones      |

Add a script. The `"scripts"` part becomes

```json
  "scripts": {
    "check": "tsc --noEmit",
    "lint": "eslint src",
    "test": "vitest run --reporter=verbose"
  },
```

```bash
npm run lint
```

```text
> 29-quality@1.0.0 lint
> eslint src


D:\ts-practice\29-quality\src\orders.ts
   1:10  error  'errorMessage' is defined but never used                                                                                                                            @typescript-eslint/no-unused-vars
   4:8   error  Async function 'placeOrder' has no 'await' expression                                                                                                               @typescript-eslint/require-await
   5:3   error  Promises must be awaited, end with a call to .catch, end with a call to .then with a rejection handler or be explicitly marked as ignored with the `void` operator  @typescript-eslint/no-floating-promises
   6:7   error  'message' is never reassigned. Use 'const' instead                                                                                                                  prefer-const
  11:20  error  Promise returned in function argument where a void return was expected                                                                                              @typescript-eslint/no-misused-promises

D:\ts-practice\29-quality\src\products.test.ts
  4:40  error  Async arrow function 'fakeFetch' has no 'await' expression  @typescript-eslint/require-await

D:\ts-practice\29-quality\src\shipping.ts
  14:40  error  Invalid type "never" of template literal expression  @typescript-eslint/restrict-template-expressions

✖ 7 problems (7 errors, 0 warnings)
  1 error and 0 warnings potentially fixable with the `--fix` option.
```

Every problem has a line and column, a message and, at the end, the **rule** that found it. Search for the rule's name to read its documentation.

| Line            | Rule                          | What it found                                                |
| --------------- | ----------------------------- | ------------------------------------------------------------ |
| orders.ts 1     | `no-unused-vars`              | An import nobody uses. Session 27 left this check to a linter |
| orders.ts 4     | `require-await`               | An `async` function that waits for nothing: often a sign of a missing `await` |
| orders.ts 5     | `no-floating-promises`        | Session 23's floating promise: the order is "saved" even if the card is declined |
| orders.ts 6     | `prefer-const`                | `let` for a variable that never changes                      |
| orders.ts 11    | `no-misused-promises`         | Session 23's `forEach(async ...)`: `forEach` throws the promises away |
| products.test.ts 4 | `require-await`            | The fake fetcher is `async` but waits for nothing            |
| shipping.ts 14  | `restrict-template-expressions` | A `never` value inside a template literal                  |

Several of these rules only work with types: nothing in the text `chargeCard(...)` says that it returns a promise, and only TypeScript knows that `unknownZone` is `never`.

![The file orders.ts with five highlighted lines, each connected to the rule that flags it: no-unused-vars, require-await, no-floating-promises, prefer-const and no-misused-promises, while tsc reports no errors](images/29-linting-testing-debugging/lint-run.gif)

---

## Fixing What ESLint Found

Some rules can fix their problems themselves. `--fix` applies those fixes and reports the rest

```bash
npx eslint src --fix
```

```text
✖ 6 problems (6 errors, 0 warnings)
```

`let message` is now `const message`. The rest needs you. In `orders.ts`, remove the unused import, `await` the payment, and use a `for...of` loop like Session 23

src/orders.ts

```typescript
import { chargeCard, sendReceipt } from "./payments.ts";

export async function placeOrder(orderId: number, amountPaise: number): Promise<void> {
  await chargeCard({ orderId, amountPaise });
  const message = `order ${orderId} saved`;
  console.log(message);
}

export async function sendReceipts(orderIds: number[]): Promise<void> {
  for (const id of orderIds) {
    await sendReceipt(id);
  }
}
```

In `shipping.ts`, the rule allows only types that turn into readable text, like `string` and `number`, inside a template, and `never` is not one of them. This branch exists for outside data, so turn the value into text on purpose with `String(...)`

```typescript
      throw new Error(`unknown zone: ${String(unknownZone)}`);
```

The fake fetcher is a different case: it is `async` only to return a promise, and it has nothing to wait for. Here the rule is wrong, so turn it off for **one line**, with a reason after `--`, like `@ts-expect-error` in Session 28

src/products.test.ts

```typescript
// eslint-disable-next-line @typescript-eslint/require-await -- a fake answers at once, it has nothing to wait for
const fakeFetch: Fetcher = async (url) => {
```

```bash
npm run lint
```

```text
> 29-quality@1.0.0 lint
> eslint src
```

No problems, and `npm test` still passes all 10 tests. A disable comment can also become stale: if the line changes so the rule no longer fires, ESLint reports the comment itself

```text
  4:1  warning  Unused eslint-disable directive (no problems were reported from '@typescript-eslint/require-await')
```

In VS Code, the **ESLint** extension shows these problems as underlines while you type, the same way `tsc` errors appear.

---

## Reading a Stack Trace

A program to debug: it charges a keyboard of Rs. 1499 with GST

src/app.ts

```typescript
import { chargeCard } from "./payments.ts";
import { withGst } from "./prices.ts";

const amountPaise = withGst(149900, 18);
const receipt = await chargeCard({ orderId: 1001, amountPaise });
console.log(receipt);
```

```bash
node src/app.ts
```

```text
file:///D:/ts-practice/29-quality/src/payments.ts:11
    throw new Error(`card declined for order ${payment.orderId}`);
          ^

Error: card declined for order 1001
    at chargeCard (file:///D:/ts-practice/29-quality/src/payments.ts:11:11)
    at async file:///D:/ts-practice/29-quality/src/app.ts:5:17

Node.js v24.19.0
```

| Part                                   | Meaning                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| `payments.ts:11` and the code with `^` | Where the error was thrown: line 11, column 11              |
| `Error: card declined for order 1001`  | The error's class and message                                |
| `at chargeCard (...payments.ts:11:11)` | The **stack**: the newest call first                         |
| `at async ...app.ts:5:17`              | Who called it: line 5 of `app.ts`. `async` means "after an `await`" |

Read it top to bottom: what went wrong, where, and who called that code. The line numbers are the lines of your `.ts` files, and that is not obvious: Node.js removes the types before it runs the file. The section about `node inspect` shows how the lines stay correct.

---

## Built Code and Source Maps

In production, the built JavaScript in `dist` runs (Session 02). Tests do not belong there: Vitest would even run them a second time from `dist` (Beginner Mistakes). Make a second config for builds, which extends the first and leaves the tests out (Session 27)

tsconfig.build.json

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

`npm run check` still checks every file, tests included; only the build uses the new file. The scripts, complete

```json
  "scripts": {
    "check": "tsc --noEmit",
    "lint": "eslint src",
    "test": "vitest run --reporter=verbose",
    "build": "tsc -p tsconfig.build.json",
    "start": "node --enable-source-maps dist/app.js"
  },
```

```bash
npm run build
node dist/app.js
```

```text
file:///D:/ts-practice/29-quality/dist/payments.js:5
        throw new Error(`card declined for order ${payment.orderId}`);
              ^

Error: card declined for order 1001
    at chargeCard (file:///D:/ts-practice/29-quality/dist/payments.js:5:15)
    at async file:///D:/ts-practice/29-quality/dist/app.js:4:17

Node.js v24.19.0
```

Line 5, column 15 of `payments.js`. Your `payments.ts` has the `throw` on line 11. The built file shows why

dist/payments.js

```javascript
import { setTimeout as sleep } from "node:timers/promises";
export async function chargeCard(payment) {
    await sleep(50);
    if (payment.amountPaise > 100000) {
        throw new Error(`card declined for order ${payment.orderId}`);
    }
```

`tsc` removed the `Payment` interface and the empty lines, and indents with 4 spaces. In a real project, a stack trace from the server would point at lines nobody wrote. The fix is the `.js.map` files from Session 02: `"sourceMap": true` writes one next to every `.js` file, and it says which line of `payments.js` came from which line of `payments.ts`. Node.js reads them only when asked, with `--enable-source-maps`, which is why the `start` script has it

```bash
npm start
```

```text
> 29-quality@1.0.0 start
> node --enable-source-maps dist/app.js

D:\ts-practice\29-quality\src\payments.ts:11
    throw new Error(`card declined for order ${payment.orderId}`);
          ^

Error: card declined for order 1001
    at chargeCard (D:\ts-practice\29-quality\src\payments.ts:11:11)
    at async <anonymous> (D:\ts-practice\29-quality\src\app.ts:5:17)

Node.js v24.19.0
```

The built code ran, and the trace shows your lines and your `.ts` code.

![payments.ts with the throw on line 11 next to the built payments.js with it on line 5; node dist/app.js reports payments.js:5:15, and with --enable-source-maps the .js.map file leads back to payments.ts:11:11](images/29-linting-testing-debugging/source-map.gif)

---

## Pausing the Program in VS Code

The stack trace says **where** the card was declined. The **why** is in the values. Instead of adding `console.log` lines and running again, pause the program and look

1. Open `src/payments.ts` and click in the margin left of line 10 (`if (payment.amountPaise > 100000)`). A red dot appears: a **breakpoint**.
2. Open a **JavaScript Debug Terminal**: press Ctrl+Shift+P, type `JavaScript Debug Terminal` and press Enter. It is a normal terminal, except that every Node.js program started in it is connected to the debugger.
3. Run `node src/app.ts` in it. The program stops **before** line 10 runs, and the line is highlighted.
4. The **VARIABLES** panel on the left shows `payment` with `orderId: 1001` and `amountPaise: 176882`. Pointing at a variable in the editor shows its value too.
5. The **DEBUG CONSOLE** at the bottom runs any expression with the current values: `payment.amountPaise > 100000` gives `true`.
6. The buttons at the top: **Continue** (F5) runs to the next breakpoint, **Step Over** (F10) runs one line, **Step Into** (F11) goes into the function called on this line, **Stop** (Shift+F5) ends the program.

The answer: with GST, the keyboard costs 176882 paise, Rs. 1768.82, and this card's limit is 100000 paise, Rs. 1000. No bug in the code this time, just a rule you can now see working.

Breakpoints work in your `.ts` files without any setup, because `node src/app.ts` keeps every line where it was (next section). A `debugger;` statement in the code works like a breakpoint when a debugger is connected, and does nothing otherwise. ESLint's `no-debugger` rule reports it, so it is not committed by accident

```text
D:\ts-practice\29-quality\src\cart.ts
  14:5  error  Unexpected 'debugger' statement  no-debugger
```

![A paused program in VS Code: a red breakpoint dot on line 10 of payments.ts, the line highlighted, the variables panel showing payment with amountPaise 176882, and the debug console answering true for payment.amountPaise > 100000](images/29-linting-testing-debugging/debugger-pause.gif)

---

## The Debugger in the Terminal: node inspect

Node.js has a debugger without an editor too. It is plain, but it works everywhere, for example on a server over SSH

```bash
node inspect src/app.ts
```

It stops before the first line runs, in `payments.ts`, because imported files run first. Type `sb('payments.ts', 10)` to set a breakpoint on line 10 of `payments.ts`, then `c` to continue to it

```text
debug> c
break in src\payments.ts:10
  8 export async function chargeCard(payment         )                  {
  9   await sleep(50);
>10   if (payment.amountPaise > 100000) {
 11     throw new Error(`card declined for order ${payment.orderId}`);
 12   }
debug> exec payment
{ orderId: 1001, amountPaise: 176882 }
debug> exec payment.amountPaise > 100000
true
debug> n
step in src\payments.ts:11
  9   await sleep(50);
*10   if (payment.amountPaise > 100000) {
>11     throw new Error(`card declined for order ${payment.orderId}`);
 12   }
 13   return `charged ${payment.amountPaise} paise for order ${payment.orderId}`;
```

| Command                 | Does                                              |
| ----------------------- | ------------------------------------------------- |
| `sb('file.ts', 10)`     | Set a breakpoint on line 10 of that file          |
| `c`                     | Continue to the next breakpoint                   |
| `n`                     | Next: run one line                                |
| `s`                     | Step into the function called on this line        |
| `exec expression`       | Show the value of an expression                   |
| `.exit`                 | Leave the debugger                                |

Now look at line 8: `chargeCard(payment         )                  {`. This is the code Node.js really runs. It does not delete the types, it **replaces them with spaces**: `: Payment` became 9 spaces and `: Promise<string>` 17. The `interface` lines 3 to 6 are blank lines. Every line and every column stays where it was, so stack traces, breakpoints and the debugger all match your `.ts` file without a source map. The built JavaScript is different, because `tsc` writes new code, and needs the `.js.map` files.

![payments.ts next to what Node.js runs: the interface lines are blank and the type annotations on line 8 became spaces, so node src/app.ts reports payments.ts:11:11 and node inspect stops at payments.ts line 10 without a source map](images/29-linting-testing-debugging/what-node-runs.gif)

---

## Mini Example 1: The Bug from Session 01

Session 01 showed `sendEmail("Welcome to the shop!", "riya@example.com")`: two strings in the wrong order, and `tsc` could not see it. Here is the same bug in code that can be tested. The mailer is a parameter, like the fetcher

src/emails.ts

```typescript
export type Mailer = (to: string, subject: string) => Promise<void>;

export async function welcomeCustomer(name: string, address: string, mailer: Mailer): Promise<void> {
  await mailer(`Welcome to the shop, ${name}!`, address);
}
```

src/emails.test.ts

```typescript
import { expect, it, vi } from "vitest";
import { welcomeCustomer, type Mailer } from "./emails.ts";

it("sends the welcome email to the customer", async () => {
  const mailer = vi.fn<Mailer>();
  await welcomeCustomer("Riya", "riya@example.com", mailer);
  expect(mailer).toHaveBeenCalledWith("riya@example.com", "Welcome to the shop, Riya!");
});
```

`npm run check` is quiet. The test is not

```bash
npm test -- src/emails.test.ts
```

```text
 × src/emails.test.ts > sends the welcome email to the customer 12ms
   → expected "vi.fn()" to be called with arguments: [ 'riya@example.com', …(1) ]

Received:

  1st vi.fn() call:

  [
-   "riya@example.com",
    "Welcome to the shop, Riya!",
+   "riya@example.com",
  ]


Number of calls: 1
```

The diff shows the two arguments in swapped places. Swapping them back fixes this call, but the next caller can make the same mistake. A better fix makes the mistake hard to write: an object with **named** properties (Session 07)

src/emails.ts

```typescript
export interface Email {
  to: string;
  subject: string;
}

export type Mailer = (email: Email) => Promise<void>;

export async function welcomeCustomer(name: string, address: string, mailer: Mailer): Promise<void> {
  await mailer({ to: address, subject: `Welcome to the shop, ${name}!` });
}
```

and the check in the test

```typescript
  expect(mailer).toHaveBeenCalledWith({ to: "riya@example.com", subject: "Welcome to the shop, Riya!" });
```

```text
 ✓ src/emails.test.ts > sends the welcome email to the customer 2ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

`to: address` is easy to read in a review; the second string in a list of two is not. The types and the test now protect the same call from two sides.

---

## Mini Example 2: From a Bug Report to a Test

A customer writes: "The cart page says I have **012 items**. I have 3." The code that builds that text

src/cart.ts

```typescript
export interface CartLine {
  name: string;
  pricePaise: number;
  quantity: number;
}

export interface Cart {
  lines: CartLine[];
}

export function itemCount(cart: Cart): number {
  let count = 0;
  for (const line of cart.lines) {
    count += line.quantity;
  }
  return count;
}

export function cartSummary(body: string): string {
  const cart = JSON.parse(body) as Cart;
  return `${itemCount(cart)} items`;
}
```

**Step 1: a test that shows the bug.** Before you change any code, make the bug happen in a test. The website sends JSON, and numbers that arrive as text are a common cause of odd results like "012", so try that first

src/cart.test.ts

```typescript
import { expect, it } from "vitest";
import { cartSummary } from "./cart.ts";

it("counts the items when the website sends quantities as text", () => {
  const body = JSON.stringify({
    lines: [
      { name: "Keyboard", pricePaise: 149900, quantity: "1" },
      { name: "Mouse", pricePaise: 49900, quantity: "2" }
    ]
  });
  expect(cartSummary(body)).toBe("3 items");
});
```

```bash
npm test -- src/cart.test.ts
```

```text
 × src/cart.test.ts > counts the items when the website sends quantities as text 6ms
   → expected '012 items' to be '3 items' // Object.is equality
```

The bug is reproduced: `012 items`. `npm run check` and `npm run lint` are both quiet.

**Step 2: look at the values.** In the JavaScript Debug Terminal, set a breakpoint on `count += line.quantity;` in `cart.ts` and run only this test file

```bash
npx vitest run src/cart.test.ts
```

Vitest runs the test in its own Node.js process, and the debug terminal connects to it. At the first stop, the VARIABLES panel shows `count` as `0` and `line.quantity` as `"1"` in quotes: text, not a number. In the DEBUG CONSOLE, `typeof line.quantity` answers `'string'`. Step Over once, and `count` becomes `"01"`: `+` with text joins text (JavaScript course, Session 04). After the second line it is `"012"`.

**Step 3: the cause.** The type says `quantity: number`, but `JSON.parse(body) as Cart` (Session 13) checked nothing. `tsc` believed the `as`, ESLint has no rule against it in the recommended set, and the website sent text. This is Session 26's lesson: data from outside is checked at run time. Use the shared `validate()`; Joi's `number()` also turns the text `"1"` into the number 1

src/cart.ts

```typescript
import Joi from "joi";
import { validate } from "./validation.ts";
```

and, below `itemCount`

```typescript
const lineSchema = Joi.object<CartLine, true>({
  name: Joi.string().required(),
  pricePaise: Joi.number().integer().min(0).required(),
  quantity: Joi.number().integer().min(1).required()
});

const cartSchema = Joi.object<Cart, true>({
  lines: Joi.array().items(lineSchema).required()
});

export function cartSummary(body: string): string {
  const cart = validate(cartSchema, JSON.parse(body), "cart");
  return `${itemCount(cart)} items`;
}
```

**Step 4: the same test, green.**

```text
 ✓ src/cart.test.ts > counts the items when the website sends quantities as text 4ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

The test stays in the project. If someone removes the validation one day, it fails again. A test written for a bug that was fixed is called a **regression test**: it stops the bug from coming back.

The whole project now passes every check

```text
> 29-quality@1.0.0 check
> tsc --noEmit
```

```text
 Test Files  5 passed (5)
      Tests  12 passed (12)
```

![A bug report becomes a fix in six steps: the report of 012 items, a failing test, the debugger showing the quantity as text, the cause JSON.parse with as, the Joi validation that turns text into numbers, and the same test passing as a guard](images/29-linting-testing-debugging/bug-report.gif)

---

## Beginner Mistakes

### Mistake 1: Trusting green tests to check the types

Vitest removes the types and runs the code, exactly like `node file.ts`. A test that passes `"18"` where a `number` belongs can be green. Run `npm run check` and `npm test`, both, before every commit.

---

### Mistake 2: Calling the function inside expect for toThrow

`expect(shippingPaise("moon", 19900)).toThrow(...)` calls `shippingPaise` **before** `expect` runs, so the error ends the test instead of being checked

```text
 × src/shipping.test.ts > shippingPaise > refuses a zone that is not in the type 4ms
   → unknown zone: moon
```

Pass a function: `expect(() => shippingPaise("moon", 19900)).toThrow(...)`.

---

### Mistake 3: Forgetting await before expect(...).rejects

Without `await`, the check is a floating promise inside the test. Vitest 5 fails the test with "was not awaited", and ESLint reports it too

```text
  27:47  error  Async arrow function has no 'await' expression                                                                                                                      @typescript-eslint/require-await
  28:5   error  Promises must be awaited, end with a call to .catch, end with a call to .then with a rejection handler or be explicitly marked as ignored with the `void` operator  @typescript-eslint/no-floating-promises
```

---

### Mistake 4: Building the tests into dist

With the plain `tsc` build, `dist` gets `prices.test.js`, and Vitest finds it there too: every test runs twice

```text
 Test Files  2 passed (2)
      Tests  4 passed (4)
```

(for one test file with two tests). Build with `tsconfig.build.json`, which excludes `*.test.ts`. If `dist` already has test files, delete the folder and build again.

---

### Mistake 5: Running npm audit fix --force after installing ESLint

`npm audit` says the fix would install `eslint@4.0.0`, a version from 2017 that does not understand this setup. The warnings are about tools that run on your computer only. Leave them, and update ESLint the normal way when new versions come out.

---

### Mistake 6: Forcing typescript-eslint onto TypeScript 7

`npm install ... --legacy-peer-deps` installs it anyway, and ESLint then stops

```text
typescript-eslint does not support TS 7.0.
Please see https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/#running-side-by-side-with-typescript-6.0 to run typescript-eslint using the TS 6 API.
```

Use the two aliases from this session: TypeScript 7 for `tsc`, TypeScript 6 for ESLint.

---

### Mistake 7: Turning a lint rule off for the whole project

When a rule is wrong for one line, turn it off for that line, with a reason: `// eslint-disable-next-line rule-name -- reason`. Turning a rule off everywhere because of one line also removes it from the lines where it finds real bugs, like switching off `strict` because of one error.

---

### Mistake 8: Leaving a debugger statement in the code

Without a debugger connected, `debugger;` does nothing, so it is easy to forget. ESLint's `no-debugger` reports it. Prefer breakpoints in the editor: they never end up in a commit.

---

### Mistake 9: Searching dist line numbers in your .ts files

`payments.js:5:15` is line 5 of the built file, and your code has the line at 11. Start built code with `node --enable-source-maps`, and keep `"sourceMap": true` in `tsconfig.json`.

---

### Mistake 10: Expecting types to check mock calls

`vi.fn<Fetcher>()` checks the answers you give the mock (TS2353 for a wrong one), but `toHaveBeenCalledWith(7)` is accepted by `tsc`. Those arguments are checked only when the test runs, so run the tests.

---

## Practice Exercises

### Exercise 1

Write `discountPaise(pricePaise: number, percent: number): number` in `src/prices.ts`, which returns the discount rounded to whole paise. Test 10% of 149900, 15% of 499, and 0%. Which test would have failed without rounding?

### Exercise 2

`shippingPaise` makes shipping free from 50000 paise. Add tests for 49999 and 50000 paise in both zones. Change `>=` to `>` in the function: which tests fail, and what do their messages say?

### Exercise 3

Test `errorChain` from `src/errors.ts` (Session 23): an `AppError` whose `cause` is an `InvalidDataError` should give `"... <- ... is invalid: ..."`. Write one test for a plain `Error` and one for a string thrown as an error.

### Exercise 4

Test `validate()` from `src/validation.ts` directly with `productSchema`: valid data returns a clean object (extra fields removed), and invalid data throws an `InvalidDataError`. Check the error with `toThrow(InvalidDataError)`, then check its `problems` with `toHaveProperty`.

### Exercise 5

In `orders.test.ts`, test `placeOrder` without waiting 50 ms per payment: give `placeOrder` a `charge` parameter of the type `(payment: Payment) => Promise<string>`, pass a `vi.fn` in the test, and check that it was called with `{ orderId: 1001, amountPaise: 49900 }`. What does `mockRejectedValue(new Error("card declined"))` let you test?

### Exercise 6

Add a rule to `eslint.config.js`: a fourth object `{ rules: { eqeqeq: "error" } }` inside `defineConfig`. Write `if (zone == "local")` somewhere and run `npm run lint`. Then try `npx eslint src --fix`: does this rule fix itself?

### Exercise 7

Run `npm test` in a JavaScript Debug Terminal with a breakpoint in `getProduct` on the line with `validate(...)`. When it stops for product 2, what is in `data`? Continue (F5): how many times does it stop for product 2, and why?

### Exercise 8

Set `"sourceMap": false` in `tsconfig.json`, delete `dist`, run `npm run build` and `npm start`. Which file and line does the trace show now? Put `true` back and compare.

---

## Interview Questions

### What is the difference between tsc, a linter and tests?

`tsc` checks that types fit. A linter like ESLint finds patterns that are usually mistakes, such as a promise nobody awaits or an unused variable. Tests run the code and compare results with known answers. Each finds bugs the others miss, so a project uses all three.

### Does Vitest check TypeScript types?

No. It removes the types and runs the code, like `node file.ts`. Type errors in test files, and in the code they test, are found by `tsc --noEmit`, which has to run as well. `expectTypeOf` lines are also checked by `tsc`, not at run time.

### How do you test code that calls an external API?

Pass the function that does the request in as a parameter, and give the test a fake that answers like the API: a good answer, a wrong shape, an error status. The tests then need no server, run in milliseconds and give the same answer every time.

### What is a mock, and what does TypeScript add to it?

A mock is a fake function that records its calls and returns answers you choose, made with `vi.fn()`. With a type argument, `vi.fn<Fetcher>()`, it fits where the real function fits, and a fake answer of the wrong type is an error. The arguments in `toHaveBeenCalledWith` are only checked when the test runs.

### How do you test that an async function rejects?

`await expect(promise).rejects.toThrow("message")`, or `.rejects.toMatchObject({ statusCode: 502 })` for properties. The `await` is required; without it the check may finish after the test, and Vitest 5 fails the test.

### When is @ts-expect-error acceptable in a test?

When the test passes a wrong value on purpose, to test the code that handles values from outside the type system, such as data from JavaScript callers. Write the reason on the same line. If the type later accepts the value, TS2578 reports the comment.

### What is type-aware linting?

Lint rules that ask TypeScript for the types of the code, for example `no-floating-promises`, which has to know that a call returns a promise. typescript-eslint turns it on with `recommendedTypeChecked` and `projectService: true`.

### Why does typescript-eslint need TypeScript 6 in a TypeScript 7 project?

TypeScript 7.0 is a native program without the JavaScript API that tools call; a new API is planned for 7.1. The workaround is two aliases: TypeScript 7 under `@typescript/native` for `tsc`, and `@typescript/typescript6` under the name `typescript` for the tools.

### What is a floating promise, and how do you find them?

A promise that nobody awaits, returns or catches. Its errors become unhandled rejections, and the code after it runs before it finishes. `tsc` does not report it; the `no-floating-promises` lint rule does, and `no-misused-promises` finds `async` callbacks passed where nothing waits for them, like `forEach`.

### What is a source map?

A `.js.map` file that tells which line and column of the built JavaScript came from which place in the `.ts` file. With `--enable-source-maps`, Node.js uses it so that stack traces of built code show `.ts` lines; editors and debuggers use it too.

### Why are the line numbers right when you run node src/app.ts?

Node.js replaces the types with spaces instead of removing them, so every line and column of the running code is where it is in the `.ts` file. No source map is needed. Code built by `tsc` is rewritten and needs the `.js.map` files.

### What is a regression test?

A test written for a bug that was fixed. It first reproduces the bug and fails, then passes after the fix, and stays in the project so that the same bug can not come back unnoticed.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [28. Migrating JavaScript to TypeScript](28-migrating-javascript-to-typescript.md) | [Course outline](README.md#course-outline) | [30. Node.js with TypeScript](30-nodejs-with-typescript.md) |
