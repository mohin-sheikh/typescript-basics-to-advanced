# MongoDB and Mongoose with TypeScript

> **Session 32 of 33** | Part 7: TypeScript with Node.js | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [Mongoose and Its Types](#mongoose-and-its-types)
- [Settings for the Database](#settings-for-the-database)
- [A Schema from an Interface](#a-schema-from-an-interface)
- [The Model and Its Documents](#the-model-and-its-documents)
- [The Products Become Async](#the-products-become-async)
- [Connecting Before Listening](#connecting-before-listening)
- [Errors from Mongoose and MongoDB](#errors-from-mongoose-and-mongodb)
- [Queries Are Typed, Mostly](#queries-are-typed-mostly)
- [Tests with a Database in Memory](#tests-with-a-database-in-memory)
- [Mini Example 1: Orders That Point to Products](#mini-example-1-orders-that-point-to-products)
- [Mini Example 2: Letting Mongoose Write the Type](#mini-example-2-letting-mongoose-write-the-type)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

Session 31's API keeps its products in an array, so every restart forgets the products an admin added. This session stores them in MongoDB with **Mongoose**, the library the Node.js course uses (its [Sessions 17 to 19](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/19-mongoose.md) teach MongoDB and Mongoose in JavaScript). Copy the folder `31-express-api` to `32-mongoose-api`, without `node_modules` and `dist`. Inside the new folder

```bash
npm pkg set name=32-mongoose-api
npm install
npm install mongoose
```

```text
added 16 packages, and audited 229 packages in 6s
```

Mongoose brings the official MongoDB driver, the package `mongodb`, with it. The versions we tested: mongoose 9.11.4, with the driver 7.7.0.

You also need a MongoDB to talk to. The Node.js course's [Session 17](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/17-mongodb-introduction.md) sets up either a free MongoDB Atlas cluster or MongoDB Community Server on your computer. This lesson uses a local one at `mongodb://127.0.0.1:27017` (we tested with MongoDB 8.2.6). With Atlas, put your own connection string in `.env` (next sections).

---

## Mongoose and Its Types

Mongoose is written in JavaScript, but unlike Express it ships its own `.d.ts` files (Session 25 listed it under "its own"). The package says where they are

```bash
npm view mongoose types
```

```text
./types/index.d.ts
```

So there is nothing else to install. A package `@types/mongoose` still exists from the time before Mongoose had its own types. Its last version is an empty stub, and npm tells you to leave it alone

```bash
npm view @types/mongoose deprecated
```

```text
Mongoose publishes its own types, so you do not need to install this package.
```

`mongoose` goes into `dependencies`, because the server runs it. Its import has both kinds of names

```typescript
import { Schema, model, type HydratedDocument } from "mongoose";
```

and, in the files that connect, the default import `import mongoose from "mongoose"`: the object with `connect()`, `disconnect()`, Mongoose's error classes and the driver.

The types you meet in this session

| Name                    | What it is                                                                | Seen in      |
| ----------------------- | ------------------------------------------------------------------------- | ------------ |
| `Schema<T>`             | The rules for one kind of document, checked against the interface `T`    | Session 20 (generic classes) |
| `Model<T, ...>`         | What `model()` returns: `create`, `find`, `findById`, `findByIdAndUpdate` | Session 14 (generic `Model<T>`) |
| `HydratedDocument<T>`   | One document loaded from MongoDB: your fields, `_id`, `__v` and methods   | This session |
| `Types.ObjectId`        | The type of `_id`                                                          | This session |
| `InferSchemaType<S>`    | A type computed from a schema (Mini Example 2)                            | Session 18 (conditional types) |

![The interface NewProduct goes into new Schema of NewProduct, model() turns the schema into a Model, create and findById give a HydratedDocument of NewProduct (or null), and toProduct turns it into the Product the API sends](images/32-mongoose-with-typescript/mongoose-types.gif)

---

## Settings for the Database

Session 28 left the database settings out of `config` until this session. Two more lines in `.env`

.env

```text
PORT=3000
MONGODB_URI=mongodb://127.0.0.1:27017
DB_NAME=shop
```

and in `.env.example`, which is committed (Session 30)

.env.example

```text
# Copy this file to .env and change the values
NODE_ENV=development
PORT=3000
MONGODB_URI=mongodb://127.0.0.1:27017
DB_NAME=shop
```

Then the settings, with the same pattern as before: an interface, a strict Joi schema, one `validate` call (Session 26)

src/config/index.ts

```typescript
import Joi from "joi";
import { validate } from "../utils/validation.ts";

const NODE_ENVS = ["development", "production", "test"] as const;

interface Env {
  NODE_ENV: (typeof NODE_ENVS)[number];
  PORT: number;
  MONGODB_URI: string;
  DB_NAME: string;
}

const envSchema = Joi.object<Env, true>({
  NODE_ENV: Joi.string().valid(...NODE_ENVS).default("development"),
  PORT: Joi.number().port().default(3000),
  MONGODB_URI: Joi.string().uri({ scheme: ["mongodb", "mongodb+srv"] }).required(),
  DB_NAME: Joi.string().default("shop")
});

const env = validate(envSchema, process.env, "environment");

export const config = {
  env: env.NODE_ENV,
  port: env.PORT,
  mongodbUri: env.MONGODB_URI,
  dbName: env.DB_NAME
};
```

| Line                                            | Why                                                                  |
| ----------------------------------------------- | -------------------------------------------------------------------- |
| `.uri({ scheme: ["mongodb", "mongodb+srv"] })`  | A local address starts with `mongodb://`, an Atlas one with `mongodb+srv://` |
| `.required()` on `MONGODB_URI`                  | There is no good default for the database address                     |
| `DB_NAME` with `.default("shop")`               | The database name stays separate from the address (it goes into the `dbName` option, as in the Node.js course) |

A `.env` from Session 31, with only `PORT=3000`, now stops the server before anything else happens

```text
InvalidDataError: environment is invalid: MONGODB_URI is required
```

and an address without its scheme, `MONGODB_URI=127.0.0.1:27017`, gives

```text
InvalidDataError: environment is invalid: MONGODB_URI must be a valid uri with a scheme matching the mongodb|mongodb\+srv pattern
```

Joi writes the allowed schemes as a regular expression: `\+` is a plain plus sign. Both messages come with the stack trace and the `problems` list of Session 26. `config.mongodbUri` and `config.dbName` are `string`s, so the rest of the code never sees `undefined`.

---

## A Schema from an Interface

The products file keeps Session 31's types, with two changes: MongoDB ids are text, and a type for changes (used by a new `PATCH` route later)

src/products.ts (the start)

```typescript
import Joi from "joi";
import { Schema, model, type HydratedDocument } from "mongoose";

export interface Product {
  id: string;
  name: string;
  pricePaise: number;
}

export type NewProduct = Omit<Product, "id">;
export type ProductChanges = Partial<NewProduct>;

export const newProductSchema = Joi.object<NewProduct, true>({
  name: Joi.string().trim().min(2).required(),
  pricePaise: Joi.number().integer().min(0).required()
});

export const productChangesSchema = Joi.object<ProductChanges, true>({
  name: Joi.string().trim().min(2),
  pricePaise: Joi.number().integer().min(0)
}).min(1);
```

`ProductChanges` is `Partial<NewProduct>` (Session 17): both fields optional. `.min(1)` on the Joi object asks for at least one of them.

Then the part Mongoose reads. A **schema** describes the documents of one collection

src/products.ts (continued)

```typescript
// MongoDB stores the fields of a NewProduct and adds an _id
const productSchema = new Schema<NewProduct>({
  name: { type: String, required: true, unique: true },
  pricePaise: { type: Number, required: true, min: 0 }
});

export const ProductModel = model("Product", productSchema);
```

| `NewProduct` says (for `tsc`) | The schema says (for Mongoose, while the code runs)              |
| ----------------------------- | ---------------------------------------------------------------- |
| `name: string`                | `type: String`                                                   |
| `pricePaise: number`          | `type: Number`                                                   |
| no `?`                        | `required: true`                                                 |
| nothing                       | `unique: true`, `min: 0`: rules a type has no words for         |

Mind the capital letters. `String` and `Number` are JavaScript functions, values that Mongoose reads while the program runs; `string` and `number` are TypeScript types, which disappear (Session 08). The schema has to be a value, because types are gone by the time Mongoose saves anything.

The `<NewProduct>` in `new Schema<NewProduct>` connects the two: `tsc` checks the schema against the interface. A typo in a field name

```typescript
  pricePiase: { type: Number, required: true, min: 0 }
```

```text
src/products.ts:26:3 - error TS2561: Object literal may only specify known properties, but 'pricePiase' does not exist in type 'NewProduct | { name?: SchemaDefinitionProperty<string, NewProduct, Document<unknown, {}, NewProduct, { id: string; }, DefaultSchemaOptions> & Omit<...> & HydratedDocumentOverrides<...>>; pricePaise?: SchemaDefinitionProperty<...>; }'. Did you mean to write 'pricePaise'?

26   pricePiase: { type: Number, required: true, min: 0 }
     ~~~~~~~~~~
```

Read the start and the end (Session 05): `'pricePiase' does not exist`, `Did you mean to write 'pricePaise'?`. The middle is Mongoose's own type for a schema field. Four more errors follow, about 60 lines each, at every place that uses the model, because a broken schema gives the documents a broken type. The summary says so

```text
Found 5 errors in the same file, starting at: src/products.ts:26
```

Fix the first one and all five are gone. A wrong kind of value is caught too

```typescript
  pricePaise: { type: String, required: true, min: 0 }
```

```text
src/products.ts:26:3 - error TS2322: Type '{ type: StringConstructor; required: true; min: number; }' is not assignable to type 'SchemaDefinitionProperty<number, NewProduct, Document<unknown, {}, NewProduct, { id: string; }, DefaultSchemaOptions> & Omit<NewProduct & { ...; } & { ...; }, "id"> & HydratedDocumentOverrides<...>> | undefined'.
  Types of property 'type' are incompatible.
    Type 'StringConstructor' is not assignable to type 'typeof Mixed | NumberSchemaDefinition | undefined'.
      Type 'StringConstructor' is missing the following properties from type 'NumberConstructor': MAX_VALUE, MIN_VALUE, NaN, NEGATIVE_INFINITY, and 10 more.
```

`StringConstructor` is the type of the `String` function; a `number` field needs `Number`.

Now the part `tsc` does **not** check. Delete the `pricePaise` line from the schema: `npm run check` passes. Start the server and add a product as an admin

```text
>>> POST /products {"name":"Webcam","pricePaise":249900}
201 application/json; charset=utf-8
{"id":"6ae5ed0908257c4674777c89","name":"Webcam"}
```

The price is gone. Mongoose saves only the fields its schema knows and drops the rest without a word, while the interface still promises `pricePaise: number`. The same goes for `required: true`: without it, `tsc` sees no difference, and MongoDB accepts products without a name.

| `new Schema<NewProduct>` checks              | It does not check                         |
| -------------------------------------------- | ----------------------------------------- |
| Every field name in the schema (TS2561)      | That every field of the interface is in the schema |
| The kind of value: `String` for a `string`   | `required: true` for fields without `?`   |
| The rule values: `min: "zero"` is an error   | `unique`, and whether `min: 0` makes sense |

So the interface and the schema are two descriptions that must agree, and `tsc` checks only part of the agreement. The tests at the end of this session check the rest.

![tsc checks a schema against its interface: a misspelled field and a String for a number are errors; a missing field and a missing required pass, and the Webcam is saved without its price](images/32-mongoose-with-typescript/schema-checks.gif)

---

## The Model and Its Documents

`model("Product", productSchema)` makes the **model**: the object with `create`, `find`, `findById` and the rest. Mongoose names the collection after it: `"Product"` becomes `products` (Node.js course, Session 19).

Its type is the generic `Model` that Session 14 promised, with seven type arguments that Mongoose fills in. Let `tsc` infer it. Even the short `Model<NewProduct>`, written by hand, is refused

```typescript
export const ProductModel: Model<NewProduct> = model("Product", productSchema);
```

```text
src/products.ts:29:14 - error TS2375: Type 'Model<NewProduct, {}, {}, { id: string; }, Document<unknown, {}, NewProduct, { id: string; }, DefaultSchemaOptions> & Omit<NewProduct & { ...; } & { ...; }, "id"> & HydratedDocumentOverrides<...>, Schema<...>, NewProduct & ... 1 more ... & { ...; }>' is not assignable to type 'Model<NewProduct, {}, {}, {}, Document<unknown, {}, NewProduct, {}, DefaultSchemaOptions> & NewProduct & { ...; } & { ...; } & { ...; }, any, NewProduct & ... 1 more ... & { ...; }>' with 'exactOptionalPropertyTypes: true'. Consider adding 'undefined' to the types of the target's properties.
```

and a dozen lines later, the reason

```text
                    Property 'id' is missing in type '{}' but required in type '{ id: string; }'.
```

The real model has a fourth type argument, `{ id: string; }`: the `id` that Mongoose adds to every document. The hand-written one does not.

What does a document look like to `tsc`? Use the "surely wrong value" trick of Session 18 in a small script

src/peek.ts

```typescript
import mongoose from "mongoose";
import { config } from "./config/index.ts";
import { ProductModel } from "./products.ts";

await mongoose.connect(config.mongodbUri, { dbName: config.dbName });

const doc = await ProductModel.create({ name: "Lamp", pricePaise: 89900 });
const check: number = doc;
console.log(JSON.stringify(doc));
console.log(doc.id, doc._id);

await mongoose.disconnect();
```

```text
src/peek.ts:8:7 - error TS2322: Type 'Document<unknown, {}, NewProduct, { id: string; }, DefaultSchemaOptions> & Omit<NewProduct & { _id: ObjectId; } & { ...; }, "id"> & HydratedDocumentOverrides<...>' is not assignable to type 'number'.
```

| Part of the type                              | What it gives you                                        |
| --------------------------------------------- | -------------------------------------------------------- |
| `Document<unknown, {}, NewProduct, { id: string; }, ...>` | Mongoose's methods (`save()`, `toJSON()`, ...) and `id: string` |
| `NewProduct`                                  | Your fields: `name`, `pricePaise`                        |
| `{ _id: ObjectId; }`                          | The id MongoDB adds                                      |
| the `{ ...; }` that `tsc` shortened           | `{ __v: number; }`, the version number Mongoose adds     |

Mongoose has a short name for the whole thing: `HydratedDocument<NewProduct>`. "Hydrated" means a plain record from the database turned into a full document object, with methods.

Remove the `check` line and run the script (the connection is explained in [Connecting Before Listening](#connecting-before-listening))

```bash
node --env-file=.env src/peek.ts
```

```text
{"name":"Lamp","pricePaise":89900,"_id":"6ae5ed0fe161a50f47dcf802","__v":0}
6ae5ed0fe161a50f47dcf802 new ObjectId('6ae5ed0fe161a50f47dcf802')
```

`_id` is an `ObjectId` object; `id` is the same id as a `string`, just as the type said. The JSON has `_id` and `__v`, not the `Product` shape the API promises. So one small function decides what leaves the server, like `PublicUser` in Session 17

src/products.ts (continued)

```typescript
type ProductDocument = HydratedDocument<NewProduct>;

function toProduct(doc: ProductDocument): Product {
  return { id: doc.id, name: doc.name, pricePaise: doc.pricePaise };
}
```

The return type `Product` checks the answer: a forgotten field is an error (Session 07), and `__v` cannot slip out.

`findById` is the `null` that Session 11 promised: no document has that id. Use the result without checking

```typescript
export async function findProduct(id: string): Promise<Product> {
  const doc = await ProductModel.findById(id);
  return { id: doc.id, name: doc.name, pricePaise: doc.pricePaise };
}
```

```text
src/products.ts:45:16 - error TS18047: 'doc' is possibly 'null'.

45   return { id: doc.id, name: doc.name, pricePaise: doc.pricePaise };
                  ~~~
```

once for each `doc.`, three errors in all. The type of `doc` is `HydratedDocument<NewProduct> | null`, and a `null` check narrows it (Session 10).

![A Mongoose document has your fields, an ObjectId _id, a __v and methods; JSON.stringify sends _id and __v, while toProduct sends id, name and pricePaise; findById can give null](images/32-mongoose-with-typescript/document-json.gif)

---

## The Products Become Async

Here is the rest of the products file. Every exported function now talks to MongoDB, so each one is `async` (Session 23)

src/products.ts (continued)

```typescript
type ProductDocument = HydratedDocument<NewProduct>;

function toProduct(doc: ProductDocument): Product {
  return { id: doc.id, name: doc.name, pricePaise: doc.pricePaise };
}

export async function listProducts(maxPricePaise?: number): Promise<Product[]> {
  const filter = maxPricePaise === undefined ? {} : { pricePaise: { $lte: maxPricePaise } };
  const docs = await ProductModel.find(filter).sort({ pricePaise: 1 });
  return docs.map(toProduct);
}

export async function findProduct(id: string): Promise<Product | undefined> {
  const doc = await ProductModel.findById(id);
  return doc === null ? undefined : toProduct(doc);
}

export async function addProduct(data: NewProduct): Promise<Product> {
  const doc = await ProductModel.create(data);
  return toProduct(doc);
}

export async function updateProduct(id: string, changes: ProductChanges): Promise<Product | undefined> {
  const doc = await ProductModel.findByIdAndUpdate(id, changes, { returnDocument: "after", runValidators: true });
  return doc === null ? undefined : toProduct(doc);
}
```

| Function         | MongoDB part                                                                  |
| ---------------- | ----------------------------------------------------------------------------- |
| `listProducts`   | `{}` finds every product; `{ pricePaise: { $lte: ... } }` the ones at most that price; `sort` puts the cheapest first |
| `findProduct`    | `null` from MongoDB becomes `undefined`, the answer Session 31's routes already check |
| `addProduct`     | `create` saves and returns the new document                                  |
| `updateProduct`  | `returnDocument: "after"` returns the changed product; `runValidators: true` applies `min: 0` to updates too (Node.js course, Session 19) |

The names are the same as in Session 31, with three changes: `listProducts` filters in the database, `id` is a `string`, and every return type is a `Promise`. Keep Session 31's routes as they are and ask `tsc`

```text
src/routes/products.ts:21:62 - error TS2339: Property 'filter' does not exist on type 'Promise<Product[]>'.

21   res.json(maxPricePaise === undefined ? products : products.filter((product) => product.pricePaise <= maxPricePaise));
                                                                ~~~~~~

  src/routes/products.ts:21:62 - Did you forget to use 'await'?
    21   res.json(maxPricePaise === undefined ? products : products.filter((product) => product.pricePaise <= maxPricePaise));
                                                                    ~~~~~~

src/routes/products.ts:21:70 - error TS7006: Parameter 'product' implicitly has an 'any' type.

21   res.json(maxPricePaise === undefined ? products : products.filter((product) => product.pricePaise <= maxPricePaise));
                                                                        ~~~~~~~

src/routes/products.ts:25:31 - error TS2345: Argument of type 'number' is not assignable to parameter of type 'string'.

25   const product = findProduct(Number(req.params.id));
                                 ~~~~~~~~~~~~~~~~~~~~~


Found 3 errors in the same file, starting at: src/routes/products.ts:21
```

The first is Session 23's hint: a `Promise` has no `filter` (the second error follows from it). The third is the id type. But two lines changed their meaning **without** an error

```typescript
  const product = findProduct(Number(req.params.id));
  if (product === undefined) {
```

`product` is now a `Promise`, which is never `undefined`, so the 404 can never happen. `tsc` allows any comparison with `undefined`. And

```typescript
  res.status(201).json(addProduct(data));
```

`res.json()` takes `any` (Session 31), so a `Promise` goes in. Start the server and add a product

```text
>>> POST /products {"name":"Webcam","pricePaise":249900}
201 application/json; charset=utf-8
{}
```

The Webcam was saved, and the client got `{}`, which is what `JSON.stringify` makes of a `Promise`. `npm run lint` only repeats line 21 (`no-unsafe-call` and `no-unsafe-member-access`, because `product` is `any` there). This is the lesson of the move: `tsc` points you to most places where a value became a `Promise`, but read every caller yourself. Here are the new routes, with `await` in every handler and a new `PATCH` for admins

src/routes/products.ts

```typescript
import express, { type Request } from "express";
import Joi from "joi";
import { requireAdmin, requireUser } from "../middleware/auth.ts";
import { addProduct, findProduct, listProducts, newProductSchema, productChangesSchema, updateProduct } from "../products.ts";
import { AppError } from "../utils/errors.ts";
import { validate } from "../utils/validation.ts";

interface ProductQuery {
  maxPricePaise?: number;
}

const querySchema = Joi.object<ProductQuery, true>({
  maxPricePaise: Joi.number().integer().min(0)
});

export const productsRouter = express.Router();

productsRouter.get("/", async (req, res) => {
  const { maxPricePaise } = validate(querySchema, req.query, "query");
  res.json(await listProducts(maxPricePaise));
});

productsRouter.get("/:id", async (req, res) => {
  const product = await findProduct(req.params.id);
  if (product === undefined) {
    throw new AppError(`product ${req.params.id} not found`, 404);
  }
  res.json(product);
});

productsRouter.post("/", requireUser, requireAdmin, async (req, res) => {
  const data = validate(newProductSchema, req.body, "product");
  res.status(201).json(await addProduct(data));
});

productsRouter.patch("/:id", requireUser, requireAdmin, async (req: Request<{ id: string }>, res) => {
  const changes = validate(productChangesSchema, req.body, "changes");
  const product = await updateProduct(req.params.id, changes);
  if (product === undefined) {
    throw new AppError(`product ${req.params.id} not found`, 404);
  }
  res.json(product);
});
```

The handlers are `async`, and Express 5 sends their rejected promises to the error handler (Session 31). Why does the `PATCH` handler write `req: Request<{ id: string }>`, when the `GET` handler gets `req.params.id` from the route text? Write it like the others

```typescript
productsRouter.patch("/:id", requireUser, requireAdmin, async (req, res) => {
```

```text
src/routes/products.ts:38:39 - error TS2345: Argument of type 'string | string[] | undefined' is not assignable to parameter of type 'string'.
  Type 'undefined' is not assignable to type 'string'.

38   const product = await updateProduct(req.params.id, changes);
                                         ~~~~~~~~~~~~~
```

The middlewares in front decide. `requireUser` and `requireAdmin` have the type `RequestHandler`, whose `req.params` is the plain dictionary of Session 31, and Express's types take the parameters' type from the handlers when they have one. So the route text loses, and `req.params.id` is `string | string[] | undefined` (the `undefined` comes from `noUncheckedIndexedAccess`, Session 06). Session 31's fix for named handlers brings the check back: `Request<{ id: string }>`.

One file did not change at all: `app.ts`. As Session 24 promised, the data moved from an array to MongoDB without touching it.

![products.ts becomes async: tsc finds filter on a Promise and the number id, but the comparison with undefined and res.json of a Promise pass, and the POST answers 201 with an empty object](images/32-mongoose-with-typescript/async-ripple.gif)

![Middleware typed RequestHandler in front of a /:id handler brings the plain params dictionary, so req.params.id becomes string, string array or undefined; Request of id string on the handler restores it](images/32-mongoose-with-typescript/params-middleware.gif)

---

## Connecting Before Listening

The server connects first and listens only when the database is there

src/server.ts

```typescript
import mongoose from "mongoose";
import { createApp } from "./app.ts";
import { config } from "./config/index.ts";
import { errorMessage } from "./utils/errors.ts";

try {
  await mongoose.connect(config.mongodbUri, { dbName: config.dbName, serverSelectionTimeoutMS: 5000 });
} catch (err) {
  console.error(`could not connect to MongoDB: ${errorMessage(err)}`);
  process.exit(1);
}
console.log(`connected to MongoDB, database ${config.dbName}`);

const app = createApp();

const server = app.listen(config.port, () => {
  console.log(`${config.env} server on http://localhost:${config.port}`);
});

async function shutdown(signal: NodeJS.Signals): Promise<void> {
  console.log(`${signal} received, closing the server`);
  server.close();
  await mongoose.disconnect();
  console.log("server closed, MongoDB disconnected");
}

function onSignal(signal: NodeJS.Signals): void {
  shutdown(signal).catch((err: unknown) => {
    console.error(err);
  });
}

process.on("SIGINT", onSignal);
process.on("SIGTERM", onSignal);
```

| Part                                      | Why                                                                          |
| ----------------------------------------- | ---------------------------------------------------------------------------- |
| `await` at the top level                  | The module waits for the connection (Session 24); `listen` runs after it     |
| `catch (err)` with `errorMessage(err)`    | `err` is `unknown` (Session 04); the helper is from Session 23              |
| `process.exit(1)`                         | Its return type is `never`, so `tsc` knows the code below runs only when connected |
| `dbName: config.dbName`                   | Chooses the database; never glue the name onto the address (Node.js course, Session 19) |
| `serverSelectionTimeoutMS: 5000`          | Give up after 5 seconds; without it, the driver waits 30                     |
| `shutdown` is `async`, `onSignal` is not  | Every call to an `async` function gets an `await`, a `return` or a `.catch()` (Session 23); `process.on` wants a function that returns nothing (`no-misused-promises`, Session 29) |

The options object has a type, `ConnectOptions`, so the driver's option names are checked. The `MS` at the end is easy to get wrong

```text
src/server.ts:7:70 - error TS2561: Object literal may only specify known properties, but 'serverSelectionTimeoutMs' does not exist in type 'ConnectOptions'. Did you mean to write 'serverSelectionTimeoutMS'?
```

`npm run dev`, then add two products as Riya (the admin token `Authorization: Bearer token-riya` of Session 31) and list them

```text
connected to MongoDB, database shop
development server on http://localhost:3000
>>> POST /products {"name":"Keyboard","pricePaise":149900}
201 application/json; charset=utf-8
{"id":"6ae5ede98414b8564634c4f0","name":"Keyboard","pricePaise":149900}
>>> POST /products {"name":"Mouse","pricePaise":49900}
201 application/json; charset=utf-8
{"id":"6ae5ede98414b8564634c4f1","name":"Mouse","pricePaise":49900}
>>> GET /products
200 application/json; charset=utf-8
[{"id":"6ae5ede98414b8564634c4f1","name":"Mouse","pricePaise":49900},{"id":"6ae5ede98414b8564634c4f0","name":"Keyboard","pricePaise":149900}]
```

Stop the server and start it again

```text
connected to MongoDB, database shop
development server on http://localhost:3000
>>> GET /products
200 application/json; charset=utf-8
[{"id":"6ae5ede98414b8564634c4f1","name":"Mouse","pricePaise":49900},{"id":"6ae5ede98414b8564634c4f0","name":"Keyboard","pricePaise":149900}]
```

Same products, same ids: the restart forgot nothing. Your ids will be different; MongoDB makes a new one for every document.

With a wrong address in `.env`, `MONGODB_URI=mongodb://127.0.0.1:27018`, the server waits five seconds, then

```text
could not connect to MongoDB: connect ECONNREFUSED 127.0.0.1:27018
```

and stops with exit code 1, without ever listening. And `Ctrl+C` closes both

```text
SIGINT received, closing the server
server closed, MongoDB disconnected
```

![server.ts reads the settings, connects to MongoDB within 5 seconds and only then listens; a wrong address prints could not connect and exits with code 1; Ctrl+C closes the server and disconnects](images/32-mongoose-with-typescript/connect-first.gif)

---

## Errors from Mongoose and MongoDB

Two new kinds of errors reach the error handler, and Session 31's handler turns both into a 500. An id that is not an ObjectId

```text
>>> GET /products/abc
CastError: Cast to ObjectId failed for value "abc" (type string) at path "_id" for model "Product"
    at SchemaObjectId.cast (D:\ts-practice\32-mongoose-api\node_modules\mongoose\lib\schema\objectId.js:252:11)
```

and a second Keyboard, which the unique index refuses

```text
>>> POST /products {"name":"Keyboard","pricePaise":149900}
MongoServerError: E11000 duplicate key error collection: shop.products index: name_1 dup key: { name: "Keyboard" }
    at InsertOneOperation.handleOk (D:\ts-practice\32-mongoose-api\node_modules\mongodb\lib\operations\insert.js:55:19)
```

each followed by its stack trace in the server's terminal, and the client gets

```text
500 application/json; charset=utf-8
{"error":"something went wrong"}
```

An ObjectId is 24 hexadecimal characters, so `"abc"` cannot even be looked up: Mongoose throws a **CastError**. The second error comes from MongoDB itself: `unique: true` made an index, and the index refuses a second name `Keyboard`. Session 26 said that rules like "is this name already taken" belong to the database; this is that rule.

The Node.js course's handler checks `err.name === "CastError"` and `err.code === 11000`. With `err: unknown` (Session 31), that is four errors

```typescript
  } else if (err.name === "CastError") {
    res.status(400).json({ error: `invalid ${err.path}: ${err.value}` });
  } else if (err.code === 11000) {
```

```text
src/middleware/errors.ts:18:14 - error TS18046: 'err' is of type 'unknown'.

18   } else if (err.name === "CastError") {
                ~~~
```

Narrow with `instanceof` instead (Session 21). Mongoose exports its error classes as `mongoose.Error`, and the driver's classes as `mongoose.mongo`, so `mongodb` needs no import of its own

src/middleware/errors.ts

```typescript
import type { ErrorRequestHandler, RequestHandler } from "express";
import mongoose from "mongoose";
import { AppError } from "../utils/errors.ts";
import { InvalidDataError } from "../utils/validation.ts";

export const notFound: RequestHandler = (req, res) => {
  res.status(404).json({ error: `${req.method} ${req.path} does not exist` });
};

// eslint-disable-next-line @typescript-eslint/no-unused-vars -- Express finds error handlers by their four parameters
export const errorHandler: ErrorRequestHandler = (err: unknown, req, res, next) => {
  if (err instanceof InvalidDataError) {
    res.status(400).json({ error: err.message, problems: err.problems });
  } else if (err instanceof AppError) {
    res.status(err.statusCode).json({ error: err.message });
  } else if (err instanceof SyntaxError) {
    res.status(400).json({ error: "the body is not valid JSON" });
  } else if (err instanceof mongoose.Error.CastError) {
    res.status(400).json({ error: `invalid ${err.path}: ${err.value}` });
  } else if (err instanceof mongoose.mongo.MongoServerError && err.code === 11000) {
    res.status(409).json({ error: "already exists" });
  } else {
    console.error(err);
    res.status(500).json({ error: "something went wrong" });
  }
};
```

| After the check                       | Type of the property                   |
| ------------------------------------- | -------------------------------------- |
| `err.path` on a `CastError`           | `string` (`"_id"`)                     |
| `err.value` on a `CastError`          | `any`: whatever value failed           |
| `err.code` on a `MongoServerError`    | `string \| number \| undefined`, so `=== 11000` is fine |

The error also knows which value was the duplicate, `err.keyValue`. Sending it looks helpful

```typescript
    res.status(409).json({ error: "already exists", duplicate: err.keyValue });
```

`tsc` agrees, ESLint does not

```text
  21:53  error  Unsafe assignment of an `any` value  @typescript-eslint/no-unsafe-assignment
```

The driver's `MongoServerError` declares `[key: string]: any`, so every property it does not list by name, like `keyValue`, is `any`. The short message is enough; the client knows what it sent. Now the same two requests

```text
>>> GET /products/abc
400 application/json; charset=utf-8
{"error":"invalid _id: abc"}
>>> POST /products {"name":"Keyboard","pricePaise":149900}
409 application/json; charset=utf-8
{"error":"already exists"}
```

You can also check an id before asking: `mongoose.isValidObjectId()` gives `false` for `"abc"` and `true` for a real id. That is the `validateObjectId` that Session 28 left for this session; here the error handler does the job in one place for every route.

![The error handler narrows err with instanceof: a CastError becomes 400 invalid _id, a MongoServerError with code 11000 becomes 409 already exists, anything else is logged and becomes 500](images/32-mongoose-with-typescript/mongo-errors.gif)

---

## Queries Are Typed, Mostly

Mongoose's methods have types, and many mistakes in them are caught. Try these in a scratch file

src/query-check.ts

```typescript
import { ProductModel, type ProductChanges } from "./products.ts";

export async function tryQueries(id: string): Promise<void> {
  await ProductModel.find().sort({ pricePaise: "up" });
  await ProductModel.findByIdAndUpdate(id, { pricePaise: 1 }, { returnDocument: "afterr" });

  const changes: ProductChanges = { pricePiase: 39900 };
  await ProductModel.findByIdAndUpdate(id, changes);
}
```

```text
src/query-check.ts:4:34 - error TS2345: Argument of type '{ pricePaise: "up"; }' is not assignable to parameter of type 'string | [string, SortOrder][] | Record<string, { $meta: any; } | SortOrder> | null | undefined'.
  Types of property 'pricePaise' are incompatible.
    Type '"up"' is not assignable to type '{ $meta: any; } | SortOrder | undefined'.

4   await ProductModel.find().sort({ pricePaise: "up" });
                                   ~~~~~~~~~~~~~~~~~~~~

src/query-check.ts:5:65 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Type '"afterr"' is not assignable to type '"after" | "before" | undefined'. Did you mean '"after"'?

5   await ProductModel.findByIdAndUpdate(id, { pricePaise: 1 }, { returnDocument: "afterr" });
                                                                  ~~~~~~~~~~~~~~
```

and a third error, TS2561 "Did you mean to write 'pricePaise'?", for `changes`. A filter with the wrong kind of value, like `{ pricePaise: { $lte: "cheap" } }`, is refused too, with a long "No overload matches this call".

But some typos pass. Delete `query-check.ts` and try this one; `npm run check` is happy with it

src/query-holes.ts

```typescript
import mongoose from "mongoose";
import { config } from "./config/index.ts";
import { ProductModel } from "./products.ts";

await mongoose.connect(config.mongodbUri, { dbName: config.dbName });

const cheap = await ProductModel.find({ pricePiase: { $lte: 50000 } });
console.log("cheap products:", cheap.length);

const mouse = await ProductModel.findOneAndUpdate({ name: "Mouse" }, { pricePiase: 39900 }, { returnDocument: "after" });
console.log("mouse price:", mouse?.pricePaise);

const keyboard = await ProductModel.findOneAndUpdate(
  { name: "Keyboard" },
  { pricePaise: -100 },
  { returnDocument: "after", runValidator: true }
);
console.log("keyboard price:", keyboard?.pricePaise);

await mongoose.disconnect();
```

With the Keyboard and the Mouse in the database

```text
cheap products: 0
mouse price: 49900
keyboard price: -100
```

| Typo                              | What happened                                                              |
| --------------------------------- | -------------------------------------------------------------------------- |
| `pricePiase` in a filter          | MongoDB looked for products with a field `pricePiase`. There are none      |
| `pricePiase` in an update         | Mongoose dropped the unknown field (strict schema). Nothing changed        |
| `runValidator` in the options     | An unknown option is just ignored, so no validator ran: -100 was saved     |

Mongoose's types for filters, updates and options accept names they do not know, on purpose: MongoDB queries may name nested fields like `"address.city"`, and the driver has many options. Set the Keyboard's price back with `PATCH /products/<its id>` and `{ "pricePaise": 149900 }`.

Two habits close these holes:

1. Build updates from typed values. `updateProduct(id, changes: ProductChanges)` takes `changes` from `validate`, and a typo while writing such a value is TS2561, as above.
2. Keep each filter in one function, like `listProducts`, and cover it with a test. The next section's test finds the filter typo that `tsc` cannot.

![tsc catches a sort value up, returnDocument afterr and a typo in a typed ProductChanges value; it lets through a typo in a filter (0 products), in an update (nothing changed) and in the runValidator option (-100 saved)](images/32-mongoose-with-typescript/query-holes.gif)

---

## Tests with a Database in Memory

Session 31's tests used the array. Now the products live in MongoDB, and tests must never touch your real database. **mongodb-memory-server** starts a real, empty MongoDB only for the tests, as in the Node.js course's [Session 28](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/28-testing-apis.md)

```bash
npm install --save-dev mongodb-memory-server
```

```text
added 34 packages, and audited 263 packages in 8s
npm warn allow-scripts 1 package has install scripts not yet covered by allowScripts:
npm warn allow-scripts   mongodb-memory-server@11.3.0 (postinstall: node ./postinstall.js)
npm warn allow-scripts
npm warn allow-scripts Run `npm approve-scripts --allow-scripts-pending` to review, or `npm approve-scripts <pkg>` to allow.
```

The package has an install script that downloads MongoDB, and npm 11.17 (the version we used) warns that you have not approved it yet. You do not need to: when the first test starts the memory server and finds no MongoDB, it downloads one itself, once, and later runs reuse it. We tested mongodb-memory-server 11.3.0.

The test file starts the database before the tests and gives every test the same two products

src/app.test.ts (the start)

```typescript
import { createServer } from "node:http";
import { MongoMemoryServer } from "mongodb-memory-server";
import mongoose from "mongoose";
import { afterAll, beforeAll, beforeEach, describe, expect, it } from "vitest";
import { createApp } from "./app.ts";
import { addProduct, ProductModel } from "./products.ts";

const server = createServer(createApp());
let mongo: MongoMemoryServer;
let baseUrl = "";
let mouseId = "";

beforeAll(async () => {
  mongo = await MongoMemoryServer.create();
  await mongoose.connect(mongo.getUri());
  await ProductModel.init(); // wait until the unique index on name exists
  await new Promise<void>((resolve) => {
    server.listen(0, resolve);
  });
  const address = server.address();
  if (address === null || typeof address === "string") {
    throw new Error("the server has no port");
  }
  baseUrl = `http://localhost:${address.port}`;
});

beforeEach(async () => {
  await ProductModel.deleteMany();
  await addProduct({ name: "Keyboard", pricePaise: 149900 });
  const mouse = await addProduct({ name: "Mouse", pricePaise: 49900 });
  mouseId = mouse.id;
});

afterAll(async () => {
  server.close();
  await mongoose.disconnect();
  await mongo.stop();
});

function sendProduct(method: string, path: string, body: string, token: string): Promise<Response> {
  return fetch(`${baseUrl}${path}`, {
    method,
    headers: { "content-type": "application/json", authorization: `Bearer ${token}` },
    body
  });
}
```

| Part                                  | Why                                                                     |
| ------------------------------------- | ----------------------------------------------------------------------- |
| `let mongo: MongoMemoryServer;`       | Set in `beforeAll`, used in `afterAll`. A variable read inside a callback is not checked for "used before assigned" |
| `mongo.getUri()`                      | The address of the new database, on a free port                         |
| `ProductModel.init()`                 | Mongoose builds the unique index in the background after connecting; this waits for it, so the duplicate test is reliable |
| `beforeEach`                          | Every test starts with exactly Keyboard and Mouse; the Mouse's id is new every time, so it is kept in `mouseId` |
| `afterAll`                            | Close the server, disconnect, stop the database                         |
| `sendProduct`                         | Session 31's `postProduct`, now for `POST` and `PATCH`                  |

The tests themselves change where the ids changed

src/app.test.ts (some of the tests)

```typescript
  it("returns one product", async () => {
    const res = await fetch(`${baseUrl}/products/${mouseId}`);
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ id: mouseId, name: "Mouse", pricePaise: 49900 });
  });

  it("answers 404 for a missing product", async () => {
    const res = await fetch(`${baseUrl}/products/6720a1b2c3d4e5f6a7b8c9d0`);
    expect(res.status).toBe(404);
    expect(await res.json()).toEqual({ error: "product 6720a1b2c3d4e5f6a7b8c9d0 not found" });
  });

  it("answers 400 for an id that is not an ObjectId", async () => {
    const res = await fetch(`${baseUrl}/products/99`);
    expect(res.status).toBe(400);
    expect(await res.json()).toEqual({ error: "invalid _id: 99" });
  });
```

`/products/99` was a 404 in Session 31; now `99` is not even an ObjectId. A new product's id is unknown in advance, so that test checks the rest with `toMatchObject`, and asks the database directly

```typescript
  it("adds a valid product for an admin", async () => {
    const res = await sendProduct("POST", "/products", JSON.stringify({ name: "Webcam", pricePaise: 249900 }), "token-riya");
    expect(res.status).toBe(201);
    expect(await res.json()).toMatchObject({ name: "Webcam", pricePaise: 249900 });
    expect(await ProductModel.countDocuments()).toBe(3);
  });
```

`toEqual({ id: expect.any(String), ... })` would work too, but `expect.any()` returns `any`, and ESLint reports it (`no-unsafe-assignment`). Two more tests cover this session's new answers

```typescript
  it("refuses a second product with the same name", async () => {
    const res = await sendProduct("POST", "/products", JSON.stringify({ name: "Mouse", pricePaise: 59900 }), "token-riya");
    expect(res.status).toBe(409);
    expect(await res.json()).toEqual({ error: "already exists" });
  });

  it("changes the price of a product", async () => {
    const res = await sendProduct("PATCH", `/products/${mouseId}`, JSON.stringify({ pricePaise: 39900 }), "token-riya");
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ id: mouseId, name: "Mouse", pricePaise: 39900 });
  });
```

`npm test`

```text
 RUN  v5.0.5 D:/ts-practice/32-mongoose-api

 ✓ src/app.test.ts > the products API > answers the health check 63ms
 ✓ src/app.test.ts > the products API > returns one product 32ms
 ✓ src/app.test.ts > the products API > answers 404 for a missing product 28ms
 ✓ src/app.test.ts > the products API > answers 400 for an id that is not an ObjectId 27ms
 ✓ src/app.test.ts > the products API > filters by price 25ms
 ✓ src/app.test.ts > the products API > refuses a price given twice 21ms
 ✓ src/app.test.ts > the products API > needs a logged-in user to add a product 16ms
 ✓ src/app.test.ts > the products API > lets only admins add products 25ms
 ✓ src/app.test.ts > the products API > adds a valid product for an admin 35ms
 ✓ src/app.test.ts > the products API > refuses an invalid product and lists the problems 25ms
 ✓ src/app.test.ts > the products API > refuses a body that is not JSON 31ms
 ✓ src/app.test.ts > the products API > refuses a second product with the same name 15ms
 ✓ src/app.test.ts > the products API > changes the price of a product 16ms
 ✓ src/app.test.ts > the products API > tells who is logged in 11ms

 Test Files  1 passed (1)
      Tests  14 passed (14)
   Start at  19:03:05
   Duration  2.07s (tests 56%, import 41%, transform 3%)
```

Fourteen tests against a real MongoDB in a few seconds, and the Node.js course's Jest trap (a driver import that Jest blocks) does not happen with Vitest. Now put the typo of the last section into `listProducts`

```typescript
  const filter = maxPricePaise === undefined ? {} : { pricePiase: { $lte: maxPricePaise } };
```

`npm run check` and `npm run lint` pass. `npm test` does not

```text
 × src/app.test.ts > the products API > filters by price 33ms
   → expected [] to deeply equal [ { …(3) } ]
```

```text
      Tests  1 failed | 13 passed (14)
```

This is the division of work for database code: `tsc` checks the shapes it can see, and a test with a real database checks what only MongoDB knows.

![beforeAll starts an empty MongoDB in memory, connects and waits for the index; beforeEach saves Keyboard and Mouse; 14 tests run; afterAll disconnects and stops it; a typo in the filter makes the price test fail](images/32-mongoose-with-typescript/memory-db.gif)

---

## Mini Example 1: Orders That Point to Products

An order stores which product was bought, as a reference: the product's `_id`. Session 08 promised that `HasTimestamps` comes back with Mongoose; here it is. A first try

src/orders.ts (first try)

```typescript
import { Schema, Types, model } from "mongoose";

export interface Order {
  product: Types.ObjectId;
  quantity: number;
}

const orderSchema = new Schema<Order>({
  product: { type: Schema.Types.ObjectId, ref: "Product", required: true },
  quantity: { type: Number, required: true, min: 1 }
}, { timestamps: true });

export const OrderModel = model("Order", orderSchema);

export async function orderSummary(id: string): Promise<string | undefined> {
  const order = await OrderModel.findById(id).populate("product");
  if (order === null) {
    return undefined;
  }
  return `${order.quantity} x ${order.product.name}, ordered at ${order.createdAt.toISOString()}`;
}
```

`ref: "Product"` names the model the id belongs to. `populate("product")` replaces the id with the product document when the order is loaded (Node.js course, Session 19). `{ timestamps: true }` makes Mongoose set `createdAt` and `updatedAt`. Both work while the code runs, and `tsc` knows neither

```text
src/orders.ts:20:47 - error TS2339: Property 'name' does not exist on type 'ObjectId'.

20   return `${order.quantity} x ${order.product.name}, ordered at ${order.createdAt.toISOString()}`;
                                                 ~~~~

src/orders.ts:20:73 - error TS2339: Property 'createdAt' does not exist on type 'Document<unknown, {}, Order, { id: string; }, DefaultSchemaOptions> & Omit<Order & { _id: ObjectId; } & { __v: number; }, "id"> & HydratedDocumentOverrides<...>'.
```

The type of `product` comes from the interface, which says `ObjectId`, and the interface has no `createdAt`. Tell `tsc` both

src/orders.ts

```typescript
import { Schema, Types, model } from "mongoose";
import type { NewProduct } from "./products.ts";

interface HasTimestamps {
  createdAt: Date;
  updatedAt: Date;
}

export interface Order extends HasTimestamps {
  product: Types.ObjectId;
  quantity: number;
}

const orderSchema = new Schema<Order>({
  product: { type: Schema.Types.ObjectId, ref: "Product", required: true },
  quantity: { type: Number, required: true, min: 1 }
}, { timestamps: true });

export const OrderModel = model("Order", orderSchema);

export async function orderSummary(id: string): Promise<string | undefined> {
  const order = await OrderModel.findById(id).populate<{ product: NewProduct }>("product");
  if (order === null) {
    return undefined;
  }
  return `${order.quantity} x ${order.product.name}, ordered at ${order.createdAt.toISOString()}`;
}
```

| Change                                     | Effect                                                              |
| ------------------------------------------ | ------------------------------------------------------------------- |
| `interface Order extends HasTimestamps`    | Session 08's building block, with `Date`s, because MongoDB stores real dates |
| `populate<{ product: NewProduct }>`        | After this call, `order.product` has the type `NewProduct`          |

`HasTimestamps` is not in the schema, and that is fine: Mongoose adds the two fields itself, and `tsc` does not ask for every interface field in the schema (you saw that above). A script to try it

src/orders-demo.ts

```typescript
import mongoose from "mongoose";
import { config } from "./config/index.ts";
import { OrderModel, orderSummary } from "./orders.ts";
import { ProductModel } from "./products.ts";

await mongoose.connect(config.mongodbUri, { dbName: config.dbName });

const keyboard = await ProductModel.findOne({ name: "Keyboard" });
if (keyboard === null) {
  throw new Error("add the Keyboard first");
}
const order = await OrderModel.create({ product: keyboard._id, quantity: 2 });
console.log(await orderSummary(order.id));

await mongoose.disconnect();
```

```text
2 x Keyboard, ordered at 2026-10-31T13:31:41.345Z
```

The type in `populate<...>` is a promise you make, like `as` (Session 13). A typo in the path, `populate<{ product: NewProduct }>("prodcut")`, passes `tsc`; Mongoose stops it while the code runs

```text
StrictPopulateError: Cannot populate path `prodcut` because it is not in your schema. Set the `strictPopulate` option to false to override.
```

And if the Keyboard is deleted after the order was made, `populate` finds nothing and sets `order.product` to `null`, while the type still says `NewProduct`

```text
TypeError: Cannot read properties of null (reading 'name')
```

When a referenced document can disappear, say so in the type: `populate<{ product: NewProduct | null }>`, and check for `null` (Exercise 6).

![An order stores the product's ObjectId; populate without a type argument keeps the type ObjectId, populate with product NewProduct changes it; HasTimestamps adds createdAt and updatedAt to the interface](images/32-mongoose-with-typescript/populate.gif)

---

## Mini Example 2: Letting Mongoose Write the Type

This course writes the interface first and checks the schema against it. Mongoose can also go the other way: write only the schema and compute the type from it with `InferSchemaType` (a conditional type, Session 18). Product reviews, this way

src/reviews.ts (the start)

```typescript
import { Schema, model, type InferSchemaType } from "mongoose";

const reviewSchema = new Schema({
  product: { type: Schema.Types.ObjectId, ref: "Product", required: true },
  rating: { type: Number, required: true, min: 1, max: 5 },
  comment: String,
  status: { type: String, enum: ["pending", "published"], default: "pending" }
}, { timestamps: true });

export type Review = InferSchemaType<typeof reviewSchema>;
export const ReviewModel = model("Review", reviewSchema);
```

No `<...>` on `new Schema`. `typeof reviewSchema` is the schema's type (Session 15), and `InferSchemaType` reads it. What came out? The "surely wrong value" trick again

```typescript
export function peek(review: Review): number {
  return review;
}
```

```text
src/reviews.ts:18:3 - error TS2322: Type '{ product: ObjectId; rating: number; comment?: string | null; status: "pending" | "published"; } & DefaultTimestampProps' is not assignable to type 'number'.
```

| Schema                                        | Type                                  |
| --------------------------------------------- | ------------------------------------- |
| `{ type: Number, required: true }`            | `rating: number`                      |
| `comment: String`, not required               | `comment?: string \| null`            |
| `enum: ["pending", "published"]` with a default | `status: "pending" \| "published"` (Session 09) |
| `{ timestamps: true }`                        | `DefaultTimestampProps`: `createdAt` and `updatedAt` |

The schema and the type cannot disagree here, because there is only one of them, and even `timestamps` reaches the type. The price is in the optional fields: a field that is not required may be missing or `null`

```typescript
export function reviewLine(review: Review): string {
  return `${review.rating}/5 ${review.comment.trim()}`;
}
```

```text
src/reviews.ts:14:32 - error TS18049: 'review.comment' is possibly 'null' or 'undefined'.
```

Session 11's tools fix it

```typescript
export function reviewLine(review: Review): string {
  return `${review.rating}/5 ${review.comment?.trim() ?? "(no comment)"}`;
}
```

The `enum` became a literal union, so a status the schema does not allow is an error

```typescript
export function archive(review: Review): void {
  review.status = "archived";
}
```

```text
src/reviews.ts:18:3 - error TS2322: Type '"archived"' is not assignable to type '"pending" | "published"'.
```

And the type is only as good as the schema. Forget `required: true` on `rating`, and code that relied on a number finds out

```typescript
export function stars(review: Review): string {
  return "*".repeat(review.rating);
}
```

```text
src/reviews.ts:18:21 - error TS2345: Argument of type 'number | null | undefined' is not assignable to parameter of type 'number'.
  Type 'undefined' is not assignable to type 'number'.
```

A script with two reviews of the Mouse

src/reviews-demo.ts

```typescript
import mongoose from "mongoose";
import { config } from "./config/index.ts";
import { ProductModel } from "./products.ts";
import { ReviewModel, reviewLine } from "./reviews.ts";

await mongoose.connect(config.mongodbUri, { dbName: config.dbName });

const mouse = await ProductModel.findOne({ name: "Mouse" });
if (mouse === null) {
  throw new Error("add the Mouse first");
}
const review = await ReviewModel.create({ product: mouse._id, rating: 4, comment: " Quiet clicks " });
console.log(reviewLine(review), review.status);
const short = await ReviewModel.create({ product: mouse._id, rating: 5 });
console.log(reviewLine(short), short.status);

await mongoose.disconnect();
```

```text
4/5 Quiet clicks pending
5/5 (no comment) pending
```

| Interface first (`new Schema<NewProduct>`)        | Schema first (`InferSchemaType`)                     |
| ------------------------------------------------- | ---------------------------------------------------- |
| The interface is shared with Joi and the API      | The type exists only because of the schema           |
| A field missing from the schema passes `tsc`      | Schema and type always agree                         |
| Optional fields are exactly what you wrote        | Optional fields are `\| null \| undefined`           |
| `timestamps` must be added to the interface       | `timestamps` reaches the type by itself              |

This course keeps interface first for data that crosses the API, because the same interface drives Joi, the routes and the tests. Schema first is handy for data that only the database code uses.

![A schema without a type argument goes into InferSchemaType, which gives rating number, comment optional string or null, status pending or published, plus createdAt and updatedAt; forgetting required makes rating number, null or undefined](images/32-mongoose-with-typescript/infer-schema.gif)

---

## Beginner Mistakes

### Mistake 1: Installing @types/mongoose

`@types/mongoose` is from the time before Mongoose had its own types, and npm marks it deprecated. Its last version contains no types at all, only a dependency on `mongoose` itself: it adds nothing but a line in `package.json`. Mongoose's types come with `npm install mongoose`.

---

### Mistake 2: A field in the interface but not in the schema

`new Schema<NewProduct>` refuses misspelled and wrongly typed fields, but not missing ones. A field the schema does not know is dropped when saving, without an error: the Webcam was saved without its price. Compare the interface and the schema field by field, and let a test save and read a complete document.

---

### Mistake 3: Forgetting await on a database call

`res.status(201).json(addProduct(data))` passes `tsc`, saves the product, and answers `{}`. `if (product === undefined)` on a `Promise` is never true. When a function becomes `async`, read every caller, not only the ones `tsc` lists.

---

### Mistake 4: Using findById without checking for null

`findById`, `findOne` and `findByIdAndUpdate` give `null` when nothing matches. `doc.name` is TS18047; `doc!.name` silences it and crashes later (Session 11). Check `doc === null`, then answer 404.

---

### Mistake 5: Sending documents straight to the client

`res.json(doc)` sends `_id` and `__v`, and later every field you add to the schema, including ones meant to stay private. A function like `toProduct` with the return type `Product` decides what leaves the server, and `tsc` checks it.

---

### Mistake 6: Trusting tsc with filters, updates and options

A typo in a filter finds nothing, a typo in an update changes nothing, and `runValidator` instead of `runValidators` saves a price of -100, all without an error. Build updates from typed values like `ProductChanges`, keep filters in one place, and test them against a real database.

---

### Mistake 7: { new: true } from older tutorials

`findByIdAndUpdate(id, changes, { new: true })` passes `tsc`, works, and prints a warning in Mongoose 9

```text
(node:3496) [MONGOOSE] Warning: mongoose: the `new` option for `findOneAndUpdate()` and `findOneAndReplace()` is deprecated. Use `returnDocument: 'after'` instead.
```

Write `returnDocument: "after"`, whose value `tsc` checks ("afterr" is an error).

---

### Mistake 8: Listening before connecting, or never connecting

Without `mongoose.connect`, Mongoose holds every query and gives up after 10 seconds: "Operation products.find() buffering timed out after 10000ms", which reaches the client as a 500. A route that always takes 10 seconds and fails points here. Connect first, with `await`, and listen after.

---

### Mistake 9: Reading err.code on an unknown error

`err.code === 11000` is fine in the Node.js course's JavaScript and TS18046 in a TypeScript error handler. Narrow first: `err instanceof mongoose.mongo.MongoServerError && err.code === 11000`, and `err instanceof mongoose.Error.CastError` for bad ids.

---

### Mistake 10: Writing the model's type by hand

`const ProductModel: Model<NewProduct> = model(...)` is refused with a twenty-line TS2375: the real type has more type arguments, like the `id` Mongoose adds. Let `tsc` infer the model, and use `HydratedDocument<NewProduct>` when you need to name a document.

---

## Practice Exercises

### Exercise 1

Add `DELETE /products/:id`, only for admins, with `ProductModel.findByIdAndDelete(id)`. Answer 204 when a product was deleted and 404 when not. What type does `findByIdAndDelete` give, and where do you need `Request<{ id: string }>`? Write the tests first.

### Exercise 2

Add `stock: number` to products, with 0 as the default for new ones. Change the interface first and follow `tsc` through the files it lists. Which place does `tsc` not send you to? (Hint: the schema.) Prove with a test that the stock is really saved.

### Exercise 3

Write `src/seed.ts`, run by a new script `"seed": "node --env-file-if-exists=.env src/seed.ts"`: it connects, deletes all products, saves three with `ProductModel.create([...])`, prints how many there are with `countDocuments()`, and disconnects. Use top-level `await`.

### Exercise 4

Add `?q=mou` to `GET /products`: products whose name contains the text, ignoring case. Put `q` in `ProductQuery` and the Joi schema, and build the filter `{ name: { $regex: q, $options: "i" } }` inside `listProducts`. Write a test, then misspell `name` in the filter and watch the test catch it.

### Exercise 5

Add paging: `?page=2&limit=10` with `.skip()` and `.limit()`. Give both Joi rules with defaults (`page` 1, `limit` 20, at most 100). What does `tsc` say if you pass `req.query.limit` to `.limit()` directly, and why does `validate` fix it?

### Exercise 6

Make `orderSummary` survive a deleted product: `populate<{ product: NewProduct | null }>`, and return `"2 x (deleted product)"` when it is `null`. Which line does `tsc` mark as soon as you add `| null`?

### Exercise 7

Add `POST /orders` for logged-in users with a body `{ productId, quantity }`. Validate it with Joi (`productId` as a string of 24 hexadecimal characters: `Joi.string().hex().length(24)`), answer 404 when the product does not exist, and 201 with the order summary otherwise.

### Exercise 8

Rewrite the `Order` model of Mini Example 1 schema first, with `InferSchemaType`, and compare the two types with the "surely wrong value" trick. What happens to `createdAt`, and to `quantity` if you remove `required: true`?

---

## Interview Questions

### Does Mongoose need a @types package?

No. Mongoose ships its own declaration files (`"types": "./types/index.d.ts"` in its package). `@types/mongoose` is deprecated.

### What does new Schema<T>() check, and what does it not?

It checks that every field in the schema exists in `T` and has a matching kind of value (`Number` for `number`), and that rule values like `min` have the right type. It does not check that every field of `T` is in the schema, or that fields without `?` have `required: true`.

### What is a HydratedDocument?

The type of a document loaded or created through a model: your fields, `_id: ObjectId`, `__v: number`, the string `id`, and Mongoose's methods like `save()` and `toJSON()`. `HydratedDocument<T>` is the short name for it.

### What is the difference between _id and id?

`_id` is the `ObjectId` that MongoDB stores. `id` is a property Mongoose adds that returns the same id as a `string`. APIs usually send `id`.

### Why does findById return T | null?

Because no document may have that id. The `null` in the type makes `tsc` ask for a check (TS18047) before the document is used.

### Why can a typo in a filter or an update pass tsc?

Mongoose's filter, update and option types accept names they do not know, because MongoDB queries may name nested paths and operators. A misspelled filter finds nothing and a misspelled update changes nothing. Build updates from typed values and test filters against a real database.

### How do you handle a CastError and a duplicate key in a TypeScript error handler?

`err` is `unknown`, so narrow it: `err instanceof mongoose.Error.CastError` (answer 400) and `err instanceof mongoose.mongo.MongoServerError && err.code === 11000` (answer 409).

### Why validate with Joi when the schema has rules too?

Joi checks the request before any database work and reports all problems in one answer, with a typed result. The schema's rules (and the unique index) are the last line of defense in the database. They also catch what Joi cannot know, like a name that is already taken.

### How do you type populate?

With a type argument that says what the path becomes: `.populate<{ product: NewProduct }>("product")`. Without it, the property keeps the type of the reference, `ObjectId`. The type is not checked against the path, so a typo in the path or a deleted document shows up only while the code runs.

### Interface first or InferSchemaType?

Interface first shares one type with validation, routes and tests, but `tsc` does not ensure every field is in the schema. `InferSchemaType` always matches the schema and includes timestamps, but optional fields become `T | null | undefined`, and a forgotten `required` changes the type.

### Why should a server connect to MongoDB before it listens?

So that it never answers requests it cannot serve. With top-level `await`, `listen` runs only after the connection; if it fails, the server prints why and exits with code 1. A short `serverSelectionTimeoutMS` makes that failure fast.

### How do you test code that uses MongoDB without touching the real database?

Start an empty MongoDB for the tests with mongodb-memory-server, connect Mongoose to `getUri()` in `beforeAll`, reset the data in `beforeEach`, and disconnect and stop it in `afterAll`. `Model.init()` waits for indexes like `unique`.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [31. Express with TypeScript](31-express-with-typescript.md) | [Course outline](README.md#course-outline) | [33. Final Project: Task Management API in TypeScript](33-final-project.md) |
