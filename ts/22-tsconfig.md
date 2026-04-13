# Bo'lim 22: tsconfig.json Mastery

> `tsconfig.json` — TypeScript proyektining markaziy konfiguratsiya fayli. U kompilatorga qanday type check qilish, qaysi fayllarni compile qilish, qaysi target ga emit qilish, va qanday module system ishlatishni aytadi. Har bir muhim compiler option — nazariya, amaliy misollar, va real-world best practices bilan.

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

`tsconfig.json` — TypeScript proyektining root direktoriyasida joylashgan konfiguratsiya fayli. `tsc` shu faylni o'qib ishlaydi.

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

- **`compilerOptions`** — kompilator sozlamalari (eng katta qism)
- **`include`** — qaysi fayllar compile qilinadi (glob pattern)
- **`exclude`** — qaysilarni chiqarib tashlash
- **`files`** — aniq fayllar ro'yxati (glob emas)
- **`extends`** — boshqa tsconfig dan meros olish
- **`references`** — project references (monorepo)

`tsc --init` — standart tsconfig yaratadi. `tsc --showConfig` — effective config ni ko'rsatadi (extends resolved).

---

## Strict Family

### Nazariya

`strict: true` — **barcha strict flag larni** birdan yoqadi. Bu meta-flag — 8 ta individual flag ning shortcut i. **Har doim yoqing** — yangi loyihalarda standard.

| Flag | Nima qiladi |
|------|-------------|
| `noImplicitAny` | Type annotation yo'q bo'lsa `any` deb taxmin qilmaydi — xato beradi |
| `strictNullChecks` | `null`/`undefined` — alohida type (har joyda assignable emas) |
| `strictFunctionTypes` | Function parameter larda **contravariance** enforce (method syntax bundan mustasno) |
| `strictBindCallApply` | `bind`, `call`, `apply` parametrlarini type-check qiladi |
| `strictPropertyInitialization` | Class property lar constructor da initialize bo'lishi kerak |
| `noImplicitThis` | `this` type aniq bo'lmasa — xato |
| `alwaysStrict` | Har JS faylga `"use strict"` qo'shadi |
| `useUnknownInCatchVariables` (TS 4.4+) | `catch(e)` da `e: unknown` (TS 4.0+ `any` edi) |

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

**`target`** — JS output ning qaysi ECMAScript versiyada bo'lishini belgilaydi. `"ES2022"` deganda — optional chaining, nullish coalescing, class fields — barchasi native qoladi. `"ES5"` deganda — barchasi polyfill/downlevel compile bo'ladi.

**`lib`** — qaysi **built-in type declaration** lar mavjud. `target` ga qarab default lib tanlandi, lekin alohida sozlash mumkin.

**Muhim farq:** `target` = output syntax, `lib` = available types. `target: "ES5"` bo'lsa-da, `lib: ["ES2020", "DOM"]` qo'yib `Promise`, `Map`, `Set` type larini ishlatish mumkin (polyfill bilan).

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
// Node.js 20
{ "target": "ES2022", "lib": ["ES2022"] }

// Browser app (IE11 support kerak emas)
{ "target": "ES2020", "lib": ["ES2020", "DOM", "DOM.Iterable"] }

// Browser app (eski browser)
{ "target": "ES5", "lib": ["ES2020", "DOM"] }
// ⚠️ Promise, Map polyfill kerak (core-js)
```

</details>

---

## Module Options

### Nazariya

Module-related option lar [Bo'lim 17](17-modules.md) da batafsil. Qisqacha:

| Option | Qiymat | Muhit |
|--------|--------|-------|
| `module` | `"Node16"` | Node.js ESM |
| `module` | `"ESNext"` / `"Preserve"` | Bundler (Vite, Webpack) |
| `module` | `"CommonJS"` | Node.js legacy |
| `moduleResolution` | `"Node16"` | Node.js |
| `moduleResolution` | `"Bundler"` | Vite/Webpack |
| `verbatimModuleSyntax` | `true` | Yangi loyihalar |
| `isolatedModules` | `true` | esbuild/SWC/Vite |
| `moduleDetection` | `"force"` | Barcha fayllar module |

**Qoida:** `module` va `moduleResolution` — **juftlikda** sozlang.

---

## Paths va Directories

### Nazariya

| Option | Nima qiladi |
|--------|-------------|
| `outDir` | Compiled JS fayllar qayerga yoziladi |
| `rootDir` | Source fayllar root papkasi |
| `baseUrl` | Import resolve uchun base directory |
| `paths` | Import alias lar (`@/*` → `src/*`) |
| `rootDirs` | Virtual directory — bir nechta papkani bitta dek ko'rsatish |

**Muhim:** `paths` faqat type-checking uchun — JS output da import **o'zgarmaydi**. Bundler (Vite) yoki runtime tool (tsconfig-paths) alohida konfiguratsiya kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
{
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

</details>

---

## Declaration va Source Maps

### Nazariya

| Option | Nima qiladi |
|--------|-------------|
| `declaration` | `.d.ts` fayllar generate qilish |
| `declarationDir` | `.d.ts` fayllar qayerga yoziladi |
| `declarationMap` | `.d.ts.map` — IDE da source ga navigate |
| `emitDeclarationOnly` | Faqat `.d.ts`, `.js` emas (bundler JS yaratganda) |
| `isolatedDeclarations` (TS 5.5+) | Fayl-bo'yicha declaration, explicit types majburiy |
| `sourceMap` | `.js.map` fayllar — debugger da TS source ko'rinadi |
| `inlineSourceMap` | Source map ni JS fayl ichiga inline qo'shadi |
| `stripInternal` | `@internal` JSDoc li declaration larni `.d.ts` dan olib tashlaydi |

Batafsil [Bo'lim 18: Declaration Files](18-declaration-files.md).

---

## Checking Options

### Nazariya

`strict` dan tashqari qo'shimcha tekshiruvlar:

| Option | Nima qiladi | Tavsiya |
|--------|-------------|---------|
| `noUncheckedIndexedAccess` | Index access natijasi `T \| undefined` | ✅ Production da doim |
| `exactOptionalPropertyTypes` | `?` — "yo'q bo'lishi mumkin", `undefined` EMAS | ✅ Yangi loyihalarda |
| `noImplicitOverride` | `override` keyword majburiy (TS 4.3+) | ✅ |
| `noPropertyAccessFromIndexSignature` | Index signature da faqat `obj["key"]`, `obj.key` emas | Ixtiyoriy |
| `noUnusedLocals` | Ishlatilmagan local variable → error | ✅ CI da |
| `noUnusedParameters` | Ishlatilmagan parametr → error | ✅ CI da |
| `noImplicitReturns` | Barcha code path da return bo'lishi kerak | ✅ |
| `noFallthroughCasesInSwitch` | switch da break/return bo'lmasa error | ✅ |

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

| Option | Nima qiladi |
|--------|-------------|
| `noEmit` | JS hech narsa emit qilmaydi (faqat type-check) — bundler JS yaratganda |
| `removeComments` | JS output dan comment larni olib tashlaydi |
| `downlevelIteration` | `for...of`, spread, destructuring ni eski target uchun to'g'ri emit |
| `importHelpers` | `tslib` package dan helper import (`__awaiter`, `__spread`) |
| `useDefineForClassFields` | Class fields ni `Object.defineProperty` bilan emit (ES2022 semantics) |

`noEmit: true` — **Vite/Next.js loyihalarda standart**. Bundler JS yaratadi, tsc faqat type-check qiladi.

---

## Interop Options

### Nazariya

| Option | Nima qiladi |
|--------|-------------|
| `esModuleInterop` | CJS module dan default import mumkin (`import fs from "fs"`) |
| `allowSyntheticDefaultImports` | Type-check da default import ruxsat (emit o'zgarmaydi) |
| `forceConsistentCasingInFileNames` | Fayl nomi case sensitive (macOS da muhim) |
| `allowJs` | `.js` fayllarni TS project ga qo'shish |
| `checkJs` | `.js` fayllarni type-check qilish (JSDoc asosida) |
| `resolveJsonModule` | `.json` fayllarni import qilish |

`esModuleInterop: true` — **har doim yoqing**. Bu CJS/ESM interop muammolarini hal qiladi.

---

## include, exclude, files

### Nazariya

```json
{
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"],
  "files": ["src/global.d.ts"]
}
```

- **`include`** — glob pattern. Default: `**/*` (barcha `.ts`/`.tsx`).
- **`exclude`** — `include` dan chiqarish. Default: `["node_modules", "bower_components"]`.
- **`files`** — aniq fayllar. Glob **ishlamaydi**.

**Muhim:** `exclude` faqat `include` ga ta'sir qiladi. Agar excluded fayl boshqa fayl tomonidan `import` qilinsa — **baribir compile bo'ladi**.

---

## extends — tsconfig Inheritance

### Nazariya

`extends` — boshqa tsconfig dan meros olish. Monorepo da umumiy sozlamalarni base config ga chiqarish uchun.

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

TS 5.0+ da **array extends** — bir nechta config dan meros:

```json
{
  "extends": ["@tsconfig/node20/tsconfig.json", "./tsconfig.base.json"]
}
```

**npm package lar:** `@tsconfig/node20`, `@tsconfig/strictest`, `@tsconfig/recommended` — tayyor preset lar.

---

## Project References

### Nazariya

Project references — monorepo da **bir nechta TS project** ni bog'lash. `composite: true` + `references` bilan har project alohida compile bo'ladi, lekin bir-biriga type-safe reference qiladi.

```json
// packages/core/tsconfig.json
{
  "compilerOptions": { "composite": true, "outDir": "dist", "rootDir": "src" },
  "include": ["src"]
}

// packages/server/tsconfig.json
{
  "compilerOptions": { "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../core" }],
  "include": ["src"]
}

// Root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "packages/core" },
    { "path": "packages/server" }
  ]
}
```

`tsc --build` (yoki `tsc -b`) — dependency tartibida compile qiladi. `incremental: true` bilan faqat o'zgargan project lar qayta compile bo'ladi.

---

## ESLint + TypeScript

### Nazariya

`typescript-eslint` — TypeScript uchun ESLint plugin. **Type-aware linting** — TypeScript ning type ma'lumotidan foydalanib, `no-floating-promises`, `no-unsafe-assignment` kabi qoidalarni enforce qiladi.

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

| Loyiha turi | `module` | `moduleResolution` | Boshqa muhim |
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
import { x } from "./internal/x"; // ✅ — BARIBIR compile bo'ladi!
// exclude faqat "include" ga ta'sir qiladi, import ga emas
```

### 2. `target: "ES5"` bilan `Promise` type yo'q

```json
{ "compilerOptions": { "target": "ES5" } }
// lib default: ["ES5"] — Promise type mavjud emas!
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

### 4. `composite: true` — `declaration: true` va `rootDir` majburiy

```json
// ❌ — composite siz declaration emit yo'q
{ "compilerOptions": { "composite": true } }
// composite avtomatik declaration: true talab qiladi
// Agar rootDir belgilanmagan bo'lsa — tsconfig.json papkasi rootDir bo'ladi
```

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
export const enum Color { Red, Green } // TS OK, esbuild CRASH!
// FIX: isolatedModules: true
```

### ❌ Xato 5: `target` va `lib` ni aralashtirish

```json
// ❌ — target ES5 lekin lib default (ES5) — Promise type yo'q
{ "target": "ES5" }

// ✅ — lib alohida belgilash
{ "target": "ES5", "lib": ["ES2015", "DOM"] }
// Syntax ES5, lekin Promise, Map type lari mavjud (polyfill bilan)
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

`tsconfig.json` — TypeScript proyektning asosiy konfiguratsiya fayli. To'g'ri sozlash type safety, DX, va build performance ga ta'sir qiladi.

**Eng muhim qoidalar:**

1. **`strict: true`** — har doim
2. **`module`/`moduleResolution`** juftligi — Node16/Node16 yoki ESNext/Bundler
3. **`target` vs `lib`** — output syntax vs available types
4. **Bundler** — `noEmit`, `isolatedModules`
5. **Library** — `declaration`, `declarationMap`, `isolatedDeclarations`
6. **Monorepo** — `composite`, `references`, `tsc --build`
7. **`noUncheckedIndexedAccess`** — production da doim
8. **`verbatimModuleSyntax`** — yangi loyihalarda
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
