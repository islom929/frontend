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

TypeScript **structural typing** ishlatadi — ikki type ning "shakli" (property/method to'plami) bir xil bo'lsa, ular bir-biriga assignable. Bu odatda foydali, lekin ba'zi hollarda zararli — `UserId` va `PostId` ikkalasi `number` bo'lsa ham, ular **semantik** farq qiladi va aralashtirilmasligi kerak.

**NIMA UCHUN:** Strukturaviy `number` type bu xatoga to'sqinlik qilmaydi:
```typescript
function getUser(id: number) { /* ... */ }
const postId: number = 42;
getUser(postId); // ✅ TS, lekin SEMANTIK XATO
```

Branded types — base type ga **fiziktan mavjud bo'lmagan** phantom property qo'shib, structural typing'ni "sindiradi". Natija: ikki struktural bir xil type kompile-time da farqlanadi (nominal typing).

**QANDAY ISHLAYDI:**

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<number, "UserId">;
//  ↓ effectively: number & { readonly __brand: "UserId" }
```

`UserId` `number` ning intersection'i + phantom property. Runtime'da `__brand` property mavjud emas (har raqam shunchaki number). Lekin compile-time'da TS bu typeni boshqa branded number'lardan farqlaydi — `__brand: "PostId"` `__brand: "UserId"` ga assignable emas (literal type'lar mos kelmaydi).

**Constructor pattern majburiy:** `const x: UserId = 42` ban (number → UserId assignable emas, brand yo'q). Faqat constructor function orqali yaratiladi:

```typescript
function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("Invalid UserId");
  return id as UserId;  // assertion — runtime'da o'zgarmaydi, faqat type'ni "muhrlaydi"
}
```

Bu **validation gate** beradi — har `UserId` validate qilingan number ekanligi kafolatlanadi.

Batafsil [Bo'lim 16: Brand<T, B>](16-custom-utility-types.md#brandt-b--brandednominal-types) va [Bo'lim 21: Design Patterns](21-design-patterns.md).

<details>
<summary><strong>Under the Hood</strong></summary>

**Type system mexanizmi:**

```typescript
type UserId = number & { readonly __brand: "UserId" };
```

Bu intersection type. TS structural assignability'ni tekshirganda:
1. Source `number` (literal `42`)
2. Target `number & { __brand: "UserId" }`
3. `number` qismi mos (number ⊆ number)
4. Lekin `{ __brand: "UserId" }` qismi → `42` da `__brand` property yo'q → assignable emas

**Phantom property paradox:** Property `__brand` runtime'da yaratilmaydi — bu **fantom** (compile-time-only). Lekin TS uni real property dek tekshiradi. `as UserId` assertion bu tekshiruvni bypass qiladi (assertion "men bilaman" deydi).

**Symbol-based brand (kuchliroq encapsulation):**

```typescript
declare const userIdBrand: unique symbol;
type UserId = number & { readonly [userIdBrand]: void };
```

`unique symbol` — har deklaratsiya alohida unique type yaratadi. `userIdBrand` faqat shu module'da expose, boshqa module'da `__brand: "UserId"` literal yaratib bypass qilib bo'lmaydi.

**Runtime cost:** **NOL** — branded types pure compile-time. Emit'da `as UserId` olib tashlanadi, raqam shunchaki raqam bo'lib qoladi. Hech qanday runtime overhead.

```typescript
// Source
const id = createUserId(42);

// Emit (target ES2022)
const id = createUserId(42);  // ↑ assertion olib tashlandi, function call qoldi
```

**`Brand` vs Opaque types:** "Opaque" termin bu pattern uchun ham ishlatiladi (boshqa tilllardan kelgan — Flow, Haskell newtype). TS'da `Brand` va `Opaque` synonyms — implementation bir xil.

**Arithmetic problem:** Branded number'lar arithmetic operation'larda brand'ni **yo'qotadi**:

```typescript
const price: USD = createUSD(10);
const doubled = price * 2;  // doubled: number (USD emas!)
```

Sabab: `*`, `+`, `-`, `/` operator'larining return type'i `number`. Brand intersection emas. Yechim: `unsafeMultiply` kabi explicit function'lar yozish yoki brand'ni qayta tasdiqlash (`(price * 2) as USD`).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
type Email = Brand<string, "Email">;

function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("Invalid UserId");
  return id as UserId;
}

function createPostId(id: number): PostId {
  if (id <= 0) throw new Error("Invalid PostId");
  return id as PostId;
}

function createEmail(value: string): Email {
  if (!value.includes("@")) throw new Error("Invalid email");
  return value as Email;
}

function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

const userId = createUserId(1);
const postId = createPostId(1);

getUser(userId);  // ✅
// getUser(postId); // ❌ — PostId ≠ UserId, brand mos kelmaydi
// getUser(42);     // ❌ — number ≠ UserId, brand yo'q

// === Symbol-based brand (kuchliroq) ===
declare const emailBrand: unique symbol;
type SafeEmail = string & { readonly [emailBrand]: void };

function createSafeEmail(v: string): SafeEmail {
  if (!/^.+@.+\..+$/.test(v)) throw new Error("Invalid email");
  return v as SafeEmail;
}

// === Zod brand integration ===
import { z } from "zod";
const UserIdSchema = z.number().positive().brand<"UserId">();
type ZodUserId = z.infer<typeof UserIdSchema>;
// Zod parse + brand bir vaqtning o'zida
```

</details>

---

## Exhaustive Pattern Matching

### Nazariya

Exhaustive matching — discriminated union'ning **barcha variant'larini** kompile-time da handle qilganingizni kafolatlash. Yangi variant qo'shilganda kompilator handle qilinmagan joylar bo'yicha error chiqaradi — runtime'gacha kechiktirilmaydi.

**NIMA UCHUN:** Discriminated union'da `switch` ishlatganda yangi case qo'shsangiz, kompilator buni o'z-o'zidan ushlamaydi:

```typescript
type Status = "active" | "inactive";
function label(s: Status): string {
  switch (s) {
    case "active": return "Active";
    case "inactive": return "Inactive";
  }
}
// Keyin Status'ga "suspended" qo'shilsa:
// label("suspended") → return undefined (silent bug)
```

Exhaustive check — `default` case'da `assertNever` ishlatib, kompilator'ni "barcha case handle qilingan" deb tasdiqlashga majbur qilamiz.

**QANDAY ISHLAYDI — `never` mexanizmi:**

```typescript
function assertNever(x: never): never {
  throw new Error(`Unexpected: ${JSON.stringify(x)}`);
}
```

`never` — TypeScript ning bottom type'i (hech qanday qiymat unga assignable emas, faqat `never` o'zi). `switch` ichida har case match qilingach, control flow analysis qoldiq type'ni narrow qiladi:

```
shape: "circle" | "square" | "triangle"
   ↓ case "circle"
shape: "square" | "triangle"   (control flow narrowing)
   ↓ case "square"
shape: "triangle"
   ↓ case "triangle"
shape: never  ← bu yerga yetib kelishi mumkin emas
   ↓
default → assertNever(shape) ✅ (shape: never, parameter accept qiladi)
```

Agar yangi variant `"pentagon"` qo'shilsa:
```
default da shape: "pentagon"
   ↓
assertNever(shape) → ❌ "Type 'pentagon' is not assignable to parameter of type 'never'"
```

Bu pattern compile-time exhaustiveness garantee beradi — runtime testlar shart emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**Control flow analysis va narrowing:** TS kompilator har `case` blokdan keyin discriminant property'ni narrow qiladi (CFA — Control Flow Analysis). Discriminant — union'dagi barcha member'lar ulashadigan literal property (odatda `kind`, `type`, `tag`).

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

declare const shape: Shape;
if (shape.kind === "circle") {
  shape.radius;  // ✅ TS narrow: { kind: "circle"; radius: number }
  shape.side;    // ❌ Property doesn't exist
}
```

**Bottom type `never` semantics:** TS type system'da `never` — bo'sh set (no values). Universal subtype — har type'ga assignable, lekin hech qanday qiymat `never` ga assign qilib bo'lmaydi (faqat `never` o'zi). Bu xususiyat exhaustiveness uchun ideal:
- `assertNever(x: never)` — `x` ga faqat exhaustive narrowed branch'dan kelishi mumkin
- Agar variant qoldirilsa, `x` aslida narrowed type bo'ladi → kompilator error

**Record map vs Switch trade-off:**

| Approach | Foyda | Kamchilik |
|----------|-------|----------|
| `switch + assertNever` | Type narrowing avtomatik | Yangi variant qo'shilsa, default'gacha tushadi |
| `Record<Kind, Handler>` | Compile-time exhaustive (property must exist) | Handler ichida `Extract<...>` yoki `any` kerak |
| `satisfies Record<...>` (TS 4.9+) | Type check + narrow values | Bir oz verbose syntax |

**Runtime cost:** `assertNever` function — kichkina overhead (faqat unreachable branch'da). Production'da `process.env.NODE_ENV === "production"` bilan minimal'ga olib kelinishi mumkin.

</details>

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
    // Yangi variant ("pentagon") qo'shilsa →
    // bu yerda compile error: pentagon never ga assign bo'lmaydi
  }
}

// === Record map bilan exhaustive ===
// Handler type'i Extract bilan aniqlanmasa, parameter type lost bo'ladi
// (s qiymati shape'ning union type'i bo'lib qoladi — har property aniq emas)
type ShapeHandler<K extends Shape["kind"]> = (s: Extract<Shape, { kind: K }>) => number;

const areaCalculators = {
  circle: (s) => Math.PI * s.radius ** 2,
  square: (s) => s.side ** 2,
  triangle: (s) => (s.base * s.height) / 2,
} satisfies { [K in Shape["kind"]]: ShapeHandler<K> };
// Yangi kind qo'shilsa — bu yerda error (property missing)

// === satisfies bilan inline (TS 4.9+) ===
const handlers = {
  circle: (s: Extract<Shape, { kind: "circle" }>) => Math.PI * s.radius ** 2,
  square: (s: Extract<Shape, { kind: "square" }>) => s.side ** 2,
  triangle: (s: Extract<Shape, { kind: "triangle" }>) => (s.base * s.height) / 2,
} satisfies { [K in Shape["kind"]]: (s: Extract<Shape, { kind: K }>) => number };
```

</details>

---

## `const` Assertions

### Nazariya

`as const` (const assertion) — object va array literal'ni **deeply readonly** va **literal types** bilan inferred qiladi. TS 3.4+ da kiritilgan, enum'ga zero-runtime-cost alternativa sifatida ishlatiladi.

**NIMA UCHUN:** Default'da TS object literal'ni widen qiladi (kontekstga moslab `string`/`number` type'ga aylantiradi). Bu ko'p hollarda zarur, lekin discriminant constant'lar yoki enum-like values uchun aniq literal kerak:

```typescript
// const assertion'sis
const config = { mode: "production", port: 3000 };
// config: { mode: string; port: number } — literal'lar widen qilindi

// `as const` bilan
const config = { mode: "production", port: 3000 } as const;
// config: { readonly mode: "production"; readonly port: 3000 }
```

**Uch effekt birga:**
1. **Object property'lari `readonly`** — mutation ban
2. **Array `readonly tuple`** — `as const` array'ni `readonly [T1, T2, ...]` tuple ga aylantiradi
3. **String/number literal'lar widen qilinmaydi** — `"production"` `string` ga aylantirilmaydi, qoladi `"production"`

**`satisfies` operator (TS 4.9+):** `as const` literal types saqlaydi, lekin shape constraint berolmaydi. `satisfies` bo'sh joyni to'ldiradi — type'ni tekshirib, lekin **inferred type'ni o'zgartirmaydi** (narrow saqlaydi):

```typescript
const routes = { home: "/", about: "/about" } as const satisfies Record<string, string>;
// 1. as const → literal types ("/", "/about")
// 2. satisfies → "Record<string, string>" constraint tekshiriladi
// 3. routes type: { readonly home: "/"; readonly about: "/about" } (narrow saqlandi)
```

`as Record<string, string>` ishlatsa, type'ni widen qiladi (literal'lar yo'qoladi). `satisfies` esa **faqat tekshiradi**, type'ni o'zgartirmaydi.

**`const` type parameter (TS 5.0+):** Generic function'larda parameter'ga `const` modifier qo'shsa, kompilator argument'ni avtomatik `as const` deb infer qiladi:

```typescript
function pick<const T extends readonly string[]>(arr: T): T[number] {
  return arr[Math.floor(Math.random() * arr.length)];
}

const result = pick(["red", "green", "blue"]);
// result: "red" | "green" | "blue" (literal union, `string` emas)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Widening mexanizmi:** TS inference'da literal'lar default'da widen qilinadi (mutability assumption'iga ko'ra):

```typescript
let x = "hello";        // x: string (widened — let mutable)
const y = "hello";      // y: "hello" (const, primitive narrowed)
const arr = [1, 2];     // arr: number[] (object literal widened)
const arr2 = [1, 2] as const; // arr2: readonly [1, 2] (narrowed)
```

Reason: `const arr = [1, 2]; arr.push(3)` legal — kompilator default'da mutation expect qiladi.

**`as const` semantics:**
- Primitive literal → literal type (`"x"` → `"x"`, `42` → `42`)
- Array literal → `readonly` tuple (`[1, 2]` → `readonly [1, 2]`)
- Object literal → barcha property `readonly`, nested ham
- Function expression → o'zgarmaydi (function literal mavjud emas)

**Enum vs `as const`:**

```typescript
// enum
enum Status { Active = "active", Inactive = "inactive" }
// Emit: function-based runtime object
// TS-only — JS source'ga compile bo'lmaydi (verbatimModuleSyntax bilan ban)

// as const alternative
const Status = { Active: "active", Inactive: "inactive" } as const;
type Status = (typeof Status)[keyof typeof Status];  // "active" | "inactive"
// Emit: oddiy object literal
// JS-compatible, zero overhead
```

`as const` afzal:
- Bundle size kichikroq (runtime helper yo'q)
- Tree-shaking yaxshi (har property ajratiladi)
- `verbatimModuleSyntax` bilan compatible
- `Status["Active"]` value `"active"` (string), enum'da numeric/string mixed bo'lishi mumkin (kutilmagan)

**`satisfies` bilan use case'lar:**

```typescript
// API endpoint definitions
const endpoints = {
  users: { method: "GET", path: "/users" },
  createUser: { method: "POST", path: "/users" },
} as const satisfies Record<string, { method: string; path: string }>;

// endpoints.users.method: "GET" (narrow literal saqlandi)
// Lekin shape constraint tekshirildi: har endpoint method+path bo'lishi shart
```

**Runtime cost:** `as const` — pure compile-time, runtime'da yo'q. `satisfies` ham pure compile-time.

</details>

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

Type-safe builder — fluent API'da `.build()` chaqirilganda **barcha required field'lar to'ldirilganligini compile-time da kafolatlaydigan** pattern. Classic builder pattern (`new Builder().setX().setY().build()`) runtime'da xatolar tashlaydi. Type-safe versiya bu xatolarni TS type system orqali ushlaydi.

**NIMA UCHUN:** Classic builder kamchiliklari:

```typescript
// ❌ Classic builder — runtime risk
const config = new DbBuilder()
  .setHost("localhost")
  // .setPort(5432) UNUTILDI
  .build(); // runtime: throw "port required"
```

Type-safe builder kompile-time'da bu xatoni ushlaydi — `.build()` faqat barcha required field'lar set bo'lganda chaqirilishi mumkin.

**Ikki yondashuv:**

**1. Step builder** — har qadam **boshqa interface** qaytaradi. Har step keyingi mumkin bo'lgan method'larni cheklaydi:

```typescript
interface NeedsHost { setHost(h: string): NeedsPort; }
interface NeedsPort { setPort(p: number): NeedsDb; }
interface NeedsDb { setDb(d: string): Buildable; }
interface Buildable { build(): DbConfig; }
```

Foyda: explicit step order. Kamchilik: verbose, ko'p interface, har step tartibi qattiq.

**2. Phantom type builder** — generic `Provided` parameter har `set()` chaqirilganda kengayadi. `build()` faqat `Provided` `RequiredKeys<T>` ni qamrab olsa ruxsat:

```typescript
class Builder<T, Provided extends keyof T = never> {
  set<K extends keyof T>(k: K, v: T[K]): Builder<T, Provided | K> { ... }
  build(this: RequiredKeys<T> extends Provided ? Builder<T, Provided> : never): T { ... }
}
```

Foyda: flexible order, kam interface. Kamchilik: type magic murakkab (junior'larga tushunish qiyin).

Batafsil [Bo'lim 21: Builder Pattern](21-design-patterns.md#creational-builder-pattern).

<details>
<summary><strong>Under the Hood</strong></summary>

**Phantom type builder mexanizmi qadam-baqadam:**

```typescript
type RequiredKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? never : K;
}[keyof T];
// Mapped type — har key uchun:
// - undefined T[K] ga assignable? → never (optional)
// - aks holda → K (required)
// [keyof T] — barcha "never" larni filter, faqat K qoladi
```

```typescript
interface DbConfig {
  host: string;       // required
  port: number;       // required
  database: string;   // required
  username?: string;  // optional
  ssl?: boolean;      // optional
}

type R = RequiredKeys<DbConfig>;
// "host" | "port" | "database"
```

**`set()` return type kengayishi:**

```typescript
new Builder<DbConfig>()              // Builder<DbConfig, never>
  .set("host", "localhost")          // Builder<DbConfig, "host">
  .set("port", 5432)                 // Builder<DbConfig, "host" | "port">
  .set("database", "mydb")           // Builder<DbConfig, "host" | "port" | "database">
  .build();                          // ✅ RequiredKeys<T> ("host"|"port"|"database") ⊆ Provided
```

**`build()` constraint mexanizmi — `this` parameter trick:**

```typescript
build(
  this: RequiredKeys<T> extends Provided ? Builder<T, Provided> : never
): T { ... }
```

TS'da `this` parameter'i — method chaqiruvchi `this` type'iga constraint. `RequiredKeys<T> extends Provided` — conditional type:
- Required'lar berildi → `this: Builder<T, Provided>` (method chaqirish mumkin)
- Yetishmadi → `this: never` (method chaqirish mumkin emas, kompilator error)

```typescript
new Builder<DbConfig>()
  .set("host", "x")
  .build();
// Provided: "host"
// RequiredKeys<DbConfig>: "host" | "port" | "database"
// "host" | "port" | "database" extends "host"? → ❌ false
// this: never → "Property 'build' is not callable"
```

**Runtime cost:** Yo'q — barcha generic constraint'lar kompile-time. Build'da TypeSafeBuilder oddiy class sifatida emit qilinadi. Data accumulation `data: Partial<T>` orqali — runtime ishlash standart.

**Trade-off:** Phantom type builder TS magic ko'p (mapped type, conditional `this`, type assertion'lar `as unknown as`). Library API sifatida yaxshi (consumer murakkablikni ko'rmaydi), application kod uchun step builder yoki Zod schema tezroq.

</details>

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

Type-safe event emitter — event nomi va payload kompile-time'da tekshiriladigan publisher/subscriber pattern. Klassik `EventEmitter` (Node.js core) string event name va `any[]` payload bilan ishlaydi — type safety yo'q. Generic event map bu kamchilikni hal qiladi.

**NIMA UCHUN:** Klassik Node `EventEmitter`:

```typescript
emitter.on("user.login", (user, timestamp) => {
  // user: any, timestamp: any
});
emitter.emit("user.logn", "ali");  // typo, runtime'da silent
```

Type-safe versiya event nomidan tortib payload type'ini ham tekshiradi.

**QANDAY ISHLAYDI:**

```typescript
interface AppEvents {
  userLoggedIn: (userId: string, timestamp: Date) => void;
  orderPlaced: (orderId: string, total: number) => void;
}

const emitter = new TypedEmitter<AppEvents>();
emitter.on("userLoggedIn", (userId, ts) => { ... });
//          ↑ string literal autocomplete + check
//                          ↑ userId: string, ts: Date (inferred)
```

Event map (`interface AppEvents`) — single source of truth. Har key event nomi, qiymati listener function signature'i. Emitter generic `T extends EventMap` bu map'ni qabul qiladi va `keyof T` (event nomlar) va `Parameters<T[K]>` (payload) orqali type-check qiladi.

[Bo'lim 21: Observer Pattern](21-design-patterns.md#behavioral-observer-pattern) da pattern asoslari berilgan.

<details>
<summary><strong>Under the Hood</strong></summary>

**Generic event map mexanizmi:**

```typescript
type EventMap = Record<string, (...args: any[]) => void>;
```

`EventMap` constraint — har property string key + function value. `any[]` shu yerda zarur — har event'da payload soni va type'i farq qiladi, lekin generic constraint sifatida `any[]` qabul qilamiz. Concrete subtype (`AppEvents`) `any[]` ni o'z literal type'lari bilan o'rnini bosadi.

**Type inference zanjir:**

```typescript
on<K extends keyof T>(event: K, listener: T[K]): this
//   ↓ chaqiruv
emitter.on("userLoggedIn", fn);
//          ↑ K narrowing: "userLoggedIn"
//                       ↑ listener type: T["userLoggedIn"] = (userId: string, ts: Date) => void
```

`K extends keyof T` — K'ni event nomlari union'iga cheklaydi. `T[K]` — indexed access type, kompilator K'ni narrow'lab listener signature'ini topadi.

**`Parameters<T[K]>` utility:** Built-in utility — function type'dan parameter tuple'ni chiqaradi:

```typescript
type Fn = (a: string, b: number) => void;
type P = Parameters<Fn>;  // [a: string, b: number]
```

`emit<K extends keyof T>(event: K, ...args: Parameters<T[K]>)` — `args` tuple aniq event'ning expected parameter'lariga teng bo'lishi shart.

**Listener storage type erasure:**

```typescript
private listeners = new Map<keyof T, Set<(...args: any[]) => void>>();
```

Internal storage `any[]` ishlatadi — chunki bir map'da bir nechta event turlari saqlanishi mumkin (har biri o'z signature'i bilan). Public API generic constraint orqali type safety beradi, internal `any` consumer'ga ko'rinmaydi. Bu yagona o'rinli `any` (interface boundary'da generic, ichida heterogeneous storage).

**Built-in alternative — `EventTarget` (DOM, Node 15+):** Native API ham mavjud, lekin payload single `Event` object'da (multi-arg yo'q). Type-safety uchun custom wrapper kerak.

**Production library'lar:** `mitt`, `nanoevents`, `tiny-typed-emitter`, `eventemitter3` — har biri turli trade-off (bundle size, API stilini, type safety darajasi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Internal storage uchun any[] — generic constraint zarurati
// (public API type-safe, ichida heterogeneous storage)
type EventMap = Record<string, (...args: any[]) => void>;

class TypedEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<(...args: any[]) => void>>();

  on<K extends keyof T>(event: K, listener: T[K]): this {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener);
    return this;
  }

  off<K extends keyof T>(event: K, listener: T[K]): this {
    this.listeners.get(event)?.delete(listener);
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
  // userId: string, ts: Date — ✅ inferred
  console.log(`${userId} logged in at ${ts.toISOString()}`);
});

emitter.emit("userLoggedIn", "user-1", new Date()); // ✅
// emitter.emit("userLoggedIn", 42, "wrong");       // ❌ — type mismatch
// emitter.emit("unknownEvent", "x");               // ❌ — event yo'q
```

</details>

---

## Runtime Validation + Types

### Nazariya

TypeScript type'lar **kompile-time-only** — emit'dan keyin yo'q. Runtime'da `JSON.parse`, `fetch`, `localStorage`, form input — barchasi `unknown`/`any` qaytaradi, TS bularning shape'ini kafolatlamaydi. **Runtime validation library'lar** schema'ni bir marta yozib, ham runtime validation ham compile-time type'ni undan chiqaradi (single source of truth).

**NIMA UCHUN:** Ko'p loyihalarda quyidagi anti-pattern:

```typescript
// ❌ Type assertion bilan TS'ni "aldash" — runtime crash
const user = (await fetch("/api/user").then(r => r.json())) as User;
user.name.toUpperCase(); // ❌ Agar API noto'g'ri shape qaytarsa, runtime'da crash
```

Schema-based validation runtime'da to'g'ri tekshiradi va type'ni xavfsiz chiqaradi:

```typescript
const UserSchema = z.object({ name: z.string(), email: z.string().email() });
type User = z.infer<typeof UserSchema>;

const user = UserSchema.parse(await response.json());
// Agar shape mos kelmasa — Zod error tashlaydi (parse vaqtida)
// Mos kelsa — user: User (kafolatlangan)
```

**Library tanlash criteria:**

| Library | Xususiyat | Trade-off |
|---------|-----------|-----------|
| **Zod** | Chaining API, eng katta ecosystem, integrations (tRPC, React Hook Form) | Tree-shaking cheklangan — barcha builder kodlari bundle'da |
| **Valibot** | Tree-shakeable function'lar, modular | Yangi (kichikroq ecosystem), API'si turlicha |
| **io-ts** | Functional (fp-ts bilan), `Either<Errors, T>` qaytaradi | fp-ts dependency, steeper learning curve |
| **ArkType** | TS-syntax-like schema (`"string | number"`), tezroq runtime | Bundle hajmi katta, ecosystem yangi |

**Bundle hajmlar tez-tez o'zgaradi** — `bundlephobia.com` yoki `pkg-size.dev` orqali real-time tekshirish tavsiya. Asosiy farq: Zod monolithic, Valibot composable function'lar (faqat ishlatilgani bundle'ga tushadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Schema → Type chiqarish mexanizmi (Zod misolida):**

```typescript
const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

type User = z.infer<typeof UserSchema>;
// { name: string; age: number }
```

`z.object({...})` runtime'da `ZodObject` instance qaytaradi. Type darajasida `ZodObject<{ name: ZodString; age: ZodNumber }>` — generic parameter. `z.infer<T>` conditional type'lar zanjiri orqali generic parameter'dan "output type"'ni chiqaradi:

```typescript
type infer<T extends ZodType<any>> = T["_output"];
// ZodObject ichida _output phantom property bor (compile-time-only)
```

Bu **type-level computation** — runtime'da hech narsa ishlamaydi, faqat TS compiler schema struct'ini "o'qib" type yaratadi.

**`safeParse` vs `parse`:**

```typescript
// parse — error tashlaydi (throw)
const user = UserSchema.parse(data);  // throws ZodError

// safeParse — discriminated union qaytaradi (Result-like)
const result = UserSchema.safeParse(data);
if (result.success) {
  result.data;     // User
} else {
  result.error;    // ZodError
}
```

`safeParse` Result pattern'ga yaqin — error handling explicit. Production'da `safeParse` afzal (silent failure'lar yo'q).

**Validation runtime overhead:** Har `parse()` chaqiruvi schema'ni traversal qiladi — har property tekshiriladi, regex match'lar bajariladi. Hot path'larda (har request'da) cost'lik. Yechim:
- **Boundary validation** — faqat data'ning kirish nuqtasida (API response, form submit)
- **Memoization** — bir xil schema bir xil data uchun cache
- **Schema compilation** — Zod'da `z.optimizer` (experimental), Ajv da JSON Schema → compiled function

**TypeBox alternativa:** JSON Schema generate qiladigan library — Ajv (eng tez JSON validator) bilan ishlaydi. Performance-critical (API gateway, microservice) loyihalarda Zod'dan tezroq.

```typescript
import { Type, Static } from "@sinclair/typebox";

const User = Type.Object({
  name: Type.String(),
  age: Type.Number(),
});
type User = Static<typeof User>;
// JSON Schema generated → Ajv compile → tezroq validate
```

**Type vs Runtime mismatch danger:** TS'da type guard (`function isUser(x): x is User`) yozish va schema'ni alohida saqlash mumkin, lekin ikkisi sync'dan chiqishi mumkin. Single source of truth (schema'dan type infer) bu xatoni yo'qotadi.

</details>

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

`throw`/`try/catch` JavaScript'ning standart error handling mexanizmi. Lekin TypeScript'da uning kamchiliklari bor:
- **Type system'ga ko'rinmaydi** — function signature `getUser(): User` xato tashlashi mumkinligi ko'rsatilmaydi (Java'da `throws` mavjud, TS'da yo'q)
- **`catch (e)`** parameter'i `unknown` (TS 4.4+ strict) — har gal narrowing kerak
- **Implicit control flow** — har function'da exception bubble up qilishi mumkin

**Result type** — error'ni qiymat sifatida qaytarish (functional pattern). Caller error'ni explicit handle qilishga kompile-time'da majbur:

```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function getUser(id: number): Result<User, "NotFound" | "Unauthorized"> {
  if (!authorized) return { ok: false, error: "Unauthorized" };
  const user = findUser(id);
  if (!user) return { ok: false, error: "NotFound" };
  return { ok: true, value: user };
}

const r = getUser(42);
// r.value;  // ❌ — kompilator narrowing talab qiladi
if (r.ok) r.value;  // ✅ User
else r.error;       // ✅ "NotFound" | "Unauthorized"
```

**Bu "Railway-oriented programming"** (Scott Wlaschin tomonidan F# context'ida nomlangan) — har function ikki "rail"da harakat qiladi: success rail (Ok) va failure rail (Err). Composition: `map`/`flatMap` natijani success rail'da ushlaydi, error rail'ga tushgan qiymat oxirgacha o'tib boradi.

**`throw` vs Result trade-off:**

| Aspect | `throw` | `Result<T, E>` |
|--------|---------|----------------|
| Type system'da ko'rinish | Yo'q | Ha (return type'da) |
| Caller handle majburiyligi | Yo'q (catch ixtiyoriy) | Ha (narrowing zarur) |
| Composition | `try/catch` har layer'da | `map`/`flatMap` zanjirida |
| Performance | Stack unwind cost | Object allocation cost |
| Ecosystem | Built-in, native | Library kerak yoki manual |

**Qachon `throw` afzal:** Truly exceptional condition (out-of-memory, programmer error). **Qachon Result afzal:** Expected business error'lar (validation, not-found, unauthorized) — bular flow'ning normal qismi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Discriminated union mexanizmi:** Result type discriminated union — `ok` field discriminant. TS control flow analysis `if (r.ok)` orqali type'ni narrow qiladi:

```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

declare const r: Result<string, Error>;
if (r.ok) {
  r.value;  // string (narrowed)
  // r.error; // ❌ property doesn't exist
} else {
  r.error;  // Error
  // r.value; // ❌
}
```

`ok: true` va `ok: false` — literal type'lar (boolean union'ning two branches). Narrowing exact-match orqali ishlaydi.

**Type-safe constructor'lar:**

```typescript
function ok<T>(value: T): Result<T, never> { return { ok: true, value }; }
function err<E>(error: E): Result<never, E> { return { ok: false, error }; }
```

`Result<T, never>` — error rail bo'sh, faqat success. `Result<never, E>` — success rail bo'sh, faqat error. Union construction'da:

```typescript
type T1 = Result<string, never>;  // { ok: true; value: string } | never
type T2 = Result<string, Error>;  // success branch'i T1 bilan compatible
// never qism union'dan tushib ketadi
```

**`map` va `flatMap` semantics:**

```typescript
function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return r.ok ? ok(fn(r.value)) : r;
}
```

`map` — success qiymatni transform qiladi, error o'tkazib yuboradi. `r` ning `r.ok === false` versiyasi `Result<never, E>` — bu `Result<U, E>` ga assignable (never universal subtype).

```typescript
function flatMap<T, U, E>(r: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return r.ok ? fn(r.value) : r;
}
```

`flatMap` — success'da `fn` ham Result qaytaradi, double-wrapping oldi olinadi (`Result<Result<U, E>, E>` emas, `Result<U, E>`). Bu monadic bind operation.

**Error union'lar composition:** `flatMap` ikki turli error type'larni birlashtirishi mumkin:

```typescript
function step1(): Result<number, "E1"> { ... }
function step2(n: number): Result<string, "E2"> { ... }

const r = flatMap(step1(), step2);
// r: Result<string, "E1" | "E2">  ← error union
```

Bu kuchli — har step o'z error type'ini deklaratsiya qiladi, pipeline barcha mumkin bo'lgan error'larni birlashtiradi.

**Production library — `neverthrow`:** Stable TS Result library. Async variant — `ResultAsync<T, E>` Promise'larni handle qiladi. `.match()`, `.unwrapOr()`, `.andThen()` (flatMap) — fluent API.

**Performance comparison:** Result'da har step object allocation. `throw` da stack unwind. Modern V8'da har ikkalasi optimized — hot path'da Result biroz tezroq (predictable control flow), cold path'da bir xil. Microbenchmark'lar context-specific.

</details>

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
