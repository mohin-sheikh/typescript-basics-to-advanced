# keyof, typeof and Indexed Access

> **Session 15 of 33** | Part 4: Generics and Type Tools | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Two Kinds of typeof](#two-kinds-of-typeof)
- [typeof: The Type of a Value](#typeof-the-type-of-a-value)
- [keyof: The Keys of a Type](#keyof-the-keys-of-a-type)
- [keyof typeof: The Keys of an Object You Wrote](#keyof-typeof-the-keys-of-an-object-you-wrote)
- [Indexed Access: A Type from Inside a Type](#indexed-access-a-type-from-inside-a-type)
- [A Union from an Array](#a-union-from-an-array)
- [keyof in Generics: K extends keyof T](#keyof-in-generics-k-extends-keyof-t)
- [keyof and Index Signatures](#keyof-and-index-signatures)
- [Object.keys, Revisited](#objectkeys-revisited)
- [Mini Example 1: A Product Report](#mini-example-1-a-product-report)
- [Mini Example 2: The Logger with One Source of Truth](#mini-example-2-the-logger-with-one-source-of-truth)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `15-keyof-typeof` inside `ts-practice`, with the one-line `tsconfig.json`

15-keyof-typeof/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## Two Kinds of typeof

Until now, every type was written by hand: interfaces, unions, aliases. This session builds types **from** other things: from values, and from other types. The first tool has a familiar name

typeof-two.ts

```typescript
const defaults = { port: 3000, host: "localhost" };

console.log(typeof defaults);

type Defaults = typeof defaults;

const testing: Defaults = { port: 0, host: "127.0.0.1" };
console.log(testing);
```

```bash
npx tsc --noEmit
node typeof-two.ts
```

```text
object
{ port: 0, host: '127.0.0.1' }
```

The same word does two different jobs

| Where                           | Example                         | Gives                                          |
| ------------------------------- | ------------------------------- | ---------------------------------------------- |
| In code (JavaScript)            | `console.log(typeof defaults)`  | A text at run time: `"object"`                 |
| In a type (TypeScript)          | `type Defaults = typeof defaults` | The type `tsc` inferred: `{ port: number; host: string; }` |

The JavaScript `typeof` runs in Node.js and you used it for narrowing in Session 10. The TypeScript `typeof` exists only for `tsc`, and disappears like every type.

![The word typeof used twice: in code it runs and prints the text "object"; in a type it asks tsc for the inferred type { port: number; host: string }](images/15-keyof-typeof-indexed-access/two-typeofs.gif)

---

## typeof: The Type of a Value

`typeof` in a type copies whatever `tsc` knows about a variable. The type is checked like any other

typeof-wrong.ts

```typescript
const defaults = { port: 3000, host: "localhost" };
type Defaults = typeof defaults;

const broken: Defaults = { port: "3000", host: "localhost" };
```

```text
typeof-wrong.ts:4:28 - error TS2322: Type 'string' is not assignable to type 'number'.

4 const broken: Defaults = { port: "3000", host: "localhost" };
                             ~~~~

  typeof-wrong.ts:1:20 - The expected type comes from property 'port' which is declared here on type '{ port: number; host: string; }'
    1 const defaults = { port: 3000, host: "localhost" };
                         ~~~~~~~~~~


Found 1 error in typeof-wrong.ts:4
```

The extra line points into the **object** itself: the value is where the type came from.

| Value                                             | `typeof` gives                                       |
| ------------------------------------------------- | ---------------------------------------------------- |
| `{ port: 3000, host: "localhost" }`               | `{ port: number; host: string; }`                    |
| `{ port: 3000, host: "localhost" } as const`      | `{ readonly port: 3000; readonly host: "localhost"; }` |
| `function formatRupees(paise: number): string`    | `(paise: number) => string`                          |

When is this useful? When the value comes first, and you would otherwise write the same shape twice: default settings, a lookup table, a list of allowed values. For data like orders and customers, an interface is still the better start (Session 08).

---

## keyof: The Keys of a Type

`keyof` takes an object type and gives a union of its property names

keyof.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

type ProductKey = keyof Product;

const sortKey: ProductKey = "pricePaise";
const wrongKey: ProductKey = "price";
export { sortKey, wrongKey };
```

```text
keyof.ts:10:7 - error TS2322: Type '"price"' is not assignable to type 'keyof Product'.

10 const wrongKey: ProductKey = "price";
         ~~~~~~~~


Found 1 error in keyof.ts:10
```

`keyof Product` is `"id" | "name" | "pricePaise"`, a literal union like in Session 09. When a property is added to `Product`, `keyof Product` grows with it.

![The interface Product with three properties: keyof pulls out the three names and joins them into the union "id" | "name" | "pricePaise"](images/15-keyof-typeof-indexed-access/keyof-keys.gif)

---

## keyof typeof: The Keys of an Object You Wrote

Sessions 07, 09 and 13 all ran into the same problem: the keys of `prices` had to be typed again as `"keyboard" | "mouse"`. Combine the two tools

price-of.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };

type ProductName = keyof typeof prices;

function priceOf(product: ProductName): number {
  return prices[product];
}

console.log(priceOf("keyboard"));
console.log(priceOf("mouse"));
```

```text
149900
49900
```

Read it from the inside out: `typeof prices` is `{ keyboard: number; mouse: number; }`, and `keyof` of that is `"keyboard" | "mouse"`. A product that is not in the object is refused

```typescript
console.log(priceOf("monitor"));
```

```text
price-of.ts:9:21 - error TS2345: Argument of type '"monitor"' is not assignable to parameter of type '"keyboard" | "mouse"'.

9 console.log(priceOf("monitor"));
                      ~~~~~~~~~


Found 1 error in price-of.ts:9
```

Now add the monitor to the object, and only there

price-of.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900, monitor: 1299900 };

type ProductName = keyof typeof prices;

function priceOf(product: ProductName): number {
  return prices[product];
}

console.log(priceOf("monitor"));
```

```text
1299900
```

`ProductName` grew by itself. The object is the **single source of truth**: one place to change, and the type follows.

Both words are needed. `keyof` works on types, and `prices` is a value

no-typeof.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };
type ProductName = keyof prices;
```

```text
no-typeof.ts:2:26 - error TS2749: 'prices' refers to a value, but is being used as a type here. Did you mean 'typeof prices'?

2 type ProductName = keyof prices;
                           ~~~~~~


Found 1 error in no-typeof.ts:2
```

![The prices object, a value, goes through typeof and becomes a type; keyof turns the type into "keyboard" | "mouse". A monitor is added to the object, and the union grows by itself](images/15-keyof-typeof-indexed-access/keyof-typeof.gif)

---

## Indexed Access: A Type from Inside a Type

Square brackets, used on a **type**, give the type of a property

indexed.ts

```typescript
interface Order {
  id: number;
  customer: { name: string; city: string };
  items: { name: string; pricePaise: number; quantity: number }[];
  status: "pending" | "paid" | "shipped";
}

type OrderStatus = Order["status"];
type Customer = Order["customer"];
type OrderItem = Order["items"][number];

function lineTotal(item: OrderItem): number {
  return item.pricePaise * item.quantity;
}

const status: OrderStatus = "paid";
const riya: Customer = { name: "Riya", city: "Pune" };
console.log(status, riya.city, lineTotal({ name: "Mouse", pricePaise: 49900, quantity: 2 }));
```

```text
paid Pune 99800
```

| Type                         | Gives                                                  |
| ---------------------------- | ------------------------------------------------------ |
| `Order["status"]`            | `"pending" \| "paid" \| "shipped"`                     |
| `Order["customer"]`          | `{ name: string; city: string; }`                      |
| `Order["items"]`             | `{ name: string; pricePaise: number; quantity: number; }[]` |
| `Order["items"][number]`     | `{ name: string; pricePaise: number; quantity: number; }`, one item |
| `Order["id" \| "status"]`    | `number \| "pending" \| "paid" \| "shipped"`           |

`[number]` means "the type you get when you index the array with any number": the item type. A union of keys gives a union of their types.

This is how you reuse a piece of someone else's type without copying it: when `Order` changes, `OrderItem` changes with it. Only real property names are allowed

indexed-wrong.ts

```typescript
interface Order {
  id: number;
  status: "pending" | "paid" | "shipped";
}

type Total = Order["total"];
const status: Order["status"] = "delivered";
```

```text
indexed-wrong.ts:6:20 - error TS2339: Property 'total' does not exist on type 'Order'.

6 type Total = Order["total"];
                     ~~~~~~~

indexed-wrong.ts:7:7 - error TS2322: Type '"delivered"' is not assignable to type '"paid" | "pending" | "shipped"'.

7 const status: Order["status"] = "delivered";
        ~~~~~~


Found 2 errors in the same file, starting at: indexed-wrong.ts:6
```

Type positions always use the bracket form with quotes, `Order["status"]`. `Order.status` works only on values.

![The Order type with arrows pulling pieces out: Order["status"] gives the status union, Order["customer"] the customer object, Order["items"][number] one item of the array](images/15-keyof-typeof-indexed-access/indexed-access.gif)

---

## A Union from an Array

Session 13 made a readonly tuple with `as const`. With `typeof` and `[number]`, the tuple gives a union

levels.ts

```typescript
const LOG_LEVELS = ["debug", "info", "warn", "error"] as const;

type LogLevel = (typeof LOG_LEVELS)[number];

function isLogLevel(value: string): value is LogLevel {
  return (LOG_LEVELS as readonly string[]).includes(value);
}

for (const text of ["warn", "verbose"]) {
  console.log(`${text}: ${isLogLevel(text) ? "a log level" : "not a log level"}`);
}

for (const level of LOG_LEVELS) {
  console.log(level.toUpperCase());
}
```

```text
warn: a log level
verbose: not a log level
DEBUG
INFO
WARN
ERROR
```

| Step                               | Type                                              |
| ---------------------------------- | ------------------------------------------------- |
| `typeof LOG_LEVELS`                | `readonly ["debug", "info", "warn", "error"]`     |
| `(typeof LOG_LEVELS)[number]`      | `"debug" \| "info" \| "warn" \| "error"`          |

One array is now both: a list you can loop over **at run time**, and a union type **for `tsc`**. Without `as const`, the array is a `string[]`, and `(typeof LOG_LEVELS)[number]` is just `string`.

The `as readonly string[]` in `isLogLevel` solves a problem from Session 10. Written plainly, `includes` refuses a `string`

levels-includes.ts

```typescript
const LOG_LEVELS = ["debug", "info", "warn", "error"] as const;

type LogLevel = (typeof LOG_LEVELS)[number];

function isLogLevel(value: string): value is LogLevel {
  return LOG_LEVELS.includes(value);
}
```

```text
levels-includes.ts:6:30 - error TS2345: Argument of type 'string' is not assignable to parameter of type '"debug" | "error" | "info" | "warn"'.

6   return LOG_LEVELS.includes(value);
                               ~~~~~


Found 1 error in levels-includes.ts:6
```

`includes` on a tuple of four literals only accepts those four literals, which is exactly what we want to find out. Widening the array to `readonly string[]` is a safe assertion (Session 13): every one of the four is a string. This exact pattern, an `as const` array with a type guard, is very common in real code.

![LOG_LEVELS as const is a readonly tuple of four texts; typeof gives its type, and [number] turns the four positions into one union](images/15-keyof-typeof-indexed-access/array-union.gif)

---

## keyof in Generics: K extends keyof T

Session 14's generics and this session's tools meet here. Take one property from every item in a list

pluck.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

const products: Product[] = [
  { id: 1, name: "Keyboard", pricePaise: 149900 },
  { id: 2, name: "Mouse", pricePaise: 49900 }
];

function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}

const names = pluck(products, "name");
const prices = pluck(products, "pricePaise");

console.log(names.join(", "));
console.log(prices.reduce((sum, price) => sum + price, 0));
```

```text
Keyboard, Mouse
199800
```

| Part               | Meaning                                                      |
| ------------------ | ------------------------------------------------------------ |
| `T`                | The item type: `Product`, from `products`                    |
| `K extends keyof T` | The key must be one of the item's property names            |
| `T[K]`             | The type of that property: indexed access with a type parameter |
| `T[K][]`           | An array of those                                            |

For `pluck(products, "name")`, `K` is `"name"`, so the result is `Product["name"][]`, which is `string[]`. For `"pricePaise"`, it is `number[]`. Both mistakes are caught

```typescript
const prices = pluck(products, "price");
const names = pluck(products, "name");
console.log(names.toUpperCase());
```

```text
pluck.ts:16:32 - error TS2345: Argument of type '"price"' is not assignable to parameter of type 'keyof Product'.

16 const prices = pluck(products, "price");
                                  ~~~~~~~

pluck.ts:18:19 - error TS2339: Property 'toUpperCase' does not exist on type 'string[]'.

18 console.log(names.toUpperCase());
                     ~~~~~~~~~~~


Found 2 errors in the same file, starting at: pluck.ts:16
```

Why the second type parameter? With `key: keyof T` instead, `tsc` knows the key is **one of** the keys, but not which one

pluck-plain.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

function pluck<T>(items: readonly T[], key: keyof T): T[keyof T][] {
  return items.map((item) => item[key]);
}

declare const products: Product[];
const names = pluck(products, "name");
console.log(names[0]?.toUpperCase());
```

```text
pluck-plain.ts:13:23 - error TS2339: Property 'toUpperCase' does not exist on type 'string | number'.
  Property 'toUpperCase' does not exist on type 'number'.

13 console.log(names[0]?.toUpperCase());
                         ~~~~~~~~~~~


Found 1 error in pluck-plain.ts:13
```

(`declare const` tells `tsc` that a variable exists without giving it a value: handy for testing types without real data. Do not run this file.)

The result became `T[keyof T]`, the types of **all** properties: `string | number`. `K extends keyof T` remembers the exact key, so `T[K]` is the exact type. This pair, `T` and `K extends keyof T`, appears in many libraries.

![With key: keyof T, the key "name" goes in but tsc only remembers "one of the keys", so the result is string | number; with K extends keyof T, K remembers "name" and the result is string](images/15-keyof-typeof-indexed-access/keyof-vs-k.gif)

![pluck(products, "name"): T becomes Product, K becomes "name", and T[K] looks up Product["name"], which is string, so the result is string[]](images/15-keyof-typeof-indexed-access/pluck-generic.gif)

---

## keyof and Index Signatures

For an object with an index signature (Session 07), `keyof` gives the key type, not a list of names

| Type                                      | `keyof` gives        |
| ----------------------------------------- | -------------------- |
| `interface Stock { [productId: string]: number }` | `string \| number` |

Why `number` too? In JavaScript, `stock[1]` is the same as `stock["1"]`: number keys are turned into texts. So a string index signature accepts numbers as keys as well.

---

## Object.keys, Revisited

Session 13 wrote the keys of `prices` by hand in an assertion. With `keyof typeof`, the assertion follows the object

keys.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };

for (const product of Object.keys(prices) as (keyof typeof prices)[]) {
  console.log(`${product}: ${prices[product]}`);
}
```

```text
keyboard: 149900
mouse: 49900
```

The reason for the assertion has not changed: `Object.keys` returns `string[]`, because at run time an object may have more keys than its type. For an object made right here, the assertion is honest.

---

## Mini Example 1: A Product Report

A small report that can sort products by any property and print any columns, with every property name checked

report.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
}

const products: Product[] = [
  { id: 1, name: "Keyboard", pricePaise: 149900, stock: 12 },
  { id: 2, name: "Mouse", pricePaise: 49900, stock: 0 },
  { id: 3, name: "Monitor", pricePaise: 1299900, stock: 4 }
];

function sortBy<T, K extends keyof T>(items: readonly T[], key: K): T[] {
  return [...items].sort((a, b) => (a[key] < b[key] ? -1 : a[key] > b[key] ? 1 : 0));
}

function printTable<T>(rows: readonly T[], columns: readonly (keyof T)[]): void {
  console.log(columns.map((column) => String(column).padEnd(12)).join("").trimEnd());
  for (const row of rows) {
    console.log(columns.map((column) => String(row[column]).padEnd(12)).join("").trimEnd());
  }
}

printTable(sortBy(products, "pricePaise"), ["name", "pricePaise"]);
console.log("");
printTable(sortBy(products, "stock"), ["name", "stock", "id"]);
```

```bash
npx tsc --noEmit
node report.ts
```

```text
name        pricePaise
Mouse       49900
Keyboard    149900
Monitor     1299900

name        stock       id
Mouse       0           2
Monitor     4           3
Keyboard    12          1
```

| Code                                                  | From this session                                |
| ----------------------------------------------------- | ------------------------------------------------ |
| `sortBy<T, K extends keyof T>(items, key: K)`         | The key must be a property of the items          |
| `a[key] < b[key]`                                     | Reading a property with a type parameter key     |
| `columns: readonly (keyof T)[]`                       | An array of property names                       |
| `String(column)`                                      | `keyof T` may be a number or a symbol (below)    |
| `sortBy(products, "pricePaise")`                      | `T` is `Product`, `K` is `"pricePaise"`          |

Property names that do not exist never reach Node.js

```typescript
printTable(products, ["name", "price"]);
sortBy(products, "colour");
```

```text
report.ts:28:31 - error TS2322: Type '"price"' is not assignable to type 'keyof Product'.

28 printTable(products, ["name", "price"]);
                                 ~~~~~~~

report.ts:29:18 - error TS2345: Argument of type '"colour"' is not assignable to parameter of type 'keyof Product'.

29 sortBy(products, "colour");
                    ~~~~~~~~


Found 2 errors in the same file, starting at: report.ts:28
```

In JavaScript, `sortBy(products, "colour")` would compare `undefined` with `undefined` and return the list unsorted, without a word.

Why `String(column)`? Inside a generic function, `tsc` does not know what `T` is, so `keyof T` could be any kind of key. Try `column.padEnd(12)`

```text
report.ts:19:46 - error TS2339: Property 'padEnd' does not exist on type 'string | number | symbol'.
  Property 'padEnd' does not exist on type 'number'.

19   console.log(columns.map((column) => column.padEnd(12)).join("").trimEnd());
                                                ~~~~~~


Found 1 error in report.ts:19
```

Object keys in JavaScript can be texts, numbers (turned into texts) or symbols. `String(column)` handles all three.

![The product report: sortBy reorders the rows by pricePaise, then by stock, and printTable shows only the chosen columns; a column name that does not exist is stopped with keyof Product](images/15-keyof-typeof-indexed-access/product-report.gif)

---

## Mini Example 2: The Logger with One Source of Truth

The logger from Sessions 09 and 10 had its levels in three places: the `LogLevel` union, the `LevelRanks` interface, and the `ranks` object. Adding a level meant changing all three. Now the object is the only place

logger.ts

```typescript
const LEVEL_RANKS = { debug: 10, info: 20, warn: 30, error: 40 } as const;

type LogLevel = keyof typeof LEVEL_RANKS;

function isLogLevel(value: string): value is LogLevel {
  return Object.hasOwn(LEVEL_RANKS, value);
}

function readMinimumLevel(): LogLevel {
  const raw = process.env.LOG_LEVEL?.trim().toLowerCase();
  if (raw === undefined) {
    return "info";
  }
  if (isLogLevel(raw)) {
    return raw;
  }
  console.log(`Unknown LOG_LEVEL "${raw}", using "info"`);
  return "info";
}

const minimumLevel = readMinimumLevel();

function log(level: LogLevel, message: string): void {
  if (LEVEL_RANKS[level] < LEVEL_RANKS[minimumLevel]) {
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

With a `.env` file containing `LOG_LEVEL=DEBUG` (Session 10)

```bash
node --env-file=.env logger.ts
```

```text
[DEBUG] Cart loaded with 3 items
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
```

| Before (Sessions 09 and 10)                          | Now                                                   |
| ---------------------------------------------------- | ----------------------------------------------------- |
| `type LogLevel = "debug" \| "info" \| "warn" \| "error"` | `type LogLevel = keyof typeof LEVEL_RANKS`        |
| `interface LevelRanks { debug: number; ... }`        | Not needed: `typeof LEVEL_RANKS`                      |
| `const ranks: LevelRanks = { ... }`                  | `const LEVEL_RANKS = { ... } as const`                |
| `isLogLevel` with four `===` checks                  | `Object.hasOwn(LEVEL_RANKS, value)`                   |

Now add a level, in one place only: change the first line to `{ debug: 10, info: 20, warn: 30, error: 40, fatal: 50 }`, and add a call at the end

```typescript
log("fatal", "Database connection lost");
```

```text
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
[FATAL] Database connection lost
```

`LogLevel`, `isLogLevel` and the ranks all know about `fatal`, and `LOG_LEVEL=fatal` in `.env` works too. Typos are still caught, from the same source

```text
logger.ts:34:5 - error TS2345: Argument of type '"warning"' is not assignable to parameter of type '"debug" | "error" | "info" | "warn"'.

34 log("warning", "Disk almost full");
       ~~~~~~~~~


Found 1 error in logger.ts:34
```

Why `Object.hasOwn` and not `value in LEVEL_RANKS`? `in` also finds properties that every object inherits from JavaScript, like `constructor`. With `return value in LEVEL_RANKS;` and `LOG_LEVEL=constructor` in `.env`

```text
[DEBUG] Cart loaded with 3 items
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
```

`"constructor"` passed the type guard, so `tsc` treated it as a `LogLevel` (Session 10: type guards are trusted). Its "rank" is a function, every comparison with it is `false`, and the debug messages appear. With `Object.hasOwn`, the same `.env` gives

```text
Unknown LOG_LEVEL "constructor", using "info"
[INFO ] Order 1001 placed by Riya
[WARN ] Low stock: Mouse (2 left)
[ERROR] Payment failed for order 1002
```

![The text constructor meets two checks: "constructor" in LEVEL_RANKS says yes because every object inherits constructor; Object.hasOwn(LEVEL_RANKS, "constructor") says no, and only the four real levels pass](images/15-keyof-typeof-indexed-access/in-vs-hasown.gif)

![One object LEVEL_RANKS feeds three things: the LogLevel type, the isLogLevel guard and the ranks used by log. fatal: 50 is added to the object, and all three update together](images/15-keyof-typeof-indexed-access/single-source.gif)

---

## Beginner Mistakes

### Mistake 1: keyof on a value

`keyof prices` gives `error TS2749: 'prices' refers to a value, but is being used as a type here. Did you mean 'typeof prices'?` Write `keyof typeof prices`.

---

### Mistake 2: Mixing up the two typeofs

`typeof defaults` in code is the text `"object"` at run time. In a type, `typeof defaults` is the inferred type. They share a name, not a job.

---

### Mistake 3: Forgetting as const before typeof ...[number]

Without `as const`, `["debug", "info"]` is a `string[]`, and `(typeof LOG_LEVELS)[number]` is just `string`. Add `as const` to keep the literals.

---

### Mistake 4: Calling includes with a string on an as const array

`LOG_LEVELS.includes(value)` with a `string` gives `error TS2345`. Widen the array for the check: `(LOG_LEVELS as readonly string[]).includes(value)`.

---

### Mistake 5: Using key: keyof T instead of K extends keyof T

`pluck(products, "name")` returned `(string | number)[]` with `key: keyof T`. A separate `K extends keyof T` keeps the exact key, and `T[K]` the exact type.

---

### Mistake 6: Using the dot on a type

`Order.status` in a type position is not allowed. Indexed access uses brackets and quotes: `Order["status"]`.

---

### Mistake 7: Expecting keyof T to be only strings

Inside a generic function, `column.padEnd(12)` gave `error TS2339 ... on type 'string | number | symbol'`. Use `String(column)`.

---

### Mistake 8: Checking membership with in

`"constructor" in LEVEL_RANKS` is `true`, because every object inherits `constructor`. Use `Object.hasOwn(object, key)` to check an object's own keys.

---

### Mistake 9: Building every type from values

`typeof` is great for settings and lookup tables you write once. For data that comes and goes, like orders and customers, an interface is clearer and documents the shape on its own.

---

### Mistake 10: Copying part of a type by hand

Writing `{ name: string; pricePaise: number; quantity: number }` again for one order item drifts apart from `Order` over time. Use `Order["items"][number]`.

---

## Practice Exercises

### Exercise 1

Write `const theme = { primary: "#3178c6", fontSize: 16, darkMode: false };` and `type Theme = typeof theme`. Create a second theme of type `Theme` with a wrong type for `fontSize`, and read the error. Where does the extra line point?

### Exercise 2

Write `const shippingRates = { standard: 4000, express: 9900, sameDay: 19900 };` and `shippingCost(speed: keyof typeof shippingRates): number`. Add a new speed to the object only, and call `shippingCost` with it.

### Exercise 3

From `interface ApiResponse { status: number; data: { orders: { id: number; total: number }[]; page: number } }`, use indexed access to write the type of one order, without writing `{ id: number; total: number }` again.

### Exercise 4

Write `const PAYMENT_METHODS = ["card", "upi", "cod"] as const;`, the type `PaymentMethod` from it, and a type guard `isPaymentMethod`. Print all methods in a loop, then check `"upi"` and `"cash"`.

### Exercise 5

Write `getProperty<T, K extends keyof T>(item: T, key: K): T[K]`. Call it with a product and `"pricePaise"`, and point at the result. Then call it with `"weight"`.

### Exercise 6

Write `updateField<T, K extends keyof T>(item: T, key: K, value: T[K]): T` that returns a copy with one property changed. Why does `updateField(product, "pricePaise", "free")` fail?

### Exercise 7

In Mini Example 1, add a `direction: "asc" | "desc" = "asc"` parameter to `sortBy`, and print the products from the most expensive to the cheapest.

### Exercise 8

In Mini Example 2, add a second object `LEVEL_LABELS = { debug: "DEBUG", info: "INFO", ... } as const` and use it in `log`. Then add `fatal` to `LEVEL_RANKS` only. Does `tsc` notice that `LEVEL_LABELS` has no `fatal`? (Hint: try `LEVEL_LABELS[level]` in `log`.)

---

## Interview Questions

### What does keyof do?

It takes an object type and gives a union of its property names: `keyof Product` is `"id" | "name" | "pricePaise"`. For a type with a string index signature, it gives `string | number`.

### What is the difference between JavaScript typeof and TypeScript typeof?

JavaScript `typeof` runs at run time and returns a text like `"object"` or `"string"`. TypeScript `typeof`, used in a type position, gives the type `tsc` inferred for a variable, and exists only at compile time.

### What does keyof typeof do?

It gives the keys of an object **value** as a union: `typeof` turns the value into its type, and `keyof` takes that type's keys. It lets an object be the single source of truth for a set of allowed keys.

### What is an indexed access type?

A type written with brackets, like `Order["status"]`, that gives the type of a property. `Order["items"][number]` gives the item type of an array property, and a union of keys gives a union of types.

### How do you get a union type from an array of strings?

Make the array a readonly tuple with `as const`, then write `(typeof ARRAY)[number]`. Without `as const`, the result is just `string`.

### Why does includes refuse a string on an as const array?

Its parameter type is the union of the array's literals, so a general `string` does not fit. Widening the array with `as readonly string[]` for the check is safe.

### What does K extends keyof T mean?

`K` is a type parameter that must be one of the property names of `T`. Together with `T[K]`, it lets a function know the exact type of the property it works with, like in `pluck` or `getProperty`.

### What is the difference between key: keyof T and K extends keyof T?

`key: keyof T` accepts any key, but forgets which one, so `T[keyof T]` is the union of all property types. `K extends keyof T` remembers the exact key, so `T[K]` is that property's type.

### Why can keyof T contain number and symbol?

JavaScript object keys can be strings, numbers (stored as strings) or symbols. In a generic function where `T` is unknown, `keyof T` may be any of them, so string methods need `String(key)`.

### Why use Object.hasOwn instead of in to check a key?

`in` also returns `true` for inherited properties like `constructor` and `toString`. `Object.hasOwn` only checks the object's own properties, which is what a lookup table needs.

### What does "single source of truth" mean here?

The allowed values live in one place, usually an object or an `as const` array, and the types are built from it with `typeof`, `keyof` and indexed access. Adding a value in that one place updates every type.

### When should you write an interface instead of deriving a type with typeof?

For data shapes like orders, customers and request bodies, an interface documents the shape on its own and is easier to read. Deriving with `typeof` fits values written once in the code, like settings, lookup tables and lists of allowed values.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [14. Generics](14-generics.md) | [Course outline](README.md#course-outline) | [16. Enums and Constant Objects](16-enums-and-constant-objects.md) |
