# null, undefined and Optional Values

> **Session 11 of 33** | Part 2: Typing Your Data | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Two Kinds of "Nothing"](#two-kinds-of-nothing)
- [What strictNullChecks Does](#what-strictnullchecks-does)
- [Where undefined and null Come From](#where-undefined-and-null-come-from)
- [The "Possibly" Errors](#the-possibly-errors)
- [Checking for undefined and null](#checking-for-undefined-and-null)
- [Optional Chaining: ?.](#optional-chaining-)
- [Default Values: ??](#default-values-)
- [?? or ||?](#-or-)
- [Filling In a Default: ??=](#filling-in-a-default-)
- [The Non-Null Assertion: !](#the-non-null-assertion-)
- [Throwing Instead of Returning undefined](#throwing-instead-of-returning-undefined)
- [phone?, undefined or null?](#phone-undefined-or-null)
- [null in JSON](#null-in-json)
- [Destructuring Defaults Ignore null](#destructuring-defaults-ignore-null)
- [Which Tool to Use](#which-tool-to-use)
- [Mini Example 1: Looking Up Orders](#mini-example-1-looking-up-orders)
- [Mini Example 2: Server Options with Defaults](#mini-example-2-server-options-with-defaults)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `11-null-undefined` inside `ts-practice`, with the one-line `tsconfig.json`

11-null-undefined/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## Two Kinds of "Nothing"

JavaScript has two values for "no value", and TypeScript has a type for each

| Value       | Means                         | Where it comes from                                                   |
| ----------- | ----------------------------- | --------------------------------------------------------------------- |
| `undefined` | "not set", "not there"        | A missing property, an array index that does not exist, `find` with no match, a function with no `return` |
| `null`      | "empty, on purpose"           | Someone wrote it: a JSON field like `"deliveredOn": null`, `match` with no match, a database lookup that found nothing |

Both are their own types: the type `undefined` has only the value `undefined`, and the type `null` has only `null`. A value that may be missing is a union with one of them, like `string | undefined`, which you have already met many times.

A convention many TypeScript projects follow, and this course too

- `undefined` for "not given" in your own code: optional properties, a lookup that found nothing.
- `null` where the data itself says "empty": JSON from an API, a database field.

![Two shelves: undefined is an empty slot where nothing was ever put, null is a slot with a card that says "nothing" put there on purpose](images/11-null-undefined-and-optional-values/two-nothings.gif)

---

## What strictNullChecks Does

Find an order that does not exist

find-order.ts

```typescript
const orders = [
  { id: 1001, customer: "Riya" },
  { id: 1002, customer: "Sara" }
];

const order = orders.find((item) => item.id === 1003);
console.log(order.customer.toUpperCase());
```

```text
find-order.ts:7:13 - error TS18048: 'order' is possibly 'undefined'.

7 console.log(order.customer.toUpperCase());
              ~~~~~


Found 1 error in find-order.ts:7
```

`find` returns `undefined` when nothing matches, and `tsc` will not let you use the result without handling that. This check comes from the setting `strictNullChecks`, which `strict` turns on (Session 02).

To see what TypeScript would be without it, turn it off in this folder only. `exactOptionalPropertyTypes` builds on `strictNullChecks` (without it, `tsc` stops with TS5052), so it must go too

11-null-undefined/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "strictNullChecks": false,
    "exactOptionalPropertyTypes": false
  }
}
```

```bash
npx tsc --noEmit
node find-order.ts
```

`tsc` reports nothing, and Node.js crashes

```text
file:///D:/ts-practice/11-null-undefined/find-order.ts:7
console.log(order.customer.toUpperCase());
                  ^

TypeError: Cannot read properties of undefined (reading 'customer')
    at file:///D:/ts-practice/11-null-undefined/find-order.ts:7:19
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Without `strictNullChecks`, `null` and `undefined` fit into **every** type, so `tsc` believes `order` is always an order. "Cannot read properties of undefined" is one of the most common crashes in JavaScript, and this setting is what prevents it. Change the `tsconfig.json` back to the one line, and never turn this setting off in a real project.

![With strictNullChecks off, tsc gives a green check and Node.js crashes with Cannot read properties of undefined; with it on, tsc stops at TS18048 before the code runs](images/11-null-undefined-and-optional-values/strict-off-crash.gif)

---

## Where undefined and null Come From

sources.ts

```typescript
const names = ["Riya", "Sara"];
const orders = [{ id: 1001, customer: "Riya" }];
const stock = new Map([["KB-01", 12]]);

const first = names[0];
const found = orders.find((item) => item.id === 1003);
const keyboards = stock.get("KB-01");
const port = process.env.PORT;
const digits = "ORD-1001".match(/[0-9]+/);
const parsed: unknown = JSON.parse("null");
```

Point at each name in VS Code to see its type

| Code                                   | Type                                     | Why                                           |
| -------------------------------------- | ---------------------------------------- | --------------------------------------------- |
| `names[0]`                             | `string \| undefined`                    | The array may be shorter (Session 06)         |
| `orders.find(...)`                     | `{ id: number; customer: string; } \| undefined` | Maybe nothing matches                 |
| `stock.get("KB-01")`                   | `number \| undefined`                    | A `Map` may not have that key                 |
| `process.env.PORT`                     | `string \| undefined`                    | The variable may not be set (Session 05)      |
| `customer.phone` with `phone?: string` | `string \| undefined`                    | The property may be left out (Session 07)     |
| `"ORD-1001".match(/[0-9]+/)`           | `RegExpMatchArray \| null`               | `match` returns `null` when nothing matches   |
| `JSON.parse("null")`                   | `any`, so use `unknown` (Session 10)     | JSON can contain `null` anywhere              |

`match` is a good example of `null` from JavaScript itself

match.ts

```typescript
console.log("ORD-1001".match(/[0-9]+/));
console.log("WEB".match(/[0-9]+/));
```

```text
[ '1001', index: 4, input: 'ORD-1001', groups: undefined ]
null
```

Later in the course you will meet more: a Mongoose `findById` returns `null` when no document has that id (Session 32).

![Values flow from find, Map.get, an array index, process.env and an optional property into a T | undefined box, and from match, JSON and a database lookup into a T | null box](images/11-null-undefined-and-optional-values/where-from.gif)

---

## The "Possibly" Errors

Five messages, one problem: the value may not be there

messages.ts

```typescript
const digits = "ORD-1001".match(/[0-9]+/);
console.log(digits[0]);

console.log("WEB".match(/[0-9]+/)[0]);

const token = "Basic".split(" ")[1];
console.log(token.length);
console.log("Basic".split(" ")[1].length);

function phoneDigits(phone: string | null | undefined): number {
  return phone.length;
}
```

```text
messages.ts:2:13 - error TS18047: 'digits' is possibly 'null'.

2 console.log(digits[0]);
              ~~~~~~

messages.ts:4:13 - error TS2531: Object is possibly 'null'.

4 console.log("WEB".match(/[0-9]+/)[0]);
              ~~~~~~~~~~~~~~~~~~~~~

messages.ts:7:13 - error TS18048: 'token' is possibly 'undefined'.

7 console.log(token.length);
              ~~~~~

messages.ts:8:13 - error TS2532: Object is possibly 'undefined'.

8 console.log("Basic".split(" ")[1].length);
              ~~~~~~~~~~~~~~~~~~~~~

messages.ts:11:10 - error TS18049: 'phone' is possibly 'null' or 'undefined'.

11   return phone.length;
            ~~~~~


Found 5 errors in the same file, starting at: messages.ts:2
```

| Code     | Message                                  | When                                    |
| -------- | ---------------------------------------- | --------------------------------------- |
| TS18048  | `'x' is possibly 'undefined'.`           | A named value that may be `undefined`   |
| TS18047  | `'x' is possibly 'null'.`                | A named value that may be `null`        |
| TS18049  | `'x' is possibly 'null' or 'undefined'.` | A named value that may be either        |
| TS2532   | `Object is possibly 'undefined'.`        | The same, for a value without a name    |
| TS2531   | `Object is possibly 'null'.`             | The same, for a value without a name    |

All five are fixed with the tools in the rest of this session.

---

## Checking for undefined and null

You did this in Session 10: compare, and `tsc` narrows. For a value that may be **either**, one comparison covers both

phone-line.ts

```typescript
function phoneLine(phone: string | null | undefined): string {
  if (phone == null) {
    return "no phone";
  }
  return `phone: ${phone} (${phone.length} digits)`;
}

console.log(phoneLine("9876543210"));
console.log(phoneLine(null));
console.log(phoneLine(undefined));
```

```text
phone: 9876543210 (10 digits)
no phone
no phone
```

`phone == null`, with two `=`, is true for `null` **and** `undefined`, and for nothing else. It is the one place where many teams allow `==` instead of `===` (ESLint's `eqeqeq` rule has an option for exactly this). With `===`, you check only one of the two

phone-null.ts

```typescript
function phoneDigits(phone: string | null | undefined): number {
  if (phone === null) {
    return 0;
  }
  return phone.length;
}
```

```text
phone-null.ts:5:10 - error TS18048: 'phone' is possibly 'undefined'.

5   return phone.length;
           ~~~~~


Found 1 error in phone-null.ts:5
```

`null` is gone, `undefined` is still there.

| Check                | Removes                    |
| -------------------- | -------------------------- |
| `x === undefined`    | `undefined`                |
| `x === null`         | `null`                     |
| `x == null`          | both                       |
| `if (x)`             | both, but also `0`, `""` and `false` (Session 10) |

![Three values, a phone number, null and undefined, go through === null and == null: the first lets undefined slip through, the second stops both](images/11-null-undefined-and-optional-values/equals-null.gif)

---

## Optional Chaining: ?.

Reading through something that may be missing is so common that JavaScript has an operator for it

address.ts

```typescript
interface Address {
  city: string;
  pincode: string;
}

interface Customer {
  name: string;
  address?: Address;
}

const riya: Customer = { name: "Riya", address: { city: "Pune", pincode: "411001" } };
const sara: Customer = { name: "Sara" };

console.log(riya.address?.city);
console.log(sara.address?.city);
console.log(sara.address?.city ?? "city not given");
```

```text
Pune
undefined
city not given
```

`a?.b` reads `b` when `a` is there. When `a` is `null` or `undefined`, it stops and gives `undefined`, instead of crashing. Without the `?`

```typescript
console.log(sara.address.city);
```

```text
address.ts:14:13 - error TS18048: 'sara.address' is possibly 'undefined'.

14 console.log(sara.address.city);
               ~~~~~~~~~~~~


Found 1 error in address.ts:14
```

In JavaScript, the same line is the crash from before

address.js

```javascript
const sara = { name: "Sara" };
console.log(sara.address.city);
```

```text
file:///D:/ts-practice/11-null-undefined/address.js:2
console.log(sara.address.city);
                         ^

TypeError: Cannot read properties of undefined (reading 'city')
    at file:///D:/ts-practice/11-null-undefined/address.js:2:26
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

The result of `?.` can always be `undefined`, so its type says so. `sara.address?.city` is `string | undefined`, not `string`

```typescript
const city: string = sara.address?.city;
```

```text
address.ts:14:7 - error TS2322: Type 'string | undefined' is not assignable to type 'string'.
  Type 'undefined' is not assignable to type 'string'.

14 const city: string = sara.address?.city;
         ~~~~


Found 1 error in address.ts:14
```

| Form            | Example                       | Means                                     |
| --------------- | ----------------------------- | ----------------------------------------- |
| Property        | `customer.address?.city`      | Read `city` if `address` is there         |
| Index           | `orders[0]?.items`            | Read `items` if the first order exists    |
| Index after `?` | `names?.[0]`                  | Index only if `names` is there            |
| Method call     | `customer.notify?.()`         | Call `notify` only if it exists           |

![sara.address?.city: the chain stops at the missing address and gives undefined; ?? then turns undefined into "city not given". For riya the chain goes all the way to Pune](images/11-null-undefined-and-optional-values/optional-chaining.gif)

---

## Default Values: ??

You have used `??` since Session 05: `a ?? b` gives `a`, unless `a` is `null` or `undefined`, then `b`. With a default, the `undefined` is gone from the type: `sara.address?.city ?? "city not given"` is a plain `string`.

`?.` and `??` are a pair: `?.` reads safely and may give `undefined`, `??` replaces that `undefined` with something useful.

---

## ?? or ||?

Older JavaScript code uses `||` for defaults. It is not the same

quantity.ts

```typescript
const quantities = [3, 0, undefined];

for (const quantity of quantities) {
  console.log(`|| gives ${quantity || 1}, ?? gives ${quantity ?? 1}`);
}
```

```text
|| gives 3, ?? gives 3
|| gives 1, ?? gives 0
|| gives 1, ?? gives 1
```

`||` replaces every **falsy** value, the same trap as truthiness checks in Session 10: a quantity of `0` became `1`. `??` replaces only `null` and `undefined`.

| Left side   | `\|\| "default"` | `?? "default"` |
| ----------- | ---------------- | -------------- |
| `undefined` | `"default"`      | `"default"`    |
| `null`      | `"default"`      | `"default"`    |
| `0`         | `"default"`      | `0`            |
| `""`        | `"default"`      | `""`           |
| `false`     | `"default"`      | `false`        |

Use `??` for defaults. Use `||` only when `0`, `""` and `false` really should be replaced too.

![The values 3, 0 and undefined go through || 1 and ?? 1: || turns 0 into 1, ?? keeps 0 and only replaces undefined](images/11-null-undefined-and-optional-values/nullish-vs-or.gif)

---

## Filling In a Default: ??=

`a ??= b` assigns `b` to `a` only when `a` is `null` or `undefined`

settings.ts

```typescript
interface Settings {
  pageSize?: number;
  currency?: string;
}

const settings: Settings = { currency: "INR" };
settings.pageSize ??= 20;
settings.currency ??= "USD";
console.log(settings);
```

```text
{ currency: 'INR', pageSize: 20 }
```

`pageSize` was missing, so it got `20`. `currency` was already set, so it stayed `"INR"`. It is short for `if (settings.pageSize === undefined || settings.pageSize === null) { settings.pageSize = 20; }`.

---

## The Non-Null Assertion: !

A `!` after a value tells `tsc`: "this is not `null` or `undefined`, trust me"

non-null.ts

```typescript
const orders = [
  { id: 1001, customer: "Riya" },
  { id: 1002, customer: "Sara" }
];

const first = orders.find((item) => item.id === 1001)!;
console.log(first.customer);

const missing = orders.find((item) => item.id === 1003)!;
console.log(missing.customer);
```

```text
Riya
file:///D:/ts-practice/11-null-undefined/non-null.ts:10
console.log(missing.customer);
                    ^

TypeError: Cannot read properties of undefined (reading 'customer')
    at file:///D:/ts-practice/11-null-undefined/non-null.ts:10:21
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` reported nothing for either line. The `!` removed `undefined` from the type, but not from the value, so the second line crashed exactly as if `strictNullChecks` were off.

Like a type guard in Session 10, `!` is a promise that `tsc` does not check. It is only safe when you know something `tsc` cannot see, and you are sure. In almost every case, a check or a default is better. Session 13 covers this kind of "trust me" code in general.

![The ! after find(...) peels the undefined label off the type, but the value is still undefined, and the next line crashes](images/11-null-undefined-and-optional-values/non-null-crash.gif)

---

## Throwing Instead of Returning undefined

When a missing value is a real error, handle it once, in a function that throws

require-order.ts

```typescript
interface Order {
  readonly id: number;
  customer: string;
}

const orders: Order[] = [
  { id: 1001, customer: "Riya" },
  { id: 1002, customer: "Sara" }
];

function requireOrder(id: number): Order {
  const order = orders.find((item) => item.id === id);
  if (order === undefined) {
    throw new Error(`Order ${id} not found`);
  }
  return order;
}

console.log(requireOrder(1002).customer);
console.log(requireOrder(1003).customer);
```

```text
Sara
file:///D:/ts-practice/11-null-undefined/require-order.ts:14
    throw new Error(`Order ${id} not found`);
          ^

Error: Order 1003 not found
    at requireOrder (file:///D:/ts-practice/11-null-undefined/require-order.ts:14:11)
    at file:///D:/ts-practice/11-null-undefined/require-order.ts:20:13
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Compare with the `!` version: it still stops, but with `Order 1003 not found` instead of "Cannot read properties of undefined", and every caller gets an `Order`, never `undefined`. In an Express app, this is the place where you answer with "404 Not Found" (Session 31).

| Return type         | Use when                                          | The caller                       |
| ------------------- | ------------------------------------------------- | -------------------------------- |
| `Order \| undefined` | "Not found" is normal, like a search             | Must check, like after `find`    |
| `Order`, and throw   | "Not found" is an error, like a wrong id in a URL | Can use the result directly     |

---

## phone?, undefined or null?

Three ways to write a property that may have no value. They are not the same

phone-shapes.ts

```typescript
interface MaybeMissing {
  phone?: string;
}

interface MaybeUndefined {
  phone: string | undefined;
}

interface MaybeNull {
  phone: string | null;
}

const a: MaybeMissing = {};
const b: MaybeUndefined = {};
const c: MaybeNull = {};

const d: MaybeMissing = { phone: undefined };
const e: MaybeUndefined = { phone: undefined };
const f: MaybeNull = { phone: null };
```

```text
phone-shapes.ts:14:7 - error TS2741: Property 'phone' is missing in type '{}' but required in type 'MaybeUndefined'.

14 const b: MaybeUndefined = {};
         ~

  phone-shapes.ts:6:3 - 'phone' is declared here.
    6   phone: string | undefined;
        ~~~~~

phone-shapes.ts:15:7 - error TS2741: Property 'phone' is missing in type '{}' but required in type 'MaybeNull'.

15 const c: MaybeNull = {};
         ~

  phone-shapes.ts:10:3 - 'phone' is declared here.
    10   phone: string | null;
         ~~~~~

phone-shapes.ts:17:7 - error TS2375: Type '{ phone: undefined; }' is not assignable to type 'MaybeMissing' with 'exactOptionalPropertyTypes: true'. Consider adding 'undefined' to the types of the target's properties.
  Types of property 'phone' are incompatible.
    Type 'undefined' is not assignable to type 'string'.

17 const d: MaybeMissing = { phone: undefined };
         ~


Found 3 errors in the same file, starting at: phone-shapes.ts:14
```

| Property                   | `{}`             | `{ phone: undefined }` | `{ phone: null }` | Means                               |
| -------------------------- | ---------------- | ---------------------- | ----------------- | ----------------------------------- |
| `phone?: string`           | Yes              | No (TS2375)            | No (TS2322)       | May be left out                     |
| `phone: string \| undefined` | No (TS2741)    | Yes                    | No (TS2322)       | Must be written, may be `undefined` |
| `phone: string \| null`    | No (TS2741)      | No (TS2322)            | Yes               | Must be written, `null` means "none" |

The TS2375 comes from `exactOptionalPropertyTypes` in your tsconfig (Session 07): "optional" means "may be left out", not "may be `undefined`". Without that setting, `phone?: string` would also accept `phone: undefined`.

In practice

- **`phone?: string`** for your own data where a property may be missing. This is the most common one.
- **`phone: string | null`** for data that always has the key, like JSON from an API or a database row with an empty column.
- **`phone: string | undefined`** is rare: it forces every object to write `phone`, even when there is none.

![Three property shapes as three slots; the objects {}, { phone: undefined } and { phone: null } try each slot, and each slot accepts exactly one of them](images/11-null-undefined-and-optional-values/phone-shapes.gif)

---

## null in JSON

json-null.ts

```typescript
const order = { id: 1001, couponCode: undefined, deliveredOn: null };

console.log(order);
console.log(JSON.stringify(order));
```

```text
{ id: 1001, couponCode: undefined, deliveredOn: null }
{"id":1001,"deliveredOn":null}
```

JSON has no `undefined`. `JSON.stringify` silently drops a property whose value is `undefined`, and keeps `null`. That is why APIs send `"deliveredOn": null` when an order is not delivered yet: the field stays in the response, so the receiver can tell "not delivered" apart from "this API has no such field". And it is why a type for JSON data uses `string | null`, not `string | undefined`.

![An object with couponCode: undefined and deliveredOn: null goes through JSON.stringify: couponCode falls out, deliveredOn: null comes out the other side](images/11-null-undefined-and-optional-values/json-null.gif)

---

## Destructuring Defaults Ignore null

A default in destructuring looks like `??`, but it is not

page-size.ts

```typescript
interface ListOptions {
  pageSize?: number | null;
}

function pageSizeOf(options: ListOptions): number {
  const { pageSize = 20 } = options;
  return pageSize;
}
```

```text
page-size.ts:7:3 - error TS2322: Type 'number | null' is not assignable to type 'number'.
  Type 'null' is not assignable to type 'number'.

7   return pageSize;
    ~~~~~~


Found 1 error in page-size.ts:7
```

`tsc` knows a JavaScript detail many developers miss: `= 20` is used only when the value is `undefined`, not when it is `null`

page-size.js

```javascript
function pageSizeOf(options) {
  const { pageSize = 20 } = options;
  return pageSize;
}

console.log(pageSizeOf({}));
console.log(pageSizeOf({ pageSize: undefined }));
console.log(pageSizeOf({ pageSize: null }));
```

```text
20
20
null
```

The same is true for default parameter values (Session 12). When `null` is possible, use `??`

page-size.ts

```typescript
interface ListOptions {
  pageSize?: number | null;
}

function pageSizeOf(options: ListOptions): number {
  return options.pageSize ?? 20;
}

console.log(pageSizeOf({}));
console.log(pageSizeOf({ pageSize: null }));
console.log(pageSizeOf({ pageSize: 50 }));
```

```text
20
20
50
```

![const { pageSize = 20 }: an undefined pageSize gets the default 20, a null pageSize passes the default untouched and comes out as null; with ?? both get 20](images/11-null-undefined-and-optional-values/destructure-null.gif)

---

## Which Tool to Use

| You want to                                         | Use                                    |
| --------------------------------------------------- | -------------------------------------- |
| Do something different when the value is missing    | A check: `=== undefined`, `== null`    |
| Read a property through something that may be missing | `?.`                                 |
| Use a default when the value is missing             | `??`                                   |
| Fill in a missing property                          | `??=`                                  |
| Stop with a clear error when it is missing          | A function that throws                 |
| Tell `tsc` "trust me"                               | `!`, but almost never                  |

---

## Mini Example 1: Looking Up Orders

A small command-line tool: give it order ids, and it prints what it knows about each

orders.ts

```typescript
interface Delivery {
  city: string;
  deliveredOn: string | null;
}

interface Order {
  readonly id: number;
  customer: string;
  couponCode?: string;
  delivery?: Delivery;
}

const orders: Order[] = [
  { id: 1001, customer: "Riya", couponCode: "FESTIVE20", delivery: { city: "Pune", deliveredOn: "2026-10-20" } },
  { id: 1002, customer: "Sara", delivery: { city: "Mumbai", deliveredOn: null } },
  { id: 1003, customer: "Aman" }
];

function findOrder(id: number): Order | undefined {
  return orders.find((order) => order.id === id);
}

function deliveryLine(order: Order): string {
  if (order.delivery === undefined) {
    return "not shipped yet";
  }
  const { city, deliveredOn } = order.delivery;
  if (deliveredOn === null) {
    return `on the way to ${city}`;
  }
  return `delivered in ${city} on ${deliveredOn}`;
}

for (const text of process.argv.slice(2)) {
  const order = findOrder(Number(text));
  if (order === undefined) {
    console.log(`#${text}: not found`);
    continue;
  }
  const coupon = order.couponCode ?? "no coupon";
  console.log(`#${order.id} ${order.customer}: ${deliveryLine(order)} (${coupon})`);
}

console.log(`City of 1002: ${findOrder(1002)?.delivery?.city ?? "unknown"}`);
console.log(`City of 1003: ${findOrder(1003)?.delivery?.city ?? "unknown"}`);
```

```bash
npx tsc --noEmit
node orders.ts 1001 1002 1003 9999
```

```text
#1001 Riya: delivered in Pune on 2026-10-20 (FESTIVE20)
#1002 Sara: on the way to Mumbai (no coupon)
#1003 Aman: not shipped yet (no coupon)
#9999: not found
City of 1002: Mumbai
City of 1003: unknown
```

| Value                      | Type                      | Handled with                         |
| -------------------------- | ------------------------- | ------------------------------------ |
| `findOrder(...)`           | `Order \| undefined`      | `if (order === undefined)` and `continue` |
| `order.couponCode`         | `string \| undefined`     | `?? "no coupon"`                     |
| `order.delivery`           | `Delivery \| undefined`   | `if (order.delivery === undefined)`  |
| `deliveredOn`              | `string \| null`          | `if (deliveredOn === null)`          |
| `findOrder(1003)?.delivery?.city` | `string \| undefined` | `?.` twice, then `?? "unknown"`    |

Every kind of "nothing" from this session in one file: `undefined` from `find`, from optional properties, and `null` with a meaning ("shipped, not delivered yet").

Now delete the `if (order === undefined) { ... }` block in the loop

```text
orders.ts:37:18 - error TS18048: 'order' is possibly 'undefined'.

37   const coupon = order.couponCode ?? "no coupon";
                    ~~~~~

orders.ts:38:19 - error TS18048: 'order' is possibly 'undefined'.

38   console.log(`#${order.id} ${order.customer}: ${deliveryLine(order)} (${coupon})`);
                     ~~~~~

orders.ts:38:31 - error TS18048: 'order' is possibly 'undefined'.

38   console.log(`#${order.id} ${order.customer}: ${deliveryLine(order)} (${coupon})`);
                                 ~~~~~

orders.ts:38:63 - error TS2345: Argument of type 'Order | undefined' is not assignable to parameter of type 'Order'.
  Type 'undefined' is not assignable to type 'Order'.

38   console.log(`#${order.id} ${order.customer}: ${deliveryLine(order)} (${coupon})`);
                                                                 ~~~~~


Found 4 errors in the same file, starting at: orders.ts:37
```

Four places that would crash for `9999`. The `??` on line 37 does not help: it protects against a missing `couponCode`, not a missing `order`.

![The ids 1001, 1002, 1003 and 9999 go into findOrder; three come out as orders and take different delivery paths (delivered, on the way, not shipped), and 9999 comes out as undefined and prints not found](images/11-null-undefined-and-optional-values/order-lookup.gif)

---

## Mini Example 2: Server Options with Defaults

Every server has options with defaults. Here `null` has a meaning of its own: `maxUploadMb: null` means "no upload limit", which is different from "not set, use the default"

server-options.ts

```typescript
type LogLevel = "debug" | "info" | "warn" | "error";

interface ServerOptions {
  port?: number;
  host?: string;
  logLevel?: LogLevel;
  maxUploadMb?: number | null;
}

interface ResolvedOptions {
  port: number;
  host: string;
  logLevel: LogLevel;
  maxUploadMb: number | null;
}

function resolveOptions(options: ServerOptions): ResolvedOptions {
  return {
    port: options.port ?? 3000,
    host: options.host ?? "localhost",
    logLevel: options.logLevel ?? "info",
    maxUploadMb: options.maxUploadMb === undefined ? 10 : options.maxUploadMb
  };
}

function optionsLine(options: ResolvedOptions): string {
  const upload = options.maxUploadMb === null ? "no upload limit" : `uploads up to ${options.maxUploadMb} MB`;
  return `${options.host}:${options.port}, ${options.logLevel}, ${upload}`;
}

const configs: ServerOptions[] = [
  {},
  { port: 0, logLevel: "debug" },
  { host: "0.0.0.0", maxUploadMb: null },
  { port: 8080, maxUploadMb: 50 }
];

for (const config of configs) {
  console.log(optionsLine(resolveOptions(config)));
}
```

```bash
npx tsc --noEmit
node server-options.ts
```

```text
localhost:3000, info, uploads up to 10 MB
localhost:0, debug, uploads up to 10 MB
0.0.0.0:3000, info, no upload limit
localhost:8080, info, uploads up to 50 MB
```

Two types: `ServerOptions`, where everything is optional, and `ResolvedOptions`, where nothing is. `resolveOptions` turns one into the other, and the rest of the app only sees the resolved one, with no `undefined` to check.

Port `0` is a real setting: in Node.js, listening on port `0` asks the system for any free port, which is handy in tests. Change the first line of `resolveOptions` to the older style, `port: options.port || 3000`

```text
localhost:3000, info, uploads up to 10 MB
localhost:3000, debug, uploads up to 10 MB
0.0.0.0:3000, info, no upload limit
localhost:8080, info, uploads up to 50 MB
```

The second server now uses port 3000, and `tsc` cannot notice: both versions are type-correct. Change it back to `??`.

Now try `??` for the upload limit too: `maxUploadMb: options.maxUploadMb ?? 10`

```text
localhost:3000, info, uploads up to 10 MB
localhost:0, debug, uploads up to 10 MB
0.0.0.0:3000, info, uploads up to 10 MB
localhost:8080, info, uploads up to 50 MB
```

The third server asked for "no limit" and got 10 MB, because `??` treats `null` like `undefined`. When `null` means something, check for `undefined` only, as the original code does.

And if you forget the default completely, `maxUploadMb: options.maxUploadMb`, the second type catches it

```text
server-options.ts:22:5 - error TS2322: Type 'number | null | undefined' is not assignable to type 'number | null'.
  Type 'undefined' is not assignable to type 'number | null'.

22     maxUploadMb: options.maxUploadMb
       ~~~~~~~~~~~

  server-options.ts:14:3 - The expected type comes from property 'maxUploadMb' which is declared here on type 'ResolvedOptions'
    14   maxUploadMb: number | null;
         ~~~~~~~~~~~


Found 1 error in server-options.ts:22
```

One more: with `exactOptionalPropertyTypes`, `{ port: undefined }` in `configs` is an error (TS2375, as in `phone-shapes.ts`). To use the default, leave the property out.

![Four configs go through resolveOptions; port 0 survives ?? but becomes 3000 with ||, and maxUploadMb null survives the undefined check but becomes 10 with ??](images/11-null-undefined-and-optional-values/options-defaults.gif)

---

## Beginner Mistakes

### Mistake 1: Turning off strictNullChecks

Without it, `null` and `undefined` fit into every type, `tsc` reported nothing for `order.customer`, and Node.js crashed with `Cannot read properties of undefined`. Keep `strict` on.

---

### Mistake 2: Using ! to make the error go away

`orders.find(...)!` removes `undefined` from the type, not from the value, and the code crashed on the next line. Check the value, give a default, or throw a clear error.

---

### Mistake 3: Using || for a default

`quantity || 1` turned `0` into `1`, and `options.port || 3000` turned port `0` into `3000`. Use `??`, which only replaces `null` and `undefined`.

---

### Mistake 4: Using ?? when null has a meaning

`options.maxUploadMb ?? 10` turned "no limit" (`null`) into 10 MB. When `null` means something, check `=== undefined` only.

---

### Mistake 5: Forgetting that find can return undefined

`orders.find(...).customer` gives `error TS18048: 'order' is possibly 'undefined'.` The same goes for `Map.get`, `match` (`null`) and array indexes.

---

### Mistake 6: Treating the result of ?. as a full value

`const city: string = sara.address?.city` gives `error TS2322: Type 'string | undefined' is not assignable to type 'string'.` Add a default with `??`, or keep the type `string | undefined`.

---

### Mistake 7: Checking only null when undefined is possible

After `if (phone === null) return 0;`, `phone.length` still gives `error TS18048: 'phone' is possibly 'undefined'.` Use `phone == null` to check both.

---

### Mistake 8: Expecting a destructuring default to replace null

`const { pageSize = 20 } = options` keeps `null` as `null`. `tsc` shows it with `Type 'number | null' is not assignable to type 'number'.` Use `options.pageSize ?? 20`.

---

### Mistake 9: Writing undefined into an optional property

`{ phone: undefined }` for `phone?: string` gives `error TS2375` with `exactOptionalPropertyTypes`. Leave the property out.

---

### Mistake 10: Using null and undefined for the same thing

When some code writes `null` and other code `undefined` for "no phone", every check must handle both. Pick one meaning for each: `undefined` for "not given", `null` for "empty on purpose".

---

### Mistake 11: Putting ?. everywhere, just in case

`order?.customer?.name` on values that can never be missing hides real bugs: when something is wrong, you get a silent `undefined` instead of an error. Use `?.` where the type says the value may be missing.

---

## Practice Exercises

### Exercise 1

Make `const prices = new Map([["KB-01", 149900], ["MS-02", 49900]]);` and write `priceLine(sku: string): string` that returns `KB-01: Rs. 1499.00`, or `MN-03: no price` for a code that is not in the map. Solve it once with a check and once with `??`.

### Exercise 2

Add `landmark?: string` to `Address` in `address.ts`. Print `Pune, near FC Road` for a customer with a landmark, `Pune` without one, and `no address` for Sara, using `?.` and `??`.

### Exercise 3

For the texts `["Riya", "", undefined]`, print a greeting with the default `"Guest"`, once with `||` and once with `??`. Which result do you want for `""`, and why?

### Exercise 4

Rewrite `non-null.ts` without `!`. Use a function like `requireOrder` that throws, and make the error message contain the id.

### Exercise 5

Write `interface Product { name: string; pricePaise: number; discountPercent: number | null }`, where `null` means "no discount". Print the final price of three products. Why is `discountPercent?: number` a worse choice here, if the data comes from JSON?

### Exercise 6

Parse `'{ "id": 1001, "deliveredOn": null }'` as `unknown`, and write a type guard `isDeliveryStatus(value: unknown): value is DeliveryStatus` for `interface DeliveryStatus { id: number; deliveredOn: string | null }`. Test it with `"deliveredOn": "2026-10-20"`, `null`, and `42`.

### Exercise 7

In Mini Example 1, add `phone?: string` to `Order` and print `call 9876543210` or `no phone` for each order. Then run `node orders.ts abc`. What does `Number("abc")` give, and why does the output say "not found"?

### Exercise 8

In Mini Example 2, add `timeoutSeconds?: number | null` with the default `30`, where `null` means "no timeout" and `0` is a valid value. Add configs that test all four cases: left out, `null`, `0` and `60`.

---

## Interview Questions

### What is the difference between null and undefined?

`undefined` means "not set": a missing property, a variable without a value, `find` with no match. `null` means "empty, on purpose": someone wrote it, like a JSON field or a database lookup that found nothing. In TypeScript, each is its own type.

### What does strictNullChecks do?

It removes `null` and `undefined` from every other type, so a `string` is always a string. A value that may be missing needs a union like `string | undefined`, and must be checked before use. `strict` turns it on.

### What is optional chaining?

The `?.` operator: `a?.b` reads `b` when `a` is not `null` or `undefined`, and gives `undefined` otherwise, without crashing. It also works for indexes (`a?.[0]`) and calls (`a.method?.()`), and its result type includes `undefined`.

### What is the difference between ?? and ||?

`??` uses the right side only when the left side is `null` or `undefined`. `||` uses it for every falsy value, including `0`, `""` and `false`, which is often a bug for numbers and texts.

### What is the non-null assertion operator?

A `!` after a value, like `order!`, which tells TypeScript the value is not `null` or `undefined`. TypeScript does not check it, so if it is wrong, the code crashes at run time. It should be rare.

### What is the difference between phone?: string and phone: string | undefined?

`phone?: string` may be left out of the object. `phone: string | undefined` must be written, but its value may be `undefined`. With `exactOptionalPropertyTypes`, `phone?: string` does not accept an explicit `undefined`.

### Why do APIs often use null instead of undefined?

JSON has no `undefined`: `JSON.stringify` drops properties whose value is `undefined`. `null` keeps the field in the response, so "no value" is sent explicitly.

### How do you check for null and undefined at once?

With `value == null` (loose equality), which is true only for `null` and `undefined`. `value === null` checks only `null`. TypeScript narrows correctly after both.

### What does ??= do?

`a ??= b` assigns `b` to `a` only if `a` is `null` or `undefined`. It is often used to fill in default values.

### Why does Array.prototype.find return T | undefined?

Because no element might match. TypeScript makes the caller handle that case, instead of crashing with "Cannot read properties of undefined".

### Do destructuring defaults apply to null?

No. A default like `const { pageSize = 20 } = options` is used only when the value is `undefined`. A `null` stays `null`. Use `??` when `null` is possible.

### What does exactOptionalPropertyTypes change?

It makes an optional property mean only "may be left out". Without it, `phone?: string` also accepts `phone: undefined`. With it, writing `undefined` explicitly is an error (TS2375), unless the type says `string | undefined`.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [10. Narrowing](10-narrowing.md) | [Course outline](README.md#course-outline) | [12. Typing Functions](12-typing-functions.md) |
