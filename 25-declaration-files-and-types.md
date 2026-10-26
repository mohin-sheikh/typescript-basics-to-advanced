# Declaration Files and @types

> **Session 25 of 33** | Part 6: Real Programs | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [What a Declaration File Is](#what-a-declaration-file-is)
- [Where TypeScript Finds Types](#where-typescript-finds-types)
- [A Package Without Types](#a-package-without-types)
- [Writing Types for a Package Yourself](#writing-types-for-a-package-yourself)
- [Installing Types from @types](#installing-types-from-types)
- [Packages That Bring Their Own Types](#packages-that-bring-their-own-types)
- [Matching @types/node to Your Node.js](#matching-typesnode-to-your-nodejs)
- [A .d.ts File for Your Own JavaScript](#a-dts-file-for-your-own-javascript)
- [Adding to Global Types with declare global](#adding-to-global-types-with-declare-global)
- [skipLibCheck and Your Own .d.ts Files](#skiplibcheck-and-your-own-dts-files)
- [Mini Example 1: Typing an Old JavaScript Module](#mini-example-1-typing-an-old-javascript-module)
- [Mini Example 2: Login Tokens with jsonwebtoken](#mini-example-2-login-tokens-with-jsonwebtoken)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `25-declaration-files` inside `ts-practice`, with the one-line `tsconfig.json`

25-declaration-files/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

This session installs a few npm packages. Run `npm install` inside `25-declaration-files`: npm looks for the nearest folder with a `package.json`, so the packages go into `ts-practice/node_modules`, next to `typescript`.

Every value in this course so far had a type, either because you wrote it in TypeScript, or because someone wrote the types for you: for JavaScript's built-ins, and for Node.js. This session is about the files that hold those types, **declaration files**, and about what to do when a JavaScript package, or a JavaScript file of your own, has none.

---

## What a Declaration File Is

Session 02 turned on `declaration` in your `tsconfig.json`, so every build also writes a `.d.ts` file. Build a module with a bit of everything

pricing.ts

```typescript
export interface CartLine {
  name: string;
  pricePaise: number;
  quantity: number;
}

export const GST_PERCENT = 18;

export function lineTotal(line: CartLine): number {
  return line.pricePaise * line.quantity;
}

export function cartTotal(lines: CartLine[]) {
  const subtotal = lines.reduce((sum, line) => sum + lineTotal(line), 0);
  return { subtotal, gst: Math.round((subtotal * GST_PERCENT) / 100) };
}

export class Cart {
  #lines: CartLine[] = [];

  add(line: CartLine): void {
    this.#lines.push(line);
  }

  get count(): number {
    return this.#lines.length;
  }
}
```

```bash
npx tsc
```

The build writes into `ts-practice/dist` (Session 02). Open `../dist/pricing.js`

```javascript
export const GST_PERCENT = 18;
export function lineTotal(line) {
    return line.pricePaise * line.quantity;
}
export function cartTotal(lines) {
    const subtotal = lines.reduce((sum, line) => sum + lineTotal(line), 0);
    return { subtotal, gst: Math.round((subtotal * GST_PERCENT) / 100) };
}
export class Cart {
    #lines = [];
    add(line) {
        this.#lines.push(line);
    }
    get count() {
        return this.#lines.length;
    }
}
```

and `../dist/pricing.d.ts`

```typescript
export interface CartLine {
    name: string;
    pricePaise: number;
    quantity: number;
}
export declare const GST_PERCENT = 18;
export declare function lineTotal(line: CartLine): number;
export declare function cartTotal(lines: CartLine[]): {
    subtotal: number;
    gst: number;
};
export declare class Cart {
    #private;
    add(line: CartLine): void;
    get count(): number;
}
//# sourceMappingURL=pricing.d.ts.map
```

The `.js` file is the code without the types. The `.d.ts` file is the types without the code. Together they say exactly what `pricing.ts` says.

| In `pricing.ts`                       | In `pricing.d.ts`                                              |
| ------------------------------------- | -------------------------------------------------------------- |
| `export interface CartLine { ... }`   | The same: an interface is only a type                          |
| `export const GST_PERCENT = 18;`      | `export declare const GST_PERCENT = 18;`                       |
| A function with a body                | Only its signature (name, parameters, return type), ending with `;` |
| `cartTotal` without a return type     | The return type TypeScript inferred, written out               |
| `#lines: CartLine[] = []`             | `#private;`: the class has private parts, but not even their names are shown |

`declare` means "this exists when the program runs, but not here; here is only its type". A `.d.ts` file contains nothing else: no function bodies, no values.

Why does this exist? Because npm packages ship JavaScript. Even a package written in TypeScript is usually published like this build: `.js` files for Node.js, plus `.d.ts` files for TypeScript. When you write `import Joi from "joi"`, Node.js runs somebody's built JavaScript, and TypeScript reads their declaration file.

![pricing.ts is split by a build into two files: pricing.js keeps the code without types, for Node.js, and pricing.d.ts keeps the types without code, for TypeScript](images/25-declaration-files-and-types/dts-split.gif)

---

## Where TypeScript Finds Types

Everything you use gets its types from a declaration file somewhere

| What you use                                  | Where its types come from                                        | Seen in      |
| --------------------------------------------- | ---------------------------------------------------------------- | ------------ |
| `Math`, `Array`, `Promise`, `JSON`            | `lib.es5.d.ts` and the other `lib` files of the `typescript` package | Session 05 |
| `process`, `node:fs`, `Buffer`                | The package `@types/node`                                        | Session 02   |
| A package that brings its own types, like Joi | A `.d.ts` file inside the package                                | This session |
| A JavaScript package without types, like `ms` | A separate package, `@types/ms`                                  | This session |
| Your own JavaScript files, or a package with no types anywhere | A `.d.ts` file you write                        | This session |

In VS Code, **F12** on `toFixed` opens `lib.es5.d.ts` (Session 05). F12 on anything from a package usually opens its `.d.ts` file the same way. Reading those files is a normal part of using a package.

One line of your `tsconfig.json` is about this too: `"types": ["node"]` (Session 02). It lists the type packages that are **global**, usable without an import, like `process`. Packages that you import find their types by themselves; you never add them to `types`.

![Five sources of types around your code: the lib files inside the typescript package, @types/node for Node.js, a package's own .d.ts file, a separate @types package, and .d.ts files you write yourself](images/25-declaration-files-and-types/type-sources.gif)

---

## A Package Without Types

`ms` is a tiny npm package that turns durations like `"7d"` into milliseconds. It is the reason `expiresIn: "7d"` works in the Node.js course ([Session 22](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/22-authentication-using-jwt.md)): `jsonwebtoken` uses `ms` inside

```bash
npm install ms
```

duration.ts

```typescript
import ms from "ms";

console.log(ms("7d"));
console.log(ms("2 hours"));
console.log(ms(90000));
```

```bash
npx tsc --noEmit
```

```text
duration.ts:1:16 - error TS7016: Could not find a declaration file for module 'ms'. 'D:/ts-practice/node_modules/ms/index.js' implicitly has an 'any' type.
  Try `npm i --save-dev @types/ms` if it exists or add a new declaration (.d.ts) file containing `declare module 'ms';`

1 import ms from "ms";
                 ~~~~


Found 1 error in duration.ts:1
```

```bash
node duration.ts
```

```text
604800000
7200000
2m
```

Node.js runs it: the package has its JavaScript, `index.js`, and Node.js needs nothing more. TypeScript found the same `index.js`, but no declaration file, so it knows nothing about `ms`. Without types, `ms` would be `any`, and `strict` does not allow an `any` that nobody wrote (`noImplicitAny`, Session 04).

`ms` is written in CommonJS (`module.exports = function ...`), which is why it is imported with a default import (Session 24).

The message names the two ways out, and they are the next two sections: write a declaration yourself, or install a package with the types, `@types/ms`.

![import ms reaches two readers: Node.js finds index.js and runs it; TypeScript finds the same index.js but no declaration file, so ms would be any, and strict mode reports TS7016 with two ways out](images/25-declaration-files-and-types/missing-types.gif)

---

## Writing Types for a Package Yourself

Types for a package go into a `.d.ts` file of your own. The file name does not matter; a folder `types` is a common place

types/ms.d.ts

```typescript
declare module "ms" {
  export default function ms(value: string): number;
}
```

`declare module "ms"` means "here are the types of the package `ms`". Inside, you write what the package exports, without code. `export default` matches the default import. No file has to import `types/ms.d.ts`: `tsc` reads every file in the folder, and the declaration counts everywhere.

```bash
npx tsc --noEmit
```

```text
duration.ts:5:16 - error TS2345: Argument of type 'number' is not assignable to parameter of type 'string'.

5 console.log(ms(90000));
                 ~~~~~


Found 1 error in duration.ts:5
```

TS7016 is gone, and `ms("7d")` is a `number` now. The new error is fair: your declaration only knows `ms(text)`, but the JavaScript also accepts a number and gives back a text like `"2m"`. Add that as a second signature, an overload (Session 12)

types/ms.d.ts

```typescript
declare module "ms" {
  export default function ms(value: string): number;
  export default function ms(value: number): string;
}
```

`npx tsc --noEmit` passes. Now make a typo, at the end of `duration.ts`

```typescript
console.log(ms("7 dayz"));
```

`tsc` still passes, and

```bash
node duration.ts
```

```text
604800000
7200000
2m
undefined
```

Your type says "any string", and `"7 dayz"` is a string. A declaration you write yourself is only as exact as you make it, and nobody compares it with the JavaScript.

The shortest declaration is the one the TS7016 message suggests. Replace the whole file with

types/ms.d.ts

```typescript
declare module "ms";
```

This **shorthand** declaration makes everything from `ms` type `any`. Even a line that cannot work passes `tsc`; add it at the end of `duration.ts`

```typescript
console.log(ms("7d").toUpperCase());
```

```text
604800000
7200000
2m
undefined
file:///D:/ts-practice/25-declaration-files/duration.ts:7
console.log(ms("7d").toUpperCase());
                     ^

TypeError: ms(...).toUpperCase is not a function
    at file:///D:/ts-practice/25-declaration-files/duration.ts:7:22
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

The shorthand only switches the error off. Use it for a moment, when you are stuck, never as the fix.

![Two hand-written declarations for ms: the full one types ms of a text as a number and ms of a number as a text, but still accepts the typo 7 dayz; the shorthand makes ms any, so even toUpperCase on a number passes tsc and crashes](images/25-declaration-files-and-types/declare-module.gif)

---

## Installing Types from @types

Before you write types yourself, look for them. Volunteers have written types for thousands of JavaScript packages in a GitHub project called **DefinitelyTyped**, and each one is published on npm as `@types/` plus the package name. You already use one: `@types/node`.

Delete the `types` folder and the `toUpperCase` line of `duration.ts` (keep the `"7 dayz"` line), then

```bash
npm install --save-dev @types/ms
npx tsc --noEmit
```

```text
duration.ts:6:16 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Argument of type '"7 dayz"' is not assignable to parameter of type 'StringValue'.

6 console.log(ms("7 dayz"));
                 ~~~~~~~~

  ../node_modules/@types/ms/index.d.ts:16:18 - The last overload is declared here.
    16 declare function ms(value: ms.StringValue): number;
                        ~~


Found 1 error in duration.ts:6
```

The typo is caught. The types come from `node_modules/@types/ms/index.d.ts`. It is short, so read it (F12 on `ms`). The two functions

```typescript
declare function ms(value: number, options?: { long: boolean }): string;
```

```typescript
declare function ms(value: ms.StringValue): number;
```

and the end of the file

```typescript
    type UnitAnyCase = Unit | Uppercase<Unit> | Lowercase<Unit>;

    type StringValue =
        | `${number}`
        | `${number}${UnitAnyCase}`
        | `${number} ${UnitAnyCase}`;
}

export = ms;
```

| Part                              | Meaning                                                              |
| --------------------------------- | -------------------------------------------------------------------- |
| Two `declare function ms(...)`    | Two overloads: a number gives a text, a `StringValue` gives a number |
| `declare namespace ms { ... }`    | An older way to group types under a name, so they are used as `ms.StringValue`. You can also write `import type { StringValue } from "ms"` (Mini Example 2) |
| `type Unit = ...`                 | Every unit name `ms` understands: `"Days"`, `"D"`, `"Hours"`, `"H"` and so on |
| `UnitAnyCase`                     | Each unit as written, in capitals, or in small letters (Session 19)  |
| `StringValue`                     | A template literal type (Session 19): a number, then a unit, with or without a space |
| `export = ms`                     | CommonJS's `module.exports = ms`, written as a type                  |

You will see `export =` in many packages' types. In your own declarations, write `export default` as in the last section ([Mistake 7](#mistake-7-writing-export--in-your-own-declarations)).

Remove the `"7 dayz"` line. One more line shows where the types stop helping

```typescript
console.log(ms("3600"));
```

It passes `tsc`, because `` `${number}` `` allows a number without a unit, and prints `3600`: that means 3600 **milliseconds**. In the Node.js course this was a real trap: `JWT_EXPIRES_IN=3600` made tokens that were valid for 3.6 seconds. Types catch many mistakes, not all of them. Mini Example 2 closes this gap.

Three more things about `@types` packages:

- **`--save-dev`**: types are only needed for checking, never when the code runs. Like `typescript`, they belong in `devDependencies`.
- **The version**: `@types/ms` 2.1.0 is for `ms` 2.1. The first two numbers of an `@types` package follow the package it describes.
- **Your own declaration wins**: if you keep a `declare module "ms"` file, TypeScript uses it instead of `@types/ms`. With the two-signature `types/ms.d.ts` from the last section still there, `ms("7 dayz")` passes again. Delete your own declaration when you install `@types`.

![The search for the types of ms: a declare module ms in your project wins; otherwise TypeScript looks in the package, then in node_modules/@types/ms; there StringValue accepts 7d and 2 hours, refuses 7 dayz, and accepts 3600, which means milliseconds](images/25-declaration-files-and-types/types-ms.gif)

---

## Packages That Bring Their Own Types

Many packages ship their own `.d.ts` files, often because they are written in TypeScript. Joi is one. Session 26 uses it, so install it now

```bash
npm install joi
```

joi-check.ts

```typescript
import Joi from "joi";

const nameSchema = Joi.strng().min(2);
console.log(nameSchema.validate("A").error?.message);
```

```text
joi-check.ts:3:24 - error TS2551: Property 'strng' does not exist on type 'Root'. Did you mean 'string'?

3 const nameSchema = Joi.strng().min(2);
                         ~~~~~

  ../node_modules/joi/lib/index.d.ts:2354:5 - 'string' is declared here.
    2354     string<TSchema = string>(): StringSchema<TSchema>;
             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in joi-check.ts:3
```

You installed no `@types/joi`, and the typo is still caught: the types come from `node_modules/joi/lib/index.d.ts`, inside the package itself. Fix `strng` to `string`

```bash
node joi-check.ts
```

```text
"value" length must be at least 2 characters long
```

How do you know which kind of package you have, before you install it? Ask npm

```bash
npm view joi types
```

```text
lib/index.d.ts
```

A path means the package brings its own types (the field `"types"` in its `package.json`). `npm view ms types` prints nothing. On the package's page on npmjs.com, a small **TS** icon next to the name means "own types", and a **DT** icon means "types in an `@types` package".

Do not install an `@types` package for a package that has its own types. For many of them, an old `@types` package still exists, but it is an empty stub

```bash
npm view @types/joi deprecated
```

```text
This is a stub types definition. joi provides its own type definitions, so you do not need this installed.
```

The packages of the Node.js course

| Package         | Types                            |
| --------------- | -------------------------------- |
| `express`       | `@types/express` (Session 31)    |
| `jsonwebtoken`  | `@types/jsonwebtoken`            |
| `mongoose`      | Its own (Session 32)             |
| `joi`           | Its own (Session 26)             |
| `bcryptjs`      | Its own                          |
| `dotenv`        | Its own                          |

![Two kinds of packages: joi has a types field in its package.json pointing to its own lib/index.d.ts, while ms has none and gets its types from @types/ms; @types/joi is an empty stub that only says joi has its own types](images/25-declaration-files-and-types/own-types.gif)

---

## Matching @types/node to Your Node.js

Open `ts-practice/package.json`. Session 02 installed the newest `@types/node`, which was version 26 when this course was written (`"@types/node": "^26.6.5"`). The course runs on Node.js 24. `@types/node` 26 describes Node.js 26, including functions that Node.js 24 does not have. One example is `process.addUncaughtExceptionCaptureCallback`, added in Node.js 25.9. You will not need it; it only shows the problem

crash-handler.ts

```typescript
process.addUncaughtExceptionCaptureCallback((err: unknown) => {
  console.log("caught:", err);
  return true;
});
```

`npx tsc --noEmit` finds nothing wrong. Node.js 24 disagrees

```bash
node crash-handler.ts
```

```text
file:///D:/ts-practice/25-declaration-files/crash-handler.ts:1
process.addUncaughtExceptionCaptureCallback((err         ) => {
        ^

TypeError: process.addUncaughtExceptionCaptureCallback is not a function
    at file:///D:/ts-practice/25-declaration-files/crash-handler.ts:1:9
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

(The gap after `err` is where `: unknown` was. Node.js replaces types with spaces, so the line and column numbers still match your file.)

The rule: the first number of `@types/node` matches the first number of `node --version`

```bash
node --version
```

```text
v24.19.0
```

```bash
npm install --save-dev @types/node@24
npx tsc --noEmit
```

```text
crash-handler.ts:1:9 - error TS2551: Property 'addUncaughtExceptionCaptureCallback' does not exist on type 'Process'. Did you mean 'hasUncaughtExceptionCaptureCallback'?

1 process.addUncaughtExceptionCaptureCallback((err: unknown) => {
          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  ../node_modules/@types/node/process.d.ts:1308:17 - 'hasUncaughtExceptionCaptureCallback' is declared here.
    1308                 hasUncaughtExceptionCaptureCallback(): boolean;
                         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in crash-handler.ts:1
```

Now `tsc` knows what your Node.js knows. `package.json` says `"@types/node": "^24.19.2"`, and the `^` keeps it at 24 when you update. Delete `crash-handler.ts`. From here on, the course uses `@types/node` 24; when a project moves to a newer Node.js, move `@types/node` with it.

![@types/node 26 describes Node.js 26, so tsc accepts a function that Node.js 24 does not have and the program crashes; with @types/node 24 the types match the Node.js that runs the code, and tsc reports TS2551](images/25-declaration-files-and-types/types-node-version.gif)

---

## A .d.ts File for Your Own JavaScript

Real projects often contain old JavaScript files, for example coupon code from an older project. This one is an ES module, because the package says `"type": "module"`

legacy/coupons.js

```javascript
export const COUPONS = ["WELCOME10", "FESTIVE20"];

export function applyCoupon(pricePaise, code) {
  if (code === "WELCOME10") {
    return Math.round(pricePaise * 0.9);
  }
  if (code === "FESTIVE20") {
    return Math.round(pricePaise * 0.8);
  }
  return pricePaise;
}
```

checkout.ts

```typescript
import { COUPONS, applyCoupon } from "./legacy/coupons.js";

console.log(COUPONS.join(", "));
console.log(applyCoupon(49900, "WELCOME10"));
console.log(applyCoupon("49900", 10));
```

The import path ends with `.js`, because the file really is JavaScript: Session 24's rule is to write the real file name.

```text
checkout.ts:1:38 - error TS7016: Could not find a declaration file for module './legacy/coupons.js'. 'D:/ts-practice/25-declaration-files/legacy/coupons.js' implicitly has an 'any' type.

1 import { COUPONS, applyCoupon } from "./legacy/coupons.js";
                                       ~~~~~~~~~~~~~~~~~~~~~


Found 1 error in checkout.ts:1
```

The same TS7016 as for `ms`, without the `@types` hint. Node.js runs it anyway

```bash
node checkout.ts
```

```text
WELCOME10, FESTIVE20
44910
49900
```

The last call swapped the arguments' types: a text for the price and a number for the code. JavaScript does not mind; no coupon matches, so the text comes back unchanged. Write the types in a file with the same name and the ending `.d.ts`, next to the JavaScript

legacy/coupons.d.ts

```typescript
export declare const COUPONS: string[];

export declare function applyCoupon(pricePaise: number, code: string): number;
```

```text
checkout.ts:5:25 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

5 console.log(applyCoupon("49900", 10));
                          ~~~~~~~


Found 1 error in checkout.ts:5
```

When something imports `coupons.js`, TypeScript now reads `coupons.d.ts`. Node.js still runs `coupons.js` and never looks at the `.d.ts`. Fix the last line

checkout.ts

```typescript
import { COUPONS, applyCoupon } from "./legacy/coupons.js";

console.log(COUPONS.join(", "));
console.log(applyCoupon(49900, "WELCOME10"));
console.log(applyCoupon(49900, "FESTIVE20"));
```

```text
WELCOME10, FESTIVE20
44910
39920
```

TypeScript does not compare `coupons.d.ts` with `coupons.js`. If the declaration is wrong, or the JavaScript changes later, nobody notices ([Mini Example 1](#mini-example-1-typing-an-old-javascript-module)). Session 28 shows another way: letting TypeScript check the JavaScript files themselves.

![checkout.ts imports ./legacy/coupons.js; Node.js loads coupons.js and runs it, while TypeScript reads coupons.d.ts with the same name next to it; nothing compares the two files](images/25-declaration-files-and-types/js-with-dts.gif)

---

## Adding to Global Types with declare global

Sometimes the types exist, and you want to add to them. Session 08 showed that two interfaces with the same name **merge**. `@types/node` describes `process.env` with an interface

```typescript
interface ProcessEnv extends Dict<string> {}
```

`Dict<string>` is an index signature, `{ [key: string]: string | undefined }` (Session 07): every name is allowed, and every value might be missing. Add your own variables in a declaration file

env.d.ts

```typescript
export {};

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV?: "development" | "production" | "test";
      PORT?: string;
    }
  }
}
```

| Line                          | Meaning                                                                      |
| ----------------------------- | ---------------------------------------------------------------------------- |
| `export {};`                  | Makes the file a module. `moduleDetection: "force"` (Session 24) does not apply to `.d.ts` files, and `declare global` only works in a module |
| `declare global { ... }`      | "Add this to the global types", instead of declaring something inside this file |
| `namespace NodeJS { ... }`    | The group in which `@types/node` keeps `ProcessEnv`. Same name, same group   |
| `interface ProcessEnv { ... }` | Merges with the `ProcessEnv` of `@types/node`                              |

server-mode.ts

```typescript
const mode = process.env.NODE_ENV;
console.log(`mode: ${mode}`);

if (mode === "prod") {
  console.log("running in production");
}
```

```text
server-mode.ts:4:5 - error TS2367: This comparison appears to be unintentional because the types '"development" | "production" | "test" | undefined' and '"prod"' have no overlap.

4 if (mode === "prod") {
      ~~~~~~~~~~~~~~~


Found 1 error in server-mode.ts:4
```

`mode` has your literal union now (Session 09), and the misspelled value is caught. Change `"prod"` to `"production"`. Know the limits too. Add a line with a misspelled name

```typescript
console.log(`port: ${process.env.PROT}`);
```

`tsc` passes: the index signature of `@types/node` is still there, so every name is allowed. And the values are not checked at all

.env

```text
NODE_ENV=staging
```

```bash
node --env-file=.env server-mode.ts
```

```text
mode: staging
port: undefined
```

The type says `NODE_ENV` is one of three values, or missing; when the program runs, it is `"staging"`. Like every declaration, it is a promise that nobody checks. These types help while you write code, with suggestions and with comparisons like the one above. Checking the real values needs code that runs: Session 26 does that with Joi.

Session 31 uses exactly this pattern, `declare global` with a namespace and a merged interface, to add a logged-in `user` to Express's request.

![The ProcessEnv of @types/node, with any name allowed, merges with your ProcessEnv with NODE_ENV and PORT; the merged type catches the comparison with prod, but still allows PROT, and at run time NODE_ENV is staging anyway](images/25-declaration-files-and-types/declare-global.gif)

---

## skipLibCheck and Your Own .d.ts Files

Your `tsconfig.json` has `"skipLibCheck": true` (Session 02): do not check `.d.ts` files. That makes `tsc` faster, and it skips problems in the types of installed packages, which you could not fix anyway. But it skips **every** `.d.ts` file, including yours. Put a typo into `legacy/coupons.d.ts`

```typescript
export declare function applyCoupon(pricePaise: numbr, code: string): number;
```

and a wrong call at the end of `checkout.ts`

```typescript
console.log(applyCoupon("49900", "FESTIVE20"));
```

`npx tsc --noEmit` finds nothing. TypeScript cannot use the unknown type `numbr`, so it treats the parameter like `any`, and the wrong call slips through. Check once with the option turned off

```bash
npx tsc --noEmit --skipLibCheck false
```

```text
legacy/coupons.d.ts:3:49 - error TS2552: Cannot find name 'numbr'. Did you mean 'number'?

3 export declare function applyCoupon(pricePaise: numbr, code: string): number;
                                                  ~~~~~


Found 1 error in legacy/coupons.d.ts:3
```

More mistakes that only show up this way

| Mistake in your own `.d.ts` file                | With `skipLibCheck: true`              | With `--skipLibCheck false`                     |
| ----------------------------------------------- | -------------------------------------- | ----------------------------------------------- |
| A misspelled type name                          | No error; the type acts like `any`     | TS2552 "Cannot find name"                       |
| A function with a body                          | No error                               | TS1183 "An implementation cannot be declared in ambient contexts." |
| `declare global` without `export {}`            | No error, and your additions are ignored | TS2669 "Augmentations for the global scope can only be directly nested in external modules or ambient module declarations." |
| `export = ms` in your own `declare module`      | No error                               | TS1203 "Export assignment cannot be used when targeting ECMAScript modules." |

Keep `skipLibCheck: true`, and run `npx tsc --noEmit --skipLibCheck false` whenever you change a `.d.ts` file of your own. In this folder it reports only your own mistakes: the types of the packages used in this session pass it. Fix `numbr` and remove the wrong call.

![tsc with skipLibCheck checks the .ts files and walks past every .d.ts file, so a typo in coupons.d.ts stays hidden; with skipLibCheck false it also reads the .d.ts files and reports TS2552](images/25-declaration-files-and-types/skiplibcheck.gif)

---

## Mini Example 1: Typing an Old JavaScript Module

The billing team has an old helper module in plain JavaScript, and new TypeScript code needs it

```
billing/
  legacy/invoice.js
  legacy/invoice.d.ts
  print-invoice.ts
```

billing/legacy/invoice.js

```javascript
export const GST_PERCENT = 18;

export function invoiceNumber(id) {
  return "INV-" + String(id).padStart(5, "0");
}

export function buildInvoice(customer, lines) {
  const subtotal = lines.reduce((sum, line) => sum + line.pricePaise * line.quantity, 0);
  const gst = Math.round((subtotal * GST_PERCENT) / 100);
  return { customer, lines, subtotal, gst, total: subtotal + gst };
}

export function formatPaise(paise) {
  return "Rs " + (paise / 100).toFixed(2);
}
```

A teammate writes the declaration from memory of what the module does

billing/legacy/invoice.d.ts

```typescript
export declare const GST_PERCENT: number;

export interface InvoiceLine {
  name: string;
  pricePaise: number;
  quantity: number;
}

export interface Invoice {
  customer: string;
  lines: InvoiceLine[];
  subtotalPaise: number;
  gstPaise: number;
  totalPaise: number;
}

export declare function invoiceNumber(id: number): string;
export declare function buildInvoice(customer: string, lines: InvoiceLine[]): Invoice;
export declare function formatPaise(paise: number): string;
```

The declaration exports two interfaces, `InvoiceLine` and `Invoice`, that the JavaScript never names. That is allowed: a `.d.ts` file can describe shapes that only exist for TypeScript.

billing/print-invoice.ts

```typescript
import { buildInvoice, formatPaise, invoiceNumber } from "./legacy/invoice.js";
import type { InvoiceLine } from "./legacy/invoice.js";

const lines: InvoiceLine[] = [
  { name: "Keyboard", pricePaise: 149900, quantity: 1 },
  { name: "Mouse", pricePaise: 49900, quantity: 2 }
];

const invoice = buildInvoice("Asha", lines);

console.log(`${invoiceNumber(42)} for ${invoice.customer}`);
for (const line of invoice.lines) {
  console.log(`  ${line.quantity} x ${line.name}`);
}
console.log(`Subtotal: ${formatPaise(invoice.subtotalPaise)}`);
console.log(`GST:      ${formatPaise(invoice.gstPaise)}`);
console.log(`Total:    ${formatPaise(invoice.totalPaise)}`);
```

```bash
npx tsc --noEmit
node billing/print-invoice.ts
```

```text
INV-00042 for Asha
  1 x Keyboard
  2 x Mouse
Subtotal: Rs NaN
GST:      Rs NaN
Total:    Rs NaN
```

`tsc` found nothing, and the money is `NaN`. Look at what `buildInvoice` really returns: `{ customer, lines, subtotal, gst, total: subtotal + gst }`. The JavaScript says `subtotal`, `gst` and `total`; the declaration promised `subtotalPaise`, `gstPaise` and `totalPaise`. TypeScript believed the declaration. When the program runs, `invoice.subtotalPaise` is `undefined`, and `undefined / 100` is `NaN`.

Make the declaration match the JavaScript

billing/legacy/invoice.d.ts

```typescript
export interface Invoice {
  customer: string;
  lines: InvoiceLine[];
  subtotal: number;
  gst: number;
  total: number;
}
```

```text
billing/print-invoice.ts:15:46 - error TS2339: Property 'subtotalPaise' does not exist on type 'Invoice'.

15 console.log(`Subtotal: ${formatPaise(invoice.subtotalPaise)}`);
                                                ~~~~~~~~~~~~~

billing/print-invoice.ts:16:46 - error TS2339: Property 'gstPaise' does not exist on type 'Invoice'.

16 console.log(`GST:      ${formatPaise(invoice.gstPaise)}`);
                                                ~~~~~~~~

billing/print-invoice.ts:17:46 - error TS2339: Property 'totalPaise' does not exist on type 'Invoice'.

17 console.log(`Total:    ${formatPaise(invoice.totalPaise)}`);
                                                ~~~~~~~~~~


Found 3 errors in the same file, starting at: billing/print-invoice.ts:15
```

Now the errors point at every line that trusted the wrong names. Fix them

billing/print-invoice.ts (the last three lines)

```typescript
console.log(`Subtotal: ${formatPaise(invoice.subtotal)}`);
console.log(`GST:      ${formatPaise(invoice.gst)}`);
console.log(`Total:    ${formatPaise(invoice.total)}`);
```

```text
INV-00042 for Asha
  1 x Keyboard
  2 x Mouse
Subtotal: Rs 2497.00
GST:      Rs 449.46
Total:    Rs 2946.46
```

A correct declaration protects every caller. Write the mouse's quantity as a text

```typescript
  { name: "Mouse", pricePaise: 49900, quantity: "2" }
```

```text
billing/print-invoice.ts:6:39 - error TS2322: Type 'string' is not assignable to type 'number'.

6   { name: "Mouse", pricePaise: 49900, quantity: "2" }
                                        ~~~~~~~~

  billing/legacy/invoice.d.ts:6:3 - The expected type comes from property 'quantity' which is declared here on type 'InvoiceLine'
    6   quantity: number;
        ~~~~~~~~


Found 1 error in billing/print-invoice.ts:6
```

The second location points into your declaration file. Change it back to `2`.

When you write a `.d.ts` file by hand: read the JavaScript, especially what each function returns, and run the code once with real data. TypeScript can only check the code against the declaration, never the declaration against the code.

![buildInvoice in invoice.js returns subtotal, gst and total, but invoice.d.ts promises subtotalPaise, gstPaise and totalPaise; tsc trusts the declaration and the output is Rs NaN; after the declaration is fixed, tsc points at the three lines and the totals come out right](images/25-declaration-files-and-types/invoice-dts.gif)

---

## Mini Example 2: Login Tokens with jsonwebtoken

The Node.js course signs and checks login tokens with `jsonwebtoken` (Session 22). Here is the same in TypeScript, in a folder `auth/`. `jsonwebtoken` is JavaScript without its own types; its types are in `@types/jsonwebtoken`

```bash
npm install jsonwebtoken
npm install --save-dev @types/jsonwebtoken
```

`@types/jsonwebtoken` uses a type from `@types/ms`. Near the top of its `index.d.ts`

```typescript
import type { StringValue } from "ms";
```

So npm installs `@types/ms` along with it (you already have it).

**The import.** A first try, in a scratch file

auth/try-jwt.ts

```typescript
import { sign, verify } from "jsonwebtoken";

const secret = "change-me-in-production";
const token = sign({ id: 7 }, secret, { expiresIn: "7d" });
console.log(verify(token, secret));
```

`npx tsc --noEmit` passes: the types say the package exports `sign` and `verify`. Node.js disagrees

```bash
node auth/try-jwt.ts
```

```text
file:///D:/ts-practice/25-declaration-files/auth/try-jwt.ts:1
import { sign, verify } from "jsonwebtoken";
         ^^^^
SyntaxError: Named export 'sign' not found. The requested module 'jsonwebtoken' is a CommonJS module, which may not support all module.exports as named exports.
CommonJS modules can always be imported via the default export, for example using:

import pkg from 'jsonwebtoken';
const { sign, verify } = pkg;

    at #asyncInstantiate (node:internal/modules/esm/module_job:327:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:431:5)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`jsonwebtoken` is a CommonJS package. Node.js offers named imports from CommonJS only when it can find the names by reading the code, and here it cannot. The types describe what `module.exports` contains, not whether Node.js can see the names, so `tsc` cannot warn you. Do what the message says: use the default import (as promised in Session 24)

auth/try-jwt.ts

```typescript
import jwt from "jsonwebtoken";

const secret = "change-me-in-production";
const token = jwt.sign({ id: 7 }, secret, { expiresIn: "7d" });
console.log(jwt.verify(token, secret));
```

```text
{ id: 7, iat: 1793000666, exp: 1793605466 }
```

Your numbers will differ: `iat` (issued at) and `exp` (expires) are times in seconds, and `exp - iat` is 604800 seconds, 7 days.

**The expiry from .env.** The Node.js course read the expiry with `process.env.JWT_EXPIRES_IN || "7d"`. In TypeScript, with `??` (Session 11)

```typescript
const token = jwt.sign({ id: 7 }, secret, { expiresIn: process.env.JWT_EXPIRES_IN ?? "7d" });
```

```text
auth/try-jwt.ts:4:45 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Object literal may only specify known properties, and 'expiresIn' does not exist in type 'SignCallback'.

4 const token = jwt.sign({ id: 7 }, secret, { expiresIn: process.env.JWT_EXPIRES_IN ?? "7d" });
                                              ~~~~~~~~~

  ../node_modules/@types/jsonwebtoken/index.d.ts:189:17 - The last overload is declared here.
    189 export function sign(
                        ~~~~


Found 1 error in auth/try-jwt.ts:4
```

A confusing message: what is a `SignCallback`? `jwt.sign` has five overloads, and TS2769 shows only the error of the last one (Session 05). The last one is the form with a callback, which you never meant to use. To get a clear message, give the options their own variable, with the options type from the package

```typescript
const options: jwt.SignOptions = { expiresIn: process.env.JWT_EXPIRES_IN ?? "7d" };
const token = jwt.sign({ id: 7 }, secret, options);
```

```text
auth/try-jwt.ts:4:36 - error TS2322: Type 'string' is not assignable to type 'number | StringValue'.

4 const options: jwt.SignOptions = { expiresIn: process.env.JWT_EXPIRES_IN ?? "7d" };
                                     ~~~~~~~~~

  ../node_modules/@types/jsonwebtoken/index.d.ts:43:5 - The expected type comes from property 'expiresIn' which is declared here on type 'SignOptions'
    43     expiresIn?: StringValue | number;
           ~~~~~~~~~


Found 1 error in auth/try-jwt.ts:4
```

Now the real problem is visible: the variable from `.env` is any `string`, and `expiresIn` wants a number or a `StringValue` from `@types/ms`. (The default import `jwt` also gives you the package's types: `jwt.SignOptions`.)

So check the text, with a type predicate (Session 10). Delete `auth/try-jwt.ts`; the real code goes into a module

auth/tokens.ts

```typescript
import jwt from "jsonwebtoken";
import type { StringValue } from "ms";

interface TokenSettings {
  secret: string;
  expiresIn: StringValue;
}

function isDuration(text: string): text is StringValue {
  return /^[0-9]+[smhdwy]$/.test(text);
}

function readSettings(): TokenSettings {
  const secret = process.env.JWT_SECRET;
  const expiresIn = process.env.JWT_EXPIRES_IN ?? "7d";
  if (secret === undefined) {
    throw new Error("JWT_SECRET is missing");
  }
  if (!isDuration(expiresIn)) {
    throw new Error(`JWT_EXPIRES_IN needs a number and a unit, like 15m, 1h or 7d, not "${expiresIn}"`);
  }
  return { secret, expiresIn };
}

const settings = readSettings();

export function createToken(userId: number): string {
  return jwt.sign({ id: userId }, settings.secret, { expiresIn: settings.expiresIn });
}
```

| Part                                    | Why                                                              |
| --------------------------------------- | ---------------------------------------------------------------- |
| `import type { StringValue } from "ms"` | The type from `@types/ms`. A type-only import is removed before the code runs, so this file never loads `ms` |
| `isDuration(text): text is StringValue` | A regular expression (JavaScript course, [Session 08](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/08-strings-and-template-literals.md)): digits, then one unit letter, like `15m`, `1h`, `7d`. When it returns `true`, TypeScript treats `text` as a `StringValue` |
| `readSettings()`                        | Reads both variables once, when the module loads, and stops the program with a clear message when one is wrong |

The check also closes the gap from the `ms` section: `"3600"` has no unit, so `isDuration` refuses it, although `StringValue` allows it.

**Reading the token.** A first version

```typescript
export function readUserId(token: string): number {
  const payload = jwt.verify(token, settings.secret);
  return payload.id;
}
```

```text
auth/tokens.ts:33:18 - error TS2339: Property 'id' does not exist on type 'string | JwtPayload'.
  Property 'id' does not exist on type 'string'.

33   return payload.id;
                    ~~


Found 1 error in auth/tokens.ts:33
```

`jwt.verify` returns `JwtPayload | string`, because a token can carry a plain text instead of an object. Rule out the text first

```typescript
  if (typeof payload === "string") {
    throw new Error("the token has no user id");
  }
  return payload.id;
```

`tsc` passes. Too easily: look at `JwtPayload` in `@types/jsonwebtoken`

```typescript
export interface JwtPayload {
    [key: string]: any;
```

Every property name gives `any`, so `payload.id` is `any`, and returning it as a `number` is not checked at all. Here is a token from an older version of the app, which stored `userId` instead of `id`

auth/old-token.ts

```typescript
import jwt from "jsonwebtoken";
import { readUserId } from "./tokens.ts";

const oldToken = jwt.sign({ userId: 7 }, process.env.JWT_SECRET ?? "");
console.log(`user id: ${readUserId(oldToken)}`);
```

```text
user id: undefined
```

A function typed to return a `number` returned `undefined`. `@types` packages are written by people, and some of them use `any` where they cannot know better. Check the value itself

auth/tokens.ts (readUserId)

```typescript
export function readUserId(token: string): number {
  const payload = jwt.verify(token, settings.secret);
  if (typeof payload === "string" || typeof payload.id !== "number") {
    throw new Error("the token has no user id");
  }
  return payload.id;
}
```

**Running it.**

auth/login.ts

```typescript
import jwt from "jsonwebtoken";
import { createToken, readUserId } from "./tokens.ts";

const token = createToken(7);
console.log(`token: ${token.slice(0, 20)}...`);
console.log(`user id: ${readUserId(token)}`);

for (const bad of [token + "x", "hello"]) {
  try {
    readUserId(bad);
  } catch (err) {
    if (err instanceof jwt.JsonWebTokenError) {
      console.log(`refused: ${err.message}`);
    } else {
      throw err;
    }
  }
}
```

auth/.env

```text
JWT_SECRET=change-me-in-production
JWT_EXPIRES_IN=7d
```

```bash
node --env-file=auth/.env auth/login.ts
```

```text
token: eyJhbGciOiJIUzI1NiIs...
user id: 7
refused: invalid signature
refused: jwt malformed
```

`JsonWebTokenError` is a class that the package exports, and `@types/jsonwebtoken` declares it, so `instanceof` narrows `err` from `unknown` (Sessions 21 and 23). The old token is refused now, with a message instead of `undefined`

```bash
node --env-file=auth/.env auth/old-token.ts
```

```text
file:///D:/ts-practice/25-declaration-files/auth/tokens.ts:34
    throw new Error("the token has no user id");
          ^

Error: the token has no user id
    at readUserId (file:///D:/ts-practice/25-declaration-files/auth/tokens.ts:34:11)
    at file:///D:/ts-practice/25-declaration-files/auth/old-token.ts:5:25
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Finally, the trap from the Node.js course. Change the expiry

auth/.env

```text
JWT_SECRET=change-me-in-production
JWT_EXPIRES_IN=3600
```

```text
file:///D:/ts-practice/25-declaration-files/auth/tokens.ts:20
    throw new Error(`JWT_EXPIRES_IN needs a number and a unit, like 15m, 1h or 7d, not "${expiresIn}"`);
          ^

Error: JWT_EXPIRES_IN needs a number and a unit, like 15m, 1h or 7d, not "3600"
    at readSettings (file:///D:/ts-practice/25-declaration-files/auth/tokens.ts:20:11)
    at file:///D:/ts-practice/25-declaration-files/auth/tokens.ts:25:18
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

The program stops at the start, with a message that says what to fix, instead of handing out tokens that expire after 3.6 seconds. Set it back to `7d`.

![A text from .env passes the isDuration check and becomes a StringValue that jwt.sign accepts, while 3600 is refused at startup; jwt.verify returns a string or a JwtPayload whose properties are any, and two checks turn it into a number](images/25-declaration-files-and-types/jwt-types.gif)

---

## Beginner Mistakes

### Mistake 1: Silencing TS7016 with the shorthand

`declare module "ms";` makes the error go away by making everything from the package `any`. `ms("7d").toUpperCase()` then passes `tsc` and crashes. Install the `@types` package, or write a real declaration.

---

### Mistake 2: Keeping your own declaration after installing @types

A `declare module "ms"` in your project wins over `@types/ms`, so the better types are never used. Delete your declaration when you install `@types`.

---

### Mistake 3: Installing @types for a package that has its own types

`@types/joi` is an empty stub: "joi provides its own type definitions". Check with `npm view <package> types` before installing `@types`.

---

### Mistake 4: Using an @types/node newer than your Node.js

`@types/node` 26 lets `process.addUncaughtExceptionCaptureCallback` pass `tsc`, and Node.js 24 says "is not a function". Install the `@types/node` version that matches `node --version`.

---

### Mistake 5: Putting code in a .d.ts file

A function with a body in a `.d.ts` file is `error TS1183`, but `skipLibCheck` hides it. A `.d.ts` file holds only types; the code belongs in the `.js` or `.ts` file.

---

### Mistake 6: Forgetting export {} in a file with declare global

Without `export {}`, a `.d.ts` file is not a module, and `declare global` is ignored without any message (TS2669 only with `--skipLibCheck false`). Start such a file with `export {};`.

---

### Mistake 7: Writing export = in your own declarations

`export = ms` is how many packages' types describe CommonJS, but in your own `declare module` it is `error TS1203` (hidden by `skipLibCheck`). Write `export default` for a package you import with a default import.

---

### Mistake 8: Trusting a hand-written .d.ts without running the code

TypeScript checks your code against the declaration, never the declaration against the JavaScript. A wrong property name gave `Rs NaN` with no error in Mini Example 1. Read what the JavaScript returns, and run it once.

---

### Mistake 9: Using named imports from a CommonJS package

`import { sign } from "jsonwebtoken"` passes `tsc`, and Node.js stops with "Named export 'sign' not found". Use the default import, `import jwt from "jsonwebtoken"`, and `jwt.sign`.

---

### Mistake 10: Reading only the last overload's message

TS2769 shows the error of the last overload, which can be about a form you never meant to use ("does not exist in type 'SignCallback'"). Put the argument into a variable with the expected type, like `jwt.SignOptions`, to get a clear message.

---

### Mistake 11: Thinking that types for process.env check the values

Your `ProcessEnv` types say what you expect, but `NODE_ENV=staging` still arrives as `"staging"`. Check the real values when the program starts (Session 26).

---

## Practice Exercises

### Exercise 1

Write `legacy/units.js` with `export function toPaise(rupees)` and `export function toRupees(paise)`, and a `units.d.ts` for it. Use both from a `.ts` file, make one wrong call to see the error, and run `npx tsc --noEmit --skipLibCheck false`.

### Exercise 2

Add to `pricing.ts` a function with an optional parameter, a function with a default parameter, and a method in `Cart` that returns a tuple. Build, and read how each one looks in `../dist/pricing.d.ts`.

### Exercise 3

For `cors`, `morgan`, `multer`, `winston` and `helmet`, find out whether each package brings its own types or needs an `@types` package. Use `npm view <name> types` and `npm view @types/<name> deprecated`.

### Exercise 4

Read `node_modules/@types/ms/index.d.ts` again. Write five texts that `StringValue` accepts and five that it refuses, and check your answers with `ms(...)` calls in a file.

### Exercise 5

Add `LOG_LEVEL?: "debug" | "info" | "warn" | "error"` to `ProcessEnv` in `env.d.ts`. Write a function `readLogLevel()` that returns the level, or `"info"` when it is missing. Then set `LOG_LEVEL=verbose` in `.env` and see what your function returns.

### Exercise 6

Make the four mistakes from the table in the skipLibCheck section on purpose, one at a time. Check that `npx tsc --noEmit` stays silent and that `npx tsc --noEmit --skipLibCheck false` finds each one.

### Exercise 7

In Mini Example 1, give `buildInvoice` a third parameter `discountPaise` with a default of `0`, subtracted before GST. Update `invoice.d.ts` so the parameter is optional, and print an invoice with and without a discount.

### Exercise 8

In Mini Example 2, put a role (`"admin"` or `"customer"`) into the token, and write `readUser(token)` that returns `{ id: number; role: "admin" | "customer" }` after checking both values. Then sign a token that has already expired and handle `jwt.TokenExpiredError` separately, printing its `expiredAt`.

---

## Interview Questions

### What is a .d.ts file?

A declaration file: it describes the types of a module without its code. TypeScript reads it to check the code that uses the module, while the JavaScript file next to it is what runs.

### What does the declare keyword mean?

"This exists when the program runs, but it is defined somewhere else; here is only its type." Declaration files are made of such declarations, with no function bodies and no values.

### Where does TypeScript find the types of an npm package?

First in the package itself (the `"types"` field of its `package.json`). If the package has none, in `node_modules/@types/<name>`. A `declare module "<name>"` in your own project wins over both.

### What is DefinitelyTyped?

A GitHub project where volunteers write types for JavaScript packages that have none. Each one is published on npm as `@types/<name>`, like `@types/express` or `@types/ms`.

### What does TS7016 mean, and how do you fix it?

"Could not find a declaration file for module": the JavaScript exists, but TypeScript has no types for it, so it would be `any`. Install the `@types` package if it exists; otherwise write a declaration. For your own JavaScript file, write a `.d.ts` file with the same name next to it.

### Why do @types packages go into devDependencies?

They are only needed to check the code. Nothing reads them when the program runs.

### What does declare module "x"; do, and why is it risky?

It declares the package `x` without describing it, so everything from it is `any`. The error disappears, and so do all the checks.

### What does export = mean in a .d.ts file?

It describes a CommonJS module whose `module.exports` is the given value, like `export = ms` for `module.exports = ms`. It is common in `@types` packages; in your own declarations in this setup, use `export default`.

### How do you add your own environment variables to the type of process.env?

In a `.d.ts` file that starts with `export {};`, write `declare global { namespace NodeJS { interface ProcessEnv { ... } } }`. The interface merges with the one from `@types/node`. It only describes what you expect; it does not check the real values.

### What does skipLibCheck do, and what is the catch?

It tells `tsc` not to check `.d.ts` files, which is faster and skips problems in installed packages. The catch: it also skips your own `.d.ts` files, so mistakes there stay hidden. Check them now and then with `--skipLibCheck false`.

### Why should the version of @types/node match your Node.js?

`@types/node` describes one version of Node.js. A newer one lets `tsc` accept functions your Node.js does not have, and the program crashes with "is not a function".

### Why can import { sign } from "jsonwebtoken" pass tsc but fail in Node.js?

The types say what the CommonJS package exports. Node.js offers named imports from CommonJS only for names it can find by reading the code, and for `jsonwebtoken` it cannot. The default import always works.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [24. Modules in TypeScript](24-modules.md) | [Course outline](README.md#course-outline) | [26. Runtime Validation with Joi](26-runtime-validation-with-joi.md) |
