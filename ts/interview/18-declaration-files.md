# Interview: Declaration Files

> `.d.ts` fayllar, `declare` keyword, ambient declarations, `declaration` emit, `declarationMap`, `isolatedDeclarations` (TS 5.5+), DefinitelyTyped, `@types/*`, `typeRoots` va `types`, triple-slash directives, declaration testing (`tsd`, `attw`), library publishing bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar) (1-12)
- [Output savollari](#output-savollari) (13-17)
- [Coding challenges](#coding-challenges) (18-21)
- [Bug fix savollari](#bug-fix-savollari) (22-23)

---

## Nazariy savollar

### Savol 1: Declaration file (`.d.ts`) nima va nima uchun kerak? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`.d.ts` — JavaScript kodiga **type information** beradigan fayl. Faqat type'lar, signature'lar — implementation (function body, class method logic) **yo'q**. TypeScript bu fayllarni JS code'ga type safety qo'shish uchun o'qiydi.

### To'liq tushuntirish

Declaration file ikki maqsadda:

1. **JavaScript kutubxonalarni TypeScript da ishlatish** — kutubxona JS'da yozilgan, lekin TS project'da type safety kerak
2. **TypeScript library distribution** — library `.ts` source'dan `.js` + `.d.ts` ga compile, package sifatida publish

**Tuzilish:**

```
Source files:               Compiled output:
┌──────────┐                 ┌──────────────┐
│  app.ts  │──→ compile ──→  │   app.js     │  ← runtime
│          │──→ emit d.ts ──→│   app.d.ts   │  ← type info
└──────────┘                 └──────────────┘

External package:
┌──────────────────────┐
│  express/            │
│  ├── index.js        │  ← runtime
│  └── index.d.ts      │  ← TS shu faylni o'qiydi
└──────────────────────┘
```

**Kompilatorda behavior:**

`.d.ts` fayllar JS'ga **compile qilinmaydi** — allaqachon faqat type ma'lumot. Kompilator ularni faqat type-checking va inference uchun o'qiydi.

**Qachon kerak:**

1. **JS library uchun type** — `@types/lodash`, `@types/jquery` kabi
2. **Library publish** — TS library NPM'ga `.js` + `.d.ts` bilan
3. **Global type** — `window`, `process`, `document` (lib.d.ts)
4. **Non-TS file import** — CSS, JSON, image type'lari
5. **Webpack DefinePlugin** — global constant'lar type

### Kod misol

```typescript
// utils.ts — implementation bilan
export function formatDate(date: Date): string {
  return date.toISOString().split("T")[0];
}

export interface DateFormatOptions {
  locale: string;
  timezone: string;
}
```

```typescript
// utils.d.ts — faqat type (implementation yo'q)
export declare function formatDate(date: Date): string;

export interface DateFormatOptions {
  locale: string;
  timezone: string;
}
// Function body yo'q — faqat signature
// Interface — pure type, declare kerak emas
```

**JS library uchun declaration:**

```javascript
// analytics-lib.js (JavaScript, types yo'q)
function init(config) { /* ... */ }
function track(event, props) { /* ... */ }
module.exports = { init, track };
```

```typescript
// types/analytics-lib.d.ts
declare module "analytics-lib" {
  export interface Config {
    apiKey: string;
    debug?: boolean;
  }

  export interface EventProps {
    userId?: string;
    [key: string]: unknown;
  }

  export function init(config: Config): void;
  export function track(event: string, props?: EventProps): void;
}
```

### Edge Cases

- **`.d.ts` faylda value (implementation)** — compile error. `.d.ts` faqat type construct'lar uchun.
- **Mixed `.ts` da `declare`** — `.ts` faylda `declare` ham yozish mumkin. Ambient declaration emit qilinmaydi.
- **Auto-acquisition** — VSCode/IDE `@types/*` package'larni avtomatik qidiradi va include qiladi.
- **`lib.d.ts`** — TypeScript o'zining standard library type'lari (`lib.es2022.d.ts`, `lib.dom.d.ts`). `lib` tsconfig option bilan tanlanadi.

### Follow-up savollar

1. **"`.d.ts` faylda interface va class farqi?"** — Interface — pure type. Class — runtime construct, `declare class` kerak (faqat shape, body yo'q).
2. **"`.d.ts` faylda value declare qilish mumkinmi?"** — Faqat `declare` bilan (`declare const X`, `declare function Y()`). Implementation yo'q.

</details>

---

### Savol 2: `declare` keyword qanday holatlarda ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`declare` — "bu narsa boshqa joyda mavjud, men faqat type'ini aytayapman" signal. JS'ga emit qilinmaydi. `const`, `let`, `var`, `function`, `class`, `module`, `namespace`, `global`, `enum` bilan ishlatiladi.

### To'liq tushuntirish

`declare` keyword **ambient declaration** yaratadi — type-level only construct, runtime'da mavjud emas. Compiler tekshirishlar uchun bilishi kerak, lekin implementation kelganini taxmin qiladi.

**Constructs:**

| Construct | Maqsadi |
|-----------|---------|
| `declare const/let/var` | Tashqi variable (global yoki bundler-injected) |
| `declare function` | Tashqi function |
| `declare class` | Tashqi class (shape only, body yo'q) |
| `declare module "name"` | Module type (JS library yoki non-TS file) |
| `declare namespace` | Type'lar guruhi (UMD, global) |
| `declare global` | Global scope'ga augmentation |
| `declare enum` | Tashqi enum (ko'pincha `@types`'da) |

**Emit behavior:**

`declare` bilan yozilgan har qanday construct **JS output ga kirmaydi**. Pure compile-time.

**`declare` keraksiz holatlar:**
- Interface — zaten pure type
- Type alias — zaten pure type

**`.d.ts` da `declare`:**

`.d.ts` faylda barcha top-level declarations implicitly `declare`. Lekin explicit yozish kerak `function`, `class`, `var`/`let`/`const` uchun (clarity).

**`.ts` da `declare`:**

Implementation yozilmaganini bildiradi. Compiler ishonish kerak — runtime'da boshqa joydan keladi.

### Kod misol

**`declare const`:**

```typescript
// Webpack DefinePlugin injection
declare const __APP_VERSION__: string;
declare const __DEV__: boolean;
declare const __BUILD_DATE__: string;

// jQuery (legacy)
declare const $: JQueryStatic;
declare const jQuery: typeof $;
```

**`declare function`:**

```typescript
// Built-in function polyfill
declare function structuredClone<T>(value: T): T;

// Custom global helper
declare function logger(level: "info" | "error", message: string): void;

// Overload
declare function fetch(input: string): Promise<Response>;
declare function fetch(input: string | URL, init?: RequestInit): Promise<Response>;
```

**`declare class`:**

```typescript
declare class EventEmitter {
  on(event: string, listener: (...args: any[]) => void): this;
  emit(event: string, ...args: any[]): boolean;
  off(event: string, listener: (...args: any[]) => void): this;
  removeAllListeners(event?: string): this;
}
```

**`declare module`:**

```typescript
// Untyped library
declare module "lodash" {
  export function chunk<T>(array: T[], size: number): T[][];
  export function debounce<T extends (...args: any[]) => any>(func: T, wait: number): T;
}

// Wildcard module — CSS, images
declare module "*.css" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.svg" {
  import type { FC, SVGProps } from "react";
  const SVGComponent: FC<SVGProps<SVGSVGElement>>;
  export default SVGComponent;
}
```

**`declare namespace`:**

```typescript
// UMD library — script tag bilan yuklangan
declare namespace google.maps {
  class Map {
    constructor(element: HTMLElement, options: MapOptions);
    setCenter(latLng: LatLng): void;
  }
  interface MapOptions {
    center: LatLng;
    zoom: number;
  }
  interface LatLng { lat: number; lng: number }
}
```

**`declare global`:**

```typescript
// global.d.ts
export {}; // module qilish uchun

declare global {
  interface Window {
    __STORE__: Map<string, unknown>;
    analytics: { track(event: string): void };
  }
  function sleep(ms: number): Promise<void>;
  var APP_NAME: string;
}
```

### Edge Cases

- **`declare` `.ts` faylda emit** — `.ts` faylda `declare`'li construct JS output'ga emit qilinmaydi. Pure compile-time.
- **`declare class` ichida `private` member** — `.d.ts` da private member nomi ko'rinadi, type'i yo'q (structural compat uchun).
- **`declare module` `node_modules`'dagi paket'ni almashtirish** — Augmentation context'da merge, ambient context'da almashtiradi.
- **TS 5.0+ `const type parameters`** — `declare` bilan generic'ga `const T` ishlatish mumkin.

### Follow-up savollar

1. **"`declare const` runtime'da mavjud emas — qanday yechim?"** — Webpack `DefinePlugin`, Vite `define` option, yoki CDN script tag injection.
2. **"`declare class` ichida static method qanday?"** — Oddiy `static methodName(): void`. Constructor ham declare qilinadi: `constructor(arg: Type)`.

</details>

---

### Savol 3: Avtomatik vs qo'lda declaration generate qilish [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Avtomatik — `declaration: true` tsconfig option bilan compiler `.ts` dan `.d.ts` generate qiladi (TypeScript library uchun). Qo'lda — JS library, global script, non-standard module'lar uchun ishlab chiqilgan declaration fayllar.

### To'liq tushuntirish

**Avtomatik generation:**

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

| Option | Behavior |
|--------|----------|
| `declaration: true` | `.d.ts` fayllar generate qilinadi |
| `declarationDir` | `.d.ts` qayerga yoziladi |
| `declarationMap: true` | `.d.ts.map` — IDE "Go to Definition" da source'ga |
| `emitDeclarationOnly: true` | Faqat `.d.ts`, `.js` emas (bundler JS yaratadi) |

**Mexanizm:**

Compiler ikki parallel emit ishlaydi:
1. JS output — function body, implementation
2. `.d.ts` output — **implementation erasure**: body olib tashlanadi, signature saqlanadi

Inferred type'lar avtomatik hisoblanib `.d.ts` ga yoziladi. Lekin `isolatedDeclarations: true` da explicit type kerak.

**`private` member behavior:**

```typescript
// Source
class User {
  private password: string;
  constructor(public name: string, password: string) {
    this.password = password;
  }
}

// Generated .d.ts
declare class User {
  private password; // ← type YO'Q
  name: string;
  constructor(name: string, password: string);
}
```

`private` member type'i ko'rinmaydi — structural compatibility uchun faqat **mavjudligi** ko'rsatiladi.

**Qo'lda declaration:**

Kerak holatlar:
1. **JavaScript library** — source TS emas, `@types`'da ham yo'q
2. **Global script** — CDN'dan yuklangan
3. **Non-standard module** — CSS, image, font fayllar
4. **Type augmentation** — mavjud library type'larini kengaytirish

**Qo'lda yozilgan `.d.ts` runtime API'ga mosligi compiler tomonidan tekshirilmaydi.** Noto'g'ri type → runtime error.

### Kod misol

**Avtomatik — TypeScript library:**

```typescript
// src/index.ts
export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount);
}

export interface FormatOptions {
  locale: string;
  minimumFractionDigits: number;
}

export class Formatter {
  private locale: string;
  constructor(locale: string) {
    this.locale = locale;
  }
  format(value: number): string {
    return value.toLocaleString(this.locale);
  }
}
```

`tsc` natijasi:

```typescript
// dist/index.d.ts
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

`package.json`:

```json
{
  "name": "my-formatter",
  "version": "1.0.0",
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

**Qo'lda — JS library uchun:**

```typescript
// types/payment-gateway.d.ts
declare module "payment-gateway" {
  export interface PaymentConfig {
    apiKey: string;
    environment: "sandbox" | "production";
    timeout?: number;
  }

  export interface PaymentRequest {
    amount: number;
    currency: string;
    description?: string;
  }

  export interface PaymentResult {
    transactionId: string;
    status: "pending" | "approved" | "declined";
    timestamp: string;
  }

  export class PaymentClient {
    constructor(config: PaymentConfig);
    charge(request: PaymentRequest): Promise<PaymentResult>;
    refund(transactionId: string): Promise<PaymentResult>;
  }

  export default function createClient(config: PaymentConfig): PaymentClient;
}
```

**Global script — CDN library:**

```typescript
// types/google-tag-manager.d.ts
declare function gtag(command: "event", eventName: string, params?: Record<string, unknown>): void;
declare function gtag(command: "config", targetId: string, params?: Record<string, unknown>): void;
declare function gtag(command: "set", params: Record<string, unknown>): void;

declare const dataLayer: Array<Record<string, unknown>>;
```

### Edge Cases

- **`declaration: true` xato bilan inferred type** — Inferred return type ba'zan source dependency'larga reference qiladi. Library publish'da explicit type yozish ishonchli.
- **`stripInternal` option** — `/** @internal */` JSDoc bilan member'lar `.d.ts` da o'chiriladi.
- **`removeComments`** — `.d.ts` da JSDoc saqlanadi default (intellisense uchun). `removeComments: true` bilan o'chiriladi.
- **Manual `.d.ts` API drift** — JS library yangilanganda manual `.d.ts` eskirib qoladi. Versioning muhim.

### Follow-up savollar

1. **"`declarationMap` ishlatish qachon foydali?"** — Library development, monorepo (cross-package navigate), debugging. Production'da o'chirish mumkin (qo'shimcha fayl).
2. **"`emitDeclarationOnly` qachon ishlatiladi?"** — Bundler (esbuild, Rollup, Vite) JS yaratganda, faqat `.d.ts` kerak.

</details>

---

### Savol 4: `isolatedDeclarations` (TS 5.5+) nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedDeclarations` — har fayl declaration'ni boshqa fayllar type ma'lumotisiz generate qilishni majburlaydi. Tezkor tool'lar (esbuild, SWC) parallel `.d.ts` emit qilish imkonini beradi. Exported function/variable'larda **explicit type** majburiy.

### To'liq tushuntirish

**Standard `declaration: true` muammosi:**

Kompilator inferred return type'larni `.d.ts` ga yozadi. Bu cross-file type inference talab qiladi — butun project tahlil kerak. Sekin va parallelization yo'q.

**`isolatedDeclarations: true` yechim:**

Har fayl `.d.ts` ni **mustaqil** generate qila olishi kerak. Cross-file inference yo'q. Cheklov: exported declaration'lar explicit type'ga ega bo'lishi shart.

| Aspect | `declaration: true` | `isolatedDeclarations: true` |
|--------|---------------------|------------------------------|
| Speed | Sekin (butun project) | Tez (per-file) |
| Parallelization | ❌ | ✅ |
| esbuild/SWC integration | ❌ | ✅ |
| Explicit return type | Ixtiyoriy | Majburiy (exported) |
| Explicit variable type | Ixtiyoriy | Majburiy (exported) |
| Local code constraint | Yo'q | Yo'q (faqat exported) |

**Mexanizm:**

Tezkor transpiler'lar (esbuild, SWC) `.d.ts` emit qila olish uchun har fayl ni izolyatsiyada ko'rishi kerak. `isolatedDeclarations` shu cheklovni TypeScript tomonida enforce qiladi.

**Qachon kerak:**
- **Monorepo** — ko'p package, parallel build
- **Library publishing** — esbuild bilan tez build
- **CI/CD pipeline** — declaration emit'ni parallel qilish

### Kod misol

```json
{
  "compilerOptions": {
    "isolatedDeclarations": true,
    "declaration": true
  }
}
```

**Error misollar:**

```typescript
// ❌ Return type yo'q
export function add(a: number, b: number) {
  return a + b;
}

// ✅ Explicit return type
export function add(a: number, b: number): number {
  return a + b;
}

// ❌ Variable type inference
export const config = { host: "localhost", port: 3000 };

// ✅ Explicit type
interface ServerConfig { host: string; port: number }
export const config: ServerConfig = { host: "localhost", port: 3000 };

// ❌ Async function return inference
export async function getUser(id: number) {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

// ✅ Explicit return type
export async function getUser(id: number): Promise<User> {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

// ❌ Class method return inference
export class UserService {
  getAll() { return []; }
}

// ✅ Explicit
export class UserService {
  getAll(): User[] { return []; }
}
```

**Local code cheklanmaydi:**

```typescript
// Local — exported emas, cheklov yo'q
const internal = { x: 1, y: 2 };
function helper(n: number) { return n * 2; }

// Exported — explicit type majburiy
interface PublicConfig { x: number; y: number }
export const publicConfig: PublicConfig = internal;
export function publicHelper(n: number): number {
  return helper(n);
}
```

**Library example:**

```typescript
// math-lib.ts
export interface Vector2D { x: number; y: number }
export interface Vector3D extends Vector2D { z: number }

export function add2D(a: Vector2D, b: Vector2D): Vector2D {
  return { x: a.x + b.x, y: a.y + b.y };
}

export function add3D(a: Vector3D, b: Vector3D): Vector3D {
  return { x: a.x + b.x, y: a.y + b.y, z: a.z + b.z };
}

export const ZERO_2D: Vector2D = { x: 0, y: 0 };
export const ZERO_3D: Vector3D = { x: 0, y: 0, z: 0 };

export class Calculator {
  private precision: number;
  
  constructor(precision: number = 2) {
    this.precision = precision;
  }
  
  round(value: number): number {
    return Number(value.toFixed(this.precision));
  }
}
```

### Edge Cases

- **Implicit `any` return** — `function f() {}` `void` deb infer qilinadi. `isolatedDeclarations`'da explicit `: void` kerak.
- **Class private member type** — `private password: string` — explicit yozish, lekin `.d.ts`'da nom faqat ko'rinadi.
- **Re-export** — `export { X } from "./mod"` — `X` original module'da explicit bo'lishi kerak.
- **Generic constraint inference** — `function f<T>(x: T) { ... }` return type'i `T` bo'lsa, kerak: `: T`.

### Follow-up savollar

1. **"Qaysi tool `isolatedDeclarations` bilan `.d.ts` emit qiladi?"** — `tsc --isolatedDeclarations` reference implementation. SWC (`fast_dts`) va OXC type-checker'siz, per-file `.d.ts` emit qiladi. esbuild o'zi `.d.ts` chiqarmaydi — plugin yoki alohida `tsc`/`tsgo` qadami orqali generate qilinadi.
2. **"Codebase'ni `isolatedDeclarations` ga migration qanday?"** — `tsc --isolatedDeclarations --noEmit` bilan birinchi error'larni topish. `@typescript-eslint/explicit-module-boundary-types` rule bilan exported declaration'larda explicit type'ni enforce qilish. Bosqichma-bosqich fayllar.

</details>

---

### Savol 5: DefinitelyTyped (`@types/*`) qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

DefinitelyTyped — community type definitions repository ([github.com/DefinitelyTyped/DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped)). JavaScript library uchun type'lar `@types/` scope'da NPM package sifatida publish qilinadi.

### To'liq tushuntirish

**Workflow:**

1. JS library (express, lodash, react va boshqa) o'zida type yo'q
2. Community DefinitelyTyped repository'da `types/<package>` folder yaratadi
3. CI'dan o'tib NPM'ga `@types/<package>` sifatida publish bo'ladi
4. Developer'lar `npm install --save-dev @types/<package>` qiladi

**Type qidiruv tartibi:**

```
1. Package o'zida:
   package.json → "types": "./dist/index.d.ts"
   package.json → "exports" → "types" conditional

2. @types package:
   node_modules/@types/<package-name>/index.d.ts

3. typeRoots (tsconfig):
   Belgilangan papkalar ichida qidirish
```

Kompilator import statement uchun bu tartibni tekshiradi. Birinchi mos keladigan source ishlatiladi.

**Versioning:**

DefinitelyTyped konvensiyasi: `@types/<pkg>` ning major.minor versiyasi u type qilayotgan library major.minor'iga moslashtiriladi. Lekin har minor bump'da yangilanishi shart emas — `@types` library'dan orqada qolishi mumkin:
- `express@^4.18.0` → `@types/express@^4.17.0` (`@types` 4.17 versiyaga yozilgan, lekin 4.18 API'ni qamrab oladi)
- `react@^18.2.0` → `@types/react@^18.2.0`

`@types` ning patch versiyasi mustaqil — type fix'lar uchun ko'tariladi.

**`@types` kerak emas, agar:**
- Library o'zida type'lar bor (`package.json "types"` field)
- Library TypeScript'da yozilgan (avtomatik type)

Ikkalasi install qilinsa → conflict.

### Kod misol

```bash
# Library install
npm install express lodash axios

# Type'lar — alohida
npm install --save-dev @types/express @types/lodash
# @types/axios — KERAK EMAS! axios'da o'z type'lari bor
```

```json
// package.json
{
  "dependencies": {
    "express": "^4.18.2",
    "lodash": "^4.17.21",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/lodash": "^4.14.202",
    "@types/node": "^20.10.0"
    // @types/axios — yo'q
  }
}
```

**Foydalanish:**

```typescript
// Type'lar avtomatik mavjud
import express, { Request, Response } from "express";
import { chunk, debounce } from "lodash";
import axios from "axios"; // axios o'z type'lari

const app = express();

app.get("/users", (req: Request, res: Response) => {
  const groups = chunk([1, 2, 3, 4, 5], 2);
  res.json({ groups });
});

const search = debounce((query: string) => {
  return axios.get(`/api/search?q=${query}`);
}, 300);
```

**Custom typings — local override:**

Agar `@types` eskirgan yoki noto'g'ri bo'lsa:

```typescript
// types/custom-express.d.ts
import "express";

declare module "express-serve-static-core" {
  interface Request {
    customField: string; // local augmentation
  }
}
```

```json
// tsconfig.json
{
  "include": ["src", "types"]
}
```

### Edge Cases

- **Library va `@types` versiyasi mismatch** — eski library + yangi `@types` → kutilmagan API'lar type'da bor.
- **Side-by-side install** — `@types/jest` va `@types/mocha` ikkalasi installed → global `describe`/`it` conflict. `types` array bilan tanlash.
- **`@types/node` katta** — barcha Node.js API'lari type'lari. Browser-only project'larda `types: []` bilan exclude qilish.
- **DT submission flow** — DefinitelyTyped CI tomonidan tekshiriladi, owner approve qiladi.

### Follow-up savollar

1. **"`@types/<library>` qanday yaratiladi?"** — DefinitelyTyped repository'ga PR. Masalan `lodash` uchun: `types/lodash/index.d.ts`, `types/lodash/lodash-tests.ts`, `types/lodash/tsconfig.json` kabi struktura.
2. **"Library `@types` o'rniga bundle qilinishi kerakmi?"** — Modern best practice — library o'zi `.d.ts` qo'shish. `@types` external nightly maintenance — versiya drift xavfi.

</details>

---

### Savol 6: `typeRoots` va `types` farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`typeRoots` — type qidiriladigan **papkalar**. `types` — automatic include qilinadigan aniq **package'lar**. `typeRoots` qayerdan, `types` qaysi narsalarni belgilaydi.

### To'liq tushuntirish

**`typeRoots`:**

Type definitions qidiriladigan root papkalar. Default: `["./node_modules/@types"]`.

```json
{
  "compilerOptions": {
    "typeRoots": [
      "./node_modules/@types",
      "./custom-types"
    ]
  }
}
```

Behavior:
- `typeRoots` belgilangan papkalar ichidagi har bir folder = package
- Folder'lar avtomatik include qilinadi (`types` belgilanmagan bo'lsa)
- `typeRoots: []` — default qidiruvni o'chiradi

**`types`:**

Faqat belgilangan aniq package'lar avtomatik include qilinadi:

```json
{
  "compilerOptions": {
    "types": ["node", "vitest/globals"]
  }
}
```

Behavior:
- Faqat `@types/node` va `vitest/globals` include. `types` entry — `@types/<name>` package yoki package'ning o'zi (`vitest/globals` — `vitest` package ichidagi global declaration subpath, `@types`'da emas)
- Boshqa `@types/*` ignore (tsconfig perspective'dan)
- Lekin `import` statement orqali ishlatib bo'ladi

**Conflict scenario:**

```bash
npm install --save-dev @types/jest @types/mocha
```

Ikkalasi `describe`, `it`, `expect` global function'larni declare qiladi → type conflict.

Yechim:

```json
{ "compilerOptions": { "types": ["jest"] } }
```

`types` belgilanganda compiler faqat `@types/jest` ni include qiladi. `@types/mocha` ignore.

**Eng yaxshi praktika:**

| Holat | Konfiguratsiya |
|-------|----------------|
| Default | Hech nima — `typeRoots: ["./node_modules/@types"]` ishlaydi |
| Custom types folder | `typeRoots: ["./node_modules/@types", "./types"]` |
| Global pollution control | `types: ["node", "vitest"]` |
| `@types/*` ishlatmaslik | `types: []` |

### Kod misol

**Custom types folder:**

```
project/
├── src/
│   └── app.ts
├── types/
│   └── custom-globals.d.ts
└── tsconfig.json
```

```json
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./types"]
  },
  "include": ["src"]
}
```

```typescript
// types/custom-globals.d.ts
declare const APP_VERSION: string;
declare const APP_BUILD_DATE: string;
```

```typescript
// src/app.ts
console.log(APP_VERSION);  // ✅ — custom-globals'dan
console.log(APP_BUILD_DATE); // ✅
```

**Conflict resolution:**

```json
// Jest project
{
  "compilerOptions": {
    "types": ["node", "jest"]
  }
}
```

```json
// Vitest project (without Jest)
{
  "compilerOptions": {
    "types": ["node", "vitest/globals"]
  }
}
```

**Pure Node.js (no DOM types):**

```json
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "types": ["node"]
  }
}
```

DOM type'lar (Window, Document, fetch) include qilinmaydi.

### Edge Cases

- **`types: []` (bo'sh)** — Hech qanday `@types/*` avtomatik include emas. `import`'da ishlatish mumkin.
- **`include` bilan farq** — `include` source file'lar. `types` automatic type acquisition. Ikki turli mexanizm.
- **`types` ichida custom folder** — `types: ["./custom"]` ishlamaydi. `types` faqat package nom (`@types/<name>` yoki package o'zi).
- **TypeScript automatic type acquisition** — VSCode default'da package'lar uchun `@types/*` qidiradi. tsconfig'da `types: []` bilan ham hozir ishlaydi (IDE feature).

### Follow-up savollar

1. **"`typeRoots` va `paths` farqi?"** — `typeRoots` type definition root'lar. `paths` import alias mapping. Ikkalasi har xil mexanizm.
2. **"`types: ["my-lib"]` lekin `my-lib` `@types`'da yo'q — error?"** — Compiler error: "Cannot find type definition file for 'my-lib'". `@types/my-lib` install yoki `types`'dan olib tashlash kerak.

</details>

---

### Savol 7: `package.json` `exports` va `types` tartibi nima uchun muhim? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`package.json` `exports` ichida `"types"` conditional **birinchi** bo'lishi kerak. Compiler har condition uchun match qilingan JS faylning type-equivalent declaration'ini qidiradi (`.mjs` → `.d.mts`). `"types"` oxirida bo'lsa, type ko'pincha fallback orqali topiladi — lekin noto'g'ri format declaration ishlatilib qoladi (Masquerading), `@arethetypeswrong/cli` `FallbackCondition` warning beradi.

### To'liq tushuntirish

**`package.json "exports" field:**

Modern Node.js (16+) va TypeScript (4.7+) `exports` field'ni qo'llab-quvvatlaydi. Conditional exports — har xil context uchun har xil fayl:

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

Conditional'lar:
- `"types"` — TypeScript compiler
- `"import"` — ESM import
- `"require"` — CommonJS require
- `"node"` — Node.js runtime
- `"browser"` — browser environment
- `"default"` — fallback

**Critical fakt:** Resolution **birinchi mos keladigan** condition'ni oladi. Tartib muhim.

**Noto'g'ri tartib:**

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"  // ← oxirda
    }
  }
}
```

TypeScript compiler:
1. `"import"` → `.mjs` → yonida `.d.mts` qidiradi → topilmasa, keyingi condition'larga fallback qiladi
2. Oxirida `"types"` (`.d.ts`) ga fallback orqali yetadi — type topiladi, lekin `.d.ts` ESM `.mjs` uchun ishlatiladi (Masquerading). `@arethetypeswrong/cli` buni `FallbackCondition` deb belgilaydi

**To'g'ri tartib:**

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",  // ← BIRINCHI
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

TypeScript:
1. `"types"` ni topadi → `.d.ts` ga point qiladi ✅
2. Type'larni o'qiydi

Node.js (runtime):
1. `"types"` ignore (Node.js bilmaydi)
2. `"import"` yoki `"require"` ga qarab tanlanadi

**Tekshirish:**

```bash
npx @arethetypeswrong/cli ./my-package-1.0.0.tgz
# yoki published package uchun
npx @arethetypeswrong/cli --pack .
```

### Kod misol

**To'liq library `package.json`:**

```json
{
  "name": "my-lib",
  "version": "1.0.0",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./utils": {
      "types": "./dist/utils.d.ts",
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.cjs"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist"],
  "sideEffects": false
}
```

**Dual format (CJS + ESM) — alohida types:**

```json
{
  "exports": {
    ".": {
      "import": {
        "types": "./dist/esm/index.d.mts",
        "default": "./dist/esm/index.mjs"
      },
      "require": {
        "types": "./dist/cjs/index.d.cts",
        "default": "./dist/cjs/index.cjs"
      }
    }
  }
}
```

Bu yondashuv ESM va CJS uchun **alohida** type fayllar — `.d.mts` va `.d.cts`. Modern best practice.

**Subpath exports:**

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./schemas": {
      "types": "./dist/schemas/index.d.ts",
      "default": "./dist/schemas/index.js"
    },
    "./schemas/*": {
      "types": "./dist/schemas/*.d.ts",
      "default": "./dist/schemas/*.js"
    }
  }
}
```

Foydalanish:

```typescript
import { Client } from "my-lib";
import { UserSchema } from "my-lib/schemas";
import { ProductSchema } from "my-lib/schemas/product";
```

### Edge Cases

- **`"types"` legacy field** — `package.json` top-level `"types"` field (eski Node.js'larga uchun). `"exports"` bilan ikkalasi ham bo'lsa, `"exports"` priority.
- **`"main"` field** — Legacy CJS entry point. Modern setup'da `"exports"` to'liq almashtirib bo'ladi, lekin Node.js eski versiyalarga `"main"` saqlanadi.
- **Wildcard pattern** — `"./schemas/*"` glob pattern. Har bir export uchun ishlatilmasin (file'lar avtomatik resolve).
- **`@arethetypeswrong/cli`** — Type distribution muammolarini tekshirish CLI. CI'da ishlatish tavsiya.

### Follow-up savollar

1. **"`moduleResolution: "node"` `exports`'ni qaramaydi — qachon muammo?"** — Legacy code base'lar yoki eski tsconfig. Modern setup'da `node16`/`nodenext`/`bundler`.
2. **"Dual package hazard nima?"** — Bitta library ikkita instance (ESM va CJS) sifatida bundle qilinishi. State'li library'larda muammoli.

</details>

---

### Savol 8: Ambient declarations vs Module augmentation — farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Ambient declaration** — yangi (mustaqil) declaration yaratish. **Module augmentation** — mavjud declaration'ni kengaytirish (merging). Farq: ambient declaration script faylda, module augmentation module faylda (`import`/`export` bilan).

### To'liq tushuntirish

**Ambient declaration:**

Implementation'siz, faqat type beruvchi declaration. Script faylda yozilgan `declare module "name"` yangi/mustaqil module declaration.

```typescript
// types/untyped-lib.d.ts (script — import/export YO'Q)
declare module "untyped-lib" {
  export function doSomething(): void;
}
```

Compiler bu fayl ni "untyped-lib module mavjud va shu shape'da" deb biladi. Boshqa declaration'lar bilan **merge bo'lmaydi**.

**Module augmentation:**

Mavjud module'ga property/method qo'shish (declaration merging orqali). Module faylda (`import`/`export` bor):

```typescript
// types/express-augment.d.ts (module — import bor)
import "express"; // ← faylni MODULE qiladi

declare module "express-serve-static-core" {
  interface Request {
    userId: string;
  }
}
```

Endi `declare module` — augmentation. Mavjud Request interface'ga `userId` qo'shiladi.

**Mexanizm:**

| Aspekt | Ambient | Augmentation |
|--------|---------|--------------|
| Fayl konteksti | Script | Module |
| `import`/`export` bor | ❌ | ✅ |
| Behavior | Yangi declaration | Mavjudga merge |
| Mavjud module bilan | Override harakati | Kengayadi |
| Conflict | Mavjud yo'qolishi mumkin | Bir-biriga qo'shiladi |

**Texnik jihatdan:** Ambient `declare module "name"` script faylda — agar shu nomdagi module `node_modules`'da declaration bilan mavjud bo'lsa, resolution context'iga qarab birinchi topilgan declaration ishlatiladi. Predictiv merge yo'q. Augmentation (module fayl) — guaranteed declaration merging.

### Kod misol

**Ambient declaration — JS library type:**

```typescript
// types/legacy-lib.d.ts (script)
declare module "legacy-lib" {
  export interface Config {
    apiKey: string;
    timeout?: number;
  }

  export function init(config: Config): void;
  export function track(event: string): void;
}
```

```typescript
// app.ts
import { init, track, Config } from "legacy-lib";

const cfg: Config = { apiKey: "..." };
init(cfg);
track("page_view");
```

**Wildcard ambient — CSS/asset import:**

```typescript
// types/assets.d.ts (script)
declare module "*.module.css" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.svg" {
  import type { FC, SVGProps } from "react";
  const SVGComponent: FC<SVGProps<SVGSVGElement>>;
  export default SVGComponent;
}

declare module "*.png" {
  const src: string;
  export default src;
}
```

```typescript
// app.ts
import styles from "./Button.module.css";   // type: { [key: string]: string }
import Logo from "./logo.svg";                // type: FC<SVGProps>
import banner from "./banner.png";            // type: string
```

**Module augmentation — Express Request:**

```typescript
// types/express-augment.d.ts (module — import bor)
import "express";

declare module "express-serve-static-core" {
  interface Request {
    userId: string;
    user: { id: string; role: "admin" | "user" } | null;
    requestId: string;
  }
}
```

```typescript
// middleware.ts
import { Request } from "express";

function authMiddleware(req: Request) {
  req.userId = "user-1";    // ✅ — augmented
  req.user = null;            // ✅
  req.body;                   // ✅ — original
  req.params;                 // ✅ — original
}
```

**Day.js plugin augmentation:**

```typescript
// types/dayjs-plugins.d.ts (module)
import "dayjs";

declare module "dayjs" {
  interface Dayjs {
    fromNow(withoutSuffix?: boolean): string; // relativeTime plugin
    tz(timezone?: string): Dayjs;              // timezone plugin
  }
}
```

### Edge Cases

- **Augmentation `interface` only** — Class va type alias declaration merging cheklangan. Augmentation interface va namespace bilan reliable.
- **Ambient declaration `import` bilan** — Module fayl ichida `declare module "name"` augmentation bo'ladi. Ambient effect (override) faqat script faylda.
- **Wildcard ambient with conflict** — `declare module "*.css"` ko'p marta declare qilinsa — birinchisini oladi.
- **`tsconfig.json include`** — Augmentation fayli `include` da bo'lishi shart.

### Follow-up savollar

1. **"Augmentation tartibi muhimmi?"** — Yo'q. TypeScript compile time'da barcha augmentation'larni topib merge qiladi. Lekin runtime'da consumer kod augmentation'siz ko'rishi mumkin (kim qachon load qiladi).
2. **"`global` augmentation script faylda ishlaydi?"** — Yo'q. `declare global` faqat module faylda. `export {}` bilan module qilish kerak.

</details>

---

### Savol 9: Triple-slash directives qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Triple-slash directive (`/// <reference ... />`) — fayl boshida yoziladigan maxsus comment. Kompilatorga qo'shimcha type fayllarni include qilishni aytadi. Zamonaviy TS'da kam ishlatiladi (`import` afzal), lekin `.d.ts` fayllarida va global type dependency'larda hali kerak.

### To'liq tushuntirish

Triple-slash directive — TypeScript-specific comment, parser tomonidan special handle qilinadi. Standart TS comment emas.

**Mavjud directive'lar:**

| Directive | Maqsadi |
|-----------|---------|
| `/// <reference path="..." />` | Boshqa `.d.ts` fayl include |
| `/// <reference types="..." />` | `@types/*` package include |
| `/// <reference lib="..." />` | Built-in lib (es2020, dom) include |
| `/// <reference no-default-lib="true" />` | Default lib o'chirish (custom env) |

**Critical cheklov:** Triple-slash directive'lar **faqat fayl boshida** ishlaydi — har qanday actual statement (import, declaration) dan **oldin**. O'rtada yoki oxirida yozilsa — oddiy comment sifatida ignore.

**Modern alternative — `import`:**

```typescript
// Eski (triple-slash)
/// <reference types="node" />

// Yangi (import)
import type { Buffer } from "node:buffer";
```

**Qachon hali kerak:**

1. **`.d.ts` fayl** — `import` ishlatish mumkin emas (faqat type), lekin global dependency kerak
2. **Global lib augmentation** — `declare global` ichida boshqa type'lardan foydalanish
3. **Lib override** — fayl-bo'yicha `lib` ni o'zgartirish
4. **Legacy code** — eski codebase'da hali ishlatiladigan

### Kod misol

**Path reference:**

```typescript
// app.ts
/// <reference path="./global-types.d.ts" />

// Endi global-types.d.ts'dagi declaration'lar mavjud
console.log(APP_VERSION); // global-types.d.ts'da declare qilingan
```

**Types reference (Node.js):**

```typescript
// utils.d.ts (faqat type fayl)
/// <reference types="node" />

declare module "my-fs-helper" {
  import type { PathLike } from "node:fs";
  
  export function readFileSafe(path: PathLike): string;
}
```

**Lib reference:**

```typescript
// browser-polyfill.d.ts
/// <reference lib="dom" />
/// <reference lib="es2020" />

declare global {
  interface Window {
    customPolyfill: () => void;
  }
}

export {};
```

**No-default-lib (custom environment):**

```typescript
// custom-env.d.ts
/// <reference no-default-lib="true" />
/// <reference lib="es5" />

// faqat ES5 lib, DOM yo'q
```

**`.d.ts`'da global type:**

```typescript
// types/google-analytics.d.ts
/// <reference types="gtag.js" />
// "gtag.js" — bare nom, compiler @types/gtag.js paketiga resolve qiladi

declare global {
  interface Window {
    dataLayer: Record<string, unknown>[];
  }
}

export {};
```

### Edge Cases

- **Fayl boshida bo'lishi shart** — Birinchi statement'dan oldin. Comment, JSDoc'dan oldin.
- **`no-default-lib`** — `lib.d.ts` o'chiriladi. Custom embedded environment uchun.
- **`@types/*` resolution** — `/// <reference types="X" />` `@types/X` ni `typeRoots` orqali resolve qiladi.
- **Performance** — Triple-slash dependency butun project type checking ga ta'sir qiladi (har fayl o'qiladi).

### Follow-up savollar

1. **"`import` bilan farq nima — har ikkalasi ham include qiladi?"** — `import` runtime + type. Triple-slash faqat type-checking. Triple-slash'da runtime emit yo'q.
2. **"`tsconfig.json types` array vs triple-slash farqi?"** — `types` array global (butun project). Triple-slash per-fayl. Granular control kerak bo'lganda triple-slash.

</details>

---

### Savol 10: Declaration file testing — `tsd` va `attw` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Library `.d.ts` fayllar production package bilan birga chiqariladi — type'lar to'g'ri ekanligini testlash kerak. **`tsd`** — assertion-based type test. **`@arethetypeswrong/cli` (`attw`)** — package type distribution muammolarini topish.

### To'liq tushuntirish

**Nima uchun testlash:**

Noto'g'ri `.d.ts` foydalanuvchilarga:
- Type error (compile fail)
- Noto'g'ri type (silent type unsafe code)
- Missing exports
- Wrong overloads

Runtime tests'dan farqli, type tests **compile-time assertion**.

**`tsd` — type assertions:**

```typescript
// index.test-d.ts
import { expectType, expectError, expectAssignable } from "tsd";
import { createUser, User } from "./index.js";

const user = createUser("Ali", 25);
expectType<User>(user);
expectType<string>(user.name);
expectType<number>(user.age);

expectError(createUser(123, "twenty-five")); // arg type xato
expectError(createUser("Ali"));               // arg count kam
```

Komand:
```bash
npx tsd
```

`tsd` natijasi compile errors sifatida ko'rsatiladi.

**`@arethetypeswrong/cli` (`attw`):**

Package type distribution muammolarini topish. Multi-config check:
- CommonJS resolution
- ESM resolution
- Bundler resolution
- Conditional exports
- `package.json` field'lari

```bash
# Local
npx @arethetypeswrong/cli --pack .

# Published
npx @arethetypeswrong/cli ./my-lib-1.0.0.tgz
```

Natija — har bir resolution context'da type topilishi, conflict'lar, missing exports.

**`dtslint` — DefinitelyTyped uchun:**

```bash
npx dtslint types/my-package
```

DefinitelyTyped'da `@types/*` PR'lari CI'da `dtslint` orqali tekshiriladi. Local'da kam ishlatiladi.

### Kod misol

**Library kodi:**

```typescript
// src/index.ts
export interface User {
  id: number;
  name: string;
  age: number;
}

export function createUser(name: string, age: number): User {
  return { id: Date.now(), name, age };
}

export function getUser(id: number): Promise<User | null> {
  return fetch(`/api/users/${id}`).then(r => (r.ok ? r.json() : null));
}

export class UserCollection<T extends User> {
  private items: T[] = [];
  
  add(item: T): void { this.items.push(item); }
  find(predicate: (item: T) => boolean): T | undefined {
    return this.items.find(predicate);
  }
}
```

**`tsd` testlari:**

```typescript
// test-d/index.test-d.ts
import { expectType, expectError, expectAssignable, expectNotAssignable } from "tsd";
import { createUser, getUser, User, UserCollection } from "../src/index.js";

// Function return type
const user = createUser("Ali", 25);
expectType<User>(user);
expectType<number>(user.id);
expectType<string>(user.name);
expectType<number>(user.age);

// Function argument errors
expectError(createUser(123, 25));            // ❌ name not number
expectError(createUser("Ali", "old"));        // ❌ age not string
expectError(createUser("Ali"));               // ❌ missing argument

// Async return
expectType<Promise<User | null>>(getUser(1));
expectError(getUser("1")); // ❌ id must be number

// Generic class
const collection = new UserCollection<User>();
expectType<UserCollection<User>>(collection);

interface Admin extends User { role: "admin" }
const adminColl = new UserCollection<Admin>();
expectAssignable<Admin | undefined>(adminColl.find(a => a.role === "admin"));
expectNotAssignable<User>(adminColl.find(a => a.role === "admin")); // undefined ham qaytishi mumkin

// Constraint check
interface Product { sku: string }
expectError(new UserCollection<Product>()); // ❌ Product User'ni extend qilmaydi
```

**`package.json` setup:**

```json
{
  "name": "my-lib",
  "version": "1.0.0",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test:types": "tsd",
    "test:dist": "attw --pack ."
  },
  "tsd": {
    "directory": "test-d"
  },
  "devDependencies": {
    "tsd": "^0.30.0",
    "@arethetypeswrong/cli": "^0.13.0"
  }
}
```

**`attw` natijasi misol:**

```
$ npx attw --pack .
my-lib v1.0.0

┌───────────────────┬──────────────────────────┐
│                   │ "my-lib"                 │
├───────────────────┼──────────────────────────┤
│ node10            │ 🟢                       │
│ node16 (from CJS) │ 🟢                       │
│ node16 (from ESM) │ ❌ Resolution failed     │ ← MUAMMO!
│ bundler           │ 🟢                       │
└───────────────────┴──────────────────────────┘
```

### Edge Cases

- **`tsd` slow compile** — Har test fayl alohida TypeScript compile. Katta library'larda sekin.
- **`tsd` skip diagnostics** — `// @ts-expect-error` bilan specific error tekshirish.
- **`attw` false positives** — Library spetsifik konfiguratsiya'da false positive bo'lishi mumkin. Manual verification kerak.
- **Type test coverage** — Hech qanday automatic coverage metric. Manual test yozish.

### Follow-up savollar

1. **"`tsd` `expectType` exact match?"** — Ha. `expectType<string>(x)` `x` aynan `string` bo'lishi kerak. `string | undefined` — fail.
2. **"CI'da `attw` qanday integrate qilinadi?"** — `prepublishOnly` script yoki GitHub Actions step. `--pack` orqali tar.gz yaratiladi va tekshiriladi.

</details>

---

### Savol 11: `declarationMap` nima va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`declarationMap: true` — `.d.ts.map` source map fayllar generate qiladi. IDE "Go to Definition" da `.d.ts` o'rniga original `.ts` source'ga olib boradi. Library development va monorepo'da foydali.

### To'liq tushuntirish

**Default behavior (`declarationMap: false`):**

```
declaration: true → src/utils.ts → dist/utils.js + dist/utils.d.ts

"Go to Definition" → dist/utils.d.ts (faqat signature ko'rinadi)
```

Foydalanuvchi function'ga "Go to Definition" qilsa, faqat declaration ko'radi — body emas.

**`declarationMap: true`:**

```
declaration: true → src/utils.ts → dist/utils.js + dist/utils.d.ts + dist/utils.d.ts.map

"Go to Definition" → src/utils.ts (to'liq source code)
```

`.d.ts.map` fayl IDE'ga `.d.ts`'ning har bir declaration'i `.ts` faylda qayerda joylashganini aytadi. IDE bu mapping'ni o'qib, foydalanuvchini source code'ga olib boradi.

**Qachon kerak:**

- **Library development** — Library yaratuvchilar consumer'larga source navigation imkonini berishi
- **Monorepo** — Package'lar orasida navigate qilish (`packages/ui` dan `packages/utils`'ga)
- **Open source** — Foydalanuvchilar implementation'ni ko'rib o'rganishi
- **Debugging** — Type-level muammo qaerdan kelganini topish

**Qachon kerak emas:**

- **Closed source** — Source'ni hide qilish kerak bo'lsa
- **Production build** — Qo'shimcha fayl size
- **End-user app** — Library consumer emas

### Kod misol

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src"]
}
```

**Generate qilingan fayllar:**

```
src/utils.ts
       ↓
dist/utils.js        ← runtime
dist/utils.js.map    ← source map (runtime → source)
dist/utils.d.ts      ← type
dist/utils.d.ts.map  ← declaration map (type → source)
```

**`.d.ts.map` ichidagi format:**

```json
{
  "version": 3,
  "file": "utils.d.ts",
  "sourceRoot": "",
  "sources": ["../src/utils.ts"],
  "names": [],
  "mappings": "AAAA,wBAAgB,UAAU,..."
}
```

Mapping'lar — declaration'ning har bir element'i source'ning qaysi line/column'da.

**Library `package.json`:**

```json
{
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "files": [
    "dist/**/*.js",
    "dist/**/*.d.ts",
    "dist/**/*.d.ts.map",
    "src/**/*.ts"
  ]
}
```

`src/` ni ham include qilish kerak — IDE source'ga point qila olish uchun.

**IDE behavior:**

```typescript
// Consumer app
import { formatDate } from "my-lib";

formatDate(new Date());
//   ↑
// "Go to Definition"
//   ↓
// declarationMap: false → node_modules/my-lib/dist/index.d.ts
// declarationMap: true  → node_modules/my-lib/src/utils.ts
```

### Edge Cases

- **Publish'da `src/` shart** — `.d.ts.map` source'ni reference qiladi. `src/` package'da yo'q bo'lsa, IDE "file not found" beradi.
- **Monorepo workspace** — Workspace package'lari `src/` ni `dist/` bilan birga publish qilmasligi mumkin. Lokal development'da ishlaydi (relative path), publish qilingandan keyin yo'q.
- **`inlineSourceMap` alternative** — `inlineSources: true` bilan source map fayl ichida embedded. Lekin `.d.ts` uchun ekvivalent yo'q.
- **Privacy** — Closed-source library'da `src/` publish qilmaslik. `declarationMap: false`.

### Follow-up savollar

1. **"`declarationMap` build size'ga ta'sir?"** — Ha. Har `.d.ts` uchun `.d.ts.map`. Source'ni ham bundle'ga qo'shsa, sezilarli. Production-optional.
2. **"`sourceMap` va `declarationMap` farqi?"** — `sourceMap` runtime debug uchun (`.js.map`). `declarationMap` type navigation uchun (`.d.ts.map`). Ikkalasi ham IDE'da ishlaydi.

</details>

---

### Savol 12: `private` member `.d.ts` da qanday ko'rsatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript `private` member `.d.ts` da **nomi bilan**, lekin **type'i ko'rinmaydi**. Sabab — structural compatibility: consumer access qila olmaydi, lekin field mavjudligi shape'ga ta'sir qiladi.

### To'liq tushuntirish

TypeScript `private` keyword **compile-time only** — class consumer kompilatordan tekshiruv natijasi. Runtime'da JavaScript da private yo'q (ECMAScript `#private` bilan farq).

**Structural typing va private:**

TypeScript structural typing — class'lar shape (struktura) bo'yicha taqqoslanadi. Lekin `private` member structural matching uchun "barrier" — ikki sinfda bir xil shape lekin `private` field bo'lsa, ular bir-biriga assignable emas.

```typescript
class A {
  private secret: string = "a";
}

class B {
  private secret: string = "b";
}

const a: A = new A();
const b: A = new B(); // ❌ Error — private 'secret' har xil class'da
```

Compiler `private` field "ownership"'ini tekshiradi — qaysi class'da declare qilingan.

**`.d.ts` generation:**

```typescript
// Source
class User {
  private password: string;
  public name: string;
  
  constructor(name: string, password: string) {
    this.name = name;
    this.password = password;
  }
}

// Generated .d.ts
declare class User {
  private password;      // ← NOM bor, TYPE yo'q
  name: string;          // ← public — to'liq
  constructor(name: string, password: string);
}
```

**Nima uchun type yo'q:**

- Structural matching uchun faqat field mavjudligi va `private` flag muhim
- Type leak'ni oldini olish — internal implementation detail
- API surface area kichraytirish

**Behavior consumer code'da:**

```typescript
import { User } from "my-lib";

const user = new User("Ali", "secret");
user.name = "Vali";       // ✅ — public
user.password = "new";    // ❌ — private access
user.password;            // ❌ — private read
```

Consumer `password`'ga access qila olmaydi, lekin field mavjudligini compiler biladi.

### Kod misol

**Class hierarchy:**

```typescript
// Source
class BaseUser {
  protected id: number;
  private secret: string;
  
  constructor(id: number, secret: string) {
    this.id = id;
    this.secret = secret;
  }
}

class AdminUser extends BaseUser {
  private permissions: string[];
  
  constructor(id: number, secret: string, permissions: string[]) {
    super(id, secret);
    this.permissions = permissions;
  }
  
  hasPermission(perm: string): boolean {
    return this.permissions.includes(perm);
  }
}

// Generated .d.ts
declare class BaseUser {
  protected id: number;
  private secret;             // ← type yo'q
  constructor(id: number, secret: string);
}

declare class AdminUser extends BaseUser {
  private permissions;        // ← type yo'q
  constructor(id: number, secret: string, permissions: string[]);
  hasPermission(perm: string): boolean;
}
```

**ES `#private` farq:**

```typescript
// Source — ECMAScript private (TS 3.8+)
class Account {
  #balance: number = 0;
  
  deposit(amount: number): void {
    this.#balance += amount;
  }
}

// Generated .d.ts
declare class Account {
  #private; // ← faqat brand, member nomi ham yo'q
  deposit(amount: number): void;
}
```

ECMAScript `#private` — runtime'da real private. `.d.ts`'da faqat `#private` brand (structural barrier) ko'rsatiladi, hech qanday member nomi yo'q.

**`protected` member:**

```typescript
class Service {
  protected logger: Logger;
}

// Generated .d.ts
declare class Service {
  protected logger: Logger;   // ← type bor — subclass'larga kerak
}
```

`protected` `private`'dan farqli — subclass'larga access kerak, shuning uchun type ko'rinadi.

### Edge Cases

- **`private` constructor** — `private constructor()` `.d.ts`'da ko'rsatiladi. Class instantiation'ni boshqarish (singleton).
- **`private static` member** — Ham nomi bor, type'i yo'q. Static structural matching.
- **`isolatedDeclarations` bilan private** — Explicit type yozish kerak. `.d.ts`'da type olib tashlanadi.
- **Mixin pattern** — `private` member mixin'larda muammoli bo'lishi mumkin (structural matching).

### Follow-up savollar

1. **"`private` field consumer'da runtime'da ko'rinadimi?"** — Ha, TypeScript `private` faqat compile-time. `(user as any).password` bilan access. `#private` ECMAScript runtime block.
2. **"`tsc --declaration` bilan `private` yo'q `// @ts-ignore` qilish mumkinmi?"** — Yo'q, compiler hech qachon `private` type'ni emit qilmaydi. Bu spec behavior.

</details>

---

## Output savollari

### Savol 13: `.d.ts` output [Middle]

**Savol:** Quyidagi TypeScript kodning `.d.ts` output ini yozing (`declaration: true`):

```typescript
export type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

export interface RequestConfig {
  url: string;
  method: HttpMethod;
  headers?: Record<string, string>;
  timeout?: number;
}

export async function request<T>(config: RequestConfig): Promise<T> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), config.timeout ?? 5000);
  try {
    const response = await fetch(config.url, {
      method: config.method,
      headers: config.headers,
      signal: controller.signal,
    });
    return response.json();
  } finally {
    clearTimeout(timer);
  }
}

export class ApiClient {
  private baseUrl: string;
  protected defaultHeaders: Record<string, string>;
  
  constructor(baseUrl: string, defaultHeaders: Record<string, string> = {}) {
    this.baseUrl = baseUrl;
    this.defaultHeaders = defaultHeaders;
  }
  
  get<T>(path: string): Promise<T> {
    return request({ url: `${this.baseUrl}${path}`, method: "GET", headers: this.defaultHeaders });
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
export type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

export interface RequestConfig {
  url: string;
  method: HttpMethod;
  headers?: Record<string, string>;
  timeout?: number;
}

export declare function request<T>(config: RequestConfig): Promise<T>;

export declare class ApiClient {
  private baseUrl;
  protected defaultHeaders: Record<string, string>;
  constructor(baseUrl: string, defaultHeaders?: Record<string, string>);
  get<T>(path: string): Promise<T>;
}
```

### To'liq tushuntirish

**O'zgarganlar:**

1. **Function/method body** — olib tashlandi, faqat signature
2. **`declare` keyword** — function va class oldida qo'shildi
3. **`async`** — yo'q (return type `Promise<T>` allaqachon mavjud)
4. **`private baseUrl: string`** → `private baseUrl;` (type olib tashlandi — structural barrier)
5. **`protected defaultHeaders`** → type saqlandi (subclass'larga kerak)
6. **Default parameter `= {}`** — `defaultHeaders?: Record<string, string>` (default value qisqartirildi, optional marker qo'shildi)
7. **Local variable'lar** (`controller`, `timer`) — olib tashlandi
8. **Type alias va interface** — o'zgarishsiz

**Default parameter logic:**

`constructor(headers: X = {})` → `.d.ts`'da `constructor(headers?: X)`. Compiler default value'ni qisqartirib, optional marker'ni qo'shadi.

### Edge Cases

- **`stripInternal: true`** — `/** @internal */` JSDoc'li member'lar `.d.ts`'dan olib tashlanadi.
- **`removeComments: true`** — JSDoc komment'lar `.d.ts`'da o'chiriladi. Default false (IntelliSense uchun saqlanadi).
- **Inferred return type** — Function'da explicit return type bo'lmasa, compiler inferred type'ni `.d.ts`'da yozadi.

</details>

---

### Savol 14: `private` member output [Middle]

**Savol:** Quyidagi class'ning `.d.ts` output'ini yozing:

```typescript
export class Database {
  private connection: Connection;
  private retryCount: number = 3;
  protected timeout: number = 5000;
  public readonly version: string = "1.0";
  
  constructor(url: string) {
    this.connection = createConnection(url);
  }
  
  private reconnect(): void { /* ... */ }
  
  public async query<T>(sql: string): Promise<T[]> {
    return this.connection.exec<T>(sql);
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
export declare class Database {
  private connection;
  private retryCount;
  protected timeout: number;
  readonly version: string;
  constructor(url: string);
  private reconnect;
  query<T>(sql: string): Promise<T[]>;
}
```

### To'liq tushuntirish

**Diqqat qiladigan o'zgarishlar:**

1. **`private connection: Connection`** → `private connection;`
   - Type olib tashlandi (private structural barrier)

2. **`private retryCount: number = 3`** → `private retryCount;`
   - Type ham, default value ham olib tashlandi
   - Lekin `retryCount` nomi qoldi (structural matching uchun)

3. **`protected timeout: number = 5000`** → `protected timeout: number;`
   - Type SAQLANDI (subclass'larga kerak)
   - Default value olib tashlandi

4. **`public readonly version: string = "1.0"`** → `readonly version: string;`
   - `public` — default, kompilator olib tashlaydi
   - `readonly` saqlanadi
   - Default value olib tashlanadi
   - Type `string` — explicit annotation'dan olinadi. (Annotation bo'lmaganda `readonly version = "1.0"` literal type `"1.0"` ni saqlardi — `readonly` property literal'ni widen qilmaydi)

5. **`private reconnect(): void`** → `private reconnect;`
   - Return type yo'q (private method nomi faqat structural)

6. **`public async query<T>`** → `query<T>(sql: string): Promise<T[]>;`
   - `public` o'chadi, `async` o'chadi (return type `Promise<T[]>` allaqachon)

**`public` keyword default:**

TypeScript class member'lar default `public`. `.d.ts` generation'da explicit `public` keyword olib tashlanadi.

### Edge Cases

- **`#private` (ECMAScript)** — `#field;` o'rniga `#private;` brand. Member nomi ham ko'rinmaydi.
- **`override` keyword** — Saqlanadi `.d.ts`'da (TS 4.3+).
- **`accessor`** — TS 4.9+ auto-accessor. `.d.ts`'da `accessor` keyword saqlanadi.
- **Decorators** — `.d.ts`'da ko'rinmaydi (decorator runtime construct).

</details>

---

### Savol 15: `exports` tartibi natija [Middle+]

**Savol:** Quyidagi `package.json` da TypeScript compiler `import "my-lib"` qaysi faylni topadi?

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

`moduleResolution: "node16"` ishlatiladi.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript `"import"` condition'iga mos kelib `./dist/index.mjs` ga point qiladi, yonida `index.d.mts` qidiradi — topa olmaydi. Keyin `"types"` condition'iga "fallback" qilib `./dist/index.d.ts` ni oladi (resolution buzilmaydi, lekin bu fragile fallback). `@arethetypeswrong/cli` buni `FallbackCondition` muammosi deb belgilaydi.

### To'liq tushuntirish

**Type resolution mexanizmi:**

Compiler `exports` ichidagi conditional'larni **yuqoridan pastga** tekshiradi. Type uchun har condition'da match qilingan JS fayl yonidan type-equivalent declaration qidiradi (`.mjs` → `.d.mts`, `.cjs` → `.d.cts`, `.js` → `.d.ts`).

**Bu misolda (`moduleResolution: "node16"`, ESM context):**

1. `"import"` → match → `./dist/index.mjs` → yonida `./dist/index.d.mts` qidiradi → **topilmadi**
2. Node algoritmi bo'yicha resolution shu yerda to'xtashi kerak edi, lekin TypeScript keyingi condition'ga o'tadi (bu TS bug — `FallbackCondition`)
3. `"require"` → `./dist/index.cjs` → `./dist/index.d.cts` → topilmadi
4. `"types"` → `./dist/index.d.ts` → **topildi**

Natijada type'lar topiladi, hard `Cannot find module` error chiqmaydi. Lekin bu fallback ishonchsiz:

- `.d.ts` ESM `.mjs` uchun ishlatildi — `.d.ts` format'i nearest `package.json` bo'yicha aniqlanadi (ko'pincha CommonJS), runtime esa ESM (Masquerading).
- Fallback TS bug'iga tayanadi — kelajakda yo'qolishi mumkin.

**`@arethetypeswrong/cli` buni `FallbackCondition` deb belgilaydi:** type faqat keyingi condition'ga fallback orqali topildi, runtime resolution'i bilan mos kelmasligi mumkin.

**`bundler` resolution:**

`moduleResolution: "bundler"` da type-equivalent sibling lookup bir xil ishlaydi (`.mjs` → `.d.mts`). Sibling topilmasa, aynan shu fallback yuzaga keladi. Farq — `bundler` runtime format'ni Node algoritmidagidek qattiq enforce qilmaydi.

**To'g'ri konfiguratsiya:**

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",   // ← BIRINCHI
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

Endi:
1. `"types"` → match → `./dist/index.d.ts` ✅
2. Type'lar topiladi
3. Runtime'da `"import"`/`"require"` Node.js tomonidan ishlatiladi

**Verification:**

```bash
npx @arethetypeswrong/cli --pack .
```

Wrong order: type fallback orqali topiladi, lekin `FallbackCondition` warning chiqadi (`.mjs` uchun `.d.ts` ishlatildi — Masquerading xavfi). Correct order: `🟢 OK`, har bir resolution context'da to'g'ri declaration.

### Edge Cases

- **Bir xil context ichida** — `"node": { "import": "...", "types": "..." }` — nested conditional. Inner'da ham `"types"` birinchi.
- **Wildcard pattern** — `"./schemas/*"` da har bir resolution alohida tekshiriladi.
- **`types` legacy field** — `package.json` top-level `"types"` field `exports` siz ishlaydi. Modern setup'da `exports` afzal.

</details>

---

### Savol 16: `isolatedDeclarations` error [Middle+]

**Savol:** Quyidagi kodlardan qaysilari `isolatedDeclarations: true` da error beradi?

```typescript
// A
export function add(a: number, b: number) { return a + b; }

// B
export function multiply(a: number, b: number): number { return a * b; }

// C
export const config = { host: "localhost", port: 3000 };

// D
export const TAX_RATE: number = 0.12;

// E
const internal = { x: 1 };  // not exported

// F
export class Calc {
  add(a: number, b: number) { return a + b; }
}

// G
export class Service {
  getName(): string { return "service"; }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A, C, F — error (explicit type yo'q). B, D, E, G — OK.

### To'liq tushuntirish

```
A — ❌ Error. Return type yo'q
    Fix: function add(a: number, b: number): number { ... }

B — ✅ OK. Explicit return type ": number"

C — ❌ Error. Const initializer inferred type
    Fix: const config: { host: string; port: number } = { ... }
    Yoki: interface Config qilib explicit

D — ✅ OK. Explicit ": number" annotation

E — ✅ OK. Local (exported emas) — cheklov yo'q

F — ❌ Error. Method "add" return type yo'q
    Fix: add(a: number, b: number): number { ... }

G — ✅ OK. Method "getName" explicit return type
```

**Rule:**

`isolatedDeclarations` da:
- Exported function — explicit return type majburiy
- Exported const/let — explicit annotation majburiy (initializer dan inference yo'q)
- Exported class method — explicit return type majburiy
- Local code — cheklanmaydi

**Fix all:**

```typescript
// A → B style
export function add(a: number, b: number): number {
  return a + b;
}

// C → annotation
interface AppConfig { host: string; port: number }
export const config: AppConfig = { host: "localhost", port: 3000 };

// F → class method
export class Calc {
  add(a: number, b: number): number { return a + b; }
}
```

### Edge Cases

- **Inferred void** — `function f() {}` (no return) — `isolatedDeclarations` `: void` talab qiladi.
- **Generic constraint** — `function f<T>(x: T)` — return type kerak. `: T` yoki specific.
- **Conditional return** — `function f(): string | number` — explicit union.

</details>

---

### Savol 17: `@types` conflict natija [Middle]

**Savol:** Quyidagi konfiguratsiya bilan `describe`, `it`, `expect` qaysi `@types` package'dan keladi?

```bash
npm install --save-dev @types/jest @types/mocha @types/node
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "types": ["jest", "node"]
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`describe`, `it` — `@types/jest`'dan. `expect` — `@types/jest`'dan. `@types/mocha` butunlay ignore qilinadi.

### To'liq tushuntirish

**`types` array behavior:**

`types: ["jest", "node"]` — faqat shu package'lar avtomatik include. Boshqa `@types/*` (jumladan `@types/mocha`) ignore.

**Jest API:**
- `describe(name, fn)` — `@types/jest`
- `it(name, fn)` — `@types/jest`
- `test(name, fn)` — `@types/jest`
- `expect(value)` — `@types/jest`
- `beforeEach(fn)` — `@types/jest`

**Mocha API:**
- `describe(name, fn)` — `@types/mocha`
- `it(name, fn)` — `@types/mocha`
- (Mocha'da `expect` yo'q — `chai` paketi'dan kelar edi)

**`types` array bilan:**

```typescript
describe("User", () => {
  it("creates", () => {
    expect(1).toBe(1);
  });
});
```

`describe`, `it` global function'lar `@types/jest`'dan. `expect` ham `@types/jest`.

**`types: []` (bo'sh) bo'lsa:**

`describe`, `it`, `expect` undefined → compile error "Cannot find name 'describe'".

**Mocha kerak bo'lsa:**

```json
{
  "compilerOptions": {
    "types": ["mocha", "node"]
  }
}
```

Yoki ikkalasini qo'llab-quvvatlash (`expect` ni explicit import):

```json
{
  "compilerOptions": {
    "types": ["node"]
  }
}
```

```typescript
import { describe, it, expect } from "@jest/globals";
// yoki
import { describe, it } from "mocha";
import { expect } from "chai";
```

### Edge Cases

- **`@types/jest` va `@jest/globals`** — Modern Jest (v28+) — `@jest/globals` explicit import. `@types/jest` global types — eskirgan.
- **`vitest/globals`** — Vitest globals: `types: ["vitest/globals"]`.
- **`types: []` strict mode** — Hech qanday automatic `@types/*`. Har bir type explicit import.

</details>

---

## Coding challenges

### Savol 18: Untyped library uchun declaration [Middle]

**Savol:** Quyidagi JavaScript library uchun `.d.ts` yozing:

```javascript
// string-utils.js
function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

function truncate(str, length, suffix = "...") {
  return str.length > length ? str.slice(0, length) + suffix : str;
}

function pad(str, length, char = " ", direction = "right") {
  const padding = char.repeat(Math.max(0, length - str.length));
  return direction === "right" ? str + padding : padding + str;
}

module.exports = { capitalize, truncate, pad };
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`declare module "string-utils"` bilan ambient module declaration.

### Kod misol

```typescript
// types/string-utils.d.ts
declare module "string-utils" {
  export function capitalize(str: string): string;
  
  export function truncate(str: string, length: number, suffix?: string): string;
  
  export function pad(
    str: string,
    length: number,
    char?: string,
    direction?: "left" | "right",
  ): string;
}
```

**Foydalanish:**

```typescript
// app.ts
import { capitalize, truncate, pad } from "string-utils";

const name = capitalize("ali");                  // "Ali"
const short = truncate("Hello World", 5);         // "Hello..."
const padded = pad("42", 5, "0", "left");         // "00042"
```

**Yaxshilangan variant — strict literal types:**

```typescript
declare module "string-utils" {
  export type PadDirection = "left" | "right";
  
  export interface PadOptions {
    char?: string;
    direction?: PadDirection;
  }
  
  export function capitalize(str: string): string;
  export function truncate(str: string, length: number, suffix?: string): string;
  export function pad(str: string, length: number, char?: string, direction?: PadDirection): string;
}
```

### Edge Cases

- **Default value type** — `direction = "right"` JS source'da. `.d.ts`'da optional + literal union ko'rsatish kerak.
- **`module.exports = { ... }` mapping** — Named exports sifatida declare qilish.
- **`export = ...` syntax** — CommonJS pattern uchun: `export = capitalize;` o'rniga `declare module "name" { export default function (...) }` kabi.

</details>

---

### Savol 19: Library package.json [Middle]

**Savol:** TypeScript library uchun `package.json` yozing. Talablar:
- ESM va CJS dual format
- Sub-path export (`my-lib/utils`)
- `types` to'g'ri tartibda
- Modern Node.js (16+) qo'llab-quvvatlash

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`exports` field ichida `"types"` birinchi, subpath uchun alohida conditional.

### Kod misol

```json
{
  "name": "my-formatter",
  "version": "1.0.0",
  "description": "Modern formatter library",
  "type": "module",
  "main": "./dist/cjs/index.cjs",
  "module": "./dist/esm/index.mjs",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/types/index.d.mts",
        "default": "./dist/esm/index.mjs"
      },
      "require": {
        "types": "./dist/types/index.d.cts",
        "default": "./dist/cjs/index.cjs"
      }
    },
    "./utils": {
      "import": {
        "types": "./dist/types/utils.d.mts",
        "default": "./dist/esm/utils.mjs"
      },
      "require": {
        "types": "./dist/types/utils.d.cts",
        "default": "./dist/cjs/utils.cjs"
      }
    },
    "./package.json": "./package.json"
  },
  "files": [
    "dist",
    "src"
  ],
  "scripts": {
    "build": "tsc -p tsconfig.cjs.json && tsc -p tsconfig.esm.json",
    "test:types": "tsd",
    "test:dist": "attw --pack ."
  },
  "sideEffects": false,
  "engines": {
    "node": ">=16"
  },
  "peerDependencies": {},
  "devDependencies": {
    "typescript": "^5.5.0",
    "tsd": "^0.30.0",
    "@arethetypeswrong/cli": "^0.13.0"
  }
}
```

**Folder structure:**

```
dist/
├── cjs/
│   ├── index.cjs
│   └── utils.cjs
├── esm/
│   ├── index.mjs
│   └── utils.mjs
└── types/
    ├── index.d.mts
    ├── index.d.cts
    ├── utils.d.mts
    └── utils.d.cts
```

**`tsconfig.esm.json`:**

```json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "target": "es2022",
    "declaration": true,
    "declarationDir": "./dist/types",
    "outDir": "./dist/esm"
  },
  "include": ["src"]
}
```

**`tsconfig.cjs.json`:**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node",
    "target": "es2020",
    "outDir": "./dist/cjs"
  },
  "include": ["src"]
}
```

**Foydalanish (consumer):**

```typescript
// ESM consumer
import { format } from "my-formatter";
import { truncate } from "my-formatter/utils";

// CJS consumer
const { format } = require("my-formatter");
const { truncate } = require("my-formatter/utils");
```

### Edge Cases

- **`type: "module"` package.json** — Bu library ESM-first. CJS users uchun `"require"` conditional ishlaydi.
- **`./package.json` export** — Tools (npm, yarn) ba'zan `package.json` ni read qiladi. Explicit export tavsiya.
- **`sideEffects: false`** — Tree shaking yaxshilanadi. Side effect bo'lsa, `["./polyfills.js"]` ro'yxat.

### Follow-up savollar

1. **"Bitta format ham yetadi mi?"** — Ha, pure ESM yetadi (Node 16+ ESM support). Lekin CJS users uchun dual format universal.
2. **"`exports` field'siz qanday?"** — Legacy `"main"` + `"types"` ishlaydi. Lekin `exports` modern resolution support beradi (`node16`, `bundler`).

</details>

---

### Savol 20: Type-safe generic library declaration [Senior]

**Savol:** Type-safe event emitter library uchun `.d.ts` yozing. API:

```typescript
const emitter = createEmitter<{
  login: (userId: string) => void;
  message: (text: string, channel: string) => void;
  logout: () => void;
}>();

emitter.on("login", (userId) => {});       // userId: string
emitter.emit("message", "hi", "general");   // type-safe args
emitter.off("logout", handler);              // ✅
emitter.emit("unknown", "x");                // ❌ error
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`EventMap` generic constraint + `keyof Events` + `Parameters<...>` bilan to'liq type safety.

### Kod misol

```typescript
declare module "typed-emitter" {
  type EventMap = Record<string, (...args: any[]) => void>;
  
  export interface TypedEmitter<Events extends EventMap> {
    on<K extends keyof Events>(event: K, listener: Events[K]): this;
    once<K extends keyof Events>(event: K, listener: Events[K]): this;
    emit<K extends keyof Events>(event: K, ...args: Parameters<Events[K]>): boolean;
    off<K extends keyof Events>(event: K, listener: Events[K]): this;
    removeAllListeners<K extends keyof Events>(event?: K): this;
    listenerCount<K extends keyof Events>(event: K): number;
    listeners<K extends keyof Events>(event: K): Array<Events[K]>;
  }
  
  export function createEmitter<Events extends EventMap>(): TypedEmitter<Events>;
  
  export default createEmitter;
}
```

**Test (`tsd`):**

```typescript
// types.test-d.ts
import { expectType, expectError } from "tsd";
import { createEmitter, TypedEmitter } from "typed-emitter";

type AppEvents = {
  login: (userId: string) => void;
  message: (text: string, channel: string) => void;
  logout: () => void;
};

const emitter = createEmitter<AppEvents>();
expectType<TypedEmitter<AppEvents>>(emitter);

// on — listener arg types correct
emitter.on("login", (userId) => {
  expectType<string>(userId);
});

emitter.on("message", (text, channel) => {
  expectType<string>(text);
  expectType<string>(channel);
});

emitter.on("logout", () => {});

// emit — type-safe args
emitter.emit("login", "user-1");
emitter.emit("message", "hi", "general");
emitter.emit("logout");

// Errors
expectError(emitter.emit("login", 123));            // wrong type
expectError(emitter.emit("message", "hi"));         // missing arg
expectError(emitter.emit("unknown", "x"));          // unknown event
expectError(emitter.on("invalid", () => {}));       // unknown event

// listenerCount — type-safe
expectType<number>(emitter.listenerCount("login"));
expectError(emitter.listenerCount("unknown"));
```

**Real-world ishlatish:**

```typescript
// chat.ts
import createEmitter from "typed-emitter";

type ChatEvents = {
  "user:join": (userId: string, channel: string) => void;
  "user:leave": (userId: string, channel: string) => void;
  "message:send": (message: { from: string; text: string; channel: string }) => void;
  "error": (err: Error) => void;
};

const chat = createEmitter<ChatEvents>();

chat.on("user:join", (userId, channel) => {
  console.log(`${userId} joined ${channel}`);
});

chat.on("message:send", ({ from, text, channel }) => {
  console.log(`[${channel}] ${from}: ${text}`);
});

chat.emit("user:join", "user-1", "general");
chat.emit("message:send", { from: "user-1", text: "Hello", channel: "general" });
```

### Edge Cases

- **`Parameters<T>` tuple distribution** — Function overload'larida `Parameters` faqat oxirgi overload'ni oladi.
- **`this` return type** — Method chaining uchun. `this` saqlanadi subclass'larda.
- **Wildcard listener** — `on("*", listener)` pattern. Implementation custom.
- **Async listener** — `(args) => Promise<void>` ham qabul qilinadi (return type ignored).

### Follow-up savollar

1. **"`Events extends EventMap` constraint nima beradi?"** — `keyof Events` aniq event nom'larini cheklaydi. Unknown event'da compile error. `EventMap` qisqartirma — `Record<string, (...args: any[]) => void>`.
2. **"Listener `this` context type-safe qilish?"** — `listener: (this: ThisType, ...args) => void` signature. TS `--strictBindCallApply` bilan tekshiriladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Generic constraint mexanizmi:**

`Events extends EventMap` constraint compiler'ga `Events` har doim `Record<string, Function>` shape'ida ekanligini garantlaydi. Bundan keyin `keyof Events` operator'i — `string` literal union (event nom'lari).

```typescript
type AppEvents = {
  login: (userId: string) => void;
  logout: () => void;
};

type Keys = keyof AppEvents; // "login" | "logout"
type LoginFn = AppEvents["login"]; // (userId: string) => void
type LoginArgs = Parameters<AppEvents["login"]>; // [userId: string]
```

**`Parameters<Events[K]>` distribution:**

`emit` method signature:

```typescript
emit<K extends keyof Events>(event: K, ...args: Parameters<Events[K]>): boolean;
```

Compiler `K` ni narrow qiladi (string literal) → `Events[K]` aniq function type → `Parameters<>` aniq tuple. Variadic rest parameter type'i tuple sifatida spread.

**Compile vs runtime farq:**

Type definitions runtime'ga yetib bormaydi. Bu API faqat **compile-time** type checking. Runtime'da odatdagi event emitter (`Map<string, Set<Function>>`) ishlaydi. Library implementation'i:

```typescript
class TypedEmitterImpl {
  private listeners = new Map<string, Set<Function>>();
  
  on(event: string, listener: Function): this {
    let set = this.listeners.get(event);
    if (!set) {
      set = new Set();
      this.listeners.set(event, set);
    }
    set.add(listener);
    return this;
  }
  
  emit(event: string, ...args: unknown[]): boolean {
    const set = this.listeners.get(event);
    if (!set) return false;
    set.forEach((fn) => fn(...args));
    return true;
  }
}
```

**`EventMap` `(...args: any[])` shubhasi:**

`any[]` bu yerda generic boundary uchun — har xil signature'lar `EventMap` constraint'iga mos kelishi kerak. Real call site'da `Parameters<Events[K]>` aniq tuple bo'ladi, `any` consumer kodga tarqalmaydi. Bu typed event emitter library'larda (`typed-emitter` kabi) keng tarqalgan constraint pattern.

**`unknown[]` o'rniga `any[]`?**

```typescript
// ❌ — unknown[] constraint kompatibel emas
type EventMap = Record<string, (...args: unknown[]) => void>;
type Bad = { login: (userId: string) => void } extends EventMap ? true : false; // false

// ✅ — any[] bivariance bilan kompatibel
type EventMap = Record<string, (...args: any[]) => void>;
type Good = { login: (userId: string) => void } extends EventMap ? true : false; // true
```

`any[]` parameter bivariance (covariant + contravariant) tufayli har xil signature'larni qabul qiladi. `unknown[]` strict contravariance — `string` argument'li signature'ni qabul qilmaydi.

**Spec reference:**

TypeScript Handbook — [Generic Constraints](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-constraints), [`keyof` Operator](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html), [`Parameters<T>` utility](https://www.typescriptlang.org/docs/handbook/utility-types.html#parameterstype).

</details>

</details>

---

### Savol 21: `isolatedDeclarations` migration [Middle+]

**Savol:** Quyidagi kodni `isolatedDeclarations: true` bilan ishlashi uchun tuzating:

```typescript
export const DEFAULT_CONFIG = {
  host: "localhost",
  port: 3000,
  debug: false,
  timeout: 5000,
};

export function createServer(config = DEFAULT_CONFIG) {
  return {
    config,
    start() { console.log("started"); },
    stop() { console.log("stopped"); },
  };
}

export const requestHandler = (req: { url: string; method: string }) => ({
  status: 200,
  body: { url: req.url, method: req.method },
  headers: { "Content-Type": "application/json" },
});

export class Pipeline {
  private steps = [];
  
  add(fn: (data: any) => any) {
    this.steps.push(fn);
    return this;
  }
  
  async run(input: any) {
    let result = input;
    for (const step of this.steps) {
      result = await step(result);
    }
    return result;
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har exported declaration uchun explicit type. Helper interface'lar yarat.

### Kod misol

```typescript
// Explicit types — har qanday exported uchun
export interface ServerConfig {
  host: string;
  port: number;
  debug: boolean;
  timeout: number;
}

export interface Server {
  config: ServerConfig;
  start(): void;
  stop(): void;
}

export interface HttpRequest {
  url: string;
  method: string;
}

export interface HttpResponse {
  status: number;
  body: { url: string; method: string };
  headers: Record<string, string>;
}

export const DEFAULT_CONFIG: ServerConfig = {
  host: "localhost",
  port: 3000,
  debug: false,
  timeout: 5000,
};

export function createServer(config: ServerConfig = DEFAULT_CONFIG): Server {
  return {
    config,
    start(): void { console.log("started"); },
    stop(): void { console.log("stopped"); },
  };
}

export const requestHandler: (req: HttpRequest) => HttpResponse = (req) => ({
  status: 200,
  body: { url: req.url, method: req.method },
  headers: { "Content-Type": "application/json" },
});

export class Pipeline<T = unknown> {
  private steps: Array<(data: T) => T | Promise<T>>;
  
  constructor() {
    this.steps = [];
  }
  
  add(fn: (data: T) => T | Promise<T>): this {
    this.steps.push(fn);
    return this;
  }
  
  async run(input: T): Promise<T> {
    let result = input;
    for (const step of this.steps) {
      result = await step(result);
    }
    return result;
  }
}
```

**Asosiy o'zgarishlar:**

1. **Interface'lar yaratildi** — `ServerConfig`, `Server`, `HttpRequest`, `HttpResponse`
2. **`DEFAULT_CONFIG`** — explicit `: ServerConfig`
3. **`createServer`** — return type `: Server`
4. **`requestHandler`** — function type alias
5. **`Pipeline`** — generic `<T>` (`any` o'rniga), explicit return types

**`isolatedDeclarations` aslida nimani majburlaydi:**

Trigger — **exported** declaration'ning emit qilinadigan type'i inference'siz topilmasligi: `createServer` return type yo'q, `requestHandler` exported arrow const annotation'siz, `Pipeline.add`/`run` method return type yo'q. Bularning hammasiga explicit type qo'shildi. `private steps = []` (private field) `.d.ts`'da type'siz emit qilinadi, shuning uchun `isolatedDeclarations` uni majburlamaydi — lekin `(data: any) => any`'ni `(data: T) => T` ga aylantirish type safety uchun alohida yaxshilanish.

### Edge Cases

- **`unknown` vs `any`** — `unknown` afzal — type-safe. `any` kafe (type checker o'tib ketadi).
- **Generic default** — `Pipeline<T = unknown>` — backward compatible API.
- **Function type alias** — `(req: HttpRequest) => HttpResponse` har joyda yozish o'rniga `type Handler = (req: HttpRequest) => HttpResponse`.

</details>

---

## Bug fix savollari

### Savol 22: `package.json` types tartib bug [Middle+]

**Savol:** Quyidagi `package.json` `@arethetypeswrong/cli` da `FallbackCondition` warning beradi — type'lar faqat fragile fallback orqali topiladi va ESM consumer'larda noto'g'ri format declaration ishlatiladi. Bug'ni toping va tuzating:

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`"types"` conditional **birinchi** bo'lishi kerak — `exports` da tartib muhim.

### To'liq tushuntirish

**Muammo:**

TypeScript type uchun har condition'da match qilingan JS faylning type-equivalent declaration'ini qidiradi. Hozirgi tartib (ESM consumer):

1. `"import"` → `./dist/index.mjs` → yonida `./dist/index.d.mts` qidiradi → topilmadi
2. Node algoritmi shu yerda to'xtashi kerak, lekin TS keyingi condition'ga fallback qiladi (`FallbackCondition` bug)
3. `"types"` → `./dist/index.d.ts` topiladi — lekin `.d.ts` ESM `.mjs` uchun ishlatildi: `.d.ts` format'i nearest `package.json` bo'yicha aniqlanadi (bu yerda CommonJS), runtime esa ESM (Masquerading)

Type'lar topiladi, hard module error chiqmaydi. Lekin fallback ishonchsiz va format mos kelmasligi mumkin — `@arethetypeswrong/cli` `FallbackCondition` deb belgilaydi.

**Fix:**

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

Endi:
1. `"types"` → match (TS context) → `./dist/index.d.ts` ✅
2. Type'lar topiladi
3. Node.js runtime uchun `"import"` yoki `"require"` ishlatiladi

**Dual types (ESM va CJS uchun alohida):**

Modern best practice — har format uchun alohida `.d.ts`:

```json
{
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.mts",
        "default": "./dist/index.mjs"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  }
}
```

Bu yondashuv'da `"types"` nested condition'lar ichida. ESM va CJS context'lar bir-biriga aralashmaydi.

**Tekshirish:**

```bash
npx @arethetypeswrong/cli --pack .
```

Wrong order:
```
node16 (from CJS) → 🟢
node16 (from ESM) → FallbackCondition (type fallback orqali topildi)
```

Correct order:
```
node16 (from CJS) → 🟢
node16 (from ESM) → 🟢
bundler           → 🟢
```

### Edge Cases

- **Subpath exports** — Har subpath uchun ham `"types"` birinchi: `"./utils": { "types": "...", "import": "..." }`.
- **Legacy `"types"` top-level** — `package.json` top-level `"types"` field eski TypeScript versiyalari uchun fallback. `exports` bilan birga ishlaydi.
- **`"node"` conditional** — `"node"` ham ishlatilishi mumkin. Lekin TS perspective'dan `"types"` birinchi qolishi kerak.

</details>

---

### Savol 23: `.d.ts` declaration bug [Middle+]

**Savol:** Quyidagi qo'lda yozilgan `.d.ts` da 3 ta bug bor. Toping va tuzating:

```typescript
// types/api-client.d.ts
declare module "api-client" {
  // Bug A
  function get(url: string, options?: Record<string, unknown>): Promise<unknown>;
  function get(url: string, options: { responseType: "json" }): Promise<object>;
  
  // Bug B
  class Builder {
    setName(name: string): Builder;
    setAge(age: number): Builder;
  }
  
  // Bug C
  function transform<T>(input: T, fn: (x: any) => any): any;
  
  export { get, Builder, transform };
}
```

> `declare module "..."` body allaqachon ambient context — ichidagi declaration'lar oldidan `declare` yozish kerak emas (yozilsa `error TS1038: A 'declare' modifier cannot be used in an already ambient context`).

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Bug A:** Overload tartibi noto'g'ri — umumiy (keng) signature birinchi, aniqroq signature'ni shadow qiladi. **Bug B:** Method chaining `Builder` o'rniga `this` qaytarish kerak. **Bug C:** Generic constraint juda keng — `any` type safety yo'q.

### To'liq tushuntirish

**Bug A — Overload tartibi:**

```typescript
// ❌ — umumiy (keng) birinchi (declare module ichida)
function get(url: string, options?: Record<string, unknown>): Promise<unknown>;
function get(url: string, options: { responseType: "json" }): Promise<object>;
```

TypeScript overload resolution birinchi mos keladigan signature'ni oladi. Birinchi signature `options` ni optional `Record<string, unknown>` qiladi — `{ responseType: "json" }` argument ham, hech argument ham unga mos keladi. Ikkinchi (aniqroq `{ responseType: "json" }`, `Promise<object>` qaytaradi) hech qachon tanlanmaydi — `get(url, { responseType: "json" })` ham birinchi overload'ni oladi, return `Promise<unknown>`.

```typescript
// ✅ — aniqroq birinchi (declare module ichida)
function get(url: string, options: { responseType: "json" }): Promise<object>;
function get(url: string, options?: Record<string, unknown>): Promise<unknown>;
```

Endi `get(url, { responseType: "json" })` aniq overload'ni tanlaydi (`Promise<object>`), boshqa call'lar umumiy overload'ga tushadi.

**Bug B — Method chaining (`this` return type):**

```typescript
// ❌ — `Builder` qaytaradi (declare module ichida)
class Builder {
  setName(name: string): Builder;
  setAge(age: number): Builder;
}
```

Subclass'da method chaining buziladi:

```typescript
class AdvancedBuilder extends Builder {
  setExtra(): AdvancedBuilder { return this; }
}

const ab = new AdvancedBuilder();
ab.setName("Ali").setExtra();  // ❌ — setName Builder qaytaradi, AdvancedBuilder emas
```

**Fix:**

```typescript
// ✅ — `this` qaytaradi (declare module ichida)
class Builder {
  setName(name: string): this;
  setAge(age: number): this;
}
```

Endi subclass'da chaining ishlaydi:

```typescript
ab.setName("Ali").setExtra(); // ✅ — setName `AdvancedBuilder` qaytaradi
```

**Bug C — Generic constraint juda keng:**

```typescript
// ❌ — any type safety yo'q (declare module ichida)
function transform<T>(input: T, fn: (x: any) => any): any;
```

- `fn` parameter `any` qabul qiladi — input type `T` ga bog'lanmagan
- Return type `any` — har qanday natija
- Type safety nol

**Fix:**

```typescript
// ✅ — generic constraint (declare module ichida)
function transform<T, R>(input: T, fn: (x: T) => R): R;
```

Endi:
- `fn` faqat `T` type'idagi input qabul qiladi
- Return type `R` — `fn` ning return type'iga bog'liq
- Compile-time type safety

**Foydalanish:**

```typescript
const len = transform("hello", (s) => s.length);
// len: number, s: string ✅

const obj = transform({ x: 1 }, (o) => ({ ...o, y: 2 }));
// obj: { x: number; y: number } ✅
```

### Edge Cases

- **Overload va generic** — Overload'larda generic ham bo'lishi mumkin: `function f<T>(x: T): T; function f(x: any): any;`. Tartib: specific generic birinchi.
- **`this` type narrowing** — `T extends Builder` constraint bilan `this` type narrow qilish mumkin.
- **Generic default** — `transform<T, R = T>(...)` — `R` default `T`. Ko'p hollarda foydali.

</details>

---

## Xulosa

- **`.d.ts`** — faqat type, implementation yo'q. JS'ga compile qilinmaydi
- **`declare` keyword** — ambient declaration. JS emit qilinmaydi. `const/function/class/module/namespace/global/enum`
- **Avtomatik generation** — `declaration: true`, `declarationMap` (source navigation), `emitDeclarationOnly` (bundler bilan)
- **`isolatedDeclarations` (TS 5.5+)** — per-file emit, explicit type majburiy (exported). Tezkor tool'lar uchun
- **DefinitelyTyped `@types/*`** — community type definitions. Library'da type bo'lsa — `@types` kerak emas
- **`typeRoots`** — qidiriladigan papkalar. **`types`** — automatic include qilinadigan package'lar
- **`exports` da `"types"` birinchi** — TS resolution conditional tartib muhim
- **Ambient vs Augmentation** — Ambient (script) — yangi declaration. Augmentation (module) — mavjud bilan merge
- **Triple-slash directives** — `path`, `types`, `lib`. Modern `import` afzal, `.d.ts` da hali kerak
- **Testing** — `tsd` (type assertions), `@arethetypeswrong/cli` (distribution), `dtslint` (DT)
- **`private` member `.d.ts`** — nomi bor, type yo'q. `#private` faqat brand
- **`declarationMap`** — IDE "Go to Definition" → source code
