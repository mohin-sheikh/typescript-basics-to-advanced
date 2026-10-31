# Node.js with TypeScript

> **Session 30 of 33** | Part 7: TypeScript with Node.js | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Development and Production](#development-and-production)
- [The Project Layout](#the-project-layout)
- [Settings First](#settings-first)
- [What the Node.js Types Tell You](#what-the-nodejs-types-tell-you)
- [A Server with node:http](#a-server-with-nodehttp)
- [Reading and Checking a Request Body](#reading-and-checking-a-request-body)
- [One Bad Request Should Not Stop the Server](#one-bad-request-should-not-stop-the-server)
- [Development with node --watch](#development-with-node---watch)
- [Shutting Down Cleanly](#shutting-down-cleanly)
- [Testing the Server with fetch](#testing-the-server-with-fetch)
- [Building and Starting for Production](#building-and-starting-for-production)
- [Mini Example 1: Typed Events with EventEmitter](#mini-example-1-typed-events-with-eventemitter)
- [Mini Example 2: A Production Folder Without TypeScript](#mini-example-2-a-production-folder-without-typescript)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Part 7 builds a typed backend. This session starts it with Node.js alone: a small product API with the built-in `node:http` module, settings, tests, and a build for production. Session 31 moves the same API to Express.

The project gets its own folder and `package.json`, with the tools from Session 29. Inside `ts-practice`

```bash
mkdir 30-node-api
cd 30-node-api
npm init -y
npm pkg set type=module
npm install joi
npm install --save-dev @typescript/native@npm:typescript@7.0.2 typescript@npm:@typescript/typescript6 @types/node@24
npm install --save-dev vitest eslint @eslint/js typescript-eslint
```

Then copy five files you already have

| Copy                       | From                                   |
| -------------------------- | -------------------------------------- |
| `tsconfig.json`            | `27-api` (Session 27)                  |
| `tsconfig.build.json`      | `29-quality` (Session 29)              |
| `eslint.config.js`         | `29-quality` (Session 29)              |
| `src/utils/errors.ts`      | `29-quality/src/errors.ts` (`AppError`, Session 23) |
| `src/utils/validation.ts`  | `29-quality/src/validation.ts` (`validate()`, Session 26) |

---

## Development and Production

Since Session 02 you have run `.ts` files directly with `node`, and Session 02 said a server in production is different: it gets a build. Here is why, and how both ways look in one project

| | Development | Production |
| --- | --- | --- |
| What runs | `src/server.ts`, directly | `dist/server.js`, built by `tsc` |
| How it starts | `npm run dev`: restarts on every save | `npm start` |
| Settings from | a `.env` file | the environment of the server (or a `.env` there) |
| Installed packages | everything, tools included | only `dependencies`: here, `joi` |

Why not run the `.ts` files on the server too? Three reasons

| Reason                          | Explanation                                                   |
| ------------------------------- | ------------------------------------------------------------- |
| Nothing ships unchecked         | `npm run build` runs `tsc`, and with `noEmitOnError` (Session 27) code with type errors produces no `dist` at all |
| The server needs fewer packages | TypeScript, Vitest and ESLint stay on your computer (Mini Example 2) |
| It runs on any Node.js          | Node.js versions before 22 can not run `.ts` files; plain JavaScript runs everywhere |

Replace the `"scripts"` part of `package.json` with

```json
  "scripts": {
    "dev": "node --watch --env-file-if-exists=.env src/server.ts",
    "check": "tsc --noEmit",
    "lint": "eslint src",
    "test": "vitest run --reporter=verbose",
    "build": "tsc -p tsconfig.build.json",
    "start": "node --env-file-if-exists=.env --enable-source-maps dist/server.js"
  },
```

| Script           | When                                                          |
| ---------------- | ------------------------------------------------------------- |
| `npm run dev`    | While you write code                                          |
| `npm run check`, `npm run lint`, `npm test` | Before every commit (Session 29)   |
| `npm run build`  | Before you deploy                                             |
| `npm start`      | On the server                                                 |

`--env-file-if-exists` is like `--env-file` (Session 10), but does not fail when there is no `.env`. On a server, the settings usually come from the environment, not from a file.

![Development runs npm run dev, which starts node --watch with the env file on src/server.ts; production runs npm run build, gets dist with JavaScript and source maps, installs only joi with npm ci --omit=dev, and starts dist/server.js with npm start](images/30-nodejs-with-typescript/dev-vs-prod.gif)

---

## The Project Layout

Each file has one job. The folders follow the Node.js course's projects, so Session 33 can bring that course's API into the same shape

| File                       | Job                                                      |
| -------------------------- | -------------------------------------------------------- |
| `src/server.ts`            | Starts listening and shuts down. The only file that starts anything |
| `src/app.ts`               | Turns requests into JSON answers: `createApp()`          |
| `src/products.ts`          | The product data, its types and its Joi schema           |
| `src/config/index.ts`      | The settings, checked at startup                         |
| `src/utils/validation.ts`  | `validate()` and `InvalidDataError` (Session 26)         |
| `src/utils/errors.ts`      | `AppError` (Session 23)                                  |
| `src/app.test.ts`          | Tests that talk to the server with `fetch`               |

`app.ts` builds the server, `server.ts` starts it. Keeping them apart lets the tests start the same server on a port of their own.

![The seven files with their jobs, and the import graph: server.ts imports app.ts and config, app.ts imports products.ts and utils, config imports utils; the tests import app.ts, not server.ts](images/30-nodejs-with-typescript/project-layout.gif)

---

## Settings First

The settings module is the one from Session 28's Mini Example 2, with only what this server needs

src/config/index.ts

```typescript
import Joi from "joi";
import { validate } from "../utils/validation.ts";

const NODE_ENVS = ["development", "production", "test"] as const;

interface Env {
  NODE_ENV: (typeof NODE_ENVS)[number];
  PORT: number;
}

const envSchema = Joi.object<Env, true>({
  NODE_ENV: Joi.string().valid(...NODE_ENVS).default("development"),
  PORT: Joi.number().port().default(3000)
});

const env = validate(envSchema, process.env, "environment");

export const config = {
  env: env.NODE_ENV,
  port: env.PORT
};
```

Three small files at the project root

.env

```text
PORT=3000
```

.env.example

```text
# Copy this file to .env and change the values
NODE_ENV=development
PORT=3000
```

.gitignore

```text
node_modules
dist
.env
```

`.env` holds the settings of **your** computer and may hold secrets later, so it is never committed. `.env.example` is committed: it shows a new team member which settings exist. A first `src/server.ts` that only prints the settings

```typescript
import { config } from "./config/index.ts";

console.log(config);
```

```bash
node --env-file-if-exists=.env src/server.ts
```

```text
{ env: 'development', port: 3000 }
```

With `PORT=abc` in `.env`, the program stops before it does anything else

```text
InvalidDataError: environment is invalid: PORT must be a number
```

---

## What the Node.js Types Tell You

The types of Node.js itself come from `@types/node` (Session 25), version 24 for Node.js 24, and `"types": ["node"]` in `tsconfig.json` turns them on. They are worth reading, because they say which values you have to check. A scratch file to see one

src/scratch.ts

```typescript
import { readFile } from "node:fs/promises";

const bytes = await readFile("package.json");
const text = await readFile("package.json", "utf8");

console.log(text.toUpperCase().slice(0, 20));
console.log(bytes.toUpperCase());
```

```text
src/scratch.ts:7:19 - error TS2339: Property 'toUpperCase' does not exist on type 'NonSharedBuffer'.

7 console.log(bytes.toUpperCase());
                    ~~~~~~~~~~~


Found 1 error in src/scratch.ts:7
```

`readFile` has two overloads (Session 12): without an encoding it returns a `Buffer`, the raw bytes (`NonSharedBuffer` is the name `@types/node` gives that kind of `Buffer`); with `"utf8"` it returns a `string`. Delete `scratch.ts`. The types this session meets

| Code                                       | Type                            | What it means for you                |
| ------------------------------------------ | ------------------------------- | ------------------------------------ |
| `readFile(path)` / `readFile(path, "utf8")` | `Buffer` / `string`            | Pass the encoding to get text        |
| `req.url`, `req.method`                    | `string \| undefined`           | Give them a fallback before use      |
| `json(req)` from `node:stream/consumers`   | `Promise<unknown>`              | Validate the body (Session 26)       |
| `server.address()`                         | `string \| AddressInfo \| null` | Narrow it before reading `.port`     |
| `process.env.PORT`                         | `string \| undefined`           | Validate the settings                |

![Six Node.js APIs and their types: readFile without an encoding gives a Buffer, with utf8 a string; req.url and req.method, json(req), server.address() and process.env.PORT have types that must be checked before use](images/30-nodejs-with-typescript/node-types.gif)

---

## A Server with node:http

The products, with the type, the schema for new products, and three functions. `Omit` (Session 17) makes the type of a new product from `Product`

src/products.ts

```typescript
import Joi from "joi";

export interface Product {
  id: number;
  name: string;
  pricePaise: number;
}

export type NewProduct = Omit<Product, "id">;

export const newProductSchema = Joi.object<NewProduct, true>({
  name: Joi.string().trim().min(2).required(),
  pricePaise: Joi.number().integer().min(0).required()
});

const products: Product[] = [
  { id: 1, name: "Keyboard", pricePaise: 149900 },
  { id: 2, name: "Mouse", pricePaise: 49900 }
];
let nextId = 3;

export function listProducts(): Product[] {
  return products;
}

export function findProduct(id: number): Product | undefined {
  return products.find((product) => product.id === id);
}

export function addProduct(data: NewProduct): Product {
  const product = { id: nextId++, ...data };
  products.push(product);
  return product;
}
```

The server. `node:http` calls a function for every request, with two objects: `req` (the request) and `res` (the answer)

src/app.ts

```typescript
import { createServer, type IncomingMessage, type Server, type ServerResponse } from "node:http";
import { findProduct, listProducts } from "./products.ts";

function sendJson(res: ServerResponse, statusCode: number, body: unknown): void {
  res.writeHead(statusCode, { "content-type": "application/json" });
  res.end(JSON.stringify(body));
}

function handle(req: IncomingMessage, res: ServerResponse): void {
  const method = req.method ?? "GET";
  const url = new URL(req.url ?? "/", "http://localhost");

  if (method === "GET" && url.pathname === "/health") {
    sendJson(res, 200, { status: "ok" });
  } else if (method === "GET" && url.pathname === "/products") {
    sendJson(res, 200, listProducts());
  } else if (method === "GET" && url.pathname.startsWith("/products/")) {
    const id = url.pathname.slice("/products/".length);
    const product = findProduct(Number(id));
    if (product === undefined) {
      sendJson(res, 404, { error: `product ${id} not found` });
    } else {
      sendJson(res, 200, product);
    }
  } else {
    sendJson(res, 404, { error: `${method} ${url.pathname} does not exist` });
  }
}

export function createApp(): Server {
  return createServer(handle);
}
```

| Part                                    | Meaning                                                      |
| --------------------------------------- | ------------------------------------------------------------ |
| `IncomingMessage`, `ServerResponse`     | The types of `req` and `res`, from `@types/node`. Imported with `type` (Session 08) |
| `createServer(handle)`                  | `handle` runs for every request; it fits the type `node:http` expects |
| `res.writeHead(status, headers)`        | The status code and headers of the answer                    |
| `res.end(text)`                         | Sends the body and finishes the answer                       |
| `req.method ?? "GET"`, `req.url ?? "/"` | Both are `string \| undefined` in the types, so they get a fallback |
| `new URL(req.url, "http://localhost")`  | Splits the path from a query like `?q=mouse`; the second argument is only a base for the parsing |
| `sendJson(res, status, body: unknown)`  | One helper for every answer: any value goes in, JSON comes out |

src/server.ts

```typescript
import { createApp } from "./app.ts";
import { config } from "./config/index.ts";

const server = createApp();

server.listen(config.port, () => {
  console.log(`${config.env} server on http://localhost:${config.port}`);
});
```

```bash
node --env-file-if-exists=.env src/server.ts
```

```text
development server on http://localhost:3000
```

The program keeps running: it is waiting for requests. Open `http://localhost:3000/products/2` in the browser

```text
{"id":2,"name":"Mouse","pricePaise":49900}
```

And the other routes

| Address                 | Status | Body                                              |
| ----------------------- | ------ | ------------------------------------------------- |
| `/health`               | 200    | `{"status":"ok"}`                                  |
| `/products`             | 200    | `[{"id":1,"name":"Keyboard",...},{"id":2,...}]`    |
| `/products/99`          | 404    | `{"error":"product 99 not found"}`                 |
| `/products/abc`         | 404    | `{"error":"product abc not found"}`                |
| `DELETE /products/1`    | 404    | `{"error":"DELETE /products/1 does not exist"}`    |

Stop the server with Ctrl+C. `npm run check` and `npm run lint` are both quiet. The Node.js course explains `node:http` in more depth (Sessions 09 and 10).

![GET /products/2 goes into handle, which reads the method and the path and answers with sendJson 200 and the mouse; GET /products/99 goes the same way and gets sendJson 404 with product 99 not found](images/30-nodejs-with-typescript/request-flow.gif)

---

## Reading and Checking a Request Body

A `POST /products` request carries a new product as JSON in its **body**. Node.js delivers a body in pieces, as a stream. `json()` from `node:stream/consumers` collects the pieces and parses them, and its return type is honest: `Promise<unknown>`. Change the imports of `app.ts`

```typescript
import { createServer, type IncomingMessage, type Server, type ServerResponse } from "node:http";
import { json } from "node:stream/consumers";
import { addProduct, findProduct, listProducts, newProductSchema } from "./products.ts";
import { AppError } from "./utils/errors.ts";
import { validate } from "./utils/validation.ts";
```

add a function below `sendJson`, for bodies that are not even JSON

```typescript
async function readBody(req: IncomingMessage): Promise<unknown> {
  try {
    return await json(req);
  } catch (err) {
    throw new AppError("the body is not valid JSON", 400, { cause: err });
  }
}
```

and make `handle` async, with a new branch before the last `else`

```typescript
async function handle(req: IncomingMessage, res: ServerResponse): Promise<void> {
```

```typescript
  } else if (method === "POST" && url.pathname === "/products") {
    const data = validate(newProductSchema, await readBody(req), "product");
    sendJson(res, 201, addProduct(data));
  } else {
```

`validate` turns the `unknown` body into a `NewProduct` (Session 26), or throws an `InvalidDataError`. `npm run check` is quiet.

The browser only sends `GET` requests. To send `POST` requests, use a small script at the project root, outside `src`. It is not part of the app, so the build ignores it, and `node` runs it like any `.ts` file

try-api.ts

```typescript
const bodies = [
  '{"name":"Webcam","pricePaise":249900}',
  '{"name":" ","pricePaise":12.5,"stock":4}',
  "{ name: Webcam }"
];

for (const body of bodies) {
  try {
    const res = await fetch("http://localhost:3000/products", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body
    });
    console.log(res.status, await res.text());
  } catch (err) {
    console.log("no answer:", err instanceof Error ? err.message : err);
  }
}
```

Start the server in one terminal, and run the script in a second one

```bash
node try-api.ts
```

```text
201 {"id":3,"name":"Webcam","pricePaise":249900}
no answer: fetch failed
no answer: fetch failed
```

The webcam was added. Then nothing answered at all. The first terminal shows why

```text
development server on http://localhost:3000
file:///D:/ts-practice/30-node-api/src/utils/validation.ts:22
    throw new InvalidDataError(what, result.error.details.map((detail) => detail.message));
          ^

InvalidDataError: product is invalid: name is not allowed to be empty; pricePaise must be an integer
    at validate (file:///D:/ts-practice/30-node-api/src/utils/validation.ts:22:11)
    at Server.handle (file:///D:/ts-practice/30-node-api/src/app.ts:37:18)
    at process.processTicksAndRejections (node:internal/process/task_queues:104:5) {
  problems: [
    'name is not allowed to be empty',
    'pricePaise must be an integer'
  ]
}

Node.js v24.19.0
```

![A request body arrives as unknown from json(req); validate turns the webcam into a NewProduct and a 201 answer, the product with an empty name and 12.5 paise into an InvalidDataError with two problems, and the text that is not JSON fails in json() already](images/30-nodejs-with-typescript/body-validation.gif)

---

## One Bad Request Should Not Stop the Server

The validation worked: it found both problems. But nobody **caught** the error. `handle` is `async`, so the error became a rejected promise, and `createServer` does not wait for promises: it calls `handle` and moves on. A rejected promise that nobody handles stops Node.js (Session 23). One customer sent a bad product, and the shop was offline for everybody.

`tsc` had no complaint: an `async` function fits where a function returning nothing is expected. ESLint knows this mistake (Session 29)

```bash
npm run lint
```

```text
D:\ts-practice\30-node-api\src\app.ts
  45:23  error  Promise returned in function argument where a void return was expected  @typescript-eslint/no-misused-promises
```

The fix is one place that turns **every** error into an answer. Add `InvalidDataError` to the import from `validation.ts`

```typescript
import { InvalidDataError, validate } from "./utils/validation.ts";
```

add a function below `handle` that picks the answer for an error, narrowing the `unknown` error with `instanceof` (Session 10)

```typescript
function sendError(res: ServerResponse, err: unknown): void {
  if (err instanceof InvalidDataError) {
    sendJson(res, 400, { error: err.message, problems: err.problems });
  } else if (err instanceof AppError) {
    sendJson(res, err.statusCode, { error: err.message });
  } else {
    console.error(err);
    sendJson(res, 500, { error: "something went wrong" });
  }
}
```

and catch the promise in `createApp`

```typescript
export function createApp(): Server {
  return createServer((req, res) => {
    handle(req, res).catch((err: unknown) => {
      sendError(res, err);
    });
  });
}
```

| Error                       | Answer                                    | Why                                      |
| --------------------------- | ----------------------------------------- | ---------------------------------------- |
| `InvalidDataError`          | 400 with the list of `problems`           | The client sent bad data and can fix it  |
| `AppError`                  | Its own `statusCode`, like 400 for bad JSON | An error the code expected            |
| Anything else               | 500 "something went wrong", and a log line | A bug: the details stay in the server's log, not in the answer |

Run the script again

```text
201 {"id":3,"name":"Webcam","pricePaise":249900}
400 {"error":"product is invalid: name is not allowed to be empty; pricePaise must be an integer","problems":["name is not allowed to be empty","pricePaise must be an integer"]}
400 {"error":"the body is not valid JSON"}
```

Two clear answers instead of a crash, and `/health` in the browser still says `{"status":"ok"}`. `npm run check` and `npm run lint` are quiet again.

![With createServer(handle) and no catch, a valid POST gets 201, an invalid one ends in an InvalidDataError nobody catches, and the next request finds no server; with handle(req, res).catch(...), the invalid POST gets a 400 and the health check a 200](images/30-nodejs-with-typescript/async-crash.gif)

---

## Development with node --watch

Stopping and starting the server after every change gets old fast. `npm run dev` runs `node --watch`, which restarts the program whenever a file it uses changes

```bash
npm run dev
```

Change the mouse's price in `src/products.ts` to `44900` and save. Node.js clears the terminal and starts again

```text
Restarting 'src/server.ts'
development server on http://localhost:3000
```

`/products/2` now shows `"pricePaise":44900`. Put the price back to `49900`.

| Older JavaScript setups use | Node.js 24 has            |
| --------------------------- | ------------------------- |
| `nodemon`                   | `node --watch`            |
| `dotenv`                    | `--env-file`, `--env-file-if-exists` |
| `ts-node` or `tsx`          | `node file.ts` (type stripping, Session 02) |

The dev server does not check types: like every `node file.ts`, it only removes them. Keep `npm run check` for that, or an editor that shows the errors.

![Saving a new mouse price in products.ts makes npm run dev print Restarting src/server.ts and start the server again, and GET /products/2 answers with the new price 44900](images/30-nodejs-with-typescript/watch-mode.gif)

---

## Shutting Down Cleanly

When a server is stopped, requests may still be running. A server that stops **cleanly** stops taking new requests, lets the open ones finish, and then ends. Node.js tells your program it should stop with a **signal**: `SIGINT` when you press Ctrl+C, and `SIGTERM` when a hosting platform stops your app. The final `server.ts`

src/server.ts

```typescript
import { createApp } from "./app.ts";
import { config } from "./config/index.ts";

const server = createApp();

server.listen(config.port, () => {
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

Start it with `node --env-file-if-exists=.env src/server.ts` and press Ctrl+C

```text
development server on http://localhost:3000
SIGINT received, closing the server
server closed
```

| Part                       | Meaning                                                      |
| -------------------------- | ------------------------------------------------------------ |
| `NodeJS.Signals`           | A union of all signal names, from `@types/node` (the `NodeJS` namespace of Session 25) |
| `process.on("SIGINT", ...)` | Your function replaces the default reaction, which ends the program at once |
| `server.close(callback)`   | No new connections; the callback runs when the open ones are done |
| No `process.exit()`        | When the server is closed and nothing else is waiting, Node.js ends by itself, with exit code 0 |

Windows has no real `SIGTERM`, but listening for it is allowed, and the same code works on the Linux servers where most apps run.

![Ctrl+C sends SIGINT and a hosting platform sends SIGTERM; both call shutdown, which closes the server, lets open requests finish, prints server closed, and Node.js exits with code 0](images/30-nodejs-with-typescript/graceful-shutdown.gif)

---

## Testing the Server with fetch

Clicking through the browser is not a test. Vitest (Session 29) can start the real server and send it requests with `fetch`

src/app.test.ts

```typescript
import { afterAll, beforeAll, describe, expect, it } from "vitest";
import { createApp } from "./app.ts";

const server = createApp();
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

  it("adds a valid product", async () => {
    const res = await fetch(`${baseUrl}/products`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ name: "Webcam", pricePaise: 249900 })
    });
    expect(res.status).toBe(201);
    expect(await res.json()).toEqual({ id: 3, name: "Webcam", pricePaise: 249900 });
  });

  it("refuses an invalid product and lists the problems", async () => {
    const res = await fetch(`${baseUrl}/products`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ name: " ", pricePaise: 12.5 })
    });
    expect(res.status).toBe(400);
    expect(await res.json()).toHaveProperty("problems", ["name is not allowed to be empty", "pricePaise must be an integer"]);
  });

  it("refuses a body that is not JSON", async () => {
    const res = await fetch(`${baseUrl}/products`, { method: "POST", body: "{ name: Webcam }" });
    expect(res.status).toBe(400);
    expect(await res.json()).toEqual({ error: "the body is not valid JSON" });
  });
});
```

| Part                                   | Meaning                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| `beforeAll` / `afterAll`               | Run once before the first test and after the last one of the file |
| `server.listen(0, resolve)`            | Port 0 means "any free port", so the tests never clash with a running dev server. `new Promise<void>` (Session 23) waits until it listens |
| `server.address()`                     | Which port the system picked                                 |
| `await res.json()`                     | `unknown` with these settings, which `toEqual` accepts       |

Why the `if` around `server.address()`? Its type is `string | AddressInfo | null`: `null` before the server listens, a string for servers on a local socket file. Without the check

```text
src/address-try.ts:5:13 - error TS2531: Object is possibly 'null'.

5 console.log(server.address().port);
              ~~~~~~~~~~~~~~~~

src/address-try.ts:5:30 - error TS2339: Property 'port' does not exist on type 'string | AddressInfo'.
  Property 'port' does not exist on type 'string'.
```

After the `if`, only `AddressInfo` is left, and `.port` is a `number`.

```bash
npm test
```

```text
 ✓ src/app.test.ts > the products API > answers the health check 35ms
 ✓ src/app.test.ts > the products API > returns one product 3ms
 ✓ src/app.test.ts > the products API > answers 404 for a missing product 7ms
 ✓ src/app.test.ts > the products API > adds a valid product 7ms
 ✓ src/app.test.ts > the products API > refuses an invalid product and lists the problems 12ms
 ✓ src/app.test.ts > the products API > refuses a body that is not JSON 13ms

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

The tests import `app.ts`, never `server.ts`: that file would start a second server on port 3000 and register the signal handlers.

![The tests call server.listen(0), the system picks a free port, server.address() is narrowed from string, AddressInfo or null to AddressInfo, and six fetch tests pass against baseUrl before afterAll closes the server](images/30-nodejs-with-typescript/server-test.gif)

---

## Building and Starting for Production

`tsconfig.build.json` from Session 29 leaves the tests out of the build

tsconfig.build.json

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

```bash
npm run build
npm start
```

```text
> 30-node-api@1.0.0 start
> node --env-file-if-exists=.env --enable-source-maps dist/server.js

development server on http://localhost:3000
```

`dist` now has `server.js`, `app.js`, `products.js`, `config/index.js`, `utils/errors.js` and `utils/validation.js`, each with its `.js.map`, and no test file. `--enable-source-maps` makes stack traces show your `.ts` lines (Session 29).

On the server, `npm start` is all that runs, so every package that code **imports** has to be in `dependencies`

| Section in `package.json` | Packages here             | Installed on the server |
| ------------------------- | ------------------------- | ----------------------- |
| `dependencies`            | `joi`                     | Yes                     |
| `devDependencies`         | TypeScript (both), `@types/node`, Vitest, ESLint | No |

`npm install joi` put `joi` in `dependencies`, and `--save-dev` put the tools in `devDependencies`. The difference matters only on the server, and Mini Example 2 shows it.

---

## Mini Example 1: Typed Events with EventEmitter

Sometimes one part of a program should react to something that happened in another, without the two importing each other: for example, logging every new product. Node.js has `EventEmitter` for that (Node.js course, Session 08), and `@types/node` lets you describe its events with a type argument: the event names and, for each, a tuple (Session 06) of the values it carries

src/events.ts

```typescript
import { EventEmitter } from "node:events";
import type { Product } from "./products.ts";

interface ShopEvents {
  "product:added": [product: Product];
  "product:missing": [id: string];
}

export const events = new EventEmitter<ShopEvents>();
```

Every `emit` and every listener is now checked against that map. Some mistakes, in a scratch file

src/events-try.ts

```typescript
import { events } from "./events.ts";

events.emit("product:added", { id: 3, name: "Webcam" });
events.emit("product:missing");
events.emit("product:removed", "1");
events.on("product:missing", (id: number) => {
  console.log(id);
});
```

```text
src/events-try.ts:3:30 - error TS2741: Property 'pricePaise' is missing in type '{ id: number; name: string; }' but required in type 'Product'.
src/events-try.ts:4:8 - error TS2554: Expected 2 arguments, but got 1.
src/events-try.ts:5:32 - error TS2345: Argument of type '[string]' is not assignable to parameter of type 'never'.
src/events-try.ts:6:30 - error TS2345: Argument of type '(id: number) => void' is not assignable to parameter of type '(id: string) => void'.
```

The third message is the hard one to read: `"product:removed"` is not in the map, so no arguments fit, and TypeScript says `never` (Session 04). Read "parameter of type 'never'" here as "this event does not exist". Delete the scratch file.

Use it. In `app.ts`, import `events` and announce each new product

```typescript
import { events } from "./events.ts";
```

```typescript
  } else if (method === "POST" && url.pathname === "/products") {
    const data = validate(newProductSchema, await readBody(req), "product");
    const product = addProduct(data);
    events.emit("product:added", product);
    sendJson(res, 201, product);
```

and in `server.ts`, after `createApp()`, listen

```typescript
events.on("product:added", (product) => {
  console.log(`new product: ${product.name} (${product.pricePaise} paise)`);
});
```

`product` has the type `Product` without an annotation: the event map gave it. With the server running, `node try-api.ts` adds the webcam, and the server's terminal shows

```text
development server on http://localhost:3000
new product: Webcam (249900 paise)
```

`app.ts` does not know who listens, and `server.ts` does not know where products are added. `npm run check`, `npm run lint` and the six tests still pass.

![The event map ShopEvents types the emitter: app.ts emits product:added with a Product and server.ts receives it typed; a product without pricePaise, a missing argument, an unknown event and a listener with the wrong parameter type each give an error](images/30-nodejs-with-typescript/typed-events.gif)

---

## Mini Example 2: A Production Folder Without TypeScript

What does the server really need? Try it: make a folder `30-deploy` next to `30-node-api` (in `ts-practice`), and copy only three things into it: `dist`, `package.json` and `package-lock.json`. Add a `.env` with the settings a real server would get from its environment

30-deploy/.env

```text
NODE_ENV=production
PORT=8080
```

Inside `30-deploy`

```bash
npm ci --omit=dev
```

```text
added 8 packages, and audited 9 packages in 724ms

found 0 vulnerabilities
```

`npm ci` installs exactly the versions in `package-lock.json`, the usual way on servers, and `--omit=dev` skips `devDependencies`. 8 packages: `joi` and the 7 small packages it uses. Is TypeScript there?

```bash
npm ls typescript
```

```text
30-node-api@1.0.0 D:\ts-practice\30-deploy
`-- (empty)
```

No. And the server does not need it

```bash
npm start
```

```text
production server on http://localhost:8080
```

`/products/2` answers as before. The server runs plain JavaScript with one package, `joi`; no TypeScript, no Vitest, no ESLint.

What happens when a run-time package lands in `devDependencies` by mistake? In a copy of the project, `npm install --save-dev joi` moves it there; the build still works on your computer, and the production folder fails at once

```text
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'joi' imported from D:\ts-practice\30-joidev\dist\products.js
```

![On your computer the project has src, dist, the package files and every tool in node_modules; the production folder gets only dist and the package files, npm ci --omit=dev adds 8 packages, joi and its helpers, npm ls typescript is empty, and npm start runs the production server on port 8080](images/30-nodejs-with-typescript/deploy.gif)

---

## Beginner Mistakes

### Mistake 1: Using req.url as if it were always there

`req.url` and `req.method` are `string | undefined` in the types of `node:http`

```text
src/app.ts:22:23 - error TS2345: Argument of type 'string | undefined' is not assignable to parameter of type 'string | { toString: () => string; }'.
  Type 'undefined' is not assignable to type 'string | { toString: () => string; }'.

22   const url = new URL(req.url, "http://localhost");
                         ~~~~~~~
```

Give a fallback: `req.url ?? "/"`.

---

### Mistake 2: Passing an async handler to createServer without a catch

`createServer(handle)` with an `async` `handle` passes `tsc`, and the first rejected promise stops the whole server. Catch it where the request starts: `handle(req, res).catch(...)`. ESLint's `no-misused-promises` reports the mistake.

---

### Mistake 3: Answering errors with their real message for every error

`sendError` sends the message of `InvalidDataError` and `AppError`, which were written for clients. Any other error is a bug, and its message may show file paths or database details. Log it, and answer `500 "something went wrong"`.

---

### Mistake 4: --env-file on a server without a .env

`node --env-file=.env` stops when the file is missing

```text
node: .env: not found
```

(The first word may be the full path of `node.exe`.) Use `--env-file-if-exists`, which prints `.env not found. Continuing without it.` and goes on with the environment.

---

### Mistake 5: Starting a second server on the same port

`npm run dev` in one terminal and `npm start` in another: the second stops at once

```text
Error: listen EADDRINUSE: address already in use :::3000
```

Stop the first one, or give the second another `PORT`. Tests avoid it with port 0.

---

### Mistake 6: A misspelled signal name

`process.on("SIGTREM", shutdown)` passes `tsc`: `process.on` also accepts any string, for events you emit yourself. The handler simply never runs. Copy signal names exactly: `SIGINT`, `SIGTERM`.

---

### Mistake 7: A run-time package in devDependencies

Everything works on your computer, where all packages are installed, and the server fails with `Cannot find package 'joi'` (Mini Example 2). A package that `src` imports belongs in `dependencies`; tools belong in `devDependencies`.

---

### Mistake 8: Reading .port without narrowing server.address()

`server.address().port` gives TS2531 (`null`) and TS2339 (`string`). Check `address === null || typeof address === "string"` first.

---

### Mistake 9: Committing .env

`.env` holds the settings of one computer, and later its secrets. Put it in `.gitignore`, and commit a `.env.example` without real values.

---

### Mistake 10: Expecting npm run dev to report type errors

`node --watch` runs the code with the types removed, errors or not. Run `npm run check` (or watch the editor's errors), and let the tests and the build catch the rest.

---

## Practice Exercises

### Exercise 1

Add a query to `GET /products`: `/products?maxPricePaise=50000` returns only products up to that price. `url.searchParams.get("maxPricePaise")` returns `string | null`: what do you answer for `?maxPricePaise=abc`? Add two tests.

### Exercise 2

Add `DELETE /products/:id`. Answer 204 without a body when the product existed (`res.writeHead(204); res.end();`), and 404 when it did not. Write the tests first, and watch them fail.

### Exercise 3

Add a setting `HOST` to `src/config/index.ts` with the default `"localhost"`, and use it in `server.listen(config.port, config.host, ...)`. What does Joi answer for an empty `HOST=` line in `.env`?

### Exercise 4

Add a route that throws a plain `Error("boom")` and check with a test that the answer is 500 with `{ "error": "something went wrong" }`. Where does the real message appear?

### Exercise 5

Add an event `"product:removed": [id: number]` to `ShopEvents`, emit it from your DELETE route (Exercise 2), and listen for it in `server.ts`. Then try to emit it with a string id: what does `tsc` say?

### Exercise 6

Make a slow route: `GET /slow` waits 5 seconds (`setTimeout` from `node:timers/promises`) and then answers. Open it in the browser, press Ctrl+C in the server terminal right away, and watch: does the browser still get its answer? In which order do the log lines appear?

### Exercise 7

Run `npm run build`, then change a price in `src/products.ts` without building again, and run `npm start`. Which price does `/products/2` show, and why?

### Exercise 8

In your `30-deploy` folder, delete `node_modules` and run `npm install --omit=dev` instead of `npm ci --omit=dev`. Then delete `package-lock.json` and try `npm ci` again. What does `npm ci` need that `npm install` does not?

---

## Interview Questions

### How do you run a TypeScript Node.js app in development and in production?

In development, run the `.ts` files directly, for example with `node --watch --env-file-if-exists=.env src/server.ts`. For production, build with `tsc` into `dist`, install only `dependencies` on the server (`npm ci --omit=dev`), and start `node --enable-source-maps dist/server.js`.

### Why build for production if Node.js can run .ts files?

A build checks every file before anything ships (with `noEmitOnError`, errors mean no output), the server needs no TypeScript tools, and the JavaScript runs on any Node.js version. Source maps keep the stack traces readable.

### Where do the types of Node.js come from?

From the `@types/node` package, turned on with `"types": ["node"]` in `tsconfig.json`. Its major version should match the Node.js version on the server, so the types do not promise APIs that are missing.

### Why is req.url typed as string | undefined?

The `IncomingMessage` type is shared by requests a server receives and answers a client receives; only requests have a URL. For a server request it is always set in practice, so a fallback like `req.url ?? "/"` is enough.

### What happens when an async request handler rejects in node:http?

`node:http` does not handle the promise, so the rejection is unhandled and Node.js stops the process: every user loses the server. Catch the promise where the request starts and turn the error into an answer.

### How do you read and check a JSON request body?

Read it with `json(req)` from `node:stream/consumers`, which returns `Promise<unknown>`, catch the parsing error for bodies that are not JSON (400), and validate the value with a schema before using it as a typed value.

### Do you need nodemon, dotenv or ts-node with Node.js 24?

No. `node --watch` restarts on changes, `--env-file` and `--env-file-if-exists` load a `.env` file, and Node.js runs `.ts` files by removing the types. Type checking stays with `tsc`.

### What is a graceful shutdown?

Reacting to `SIGINT` or `SIGTERM` by closing the server: no new connections, open requests finish, then the process ends by itself. Platforms send `SIGTERM` before they stop or replace an app.

### How do you test an HTTP server without a fixed port?

Start it in the tests with `server.listen(0)`, so the system picks a free port, read the port from `server.address()` after narrowing its type, send requests with `fetch`, and close the server in `afterAll`.

### What is the difference between dependencies and devDependencies?

`dependencies` are needed when the app runs, like `joi`; `devDependencies` are needed only to develop it, like TypeScript, `@types/node`, Vitest and ESLint. `npm ci --omit=dev` installs only the first group, which is why a run-time package in the wrong group breaks production.

### How do you type the events of an EventEmitter?

Pass an event map as its type argument: `new EventEmitter<ShopEvents>()`, where each event name maps to a tuple of the values it carries. `emit` and `on` are then checked, and listeners get their parameter types for free.

### Why keep app.ts and server.ts separate?

`app.ts` builds the server without starting it, so tests can start it on their own port, and nothing in it depends on the settings or the signals of the real process. `server.ts` is the one file with side effects: it listens, logs and handles shutdown.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [29. Linting, Testing and Debugging](29-linting-testing-debugging.md) | [Course outline](README.md#course-outline) | [31. Express with TypeScript](31-express-with-typescript.md) |
