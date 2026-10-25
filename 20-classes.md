# Classes in TypeScript

> **Session 20 of 33** | Part 5: Classes | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [From a JavaScript Class to a TypeScript Class](#from-a-javascript-class-to-a-typescript-class)
- [Declaring Fields](#declaring-fields)
- [Fields Must Get a Value](#fields-must-get-a-value)
- [readonly Fields](#readonly-fields)
- [A Class Is Also a Type](#a-class-is-also-a-type)
- [instanceof with Your Own Classes](#instanceof-with-your-own-classes)
- [private Is Checked Only by tsc](#private-is-checked-only-by-tsc)
- [#private Fields Are Private at Run Time](#private-fields-are-private-at-run-time)
- [Parameter Properties](#parameter-properties)
- [Getters and Setters](#getters-and-setters)
- [Static Members](#static-members)
- [Private Constructors](#private-constructors)
- [Losing this](#losing-this)
- [Generic Classes](#generic-classes)
- [Mini Example 1: A Generic Repository](#mini-example-1-a-generic-repository)
- [Mini Example 2: A Rate Limiter](#mini-example-2-a-rate-limiter)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `20-classes` inside `ts-practice`, with the one-line `tsconfig.json`

20-classes/tsconfig.json

```json
{
  "extends": "../tsconfig.json"
}
```

---

## From a JavaScript Class to a TypeScript Class

You know classes from the [JavaScript course, Session 17](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/17-this-constructors-classes.md): a constructor, methods, `this`, `#` private fields, getters and setters, `static`. All of it works in TypeScript. TypeScript adds types, and a few checks of its own.

Start with a small JavaScript class, saved as a `.ts` file

cart.ts

```typescript
class Cart {
  constructor(customer) {
    this.customer = customer;
    this.items = [];
  }

  add(name, pricePaise) {
    this.items.push({ name, pricePaise });
  }
}

const cart = new Cart("Asha");
cart.add("Keyboard", 249900);
console.log(cart);
```

```bash
npx tsc --noEmit
```

```text
cart.ts:2:15 - error TS7006: Parameter 'customer' implicitly has an 'any' type.

2   constructor(customer) {
                ~~~~~~~~

cart.ts:3:10 - error TS2339: Property 'customer' does not exist on type 'Cart'.

3     this.customer = customer;
           ~~~~~~~~

cart.ts:4:10 - error TS2339: Property 'items' does not exist on type 'Cart'.

4     this.items = [];
           ~~~~~

cart.ts:7:7 - error TS7006: Parameter 'name' implicitly has an 'any' type.

7   add(name, pricePaise) {
        ~~~~

cart.ts:7:13 - error TS7006: Parameter 'pricePaise' implicitly has an 'any' type.

7   add(name, pricePaise) {
              ~~~~~~~~~~

cart.ts:8:10 - error TS2339: Property 'items' does not exist on type 'Cart'.

8     this.items.push({ name, pricePaise });
           ~~~~~


Found 6 errors in the same file, starting at: cart.ts:2
```

```bash
node cart.ts
```

```text
Cart {
  customer: 'Asha',
  items: [ { name: 'Keyboard', pricePaise: 249900 } ]
}
```

Node.js runs it, because it is valid JavaScript and Node.js does not check types (Session 02). `tsc` reports two kinds of errors:

- **TS7006** for the parameters, as in Session 12: every parameter needs a type.
- **TS2339** for `this.customer` and `this.items`. TypeScript wants to know the properties of an object before they are used (Session 07). For a class, that list is the **fields** declared in the class body. An assignment like `this.customer = customer` in the constructor does not declare a field.

Here is the TypeScript version, with a quantity per item and a total

cart.ts

```typescript
interface CartItem {
  name: string;
  pricePaise: number;
  quantity: number;
}

class Cart {
  customer: string;
  items: CartItem[] = [];

  constructor(customer: string) {
    this.customer = customer;
  }

  add(item: CartItem): void {
    this.items.push(item);
  }

  totalPaise(): number {
    let total = 0;
    for (const item of this.items) {
      total += item.pricePaise * item.quantity;
    }
    return total;
  }
}

const cart = new Cart("Asha");
cart.add({ name: "Keyboard", pricePaise: 249900, quantity: 1 });
cart.add({ name: "Mouse", pricePaise: 79900, quantity: 2 });

console.log(cart.totalPaise());
console.log(cart);
```

```bash
npx tsc --noEmit
node cart.ts
```

```text
409700
Cart {
  customer: 'Asha',
  items: [
    { name: 'Keyboard', pricePaise: 249900, quantity: 1 },
    { name: 'Mouse', pricePaise: 79900, quantity: 2 }
  ]
}
```

Reading the class from the top:

- `customer: string;` declares a field and its type. It has no value here; the constructor gives it one.
- `items: CartItem[] = [];` declares a field with a type and a starting value.
- `constructor(customer: string)` types its parameter like any function. A constructor has no return type: it always makes a `Cart`.
- `add(item: CartItem): void` and `totalPaise(): number` are methods, typed like the named functions in Session 12, with a return type.

Field declarations are plain JavaScript (class fields, JavaScript course Session 17). Node.js only removes the types: `customer: string;` runs as `customer;`, and `items: CartItem[] = [];` as `items = [];`.

Now every use of the class is checked. Add three lines at the end

```typescript
cart.add({ name: "Cable", pricePaise: "19900", quantity: 1 });
cart.remove("Mouse");
const empty = new Cart();
```

```text
cart.ts:34:27 - error TS2322: Type 'string' is not assignable to type 'number'.

34 cart.add({ name: "Cable", pricePaise: "19900", quantity: 1 });
                             ~~~~~~~~~~

  cart.ts:3:3 - The expected type comes from property 'pricePaise' which is declared here on type 'CartItem'
    3   pricePaise: number;
        ~~~~~~~~~~

cart.ts:35:6 - error TS2339: Property 'remove' does not exist on type 'Cart'.

35 cart.remove("Mouse");
        ~~~~~~

cart.ts:36:15 - error TS2554: Expected 1 arguments, but got 0.

36 const empty = new Cart();
                 ~~~~~~~~~~

  cart.ts:11:15 - An argument for 'customer' was not provided.
    11   constructor(customer: string) {
                     ~~~~~~~~~~~~~~~~


Found 3 errors in the same file, starting at: cart.ts:34
```

A price as text, a method the class does not have, and a missing constructor argument. The messages are the same ones you know from objects and functions.

![The JavaScript class assigns this.customer and this.items inside the constructor and tsc reports TS2339; in the TypeScript class the same two fields are listed with their types at the top of the class body, and the errors disappear](images/20-classes/fields-first.gif)

---

## Declaring Fields

A field with a starting value does not need a type annotation: TypeScript infers it from the value, as it does for variables (Session 03). Mostly that is what you want, but look at the last two fields here

order-fields.ts

```typescript
class Order {
  itemCount = 0;
  status = "pending";
  readonly currency = "INR";
  createdAt = new Date();
  notes = [];

  addNote(note: string): void {
    this.notes.push(note);
  }
}
```

```text
order-fields.ts:9:21 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'never'.

9     this.notes.push(note);
                      ~~~~


Found 1 error in order-fields.ts:9
```

Hover over each field in your editor to see the type TypeScript chose

| Field                        | Inferred type | Why                                                          |
| ---------------------------- | ------------- | ------------------------------------------------------------ |
| `itemCount = 0`              | `number`      | from the value                                               |
| `status = "pending"`         | `string`      | widened, like `let status = "pending"` (Session 09)          |
| `readonly currency = "INR"`  | `"INR"`       | a readonly field never changes, so it keeps the literal, like `const` |
| `createdAt = new Date()`     | `Date`        | from the value                                               |
| `notes = []`                 | `never[]`     | an empty array has nothing to infer from                     |

`never[]` is an array that accepts nothing, so `push` fails. A variable `const notes = []` grows its type as you push (Session 06), but a field does not. The same happens with an empty `new Map()`: a field `levels = new Map();` becomes `Map<any, any>`, with no error at all, so write `new Map<string, number>()`.

`status` is the other trap. As a `string`, it accepts `order.status = "shiped"` without a word. Give both fields a type

order-fields.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped";

class Order {
  itemCount = 0;
  status: OrderStatus = "pending";
  readonly currency = "INR";
  createdAt = new Date();
  notes: string[] = [];

  addNote(note: string): void {
    this.notes.push(note);
  }
}

const order = new Order();
order.addNote("Gift wrap, please");
order.status = "paid";
console.log(order.status, order.notes, order.currency);
```

```text
paid [ 'Gift wrap, please' ] INR
```

Now the typo is caught. Add it at the end

```typescript
order.status = "shiped";
```

```text
order-fields.ts:19:1 - error TS2820: Type '"shiped"' is not assignable to type 'OrderStatus'. Did you mean '"shipped"'?

19 order.status = "shiped";
   ~~~~~~~~~~~~


Found 1 error in order-fields.ts:19
```

A rule of thumb: let TypeScript infer numbers, dates and other clear values. Write the type for empty arrays and Maps, and for fields that hold one of a few literal values.

![Five class fields with their starting values; the inferred type appears next to each one: number, string, the literal "INR", Date, and an array of never marked in red, because an empty array leaves nothing to infer from](images/20-classes/field-types.gif)

---

## Fields Must Get a Value

`strict` includes `strictPropertyInitialization` (it was in the table in Session 04): every field without a `?` must get a value, either where it is declared or in the constructor

order.ts

```typescript
class Order {
  id: number;
  createdAt: Date;

  constructor(id: number) {
    this.id = id;
  }
}

const order = new Order(1001);
console.log(order.createdAt.toISOString());
```

```text
order.ts:3:3 - error TS2564: Property 'createdAt' has no initializer and is not definitely assigned in the constructor.

3   createdAt: Date;
    ~~~~~~~~~


Found 1 error in order.ts:3
```

`createdAt` is declared, but nothing gives it a value. This is what the check prevents

```bash
node order.ts
```

```text
file:///D:/ts-practice/20-classes/order.ts:11
console.log(order.createdAt.toISOString());
                            ^

TypeError: Cannot read properties of undefined (reading 'toISOString')
    at file:///D:/ts-practice/20-classes/order.ts:11:29
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

There are three real fixes:

- Give the field a starting value: `createdAt = new Date();`
- Set it in the constructor: `this.createdAt = new Date();`
- Make it optional, `createdAt?: Date;`, if it really can be missing. Then every use must check for `undefined` first (Session 11).

You will also see a fourth way, the **definite assignment assertion**, a `!` after the field name

```typescript
  createdAt!: Date;
```

It tells `tsc`: "trust me, this field will have a value". `tsc` is quiet now, and `node order.ts` crashes with exactly the same `TypeError`. Like the `!` after a value in Session 11, it hides the problem instead of fixing it. It is meant for fields that something outside the class fills in, where `tsc` cannot see it. In your own classes, give the field a value

```typescript
  createdAt = new Date();
```

![A checklist runs through the constructor: id gets a value, createdAt does not, and tsc reports TS2564; with createdAt!, tsc is quiet, but at run time the field is still undefined and toISOString crashes](images/20-classes/definite-assignment.gif)

---

## readonly Fields

`readonly`, which you used on object types in Session 07, works on fields too. A readonly field can be set where it is declared and in the constructor, and nowhere else

order-status.ts

```typescript
type OrderStatus = "pending" | "paid" | "shipped";

class Order {
  readonly id: number;
  readonly createdAt = new Date();
  status: OrderStatus = "pending";

  constructor(id: number) {
    this.id = id;
  }

  ship(): void {
    this.status = "shipped";
  }
}

const order = new Order(1001);
order.ship();
console.log(order.id, order.status);
```

```text
1001 shipped
```

Now try to change the readonly fields, once in `ship` and once from outside

```typescript
  ship(): void {
    this.status = "shipped";
    this.createdAt = new Date();
  }
```

```typescript
order.id = 1002;
```

```text
order-status.ts:14:10 - error TS2540: Cannot assign to 'createdAt' because it is a read-only property.

14     this.createdAt = new Date();
            ~~~~~~~~~

order-status.ts:21:7 - error TS2540: Cannot assign to 'id' because it is a read-only property.

21 order.id = 1002;
         ~~


Found 2 errors in the same file, starting at: order-status.ts:14
```

Even the class's own methods cannot change a readonly field after the constructor. Good candidates are ids, creation times and settings passed to the constructor.

Two things from earlier sessions still apply. `readonly` is checked by `tsc` only; nothing is frozen at run time. And it is shallow: a `readonly items: string[]` field cannot be replaced, but `items.push()` still works. Use the type `readonly string[]` (Session 06) to stop that too.

![A timeline of an Order's life: setting id at the declaration and in the constructor is allowed; after the constructor, an assignment in the ship method and one from outside are both stopped with TS2540](images/20-classes/readonly-field.gif)

---

## A Class Is Also a Type

`class Cart` creates two things at once:

- a **value**: the class itself, which you call with `new Cart("Asha")`
- a **type** called `Cart`: the shape of the objects it makes

So you can use `Cart` in annotations. Remove the three wrong lines, and add this to the end of `cart.ts`

```typescript
function checkout(cart: Cart): string {
  return `${cart.customer} pays ${cart.totalPaise()} paise`;
}

const lookalike: Cart = {
  customer: "Ravi",
  items: [],
  add() {},
  totalPaise() {
    return 0;
  }
};

console.log(checkout(cart));
console.log(checkout(lookalike));
console.log(lookalike instanceof Cart);
```

```text
409700
Cart {
  customer: 'Asha',
  items: [
    { name: 'Keyboard', pricePaise: 249900, quantity: 1 },
    { name: 'Mouse', pricePaise: 79900, quantity: 2 }
  ]
}
Asha pays 409700 paise
Ravi pays 0 paise
false
```

`checkout` wants a `Cart`, and `lookalike` is accepted, although no `new Cart` made it. TypeScript compares shapes, not names (structural typing, Session 07). `lookalike` has a `customer`, `items`, `add` and `totalPaise` of the right types, so for `tsc` it is a `Cart`. At run time, `instanceof` asks a different question, "was this object made by this class?", and the answer is `false`.

Most of the time this is useful: a test can pass a small fake object instead of a real one. It matters in two places: `instanceof` checks (next section), and classes with private fields (later in this session).

---

## instanceof with Your Own Classes

Session 10 used `instanceof` with `Date` and `Error`. It narrows your own classes the same way

payments.ts

```typescript
class CardPayment {
  last4: string;

  constructor(last4: string) {
    this.last4 = last4;
  }
}

class UpiPayment {
  upiId: string;

  constructor(upiId: string) {
    this.upiId = upiId;
  }
}

function describePayment(payment: CardPayment | UpiPayment): string {
  if (payment instanceof CardPayment) {
    return `card ending ${payment.last4}`;
  }
  return `UPI ${payment.upiId}`;
}

console.log(describePayment(new CardPayment("4242")));
console.log(describePayment(new UpiPayment("asha@okbank")));
```

```text
card ending 4242
UPI asha@okbank
```

Inside the `if`, `payment` is a `CardPayment`. After the `return`, only `UpiPayment` is left.

Now the gap from the previous section. A plain object with a `last4` has the shape of a `CardPayment`, so `tsc` accepts it. Add a third call

```typescript
console.log(describePayment({ last4: "0000" }));
```

```bash
npx tsc --noEmit
node payments.ts
```

```text
card ending 4242
UPI asha@okbank
UPI undefined
```

`tsc` is quiet. But `instanceof` is `false` for an object that `new` did not make, so the code takes the UPI path, where `upiId` does not exist. `instanceof` is safe when every value really comes from `new`. Data from JSON, a database or a request is always made of plain objects: for that, use a `kind` property and a discriminated union (Session 10).

![Two objects with the same shape reach two gates: both pass the tsc gate, which compares shapes; at the instanceof gate at run time, only the object made with new CardPayment passes, and the plain object falls through to the UPI branch](images/20-classes/lookalike.gif)

---

## private Is Checked Only by tsc

TypeScript has three keywords that say who may use a field or method: `public` (the default, so you rarely write it), `private` (only code inside the class) and `protected` (Session 21)

user.ts

```typescript
class User {
  readonly email: string;
  private passwordHash: string;

  constructor(email: string, passwordHash: string) {
    this.email = email;
    this.passwordHash = passwordHash;
  }

  hasPasswordHash(hash: string): boolean {
    return this.passwordHash === hash;
  }
}

const user = new User("asha@example.com", "hash-8f2a");
console.log(user.hasPasswordHash("hash-8f2a"));
console.log(user.passwordHash);
```

```text
user.ts:17:18 - error TS2341: Property 'passwordHash' is private and only accessible within class 'User'.

17 console.log(user.passwordHash);
                    ~~~~~~~~~~~~


Found 1 error in user.ts:17
```

The method `hasPasswordHash` may use `this.passwordHash`, because it is inside the class. Outside, `tsc` refuses. But `private` is a TypeScript keyword, and like every type it is removed before the code runs: Node.js sees `passwordHash;`, a normal field. Replace the last line with these three

```typescript
console.log(user);
console.log(JSON.stringify(user));
console.log(user["passwordHash"]);
```

```bash
npx tsc --noEmit
node user.ts
```

```text
true
User { email: 'asha@example.com', passwordHash: 'hash-8f2a' }
{"email":"asha@example.com","passwordHash":"hash-8f2a"}
hash-8f2a
```

`tsc` allows all three lines. `console.log` and `JSON.stringify` read every property, and access with brackets and a string is a way around `private` that `tsc` allows on purpose. The hash ends up in the log and in the JSON: an API that sends this user back to the client leaks it, the same leak as in Session 17.

`private` is a rule for other developers, checked by `tsc`. It is not protection at run time.

---

## #private Fields Are Private at Run Time

JavaScript's own private fields, written with `#` (JavaScript course Session 17), are private at run time too, and TypeScript checks them as well

user.ts

```typescript
class User {
  readonly email: string;
  #passwordHash: string;

  constructor(email: string, passwordHash: string) {
    this.email = email;
    this.#passwordHash = passwordHash;
  }

  hasPasswordHash(hash: string): boolean {
    return this.#passwordHash === hash;
  }
}

const user = new User("asha@example.com", "hash-8f2a");
console.log(user.hasPasswordHash("hash-8f2a"));
console.log(user);
console.log(JSON.stringify(user));
```

```text
true
User { email: 'asha@example.com' }
{"email":"asha@example.com"}
```

The hash is not in the log, not in the JSON, and there is no bracket trick: `user["#passwordHash"]` would be a different property that does not exist. Reading it from outside fails twice

```typescript
console.log(user.#passwordHash);
```

```text
user.ts:19:18 - error TS18013: Property '#passwordHash' is not accessible outside class 'User' because it has a private identifier.

19 console.log(user.#passwordHash);
                    ~~~~~~~~~~~~~


Found 1 error in user.ts:19
```

```bash
node user.ts
```

```text
file:///D:/ts-practice/20-classes/user.ts:19
console.log(user.#passwordHash);
                ^

SyntaxError: Private field '#passwordHash' must be declared in an enclosing class
```

`tsc` gives TS18013, and Node.js refuses the whole file with a `SyntaxError` before it runs a single line.

A `#` field also stops look-alikes. A plain object cannot have a `#passwordHash`, so it can never pass as a `User`. Replace the `user.#passwordHash` line with

```typescript
const fake: User = { email: "fake@example.com", hasPasswordHash: () => true };
```

```text
user.ts:19:7 - error TS2741: Property '#passwordHash' is missing in type '{ email: string; hasPasswordHash: () => true; }' but required in type 'User'.

19 const fake: User = { email: "fake@example.com", hasPasswordHash: () => true };
         ~~~~

  user.ts:3:3 - '#passwordHash' is declared here.
    3   #passwordHash: string;
        ~~~~~~~~~~~~~


Found 1 error in user.ts:19
```

A `private` field has the same effect on look-alikes; only the message is different.

|                                | `private`           | `#`                     |
| ------------------------------ | ------------------- | ----------------------- |
| Comes from                     | TypeScript          | JavaScript              |
| Checked by `tsc`               | yes, TS2341         | yes, TS18013            |
| At run time                    | a normal property   | really private          |
| `console.log`, `JSON.stringify` | show it            | do not show it          |
| `obj["name"]`                  | works               | not possible            |

This course uses `#` for private data. You will see `private` in many existing codebases (NestJS uses it a lot); read it as "private by agreement".

![Two User objects side by side, one with private passwordHash and one with #passwordHash; tsc blocks outside access to both, but after Node.js removes the types, the private one is a normal field that JSON.stringify prints, while the # one stays hidden](images/20-classes/private-vs-hash.gif)

---

## Parameter Properties

In many TypeScript projects you will see constructors like this one

product.ts

```typescript
class Product {
  constructor(
    readonly id: number,
    private pricePaise: number
  ) {}
}

console.log(new Product(7, 349900));
```

A modifier (`readonly`, `private`, `public` or `protected`) in front of a constructor parameter makes it a **parameter property**: a short way to declare a field and copy the parameter into it. NestJS code is full of them.

In this course's setup, it is an error

```bash
npx tsc --noEmit
```

```text
product.ts:3:5 - error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

3     readonly id: number,
      ~~~~~~~~~~~~~~~~~~~

product.ts:4:5 - error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

4     private pricePaise: number
      ~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: product.ts:3
```

```bash
node product.ts
```

```text
node:internal/modules/run_main:107
    triggerUncaughtException(
    ^

file:///D:/ts-practice/20-classes/product.ts:3
  constructor(
    readonly id: number,
             ^^^^^^^^^^
    private pricePaise: number

SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript parameter property is not supported in strip-only mode
    at parseTypeScript (node:internal/modules/typescript:68:40)
    at processTypeScriptCode (node:internal/modules/typescript:146:42)
    at stripTypeScriptModuleTypes (node:internal/modules/typescript:215:22)
    at ModuleLoader.<anonymous> (node:internal/modules/esm/translators:680:29)
    at #translate (node:internal/modules/esm/loader:434:20)
    at afterLoad (node:internal/modules/esm/loader:502:29)
    at ModuleLoader.loadAndTranslate (node:internal/modules/esm/loader:507:12)
    at #getOrCreateModuleJobAfterResolve (node:internal/modules/esm/loader:560:36)
    at afterResolve (node:internal/modules/esm/loader:607:52)
    at ModuleLoader.getOrCreateModuleJob (node:internal/modules/esm/loader:613:12) {
  code: 'ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX'
}

Node.js v24.19.0
```

Node.js runs `.ts` files by removing the types and nothing else ("strip-only mode"). A parameter property is more than a type: it needs code that is not in the file, `this.id = id` and `this.pricePaise = pricePaise`. Removing text cannot create it, so Node.js refuses, and `erasableSyntaxOnly` (Session 02) makes `tsc` report it, just like enums in Session 16. Projects that compile everything with `tsc` before running can use parameter properties. In this course, write the fields out

product.ts

```typescript
class Product {
  readonly id: number;
  private pricePaise: number;

  constructor(id: number, pricePaise: number) {
    this.id = id;
    this.pricePaise = pricePaise;
  }
}

console.log(new Product(7, 349900));
```

```text
Product { id: 7, pricePaise: 349900 }
```

It is a few lines longer, and it is exactly the code that the shortcut stands for.

![The short constructor with readonly id and private pricePaise unfolds into the long form: two field declarations plus two this assignments in the constructor; the assignments are code that is not in the file, so type stripping cannot produce them and tsc reports TS1294](images/20-classes/param-properties.gif)

---

## Getters and Setters

Getters and setters (JavaScript course Session 17) are typed like methods: a getter declares what it returns, a setter what it accepts

cart-line.ts

```typescript
class CartLine {
  readonly name: string;
  readonly pricePaise: number;
  #quantity = 1;

  constructor(name: string, pricePaise: number) {
    this.name = name;
    this.pricePaise = pricePaise;
  }

  get quantity(): number {
    return this.#quantity;
  }

  set quantity(value: number) {
    if (!Number.isInteger(value) || value < 1) {
      throw new RangeError(`Quantity must be a whole number from 1, got ${value}`);
    }
    this.#quantity = value;
  }

  get totalPaise(): number {
    return this.pricePaise * this.#quantity;
  }
}

const line = new CartLine("Mouse", 79900);
line.quantity = 3;
console.log(line.quantity, line.totalPaise);
```

```text
3 239700
```

- `quantity` has a getter and a setter, so it can be read and assigned, and every assignment goes through the setter's check.
- `totalPaise` has only a getter, so TypeScript treats it as a readonly property.
- The real value lives in the private field `#quantity`.

Add three wrong uses at the end

```typescript
line.quantity = "4";
line.totalPaise = 0;
console.log(line.totalPaise());
```

```text
cart-line.ts:30:1 - error TS2322: Type 'string' is not assignable to type 'number'.

30 line.quantity = "4";
   ~~~~~~~~~~~~~

cart-line.ts:31:6 - error TS2540: Cannot assign to 'totalPaise' because it is a read-only property.

31 line.totalPaise = 0;
        ~~~~~~~~~~

cart-line.ts:32:18 - error TS6234: This expression is not callable because it is a 'get' accessor. Did you mean to use it without '()'?
  Type 'Number' has no call signatures.

32 console.log(line.totalPaise());
                    ~~~~~~~~~~


Found 3 errors in the same file, starting at: cart-line.ts:30
```

The last message is made for exactly this mistake: a getter is read like a property, without `()`.

The type and the setter check different things. `tsc` knows that `quantity` takes a `number`, but it cannot know that `0` is a bad number. That is the setter's job, at run time. Replace the three lines with

```typescript
line.quantity = 0;
```

```bash
npx tsc --noEmit
node cart-line.ts
```

```text
3 239700
file:///D:/ts-practice/20-classes/cart-line.ts:17
      throw new RangeError(`Quantity must be a whole number from 1, got ${value}`);
            ^

RangeError: Quantity must be a whole number from 1, got 0
    at set quantity (file:///D:/ts-practice/20-classes/cart-line.ts:17:13)
    at file:///D:/ts-practice/20-classes/cart-line.ts:30:15
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

![Three assignments to line.quantity: the text "4" is stopped by tsc before the program runs, 3 passes the setter's check and is stored in #quantity, and 0 passes tsc but is stopped by the setter with a RangeError](images/20-classes/getter-setter.gif)

---

## Static Members

`static` members belong to the class itself, not to the objects it makes (JavaScript course Session 17). A common backend use is a **factory method** that builds an object from another shape, like a database row with snake_case column names

product-row.ts

```typescript
interface ProductRow {
  product_id: number;
  product_name: string;
  price_paise: number;
}

class Product {
  static readonly currency = "INR";

  readonly id: number;
  readonly name: string;
  readonly pricePaise: number;

  constructor(id: number, name: string, pricePaise: number) {
    this.id = id;
    this.name = name;
    this.pricePaise = pricePaise;
  }

  static fromRow(row: ProductRow): Product {
    return new Product(row.product_id, row.product_name, row.price_paise);
  }

  priceLabel(): string {
    return `${Product.currency} ${(this.pricePaise / 100).toFixed(2)}`;
  }
}

const row: ProductRow = { product_id: 7, product_name: "Webcam", price_paise: 349900 };
const webcam = Product.fromRow(row);

console.log(webcam);
console.log(webcam.priceLabel());
```

```text
Product { id: 7, name: 'Webcam', pricePaise: 349900 }
INR 3499.00
```

`Product.fromRow(row)` and `Product.currency` are used through the class name, and `static readonly` makes a constant that belongs to the class. Mix up the two kinds, and `tsc` says which is which

```typescript
webcam.fromRow(row);
console.log(webcam.currency);
Product.priceLabel();
```

```text
product-row.ts:34:8 - error TS2576: Property 'fromRow' does not exist on type 'Product'. Did you mean to access the static member 'Product.fromRow' instead?

34 webcam.fromRow(row);
          ~~~~~~~

product-row.ts:35:20 - error TS2576: Property 'currency' does not exist on type 'Product'. Did you mean to access the static member 'Product.currency' instead?

35 console.log(webcam.currency);
                      ~~~~~~~~

product-row.ts:36:9 - error TS2339: Property 'priceLabel' does not exist on type 'typeof Product'.

36 Product.priceLabel();
           ~~~~~~~~~~


Found 3 errors in the same file, starting at: product-row.ts:34
```

The messages name the two types from "A Class Is Also a Type". `Product` is the type of the objects, which have `priceLabel`. `typeof Product` (the `typeof` from Session 15) is the type of the class itself, which has `fromRow` and `currency`.

![The Product class box holds the static members fromRow and currency; two product objects made by it hold id, name, pricePaise and share priceLabel; Product.fromRow works, while webcam.fromRow is stopped with TS2576](images/20-classes/static-members.gif)

---

## Private Constructors

A constructor can be `private` too. Then only code inside the class can call `new`, and everyone else must go through a static method. This gives you a class whose objects are always valid

email.ts

```typescript
class Email {
  readonly #value: string;

  private constructor(value: string) {
    this.#value = value;
  }

  static parse(text: string): Email | undefined {
    const cleaned = text.trim().toLowerCase();
    if (!cleaned.includes("@")) {
      return undefined;
    }
    return new Email(cleaned);
  }

  get value(): string {
    return this.#value;
  }
}

function sendWelcome(to: Email): void {
  console.log(`Welcome mail to ${to.value}`);
}

const email = Email.parse("  Asha@Example.COM ");
if (email !== undefined) {
  sendWelcome(email);
}
console.log(Email.parse("not-an-email"));
```

```text
Welcome mail to asha@example.com
undefined
```

`Email.parse` is the only way to get an `Email`: it trims and lowercases the text, checks it, and returns `undefined` for bad input. JavaScript has no `#` for constructors, so here TypeScript's `private` is the tool to use. At run time it is a normal constructor; the check is done by `tsc`.

The `#value` field closes the other door: no plain object can pass as an `Email`. Add two lines at the end

```typescript
const typed = new Email("anything");
sendWelcome({ value: "junk" });
```

```text
email.ts:30:15 - error TS2673: Constructor of class 'Email' is private and only accessible within the class declaration.

30 const typed = new Email("anything");
                 ~~~~~~~~~~~~~~~~~~~~~

email.ts:31:13 - error TS2741: Property '#value' is missing in type '{ value: string; }' but required in type 'Email'.

31 sendWelcome({ value: "junk" });
               ~~~~~~~~~~~~~~~~~

  email.ts:2:12 - '#value' is declared here.
    2   readonly #value: string;
                 ~~~~~~


Found 2 errors in the same file, starting at: email.ts:30
```

So `sendWelcome` never needs to check its argument again: every `Email` it can receive went through `parse`.

---

## Losing this

Passing a method as a callback loses its `this` (JavaScript course Session 17). TypeScript does not catch it

controller.ts

```typescript
class OrderController {
  #orders = ["ORD-1001", "ORD-1002"];

  list(): string[] {
    return this.#orders;
  }
}

function runHandler(handler: () => string[]): void {
  console.log(handler());
}

const controller = new OrderController();
console.log(controller.list());
runHandler(controller.list);
```

```bash
npx tsc --noEmit
node controller.ts
```

```text
[ 'ORD-1001', 'ORD-1002' ]
file:///D:/ts-practice/20-classes/controller.ts:5
    return this.#orders;
                ^

TypeError: Cannot read properties of undefined (reading '#orders')
    at list (file:///D:/ts-practice/20-classes/controller.ts:5:17)
    at runHandler (file:///D:/ts-practice/20-classes/controller.ts:10:15)
    at file:///D:/ts-practice/20-classes/controller.ts:15:1
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

`tsc` passes it: `controller.list` is a function that returns `string[]`, which is exactly what `runHandler` asks for. But `runHandler` calls it as `handler()`, with no object before the dot, so `this` is `undefined`. This is a real backend bug: handing `controller.list` to a router does the same thing.

There are two fixes. The first is to wrap the call in an arrow function, so the method is still called on the object. Replace the last line with

```typescript
runHandler(() => controller.list());
```

The second is to make the method an arrow function field. An arrow function takes `this` from the place where it is created, and a field is created for each new object

```typescript
  list = (): string[] => {
    return this.#orders;
  };
```

With either fix, `runHandler(...)` works

```text
[ 'ORD-1001', 'ORD-1002' ]
[ 'ORD-1001', 'ORD-1002' ]
```

The first fix is the usual one. The second gives every object its own copy of the function instead of one shared method, which is fine for objects that exist once, like a controller.

TypeScript does have a tool for `this`: a **`this` parameter**, written first in the parameter list. It only gives `this` a type, and it is removed before the code runs

```typescript
  list(this: OrderController): string[] {
    return this.#orders;
  }
```

With the normal method changed like this, replace the last line of the file with these three

```typescript
const list = controller.list;
list();
runHandler(controller.list);
```

`tsc` reports one error, not two

```text
controller.ts:16:1 - error TS2684: The 'this' context of type 'void' is not assignable to method's 'this' of type 'OrderController'.

16 list();
   ~~~~~~


Found 1 error in controller.ts:16
```

A direct call without an object is caught, but `runHandler(controller.list)` still passes, because `runHandler` asks for a function that does not care about `this`. So the habit that protects you is the arrow function from the first fix. You will still meet `this` parameters in the type definitions of some libraries.

![The method list is taken off the controller object and handed to runHandler; called on its own, its this is undefined and reading #orders crashes; wrapped in an arrow function, the call still happens on the controller and returns the list](images/20-classes/losing-this.gif)

---

## Generic Classes

Classes can have type parameters, like functions (Session 14). You already use one: `Map` is a generic class, and `new Map<string, number>()` fills in its two type parameters. Here is a job queue for any kind of job

queue.ts

```typescript
class Queue<T> {
  #items: T[] = [];

  enqueue(item: T): void {
    this.#items.push(item);
  }

  dequeue(): T | undefined {
    return this.#items.shift();
  }

  get size(): number {
    return this.#items.length;
  }
}

interface EmailJob {
  to: string;
  template: "welcome" | "receipt";
}

const emails = new Queue<EmailJob>();
emails.enqueue({ to: "asha@example.com", template: "welcome" });
emails.enqueue({ to: "ravi@example.com", template: "receipt" });

const job = emails.dequeue();
console.log(job?.to, job?.template, emails.size);

const retries = new Queue<number>();
retries.enqueue(1003);
console.log(retries.dequeue());
```

```text
asha@example.com welcome 1
1003
```

`class Queue<T>` declares the type parameter, and inside the class `T` is used for the field, the parameters and the return types. Each `new Queue<...>()` fills it in, so `emails` takes only `EmailJob`s and `retries` only numbers

```typescript
emails.enqueue({ to: "meera@example.com", template: "invoice" });
retries.enqueue("1004");
```

```text
queue.ts:32:43 - error TS2322: Type '"invoice"' is not assignable to type '"receipt" | "welcome"'.

32 emails.enqueue({ to: "meera@example.com", template: "invoice" });
                                             ~~~~~~~~

  queue.ts:19:3 - The expected type comes from property 'template' which is declared here on type 'EmailJob'
    19   template: "welcome" | "receipt";
         ~~~~~~~~

queue.ts:33:17 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

33 retries.enqueue("1004");
                   ~~~~~~


Found 2 errors in the same file, starting at: queue.ts:32
```

Without a type argument, and with nothing to infer it from, `T` becomes `unknown`. Replace the two lines with

```typescript
const jobs = new Queue();
jobs.enqueue({ to: "asha@example.com" });
const first = jobs.dequeue();
console.log(first.to);
```

```text
queue.ts:35:13 - error TS18046: 'first' is of type 'unknown'.

35 console.log(first.to);
               ~~~~~


Found 1 error in queue.ts:35
```

Anything goes in, and nothing useful comes out. When you create an empty generic object, write the type argument.

![One Queue<T> class with a T slot; new Queue<EmailJob>() fills the slot with EmailJob and new Queue<number>() with number, giving two typed queues; an "invoice" job and the text "1004" are refused at the matching queue](images/20-classes/generic-class.gif)

---

## Mini Example 1: A Generic Repository

Backend code often keeps data access in a **repository**: a class with methods like `insert`, `findById` and `update`, so the rest of the app does not care where the data lives. Later in the course the data lives in MongoDB (Session 32). Here it lives in a `Map`, and one generic class works for every kind of record. It takes two files

repository.ts

```typescript
export class Repository<T extends { id: number }> {
  readonly name: string;
  #rows = new Map<number, T>();

  constructor(name: string) {
    this.name = name;
  }

  insert(row: T): T {
    if (this.#rows.has(row.id)) {
      throw new Error(`${this.name}: id ${row.id} already exists`);
    }
    this.#rows.set(row.id, row);
    return row;
  }

  findById(id: number): T | undefined {
    return this.#rows.get(id);
  }

  findWhere(test: (row: T) => boolean): T[] {
    return [...this.#rows.values()].filter(test);
  }

  update(id: number, changes: Partial<Omit<T, "id">>): T | undefined {
    const current = this.#rows.get(id);
    if (current === undefined) {
      return undefined;
    }
    const updated = { ...current, ...changes };
    this.#rows.set(id, updated);
    return updated;
  }

  get count(): number {
    return this.#rows.size;
  }
}
```

store.ts

```typescript
import { Repository } from "./repository.ts";

interface Product {
  id: number;
  name: string;
  pricePaise: number;
  inStock: boolean;
}

interface Customer {
  id: number;
  email: string;
  city: string;
}

const products = new Repository<Product>("products");
const customers = new Repository<Customer>("customers");

products.insert({ id: 1, name: "Keyboard", pricePaise: 249900, inStock: true });
products.insert({ id: 2, name: "Mouse", pricePaise: 79900, inStock: false });
customers.insert({ id: 1, email: "asha@example.com", city: "Pune" });

products.update(2, { inStock: true, pricePaise: 74900 });

const available = products.findWhere((product) => product.inStock);
console.log(available.map((product) => product.name));
console.log(products.findById(2));
console.log(customers.findById(1)?.city, customers.count);
```

```bash
npx tsc --noEmit
node store.ts
```

```text
[ 'Keyboard', 'Mouse' ]
{ id: 2, name: 'Mouse', pricePaise: 74900, inStock: true }
Pune 1
```

How the types fit together:

- `T extends { id: number }` (a constraint, Session 14): every record needs a numeric `id`, because the repository uses it as the Map key.
- `#rows` is private, so other code can reach the data only through the methods.
- `update` takes `Partial<Omit<T, "id">>` (Session 17): any of the record's fields except `id`.
- `findWhere` takes a test function for `T`, so `(product) => product.inStock` needs no annotation.
- `export class` makes the class available to other files, like the exported types in Session 08.

One class, two repositories, and each one knows its record type. Add these lines at the end of `store.ts`

```typescript
products.update(2, { id: 5 });
products.update(2, { pricePaise: "74900" });
customers.insert({ id: 2, email: "ravi@example.com" });
products.findWhere((product) => product.city === "Pune");
```

```text
store.ts:29:22 - error TS2353: Object literal may only specify known properties, and 'id' does not exist in type 'Partial<Omit<Product, "id">>'.

29 products.update(2, { id: 5 });
                        ~~

store.ts:30:22 - error TS2322: Type 'string' is not assignable to type 'number'.

30 products.update(2, { pricePaise: "74900" });
                        ~~~~~~~~~~

  store.ts:6:3 - The expected type comes from property 'pricePaise' which is declared here on type 'Partial<Omit<Product, "id">>'
    6   pricePaise: number;
        ~~~~~~~~~~

store.ts:31:18 - error TS2741: Property 'city' is missing in type '{ id: number; email: string; }' but required in type 'Customer'.

31 customers.insert({ id: 2, email: "ravi@example.com" });
                    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  store.ts:13:3 - 'city' is declared here.
    13   city: string;
         ~~~~

store.ts:32:41 - error TS2339: Property 'city' does not exist on type 'Product'.

32 products.findWhere((product) => product.city === "Pune");
                                           ~~~~


Found 4 errors in the same file, starting at: store.ts:29
```

A changed id, a price as text, a customer without a city, and a product field that does not exist. Some problems only the code can find. A second product with id 1 has the right type, so `insert` checks for it at run time. Replace the four lines with

```typescript
products.insert({ id: 1, name: "Monitor", pricePaise: 1299900, inStock: true });
```

```text
[ 'Keyboard', 'Mouse' ]
{ id: 2, name: 'Mouse', pricePaise: 74900, inStock: true }
Pune 1
file:///D:/ts-practice/20-classes/repository.ts:11
      throw new Error(`${this.name}: id ${row.id} already exists`);
            ^

Error: products: id 1 already exists
    at Repository.insert (file:///D:/ts-practice/20-classes/repository.ts:11:13)
    at file:///D:/ts-practice/20-classes/store.ts:29:10
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

![One Repository class is used twice: as Repository<Product> and as Repository<Customer>; insert, update and findWhere calls flow into the matching repository, and wrong records, such as a customer without a city or an update that changes the id, are stopped before they reach the Map](images/20-classes/repository.gif)

---

## Mini Example 2: A Rate Limiter

An API should not let one client send thousands of requests a minute. A **rate limiter** counts requests per client and answers `429 Too Many Requests` when a client goes over its limit. This one uses fixed windows: each client may make `limit` requests in `windowMs` milliseconds, counted from its first request

rate-limiter.ts

```typescript
interface RateLimiterOptions {
  limit: number;
  windowMs: number;
}

interface ClientWindow {
  startedAt: number;
  count: number;
}

class RateLimiter {
  readonly limit: number;
  readonly windowMs: number;
  #windows = new Map<string, ClientWindow>();

  constructor(options: RateLimiterOptions) {
    this.limit = options.limit;
    this.windowMs = options.windowMs;
  }

  static perMinute(limit: number): RateLimiter {
    return new RateLimiter({ limit, windowMs: 60_000 });
  }

  #activeWindow(clientId: string, now: number): ClientWindow | undefined {
    const current = this.#windows.get(clientId);
    if (current === undefined || now - current.startedAt >= this.windowMs) {
      return undefined;
    }
    return current;
  }

  allow(clientId: string, now = Date.now()): boolean {
    const current = this.#activeWindow(clientId, now);
    if (current === undefined) {
      this.#windows.set(clientId, { startedAt: now, count: 1 });
      return true;
    }
    if (current.count < this.limit) {
      current.count += 1;
      return true;
    }
    return false;
  }

  remaining(clientId: string, now = Date.now()): number {
    const current = this.#activeWindow(clientId, now);
    return current === undefined ? this.limit : this.limit - current.count;
  }

  get trackedClients(): number {
    return this.#windows.size;
  }
}

const limiter = RateLimiter.perMinute(3);

const requests: [string, number][] = [
  ["10.0.0.1", 0],
  ["10.0.0.1", 10_000],
  ["10.0.0.2", 15_000],
  ["10.0.0.1", 20_000],
  ["10.0.0.1", 30_000],
  ["10.0.0.1", 61_000]
];

for (const [ip, at] of requests) {
  const status = limiter.allow(ip, at) ? 200 : 429;
  console.log(`${at / 1000}s ${ip} -> ${status}, ${limiter.remaining(ip, at)} left`);
}
console.log("clients:", limiter.trackedClients);
```

```bash
npx tsc --noEmit
node rate-limiter.ts
```

```text
0s 10.0.0.1 -> 200, 2 left
10s 10.0.0.1 -> 200, 1 left
15s 10.0.0.2 -> 200, 2 left
20s 10.0.0.1 -> 200, 0 left
30s 10.0.0.1 -> 429, 0 left
61s 10.0.0.1 -> 200, 2 left
clients: 2
```

What each part does:

- `readonly limit` and `readonly windowMs` are the settings, fixed after the constructor.
- `#windows` holds one `ClientWindow` per client, and only the class can touch it.
- `static perMinute(3)` is a factory method with a readable name for the common case.
- `#activeWindow` is a private **method**: like a field, a method whose name starts with `#` can only be called inside the class. It returns the client's current window, or `undefined` when there is none or it is older than `windowMs`.
- `now = Date.now()` is a default parameter (Session 12). The loop passes fake times so the output is always the same; a real server calls `allow(ip)` and gets the current time.
- `trackedClients` is a getter, read like a property.

Request by request: `10.0.0.1` uses its three requests by 20s and is refused at 30s. At 61s, more than 60 seconds after its first request, it starts a new window. `10.0.0.2` has its own window.

The class protects its state. Add these lines at the end

```typescript
limiter.limit = 100;
limiter.#windows.clear();
const custom = new RateLimiter({ limit: 10 });
const other = limiter.perMinute(5);
```

```text
rate-limiter.ts:72:9 - error TS2540: Cannot assign to 'limit' because it is a read-only property.

72 limiter.limit = 100;
           ~~~~~

rate-limiter.ts:73:9 - error TS18013: Property '#windows' is not accessible outside class 'RateLimiter' because it has a private identifier.

73 limiter.#windows.clear();
           ~~~~~~~~

rate-limiter.ts:74:32 - error TS2741: Property 'windowMs' is missing in type '{ limit: number; }' but required in type 'RateLimiterOptions'.

74 const custom = new RateLimiter({ limit: 10 });
                                  ~~~~~~~~~~~~~

  rate-limiter.ts:3:3 - 'windowMs' is declared here.
    3   windowMs: number;
        ~~~~~~~~

rate-limiter.ts:75:23 - error TS2576: Property 'perMinute' does not exist on type 'RateLimiter'. Did you mean to access the static member 'RateLimiter.perMinute' instead?

75 const other = limiter.perMinute(5);
                         ~~~~~~~~~


Found 4 errors in the same file, starting at: rate-limiter.ts:72
```

In a server, a request handler calls the limiter. Remove the four lines, and pass the limiter with an arrow function, as in [Losing this](#losing-this)

```typescript
function handleRequest(ip: string, isAllowed: (ip: string) => boolean): number {
  return isAllowed(ip) ? 200 : 429;
}

console.log(handleRequest("10.0.0.3", (ip) => limiter.allow(ip)));
```

The output gets one more line, `200`. With `handleRequest("10.0.0.3", limiter.allow)`, `tsc` would be quiet again, and Node.js would stop with `TypeError: Cannot read properties of undefined (reading 'RateLimiter')`. The message is unusual because calling a `#` method first checks the object's class. Inside a method, "Cannot read properties of undefined" usually means that `this` was lost.

![A timeline from 0 to 61 seconds for client 10.0.0.1: requests at 0s, 10s and 20s fill a window of three slots, the request at 30s is refused with 429, and at 61s the old window has expired and a new one starts; client 10.0.0.2 has a separate window](images/20-classes/rate-limiter.gif)

---

## Beginner Mistakes

### Mistake 1: Using this.something without declaring the field

`this.items = []` in the constructor gives `error TS2339: Property 'items' does not exist on type 'Cart'.` Declare every field in the class body, with its type.

---

### Mistake 2: An empty array field without a type

`notes = [];` is a `never[]`, and `push` gives `error TS2345: Argument of type 'string' is not assignable to parameter of type 'never'.` Write `notes: string[] = [];`. An empty `new Map()` field is even quieter: it becomes `Map<any, any>`.

---

### Mistake 3: Silencing TS2564 with !

`createdAt!: Date;` makes `tsc` quiet, and the program crashes with `Cannot read properties of undefined`. Give the field a value instead.

---

### Mistake 4: Forgetting this. inside a method

`return items.length;` inside a method gives `error TS2663: Cannot find name 'items'. Did you mean the instance member 'this.items'?` Fields are always reached through `this`.

---

### Mistake 5: Using a field before it gets its value

Fields get their values from top to bottom. `totalPaise = this.subtotalPaise + this.taxPaise;` written above those two fields gives `error TS2729: Property 'subtotalPaise' is used before its initialization.` Plain JavaScript would make it `NaN`. Put the fields it uses first, or make `totalPaise` a getter.

---

### Mistake 6: Expecting private to hide data

`private` is removed before the code runs. `console.log`, `JSON.stringify` and `user["passwordHash"]` all show the value. Use a `#` field for data that must not leak.

---

### Mistake 7: Using parameter properties with erasableSyntaxOnly

`constructor(readonly id: number)` gives `error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.`, and Node.js refuses the file. Declare the field and assign it in the constructor.

---

### Mistake 8: Calling a getter with ()

`line.totalPaise()` gives `error TS6234: This expression is not callable because it is a 'get' accessor. Did you mean to use it without '()'?` Read a getter like a property.

---

### Mistake 9: Mixing up static and instance members

`webcam.fromRow(row)` gives `error TS2576` with "Did you mean to access the static member 'Product.fromRow' instead?", and `Product.priceLabel()` gives `error TS2339` on `typeof Product`. Static members are used through the class name, the others through an object.

---

### Mistake 10: Forgetting new

`const cart = Cart();` gives `error TS2348: Value of type 'typeof Cart' is not callable. Did you mean to include 'new'?` Without `tsc`, Node.js stops with `TypeError: Class constructor Cart cannot be invoked without 'new'`.

---

### Mistake 11: Passing a method as a callback

`runHandler(controller.list)` passes `tsc` and crashes, because `this` is lost. Pass `() => controller.list()` instead.

---

### Mistake 12: Trusting instanceof with plain objects

A plain object with the right shape passes `tsc` as a class type, but `instanceof` is `false` for it. For data from JSON or a database, narrow with a `kind` property.

---

## Practice Exercises

### Exercise 1

Write an `Invoice` class with a readonly `invoiceNumber: string`, a `customer: string`, and `lines`, an array of `{ description: string; amountPaise: number }`. Add `addLine(description: string, amountPaise: number): void` and `totalPaise(): number`. Create an invoice, add two lines and print the total. Then try `invoice.invoiceNumber = "INV-2"` and read the error.

### Exercise 2

Write `class Session` with `userId: number` and `expiresAt: Date`, and a constructor that sets only `userId`. Read error TS2564, then fix it in three ways: a starting value, an assignment in the constructor, and `expiresAt?: Date`. What changes for code that reads `expiresAt` with the third fix?

### Exercise 3

Write a `BankAccount` class twice: once with `private balancePaise`, once with `#balancePaise`. Print an account of each kind with `console.log` and `JSON.stringify`, and explain the difference.

### Exercise 4

This class uses parameter properties: `class Mailer { constructor(private readonly host: string, readonly port: number) {} }`. Rewrite it in the long form, then check it with `npx tsc --noEmit` and run it with `node`.

### Exercise 5

Add a `discountPercent` getter and setter to `CartLine`. The setter accepts only whole numbers from 0 to 50 and throws a `RangeError` otherwise. Change `totalPaise` to apply the discount, and test it with 10, 0, 75 and 2.5.

### Exercise 6

Write a class `Slug` with a private constructor, a `#value` field, a `value` getter, and `static from(title: string): Slug`, which trims and lowercases the title and replaces each space with `-`. Show that `new Slug("x")` and a plain `{ value: "x" }` are both refused where a `Slug` is expected.

### Exercise 7

In Mini Example 1, add `delete(id: number): boolean` and `findAll(): T[]` to `Repository`. Then add an `Order` record with `id`, `customerId` and `status: "pending" | "paid"`, make a repository for it, and use `findWhere` to list the paid orders.

### Exercise 8

In Mini Example 2, add `reset(clientId: string): void` and `static perSecond(limit: number): RateLimiter`. Show that a client that was refused is allowed again after `reset`.

---

## Interview Questions

### What does TypeScript add to JavaScript classes?

Types for fields, parameters and return values, and checks of its own: fields must be declared and must get a value, `readonly`, the `public`, `private` and `protected` keywords, and, in Session 21, abstract classes and `implements`.

### Why must fields be declared in a TypeScript class?

TypeScript needs to know the properties of an object before they are used, and it does not collect them from `this.x = ...` assignments in the constructor. Without a declaration, `this.items` gives TS2339.

### What is strictPropertyInitialization?

A check that is part of `strict`: every field that is not optional must get a value where it is declared or in the constructor. Otherwise `tsc` reports TS2564.

### What does the ! in createdAt!: Date mean?

It is the definite assignment assertion. It tells `tsc` that the field will get a value in a way `tsc` cannot see, and switches off TS2564 for that field. Nothing checks it at run time.

### What is the difference between private and #private?

`private` is a TypeScript check, removed before the code runs: the property still shows up in `console.log`, in `JSON.stringify`, and through `obj["name"]`. A `#` field is a JavaScript private field and stays hidden at run time too.

### What does readonly do on a class field?

The field can only be set where it is declared or in the constructor; later assignments, even inside the class, give TS2540. It is checked by `tsc` only, and it is shallow.

### What are parameter properties, and why can they not be used with Node.js type stripping?

A modifier on a constructor parameter, like `constructor(readonly id: number)`, declares a field and assigns it. That needs code that is not written in the file, so Node.js, which only removes types, cannot run it, and `erasableSyntaxOnly` reports TS1294.

### Is a class a type or a value?

Both. Used with `new`, the class name is a value whose type is `typeof ClassName`. Used in an annotation, it is the type of the objects the class makes.

### Can a plain object be used where a class type is expected?

Yes, if it has the same public shape, because TypeScript compares shapes. Not if the class has `private` or `#` fields. `instanceof` is still `false` for such an object.

### What is a static member, and when is it useful?

A field or method that belongs to the class instead of its objects, like `Product.fromRow(row)`. It is useful for factory methods and constants. Using it through an object gives TS2576.

### Why would you make a constructor private?

To force everyone through a static method that checks or prepares the data, like `Email.parse`, so that every object of the class is valid.

### Does TypeScript catch a lost this?

Mostly not. `runHandler(controller.list)` passes `tsc` and fails at run time. A `this` parameter catches only direct calls without an object. Wrap the call in an arrow function, or make the method an arrow function field.

### What is a generic class?

A class with type parameters, like `class Queue<T>` or the built-in `Map<K, V>`. Each `new Queue<EmailJob>()` fills in the type, so the methods only accept and return that type.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [19. Template Literal Types](19-template-literal-types.md) | [Course outline](README.md#course-outline) | [21. Inheritance, Interfaces and Abstract Classes](21-inheritance-interfaces-abstract-classes.md) |
