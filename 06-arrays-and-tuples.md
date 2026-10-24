# Arrays and Tuples

> **Session 6 of 33** | Part 2: Typing Your Data | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Typing an Array](#typing-an-array)
- [Two Ways to Write an Array Type](#two-ways-to-write-an-array-type)
- [TypeScript Infers Array Types](#typescript-infers-array-types)
- [Empty Arrays Need a Type](#empty-arrays-need-a-type)
- [Arrays of Objects](#arrays-of-objects)
- [Array Methods Know the Types](#array-methods-know-the-types)
- [Reading an Item Can Give undefined](#reading-an-item-can-give-undefined)
- [Safe Ways to Read Items](#safe-ways-to-read-items)
- [readonly Arrays](#readonly-arrays)
- [readonly Parameters Protect the Caller](#readonly-parameters-protect-the-caller)
- [const Is Not readonly](#const-is-not-readonly)
- [Arrays of Arrays](#arrays-of-arrays)
- [Tuples: A Fixed Type in Each Position](#tuples-a-fixed-type-in-each-position)
- [An Array Literal Is Not a Tuple](#an-array-literal-is-not-a-tuple)
- [Labels, Optional Items and Rest Items](#labels-optional-items-and-rest-items)
- [Tuples Can Still Grow](#tuples-can-still-grow)
- [Returning Two Values with a Tuple](#returning-two-values-with-a-tuple)
- [Tuples You Already Use](#tuples-you-already-use)
- [Array or Tuple?](#array-or-tuple)
- [Mini Example 1: A Command-Line Cart Total](#mini-example-1-a-command-line-cart-total)
- [Mini Example 2: Reading Order Lines from CSV](#mini-example-2-reading-order-lines-from-csv)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `06-arrays-tuples` inside `ts-practice`, with the one-line `tsconfig.json`

06-arrays-tuples/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

You know arrays from JavaScript ([JavaScript course, Session 12](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/12-arrays-and-array-methods.md)). This session adds the types

| Type                  | Means                                                   |
| --------------------- | ------------------------------------------------------- |
| `number[]`            | A list of numbers, any length                           |
| `readonly number[]`   | A list of numbers that cannot be changed                |
| `[string, number]`    | A **tuple**: exactly a text, then a number              |

---

## Typing an Array

Write the type of the items, then `[]`

array-types.ts

```typescript
const productNames: string[] = ["Keyboard", "Mouse", "Monitor"];
const pricesPaise: number[] = [149900, 49900, 1299900];
const inStock: boolean[] = [true, true, false];

console.log(productNames);
console.log(pricesPaise);
console.log(inStock);
```

```bash
npx tsc --noEmit
node array-types.ts
```

```text
[ 'Keyboard', 'Mouse', 'Monitor' ]
[ 149900, 49900, 1299900 ]
[ true, true, false ]
```

Read `number[]` as "an array of numbers". Every item must be a number, now and later

array-wrong.ts

```typescript
const pricesPaise: number[] = [149900, "49900", 1299900];

pricesPaise.push("99900");
```

```text
array-wrong.ts:1:40 - error TS2322: Type 'string' is not assignable to type 'number'.

1 const pricesPaise: number[] = [149900, "49900", 1299900];
                                         ~~~~~~~

array-wrong.ts:3:18 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

3 pricesPaise.push("99900");
                   ~~~~~~~


Found 2 errors in the same file, starting at: array-wrong.ts:1
```

The first error points at the one wrong item, not the whole array. The second shows that `push` knows the type too: its parameter is a `number`, because the array holds numbers.

![A number array accepts numbers and rejects a text, both in the starting list and when an item is pushed later](images/06-arrays-and-tuples/typed-array.gif)

---

## Two Ways to Write an Array Type

array-generic.ts

```typescript
const names: Array<string> = ["Riya", "Sara"];
const sameType: string[] = names;

console.log(sameType);
```

```text
[ 'Riya', 'Sara' ]
```

`Array<string>` and `string[]` are exactly the same type. The `<string>` form is called a **generic**: you will learn it in Session 14. This course writes `string[]`, like most TypeScript code.

---

## TypeScript Infers Array Types

Most of the time you do not write the type at all

array-inference.ts

```typescript
const prices = [149900, 49900];
const names = ["Riya", "Sara"];
const flags = [true, false];
const mixed = ["Keyboard", 149900];
const grid = [[1, 2], [3, 4]];
const words = "a,b,c".split(",");
const args = process.argv;
```

Point at each name in VS Code

| Variable  | Inferred type             | Why                                             |
| --------- | ------------------------- | ----------------------------------------------- |
| `prices`  | `number[]`                | Every item is a number                          |
| `names`   | `string[]`                | Every item is a text                            |
| `flags`   | `boolean[]`               | Every item is `true` or `false`                 |
| `mixed`   | `(string \| number)[]`     | Items of two types: each item is a text **or** a number (`\|` is Session 09) |
| `grid`    | `number[][]`              | An array of arrays of numbers                   |
| `words`   | `string[]`                | `split` always returns a `string[]`             |
| `args`    | `string[]`                | `process.argv` is typed by `@types/node`        |

The rule from Session 03 still holds: write the type when TypeScript cannot know it, and let it infer the rest. An array with a starting value does not need a type.

![Each array literal is shown next to the type TypeScript infers for it](images/06-arrays-and-tuples/array-inference.gif)

---

## Empty Arrays Need a Type

An empty array tells TypeScript nothing about its items

empty-array.ts

```typescript
const names = [];
names.push("Riya");
names.push(25);

console.log(names);
```

```bash
npx tsc --noEmit
node empty-array.ts
```

No errors, and

```text
[ 'Riya', 25 ]
```

TypeScript let the type grow with each `push`: after these lines, `names` is `(string | number)[]`. A list of names with a number in it is not what you wanted, and nothing stopped it.

When the pushes happen inside another function, TypeScript cannot follow them, and it says so

empty-fn.ts

```typescript
const cart = [];

function addItem(name: string) {
  cart.push(name);
}

addItem("Keyboard");
console.log(cart);
```

```text
empty-fn.ts:1:7 - error TS7034: Variable 'cart' implicitly has type 'any[]' in some locations where its type cannot be determined.

1 const cart = [];
        ~~~~

empty-fn.ts:8:13 - error TS7005: Variable 'cart' implicitly has an 'any[]' type.

8 console.log(cart);
              ~~~~


Found 2 errors in the same file, starting at: empty-fn.ts:1
```

Both are `TS7...` errors: "this would silently become `any`" (Session 05). The fix is always the same, give an empty array its type

empty-typed.ts

```typescript
const cart: string[] = [];

function addItem(name: string) {
  cart.push(name);
}

addItem("Keyboard");
addItem("Mouse");
console.log(cart);
```

```text
[ 'Keyboard', 'Mouse' ]
```

---

## Arrays of Objects

An array of objects is inferred too

cart-items.ts

```typescript
const cart = [
  { name: "Keyboard", pricePaise: 149900 },
  { name: "Mouse", pricePaise: 49900 }
];

const names = cart.map((item) => item.name);
console.log(names);
```

```text
[ 'Keyboard', 'Mouse' ]
```

The type of `cart` is `{ name: string; pricePaise: number; }[]`: an array of objects with a `name` and a `pricePaise`. Every new item must have the same shape

cart-items-wrong.ts

```typescript
const cart = [
  { name: "Keyboard", pricePaise: 149900 },
  { name: "Mouse", pricePaise: 49900 }
];

cart.push({ name: "Monitor", pricePaise: "1299900" });
```

```text
cart-items-wrong.ts:6:30 - error TS2322: Type 'string' is not assignable to type 'number'.

6 cart.push({ name: "Monitor", pricePaise: "1299900" });
                               ~~~~~~~~~~

  cart-items-wrong.ts:2:23 - The expected type comes from property 'pricePaise' which is declared here on type '{ name: string; pricePaise: number; }'
    2   { name: "Keyboard", pricePaise: 149900 },
                            ~~~~~~~~~~~~~~~~~~


Found 1 error in cart-items-wrong.ts:6
```

Writing object types yourself, like an empty cart that will hold such objects, is the next session, Session 07.

---

## Array Methods Know the Types

array-methods.ts

```typescript
const pricesPaise = [149900, 49900, 1299900];

const withGst = pricesPaise.map((price) => Math.round(price * 1.18));
const cheap = pricesPaise.filter((price) => price < 100000);
const labels = pricesPaise.map((price) => `Rs. ${price / 100}`);
const total = pricesPaise.reduce((sum, price) => sum + price, 0);
const overTenThousand = pricesPaise.find((price) => price > 1000000);

console.log(withGst);
console.log(cheap);
console.log(labels);
console.log(total);
console.log(overTenThousand);
```

```text
[ 176882, 58882, 1533882 ]
[ 49900 ]
[ 'Rs. 1499', 'Rs. 499', 'Rs. 12999' ]
1499700
1299900
```

| Variable           | Type                    | Why                                                  |
| ------------------ | ----------------------- | ---------------------------------------------------- |
| `withGst`          | `number[]`              | `map` returns what the callback returns: a number    |
| `cheap`            | `number[]`              | `filter` keeps some items, so the type stays the same |
| `labels`           | `string[]`              | This callback returns a text                         |
| `total`            | `number`                | `reduce` starts at `0` and adds numbers              |
| `overTenThousand`  | `number \| undefined`    | `find` may find nothing                              |

Look at the callbacks: `(price) => ...` has no type on `price`, and there is no TS7006 error. In Session 03 every parameter needed a type, but here TypeScript knows that `map` on a `number[]` calls the callback with a number. Because of that, mistakes inside callbacks are caught

callback.ts

```typescript
const pricesPaise = [149900, 49900, 1299900];

const labels = pricesPaise.map((price) => price.toUpperCase());
```

```text
callback.ts:3:49 - error TS2339: Property 'toUpperCase' does not exist on type 'number'.

3 const labels = pricesPaise.map((price) => price.toUpperCase());
                                                  ~~~~~~~~~~~


Found 1 error in callback.ts:3
```

`find` returns `undefined` when nothing matches, and TypeScript makes you handle it

find-price.ts

```typescript
const pricesPaise = [149900, 49900, 1299900];

const expensive = pricesPaise.find((price) => price > 5000000);
console.log(expensive.toFixed(2));
```

```text
find-price.ts:4:13 - error TS18048: 'expensive' is possibly 'undefined'.

4 console.log(expensive.toFixed(2));
              ~~~~~~~~~


Found 1 error in find-price.ts:4
```

Nothing costs more than Rs. 50,000, so at run time

```text
file:///D:/ts-practice/06-arrays-tuples/find-price.ts:4
console.log(expensive.toFixed(2));
                      ^

TypeError: Cannot read properties of undefined (reading 'toFixed')
    at file:///D:/ts-practice/06-arrays-tuples/find-price.ts:4:23
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

The arguments of other methods are checked as well

includes.ts

```typescript
const pricesPaise = [149900, 49900, 1299900];

console.log(pricesPaise.includes("49900"));
```

```text
includes.ts:3:34 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

3 console.log(pricesPaise.includes("49900"));
                                   ~~~~~~~


Found 1 error in includes.ts:3
```

JavaScript would print `false`, and the bug, a price stored as text somewhere, would stay hidden.

![A number array flows through map, filter and find, and the type of each result is shown at every step](images/06-arrays-and-tuples/array-methods.gif)

---

## Reading an Item Can Give undefined

first-item.ts

```typescript
const names = ["Riya", "Sara", "Aman"];

const first = names[0];
console.log(first.toUpperCase());
```

```text
first-item.ts:4:13 - error TS18048: 'first' is possibly 'undefined'.

4 console.log(first.toUpperCase());
              ~~~~~


Found 1 error in first-item.ts:4
```

`names[0]` exists, so why the error? The type `string[]` says "texts, **any number** of them", even zero. TypeScript does not count the items, so `names[0]` has the type `string | undefined`. And reading past the end really gives `undefined`

out-of-range.ts

```typescript
const names = ["Riya", "Sara", "Aman"];

console.log(names[5]);
console.log(names[5].toUpperCase());
```

```text
out-of-range.ts:4:13 - error TS2532: Object is possibly 'undefined'.

4 console.log(names[5].toUpperCase());
              ~~~~~~~~


Found 1 error in out-of-range.ts:4
```

```bash
node out-of-range.ts
```

```text
undefined
file:///D:/ts-practice/06-arrays-tuples/out-of-range.ts:4
console.log(names[5].toUpperCase());
                     ^

TypeError: Cannot read properties of undefined (reading 'toUpperCase')
    at file:///D:/ts-practice/06-arrays-tuples/out-of-range.ts:4:22
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

This check comes from `noUncheckedIndexedAccess` in your tsconfig (Session 02). Many projects do not turn it on, and there `names[5]` is simply `string`, and the crash above passes `tsc` silently. With it, every read by position is checked.

The same applies to `at()` and to destructuring

destructure.ts

```typescript
const names = ["Riya", "Sara"];
const [first, second, third] = names;
console.log(first, second, third);
```

```text
Riya Sara undefined
```

`first`, `second` and `third` are all `string | undefined`, and `third` really is `undefined`.

A length check does not help, because TypeScript does not connect `names.length` with the items

length-check.ts

```typescript
const names = ["Riya", "Sara", "Aman"];

if (names.length > 0) {
  console.log(names[0].toUpperCase());
}

for (let i = 0; i < names.length; i++) {
  console.log(names[i].toUpperCase());
}
```

```text
length-check.ts:4:15 - error TS2532: Object is possibly 'undefined'.

4   console.log(names[0].toUpperCase());
                ~~~~~~~~

length-check.ts:8:15 - error TS2532: Object is possibly 'undefined'.

8   console.log(names[i].toUpperCase());
                ~~~~~~~~


Found 2 errors in the same file, starting at: length-check.ts:4
```

![Reading items by position: positions 0 to 2 hold names, position 5 holds undefined, so every read has the type string or undefined; a for...of loop only visits real items](images/06-arrays-and-tuples/index-undefined.gif)

---

## Safe Ways to Read Items

first-fixed.ts

```typescript
const names = ["Riya", "Sara", "Aman"];

const first = names[0];
if (first !== undefined) {
  console.log(first.toUpperCase());
}

const second = names[1] ?? "nobody";
console.log(second.toUpperCase());

for (const name of names) {
  console.log(name.toUpperCase());
}
```

```bash
npx tsc --noEmit
node first-fixed.ts
```

```text
RIYA
SARA
RIYA
SARA
AMAN
```

| You want                     | Use                                    | The item's type     |
| ---------------------------- | -------------------------------------- | ------------------- |
| Every item                   | `for (const name of names)`, `map`, `filter`, `forEach` | `string` |
| One item, and it may be missing | `const x = names[0]` + `if (x !== undefined)` | `string` inside the `if` |
| One item, with a default     | `names[0] ?? "nobody"`                 | `string`            |
| The last item                | `names.at(-1)`, then check it          | `string \| undefined` |

Loops and array methods never need a check: they only visit items that exist. Prefer them over loops with an index, like `for (let i = 0; ...)`.

---

## readonly Arrays

Some lists should never change: the days a shop is open, the allowed sizes of a T-shirt. Write `readonly` before the type

readonly.ts

```typescript
const weekDays: readonly string[] = ["Mon", "Tue", "Wed", "Thu", "Fri"];

weekDays.push("Sat");
weekDays[0] = "Sun";
```

```text
readonly.ts:3:10 - error TS2339: Property 'push' does not exist on type 'readonly string[]'.

3 weekDays.push("Sat");
           ~~~~

readonly.ts:4:1 - error TS2542: Index signature in type 'readonly string[]' only permits reading.

4 weekDays[0] = "Sun";
  ~~~~~~~~~~~


Found 2 errors in the same file, starting at: readonly.ts:3
```

A `readonly` array has no methods that change it: no `push`, `pop`, `shift`, `splice`, `sort` or `reverse`. Methods that return a new array, like `map`, `filter` and `slice`, all work. `ReadonlyArray<string>` is the same type, written the generic way.

`readonly` exists only for `tsc`. Like every type, it disappears when the code runs (Session 01)

readonly-runtime.ts

```typescript
const weekDays: readonly string[] = ["Mon", "Tue", "Wed", "Thu", "Fri"];

// @ts-expect-error: push does not exist on a readonly array
weekDays.push("Sat");
console.log(weekDays);
```

```text
[ 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat' ]
```

For a list that must not change at run time either, use `Object.freeze`. TypeScript knows that it returns a readonly array

freeze.ts

```typescript
const weekDays = Object.freeze(["Mon", "Tue", "Wed", "Thu", "Fri"]);

// @ts-expect-error: push does not exist on a readonly array
weekDays.push("Sat");
```

The type of `weekDays` is `readonly string[]`, and Node.js refuses the change

```text
file:///D:/ts-practice/06-arrays-tuples/freeze.ts:4
weekDays.push("Sat");
         ^

TypeError: Cannot add property 5, object is not extensible
    at Array.push (<anonymous>)
    at file:///D:/ts-practice/06-arrays-tuples/freeze.ts:4:10
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

![A readonly array: reading and map work, while push and changing an item are blocked by tsc](images/06-arrays-and-tuples/readonly-array.gif)

---

## readonly Parameters Protect the Caller

`readonly` is most useful on function parameters. This function looks harmless

cheapest-first.ts

```typescript
function cheapestFirst(prices: number[]): number[] {
  return prices.sort((a, b) => a - b);
}

const cartPrices = [12999, 499, 1499];
const sorted = cheapestFirst(cartPrices);

console.log("sorted:", sorted);
console.log("cart:  ", cartPrices);
```

```text
sorted: [ 499, 1499, 12999 ]
cart:   [ 499, 1499, 12999 ]
```

`sort` changes the array in place, so the caller's cart was reordered too. Promise not to change the parameter with `readonly`

cheapest-first.ts

```typescript
function cheapestFirst(prices: readonly number[]): number[] {
  return prices.sort((a, b) => a - b);
}

const cartPrices = [12999, 499, 1499];
const sorted = cheapestFirst(cartPrices);

console.log("sorted:", sorted);
console.log("cart:  ", cartPrices);
```

```text
cheapest-first.ts:2:17 - error TS2339: Property 'sort' does not exist on type 'readonly number[]'.

2   return prices.sort((a, b) => a - b);
                  ~~~~

cheapest-first.ts:2:23 - error TS7006: Parameter 'a' implicitly has an 'any' type.

2   return prices.sort((a, b) => a - b);
                        ~

cheapest-first.ts:2:26 - error TS7006: Parameter 'b' implicitly has an 'any' type.

2   return prices.sort((a, b) => a - b);
                           ~


Found 3 errors in the same file, starting at: cheapest-first.ts:2
```

One mistake, three errors (Session 05): without a `sort` method, TypeScript does not know what `a` and `b` are. Fix the first one: `toSorted` returns a sorted **copy**

cheapest-first.ts

```typescript
function cheapestFirst(prices: readonly number[]): number[] {
  return prices.toSorted((a, b) => a - b);
}

const cartPrices = [12999, 499, 1499];
const sorted = cheapestFirst(cartPrices);

console.log("sorted:", sorted);
console.log("cart:  ", cartPrices);
```

```text
sorted: [ 499, 1499, 12999 ]
cart:   [ 12999, 499, 1499 ]
```

All three errors are gone, and the cart keeps its order. Every method that changes an array has a copying twin

| Changes the array   | Returns a new array instead       |
| ------------------- | --------------------------------- |
| `sort()`            | `toSorted()`                      |
| `reverse()`         | `toReversed()`                    |
| `splice()`          | `toSpliced()`                     |
| `list[2] = value`   | `list.with(2, value)`             |

A normal array can be passed to a `readonly` parameter, as `cartPrices` was. The other way round is an error, because the function could change a list that must not change

readonly-pass.ts

```typescript
function addDelivery(prices: number[]): void {
  prices.push(99);
}

const fixedPrices: readonly number[] = [499, 1499];
addDelivery(fixedPrices);
```

```text
readonly-pass.ts:6:13 - error TS4104: The type 'readonly number[]' is 'readonly' and cannot be assigned to the mutable type 'number[]'.

6 addDelivery(fixedPrices);
              ~~~~~~~~~~~


Found 1 error in readonly-pass.ts:6
```

A good habit: when a function only reads an array, type the parameter as `readonly`.

![sort reorders the caller's array, while toSorted on a readonly parameter returns a sorted copy and leaves the cart as it was](images/06-arrays-and-tuples/sort-copy.gif)

---

## const Is Not readonly

const-array.ts

```typescript
const cart = ["Keyboard"];

cart.push("Mouse");
console.log(cart);

cart = ["Monitor"];
```

```text
const-array.ts:6:1 - error TS2588: Cannot assign to 'cart' because it is a constant.

6 cart = ["Monitor"];
  ~~~~


Found 1 error in const-array.ts:6
```

Only the last line is an error. Remove it and run

```text
[ 'Keyboard', 'Mouse' ]
```

|                               | `const cart = [...]`   | `const cart: readonly string[] = [...]` |
| ----------------------------- | ---------------------- | --------------------------------------- |
| Point `cart` at a new array   | Error                  | Error                                   |
| `cart.push(...)`              | Allowed                | Error                                   |
| `cart[0] = ...`               | Allowed                | Error                                   |

`const` is about the **variable**: it always points to the same array. `readonly` is about the **array**: its items cannot change.

![const keeps the variable pointing at the same array, but the array can still change; readonly locks the items](images/06-arrays-and-tuples/const-vs-readonly.gif)

---

## Arrays of Arrays

sales.ts

```typescript
// orders per day, Monday to Friday, for two weeks
const ordersByWeek: number[][] = [
  [12, 15, 9, 20, 18],
  [14, 11, 16, 22, 25]
];

for (const week of ordersByWeek) {
  const weekTotal = week.reduce((sum, orders) => sum + orders, 0);
  console.log(`Week total: ${weekTotal}`);
}
```

```text
Week total: 74
Week total: 88
```

`number[][]` is "an array of `number[]`". In the loop, `week` is a `number[]`. Reading by position goes through two levels: `ordersByWeek[0]` is `number[] | undefined`, and `ordersByWeek[0]?.[1]` would be `number | undefined` (`?.` is Session 11).

---

## Tuples: A Fixed Type in Each Position

A **tuple** is an array with a fixed length, where each position has its own type

tuple.ts

```typescript
const product: [string, number] = ["Keyboard", 149900];

const productName = product[0];
const pricePaise = product[1];

console.log(productName.toUpperCase(), pricePaise / 100);
```

```text
KEYBOARD 1499
```

`productName` is a `string` and `pricePaise` is a `number`, and neither can be `undefined`: TypeScript knows a `[string, number]` always has exactly two items. That is why there is no error on `toUpperCase`, unlike with `names[0]`.

The order and the length are checked

tuple-wrong.ts

```typescript
const keyboard: [string, number] = [149900, "Keyboard"];
const mouse: [string, number] = ["Mouse"];
const monitor: [string, number] = ["Monitor", 1299900, 5];

const webcam: [string, number] = ["Webcam", 249900];
console.log(webcam[2]);
```

```text
tuple-wrong.ts:1:37 - error TS2322: Type 'number' is not assignable to type 'string'.

1 const keyboard: [string, number] = [149900, "Keyboard"];
                                      ~~~~~~

tuple-wrong.ts:1:45 - error TS2322: Type 'string' is not assignable to type 'number'.

1 const keyboard: [string, number] = [149900, "Keyboard"];
                                              ~~~~~~~~~~

tuple-wrong.ts:2:7 - error TS2322: Type '[string]' is not assignable to type '[string, number]'.
  Source has 1 element(s) but target requires 2.

2 const mouse: [string, number] = ["Mouse"];
        ~~~~~

tuple-wrong.ts:3:7 - error TS2322: Type '[string, number, number]' is not assignable to type '[string, number]'.
  Source has 3 element(s) but target allows only 2.

3 const monitor: [string, number] = ["Monitor", 1299900, 5];
        ~~~~~~~

tuple-wrong.ts:6:20 - error TS2493: Tuple type '[string, number]' of length '2' has no element at index '2'.

6 console.log(webcam[2]);
                     ~


Found 5 errors in the same file, starting at: tuple-wrong.ts:1
```

| Line | Mistake                         | Error                                         |
| ---- | ------------------------------- | --------------------------------------------- |
| 1    | Items in the wrong order        | TS2322 on each item                           |
| 2    | One item missing                | `Source has 1 element(s) but target requires 2.` |
| 3    | One item too many               | `Source has 3 element(s) but target allows only 2.` |
| 6    | Reading a position that does not exist | TS2493                                 |

![A tuple has a slot with its own type for each position: values in the right order fit, the wrong order, a missing item or an extra item do not](images/06-arrays-and-tuples/tuple-slots.gif)

---

## An Array Literal Is Not a Tuple

TypeScript never infers a tuple. Without the type, you get an array

tuple-infer.ts

```typescript
const product = ["Keyboard", 149900];

console.log(product[1] / 100);
```

```text
tuple-infer.ts:3:13 - error TS2362: The left-hand side of an arithmetic operation must be of type 'any', 'number', 'bigint' or an enum type.

3 console.log(product[1] / 100);
              ~~~~~~~~~~

tuple-infer.ts:3:13 - error TS2532: Object is possibly 'undefined'.

3 console.log(product[1] / 100);
              ~~~~~~~~~~


Found 2 errors in the same file, starting at: tuple-infer.ts:3
```

The type of `product` is `(string | number)[]`: any number of items, each a text or a number. So `product[1]` might be a text, and might not exist. When the positions matter, write the tuple type: `const product: [string, number] = ...`. (Session 13 shows another way, `as const`.)

---

## Labels, Optional Items and Rest Items

tuple-labels.ts

```typescript
const product: [name: string, pricePaise: number] = ["Keyboard", 149900];
const delivery: [city: string, days?: number] = ["Pune"];
const scores: [student: string, ...marks: number[]] = ["Riya", 82, 91, 77];

console.log(product, delivery, scores);
```

```text
[ 'Keyboard', 149900 ] [ 'Pune' ] [ 'Riya', 82, 91, 77 ]
```

| Written as                  | Means                                                                  |
| --------------------------- | ---------------------------------------------------------------------- |
| `[name: string, pricePaise: number]` | **Labels**: only names for reading, shown by VS Code. The type is still `[string, number]` |
| `[city: string, days?: number]` | **Optional item**: `days` may be left out                          |
| `[student: string, ...marks: number[]]` | **Rest items**: a text, then any number of numbers         |

Use labels whenever the positions are not obvious. `[string, number]` says little; `[name: string, pricePaise: number]` explains itself.

An optional item may be missing, so reading it gives `number | undefined`

tuple-optional.ts

```typescript
const delivery: [city: string, days?: number] = ["Pune"];

const days = delivery[1];
console.log(days.toFixed(0));
```

```text
tuple-optional.ts:4:13 - error TS18048: 'days' is possibly 'undefined'.

4 console.log(days.toFixed(0));
              ~~~~


Found 1 error in tuple-optional.ts:4
```

Two more rules

tuple-labels-wrong.ts

```typescript
const delivery: [city: string, days?: number] = ["Pune", undefined];
const scores: [student: string, ...marks: number[]] = [82, 91];
```

```text
tuple-labels-wrong.ts:1:7 - error TS2322: Type '[string, undefined]' is not assignable to type '[city: string, days?: number]'.
  Type at position 1 in source is not compatible with type at position 1 in target.
    Type 'undefined' is not assignable to type 'number'.

1 const delivery: [city: string, days?: number] = ["Pune", undefined];
        ~~~~~~~~

tuple-labels-wrong.ts:2:56 - error TS2322: Type 'number' is not assignable to type 'string'.

2 const scores: [student: string, ...marks: number[]] = [82, 91];
                                                         ~~


Found 2 errors in the same file, starting at: tuple-labels-wrong.ts:1
```

- An optional item is **left out**, not written as `undefined`. This strict rule comes from `exactOptionalPropertyTypes` in your tsconfig (Session 02).
- Rest items come after the fixed ones: `scores` must start with the student's name.

---

## Tuples Can Still Grow

A surprise: a tuple still has `push`

tuple-push.ts

```typescript
const point: [number, number] = [10, 20];

point.push(30);
console.log(point, point.length);
```

```bash
npx tsc --noEmit
node tuple-push.ts
```

No errors, and

```text
[ 10, 20, 30 ] 3
```

The type still says two items. A tuple is an array underneath, and TypeScript allows its array methods. Make the tuple `readonly`, and both kinds of change are errors

tuple-readonly.ts

```typescript
const point: readonly [number, number] = [10, 20];

point.push(30);
point[0] = 15;
```

```text
tuple-readonly.ts:3:7 - error TS2339: Property 'push' does not exist on type 'readonly [number, number]'.

3 point.push(30);
        ~~~~

tuple-readonly.ts:4:7 - error TS2540: Cannot assign to '0' because it is a read-only property.

4 point[0] = 15;
        ~


Found 2 errors in the same file, starting at: tuple-readonly.ts:3
```

For a tuple that holds fixed data, like a location or a price pair, `readonly` is the safe choice.

---

## Returning Two Values with a Tuple

A function returns one value. A tuple lets that value hold two

split-price.ts

```typescript
function splitPrice(paise: number): [number, number] {
  return [Math.floor(paise / 100), paise % 100];
}

const [rupees, paise] = splitPrice(149950);
console.log(`Rs. ${rupees} and ${paise} paise`);
```

```text
Rs. 1499 and 50 paise
```

Destructuring a tuple gives each variable its exact type: `rupees` and `paise` are `number`, not `number | undefined` as with an array ([destructure.ts](#reading-an-item-can-give-undefined)). The tuple promises both items are there.

---

## Tuples You Already Use

`Object.entries` turns an object into an array of tuples

stock.ts

```typescript
const stock = { keyboard: 12, mouse: 40, monitor: 3 };

const entries = Object.entries(stock);

for (const [product, count] of entries) {
  console.log(`${product}: ${count}`);
}
```

```text
keyboard: 12
mouse: 40
monitor: 3
```

The type of `entries` is `[string, number][]`: an array where each item is a `[key, value]` tuple. That is why the loop can destructure each item into `product` (a `string`) and `count` (a `number`).

---

## Array or Tuple?

|                         | Array: `number[]`                       | Tuple: `[string, number]`                    |
| ----------------------- | --------------------------------------- | -------------------------------------------- |
| Length                  | Any                                     | Fixed (unless it has optional or rest items) |
| Item types              | All the same                            | One per position                             |
| Reading an item         | `T \| undefined` (with your tsconfig)    | The exact type of that position              |
| Inferred from `[...]`   | Yes                                     | No, always write the type                    |
| Typical use             | A list of products, prices, names       | A pair or a small record: name and price, key and value |

When a tuple grows past two or three items, an object with named properties is usually clearer: `{ name, price, quantity }` instead of `[string, number, number]`. That is the next session.

![An array is a row of items of one type with any length; a tuple is a short row of fixed positions, each with its own type](images/06-arrays-and-tuples/array-vs-tuple.gif)

---

## Mini Example 1: A Command-Line Cart Total

A script that adds up prices typed after its name. `process.argv` is a `string[]`: the path to Node.js, the path to the file, then every word you typed

cart-total.ts

```typescript
// Prices in rupees, from the command line: node cart-total.ts 1499 499 2999
function parsePrices(args: readonly string[]): number[] {
  const prices: number[] = [];
  for (const arg of args) {
    const price = Number(arg);
    if (Number.isNaN(price) || price <= 0) {
      console.log(`Skipping "${arg}": not a price`);
    } else {
      prices.push(price);
    }
  }
  return prices;
}

function total(prices: readonly number[]): number {
  return prices.reduce((sum, price) => sum + price, 0);
}

const prices = parsePrices(process.argv.slice(2));

if (prices.length === 0) {
  console.log("Usage: node cart-total.ts <price> <price> ...");
} else {
  console.log(`Items: ${prices.length}`);
  console.log(`Total: Rs. ${total(prices)}`);
  console.log(`Cheapest first: ${prices.toSorted((a, b) => a - b).join(", ")}`);
  console.log(`Most expensive: Rs. ${Math.max(...prices)}`);
}
```

```bash
npx tsc --noEmit
node cart-total.ts 1499 499 abc 2999
```

```text
Skipping "abc": not a price
Items: 3
Total: Rs. 4997
Cheapest first: 499, 1499, 2999
Most expensive: Rs. 2999
```

```bash
node cart-total.ts
```

```text
Usage: node cart-total.ts <price> <price> ...
```

| Code                                   | Type at work                                                         |
| -------------------------------------- | -------------------------------------------------------------------- |
| `process.argv.slice(2)`                | `string[]`: every word from the command line, as text                |
| `args: readonly string[]`              | The function only reads the words                                    |
| `const prices: number[] = []`          | An empty array, so it needs its type                                 |
| `for (const arg of args)`              | `arg` is a `string`, never `undefined`                               |
| `prices.reduce(...)`                   | Returns a `number`                                                   |
| `prices.toSorted(...)`                 | A sorted copy, so `prices` keeps its order. `sort` would be an error on a `readonly` array |
| `Math.max(...prices)`                  | Spreading a `number[]` into a function that takes numbers            |

Try it: add `prices.push(0);` inside `total`, and read the error. `total` promised to only read the prices.

![Words from the command line become a string array, are turned into numbers, the bad one is skipped, and the prices are added up](images/06-arrays-and-tuples/cli-total.gif)

---

## Mini Example 2: Reading Order Lines from CSV

A CSV file has one record per line, with values separated by commas. Each line here is a product name, a price and a quantity: a perfect tuple

order-lines.ts

```typescript
const csv = `Keyboard,1499,2
Mouse,499,1
Monitor,12999,1`;

function parseLine(line: string): [name: string, price: number, quantity: number] {
  const [name, price, quantity] = line.split(",");
  if (name === undefined || price === undefined || quantity === undefined) {
    throw new Error(`Bad line: ${line}`);
  }
  return [name, Number(price), Number(quantity)];
}

const orderLines = csv.split("\n").map(parseLine);

let total = 0;
for (const [name, price, quantity] of orderLines) {
  const lineTotal = price * quantity;
  total += lineTotal;
  console.log(`${name.padEnd(8)} ${quantity} x Rs. ${price} = Rs. ${lineTotal}`);
}
console.log(`Total: Rs. ${total}`);
```

```bash
npx tsc --noEmit
node order-lines.ts
```

```text
Keyboard 2 x Rs. 1499 = Rs. 2998
Mouse    1 x Rs. 499 = Rs. 499
Monitor  1 x Rs. 12999 = Rs. 12999
Total: Rs. 16496
```

How the types flow

1. `line.split(",")` is a `string[]`, so `name`, `price` and `quantity` are each `string | undefined`.
2. The `if` with `throw` handles the missing case. After it, all three are `string`.
3. The return type is a labeled tuple, so `parseLine` must return a text and two numbers.
4. `csv.split("\n").map(parseLine)` has the type `[name: string, price: number, quantity: number][]`.
5. The loop destructures each tuple, and `price * quantity` is a number times a number.

Forget the `Number(...)` calls in the `return`, and the tuple type catches it

```text
order-lines.ts:10:17 - error TS2322: Type 'string' is not assignable to type 'number'.

10   return [name, price, quantity];
                   ~~~~~

order-lines.ts:10:24 - error TS2322: Type 'string' is not assignable to type 'number'.

10   return [name, price, quantity];
                          ~~~~~~~~


Found 2 errors in the same file, starting at: order-lines.ts:10
```

Without it, `"1499" * "2"` would still work in JavaScript, but `total += lineTotal` with texts would join them. A line with a value missing, like `Webcam,2499`, stops at the check

```text
file:///D:/ts-practice/06-arrays-tuples/order-lines.ts:9
    throw new Error(`Bad line: ${line}`);
          ^

Error: Bad line: Webcam,2499
    at parseLine (file:///D:/ts-practice/06-arrays-tuples/order-lines.ts:9:11)
    at Array.map (<anonymous>)
    at file:///D:/ts-practice/06-arrays-tuples/order-lines.ts:14:36
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

A clear error with the bad line in it, instead of `undefined` showing up in the totals.

![Each CSV line is split into texts, checked, and turned into a name, price and quantity tuple, and the tuples are added up into the order total](images/06-arrays-and-tuples/csv-tuples.gif)

---

## Beginner Mistakes

### Mistake 1: An empty array without a type

`const cart = [];` lets any value in, and inside a function it gives `error TS7034: Variable 'cart' implicitly has type 'any[]' in some locations where its type cannot be determined.` Write `const cart: string[] = [];`

---

### Mistake 2: Writing [] as a type

empty-tuple.ts

```typescript
const cart: [] = [];

cart.push("Keyboard");
```

```text
empty-tuple.ts:3:11 - error TS2345: Argument of type '"Keyboard"' is not assignable to parameter of type 'never'.

3 cart.push("Keyboard");
            ~~~~~~~~~~


Found 1 error in empty-tuple.ts:3
```

`[]` as a type is an **empty tuple**: a list with exactly zero items, so nothing fits (`never`, Session 04). An empty list of texts is `string[]`.

---

### Mistake 3: Using names[0] without a check

`names[0].toUpperCase()` gives `error TS2532: Object is possibly 'undefined'.` Store the item and check it, give a default with `??`, or loop with `for...of`.

---

### Mistake 4: Thinking a length check makes an item safe

`if (names.length > 0) { names[0].toUpperCase() }` still gives TS2532: TypeScript does not connect the length with the items. Check the item itself.

---

### Mistake 5: Using the result of find without a check

`find` returns `undefined` when nothing matches, so `expensive.toFixed(2)` gives `error TS18048: 'expensive' is possibly 'undefined'.`, and it really crashed when run.

---

### Mistake 6: Thinking const makes an array unchangeable

`const cart = ["Keyboard"]` still allows `cart.push("Mouse")`. `const` only stops `cart = [...]`. For a list that must not change, use `readonly string[]`.

---

### Mistake 7: Sorting the caller's array

`sort` and `reverse` change the array they are called on, including an array passed in by someone else. Type read-only parameters as `readonly number[]`, and use `toSorted` and `toReversed`.

---

### Mistake 8: Expecting a tuple from an array literal

`const product = ["Keyboard", 149900]` is `(string | number)[]`, so `product[1] / 100` gives TS2362 and TS2532. Write the type: `[string, number]`.

---

### Mistake 9: Pushing to a tuple

`point.push(30)` on a `[number, number]` has no error, and the "pair" has three items. Use `readonly [number, number]` for tuples that must keep their shape.

---

### Mistake 10: Writing undefined into an optional tuple item

`["Pune", undefined]` for `[city: string, days?: number]` gives `Type 'undefined' is not assignable to type 'number'.` with your tsconfig. Leave the item out: `["Pune"]`.

---

### Mistake 11: Thinking readonly protects the array at run time

`readonly` is a type, and types are erased: with the error hidden, `push` still added `"Sat"`. Use `Object.freeze` when the list must not change at run time.

---

### Mistake 12: Passing a readonly array to a function that changes it

`addDelivery(fixedPrices)` gives `error TS4104: The type 'readonly number[]' is 'readonly' and cannot be assigned to the mutable type 'number[]'.` If the function only reads, make its parameter `readonly`. If it changes the array, pass a copy: `addDelivery([...fixedPrices])`.

---

## Practice Exercises

### Exercise 1

Make `const marks: number[] = [82, 91, 77];`. Push the text `"88"`, and read the error. Then push the number `88`, and print the array.

### Exercise 2

From `const pricesPaise = [149900, 49900, 1299900, 99900];`, build one chain: keep the prices above Rs. 1,000 (`filter`), turn them into labels like `"Rs. 1499"` (`map`), and join them with `", "` (`join`). Point at each step in VS Code: what is the type after `filter`, after `map` and after `join`?

### Exercise 3

Write `function average(marks: readonly number[]): number`. It returns `0` for an empty array. Test it with `[]` and `[80, 90, 70]`. Then try to call `marks.push(100)` inside it, and read the error.

### Exercise 4

Write `function minMax(prices: readonly number[]): [min: number, max: number]` that returns `[Math.min(...prices), Math.max(...prices)]`. Destructure the result into `cheapest` and `dearest`. What does it return for an empty array? Could TypeScript catch that?

### Exercise 5

In `first-fixed.ts`, print the **last** name with `names.at(-1)`. Use a check or `??` so that `tsc` accepts it. Then empty the array and run it again.

### Exercise 6

Make a readonly tuple for your city's location: `const office: readonly [latitude: number, longitude: number] = [18.52, 73.85];`. Try to change the latitude, and read the error. Then print `Latitude: 18.52, longitude: 73.85` using destructuring.

### Exercise 7

In Mini Example 1, also print the average price, and the prices above the average (`filter`). Test with `node cart-total.ts 100 200 900`.

### Exercise 8

In Mini Example 2, add a fourth column to every CSV line: a discount in percent, like `Keyboard,1499,2,10`. Add `discountPercent: number` to the tuple type first, and let the errors guide you through `parseLine` and the loop. What is the new total?

---

## Interview Questions

### How do you type an array in TypeScript?

With the item type followed by `[]`, like `string[]`, or the generic form `Array<string>`. Both mean the same. When the array has a starting value, TypeScript usually infers the type.

### What does TypeScript infer for ["Keyboard", 149900]?

`(string | number)[]`: an array of any length where each item is a text or a number. It never infers a tuple; for that, the type must be written.

### What is a tuple?

An array with a fixed number of items, where each position has its own type, like `[string, number]`. Reading a position gives the exact type of that position.

### What is the difference between an array and a tuple?

An array has any length and one item type. A tuple has a fixed length (apart from optional and rest items) and a type per position. Arrays are for lists, tuples for small groups of values that belong together, like a key and a value.

### What does readonly string[] mean?

An array whose items cannot be changed: it has no `push`, `pop`, `sort` or other changing methods, and items cannot be assigned. It is checked only by TypeScript; at run time it is a normal array, unless it was frozen with `Object.freeze`.

### What is the difference between const and readonly for arrays?

`const` stops the variable from pointing to a different array, but the array itself can still change. `readonly` stops changes to the array's items.

### Why does names[0] have the type string | undefined?

Because of `noUncheckedIndexedAccess`. An array type does not say how many items there are, so any position may be empty. Without this setting, `names[0]` is `string`, and reading past the end is not caught.

### What does find return?

The item type or `undefined`, like `number | undefined`, because no item may match. The result must be checked before it is used.

### Why do callback parameters not need a type in map or filter?

TypeScript knows the type of the array, so it knows what the callback receives. This is called contextual typing: the parameter's type comes from where the function is used.

### What are labeled tuple elements?

Names for the positions, like `[name: string, price: number]`. They do not change the type; they make the code and the editor hints easier to read.

### Can you push to a tuple?

Yes. A tuple is an array, and TypeScript allows `push`, even though the tuple type does not change. A `readonly` tuple prevents it.

### What type does Object.entries return?

An array of key and value tuples, like `[string, number][]` for an object with number values. The keys are always `string`.

### How do you give an empty array a type?

With an annotation: `const cart: string[] = [];`. Without one, TypeScript has to guess from later code, and inside functions it reports an implicit `any[]`.

### Why use toSorted instead of sort?

`sort` changes the original array, which can surprise the code that owns it. `toSorted` returns a sorted copy, and it also works on `readonly` arrays.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [05. Reading TypeScript Errors](05-reading-typescript-errors.md) | [Course outline](README.md#course-outline) | [07. Object Types](07-object-types.md) |
