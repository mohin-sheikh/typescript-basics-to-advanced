# Final Project: Task Management API in TypeScript

> **Session 33 of 33** | Part 7: TypeScript with Node.js | [All sessions](README.md#course-outline)

## Table of Contents
- [Your Folder for This Session](#your-folder-for-this-session)
- [The Plan](#the-plan)
- [Step 1: Settings](#step-1-settings)
- [Step 2: Models](#step-2-models)
- [Step 3: Validation Rules](#step-3-validation-rules)
- [Step 4: Tokens](#step-4-tokens)
- [Step 5: Middleware](#step-5-middleware)
- [Step 6: Controllers](#step-6-controllers)
- [Step 7: Routes, App and Server](#step-7-routes-app-and-server)
- [Step 8: Tests](#step-8-tests)
- [Running the Project](#running-the-project)
- [Mini Example 1: A New Task Status](#mini-example-1-a-new-task-status)
- [Mini Example 2: What select false Hides](#mini-example-2-what-select-false-hides)
- [Where to Go Next](#where-to-go-next)
- [Beginner Mistakes](#beginner-mistakes)
- [Practice Exercises](#practice-exercises)
- [Interview Questions](#interview-questions)

---

## Your Folder for This Session

The Node.js course ends with a **Task Management API** in JavaScript (its [Session 30](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/30-mini-project.md)): users register and log in, and each user creates, lists, changes and deletes their own tasks. Session 28 promised that this course moves that API to TypeScript the way it moved `config/index.js`. This is that session: the same routes, the same rules and the same tests, written with everything from Sessions 1 to 32.

You do not need the Node.js course's project to follow along. Every file is here, and the lesson explains the parts that are new.

Copy the folder `32-mongoose-api` to `33-task-api`, without `node_modules` and `dist`. In `src`, keep only `utils/errors.ts` and `utils/validation.ts`, and delete the rest. Then

```bash
npm pkg set name=33-task-api
npm install
npm install jsonwebtoken bcryptjs
npm install --save-dev @types/jsonwebtoken
```

```text
added 14 packages, and audited 277 packages in 4s
```

```text
added 2 packages, and audited 279 packages in 4s
```

| Package               | Job                                              | Types                                   |
| --------------------- | ------------------------------------------------ | --------------------------------------- |
| `jsonwebtoken`        | Login tokens (JWT)                                | `@types/jsonwebtoken` (Session 25)      |
| `bcryptjs`            | Turns a password into a hash, and checks it      | Its own: `npm view bcryptjs types` prints `umd/index.d.ts` |

`bcryptjs` is new in this course. It has two functions you need, both returning promises: `bcrypt.hash(password, 10)` makes a hash (10 is how much work it costs), and `bcrypt.compare(candidate, hash)` gives `true` when a typed password matches. The Node.js course's [Session 23](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/23-password-hashing-using-bcrypt.md) explains why passwords are stored only as hashes. The versions we tested: jsonwebtoken 9.0.3, bcryptjs 3.0.3, @types/jsonwebtoken 9.0.10.

---

## The Plan

The API has three parts

| Route                                  | Does                                             |
| -------------------------------------- | ------------------------------------------------ |
| `POST /api/auth/register`, `POST /api/auth/login` | An account, and a token                 |
| `GET /api/auth/me`, `PATCH /api/auth/password` | Who am I; a new password (old tokens stop working) |
| `GET`, `POST /api/tasks`               | My tasks (filter by status, page by page); a new task |
| `GET /api/tasks/stats`                 | How many tasks in each status, and how many are overdue |
| `GET`, `PATCH`, `DELETE /api/tasks/:id` | One of **my** tasks                             |
| `PATCH`, `DELETE /api/users/me`        | My name or email; my account and all my tasks    |
| `GET /api/users`                       | All users, for admins only                       |

Every JavaScript file of the Node.js project gets a TypeScript partner, in the same folders

| Node.js course (JavaScript)        | This session (TypeScript)              | Built on        |
| ---------------------------------- | -------------------------------------- | --------------- |
| `src/config/index.js`              | `src/config/index.ts`                  | Sessions 26, 28, 32 |
| `src/constants/taskStatuses.js`    | `src/constants/taskStatuses.ts`        | Session 16      |
| `src/models/User.js`, `Task.js`    | `src/models/user.ts`, `task.ts`        | Session 32      |
| `src/validations/*.js`             | `src/validations/*.ts`                 | Session 26      |
| `generateToken` in a controller    | `src/services/tokens.ts`               | Session 25      |
| `src/middleware/auth.js`, `errorMiddleware.js` | `src/middleware/auth.ts`, `errors.ts` | Sessions 21, 31 |
| `src/utils/responseHelper.js`      | `src/utils/response.ts`                | Session 14      |
| `src/controllers/*.js`, `src/routes/*.js` | `src/controllers/*.ts`, `src/routes/*.ts` | Session 31 |
| `src/app.js`, `server.js`          | `src/app.ts`, `src/server.ts`          | Sessions 30 to 32 |
| `scripts/make-admin.js`            | `src/scripts/make-admin.ts`            | Session 24      |
| `tests/` (Jest and supertest)      | `src/tests/` (Vitest and `fetch`)      | Sessions 29, 32 |

Some parts of the Node.js project use packages that this course has not taught: profile pictures (multer), logging (winston, morgan), API documentation (Swagger), security headers and rate limits (helmet, cors, express-rate-limit). They stay out of this build, and [Where to Go Next](#where-to-go-next) shows how to add each one. Two things change on purpose:

* Answers send `id` (a string) instead of MongoDB's `_id`, and never `__v` (Session 32's `toProduct` idea)
* Successful answers keep the Node.js course's shape, `{ "success": true, "message": ..., "data": ... }`, and error answers are `{ "success": false, "message": ..., "errors": [...] }`

![The JavaScript files of the Node.js course's Task API move one by one into TypeScript files in the same folders; uploads, logging, API docs and rate limits wait for later](images/33-final-project/migration-map.gif)

---

## Step 1: Settings

Session 28 started the move with this file. Now it gets the database (Session 32) and the token settings (Session 26)

.env

```text
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017
DB_NAME=task_manager
JWT_SECRET=a-long-random-secret-for-development
JWT_EXPIRES_IN=7d
```

.env.example

```text
# Copy this file to .env and change the values
NODE_ENV=development
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017
DB_NAME=task_manager
# At least 16 characters. Make one with:
# node -e "console.log(crypto.randomBytes(32).toString('hex'))"
JWT_SECRET=change-me
JWT_EXPIRES_IN=7d
```

src/config/index.ts

```typescript
import Joi from "joi";
import type { StringValue } from "ms";
import { validate } from "../utils/validation.ts";

const NODE_ENVS = ["development", "production", "test"] as const;

interface Env {
  NODE_ENV: (typeof NODE_ENVS)[number];
  PORT: number;
  MONGODB_URI: string;
  DB_NAME: string;
  JWT_SECRET: string;
  JWT_EXPIRES_IN: StringValue;
}

const envSchema = Joi.object<Env, true>({
  NODE_ENV: Joi.string().valid(...NODE_ENVS).default("development"),
  PORT: Joi.number().port().default(5000),
  MONGODB_URI: Joi.string().uri({ scheme: ["mongodb", "mongodb+srv"] }).required(),
  DB_NAME: Joi.string().default("task_manager"),
  JWT_SECRET: Joi.string().min(16).required(),
  JWT_EXPIRES_IN: Joi.string<StringValue>()
    .pattern(/^[0-9]+[smhdwy]$/)
    .default("7d")
    .messages({ "string.pattern.base": "{#label} needs a number and a unit, like 15m, 1h or 7d" })
});

const env = validate(envSchema, process.env, "environment");

export const config = {
  env: env.NODE_ENV,
  port: env.PORT,
  mongodbUri: env.MONGODB_URI,
  dbName: env.DB_NAME,
  jwtSecret: env.JWT_SECRET,
  jwtExpiresIn: env.JWT_EXPIRES_IN
};
```

One line is new: `Joi.string<StringValue>()`. Joi's `string()` has a type argument that says what the rule produces, and the `.pattern(...)` makes it true while the program runs: digits and one unit letter. `StringValue` is the type of `ms` from Session 25 (it comes with `@types/jsonwebtoken`), the type `jwt.sign` wants for `expiresIn`. Write `JWT_EXPIRES_IN: string` instead, as in Session 26, and the error from Session 25 comes back where the token is made

```text
src/services/tokens.ts:20:48 - error TS2769: No overload matches this call.
  The last overload gave the following error.
    Object literal may only specify known properties, and 'expiresIn' does not exist in type 'SignCallback'.

20   return jwt.sign(payload, config.jwtSecret, { expiresIn: config.jwtExpiresIn });
                                                  ~~~~~~~~~
```

The settings are checked once, at startup. A `.env` without `JWT_SECRET` and with `JWT_EXPIRES_IN=3600` stops the server before anything else happens

```text
InvalidDataError: environment is invalid: JWT_SECRET is required; JWT_EXPIRES_IN needs a number and a unit, like 15m, 1h or 7d
```

![The text values of .env go through the Joi schema; JWT_EXPIRES_IN becomes a StringValue that jwt.sign accepts, and a missing secret or 3600 stops the server at startup](images/33-final-project/config-types.gif)

---

## Step 2: Models

One list of task statuses, used by the model, the Joi rules and the statistics (Session 16)

src/constants/taskStatuses.ts

```typescript
// Every status a task can have: the model, the Joi rules and the statistics use this list
export const TASK_STATUSES = ["pending", "in-progress", "completed"] as const;

export type TaskStatus = (typeof TASK_STATUSES)[number];
```

The user model is Session 32's pattern plus two new things: a **hook** that hashes the password before saving, and an **instance method**, `comparePassword`, that every user document has

src/models/user.ts

```typescript
import bcrypt from "bcryptjs";
import { Schema, model, type HydratedDocument, type Model } from "mongoose";

export const ROLES = ["user", "admin"] as const;
export type Role = (typeof ROLES)[number];

export interface User {
  name: string;
  email: string;
  password: string;
  tokenVersion: number;
  role: Role;
  isActive: boolean;
}

interface UserMethods {
  comparePassword(candidate: string): Promise<boolean>;
}

type UserModelType = Model<User, object, UserMethods>;
export type UserDocument = HydratedDocument<User, UserMethods>;

const userSchema = new Schema<User, UserModelType, UserMethods>({
  name: { type: String, required: true, trim: true, minlength: 2, maxlength: 50 },
  email: { type: String, required: true, unique: true, lowercase: true, trim: true },
  password: { type: String, required: true, select: false },
  tokenVersion: { type: Number, default: 0 },
  role: { type: String, enum: ROLES, default: "user" },
  isActive: { type: Boolean, default: true }
}, { timestamps: true });

// Hash the password before saving. A changed password makes older tokens invalid
userSchema.pre("save", async function () {
  if (!this.isModified("password")) {
    return;
  }
  this.password = await bcrypt.hash(this.password, 10);
  if (!this.isNew) {
    this.tokenVersion += 1;
  }
});

userSchema.method("comparePassword", function (candidate: string): Promise<boolean> {
  return bcrypt.compare(candidate, this.password);
});

export const UserModel = model("User", userSchema);

// What the API sends about a user: never the password or the token version
export interface PublicUser {
  id: string;
  name: string;
  email: string;
  role: Role;
}

export function toPublicUser(user: UserDocument): PublicUser {
  return { id: user.id, name: user.name, email: user.email, role: user.role };
}
```

| Part                                        | Why                                                         |
| ------------------------------------------- | ----------------------------------------------------------- |
| `new Schema<User, UserModelType, UserMethods>` | Three type arguments: the stored fields, the model's type, and the methods every document gets |
| `Model<User, object, UserMethods>`          | The model's type. The middle argument is for query helpers; `object` means none |
| `userSchema.pre("save", async function ...)` | A normal `function`, because Mongoose sets `this` to the document (Node.js course, Session 19) |
| `tokenVersion`                              | +1 at every password change; tokens remember the version they were made with (Node.js course, Session 30) |
| `password` with `select: false`             | Queries leave the hash out unless they ask for it: see [Mini Example 2](#mini-example-2-what-select-false-hides) |
| `toPublicUser`                              | What leaves the server, like `PublicUser` in Session 17     |

Why three type arguments? With only `new Schema<User>`, the methods are unknown to `tsc`: ten errors in three files, like

```text
src/controllers/auth.ts:29:37 - error TS2339: Property 'comparePassword' does not exist on type 'Document<unknown, {}, User, { id: string; }, DefaultSchemaOptions> & Omit<User & { _id: ObjectId; } & { __v: number; }, "id"> & HydratedDocumentOverrides<...>'.
```

```text
Found 10 errors in 3 files.

Errors  Files
     7  src/controllers/auth.ts:19
     2  src/controllers/users.ts:13
     1  src/middleware/auth.ts:24
```

Mongoose's documentation writes the middle argument as `{}`. ESLint refuses that, and says what to use

```text
  20:34  error  The `{}` ("empty object") type allows any non-nullish value, including literals like `0` and `""`.
- If you want a type meaning "any object", you probably want `object` instead.
```

With the three arguments, `this` inside the hook and the method is a typed user document, so a typo there is caught

```text
src/models/user.ts:37:8 - error TS2551: Property 'pasword' does not exist on type 'Document<unknown, {}, User, { id: string; }, DefaultSchemaOptions> & Omit<User & { _id: ObjectId; } & { __v: number; }, "comparePassword" | "id"> & HydratedDocumentOverrides<...>'. Did you mean 'password'?
```

But the method's **name** is only a string. Misspell it

```typescript
userSchema.method("comparePasword", function (candidate: string): Promise<boolean> {
```

`npm run check` passes, because `UserMethods` promises a `comparePassword` and nobody compares the promise with the string. The tests find it

```text
TypeError: user.comparePassword is not a function
    at login (D:/ts-practice/33-task-api/src/controllers/auth.ts:29:37)
```

```text
 × src/tests/auth.test.ts > POST /api/auth/login > returns a token for the right password 129ms
   → expected 500 to be 200 // Object.is equality
```

```text
      Tests  3 failed | 7 passed (10)
```

The task model

src/models/task.ts

```typescript
import { Schema, Types, model, type HydratedDocument } from "mongoose";
import { TASK_STATUSES, type TaskStatus } from "../constants/taskStatuses.ts";

export interface Task {
  title: string;
  description: string;
  status: TaskStatus;
  dueDate: Date | null;
  user: Types.ObjectId;
  createdAt: Date;
  updatedAt: Date;
}

const taskSchema = new Schema<Task>({
  title: { type: String, required: true, trim: true, maxlength: 100 },
  description: { type: String, trim: true, maxlength: 1000, default: "" },
  status: { type: String, enum: TASK_STATUSES, default: "pending" },
  dueDate: { type: Date, default: null },
  user: { type: Schema.Types.ObjectId, ref: "User", required: true }
}, { timestamps: true });

// Almost every query is "the newest tasks of one user": an index makes it fast
taskSchema.index({ user: 1, createdAt: -1 });

export const TaskModel = model("Task", taskSchema);
export type TaskDocument = HydratedDocument<Task>;

// What the API sends about a task. Dates become text in JSON, so the type says string
export interface TaskBody {
  id: string;
  title: string;
  description: string;
  status: TaskStatus;
  dueDate: string | null;
  user: string;
  createdAt: string;
  updatedAt: string;
}

export function toTaskBody(task: TaskDocument): TaskBody {
  return {
    id: task.id,
    title: task.title,
    description: task.description,
    status: task.status,
    dueDate: task.dueDate === null ? null : task.dueDate.toISOString(),
    user: task.user.toString(),
    createdAt: task.createdAt.toISOString(),
    updatedAt: task.updatedAt.toISOString()
  };
}
```

| Part                                   | Why                                                          |
| -------------------------------------- | ------------------------------------------------------------ |
| `description` with `default: ""`       | Every task has a description, so the interface needs no `?`  |
| `dueDate: Date \| null`, `default: null` | `null` means "no due date" on purpose (Session 11)        |
| `user: Types.ObjectId`, `ref: "User"`  | Every task belongs to one user (Session 32's orders)         |
| `createdAt`, `updatedAt` in the interface | `timestamps: true` sets them; the interface must say so (Session 32) |
| `TaskBody` with `string` dates         | `res.json` turns a `Date` into text, so the answer's type says `string` |
| `user: task.user.toString()`           | The owner's id as text, like `id`                            |

![The User schema takes three type arguments: the fields, the model type with its methods, and the methods; this in the hook is typed, but the name given to method() is only a string](images/33-final-project/user-model.gif)

---

## Step 3: Validation Rules

Interfaces first, then strict Joi schemas (Session 26). One file per resource

src/validations/auth.ts

```typescript
import Joi from "joi";

export interface Registration {
  name: string;
  email: string;
  password: string;
}

export interface Login {
  email: string;
  password: string;
}

export interface PasswordChange {
  currentPassword: string;
  newPassword: string;
}

// Joi.string() refuses objects like { "$gt": "" } (NoSQL injection)
const email = Joi.string().trim().lowercase().email();
// 6 to 72 characters: bcrypt only uses the first 72 bytes
const password = Joi.string().min(6).max(72);

export const registrationSchema = Joi.object<Registration, true>({
  name: Joi.string().trim().min(2).max(50).required(),
  email: email.required(),
  password: password.required()
});

// Only the shape: a wrong password, short or long, is a 401 and not a 400
export const loginSchema = Joi.object<Login, true>({
  email: email.required(),
  password: Joi.string().required()
});

export const passwordChangeSchema = Joi.object<PasswordChange, true>({
  currentPassword: Joi.string().required(),
  newPassword: password.required()
});
```

src/validations/task.ts

```typescript
import Joi from "joi";
import { TASK_STATUSES, type TaskStatus } from "../constants/taskStatuses.ts";

export interface NewTask {
  title: string;
  description?: string;
  status?: TaskStatus;
  dueDate?: Date | null;
}

export type TaskChanges = Partial<NewTask>;

export interface TaskQuery {
  status?: TaskStatus;
  page: number;
  limit: number;
}

// The rules for each field, written once
const title = Joi.string().trim().max(100);
const description = Joi.string().trim().max(1000).allow(""); // "" clears it
const status = Joi.string().valid(...TASK_STATUSES);
const dueDate = Joi.date().allow(null); // null removes the due date

export const newTaskSchema = Joi.object<NewTask, true>({ title: title.required(), description, status, dueDate });

export const taskChangesSchema = Joi.object<TaskChanges, true>({ title, description, status, dueDate })
  .min(1)
  .messages({ "object.min": "send at least one field to change" });

export const taskQuerySchema = Joi.object<TaskQuery, true>({
  status,
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(10)
});
```

src/validations/user.ts

```typescript
import Joi from "joi";

// Name or email. Passwords change through PATCH /api/auth/password, which hashes them
export interface ProfileChanges {
  name?: string;
  email?: string;
}

export const profileChangesSchema = Joi.object<ProfileChanges, true>({
  name: Joi.string().trim().min(2).max(50),
  email: Joi.string().trim().lowercase().email()
})
  .min(1)
  .messages({ "object.min": "send at least one field to change" });
```

| Rule                                       | Why                                                        |
| ------------------------------------------ | ---------------------------------------------------------- |
| `Joi.string()` for the email               | `{ "$gt": "" }` instead of an email gets a 400 before any query (NoSQL injection, Node.js course Session 22) |
| No `role` and no `user` in any schema      | `stripUnknown` drops them: nobody registers as admin or creates a task for someone else |
| `TaskChanges = Partial<NewTask>` with `.min(1)` | A `PATCH` sends any of the fields, but at least one (Session 17) |
| `page` and `limit` with defaults           | `TaskQuery` has them without `?`: after `validate` they are always numbers |

---

## Step 4: Tokens

The Node.js course made tokens inside a controller and read them in the middleware. Here both live in one small file

src/services/tokens.ts

```typescript
import Joi from "joi";
import jwt from "jsonwebtoken";
import { config } from "../config/index.ts";
import type { UserDocument } from "../models/user.ts";
import { AppError } from "../utils/errors.ts";

// What a token carries: the user's id and the password version it was made with
export interface TokenPayload {
  id: string;
  v: number;
}

const tokenPayloadSchema = Joi.object<TokenPayload, true>({
  id: Joi.string().hex().length(24).required(),
  v: Joi.number().integer().min(0).required()
});

export function createToken(user: UserDocument): string {
  const payload: TokenPayload = { id: user.id, v: user.tokenVersion };
  return jwt.sign(payload, config.jwtSecret, { expiresIn: config.jwtExpiresIn });
}

// A bad signature or an expired token throws here; the error handler answers 401
export function readToken(token: string): TokenPayload {
  const decoded = jwt.verify(token, config.jwtSecret, { algorithms: ["HS256"] });
  const result = tokenPayloadSchema.validate(decoded, { stripUnknown: true });
  if (result.error) {
    throw new AppError("Invalid token. Please log in again.", 401);
  }
  return result.value;
}
```

| Part                                   | Why                                                          |
| -------------------------------------- | ------------------------------------------------------------ |
| `TokenPayload`                         | What the API puts into a token: the user's id and the password version |
| `const payload: TokenPayload`          | A typo in the payload is an error where it is written        |
| `algorithms: ["HS256"]`                | Only accept tokens signed the way this API signs them        |
| `tokenPayloadSchema.validate(decoded, ...)` | `jwt.verify` returns `string \| JwtPayload`, and every property of `JwtPayload` is `any` (Session 25). The signature proves who made the token, not what is inside; Joi checks the inside |
| `stripUnknown: true`                   | `jwt.sign` adds `iat` (issued at) and `exp` (expires); they are not part of `TokenPayload` |

Session 26 wrote the Joi check as `const { error, value } = schema.validate(...)`. `tsc` accepts it here too, but ESLint does not

```typescript
  const { error, value } = tokenPayloadSchema.validate(decoded, { stripUnknown: true });
```

```text
  26:18  error  Unsafe object destructuring of a property with an `any` value  @typescript-eslint/no-unsafe-assignment
```

Before the `error` check, `value` is `any` in one member of the result's union type (Session 26), and destructuring takes it out of the union too early. `result.error`, then `result.value`, keeps the narrowing, as `validate()` in `utils/validation.ts` does.

![createToken puts id and v into a token; readToken verifies the signature, lets Joi check id and v, strips iat and exp, and returns a TokenPayload; a bad signature or an expired token throws](images/33-final-project/token-flow.gif)

---

## Step 5: Middleware

`req.user` is now a real user document (Session 31's `declare global`, with the type from the model)

src/types/express.d.ts

```typescript
import type { UserDocument } from "../models/user.ts";

declare global {
  namespace Express {
    interface Request {
      user?: UserDocument;
    }
  }
}
```

src/middleware/auth.ts

```typescript
import type { Request, RequestHandler } from "express";
import { UserModel, type Role, type UserDocument } from "../models/user.ts";
import { readToken } from "../services/tokens.ts";
import { AppError } from "../utils/errors.ts";

// Only let requests with a valid token through
export const protect: RequestHandler = async (req, res, next) => {
  const header = req.get("authorization") ?? "";
  if (!header.startsWith("Bearer ")) {
    throw new AppError("Not logged in. Please send a token.", 401);
  }
  const payload = readToken(header.slice("Bearer ".length));

  // The user may have been deleted or deactivated after the token was made
  const user = await UserModel.findById(payload.id);
  if (user === null || !user.isActive) {
    throw new AppError("This user no longer exists or is deactivated", 401);
  }
  // A token made before the last password change has an old version number
  if (payload.v !== user.tokenVersion) {
    throw new AppError("Password was changed. Please log in again.", 401);
  }

  req.user = user;
  next();
};

// Only let some roles through (use after protect)
export function authorize(...roles: Role[]): RequestHandler {
  return (req, res, next) => {
    const user = currentUser(req);
    if (!roles.includes(user.role)) {
      throw new AppError(`Role "${user.role}" is not allowed to do this`, 403);
    }
    next();
  };
}

export function currentUser(req: Request): UserDocument {
  if (req.user === undefined) {
    throw new AppError("Not logged in. Please send a token.", 401);
  }
  return req.user;
}
```

| Check in `protect`                       | Fails with                                           |
| ---------------------------------------- | ---------------------------------------------------- |
| The header starts with `Bearer `         | 401 Not logged in. Please send a token.              |
| `readToken`: signature, expiry, payload  | 401 from the error handler (next file)               |
| The user exists and is active            | 401 This user no longer exists or is deactivated     |
| The token's version is the current one   | 401 Password was changed. Please log in again.       |

`authorize(...roles: Role[])` takes only real roles: `authorize("admn")` is an error (Session 09's literal unions). `currentUser(req)` is Session 31's helper, now with the document type.

The error handler turns every error into the API's error shape. The decision is a plain function, `toErrorAnswer`, so it is easy to read and to test

src/middleware/errors.ts

```typescript
import type { ErrorRequestHandler, RequestHandler } from "express";
import jwt from "jsonwebtoken";
import mongoose from "mongoose";
import { AppError } from "../utils/errors.ts";
import { InvalidDataError } from "../utils/validation.ts";

// Every error answer has this shape
export interface ErrorBody {
  success: false;
  message: string;
  errors?: string[];
}

interface ErrorAnswer {
  statusCode: number;
  body: ErrorBody;
}

function fail(statusCode: number, message: string): ErrorAnswer {
  return { statusCode, body: { success: false, message } };
}

// The most specific classes first: TokenExpiredError extends JsonWebTokenError
export function toErrorAnswer(err: unknown): ErrorAnswer {
  if (err instanceof InvalidDataError) {
    return { statusCode: 400, body: { success: false, message: err.message, errors: err.problems } };
  }
  if (err instanceof AppError) {
    return fail(err.statusCode, err.message);
  }
  if (err instanceof SyntaxError) {
    return fail(400, "the body is not valid JSON");
  }
  if (err instanceof jwt.TokenExpiredError) {
    return fail(401, "Your token has expired. Please log in again.");
  }
  if (err instanceof jwt.JsonWebTokenError) {
    return fail(401, "Invalid token. Please log in again.");
  }
  if (err instanceof mongoose.Error.CastError) {
    return fail(400, `invalid ${err.path}: ${err.value}`);
  }
  if (err instanceof mongoose.mongo.MongoServerError && err.code === 11000) {
    return fail(409, "already exists");
  }
  return fail(500, "something went wrong");
}

export const notFound: RequestHandler = (req, res) => {
  const { statusCode, body } = fail(404, `${req.method} ${req.originalUrl} does not exist`);
  res.status(statusCode).json(body);
};

// eslint-disable-next-line @typescript-eslint/no-unused-vars -- Express finds error handlers by their four parameters
export const errorHandler: ErrorRequestHandler = (err: unknown, req, res, next) => {
  const { statusCode, body } = toErrorAnswer(err);
  if (statusCode === 500) {
    console.error(err);
  }
  res.status(statusCode).json(body);
};
```

The order of the checks matters, as in Session 21: `TokenExpiredError` extends `JsonWebTokenError` in `@types/jsonwebtoken`. Swap the two `if` blocks: `tsc` and ESLint are quiet, and an expired token gets the wrong message. The new test for expired tokens (Step 8) notices

```text
 × src/tests/auth.test.ts > protected routes > answers 401 for an expired token 98ms
```

```text
-   "message": "Your token has expired. Please log in again.",
+   "message": "Invalid token. Please log in again.",
```

![protect reads the header, verifies the token, loads the user, compares the version and sets req.user; the error handler checks TokenExpiredError before JsonWebTokenError, and the swapped order answers Invalid token for an expired one](images/33-final-project/protect-errors.gif)

---

## Step 6: Controllers

The answer helper from the Node.js course, now generic (Session 14): `data` keeps its type

src/utils/response.ts

```typescript
import type { Response } from "express";

// Every successful answer has this shape (Node.js course, Session 29)
export interface SuccessBody<T> {
  success: true;
  message: string;
  data: T;
}

export function success<T>(res: Response, data: T, message = "Success", statusCode = 200): void {
  const body: SuccessBody<T> = { success: true, message, data };
  res.status(statusCode).json(body);
}

export function created<T>(res: Response, data: T, message = "Created successfully"): void {
  success(res, data, message, 201);
}
```

src/controllers/auth.ts

```typescript
import type { RequestHandler } from "express";
import { currentUser } from "../middleware/auth.ts";
import { toPublicUser, UserModel, type PublicUser } from "../models/user.ts";
import { createToken } from "../services/tokens.ts";
import { AppError } from "../utils/errors.ts";
import { created, success } from "../utils/response.ts";
import { validate } from "../utils/validation.ts";
import { loginSchema, passwordChangeSchema, registrationSchema } from "../validations/auth.ts";

export interface AuthResult {
  token: string;
  user: PublicUser;
}

// POST /api/auth/register. A role in the body is dropped: everyone starts as a normal user
export const register: RequestHandler = async (req, res) => {
  const data = validate(registrationSchema, req.body, "registration");
  const user = await UserModel.create(data); // a used email: 409 from the error handler
  const result: AuthResult = { token: createToken(user), user: toPublicUser(user) };
  created(res, result, "Registered");
};

// POST /api/auth/login
export const login: RequestHandler = async (req, res) => {
  const { email, password } = validate(loginSchema, req.body, "login");
  const user = await UserModel.findOne({ email }).select("+password");

  // The same message for "no such email" and "wrong password"
  if (user === null || !(await user.comparePassword(password))) {
    throw new AppError("Invalid email or password", 401);
  }
  if (!user.isActive) {
    throw new AppError("Your account has been deactivated", 403);
  }
  const result: AuthResult = { token: createToken(user), user: toPublicUser(user) };
  success(res, result, "Logged in");
};

// GET /api/auth/me
export const getMe: RequestHandler = (req, res) => {
  success(res, toPublicUser(currentUser(req)));
};

// PATCH /api/auth/password
export const changePassword: RequestHandler = async (req, res) => {
  const { currentPassword, newPassword } = validate(passwordChangeSchema, req.body, "password change");

  // req.user was loaded without the password (select: false)
  const user = await UserModel.findById(currentUser(req)._id).select("+password");
  if (user === null || !(await user.comparePassword(currentPassword))) {
    throw new AppError("Current password is wrong", 401);
  }

  user.password = newPassword;
  await user.save(); // hashing and tokenVersion + 1 happen in the save hook
  success(res, { token: createToken(user) }, "Password changed");
};
```

`AuthResult` is exported: the tests and the client script in [Running the Project](#running-the-project) read answers with the same type the controller sends them with.

src/controllers/tasks.ts

```typescript
import type { RequestHandler } from "express";
import { TASK_STATUSES, type TaskStatus } from "../constants/taskStatuses.ts";
import { currentUser } from "../middleware/auth.ts";
import { TaskModel, toTaskBody } from "../models/task.ts";
import { AppError } from "../utils/errors.ts";
import { created, success } from "../utils/response.ts";
import { validate } from "../utils/validation.ts";
import { newTaskSchema, taskChangesSchema, taskQuerySchema } from "../validations/task.ts";

export type TaskStats = Record<TaskStatus, number> & { total: number; overdue: number };

// GET /api/tasks?status=pending&page=1&limit=10
export const getTasks: RequestHandler = async (req, res) => {
  const user = currentUser(req);
  const { status, page, limit } = validate(taskQuerySchema, req.query, "query");

  // Only my tasks, and only one status when asked
  const filter = status === undefined ? { user: user._id } : { user: user._id, status };
  const tasks = await TaskModel.find(filter)
    .sort({ createdAt: -1, _id: -1 })
    .skip((page - 1) * limit)
    .limit(limit);
  const total = await TaskModel.countDocuments(filter);

  success(res, { tasks: tasks.map(toTaskBody), page, limit, total, totalPages: Math.ceil(total / limit) });
};

// GET /api/tasks/stats
export const getStats: RequestHandler = async (req, res) => {
  const user = currentUser(req);
  const stats: TaskStats = { pending: 0, "in-progress": 0, completed: 0, total: 0, overdue: 0 };

  for (const status of TASK_STATUSES) {
    stats[status] = await TaskModel.countDocuments({ user: user._id, status });
    stats.total += stats[status];
  }

  // Not completed, and the due date is in the past
  stats.overdue = await TaskModel.countDocuments({
    user: user._id,
    status: { $ne: "completed" },
    dueDate: { $lt: new Date() }
  });

  success(res, stats);
};

// POST /api/tasks. The owner comes from the token, never from the body
export const createTask: RequestHandler = async (req, res) => {
  const data = validate(newTaskSchema, req.body, "task");
  const task = await TaskModel.create({ ...data, user: currentUser(req)._id });
  created(res, toTaskBody(task), "Task created");
};

// GET /api/tasks/:id. _id AND user: someone else's task is "not found" for you
export const getTask: RequestHandler<{ id: string }> = async (req, res) => {
  const task = await TaskModel.findOne({ _id: req.params.id, user: currentUser(req)._id });
  if (task === null) {
    throw new AppError("Task not found", 404);
  }
  success(res, toTaskBody(task));
};

// PATCH /api/tasks/:id
export const updateTask: RequestHandler<{ id: string }> = async (req, res) => {
  const changes = validate(taskChangesSchema, req.body, "changes");
  const task = await TaskModel.findOneAndUpdate(
    { _id: req.params.id, user: currentUser(req)._id },
    changes,
    { returnDocument: "after", runValidators: true }
  );
  if (task === null) {
    throw new AppError("Task not found", 404);
  }
  success(res, toTaskBody(task), "Task updated");
};

// DELETE /api/tasks/:id
export const deleteTask: RequestHandler<{ id: string }> = async (req, res) => {
  const task = await TaskModel.findOneAndDelete({ _id: req.params.id, user: currentUser(req)._id });
  if (task === null) {
    throw new AppError("Task not found", 404);
  }
  success(res, null, "Task deleted");
};
```

| Part                                       | Why                                                      |
| ------------------------------------------ | -------------------------------------------------------- |
| `RequestHandler<{ id: string }>`           | A named handler does not see the route text, so it says which parameters it needs (Session 31) |
| `{ _id: req.params.id, user: currentUser(req)._id }` | This task, **only if it is mine**. Someone else's task is "not found": a 403 would tell that it exists |
| `TaskStats = Record<TaskStatus, number> & ...` | One count for every status, checked by `tsc` (Session 17) |
| `.sort({ createdAt: -1, _id: -1 })`        | Newest first; `_id` decides between tasks made in the same millisecond |

src/controllers/users.ts

```typescript
import type { RequestHandler } from "express";
import { currentUser } from "../middleware/auth.ts";
import { TaskModel } from "../models/task.ts";
import { toPublicUser, UserModel } from "../models/user.ts";
import { AppError } from "../utils/errors.ts";
import { success } from "../utils/response.ts";
import { validate } from "../utils/validation.ts";
import { profileChangesSchema } from "../validations/user.ts";

// GET /api/users (admins only)
export const getUsers: RequestHandler = async (req, res) => {
  const users = await UserModel.find().sort({ name: 1 });
  success(res, users.map(toPublicUser));
};

// PATCH /api/users/me: name or email
export const updateMe: RequestHandler = async (req, res) => {
  const changes = validate(profileChangesSchema, req.body, "profile");
  const user = await UserModel.findByIdAndUpdate(currentUser(req)._id, changes, {
    returnDocument: "after",
    runValidators: true
  });
  if (user === null) {
    throw new AppError("This user no longer exists", 404);
  }
  success(res, toPublicUser(user), "Profile updated");
};

// DELETE /api/users/me: the tasks first, then the account
export const deleteMe: RequestHandler = async (req, res) => {
  const user = currentUser(req);
  await TaskModel.deleteMany({ user: user._id });
  await UserModel.findByIdAndDelete(user._id);
  success(res, null, "Account deleted");
};
```

![A task document goes through toTaskBody: ObjectIds become strings, dates become ISO text, and the answer is wrapped as success true, message and data](images/33-final-project/task-body.gif)

---

## Step 7: Routes, App and Server

src/routes/index.ts

```typescript
import express from "express";
import { authRouter } from "./auth.ts";
import { tasksRouter } from "./tasks.ts";
import { usersRouter } from "./users.ts";

// Every /api route in one place
export const apiRouter = express.Router();

apiRouter.get("/health", (req, res) => {
  res.json({ status: "OK" });
});

apiRouter.use("/auth", authRouter);
apiRouter.use("/tasks", tasksRouter);
apiRouter.use("/users", usersRouter);
```

src/routes/auth.ts

```typescript
import express from "express";
import { changePassword, getMe, login, register } from "../controllers/auth.ts";
import { protect } from "../middleware/auth.ts";

export const authRouter = express.Router();

authRouter.post("/register", register);
authRouter.post("/login", login);
authRouter.get("/me", protect, getMe);
authRouter.patch("/password", protect, changePassword);
```

src/routes/tasks.ts

```typescript
import express from "express";
import { createTask, deleteTask, getStats, getTask, getTasks, updateTask } from "../controllers/tasks.ts";
import { protect } from "../middleware/auth.ts";

export const tasksRouter = express.Router();

// Every task route needs a logged-in user
tasksRouter.use(protect);

tasksRouter.route("/").get(getTasks).post(createTask);
tasksRouter.get("/stats", getStats); // before /:id, or "stats" would be read as an id
tasksRouter.route("/:id").get(getTask).patch(updateTask).delete(deleteTask);
```

src/routes/users.ts

```typescript
import express from "express";
import { deleteMe, getUsers, updateMe } from "../controllers/users.ts";
import { authorize, protect } from "../middleware/auth.ts";

export const usersRouter = express.Router();

usersRouter.use(protect);

usersRouter.get("/", authorize("admin"), getUsers);
usersRouter.route("/me").patch(updateMe).delete(deleteMe);
```

`tasksRouter.use(protect)` puts the check in front of every task route, so no route can forget it. Here is a trap that `tsc` does not see. Rename the parameter in the route

```typescript
tasksRouter.route("/:taskId").get(getTask).patch(updateTask).delete(deleteTask);
```

`npm run check` and `npm run lint` pass: `getTask` declared `{ id: string }` itself, and Express's types do not compare that with the route text. While the code runs, `req.params.id` is `undefined`, so no task is ever found

```text
 × src/tests/tasks.test.ts > one task > marks a task completed, and refuses an unknown status 143ms
   → expected 404 to be 200 // Object.is equality
 × src/tests/tasks.test.ts > one task > deletes a task once 204ms
   → expected [ 404, 404 ] to deeply equal [ 200, 404 ]
 × src/tests/tasks.test.ts > one task > answers 400 for an id that is not an ObjectId 136ms
   → expected 404 to be 400 // Object.is equality
```

Session 31's inline handlers read the parameters from the route text. Named handlers trade that for controllers in their own files, and the tests take over the check.

The app

src/app.ts

```typescript
import express, { type Express } from "express";
import { errorHandler, notFound } from "./middleware/errors.ts";
import { apiRouter } from "./routes/index.ts";

export function createApp(): Express {
  const app = express();
  app.use(express.json());
  app.use("/api", apiRouter);
  app.use(notFound);
  app.use(errorHandler);
  return app;
}
```

`src/server.ts` is Session 32's file without a single change: it reads `config`, connects, and listens.

The script that makes someone an admin (the Node.js course's Session 22). With top-level `await` (Session 24), and `process.argv[2]` is `string | undefined` (Session 06)

src/scripts/make-admin.ts

```typescript
// Usage: node --env-file=.env src/scripts/make-admin.ts someone@example.com
import mongoose from "mongoose";
import { config } from "../config/index.ts";
import { UserModel } from "../models/user.ts";

const email = process.argv[2];
if (email === undefined) {
  console.log("Usage: node --env-file=.env src/scripts/make-admin.ts <email>");
  process.exit(1);
}

await mongoose.connect(config.mongodbUri, { dbName: config.dbName });
try {
  const user = await UserModel.findOneAndUpdate(
    { email: email.toLowerCase() },
    { role: "admin" },
    { returnDocument: "after" }
  );
  console.log(user === null ? `No user with email ${email}` : `${user.email} is now an admin`);
} finally {
  await mongoose.disconnect();
}
```

![The route text /:taskId and the handler's own params type id do not meet: tsc is quiet, req.params.id is undefined, and every single-task route answers 404](images/33-final-project/route-params.gif)

---

## Step 8: Tests

The config needs `MONGODB_URI` and `JWT_SECRET`, and Vitest does not read `.env`. A small Vitest settings file gives the tests their own values

vitest.config.ts

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    // The tests start their own MongoDB in memory; these values only satisfy src/config
    env: {
      MONGODB_URI: "mongodb://127.0.0.1:27017",
      JWT_SECRET: "a-secret-only-for-the-tests"
    }
  }
});
```

Without it, every test file stops before its first test

```text
InvalidDataError: environment is invalid: MONGODB_URI is required; JWT_SECRET is required
```

The tests live in `src/tests/`. Their helper file is not a test, so `tsconfig.build.json` must leave the whole folder out of the build (Session 29); with only `src/**/*.test.ts` excluded, `dist/tests/helpers.js` appears in the build

tsconfig.build.json

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts", "src/tests"]
}
```

The helpers: one database and one server per test file (Session 32), a short `api()` for requests, and `dataOf<T>()`, which reads the data of an answer with the controllers' own types

src/tests/helpers.ts

```typescript
import { createServer } from "node:http";
import { MongoMemoryServer } from "mongodb-memory-server";
import mongoose from "mongoose";
import { afterAll, beforeAll, beforeEach } from "vitest";
import { createApp } from "../app.ts";
import type { AuthResult } from "../controllers/auth.ts";
import { TaskModel } from "../models/task.ts";
import { UserModel } from "../models/user.ts";
import type { SuccessBody } from "../utils/response.ts";

let baseUrl = "";

// One test file: an empty MongoDB in memory and the API on a free port; every test starts with no data
export function useTestApi(): void {
  const server = createServer(createApp());
  let mongo: MongoMemoryServer;

  beforeAll(async () => {
    mongo = await MongoMemoryServer.create();
    await mongoose.connect(mongo.getUri());
    await UserModel.init(); // wait until the unique index on email exists
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
    await TaskModel.deleteMany();
    await UserModel.deleteMany();
  });

  afterAll(async () => {
    server.close();
    await mongoose.disconnect();
    await mongo.stop();
  });
}

interface RequestOptions {
  body?: unknown;
  token?: string;
}

export function api(method: string, path: string, options: RequestOptions = {}): Promise<Response> {
  const headers: Record<string, string> = { "content-type": "application/json" };
  if (options.token !== undefined) {
    headers.authorization = `Bearer ${options.token}`;
  }
  return fetch(`${baseUrl}/api${path}`, {
    method,
    headers,
    body: options.body === undefined ? null : JSON.stringify(options.body)
  });
}

// The data of a successful answer. The type is the one the controllers use
export async function dataOf<T>(res: Response): Promise<T> {
  const body = (await res.json()) as SuccessBody<T>;
  return body.data;
}

// Register a user and return the token: most tests need a logged-in user
export async function registerUser(name = "Sara", email = "sara@example.com"): Promise<string> {
  const res = await api("POST", "/auth/register", { body: { name, email, password: "MySecret123" } });
  const { token } = await dataOf<AuthResult>(res);
  return token;
}
```

`dataOf<T>` uses `as`: the answer comes from outside the program (Session 13). In a test that is acceptable, because a wrong shape makes the test fail anyway. The tests are the Node.js course's tests, moved from Jest and supertest to Vitest and `fetch`, plus two new ones: an expired token, and an id that is not an ObjectId

src/tests/auth.test.ts

```typescript
import jwt from "jsonwebtoken";
import { describe, expect, it } from "vitest";
import { config } from "../config/index.ts";
import type { AuthResult } from "../controllers/auth.ts";
import { UserModel, type PublicUser } from "../models/user.ts";
import { readToken } from "../services/tokens.ts";
import { api, dataOf, registerUser, useTestApi } from "./helpers.ts";

useTestApi();

describe("POST /api/auth/register", () => {
  it("creates a normal user and returns a token", async () => {
    const res = await api("POST", "/auth/register", {
      body: { name: "Sara", email: "sara@example.com", password: "MySecret123", role: "admin" }
    });
    expect(res.status).toBe(201);
    const { token, user } = await dataOf<AuthResult>(res);
    expect(token).toMatch(/^ey/);
    expect(user).toMatchObject({ name: "Sara", email: "sara@example.com", role: "user" });

    const saved = await UserModel.findOne({ email: "sara@example.com" }).select("+password");
    expect(saved?.password).toMatch(/^\$2b\$10\$/); // a bcrypt hash, not the password
  });

  it("answers 409 for an email that is already used", async () => {
    await registerUser();
    const res = await api("POST", "/auth/register", {
      body: { name: "Another Sara", email: "sara@example.com", password: "MySecret123" }
    });
    expect(res.status).toBe(409);
  });

  it("answers 400 for a short password", async () => {
    const res = await api("POST", "/auth/register", { body: { name: "Sara", email: "sara@example.com", password: "123" } });
    expect(res.status).toBe(400);
    expect(await res.json()).toHaveProperty("errors", ["password length must be at least 6 characters long"]);
  });
});

describe("POST /api/auth/login", () => {
  it("returns a token for the right password", async () => {
    await registerUser();
    const res = await api("POST", "/auth/login", { body: { email: "sara@example.com", password: "MySecret123" } });
    expect(res.status).toBe(200);
    expect((await dataOf<AuthResult>(res)).token).toMatch(/^ey/);
  });

  it("answers 401 for a wrong password", async () => {
    await registerUser();
    const res = await api("POST", "/auth/login", { body: { email: "sara@example.com", password: "wrong-password" } });
    expect(res.status).toBe(401);
    expect(await res.json()).toEqual({ success: false, message: "Invalid email or password" });
  });

  it("refuses an object instead of an email (NoSQL injection)", async () => {
    await registerUser();
    const res = await api("POST", "/auth/login", { body: { email: { $gt: "" }, password: "MySecret123" } });
    expect(res.status).toBe(400);
    expect(await res.json()).toHaveProperty("errors", ["email must be a string"]);
  });
});

describe("protected routes", () => {
  it("GET /api/auth/me works with a token", async () => {
    const token = await registerUser();
    const res = await api("GET", "/auth/me", { token });
    expect(res.status).toBe(200);
    expect((await dataOf<PublicUser>(res)).email).toBe("sara@example.com");
  });

  it("answers 401 without a token, or with a fake one", async () => {
    const noToken = await api("GET", "/auth/me");
    const fake = await api("GET", "/auth/me", { token: "not.a.token" });
    expect(noToken.status).toBe(401);
    expect(fake.status).toBe(401);
    expect(await fake.json()).toEqual({ success: false, message: "Invalid token. Please log in again." });
  });

  it("after a password change, the old token stops working", async () => {
    const oldToken = await registerUser();
    const change = await api("PATCH", "/auth/password", {
      token: oldToken,
      body: { currentPassword: "MySecret123", newPassword: "NewSecret456" }
    });
    expect(change.status).toBe(200);
    const { token: newToken } = await dataOf<{ token: string }>(change);

    const withOld = await api("GET", "/auth/me", { token: oldToken });
    expect(withOld.status).toBe(401);
    expect(await withOld.json()).toEqual({ success: false, message: "Password was changed. Please log in again." });
    expect((await api("GET", "/auth/me", { token: newToken })).status).toBe(200);
  });

  it("answers 401 for an expired token", async () => {
    const payload = readToken(await registerUser());
    const expired = jwt.sign({ ...payload, exp: Math.floor(Date.now() / 1000) - 60 }, config.jwtSecret);
    const res = await api("GET", "/auth/me", { token: expired });
    expect(res.status).toBe(401);
    expect(await res.json()).toEqual({ success: false, message: "Your token has expired. Please log in again." });
  });
});
```

src/tests/tasks.test.ts

```typescript
import { beforeEach, describe, expect, it } from "vitest";
import type { PublicUser } from "../models/user.ts";
import { TaskModel, type TaskBody } from "../models/task.ts";
import { api, dataOf, registerUser, useTestApi } from "./helpers.ts";

useTestApi();

interface TaskPage {
  tasks: TaskBody[];
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}

let token = ""; // Sara's token, new for every test
beforeEach(async () => {
  token = await registerUser();
});

function createTask(body: object, as = token): Promise<Response> {
  return api("POST", "/tasks", { body, token: as });
}

describe("POST /api/tasks", () => {
  it("creates a task for the logged-in user", async () => {
    const res = await createTask({ title: "Write the report", user: "6acb1f0e8c1d2a0012345678" });
    expect(res.status).toBe(201);
    const task = await dataOf<TaskBody>(res);
    expect(task).toMatchObject({ title: "Write the report", description: "", status: "pending", dueDate: null });
    const me = await dataOf<PublicUser>(await api("GET", "/auth/me", { token }));
    expect(task.user).toBe(me.id); // the owner comes from the token, not from the body
  });

  it("answers 400 without a title or with a bad date", async () => {
    const noTitle = await createTask({ description: "No title" });
    const badDate = await createTask({ title: "Report", dueDate: "next Friday" });
    expect(noTitle.status).toBe(400);
    expect(await noTitle.json()).toHaveProperty("errors", ["title is required"]);
    expect(await badDate.json()).toHaveProperty("errors", ["dueDate must be a valid date"]);
  });

  it("answers 401 without a token", async () => {
    const res = await api("POST", "/tasks", { body: { title: "Report" } });
    expect(res.status).toBe(401);
  });
});

describe("GET /api/tasks", () => {
  it("shows only my own tasks", async () => {
    const benToken = await registerUser("Ben", "ben@example.com");
    await createTask({ title: "Sara's task" });
    await createTask({ title: "Ben's task" }, benToken);
    const page = await dataOf<TaskPage>(await api("GET", "/tasks", { token }));
    expect(page.total).toBe(1);
    expect(page.tasks[0]?.title).toBe("Sara's task");
  });

  it("pages: 12 tasks, page 2 with 5 per page", async () => {
    for (let i = 1; i <= 12; i++) {
      await createTask({ title: `Task ${i}` });
    }
    const page = await dataOf<TaskPage>(await api("GET", "/tasks?page=2&limit=5", { token }));
    expect(page.tasks).toHaveLength(5);
    expect(page.total).toBe(12);
    expect(page.totalPages).toBe(3);
    expect(page.tasks[0]?.title).toBe("Task 7"); // newest first: 12 to 8, then 7 to 3
  });

  it("filters by status, and refuses an unknown status", async () => {
    await createTask({ title: "A", status: "completed" });
    await createTask({ title: "B" });
    const done = await dataOf<TaskPage>(await api("GET", "/tasks?status=completed", { token }));
    const bad = await api("GET", "/tasks?status=done", { token });
    expect(done.total).toBe(1);
    expect(bad.status).toBe(400);
    expect(await bad.json()).toHaveProperty("errors", ["status must be one of [pending, in-progress, completed]"]);
  });

  it("refuses a page or limit that is not a sensible number", async () => {
    const res = await api("GET", "/tasks?page=abc&limit=500", { token });
    expect(res.status).toBe(400);
    expect(await res.json()).toHaveProperty("errors", ["page must be a number", "limit must be less than or equal to 100"]);
  });
});

describe("one task", () => {
  it("another user's task is not found", async () => {
    const benToken = await registerUser("Ben", "ben@example.com");
    const bens = await dataOf<TaskBody>(await createTask({ title: "Ben's secret" }, benToken));
    const url = `/tasks/${bens.id}`;
    const read = await api("GET", url, { token });
    const change = await api("PATCH", url, { token, body: { title: "Hacked" } });
    const remove = await api("DELETE", url, { token });
    expect([read.status, change.status, remove.status]).toEqual([404, 404, 404]);
    expect((await TaskModel.findById(bens.id))?.title).toBe("Ben's secret"); // unchanged
  });

  it("marks a task completed, and refuses an unknown status", async () => {
    const task = await dataOf<TaskBody>(await createTask({ title: "Report" }));
    const url = `/tasks/${task.id}`;
    const done = await api("PATCH", url, { token, body: { status: "completed" } });
    const bad = await api("PATCH", url, { token, body: { status: "done" } });
    expect(done.status).toBe(200);
    expect((await dataOf<TaskBody>(done)).status).toBe("completed");
    expect(bad.status).toBe(400);
  });

  it("deletes a task once", async () => {
    const task = await dataOf<TaskBody>(await createTask({ title: "Report" }));
    const first = await api("DELETE", `/tasks/${task.id}`, { token });
    const second = await api("DELETE", `/tasks/${task.id}`, { token });
    expect([first.status, second.status]).toEqual([200, 404]);
  });

  it("answers 400 for an id that is not an ObjectId", async () => {
    const res = await api("GET", "/tasks/abc", { token });
    expect(res.status).toBe(400);
  });
});

it("GET /api/tasks/stats counts my tasks", async () => {
  await createTask({ title: "A" });
  await createTask({ title: "B", status: "in-progress" });
  await createTask({ title: "C", status: "completed" });
  await createTask({ title: "D", dueDate: "2020-01-01" }); // pending and in the past
  const res = await api("GET", "/tasks/stats", { token });
  expect(await dataOf<unknown>(res)).toEqual({ pending: 2, "in-progress": 1, completed: 1, total: 4, overdue: 1 });
});
```

src/tests/users.test.ts

```typescript
import { expect, it } from "vitest";
import { TaskModel } from "../models/task.ts";
import { UserModel, type PublicUser } from "../models/user.ts";
import { api, dataOf, registerUser, useTestApi } from "./helpers.ts";

useTestApi();

it("PATCH /api/users/me changes the name, but never the password", async () => {
  const token = await registerUser();
  const res = await api("PATCH", "/users/me", { token, body: { name: "Sara Khan", password: "hacked" } });
  expect(res.status).toBe(200);
  expect((await dataOf<PublicUser>(res)).name).toBe("Sara Khan");
  const login = await api("POST", "/auth/login", { body: { email: "sara@example.com", password: "MySecret123" } });
  expect(login.status).toBe(200); // the old password still works
});

it("DELETE /api/users/me deletes the account and its tasks", async () => {
  const token = await registerUser();
  await api("POST", "/tasks", { token, body: { title: "Report" } });
  const res = await api("DELETE", "/users/me", { token });
  expect(res.status).toBe(200);
  expect(await UserModel.countDocuments()).toBe(0);
  expect(await TaskModel.countDocuments()).toBe(0);
  const after = await api("GET", "/auth/me", { token });
  expect(after.status).toBe(401); // the token of a deleted user is useless
});

it("GET /api/users is for admins only", async () => {
  const saraToken = await registerUser();
  const adminToken = await registerUser("Admin", "admin@example.com");
  await UserModel.updateOne({ email: "admin@example.com" }, { role: "admin" }); // like make-admin.ts
  const asUser = await api("GET", "/users", { token: saraToken });
  const asAdmin = await api("GET", "/users", { token: adminToken });
  expect(asUser.status).toBe(403);
  expect(asAdmin.status).toBe(200);
  expect(await dataOf<PublicUser[]>(asAdmin)).toHaveLength(2);
});
```

`npm test`. The three files run at the same time, each with its own MongoDB, so their lines mix

```text
 RUN  v5.0.5 D:/ts-practice/33-task-api

 ✓ src/tests/auth.test.ts > POST /api/auth/register > creates a normal user and returns a token 213ms
 ✓ src/tests/users.test.ts > PATCH /api/users/me changes the name, but never the password 363ms
 ✓ src/tests/tasks.test.ts > POST /api/tasks > creates a task for the logged-in user 235ms
 ✓ src/tests/auth.test.ts > POST /api/auth/register > answers 409 for an email that is already used 251ms
 ✓ src/tests/tasks.test.ts > POST /api/tasks > answers 400 without a title or with a bad date 179ms
 ✓ src/tests/auth.test.ts > POST /api/auth/register > answers 400 for a short password 9ms
 ✓ src/tests/users.test.ts > DELETE /api/users/me deletes the account and its tasks 180ms
 ✓ src/tests/tasks.test.ts > POST /api/tasks > answers 401 without a token 121ms
 ✓ src/tests/auth.test.ts > POST /api/auth/login > returns a token for the right password 246ms
 ✓ src/tests/users.test.ts > GET /api/users is for admins only 288ms
 ✓ src/tests/tasks.test.ts > GET /api/tasks > shows only my own tasks 296ms
 ✓ src/tests/auth.test.ts > POST /api/auth/login > answers 401 for a wrong password 247ms
 ✓ src/tests/auth.test.ts > POST /api/auth/login > refuses an object instead of an email (NoSQL injection) 122ms
 ✓ src/tests/tasks.test.ts > GET /api/tasks > pages: 12 tasks, page 2 with 5 per page 309ms
 ✓ src/tests/auth.test.ts > protected routes > GET /api/auth/me works with a token 110ms
 ✓ src/tests/auth.test.ts > protected routes > answers 401 without a token, or with a fake one 13ms
 ✓ src/tests/tasks.test.ts > GET /api/tasks > filters by status, and refuses an unknown status 161ms
 ✓ src/tests/tasks.test.ts > GET /api/tasks > refuses a page or limit that is not a sensible number 115ms
 ✓ src/tests/auth.test.ts > protected routes > after a password change, the old token stops working 347ms
 ✓ src/tests/auth.test.ts > protected routes > answers 401 for an expired token 118ms
 ✓ src/tests/tasks.test.ts > one task > another user's task is not found 309ms
 ✓ src/tests/tasks.test.ts > one task > marks a task completed, and refuses an unknown status 136ms
 ✓ src/tests/tasks.test.ts > one task > deletes a task once 141ms
 ✓ src/tests/tasks.test.ts > one task > answers 400 for an id that is not an ObjectId 132ms
 ✓ src/tests/tasks.test.ts > GET /api/tasks/stats counts my tasks 247ms

 Test Files  3 passed (3)
      Tests  25 passed (25)
   Start at  19:36:23
   Duration  4.12s (tests 73%, import 26%, transform 2%)
```

`npm run check`, `npm run lint` and `npm run build` pass too, and the build has every source file and no test file

```text
dist/app.js
dist/config/index.js
dist/constants/taskStatuses.js
dist/controllers/auth.js
dist/controllers/tasks.js
dist/controllers/users.js
dist/middleware/auth.js
dist/middleware/errors.js
dist/models/task.js
dist/models/user.js
dist/routes/auth.js
dist/routes/index.js
dist/routes/tasks.js
dist/routes/users.js
dist/scripts/make-admin.js
dist/scripts/try-api.js
dist/server.js
dist/services/tokens.js
dist/utils/errors.js
dist/utils/response.js
dist/utils/validation.js
dist/validations/auth.js
dist/validations/task.js
dist/validations/user.js
```

![vitest.config.ts gives the tests their settings; each of the three test files starts its own MongoDB in memory and its own server, and 25 tests pass in about 4 seconds](images/33-final-project/test-setup.gif)

---

## Running the Project

With MongoDB running (Session 32), `npm run dev`

```text
connected to MongoDB, database task_manager
development server on http://localhost:5000
```

A client script walks through the API. It runs in a second terminal and uses the API's own types; the answer is a union on `success` (Session 10), so `answer.data` is only read after the check

src/scripts/try-api.ts

```typescript
// A walk through the API. Start the server first (npm run dev), then in a second terminal:
//   node src/scripts/try-api.ts
import type { AuthResult } from "../controllers/auth.ts";
import type { TaskStats } from "../controllers/tasks.ts";
import type { ErrorBody } from "../middleware/errors.ts";
import type { TaskBody } from "../models/task.ts";
import { errorMessage } from "../utils/errors.ts";
import type { SuccessBody } from "../utils/response.ts";

const BASE_URL = "http://localhost:5000/api";

// Send a request; give back the data of a successful answer, or throw the error message
async function call<T>(method: string, path: string, token?: string, body?: unknown): Promise<T> {
  const headers: Record<string, string> = { "content-type": "application/json" };
  if (token !== undefined) {
    headers.authorization = `Bearer ${token}`;
  }
  const res = await fetch(`${BASE_URL}${path}`, {
    method,
    headers,
    body: body === undefined ? null : JSON.stringify(body)
  });
  const answer = (await res.json()) as SuccessBody<T> | ErrorBody;
  if (!answer.success) {
    throw new Error(`${res.status} ${answer.message}`);
  }
  return answer.data;
}

const sara = await call<AuthResult>("POST", "/auth/register", undefined, {
  name: "Sara",
  email: "sara@example.com",
  password: "MySecret123"
});
console.log(`registered ${sara.user.name} as ${sara.user.role}`);

const report = await call<TaskBody>("POST", "/tasks", sara.token, { title: "Write the report", dueDate: "2026-11-15" });
console.log(`created "${report.title}": ${report.status}, due ${report.dueDate}`);

const started = await call<TaskBody>("PATCH", `/tasks/${report.id}`, sara.token, { status: "in-progress" });
console.log(`changed "${started.title}" to ${started.status}`);

await call<TaskBody>("POST", "/tasks", sara.token, { title: "Pay the bills", dueDate: "2026-10-01" });
console.log("stats:", await call<TaskStats>("GET", "/tasks/stats", sara.token));

const ben = await call<AuthResult>("POST", "/auth/register", undefined, {
  name: "Ben",
  email: "ben@example.com",
  password: "BenSecret456"
});
try {
  await call<TaskBody>("GET", `/tasks/${report.id}`, ben.token);
} catch (err) {
  console.log(`Ben reads Sara's task: ${errorMessage(err)}`);
}
```

```bash
node src/scripts/try-api.ts
```

```text
registered Sara as user
created "Write the report": pending, due 2026-11-15T00:00:00.000Z
changed "Write the report" to in-progress
stats: { pending: 1, 'in-progress': 1, completed: 0, total: 2, overdue: 1 }
Ben reads Sara's task: 404 Task not found
```

"Pay the bills" was due on 1 October, so it counts as overdue. Make Ben an admin

```bash
node --env-file=.env src/scripts/make-admin.ts ben@example.com
```

```text
ben@example.com is now an admin
```

and with an unknown email or none at all

```text
No user with email nobody@example.com
```

```text
Usage: node --env-file=.env src/scripts/make-admin.ts <email>
```

The last one ends with exit code 1. For production: `npm run build` and `npm start`, as in Session 30.

---

## Mini Example 1: A New Task Status

The team wants an `archived` status. Change the one list

src/constants/taskStatuses.ts

```typescript
export const TASK_STATUSES = ["pending", "in-progress", "completed", "archived"] as const;
```

`TaskStatus` gets the new member, and so does everything built from the list: Mongoose's `enum`, Joi's `.valid(...)`, its error message, and the loop in `getStats`. One place does not follow by itself, and `tsc` points to it

```text
src/controllers/tasks.ts:31:9 - error TS2322: Type '{ pending: number; "in-progress": number; completed: number; total: number; overdue: number; }' is not assignable to type 'TaskStats'.
  Property 'archived' is missing in type '{ pending: number; "in-progress": number; completed: number; total: number; overdue: number; }' but required in type 'Record<"archived" | "completed" | "in-progress" | "pending", number>'.
```

Add `archived: 0` to the starting counts, and the statistics count archived tasks too. The JavaScript version filled its counts in a loop and needed no change there; here the object has the type `TaskStats`, so it must name every status from the start, and `tsc` says which one is missing. Then update the stats test's expected object, and decide with the team whether archived tasks belong in `overdue`.

![Adding archived to TASK_STATUSES updates TaskStatus, the Mongoose enum, the Joi rule and the stats loop; tsc points to the one object literal that must list every status](images/33-final-project/new-status.gif)

---

## Mini Example 2: What select false Hides

`password` has `select: false`, so a query leaves it out unless it asks with `.select("+password")`. Forget that in `login`

```typescript
  const user = await UserModel.findOne({ email });
```

`npm run check` passes: the type of a user document says `password: string`, always. Register Sara, then log in

```text
>>> POST /api/auth/login {"email":"sara@example.com","password":"MySecret123"}
Error: Illegal arguments: string, undefined
    at Object.compare (file:///D:/ts-practice/33-task-api/node_modules/bcryptjs/index.js:279:12)
    at model.comparePassword (file:///D:/ts-practice/33-task-api/src/models/user.ts:44:17)
500 application/json; charset=utf-8
{"success":false,"message":"something went wrong"}
```

`bcrypt.compare` got `undefined` instead of a hash. Mongoose's types describe the fields a document **can** have; `select` decides which fields a query really loads, and the types cannot follow that. The same is true for `req.user`: `protect` loads the user without the password, so `currentUser(req).password` is `undefined` although its type says `string`. That is why `changePassword` loads the user again, with `.select("+password")`.

A schema option that the types cannot see deserves a test. The login tests would have failed with this change.

![The type of a user document always has password string, but a query without select plus password loads no hash, and bcrypt.compare fails with Illegal arguments: string, undefined](images/33-final-project/select-false.gif)

---

## Where to Go Next

The parts of the Node.js project that this build left out, with what they need in TypeScript

| Feature                  | Node.js course          | Packages and their types                                   |
| ------------------------ | ----------------------- | ---------------------------------------------------------- |
| Profile pictures         | [Session 25](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/25-file-uploads.md) | `multer` with `@types/multer`, which adds `req.file` to Express's `Request` (like `req.user`) |
| Logging                  | [Session 26](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/26-logging.md) | `winston` (its own types), `morgan` with `@types/morgan`   |
| API documentation        | [Session 27](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/27-api-documentation.md) | `swagger-ui-express` and `swagger-jsdoc`, both with `@types/...` |
| Security headers, CORS, rate limits | [Session 14](https://github.com/mohin-sheikh/nodejs-learning-path/blob/main/14-middleware.md) | `helmet` and `express-rate-limit` (their own types), `cors` with `@types/cors` |

Each one is an exercise below. The way to add them is the way of this course: install the package (and its `@types` if it needs one), describe your own data with interfaces, check what comes from outside with Joi, and let `tsc`, ESLint and the tests tell you what you missed.

After this course

* The [Node.js Learning Path](https://github.com/mohin-sheikh/nodejs-learning-path) goes deeper into servers, MongoDB, authentication, uploads, logging and testing, in JavaScript. Everything there can be written the way this course writes it
* NestJS, a framework many companies use for TypeScript APIs, is built on the decorators you read in Session 22

![After the course: the Node.js Learning Path for more backend topics, NestJS for a framework built on decorators, and your own projects written the way this course writes them](images/33-final-project/next-steps.gif)

---

## Beginner Mistakes

### Mistake 1: A schema without the methods type argument

`new Schema<User>` alone makes `user.comparePassword` an error (TS2339) and the documents' types disagree everywhere. Give the schema all three: `new Schema<User, UserModelType, UserMethods>`.

---

### Mistake 2: Misspelling a method's name

`userSchema.method("comparePasword", ...)` passes `tsc`; the login fails with "user.comparePassword is not a function". The name is a plain string. Copy it from the interface, and keep a test that calls the method.

---

### Mistake 3: Forgetting .select("+password")

With `select: false`, a normal query has no password, but the type says it has. `bcrypt.compare` then throws "Illegal arguments: string, undefined". Ask for the field in every query that needs it.

---

### Mistake 4: Trusting the token's payload

`jwt.verify` checks the signature and gives back `string | JwtPayload`, whose properties are `any`. `payload.id` passes `tsc` even when it is missing. Check the payload with a schema, as `readToken` does.

---

### Mistake 5: Checking JsonWebTokenError before TokenExpiredError

The child class must come first (Session 21). The other order compiles and lints, and every expired token is reported as "Invalid token".

---

### Mistake 6: A route parameter with another name than the handler's

`route("/:taskId")` with a handler typed `RequestHandler<{ id: string }>` compiles, and `req.params.id` is `undefined`. Keep the names equal, and let the tests that read one task catch a difference.

---

### Mistake 7: A query without the owner

`TaskModel.findById(req.params.id)` compiles and returns anyone's task. Every task query includes `user: currentUser(req)._id`. `tsc` cannot know your access rules: the test "another user's task is not found" does.

---

### Mistake 8: Destructuring Joi's result before the check

`const { error, value } = schema.validate(...)` makes ESLint report an unsafe `any`. Check `result.error` first, then use `result.value`.

---

### Mistake 9: {} for "no query helpers"

Mongoose's documentation writes `Model<User, {}, UserMethods>`. ESLint's `no-empty-object-type` refuses `{}`; `object` means the same here and passes.

---

### Mistake 10: Tests that need your .env

Vitest does not read `.env`, and the config refuses to start without `MONGODB_URI` and `JWT_SECRET`. Give the tests their own values in `vitest.config.ts`, never your real secret.

---

## Practice Exercises

### Exercise 1

Add profile pictures: `PATCH /api/users/me/avatar` with multer and `@types/multer`. Add `avatar: string | null` to `User` and `PublicUser`. What type does `req.file` have, and why is it possibly `undefined`? Port the Node.js course's checks (size, extension, the real first bytes) and its test.

### Exercise 2

Add a login limiter with `express-rate-limit` (its own types): 10 tries in 15 minutes, answered with the API's error shape. Port the Node.js course's test "too many logins answer 429". Why must it be the last test in its file?

### Exercise 3

Add logging with winston: a `logger.ts` that reads a `LOG_LEVEL` setting (add it to `Env`, with a list of levels `as const`, as Session 28 did), and log failed logins with `logger.warn`. Keep the tests quiet when `NODE_ENV` is `test`.

### Exercise 4

Write unit tests for `toErrorAnswer` without a server: give it an `InvalidDataError`, an `AppError`, a `jwt.TokenExpiredError` and a plain `Error`, and check each answer. Then swap the two JWT checks and watch a unit test fail.

### Exercise 5

Add `?search=report` to `GET /api/tasks`: tasks whose title contains the text, ignoring case. Put it in `TaskQuery` and `taskQuerySchema`, and build the filter in `getTasks`. Write the test first.

### Exercise 6

Let admins deactivate a user: `PATCH /api/users/:id` with `{ "isActive": false }`, only for admins. Use `RequestHandler<{ id: string }>` and a Joi schema. Prove with a test that the user's token stops working at once.

### Exercise 7

Add the `archived` status of Mini Example 1 for real: follow `tsc`, update the tests, and make `overdue` ignore archived tasks. Which test fails before you change `getStats`?

### Exercise 8

Write `src/scripts/seed.ts`: it connects, deletes all users and tasks, registers two users through `UserModel.create` (the hook hashes their passwords), gives each three tasks with different statuses, and prints the counts. Run `try-api.ts` against the seeded database: what changes?

---

## Interview Questions

### How does the API know who is calling, and how is that typed?

The client sends `Authorization: Bearer <token>`. `protect` verifies the token, checks its payload with Joi, loads the user and sets `req.user`. `req.user` exists on Express's `Request` because of `declare global { namespace Express { interface Request { user?: UserDocument } } }`, and `currentUser(req)` turns the optional value into a `UserDocument`.

### Why check a JWT's payload after jwt.verify?

`jwt.verify` proves who signed the token, not what it contains. Its result is `string | JwtPayload`, and `JwtPayload` has an index signature with `any`. A schema turns it into a real `TokenPayload`, or rejects it.

### How do you type Mongoose instance methods?

Describe them in an interface (`UserMethods`), pass it as the third type argument of `Schema` and to the model's type (`Model<User, object, UserMethods>`), and add them with `schema.method("name", function ...)`. The method's `this` is then a typed document; its name is not checked.

### What does select: false mean for the types?

Nothing. The type of a document still has the field, but a query without `.select("+field")` does not load it, so the value is `undefined` while the code runs.

### Why does the order of instanceof checks matter in an error handler?

A child class is also an instance of its parent. `TokenExpiredError` extends `JsonWebTokenError`, so checking the parent first catches both and the child's branch never runs. `tsc` does not warn about it.

### How do the tests share types with the API?

They import the types the controllers use, like `AuthResult`, `TaskBody` and `PublicUser`, and read answers with `dataOf<T>()`. When an answer's type changes, the tests that use it stop compiling.

### Why do the controllers use RequestHandler<{ id: string }>?

A handler written on its own does not see the route text, so `req.params` would be the plain dictionary. The type argument says which parameters the handler needs. Express does not check it against the route, so the tests do.

### Why does the project have a separate tsconfig.build.json?

The build should contain only code that runs in production. It extends `tsconfig.json` and excludes the tests and their helpers, so `npm run build` neither compiles them nor fails on them, while `npm run check` still checks everything.

### What does Joi.string<StringValue>() promise?

The type argument tells `tsc` that the value is a `StringValue`, the type `jwt.sign` needs for `expiresIn`. The `.pattern(...)` rule makes that true while the program runs. Without the pattern, the type would be a lie.

### How do you make sure users only see their own tasks?

Every task query includes the owner: `{ _id: req.params.id, user: currentUser(req)._id }`. Someone else's task is then "not found", a 404, which does not even reveal that it exists. A test logs in as a second user and checks it.

### Why are the dates strings in TaskBody?

JSON has no date type: `res.json` turns a `Date` into ISO text. The answer type describes what the client receives, so `toTaskBody` calls `toISOString()` itself and the type says `string`.

### What would you add before a real launch?

The parts left out here: rate limits on login, security headers and CORS, logging with request ids, and API documentation. Also a real secret in the server's environment, HTTPS, database backups, and tests running on every push.

---

| Previous | All sessions | Next |
| :--- | :---: | ---: |
| [32. MongoDB and Mongoose with TypeScript](32-mongoose-with-typescript.md) | [Course outline](README.md#course-outline) | [Node.js Learning Path](https://github.com/mohin-sheikh/nodejs-learning-path) |
