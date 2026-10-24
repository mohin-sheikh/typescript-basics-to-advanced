# Object Types

> **Session 7 of 33** | Part 2: Typing Your Data | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [TypeScript Knows the Shape of an Object](#typescript-knows-the-shape-of-an-object)
- [No New Properties Later](#no-new-properties-later)
- [Writing an Object Type](#writing-an-object-type)
- [Object Types for Parameters](#object-types-for-parameters)
- [Objects Inside Objects](#objects-inside-objects)
- [Arrays of Objects](#arrays-of-objects)
- [Optional Properties](#optional-properties)
- [Optional Means Left Out, Not undefined](#optional-means-left-out-not-undefined)
- [readonly Properties](#readonly-properties)
- [Extra Properties in Object Literals](#extra-properties-in-object-literals)
- [Structural Typing: The Shape Is What Counts](#structural-typing-the-shape-is-what-counts)
- [Index Signatures: Objects Used as Lookups](#index-signatures-objects-used-as-lookups)
- [Reading a Property with a Variable Key](#reading-a-property-with-a-variable-key)
- [Destructuring with Types](#destructuring-with-types)
- [Spreading Objects](#spreading-objects)
- [Methods in Object Types](#methods-in-object-types)
- [A Type Is Not a Check: JSON Data](#a-type-is-not-a-check-json-data)
- [Mini Example 1: An Order Summary](#mini-example-1-an-order-summary)
- [Mini Example 2: A Stock Lookup](#mini-example-2-a-stock-lookup)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `07-object-types` inside `ts-practice`, with the one-line `tsconfig.json`

07-object-types/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

Objects are the heart of backend code: a user, a product, an order, a request body. You know them from JavaScript ([JavaScript course, Session 13](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/13-objects.md)). This session is about describing their **shape**: which properties they have, and the type of each one.

---

## TypeScript Knows the Shape of an Object

You have seen it in the errors of Sessions 05 and 06: TypeScript infers the type of every object

inferred-object.ts

```typescript
const product = { name: "Keyboard", pricePaise: 149900, inStock: true };

product.pricePaise = 129900;
product.pricePaise = "1299";
product.color = "black";
```

```text
inferred-object.ts:4:1 - error TS2322: Type 'string' is not assignable to type 'number'.

4 product.pricePaise = "1299";
  ~~~~~~~~~~~~~~~~~~

inferred-object.ts:5:9 - error TS2339: Property 'color' does not exist on type '{ name: string; pricePaise: number; inStock: boolean; }'.

5 product.color = "black";
          ~~~~~


Found 2 errors in the same file, starting at: inferred-object.ts:4
```

The type of `product` is `{ name: string; pricePaise: number; inStock: boolean; }`. A property can get a new value of the **same** type (line 3 is fine), but not a value of another type (line 4).

![An object literal gets its shape as a type: a new price that is a number fits, a price as text and a new color property do not](images/07-object-types/object-shape.gif)

---

## No New Properties Later

Line 5 above shows the biggest change from JavaScript. In JavaScript you add properties whenever you like. In TypeScript, the shape is fixed when the object is made

add-later.ts

```typescript
const user = { name: "Riya" };

user.email = "riya@example.com";
```

```text
add-later.ts:3:6 - error TS2339: Property 'email' does not exist on type '{ name: string; }'.

3 user.email = "riya@example.com";
       ~~~~~


Found 1 error in add-later.ts:3
```

If a property comes later, say so up front: write the type with an **optional** property (explained [below](#optional-properties))

add-later.ts

```typescript
const user: { name: string; email?: string } = { name: "Riya" };

user.email = "riya@example.com";
console.log(user);
```

```text
{ name: 'Riya', email: 'riya@example.com' }
```

---

## Writing an Object Type

An object type lists each property and its type, between `{ }`

object-type.ts

```typescript
const product: { name: string; pricePaise: number; inStock: boolean } = {
  name: "Keyboard",
  pricePaise: 149900,
  inStock: true
};

console.log(`${product.name}: Rs. ${product.pricePaise / 100}`);
```

```text
Keyboard: Rs. 1499
```

The type looks like the object, with types where the values are. A missing property, a wrong type and an extra property are all errors

object-wrong.ts

```typescript
const mouse: { name: string; pricePaise: number } = { name: "Mouse" };
const monitor: { name: string; pricePaise: number } = { name: "Monitor", pricePaise: "12999" };
const webcam: { name: string; pricePaise: number } = { name: "Webcam", pricePaise: 249900, color: "black" };
```

```text
object-wrong.ts:1:7 - error TS2741: Property 'pricePaise' is missing in type '{ name: string; }' but required in type '{ name: string; pricePaise: number; }'.

1 const mouse: { name: string; pricePaise: number } = { name: "Mouse" };
        ~~~~~

  object-wrong.ts:1:30 - 'pricePaise' is declared here.
    1 const mouse: { name: string; pricePaise: number } = { name: "Mouse" };
                                   ~~~~~~~~~~

object-wrong.ts:2:74 - error TS2322: Type 'string' is not assignable to type 'number'.

2 const monitor: { name: string; pricePaise: number } = { name: "Monitor", pricePaise: "12999" };
                                                                           ~~~~~~~~~~

  object-wrong.ts:2:32 - The expected type comes from property 'pricePaise' which is declared here on type '{ name: string; pricePaise: number; }'
    2 const monitor: { name: string; pricePaise: number } = { name: "Monitor", pricePaise: "12999" };
                                     ~~~~~~~~~~

object-wrong.ts:3:92 - error TS2353: Object literal may only specify known properties, and 'color' does not exist in type '{ name: string; pricePaise: number; }'.

3 const webcam: { name: string; pricePaise: number } = { name: "Webcam", pricePaise: 249900, color: "black" };
                                                                                             ~~~~~


Found 3 errors in the same file, starting at: object-wrong.ts:1
```

The three errors from Session 05, now with types you wrote yourself: TS2741 (missing), TS2322 (wrong type) and TS2353 (extra).

Properties can be separated with `;`, with `,`, or just with new lines. All three are the same type

separators.ts

```typescript
const a: { name: string; city: string } = { name: "Riya", city: "Pune" };
const b: { name: string, city: string } = { name: "Sara", city: "Mumbai" };
const c: {
  name: string
  city: string
} = { name: "Aman", city: "Delhi" };

console.log(a, b, c);
```

```text
{ name: 'Riya', city: 'Pune' } { name: 'Sara', city: 'Mumbai' } { name: 'Aman', city: 'Delhi' }
```

Most code uses `;`, and that is what this course does.

When the value is right there, inference is enough: `const product = { ... }` already has a shape. You write an object type when TypeScript cannot see the value: for **parameters**, for empty arrays, and for properties that come later.

---

## Object Types for Parameters

This is where object types matter most. A function cannot guess what it will be given

format-product.ts

```typescript
function formatProduct(product: {
  name: string;
  pricePaise: number;
}): string {
  return `${product.name}: Rs. ${(product.pricePaise / 100).toFixed(2)}`;
}

console.log(formatProduct({ name: "Keyboard", pricePaise: 149900 }));
console.log(formatProduct({ name: "Mouse" }));
```

```text
format-product.ts:9:27 - error TS2741: Property 'pricePaise' is missing in type '{ name: string; }' but required in type '{ name: string; pricePaise: number; }'.

9 console.log(formatProduct({ name: "Mouse" }));
                            ~~~~~~~~~~~~~~~~~

  format-product.ts:3:3 - 'pricePaise' is declared here.
    3   pricePaise: number;
        ~~~~~~~~~~


Found 1 error in format-product.ts:9
```

Without the check, JavaScript would run it and print

```text
Keyboard: Rs. 1499.00
Mouse: Rs. NaN
```

`undefined / 100` is `NaN`, and a customer sees "Rs. NaN" on the invoice. A long object type can be written over several lines, like this one.

![A function expects an object with a name and a price; an object without the price is stopped by tsc instead of printing Rs. NaN](images/07-object-types/param-shape.gif)

---

## Objects Inside Objects

A property can be an object, with its own object type

nested.ts

```typescript
const order: {
  id: number;
  customer: { name: string; city: string };
} = {
  id: 1001,
  customer: { name: "Riya", city: "Pune" }
};

console.log(`Order ${order.id} for ${order.customer.name} in ${order.customer.city}`);

order.customer.city = 411001;
```

```text
nested.ts:11:1 - error TS2322: Type 'number' is not assignable to type 'string'.

11 order.customer.city = 411001;
   ~~~~~~~~~~~~~~~~~~~


Found 1 error in nested.ts:11
```

The check goes all the way down: `411001` is a PIN code, not a city. Without the last line

```text
Order 1001 for Riya in Pune
```

---

## Arrays of Objects

Session 06 left one question open: how to type an **empty** array of objects. Put the object type before `[]`

typed-cart.ts

```typescript
const cart: { name: string; pricePaise: number }[] = [];

cart.push({ name: "Keyboard", pricePaise: 149900 });
cart.push({ name: "Mouse", price: 49900 });
```

```text
typed-cart.ts:4:28 - error TS2353: Object literal may only specify known properties, and 'price' does not exist in type '{ name: string; pricePaise: number; }'.

4 cart.push({ name: "Mouse", price: 49900 });
                             ~~~~~


Found 1 error in typed-cart.ts:4
```

`price` instead of `pricePaise`: a typo that would have put an item without a price into the cart. With `pricePaise: 49900`

```text
[
  { name: 'Keyboard', pricePaise: 149900 },
  { name: 'Mouse', pricePaise: 49900 }
]
```

---

## Optional Properties

A `?` after the name means the property may be missing

optional.ts

```typescript
const customer: { name: string; phone?: string } = { name: "Riya" };

console.log(customer.phone);
console.log(customer.phone.length);
```

```text
optional.ts:4:13 - error TS18048: 'customer.phone' is possibly 'undefined'.

4 console.log(customer.phone.length);
              ~~~~~~~~~~~~~~


Found 1 error in optional.ts:4
```

Reading an optional property gives `string | undefined`. Line 3 is fine, because `console.log` accepts anything. Line 4 would crash

```text
undefined
file:///D:/ts-practice/07-object-types/optional.ts:4
console.log(customer.phone.length);
                           ^

TypeError: Cannot read properties of undefined (reading 'length')
    at file:///D:/ts-practice/07-object-types/optional.ts:4:28
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Check it first, as you did with array items

contact-line.ts

```typescript
function contactLine(customer: { name: string; phone?: string }): string {
  if (customer.phone === undefined) {
    return `${customer.name} (no phone)`;
  }
  return `${customer.name}: ${customer.phone}`;
}

console.log(contactLine({ name: "Riya" }));
console.log(contactLine({ name: "Sara", phone: "98765 43210" }));
```

```text
Riya (no phone)
Sara: 98765 43210
```

After the `if` with `return`, TypeScript knows `customer.phone` is a `string`. A default with `??` works too: `customer.phone ?? "not given"`.

![An optional phone may be missing: reading it gives string or undefined, and a check or a default handles the missing case](images/07-object-types/optional-property.gif)

---

## Optional Means Left Out, Not undefined

optional-undefined.ts

```typescript
const customer: { name: string; phone?: string } = { name: "Aman", phone: undefined };
```

```text
optional-undefined.ts:1:7 - error TS2375: Type '{ name: string; phone: undefined; }' is not assignable to type '{ name: string; phone?: string; }' with 'exactOptionalPropertyTypes: true'. Consider adding 'undefined' to the types of the target's properties.
  Types of property 'phone' are incompatible.
    Type 'undefined' is not assignable to type 'string'.

1 const customer: { name: string; phone?: string } = { name: "Aman", phone: undefined };
        ~~~~~~~~


Found 1 error in optional-undefined.ts:1
```

Your tsconfig has `exactOptionalPropertyTypes` (Session 02, and the same rule as optional tuple items in Session 06). `phone?: string` means "a text, or not there at all". Writing `phone: undefined` is a third thing, a property that exists with no value, and the rule keeps those apart. Leave the property out instead: `{ name: "Aman" }`.

To remove an optional property, use `delete`. A required property cannot be deleted

delete.ts

```typescript
const customer: { name: string; phone?: string } = { name: "Sara", phone: "98765 43210" };

delete customer.phone;
delete customer.name;
```

```text
delete.ts:4:8 - error TS2790: The operand of a 'delete' operator must be optional.

4 delete customer.name;
         ~~~~~~~~~~~~~


Found 1 error in delete.ts:4
```

---

## readonly Properties

`readonly` before a property means it can be set only when the object is made

readonly-order.ts

```typescript
const order: { readonly id: number; status: string } = { id: 1001, status: "placed" };

order.status = "shipped";
order.id = 1002;
```

```text
readonly-order.ts:4:7 - error TS2540: Cannot assign to 'id' because it is a read-only property.

4 order.id = 1002;
        ~~


Found 1 error in readonly-order.ts:4
```

An order's status changes; its id never should. `readonly` is **shallow**: it protects the property, not the object inside it

shallow.ts

```typescript
const order: { readonly id: number; readonly customer: { name: string } } = {
  id: 1001,
  customer: { name: "Riya" }
};

order.customer.name = "Sara";
order.customer = { name: "Aman" };
```

```text
shallow.ts:7:7 - error TS2540: Cannot assign to 'customer' because it is a read-only property.

7 order.customer = { name: "Aman" };
        ~~~~~~~~


Found 1 error in shallow.ts:7
```

Line 6 is allowed: `customer` cannot point to a new object, but the object's own `name` is not `readonly`. To protect it too, write `customer: { readonly name: string }`. Like `readonly` arrays, it exists only for `tsc`.

![The readonly id is locked while the status can change; readonly is shallow, so the customer's name inside can still change](images/07-object-types/readonly-property.gif)

---

## Extra Properties in Object Literals

excess.ts

```typescript
function formatProduct(product: { name: string; pricePaise: number }): string {
  return `${product.name}: Rs. ${(product.pricePaise / 100).toFixed(2)}`;
}

console.log(formatProduct({ name: "Keyboard", pricePaise: 149900, color: "black" }));
```

```text
excess.ts:5:67 - error TS2353: Object literal may only specify known properties, and 'color' does not exist in type '{ name: string; pricePaise: number; }'.

5 console.log(formatProduct({ name: "Keyboard", pricePaise: 149900, color: "black" }));
                                                                    ~~~~~


Found 1 error in excess.ts:5
```

An extra property in an object written right there (an **object literal**) is almost always a mistake: a typo like `price` for `pricePaise`, or a property the function will never read. So TypeScript reports it. Now put the same object in a variable first

structural.ts

```typescript
function formatProduct(product: { name: string; pricePaise: number }): string {
  return `${product.name}: Rs. ${(product.pricePaise / 100).toFixed(2)}`;
}

const keyboard = { name: "Keyboard", pricePaise: 149900, color: "black" };
const cartLine = { name: "Mouse", pricePaise: 49900, quantity: 2 };

console.log(formatProduct(keyboard));
console.log(formatProduct(cartLine));
```

```bash
npx tsc --noEmit
node structural.ts
```

No errors, and

```text
Keyboard: Rs. 1499.00
Mouse: Rs. 499.00
```

`keyboard` has a `color` and `cartLine` has a `quantity`, but both are accepted. A variable may be used in other places where its extra properties matter, so TypeScript only checks that the needed properties are there. This extra check for literals is called the **excess property check**.

![An object literal with an extra color property is rejected, while the same object stored in a variable is accepted because it has everything the function needs](images/07-object-types/excess-vs-variable.gif)

---

## Structural Typing: The Shape Is What Counts

`formatProduct` never asked for "a product". It asked for "something with a `name` that is a `string` and a `pricePaise` that is a `number`". Any object with that shape fits, whatever it is called and wherever it comes from. This is **structural typing**: TypeScript compares shapes, not names.

An object without the needed properties does not fit

structural-missing.ts

```typescript
function formatProduct(product: { name: string; pricePaise: number }): string {
  return `${product.name}: Rs. ${(product.pricePaise / 100).toFixed(2)}`;
}

const banner = { name: "Diwali sale", discountPercent: 20 };

console.log(formatProduct(banner));
```

```text
structural-missing.ts:7:27 - error TS2741: Property 'pricePaise' is missing in type '{ name: string; discountPercent: number; }' but required in type '{ name: string; pricePaise: number; }'.

7 console.log(formatProduct(banner));
                            ~~~~~~

  structural-missing.ts:1:49 - 'pricePaise' is declared here.
    1 function formatProduct(product: { name: string; pricePaise: number }): string {
                                                      ~~~~~~~~~~


Found 1 error in structural-missing.ts:7
```

| Object                                                   | Fits `{ name: string; pricePaise: number }`? |
| -------------------------------------------------------- | -------------------------------------------- |
| `{ name: "Keyboard", pricePaise: 149900 }`               | Yes                                          |
| `keyboard`, with an extra `color`                        | Yes (it is a variable)                       |
| `{ name: "Keyboard", pricePaise: 149900, color: "black" }` written in the call | No: the excess property check |
| `banner`, without `pricePaise`                           | No: TS2741                                   |

Languages like Java and C# compare names instead: a value fits only if it was declared as that type. TypeScript follows JavaScript, where objects are often made on the fly, so it compares shapes.

![Objects of different kinds all fit a function that needs a name and a price, as long as they have both; an object without a price does not fit](images/07-object-types/structural-typing.gif)

---

## Index Signatures: Objects Used as Lookups

Some objects are not records with fixed properties, but lookups: stock by product id, prices by city, settings by name. You do not know the keys in advance. An **index signature** describes them

stock-index.ts

```typescript
const stock: { [productId: string]: number } = {
  "KB-01": 12,
  "MS-02": 0
};

stock["MN-03"] = 5;
stock["WC-04"] = "seven";

const keyboards = stock["KB-01"];
console.log(keyboards.toFixed(0));
```

```text
stock-index.ts:7:1 - error TS2322: Type 'string' is not assignable to type 'number'.

7 stock["WC-04"] = "seven";
  ~~~~~~~~~~~~~~

stock-index.ts:10:13 - error TS18048: 'keyboards' is possibly 'undefined'.

10 console.log(keyboards.toFixed(0));
               ~~~~~~~~~


Found 2 errors in the same file, starting at: stock-index.ts:7
```

Read `{ [productId: string]: number }` as "any text key, and each value is a number". `productId` is only a label for the key, like the labels in tuples.

- **Writing**: any key is allowed (line 6), but the value must be a number (line 7).
- **Reading**: the key may not exist, so `stock["KB-01"]` is `number | undefined`, as with array items (`noUncheckedIndexedAccess`).

With the wrong line removed, and the reads printed directly

stock-index.ts

```typescript
const stock: { [productId: string]: number } = {
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

`"XX-99"` is not in the stock, and the value really is `undefined`. Session 17 shows a shorter way to write this type, `Record<string, number>`.

![A stock lookup accepts any product id as a key with a number value; reading an id that is not there gives undefined](images/07-object-types/index-signature.gif)

---

## Reading a Property with a Variable Key

An object with fixed properties cannot be read with just any text

price-of.ts

```typescript
const prices = { keyboard: 149900, mouse: 49900 };

function priceOf(product: string): number {
  return prices[product];
}
```

```text
price-of.ts:4:10 - error TS7053: Element implicitly has an 'any' type because expression of type 'string' can't be used to index type '{ keyboard: number; mouse: number; }'.
  No index signature with a parameter of type 'string' was found on type '{ keyboard: number; mouse: number; }'.

4   return prices[product];
           ~~~~~~~~~~~~~~~


Found 1 error in price-of.ts:4
```

`product` could be `"monitor"` or `"hello"`, and `prices` has neither. The second line says what is missing: an index signature. Two fixes

- If any key is possible, give `prices` an index signature: `const prices: { [product: string]: number } = { ... }`. Then `prices[product]` is `number | undefined`, and you check it.
- If only `"keyboard"` and `"mouse"` are allowed, the parameter should say so. That needs union types (Session 09) and `keyof` (Session 15).

---

## Destructuring with Types

The type goes **after** the whole pattern

greet.ts

```typescript
function greet({ firstName, city }: { firstName: string; city: string }): string {
  return `Hello ${firstName} from ${city}`;
}

console.log(greet({ firstName: "Riya", city: "Pune" }));
```

```text
Hello Riya from Pune
```

A very common mistake is to put the types inside the pattern

greet-wrong.ts

```typescript
function greet({ firstName: string, city: string }) {
  return `Hello ${firstName} from ${city}`;
}
```

```text
greet-wrong.ts:1:29 - error TS2300: Duplicate identifier 'string'.

1 function greet({ firstName: string, city: string }) {
                              ~~~~~~

greet-wrong.ts:1:29 - error TS7031: Binding element 'string' implicitly has an 'any' type.

1 function greet({ firstName: string, city: string }) {
                              ~~~~~~

greet-wrong.ts:1:43 - error TS2300: Duplicate identifier 'string'.

1 function greet({ firstName: string, city: string }) {
                                            ~~~~~~

greet-wrong.ts:1:43 - error TS7031: Binding element 'string' implicitly has an 'any' type.

1 function greet({ firstName: string, city: string }) {
                                            ~~~~~~

greet-wrong.ts:2:19 - error TS2304: Cannot find name 'firstName'.

2   return `Hello ${firstName} from ${city}`;
                    ~~~~~~~~~

greet-wrong.ts:2:37 - error TS2304: Cannot find name 'city'.

2   return `Hello ${firstName} from ${city}`;
                                      ~~~~


Found 6 errors in the same file, starting at: greet-wrong.ts:1
```

Six errors from one misunderstanding. In JavaScript destructuring, `{ firstName: string }` means "take `firstName` and **rename** it to `string`" ([JavaScript course, Session 14](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/14-destructuring-spread-rest.md)). So there are two variables called `string`, no `firstName` and no `city`. The colon inside a pattern always renames; the type comes after the pattern.

Default values work with optional properties

welcome.ts

```typescript
function welcome({ name, title = "Customer" }: { name: string; title?: string }): string {
  return `Welcome, ${title} ${name}`;
}

console.log(welcome({ name: "Riya" }));
console.log(welcome({ name: "Sara", title: "Dr." }));
```

```text
Welcome, Customer Riya
Welcome, Dr. Sara
```

`title` is optional in the type, and the default fills it in, so inside the function `title` is a `string`.

![Inside a destructuring pattern the colon renames a property; the types belong in a separate object type after the pattern](images/07-object-types/destructure-types.gif)

---

## Spreading Objects

Spread copies properties into a new object, and the types come along

spread.ts

```typescript
const product = { name: "Keyboard", pricePaise: 149900 };

const onSale = { ...product, pricePaise: 129900 };
console.log(onSale);

const broken: { name: string; pricePaise: number } = { ...product, pricePaise: "1299" };
```

```text
spread.ts:6:68 - error TS2322: Type 'string' is not assignable to type 'number'.

6 const broken: { name: string; pricePaise: number } = { ...product, pricePaise: "1299" };
                                                                     ~~~~~~~~~~

  spread.ts:6:31 - The expected type comes from property 'pricePaise' which is declared here on type '{ name: string; pricePaise: number; }'
    6 const broken: { name: string; pricePaise: number } = { ...product, pricePaise: "1299" };
                                    ~~~~~~~~~~


Found 1 error in spread.ts:6
```

`onSale` has the type `{ name: string; pricePaise: number; }`, and without the last line it prints

```text
{ name: 'Keyboard', pricePaise: 129900 }
```

A property written after the spread replaces the copied one, and its type is checked against the type you asked for.

---

## Methods in Object Types

An object type can include a method: its name, `()`, and what it returns

cart-item.ts

```typescript
const cartItem: {
  name: string;
  pricePaise: number;
  quantity: number;
  total(): number;
} = {
  name: "Keyboard",
  pricePaise: 149900,
  quantity: 2,
  total() {
    return this.pricePaise * this.quantity;
  }
};

console.log(cartItem.total());
console.log(cartItem.totl());
```

```text
cart-item.ts:16:22 - error TS2551: Property 'totl' does not exist on type '{ name: string; pricePaise: number; quantity: number; total(): number; }'. Did you mean 'total'?

16 console.log(cartItem.totl());
                        ~~~~

  cart-item.ts:5:3 - 'total' is declared here.
    5   total(): number;
        ~~~~~~~~~~~~~~~~


Found 1 error in cart-item.ts:16
```

Inside `total`, `this` has the type of the object, so `this.pricePaise` and `this.quantity` are numbers. Without the last line, the output is `299800`. Methods with parameters, and the types of functions in general, are Session 12.

---

## A Type Is Not a Check: JSON Data

json-trust.ts

```typescript
const body = '{ "id": 1001, "totalPaise": "49900" }';

const order: { id: number; totalPaise: number } = JSON.parse(body);
console.log(order.totalPaise + 100);
```

```bash
npx tsc --noEmit
node json-trust.ts
```

No errors, and

```text
49900100
```

`JSON.parse` returns `any` (Session 04), and `any` fits into every type. The annotation is a **promise** you make to TypeScript, not a check. The JSON said `"49900"`, a text, and `+ 100` joined texts.

Data from outside the program (request bodies, files, other APIs) can be anything. TypeScript checks your code, not that data. Checking data at run time is Session 26, with Joi. Until then: be suspicious of every type written on the result of `JSON.parse`.

![JSON text goes through JSON.parse as any; the type annotation is trusted without a check, and the text price joins with 100 at run time](images/07-object-types/json-trust.gif)

---

## Mini Example 1: An Order Summary

order-summary.ts

```typescript
function formatRupees(paise: number): string {
  return `Rs. ${(paise / 100).toFixed(2)}`;
}

function lineTotal(item: { pricePaise: number; quantity: number }): number {
  return item.pricePaise * item.quantity;
}

function orderSummary(order: {
  readonly id: number;
  customer: { name: string; city: string };
  items: { name: string; pricePaise: number; quantity: number }[];
  couponCode?: string;
}): string {
  const lines: string[] = [`Order ${order.id} for ${order.customer.name} (${order.customer.city})`];
  let total = 0;
  for (const item of order.items) {
    total += lineTotal(item);
    lines.push(`  ${item.name.padEnd(9)} ${item.quantity} x ${formatRupees(item.pricePaise)}`);
  }
  lines.push(`  Coupon: ${order.couponCode ?? "none"}`);
  lines.push(`  Total: ${formatRupees(total)}`);
  return lines.join("\n");
}

console.log(orderSummary({
  id: 1001,
  customer: { name: "Riya", city: "Pune" },
  items: [
    { name: "Keyboard", pricePaise: 149900, quantity: 2 },
    { name: "Mouse", pricePaise: 49900, quantity: 1 }
  ]
}));

console.log(orderSummary({
  id: 1002,
  customer: { name: "Sara", city: "Mumbai" },
  items: [{ name: "Monitor", pricePaise: 1299900, quantity: 1 }],
  couponCode: "FESTIVE20"
}));
```

```bash
npx tsc --noEmit
node order-summary.ts
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

| Part of the type                    | What it does                                                       |
| ----------------------------------- | ------------------------------------------------------------------ |
| `readonly id: number`               | `orderSummary` cannot change the order id by accident              |
| `customer: { name; city }`          | An object inside the object                                        |
| `items: { ... }[]`                  | An array of objects                                                |
| `couponCode?: string`               | The first order has none; `??` prints `"none"` for it              |
| `lineTotal(item)`                   | `item` has a `name` too, but `lineTotal` only needs `pricePaise` and `quantity`: structural typing |

Add an item without a quantity

```typescript
console.log(orderSummary({
  id: 1003,
  customer: { name: "Aman", city: "Delhi" },
  items: [{ name: "Webcam", pricePaise: 249900 }]
}));
```

```text
order-summary.ts:29:11 - error TS2741: Property 'quantity' is missing in type '{ name: string; pricePaise: number; }' but required in type '{ name: string; pricePaise: number; quantity: number; }'.

29   items: [{ name: "Webcam", pricePaise: 249900 }]
             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  order-summary.ts:12:46 - 'quantity' is declared here.
    12   items: { name: string; pricePaise: number; quantity: number }[];
                                                    ~~~~~~~~


Found 1 error in order-summary.ts:29
```

The error points at the exact item, deep inside the call. (The line number is 29 because this call replaces the two calls at the end of the file.)

Look at the type of `orderSummary`'s parameter: seven lines, and the item shape `{ name: string; pricePaise: number; quantity: number }` is buried inside it. A second function that works with items would have to repeat it, and a change would mean finding every copy. Session 08 solves this: give a type a **name**, write it once, use it everywhere.

![An order with a customer, items and an optional coupon is checked against the nested object type and printed as a summary](images/07-object-types/order-summary.gif)

---

## Mini Example 2: A Stock Lookup

stock.ts

```typescript
const stock: { [productId: string]: number } = {
  "KB-01": 12,
  "MS-02": 0,
  "MN-03": 3
};

function reserve(productId: string, quantity: number): string {
  const available = stock[productId];
  if (available === undefined) {
    return `${productId}: unknown product`;
  }
  if (available < quantity) {
    return `${productId}: only ${available} left`;
  }
  stock[productId] = available - quantity;
  return `${productId}: reserved ${quantity}, ${available - quantity} left`;
}

console.log(reserve("KB-01", 2));
console.log(reserve("MS-02", 1));
console.log(reserve("MN-03", 5));
console.log(reserve("XX-99", 1));
console.log(stock);
```

```bash
npx tsc --noEmit
node stock.ts
```

```text
KB-01: reserved 2, 10 left
MS-02: only 0 left
MN-03: only 3 left
XX-99: unknown product
{ 'KB-01': 10, 'MS-02': 0, 'MN-03': 3 }
```

Remove the `if (available === undefined)` block, and `tsc` finds every place that would break

```text
stock.ts:9:7 - error TS18048: 'available' is possibly 'undefined'.

9   if (available < quantity) {
        ~~~~~~~~~

stock.ts:12:22 - error TS18048: 'available' is possibly 'undefined'.

12   stock[productId] = available - quantity;
                        ~~~~~~~~~

stock.ts:13:49 - error TS18048: 'available' is possibly 'undefined'.

13   return `${productId}: reserved ${quantity}, ${available - quantity} left`;
                                                   ~~~~~~~~~


Found 3 errors in the same file, starting at: stock.ts:9
```

Without TypeScript, `reserve("XX-99", 1)` would compare `undefined < 1` (which is `false`) and then store `undefined - 1`, which is `NaN`, as the stock of a product that does not exist. With the check, an unknown id gets a clear message, and the stock stays clean.

![Reserving stock: a known product is reduced, an empty one is refused, and an unknown id is caught by the undefined check](images/07-object-types/stock-reserve.gif)

---

## Beginner Mistakes

### Mistake 1: Adding a property later

`user.email = "..."` on `{ name: "Riya" }` gives `error TS2339: Property 'email' does not exist on type '{ name: string; }'.` Declare it up front as optional: `{ name: string; email?: string }`.

---

### Mistake 2: Types inside a destructuring pattern

`function greet({ firstName: string, city: string })` gives six errors: the colon renames. Write `function greet({ firstName, city }: { firstName: string; city: string })`.

---

### Mistake 3: Reading an optional property without a check

`customer.phone.length` gives `error TS18048: 'customer.phone' is possibly 'undefined'.` Check it, or give a default with `??`.

---

### Mistake 4: Writing undefined into an optional property

`{ name: "Aman", phone: undefined }` gives TS2375 with `exactOptionalPropertyTypes`. Leave the property out.

---

### Mistake 5: Thinking readonly protects everything inside

`readonly customer` stops `order.customer = ...`, but `order.customer.name = "Sara"` is still allowed. Mark the inner properties `readonly` too when they must not change.

---

### Mistake 6: Expecting an error for extra properties in a variable

`formatProduct(keyboard)` is fine even though `keyboard` has a `color`. Only object literals get the excess property check. That is structural typing, not a bug.

---

### Mistake 7: Trusting a type written on JSON.parse

`const order: { totalPaise: number } = JSON.parse(body)` is never checked, and a text price slipped through and printed `49900100`. Data from outside must be checked at run time (Session 26).

---

### Mistake 8: Reading an object with a string variable

`prices[product]` with `product: string` gives `error TS7053`. Use an index signature for real lookups, or limit the keys (Sessions 09 and 15).

---

### Mistake 9: Forgetting that lookups can miss

`stock["KB-01"]` is `number | undefined` with an index signature. Check the value before you do math with it, as `reserve` does.

---

### Mistake 10: Deleting a required property

`delete customer.name` gives `error TS2790: The operand of a 'delete' operator must be optional.` Only optional properties can be removed.

---

### Mistake 11: Using object or {} as a type

`object` and `{}` say nothing about the properties (Session 04), so reading `config.port` fails. Write the shape: `{ port: number }`.

---

### Mistake 12: Writing a type for every object

`const product: { name: string; pricePaise: number } = { name: "Keyboard", pricePaise: 149900 }` repeats what TypeScript already knows. Write object types for parameters, empty arrays and properties that come later; let inference do the rest.

---

## Practice Exercises

### Exercise 1

Write `function describeUser(user: { name: string; age: number; city: string }): string` that returns a line like `Riya, 25, from Pune`. Call it once correctly, once with `age: "25"`, and once without `city`. Read both errors.

### Exercise 2

Add an optional `email?: string` to the user. Change `describeUser` to add ` (riya@example.com)` when there is an email. Test it with and without one.

### Exercise 3

Make `const settings: { readonly appName: string; port: number } = { appName: "shop-api", port: 3000 };`. Change the port, then try to change the app name. Which line is an error, and what does it say?

### Exercise 4

In `structural.ts`, add a third object: `const service = { name: "Gift wrap", pricePaise: 4900, durationMinutes: 5 };`. Does `formatProduct(service)` work? Now pass the same object written directly in the call. What changes, and why?

### Exercise 5

Make a lookup of city delivery days: `const deliveryDays: { [city: string]: number } = { Pune: 2, Mumbai: 1, Delhi: 4 };`. Write `function deliveryMessage(city: string): string` that returns `Delivery to Pune in 2 days`, or `We do not deliver to Goa yet` for a city that is not in the lookup.

### Exercise 6

Rewrite `contactLine` from [Optional Properties](#optional-properties) with destructuring and a default: `({ name, phone = "not given" }: { name: string; phone?: string })`. Is the `if` still needed?

### Exercise 7

In Mini Example 1, apply the coupon: `FESTIVE20` takes 20% off the total, any other code nothing. Print the discount as its own line. Which property tells you whether there is a coupon, and what is its type inside `orderSummary`?

### Exercise 8

In Mini Example 2, add `function restock(productId: string, quantity: number): string`. It adds to an existing product, or adds a new product with that quantity. Then call `reserve("XX-99", 1)` again after `restock("XX-99", 5)`.

---

## Interview Questions

### How do you describe the shape of an object in TypeScript?

With an object type: the property names and their types between braces, like `{ name: string; pricePaise: number }`. It can be written on a variable, a parameter or a return type, and TypeScript infers it for object literals.

### Can you add a property to an object after it is created?

Not if the property is not part of its type: TypeScript reports TS2339. The property must be declared up front, usually as optional with `?`.

### What is an optional property?

A property marked with `?`, like `phone?: string`. It may be missing, so reading it gives `string | undefined`, and it must be checked before use.

### What does exactOptionalPropertyTypes change?

It separates "missing" from "set to undefined". With it, `phone?: string` accepts a text or no property at all, but not `phone: undefined`, unless `undefined` is added to the type.

### What does readonly do on a property?

It allows the property to be set only when the object is created. Assigning it later is an error (TS2540). It is shallow and exists only at compile time.

### What is structural typing?

TypeScript decides whether a value fits a type by comparing shapes, not names. Any object with the required properties, of the right types, fits, even if it has more properties or was never declared as that type.

### What is the excess property check?

An extra check for object literals: a property that the target type does not have is reported as an error (TS2353), because it is probably a typo. Objects stored in variables first are not checked this way.

### What is an index signature?

A type for objects used as lookups, with keys that are not known in advance: `{ [key: string]: number }`. Any key can be written, and reading a key gives the value type, or `undefined` with `noUncheckedIndexedAccess`.

### Why does obj[key] fail when key is a string?

Because the object has only specific properties, and a `string` could be any text. TypeScript reports TS7053. The fix is an index signature, or a key type that lists only the real property names.

### How do you add types to a destructured parameter?

After the whole pattern: `({ name, city }: { name: string; city: string })`. A colon inside the pattern renames the property, it does not give a type.

### Is a type annotation on JSON.parse checked?

No. `JSON.parse` returns `any`, which fits every type, so the annotation is trusted without a check. Data from outside needs runtime validation, for example with Joi.

### What is the difference between object, {} and a specific object type?

`object` means any non-primitive value and `{}` almost any value; neither describes properties. A specific object type like `{ port: number }` lists the properties, so they can be read and checked.

### When should you write an object type, and when let TypeScript infer it?

Let TypeScript infer it when the value is right there. Write it when TypeScript cannot see the value: function parameters, empty arrays, properties that come later, and data you want to describe once and reuse.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [06. Arrays and Tuples](06-arrays-and-tuples.md) | [Course outline](README.md#course-outline) | [08. Type Aliases and Interfaces](08-type-aliases-and-interfaces.md) |
