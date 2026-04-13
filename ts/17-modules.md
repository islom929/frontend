# Bo'lim 17: Module System TypeScript da

> TypeScript ning module system i — JavaScript ES Modules ustiga qurilgan, type-specific import/export mexanizmlari qo'shilgan system. `import type`, `verbatimModuleSyntax`, module resolution strategiyalari, path aliases, ambient modules, module augmentation — barchasi TypeScript ning type system ini module chegaralari orqali boshqarish uchun xizmat qiladi. Bu bo'limda JavaScript ning module asoslarini qaytarmaymiz ([JS: Modules](../js/15-modules.md) da batafsil) — faqat TypeScript qo'shadigan yangi tushunchalarni yoritamiz.

---

## Mundarija

- [ES Modules TypeScript da](#es-modules-typescript-da)
- [`import type` va `export type`](#import-type-va-export-type)
- [Inline Type Imports](#inline-type-imports)
- [`verbatimModuleSyntax`](#verbatimmodulesyntax)
- [Module Resolution Strategies](#module-resolution-strategies)
- [`moduleDetection`](#moduledetection)
- [`isolatedModules`](#isolatedmodules)
- [Path Aliases](#path-aliases)
- [Barrel Exports](#barrel-exports)
- [Ambient Modules](#ambient-modules)
- [Module Augmentation](#module-augmentation)
- [Global Augmentation](#global-augmentation)
- [Dynamic Imports](#dynamic-imports)
- [Circular Dependencies va Types](#circular-dependencies-va-types)
- [`.mts` va `.cts` Fayl Kengaytmalari](#mts-va-cts-fayl-kengaytmalari)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## ES Modules TypeScript da

### Nazariya

TypeScript JavaScript ning ES Module syntax ini to'liq qo'llab-quvvatlaydi — `import`, `export`, `default export`, `re-export`. TypeScript bu syntax ustiga **type-specific** qo'shimchalar kiritadi:

1. **Type annotation lar** — export/import qilinadigan qiymatlar type annotated bo'lishi mumkin
2. **`import type`** — faqat type import (JS ga hech narsa tushmaydi)
3. **`export type`** — faqat type export
4. **Interface va type alias export** — JS da mavjud bo'lmagan construct lar

TypeScript da module — **fayl darajasida** aniqlanadi. Agar faylda kamida bitta `import` yoki `export` statement bo'lsa — bu fayl **module**. Aks holda — **script** (global scope da ishlaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `import` va `export` statement larni `tsconfig.json` dagi `module` option ga qarab transform qiladi:

- `"module": "commonjs"` — `export class X {}` → `exports.X = X`, `import { X }` → `const { X } = require("...")`
- `"module": "esnext"` — import/export syntax o'zgarishsiz qoladi, faqat type annotation lar o'chiriladi

`export interface User {}` va `export type UserRole = ...` **to'liq o'chadi** — ular JS output da hech qanday iz qoldirmaydi chunki interface va type alias faqat compile-time construct.

Kompilator har bir fayl uchun **dependency graph** quradi — `import` statement lar orqali qaysi fayl qaysi faylga bog'liqligini aniqlaydi. Bu `tsc --build` va incremental compilation da ishlatiladi — faqat o'zgargan fayl va uning dependent lari qayta compile qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// user.ts — module
export interface User {
  id: number;
  name: string;
  email: string;
}

export type UserRole = "admin" | "user" | "moderator";

export class UserService {
  getUser(id: number): User {
    return { id, name: "Ali", email: "ali@mail.com" };
  }
}

export const DEFAULT_ROLE: UserRole = "user";

export default class AuthService {
  login(email: string, password: string): User | null {
    return null;
  }
}
```

```typescript
// app.ts — import
import AuthService, { User, UserRole, UserService, DEFAULT_ROLE } from "./user";

const auth = new AuthService();
const service = new UserService();

const user: User = service.getUser(1);
const role: UserRole = DEFAULT_ROLE;
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// === TypeScript (module: "esnext") ===
import type { User } from "./types";
import { UserService } from "./services";

const service = new UserService();
```

```javascript
// === JavaScript ===
import { UserService } from "./services";
// import type — o'chirildi

const service = new UserService();
```

```typescript
// === TypeScript (module: "commonjs") ===
import { UserService } from "./services";
const service = new UserService();
```

```javascript
// === JavaScript (CJS) ===
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
const services_1 = require("./services");

const service = new services_1.UserService();
```

</details>

---

## `import type` va `export type`

### Nazariya

TypeScript 3.8 da qo'shilgan `import type` va `export type` — **faqat type-level** import/export. Bu statement lar JS ga compile qilinganda **to'liq o'chiriladi**. Runtime da hech qanday iz qolmaydi.

Bu nima uchun kerak:

1. **Bundle size** — faqat type uchun import qilsangiz, bundler shu import ni o'chirishi kerak. `import type` bilan bu aniq belgilanadi.
2. **Circular dependencies** — `import type` runtime da mavjud emas → circular dependency "uziladi".
3. **Side effects** — ba'zi module lar import qilinganda side effect ishlaydi. `import type` bilan side effect **ishlamaydi**.
4. **Aniqlik** — kodda type va value import lari ajratilganda, o'quvchi uchun nima runtime da mavjud ekanini darhol ko'rish mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `import type` ni ko'rganda — emit bosqichida bu statement ni **to'liq skip** qiladi. Oddiy `import` da esa kompilator o'zi aniqlashga harakat qiladi — agar import qilingan nom faqat type pozitsiyada ishlatilgan bo'lsa, emit qilmaydi. Lekin bu heuristic ba'zan noto'g'ri ishlaydi — ayniqsa re-export larda va circular dependency larda.

```
TypeScript Compiler Pipeline:

Source (.ts) → Scanner → Parser → AST → Type Checker → Emitter → Output (.js)
                                                         │
                                           ┌─────────────┤
                                           │ import type  │ → O'CHIRILADI
                                           │ export type  │ → O'CHIRILADI
                                           │ interface    │ → O'CHIRILADI
                                           │ type alias   │ → O'CHIRILADI
                                           └─────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// types.ts
export interface User { id: number; name: string; }
export type UserRole = "admin" | "user";
export class UserService {
  getAll(): User[] { return []; }
}
```

```typescript
// app.ts — type-only import
import type { User, UserRole } from "./types";
// ✅ Faqat type uchun — JS da import bo'lmaydi

import { UserService } from "./types";
// ✅ Runtime da kerak — JS da import qoladi

const service = new UserService();
const users: User[] = service.getAll();
```

```typescript
// re-export
export type { User, UserRole } from "./types";
// ✅ Faqat type re-export — JS da hech narsa
```

**`import type` bilan cheklovlar:**

```typescript
import type { UserService } from "./types";

// ❌ Error — import type bilan runtime da ishlatish mumkin emas
const service = new UserService();
// 'UserService' cannot be used as a value because it was imported using 'import type'

// ✅ Faqat type position da ishlatish mumkin
let user: UserService;                        // ✅ — type annotation
function process(service: UserService): void {} // ✅ — parameter type
type UserList = UserService[];                  // ✅ — array type
```

</details>

---

## Inline Type Imports

### Nazariya

TypeScript 4.5 da qo'shilgan **inline type import** — bitta `import` statement ichida ba'zi nomlarni `type` deb belgilash:

```typescript
// Oldin — ikki alohida statement kerak edi
import type { User, UserRole } from "./types";
import { UserService } from "./types";

// Endi — bitta statement
import { type User, type UserRole, UserService } from "./types";
```

Bu ancha qulayroq — bitta module dan bir nechta narsa import qilganda alohida-alohida statement yozish kerak emas.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// models.ts
export interface Product { id: number; name: string; price: number; }
export type Category = "electronics" | "clothing" | "food";
export class ProductService {
  findByCategory(category: Category): Product[] { return []; }
}
export const TAX_RATE = 0.12;
```

```typescript
// shop.ts
import { type Product, type Category, ProductService, TAX_RATE } from "./models";
//        ^^^^ type     ^^^^ type      ^^^^^ value    ^^^^ value

const service = new ProductService();
const items: Product[] = service.findByCategory("electronics");
const total = items.reduce((sum, p) => sum + p.price, 0);
const withTax = total * (1 + TAX_RATE);
```

JS ga compile bo'lganda:

```javascript
import { ProductService, TAX_RATE } from "./models";
// Product va Category — o'chirildi (type edi)

const service = new ProductService();
const items = service.findByCategory("electronics");
const total = items.reduce((sum, p) => sum + p.price, 0);
const withTax = total * (1 + TAX_RATE);
```

</details>

---

## `verbatimModuleSyntax`

### Nazariya

TypeScript 5.0 da qo'shilgan `verbatimModuleSyntax` — module import/export ni boshqarishning **yangi standart** usuli. Bu option yoqilganda:

1. **`import type` majburiy** — agar import faqat type uchun ishlatilsa, `import type` yoki inline `type` yozish **shart**
2. **Emit natijalari predictable** — kompilator hech qanday "guessing" qilmaydi
3. **Eskisi o'rniga** — `importsNotUsedAsValues` va `preserveValueImports` option larini almashtiradi

Bu nima uchun yaxshiroq:
- **Aniqlik** — har bir import ning type yoki value ekanligi aniq ko'rinadi
- **Predictability** — kompilator emit natijasi hech qachon kutilmagan bo'lmaydi
- **Bundler compatibility** — bundler lar (esbuild, SWC) faylni alohida transpile qilganda, type import larni aniq bilishi kerak

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
// tsconfig.json
{
  "compilerOptions": {
    "verbatimModuleSyntax": true
  }
}
```

```typescript
// ❌ Error — verbatimModuleSyntax yoqilganda
import { User, UserService } from "./types";
// Agar User faqat type sifatida ishlatilsa — error:
// 'User' is a type and must be imported using a type-only import

// ✅ To'g'ri
import { type User, UserService } from "./types";
// User — type (o'chiriladi), UserService — value (qoladi)
```

`verbatimModuleSyntax` dan **oldin** kompilator heuristic ishlatardi — faqat type pozitsiyada ishlatilgan import larni avtomatik o'chirar edi. Bu ba'zan noto'g'ri natija berardi. Yangi approach da developer niyati source code da aniq ko'rinadi: `type` keyword bor → o'chiriladi, `type` keyword yo'q → JS ga qoladi.

</details>

---

## Module Resolution Strategies

### Nazariya

Module resolution — kompilator `import` statement dagi module specifier ni **aniq fayl yo'liga** aylantirish jarayoni. `import { User } from "./user"` — `"./user"` qaysi faylga point qiladi?

TypeScript da bir nechta resolution strategiya bor:

| Strategy | tsconfig value | Muhit | Xususiyat |
|----------|---------------|-------|-----------|
| **Classic** | `"classic"` | Legacy TS | Hech kim ishlatmaydi |
| **Node** | `"node"` | Node.js CJS | `node_modules` lookup, `index.js` |
| **Node16** | `"node16"` | Node.js ESM | `exports` field, `.js` extension kerak |
| **NodeNext** | `"nodenext"` | Node.js ESM | `node16` bilan bir xil, keyingi versiyalarga moslashadi |
| **Bundler** | `"bundler"` | Webpack/Vite | ESM + extension-less imports |

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `import { X } from "specifier"` ko'rganda quyidagi jarayonni boshlaydi:

**Specifier turi aniqlash:**
- `"./user"` — relative path (`.` yoki `..` bilan boshlanadi)
- `"lodash"` — bare specifier (package name)
- `"/abs/path"` — absolute path

**Fayl qidirish (strategiyaga qarab):**

```
"node":    .ts → .tsx → .d.ts → /index.ts → .js
"node16":  .ts → .tsx → .d.ts (extension required!)
"bundler": .ts → .tsx → .d.ts → /index.ts → .js
           + package.json "exports" field support
```

**`"node16"` va `"bundler"` ning critical farqi:**
- `"node16"` — relative import uchun **extension majburiy** (`./user.js`)
- `"bundler"` — extension-less (`./user`) ruxsat beriladi
- Kompilator `.js` extension ni `.ts` ga map qiladi — `import "./user.js"` aslida `./user.ts` faylni topadi

**`exports` field support:**
`"node16"` va `"bundler"` da kompilator `package.json` dagi `exports` map ni parse qiladi va conditional export larni resolve qiladi. `"node"` strategiya `exports` ni butunlay skip qiladi.

Resolution natijasi cache qilinadi — bir xil specifier ikkinchi marta resolve qilinmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`"node"` — Classic Node.js Resolution:**

```
import { User } from "./user"

1. ./user.ts          ← TS fayl bormi?
2. ./user.tsx
3. ./user.d.ts
4. ./user/index.ts    ← Directory + index.ts
5. ./user/index.tsx
6. ./user.js
7. ./user/index.js
```

**`"node16"` — Zamonaviy Node.js:**

```typescript
import { User } from "./user.js";
// ⚠️ .js kengaytma MAJBURIY (ESM da extension-less yo'q)
// Kompilator .js → .ts mapping qiladi
```

```json
// winston/package.json — exports field
{
  "exports": {
    ".": "./dist/index.js",
    "./transports": "./dist/transports/index.js"
  }
}
```

**`"bundler"` — Vite/Webpack Muhiti (TS 5.0+):**

```typescript
import { User } from "./user";
// ✅ Extension-less — bundler resolve qiladi
```

**Qachon qaysi strategiya:**

| Loyiha turi | Strategiya |
|-------------|-----------|
| React app (Vite, CRA, Next.js) | `"bundler"` |
| Node.js API (ESM) | `"node16"` yoki `"nodenext"` |
| Node.js API (legacy CJS) | `"node"` |
| Library/Package | `"node16"` (eng keng moslik) |
| Monorepo | `"bundler"` yoki `"node16"` |

</details>

---

## `moduleDetection`

### Nazariya

`moduleDetection` — kompilatorga fayl module mi yoki script mi ekanini qanday aniqlashni aytadi:

| Qiymat | Xatti-harakat |
|--------|--------------|
| `"auto"` (default) | `import`/`export` bor → module. `package.json` da `"type": "module"` → module. Aks holda → script |
| `"force"` | Barcha fayl lar module (hatto `import`/`export` yo'q bo'lsa ham) |
| `"legacy"` | Faqat `import`/`export` bor bo'lsa module |

`"force"` — **tavsiya etiladi**. Bu global scope ifloslanishini oldini oladi va har bir fayl ni izolyatsiya qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// constants.ts — import/export yo'q
const API_URL = "https://api.example.com";
const TIMEOUT = 5000;
```

- `"legacy"` bilan: bu fayl **script** — `API_URL` global scope da
- `"force"` bilan: bu fayl **module** — o'zgaruvchilar izolyatsiyalangan

```json
// tsconfig.json
{
  "compilerOptions": {
    "moduleDetection": "force"
  }
}
```

</details>

---

## `isolatedModules`

### Nazariya

`isolatedModules` — kompilatorga **fayl-bo'yicha transpilation** uchun cheklovlar qo'yadi. Bu option yoqilganda, har bir fayl **mustaqil ravishda** (boshqa fayl larning type ma'lumotisiz) transpile qilina olishi kerak.

Bu nima uchun kerak? Zamonaviy transpiler lar — **esbuild**, **SWC**, **Babel** — tezlik uchun har bir faylni **alohida** transpile qiladi. Ular type information ga ega emas — faqat syntax transformation qiladi.

**`isolatedModules` cheklovlari:**

1. **Type-only re-export** — `export type` ishlatish kerak
2. **`const enum`** — cross-file inline mumkin emas → oddiy enum ishlatish
3. **Namespace declaration merging** — ikki fayl orasida ishlamaydi
4. **Har bir fayl module** — `export {}` kerak

**Qachon yoqish:** Deyarli doim. esbuild, SWC, Vite, Next.js ishlatayotgan bo'lsangiz — yoqing.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ❌ Error — isolatedModules yoqilganda
export { User } from "./types";
// Transpiler bilmaydi User type mi yoki value mi

// ✅ To'g'ri
export type { User } from "./types";
```

```typescript
// ❌ Error — const enum cross-file ishlamaydi
export const enum Direction { Up, Down, Left, Right }

// ✅ To'g'ri — oddiy enum
export enum Direction { Up, Down, Left, Right }

// Yoki union type — yaxshiroq
export type Direction = "up" | "down" | "left" | "right";
```

```typescript
// ❌ Error — import/export yo'q fayl
const x = 42;

// ✅ To'g'ri
export const x = 42;
// yoki
const x = 42;
export {}; // bo'sh export — faylni module qiladi
```

</details>

---

## Path Aliases

### Nazariya

Path aliases — `tsconfig.json` dagi `paths` va `baseUrl` orqali import yo'llarini qisqartirish. `../../../../` o'rniga `@components/Button` kabi alias ishlatish mumkin.

**Muhim:** TypeScript `paths` ni **faqat type-checking** uchun ishlatadi — compile qilganda **almashtirilmaydi**. Runtime da resolve qilish uchun **bundler** (Webpack, Vite) yoki **runtime tool** (tsconfig-paths) kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

```typescript
// Oldin — relative path
import { Button } from "../../../../components/Button";
import { formatDate } from "../../../utils/date";

// Keyin — alias bilan
import { Button } from "@components/Button";
import { formatDate } from "@utils/date";
```

**Vite konfiguratsiyasi** (alohida resolve ham kerak):

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import path from "path";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
      "@components": path.resolve(__dirname, "./src/components"),
    },
  },
});
```

```
TS source:  import { Button } from "@components/Button"
                                      │
TS compiler: ✅ Type check — paths bo'yicha topildi
                                      │
JS output:  import { Button } from "@components/Button" ← O'ZGARMAGAN!
                                      │
Bundler:    → src/components/Button.js ← Bundler resolve qiladi
```

</details>

---

## Barrel Exports

### Nazariya

Barrel export — bitta `index.ts` fayl orqali papka ichidagi barcha export larni bir joyga to'plash:

```typescript
// Oldin — har bir fayl dan alohida import
import { User } from "./models/user";
import { Product } from "./models/product";

// Barrel bilan — bitta import
import { User, Product } from "./models";
```

**Afzalliklari:** Toza import lar, refactoring oson.

**Muammolari:** Tree shaking buzilishi (bundler barrel ni to'liq load qilishi mumkin), circular dependency xavfi.

**Yechim:** Type-only barrel lar muammosiz — `export type` JS ga hech narsa emit qilmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// models/user.ts
export interface User { id: number; name: string; }

// models/product.ts
export interface Product { id: number; title: string; price: number; }

// models/index.ts — BARREL
export type { User } from "./user";
export type { Product } from "./product";
// JS da hech narsa — tree shaking ta'sirlanmaydi

// Value barrel (ehtiyot bo'ling):
export { UserService } from "./user-service";
export { ProductService } from "./product-service";
```

**Tree shaking muammosi:**

```typescript
// ❌ Muammo — to'liq barrel import
import { User } from "./models";
// Bundler ./models/index.ts ni to'liq evaluate qilishi mumkin
// product.ts ham load bo'ladi — kerak emas!

// ✅ Type-only barrel — muammosiz
export type { User } from "./user";
// JS da hech narsa bo'lmaydi
```

**Circular dependency xavfi:**

```typescript
// user.ts
import { Post } from "./index"; // ← barrel orqali
// post.ts
import { User } from "./index"; // ← barrel orqali
// Circular! user → index → post → index → user
```

</details>

---

## Ambient Modules

### Nazariya

Ambient module — kompilatorga **type qo'shimchasiz module** haqida xabar berish. CSS, JSON, image fayllarni import qilganda kompilator ularning type ini bilmaydi. `declare module` bilan type berish mumkin.

`declare module` **hech qanday JS emit qilmaydi** — u faqat kompilatorga "bu module mavjud va type i shunday" deydi. Runtime da CSS/image import larni bundler handle qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// global.d.ts yoki env.d.ts
declare module "*.css" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.scss" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.png" {
  const src: string;
  export default src;
}

declare module "*.svg" {
  import type { FC, SVGProps } from "react";
  const SVGComponent: FC<SVGProps<SVGSVGElement>>;
  export default SVGComponent;
}
```

```typescript
// app.ts — endi import qilish mumkin
import styles from "./app.css";   // type: { [key: string]: string }
import logo from "./logo.png";     // type: string
import Icon from "./icon.svg";     // type: FC<SVGProps<SVGSVGElement>>
```

**Aniq module declaration:**

```typescript
declare module "some-untyped-library" {
  export function doSomething(input: string): number;
  export interface Config { verbose: boolean; timeout: number; }
  export default function init(config: Config): void;
}
```

</details>

---

## Module Augmentation

### Nazariya

Module augmentation — mavjud module ning type larini **kengaytirish** (yangi property, method qo'shish). Bu `declare module` bilan module ni qayta ochish orqali amalga oshadi. Original module kodi o'zgarmaydi — faqat type qo'shiladi.

Bu [declaration merging](04-objects-interfaces.md) mexanizmiga asoslangan — bir xil nomli `interface` lar avtomatik birlashadi.

**Muhim:** Module augmentation faqat **module** fayl ichida ishlaydi. Fayl module bo'lishi uchun `import` yoki `export` bo'lishi kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

**Express — Request ga property qo'shish:**

```typescript
// types/express.d.ts
import "express"; // ← faylni module qiladi

declare module "express-serve-static-core" {
  interface Request {
    userId: string;
    role: "admin" | "user";
    startTime: number;
  }
}
```

```typescript
// middleware.ts
import { Request, Response, NextFunction } from "express";

function authMiddleware(req: Request, res: Response, next: NextFunction) {
  req.userId = "user-123";    // ✅ — augmented property
  req.role = "admin";         // ✅
  req.startTime = Date.now(); // ✅
  next();
}
```

**`process.env` kengaytirish:**

```typescript
// env.d.ts
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: "development" | "production" | "test";
    DATABASE_URL: string;
    API_KEY: string;
    PORT: string;
  }
}
```

```typescript
// app.ts
const dbUrl = process.env.DATABASE_URL; // ✅ type: string
const port = process.env.PORT;          // ✅ type: string
```

</details>

---

## Global Augmentation

### Nazariya

Global augmentation — global scope ga yangi type yoki variable qo'shish. Module fayl ichida `declare global { ... }` bloki orqali amalga oshadi. Window object ga property qo'shish, global function declare qilish kabi holatlar uchun ishlatiladi.

**Muhim:** `declare global` faqat **module** fayl ichida ishlaydi. `export {}` — bo'sh export — faylni module qiladi.

`declare global` faqat **type-level** declaration. JS da hech narsa qolmaydi — developer o'zi runtime da qiymat assign qilishi kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// globals.d.ts
export {}; // ← faylni module qiladi

declare global {
  interface Window {
    __APP_VERSION__: string;
    __DEBUG__: boolean;
    analytics: {
      track(event: string, data: Record<string, unknown>): void;
    };
  }

  function sleep(ms: number): Promise<void>;
  var APP_NAME: string;
}
```

```typescript
// app.ts
console.log(window.__APP_VERSION__); // ✅ type: string
window.analytics.track("page_view", { path: "/" }); // ✅
await sleep(1000); // ✅ — global function
console.log(APP_NAME); // ✅ — global variable
```

</details>

---

## Dynamic Imports

### Nazariya

Dynamic import — `import()` expression bilan runtime da module ni **lazy load** qilish. TypeScript bu expression ning return type ini avtomatik aniqlaydi — `Promise<typeof Module>`.

Runtime da `import()` JS engine ning native dynamic import mexanizmi. Kompilator `module` target ga qarab emit qiladi — `"esnext"` da `import()` qoladi, `"commonjs"` da `require()` ga aylanishi mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Statik import — module darhol load bo'ladi
import { HeavyChart } from "./heavy-chart";

// Dynamic import — kerak bo'lganda load bo'ladi
async function showChart() {
  const { HeavyChart } = await import("./heavy-chart");
  // type: typeof import("./heavy-chart") — to'liq type safe
  const chart = new HeavyChart();
  chart.render();
}
```

**Conditional import:**

```typescript
async function getFormatter(locale: string) {
  if (locale === "uz") {
    const { UzFormatter } = await import("./formatters/uz");
    return new UzFormatter();
  } else {
    const { EnFormatter } = await import("./formatters/en");
    return new EnFormatter();
  }
}
```

**Type-only dynamic import — faqat type olish uchun:**

```typescript
// Runtime da import bo'lmaydi
type ChartModule = typeof import("./heavy-chart");
type ChartType = import("./heavy-chart").HeavyChart;
```

</details>

---

## Circular Dependencies va Types

### Nazariya

Circular dependency — A moduli B ni import qiladi, B esa A ni import qiladi. Bu runtime da muammo yaratishi mumkin (undefined exports). **Lekin type-only import lar bilan circular dependency muammo bo'lmaydi** — chunki ular runtime da mavjud emas.

`import type` to'liq erase qilinadi — JS output da `require(...)` yoki `import ...` paydo bo'lmaydi. Runtime da module lar bir-birini import qilmaydi → circular dependency yo'qoladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ❌ Runtime circular dependency — muammoli
// user.ts
import { Post } from "./post";
export class User { posts: Post[] = []; }

// post.ts
import { User } from "./user"; // ← CIRCULAR!
export class Post {
  author: User;
  constructor(author: User) { this.author = author; }
}
```

```typescript
// ✅ Type-only import bilan — muammosiz
// user.ts
import type { Post } from "./post"; // ← runtime da yo'q
export class User { posts: Post[] = []; }

// post.ts
import type { User } from "./user"; // ← circular uzildi
export class Post {
  author: User;
  constructor(author: User) { this.author = author; }
}
```

**Qoida:** Agar ikki module bir-biridan faqat **type** uchun import qilsa — `import type` circular dependency ni hal qiladi. Agar **value** ham kerak bo'lsa — type larni alohida fayl ga ajratish kerak.

</details>

---

## `.mts` va `.cts` Fayl Kengaytmalari

### Nazariya

TypeScript 4.7 da qo'shilgan `.mts` va `.cts` kengaytmalari — fayl ning module formati ni **aniq** belgilash:

| Kengaytma | Module formati | JS output | import syntax |
|-----------|---------------|-----------|---------------|
| `.ts` | `tsconfig.json` yoki `package.json` ga qarab | `.js` | `import` yoki `require` |
| `.mts` | **Doim ESM** | `.mjs` | `import` |
| `.cts` | **Doim CJS** | `.cjs` | `require()` |

Bu kengaytmalar `module: "node16"` yoki `module: "nodenext"` bilan ishlaydi.

Bu qachon kerak:
- **Aralash project** — CJS va ESM fayllar bir project da
- **Library yaratish** — CJS va ESM uchun dual package
- **Aniqlik** — fayl ning module formati fayl nomidan ko'rinadi

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// utils.mts — DOIM ESM
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
// → Compile: utils.mjs (ESM)

// legacy.cts — DOIM CJS
export function oldGreet(name: string): string {
  return `Hello, ${name}!`;
}
// → Compile: legacy.cjs (CJS)
```

Kompilator `.mts` faylda CJS syntax (`require()`, `module.exports`) ko'rsa — error. `.cts` faylda ESM syntax (`import ... from`) ko'rsa — error. Bu fayl kengaytmasi va actual syntax mosligini enforced qiladi.

Import resolution da ham ta'sir qiladi: `import "./utils.mjs"` ko'rganda kompilator `./utils.mts` faylni qidiradi.

</details>

---

## Edge Cases va Gotchas

### 1. `import type` bilan `typeof` — value vs type kontekst

```typescript
import type { UserService } from "./services";

// ❌ — typeof VALUE position da — runtime da UserService ga reference kerak
// type ServiceType = InstanceType<typeof UserService>;

// ✅ — import type VALUE position da ishlatib bo'lmaydi
// Yechim: alohida import
import { UserService } from "./services"; // value import
type ServiceType = InstanceType<typeof UserService>; // ✅
```

### 2. `verbatimModuleSyntax` va side-effect import lar

```typescript
// verbatimModuleSyntax: true

import "./polyfill"; // ✅ — side-effect import, qoladi
import "reflect-metadata"; // ✅ — side-effect import, qoladi

// Bu import lar hech narsa import qilmaydi (specifier yo'q),
// lekin module ni EXECUTE qiladi. verbatimModuleSyntax bu import larni
// o'chirmaydi — ular value import (type emas).
```

### 3. `module: "node16"` da `.js` extension — `.ts` fayl uchun

```typescript
// Source fayl: ./user.ts
// Import: import { User } from "./user.js"
//                                     ^^^^
// Bu confusing ko'rinadi — TS faylga .js kengaytma yozish!
// Lekin mantiq: compile natijasi ./user.js bo'ladi,
// import specifier compile natijasiga point qiladi.
// Kompilator .js → .ts mapping qiladi.
```

### 4. `declare module` — script vs module kontekst farqi

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

### 5. `const enum` cross-file inline — `isolatedModules` da ishlamaydi

```typescript
// constants.ts
export const enum Color { Red = 0, Green = 1, Blue = 2 }

// app.ts (isolatedModules: true)
import { Color } from "./constants";
const c = Color.Red;
// ❌ — transpiler constants.ts ni ko'rmaydi, inline qila olmaydi

// tsc (isolatedModules: false) bilan:
// const c = 0; ← inline replace ✅
// esbuild/SWC bilan: ❌ — ular const enum ni tushunmaydi
```

---

## Common Mistakes

### ❌ Xato 1: `import type` bilan runtime qiymat ishlatish

```typescript
import type { UserService } from "./services";
const service = new UserService();
// ❌ Error: 'UserService' cannot be used as a value
```

**Nima uchun:** `import type` — faqat type position da. `new`, variable assignment, `typeof` (value position) — runtime operatsiyalar.

### ❌ Xato 2: Path aliases bundler da resolve qilmaslik

```json
{ "compilerOptions": { "paths": { "@/*": ["src/*"] } } }
```

```typescript
import { Button } from "@/components/Button";
// TS — ✅ type check o'tadi
// Runtime — ❌ Module not found
```

**Nima uchun:** `paths` faqat type-checking uchun. Bundler (Vite, Webpack) yoki runtime tool alohida konfiguratsiya kerak.

### ❌ Xato 3: `node16`/`nodenext` da extension-less import

```typescript
// moduleResolution: "node16"
import { User } from "./user";
// ❌ — extension kerak!

import { User } from "./user.js"; // ✅
```

**Nima uchun:** Node.js ESM da import specifier to'liq fayl yo'li bo'lishi kerak. `.js` yoziladi chunki compile natijasi `.js`.

### ❌ Xato 4: `const enum` ni `isolatedModules` bilan ishlatish

```typescript
export const enum Color { Red, Green, Blue }
// ❌ — isolatedModules da cross-file inline mumkin emas

export enum Color { Red, Green, Blue } // ✅ oddiy enum
export type Color = "red" | "green" | "blue"; // ✅ union type
```

**Nima uchun:** `const enum` inline replace uchun boshqa fayl ma'lumoti kerak — `isolatedModules` da bu mavjud emas.

### ❌ Xato 5: Module augmentation da `import` ni unutish

```typescript
// ❌ import yo'q — ambient declaration (yangi module yaratadi!)
declare module "express" {
  interface Request { userId: string; }
}

// ✅ import bor — augmentation (mavjudni kengaytiradi)
import "express";
declare module "express" {
  interface Request { userId: string; }
}
```

**Nima uchun:** `declare module` script faylda — ambient declaration. Module faylda — augmentation. `import` faylni module qiladi.

---

## Amaliy Mashqlar

### Mashq 1: Type-only Barrel (Oson)

**Savol:** Quyidagi type larni barrel export qiling. `verbatimModuleSyntax: true` da ishlashi kerak.

```typescript
// models/user.ts
export interface User { id: number; name: string; }
// models/product.ts
export interface Product { id: number; price: number; }
// models/order.ts
export interface Order { id: number; items: Product[]; }
```

<details>
<summary>Javob</summary>

```typescript
// models/index.ts
export type { User } from "./user";
export type { Product } from "./product";
export type { Order } from "./order";
```

**Tushuntirish:** `export type` — type-only re-export. `verbatimModuleSyntax` da barcha type re-export lari `export type` bo'lishi shart. JS output da hech narsa bo'lmaydi.

</details>

---

### Mashq 2: Express Augmentation (O'rta)

**Savol:** Express `Request` ga `user`, `requestId`, `startTime` property larini qo'shing.

<details>
<summary>Javob</summary>

```typescript
// types/express.d.ts
import "express";

declare module "express-serve-static-core" {
  interface Request {
    user: { id: string; role: "admin" | "user" } | null;
    requestId: string;
    startTime: number;
  }
}
```

**Tushuntirish:** `import "express"` faylni module qiladi → `declare module` augmentation bo'ladi. Interface declaration merging tufayli yangi property lar mavjud `Request` ga qo'shiladi.

</details>

---

### Mashq 3: Circular Dependency Fix (O'rta)

**Savol:** Quyidagi circular dependency ni hal qiling:

```typescript
// user.ts
import { Post } from "./post";
export class User { posts: Post[] = []; }

// post.ts
import { User } from "./user";
export class Post { author: User; constructor(a: User) { this.author = a; } }
```

<details>
<summary>Javob</summary>

```typescript
// user.ts
import type { Post } from "./post"; // type-only
export class User { posts: Post[] = []; }

// post.ts
import type { User } from "./user"; // type-only
export class Post {
  author: User;
  constructor(author: User) { this.author = author; }
}
```

**Tushuntirish:** `import type` runtime da mavjud emas — circular dependency "uziladi". Agar runtime da ham kerak bo'lsa — type larni alohida `types.ts` faylga ajratish kerak.

</details>

---

### Mashq 4: Ambient Module Declaration (Qiyin)

**Savol:** Typed olmagan `image-optimizer` kutubxonasi uchun declaration yozing:
- Default export: `optimize(path: string, options?: Options): Promise<Result>`
- Named export: `supportedFormats: string[]`
- `Options`: `{ quality?: number; width?: number; height?: number; format?: "webp" | "avif" | "jpeg" }`
- `Result`: `{ path: string; size: number; originalSize: number }`

<details>
<summary>Javob</summary>

```typescript
// types/image-optimizer.d.ts
declare module "image-optimizer" {
  interface Options {
    quality?: number;
    width?: number;
    height?: number;
    format?: "webp" | "avif" | "jpeg";
  }

  interface Result {
    path: string;
    size: number;
    originalSize: number;
  }

  export const supportedFormats: string[];
  export default function optimize(path: string, options?: Options): Promise<Result>;
}
```

**Tushuntirish:** `declare module` — ambient module declaration. Ichidagi barcha construct lar faqat type-level — runtime da mavjud emas.

</details>

---

### Mashq 5: `tsconfig.json` Konfiguratsiyasi (Qiyin)

**Savol:** Vite + React loyihasi uchun module-related `tsconfig.json` konfiguratsiyasini yozing. Talablar: `verbatimModuleSyntax`, `isolatedModules`, bundler resolution, `@/` alias, `force` module detection.

<details>
<summary>Javob</summary>

```json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "target": "es2020",
    "jsx": "react-jsx",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

**Tushuntirish:**
- `module: "esnext"` — ESM output (Vite ESM bilan ishlaydi)
- `moduleResolution: "bundler"` — extension-less import, `exports` support
- `moduleDetection: "force"` — barcha fayllar module
- `verbatimModuleSyntax: true` — `import type` majburiy, predictable emit
- `isolatedModules: true` — esbuild/SWC compatible
- `paths` — alias (Vite config da ham `resolve.alias` kerak)

</details>

---

## Xulosa

Bu bo'limda TypeScript ning module system iga xos tushunchalar o'rganildi:

**Type-only imports:**
- **`import type`** (TS 3.8+) — faqat type uchun, JS da o'chiriladi
- **Inline `type`** (TS 4.5+) — `import { type User, Service }` — bitta statement da aralash
- **`verbatimModuleSyntax`** (TS 5.0+) — yangi standart, predictable emit

**Module resolution:**
- **`"node"`** — legacy CJS, `exports` field qo'llab-quvvatlamaydi
- **`"node16"`/`"nodenext"`** — zamonaviy Node.js ESM, `.js` extension kerak
- **`"bundler"`** (TS 5.0+) — Vite/Webpack muhiti, extension-less

**Konfiguratsiya:**
- **`moduleDetection: "force"`** — barcha fayllar module (tavsiya)
- **`isolatedModules: true`** — fayl-bo'yicha transpilation (esbuild, SWC, Vite uchun)
- **Path aliases** — `paths` + bundler konfiguratsiyasi

**Module boshqaruv:**
- **Barrel exports** — `index.ts` orqali export to'plash. Type-only barrel muammosiz
- **Ambient modules** — `declare module "*.css"` — type qo'shimchasiz fayl lar uchun
- **Module augmentation** — mavjud module type larini kengaytirish (`import` kerak!)
- **Global augmentation** — `declare global` — global scope ga type qo'shish
- **Dynamic imports** — `await import("./module")` — lazy loading, type-safe
- **Circular deps** — `import type` bilan hal qilish
- **`.mts`/`.cts`** (TS 4.7+) — ESM/CJS ni fayl darajasida aniq belgilash

**Bog'liq bo'limlar:**
- [JS: Modules](../js/15-modules.md) — JavaScript module asoslari
- [Bo'lim 4: Objects va Interfaces](04-objects-interfaces.md) — declaration merging
- [Bo'lim 18: Declaration Files](18-declaration-files.md) — `.d.ts` fayllar chuqur
- [Bo'lim 22: tsconfig.json](22-tsconfig.md) — module-related options batafsil

---

**Keyingi bo'lim:** [18-declaration-files.md](18-declaration-files.md) — Declaration fayllar (.d.ts) chuqur: ambient declarations, `@types/*` packages, DefinitelyTyped ecosystem, module augmentation patterns, triple-slash directives, va declaration merging.
