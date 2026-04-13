# Bo'lim 18: Declaration Files (.d.ts)

> Declaration files (`.d.ts`) — JavaScript kodiga type information beradigan fayllar. Ular faqat type declaration lar (interface, type alias, function signature) ni o'z ichiga oladi — implementation (actual code) yo'q. Bu JavaScript kutubxonalarga TypeScript type safety qo'shish, library type larini distribute qilish, va global type larni declare qilish uchun ishlatiladi. Bu bo'limda `declare` keyword, avtomatik va qo'lda declaration yaratish, `isolatedDeclarations`, DefinitelyTyped, va declaration testing o'rganiladi.

---

## Mundarija

- [Declaration Files Nima](#declaration-files-nima)
- [`.d.ts` vs `.ts` Farqi](#dts-vs-ts-farqi)
- [`declare` Keyword](#declare-keyword)
- [Declaration File Yaratish — Avtomatik](#declaration-file-yaratish--avtomatik)
- [Declaration File Yaratish — Qo'lda](#declaration-file-yaratish--qolda)
- [`isolatedDeclarations` (TS 5.5+)](#isolateddeclarations-ts-55)
- [DefinitelyTyped — `@types/*`](#definitelytyped--types)
- [`typeRoots` va `types`](#typeroots-va-types)
- [Ambient Declarations](#ambient-declarations)
- [Triple-Slash Directives](#triple-slash-directives)
- [Declaration File Testing](#declaration-file-testing)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Declaration Files Nima

### Nazariya

Declaration file (`.d.ts`) — JavaScript code uchun **type information** beruvchi fayl. U faqat type lar, interface lar, function signature lar, class shape larni declare qiladi — actual implementation (function body, class method logic) yo'q.

Declaration file lar quyidagi hollarda kerak:

1. **JavaScript kutubxona uchun type** — kutubxona JS da yozilgan, lekin TypeScript project da ishlatmoqchisiz
2. **Library distribute qilish** — `.js` + `.d.ts` sifatida distribute qilinadi
3. **Global type lar** — `window`, `document`, `process` kabi global object lar (`lib.d.ts`)
4. **Non-TS fayl lar** — CSS, JSON, image fayllar import qilganda type berish ([Bo'lim 17](17-modules.md#ambient-modules))

Kompilator `.d.ts` fayllarni **faqat type-checking** uchun o'qiydi. Ular JS ga compile **qilinmaydi** — allaqachon faqat type ma'lumot.

```
Source fayllar:           Declaration fayllar:
┌──────────┐              ┌──────────────┐
│  app.ts  │──compile──→  │   app.js     │  (runtime code)
│          │──emit d.ts→  │   app.d.ts   │  (type info faqat)
└──────────┘              └──────────────┘

node_modules:
┌──────────────────────┐
│  express/            │
│  ├── index.js        │  (runtime code)
│  └── index.d.ts      │  (type info — TS shu faylni o'qiydi)
└──────────────────────┘
```

---

## `.d.ts` vs `.ts` Farqi

### Nazariya

| Xususiyat | `.ts` | `.d.ts` |
|-----------|-------|---------|
| Implementation (function body) | ✅ Bor | ❌ Yo'q |
| Type annotations | ✅ | ✅ |
| JS ga compile bo'ladi | ✅ | ❌ (allaqachon type faqat) |
| `declare` keyword kerak | ❌ (ixtiyoriy) | ✅ (ko'p hollarda) |
| Runtime da mavjud | ✅ (compile natijasi) | ❌ |

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// utils.ts — to'liq implementation
export function formatDate(date: Date): string {
  return date.toISOString().split("T")[0];
}

export interface DateOptions {
  locale: string;
  timezone: string;
}
```

```typescript
// utils.d.ts — faqat type (implementation yo'q)
export declare function formatDate(date: Date): string;

export interface DateOptions {
  locale: string;
  timezone: string;
}
// Function body yo'q — faqat signature
// Interface — zaten faqat type, declare kerak emas
```

Kompilator `.d.ts` faylni o'qiganda faqat type information oladi va symbol table ga qo'shadi. `.ts` fayldan farqli, `.d.ts` da kompilator **type inference qilmaydi** — barcha type lar explicit yozilgan bo'lishi kerak. JS emit bosqichi butunlay skip bo'ladi.

</details>

---

## `declare` Keyword

### Nazariya

`declare` keyword — kompilatorga "bu narsa **boshqa joyda** mavjud, men faqat type ini aytayapman" degan signal. `declare` bilan yozilgan code JS ga **emit qilinmaydi** — faqat type-checking uchun.

Kompilator `declare` li declaration larni type-check qiladi (parametr type lari, return type lari tekshiriladi), lekin **implementation mavjudligini talab qilmaydi**. Oddiy function da body bo'lmasa error, lekin `declare function` da body talab qilinmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

**`declare const` / `declare let` / `declare var`:**

```typescript
declare const $: (selector: string) => any;
declare const jQuery: typeof $;
declare const __DEV__: boolean;
declare const __VERSION__: string;
```

**`declare function`:**

```typescript
declare function structuredClone<T>(value: T): T;

declare function fetch(input: string): Promise<Response>;
declare function fetch(input: string | Request, init?: RequestInit): Promise<Response>;
```

**`declare class`:**

```typescript
declare class EventEmitter {
  on(event: string, listener: (...args: any[]) => void): this;
  emit(event: string, ...args: any[]): boolean;
  removeListener(event: string, listener: (...args: any[]) => void): this;
}
```

**`declare module`:**

```typescript
declare module "lodash" {
  export function chunk<T>(array: T[], size: number): T[][];
  export function debounce<T extends (...args: any[]) => any>(func: T, wait: number): T;
}

declare module "*.svg" {
  const content: string;
  export default content;
}
```

**`declare namespace`:**

```typescript
declare namespace Express {
  interface Request { body: any; params: Record<string, string>; }
  interface Response { json(data: any): void; status(code: number): Response; }
}
```

**`declare global`:**

```typescript
export {};
declare global {
  interface Window { __STORE__: any; }
  function sleep(ms: number): Promise<void>;
}
```

**`declare enum`:**

```typescript
declare enum LogLevel { Debug = 0, Info = 1, Warn = 2, Error = 3 }
```

</details>

---

## Declaration File Yaratish — Avtomatik

### Nazariya

TypeScript compiler `.ts` fayllardan avtomatik `.d.ts` fayllar generate qila oladi. Bu library yaratganda eng ko'p ishlatiladigan usul — source `.ts` da yoziladi, publish qilganda `.js` + `.d.ts` generate bo'ladi.

**tsconfig Options:**

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./dist/types",
    "declarationMap": true,
    "emitDeclarationOnly": true
  }
}
```

- `declaration: true` — `.d.ts` fayl lar generate bo'ladi
- `declarationDir` — `.d.ts` fayllar qayerga yoziladi
- `declarationMap: true` — `.d.ts.map` fayl lar — IDE "Go to Definition" da `.ts` source ga o'tadi
- `emitDeclarationOnly: true` — faqat `.d.ts`, `.js` emas (bundler JS yaratganda)

<details>
<summary><strong>Under the Hood</strong></summary>

`declaration: true` yoqilganda, kompilator ikki parallel emit ishlaydi: biri JS output yaratadi, ikkinchisi `.d.ts` output. Declaration emit — **implementation erasure**: function body, class method body, variable initializer lar olib tashlanadi, faqat signature saqlanadi.

Agar function ning explicit return type bo'lmasa, kompilator inferred return type ni hisoblab `.d.ts` ga yozadi. Shuning uchun standart `declaration: true` da explicit return type yozmaslik mumkin — kompilator o'zi hisoblaydi (lekin `isolatedDeclarations` da bu ishlamaydi).

`private` member lar declaration da maxsus holat: private field lar `.d.ts` ga **nomi bilan** yoziladi, lekin type i yozilmaydi. Bu structural compatibility uchun — consumer private field ga access qilolmaydi, lekin field mavjudligi ko'rsatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// src/utils.ts
export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount);
}

export interface FormatOptions {
  locale: string;
  minimumFractionDigits: number;
}

export class Formatter {
  private locale: string;
  constructor(locale: string) { this.locale = locale; }
  format(value: number): string { return value.toLocaleString(this.locale); }
}
```

`tsc --declaration` natijasi:

```typescript
// dist/utils.d.ts (avtomatik generate)
export declare function formatCurrency(amount: number, currency: string): string;

export interface FormatOptions {
  locale: string;
  minimumFractionDigits: number;
}

export declare class Formatter {
  private locale;
  constructor(locale: string);
  format(value: number): string;
}
```

**`package.json` da Type Entry:**

```json
{
  "name": "my-library",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

`"types"` conditional export — `exports` ichida **birinchi** bo'lishi kerak.

</details>

---

## Declaration File Yaratish — Qo'lda

### Nazariya

Qo'lda declaration yozish quyidagi hollarda kerak:

1. **JavaScript kutubxona** — source TypeScript da emas, `@types/*` package ham yo'q
2. **Global variable lar** — CDN dan load qilingan script lar
3. **Non-standard module lar** — CSS, image va boshqa fayl turlari

Kompilator qo'lda yozilgan `.d.ts` ni avtomatik generate qilingan bilan **identik tarzda** ishlatadi — farq yo'q.

**Muhim:** Qo'lda yozilgan declaration larning runtime API ga mosligi kompilator tomonidan tekshirilmaydi. Noto'g'ri type yozilsa — runtime error bo'ladi, lekin compile-time error bo'lmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

**JS kutubxona uchun to'liq declaration:**

```typescript
// types/analytics.d.ts
declare module "analytics-lib" {
  export interface AnalyticsConfig {
    apiKey: string;
    debug?: boolean;
    batchSize?: number;
  }

  export interface Event {
    name: string;
    properties?: Record<string, string | number | boolean>;
    timestamp?: Date;
  }

  export class Analytics {
    constructor(config: AnalyticsConfig);
    track(event: Event): void;
    identify(userId: string, traits?: Record<string, unknown>): void;
    flush(): Promise<void>;
  }

  export default function createAnalytics(config: AnalyticsConfig): Analytics;
}
```

**Global script declaration:**

```typescript
// types/google-maps.d.ts
declare namespace google.maps {
  class Map {
    constructor(element: HTMLElement, options: MapOptions);
    setCenter(latLng: LatLng): void;
    setZoom(zoom: number): void;
  }

  interface MapOptions { center: LatLng; zoom: number; mapTypeId?: string; }
  interface LatLng { lat: number; lng: number; }

  class Marker {
    constructor(options: MarkerOptions);
    setPosition(latLng: LatLng): void;
    setMap(map: Map | null): void;
  }

  interface MarkerOptions { position: LatLng; map: Map; title?: string; }
}
```

</details>

---

## `isolatedDeclarations` (TS 5.5+)

### Nazariya

`isolatedDeclarations` — har bir faylning `.d.ts` sini **boshqa fayl larning type ma'lumotisiz** generate qilish imkonini beradigan tsconfig option. Bu `isolatedModules` ning declaration versiyasi.

**Nima uchun kerak:** Standart `declaration: true` da kompilator butun project ni tahlil qiladi — sekin. `isolatedDeclarations` bilan har bir fayl **mustaqil** ravishda declaration generate qila oladi — bu parallelization va tezkor tool lar (esbuild, SWC) bilan ishlash imkonini beradi.

**Cheklov:** Barcha exported function va variable larning **explicit return type** va **explicit type annotation** bo'lishi kerak. Inference ishlamaydi.

| Xususiyat | `declaration: true` | `isolatedDeclarations: true` |
|-----------|--------------------|-----------------------------|
| Tezlik | Sekin (butun project) | Tez (fayl-bo'yicha) |
| Parallelization | ❌ | ✅ |
| esbuild/SWC bilan | ❌ | ✅ |
| Explicit types | Ixtiyoriy | Majburiy (exported) |

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// isolatedDeclarations: true

// ❌ Error — return type yo'q
export function add(a: number, b: number) {
  return a + b;
}

// ✅ — explicit return type
export function add(a: number, b: number): number {
  return a + b;
}

// ❌ Error — initializer dan type inference kerak
export const config = { host: "localhost", port: 3000 };

// ✅ — explicit type
export interface Config { host: string; port: number; }
export const config: Config = { host: "localhost", port: 3000 };

// ✅ Local (unexported) code cheklanmaydi
const internal = { x: 1, y: 2 }; // OK — exported emas
function helper(n: number) { return n * 2; } // OK — exported emas
```

</details>

---

## DefinitelyTyped — `@types/*`

### Nazariya

**DefinitelyTyped** — community tomonidan boshqariladigan eng katta TypeScript declaration file lari repositoriyasi. JavaScript kutubxonalar uchun type definition lar. `@types/` scope da npm package sifatida install qilinadi:

```bash
npm install --save-dev @types/node       # Node.js API type lari
npm install --save-dev @types/express    # Express type lari
npm install --save-dev @types/lodash     # Lodash type lari
npm install --save-dev @types/react      # React type lari
```

**Type lar qayerda qidiriladi:**

```
1. Package o'zida:
   package.json → "types": "./dist/index.d.ts"
   package.json → "exports" → "types" conditional

2. @types package:
   node_modules/@types/package-name/index.d.ts

3. typeRoots (tsconfig):
   Belgilangan papkalar ichida
```

Agar kutubxona o'zida type lar bo'lsa — `@types` package **kerak emas** (va install qilinmasligi kerak — conflict yaratadi).

**Versiya moslashtirish:** `@types` package versiyasi kutubxona ning major.minor versiyasiga mos bo'lishi kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
{
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "^4.17.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/lodash": "^4.17.0"
  }
}
```

Kompilator module import qilganda type larni quyidagi tartibda qidiradi:
1. Package o'zida `"types"` field → topilsa — ishlatadi
2. `@types/package-name` → fallback
3. Ikkalasi ham yo'q → `any` (yoki error `noImplicitAny` bilan)

</details>

---

## `typeRoots` va `types`

### Nazariya

`tsconfig.json` dagi `typeRoots` va `types` — qaysi type declaration lar avtomatik include qilinishini boshqaradi.

**`typeRoots`** — type definition lar qidirilishi kerak bo'lgan root papkalar:

```json
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./custom-types"]
  }
}
```

Default: `["./node_modules/@types"]`. Agar `typeRoots` belgilanmasa — kompilator `node_modules/@types` ni avtomatik qidiradi.

**`types`** — qaysi aniq package larni include qilish:

```json
{
  "compilerOptions": {
    "types": ["node", "jest"]
  }
}
```

Bu qachon kerak:
- **Jest va Vitest conflict** — ikkalasining `describe`, `it` global type lari to'qnashadi
- **Global pollution oldini olish** — faqat kerakli `@types` larni include qilish

**Muhim:** `types` array **faqat automatic inclusion** ga ta'sir qiladi. Kodda explicit `import` yoki `/// <reference types="..." />` bo'lsa — `types` filter ni bypass qiladi.

---

## Ambient Declarations

### Nazariya

Ambient declaration — **implementation yo'q**, faqat type ma'lumot beruvchi declaration. `.d.ts` fayllarning barcha content i ambient. `.ts` faylda esa `declare` keyword bilan ambient declaration yaratiladi.

Ambient declaration lar **declaration merging** orqali ishlaydi — bir xil nomli interface lar avtomatik birlashadi. Bir nechta fayl bir xil module ni declare qilsa, ularning declaration lari **birlashtiriladi**.

<details>
<summary><strong>Kod Misollari</strong></summary>

**UMD Library:**

```typescript
// types/moment.d.ts
export as namespace moment; // global sifatida ham mavjud

export interface Moment {
  format(template: string): string;
  add(amount: number, unit: string): Moment;
  isValid(): boolean;
}

export function moment(date?: string | Date): Moment;
export default moment;
```

```typescript
// Module sifatida:
import moment from "moment";
const date = moment().format("YYYY-MM-DD");

// Script da global sifatida (agar UMD build load qilingan bo'lsa):
const date = moment().format("YYYY-MM-DD");
```

**Global Library:**

```typescript
// types/globals.d.ts
declare function gtag(command: "event", action: string, params: Record<string, unknown>): void;
declare function gtag(command: "config", id: string, params?: Record<string, unknown>): void;
declare const dataLayer: Array<Record<string, unknown>>;
```

</details>

---

## Triple-Slash Directives

### Nazariya

Triple-slash directive — fayl boshida `/// <reference ... />` formatida yoziladigan maxsus comment. Kompilatorga qo'shimcha type fayl larni include qilishni aytadi. Zamonaviy TypeScript da kam ishlatiladi (`import` afzalroq), lekin ba'zi hollarda hali kerak.

**Muhim cheklov:** Triple-slash directive lar **faqat fayl boshida** ishlaydi — birinchi actual statement dan oldin bo'lishi kerak. Fayl o'rtasida yozilgan `/// <reference />` oddiy comment sifatida ignore qilinadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
/// <reference types="node" />
// @types/node ni include qiladi

/// <reference types="jest" />
// @types/jest ni include qiladi

/// <reference path="./global-types.d.ts" />
// Bu fayl dagi type larni include qiladi

/// <reference lib="es2020" />
// Built-in lib ni include qiladi
```

**Qachon kerak:**
- `.d.ts` fayllarida — `import` ishlatish mumkin bo'lmagan joyda
- Global type dependency — fayl global scope dagi type larga bog'liq bo'lganda
- `lib` override — aniq fayl uchun lib ni o'zgartirish kerak bo'lganda

</details>

---

## Declaration File Testing

### Nazariya

Declaration file lar production package bilan birga chiqariladi — shuning uchun ularni **test qilish** muhim. Noto'g'ri `.d.ts` fayl foydalanuvchilarga type error beradi yoki noto'g'ri type beradi.

3 ta asosiy tool:

1. **`tsd`** — assertion-based type test (eng keng tarqalgan)
2. **`dtslint`** — DefinitelyTyped uchun maxsus
3. **`@arethetypeswrong/cli` (`attw`)** — package type distribution tekshirish

<details>
<summary><strong>Kod Misollari</strong></summary>

**`tsd` — type assertions:**

```typescript
// index.test-d.ts
import { expectType, expectError, expectNotType } from "tsd";
import { createUser, User } from "./index.js";

const user = createUser("Ali", 25);
expectType<User>(user);
expectType<string>(user.name);

expectError(createUser(123, "yigirma"));
expectNotType<any>(user);
```

```json
{ "scripts": { "test:types": "tsd" } }
```

**`@arethetypeswrong/cli` — package distribution:**

```bash
npx @arethetypeswrong/cli ./my-package-1.0.0.tgz
# CJS/ESM resolution muammolarini ko'rsatadi
# ✅ "main" — CommonJS — types match
# ❌ "exports['.'].import" — types not found  ← MUAMMO!
```

</details>

---

## Edge Cases va Gotchas

### 1. `declare module` — script vs module kontekstda farqli ishlaydi

```typescript
// FILE: types.d.ts (script — import/export YO'Q)
declare module "express" {
  interface Request { userId: string; }
}
// Bu AMBIENT MODULE DECLARATION — yangi module yaratadi
// Express ning mavjud Request ga QO'SHMAYDI — uni OVERRIDE qiladi!

// FILE: types.d.ts (module — import bor)
import "express";
declare module "express" {
  interface Request { userId: string; }
}
// Bu MODULE AUGMENTATION — mavjud module ni kengaytiradi ✅
```

### 2. `private` member lar `.d.ts` da type siz ko'rinadi

```typescript
// Source:
class User {
  private password: string;
  constructor(public name: string, password: string) {
    this.password = password;
  }
}

// Generated .d.ts:
declare class User {
  private password; // ← TYPE YO'Q!
  name: string;
  constructor(name: string, password: string);
}

// Nima uchun: private member type i ko'rinmaydi — structural compatibility
// uchun faqat MAVJUDLIGI ko'rsatiladi. Consumer access qilolmaydi.
```

### 3. `@types` va library o'z type lari conflict

```bash
npm install axios         # Axios — o'z type lari bor
npm install @types/axios  # ❌ CONFLICT!
```

Agar library `package.json` da `"types"` field bor bo'lsa — `@types` **kerak emas**. Ikkalasi install bo'lsa, kompilator qaysi birini ishlatishni bilmasligi mumkin.

### 4. `exports` da `types` tartib muhim

```json
// ❌ — "types" oxirida
{ "exports": { ".": { "import": "./dist/index.mjs", "types": "./dist/index.d.ts" } } }

// ✅ — "types" BIRINCHI
{ "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.mjs" } } }
```

Kompilator birinchi mos keladigan condition ni oladi. `"types"` birinchi bo'lmasa — `.mjs` faylni oladi va `.d.ts` ni topolmaydi.

### 5. `declare` li declaration runtime da mavjud emas

```typescript
declare const __DEV__: boolean;

if (__DEV__) {
  console.log("debug mode");
}

// TS — ✅ compile bo'ladi
// Runtime — ❌ ReferenceError: __DEV__ is not defined
// (agar Webpack DefinePlugin yoki shunga o'xshash tool set qilmagan bo'lsa)
```

`declare` faqat kompilatorga signal beradi. Runtime da bu qiymatni boshqa joydan (bundler, CDN script) ta'minlash — developer ning mas'uliyati.

---

## Common Mistakes

### ❌ Xato 1: `.d.ts` faylda implementation yozish

```typescript
// ❌ — .d.ts da function body bo'lishi mumkin emas
export function formatDate(date: Date): string {
  return date.toISOString(); // ❌ Error!
}

// ✅ Faqat signature
export declare function formatDate(date: Date): string;
```

### ❌ Xato 2: `@types` package va library o'z type larini birga install qilish

```bash
npm install axios         # o'z type lari bor
npm install @types/axios  # ❌ — kerak emas!
```

**Nima uchun:** Ikki turli type definition conflict yaratadi.

### ❌ Xato 3: `isolatedDeclarations` da explicit type yozmaslik

```typescript
// ❌ — return type inference kerak
export function getUsers() {
  return fetch("/api/users").then(r => r.json());
}

// ✅ — explicit return type
export async function getUsers(): Promise<User[]> {
  return fetch("/api/users").then(r => r.json());
}
```

### ❌ Xato 4: `exports` da `types` ni oxirida yozish

```json
// ❌
{ "exports": { ".": { "import": "./dist/index.mjs", "types": "./dist/index.d.ts" } } }

// ✅ — "types" BIRINCHI
{ "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.mjs" } } }
```

### ❌ Xato 5: Module augmentation da `import` ni unutish

```typescript
// ❌ — import yo'q → ambient declaration (override!)
declare module "express" { interface Request { userId: string; } }

// ✅ — import bor → augmentation (kengaytirish)
import "express";
declare module "express" { interface Request { userId: string; } }
```

---

## Amaliy Mashqlar

### Mashq 1: Declaration File Yozish (Oson)

**Savol:** Quyidagi JS kutubxona uchun `.d.ts` yozing:

```javascript
function capitalize(str) { return str.charAt(0).toUpperCase() + str.slice(1); }
function truncate(str, length) { return str.length > length ? str.slice(0, length) + "..." : str; }
module.exports = { capitalize, truncate };
```

<details>
<summary>Javob</summary>

```typescript
declare module "string-utils" {
  export function capitalize(str: string): string;
  export function truncate(str: string, length: number): string;
}
```

</details>

---

### Mashq 2: Package Type Config (O'rta)

**Savol:** TypeScript library uchun `package.json` — ESM va CJS dual package, sub-path exports.

<details>
<summary>Javob</summary>

```json
{
  "name": "my-lib",
  "version": "1.0.0",
  "main": "./dist/cjs/index.cjs",
  "module": "./dist/esm/index.mjs",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/esm/index.mjs",
      "require": "./dist/cjs/index.cjs"
    },
    "./utils": {
      "types": "./dist/types/utils.d.ts",
      "import": "./dist/esm/utils.mjs",
      "require": "./dist/cjs/utils.cjs"
    }
  },
  "files": ["dist"]
}
```

</details>

---

### Mashq 3: Generic Library Declaration (Qiyin)

**Savol:** Type-safe event emitter kutubxonasi uchun declaration yozing.

<details>
<summary>Javob</summary>

```typescript
declare module "typed-emitter" {
  type EventMap = Record<string, (...args: any[]) => void>;

  export class TypedEmitter<Events extends EventMap> {
    on<K extends keyof Events>(event: K, listener: Events[K]): this;
    once<K extends keyof Events>(event: K, listener: Events[K]): this;
    emit<K extends keyof Events>(event: K, ...args: Parameters<Events[K]>): boolean;
    off<K extends keyof Events>(event: K, listener: Events[K]): this;
  }

  export default TypedEmitter;
}
```

</details>

---

### Mashq 4: `isolatedDeclarations` Migration (O'rta)

**Savol:** Quyidagi kodni `isolatedDeclarations: true` bilan ishlashi uchun tuzating:

```typescript
export const DEFAULT_CONFIG = { host: "localhost", port: 3000, debug: false };
export function createServer(config = DEFAULT_CONFIG) { return { config, start() {} }; }
export const handler = (req: any) => ({ status: 200, body: req.url });
```

<details>
<summary>Javob</summary>

```typescript
export interface ServerConfig { host: string; port: number; debug: boolean; }
export interface Server { config: ServerConfig; start(): void; }
export interface HandlerResult { status: number; body: string; }

export const DEFAULT_CONFIG: ServerConfig = { host: "localhost", port: 3000, debug: false };
export function createServer(config: ServerConfig = DEFAULT_CONFIG): Server {
  return { config, start() {} };
}
export const handler: (req: any) => HandlerResult = (req) => ({ status: 200, body: req.url });
```

</details>

---

### Mashq 5: `skipLibCheck` Diagnostika (Oson)

**Savol:** Loyihangizda `@types/jest` va `@types/mocha` ikkalasi ham install. Ikkalasi `describe` va `it` global function larni turli type lar bilan declare qiladi → conflict. Qanday hal qilasiz?

<details>
<summary>Javob</summary>

```json
{
  "compilerOptions": {
    "types": ["jest"]
  }
}
```

`types` array bilan faqat `@types/jest` ni include qilish. `@types/mocha` ni butunlay uninstall qilish ham yaxshi:

```bash
npm uninstall @types/mocha
```

**Tushuntirish:** `types` array belgilanganda kompilator **faqat** ko'rsatilgan `@types` package larni avtomatik include qiladi. Qolganlarni ignore qiladi. Bu global type pollution va conflict larni oldini oladi.

</details>

---

## Xulosa

Bu bo'limda TypeScript declaration file system i o'rganildi:

**Declaration files asoslari:**
- **`.d.ts`** — faqat type information, implementation yo'q. JS ga compile qilinmaydi.
- **`declare` keyword** — `const`, `function`, `class`, `module`, `namespace`, `global`, `enum`
- **`.d.ts` vs `.ts`** — declaration da body yo'q, faqat signature va type.

**Yaratish usullari:**
- **Avtomatik** — `declaration: true`. `declarationMap` — IDE source navigation. `emitDeclarationOnly` — bundler bilan.
- **`isolatedDeclarations` (TS 5.5+)** — fayl-bo'yicha, explicit types majburiy. Monorepo performance.
- **Qo'lda** — JS kutubxona, global script, non-standard module lar uchun.

**Ecosystem:**
- **DefinitelyTyped** — `@types/*` package lar
- **`typeRoots`** va **`types`** — qaysi type definition lar include qilinishini boshqarish
- **`exports` da `"types"` birinchi** bo'lishi kerak

**Declaration turlari:**
- **Ambient module** — `declare module "name"`
- **Wildcard module** — `declare module "*.css"`
- **Global** — `declare global { ... }`
- **UMD** — `export as namespace`

**Testing:** `tsd`, `@arethetypeswrong/cli`, `dtslint`

**Bog'liq bo'limlar:**
- [Bo'lim 17: Modules](17-modules.md) — module augmentation, ambient modules, `import type`
- [Bo'lim 4: Objects va Interfaces](04-objects-interfaces.md) — declaration merging
- [Bo'lim 22: tsconfig.json](22-tsconfig.md) — declaration-related options

---

**Keyingi bo'lim:** [19-decorators.md](19-decorators.md) — Decorators: legacy va TC39 Stage 3 standartlari, class/method/property/accessor decorators, decorator factories, composition, execution order, va real-world patterns.
