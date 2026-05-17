# Interview: Migration va Tooling

> JS → TS migration strategiyasi (gradual vs big bang, leaf-first), `allowJs`/`checkJs`, JSDoc annotations, `@ts-ignore` vs `@ts-expect-error`, SWC/esbuild/tsx, Node.js `--experimental-strip-types`, `isolatedModules` vs `isolatedDeclarations`, `type-coverage` bo'yicha interview savollari. Har javob mustaqil — kontekst javob ichida.

---

## Mundarija

- **Nazariy** (Savol 1-7, 11) — gradual vs big bang, `allowJs`/`checkJs`, leaf-first, pragma'lar, SWC/esbuild semantics, tsx/ts-node/Node strip-types, JSDoc, `isolatedModules` vs `isolatedDeclarations`
- **Coding va Output** (Savol 8, 9) — tsconfig audit, `const enum` + SWC bug
- **Tooling deep dive** (Savol 10, 12, 13) — `type-coverage` CI, 5-bosqich strict migration plan, SWC + NestJS decorator metadata
- [Xulosa](#xulosa)

---

## Savol 1: JS → TS migration — "big bang" vs "gradual"? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Gradual (bosqichma-bosqich) — yagona to'g'ri yondashuv. "Big bang" (bir PR'da hammasi) katta loyihada 500+ fayl, review imkonsiz, merge conflict, rollback xavfli. Har bosqich alohida PR.

### To'liq tushuntirish

Gradual migration bosqichlari:

1. **`allowJs: true`** — `.js` va `.ts` birgalikda. Hech qaysi `.js` o'zgarmaydi, ammo TypeScript tooling (IDE intellisense) ishlaydi.
2. **`checkJs: true`** (yoki per-file `// @ts-check`) — `.js` fayllar TS tomonidan type-check qilinadi (inference + JSDoc).
3. **Leaf-first** `.js` → `.ts` migration — dependency tree pastidan boshlab.
4. **Yangi kod faqat `.ts`** — convention qo'yish.
5. **`strict` sub-flag'lar birma-bir**: `noImplicitAny` → `strictNullChecks` → ... → `strict: true`.
6. **Eski fayllarni ketma-ket** migrate qilish (har PR 5-10 fayl).
7. **`allowJs: false`** — to'liq TS loyiha.

Big bang muammolari:

- 500+ fayl, har biri 10+ TS xatosi → 5000+ xato. Tuzatish vaqti haftalar.
- Single massive PR — review imkonsiz, har commit conflict.
- Rollback xavf — bir bug butun build'ni buzadi.
- Team momentum yo'qoladi — boshqa work block bo'ladi.

Gradual afzalligi:

- Har PR review qilinadi (CI green).
- Boshqa work blok bo'lmaydi.
- Bosqichma-bosqich learning (team TS'ni o'rganadi).
- Rollback bitta PR scope'ida.

### Kod misol

```json
// === Bosqich 1: allowJs ===
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "outDir": "dist",
    "strict": false,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "module": "Node16",
    "moduleResolution": "Node16"
  },
  "include": ["src"]
}
```

```json
// === Bosqich 2: checkJs yoqish ===
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,         // ← endi JS ham tekshiriladi
    "strict": false,
    "noImplicitAny": false
  }
}
```

```javascript
// === Bosqich 3: per-file @ts-check (opt-in) ===
// src/utils/format.js
// @ts-check

/** @param {Date} date */
function formatDate(date) {
  return date.toISOString();
}
```

```json
// === Bosqich 4: strict sub-flag'lar birma-bir ===
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "strict": false,
    "noImplicitAny": true,        // ← birinchi
    "strictNullChecks": true      // ← keyin
  }
}
```

```json
// === Bosqich 5: to'liq strict + .ts ===
{
  "compilerOptions": {
    "allowJs": false,    // faqat .ts
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Edge Cases

- **Third-party untyped library**: `@types/library` yo'q bo'lsa — `declare module "library";` quick fix, keyin community types yoki manual `.d.ts`.
- **Test fayllar**: `*.test.js` ham migrate kerak. Jest `ts-jest` yoki Vitest (native TS).
- **Generated kod (Prisma, GraphQL)**: codegen'dan `.ts` generate qilish — manual o'zgartirish kerak emas.
- **Mixed `.js` va `.ts` import**: `import { x } from "./foo"` `foo.ts` ni topadi (extension yo'q), `foo.js` ham. `moduleResolution: "Node16"` da explicit extension majburiy.

### Follow-up savollar

1. "Migration uchun qancha vaqt kerak (500+ fayl loyiha)?" — Aniq raqam aytib bo'lmaydi: team size, complexity, codebase architecture, parallel feature work — barchasi ta'sir qiladi. Gradual migration odatda haftalardan oylargacha cho'ziladi. Big bang katta loyihada amaliy emas (rollback risk).
2. "Migration paytida yangi feature'lar yozsa bo'ladimi?" — Ha. `.ts` faqat yangi kod uchun. Parallel work — feature team va migration team.

</details>

---

## Savol 2: `allowJs` va `checkJs` farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`allowJs: true` — TS proyektga `.js` fayllarni import/include qilishga ruxsat (type-check qilinmaydi). `checkJs: true` — `.js` fayllar ham type-check qilinadi (inference + JSDoc orqali). Per-file `// @ts-check` opt-in yoki `// @ts-nocheck` opt-out.

### To'liq tushuntirish

**`allowJs: true`**:

- `.js` fayllar TS compilation'ga qo'shiladi (emit, `outDir`'ga ko'chiriladi).
- `.ts` fayllardan `.js` import qilish mumkin.
- `.js` fayllar **type-check qilinmaydi** — barcha qiymat `any`.
- Migration boshlanishi uchun zarur (gradual).

**`checkJs: true`**:

- `.js` fayllar ham type-check qilinadi.
- Type inference + JSDoc annotation'lar o'qiladi.
- `.js` faylda type error → CI fail.
- Strict mode: requires `allowJs: true`.

**Per-file pragma'lar**:

- `// @ts-check` — shu faylni tekshir (opt-in, `checkJs: false` umumiy bo'lsa). Migration boshida ayrim fayllarni tayyorlash uchun.
- `// @ts-nocheck` — shu faylni o'tkazib yubor (`checkJs: true` umumiy bo'lsa). Legacy fayl tekshirilmasin.

Strategiya:

- `checkJs: false` + tayyor `.js` fayllarga `// @ts-check` — "opt-in" gradual.
- `checkJs: true` + legacy fayllarga `// @ts-nocheck` — "opt-out" gradual.

### Kod misol

```json
// === Strategiya 1: Opt-in (checkJs: false) ===
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false
  }
}
```

```javascript
// src/utils/legacy.js — type-check qilinmaydi
function calculate(x) {
  return x * 2;
}

// src/utils/format.js
// @ts-check — shu fayl opt-in
/** @param {string} name */
function greet(name) {
  return `Hello, ${name}`;
}

greet(42); // ❌ TS error (faqat shu faylda, @ts-check tufayli)
```

```json
// === Strategiya 2: Opt-out (checkJs: true) ===
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true
  }
}
```

```javascript
// src/utils/legacy.js
// @ts-nocheck — bu fayl tekshirilmasin (migration kelajakda)

function calculate(x) {
  return x * 2; // har qanday xato yashirin
}

// src/utils/format.js — checkJs tufayli avtomatik tekshiriladi
/** @param {string} name */
function greet(name) {
  return `Hello, ${name}`;
}

// greet(42); // ❌ TS error
```

```json
// === Production: faqat .ts ===
{
  "compilerOptions": {
    "allowJs": false,
    "checkJs": false,
    "strict": true
  }
}
```

### Edge Cases

- **`@ts-nocheck` butun fayl uchun**: faqat birinchi qatorda effective. Comment kabi joylashtirish kerak.
- **`@ts-check` `.ts` faylda mantiqsiz** — `.ts` har doim tekshiriladi.
- **`@ts-expect-error` + `@ts-check` kombinatsiyasi**: `.js` faylda ham ishlaydi (`checkJs: true` yoki `@ts-check` bilan).
- **`node_modules` `.js`**: `checkJs: true` umuman `node_modules`'ni tekshirmaydi (default `exclude`). `skipLibCheck: true` `.d.ts`'ni ham skip.

### Follow-up savollar

1. "`allowJs: false` lekin `.js` fayl import qilsam?" — TS error: cannot find module. Migration to'liq tugaganligini bildiradi.
2. "JSDoc'ni `.ts` faylda yozsa bo'ladimi?" — Ha, lekin redundant. `.ts` native type syntax afzal (`function add(a: number, b: number): number`).

</details>

---

## Savol 3: "Leaf-first" migration — nima uchun? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Leaf-first — dependency tree'ning **leaf** fayllaridan (no dependencies) boshlab migrate qilish. Sabab: leaf'lar mustaqil, boshqa fayllarni buzmaydi. Entry point'dan boshlash cascade xatolar keltiradi.

### To'liq tushuntirish

Dependency tree:

```
app.js (entry — barchasini import qiladi)
  ├── routes/userRoutes.js
  │     └── services/user.js
  │           └── models/user.js (leaf — import yo'q)
  ├── routes/postRoutes.js
  └── utils/helpers.js (leaf)
```

Leaf — boshqa local fayllarni `import` qilmaydigan modul.

Leaf-first sabablari:

1. **Mustaqillik**: leaf'ni `.ts`'ga o'tkazganda boshqa fayllar ta'sirlanmaydi (faqat `.js` `.ts`'dan import qiladi, `allowJs` bilan ishlaydi).
2. **Kichik PR**: bitta leaf fayl + uning test'lari. Review oson.
3. **Type "yuqoriga oqadi"**: leaf type qo'yilganda — parent (`services/user.js`) endi typed leaf'ni ko'radi (inference yaxshilanadi).
4. **Cascade xato yo'q**: entry'dan boshlasangiz, har import qilingan `.js`'da `any` cascade'da entry'ga yetib boradi.

Tartib:

1. `models/user.js` → `.ts` (leaf, no imports).
2. `utils/helpers.js` → `.ts` (leaf).
3. `services/user.js` → `.ts` (endi user model typed).
4. `routes/userRoutes.js` → `.ts`.
5. `app.js` → `.ts` (eng oxiri).

### Kod misol

```javascript
// === Bosqich 0: hammasi .js ===

// models/user.js (leaf)
function User(id, name) {
  this.id = id;
  this.name = name;
}
module.exports = User;

// services/user.js
const User = require("../models/user");
function getUserById(id) {
  return new User(id, "Ali");
}
module.exports = { getUserById };

// app.js (entry)
const { getUserById } = require("./services/user");
console.log(getUserById(1));
```

```typescript
// === Bosqich 1: models/user.ts (leaf birinchi) ===
export interface User {
  id: number;
  name: string;
}

export function createUser(id: number, name: string): User {
  return { id, name };
}
```

```javascript
// services/user.js — hali .js, lekin typed User'ni "biladi"
// @ts-check
const { createUser } = require("../models/user");

/** @param {number} id */
function getUserById(id) {
  return createUser(id, "Ali");
  // VSCode: User type inferred (TS .ts faylni ko'radi)
}

module.exports = { getUserById };
```

```typescript
// === Bosqich 2: services/user.ts ===
import { createUser, User } from "../models/user";

export function getUserById(id: number): User {
  return createUser(id, "Ali");
}
```

```typescript
// === Bosqich 3: app.ts (oxirgi) ===
import { getUserById } from "./services/user";
console.log(getUserById(1));
```

### Edge Cases

- **Circular dependency**: A → B → A — leaf yo'q. Refactoring kerak (shared module ajratish) yoki har ikkalasini bir vaqtda migrate.
- **Dynamic import**: `import("./mod")` runtime'da resolve — dependency tree'da yashirin. `grep -r "import("` bilan topish.
- **Test fayllar leaf'larmi?** — Odatda yes (test fayl test qilinayotgan modul'ni import qiladi, boshqalar emas). Test bilan birga migrate.
- **Generated kod**: `prisma generate`, GraphQL codegen — boshidan `.ts` generate qilish, manual migration kerak emas.

### Follow-up savollar

1. "Dependency tree'ni qanday topish?" — `madge` (Node.js), `dpdm` paket'lari — visualizer. `madge --circular src/` circular topadi.
2. "Faqat leaf migrate qilib boshqasini qoldirsa bo'ladimi?" — Vaqtinchalik OK, lekin barcha kodni typed qilmaguncha `any` cascade'lari saqlanadi. Roadmap qo'yish.

</details>

---

## Savol 4: `@ts-ignore` va `@ts-expect-error` farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`@ts-ignore` — keyingi qator xatosini yashiradi (xato bo'lmasa ham). `@ts-expect-error` — xato bo'lishini kutadi, xato bo'lmasa o'zi yangi xato beradi ("Unused @ts-expect-error directive"). `@ts-expect-error` afzal — code tuzatilganda eslatadi.

### To'liq tushuntirish

`@ts-ignore` muammosi:

- TS xatoni silent yashiradi. Vaqt o'tib kod tuzatiladi (xato yo'qoladi) — `@ts-ignore` qoldiriladi (dead suppress).
- Code review'da `@ts-ignore` ko'rilsa — qaysi xato uchun ekanligi noma'lum.
- Cleanup'ni hech kim qilmaydi.

`@ts-expect-error` (TS 3.9+) yechimi:

- Xato bo'lsa — yashiradi (xuddi `@ts-ignore`).
- Xato yo'q bo'lsa — yangi error: "Unused '@ts-expect-error' directive".
- Code tuzatilganda — TS o'zi eslatadi, suppress'ni olib tashlash kerak.

Convention: har `@ts-expect-error` + izoh:

```typescript
// @ts-expect-error — TODO: JIRA-1234 fix types after library update
```

ESLint rule `@typescript-eslint/ban-ts-comment` — `@ts-ignore`'ni taqiqlaydi va `@ts-expect-error`'ga aylantirishni tavsiya qiladi (eski `prefer-ts-expect-error` rule typescript-eslint v8'da `ban-ts-comment` ichiga qo'shilgan).

### Kod misol

```typescript
// === @ts-ignore ===
// @ts-ignore
const x: number = "hello"; // Xato yashirildi ✅

// Vaqt o'tdi, kod tuzatildi:
// @ts-ignore
const y: number = 42; // Xato yo'q, lekin @ts-ignore qoldi — silent dead suppress

// === @ts-expect-error ===
// @ts-expect-error — library types broken (issue #123)
const a: number = "hello"; // Xato kutilgan ✅ suppress

// Vaqt o'tdi, library tuzatildi:
// @ts-expect-error
const b: number = 42;
// ❌ Error: Unused '@ts-expect-error' directive
// → Developer eslab oladi: suppress olib tashlash kerak

// === Real-world misol ===
// Library type'lari xato — issue ochilgan
import { someBuggyFunction } from "buggy-lib";

// @ts-expect-error — buggy-lib@1.2 types broken, fixed in 1.3 (track: issue #456)
const result = someBuggyFunction(42);

// Library 1.3'ga update qilingach, types to'g'ri bo'ladi
// → TS "Unused @ts-expect-error" error → suppress olib tashlanadi

// === ESLint config (typescript-eslint v8+) ===
// eslint.config.mjs
export default [
  {
    rules: {
      // ban-ts-comment ichiga prefer-ts-expect-error mantiq'i ko'chirilgan
      "@typescript-eslint/ban-ts-comment": ["error", {
        "ts-ignore": "allow-with-description",        // yoki false — butunlay taqiqlash
        "ts-expect-error": "allow-with-description",
        "ts-nocheck": true,                            // butunlay taqiqlash
        "ts-check": false,
        minimumDescriptionLength: 10,
      }],
    },
  },
];
```

### Edge Cases

- **`@ts-expect-error` `@ts-ignore`'dan keyin qo'shilgan**: existing codebase'da migration — `@ts-ignore`'ni `@ts-expect-error`'ga aylantirish. ESLint auto-fix.
- **Multi-line statement**: faqat keyingi statement uchun ishlaydi. Multi-line expression — har qatorda alohida.
- **Type-only error vs runtime error**: pragma faqat TS type error'ni yashiradi. Runtime'da kod baribir crash bo'ladi.
- **Production code'da `@ts-expect-error`**: minimalize qilish. Har biri tech debt — ticket bilan track qilish.

### Follow-up savollar

1. "Mavjud `@ts-ignore`'larni qanday topib almashtirish?" — `grep -r "@ts-ignore" src/` yoki ESLint rule auto-fix. Har birini audit qilish (suppress sababini topish).
2. "`@ts-expect-error` test fayllarda?" — Ha, intentional type error'ni assertion qilish uchun (`expect-type` library). Type test'lar uchun standart pattern.

</details>

---

## Savol 5: SWC va esbuild — "transpile fast, check separately" [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

SWC (Rust) va esbuild (Go) — TypeScript'ni JS'ga transpile qiladi, lekin **type-check qilmaydi**. Build = SWC/esbuild (tez), Type check = `tsc --noEmit` (CI'da). Ikki pipeline alohida — speed va safety birga.

### To'liq tushuntirish

`tsc` ikki ish qiladi: type-check (semantic) va emit (transpile). Type-check sekin (cross-file inference), emit nisbatan tez.

Modern bundler/transpiler'lar (SWC, esbuild, Babel) type-check qilmaydi — faqat type annotation'larni olib tashlaydi (`(x: number) => x` → `(x) => x`). Bu juda tez bo'lishi mumkin (native Rust/Go, parallel).

Strategiya: ikki step'ni ajratish.

- **Development**: `tsx` (esbuild + watch) yoki Vite (esbuild + HMR) — type-check IDE'da (TS language server background'da).
- **CI**: `tsc --noEmit` (type-check faqat) + SWC/esbuild (build) — parallel.
- **Production build**: SWC/esbuild — speed muhim.

Trade-off: SWC/esbuild type-check'siz — agar `tsc --noEmit` o'tkazib yuborilsa, runtime'da type bug topiladi.

| Tool | Type check | Mechanism | Asosiy farq |
|------|------------|-----------|-------------|
| tsc | ✅ | TypeScript compiler (JS) | Semantic analysis + emit, type-check sekin |
| SWC | ❌ | Rust, parallel | Faqat transpile (annotation strip + syntax lowering) |
| esbuild | ❌ | Go native | Faqat transpile, bundle ham qiladi |
| Babel + preset-typescript | ❌ | JS plugin pipeline | Faqat transpile, legacy ecosystem |

Aniq performance raqamlari benchmark'larga bog'liq (loyiha hajmi, machine, config). Hozirgi rasmiy docs'da `tsc`'ga nisbatan o'lchov keltirilgan, ammo konkret raqam loyihaga qarab farq qiladi.

### Kod misol

```json
// === package.json scripts ===
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
# === SWC bilan build ===
npx swc src -d dist --config-file .swcrc
# Faqat transpile, type-check yo'q

# === esbuild bilan bundle ===
npx esbuild src/index.ts \
  --bundle \
  --outfile=dist/index.js \
  --platform=node \
  --target=node20 \
  --format=esm

# === Type check alohida ===
npx tsc --noEmit
# Faqat type-check, emit yo'q

# === Parallel CI ===
npm-run-all --parallel type-check lint
# yoki concurrently
npx concurrently "tsc --noEmit" "eslint src/"
```

```json
// === .swcrc — SWC config ===
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true
    },
    "target": "es2022",
    "transform": {
      "decoratorMetadata": true
    }
  },
  "module": {
    "type": "es6"
  }
}
```

```typescript
// === Type error SWC ko'rmaydi ===
// src/bug.ts
function add(a: number, b: number): number {
  return a + b;
}

add("1", "2"); // ❌ TS error
// SWC: silent (type-check qilmaydi)
// tsc --noEmit: error topiladi
// Runtime: "12" qaytaradi (string concat) — bug!
```

### Edge Cases

- **Decorator metadata** (NestJS, Angular): `decoratorMetadata: true` SWC config'da. esbuild legacy decorator'ni qo'llab-quvvatlamaydi (TC39 modern only).
- **`const enum` cross-file**: SWC va esbuild ham qo'llab-quvvatlamaydi (`isolatedModules` cheklov). `enum` (non-const) ishlaydi.
- **Source map**: production debugging uchun majburiy. SWC `sourceMaps: true`, esbuild `--sourcemap`.
- **Path alias (`paths`)**: SWC va esbuild ham `tsconfig.json` `paths`'ni avtomatik o'qimaydi — plugin (`tsconfig-paths-webpack-plugin`, `esbuild-plugin-tsc`) yoki manual config.

### Follow-up savollar

1. "Type-check'ni o'tkazib yuborgan team qanday bug topadi?" — Production crash. CI'da `tsc --noEmit` majburiy (pre-commit hook ham yoqilishi mumkin: `husky` + `lint-staged`).
2. "tsc o'rniga SWC butunlay almashtira oladimi?" — Yo'q, type-check'ni qilmaydi. tsc complementary (type-check). Build'ni SWC qiladi.

</details>

---

## Savol 6: `tsx`, `ts-node`, Node.js strip-types — qaysi farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`tsx` — esbuild-based, tez, type-check yo'q (dev'da eng qulay). `ts-node` — tsc API, sekin, type-check bor (legacy). Node.js `--experimental-strip-types` (22.6+) — native annotation strip, type-check yo'q, eng tez.

### To'liq tushuntirish

TS faylni to'g'ridan-to'g'ri Node.js'da run qilish — bundle'siz development.

**`ts-node`** (legacy):

- TypeScript compiler API ishlatadi.
- Type-check qiladi (sekin).
- Eski, deprecated emas, lekin yangi loyihada `tsx` afzal.

**`tsx`** (modern):

- esbuild-based, juda tez.
- Type-check yo'q (faqat transpile).
- Watch mode (`tsx watch`).
- Path alias (`tsconfig.json` paths) avtomatik.

**Node.js `--experimental-strip-types`** (22.6+):

- Type annotation'larni native parser bilan strip (transpile yo'q).
- `enum`, `namespace`, parameter property qo'llab-quvvatlamaydi (`erasableSyntaxOnly` mos).
- Initial qo'llab-quvvatlash Node 22.6'da flag bilan keldi.
- Keyingi Node versiyalari (22 LTS va 23.x) flag'siz default'ga aylantirgan.

| Tool | Type check | Speed | Mechanism |
|------|:----------:|--------|-----------|
| `ts-node` | ✅ | Sekin | TSC compiler API |
| `tsx` | ❌ | Tez | esbuild transpile |
| Node strip-types | ❌ | Eng tez | Native annotation strip |

Tavsiya:

- **Dev**: `tsx` (tez, watch).
- **CI type check**: `tsc --noEmit`.
- **CI test**: `vitest` (native TS) yoki `tsx` + `node --test`.
- **Production**: `tsc` build + `node` run.

### Kod misol

```bash
# === tsx (modern dev) ===
npx tsx src/index.ts
npx tsx watch src/index.ts        # watch mode
npx tsx --tsconfig ./tsconfig.json src/index.ts

# === ts-node (legacy) ===
npx ts-node src/index.ts
npx ts-node --transpile-only src/index.ts  # type-check skip
npx ts-node-esm src/index.ts                # ESM mode

# === Node.js native (22.6+ flag, keyingi versiyalarda default) ===
node --experimental-strip-types src/index.ts

# Yangi Node versiyalarda (default'ga aylanganidan keyin):
node src/index.ts

# === Cheklovlar ===
# src/legacy.ts
enum Color { Red, Green }     // ❌ Node strip-types xato
namespace Utils { /* ... */ } // ❌
class User {
  constructor(public name: string) {} // ❌ parameter property
}

# === Erasable variant (Node strip-types mos) ===
const Color = { Red: 0, Green: 1 } as const;
type Color = typeof Color[keyof typeof Color];
// namespace o'rniga module/file
class User {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}
```

```json
// === package.json scripts (modern stack) ===
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "type-check": "tsc --noEmit",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run"
  },
  "devDependencies": {
    "tsx": "^4.x",
    "typescript": "^5.x",
    "vitest": "^1.x"
  }
}
```

### Edge Cases

- **`tsx` `.cjs` va `.mjs` farqi**: `tsx` `package.json` `"type"` field'ni o'qib avtomatik aniqlaydi. Override: `tsx --tsconfig` yoki rename.
- **Node strip-types va library**: `node_modules` ichidagi `.ts` fayllarni strip qilmaydi (security). Faqat o'z source. Library publish'da `.js` + `.d.ts` standart.
- **ESM/CJS interop**: `tsx` har ikkalasi bilan ishlaydi (auto-detect). `ts-node` CJS default, ESM uchun `ts-node-esm`.
- **Source map debugging**: `tsx` source map'ni inline qiladi. Node DevTools'da TS source ko'rinadi.

### Follow-up savollar

1. "Production'da `tsx` ishlatish mumkinmi?" — Mumkin, ammo har request'da transpile (cold start sekin). `tsc` build + `node dist/` afzal (warm start tez).
2. "Bun bilan farqi?" — Bun native TS runtime (Zig). `bun run src/index.ts` to'g'ridan-to'g'ri. Lekin Node.js ecosystem compatibility hozircha to'liq emas (production'da audit kerak).

</details>

---

## Savol 7: JSDoc bilan type-safe JS — to'liq misol [Middle]

**Savol:** `// @ts-check` bilan JS faylda generic function, typedef, type import bilan kod yozing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JSDoc TypeScript syntax'ning JS-mos versiyasi: `@typedef`, `@param`, `@returns`, `@template` (generic), `@type`, type import (`import("./mod").User`). `checkJs: true` yoki per-file `// @ts-check` bilan ishlaydi.

### To'liq tushuntirish

JSDoc TypeScript inference uchun:

- **`@typedef`** — type alias (interface'ga ekvivalent).
- **`@param {Type} name`** — function parameter.
- **`@returns {Type}`** — function return type.
- **`@type {Type}`** — variable annotation.
- **`@template T`** — generic parameter.
- **`@template {Constraint} T`** — bounded generic.
- **`import("path").Type`** — boshqa fayldan type import.
- **`/** @type {Type} */ (expression)`** — inline type cast (parentheses muhim).

Strategiya: tayyor JS modul'larga `@ts-check` + JSDoc — `.ts`'ga o'tkazmasdan type-safety. Migration'ning oraliq bosqichi.

### Kod misol

```javascript
// @ts-check

// === Typedef ===
/**
 * @typedef {Object} User
 * @property {number} id
 * @property {string} name
 * @property {string} email
 * @property {"admin" | "user"} role
 * @property {Date} createdAt
 */

/**
 * @typedef {Object} CreateUserInput
 * @property {string} name
 * @property {string} email
 * @property {"admin" | "user"} [role] — optional
 */

// === Function with JSDoc ===
/**
 * @param {string} name
 * @param {string} email
 * @returns {User}
 */
function createUser(name, email) {
  return {
    id: Date.now(),
    name,
    email,
    role: "user",
    createdAt: new Date(),
  };
}

// === Generic function ===
/**
 * @template T
 * @param {T[]} arr
 * @param {(item: T) => boolean} predicate
 * @returns {T | undefined}
 */
function findItem(arr, predicate) {
  return arr.find(predicate);
}

// === Bounded generic ===
/**
 * @template {{ id: number }} T
 * @param {T[]} items
 * @param {number} id
 * @returns {T | undefined}
 */
function findById(items, id) {
  return items.find((i) => i.id === id);
}

// === Type variable annotation ===
/** @type {User[]} */
const users = [];

// === Type import (boshqa fayldan) ===
/** @type {import("./models/product").Product} */
let currentProduct;

// === Inline cast ===
const canvas = /** @type {HTMLCanvasElement} */ (document.getElementById("game"));
const ctx = canvas.getContext("2d"); // ✅ CanvasRenderingContext2D inferred

// === Type guard ===
/**
 * @param {unknown} value
 * @returns {value is User}
 */
function isUser(value) {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value
  );
}

// === Class with JSDoc ===
class UserService {
  /** @type {User[]} */
  #users = [];

  /**
   * @param {CreateUserInput} input
   * @returns {User}
   */
  create(input) {
    const user = createUser(input.name, input.email);
    this.#users.push(user);
    return user;
  }

  /**
   * @param {number} id
   * @returns {User | undefined}
   */
  findById(id) {
    return findById(this.#users, id);
  }
}

// === Usage ===
const service = new UserService();
const user = service.create({ name: "Ali", email: "ali@test.com" });
// user: User ✅
console.log(user.role); // ✅ "admin" | "user" narrowed
```

### Edge Cases

- **JSDoc inline cast parentheses majburiy**: `/** @type {T} */ (expr)` — parentheses'siz keyingi statement'ga qo'llanadi.
- **Generic constraint syntax**: `@template {Constraint} T` — `Constraint`'ni TS syntax bilan ishlatish.
- **Imported type re-export**: `/** @typedef {import("./mod").User} User */` — barrel pattern.
- **`@ts-check` strictness**: `noImplicitAny: true` flag JS faylga ham qo'llaniladi — JSDoc'siz parameter `any` infer qilinsa error. JSDoc `@param` annotation explicit type beradi.

### Follow-up savollar

1. "JSDoc'da `Partial`, `Pick`, `Omit` ishlatib bo'ladimi?" — Ha: `@param {Partial<User>} u`. TypeScript utility type'lar JSDoc'da to'liq ishlaydi.
2. "JSDoc inference TS sifatida toza ishlaydi?" — Ko'p case'da yes. Murakkab generic chain'larda TS native syntax aniqroq. Library author'lar uchun `.ts` afzal.

</details>

---

## Savol 8: Migration tsconfig — 3 ta xato toping [Middle]

**Savol:** Bu tsconfig migration boshlanishi uchun yozilgan. 3 ta muammoni toping va tuzating.

```json
{
  "compilerOptions": {
    "module": "Node16",
    "allowJs": true,
    "checkJs": true,
    "strict": true,
    "skipLibCheck": false,
    "incremental": false
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

3 ta xato: `strict: true` migration boshida yuzlab xato beradi (gradual sub-flag tavsiya), `skipLibCheck: false` `node_modules` `.d.ts`'larni sekin tekshiradi, `incremental: false` har safar full recompile.

### To'liq tushuntirish

**Xato 1: `strict: true` migration boshida**

Existing JS codebase'da `strict: true` yoqilganda:

- `noImplicitAny`: har parameter `any` deb taxmin qilingan → 100+ error.
- `strictNullChecks`: har `null`/`undefined` check'siz access → 50+ error.
- `strictPropertyInitialization`: har class field constructor'da initialize qilinmagan → 20+ error.

Yechim: sub-flag'larni birma-bir yoqish.

**Xato 2: `skipLibCheck: false`**

Default. `node_modules` ichidagi har `.d.ts` (Lodash, React types) tekshiriladi:

- Type conflict library'lar orasida (`@types/express` vs `@types/node`) → error.
- Build sezilarli sekin (har dependency tekshiriladi).

Yechim: `skipLibCheck: true` (90% loyihada standart).

**Xato 3: `incremental: false`**

Har TSC chaqirilganda butun loyiha qayta compile. `incremental: true` `.tsbuildinfo` cache yaratadi — keyingi run'da faqat o'zgarganlar.

Yechim: `incremental: true` (CI'da `.tsbuildinfo` cache qilinishi mumkin, lokal'da har vaqt tez).

### Kod misol

```json
// === ❌ Original — muammolar ===
{
  "compilerOptions": {
    "module": "Node16",
    "allowJs": true,
    "checkJs": true,
    "strict": true,           // ← migration boshida 100+ error
    "skipLibCheck": false,    // ← sekin, library conflict
    "incremental": false      // ← har safar full rebuild
  }
}

// === ✅ Migration bosqich 1 (boshlanish) ===
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "allowJs": true,
    "checkJs": true,
    "strict": false,
    "noImplicitAny": false,   // ← keyingi bosqichda yoqish
    "skipLibCheck": true,     // ← speed
    "incremental": true,      // ← cache
    "esModuleInterop": true,
    "outDir": "dist"
  },
  "include": ["src"]
}

// === Migration bosqich 2 — noImplicitAny ===
{
  "compilerOptions": {
    "noImplicitAny": true     // ← endi har parameter type majburiy
  }
}

// === Migration bosqich 3 — strictNullChecks ===
{
  "compilerOptions": {
    "noImplicitAny": true,
    "strictNullChecks": true  // ← null/undefined narrowing majburiy
  }
}

// === Migration tugagach — full strict ===
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": true,
    "incremental": true
  }
}
```

### Edge Cases

- **`skipLibCheck: false` qachon kerak?** — Library publish loyiha — o'z `.d.ts` `node_modules` types bilan to'qnashmaslik. Lekin baribir `skipLibCheck: true` afzal (CI bug'lar exit'da topiladi).
- **`incremental: true` + `composite: true`**: project references bilan birga ishlaydi. `composite` `incremental`'ni avtomatik yoqadi.
- **`.tsbuildinfo` file location**: default `outDir`'da. `tsBuildInfoFile`'da custom path.
- **`strict: false` + `noImplicitAny: true`**: explicit override pattern. Gradual migration'da har sub-flag'ni alohida boshqarish.

### Follow-up savollar

1. "`strict: true`'ga o'tish vaqti qachon?" — `noImplicitAny` + `strictNullChecks` + `strictFunctionTypes` clean bo'lganda. Loyiha team'iga bog'liq (2-6 oy gradual).
2. "`skipLibCheck: true` xavfsizmi?" — Ha. `@types/*` paketlar maintainer'lar tomonidan tekshirilgan. O'z `.d.ts`'lar `include`'da tekshiriladi.

</details>

---

## Savol 9: `const enum` + SWC bug — Output [Middle]

**Savol:** Bu build nima uchun ishlamaydi? 2 ta muammoni ko'rsating.

```json
{
  "scripts": { "build": "swc src -d dist" }
}
```

```typescript
// src/direction.ts
const enum Direction { Up, Down, Left, Right }
export function move(dir: Direction): string {
  return `Moving ${dir}`;
}

// src/main.ts
import { move, Direction } from "./direction";
console.log(move(Direction.Up));
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ikki muammo: (1) `const enum` cross-file SWC bilan ishlamaydi — SWC fayl-by-fayl transpile qiladi, boshqa fayldagi enum qiymatlarini inline qila olmaydi. (2) Type-check script yo'q — SWC type-check qilmaydi.

### To'liq tushuntirish

**Muammo 1: `const enum` cross-file**

`tsc` `const enum`'ni inline qiladi (`Direction.Up` → `0`):

```typescript
// tsc emit
console.log(move(0)); // Direction.Up inline
```

SWC bitta faylni ko'rib transpile qiladi — `main.ts`'da `Direction` import'i kelganda enum object yo'q (`const enum` runtime'da emit qilinmaydi). Natija:

```javascript
// SWC emit (xato)
import { move, Direction } from "./direction";
console.log(move(Direction.Up));
// Runtime: ReferenceError — Direction is not defined
```

Yechim: oddiy `enum` yoki `as const` object.

**Muammo 2: SWC type-check qilmaydi**

SWC faqat transpile (annotation strip). Type error'lar build'da topilmaydi:

```typescript
move("invalid"); // SWC: silent, tsc: error
```

CI'da `tsc --noEmit` majburiy.

### Kod misol

```typescript
// === ❌ Original — buzilgan ===
// src/direction.ts
const enum Direction { Up, Down, Left, Right }
export function move(dir: Direction): string {
  return `Moving ${dir}`;
}

// === ✅ Yechim 1: oddiy enum (runtime object) ===
// src/direction.ts
export enum Direction { Up, Down, Left, Right }
export function move(dir: Direction): string {
  return `Moving ${dir}`;
}
// SWC emit: IIFE bilan enum object yaratiladi
// var Direction = (function() { Direction[Direction["Up"] = 0] = "Up"; ... })()

// === ✅ Yechim 2: as const object (modern, erasable) ===
// src/direction.ts
export const Direction = {
  Up: 0,
  Down: 1,
  Left: 2,
  Right: 3,
} as const;

export type Direction = typeof Direction[keyof typeof Direction];

export function move(dir: Direction): string {
  return `Moving ${dir}`;
}

// === ✅ Yechim 3: string literal union (eng oddiy) ===
export type Direction = "up" | "down" | "left" | "right";

export function move(dir: Direction): string {
  return `Moving ${dir}`;
}

// === To'g'ri package.json scripts ===
```

```json
{
  "scripts": {
    "build": "swc src -d dist",
    "type-check": "tsc --noEmit",
    "ci": "npm run type-check && npm run build && npm run test"
  }
}
```

```json
// === tsconfig.json — isolatedModules majburiy ===
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "isolatedModules": true,     // ← const enum'ni TS o'zi tutsin
    "verbatimModuleSyntax": true
  }
}
// isolatedModules: true yoqilganda const enum TS error beradi:
// "'const' enums are not supported with 'isolatedModules'"
```

### Edge Cases

- **`preserveConstEnums: true`**: `tsc` `const enum`'ni runtime object'ga aylantiradi. Ammo `isolatedModules`'da baribir cheklanadi.
- **Babel `@babel/preset-typescript`**: `const enum` umuman qo'llab-quvvatlamaydi (legacy).
- **Vite `define`**: build-time'da constant'larni inline qilish — `const enum` alternativasi (`define: { "process.env.NODE_ENV": "production" }`).
- **Performance argument**: `const enum` build-size'ni kichraytiradi (object emas, inline). Modern bundler tree-shaking bilan farq minimal.

### Follow-up savollar

1. "`enum` umuman ishlatmaslik kerakmi?" — TypeScript community shift'i: `as const` object afzal (erasable, tree-shakeable, JSON-serializable). Modern style guide'lar `enum`'dan qochadi.
2. "Mavjud `const enum`'larni qanday topish va almashtirish?" — `grep -r "const enum" src/`. Manual rewrite (`as const` object + type alias). ESLint rule `@typescript-eslint/prefer-as-const`.

</details>

---

## Savol 10: `type-coverage` — CI integration [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`type-coverage` — TypeScript loyihada `any` type foizini o'lchaydi. Migration progressini track qilish va regression'ni oldini olish uchun. CI'da `--at-least 90` threshold bilan ishlatiladi.

### To'liq tushuntirish

`type-coverage` qanday ishlaydi:

1. TS AST'ni o'qiydi.
2. Har identifier (variable, parameter, return value) type'ini tekshiradi.
3. `any` type'iga ega bo'lganlarni hisoblaydi.
4. Coverage = (typed / total) * 100.

Foydali metric'lar:

- **Overall coverage**: butun loyiha bo'yicha.
- **Detailed report**: qaysi fayl/qator `any` ekanligini ko'rsatadi.
- **Strict mode**: generic `<T>` ham `any` deb hisoblaydi (`function id<T>(x: T): T` da `T` baribir `any` runtime'da).

CI integration:

- `npm run type-coverage --at-least 90` — 90%'dan past bo'lsa, exit 1 (CI fail).
- Har PR'da threshold tekshiriladi.
- Migration paytida threshold sekin ko'tariladi (70% → 80% → 90%).

Tahminiy daraja:

| Coverage | Daraja |
|----------|--------|
| < 50% | Migration boshlanmagan |
| 50-70% | Migration o'rtasida |
| 70-85% | Yaxshi (`any`'lar maxsus joyda) |
| 85-95% | Mature codebase |
| 95%+ | Production-ready, strict |

### Kod misol

```bash
# === Asosiy commands ===

# Overall coverage
npx type-coverage
# 87.45% (8543/9764)

# Detailed report — qaysi qatorda any
npx type-coverage --detail

# Strict mode (generic <T> ham any deb hisoblaydi)
npx type-coverage --strict

# CI threshold
npx type-coverage --at-least 90

# JSON output (programmatic)
npx type-coverage --reportSemanticError
```

```json
// === package.json ===
{
  "scripts": {
    "type-check": "tsc --noEmit",
    "type-coverage": "type-coverage --strict --at-least 90",
    "ci": "npm run type-check && npm run type-coverage && npm test"
  },
  "devDependencies": {
    "type-coverage": "^2.x",
    "typescript": "^5.x"
  }
}
```

```yaml
# === .github/workflows/ci.yml ===
name: CI
on: [push, pull_request]
jobs:
  type-coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run type-check
      - name: Type coverage
        run: npx type-coverage --strict --at-least 90
      - name: Block decrease
        run: |
          CURRENT=$(npx type-coverage --strict | grep -oP '\d+\.\d+(?=%)')
          THRESHOLD=90
          if (( $(echo "$CURRENT < $THRESHOLD" | bc -l) )); then
            echo "Coverage $CURRENT% < $THRESHOLD%"
            exit 1
          fi
```

```typescript
// === any'larni progressiv kamaytirish ===

// ❌ Bosqich 1: hammasi any
function parse(input: any): any {
  return JSON.parse(input);
}

// ✅ Bosqich 2: input type
function parse(input: string): any {
  return JSON.parse(input);
}

// ✅ Bosqich 3: return type (Zod)
import { z } from "zod";
const Schema = z.object({ name: z.string() });

function parse(input: string): z.infer<typeof Schema> {
  return Schema.parse(JSON.parse(input));
}

// ✅ Bosqich 4: full type-safe
function parse<T extends z.ZodTypeAny>(
  input: string,
  schema: T
): z.infer<T> {
  return schema.parse(JSON.parse(input));
}
```

### Edge Cases

- **Generic `any` strict mode**: `function id<T>(x: T): T` — `T` runtime'da `any`. `--strict` mode buni `any` deb hisoblaydi. Loose mode'da typed.
- **Third-party untyped library**: `@types/x` yo'q paket — har import `any` cascade. `declare module "x";` quick fix yoki `@types/x` qo'shish.
- **`any` cast (`as any`)**: type-coverage tutadi. Lint rule (`@typescript-eslint/no-explicit-any`) ham ishlatish.
- **JSON.parse natural `any`**: built-in API. Zod/Valibot bilan parse — typed.

### Follow-up savollar

1. "Threshold qachon oshirish?" — Har milestone'da +5% (70 → 75 → 80). Team velocity'ga qarab. PR'larda regression bloklash.
2. "`any` qachon mantiqiy?" — External API boundary (validation'gacha), generic `Function.bind` chaining, eski library wrapper. Documented exception'lar.

</details>

---

## Savol 11: `isolatedModules` vs `isolatedDeclarations` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedModules` — `.js` emit uchun (transpilation). Har fayl mustaqil transpile qilinishi mumkinligini kafolatlaydi (SWC/esbuild uchun). `isolatedDeclarations` (TS 5.5+) — `.d.ts` emit uchun (declaration). Har fayl mustaqil declaration emit qilinishi mumkinligini kafolatlaydi (parallel build).

### To'liq tushuntirish

Ikki flag — turli emit fazasiga ta'sir qiladi, lekin printsip bir xil (file isolation).

**`isolatedModules`** cheklovlari:

- `const enum` cross-file inline talab qiladi → taqiq.
- Ambiguous re-export (`export { X } from "./mod"`) — X type yoki value bo'lishi noma'lum → `export type` majburiy.
- `export = X` legacy CJS → taqiq.
- Non-module fayl (`import`/`export`'siz) → error.

**`isolatedDeclarations`** cheklovlari:

- Har exported function'da explicit return type majburiy.
- Har exported variable'da explicit type annotation (yoki literal narrowing) majburiy.
- Class public method'larida explicit return type majburiy.
- Tool (`tsc` yoki yangi declaration emitter) faqat shu faylni o'qib `.d.ts` yarata olishi kerak — boshqa fayllardan cross-file inference TAQIQ.

Birga ishlatish — SWC/esbuild parallel build:

- `.js` emit: SWC (fayl-by-fayl, `isolatedModules`).
- `.d.ts` emit: alohida tool (`tsc --emitDeclarationOnly` yoki yangi tool'lar). Har fayl mustaqil → parallel.

### Kod misol

```typescript
// === isolatedModules cheklovi ===

// ❌ const enum cross-file
export const enum Status { Active = "A", Inactive = "I" }
// → 'const' enums are not supported with 'isolatedModules'

// ❌ Ambiguous re-export
export { User } from "./types";
// User type yoki value? Bundler bitta faylda bilmaydi
// → 'isolatedModules' bilan 'export type' majburiy

// ❌ export =
class Foo {}
export = Foo;
// → 'export =' bilan isolatedModules mos kelmaydi

// ✅ To'g'rilangan
export const Status = { Active: "A", Inactive: "I" } as const;
export type Status = typeof Status[keyof typeof Status];

export type { User } from "./types";

export default class Foo {}

// === isolatedDeclarations cheklovi ===

// ❌ Implicit return type
export function createUser(name: string) {
  return { id: Date.now(), name };
}
// → Function must have an explicit return type annotation

// ❌ Implicit variable type
export const config = { port: 3000, host: "localhost" };
// → Variable must have an explicit type annotation

// ❌ Class method implicit return type
export class UserService {
  getUser(id: number) {
    return { id, name: "Ali" };
  }
}

// ✅ To'g'rilangan
interface User {
  id: number;
  name: string;
}

export function createUser(name: string): User {
  return { id: Date.now(), name };
}

interface Config {
  port: number;
  host: string;
}

export const config: Config = { port: 3000, host: "localhost" };
// Yoki: as const satisfies Config

export class UserService {
  getUser(id: number): User {
    return { id, name: "Ali" };
  }
}

// === Birga ishlatish ===
```

```json
// tsconfig.json — library publish uchun ideal
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "isolatedModules": true,
    "isolatedDeclarations": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true
  }
}
```

```json
// package.json — parallel build
{
  "scripts": {
    "build:js": "swc src -d dist",
    "build:dts": "tsc --emitDeclarationOnly",
    "build": "npm-run-all --parallel build:js build:dts",
    "type-check": "tsc --noEmit"
  }
}
```

### Edge Cases

- **`isolatedDeclarations` `private` method**: `.d.ts`'ga emit qilinmaydi (private API). Faqat `public`/`protected` uchun explicit return type kerak.
- **Generic inference**: `function fn<T>(x: T)` — return type explicit (`: T`) kerak. TS o'zi infer qila olmaydi (cross-file context).
- **`as const satisfies`** — `isolatedDeclarations` mos: literal narrowing + constraint, explicit type yozish kerak emas (qiymatda ko'rinadi).
- **Overload signature**: har overload explicit type talab qilinadi.

### Follow-up savollar

1. "`isolatedDeclarations` library author uchun majburiymi?" — Production tavsiya: explicit type discipline + parallel build. Lekin existing library'da migration vaqt talab qiladi.
2. "Yangi loyihada qaysi yoqish?" — Ikkalasi: modern stack default. Vite + library publish setup'ida ikkalasi yoqilgan.

</details>

---

## Savol 12: Strict migration plan — 5 bosqich [Junior+]

**Savol:** Existing JS loyiha (500 fayl) `strict: true` ga 5 bosqichlik migration plan yozing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

5 bosqich: (1) `noImplicitAny` → (2) `strictNullChecks` → (3) `strictFunctionTypes` + `strictBindCallApply` → (4) `strictPropertyInitialization` + `noImplicitThis` → (5) `strict: true` (barchasini birlashtirish). Har bosqich alohida PR + CI green.

### To'liq tushuntirish

Strict sub-flag'lar har biri turli tahminiy ish soni keltiradi (loyihaga bog'liq):

**Bosqich 1: `noImplicitAny`**

- Eng katta bosqich. Har parameter type annotation kerak.
- Tahminiy soat: katta (500 fayl loyihada ko'p kun).
- PR strategy: har modul (subdirectory) alohida.

**Bosqich 2: `strictNullChecks`**

- Har `null`/`undefined` access narrowing.
- Optional chaining (`?.`), nullish coalescing (`??`) keng ishlatiladi.
- Tahminiy soat: o'rta-katta.

**Bosqich 3: `strictFunctionTypes` + `strictBindCallApply`**

- Function parameter contravariance.
- `bind`/`call`/`apply` argument typing.
- Tahminiy soat: o'rta (callback-heavy kodda ko'proq).

**Bosqich 4: `strictPropertyInitialization` + `noImplicitThis`**

- Class field constructor'da initialize qilinishi.
- `this` type aniq bo'lishi (callback'da `bind` yoki arrow function).
- Tahminiy soat: kichik-o'rta.

**Bosqich 5: `strict: true`**

- `alwaysStrict`, `useUnknownInCatchVariables`.
- `catch (e: unknown)` narrowing.
- Tahminiy soat: kichik.

### Kod misol

```json
// === Bosqich 0: boshlanish ===
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "strict": false,
    "skipLibCheck": true,
    "incremental": true,
    "module": "Node16",
    "moduleResolution": "Node16"
  }
}
```

```json
// === Bosqich 1: noImplicitAny ===
{
  "compilerOptions": {
    "noImplicitAny": true
  }
}
```

```typescript
// ❌ Implicit any
function add(a, b) { return a + b; }

// ✅ Explicit
function add(a: number, b: number): number {
  return a + b;
}
```

```json
// === Bosqich 2: strictNullChecks ===
{
  "compilerOptions": {
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

```typescript
// ❌ Null access
function getLength(s: string | null) {
  return s.length; // Object is possibly 'null'
}

// ✅ Narrowing
function getLength(s: string | null): number {
  return s?.length ?? 0;
}
```

```json
// === Bosqich 3: strictFunctionTypes + strictBindCallApply ===
{
  "compilerOptions": {
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true
  }
}
```

```typescript
// ❌ Bind argument type
function greet(name: string) { return `Hello, ${name}`; }
const fn = greet.bind(null, 42); // Error: argument not assignable

// ✅
const fn = greet.bind(null, "Ali");
```

```json
// === Bosqich 4: strictPropertyInitialization + noImplicitThis ===
{
  "compilerOptions": {
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true
  }
}
```

```typescript
// ❌ Property not initialized
class User {
  name: string; // Error: not initialized
}

// ✅
class User {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}

// Yoki definite assignment
class UserDI {
  name!: string; // ! — boshqa joyda set qilinadi (DI framework)
}
```

```json
// === Bosqich 5: full strict ===
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true
  }
}
```

```typescript
// useUnknownInCatchVariables: true
try {
  JSON.parse("invalid");
} catch (e) {
  // e: unknown
  if (e instanceof Error) console.log(e.message);
}
```

### Edge Cases

- **CI threshold during migration**: har bosqich orasida `type-coverage` threshold sekin oshiriladi (`--at-least 70 → 75 → 80 → 85 → 90`).
- **`!` non-null assertion abuse**: `strictNullChecks` yoqilganda dasturchilar `!` bilan tezkor "fix" qilishi mumkin — bu xavfli. Code review'da bloklash, ESLint (`@typescript-eslint/no-non-null-assertion`).
- **DI framework (NestJS)**: `strictPropertyInitialization` + `@Injectable()` — `!: Service` (definite assignment) yoki `declare` modifier.
- **Test fayllar `strict`'siz**: alohida `tsconfig.test.json` `extends` bilan strict'ni o'chirishi mumkin (legacy test'lar tezroq migrate qilish uchun). Lekin uzun muddat — strict.

### Follow-up savollar

1. "Migration vaqtida feature work to'xtatiladimi?" — Yo'q. Parallel: feature team `.ts` yangi kodda, migration team mavjud kod bilan. Har sprint 5-10% coverage o'sishi target.
2. "Roll-back stratery?" — Har bosqich alohida PR + revert imkoniyat. CI baseline'i — har sub-flag yoqilgandan keyin team bir-ikki kun stabilize qiladi.

</details>

---

## Savol 13: SWC bilan decorator metadata bug [Senior]

**Savol:** NestJS loyiha SWC'ga ko'chirildi. `@Injectable()` decorator'lar runtime'da ishlamayapti. Sabab nima?

```typescript
@Injectable()
export class UserService {
  constructor(private repo: UserRepository) {}
}
```

```json
{
  "scripts": { "build": "swc src -d dist" }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

SWC config'da `decoratorMetadata: true` va `legacyDecorator: true` (NestJS legacy decorator spec ishlatadi) yoqilmagan. Reflect metadata constructor parameter type'larini saqlamasa, NestJS DI container parameter'larni resolve qila olmaydi.

### To'liq tushuntirish

NestJS DI mexanizmi:

1. `@Injectable()` class'ni provider sifatida marker qiladi.
2. `Reflect.getMetadata("design:paramtypes", UserService)` — constructor parameter'lar type'larini oladi.
3. Container har parameter type (`UserRepository`) uchun mos instance topadi va inject qiladi.

`design:paramtypes` metadata — TypeScript compiler tomonidan emit qilinadi `emitDecoratorMetadata: true` flag bilan. SWC default'da bu metadata'ni emit qilmaydi — alohida config kerak.

NestJS legacy decorator spec'ni (TC39 stage 2, "experimentalDecorators") ishlatadi. Yangi TC39 stage 3 decorator API mos kelmaydi — `legacyDecorator: true` zarur.

### Kod misol

```typescript
// === Original kod ===
import { Injectable } from "@nestjs/common";

@Injectable()
export class UserService {
  constructor(private repo: UserRepository) {}

  async findById(id: number) {
    return this.repo.find(id);
  }
}
```

```jsonc
// === ❌ .swcrc — metadata yo'q ===
{
  "jsc": {
    "parser": {
      "syntax": "typescript"
    },
    "target": "es2022"
  }
}
// SWC emit: decorator ishlaydi, ammo metadata yo'q
// NestJS runtime: Cannot resolve dependencies of UserService
```

```jsonc
// === ✅ .swcrc — to'g'ri config ===
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true                  // ← decorator syntax enable
    },
    "transform": {
      "legacyDecorator": true,            // ← NestJS legacy spec
      "decoratorMetadata": true           // ← Reflect.metadata emit
    },
    "target": "es2022",
    "keepClassNames": true                // ← class.name preserve (DI uses)
  },
  "module": {
    "type": "es6"
  }
}
```

```typescript
// === Verification ===
import "reflect-metadata"; // ← entry point'da import majburiy

import { UserService } from "./user.service";

const paramTypes = Reflect.getMetadata("design:paramtypes", UserService);
console.log(paramTypes); // [class UserRepository] ✅
// SWC config to'g'ri bo'lsa — array of class references
// Aks holda: undefined
```

```json
// === package.json ===
{
  "scripts": {
    "build": "swc src -d dist",
    "build:tsc": "tsc",
    "type-check": "tsc --noEmit",
    "start:prod": "node dist/main.js"
  },
  "dependencies": {
    "@nestjs/common": "^10.x",
    "@nestjs/core": "^10.x",
    "reflect-metadata": "^0.2.x"
  },
  "devDependencies": {
    "@swc/cli": "^0.3.x",
    "@swc/core": "^1.x"
  }
}
```

```json
// === tsconfig.json — tsc fallback uchun ===
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "outDir": "dist",
    "rootDir": "src",
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

### Edge Cases

- **TS 5.0+ native decorator (TC39 stage 3)**: NestJS hozircha legacy decorator ishlatadi. Migration uchun NestJS major version kutilmoqda (`experimentalDecorators: false` mos kelmaydi).
- **Circular dependency**: `Reflect.metadata` parameter type'larini saqlaydi. Class A → B → A circular bo'lsa, metadata `undefined` bo'lishi mumkin. NestJS `@Inject(forwardRef(() => B))` yechimi.
- **Multiple emit pipeline'lar**: TypeScript tsc + SWC bir vaqtda — config'lar mos kelishi kerak (`emitDecoratorMetadata`/`decoratorMetadata` ikkalasi).
- **esbuild decorator support**: esbuild legacy decorator'ni qo'llab-quvvatlamaydi (TC39 modern only). NestJS uchun SWC tavsiya.

### Follow-up savollar

1. "`reflect-metadata` polyfill nima uchun kerak?" — `Reflect.metadata` API standartning bir qismi emas (TC39 stage 1 proposal). Polyfill `reflect-metadata` paket'i `Reflect`'ga metadata API qo'shadi.
2. "NestJS production'da SWC vs tsc?" — SWC sezilarli tez build (CI'da). NestJS team SWC'ni rasmiy qo'llab-quvvatlaydi. tsc fallback uchun saqlash mumkin.

<details>
<summary><strong>Deep Dive</strong></summary>

**`design:paramtypes` qanday emit qilinadi**

TypeScript compiler `emitDecoratorMetadata: true` flag bilan har decorated class uchun quyidagi metadata key'larni emit qiladi:

- `design:type` — property type'i (single).
- `design:paramtypes` — method/constructor parameter type'lari (array).
- `design:returntype` — method return type'i.

Emit qilingan kod:

```javascript
// constructor(private repo: UserRepository) {}
// emitDecoratorMetadata: true bilan
__decorate([
  Injectable()
], UserService);
__decorate([
  __metadata("design:paramtypes", [UserRepository])  // ← class reference array
], UserService);
```

`UserRepository` class reference'i runtime'da DI container'ga "qaysi instance kerak" deb javob beradi.

**SWC implementatsiyasi**

SWC `decoratorMetadata: true` config bilan TypeScript compiler emit'iga ekvivalent metadata yaratadi. Rust kod TypeScript type checker'i emas — type annotation'lardan class reference'larni statik tarzda ekstrakt qiladi.

Cheklov: SWC interface/type alias parameter type'larni metadata'ga emit qila olmaydi (faqat class reference). Bu TypeScript bilan ham xuddi shunday — interface runtime'da yo'q.

**Stage 2 (legacy) vs Stage 3 decorator farqi**

| Xususiyat | Stage 2 (legacy) | Stage 3 (TS 5.0+) |
|-----------|------------------|-------------------|
| `experimentalDecorators` | majburiy | yopiq (default) |
| `emitDecoratorMetadata` | qo'llab-quvvatlanadi | qo'llab-quvvatlanmaydi |
| Decorator signature | `(target, propertyKey, descriptor)` | `(value, context)` |
| Class field | `target` instance/prototype | `context.kind === "field"` |
| Parameter decorator | mavjud | hozircha taklif yo'q |
| NestJS support | rasmiy | hozircha yo'q |

NestJS DI mexanizmi `emitDecoratorMetadata`'ga bog'liq, shu sababli stage 2'da qoladi.

**Circular dependency va metadata `undefined`**

```typescript
// a.ts
@Injectable()
export class A {
  constructor(private b: B) {}  // ← B class hali define qilinmagan
}

// b.ts
@Injectable()
export class B {
  constructor(private a: A) {}
}
```

Class declaration order tufayli `B` reference `a.ts` execution paytida `undefined` bo'lishi mumkin. Metadata array: `[undefined]`. NestJS yechimi:

```typescript
@Injectable()
export class A {
  constructor(@Inject(forwardRef(() => B)) private b: B) {}
}
```

`forwardRef` lazy resolution — DI container instance kerak bo'lganda evaluate qiladi.

</details>

</details>

---

## Xulosa

- Gradual migration — leaf-first, bosqichma-bosqich PR'lar (big bang katta loyihada ishlamaydi)
- `allowJs` → `.js`/`.ts` birgalikda, `checkJs` → `.js` ham type-check
- Leaf-first — dependency tree pastidan, cascade xato yo'q, type "yuqoriga oqadi"
- `@ts-expect-error` > `@ts-ignore` — kod tuzatilganda eslatadi (unused directive error)
- SWC/esbuild — transpile (type-check yo'q), `tsc --noEmit` parallel CI'da majburiy
- `tsx` — dev'da eng qulay (esbuild-based), `ts-node` legacy, Node strip-types eng tez (erasable syntax cheklovi)
- JSDoc — `.ts`'siz type-safety: `@typedef`, `@param`, `@template`, `import(...).Type`
- Migration tsconfig boshida: `strict: false`, sub-flag'lar birma-bir, `skipLibCheck: true`, `incremental: true`
- `const enum` SWC/esbuild bilan cross-file ishlamaydi — `as const` object alternativasi
- `type-coverage` — migration progress o'lchash, CI'da `--at-least 90 --strict` threshold
- `isolatedModules` vs `isolatedDeclarations` — `.js` emit vs `.d.ts` emit, parallel build
- 5-bosqich strict migration: `noImplicitAny` → `strictNullChecks` → `strictFunctionTypes` → `strictPropertyInitialization` → `strict: true`
- SWC + NestJS: `decoratorMetadata: true`, `legacyDecorator: true`, `keepClassNames: true`, `reflect-metadata` polyfill

**Keyingi bo'lim:** [25-build-tools.md](25-build-tools.md) — `tsc` build pipeline, project references, `composite`, `paths` resolution, bundler integration.
