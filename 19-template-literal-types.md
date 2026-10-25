# Template Literal Types

> **Session 19 of 33** | Part 4: Generics and Type Tools | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Template Literals, Now for Types](#template-literals-now-for-types)
- [What ${number} Really Accepts](#what-number-really-accepts)
- [Combining Unions](#combining-unions)
- [Building Strings That Fit](#building-strings-that-fit)
- [Changing Case: Uppercase and Friends](#changing-case-uppercase-and-friends)
- [Routes and Endpoints](#routes-and-endpoints)
- [Strings from Outside](#strings-from-outside)
- [Renaming Keys with as](#renaming-keys-with-as)
- [Taking Strings Apart with infer](#taking-strings-apart-with-infer)
- [Mini Example 1: A Typed Event Bus](#mini-example-1-a-typed-event-bus)
- [Mini Example 2: Typed Route Parameters](#mini-example-2-typed-route-parameters)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `19-template-literal-types` inside `ts-practice`, with the one-line `tsconfig.json`

19-template-literal-types/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## Template Literals, Now for Types

In JavaScript, a template literal builds a string: `` `/orders/${id}` `` ([JavaScript course, Session 08](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/08-strings-and-template-literals.md)). TypeScript lets you write the same thing as a **type**. Then it describes a **format** of string, not one exact string

order-id.ts

```typescript
type OrderId = `ORD-${number}`;

function orderLink(id: OrderId): string {
  return `/orders/${id}`;
}

console.log(orderLink("ORD-1001"));
```

```bash
npx tsc --noEmit
node order-id.ts
```

```text
/orders/ORD-1001
```

`` `ORD-${number}` `` means "the text `ORD-`, followed by a number". Inside a template literal **type**, `${...}` holds a type, not a value: `${number}` is "any number written as text", `${string}` is "any text". Strings that do not fit are refused

```typescript
orderLink("ORD-abc");
orderLink("1001");
orderLink("ord-1001");
```

```text
order-id.ts:8:11 - error TS2345: Argument of type '"ORD-abc"' is not assignable to parameter of type '`ORD-${number}`'.

8 orderLink("ORD-abc");
            ~~~~~~~~~

order-id.ts:9:11 - error TS2345: Argument of type '"1001"' is not assignable to parameter of type '`ORD-${number}`'.

9 orderLink("1001");
            ~~~~~~

order-id.ts:10:11 - error TS2345: Argument of type '"ord-1001"' is not assignable to parameter of type '`ORD-${number}`'.

10 orderLink("ord-1001");
             ~~~~~~~~~~


Found 3 errors in the same file, starting at: order-id.ts:8
```

The fixed part is exact, even its case: `ord-` is not `ORD-`. Without such a type, a function that takes `id: string` accepts all three, and the mistake shows up later as an order that cannot be found.

![Three strings go to orderLink: ORD-1001 fits the pattern ORD- followed by a number, while ORD-abc, 1001 and ord-1001 are refused](images/19-template-literal-types/pattern-check.gif)

---

## What ${number} Really Accepts

`${number}` accepts more than digits. Roughly, it accepts any text that JavaScript can read as a number

number-pattern.ts

```typescript
type OrderId = `ORD-${number}`;

const a: OrderId = "ORD-1.5";
const b: OrderId = "ORD-1e3";
const c: OrderId = "ORD- 5";
const d: OrderId = "ORD-0x1F";
const e: OrderId = "ORD-007";
const f: OrderId = "ORD-";
const g: OrderId = "ORD-Infinity";
```

```text
number-pattern.ts:8:7 - error TS2322: Type '"ORD-"' is not assignable to type '`ORD-${number}`'.

8 const f: OrderId = "ORD-";
        ~

number-pattern.ts:9:7 - error TS2322: Type '"ORD-Infinity"' is not assignable to type '`ORD-${number}`'.

9 const g: OrderId = "ORD-Infinity";
        ~


Found 2 errors in the same file, starting at: number-pattern.ts:8
```

A decimal, `1e3`, a space before the number, a hex number like `0x1F` and leading zeros all fit. Only the empty part and `Infinity` are refused. So a template literal type is a good **first** check that catches typos and mix-ups, not a full validation. When the exact format matters, check the text at run time too, as in [Strings from Outside](#strings-from-outside).

![Seven order ids are tested against ORD- followed by a number: decimals, exponents, a leading space, hex and leading zeros fit, only an empty part and Infinity are refused](images/19-template-literal-types/number-pattern.gif)

---

## Combining Unions

Put a union inside a template literal type, and you get every combination. A T-shirt shop has codes for each size and colour

sku.ts

```typescript
type Size = "S" | "M" | "L";
type Color = "black" | "white";
type Sku = `TSHIRT-${Size}-${Color}`;

const sizes: Size[] = ["S", "M", "L"];
const colors: Color[] = ["black", "white"];
const skus: Sku[] = [];

for (const size of sizes) {
  for (const color of colors) {
    skus.push(`TSHIRT-${size}-${color}`);
  }
}

console.log(skus);
```

```bash
npx tsc --noEmit
node sku.ts
```

```text
[
  'TSHIRT-S-black',
  'TSHIRT-S-white',
  'TSHIRT-M-black',
  'TSHIRT-M-white',
  'TSHIRT-L-black',
  'TSHIRT-L-white'
]
```

3 sizes times 2 colours gives 6 codes, in the type and at run time. A code that does not exist is refused, with a suggestion

```typescript
const extra: Sku = "TSHIRT-XL-black";
```

```text
sku.ts:16:7 - error TS2820: Type '"TSHIRT-XL-black"' is not assignable to type '"TSHIRT-L-black" | "TSHIRT-L-white" | "TSHIRT-M-black" | "TSHIRT-M-white" | "TSHIRT-S-black" | "TSHIRT-S-white"'. Did you mean '"TSHIRT-L-black"'?

16 const extra: Sku = "TSHIRT-XL-black";
         ~~~~~


Found 1 error in sku.ts:16
```

The message lists all six members, because `Sku` is now a plain union of literals (Session 09). Add `"XL"` to `Size`, and the code is accepted, with no other change.

Unions multiply quickly. A PIN of six digits, written as six `Digit` unions in a row, would be a million members, and `tsc` refuses: `error TS2590: Expression produces a union type that is too complex to represent.` Use `${number}` or `${string}` for open-ended parts.

![Three sizes and two colours combine into six T-shirt codes; a code with size XL is not one of them](images/19-template-literal-types/combine-unions.gif)

---

## Building Strings That Fit

In the loop above, `` `TSHIRT-${size}-${color}` `` is a normal JavaScript template literal. Because `skus.push` expects a `Sku`, TypeScript checks the template literal against that type: `size` is a `Size` and `color` is a `Color`, so the result is one of the six codes.

The check needs that expected type. Save the code in a variable first

```typescript
    const code = `TSHIRT-${size}-${color}`;
    skus.push(code);
```

```text
sku.ts:12:15 - error TS2345: Argument of type 'string' is not assignable to parameter of type '"TSHIRT-L-black" | "TSHIRT-L-white" | "TSHIRT-M-black" | "TSHIRT-M-white" | "TSHIRT-S-black" | "TSHIRT-S-white"'.

12     skus.push(code);
                 ~~~~


Found 1 error in sku.ts:12
```

Without a type to aim at, a template literal in the code is a plain `string`. Annotate the variable, `const code: Sku = ...`, or build the string where the type is expected.

![A template literal built inside skus.push is checked against Sku and fits; the same template literal saved in a variable first becomes a plain string and is refused](images/19-template-literal-types/build-strings.gif)

---

## Changing Case: Uppercase and Friends

TypeScript has four built-in types that change the case of string literal types

case.ts

```typescript
const a: Uppercase<"get" | "post"> = 0;
const b: Lowercase<"GET"> = 0;
const c: Capitalize<"order"> = 0;
const d: Uncapitalize<"Order"> = 0;
```

```text
case.ts:1:7 - error TS2322: Type '0' is not assignable to type '"GET" | "POST"'.

1 const a: Uppercase<"get" | "post"> = 0;
        ~

case.ts:2:7 - error TS2322: Type '0' is not assignable to type '"get"'.

2 const b: Lowercase<"GET"> = 0;
        ~

case.ts:3:7 - error TS2322: Type '0' is not assignable to type '"Order"'.

3 const c: Capitalize<"order"> = 0;
        ~

case.ts:4:7 - error TS2322: Type '0' is not assignable to type '"order"'.

4 const d: Uncapitalize<"Order"> = 0;
        ~


Found 4 errors in the same file, starting at: case.ts:1
```

The "surely wrong value" trick from Session 18 shows the results: `Uppercase` and `Lowercase` change every letter, `Capitalize` and `Uncapitalize` only the first one. With a union, each member is changed.

These are types only. They do not change any string at run time, and the string methods do not know about them

upper.ts

```typescript
function toUpperMethod<M extends string>(method: M): Uppercase<M> {
  return method.toUpperCase();
}
```

```text
upper.ts:2:3 - error TS2322: Type 'string' is not assignable to type 'Uppercase<M>'.

2   return method.toUpperCase();
    ~~~~~~


Found 1 error in upper.ts:2
```

`toUpperCase()` is declared to return `string`. Here an assertion (Session 13) is fine, because the code really does what the type says

```typescript
function toUpperMethod<M extends string>(method: M): Uppercase<M> {
  return method.toUpperCase() as Uppercase<M>;
}

const method: "POST" = toUpperMethod("post");
console.log(method);
```

```text
POST
```

`toUpperMethod("post")` has the type `"POST"`, so it fits a variable of type `"POST"`.

![Four case types change string literal types: get becomes GET, GET becomes get, order becomes Order, Order becomes order; toUpperCase alone returns a plain string](images/19-template-literal-types/change-case.gif)

---

## Routes and Endpoints

Template literal types fit backend strings well: paths, endpoints and event names

endpoints.ts

```typescript
type Method = "GET" | "POST" | "DELETE";
type ApiPath = `/api/${string}`;
type Endpoint = `${Method} ${ApiPath}`;

const endpoints: Endpoint[] = [
  "GET /api/orders",
  "POST /api/orders",
  "DELETE /api/orders/1001"
];

console.log(endpoints);
```

```bash
npx tsc --noEmit
node endpoints.ts
```

```text
[ 'GET /api/orders', 'POST /api/orders', 'DELETE /api/orders/1001' ]
```

An endpoint is a method, a space, and a path that starts with `/api/`. Add three wrong ones to the list

```typescript
  "GET /orders",
  "PATCH /api/orders/1001",
  "get /api/orders"
```

```text
endpoints.ts:9:3 - error TS2322: Type '"GET /orders"' is not assignable to type '`DELETE /api/${string}` | `GET /api/${string}` | `POST /api/${string}`'.

9   "GET /orders",
    ~~~~~~~~~~~~~

endpoints.ts:10:3 - error TS2322: Type '"PATCH /api/orders/1001"' is not assignable to type '`DELETE /api/${string}` | `GET /api/${string}` | `POST /api/${string}`'.

10   "PATCH /api/orders/1001",
     ~~~~~~~~~~~~~~~~~~~~~~~~

endpoints.ts:11:3 - error TS2322: Type '"get /api/orders"' is not assignable to type '`DELETE /api/${string}` | `GET /api/${string}` | `POST /api/${string}`'.

11   "get /api/orders"
     ~~~~~~~~~~~~~~~~~


Found 3 errors in the same file, starting at: endpoints.ts:9
```

The message shows how TypeScript sees `Endpoint`: one pattern for each method. A missing `/api`, an unknown method and a lowercase method are all caught.

![An endpoint type is built from a method union and an api path pattern; GET /api/orders fits, while GET /orders, PATCH and lowercase get are refused](images/19-template-literal-types/endpoints.gif)

---

## Strings from Outside

Text from a URL, a file or a request body has the type `string`. TypeScript cannot know whether it fits `` `ORD-${number}` ``

order-guard.ts

```typescript
type OrderId = `ORD-${number}`;

function isOrderId(text: string): text is OrderId {
  return /^ORD-\d+$/.test(text);
}

function orderLink(id: OrderId): string {
  return `/orders/${id}`;
}

const inputs: string[] = ["ORD-1001", "ORD-abc", "1002"];

for (const input of inputs) {
  if (isOrderId(input)) {
    console.log(orderLink(input));
  } else {
    console.log(`Not an order id: ${input}`);
  }
}
```

```bash
npx tsc --noEmit
node order-guard.ts
```

```text
/orders/ORD-1001
Not an order id: ORD-abc
Not an order id: 1002
```

`isOrderId` is a type predicate (Session 10). The regular expression ([JavaScript course, Session 08](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/08-strings-and-template-literals.md)) means "starts with `ORD-`, then one or more digits, and nothing else". It is stricter than the type: `ORD-1.5` fits the type but not the check. That is the safe direction. A check that lets through text the type does not allow would be a lie to `tsc`.

Without the check, `orderLink(input)` is refused

```text
order-guard.ts:14:25 - error TS2345: Argument of type 'string' is not assignable to parameter of type '`ORD-${number}`'.

14   console.log(orderLink(input));
                           ~~~~~


Found 1 error in order-guard.ts:14
```

![Strings from outside are plain text; the isOrderId check lets ORD-1001 through as an OrderId and stops ORD-abc and 1002](images/19-template-literal-types/strings-from-outside.gif)

---

## Renaming Keys with as

Session 18 used `as` in a mapped type to **filter** keys. With a template literal type, `as` can **rename** them. A classic example makes a getter for every property

getters.ts

```typescript
interface Product {
  name: string;
  pricePaise: number;
}

type Getters<T> = {
  [K in keyof T as `get${Capitalize<K & string>}`]: () => T[K];
};

const product: Product = { name: "Keyboard", pricePaise: 149900 };

const getters: Getters<Product> = {
  getName: () => product.name,
  getPricePaise: () => product.pricePaise
};

console.log(getters.getName(), getters.getPricePaise());
```

```bash
npx tsc --noEmit
node getters.ts
```

```text
Keyboard 149900
```

For each key `K`, the new key is `get` plus `K` with a capital first letter: `name` becomes `getName`. The value is a function that returns the old type, `T[K]`. The `{}` trick shows the new keys

```typescript
const check: Getters<Product> = {};
```

```text
getters.ts:12:7 - error TS2739: Type '{}' is missing the following properties from type 'Getters<Product>': getName, getPricePaise

12 const check: Getters<Product> = {};
         ~~~~~


Found 1 error in getters.ts:12
```

(This replaces everything after the `product` line.)

Why `K & string`? Remove it, and write `Capitalize<K>`

```text
getters.ts:7:37 - error TS2344: Type 'K' does not satisfy the constraint 'string'.
  Type 'keyof T' is not assignable to type 'string'.
    Type 'string | number | symbol' is not assignable to type 'string'.
      Type 'number' is not assignable to type 'string'.

7   [K in keyof T as `get${Capitalize<K>}`]: () => T[K];
                                      ~


Found 1 error in getters.ts:7
```

Keys can be numbers and symbols too (Session 15), but `Capitalize` only works on strings. `K & string` (an intersection, Session 08) keeps only the string keys.

### A backend example: environment variable names

Configuration often comes from environment variables, with a prefix and in upper case. The names can be built from the config type

db-config.ts

```typescript
interface DbConfig {
  host: string;
  port: number;
  user: string;
  name: string;
}

type EnvNames<T, Prefix extends string> = {
  [K in keyof T as `${Prefix}_${Uppercase<K & string>}`]?: string;
};

type DbEnv = EnvNames<DbConfig, "DB">;

function readDbConfig(env: DbEnv): DbConfig {
  return {
    host: env.DB_HOST ?? "localhost",
    port: Number(env.DB_PORT ?? "27017"),
    user: env.DB_USER ?? "app",
    name: env.DB_NAME ?? "shop"
  };
}

console.log(readDbConfig(process.env));
```

.env

```text
DB_HOST=db.internal
DB_PORT=27018
```

```bash
npx tsc --noEmit
node --env-file=.env db-config.ts
```

```text
{ host: 'db.internal', port: 27018, user: 'app', name: 'shop' }
```

`DbEnv` has the optional keys `DB_HOST`, `DB_PORT`, `DB_USER` and `DB_NAME`, each a `string`. `process.env` fits, because it can hold any name. `node --env-file=.env` loads the file, as in Session 10. A misspelled variable name is caught

```text
db-config.ts:16:15 - error TS2551: Property 'DB_HOTS' does not exist on type 'EnvNames<DbConfig, "DB">'. Did you mean 'DB_HOST'?

16     host: env.DB_HOTS ?? "localhost",
                 ~~~~~~~


Found 1 error in db-config.ts:16
```

When `DbConfig` gets a new setting, `DbEnv` gets the new variable name by itself.

![The keys of DbConfig are renamed one by one: host becomes DB_HOST, port becomes DB_PORT, and so on; getters turn name into getName](images/19-template-literal-types/rename-keys.gif)

---

## Taking Strings Apart with infer

`infer` (Session 18) also works inside template literal types. Then it takes **text** apart

split.ts

```typescript
type EntityOf<E> = E extends `${infer Entity}:${string}` ? Entity : never;
type ActionOf<E> = E extends `${string}:${infer Action}` ? Action : never;

const a: EntityOf<"order:created"> = 0;
const b: ActionOf<"order:created"> = 0;
const c: EntityOf<"order:created" | "user:registered"> = 0;
const d: EntityOf<"ordercreated"> = 0;
```

```text
split.ts:4:7 - error TS2322: Type '0' is not assignable to type '"order"'.

4 const a: EntityOf<"order:created"> = 0;
        ~

split.ts:5:7 - error TS2322: Type '0' is not assignable to type '"created"'.

5 const b: ActionOf<"order:created"> = 0;
        ~

split.ts:6:7 - error TS2322: Type '0' is not assignable to type '"order" | "user"'.

6 const c: EntityOf<"order:created" | "user:registered"> = 0;
        ~

split.ts:7:7 - error TS2322: Type '0' is not assignable to type 'never'.

7 const d: EntityOf<"ordercreated"> = 0;
        ~


Found 4 errors in the same file, starting at: split.ts:4
```

- `` `${infer Entity}:${string}` `` means "some text, a colon, more text; call the first part `Entity`".
- With a union, each member is taken apart on its own (distribution, Session 18).
- `"ordercreated"` has no colon, the pattern does not match, and the result is `never`.

Mini Example 2 uses this idea, plus recursion, to read the parameters out of a route.

![The text order:created is split at the colon into the entity order and the action created; a union is split member by member, and text without a colon gives never](images/19-template-literal-types/split-infer.gif)

---

## Mini Example 1: A Typed Event Bus

Large backends often send events: "an order was created", "a user registered". Other parts of the code listen and react, for example by sending an email. Event names like `order:created` are strings, so mistakes are easy. One interface ties every name to the data that comes with it

event-bus.ts

```typescript
interface EventMap {
  "order:created": { orderId: number; totalPaise: number };
  "order:paid": { orderId: number; method: "card" | "upi" };
  "user:registered": { userId: number; email: string };
}

type EventName = keyof EventMap;
type OrderEvent = Extract<EventName, `order:${string}`>;
type Handler<E extends EventName> = (payload: EventMap[E]) => void;

const handlers: { [E in EventName]: Handler<E>[] } = {
  "order:created": [],
  "order:paid": [],
  "user:registered": []
};

function on<E extends EventName>(event: E, handler: Handler<E>): void {
  handlers[event].push(handler);
}

function emit<E extends EventName>(event: E, payload: EventMap[E]): void {
  for (const handler of handlers[event]) {
    handler(payload);
  }
}

function logOrderEvents(events: OrderEvent[]): void {
  for (const event of events) {
    on(event, (payload) => console.log(`[log] ${event} for order ${payload.orderId}`));
  }
}

on("order:created", (payload) => console.log(`New order ${payload.orderId}: Rs. ${payload.totalPaise / 100}`));
on("order:paid", (payload) => console.log(`Order ${payload.orderId} paid by ${payload.method}`));
on("user:registered", (payload) => console.log(`Welcome email to ${payload.email}`));
logOrderEvents(["order:created", "order:paid"]);

emit("order:created", { orderId: 1001, totalPaise: 149900 });
emit("order:paid", { orderId: 1001, method: "upi" });
emit("user:registered", { userId: 7, email: "asha@example.com" });
```

```bash
npx tsc --noEmit
node event-bus.ts
```

```text
New order 1001: Rs. 1499
[log] order:created for order 1001
Order 1001 paid by upi
[log] order:paid for order 1001
Welcome email to asha@example.com
```

| Code                                                   | From this session and earlier                                     |
| ------------------------------------------------------ | ----------------------------------------------------------------- |
| `interface EventMap { "order:created": {...} }`        | Keys can be any text in quotes (Session 07)                       |
| ``Extract<EventName, `order:${string}`>``              | All event names that start with `order:` (Session 17)             |
| `Handler<E>` with `EventMap[E]`                        | The payload type for one event (Sessions 14 and 15)               |
| `{ [E in EventName]: Handler<E>[] }`                   | A list of handlers for every event (Session 18)                   |
| `on<E extends EventName>(event: E, handler: Handler<E>)` | `E` connects the name and the handler (Session 14)              |
| `(payload) => ...`                                     | The payload type comes from the event name: no annotation needed  |

Every mistake with names and payloads is caught

```typescript
emit("order:shipped", { orderId: 1001 });
emit("order:paid", { orderId: 1001, method: "cash" });
on("order:paid", (payload) => console.log(payload.email));
logOrderEvents(["order:created", "user:registered"]);
```

```text
event-bus.ts:42:6 - error TS2345: Argument of type '"order:shipped"' is not assignable to parameter of type 'keyof EventMap'.

42 emit("order:shipped", { orderId: 1001 });
        ~~~~~~~~~~~~~~~

event-bus.ts:43:37 - error TS2322: Type '"cash"' is not assignable to type '"card" | "upi"'.

43 emit("order:paid", { orderId: 1001, method: "cash" });
                                       ~~~~~~

  event-bus.ts:3:36 - The expected type comes from property 'method' which is declared here on type '{ orderId: number; method: "card" | "upi"; }'
    3   "order:paid": { orderId: number; method: "card" | "upi" };
                                         ~~~~~~

event-bus.ts:44:51 - error TS2339: Property 'email' does not exist on type '{ orderId: number; method: "card" | "upi"; }'.

44 on("order:paid", (payload) => console.log(payload.email));
                                                     ~~~~~

event-bus.ts:45:34 - error TS2322: Type '"user:registered"' is not assignable to type 'OrderEvent'.

45 logOrderEvents(["order:created", "user:registered"]);
                                    ~~~~~~~~~~~~~~~~~


Found 4 errors in the same file, starting at: event-bus.ts:42
```

1. An event that does not exist.
2. A payment method that does not exist.
3. A property that the payload of this event does not have.
4. A user event where only order events are allowed.

Now the shop adds refunds. Remove the four lines, and add a new event to `EventMap`, after `order:paid`

```typescript
  "order:refunded": { refundId: number; amountPaise: number };
```

```text
event-bus.ts:12:7 - error TS2741: Property '"order:refunded"' is missing in type '{ "order:created": never[]; "order:paid": never[]; "user:registered": never[]; }' but required in type '{ "order:created": Handler<"order:created">[]; "order:paid": Handler<"order:paid">[]; "order:refunded": Handler<"order:refunded">[]; "user:registered": Handler<"user:registered">[]; }'.

12 const handlers: { [E in EventName]: Handler<E>[] } = {
         ~~~~~~~~

event-bus.ts:30:76 - error TS2339: Property 'orderId' does not exist on type '{ orderId: number; totalPaise: number; } | { orderId: number; method: "card" | "upi"; } | { refundId: number; amountPaise: number; }'.
  Property 'orderId' does not exist on type '{ refundId: number; amountPaise: number; }'.

30     on(event, (payload) => console.log(`[log] ${event} for order ${payload.orderId}`));
                                                                              ~~~~~~~


Found 2 errors in the same file, starting at: event-bus.ts:12
```

Two useful errors:

1. The handler table has no list for the new event.
2. `OrderEvent` now includes `order:refunded` by itself, because it starts with `order:`. The order log reads `payload.orderId`, and a refund has no `orderId`.

The second one is a real design question: a refund belongs to an order, so it should say which one. Give the refund an `orderId`, and add its list to the table

```typescript
  "order:refunded": { orderId: number; amountPaise: number };
```

```typescript
  "order:paid": [],
  "order:refunded": [],
```

and `tsc` is quiet again. Node.js has a built-in `EventEmitter` that works like this bus ([Node.js course, Session 08](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/08-events-and-eventemitter.md)), and the same typing ideas apply there.

![Event names are tied to payload types; emitting order:paid with method cash is refused, and adding order:refunded makes the handler table and the order log ask for changes](images/19-template-literal-types/event-bus.gif)

---

## Mini Example 2: Typed Route Parameters

Routes like `/orders/:orderId/items/:itemId` have **parameters**: the parts after a colon. A function that fills them in should ask for exactly those parameters. The type can read them out of the route text

build-path.ts

```typescript
type RouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | RouteParams<`/${Rest}`>
    : Path extends `${string}:${infer Param}`
      ? Param
      : never;

type ParamsOf<Path extends string> = {
  [K in RouteParams<Path>]: string;
};

function buildPath<Path extends string>(path: Path, params: ParamsOf<Path>): string {
  let result: string = path;
  for (const [name, value] of Object.entries<string>(params)) {
    result = result.replace(`:${name}`, value);
  }
  return result;
}

console.log(buildPath("/orders/:orderId", { orderId: "1001" }));
console.log(buildPath("/orders/:orderId/items/:itemId", { orderId: "1001", itemId: "7" }));
console.log(buildPath("/health", {}));
```

```bash
npx tsc --noEmit
node build-path.ts
```

```text
/orders/1001
/orders/1001/items/7
/health
```

`RouteParams` calls itself, like a recursive function ([JavaScript course, Session 10](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/10-functions.md)). Follow `"/orders/:orderId/items/:itemId"`:

| Step | Path checked                       | Matches                                   | Result so far                     |
| ---- | ---------------------------------- | ----------------------------------------- | --------------------------------- |
| 1    | `/orders/:orderId/items/:itemId`   | `:` + `orderId` + `/` + `items/:itemId`   | `"orderId" \| RouteParams<"/items/:itemId">` |
| 2    | `/items/:itemId`                   | No `/` after the parameter; the second pattern: `:` + `itemId` | `"itemId"`   |
| 3    | (done)                             |                                           | `"orderId" \| "itemId"`           |

For `"/health"` neither pattern matches, so the result is `never`, and `ParamsOf<"/health">` is `{}`. Check the results with the tricks from Session 18: replace the three `console.log` lines with

```typescript
const a: RouteParams<"/orders/:orderId/items/:itemId"> = 0;
const b: RouteParams<"/health"> = 0;
const c: ParamsOf<"/orders/:orderId/items/:itemId"> = {};
```

```text
build-path.ts:20:7 - error TS2322: Type '0' is not assignable to type '"itemId" | "orderId"'.

20 const a: RouteParams<"/orders/:orderId/items/:itemId"> = 0;
         ~

build-path.ts:21:7 - error TS2322: Type '0' is not assignable to type 'never'.

21 const b: RouteParams<"/health"> = 0;
         ~

build-path.ts:22:7 - error TS2739: Type '{}' is missing the following properties from type 'ParamsOf<"/orders/:orderId/items/:itemId">': itemId, orderId

22 const c: ParamsOf<"/orders/:orderId/items/:itemId"> = {};
         ~


Found 3 errors in the same file, starting at: build-path.ts:20
```

| Code                                   | From this session and earlier                                        |
| -------------------------------------- | -------------------------------------------------------------------- |
| `` `${string}:${infer Param}/${infer Rest}` `` | Text, a colon, the parameter, a slash, the rest              |
| `` RouteParams<`/${Rest}`> ``          | The type calls itself on the rest of the route                       |
| `[K in RouteParams<Path>]: string`     | One text value for each parameter (Session 18)                       |
| `buildPath<Path extends string>`       | `Path` is inferred as the exact literal route (Session 14)           |
| `Object.entries<string>(params)`       | Without `<string>`, `value` is `unknown` for this generic type, and `replace` refuses it (TS2769) |

Now the mistakes. Put the three `console.log` lines back instead of the test lines, and add after them

```typescript
buildPath("/orders/:orderId/items/:itemId", { orderId: "1001" });
buildPath("/orders/:orderId", { orderID: "1001" });
buildPath("/orders/:orderId", { orderId: 1001 });
```

```text
build-path.ts:24:45 - error TS2741: Property 'itemId' is missing in type '{ orderId: string; }' but required in type 'ParamsOf<"/orders/:orderId/items/:itemId">'.

24 buildPath("/orders/:orderId/items/:itemId", { orderId: "1001" });
                                               ~~~~~~~~~~~~~~~~~~~

build-path.ts:25:33 - error TS2561: Object literal may only specify known properties, but 'orderID' does not exist in type 'ParamsOf<"/orders/:orderId">'. Did you mean to write 'orderId'?

25 buildPath("/orders/:orderId", { orderID: "1001" });
                                   ~~~~~~~

build-path.ts:26:33 - error TS2322: Type 'number' is not assignable to type 'string'.

26 buildPath("/orders/:orderId", { orderId: 1001 });
                                   ~~~~~~~

  build-path.ts:8:38 - The expected type comes from property 'orderId' which is declared here on type 'ParamsOf<"/orders/:orderId">'
     8 type ParamsOf<Path extends string> = {
                                            ~
     9   [K in RouteParams<Path>]: string;
       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    10 };
       ~


Found 3 errors in the same file, starting at: build-path.ts:24
```

A missing parameter, a misspelled one (with a suggestion), and a number where the URL needs text.

One limit: the route must be a literal in the code. A route stored as a plain `string` has no known parameters, so nothing is checked

```typescript
const fromConfig: string = "/orders/:orderId";
buildPath(fromConfig, {});
```

This passes `tsc`, and the result would still contain `:orderId`.

Express does the same thing in its type definitions (the `@types/express` package and the packages it uses): the type of `req.params` is read out of the route text with template literal types, `infer` and recursion, like `RouteParams` here. Session 31 shows it in a real Express app.

![The route text is taken apart step by step: orderId is read first, then the rest of the route gives itemId; buildPath asks for exactly these two parameters](images/19-template-literal-types/route-params.gif)

---

## Beginner Mistakes

### Mistake 1: Writing the type with quotes instead of backticks

`type OrderId = "ORD-${number}"` is a plain string literal type that contains the characters `${number}`. `"ORD-1001"` then gives `error TS2322: Type '"ORD-1001"' is not assignable to type '"ORD-${number}"'.` Use backticks.

---

### Mistake 2: Expecting ${number} to mean digits only

`"ORD-1.5"`, `"ORD-1e3"`, `"ORD- 5"` and `"ORD-0x1F"` all fit `` `ORD-${number}` ``. For an exact format, also check the text at run time.

---

### Mistake 3: Passing a plain string

Text from outside is `string`, and `orderLink(input)` gives `error TS2345`. Check it first with a type predicate like `isOrderId`.

---

### Mistake 4: A type predicate that is looser than the type

`return text.startsWith("ORD-")` would let `"ORD-abc"` through as an `OrderId`. Like the lying predicate in Session 10, `tsc` believes it. Make the check at least as strict as the type.

---

### Mistake 5: Building the string in a variable first

``const code = `TSHIRT-${size}-${color}` `` is a plain `string`, and `skus.push(code)` gave `error TS2345`. Build the string where the type is expected, or annotate the variable.

---

### Mistake 6: Expecting toUpperCase to return Uppercase<T>

String methods return `string`. `return method.toUpperCase()` in a function typed `Uppercase<M>` gave `error TS2322`. After a real conversion, `as Uppercase<M>` is fine.

---

### Mistake 7: Forgetting & string when renaming keys

`Capitalize<K>` with `K in keyof T` gives `error TS2344: Type 'K' does not satisfy the constraint 'string'.` Keys may be numbers or symbols. Write `Capitalize<K & string>`.

---

### Mistake 8: Combining too many unions

Six `Digit` unions in a row make a million members, and `tsc` stops with `error TS2590`. Use `${number}` or `${string}` for parts that can be anything.

---

### Mistake 9: Expecting a plain string route to be checked

`buildPath(fromConfig, {})` with `fromConfig: string` passed `tsc`, although a parameter was missing. The parameters can only be read from a literal route in the code.

---

### Mistake 10: Turning every string into a template literal type

Use them for real formats that matter: ids, routes, event names, environment variable names. A name or an address does not need one, and very clever string types are hard to read.

---

## Practice Exercises

### Exercise 1

Write `` type Percent = `${number}%` `` and test `"15%"`, `"15"` and `"%"`. Then write `isPercent(text: string): text is Percent` with a regular expression, and use it on a list of inputs.

### Exercise 2

Add a sleeve type `"half" | "full"` to the T-shirt codes: `` `TSHIRT-${Size}-${Color}-${Sleeve}` ``. How many codes are there? Check your answer with the "surely wrong value" trick, and update the loop to build them all.

### Exercise 3

Write `` type VersionedPath = `/v${number}/${string}` `` for API paths like `/v1/orders`. Which of `/v1/orders`, `/v2.1/orders`, `/orders` and `/vX/orders` fit? Explain each.

### Exercise 4

Write `Setters<T>`, which turns every key into `set` plus the capitalized key, with the type `(value: T[K]) => void`. Create setters for a `Product` object and use them.

### Exercise 5

Use `EnvNames` for a mail configuration `{ host: string; port: number; from: string }` with the prefix `"MAIL"`. Write `readMailConfig`, put the variables in a `.env` file, and run it with `node --env-file=.env`.

### Exercise 6

Write `` type Extension<F> = F extends `${string}.${infer Ext}` ? Ext : never `` and test it with `"report.pdf"`, `"photo.final.jpg"` and `"README"`. Explain the result for the second file name.

### Exercise 7

In Mini Example 1, add a `"user:deleted"` event with `{ userId: number }`. Follow the errors. Then write `logUserEvents` with `` Extract<EventName, `user:${string}`> `` that logs the `userId` of every user event.

### Exercise 8

In Mini Example 2, test a route with a parameter at the start, `/:shop/orders/:orderId`. Then write `buildUrl(path, params)` that puts `https://shop.example.com` in front of the result of `buildPath`, with the same typed parameters.

---

## Interview Questions

### What is a template literal type?

A string type written with backticks, like `` `ORD-${number}` ``. Inside `${}` there is a type, and the result describes a format of string instead of one exact string.

### What happens when you put a union inside a template literal type?

TypeScript makes every combination. `` `${"S" | "M"}-${"black" | "white"}` `` is a union of four literal types. Very large combinations give TS2590.

### What does ${number} accept?

Text that JavaScript can read as a number, not only digits: decimals, exponents like `1e3`, hex numbers and leading spaces fit. An empty part and `Infinity` do not.

### Are template literal types checked at run time?

No. Like all types, they are removed before the code runs. Text from outside is a `string` and must be checked in code, for example with a type predicate and a regular expression.

### How do you turn a plain string into a template literal type safely?

With a type predicate: `function isOrderId(text: string): text is OrderId` that checks the text and returns a boolean. Inside `if (isOrderId(text))`, the text has the narrower type.

### What are Uppercase, Lowercase, Capitalize and Uncapitalize?

Built-in types that change the case of string literal types: `Uppercase<"get">` is `"GET"`, `Capitalize<"order">` is `"Order"`. They change types, not values.

### Why does toUpperCase() not return Uppercase<T>?

String methods are declared to return `string`. A generic function that really converts the text can use `as Uppercase<T>` for its return value.

### How do you rename keys in a mapped type?

With `as` and a template literal type: `` [K in keyof T as `get${Capitalize<K & string>}`]: () => T[K] `` turns `name` into `getName`.

### Why is K & string needed there?

`keyof T` can include number and symbol keys, but `Capitalize` and template literal types need strings. `K & string` keeps only the string keys.

### How can infer take a string apart?

Inside a template literal pattern: `` E extends `${infer Entity}:${string}` ? Entity : never `` gives `"order"` for `"order:created"`.

### How can a type find the parameters of a route like /orders/:orderId?

With a recursive conditional type that matches `` `${string}:${infer Param}/${infer Rest}` ``, keeps `Param`, and calls itself on the rest. Express's types use this for `req.params`.

### What are the limits of template literal types?

They only check literal strings known in the code; a plain `string` gets no checking. Patterns like `${number}` are looser than they look. Too many combinations hit a size limit. And complex string types are hard for teammates to read.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [18. Mapped and Conditional Types](18-mapped-and-conditional-types.md) | [Course outline](README.md#course-outline) | [20. Classes in TypeScript](20-classes.md) |
