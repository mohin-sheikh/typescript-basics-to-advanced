# Generics

> **Session 14 of 33** | Part 4: Generics and Type Tools | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [The Problem: One Function, Many Types](#the-problem-one-function-many-types)
- [Your First Generic Function](#your-first-generic-function)
- [tsc Finds the Type Argument](#tsc-finds-the-type-argument)
- [You Have Used Generics Already](#you-have-used-generics-already)
- [More Than One Type Parameter](#more-than-one-type-parameter)
- [Constraints: extends](#constraints-extends)
- [Generic Interfaces and Types](#generic-interfaces-and-types)
- [Default Type Arguments](#default-type-arguments)
- [When tsc Cannot Find T](#when-tsc-cannot-find-t)
- [When Not to Use a Generic](#when-not-to-use-a-generic)
- [Naming Type Parameters](#naming-type-parameters)
- [Mini Example 1: A Generic In-Memory Store](#mini-example-1-a-generic-in-memory-store)
- [Mini Example 2: A Result Type for Safe Parsing](#mini-example-2-a-result-type-for-safe-parsing)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `14-generics` inside `ts-practice`, with the one-line `tsconfig.json`

14-generics/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## The Problem: One Function, Many Types

Get the first item of a list. For numbers and for texts, you would write the same function twice

duplicate.ts

```typescript
function firstNumber(items: number[]): number | undefined {
  return items[0];
}

function firstString(items: string[]): string | undefined {
  return items[0];
}

console.log(firstNumber([149900, 49900]));
console.log(firstString(["Riya", "Sara"]));
```

```text
149900
Riya
```

Same body, different types, and the next type (products, orders) needs a third copy. Two quick fixes both fail. With `any`, the types are gone

first-any.ts

```typescript
function firstItem(items: any[]): any {
  return items[0];
}

const price = firstItem([149900, 49900]);
console.log(price.toUpperCase());
```

```text
file:///D:/ts-practice/14-generics/first-any.ts:6
console.log(price.toUpperCase());
                  ^

TypeError: price.toUpperCase is not a function
    at file:///D:/ts-practice/14-generics/first-any.ts:6:19
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` said nothing: `price` is `any`. With `unknown`, the result is safe but useless

first-unknown.ts

```typescript
function firstItem(items: unknown[]): unknown {
  return items[0];
}

const price = firstItem([149900, 49900]);
console.log(price.toFixed(2));
```

```text
first-unknown.ts:6:13 - error TS18046: 'price' is of type 'unknown'.

6 console.log(price.toFixed(2));
              ~~~~~


Found 1 error in first-unknown.ts:6
```

We passed numbers, but `tsc` forgot that on the way out. What we want: "whatever type goes in, the same type comes out".

![Three versions of firstItem side by side: with any, a number goes in and an unchecked any comes out; with unknown, a locked box comes out; with a generic, a number goes in and a number comes out](images/14-generics/any-unknown-generic.gif)

---

## Your First Generic Function

first-item.ts

```typescript
function firstItem<T>(items: T[]): T | undefined {
  return items[0];
}

const price = firstItem([149900, 49900]);
const customer = firstItem(["Riya", "Sara"]);

console.log(price?.toFixed(2));
console.log(customer?.toUpperCase());
```

```text
149900.00
RIYA
```

| Part                 | Meaning                                                      |
| -------------------- | ------------------------------------------------------------ |
| `<T>`                | A **type parameter**: a placeholder for a type, named `T`    |
| `items: T[]`         | The parameter is an array of that type                       |
| `: T \| undefined`   | The result is an item of the same type, or `undefined`       |

A type parameter works like a normal parameter, but for types: each call fills it in. For `firstItem([149900, 49900])`, `T` is `number`, so the result is `number | undefined`. For the names, `T` is `string`. One function, and the types are kept

first-item.ts

```typescript
function firstItem<T>(items: T[]): T | undefined {
  return items[0];
}

const price = firstItem([149900, 49900]);
console.log(price?.toUpperCase());
```

```text
first-item.ts:6:20 - error TS2339: Property 'toUpperCase' does not exist on type 'number'.

6 console.log(price?.toUpperCase());
                     ~~~~~~~~~~~


Found 1 error in first-item.ts:6
```

The `any` version crashed at run time with exactly this mistake. A function with type parameters is called a **generic** function.

![firstItem<T> as a machine with a slot T: numbers go in and T becomes number, so a number comes out; texts go in and T becomes string, so a text comes out](images/14-generics/type-parameter.gif)

---

## tsc Finds the Type Argument

You rarely write the type yourself: `tsc` works it out from the arguments. Point at each variable in VS Code

| Call                                  | `T` is              | Result type                      |
| ------------------------------------- | ------------------- | -------------------------------- |
| `firstItem([149900, 49900])`          | `number`            | `number \| undefined`            |
| `firstItem(["Riya", "Sara"])`         | `string`            | `string \| undefined`            |
| `firstItem([1001, "WEB-1002"])`       | `string \| number`  | `string \| number \| undefined`  |
| `firstItem([])`                       | `never`             | `undefined`                      |
| `firstItem<string>(["Riya"])`         | `string`, written   | `string \| undefined`            |

![The arguments of each call flow up into T: [149900, 49900] makes T number, ["Riya", "Sara"] makes T string, a mixed array makes T string | number](images/14-generics/infer-t.gif)

The empty array has no items, so `T` is `never` (no value at all, Session 04), and only `undefined` is left. The last row writes the **type argument** yourself, in angle brackets after the name. Then the arguments are checked against it

explicit.ts

```typescript
function firstItem<T>(items: T[]): T | undefined {
  return items[0];
}

const id = firstItem<string>([1001, 1002]);
```

```text
explicit.ts:5:31 - error TS2322: Type 'number' is not assignable to type 'string'.

5 const id = firstItem<string>([1001, 1002]);
                                ~~~~

explicit.ts:5:37 - error TS2322: Type 'number' is not assignable to type 'string'.

5 const id = firstItem<string>([1001, 1002]);
                                      ~~~~


Found 2 errors in the same file, starting at: explicit.ts:5
```

Let `tsc` infer it in most calls; write it when there is nothing to infer from (below), or when you want to say exactly which type you expect.

---

## You Have Used Generics Already

The angle brackets have been in your code and in hover texts all along

| Type                     | Means                                    | Seen in   |
| ------------------------ | ---------------------------------------- | --------- |
| `Array<string>`          | The same as `string[]`                   | 06        |
| `ReadonlyArray<number>`  | The same as `readonly number[]`          | 06        |
| `Map<string, number>`    | A `Map` with text keys and number values | 11        |
| `Promise<string>`        | A value that arrives later               | 23        |

Array methods are generic functions too. In TypeScript's own type definitions, `map` is declared as `map<U>(callbackfn: (value: T, index: number, array: T[]) => U, thisArg?: any): U[]`. `U` is whatever your callback returns, so `names.map((name) => name.length)` gives `number[]`. Point at `map` in such a call, and VS Code shows the filled-in version, `map<number>(...)`.

`Array` is a generic interface, `Map` a generic class (Session 20). The rest of this session shows how to write your own.

---

## More Than One Type Parameter

A function can have several type parameters, separated by commas

get-or-default.ts

```typescript
function getOrDefault<K, V>(map: Map<K, V>, key: K, fallback: V): V {
  return map.get(key) ?? fallback;
}

const stock = new Map([["KB-01", 12], ["MS-02", 0]]);

console.log(getOrDefault(stock, "KB-01", 0));
console.log(getOrDefault(stock, "MS-02", 99));
console.log(getOrDefault(stock, "MN-03", 0));
```

```text
12
0
0
```

`stock` is a `Map<string, number>`, so `K` is `string` and `V` is `number`. Both are then checked in every other place they appear

get-or-default.ts

```typescript
function getOrDefault<K, V>(map: Map<K, V>, key: K, fallback: V): V {
  return map.get(key) ?? fallback;
}

const stock = new Map([["KB-01", 12], ["MS-02", 0]]);

console.log(getOrDefault(stock, "MN-03", "none"));
console.log(getOrDefault(stock, 1001, 0));
```

```text
get-or-default.ts:7:42 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

7 console.log(getOrDefault(stock, "MN-03", "none"));
                                           ~~~~~~

get-or-default.ts:8:33 - error TS2345: Argument of type 'number' is not assignable to parameter of type 'string'.

8 console.log(getOrDefault(stock, 1001, 0));
                                  ~~~~


Found 2 errors in the same file, starting at: get-or-default.ts:7
```

The fallback must be a value of the map's type, and the key must be a key of the map's type. The connection between the parameters is exactly what generics describe. (`??` and not `||`, so the stock of `0` stays `0`, from Session 11.)

![getOrDefault<K, V>: the Map<string, number> fixes K to string and V to number, and those two types flow into the key and fallback parameters and the result](images/14-generics/two-parameters.gif)

---

## Constraints: extends

Find any record by its `id`. Without more information, `T` could be anything, even a number

find-by-id.ts

```typescript
function findById<T>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);
}
```

```text
find-by-id.ts:2:36 - error TS2339: Property 'id' does not exist on type 'T'.

2   return items.find((item) => item.id === id);
                                     ~~


Found 1 error in find-by-id.ts:2
```

Using `{ id: number }` instead of `T` loses everything else about the items

find-plain.ts

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

function findById(items: { id: number }[], id: number): { id: number } | undefined {
  return items.find((item) => item.id === id);
}

console.log(findById(products, 1)?.name);
```

```text
find-plain.ts:16:36 - error TS2339: Property 'name' does not exist on type '{ id: number; }'.

16 console.log(findById(products, 1)?.name);
                                      ~~~~


Found 1 error in find-plain.ts:16
```

The answer is both: a type parameter with a **constraint**

find-by-id.ts

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

function findById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);
}

const keyboard = findById(products, 1);
console.log(keyboard?.name);
console.log(findById(products, 3)?.name);
```

```text
Keyboard
undefined
```

`T extends { id: number }` means: "`T` can be any type, as long as it has an `id` that is a number". Inside the function, `item.id` is allowed. Outside, `T` is the full type, so `keyboard` is a `Product` with its `name`. Things without an `id` are refused

```typescript
console.log(findById(["Riya", "Sara"], 1));
console.log(findById([{ sku: "KB-01" }], 1));
```

```text
find-by-id.ts:16:23 - error TS2322: Type 'string' is not assignable to type '{ id: number; }'.

16 console.log(findById(["Riya", "Sara"], 1));
                         ~~~~~~

find-by-id.ts:16:31 - error TS2322: Type 'string' is not assignable to type '{ id: number; }'.

16 console.log(findById(["Riya", "Sara"], 1));
                                 ~~~~~~

find-by-id.ts:17:25 - error TS2353: Object literal may only specify known properties, and 'sku' does not exist in type '{ id: number; }'.

17 console.log(findById([{ sku: "KB-01" }], 1));
                           ~~~


Found 3 errors in the same file, starting at: find-by-id.ts:16
```

| Version                                   | Inside: `item.id`? | Outside: the result is      |
| ----------------------------------------- | ------------------ | --------------------------- |
| `<T>`                                     | No (TS2339)        | `T`, the full type          |
| `{ id: number }[]`                        | Yes                | Only `{ id: number }`       |
| `<T extends { id: number }>`              | Yes                | `T`, the full type          |

The `extends` here is the same word as in interfaces (Session 08), with the same meaning: "has at least these properties".

![T extends { id: number } is a gate that only lets in types with an id: Product and Customer pass and keep all their properties, a string and an object without id are stopped](images/14-generics/constraint.gif)

---

## Generic Interfaces and Types

Interfaces and type aliases can have type parameters too. APIs often wrap every answer in the same envelope

api-response.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

interface ApiResponse<T> {
  status: number;
  data: T;
}

interface Paginated<T> {
  items: T[];
  page: number;
  totalPages: number;
}

const single: ApiResponse<Product> = {
  status: 200,
  data: { id: 1, name: "Keyboard", pricePaise: 149900 }
};

const page: ApiResponse<Paginated<Product>> = {
  status: 200,
  data: {
    items: [{ id: 2, name: "Mouse", pricePaise: 49900 }],
    page: 1,
    totalPages: 3
  }
};

console.log(single.data.name);
console.log(`${page.data.items.length} item, page ${page.data.page} of ${page.data.totalPages}`);
```

```text
Keyboard
1 item, page 1 of 3
```

`ApiResponse<Product>` is "an `ApiResponse` where `T` is `Product`", so `data` is a `Product`. Generic types can be nested: `ApiResponse<Paginated<Product>>` has `data.items` of type `Product[]`. The envelope is written once and used for every kind of data.

With a generic type you must give the type argument, and the data is checked against it

api-response.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

interface ApiResponse<T> {
  status: number;
  data: T;
}

const single: ApiResponse<Product> = {
  status: 200,
  data: { id: 1, name: "Keyboard" }
};

const missingArgument: ApiResponse = { status: 500, data: null };
```

```text
api-response.ts:14:3 - error TS2741: Property 'pricePaise' is missing in type '{ id: number; name: string; }' but required in type 'Product'.

14   data: { id: 1, name: "Keyboard" }
     ~~~~

  api-response.ts:4:3 - 'pricePaise' is declared here.
    4   pricePaise: number;
        ~~~~~~~~~~

  api-response.ts:9:3 - The expected type comes from property 'data' which is declared here on type 'ApiResponse<Product>'
    9   data: T;
        ~~~~

api-response.ts:17:24 - error TS2314: Generic type 'ApiResponse<T>' requires 1 type argument(s).

17 const missingArgument: ApiResponse = { status: 500, data: null };
                          ~~~~~~~~~~~


Found 2 errors in the same file, starting at: api-response.ts:14
```

Notice the last related line: `on type 'ApiResponse<Product>'`. Error messages show the type arguments too.

![ApiResponse<T> is an envelope with a slot for data; with Product the slot holds a product, with Paginated<Product> it holds a page that holds products](images/14-generics/generic-interface.gif)

---

## Default Type Arguments

A type parameter can have a default, used when no argument is given

api-default.ts

```typescript
interface ApiResponse<T = unknown> {
  status: number;
  data: T;
}

const failed: ApiResponse = { status: 500, data: null };
const ok: ApiResponse<string[]> = { status: 200, data: ["Riya", "Sara"] };

console.log(ok.data.join(", "));
console.log(failed.data.length);
```

```text
api-default.ts:10:13 - error TS18046: 'failed.data' is of type 'unknown'.

10 console.log(failed.data.length);
               ~~~~~~~~~~~


Found 1 error in api-default.ts:10
```

`ApiResponse` alone now means `ApiResponse<unknown>`: allowed, and safe, because `unknown` data must be checked before use. `unknown` is a good default for "I do not know the data yet", much better than `any`.

---

## When tsc Cannot Find T

`tsc` infers `T` from the **arguments**. A function where `T` appears only in the return type gives it nothing to infer from

empty-list.ts

```typescript
function emptyList<T>(): T[] {
  return [];
}

const names = emptyList();
names.push("Riya");
console.log(names[0]?.toUpperCase());
```

```text
empty-list.ts:7:23 - error TS2339: Property 'toUpperCase' does not exist on type '{}'.

7 console.log(names[0]?.toUpperCase());
                        ~~~~~~~~~~~


Found 1 error in empty-list.ts:7
```

With nothing to go on, `T` became `unknown`, so `names` is `unknown[]` (point at it). The message says `'{}'` because `?.` removed `null` and `undefined` from `unknown`, and `{}` is "any value except those two". Give the type argument yourself

empty-list.ts

```typescript
function emptyList<T>(): T[] {
  return [];
}

const names = emptyList<string>();
names.push("Riya");
console.log(names[0]?.toUpperCase());
```

```text
RIYA
```

The same is true for `new Map()`: with no entries and no type arguments, `tsc` has nothing to infer from, while `new Map<string, number>()` says what goes in. Mini Example 1 creates stores the same way.

![emptyList() has no argument to infer T from, so T becomes unknown and the list is unknown[]; emptyList<string>() fills T by hand and the list is string[]](images/14-generics/explicit-type-argument.gif)

---

## When Not to Use a Generic

A type parameter is useful when it **connects** two places: a parameter and the result, or two parameters. When `T` appears only once, it connects nothing

print-value.ts

```typescript
function printValue<T>(value: T): void {
  console.log(`Value: ${String(value)}`);
}

function printUnknown(value: unknown): void {
  console.log(`Value: ${String(value)}`);
}

printValue(42);
printUnknown("Riya");
```

```text
Value: 42
Value: Riya
```

Both functions accept anything and do exactly the same. The `unknown` version is simpler to read. A rule from the TypeScript handbook: if a type parameter appears in only one place, you probably do not need it.

![In firstItem, T appears in the parameter and in the result, and a line connects them; in printValue, T appears only once, connects nothing, and could just as well be unknown](images/14-generics/connects.gif)

| Use a generic when                                | Example                                    |
| ------------------------------------------------- | ------------------------------------------ |
| The result type depends on an argument type       | `firstItem<T>(items: T[]): T \| undefined` |
| Two arguments must have related types             | `getOrDefault(map, key, fallback)`         |
| A type is a container for another type            | `ApiResponse<T>`, `Paginated<T>`           |

| Do not use one when                               | Instead                                    |
| ------------------------------------------------- | ------------------------------------------ |
| `T` appears only once                             | `unknown`, or the real type                |
| The function only ever works with one type        | That type                                  |

---

## Naming Type Parameters

| Name    | Usual meaning                       |
| ------- | ----------------------------------- |
| `T`     | "Type", the main one                |
| `U`     | A second type, after `T`            |
| `K`, `V` | Key and value, as in `Map<K, V>`   |
| `E`     | Element, or error                   |

Single capital letters are the convention for short functions. For longer code, descriptive names that start with `T` are common: `TItem`, `TResponse`.

---

## Mini Example 1: A Generic In-Memory Store

Every backend has the same operations for many kinds of data: add, find by id, list all. Write them once

store.ts

```typescript
interface Store<T extends { id: number }> {
  add(item: T): void;
  findById(id: number): T | undefined;
  all(): readonly T[];
}

function createStore<T extends { id: number }>(): Store<T> {
  const items: T[] = [];
  return {
    add(item) {
      items.push(item);
    },
    findById(id) {
      return items.find((item) => item.id === id);
    },
    all() {
      return items;
    }
  };
}

interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

interface Customer {
  id: number;
  name: string;
  city: string;
}

const products = createStore<Product>();
const customers = createStore<Customer>();

products.add({ id: 1, name: "Keyboard", pricePaise: 149900 });
products.add({ id: 2, name: "Mouse", pricePaise: 49900 });
customers.add({ id: 1, name: "Riya", city: "Pune" });

for (const product of products.all()) {
  console.log(`#${product.id} ${product.name}: Rs. ${product.pricePaise / 100}`);
}

const riya = customers.findById(1);
console.log(`Customer 1: ${riya?.name ?? "not found"} from ${riya?.city ?? "unknown"}`);
console.log(`Product 9: ${products.findById(9)?.name ?? "not found"}`);
```

```bash
npx tsc --noEmit
node store.ts
```

```text
#1 Keyboard: Rs. 1499
#2 Mouse: Rs. 499
Customer 1: Riya from Pune
Product 9: not found
```

| Code                                                  | From this session                                 |
| ----------------------------------------------------- | ------------------------------------------------- |
| `interface Store<T extends { id: number }>`           | A generic interface with a constraint             |
| `function createStore<T extends { id: number }>(): Store<T>` | A generic function that returns a generic type |
| `createStore<Product>()`                              | An explicit type argument: nothing to infer from  |
| `add(item)`, `findById(id)` without annotations       | Contextual typing from `Store<T>` (Session 12)    |
| `all(): readonly T[]`                                 | Callers cannot push into the store's own array    |

Each store keeps its own type. Mix them up, and `tsc` notices

```typescript
customers.add({ id: 2, name: "Sara" });
products.add(customers.findById(1));
```

```text
store.ts:48:15 - error TS2741: Property 'city' is missing in type '{ id: number; name: string; }' but required in type 'Customer'.

48 customers.add({ id: 2, name: "Sara" });
                 ~~~~~~~~~~~~~~~~~~~~~~~

  store.ts:31:3 - 'city' is declared here.
    31   city: string;
         ~~~~

store.ts:49:14 - error TS2345: Argument of type 'Customer | undefined' is not assignable to parameter of type 'Product'.
  Type 'undefined' is not assignable to type 'Product'.

49 products.add(customers.findById(1));
                ~~~~~~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: store.ts:48
```

A store for something without an `id` breaks the constraint

```typescript
const orders = createStore<string>();
```

```text
store.ts:48:28 - error TS2344: Type 'string' does not satisfy the constraint '{ id: number; }'.

48 const orders = createStore<string>();
                              ~~~~~~


Found 1 error in store.ts:48
```

And forgetting the type argument, `const products = createStore();`, makes `T` the constraint itself, `{ id: number }`, so a product with a `name` is "too much"

```text
store.ts:37:23 - error TS2353: Object literal may only specify known properties, and 'name' does not exist in type '{ id: number; }'.

37 products.add({ id: 1, name: "Keyboard", pricePaise: 149900 });
                         ~~~~

store.ts:38:23 - error TS2353: Object literal may only specify known properties, and 'name' does not exist in type '{ id: number; }'.

38 products.add({ id: 2, name: "Mouse", pricePaise: 49900 });
                         ~~~~

store.ts:42:41 - error TS2339: Property 'name' does not exist on type '{ id: number; }'.

42   console.log(`#${product.id} ${product.name}: Rs. ${product.pricePaise / 100}`);
                                           ~~~~

store.ts:42:62 - error TS2339: Property 'pricePaise' does not exist on type '{ id: number; }'.

42   console.log(`#${product.id} ${product.name}: Rs. ${product.pricePaise / 100}`);
                                                                ~~~~~~~~~~

store.ts:47:49 - error TS2339: Property 'name' does not exist on type '{ id: number; }'.

47 console.log(`Product 9: ${products.findById(9)?.name ?? "not found"}`);
                                                   ~~~~


Found 5 errors in the same file, starting at: store.ts:37
```

Mongoose models (Session 32) work on the same idea: one generic `Model<T>`, a typed model for each kind of document.

![createStore<Product>() and createStore<Customer>() come out of the same factory as two typed stores: a product fits only the product store, a customer only the customer store](images/14-generics/generic-store.gif)

---

## Mini Example 2: A Result Type for Safe Parsing

Instead of throwing, many functions return either a value or an error. A generic type says that once for every kind of value, using the discriminated union from Session 10

result.ts

```typescript
type Result<T> = { ok: true; value: T } | { ok: false; error: string };

function parseQuantity(text: string): Result<number> {
  const quantity = Number(text);
  if (!Number.isInteger(quantity) || quantity < 1) {
    return { ok: false, error: `"${text}" is not a quantity` };
  }
  return { ok: true, value: quantity };
}

function parseCoupon(text: string): Result<string> {
  const code = text.trim().toUpperCase();
  if (!/^[A-Z]+[0-9]+$/.test(code)) {
    return { ok: false, error: `"${text}" is not a coupon code` };
  }
  return { ok: true, value: code };
}

function showResult<T>(result: Result<T>, show: (value: T) => string): string {
  if (result.ok) {
    return show(result.value);
  }
  return `Error: ${result.error}`;
}

for (const text of ["3", "0", "two"]) {
  console.log(showResult(parseQuantity(text), (quantity) => `${quantity} x Keyboard = Rs. ${quantity * 1499}`));
}

for (const text of [" festive20 ", "20%OFF"]) {
  console.log(showResult(parseCoupon(text), (code) => `Coupon ${code} applied`));
}
```

```bash
npx tsc --noEmit
node result.ts
```

```text
3 x Keyboard = Rs. 4497
Error: "0" is not a quantity
Error: "two" is not a quantity
Coupon FESTIVE20 applied
Error: "20%OFF" is not a coupon code
```

| Code                                   | From this session and earlier                                  |
| -------------------------------------- | -------------------------------------------------------------- |
| `type Result<T> = { ok: true; ... } \| { ok: false; ... }` | A generic discriminated union                 |
| `Result<number>`, `Result<string>`     | One type for both parsers                                      |
| `if (result.ok)`                       | Narrowing on the `ok` property (Session 10)                    |
| `showResult<T>(result: Result<T>, show: (value: T) => string)` | `T` connects the result and the callback |
| `(quantity) => ...`                    | `quantity` is a `number`: `T` came from `parseQuantity`        |

The `Result` forces every caller to look at `ok` before using `value`

```typescript
const quantity = parseQuantity("3");
console.log(quantity.value * 1499);
```

```text
result.ts:34:22 - error TS2339: Property 'value' does not exist on type 'Result<number>'.
  Property 'value' does not exist on type '{ ok: false; error: string; }'.

34 console.log(quantity.value * 1499);
                        ~~~~~


Found 1 error in result.ts:34
```

And the callback gets the right type from `T`

```typescript
console.log(showResult(parseCoupon("FESTIVE20"), (code) => `${Math.round(code)}`));
```

```text
result.ts:33:74 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

33 console.log(showResult(parseCoupon("FESTIVE20"), (code) => `${Math.round(code)}`));
                                                                            ~~~~


Found 1 error in result.ts:33
```

`code` is a `string`, because `parseCoupon` returns a `Result<string>`. A thrown error is invisible in a function's type; a `Result` is part of it, so `tsc` makes sure every error is handled.

![Three texts go into parseQuantity and come out as Result<number>: "3" in the ok box with value 3, "0" and "two" in the error box; showResult opens each box the right way](images/14-generics/result-type.gif)

---

## Beginner Mistakes

### Mistake 1: Using any to make a function work for every type

`firstItem(items: any[]): any` lost the types, and `price.toUpperCase()` crashed at run time. A type parameter keeps them: `firstItem<T>(items: T[]): T | undefined`.

---

### Mistake 2: Using a property of T without a constraint

`item.id` in `findById<T>` gives `error TS2339: Property 'id' does not exist on type 'T'.` Add a constraint: `<T extends { id: number }>`.

---

### Mistake 3: Using the constraint type instead of a type parameter

`findById(items: { id: number }[])` returns only `{ id: number }`, and `.name` gives `error TS2339`. With `T extends { id: number }`, the result keeps the full type.

---

### Mistake 4: Forgetting the type argument when there is nothing to infer from

`emptyList()` gave `unknown[]`, and `createStore()` gave a store of plain `{ id: number }`. Write `emptyList<string>()` and `createStore<Product>()`.

---

### Mistake 5: Leaving out the type argument of a generic type

`const response: ApiResponse = ...` gives `error TS2314: Generic type 'ApiResponse<T>' requires 1 type argument(s).` Write `ApiResponse<Product>`, or give the parameter a default.

---

### Mistake 6: A type parameter that appears only once

`function printValue<T>(value: T): void` is the same as `value: unknown`, just harder to read. Use a generic only when `T` connects two places.

---

### Mistake 7: Using any as a default type argument

`interface ApiResponse<T = any>` turns off the checks for every response without a type argument. Use `T = unknown`.

---

### Mistake 8: Writing the type argument when tsc can infer it

`firstItem<number>([149900, 49900])` is correct but noisy. Let `tsc` infer it from the arguments, and write it only when needed.

---

### Mistake 9: Using a Result value without checking ok

`parseQuantity("3").value` gives `error TS2339: Property 'value' does not exist on type 'Result<number>'.` Check `result.ok` first, and `tsc` narrows to the right member.

---

### Mistake 10: Giving type parameters confusing names

`function convert<Product, Order>(...)` looks like it uses your interfaces, but `Product` and `Order` are new type parameters that hide them. Use `T`, `U` or names like `TItem`.

---

## Practice Exercises

### Exercise 1

Write `lastItem<T>(items: T[]): T | undefined` with `items.at(-1)`. Call it with numbers, texts and an empty array, and point at each result in VS Code.

### Exercise 2

Write `wrapInArray<T>(value: T | T[]): T[]`, which returns the array unchanged and wraps a single value in an array. Use `Array.isArray` (Session 10). What is `T` for `wrapInArray("sale")` and for `wrapInArray(["sale", "new"])`?

### Exercise 3

Write `pluckNames<T extends { name: string }>(items: T[]): string[]`. Call it with products and with customers from Mini Example 1. Then try it with `[{ title: "Notes" }]`.

### Exercise 4

Write `interface Paginated<T>` (as above) and a function `paginate<T>(items: T[], page: number, pageSize: number): Paginated<T>` that uses `slice`. Paginate the numbers 1 to 25 with a page size of 10, and print all three pages.

### Exercise 5

Write `countBy<T>(items: T[], keyOf: (item: T) => string): Map<string, number>`, and count orders by status: `countBy(orders, (order) => order.status)`. Point at `order` in the callback: which type does it have, and where did it come from?

### Exercise 6

Add `remove(id: number): boolean` to `Store<T>` in Mini Example 1, and implement it with `findIndex` and `splice`. Then add `update(item: T): boolean`, which replaces the item with the same `id`.

### Exercise 7

In Mini Example 2, write `parseEmail(text: string): Result<string>` (it must contain one `@`) and use `showResult` with it. Then write `mapResult<T, U>(result: Result<T>, change: (value: T) => U): Result<U>`, and turn a `Result<number>` quantity into a `Result<string>` label.

### Exercise 8

Write `type ApiResponse<T = unknown>` and a function `isOk<T>(response: ApiResponse<T>): boolean` that checks `status < 400`. Why does this function not need `T` at all? Rewrite it without the type parameter.

---

## Interview Questions

### What are generics?

A way to write functions and types with type parameters, placeholders that are filled in with a real type at each use. They let one function or type work with many types while keeping the exact type, instead of falling back to `any`.

### What is the difference between a type parameter and a type argument?

The type parameter is the placeholder in the definition, like `T` in `function firstItem<T>`. The type argument is the real type used for it at a call or use, like `string` in `firstItem<string>(...)` or `Array<string>`.

### How does TypeScript know what T is?

It infers it from the arguments of the call. `firstItem([1, 2])` makes `T` `number`. When there are no arguments to infer from, `T` becomes `unknown` (or the constraint), and you write the type argument yourself.

### What is a generic constraint?

A limit on a type parameter, written with `extends`: `<T extends { id: number }>` accepts only types that have a numeric `id`. Inside the function you can use `id`; outside, the full type is kept.

### Why not just use any instead of a generic?

`any` turns off checking for the value and everything it touches, and the result loses its type. A generic keeps the exact type, so mistakes in the caller are still caught.

### Why not use unknown instead of a generic?

`unknown` is safe, but the caller gets `unknown` back and must check it again. A generic connects input and output: what goes in as a `number` comes out as a `number`.

### Can interfaces and type aliases be generic?

Yes: `interface ApiResponse<T> { data: T }` and `type Result<T> = ...`. They are used with a type argument, like `ApiResponse<Product>`, and can be nested.

### What is a default type argument?

A type used when no argument is given: `interface ApiResponse<T = unknown>` makes `ApiResponse` mean `ApiResponse<unknown>`.

### Which built-in types are generic?

`Array<T>`, `ReadonlyArray<T>`, `Map<K, V>`, `Set<T>`, `Promise<T>`, and many more. Array methods like `map<U>` and `filter` are generic functions.

### When should you not use a generic?

When the type parameter appears only once, for example only in a parameter. Then it connects nothing, and `unknown` or the real type is simpler.

### What is a Result type?

A generic discriminated union like `{ ok: true; value: T } | { ok: false; error: string }`. Functions return it instead of throwing, and callers must check `ok` before they can read `value`.

### What does TS2344 mean?

"Type X does not satisfy the constraint Y": a type argument was given that does not meet the type parameter's `extends` constraint, like `createStore<string>()` when `T` must have an `id`.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [13. Type Assertions, as const and satisfies](13-assertions-as-const-and-satisfies.md) | [Course outline](README.md#course-outline) | [15. keyof, typeof and Indexed Access](15-keyof-typeof-indexed-access.md) |
