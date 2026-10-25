# Utility Types

> **Session 17 of 33** | Part 4: Generics and Type Tools | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [One Shape, Many Versions](#one-shape-many-versions)
- [Partial: Every Property Optional](#partial-every-property-optional)
- [Required: Every Property Needed](#required-every-property-needed)
- [Readonly: No Changes Allowed](#readonly-no-changes-allowed)
- [Pick: Keep Some Properties](#pick-keep-some-properties)
- [Omit: Remove Some Properties](#omit-remove-some-properties)
- [Types Do Not Remove Data](#types-do-not-remove-data)
- [Record: An Object with Known Keys](#record-an-object-with-known-keys)
- [Exclude and Extract: Filtering a Union](#exclude-and-extract-filtering-a-union)
- [NonNullable: Removing null and undefined](#nonnullable-removing-null-and-undefined)
- [ReturnType and Parameters: Types from Functions](#returntype-and-parameters-types-from-functions)
- [Combining Utility Types](#combining-utility-types)
- [Where Utility Types Come From](#where-utility-types-come-from)
- [Which Utility Type Do I Need?](#which-utility-type-do-i-need)
- [Mini Example 1: A Typed Product API](#mini-example-1-a-typed-product-api)
- [Mini Example 2: An Order Report](#mini-example-2-an-order-report)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `17-utility-types` inside `ts-practice`, with the one-line `tsconfig.json`

17-utility-types/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## One Shape, Many Versions

A backend works with the same thing in several shapes. Take a product in an online shop:

- the **stored** product has everything, including `id` and `createdAt`
- a request that **creates** a product sends no `id` and no `createdAt`, because the server adds them
- a request that **updates** a product sends only the fields that change
- a **list** of products shows only a few fields

Written by hand, that is four interfaces

product.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

interface NewProduct {
  name: string;
  pricePaise: number;
  stock: number;
}

interface ProductChanges {
  name?: string;
  pricePaise?: number;
  stock?: number;
}

interface ProductSummary {
  id: number;
  name: string;
  pricePaise: number;
}
```

It works, but the same properties are written again and again. Rename `pricePaise`, or add a `category` to `Product`, and three more interfaces must be changed by hand. Nothing reminds you, so sooner or later they stop matching.

**Utility types** build the other shapes from `Product`

product.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

type NewProduct = Omit<Product, "id" | "createdAt">;
type ProductChanges = Partial<NewProduct>;
type ProductSummary = Pick<Product, "id" | "name" | "pricePaise">;
```

These three lines give exactly the same types as the three interfaces above. `Product` is written once: `Omit` removes properties, `Partial` makes them optional, and `Pick` keeps a few. When `Product` changes, the others follow.

A utility type is a **generic type** that comes with TypeScript (Session 14). You pass types in the angle brackets and get a new type back. There is nothing to install or import: utility types are always available, like `string`.

A utility type gives a type, not an object shape written with `{ }`, so the result is named with `type` (Session 08's rule: `interface` for object shapes, `type` for everything else).

To see which properties a type has, assign an empty object to it and read the error

```typescript
const check: NewProduct = {};
const summary: ProductSummary = {};
```

```text
product.ts:13:7 - error TS2739: Type '{}' is missing the following properties from type 'NewProduct': name, pricePaise, stock

13 const check: NewProduct = {};
         ~~~~~

product.ts:14:7 - error TS2739: Type '{}' is missing the following properties from type 'ProductSummary': id, name, pricePaise

14 const summary: ProductSummary = {};
         ~~~~~~~


Found 2 errors in the same file, starting at: product.ts:13
```

Remove the two lines again after reading the errors.

![One Product interface in the middle, with NewProduct, ProductChanges, ProductSummary and a readonly Product built from it by utility types](images/17-utility-types/one-source.gif)

---

## Partial: Every Property Optional

`Partial<T>` makes every property of `T` optional. It fits an update, where any field may change

partial.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

function applyChanges(product: Product, changes: Partial<Product>): Product {
  return { ...product, ...changes };
}

const mouse: Product = {
  id: 2,
  name: "Mouse",
  pricePaise: 59900,
  stock: 40,
  createdAt: "2026-10-20"
};

console.log(applyChanges(mouse, { pricePaise: 54900 }));
console.log(applyChanges(mouse, { name: "Wireless Mouse", stock: 35 }));
```

```bash
npx tsc --noEmit
node partial.ts
```

```text
{
  id: 2,
  name: 'Mouse',
  pricePaise: 54900,
  stock: 40,
  createdAt: '2026-10-20'
}
{
  id: 2,
  name: 'Wireless Mouse',
  pricePaise: 59900,
  stock: 35,
  createdAt: '2026-10-20'
}
```

`{ ...product, ...changes }` copies the product, then puts the changes on top (Session 07, Spreading Objects). `Partial<Product>` is the same as writing this by hand

```typescript
interface PartialProduct {
  id?: number;
  name?: string;
  pricePaise?: number;
  stock?: number;
  createdAt?: string;
}
```

Even `{}` is a valid `Partial<Product>`: nothing changes. Every property is optional, but the types are still checked. Add these lines

```typescript
applyChanges(mouse, { pricePaise: "free" });
applyChanges(mouse, { price: 54900 });
applyChanges(mouse, { stock: undefined });
```

```text
partial.ts:23:23 - error TS2322: Type 'string' is not assignable to type 'number'.

23 applyChanges(mouse, { pricePaise: "free" });
                         ~~~~~~~~~~

  partial.ts:4:3 - The expected type comes from property 'pricePaise' which is declared here on type 'Partial<Product>'
    4   pricePaise: number;
        ~~~~~~~~~~

partial.ts:24:23 - error TS2353: Object literal may only specify known properties, and 'price' does not exist in type 'Partial<Product>'.

24 applyChanges(mouse, { price: 54900 });
                         ~~~~~

partial.ts:25:21 - error TS2379: Argument of type '{ stock: undefined; }' is not assignable to parameter of type 'Partial<Product>' with 'exactOptionalPropertyTypes: true'. Consider adding 'undefined' to the types of the target's properties.
  Types of property 'stock' are incompatible.
    Type 'undefined' is not assignable to type 'number'.

25 applyChanges(mouse, { stock: undefined });
                       ~~~~~~~~~~~~~~~~~~~~


Found 3 errors in the same file, starting at: partial.ts:23
```

1. A wrong type is refused, and the message names the type: `'Partial<Product>'`.
2. An unknown property is refused.
3. `undefined` is refused, because of `exactOptionalPropertyTypes` (Session 11): optional means "may be left out", not "may be `undefined`". This matters here. A spread copies `stock: undefined` like any other value, so the product would lose its stock. A property that is left out is not copied at all.

There is one problem. `Partial<Product>` also lets an update change `id` and `createdAt`

```typescript
console.log(applyChanges(mouse, { id: 99 }).id);
```

```text
99
```

[Combining Utility Types](#combining-utility-types) fixes that.

![Partial puts a question mark on every property of Product; a price change and an empty object fit, a text price and stock undefined are refused](images/17-utility-types/partial.gif)

---

## Required: Every Property Needed

`Required<T>` is the opposite: it removes every `?`. It fits options after the defaults are filled in

required.ts

```typescript
interface ServerOptions {
  port?: number;
  host?: string;
  logRequests?: boolean;
}

const DEFAULTS: Required<ServerOptions> = {
  port: 3000,
  host: "localhost",
  logRequests: false
};

function startServer(options: ServerOptions): void {
  const settings: Required<ServerOptions> = { ...DEFAULTS, ...options };
  const logging = settings.logRequests ? "on" : "off";
  console.log(`Listening on ${settings.host}:${settings.port}, logging ${logging}`);
}

startServer({});
startServer({ port: 8080, logRequests: true });
```

```bash
npx tsc --noEmit
node required.ts
```

```text
Listening on localhost:3000, logging off
Listening on localhost:8080, logging on
```

Callers pass only what they want to change; inside the function, every setting has a value. In Session 11's Mini Example 2 you wrote a second interface, `ResolvedOptions`, for the options after the defaults. For the `ServerOptions` of that example, `Required<ServerOptions>` is exactly the same type as `ResolvedOptions`, without writing it by hand.

`{ ...DEFAULTS, ...options }` is safe because `exactOptionalPropertyTypes` refuses `{ port: undefined }`. Without that setting, such a value would overwrite the default `3000` with `undefined`.

`Required` also makes sure the defaults are complete. Remove `logRequests: false` from `DEFAULTS`

```text
required.ts:7:7 - error TS2741: Property 'logRequests' is missing in type '{ port: number; host: string; }' but required in type 'Required<ServerOptions>'.

7 const DEFAULTS: Required<ServerOptions> = {
        ~~~~~~~~

  required.ts:4:3 - 'logRequests' is declared here.
    4   logRequests?: boolean;
        ~~~~~~~~~~~


Found 1 error in required.ts:7
```

When a new option is added to `ServerOptions`, this error is the reminder to give it a default.

![Optional server options with question marks are merged over complete defaults; the result has every setting, and defaults with a missing setting are refused](images/17-utility-types/required-defaults.gif)

---

## Readonly: No Changes Allowed

`Readonly<T>` makes every property `readonly` (Session 07). Use it for a parameter that the function must not change. This function has a bug

readonly.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

function discountedPrice(product: Readonly<Product>, percent: number): number {
  product.pricePaise = product.pricePaise - (product.pricePaise * percent) / 100;
  return product.pricePaise;
}
```

```text
readonly.ts:10:11 - error TS2540: Cannot assign to 'pricePaise' because it is a read-only property.

10   product.pricePaise = product.pricePaise - (product.pricePaise * percent) / 100;
             ~~~~~~~~~~


Found 1 error in readonly.ts:10
```

The function should only calculate a price, but it would also change the product itself, for every part of the program that reads it later. The fixed version returns the new price and leaves the product alone

```typescript
function discountedPrice(product: Readonly<Product>, percent: number): number {
  return product.pricePaise - (product.pricePaise * percent) / 100;
}

const mouse: Product = {
  id: 2,
  name: "Mouse",
  pricePaise: 59900,
  stock: 40,
  createdAt: "2026-10-20"
};

console.log(discountedPrice(mouse, 10));
console.log(mouse.pricePaise);
```

```text
53910
59900
```

`mouse` is a normal `Product`, and it can still be passed in. `Readonly<Product>` in a parameter does not ask for a special object. It is the function's promise not to change it.

`Readonly` is **shallow**, like `readonly` properties and `Object.freeze`

shallow.ts

```typescript
interface Cart {
  owner: string;
  items: string[];
}

function checkout(cart: Readonly<Cart>): void {
  cart.items.push("Free sticker");
  cart.owner = "someone else";
}
```

```text
shallow.ts:8:8 - error TS2540: Cannot assign to 'owner' because it is a read-only property.

8   cart.owner = "someone else";
         ~~~~~


Found 1 error in shallow.ts:8
```

Only `owner` is refused. `cart.items` cannot be replaced, but the array itself can still change. To protect it too, write `items: readonly string[]` in the interface (Session 06).

`Readonly<T>` and `as const` (Session 13) are related. `Readonly<T>` is written on a **type**. `as const` is written on a **value**, and also keeps the literal values.

![Readonly locks the owner of a cart, so assigning it is refused, but pushing into the items array still works because readonly is shallow](images/17-utility-types/readonly-shallow.gif)

---

## Pick: Keep Some Properties

`Pick<T, Keys>` keeps only the listed properties. A product list does not need the stock or the creation date

pick.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

type ProductSummary = Pick<Product, "id" | "name" | "pricePaise">;

function toSummary(product: Product): ProductSummary {
  return { id: product.id, name: product.name, pricePaise: product.pricePaise };
}

const mouse: Product = {
  id: 2,
  name: "Mouse",
  pricePaise: 59900,
  stock: 40,
  createdAt: "2026-10-20"
};

console.log(toSummary(mouse));
```

```bash
npx tsc --noEmit
node pick.ts
```

```text
{ id: 2, name: 'Mouse', pricePaise: 59900 }
```

The keys are a union of literal names (Session 09). They must be real keys of `Product`, which is the `K extends keyof T` constraint from Session 15. Change `"name"` to `"title"`

```text
pick.ts:9:37 - error TS2344: Type '"id" | "pricePaise" | "title"' does not satisfy the constraint 'keyof Product'.
  Type '"title"' is not assignable to type 'keyof Product'.

9 type ProductSummary = Pick<Product, "id" | "title" | "pricePaise">;
                                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

pick.ts:12:28 - error TS2353: Object literal may only specify known properties, and 'name' does not exist in type 'ProductSummary'.

12   return { id: product.id, name: product.name, pricePaise: product.pricePaise };
                              ~~~~


Found 2 errors in the same file, starting at: pick.ts:9
```

The first error is the cause: `"title"` is not a key of `Product`. The second only follows from it: with the wrong key, `ProductSummary` has no `name`. Fix the first error, and the second disappears.

---

## Omit: Remove Some Properties

`Omit<T, Keys>` is the opposite of `Pick`: it keeps everything **except** the listed properties. A request that creates a product has no `id` and no `createdAt`

omit.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

type NewProduct = Omit<Product, "id" | "createdAt">;

const keyboard: NewProduct = {
  name: "Keyboard",
  pricePaise: 149900,
  stock: 12
};

console.log(keyboard);
```

```bash
npx tsc --noEmit
node omit.ts
```

```text
{ name: 'Keyboard', pricePaise: 149900, stock: 12 }
```

Replace the `console.log` line with two more products, one with an `id` and one without a stock

```typescript
const monitor: NewProduct = {
  id: 3,
  name: "Monitor",
  pricePaise: 899900,
  stock: 5
};

const webcam: NewProduct = {
  name: "Webcam",
  pricePaise: 249900
};
```

```text
omit.ts:18:3 - error TS2353: Object literal may only specify known properties, and 'id' does not exist in type 'NewProduct'.

18   id: 3,
     ~~

omit.ts:24:7 - error TS2741: Property 'stock' is missing in type '{ name: string; pricePaise: number; }' but required in type 'NewProduct'.

24 const webcam: NewProduct = {
         ~~~~~~

  omit.ts:5:3 - 'stock' is declared here.
    5   stock: number;
        ~~~~~


Found 2 errors in the same file, starting at: omit.ts:18
```

Use `Pick` when you keep a few properties, and `Omit` when you remove a few. Both give the same kind of type.

![A Product with five properties: Pick keeps id, name and price; Omit removes id and createdAt and keeps the rest](images/17-utility-types/pick-omit.gif)

### Omit does not check its keys

Remove the two new products again, and misspell a key in the type

```typescript
type NewProduct = Omit<Product, "id" | "createAt">;
```

```text
omit.ts:11:7 - error TS2741: Property 'createdAt' is missing in type '{ name: string; pricePaise: number; stock: number; }' but required in type 'NewProduct'.

11 const keyboard: NewProduct = {
         ~~~~~~~~

  omit.ts:6:3 - 'createdAt' is declared here.
    6   createdAt: string;
        ~~~~~~~~~


Found 1 error in omit.ts:11
```

The line with the typo has no error. `Pick` refused `"title"` with TS2344, but `Omit` accepts `"createAt"`, removes nothing, and the error shows up later, where the type is used. The definitions in TypeScript's own files show why

```typescript
type Pick<T, K extends keyof T> = {
```

```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

`Pick` asks for `keyof T`, the real keys. `Omit` asks for `keyof any`, which is any key at all (`string | number | symbol`). Removing a key that does not exist is allowed; it just does nothing. When an `Omit` type still has a property you removed, check the spelling first.

![Pick with a misspelled key is refused at the type; Omit with a misspelled key passes, removes nothing, and the error appears far away where the type is used](images/17-utility-types/omit-no-check.gif)

---

## Types Do Not Remove Data

`Omit` is often used to keep secrets out of responses. A user is stored with a password hash, which must never be sent to the client

public-user.ts

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  passwordHash: string;
}

type PublicUser = Omit<User, "passwordHash">;

const asha: User = {
  id: 7,
  name: "Asha",
  email: "asha@example.com",
  passwordHash: "hash-placeholder"
};

function toPublic(user: User): PublicUser {
  return user;
}

console.log(JSON.stringify(toPublic(asha)));
```

```bash
npx tsc --noEmit
node public-user.ts
```

```text
{"id":7,"name":"Asha","email":"asha@example.com","passwordHash":"hash-placeholder"}
```

`tsc` is silent, and the password hash is in the response. There are two reasons:

1. A `User` has every property that a `PublicUser` needs, so it fits. Extra properties are allowed when a variable is passed or returned (Session 07, structural typing).
2. Types are removed before the code runs (Session 02). `Omit` changes what TypeScript lets you **read**, not what the object **contains**.

To really leave the property out, build a new object without it. Rest destructuring ([JavaScript course, Session 14](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/14-destructuring-spread-rest.md)) does exactly that

```typescript
function toPublic(user: User): PublicUser {
  const { passwordHash, ...publicFields } = user;
  return publicFields;
}
```

```text
{"id":7,"name":"Asha","email":"asha@example.com"}
```

`publicFields` holds every property except `passwordHash`, and TypeScript knows that: its type is `{ id: number; name: string; email: string; }`.

A new object literal is checked for extra properties (Session 07), so this version would have been caught

```typescript
function toPublic(user: User): PublicUser {
  return { id: user.id, name: user.name, email: user.email, passwordHash: user.passwordHash };
}
```

```text
public-user.ts:18:61 - error TS2353: Object literal may only specify known properties, and 'passwordHash' does not exist in type 'PublicUser'.

18   return { id: user.id, name: user.name, email: user.email, passwordHash: user.passwordHash };
                                                               ~~~~~~~~~~~~


Found 1 error in public-user.ts:18
```

But `return user` is not a new literal, and it is not checked. When data leaves your server, build the object you send.

![A user with a password hash goes through toPublic typed as PublicUser and the JSON still contains the hash; with rest destructuring the hash is really left out](images/17-utility-types/types-dont-remove.gif)

---

## Record: An Object with Known Keys

`Record<Keys, Value>` is an object type whose keys are `Keys` and whose values are all `Value`. With a literal union as the keys, it is a table with one row for every value

record.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "delivered" | "cancelled";

const STATUS_LABELS: Record<OrderStatus, string> = {
  pending: "Waiting for payment",
  paid: "Paid, packing",
  shipped: "On the way",
  delivered: "Delivered",
  cancelled: "Cancelled"
};

function statusLine(id: number, status: OrderStatus): string {
  return `Order ${id}: ${STATUS_LABELS[status]}`;
}

console.log(statusLine(1001, "paid"));
console.log(statusLine(1002, "shipped"));
```

```bash
npx tsc --noEmit
node record.ts
```

```text
Order 1001: Paid, packing
Order 1002: On the way
```

Every key is required. Remove the `delivered` line

```text
record.ts:3:7 - error TS2741: Property 'delivered' is missing in type '{ pending: string; paid: string; shipped: string; cancelled: string; }' but required in type 'Record<OrderStatus, string>'.

3 const STATUS_LABELS: Record<OrderStatus, string> = {
        ~~~~~~~~~~~~~


Found 1 error in record.ts:3
```

And a misspelled key gets a suggestion. Put `delivered` back and write `shiped`

```text
record.ts:6:3 - error TS2561: Object literal may only specify known properties, but 'shiped' does not exist in type 'Record<OrderStatus, string>'. Did you mean to write 'shipped'?

6   shiped: "On the way",
    ~~~~~~


Found 1 error in record.ts:6
```

Because every status has a label, `STATUS_LABELS[status]` is a `string`, never `undefined`. When a new status is added to `OrderStatus`, the table shows an error until the new label is added.

### Record with string keys

Session 07 promised a shorter way to write a lookup like `{ [productId: string]: number }`. Here it is

stock.ts

```typescript
const stock: Record<string, number> = {
  "KB-01": 12,
  "MS-02": 0
};

stock["MN-03"] = 5;

console.log(stock["KB-01"]);
console.log(stock["XX-99"]);
console.log(stock);
```

```text
12
undefined
{ 'KB-01': 12, 'MS-02': 0, 'MN-03': 5 }
```

`Record<string, number>` is exactly the index signature `{ [key: string]: number }`, so it behaves the same: any key may be written, and reading a key may give `undefined`

```typescript
const left: number = stock["KB-01"];
```

```text
stock.ts:11:7 - error TS2322: Type 'number | undefined' is not assignable to type 'number'.
  Type 'undefined' is not assignable to type 'number'.

11 const left: number = stock["KB-01"];
         ~~~~


Found 1 error in stock.ts:11
```

Session 13 also promised one: in its `routes.ts`, the route table ended with `satisfies { [name: string]: RouteConfig }`. Change that line to

```typescript
} satisfies Record<string, RouteConfig>;
```

The output and every error from Session 13 stay exactly the same.

Which one to use:

| Keys                                     | Type                   | Every key required? | A read can be `undefined`? |
| ---------------------------------------- | ---------------------- | ------------------- | -------------------------- |
| A fixed set, like the order statuses     | `Record<OrderStatus, V>` | Yes               | No                         |
| Not known in advance, like product ids   | `Record<string, V>`      | No                | Yes                        |

![A Record of order statuses has one slot for every status and a missing one is refused; a Record with string keys accepts any key, and reading an unknown key gives undefined](images/17-utility-types/record-keys.gif)

---

## Exclude and Extract: Filtering a Union

`Exclude<Union, Members>` removes members from a union. `Extract<Union, Members>` keeps only the matching ones

status-filter.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "delivered" | "cancelled";

type OpenStatus = Exclude<OrderStatus, "delivered" | "cancelled">;
type ClosedStatus = Extract<OrderStatus, "delivered" | "cancelled">;

function cancelOrder(id: number, status: OpenStatus): void {
  console.log(`Order ${id} was ${status}, now cancelled`);
}

function archiveOrder(id: number, status: ClosedStatus): void {
  console.log(`Order ${id} (${status}) moved to the archive`);
}

cancelOrder(1001, "paid");
archiveOrder(1002, "delivered");
```

```bash
npx tsc --noEmit
node status-filter.ts
```

```text
Order 1001 was paid, now cancelled
Order 1002 (delivered) moved to the archive
```

`OpenStatus` is `"pending" | "paid" | "shipped"`, and `ClosedStatus` is `"delivered" | "cancelled"`. A new status in `OrderStatus` joins `OpenStatus` by itself, because `Exclude` keeps everything it does not remove ([Mini Example 2](#mini-example-2-an-order-report) uses this). `ClosedStatus` only keeps the two listed statuses. Add the wrong calls

```typescript
cancelOrder(1003, "delivered");
archiveOrder(1004, "paid");
```

```text
status-filter.ts:16:19 - error TS2345: Argument of type '"delivered"' is not assignable to parameter of type 'OpenStatus'.

16 cancelOrder(1003, "delivered");
                     ~~~~~~~~~~~

status-filter.ts:17:20 - error TS2345: Argument of type '"paid"' is not assignable to parameter of type 'ClosedStatus'.

17 archiveOrder(1004, "paid");
                      ~~~~~~


Found 2 errors in the same file, starting at: status-filter.ts:16
```

Like `Omit`, `Exclude` does not check what you remove. Remove the two wrong calls, and misspell one status in the type, then cancel an order that is already cancelled

```typescript
type OpenStatus = Exclude<OrderStatus, "delivered" | "canceled">;
```

```typescript
cancelOrder(1005, "cancelled");
```

```text
Order 1001 was paid, now cancelled
Order 1002 (delivered) moved to the archive
Order 1005 was cancelled, now cancelled
```

No error: `"canceled"` is not in the union, so nothing was removed, and `"cancelled"` is still an `OpenStatus`.

### Extract with object types

`Extract` and `Exclude` also work on unions of objects, like the discriminated unions from Session 10. This is useful when the members have no names of their own

payment.ts

```typescript
type Payment =
  | { method: "card"; cardLast4: string }
  | { method: "upi"; upiId: string }
  | { method: "cod" };

type CardPayment = Extract<Payment, { method: "card" }>;
type OnlinePayment = Exclude<Payment, { method: "cod" }>;

function cardLine(payment: CardPayment): string {
  return `Card ending ${payment.cardLast4}`;
}

function refundTo(payment: OnlinePayment): string {
  if (payment.method === "card") {
    return cardLine(payment);
  }
  return `UPI ${payment.upiId}`;
}

console.log(refundTo({ method: "card", cardLast4: "4242" }));
console.log(refundTo({ method: "upi", upiId: "asha@okbank" }));
```

```text
Card ending 4242
UPI asha@okbank
```

`Extract<Payment, { method: "card" }>` keeps the members that fit `{ method: "card" }`, which is the card payment. `Exclude<Payment, { method: "cod" }>` drops cash on delivery, because there is nothing to refund to. The check on `method` narrows as in Session 10. A cash payment is refused

```typescript
refundTo({ method: "cod" });
```

```text
payment.ts:22:12 - error TS2322: Type '"cod"' is not assignable to type '"card" | "upi"'.

22 refundTo({ method: "cod" });
              ~~~~~~

  payment.ts:2:7 - The expected type comes from property 'method' which is declared here on type 'OnlinePayment'
    2   | { method: "card"; cardLast4: string }
            ~~~~~~


Found 1 error in payment.ts:22
```

A typo like `Extract<Payment, { method: "crad" }>` gives no error either. It gives `never`, because no member matches.

![A row of order status chips: Exclude drops delivered and cancelled and leaves the open statuses; Extract keeps only the matching ones](images/17-utility-types/exclude-extract.gif)

---

## NonNullable: Removing null and undefined

`NonNullable<T>` removes `null` and `undefined` from a type

```typescript
type MaybePhone = string | null | undefined;
type Phone = NonNullable<MaybePhone>;
```

`Phone` is `string`. With a type you wrote yourself, you could just write `string`. `NonNullable` is most useful in generic code, where the type is not known in advance

compact.ts

```typescript
function compact<T>(items: readonly T[]): NonNullable<T>[] {
  const result: NonNullable<T>[] = [];
  for (const item of items) {
    if (item !== null && item !== undefined) {
      result.push(item);
    }
  }
  return result;
}

const phones = compact(["98200 12345", null, undefined, "98200 67890"]);
const quantities = compact([2, undefined, 0, 5]);

console.log(phones);
console.log(quantities);
```

```bash
npx tsc --noEmit
node compact.ts
```

```text
[ '98200 12345', '98200 67890' ]
[ 2, 0, 5 ]
```

`phones` is `string[]` and `quantities` is `number[]`: the `null` and `undefined` are gone from the types, not only from the arrays. `0` is kept, because the check is `!== null && !== undefined`, not a truthiness check (Session 11).

---

## ReturnType and Parameters: Types from Functions

This course writes a return type on every named function, often with a named interface, so you can reuse those names. Code written by others is not always like that. A module from another team, or a library, may use inline types and export no names for them

shipping.ts

```typescript
export function quoteShipping(parcel: { city: string; weightGrams: number; express: boolean }): { costPaise: number; days: number } {
  const base = parcel.weightGrams <= 500 ? 4000 : 7000;
  return parcel.express ? { costPaise: base * 2, days: 1 } : { costPaise: base, days: 4 };
}
```

Your code needs both types. Instead of copying them, take them from the function

checkout.ts

```typescript
import { quoteShipping } from "./shipping.ts";

type Parcel = Parameters<typeof quoteShipping>[0];
type Quote = ReturnType<typeof quoteShipping>;

function describeQuote(quote: Quote): string {
  return `Rs. ${quote.costPaise / 100}, ${quote.days} day(s)`;
}

const parcel: Parcel = { city: "Pune", weightGrams: 800, express: false };

console.log(describeQuote(quoteShipping(parcel)));
console.log(describeQuote(quoteShipping({ ...parcel, express: true })));
```

```bash
npx tsc --noEmit
node checkout.ts
```

```text
Rs. 70, 4 day(s)
Rs. 140, 1 day(s)
```

Read the two type lines step by step:

- `typeof quoteShipping` is the type of the function (Session 15).
- `ReturnType<...>` takes what it returns: `{ costPaise: number; days: number }`.
- `Parameters<...>` takes all parameter types as a tuple (Session 06): `[parcel: { city: string; weightGrams: number; express: boolean }]`. `[0]` picks the first one (Session 15).

The types stay checked. A parcel without `express`

```typescript
const bad: Parcel = { city: "Pune", weightGrams: 800 };
```

```text
checkout.ts:14:7 - error TS2741: Property 'express' is missing in type '{ city: string; weightGrams: number; }' but required in type '{ city: string; weightGrams: number; express: boolean; }'.

14 const bad: Parcel = { city: "Pune", weightGrams: 800 };
         ~~~

  shipping.ts:1:76 - 'express' is declared here.
    1 export function quoteShipping(parcel: { city: string; weightGrams: number; express: boolean }): { costPaise: number; days: number } {
                                                                                 ~~~~~~~


Found 1 error in checkout.ts:14
```

The related information points into `shipping.ts`: the type really comes from there. If the other team changes the function, your types change with it.

Forgetting `typeof` is a common slip

```typescript
type Quote = ReturnType<quoteShipping>;
```

```text
checkout.ts:4:25 - error TS2749: 'quoteShipping' refers to a value, but is being used as a type here. Did you mean 'typeof quoteShipping'?

4 type Quote = ReturnType<quoteShipping>;
                          ~~~~~~~~~~~~~


Found 1 error in checkout.ts:4
```

`ReturnType` needs a type, and a function is a value. The message even suggests the fix.

### A line you will see in real projects

`setTimeout` returns a number in a browser, but an object in Node.js. Instead of choosing, many projects write

timer.ts

```typescript
let timer: ReturnType<typeof setTimeout> | undefined;

function saveLater(text: string): void {
  clearTimeout(timer);
  timer = setTimeout(() => console.log(`Saved: ${text}`), 500);
}

saveLater("Hel");
saveLater("Hello");
saveLater("Hello world");
```

```text
Saved: Hello world
```

Each call cancels the timer of the call before, so only the last text is saved, half a second later. In this project, `ReturnType<typeof setTimeout>` is `NodeJS.Timeout`, the type from `@types/node`.

![The quoteShipping function with its parameter and return type; Parameters takes the parameter list and index 0 gives Parcel, ReturnType takes the return type Quote](images/17-utility-types/types-from-functions.gif)

---

## Combining Utility Types

A utility type takes a type, and a utility type gives one, so they can be nested. `Partial<Product>` allowed changing `id`. The right type for an update leaves `id` and `createdAt` out first

combine.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

type ProductChanges = Partial<Omit<Product, "id" | "createdAt">>;
type ProductPatch = Pick<Product, "id"> & ProductChanges;

function describePatch(patch: ProductPatch): string {
  const fields = Object.keys(patch).filter((key) => key !== "id");
  return `Product ${patch.id}: ${fields.join(", ")}`;
}

console.log(describePatch({ id: 2, pricePaise: 54900 }));
console.log(describePatch({ id: 3, name: "Webcam HD", stock: 8 }));
```

```bash
npx tsc --noEmit
node combine.ts
```

```text
Product 2: pricePaise
Product 3: name, stock
```

Read nested utility types from the inside out:

1. `Omit<Product, "id" | "createdAt">` gives `name`, `pricePaise` and `stock`.
2. `Partial<...>` makes those three optional.

`ProductPatch` joins two types with `&` (Session 08): a required `id` from `Pick`, plus the optional changes. That is the body of a request that sends the id together with the changes. Both rules are checked

```typescript
const a: ProductPatch = { pricePaise: 54900 };
const b: ProductPatch = { id: 2, createdAt: "2026-01-01" };
```

```text
combine.ts:20:7 - error TS2322: Type '{ pricePaise: number; }' is not assignable to type 'ProductPatch'.
  Property 'id' is missing in type '{ pricePaise: number; }' but required in type 'Pick<Product, "id">'.

20 const a: ProductPatch = { pricePaise: 54900 };
         ~

  combine.ts:2:3 - 'id' is declared here.
    2   id: number;
        ~~

combine.ts:21:34 - error TS2353: Object literal may only specify known properties, and 'createdAt' does not exist in type 'ProductPatch'.

21 const b: ProductPatch = { id: 2, createdAt: "2026-01-01" };
                                    ~~~~~~~~~


Found 2 errors in the same file, starting at: combine.ts:20
```

Give each step a name, like `ProductChanges` here. Three or four utility types nested in one line are hard to read, and the name appears in the error messages.

### Utility types in generic functions

Utility types work with type parameters too (Session 14). This `applyChanges` works for any object

generic.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

function applyChanges<T>(item: T, changes: Partial<T>): T {
  return { ...item, ...changes };
}

const mouse: Product = { id: 2, name: "Mouse", pricePaise: 59900, stock: 40, createdAt: "2026-10-20" };
const options = { port: 3000, host: "localhost" };

console.log(applyChanges(mouse, { stock: 35 }));
console.log(applyChanges(options, { port: 8080 }));
```

```text
{
  id: 2,
  name: 'Mouse',
  pricePaise: 59900,
  stock: 35,
  createdAt: '2026-10-20'
}
{ port: 8080, host: 'localhost' }
```

`T` is inferred from the first argument, and `Partial<T>` checks the second one against it

```typescript
applyChanges(mouse, { colour: "black" });
applyChanges(options, { port: "8080" });
```

```text
generic.ts:19:23 - error TS2353: Object literal may only specify known properties, and 'colour' does not exist in type 'Partial<Product>'.

19 applyChanges(mouse, { colour: "black" });
                         ~~~~~~

generic.ts:20:25 - error TS2322: Type 'string' is not assignable to type 'number'.

20 applyChanges(options, { port: "8080" });
                           ~~~~

  generic.ts:14:19 - The expected type comes from property 'port' which is declared here on type 'Partial<{ port: number; host: string; }>'
    14 const options = { port: 3000, host: "localhost" };
                         ~~~~~~~~~~


Found 2 errors in the same file, starting at: generic.ts:19
```

![Reading Partial of Omit from the inside out: first Omit removes id and createdAt, then Partial makes the rest optional](images/17-utility-types/combine.gif)

---

## Where Utility Types Come From

Utility types are not magic. They are written in TypeScript, in `lib.es5.d.ts`, the file that appeared in error messages in Session 05

```typescript
/**
 * Make all properties in T optional
 */
type Partial<T> = {
    [P in keyof T]?: T[P];
};
```

Read it as: "for each key `P` of `T`, a property `P` that is optional (`?`) and has the type `T[P]`". You know `keyof T` and `T[P]` from Session 15. `Pick` is built the same way

```typescript
/**
 * From T, pick a set of properties whose keys are in the union K
 */
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};
```

The `[P in ...]` part is a **mapped type**. `Exclude` uses `extends ? :` (a **conditional type**), and `ReturnType` uses `infer`. Session 18 teaches all three, and then you can write your own utility types.

---

## Which Utility Type Do I Need?

| Utility type        | What it gives                              | Typical backend use                         |
| ------------------- | ------------------------------------------ | ------------------------------------------- |
| `Partial<T>`        | Every property optional                    | The body of an update request               |
| `Required<T>`       | Every property required                    | Options after the defaults are filled in    |
| `Readonly<T>`       | Every property readonly                    | A parameter the function must not change    |
| `Pick<T, Keys>`     | Only the listed properties                 | List items, summaries                       |
| `Omit<T, Keys>`     | Everything except the listed properties    | The body of a create request, public data   |
| `Record<Keys, V>`   | An object with those keys, all of type `V` | Lookup tables, counters                     |
| `Exclude<U, M>`     | The union without the members `M`          | Open statuses, online payments              |
| `Extract<U, M>`     | Only the members of the union that match `M` | One member of a discriminated union       |
| `NonNullable<T>`    | `T` without `null` and `undefined`         | Generic helpers that drop empty values      |
| `Parameters<F>`     | The parameter types, as a tuple            | Types that a library does not export        |
| `ReturnType<F>`     | The return type                            | Types that a library does not export        |

There are more. You will meet `Awaited<T>`, for promises, in Session 23.

---

## Mini Example 1: A Typed Product API

The core of a products API, with the data kept in an array for now (Session 32 uses MongoDB). Every request and response type comes from `Product`

product-api.ts

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  createdAt: string;
}

type NewProduct = Omit<Product, "id" | "createdAt">;
type ProductChanges = Partial<NewProduct>;
type ProductSummary = Pick<Product, "id" | "name" | "pricePaise">;

const products: Product[] = [];
let nextId = 1;

function createProduct(input: NewProduct): Readonly<Product> {
  const product: Product = { ...input, id: nextId, createdAt: new Date().toISOString() };
  nextId += 1;
  products.push(product);
  return product;
}

function updateProduct(id: number, changes: ProductChanges): Readonly<Product> | undefined {
  const index = products.findIndex((product) => product.id === id);
  const existing = products[index];
  if (existing === undefined) {
    return undefined;
  }
  const updated: Product = { ...existing, ...changes };
  products[index] = updated;
  return updated;
}

function listProducts(): ProductSummary[] {
  return products.map((product) => ({ id: product.id, name: product.name, pricePaise: product.pricePaise }));
}

createProduct({ name: "Keyboard", pricePaise: 149900, stock: 12 });
createProduct({ name: "Mouse", pricePaise: 59900, stock: 40 });
updateProduct(2, { pricePaise: 54900 });

console.log(listProducts());
console.log(updateProduct(9, { stock: 0 }));
```

```bash
npx tsc --noEmit
node product-api.ts
```

```text
[
  { id: 1, name: 'Keyboard', pricePaise: 149900 },
  { id: 2, name: 'Mouse', pricePaise: 54900 }
]
undefined
```

| Code                                                   | From this session and earlier                                   |
| ------------------------------------------------------ | --------------------------------------------------------------- |
| `Omit<Product, "id" \| "createdAt">`                   | The server sets `id` and `createdAt`, so a request cannot send them |
| `Partial<NewProduct>`                                  | An update sends only what changes, and never `id`              |
| `Pick<Product, "id" \| "name" \| "pricePaise">`        | The list shows three fields                                    |
| `Readonly<Product>` as the return type                 | The caller gets the stored object, and must not change it      |
| `products[index]` is `Product \| undefined`            | Session 06. `findIndex` gives `-1` when nothing is found, and `products[-1]` is `undefined`, so one check covers both |
| `{ ...existing, ...changes }`                          | Safe, because `exactOptionalPropertyTypes` refuses `undefined` values |

`listProducts` builds new objects instead of returning `products`. As in [Types Do Not Remove Data](#types-do-not-remove-data), returning the stored objects would pass `tsc`, but send the stock and the date too.

Now the wrong requests

```typescript
createProduct({ name: "Monitor", pricePaise: 899900 });
createProduct({ id: 7, name: "Monitor", pricePaise: 899900, stock: 5 });
updateProduct(1, { id: 5 });
updateProduct(1, { stock: undefined });
```

```text
product-api.ts:45:15 - error TS2741: Property 'stock' is missing in type '{ name: string; pricePaise: number; }' but required in type 'NewProduct'.

45 createProduct({ name: "Monitor", pricePaise: 899900 });
                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  product-api.ts:5:3 - 'stock' is declared here.
    5   stock: number;
        ~~~~~

product-api.ts:46:17 - error TS2353: Object literal may only specify known properties, and 'id' does not exist in type 'NewProduct'.

46 createProduct({ id: 7, name: "Monitor", pricePaise: 899900, stock: 5 });
                   ~~

product-api.ts:47:20 - error TS2353: Object literal may only specify known properties, and 'id' does not exist in type 'Partial<NewProduct>'.

47 updateProduct(1, { id: 5 });
                      ~~

product-api.ts:48:18 - error TS2379: Argument of type '{ stock: undefined; }' is not assignable to parameter of type 'Partial<NewProduct>' with 'exactOptionalPropertyTypes: true'. Consider adding 'undefined' to the types of the target's properties.
  Types of property 'stock' are incompatible.
    Type 'undefined' is not assignable to type 'number'.

48 updateProduct(1, { stock: undefined });
                    ~~~~~~~~~~~~~~~~~~~~


Found 4 errors in the same file, starting at: product-api.ts:45
```

Remove them, and try to change a product that `createProduct` returned

```typescript
const webcam = createProduct({ name: "Webcam", pricePaise: 249900, stock: 8 });
webcam.stock = 0;
```

```text
product-api.ts:46:8 - error TS2540: Cannot assign to 'stock' because it is a read-only property.

46 webcam.stock = 0;
          ~~~~~


Found 1 error in product-api.ts:46
```

`webcam` is the same object as the one in `products`. Without `Readonly`, this line would change the stored product without going through `updateProduct`.

Now the real advantage. Remove the two lines, and add a `category` to `Product`

```typescript
interface Product {
  id: number;
  name: string;
  pricePaise: number;
  stock: number;
  category: string;
  createdAt: string;
}
```

```text
product-api.ts:39:15 - error TS2741: Property 'category' is missing in type '{ name: string; pricePaise: number; stock: number; }' but required in type 'NewProduct'.

39 createProduct({ name: "Keyboard", pricePaise: 149900, stock: 12 });
                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  product-api.ts:6:3 - 'category' is declared here.
    6   category: string;
        ~~~~~~~~

product-api.ts:40:15 - error TS2741: Property 'category' is missing in type '{ name: string; pricePaise: number; stock: number; }' but required in type 'NewProduct'.

40 createProduct({ name: "Mouse", pricePaise: 59900, stock: 40 });
                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  product-api.ts:6:3 - 'category' is declared here.
    6   category: string;
        ~~~~~~~~


Found 2 errors in the same file, starting at: product-api.ts:39
```

One line changed in `Product`, and `tsc` lists exactly the places that must change: the two create requests. `NewProduct` now needs a category, `ProductChanges` may change it, and `ProductSummary` still leaves it out. No other type had to be touched. Add the categories

```typescript
createProduct({ name: "Keyboard", pricePaise: 149900, stock: 12, category: "accessories" });
createProduct({ name: "Mouse", pricePaise: 59900, stock: 40, category: "accessories" });
```

and the output is the same as before.

In a real API, the request body arrives as JSON from outside. These types describe what it **should** be. Checking that it really is comes in Session 26, with Joi.

![Create, update and list requests each use a type built from Product; adding a category to Product turns only the two create calls red](images/17-utility-types/product-api.gif)

---

## Mini Example 2: An Order Report

A daily report for a shop: how many orders are in each status, and what to do next with the open ones. `Record` makes sure that no status is forgotten

order-report.ts

```typescript
const OrderStatus = {
  Pending: "pending",
  Paid: "paid",
  Shipped: "shipped",
  Delivered: "delivered",
  Cancelled: "cancelled"
} as const;

type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
type OpenStatus = Exclude<OrderStatus, "delivered" | "cancelled">;

interface Order {
  id: number;
  customer: string;
  status: OrderStatus;
}

const STATUS_LABELS: Record<OrderStatus, string> = {
  pending: "Waiting for payment",
  paid: "Paid, packing",
  shipped: "On the way",
  delivered: "Delivered",
  cancelled: "Cancelled"
};

const NEXT_ACTION: Record<OpenStatus, string> = {
  pending: "send a payment reminder",
  paid: "pack and ship",
  shipped: "track the parcel"
};

function isOpen(status: OrderStatus): status is OpenStatus {
  return status !== "delivered" && status !== "cancelled";
}

function countByStatus(orders: readonly Order[]): Record<OrderStatus, number> {
  const counts: Record<OrderStatus, number> = { pending: 0, paid: 0, shipped: 0, delivered: 0, cancelled: 0 };
  for (const order of orders) {
    counts[order.status] += 1;
  }
  return counts;
}

const orders: Order[] = [
  { id: 1001, customer: "Asha", status: "paid" },
  { id: 1002, customer: "Ravi", status: "delivered" },
  { id: 1003, customer: "Meera", status: "pending" },
  { id: 1004, customer: "John", status: "paid" },
  { id: 1005, customer: "Sara", status: "cancelled" }
];

const counts = countByStatus(orders);
for (const status of Object.values(OrderStatus)) {
  console.log(`${STATUS_LABELS[status].padEnd(20)} ${counts[status]}`);
}

console.log("To do:");
for (const order of orders) {
  if (isOpen(order.status)) {
    console.log(`  ${order.id} ${order.customer}: ${NEXT_ACTION[order.status]}`);
  }
}
```

```bash
npx tsc --noEmit
node order-report.ts
```

```text
Waiting for payment  1
Paid, packing        2
On the way           0
Delivered            1
Cancelled            1
To do:
  1001 Asha: pack and ship
  1003 Meera: send a payment reminder
  1004 John: pack and ship
```

| Code                                                   | From this session and earlier                                  |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `const OrderStatus = { ... } as const` + `type OrderStatus` | The constant object pattern (Session 16)                  |
| `Exclude<OrderStatus, "delivered" \| "cancelled">`     | The statuses that still need work                              |
| `Record<OrderStatus, string>`                          | A label for every status                                       |
| `Record<OpenStatus, string>`                           | An action for every open status, and only for those            |
| `Record<OrderStatus, number>`                          | A counter for every status, starting at 0                      |
| `status is OpenStatus`                                 | A type predicate (Session 10), so `NEXT_ACTION[order.status]` is allowed |
| `Object.values(OrderStatus)`                           | The statuses in a fixed order, typed as `OrderStatus` (Session 16) |

Why the `isOpen` check is needed: without it, `order.status` could be `"cancelled"`, which has no next action

```text
order-report.ts:59:51 - error TS7053: Element implicitly has an 'any' type because expression of type 'OrderStatus' can't be used to index type 'Record<OpenStatus, string>'.
  Property 'cancelled' does not exist on type 'Record<OpenStatus, string>'.

59   console.log(`  ${order.id} ${order.customer}: ${NEXT_ACTION[order.status]}`);
                                                     ~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in order-report.ts:59
```

Now the shop starts accepting returns. Add a new status to the constant object

```typescript
  Cancelled: "cancelled",
  Returned: "returned"
```

```text
order-report.ts:19:7 - error TS2741: Property 'returned' is missing in type '{ pending: string; paid: string; shipped: string; delivered: string; cancelled: string; }' but required in type 'Record<OrderStatus, string>'.

19 const STATUS_LABELS: Record<OrderStatus, string> = {
         ~~~~~~~~~~~~~

order-report.ts:27:7 - error TS2741: Property 'returned' is missing in type '{ pending: string; paid: string; shipped: string; }' but required in type 'Record<OpenStatus, string>'.

27 const NEXT_ACTION: Record<OpenStatus, string> = {
         ~~~~~~~~~~~

order-report.ts:38:9 - error TS2741: Property 'returned' is missing in type '{ pending: number; paid: number; shipped: number; delivered: number; cancelled: number; }' but required in type 'Record<OrderStatus, number>'.

38   const counts: Record<OrderStatus, number> = { pending: 0, paid: 0, shipped: 0, delivered: 0, cancelled: 0 };
           ~~~~~~


Found 3 errors in the same file, starting at: order-report.ts:19
```

Three tables, three errors, each one at the table that needs a new row. The second error is a question: `"returned"` is not excluded, so it counts as open. Is there something to do with a returned order? Yes, a refund, so it stays open. Add the rows

```typescript
  cancelled: "Cancelled",
  returned: "Returned, refund due"
```

```typescript
  shipped: "track the parcel",
  returned: "issue the refund"
```

```typescript
  const counts: Record<OrderStatus, number> = { pending: 0, paid: 0, shipped: 0, delivered: 0, cancelled: 0, returned: 0 };
```

and a returned order to the list

```typescript
  { id: 1006, customer: "Kiran", status: "returned" }
```

```text
Waiting for payment  1
Paid, packing        2
On the way           0
Delivered            1
Cancelled            1
Returned, refund due 1
To do:
  1001 Asha: pack and ship
  1003 Meera: send a payment reminder
  1004 John: pack and ship
  1006 Kiran: issue the refund
```

The report loop did not change at all: `Object.values(OrderStatus)` already includes the new status.

![The report counts orders per status and lists actions for open ones; adding a returned status turns the label table, the action table and the counters red until each gets a row](images/17-utility-types/order-report.gif)

---

## Beginner Mistakes

### Mistake 1: Writing the same shape by hand several times

`NewProduct`, `ProductChanges` and `ProductSummary` written as separate interfaces stop matching `Product` sooner or later. Build them from `Product` with `Omit`, `Partial` and `Pick`.

---

### Mistake 2: Using Partial<Product> for an update

`Partial<Product>` lets an update change `id` and `createdAt`: `applyChanges(mouse, { id: 99 })` passed. Remove those fields first: `Partial<Omit<Product, "id" | "createdAt">>`.

---

### Mistake 3: Expecting Omit or Exclude to catch a typo

`Omit<Product, "id" | "createAt">` and `Exclude<OrderStatus, "canceled">` give no error. They remove nothing, and the problem appears somewhere else, or not at all. `Pick` and `Record` with a union key do check their keys.

---

### Mistake 4: Thinking Omit removes data

`return user` from a function typed `PublicUser` passed `tsc` and still sent `passwordHash`. Types are removed before the code runs. Build a new object, for example with `const { passwordHash, ...publicFields } = user`.

---

### Mistake 5: Record<string, V> when the keys are known

`Record<string, string>` for status labels accepts a missing status, and every read may be `undefined`. `Record<OrderStatus, string>` requires a label for every status, and reads are always `string`.

---

### Mistake 6: Forgetting typeof

`ReturnType<quoteShipping>` gives `error TS2749: 'quoteShipping' refers to a value, but is being used as a type here.` Write `ReturnType<typeof quoteShipping>`.

---

### Mistake 7: Expecting Readonly to be deep

`Readonly<Cart>` refused `cart.owner = ...` but allowed `cart.items.push(...)`. Make the array itself readonly with `items: readonly string[]`.

---

### Mistake 8: Sending undefined in an update

`updateProduct(1, { stock: undefined })` gives `error TS2379`. With a spread, `undefined` would overwrite the real value. Leave the property out instead.

---

### Mistake 9: Nesting utility types in one long line

`Readonly<Partial<Omit<Pick<Product, ...>, ...>>>` is hard to read and hard to fix. Name each step, like `NewProduct` and `ProductChanges`, and the names also make the error messages shorter.

---

### Mistake 10: Using ReturnType where a named type is clearer

For your own functions, write a named interface and use it as the return type. `ReturnType` and `Parameters` are for functions whose types have no name you can import, like code from another team or a library.

---

## Practice Exercises

### Exercise 1

In `17-utility-types`, write an interface `Customer` with `id`, `name`, `email`, an optional `phone` and `createdAt`. Build `NewCustomer` (no `id`, no `createdAt`), `CustomerChanges` (everything in `NewCustomer`, optional) and `CustomerCard` (only `id` and `name`). Use the `{}` trick to check each one.

### Exercise 2

Write `updateCustomer(customer: Customer, changes: CustomerChanges): Customer` with a spread. Call it with a new email, then with `{ phone: undefined }`, and explain the error. How would you remove a phone number from a customer instead? (Hint: think about `phone: string | null`, Session 11.)

### Exercise 3

Write `interface LoggerOptions { level?: "debug" | "info" | "warn" | "error"; json?: boolean }`, a `DEFAULTS: Required<LoggerOptions>`, and `createLogger(options: LoggerOptions): void`, which prints the final settings. Then add an option `prefix?: string` and follow the error.

### Exercise 4

Repeat the `PublicUser` example with your own `User`. Print the JSON from the leaking version and from the fixed version. Then add a second secret, `resetToken: string`, to `User`, and update `PublicUser` and `toPublic` so it never leaves the server.

### Exercise 5

Write `type Method = "GET" | "POST" | "DELETE"` and a function that counts the methods in a list like `["GET", "POST", "GET"]` into a `Record<Method, number>`. Then add `"PUT"` to `Method` and read the error.

### Exercise 6

In Session 16's Mini Example 1, change `satisfies { [status: string]: readonly OrderStatus[] }` to `satisfies Record<OrderStatus, readonly OrderStatus[]>`. Remove the `delivered` row, and compare the errors with the ones you got in Session 16. Where is the first error now?

### Exercise 7

In Mini Example 1, add `findProduct(id: number): Readonly<Product> | undefined` and `deleteProduct(id: number): boolean`. Then add a type `ProductDetails` for a details page: everything except `createdAt`.

### Exercise 8

Write `type Timer = ReturnType<typeof setTimeout>` and a function `remindLater(text: string, ms: number): Timer` that prints the text after `ms` milliseconds. Start two reminders, and cancel the first one with `clearTimeout` before it runs.

---

## Interview Questions

### What are utility types?

Generic types that come with TypeScript and build a new type from an existing one, like `Partial<T>`, `Pick<T, K>` or `Record<K, V>`. They let you write a shape once and build its variants from it.

### What does Partial<T> do, and where is it used?

It makes every property of `T` optional. It is typical for the body of an update request, where only the changed fields are sent, usually as `Partial<Omit<T, "id">>`.

### What is the difference between Pick and Omit?

`Pick<T, K>` keeps only the listed keys; `Omit<T, K>` keeps everything except them. `Pick` checks that the keys exist in `T`, `Omit` does not.

### Why does Omit not report a misspelled key?

Its definition is `Omit<T, K extends keyof any>`, so any key at all is allowed. Removing a key that does not exist is not an error; it simply removes nothing.

### Does Omit<User, "passwordHash"> remove the password from a response?

No. Types are removed before the code runs, and an object with extra properties still fits the type. The property must be removed from the object itself, for example with rest destructuring.

### What is Record<K, V>, and how is it different from an index signature?

An object type with keys `K` and values `V`. `Record<string, V>` is the same as `{ [key: string]: V }`. With a union of literals as `K`, every key is required, and reading a key never gives `undefined`.

### What is the difference between Exclude and Extract?

Both filter a union. `Exclude<U, M>` removes the members that fit `M`; `Extract<U, M>` keeps only those. They also work with object members, like `Extract<Payment, { method: "card" }>`.

### What does NonNullable do?

It removes `null` and `undefined` from a type: `NonNullable<string | null | undefined>` is `string`. It is most useful in generic code.

### How do you get the type of a function's parameters or return value?

With `Parameters<typeof fn>` and `ReturnType<typeof fn>`. `Parameters` gives a tuple, so `Parameters<typeof fn>[0]` is the first parameter. `typeof` is needed because a function is a value.

### Is Readonly<T> deep?

No. It makes the top-level properties readonly, but arrays and objects inside can still change. They need their own `readonly`.

### Why does Partial<T> refuse { stock: undefined } in this project?

Because of `exactOptionalPropertyTypes`: an optional property may be left out, but not set to `undefined`. That protects code like `{ ...product, ...changes }` from overwriting real values with `undefined`.

### Why build types from other types instead of writing them by hand?

There is one source of truth. When the base type changes, every type built from it changes too, and `tsc` points at exactly the code that must be updated.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [16. Enums and Constant Objects](16-enums-and-constant-objects.md) | [Course outline](README.md#course-outline) | [18. Mapped and Conditional Types](18-mapped-and-conditional-types.md) |
