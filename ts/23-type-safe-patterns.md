# Bo'lim 23: Type-Safe Patterns

> Type-safe patterns — TypeScript ning type system ini to'liq ishlatib, **compile-time** da xatolarni ushlaydigan arxitektura pattern lar. Bu bo'limda branded/opaque types (nominal typing), exhaustive pattern matching, `const` assertions, type-safe builder va event emitter, runtime validation library lar (Zod, Valibot, io-ts, ArkType), va Result type orqali error handling o'rganiladi.

---

## Mundarija

- [Branded/Opaque Types — Nominal Typing](#brandedopaque-types--nominal-typing)
- [Exhaustive Pattern Matching](#exhaustive-pattern-matching)
- [`const` Assertions](#const-assertions)
- [Type-Safe Builders](#type-safe-builders)
- [Type-Safe Event Emitters](#type-safe-event-emitters)
- [Runtime Validation + Types](#runtime-validation--types)
- [Error Handling Patterns](#error-handling-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Branded/Opaque Types — Nominal Typing

### Nazariya

TypeScript **structural typing** ishlatadi — ikki type ning "shakli" bir xil bo'lsa, ular compatible. Branded types bu cheklovni "sindirib", **nominal typing** beradi. Asosiy g'oya: base type ga phantom property qo'shib, ikki struktural bir xil type ni compile-time da farqlash.

Batafsil [Bo'lim 16: Brand<T, B>](16-custom-utility-types.md#brandt-b--brandednominal-types) va [Bo'lim 21: Design Patterns](21-design-patterns.md).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
type Email = Brand<string, "Email">;

function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("Invalid ID");
  return id as UserId;
}

function createEmail(value: string): Email {
  if (!value.includes("@")) throw new Error("Invalid email");
  return value as Email;
}

function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

const userId = createUserId(1);
const postId = createPostId(1);

getUser(userId); // ✅
getUser(postId); // ❌ — PostId ≠ UserId
getUser(42);     // ❌ — number ≠ UserId

// Zod bilan:
import { z } from "zod";
const UserIdSchema = z.number().positive().brand<"UserId">();
type ZodUserId = z.infer<typeof UserIdSchema>;
```

</details>

---

## Exhaustive Pattern Matching

### Nazariya

Exhaustive matching — union type ning **barcha variant** larini handle qilganingizni compile-time da kafolatlash. `switch` + `never` pattern — yangi variant qo'shilganda TS error beradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function assertNever(x: never): never {
  throw new Error(`Unexpected: ${JSON.stringify(x)}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.side ** 2;
    case "triangle": return (shape.base * shape.height) / 2;
    default: return assertNever(shape);
    // Agar yangi shape qo'shilsa (masalan "pentagon") —
    // bu yerda compile error bo'ladi: pentagon never ga assign bo'lmaydi
  }
}

// === Record map bilan exhaustive ===
const areaCalculators: Record<Shape["kind"], (s: any) => number> = {
  circle: (s) => Math.PI * s.radius ** 2,
  square: (s) => s.side ** 2,
  triangle: (s) => (s.base * s.height) / 2,
  // Yangi kind qo'shilsa — bu yerda error (property missing)
};

// === satisfies bilan (TS 4.9+) ===
const handlers = {
  circle: (s: Extract<Shape, { kind: "circle" }>) => Math.PI * s.radius ** 2,
  square: (s: Extract<Shape, { kind: "square" }>) => s.side ** 2,
  triangle: (s: Extract<Shape, { kind: "triangle" }>) => (s.base * s.height) / 2,
} satisfies Record<Shape["kind"], (s: any) => number>;
```

</details>

---

## `const` Assertions

### Nazariya

`as const` — object va array larni **deeply readonly** va **literal types** bilan qiladi. Bu enum alternative sifatida zero runtime cost bilan ishlaydi.

`satisfies` (TS 4.9+) + `as const` — type tekshiruv + narrow qiymat birga.

`const` type parameter (TS 5.0+) — generic da avtomatik `as const` narrowing.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === as const — enum alternative ===
const HttpStatus = {
  OK: 200,
  NotFound: 404,
  ServerError: 500,
} as const;

type HttpStatus = (typeof HttpStatus)[keyof typeof HttpStatus];
// 200 | 404 | 500

// === satisfies + as const ===
const Routes = {
  home: "/",
  about: "/about",
  user: "/user/:id",
} as const satisfies Record<string, string>;
// Type checking (Record) + narrow values ("/", "/about", "/user/:id")

// === const type parameter (TS 5.0+) ===
function createConfig<const T extends Record<string, unknown>>(config: T): T {
  return config;
}

const cfg = createConfig({ host: "localhost", port: 3000 });
// cfg: { readonly host: "localhost"; readonly port: 3000 }
// const keyword siz: { host: string; port: number }
```

</details>

---

## Type-Safe Builders

### Nazariya

Type-safe builder — compile-time da required field lar to'ldirilganini kafolatlaydi. Ikki approach: **step builder** (har step alohida interface) va **phantom type builder** (generic state tracking).

Batafsil [Bo'lim 21: Builder Pattern](21-design-patterns.md#creational-builder-pattern).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Phantom type builder ===
type RequiredKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? never : K;
}[keyof T];

class TypeSafeBuilder<T, Provided extends keyof T = never> {
  private data: Partial<T> = {};

  set<K extends keyof T>(key: K, value: T[K]): TypeSafeBuilder<T, Provided | K> {
    (this.data as Record<string, unknown>)[key as string] = value;
    return this as unknown as TypeSafeBuilder<T, Provided | K>;
  }

  build(
    this: RequiredKeys<T> extends Provided ? TypeSafeBuilder<T, Provided> : never
  ): T {
    return { ...this.data } as T;
  }
}

interface DbConfig {
  host: string;
  port: number;
  database: string;
  username?: string;
  ssl?: boolean;
}

const cfg = new TypeSafeBuilder<DbConfig>()
  .set("host", "localhost")
  .set("port", 5432)
  .set("database", "mydb")
  .build(); // ✅ — barcha required lar berildi

// new TypeSafeBuilder<DbConfig>().set("host", "x").build();
// ❌ — port va database kerak
```

</details>

---

## Type-Safe Event Emitters

### Nazariya

Generic event map bilan type-safe event emitter — event name va payload compile-time da tekshiriladi. [Bo'lim 21: Observer Pattern](21-design-patterns.md#behavioral-observer-pattern) da batafsil.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type EventMap = Record<string, (...args: any[]) => void>;

class TypedEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<(...args: any[]) => void>>();

  on<K extends keyof T>(event: K, listener: T[K]): this {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener);
    return this;
  }

  emit<K extends keyof T>(event: K, ...args: Parameters<T[K]>): void {
    this.listeners.get(event)?.forEach(fn => fn(...args));
  }
}

interface AppEvents {
  userLoggedIn: (userId: string, timestamp: Date) => void;
  orderPlaced: (orderId: string, total: number) => void;
}

const emitter = new TypedEmitter<AppEvents>();
emitter.on("userLoggedIn", (userId, ts) => {
  // userId: string, ts: Date — ✅ type-safe
});
emitter.emit("userLoggedIn", "user-1", new Date()); // ✅
// emitter.emit("userLoggedIn", 42, "wrong");        // ❌
```

</details>

---

## Runtime Validation + Types

### Nazariya

Type-safe pattern lar faqat compile-time. External data (API, user input) uchun **runtime validation** kerak. Zamonaviy library lar schema dan type ni avtomatik chiqaradi — **single source of truth**.

| Library | Xususiyat | Bundle size |
|---------|-----------|------------|
| **Zod** | Eng ommabop, chaining API | ~14kb |
| **Valibot** | Tree-shakeable, modular | ~1-2kb (ishlatilganga qarab) |
| **io-ts** | Functional (fp-ts bilan) | ~6kb |
| **ArkType** | TS syntax ga yaqin, tez | ~25kb |

<details>
<summary><strong>Kod Misollari</strong></summary>

**Zod:**

```typescript
import { z } from "zod";

const UserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().min(18),
  role: z.enum(["admin", "user"]),
});

type User = z.infer<typeof UserSchema>;
// { name: string; email: string; age: number; role: "admin" | "user" }

const result = UserSchema.safeParse(apiResponse);
if (result.success) {
  const user: User = result.data; // ✅ type-safe
} else {
  console.error(result.error.issues);
}
```

**Valibot (tree-shakeable):**

```typescript
import * as v from "valibot";

const UserSchema = v.object({
  name: v.pipe(v.string(), v.minLength(2)),
  email: v.pipe(v.string(), v.email()),
  age: v.pipe(v.number(), v.minValue(18)),
});

type User = v.InferOutput<typeof UserSchema>;
```

**io-ts (functional):**

```typescript
import * as t from "io-ts";

const User = t.type({
  name: t.string,
  email: t.string,
  age: t.number,
});

type User = t.TypeOf<typeof User>;

const result = User.decode(apiResponse);
// Either<Errors, User>
```

</details>

---

## Error Handling Patterns

### Nazariya

`throw` o'rniga **Result type** — caller xatoni handle qilishga **compile-time da majbur bo'ladi**. Bu "Railway-oriented programming" — Ok rail va Error rail.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> { return { ok: true, value }; }
function err<E>(error: E): Result<never, E> { return { ok: false, error }; }

function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return r.ok ? ok(fn(r.value)) : r;
}

function flatMap<T, U, E>(r: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return r.ok ? fn(r.value) : r;
}

function match<T, E, U>(r: Result<T, E>, h: { ok: (v: T) => U; err: (e: E) => U }): U {
  return r.ok ? h.ok(r.value) : h.err(r.error);
}

// Pipeline:
function validateAge(age: number): Result<number, string> {
  return age >= 18 ? ok(age) : err("Must be 18+");
}

function createUser(name: string, age: number): Result<{ name: string; age: number }, string> {
  if (!name) return err("Name required");
  return ok({ name, age });
}

const result = flatMap(validateAge(25), (age) => createUser("Ali", age));
match(result, {
  ok: (user) => console.log(`Created: ${user.name}`),
  err: (e) => console.error(`Error: ${e}`),
});
```

**neverthrow — production-ready Result:**

```typescript
import { ok, err, Result, ResultAsync } from "neverthrow";

function divide(a: number, b: number): Result<number, string> {
  return b === 0 ? err("Division by zero") : ok(a / b);
}

divide(10, 2)
  .map(n => n * 2)
  .match(
    (val) => console.log(`Result: ${val}`),
    (error) => console.error(error),
  );
```

</details>

---

## Edge Cases va Gotchas

### 1. Brand arithmetic da yo'qoladi

```typescript
type USD = Brand<number, "USD">;
const price = 10 as USD;
const doubled = price * 2;
// doubled: number — brand YO'QOLDI!
// + - * / natijasi har doim number
```

### 2. `as const` va mutable operation lar

```typescript
const arr = [1, 2, 3] as const;
// arr: readonly [1, 2, 3]
// arr.push(4);    // ❌ — readonly
// arr[0] = 10;    // ❌ — readonly

// Lekin reference orqali bypass mumkin:
const mutable: number[] = [...arr]; // ✅ — spread bilan yangi array
mutable.push(4); // ✅
```

### 3. Exhaustive check — `default` branch da `never` ishlashni to'xtatadi

```typescript
type Color = "red" | "green" | "blue";

function toHex(color: Color): string {
  switch (color) {
    case "red": return "#ff0000";
    case "green": return "#00ff00";
    // "blue" UNUTILDI
    default: return assertNever(color);
    // ❌ Compile error: 'string' is not assignable to 'never'
    // Bu YAXSHI — xato compile-time da topildi
  }
}
```

### 4. Zod `.transform()` — infer type transform dan keyingi type bo'ladi

```typescript
const Schema = z.object({
  age: z.string().transform(Number),
});

type Input = z.input<typeof Schema>;
// { age: string } — parse dan oldingi type

type Output = z.infer<typeof Schema>;
// { age: number } — parse dan keyingi type (transform natijasi)
```

### 5. Result type va `throw` aralashtirganda type safety yo'qoladi

```typescript
function riskyFn(): Result<string, Error> {
  JSON.parse("invalid"); // ❌ Bu THROW qiladi!
  // Result pattern ichida throw — catch qilinmaydi
  // Result va throw — birga ishlatmang, birini tanlang
  return ok("done");
}

// ✅ — try/catch bilan Result ga o'rash
function safeParse(json: string): Result<unknown, Error> {
  try { return ok(JSON.parse(json)); }
  catch (e) { return err(e instanceof Error ? e : new Error(String(e))); }
}
```

---

## Common Mistakes

### ❌ Xato 1: Branded type ni `as` siz yaratish

```typescript
type UserId = Brand<number, "UserId">;
const id: UserId = 42; // ❌ — number UserId ga assign bo'lmaydi

// ✅ — constructor function orqali
const id = createUserId(42); // runtime validation + brand
```

### ❌ Xato 2: Exhaustive check siz switch

```typescript
// ❌ — yangi variant qo'shilganda hech qanday error yo'q
function handle(status: Status) {
  switch (status) {
    case "active": return "Active";
    case "inactive": return "Inactive";
    // Yangi "suspended" qo'shilsa — jim o'tib ketadi
  }
}

// ✅ — assertNever bilan
default: return assertNever(status);
```

### ❌ Xato 3: `as const` va `satisfies` ni aralashtirmaslik

```typescript
// ❌ — faqat as const: type tekshiruv yo'q
const routes = { home: "/", about: "/about" } as const;
// Typo tutilmaydi — { hme: "/" } ham o'tib ketadi

// ✅ — satisfies + as const
const routes = { home: "/", about: "/about" } as const satisfies Record<string, string>;
// Type tekshiruv BOR + narrow values saqlanadi
```

### ❌ Xato 4: Zod schema va TS type alohida yozish

```typescript
// ❌ — ikki joyda maintain qilish kerak
interface User { name: string; age: number; }
const UserSchema = z.object({ name: z.string(), age: z.number() });
// User va UserSchema sync dan chiqishi mumkin

// ✅ — single source of truth
const UserSchema = z.object({ name: z.string(), age: z.number() });
type User = z.infer<typeof UserSchema>; // Schema dan chiqariladi
```

### ❌ Xato 5: Result type da error union ni handle qilmaslik

```typescript
// ❌ — flatMap da error type lar birlashadi, lekin match da barchasi handle bo'lmasa
function process(): Result<string, "NotFound" | "Forbidden" | "ServerError"> { /* ... */ }

match(process(), {
  ok: (v) => console.log(v),
  err: (e) => console.log(e), // e: "NotFound" | "Forbidden" | "ServerError"
  // Har bir error alohida handle qilish yaxshiroq
});
```

---

## Amaliy Mashqlar

### Mashq 1: Exhaustive Config (Oson)

**Savol:** `Theme` type ning barcha variant lari uchun CSS config yarating. Yangi theme qo'shilganda compile error berilsin.

<details>
<summary>Javob</summary>

```typescript
type Theme = "light" | "dark" | "system";

const themeConfig: Record<Theme, { bg: string; fg: string }> = {
  light: { bg: "#fff", fg: "#000" },
  dark: { bg: "#000", fg: "#fff" },
  system: { bg: "auto", fg: "auto" },
};
// Yangi theme qo'shilsa — bu yerda compile error
```

</details>

---

### Mashq 2: Branded Types System (O'rta)

**Savol:** `UserId`, `PostId`, `Email` branded types + constructor functions + type-safe API.

<details>
<summary>Javob</summary>

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
type Email = Brand<string, "Email">;

function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("Invalid UserId");
  return id as UserId;
}
function createEmail(v: string): Email {
  if (!v.includes("@")) throw new Error("Invalid email");
  return v as Email;
}

function getUser(id: UserId): void { /* ... */ }
function sendEmail(to: Email): void { /* ... */ }

getUser(createUserId(1)); // ✅
sendEmail(createEmail("ali@test.com")); // ✅
// getUser(createPostId(1)); // ❌
```

</details>

---

### Mashq 3: Zod Schema (O'rta)

**Savol:** User registration form uchun Zod schema yozing — `name` (2+ char), `email`, `age` (18+), `role` enum.

<details>
<summary>Javob</summary>

```typescript
import { z } from "zod";

const RegisterSchema = z.object({
  name: z.string().min(2, "Name too short"),
  email: z.string().email("Invalid email"),
  age: z.number().min(18, "Must be 18+"),
  role: z.enum(["admin", "user", "moderator"]),
});

type RegisterInput = z.infer<typeof RegisterSchema>;

function register(input: unknown): RegisterInput {
  return RegisterSchema.parse(input);
}
```

</details>

---

### Mashq 4: Result Pipeline (Qiyin)

**Savol:** `Result<T, E>`, `map`, `flatMap`, `match` yozing. User registration pipeline yarating.

<details>
<summary>Javob</summary>

```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
function ok<T>(v: T): Result<T, never> { return { ok: true, value: v }; }
function err<E>(e: E): Result<never, E> { return { ok: false, error: e }; }
function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return r.ok ? ok(fn(r.value)) : r;
}
function flatMap<T, U, E>(r: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return r.ok ? fn(r.value) : r;
}
function match<T, E, U>(r: Result<T, E>, h: { ok: (v: T) => U; err: (e: E) => U }): U {
  return r.ok ? h.ok(r.value) : h.err(r.error);
}

// Pipeline:
const result = flatMap(
  flatMap(
    validateInput({ name: "Ali", email: "ali@test.com", age: 25 }),
    validateEmail
  ),
  createUser
);
```

</details>

---

### Mashq 5: Phantom Type Builder (Qiyin)

**Savol:** `TypeSafeBuilder<T>` — `build()` faqat barcha required field lar `.set()` qilinganda.

<details>
<summary>Javob</summary>

```typescript
type RequiredKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? never : K;
}[keyof T];

class TypeSafeBuilder<T, Provided extends keyof T = never> {
  private data: Partial<T> = {};

  set<K extends keyof T>(key: K, value: T[K]): TypeSafeBuilder<T, Provided | K> {
    (this.data as Record<string, unknown>)[key as string] = value;
    return this as unknown as TypeSafeBuilder<T, Provided | K>;
  }

  build(this: RequiredKeys<T> extends Provided ? TypeSafeBuilder<T, Provided> : never): T {
    return { ...this.data } as T;
  }
}
```

</details>

---

## Xulosa

Bu bo'limda type-safe production pattern larni o'rgandik:

- **Branded types** — nominal typing, `UserId` ≠ `PostId` compile-time da
- **Exhaustive matching** — `switch` + `never`, yangi variant → compile error
- **`as const`** — literal types + readonly, enum alternative
- **`satisfies`** (TS 4.9+) — type checking + narrow value birga
- **Type-safe builders** — step builder, phantom type builder
- **Runtime validation** — Zod, Valibot, io-ts, ArkType — schema → type
- **Result type** — `Ok<T> | Err<E>`, error handling compile-time enforcement

**Bog'liq bo'limlar:**
- [Bo'lim 5: Union/Intersection](05-unions-intersections.md) — discriminated unions
- [Bo'lim 16: Custom Utility Types](16-custom-utility-types.md) — Brand, PathKeys
- [Bo'lim 21: Design Patterns](21-design-patterns.md) — Builder, Observer, Result

---

**Keyingi bo'lim:** [24-migration-tooling.md](24-migration-tooling.md) — JS → TS migration, `allowJs`, `checkJs`, build tools, monorepo tooling.
