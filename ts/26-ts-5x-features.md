# Bo'lim 26: TypeScript 5.x Yangiliklari

> TypeScript 5.x seriyasi — tilning eng tez rivojlangan davri. Har bir minor release yangi type-level imkoniyatlar, developer experience yaxshilanishlar, va runtime standartlariga moslashish olib keldi. Har feature uchun: nima o'zgardi, oldin qanday edi, hozir qanday, va amaliy misollar. Ba'zi feature lar boshqa bo'limlarda chuqur yoritilgan — ular uchun qisqacha recap va cross-reference beriladi.

---

## Mundarija

- [TypeScript 5.0](#typescript-50)
- [TypeScript 5.1](#typescript-51)
- [TypeScript 5.2](#typescript-52)
- [TypeScript 5.3](#typescript-53)
- [TypeScript 5.4](#typescript-54)
- [TypeScript 5.5](#typescript-55)
- [TypeScript 5.6](#typescript-56)
- [TypeScript 5.7](#typescript-57)
- [TypeScript 5.8](#typescript-58)
- [Version Selection Guide](#version-selection-guide)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## TypeScript 5.0

### Nazariya

TS 5.0 — TC39 decorators, `const` type parameters, va bundler-friendly module resolution.

**TC39 Stage 3 Decorators** — yangi ECMAScript standart. `experimentalDecorators` kerak emas. Batafsil [Bo'lim 19](19-decorators.md).

**`const` type parameters** — generic da `as const` behavior:

```typescript
function routes<const T extends readonly { path: string }[]>(r: T) { return r; }
const r = routes([{ path: "/" }, { path: "/about" }]);
// r: readonly [{ readonly path: "/" }, { readonly path: "/about" }]
// const keyword siz: { path: string }[]
```

**`moduleResolution: "bundler"`** — Vite/Webpack muhiti uchun. Extension-less imports + `exports` field support. Batafsil [Bo'lim 17](17-modules.md).

**`verbatimModuleSyntax`** — `import type` majburiy, predictable emit. Batafsil [Bo'lim 17](17-modules.md#verbatimmodulesyntax).

**All enums are union enums** — enum member lar union type sifatida ishlaydi.

---

## TypeScript 5.1

### Nazariya

**Implicit undefined return** — `undefined` qaytaradigan funksiyalarda explicit `return undefined` yozish kerak emas:

```typescript
// TS 5.0 da:
function setName(name: string): undefined {
  this.name = name;
  return undefined; // Kerak edi
}

// TS 5.1 da:
function setName(name: string): undefined {
  this.name = name;
  // return undefined — KERAK EMAS
}
```

**Unrelated getter/setter types** — getter va setter turli type lar qaytarishi mumkin:

```typescript
class Container {
  #value: unknown;

  get value(): unknown { return this.#value; }

  set value(val: string | number | boolean) {
    this.#value = val;
  }
}
```

---

## TypeScript 5.2

### Nazariya

**`using` / `await using` — Explicit Resource Management** (TC39). Resource larni avtomatik tozalash — `try/finally` o'rniga `using` keyword.

```typescript
// TS 5.2+ — auto cleanup
function processFile(path: string) {
  using handle = openFile(path);
  // handle scope tugaganda avtomatik close bo'ladi
  return handle.read();
  // handle[Symbol.dispose]() avtomatik chaqiriladi
}

// Async version
async function fetchData() {
  await using conn = await connectDB();
  return conn.query("SELECT * FROM users");
  // conn[Symbol.asyncDispose]() avtomatik chaqiriladi
}
```

**`Symbol.dispose` va `Symbol.asyncDispose`** — resource class larga implement qilinadi:

```typescript
class FileHandle implements Disposable {
  constructor(private fd: number) {}

  read() { return "file content"; }

  [Symbol.dispose]() {
    closeFile(this.fd);
    console.log("File closed");
  }
}

function openFile(path: string): FileHandle {
  return new FileHandle(42);
}
```

**Decorator Metadata** — `Symbol.metadata` orqali native metadata. Batafsil [Bo'lim 19](19-decorators.md#decorator-metadata--symbolmetadata).

---

## TypeScript 5.3

### Nazariya

**Import Attributes** — import da metadata berish:

```typescript
import config from "./config.json" with { type: "json" };
// Runtime ga bu JSON ekanini aytadi
```

**`switch(true)` narrowing** — `switch(true)` da condition bilan type narrowing:

```typescript
function classify(x: string | number | boolean) {
  switch (true) {
    case typeof x === "string":
      return x.toUpperCase(); // x: string ✅
    case typeof x === "number":
      return x.toFixed(2); // x: number ✅
    default:
      return String(x); // x: boolean
  }
}
```

---

## TypeScript 5.4

### Nazariya

**`NoInfer<T>`** — inference bloklash. Batafsil [Bo'lim 15](15-utility-types.md#noinfert-ts-54).

```typescript
function createStore<T>(initial: T, fallback: NoInfer<T>): T {
  return initial ?? fallback;
}

createStore(42, "hello"); // ❌ — T faqat 42 dan infer (number), "hello" mos emas
createStore(42, 0);       // ✅ — T = number
```

**Preserved narrowing in closures** — closure ichida **last assignment** dan keyin narrowing saqlanadi:

```typescript
function example() {
  let value: string | number;

  value = "hello"; // last assignment
  setTimeout(() => {
    console.log(value.toUpperCase()); // ✅ TS 5.4+ da: string
    // TS 5.3 da: string | number (narrowing yo'qolardi)
  }, 100);
}
```

**`Object.groupBy` va `Map.groupBy`** — ES2024 type lar:

```typescript
const users = [
  { name: "Ali", role: "admin" },
  { name: "Vali", role: "user" },
  { name: "Gani", role: "admin" },
];

const grouped = Object.groupBy(users, u => u.role);
// { admin: [Ali, Gani], user: [Vali] }
```

---

## TypeScript 5.5

### Nazariya

**`isolatedDeclarations`** — fayl-bo'yicha declaration emit. Batafsil [Bo'lim 18](18-declaration-files.md#isolateddeclarations-ts-55).

**Inferred type predicates** — `filter` callback da avtomatik type narrowing:

```typescript
const values = [1, null, 2, undefined, 3];

// TS 5.4 da:
const nums = values.filter((v): v is number => v != null);

// TS 5.5 da — avtomatik infer:
const nums = values.filter(v => v != null);
// nums: number[] — TS avtomatik type predicate infer qildi!
```

---

## TypeScript 5.6

### Nazariya

**Always-truthy/nullish checks** — doim truthy yoki doim nullish bo'lgan expression lar uchun ogohlantirish:

```typescript
function process(x: string) {
  if (x) { /* ... */ }
  // TS 5.6: Warning — x doim truthy (string, bo'sh emas)

  if (/regex/) { /* ... */ }
  // TS 5.6: Warning — regex doim truthy!
}
```

**Iterator helpers** — `Iterator.prototype` method lari:

```typescript
function* numbers() { let i = 0; while (true) yield i++; }

const first10Even = numbers()
  .filter(n => n % 2 === 0)
  .take(10)
  .toArray();
// [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

---

## TypeScript 5.7

### Nazariya

**Path rewriting** — `moduleResolution: "node16"` da import specifier larni `.ts` → `.js` ga avtomatik rewrite:

```typescript
// tsconfig: "rewriteRelativeImportExtensions": true
import { utils } from "./utils.ts";
// Compiled JS: import { utils } from "./utils.js";
```

**`es2024` target** — `Symbol.dispose`, `Object.groupBy`, `Promise.withResolvers` type lari.

---

## TypeScript 5.8

### Nazariya

**`erasableSyntaxOnly`** — faqat **type erasure** bilan o'chiriladigan syntax ruxsat. `enum`, `namespace`, `parameter properties` — TAQIQLANADI. Node.js ning `--experimental-strip-types` flag iga mo'ljallangan.

```json
{ "compilerOptions": { "erasableSyntaxOnly": true } }
```

```typescript
// ❌ — erasableSyntaxOnly da TAQIQLANADI:
enum Color { Red, Green }           // ❌ — runtime kod hosil qiladi
namespace Utils { export const x = 1; } // ❌ — runtime IIFE hosil qiladi
class A { constructor(public name: string) {} } // ❌ — parameter property

// ✅ — ruxsat:
type Color = "red" | "green";       // ✅ — type erasure
const Utils = { x: 1 };             // ✅ — oddiy JS
class A { name: string; constructor(name: string) { this.name = name; } } // ✅
```

**Granular branch checks** — `if/else` branch larda aniqroq type narrowing.

---

## Version Selection Guide

### Nazariya

| Loyiha turi | Tavsiya etilgan TS versiya | Sabab |
|-------------|--------------------------|-------|
| Yangi loyiha | **5.7+** | Eng so'nggi feature lar |
| Node.js 22+ (native TS) | **5.8+** | `erasableSyntaxOnly` |
| Angular | **5.4-5.6** | Angular TS versiya cheklovi |
| Legacy maintenance | **5.0+** | Minimal yangilash |

**Migration qoida:** Har 2-3 oyda minor versiya yangilash. Major breaking change lar kam — odatda smooth upgrade.

---

## Edge Cases va Gotchas

### 1. `const` type parameter — har joyda `readonly` bo'ladi

```typescript
function getConfig<const T>(config: T) { return config; }
const cfg = getConfig({ host: "localhost", port: 3000 });
// cfg: { readonly host: "localhost"; readonly port: 3000 }
// ⚠️ cfg.host = "other"; // ❌ — readonly!
// const keyword siz — { host: string; port: number } (mutable)
```

### 2. `using` — faqat `Symbol.dispose` implement qilingan object lar bilan

```typescript
// ❌ — Symbol.dispose yo'q
using x = { value: 42 };
// Error: Type '{ value: number }' is not assignable to 'Disposable'

// ✅ — Disposable implement qilish
using x = { value: 42, [Symbol.dispose]() { console.log("disposed"); } };
```

### 3. `erasableSyntaxOnly` — `enum` va `namespace` TAQIQ

```typescript
// ❌ — enum runtime kod hosil qiladi
enum Status { Active, Inactive }

// ✅ — type erasure bilan muqobil
const Status = { Active: 0, Inactive: 1 } as const;
type Status = (typeof Status)[keyof typeof Status];
```

### 4. Inferred type predicates — ba'zan kutilmagan natija

```typescript
const items = [1, "hello", null, true];
const strings = items.filter(x => typeof x === "string");
// TS 5.5: strings: string[] — ✅ to'g'ri infer

// Lekin murakkab callback larda infer ISHLAMASLIGI mumkin
const complex = items.filter(x => {
  if (typeof x === "string") return true;
  return false;
});
// Ba'zan (string | number | boolean | null)[] qaytarishi mumkin
```

### 5. `NoInfer` — faqat inference bloklaydi, type check YO'Q

```typescript
function fn<T>(a: T, b: NoInfer<T>): T { return a; }

// NoInfer T ni b dan infer qilmaslikni aytadi
// Lekin b ning type i T ga mos kelishi KERAK (type check bor)
fn(42, "hello"); // ❌ — string ≠ number (type check ishlaydi)
fn(42, 0);       // ✅
```

---

## Common Mistakes

### ❌ Xato 1: `using` ni `Symbol.dispose` siz ishlatish

```typescript
// ❌ — Disposable implement qilinmagan
using conn = createConnection();
// Error agar createConnection Disposable qaytarmasa

// ✅ — Disposable qaytarish
function createConnection(): Disposable { /* ... */ }
```

### ❌ Xato 2: `erasableSyntaxOnly` da enum ishlatish

```typescript
// ❌ — Node.js --experimental-strip-types bilan CRASH
enum Color { Red, Green, Blue }

// ✅ — as const muqobil
const Color = { Red: 0, Green: 1, Blue: 2 } as const;
type Color = (typeof Color)[keyof typeof Color];
```

### ❌ Xato 3: `const` type parameter ni haddan tashqari ishlatish

```typescript
// ❌ — hamma joyda const kerak emas
function add<const T extends number>(a: T, b: T): number { return a + b; }
// T: 42, 5 kabi literal — lekin arithmetic natijasi number

// ✅ — const faqat config/routing kabi literal kerak bo'lganda
function defineRoutes<const T extends string[]>(routes: T) { return routes; }
```

### ❌ Xato 4: TS 5.5 inferred predicates ga haddan tashqari ishonish

```typescript
// ❌ — murakkab callback da infer ishlamasligi mumkin
const filtered = items.filter(item => {
  // Murakkab logic — TS infer qilolmasligi mumkin
  return someComplexCheck(item);
});

// ✅ — explicit type predicate (xavfsizroq)
const filtered = items.filter((item): item is ValidItem => someComplexCheck(item));
```

### ❌ Xato 5: `verbatimModuleSyntax` qo'ymaslik (yangi loyihada)

```json
// ❌ — kompilator heuristic bilan import o'chiradi (unpredictable)
{ "compilerOptions": { } }

// ✅ — explicit, predictable
{ "compilerOptions": { "verbatimModuleSyntax": true } }
```

---

## Amaliy Mashqlar

### Mashq 1: `using` bilan Resource Management (Oson)

**Savol:** `DatabaseConnection` class yarating — `Disposable` implement. `using` bilan ishlatib ko'ring.

<details>
<summary>Javob</summary>

```typescript
class DatabaseConnection implements Disposable {
  constructor(private url: string) { console.log(`Connected: ${url}`); }
  query(sql: string) { return []; }
  [Symbol.dispose]() { console.log(`Disconnected: ${this.url}`); }
}

function processData() {
  using db = new DatabaseConnection("postgres://localhost/mydb");
  db.query("SELECT * FROM users");
  // scope tugaganda avtomatik disconnect
}
processData();
// "Connected: postgres://localhost/mydb"
// "Disconnected: postgres://localhost/mydb"
```

</details>

---

### Mashq 2: `NoInfer` Pattern (O'rta)

**Savol:** `createStore<T>(initial, fallback)` — fallback dan infer qilmaslik.

<details>
<summary>Javob</summary>

```typescript
function createStore<T>(initial: T, fallback: NoInfer<T>) { return initial ?? fallback; }

createStore(42, 0);       // ✅ T = number
createStore("hello", ""); // ✅ T = string
// createStore(42, "x");  // ❌ — string ≠ number
```

</details>

---

### Mashq 3: Inferred Type Predicates (O'rta)

**Savol:** TS 5.5 da `.filter()` bilan null/undefined olib tashlash — explicit predicate KERAK EMAS.

<details>
<summary>Javob</summary>

```typescript
interface RawItem { id: string | null; value: string | number | undefined; label?: string; }

function processItems(raw: RawItem[]) {
  return raw
    .filter(item => item.id !== null)
    .filter(item => item.value !== undefined)
    .map(item => ({
      id: item.id, // string (null filtered)
      numValue: typeof item.value === "string" ? parseFloat(item.value) : item.value,
      label: item.label ?? `Item ${item.id}`,
    }))
    .filter(item => !isNaN(item.numValue));
}
```

</details>

---

### Mashq 4: `erasableSyntaxOnly` Migration (O'rta)

**Savol:** `enum`, `namespace`, `parameter property` ni `erasableSyntaxOnly` bilan mos qiling.

<details>
<summary>Javob</summary>

```typescript
// enum → as const
const LogLevel = { Debug: 0, Info: 1, Warn: 2, Error: 3 } as const;
type LogLevel = (typeof LogLevel)[keyof typeof LogLevel];

// namespace → oddiy object
const StringUtils = {
  capitalize: (s: string) => s[0].toUpperCase() + s.slice(1),
};

// parameter property → explicit
class Logger {
  name: string;
  private level: LogLevel;
  constructor(name: string, level: LogLevel = LogLevel.Info) {
    this.name = name;
    this.level = level;
  }
}
```

</details>

---

### Mashq 5: TS 5.x Feature Combination (Qiyin)

**Savol:** `const` type params + `satisfies` + `NoInfer` + `using` + inferred predicates.

<details>
<summary>Javob</summary>

```typescript
// const type params
function defineEndpoints<const T extends readonly { path: string; method: string }[]>(e: T) { return e; }
const API = defineEndpoints([{ path: "/users", method: "GET" }, { path: "/posts", method: "POST" }]);

// satisfies
type ApiConfig = { baseUrl: string; timeout: number };
const config = { baseUrl: "https://api.example.com", timeout: 5000 } satisfies ApiConfig;

// using + NoInfer
class ApiClient implements Disposable {
  constructor(private baseUrl: string) {}
  async fetch<T>(path: string, fallback: NoInfer<T>): Promise<T> {
    try { const res = await fetch(`${this.baseUrl}${path}`); return res.json(); }
    catch { return fallback; }
  }
  [Symbol.dispose]() { console.log("Client disposed"); }
}

// Inferred predicates
async function getActiveUsers() {
  using client = new ApiClient(config.baseUrl);
  const users = await client.fetch<({ name: string; active: boolean } | null)[]>("/users", []);
  return users
    .filter(u => u !== null)
    .filter(u => u.active)
    .map(u => u.name);
}
```

</details>

---

## Xulosa

| Versiya | Eng Muhim Feature | Kategoriya |
|---------|-------------------|------------|
| **5.0** | TC39 Decorators, `const` type params, bundler resolution | Language, Config |
| **5.1** | Implicit undefined returns, unrelated getter/setter | Ergonomics |
| **5.2** | `using`/`await using` (Explicit Resource Management) | **Runtime** |
| **5.3** | Import Attributes, `switch(true)` narrowing | Modules, Narrowing |
| **5.4** | `NoInfer<T>`, closure narrowing | Inference |
| **5.5** | `isolatedDeclarations`, inferred type predicates | **Performance** |
| **5.6** | Always-truthy checks, iterator helpers | Safety |
| **5.7** | Path rewriting, `es2024` target | Build |
| **5.8** | `erasableSyntaxOnly`, granular branch checks | **Node.js** |

**Trendlar:** JS standartlariga moslashish, DX yaxshilanish, build performance, Node.js ecosystem integratsiya.

**Bog'liq bo'limlar:**
- [Bo'lim 15: Utility Types](15-utility-types.md) — `NoInfer<T>`, `Awaited<T>`
- [Bo'lim 17: Modules](17-modules.md) — `verbatimModuleSyntax`, bundler resolution
- [Bo'lim 18: Declaration Files](18-declaration-files.md) — `isolatedDeclarations`
- [Bo'lim 19: Decorators](19-decorators.md) — TC39 decorators

---

**Bu kursning oxirgi bo'limi.** TypeScript ning barcha asosiy va ilg'or imkoniyatlari to'liq yoritilgan.

- [00-index.md](00-index.md) — Kurs mundarijasi va progress
- [interview/00-index.md](interview/00-index.md) — Interview savollari (Faza 2)
