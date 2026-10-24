# Narrowing

> **Session 10 of 33** | Part 2: Typing Your Data | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [What Narrowing Is](#what-narrowing-is)
- [typeof Checks](#typeof-checks)
- [The typeof null Trap](#the-typeof-null-trap)
- [Narrowing Follows Your Code](#narrowing-follows-your-code)
- [Equality Checks](#equality-checks)
- [Comparisons with < and > Do Not Narrow](#comparisons-with--and--do-not-narrow)
- [Truthiness Checks](#truthiness-checks)
- [The in Operator](#the-in-operator)
- [instanceof](#instanceof)
- [Array.isArray](#arrayisarray)
- [Discriminated Unions](#discriminated-unions)
- [Exhaustiveness Checking with never](#exhaustiveness-checking-with-never)
- [Your Own Type Guards](#your-own-type-guards)
- [tsc Can Write the Type Guard Itself](#tsc-can-write-the-type-guard-itself)
- [Checking the Properties of an Unknown Object](#checking-the-properties-of-an-unknown-object)
- [Which Check to Use](#which-check-to-use)
- [Mini Example 1: Shipping a Mixed Cart](#mini-example-1-shipping-a-mixed-cart)
- [Mini Example 2: The Log Level from a .env File](#mini-example-2-the-log-level-from-a-env-file)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `10-narrowing` inside `ts-practice`, with the one-line `tsconfig.json`

10-narrowing/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## What Narrowing Is

In Session 09, `id.toUpperCase()` failed on `string | number`, because `id` might be a number. The fix is JavaScript you already know: check the type first.

format-order-id.ts

```typescript
function formatOrderId(id: string | number): string {
  if (typeof id === "string") {
    return id.toUpperCase();
  }
  return `ORD-${id.toFixed(0).padStart(6, "0")}`;
}

console.log(formatOrderId("web-1002"));
console.log(formatOrderId(1001));
```

```bash
npx tsc --noEmit
node format-order-id.ts
```

```text
WEB-1002
ORD-001001
```

No errors. Point at `id` in VS Code in three places

| Where                                  | Type of `id`        |
| -------------------------------------- | ------------------- |
| The parameter                          | `string \| number`  |
| Inside `if (typeof id === "string")`   | `string`            |
| After the `if`                         | `number`            |

Inside the `if`, `id` can only be a string, so `toUpperCase()` is allowed. The `if` ends with `return`, so the code after it runs only for numbers, and there `id` is a `number`, with `toFixed`.

Following your checks, and giving a variable a smaller type in each part of the code, is called **narrowing**. You saw it once in Session 04, with `unknown`. Swap the two methods, and the errors show the narrowed types

swapped.ts

```typescript
function formatOrderId(id: string | number): string {
  if (typeof id === "string") {
    return id.toPrecision(6);
  }
  return id.toUpperCase();
}
```

```text
swapped.ts:3:15 - error TS2339: Property 'toPrecision' does not exist on type 'string'.

3     return id.toPrecision(6);
                ~~~~~~~~~~~

swapped.ts:5:13 - error TS2339: Property 'toUpperCase' does not exist on type 'number'.

5   return id.toUpperCase();
              ~~~~~~~~~~~


Found 2 errors in the same file, starting at: swapped.ts:3
```

`on type 'string'` and `on type 'number'`, not `on type 'string | number'`: in each part, `tsc` already knows which one it is.

![formatOrderId with the type of id shown at three points: string | number at the parameter, string inside the if, number after the if](images/10-narrowing/narrowing-branches.gif)

---

## typeof Checks

`typeof` gives the same texts as in JavaScript, and `tsc` understands each of them

| Check                         | Narrows to                         |
| ----------------------------- | ---------------------------------- |
| `typeof x === "string"`       | `string`                           |
| `typeof x === "number"`       | `number`                           |
| `typeof x === "boolean"`      | `boolean`                          |
| `typeof x === "bigint"`       | `bigint`                           |
| `typeof x === "symbol"`       | `symbol`                           |
| `typeof x === "undefined"`    | `undefined`                        |
| `typeof x === "function"`     | a function                         |
| `typeof x === "object"`       | an object, an array, **or `null`** |

`!==` works too: after `if (typeof x !== "string") { return; }`, `x` is a `string`.

The last row needs a closer look.

---

## The typeof null Trap

Count the keys of any object

key-count.ts

```typescript
function keyCount(value: unknown): number {
  if (typeof value === "object") {
    return Object.keys(value).length;
  }
  return 0;
}
```

```text
key-count.ts:3:24 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Argument of type 'object | null' is not assignable to parameter of type 'object'.
      Type 'null' is not assignable to type 'object'.

3     return Object.keys(value).length;
                         ~~~~~

  ../node_modules/@typescript/typescript-win32-x64/lib/lib.es5.d.ts:262:5 - The last overload is declared here.
    262     keys(o: object): string[];
            ~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in key-count.ts:3
```

Read the indented lines: inside the `if`, `value` is `object | null`. In JavaScript, `typeof null` is `"object"`, a mistake from the first version of the language that can never be fixed. `tsc` knows it. JavaScript does not warn you

null-crash.js

```javascript
console.log(typeof null);
console.log(Object.keys(null).length);
```

```text
object
file:///D:/ts-practice/10-narrowing/null-crash.js:2
console.log(Object.keys(null).length);
                   ^

TypeError: Cannot convert undefined or null to object
    at Object.keys (<anonymous>)
    at file:///D:/ts-practice/10-narrowing/null-crash.js:2:20
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Add a second check, and `null` is gone

key-count.ts

```typescript
function keyCount(value: unknown): number {
  if (typeof value === "object" && value !== null) {
    return Object.keys(value).length;
  }
  return 0;
}

console.log(keyCount({ id: 1001, customer: "Riya" }));
console.log(keyCount(null));
console.log(keyCount("Riya"));
```

```text
2
0
0
```

`&&` narrows step by step: after the first check `value` is `object | null`, after `value !== null` it is `object`. Remember the pair `typeof x === "object" && x !== null`, you will write it often.

![typeof value === "object" is a door that lets objects, arrays and null through; adding value !== null closes the door for null](images/10-narrowing/typeof-null.gif)

---

## Narrowing Follows Your Code

`tsc` reads your code the way Node.js runs it. A `throw` ends a path just like a `return`

require-text.ts

```typescript
function requireText(value: unknown): string {
  if (typeof value !== "string") {
    throw new Error(`Expected a text, got ${typeof value}`);
  }
  return value.trim();
}

console.log(requireText("  Riya  "));
console.log(requireText(42));
```

```text
Riya
file:///D:/ts-practice/10-narrowing/require-text.ts:3
    throw new Error(`Expected a text, got ${typeof value}`);
          ^

Error: Expected a text, got number
    at requireText (file:///D:/ts-practice/10-narrowing/require-text.ts:3:11)
    at file:///D:/ts-practice/10-narrowing/require-text.ts:9:13
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` reports nothing, and the `throw` is on purpose: only texts get past it, so `value.trim()` is safe. This "check, and stop early" pattern keeps the main code flat, without a big `else`.

An assignment narrows too

assignment.ts

```typescript
let orderId: string | number = "WEB-1002";
console.log(orderId.toLowerCase());

orderId = 1003;
console.log(orderId.toFixed(2));
console.log(orderId.toLowerCase());
```

```text
assignment.ts:6:21 - error TS2339: Property 'toLowerCase' does not exist on type 'number'.

6 console.log(orderId.toLowerCase());
                      ~~~~~~~~~~~


Found 1 error in assignment.ts:6
```

The declared type stays `string | number`, so both assignments are allowed. But after each one, `tsc` knows which of the two the variable holds right now: a `string` on line 2, a `number` from line 4.

![A path through requireText: unknown at the start, the throw branch takes everything that is not a string, and only string reaches value.trim()](images/10-narrowing/control-flow.gif)

---

## Equality Checks

`===` and `!==` narrow literal unions, one value at a time

next-step.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

function nextStep(status: OrderStatus): string {
  if (status === "cancelled") {
    return "Nothing to do";
  }
  if (status === "pending") {
    return "Ask for payment";
  }
  return `Track the order (${status})`;
}

console.log(nextStep("pending"));
console.log(nextStep("shipped"));
```

```text
Ask for payment
Track the order (shipped)
```

After the first `if`, `status` is `"pending" | "paid" | "shipped"`. After the second, `"paid" | "shipped"`. You can see it in an error: copy the second `if` and paste it once more, right after itself

```text
next-step.ts:10:7 - error TS2367: This comparison appears to be unintentional because the types '"paid" | "shipped"' and '"pending"' have no overlap.

10   if (status === "pending") {
         ~~~~~~~~~~~~~~~~~~~~


Found 1 error in next-step.ts:10
```

`"pending"` was already handled above, so it can never match here. This is also how Session 09's `switch` worked: each `case` removes one value, and after the last `case` nothing is left.

![The four OrderStatus values as chips; the check for cancelled removes one chip, the check for pending another, and paid | shipped is left](images/10-narrowing/equality-shrink.gif)

---

## Comparisons with < and > Do Not Narrow

Session 09 left this open: a calculated number does not fit `Rating`. A range check looks like the fix

rating-range.ts

```typescript
type Rating = 1 | 2 | 3 | 4 | 5;

function stars(rating: Rating): string {
  return "*".repeat(rating) + "-".repeat(5 - rating);
}

const average = Math.round(4.4);
if (average >= 1 && average <= 5) {
  console.log(stars(average));
}
```

```text
rating-range.ts:9:21 - error TS2345: Argument of type 'number' is not assignable to parameter of type 'Rating'.

9   console.log(stars(average));
                      ~~~~~~~


Found 1 error in rating-range.ts:9
```

`average >= 1 && average <= 5` is true for `2.5` too, and `2.5` is not a `Rating`. `tsc` narrows with `===`, `typeof` and the other checks in this session, never with `<` or `>`. The fix is your own type guard, later in this session.

---

## Truthiness Checks

An `if` with just a value narrows too. It removes `undefined`, `null`, and the other "falsy" values

coupon-line.ts

```typescript
function couponLine(couponCode: string | undefined): string {
  if (couponCode) {
    return `Coupon: ${couponCode.toUpperCase()}`;
  }
  return "No coupon";
}

console.log(couponLine("festive20"));
console.log(couponLine(undefined));
```

```text
Coupon: FESTIVE20
No coupon
```

Inside the `if`, `couponCode` is a `string`. After it, `tsc` still says `string | undefined`, because the empty text `""` is falsy too and also ends up there.

That is fine for a coupon code, where `""` means "no coupon". For numbers it is a trap

stock-line.ts

```typescript
function stockLine(stock: number | undefined): string {
  if (stock) {
    return `${stock} left`;
  }
  return "Stock unknown";
}

console.log(stockLine(12));
console.log(stockLine(undefined));
console.log(stockLine(0));
```

```text
12 left
Stock unknown
Stock unknown
```

`0` is falsy, so a product that is sold out shows "Stock unknown". `tsc` cannot catch this: the code is type-correct, the logic is wrong. Compare with `undefined` instead

stock-line.ts

```typescript
function stockLine(stock: number | undefined): string {
  if (stock !== undefined) {
    return `${stock} left`;
  }
  return "Stock unknown";
}

console.log(stockLine(12));
console.log(stockLine(undefined));
console.log(stockLine(0));
```

```text
12 left
Stock unknown
0 left
```

Session 11 is all about `undefined` and `null`, with `?.` and `??`.

![Three stock values pass if (stock): 12 goes in, but undefined and 0 both fall to Stock unknown; with stock !== undefined, 0 goes in as 0 left](images/10-narrowing/truthiness-zero.gif)

---

## The in Operator

`"name" in object` is true when the object has that property. When the members of a union have different properties, `in` tells them apart

delivery-in.ts

```typescript
interface PhysicalProduct {
  name: string;
  weightGrams: number;
}

interface DigitalProduct {
  name: string;
  downloadUrl: string;
}

function deliveryNote(item: PhysicalProduct | DigitalProduct): string {
  if ("downloadUrl" in item) {
    return `${item.name}: download from ${item.downloadUrl}`;
  }
  return `${item.name}: ships, ${item.weightGrams} g`;
}

console.log(deliveryNote({ name: "Keyboard", weightGrams: 900 }));
console.log(deliveryNote({ name: "TypeScript Notes", downloadUrl: "https://example.com/ts-notes.pdf" }));
```

```text
Keyboard: ships, 900 g
TypeScript Notes: download from https://example.com/ts-notes.pdf
```

Only `DigitalProduct` has `downloadUrl`, so inside the `if` the item is a `DigitalProduct`, and after it a `PhysicalProduct`.

Watch out for typos in the text. Write `"downloadURL"` with a capital URL

```text
delivery-in.ts:13:48 - error TS2551: Property 'downloadUrl' does not exist on type '(DigitalProduct | PhysicalProduct) & Record<"downloadURL", unknown>'. Did you mean 'downloadURL'?
  Property 'downloadUrl' does not exist on type 'PhysicalProduct & Record<"downloadURL", unknown>'.

13     return `${item.name}: download from ${item.downloadUrl}`;
                                                  ~~~~~~~~~~~

delivery-in.ts:15:39 - error TS2339: Property 'weightGrams' does not exist on type 'DigitalProduct | PhysicalProduct'.
  Property 'weightGrams' does not exist on type 'DigitalProduct'.

15   return `${item.name}: ships, ${item.weightGrams} g`;
                                         ~~~~~~~~~~~


Found 2 errors in the same file, starting at: delivery-in.ts:13
```

The `in` line itself is fine: any object might have a `downloadURL`. The errors come where the properties are used. `Record<"downloadURL", unknown>` means "an object that has a `downloadURL` of unknown type", and nothing was narrowed. Many developers prefer a `kind` property for this reason, shown below.

---

## instanceof

`instanceof` checks which class made an object: `Date`, `Error`, `Map`, and later your own classes (Session 20)

to-date.ts

```typescript
function toDate(value: string | Date): Date {
  if (value instanceof Date) {
    return value;
  }
  return new Date(value);
}

console.log(toDate("2026-10-24").getFullYear());
console.log(toDate(new Date(2026, 0, 15)).getMonth());
```

```text
2026
0
```

The most common use is in `catch`, where the error is `unknown` (Session 04)

error-message.ts

```typescript
function errorMessage(err: unknown): string {
  if (err instanceof Error) {
    return err.message;
  }
  return String(err);
}

try {
  JSON.parse("{ bad json");
} catch (err) {
  console.log(errorMessage(err));
}
```

```text
Expected property name or '}' in JSON at position 2 (line 1 column 3)
```

---

## Array.isArray

`typeof` says `"object"` for arrays too, so it cannot tell an array from other objects. `Array.isArray` can

tag-list.ts

```typescript
function tagList(tags: string | string[]): string[] {
  if (Array.isArray(tags)) {
    return tags;
  }
  return tags.split(",");
}

console.log(tagList("sale,new"));
console.log(tagList(["sale", "electronics"]));
```

```text
[ 'sale', 'new' ]
[ 'sale', 'electronics' ]
```

You will need this in Express: a query string with a repeated name, like `?tag=sale&tag=new`, gives an array, while `?tag=sale` gives a single text (Session 31).

---

## Discriminated Unions

Session 09 introduced `CatalogItem` with a `kind` property. Now it pays off

catalog-item.ts

```typescript
interface PhysicalProduct {
  kind: "physical";
  name: string;
  pricePaise: number;
  weightGrams: number;
}

interface DigitalProduct {
  kind: "digital";
  name: string;
  pricePaise: number;
  downloadUrl: string;
}

type CatalogItem = PhysicalProduct | DigitalProduct;

function weightOf(item: CatalogItem): number {
  if (item.kind === "physical") {
    return item.weightGrams;
  }
  return 0;
}

function deliveryNote(item: CatalogItem): string {
  switch (item.kind) {
    case "physical":
      return `ships, ${item.weightGrams} g`;
    case "digital":
      return `download from ${item.downloadUrl}`;
  }
}

const items: CatalogItem[] = [
  { kind: "physical", name: "Keyboard", pricePaise: 149900, weightGrams: 900 },
  { kind: "digital", name: "TypeScript Notes", pricePaise: 29900, downloadUrl: "https://example.com/ts-notes.pdf" }
];

for (const item of items) {
  console.log(`${item.name}: ${deliveryNote(item)}, weight ${weightOf(item)} g`);
}
```

```text
Keyboard: ships, 900 g, weight 900 g
TypeScript Notes: download from https://example.com/ts-notes.pdf, weight 0 g
```

`item.weightGrams` was TS2339 in Session 09. Now it works: when `item.kind === "physical"`, only `PhysicalProduct` has that `kind`, so the item is a `PhysicalProduct`. This is a **discriminated union**, and it needs three things

| Part                                              | Here                                 |
| ------------------------------------------------- | ------------------------------------ |
| Every member has the same property                | `kind`                               |
| Its type is a different literal in each member    | `"physical"`, `"digital"`            |
| You check that property                           | `item.kind === "physical"`, `switch (item.kind)` |

The property is often called `kind`, `type` or `status`. Inside each `case`, `tsc` knows exactly which member you have. Copy the `physical` line into the `digital` case by mistake

```text
catalog-item.ts:29:29 - error TS2339: Property 'weightGrams' does not exist on type 'DigitalProduct'.

29       return `ships, ${item.weightGrams} g`;
                               ~~~~~~~~~~~


Found 1 error in catalog-item.ts:29
```

Discriminated unions are everywhere in backend code: payment results (`status: "success"` or `"failed"`), events, messages, and API responses.

![Two catalog items go into switch (item.kind): the physical one goes to its case, where weightGrams unlocks; the digital one goes to its case, where downloadUrl unlocks](images/10-narrowing/discriminated-switch.gif)

---

## Exhaustiveness Checking with never

In Session 09, a missing `case` gave TS2366, because the function had to return a string. A function that returns nothing gets no such help

notify.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

function notifyCustomer(status: OrderStatus): void {
  switch (status) {
    case "pending":
      console.log("Reminder: please complete your payment");
      break;
    case "paid":
      console.log("Thanks! We are packing your order");
      break;
    case "shipped":
      console.log("Your order is on the way");
      break;
    case "cancelled":
      console.log("Your order was cancelled");
      break;
    default: {
      const unhandled: never = status;
      throw new Error(`Unknown status: ${unhandled}`);
    }
  }
}

notifyCustomer("paid");
notifyCustomer("shipped");
```

```text
Thanks! We are packing your order
Your order is on the way
```

Look at the `default`. After four cases, no value is left, so there `status` has the type `never` from Session 04: "no value at all". A `never` fits into a `never` variable, and `tsc` is happy.

Now add `| "delivered"` to `OrderStatus`

```text
notify.ts:18:13 - error TS2322: Type '"delivered"' is not assignable to type 'never'.

18       const unhandled: never = status;
               ~~~~~~~~~


Found 1 error in notify.ts:18
```

In the `default`, `status` is now `"delivered"`, which does not fit `never`. The error names the exact value you forgot. Without the `default` block, `tsc` would report nothing at all, and delivered orders would get no message.

The `throw` is for values that get past `tsc`, like a status in JSON from another system: the code stops loudly instead of quietly doing nothing.

Many projects put this check in a small helper

assert-never.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled" | "delivered";

function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${String(value)}`);
}

function statusLabel(status: OrderStatus): string {
  switch (status) {
    case "pending":
      return "Waiting for payment";
    case "paid":
      return "Paid, packing now";
    case "shipped":
      return "On the way";
    case "cancelled":
      return "Cancelled";
    default:
      return assertNever(status);
  }
}
```

```text
assert-never.ts:18:26 - error TS2345: Argument of type '"delivered"' is not assignable to parameter of type 'never'.

18       return assertNever(status);
                            ~~~~~~


Found 1 error in assert-never.ts:18
```

![Each case removes one status; after the last case nothing is left, so status is never in the default. When delivered is added to the union, it reaches the default and does not fit never](images/10-narrowing/exhaustive-never.gif)

---

## Your Own Type Guards

Back to the rating. Write the check as a function, with a special return type

rating-guard.ts

```typescript
type Rating = 1 | 2 | 3 | 4 | 5;

function isRating(value: number): value is Rating {
  return Number.isInteger(value) && value >= 1 && value <= 5;
}

function stars(rating: Rating): string {
  return "*".repeat(rating) + "-".repeat(5 - rating);
}

for (const average of [4.4, 2.6, 7]) {
  const rounded = Math.round(average);
  if (isRating(rounded)) {
    console.log(`${average} -> ${stars(rounded)}`);
  } else {
    console.log(`${average} -> not a rating`);
  }
}
```

```text
4.4 -> ****-
2.6 -> ***--
7 -> not a rating
```

`value is Rating` is a **type predicate**, and a function with one is a **type guard**. The function still returns `true` or `false`, but the return type tells `tsc`: "when this returns `true`, the argument is a `Rating`". So inside the `if`, `rounded` is a `Rating`.

With a plain `boolean` return type, `tsc` learns nothing from the call

rating-bool.ts

```typescript
type Rating = 1 | 2 | 3 | 4 | 5;

function isRating(value: number): boolean {
  return Number.isInteger(value) && value >= 1 && value <= 5;
}

function stars(rating: Rating): string {
  return "*".repeat(rating) + "-".repeat(5 - rating);
}

const rounded = Math.round(4.4);
if (isRating(rounded)) {
  console.log(stars(rounded));
}
```

```text
rating-bool.ts:13:21 - error TS2345: Argument of type 'number' is not assignable to parameter of type 'Rating'.

13   console.log(stars(rounded));
                       ~~~~~~~


Found 1 error in rating-bool.ts:13
```

Be careful: `tsc` **trusts** a type predicate and does not check the body. A wrong check is not an error

rating-lie.ts

```typescript
type Rating = 1 | 2 | 3 | 4 | 5;

function isRating(value: number): value is Rating {
  return value > 0;
}

function stars(rating: Rating): string {
  return "*".repeat(rating) + "-".repeat(5 - rating);
}

const rounded = Math.round(7.2);
if (isRating(rounded)) {
  console.log(stars(rounded));
}
```

```text
file:///D:/ts-practice/10-narrowing/rating-lie.ts:8
  return "*".repeat(rating) + "-".repeat(5 - rating);
                                  ^

RangeError: Invalid count value: -2
    at String.repeat (<anonymous>)
    at stars (file:///D:/ts-practice/10-narrowing/rating-lie.ts:8:35)
    at file:///D:/ts-practice/10-narrowing/rating-lie.ts:13:15
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` had no errors, and `7` went into `stars` as a "Rating". A type guard is a promise you make to `tsc`, so keep it small and test it.

![isRating is a gate between number and Rating: 4 passes and becomes a Rating, 7 is stopped; a lying gate lets 7 through and stars crashes](images/10-narrowing/type-predicate.gif)

---

## tsc Can Write the Type Guard Itself

Since TypeScript 5.5, `tsc` works out simple type guards on its own

parcels.ts

```typescript
interface PhysicalProduct {
  kind: "physical";
  name: string;
  pricePaise: number;
  weightGrams: number;
}

interface DigitalProduct {
  kind: "digital";
  name: string;
  pricePaise: number;
  downloadUrl: string;
}

type CatalogItem = PhysicalProduct | DigitalProduct;

const items: CatalogItem[] = [
  { kind: "physical", name: "Keyboard", pricePaise: 149900, weightGrams: 900 },
  { kind: "digital", name: "TypeScript Notes", pricePaise: 29900, downloadUrl: "https://example.com/ts-notes.pdf" },
  { kind: "physical", name: "Monitor", pricePaise: 1299900, weightGrams: 4200 }
];

const parcels = items.filter((item) => item.kind === "physical");
const totalWeight = parcels.reduce((sum, item) => sum + item.weightGrams, 0);
console.log(`${parcels.length} parcels, ${totalWeight} g`);
```

```text
2 parcels, 5100 g
```

Point at `parcels`: it is `PhysicalProduct[]`, not `CatalogItem[]`, so `item.weightGrams` works in `reduce`. `tsc` saw that the callback checks `kind`, and treated it as a type guard.

| Callback for `filter`                   | Result                |
| --------------------------------------- | --------------------- |
| `(item) => item.kind === "physical"`    | `PhysicalProduct[]`   |
| `(item) => item.name.length > 8`        | `CatalogItem[]`       |

The second one does not narrow anything, so the type stays as it was. The same works for functions: `function isPhysical(item: CatalogItem) { return item.kind === "physical"; }` gets the return type `item is PhysicalProduct` without you writing it. Older TypeScript versions returned `CatalogItem[]` here, so you will still see hand-written type guards in many projects.

---

## Checking the Properties of an Unknown Object

Session 04 ended with the bug from Session 01: `JSON.parse` returned `any`, `"25" + 1` gave `251`, and `tsc` said nothing. With `unknown`, each step needs a check. Try them one by one

user-age.ts

```typescript
const body = '{ "name": "Riya", "age": "25" }';
const user: unknown = JSON.parse(body);

console.log(user.age);
if (typeof user === "object") {
  console.log(user.age);
}
if (typeof user === "object" && user !== null) {
  console.log(user.age);
}
if (typeof user === "object" && user !== null && "age" in user) {
  console.log(user.age + 1);
}
```

```text
user-age.ts:4:13 - error TS18046: 'user' is of type 'unknown'.

4 console.log(user.age);
              ~~~~

user-age.ts:6:15 - error TS18047: 'user' is possibly 'null'.

6   console.log(user.age);
                ~~~~

user-age.ts:6:20 - error TS2339: Property 'age' does not exist on type 'object'.

6   console.log(user.age);
                     ~~~

user-age.ts:9:20 - error TS2339: Property 'age' does not exist on type 'object'.

9   console.log(user.age);
                     ~~~

user-age.ts:12:15 - error TS18046: 'user.age' is of type 'unknown'.

12   console.log(user.age + 1);
                 ~~~~~~~~


Found 5 errors in the same file, starting at: user-age.ts:4
```

| Checks so far                   | `user` is                          | The problem                         |
| ------------------------------- | ---------------------------------- | ----------------------------------- |
| none                            | `unknown`                          | Nothing can be used (TS18046)       |
| `typeof user === "object"`      | `object \| null`                   | Could be `null` (TS18047)           |
| `&& user !== null`              | `object`                           | An object with no known properties (TS2339) |
| `&& "age" in user`              | an object with an `age`            | `user.age` is `unknown` (TS18046)   |

One more check, on the property itself

user-age.ts

```typescript
const body = '{ "name": "Riya", "age": "25" }';
const user: unknown = JSON.parse(body);

if (typeof user === "object" && user !== null && "age" in user && typeof user.age === "number") {
  console.log(user.age + 1);
} else {
  console.log("age is missing or not a number");
}
```

```text
age is missing or not a number
```

The check found the bug: `age` is the text `"25"`, not a number. Instead of a wrong `251`, the code takes the safe path.

For a whole object, put the checks into a type guard

order-body.ts

```typescript
interface OrderBody {
  id: number;
  customer: string;
}

function isOrderBody(value: unknown): value is OrderBody {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    typeof value.id === "number" &&
    "customer" in value &&
    typeof value.customer === "string"
  );
}

const bodies = ['{ "id": 1001, "customer": "Riya" }', '{ "id": "1002", "customer": "Sara" }'];

for (const text of bodies) {
  const body: unknown = JSON.parse(text);
  if (isOrderBody(body)) {
    console.log(`Order ${body.id} for ${body.customer}`);
  } else {
    console.log(`Rejected: ${text}`);
  }
}
```

```text
Order 1001 for Riya
Rejected: { "id": "1002", "customer": "Sara" }
```

This is the safe way to use data from outside: request bodies, files, other APIs. For two properties it is fine. For a real request body with ten properties, nested objects and rules like "at least 1", it gets long, and Session 26 does the same job with Joi.

![unknown behind four locks: typeof object opens the first, !== null the second, "age" in user the third, typeof user.age === "number" the last, and only then can age be used as a number](images/10-narrowing/unknown-locks.gif)

---

## Which Check to Use

| You have                                   | Check                              | Example                         |
| ------------------------------------------ | ---------------------------------- | ------------------------------- |
| `string \| number` and other primitives    | `typeof`                           | `typeof id === "string"`        |
| A literal union                            | `===`, `!==`, `switch`             | `status === "paid"`             |
| A value or `undefined`                     | `!== undefined`                    | `stock !== undefined`           |
| Objects with different properties          | `in`                               | `"downloadUrl" in item`         |
| Objects made by a class                    | `instanceof`                       | `err instanceof Error`          |
| One value or an array                      | `Array.isArray`                    | `Array.isArray(tags)`           |
| Objects with a `kind`                      | `===` or `switch` on the property  | `switch (item.kind)`            |
| A rule `tsc` cannot follow                 | Your own type guard                | `isRating(rounded)`             |
| `unknown` from outside                     | `typeof`, `!== null`, `in`, or Joi | `isOrderBody(body)`             |

![The checks of this session as a toolbox: typeof, ===, !== undefined, in, instanceof, Array.isArray, a kind property and your own type guard, each with the kind of union it splits](images/10-narrowing/checks-toolbox.gif)

---

## Mini Example 1: Shipping a Mixed Cart

Session 08 left this open: go through a cart and treat each kind of product differently

cart.ts

```typescript
interface PhysicalProduct {
  kind: "physical";
  name: string;
  pricePaise: number;
  weightGrams: number;
}

interface DigitalProduct {
  kind: "digital";
  name: string;
  pricePaise: number;
  downloadUrl: string;
}

type CatalogItem = PhysicalProduct | DigitalProduct;

// Rs. 40 for the first 500 g, and Rs. 20 for every 500 g after that
function shippingPaise(item: PhysicalProduct): number {
  const extraSteps = Math.ceil(Math.max(0, item.weightGrams - 500) / 500);
  return 4000 + extraSteps * 2000;
}

function deliveryNote(item: CatalogItem): string {
  switch (item.kind) {
    case "physical":
      return `parcel, ${item.weightGrams} g, shipping Rs. ${shippingPaise(item) / 100}`;
    case "digital":
      return `download: ${item.downloadUrl}`;
    default: {
      const unhandled: never = item;
      throw new Error(`Unknown item: ${JSON.stringify(unhandled)}`);
    }
  }
}

const cart: CatalogItem[] = [
  { kind: "physical", name: "Keyboard", pricePaise: 149900, weightGrams: 900 },
  { kind: "digital", name: "TypeScript Notes", pricePaise: 29900, downloadUrl: "https://example.com/ts-notes.pdf" },
  { kind: "physical", name: "Monitor", pricePaise: 1299900, weightGrams: 4200 }
];

for (const item of cart) {
  console.log(`${item.name.padEnd(17)} ${deliveryNote(item)}`);
}

const parcels = cart.filter((item) => item.kind === "physical");
const shipping = parcels.reduce((sum, item) => sum + shippingPaise(item), 0);
const itemsTotal = cart.reduce((sum, item) => sum + item.pricePaise, 0);

console.log(`Parcels: ${parcels.length}, shipping: Rs. ${shipping / 100}`);
console.log(`Total: Rs. ${((itemsTotal + shipping) / 100).toFixed(2)}`);
```

```bash
npx tsc --noEmit
node cart.ts
```

```text
Keyboard          parcel, 900 g, shipping Rs. 60
TypeScript Notes  download: https://example.com/ts-notes.pdf
Monitor           parcel, 4200 g, shipping Rs. 200
Parcels: 2, shipping: Rs. 260
Total: Rs. 15057.00
```

| Code                                          | What narrows                                  |
| --------------------------------------------- | --------------------------------------------- |
| `case "physical": ... shippingPaise(item)`    | `item` is a `PhysicalProduct`, so it fits     |
| `case "digital": ... item.downloadUrl`        | `item` is a `DigitalProduct`                  |
| `const unhandled: never = item`               | Nothing is left, so a forgotten kind is caught |
| `cart.filter((item) => item.kind === "physical")` | `parcels` is `PhysicalProduct[]`          |
| `item.pricePaise` in the last `reduce`        | No check needed: every member has it          |

Now the shop sells gift cards, which are emailed. Add a third interface after `DigitalProduct`, and change the union

```typescript
interface GiftCard {
  kind: "giftcard";
  name: string;
  pricePaise: number;
  recipientEmail: string;
}

type CatalogItem = PhysicalProduct | DigitalProduct | GiftCard;
```

```text
cart.ts:37:13 - error TS2322: Type 'GiftCard' is not assignable to type 'never'.

37       const unhandled: never = item;
               ~~~~~~~~~


Found 1 error in cart.ts:37
```

`tsc` names the kind that `deliveryNote` does not handle. Add the case after the `digital` one

```typescript
    case "giftcard":
      return `emailed to ${item.recipientEmail}`;
```

And put a gift card in the cart (with a comma after the monitor)

```typescript
  { kind: "giftcard", name: "Gift card", pricePaise: 100000, recipientEmail: "sara@example.com" }
```

```text
Keyboard          parcel, 900 g, shipping Rs. 60
TypeScript Notes  download: https://example.com/ts-notes.pdf
Monitor           parcel, 4200 g, shipping Rs. 200
Gift card         emailed to sara@example.com
Parcels: 2, shipping: Rs. 260
Total: Rs. 16057.00
```

`parcels` and the shipping total needed no change: the filter still keeps only physical products, and `tsc` still knows it.

![Cart items go through switch (item.kind): parcels get shipping, the e-book a download link. A gift card is added to the union, falls through to the default, and TS2322 names GiftCard until its case is added](images/10-narrowing/mixed-cart.gif)

---

## Mini Example 2: The Log Level from a .env File

Session 09's logger had a fixed minimum level. A real app reads it from an environment variable, so you can change it without changing code. `process.env.LOG_LEVEL` is `string | undefined`, and Session 09 showed that a `string` does not fit `LogLevel`. Narrowing fixes that

logger.ts

```typescript
type LogLevel = "debug" | "info" | "warn" | "error";

interface LevelRanks {
  debug: number;
  info: number;
  warn: number;
  error: number;
}

const ranks: LevelRanks = { debug: 10, info: 20, warn: 30, error: 40 };

function isLogLevel(value: string): value is LogLevel {
  return value === "debug" || value === "info" || value === "warn" || value === "error";
}

function readMinimumLevel(): LogLevel {
  const raw = process.env.LOG_LEVEL;
  if (raw === undefined) {
    return "info";
  }
  const level = raw.trim().toLowerCase();
  if (isLogLevel(level)) {
    return level;
  }
  console.log(`Unknown LOG_LEVEL "${raw}", using "info"`);
  return "info";
}

const minimumLevel = readMinimumLevel();

function log(level: LogLevel, message: string): void {
  if (ranks[level] < ranks[minimumLevel]) {
    return;
  }
  console.log(`[${level.toUpperCase().padEnd(5)}] ${message}`);
}

log("debug", "Cart loaded with 3 items");
log("info", "Order 1001 placed by Riya");
log("warn", "Low stock: Mouse (2 left)");
log("error", "Payment failed for order 1002");
```

```bash
npx tsc --noEmit
node logger.ts
```

```text
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
```

No `LOG_LEVEL` is set, so `raw === undefined` and the level is `"info"`. Now make a file `.env` next to `logger.ts`

.env

```text
LOG_LEVEL=DEBUG
```

Node.js reads a `.env` file itself when you start it with `--env-file`, and puts its values into `process.env`

```bash
node --env-file=.env logger.ts
```

```text
[DEBUG] Cart loaded with 3 items
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
```

`"DEBUG"` became `"debug"` with `toLowerCase()`, passed `isLogLevel`, and is now a `LogLevel`. Change `.env` to `LOG_LEVEL=verbose` and run the same command

```text
Unknown LOG_LEVEL "verbose", using "info"
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
```

| Line                                 | Type of the value                      |
| ------------------------------------ | -------------------------------------- |
| `const raw = process.env.LOG_LEVEL;` | `string \| undefined`                  |
| After `if (raw === undefined)`       | `string`                               |
| `const level = raw.trim().toLowerCase();` | `string`                          |
| Inside `if (isLogLevel(level))`      | `LogLevel`                             |

Try the shortcut: replace the four lines from `const level` to the closing `}` of the second `if` with `return raw;`

```text
logger.ts:21:3 - error TS2322: Type 'string' is not assignable to type 'LogLevel'.

21   return raw;
     ~~~~~~


Found 1 error in logger.ts:21
```

Any text could be in `.env`. Without the check, `"verbose"` would become the minimum level, `ranks["verbose"]` would be `undefined`, and every comparison with it would be `false`.

`isLogLevel` could even be written without `: value is LogLevel`: since TypeScript 5.5, `tsc` gives it the return type `value is "debug" | "error" | "info" | "warn"` by itself.

![Three runs of logger.ts: with no .env the level is info; with LOG_LEVEL=DEBUG the text passes isLogLevel and becomes debug; with verbose the guard says no, a warning prints, and info is used](images/10-narrowing/env-level.gif)

---

## Beginner Mistakes

### Mistake 1: Forgetting that typeof null is "object"

After `typeof value === "object"`, the value is `object | null`, and `Object.keys(value)` gives `error TS2769`. Add `&& value !== null`.

---

### Mistake 2: A truthiness check on a number

`if (stock)` treats `0` like `undefined`, and a sold-out product showed "Stock unknown". For numbers, write `if (stock !== undefined)`.

---

### Mistake 3: Expecting < and > to narrow

`if (average >= 1 && average <= 5)` still leaves `average` a `number`, and `stars(average)` gives `error TS2345`. Use a type guard like `isRating`.

---

### Mistake 4: A type guard that returns boolean

`function isRating(value: number): boolean` tells `tsc` nothing. The return type must be a type predicate: `value is Rating`.

---

### Mistake 5: A type guard that lies

`tsc` does not check the body of a type guard. `return value > 0` with `value is Rating` let `7` through, and `stars` crashed with `RangeError: Invalid count value: -2`. Make the check exactly match the type.

---

### Mistake 6: Using includes to check a literal union

`const levels: LogLevel[] = [...]` and `levels.includes(raw)` with a `string` gives `error TS2345: Argument of type 'string' is not assignable to parameter of type 'LogLevel'.`, and `includes` does not narrow anyway. Write a type guard with `===`.

---

### Mistake 7: A typo in an in check

`"downloadURL" in item` is not an error itself, but it narrows nothing, and the errors appear where the properties are used, with `Record<"downloadURL", unknown>` in the message. A `kind` property avoids this.

---

### Mistake 8: A switch without the never check

In a function that returns nothing, a new value in the union gives no error, and the new case silently does nothing. Add `default: { const unhandled: never = value; throw ... }`.

---

### Mistake 9: The wrong property in a case

In `case "digital":`, `item.weightGrams` gives `error TS2339: Property 'weightGrams' does not exist on type 'DigitalProduct'.` Read the type name in the message: it tells you which case you are in.

---

### Mistake 10: Forgetting that an assignment changes the type

After `orderId = 1003`, `orderId.toLowerCase()` gives `error TS2339: Property 'toLowerCase' does not exist on type 'number'.` The variable has the type of the value it holds right now.

---

### Mistake 11: Checking only that JSON is an object

`typeof user === "object" && user !== null` is not enough: `user.age` is still unknown. Check every property you use, as `isOrderBody` does, or use a validation library (Session 26).

---

## Practice Exercises

### Exercise 1

Write `formatPrice(price: number | string): string`. For a number, return `Rs. 1499.00` (with `toFixed(2)`). For a text like `"1499"`, turn it into a number with `Number(price)` first. Point at `price` in both branches.

### Exercise 2

Write `describeValue(value: unknown): string` that returns `text (4 characters)`, `number`, `list of 3 items`, `null`, `object` or `something else`. Does the order of your checks matter? Try putting the `object` check before the array check.

### Exercise 3

Change `stockLine` so that `0` gives `Out of stock`, other numbers `12 left`, and `undefined` gives `Stock unknown`.

### Exercise 4

In `catalog-item.ts`, add `interface ServiceBooking { kind: "service"; name: string; pricePaise: number; date: string }` to the union. Which function shows an error? `weightOf` shows none: what does it return for a booking, and is that right? Handle the new kind in both functions.

### Exercise 5

In `notify.ts`, add `"delivered"` and `"returned"` to `OrderStatus`. Read the error, then add both cases. What does the error say when only one of the two is handled?

### Exercise 6

Write `type PaymentMethod = "card" | "upi" | "cod";` and a type guard `isPaymentMethod(value: string): value is PaymentMethod`. Read `process.argv[2]` (it is `string | undefined`), and print `Paying by upi` or `Unknown payment method`. Run it as `node pay.ts upi` and `node pay.ts cash`.

### Exercise 7

Write `isProduct(value: unknown): value is Product` for `interface Product { id: string; name: string; pricePaise: number }`. Test it with three JSON texts: a correct one, one with `"pricePaise": "1499"`, and one without `name`.

### Exercise 8

In Mini Example 1, after adding gift cards, change the filter to `cart.filter((item) => item.kind !== "digital")`. Read the error on the next line. Why is `parcels` not `PhysicalProduct[]` any more?

---

## Interview Questions

### What is narrowing?

TypeScript follows the checks in your code and gives a variable a smaller type in each part of it. After `typeof id === "string"`, `id` of type `string | number` is a `string` inside the `if`, and a `number` after an `if` that returns.

### Which checks narrow a type?

`typeof`, `===` and `!==`, truthiness (`if (value)`), `in`, `instanceof`, `Array.isArray`, checks on a discriminant property, type predicates, and assignments. `<` and `>` do not narrow.

### Why does typeof value === "object" not remove null?

Because `typeof null` is `"object"` in JavaScript. TypeScript narrows to `object | null`, so you add `value !== null`.

### What is a discriminated union?

A union of object types that share a property with a different literal type in each member, like `kind: "physical"` and `kind: "digital"`. Checking that property, with `===` or `switch`, narrows the object to one member.

### How do you make sure a switch handles every member of a union?

Add a `default` that assigns the value to a variable of type `never`, or passes it to an `assertNever(value: never)` function. If a member is not handled, it reaches the `default` and does not fit `never`, so `tsc` reports it by name.

### What is a type predicate?

A return type like `value is Rating` on a function that returns a boolean. When the function returns `true`, TypeScript narrows the argument to that type. Such a function is called a user-defined type guard.

### Does TypeScript check that a type predicate is correct?

No. It trusts the predicate. A wrong check lets wrong values through without any error, so type guards should be small and tested.

### What are inferred type predicates?

Since TypeScript 5.5, a function or callback whose body is a narrowing check gets a type predicate automatically. `items.filter((item) => item.kind === "physical")` returns `PhysicalProduct[]` instead of the original union array.

### What is the difference between in and a discriminant check?

`in` checks whether an object has a property, so it works for unions without a shared property. A discriminant check compares a shared literal property like `kind`. It is clearer, works with `switch`, and a typo in the value is an error, while a typo in an `in` text is not.

### How do you safely use data from JSON.parse?

Treat it as `unknown`, then check it before use: `typeof x === "object"`, `x !== null`, `"prop" in x`, and the type of each property, usually inside a type guard. For bigger objects, use a validation library like Joi.

### Why can a truthiness check be a bug?

It removes every falsy value, not only `undefined` and `null`: also `0`, `""`, `false` and `NaN`. A stock of `0` or an empty name would be treated as missing. Compare with `undefined` or `null` instead.

### Why does narrowing not work with < and >?

A range like "between 1 and 5" includes values such as `2.5`, which are not in a literal union like `1 | 2 | 3 | 4 | 5`. TypeScript only narrows with checks that pin down exact types or values, so for ranges you write a type guard.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [09. Union and Literal Types](09-union-and-literal-types.md) | [Course outline](README.md#course-outline) | [11. null, undefined and Optional Values](11-null-undefined-and-optional-values.md) |
