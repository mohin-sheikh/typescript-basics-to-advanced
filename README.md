# TypeScript: Basics to Advanced

**A free, beginner-friendly TypeScript course for JavaScript developers.**
You know JavaScript. In 33 sessions you learn TypeScript the way companies use it: from your first type to generics, classes, validation, and a fully typed Node.js API. Everything runs with Node.js, and every example is tested.

[![Sessions](https://img.shields.io/badge/sessions-33-3178c6)](#course-outline)
[![TypeScript 7](https://img.shields.io/badge/TypeScript-7.0-3178c6?logo=typescript&logoColor=white)](#what-you-need)
[![Tested on Node.js 24](https://img.shields.io/badge/tested%20on-Node.js%2024-339933?logo=node.js&logoColor=white)](#what-you-need)
[![Published](https://img.shields.io/badge/published-25%20of%2033-f59e0b)](#course-outline)
[![Free](https://img.shields.io/badge/price-free-16a34a)](#how-to-use-this-course)

![The course in seven parts, from your first type to a typed Node.js API, followed by the Node.js course](images/01-introduction-to-typescript/learning-path.gif)

---

## Table of Contents

* [What Makes This Course Different](#what-makes-this-course-different)
* [Who This Course Is For](#who-this-course-is-for)
* [What You Need](#what-you-need)
* [Quick Start](#quick-start)
* [How to Use This Course](#how-to-use-this-course)
* [Course Outline](#course-outline)
* [Find a Topic Fast](#find-a-topic-fast)
* [What You Will Be Able to Do](#what-you-will-be-able-to-do)
* [Before and After This Course](#before-and-after-this-course)
* [Repository Structure](#repository-structure)
* [Feedback](#feedback)

---

## What Makes This Course Different

| | |
| --- | --- |
| **Built on TypeScript 7** | Most tutorials online use TypeScript 5. This course uses the newest version, 7.0, and tells you when an older tutorial shows something different. |
| **Backend from day one** | No web pages or browser code. You run everything with Node.js, with examples from real servers: users, orders, carts, logins and APIs. |
| **Every example is tested** | Every output in a session is real: the output of `node`, and the exact error messages of `tsc`, so you can compare them with yours. |
| **Learn by seeing** | Animated diagrams show what TypeScript sees in your code: types, checks, and what is left when the code runs. |
| **Learn from mistakes** | Every session has beginner mistakes, each with the wrong code, the real error (with its `TS` number) and the fix. |
| **Practice and interviews** | Practice exercises and interview questions, with answers, at the end of every session. |
| **One idea at a time** | A session only uses what earlier sessions taught. Nothing appears before it is explained. |
| **Part of a learning path** | It starts where the [JavaScript course](https://github.com/mohin-sheikh/javascript-basics-to-advanced) ends, and leads into the [Node.js course](https://github.com/mohin-sheikh/nodejs-learning-path). |

---

## Who This Course Is For

* JavaScript developers who want to switch to TypeScript
* Learners who finished the [JavaScript course](https://github.com/mohin-sheikh/javascript-basics-to-advanced) and want to go further
* Backend and full-stack developers whose company uses TypeScript
* Anyone preparing for a TypeScript or Node.js interview

You need to know basic JavaScript. You do not need any TypeScript experience.

---

## What You Need

| You need                   | Details                                                                 |
| -------------------------- | ----------------------------------------------------------------------- |
| Basic JavaScript           | Variables, functions, arrays, objects, classes, modules and promises. [Session 01](01-introduction-to-typescript.md#what-you-need-from-javascript) lists where each one is taught |
| Node.js 24 or newer        | The LTS version from [nodejs.org](https://nodejs.org/). Node.js 24 can run `.ts` files directly |
| A code editor              | [VS Code](https://code.visualstudio.com/) is recommended: it has TypeScript built in |
| A terminal                 | The one built into VS Code is enough                                    |
| Internet                   | Only to install the tools. Every example runs offline                   |

TypeScript itself is installed in each project with npm, in Session 02. The course was tested with TypeScript 7.0.2 and Node.js 24.

---

## Quick Start

**1. Check Node.js** in a terminal

```bash
node -v
```

```text
v24.19.0
```

Any version from 24 up works.

**2. Make a practice folder** called `ts-practice` and open it in VS Code (**File > Open Folder**).

**3. Open [Session 01](01-introduction-to-typescript.md)** and do [Try It Yourself](01-introduction-to-typescript.md#try-it-yourself). You need nothing else yet: Session 02 installs TypeScript and sets up the project you use for the rest of the course.

---

## How to Use This Course

**1. Go in order.** Every session builds on the ones before it. The navigation at the bottom of each session takes you to the next one.

**2. Type the code yourself.** Do not copy and paste. Typing is how you learn where the types go, and your own mistakes teach you to read error messages.

**3. Check and run.** Use `npx tsc` to check the types and `node` to run the code. Compare both outputs with the session.

**4. Read every error.** TypeScript's error messages are your teacher. Read the whole message before you change anything.

**5. Finish each session.** Read the beginner mistakes, do the practice exercises, and answer the interview questions out loud before you look at the answers.

You can read every session right here on GitHub, or download the course to read offline

```bash
git clone https://github.com/mohin-sheikh/typescript-basics-to-advanced.git
cd typescript-basics-to-advanced
```

### What is inside each session

| Part of a session         | What it gives you                                                       |
| ------------------------- | ----------------------------------------------------------------------- |
| Short sections            | One idea at a time, with JavaScript and TypeScript side by side where it helps |
| Tested code examples      | A file name, the code, and the real output of `node` and `tsc`          |
| Animated diagrams         | A picture of what TypeScript sees in your code                          |
| Mini examples             | One or two small backend programs that combine the session's topics     |
| Beginner mistakes         | The wrong code, the real error, and the correct code                    |
| Practice exercises        | Tasks to do on your own                                                 |
| Interview questions       | Short questions with clear answers, to check yourself                   |

### How long it takes

Plan for **1 to 2 hours per session**, including the exercises. At one session a day, most learners finish in **6 to 7 weeks**.

---

## Course Outline

Sessions are published one by one. Sessions with a link are ready to read.

### Part 1: Getting Started

Your first types, the tools, and how to read TypeScript's error messages.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 01 | [Introduction to TypeScript](01-introduction-to-typescript.md)  | What TypeScript is, why companies use it, compile time vs run time, type erasure |
| 02 | [Setup: Your First TypeScript Project](02-setup-and-first-project.md) | Installing TypeScript, `tsconfig.json`, checking with `tsc`, running with `node` |
| 03 | [Basic Types and Type Inference](03-basic-types-and-inference.md) | Type annotations, the primitive types, when TypeScript knows the type by itself |
| 04 | [any, unknown, never and void](04-any-unknown-never-void.md)   | The special types, and why `any` is dangerous                           |
| 05 | [Reading TypeScript Errors](05-reading-typescript-errors.md)    | The parts of an error, and the 20 errors you will see most              |

### Part 2: Typing Your Data

Describing the data your program works with.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 06 | [Arrays and Tuples](06-arrays-and-tuples.md)                    | Typed lists, `readonly` arrays, tuples, and items that may be missing   |
| 07 | [Object Types](07-object-types.md)                              | Object shapes, optional and readonly properties, structural typing      |
| 08 | [Type Aliases and Interfaces](08-type-aliases-and-interfaces.md) | Naming types, `type` vs `interface`, extending types                    |
| 09 | [Union and Literal Types](09-union-and-literal-types.md)         | Values of several types, and exact values like `"pending" \| "paid"`    |
| 10 | [Narrowing](10-narrowing.md)                                    | How TypeScript follows your `if` checks, discriminated unions           |
| 11 | [null, undefined and Optional Values](11-null-undefined-and-optional-values.md) | Strict null checks, `?.`, `??`, and values that may be missing          |

### Part 3: Functions

Typing the code that does the work.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 12 | [Typing Functions](12-typing-functions.md)                      | Parameters, return types, callbacks, overloads                          |
| 13 | [Type Assertions, as const and satisfies](13-assertions-as-const-and-satisfies.md) | Telling TypeScript what you know, and when that is dangerous            |

### Part 4: Generics and Type Tools

Reusable types, and types built from other types.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 14 | [Generics](14-generics.md)                                      | Functions and types that work with any type, safely                     |
| 15 | [keyof, typeof and Indexed Access](15-keyof-typeof-indexed-access.md) | Building types from values and from other types                         |
| 16 | [Enums and Constant Objects](16-enums-and-constant-objects.md)  | Enums, why Node.js cannot run them, and the modern alternative          |
| 17 | [Utility Types](17-utility-types.md)                            | `Partial`, `Pick`, `Omit`, `Record` and more, for request and response types |
| 18 | [Mapped and Conditional Types](18-mapped-and-conditional-types.md) | Building your own type tools, `infer`                                   |
| 19 | [Template Literal Types](19-template-literal-types.md)          | Types for strings like routes and event names                           |

### Part 5: Classes

Object-oriented TypeScript, as used in frameworks like NestJS.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 20 | [Classes in TypeScript](20-classes.md)                          | Typed fields, `private` vs `#private`, `readonly`, getters and `static` |
| 21 | [Inheritance, Interfaces and Abstract Classes](21-inheritance-interfaces-abstract-classes.md) | `extends`, `implements`, abstract classes, typed error classes          |
| 22 | [Decorators](22-decorators.md)                                  | Writing decorators, and reading the ones NestJS uses                    |

### Part 6: Real Programs

What every real TypeScript project needs.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 23 | [Async TypeScript and Error Handling](23-async-and-error-handling.md) | `Promise<T>`, typed `fetch` results, errors in `catch` are `unknown`    |
| 24 | [Modules in TypeScript](24-modules.md)                          | `import type`, ES modules and CommonJS, `.ts` imports                   |
| 25 | [Declaration Files and @types](25-declaration-files-and-types.md) | `.d.ts` files, `@types` packages, typing a JavaScript library           |
| 26 | Runtime Validation with Joi                                     | Checking outside data at run time, and keeping it in line with your types |
| 27 | tsconfig Deep Dive                                              | Every important setting, and what changed in TypeScript 6 and 7          |
| 28 | Migrating JavaScript to TypeScript                              | Moving a JavaScript project to TypeScript, step by step                 |
| 29 | Linting, Testing and Debugging                                  | ESLint, tests with Vitest, and debugging TypeScript                     |

### Part 7: TypeScript with Node.js

A typed backend from start to finish.

| #  | Session                                                         | You will learn                                                          |
| -- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| 30 | Node.js with TypeScript                                         | A Node.js project in TypeScript: run in development, build for production |
| 31 | Express with TypeScript                                         | Typed requests, responses and middleware, `req.user`                    |
| 32 | MongoDB and Mongoose with TypeScript                            | Typed schemas, models and queries                                       |
| 33 | Final Project: Task Management API in TypeScript                | A complete, typed and tested REST API                                   |

---

## Find a Topic Fast

Already know some TypeScript, or revising before an interview? Jump straight to a topic.

| Topic                                         | Session |
| --------------------------------------------- | ------- |
| What TypeScript is, and type erasure          | [01](01-introduction-to-typescript.md) |
| `tsconfig.json` and the `tsc` command         | [02](02-setup-and-first-project.md), 27 |
| `any` vs `unknown`                            | [04](04-any-unknown-never-void.md) |
| Reading error messages like `TS2322`          | [05](05-reading-typescript-errors.md) |
| Arrays, tuples and `readonly` arrays          | [06](06-arrays-and-tuples.md) |
| Object types, optional properties, structural typing | [07](07-object-types.md) |
| `type` vs `interface`                         | [08](08-type-aliases-and-interfaces.md) |
| Union types and narrowing                     | [09](09-union-and-literal-types.md), [10](10-narrowing.md) |
| Literal types and literal unions              | [09](09-union-and-literal-types.md) |
| Discriminated unions                          | [10](10-narrowing.md) |
| Type guards and exhaustive checks with `never` | [10](10-narrowing.md) |
| Strict null checks                            | [11](11-null-undefined-and-optional-values.md) |
| Optional chaining `?.` and `??` vs `\|\|`  | [11](11-null-undefined-and-optional-values.md) |
| Function types, callbacks and optional parameters | [12](12-typing-functions.md) |
| Function overloads                            | [12](12-typing-functions.md) |
| `as`, `as const` and `satisfies`              | [13](13-assertions-as-const-and-satisfies.md) |
| Assertion functions (`asserts value is T`)    | [13](13-assertions-as-const-and-satisfies.md) |
| Generics                                      | [14](14-generics.md) |
| Generic constraints (`T extends ...`)         | [14](14-generics.md) |
| `keyof` and `typeof`                          | [15](15-keyof-typeof-indexed-access.md) |
| Indexed access types like `Order["status"]`   | [15](15-keyof-typeof-indexed-access.md) |
| Enums                                         | [16](16-enums-and-constant-objects.md) |
| Constant objects with `as const` instead of enums | [16](16-enums-and-constant-objects.md) |
| `Partial`, `Pick`, `Omit` and `Record`        | [17](17-utility-types.md) |
| `Exclude`, `Extract`, `ReturnType` and `Parameters` | [17](17-utility-types.md) |
| Mapped and conditional types, `infer`         | [18](18-mapped-and-conditional-types.md) |
| Writing your own utility types                | [18](18-mapped-and-conditional-types.md) |
| Template literal types for routes and event names | [19](19-template-literal-types.md) |
| Class fields, getters, `static` and generic classes | [20](20-classes.md) |
| `private` vs `#private`, abstract classes     | [20](20-classes.md), [21](21-inheritance-interfaces-abstract-classes.md) |
| Inheritance, `implements` and typed error classes | [21](21-inheritance-interfaces-abstract-classes.md) |
| Decorators, and reading NestJS decorators     | [22](22-decorators.md) |
| Typing promises and errors                    | [23](23-async-and-error-handling.md) |
| `Awaited`, error causes and typed `fetch` results | [23](23-async-and-error-handling.md) |
| ES modules, barrel files, JSON imports and `import()` | [24](24-modules.md) |
| `.d.ts` files and `@types`                    | [25](25-declaration-files-and-types.md) |
| Typing a JavaScript library, `declare module` and `declare global` | [25](25-declaration-files-and-types.md) |
| Typing Express requests and Mongoose models   | 31, 32  |

---

## What You Will Be Able to Do

After this course you will be able to

* Read and write TypeScript the way professional teams do
* Set up a TypeScript project and understand every line of `tsconfig.json`
* Read TypeScript's error messages and fix the cause
* Describe your data with types, unions, generics and utility types
* Check data from outside the program at run time, with Joi
* Move a JavaScript project to TypeScript, step by step
* Build, test and run a typed Node.js API with Express and MongoDB
* Answer TypeScript questions in an interview

---

## Before and After This Course

| Course                                                                                   | What it teaches                                       |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| [JavaScript: Basics to Advanced](https://github.com/mohin-sheikh/javascript-basics-to-advanced) | The JavaScript language. Take it first if JavaScript is new to you |
| **TypeScript: Basics to Advanced** (this course)                                         | TypeScript, on top of the JavaScript you know         |
| [Node.js Learning Path](https://github.com/mohin-sheikh/nodejs-learning-path)            | Servers, REST APIs with Express, and MongoDB           |

All three courses use the same style, the same words and the same kind of examples (orders, users, carts), so moving between them is easy.

---

## Repository Structure

```text
typescript-basics-to-advanced/
├── 01-introduction-to-typescript.md    one file per session
├── 02-setup-and-first-project.md
├── 03-basic-types-and-inference.md
├── 04-any-unknown-never-void.md
├── 05-reading-typescript-errors.md
├── 06-arrays-and-tuples.md
├── 07-object-types.md
├── 08-type-aliases-and-interfaces.md
├── 09-union-and-literal-types.md
├── 10-narrowing.md
├── 11-null-undefined-and-optional-values.md
├── 12-typing-functions.md
├── 13-assertions-as-const-and-satisfies.md
├── 14-generics.md
├── 15-keyof-typeof-indexed-access.md
├── 16-enums-and-constant-objects.md
├── 17-utility-types.md
├── 18-mapped-and-conditional-types.md
├── 19-template-literal-types.md
├── 20-classes.md
├── 21-inheritance-interfaces-abstract-classes.md
├── 22-decorators.md
├── 23-async-and-error-handling.md
├── 24-modules.md
├── 25-declaration-files-and-types.md
├── ...
├── images/                             animated diagrams, one folder per session
└── README.md
```

---

## Feedback

Found a mistake, or a part that is hard to understand? Please [open an issue](https://github.com/mohin-sheikh/typescript-basics-to-advanced/issues) or send a pull request. Every report helps the next learner.

If this course helps you, give it a star on GitHub so others can find it too.

---

**Ready? Start with [Session 01: Introduction to TypeScript](01-introduction-to-typescript.md).**
