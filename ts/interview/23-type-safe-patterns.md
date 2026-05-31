# Interview: Type-Safe Patterns

> Branded/opaque types (nominal typing), exhaustive pattern matching, `as const satisfies`, type-safe builder, type-safe event emitter, runtime validation (Zod/Valibot/io-ts/ArkType), Result type bilan error handling bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Output savollar](#output-savollar)
- [Amaliy savollar (Coding)](#amaliy-savollar-coding)
- [Bug fix savollar](#bug-fix-savollar)
- [Xulosa](#xulosa)

---

## Nazariy savollar

### Savol 1: Branded (opaque) type nima va qachon kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Branded type — phantom property orqali TypeScript'ning structural typing'ini "sindirib", nominal typing beradi. Bir xil base type'ga ega bo'lgan ikki ID (`UserId` va `PostId`, ikkalasi `number`)'ni compile-time'da ajratish uchun ishlatiladi.

### To'liq tushuntirish

TypeScript structural type system'da type'larning **shakli** muhim — agar ikki type bir xil shape'ga ega bo'lsa, ular interchangeable. Bu xavfli:

```typescript
function getUser(id: number) {}
function getPost(id: number) {}

getUser(postId); // ✅ TS — lekin runtime xato (noto'g'ri ID)
```

Branded type runtime'da mavjud bo'lmagan (phantom) property qo'shadi:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<number, "UserId">;
```

Phantom — `__brand` haqiqatan runtime'da yo'q. Faqat type system'da mavjud, intersection orqali. Constructor function ichida `as` assertion bilan brand "beriladi":

```typescript
function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("Invalid UserId");
  return id as UserId;
}
```

Endi:

- `UserId` va `PostId` ikkalasi `number`, lekin compile-time'da intersection brand'lari farqli — mos kelmaydi.
- `42 as UserId` ham xato (raw number → branded type).
- Faqat `createUserId(42)` qaytaradigan qiymat `UserId`.

Foyda: API surface, database query, ORM relation'larda noto'g'ri ID'ni compile-time'da ushlash. Validation logic constructor ichida — har ID validated bo'lishi kafolatlangan.

### Kod misol

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
type Email = Brand<string, "Email">;

function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("UserId must be positive");
  return id as UserId;
}

function createPostId(id: number): PostId {
  if (id <= 0) throw new Error("PostId must be positive");
  return id as PostId;
}

function createEmail(value: string): Email {
  if (!value.includes("@")) throw new Error("Invalid email");
  return value as Email;
}

function getUser(id: UserId): { id: UserId; name: string } {
  return { id, name: "Ali" };
}

function getPost(id: PostId): { id: PostId; title: string } {
  return { id, title: "Hello" };
}

const userId = createUserId(1);
const postId = createPostId(1);

getUser(userId);          // ✅
// getUser(postId);       // ❌ Argument of type 'PostId' not assignable to 'UserId'
// getUser(42);            // ❌ Argument of type 'number' not assignable to 'UserId'
// getUser(42 as UserId); // Compile'da o'tadi, lekin validation bypass

// === unique symbol bilan to'liq nominal ===
declare const EmailTag: unique symbol;
type StrongEmail = string & { readonly [EmailTag]: void };

function makeStrongEmail(v: string): StrongEmail {
  if (!v.includes("@")) throw new Error("Invalid");
  return v as StrongEmail;
}

// __brand: "Email" va __brand: "Phone" — agar developer xato qilib bir xil string yozsa,
// type collision. unique symbol har declaration uchun unique — collision yo'q.
```

### Edge Cases

- **Arithmetic brand'ni yo'qotadi**: `const total = price * 2;` — `total: number`, `USD` emas. Operator natijasi har doim base type'ga aylanadi. Yechim: helper function (`addUSD(a: USD, b: USD): USD`).
- **JSON serialization**: `JSON.stringify(userId)` runtime'da oddiy `number` — branded shakl yo'q. Parse'da qayta branding qilish kerak (`createUserId(JSON.parse(...))`).
- **`as unknown as Brand`**: bypass mumkin. Lint rule (`@typescript-eslint/consistent-type-assertions`) yoki encapsulation (constructor faqat alohida module'da export).
- **Generic infer**: `T extends Brand<infer U, any>` bilan brand'ni "ochish" mumkin (advanced utility).

### Follow-up savollar

1. "Branded type runtime'da ham unique bo'lishi mumkinmi?" — Class instance bilan (`class UserId { constructor(public value: number) {} }`). Lekin verbose va serialization murakkab. Phantom branded type modern stil.
2. "Zod `.brand()` qanday ishlaydi?" — Zod schema'ga brand qo'shadi, `z.infer<typeof Schema>` branded type qaytaradi. Runtime'da `.parse()` validation + brand'ni ta'minlaydi (single source of truth).

</details>

---

### Savol 2: Exhaustive pattern matching — `never` qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`switch` statement union'ning barcha variant'larini handle qilganini compile-time'da kafolatlash uchun `default` branch'da `assertNever(x: never)` ishlatiladi. Yangi variant qo'shilsa — TS error: `Argument of type 'NewVariant' is not assignable to parameter of type 'never'`.

### To'liq tushuntirish

`never` type — hech qachon yuzaga kelmaydigan qiymat. Bo'sh union (`Extract<"a", "b">` → `never`), throw qilingan function'ning return type.

Exhaustive check mexanizmi:

1. `switch` har case'da TS narrowing qiladi — union'dan handled variantni olib tashlaydi.
2. `default` branch'ga yetganda — qolgan union (handled emas) `never` bo'lishi kerak.
3. Agar union to'liq handle qilingan bo'lsa — `default`'da `shape` (yoki nima) `never`.
4. Agar yangi variant qo'shilsa va `switch`'da yangi case yo'q — `default`'da `shape` `NewVariant` bo'ladi, `never` emas.
5. `assertNever(shape)` `never` argument'ni kutadi — `NewVariant` mos kelmaydi → compile error.

Bu pattern "Switch + Exhaustive Check" — discriminated union bilan birga eng kuchli pattern.

Alternativ — `Record<DiscriminatorKey, Handler>` object map: yangi variant qo'shilsa property missing error.

### Kod misol

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function assertNever(value: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(value)}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":   return Math.PI * shape.radius ** 2;
    case "square":   return shape.side ** 2;
    case "triangle": return (shape.base * shape.height) / 2;
    default:         return assertNever(shape);
    // shape: never bu yerda — barcha variant handled
  }
}

// === Yangi variant qo'shilsa ===
type ShapeV2 =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number }
  | { kind: "pentagon"; side: number }; // ← yangi

function areaV2(shape: ShapeV2): number {
  switch (shape.kind) {
    case "circle":   return Math.PI * shape.radius ** 2;
    case "square":   return shape.side ** 2;
    case "triangle": return (shape.base * shape.height) / 2;
    default:         return assertNever(shape);
    // ❌ Error: Argument of type '{ kind: "pentagon"; side: number }'
    //    is not assignable to parameter of type 'never'
  }
}

// === Record-based exhaustive map ===
type AreaCalculator<S extends Shape> = (s: S) => number;

const areaCalculators: {
  [K in Shape["kind"]]: AreaCalculator<Extract<Shape, { kind: K }>>;
} = {
  circle:   (s) => Math.PI * s.radius ** 2,
  square:   (s) => s.side ** 2,
  triangle: (s) => (s.base * s.height) / 2,
  // Yangi kind qo'shilsa — property missing error
};

function areaMap<S extends Shape>(shape: S): number {
  return (areaCalculators[shape.kind] as AreaCalculator<S>)(shape);
}

// === satisfies bilan (TS 4.9+) ===
const handlers = {
  circle:   (s: Extract<Shape, { kind: "circle" }>)   => Math.PI * s.radius ** 2,
  square:   (s: Extract<Shape, { kind: "square" }>)   => s.side ** 2,
  triangle: (s: Extract<Shape, { kind: "triangle" }>) => (s.base * s.height) / 2,
} satisfies { [K in Shape["kind"]]: (s: Extract<Shape, { kind: K }>) => number };
```

### Edge Cases

- **`default` branch return type**: `assertNever` `never` qaytaradi — barcha branch'lar bir xil type'ga moslashadi (`number` masalan). Agar `default` `void` qaytarsa, switch return type kengaytiriladi.
- **`if`/`else` exhaustive**: faqat `if`/`else if` zanjirida ham ishlaydi, lekin `switch`'da TS narrowing aniqroq. `if (shape.kind === "circle") {...} else if (...)` — last `else`'da `assertNever(shape)`.
- **Discriminator union literal bo'lmasa**: `kind: string` umumiy bo'lsa, narrowing ishlamaydi. Discriminator har doim literal type ('circle', 'square') bo'lishi kerak.
- **Logging emas, throw**: `assertNever` `throw` qiladi (`never` qaytaradi). Production'da logging + fallback kerak bo'lsa, alohida helper (`logUnknownVariant`).

### Follow-up savollar

1. "TS 5.x'da `match` keyword (Rust kabi) bormi?" — Yo'q, TC39 proposal stage 1 (`Pattern Matching`). Hozircha `switch` + `assertNever` standart pattern.
2. "Non-discriminated union'da exhaustive qanday qilish?" — Discriminator qo'shish (`kind`/`type` field) yoki `instanceof` (class hierarchy). Plain object union exhaustive check'ni qiyinlashtiradi.

</details>

---

### Savol 3: `as const satisfies` — ikkalasi birga nima beradi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`as const` literal type + readonly beradi (deep). `satisfies` type validation qiladi. Birgalikda: type constraint tekshirish + literal qiymatlarni saqlash. Type annotation (`: T`)'dan farqli — narrow qiymat yo'qolmaydi.

### To'liq tushuntirish

Uchta yondashuv farqi:

**1. Type annotation (`: T`)** — qiymat type'ga moslashadi, lekin literal narrowing yo'qoladi:

```typescript
const t: Theme = { primary: "#007bff", size: 16 };
// t.primary: string | number (Theme'dagi type)
// "#007bff" literal yo'qotildi
```

**2. `as const`** — literal type + deep readonly, lekin type constraint tekshirilmaydi:

```typescript
const t = { primary: "#007bff", size: 16 } as const;
// t.primary: "#007bff" (literal)
// Lekin typo (primery, primry) tutilmaydi
```

**3. `as const satisfies T`** — ikkalasi:

```typescript
const t = { primary: "#007bff", size: 16 } as const satisfies Theme;
// t.primary: "#007bff" (literal saqlandi)
// Constraint Theme bilan tekshirildi (typo error beradi)
```

`satisfies` (TS 4.9+) operator — left side qiymat right side type'ga "rioya qiladi" (assignable), lekin qiymatning inferred type'ini o'zgartirmaydi. Bu farq:

- `const x: T = ...` — `x` type'i `T` (qiymatning aniq shakli yo'qoladi).
- `const x = ... satisfies T` — `x` type'i qiymatning inferred type, `T` faqat constraint.

### Kod misol

```typescript
type Theme = Record<string, string | number>;

// === ❌ Type annotation — literal yo'qoladi ===
const t1: Theme = { primary: "#007bff", size: 16 };
t1.primary; // string | number — aniq emas

// === ❌ Faqat as const — typo tutilmaydi ===
const t2 = { primery: "#007bff" } as const;
// typo "primery" — TS hech narsa demaydi

// === ✅ as const satisfies — eng kuchli ===
const t3 = { primary: "#007bff", size: 16 } as const satisfies Theme;
t3.primary; // "#007bff" literal
t3.size;    // 16 literal

// const t4 = { primery: "#007bff" } as const satisfies Theme;
// Theme = Record<string, string | number> — typo o'tadi (string key OK)

// Stricter constraint bilan:
type StrictTheme = { primary: string; size: number };
// const t5 = { primery: "#007bff", size: 16 } as const satisfies StrictTheme;
// ❌ Object literal may only specify known properties

// === Route routing exhaustive ===
const ROUTES = {
  home: "/",
  users: "/users",
  userDetail: "/users/:id",
} as const satisfies Record<string, `/${string}`>;
// Constraint: barcha qiymat "/" bilan boshlanadi
// ROUTES.userDetail: "/users/:id" literal

// === const type parameter (TS 5.0+) bilan combination ===
function createConfig<const T extends Record<string, unknown>>(config: T): T {
  return config;
}

const cfg = createConfig({ host: "localhost", port: 3000 });
// cfg: { readonly host: "localhost"; readonly port: 3000 }
// `const` keyword'siz: { host: string; port: number }
```

### Edge Cases

- **Mutable `as const satisfies`**: `as const` deep readonly qiladi. Agar mutation kerak bo'lsa, faqat `satisfies` (without `as const`) ishlatish: `const t = { ... } satisfies Theme`.
- **`satisfies` generic constraint bilan**: `function fn<T>(x: T satisfies Constraint)` — invalid syntax. `satisfies` faqat expression position'da, type parameter constraint o'rniga `extends` ishlatiladi.
- **Discriminated union narrowing**: `satisfies` bilan literal saqlanadi, discriminator narrowing aniqroq ishlaydi (`if (cfg.kind === "circle") cfg.radius // number literal`).
- **`as` (assertion) bilan farq**: `satisfies` tekshiradi, `as` "majburlaydi" (xavfli). `satisfies` xato bo'lganda compiler xato beradi, `as` esa unsafe cast.

### Follow-up savollar

1. "`satisfies` `as const` siz qachon ishlatish?" — Mutable object'ga constraint tekshirish kerak bo'lganda. Masalan, runtime'da modify qilinadigan config: `const cfg = { count: 0 } satisfies CounterConfig`.
2. "`as const` o'rniga `readonly` modifier ishlatib bo'ladimi?" — Type level'da ha (`readonly { primary: string }`), lekin `as const` qiymat darajasida deep readonly + literal narrowing. Stricter.

</details>

---

### Savol 4: Zod `z.infer` qanday ishlaydi? Single source of truth [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`z.infer<typeof Schema>` Zod schema'dan TypeScript type'ini chiqaradi. Schema runtime validation va compile-time type uchun single source of truth — interface'ni alohida yozish kerak emas, sync'dan chiqish ehtimoli yo'q.

### To'liq tushuntirish

TypeScript type'lari compile-time'da o'chiriladi. Runtime'da `JSON.parse` natijasi `any`, API response untyped — type-safety to'liq emas. Zod (yoki Valibot/io-ts/ArkType) schema runtime validator yaratadi va inference orqali TS type chiqaradi.

`z.object({...})` chaqirilganda Zod ichki structure yaratadi:

```typescript
type ZodObject<T> = {
  parse(data: unknown): T;
  safeParse(data: unknown): { success: true; data: T } | { success: false; error: ZodError };
  _output: T; // phantom type
};

type infer<T extends ZodType> = T["_output"];
```

`z.infer<typeof Schema>` `Schema["_output"]` ni qaytaradi — TS conditional types orqali inferred TS type.

Foyda:

- **Single source of truth**: schema o'zgarsa, type avtomatik yangilanadi.
- **Runtime + compile-time safety**: `.parse()` runtime'da validate, infer compile-time.
- **API boundary**: external data (fetch, JSON, form input) `unknown`'dan typed`'ga xavfsiz o'tish.

`.parse()` invalid'da `throw`, `.safeParse()` `{success, data, error}` object qaytaradi. Production'da `safeParse` afzal (error handling).

### Kod misol

```typescript
import { z } from "zod";

// Schema — single source of truth
const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(2).max(100),
  email: z.email(),
  age: z.number().min(18).max(120),
  role: z.enum(["admin", "user", "moderator"]),
  createdAt: z.date(),
});

// TS type — schema'dan chiqarildi
type User = z.infer<typeof UserSchema>;
// {
//   id: number;
//   name: string;
//   email: string;
//   age: number;
//   role: "admin" | "user" | "moderator";
//   createdAt: Date;
// }

// === Runtime validation ===
const apiResponse: unknown = { id: 1, name: "Ali", email: "ali@test.com", age: 25, role: "user", createdAt: new Date() };

// parse — throw qiladi invalid'da
try {
  const user = UserSchema.parse(apiResponse); // user: User
  console.log(user.email); // type-safe
} catch (e) {
  if (e instanceof z.ZodError) console.error(e.issues);
}

// safeParse — Result-like
const result = UserSchema.safeParse(apiResponse);
if (result.success) {
  const user = result.data; // user: User
} else {
  console.error(result.error.issues);
}

// === API integration ===
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();
  return UserSchema.parse(data); // ← runtime validation, type-safe
}

// === Branded type with Zod ===
const EmailSchema = z.email().brand<"Email">();
type Email = z.infer<typeof EmailSchema>;
// string & z.$brand<"Email">

function sendEmail(to: Email): void {
  console.log(`Sending to ${to}`);
}

const email = EmailSchema.parse("ali@test.com"); // Email
sendEmail(email);                                // ✅
// sendEmail("raw");                              // ❌ string ≠ Email

// === Transform ===
const NumberFromString = z.string().transform((v) => parseInt(v, 10));
type Input = z.input<typeof NumberFromString>;   // string (transform'dan oldin)
type Output = z.infer<typeof NumberFromString>;  // number (transform'dan keyin)
```

### Edge Cases

- **`z.input` vs `z.infer`**: `transform`/`pipe` ishlatilganda input va output type farqlanadi. Form'da `input` (raw qiymat), database'da `infer` (parsed).
- **Schema composition**: `UserSchema.extend({...})`, `UserSchema.pick({...})`, `UserSchema.omit({...})` — schema operatsiyalari TS `Pick`/`Omit` bilan parallel.
- **`.refine()` bilan custom validation**: type o'zgarmaydi, faqat runtime check (`age` `min(18)` lekin majburiy emas, `.refine()` bilan custom rule).
- **Recursive schema**: `z.lazy(() => Tree)` infinite recursion uchun. Type esa `type Tree = { name: string; children: Tree[] }` qo'lda yozish kerak (Zod inference recursive case'da chegaralangan).

### Follow-up savollar

1. "Zod bundle size katta — qaysi alternativa?" — Valibot: modular va tree-shakeable, faqat ishlatilgan validator'lar bundle'ga kiradi. Pipe API, har validator alohida import. Aniq raqam release'da o'zgaradi — Bundlephobia bilan tekshirish.
2. "tRPC bilan Zod qanday ishlaydi?" — tRPC procedure'da `.input(Schema)` bilan input validation. Schema'dan type chiqariladi — client'da to'liq type-safe.

</details>

---

### Savol 5: Zod vs Valibot vs io-ts vs ArkType — qaysi qachon? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Zod — chaining API, eng katta ecosystem (tRPC, React Hook Form, Next.js). Valibot — pipe API, tree-shakeable, bundle size kichik (frontend). io-ts — functional (fp-ts bilan). ArkType — TS type syntax'ga yaqin.

### To'liq tushuntirish

Har library trade-off'lari:

**Zod**:

- API: method chaining (`z.string().min(2)`) + top-level format'lar (`z.email()`, `z.url()` — v4'da string format'lar `z.string().email()` o'rniga top-level, eski chained forma deprecated).
- Bundle: bitta paket, har feature import qilingan vaqtda ham chained method'lar tree-shake qilinmaydi.
- Ecosystem: eng katta (tRPC, RHF, Next.js, OpenAPI generation).
- Brand: `.brand()` native support.

**Valibot**:

- API: pipe/functional (`v.pipe(v.string(), v.email())`).
- Bundle: modular — har validator alohida import. Frontend'da bundle'ga faqat ishlatilgan validator'lar qo'shiladi.
- Ecosystem: o'sib bormoqda, Zod'dan kichikroq.
- Brand: native yo'q (manual implement).

**io-ts**:

- API: functional, `fp-ts` integration (`Either`, `Option`).
- Bundle: o'rta.
- Ecosystem: functional programming community.
- Hosil: `Either<Errors, T>` (throw qilmaydi).

**ArkType**:

- API: TypeScript type syntax'ga juda yaqin (`type("string>=5")` `string` literal).
- Bundle: katta.
- Performance: schema compile time'da optimized (claim).
- Ecosystem: yangi, kichik.

Tanlash matrix:

| Loyiha | Tanlov | Sabab |
|--------|--------|-------|
| Full-stack (tRPC) | Zod | Ecosystem |
| Frontend (mobile-first) | Valibot | Bundle size |
| Functional codebase | io-ts | fp-ts integration |
| Type-first prototype | ArkType | Concise syntax |

### Kod misol

```typescript
declare const data: unknown; // external input (fetch/JSON)

// === Zod ===
import { z } from "zod";
const ZodUser = z.object({
  name: z.string().min(2),
  email: z.email(),
});
type ZodUserType = z.infer<typeof ZodUser>;

// === Valibot (tree-shakeable) ===
import * as v from "valibot";
const ValibotUser = v.object({
  name: v.pipe(v.string(), v.minLength(2)),
  email: v.pipe(v.string(), v.email()),
});
type ValibotUserType = v.InferOutput<typeof ValibotUser>;

// === io-ts (functional) ===
import * as t from "io-ts";
import { isLeft } from "fp-ts/Either";

const IoUser = t.type({
  name: t.string,
  email: t.string,
});
type IoUserType = t.TypeOf<typeof IoUser>;

const decoded = IoUser.decode(data);
if (isLeft(decoded)) {
  console.error("Invalid");
} else {
  const user: IoUserType = decoded.right;
}

// === ArkType ===
import { type } from "arktype";

const ArkUser = type({
  name: "string>=2",
  email: "string.email",
});
type ArkUserType = typeof ArkUser.infer;

const result = ArkUser(data);
if (result instanceof type.errors) {
  console.error(result.summary);
} else {
  const user: ArkUserType = result;
}
```

### Edge Cases

- **Bundle size comparison**: aniq raqamlar har release'da o'zgaradi (Bundlephobia, Pkgsize bilan tekshirish). Tree-shaking effective bundle'ga ta'sir qiladi — full Zod import vs ishlatilgan validator'lar.
- **Validation performance**: Schema compile (one-time) + per-data validation (run-time). Hot path'da benchmark kerak (loyihaga xos).
- **Error messages**: Zod default messages eng yaxshi. Valibot custom message majburiy ko'p case'da.
- **Type inference depth**: ArkType eng aniq (TS syntax mirror). Zod ham qudratli, ammo conditional/recursive case'da chegaralangan.

### Follow-up savollar

1. "Library migration (Zod → Valibot) qancha ish?" — Schema syntax to'liq farqli, manual rewrite. Codemod yo'q. Kichik schema'lar tez, katta — uzoq.
2. "Multi-validator strategy mumkinmi?" — Mumkin, lekin overhead. Boundary'ga 1 library tanlash afzal. tRPC + Zod, RHF + Zod — convention.

</details>

---

### Savol 6: Result type — `throw` o'rniga nima uchun? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Result<T, E>` (discriminated union `{ok: true, value: T} | {ok: false, error: E}`) error'ni return type'ga ko'taradi. Caller xatoni handle qilishga compile-time'da majbur bo'ladi (`throw` esa runtime'da silently propagate qilinadi).

### To'liq tushuntirish

`throw`/`catch` muammolari TypeScript'da:

1. **Error type yo'qoladi**: `try {...} catch (e) {...}` — `e: unknown` (TS 4.4+). Caller qaysi error type'ni kutishi mumkinligini bilmaydi.
2. **Implicit propagation**: function `throw` qilsa, signature'da ko'rinmaydi. Caller `catch` qilishi yodda qolmasligi mumkin.
3. **Control flow noaniq**: chained call'da xato qaysi joydan kelganini debug qilish qiyin.

`Result<T, E>` yondashuvi:

- **Explicit signature**: `function divide(a: number, b: number): Result<number, "DIVISION_BY_ZERO">` — caller error type'ni ko'radi.
- **Compile-time enforcement**: discriminated union narrowing — `if (result.ok)` bilan handle qilmasdan `result.value`'ga kira olmaysiz.
- **Composition**: `map`/`flatMap` bilan pipeline (Railway-oriented programming).

Trade-off: verbose, butun codebase'da consistent ishlatish kerak. Async + Result `Promise<Result<T, E>>` — verbose, `neverthrow`/`fp-ts` library'lar yordam beradi.

### Kod misol

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// === Compose helpers ===
function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return r.ok ? ok(fn(r.value)) : r;
}

function flatMap<T, U, E>(r: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return r.ok ? fn(r.value) : r;
}

function match<T, E, U>(
  r: Result<T, E>,
  handlers: { ok: (v: T) => U; err: (e: E) => U }
): U {
  return r.ok ? handlers.ok(r.value) : handlers.err(r.error);
}

// === Pipeline ===
type ValidationError = "EMPTY_NAME" | "INVALID_AGE";

function validateName(name: string): Result<string, ValidationError> {
  if (name.length === 0) return err("EMPTY_NAME");
  return ok(name);
}

function validateAge(age: number): Result<number, ValidationError> {
  if (age < 18 || age > 120) return err("INVALID_AGE");
  return ok(age);
}

function createUser(name: string, age: number): Result<{ name: string; age: number }, ValidationError> {
  return flatMap(validateName(name), (validName) =>
    flatMap(validateAge(age), (validAge) =>
      ok({ name: validName, age: validAge })
    )
  );
}

const result = createUser("Ali", 25);
match(result, {
  ok: (user) => console.log(`Created: ${user.name}`),
  err: (e) => {
    // e: ValidationError — exhaustive handling
    if (e === "EMPTY_NAME") console.error("Name required");
    if (e === "INVALID_AGE") console.error("Age must be 18-120");
  },
});

// === neverthrow library (production-ready) ===
import { ok as nOk, err as nErr, Result as NResult } from "neverthrow";

function divide(a: number, b: number): NResult<number, "DIV_ZERO"> {
  return b === 0 ? nErr("DIV_ZERO") : nOk(a / b);
}

divide(10, 2)
  .map((n) => n * 2)
  .match(
    (val) => console.log(`Result: ${val}`),
    (error) => console.error(error),
  );

// === Async Result ===
async function fetchUser(id: number): Promise<Result<User, "NOT_FOUND" | "NETWORK_ERROR">> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (response.status === 404) return err("NOT_FOUND");
    if (!response.ok) return err("NETWORK_ERROR");
    return ok(await response.json() as User);
  } catch {
    return err("NETWORK_ERROR");
  }
}
```

### Edge Cases

- **Result + throw aralashtirish**: agar function ichida `JSON.parse` chaqirilsa (throw), Result pattern buziladi — try/catch bilan o'rash kerak. Cohesion muhim.
- **Async Result**: `Promise<Result<T, E>>` — `await result.value` ishlamaydi (`Promise<Result>`'ni avval await qilish kerak). `neverthrow.ResultAsync` aniq abstraction.
- **Error union union**: bir nechta function chain'da error type'lar union'lashadi (`E1 | E2 | E3`). Exhaustive handle barchasini.
- **Stack trace**: `throw` stack trace'ga ega, `Result.error` faqat data. Production debugging uchun error'ga stack qo'shish kerak bo'lishi mumkin.

### Follow-up savollar

1. "Result pattern Node.js'da overkill emasmi?" — Domain logic'da foydali, glue code'da overhead. Convention: domain → Result, infrastructure (DB, HTTP) → try/catch boundary.
2. "Rust/Go Result'dan farq?" — Rust `?` operator native, TypeScript'da yo'q (manual `flatMap`/`neverthrow`). Go ikkita return value (`val, err`) — TS tuple bilan emulate qilish mumkin (`[val, null] | [null, err]`).

</details>

---

---

## Output savollar

### Savol 7: Branded types — Output [Middle]

**Savol:** Output va `typeof` natijasini ayting:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type USD = Brand<number, "USD">;

function usd(amount: number): USD {
  return amount as USD;
}

function addUSD(a: USD, b: USD): USD {
  return usd((a as number) + (b as number));
}

const total = addUSD(usd(29.99), usd(2.40));
console.log(total);
console.log(typeof total);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output: `32.39` va `number`. Branded type faqat compile-time — runtime'da `USD` oddiy `number`. `__brand` property mavjud emas (phantom).

### To'liq tushuntirish

`Brand<T, B>` `T & { readonly __brand: B }` intersection. TypeScript compile fazasida:

- `usd(29.99)` qaytadi: `29.99 as USD` — compile-time'da `USD`, runtime'da `29.99`.
- `(a as number) + (b as number)` — runtime'da `29.99 + 2.40 = 32.39`.
- `usd(...)` qayta `as USD` — compile-time'da `USD`.

Runtime'da `total` oddiy primitive `number`:

- `console.log(total)` → `32.39`.
- `typeof total` → `"number"`.

`__brand` property runtime'da yo'q — assertion `as USD` runtime'da hech narsa qo'shmaydi (type erasure).

Float precision: `29.99 + 2.40` JavaScript IEEE 754 floating point — natija aniq `32.39` (bu case'da). Boshqa qiymat (`0.1 + 0.2`) precision xatosi beradi (`0.30000000000000004`).

### Kod misol

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type USD = Brand<number, "USD">;

function usd(amount: number): USD {
  return amount as USD;
}

const price = usd(29.99);

// === Runtime ===
console.log(price);                    // 29.99
console.log(typeof price);             // "number"
console.log(Object.hasOwn(Object(price), "__brand")); // false — __brand property yo'q
// E'tibor: "__brand" in price to'g'ridan-to'g'ri ishlamaydi — `in` operatori
// primitive (number)'da TypeError beradi, shuning uchun Object(price) wrapper kerak

// === Compile-time ===
// price: USD (intersection number & { __brand: "USD" })
// IDE'da hover qilganda — USD ko'rinadi

// === Arithmetic brand'ni yo'qotadi ===
const doubled = price * 2;
// doubled: number (USD emas) — operator natijasi base type'ga aylanadi

// === Helper bilan brand saqlash ===
function multiplyUSD(amount: USD, factor: number): USD {
  return usd((amount as number) * factor);
}

const doubledTyped = multiplyUSD(price, 2);
// doubledTyped: USD ✅
```

### Edge Cases

- **`Object.keys(branded)` runtime'da boshlang'ich primitive bo'yicha**: `Object.keys(usd(1))` → `[]` (primitive number'da key yo'q). `__brand`'ni izlash mantiqsiz.
- **Serialization**: `JSON.stringify(price)` → `"29.99"` (number serialized). `JSON.parse` qaytarganda raw `number` — `createUSD` bilan qayta brand qilish kerak.
- **`instanceof`**: branded primitive'da ishlamaydi (`instanceof` faqat objects/functions). Class-based ID uchun ishlatish mumkin.
- **DevTools**: console'da branded primitive farqlanmaydi — debug'da brand info yo'q.

### Follow-up savollar

1. "Branded type runtime'da ham property qo'shsa bo'ladimi?" — Object-based brand'da mumkin: `class USD { constructor(public value: number) {} }`. Lekin verbose, serialization murakkab.
2. "Production'da arithmetic operatsiya brand'ni saqlashga universal yechim?" — Yo'q. Har operatsiya uchun helper (`addUSD`, `multiplyUSD`). Yoki class-based money library (`big.js`, `dinero.js`).

</details>

---

---

## Bug fix savollar

### Savol 8: Reducer exhaustive — bug fix [Middle]

**Savol:** Xatoni toping va `assertNever` bilan tuzating. Yangi action qo'shilsa compile-time'da error berishi kerak.

```typescript
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Xato: `"RESET"` case handled emas, function `undefined` qaytarishi mumkin (return type `number` deb e'lon qilingan). `assertNever` bilan default branch qo'shib, future-proof exhaustive check.

### To'liq tushuntirish

Original kodda:

1. `"INCREMENT"` va `"DECREMENT"` handled.
2. `"RESET"` action kelsa — switch hech qaysi case'ga tushmaydi, function `undefined` qaytaradi.
3. TS `noImplicitReturns: true` bo'lsa — error: "Not all code paths return a value". Bo'lmasa — silent bug.

To'g'rilash:

- `"RESET"` case qo'shish (state'ni 0'ga qaytarish — domain logic).
- `default` branch'da `assertNever` — yangi action qo'shilganda compile error.

`noImplicitReturns: true` `strict`'ga **kirmaydi** — yangi loyihalarda yoqish tavsiya.

### Kod misol

```typescript
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" };

function assertNever(value: never): never {
  throw new Error(`Unhandled action: ${JSON.stringify(value)}`);
}

// === ✅ To'g'rilangan ===
function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
    case "RESET":     return 0;
    default:          return assertNever(action);
  }
}

// === Yangi action qo'shilsa ===
type ActionV2 =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" }
  | { type: "SET"; value: number }; // ← yangi

function reducerV2(state: number, action: ActionV2): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
    case "RESET":     return 0;
    default:          return assertNever(action);
    // ❌ Argument of type '{ type: "SET"; value: number }'
    //    is not assignable to parameter of type 'never'
  }
}

// === Fix ===
function reducerV2Fixed(state: number, action: ActionV2): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
    case "RESET":     return 0;
    case "SET":       return action.value;
    default:          return assertNever(action);
  }
}

// === useReducer (React) bilan ===
import { useReducer } from "react";

function Counter() {
  const [state, dispatch] = useReducer(reducerV2Fixed, 0);

  return null; // ...
  // dispatch({ type: "INCREMENT", amount: 1 }); // ✅
  // dispatch({ type: "UNKNOWN" });               // ❌ TS error
}
```

### Edge Cases

- **`noImplicitReturns`**: bu flag yoqilmagan bo'lsa, original code TS error bermaydi (silent bug). Always enable in production.
- **Production error handling**: `assertNever` throw qiladi — production'da log + fallback kerak bo'lishi mumkin. Wrapper helper (`logAndDefault`).
- **Default state**: Redux pattern'da `default: return state;` — silent default. Exhaustive check'siz xavfli (yangi action ignore qilinadi). `assertNever` afzal.
- **`Action` type external (Redux Toolkit)**: `createSlice` generated action type'lar — har `case` `createSlice.actions.X.match(action)` orqali narrow qilinadi. RTK exhaustive built-in.

### Follow-up savollar

1. "Redux Toolkit `createSlice` exhaustive ta'minlaydi?" — Builder pattern (`builder.addCase(...)`) — har action explicit handled. Default builder error bermaydi, lekin missing case TS warning.
2. "XState bilan exhaustive qanday?" — State machine guard'lar va action'lar typed — TS narrowing avtomatik. State combination'lar compile-time'da tekshiriladi.

</details>

---

### Savol 9: `DeepValues` — Output [Middle+]

**Savol:** `RoutePath` type nima? Qaysi assignment compile bo'ladi?

```typescript
const ROUTES = {
  home: "/",
  users: { list: "/users", detail: "/users/:id" },
  posts: { list: "/posts", detail: "/posts/:id" },
} as const;

type DeepValues<T> = T extends object
  ? { [K in keyof T]: DeepValues<T[K]> }[keyof T]
  : T;

type RoutePath = DeepValues<typeof ROUTES>;

const x: RoutePath = "/users/:id"; // ?
const y: RoutePath = "/about";     // ?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`RoutePath` = `"/" | "/users" | "/users/:id" | "/posts" | "/posts/:id"`. `x` valid (union'da bor), `y` compile error (`"/about"` union'da yo'q).

### To'liq tushuntirish

`DeepValues<T>` recursive conditional type:

1. **Base case**: `T` object emas — `T` ni qaytaradi (literal qiymat).
2. **Recursive case**: `T` object — har property uchun `DeepValues<T[K]>` chaqiriladi va indexed access (`[keyof T]`) bilan distribute qilinadi.

`as const` `ROUTES`'ni deeply readonly literal qiladi:

```typescript
typeof ROUTES = {
  readonly home: "/";
  readonly users: { readonly list: "/users"; readonly detail: "/users/:id" };
  readonly posts: { readonly list: "/posts"; readonly detail: "/posts/:id" };
};
```

Recursion:

1. `DeepValues<typeof ROUTES>`:
   - `home: "/"` — string literal, base case, qaytadi `"/"`.
   - `users: {...}` — object, recurse.
   - `posts: {...}` — object, recurse.
   - `[keyof T]` distribute: `"/" | DeepValues<{...users}> | DeepValues<{...posts}>`.
2. `DeepValues<{list: "/users"; detail: "/users/:id"}>`:
   - `list: "/users"` → `"/users"`.
   - `detail: "/users/:id"` → `"/users/:id"`.
   - `[keyof T]` → `"/users" | "/users/:id"`.
3. Similar `posts` → `"/posts" | "/posts/:id"`.
4. Final union: `"/" | "/users" | "/users/:id" | "/posts" | "/posts/:id"`.

`y = "/about"` union'da yo'q → compile error.

### Kod misol

```typescript
const ROUTES = {
  home: "/",
  users: { list: "/users", detail: "/users/:id" },
  posts: { list: "/posts", detail: "/posts/:id" },
} as const;

type DeepValues<T> = T extends object
  ? { [K in keyof T]: DeepValues<T[K]> }[keyof T]
  : T;

type RoutePath = DeepValues<typeof ROUTES>;
// "/" | "/users" | "/users/:id" | "/posts" | "/posts/:id"

const x: RoutePath = "/users/:id"; // ✅
// const y: RoutePath = "/about";
// ❌ Type '"/about"' is not assignable to type 'RoutePath'

// === Type-safe navigation ===
function navigate(path: RoutePath): void {
  console.log(`Navigating to ${path}`);
}

navigate("/users");        // ✅
navigate(ROUTES.posts.detail); // ✅ — "/posts/:id"
// navigate("/random");    // ❌

// === Real-world router config ===
type Router = {
  [K in keyof typeof ROUTES]: typeof ROUTES[K];
};
// Router'da har key explicit ko'rinadi
```

### Edge Cases

- **`as const` siz**: `home: "/"` `string` literal'i yo'qoladi — `RoutePath` `string` bo'lib qoladi.
- **Array elements**: `routes: ["/a", "/b"] as const` — `DeepValues` array element'larini ham union qiladi (array `keyof` `"0" | "1" | "length" | ...` ammo TS filter qiladi).
- **Function values**: `{handler: () => {}}` — `DeepValues<() => {}>` function ham object, recursive ishlamaydi. Generic guard kerak.
- **`Symbol` keys**: `as const` symbol key'larni include qilmaydi (string key'lar).

### Follow-up savollar

1. "TypeScript'da recursive type limit bormi?" — Ha. Juda chuqur yoki cheksiz recursion `TS2589: Type instantiation is excessively deep and possibly infinite` error beradi. Instantiation depth limiti modern TS'da 100 (compiler ichida `instantiationDepth === 100` tekshiruvi; tarix: 50 → 500 PR #45025'da → 100 PR #45711'da tail-recursion bilan). Yechim: recursive type'ni kichik qismlarga bo'lish, conditional type'ni tail-recursive yozish, yoki counter type bilan depth cheklash (`type Paths<T, Depth extends number[] = []>`).
2. "DeepKeys (path string) qanday qilinadi?" — Template literal types bilan:
```typescript
type Paths<T> = T extends object
  ? { [K in keyof T]-?: K extends string
      ? T[K] extends object ? `${K}` | `${K}.${Paths<T[K]>}` : `${K}`
      : never
    }[keyof T]
  : never;
```

</details>

---

### Savol 10: Zod validation yo'qligi — bug fix [Middle+]

**Savol:** Bu kodda runtime safety yo'q. Schema bor lekin ishlatilmagan. Tuzating.

```typescript
import { z } from "zod";

const ProductSchema = z.object({
  name: z.string(),
  price: z.number().positive(),
  inStock: z.boolean(),
});
type Product = z.infer<typeof ProductSchema>;

async function fetchProduct(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  const data = await response.json();
  return data;
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`response.json()` `Promise<any>` qaytaradi — runtime'da nima kelishini bilmaymiz. Schema mavjud, ammo `parse`/`safeParse` chaqirilmagan. Type assertion (`return data`) compile'da o'tadi, lekin runtime'da invalid data `Product`'ga "majburlanadi".

### To'liq tushuntirish

Original kodda problem:

1. `data: any` (`fetch().json()` default).
2. `return data` `any` → `Product` o'tadi (TS `any`'ni har joyga assignable hisoblaydi).
3. Runtime'da API `{name: 123, price: -5}` qaytarsa — `Product` type'ga ko'ra `name: string`, `price: positive` kutiladi, lekin runtime'da xato.
4. `product.name.toUpperCase()` chaqirilganda crash (`123.toUpperCase` yo'q).

Yechim — schema'ni runtime'da ishlatish:

1. `data: unknown` explicit yozish — type-safety boundary.
2. `ProductSchema.parse(data)` — invalid bo'lsa `throw`, valid bo'lsa `Product`.
3. Yoki `safeParse(data)` — `{success, data, error}` (production'da afzal).

### Kod misol

```typescript
import { z } from "zod";

const ProductSchema = z.object({
  name: z.string(),
  price: z.number().positive(),
  inStock: z.boolean(),
});
type Product = z.infer<typeof ProductSchema>;

// === ✅ Yechim 1 — parse (throw on invalid) ===
async function fetchProductV1(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  const data: unknown = await response.json();
  return ProductSchema.parse(data); // ✅ runtime validation
}

// === ✅ Yechim 2 — safeParse (Result-like) ===
type FetchError = "NETWORK" | "INVALID_DATA" | "NOT_FOUND";
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

async function fetchProductV2(id: number): Promise<Result<Product, FetchError>> {
  try {
    const response = await fetch(`/api/products/${id}`);
    if (response.status === 404) return { ok: false, error: "NOT_FOUND" };
    if (!response.ok) return { ok: false, error: "NETWORK" };

    const data: unknown = await response.json();
    const result = ProductSchema.safeParse(data);

    if (result.success) {
      return { ok: true, value: result.data };
    }
    return { ok: false, error: "INVALID_DATA" };
  } catch {
    return { ok: false, error: "NETWORK" };
  }
}

// Ishlatish:
const fetchResult = await fetchProductV2(42);
if (fetchResult.ok) {
  console.log(fetchResult.value.name); // ✅ type-safe Product
} else {
  console.error(`Error: ${fetchResult.error}`);
}

// === ✅ Yechim 3 — neverthrow integration ===
import { ResultAsync } from "neverthrow";

function fetchProductV3(id: number): ResultAsync<Product, Error> {
  return ResultAsync.fromPromise(
    fetch(`/api/products/${id}`).then((r) => r.json()),
    (e) => new Error(`Fetch failed: ${e}`)
  ).map((data: unknown) => ProductSchema.parse(data));
}
```

### Edge Cases

- **`unknown` boundary**: `response.json()` har doim `unknown`'ga cast qilish kerak. `as Product` bypass — never do.
- **`safeParse` performance**: complex schema (deep nested) parse'i lenta. Hot path'da benchmark, caching mumkin.
- **Partial validation**: faqat critical field'larni validate qilish strategy (boshqalarini permissive). `.passthrough()` bilan extra field'larni o'tkazib yuborish.
- **Error logging**: `result.error.issues` har validation xatosi (`path`, `message`, `code`). Production'da Sentry'ga log.

### Follow-up savollar

1. "Schema validation har request'da overhead qancha?" — Schema murakkabligi va data hajmiga bog'liq. Aksariyat case'da negligible, hot path'da benchmark + caching.
2. "Client-side va server-side schema sharing?" — Monorepo'da `packages/schemas`. tRPC pattern — server schema'lari client'da type sifatida ishlatiladi (zero runtime cost client'da).

</details>

---

---

## Amaliy savollar (Coding)

### Savol 11: Phantom Type Builder — kompleks generic [Senior]

**Savol:** `TypeSafeBuilder<T>` ni implement qiling — `.build()` faqat barcha required field'lar `.set()` qilinganda compile bo'lsin. Optional field'lar majburiy emas.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic type parameter `Provided extends keyof T = never` orqali "berilgan" field'larni track qilish. `.set(key, value)` yangi `Provided | K` qaytaradi. `.build()` `this` constraint orqali `RequiredKeys<T> extends Provided` tekshiradi.

### To'liq tushuntirish

Builder pattern type-safe variant'ida:

1. **`RequiredKeys<T>`** utility — `T`'dagi required (non-optional) key'larni topadi. `[K in keyof T]-?: undefined extends T[K] ? never : K` — optional field'lar `undefined`'ni include qiladi, `extends` filter.
2. **`TypeSafeBuilder<T, Provided>`** — `Provided` "qachongacha berilgan key'lar" tracking. Default `never` (hech narsa berilmagan).
3. **`.set<K>(key, value)`** — yangi `Provided | K` builder qaytaradi. Type narrowing chain orqali har `.set()` provided'ni kengaytiradi.
4. **`.build()`** — `this` parameter constraint: `RequiredKeys<T> extends Provided`. Agar barcha required key'lar `Provided`'da bo'lsa — `this` `TypeSafeBuilder<T, Provided>`'ga assignable. Yo'q bo'lsa — `never`, build chaqirib bo'lmaydi.

`this` parameter constraint — TS feature, method'ni faqat ma'lum condition'da chaqirish imkonini beradi.

### Kod misol

```typescript
type RequiredKeys<T> = {
  [K in keyof T]-?: undefined extends T[K] ? never : K;
}[keyof T];

class TypeSafeBuilder<T, Provided extends keyof T = never> {
  private data: Partial<T> = {};

  set<K extends keyof T>(
    key: K,
    value: T[K]
  ): TypeSafeBuilder<T, Provided | K> {
    (this.data as Record<string, unknown>)[key as string] = value;
    return this as unknown as TypeSafeBuilder<T, Provided | K>;
  }

  build(
    this: RequiredKeys<T> extends Provided ? TypeSafeBuilder<T, Provided> : never
  ): T {
    return { ...this.data } as T;
  }
}

// === Ishlatish ===
interface DbConfig {
  host: string;       // required
  port: number;       // required
  database: string;   // required
  username?: string;  // optional
  ssl?: boolean;      // optional
}

// ✅ Barcha required berilgan
const cfg1 = new TypeSafeBuilder<DbConfig>()
  .set("host", "localhost")
  .set("port", 5432)
  .set("database", "mydb")
  .build(); // ✅

// ✅ Optional ham berildi
const cfg2 = new TypeSafeBuilder<DbConfig>()
  .set("host", "localhost")
  .set("port", 5432)
  .set("database", "mydb")
  .set("ssl", true)
  .build(); // ✅

// ❌ Required field yo'q
// const cfg3 = new TypeSafeBuilder<DbConfig>()
//   .set("host", "localhost")
//   .build();
// Error: 'this' context of type 'TypeSafeBuilder<DbConfig, "host">'
//        is not assignable to method's 'this' of type 'never'
// Sabab: RequiredKeys<DbConfig> = "host" | "port" | "database", Provided = "host"
//        "host" | "port" | "database" extends "host" → false → this type 'never'

// ❌ Type mismatch
// new TypeSafeBuilder<DbConfig>()
//   .set("port", "5432") // ❌ Type 'string' not assignable to 'number'

// === Step builder alternative ===
interface DbConfigStepBuilder {
  setHost(host: string): DbConfigStepBuilder;
  setPort(port: number): DbConfigStepBuilderWithPort;
}

interface DbConfigStepBuilderWithPort {
  setDatabase(db: string): DbConfigStepBuilderComplete;
}

interface DbConfigStepBuilderComplete {
  setUsername(u: string): DbConfigStepBuilderComplete;
  setSsl(ssl: boolean): DbConfigStepBuilderComplete;
  build(): DbConfig;
}
// Verbose — har step alohida interface
```

### Edge Cases

- **`RequiredKeys<T>`'da `undefined extends T[K]`**: optional field default'da `undefined` include qiladi. `strictNullChecks` yoqilmaganda hammasi `undefined` include qiladi — utility ishlamaydi.
- **`Provided` runtime'da yo'q**: faqat compile-time tracking. Runtime'da `set` har qanday key'ni qabul qiladi (kafolat yo'q).
- **Method chaining'da `this` re-assertion**: `this as unknown as ...` — runtime'da hech narsa, faqat compile-time type narrowing.
- **Generic constraint complex**: `T` deep nested bo'lsa, `RequiredKeys` faqat top-level. Nested required uchun recursive utility kerak.

### Follow-up savollar

1. "Step builder vs phantom type builder — qaysi afzal?" — Step builder har step explicit interface (verbose, ammo o'qish oson). Phantom type — kuchli, ammo generic murakkab. Library author tanloviga bog'liq.
2. "TypeScript 5.x'da yangi builder API'lar bormi?" — Native yo'q, lekin `const` type parameter (`<const T>`) inference'ni yaxshiladi. Builder'da `as const` kerak emas ko'p case'da.

<details>
<summary><strong>Deep Dive</strong></summary>

**`this` parameter constraint mexanizmi.** TS'da method'ning `this` parameter'i type checker tomonidan call-site'da method receiver type'i bilan tekshiriladi. `build(this: RequiredKeys<T> extends Provided ? TypeSafeBuilder<T, Provided> : never): T` — agar conditional `never`'ga aylansa, `this`'ning kutilgan type'i `never`. Builder instance `never`'ga assignable emas (faqat `never` o'zi `never`'ga assignable) — call rad etiladi. Bu `[[Call]]`'dan oldingi assignability check.

**`Partial<T>` runtime ↔ compile-time mapping.** `private data: Partial<T> = {}` — compile-time'da har key optional, runtime'da object o'sib boradi. `(this.data as Record<string, unknown>)[key as string] = value` — generic indexing'ni runtime'da bypass qiladi (TS strict mode'da `Partial<T>[K]` directly assign mumkin emas, `K` parameterized).

**Conditional type distribution.** `RequiredKeys<T>`'da `[K in keyof T]-?` mapped type'da `-?` modifier optional flag'ni olib tashlaydi. Keyin `undefined extends T[K] ? never : K` — `T[K]` original (optional bo'lsa `T[K] | undefined`). `[keyof T]` indexed access barcha `K` qiymatlarni union qiladi, `never` filter qilinadi (`never` union'da yo'qoladi).

**Soundness chegarasi.** Phantom builder kafolatlanmagan invariant'ga tayanadi: `set` har doim `Provided | K` qaytaradi, lekin runtime'da bir xil instance. Agar user `as TypeSafeBuilder<T, never>` cast qilsa — protection bypass. Library author lint rule (`@typescript-eslint/consistent-type-assertions`) yoki encapsulation (constructor private + factory function) qo'shadi.

**Builder ↔ Object literal trade-off.** Object literal `satisfies` bilan deyarli bir xil safety beradi (`{} satisfies Required<T>`), lekin builder pattern step-by-step API beradi (IDE autocomplete har step'da progressive). Library API tanlovi — builder fluent chain, literal — config-style.

</details>

</details>

---

### Savol 12: Email branded type + Zod integration [Senior]

**Savol:** `Email` branded type + constructor `createEmail` + Zod schema'dan `EmailSchema` yaratish. Faqat validated email qabul qiladigan `sendEmail` function.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Brand<string, "Email">` orqali `Email` type. `createEmail(value: string): Email` constructor validation bilan. Zod variant: `z.email().brand<"Email">()` — runtime + compile-time brand birga.

### To'liq tushuntirish

Ikki strategiya:

**1. Manual brand**: phantom property + constructor function.

- `type Email = string & { readonly __brand: "Email" }`.
- `createEmail(v: string)`: validate, return `v as Email`.
- `sendEmail(to: Email)`: faqat constructor'dan o'tgan qiymat qabul qiladi.

**2. Zod brand**: schema'ga brand qo'shish.

- `z.email().brand<"Email">()` — runtime regex + compile-time brand.
- `.parse(v)` → `Email` (validated).

**3. `unique symbol` brand**: collision-free nominal typing.

- `declare const EmailTag: unique symbol; type Email = string & { [EmailTag]: void }`.
- Har declaration unique — boshqa `__brand: "Email"` bilan to'qnashmaydi.

### Kod misol

```typescript
import { z } from "zod";

// === Strategiya 1: Manual brand ===
type Brand<T, B extends string> = T & { readonly __brand: B };
type Email = Brand<string, "Email">;

function createEmail(value: string): Email {
  // Production'da yaxshi regex yoki email validation library
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(value)) {
    throw new Error(`Invalid email: ${value}`);
  }
  return value as Email;
}

function sendEmail(to: Email, subject: string, body: string): void {
  console.log(`Sending to ${to}: ${subject}`);
}

const email = createEmail("ali@test.com"); // ✅ Email
sendEmail(email, "Hello", "Body");          // ✅
// sendEmail("raw@string.com", "x", "y");   // ❌ string ≠ Email
// sendEmail("invalid", "x", "y");           // ❌ string ≠ Email

// === Strategiya 2: unique symbol (collision-free) ===
declare const EmailTag: unique symbol;
type EmailStrong = string & { readonly [EmailTag]: void };

function createEmailStrong(value: string): EmailStrong {
  if (!value.includes("@")) throw new Error("Invalid");
  return value as EmailStrong;
}

// `__brand: "Email"` va `__brand: "Phone"` — agar string match qilsa, collision
// unique symbol har declare uchun unique — collision yo'q

// === Strategiya 3: Zod integration (recommended) ===
const EmailSchema = z.email().brand<"Email">();
type ZodEmail = z.infer<typeof EmailSchema>;
// string & z.$brand<"Email">

function createZodEmail(value: string): ZodEmail {
  return EmailSchema.parse(value); // throw qiladi invalid'da
}

function safeCreateZodEmail(value: string): ZodEmail | null {
  const result = EmailSchema.safeParse(value);
  return result.success ? result.data : null;
}

const zodEmail = createZodEmail("ali@test.com");
// zodEmail: string & z.$brand<"Email">

// === Combine: User schema with branded fields ===
const UserIdSchema = z.number().int().positive().brand<"UserId">();
type UserId = z.infer<typeof UserIdSchema>;

const UserSchema = z.object({
  id: UserIdSchema,
  email: EmailSchema,
  name: z.string().min(2),
});
type User = z.infer<typeof UserSchema>;
// {
//   id: number & z.$brand<"UserId">;
//   email: string & z.$brand<"Email">;
//   name: string;
// }

async function fetchUser(id: UserId): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();
  return UserSchema.parse(data); // ✅ full runtime + brand validation
}
```

### Edge Cases

- **Brand'ni JSON serialize**: `JSON.stringify(email)` → raw string. Parse'da qayta brand qilish kerak (`EmailSchema.parse(JSON.parse(...))`).
- **`z.brand()` chaining**: `.brand<"X">().brand<"Y">()` — second brand intersection'ga qo'shiladi (`string & z.$brand<"X"> & z.$brand<"Y">`). Mantiqsiz, faqat birini ishlatish.
- **Generic over branded**: `function isEmail<T extends Email>(v: T): boolean` — `T extends Email` faqat `Email` accept qiladi (raw string yo'q).
- **Database storage**: Branded type DB'da raw string sifatida saqlanadi. ORM (Prisma, Drizzle) brand'ni saqlamaydi — query result'ni manual brand qilish kerak.

### Follow-up savollar

1. "Branded type test'da qanday mock qilish?" — `as Email` faqat test'da OK (faktoring trick). Yoki `createEmail` test util'ga export.
2. "Brand validation logic'ni qayerda yozish?" — Schema'da (Zod) — runtime'da har joyda ishlaydi. Manual brand — constructor function'da. Library author tanlovi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Zod brand implementation.** `z.email().brand<"Email">()` — schema'ning output type'iga `z.$brand<"Email">` intersection qo'shadi. Zod v4'da brand marker'i `$brand` unique symbol orqali ifodalanadi:

```typescript
// Zod v4 brand marker (soddalashtirilgan ko'rinish)
declare const $brand: unique symbol;
type Branded<B extends string | number | symbol> = { [$brand]: { [k in B]: true } };
```

Runtime'da brand faqat type-level marker — `parse` qaytaradigan qiymat oddiy string, hech qanday qo'shimcha property yo'q. Compile-time'da `z.infer` `string & z.$brand<"Email">` chiqaradi — bu intersection raw string'ni branded type'ga assign qilishni cheklaydi.

**`unique symbol` vs string brand collision.** `Brand<string, "Email">` ikki module'da bir xil literal `"Email"` ishlatilsa — TS strukturally bir xil deb hisoblaydi (intersection brand key bir xil). `declare const EmailTag: unique symbol` — har declaration unique tag yaratadi (TS spec: `unique symbol` har declaration uchun nominal identity). Kross-module brand uchun `unique symbol` xavfsizroq.

**Type erasure va runtime check ajralishi.** Manual brand'da `createEmail` validation logic constructor ichida — bypass qilish mumkin (`value as Email`). Zod brand'da `.parse()` har joyda ishlatilsa — runtime'da har bir Email instance schema'dan o'tgan. "Single source of truth" mantiqi — schema bo'yicha runtime + compile-time bir xil kafolat.

**Database integration soundness gap.** ORM (Prisma, Drizzle) generated type'lar brand'ni saqlamaydi — DB'dan kelgan `string`'ni `Email` deb assign qilish bypass'ga yo'l ochadi. Solution: repository layer'da `EmailSchema.parse(row.email)` har query result uchun. Yoki Drizzle custom type (`.$type<Email>()`) — compile-time brand, ammo runtime validation alohida.

**Generic over branded constraint.** `function isEmail<T extends Email>(v: T): boolean` — `T extends Email` faqat branded'ni qabul qiladi. Lekin `T extends string`'da `T = "raw@string.com"` literal — branded yo'q. Branded constraint kuchli boundary belgilaydi.

</details>

</details>

---

## Xulosa

- Branded types — phantom property orqali nominal typing, `UserId ≠ PostId` compile-time'da. `unique symbol` collision-free
- Exhaustive matching — `switch` + `assertNever(x: never)` yangi variant qo'shilganda compile error
- `as const satisfies T` — literal narrowing + type constraint birga (type annotation literal'ni yo'qotadi)
- Zod `z.infer<typeof Schema>` — schema'dan TS type chiqarish, single source of truth
- Zod vs Valibot vs io-ts vs ArkType — bundle/ecosystem/API style trade-off
- Result type — `{ok: true, value: T} | {ok: false, error: E}` — error caller'ga compile-time'da majbur
- Branded type arithmetic'da yo'qoladi — har operatsiya uchun helper function
- Reducer exhaustive — `default: assertNever(action)` future-proof
- Phantom type builder — `Provided extends keyof T` tracking, `.build()` constraint
- Zod `.brand()` + schema — runtime validation + compile-time nominal birga
- Recursive utility (`DeepValues`) — `as const` bilan literal types tree'dan union yig'ish
- API boundary'da har doim `unknown` → schema parse → typed (har joyda `any` taqiqlanadi)
