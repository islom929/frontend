# Interview: Module System

> `import type`, `verbatimModuleSyntax`, module resolution (`node`, `node16`, `nodenext`, `bundler`), `isolatedModules`, `moduleDetection`, path aliases, barrel exports, ambient modules, module augmentation, `declare global`, dynamic imports, circular dependencies, `.mts`/`.cts` bo'yicha interview savollari.

---

## Mundarija

### Nazariy savollar
1. [`import type` va oddiy `import` farqi?](#savol-1-import-type-va-oddiy-import-farqi-junior) [Junior+]
2. [`verbatimModuleSyntax` (TS 5.0+) nima va nima uchun yaxshiroq?](#savol-2-verbatimmodulesyntax-ts-50-nima-va-nima-uchun-yaxshiroq-middle) [Middle]
3. [Module resolution strategiyalari — qachon qaysi biri?](#savol-3-module-resolution-strategiyalari--qachon-qaysi-biri-middle) [Middle]
4. [`isolatedModules` nima va qanday cheklovlar qo'yadi?](#savol-4-isolatedmodules-nima-va-qanday-cheklovlar-qoyadi-middle) [Middle]
5. [Module augmentation — qanday ishlaydi va `import` nima uchun kerak?](#savol-5-module-augmentation--qanday-ishlaydi-va-import-nima-uchun-kerak-middle) [Middle+]
6. [`declare global` nima va qachon ishlatiladi?](#savol-6-declare-global-nima-va-qachon-ishlatiladi-middle) [Middle]
7. [Inline type imports — `import { type X }` farqi?](#savol-7-inline-type-imports--import--type-x--farqi-junior) [Junior+]
8. [Path aliases (`paths`/`baseUrl`) qanday ishlaydi va nima uchun bundler kerak?](#savol-8-path-aliases-pathsbaseurl-qanday-ishlaydi-va-nima-uchun-bundler-kerak-middle) [Middle]
9. [Barrel exports — afzalliklari va muammolari?](#savol-9-barrel-exports--afzalliklari-va-muammolari-middle) [Middle]
10. [Dynamic imports — type safety qanday saqlanadi?](#savol-10-dynamic-imports--type-safety-qanday-saqlanadi-middle) [Middle]
11. [Circular dependencies — `import type` bilan qanday hal qilinadi?](#savol-11-circular-dependencies--import-type-bilan-qanday-hal-qilinadi-middle) [Middle+]
12. [`.mts` va `.cts` fayl kengaytmalari — qachon kerak?](#savol-12-mts-va-cts-fayl-kengaytmalari--qachon-kerak-middle) [Middle+]
13. [ESM va CJS interop — Node.js'da ichki mexanizm](#savol-13-esm-va-cjs-interop--nodejsda-ichki-mexanizm-senior) [Senior]

### Output savollari
14. [`verbatimModuleSyntax` JS output](#savol-14-verbatimmodulesyntax-js-output-middle) [Middle+]
15. [`import type` bilan runtime ishlatish](#savol-15-import-type-bilan-runtime-ishlatish-junior) [Junior+]
16. [Module augmentation tartibi](#savol-16-module-augmentation-tartibi-middle) [Middle+]
17. [Module resolution conflict](#savol-17-module-resolution-conflict-middle) [Middle+]
18. [`paths` runtime](#savol-18-paths-runtime-middle) [Middle]
19. [`isolatedModules` const enum](#savol-19-isolatedmodules-const-enum-middle) [Middle]

### Coding challenges
20. [`process.env` type-safe](#savol-20-processenv-type-safe-middle) [Middle+]
21. [Type-only barrel](#savol-21-type-only-barrel-middle) [Middle]
22. [Vite + React tsconfig konfiguratsiya](#savol-22-vite--react-tsconfig-konfiguratsiya-middle) [Middle+]

### Bug fix savollari
23. [Module augmentation override](#savol-23-module-augmentation-override-middle) [Middle+]
24. [Circular dependency bug](#savol-24-circular-dependency-bug-middle) [Middle+]

---

## Nazariy savollar

### Savol 1: `import type` va oddiy `import` farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`import type` — faqat type-level import. JS ga compile qilinganda to'liq o'chiriladi. Oddiy `import` — runtime'da mavjud, JS output ga kiradi.

### To'liq tushuntirish

TypeScript 3.8 da qo'shilgan `import type` syntax — type-only import explicit belgilash. Bu statement JS ga compile qilinganda **to'liq o'chiriladi** — runtime'da hech qanday iz qolmaydi.

**Asosiy farqlar:**

| Xususiyat | `import` | `import type` |
|-----------|----------|---------------|
| JS output da qoladi | ✅ | ❌ |
| `new Class()` qilish mumkin | ✅ | ❌ |
| Type annotation'da | ✅ | ✅ |
| Bundle size ta'siri | Ha | Yo'q |
| Side effects ishlaydi | ✅ | ❌ |
| Circular dependency | Muammo | Hal qiladi |

**Nima uchun kerak:**

1. **Bundle size** — type-only import bundler tomonidan aniq olib tashlanadi
2. **Circular dependencies** — type-only import runtime'da yo'q → circular uziladi
3. **Side effects** — `import "./polyfill"` kabi side effect import'larni type-only qilib bo'lmaydi (runtime'da kerak)
4. **Aniqlik** — code review'da type va value import'lar darhol ko'rinadi
5. **Bundler compatibility** — esbuild/SWC type-only import'larni aniq biladi

### Kod misol

```typescript
// types.ts
export interface User { id: number; name: string }
export type Role = "admin" | "user";
export class UserService {
  getAll(): User[] { return []; }
}
```

```typescript
// app.ts
import type { User, Role } from "./types";   // ← type-only
import { UserService } from "./types";        // ← runtime kerak

const service = new UserService();             // ✅ — UserService runtime'da bor
const users: User[] = service.getAll();        // ✅ — User type annotation'da

// ❌ Error — import type bilan kelgan User'ni value sifatida ishlatib bo'lmaydi
// const role: Role = Role; // 'Role' cannot be used as a value (import type)
```

**Inline syntax (TS 4.5+):**

```typescript
// Bitta statement'da aralash
import { type User, type Role, UserService } from "./types";
//        ^^^^      ^^^^      type-only       value
```

**JS output:**

```javascript
// app.js — TypeScript compile natijasi
import { UserService } from "./types";
// User, Role — o'chirildi

const service = new UserService();
const users = service.getAll();
```

### Edge Cases

- **Side-effect import** — `import "./polyfill"` — value import. `import type "./polyfill"` syntax xato (type-only side effect mumkin emas).
- **Re-export** — `export type { User } from "./types"` — type-only re-export. JS da o'chadi.
- **Default + named aralash** — `import type Default, { Named } from "./mod"` — ikkala ham type-only.
- **`typeof` value position'da** — `import type { OrderService } from "./services"; type T = typeof OrderService` — error! `typeof` value position'da `OrderService`'ni runtime'da talab qiladi.

### Follow-up savollar

1. **"`import { type X }` va `import type { X }` farqi?"** — Funksional bir xil. Inline form (TS 4.5+) bitta statement'da type va value aralashtirish imkonini beradi.
2. **"`export type` ham bor mi?"** — Ha. `export type { User }` yoki `export { type User }` — type-only re-export.

</details>

---

### Savol 2: `verbatimModuleSyntax` (TS 5.0+) nima va nima uchun yaxshiroq? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`verbatimModuleSyntax` — TS 5.0 da qo'shilgan tsconfig option. Import/export'larni **aynan yozilgan holatda** emit qiladi (heuristic yo'q). `import type` ni majburiy qiladi.

### To'liq tushuntirish

**Avval (heuristic-based):**

Standard `import { X }` bilan compiler aniqlashga harakat qilardi — `X` faqat type position'da ishlatilganmi? Agar ha — JS output'dan olib tashlardi. Aks holda — qoldirardi.

Muammolar:
- **Bundler bilan moslashmas** — esbuild/SWC fayllarni alohida transpile qiladi, full type information yo'q
- **Re-export edge case** — `export { X } from "./mod"` da X type bo'lsa ham, transpiler bilmaydi
- **Predictability past** — yozilgan import va emit natijasi har xil

**Yangi (verbatim):**

`verbatimModuleSyntax: true` — compiler hech qanday heuristic qo'llamaydi. Yozilgan aynan o'sha holatda emit qilinadi:

- `import { X }` → `import { X }` (JS'da qoladi)
- `import type { X }` → o'chiriladi
- `import { type X }` → o'chiriladi (inline type)

Agar `X` type bo'lsa va `import type` ishlatmasangiz — compile error.

**Eski option'larni almashtiradi:**
- `importsNotUsedAsValues` (deprecated)
- `preserveValueImports` (deprecated)

**Nima uchun yaxshiroq:**
1. **Predictability** — emit natijasi 100% yozilgan code'ga mos
2. **Bundler-friendly** — esbuild/SWC fayl-bo'yicha transpile bilan moslashadi
3. **Aniqlik** — har import type/value ekanligi source'da ko'rinadi
4. **Future-proof** — Node.js native TypeScript stripping bilan moslashadi

### Kod misol

```json
// tsconfig.json
{
  "compilerOptions": {
    "verbatimModuleSyntax": true,
    "module": "esnext",
    "moduleResolution": "bundler"
  }
}
```

```typescript
// types.ts
export interface User { id: number; name: string }
export class UserService {
  getAll(): User[] { return []; }
}
```

```typescript
// ❌ Error — verbatimModuleSyntax: true
import { User, UserService } from "./types";
// "'User' is a type and must be imported using a type-only import
//  when 'verbatimModuleSyntax' is enabled."

// ✅ To'g'ri — inline type
import { type User, UserService } from "./types";

// ✅ To'g'ri — alohida statement
import type { User } from "./types";
import { UserService } from "./types";
```

**Re-export:**

```typescript
// ❌ Error
export { User } from "./types";

// ✅ Type-only re-export
export type { User } from "./types";
```

**Side-effect import — saqlanadi:**

```typescript
// ✅ — value import (no specifier), JS'da qoladi
import "./polyfill";
import "reflect-metadata";
```

### Edge Cases

- **`isolatedModules` bilan birga ishlatish** — odatda ikkalasini ham yoqish kerak. Birgalikda strict module semantics.
- **`module: "node16"` bilan** — `verbatimModuleSyntax` Node.js ESM/CJS aniq ajratiladi. `.mts` ESM, `.cts` CJS.
- **Library publish** — `verbatimModuleSyntax: true` bilan library'ning emit natijasi predictable. Consumer'lar bundle qilganda muammo kamayadi.

### Follow-up savollar

1. **"`importsNotUsedAsValues: 'error'` bilan farq?"** — `importsNotUsedAsValues` faqat **diagnostic** beradi (warning). `verbatimModuleSyntax` esa **emit** semantikasini ham o'zgartiradi. Yangi standart va to'liqroq.
2. **"`verbatimModuleSyntax` `esModuleInterop` bilan ishlaydi?"** — Ha, lekin CommonJS interop default import'larida (`import X from "cjs-module"`) developer e'tibor talab qiladi. `verbatim` heuristic yo'q.

</details>

---

### Savol 3: Module resolution strategiyalari — qachon qaysi biri? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`"node"` — legacy CJS. `"node16"`/`"nodenext"` — Modern Node.js (CJS + ESM). `"bundler"` — Vite/Webpack/esbuild. `"classic"` — deprecated, ishlatilmasin.

### To'liq tushuntirish

Module resolution — compiler `import "./user"` ni qaysi faylga point qilishini aniqlash jarayoni. TypeScript bir nechta strategiya taklif qiladi:

| Strategy | Extension talab | `exports` field | `package.json "type"` | Muhit |
|----------|:---------------:|:---------------:|:---------------------:|-------|
| `"node"` | Yo'q | ❌ | Inkor qiladi | Legacy CJS |
| `"node16"` | ESM uchun majburiy | ✅ | E'tiborga oladi | Modern Node.js |
| `"nodenext"` | ESM uchun majburiy | ✅ | E'tiborga oladi | Future Node.js |
| `"bundler"` | Yo'q | ✅ | E'tiborga oladi | Vite/Webpack/esbuild |
| `"classic"` | Yo'q | ❌ | Yo'q | Deprecated |

**`"node"` resolution:**

```
import { User } from "./user"

1. ./user.ts
2. ./user.tsx
3. ./user.d.ts
4. ./user/index.ts
5. ./user/index.tsx
6. ./user/index.d.ts
7. ./user.js (allowJs bo'lsa)
```

Hech qachon `exports` field qaramaydi — eski Node.js compatibility.

**`"node16"` / `"nodenext"` resolution:**

Node.js ESM/CJS specification bo'yicha:
- `package.json "type": "module"` → ESM context
- `package.json "type": "commonjs"` yoki yo'q → CJS context
- `.mts` fayl → ESM
- `.cts` fayl → CJS
- `.ts` fayl → kontekstga qarab

ESM context'da relative import **extension majburiy**:

```typescript
// ❌ Error in ESM context
import { User } from "./user";

// ✅ — .js yoziladi (compile natijasi)
import { User } from "./user.js";
// Compiler .js → .ts mapping qiladi
```

**`"bundler"` resolution (TS 5.0+):**

Bundler context'da:
- Extension-less import ruxsat
- `exports` field qo'llab-quvvatlaydi
- `package.json "type"` e'tiborga oladi
- Conditional exports (`"import"`, `"require"`, `"types"`)

**Tanlash matrix:**

| Loyiha | Strategy |
|--------|----------|
| Vite + React/Vue | `"bundler"` |
| Next.js (app router) | `"bundler"` |
| Node.js ESM API (Fastify, Hono) | `"node16"` yoki `"nodenext"` |
| Node.js legacy CJS | `"node"` |
| Library (npm publish) | `"nodenext"` |
| Monorepo workspace | `"bundler"` yoki `"node16"` |

### Kod misol

```json
// Vite + React tsconfig.json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true,
    "isolatedModules": true
  }
}
```

```typescript
// ✅ Extension-less ruxsat
import { Button } from "./components/Button";
import { format } from "date-fns";
```

```json
// Node.js ESM API tsconfig.json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext"
  }
}
```

```typescript
// ✅ Extension majburiy
import { router } from "./routes.js";
import { logger } from "./utils/logger.js";
```

```json
// Library tsconfig.json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "declaration": true,
    "emitDeclarationOnly": true
  }
}
```

### Edge Cases

- **`"node16"` da `.js` extension** — `.ts` source faylga `.js` extension yozish confusing ko'rinadi, lekin mantiq: compile natijasi `.js`. Compiler `.js` → `.ts` mapping qiladi.
- **`exports` field birinchi mos** — `package.json` da `exports` ichida `"types"` **birinchi** bo'lishi kerak. Aks holda compiler `.mjs` ni topib `.d.ts` o'rniga uni o'qiydi.
- **`"node"` deprecated emas, lekin kam ishlatiladi** — Legacy CJS only loyihalar uchun saqlangan.
- **Conditional exports parsing** — `"node16"` va `"bundler"` `package.json` `exports` ichidagi conditional (`"node"`, `"browser"`, `"types"`) ni o'zicha resolve qiladi.

### Follow-up savollar

1. **"`node16` va `nodenext` farqi?"** — `node16` — Node.js 16 spec'iga to'g'ri. `nodenext` — keyingi Node versiyalariga moslashadi (forward-compatible). Library larda `nodenext` afzal.
2. **"`bundler` resolution Node.js'da ishlaydimi?"** — Yo'q. Pure type-checking uchun. Build vaqtida bundler (Vite, Webpack) yoki tsx/ts-node runtime resolve qiladi.

</details>

---

### Savol 4: `isolatedModules` nima va qanday cheklovlar qo'yadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedModules: true` — har fayl boshqa fayllar type ma'lumotisiz mustaqil transpile qilinishi mumkin bo'lishini kafolatlaydi. esbuild, SWC, Babel kabi tezkor transpiler lar bilan moslashish uchun.

### To'liq tushuntirish

Standard TypeScript compiler (tsc) butun project type information ga ega — har fayl context bilan compile bo'ladi. Lekin zamonaviy transpiler lar (esbuild, SWC, Babel) tezlik uchun har fayl ni **alohida** parse qiladi — type information yo'q.

`isolatedModules: true` shunday cheklovlar qo'yadi:

**1. Type-only re-export `export type` bilan:**

```typescript
// ❌ — transpiler User type yoki value ekanini bilmaydi
export { User } from "./types";

// ✅ — explicit type re-export
export type { User } from "./types";
```

**2. `const enum` cross-file inline mumkin emas:**

`const enum` value'lar compile-time'da inline replace bo'ladi. Bu boshqa fayldan type information talab qiladi. Transpiler bilmaydi.

`preserveConstEnums: true` (TS 1.4'dan beri mavjud) const enum'ni inline qilish o'rniga oddiy enum kabi runtime object emit qiladi → transpiler bilan ishlaydi. Lekin bu inline optimization'dan voz kechadi.

```typescript
// ❌ — cross-file inline mumkin emas
export const enum Color { Red, Green, Blue }

// ✅ — oddiy enum (runtime object)
export enum Color { Red, Green, Blue }

// ✅ — union type (yaxshiroq alternative)
export type Color = "red" | "green" | "blue";
```

**3. Har fayl module bo'lishi kerak:**

```typescript
// ❌ — import/export yo'q → script
const PI = 3.14;

// ✅ — module
export const PI = 3.14;

// ✅ — yoki bo'sh export
const PI = 3.14;
export {};
```

**4. `namespace` cross-file merging muammoli:**

Namespace `namespace Validation { ... }` ikki turli faylda declare qilingan bo'lsa — transpiler bir-biriga merge qilolmaydi (har faylni alohida ko'radi).

**Qachon yoqish:** Deyarli doim. Vite, Next.js, Remix, esbuild ishlatayotgan bo'lsangiz — yoqing.

### Kod misol

```json
{
  "compilerOptions": {
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  }
}
```

```typescript
// constants.ts
export const API_URL = "https://api.example.com";
export const MAX_RETRIES = 3;

// ❌ Error — const enum
// export const enum LogLevel { Debug, Info, Warn, Error }

// ✅ — oddiy enum yoki union
export enum LogLevel { Debug, Info, Warn, Error }
// yoki
export type LogLevel = "debug" | "info" | "warn" | "error";
```

```typescript
// types/index.ts — barrel
// ❌ Error
export { User } from "./user";

// ✅ — type explicit
export type { User } from "./user";

// ✅ — value re-export (User runtime'da kerak bo'lsa)
import { User } from "./user";
export { User };
```

### Edge Cases

- **`tsc` (`isolatedModules: false`) vs `esbuild`** — `tsc` `const enum` ni inline qiladi, lekin esbuild qila olmaydi → runtime'da error.
- **`namespace` declaration merging** — `isolatedModules` bilan bir fayl ichida ishlaydi, lekin cross-file emas.
- **Script (non-module) fayllar** — Type-only fayl ni script qoldirsa, `isolatedModules` `export {}` qo'shishni majburlaydi.

### Follow-up savollar

1. **"`isolatedModules` va `verbatimModuleSyntax` ikkalasini ham yoqish kerakmi?"** — Ha, ikkalasi to'ldiruvchi. `isolatedModules` strict transpilation, `verbatim` strict import/export emit.
2. **"`preserveConstEnums: true` nima qiladi?"** — `const enum`'ni inline qilish o'rniga oddiy enum kabi runtime object emit qiladi. Const enum project bo'ylab shu option bilan compile qilinsa, transpiler `Color.Red` ni runtime object'da topadi → `isolatedModules` muammosi yo'qoladi (inline optimization'dan voz kechiladi).

</details>

---

### Savol 5: Module augmentation — qanday ishlaydi va `import` nima uchun kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Module augmentation — mavjud module'ning type'larini kengaytirish. `declare module "name"` bilan amalga oshadi. Module faylda (`import`/`export` bor) `declare module` aniq **augmentation** sifatida tan olinadi; script faylda esa **ambient module declaration** bo'ladi — u ham mavjud module bilan merge qiladi, lekin niyat noaniq va target xato bo'lsa jim ishlamaydi.

### To'liq tushuntirish

TypeScript da fayl ikki turdan biri:
- **Module** — fayl ichida kamida bitta `import` yoki `export` bor
- **Script** — global scope'da ishlaydi, import/export yo'q

`declare module "name"` script va module faylda **har xil ishlaydi**:

**Script faylda:**

```typescript
// types.d.ts (import/export YO'Q)
declare module "express" {
  interface Request { userId: string; }
}
```

Bu **ambient module declaration**. Bir xil nomli ambient `declare module "express"` declaration'lar (`@types/express`'dagi bilan ham) **merge bo'ladi** — almashtirmaydi. Lekin niyat noaniq: bu yerda `Request` `express` module scope'ida ochiladi. Agar haqiqiy `Request` boshqa module'da (modern `@types/express`'da — `express-serve-static-core`) bo'lsa, augmentation noto'g'ri target'ga tushadi va jim ishlamaydi (16 va 23-savollarga qarang).

**Module faylda:**

```typescript
// types.d.ts
import "express"; // ← faylni MODULE qiladi

declare module "express-serve-static-core" {
  interface Request { userId: string; }
}
```

Endi `declare module "express-serve-static-core"` — **module augmentation**. Modern `@types/express`'da `Request` aynan shu module'da yashaydi, shuning uchun augmentation haqiqiy `Request` interface'iga merge bo'ladi (interface declaration merging mexanizmi orqali).

**Mexanizm:**
1. `import "express"` faylni module qiladi va Express'ni resolve qiladi
2. Compiler `declare module "express-serve-static-core"` ni augmentation deb tan oladi
3. `Request` interface ikki marta declare qilingan (`@types`'dagi + augmentation'dagi) → merge

**Real-world misollar:**
- Express `Request` ga `userId`, `user`, `role` qo'shish
- `process.env` type-safe qilish
- Library'ga yangi method qo'shish

### Kod misol

```typescript
// types/express.d.ts
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
// middleware/auth.ts
import { Request, Response, NextFunction } from "express";

export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  req.userId = "user-42";       // ✅ — augmented
  req.user = { id: "42", role: "admin" }; // ✅
  req.requestId = crypto.randomUUID();    // ✅
  next();
}
```

**`process.env` augmentation:**

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
const env = process.env.NODE_ENV;     // "development" | "production" | "test"
const db = process.env.DATABASE_URL;   // string
// process.env.RANDOM_VAR;             // ❌ — declared emas
```

**Library augmentation (Day.js plugin):**

```typescript
// types/dayjs-plugin.d.ts
import "dayjs";

declare module "dayjs" {
  interface Dayjs {
    customMethod(): string;
  }
}
```

### Edge Cases

- **Express ichida `express-serve-static-core`** — Real-world Express `Request` interface aslida `express-serve-static-core` package'da. Augmentation o'sha module'ga yo'naltirilishi kerak (yuqoridagi misol).
- **Sub-module augmentation** — `declare module "lodash/fp"` — lodash'ning sub-path. Sub-path resolution kerak.
- **`import "express"` o'rniga `export {}`** — ikkalasi ham faylni module qiladi. Lekin `import` Express ni resolve qiladi va aniq augmentation niyati ko'rinadi.
- **`tsconfig.json` `include`** — augmentation fayli `include` da bo'lishi kerak, aks holda compiler ko'rmaydi.

### Follow-up savollar

1. **"Augmentation ichida `class` declare qilish mumkin?"** — Yo'q. Faqat interface (declaration merging) va type aliases (limited). Class augmentation declaration merging cheklovi tufayli to'liq ishlamaydi.
2. **"Library'ga yangi default export qo'shish mumkinmi?"** — Yo'q. Default export augmentation cheklangan — interface yangi method qo'shish mumkin, lekin yangi default value qo'shish mumkin emas.

</details>

---

### Savol 6: `declare global` nima va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`declare global` — module fayl ichida global scope'ga yangi type, variable, function declare qilish. `export {}` bilan faylni module qilish kerak.

### To'liq tushuntirish

Standart TypeScript da `declare const`, `declare function` global scope'ga directly qo'shiladi — agar fayl **script** bo'lsa. Lekin modern project'larda barcha fayllar module — global declaration ishlamaydi.

`declare global { ... }` blok — module fayl ichidan global scope'ni augment qilish:

```typescript
// faylni module qilish uchun
export {};

declare global {
  // global type'lar va variable'lar
}
```

**`export {}` nima uchun:** `declare global` faqat **module** fayl ichida ishlaydi. Bo'sh export faylni module qiladi (import/export bor — module).

**Tipik ishlatilishi:**

1. **`window` augmentation** — third-party script `window` ga property qo'shganda
2. **Global variable** — Webpack `DefinePlugin` orqali inject qilingan
3. **Polyfill type** — global `structuredClone`, `crypto.randomUUID`
4. **`process.env`** — Node.js (alternative: `NodeJS.ProcessEnv` augmentation)

### Kod misol

```typescript
// globals.d.ts
export {}; // ← faylni module qiladi

declare global {
  // Window augmentation
  interface Window {
    __APP_VERSION__: string;
    __FEATURE_FLAGS__: Record<string, boolean>;
    gtag: (command: "event" | "config", name: string, params?: object) => void;
    dataLayer: Array<Record<string, unknown>>;
  }

  // Global function
  function sleep(ms: number): Promise<void>;

  // Global variable (var — function-scoped, global)
  var APP_NAME: string;
  var __DEV__: boolean;

  // Global type
  type Maybe<T> = T | null | undefined;
}
```

```typescript
// app.ts
console.log(window.__APP_VERSION__);  // ✅ string
window.gtag("event", "page_view");     // ✅
await sleep(1000);                     // ✅ global function
const m: Maybe<string> = null;         // ✅ global type
```

**Webpack DefinePlugin bilan:**

```typescript
// webpack.config.js (DefinePlugin)
new webpack.DefinePlugin({
  "__APP_VERSION__": JSON.stringify("1.0.0"),
  "__DEV__": JSON.stringify(true),
});

// globals.d.ts
export {};
declare global {
  const __APP_VERSION__: string;
  const __DEV__: boolean;
}

// app.ts
if (__DEV__) {
  console.log(`App v${__APP_VERSION__}`);
}
```

### Edge Cases

- **`var` vs `const` global da** — `var` haqiqiy global, `window.x` orqali ham access. `const`/`let` — global scope'da, lekin `window` ga qo'shilmaydi (ES module semantika).
- **`declare global` ichida `import`** — TypeScript 4.5+ da `import("module").Type` ishlaydi global blok ichida.
- **DOM types va `lib`** — `lib: ["DOM"]` bo'lsa Window interface mavjud. `declare global` orqali augmentation. `lib: ["ESNext"]` bilan DOM yo'q.
- **TypeScript 5.5 `using`/`await using`** — yangi global type'lar (Disposable, AsyncDisposable). `lib` kerak.

### Follow-up savollar

1. **"`declare global` faqat .d.ts'da ishlaydimi?"** — Yo'q. .ts faylda ham ishlaydi, agar fayl module bo'lsa (`export {}` yoki boshqa import/export).
2. **"`window.__APP_VERSION__ = "1.0.0"` runtime'da kerak emasmi?"** — Type-level declare runtime qiymat yaratmaydi. Developer o'zi assign qilishi kerak (HTML script tag, DefinePlugin, polyfill).

</details>

---

### Savol 7: Inline type imports — `import { type X }` farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Inline type imports (TS 4.5+) — bitta `import` statement ichida ba'zi nomlarni `type` deb belgilash. `import type { X }` bilan funksional bir xil, lekin bitta statement'da type va value aralashtirish imkonini beradi.

### To'liq tushuntirish

TypeScript 4.5 da inline type modifier qo'shilgan. Avval type va value uchun alohida import statement kerak edi:

```typescript
// TS 4.5'gacha
import type { User, Role } from "./types";
import { UserService } from "./types";
```

TS 4.5+ da bir statement'da aralashtirish mumkin:

```typescript
import { type User, type Role, UserService } from "./types";
```

**Behavior:**
- `type User` — type-only (compile-time, JS'da o'chadi)
- `UserService` — value (JS'da qoladi)

JS output:

```javascript
import { UserService } from "./types";
```

**`import type` vs inline type:**

| Form | Hamma type | Aralash | Default + named |
|------|:----------:|:-------:|:---------------:|
| `import type { X, Y } from "..."` | ✅ | ❌ | `import type Default, { Named }` |
| `import { type X, Y } from "..."` | ❌ | ✅ | `import Default, { type Named }` |

**Qachon qaysi:**
- Hamma import type-only → `import type` (clean syntax)
- Aralash type/value → inline `type` modifier

### Kod misol

```typescript
// product.ts
export interface Product { id: number; name: string; price: number }
export type Category = "electronics" | "clothing" | "food";
export class ProductService {
  getAll(): Product[] { return []; }
}
export const TAX_RATE = 0.12;
```

```typescript
// shop.ts — inline type
import { type Product, type Category, ProductService, TAX_RATE } from "./product";

const service = new ProductService();
const items: Product[] = service.getAll();
const category: Category = "electronics";
const totalWithTax = items.reduce((s, p) => s + p.price, 0) * (1 + TAX_RATE);
```

```typescript
// Equivalent — alternative syntax
import type { Product, Category } from "./product";
import { ProductService, TAX_RATE } from "./product";
```

**Default + named aralash:**

```typescript
// AuthService default, User type
import Auth, { type User } from "./auth";

// Default type, named value — bitta statement'da bo'lmaydi
import type Config, { logger } from "./app";
// ❌ Bu syntax to'g'ri, lekin `import type` prefix HAMMASINI type-only qiladi:
//    Config ham, logger ham type-only bo'ladi. logger value bo'lsa — runtime'da yo'qoladi.

// To'g'ri — type va value alohida statement:
import type Config from "./app";
import { logger } from "./app";
```

### Edge Cases

- **`verbatimModuleSyntax: true` bilan** — type modifier majburiy bo'ladi. Compiler heuristic'siz.
- **Re-export inline** — `export { type X, Y } from "./mod"` — inline syntax export uchun ham ishlaydi.
- **Default import** — `import type Default, { Named }` — ikkala ham type. Aralash uchun alohida statement kerak.

### Follow-up savollar

1. **"`import { type X as Y }` ishlaydimi?"** — Ha. Inline type + rename: `import { type Product as Item } from "./mod"`.
2. **"JS output da aniq qanday ko'rinadi?"** — Type'lar olib tashlanadi: `import { Service } from "./mod"`. Faqat value'lar qoladi.

</details>

---

### Savol 8: Path aliases (`paths`/`baseUrl`) qanday ishlaydi va nima uchun bundler kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`paths` va `baseUrl` — `tsconfig.json` da import yo'llarini qisqartirish. Lekin TypeScript faqat **type-checking** uchun ishlatadi — compile vaqtida path'larni almashtirmaydi. Runtime'da bundler (Vite, Webpack) yoki tool (tsconfig-paths) kerak.

### To'liq tushuntirish

**Path aliases** — relative path (`../../../../`) o'rniga semantik alias (`@components/Button`) ishlatish.

**`tsconfig.json`:**

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

- `baseUrl` — `paths` resolution'ning starting point
- `paths` — alias → real path mapping. `*` glob pattern

**Critical fakt:** TypeScript `paths` ni faqat type-checking uchun ishlatadi. **Compile vaqtida path'larni almashtirmaydi** — JS output da alias o'zgarmagan holatda qoladi:

```typescript
// TS source
import { Button } from "@components/Button";

// JS output (TypeScript compile natijasi)
import { Button } from "@components/Button"; // ← O'ZGARMAGAN!
```

Bu JS faylni Node.js yoki browser ishga tushurganda `@components/Button` ni topa olmaydi.

**Yechim:** Bundler yoki runtime tool kerak:

1. **Vite** — `vite.config.ts` da `resolve.alias`
2. **Webpack** — `resolve.alias`
3. **Next.js** — tsconfig'dan avtomatik o'qiydi
4. **Jest** — `moduleNameMapper`
5. **Node.js runtime** — `tsconfig-paths` package
6. **esbuild** — `alias` option

### Kod misol

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

```typescript
// Oldin — relative
import { Button } from "../../../../components/Button";
import { formatDate } from "../../../utils/date";

// Keyin — alias
import { Button } from "@components/Button";
import { formatDate } from "@utils/date";
```

**Vite konfiguratsiyasi:**

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import path from "node:path";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
      "@components": path.resolve(__dirname, "./src/components"),
      "@utils": path.resolve(__dirname, "./src/utils"),
    },
  },
});
```

**Jest konfiguratsiyasi:**

```javascript
// jest.config.js
module.exports = {
  moduleNameMapper: {
    "^@/(.*)$": "<rootDir>/src/$1",
    "^@components/(.*)$": "<rootDir>/src/components/$1",
  },
};
```

**Workflow:**

```
TS source:    import { Button } from "@components/Button"
                 │
TS compiler:  ✅ Type check — paths bo'yicha topildi
                 │
JS output:    import { Button } from "@components/Button" ← o'zgarmagan
                 │
Bundler/Tool: → src/components/Button.js ← resolve qiladi
```

### Edge Cases

- **`baseUrl` siz `paths`** — TS 5.0+ da `baseUrl` ixtiyoriy. `paths` o'zi yetadi.
- **Triple-slash path** — `paths` `/// <reference path="..." />` ga ta'sir qilmaydi. Triple-slash o'zi resolve qiladi.
- **`exports` field bilan conflict** — `paths` external library'larga ham qo'llaniladi. Library `exports` field bilan boshqa path qaytarsa, conflict bo'lishi mumkin.
- **Monorepo workspace** — `paths` o'rniga workspace dependencies (`npm workspace`, `pnpm`, `yarn workspaces`) afzal — runtime'da ham ishlaydi.

### Follow-up savollar

1. **"`@types/*` namespace bilan `@/*` alias conflict bo'ladimi?"** — Ehtimol bor. `@types` package resolution alohida tarmoq. Lekin agar alias `@types/*` deb belgilansa — conflict yuzaga keladi.
2. **"`tsconfig-paths` runtime'da qanday ishlaydi?"** — Node.js `require` ni patch qiladi va `paths` mapping'ga ko'ra resolve qiladi. Production'da bundler afzal — runtime overhead yo'q.

</details>

---

### Savol 9: Barrel exports — afzalliklari va muammolari? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Barrel — `index.ts` orqali papka export'larini birlashtirish. Toza import'lar, lekin tree shaking buzilishi va circular dependency xavfi bor. Type-only barrel xavfsiz.

### To'liq tushuntirish

**Barrel pattern:**

```typescript
// models/index.ts (barrel)
export { User } from "./user";
export { Product } from "./product";
export { Order } from "./order";
```

Foydalanish:

```typescript
// Oldin
import { User } from "./models/user";
import { Product } from "./models/product";

// Barrel bilan
import { User, Product } from "./models";
```

**Afzalliklari:**
1. **Toza import'lar** — fayl path emas, modul path
2. **Refactoring oson** — fayllar reorganizatsiya qilinsa, barrel saqlaydi
3. **Public API aniq** — qaysi narsalar export qilingan ko'rinadi
4. **Cleaner consumer code** — bitta `from "./models"`

**Muammolari:**

**1. Tree shaking buzilishi:**

Modern bundler (Vite, Webpack, Rollup) tree shaking qiladi — kerak bo'lmagan code'ni olib tashlaydi. Lekin barrel ko'pincha bunga to'sqinlik qiladi:

```typescript
// app.ts
import { User } from "./models";
// Bundler:
// 1. models/index.ts ni o'qiydi
// 2. user.ts ni resolve qiladi
// 3. Lekin user.ts'da side effect bo'lsa — product.ts, order.ts ham load bo'ladi
```

Side effect lar tree shaker'ni "shubhalantiradi" — barcha barrel exports'ni load qilishi mumkin.

**2. Circular dependency xavfi:**

```typescript
// user.ts
import { Post } from "./index"; // barrel orqali
// post.ts
import { User } from "./index"; // barrel orqali

// Circular: user → index → post → index → user
```

**3. IDE auto-import:**

IDE odatda barrel'dan import qo'shadi (fayl path emas) — bu noaniqlik yaratadi.

**4. Compile time impact:**

Barrel bilan har import butun barrel'ni resolve qiladi → compile time oshadi.

### Kod misol

**Value barrel — ehtiyot bo'lish kerak:**

```typescript
// models/index.ts
export { User } from "./user";
export { Product } from "./product";
export { UserService } from "./user-service";
export { ProductService } from "./product-service";
```

```typescript
// app.ts
import { UserService } from "./models";
// Bundler — `models` ni butun load qilishi mumkin
// `product.ts` va `product-service.ts` ham bundle ga kiradi
```

**Type-only barrel — muammosiz:**

```typescript
// models/index.ts — TYPE-ONLY
export type { User } from "./user";
export type { Product } from "./product";
export type { Order } from "./order";
```

```typescript
// app.ts
import type { User, Product } from "./models";
// JS output da hech narsa — type-only
// Tree shaking ta'sirlanmaydi ✅
```

**Aralash barrel — `verbatimModuleSyntax` bilan:**

```typescript
// models/index.ts
export type { User } from "./user";
export type { Product } from "./product";
export { UserService } from "./user-service";
export { ProductService } from "./product-service";
```

**Selective barrel — eng yaxshi praktika:**

Faqat public API ni barrel'da, internal narsalar to'g'ridan-to'g'ri import:

```typescript
// models/index.ts — public API
export type { User, UserRole } from "./user";
export { UserService } from "./user-service";

// _internal/index.ts emas — internal narsalar to'g'ridan-to'g'ri
```

### Edge Cases

- **`/* @__PURE__ */` annotation** — Bundler'larga "bu function call side effect siz" deb aytadi. Tree shaking yaxshiroq.
- **`package.json "sideEffects": false`** — Library export'ida side effect yo'q deb belgilash. Bundler eng agressive tree shaking.
- **Re-export rename** — `export { User as Customer } from "./user"`. Alias bilan re-export.
- **`export * from`** — Barcha export'larni propagatsiya. Lekin name collision xavfi.

### Follow-up savollar

1. **"`export *` va explicit re-export farqi?"** — `export *` barchasini propagatsiya qiladi. Explicit (`export { X, Y }`) — controlled. `export *` da name collision compile error.
2. **"Barrel ishlatmaslik tavsiya etiladimi?"** — Library publish'da ehtiyot bo'lish kerak (tree shaking). App'da pragmatik — type-only barrel muammosiz.

</details>

---

### Savol 10: Dynamic imports — type safety qanday saqlanadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`import()` expression — runtime'da module'ni lazy load. TypeScript qaytaradigan promise type'ini avtomatik `Promise<typeof import("./module")>` deb infer qiladi. To'liq type-safe.

### To'liq tushuntirish

**Statik import** — module darhol load bo'ladi (start-up time):

```typescript
import { HeavyChart } from "./heavy-chart";
```

**Dynamic import** — `import()` expression bilan kerak bo'lganda load:

```typescript
async function showChart() {
  const mod = await import("./heavy-chart");
  // mod type: typeof import("./heavy-chart")
  // = { HeavyChart: typeof HeavyChart; createChart: (...) => HeavyChart; ... }
}
```

`import()` ES specification'ning bir qismi. TypeScript bu expression'ning return type'ini avtomatik infer qiladi — `Promise<typeof Module>`.

**Type-safe access:**

```typescript
async function load() {
  const { HeavyChart, createChart } = await import("./heavy-chart");
  // Destructuring — har bir nom to'g'ri type bilan
  
  const chart = new HeavyChart();       // ✅ — class
  const c2 = createChart();              // ✅ — function
  chart.render();                        // ✅ — method
}
```

**Conditional loading:**

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

**Type-only dynamic import** — runtime'da bo'lmaydi:

```typescript
// Faqat type olish — runtime'da import bo'lmaydi
type ChartType = import("./heavy-chart").HeavyChart;
type ChartModule = typeof import("./heavy-chart");

// const X = import("./mod"); // ← runtime import (Promise)
// type X = import("./mod"); // ← pure type, runtime'da yo'q
```

### Kod misol

```typescript
// heavy-chart.ts
export class HeavyChart {
  render(): string { return "chart rendered"; }
}

export function createChart(): HeavyChart {
  return new HeavyChart();
}

export const CHART_VERSION = "2.0";
```

```typescript
// app.ts
async function loadAndRender(): Promise<string> {
  // Lazy load — chart kerak bo'lganda
  const { HeavyChart, CHART_VERSION } = await import("./heavy-chart");
  console.log(`Using chart v${CHART_VERSION}`);
  
  const chart = new HeavyChart();
  return chart.render();
}

// Webpack/Vite kabi bundler'lar dynamic import'ni separate chunk qiladi
// → ./assets/heavy-chart-abc123.js (code split)
```

**Conditional module:**

```typescript
type Locale = "uz" | "en" | "ru";

async function getTranslations(locale: Locale) {
  switch (locale) {
    case "uz": {
      const mod = await import("./i18n/uz");
      return mod.translations;
    }
    case "en": {
      const mod = await import("./i18n/en");
      return mod.translations;
    }
    case "ru": {
      const mod = await import("./i18n/ru");
      return mod.translations;
    }
  }
}
```

**Type-only — circular dependency uchun:**

```typescript
// user.ts
type Post = import("./post").Post; // type-only, runtime'da yo'q

export class User {
  posts: Post[] = [];
}

// post.ts
type User = import("./user").User; // type-only

export class Post {
  author: User;
  constructor(author: User) { this.author = author; }
}
```

### Edge Cases

- **`module: "commonjs"`** — `import()` `require()` ga compile qilinadi. Behavior bir xil, lekin chunking ishlamasligi mumkin.
- **`module: "esnext"` va Vite/Webpack** — bundler `import()` ni alohida chunk qiladi (code splitting).
- **`await import()` Promise rejection** — module load fail bo'lsa, Promise reject. Try/catch shart.
- **Webpack magic comments** — `import(/* webpackChunkName: "chart" */ "./chart")` chunk nomini belgilash.

### Follow-up savollar

1. **"`import()` Promise'ni `then` bilan ishlatish mumkinmi?"** — Ha. `import("./mod").then(mod => mod.X())`. Async/await syntactic sugar.
2. **"Dynamic import bilan SSR'da nima bo'ladi?"** — Server-side'da ham ishlaydi. Lekin Next.js kabi framework'larda dynamic import bilan client-only module belgilanadi.

</details>

---

### Savol 11: Circular dependencies — `import type` bilan qanday hal qilinadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`import type` JS output'da to'liq o'chiriladi → runtime'da modullar bir-birini import qilmaydi → circular dependency uziladi. Agar value ham kerak bo'lsa — type'larni alohida fayl ga ajratish.

### To'liq tushuntirish

**Circular dependency muammosi:**

A moduli B'ni import qiladi, B esa A'ni. Runtime'da modullar **synchronous** load bo'ladi:

```
Load A → A boshlanadi → A import B'ni so'raydi
Load B → B boshlanadi → B import A'ni so'raydi (lekin A hali to'liq load bo'lmagan!)
→ B A'ning incomplete versiyasini oladi
→ Runtime error yoki undefined export
```

**`import type` yechimi:**

`import type` JS output'da to'liq o'chiriladi:

```typescript
// user.ts
import type { Post } from "./post"; // ← JS'da o'chadi
export class User { posts: Post[] = []; }

// JS output:
// (import statement yo'q)
// export class User { ... }
```

Runtime'da `user.ts` `post.ts`'ni **import qilmaydi**. Circular yo'q.

**Cheklov:** Faqat type kerak bo'lganda ishlaydi. Agar runtime'da `Post` class'ni instantiation qilish kerak bo'lsa — `import type` ishlamaydi.

**Yechim — types.ts:**

Type'larni alohida fayl ga ajratish:

```typescript
// types.ts — faqat type'lar
export interface IUser { id: number; posts: IPost[] }
export interface IPost { id: number; author: IUser }

// user.ts — implementation
import type { IUser, IPost } from "./types";
export class User implements IUser { /* ... */ }

// post.ts — implementation
import type { IUser, IPost } from "./types";
export class Post implements IPost { /* ... */ }
```

### Kod misol

**Muammoli kod:**

```typescript
// user.ts
import { Post } from "./post";

export class User {
  id: number;
  posts: Post[] = [];
  
  constructor(id: number) { this.id = id; }
  
  addPost(content: string) {
    const post = new Post(content, this); // ❌ — runtime'da Post undefined bo'lishi mumkin
    this.posts.push(post);
  }
}

// post.ts
import { User } from "./user";

export class Post {
  content: string;
  author: User;
  
  constructor(content: string, author: User) {
    this.content = content;
    this.author = author;
  }
}
```

**Hal qilish — type-only:**

```typescript
// user.ts
import type { Post } from "./post"; // ← type-only

export class User {
  id: number;
  posts: Post[] = [];
  constructor(id: number) { this.id = id; }
}

// post.ts
import type { User } from "./user"; // ← type-only

export class Post {
  content: string;
  author: User;
  constructor(content: string, author: User) {
    this.content = content;
    this.author = author;
  }
}
```

**Bu yerda muammo:** `User.addPost` `new Post(...)` ishlatishi kerak edi. Type-only import bilan `Post` runtime'da yo'q. Yechim — dependency injection yoki types.ts pattern:

```typescript
// types.ts
export interface IUser { id: number; posts: IPost[] }
export interface IPost { content: string; author: IUser }

// user.ts
import type { IUser, IPost } from "./types";
import { Post } from "./post"; // ← runtime — lekin Post'da User import qilmaymiz endi

export class User implements IUser {
  id: number;
  posts: Post[] = [];
  constructor(id: number) { this.id = id; }
  addPost(content: string) {
    this.posts.push(new Post(content, this));
  }
}

// post.ts
import type { IUser, IPost } from "./types"; // ← type-only, circular yo'q

export class Post implements IPost {
  content: string;
  author: IUser; // ← interface, class emas
  constructor(content: string, author: IUser) {
    this.content = content;
    this.author = author;
  }
}
```

Endi `user.ts` `Post` ni runtime'da import qiladi, lekin `post.ts` faqat `IUser` interface'ni import qiladi (type-only) — circular yo'q.

### Edge Cases

- **CommonJS circular** — Node.js CJS da ham circular bor, lekin behavior har xil (partial exports). ESM da strict.
- **Function call vs class instantiation** — function call ko'pincha lazy (executed when called), class field reference esa eager. Class field'lar circular'ga ko'proq ta'sir qiladi.
- **`import type` runtime check** — agar developer xato bilan `import type` qilingan narsani runtime'da ishlatsa — compile error.
- **Re-export circular** — barrel'lar orqali circular ko'pincha yuzaga keladi. `import type { X } from "./index"` xavfsizroq.

### Follow-up savollar

1. **"Circular dependency'ni qanday topish mumkin?"** — `madge --circular`, `dpdm`, ESLint `import/no-cycle` rule. CI'da check qilish tavsiya.
2. **"CommonJS va ESM circular farqi?"** — CJS modullari synchronous evaluate qilinadi — circular'da partial (incomplete) exports qaytadi. ESM static graph — circular'ni live binding bilan hal qiladi.

</details>

---

### Savol 12: `.mts` va `.cts` fayl kengaytmalari — qachon kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`.mts` — **doim ESM** (compile `.mjs`). `.cts` — **doim CJS** (compile `.cjs`). `.ts` — `package.json "type"` yoki tsconfig'ga qarab. Aralash project'da fayl darajasida module format'ni belgilash uchun.

### To'liq tushuntirish

TypeScript 4.7 da Node.js'ning `.mjs` va `.cjs` kengaytmalariga moslab qo'shilgan. Module format'ni **fayl darajasida** aniq belgilash:

| Kengaytma | Module formati | JS output | Import syntax |
|-----------|---------------|-----------|---------------|
| `.ts` | `tsconfig.json` yoki `package.json "type"` | `.js` | `import` yoki `require` |
| `.mts` | **Doim ESM** | `.mjs` | `import` |
| `.cts` | **Doim CJS** | `.cjs` | `require()` |

Bu `module: "node16"` yoki `module: "nodenext"` bilan ishlaydi.

**Qachon kerak:**

1. **Aralash project** — Node.js'da bitta package ichida CJS va ESM fayllar
2. **Dual package** — library CJS va ESM ikkalasini ham qo'llab-quvvatlaydi
3. **Aniqlik** — fayl ning module format'i nomdan ko'rinadi
4. **Migration** — eski CJS code'dan ESM ga bosqichma-bosqich ko'chish

**Compiler behavior:**

- `.mts` faylda CJS syntax (`require()`, `module.exports`) → compile error
- `.cts` faylda ESM syntax (`import ... from`) → compile error
- Fayl kengaytmasi va actual syntax mos kelishi enforced

**Import resolution:**

```typescript
// fayl source: ./utils.mts
// Import: import "./utils.mjs"  // ← .mjs (compile natijasi)
// Compiler .mjs → .mts mapping qiladi
```

### Kod misol

```typescript
// utils.mts — DOIM ESM
export function greet(name: string): string {
  return `Hello, ${name}!`;
}

export const VERSION = "1.0.0";

// → Compile natija: utils.mjs (ESM format)
```

```typescript
// legacy.cts — DOIM CJS
export function oldGreet(name: string): string {
  return `Hello, ${name}!`;
}

// → Compile natija: legacy.cjs (CJS format)
```

```typescript
// main.mts — ESM faylda CJS import
import { oldGreet } from "./legacy.cjs"; // ✅ — interop bilan

console.log(oldGreet("Ali"));
```

```typescript
// legacy-main.cts — CJS faylda ESM import
const { greet } = require("./utils.mjs"); // ❌ — CJS require ESM'ni sync chaqira olmaydi

// To'g'ri usul — dynamic import
async function main() {
  const { greet } = await import("./utils.mjs");
  console.log(greet("Ali"));
}
```

**Library dual package:**

```json
// package.json
{
  "name": "my-lib",
  "main": "./dist/cjs/index.cjs",
  "module": "./dist/esm/index.mjs",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/esm/index.mjs",
      "require": "./dist/cjs/index.cjs"
    }
  }
}
```

```
src/
  index.mts          ← ESM source
  legacy.cts         ← CJS source (eski API)
  shared.ts          ← package.json "type" ga qarab
```

### Edge Cases

- **`.mts` faylda `__filename`/`__dirname`** — ESM'da yo'q. `import.meta.url` ishlatish kerak.
- **`.cts` faylda `import.meta`** — CJS'da yo'q. `__filename`/`__dirname` ishlatish.
- **Type declaration** — `.d.mts` va `.d.cts` ham mavjud. Library type'lar uchun.
- **Top-level await** — faqat `.mts` (ESM) da ishlaydi. `.cts` da yo'q.

### Follow-up savollar

1. **"`.ts` faylda module format'ni nima belgilaydi?"** — `package.json "type"` (`"module"` yoki `"commonjs"`) + tsconfig `module` option.
2. **"Library'da `.mts` va `.cts` bir-biriga reference qila oladimi?"** — Ha, lekin ehtiyot. ESM CJS'ni `import` qila oladi (default export). CJS ESM'ni faqat `await import()` bilan.

</details>

---

### Savol 13: ESM va CJS interop — Node.js'da ichki mexanizm [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

ESM va CJS — ikki turli module system. Node.js ESM CJS'ni `import` qila oladi (default export sifatida `module.exports`). CJS ESM'ni faqat `await import()` orqali — synchronous `require()` ishlamaydi, chunki ESM async evaluation graph'iga ega.

### To'liq tushuntirish

**Tubdan farq:**

| Xususiyat | CommonJS | ES Modules |
|-----------|----------|-----------|
| Loading | Synchronous `require()` | Async static graph + evaluation |
| Resolution | Runtime'da fayl-fayl | Parse-time'da static |
| Bindings | Copy of value (snapshot) | Live binding (export reference) |
| Top-level await | Yo'q | Bor |
| `this` top-level | `module.exports` | `undefined` |
| File extension | `.cjs` / `.js` (type=commonjs) | `.mjs` / `.js` (type=module) |
| Circular | Partial exports (snapshot) | Live binding bilan tiklanadi |

**ESM → CJS import (Node.js'da):**

Node.js ESM CJS modulni import qilganda, butun `module.exports` ni **default export** sifatida ko'rsatadi:

```typescript
// legacy.cjs
module.exports = { greet: (name: string) => `Hello, ${name}` };
module.exports.VERSION = "1.0";
```

```typescript
// modern.mts
// Default import — module.exports butunlay
import legacy from "./legacy.cjs";
console.log(legacy.greet("Ali"));     // "Hello, Ali"
console.log(legacy.VERSION);          // "1.0"

// Named import — Node.js syntax detection orqali ba'zi properties
import { greet, VERSION } from "./legacy.cjs";
// Node.js cjs-module-lexer bilan named exports'ni analyze qiladi
```

**Cheklov:** named import ishlashi `cjs-module-lexer` static analiziga bog'liq. Dynamic property assignment (`module.exports[key] = ...`) named import'da ko'rinmaydi.

**CJS → ESM import:**

```typescript
// legacy.cts
// ❌ require ESM'ni sync chaqira olmaydi
// const { greet } = require("./modern.mjs"); // → ERR_REQUIRE_ESM

// ✅ Dynamic import (async)
async function main() {
  const { greet } = await import("./modern.mjs");
  console.log(greet("Ali"));
}
```

Node.js 22.12+ (LTS) da `require()` ESM uchun **default yoqilgan** holatda ishlay boshladi (avval `--experimental-require-module` flag talab qilardi) — lekin faqat **top-level await yo'q** modullarda. Hali experimental, `--no-experimental-require-module` bilan o'chirsa bo'ladi.

**`esModuleInterop` — TypeScript level interop:**

`esModuleInterop: true` (`tsc --init` generatsiya qilgan config'da yoqilgan, lekin compiler default'i `false`) — TypeScript CJS namespace import'ni standart ESM default import sifatida ko'rsatadi:

```typescript
// `esModuleInterop: false`
import express from "express";
// ❌ Compile error — express '@types' `export =` ishlatadi,
//    default import allowSyntheticDefaultImports'siz ruxsat etilmaydi.
//    Namespace import (`import * as express`) kerak bo'lardi.

// `esModuleInterop: true`
import express from "express";
const app = express();  // ✅ — default import ishlaydi
```

Compiler `__importDefault` helper qo'shadi — `{ default: requireResult }` wrap qiladi. Side effect: namespace import semantikasi o'zgaradi.

**`isolatedModules` va interop:**

Bundler'lar (esbuild, SWC) `esModuleInterop` semantikasini partially qo'llab-quvvatlaydi. `verbatimModuleSyntax: true` bilan birga ishlatish predictable.

### Kod misol

```typescript
// === Dual package — ESM va CJS ikkalasini ham qo'llab-quvvatlash ===
// package.json
{
  "name": "@app/utils",
  "type": "module",
  "main": "./dist/cjs/index.cjs",
  "module": "./dist/esm/index.mjs",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/esm/index.mjs",
      "require": "./dist/cjs/index.cjs",
      "default": "./dist/esm/index.mjs"
    }
  }
}
```

```typescript
// src/index.mts (source)
export function formatPrice(value: number, currency: string): string {
  return `${value.toFixed(2)} ${currency}`;
}

export const TAX_RATES = { USD: 0.07, UZS: 0.12 } as const;
```

```typescript
// === Consumer code ===
// ESM consumer
import { formatPrice, TAX_RATES } from "@app/utils";
console.log(formatPrice(99.99, "USD"));  // "99.99 USD"

// CJS consumer (legacy code)
const { formatPrice, TAX_RATES } = require("@app/utils");
console.log(formatPrice(99.99, "USD"));  // "99.99 USD"
```

**Dual package hazard:**

Bir vaqtda CJS va ESM versiyalari load bo'lsa — ikki turli module instance. State, singleton, instanceof check'lar buziladi:

```typescript
// app.mjs (ESM consumer)
import { Logger } from "@app/utils";          // ESM version
const log = new Logger();

// some-legacy.cjs (CJS consumer of same package)
const { Logger: CjsLogger } = require("@app/utils");  // CJS version
console.log(log instanceof CjsLogger);  // ❌ false — different classes
```

**Yechim:** state'ni o'zgarmas (immutable) qilish yoki shared module pattern (`@app/utils/shared-state`).

### Edge Cases

- **Top-level await CJS'da:** `.mts` ESM'da ruxsat, `.cts` CJS'da yo'q. `require()` sync — async wait qila olmaydi.
- **`__filename` / `import.meta.url` ekvivalenti** — ESM'da: `const __filename = fileURLToPath(import.meta.url)`. CJS'da `__filename` global.
- **JSON import attributes** — ESM'da `import data from "./data.json" with { type: "json" }` (TS 5.3+, Node 22+). Stage 3 spec.
- **`require()` ESM'ni sync — Node 22.12+** — default yoqilgan (avval flag kerak edi), faqat top-level await yo'q modullarda. Hali experimental.
- **Webpack/Vite namespace** — Bundler ESM/CJS interop'ni o'zicha implement qiladi. Node.js'dan farq qilishi mumkin (default property unwrapping).

### Follow-up savollar

1. **"`__esModule` marker nima?"** — TypeScript/Babel transpiled CJS emit'i. `module.exports.__esModule = true` — bu fayl asli ESM bo'lganini bildiradi va transpiled consumer'ning `__importDefault` helper'i default export'ni to'g'ri unwrap qilishi uchun. Node.js'ning **native** interop'i bu marker'ni o'qimaydi — u named export'larni `cjs-module-lexer` orqali, default'ni esa butun `module.exports` sifatida aniqlaydi.
2. **"Library yozar ekan ESM-only chiqarish mumkinmi?"** — Ha, lekin ehtiyot. CJS consumer'lar `await import()` qilishi shart. Modern library lar (chalk v5, node-fetch v3) shu yo'lni tanlagan.

<details>
<summary><strong>Deep Dive</strong></summary>

**Node.js loader internals:**

ESM loader (`ModuleLoader`) ikki bosqichli:
1. **Resolution** — URL'ga aylantirish (`import.meta.resolve`)
2. **Loading** — `format` (`module`/`commonjs`/`json`/`wasm`), `source`, `shortCircuit`

```javascript
// Custom loader (Node 20.6+)
// loader.mjs
export async function resolve(specifier, context, nextResolve) {
  if (specifier.endsWith(".css")) {
    return {
      url: new URL(specifier, context.parentURL).href,
      format: "module",
      shortCircuit: true,
    };
  }
  return nextResolve(specifier);
}

export async function load(url, context, nextLoad) {
  if (url.endsWith(".css")) {
    return {
      format: "module",
      source: `export default ${JSON.stringify(await fs.readFile(new URL(url), "utf-8"))}`,
      shortCircuit: true,
    };
  }
  return nextLoad(url);
}
```

**ESM evaluation graph:**

ESM modullari uch fazada evaluate bo'ladi:
1. **Construction** — fayllar parse, dependency graph quriladi
2. **Instantiation** — exports/imports linkage (live binding)
3. **Evaluation** — top-level code ishlaydi (top-to-bottom, dependency-first)

Live binding shuni anglatadiki, import qilingan nom value snapshot emas, export'ning joriy holatiga reference. Lekin `const`/`let` binding'lar TDZ'ga bo'ysunadi — circular'da ular initialize bo'lishidan oldin **o'qib bo'lmaydi**.

Entry `a.mts` bo'lsa, ESM dependency-first evaluate qiladi: avval `b.mts` ishga tushadi. Top-level'da darhol `a`'ni o'qish TDZ xatosini beradi:

```typescript
// a.mts (entry)
import { getB } from "./b.mjs";
export const a = 1;
export function getA() { return a; }
console.log("a body, getB() =", getB());  // getB() — b hozir 2 (b.mts allaqachon evaluate bo'lgan)

// b.mts
import { getA } from "./a.mjs";
export const b = 2;
export function getB() { return b; }
// console.log(getA()); // ❌ — bu yerda chaqirilsa a hali TDZ'da → ReferenceError
```

Function orqali kechiktirilgan access (`getA`/`getB`) ishlaydi, chunki funksiya chaqirilganda ikkala module ham evaluate bo'lgan. Top-level'da to'g'ridan-to'g'ri `const` o'qish esa TDZ tufayli xato beradi.

**`cjs-module-lexer` analiz:**

Node.js CJS'dan named exports'ni quyidagicha topadi:
- `module.exports.greet = ...`
- `exports.greet = ...`
- `module.exports = { greet, VERSION }` (object literal)
- `Object.defineProperty(exports, "greet", ...)` — partial support

Dynamic assignment (`module.exports[key] = ...`) yoki conditional export — named import'da ko'rinmaydi.

</details>

</details>

---

## Output savollari

### Savol 14: `verbatimModuleSyntax` JS output [Middle+]

**Savol:** `verbatimModuleSyntax: true`, `module: "esnext"` da JS output nima bo'ladi?

```typescript
import type { User } from "./types";
import { type Product, OrderService } from "./services";
import { TAX_RATE } from "./config";
import "./polyfill";

export type { User };
export interface Config { debug: boolean }

const service = new OrderService();
const total = service.calculate() * (1 + TAX_RATE);
const user: User = { id: 1, name: "Ali" };
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```javascript
import { OrderService } from "./services";
import { TAX_RATE } from "./config";
import "./polyfill";

const service = new OrderService();
const total = service.calculate() * (1 + TAX_RATE);
const user = { id: 1, name: "Ali" };
```

### To'liq tushuntirish

**O'chirilganlar:**
- `import type { User }` — type-only import (to'liq o'chadi)
- `import { type Product }` — inline type specifier (Product o'chadi, OrderService qoladi)
- `export type { User }` — type-only re-export
- `export interface Config` — interface (pure type construct)
- `: User` annotation — type annotation

**Saqlanganlar:**
- `import { OrderService }` — value import
- `import { TAX_RATE }` — value import
- `import "./polyfill"` — side-effect import (value)
- `new OrderService()` — runtime value
- Variable declaration'lar

**Behavior detail:**

```typescript
// Source
import { type Product, OrderService } from "./services";

// Output
import { OrderService } from "./services";
// type Product — inline modifier bilan o'chadi
```

`verbatimModuleSyntax` bilan compiler hech qanday heuristic qo'llamaydi. Aynan source'da yozilgan yo'sin emit qiladi:
- `type` modifier bor → o'chadi
- `type` modifier yo'q → JS'da qoladi

### Edge Cases

- **`import "./side-effect"` bilan `verbatimModuleSyntax`** — side-effect import value sifatida saqlanadi.
- **Re-export type-only emas** — `export { OrderService } from "./services"` — saqlanadi (OrderService value).
- **Compile target ta'siri** — `target: "es5"` da `const` `var` ga aylanishi mumkin, lekin import structure o'zgarmaydi.

</details>

---

### Savol 15: `import type` bilan runtime ishlatish [Junior+]

**Savol:** Quyidagi kod compile bo'ladimi?

```typescript
import type { UserService } from "./services";

const service = new UserService();
let svc: UserService;
function handle(s: UserService) {}
type Svc = InstanceType<typeof UserService>;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
const service = new UserService();           // ❌ Error
let svc: UserService;                         // ✅ OK (type annotation)
function handle(s: UserService) {}            // ✅ OK (parameter type)
type Svc = InstanceType<typeof UserService>;  // ❌ Error (typeof value position)
```

### To'liq tushuntirish

`import type` faqat **type position**'da ishlatish mumkin:
- ✅ Type annotation (`: UserService`)
- ✅ Parameter type (`function f(x: UserService)`)
- ✅ Return type (`function f(): UserService`)
- ✅ Generic argument (`Array<UserService>`)
- ✅ `implements` clause (`class AdminService implements UserService` — `implements` type position)

Lekin **value position**'da ishlatib bo'lmaydi:
- ❌ `new UserService()` — instantiation (runtime call)
- ❌ `class X extends UserService` — `extends` clause expression runtime'da value sifatida evaluate bo'ladi (`implements`'dan farqli)
- ❌ `typeof UserService` — value reference (`typeof` ning argumenti — value)
- ❌ `UserService.staticMethod()` — static method call
- ❌ Variable assignment (`const x = UserService`)

**Error message:**

```
'UserService' cannot be used as a value because it was imported using 'import type'.
```

**`typeof` paradox:** `typeof X` — type expression, lekin argument `X` value position'da. Compiler `typeof` argument'ni value sifatida talab qiladi. Type-only import bu kontekstda ishlamaydi.

### Edge Cases

- **`typeof` value position vs type position** — Aralashtirib yuborilishi mumkin. `type T = typeof UserService` — `T` type, lekin `typeof UserService` ichida `UserService` value.
- **Class type vs instance type** — `import type { UserService }` faqat **instance type** ni beradi. Class itself (constructor) value.
- **`InstanceType<typeof X>` workaround** — Ham value import kerak. Type-only bilan ishlamaydi.

</details>

---

### Savol 16: Module augmentation tartibi [Middle+]

**Savol:** Quyidagi `.d.ts` augmentation `req.userId` ni modern `@types/express` request'iga qo'sha olmaydi. Nima uchun? `auth` ichidagi satrlar error beradimi?

```typescript
// types/express.d.ts (faqat shu fayl)
declare module "express" {
  interface Request {
    userId: string;
  }
}

// middleware.ts
import { Request } from "express";

function auth(req: Request) {
  req.userId = "user-1";  // A
  req.body;                // B
  req.params;              // C
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Hech bir satr error bermaydi (A, B, C — hammasi compile bo'ladi). Bug **jim**: augmentation noto'g'ri module'ga yo'naltirilgan. Modern `@types/express`'da `Request` interface `express-serve-static-core`'da, `express` emas. `declare module "express" { interface Request }` — boshqa, bog'liqsiz `Request` yaratadi → haqiqiy request type'iga ta'sir qilmaydi.

```
A — ✅ (lekin userId real Request'ga emas, soxta Request'ga tegishli)
B — ✅ body mavjud (express-serve-static-core'dagi original Request'dan)
C — ✅ params mavjud (original Request'dan)
```

### To'liq tushuntirish

**Avval bir noto'g'ri taxminni yo'qotamiz:** ".d.ts script'da `declare module "express"` Express'ni butunlay almashtiradi" — bu noto'g'ri. Bir xil nomli ambient `declare module` declaration'lar **merge bo'ladi** (`@types/express`'dagi `express` module bilan ham). Shuning uchun `body`, `params` yo'qolmaydi — error bermaydi.

**Asl bug — noto'g'ri augmentation target:**

Modern `@types/express`'da `Request` interface `express-serve-static-core` package'da declare qilingan; `express` uni faqat re-export qiladi. `declare module "express" { interface Request {...} }` esa `express` module scope'ida **yangi, alohida** `Request` interface ochadi — u `express-serve-static-core`'dagi haqiqiy `Request` bilan merge bo'lmaydi. Natija: `userId` typing haqiqiy request object'iga yetib bormaydi.

**Fix:**

```typescript
// types/express.d.ts
import "express"; // ← faylni MODULE qiladi (.ts module faylda augmentation uchun shart)

declare module "express-serve-static-core" {
  interface Request {
    userId: string;
  }
}
```

Endi:
- `declare module "express-serve-static-core"` — haqiqiy `Request` interface'iga merge
- `Request` interface: original properties (`body`, `params`, ...) + `userId`

**Script vs module nuance:** `.d.ts` faylda `declare module` default ambient declaration sifatida ishlaydi (va merge qiladi). Lekin `.ts` (yoki module bo'lishi kutilgan) faylda `declare module` faqat fayl module bo'lganda (`import`/`export` bor) augmentation bo'ladi — aks holda ambient declaration. `import "express"` ikkala holatda ham xavfsiz: faylni module qiladi va niyatni aniq ko'rsatadi.

### Edge Cases

- **Eski `@types/express`** — Eski versiyalarda `Request` to'g'ridan-to'g'ri `express` module'da bo'lgan; o'shanda `declare module "express"` to'g'ri ishlardi. Versiyaga bog'liq.
- **`export {}` bilan ham module bo'ladi** — `.ts` faylni module qilishning eng yengil usuli. `import "express"` esa qo'shimcha ravishda Express'ni resolve qiladi.
- **`tsconfig.json` `include`** — augmentation fayli compile'ga kirishi kerak, aks holda hech qanday ta'sir bermaydi.

</details>

---

### Savol 17: Module resolution conflict [Middle+]

**Savol:** Quyidagi konfiguratsiyada `import { User } from "./user"` qaysi faylni topadi?

```
src/
  user.ts
  user.tsx
  user/
    index.ts
```

```json
{ "compilerOptions": { "moduleResolution": "node" } }
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`./user.ts` ni topadi. Resolution tartibi: `.ts` → `.tsx` → `.d.ts` → `./user/index.ts`. Faqat birinchi mos faylni oladi.

### To'liq tushuntirish

`moduleResolution: "node"` da relative import resolution tartibi:

```
import { User } from "./user"

1. ./user.ts          ← BU TOPILDI ✅ — qolganlari tekshirilmaydi
2. ./user.tsx
3. ./user.d.ts
4. ./user/index.ts
5. ./user/index.tsx
6. ./user/index.d.ts
7. ./user.js (allowJs bo'lsa)
8. ./user/index.js
```

`.ts` birinchi tekshiriladi va topiladi. `.tsx` va `./user/index.ts` ignore qilinadi.

**Conflict — folder va file bir xil nom:**

```
src/
  user.ts       ← bu olinadi
  user/         ← bu ignore qilinadi
    index.ts
    profile.ts
```

`import { User } from "./user"` — `user.ts` ni oladi. `user/profile.ts` ga kirish uchun `import "./user/profile"` yozish kerak.

**`moduleResolution: "node16"` da:**

Extension majburiy:

```typescript
import { User } from "./user";        // ❌ Error
import { User } from "./user.js";      // ✅ — .ts ga map qilinadi
import { User } from "./user/index.js"; // ✅ — folder
```

### Edge Cases

- **`paths` mapping bilan** — `paths` resolution `node`/`bundler` resolution'dan **avval** ishga tushadi. Alias topilsa, o'sha priority.
- **`exports` field external library** — `node16`/`bundler` da `package.json "exports"` qaramaganda eng birinchi mos condition.
- **Case sensitivity** — macOS/Windows case-insensitive, Linux sensitive. `forceConsistentCasingInFileNames: true` cross-platform safety.

</details>

---

### Savol 18: `paths` runtime [Middle]

**Savol:** Quyidagi kod compile bo'ladi va run bo'ladimi?

```json
// tsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }
```

```typescript
// src/app.ts
import { Button } from "@/components/Button";

console.log(Button);
```

Node.js da `node dist/app.js` ishlatilsa nima bo'ladi?

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Compile o'tadi (type-check ✅), lekin Node.js runtime'da fail bo'ladi — `Cannot find module '@/components/Button'`.

### To'liq tushuntirish

**Compile-time:**

TypeScript `paths` ni faqat type-checking uchun ishlatadi. `@/components/Button` `src/components/Button` ga map qilinadi va compiler topib type-check qiladi.

**Compile output:**

```javascript
// dist/app.js (TypeScript compile natijasi)
import { Button } from "@/components/Button";  // ← O'ZGARMAGAN
console.log(Button);
```

TypeScript path'larni replace **qilmaydi**. JS faylda alias o'zgarmagan holatda qoladi.

**Runtime (Node.js):**

```bash
$ node dist/app.js
Error: Cannot find module '@/components/Button'
```

Node.js `@/components/Button`'ni topa olmaydi — `@/` Node.js'ga ma'lum bo'lgan resolution yo'li emas.

**Yechimlar:**

1. **`tsconfig-paths` runtime patch:**

```bash
npm install --save-dev tsconfig-paths
node -r tsconfig-paths/register dist/app.js
```

2. **`tsc-alias` build vaqtida path'larni replace qilish:**

```bash
npm install --save-dev tsc-alias
tsc && tsc-alias
```

3. **Bundler (Vite, esbuild)** — production build'da bundler resolve qiladi.

4. **`tsx`/`ts-node`** development da — `tsconfig-paths` integratsiyalashgan.

### Edge Cases

- **Vite dev server** — `vite.config.ts` da `resolve.alias` shart. Vite alias'ni runtime'da resolve qiladi.
- **Jest** — `moduleNameMapper` orqali alias resolve.
- **Webpack** — `resolve.alias`.
- **Babel + Webpack** — `@babel/plugin-module-resolver` ham alias resolve qiladi.

</details>

---

### Savol 19: `isolatedModules` const enum [Middle]

**Savol:** Quyidagi kod `isolatedModules: true` bilan compile bo'ladimi?

```typescript
// constants.ts
export const enum Color {
  Red = "red",
  Green = "green",
  Blue = "blue",
}

// app.ts
import { Color } from "./constants";
const c = Color.Red;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`tsc` bilan compile bo'ladi (value inline qilinadi), lekin esbuild/SWC/Babel bilan transpile qilinsa runtime'da `Color` topilmaydi. `preserveConstEnums: true` bilan tuzatish mumkin.

### To'liq tushuntirish

`const enum` value'lar compile-time'da **inline** replace bo'ladi. Bu boshqa fayldan type information talab qiladi — fayl-bo'yicha transpilation imkonsiz.

**`tsc` behavior (project-level compile):**

```javascript
// app.ts → app.js
const c = "red";  // ← Color.Red inline replace qilingan
```

`tsc` butun project'ni biladi → const enum value'ni o'qiy oladi.

**esbuild/SWC/Babel behavior (file-level transpilation):**

```javascript
// app.ts → app.js
const c = Color.Red;  // ← `Color` runtime'da yo'q — runtime error!
```

Transpiler'lar `constants.ts`'dagi `const enum` ni o'qimaydi → `Color` runtime'da mavjud emas → error.

**`isolatedModules: true` va `const enum`:**

Project ichidagi (non-ambient) const enum'ni `import` qilganda `tsc` xato bermaydi — value'ni inline qiladi. Lekin **ambient** const enum'ga (`.d.ts` yoki `declare const enum`) murojaat qilinsa, `isolatedModules` bilan TS2748 xatosi chiqadi:

```
error TS2748: Cannot access ambient const enums when '--isolatedModules' flag is provided.
```

Asosiy xavf shu: kod `tsc` bilan compile bo'lsa-da, esbuild/SWC bilan transpile qilinganda jim runtime error beradi.

**Yechimlar:**

1. **Oddiy enum:**
```typescript
export enum Color { Red = "red", Green = "green", Blue = "blue" }
// Runtime object yaratiladi — har joyda ishlaydi
```

2. **Union type (yaxshiroq):**
```typescript
export type Color = "red" | "green" | "blue";
export const Color = { Red: "red", Green: "green", Blue: "blue" } as const;
```

3. **`preserveConstEnums: true`** (const enum'ni runtime object sifatida emit qiladi):
```json
{ "compilerOptions": { "preserveConstEnums": true, "isolatedModules": true } }
```
Const enum project bo'ylab shu option bilan compile qilingan bo'lsa, `Color` runtime object sifatida saqlanadi → transpiler `Color.Red` ni topadi.

### Edge Cases

- **`preserveConstEnums`** — const enum'ni inline qilish o'rniga runtime object emit qiladi (TS 1.4'dan beri mavjud). Inline optimization yo'qoladi, lekin transpiler bilan moslashadi.
- **`esModuleInterop`** ta'siri yo'q — bu boshqa muammo.
- **Numeric vs string const enum** — Ikkalasi ham bir xil cheklov. Lekin numeric'da inline'lash optimization'i ko'proq foyda berardi (string'da kam).

</details>

---

## Coding challenges

### Savol 20: `process.env` type-safe [Middle+]

**Savol:** `process.env`'ni type-safe qiling. Faqat declared variable'larga ruxsat, runtime validation ham qo'shing.

```typescript
// process.env.DATABASE_URL → string (undefined emas)
// process.env.NODE_ENV → "development" | "production" | "test"
// process.env.RANDOM_VAR → ❌ compile error
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`declare namespace NodeJS { interface ProcessEnv { ... } }` augmentation + runtime validation function.

### Kod misol

```typescript
// env.d.ts
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: "development" | "production" | "test";
    DATABASE_URL: string;
    API_KEY: string;
    PORT: string;
    LOG_LEVEL: "debug" | "info" | "warn" | "error";
    REDIS_URL?: string; // optional
  }
}
```

```typescript
// env.ts — runtime validation
const requiredEnvVars = [
  "DATABASE_URL",
  "API_KEY",
  "PORT",
  "LOG_LEVEL",
  "NODE_ENV",
] as const satisfies ReadonlyArray<keyof NodeJS.ProcessEnv>;

function validateEnv(): void {
  const missing: string[] = [];
  for (const key of requiredEnvVars) {
    if (!process.env[key]) {
      missing.push(key);
    }
  }
  if (missing.length > 0) {
    throw new Error(`Missing required env vars: ${missing.join(", ")}`);
  }
}

// Type-safe getter — runtime check
function getEnv<K extends keyof NodeJS.ProcessEnv>(
  key: K,
): NonNullable<NodeJS.ProcessEnv[K]> {
  const value = process.env[key];
  if (!value) {
    throw new Error(`Missing env var: ${key}`);
  }
  return value as NonNullable<NodeJS.ProcessEnv[K]>;
}

// Usage
validateEnv();
const dbUrl = getEnv("DATABASE_URL");       // string ✅
const env = getEnv("NODE_ENV");              // "development" | "production" | "test" ✅
// getEnv("RANDOM_VAR");                     // ❌ — declared emas
```

**Zod alternative — runtime + types from schema:**

```typescript
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  DATABASE_URL: z.url(),
  API_KEY: z.string().min(1),
  PORT: z.string().regex(/^\d+$/),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]),
});

export const env = EnvSchema.parse(process.env); // runtime validated
export type Env = z.infer<typeof EnvSchema>;     // type from schema
```

### Edge Cases

- **Compile-time vs runtime** — Type declaration faqat kompilatorga signal. Runtime'da `process.env.X` hali `undefined` bo'lishi mumkin.
- **`.env` file** — Node.js o'zi `.env` yuklamaydi. `dotenv` paketi yoki Node.js 20+ `--env-file` flag.
- **Test environment** — Test'larda env variable'larni set qilish yoki mock.

### Follow-up savollar

1. **"`declare module "process"` ishlatish mumkinmi?"** — Yo'q. `process` ning interface'i `NodeJS.Process`. Namespace augmentation yagona usul.
2. **"`env` zod schema bilan runtime validation qachon?"** — App start vaqtida, har request'da emas. Failure → app boot fail.

</details>

---

### Savol 21: Type-only barrel [Middle]

**Savol:** Quyidagi type'larni barrel export qiling. `verbatimModuleSyntax: true` da ishlashi kerak. Tree shaking ta'sirsiz.

```typescript
// models/user.ts
export interface User { id: number; name: string }
export type UserRole = "admin" | "user";

// models/product.ts
export interface Product { id: number; price: number }
export type Currency = "USD" | "UZS";

// models/order.ts
export interface Order { id: number; items: Product[]; status: OrderStatus }
export type OrderStatus = "pending" | "paid" | "shipped";
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`export type { ... } from "..."` — type-only re-export. JS output'da hech narsa.

### Kod misol

```typescript
// models/index.ts
export type { User, UserRole } from "./user";
export type { Product, Currency } from "./product";
export type { Order, OrderStatus } from "./order";
```

**Foydalanish:**

```typescript
// app.ts
import type { User, Order, OrderStatus } from "./models";

function processOrder(user: User, order: Order, status: OrderStatus): void {
  // ...
}
```

**JS output:**

```javascript
// models/index.js — bo'sh fayl (faqat type)
// app.js — barcha type import'lar olib tashlangan
function processOrder(user, order, status) { /* ... */ }
```

**Aralash barrel (type + value):**

```typescript
// models/index.ts
// Types
export type { User, UserRole } from "./user";
export type { Product, Currency } from "./product";
export type { Order, OrderStatus } from "./order";

// Services (runtime)
export { UserService } from "./user-service";
export { ProductService } from "./product-service";
export { OrderService } from "./order-service";
```

### Edge Cases

- **`export * from`** — Re-export hammasi: `export * from "./user"`. Lekin name collision xavfi va `verbatimModuleSyntax` bilan type/value ajratish qiyin.
- **Re-export rename** — `export type { User as Customer } from "./user"`.
- **Namespace re-export** — `export * as Models from "./models"` — barcha exports'ni namespace ostida.

### Follow-up savollar

1. **"Type-only barrel bilan circular yo'q chunki?"** — Type-only import JS'da o'chadi → runtime'da modullar bir-birini chaqirmaydi. Circular yo'q.
2. **"Barrel sekinlashtiradi degan da'vo to'g'rimi?"** — Compile time'da — ha (har import butun barrel resolve qiladi). Runtime'da — type-only bo'lsa, ta'siri yo'q.

</details>

---

### Savol 22: Vite + React tsconfig konfiguratsiya [Middle+]

**Savol:** Vite + React + TypeScript loyihasi uchun to'liq `tsconfig.json` yozing. Talablar: `verbatimModuleSyntax`, `isolatedModules`, bundler resolution, `@/` alias, force module detection, strict mode.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bundler resolution + verbatim + isolated + alias + strict.

### Kod misol

```json
{
  "compilerOptions": {
    // Module
    "module": "esnext",
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true,
    "isolatedModules": true,

    // Target
    "target": "es2022",
    "lib": ["es2022", "dom", "dom.iterable"],
    "jsx": "react-jsx",

    // Strict
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,

    // Paths
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },

    // Build
    "skipLibCheck": true,
    "allowImportingTsExtensions": false,
    "noEmit": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src", "vite.config.ts"],
  "exclude": ["node_modules", "dist"]
}
```

**Vite konfiguratsiyasi (alias runtime resolution):**

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "node:path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

**Tushuntirish:**

| Option | Sabab |
|--------|-------|
| `module: "esnext"` | Vite ESM output beradi |
| `moduleResolution: "bundler"` | Extension-less, `exports` support |
| `moduleDetection: "force"` | Barcha fayllar module — global scope toza |
| `verbatimModuleSyntax: true` | Predictable emit, esbuild/SWC bilan moslashish |
| `isolatedModules: true` | Vite (esbuild) fayl-bo'yicha transpile |
| `target: "es2022"` | Modern browser, top-level await |
| `jsx: "react-jsx"` | New JSX transform (React 17+) |
| `strict: true` | strictNullChecks, noImplicitAny va boshqa strict flags |
| `noUncheckedIndexedAccess` | `arr[0]` → `T | undefined` |
| `exactOptionalPropertyTypes` | `x?: string` ga `undefined` assign error |
| `paths` | Type-checking uchun (Vite alohida resolve) |
| `noEmit: true` | Vite bundle qiladi — `tsc` emit kerak emas |
| `skipLibCheck: true` | `.d.ts` library'lar tekshirilmaydi — tezroq |

### Edge Cases

- **`allowImportingTsExtensions`** — TS 5.0+. `import "./file.ts"` (extension bilan TS fayl). Vite default rad qiladi.
- **`moduleSuffixes`** — `.ios.ts`, `.android.ts` kabi platform-specific. React Native uchun.
- **`incremental` build** — Monorepo'da `incremental: true` va `composite: true` project references bilan.

### Follow-up savollar

1. **"`noEmit: true` bilan tsc nima ish qiladi?"** — Faqat type-check. Vite o'zi bundling qiladi (esbuild orqali).
2. **"`composite: true` qachon kerak?"** — Monorepo'da TypeScript project references. Cross-package type-check va incremental build.

</details>

---

## Bug fix savollari

### Savol 23: Module augmentation override [Middle+]

**Savol:** Quyidagi augmentation modern `@types/express`'da `req.userId` ni haqiqiy request type'iga qo'sha olmaydi (jim ishlamaydi). Toping va tuzating:

```typescript
// types/custom.d.ts
declare module "express" {
  interface Request {
    userId: string;
  }
}

// middleware.ts
import { Request } from "express";

function auth(req: Request) {
  req.userId = "user-1";   // augmentation niyati — lekin real Request'ga yetmaydi
  console.log(req.body);   // body bor (original Request'dan)
  console.log(req.params); // params bor (original Request'dan)
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bug — noto'g'ri augmentation target. Modern `@types/express`'da `Request` interface `express-serve-static-core`'da, `express`'da emas. `declare module "express" { interface Request }` — alohida, bog'liqsiz `Request` yaratadi → haqiqiy request type'iga merge bo'lmaydi. To'g'ri module — `express-serve-static-core`.

### To'liq tushuntirish

**Noto'g'ri taxmin — `body`/`params` yo'qolmaydi:** Bir xil nomli ambient `declare module` declaration'lar merge bo'ladi, almashtirmaydi. Shuning uchun `req.body`, `req.params` saqlanadi va error bermaydi. Bug butunlay boshqa joyda.

**Asl bug — wrong augmentation target:**

Modern Express `@types`'da `Request` interface aslida `express-serve-static-core` package'da declare qilingan; `express` uni re-export qiladi. `declare module "express"` ichidagi `Request` esa `express` scope'ida yangi, alohida interface ochadi — `express-serve-static-core`'dagi haqiqiy `Request` bilan merge bo'lmaydi. Natija: `userId` typing haqiqiy request'ga yetib bormaydi.

**Fix:**

```typescript
// types/custom.d.ts
import "express"; // ← faylni MODULE qiladi va Express resolve qiladi

declare module "express-serve-static-core" {
  interface Request {
    userId: string;
  }
}
```

Endi:
- `import "express"` faylni module qiladi
- `declare module "express-serve-static-core"` — augmentation
- Express'ning `Request` (express-serve-static-core dan keladi) ga `userId` qo'shiladi
- `body`, `params`, `query`, `cookies` va boshqa original properties saqlanadi

**Verification:**

```typescript
// middleware.ts
import { Request, Response, NextFunction } from "express";

function auth(req: Request, res: Response, next: NextFunction) {
  req.userId = "user-1";     // ✅ — augmented
  console.log(req.body);     // ✅ — original
  console.log(req.params);   // ✅ — original
  console.log(req.query);    // ✅ — original
  next();
}
```

### Edge Cases

- **Eski `@types/express` versiya** — Eski versiyalarda `Request` `express` module'da bo'lgan. Augmentation `declare module "express"` ishlardi.
- **`tsconfig.json include`** — `types/custom.d.ts` `include` da bo'lishi kerak.
- **Type acquisition** — VSCode'da automatic type acquisition fayl ko'rmaslik mumkin. `tsc --listFiles` bilan tekshirish.

</details>

---

### Savol 24: Circular dependency bug [Middle+]

**Savol:** Quyidagi kodda circular dependency bor va `post.ts` `User`'ni runtime'da (`extends`) ishlatadi. Modullar yuklanish tartibida `User` hali aniqlanmagan paytda `post.ts` evaluate bo'ladi. Toping va tuzating:

```typescript
// user.ts
import { Post } from "./post";

export class User {
  posts: Post[] = [];
  addPost(content: string) {
    this.posts.push(new Post(content, this));
  }
}

// post.ts
import { User } from "./user";

// User runtime'da kerak — extends value reference
export class Post extends User {
  constructor(public content: string, public author: User) {
    super();
  }
}

// main.ts
import { User } from "./user";
const u = new User();
u.addPost("Hello");
// ❌ Runtime: ReferenceError: Cannot access 'User' before initialization
//   (post.ts evaluate bo'lganda user.ts hali to'liq yuklanmagan)
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`user.ts` `Post`'ni runtime'da (`new Post`), `post.ts` esa `User`'ni runtime'da (`extends User`) ishlatadi — ikki tomonlama runtime cycle. `extends` value reference bo'lgani uchun `import type` uni hal qila olmaydi. Yechim — `extends` ni olib tashlash (composition) yoki umumiy base'ni uchinchi modulga ajratish.

### To'liq tushuntirish

**Muammo:**
- `user.ts` `Post`'ni instantiate qiladi → runtime'da kerak
- `post.ts` `Post extends User` qiladi → `User` class-evaluation paytida runtime'da kerak

`main.ts` `user.ts` ni yuklaydi → `user.ts` `post.ts` ni so'raydi → `post.ts` `extends User` uchun `User`'ni o'qiydi, lekin `user.ts` hali to'liq evaluate bo'lmagan → `User` binding TDZ'da → `ReferenceError`.

**Yechim 1 — `extends` ni olib tashlash (composition):**

`Post`'ga `User`'ni runtime'da kerak qiladigan `extends` yo'q. `User` faqat type sifatida → `import type`:

```typescript
// user.ts
import { Post } from "./post"; // ← runtime import (kerak)

export class User {
  posts: Post[] = [];
  addPost(content: string) {
    this.posts.push(new Post(content, this));
  }
}

// post.ts
import type { User } from "./user"; // ← type-only, runtime'da yo'q

export class Post {
  constructor(public content: string, public author: User) {}
}
```

Endi runtime dependency faqat `user → post` bir tomonga. `post.ts` `User`'ni runtime'da o'qimaydi → cycle uziladi.

**Yechim 2 — Types ajratish:**

```typescript
// types.ts
export interface IUser { posts: IPost[] }
export interface IPost { content: string; author: IUser }

// user.ts
import { Post } from "./post";
import type { IUser, IPost } from "./types";

export class User implements IUser {
  posts: Post[] = [];
  addPost(content: string) {
    this.posts.push(new Post(content, this));
  }
}

// post.ts
import type { IUser, IPost } from "./types";

export class Post implements IPost {
  constructor(public content: string, public author: IUser) {}
}
```

**Yechim 3 — Dependency injection:**

```typescript
// post.ts (User'ga reference yo'q)
export class Post {
  constructor(public content: string, public authorId: number) {}
}

// user.ts
import { Post } from "./post";

export class User {
  constructor(public id: number) {}
  posts: Post[] = [];
  addPost(content: string) {
    this.posts.push(new Post(content, this.id));
  }
}
```

### Edge Cases

- **ESM strict semantics** — ESM'da circular `live binding` bilan ishlaydi, lekin class declaration TDZ (temporal dead zone) tufayli muammoli.
- **CJS partial exports** — CJS'da circular partial exports bilan ishlaydi. `Post` undefined emas, lekin incomplete.
- **`madge` bilan tekshirish** — `npx madge --circular src/` circular dependencies'ni topadi.

### Follow-up savollar

1. **"`import type` har doim circular'ni hal qiladimi?"** — Yo'q. Faqat type-only kerak bo'lgan holatlarda. Runtime dependency bo'lsa — refactor kerak.
2. **"Bidirectional class reference qanday best-practice?"** — Ko'p hollarda design smell. ID-based reference yoki repository pattern afzal.

</details>

---

## Xulosa

- **`import type`** — JS'da o'chiriladi. Bundle size, circular dependencies, predictable emit
- **`verbatimModuleSyntax` (TS 5.0+)** — `import type` majburiy, predictable emit, bundler-friendly
- **Module resolution:** `"bundler"` (Vite/Webpack), `"node16"`/`"nodenext"` (Node.js ESM), `"node"` (legacy)
- **`isolatedModules`** — fayl-bo'yicha transpile, `const enum` cheklov, `export type` majburiy
- **Module augmentation** — `import` + `declare module` (import yo'q = ambient declaration, override!)
- **`declare global`** — `export {}` bilan module qilish, global scope augmentation
- **Path aliases** — `paths`/`baseUrl` faqat type-check, runtime'da bundler kerak
- **Barrel exports** — type-only xavfsiz, value barrel tree shaking ehtiyot
- **Dynamic imports** — `import()` Promise, code splitting, type-safe
- **Circular dependencies** — `import type` hal qiladi (faqat type bo'lsa), types ajratish (runtime bilan)
- **`.mts`/`.cts`** — Fayl darajasida ESM/CJS belgilash
