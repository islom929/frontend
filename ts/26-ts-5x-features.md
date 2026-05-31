# Bo'lim 26: TypeScript 5.x Yangiliklari

> TypeScript 5.x seriyasi har minor release'da yangi type-level imkoniyatlar, developer experience yaxshilanishlar va runtime standartlariga moslashish olib keldi. Har feature uchun: nima o'zgardi, oldin qanday edi, hozir qanday va amaliy misollar. Ba'zi feature'lar boshqa bo'limlarda chuqur yoritilgan — ular uchun qisqacha recap va cross-reference beriladi.

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

**`const` type parameters** — generic'da `as const` behavior:

```typescript
function routes<const T extends readonly { path: string }[]>(r: T) { return r; }
const r = routes([{ path: "/" }, { path: "/about" }]);
// r: readonly [{ readonly path: "/" }, { readonly path: "/about" }]
// const keyword'siz: { path: string }[]
```

**`moduleResolution: "bundler"`** — Vite/Webpack muhiti uchun. Extension-less imports + `exports` field support. Batafsil [Bo'lim 17](17-modules.md).

**`verbatimModuleSyntax`** — `import type` majburiy, predictable emit. Batafsil [Bo'lim 17](17-modules.md#verbatimmodulesyntax).

**All enums are union enums** — TS 5.0'gacha `enum`'lar ikki xil edi: literal enum (har member o'z literal type'iga ega) va computed enum (har member umumiy `number`/`string` type'ga ega). TS 5.0 da har enum **union type** sifatida ishlanadi va har member uchun alohida literal type hisoblanadi:

```typescript
enum Status {
  Active = 1,
  Banned = 1 << 1, // computed (constant expression)
}

function check(s: Status) {
  if (s === Status.Active) {
    // TS 5.0+: s narrowed to Status.Active
    // TS 4.x: s: Status (umumiy)
  }
}
```

Bu narrowing va exhaustiveness check'ni aniqroq qiladi.

---

## TypeScript 5.1

### Nazariya

**Implicit undefined return** — `undefined` qaytaradigan funksiyalarda explicit `return undefined` yozish kerak emas:

```typescript
// TS 5.0 da — return undefined MAJBURIY edi
function getUserById(id: number): User | undefined {
  const user = users.find(u => u.id === id);
  if (!user) return undefined; // explicit return kerak edi
  return user;
}

// TS 5.1 da — return undefined keraksiz
function getUserById(id: number): User | undefined {
  const user = users.find(u => u.id === id);
  if (!user) return; // ✅ implicit undefined
  return user;
}

// Faqat `: undefined` return type'i bilan ham:
function logAction(msg: string): undefined {
  console.log(msg);
  // return undefined — KERAK EMAS
}
```

**Unrelated getter/setter types** — getter return type'i va setter parameter type'i bir-biriga assignable bo'lishi shart emas (TS 5.1'gacha kerak edi). Bu API design'da foydali: setter cheklangan input qabul qiladi, getter normalized output beradi:

```typescript
class Temperature {
  #celsius: number = 0;

  // Setter qabul qiladi: Celsius yoki Fahrenheit string
  set value(input: number | `${number}F`) {
    if (typeof input === "string") {
      const fahrenheit = parseFloat(input);
      this.#celsius = (fahrenheit - 32) * 5 / 9;
    } else {
      this.#celsius = input;
    }
  }

  // Getter doim Celsius number qaytaradi
  get value(): number {
    return this.#celsius;
  }
}

const t = new Temperature();
t.value = "100F"; // ✅ string input
t.value = 25;     // ✅ number input
const c: number = t.value; // ✅ number output
```

---

## TypeScript 5.2

### Nazariya

**`using` / `await using` — Explicit Resource Management** (TC39). Resource'larni avtomatik tozalash — `try/finally` o'rniga `using` keyword.

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

**`Symbol.dispose` va `Symbol.asyncDispose`** — resource class'larga `Disposable` yoki `AsyncDisposable` interface implement qilinadi:

```typescript
import * as fs from "node:fs";

class FileHandle implements Disposable {
  private fd: number;

  constructor(path: string) {
    this.fd = fs.openSync(path, "r");
  }

  read(): string {
    const buffer = Buffer.alloc(1024);
    const bytesRead = fs.readSync(this.fd, buffer);
    return buffer.toString("utf8", 0, bytesRead);
  }

  [Symbol.dispose](): void {
    fs.closeSync(this.fd);
    console.log("File handle closed");
  }
}

function openFile(path: string): FileHandle {
  return new FileHandle(path);
}
```

**Decorator Metadata** — `Symbol.metadata` orqali native metadata. Batafsil [Bo'lim 19](19-decorators.md#decorator-metadata--symbolmetadata).

---

## TypeScript 5.3

### Nazariya

**Import Attributes** — import'da metadata berish:

```typescript
import config from "./config.json" with { type: "json" };
// Runtime'ga bu JSON ekanini aytadi
```

**`switch(true)` narrowing** — `switch(true)` pattern'da har `case` expression discriminant sifatida type narrowing'ga sabab bo'ladi:

```typescript
function classify(x: string | number | boolean) {
  switch (true) {
    case typeof x === "string":
      return x.toUpperCase(); // x: string
    case typeof x === "number":
      return x.toFixed(2);    // x: number
    default:
      return String(x);       // x: boolean
  }
}
```

**TS 5.3'gacha** `switch(true)` ishlatish mumkin edi, lekin narrowing ishlamasdi (har `case`'da `x: string | number | boolean`). 5.3'da checker `case` expression'larni type guard sifatida tan oladi.

**`if/else if` muqobil:** aynan shu narrowing `if/else if` zanjirida ham ishlaydi. `switch(true)` — pattern matching ko'rinishidagi muqobil sintaksis.

---

## TypeScript 5.4

### Nazariya

**`NoInfer<T>`** — inference bloklash. Batafsil [Bo'lim 15](15-utility-types.md#noinfert-ts-54).

```typescript
function createStore<T>(initial: T, fallback: NoInfer<T>): T {
  return initial ?? fallback;
}

createStore(42, "hello"); // ❌ — T faqat 42'dan infer (number), "hello" mos emas
createStore(42, 0);       // ✅ — T = number
```

**Preserved narrowing in closures** — closure ichida **last assignment** dan keyin narrowing saqlanadi:

```typescript
function scheduleGreeting() {
  let message: string | number;

  message = "hello"; // last assignment
  setTimeout(() => {
    console.log(message.toUpperCase()); // ✅ TS 5.4+ da: string
    // TS 5.3 da: string | number (narrowing yo'qolardi)
  }, 100);
}
```

**`Object.groupBy` va `Map.groupBy`** — ES2024 type'lar:

```typescript
const users = [
  { name: "Ali", role: "admin" },
  { name: "Vali", role: "user" },
  { name: "Gani", role: "admin" },
];

const grouped = Object.groupBy(users, u => u.role);
// {
//   admin: [{ name: "Ali", role: "admin" }, { name: "Gani", role: "admin" }],
//   user:  [{ name: "Vali", role: "user" }]
// }
// grouped type: Partial<Record<string, { name: string; role: string }[]>>
```

---

## TypeScript 5.5

### Nazariya

**`isolatedDeclarations`** — fayl-bo'yicha declaration emit. Batafsil [Bo'lim 18](18-declaration-files.md#isolateddeclarations-ts-55).

**Inferred type predicates** — `filter` callback'da avtomatik type narrowing. TS 5.5 da compiler callback body'ni analiz qiladi va agar narrowing pattern aniqlansa, type predicate sifatida infer qiladi:

```typescript
// TS 5.4 va undan oldin — explicit predicate kerak:
const values1: (number | null | undefined)[] = [1, null, 2, undefined, 3];
const nums1 = values1.filter((v): v is number => v != null);
// nums1: number[]

// TS 5.5+ — avtomatik infer:
const values2: (number | null | undefined)[] = [1, null, 2, undefined, 3];
const nums2 = values2.filter(v => v != null);
// nums2: number[] — TS avtomatik type predicate infer qildi
```

**Mexanizm:** checker predicate'ni infer qilishi uchun to'rt shart bajarilishi kerak: (1) funksiyada explicit return type yoki predicate annotation yo'q; (2) bitta `return` statement, implicit return yo'q; (3) parameter mutatsiya qilinmaydi; (4) qaytarilgan boolean expression aynan parameter narrowing'iga bog'liq. Shu sababli `!= null` va `typeof x === "string"` infer bo'ladi, lekin `!!value` kabi truthiness check infer bo'lmaydi — `false` bo'lganda `value` `undefined` ham, `0` ham bo'lishi mumkin, ya'ni "if and only if" semantikasi buziladi.

---

## TypeScript 5.6

### Nazariya

**Always-truthy/nullish checks** — TS 5.6 condition ichida **syntaktik tarzda** doim truthy yoki erishib bo'lmaydigan (unreachable) deb aniqlangan expression'lar uchun xato beradi (bug guard):

```typescript
// ❌ TS 5.6 — Regex literal condition'da doim truthy (object)
function isValid(input: string) {
  if (/^[a-z]+$/) { // ❌ "This kind of expression is always truthy."
    return true;
  }
  return false;
  // Programmer ehtimol input.match(/^[a-z]+$/) yozmoqchi edi
}

// ❌ Condition ichidagi arrow function literal doim truthy
function check(count: number) {
  if (count => 0) { /* ... */ } // ❌ "always truthy" — `>=` yozmoqchi edi
}

// ❌ ?? ning chap operandi hech qachon nullish bo'lmasa, o'ng operand unreachable
function getLabel(name: string) {
  return name ?? "anonymous"; // ❌ "right operand is unreachable" — name hech qachon nullish emas
}
```

**Diqqat:** check **syntaktik aniqlik** talab qiladi — regex/arrow function literal, yoki type'i hech qachon nullish bo'lmaydigan `??` chap operandi. `true`, `false`, `0`, `1` literal'lari (`while (true)` kabi idiomatik kod uchun) bu tekshiruvdan ozod.

**Iterator helpers** — ECMAScript Iterator Helpers proposal (Stage 4, ES2025) uchun TS type'lar. `Iterator.prototype.filter`, `map`, `take`, `toArray` lazy iteration imkonini beradi (infinite generator'lar bilan ham ishlaydi):

```typescript
function* numbers() {
  let i = 0;
  while (true) yield i++;
}

const first10Even = numbers()
  .filter(n => n % 2 === 0)
  .take(10)
  .toArray();
// [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

**Runtime support:** Node.js 22+, Chrome 122+, Firefox 131+. Eski environment'lar uchun polyfill (`core-js/proposals/iterator-helpers`). TS 5.6'da faqat type'lar qo'shilgan — runtime allohida.

---

## TypeScript 5.7

### Nazariya

**Path rewriting** — `rewriteRelativeImportExtensions: true` flag bilan compiler `.ts`/`.tsx`/`.mts`/`.cts` extension'lardan `.js`/`.jsx`/`.mjs`/`.cjs`'ga avtomatik rewrite qiladi. Bu Node.js ESM (extension majburiy) va `--allowImportingTsExtensions` bilan birgalikda ishlaydi:

```jsonc
{
  "compilerOptions": {
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true
  }
}
```

```typescript
// Source: src/app.ts
import { utils } from "./utils.ts";
// Emit: dist/app.js
// import { utils } from "./utils.js";
```

**Cheklov:** faqat **relative** import'larga ishlaydi (`./`, `../`). Package import'lar (`node_modules`) o'zgartirilmaydi.

**`es2024` target** — `Object.groupBy`, `Map.groupBy`, `Promise.withResolvers`, `ArrayBuffer.prototype.resize` type'lari. `Symbol.dispose`/`Symbol.asyncDispose` esa alohida `esnext.disposable` lib'da (TS 5.2'da qo'shilgan) — `lib: ["es2024", "esnext.disposable"]` bilan yoqiladi.

---

## TypeScript 5.8

### Nazariya

**`erasableSyntaxOnly`** — faqat **type erasure** bilan o'chiriladigan syntax'ga ruxsat. `enum`, `namespace`, `parameter properties` — TAQIQLANADI. Node.js'ning `--experimental-strip-types` flag'iga mo'ljallangan.

```json
{ "compilerOptions": { "erasableSyntaxOnly": true } }
```

```typescript
// ❌ — erasableSyntaxOnly'da TAQIQLANADI:
enum Color { Red, Green }           // ❌ — runtime kod hosil qiladi
namespace Utils { export const x = 1; } // ❌ — runtime IIFE hosil qiladi
class A { constructor(public name: string) {} } // ❌ — parameter property

// ✅ — ruxsat:
type Color = "red" | "green";       // ✅ — type erasure
const Utils = { x: 1 };             // ✅ — oddiy JS
class A { name: string; constructor(name: string) { this.name = name; } } // ✅
```

**Granular Checks for Branches in Return Expressions** — `return` statement ichida to'g'ridan-to'g'ri yozilgan conditional expression (`cond ? a : b`) har bir branch'i funksiyaning declared return type'iga **alohida** tekshiriladi. Muammo `any` aralashganda kelib chiqardi: `any | string` union'i `any`'ga sodda­lashadi, shu sababli checker butun ternary natijasini tekshirganda branch'ning xato type'i `any` ichida yo'qolib ketardi. TS 5.8 har branch'ni mustaqil tekshirib bu bo'shliqni yopadi:

```typescript
declare const untypedCache: Map<any, any>;

function getUrlObject(urlString: string): URL {
  return untypedCache.has(urlString)
    ? untypedCache.get(urlString) // any
    : urlString;                  // ❌ TS 5.8 — 'string' 'URL' ga assignable emas
}
```

`untypedCache.get()` `any` qaytaradi, `any | string` union esa `any`'ga soddalashadi — TS 5.8'gacha checker butun ternary natijasini `any` deb ko'rib `string` branch'idagi xatoni o'tkazib yuborardi. TS 5.8 har branch'ni alohida `URL`'ga solishtirib xatoni ushlaydi. Generic conditional va indexed access return type'lar bilan kengaytirilgan tekshiruv esa TS 5.9'ga qoldirildi.

---

## Version Selection Guide

### Nazariya

| Loyiha turi | Tavsiya etilgan TS versiya | Sabab |
|-------------|--------------------------|-------|
| Yangi loyiha | **5.8** (eng so'nggi stable) | To'liq feature set + bug fix |
| Node.js native TS run | **5.8** | `erasableSyntaxOnly` + `--experimental-strip-types` |
| Angular / Vue / framework | framework `peerDependencies` ko'rsatgan range | Har framework qo'llab-quvvatlangan TS range'ni e'lon qiladi |
| Legacy maintenance | mavjud major'da eng so'nggi minor | Minimal breaking risk |

**Migration qoida:** minor versiyani muntazam yangilab borish. Minor release'lar orasida breaking change kam, lekin har release notes'dagi "Breaking Changes" bo'limini o'qib chiqish kerak.

---

## Edge Cases va Gotchas

### 1. `const` type parameter — inference `as const`'ga teng

`<const T>` modifier'i type parameter inference'iga `as const` semantikasini qo'shadi: object literal'lar `readonly`, string literal'lar narrow type, array'lar tuple bo'ladi.

```typescript
function getConfig<const T>(config: T) { return config; }
const cfg = getConfig({ host: "localhost", port: 3000 });
// cfg: { readonly host: "localhost"; readonly port: 3000 }
// cfg.host = "other"; // ❌ readonly
```

```typescript
// const SIZ — wide inference:
function noConst<T>(routes: T) { return routes; }
const r1 = noConst([{ path: "/" }]);
// r1: { path: string }[]

// const BILAN — literal/tuple inference:
function withConst<const T>(routes: T) { return routes; }
const r2 = withConst([{ path: "/" }]);
// r2: readonly [{ readonly path: "/" }]
```

**Qachon kerak:** routing config, state machine, schema definition — call site'da `as const` yozmasdan literal inference.

**Qachon kerak emas:** numeric/general computation — `<const T extends number>`'da `T = 42` infer bo'ladi, lekin arithmetic foydasiz (`a + b` baribir `number`).

### 2. `using` — faqat `Symbol.dispose` implement qilingan object'lar bilan

```typescript
// ❌ — Symbol.dispose yo'q
using resource = { value: 42 };
// Error: Type '{ value: number; }' must have a '[Symbol.dispose]()' method.

// ✅ — Disposable implement qilish
using resource = { value: 42, [Symbol.dispose]() { console.log("disposed"); } };
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
const items: (number | string | null | boolean)[] = [1, "hello", null, true];

// ✅ Oddiy callback — to'g'ri infer
const strings = items.filter(x => typeof x === "string");
// strings: string[]

// ⚠️ Murakkab control flow — infer ishlamasligi mumkin
const filtered = items.filter(x => {
  if (typeof x === "string") return true;
  if (typeof x === "number" && x > 0) return true;
  return false;
});
// filtered: (number | string | null | boolean)[] — narrow EMAS
// Sabab: infer sharti — bitta `return` statement; bu callback'da uchta `return` bor
```

**Yechim:** Murakkab logic uchun explicit type predicate:

```typescript
const filtered = items.filter((x): x is string | number =>
  typeof x === "string" || (typeof x === "number" && x > 0)
);
// filtered: (string | number)[]
```

### 5. `NoInfer` — faqat inference bloklaydi, type check YO'Q

```typescript
function pickFirst<T>(primary: T, fallback: NoInfer<T>): T { return primary; }

// NoInfer fallback'dan T'ni infer qilmaslikni aytadi
// Lekin fallback type'i T'ga mos kelishi KERAK (type check bor)
pickFirst(42, "hello"); // ❌ — string ≠ number (type check ishlaydi)
pickFirst(42, 0);       // ✅
```

---

## Common Mistakes

### ❌ Xato 1: `using`'ni `Symbol.dispose`'siz ishlatish

```typescript
// ❌ — qaytarilgan object Disposable emas
function createConnection(): { query(sql: string): unknown[] } {
  return { query: () => [] };
}
using connection = createConnection();
// Error: ... must have a '[Symbol.dispose]()' method.

// ✅ — Disposable qaytarish
function createDisposableConnection(): Disposable & { query(sql: string): unknown[] } {
  return { query: () => [], [Symbol.dispose]() { /* close */ } };
}
using connection = createDisposableConnection();
```

### ❌ Xato 2: `erasableSyntaxOnly`'da enum ishlatish

```typescript
// ❌ — Node.js --experimental-strip-types bilan CRASH
enum Color { Red, Green, Blue }

// ✅ — as const muqobil
const Color = { Red: 0, Green: 1, Blue: 2 } as const;
type Color = (typeof Color)[keyof typeof Color];
```

### ❌ Xato 3: `const` type parameter'ni haddan tashqari ishlatish

```typescript
// ❌ — hamma joyda const kerak emas
function add<const T extends number>(a: T, b: T): number { return a + b; }
// T: 42, 5 kabi literal — lekin arithmetic natijasi number

// ✅ — const faqat config/routing kabi literal kerak bo'lganda
function defineRoutes<const T extends string[]>(routes: T) { return routes; }
```

### ❌ Xato 4: TS 5.5 inferred predicates'ga haddan tashqari ishonish

```typescript
// ❌ — murakkab callback'da infer ishlamasligi mumkin
const filtered = items.filter(item => {
  // Murakkab logic — TS infer qilolmasligi mumkin
  return someComplexCheck(item);
});

// ✅ — explicit type predicate (xavfsizroq)
const filtered = items.filter((item): item is ValidItem => someComplexCheck(item));
```

### ❌ Xato 5: `verbatimModuleSyntax` qo'ymaslik (yangi loyihada)

```json
// ❌ — compiler heuristic bilan import o'chiradi (unpredictable)
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

**Savol:** `createStore<T>(initial, fallback)` — fallback'dan infer qilmaslik.

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

**Savol:** TS 5.5 da `.filter()` bilan `null`/`undefined` olib tashlash — explicit predicate KERAK EMAS. Inference faqat parameter'ning **o'ziga** narrowing qilinganda ishlaydi, parameter property'siga emas.

<details>
<summary>Javob</summary>

```typescript
const ids: (string | null)[] = ["a", null, "b", null, "c"];

// ✅ check parameter'ning o'ziga — TS 5.5 type predicate infer qiladi
const validIds = ids.filter(id => id !== null);
// validIds: string[]

const values: (number | undefined)[] = [1, undefined, 2, 3, undefined];
const validValues = values.filter(value => value !== undefined);
// validValues: number[]

// ⚠️ property check — inference ISHLAMAYDI (parameter o'zi narrow bo'lmaydi):
interface RawItem { id: string | null; }
const items: RawItem[] = [{ id: "a" }, { id: null }];
const kept = items.filter(item => item.id !== null);
// kept: RawItem[] — item.id hali ham string | null
// Bunday holatda explicit predicate kerak:
const kept2 = items.filter((item): item is RawItem & { id: string } => item.id !== null);
// kept2: (RawItem & { id: string })[]
```

</details>

---

### Mashq 4: `erasableSyntaxOnly` Migration (O'rta)

**Savol:** `enum`, `namespace`, `parameter property`'ni `erasableSyntaxOnly` bilan mos qiling.

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
| **5.5** | `isolatedDeclarations`, inferred type predicates | Build, Inference |
| **5.6** | Always-truthy checks, iterator helpers | Safety |
| **5.7** | Path rewriting, `es2024` target | Build |
| **5.8** | `erasableSyntaxOnly`, return branch granular checks | **Node.js** |

**Trendlar:** JS standartlariga moslashish, DX yaxshilanish, build performance, Node.js ecosystem integration.

**Bog'liq bo'limlar:**
- [Bo'lim 15: Utility Types](15-utility-types.md) — `NoInfer<T>`, `Awaited<T>`
- [Bo'lim 17: Modules](17-modules.md) — `verbatimModuleSyntax`, bundler resolution
- [Bo'lim 18: Declaration Files](18-declaration-files.md) — `isolatedDeclarations`
- [Bo'lim 19: Decorators](19-decorators.md) — TC39 decorators

---

**Bu kursning oxirgi bo'limi.** TypeScript ning barcha asosiy va ilg'or imkoniyatlari to'liq yoritilgan.

- [00-index.md](00-index.md) — Kurs mundarijasi va progress
- [interview/00-index.md](interview/00-index.md) — Interview savollari (Faza 2)
