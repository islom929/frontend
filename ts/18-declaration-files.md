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
- [Module Augmentation](#module-augmentation)
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
3. **Global type lar** — global object lar: `window`, `document`, `process` (`lib.d.ts`)
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

`.ts` fayllar **runtime kod va type information** ikkalasini ham o'z ichiga oladi — kompilator JS emit qilganda implementation saqlanadi, type'lar olib tashlanadi. `.d.ts` fayllar esa **faqat type-level** konstruksiyalardan iborat: ulardan JS hech qachon generate qilinmaydi, ular faqat type-checker'ga signal beradi. Aslida butun `.d.ts` fayl **ambient kontekst**ga ega — ichidagi har bir declaration implicit ravishda `declare` modifier'ga ega deb hisoblanadi.

| Xususiyat | `.ts` | `.d.ts` |
|-----------|-------|---------|
| Implementation (function body) | ✅ Bor | ❌ Yo'q |
| Type annotations | ✅ | ✅ |
| JS ga compile bo'ladi | ✅ | ❌ (allaqachon type faqat) |
| `declare` keyword kerak | ❌ (ixtiyoriy) | ❌ (implicit ambient) |
| Runtime da mavjud | ✅ (compile natijasi) | ❌ |
| Top-level expression | ✅ | ❌ (faqat declaration) |
| Type inference | ✅ | ❌ (explicit type kerak) |

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
// Interface — type-level konstruksiya, declare kerak emas
// `.d.ts` da `declare` implicit (butun fayl ambient kontekst), lekin
// uslub uchun function/var/class oldida explicit yoziladi.
```

Kompilator `.d.ts` faylni o'qiganda faqat type information oladi va symbol table ga qo'shadi. `.ts` fayldan farqli, `.d.ts` da kompilator **type inference qilmaydi** — barcha type lar explicit yozilgan bo'lishi kerak. JS emit bosqichi butunlay skip bo'ladi.

</details>

---

## `declare` Keyword

### Nazariya

`declare` keyword — kompilatorga "bu narsa **boshqa joyda** mavjud, men faqat type ini aytayapman" degan signal. `declare` bilan yozilgan code JS ga **emit qilinmaydi** — faqat type-checking uchun.

Kompilator `declare` li declaration larni type-check qiladi (parametr type lari, return type lari tekshiriladi), lekin **implementation mavjudligini talab qilmaydi**. Oddiy function da body bo'lmasa error, lekin `declare function` da body talab qilinmaydi.

`declare` ishlatilishi mumkin bo'lgan declaration turlari: `var` / `let` / `const`, `function`, `class`, `enum`, `namespace`, `module`, `global`. Type alias (`type`) va `interface` da `declare` **kerak emas** — ular zaten faqat type-level konstruksiyalar va runtime'da mavjud emas.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator parser bosqichida `declare` modifier'ni ko'rgach, AST node'ga `NodeFlags.Ambient` flag o'rnatadi. Type-checker bu node'ni standart node bilan bir xil tahlil qiladi (signature matching, type compatibility), lekin emit bosqichida ikkala transformer ham — declaration emit (`.d.ts` yaratuvchi) va JS emit (TypeScript syntax'ini olib tashlovchi) — ambient node'larni **butunlay tashlab ketadi**.

Natijada `declare const x: number` source code'dan **na `.js` ga**, **na `.d.ts` ga** alohida `declare` keyword bilan yozilmaydi (`.d.ts` content'i `declare` o'rniga implicit ambient — chunki butun `.d.ts` fayl ambient kontekst). Lekin `.ts` faylda yozilgan `declare const` `.d.ts` emit'da `declare const` saqlanadi (top-level statement bo'lsa).

`declare global { ... }` blok faqat **module** kontekstida (fayl `import` yoki `export` ga ega) ishlaydi. Script kontekstida (no import/export) `declare global` syntactic xato — chunki butun fayl zaten global. Shu sababli ko'p misol'larda `export {};` qo'shiladi — fayl module'ga aylantirish uchun.

Internal'da `declare module "name" { ... }` bilan ambient module yaratilgach, type resolution algoritmi (`tryFindAmbientModule` `checker.ts`'da) bu nomni `node_modules` resolution'dan oldin tekshiradi. Wildcard pattern (`"*.svg"`) maxsus `PatternAmbientModule` strukturasiga saqlanadi — checker string match orqali tanlaydi.

</details>

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

**`declare module`** (ambient module — type'lari yo'q JS kutubxona uchun):

```typescript
// types/legacy-sdk.d.ts — `@types/legacy-sdk` yo'q, qo'lda yozish kerak
declare module "legacy-sdk" {
  export interface ClientConfig { apiKey: string; timeout?: number; }
  export class Client {
    constructor(config: ClientConfig);
    request<T>(endpoint: string): Promise<T>;
  }
}

declare module "*.svg" {
  const content: string;
  export default content;
}
```

**`declare namespace`** (UMD library yoki global namespace API uchun):

```typescript
declare namespace GoogleAnalytics {
  interface TrackingConfig { trackingId: string; debug?: boolean; }
  function init(config: TrackingConfig): void;
  function trackEvent(category: string, action: string): void;
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

`"types"` conditional export — `exports` ichida **birinchi** bo'lishi kerak (kompilator first-match resolution ishlatadi: agar `"import"` yoki `"require"` oldinda bo'lsa, `.d.ts` topilmaydi).

Top-level `"types"` field — `exports` field'ni qo'llab-quvvatlamaydigan eski tool'lar uchun fallback. `moduleResolution: "node16"` yoki `"bundler"` da `exports` ustun keladi.

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

  export interface AnalyticsEvent {
    name: string;
    properties?: Record<string, string | number | boolean>;
    timestamp?: Date;
  }

  export class Analytics {
    constructor(config: AnalyticsConfig);
    track(event: AnalyticsEvent): void;
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
<summary><strong>Under the Hood</strong></summary>

Standart `declaration: true` da declaration emit cross-file type inference'ga tayanadi. Masalan `function getUser() { return fetchUser(); }` declaration'i yaratilganda kompilator `fetchUser` return type'ini izlaydi, kerak bo'lsa boshqa fayldagi import zanjirini kuzatadi. Bu type graph'ning to'liq qurilishini talab qiladi — har bir fayl boshqalardan **bog'liq** bo'ladi va parallel ishlash imkonsiz.

`isolatedDeclarations: true` qoidasi kompilatorga "har bir export'ning type'ini fayldagi ma'lumotdan **yagona o'qish bilan** aniqlash mumkin bo'lsin" deb majburlaydi. Inferred return type, inferred property type, va boshqa cross-file inference talab qiladigan pattern'lar uchun explicit annotation talab qilinadi. Kompilator yetishmagan annotation'ni `error TS9007` va boshqa `9xxx` seriyadagi diagnostics orqali bildiradi.

Bu cheklov tashqi tool'larga (esbuild, SWC, Bun) declaration emit'ni `tsc`'siz, fayl-fayl, parallel oqimda amalga oshirish imkonini beradi. Faqat syntactic information yetarli — semantic resolver kerak emas. Natija: monorepo'da yuzlab paketning declaration'larini `tsc` ishlatishdan tezroq generate qilish mumkin. TypeScript jamoasi bu xususiyatni 5.5 da kiritgan (2024-iyun).

</details>

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

**DefinitelyTyped** — community tomonidan boshqariladigan eng katta TypeScript declaration file lari repository'si. JavaScript kutubxonalar uchun type definition lar. `@types/` scope da npm package sifatida install qilinadi:

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

**`typesVersions` field** (TS 3.1+) — bir paket ichida turli TypeScript versiyalari uchun turli declaration fayllar berishga imkon beradi:

```json
{
  "name": "my-lib",
  "types": "./dist/index.d.ts",
  "typesVersions": {
    ">=4.5": { "*": ["./dist/ts4.5/*"] },
    ">=3.9": { "*": ["./dist/ts3.9/*"] }
  }
}
```

Kompilator semver match qiladi: TS 5.0 ishlatuvchi consumer `>=4.5` matchni topadi va `./dist/ts4.5/*` papkadagi declaration'larni oladi. Bu yangi syntax (`satisfies`, `const` type params) ishlatadigan paket uchun eski TS versiyalariga downgrade-compatible declaration berish uchun foydali.

---

## Ambient Declarations

### Nazariya

Ambient declaration — **implementation yo'q**, faqat type ma'lumot beruvchi declaration. `.d.ts` fayllarning barcha content i ambient. `.ts` faylda esa `declare` keyword bilan ambient declaration yaratiladi.

Ambient declaration lar **declaration merging** orqali ishlaydi — bir xil nomli interface lar avtomatik birlashadi. Bir nechta fayl bir xil module ni declare qilsa, ularning declaration lari **birlashtiriladi**.

<details>
<summary><strong>Kod Misollari</strong></summary>

**UMD Library:**

```typescript
// types/date-lib.d.ts
export as namespace dateLib; // global sifatida `dateLib` nomi bilan mavjud

export interface DateValue {
  format(template: string): string;
  add(amount: number, unit: string): DateValue;
  isValid(): boolean;
}

export function create(input?: string | Date): DateValue;
```

```typescript
// 1) Module sifatida (bundler / Node.js):
import { create } from "date-lib";
const today = create().format("YYYY-MM-DD");

// 2) Script da global sifatida (UMD build <script> tag bilan load qilingan):
//    import statement YO'Q — `dateLib` global scope'da mavjud
const todayGlobal = dateLib.create().format("YYYY-MM-DD");
```

`export as namespace dateLib` — UMD global export declaration. Kompilatorga: "agar consumer module sifatida import qilmasa, `dateLib` global nomi orqali ham bu paketga murojaat qila oladi" degan signal. Faqat `.d.ts` faylda yozilishi mumkin — `.ts` faylda `error TS1315` beradi.

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
// @types/node paket'ini include qiladi

/// <reference types="jest" />
// @types/jest paket'ini include qiladi

/// <reference path="./global-types.d.ts" />
// Bu fayldagi type'larni include qiladi (relative path)

/// <reference lib="es2020" />
// Built-in lib'ni include qiladi (tsconfig "lib" array'ga ekvivalent)

/// <reference no-default-lib="true"/>
// `lib.d.ts` ni butunlay o'chiradi (faqat o'z type lar yozish uchun)

/// <reference types="node" resolution-mode="import" />
// TS 4.7+: paket type'lari ESM resolution rejimida o'qiladi
```

**Qachon kerak:**
- `.d.ts` fayllarida — top-level `import` global scope'ni module'ga aylantirib qo'yadigan hollarda
- Global type dependency — fayl global scope'dagi type'larga bog'liq bo'lganda (`@types/node` `Buffer` global)
- `lib` override — aniq fayl uchun lib'ni o'zgartirish kerak bo'lganda
- `resolution-mode` (TS 4.7+) — paket ESM va CJS uchun turli `.d.ts` chiqarsa, qaysi rejimda o'qish kerakligini aytish

</details>

---

## Module Augmentation

### Nazariya

Module augmentation — **mavjud module'ning** type declaration'larini **kengaytirish** mexanizmi. Bu yondashuv `@types/*` paketni `fork` qilmasdan, kutubxonaning Request, Window yoki boshqa public interface'lariga maydon qo'shish imkonini beradi.

Ikkita asosiy holat: **third-party module augmentation** (`declare module "express"`) va **global augmentation** (`declare global { interface Window { ... } }`).

Augmentation ishlashi uchun ikki shart bajarilishi kerak:

1. **Fayl module bo'lishi** — `import` yoki `export` statement bo'lsa. Aks holda `declare module "..."` mavjud module'ni augment qilmaydi — **yangi ambient module yaratadi** va asl declaration'ni override qiladi.
2. **`interface` orqali merging** — augment qilingan declaration `interface` yoki `namespace` bo'lishi kerak. `type` alias merging'ni qo'llab-quvvatlamaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Declaration merging algoritmi (`mergeSymbol` `checker.ts`'da) bir xil nomli `InterfaceDeclaration` symbol'larni topadi va ularning member'larini **yagona symbol table** ga birlashtiradi. Bir xil property turli type bilan ikki interface'da bo'lsa — `Subsequent property declarations must have the same type` xatosi.

Module augmentation uchun maxsus rule: `declare module "x"` blok'i module faylda (top-level `import`/`export` bor) bo'lganda kompilator uni augmenting declaration sifatida belgilab, asl `"x"` module symbol'iga merge qiladi. Aks holda (fayl script bo'lsa) — yangi mustaqil ambient module declaration sifatida qaraladi, merge bo'lmaydi.

Global augmentation (`declare global { ... }`) faqat module faylda ruxsat etiladi. Kompilator `declare global` blok'ini script context'ga ko'tarib, ichidagi interface/var/function'larni global scope'ga qo'shadi. Bu mexanizm orqali `@types/jest` `describe`, `it`, `expect` global'larni qo'shadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Express `Request` ga `user` field qo'shish:**

```typescript
// types/express.d.ts
import "express"; // ← fayl module'ga aylantirish uchun

declare module "express" {
  interface Request {
    user?: { id: string; role: "admin" | "member" };
  }
}
```

```typescript
// app.ts — augmentation avtomatik ishlaydi
import express, { Request, Response } from "express";

const app = express();
app.get("/profile", (req: Request, res: Response) => {
  const userId = req.user?.id; // ✅ typed!
  res.json({ userId });
});
```

**Global augmentation — `Window` ga custom property:**

```typescript
// types/window.d.ts
export {}; // module qilish uchun

declare global {
  interface Window {
    __APP_VERSION__: string;
    analytics: { track(event: string): void };
  }
}
```

```typescript
window.__APP_VERSION__ = "1.0.0"; // ✅
window.analytics.track("page_view"); // ✅
```

**Generic library augmentation** (`fastify` plugin uchun):

```typescript
import "fastify";

declare module "fastify" {
  interface FastifyRequest {
    requestId: string;
    startTime: number;
  }
  interface FastifyInstance {
    db: { query<T>(sql: string): Promise<T[]> };
  }
}
```

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
// Fayl script kontekstida — bu mustaqil ambient module e'loni
// (mavjud Express modulidagi Request'ga MERGE bo'lmaydi).

// FILE: types.d.ts (module — import yoki export bor)
import "express";
declare module "express" {
  interface Request { userId: string; }
}
// Fayl module kontekstida — bu MODULE AUGMENTATION,
// mavjud Express Request interface'iga qo'shiladi ✅
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
npm install axios         # Axios — o'z type lari bor (bundled .d.ts)
npm install @types/axios  # ❌ Keraksiz — deprecated stub
```

Agar library `package.json` da `"types"` field bor bo'lsa — `@types` **kerak emas**. `@types/axios` deprecated stub paket (bo'sh declaration) — axios o'z type'larini ship qiladi. Resolution tartibida package o'zining `"types"` field'i `@types`'dan ustun keladi, shuning uchun `@types/axios` faqat keraksiz dependency bo'lib qoladi, real conflict emas.

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
npm install @types/axios  # ❌ — kerak emas (deprecated stub)
```

**Nima uchun:** Library `package.json` da `"types"` field bilan o'z declaration'ini ship qilsa, resolution shu field'ni `@types`'dan oldin oladi. `@types/axios` esa deprecated bo'sh stub — keraksiz dependency, lekin agar `@types` paketi real (eskirgan) type'lar olib kelsa, ular library'ning yangi type'laridan farq qilib stale type error berishi mumkin.

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

### ❌ Xato 5: `type` alias bilan module augmentation qilish

```typescript
import "express";

// ❌ — `type` alias merging'ni qo'llab-quvvatlamaydi
declare module "express" {
  type Request = { user: { id: string } }; // ← Error: Duplicate identifier
}

// ✅ — `interface` orqali augmentation
declare module "express" {
  interface Request { user: { id: string }; }
}
```

**Nima uchun:** Declaration merging algoritmi faqat `interface` va `namespace`'ni birlashtiradi. `type` alias bir martagina e'lon qilinadi va qayta e'lon `Duplicate identifier` xatosi keltirib chiqaradi.

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
export const handler = (req: { url: string }) => ({ status: 200, body: req.url });
```

<details>
<summary>Javob</summary>

```typescript
export interface ServerConfig { host: string; port: number; debug: boolean; }
export interface Server { config: ServerConfig; start(): void; }
export interface HandlerRequest { url: string; }
export interface HandlerResult { status: number; body: string; }

export const DEFAULT_CONFIG: ServerConfig = { host: "localhost", port: 3000, debug: false };
export function createServer(config: ServerConfig = DEFAULT_CONFIG): Server {
  return { config, start() {} };
}
export const handler: (req: HandlerRequest) => HandlerResult = (req) => ({ status: 200, body: req.url });
```

</details>

---

### Mashq 5: Global Type Conflict Hal Qilish (Oson)

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
- **`isolatedDeclarations` (TS 5.5+)** — fayl-bo'yicha mustaqil emit, explicit types majburiy. Parallelization (esbuild/SWC/Bun) bilan monorepo build tezligi.
- **Qo'lda** — JS kutubxona, global script, non-standard module lar uchun.

**Ecosystem:**
- **DefinitelyTyped** — `@types/*` package lar
- **`typeRoots`** va **`types`** — qaysi type definition lar include qilinishini boshqarish
- **`exports` da `"types"` birinchi** bo'lishi kerak
- **`typesVersions`** — bir paket ichida turli TS versiyalari uchun turli declaration'lar

**Declaration turlari:**
- **Ambient module** — `declare module "name"`
- **Wildcard module** — `declare module "*.css"`
- **Global** — `declare global { ... }`
- **UMD** — `export as namespace`
- **Module augmentation** — mavjud module/global'ga property qo'shish (interface merging, `import` shart)

**Testing:** `tsd`, `@arethetypeswrong/cli`, `dtslint`

**Bog'liq bo'limlar:**
- [Bo'lim 17: Modules](17-modules.md) — module augmentation, ambient modules, `import type`
- [Bo'lim 4: Objects va Interfaces](04-objects-interfaces.md) — declaration merging
- [Bo'lim 22: tsconfig.json](22-tsconfig.md) — declaration-related options

---

**Keyingi bo'lim:** [19-decorators.md](19-decorators.md) — Decorators: legacy va TC39 Stage 3 standartlari, class/method/property/accessor decorators, decorator factories, composition, execution order, va real-world patterns.
