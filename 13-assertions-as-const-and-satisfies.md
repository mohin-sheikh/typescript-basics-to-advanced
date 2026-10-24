# Type Assertions, as const and satisfies

> **Session 13 of 33** | Part 3: Functions | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Telling tsc What You Know: as](#telling-tsc-what-you-know-as)
- [as Is Not a Check](#as-is-not-a-check)
- [as Does Not Change the Value](#as-does-not-change-the-value)
- [When tsc Refuses an as](#when-tsc-refuses-an-as)
- [When as Is the Right Tool](#when-as-is-the-right-tool)
- [Assertion Functions: Check, Then Trust](#assertion-functions-check-then-trust)
- [The "Trust Me" Family](#the-trust-me-family)
- [as const](#as-const)
- [as const for Tuples](#as-const-for-tuples)
- [as const Stops Widening](#as-const-stops-widening)
- [satisfies: Check Without Changing the Type](#satisfies-check-without-changing-the-type)
- [Annotation, as, satisfies or as const?](#annotation-as-satisfies-or-as-const)
- [Mini Example 1: Loading a Config File](#mini-example-1-loading-a-config-file)
- [Mini Example 2: A Route Table with satisfies](#mini-example-2-a-route-table-with-satisfies)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `13-assertions` inside `ts-practice`, with the one-line `tsconfig.json`

13-assertions/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## Telling tsc What You Know: as

`JSON.parse` returns `any` (Session 04). A very common line in real projects tells `tsc` what the data is

as-basic.ts

```typescript
interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

const body = '{ "id": 1001, "customer": "Riya", "totalPaise": 349700 }';
const order = JSON.parse(body) as Order;

console.log(`${order.customer.toUpperCase()} owes Rs. ${order.totalPaise / 100}`);
```

```bash
npx tsc --noEmit
node as-basic.ts
```

```text
RIYA owes Rs. 3497
```

`value as Order` is a **type assertion**: "treat this value as an `Order`". From that line on, `order` has the type `Order`, with autocomplete and checks. Like every type, the `as Order` disappears when the code runs.

The important word is **assertion**: you claim something, and `tsc` believes you. Here is the same code with data that does not match

as-lie.ts

```typescript
interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

const body = '{ "id": 1002, "name": "Sara", "totalPaise": "49900" }';
const order = JSON.parse(body) as Order;

console.log(`Total with shipping: ${order.totalPaise + 4000}`);
console.log(order.customer.toUpperCase());
```

```text
Total with shipping: 499004000
file:///D:/ts-practice/13-assertions/as-lie.ts:11
console.log(order.customer.toUpperCase());
                           ^

TypeError: Cannot read properties of undefined (reading 'toUpperCase')
    at file:///D:/ts-practice/13-assertions/as-lie.ts:11:28
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`npx tsc --noEmit` reported nothing. The total is the text `"49900"` joined with `4000`, and `customer` does not exist (the data says `name`). The assertion was a lie, and `tsc` had no way to know.

![as Order is a label stuck on a box: the box from JSON.parse is not opened or checked. With good data the label is true; with bad data the label still says Order, and the code crashes later](images/13-assertions-as-const-and-satisfies/as-label.gif)

---

## as Is Not a Check

Compare an annotation with an assertion on the same incomplete object

as-vs-annotation.ts

```typescript
interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

const typed: Order = { id: 1003 };
const asserted = { id: 1003 } as Order;
```

```text
as-vs-annotation.ts:7:7 - error TS2739: Type '{ id: number; }' is missing the following properties from type 'Order': customer, totalPaise

7 const typed: Order = { id: 1003 };
        ~~~~~


Found 1 error in as-vs-annotation.ts:7
```

Only the annotation is an error. The assertion passed, although two properties are missing. Use it, and Node.js finds out

as-run.ts

```typescript
interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

const asserted = { id: 1003 } as Order;
console.log(asserted.customer.toUpperCase());
```

```text
file:///D:/ts-practice/13-assertions/as-run.ts:8
console.log(asserted.customer.toUpperCase());
                              ^

TypeError: Cannot read properties of undefined (reading 'toUpperCase')
    at file:///D:/ts-practice/13-assertions/as-run.ts:8:31
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

| Code                               | What `tsc` checks                               |
| ---------------------------------- | ----------------------------------------------- |
| `const typed: Order = value`       | The value must be a full `Order`                |
| `const asserted = value as Order`  | Only that the two types **could** be related    |

When you make an object yourself, always use an annotation.

![The object { id: 1003 } goes to two doors: the annotation door : Order checks it and refuses it with TS2739; the as Order door only looks at the label and lets it through, and it crashes at customer.toUpperCase()](images/13-assertions-as-const-and-satisfies/as-vs-annotation.gif)

---

## as Does Not Change the Value

An assertion is not a conversion. It changes what `tsc` thinks, never the value

as-double.ts

```typescript
const quantity = "3" as unknown as number;
console.log(quantity + 1);
```

```text
31
```

`tsc` thinks `quantity` is a number, so `quantity + 1` should be `4`. At run time it is still the text `"3"`, and `+` joins texts. To really turn a text into a number, use `Number("3")`.

The double `as unknown as` is explained in the next section. When you see it in code, read it as a warning sign.

![The text "3" gets the label number with as unknown as number; at run time the label is gone, the value is still "3", and "3" + 1 gives "31"](images/13-assertions-as-const-and-satisfies/as-no-conversion.gif)

---

## When tsc Refuses an as

`tsc` does a small check: the two types must overlap

as-refused.ts

```typescript
const quantity = "3" as number;
```

```text
as-refused.ts:1:18 - error TS2352: Conversion of type 'string' to type 'number' may be a mistake because neither type sufficiently overlaps with the other. If this was intentional, convert the expression to 'unknown' first.

1 const quantity = "3" as number;
                   ~~~~~~~~~~~~~


Found 1 error in as-refused.ts:1
```

No string is ever a number, so the assertion is certainly wrong. The message also shows the way around the check: assert to `unknown` first, because everything overlaps with `unknown`. That is what `as unknown as number` did, and why it is a warning sign: it switches off even this small check.

The overlap check is weak. These all pass

as-ok.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

const text: string = "paid";
const status = text as OrderStatus;
const back = status as string;
const wrong = "delivered" as OrderStatus;
export { status, back, wrong };
```

No errors at all, even for `"delivered"`, which is not an order status. `"delivered"` is a string, a string **could** be an `OrderStatus`, so `tsc` lets the assertion through. An annotation, `const wrong: OrderStatus = "delivered"`, would be an error (Session 09).

| From                 | To            | Allowed?                                  |
| -------------------- | ------------- | ----------------------------------------- |
| `string`             | `OrderStatus` | Yes: narrower, could be right             |
| `OrderStatus`        | `string`      | Yes: wider, always right                  |
| `"delivered"`        | `OrderStatus` | Yes, though it is wrong                   |
| `{ id: 1003 }`       | `Order`       | Yes: it has some of the properties        |
| `{ note: "gift" }`   | `Order`       | No: TS2352, nothing in common             |
| `string`             | `number`      | No: TS2352                                |

![Pairs of types meet at the overlap check: string to OrderStatus passes, "delivered" to OrderStatus passes too, string to number is refused with TS2352, and as unknown as walks around the check](images/13-assertions-as-const-and-satisfies/overlap-check.gif)

---

## When as Is the Right Tool

`as` is right when you know something `tsc` cannot know, and you are sure. A typical example: `Object.keys` returns `string[]`, because at run time an object may have more keys than its type says

keys.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };

for (const product of Object.keys(prices)) {
  console.log(`${product}: ${prices[product]}`);
}
```

```text
keys.ts:4:30 - error TS7053: Element implicitly has an 'any' type because expression of type 'string' can't be used to index type '{ keyboard: number; mouse: number; }'.
  No index signature with a parameter of type 'string' was found on type '{ keyboard: number; mouse: number; }'.

4   console.log(`${product}: ${prices[product]}`);
                               ~~~~~~~~~~~~~~~


Found 1 error in keys.ts:4
```

The object was made two lines above, so you know its only keys are `"keyboard"` and `"mouse"`

keys.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };

for (const product of Object.keys(prices) as ("keyboard" | "mouse")[]) {
  console.log(`${product}: ${prices[product]}`);
}
```

```text
keyboard: 149900
mouse: 49900
```

Even here, a way without `as` exists and is often better: `Object.entries` gives the key and the value together

entries.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };

for (const [product, pricePaise] of Object.entries(prices)) {
  console.log(`${product}: ${pricePaise}`);
}
```

```text
keyboard: 149900
mouse: 49900
```

Session 15 shows how to write `"keyboard" | "mouse"` without typing the keys twice.

Rules of thumb

- A value **from outside** (JSON, a request, a file, an environment variable): check it, do not assert it.
- A value **you create**: annotate it, or use `satisfies` (below).
- `as`: only for knowledge `tsc` cannot have, as close as possible to where you know it, and never to make an error disappear.

---

## Assertion Functions: Check, Then Trust

Session 10's type guards return `true` or `false`. An **assertion function** checks a value and throws if it is wrong. After the call, `tsc` treats the value as the asserted type

assert-order.ts

```typescript
interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

function assertOrder(value: unknown): asserts value is Order {
  if (
    typeof value !== "object" ||
    value === null ||
    !("id" in value) ||
    typeof value.id !== "number" ||
    !("customer" in value) ||
    typeof value.customer !== "string" ||
    !("totalPaise" in value) ||
    typeof value.totalPaise !== "number"
  ) {
    throw new Error("Not a valid order");
  }
}

const good: unknown = JSON.parse('{ "id": 1001, "customer": "Riya", "totalPaise": 349700 }');
assertOrder(good);
console.log(`${good.customer} owes Rs. ${good.totalPaise / 100}`);

const bad: unknown = JSON.parse('{ "id": 1002, "name": "Sara", "totalPaise": "49900" }');
assertOrder(bad);
console.log(bad.customer);
```

```text
Riya owes Rs. 3497
file:///D:/ts-practice/13-assertions/assert-order.ts:18
    throw new Error("Not a valid order");
          ^

Error: Not a valid order
    at assertOrder (file:///D:/ts-practice/13-assertions/assert-order.ts:18:11)
    at file:///D:/ts-practice/13-assertions/assert-order.ts:27:1
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

| Part                              | Meaning                                                     |
| --------------------------------- | ----------------------------------------------------------- |
| `asserts value is Order`          | "If this function returns at all, `value` is an `Order`"    |
| The `if` with `throw`             | The real check, at run time                                 |
| `assertOrder(good);` on its own line | From the next line on, `good` is an `Order`              |

Compare with `as Order` at the start of this session: the bad data now stops right away, with a clear message, instead of producing `499004000` and crashing two lines later. Before the call, `good` is still `unknown`

```text
assert-before.ts:14:13 - error TS18046: 'good' is of type 'unknown'.

14 console.log(good.customer);
               ~~~~


Found 1 error in assert-before.ts:14
```

One rule: an assertion function must be a `function` declaration, or a variable with an explicit function type. An arrow function in a plain `const` gives TS2775

assert-arrow.ts

```typescript
interface Order {
  id: number;
  customer: string;
  totalPaise: number;
}

const assertOrder = (value: unknown): asserts value is Order => {
  if (typeof value !== "object" || value === null) {
    throw new Error("Not a valid order");
  }
};

const good: unknown = JSON.parse("{}");
assertOrder(good);
```

```text
assert-arrow.ts:14:1 - error TS2775: Assertions require every name in the call target to be declared with an explicit type annotation.

14 assertOrder(good);
   ~~~~~~~~~~~

  assert-arrow.ts:7:7 - 'assertOrder' needs an explicit type annotation.
    7 const assertOrder = (value: unknown): asserts value is Order => {
            ~~~~~~~~~~~


Found 1 error in assert-arrow.ts:14
```

Node.js has a built-in one you may know: `assert(value)` from `node:assert` is typed as `asserts value`.

![Two JSON values go into assertOrder: the good one passes and comes out labelled Order for the following lines; the bad one is stopped at the check with Error: Not a valid order](images/13-assertions-as-const-and-satisfies/assertion-function.gif)

---

## The "Trust Me" Family

You have now met four ways to tell `tsc` more than it can see

| Tool                                  | Example                        | Checks at run time?  | Session |
| ------------------------------------- | ------------------------------ | -------------------- | ------- |
| Type assertion                        | `value as Order`               | No                   | 13      |
| Non-null assertion                    | `order!`                       | No                   | 11      |
| Type guard                            | `function isOrder(v): v is Order` | Yes, you write it | 10      |
| Assertion function                    | `function assertOrder(v): asserts v is Order` | Yes, you write it | 13 |

`tsc` trusts all four without checking them. The difference: the last two contain a real check, so a wrong value is caught when the code runs. The first two contain nothing, so a wrong value travels on until something crashes.

![The same wrong value meets four tools: as Order and ! let it pass and it crashes later; a type guard and an assertion function check it and stop it right away](images/13-assertions-as-const-and-satisfies/trust-family.gif)

---

## as const

`as const` is not really an assertion: it asks `tsc` to keep the most exact type, and makes everything read-only. Start with a single value

const-primitive.ts

```typescript
let method = "GET";
let fixedMethod = "GET" as const;
fixedMethod = "POST";
export { method };
```

```text
const-primitive.ts:3:1 - error TS2322: Type '"POST"' is not assignable to type '"GET"'.

3 fixedMethod = "POST";
  ~~~~~~~~~~~


Found 1 error in const-primitive.ts:3
```

`method` is widened to `string` (Session 03). `fixedMethod` keeps the literal type `"GET"`. On objects, it does more

config.ts

```typescript
const config = {
  port: 3000,
  env: "production",
  allowedOrigins: ["https://shop.example.com", "https://admin.example.com"]
} as const;

config.port = 8080;
config.allowedOrigins.push("https://evil.example.com");
export { config };
```

```text
config.ts:7:8 - error TS2540: Cannot assign to 'port' because it is a read-only property.

7 config.port = 8080;
         ~~~~

config.ts:8:23 - error TS2339: Property 'push' does not exist on type 'readonly ["https://shop.example.com", "https://admin.example.com"]'.

8 config.allowedOrigins.push("https://evil.example.com");
                        ~~~~


Found 2 errors in the same file, starting at: config.ts:7
```

Point at `config` in VS Code, with and without `as const`

| Property          | Without `as const` | With `as const`                                                     |
| ----------------- | ------------------ | ------------------------------------------------------------------- |
| `port`            | `number`           | `readonly port: 3000`                                               |
| `env`             | `string`           | `readonly env: "production"`                                        |
| `allowedOrigins`  | `string[]`         | `readonly ["https://shop.example.com", "https://admin.example.com"]` |

Every property becomes `readonly`, all the way down, and every value keeps its literal type. Remember: like all types, this exists only for `tsc`. To freeze an object at run time too, use `Object.freeze` (Session 06).

![A config object goes through as const: number becomes 3000, string becomes "production", the array becomes a readonly tuple, and a lock appears on every property](images/13-assertions-as-const-and-satisfies/as-const-freeze.gif)

---

## as const for Tuples

Session 06 promised another way to make tuples. An array literal is widened to an array type, `as const` keeps each position

| Code                                                | Type                                            |
| --------------------------------------------------- | ----------------------------------------------- |
| `["debug", "info", "warn", "error"]`                | `string[]`                                      |
| `["debug", "info", "warn", "error"] as const`       | `readonly ["debug", "info", "warn", "error"]`   |
| `["Keyboard", 149900] as const`                     | `readonly ["Keyboard", 149900]`                 |

This is most useful for functions that return two values

min-max.ts

```typescript
function minMax(values: number[]) {
  return [Math.min(...values), Math.max(...values)];
}

const [lowest, highest] = minMax([49900, 149900, 29900]);
console.log(highest - lowest);
```

```text
min-max.ts:6:13 - error TS18048: 'highest' is possibly 'undefined'.

6 console.log(highest - lowest);
              ~~~~~~~

min-max.ts:6:23 - error TS18048: 'lowest' is possibly 'undefined'.

6 console.log(highest - lowest);
                        ~~~~~~


Found 2 errors in the same file, starting at: min-max.ts:6
```

The return type is `number[]`, and destructuring an array gives `number | undefined` (Session 06). With `as const`, the result is a tuple of exactly two numbers

min-max.ts

```typescript
function minMax(values: number[]): readonly [number, number] {
  return [Math.min(...values), Math.max(...values)] as const;
}

const [lowest, highest] = minMax([49900, 149900, 29900]);
console.log(highest - lowest);
```

```text
120000
```

The return type `readonly [number, number]` is what `tsc` infers from `as const` anyway. Written out, it follows the course rule from Session 12.

![minMax returns [29900, 149900]: as a number[] the destructured values may be undefined; with as const it is a box with exactly two slots, and both values are numbers](images/13-assertions-as-const-and-satisfies/as-const-tuple.gif)

---

## as const Stops Widening

Session 09 showed that an object property is widened to `string`, so `update.status` did not fit `OrderStatus`. `as const` is the promised third fix

widening.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

function setStatus(id: number, status: OrderStatus): void {
  console.log(`Order ${id} is now ${status}`);
}

const update = { id: 1003, status: "shipped" } as const;
setStatus(update.id, update.status);
```

```text
Order 1003 is now shipped
```

`update.status` is `"shipped"`, which fits `OrderStatus`. An annotation with an interface, as in Session 09, is still the clearer choice for data like this. `as const` shines for fixed values that never change: settings, lists of allowed values, lookup tables.

---

## satisfies: Check Without Changing the Type

An annotation has a side effect: the variable gets the annotated type, and loses what `tsc` knew about the value

server-config.ts

```typescript
interface ServerConfig {
  port: number | string;
  host: string;
}

const annotated: ServerConfig = { port: 3000, host: "localhost" };
console.log(annotated.port.toFixed(0));
```

```text
server-config.ts:7:28 - error TS2339: Property 'toFixed' does not exist on type 'string | number'.
  Property 'toFixed' does not exist on type 'string'.

7 console.log(annotated.port.toFixed(0));
                             ~~~~~~~


Found 1 error in server-config.ts:7
```

The value clearly has a number, but the annotation said "number or text". `satisfies` checks the value against the type, and keeps the value's own type

server-config.ts

```typescript
interface ServerConfig {
  port: number | string;
  host: string;
}

const checked = { port: 3000, host: "localhost" } satisfies ServerConfig;
console.log(checked.port.toFixed(0));
```

```text
3000
```

And the check is as strict as an annotation, not weak like `as`

server-config.ts

```typescript
interface ServerConfig {
  port: number | string;
  host: string;
}

const typo = { prot: 3000, host: "localhost" } satisfies ServerConfig;
const missing = { port: 3000 } satisfies ServerConfig;
const wrongType = { port: true, host: "localhost" } satisfies ServerConfig;
```

```text
server-config.ts:6:16 - error TS2353: Object literal may only specify known properties, and 'prot' does not exist in type 'ServerConfig'.

6 const typo = { prot: 3000, host: "localhost" } satisfies ServerConfig;
                 ~~~~

server-config.ts:7:32 - error TS2741: Property 'host' is missing in type '{ port: number; }' but required in type 'ServerConfig'.

7 const missing = { port: 3000 } satisfies ServerConfig;
                                 ~~~~~~~~~

  server-config.ts:3:3 - 'host' is declared here.
    3   host: string;
        ~~~~

server-config.ts:8:21 - error TS2322: Type 'boolean' is not assignable to type 'string | number'.

8 const wrongType = { port: true, host: "localhost" } satisfies ServerConfig;
                      ~~~~

  server-config.ts:2:3 - The expected type comes from property 'port' which is declared here on type 'ServerConfig'
    2   port: number | string;
        ~~~~


Found 3 errors in the same file, starting at: server-config.ts:6
```

A typo, a missing property and a wrong type, all caught. With `as ServerConfig`, the `missing` line would pass without a word.

`as const` and `satisfies` can be combined: `{ port: 3000, host: "localhost" } as const satisfies ServerConfig` is checked against `ServerConfig`, and keeps `readonly port: 3000`.

![The same object goes three ways: an annotation checks it but turns port into number | string; as does not check it; satisfies checks it and keeps port as number](images/13-assertions-as-const-and-satisfies/satisfies-check.gif)

---

## Annotation, as, satisfies or as const?

| Write                          | Checks the value?               | Type of the variable afterwards           | Use for                                  |
| ------------------------------ | ------------------------------- | ----------------------------------------- | ---------------------------------------- |
| `const x: T = value`           | Yes, fully                      | `T`                                       | Most values you create                   |
| `const x = value satisfies T`  | Yes, fully                      | The value's own, more exact type          | Config and lookup objects                |
| `const x = value as const`     | No                              | Exact literals, everything `readonly`     | Fixed values, tuples                     |
| `const x = value as T`         | Only "could they overlap?"      | `T`                                       | Knowledge `tsc` cannot have, rarely      |

For data from outside, none of these is enough: check it with a type guard or an assertion function, or a validation library (Session 26).

---

## Mini Example 1: Loading a Config File

A server reads its settings from `config.json` at start-up. Two versions of the loader, the same data

config.json

```json
{
  "port": "8080",
  "host": "0.0.0.0",
  "logLevel": "warn"
}
```

Someone wrote the port in quotes. The first version uses `as`, as many projects do

load-config.ts

```typescript
import { readFileSync } from "node:fs";

interface AppConfig {
  port: number;
  host: string;
  logLevel: "debug" | "info" | "warn" | "error";
}

const config = JSON.parse(readFileSync("config.json", "utf8")) as AppConfig;

console.log(`Server on ${config.host}:${config.port}, log level ${config.logLevel}`);
console.log(`Admin panel on port ${config.port + 1}`);
```

```bash
npx tsc --noEmit
node load-config.ts
```

```text
Server on 0.0.0.0:8080, log level warn
Admin panel on port 80801
```

No error from `tsc`, no error from Node.js, and a wrong port: `"8080" + 1` is `"80801"`. The server starts, and the admin panel listens somewhere nobody expects. With a correct `config.json` (`"port": 8080`), the same code prints `8081`, so the bug only appears with bad data.

The second version checks before it trusts

load-config.ts

```typescript
import { readFileSync } from "node:fs";

interface AppConfig {
  port: number;
  host: string;
  logLevel: "debug" | "info" | "warn" | "error";
}

function assertAppConfig(value: unknown): asserts value is AppConfig {
  if (typeof value !== "object" || value === null) {
    throw new Error("config.json must contain an object");
  }
  if (!("port" in value) || typeof value.port !== "number") {
    throw new Error("config.json: port must be a number");
  }
  if (!("host" in value) || typeof value.host !== "string") {
    throw new Error("config.json: host must be a text");
  }
  if (!("logLevel" in value) || typeof value.logLevel !== "string" || !["debug", "info", "warn", "error"].includes(value.logLevel)) {
    throw new Error("config.json: logLevel must be debug, info, warn or error");
  }
}

const config: unknown = JSON.parse(readFileSync("config.json", "utf8"));
assertAppConfig(config);

console.log(`Server on ${config.host}:${config.port}, log level ${config.logLevel}`);
console.log(`Admin panel on port ${config.port + 1}`);
```

```text
file:///D:/ts-practice/13-assertions/load-config.ts:14
    throw new Error("config.json: port must be a number");
          ^

Error: config.json: port must be a number
    at assertAppConfig (file:///D:/ts-practice/13-assertions/load-config.ts:14:11)
    at file:///D:/ts-practice/13-assertions/load-config.ts:25:1
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

The server refuses to start, and the message says exactly what to fix. Fix `config.json` to `"port": 8080`, and it prints

```text
Server on 0.0.0.0:8080, log level warn
Admin panel on port 8081
```

With `"logLevel": "verbose"`, it stops with `config.json: logLevel must be debug, info, warn or error`.

| Version             | Bad `config.json`                          | Where you find out                  |
| ------------------- | ------------------------------------------ | ----------------------------------- |
| `as AppConfig`      | Runs, with a wrong port                    | Later, maybe in production          |
| `assertAppConfig`   | Stops at start-up with a clear message     | Immediately                         |

Remove the `assertAppConfig(config);` line, and `tsc` reminds you that `config` is still `unknown`

```text
load-config.ts:26:26 - error TS18046: 'config' is of type 'unknown'.

26 console.log(`Server on ${config.host}:${config.port}, log level ${config.logLevel}`);
                            ~~~~~~

load-config.ts:26:41 - error TS18046: 'config' is of type 'unknown'.

26 console.log(`Server on ${config.host}:${config.port}, log level ${config.logLevel}`);
                                           ~~~~~~

load-config.ts:26:67 - error TS18046: 'config' is of type 'unknown'.

26 console.log(`Server on ${config.host}:${config.port}, log level ${config.logLevel}`);
                                                                     ~~~~~~

load-config.ts:27:36 - error TS18046: 'config' is of type 'unknown'.

27 console.log(`Admin panel on port ${config.port + 1}`);
                                      ~~~~~~


Found 4 errors in the same file, starting at: load-config.ts:26
```

"Fail fast" at start-up is a real backend practice: a server with broken settings should not start at all. Session 26 does the same with Joi, in fewer lines.

![config.json with "port": "8080" goes two ways: through as AppConfig it starts the server with the admin panel on port 80801; through assertAppConfig it is stopped at start-up with port must be a number](images/13-assertions-as-const-and-satisfies/config-loader.gif)

---

## Mini Example 2: A Route Table with satisfies

Many apps keep a table of their routes, with names, for menus, permissions or docs

routes.ts

```typescript
type Method = "GET" | "POST" | "DELETE";

interface RouteConfig {
  method: Method;
  path: string;
  auth: boolean;
}

const routes = {
  listOrders: { method: "GET", path: "/orders", auth: false },
  createOrder: { method: "POST", path: "/orders", auth: true },
  cancelOrder: { method: "DELETE", path: "/orders/:id", auth: true }
} satisfies { [name: string]: RouteConfig };

function describeRoute(route: RouteConfig): string {
  const access = route.auth ? "login required" : "public";
  return `${route.method.padEnd(6)} ${route.path.padEnd(12)} ${access}`;
}

for (const [name, route] of Object.entries(routes)) {
  console.log(`${name.padEnd(12)} ${describeRoute(route)}`);
}

console.log(`Create an order: ${routes.createOrder.method} ${routes.createOrder.path}`);
```

```bash
npx tsc --noEmit
node routes.ts
```

```text
listOrders   GET    /orders      public
createOrder  POST   /orders      login required
cancelOrder  DELETE /orders/:id  login required
Create an order: POST /orders
```

`satisfies { [name: string]: RouteConfig }` says: "any names, but every value must be a `RouteConfig`" (Session 17 writes this type shorter, as `Record<string, RouteConfig>`). Every route is checked. Change the last method to `"PUT"`

```text
routes.ts:12:18 - error TS2322: Type '"PUT"' is not assignable to type 'Method'.

12   cancelOrder: { method: "PUT", path: "/orders/:id", auth: true }
                    ~~~~~~

  routes.ts:4:3 - The expected type comes from property 'method' which is declared here on type 'RouteConfig'
    4   method: Method;
        ~~~~~~

routes.ts:21:51 - error TS2345: Argument of type '{ method: "GET"; path: string; auth: false; } | { method: "POST"; path: string; auth: true; } | { method: "PUT"; path: string; auth: true; }' is not assignable to parameter of type 'RouteConfig'.
  Type '{ method: "PUT"; path: string; auth: true; }' is not assignable to type 'RouteConfig'.
    Types of property 'method' are incompatible.
      Type '"PUT"' is not assignable to type 'Method'.

21   console.log(`${name.padEnd(12)} ${describeRoute(route)}`);
                                                     ~~~~~


Found 2 errors in the same file, starting at: routes.ts:12
```

The first error is in the table, where the typo is. The second shows that `routes` kept its own exact type, including the wrong route.

And the names stay known. Change `"PUT"` back, and add a line that uses a route that does not exist

```typescript
console.log(routes.deleteOrder.path);
```

```text
routes.ts:25:20 - error TS2339: Property 'deleteOrder' does not exist on type '{ listOrders: { method: "GET"; path: string; auth: false; }; createOrder: { method: "POST"; path: string; auth: true; }; cancelOrder: { method: "DELETE"; path: string; auth: true; }; }'.

25 console.log(routes.deleteOrder.path);
                      ~~~~~~~~~~~


Found 1 error in routes.ts:25
```

Now try the table with an annotation instead: `const routes: { [name: string]: RouteConfig } = { ... };`, with `};` at the end

```text
routes.ts:24:33 - error TS18048: 'routes.createOrder' is possibly 'undefined'.

24 console.log(`Create an order: ${routes.createOrder.method} ${routes.createOrder.path}`);
                                   ~~~~~~~~~~~~~~~~~~

routes.ts:24:62 - error TS18048: 'routes.createOrder' is possibly 'undefined'.

24 console.log(`Create an order: ${routes.createOrder.method} ${routes.createOrder.path}`);
                                                                ~~~~~~~~~~~~~~~~~~

routes.ts:25:13 - error TS18048: 'routes.deleteOrder' is possibly 'undefined'.

25 console.log(routes.deleteOrder.path);
               ~~~~~~~~~~~~~~~~~~


Found 3 errors in the same file, starting at: routes.ts:24
```

With the annotation, `routes` is "an object with any names", so `tsc` forgot which routes exist: `createOrder`, which is right there, is now "possibly undefined", and `deleteOrder` is not an error any more, only "possibly undefined" too.

| Route table                         | Values checked? | Known names?  | `routes.createOrder`       | `routes.deleteOrder`       |
| ----------------------------------- | --------------- | ------------- | -------------------------- | -------------------------- |
| `: { [name: string]: RouteConfig }` | Yes             | No            | `RouteConfig \| undefined` | `RouteConfig \| undefined` |
| `satisfies { [name: string]: RouteConfig }` | Yes     | Yes           | The exact route            | TS2339: does not exist     |

This is the situation `satisfies` was made for: check every value, keep every name.

![The route table goes through satisfies: each route is checked against RouteConfig, and the three names stay on the result; with an annotation, the names are lost and every lookup may be undefined](images/13-assertions-as-const-and-satisfies/route-table.gif)

---

## Beginner Mistakes

### Mistake 1: Using as on data from outside

`JSON.parse(body) as Order` with wrong data printed `499004000` and crashed on `customer`. `tsc` cannot check data it never sees. Check it with a type guard or an assertion function.

---

### Mistake 2: Using as instead of an annotation

`{ id: 1003 } as Order` passes with two missing properties, while `const typed: Order = { id: 1003 }` gives `error TS2739`. When you create a value, annotate it.

---

### Mistake 3: Expecting as to convert a value

`"3" as unknown as number` is still `"3"`, and `+ 1` gave `"31"`. Use `Number("3")` to convert.

---

### Mistake 4: as unknown as or as any to make an error go away

TS2352 tells you an assertion is almost certainly wrong. `as unknown as` silences it, and the bug stays. Fix the types instead.

---

### Mistake 5: Trusting as to catch typos

`"delivered" as OrderStatus` gives no error, because a string could be an `OrderStatus`. Write `const status: OrderStatus = "delivered"` and the typo is caught.

---

### Mistake 6: Writing an assertion function as an arrow function

`const assertOrder = (value: unknown): asserts value is Order => { ... }` gives `error TS2775` when called. Use a `function` declaration.

---

### Mistake 7: Thinking as const freezes the object

`as const` makes properties `readonly` for `tsc` only. At run time, the object can still be changed by code that `tsc` does not check. Use `Object.freeze` when it must be frozen for real.

---

### Mistake 8: Using as const where an annotation is clearer

`{ id: 1003, status: "shipped" } as const` works, but `const update: StatusUpdate = { ... }` says what the object is and checks it. Use `as const` for fixed values, not for normal data.

---

### Mistake 9: Annotating a lookup table and losing its names

`const routes: { [name: string]: RouteConfig } = { ... }` made `routes.createOrder` possibly undefined. Use `satisfies`.

---

### Mistake 10: Mixing up satisfies and as

`satisfies` checks fully and keeps the exact type. `as` hardly checks and replaces the type. If you want `tsc` to catch mistakes in a value, `satisfies` (or an annotation) is the one.

---

### Mistake 11: Returning a pair as an array

`return [min, max]` has the type `number[]`, and destructuring gave `possibly 'undefined'`. Return `[min, max] as const`, or write the return type `readonly [number, number]`.

---

## Practice Exercises

### Exercise 1

Parse `'{ "name": "Keyboard", "pricePaise": 149900 }'` with `as Product`, then parse a version where `pricePaise` is `"149900"`. Print `pricePaise * 2` for both. Which result is wrong, and why did `tsc` not warn you?

### Exercise 2

Turn Exercise 1 into an assertion function `assertProduct(value: unknown): asserts value is Product`, and run both texts again. Make the error message name the property that is wrong.

### Exercise 3

For each line, predict whether `tsc` reports TS2352, then check: `42 as string`, `"42" as unknown as number`, `{ name: "Riya" } as Customer` (with `interface Customer { name: string; city: string }`), `{ age: 25 } as Customer`.

### Exercise 4

Write `const HTTP_STATUS = { ok: 200, created: 201, notFound: 404 }` once without and once with `as const`. Point at `HTTP_STATUS.ok` in both versions. Then pass it to `function send(status: 200 | 201 | 404): void`. Which version works?

### Exercise 5

Write `function splitName(fullName: string)` that returns the first and last name as a tuple with `as const`. Destructure the result and call `.toUpperCase()` on both parts.

### Exercise 6

Write `interface Theme { primary: string; fontSize: number | string }`. Create a theme with `satisfies` where `fontSize` is `16`, and one with an annotation. In which one can you write `theme.fontSize + 2` without an error?

### Exercise 7

In Mini Example 2, add a fourth route `updateOrder` with method `"PATCH"`. Read the error, then add `"PATCH"` to `Method`. Then print `routes.updateOrder.method`: what type does VS Code show?

### Exercise 8

In Mini Example 1, give `AppConfig` an optional `maxUploadMb?: number` and extend `assertAppConfig`: if the property is present, it must be a number. Test with the property missing, with `20`, and with `"20"`.

---

## Interview Questions

### What is a type assertion?

`value as Type` tells TypeScript to treat a value as a given type. It changes only the type that TypeScript uses, not the value, and it is removed when the code runs.

### What is the difference between a type annotation and a type assertion?

An annotation (`const x: Order = value`) checks that the value really is an `Order`. An assertion (`value as Order`) only checks that the types could be related, and then believes you. Missing properties pass an assertion but fail an annotation.

### When does TypeScript refuse an assertion?

When the two types do not overlap at all, like `string` and `number`. It reports TS2352 and suggests converting to `unknown` first, which removes even that check.

### Why is as unknown as dangerous?

Every type overlaps with `unknown`, so `as unknown as T` can turn any value into any type without a single check. It hides real type errors.

### Does as convert values?

No. `"3" as unknown as number` is still the text `"3"` at run time. Conversions need real code, like `Number("3")`.

### What is an assertion function?

A function with the return type `asserts value is T`. It throws if the value is wrong; if it returns, TypeScript treats the value as `T` from the next line on. Unlike `as`, it contains a real run-time check.

### What does as const do?

It keeps the most exact type of a value: literal types instead of `string` or `number`, readonly tuples instead of arrays, and `readonly` on every property, deeply. It does not freeze the object at run time.

### What is satisfies?

An operator that checks a value against a type, with the same strictness as an annotation, but keeps the value's own inferred type. Typos, missing properties and wrong types are caught, and exact types like literal values and known keys are kept.

### When would you use satisfies instead of an annotation?

For config objects and lookup tables, where you want every value checked but still want TypeScript to know the exact keys and value types, for example to get autocomplete and errors for names that do not exist.

### Can you combine as const and satisfies?

Yes: `value as const satisfies T` checks the value against `T` and keeps the readonly literal types from `as const`.

### What is the non-null assertion operator, and how does it relate to as?

`value!` removes `null` and `undefined` from the type. Like `as`, it is not checked at run time, so if the value really is missing, the code crashes later.

### How should you handle JSON from an API in TypeScript?

Treat it as `unknown` and check it before use: with type guards, assertion functions, or a validation library like Joi. An assertion like `as Order` only hides the problem.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [12. Typing Functions](12-typing-functions.md) | [Course outline](README.md#course-outline) | [14. Generics](14-generics.md) |
