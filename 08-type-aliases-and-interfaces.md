# Type Aliases and Interfaces

> **Session 8 of 33** | Part 2: Typing Your Data | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Why Give a Type a Name](#why-give-a-type-a-name)
- [Type Aliases](#type-aliases)
- [Error Messages Use the Name](#error-messages-use-the-name)
- [An Alias Can Name Any Type](#an-alias-can-name-any-type)
- [An Alias Is Not a New Type](#an-alias-is-not-a-new-type)
- [Interfaces](#interfaces)
- [Extending an Interface](#extending-an-interface)
- [Extending Several Interfaces](#extending-several-interfaces)
- [Intersections: Combining Types with &](#intersections-combining-types-with-)
- [extends or &: What Happens on a Conflict](#extends-or--what-happens-on-a-conflict)
- [Interfaces Merge, Type Aliases Do Not](#interfaces-merge-type-aliases-do-not)
- [type or interface?](#type-or-interface)
- [Sharing Types Between Files](#sharing-types-between-files)
- [A Type That Refers to Itself](#a-type-that-refers-to-itself)
- [Naming Types](#naming-types)
- [Mini Example 1: The Order Summary in Three Files](#mini-example-1-the-order-summary-in-three-files)
- [Mini Example 2: A Product Catalog with extends](#mini-example-2-a-product-catalog-with-extends)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `08-aliases-interfaces` inside `ts-practice`, with the one-line `tsconfig.json`

08-aliases-interfaces/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## Why Give a Type a Name

At the end of Session 07, `orderSummary` had a parameter type seven lines long, and the item shape `{ name: string; pricePaise: number; quantity: number }` was buried inside it. Every function that works with items would have to write that shape again, and adding one property would mean finding every copy.

The fix: write the type **once**, give it a name, and use the name everywhere. TypeScript has two ways to do that, **type aliases** and **interfaces**. You will see both in every TypeScript project.

![The same long object type written in three places is replaced by one named type used in all three](images/08-type-aliases-and-interfaces/name-a-type.gif)

---

## Type Aliases

`type`, a name, `=`, and any type

alias.ts

```typescript
type Product = {
  name: string;
  pricePaise: number;
};

function formatProduct(product: Product): string {
  return `${product.name}: Rs. ${(product.pricePaise / 100).toFixed(2)}`;
}

const keyboard: Product = { name: "Keyboard", pricePaise: 149900 };
const cart: Product[] = [keyboard, { name: "Mouse", pricePaise: 49900 }];

for (const item of cart) {
  console.log(formatProduct(item));
}
```

```bash
npx tsc --noEmit
node alias.ts
```

```text
Keyboard: Rs. 1499.00
Mouse: Rs. 499.00
```

`Product` is now a name for `{ name: string; pricePaise: number }`, and it works everywhere a type goes: a parameter, a variable, an array (`Product[]`). Everything from Session 07 still applies: optional and `readonly` properties, the excess property check, structural typing.

Like every type, an alias disappears when the code runs. It is only there for `tsc` and your editor.

---

## Error Messages Use the Name

alias-error.ts

```typescript
type Product = {
  name: string;
  pricePaise: number;
};

const mouse: Product = { name: "Mouse" };
const monitor: Product = { name: "Monitor", pricePaise: "12999" };
```

```text
alias-error.ts:6:7 - error TS2741: Property 'pricePaise' is missing in type '{ name: string; }' but required in type 'Product'.

6 const mouse: Product = { name: "Mouse" };
        ~~~~~

  alias-error.ts:3:3 - 'pricePaise' is declared here.
    3   pricePaise: number;
        ~~~~~~~~~~

alias-error.ts:7:45 - error TS2322: Type 'string' is not assignable to type 'number'.

7 const monitor: Product = { name: "Monitor", pricePaise: "12999" };
                                              ~~~~~~~~~~

  alias-error.ts:3:3 - The expected type comes from property 'pricePaise' which is declared here on type 'Product'
    3   pricePaise: number;
        ~~~~~~~~~~


Found 2 errors in the same file, starting at: alias-error.ts:6
```

Compare with Session 07: `required in type 'Product'` instead of `required in type '{ name: string; pricePaise: number; }'`, and the extra lines point to the property inside the type, where you would fix it. With bigger types this makes errors much shorter. VS Code shows the name too when you point at `keyboard`.

---

## An Alias Can Name Any Type

Not only objects

alias-kinds.ts

```typescript
type ProductId = string;
type PricePair = [rupees: number, paise: number];
type Cart = { name: string; pricePaise: number }[];
type Stock = { [productId: ProductId]: number };

const id: ProductId = "KB-01";
const price: PricePair = [1499, 50];
const cart: Cart = [{ name: "Keyboard", pricePaise: 149900 }];
const stock: Stock = { [id]: 12 };

console.log(id, price, cart, stock);
```

```text
KB-01 [ 1499, 50 ] [ { name: 'Keyboard', pricePaise: 149900 } ] { 'KB-01': 12 }
```

| Alias         | Names                                    |
| ------------- | ---------------------------------------- |
| `ProductId`   | A primitive: `string`                    |
| `PricePair`   | A labeled tuple (Session 06)             |
| `Cart`        | An array of objects                      |
| `Stock`       | An object with an index signature (Session 07) |

A name like `ProductId` documents what a value means: `id: ProductId` says more than `id: string`. In the next sessions you will name union types and function types with `type` too.

---

## An Alias Is Not a New Type

same-type.ts

```typescript
type Paise = number;
type Rupees = number;

const pricePaise: Paise = 149900;
const shownRupees: Rupees = pricePaise;

console.log(`Rs. ${shownRupees}`);
```

```bash
npx tsc --noEmit
node same-type.ts
```

No errors, and

```text
Rs. 149900
```

A keyboard for one and a half lakh rupees. `Paise` and `Rupees` are both just other names for `number`, and a `number` fits a `number`. An alias gives a type a **name**; it does not make a separate type. This course keeps writing the unit in the variable name, `pricePaise`, because the type cannot tell the two apart.

![Paise and Rupees are two labels on the same number type, so a paise value flows into a rupees variable without an error](images/08-type-aliases-and-interfaces/alias-not-new.gif)

---

## Interfaces

The second way to name an object type

interface.ts

```typescript
interface Product {
  readonly sku: string;
  name: string;
  pricePaise: number;
  inStock?: boolean;
}

const keyboard: Product = { sku: "KB-01", name: "Keyboard", pricePaise: 149900 };

keyboard.pricePaise = 129900;
keyboard.sku = "KB-02";
console.log(keyboard.inStock.valueOf());
```

```text
interface.ts:11:10 - error TS2540: Cannot assign to 'sku' because it is a read-only property.

11 keyboard.sku = "KB-02";
            ~~~

interface.ts:12:13 - error TS18048: 'keyboard.inStock' is possibly 'undefined'.

12 console.log(keyboard.inStock.valueOf());
               ~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: interface.ts:11
```

`interface`, a name, and the properties between `{ }`, with **no `=`**. `readonly` and `?` work as in Session 07. For describing an object, an interface and a type alias do the same job. The differences are in what else they can do, coming up next.

Without the two wrong lines

```text
{ sku: 'KB-01', name: 'Keyboard', pricePaise: 129900 }
```

Index signatures and methods work in interfaces too

interface-index.ts

```typescript
interface Stock {
  [productId: string]: number;
}

interface CartItem {
  name: string;
  pricePaise: number;
  quantity: number;
  total(): number;
}

const stock: Stock = { "KB-01": 12 };
const item: CartItem = {
  name: "Keyboard",
  pricePaise: 149900,
  quantity: 2,
  total() {
    return this.pricePaise * this.quantity;
  }
};

console.log(stock["KB-01"], item.total());
```

```text
12 299800
```

---

## Extending an Interface

Many types share a base: every product has an id, a name and a price, and a digital product also has a download link. `extends` builds one interface on another

extends.ts

```typescript
interface Product {
  readonly id: string;
  name: string;
  pricePaise: number;
}

interface DigitalProduct extends Product {
  downloadUrl: string;
}

const ebook: DigitalProduct = {
  id: "EB-01",
  name: "TypeScript Notes",
  pricePaise: 29900,
  downloadUrl: "https://example.com/ts-notes.pdf"
};

console.log(`${ebook.name}: ${ebook.downloadUrl}`);
```

```text
TypeScript Notes: https://example.com/ts-notes.pdf
```

`DigitalProduct` has everything from `Product` plus `downloadUrl`. All of them are required

extends-missing.ts

```typescript
interface Product {
  readonly id: string;
  name: string;
  pricePaise: number;
}

interface DigitalProduct extends Product {
  downloadUrl: string;
}

const course: DigitalProduct = {
  id: "VC-01",
  name: "Video course",
  downloadUrl: "https://example.com/course"
};
```

```text
extends-missing.ts:11:7 - error TS2741: Property 'pricePaise' is missing in type '{ id: string; name: string; downloadUrl: string; }' but required in type 'DigitalProduct'.

11 const course: DigitalProduct = {
         ~~~~~~

  extends-missing.ts:4:3 - 'pricePaise' is declared here.
    4   pricePaise: number;
        ~~~~~~~~~~


Found 1 error in extends-missing.ts:11
```

The extra lines point to line 4, inside `Product`: the missing property was inherited from there. And because of structural typing, a `DigitalProduct` fits anywhere a `Product` is needed: it has everything a `Product` has.

![A DigitalProduct box is built on top of the Product box: it has the id, name and price from Product, plus its own download link](images/08-type-aliases-and-interfaces/interface-extends.gif)

---

## Extending Several Interfaces

An interface can extend more than one, separated by commas

user.ts

```typescript
interface HasId {
  readonly id: string;
}

interface HasTimestamps {
  createdAt: string;
  updatedAt: string;
}

interface User extends HasId, HasTimestamps {
  name: string;
  email: string;
}

const user: User = {
  id: "U-1",
  name: "Riya",
  email: "riya@example.com",
  createdAt: "2026-10-01",
  updatedAt: "2026-10-20"
};

console.log(`${user.id} ${user.name}, updated ${user.updatedAt}`);
```

```text
U-1 Riya, updated 2026-10-20
```

Small building blocks like `HasId` and `HasTimestamps` are common in backend code: almost every record in a database has an id and timestamps. You will see this pattern again with Mongoose in Session 32.

---

## Intersections: Combining Types with &

Type aliases cannot use `extends`. They combine types with `&`, an **intersection**: "everything from this type **and** everything from that one"

intersection.ts

```typescript
type Product = {
  readonly id: string;
  name: string;
  pricePaise: number;
};

type DigitalProduct = Product & { downloadUrl: string };

const ebook: DigitalProduct = {
  id: "EB-01",
  name: "TypeScript Notes",
  pricePaise: 29900,
  downloadUrl: "https://example.com/ts-notes.pdf"
};

console.log(`${ebook.name}: ${ebook.downloadUrl}`);
```

```text
TypeScript Notes: https://example.com/ts-notes.pdf
```

The result is the same as with `extends`. `&` can join any number of types: `HasId & HasTimestamps & { name: string }`.

![Two object types are joined with an ampersand into one type that has the properties of both](images/08-type-aliases-and-interfaces/intersection.gif)

---

## extends or &: What Happens on a Conflict

What if both sides have the same property with different types? A free sample whose `pricePaise` is a text, say

extends-conflict.ts

```typescript
interface Product {
  name: string;
  pricePaise: number;
}

interface FreeSample extends Product {
  pricePaise: string;
}
```

```text
extends-conflict.ts:6:11 - error TS2430: Interface 'FreeSample' incorrectly extends interface 'Product'.
  Types of property 'pricePaise' are incompatible.
    Type 'string' is not assignable to type 'number'.

6 interface FreeSample extends Product {
            ~~~~~~~~~~


Found 1 error in extends-conflict.ts:6
```

`extends` reports the conflict right where the interface is declared. Now the same with `&`

intersection-conflict.ts

```typescript
type Product = {
  name: string;
  pricePaise: number;
};

type FreeSample = Product & { pricePaise: string };

const sample: FreeSample = { name: "Sticker", pricePaise: 0 };
```

```text
intersection-conflict.ts:8:47 - error TS2322: Type 'number' is not assignable to type 'never'.

8 const sample: FreeSample = { name: "Sticker", pricePaise: 0 };
                                                ~~~~~~~~~~

  intersection-conflict.ts:3:3 - The expected type comes from property 'pricePaise' which is declared here on type 'FreeSample'
    3   pricePaise: number;
        ~~~~~~~~~~


Found 1 error in intersection-conflict.ts:8
```

No error on line 6. The intersection is allowed, and `pricePaise` becomes something that is a `number` **and** a `string` at the same time. No value is both, so the type is `never` (Session 04), and the error shows up only later, when you try to make a `FreeSample`, with a confusing message about `never`.

When you see "not assignable to type 'never'" on a property, look for an intersection with a conflict.

![extends reports a conflicting property where the interface is declared, while an intersection quietly turns the property into never and fails later](images/08-type-aliases-and-interfaces/conflict.gif)

---

## Interfaces Merge, Type Aliases Do Not

Declare the same interface twice, and TypeScript joins them into one

merging.ts

```typescript
interface Customer {
  name: string;
}

interface Customer {
  phone: string;
}

const customer: Customer = { name: "Riya" };
```

```text
merging.ts:9:7 - error TS2741: Property 'phone' is missing in type '{ name: string; }' but required in type 'Customer'.

9 const customer: Customer = { name: "Riya" };
        ~~~~~~~~

  merging.ts:6:3 - 'phone' is declared here.
    6   phone: string;
        ~~~~~


Found 1 error in merging.ts:9
```

`Customer` now has both `name` and `phone`. This is called **declaration merging**. It exists so that you can add properties to types from libraries: in Session 31 you will use it to add a logged-in `user` to Express's request type. Inside your own code, it is usually an accident: two interfaces with the same name in one file quietly become one.

A type alias with the same name twice is an error

duplicate-alias.ts

```typescript
type Customer = { name: string };
type Customer = { phone: string };
```

```text
duplicate-alias.ts:1:6 - error TS2300: Duplicate identifier 'Customer'.

1 type Customer = { name: string };
       ~~~~~~~~

duplicate-alias.ts:2:6 - error TS2300: Duplicate identifier 'Customer'.

2 type Customer = { phone: string };
       ~~~~~~~~


Found 2 errors in the same file, starting at: duplicate-alias.ts:1
```

![Two interface declarations with the same name merge into one type with both properties, while two type aliases with the same name are a duplicate error](images/08-type-aliases-and-interfaces/declaration-merging.gif)

---

## type or interface?

|                                   | `type`                              | `interface`                          |
| --------------------------------- | ----------------------------------- | ------------------------------------ |
| Name an object shape              | Yes                                 | Yes                                  |
| Name a primitive, tuple or array  | Yes: `type ProductId = string`      | No                                   |
| Name a union (Session 09)         | Yes                                 | No                                   |
| Build on another type             | `&` (intersection)                  | `extends`                            |
| Conflicting property              | Becomes `never`, fails later        | TS2430 where it is declared          |
| Same name declared twice          | TS2300                              | Merged into one                      |
| Syntax                            | `type Product = { ... };`           | `interface Product { ... }`          |

For plain object shapes both work, and teams pick one style and stick to it. The TypeScript handbook's advice is to use `interface` until you need something only `type` can do. The rule this course follows:

- **`interface` for object shapes**: products, orders, users, request bodies.
- **`type` for everything else**: primitives, tuples, unions, function types, and combinations.

A small preview of something only `type` can do, a union of exact texts (Session 09)

status.ts

```typescript
type OrderStatus = "placed" | "shipped" | "delivered";

let status: OrderStatus = "placed";
status = "shipped";
status = "lost";
```

```text
status.ts:5:1 - error TS2322: Type '"lost"' is not assignable to type 'OrderStatus'.

5 status = "lost";
  ~~~~~~


Found 1 error in status.ts:5
```

![A side-by-side comparison of type and interface: both name object shapes, type also names unions and tuples, interface extends and merges](images/08-type-aliases-and-interfaces/type-vs-interface.gif)

---

## Sharing Types Between Files

The point of naming a type is to use it everywhere, also in other files. Export it like a function, and import it with **`import type`**

shared/types.ts

```typescript
export interface Product {
  readonly id: string;
  name: string;
  pricePaise: number;
}
```

shared/catalog.ts

```typescript
import type { Product } from "./types.ts";

const products: Product[] = [
  { id: "KB-01", name: "Keyboard", pricePaise: 149900 },
  { id: "MS-02", name: "Mouse", pricePaise: 49900 }
];

console.log(products.map((product) => product.name).join(", "));
```

```bash
npx tsc --noEmit
node shared/catalog.ts
```

```text
Keyboard, Mouse
```

Why `import type`? Leave the word out

```typescript
import { Product } from "./types.ts";
```

```text
shared/catalog.ts:1:10 - error TS1484: 'Product' is a type and must be imported using a type-only import when 'verbatimModuleSyntax' is enabled.

1 import { Product } from "./types.ts";
           ~~~~~~~


Found 1 error in shared/catalog.ts:1
```

And this time Node.js fails too

```text
file:///D:/ts-practice/08-aliases-interfaces/shared/catalog.ts:1
import { Product } from "./types.ts";
         ^^^^^^^
SyntaxError: The requested module './types.ts' does not provide an export named 'Product'
    at #asyncInstantiate (node:internal/modules/esm/module_job:327:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:431:5)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

When Node.js removes the types, `export interface Product` disappears from `types.ts`, so there is nothing called `Product` to import. Node.js reads one file at a time and cannot know that `Product` was only a type. `import type` tells it: remove this whole line. That is what `verbatimModuleSyntax` in your tsconfig (Session 02) checks for you.

When one import brings both a type and a value, mark just the type

inline/shop.ts

```typescript
import { type Product, formatProduct } from "./products.ts";

const mouse: Product = { id: "MS-02", name: "Mouse", pricePaise: 49900 };
console.log(formatProduct(mouse));
```

```text
Mouse: Rs. 499.00
```

(`inline/products.ts` is `shared/types.ts` plus an exported `formatProduct` function.)

![import type is removed when the code runs, so it works; a plain import of a type survives and asks for an export that no longer exists, and Node.js stops](images/08-type-aliases-and-interfaces/import-type.gif)

---

## A Type That Refers to Itself

A product category can have sub-categories, which can have their own. A type can use its own name

categories.ts

```typescript
interface Category {
  name: string;
  children: Category[];
}

const electronics: Category = {
  name: "Electronics",
  children: [
    { name: "Keyboards", children: [] },
    { name: "Monitors", children: [{ name: "Gaming monitors", children: [] }] }
  ]
};

function printTree(category: Category, depth: number): void {
  console.log(`${"  ".repeat(depth)}- ${category.name}`);
  for (const child of category.children) {
    printTree(child, depth + 1);
  }
}

printTree(electronics, 0);
```

```text
- Electronics
  - Keyboards
  - Monitors
    - Gaming monitors
```

Every level is checked with the same type, however deep it goes. Menus, comment threads and folder trees all have this shape.

---

## Naming Types

| Rule                                  | Example                                  |
| ------------------------------------- | ---------------------------------------- |
| PascalCase: each word starts with a capital | `OrderItem`, `DigitalProduct`, `ProductId` |
| A noun: what the value **is**         | `Customer`, not `CustomerData` or `ICustomer` |
| Singular for one item                 | `Product`, and `Product[]` for a list    |

You may see interfaces named with an `I` in front, like `IProduct`, in older code and in C# projects. Modern TypeScript code leaves it out.

The syntax is easy to mix up, and both mistakes are syntax errors

interface-equals.ts

```typescript
interface Product = {
  name: string;
}
```

```text
interface-equals.ts:1:19 - error TS1005: '{' expected.

1 interface Product = {
                    ~


Found 1 error in interface-equals.ts:1
```

type-no-equals.ts

```typescript
type Product {
  name: string;
}
```

```text
type-no-equals.ts:1:14 - error TS1005: '=' expected.

1 type Product {
               ~


Found 1 error in type-no-equals.ts:1
```

`type` has `=`, `interface` does not.

---

## Mini Example 1: The Order Summary in Three Files

Session 07's order summary, now with named types shared between files. Make a folder `shop` with three files

shop/types.ts

```typescript
export interface Customer {
  name: string;
  city: string;
}

export interface OrderItem {
  name: string;
  pricePaise: number;
  quantity: number;
}

export interface Order {
  readonly id: number;
  customer: Customer;
  items: OrderItem[];
  couponCode?: string;
}
```

shop/totals.ts

```typescript
import type { OrderItem } from "./types.ts";

export function lineTotal(item: OrderItem): number {
  return item.pricePaise * item.quantity;
}

export function orderTotal(items: readonly OrderItem[]): number {
  return items.reduce((sum, item) => sum + lineTotal(item), 0);
}
```

shop/order-summary.ts

```typescript
import type { Order } from "./types.ts";
import { orderTotal } from "./totals.ts";

function formatRupees(paise: number): string {
  return `Rs. ${(paise / 100).toFixed(2)}`;
}

function orderSummary(order: Order): string {
  const lines: string[] = [`Order ${order.id} for ${order.customer.name} (${order.customer.city})`];
  for (const item of order.items) {
    lines.push(`  ${item.name.padEnd(9)} ${item.quantity} x ${formatRupees(item.pricePaise)}`);
  }
  lines.push(`  Coupon: ${order.couponCode ?? "none"}`);
  lines.push(`  Total: ${formatRupees(orderTotal(order.items))}`);
  return lines.join("\n");
}

const orders: Order[] = [
  {
    id: 1001,
    customer: { name: "Riya", city: "Pune" },
    items: [
      { name: "Keyboard", pricePaise: 149900, quantity: 2 },
      { name: "Mouse", pricePaise: 49900, quantity: 1 }
    ]
  },
  {
    id: 1002,
    customer: { name: "Sara", city: "Mumbai" },
    items: [{ name: "Monitor", pricePaise: 1299900, quantity: 1 }],
    couponCode: "FESTIVE20"
  }
];

for (const order of orders) {
  console.log(orderSummary(order));
}
```

```bash
npx tsc --noEmit
node shop/order-summary.ts
```

```text
Order 1001 for Riya (Pune)
  Keyboard  2 x Rs. 1499.00
  Mouse     1 x Rs. 499.00
  Coupon: none
  Total: Rs. 3497.00
Order 1002 for Sara (Mumbai)
  Monitor   1 x Rs. 12999.00
  Coupon: FESTIVE20
  Total: Rs. 12999.00
```

The same output as Session 07, and much easier to read

| Session 07                                            | Now                                    |
| ----------------------------------------------------- | -------------------------------------- |
| A seven-line object type on the parameter             | `order: Order`                         |
| The item shape written inside the order type          | `OrderItem`, used by two files         |
| Everything in one file                                | Types, calculations and output apart   |

`totals.ts` imports a type with `import type`; `order-summary.ts` imports a type **and** a function, on two lines.

Now the shop starts tracking stock codes, and every item needs one. Add one line to `shop/types.ts`, inside `OrderItem`: `sku: string;`

```text
shop/order-summary.ts:23:7 - error TS2741: Property 'sku' is missing in type '{ name: string; pricePaise: number; quantity: number; }' but required in type 'OrderItem'.

23       { name: "Keyboard", pricePaise: 149900, quantity: 2 },
         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  shop/types.ts:10:3 - 'sku' is declared here.
    10   sku: string;
         ~~~

shop/order-summary.ts:24:7 - error TS2741: Property 'sku' is missing in type '{ name: string; pricePaise: number; quantity: number; }' but required in type 'OrderItem'.

24       { name: "Mouse", pricePaise: 49900, quantity: 1 }
         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  shop/types.ts:10:3 - 'sku' is declared here.
    10   sku: string;
         ~~~

shop/order-summary.ts:30:13 - error TS2741: Property 'sku' is missing in type '{ name: string; pricePaise: number; quantity: number; }' but required in type 'OrderItem'.

30     items: [{ name: "Monitor", pricePaise: 1299900, quantity: 1 }],
               ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  shop/types.ts:10:3 - 'sku' is declared here.
    10   sku: string;
         ~~~


Found 3 errors in the same file, starting at: shop/order-summary.ts:23
```

One change in one place, and `tsc` lists every item that needs a stock code, in another file. `totals.ts` has no errors: it never makes items, it only reads their price and quantity. Add `sku: "KB-01"` and so on to the three items, and everything works again.

![types.ts is shared by totals.ts and order-summary.ts; adding a property to OrderItem lights up every item that is missing it](images/08-type-aliases-and-interfaces/shop-files.gif)

---

## Mini Example 2: A Product Catalog with extends

A shop that sells physical products, which need shipping, and digital products, which are downloaded

catalog.ts

```typescript
interface Product {
  readonly id: string;
  name: string;
  pricePaise: number;
}

interface PhysicalProduct extends Product {
  weightGrams: number;
}

interface DigitalProduct extends Product {
  downloadUrl: string;
}

function listing(product: Product): string {
  return `${product.id}  ${product.name.padEnd(17)} Rs. ${(product.pricePaise / 100).toFixed(2)}`;
}

// Rs. 40 for the first 500 g, and Rs. 20 for every 500 g after that
function shippingPaise(item: PhysicalProduct): number {
  const extraSteps = Math.ceil(Math.max(0, item.weightGrams - 500) / 500);
  return 4000 + extraSteps * 2000;
}

const keyboard: PhysicalProduct = { id: "KB-01", name: "Keyboard", pricePaise: 149900, weightGrams: 900 };
const monitor: PhysicalProduct = { id: "MN-03", name: "Monitor", pricePaise: 1299900, weightGrams: 4200 };
const ebook: DigitalProduct = {
  id: "EB-01",
  name: "TypeScript Notes",
  pricePaise: 29900,
  downloadUrl: "https://example.com/ts-notes.pdf"
};

const catalog: Product[] = [keyboard, monitor, ebook];
for (const product of catalog) {
  console.log(listing(product));
}

console.log(`Shipping for the keyboard: Rs. ${shippingPaise(keyboard) / 100}`);
console.log(`Shipping for the monitor: Rs. ${shippingPaise(monitor) / 100}`);
```

```bash
npx tsc --noEmit
node catalog.ts
```

```text
KB-01  Keyboard          Rs. 1499.00
MN-03  Monitor           Rs. 12999.00
EB-01  TypeScript Notes  Rs. 299.00
Shipping for the keyboard: Rs. 60
Shipping for the monitor: Rs. 200
```

| Code                                  | Why it works                                                        |
| ------------------------------------- | ------------------------------------------------------------------- |
| `const catalog: Product[] = [keyboard, monitor, ebook]` | Both kinds extend `Product`, so both fit a `Product[]` |
| `listing(product: Product)`           | Uses only what every product has                                    |
| `shippingPaise(item: PhysicalProduct)`| Needs `weightGrams`, so it asks for the narrower type               |

Now try to ship the e-book

```typescript
console.log(`Shipping for the e-book: Rs. ${shippingPaise(ebook) / 100}`);
```

```text
catalog.ts:41:59 - error TS2741: Property 'weightGrams' is missing in type 'DigitalProduct' but required in type 'PhysicalProduct'.

41 console.log(`Shipping for the e-book: Rs. ${shippingPaise(ebook) / 100}`);
                                                             ~~~~~

  catalog.ts:8:3 - 'weightGrams' is declared here.
    8   weightGrams: number;
        ~~~~~~~~~~~


Found 1 error in catalog.ts:41
```

The message is short and clear because both types have names: `'DigitalProduct'` and `'PhysicalProduct'`, not two long object shapes. Without TypeScript, `undefined - 500` would give `NaN`, and the e-book's shipping would print `Rs. NaN`.

Going through `catalog` and asking "is this a physical product?" for each item needs narrowing, which is Session 10.

![Physical and digital products both extend Product and appear in the listing; only physical products can go to the shipping function, and the e-book is refused](images/08-type-aliases-and-interfaces/catalog-extends.gif)

---

## Beginner Mistakes

### Mistake 1: Thinking an alias makes a new type

`type Paise = number` and `type Rupees = number` are the same type, so a paise value went into a rupees variable without an error, and the price was shown 100 times too high. Keep the unit in the name: `pricePaise`.

---

### Mistake 2: Writing = after interface

`interface Product = { ... }` gives `error TS1005: '{' expected.` An interface has no `=`.

---

### Mistake 3: Forgetting = after type

`type Product { ... }` gives `error TS1005: '=' expected.` A type alias always has `=`.

---

### Mistake 4: Importing a type without import type

`import { Product } from "./types.ts"` gives `error TS1484`, and Node.js stops with `does not provide an export named 'Product'`. Write `import type { Product }`, or `import { type Product, formatProduct }` when you also import values.

---

### Mistake 5: Two interfaces with the same name by accident

Two `interface Customer` blocks in one file merge, and every customer suddenly needs the properties of both. Give different shapes different names.

---

### Mistake 6: The same type alias name twice

`type Customer` twice gives `error TS2300: Duplicate identifier 'Customer'.` on both lines. Rename one, or combine them with `&`.

---

### Mistake 7: A conflicting property in an intersection

`Product & { pricePaise: string }` makes `pricePaise` `never`, and the error appears only when you build a value: `Type 'number' is not assignable to type 'never'.` Remove the conflict, or use an interface with `extends`, which reports it at once.

---

### Mistake 8: Changing a property's type in an extending interface

`interface FreeSample extends Product { pricePaise: string }` gives `error TS2430: Interface 'FreeSample' incorrectly extends interface 'Product'.` An extending interface may add properties, not change their types.

---

### Mistake 9: Putting all types in the file that uses them

When a type is used by several files, put it in a shared file like `types.ts` and import it. Copies of the same type in different files drift apart.

---

### Mistake 10: Naming types like variables

`orderItem` or `order_item` as a type name makes types and values hard to tell apart. Types use PascalCase: `OrderItem`.

---

### Mistake 11: Using a wide type when a narrow one is needed

`shippingPaise(item: Product)` would not know about `weightGrams`. Ask for exactly what the function needs: `PhysicalProduct` for shipping, `Product` for a listing.

---

## Practice Exercises

### Exercise 1

Rewrite Session 07's `contactLine` with `interface Customer { name: string; phone?: string }`. Point at the parameter in VS Code: what does it show now?

### Exercise 2

Make `type Coordinates = [latitude: number, longitude: number];` and `interface Store { name: string; location: Coordinates }`. Create two stores and print `Pune store at 18.52, 73.85`.

### Exercise 3

Write `interface Employee extends HasId, HasTimestamps { name: string; role: string }` using the interfaces from `user.ts`. Leave out `updatedAt` when you create one, and read the error. Which file and line do the extra lines point to?

### Exercise 4

Write the same `DigitalProduct` twice: once with `interface ... extends`, once with `type ... = Product & { ... }`. Then give both a conflicting `name: number`. Where does each one report the problem?

### Exercise 5

In `merging.ts`, add a third `interface Customer { city: string }`. Fix the `customer` object so that `tsc` is happy. Then change all three to `type` and read the errors.

### Exercise 6

Move the interfaces of Mini Example 2 into `catalog-types.ts`, export them, and import them into `catalog.ts`. Try it once without `type` in the import, and run it with `node` to see the error yourself.

### Exercise 7

Add `interface GiftCard extends Product { valuePaise: number }` to Mini Example 2. Add a gift card to `catalog`. Can it be shipped? What does `tsc` say?

### Exercise 8

Give every `Category` in `categories.ts` an optional `productCount?: number`, and print it next to the name when it is there: `- Keyboards (12)`.

---

## Interview Questions

### What is a type alias?

A name for a type, written `type Name = ...`. It can name any type: an object shape, a primitive, a tuple, an array, a union. It does not create a new type, only a name for an existing one.

### What is an interface?

A named object type, written `interface Name { ... }` without `=`. It describes the properties and methods an object must have, and it can extend other interfaces.

### What is the difference between type and interface?

Both can name object shapes. `type` can also name primitives, tuples, unions and other types; `interface` can only describe objects. Interfaces use `extends` and can merge when declared twice; type aliases use `&` and report a duplicate name as an error.

### When would you use an interface, and when a type alias?

A common rule: interfaces for object shapes, type aliases for everything else, like unions, tuples and function types. The TypeScript handbook suggests using an interface until you need a feature that only `type` has. The most important thing is to be consistent within a project.

### What does extends do on an interface?

It builds a new interface from an existing one: the new interface has all the properties of the old one plus its own. It can extend several interfaces at once, and an incompatible property type is reported as TS2430.

### What is an intersection type?

A type made with `&` that has everything from all its parts, like `Product & { downloadUrl: string }`. If the parts have the same property with incompatible types, that property becomes `never`.

### What is declaration merging?

When two interfaces with the same name are declared in the same scope, TypeScript combines them into one interface with all the properties. It is used to add properties to types from libraries, for example Express's `Request`.

### Does a type alias create a new type?

No. `type Paise = number` is just another name for `number`, so any number fits, including one meant as rupees. Separate "branded" types need extra techniques.

### Why do you need import type?

Types disappear when TypeScript is removed, so an import of only a type must disappear too. With `verbatimModuleSyntax`, TypeScript requires `import type` for type-only imports (TS1484), so that tools like Node.js, which handle one file at a time, know to remove the line.

### Can a type refer to itself?

Yes. An interface or type alias can use its own name in a property, like `children: Category[]`. This describes trees: categories, menus, comment threads.

### Why do named types make error messages better?

Errors and editor hints show the name, like `'PhysicalProduct'`, instead of the full object shape, and the extra lines point to the property inside the named type.

### How should types be named?

In PascalCase, as nouns, singular for one item: `OrderItem`, `Customer`. The `I` prefix, like `IProduct`, is an older style that modern TypeScript code avoids.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [07. Object Types](07-object-types.md) | [Course outline](README.md#course-outline) | [09. Union and Literal Types](09-union-and-literal-types.md) |
