# tsconfig Deep Dive

> **Session 27 of 33** | Part 6: Real Programs | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [How tsc Finds Its Settings](#how-tsc-finds-its-settings)
- [Seeing the Final Settings with --showConfig](#seeing-the-final-settings-with---showconfig)
- [extends, include and exclude](#extends-include-and-exclude)
- [strict and Its Family](#strict-and-its-family)
- [Extra Checks Beyond strict](#extra-checks-beyond-strict)
- [target: Which JavaScript a Build Writes](#target-which-javascript-a-build-writes)
- [lib: Which Built-in Types Exist](#lib-which-built-in-types-exist)
- [Output: rootDir, outDir and noEmitOnError](#output-rootdir-outdir-and-noemitonerror)
- [What Changed in TypeScript 6 and 7](#what-changed-in-typescript-6-and-7)
- [Mini Example 1: Updating an Old tsconfig](#mini-example-1-updating-an-old-tsconfig)
- [Mini Example 2: A tsconfig for a Node.js API](#mini-example-2-a-tsconfig-for-a-nodejs-api)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `27-tsconfig` inside `ts-practice`, with the one-line `tsconfig.json`, and a first file

27-tsconfig/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

app.ts

```typescript
export const session = 27;
console.log(`Session ${session}: tsconfig`);
```

The two Mini Examples get folders of their own, `27-old-project` and `27-api`, each with a complete `tsconfig.json` instead of the one-line file.

Session 02 created your `tsconfig.json` with `tsc --init`, made four changes, and promised to explain the rest later. Most settings have since been explained in the session where they mattered. This session puts it all together: how `tsc` finds and combines its settings, the `strict` family, the extra checks, `target` and `lib`, the output settings, and what TypeScript 6 and 7 changed, which is why tutorials from before 2026 show a very different file.

---

## How tsc Finds Its Settings

`npx tsc` looks for a `tsconfig.json` in the current folder. If there is none, it looks in the parent folder, then in that folder's parent, and so on. Two consequences:

1. Each session folder works because it has its own one-line `tsconfig.json`: `tsc` stops there.
2. A new folder **without** its own `tsconfig.json` silently uses `ts-practice/tsconfig.json`, so `npx tsc --noEmit` checks every file of every session, including all the mistakes you made on purpose. If you ever see errors from files you did not touch, check that the folder has its `tsconfig.json`.

`-p` (or `--project`) points `tsc` at another folder or config file: `npx tsc -p 27-api` from `ts-practice` checks that project.

Settings on the command line win over the file: `npx tsc --noEmit` is your settings plus `noEmit`. But naming a file is different. Try to build only `app.ts`

```bash
npx tsc app.ts
```

```text
error TS5112: tsconfig.json is present but will not be loaded if files are specified on commandline. Use '--ignoreConfig' to skip this error.
```

A file name on the command line means "ignore `tsconfig.json`", and since TypeScript 6 that is an error unless you say so with `--ignoreConfig`. You will use that in the `target` section.

![tsc looks for tsconfig.json in the current folder and then in each parent folder; a session folder with its own file stops the search there, while a folder without one ends up using the ts-practice file and checks every session](images/27-tsconfig-deep-dive/config-search.gif)

---

## Seeing the Final Settings with --showConfig

Your session file says only `"extends": "../tsconfig.json"`. What does `tsc` really use?

```bash
npx tsc --showConfig
```

```text
{
    "compilerOptions": {
        "declaration": true,
        "declarationMap": true,
        "erasableSyntaxOnly": true,
        "exactOptionalPropertyTypes": true,
        "isolatedModules": true,
        "module": "nodenext",
        "moduleDetection": "force",
        "noUncheckedIndexedAccess": true,
        "noUncheckedSideEffectImports": true,
        "outDir": "../dist",
        "rewriteRelativeImportExtensions": true,
        "skipLibCheck": true,
        "strict": true,
        "sourceMap": true,
        "target": "esnext",
        "types": [
            "node"
        ],
        "verbatimModuleSyntax": true,
        "moduleResolution": "nodenext",
        "preserveConstEnums": true,
        "allowImportingTsExtensions": true
    },
    "files": [
        "./app.ts"
    ],
    "exclude": [
        "D:/ts-practice/dist"
    ]
}
```

`--showConfig` prints the final settings and builds nothing. Four things to notice

| What you see                         | Why                                                          |
| ------------------------------------ | ------------------------------------------------------------ |
| Every setting from `ts-practice/tsconfig.json` | `extends` copied them in                           |
| `"outDir": "../dist"`                | The base file says `./dist`. A path in a config file counts from **that file's** folder, so it still means `ts-practice/dist` |
| `moduleResolution`, `preserveConstEnums`, `allowImportingTsExtensions` | You never wrote them. Other settings switch them on: `module: "nodenext"` brings `moduleResolution: "nodenext"`, and `rewriteRelativeImportExtensions` brings `allowImportingTsExtensions` |
| `files` and `exclude`                | The files `tsc` found (only `app.ts` so far), and the output folder, which is never checked |

Use `--showConfig` whenever a setting seems to have no effect: it shows what `tsc` actually sees.

---

## extends, include and exclude

| Key                  | Meaning                                                    | When you leave it out                                  |
| -------------------- | ---------------------------------------------------------- | ------------------------------------------------------ |
| `"extends"`          | Start from another config file, then change what you need  | Nothing is inherited                                   |
| `"include"`          | Which files to check, as patterns: `["src"]`, `["src/**/*.ts"]` | Every `.ts` file in the folder and its subfolders |
| `"exclude"`          | Which of those to skip again                               | `node_modules` and the `outDir` are skipped            |
| `"files"`            | An exact list of files, rarely used                        | -                                                      |

How `extends` combines files: the settings inside `compilerOptions` are merged one by one (yours win), but `include`, `exclude` and `files` are **replaced** completely. A test: a parent file with `"include": ["app.ts"]`, and a child file in `sub/` with `"include": ["*.ts"]`. `npx tsc --showConfig -p sub` lists only `./b.ts`, the child's file, not `app.ts`. Chains work too: here the base `outDir` became `"../../dist"`, still `ts-practice/dist`.

![The base tsconfig and the session tsconfig are combined into the final settings: compilerOptions are merged option by option, include and exclude are replaced, and relative paths keep pointing where they pointed in their own file](images/27-tsconfig-deep-dive/show-config.gif)

---

## strict and Its Family

`"strict": true` is not one check but a group. In TypeScript 7 it turns on eight options

| Option                           | What it does                                                    | Seen in    |
| -------------------------------- | --------------------------------------------------------------- | ---------- |
| `noImplicitAny`                  | No `any` that nobody wrote, like an untyped parameter (TS7006)  | 03, 04     |
| `strictNullChecks`               | `null` and `undefined` are their own types (TS18048)            | 11         |
| `strictFunctionTypes`            | Function parameters are checked strictly when functions are passed around | - |
| `strictBindCallApply`            | `bind`, `call` and `apply` check their arguments                | -          |
| `strictPropertyInitialization`   | Class fields must get a value (TS2564)                          | 20         |
| `noImplicitThis`                 | `this` must have a known type                                   | 20         |
| `useUnknownInCatchVariables`     | `err` in `catch` is `unknown`, not `any` (TS18046)              | 04, 23     |
| `strictBuiltinIteratorReturn`    | Built-in iterators have exact return types                      | -          |

You can switch one of them off while keeping the rest. Try it with `catch`

catch.ts

```typescript
try {
  JSON.parse("{");
} catch (err) {
  console.log(err.mesage);
}
```

```text
catch.ts:4:15 - error TS18046: 'err' is of type 'unknown'.

4   console.log(err.mesage);
                ~~~


Found 1 error in catch.ts:4
```

Now switch that one member off

27-tsconfig/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "useUnknownInCatchVariables": false
  }
}
```

`tsc` is quiet, and `node catch.ts` prints `undefined`: `err` is `any` again, and the typo `mesage` is back to being your problem. Every member you switch off brings back the kind of bug it was made for. Remove the option again.

Two more facts: `exactOptionalPropertyTypes` needs `strictNullChecks` (Session 11, TS5052), and `"alwaysStrict": false` no longer exists in TypeScript 7 (it gives TS5108), so the output is always in strict mode.

![strict unfolds into eight options, each with the session where it caught something: untyped parameters, null and undefined, class fields, this, and err in catch](images/27-tsconfig-deep-dive/strict-family.gif)

---

## Extra Checks Beyond strict

`tsc --init` wrote six more checks as comments, under "Style Options". They catch real bugs too. This file has five of them

style.ts

```typescript
import { readFile } from "node:fs/promises";

function discountPercent(code: string) {
  if (code === "WELCOME10") {
    return 10;
  }
}

function statusLabel(status: string, verbose: boolean): string {
  switch (status) {
    case "paid":
      console.log("payment received");
    case "shipped":
      return "on the way";
    default:
      return "waiting";
  }
}

const limits: Record<string, number> = { orders: 10 };
console.log(limits.orders, discountPercent("WELCOME10"), statusLabel("paid", false));
```

With the course settings, `tsc` finds nothing, and

```text
payment received
10 10 on the way
```

A paid order is labelled "on the way": the `case "paid"` has no `return`, so it falls through into the next case. Switch on the five checks

27-tsconfig/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true
  }
}
```

```text
style.ts:1:10 - error TS6133: 'readFile' is declared but its value is never read.

1 import { readFile } from "node:fs/promises";
           ~~~~~~~~

style.ts:3:10 - error TS7030: Not all code paths return a value.

3 function discountPercent(code: string) {
           ~~~~~~~~~~~~~~~

style.ts:9:38 - error TS6133: 'verbose' is declared but its value is never read.

9 function statusLabel(status: string, verbose: boolean): string {
                                       ~~~~~~~

style.ts:11:5 - error TS7029: Fallthrough case in switch.

11     case "paid":
       ~~~~~~~~~~~~

style.ts:21:20 - error TS4111: Property 'orders' comes from an index signature, so it must be accessed with ['orders'].

21 console.log(limits.orders, discountPercent("WELCOME10"), statusLabel("paid", false));
                      ~~~~~~


Found 5 errors in the same file, starting at: style.ts:1
```

| Option                                | Catches                                                     | Advice                       |
| ------------------------------------- | ----------------------------------------------------------- | ---------------------------- |
| `noImplicitReturns`                   | A function that returns a value on some paths and nothing on others (`discountPercent` returns `10 \| undefined`) | Turn it on |
| `noFallthroughCasesInSwitch`          | A `case` without `return` or `break` that runs into the next one | Turn it on              |
| `noImplicitOverride`                  | A method that replaces a parent's method without `override` (Session 21) | Turn it on       |
| `noUnusedLocals`, `noUnusedParameters` | Imports, variables and parameters nobody uses              | Useful, but noisy while you are still writing; many teams leave this to a linter (Session 29) |
| `noPropertyAccessFromIndexSignature`  | `limits.orders` on an index signature: write `limits["orders"]`, so it looks like a key that might be missing | A matter of taste. It also flags `process.env.PORT` (TS4111), so it is often left off in backends |

Remove the five options again; the course keeps `ts-practice` as it is, and Mini Example 2 uses the three recommended ones.

![Five extra checks each catch one problem in style.ts: an unused import, a function that sometimes returns nothing, an unused parameter, a switch case that falls through, and dot access on an index signature](images/27-tsconfig-deep-dive/extra-checks.gif)

---

## target: Which JavaScript a Build Writes

`target` decides which version of JavaScript `tsc` writes when it **builds**. Newer syntax is rewritten into older syntax for older runtimes. A class with a `#private` field and `??=`

cart.ts

```typescript
export class Cart {
  #items: string[] = [];
  coupon?: string;

  add(item: string): number {
    this.coupon ??= "WELCOME10";
    this.#items.push(item);
    return this.#items.length;
  }
}
```

Build just this file for ES2019, ignoring `tsconfig.json`

```bash
npx tsc cart.ts --ignoreConfig --target es2019 --outDir out
```

out/cart.js

```javascript
var __classPrivateFieldGet = (this && this.__classPrivateFieldGet) || function (receiver, state, kind, f) {
    if (kind === "a" && !f) throw new TypeError("Private accessor was defined without a getter");
    if (typeof state === "function" ? receiver !== state || !f : !state.has(receiver)) throw new TypeError("Cannot read private member from an object whose class did not declare it");
    return kind === "m" ? f : kind === "a" ? f.call(receiver) : f ? f.value : state.get(receiver);
};
var _Cart_items;
export class Cart {
    constructor() {
        _Cart_items.set(this, []);
    }
    add(item) {
        var _a;
        (_a = this.coupon) !== null && _a !== void 0 ? _a : (this.coupon = "WELCOME10");
        __classPrivateFieldGet(this, _Cart_items, "f").push(item);
        return __classPrivateFieldGet(this, _Cart_items, "f").length;
    }
}
_Cart_items = new WeakMap();
```

ES2019 has no `#private` fields and no `??=`, so `tsc` rebuilt them with a `WeakMap` and a helper function. Now without `--target`, which in TypeScript 7 means the newest JavaScript

```bash
npx tsc cart.ts --ignoreConfig --outDir out
```

```javascript
export class Cart {
    #items = [];
    coupon;
    add(item) {
        this.coupon ??= "WELCOME10";
        this.#items.push(item);
        return this.#items.length;
    }
}
```

Only the types are gone. For this course `target: "esnext"` is right: Node.js 24 understands all of it, and when Node.js runs your `.ts` files directly, `target` plays no part at all (Node.js only removes the types). The one exception you met was decorators in Session 22, which needed `es2022` because Node.js cannot run `@` yet. Delete the `out` folder.

![The same Cart class built twice: with target es2019 the private field and ??= are rewritten into a WeakMap and a helper function; with esnext only the types are removed](images/27-tsconfig-deep-dive/target-output.gif)

---

## lib: Which Built-in Types Exist

`lib` decides which built-in names TypeScript knows. Without a `lib` setting, you get the JavaScript of your `target` **plus the browser** (the "DOM": `document`, `window`, `localStorage` and so on). On a server that is a trap

who.ts

```typescript
console.log(`Hello, ${name}`);
console.log(document.title);
```

`tsc` finds nothing wrong. In a browser, `name` is the window's name. In Node.js

```text
file:///D:/ts-practice/27-tsconfig/who.ts:1
console.log(`Hello, ${name}`);
                      ^

ReferenceError: name is not defined
    at file:///D:/ts-practice/27-tsconfig/who.ts:1:23
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

You probably meant a variable `name` that you forgot to declare, and the browser types hid the mistake. The `tsc --init` file even has a comment for it: "For nodejs: `"lib": ["esnext"]`". Try it

27-tsconfig/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "lib": ["esnext"]
  }
}
```

```text
who.ts:1:23 - error TS2304: Cannot find name 'name'.

1 console.log(`Hello, ${name}`);
                        ~~~~

who.ts:2:13 - error TS2584: Cannot find name 'document'. Do you need to change your target library? Try changing the 'lib' compiler option to include 'dom'.

2 console.log(document.title);
              ~~~~~~~~


Found 2 errors in the same file, starting at: who.ts:1
```

Both caught. What about `fetch` and `Response` (Session 23)? Node.js has them too, and `@types/node` describes them, so they still work

fetchy.ts

```typescript
const res = new Response(JSON.stringify({ ok: true }));
const data: unknown = await res.json();
console.log(data, new URL("http://localhost:4000/products/1").pathname);
```

```text
{ ok: true } /products/1
```

Delete `who.ts`. For a backend, `"lib": ["esnext"]` is the better choice; Mini Example 2 uses it. The `ts-practice` file keeps the default so that all earlier sessions stay exactly as you saw them.

![Without a lib setting, TypeScript also loads the browser's types, so name and document pass tsc and fail in Node.js; with lib esnext only JavaScript itself plus @types/node are known, and both names are caught](images/27-tsconfig-deep-dive/lib-dom.gif)

---

## Output: rootDir, outDir and noEmitOnError

These settings only matter when you **build** with `tsc` (Session 02, Mini Example 2)

| Setting           | Meaning                                                              |
| ----------------- | -------------------------------------------------------------------- |
| `outDir`          | Where the built files go                                             |
| `rootDir`         | The folder of your source files; its layout is copied into `outDir`  |
| `sourceMap`       | Also write `.js.map` files, so errors and debuggers point to `.ts` lines (Session 29) |
| `declaration`     | Also write `.d.ts` files (Session 25). Needed for libraries, not for apps |
| `noEmit`          | Check only, write nothing                                            |
| `noEmitOnError`   | Write nothing if there is any error                                  |

Since TypeScript 6, `rootDir` must be set when your code lives in a subfolder like `src`. A small project in a folder `layout`, with its code in `layout/src/app.ts`

layout/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist"
  }
}
```

```bash
npx tsc -p layout
```

```text
layout/tsconfig.json:4:5 - error TS5011: The common source directory of 'tsconfig.json' is './src'. The 'rootDir' setting must be explicitly set to this or another path to adjust your output's file layout.
  Visit https://aka.ms/ts6 for migration information.

4     "outDir": "./dist"
      ~~~~~~~~


Found 1 error in layout/tsconfig.json:4
```

TypeScript 5 guessed `rootDir` from where the files are, and wrote `dist/app.js`. TypeScript 6 and 7 use the config file's own folder, which would give `dist/src/app.js`, so they stop and ask you to decide. Add the line

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

`npx tsc -p layout --listEmittedFiles` lists what was written

```text
TSFILE: D:/ts-practice/27-tsconfig/layout/dist/app.js.map
TSFILE: D:/ts-practice/27-tsconfig/layout/dist/app.js
TSFILE: D:/ts-practice/27-tsconfig/layout/dist/app.d.ts.map
TSFILE: D:/ts-practice/27-tsconfig/layout/dist/app.d.ts
```

`dist/app.js`, as intended. (`--noEmit` checks do not report TS5011, because they write no files.)

The last one, `noEmitOnError`: remember that `tsc` writes the `.js` files even when there are errors (Session 02). In a real project that can put broken code into `dist`. With `noEmitOnError`, a build with errors writes nothing; Mini Example 2 shows it. Delete the `layout` folder.

![The same src/app.ts built two ways: TypeScript 5 guessed rootDir and wrote dist/app.js, TypeScript 6 and 7 stop with TS5011 until rootDir says src, and then write dist/app.js](images/27-tsconfig-deep-dive/rootdir-layout.gif)

---

## What Changed in TypeScript 6 and 7

Before TypeScript 5.9, `tsc --init` wrote a file of 113 lines: about a hundred options as comments, and six active ones, `"target": "es2016"`, `"module": "commonjs"`, `"esModuleInterop": true`, `"forceConsistentCasingInFileNames": true`, `"strict": true` and `"skipLibCheck": true`. Most tutorials and older projects still show that file. TypeScript 6.0 then changed what an empty config means, and TypeScript 7.0 (the native compiler from Session 01) removed old options.

What an empty `tsconfig.json` (`{}`) means, tested with each version

| Setting      | TypeScript 5.8                        | TypeScript 6.0 and 7.0                      |
| ------------ | ------------------------------------- | ------------------------------------------- |
| `strict`     | Off: `function f(n)` passes           | On: TS7006                                  |
| `target`     | ES5: classes become functions, `#private` is refused | The newest JavaScript            |
| `module`     | CommonJS (`exports.Box = ...`)        | ES modules (`export class Box`)             |
| `types`      | Every `@types` package in `node_modules` | None: `process` is unknown without `"types": ["node"]` |
| `rootDir`    | Guessed from the files                | The config's folder; must be set for `src` layouts (TS5011) |

![An empty tsconfig.json meant no strict checks, ES5, CommonJS and every @types package in TypeScript 5.8; in TypeScript 6.0 and 7.0 it means strict checks, the newest JavaScript, ES modules and no automatic types](images/27-tsconfig-deep-dive/empty-config.gif)

Options that TypeScript 6.0 deprecated and TypeScript 7.0 removed

| Option                                     | TypeScript 6.0 | TypeScript 7.0 | Use instead                          |
| ------------------------------------------ | -------------- | -------------- | ------------------------------------ |
| `"target": "es5"`                          | TS5107         | TS5108         | `esnext` (or a newer year)           |
| `"moduleResolution": "node"` (`node10`), `"classic"` | TS5107 | TS5108   | `nodenext` (comes with `module`)     |
| `"module": "amd"`, `"umd"`, `"system"`     | TS5107         | TS5108         | `nodenext`, or `preserve` for bundlers |
| `"baseUrl"`                                | TS5101         | TS5102         | Relative imports, or `paths`         |
| `"outFile"`                                | TS5101         | TS5102         | A bundler                            |
| `"esModuleInterop": false`, `"allowSyntheticDefaultImports": false` | TS5107 | TS5108 | Leave them out (always on now) |
| `"downlevelIteration"`                     | TS5101         | TS5102         | Leave it out                         |
| `"alwaysStrict": false`                    | TS5107         | TS5108         | Leave it out                         |

In TypeScript 6.0 these were errors you could silence for a while, for example

```text
tsconfig.json:4:15 - error TS5107: Option 'target=ES5' is deprecated and will stop functioning in TypeScript 7.0. Specify compilerOption '"ignoreDeprecations": "6.0"' to silence this error.
```

In TypeScript 7.0 the option is simply gone, and `"ignoreDeprecations": "6.0"` no longer helps

```text
tsconfig.json:3:23 - error TS5102: Option 'baseUrl' has been removed. Please remove it from your configuration.
  Use '"paths": {"*": ["./*"]}' instead.
```

A word on `paths`, which old projects often combine with `baseUrl` for imports like `"@/util.ts"`: it still exists, but only `tsc` reads it. Node.js does not read `tsconfig.json` at all, so the same import fails with `ERR_MODULE_NOT_FOUND`, and `tsc` itself warns (TS2877) that such a path will not be rewritten in a build. This course uses relative imports.

![A timeline of five old options: accepted by TypeScript 5.8, deprecated by 6.0 with an ignoreDeprecations escape hatch, and removed by 7.0 with TS5108 or TS5102](images/27-tsconfig-deep-dive/removed-options.gif)

---

## Mini Example 1: Updating an Old tsconfig

You join a project that was set up from a 2023 tutorial. Make a folder `27-old-project` in `ts-practice` with this file (no `extends`), and two source files

27-old-project/tsconfig.json

```json
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "moduleResolution": "node",
    "baseUrl": ".",
    "rootDir": "./src",
    "outDir": "./dist",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

src/money.ts

```typescript
export function formatRupees(paise: number): string {
  return `Rs ${(paise / 100).toFixed(2)}`;
}
```

src/app.ts

```typescript
import { formatRupees } from "./money";

console.log(formatRupees(149900));
console.log(`running on Node.js ${process.version}`);
```

**Step 1: removed options.** Run `npx tsc` inside `27-old-project`

```text
tsconfig.json:5:25 - error TS5108: Option 'moduleResolution=node10' has been removed. Please remove it from your configuration.

5     "moduleResolution": "node",
                          ~~~~~~

tsconfig.json:6:5 - error TS5102: Option 'baseUrl' has been removed. Please remove it from your configuration.
  Use '"paths": {"*": ["./*"]}' instead.

6     "baseUrl": ".",
      ~~~~~~~~~


Found 2 errors in the same file, starting at: tsconfig.json:5
```

The project never used `baseUrl` for anything, so delete both lines.

**Step 2: types are no longer loaded by themselves.**

```text
src/app.ts:4:35 - error TS2591: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node` and then add 'node' to the types field in your tsconfig.

4 console.log(`running on Node.js ${process.version}`);
                                    ~~~~~~~


Found 1 error in src/app.ts:4
```

In TypeScript 5 this project got `@types/node` automatically. Add `"types": ["node"]` after `"skipLibCheck": true`.

**Step 3: it builds, but does it run?** `npx tsc` passes now

```bash
node dist/app.js
```

```text
file:///D:/ts-practice/27-old-project/dist/app.js:2
Object.defineProperty(exports, "__esModule", { value: true });
                      ^

ReferenceError: exports is not defined in ES module scope
This file is being treated as an ES module because it has a '.js' file extension and '\\?\D:\ts-practice\package.json' contains "type": "module". To treat it as a CommonJS script, rename it to use the '.cjs' file extension.
    at file:///D:/ts-practice/27-old-project/dist/app.js:2:23
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`"module": "commonjs"` wrote CommonJS (`exports`), but the package says `"type": "module"`, so Node.js reads the files as ES modules. `tsc` did not warn, because `commonjs` ignores what `package.json` says. Change it to `"module": "nodenext"`, which follows `package.json` (Session 24).

**Step 4: ES module rules.**

```text
src/app.ts:1:30 - error TS2835: Relative import paths need explicit file extensions in ECMAScript imports when '--moduleResolution' is 'node16' or 'nodenext'. Did you mean './money.js'?

1 import { formatRupees } from "./money";
                               ~~~~~~~~~


Found 1 error in src/app.ts:1
```

As in Session 24, write `"./money.ts"` (not the suggested `.js`)

```text
src/app.ts:1:30 - error TS5097: An import path can only end with a '.ts' extension when 'allowImportingTsExtensions' is enabled.

1 import { formatRupees } from "./money.ts";
                               ~~~~~~~~~~~~


Found 1 error in src/app.ts:1
```

`rewriteRelativeImportExtensions` allows `.ts` imports and rewrites them to `.js` in the build (Session 02). Add it, and change `target` to `esnext` while you are there

27-old-project/tsconfig.json

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "rootDir": "./src",
    "outDir": "./dist",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "types": ["node"],
    "rewriteRelativeImportExtensions": true
  }
}
```

```bash
npx tsc
node dist/app.js
```

```text
Rs 1499.00
running on Node.js v24.19.0
```

And `node src/app.ts` prints the same, because the source now follows the rules for running `.ts` files directly. Five errors, each with a clear message, and the project is on TypeScript 7. (`esModuleInterop` and `forceConsistentCasingInFileNames` can stay: they are on by default now, so the lines are harmless.)

![The old tsconfig goes through five fixes: removed options deleted, types node added, the build crashes on exports because of commonjs, module nodenext asks for file extensions, and rewriteRelativeImportExtensions makes the build and node src/app.ts both work](images/27-tsconfig-deep-dive/old-project.gif)

---

## Mini Example 2: A tsconfig for a Node.js API

Everything from this session in one file, for a real backend project. Make a folder `27-api` in `ts-practice`. This is the file Sessions 30 to 33 start from

27-api/tsconfig.json

```json
{
  "compilerOptions": {
    // Files: src/ in, dist/ out
    "rootDir": "./src",
    "outDir": "./dist",

    // Node.js 24 with ES modules, and no browser types
    "module": "nodenext",
    "target": "esnext",
    "lib": ["esnext"],
    "types": ["node"],

    // Run .ts files with node, build them with tsc
    "erasableSyntaxOnly": true,
    "rewriteRelativeImportExtensions": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noUncheckedSideEffectImports": true,

    // Checks
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,

    // Output
    "sourceMap": true,
    "noEmitOnError": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

| Group     | What changed compared with `ts-practice/tsconfig.json`              |
| --------- | ------------------------------------------------------------------- |
| Files     | `rootDir` and `outDir` for the `src` and `dist` layout, and `include: ["src"]`, so only the app is checked and built |
| Node.js   | `lib: ["esnext"]`: no browser types                                  |
| Checks    | The three recommended extra checks                                   |
| Output    | `noEmitOnError`, and no `declaration` files: an app is not a library |

Two source files

src/prices.ts

```typescript
const PRICES: Record<string, number> = { keyboard: 149900, mouse: 49900 };

export function priceOf(item: string): number | undefined {
  return PRICES[item];
}
```

src/app.ts

```typescript
import { priceOf } from "./prices.ts";

for (const item of ["keyboard", "webcam"]) {
  const price = priceOf(item);
  if (price === undefined) {
    console.log(`${item}: not sold here`);
  } else {
    console.log(`${item}: Rs ${(price / 100).toFixed(2)}`);
  }
}
```

Build it, inside `27-api`

```bash
npx tsc --listEmittedFiles
```

```text
TSFILE: D:/ts-practice/27-api/dist/prices.js.map
TSFILE: D:/ts-practice/27-api/dist/prices.js
TSFILE: D:/ts-practice/27-api/dist/app.js.map
TSFILE: D:/ts-practice/27-api/dist/app.js
```

```bash
node dist/app.js
```

```text
keyboard: Rs 1499.00
webcam: not sold here
```

`node src/app.ts` prints the same: one setup for running in development and for building for production. Now make a mistake at the end of `src/app.ts`, delete `dist`, and build again

```typescript
console.log(`served by ${name}`);
```

```text
src/app.ts:11:26 - error TS2304: Cannot find name 'name'.

11 console.log(`served by ${name}`);
                            ~~~~


Found 1 error in src/app.ts:11
```

Two settings worked together: `lib: ["esnext"]` caught the browser global, and `noEmitOnError` kept the broken code out: there is no `dist` folder at all. Remove the line. A file `notes/scratch.ts` next to `src` would not be checked or built either, because of `include: ["src"]`.

`npx tsc --showConfig` confirms the result: the same list of options, `"files"` with only `./src/app.ts` and `./src/prices.ts`, and `dist` excluded.

![The API tsconfig in five groups, files, Node.js, running and building, checks and output, with the problem each group prevents; a build with an error writes no dist folder](images/27-tsconfig-deep-dive/api-tsconfig.gif)

---

## Beginner Mistakes

### Mistake 1: Running tsc in a folder without its own tsconfig.json

`tsc` then walks up and uses the parent folder's file, and checks every file below it. In `ts-practice` that means the errors of every session. Give each project folder its own `tsconfig.json`.

---

### Mistake 2: Naming a file while a tsconfig.json is there

`npx tsc app.ts` gives `error TS5112`. Run `npx tsc` for the project, `-p` for another project, or add `--ignoreConfig` if you really want to ignore the file.

---

### Mistake 3: Copying a tsconfig from an old tutorial

`"moduleResolution": "node"`, `"baseUrl"`, `"target": "es5"` and similar give TS5108 or TS5102 in TypeScript 7. Start from `tsc --init`, or from the file in Mini Example 2.

---

### Mistake 4: Switching off strict, or a part of it, to make errors go away

Every member of `strict` catches a kind of bug; with `useUnknownInCatchVariables: false`, `err.mesage` passes again. Fix the code instead.

---

### Mistake 5: Keeping the browser types in a backend

Without `lib`, TypeScript also loads the DOM, so `name` and `document` pass `tsc` and fail in Node.js. Use `"lib": ["esnext"]`.

---

### Mistake 6: Expecting paths to work in Node.js

`paths` is only read by `tsc`. Node.js does not read `tsconfig.json`, and `"@/util.ts"` gives `ERR_MODULE_NOT_FOUND`. Use relative imports.

---

### Mistake 7: A src folder without rootDir

Since TypeScript 6, a build with the code in `src` and no `rootDir` gives `error TS5011`. Add `"rootDir": "./src"`.

---

### Mistake 8: module commonjs in a "type": "module" project

`tsc` writes `exports`, and Node.js fails with `exports is not defined in ES module scope`. Use `"module": "nodenext"`, which follows `package.json`.

---

### Mistake 9: Building with errors

`tsc` writes `.js` files even when there are errors, so broken code can end up in `dist`. Add `"noEmitOnError": true` to projects you build.

---

### Mistake 10: Choosing an old target without a reason

An older `target` rewrites modern code into longer code with helpers, like the `WeakMap` version of `#items`. For Node.js 24, use `esnext`.

---

## Practice Exercises

### Exercise 1

In `27-tsconfig`, run `npx tsc --showConfig`, then add `"noImplicitReturns": true` to the session file and run it again. Which lines changed? Then make a folder `27-tsconfig/sub` with a `tsconfig.json` that extends `../tsconfig.json` and sets `outDir` to `./build`, and check with `--showConfig -p sub` where the output would go.

### Exercise 2

Write a function with a `switch` over `"card" | "upi" | "cod"` that forgets a `return` in one case. Find the bug once with `noFallthroughCasesInSwitch` and once with `noImplicitReturns`.

### Exercise 3

Build `cart.ts` with `--target es2015`, `es2020` and `es2022` (using `--ignoreConfig`). In which version does `??=` stop being rewritten? And `#items`?

### Exercise 4

Write a file that uses `localStorage.setItem("cart", "[]")`. Check it with the default `lib` and with `"lib": ["esnext"]`, then run it with Node.js. Explain all three results.

### Exercise 5

Take an old tsconfig from any tutorial or project you find (or write one with `"target": "es5"`, `"module": "commonjs"`, `"moduleResolution": "node"` and `"outFile"`), and make it work with TypeScript 7, one error at a time.

### Exercise 6

In `27-api`, add a folder `src/routes` with one file, and a folder `tests` outside `src` with a file that has a type error. Check that `npx tsc` ignores `tests`. Then write a second config `tsconfig.test.json` that extends `tsconfig.json`, sets `"rootDir": "."` and `"noEmit": true`, and includes both folders, and check with `npx tsc -p tsconfig.test.json`.

### Exercise 7

In `27-api`, switch on `noUnusedLocals` and `noUnusedParameters` and write a few unused variables and parameters. Find out how to mark a parameter as unused on purpose (hint: start its name with `_`).

### Exercise 8

Make a table of every option in `ts-practice/tsconfig.json`, with one sentence each in your own words and the session where it mattered. Compare it with the table in Session 02.

---

## Interview Questions

### How does tsc find its tsconfig.json?

It looks in the current folder, then in each parent folder, and uses the first one it finds. `-p` points it at another project. Naming files on the command line ignores the config, which since TypeScript 6 needs `--ignoreConfig` (otherwise TS5112).

### What does --showConfig do?

It prints the final settings after `extends` and all implied options, plus the list of files, without building anything. It is the first thing to try when a setting seems to have no effect.

### How does extends combine two config files?

Options in `compilerOptions` are merged one by one, and the extending file wins. `include`, `exclude` and `files` are replaced, not merged. Relative paths count from the file they are written in.

### What does strict turn on?

A group of checks: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables` and `strictBuiltinIteratorReturn`. Since TypeScript 6 it is on even without a config.

### Which checks beyond strict would you turn on?

`noImplicitReturns`, `noFallthroughCasesInSwitch` and `noImplicitOverride` catch real bugs with few false alarms. `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` are already in the file `tsc --init` writes. Unused-code checks are often left to a linter.

### What is the difference between target and lib?

`target` decides which JavaScript syntax a build writes, rewriting newer syntax for older runtimes. `lib` decides which built-in types exist, such as `Promise` or the browser's `document`. Without `lib`, it follows the target and adds the DOM.

### Why use lib: ["esnext"] for a Node.js project?

The default includes browser types, so names like `name`, `document` or `localStorage` pass `tsc` and fail in Node.js. With `esnext` they are errors, while Node.js's own globals still come from `@types/node`.

### What is rootDir, and why does TypeScript 6 ask for it?

The folder whose layout is copied into `outDir`. TypeScript 5 guessed it from the files; TypeScript 6 and 7 use the config's folder and report TS5011 when the code is in a subfolder like `src`, so the output layout never changes by surprise.

### What does noEmitOnError do?

`tsc` normally writes JavaScript even when there are type errors. With `noEmitOnError`, any error means nothing is written, so a broken build cannot replace working files.

### What changed in TypeScript 6 and 7?

TypeScript 6 changed the defaults (strict on, newest target, ES modules, no automatic `@types`, explicit `rootDir`) and deprecated old options like `target: es5`, `moduleResolution: node`, `baseUrl` and `outFile`. TypeScript 7, the native compiler, removed them: they now give TS5108 or TS5102.

### Does Node.js read tsconfig.json?

No. Node.js only removes the types and runs the file. Settings like `paths` work in `tsc` but not in Node.js, which is why this course uses relative imports and rules (like `erasableSyntaxOnly`) that keep the code runnable by Node.js.

### Why does a CommonJS build fail in a "type": "module" project?

`module: commonjs` writes `exports` and `require`, but `"type": "module"` makes Node.js treat `.js` files as ES modules, where `exports` does not exist. `module: nodenext` reads `package.json` and writes the right kind.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [26. Runtime Validation with Joi](26-runtime-validation-with-joi.md) | [Course outline](README.md#course-outline) | [28. Migrating JavaScript to TypeScript](28-migrating-javascript-to-typescript.md) |
