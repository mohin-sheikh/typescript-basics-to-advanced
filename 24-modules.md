# Modules in TypeScript

> **Session 24 of 33** | Part 6: Real Programs | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Every File Is a Module](#every-file-is-a-module)
- [Import Paths Need the .ts Extension](#import-paths-need-the-ts-extension)
- [import type, One More Time](#import-type-one-more-time)
- [Named Exports Instead of Default Exports](#named-exports-instead-of-default-exports)
- [Barrel Files](#barrel-files)
- [Circular Imports](#circular-imports)
- [Node.js Built-ins and import.meta](#nodejs-built-ins-and-importmeta)
- [Importing JSON](#importing-json)
- [Loading a Module Later with import()](#loading-a-module-later-with-import)
- [Why This Course Writes ES Modules](#why-this-course-writes-es-modules)
- [Mini Example 1: A Shop in Feature Folders](#mini-example-1-a-shop-in-feature-folders)
- [Mini Example 2: Payment Providers Loaded on Demand](#mini-example-2-payment-providers-loaded-on-demand)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `24-modules` inside `ts-practice`, with the one-line `tsconfig.json`. This session uses subfolders inside it; `npx tsc --noEmit` checks them all

24-modules/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

You know modules from the JavaScript course ([Session 21](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/21-modules.md)): `import` and `export`, CommonJS and ES modules. This session is about what TypeScript adds, and the rules of this course's setup: ES modules (`"type": "module"`, Session 02) with `module: "nodenext"`, run directly by Node.js.

---

## Every File Is a Module

Two files that both have a top-level `ratePercent`

shipping.ts

```typescript
const ratePercent = 5;
console.log(`shipping: ${ratePercent}%`);
```

discounts.ts

```typescript
const ratePercent = 10;
console.log(`discount: ${ratePercent}%`);
```

```bash
npx tsc --noEmit
node shipping.ts
```

```text
shipping: 5%
```

No conflict, because each file is a **module**, with its own top-level scope. That is what the line `"moduleDetection": "force"` in your tsconfig (Session 02) says: treat every file as a module, even one without `import` or `export`.

TypeScript also knows the older rule, where a file without `import` or `export` is a **script**, and the top-level names of all scripts share one global scope. Try it: add one option to this folder's `tsconfig.json`

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "moduleDetection": "legacy"
  }
}
```

```text
discounts.ts:1:7 - error TS2451: Cannot redeclare block-scoped variable 'ratePercent'.

1 const ratePercent = 10;
        ~~~~~~~~~~~

  shipping.ts:1:7 - 'ratePercent' was also declared here.
    1 const ratePercent = 5;
            ~~~~~~~~~~~

shipping.ts:1:7 - error TS2451: Cannot redeclare block-scoped variable 'ratePercent'.

1 const ratePercent = 5;
        ~~~~~~~~~~~

  discounts.ts:1:7 - 'ratePercent' was also declared here.
    1 const ratePercent = 10;
            ~~~~~~~~~~~


Found 2 errors in 2 files.

Errors  Files
     1  discounts.ts:1
     1  shipping.ts:1
```

As scripts, the two files fight over one global `ratePercent`. Remove the option again. This also explains the `export {};` that `tsc` adds at the end of some built files (Session 01): a file without imports or exports gets it, so that the JavaScript file is a module too, with its own scope.

![Two files, shipping.ts and discounts.ts, each with ratePercent in its own module box; with moduleDetection legacy the walls between the boxes disappear, both names land in one shared global space, and tsc reports TS2451](images/24-modules/file-scope.gif)

---

## Import Paths Need the .ts Extension

In ES modules, Node.js uses import paths exactly as written: it does not try adding `.ts` or `.js`, and it does not look for an `index` file in a folder. So the path must name the file, with its extension (Session 02)

pricing/tax.ts

```typescript
export const GST_PERCENT = 18;

export function addGst(paise: number): number {
  return Math.round(paise * (1 + GST_PERCENT / 100));
}
```

checkout.ts

```typescript
import { addGst, GST_PERCENT } from "./pricing/tax.ts";

console.log(`GST ${GST_PERCENT}%:`, addGst(49900));
```

```text
GST 18%: 58882
```

Leave the extension out

```typescript
import { addGst, GST_PERCENT } from "./pricing/tax";
```

```text
checkout.ts:1:37 - error TS2835: Relative import paths need explicit file extensions in ECMAScript imports when '--moduleResolution' is 'node16' or 'nodenext'. Did you mean './pricing/tax.js'?

1 import { addGst, GST_PERCENT } from "./pricing/tax";
                                      ~~~~~~~~~~~~~~~


Found 1 error in checkout.ts:1
```

```bash
node checkout.ts
```

```text
node:internal/modules/esm/resolve:271
    throw new ERR_MODULE_NOT_FOUND(
          ^

Error [ERR_MODULE_NOT_FOUND]: Cannot find module 'D:\ts-practice\24-modules\pricing\tax' imported from D:\ts-practice\24-modules\checkout.ts
    at finalizeResolution (node:internal/modules/esm/resolve:271:11)
    at moduleResolve (node:internal/modules/esm/resolve:865:10)
    at defaultResolve (node:internal/modules/esm/resolve:992:11)
    at #cachedDefaultResolve (node:internal/modules/esm/loader:701:20)
    at #resolveAndMaybeBlockOnLoaderThread (node:internal/modules/esm/loader:721:38)
    at ModuleLoader.resolveSync (node:internal/modules/esm/loader:759:56)
    at #resolve (node:internal/modules/esm/loader:683:17)
    at ModuleLoader.getOrCreateModuleJob (node:internal/modules/esm/loader:603:35)
    at ModuleJob.syncLink (node:internal/modules/esm/module_job:163:33)
    at ModuleJob.link (node:internal/modules/esm/module_job:253:17) {
  code: 'ERR_MODULE_NOT_FOUND',
  url: 'file:///D:/ts-practice/24-modules/pricing/tax'
}

Node.js v24.19.0
```

Careful with the suggestion "Did you mean './pricing/tax.js'?". That advice is for projects that compile everything to `.js` first and run the `.js` files. Here Node.js runs the `.ts` files, and `./pricing/tax.js` passes `tsc` but fails in Node.js with the same `ERR_MODULE_NOT_FOUND`. In this course, always write `.ts`; when you build with `tsc`, `rewriteRelativeImportExtensions` turns it into `.js` in the output (Session 02).

| Import path                        | `tsc`   | Node.js                       |
| ---------------------------------- | ------- | ----------------------------- |
| `"./pricing/tax.ts"`               | OK      | OK                            |
| `"./pricing/tax.js"`               | OK      | `ERR_MODULE_NOT_FOUND`        |
| `"./pricing/tax"`                  | TS2835  | `ERR_MODULE_NOT_FOUND`        |
| `"./pricing"` (a folder)           | TS2834  | `ERR_UNSUPPORTED_DIR_IMPORT`  |

The last row is a difference from `require` in the JavaScript course: ES modules never pick up a folder's `index` file by themselves. Write `"./pricing/index.ts"`.

![Four import paths go to two checkers: the .ts path passes both tsc and Node.js; the .js path passes tsc but Node.js finds no such file; the path without extension and the folder path are stopped by both](images/24-modules/import-paths.gif)

---

## import type, One More Time

Session 08 introduced `import type`: a type-only import is removed completely before the code runs, so it must contain only types. Put a value in it by mistake

models.ts

```typescript
export interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

export function formatPrice(paise: number): string {
  return `Rs ${(paise / 100).toFixed(2)}`;
}
```

use-models.ts

```typescript
import type { Product, formatPrice } from "./models.ts";

const keyboard: Product = { id: 1, name: "Keyboard", pricePaise: 149900 };
console.log(formatPrice(keyboard.pricePaise));
```

```text
use-models.ts:4:13 - error TS1361: 'formatPrice' cannot be used as a value because it was imported using 'import type'.

4 console.log(formatPrice(keyboard.pricePaise));
              ~~~~~~~~~~~

  use-models.ts:1:24 - 'formatPrice' was imported here.
    1 import type { Product, formatPrice } from "./models.ts";
                             ~~~~~~~~~~~


Found 1 error in use-models.ts:4
```

Mark only the type

```typescript
import { type Product, formatPrice } from "./models.ts";
```

```text
Rs 1499.00
```

| You write                                     | At run time                              |
| --------------------------------------------- | ---------------------------------------- |
| `import type { Product } from "./models.ts"`  | the whole line is removed                |
| `import { type Product, formatPrice } ...`    | `Product` is removed, `formatPrice` stays |
| `import { Product } ...` (a type)             | TS1484, and Node.js fails (Session 08)   |
| `export type { Product } from "./models.ts"`  | removed: re-exports a type only          |

---

## Named Exports Instead of Default Exports

A module can have one **default export** (`export default function ...`), imported without braces and under any name. This course uses **named exports** only: the name is the same in every file, your editor can import it for you, and `tsc` checks it. Import a default that does not exist

```typescript
import { type Product } from "./models.ts";
import formatPrice from "./models.ts";
```

```text
use-models.ts:2:8 - error TS2613: Module '"D:/ts-practice/24-modules/models"' has no default export. Did you mean to use 'import { formatPrice } from "D:/ts-practice/24-modules/models"' instead?

2 import formatPrice from "./models.ts";
         ~~~~~~~~~~~


Found 1 error in use-models.ts:2
```

With a default export, the same function could be imported as `formatPrice` in one file and `money` in another, and nobody would notice until searching for it fails. You will still meet default imports from packages: `import path from "node:path"` and `import express from "express"` (Session 31) import a module's default. That is normal for packages written in CommonJS (see [Why This Course Writes ES Modules](#why-this-course-writes-es-modules)).

![Two files import the same default export under two different names, price and money, while a named export keeps one name everywhere, and a wrong name is caught by tsc](images/24-modules/named-vs-default.gif)

---

## Barrel Files

A folder often gets an `index.ts` that re-exports what the rest of the app may use. It is called a **barrel**. Other files import from the folder's `index.ts` and do not need to know its inner files

catalog/products.ts

```typescript
import type { Product } from "../models.ts";

const products: Product[] = [
  { id: 1, name: "Keyboard", pricePaise: 149900 },
  { id: 2, name: "Mouse", pricePaise: 49900 }
];

export function findProduct(id: number): Product | undefined {
  return products.find((product) => product.id === id);
}
```

catalog/index.ts

```typescript
export * from "./products.ts";
export type { Product } from "../models.ts";
```

store.ts

```typescript
import { findProduct, type Product } from "./catalog/index.ts";

const product: Product | undefined = findProduct(2);
console.log(product?.name);
```

```text
Mouse
```

- `export * from "./products.ts"` re-exports every export of that file. The array `products` is not exported there, so it stays private to the folder.
- `export type { Product } from "../models.ts"` passes a type on, so users of the catalog get everything from one place.

Only what the barrel exports is reachable. Add at the end of `store.ts`

```typescript
import { products } from "./catalog/index.ts";
```

```text
store.ts:5:10 - error TS2724: '"./catalog/index.ts"' has no exported member named 'products'. Did you mean 'Product'?

5 import { products } from "./catalog/index.ts";
           ~~~~~~~~


Found 1 error in store.ts:5
```

`export *` has one trap. Remove that line, and add an old catalog file with a function of the same name

catalog/legacy.ts

```typescript
export function findProduct(sku: string): string {
  return `legacy product ${sku}`;
}
```

and a third line in `catalog/index.ts`

```typescript
export * from "./legacy.ts";
```

```text
catalog/index.ts:3:1 - error TS2308: Module "./products.ts" has already exported a member named 'findProduct'. Consider explicitly re-exporting to resolve the ambiguity.

3 export * from "./legacy.ts";
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error in catalog/index.ts:3
```

```text
file:///D:/ts-practice/24-modules/store.ts:1
import { findProduct,              } from "./catalog/index.ts";
         ^^^^^^^^^^^
SyntaxError: The requested module './catalog/index.ts' contains conflicting star exports for name 'findProduct'
    at #asyncInstantiate (node:internal/modules/esm/module_job:327:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:431:5)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Two `findProduct`s, so neither wins, and Node.js refuses. (The empty space after `findProduct,` is where Node.js removed `type Product`.) Re-export the second one under its own name

```typescript
export { findProduct as findLegacyProduct } from "./legacy.ts";
```

```text
Mouse
```

Keep barrels small and per feature. A barrel loads every file it re-exports, so one giant `index.ts` for the whole app makes every import load everything.

![The catalog folder has two inner files and a barrel index.ts at its door; store.ts imports only through the door; products, which the barrel does not export, stays inside; two export star lines that both bring findProduct collide at the door](images/24-modules/barrel.gif)

---

## Circular Imports

Two modules that import each other form a **cycle**. TypeScript allows it, and sometimes it works. Sometimes it does not

money.ts

```typescript
import { config } from "./app-config.ts";

export const DEFAULT_CURRENCY = "INR";

export function withTax(paise: number): number {
  return Math.round(paise * (1 + config.taxPercent / 100));
}
```

app-config.ts

```typescript
import { DEFAULT_CURRENCY } from "./money.ts";

export const config = { currency: DEFAULT_CURRENCY, taxPercent: 18 };
```

invoice.ts

```typescript
import { withTax } from "./money.ts";
import { config } from "./app-config.ts";

console.log(withTax(10000), config.currency);
```

```bash
npx tsc --noEmit
node invoice.ts
```

```text
file:///D:/ts-practice/24-modules/app-config.ts:3
export const config = { currency: DEFAULT_CURRENCY, taxPercent: 18 };
                                  ^

ReferenceError: Cannot access 'DEFAULT_CURRENCY' before initialization
    at file:///D:/ts-practice/24-modules/app-config.ts:3:35
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` is quiet, because every name exists. The problem is the **order**. `invoice.ts` imports `money.ts`, which first imports `app-config.ts`. That file imports `money.ts` back, but `money.ts` is already being loaded, so Node.js does not start it again: it runs `app-config.ts` first, while `DEFAULT_CURRENCY` in `money.ts` has no value yet. Swap the two import lines in `invoice.ts`, and the program prints `11800 INR`: the same files, another order, no crash. That makes cycles hard to spot.

The fix is to break the cycle. Move the shared value into a third module that imports nothing

currency.ts

```typescript
export const DEFAULT_CURRENCY = "INR";
```

app-config.ts

```typescript
import { DEFAULT_CURRENCY } from "./currency.ts";

export const config = { currency: DEFAULT_CURRENCY, taxPercent: 18 };
```

money.ts

```typescript
import { config } from "./app-config.ts";

export function withTax(paise: number): number {
  return Math.round(paise * (1 + config.taxPercent / 100));
}
```

```text
11800 INR
```

Now the imports go one way only: `invoice.ts` to `money.ts` to `app-config.ts` to `currency.ts`. A cycle between two type-only imports is harmless, because `import type` disappears before the code runs. Cycles between values are the ones to avoid.

![A load order timeline: invoice starts money, money starts app-config, app-config asks money for DEFAULT_CURRENCY while money is still waiting, and the read fails; after the fix, currency is loaded first and every module finds its values ready](images/24-modules/circular.gif)

---

## Node.js Built-ins and import.meta

Node.js's own modules are imported with the `node:` prefix, and their types come from `@types/node` (Session 02). Here a module reads a data file that sits next to it

paths.ts

```typescript
import { readFile } from "node:fs/promises";
import path from "node:path";

const ratesFile = path.join(import.meta.dirname, "data", "rates.txt");

console.log(path.basename(import.meta.filename));
console.log(await readFile(ratesFile, "utf8"));
```

With a file `data/rates.txt` that contains `GST 18%, shipping 5%`

```text
paths.ts
GST 18%, shipping 5%

```

`import.meta.dirname` is the folder of the current module, and `import.meta.filename` its full path. A path built from them works no matter which folder you start `node` from; a plain `"data/rates.txt"` is relative to that starting folder instead.

CommonJS code uses `__dirname` for this. In an ES module it does not exist, and `tsc` cannot warn you, because `@types/node` declares it for CommonJS files

where-am-i.ts

```typescript
console.log(import.meta.dirname);
console.log(__dirname);
```

```bash
npx tsc --noEmit
node where-am-i.ts
```

```text
D:\ts-practice\24-modules
file:///D:/ts-practice/24-modules/where-am-i.ts:2
console.log(__dirname);
            ^

ReferenceError: __dirname is not defined in ES module scope
    at file:///D:/ts-practice/24-modules/where-am-i.ts:2:13
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

When you copy code from the Node.js course (CommonJS), replace `__dirname` with `import.meta.dirname` and `__filename` with `import.meta.filename`.

![The same question, where am I, asked two ways: import.meta.dirname answers with the folder of the module; __dirname passes tsc, because @types/node knows it, but in an ES module it does not exist and Node.js stops with a ReferenceError](images/24-modules/import-meta.gif)

---

## Importing JSON

ES modules can import a JSON file directly, with an **import attribute** that says it is JSON. TypeScript reads the file and gives the import a type that matches its contents

settings.json

```json
{ "port": 3000, "currency": "INR", "features": ["reviews", "wishlist"] }
```

server-config.ts

```typescript
import settings from "./settings.json" with { type: "json" };

const port: number = settings.port;
console.log(`listening on ${port}, currency ${settings.currency}`);
console.log(settings.features.includes("reviews"));
```

```text
listening on 3000, currency INR
true
```

`settings` has the type `{ port: number; currency: string; features: string[] }`, built from the file. A typo is caught

```typescript
console.log(settings.prot);
```

```text
server-config.ts:6:22 - error TS2339: Property 'prot' does not exist on type '{ port: number; currency: string; features: string[]; }'.

6 console.log(settings.prot);
                       ~~~~


Found 1 error in server-config.ts:6
```

Without `with { type: "json" }`, both refuse: `tsc` gives `error TS1543: Importing a JSON file into an ECMAScript module requires a 'type: "json"' import attribute when 'module' is set to 'NodeNext'.`, and Node.js stops with `ERR_IMPORT_ATTRIBUTE_MISSING`.

The type is only as good as the file. It describes the JSON you wrote, so it fits files that are part of your code, like default settings. Data that comes from outside at run time still needs checking (Session 26).

![TypeScript opens settings.json, reads each value, and builds the type port number, currency string, features string array; the with type json label is required by both tsc and Node.js](images/24-modules/json-import.gif)

---

## Loading a Module Later with import()

`import` at the top of a file loads a module when the program starts. The function `import()` loads it later, when the code reaches it, and returns a promise of the module. That is useful for code that is rarely needed, like a heavy report

reports/sales-report.ts

```typescript
console.log("  (loading the report module)");

export function buildSalesReport(totals: number[]): string {
  const sum = totals.reduce((total, value) => total + value, 0);
  return `report: ${totals.length} orders, Rs ${sum / 100}`;
}
```

routes.ts

```typescript
async function handleRequest(path: string): Promise<string> {
  if (path === "/reports/sales") {
    const { buildSalesReport } = await import("./reports/sales-report.ts");
    return buildSalesReport([249900, 79900]);
  }
  return `200 ${path}`;
}

console.log(await handleRequest("/orders"));
console.log(await handleRequest("/reports/sales"));
console.log(await handleRequest("/reports/sales"));
```

```text
200 /orders
  (loading the report module)
report: 2 orders, Rs 3298
report: 2 orders, Rs 3298
```

The report module is loaded on the first report request, and only once: modules are kept after their first load (JavaScript course, Session 21). With a path written out like this, `import()` is fully typed: `buildSalesReport("249900")` gives `error TS2345: Argument of type 'string' is not assignable to parameter of type 'number[]'.`

With a path built at run time, TypeScript cannot know which file you mean, and the module is `any`

```typescript
const reportName = "sales-report";
const loose = await import(`./reports/${reportName}.ts`);
console.log(loose.buildSalesReprot([100]));
```

`tsc` is quiet about the typo, and Node.js fails with `TypeError: loose.buildSalesReprot is not a function`. Keep `import()` paths as plain text, like Mini Example 2 does.

![A request for /orders runs without the report module; the first request for /reports/sales loads the module and prints its loading line; the second request uses the module that is already loaded](images/24-modules/dynamic-import.gif)

---

## Why This Course Writes ES Modules

The Node.js course uses CommonJS: `require` and `module.exports`. TypeScript can describe CommonJS too, in `.cts` files or in a project with `"type": "commonjs"`, but this course's settings do not fit it well. A `.cts` file is CommonJS, and ES module syntax is not allowed in it

legacy.cts

```typescript
import { readFileSync } from "node:fs";

export function total(prices: number[]): number {
  return prices.reduce((sum, price) => sum + price, 0);
}
```

```text
legacy.cts:1:10 - error TS1286: ECMAScript imports and exports cannot be written in a CommonJS file under 'verbatimModuleSyntax'.

1 import { readFileSync } from "node:fs";
           ~~~~~~~~~~~~

legacy.cts:3:1 - error TS1287: A top-level 'export' modifier cannot be used on value declarations in a CommonJS module when 'verbatimModuleSyntax' is enabled.

3 export function total(prices: number[]): number {
  ~~~~~~


Found 2 errors in the same file, starting at: legacy.cts:1
```

The TypeScript way to export from CommonJS, `export = { total }`, is not erasable syntax, so it gives `error TS1294` in this setup. And plain `module.exports` runs, but TypeScript does not read it as an export in a `.ts` or `.cts` file

legacy.cts

```typescript
function total(prices: number[]): number {
  return prices.reduce((sum, price) => sum + price, 0);
}

module.exports = { total };
```

use-legacy.ts

```typescript
import legacy from "./legacy.cts";

console.log(legacy.total([49900, 19900]));
```

```text
use-legacy.ts:3:20 - error TS2339: Property 'total' does not exist on type 'typeof import("D:/ts-practice/24-modules/legacy")'.

3 console.log(legacy.total([49900, 19900]));
                     ~~~~~


Found 1 error in use-legacy.ts:3
```

Node.js runs it and prints `69800`, but the types are gone. So in TypeScript, write your own code as ES modules. CommonJS still matters in one place: many npm packages are written in it. Importing them from ES modules works, usually through a default import (`import express from "express"`), and their types come from `.d.ts` files, which are Session 25.

![Three ways to write a CommonJS module in TypeScript are each refused or lose their types: import and export in a .cts file, export equals, and module.exports; an ES module with export passes and keeps its types](images/24-modules/commonjs.gif)

---

## Mini Example 1: A Shop in Feature Folders

Real backends group files by **feature**: everything about customers in one folder, everything about orders in another, and shared code in a third. Each feature folder has a barrel, and other folders import only through it

```
shop/
  shared/types.ts
  shared/money.ts
  customers/repository.ts
  customers/index.ts
  orders/repository.ts
  orders/summary.ts
  orders/index.ts
  shop.json
  app.ts
```

shop/shared/types.ts

```typescript
export interface Customer {
  id: number;
  name: string;
  city: string;
}

export interface Order {
  id: number;
  customerId: number;
  totalPaise: number;
}
```

shop/shared/money.ts

```typescript
export function formatRupees(paise: number): string {
  return `Rs ${(paise / 100).toFixed(2)}`;
}
```

shop/customers/repository.ts

```typescript
import type { Customer } from "../shared/types.ts";

const customers: Customer[] = [
  { id: 7, name: "Asha", city: "Pune" },
  { id: 8, name: "Ravi", city: "Indore" }
];

export function findCustomer(id: number): Customer | undefined {
  return customers.find((customer) => customer.id === id);
}
```

shop/customers/index.ts

```typescript
export { findCustomer } from "./repository.ts";
```

shop/orders/repository.ts

```typescript
import type { Order } from "../shared/types.ts";

const orders: Order[] = [
  { id: 1001, customerId: 7, totalPaise: 249900 },
  { id: 1002, customerId: 7, totalPaise: 79900 },
  { id: 1003, customerId: 8, totalPaise: 19900 }
];

export function ordersFor(customerId: number): Order[] {
  return orders.filter((order) => order.customerId === customerId);
}
```

shop/orders/summary.ts

```typescript
import type { Customer, Order } from "../shared/types.ts";
import { formatRupees } from "../shared/money.ts";

export function summarize(customer: Customer, orders: Order[]): string {
  const total = orders.reduce((sum, order) => sum + order.totalPaise, 0);
  return `${customer.name} (${customer.city}): ${orders.length} order(s), ${formatRupees(total)}`;
}
```

shop/orders/index.ts

```typescript
export { ordersFor } from "./repository.ts";
export { summarize } from "./summary.ts";
```

shop/shop.json

```json
{ "shopName": "ShopAPI", "currency": "INR" }
```

shop/app.ts

```typescript
import shop from "./shop.json" with { type: "json" };
import { findCustomer } from "./customers/index.ts";
import { ordersFor, summarize } from "./orders/index.ts";

console.log(`${shop.shopName} (${shop.currency})`);

for (const id of [7, 8, 9]) {
  const customer = findCustomer(id);
  if (customer === undefined) {
    console.log(`customer ${id} not found`);
    continue;
  }
  console.log(summarize(customer, ordersFor(id)));
}
```

```bash
npx tsc --noEmit
node shop/app.ts
```

```text
ShopAPI (INR)
Asha (Pune): 2 order(s), Rs 3298.00
Ravi (Indore): 1 order(s), Rs 199.00
customer 9 not found
```

How it fits together:

- `shared/types.ts` holds only types, so every import of it is an `import type`. No feature imports another feature's inner files.
- The barrels list the public functions of each feature. The arrays inside the repositories are private: `import { orders } from "./orders/index.ts"` gives `error TS2305: Module '"./orders/index.ts"' has no exported member 'orders'.`
- The imports only go one way: `app.ts` uses the features, the features use `shared`, and `shared` uses nothing. No cycles.
- Later, a repository can switch from an array to MongoDB (Session 32) without any change in `app.ts`.

![The shop as a graph of folders: app.ts at the top imports the customers and orders barrels; each barrel leads to its inner files; both features point down to shared types and money; every arrow points downwards, so there are no cycles](images/24-modules/shop-modules.gif)

---

## Mini Example 2: Payment Providers Loaded on Demand

A shop supports several payment providers, but most orders use only one or two. Each provider lives in its own module and is loaded with `import()` the first time it is needed. A table of loaders keeps everything typed

payments/types.ts

```typescript
export interface PaymentProvider {
  readonly name: string;
  charge(amountPaise: number): string;
}
```

payments/providers/card.ts

```typescript
import type { PaymentProvider } from "../types.ts";

console.log("  (loading the card provider)");

export const provider: PaymentProvider = {
  name: "card",
  charge(amountPaise: number): string {
    return `card charged ${amountPaise} paise`;
  }
};
```

payments/providers/upi.ts

```typescript
import type { PaymentProvider } from "../types.ts";

console.log("  (loading the upi provider)");

export const provider: PaymentProvider = {
  name: "upi",
  charge(amountPaise: number): string {
    return `UPI request for ${amountPaise} paise`;
  }
};
```

payments/registry.ts

```typescript
import type { PaymentProvider } from "./types.ts";

const loaders = {
  card: () => import("./providers/card.ts"),
  upi: () => import("./providers/upi.ts")
} satisfies Record<string, () => Promise<{ provider: PaymentProvider }>>;

export type ProviderName = keyof typeof loaders;

export function isProviderName(name: string): name is ProviderName {
  return Object.hasOwn(loaders, name);
}

export async function getProvider(name: ProviderName): Promise<PaymentProvider> {
  const loaded = await loaders[name]();
  return loaded.provider;
}
```

payments/checkout.ts

```typescript
import { getProvider, isProviderName } from "./registry.ts";

const requests = ["upi", "card", "upi", "crypto"];

for (const name of requests) {
  if (!isProviderName(name)) {
    console.log(`${name}: unknown payment method`);
    continue;
  }
  const provider = await getProvider(name);
  console.log(`${name}: ${provider.charge(49900)}`);
}
```

```bash
npx tsc --noEmit
node payments/checkout.ts
```

```text
  (loading the upi provider)
upi: UPI request for 49900 paise
  (loading the card provider)
card: card charged 49900 paise
upi: UPI request for 49900 paise
crypto: unknown payment method
```

What each part does:

- `loaders` maps a name to a function that loads that provider's module. Each path is plain text, so each `import()` is fully typed.
- `satisfies Record<string, () => Promise<{ provider: PaymentProvider }>>` (Sessions 13 and 17) checks that every module exports a correct `provider`, without changing the type of `loaders`.
- `ProviderName` is `keyof typeof loaders` (Session 15): `"card" | "upi"`. Adding a loader adds a name.
- `isProviderName` turns a plain string from outside into a `ProviderName`, with `Object.hasOwn` (Session 15's safe check).
- The output shows the lazy loading: each provider loads when it is first used, and the second `upi` request reuses the loaded module.

Every part is checked. Ask for a provider by a name that does not exist, at the end of `checkout.ts`

```typescript
await getProvider("crypto");
```

```text
payments/checkout.ts:14:19 - error TS2345: Argument of type '"crypto"' is not assignable to parameter of type '"card" | "upi"'.

14 await getProvider("crypto");
                     ~~~~~~~~


Found 1 error in payments/checkout.ts:14
```

Remove that line, and add a wallet provider that forgot its `charge` method

payments/providers/wallet.ts

```typescript
console.log("  (loading the wallet provider)");

export const provider = {
  name: "wallet"
};
```

```typescript
  upi: () => import("./providers/upi.ts"),
  wallet: () => import("./providers/wallet.ts")
```

```text
payments/registry.ts:6:17 - error TS2322: Type 'Promise<typeof import("D:/ts-practice/24-modules/payments/providers/wallet")>' is not assignable to type 'Promise<{ provider: PaymentProvider; }>'.
  Type 'typeof import("D:/ts-practice/24-modules/payments/providers/wallet")' is not assignable to type '{ provider: PaymentProvider; }'.
    Types of property 'provider' are incompatible.
      Property 'charge' is missing in type '{ name: string; }' but required in type 'PaymentProvider'.

6   wallet: () => import("./providers/wallet.ts")
                  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  payments/types.ts:3:3 - 'charge' is declared here.
    3   charge(amountPaise: number): string;
        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  payments/registry.ts:7:28 - The expected type comes from the return type of this signature.
    7 } satisfies Record<string, () => Promise<{ provider: PaymentProvider }>>;
                                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

payments/registry.ts:17:3 - error TS2322: Type 'PaymentProvider | { name: string; }' is not assignable to type 'PaymentProvider'.
  Property 'charge' is missing in type '{ name: string; }' but required in type 'PaymentProvider'.

17   return loaded.provider;
     ~~~~~~

  payments/types.ts:3:3 - 'charge' is declared here.
    3   charge(amountPaise: number): string;
        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: payments/registry.ts:6
```

`typeof import("...")` in the message is the type of a whole module. `tsc` looked into `wallet.ts` without running it, and found that its `provider` has no `charge`.

![The loaders table has a row per provider; a request for upi loads the upi module the first time and reuses it the second time, card loads when it is first asked for, and crypto is turned away by isProviderName without loading anything](images/24-modules/lazy-providers.gif)

---

## Beginner Mistakes

### Mistake 1: Leaving out the file extension

`import { addGst } from "./pricing/tax"` gives `error TS2835` and `ERR_MODULE_NOT_FOUND`. Write `"./pricing/tax.ts"`.

---

### Mistake 2: Following the .js suggestion

TS2835 suggests `'./pricing/tax.js'`. In this course, Node.js runs the `.ts` files, and a `.js` path passes `tsc` but fails in Node.js. Write `.ts`; a build rewrites it.

---

### Mistake 3: Importing a folder

`import ... from "./pricing"` gives `error TS2834` and `ERR_UNSUPPORTED_DIR_IMPORT`. ES modules do not look for `index` files. Write `"./pricing/index.ts"`.

---

### Mistake 4: Putting a value in import type

`import type { formatPrice }` and then calling `formatPrice` gives `error TS1361`. Use `import { type Product, formatPrice }`.

---

### Mistake 5: Importing a default that does not exist

`import formatPrice from "./models.ts"` gives `error TS2613` with "Did you mean to use 'import { formatPrice } ...'". Use braces for named exports.

---

### Mistake 6: Two export * lines with the same name

`export *` from two files that both export `findProduct` gives `error TS2308`, and Node.js refuses with "conflicting star exports". Re-export one of them by name, with `as`.

---

### Mistake 7: Creating a circular import between values

Two modules that import values from each other can fail with `ReferenceError: Cannot access ... before initialization`, depending on which file starts. `tsc` does not see it. Move the shared values into a third module.

---

### Mistake 8: Using __dirname in an ES module

`__dirname` passes `tsc` and fails with `ReferenceError: __dirname is not defined in ES module scope`. Use `import.meta.dirname`.

---

### Mistake 9: Importing JSON without the attribute

`import settings from "./settings.json"` gives `error TS1543` and `ERR_IMPORT_ATTRIBUTE_MISSING`. Add `with { type: "json" }`.

---

### Mistake 10: Building import() paths at run time

`` import(`./reports/${name}.ts`) `` gives a module of type `any`, so typos pass `tsc`. Write each path as plain text, for example in a table of loaders.

---

### Mistake 11: Writing CommonJS in TypeScript files

`module.exports` in a `.cts` file runs, but TypeScript does not see the export (TS2339 where it is used), and `export =` gives TS1294 in this setup. Write ES modules.

---

## Practice Exercises

### Exercise 1

Split `pricing/tax.ts` into `pricing/gst.ts` and `pricing/shipping.ts` (with `addShipping(paise, city)`), add a barrel `pricing/index.ts`, and update `checkout.ts` to import only from the barrel.

### Exercise 2

Write `units.ts` with `export type Paise = number` and `export function toRupees(paise: Paise): number`. Import both in another file: first with `import type` for both (read the error), then correctly.

### Exercise 3

Build a cycle on purpose: `a.ts` exports `const A = "a" + B` and imports `B` from `b.ts`; `b.ts` exports `const B = "b"` and imports `A`. Start the program from `a.ts`, then from `b.ts`, and explain both results.

### Exercise 4

Write `logger.ts` that appends lines to a file `logs/app.log` next to the module, using `import.meta.dirname` and `appendFile` from `node:fs/promises`. Run it from the `ts-practice` folder and from `24-modules`, and check that the log ends up in the same place.

### Exercise 5

Import `package.json` of your `ts-practice` folder with `with { type: "json" }` and print its `name` and `type`. Then try to read a field that does not exist and read the error.

### Exercise 6

In `routes.ts`, add a `/reports/stock` route that loads a new module `reports/stock-report.ts` with `import()`. Make sure each report module is loaded only when its route is used.

### Exercise 7

In Mini Example 1, add a `products` feature folder with a repository and a barrel, and give each order a list of product ids. Make `summarize` print product names, without creating a cycle between `orders` and `products`.

### Exercise 8

In Mini Example 2, add a working `wallet` provider, and a function `chargeWithFallback(names: ProviderName[], amountPaise)` that tries the providers in order and stops at the first one whose `charge` does not throw.

---

## Interview Questions

### Why must every relative import end with .ts in this setup?

Node.js resolves ES module paths exactly as written and runs the `.ts` files directly. A missing extension gives TS2835 and `ERR_MODULE_NOT_FOUND`; a `.js` extension fails in Node.js. A build rewrites `.ts` to `.js` with `rewriteRelativeImportExtensions`.

### What does moduleDetection: force do?

It makes TypeScript treat every file as a module, with its own top-level scope, even without `import` or `export`. With the older rule, such files are scripts that share one global scope.

### What is the difference between import type and import { type X }?

`import type { A, B }` removes the whole import at run time, so it may contain only types. `import { type A, b }` removes only `A` and keeps the value `b`.

### Why prefer named exports?

The name is the same in every file, editors can import it automatically, and `tsc` checks it. A default export can be imported under any name.

### What is a barrel file?

An `index.ts` that re-exports a folder's public parts, so other code imports from one place. It hides the inner files, but loads everything it re-exports.

### What problem do circular imports cause?

When two modules import values from each other, one of them runs before the other has set its values, and reading such a value throws a `ReferenceError`. Whether it happens depends on load order, and TypeScript does not report it.

### How do you get the current folder in an ES module?

With `import.meta.dirname` (and `import.meta.filename` for the file). `__dirname` does not exist in ES modules, although `tsc` accepts it.

### How do you import JSON in TypeScript with Node.js?

`import data from "./file.json" with { type: "json" }`. TypeScript gives `data` a type built from the file's contents. Without the attribute, `tsc` reports TS1543 and Node.js refuses.

### What does import() return, and how is it typed?

A promise of the module's exports. With a literal path, TypeScript knows the module's types; with a path built at run time, the module is `any`.

### What is typeof import("./x.ts")?

The type of the whole module `./x.ts`: an object type with all its exports. It is used, for example, to describe what an `import()` returns.

### Can you write CommonJS in TypeScript?

Yes, with `.cts` files or `"type": "commonjs"`, but with `verbatimModuleSyntax` and `erasableSyntaxOnly` it is awkward: ES syntax is refused, `export =` is not erasable, and `module.exports` is not typed. Write ES modules, and import CommonJS packages with their `.d.ts` types.

### How do you load optional code only when it is needed, without losing types?

With `import()` and literal paths, for example in a table of loader functions checked with `satisfies`, so each module's exports are verified.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [23. Async TypeScript and Error Handling](23-async-and-error-handling.md) | [Course outline](README.md#course-outline) | [25. Declaration Files and @types](25-declaration-files-and-types.md) |
