# Bo'lim 22: tsconfig.json Mastery

> `tsconfig.json` — TypeScript project'ining markaziy configuration fayli. U compiler'ga qanday type check qilish, qaysi fayllarni compile qilish, qaysi target'ga emit qilish, va qanday module system ishlatishni aytadi. Har bir muhim compiler option — nazariya, amaliy misollar, va real-world best practices bilan.

---

## Mundarija

- [tsconfig.json Asoslari](#tsconfigjson-asoslari)
- [Strict Family](#strict-family)
- [Target va Lib](#target-va-lib)
- [Module Options](#module-options)
- [Paths va Directories](#paths-va-directories)
- [Declaration va Source Maps](#declaration-va-source-maps)
- [Checking Options](#checking-options)
- [Emit Options](#emit-options)
- [Interop Options](#interop-options)
- [include, exclude, files](#include-exclude-files)
- [extends — tsconfig Inheritance](#extends--tsconfig-inheritance)
- [Project References](#project-references)
- [ESLint + TypeScript](#eslint--typescript)
- [Best Practices Per Project Type](#best-practices-per-project-type)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## tsconfig.json Asoslari

### Nazariya

`tsconfig.json` — TypeScript project'ining root directory'sida joylashgan configuration fayli. `tsc` shu faylni o'qib, compiler sozlamalari, qaysi fayllar compile qilinishi, qaysi target'ga emit qilish, va qanday module system ishlatilishini aniqlaydi.

**NIMA UCHUN kerak:** TypeScript turli muhitlarda (browser, Node.js, Deno, bundler) turlicha compile bo'lishi kerak. Har project o'z `target`, `module`, `lib` qiymatlariga muhtoj. `tsconfig.json` bularni declarative tarzda saqlaydi va `tsc` jarayoni reproducible bo'ladi.

```json
{
  "compilerOptions": { },
  "include": [],
  "exclude": [],
  "files": [],
  "extends": "",
  "references": []
}
```

- **`compilerOptions`** — compiler sozlamalari (eng katta qism)
- **`include`** — qaysi fayllar compile qilinadi (glob pattern)
- **`exclude`** — qaysilarni chiqarib tashlash
- **`files`** — aniq fayllar ro'yxati (glob emas)
- **`extends`** — boshqa tsconfig'dan meros olish
- **`references`** — project references (monorepo)

`tsc --init` — standart tsconfig yaratadi. `tsc --showConfig` — effective config ni ko'rsatadi (`extends` resolved, default'lar o'rnatilgan).

<details>
<summary><strong>Under the Hood</strong></summary>

**Config resolution algoritmi:**

1. `tsc` joriy directory'dan boshlab yuqoriga `tsconfig.json` qidiradi (yoki `--project` flag bilan aniq path)
2. `extends` zanjirini topib chiqadi (recursive merge) — base config'lar bottom-up qo'llanadi
3. `compilerOptions` extends bo'yicha override qilinadi (oxirgi g'olib)
4. `include`/`exclude`/`files` — extends bo'lsa **override** qiladi (merge emas)
5. Default qiymatlar to'ldiriladi (`module` default `target` ga bog'liq, `moduleResolution` default `module` ga bog'liq)
6. `files` ro'yxati + `include` glob'lar resolve qilinadi, `exclude` filter qiladi
7. Import graph quriladi — `import` orqali ulangan har bir fayl compile setiga qo'shiladi (`exclude` bunga ta'sir qilmaydi)

```
tsconfig.json o'qildi
        ↓
extends zanjir resolve (recursive)
        ↓
compilerOptions merge (override)
        ↓
default'lar to'ldirildi
        ↓
files + include - exclude → root fayllar
        ↓
import graph traverse → barcha kerakli fayllar
        ↓
type-check + emit
```

**Default behavior nuance'lari:**

- `module` default: `target === "ES3" | "ES5"` → `"CommonJS"`, aks holda `"ES6"`
- `moduleResolution` default: `module === "CommonJS"` → `"Node10"`, `module === "Node16"/"NodeNext"` → mos kelgan
- `target` default (hech qaysi `target` berilmasa): `"ES3"` — bu implicit default TS 5.x gacha saqlangan (TS 5.0 da `ES3` deprecated qilindi, lekin default qiymat o'zgarmadi). `tsc --init` esa fayl shablonida `target` ni boshqacha yozadi — bu shablon qiymati, compiler'ning implicit default'i emas
- `strict: false` — har strict family flag alohida-alohida default `false`

</details>

---

## Strict Family

### Nazariya

`strict: true` — **strict family** flag'larini birdan yoqadigan meta-flag. **NIMA UCHUN:** strict bo'lmagan TypeScript JavaScript ustida soft type layer beradi (implicit `any`, `null` ignore qilinadi) — compiler xatolarni o'tkazib yuboradi. Strict mode bu yumshoq xatti-harakatlarni o'chiradi, type safety'ni full enforce qiladi.

**QANDAY ISHLAYDI:** `strict: true` declarative — compiler har strict family flag'ni `true` deb default qiladi. Individual flag'ni `false` qilib override qilish mumkin (`"strict": true, "strictNullChecks": false`). Yangi TS versiyada strict family'ga yangi flag qo'shilsa, `strict: true` avtomatik yangi flag'ni ham yoqadi (breaking change risk — releases'da diqqat).

| Flag | Nima qiladi |
|------|-------------|
| `noImplicitAny` | Type annotation yo'q bo'lsa compiler implicit `any` deb taxmin qilmaydi — xato beradi |
| `strictNullChecks` | `null` va `undefined` har type ga avtomatik assignable emas — alohida member sifatida narrowing kerak |
| `strictFunctionTypes` | Function parameter larida **contravariance** enforce (parameter pozitsiyasi). `method` syntax bundan mustasno (bivariance bilan qoladi — backward-compat) |
| `strictBindCallApply` | `Function.prototype.bind`/`call`/`apply` parametrlarini target function'ga moslashtirib type-check qiladi |
| `strictPropertyInitialization` | Class instance property lar `constructor` ichida (yoki declaration'da) initialize bo'lishi kerak — aks holda `undefined` bo'lib qoladi |
| `noImplicitThis` | Function ichida `this` aniqlanmasa (implicit `any`) — xato |
| `alwaysStrict` | Har emit qilingan JS modulga `"use strict"` directive qo'shadi (ES2015+ modullar avtomatik strict, lekin script fayllar emas) |
| `useUnknownInCatchVariables` (TS 4.4+) | `try { } catch (e)` da `e` type'i `unknown` (avval `any` edi — narrowing majburiy emas edi) |
| `strictBuiltinIteratorReturn` (TS 5.6+) | Built-in iterator (`Map.entries()`, `Set.values()`) `return()` natijasi `IteratorResult<T, undefined>` — avval `any` edi |

<details>
<summary><strong>Under the Hood</strong></summary>

**`strictNullChecks` mexanizmi:** Compiler har type'ni `T` yoki `T | null | undefined` deb ajratadi. Ilgari `null` har type'ning member edi (bottom type sifatida) — `let s: string = null` o'tar edi. Strict mode'da `null` va `undefined` alohida unit type'lar. Type checker control flow analysis orqali narrowing qiladi (`if (x !== null) { /* x: T */ }`).

**`strictFunctionTypes` va variance:** TS function type assignability'ni tekshirganda parameter pozitsiyasini contravariant qiladi (return pozitsiyasi covariant). `(x: Animal) => void` ni `(x: Dog) => void` ga assign qilib bo'lmaydi — chunki birinchi function har Animal'ni qabul qiladi, ikkinchisi faqat Dog'ni. **Method syntax** (`f(x: T): U` shape) bivariant qoldirilgan (backward-compat) — bu maxsus bypass.

**`strictPropertyInitialization` mexanizmi:** Compiler class constructor body'sini Definite Assignment Analysis bilan tekshiradi — har declared property constructor tugaganda assigned bo'lishi shart. Bypass: `name!: string` (definite assignment assertion) yoki `name: string | undefined`.

**`useUnknownInCatchVariables` motivatsiyasi:** Promise rejection yoki `throw` har qanday qiymatni tashlashi mumkin (`throw 42`, `throw "string"`, `throw {}`) — `Error` instance bo'lishi kafolatlanmagan. `unknown` narrowing'ni majbur qiladi: `if (e instanceof Error) { ... }`.

```typescript
// strictFunctionTypes — variance misol
type AnimalHandler = (a: Animal) => void;
type DogHandler = (d: Dog) => void;
declare let ah: AnimalHandler;
declare let dh: DogHandler;
ah = dh; // ❌ contravariant: Dog handler Animal'ni qabul qila olmaydi
dh = ah; // ✅ covariant subtype
```

**Performance ta'siri:** Strict family flag'lar type-check vaqtini oshiradi (qo'shimcha control flow analysis), lekin emit vaqtiga ta'sir qilmaydi. Yangi project strict mode bilan boshlanishi kerak: strict-off holda yozilgan kod keyinroq strict'ga migrate qilinganda yuzlab `null`/`any` xatosi bir vaqtda chiqadi, shuning uchun boshidan strict yoqilgani arzonga tushadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === strictNullChecks ===
function getLength(s: string | null): number {
  // return s.length; // ❌ — s null bo'lishi mumkin
  return s?.length ?? 0; // ✅
}

// === noImplicitAny ===
// function log(msg) {} // ❌ — msg implicitly 'any'
function log(msg: string) {} // ✅

// === strictPropertyInitialization ===
class User {
  // name: string; // ❌ — constructor da set qilinmagan
  name: string;
  constructor(name: string) { this.name = name; } // ✅
}

// === useUnknownInCatchVariables ===
try { /* ... */ } catch (e) {
  // e: unknown — endi narrowing kerak
  if (e instanceof Error) console.log(e.message);
}

// === strictFunctionTypes ===
type Handler = (event: MouseEvent) => void;
type GeneralHandler = (event: Event) => void;

let mouseHandler: Handler = (e) => console.log(e.clientX);
// let general: GeneralHandler = mouseHandler; // ❌ — contravariant
```

</details>

---

## Target va Lib

### Nazariya

**`target`** — compiler generate qiladigan JavaScript'ning ECMAScript versiyasini belgilaydi. `"ES2022"` deganda optional chaining (`?.`), nullish coalescing (`??`), class fields, private fields (`#x`), top-level await — barchasi native syntax sifatida qoladi. `"ES5"` deganda compiler downlevel transformation qiladi: `class` → ES5 prototype-based function, `let/const` → `var` + IIFE scope, `async/await` → generator + helper. Faqat **syntax** transform qilinadi — runtime API'lar (`Promise`, `Map`) polyfill emas, faqat type sifatida qaytariladi.

**`lib`** — qaysi built-in type declaration'lar (`lib.*.d.ts` fayllar) compile contextida mavjud bo'lishini belgilaydi. Bu `Promise`, `Map`, `Symbol.asyncIterator`, `document`, `HTMLElement` kabi type'larni o'z ichiga oladi. `lib` belgilanmasa, compiler `target` ga mos kelgan default lib'larni yuklaydi.

**Asosiy farq:**

| Aspect | `target` | `lib` |
|--------|----------|-------|
| Nimaga ta'sir | Emit qilingan JS syntax | Type-check'da mavjud type'lar |
| Runtime | Transformation | Hech qanday emit yo'q |
| Misol | `target: "ES5"` → `class` ni function qiladi | `lib: ["ES2020"]` → `Promise<T>` type'i mavjud |

**Practical combination:** `target: "ES5"` bo'lsa-da, `lib: ["ES2020", "DOM"]` qo'yib `Promise`, `Map`, `Set` type'larini ishlatish mumkin — lekin runtime'da polyfill (`core-js`) alohida qo'shish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**`lib.*.d.ts` fayllar:** TypeScript distribution'da `lib.es5.d.ts`, `lib.es2015.core.d.ts`, `lib.es2015.iterable.d.ts`, `lib.dom.d.ts`, va boshqalar bor (TS package'ning `lib/` folder'ida). Har `lib` qiymati ushbu fayllarni `node_modules/typescript/lib/` dan o'qishni triggerlaydi.

**Downlevel iteration mexanizmi (`downlevelIteration`):** ES5 target'da `for...of` faqat array uchun ishlaydi (length+index). String, Set, Map, custom iterator'lar uchun ishlamaydi. `downlevelIteration: true` qo'yilsa, compiler `for...of`'ni `__values` helper bilan o'raydi — `Symbol.iterator` protocol'ni runtime'da ishlatadi. Bu emit hajmini oshiradi (har faylga helper'lar inline) — `importHelpers: true` + `tslib` package bilan helper'larni umumiy runtime'dan import qilib bundle size kichraytiriladi.

```typescript
// TS source
for (const c of "hello") console.log(c);

// target: ES5 + downlevelIteration: false → ❌ runtime xato (string ga length+index ishlatadi)
// target: ES5 + downlevelIteration: true → __values helper bilan to'g'ri emit
```

**Lib evolution:** Har TS release lib fayllarini yangilaydi (yangi browser API, yangi ES feature). TS 5.0 da `lib.es2023.d.ts` qo'shildi (`Array.prototype.findLast`, `Array.prototype.toSorted`). Lib version'ni TS version bilan bog'lash kerak — eski TS bilan yangi `lib` ishlatib bo'lmaydi.

**`target` va emit helper'lar:** ES5 target ko'p helper function generate qiladi (`__extends`, `__assign`, `__awaiter`, `__generator`). `importHelpers: true` + `tslib` package ularni bitta runtime'dan import qilib bundle size'ni kamaytiradi (har faylga duplicate emas).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
// Node.js 20 (ES2023 syntax + types)
{ "target": "ES2022", "lib": ["ES2022"] }

// Browser app (modern, IE11 yo'q)
{ "target": "ES2020", "lib": ["ES2020", "DOM", "DOM.Iterable"] }

// Browser app (eski browser — IE11 support)
{ "target": "ES5", "lib": ["ES2020", "DOM"], "downlevelIteration": true }
// Promise, Map, Set polyfill kerak (core-js / runtime)
```

</details>

---

## Module Options

### Nazariya

Module-related option'lar TypeScript'ning import/export ni qanday compile qilishini va modul fayllarni qanday topishini boshqaradi.

**`module`** — emit qilingan JS qanday module format ishlatadi. `"CommonJS"` → `require`/`module.exports`. `"ESNext"`/`"ES2022"` → native `import`/`export` (bundler hal qiladi). `"Node16"`/`"NodeNext"` → fayl extension va `package.json` `"type"` field'ga qarab CJS yoki ESM emit (Node.js native ESM uchun). `"Preserve"` (TS 5.4+) → ESM import syntax'ni o'zgartirmasdan saqlaydi (bundler oxirgi transformation qiladi).

**`moduleResolution`** — compiler import path'ni qanday resolve qiladi (qaysi faylni topadi). `"Node10"` (eski "Node") — klassik Node algoritmi (`node_modules` qidirish + `package.json` `"main"`). `"Node16"`/`"NodeNext"` — modern Node ESM rules (`exports` field, conditional exports, `.js`/`.mjs`/`.cjs` extension). `"Bundler"` (TS 5.0+) — bundler-friendly (extension yo'q, `exports` field qo'llaniladi, lekin Node ESM constraint'lariga qattiq emas).

| Option | Qiymat | Muhit |
|--------|--------|-------|
| `module` | `"Node16"` / `"NodeNext"` | Node.js native ESM |
| `module` | `"ESNext"` / `"Preserve"` (TS 5.4+) | Bundler (Vite, Webpack, esbuild) |
| `module` | `"CommonJS"` | Node.js legacy (CJS-only) |
| `moduleResolution` | `"Node16"` / `"NodeNext"` | Node.js (ESM + CJS dual) |
| `moduleResolution` | `"Bundler"` (TS 5.0+) | Vite/Webpack/Next.js |
| `verbatimModuleSyntax` | `true` (TS 5.0+) | Yangi project'lar (type import explicit) |
| `isolatedModules` | `true` | esbuild/SWC/Vite (single-file transpilation) |
| `moduleDetection` | `"force"` | Har fayl module sifatida talqin (top-level scope global emas) |

**Qoida:** `module` va `moduleResolution` **juftlikda** sozlanishi shart — nomos combination runtime xatolarga olib keladi (masalan `module: "ESNext"` + `moduleResolution: "Node10"` → `exports` field ishlamaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

**`verbatimModuleSyntax` mexanizmi (TS 5.0+):** Avval `import { Foo } from "./mod"` da `Foo` faqat type sifatida ishlatilsa, emit'da import elide qilinardi (compile-time da olib tashlash). Bu `isolatedModules` bilan ishlamasdi — single-file transpiler (esbuild) `Foo` type ekanini bilolmaydi. `verbatimModuleSyntax: true` → `import type { Foo }` explicit yozish kerak. Type import elide qilinadi, value import qoldiriladi. Natija: emit predictable, esbuild/SWC bilan to'liq compatible.

```typescript
// ✅ verbatimModuleSyntax: true
import type { User } from "./types";  // elide qilinadi (emit'da yo'q)
import { fetchUser } from "./api";    // emit'da qoladi

// ❌ — type import explicit emas
import { User, fetchUser } from "./mixed";
// Error: 'User' is a type and must be imported using a type-only import
```

**`isolatedModules` constraint'lari:** Single-file transpiler'lar (esbuild, SWC, swc-loader) har faylni alohida-alohida compile qiladi — boshqa fayllar context'i mavjud emas. Bu ba'zi TS feature'larni buzadi:
- `export const enum` → cross-file inline qilolmaydi → ban
- `import x = y.z` (namespace) → ban
- Type-only re-export → ambiguous (type yoki value?) → `export type` aniq talab
- `declaration merge` → ban (cross-file)

**`moduleDetection`** — fayl module yoki script ekanini aniqlash:
- `"auto"` (default) → `import`/`export` bo'lsa module
- `"force"` → har `.ts`/`.tsx` faylni module deb belgilaydi (top-level `var` global scope'ga chiqmaydi)
- `"legacy"` → eski TS xatti-harakati (deprecated)

`moduleDetection: "force"` — modern project'lar uchun tavsiya (har fayl izolyatsiyalangan scope).

```
Source fayl
    ↓
moduleDetection check (module yoki script?)
    ↓
module bo'lsa → ESM/CJS bo'yicha emit
    ↓
moduleResolution → import path resolve
    ↓
verbatimModuleSyntax → type vs value import ajratish
```

**`Preserve` qiymati (TS 5.4+):** `module: "Preserve"` — TS source ESM syntax'ni emit'da o'zgartirmasdan saqlaydi. Bundler (Vite/Rollup) keyin o'z transformation qiladi. Bu `moduleResolution: "Bundler"` bilan ideal juftlik — TS faqat type-check, transformation bundler'da.

</details>

---

## Paths va Directories

### Nazariya

Bu option'lar source/output directory tuzilishini va import path alias'larini boshqaradi.

| Option | Nima qiladi |
|--------|-------------|
| `outDir` | Emit qilingan `.js` (va `.d.ts`) fayllar qayerga yoziladi |
| `rootDir` | Source fayllar root papkasi — `outDir` ichidagi directory structure'ni belgilaydi |
| `baseUrl` | Import resolve uchun base directory (TS 4.1 dan oldin `paths` uchun majburiy edi) |
| `paths` | Import alias mapping'lari (`@/*` → `src/*`) — faqat type-check |
| `rootDirs` | Bir nechta papkani bitta virtual directory dek ko'rsatadi (kamdan-kam kerak) |

**Muhim nuance — `paths` runtime'da ishlamaydi:** Compiler import path'ni `paths` orqali resolve qiladi (faylni topadi), lekin **emit qilingan JS'da import path o'zgarmaydi** — compiler path rewriting qilmaydi. Runtime tomonda alohida tool kerak:
- **Bundler (Vite/Webpack)** — `vite.config.ts` da `resolve.alias` yoki `webpack.config.js` da `resolve.alias` parallel sozlanadi
- **Node.js runtime** — `tsconfig-paths` package (development), yoki `tsc-alias`/`tsup` (build vaqtida path rewriting)

**`rootDir` mexanizmi:** `rootDir: "src"`, `outDir: "dist"`, fayl `src/utils/log.ts` → emit `dist/utils/log.js`. `rootDir` belgilanmasa, compiler avtomatik **eng past umumiy directory**'ni hisoblaydi — bu kutilmagan output strukturasiga olib kelishi mumkin (`src/order.ts` va `tests/order.test.ts` ikkalasi compile bo'lsa, common root project root'i bo'lib qoladi va emit `dist/src/...` + `dist/tests/...` ko'rinishida chiqadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**`paths` resolution algoritmi:**

1. Import statement uchraydi: `import { log } from "@/utils/log"`
2. Compiler `paths` ro'yxatini tekshiradi: `"@/*": ["src/*"]`
3. `"@/utils/log"` patternga mos keladi: `@/*` → `*` qismi `utils/log`
4. Substitution: `src/utils/log`
5. Resolve `src/utils/log.ts` ni topadi → import bog'lanadi
6. Emit'da import path **o'zgarmaydi** — `import { log } from "@/utils/log"` qoladi
7. Runtime tool (bundler/tsconfig-paths) path'ni real fayl yo'liga rewrite qiladi

**`paths` array — multiple fallback:**

```json
{
  "paths": {
    "lodash": ["./vendor/lodash", "node_modules/lodash"]
  }
}
```

Birinchi mavjud yo'l tanlanadi (vendor'da bo'lsa vendor'dan, aks holda node_modules'dan). Monorepo'da lokal package'ni tezroq resolve qilish uchun ishlatiladi.

**`rootDirs` — virtual merge:**

```json
{ "rootDirs": ["src/generated", "src/manual"] }
```

`src/manual/api.ts` ichida `import { Schema } from "./schema"` yozilsa, compiler `src/manual/schema.ts` va `src/generated/schema.ts` ikkalasini ham qidiradi. Bu code generation pipeline'larda foydali (manual+generated fayllar bir scope dek ko'rinadi).

**`baseUrl` evolution:** TS 4.1 dan oldin `paths` `baseUrl` ga muhtoj edi (path'lar `baseUrl` ga nisbatan resolve bo'lardi). TS 4.1+ da `baseUrl` o'rnatilmasa, `paths` `tsconfig.json` joylashgan directory'ga nisbatan resolve qilinadi — `baseUrl` ixtiyoriy bo'ldi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
// tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

```typescript
// src/pages/home.ts
import { Button } from "@components/Button";  // resolve: src/components/Button.ts
import { formatDate } from "@utils/date";     // resolve: src/utils/date.ts
```

```typescript
// vite.config.ts — bundler tomonida parallel alias
import path from "node:path";
export default {
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src"),
      "@components": path.resolve(__dirname, "src/components"),
    },
  },
};
```

</details>

---

## Declaration va Source Maps

### Nazariya

Declaration option'lari TypeScript'ning `.d.ts` (type declaration) va source map fayllarini generate qilishini boshqaradi. Library publish qilganda yoki monorepo'da type'ni boshqa package'lar bilan ulashganda kritik.

| Option | Nima qiladi |
|--------|-------------|
| `declaration` | Har emit qilingan `.js` uchun yondoshida `.d.ts` (type-only) generate qiladi |
| `declarationDir` | `.d.ts` fayllar yoziladigan alohida directory (default `outDir`) |
| `declarationMap` | `.d.ts.map` — IDE'da `.d.ts` symbol'idan TS source'iga navigate |
| `emitDeclarationOnly` | Faqat `.d.ts` emit qiladi, `.js` emas (bundler JS yaratganda foydali) |
| `isolatedDeclarations` (TS 5.5+) | Har fayl declaration'ni mustaqil generate qila olishi shart — explicit return type'lar majburiy |
| `sourceMap` | `.js.map` fayllar generate — debugger TS source'da breakpoint qo'yishi mumkin |
| `inlineSourceMap` | Source map'ni alohida fayl o'rniga `.js` ichiga base64 inline qo'shadi |
| `inlineSources` | Source map ichiga original TS source matnini inline (debug stack'da ko'rinadi) |
| `stripInternal` | JSDoc `@internal` tag'li declaration'larni `.d.ts` dan olib tashlaydi |

<details>
<summary><strong>Under the Hood</strong></summary>

**`isolatedDeclarations` motivatsiyasi (TS 5.5+):** Klassik `declaration: true` mode'da compiler type inference orqali `.d.ts` ni yarata oladi (return type'ni aytmasangiz ham, compiler function body'dan infer qiladi). Bu **butun project'ni** type-check qilishni talab qiladi — har function'ning ichiga kirish kerak.

`isolatedDeclarations: true` qo'shilsa, compiler har faylni mustaqil declaration emit qilolishi shart. Buning uchun har exported function/class explicit return type yozishi kerak:

```typescript
// ❌ isolatedDeclarations xato
export function getUser(id: number) {
  return { id, name: "Ali" };
}
// Error: Function must have an explicit return type annotation
//        with --isolatedDeclarations.

// ✅
export function getUser(id: number): { id: number; name: string } {
  return { id, name: "Ali" };
}
```

**Foyda:** Build tool (esbuild, swc-dts-generator) declaration'ni TS compiler'siz, faqat parse + AST manipulation orqali generate qilishi mumkin → sezilarli tezroq library build (TS team `isolatedDeclarations` ni TS 5.5 announcement'da maxsus shu maqsad bilan kiritgan).

**Source map mexanizmi:** `.js.map` Source Map V3 format'ida (JSON):
- `version` — format versiyasi
- `sources` — original fayllar ro'yxati
- `mappings` — base64 VLQ-encoded mapping'lar (har JS pozitsiyadan TS pozitsiyaga)
- `sourcesContent` (inline mode) — original source matni

```
src/user.ts:10  →  dist/user.js:8 (compiled)
                 ↓ .js.map
                 debugger pauses at src/user.ts:10
```

**`declarationMap` mexanizmi:** `.d.ts.map` — declaration fayl'dagi symbol'larni TS source pozitsiyasiga bog'laydi. IDE (VS Code) "Go to Definition" `.d.ts` ga emas, TS source ga olib boradi. Library consumer uchun foydali (node_modules ichidagi `.d.ts` o'rniga real source).

**`stripInternal` use case:** Library author public va internal API'larini bir project'da saqlashi mumkin. `@internal` JSDoc bilan belgilangan declaration'lar `.d.ts` dan tushiriladi:

```typescript
/** Public API */
export function create(name: string): User { /* ... */ }

/** @internal — testing uchun, public emas */
export function _resetCache(): void { /* ... */ }

// .d.ts: faqat create() qoladi, _resetCache() yo'q
```

</details>

Batafsil [Bo'lim 18: Declaration Files](18-declaration-files.md).

---

## Checking Options

### Nazariya

`strict` dan tashqari qo'shimcha type-check flag'lari — production-grade qattiqlikni oshiradi. `strict: true` bu flag'larni **avtomatik yoqmaydi** — har biri alohida configuration kerak.

| Option | Nima qiladi | Tavsiya |
|--------|-------------|---------|
| `noUncheckedIndexedAccess` | Index access (`arr[i]`, `obj[key]`) natijasi `T \| undefined` | Production'da doim |
| `exactOptionalPropertyTypes` | `prop?: T` — "yo'q bo'lishi mumkin", `T \| undefined` EMAS | Yangi project'larda |
| `noImplicitOverride` | Subclass'da parent method override qilganda `override` keyword majburiy (TS 4.3+) | Tavsiya |
| `noPropertyAccessFromIndexSignature` | Index signature'dagi property uchun faqat `obj["key"]`, dot syntax `obj.key` ban | Ixtiyoriy |
| `noUnusedLocals` | Ishlatilmagan local variable → error | CI da |
| `noUnusedParameters` | Ishlatilmagan function parameter → error (`_` prefix bilan skip) | CI da |
| `noImplicitReturns` | Function'ning barcha code path'lari `return` qaytarishi shart | Tavsiya |
| `noFallthroughCasesInSwitch` | `switch` case'da `break`/`return`/`throw` yo'q bo'lsa fallthrough → error | Tavsiya |
| `allowUnreachableCode` | `false` — unreachable code (return'dan keyin) → warning | Tavsiya |
| `allowUnusedLabels` | `false` — ishlatilmagan label → warning | Tavsiya |

<details>
<summary><strong>Under the Hood</strong></summary>

**`noUncheckedIndexedAccess` mexanizmi:** TypeScript array va record'larda index access type'ini element type'iga teng deb default qiladi (`number[]` → `arr[0]: number`). Realistic da `arr[100]` mavjud bo'lmasligi mumkin. Bu flag index access'ni `T | undefined` qiladi — narrowing majburiy.

```typescript
// noUncheckedIndexedAccess: false (default)
const arr = [1, 2, 3];
const x = arr[100];      // x: number — yolg'on (haqiqatda undefined)
x.toFixed();             // ❌ runtime crash

// noUncheckedIndexedAccess: true
const x = arr[100];      // x: number | undefined
x.toFixed();             // ❌ compile error — narrowing kerak
if (x !== undefined) x.toFixed(); // ✅
```

**`exactOptionalPropertyTypes` farqi:** Klassik mode'da `port?: number` `number | undefined` ga teng:
```typescript
interface Config { port?: number; }
const c: Config = { port: undefined }; // ✅ standart mode'da o'tadi
```

Strict mode'da `?` "property yo'q bo'lishi mumkin" degani — `undefined` qiymat **boshqa**. `port?: number` qabul qiladi: yo'q bo'lish, yoki `number` qiymat. `undefined` ban.

```typescript
const c1: Config = {};                          // ✅ yo'q
const c2: Config = { port: 3000 };              // ✅ qiymat
const c3: Config = { port: undefined };         // ❌ exactOptionalPropertyTypes
```

**`noImplicitOverride` motivatsiyasi:** Subclass'da parent method'ni override qilganda `override` keyword yozish majburiy. Agar parent method nomi keyinroq o'zgartirilsa, subclass'dagi "override" implicit yangi method bo'lib qoladi (bug):

```typescript
class Base { greet() { return "Hi"; } }
class Child extends Base {
  override greet() { return "Hello"; }  // ✅ explicit
}
// Keyin Base'da greet → sayHello deb nomlangan bo'lsa:
// Child.greet override qilmaydi (Base'da bunday method yo'q) → compiler xato beradi
```

**`noFallthroughCasesInSwitch` semantics:** `case` bo'shliqsiz boshqa `case`'ga "tushish" (fallthrough) JavaScript'da legal. Aksariyat hollarda bug:

```typescript
switch (x) {
  case "a": doA();  // ❌ break yo'q — "b" ga ham tushadi
  case "b": doB(); break;
}
```

Lekin explicit fallthrough kerak bo'lsa (kamdan-kam), bo'sh body bilan ruxsat: `case "a": case "b": doAB(); break;`.

**Performance ta'sir:** Bu flag'lar control flow analysis'ni chuqurlashtiradi — yirik project'da type-check vaqtini biroz oshirishi mumkin. Lekin bug catch rate bu qo'shimcha vaqtni qoplaydi (xato production'ga chiqmasdan ushlanadi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === noUncheckedIndexedAccess ===
const headers: Record<string, string> = {};
const ct = headers["Content-Type"];
// ct: string | undefined (noUncheckedIndexedAccess bilan)
// ct: string (siz — xavfli!)
if (ct) console.log(ct.toLowerCase()); // ✅ narrowing

// === exactOptionalPropertyTypes ===
interface Config { host: string; port?: number; }
const c1: Config = { host: "localhost" };              // ✅ — port yo'q
// const c2: Config = { host: "localhost", port: undefined }; // ❌

// === noImplicitOverride ===
class Base { greet() { return "Hello"; } }
class Child extends Base {
  override greet() { return "Hi"; } // ✅ — override keyword
  // greet() { return "Hi"; }       // ❌ — override kerak
}
```

</details>

---

## Emit Options

### Nazariya

Emit option'lari compiler generate qiladigan JavaScript output'ning shaklini va xatti-harakatini belgilaydi.

| Option | Nima qiladi |
|--------|-------------|
| `noEmit` | JS emit qilmaydi (faqat type-check) — bundler JS yaratganda |
| `removeComments` | JS output'dan comment'larni olib tashlaydi |
| `downlevelIteration` | `for...of`, spread, destructuring'ni eski `target` uchun protocol-correct emit qiladi |
| `importHelpers` | TS helper'larni (`__awaiter`, `__spread`, `__extends`) `tslib` package'dan import qiladi (har faylga inline emas) |
| `useDefineForClassFields` | Class field'larni `Object.defineProperty` bilan emit (ES2022 native semantics) |
| `preserveConstEnums` | `const enum` declaration'larni emit'da qoldiradi (inline o'rniga) |
| `newLine` | Emit qilingan fayl'da `crlf` yoki `lf` line ending |

`noEmit: true` — Vite/Next.js/esbuild project'larida standart. Bundler JS yaratadi, `tsc` faqat type-check (`tsc --noEmit`).

<details>
<summary><strong>Under the Hood</strong></summary>

**`useDefineForClassFields` semantics farqi:** Class field'lar ECMAScript spec va eski TS xatti-harakati o'rtasida farqi:

```typescript
// Source
class Animal { name = "cat"; }
class Dog extends Animal {
  name = "rex";       // child field
  speak() { console.log(this.name); }
}
```

- **`useDefineForClassFields: false`** (eski default): `name = "rex"` constructor ichida `this.name = "rex"` deb emit. Parent class'ning getter/setter'larini trigger qiladi.
- **`useDefineForClassFields: true`** (ES2022 native): `Object.defineProperty(this, "name", { value: "rex", writable: true, ... })` deb emit. Parent setter'lar **trigger bo'lmaydi**. ECMAScript spec'iga to'liq mos.

TS 4.3+ `target: "ES2022"` bilan default `true`. Eski Angular/Vue decorator pattern'lar yangi semantics'da buzilishi mumkin.

**`importHelpers` mexanizmi:** Compiler emit'da TS-specific runtime helper'lar generate qiladi (`__extends`, `__awaiter`, `__generator`, `__spread`). Har fayl uchun alohida inline — duplicate. `importHelpers: true` qo'yilsa, helper'lar `tslib` package'dan import qilinadi:

```javascript
// importHelpers: false (default)
var __extends = (this && this.__extends) || (function () { ... })();
// Har bir fayl tepasida shu helper

// importHelpers: true
import { __extends } from "tslib";
// Faqat import — runtime tslib'dan
```

Library publish qilganda `tslib` ni `peerDependency` yoki `dependency` deb qo'shish kerak.

**`noEmit` use case:** Modern bundler'lar (esbuild, swc, Vite) TypeScript'ni `tsc` ga nisbatan sezilarli tezroq compile qiladi (chunki ular type-check qilmaydi — faqat strip type annotations). Type safety uchun `tsc --noEmit` separate jarayonda type-check qiladi (CI'da yoki dev watch mode'da).

```
              ┌──────────────┐
   .ts        │   esbuild    │   .js (tez emit)
   ───────────┤              ├────────────►
              └──────────────┘
                     │
                     │ parallel
                     ▼
              ┌──────────────┐
              │  tsc --noEmit│   Type errors (lekin emit yo'q)
              └──────────────┘
```

</details>

---

## Interop Options

### Nazariya

Interop option'lari ESM/CommonJS module system'lari va boshqa fayl turlari (JSON, JS) bilan ishlashni yengillashtiradi.

| Option | Nima qiladi |
|--------|-------------|
| `esModuleInterop` | CJS module'dan default import sintaksi ruxsat (`import fs from "fs"`) + namespace helper'lar |
| `allowSyntheticDefaultImports` | Default export'i yo'q module'dan default import type-check'da ruxsat (emit o'zgarmaydi) |
| `forceConsistentCasingInFileNames` | Fayl nomi case sensitive — macOS/Windows case-insensitive FS'da muhim |
| `allowJs` | `.js` fayllarni TS project'ga qo'shadi (mixed JS+TS project) |
| `checkJs` | `.js` fayllarni type-check qiladi (JSDoc annotation'lar asosida) |
| `resolveJsonModule` | `.json` fayllarni `import` qilish ruxsat (type'i avtomatik infer) |
| `allowArbitraryExtensions` (TS 5.0+) | `*.css`, `*.svg` kabi non-JS fayllarni import qilish uchun declaration topish |

`esModuleInterop: true` — modern project'larda standart. CJS/ESM interop muammolarini hal qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`esModuleInterop` mexanizmi:** CommonJS `module.exports = X` bilan ESM'ning `export default X` boshqacha shape'ga ega:
- CJS: `module.exports = { port: 3000 }` → `require("./config")` qaytaradi `{ port: 3000 }`
- ESM: `export default { port: 3000 }` → import qilganda `{ default: { port: 3000 } }`

`esModuleInterop: false` bilan TS strict ESM:
```typescript
import fs from "fs";  // ❌ — fs'da default export yo'q (CJS module)
import * as fs from "fs"; // ✅
```

`esModuleInterop: true` bilan compiler `__importDefault` helper qo'shadi:
```javascript
// Emit
const fs_1 = __importDefault(require("fs"));
// __importDefault: agar mod.__esModule bo'lsa qaytar, aks holda { default: mod } qaytar
fs_1.default.readFile(...);
```

Bu `import fs from "fs"` syntax'ni CJS module bilan ham ishlatishga imkon beradi.

**`allowSyntheticDefaultImports` farqi:** `esModuleInterop` `true` qilsa, bu flag avtomatik `true` bo'ladi. Lekin alohida yoqish mumkin — type-check da ruxsat berib, emit'ni o'zgartirmaslik uchun (bundler interop hal qilganda).

**`resolveJsonModule` mexanizmi:** `.json` fayl import qilinsa, compiler JSON content'ni parse qilib type infer qiladi:

```typescript
// config.json: { "name": "MyApp", "version": "1.0.0" }
import config from "./config.json";
// config: { name: string; version: string }
config.name; // ✅ type-safe
```

Emit'da `require("./config.json")` qoladi (bundler/Node JSON'ni handle qiladi). Bu `module: "Node16"`/`"NodeNext"` ESM context'da murakkabroq: native ESM JSON import import attributes talab qiladi — `import config from "./config.json" with { type: "json" }`. `with` syntax (import attributes) TS 5.3'da qo'shilgan; eski `assert { type: "json" }` (import assertions, TS 4.5) deprecated.

**`forceConsistentCasingInFileNames` motivatsiyasi:** macOS HFS+/APFS va Windows NTFS case-insensitive (`User.ts` va `user.ts` bir fayl). Linux ext4/btrfs case-sensitive. Bu cross-platform bug manbasi — Mac'da ishlovchi project CI Linux'da buziladi:

```typescript
// File: src/User.ts
import { User } from "./user";   // Mac/Win: ✅, Linux CI: ❌ ("./user" topilmaydi)
```

`forceConsistentCasingInFileNames: true` qo'shilsa, compiler har import'ni FS'dagi aniq case bilan tekshiradi.

**`allowJs` + `checkJs` migration strategy:** JS project'ni TS'ga aylantirishda incremental approach:
1. `allowJs: true` — JS fayllarni project'ga qo'shish (type-check yo'q, lekin import graph'da)
2. `checkJs: true` — JSDoc annotation'lar asosida type-check
3. Bir-bir faylni `.js` → `.ts` ga ko'chirish

</details>

---

## include, exclude, files

### Nazariya

Bu uch option compile setiga qaysi fayllar kirishini boshqaradi. Ular **input bilan emas, root fayllar** bilan ishlaydi — root fayllardan import graph traversal orqali boshqa fayllar avtomatik qo'shiladi.

```json
{
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"],
  "files": ["src/global.d.ts"]
}
```

- **`include`** — glob pattern array. Default `**/*` (barcha `.ts`/`.tsx`/`.d.ts`)
- **`exclude`** — `include` natijasidan chiqarish (filter). Default `["node_modules", "bower_components", "jspm_packages", outDir]`
- **`files`** — aniq fayllar ro'yxati. Glob **ishlamaydi** — har fayl absolute/relative path

**Glob pattern qoidalari (`include`/`exclude`):**

| Pattern | Mos keladi |
|---------|-----------|
| `*` | Bir directory bo'lagi, har character (slash'siz) |
| `?` | Bir character |
| `**/` | Recursive — nol yoki ko'p directory |

**KRITIK nuance — `exclude` import qilingan faylni to'xtatmaydi:**

`exclude` faqat `include`/`files` orqali topilgan **root fayllar** ro'yxatini filtr qiladi. Lekin TS compile graph import'lar orqali kengayadi. Agar `include` da topilgan fayl `exclude` qilingan faylni `import` qilsa — bu fayl **baribir compile bo'ladi**.

```json
{ "include": ["src/**/*"], "exclude": ["src/internal/**"] }
```

```typescript
// src/app.ts  (root sifatida include qilingan)
import { hashToken } from "./internal/crypto";
// src/internal/crypto.ts — exclude'da, lekin import orqali kelgan → COMPILE
```

**`files` va `include` o'zaro:** Ikkalasi ham root fayllarni belgilaydi. `files` aniq, `include` glob — birga ishlatish mumkin. `files: []` (bo'sh) — `references` bilan project references mode'da metadata-only tsconfig uchun (root tsconfig).

<details>
<summary><strong>Under the Hood</strong></summary>

**Compile graph algoritmi:**

```
1. Root fayllar ro'yxati = files ∪ (include - exclude)
2. Har root fayl uchun:
   - Parse → AST
   - Import statement'larni topib resolve
   - Topilgan fayllarni queue ga qo'shish
3. Queue empty bo'lguncha takrorlash
4. Yakuniy set = barcha kirgan fayllar (root + transitive)
5. Har biri uchun type-check + (emit yoqilsa) emit
```

**Performance ta'sir:** `include` glob'lar kengayganda (`**/*` butun project'da) FS skan vaqti oshadi. Yirik monorepo'da `include: ["src/**/*"]` aniq aytish (faqat src) tezroq. `exclude` da `node_modules` yozish ortiqcha — compiler default'da uni chiqarib tashlaydi.

**`tsc --listFiles`** — qaysi fayllar compile setida ekanini chiqaradi. Debug uchun foydali (kutilmagan fayl kelganini ko'rsatadi).

**`tsc --traceResolution`** — har import resolve qadamlarini chiqaradi (qaysi yo'l, qaysi `package.json` o'qildi, nima topildi/topilmadi).

</details>

---

## extends — tsconfig Inheritance

### Nazariya

`extends` — boshqa `tsconfig.json` dan compiler sozlamalarini meros olish. Monorepo va shared config'lar uchun asosiy mexanizm.

**Merge qoidalari:**
- `compilerOptions` — **shallow merge** (child option base option'ni override qiladi)
- `include`, `exclude`, `files` — child **butunlay override** qiladi (merge emas)
- `references` — child override qiladi
- `extends` — recursive (extends ichidagi config ham extends qila oladi)

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}

// packages/server/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "Node16",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

**TS 5.0+ — array extends** (bir nechta config dan meros, tartib muhim — keyingisi g'olib):

```json
{
  "extends": ["@tsconfig/node20/tsconfig.json", "./tsconfig.base.json"]
}
```

**Tayyor preset npm package'lar** (`@tsconfig/*` namespace):
- `@tsconfig/strictest` — eng qattiq strict + checking flags
- `@tsconfig/node20`, `@tsconfig/node18`, `@tsconfig/node16` — Node version'lar
- `@tsconfig/vite-react`, `@tsconfig/next` — framework-specific
- `@tsconfig/recommended` — sensible default'lar

<details>
<summary><strong>Under the Hood</strong></summary>

**Resolution algoritmi:**

```
tsconfig.json o'qildi
        ↓
extends bormi?
   ↓ ha
extends path resolve:
   - "./relative.json" → relative path
   - "@scope/preset" → node_modules
   - "/absolute.json" → absolute path
        ↓
target config'ni recursive load (uning extends'i ham resolve)
        ↓
Bottom-up merge:
   - Eng chuqur base avval
   - Yuqoriga ko'tarilib har layer override
   - Top tsconfig oxirgi g'olib
        ↓
Default'lar to'ldirildi
        ↓
Compile boshlanadi
```

**`extends` array semantics (TS 5.0+):**

```json
{ "extends": ["base-a.json", "base-b.json"] }
```

Tartib: `base-a` → `base-b` → joriy config. Conflict bo'lsa, **keyingisi g'olib**. `base-b` `base-a` ni override qiladi, joriy config `base-b` ni override qiladi.

**Inheritance limitations:** `compilerOptions.paths` butunlay override qiladi (merge emas). Agar base'da `paths` bor va child'da yana qo'shmoqchi bo'lsangiz — `extends` orqali emas, qo'lda copy qilish kerak. Bu monorepo'da path alias'larni shared qilishni qiyinlashtiradi.

**`tsc --showConfig`** — resolved final config'ni chiqaradi (extends'lar merge qilingan, default'lar to'ldirilgan). Debug uchun majburiy tool — extends chain qanday ishlayotganini ko'rsatadi.

</details>

---

## Project References

### Nazariya

Project references — yirik project'ni mustaqil compile bo'ladigan **kichik unit'larga** bo'lish mexanizmi. Har project alohida `tsconfig.json` ga ega, lekin bir-biriga type-safe `references` orqali bog'lanadi.

**Asosiy foydalari:**
1. **Incremental build** — faqat o'zgargan project'lar qayta compile bo'ladi (build cache `.tsbuildinfo` orqali)
2. **Parallel compilation** — mustaqil project'lar parallel kompile (build tool darajasida)
3. **Encapsulation** — har project o'z `compilerOptions`, `include`, `exclude` ga ega
4. **Type-safe boundaries** — circular dependency'lar build vaqtida ushlanadi

**`composite: true` ta'siri** (referenced project'da majburiy):
- `declaration: true` avtomatik (`.d.ts` generate kerak — boshqa project'lar uchun)
- `rootDir` default → `tsconfig.json` joylashgan papka (eng past umumiy yo'l emas)
- `incremental: true` avtomatik (`.tsbuildinfo` generate)
- Har source fayl `include`/`files` orqali aniq qamrab olingan bo'lishi shart — `composite` mode'da compile root set faqat `import` graph traversal'iga tayanmaydi, har fayl `include`/`files` pattern'iga tushishi kerak

```json
// packages/core/tsconfig.json — referenced project
{
  "compilerOptions": { "composite": true, "outDir": "dist", "rootDir": "src" },
  "include": ["src"]
}

// packages/server/tsconfig.json — referrer
{
  "compilerOptions": { "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../core" }],
  "include": ["src"]
}

// Root tsconfig.json — solution-level
{
  "files": [],
  "references": [
    { "path": "packages/core" },
    { "path": "packages/server" }
  ]
}
```

**Build commands:**
- `tsc --build` (yoki `tsc -b`) — dependency graph quradi, topological order'da compile
- `tsc -b --clean` — `.tsbuildinfo` va emit fayllarni tozalaydi
- `tsc -b --force` — incremental cache'ni ignore qilib to'liq qayta build
- `tsc -b --watch` — fayl o'zgarishlarini kuzatib incremental rebuild

<details>
<summary><strong>Under the Hood</strong></summary>

**`.tsbuildinfo` format:**

Project'ni birinchi marta build qilganda compiler `.tsbuildinfo` fayl yaratadi (JSON):
- Source fayllar fingerprint (modification time + content hash)
- Output fayllar map
- Dependency graph snapshot
- Compile metadata

Keyingi build'larda compiler har fayl fingerprint'ni tekshiradi:
- O'zgarmagan → skip
- O'zgargan → qayta compile + downstream'larni invalidate

```
Build 1: barcha fayllar compile (cold)
   ↓ .tsbuildinfo yozildi

Build 2: faqat user.ts o'zgargan
   ↓
   Compiler user.ts hash'ni solishtiradi → o'zgargan
   user.ts qayta compile
   user.ts ga bog'liq fayllar (downstream) topiladi
   Faqat ular qayta compile (boshqalar skip)
```

**Topological build order:**

```
References graph:
   server → core → utils

tsc -b build order:
1. utils (no dependencies)
2. core (depends on utils, utils tugagach)
3. server (depends on core, core tugagach)
```

Circular reference (`core → server → core`) bo'lsa, compiler build vaqtida error beradi.

**Referenced project'lar uchun emit constraint'lari:** Referenced project `composite: true` → `.d.ts` emit qiladi. Referrer project import qilganda real `.ts` source'ga emas, emitted `.d.ts` ga qaraydi. Bu encapsulation beradi: referenced internal'lar leak bo'lmaydi (`stripInternal` bilan kuchaytiriladi).

**`disableSourceOfProjectReferenceRedirect`:** Default'da TS editor (VS Code) "Go to Definition" referenced project'ning real source'iga olib boradi (yaxshi DX). Bu flag'ni `true` qilsa, faqat `.d.ts` ga olib boradi (production emit bilan to'liq mos). Kamdan-kam ishlatiladi.

**Performance trade-off:** Project references o'rnatish overhead beradi (config'lar ko'p). Foyda yirik monorepo'da ko'p package bo'lganda seziladi. Kichik project'da single `tsconfig.json` afzal.

</details>

---

## ESLint + TypeScript

### Nazariya

`typescript-eslint` — TypeScript uchun ESLint plugin va parser. Klassik ESLint AST orqali ishlaydi (faqat syntax), `typescript-eslint` esa **type information** ham ishlatadi — TS compiler API orqali real type'larni o'qib qoidalar enforce qiladi.

**Type-aware vs Syntax-only:**

| Rule turi | Misol | Talab |
|----------|-------|-------|
| Syntax-only | `prefer-const`, `no-var` | Faqat AST |
| Type-aware | `no-floating-promises`, `no-unsafe-assignment`, `no-misused-promises` | TS Program (type info) |

Type-aware rule'lar yoqilsa, `parserOptions.projectService: true` (yoki eski `project: "./tsconfig.json"`) kerak. Bu lint vaqtini sezilarli oshiradi (TS Program quriladi), lekin kuchli rule'lar beradi:
- **`no-floating-promises`** — handle qilinmagan Promise (await/`.then`/`.catch` yo'q) → error
- **`no-misused-promises`** — `if (asyncFn())` yoki `void` callback'da Promise ishlatish → error
- **`no-unsafe-assignment`** — `any` qiymatni typed variable'ga assignment → error

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// eslint.config.mjs (flat config)
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/no-floating-promises": "error",
    },
  },
  {
    files: ["**/*.test.ts"],
    rules: {
      "@typescript-eslint/no-explicit-any": "off",
    },
  },
  { ignores: ["dist/", "node_modules/"] }
);
```

</details>

---

## Best Practices Per Project Type

### Nazariya

| Project turi | `module` | `moduleResolution` | Boshqa muhim |
|-------------|----------|-------------------|--------------|
| **React (Vite)** | `"ESNext"` | `"Bundler"` | `noEmit`, `isolatedModules`, `jsx: "react-jsx"` |
| **Next.js** | `"ESNext"` | `"Bundler"` | `noEmit`, plugin: `"next"` |
| **Node.js API** | `"Node16"` | `"Node16"` | `declaration`, `sourceMap` |
| **Library** | `"Node16"` | `"Node16"` | `declaration`, `declarationMap`, `isolatedDeclarations` |
| **Monorepo** | `"Node16"` | `"Node16"` | `composite`, `references`, `incremental` |

<details>
<summary><strong>Kod Misollari</strong></summary>

**React (Vite):**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

**Node.js API:**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
```

</details>

---

## Edge Cases va Gotchas

### 1. `exclude` import qilingan faylni to'xtatmaydi

```json
{ "include": ["src"], "exclude": ["src/internal"] }
```

```typescript
// src/app.ts
import { hashToken } from "./internal/crypto"; // BARIBIR compile bo'ladi
// exclude faqat "include" ga ta'sir qiladi, import ga emas
```

### 2. `target: "ES5"` bilan `Promise` type yo'q

```json
{ "compilerOptions": { "target": "ES5" } }
// lib default (ES5 target): ["DOM", "ES5", "ScriptHost"] — Promise type yo'q
// FIX: "lib": ["ES2015", "DOM"]
```

### 3. `useDefineForClassFields: true` — class property semantics o'zgaradi

```typescript
class Base { x = 1; }
class Child extends Base { x = 2; }
// useDefineForClassFields: true → Object.defineProperty semantics
// useDefineForClassFields: false → assignment semantics
// Bu accessor override va inheritance da farq qiladi
```

### 4. `composite: true` — `declaration` avtomatik, `rootDir` inferred

```json
{ "compilerOptions": { "composite": true } }
// composite: true → declaration: true avtomatik yoqiladi (explicit yozish shart emas)
// rootDir berilmasa → tsconfig.json papkasi (yoki include'dan eng past umumiy yo'l) inferred
```

`composite` boshqa qiymatlarga ham ta'sir qiladi: `incremental: true` avtomatik yoqiladi, `tsBuildInfoFile` default joyi belgilanadi. Eslatma: `composite: true` `noEmit: true` bilan birga ishlamaydi (`tsc` error TS5053 beradi) — referenced project `.d.ts` emit qilishi shart. `noEmitOnError` qat'iyligi esa `composite`'dan emas, build mode'dan keladi: `tsc -b` har project'ni go'yo `noEmitOnError` yoqilgandek compile qiladi (xato bo'lgan project emit qilmaydi va keyingi build'da skip bo'lib qolmaydi).

### 5. `verbatimModuleSyntax` CJS da import syntax ni cheklaydi

```typescript
// module: "CommonJS" + verbatimModuleSyntax: true
import { readFile } from "fs"; // ❌ — ESM syntax CJS module da!
// FIX: import readFile = require("fs") yoki module: "Node16"
```

---

## Common Mistakes

### ❌ Xato 1: `strict: true` qo'ymaslik

```json
// ❌ — strict o'chirilgan
{ "compilerOptions": { } }
// null/undefined tekshirilmaydi, any hamma joyda ruxsat

// ✅ — har doim yoqing
{ "compilerOptions": { "strict": true } }
```

### ❌ Xato 2: `module` va `moduleResolution` mos kelmasligi

```json
// ❌ — module ESNext lekin resolution node (legacy)
{ "module": "ESNext", "moduleResolution": "Node" }
// exports field qo'llab-quvvatlanmaydi!

// ✅ — to'g'ri juftliklar
{ "module": "Node16", "moduleResolution": "Node16" }
{ "module": "ESNext", "moduleResolution": "Bundler" }
```

### ❌ Xato 3: `noEmit` qo'ymaslik (bundler bilan)

```json
// ❌ — Vite bilan tsc ham JS emit qiladi → conflict
{ "compilerOptions": { } }
// Ikki joyda JS yaratiladi!

// ✅ — bundler JS yaratadi, tsc faqat type-check
{ "compilerOptions": { "noEmit": true } }
```

### ❌ Xato 4: `isolatedModules` yoqmaslik (esbuild/SWC bilan)

```typescript
// isolatedModules: false + esbuild
export const enum OrderStatus { Pending, Shipped } // tsc OK, lekin xavfli
// esbuild har faylni alohida transpile qiladi — const enum qiymatini
// boshqa faylga inline qila olmaydi. isolatedModules: true buni
// compile-time error qilib oldindan ushlaydi.
// FIX: isolatedModules: true (yoki oddiy enum ishlatish)
```

### ❌ Xato 5: `paths` alias bundler/runtime sozlamasiz

```json
// ❌ — tsconfig'da paths, lekin runtime'da resolve qilolmaydi
{ "compilerOptions": { "paths": { "@/*": ["src/*"] } } }
```

```typescript
// src/app.ts — tsc type-check ✅, lekin Node.js da:
import { formatDate } from "@/utils";
// Runtime: Error: Cannot find module '@/utils'
```

```typescript
// ✅ — bundler tomonida parallel alias
// vite.config.ts
export default { resolve: { alias: { "@": "/src" } } };

// Yoki Node.js'da tsc-alias / tsconfig-paths bilan path rewrite
```

---

## Amaliy Mashqlar

### Mashq 1: Node.js API tsconfig (Oson)

**Savol:** Node.js 20 API uchun tsconfig yarating.

<details>
<summary>Javob</summary>

```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "Node16", "moduleResolution": "Node16",
    "strict": true, "noUncheckedIndexedAccess": true,
    "outDir": "dist", "rootDir": "src",
    "declaration": true, "sourceMap": true,
    "esModuleInterop": true, "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
```

</details>

---

### Mashq 2: Library tsconfig (O'rta)

**Savol:** `isolatedDeclarations`, `verbatimModuleSyntax`, test excluded.

<details>
<summary>Javob</summary>

```json
{
  "compilerOptions": {
    "target": "ES2020", "module": "Node16", "moduleResolution": "Node16",
    "strict": true, "noUncheckedIndexedAccess": true,
    "outDir": "dist", "rootDir": "src",
    "declaration": true, "declarationMap": true,
    "isolatedDeclarations": true, "stripInternal": true,
    "verbatimModuleSyntax": true, "esModuleInterop": true, "skipLibCheck": true
  },
  "include": ["src"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

</details>

---

### Mashq 3: Monorepo setup (Qiyin)

**Savol:** `core`, `server`, `client` — base config, project references.

<details>
<summary>Javob</summary>

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022", "module": "Node16", "moduleResolution": "Node16",
    "strict": true, "composite": true, "incremental": true,
    "declaration": true, "declarationMap": true, "sourceMap": true,
    "esModuleInterop": true, "skipLibCheck": true
  }
}

// packages/core/tsconfig.json
{ "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src" },
  "include": ["src"] }

// packages/server/tsconfig.json
{ "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src" },
  "include": ["src"],
  "references": [{ "path": "../core" }] }

// packages/client/tsconfig.json
{ "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src",
    "lib": ["ES2022", "DOM", "DOM.Iterable"], "jsx": "react-jsx" },
  "include": ["src"],
  "references": [{ "path": "../core" }] }

// tsconfig.json (root)
{ "files": [],
  "references": [
    { "path": "packages/core" },
    { "path": "packages/server" },
    { "path": "packages/client" }
  ] }
```

</details>

---

### Mashq 4: ESLint flat config (O'rta)

**Savol:** `typescript-eslint` strict, `_` prefix unused skip, test da `any` ruxsat.

<details>
<summary>Javob</summary>

```javascript
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/no-floating-promises": "error",
    },
  },
  {
    files: ["**/*.test.ts"],
    rules: { "@typescript-eslint/no-explicit-any": "off" },
  },
  { ignores: ["dist/"] }
);
```

</details>

---

### Mashq 5: Checking options (Oson)

**Savol:** `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride` bilan kodni xavfsiz qiling.

<details>
<summary>Javob</summary>

```typescript
// exactOptionalPropertyTypes: true
interface Config { host: string; port?: number; }
const c: Config = { host: "localhost" };
// const c2: Config = { host: "localhost", port: undefined }; // ❌

// noUncheckedIndexedAccess: true
const headers: Record<string, string> = {};
const ct = headers["Content-Type"]; // string | undefined
if (ct) console.log(ct.toLowerCase());

// noImplicitOverride: true
class Base { greet() { return "Hello"; } }
class Child extends Base { override greet() { return "Hi"; } }
```

</details>

---

## Xulosa

`tsconfig.json` — TypeScript project'ining asosiy configuration fayli. To'g'ri sozlash type safety, DX, va build performance ga ta'sir qiladi.

**Eng muhim qoidalar:**

1. **`strict: true`** — har doim
2. **`module`/`moduleResolution`** juftligi — Node16/Node16 yoki ESNext/Bundler
3. **`target` vs `lib`** — output syntax vs available types
4. **Bundler** — `noEmit`, `isolatedModules`
5. **Library** — `declaration`, `declarationMap`, `isolatedDeclarations`
6. **Monorepo** — `composite`, `references`, `tsc --build`
7. **`noUncheckedIndexedAccess`** — production da doim
8. **`verbatimModuleSyntax`** — yangi project'larda
9. **ESLint** — `typescript-eslint` strict + type-aware

**"Golden" tsconfig — Node.js:**

```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "Node16", "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "strict": true, "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true, "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true, "esModuleInterop": true,
    "skipLibCheck": true, "forceConsistentCasingInFileNames": true,
    "declaration": true, "declarationMap": true, "sourceMap": true,
    "outDir": "dist", "rootDir": "src", "incremental": true
  },
  "include": ["src"]
}
```

**Bog'liq bo'limlar:**
- [Bo'lim 17: Modules](17-modules.md) — module resolution strategies
- [Bo'lim 18: Declaration Files](18-declaration-files.md) — declaration emit options

---

**Keyingi bo'lim:** [23-type-safe-patterns.md](23-type-safe-patterns.md) — Branded types, exhaustive matching, runtime validation, va boshqa type-safe pattern lar.
