# Bo'lim 24: Migration va Advanced Tooling

> JavaScript dan TypeScript ga o'tish — bosqichma-bosqich jarayon. `allowJs`, `checkJs`, JSDoc annotations, pragma comments, va oxirida `strict: true`. Bundan tashqari, compilation tezligi, runtime execution, tezkor transpiler lar (SWC, esbuild), va zamonaviy toollar (Biome, Bun, Deno) — developer productivity ga bevosita ta'sir qiladi.

---

## Mundarija

- [JS dan TS ga Migration Strategiyasi](#js-dan-ts-ga-migration-strategiyasi)
- [Strict Mode ga Qadam-baqadam O'tish](#strict-mode-ga-qadam-baqadam-otish)
- [JSDoc Annotations](#jsdoc-annotations)
- [Pragma Comments](#pragma-comments)
- [Performance — tsc Compilation Tezligi](#performance--tsc-compilation-tezligi)
- [Runtime Execution va Transpilation](#runtime-execution-va-transpilation)
- [Zamonaviy Toollar](#zamonaviy-toollar)
- [Type Coverage](#type-coverage)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## JS dan TS ga Migration Strategiyasi

### Nazariya

Migration — JavaScript codebase ni TypeScript ga **bosqichma-bosqich** o'tkazish jarayoni. Birdan o'tkazish (`.js` → `.ts` + `strict: true`) yuzlab xato chiqaradi va PR review imkonsiz bo'ladi.

**Bosqichlar:**

1. **`allowJs: true`** — TypeScript compiler `.js` fayllarni ham kompilyatsiyaga qo'shadi (`.ts` bilan birga). Bu bosqichda type check yo'q, lekin TS toolchain (jump-to-definition, refactor) `.js` da ishlay boshlaydi.

2. **`checkJs: true`** — JS fayllarni ham type-check qiladi. JSDoc annotation, control flow analysis va inference orqali implicit type'lar tekshiriladi. Bu xato'larni `.ts` ga o'tkazishdan oldin topadi.

3. **Leaf-first migration** — dependency tree pastidan (util lar, constant lar) boshlab `.js` → `.ts` ga o'tkazish. Sabab: pastdagi fayl type'lari yuqori fayllarga propagate qilinadi. Yuqoridan boshlash — har o'zgarishda dependencies type'siz qoladi.

4. **`.d.ts` declaration files** — third-party JS library yoki murakkab modul'lar uchun alohida declaration yozish. `.ts` ga ko'chirib bo'lmaydigan kod uchun.

5. **Strict mode bosqichma-bosqich** — `noImplicitAny` → `strictNullChecks` → boshqalar → `strict: true`.

**Kichik PR'lar muhim:** har migration bosqichi alohida commit/PR. Sabab: katta migration PR'larda type xato review qilib bo'lmaydi, regression risk yuqori.

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
// 1-bosqich: tsconfig.json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "outDir": "dist",
    "strict": false
  },
  "include": ["src"]
}
```

```json
// 2-bosqich: checkJs yoqish
{ "compilerOptions": { "allowJs": true, "checkJs": true } }
```

```json
// 3-bosqich: strict bosqichma-bosqich
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

```json
// Oxirgi bosqich: to'liq strict
{ "compilerOptions": { "strict": true } }
```

</details>

---

## Strict Mode ga Qadam-baqadam O'tish

### Nazariya

`strict: true` — `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables`, `alwaysStrict` sub-flag'larini bir vaqtda yoqadi. Katta loyihada birdan yoqish yuzlab xato chiqaradi va developer'lar `@ts-ignore` bilan yopa boshlaydi (texnik qarz ortadi).

**Yechim — sub-flag'larni birma-bir yoqish** (har biri alohida PR):

1. **`noImplicitAny: true`** — implicit `any` parameter/variable'larni xato sifatida belgilaydi. Eng ko'p xato chiqaradi (har annotation'siz parameter), lekin eng muhim — `any` migration progress'ini bloklash uchun.

2. **`strictNullChecks: true`** — `null` va `undefined` ni har type ga assign qilish taqiqlanadi. `T | null | undefined` explicit yozilishi kerak. Null safety bugs (TypeError: cannot read property of undefined) ni kompilyatsiya vaqtida tutadi.

3. **`strictFunctionTypes: true`** — function parameter contravariance ni majbur qiladi (method shorthand'dan tashqari). [Bo'lim 25](25-type-compatibility.md) da batafsil.

4. **`strictPropertyInitialization: true`** — class property'lar constructor'da yoki declaration'da initialize qilinishi kerak (`strictNullChecks` ga bog'liq).

5. **`useUnknownInCatchVariables: true`** — `catch (e)` da `e` ning type'i `any` o'rniga `unknown`. Type narrowing majburiy bo'ladi.

6. **Oxirida `strict: true`** — barcha sub-flag'larni `strict` bilan almashtirish. Yangi qo'shilgan sub-flag'lar avtomatik yoqilishi uchun.

**Override mexanizmi:** `strict: false` + `noImplicitAny: true` — sub-flag'lar `strict` ni override qiladi (TypeScript precedence qoidasi). Bu bosqichma-bosqich yoqish imkonini beradi.

---

## JSDoc Annotations

### Nazariya

JSDoc — `.js` fayllarni `.ts` ga o'tkazmasdan type safety berish usuli. `checkJs: true` bilan TypeScript compiler JSDoc annotation'larni o'qiydi va worth qiladi: `@typedef` interface'ga, `@param`/`@returns` function signature'ga aylanadi.

**Mexanizm:** TypeScript parser JSDoc comment'larni alohida grammar bilan parse qiladi (TSDoc subset). Type checker `.ts` faylda yozilgan annotation bilan teng huquqli ishlaydi — inference, narrowing, generic'lar — barchasi mavjud.

**Qachon foydali:**
- Vendor JS library'lariga lokal type annotation
- Migration'ning oraliq bosqichi (`.js` saqlanib, type safety qo'shiladi)
- Build step yo'q loyihalar (browser uchun raw `.js`)

**Cheklov:** JSDoc syntax `.ts` syntax'idan uzunroq va imkoniyatlari kamroq — generic constraint'lar, conditional type'lar JSDoc'da murakkab yoziladi. Long-term yechim `.ts` ga o'tish.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// @ts-check

/**
 * @typedef {Object} User
 * @property {number} id
 * @property {string} name
 * @property {string} [email] — optional
 */

/**
 * @param {User[]} users
 * @param {string} name
 * @returns {User | undefined}
 */
function findUser(users, name) {
  return users.find(u => u.name === name);
}

/** @type {User[]} */
const users = [
  { id: 1, name: "Ali" },
  { id: 2, name: "Vali", email: "v@test.com" },
];

const result = findUser(users, "Ali");
// result: User | undefined — ✅ type-safe!
```

</details>

---

## Pragma Comments

### Nazariya

Pragma comment'lar — fayl yoki qator darajasida TypeScript checker xulq'ini o'zgartirish.

| Pragma | Nima qiladi | Qachon |
|--------|-------------|--------|
| `// @ts-check` | JS faylda type checking yoqadi (lokal, `checkJs` siz) | `.js` faylda opt-in type check |
| `// @ts-nocheck` | Butun faylda type checking o'chiradi | `checkJs: true` da legacy fayl |
| `// @ts-ignore` | Keyingi qator xatosini yashiradi (xato bormi-yo'qmi farqi yo'q) | Sticky suppress (tavsiya etilmaydi) |
| `// @ts-expect-error` | Keyingi qatorda xato **bo'lishi kerak** (TS 3.9+) | Kelajakda tuzatilishi kerak xato |

**`@ts-ignore` vs `@ts-expect-error` farqi:**

- `@ts-ignore` — xato bor-yo'qligini tekshirmaydi. Agar kod keyin tuzatilsa, `@ts-ignore` o'zi qoladi (silent dead code).
- `@ts-expect-error` — xato KUTILADI. Agar xato yo'q bo'lsa (kod tuzatilsa), compiler "Unused '@ts-expect-error'" xato beradi. Bu suppress'ni avtomatik tozalaydi.

**Qoida:** `@ts-expect-error` doim afzal. Sticky `@ts-ignore` faqat third-party library bug'lari uchun (suppress'ni tozalashga sabab yo'q hollarda).

---

## Performance — tsc Compilation Tezligi

### Nazariya

`tsc` compilation katta loyihalarda sekin bo'ladi — type checker har fayl uchun butun program graph'ini analiz qiladi (cross-file type inference, structural compatibility). Optimizatsiya'lar — caching, parallelism, va checker yukini kamaytirish.

| Optimizatsiya | Mexanizm |
|--------------|----------|
| `incremental: true` | Build state'ni `.tsbuildinfo` cache faylga saqlaydi. Keyingi build'da faqat o'zgargan fayl va uning dependents'lari qayta tekshiriladi |
| `skipLibCheck: true` | `node_modules/**/*.d.ts` fayllarini type-check qilmaydi (faqat sintaksis tekshirish). Library type'larida xato bo'lsa ham build muvaffaqiyatli |
| Project References | Monorepo'da har package alohida `tsconfig` + `composite: true` bilan mustaqil build va cache. `tsc --build` topological order'da kompilyatsiya qiladi |
| `isolatedModules` | Har fayl mustaqil transpile bo'lishi mumkinligini majbur qiladi. SWC/esbuild uchun kerak (ular cross-file analysis qilmaydi) |
| `isolatedDeclarations` (TS 5.5+) | `.d.ts` emit uchun har faylda explicit type annotation majbur. Parallel declaration generation imkonini beradi |

**Profiling:**

```bash
tsc --generateTrace ./trace --incremental false
# trace/ ichida: trace.json + types.json
# Chrome chrome://tracing yoki Perfetto UI'da: trace.json yuklash
# Vaqt sarflagan checker bosqichlari (checkSourceFile, getTypeOfNode) ko'rinadi
```

`types.json` — eng katta type'lar (memory'da). `trace.json` — vaqt taqsimoti.

---

## Runtime Execution va Transpilation

### Nazariya

TypeScript kod runtime'da ikki xil ishlatiladi: **runtime execution** (dev/scripting) va **transpilation** (production build). Tool tanlash trade-off: tezlik vs type safety.

| Tool | Mexanizm | Type check |
|------|----------|------------|
| **`tsc`** | TypeScript compiler — type check + JS emit | Bor (to'liq) |
| **`tsx`** | esbuild-based runtime wrapper (Node loader). TS faylni direct `node` ga uzatadi | Yo'q |
| **`ts-node`** | TypeScript compiler API'sini Node `require` hook bilan ulashtiradi. Eski (default `--swc` flag yo'q) | Bor (default) |
| **SWC** | Rust-based transpiler. AST-based o'zgartirish, cross-file analysis yo'q | Yo'q (faqat parse) |
| **esbuild** | Go-based bundler/transpiler. SWC bilan o'xshash, lekin bundle ham qiladi | Yo'q |
| **Node.js `--experimental-strip-types`** | Node 22.6+ da experimental: type annotation'larni strip qiladi (transform yo'q — `enum`, decorators ishlamaydi). Node 23.6+ da default yoqilgan | Yo'q |
| **Node.js `--experimental-transform-types`** | Node 22.7+ — `enum`, `namespace`, parameter property'larni transform qiladi | Yo'q |

**`tsx` vs `ts-node` farqi:** `tsx` esbuild bilan transpile qiladi (10-100x tezroq), lekin type check yo'q. `ts-node` default'da TypeScript compiler ishlatadi (sekin, lekin type check bor). Dev'da `tsx` afzal — type check'ni alohida `tsc --noEmit` orqali bajarish.

**Pipeline:** Production'da build (SWC/esbuild — tez transpile) va type check (`tsc --noEmit` — to'liq tekshirish) **alohida step**'lar. Sabab: bundler'da type check'ni qo'shish CI vaqtini ko'paytiradi va parallel run imkoniyatini bloklaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "swc src -d dist --strip-leading-paths",
    "type-check": "tsc --noEmit",
    "ci": "npm run type-check && npm run build && npm run test"
  }
}
```

```bash
# SWC bilan build (tez)
npx swc src -d dist

# esbuild bilan bundle
npx esbuild src/index.ts --bundle --outfile=dist/index.js --platform=node

# tsx bilan development
npx tsx watch src/index.ts
```

</details>

---

## Zamonaviy Toollar

### Nazariya

| Tool | O'rniga | Afzalligi |
|------|---------|-----------|
| **Biome** | ESLint + Prettier | Rust-based, tez, bitta tool — lint + format bitta config'da |
| **Bun** | Node.js + npm | Native TS runtime, JavaScriptCore-based, built-in bundler/test runner |
| **Deno** | Node.js | Native TS runtime, secure-by-default (permission model: `--allow-read`, `--allow-net`), URL-based ESM import |

**`Deno` secure-by-default:** Default'da fayl tizimi, network, env variable'larga ruxsat yo'q. Har resurs uchun flag kerak (`--allow-read=/path`, `--allow-net=api.example.com`). Bu supply-chain attack'larni cheklaydi.

**`Bun` performance:** JavaScriptCore (Safari engine) + Zig native code. TS transpile esbuild bilan, lekin runtime'da Node.js dan tez (file I/O, HTTP).

**`Biome` cheklov:** ESLint plugin ekosistemasi mavjud emas (custom rule yoki react-specific lint kerak bo'lsa cheklov). Format va asosiy lint uchun yetadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```bash
# Biome — lint + format
npx @biomejs/biome check src/
npx @biomejs/biome check --write src/  # auto-fix

# Bun — native TS
bun run src/index.ts  # tsx kerak emas
bun test              # vitest kerak emas

# Deno — native TS
deno run src/index.ts
deno test
```

</details>

---

## Type Coverage

### Nazariya

`type-coverage` — loyihadagi **non-`any`** type'larning foizini o'lchaydi. TypeScript compiler API'sini ishlatib har identifier'ning type'ini tekshiradi: agar `any` bo'lsa — uncovered, aks holda covered.

```bash
npx type-coverage --at-least 90 --strict
# 90% dan kam bo'lsa — exit code 1 (CI da fail)
```

**`--strict` flag:** generic'larda inferred `any` (masalan, `JSON.parse(x)` → `any`) ham uncovered hisoblanadi. Strict'siz — faqat explicit `any` annotation.

**Migration use case:** har PR'da type coverage'ni o'lchash. Threshold (90%, 95%) belgilanadi — kamaysa CI fail. Bu `any` ning regress'ini oldini oladi.

**Cheklov:** `type-coverage` static — runtime behavior'ni tekshirmaydi. `as` casting (`x as User`) covered hisoblanadi, lekin runtime'da xato bo'lishi mumkin. `as unknown as T` pattern'i — type safety bypass — coverage tomonidan ushlanmaydi.

---

## Edge Cases va Gotchas

### 1. `checkJs` + lokal JS — kutilmagan xatolar

`checkJs: true` yoqilganda `node_modules` default'da exclude qilinadi (`skipLibCheck` ham yordam beradi), lekin loyiha JS fayllarida implicit `any`, undefined property access kabi xatolar massa'da chiqishi mumkin.

```javascript
// src/utils.js — checkJs: true ostida
function calculate(a, b) {       // ❌ implicit any (noImplicitAny bilan)
  return a.value + b.value;       // ❌ Property 'value' does not exist
}

// Yechim 1: vaqtinchalik suppress
// @ts-nocheck
function calculate(a, b) { return a.value + b.value; }

// Yechim 2: JSDoc bilan annotation
/** @param {{value: number}} a @param {{value: number}} b */
function calculate(a, b) { return a.value + b.value; }
```

### 2. `@ts-ignore` vs `@ts-expect-error` — muhim farq

```typescript
// @ts-ignore — doim xatoni yashiradi (kerak bo'lmasa ham)
// @ts-expect-error — xato BO'LMASA o'zi xato beradi

// @ts-expect-error
const x: number = "hello"; // Xato bor → suppress ✅

// @ts-expect-error
const y: number = 42; // Xato YO'Q → "Unused @ts-expect-error" ❌
// Bu yaxshi — keraksiz suppress tozalanadi
```

### 3. SWC/esbuild — `const enum` cross-file va namespace merging ishlamaydi

```typescript
// shared.ts
export const enum Status { Active, Inactive }

// app.ts
import { Status } from "./shared";
console.log(Status.Active);
// tsc: 0 (inline qilinadi)
// SWC/esbuild: Status.Active — Status object qaytaradi (ishlamaydi)
```

`const enum` inlining cross-file static analysis talab qiladi — SWC/esbuild har faylni izolyatsiya qilib transpile qiladi (`isolatedModules` modeli). Namespace merging ham xuddi shu sababdan ishlamaydi.

**Yechim:** `isolatedModules: true` yoqing, `const enum` o'rniga oddiy `enum` yoki `as const` object pattern:

```typescript
export const Status = { Active: 0, Inactive: 1 } as const;
export type Status = (typeof Status)[keyof typeof Status];
```

### 4. `allowJs` + `declaration` — `.js` dan `.d.ts` generate bo'lmaydi

```json
// allowJs: true + declaration: true
// .ts fayllar uchun .d.ts generate bo'ladi
// .js fayllar uchun — ❌ GENERATE BO'LMAYDI
// Yechim: .js → .ts ga o'tkazish yoki qo'lda .d.ts yozish
```

### 5. `type-coverage` — generic `any` ni tutmaydi

```typescript
function parse<T>(json: string): T {
  return JSON.parse(json); // T bu yerda aslida any
  // type-coverage buni "covered" deb hisoblaydi
}
// Yechim: --strict flag bilan type-coverage
```

---

## Common Mistakes

### ❌ Xato 1: Birdan `strict: true` yoqish

Katta loyihada yuzlab xato chiqadi → developer lar `@ts-ignore` bilan yopadi. **Bosqichma-bosqich** yoqish yaxshiroq.

### ❌ Xato 2: `@ts-ignore` haddan tashqari ishlatish

```typescript
// ❌ — xatoni yashiradi, keyin esdan chiqadi
// @ts-ignore
const port: number = "8080";

// ✅ — @ts-expect-error + ticket reference
// @ts-expect-error TODO: fix after API type migration (ticket #1234)
const port: number = "8080";
// Kod tuzatilsa, @ts-expect-error o'zi xato beradi — suppress tozalanadi
```

### ❌ Xato 3: SWC/esbuild bilan `const enum` ishlatish

`const enum` cross-file inline talab qiladi — SWC/esbuild buni qilolmaydi. Oddiy enum yoki union type ishlatish.

### ❌ Xato 4: `tsc` bilan build + type-check birgalikda

```json
// ❌ — sekin
{ "scripts": { "build": "tsc" } }

// ✅ — tez: build va type-check alohida
{
  "scripts": {
    "build": "swc src -d dist",
    "type-check": "tsc --noEmit"
  }
}
```

### ❌ Xato 5: Type coverage ni track qilmaslik

Migration paytida `any` count o'sib boradi. `type-coverage` bilan CI da threshold qo'ymaslik → regress.

---

## Amaliy Mashqlar

### Mashq 1: Migration tsconfig (Oson)

**Savol:** 1-bosqich tsconfig — `allowJs`, `checkJs: false`, `strict: false`.

<details>
<summary>Javob</summary>

```json
{
  "compilerOptions": {
    "target": "ES2020", "module": "Node16", "moduleResolution": "Node16",
    "allowJs": true, "checkJs": false, "strict": false,
    "outDir": "dist", "esModuleInterop": true, "skipLibCheck": true
  },
  "include": ["src"]
}
```

</details>

---

### Mashq 2: JSDoc Type Safety (O'rta)

**Savol:** JS faylni to'liq JSDoc bilan type-safe qiling.

<details>
<summary>Javob</summary>

```javascript
// @ts-check
/** @typedef {{ id: number; name: string; price: number; inStock: boolean }} Product */
/** @typedef {"price" | "name" | "id"} SortField */

/**
 * @param {Product[]} products
 * @param {{ minPrice?: number; maxPrice?: number; sortBy?: SortField }} [options]
 * @returns {Product[]}
 */
function filterProducts(products, options = {}) {
  const { minPrice = 0, maxPrice = Infinity, sortBy = "name" } = options;
  return products
    .filter(p => p.price >= minPrice && p.price <= maxPrice)
    .sort((a, b) => sortBy === "price" ? a.price - b.price : a.name.localeCompare(b.name));
}
```

</details>

---

### Mashq 3: Zamonaviy Toolchain (O'rta)

**Savol:** SWC build, Biome lint, tsx dev, vitest test, type-coverage — package.json scripts.

<details>
<summary>Javob</summary>

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "swc src -d dist --strip-leading-paths",
    "type-check": "tsc --noEmit",
    "lint": "biome check src/",
    "test": "vitest run",
    "type-coverage": "type-coverage --at-least 90 --strict",
    "ci": "npm run type-check && npm run type-coverage && npm run lint && npm run test && npm run build"
  }
}
```

</details>

---

### Mashq 4: Project References (Qiyin)

**Savol:** 3 package monorepo — types, core, api.

<details>
<summary>Javob</summary>

```jsonc
// packages/types/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

```jsonc
// packages/core/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "references": [{ "path": "../types" }],
  "include": ["src"]
}
```

```jsonc
// packages/api/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "references": [
    { "path": "../types" },
    { "path": "../core" }
  ],
  "include": ["src"]
}
```

```jsonc
// Root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "packages/types" },
    { "path": "packages/core" },
    { "path": "packages/api" }
  ]
}
```

```bash
# Build (topological order — types → core → api)
tsc --build

# Watch mode
tsc --build --watch
```

</details>

---

### Mashq 5: Strict Migration Plan (Oson)

**Savol:** `strict: true` ga 5 bosqichlik migration plan yozing.

<details>
<summary>Javob</summary>

1. `noImplicitAny: true` → barcha `any` larni explicit type bilan almashtirish
2. `strictNullChecks: true` → null/undefined tekshiruvlari qo'shish
3. `strictFunctionTypes: true` → function parameter type lar tuzatish
4. `strictPropertyInitialization: true` → class property lar initialize
5. `strict: true` → barcha sub-flag larni `strict` bilan almashtirish

</details>

---

## Xulosa

**Migration:** `allowJs` → `checkJs` → leaf-first `.ts` → strict bosqichma-bosqich.

**Performance:** `incremental`, `skipLibCheck`, project references, `isolatedModules`/`isolatedDeclarations`.

**Tooling:** Build = SWC/esbuild (tez), Type check = tsc, Dev = tsx, Lint = Biome, Test = vitest.

**Zamonaviy:** Node.js native TS strip (22.6+), Bun, Deno — native TS runtime.

**Type coverage:** `type-coverage` bilan migration progress o'lchash, CI da threshold.

---

**Keyingi bo'lim:** [25-type-compatibility.md](25-type-compatibility.md) — Structural typing, covariance, contravariance, variance annotations, va type compatibility qoidalari.
