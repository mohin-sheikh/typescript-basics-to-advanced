# Runtime Validation with Joi

> **Session 26 of 33** | Part 6: Real Programs | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Why Types Need Help at Run Time](#why-types-need-help-at-run-time)
- [Your First Joi Schema](#your-first-joi-schema)
- [Giving the Result a Type](#giving-the-result-a-type)
- [Keeping the Schema and the Type in Line](#keeping-the-schema-and-the-type-in-line)
- [Validation Options and Their Type](#validation-options-and-their-type)
- [One validate() for the Whole Program](#one-validate-for-the-whole-program)
- [What Joi Accepts and Refuses](#what-joi-accepts-and-refuses)
- [Checking a Settings File at Startup](#checking-a-settings-file-at-startup)
- [Checking Environment Variables](#checking-environment-variables)
- [Mini Example 1: An Order Request with Items](#mini-example-1-an-order-request-with-items)
- [Mini Example 2: Checking Data from Another API](#mini-example-2-checking-data-from-another-api)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `26-validation` inside `ts-practice`, with the one-line `tsconfig.json`

26-validation/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

Joi was installed in Session 25. If you skipped that, run `npm install joi` (this session was tested with joi 18.3.0). Joi brings its own types, so there is no `@types` package to install.

---

## Why Types Need Help at Run Time

Types disappear before the code runs (Session 01). Inside your program that is fine: `tsc` has checked every value you create. But a backend also receives data it did not create

| Data from outside           | Example                                    | What TypeScript knows        |
| --------------------------- | ------------------------------------------ | ---------------------------- |
| A request body              | `{ "customer": "Asha", "quantity": "2" }`  | Nothing: it arrives as text  |
| A query string              | `?page=2`                                  | Always text                  |
| Environment variables       | `PORT=3000` in `.env`                      | `string \| undefined` (Session 25) |
| A settings file             | `config.json`                              | `JSON.parse` returns `any`   |
| Another API                 | `await res.json()`                         | `any` (Session 23)           |

A type annotation or `as` on such data is only a promise (Session 13). You have kept that promise by hand three times already: `isOrderBody` in Session 10, `assertAppConfig` in Session 13 and `isProduct` in Session 23. Each was 10 to 15 lines for two or three properties, with every message written by you. A real request body has ten properties, nested objects, lists, and rules like "between 1 and 10".

A **validation library** does that job. You describe the rules once, as a **schema**, and the library checks any value against it when the program runs, and lists every problem it finds. This course uses **Joi**, the same library as the Node.js course. (Another popular one is Zod; the idea is the same.)

The rule for where to use it: check data **at the edges** of your program, once, where it comes in. After that, the rest of your code can trust its types.

![Data from outside, a request body, environment variables, a settings file and another API, all pass through one Joi check at the edge of the program; what gets through has a type the rest of the code can trust, and what does not is refused with a list of problems](images/26-runtime-validation-with-joi/edge-check.gif)

---

## Your First Joi Schema

first-schema.ts

```typescript
import Joi from "joi";

const orderSchema = Joi.object({
  customer: Joi.string().trim().min(2).required(),
  quantity: Joi.number().integer().min(1).required(),
  payment: Joi.string().valid("card", "upi", "cod").required()
});

const good = orderSchema.validate({ customer: "  Asha ", quantity: "2", payment: "upi" });
console.log(good);

const bad = { customer: "A", quantity: 0, payment: "paypal" };
console.log(orderSchema.validate(bad).error?.message);
console.log(orderSchema.validate(bad, { abortEarly: false }).error?.details.map((detail) => detail.message));
```

```bash
node first-schema.ts
```

```text
{ value: { customer: 'Asha', quantity: 2, payment: 'upi' } }
"customer" length must be at least 2 characters long
[
  '"customer" length must be at least 2 characters long',
  '"quantity" must be greater than or equal to 1',
  '"payment" must be one of [card, upi, cod]'
]
```

`schema.validate(data)` never throws. It returns an object with `value`, the data, and `error`, which is only there when something is wrong (that is why the code uses `?.`, Session 11). Look at `value` in the first answer: Joi also **cleaned** the data. The spaces around `"  Asha "` are gone, and the text `"2"` became the number `2`.

| Rule                            | Meaning                                                      |
| ------------------------------- | ------------------------------------------------------------ |
| `Joi.object({ ... })`           | An object with these keys; any other key is an error         |
| `Joi.string()`                  | A text. A number or an object is refused                     |
| `.trim()`                       | Remove spaces at both ends before checking                   |
| `.min(2)`, `.max(200)`          | The length of a text, or the size of a number                |
| `Joi.number()`                  | A number. The text `"2"` becomes `2`; `"abc"` is refused     |
| `.integer()`                    | A whole number: `2.5` is refused                             |
| `.valid("card", "upi", "cod")`  | Only these values                                            |
| `.required()`                   | The key must be there. Without it, the key is optional       |

By default Joi stops at the first problem. With the option `abortEarly: false`, `error.details` lists every problem, each with a `message`.

So far this is plain JavaScript, the same as in the Node.js course. Now the TypeScript part. Hover over `good.value` in VS Code: it is `any`. Add a typo at the end

```typescript
console.log(good.value.quantty);
```

`tsc` passes, and Node.js prints `undefined`. The check ran, but its result has no type, so the rest of the program is back to guessing.

![An order goes through the rules one by one: the spaces around Asha are trimmed, the text 2 becomes the number 2, upi is in the list; a bad order collects three messages instead](images/26-runtime-validation-with-joi/first-schema.gif)

---

## Giving the Result a Type

Write the shape as an interface (the course rule since Session 08), and give it to `Joi.object` as a type argument (Session 14)

typed-order.ts

```typescript
import Joi from "joi";

interface NewOrder {
  customer: string;
  quantity: number;
  payment: "card" | "upi" | "cod";
  note?: string;
}

const orderSchema = Joi.object<NewOrder>({
  customer: Joi.string().trim().min(2).required(),
  quantity: Joi.number().integer().min(1).required(),
  payment: Joi.string().valid("card", "upi", "cod").required(),
  note: Joi.string().trim().max(200)
});

const result = orderSchema.validate({ customer: "Asha", quantity: "2", payment: "upi" });

if (result.error) {
  console.log(result.error.message);
} else {
  console.log(`${result.value.quantty} for ${result.value.customer}`);
}
```

```text
typed-order.ts:22:31 - error TS2551: Property 'quantty' does not exist on type 'NewOrder'. Did you mean 'quantity'?

22   console.log(`${result.value.quantty} for ${result.value.customer}`);
                                 ~~~~~~~

  typed-order.ts:5:3 - 'quantity' is declared here.
    5   quantity: number;
        ~~~~~~~~


Found 1 error in typed-order.ts:22
```

The typo is caught. Fix it to `quantity`, and `node typed-order.ts` prints `2 for Asha`. Where does the type come from? Joi's `index.d.ts` declares the result of `validate()` like this

```typescript
  type ValidationResult<TSchema = any> =
    | {
        error: undefined;
        warning?: ValidationError;
        value: TSchema;
      }
    | {
        error: ValidationError;
        warning?: ValidationError;
        value: any;
      };
```

A discriminated union (Session 10), with `error` as the tag: when `error` is `undefined`, `value` is your `NewOrder`; when there is an error, `value` is `any`. So `value` only has its type **after** you check `error`. Before the check, `result.value` is `NewOrder | any`, which is just `any`, and `result.value.quantty` passes `tsc` again ([Mistake 1](#mistake-1-using-value-before-checking-error)).

Destructuring works the same way

```typescript
const { error, value } = orderSchema.validate({ customer: "Asha", quantity: "2", payment: "upi" });

if (error) {
  console.log(error.message);
} else {
  console.log(`${value.quantity} for ${value.customer}`);
}
```

![validate returns one of two shapes: with error undefined, value is a NewOrder; with an error, value is any; only checking error tells TypeScript which shape it has](images/26-runtime-validation-with-joi/typed-result.gif)

---

## Keeping the Schema and the Type in Line

The interface and the schema now describe the same thing twice, and the type of `value` is only right if they agree. How much does TypeScript check? Misspell a key in the schema

```typescript
  quantty: Joi.number().integer().min(1).required(),
```

```text
typed-order.ts:12:3 - error TS2561: Object literal may only specify known properties, but 'quantty' does not exist in type 'PartialSchemaMap<NewOrder>'. Did you mean to write 'quantity'?

12   quantty: Joi.number().integer().min(1).required(),
     ~~~~~~~


Found 1 error in typed-order.ts:12
```

Good. Now delete the `quantity` line completely. `tsc` says nothing, because `PartialSchemaMap` means every key may be left out. When the program runs, Joi refuses the order with `"quantity" is not allowed`, since the schema does not know that key. That is the gentle case; with the option `stripUnknown` (next section), the field would silently disappear while the type still says `quantity: number`.

Joi has a stricter mode: a second type argument `true`

```typescript
const orderSchema = Joi.object<NewOrder, true>({
```

```text
typed-order.ts:10:48 - error TS2741: Property 'quantity' is missing in type '{ customer: Joi.StringSchema<string>; payment: Joi.StringSchema<string>; note: Joi.StringSchema<string>; }' but required in type 'StrictSchemaMap<NewOrder>'.

 10 const orderSchema = Joi.object<NewOrder, true>({
                                                   ~
 11   customer: Joi.string().trim().min(2).required(),
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
... 
 13   note: Joi.string().trim().max(200)
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 14 });
    ~

  typed-order.ts:5:3 - 'quantity' is declared here.
    5   quantity: number;
        ~~~~~~~~


Found 1 error in typed-order.ts:10
```

In strict mode every property of the interface needs a rule, the optional ones too (leaving out `note` gives the same TS2741). And the kind of rule must fit the property's type. Put the `quantity` line back, as a text rule

```typescript
  quantity: Joi.string().required(),
```

```text
typed-order.ts:12:3 - error TS2740: Type 'StringSchema<string>' is missing the following properties from type 'NumberSchema<number>': greater, integer, less, multiple, and 6 more.

12   quantity: Joi.string().required(),
     ~~~~~~~~


Found 1 error in typed-order.ts:12
```

"`Joi.string()` is not a `NumberSchema`": a `number` property needs `Joi.number()`. Strict mode catches a lot, but not everything. Remove `.required()` from `customer`, and validate an order without a customer

```typescript
  customer: Joi.string().trim().min(2),
```

```typescript
const result = orderSchema.validate({ quantity: "2", payment: "upi" });
```

```text
2 for undefined
```

No error from `tsc`, none from Joi. The type says `customer: string`, but the value is `undefined`. To TypeScript, a rule with `.required()` and one without look the same. Put `.required()` back, and Joi answers `"customer" is required`.

The same goes for the values in `.valid()`: `.valid("card", "upi", "paypal")` passes `tsc`, although the type says `"cod"`. The fix is to write the list once, with `as const`, and build both the type and the rule from it (Sessions 13 and 15). Here is the finished file; make it a module of its own

order-schema.ts

```typescript
import Joi from "joi";

export const PAYMENTS = ["card", "upi", "cod"] as const;
export type Payment = (typeof PAYMENTS)[number];

export interface NewOrder {
  customer: string;
  quantity: number;
  payment: Payment;
  note?: string;
}

export const orderSchema = Joi.object<NewOrder, true>({
  customer: Joi.string().trim().min(2).required(),
  quantity: Joi.number().integer().min(1).required(),
  payment: Joi.string().valid(...PAYMENTS).required(),
  note: Joi.string().trim().max(200)
});
```

What each version catches

| Mistake in the schema              | `Joi.object({ })` | `Joi.object<NewOrder>` | `Joi.object<NewOrder, true>` |
| ---------------------------------- | ----------------- | ---------------------- | ---------------------------- |
| A misspelled key                   | Nothing           | TS2561                 | TS2561                       |
| A property without a rule          | Nothing           | Nothing                | TS2741                       |
| The wrong kind of rule             | Nothing           | Nothing                | TS2740                       |
| `.required()` forgotten            | Nothing           | Nothing                | Nothing: keep `.required()` on every property without `?` |
| Other values in `.valid()`         | Nothing           | Nothing                | Nothing: build both from one `as const` list |

The course rule from now on: an interface first, then `Joi.object<Interface, true>`, `.required()` on every property that has no `?`, and one `as const` list for each set of allowed values.

![Five ways a schema can drift from its interface, checked by three versions of Joi.object: the plain one catches nothing, the typed one catches a misspelled key, the strict one also catches a missing rule and the wrong kind of rule, and none of them sees a forgotten required or other values in valid](images/26-runtime-validation-with-joi/schema-drift.gif)

---

## Validation Options and Their Type

`validate()` takes options as its second argument

| Option                               | Default | What it does                                              |
| ------------------------------------ | ------- | --------------------------------------------------------- |
| `abortEarly`                         | `true`  | Stop at the first problem. With `false`, every problem is reported |
| `stripUnknown`                       | `false` | Remove keys the schema does not list. Without it, an extra key is an error: `"role" is not allowed` |
| `convert`                            | `true`  | Turn `"2"` into `2`, trim, and so on. With `false`, `"2"` is refused: `must be a number` |
| `errors: { wrap: { label: false } }` | quotes  | `quantity is required` instead of `"quantity" is required` |

A program uses the same options everywhere, so you put them in a variable

options.ts

```typescript
import { orderSchema } from "./order-schema.ts";

const options = {
  abortEarly: false,
  stripUnknown: true,
  errors: { wrap: { label: false } }
};

const result = orderSchema.validate({ customer: "A" }, options);
console.log(result.error?.message);
```

```text
options.ts:9:56 - error TS2345: Argument of type '{ abortEarly: boolean; stripUnknown: boolean; errors: { wrap: { label: boolean; }; }; }' is not assignable to parameter of type 'ValidationOptions'.
  The types of 'errors.wrap.label' are incompatible between these types.
    Type 'boolean' is not assignable to type 'string | false'.

9 const result = orderSchema.validate({ customer: "A" }, options);
                                                         ~~~~~~~


Found 1 error in options.ts:9
```

Session 13's widening: in a variable, `label: false` becomes `label: boolean`, and Joi only accepts `false` or a text. Give the variable Joi's own type for options (Session 25: a package's types come with it)

```typescript
import Joi from "joi";
import { orderSchema } from "./order-schema.ts";

const options: Joi.ValidationOptions = {
```

```text
customer length must be at least 2 characters long. quantity is required. payment is required
```

![Four options and what each one does to the result; an options object in a plain variable turns label false into boolean and validate refuses it with TS2345, while the same object typed as Joi.ValidationOptions keeps false](images/26-runtime-validation-with-joi/options.gif)

---

## One validate() for the Whole Program

Every place that checks data needs the same steps: validate with the options, check `error`, collect the messages, use `value`. Write them once, as a generic function (Session 14) that throws a typed error class (Session 21)

validation.ts

```typescript
import Joi from "joi";

export class InvalidDataError extends Error {
  readonly problems: string[];

  constructor(what: string, problems: string[]) {
    super(`${what} is invalid: ${problems.join("; ")}`);
    this.name = "InvalidDataError";
    this.problems = problems;
  }
}

const options: Joi.ValidationOptions = {
  abortEarly: false,
  stripUnknown: true,
  errors: { wrap: { label: false } }
};

export function validate<T>(schema: Joi.ObjectSchema<T>, data: unknown, what: string): T {
  const result = schema.required().validate(data, options);
  if (result.error) {
    throw new InvalidDataError(what, result.error.details.map((detail) => detail.message));
  }
  return result.value;
}
```

| Part                                   | Why                                                          |
| -------------------------------------- | ------------------------------------------------------------ |
| `schema: Joi.ObjectSchema<T>`          | `Joi.object<NewOrder, true>(...)` has the type `ObjectSchema<NewOrder>`, so `T` is inferred as `NewOrder` |
| `data: unknown`                        | Anything can come in; nothing about it is assumed            |
| `schema.required()`                    | Without it, `undefined` passes. See below                    |
| `InvalidDataError` with `problems`     | The caller can show the whole list, or turn it into a 400 answer (Session 31) |
| `return result.value`                  | The cleaned value, typed `T`, after `error` was checked      |

Why `schema.required()`? Joi only checks what is there, so "nothing at all" passes an object schema

plain-undefined.ts

```typescript
import { orderSchema } from "./order-schema.ts";

console.log(orderSchema.validate(undefined));
```

```text
{ value: undefined }
```

No error, and a `value` that TypeScript would call a `NewOrder`. A request without a body is exactly this case. `.required()` on the whole schema makes Joi answer `value is required`.

Use it

check-orders.ts

```typescript
import { orderSchema } from "./order-schema.ts";
import { InvalidDataError, validate } from "./validation.ts";

const inputs: unknown[] = [
  { customer: " Asha ", quantity: "2", payment: "upi", role: "admin" },
  { customer: "A", quantity: 0, payment: "paypal" },
  "just text",
  undefined
];

for (const input of inputs) {
  try {
    const order = validate(orderSchema, input, "order");
    console.log(order);
  } catch (err) {
    if (err instanceof InvalidDataError) {
      console.log(err.message);
    } else {
      throw err;
    }
  }
}
```

```text
{ customer: 'Asha', quantity: 2, payment: 'upi' }
order is invalid: customer length must be at least 2 characters long; quantity must be greater than or equal to 1; payment must be one of [card, upi, cod]
order is invalid: value must be of type object
order is invalid: value is required
```

`order` is a `NewOrder`, inferred from the schema. The `role` the client sent is gone (`stripUnknown`), and the code after `validate` only ever sees clean, typed data.

![unknown data goes into validate with a schema; out comes either a value of type T, cleaned and without unknown keys, or an InvalidDataError with the list of problems](images/26-runtime-validation-with-joi/validate-function.gif)

---

## What Joi Accepts and Refuses

The answers of `validate(orderSchema, ...)` for a valid order with one property changed (all tested)

| Change                       | Answer                                              |
| ---------------------------- | --------------------------------------------------- |
| `quantity: "2"`              | Accepted, becomes `2`                               |
| `quantity: "abc"`            | `quantity must be a number`                         |
| `quantity: 2.5`              | `quantity must be an integer`                       |
| `quantity: true`             | `quantity must be a number`                         |
| `customer: 42`               | `customer must be a string`                         |
| `customer: "   "`            | `customer is not allowed to be empty` (trimmed first) |
| `customer: { $gt: "" }`      | `customer must be a string` (a NoSQL injection attempt, Node.js course) |
| `payment: "UPI"`             | `payment must be one of [card, upi, cod]`           |
| `note: ""`                   | `note is not allowed to be empty`; add `.allow("")` to accept it |
| `role: "admin"` (unknown key) | Accepted, `role` removed by `stripUnknown`         |

Conversions are why you always use the **returned** value, never the input: in the input, `quantity` is still the text `"2"`. If you want no conversions at all, pass `convert: false`; then `quantity: "2"` gives `quantity must be a number`.

![Seven changed orders go through the schema: the text 2 is accepted and becomes a number, an unknown role is accepted and removed, and abc, 2.5, an empty customer, an object instead of a text and UPI in capitals are refused with their messages](images/26-runtime-validation-with-joi/accept-refuse.gif)

---

## Checking a Settings File at Startup

Session 13 checked `config.json` with a hand-written assertion function of 15 lines, and promised the same with Joi in fewer lines. Here it is, with the same `config.json`

config.json

```json
{ "port": "8080", "host": "0.0.0.0", "logLevel": "warn" }
```

load-config.ts

```typescript
import { readFileSync } from "node:fs";
import Joi from "joi";
import { validate } from "./validation.ts";

const LOG_LEVELS = ["debug", "info", "warn", "error"] as const;

interface AppConfig {
  port: number;
  host: string;
  logLevel: (typeof LOG_LEVELS)[number];
}

const configSchema = Joi.object<AppConfig, true>({
  port: Joi.number().port().required(),
  host: Joi.string().required(),
  logLevel: Joi.string().valid(...LOG_LEVELS).required()
});

const config = validate(configSchema, JSON.parse(readFileSync("config.json", "utf8")), "config.json");

console.log(`Server on ${config.host}:${config.port}, log level ${config.logLevel}`);
console.log(`Admin panel on port ${config.port + 1}`);
```

```bash
node load-config.ts
```

```text
Server on 0.0.0.0:8080, log level warn
Admin panel on port 8081
```

Session 13's assertion function refused `"port": "8080"` (and its `as AppConfig` version put the admin panel on port `80801`). Joi converts the text to the number `8080`, so the admin panel is on `8081`. `.port()` also checks the range. Break the file in two places

config.json

```json
{ "port": 99999, "host": "0.0.0.0", "logLevel": "verbose" }
```

```text
file:///D:/ts-practice/26-validation/validation.ts:22
    throw new InvalidDataError(what, result.error.details.map((detail) => detail.message));
          ^

InvalidDataError: config.json is invalid: port must be a valid port; logLevel must be one of [debug, info, warn, error]
    at validate (file:///D:/ts-practice/26-validation/validation.ts:22:11)
    at file:///D:/ts-practice/26-validation/load-config.ts:19:16
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5) {
  problems: [
    'port must be a valid port',
    'logLevel must be one of [debug, info, warn, error]'
  ]
}

Node.js v24.19.0
```

Both problems at once, and the program does not start. Session 23's `loadSettings` returned `Promise<unknown>` for the same reason: pass its result to `validate`, and it becomes typed settings.

![Session 13's assertAppConfig needed four if blocks; the Joi schema needs three rules, also converts the text 8080 to a number, and reports both problems of a broken config.json at once](images/26-runtime-validation-with-joi/config-check.gif)

---

## Checking Environment Variables

Session 25 gave `process.env` types, and showed their limit: `NODE_ENV=staging` arrived anyway. Environment variables are the most common settings of a backend, and all of them are text. Check them when the program starts

env-settings.ts

```typescript
import Joi from "joi";
import { validate } from "./validation.ts";

const NODE_ENVS = ["development", "production", "test"] as const;

interface EnvSettings {
  NODE_ENV: (typeof NODE_ENVS)[number];
  PORT: number;
  JWT_SECRET: string;
  JWT_EXPIRES_IN: string;
}

const envSchema = Joi.object<EnvSettings, true>({
  NODE_ENV: Joi.string().valid(...NODE_ENVS).default("development"),
  PORT: Joi.number().port().default(3000),
  JWT_SECRET: Joi.string().min(16).required(),
  JWT_EXPIRES_IN: Joi.string()
    .pattern(/^[0-9]+[smhdwy]$/)
    .default("7d")
    .messages({ "string.pattern.base": "{#label} needs a number and a unit, like 15m, 1h or 7d" })
});

export const settings = validate(envSchema, process.env, "environment");
```

start.ts

```typescript
import { settings } from "./env-settings.ts";

console.log(settings);
console.log(`${settings.NODE_ENV} server on port ${settings.PORT}`);
```

| Part                                  | Why                                                          |
| ------------------------------------- | ------------------------------------------------------------ |
| `.default("development")`             | Used when the variable is missing. A rule with a default never leaves the property empty, so the interface can say `NODE_ENV` without `?` |
| `Joi.number().port()`                 | `"3000"` from the environment becomes the number `3000`      |
| `JWT_SECRET: ... .required()`         | No default for a secret: the program must not start without one |
| `.pattern(...)` with `.messages(...)` | The check from Session 25's `isDuration`, with your own message. `{#label}` is replaced by the key's name |
| `validate(envSchema, process.env, ...)` | `stripUnknown` removes the other variables, like `PATH`, from `settings` |

A broken `.env`

.env

```text
NODE_ENV=staging
PORT=abc
JWT_EXPIRES_IN=3600
```

```bash
node --env-file=.env start.ts
```

```text
file:///D:/ts-practice/26-validation/validation.ts:22
    throw new InvalidDataError(what, result.error.details.map((detail) => detail.message));
          ^

InvalidDataError: environment is invalid: NODE_ENV must be one of [development, production, test]; PORT must be a number; JWT_SECRET is required; JWT_EXPIRES_IN needs a number and a unit, like 15m, 1h or 7d
    at validate (file:///D:/ts-practice/26-validation/validation.ts:22:11)
    at file:///D:/ts-practice/26-validation/env-settings.ts:23:25
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5) {
  problems: [
    'NODE_ENV must be one of [development, production, test]',
    'PORT must be a number',
    'JWT_SECRET is required',
    'JWT_EXPIRES_IN needs a number and a unit, like 15m, 1h or 7d'
  ]
}

Node.js v24.19.0
```

Four problems, found before the server does anything. A good `.env`

.env

```text
PORT=8080
JWT_SECRET=change-me-in-production
```

```text
{
  JWT_SECRET: 'change-me-in-production',
  PORT: 8080,
  NODE_ENV: 'development',
  JWT_EXPIRES_IN: '7d'
}
development server on port 8080
```

`PORT` is a number now (no quotes), and the two missing variables got their defaults. Any module can `import { settings }` and trust every property: this is the pattern Sessions 30 to 33 use for configuration.

![The text values of process.env go through the environment schema: staging, abc and 3600 are refused and a missing JWT_SECRET is reported, all four at startup; with a good .env, PORT becomes the number 8080 and the missing values get their defaults](images/26-runtime-validation-with-joi/env-check.gif)

---

## Mini Example 1: An Order Request with Items

A real order request has a list of items and a delivery address. Schemas can be nested like the interfaces

shop/order-request.ts

```typescript
import Joi from "joi";

const PAYMENTS = ["card", "upi", "cod"] as const;

export interface OrderItem {
  productId: number;
  quantity: number;
}

export interface Address {
  city: string;
  pin: string;
}

export interface OrderRequest {
  items: OrderItem[];
  address: Address;
  payment: (typeof PAYMENTS)[number];
  note?: string;
}

const itemSchema = Joi.object<OrderItem, true>({
  productId: Joi.number().integer().positive().required(),
  quantity: Joi.number().integer().min(1).max(10).required()
});

const addressSchema = Joi.object<Address, true>({
  city: Joi.string().trim().min(2).required(),
  pin: Joi.string()
    .pattern(/^[0-9]{6}$/)
    .required()
    .messages({ "string.pattern.base": "{#label} must be 6 digits" })
});

export const orderRequestSchema = Joi.object<OrderRequest, true>({
  items: Joi.array()
    .items(itemSchema)
    .min(1)
    .required()
    .messages({ "array.min": "an order needs at least one item" }),
  address: addressSchema.required(),
  payment: Joi.string().valid(...PAYMENTS).required(),
  note: Joi.string().trim().max(200)
});
```

| Part                                 | Meaning                                                      |
| ------------------------------------ | ------------------------------------------------------------ |
| `Joi.array().items(itemSchema)`      | A list, and every element is checked with `itemSchema`       |
| `.min(1)` on the array               | At least one element                                         |
| `addressSchema.required()`           | A schema can be used inside another one, like a type inside a type |
| `.positive()`                        | Greater than 0                                               |
| `.pattern(/^[0-9]{6}$/)`             | Exactly six digits (a regular expression, JavaScript course Session 08): an Indian PIN code |

The order code. Prices come from the server, never from the client

shop/place-order.ts

```typescript
import { InvalidDataError, validate } from "../validation.ts";
import { orderRequestSchema } from "./order-request.ts";
import type { OrderRequest } from "./order-request.ts";

const PRICES = new Map<number, number>([
  [1, 149900],
  [2, 49900]
]);

function totalPaise(order: OrderRequest): number {
  let total = 0;
  for (const item of order.items) {
    const price = PRICES.get(item.productId);
    if (price === undefined) {
      throw new Error(`product ${item.productId} does not exist`);
    }
    total += price * item.quantity;
  }
  return total;
}

const requests: unknown[] = [
  {
    items: [{ productId: 1, quantity: 1 }, { productId: 2, quantity: "2" }],
    address: { city: " Pune ", pin: "411001" },
    payment: "upi",
    totalPaise: 100
  },
  {
    items: [{ productId: 1, quantity: 0 }, { productId: "two", quantity: 1 }],
    address: { city: "Pune", pin: "41100" },
    payment: "upi"
  },
  { items: [], address: { city: "Pune", pin: "411001" }, payment: "cod" },
  { items: [{ productId: 99, quantity: 1 }], address: { city: "Pune", pin: "411001" }, payment: "card" }
];

for (const request of requests) {
  try {
    const order = validate(orderRequestSchema, request, "order");
    const rupees = (totalPaise(order) / 100).toFixed(2);
    console.log(`accepted: ${order.items.length} item(s) to ${order.address.city}, Rs ${rupees} by ${order.payment}`);
  } catch (err) {
    if (err instanceof InvalidDataError) {
      console.log(`refused: ${err.problems.join("; ")}`);
    } else if (err instanceof Error) {
      console.log(`refused: ${err.message}`);
    } else {
      throw err;
    }
  }
}
```

```bash
node shop/place-order.ts
```

```text
accepted: 2 item(s) to Pune, Rs 2497.00 by upi
refused: items[0].quantity must be greater than or equal to 1; items[1].productId must be a number; address.pin must be 6 digits
refused: an order needs at least one item
refused: product 99 does not exist
```

Read the four answers

1. The first client sent `totalPaise: 100`, hoping to pay Rs 1. `stripUnknown` removed it, and the server computed Rs 2497.00 from its own prices. `" Pune "` was trimmed, and `"2"` became `2`.
2. Every problem, with its place: `items[0].quantity` is the first item's quantity. A client can fix all of them in one go.
3. Your own message instead of Joi's `"items" must contain at least 1 items`.
4. The shape was fine; product 99 just does not exist. Joi checks the **shape** of data. Rules that need your data, like "does this product exist" or "is this email already taken", belong in your code (and, from Session 32, the database).

And `order` is a fully typed `OrderRequest`: `order.address.city` is a `string`, `order.items[0]` is an `OrderItem | undefined` (Session 06).

![An order request goes through the nested schemas: each item through itemSchema, the address through addressSchema; the client's totalPaise is stripped and the server computes the total from its own prices; problems are reported with their place, like items[0].quantity](images/26-runtime-validation-with-joi/order-request.gif)

---

## Mini Example 2: Checking Data from Another API

Session 23 checked a product API's answers with a hand-written `isProduct` and promised to do it with Joi. Copy `errors.ts` (with `AppError`, `errorMessage` and `errorChain`) from your `23-async` folder into `26-validation`

products/products.ts

```typescript
import Joi from "joi";
import { AppError, errorChain } from "../errors.ts";
import { validate } from "../validation.ts";

type Fetcher = (url: string) => Promise<Response>;

interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

const productSchema = Joi.object<Product, true>({
  id: Joi.number().integer().required(),
  name: Joi.string().required(),
  pricePaise: Joi.number().integer().min(0).required()
});

async function getProduct(id: number, fetcher: Fetcher): Promise<Product> {
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

const fakeFetch: Fetcher = async (url) => {
  if (url.endsWith("/1")) {
    return new Response(JSON.stringify({ id: 1, name: "Keyboard", pricePaise: 149900, stock: 12 }));
  }
  if (url.endsWith("/2")) {
    return new Response(JSON.stringify({ id: 2, name: "Mouse", price: 499 }));
  }
  return new Response(JSON.stringify({ error: "not found" }), { status: 404 });
};

for (const id of [1, 2, 3]) {
  try {
    const product = await getProduct(id, fakeFetch);
    console.log(product);
  } catch (err) {
    console.log(errorChain(err));
  }
}
```

```bash
node products/products.ts
```

```text
{ id: 1, name: 'Keyboard', pricePaise: 149900 }
product 2: unexpected data from the API <- product 2 is invalid: pricePaise is required
product 3: the API answered 404
```

Compare with Session 23

| Session 23 (`isProduct`)                     | Now (Joi)                                             |
| -------------------------------------------- | ----------------------------------------------------- |
| A type predicate of 9 lines                  | A schema of 3 rules, next to the interface             |
| "unexpected data from the API"               | The same, plus the cause: `pricePaise is required`    |
| Extra fields like `stock` pass through       | Removed: the product has exactly the `Product` shape  |
| Only the type is checked                     | Also rules: whole numbers, not negative               |

The `AppError` with status 502 ("bad gateway": the other server sent something wrong) keeps the `InvalidDataError` as its `cause` (Session 23), so a log shows both what failed and why. Session 29 tests `getProduct` with a fake fetcher, exactly like this one.

![getProduct asks the API, turns res.json() into unknown and validates it: product 1 comes out as a clean Product without the extra stock field, product 2 is wrapped in an AppError 502 whose cause says pricePaise is required, and product 3 stops at the 404](images/26-runtime-validation-with-joi/api-data.gif)

---

## Beginner Mistakes

### Mistake 1: Using value before checking error

`result.value` is `NewOrder | any` until you check `result.error`, which is just `any`, so typos pass `tsc`. Check `error` first, or use a `validate()` helper that does it for you.

---

### Mistake 2: A schema without a type

`Joi.object({ ... })` without a type argument makes `value` `any`. Write `Joi.object<NewOrder, true>({ ... })`.

---

### Mistake 3: Using the loose Joi.object<T> instead of the strict one

`Joi.object<NewOrder>` lets you leave out a property or use the wrong kind of rule. With `stripUnknown`, a property without a rule silently disappears while its type stays. Add `true`: `Joi.object<NewOrder, true>`.

---

### Mistake 4: Forgetting .required()

TypeScript cannot see `.required()`. Without it, a missing `customer` passes Joi, and `value.customer` is `undefined` while its type says `string`. Every property without `?` needs `.required()` or a `.default()`.

---

### Mistake 5: Writing the allowed values twice

`.valid("card", "upi", "paypal")` next to the type `"card" | "upi" | "cod"` passes `tsc`. Write the values once, `const PAYMENTS = [...] as const`, and use `(typeof PAYMENTS)[number]` and `.valid(...PAYMENTS)`.

---

### Mistake 6: Letting undefined through

An object schema accepts `undefined`, and a request without a body is exactly that. Validate with `schema.required()`.

---

### Mistake 7: An options variable without a type

`const options = { errors: { wrap: { label: false } } }` widens `false` to `boolean`, and `validate(data, options)` gives TS2345. Write `const options: Joi.ValidationOptions = { ... }`.

---

### Mistake 8: Using the input instead of the returned value

After validation, the input still has `"2"`, untrimmed texts and unknown keys like `totalPaise`. Use only the value that `validate` returns.

---

### Mistake 9: Checking business rules with the schema

"Product 99 does not exist" or "this email is taken" need your data, not a schema. Joi checks the shape; your code (and later the database) checks the rest.

---

### Mistake 10: Checking the same data again and again

Validate once, at the edge where data comes in. After that, the type is true, and functions inside the program take a `NewOrder`, not `unknown`.

---

## Practice Exercises

### Exercise 1

Rewrite Session 10's `isOrderBody` as a Joi schema with `Joi.object<OrderBody, true>`, and run the same two bodies through `validate`. Compare the messages.

### Exercise 2

Write `interface SignUp { name: string; email: string; age?: number; }` and a strict schema for it: a trimmed name of 2 to 50 characters, a lowercase email (`.email()`), and an optional whole-number age from 18 to 120. Test it with five inputs, including `{ "email": { "$gt": "" } }`.

### Exercise 3

Make every mistake from the table in [Keeping the Schema and the Type in Line](#keeping-the-schema-and-the-type-in-line) on purpose, once with `Joi.object<NewOrder>` and once with `Joi.object<NewOrder, true>`, and write down which ones `tsc` reports.

### Exercise 4

Add `LOG_LEVEL` to `env-settings.ts`, with the values from `load-config.ts` and the default `"info"`. Share one `as const` list between both files by moving it into its own module.

### Exercise 5

Write a schema for an update request, `Partial<NewOrder>` (Session 17), where every property is optional but at least one must be sent (`.min(1)` on the object, with your own message).

### Exercise 6

Write a schema for a list page's query string, `?page=2&limit=20&sort=price`: `page` and `limit` are whole numbers with defaults 1 and 20, `limit` at most 100, and `sort` is one of `"price"`, `"name"` or `"newest"`. Query values are always text, so validate `{ page: "2", limit: "500" }` and `{ sort: "cheapest" }`.

### Exercise 7

In Mini Example 1, add a `coupon` property: optional, uppercase letters and digits only, 4 to 12 characters (`.uppercase()`, `.alphanum()`). Apply 10% off for `"WELCOME10"` in `totalPaise`, and refuse unknown coupons with your own error.

### Exercise 8

In Mini Example 2, write `getProducts(fetcher)` for an API that answers with a list of products. Use `Joi.array().items(productSchema)` inside an object schema `{ products: ... }`, and make a fake fetcher whose list contains one broken product. What does the error message say about where the problem is?

---

## Interview Questions

### Why do you need runtime validation in TypeScript?

Types are removed before the code runs, so they cannot check data that arrives while it runs: request bodies, query strings, environment variables, files, other APIs. A validation library checks such data against rules when the program runs.

### Where should validation happen?

At the edges of the program, once, where data comes in. After that, the data has a type that the rest of the code can trust.

### What does schema.validate() return?

An object with `value` and, if something is wrong, `error`. It never throws. In Joi's types it is a union: when `error` is `undefined`, `value` has the schema's type; otherwise `value` is `any`.

### How do you give a Joi schema a TypeScript type?

With a type argument: `Joi.object<NewOrder>({ ... })`. With a second argument `true`, `Joi.object<NewOrder, true>`, every property needs a rule of the right kind.

### What can TypeScript not check about a Joi schema?

Whether a rule has `.required()`, and which values are in `.valid()`. A missing `.required()` lets `undefined` through while the type says otherwise. Keep `.required()` on every required property, and build `.valid()` and the type from one `as const` list.

### What do abortEarly, stripUnknown and convert do?

`abortEarly: false` reports every problem instead of the first. `stripUnknown: true` removes keys the schema does not list. `convert` (on by default) turns `"2"` into `2`, trims texts and similar; with `false`, such values are refused.

### Why must you use the returned value, not the input?

The returned value is the cleaned one: converted numbers, trimmed texts, defaults filled in, unknown keys removed. The input is still the raw data.

### Why does validate(undefined) pass, and how do you stop it?

Joi only checks what is there, and an optional object schema accepts "nothing". Use `schema.required()`, so `undefined` gives `value is required`.

### How do you validate environment variables?

With a schema over `process.env` when the program starts: `Joi.number()` turns `"3000"` into `3000`, `.default()` fills in missing values, `.required()` stops the start without a secret, and `stripUnknown` keeps only the variables you listed.

### What is the difference between a schema check and a business rule?

A schema checks the shape of data: types, lengths, formats, allowed values. A business rule needs your data, like "this product exists" or "this email is not taken", and belongs in your own code or the database.

### Joi or Zod?

Both check data at run time. Zod builds the TypeScript type from the schema; with Joi you write an interface and a schema and keep them in line, with help from `Joi.object<T, true>`. This course uses Joi, like the Node.js course.

### Why throw a custom error class from a validate helper?

Callers can recognise it with `instanceof` and use its list of problems: an API answers 400 with the list, a startup check stops the program, and an API client wraps it as the `cause` of its own error.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [25. Declaration Files and @types](25-declaration-files-and-types.md) | [Course outline](README.md#course-outline) | [27. tsconfig Deep Dive](27-tsconfig-deep-dive.md) |
