# Interview: Module System

> `import type`, `verbatimModuleSyntax`, module resolution, `isolatedModules`, barrel exports, module augmentation, `declare global`, dynamic imports, circular dependencies bo'yicha interview savollari.

---

## Nazariy savollar

### 1. `import type` nima? Oddiy `import` dan farqi?

<details>
<summary>Javob</summary>

`import type` — faqat **type-level** import. JS ga compile bo'lganda **to'liq o'chiriladi**:

```typescript
import { UserService } from "./services";  // JS da qoladi
import type { User } from "./types";       // JS da o'chiriladi
```

| | `import` | `import type` |
|---|---------|---------------|
| JS output | Qoladi | O'chiriladi |
| `new Class()` | ✅ | ❌ |
| Type annotation | ✅ | ✅ |
| Bundle size | Ta'sir qiladi | Ta'sir qilmaydi |
| Side effects | Ishlaydi | Ishlamaydi |

Inline variant: `import { type User, UserService } from "./types"` — aralash import.

</details>

### 2. `verbatimModuleSyntax` nima? Nima uchun yaxshiroq?

<details>
<summary>Javob</summary>

`verbatimModuleSyntax` (TS 5.0+) — `import type` ni **majburiy** qiladi. Yoqilganda:

```typescript
import { User } from "./types";      // ❌ Error — User faqat type da ishlatilgan
import type { User } from "./types"; // ✅
import { type User } from "./types"; // ✅
```

**Nima uchun yaxshiroq:**
- **Aniqlik** — har import ning type/value ekanligi kodni o'qiganda ko'rinadi
- **Predictability** — yozilgan import aynan o'sha holatda JS ga chiqadi, heuristic yo'q
- **Bundler mos** — esbuild, SWC faylni alohida transpile qilganda type import larni aniq biladi

`importsNotUsedAsValues` va `preserveValueImports` ni almashtiradi (ular deprecated).

</details>

### 3. Module resolution strategiyalari — qachon qaysi biri?

<details>
<summary>Javob</summary>

| Strategy | Extension kerak? | `exports` support | Muhit |
|----------|:---------------:|:-----------------:|-------|
| `"node"` | Yo'q | ❌ | Legacy CJS |
| `"node16"` / `"nodenext"` | Ha (`.js`) | ✅ | Node.js ESM |
| `"bundler"` | Yo'q | ✅ | Vite/Webpack |

```typescript
// "node16" — extension majburiy
import { User } from "./user.js"; // ✅ (.js yoziladi, TS .ts ga map qiladi)
import { User } from "./user";    // ❌

// "bundler" — extension ixtiyoriy
import { User } from "./user";    // ✅
```

**Tanlash:**

| Loyiha | Strategy |
|--------|----------|
| Vite + React/Vue, Next.js | `"bundler"` |
| Node.js ESM (Express, Fastify) | `"node16"` / `"nodenext"` |
| Library (npm publish) | `"nodenext"` |
| Legacy CJS | `"node"` |

`"node16"` da `.js` sababi: Node.js ESM spec da import path to'liq bo'lishi kerak. TS `.js` ni `.ts` source ga map qiladi.

</details>

### 4. `isolatedModules` nima? Qanday cheklovlar qo'yadi?

<details>
<summary>Javob</summary>

`isolatedModules: true` — har fayl **mustaqil** transpile bo'lishi uchun cheklovlar. esbuild, SWC, Babel — faylni alohida (project kontekstsiz) transpile qiladi.

**Cheklovlar:**

1. **Type-only re-export** — `export type` kerak:
```typescript
export { User } from "./types";      // ❌ transpiler bilmaydi
export type { User } from "./types"; // ✅
```

2. **`const enum`** — taqiqlangan (boshqa fayldagi qiymat kerak)

3. **Har fayl module** bo'lishi kerak (import/export)

**Qachon yoqish:** Deyarli doim. Vite, Next.js, esbuild ishlatayotgan har qanday project da.

</details>

### 5. Barrel export nima? Qanday muammolari bor?

<details>
<summary>Javob</summary>

Barrel — `index.ts` orqali papka export larini birlashtirish:

```typescript
// models/index.ts
export { User } from "./user";
export { Product } from "./product";
export { Order } from "./order";

// Import
import { User, Product } from "./models";
```

**Muammolari:**

1. **Tree shaking buzilishi** — bundler barrel ni to'liq evaluate qilishi mumkin (faqat `User` kerak, lekin `Product`, `Order` ham load bo'ladi)

2. **Circular dependency** — barrel orqali oson paydo bo'ladi

3. **IDE auto-import** — barrel dan import qo'shadi, to'g'ridan-to'g'ri fayl o'rniga

**Yechim — type-only barrel:**
```typescript
export type { User } from "./user";
export type { Product } from "./product";
// JS da hech narsa — tree shaking ta'sirlanmaydi ✅
```

</details>

### 6. Module augmentation nima? Qanday ishlaydi?

<details>
<summary>Javob</summary>

Module augmentation — mavjud module type larini kengaytirish. `declare module` + `import` bilan:

```typescript
// types/express.d.ts
import "express"; // ← MODULE qilish uchun import KERAK

declare module "express" {
  interface Request {
    userId: string;
    role: "admin" | "user";
  }
}
```

```typescript
// middleware.ts
import { Request } from "express";
function auth(req: Request) {
  req.userId = "user-123"; // ✅ augmented property
}
```

**Muhim:** `import "express"` bo'lmasa — fayl **script** bo'ladi va `declare module` mavjud express ni **butunlay almashtiradi** (augment qilmaydi). `import` faylni module qiladi → augmentation bo'ladi.

</details>

### 7. `declare global` nima? Qachon ishlatiladi?

<details>
<summary>Javob</summary>

Global scope ga yangi type/variable qo'shish. Module fayl ichida ishlaydi:

```typescript
// globals.d.ts
export {}; // ← faylni module qiladi

declare global {
  interface Window {
    __APP_VERSION__: string;
    analytics: { track(event: string): void };
  }
  var APP_NAME: string;
}
```

```typescript
console.log(window.__APP_VERSION__); // ✅ string
console.log(APP_NAME);              // ✅
```

**`export {}` nima uchun kerak:** `declare global` faqat module fayl ichida ishlaydi. Bo'sh export faylni module qiladi.

**Qachon:** `window` ga polyfill/third-party, Webpack DefinePlugin inject, `process.env` kengaytirish.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Module augmentation — xatoni toping (Daraja: Middle)

**Savol:** Bu kodda nima xato? Express ning original type lari (body, params) yo'qoladi:

```typescript
// types/custom.d.ts
declare module "express" {
  interface Request {
    userId: string;
  }
}
```

<details>
<summary>Yechim</summary>

**Xato:** `import` yo'q — fayl **script**. Script da `declare module "express"` **ambient module declaration** — mavjud express ni **butunlay almashtiradi**. `body`, `params`, `query` — hammasi yo'qoladi!

```typescript
// ✅ Tuzatish — import qo'shish
import "express"; // ← faylni MODULE qiladi

declare module "express" {
  interface Request {
    userId: string; // ← mavjud Request ga QO'SHILADI
  }
}
```

Endi `Request` ning barcha original property lari saqlanadi + `userId` qo'shiladi. `import` → module → `declare module` = **augmentation** (almashtirish emas).

</details>

### 2. Dynamic import — type safety (Daraja: Middle)

**Savol:** Dynamic import bilan lazy loaded module type-safe ishlaydi mi? Output va type ni ayting:

```typescript
// heavy-module.ts
export class HeavyChart { render(): string { return "chart"; } }
export function createChart(): HeavyChart { return new HeavyChart(); }

// app.ts
async function loadChart() {
  const mod = await import("./heavy-module");
  const chart = new mod.HeavyChart();
  return chart.render();
}
```

<details>
<summary>Yechim</summary>

**Ha, type-safe.** TS dynamic import ning return type ini avtomatik aniqlaydi:

```typescript
const mod = await import("./heavy-module");
// mod type: typeof import("./heavy-module")
// mod.HeavyChart — ✅ class
// mod.createChart — ✅ function

const chart = new mod.HeavyChart();
chart.render(); // ✅ string — type-safe
```

**Type-only dynamic import** (runtime import YO'Q):

```typescript
// Faqat type olish — runtime da import bo'lmaydi
type ChartType = import("./heavy-module").HeavyChart;
```

Bu type-level construct — JS ga tushmaydi.

</details>

### 3. Circular dependency — `import type` bilan hal qilish (Daraja: Middle+)

**Savol:** Bu kodda circular dependency bor. `import type` bilan hal qiling:

```typescript
// user.ts
import { Post } from "./post";
export class User { posts: Post[] = []; }

// post.ts
import { User } from "./user"; // ← CIRCULAR
export class Post { author: User; constructor(a: User) { this.author = a; } }
```

<details>
<summary>Yechim</summary>

```typescript
// ✅ user.ts
import type { Post } from "./post"; // JS da o'chiriladi
export class User { posts: Post[] = []; }

// ✅ post.ts
import type { User } from "./user"; // JS da o'chiriladi
export class Post { author: User; constructor(a: User) { this.author = a; } }
```

**Nima uchun ishlaydi:** `import type` JS da **o'chiriladi**. Runtime da circular yo'q — hech qaysi modul boshqasini import qilmaydi. TS type checker ikkalasining type larini ko'radi (compile-time da circular muammo emas).

**Cheklov:** Agar runtime da ham bir-birini chaqirish kerak — type larni alohida `types.ts` ga ajratish:

```typescript
// types.ts — faqat type lar
export interface IUser { posts: IPost[]; }
export interface IPost { author: IUser; }

// user.ts — type import + implementation
import type { IPost } from "./types";
export class User implements IUser { posts: IPost[] = []; }
```

</details>

### 4. JS output — nima o'chiriladi? (Daraja: Middle+)

**Savol:** `verbatimModuleSyntax: true`, `module: "esnext"` da JS output yozing:

```typescript
import type { User } from "./types";
import { type Product, OrderService } from "./services";
import { TAX_RATE } from "./config";

export type { User };
export interface Config { debug: boolean; }

const service = new OrderService();
const total = service.calculate() * (1 + TAX_RATE);
const user: User = { id: 1, name: "Ali" };
```

<details>
<summary>Yechim</summary>

```javascript
import { OrderService } from "./services";
import { TAX_RATE } from "./config";

const service = new OrderService();
const total = service.calculate() * (1 + TAX_RATE);
const user = { id: 1, name: "Ali" };
```

**O'chirilganlar:**
- `import type { User }` — type-only import
- `import { type Product }` — inline type specifier
- `export type { User }` — type-only export
- `export interface Config` — interface (type)
- `: User` annotation — type annotation

**Qolganlar:** Faqat runtime qiymatlar — `OrderService`, `TAX_RATE`, `const`, function call.

</details>

### 5. `process.env` type-safe qilish (Daraja: Middle+)

**Savol:** `process.env` ni type-safe qiling — faqat ma'lum env variable larga ruxsat, runtime validation ham qo'shing:

```typescript
// process.env.DATABASE_URL → string (undefined emas)
// process.env.NODE_ENV → "development" | "production" | "test"
// process.env.RANDOM_VAR → ❌ compile error
```

<details>
<summary>Yechim</summary>

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
const env = process.env.NODE_ENV;      // "development" | "production" | "test"
const db = process.env.DATABASE_URL;    // string
// process.env.RANDOM_VAR;             // ❌ Property does not exist
```

**Muammo:** Bu compile-time. Runtime da `DATABASE_URL` hali ham `undefined` bo'lishi mumkin. Runtime validation:

```typescript
function getEnv(key: keyof NodeJS.ProcessEnv): string {
  const value = process.env[key];
  if (!value) throw new Error(`Missing env var: ${key}`);
  return value;
}

const dbUrl = getEnv("DATABASE_URL"); // ✅ runtime checked
```

**Tushuntirish:** `declare namespace NodeJS` — global namespace augmentation. `ProcessEnv` interface ga declaration merging orqali property qo'shiladi. Compile-time da `process.env.X` faqat declared key larni qabul qiladi.

</details>

---

## Xulosa

- `import type` — JS da o'chiriladi, faqat type uchun. Bundle size ga ta'sir qilmaydi
- `verbatimModuleSyntax` — type import majburiy, predictable emit, bundler-mos
- Module resolution: `"bundler"` (Vite/Webpack), `"node16"` (Node.js ESM), `"node"` (legacy)
- `isolatedModules` — deyarli doim yoqish, `const enum` taqiqlanadi
- Barrel exports — tree shaking muammo, `export type` bilan hal
- Module augmentation — `import` + `declare module` (import yo'q = butunlay almashtirish!)
- `declare global` — global type qo'shish, `export {}` module qilish uchun
- `moduleDetection: "force"` — barcha fayllar module, global scope ifloslanishini oldini oladi

[Asosiy bo'limga qaytish →](../17-modules.md)
