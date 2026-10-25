# Decorators

> **Session 22 of 33** | Part 5: Classes | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Decorators Need a Build Step](#decorators-need-a-build-step)
- [What a Decorator Receives](#what-a-decorator-receives)
- [Replacing a Method](#replacing-a-method)
- [Decorator Factories](#decorator-factories)
- [Stacking Decorators](#stacking-decorators)
- [Class Decorators](#class-decorators)
- [Accessor Decorators](#accessor-decorators)
- [Two Kinds of Decorators](#two-kinds-of-decorators)
- [Reading NestJS Code](#reading-nestjs-code)
- [Mini Example 1: A Decorator-Based Router](#mini-example-1-a-decorator-based-router)
- [Mini Example 2: Validated Order Lines](#mini-example-2-validated-order-lines)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Make a folder `22-decorators` inside `ts-practice`. This session's code cannot run directly with `node file.ts` (the next section shows why), so the folder's `tsconfig.json` builds JavaScript into a `dist` folder, like the enum example in Session 16

22-decorators/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "target": "es2022",
    "outDir": "./dist",
    "sourceMap": false,
    "declaration": false,
    "declarationMap": false
  }
}
```

- `"target": "es2022"` tells `tsc` which JavaScript version to write. For an older target than the newest, `tsc` translates decorators into plain JavaScript.
- `"outDir": "./dist"` puts the built files in `22-decorators/dist`.
- The last three options switch off extra files (source maps and `.d.ts` files) that this session does not need.

In this session you run code in two steps: `npx tsc` builds every `.ts` file in the folder, then `node dist/file.js` runs one of them. `npx tsc` still writes the files when it finds errors, so read its output before you run anything.

---

## Decorators Need a Build Step

A **decorator** is a function that you attach to a class or one of its members with `@`. It runs when the class is created, and can watch or change what it is attached to

announce.ts

```typescript
function announce(method: unknown, context: ClassMethodDecoratorContext): void {
  console.log(`decorating ${String(context.name)} (a ${context.kind})`);
}

class OrderService {
  @announce
  placeOrder(item: string): string {
    return `order placed: ${item}`;
  }
}

console.log("class is ready");
const service = new OrderService();
console.log(service.placeOrder("Keyboard"));
console.log(service.placeOrder("Mouse"));
```

`npx tsc --noEmit` has nothing to report. But try to run the file directly

```bash
node announce.ts
```

```text
file:///D:/ts-practice/22-decorators/announce.ts:6
  @announce
  ^

SyntaxError: Invalid or unexpected token
    at compileSourceTextModule (node:internal/modules/esm/utils:318:16)
    at ModuleLoader.moduleStrategy (node:internal/modules/esm/translators:89:18)
    at ModuleLoader.<anonymous> (node:internal/modules/esm/translators:681:10)
    at #translate (node:internal/modules/esm/loader:434:20)
    at afterLoad (node:internal/modules/esm/loader:502:29)
    at ModuleLoader.loadAndTranslate (node:internal/modules/esm/loader:507:12)
    at #getOrCreateModuleJobAfterResolve (node:internal/modules/esm/loader:560:36)
    at afterResolve (node:internal/modules/esm/loader:607:52)
    at ModuleLoader.getOrCreateModuleJob (node:internal/modules/esm/loader:613:12)
    at node:internal/modules/esm/loader:632:32

Node.js v24.19.0
```

Decorators are a JavaScript feature that is still being added to the language. TypeScript understands them, but the JavaScript engine inside Node.js 24 does not run them yet. They are not types, so Node.js's type stripping leaves them in the code, and the engine stops at the `@`. (`node --experimental-transform-types` from Session 16 gives the same error.)

So `tsc` has to translate them. Build and run

```bash
npx tsc
node dist/announce.js
```

```text
decorating placeOrder (a method)
class is ready
order placed: Keyboard
order placed: Mouse
```

`dist/announce.js` is your code with the `@announce` line replaced by about forty lines of helper code that do the same thing in plain JavaScript. When Node.js supports decorators one day, the build step will no longer be needed.

![node announce.ts stops at the @ sign with a SyntaxError; npx tsc turns announce.ts into dist/announce.js, where the decorator has become plain helper code, and node dist/announce.js runs it](images/22-decorators/build-step.gif)

---

## What a Decorator Receives

Look at the order of the output: `decorating placeOrder` comes **before** `class is ready`, and only once, although `placeOrder` was called twice. The decorator runs one time, while the class is being created, before any object exists.

`@announce` above a method means: call `announce(theMethod, context)` once. The two parameters are

- the method itself. `announce` does not use it, so its type is `unknown`.
- a **context** object with facts about the member. Its type, `ClassMethodDecoratorContext`, is built into TypeScript, and it holds `kind` (`"method"`), `name` (`"placeOrder"`), `static` and `private` (true or false), and a function `addInitializer`, used in Mini Example 1.

`context.name` is a `string | symbol`, because a method can be named with a symbol, so the decorator turns it into text with `String(...)`. If a decorator returns nothing (`void`), the method stays as it was. If it returns a function, that function **replaces** the method.

![A timeline: while the class OrderService is created, @announce runs once and prints its line; then class is ready, then two calls to placeOrder run without the decorator running again](images/22-decorators/decorator-timing.gif)

---

## Replacing a Method

A logging decorator wraps every call: print the arguments, call the real method, print the result. To keep all types, it uses generics (Session 14) and a `this` parameter (Session 20). Put it in its own file, so that other files can import it

decorators.ts

```typescript
export type Method<This, Args extends unknown[], Return> = (this: This, ...args: Args) => Return;

export function logged<This, Args extends unknown[], Return>(
  method: Method<This, Args, Return>,
  context: ClassMethodDecoratorContext<This, Method<This, Args, Return>>
): Method<This, Args, Return> {
  const name = String(context.name);
  return function (this: This, ...args: Args): Return {
    console.log(`-> ${name}(${args.join(", ")})`);
    const result = method.apply(this, args);
    console.log(`<- ${name} returned ${result}`);
    return result;
  };
}
```

`Method<This, Args, Return>` is a name for "a method of an object of type `This`, taking the parameters `Args` and returning `Return`". TypeScript fills in the three type parameters from the method that `@logged` is attached to. The replacement function has exactly the same type, and `method.apply(this, args)` calls the original method on the same object with the same arguments.

order-service.ts

```typescript
import { logged } from "./decorators.ts";

class OrderService {
  #nextId = 1001;

  @logged
  placeOrder(item: string, quantity: number): string {
    const id = this.#nextId;
    this.#nextId += 1;
    return `ORD-${id}: ${quantity} x ${item}`;
  }

  @logged
  cancel(orderId: string): boolean {
    return orderId.startsWith("ORD-");
  }
}

const service = new OrderService();
service.placeOrder("Keyboard", 1);
service.cancel("ORD-1001");
```

```bash
npx tsc
node dist/order-service.js
```

```text
-> placeOrder(Keyboard, 1)
<- placeOrder returned ORD-1001: 1 x Keyboard
-> cancel(ORD-1001)
<- cancel returned true
```

One decorator, two very different methods, and neither of them contains a single line of logging code. Because the replacement keeps the method's type, calls are still checked. And `@logged` only fits methods. Add a wrong call and a decorated field at the end

```typescript
service.placeOrder("Mouse");

class Cart {
  @logged
  total = 0;
}
```

```text
order-service.ts:22:9 - error TS2554: Expected 2 arguments, but got 1.

22 service.placeOrder("Mouse");
           ~~~~~~~~~~

  order-service.ts:7:28 - An argument for 'quantity' was not provided.
    7   placeOrder(item: string, quantity: number): string {
                                 ~~~~~~~~~~~~~~~~

order-service.ts:25:4 - error TS1240: Unable to resolve signature of property decorator when called as an expression.
  Argument of type 'undefined' is not assignable to parameter of type 'Method<Cart, unknown[], unknown>'.

25   @logged
      ~~~~~~

order-service.ts:25:4 - error TS1270: Decorator function return type 'Method<Cart, unknown[], unknown>' is not assignable to type 'void | ((this: Cart, value: number) => number)'.
  Type 'Method<Cart, unknown[], unknown>' is not assignable to type '(this: Cart, value: number) => number'.
    Type 'unknown' is not assignable to type 'number'.

25   @logged
      ~~~~~~


Found 3 errors in the same file, starting at: order-service.ts:22
```

Decorator errors are long. Read the first line of each: TS1240 says "property decorator", so `@logged` was put on a field, and a field decorator receives `undefined` instead of a method.

![A call to placeOrder first enters the wrapper made by @logged, which prints the arguments, then passes the call through to the original method, which returns ORD-1001, and the wrapper prints the result on the way out](images/22-decorators/replace-method.gif)

---

## Decorator Factories

A decorator that needs settings is made by a **decorator factory**: a function that takes the settings and returns the decorator. `@requireRole("admin")` calls `requireRole("admin")` first, and the decorator it returns is attached to the method. Add this to the end of `decorators.ts`

```typescript
export type Role = "customer" | "support" | "admin";

export function requireRole(role: Role) {
  return function <This extends { role: Role }, Args extends unknown[], Return>(
    method: Method<This, Args, Return>,
    context: ClassMethodDecoratorContext<This, Method<This, Args, Return>>
  ): Method<This, Args, Return> {
    return function (this: This, ...args: Args): Return {
      if (this.role !== role) {
        throw new Error(`${String(context.name)} needs role ${role}, got ${this.role}`);
      }
      return method.apply(this, args);
    };
  };
}
```

`This extends { role: Role }` is a constraint (Session 14): the decorator can only be used in classes whose objects have a `role`, so `this.role` is safe inside. `requireRole` itself has no return type annotation: written out, it would repeat the whole inner function's type. This is one of the few places where letting TypeScript infer it reads better.

refunds.ts

```typescript
import { requireRole, type Role } from "./decorators.ts";

class RefundService {
  readonly role: Role;

  constructor(role: Role) {
    this.role = role;
  }

  @requireRole("support")
  refund(orderId: string): string {
    return `refunded ${orderId}`;
  }

  @requireRole("admin")
  deleteOrder(orderId: string): string {
    return `deleted ${orderId}`;
  }
}

const support = new RefundService("support");
console.log(support.refund("ORD-1001"));

try {
  support.deleteOrder("ORD-1001");
} catch (err) {
  console.log(err instanceof Error ? err.message : err);
}
```

```bash
npx tsc
node dist/refunds.js
```

```text
refunded ORD-1001
deleteOrder needs role admin, got support
```

The rule "only admins may delete orders" is written once, at the top of the method, where everyone reading the class sees it. A misspelled role, `@requireRole("admn")`, gives `error TS2345: Argument of type '"admn"' is not assignable to parameter of type 'Role'.` And the constraint is checked too. Add a class without a `role` at the end

```typescript
class ReportService {
  @requireRole("admin")
  exportCsv(): string {
    return "id,total";
  }
}
```

```text
refunds.ts:31:4 - error TS1241: Unable to resolve signature of method decorator when called as an expression.
  Argument of type 'ClassMethodDecoratorContext<ReportService, () => string> & { name: "exportCsv"; private: false; static: false; }' is not assignable to parameter of type 'ClassMethodDecoratorContext<{ role: Role; }, Method<{ role: Role; }, [], string>>'.
    The types of 'access.has' are incompatible between these types.
      Type '(object: ReportService) => boolean' is not assignable to type '(object: { role: Role; }) => boolean'.
        Types of parameters 'object' and 'object' are incompatible.
          Property 'exportCsv' is missing in type '{ role: Role; }' but required in type 'ReportService'.

31   @requireRole("admin")
      ~~~~~~~~~~~~~~~~~~~~

  refunds.ts:32:3 - 'exportCsv' is declared here.
    32   exportCsv(): string {
         ~~~~~~~~~


Found 1 error in refunds.ts:31
```

A hard message to read. The useful part is in the second line: the decorator wants a context for `{ role: Role }`, and it got one for `ReportService`, which has no `role`.

![requireRole("admin") is called first and hands back a decorator that remembers "admin"; that decorator wraps deleteOrder with a check, so a call from a support user is stopped with an error, while refund, wrapped with "support", goes through](images/22-decorators/factory.gif)

---

## Stacking Decorators

A method can have several decorators. Remove `ReportService`, import `logged` too, and put it above the role check

```typescript
import { logged, requireRole, type Role } from "./decorators.ts";
```

```typescript
  @logged
  @requireRole("admin")
  deleteOrder(orderId: string): string {
```

Then try it with an admin as well, at the end of the file

```typescript
const admin = new RefundService("admin");
admin.deleteOrder("ORD-1002");
```

```text
refunded ORD-1001
-> deleteOrder(ORD-1001)
deleteOrder needs role admin, got support
-> deleteOrder(ORD-1002)
<- deleteOrder returned deleted ORD-1002
```

Decorators are applied from the bottom up: `@requireRole` wraps the method first, then `@logged` wraps the result. So a call goes through them from the **top** down: `@logged` prints its line, then the role check runs. For the support user the check throws, so there is no `<-` line. Swap the two lines, and the denied call is not logged at all:

```text
refunded ORD-1001
deleteOrder needs role admin, got support
-> deleteOrder(ORD-1002)
<- deleteOrder returned deleted ORD-1002
```

Decide which order you want: log every attempt, or log only the calls that were allowed.

![Two wrappers around deleteOrder like layers of an onion: @requireRole is applied first and sits on the inside, @logged is applied second and sits outside; a call passes the outer layer first, so the attempt is logged before the role check stops it](images/22-decorators/stacking.gif)

---

## Class Decorators

A decorator above a class receives the class itself. A common use is **registering** classes, so that a framework can find them later. This scheduler collects background jobs, each with its own interval

jobs.ts

```typescript
interface Job {
  run(): string;
}

interface ScheduledJob {
  interval: string;
  name: string;
  create: new () => Job;
}

const cronTable: ScheduledJob[] = [];

function every(interval: string) {
  return function (value: new () => Job, context: ClassDecoratorContext): void {
    cronTable.push({ interval, name: String(context.name), create: value });
  };
}

@every("5 minutes")
class CleanupSessionsJob implements Job {
  run(): string {
    return "removed 3 expired sessions";
  }
}

@every("1 day")
class DailyReportJob implements Job {
  run(): string {
    return "sent the daily report";
  }
}

for (const entry of cronTable) {
  const job = new entry.create();
  console.log(`[every ${entry.interval}] ${entry.name}: ${job.run()}`);
}
```

```text
[every 5 minutes] CleanupSessionsJob: removed 3 expired sessions
[every 1 day] DailyReportJob: sent the daily report
```

`new () => Job` is a **constructor type**: something you can call with `new` and no arguments, that makes a `Job`. The loop never names the two classes; the decorators put them in `cronTable`. Because of that type, only real jobs can be scheduled. Add at the end

```typescript
@every("1 hour")
class Mailer {
  send(to: string): string {
    return `mail to ${to}`;
  }
}
```

```text
jobs.ts:38:2 - error TS1238: Unable to resolve signature of class decorator when called as an expression.
  Argument of type 'typeof Mailer' is not assignable to parameter of type 'new () => Job'.
    Property 'run' is missing in type 'Mailer' but required in type 'Job'.

38 @every("1 hour")
    ~~~~~~~~~~~~~~~

  jobs.ts:2:3 - 'run' is declared here.
    2   run(): string;
        ~~~~~~~~~~~~~~


Found 1 error in jobs.ts:38
```

![Two job classes are created one after the other; @every catches each one as it is defined and writes a row into the cronTable with its interval; a Mailer class without run is turned away with TS1238](images/22-decorators/class-decorator.gif)

---

## Accessor Decorators

Decorators can also watch a value. They need a property written with the keyword `accessor`: `accessor quantity = 1` makes a hidden private field plus a getter and a setter for it, and a decorator can replace that getter and setter. This one checks every value that is stored

cart-line.ts

```typescript
function range(min: number, max: number) {
  return function <This>(
    target: ClassAccessorDecoratorTarget<This, number>,
    context: ClassAccessorDecoratorContext<This, number>
  ): ClassAccessorDecoratorResult<This, number> {
    const name = String(context.name);
    function check(value: number): number {
      if (!Number.isInteger(value) || value < min || value > max) {
        throw new RangeError(`${name} must be a whole number from ${min} to ${max}, got ${value}`);
      }
      return value;
    }
    return {
      init: check,
      set(this: This, value: number): void {
        target.set.call(this, check(value));
      }
    };
  };
}

class CartLine {
  @range(1, 10) accessor quantity = 1;
  @range(0, 50) accessor discountPercent = 0;
}

const line = new CartLine();
line.quantity = 3;
line.discountPercent = 10;
console.log(line.quantity, line.discountPercent);
line.quantity = 30;
```

```text
3 10
file:///D:/ts-practice/22-decorators/dist/cart-line.js:40
                throw new RangeError(`${name} must be a whole number from ${min} to ${max}, got ${value}`);
                      ^

RangeError: quantity must be a whole number from 1 to 10, got 30
    at check (file:///D:/ts-practice/22-decorators/dist/cart-line.js:40:23)
    at CartLine.set (file:///D:/ts-practice/22-decorators/dist/cart-line.js:47:39)
    at file:///D:/ts-practice/22-decorators/dist/cart-line.js:83:15
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

- `target` holds the original `get` and `set`.
- The decorator returns new versions: `init` checks the starting value, `set` checks a value before passing it on to the original `set`. There is no `get`, so reading stays as it was.

Session 20 wrote the same check by hand in a setter. The decorator writes it once, for every number property that needs a range. It is also typed for numbers only: `@range(1, 10) accessor sku = "KB-101"` gives `error TS1240` and `error TS1270`, because a `string` accessor is not a `number` accessor.

Note that the error points into `dist/cart-line.js`, the built file, not into your `.ts` file. That is the price of the build step.

![line.quantity = 3 goes through the setter that @range put in front of the hidden field, passes the check and is stored; line.quantity = 30 is stopped by the same check with a RangeError before it reaches the field](images/22-decorators/accessor-decorator.gif)

---

## Two Kinds of Decorators

Everything so far uses **standard decorators**, the version that is becoming part of JavaScript. TypeScript has supported them since version 5.0, with no option needed. There is an older kind, from before the JavaScript design was finished, which you switch on with `"experimentalDecorators": true`. Many popular libraries still use the older kind, so you will meet both

|                               | Standard decorators              | `experimentalDecorators`                       |
| ----------------------------- | -------------------------------- | ---------------------------------------------- |
| Turned on by                  | nothing, it is the default       | `"experimentalDecorators": true`               |
| A method decorator receives   | `(method, context)`              | `(target, propertyKey, descriptor)`            |
| Parameter decorators          | not allowed                      | allowed, like `@Param("id") id: string`        |
| Type information at run time  | no                               | yes, with `"emitDecoratorMetadata": true`      |
| Used by                       | new code                         | NestJS, TypeORM, class-validator               |

The two kinds look the same in code (`@Name`), but the functions behind them receive different arguments, so a decorator written for one kind does not work with the other. Copy a NestJS-style decorator into this folder

nest-in-standard.ts

```typescript
function Get(path: string) {
  return function (target: object, propertyKey: string, descriptor: PropertyDescriptor): void {
    console.log(`@Get("${path}") on ${propertyKey}`);
  };
}

function Param(name: string) {
  return function (target: object, propertyKey: string | undefined, parameterIndex: number): void {
    console.log(`@Param("${name}") on parameter ${parameterIndex} of ${propertyKey}`);
  };
}

class OrdersController {
  @Get(":id")
  findOne(@Param("id") id: string): string {
    return `order ${id}`;
  }
}
```

```text
nest-in-standard.ts:14:3 - error TS1241: Unable to resolve signature of method decorator when called as an expression.
  The runtime will invoke the decorator with 2 arguments, but the decorator expects 3.

14   @Get(":id")
     ~~~~~~~~~~~

  nest-in-standard.ts:2:57 - An argument for 'descriptor' was not provided.
    2   return function (target: object, propertyKey: string, descriptor: PropertyDescriptor): void {
                                                              ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

nest-in-standard.ts:15:11 - error TS1206: Decorators are not valid here.

15   findOne(@Param("id") id: string): string {
             ~


Found 2 errors in the same file, starting at: nest-in-standard.ts:14
```

The first message says it exactly: standard decorators get 2 arguments, this one expects the old 3. The second one: standard decorators cannot be put on parameters. Delete `nest-in-standard.ts` before you go on.

![Two doors for the same @Get(":id") line: in standard mode the decorator is called with method and context, two arguments; in experimentalDecorators mode with target, propertyKey and descriptor, three arguments; a decorator written for one door does not fit the other](images/22-decorators/two-kinds.gif)

---

## Reading NestJS Code

NestJS, the popular Node.js framework, is built on `experimentalDecorators`. You do not need NestJS to understand its code: write tiny versions of four of its decorators, which only print what they are given. Make a second folder, `22-nest-style`, with its own settings

22-nest-style/tsconfig.json

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "target": "es2022",
    "outDir": "./dist",
    "sourceMap": false,
    "declaration": false,
    "declarationMap": false,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "erasableSyntaxOnly": false
  }
}
```

`"erasableSyntaxOnly": false` allows parameter properties (Session 20) in this folder. This code is built by `tsc` anyway, and NestJS code uses them everywhere.

nest-style.ts

```typescript
function Injectable() {
  return function (target: Function): void {
    console.log(`@Injectable() on ${target.name}`);
  };
}

function Controller(prefix: string) {
  return function (target: Function): void {
    console.log(`@Controller("${prefix}") on ${target.name}`);
  };
}

function Get(path: string) {
  return function (target: object, propertyKey: string, descriptor: PropertyDescriptor): void {
    console.log(`@Get("${path}") on ${propertyKey}`);
  };
}

function Param(name: string) {
  return function (target: object, propertyKey: string | undefined, parameterIndex: number): void {
    console.log(`@Param("${name}") on parameter ${parameterIndex} of ${propertyKey}`);
  };
}

@Injectable()
class OrdersService {
  findOne(id: string): string {
    return `order ${id}`;
  }
}

@Controller("orders")
class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  @Get(":id")
  findOne(@Param("id") id: string): string {
    return this.ordersService.findOne(id);
  }
}

const controller = new OrdersController(new OrdersService());
console.log(controller.findOne("1001"));
```

```bash
cd ../22-nest-style
npx tsc
node dist/nest-style.js
```

```text
@Injectable() on OrdersService
@Param("id") on parameter 0 of findOne
@Get(":id") on findOne
@Controller("orders") on OrdersController
order 1001
```

The bottom half of `nest-style.ts` is how a real NestJS controller looks. Reading it:

- `@Injectable()` marks a class that NestJS may create and hand to others.
- `@Controller("orders")` says: this class answers requests under `/orders`.
- `@Get(":id")` says: this method answers `GET /orders/:id`.
- `@Param("id") id: string` is a **parameter decorator**: fill this parameter from the route parameter `id` (the `:id` from Session 19's route types).
- `constructor(private readonly ordersService: OrdersService) {}` asks for an `OrdersService`. In a real app you never write `new OrdersController(...)`: NestJS creates the service and passes it in (dependency injection, Session 21).

How does NestJS know that the constructor wants an `OrdersService`? Types are gone at run time. That is what `emitDecoratorMetadata` is for. Part of `dist/nest-style.js`:

```text
__decorate([
    Get(":id"),
    __param(0, Param("id")),
    __metadata("design:type", Function),
    __metadata("design:paramtypes", [String]),
    __metadata("design:returntype", String)
], OrdersController.prototype, "findOne", null);
OrdersController = __decorate([
    Controller("orders"),
    __metadata("design:paramtypes", [OrdersService])
], OrdersController);
```

`tsc` wrote the constructor's parameter types into the JavaScript: `design:paramtypes` is `[OrdersService]`, the class itself, a real value. NestJS reads it with the `reflect-metadata` package and creates an `OrdersService` for you. (Without that package, as here, `__metadata` does nothing.) This is a rare case where type annotations end up in the JavaScript, and it only works with `experimentalDecorators`.

![tsc reads the constructor of OrdersController, sees the parameter type OrdersService, and writes design:paramtypes [OrdersService] into the built JavaScript; at run time NestJS reads that list, creates an OrdersService and passes it to the controller](images/22-decorators/nest-metadata.gif)

---

## Mini Example 1: A Decorator-Based Router

Go back to the `22-decorators` folder. Express (Node.js course) registers routes with calls like `app.get("/orders", handler)`. NestJS does it with decorators on controller methods. Here is a small router in that style, with standard decorators, a template literal type for the routes (Session 19), and one more context feature, `addInitializer`

router.ts

```typescript
type RouteKey = `${"GET" | "POST" | "DELETE"} /${string}`;
type Handler = (body: unknown) => string;

const routes = new Map<RouteKey, Handler>();

function route(key: RouteKey) {
  return function <This>(
    method: (this: This, body: unknown) => string,
    context: ClassMethodDecoratorContext<This, (this: This, body: unknown) => string>
  ): void {
    context.addInitializer(function (this: This) {
      routes.set(key, (body) => method.call(this, body));
    });
  };
}

interface Order {
  id: number;
  item: string;
}

function isNewOrder(body: unknown): body is { item: string } {
  return typeof body === "object" && body !== null && "item" in body && typeof body.item === "string";
}

class OrdersController {
  #orders: Order[] = [{ id: 1001, item: "Keyboard" }];

  @route("GET /orders")
  list(): string {
    return `200 ${JSON.stringify(this.#orders)}`;
  }

  @route("POST /orders")
  create(body: unknown): string {
    if (!isNewOrder(body)) {
      return "400 item is required";
    }
    const order: Order = { id: 1001 + this.#orders.length, item: body.item };
    this.#orders.push(order);
    return `201 ${JSON.stringify(order)}`;
  }
}

console.log("routes before:", routes.size);
new OrdersController();
console.log("routes after:", [...routes.keys()]);

function dispatch(key: RouteKey, body?: unknown): string {
  const handler = routes.get(key);
  return handler === undefined ? `404 no route for ${key}` : handler(body);
}

console.log(dispatch("GET /orders"));
console.log(dispatch("POST /orders", { item: "Mouse" }));
console.log(dispatch("POST /orders", { name: "Mouse" }));
console.log(dispatch("GET /orders"));
console.log(dispatch("DELETE /orders"));
```

```bash
npx tsc
node dist/router.js
```

```text
routes before: 0
routes after: [ 'GET /orders', 'POST /orders' ]
200 [{"id":1001,"item":"Keyboard"}]
201 {"id":1002,"item":"Mouse"}
400 item is required
200 [{"id":1001,"item":"Keyboard"},{"id":1002,"item":"Mouse"}]
404 no route for DELETE /orders
```

How it works:

- `context.addInitializer(fn)` asks for `fn` to run **each time an object is created**, with `this` set to the new object. That is why there are no routes before `new OrdersController()`, and two after it.
- Each route stores `(body) => method.call(this, body)`: the method, tied to that object. Session 20's "losing this" cannot happen here, and `this.#orders` works inside `list` and `create`.
- The body is `unknown`, as it is for real requests, so `create` checks it with a type predicate (Session 10) before using `body.item`.

The decorator checks both the route and the method it is attached to. Add two methods to the controller, before `create`

```typescript
  @route("PUT /orders/1001")
  replace(body: unknown): string {
    return `200 ${String(body)}`;
  }

  @route("GET /orders/count")
  count(): number {
    return this.#orders.length;
  }
```

```text
router.ts:34:10 - error TS2345: Argument of type '"PUT /orders/1001"' is not assignable to parameter of type '`DELETE /${string}` | `GET /${string}` | `POST /${string}`'.

34   @route("PUT /orders/1001")
            ~~~~~~~~~~~~~~~~~~

router.ts:39:4 - error TS1241: Unable to resolve signature of method decorator when called as an expression.
  Argument of type '() => number' is not assignable to parameter of type '(this: OrdersController, body: unknown) => string'.
    Type 'number' is not assignable to type 'string'.

39   @route("GET /orders/count")
      ~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 2 errors in the same file, starting at: router.ts:34
```

`PUT` is not one of the router's methods, and a handler must return the response text, not a number.

![new OrdersController() runs the initializers that @route left behind: each one writes a route key and a handler tied to the new controller into the routes Map; dispatch then looks up GET /orders and POST /orders, and DELETE /orders finds nothing and gets a 404](images/22-decorators/router.gif)

---

## Mini Example 2: Validated Order Lines

Libraries like class-validator check incoming data with decorators: `@MaxLength(12)`, `@Min(1)`. Here is the same idea with standard accessor decorators, checked whenever a value is stored. First, a small library of validators

validators.ts

```typescript
export type AccessorDecorator<Value> = <This>(
  target: ClassAccessorDecoratorTarget<This, Value>,
  context: ClassAccessorDecoratorContext<This, Value>
) => ClassAccessorDecoratorResult<This, Value>;

function validated<Value>(check: (value: Value, name: string) => Value): AccessorDecorator<Value> {
  return function (target, context) {
    const name = String(context.name);
    return {
      init: (value) => check(value, name),
      set(value) {
        target.set.call(this, check(value, name));
      }
    };
  };
}

export function range(min: number, max: number): AccessorDecorator<number> {
  return validated<number>((value, name) => {
    if (!Number.isInteger(value) || value < min || value > max) {
      throw new RangeError(`${name} must be a whole number from ${min} to ${max}, got ${value}`);
    }
    return value;
  });
}

export function maxLength(limit: number): AccessorDecorator<string> {
  return validated<string>((value, name) => {
    if (value.length > limit) {
      throw new RangeError(`${name} must have at most ${limit} characters, got "${value}"`);
    }
    return value;
  });
}

export const trimmed: AccessorDecorator<string> = validated<string>((value) => value.trim());
```

- `AccessorDecorator<Value>` names the type of an accessor decorator for one kind of value. With that name, each factory gets a short return type, and the function inside `validated` needs no annotations: TypeScript takes the types of `target`, `context`, `value` and `this` from it (contextual typing).
- `validated` turns a simple `check` function into a decorator. Every validator is then just its check: `range`, `maxLength`, and `trimmed`, which does not refuse anything but cleans the text.

order-line.ts

```typescript
import { maxLength, range, trimmed } from "./validators.ts";

class OrderLine {
  @trimmed @maxLength(12) accessor sku = "";
  @range(1, 10) accessor quantity = 1;
  @range(0, 50) accessor discountPercent = 0;

  constructor(sku: string, quantity: number) {
    this.sku = sku;
    this.quantity = quantity;
  }
}

const inputs: [string, number][] = [
  ["  KB-101 ", 2],
  ["MOUSE-WIRELESS-2", 1],
  ["CABLE-1M", 0]
];

for (const [sku, quantity] of inputs) {
  try {
    const line = new OrderLine(sku, quantity);
    console.log(`ok: "${line.sku}" x ${line.quantity}`);
  } catch (err) {
    console.log(`rejected: ${err instanceof Error ? err.message : err}`);
  }
}
```

```bash
npx tsc
node dist/order-line.js
```

```text
ok: "KB-101" x 2
rejected: sku must have at most 12 characters, got "MOUSE-WIRELESS-2"
rejected: quantity must be a whole number from 1 to 10, got 0
```

`@trimmed @maxLength(12)` are stacked. `@trimmed` is the outer one, so a value is trimmed first and then measured: `"  KB-101 "` is stored as `"KB-101"`. The constructor's assignments go through the same checks, so a bad `OrderLine` can never be created. Later changes are checked too. Add at the end

```typescript
const line = new OrderLine("KB-101", 1);
line.discountPercent = 75;
```

```text
ok: "KB-101" x 2
rejected: sku must have at most 12 characters, got "MOUSE-WIRELESS-2"
rejected: quantity must be a whole number from 1 to 10, got 0
file:///D:/ts-practice/22-decorators/dist/validators.js:15
            throw new RangeError(`${name} must be a whole number from ${min} to ${max}, got ${value}`);
                  ^

RangeError: discountPercent must be a whole number from 0 to 50, got 75
    at file:///D:/ts-practice/22-decorators/dist/validators.js:15:19
    at OrderLine.set (file:///D:/ts-practice/22-decorators/dist/validators.js:7:39)
    at file:///D:/ts-practice/22-decorators/dist/order-line.js:88:22
    at ModuleJob.run (node:internal/modules/esm/module_job:439:25)
    at async node:internal/modules/esm/loader:643:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v24.19.0
```

The types still work as well: `@maxLength(12) accessor quantity = 1` gives `error TS1240` and `error TS1270`, because `maxLength` is for strings.

One thing to know before you send an `OrderLine` in an API response. Replace the last line with these two

```typescript
line.discountPercent = 10;
console.log(line, JSON.stringify(line));
```

```text
ok: "KB-101" x 2
rejected: sku must have at most 12 characters, got "MOUSE-WIRELESS-2"
rejected: quantity must be a whole number from 1 to 10, got 0
OrderLine {} {}
```

The values live in hidden private fields, behind getters and setters, so `console.log` and `JSON.stringify` see nothing. A `toJSON` method (JavaScript course, Session 19) decides what `JSON.stringify` writes. Add it to the class, after the constructor

```typescript
  toJSON(): { sku: string; quantity: number; discountPercent: number } {
    return { sku: this.sku, quantity: this.quantity, discountPercent: this.discountPercent };
  }
```

```text
OrderLine {} {"sku":"KB-101","quantity":1,"discountPercent":10}
```

![Three raw inputs pass through the stacked checks of OrderLine: "  KB-101 " is trimmed and accepted, MOUSE-WIRELESS-2 is too long for maxLength(12), and a quantity of 0 is outside range(1, 10); only the valid line is created](images/22-decorators/validated-line.gif)

---

## Beginner Mistakes

### Mistake 1: Running a file with decorators directly

`node file.ts` stops at the first `@` with `SyntaxError: Invalid or unexpected token`. Node.js 24 cannot run decorators yet. Build with `tsc` and a target like `es2022`, then run the `.js` file.

---

### Mistake 2: Building with the newest target

With `"target": "esnext"`, `tsc` copies the `@` lines into `dist` unchanged, and Node.js stops at them again. Use `es2022`, so that `tsc` translates them.

---

### Mistake 3: Expecting the decorator to run on every call

A decorator runs once, when the class is created. Only the function it returns runs on every call.

---

### Mistake 4: Forgetting to call the original method

A replacement that does not call `method.apply(this, args)` replaces the method completely, and the original code never runs. Because the replacement is typed to return `Return`, forgetting the `return` gives `error TS2355: A function whose declared type is neither 'undefined', 'void', nor 'any' must return a value.`

---

### Mistake 5: Calling the original method without this

`method(...args)` instead of `method.apply(this, args)` would lose `this` (Session 20). Because the `Method` type declares `this`, `tsc` catches it: `error TS2684: The 'this' context of type 'void' is not assignable to method's 'this' of type 'This'.`

---

### Mistake 6: Putting a method decorator on a field

`@logged` above `total = 0` gives `error TS1240: Unable to resolve signature of property decorator when called as an expression.` Each kind of member needs its own kind of decorator.

---

### Mistake 7: Forgetting the parentheses of a factory

A factory must be called: `@requireRole("admin")`, `@range(1, 10)`. Writing `@requireRole` passes the method to the factory instead of a role, and `tsc` reports a long TS1241 error.

---

### Mistake 8: Not thinking about the order of stacked decorators

The decorator closest to the method is applied first, and the top one runs first on each call. `@logged` above `@requireRole` logs denied calls; below it, it does not.

---

### Mistake 9: Mixing the two kinds of decorators

A decorator written for `experimentalDecorators` (three parameters) in a standard project gives `error TS1241` with "The runtime will invoke the decorator with 2 arguments, but the decorator expects 3." Parameter decorators give `error TS1206: Decorators are not valid here.` Check which kind a library needs.

---

### Mistake 10: Sending accessor properties as JSON

Values of `accessor` properties are hidden, so `JSON.stringify` gives `{}`. Add a `toJSON` method, or build a plain object for the response.

---

### Mistake 11: Using decorators for everything

A decorator hides behaviour behind one line. Use it for things that repeat across many methods or classes, like logging, access checks, validation and registration. A one-off check is clearer as normal code inside the method.

---

## Practice Exercises

### Exercise 1

Write a decorator `@deprecated` that prints a warning like `"cancel is deprecated"` the first time a method is called, and never again for that method. Use it on `OrderService.cancel` and call it three times.

### Exercise 2

Write a decorator factory `@timed(label: string)` that measures how long a method takes with `Date.now()` and prints `"label: N ms"`. Use it on a method that loops a million times.

### Exercise 3

Write `@memoize` for methods with one `number` parameter: it stores results in a `Map` and returns the stored result for a number it has seen before. Count the real calls on a slow `priceWithTax(paise: number)` method to prove it.

### Exercise 4

Change `requireRole` so that it takes several roles: `@requireRole("support", "admin")`. The method is allowed when `this.role` is one of them.

### Exercise 5

In `jobs.ts`, add a second class decorator `@disabled` that removes a job from `cronTable` again. Stack it with `@every` and explain which order the two must be in.

### Exercise 6

In Mini Example 1, add `DELETE /orders` with a body `{ id: number }` that removes an order and answers `204` or `404`. Then try to register two methods for the same route key and decide what the router should do.

### Exercise 7

In Mini Example 2, add a validator `@oneOf("INR", "USD")` for a `currency` accessor, and a validator `@pattern(regex)` for the `sku`, like `/^[A-Z]+-\d+$/`.

### Exercise 8

In `22-nest-style`, add a `@Post()` decorator and a `@Body()` parameter decorator that print like the others, and use them on a `create(body: string)` method. Look at the `__decorate` part of the built file and find your decorators in it.

---

## Interview Questions

### What is a decorator?

A function attached with `@` to a class, method, accessor or field. It runs once, when the class is created, receives the member and a context object, and can replace the member or register it somewhere.

### Why can Node.js not run decorators directly?

The JavaScript engine in Node.js 24 does not support decorators yet, and type stripping leaves them in because they are not types. `tsc` must translate them, with a target like `es2022`.

### What does a method decorator receive and return?

In standard decorators: the method and a `ClassMethodDecoratorContext` with its name, kind and more. It returns nothing to keep the method, or a new function that replaces it.

### What is a decorator factory?

A function that takes settings and returns a decorator: `@requireRole("admin")` calls `requireRole("admin")` and attaches the decorator it returns.

### In which order do stacked decorators run?

They are applied from the bottom up, so the top decorator ends up outermost and runs first on each call.

### What can a class decorator do?

It receives the class itself. It can register the class, for example in a list of jobs or routes, add initializers, or return a replacement class.

### What are accessor decorators?

Decorators for properties declared with `accessor`. They receive the original getter and setter and can return new ones, which is useful for validation and change tracking.

### What is addInitializer used for?

It registers a function that runs when an object is created (for instance members), with `this` set to that object. It is used to set things up per object, like binding methods into a router.

### What is the difference between standard decorators and experimentalDecorators?

Standard decorators follow the JavaScript design and need no option. `experimentalDecorators` is TypeScript's older version: different decorator parameters, parameter decorators, and type metadata with `emitDecoratorMetadata`. NestJS, TypeORM and class-validator use the older one.

### What does emitDecoratorMetadata do?

It makes `tsc` write type information into the JavaScript, like `design:paramtypes` with a constructor's parameter types. NestJS reads it with `reflect-metadata` to know which services to inject.

### How do you read @Controller, @Get and @Param in NestJS code?

`@Controller("orders")` makes a class answer requests under `/orders`, `@Get(":id")` makes a method answer `GET /orders/:id`, and `@Param("id")` fills a parameter from the route parameter `id`.

### When should you not use a decorator?

When the behaviour is used only once, or when hiding it behind one line makes the code harder to follow. Plain function calls are easier to debug.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [21. Inheritance, Interfaces and Abstract Classes](21-inheritance-interfaces-abstract-classes.md) | [Course outline](README.md#course-outline) | [23. Async and Error Handling](23-async-and-error-handling.md) |
