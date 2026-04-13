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

Migration bosqichlari:

1. **`allowJs: true`** — `.js` va `.ts` fayllar birgalikda ishlaydi
2. **`checkJs: true`** — JS fayllarni ham type-check qiladi (JSDoc asosida)
3. **Leaf-first** — dependency tree pastidan (util lar) boshlab, `.js` → `.ts` ga o'tkazish
4. **Type declaration** — murakkab type lar uchun `.d.ts` fayllar yozish
5. **Strict** — oxirida `strict: true` ga o'tish

**Muhim:** Kichik PR lar — har bir migration bosqichi alohida commit/PR.

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

`strict: true` ni birdan yoqish katta loyihada yuzlab xato chiqaradi. Yechim — strict sub-flag larni **birma-bir** yoqish:

1. `noImplicitAny` — eng ko'p xato, lekin eng muhim
2. `strictNullChecks` — null safety
3. `strictFunctionTypes` — function parameter safety
4. `strictPropertyInitialization` — class property lar
5. `useUnknownInCatchVariables` — catch block safety
6. Oxirida `strict: true` — barcha sub-flag lar o'rniga

Har bir bosqichda xatolarni tuzatib, alohida PR sifatida merge qilish.

---

## JSDoc Annotations

### Nazariya

JSDoc — `.js` fayllarni `.ts` ga o'tkazmasdan type safety berish usuli. `checkJs: true` bilan TypeScript JSDoc annotation larni o'qib type-check qiladi.

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

| Pragma | Nima qiladi |
|--------|-------------|
| `// @ts-check` | JS faylda type checking yoqish |
| `// @ts-nocheck` | Butun faylda type checking o'chirish |
| `// @ts-ignore` | Keyingi qator xatosini yashirish (eski) |
| `// @ts-expect-error` | Keyingi qatorda xato **bo'lishi kerak** (TS 3.9+) |

**Qoida:** `@ts-ignore` o'rniga `@ts-expect-error` ishlatish — agar xato keyin tuzatilsa, `@ts-expect-error` **o'zi xato beradi** (keraksiz suppress).

---

## Performance — tsc Compilation Tezligi

### Nazariya

Katta loyihalarda `tsc` sekin bo'lishi mumkin. Asosiy optimizatsiyalar:

| Optimizatsiya | Ta'siri |
|--------------|---------|
| `incremental: true` | Faqat o'zgargan fayllar qayta compile (`.tsbuildinfo` cache) |
| `skipLibCheck: true` | `.d.ts` fayllarni tekshirmaydi |
| Project References | Monorepo da mustaqil build + cache |
| `isolatedModules` | Fayl-bo'yicha transpilation (SWC/esbuild uchun) |
| `isolatedDeclarations` (TS 5.5+) | Fayl-bo'yicha declaration emit |

**Profiling:** `tsc --generateTrace ./trace` → Chrome DevTools da analyze.

---

## Runtime Execution va Transpilation

### Nazariya

| Tool | Nima qiladi | Tezlik |
|------|-------------|--------|
| **`tsc`** | Type check + JS emit | Baseline |
| **`tsx`** | TS faylni to'g'ridan-to'g'ri ishga tushirish (dev) | Tez (esbuild-based) |
| **`ts-node`** | TS runtime (eski) | O'rta |
| **SWC** | Rust-based transpiler, type check YO'Q | Juda tez |
| **esbuild** | Go-based bundler/transpiler | Juda tez |
| **Node.js `--experimental-strip-types`** (22.6+) | Native TS type strip | Tez |

**Qoida:** Build = SWC/esbuild (tez), Type check = tsc (to'liq) — **alohida pipeline**.

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
| **Biome** | ESLint + Prettier | Rust-based, tez, bitta tool |
| **Bun** | Node.js + npm | Native TS, tez runtime, bundler |
| **Deno** | Node.js | Native TS, xavfsiz, URL import |

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

`type-coverage` — loyihadagi `any` type foizini o'lchaydi. Migration progressini track qilish uchun.

```bash
npx type-coverage --at-least 90 --strict
# 90% dan kam bo'lsa — fail (CI da threshold)
```

CI da integratsiya — har PR da type coverage tekshiriladi, kamaysa — block.

---

## Edge Cases va Gotchas

### 1. `checkJs` + third-party JS — kutilmagan xatolar

```javascript
// node_modules dagi JS fayl checkJs ta'sirida emas (default exclude)
// Lekin o'z JS fayllaringizda kutilmagan xatolar chiqishi mumkin
// @ts-nocheck bilan vaqtinchalik o'chirish mumkin
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

### 3. SWC/esbuild — enum va namespace ni to'g'ri handle qilmaydi

```typescript
// SWC/esbuild bilan const enum ISHLAMAYDI (isolatedModules cheklovi)
// Namespace merging ham ISHLAMAYDI
// Yechim: isolatedModules: true yoqib, bu pattern larni ishlatmang
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
const x: number = "hello";

// ✅ — @ts-expect-error + ticket
// @ts-expect-error TODO: fix in JIRA-123
const x: number = "hello";
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

```json
// packages/types/tsconfig.json
{ "compilerOptions": { "composite": true, "declaration": true, "outDir": "dist", "rootDir": "src" }, "include": ["src"] }

// packages/core/tsconfig.json
{ "compilerOptions": { "composite": true, "declaration": true, "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../types" }], "include": ["src"] }

// packages/api/tsconfig.json
{ "compilerOptions": { "composite": true, "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../types" }, { "path": "../core" }], "include": ["src"] }

// Root: { "files": [], "references": [{ "path": "packages/types" }, { "path": "packages/core" }, { "path": "packages/api" }] }
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
