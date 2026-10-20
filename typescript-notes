# TypeScript Notes

## Table of Contents

1. [Index Signatures](#1-index-signatures)
2. [Structural vs Nominal Types](#2-structural-vs-nominal-types)
3. [Union & Intersection](#3-union--intersection)
4. [Interfaces vs Type Aliases](#4-interfaces-vs-type-aliases)
5. [Top & Bottom Types](#5-top--bottom-types)
6. [Unit Types](#6-unit-types)
7. [Nullish Values](#7-nullish-values)
8. [Modules](#8-modules)
9. [Conditional Types](#9-conditional-types)
10. [Quick Review Questions](#10-quick-review-questions)

---

## 1. Index Signatures

Used for **dictionary-like objects**: values of a consistent type, looked up by keys.

```ts
const phones: {
  [key: string]: { country: string; area: string; number: string }
} = {}

phones.home   // OK
phones.mobile // OK – any string key is allowed
```

- Equivalent shorthand: `Record<string, Phone>`.
- With `noUncheckedIndexedAccess: true`, lookups return `Phone | undefined` (safer, since the key might not exist).

---

## 2. Structural vs Nominal Types

Type-checking is about answering one question: **"Can this type be assigned to that type?"** (type compatibility / equivalence).

TypeScript's type system is **static** — checks happen at compile time, not runtime.

| System | Checks | Example languages |
|---|---|---|
| **Nominal** | The *name* — "Are you an instance of this class?" | Java, C# |
| **Structural** | The *shape* — "Do you have the right properties of the right types?" | **TypeScript** |
| **Duck typing** | Shape too, but checked at *runtime* | JavaScript, Python |

> **Duck test:** "If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck."
> Duck typing ≈ structural typing, but the term is usually used for **dynamic** type systems.

```ts
class Car { make = "Honda" }
class Truck { make = "Ford" }

let vehicle: Car = new Truck() // ✅ OK in TS – same shape, different name
```

---

## 3. Union & Intersection

| Operator | Name | Meaning |
|---|---|---|
| `A \| B` | **Union** | Value is A **OR** B |
| `A & B` | **Intersection** | Value is A **AND** B (has everything from both) |

```ts
type Id = string | number
type Timestamped<T> = T & { createdAt: Date }
```

---

## 4. Interfaces vs Type Aliases

### Interfaces
- Describe **object shapes** (and classes).
- Inherit with `extends`.
- **Cannot** describe unions, tuples, or primitives.
- **Open** — can be redeclared; declarations **merge** (declaration merging).
  - Useful for augmenting globals or 3rd-party library types (e.g. adding a property to `Window`).
  - When merging you can only **add** properties, not override existing ones with incompatible types.

```ts
interface Window { myGlobal: string }   // merges into the built-in Window
```

### Type aliases
- Can describe **anything**: unions, tuples, primitives, functions, objects.
- **Closed** — redeclaring the same name is an error.

```ts
// Only possible with a type alias:
type UserInfoOutcomeError = readonly ["error", Error]
type Outcome = UserInfoOutcomeSuccess | UserInfoOutcomeError
```

**Rule of thumb:** use `interface` for object shapes you might want to extend/augment; use `type` for unions, tuples, and everything else.

---

## 5. Top & Bottom Types

Think of types as **sets of values**.
- **Top type** = the set of *all* possible values.
- **Bottom type** = the *empty* set (no possible value).

### Top types: `any` and `unknown`

| | `any` | `unknown` |
|---|---|---|
| Accepts any value? | ✅ | ✅ |
| Use without checks? | ✅ (access any property, call it, etc. — TS won't complain even if it doesn't exist) | ❌ must **narrow** first |
| Safety | Turns off type checking | Safe ("opaque") |

```ts
let a: any = 4
a.foo.bar.baz() // compiles, crashes at runtime

let u: unknown = 4
u.foo           // ❌ error
if (typeof u === "number") {
  u.toFixed(2)  // ✅ narrowed with a type guard
}
```

**Good uses of `unknown`:**
- Values received at runtime (API responses, data layer) — forces consumers to validate first, so errors surface earlier and with more context.
- Errors in `catch` blocks:

```ts
try {
  somethingRisky()
} catch (err: unknown) {
  if (err instanceof Error) console.log(err.message)
}
```

- By default `catch (err)` gives `err` the type `any` (common when migrating JS → TS).
- `useUnknownInCatchVariables: true` (enabled by `strict`) makes `err` **`unknown`** by default.

### Almost-top type: `object`

- All values **except primitives**.
- The 7 primitives: `string`, `number`, `boolean`, `bigint`, `symbol`, `null`, `undefined`.
- With `strictNullChecks`, `null` and `undefined` can't be assigned to `object`.

### Almost-top type: `{}` (empty object)

- All values **except `null` and `undefined`**.
- So `string`, `number`, `Date`, etc. are all assignable to `{}`.

**Trick:** intersect with `{}` to strip nullability:

```ts
type NullableStringOrNumber = string | number | null | undefined
type StringOrNumber = NullableStringOrNumber & {} // string | number
```

(This is how the built-in `NonNullable<T>` is implemented.)

### Bottom type: `never`

- Represents the empty set `{ }` — **no value can have this type**.
- Means "this should be unreachable" — no options are left.
- Great for **exhaustiveness checks**:

```ts
class UnreachableError extends Error {
  constructor(_nvr: never, message: string) {
    super(message)
  }
}

function handle(x: "a" | "b") {
  if (x === "a") return 1
  if (x === "b") return 2
  throw new UnreachableError(x, `Unhandled: ${x}`) // x is `never` here
}
// If someone adds "c" to the union, this line becomes a compile error.
```

---

## 6. Unit Types

Types whose set contains **exactly one value**.

- Literal types: `"hello"`, `42`, `true`
- `null` and `undefined` are both unit types — and they are **not interchangeable**.

`void` is *almost* a unit type: it's used for function returns, and `undefined` is assignable to it.

---

## 7. Nullish Values

`null`, `undefined`, and `void` all mean "nothing", but they are **separate types** with different meanings:

| Type | Meaning |
|---|---|
| `null` | There **is** a value, and it's intentionally nothing. A *presence* of information. |
| `undefined` | The value **isn't available** (yet?). An *absence*. |
| `void` | A function's return value **should be ignored**. Use only for function return types. |

### Non-null assertion `!`

Tells TS: "trust me, this isn't `null`/`undefined`."

```ts
cart.fruits!.push(...)
```

⚠️ It only silences the compiler. If the value *is* missing, you still get a runtime error like `Cannot read properties of undefined`. Use sparingly.

### Definite assignment assertion `!:`

Tells TS a class field **will** be initialized, even if TS can't prove it (e.g. set in an `init()` method rather than the constructor).

```ts
class ThingWithAsyncSetup {
  isSetup!: boolean // without `!`, TS complains it may be used before being initialized

  constructor() {
    this.doSetup()
  }
  private async doSetup() {
    this.isSetup = true
  }
}
```

### Optional chaining `?.`

A safer alternative to `!`. Stops and returns `undefined` as soon as something in the chain is `null`/`undefined` — no tedious nested `if` checks.

```ts
function getLastPayment(data: ResponseData): number | undefined {
  return data?.customer?.lastInvoice?.lastPayment?.amount
}
```

For older targets, this compiles to a chain of `=== null || === void 0` checks (`void 0` is just `undefined`).

### Nullish coalescing `??`

Provides a fallback **only when the value is `null` or `undefined`**.

```ts
const count = 0
count || 10 // 10  ❌ – `||` treats any falsy value (0, "", false) as missing
count ?? 10 // 0   ✅ – `??` only falls back on null/undefined
```

---

## 8. Modules

### History

No official JS module spec until **ES2015**. Before that, community formats appeared:
- **CommonJS** — still the default for Node.js
- **AMD** (RequireJS) — rare today
- **UMD** — rare today

### ES module syntax (what TS uses)

```ts
// imports
import { Blueberry, Raspberry } from "./berries" // named imports
import Kiwi from "./kiwi"                        // default import

// exports
export function makeFruitSalad() {}              // named export
export default class FruitBasket {}              // default export

// re-exports
export { lemon, lime } from "./citrus"           // re-export specific items
export * as berries from "./berries"             // re-export whole module as a namespace
```

### CommonJS interop

CommonJS (what Node understands natively):

```js
module.exports = { MyClass } // export
const fs = require("fs")     // import
fs.readFileSync("example.txt")
```

The TS / ES-module equivalent:

```ts
// namespace import
import * as fs from "fs"
fs.readFileSync("example.txt")

// or named import
import { readFileSync } from "fs"
readFileSync("example.txt")
```

### Native ES modules in Node

By file extension:

| JS | TS | Treated as |
|---|---|---|
| `.mjs` | `.mts` | ES module |
| `.cjs` | `.cts` | CommonJS |
| `.js` | `.ts` | Depends on `package.json` `"type"` |

`package.json` `"type"` field:
- `"module"` → `.js` files are ES modules
- `"commonjs"` → `.js` files are CommonJS
- **No `"type"` field → Node assumes CommonJS** (still true as of Node 20.8+)

### Importing non-TS files (e.g. images)

Declare a module type for the file pattern:

```ts
// global.d.ts
declare module "*.png" {
  const imgUrl: string
  export default imgUrl
}

// component.ts
import img from "./file.png" // img: string
```

---

## 9. Conditional Types

Ternary logic, but for types:

```ts
// value level
const isXNegative = x >= 0 ? "no" : "yes"

// type level
type CookingDevice<T> = T extends "grill" ? Grill : Oven
```

```ts
class Grill {
  startGas() {}
  stopGas() {}
}
class Oven {
  setTemperature(degrees: number) {}
}

let device1: CookingDevice<"grill"> // Grill
let device2: CookingDevice<"oven">  // Oven
```

`T extends U` means **"is T assignable to U?"**

### Distribution over unions

If you pass a **union** as `T`, the condition is evaluated **for each member individually**, and the results are unioned back together:

```ts
type R = CookingDevice<"grill" | "oven"> // Grill | Oven
```

### Built-in utilities: `Extract` & `Exclude`

```ts
type Exclude<T, U> = T extends U ? never : T // remove members assignable to U
type Extract<T, U> = T extends U ? T : never // keep only members assignable to U
```

They rely on distribution + `never` disappearing from unions.

```ts
type FavoriteColors =
  | "dark sienna"
  | "van dyke brown"
  | "yellow ochre"
  | [number, number, number]
  | { red: number; green: number; blue: number }

type StringColors = Extract<FavoriteColors, string>
// "dark sienna" | "van dyke brown" | "yellow ochre"

type ObjectColors = Extract<FavoriteColors, { red: number }>
// { red: number; green: number; blue: number }

type TupleColors = Extract<FavoriteColors, [number, number, number]>
// [number, number, number]

type NonStringColors = Exclude<FavoriteColors, string>
// [number, number, number] | { red: number; green: number; blue: number }
```

### Inference with `infer`

`infer` lets you **capture part of a type** into a new type parameter.

- Only usable inside the **condition** of a conditional type (`... extends ... infer X ...`).

```ts
type UnwrapPromise<P> = P extends PromiseLike<infer T> ? T : P

type A = UnwrapPromise<Promise<string>> // string
type B = UnwrapPromise<number>          // number (not a promise, returned as-is)
```

Other examples of the same idea (these exist as built-ins):

```ts
type ReturnType<F> = F extends (...args: any[]) => infer R ? R : never
type Parameters<F> = F extends (...args: infer P) => any ? P : never
```

---

## 10. Quick Review Questions

1. Is TypeScript nominally or structurally typed? What does that mean in practice?
2. Name two things a `type` alias can do that an `interface` can't.
3. What happens when you declare the same `interface` twice?
4. What's the difference between `any` and `unknown`?
5. What values does `{}` accept? What about `object`?
6. How do you remove `null | undefined` from a type using `{}`?
7. When would a value have the type `never`?
8. `null` vs `undefined` vs `void` — when do you use each?
9. What's the risk of the `!` non-null assertion?
10. `0 || 5` vs `0 ?? 5` — what's each result?
11. Without a `"type"` field in `package.json`, how does Node treat `.js` files?
12. What does `Exclude<"a" | "b" | 1, string>` produce?
13. Write a type that unwraps the element type of an array using `infer`.

<details>
<summary>Answers</summary>

1. Structurally — compatibility is based on shape (properties), not names.
2. Unions, tuples (also primitives, mapped/conditional types).
3. The declarations merge into one interface.
4. Both accept anything; `unknown` must be narrowed before use, `any` disables checking.
5. `{}`: everything except `null`/`undefined`. `object`: everything except primitives.
6. `T & {}`
7. When no possible value is left — e.g. after exhaustively narrowing a union.
8. `null`: intentional empty value. `undefined`: not available (yet). `void`: function return to ignore.
9. It only silences TS; if the value is actually missing you crash at runtime.
10. `5` and `0`.
11. As CommonJS.
12. `1`
13. `type ElementOf<T> = T extends (infer E)[] ? E : never`

</details>
