# Express with TypeScript

> **Session 31 of 33** | Part 7: TypeScript with Node.js | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Express and Its Types](#express-and-its-types)
- [From node:http to Express](#from-nodehttp-to-express)
- [What tsc, ESLint and the Tests Found](#what-tsc-eslint-and-the-tests-found)
- [Route Parameters Come from the Route Text](#route-parameters-come-from-the-route-text)
- [Handlers Written on Their Own](#handlers-written-on-their-own)
- [A Router for the Products](#a-router-for-the-products)
- [The Query String](#the-query-string)
- [The Body Is any](#the-body-is-any)
- [Errors in Express 5](#errors-in-express-5)
- [Adding req.user with declare global](#adding-requser-with-declare-global)
- [Mini Example 1: Typed Response Bodies](#mini-example-1-typed-response-bodies)
- [Mini Example 2: How Express Reads the Route Text](#mini-example-2-how-express-reads-the-route-text)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Session 30 built a product API with `node:http`. This session moves the same API to **Express**, the web framework the Node.js course uses, and adds a logged-in user. Copy the folder `30-node-api` to `31-express-api`, without `node_modules` and `dist`. Inside the new folder

```bash
npm pkg set name=31-express-api
npm install
npm install express
npm install --save-dev @types/express
```

`try-api.ts` and `src/events.ts` from Session 30 are not used here; delete them. The versions we tested: express 5.3.0 and @types/express 5.0.6.

---

## Express and Its Types

Express is written in JavaScript, as a CommonJS package. Its types come from a separate package, `@types/express` (Session 25), which brings more type packages with it

| Package                              | What it describes                                      |
| ------------------------------------ | ------------------------------------------------------ |
| `@types/express`                     | `express()`, `express.json()`, `express.Router()`      |
| `@types/express-serve-static-core`   | `Request`, `Response`, `RequestHandler`, route parameters |
| `@types/qs`                          | `ParsedQs`, the type of query strings                  |
| `@types/body-parser`, `@types/serve-static` | The middleware that comes with Express          |

`express` goes into `dependencies`, because the server runs it; `@types/express` into `devDependencies`, because only `tsc` reads it (Session 30). Without `@types/express`, `tsc` says what Session 25 showed for `ms`

```text
src/app.ts:1:39 - error TS7016: Could not find a declaration file for module 'express'. 'D:/ts-practice/31-express-api/node_modules/express/index.js' implicitly has an 'any' type.
  Try `npm i --save-dev @types/express` if it exists or add a new declaration (.d.ts) file containing `declare module 'express';`
```

followed by a TS7006 for every `req` and `res`: without types, nothing is known about them.

The import is the one Session 24 promised

```typescript
import express, { type Express } from "express";
```

`express` is the package's main function, imported as the **default import** of a CommonJS package. `Express`, the type of an app, is imported with `type` (Session 08): it exists only for `tsc` and is removed before the code runs.

![The express package is JavaScript in dependencies; @types/express in devDependencies brings express-serve-static-core, qs, serve-static and body-parser types with it; the import takes the express function as a default import and the Express type as a type import](images/31-express-with-typescript/express-types.gif)

---

## From node:http to Express

Session 30 needed `sendJson`, `readBody`, an `if...else` chain and `sendError`. Express has a part for each of them. Errors get their own file first

src/middleware/errors.ts

```typescript
import type { ErrorRequestHandler, RequestHandler } from "express";
import { AppError } from "../utils/errors.ts";
import { InvalidDataError } from "../utils/validation.ts";

export const notFound: RequestHandler = (req, res) => {
  res.status(404).json({ error: `${req.method} ${req.path} does not exist` });
};

export const errorHandler: ErrorRequestHandler = (err: unknown, req, res, next) => {
  if (err instanceof InvalidDataError) {
    res.status(400).json({ error: err.message, problems: err.problems });
  } else if (err instanceof AppError) {
    res.status(err.statusCode).json({ error: err.message });
  } else if (err instanceof SyntaxError) {
    res.status(400).json({ error: "the body is not valid JSON" });
  } else {
    console.error(err);
    res.status(500).json({ error: "something went wrong" });
  }
};
```

Then the app. Replace `src/app.ts`

src/app.ts

```typescript
import express, { type Express } from "express";
import { errorHandler, notFound } from "./middleware/errors.ts";
import { addProduct, findProduct, listProducts, newProductSchema } from "./products.ts";
import { AppError } from "./utils/errors.ts";
import { validate } from "./utils/validation.ts";

export function createApp(): Express {
  const app = express();
  app.use(express.json());

  app.get("/health", (req, res) => {
    res.json({ status: "ok" });
  });
  app.get("/products", (req, res) => {
    res.json(listProducts());
  });
  app.get("/products/:id", (req, res) => {
    const product = findProduct(Number(req.params.id));
    if (product === undefined) {
      throw new AppError(`product ${req.params.id} not found`, 404);
    }
    res.json(product);
  });
  app.post("/products", (req, res) => {
    const data = validate(newProductSchema, req.body, "product");
    res.status(201).json(addProduct(data));
  });

  app.use(notFound);
  app.use(errorHandler);
  return app;
}
```

| Session 30 (`node:http`)                     | Session 31 (Express)                                     |
| -------------------------------------------- | -------------------------------------------------------- |
| `if (method === "GET" && url.pathname === ...)` | `app.get("/products", handler)`: Express compares the method and the path |
| `url.pathname.slice("/products/".length)`    | `"/products/:id"` and `req.params.id`                    |
| `readBody(req)` with `json()`                | `app.use(express.json())`: the body is in `req.body`     |
| `sendJson(res, 201, data)`                   | `res.status(201).json(data)`                             |
| The last `else` with 404                     | `app.use(notFound)`: runs when no route answered         |
| `handle(...).catch(err => sendError(...))`   | `app.use(errorHandler)`: Express sends every error there |

`app.use(...)` adds **middleware**: a function that runs for every request that reaches it, in the order they were added. A route handler can simply `throw`, like the 404 above, and Express hands the error to the error handler. `res.status(...)` takes the status numbers of Session 16.

The handlers have no type annotations, and still `req` and `res` are fully typed: the types come from `app.get`, exactly like the `Handler` type gave them in Session 12.

![Each part of the node:http version has its Express replacement: express() for createServer, app.get with a route for the if chain, req.params for slicing the path, express.json() for reading the body, notFound and errorHandler for the 404 and sendError](images/31-express-with-typescript/http-to-express.gif)

---

## What tsc, ESLint and the Tests Found

`server.ts` and the tests are still Session 30's. Check

```bash
npm run check
```

```text
src/app.test.ts:9:22 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Argument of type '(value: void | PromiseLike<void>) => void' is not assignable to parameter of type '(error?: Error | undefined) => void'.
      Types of parameters 'value' and 'error' are incompatible.
        Type 'Error | undefined' is not assignable to type 'void | PromiseLike<void>'.
          Type 'Error' is not assignable to type 'void | PromiseLike<void>'.

9     server.listen(0, resolve);
                       ~~~~~~~

  node_modules/@types/express-serve-static-core/index.d.ts:1207:5 - The last overload is declared here.
    1207     listen(handle: any, listeningListener?: (error?: Error) => void): http.Server;
             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

src/app.test.ts:11:26 - error TS2339: Property 'address' does not exist on type 'Express'.

11   const address = server.address();
                            ~~~~~~~

src/app.test.ts:19:10 - error TS2339: Property 'close' does not exist on type 'Express'.

19   server.close();
            ~~~~~

src/server.ts:12:10 - error TS2339: Property 'close' does not exist on type 'Express'.

12   server.close(() => {
            ~~~~~


Found 4 errors in 2 files.
```

`createApp()` used to return a `Server`; now it returns an Express app. An app is not a server: `app.listen(...)` creates the server and returns it. Every place that relied on the old return type is listed, before anything runs. In `src/server.ts`

src/server.ts

```typescript
import { createApp } from "./app.ts";
import { config } from "./config/index.ts";

const app = createApp();

const server = app.listen(config.port, () => {
  console.log(`${config.env} server on http://localhost:${config.port}`);
});

function shutdown(signal: NodeJS.Signals): void {
  console.log(`${signal} received, closing the server`);
  server.close(() => {
    console.log("server closed");
  });
}

process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);
```

In the tests, an Express app is also a request handler, so `createServer` from `node:http` can take it, and the rest of the test file stays as it was

```typescript
import { createServer } from "node:http";
```

```typescript
const server = createServer(createApp());
```

`npm run check` is quiet. ESLint is not

```text
D:\ts-practice\31-express-api\src\middleware\errors.ts
  9:75  error  'next' is defined but never used  @typescript-eslint/no-unused-vars
```

Usually right, here wrong: Express recognizes an error handler by its **four** parameters, so `next` must stay (Beginner Mistakes shows what happens without it). Turn the rule off for that line, with the reason (Session 29)

```typescript
// eslint-disable-next-line @typescript-eslint/no-unused-vars -- Express finds error handlers by their four parameters
export const errorHandler: ErrorRequestHandler = (err: unknown, req, res, next) => {
```

Now the tests

```bash
npm test
```

```text
 × src/app.test.ts > the products API > refuses a body that is not JSON 14ms
   → expected { …(2) } to deeply equal { error: 'the body is not valid JSON' }

- Expected
+ Received

  {
-   "error": "the body is not valid JSON",
+   "error": "product is invalid: value is required",
+   "problems": [
+     "value is required",
+   ],
  }
```

A real difference between the two servers. That test sends its text without a `content-type` header. Session 30's `json(req)` read every body; `express.json()` reads only bodies that say they are JSON, so `req.body` stayed `undefined`, and `validate` reported a missing value. A real client sends the header, so the test should too

```typescript
    const res = await fetch(`${baseUrl}/products`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: "{ name: Webcam }"
    });
```

```text
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

The six tests from Session 30 pass against Express. This is why the tests were written: they checked that the move changed nothing, and found the one thing that did.

Start the server and try it: the answers are the ones from Session 30

```text
development server on http://localhost:3000
>>> GET /products/2
200 application/json; charset=utf-8
{"id":2,"name":"Mouse","pricePaise":49900}
>>> GET /products/99
404 application/json; charset=utf-8
{"error":"product 99 not found"}
```

---

## Route Parameters Come from the Route Text

Session 19 built `RouteParams`, a type that reads parameter names out of a route like `"/orders/:orderId"`, and said Express does the same. Here it is. Misspell the parameter in the `"/products/:id"` handler

```typescript
    const product = findProduct(Number(req.params.idd));
```

```text
src/app.ts:18:51 - error TS2339: Property 'idd' does not exist on type '{ id: string; }'.

18     const product = findProduct(Number(req.params.idd));
                                                     ~~~


Found 1 error in src/app.ts:18
```

The type of `req.params` is `{ id: string }`, made from the text `"/products/:id"`. A route with two parameters gives both, and Mini Example 2 looks at how. Put the `id` back.

![The route text gives the type of req.params: /:id gives id, two parameters give both; a handler written on its own with a plain Request gets a string, a list of strings or undefined for any name, and Request with id string brings the check back](images/31-express-with-typescript/route-params.gif)

---

## Handlers Written on Their Own

Bigger apps often write handlers as named functions and pass them to the route. Here is the product handler that way, with the same typo, in a router file (the next section explains routers)

```typescript
function getProduct(req: Request, res: Response): void {
  const product = findProduct(Number(req.params.idd));
  if (product === undefined) {
    throw new AppError(`product ${req.params.id} not found`, 404);
  }
  res.json(product);
}

productsRouter.get("/:id", getProduct);
```

`npm run check` passes. And the API is broken

```text
>>> GET /products/2
404 application/json; charset=utf-8
{"error":"product 2 not found"}
```

A plain `Request` has no route text. Its `params` is `ParamsDictionary`, an index signature (Session 07) where every name is allowed and the values are `string | string[]`. So `idd` passes, is `undefined` at run time, and `Number(undefined)` finds nothing. Tell the function which parameters it gets

```typescript
function getProduct(req: Request<{ id: string }>, res: Response): void {
```

```text
src/routes/products.ts:8:49 - error TS2339: Property 'idd' does not exist on type '{ id: string; }'.
```

`Request<{ id: string }>` is a generic type (Session 14) whose first type argument is the params. Inline handlers get this for free; named handlers need it written.

---

## A Router for the Products

A real API has many routes; Express groups them with a **router**, a small app for one part of the API. Move the product routes

src/routes/products.ts

```typescript
import express from "express";
import { addProduct, findProduct, listProducts, newProductSchema } from "../products.ts";
import { AppError } from "../utils/errors.ts";
import { validate } from "../utils/validation.ts";

export const productsRouter = express.Router();

productsRouter.get("/", (req, res) => {
  res.json(listProducts());
});

productsRouter.get("/:id", (req, res) => {
  const product = findProduct(Number(req.params.id));
  if (product === undefined) {
    throw new AppError(`product ${req.params.id} not found`, 404);
  }
  res.json(product);
});

productsRouter.post("/", (req, res) => {
  const data = validate(newProductSchema, req.body, "product");
  res.status(201).json(addProduct(data));
});
```

src/app.ts

```typescript
import express, { type Express } from "express";
import { errorHandler, notFound } from "./middleware/errors.ts";
import { productsRouter } from "./routes/products.ts";

export function createApp(): Express {
  const app = express();
  app.use(express.json());

  app.get("/health", (req, res) => {
    res.json({ status: "ok" });
  });
  app.use("/products", productsRouter);

  app.use(notFound);
  app.use(errorHandler);
  return app;
}
```

`app.use("/products", productsRouter)` mounts the router: its `"/:id"` answers `/products/:id`. The parameter types still come from the route text inside the router. `npm run check`, `npm run lint` and the six tests all pass. The folders follow the Node.js course (Session 21 of that course): routes in `routes/`, middleware in `middleware/`.

![A POST request passes express.json, the products router, requireUser and requireAdmin before the handler answers 201; a thrown 401 or 403 jumps to the error handler, and requests no route answered reach notFound](images/31-express-with-typescript/middleware-chain.gif)

---

## The Query String

Session 30's exercise filtered the products by price: `/products?maxPricePaise=50000`. Express puts the query in `req.query`. What does it contain? A route for a moment, `app.get("/echo", (req, res) => { res.json(req.query); })`, shows it

| Address                     | `req.query`                    |
| --------------------------- | ------------------------------ |
| `/echo`                     | `{}`                           |
| `/echo?tag=sale`            | `{"tag":"sale"}`               |
| `/echo?tag=sale&tag=new`    | `{"tag":["sale","new"]}`       |

The same name twice gives an array, as Session 10 warned. The type says even more. Use the value as text in the products router

```typescript
  const raw = req.query.maxPricePaise;
  console.log(raw.trim());
```

```text
src/routes/products.ts:19:15 - error TS18048: 'raw' is possibly 'undefined'.

19   console.log(raw.trim());
                 ~~~

src/routes/products.ts:19:19 - error TS2339: Property 'trim' does not exist on type 'string | (string | ParsedQs)[] | ParsedQs'.
  Property 'trim' does not exist on type '(string | ParsedQs)[]'.
```

A query value can be missing, a text, a list, or an object (`ParsedQs`, for queries like `?filter[price]=100` when Express is set to the `qs` parser; Express 5's default parser keeps `filter[price]` as a plain name). You could narrow it with `typeof` and `Array.isArray` (Session 10). Joi does it in one place, and turns the text into a number too. In the router, add `import Joi from "joi";` and

```typescript
interface ProductQuery {
  maxPricePaise?: number;
}

const querySchema = Joi.object<ProductQuery, true>({
  maxPricePaise: Joi.number().integer().min(0)
});
```

and the list route becomes

```typescript
productsRouter.get("/", (req, res) => {
  const { maxPricePaise } = validate(querySchema, req.query, "query");
  const products = listProducts();
  res.json(maxPricePaise === undefined ? products : products.filter((product) => product.pricePaise <= maxPricePaise));
});
```

```text
>>> GET /products?maxPricePaise=50000
200 application/json; charset=utf-8
[{"id":2,"name":"Mouse","pricePaise":49900}]
>>> GET /products?maxPricePaise=abc
400 application/json; charset=utf-8
{"error":"query is invalid: maxPricePaise must be a number","problems":["maxPricePaise must be a number"]}
>>> GET /products?maxPricePaise=1&maxPricePaise=2
400 application/json; charset=utf-8
{"error":"query is invalid: maxPricePaise must be a number","problems":["maxPricePaise must be a number"]}
>>> GET /products?sort=name
200 application/json; charset=utf-8
[{"id":1,"name":"Keyboard","pricePaise":149900},{"id":2,"name":"Mouse","pricePaise":49900}]
```

After `validate`, `maxPricePaise` is `number | undefined`, and the `const` keeps its narrowing inside the `filter` callback. An unknown name like `sort` is removed by `stripUnknown` (Session 26).

![A query value is one text for ?tag=sale, a list for the name given twice, and missing without it; validate with the query schema turns maxPricePaise into a number, so 50000 finds the mouse and a doubled value is a 400](images/31-express-with-typescript/query-shapes.gif)

---

## The Body Is any

`express.json()` puts the parsed body in `req.body`, and its type is `any`. Everything you do with it passes `tsc`

```typescript
    res.status(201).json(addProduct({ name: req.body.name.trim(), pricePaise: req.body.pricePaise }));
```

`npm run check` is quiet. A body without a name

```text
>>> POST /products {"pricePaise":100}
TypeError: Cannot read properties of undefined (reading 'trim')
```

and the client gets a 500. ESLint's type-aware rules (Session 29) do see the problem

```text
  25:39  error  Unsafe assignment of an `any` value                 @typescript-eslint/no-unsafe-assignment
  25:45  error  Unsafe call of an `any` typed value                 @typescript-eslint/no-unsafe-call
  25:54  error  Unsafe member access .name on an `any` value        @typescript-eslint/no-unsafe-member-access
```

The fix is what the router already does: `validate(newProductSchema, req.body, "product")`. Passing `any` into a parameter of type `unknown` is fine for ESLint, and what comes out is a checked `NewProduct`. The body can arrive three ways

| Request                                   | `req.body`             | Answer                                   |
| ----------------------------------------- | ---------------------- | ---------------------------------------- |
| JSON with `content-type: application/json` | The object (`any`)    | 201, or 400 with the problems            |
| Broken JSON with that header              | Not set: `express.json()` throws a `SyntaxError` | 400 "the body is not valid JSON" |
| No JSON `content-type`                    | `undefined`            | 400 "value is required"                  |

![express.json() reads a JSON body into req.body as any and the product is added; broken JSON makes it throw a SyntaxError, answered with 400; a body without the JSON content type is not read, and validate answers value is required](images/31-express-with-typescript/body-paths.gif)

---

## Errors in Express 5

Session 30's server stopped when an `async` handler rejected, and the fix needed a `.catch` on every request. Express 5 does that for you. A route for a moment, in `app.ts` (with `setTimeout as sleep` from `node:timers/promises`)

```typescript
  app.get("/slow-fail", async () => {
    await sleep(10);
    throw new Error("the database is down");
  });
```

```text
>>> GET /slow-fail
Error: the database is down
    at file:///D:/ts-practice/31-express-api/src/app.ts:32:11
500 application/json; charset=utf-8
{"error":"something went wrong"}
>>> GET /health
200 application/json; charset=utf-8
{"status":"ok"}
```

The rejected promise went to `errorHandler`, which logged it and answered 500, and the server kept running. Express 4 did not do this; the Node.js course and this course use Express 5. `npm run lint` is quiet too: in `@types/express` 5, a handler's return type is `unknown`, so `async` handlers are allowed where Session 30's `createServer` expected `void`. Remove the route.

The error handler in detail

| Part                                  | Why                                                          |
| ------------------------------------- | ------------------------------------------------------------ |
| `ErrorRequestHandler`                 | The type of a function with `err, req, res, next`; `req`, `res` and `next` get their types from it |
| `err: unknown`                        | The type says `any` for `err`. Writing `unknown` makes `tsc` insist on narrowing (Session 04) |
| `instanceof InvalidDataError` first   | The most specific class first, like `toResponse` in Session 21 |
| `instanceof SyntaxError`              | `express.json()` throws one for broken JSON                   |
| `console.error(err)` and 500          | An unexpected error is a bug: log the details, answer a plain message |

A new error class, like `class UnauthorizedError extends AppError` with 401, needs no change here: Session 21's promise kept in an Express app.

![The error handler with err as unknown narrows to InvalidDataError, AppError and SyntaxError before the 500; four parameters make it an error handler, while with three Express skips it and sends an HTML page with the stack trace](images/31-express-with-typescript/error-handler.gif)

![An async route rejects; in Session 30 that stopped the process, in Express 5 the rejection goes to next(err), the error handler answers 500, and the next health check still gets 200](images/31-express-with-typescript/express5-async.gif)

---

## Adding req.user with declare global

Adding products should need a logged-in admin. A middleware checks the request's token and puts the user on the request, so later handlers know who is asking

src/middleware/auth.ts

```typescript
import type { Request, RequestHandler } from "express";
import { AppError } from "../utils/errors.ts";

export interface AuthUser {
  id: number;
  name: string;
  role: "admin" | "customer";
}

// Demo tokens. A real API checks a signed token instead (Session 33)
const USERS_BY_TOKEN: Record<string, AuthUser> = {
  "token-riya": { id: 1, name: "Riya", role: "admin" },
  "token-arjun": { id: 2, name: "Arjun", role: "customer" }
};

export const requireUser: RequestHandler = (req, res, next) => {
  const token = req.get("authorization")?.replace("Bearer ", "");
  const user = token === undefined ? undefined : USERS_BY_TOKEN[token];
  if (user === undefined) {
    throw new AppError("log in first", 401);
  }
  req.user = user;
  next();
};

export const requireAdmin: RequestHandler = (req, res, next) => {
  if (req.user?.role !== "admin") {
    throw new AppError("only admins can do this", 403);
  }
  next();
};

export function currentUser(req: Request): AuthUser {
  if (req.user === undefined) {
    throw new AppError("log in first", 401);
  }
  return req.user;
}
```

`req.user` does not exist in Express's types. Session 08 showed that interfaces with the same name **merge**, and Session 25 used `declare global` to add to `process.env`. `@types/express-serve-static-core` declares an empty `interface Request {}` in a global namespace `Express`, exactly so that apps can add to it

src/types/express.d.ts

```typescript
import type { AuthUser } from "../middleware/auth.ts";

declare global {
  namespace Express {
    interface Request {
      user?: AuthUser;
    }
  }
}
```

The `import` makes this file a module, which `declare global` needs (Session 25 used `export {}` for the same reason). Now every Express request has `user?: AuthUser`. Use the middleware on the POST route

```typescript
productsRouter.post("/", requireUser, requireAdmin, (req, res) => {
```

(with `import { requireAdmin, requireUser } from "../middleware/auth.ts";` at the top). Middleware can be passed to one route like this, and runs in order before the handler. Add a route that tells who is logged in, in `app.ts`

```typescript
  app.get("/me", requireUser, (req, res) => {
    res.json({ name: req.user.name });
  });
```

```text
src/app.ts:14:22 - error TS18048: 'req.user' is possibly 'undefined'.

14     res.json({ name: req.user.name });
                        ~~~~~~~~


Found 1 error in src/app.ts:14
```

`requireUser` runs first and sets `req.user`, but the types can not see the order of the middleware: for them, `user` is optional on every request. `currentUser(req)` turns "possibly missing" into an `AuthUser` or a 401, in one place

```typescript
  app.get("/me", requireUser, (req, res) => {
    res.json(currentUser(req));
  });
```

(with `import { currentUser, requireUser } from "./middleware/auth.ts";`). Try the POST route as nobody, as Arjun (a customer) and as Riya (an admin)

```text
>>> POST /products {"name":"Webcam","pricePaise":249900}
401 application/json; charset=utf-8
{"error":"log in first"}
>>> POST /products {"name":"Webcam","pricePaise":249900}
403 application/json; charset=utf-8
{"error":"only admins can do this"}
>>> POST /products {"name":"Webcam","pricePaise":249900}
201 application/json; charset=utf-8
{"id":3,"name":"Webcam","pricePaise":249900}
>>> GET /me
200 application/json; charset=utf-8
{"id":2,"name":"Arjun","role":"customer"}
```

The tests, updated for the new rules: a helper sends a POST with a token, and five new tests cover the query, the login and `/me`

src/app.test.ts

```typescript
import { createServer } from "node:http";
import { afterAll, beforeAll, describe, expect, it } from "vitest";
import { createApp } from "./app.ts";

const server = createServer(createApp());
let baseUrl = "";

beforeAll(async () => {
  await new Promise<void>((resolve) => {
    server.listen(0, resolve);
  });
  const address = server.address();
  if (address === null || typeof address === "string") {
    throw new Error("the server has no port");
  }
  baseUrl = `http://localhost:${address.port}`;
});

afterAll(() => {
  server.close();
});

function postProduct(body: string, token: string): Promise<Response> {
  return fetch(`${baseUrl}/products`, {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${token}` },
    body
  });
}

describe("the products API", () => {
  it("answers the health check", async () => {
    const res = await fetch(`${baseUrl}/health`);
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ status: "ok" });
  });

  it("returns one product", async () => {
    const res = await fetch(`${baseUrl}/products/2`);
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ id: 2, name: "Mouse", pricePaise: 49900 });
  });

  it("answers 404 for a missing product", async () => {
    const res = await fetch(`${baseUrl}/products/99`);
    expect(res.status).toBe(404);
    expect(await res.json()).toEqual({ error: "product 99 not found" });
  });

  it("filters by price", async () => {
    const res = await fetch(`${baseUrl}/products?maxPricePaise=50000`);
    expect(await res.json()).toEqual([{ id: 2, name: "Mouse", pricePaise: 49900 }]);
  });

  it("refuses a price given twice", async () => {
    const res = await fetch(`${baseUrl}/products?maxPricePaise=1&maxPricePaise=2`);
    expect(res.status).toBe(400);
  });

  it("needs a logged-in user to add a product", async () => {
    const res = await fetch(`${baseUrl}/products`, { method: "POST" });
    expect(res.status).toBe(401);
  });

  it("lets only admins add products", async () => {
    const res = await postProduct(JSON.stringify({ name: "Cable", pricePaise: 19900 }), "token-arjun");
    expect(res.status).toBe(403);
  });

  it("adds a valid product for an admin", async () => {
    const res = await postProduct(JSON.stringify({ name: "Webcam", pricePaise: 249900 }), "token-riya");
    expect(res.status).toBe(201);
    expect(await res.json()).toEqual({ id: 3, name: "Webcam", pricePaise: 249900 });
  });

  it("refuses an invalid product and lists the problems", async () => {
    const res = await postProduct(JSON.stringify({ name: " ", pricePaise: 12.5 }), "token-riya");
    expect(res.status).toBe(400);
    expect(await res.json()).toHaveProperty("problems", ["name is not allowed to be empty", "pricePaise must be an integer"]);
  });

  it("refuses a body that is not JSON", async () => {
    const res = await postProduct("{ name: Webcam }", "token-riya");
    expect(res.status).toBe(400);
    expect(await res.json()).toEqual({ error: "the body is not valid JSON" });
  });

  it("tells who is logged in", async () => {
    const res = await fetch(`${baseUrl}/me`, { headers: { authorization: "Bearer token-arjun" } });
    expect(await res.json()).toEqual({ id: 2, name: "Arjun", role: "customer" });
  });
});
```

```text
 Test Files  1 passed (1)
      Tests  11 passed (11)
```

`Response` in `postProduct` is the global type of a `fetch` answer, not Express's `Response`: nothing from Express is imported in this file. `npm run check`, `npm run lint` and `npm run build` pass too.

![Express's empty global Request and the app's Request with user merge into one; requireUser sets req.user, the admin check reads req.user?.role, a handler that uses req.user.name gets TS18048, and currentUser returns an AuthUser or answers 401](images/31-express-with-typescript/req-user.gif)

---

## Mini Example 1: Typed Response Bodies

`req.params` has a type; the answers of a route can have one too. The route methods take type arguments: the route text, the params, and the response body. A route whose answers are a `Product` or an error, in a new router file

src/routes/typed.ts

```typescript
import express from "express";
import { findProduct, type Product } from "../products.ts";

interface ErrorBody {
  error: string;
}

export const productsRouter = express.Router();

productsRouter.get<"/:id", { id: string }, Product | ErrorBody>("/:id", (req, res) => {
  const product = findProduct(Number(req.params.id));
  if (product === undefined) {
    res.status(404).json({ message: `product ${req.params.id} not found` });
    return;
  }
  res.json({ ...product, price: product.pricePaise / 100 });
});
```

```text
src/routes/typed.ts:13:28 - error TS2353: Object literal may only specify known properties, and 'message' does not exist in type 'ErrorBody | Product'.

13     res.status(404).json({ message: `product ${req.params.id} not found` });
                              ~~~~~~~

src/routes/typed.ts:16:26 - error TS2353: Object literal may only specify known properties, and 'price' does not exist in type 'ErrorBody | Product'.

16   res.json({ ...product, price: product.pricePaise / 100 });
                            ~~~~~


Found 2 errors in the same file, starting at: src/routes/typed.ts:13
```

Two answers in the wrong shape: an error with `message` where the API promises `error`, and a product with an extra `price`. A client written against the old shape would break on either. With `{ error: ... }` and `res.json(product)`, `tsc` is quiet. The cost: the route text has to be written twice, once as a type argument. Use it for routes whose answers other teams or apps depend on.

![The get method takes the route, the params and the response body as type arguments; with Product or ErrorBody as the body, an error with message and a product with an extra price field are both refused with TS2353](images/31-express-with-typescript/typed-response.gif)

---

## Mini Example 2: How Express Reads the Route Text

Session 19 wrote `RouteParams` and promised to show the real one. Open `node_modules/@types/express-serve-static-core/index.d.ts`, and search for `RouteParameters`

```typescript
type RemoveTail<S extends string, Tail extends string> = S extends `${infer P}${Tail}` ? P : S;
type GetRouteParameter<S extends string> = RemoveTail<
    RemoveTail<RemoveTail<S, `/${string}`>, `-${string}`>,
    `.${string}`
>;

// dprint-ignore
export type RouteParameters<Route extends string | RegExp> = Route extends string
    ? Route extends `${infer Required}{${infer Optional}}${infer Next}`
        ? ParseRouteParameters<Required> & Partial<ParseRouteParameters<Optional>> & RouteParameters<Next>
        : ParseRouteParameters<Route>
    : ParamsFlatDictionary;

type ParseRouteParameters<Route extends string> = string extends Route ? ParamsDictionary
    : Route extends `${string}:${infer Rest}` ?
            & (
                GetRouteParameter<Rest> extends never ? ParamsDictionary
                    : { [P in GetRouteParameter<Rest>]: string }
            )
            & (Rest extends `${GetRouteParameter<Rest>}${infer Next}` ? RouteParameters<Next> : unknown)
    : Route extends `${string}*${infer Rest}` ?
            & (
                GetRouteParameter<Rest> extends never ? ParamsDictionary
                    : { [P in GetRouteParameter<Rest>]: string[] }
            )
            & (Rest extends `${GetRouteParameter<Rest>}${infer Next}` ? RouteParameters<Next> : unknown)
    : {};
```

Every tool is one you know: template literal types with `infer` (Session 19), conditional types (Session 18), a mapped type `[P in ...]`, `Partial` (Session 17), and recursion. For `"/orders/:orderId/items/:itemId"`

| Step                                         | Result                                   |
| -------------------------------------------- | ---------------------------------------- |
| `` `${string}:${infer Rest}` ``              | `Rest` is `"orderId/items/:itemId"`      |
| `GetRouteParameter<Rest>` cuts at `/`, `-` or `.` | `"orderId"`                          |
| `{ [P in ...]: string }`                     | `{ orderId: string }`                    |
| `RouteParameters<Next>` with `"/items/:itemId"` | `{ itemId: string }`, joined with `&` |

Express 5 also has two route forms Session 19 did not: `*name` matches several path segments and gives `string[]`, and `{...}` makes a part optional. Check what the type makes of them, with the "surely wrong value" trick from Session 18

src/route-types.ts

```typescript
import type { RouteParameters } from "express-serve-static-core";

type OneItem = RouteParameters<"/orders/:orderId/items/:itemId">;
type Download = RouteParameters<"/files/*path">;
type Report = RouteParameters<"/reports{/:format}">;

export const a: OneItem = { orderId: 1001 };
export const b: Download = { path: "docs/manual.pdf" };
export const c: Report = { format: 7 };
```

```text
src/route-types.ts:7:29 - error TS2322: Type 'number' is not assignable to type 'string'.

7 export const a: OneItem = { orderId: 1001 };
                              ~~~~~~~

src/route-types.ts:8:30 - error TS2322: Type 'string' is not assignable to type 'string[]'.

8 export const b: Download = { path: "docs/manual.pdf" };
                               ~~~~

  node_modules/@types/express-serve-static-core/index.d.ts:119:23 - The expected type comes from property 'path' which is declared here on type '{ path: string[]; }'
    119                     : { [P in GetRouteParameter<Rest>]: string[] }
                              ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

src/route-types.ts:9:28 - error TS2322: Type 'number' is not assignable to type 'string'.

9 export const c: Report = { format: 7 };
                             ~~~~~~
```

And in real routes, where the types meet the values

src/routes/files.ts

```typescript
import express from "express";

export const filesRouter = express.Router();

filesRouter.get("/files/*path", (req, res) => {
  res.json({ segments: req.params.path, joined: req.params.path.join("/") });
});

filesRouter.get("/reports{/:format}", (req, res) => {
  res.json({ format: req.params.format ?? "json" });
});
```

With `app.use(filesRouter)` in `app.ts`

```text
>>> GET /files/docs/manual.pdf
200 application/json; charset=utf-8
{"segments":["docs","manual.pdf"],"joined":"docs/manual.pdf"}
>>> GET /reports
200 application/json; charset=utf-8
{"format":"json"}
>>> GET /reports/csv
200 application/json; charset=utf-8
{"format":"csv"}
```

`.join` is allowed because `path` is `string[]`, and `?? "json"` because `format` may be missing: both come from the route text alone.

![RouteParameters takes the route apart: the template literal with infer finds the rest after the colon, GetRouteParameter cuts it to orderId, a mapped type makes the object, and recursion adds itemId; star routes give string arrays and braces make parameters optional](images/31-express-with-typescript/route-text.gif)

---

## Beginner Mistakes

### Mistake 1: Installing express without @types/express

`tsc` reports TS7016 for the import and TS7006 for every `req` and `res`. Install the types as a dev dependency: `npm install --save-dev @types/express`.

---

### Mistake 2: An error handler with three parameters

ESLint says `next` is unused, so it is tempting to delete it. Then Express sees an ordinary middleware, skips it for errors, and answers with its own page, stack trace included

```text
<pre>AppError: product 99 not found<br> &nbsp; &nbsp;at file:///D:/ts-practice/31-express-api/src/app.ts:20:13<br> &nbsp; &nbsp;at Layer.handleRequest (D:\ts-practice\31-express-api\node_modules\router\lib\layer.js:152:17)<br> &nbsp; &nbsp;at next (D:\ts-practice\31-express-api\node_modules\router\lib\route.js:157:13)<br> &nbsp; &nbsp;at Route.dispatch (D:\ts-practice\31-express-api\node_modules\router\lib\route.js:117:3)<br> &nbsp; &nbsp;at handle (D:\ts-practice\31-express-api\node_modules\router\index.js:435:11)<br> &nbsp; &nbsp;at Layer.handleRequest (D:\ts-practice\31-express-api\node_modules\router\lib\layer.js:152:17)<br> &nbsp; &nbsp;at D:\ts-practice\31-express-api\node_modules\router\index.js:295:15<br> &nbsp; &nbsp;at param (D:\ts-practice\31-express-api\node_modules\router\index.js:600:14)<br> &nbsp; &nbsp;at param (D:\ts-practice\31-express-api\node_modules\router\index.js:610:14)<br> &nbsp; &nbsp;at processParams (D:\ts-practice\31-express-api\node_modules\router\index.js:664:3)</pre>
```

`tsc` does not notice: a function with fewer parameters fits the type. Keep all four, with the disable comment and its reason.

---

### Mistake 3: Using req.body directly

`req.body` is `any`, so `req.body.name.trim()` passes `tsc` and crashes for a body without a name. Validate it first; ESLint's `no-unsafe-*` rules point at every direct use.

---

### Mistake 4: A named handler with a plain Request

`function getProduct(req: Request, ...)` gives `req.params` an index signature: a misspelled name passes and is `undefined`. Write the params: `Request<{ id: string }>`.

---

### Mistake 5: Trusting that a middleware set req.user

The types do not know which middleware ran. `req.user.name` gives TS18048 even after `requireUser`. Use a helper like `currentUser(req)`, or check `req.user` where it is used. Do not silence it with `!` (Session 11).

---

### Mistake 6: A declaration file that is not a module

Write `src/types/express.d.ts` without the `import` and without `export {}`, and `declare global` is ignored without a message (Session 25). The result is TS2339 on every `req.user`

```text
src/middleware/auth.ts:22:7 - error TS2339: Property 'user' does not exist on type 'Request<ParamsDictionary, any, any, ParsedQs, Record<string, any>>'.
```

---

### Mistake 7: Sending JSON without the content-type header

`express.json()` only reads bodies that say `content-type: application/json`. Without the header, `req.body` is `undefined` and every POST answers "value is required". Clients, scripts and tests must send it.

---

### Mistake 8: Treating createApp() as the server

An Express app is a request handler with a `listen` method; the server is what `app.listen(...)` returns. Keep that return value for `server.close()`, and in tests use `createServer(createApp())` or the value of `app.listen`.

---

### Mistake 9: Using a query value as a string

`req.query.x` is `string | (string | ParsedQs)[] | ParsedQs | undefined`, and the same name twice really gives an array. Validate the query with Joi, or narrow with `typeof` and `Array.isArray`.

---

### Mistake 10: Using res for the wrong Response

In a file that imports Express's `Response` type, the name no longer means the `fetch` answer. Keep Express types in the files that use Express, and in mixed files import them with clear names, for example `import type { Response as ExpressResponse } from "express"`.

---

## Practice Exercises

### Exercise 1

Add `DELETE /products/:id` to the router, only for admins. Answer 204 with `res.status(204).end()` when the product existed and 404 when it did not. Write the tests first.

### Exercise 2

Add a query `?q=mou` to `GET /products` that finds products whose name contains the text (ignoring case). Add it to `ProductQuery` and the schema with `Joi.string().trim().min(2)`. What does `?q=a` answer?

### Exercise 3

Write the handler of `GET /products/:id` as a named function `getProduct` with the right `Request` type, and pass it to the router. Then use `RequestHandler<{ id: string }>` as the type of a `const` arrow function instead: which version do you find easier to read?

### Exercise 4

Add `class ForbiddenError extends AppError` with 403 to `src/utils/errors.ts` (Session 21), and use it in `requireAdmin`. Does `errorHandler` need a change? Why not?

### Exercise 5

Give `AuthUser` a new property `email: string`, and add it to the two demo users. Which files does `tsc` send you to? Then add `user?: AuthUser` a second time in `express.d.ts` with a different type and read the error.

### Exercise 6

Type the answers of `GET /products` with the type arguments of Mini Example 1: `Product[] | ErrorBody`. Then try `res.json(listProducts()[0])`. What does `tsc` say, and why?

### Exercise 7

Use the "surely wrong value" trick on `RouteParameters<"/users/:userId/orders/:orderId.:format">`. Which names does it find, and where does `GetRouteParameter` cut `orderId.:format`?

### Exercise 8

Remove `app.use(express.json())` and run the tests. Which tests fail, and with which answers? Put it back.

---

## Interview Questions

### Where do the types of Express come from?

From `@types/express`, installed as a dev dependency. It depends on `@types/express-serve-static-core` (requests, responses, handlers, route parameters), `@types/qs` and others. Express itself is JavaScript.

### Why do inline route handlers need no type annotations?

`app.get(path, handler)` declares the handler's type, so `req`, `res` and `next` get their types from it (contextual typing). That includes `req.params`, which is read from the route text.

### What is the type of req.params, and where does it come from?

For an inline handler, it is computed from the route string by `RouteParameters`, a type built with template literal types, `infer`, conditional and mapped types and recursion: `"/products/:id"` gives `{ id: string }`, `*path` gives `string[]`, and `{/:format}` an optional property. A plain `Request` gets an index signature instead.

### What is the type of req.body, and how do you make it safe?

`any`, because Express can not know what a client sends. Validate it with a schema, which returns a typed value or throws an error that becomes a 400. Writing `Request<..., ..., NewProduct>` only states a type; it checks nothing.

### Why is a query value typed string | string[] | ... instead of string?

The same name can appear several times (`?tag=a&tag=b` gives an array), and some query parsers create objects. Validate or narrow the value before using it.

### How do you add a property like user to Express's Request?

With declaration merging: in a `.d.ts` file that is a module, write `declare global { namespace Express { interface Request { user?: AuthUser } } }`. The interface merges with the empty one Express's types declare for this purpose.

### Why is req.user still possibly undefined after an auth middleware?

The types describe every request, and they can not see in which order middleware runs. Read it through a helper that checks it and throws a 401, or check it where it is used.

### How does Express recognize error-handling middleware?

By its four parameters (`err, req, res, next`). With three, it is treated as normal middleware and skipped for errors, even though the TypeScript type accepts it.

### What changed for async handlers in Express 5?

A rejected promise from a handler is passed to the error middleware, like a thrown error. In Express 4, it was an unhandled rejection unless each handler caught it.

### What is the type of err in an Express error handler?

`ErrorRequestHandler` types it as `any`. Annotate it as `unknown` and narrow it with `instanceof`, most specific class first.

### How do you type the response body of a route?

With the route method's type arguments, for example `router.get<"/:id", { id: string }, Product | ErrorBody>(...)`, or with `Response<Product | ErrorBody>` on a named handler. `res.json` then refuses objects in another shape.

### How do you test an Express app without a fixed port?

Turn it into a server with `createServer(app)` from `node:http` (an Express app is a request handler), listen on port 0, read the port from `server.address()`, and send requests with `fetch`.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [30. Node.js with TypeScript](30-nodejs-with-typescript.md) | [Course outline](README.md#course-outline) | [32. MongoDB and Mongoose with TypeScript](32-mongoose-with-typescript.md) |
