# Typing Functions

> **Session 12 of 33** | Part 3: Functions | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Parameters](#parameters)
- [Return Types: Inferred or Written](#return-types-inferred-or-written)
- [When to Write the Return Type](#when-to-write-the-return-type)
- [Functions That Return Nothing: void](#functions-that-return-nothing-void)
- [Optional Parameters](#optional-parameters)
- [Default Parameters](#default-parameters)
- [Rest Parameters](#rest-parameters)
- [An Object as the Parameter](#an-object-as-the-parameter)
- [Function Types](#function-types)
- [Callbacks](#callbacks)
- [Callbacks That Return void](#callbacks-that-return-void)
- [Methods in Interfaces](#methods-in-interfaces)
- [Overloads](#overloads)
- [Special Return Types](#special-return-types)
- [Mini Example 1: A Pricing Pipeline](#mini-example-1-a-pricing-pipeline)
- [Mini Example 2: A Tiny Router](#mini-example-2-a-tiny-router)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `12-functions` inside `ts-practice`, with the one-line `tsconfig.json`

12-functions/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## Parameters

You have typed parameters since Session 03, and `strict` insists on it

line-total.ts

```typescript
function lineTotal(item) {
  return item.pricePaise * item.quantity;
}
```

```text
line-total.ts:1:20 - error TS7006: Parameter 'item' implicitly has an 'any' type.

1 function lineTotal(item) {
                     ~~~~


Found 1 error in line-total.ts:1
```

The types also fix the **number** of arguments. JavaScript does not care

arity.js

```javascript
function formatRupees(paise) {
  return `Rs. ${(paise / 100).toFixed(2)}`;
}

console.log(formatRupees(149900, 2));
console.log(formatRupees());
```

```text
Rs. 1499.00
Rs. NaN
```

The extra `2` was silently ignored, and the missing argument became `undefined`, then `NaN`. TypeScript checks both

arity.ts

```typescript
function formatRupees(paise: number): string {
  return `Rs. ${(paise / 100).toFixed(2)}`;
}

console.log(formatRupees(149900));
console.log(formatRupees());
console.log(formatRupees(149900, 2));
```

```text
arity.ts:6:13 - error TS2554: Expected 1 arguments, but got 0.

6 console.log(formatRupees());
              ~~~~~~~~~~~~

  arity.ts:1:23 - An argument for 'paise' was not provided.
    1 function formatRupees(paise: number): string {
                            ~~~~~~~~~~~~~

arity.ts:7:34 - error TS2554: Expected 1 arguments, but got 2.

7 console.log(formatRupees(149900, 2));
                                   ~


Found 2 errors in the same file, starting at: arity.ts:6
```

Every parameter is required unless you say otherwise. The rest of this session shows how to say otherwise.

![formatRupees has one slot: one argument fits, zero leaves the slot empty (TS2554), two leaves one argument with no slot (TS2554); JavaScript fills the empty slot with undefined and prints Rs. NaN](images/12-typing-functions/argument-count.gif)

---

## Return Types: Inferred or Written

`tsc` works out the return type from the `return` lines. Sometimes that hides a mistake

shipping.ts

```typescript
function shippingPaise(weightGrams: number) {
  if (weightGrams <= 0) {
    return "free";
  }
  return 4000 + Math.ceil(weightGrams / 500) * 2000;
}

const total = 149900 + shippingPaise(900);
console.log(total);
```

```text
shipping.ts:8:15 - error TS2365: Operator '+' cannot be applied to types 'number' and 'string | number'.

8 const total = 149900 + shippingPaise(900);
                ~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in shipping.ts:8
```

Point at `shippingPaise` in VS Code: the return type is `number | "free"`, made from both `return` lines. The real mistake is `return "free"` on line 3, but the error appears where the function is **used**, line 8. In a project, that could be another file, written by someone else.

Write the return type you mean

shipping.ts

```typescript
function shippingPaise(weightGrams: number): number {
  if (weightGrams <= 0) {
    return "free";
  }
  return 4000 + Math.ceil(weightGrams / 500) * 2000;
}

const total = 149900 + shippingPaise(900);
console.log(total);
```

```text
shipping.ts:3:5 - error TS2322: Type 'string' is not assignable to type 'number'.

3     return "free";
      ~~~~~~


Found 1 error in shipping.ts:3
```

Now the error points to the line that is wrong, and line 8 is fine: callers can trust that `shippingPaise` returns a number. The fix is `return 0;`.

![Without a return type, the wrong return "free" flows out of the function and the error appears at the caller; with : number, the error stays inside the function, on the wrong line](images/12-typing-functions/return-annotation.gif)

---

## When to Write the Return Type

One more case where `tsc` needs it: a function that calls itself in its `return`

count2.ts

```typescript
interface Category {
  name: string;
  children: Category[];
}

function countCategories(category: Category) {
  return 1 + category.children.reduce((sum, child) => sum + countCategories(child), 0);
}
```

```text
count2.ts:6:10 - error TS7023: 'countCategories' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.

6 function countCategories(category: Category) {
           ~~~~~~~~~~~~~~~

count2.ts:7:39 - error TS7024: Function implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.

7   return 1 + category.children.reduce((sum, child) => sum + countCategories(child), 0);
                                        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: count2.ts:6
```

To know the return type, `tsc` would need the return type first. With `: number` the circle is broken

count.ts

```typescript
interface Category {
  name: string;
  children: Category[];
}

function countCategories(category: Category): number {
  return 1 + category.children.reduce((sum, child) => sum + countCategories(child), 0);
}

const electronics: Category = {
  name: "Electronics",
  children: [
    { name: "Keyboards", children: [] },
    { name: "Monitors", children: [{ name: "Gaming monitors", children: [] }] }
  ]
};

console.log(countCategories(electronics));
```

```text
4
```

| Function                                         | Write the return type?                         |
| ------------------------------------------------ | ---------------------------------------------- |
| Exported, used by other files                    | Yes: it is the promise the function makes      |
| Several `return` lines                           | Yes: a wrong one is reported where it is       |
| Calls itself in a `return`                       | Yes: `tsc` may need it (TS7023)                |
| A short callback like `(item) => item.pricePaise` | No: inferred, and the caller checks it       |

From here on, the course writes the return type on every named function, and leaves it out on short callbacks.

---

## Functions That Return Nothing: void

A function without a useful result returns `void` (Session 04). Returning a value anyway is an error

void-return.ts

```typescript
function logOrder(id: number): void {
  console.log(`Order ${id}`);
  return id;
}
```

```text
void-return.ts:3:3 - error TS2322: Type 'number' is not assignable to type 'void'.

3   return id;
    ~~~~~~


Found 1 error in void-return.ts:3
```

---

## Optional Parameters

A `?` after the name makes a parameter optional. Inside the function it is `T | undefined`

greeting.ts

```typescript
function greeting(customer: string, city?: string): string {
  if (city === undefined) {
    return `Hello ${customer}`;
  }
  return `Hello ${customer} from ${city}`;
}

console.log(greeting("Riya"));
console.log(greeting("Sara", "Mumbai"));
```

```text
Hello Riya
Hello Sara from Mumbai
```

Without the check, it is Session 11's error

greeting-body.ts

```typescript
function greeting(customer: string, city?: string): string {
  return `Hello ${customer} from ${city.toUpperCase()}`;
}
```

```text
greeting-body.ts:2:36 - error TS18048: 'city' is possibly 'undefined'.

2   return `Hello ${customer} from ${city.toUpperCase()}`;
                                     ~~~~


Found 1 error in greeting-body.ts:2
```

Optional parameters must come after the required ones, because arguments are matched by position

greeting-order.ts

```typescript
function greeting(city?: string, customer: string): string {
  return `Hello ${customer} from ${city}`;
}
```

```text
greeting-order.ts:1:34 - error TS1016: A required parameter cannot follow an optional parameter.

1 function greeting(city?: string, customer: string): string {
                                   ~~~~~~~~


Found 1 error in greeting-order.ts:1
```

---

## Default Parameters

A default value makes a parameter optional for the caller, and gives it a type without an annotation

line-total.ts

```typescript
function lineTotal(pricePaise: number, quantity = 1): number {
  return pricePaise * quantity;
}

console.log(lineTotal(49900));
console.log(lineTotal(49900, 3));
console.log(lineTotal(49900, undefined));
```

```text
49900
149700
49900
```

| Where                      | Type of `quantity`                    |
| -------------------------- | ------------------------------------- |
| For the caller             | `quantity?: number`: may be left out  |
| Inside the function        | `number`: never `undefined`           |

So no check is needed inside: the default already handled `undefined`. As Session 11 showed for destructuring, a default replaces only `undefined`, not `null`. TypeScript keeps you from passing `null`

line-total.ts

```typescript
function lineTotal(pricePaise: number, quantity = 1): number {
  return pricePaise * quantity;
}

console.log(lineTotal(49900, null));
console.log(lineTotal(49900, "3"));
```

```text
line-total.ts:5:30 - error TS2345: Argument of type 'null' is not assignable to parameter of type 'number | undefined'.

5 console.log(lineTotal(49900, null));
                               ~~~~

line-total.ts:6:30 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

6 console.log(lineTotal(49900, "3"));
                               ~~~


Found 2 errors in the same file, starting at: line-total.ts:5
```

The first message shows how `tsc` sees the parameter from outside: `number | undefined`. `undefined` is allowed (it means "use the default"), `null` is not.

![Two parameters side by side: city? is a slot that may stay empty, so inside it is string | undefined; quantity = 1 is a slot that fills itself with 1, so inside it is always a number](images/12-typing-functions/optional-default.gif)

---

## Rest Parameters

`...` collects any number of arguments into an array, and the type is the array type

cart-total.ts

```typescript
function cartTotal(...pricesPaise: number[]): number {
  return pricesPaise.reduce((sum, price) => sum + price, 0);
}

console.log(cartTotal(149900, 49900, 29900));
console.log(cartTotal());

const prices = [149900, 49900];
console.log(cartTotal(...prices));
```

```text
229700
0
199800
```

Each argument is checked against the item type

```typescript
console.log(cartTotal(149900, "49900"));
```

```text
cart-total.ts:5:31 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

5 console.log(cartTotal(149900, "49900"));
                                ~~~~~~~


Found 1 error in cart-total.ts:5
```

A rest parameter must be the last one: `function log(level: LogLevel, ...messages: string[])`.

![Three separate arguments 149900, 49900 and 29900 go into cartTotal and are collected into one array pricesPaise: number[]; with no arguments the array is empty and the total is 0](images/12-typing-functions/rest-params.gif)

---

## An Object as the Parameter

When a function has many optional settings, a list of parameters gets confusing: what was the third argument again? Pass one object instead, and destructure it with defaults

list-products.ts

```typescript
interface ListOptions {
  page?: number;
  pageSize?: number;
  sortBy?: "price" | "name";
}

function listProducts({ page = 1, pageSize = 20, sortBy = "name" }: ListOptions = {}): string {
  return `page ${page}, ${pageSize} per page, sorted by ${sortBy}`;
}

console.log(listProducts());
console.log(listProducts({ page: 3 }));
console.log(listProducts({ sortBy: "price", pageSize: 50 }));
```

```text
page 1, 20 per page, sorted by name
page 3, 20 per page, sorted by name
page 1, 50 per page, sorted by price
```

| Part                                   | Meaning                                         |
| -------------------------------------- | ----------------------------------------------- |
| `{ page = 1, pageSize = 20, ... }`     | Destructure, with a default for each property   |
| `: ListOptions`                        | The type of the whole object, after the `}`     |
| `= {}`                                 | The whole argument may be left out              |

The type goes after the whole pattern. Writing `{ page: number }` inside the braces would rename `page` to a variable called `number` (Session 07). The object is checked like any object

```typescript
console.log(listProducts({ sortBy: "date" }));
console.log(listProducts({ pageSize: 50, limit: 10 }));
```

```text
list-products.ts:11:28 - error TS2322: Type '"date"' is not assignable to type '"name" | "price" | undefined'.

11 console.log(listProducts({ sortBy: "date" }));
                              ~~~~~~

list-products.ts:12:42 - error TS2353: Object literal may only specify known properties, and 'limit' does not exist in type 'ListOptions'.

12 console.log(listProducts({ pageSize: 50, limit: 10 }));
                                            ~~~~~


Found 2 errors in the same file, starting at: list-products.ts:11
```

Leave out the `= {}`, and calling with no argument is an error

```text
list-products.ts:11:13 - error TS2554: Expected 1 arguments, but got 0.

11 console.log(listProducts());
               ~~~~~~~~~~~~

  list-products.ts:7:23 - An argument matching this binding pattern was not provided.
    7 function listProducts({ page = 1, pageSize = 20, sortBy = "name" }: ListOptions): string {
                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in list-products.ts:11
```

This "options object" pattern is everywhere in Node.js libraries: `fs.readFileSync(path, { encoding: "utf8" })`, `express.json({ limit: "1mb" })`, Mongoose queries.

![An options object { page: 3 } goes into listProducts; the missing pageSize and sortBy are filled in from the defaults, and page 3, 20 per page, sorted by name comes out](images/12-typing-functions/options-object.gif)

---

## Function Types

Functions are values in JavaScript: you store them in variables, put them in arrays, and pass them to other functions. So they need a type. A **function type** looks like an arrow function without a body

price-rule.ts

```typescript
type PriceRule = (pricePaise: number) => number;

const tenPercentOff: PriceRule = (pricePaise) => Math.round(pricePaise * 0.9);
const flat500Off: PriceRule = (price) => price - 50000;

console.log(tenPercentOff(149900));
console.log(flat500Off(149900));
```

```text
134910
99900
```

`(pricePaise: number) => number` reads: "takes a number, returns a number". The parameter name in the type is only documentation: `flat500Off` calls it `price`, and that is fine. Neither arrow function needs a type on its parameter: `tsc` takes it from `PriceRule`.

A function that does not fit

price-rule.ts

```typescript
type PriceRule = (pricePaise: number) => number;

const label: PriceRule = (pricePaise) => `Rs. ${pricePaise / 100}`;
const twoArgs: PriceRule = (pricePaise: number, percent: number) => pricePaise - percent;
```

```text
price-rule.ts:3:42 - error TS2322: Type 'string' is not assignable to type 'number'.

3 const label: PriceRule = (pricePaise) => `Rs. ${pricePaise / 100}`;
                                           ~~~~~~~~~~~~~~~~~~~~~~~~~

  price-rule.ts:1:18 - The expected type comes from the return type of this signature.
    1 type PriceRule = (pricePaise: number) => number;
                       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

price-rule.ts:4:7 - error TS2322: Type '(pricePaise: number, percent: number) => number' is not assignable to type 'PriceRule'.
  Target signature provides too few arguments. Expected 2 or more, but got 1.

4 const twoArgs: PriceRule = (pricePaise: number, percent: number) => pricePaise - percent;
        ~~~~~~~


Found 2 errors in the same file, starting at: price-rule.ts:3
```

The first returns a string. The second needs two arguments, but whoever uses a `PriceRule` will call it with one, and `percent` would be `undefined`.

Fewer parameters are fine, though: a function that ignores its argument, like `() => 0`, still fits `PriceRule`. Extra arguments are simply not used.

![PriceRule as a socket: (pricePaise) => number plugs in; a function that returns a string or wants two arguments does not fit; a function with no parameters fits too](images/12-typing-functions/function-type.gif)

---

## Callbacks

A function that takes another function as a parameter writes the function type right there

find-first.ts

```typescript
function findFirst(pricesPaise: number[], test: (pricePaise: number) => boolean): number | undefined {
  for (const price of pricesPaise) {
    if (test(price)) {
      return price;
    }
  }
  return undefined;
}

const prices = [49900, 149900, 1299900];
console.log(findFirst(prices, (price) => price > 100000));
console.log(findFirst(prices, (price) => price > 5000000));
console.log(findFirst(prices, () => true));
```

```text
149900
undefined
49900
```

In `(price) => price > 100000`, `price` has no annotation, yet it is a `number`: `tsc` knows where the arrow function goes, and takes the parameter type from there. This is called **contextual typing**, and it is why `orders.map((order) => order.id)` never needed a type on `order`. Without a context, the parameter is `any` again

no-context.ts

```typescript
const isExpensive = (price) => price > 100000;
```

```text
no-context.ts:1:22 - error TS7006: Parameter 'price' implicitly has an 'any' type.

1 const isExpensive = (price) => price > 100000;
                       ~~~~~


Found 1 error in no-context.ts:1
```

Fix it with `(price: number) =>`, or with a function type on the variable.

The callback is checked like any function

```typescript
console.log(findFirst(prices, (price: number, limit: number) => price > limit));
console.log(findFirst(prices, (price) => price.toUpperCase()));
```

```text
find-first.ts:11:31 - error TS2345: Argument of type '(price: number, limit: number) => boolean' is not assignable to parameter of type '(pricePaise: number) => boolean'.
  Target signature provides too few arguments. Expected 2 or more, but got 1.

11 console.log(findFirst(prices, (price: number, limit: number) => price > limit));
                                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

find-first.ts:12:48 - error TS2339: Property 'toUpperCase' does not exist on type 'number'.

12 console.log(findFirst(prices, (price) => price.toUpperCase()));
                                                  ~~~~~~~~~~~


Found 2 errors in the same file, starting at: find-first.ts:11
```

`findFirst` calls `test` with one argument, so a callback that needs `limit` would get `undefined`.

![findFirst's parameter type (pricePaise: number) => boolean flows into the arrow function passed in, so price becomes a number without an annotation](images/12-typing-functions/contextual-typing.gif)

---

## Callbacks That Return void

A callback type with `void` means "the caller ignores what you return". So a callback that returns something is fine

void-callback.ts

```typescript
const names: string[] = [];
["Riya", "Sara"].forEach((customer) => names.push(customer));
console.log(names);
```

```text
[ 'Riya', 'Sara' ]
```

`names.push` returns the new length, a number, and `forEach` expects a callback that returns `void`. No error: `forEach` throws the number away. A **function you write** with `: void`, like `logOrder` above, may not return a value: there, `void` is a promise about your own code.

---

## Methods in Interfaces

An object with functions describes them as methods in its interface

cart.ts

```typescript
interface Cart {
  items: number[];
  add(pricePaise: number): void;
  total(): number;
}

const cart: Cart = {
  items: [],
  add(pricePaise) {
    this.items.push(pricePaise);
  },
  total() {
    return this.items.reduce((sum, price) => sum + price, 0);
  }
};

cart.add(149900);
cart.add(49900);
console.log(cart.total());
```

```text
199800
```

`add(pricePaise: number): void` is the method's parameter list and return type. In the object, `pricePaise` needs no annotation (contextual typing again), and `this` is typed as in Session 07. Both sides are checked: make `total` return a text, and replace the three lines at the end with a call that passes a text

```typescript
  total() {
    return `${this.items.length} items`;
  }
};

cart.add("49900");
```

```text
cart.ts:12:3 - error TS2322: Type '() => string' is not assignable to type '() => number'.
  Type 'string' is not assignable to type 'number'.

12   total() {
     ~~~~~

  cart.ts:4:3 - The expected type comes from property 'total' which is declared here on type 'Cart'
    4   total(): number;
        ~~~~~~~~~~~~~~~~

cart.ts:17:10 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

17 cart.add("49900");
            ~~~~~~~


Found 2 errors in the same file, starting at: cart.ts:12
```

You may also see the property style, `add: (pricePaise: number) => void;`. For everyday code both mean the same. The method style reads more naturally for objects with behaviour, and Session 20's classes use it too.

---

## Overloads

Sometimes the return type depends on the argument: one number in, one number out; an array in, an array out. A union type cannot say that

to-paise-union.ts

```typescript
function toPaise(rupees: number | number[]): number | number[] {
  if (Array.isArray(rupees)) {
    return rupees.map((amount) => Math.round(amount * 100));
  }
  return Math.round(rupees * 100);
}

const one = toPaise(1499);
console.log(one.toFixed(0));
```

```text
to-paise-union.ts:9:17 - error TS2339: Property 'toFixed' does not exist on type 'number | number[]'.
  Property 'toFixed' does not exist on type 'number[]'.

9 console.log(one.toFixed(0));
                  ~~~~~~~


Found 1 error in to-paise-union.ts:9
```

We passed one number, but `tsc` only knows "a number or an array". **Overloads** list each way to call the function, above the real one

to-paise.ts

```typescript
function toPaise(rupees: number): number;
function toPaise(rupees: number[]): number[];
function toPaise(rupees: number | number[]): number | number[] {
  if (Array.isArray(rupees)) {
    return rupees.map((amount) => Math.round(amount * 100));
  }
  return Math.round(rupees * 100);
}

const one = toPaise(1499);
const many = toPaise([1499, 499.5]);
console.log(one.toFixed(0));
console.log(many.join(", "));
```

```text
149900
149900, 49950
```

| Line                                                       | Role                                           |
| ---------------------------------------------------------- | ---------------------------------------------- |
| `function toPaise(rupees: number): number;`                | Overload 1: what callers see                   |
| `function toPaise(rupees: number[]): number[];`            | Overload 2: what callers see                   |
| `function toPaise(rupees: number \| number[]): number \| number[] { ... }` | The implementation, hidden from callers |

`one` is a `number`, `many` a `number[]`. A call that matches no overload is TS2769, which you met in Session 05

```typescript
console.log(toPaise("1499"));
```

```text
to-paise.ts:10:21 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Argument of type 'string' is not assignable to parameter of type 'number[]'.

10 console.log(toPaise("1499"));
                       ~~~~~~

  to-paise.ts:2:10 - The last overload is declared here.
    2 function toPaise(rupees: number[]): number[];
               ~~~~~~~


Found 1 error in to-paise.ts:10
```

You will meet overloads mostly in library types: `JSON.stringify` and `toLocaleDateString` in earlier sessions had them. In your own code, two simple functions (`toPaise` and `toPaiseList`) are often clearer, and generics (Session 14) solve many of the same problems.

![toPaise with two overloads: a number goes in through the first door and a number comes out, an array goes through the second and an array comes out; a string fits neither door and gets TS2769](images/12-typing-functions/overloads.gif)

---

## Special Return Types

| Return type       | Means                                          | Session |
| ----------------- | ---------------------------------------------- | ------- |
| `void`            | No useful result                               | 04      |
| `never`           | Never returns: always throws                   | 04      |
| `value is T`      | A type guard: `true` means the value is a `T`  | 10      |
| `T \| undefined`  | May not find anything                          | 11      |
| `Promise<T>`      | An `async` function that gives a `T` later     | 23      |

---

## Mini Example 1: A Pricing Pipeline

A festive sale has several rules: 20% off, then Rs. 100 off, but never below Rs. 399. Each rule is a function, made by a function

pricing.ts

```typescript
type PriceRule = (pricePaise: number) => number;

function percentOff(percent: number): PriceRule {
  return (pricePaise) => Math.round(pricePaise * (1 - percent / 100));
}

function flatOff(amountPaise: number): PriceRule {
  return (pricePaise) => pricePaise - amountPaise;
}

function atLeast(minimumPaise: number): PriceRule {
  return (pricePaise) => Math.max(pricePaise, minimumPaise);
}

function applyRules(pricePaise: number, ...rules: PriceRule[]): number {
  return rules.reduce((price, rule) => rule(price), pricePaise);
}

function formatRupees(paise: number): string {
  return `Rs. ${(paise / 100).toFixed(2)}`;
}

const festiveSale: PriceRule[] = [percentOff(20), flatOff(10000), atLeast(39900)];

const products = [
  { name: "Keyboard", pricePaise: 149900 },
  { name: "Mouse", pricePaise: 49900 },
  { name: "Monitor", pricePaise: 1299900 }
];

for (const product of products) {
  const finalPrice = applyRules(product.pricePaise, ...festiveSale);
  console.log(`${product.name.padEnd(9)} ${formatRupees(product.pricePaise).padStart(12)} -> ${formatRupees(finalPrice)}`);
}

console.log(`No rules: ${formatRupees(applyRules(49900))}`);
```

```bash
npx tsc --noEmit
node pricing.ts
```

```text
Keyboard   Rs. 1499.00 -> Rs. 1099.20
Mouse       Rs. 499.00 -> Rs. 399.00
Monitor   Rs. 12999.00 -> Rs. 10299.20
No rules: Rs. 499.00
```

The mouse would have dropped to Rs. 299.20, so `atLeast` lifted it to Rs. 399.

| Code                                         | From this session                                       |
| -------------------------------------------- | ------------------------------------------------------- |
| `type PriceRule = (pricePaise: number) => number` | A function type with a name                        |
| `function percentOff(percent: number): PriceRule` | A function that returns a function                 |
| `return (pricePaise) => ...`                 | Contextual typing: `pricePaise` is a `number` from `PriceRule` |
| `...rules: PriceRule[]`                      | A rest parameter of functions                           |
| `applyRules(product.pricePaise, ...festiveSale)` | Spreading an array into the rest parameter          |
| `rules.reduce((price, rule) => rule(price), pricePaise)` | Calling each function in turn               |

The returned function still knows `percent` after `percentOff` has finished. That is a closure, from the JavaScript course, and its type is simply `PriceRule`.

![percentOff(20) is a small factory: it takes 20 and hands out a new PriceRule that remembers percent = 20; flatOff(10000) and atLeast(39900) hand out two more rules](images/12-typing-functions/rule-factory.gif)

Try two mistakes. A "rule" that formats instead of calculating

```typescript
const showPrice: PriceRule = (pricePaise) => formatRupees(pricePaise);
```

```text
pricing.ts:37:46 - error TS2322: Type 'string' is not assignable to type 'number'.

37 const showPrice: PriceRule = (pricePaise) => formatRupees(pricePaise);
                                                ~~~~~~~~~~~~~~~~~~~~~~~~

  pricing.ts:1:18 - The expected type comes from the return type of this signature.
    1 type PriceRule = (pricePaise: number) => number;
                       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in pricing.ts:37
```

And forgetting to call the factory

```typescript
console.log(applyRules(49900, percentOff));
```

```text
pricing.ts:37:31 - error TS2345: Argument of type '(percent: number) => PriceRule' is not assignable to parameter of type 'PriceRule'.
  Type 'PriceRule' is not assignable to type 'number'.

37 console.log(applyRules(49900, percentOff));
                                 ~~~~~~~~~~

  pricing.ts:37:31 - Did you mean to call this expression?
    37 console.log(applyRules(49900, percentOff));
                                     ~~~~~~~~~~


Found 1 error in pricing.ts:37
```

`percentOff` makes a rule, it is not one. `tsc` even asks: "Did you mean to call this expression?" The fix is `percentOff(10)`.

![The keyboard price 1499 goes through three rules in a row: 20% off makes 1199.20, Rs. 100 off makes 1099.20, at least 399 leaves it; the mouse goes through the same rules and atLeast lifts 299.20 to 399](images/12-typing-functions/pricing-pipeline.gif)

---

## Mini Example 2: A Tiny Router

A web framework like Express matches each request to a function, a "handler". Here is the idea in 50 lines, with types for everything

router.ts

```typescript
type Method = "GET" | "POST";

interface HttpRequest {
  method: Method;
  path: string;
  body?: string;
}

interface HttpResponse {
  status: number;
  body: string;
}

type Handler = (request: HttpRequest) => HttpResponse;

interface Route {
  method: Method;
  path: string;
  handler: Handler;
}

const routes: Route[] = [];

function addRoute(method: Method, path: string, handler: Handler): void {
  routes.push({ method, path, handler });
}

function handle(request: HttpRequest): HttpResponse {
  const route = routes.find((item) => item.method === request.method && item.path === request.path);
  if (route === undefined) {
    return { status: 404, body: `Cannot ${request.method} ${request.path}` };
  }
  return route.handler(request);
}

addRoute("GET", "/health", () => ({ status: 200, body: "ok" }));
addRoute("GET", "/orders", () => ({ status: 200, body: "1001, 1002" }));
addRoute("POST", "/orders", (request) => ({ status: 201, body: `created order: ${request.body ?? "empty"}` }));

const requests: HttpRequest[] = [
  { method: "GET", path: "/health" },
  { method: "GET", path: "/orders" },
  { method: "POST", path: "/orders", body: "2 keyboards" },
  { method: "GET", path: "/users" }
];

for (const request of requests) {
  const response = handle(request);
  console.log(`${request.method} ${request.path} -> ${response.status} ${response.body}`);
}
```

```bash
npx tsc --noEmit
node router.ts
```

```text
GET /health -> 200 ok
GET /orders -> 200 1001, 1002
POST /orders -> 201 created order: 2 keyboards
GET /users -> 404 Cannot GET /users
```

(Express answers an unknown route with the same kind of text: `Cannot GET /users`.)

| Code                                           | From this session                                     |
| ---------------------------------------------- | ----------------------------------------------------- |
| `type Handler = (request: HttpRequest) => HttpResponse` | A function type with interfaces                |
| `handler: Handler` in `Route`                  | A function stored in an object                        |
| `() => ({ status: 200, body: "ok" })`          | Fewer parameters than `Handler`: fine                 |
| `(request) => ...`                             | `request` is an `HttpRequest` from the context        |
| `route.handler(request)`                       | Calling a function from an object                     |

(The names are `HttpRequest` and `HttpResponse` because `Request` and `Response` already exist as global types for `fetch`.)

Now break three things, one at a time, by adding a route at the end. A status as text

```typescript
addRoute("GET", "/stock", () => ({ status: "200", body: "12 keyboards" }));
```

```text
router.ts:51:36 - error TS2322: Type 'string' is not assignable to type 'number'.

51 addRoute("GET", "/stock", () => ({ status: "200", body: "12 keyboards" }));
                                      ~~~~~~

  router.ts:10:3 - The expected type comes from property 'status' which is declared here on type 'HttpResponse'
    10   status: number;
         ~~~~~~


Found 1 error in router.ts:51
```

A method the router does not support

```typescript
addRoute("DELETE", "/orders", () => ({ status: 204, body: "" }));
```

```text
router.ts:51:10 - error TS2345: Argument of type '"DELETE"' is not assignable to parameter of type 'Method'.

51 addRoute("DELETE", "/orders", () => ({ status: 204, body: "" }));
            ~~~~~~~~


Found 1 error in router.ts:51
```

A handler that forgets the body may be missing

```typescript
addRoute("POST", "/users", (request) => ({ status: 201, body: request.body.toUpperCase() }));
```

```text
router.ts:51:63 - error TS18048: 'request.body' is possibly 'undefined'.

51 addRoute("POST", "/users", (request) => ({ status: 201, body: request.body.toUpperCase() }));
                                                                 ~~~~~~~~~~~~


Found 1 error in router.ts:51
```

All three errors appear inside the handler you pass, even though the handler has no type annotations at all: everything comes from `Handler`. Session 31 does the same with the real Express types.

![Four requests arrive at handle; three match a route and their handler returns a response, GET /users matches nothing and gets 404 Cannot GET /users](images/12-typing-functions/tiny-router.gif)

---

## Beginner Mistakes

### Mistake 1: A parameter without a type

`function lineTotal(item)` gives `error TS7006: Parameter 'item' implicitly has an 'any' type.` Every parameter of a named function needs a type.

---

### Mistake 2: The wrong number of arguments

`formatRupees()` gives `error TS2554: Expected 1 arguments, but got 0.` In JavaScript, the same call printed `Rs. NaN`. Make the parameter optional or give it a default if it really may be left out.

---

### Mistake 3: No return type on a function with several returns

Without `: number`, `return "free"` made the return type `number | "free"`, and the error appeared at the caller. Write the return type, and the error points to the wrong `return`.

---

### Mistake 4: A recursive function without a return type

`countCategories` gave `error TS7023` and `error TS7024`. Add `: number`.

---

### Mistake 5: An optional parameter before a required one

`function greeting(city?: string, customer: string)` gives `error TS1016: A required parameter cannot follow an optional parameter.` Put optional parameters last, or use an options object.

---

### Mistake 6: Using an optional parameter without a check

`city.toUpperCase()` with `city?: string` gives `error TS18048: 'city' is possibly 'undefined'.` Check it, or give it a default instead of `?`.

---

### Mistake 7: Passing null to get the default

`lineTotal(49900, null)` gives `error TS2345`, because a default replaces only `undefined`. Leave the argument out, or pass `undefined`.

---

### Mistake 8: Forgetting = {} on an options object

Without it, `listProducts()` gives `error TS2554` with `An argument matching this binding pattern was not provided.` Add `= {}` after the type.

---

### Mistake 9: Passing a function instead of calling it

`applyRules(49900, percentOff)` passes the factory, not a rule. `tsc` asks `Did you mean to call this expression?` Write `percentOff(10)`.

---

### Mistake 10: A callback that needs more arguments than it gets

`(price: number, limit: number) => price > limit` for a callback that is called with one argument gives `Target signature provides too few arguments.` The second argument would be `undefined`.

---

### Mistake 11: Using overloads where a union or two functions are simpler

Overloads add lines that must stay in sync with the implementation, and a union argument like `number | number[]` matches no overload (TS2769). Prefer two clearly named functions unless you are writing a library.

---

## Practice Exercises

### Exercise 1

Write `formatPhone(phone: string, countryCode = "+91"): string` that returns `+91 98765 43210`. Call it with one and with two arguments. Point at the function name in VS Code: how is `countryCode` shown?

### Exercise 2

Write `average(...values: number[]): number | undefined` that returns `undefined` when no values are given. Call it with `average()`, `average(4, 5)` and with an array spread into it.

### Exercise 3

Write `interface CreateOrderInput { customer: string; items: string[]; couponCode?: string }` and `createOrder(input: CreateOrderInput): string`, which returns `Order for Riya: 2 items, coupon none`. Destructure the input in the parameter list.

### Exercise 4

Write `type Validator = (value: string) => string | undefined`, where the result is an error message or `undefined`. Write `notEmpty`, `maxLength(limit: number): Validator` (a factory) and `hasAt` (checks for `@`). Run all three on `""`, `"riya"` and `"riya@example.com"`, and print every message.

### Exercise 5

Write your own `mapPrices(pricesPaise: number[], change: (pricePaise: number) => number): number[]` with a `for...of` loop. Pass it `percentOff(10)` from Mini Example 1, and an arrow function that adds 18% GST.

### Exercise 6

Add `remove(pricePaise: number): boolean` to the `Cart` interface, returning `true` when an item was removed. Implement it in `cart`, and try returning nothing from it first. What does `tsc` say?

### Exercise 7

In Mini Example 1, add `roundDown(): PriceRule`, which rounds down to whole rupees (`Math.floor(pricePaise / 100) * 100`), and add it as the last rule of the sale. Then write a rule with a mistake that returns `NaN` sometimes. Can `tsc` catch it?

### Exercise 8

In Mini Example 2, add `"DELETE"` to `Method` and a route `DELETE /orders` that returns status `204`. Then fix the `request.body.toUpperCase()` handler in two ways: with `??` and with a check that returns status `400`.

---

## Interview Questions

### Should you write parameter types and return types?

Parameter types, always: without them the parameter is `any` (TS7006). Return types are inferred, but writing them on named and exported functions is good practice: a wrong `return` is reported inside the function, and recursive functions may need one.

### What is the difference between an optional parameter and a default parameter?

`city?: string` may be left out, and is `string | undefined` inside the function. `quantity = 1` may also be left out, but is always a `number` inside, because the default fills in `undefined`.

### What is a rest parameter, and what is its type?

`...prices: number[]` collects all remaining arguments into an array. Its type is an array type, and it must be the last parameter.

### What is a function type?

A type that describes a function's parameters and return type, written like an arrow function: `(pricePaise: number) => number`. It is used for variables, callbacks and properties that hold functions.

### What is contextual typing?

When TypeScript knows the type a function must have from where it is used, it gives the function's parameters their types automatically. `prices.map((price) => price * 2)` needs no annotation on `price`.

### Why can a callback have fewer parameters than its type says?

Because ignoring an argument is always safe: `() => true` can be called with one argument. A callback with more parameters than it will be called with is an error, because the extra parameters would be `undefined`.

### Why can a callback typed as () => void return a value?

A `void` return type in a callback type means the caller ignores the result. So `forEach((name) => list.push(name))` is fine even though `push` returns a number. A function you declare with `: void` must not return a value.

### What are function overloads?

Several signatures for one function, written above the implementation, so that the return type can depend on the arguments. Callers only see the overloads, not the implementation signature.

### Why does an overloaded function reject a union argument?

Each call must match one overload. `number | number[]` matches neither `number` nor `number[]` alone, and the implementation signature is not visible to callers, so the call is TS2769.

### How do you type a function that takes an options object?

Describe the object with an interface with optional properties, destructure it with defaults in the parameter list, and add `= {}` so the whole argument may be left out: `function list({ page = 1 }: ListOptions = {})`.

### How do you type methods in an interface?

With method syntax, `add(pricePaise: number): void;`, or property syntax, `add: (pricePaise: number) => void;`. The object's methods then get their parameter types from the interface.

### What does TS2554 mean?

"Expected N arguments, but got M": a function was called with too few or too many arguments. JavaScript would ignore extra arguments and fill missing ones with `undefined`.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [11. null, undefined and Optional Values](11-null-undefined-and-optional-values.md) | [Course outline](README.md#course-outline) | [13. Type Assertions, as const and satisfies](13-assertions-as-const-and-satisfies.md) |
