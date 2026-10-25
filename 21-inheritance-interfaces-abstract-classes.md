# Inheritance, Interfaces and Abstract Classes

> **Session 21 of 33** | Part 5: Classes | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [extends with Types](#extends-with-types)
- [super Comes First](#super-comes-first)
- [A Child Is Also a Parent](#a-child-is-also-a-parent)
- [Overriding Methods with override](#overriding-methods-with-override)
- [protected](#protected)
- [implements: Promising a Shape](#implements-promising-a-shape)
- [Abstract Classes](#abstract-classes)
- [Abstract Class or Interface?](#abstract-class-or-interface)
- [Typed Error Classes](#typed-error-classes)
- [Mini Example 1: Notification Senders](#mini-example-1-notification-senders)
- [Mini Example 2: Typed API Errors](#mini-example-2-typed-api-errors)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `21-inheritance` inside `ts-practice`. This time its `tsconfig.json` turns on one extra check, which [Overriding Methods with override](#overriding-methods-with-override) explains

21-inheritance/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "noImplicitOverride": true
  }
}
```

Options under `compilerOptions` are added to the ones from `../tsconfig.json`, for this folder only.

---

## extends with Types

`extends` (JavaScript course, [Session 17](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/17-this-constructors-classes.md)) works the same in TypeScript. The child class gets the parent's fields and methods, **with their types**

users.ts

```typescript
class User {
  readonly name: string;
  readonly email: string;

  constructor(name: string, email: string) {
    this.name = name;
    this.email = email;
  }

  describe(): string {
    return `${this.name} <${this.email}>`;
  }

  canDelete(): boolean {
    return false;
  }
}

class Admin extends User {
  readonly permissions: string[];

  constructor(name: string, email: string, permissions: string[]) {
    super(name, email);
    this.permissions = permissions;
  }

  hasPermission(permission: string): boolean {
    return this.permissions.includes(permission);
  }
}

const riya = new User("Riya", "riya@example.com");
const sara = new Admin("Sara", "sara@example.com", ["users", "orders"]);

console.log(riya.describe(), riya.canDelete());
console.log(sara.describe(), sara.hasPermission("orders"));
```

```bash
npx tsc --noEmit
node users.ts
```

```text
Riya <riya@example.com> false
Sara <sara@example.com> true
```

`sara` has everything a `User` has (`name`, `email`, `describe`, `canDelete`) plus what `Admin` adds (`permissions`, `hasPermission`). `super(name, email)` calls the parent's constructor, and `tsc` checks its arguments against that constructor's parameters.

The extras belong to `Admin` only, and the `Admin` constructor has its own parameter list. Add two lines at the end

```typescript
riya.hasPermission("orders");
const tom = new Admin("Tom", "tom@example.com");
```

```text
users.ts:37:6 - error TS2339: Property 'hasPermission' does not exist on type 'User'.

37 riya.hasPermission("orders");
        ~~~~~~~~~~~~~

users.ts:38:13 - error TS2554: Expected 3 arguments, but got 2.

38 const tom = new Admin("Tom", "tom@example.com");
               ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  users.ts:22:44 - An argument for 'permissions' was not provided.
    22   constructor(name: string, email: string, permissions: string[]) {
                                                  ~~~~~~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: users.ts:37
```

![The Admin type is drawn as the User block with an extra block on top: name, email, describe and canDelete come from User, and permissions and hasPermission are added by Admin; riya.hasPermission is refused because a User has only the lower block](images/21-inheritance-interfaces-abstract-classes/extends-types.gif)

---

## super Comes First

In a child class, the parent's constructor must run before the child touches `this`. JavaScript enforces it at run time, and `tsc` checks it before. Remove the two wrong lines, then swap the two lines in the `Admin` constructor

```typescript
  constructor(name: string, email: string, permissions: string[]) {
    this.permissions = permissions;
    super(name, email);
  }
```

```text
users.ts:23:5 - error TS17009: 'super' must be called before accessing 'this' in the constructor of a derived class.

23     this.permissions = permissions;
       ~~~~


Found 1 error in users.ts:23
```

```bash
node users.ts
```

```text
file:///D:/ts-practice/21-inheritance/users.ts:23
    this.permissions = permissions;
    ^

ReferenceError: Must call super constructor in derived class before accessing 'this' or returning from derived constructor
    at new Admin (file:///D:/ts-practice/21-inheritance/users.ts:23:5)
    at file:///D:/ts-practice/21-inheritance/users.ts:33:14
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

Without the `super` line at all, `tsc` adds `error TS2377: Constructors for derived classes must contain a 'super' call.` A child class that has no constructor of its own does not need one: JavaScript then calls the parent's constructor with the same arguments.

![Building an Admin goes in two steps: super runs the User constructor, which fills in name and email, and only then the Admin constructor may use this to add permissions; touching this before super is stopped with TS17009](images/21-inheritance-interfaces-abstract-classes/super-first.gif)

---

## A Child Is Also a Parent

An `Admin` has everything a `User` has, so it can be used wherever a `User` is expected. Put the constructor back in the right order and add this at the end

```typescript
const team: User[] = [riya, sara];

for (const member of team) {
  console.log(member.describe(), member.canDelete());
  if (member instanceof Admin) {
    console.log("  permissions:", member.permissions.join(", "));
  }
}
```

```text
Riya <riya@example.com> false
Sara <sara@example.com> true
Riya <riya@example.com> false
Sara <sara@example.com> false
  permissions: users, orders
```

Inside the loop, `member` is a `User`. Only the `User` members can be used, until `instanceof Admin` narrows it (Session 10, and your own classes in Session 20). The other direction does not work: not every `User` is an `Admin`. Replace the `if` block with a direct use, and add one more line at the end

```typescript
  console.log("  permissions:", member.permissions.join(", "));
}

const boss: Admin = riya;
```

```text
users.ts:42:40 - error TS2339: Property 'permissions' does not exist on type 'User'.

42   console.log("  permissions:", member.permissions.join(", "));
                                          ~~~~~~~~~~~

users.ts:45:7 - error TS2739: Type 'User' is missing the following properties from type 'Admin': permissions, hasPermission

45 const boss: Admin = riya;
         ~~~~


Found 2 errors in the same file, starting at: users.ts:42
```

Notice in the output above that Sara's `canDelete()` still says `false`: an admin should be allowed to delete. That needs an override.

![A list typed User[] holds riya and sara; both answer describe and canDelete; reaching for permissions on the list's items is blocked, until instanceof Admin opens the extra Admin block for sara only](images/21-inheritance-interfaces-abstract-classes/child-is-parent.gif)

---

## Overriding Methods with override

A child **overrides** a method by writing its own version with the same name. In TypeScript, you mark it with the keyword `override`. Put the `if` block back, remove the `boss` line, and add two methods to `Admin`, after `hasPermission`

```typescript
  override canDelete(): boolean {
    return true;
  }

  override describe(): string {
    return `${super.describe()} [admin]`;
  }
```

```text
Riya <riya@example.com> false
Sara <sara@example.com> [admin] true
Riya <riya@example.com> false
Sara <sara@example.com> [admin] true
  permissions: users, orders
```

The loop did not change, but Sara's answers did: each object runs its own class's version. `super.describe()` calls the parent's version from inside the override.

`override` says: "this method replaces one in the parent". `tsc` checks that the parent really has it. That is what the folder's `noImplicitOverride` option is for: with it, **every** override must say `override`. Make two typical mistakes, a missing `override` and a misspelled name

```typescript
  canDelete(): boolean {
    return true;
  }

  override describ(): string {
    return `${super.describe()} [admin]`;
  }
```

```text
users.ts:31:3 - error TS4114: This member must have an 'override' modifier because it overrides a member in the base class 'User'.

31   canDelete(): boolean {
     ~~~~~~~~~

users.ts:35:12 - error TS4117: This member cannot have an 'override' modifier because it is not declared in the base class 'User'. Did you mean 'describe'?

35   override describ(): string {
              ~~~~~~~


Found 2 errors in the same file, starting at: users.ts:31
```

Without the option and without `override`, a typo like `canDelte()` is simply a new method. `tsc` is quiet, and `sara.canDelete()` runs the parent's version and returns `false`. With `override` on every replaced method, both mistakes are caught. Like other modifiers, `override` is removed before the code runs.

An override must also fit the parent's version: same kind of parameters, and a return type that can be used where the parent's could. Change `canDelete` to return a text

```typescript
  override canDelete(): string {
    return "yes";
  }
```

```text
users.ts:31:12 - error TS2416: Property 'canDelete' in type 'Admin' is not assignable to the same property in base type 'User'.
  Type '() => string' is not assignable to type '() => boolean'.
    Type 'string' is not assignable to type 'boolean'.

31   override canDelete(): string {
              ~~~~~~~~~

users.ts:46:29 - error TS2322: Type 'Admin' is not assignable to type 'User'.
  The types returned by 'canDelete()' are incompatible between these types.
    Type 'string' is not assignable to type 'boolean'.

46 const team: User[] = [riya, sara];
                               ~~~~


Found 2 errors in the same file, starting at: users.ts:31
```

The second error shows why this matters: code that uses `User`s expects `canDelete()` to return a boolean. If `Admin` changed that, an admin could no longer be used as a `User`.

![Three methods in Admin are checked against User: override describe finds describe in the parent and passes; override describ finds nothing and gets TS4117 with a suggestion; canDelete without the keyword replaces a parent method silently and gets TS4114](images/21-inheritance-interfaces-abstract-classes/override-check.gif)

---

## protected

Session 20 had `private` and `#`: only the class itself can use them, and that includes **not** its child classes. The third keyword, `protected`, opens a member to child classes, but still not to outside code

accounts.ts

```typescript
class Account {
  readonly owner: string;
  protected balancePaise = 0;

  constructor(owner: string) {
    this.owner = owner;
  }

  deposit(amountPaise: number): void {
    this.balancePaise += amountPaise;
  }

  get balance(): string {
    return `${this.owner}: ${this.balancePaise} paise`;
  }
}

class CreditAccount extends Account {
  readonly limitPaise: number;

  constructor(owner: string, limitPaise: number) {
    super(owner);
    this.limitPaise = limitPaise;
  }

  withdraw(amountPaise: number): boolean {
    if (amountPaise > this.balancePaise + this.limitPaise) {
      return false;
    }
    this.balancePaise -= amountPaise;
    return true;
  }
}

const account = new CreditAccount("Riya", 100000);
account.deposit(50000);
console.log(account.withdraw(120000), account.balance);
console.log(account.withdraw(50000), account.balance);
```

```text
true Riya: -70000 paise
false Riya: -70000 paise
```

`CreditAccount` may go below zero, down to its limit, so it needs to read and change the balance. With `private balancePaise` in `Account`, the child's `this.balancePaise` would give `error TS2341`; with a `#balancePaise` field, the child's `this.#balancePaise` would give `error TS18013`. With `protected`, the child may use it, and outside code may not

```typescript
account.balancePaise = 99999999;
```

```text
accounts.ts:39:9 - error TS2445: Property 'balancePaise' is protected and only accessible within class 'Account' and its subclasses.

39 account.balancePaise = 99999999;
           ~~~~~~~~~~~~


Found 1 error in accounts.ts:39
```

| Who can use it        | In the class | In a child class | Outside code | At run time      |
| --------------------- | ------------ | ---------------- | ------------ | ---------------- |
| `public` (default)    | yes          | yes              | yes          | normal property  |
| `protected`           | yes          | yes              | TS2445       | normal property  |
| `private`             | yes          | TS2341           | TS2341       | normal property  |
| `#name`               | yes          | TS18013          | TS18013      | really private   |

Like `private`, `protected` is checked by `tsc` only: `console.log(account)` still prints `balancePaise: -70000`. Use it for the few things child classes really need, and keep the rest `#`.

![A table of four members with the same value: public, protected, private and #; three doors, the class itself, a child class and outside code, light up green or red: protected opens for the child class only, private and # only for the class itself](images/21-inheritance-interfaces-abstract-classes/access-levels.gif)

---

## implements: Promising a Shape

A class can promise to have the shape of an interface, with `implements`. The interface describes **what** something can do; each class decides **how**. Here a payment gateway is an interface, with a fake one for tests and a real-looking one for the shop

payments.ts

```typescript
interface ChargeResult {
  ok: boolean;
  reference: string;
}

interface PaymentGateway {
  readonly name: string;
  charge(amountPaise: number, token: string): ChargeResult;
}

class FakeGateway implements PaymentGateway {
  readonly name = "fake";
  readonly charged: number[] = [];

  charge(amountPaise: number, token: string): ChargeResult {
    this.charged.push(amountPaise);
    return { ok: token !== "tok_declined", reference: `fake-${this.charged.length}` };
  }
}

class CardGateway implements PaymentGateway {
  readonly name = "card";
  #nextReference = 5001;

  charge(amountPaise: number, token: string): ChargeResult {
    console.log(`  sending ${amountPaise} paise with ${token} to the card network`);
    const reference = `card-${this.#nextReference}`;
    this.#nextReference += 1;
    return { ok: true, reference };
  }
}

class OrderService {
  readonly #gateway: PaymentGateway;

  constructor(gateway: PaymentGateway) {
    this.#gateway = gateway;
  }

  checkout(orderId: number, amountPaise: number, token: string): string {
    const result = this.#gateway.charge(amountPaise, token);
    if (!result.ok) {
      return `Order ${orderId}: payment declined`;
    }
    return `Order ${orderId}: paid with ${this.#gateway.name} (${result.reference})`;
  }
}

const fake = new FakeGateway();
const testOrders = new OrderService(fake);
console.log(testOrders.checkout(1001, 49900, "tok_ok"));
console.log(testOrders.checkout(1002, 129900, "tok_declined"));
console.log(fake.charged);

const liveOrders = new OrderService(new CardGateway());
console.log(liveOrders.checkout(1003, 249900, "tok_live"));
```

```text
Order 1001: paid with fake (fake-1)
Order 1002: payment declined
[ 49900, 129900 ]
  sending 249900 paise with tok_live to the card network
Order 1003: paid with card (card-5001)
```

`OrderService` asks for a `PaymentGateway`, not for one particular class. It works with any object of that shape: the fake one in tests, the real one in production. Handing a class the objects it depends on through its constructor is called **dependency injection**, and it is the core idea of NestJS. Each class may also have extras, like `charged` on `FakeGateway`.

`implements` makes `tsc` check the class itself, right where it is written. Add two broken gateways at the end

```typescript
class BrokenGateway implements PaymentGateway {
  readonly name = "broken";

  charges(amountPaise: number): ChargeResult {
    return { ok: true, reference: `broken-${amountPaise}` };
  }
}

class QuickGateway implements PaymentGateway {
  readonly name = "quick";

  charge(amountPaise, token) {
    return { ok: true, reference: `quick-${token}` };
  }
}
```

```text
payments.ts:58:7 - error TS2420: Class 'BrokenGateway' incorrectly implements interface 'PaymentGateway'.
  Property 'charge' is missing in type 'BrokenGateway' but required in type 'PaymentGateway'.

58 class BrokenGateway implements PaymentGateway {
         ~~~~~~~~~~~~~

  payments.ts:8:3 - 'charge' is declared here.
    8   charge(amountPaise: number, token: string): ChargeResult;
        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

payments.ts:69:10 - error TS7006: Parameter 'amountPaise' implicitly has an 'any' type.

69   charge(amountPaise, token) {
            ~~~~~~~~~~~

payments.ts:69:23 - error TS7006: Parameter 'token' implicitly has an 'any' type.

69   charge(amountPaise, token) {
                         ~~~~~


Found 3 errors in the same file, starting at: payments.ts:58
```

The first error is the point of `implements`: without it, `BrokenGateway` would only fail later, at some `new OrderService(new BrokenGateway())` far away. The second one surprises many people: `implements` checks the class, but it does **not** give the class its types. The parameters of `charge` still need their own annotations.

A class can implement several interfaces, separated by commas: `class FileStore implements Reader, Writer`. And since interfaces are types only, they are gone at run time: `fake instanceof PaymentGateway` gives `error TS2693: 'PaymentGateway' only refers to a type, but is being used as a value here.`

![The PaymentGateway interface is drawn as a socket with two pins, name and charge; FakeGateway and CardGateway both fit the socket and plug into OrderService one after the other; BrokenGateway has a charges pin instead of charge and does not fit, with TS2420](images/21-inheritance-interfaces-abstract-classes/implements-contract.gif)

---

## Abstract Classes

Sometimes the child classes share real code, and only one step differs. An **abstract class** holds the shared code and leaves **abstract** members, with no body, for each child to fill in. Background jobs are a good example: every job is run and reported the same way, but each one does different work

jobs.ts

```typescript
abstract class Job {
  abstract readonly name: string;
  protected abstract execute(): number;

  run(): string {
    try {
      const count = this.execute();
      return `${this.name}: processed ${count} items`;
    } catch (err) {
      const reason = err instanceof Error ? err.message : String(err);
      return `${this.name}: failed (${reason})`;
    }
  }
}

class SessionCleanupJob extends Job {
  readonly name = "session-cleanup";
  readonly #expired: string[];

  constructor(expired: string[]) {
    super();
    this.#expired = expired;
  }

  protected execute(): number {
    return this.#expired.length;
  }
}

class DailyReportJob extends Job {
  readonly name = "daily-report";
  readonly #orderIds: number[];

  constructor(orderIds: number[]) {
    super();
    this.#orderIds = orderIds;
  }

  protected execute(): number {
    if (this.#orderIds.length === 0) {
      throw new Error("no orders today");
    }
    return this.#orderIds.length;
  }
}

const jobs: Job[] = [
  new SessionCleanupJob(["s-81", "s-82", "s-90"]),
  new DailyReportJob([1001, 1002]),
  new DailyReportJob([])
];

for (const job of jobs) {
  console.log(job.run());
}
```

```text
session-cleanup: processed 3 items
daily-report: processed 2 items
daily-report: failed (no orders today)
```

- `abstract class Job` cannot be created with `new`; it exists to be extended.
- `abstract readonly name` and `abstract execute()` have no value and no body. Every child must provide them.
- `run()` is normal shared code. Writing it once means every job reports and handles failures the same way.
- `execute` is `protected`: outside code calls `run()`, never `execute()` directly.

Filling in an abstract member does not need `override`, even with `noImplicitOverride`: there is no parent code being replaced. `override` is for methods that already have a body in the parent.

`tsc` makes sure the abstract parts are really filled in. Add at the end

```typescript
const job = new Job();

class EmptyJob extends Job {
  readonly name = "empty";
}
```

```text
jobs.ts:57:13 - error TS2511: Cannot create an instance of an abstract class.

57 const job = new Job();
               ~~~~~~~~~

jobs.ts:59:7 - error TS2515: Non-abstract class 'EmptyJob' does not implement inherited abstract member execute from class 'Job'.

59 class EmptyJob extends Job {
         ~~~~~~~~


Found 2 errors in the same file, starting at: jobs.ts:57
```

Both checks belong to `tsc`. Node.js removes the word `abstract` like any type, and `Job` becomes a normal class. Run past the error, and `new Job().run()` prints `undefined: failed (this.execute is not a function)`: no name, and no `execute` to call.

![An abstract Job blueprint has a filled-in run method and two empty slots, name and execute; SessionCleanupJob and DailyReportJob fill both slots and can be created; new Job() is refused with TS2511, and EmptyJob, which leaves execute empty, is refused with TS2515](images/21-inheritance-interfaces-abstract-classes/abstract-class.gif)

---

## Abstract Class or Interface?

Both describe what a group of classes must have. The difference is whether there is shared code

|                                    | Interface                              | Abstract class                              |
| ---------------------------------- | -------------------------------------- | ------------------------------------------- |
| Contains code                      | no, only types                         | yes: fields, methods, a constructor         |
| At run time                        | gone                                   | a normal class                              |
| A class can use                    | many: `implements A, B`                | one: `extends A`                            |
| `instanceof`                       | not possible (TS2693)                  | works                                       |
| Good for                           | contracts with very different classes  | a family of classes with shared steps       |

`PaymentGateway` is an interface: a fake gateway and a card gateway share nothing but the shape. `Job` is an abstract class: every job runs, reports and fails the same way. When unsure, start with an interface. Inheritance ties classes together tightly, so keep it to one or two levels.

![What is left at run time: the PaymentGateway interface fades away completely when the types are removed, while the abstract class Job loses its abstract members and the word abstract, and stays a normal class with its run method](images/21-inheritance-interfaces-abstract-classes/runtime-leftovers.gif)

---

## Typed Error Classes

The JavaScript course built `AppError` in [Session 18](https://github.com/mohin-sheikh/javascript-basics-to-advanced/blob/main/18-error-handling.md). In TypeScript it gets types, and `instanceof` gives you access to its extra fields

errors.ts

```typescript
class AppError extends Error {
  readonly statusCode: number;

  constructor(message: string, statusCode: number) {
    super(message);
    this.name = "AppError";
    this.statusCode = statusCode;
  }
}

class NotFoundError extends AppError {
  constructor(what: string) {
    super(`${what} not found`, 404);
    this.name = "NotFoundError";
  }
}

function findOrder(id: number): string {
  if (id !== 1001) {
    throw new NotFoundError(`Order ${id}`);
  }
  return "ORD-1001";
}

try {
  console.log(findOrder(1001));
  console.log(findOrder(1003));
} catch (err) {
  if (err instanceof AppError) {
    console.log(err.statusCode, err.name, err.message);
  } else {
    throw err;
  }
}
```

```text
ORD-1001
404 NotFoundError Order 1003 not found
```

`err` in a `catch` is `unknown` (Session 04), because anything can be thrown. `err instanceof AppError` narrows it, so `err.statusCode` is a `number` inside the `if`. A `NotFoundError` is also an `AppError`, so one check handles the whole family. Using `statusCode` before the check fails. Add a line at the start of the `catch` block

```typescript
} catch (err) {
  console.log(err.statusCode);
  if (err instanceof AppError) {
```

```text
errors.ts:29:15 - error TS18046: 'err' is of type 'unknown'.

29   console.log(err.statusCode);
                 ~~~


Found 1 error in errors.ts:29
```

The class name also shows up when nobody catches the error. Remove that line, and add a call outside the `try` at the end

```typescript
findOrder(1004);
```

```text
ORD-1001
404 NotFoundError Order 1003 not found
file:///D:/ts-practice/21-inheritance/errors.ts:20
    throw new NotFoundError(`Order ${id}`);
          ^

NotFoundError: Order 1004 not found
    at findOrder (file:///D:/ts-practice/21-inheritance/errors.ts:20:11)
    at file:///D:/ts-practice/21-inheritance/errors.ts:36:1
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5) {
  statusCode: 404
}

Node.js v24.19.0
```

Node.js prints `NotFoundError:` from `this.name`, and the extra `statusCode` in braces at the end.

`this.name = "AppError"` is an assignment in the constructor, as in the JavaScript course. Writing it as a field, `name = "AppError";`, would declare `name` again, and `Error` already has one, so with `noImplicitOverride` it gives `error TS4114` and must be written `override name = "AppError";`. Both work; this course uses the assignment.

![The error family tree: Error at the top, AppError below it adding statusCode, and NotFoundError below that; a thrown NotFoundError falls into catch as unknown, passes the instanceof AppError check, and comes out with statusCode 404 available](images/21-inheritance-interfaces-abstract-classes/error-classes.gif)

---

## Mini Example 1: Notification Senders

The JavaScript course ended Session 17 with notification senders: a parent class with the shared `send` steps, and one child class per channel. Here is the TypeScript version, with an abstract class

notifiers.ts

```typescript
interface Customer {
  name: string;
  email: string;
  phone?: string;
  deviceToken?: string;
}

abstract class Notifier {
  abstract readonly channel: string;

  protected abstract address(customer: Customer): string | undefined;

  protected format(customer: Customer, message: string): string {
    return `Hi ${customer.name}, ${message}`;
  }

  send(customer: Customer, message: string): string {
    const to = this.address(customer);
    if (to === undefined) {
      return `[${this.channel}] skipped ${customer.name}`;
    }
    return `[${this.channel}] to ${to}: ${this.format(customer, message)}`;
  }
}

class EmailNotifier extends Notifier {
  readonly channel = "email";

  protected address(customer: Customer): string {
    return customer.email;
  }
}

class SmsNotifier extends Notifier {
  readonly channel = "sms";

  protected address(customer: Customer): string | undefined {
    return customer.phone;
  }

  protected override format(customer: Customer, message: string): string {
    return `ShopAPI: ${message}`;
  }
}

const notifiers: Notifier[] = [new EmailNotifier(), new SmsNotifier()];
const customers: Customer[] = [
  { name: "Riya", email: "riya@example.com", phone: "9876543210", deviceToken: "dev-41" },
  { name: "Aman", email: "aman@example.com" }
];

for (const customer of customers) {
  for (const notifier of notifiers) {
    console.log(notifier.send(customer, "your order ORD-1001 has shipped"));
  }
}
```

```bash
npx tsc --noEmit
node notifiers.ts
```

```text
[email] to riya@example.com: Hi Riya, your order ORD-1001 has shipped
[sms] to 9876543210: ShopAPI: your order ORD-1001 has shipped
[email] to aman@example.com: Hi Aman, your order ORD-1001 has shipped
[sms] skipped Aman
```

What TypeScript adds to the JavaScript version:

- `address` is **abstract**: every channel must say where its messages go. The JavaScript version had no `address` in the parent at all, and only worked because each child happened to have one.
- `address` returns `string | undefined`, so `send` must handle the `undefined` case before it uses `to`. The JavaScript version needed a separate `canSend` method for that.
- `format` has shared code, so `SmsNotifier` replaces it with `override`.
- `address` and `format` are `protected`: they are steps inside `send`, not for outside code.

The phone is optional, and `tsc` keeps the SMS notifier honest. If `SmsNotifier.address` promised a plain `string`

```typescript
  protected address(customer: Customer): string {
    return customer.phone;
  }
```

```text
notifiers.ts:38:5 - error TS2322: Type 'string | undefined' is not assignable to type 'string'.
  Type 'undefined' is not assignable to type 'string'.

38     return customer.phone;
       ~~~~~~


Found 1 error in notifiers.ts:38
```

Adding a channel is one new class. Start a push notifier with only its channel name, after `SmsNotifier`, and add it to the list

```typescript
class PushNotifier extends Notifier {
  readonly channel = "push";
}
```

```typescript
const notifiers: Notifier[] = [new EmailNotifier(), new SmsNotifier(), new PushNotifier()];
```

```text
notifiers.ts:46:7 - error TS2515: Non-abstract class 'PushNotifier' does not implement inherited abstract member address from class 'Notifier'.

46 class PushNotifier extends Notifier {
         ~~~~~~~~~~~~


Found 1 error in notifiers.ts:46
```

`tsc` lists exactly what is missing. Fill it in, with a short format for push messages

```typescript
class PushNotifier extends Notifier {
  readonly channel = "push";

  protected address(customer: Customer): string | undefined {
    return customer.deviceToken;
  }

  protected override format(customer: Customer, message: string): string {
    return message;
  }
}
```

```text
[email] to riya@example.com: Hi Riya, your order ORD-1001 has shipped
[sms] to 9876543210: ShopAPI: your order ORD-1001 has shipped
[push] to dev-41: your order ORD-1001 has shipped
[email] to aman@example.com: Hi Aman, your order ORD-1001 has shipped
[sms] skipped Aman
[push] skipped Aman
```

The loop and `send` did not change. Outside code still cannot reach the steps, or create a bare `Notifier`

```typescript
const raw = new SmsNotifier().address({ name: "Aman", email: "aman@example.com" });
console.log(new Notifier());
```

```text
notifiers.ts:57:31 - error TS2445: Property 'address' is protected and only accessible within class 'SmsNotifier' and its subclasses.

57 const raw = new SmsNotifier().address({ name: "Aman", email: "aman@example.com" });
                                 ~~~~~~~

notifiers.ts:58:13 - error TS2511: Cannot create an instance of an abstract class.

58 console.log(new Notifier());
               ~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: notifiers.ts:57
```

(These two line numbers count from the version without `PushNotifier`.)

![Riya and Aman pass three senders: email, SMS and push all run the same send steps from the abstract Notifier, but each one answers address its own way; Aman has no phone and no device token, so address returns undefined and the SMS and push senders skip Aman](images/21-inheritance-interfaces-abstract-classes/notifiers.gif)

---

## Mini Example 2: Typed API Errors

The JavaScript course's Session 18 turned errors into server responses in one central place. Here is that idea with typed error classes: the code that does the work throws, and one function, `toResponse`, decides every answer

api-errors.ts

```typescript
class AppError extends Error {
  readonly statusCode: number;

  constructor(message: string, statusCode: number) {
    super(message);
    this.name = "AppError";
    this.statusCode = statusCode;
  }
}

class ValidationError extends AppError {
  readonly field: string;

  constructor(field: string, problem: string) {
    super(`${field} ${problem}`, 400);
    this.name = "ValidationError";
    this.field = field;
  }
}

class NotFoundError extends AppError {
  constructor(what: string) {
    super(`${what} not found`, 404);
    this.name = "NotFoundError";
  }
}

class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409);
    this.name = "ConflictError";
  }
}

interface Product {
  id: number;
  name: string;
  stock: number;
}

interface OrderRequest {
  productId: number;
  quantity: number;
}

interface HttpResponse {
  status: number;
  body: { message: string; field?: string };
}

const products: Product[] = [
  { id: 1, name: "Mouse", stock: 5 },
  { id: 2, name: "Keyboard", stock: 0 }
];

function createOrder(request: OrderRequest): string {
  if (!Number.isInteger(request.quantity) || request.quantity < 1) {
    throw new ValidationError("quantity", "must be a whole number from 1");
  }
  const product = products.find((p) => p.id === request.productId);
  if (product === undefined) {
    throw new NotFoundError(`Product ${request.productId}`);
  }
  if (product.stock < request.quantity) {
    throw new ConflictError(`Only ${product.stock} ${product.name} left`);
  }
  product.stock -= request.quantity;
  return `${request.quantity} x ${product.name}`;
}

function toResponse(err: unknown): HttpResponse {
  if (err instanceof ValidationError) {
    return { status: err.statusCode, body: { message: err.message, field: err.field } };
  }
  if (err instanceof AppError) {
    return { status: err.statusCode, body: { message: err.message } };
  }
  console.log("  [log]", err instanceof Error ? err.message : err);
  return { status: 500, body: { message: "Something went wrong" } };
}

function handleOrder(request: OrderRequest): HttpResponse {
  try {
    return { status: 201, body: { message: createOrder(request) } };
  } catch (err) {
    return toResponse(err);
  }
}

console.log(handleOrder({ productId: 1, quantity: 2 }));
console.log(handleOrder({ productId: 1, quantity: 0 }));
console.log(handleOrder({ productId: 9, quantity: 1 }));
console.log(handleOrder({ productId: 2, quantity: 1 }));
console.log(toResponse(new Error("connect ECONNREFUSED 127.0.0.1:27017")));
```

```bash
npx tsc --noEmit
node api-errors.ts
```

```text
{ status: 201, body: { message: '2 x Mouse' } }
{
  status: 400,
  body: {
    message: 'quantity must be a whole number from 1',
    field: 'quantity'
  }
}
{ status: 404, body: { message: 'Product 9 not found' } }
{ status: 409, body: { message: 'Only 0 Keyboard left' } }
  [log] connect ECONNREFUSED 127.0.0.1:27017
{ status: 500, body: { message: 'Something went wrong' } }
```

| Part                                       | Job                                                               |
| ------------------------------------------ | ----------------------------------------------------------------- |
| `AppError` and its children                | Expected errors, each with its status code; `ValidationError` adds `field` |
| `createOrder`                              | Does the work and throws; knows nothing about responses           |
| `toResponse(err: unknown)`                 | The one place that turns any error into a typed `HttpResponse`    |
| The last call                              | Any other error, like this one from a database driver, is logged and answered with a plain 500 |

`toResponse` checks `ValidationError` **first**. Every `ValidationError` is also an `AppError`, so with the checks the other way round, the `AppError` branch would answer every validation error, without its `field`. TypeScript only notices if that branch tries to use `field`. Put the `AppError` check first and give it the `field` too

```typescript
function toResponse(err: unknown): HttpResponse {
  if (err instanceof AppError) {
    return { status: err.statusCode, body: { message: err.message, field: err.field } };
  }
  if (err instanceof ValidationError) {
```

```text
api-errors.ts:73:79 - error TS2339: Property 'field' does not exist on type 'AppError'.

73     return { status: err.statusCode, body: { message: err.message, field: err.field } };
                                                                                 ~~~~~


Found 1 error in api-errors.ts:73
```

Without that `field`, the wrong order would pass `tsc` and quietly lose information. Check the most specific classes first.

A new kind of error, like `class UnauthorizedError extends AppError` with 401, needs no change in `toResponse`. The Node.js course ([Session 24](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/24-error-handling.md)) does the same in Express error middleware, and Session 31 of this course types it.

![Five calls enter handleOrder; the thrown errors drop into toResponse, which sorts them: ValidationError to 400 with its field, NotFoundError to 404, ConflictError to 409, and a plain Error from the database driver to a log line and a 500](images/21-inheritance-interfaces-abstract-classes/api-errors.gif)

---

## Beginner Mistakes

### Mistake 1: Using this before super

`this.permissions = permissions;` before `super(...)` gives `error TS17009: 'super' must be called before accessing 'this' in the constructor of a derived class.` Without any `super` call, it is `error TS2377`. Call `super(...)` first.

---

### Mistake 2: Declaring a parent's field again in the child

`channel: "email";` in a child, when the parent already sets `channel` in its constructor, gives `error TS2612: Property 'channel' will overwrite the base property in 'Notifier'.` It really does: the child's field is set to `undefined` after the parent's constructor ran, and `new EmailNotifier().channel` prints `undefined`. Do not declare it again.

---

### Mistake 3: Overriding without override

Without the keyword and without `noImplicitOverride`, a misspelled `canDelte()` is a new method, and the parent's `canDelete()` still runs. Turn on `noImplicitOverride` and write `override`; a typo then gives `error TS4117` with "Did you mean".

---

### Mistake 4: Changing the types in an override

`override canDelete(): string` in a child of a class whose `canDelete` returns `boolean` gives `error TS2416`, and the child is no longer usable as its parent. Keep the parent's parameter and return types.

---

### Mistake 5: Expecting protected to hide data

`protected`, like `private`, is checked by `tsc` only. `console.log` and `JSON.stringify` still show the value. Use `#` for data that must stay hidden.

---

### Mistake 6: Expecting implements to type the parameters

`charge(amountPaise, token)` in a class that `implements PaymentGateway` gives `error TS7006` for both parameters. `implements` checks the class; it does not fill in its types. Annotate the parameters.

---

### Mistake 7: Using instanceof with an interface

`fake instanceof PaymentGateway` gives `error TS2693: 'PaymentGateway' only refers to a type, but is being used as a value here.` Interfaces do not exist at run time. Check a property instead, or use a class.

---

### Mistake 8: Creating an abstract class

`new Job()` gives `error TS2511: Cannot create an instance of an abstract class.` Create one of its child classes.

---

### Mistake 9: Using an abstract property in the parent's constructor

``console.log(`starting ${this.name}`)`` in the constructor of `Job` gives `error TS2715: Abstract property 'name' in class 'Job' cannot be accessed in the constructor.` The child's fields get their values only after the parent's constructor, so it would print `starting undefined`.

---

### Mistake 10: Checking a parent error class before a child

`if (err instanceof AppError)` before `if (err instanceof ValidationError)` sends every validation error to the first branch, and its `field` is lost. Check the most specific class first.

---

### Mistake 11: Reading error fields without instanceof

`err.statusCode` in a `catch` gives `error TS18046: 'err' is of type 'unknown'.` Narrow it with `err instanceof AppError` first.

---

### Mistake 12: Long chains of classes

`SuperAdmin extends Admin extends Staff extends User` is hard to follow and hard to change. Keep inheritance to one or two levels, and prefer interfaces with separate classes.

---

## Practice Exercises

### Exercise 1

Write a class `Product` with `name` and `pricePaise` and a method `describe(): string`. Write `DigitalProduct` with an extra `downloadUrl`, and `PhysicalProduct` with an extra `weightGrams`. Both override `describe` with `override` and call `super.describe()`. Put all three kinds in one `Product[]` and print them.

### Exercise 2

In `users.ts`, try each constructor mistake on purpose: `this` before `super`, no `super` at all, and `super` with one argument missing. Write down the error code of each.

### Exercise 3

Add a `SavingsAccount extends Account` with `addInterest(percent: number): void`, which uses the protected `balancePaise`. Then change `balancePaise` to `#balancePaise` and read the errors in both child classes.

### Exercise 4

Write `interface Cache { get(key: string): string | undefined; set(key: string, value: string): void; }` and a class `MemoryCache implements Cache` that uses a `Map`. Write a `ProductService` that takes a `Cache` in its constructor and uses it in `findName(id: number): string`. Test it with a `MemoryCache`.

### Exercise 5

Write an abstract class `Exporter` with `protected abstract formatRow(row: { id: number; name: string }): string` and a normal method `export(rows): string` that joins the formatted rows with new lines. Write `CsvExporter` and `JsonLinesExporter`.

### Exercise 6

In Mini Example 1, add a `WhatsAppNotifier` that uses `phone` and formats messages as `"ORD-1001 update: ..."`. Then add a field `quietHours?: boolean` to `Customer` and make `send` skip customers with quiet hours, for every channel at once.

### Exercise 7

In Mini Example 2, add `UnauthorizedError` (401) and `ForbiddenError` (403), throw them from a new function `requireAdmin(role: string)`, and check that `toResponse` needs no change.

### Exercise 8

In Mini Example 2, change `ValidationError` to hold a list of problems, `fields: { field: string; problem: string }[]`, and change the response body to match. Make `createOrder` collect every problem before it throws.

---

## Interview Questions

### What does extends give a child class in TypeScript?

All fields and methods of the parent, with their types. The child can add members and override methods, and a child object can be used wherever the parent type is expected.

### Why must super() come before this in a child constructor?

The parent's constructor creates and prepares the object. Until it has run, `this` does not exist yet. `tsc` reports TS17009, and JavaScript throws a `ReferenceError`.

### What does the override keyword do?

It marks a method that replaces a parent method, and `tsc` checks that the parent really has one (TS4117 otherwise). With `noImplicitOverride`, every override must have it (TS4114), so typos and renamed parent methods are caught.

### What must an overriding method keep the same?

It must fit the parent's version: compatible parameters and a compatible return type. Otherwise TS2416, and the child could no longer be used as the parent.

### What is the difference between public, protected, private and #?

`public` members can be used everywhere. `protected` ones in the class and its children. `private` and `#` ones only in the class itself. The first three are `tsc` checks only; `#` is also private at run time.

### What does implements do?

It makes `tsc` check that a class has the shape of an interface, right at the class (TS2420 if not). It does not change the class or give its methods parameter types.

### What is the difference between an interface and an abstract class?

An interface only describes a shape and disappears at run time; a class can implement many. An abstract class can contain code, exists at run time, works with `instanceof`, and a class can extend only one.

### What is an abstract method?

A method without a body in an abstract class. Every non-abstract child class must implement it, or `tsc` reports TS2515.

### Can you create an object from an abstract class?

Not in TypeScript: TS2511. At run time it is a normal class, because `abstract` is removed with the types.

### What is dependency injection?

Handing a class the objects it depends on, usually through its constructor, typed by an interface: `new OrderService(gateway)`. It lets tests pass a fake, and it is the core idea of NestJS.

### Why use custom error classes in TypeScript?

They carry typed extra data, like `statusCode` or `field`, and `instanceof` narrows the `unknown` error in a `catch` to the class, so that data can be used safely.

### How do you handle different errors in one catch?

Check with `instanceof`, most specific class first (`ValidationError` before `AppError`), and rethrow or log anything you do not recognise.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [20. Classes in TypeScript](20-classes.md) | [Course outline](README.md#course-outline) | [22. Decorators](22-decorators.md) |
