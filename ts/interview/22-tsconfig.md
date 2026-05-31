# Interview: tsconfig.json

> `tsconfig.json` compiler options bo'yicha interview savollari: `target` vs `lib`, `strict` family, `module`/`moduleResolution` juftliklari, `paths` runtime gotcha, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `erasableSyntaxOnly`, `isolatedModules` vs `isolatedDeclarations`, project references, per-project optimal config.

---

## Mundarija

**Nazariy savollar:**
1. `target` va `lib` farqi `[Junior+]`
2. `strict: true` sub-flag'lari `[Middle]`
3. `noUncheckedIndexedAccess` — output `[Middle+]`
4. `exactOptionalPropertyTypes` — `?` va `| undefined` farqi `[Middle+]`
5. `module`/`moduleResolution` juftliklari `[Middle]`
6. `paths` va `baseUrl` — runtime gotcha `[Middle]`
7. `erasableSyntaxOnly` (TS 5.8+) `[Middle+]`
8. `isolatedModules` constraints `[Middle]`
9. Project References — `composite`/`outDir` `[Middle+]`
10. Library tsconfig — 5 ta xato `[Senior]`
11. Optimal tsconfig — React/Node/Library `[Middle+]`

**Output va bug fix:**
12. `isolatedModules` xato — output `[Middle]`
13. `noUncheckedIndexedAccess` — compile xato `[Middle]`
14. Module/moduleResolution juftlik + exclude `[Middle]`
15. `verbatimModuleSyntax` qachon va nima uchun `[Middle+]`
16. Project References — build order `[Middle+]`
17. `useDefineForClassFields` semantics `[Senior]`
18. `isolatedDeclarations` (TS 5.5+) — bug fix `[Senior]`

---

## Savol 1: `target` va `lib` orasidagi farq nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`target` — emit qilingan JS qaysi ECMAScript syntax'da bo'lishini belgilaydi. `lib` — type-checking paytida qaysi built-in API type declaration'lari mavjudligini aytadi.

### To'liq tushuntirish

`target` `tsc` ning emit fazasiga ta'sir qiladi: `"ES5"` da `async/await` `__awaiter` helper bilan downlevel qilinadi, `class` ham `prototype`-based function'ga aylantiriladi. `"ES2022"` da hammasi native syntax saqlanadi.

`lib` `tsc` ning checker fazasiga ta'sir qiladi: `Promise`, `Map`, `WeakRef`, `Array.prototype.at` kabi global'lar uchun `.d.ts` declaration'lari. `lib` ko'rsatilmaganda — `target` ga mos default yuklanadi (`target: "ES5"` → `lib: ["ES5", "DOM"]` web context'da).

Farq amaliy: eski browser (`target: "ES5"`) lekin polyfill mavjud (`core-js`) — `lib: ["ES2020", "DOM"]` qo'yib `Promise`/`Map` type'larini ishlatish mumkin. Aksincha — Node.js'da `lib: ["ES2022"]` qo'yib `DOM` global'larini chiqarib tashlash (`document is not defined` xatosini compile-time'da topish).

### Kod misol

```typescript
// === Misol 1: Node.js 20 server ===
// tsconfig.json
// { "target": "ES2022", "lib": ["ES2022"] }
//
// DOM yo'q — bu xato compile-time'da topiladi:
// document.getElementById("x");
// → Error: Cannot find name 'document'

// === Misol 2: Eski browser, polyfill bilan ===
// { "target": "ES5", "lib": ["ES2020", "DOM"] }
const cache = new Map<string, number>(); // ✅ Map type mavjud
cache.set("count", 1);
// Output JS — ES5 syntax, lekin Map runtime'da polyfill kerak (core-js)

// === Misol 3: target downlevel ===
async function fetchUser(id: number) {
  return { id, name: "Ali" };
}
// target: "ES2017" → native async function saqlanadi
// target: "ES5" → __awaiter + __generator helper'lari bilan emit
```

### Edge Cases

- `lib` o'zgartirilganda target default lib'i olib tashlanadi — qo'lda `lib: ["ES2020", "DOM"]` yozganda har ikkalasini ham yozish kerak (DOM'ni unutish — `window`/`document` type'lari yo'qoladi).
- `target: "ESNext"` — har TS release bilan o'zgaruvchan (eng yangi qabul qilingan + staged feature'larni qamrab oladi). Production'da reproducible build uchun aniq versiya tavsiya (`"ES2022"`).
- `lib: ["ES2022"]` server'da — `DOM.Iterable` yo'q. Iterator Helpers (`.map`, `.filter`, `.toArray` iterator'larda) Stage 4 proposal, ES2025'ga kiritilgan — TS 5.6+ da `lib.es2025.iterator.d.ts` qo'shildi, `lib: ["ES2025.Iterator"]` (yoki to'liq ES2025/ESNext) bilan yoqiladi. Eski runtime'da `core-js` polyfill.
- `target` `importHelpers` bilan birga — `tslib` package'dan helper import (`__awaiter` har faylda inline emas), bundle hajmi kichrayadi.

### Follow-up savollar

1. "`target: "ES5"` + `lib: ["ES5"]` da `Promise` ishlatsam nima bo'ladi?" — Compile error: `Cannot find name 'Promise'`. `lib`'ga `"ES2015"` qo'shish kerak.
2. "Vite + React loyihada `target` nima bo'lishi kerak?" — `"ES2020"`/`"ES2022"` (zamonaviy browser'lar). Vite Babel/esbuild bilan downlevel qiladi, tsc faqat type-check.

</details>

---

## Savol 2: `strict: true` qaysi sub-flag'larni yoqadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`strict: true` — meta-flag bo'lib, 8 ta sub-flag'ni birdan yoqadi: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`, `useUnknownInCatchVariables`.

### To'liq tushuntirish

Har sub-flag mustaqil yoqilishi mumkin — `strict` shunchaki barchasini bir vaqtning o'zida yoqadi. Yangi TS versiyalarda yangi strict sub-flag qo'shilsa, `strict: true` avtomatik yoqadi (forward-compatible).

Sub-flag'lar tafsiloti:

- **`noImplicitAny`** — type annotation yo'q va inference imkonsiz bo'lsa, `any` taxmin qilmaydi, error beradi.
- **`strictNullChecks`** — `null`/`undefined` alohida type, har joyda assignable emas (`string` ga `null` o'tmaydi).
- **`strictFunctionTypes`** — function parameter'larda contravariance enforce qilinadi. `method` syntax (`handleEvent(e: T): void`) bundan istisno — bivariant qoladi (DOM API compatibility uchun).
- **`strictBindCallApply`** — `Function.prototype.bind/call/apply` argumentlari type-check qilinadi.
- **`strictPropertyInitialization`** — class field constructor'da yoki declaration'da initialize qilinmasa, error (`strictNullChecks` ham yoqilgan bo'lishi kerak).
- **`noImplicitThis`** — `this` type aniqlanmasa, error.
- **`alwaysStrict`** — emit qilingan JS har faylga `"use strict"` qo'shadi va parse'da strict mode qo'llaydi.
- **`useUnknownInCatchVariables`** (TS 4.4+) — `catch (e)` blokida `e` type'i `unknown` (oldin `any` edi). TS 4.4'gacha barcha versiyalarda `any` default'i edi.

`strict`'ga **kirmaydigan** strict-like flag'lar: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature` — alohida yoqish kerak.

### Kod misol

```typescript
// === noImplicitAny ===
function greet(name) {} // ❌ Parameter 'name' implicitly has an 'any' type
function greetOk(name: string) {} // ✅

// === strictNullChecks ===
function getLength(s: string | null): number {
  // return s.length; // ❌ Object is possibly 'null'
  return s?.length ?? 0; // ✅ optional chaining + nullish coalescing
}

// === strictFunctionTypes ===
type Animal = { name: string };
type Dog = Animal & { bark(): void };

let logAnimal: (a: Animal) => void = (a) => console.log(a.name);
let logDog: (d: Dog) => void = (d) => d.bark();

// logAnimal = logDog;
// ❌ — Dog parameter Animal'dan tor, contravariance buziladi
// logDog = logAnimal; // ✅ — Animal kengroq, OK

// === strictPropertyInitialization ===
class User {
  // name: string; // ❌ Property 'name' has no initializer
  name: string;
  constructor(name: string) { this.name = name; } // ✅
}

// === useUnknownInCatchVariables ===
try {
  JSON.parse("invalid");
} catch (e) {
  // e: unknown
  if (e instanceof Error) console.log(e.message); // ✅ narrowing kerak
  // console.log(e.message); // ❌ Object is of type 'unknown'
}
```

### Edge Cases

- `strict: true` + `strictPropertyInitialization` lekin field DI framework (NestJS) tomonidan inject qilinadi — `!` definite assignment assertion (`name!: string`) yoki `@Injectable()` decorator pattern.
- `strictFunctionTypes` faqat **function syntax**'ga qo'llanadi (`(x: T) => void`), method syntax'da (`handleEvent(e: T): void`) bivariant qoladi. Bu legacy DOM API'lar (`addEventListener`) uchun maxsus qoldirilgan.
- `useUnknownInCatchVariables: false` qilib, `catch (e: Error)` yozish — TS bunga ruxsat bermaydi (`catch` parameter faqat `any` yoki `unknown` bo'lishi mumkin, spec talabi).
- `strict: true` lekin `noImplicitAny: false` — sub-flag override ishlaydi, `strict` general default beradi.

### Follow-up savollar

1. "`strict: true` lekin `noImplicitAny: false` yozsam, qaysi g'olib?" — Explicit yozilgan sub-flag g'olib. Bu pattern legacy loyiha gradual migration'da ishlatiladi.
2. "`exactOptionalPropertyTypes` nima uchun `strict`'ga kirmaydi?" — TS 4.4'da qo'shilgan, breaking change ko'p kod uchun. Opt-in qoldirildi backward compat uchun.

</details>

---

## Savol 3: `noUncheckedIndexedAccess` — Output [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`noUncheckedIndexedAccess: true` da index access natijasi `T | undefined` bo'ladi. Compile-time'da `undefined` ehtimolini tekshirishga majbur qiladi.

### To'liq tushuntirish

Default'da TypeScript index access'ni "naive" qiladi: `arr[i]` → `T`, `record[key]` → `V`. Lekin runtime'da `arr[100]` `undefined` qaytishi mumkin (out-of-bounds), `record["nonexistent"]` ham. Flag yoqilganda — TS bu reallikni model qiladi.

Behavior:

- **Arrays**: `T[]` ning `arr[i]` natijasi `T | undefined`.
- **Tuples**: known index (`tuple[0]`) o'zgarmaydi (compile-time'da bor), faqat dynamic index'da `| undefined`.
- **Records**: `Record<K, V>` ning index access natijasi `V | undefined`.
- **Mapped types**: index signature bilan ham qo'shiladi.

`strict`'ga **kirmaydi** — alohida yoqish kerak.

### Kod misol

```typescript
// noUncheckedIndexedAccess: true bilan
const headers: Record<string, string> = { "Content-Type": "application/json" };
const ct = headers["Content-Type"];
// ct: string | undefined ← yoqilmagan paytda `string` edi (xavfli)

// ❌ Bevosita ishlatish — error
// ct.toLowerCase(); // Object is possibly 'undefined'

// ✅ Narrowing
if (ct !== undefined) {
  console.log(ct.toLowerCase()); // ct: string
}

// ✅ Nullish coalescing default
const safeCt = ct ?? "text/plain";

// === Tuple farqi ===
const tuple: [number, string] = [1, "x"];
const first = tuple[0]; // number (known index, undefined emas)
const i = Math.floor(Math.random() * 2);
const dyn = tuple[i];   // number | string | undefined

// === Array misol ===
const items = ["a", "b", "c"];
const item = items[0];
// item: string | undefined ← runtime'da items bo'sh bo'lishi mumkin
```

### Edge Cases

- `for...of` loop ichida — `item` aniq `T` (undefined emas), TS narrowing qiladi. `forEach` callback ham shu tarzda.
- Destructuring: `const [first, second] = arr;` da `first: T | undefined` (out-of-bounds ehtimoli). Default value bilan tuzatish: `const [first = "x"] = arr;`.
- `Map.prototype.get` bu flag'dan **mustaqil** — har doim `V | undefined` (lib.d.ts'da shunday declare qilingan).
- `Array.prototype.at(0)` ham har doim `T | undefined` (lib.d.ts default, flag'siz ham).

### Follow-up savollar

1. "`for (let i = 0; i < arr.length; i++)` ichida `arr[i]` baribir `| undefined`mi?" — Ha, TS bu pattern'ni control flow'da narrow qila olmaydi. `for...of` ishlatish yoki `const item = arr[i]!` (xavfli) yoki `if (item) { ... }`.
2. "Tuple va array farqi `noUncheckedIndexedAccess`'da nima?" — Tuple'da **known index** `T` qoladi (`tuple[0]` masalan), array'da har `arr[i]` `T | undefined`.

</details>

---

## Savol 4: `exactOptionalPropertyTypes` — `?` va `| undefined` farqi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`exactOptionalPropertyTypes: true` da `{ x?: string }` (property mavjud bo'lmasligi mumkin) va `{ x: string | undefined }` (property mavjud, qiymati `undefined` bo'lishi mumkin) farqli type'lar bo'ladi.

### To'liq tushuntirish

Default'da TypeScript ikkala syntax'ni semantically bir xil deb hisoblaydi — `x?: T` `x: T | undefined`'ga teng. Lekin runtime'da farq bor:

- `{ name: "Ali" }` — `Object.keys` `["name"]`, `"nickname" in obj` `false`.
- `{ name: "Ali", nickname: undefined }` — `Object.keys` `["name", "nickname"]`, `"nickname" in obj` `true`.

Bu farq amaliy: database `null` vs missing field, `Object.assign` behavior, JSON serialization (`JSON.stringify` `undefined` properties'ni o'tkazib yuboradi, `null`'ni saqlaydi).

`exactOptionalPropertyTypes: true` yoqilganda:

- `x?: T` — qiymat `T` yoki property butunlay yo'q (`undefined` literal assignment ruxsat etilmaydi).
- `x: T | undefined` — property mavjud, qiymat `T` yoki `undefined`.

`strict`'ga **kirmaydi** — alohida yoqish kerak.

### Kod misol

```typescript
interface User {
  name: string;
  nickname?: string;   // property mavjud bo'lmasligi mumkin
  bio: string | undefined; // property doim mavjud, qiymati undefined bo'lishi mumkin
}

// === exactOptionalPropertyTypes: true ===

const u1: User = { name: "Ali", bio: undefined }; // ✅
const u2: User = { name: "Ali", nickname: "Al", bio: "Dev" }; // ✅

// ❌ Error: Type 'undefined' is not assignable to type 'string'
// const u3: User = { name: "Ali", nickname: undefined, bio: undefined };

// ✅ — nickname'ni butunlay yozmaslik
const u4: User = { name: "Ali", bio: undefined };

// === Runtime farq ===
console.log("nickname" in u1); // false — property yo'q
console.log("bio" in u1);      // true — property bor, qiymati undefined

// === API'ga ta'sir ===
function updateUser(u: User, patch: Partial<User>) {
  return { ...u, ...patch };
  // patch.nickname undefined bo'lsa va exactOptional yoqilgan bo'lsa —
  // patch'ga undefined yozish mumkin emas, faqat property'ni o'tkazib yuborish
}

// === JSON serialization farqi ===
JSON.stringify({ a: undefined }); // '{}' — property yo'qoladi
JSON.stringify({ a: null });      // '{"a":null}' — saqlanadi
```

### Edge Cases

- `Partial<T>` bilan birga: `Partial<User>` har property'ni `?` qiladi. `exactOptionalPropertyTypes` yoqilgan bo'lsa, `{ name: undefined }` Partial'ga assign qilinmaydi — bu kutilmagan bo'lishi mumkin.
- React `useState<string | undefined>` vs `useState<string>()` — birinchisi `T | undefined`, ikkinchisi initial undefined ammo state type `string`. Library typing bu flag bilan o'zaro ta'sirda noaniq bo'lishi mumkin.
- DOM API'larda ko'p property `T | undefined` declare qilingan, lekin runtime'da har doim mavjud. Bu flag bilan kelishilmagan typing'lar yuzaga chiqishi mumkin.
- `delete obj.x` operatsiyasi `x?: T` uchun ruxsat etiladi, `x: T | undefined` uchun ham (lekin semantically noto'g'ri — property o'chiriladi, lekin type'ga ko'ra mavjud bo'lishi kerak).

### Follow-up savollar

1. "Bu flag yoqilganda mavjud kod buzilarmi?" — Ha, ko'p loyihada `{ x: undefined }` pattern ishlatiladi. Gradual migration tavsiya: oldin `noUncheckedIndexedAccess`, keyin `exactOptionalPropertyTypes`.
2. "Library author sifatida `?` yoki `| undefined` tanlash?" — Semantic muhim. "Field yo'q bo'lishi mumkin" → `?`. "Field bor, lekin qiymati ma'lum emas" → `| undefined`. Database null field → `T | null`, missing column → `?`.

</details>

---

## Savol 5: `module` va `moduleResolution` qaysi juftlikda? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

To'g'ri juftliklar: `module: "Node16"`/`"NodeNext"` + `moduleResolution: "Node16"`/`"NodeNext"` (Node.js native), `module: "ESNext"`/`"Preserve"` + `moduleResolution: "Bundler"` (Vite/Webpack). `moduleResolution: "node"` (legacy node10) — eski loyihalar uchun, yangi kodda tavsiya etilmaydi.

### To'liq tushuntirish

`module` — emit qilingan JS'da qaysi module syntax (`require`/`import`/`define`).

`moduleResolution` — TypeScript import path'ni qanday resolve qiladi:

- **`node`** (legacy `node10`) — eski Node.js algoritmi (`exports`/`imports` field qo'llab-quvvatlanmaydi).
- **`node16`** / **`nodenext`** — Node.js modern algoritmi: `package.json` `exports`, `imports`, conditional exports, `.mts`/`.cts` extension, `type: "module"` semantics. `nodenext` = "har TS versiya bilan Node'ning eng yangi algoritmi".
- **`bundler`** (TS 5.0+) — Vite/Webpack/esbuild kabi bundler'lar uchun: `exports` field qo'llab-quvvatlanadi, ammo `.js` extension import majburiyati yo'q (bundler resolve qiladi).
- **`classic`** — TS 1.x dan qolgan, deprecated, ishlatilmaydi.

Juftlik qoidasi: `module: "Node16"` bilan `moduleResolution: "node"` (legacy) — `exports` field ishlamaydi, ko'p modern package buziladi.

### Kod misol

```json
// === Node.js 20 ESM/CJS interop ===
{
  "compilerOptions": {
    "module": "Node16",
    "moduleResolution": "Node16",
    "target": "ES2022"
  }
}
// .ts → ESM emit (package.json "type": "module")
// .ts → CJS emit (package.json "type": "commonjs")
// .mts → har doim ESM, .cts → har doim CJS

// === Vite + React ===
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "target": "ES2022",
    "noEmit": true,           // ← Vite bundler compile qiladi
    "allowImportingTsExtensions": true
  }
}

// === Legacy Node.js (avoid in new projects) ===
{
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",  // legacy
    "target": "ES2020",
    "esModuleInterop": true
  }
}

// === Library publish (dual ESM/CJS) ===
{
  "compilerOptions": {
    "module": "Node16",
    "moduleResolution": "Node16",
    "declaration": true,
    "declarationMap": true
  }
}
```

### Edge Cases

- `moduleResolution: "Node16"` + `package.json` `"type": "module"` — `.ts` faylda relative import majburiy `.js` extension bilan (`import { createUser } from "./user.js"`). `.ts` import xato beradi (NodeNext qoidalari).
- `moduleResolution: "Bundler"` — `exports` field qo'llab-quvvatlanadi, lekin Node.js'da to'g'ridan-to'g'ri run qilib bo'lmaydi (faqat bundler context).
- `module: "Preserve"` (TS 5.4+) — bundler context uchun yangi qiymat, ES syntax o'zgartirmaydi, ammo `verbatimModuleSyntax`'ga rioya qiladi.
- `module: "NodeNext"` har TS minor release'da behavior o'zgarishi mumkin (Node.js'ning eng yangi algoritmiga moslashadi). Stable production'da `"Node16"` afzal.

### Follow-up savollar

1. "`module: "ESNext"` + `moduleResolution: "Node16"` ishlaydimi?" — Yo'q, mos kelmaydi. `Node16` `module: "Node16"`/`"NodeNext"` talab qiladi (CJS/ESM hybrid emission).
2. "Library publish qilayotganda qaysi juftlik?" — `module: "Node16"`, `moduleResolution: "Node16"` — Node.js consumer'lar uchun universal. Consumer Vite'da `Bundler` ishlatadi — library'ning `.d.ts` baribir to'g'ri resolve bo'ladi.

</details>

---

## Savol 6: `paths` va `baseUrl` — eng katta xato nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`paths` faqat TypeScript type resolution uchun ishlaydi — emit qilingan JS'da import path'lar **o'zgartirilmaydi**. Runtime'da har bundler/tool alohida sozlashni talab qiladi.

### To'liq tushuntirish

`paths` declaration `compilerOptions`'da:

```json
{ "baseUrl": ".", "paths": { "@/*": ["src/*"] } }
```

TypeScript checker `@/components/Button` ni `src/components/Button` deb resolve qiladi va type-check'ni o'tkazadi. Lekin `tsc` emit fazasida — import string'ni o'zgartirmaydi. Output JS'da `@/components/Button` qoladi — Node.js bu path'ni topa olmaydi, runtime crash.

Har tool alohida sozlanadi:

- **Vite/Rollup** — `vite.config.ts`'da `resolve.alias`.
- **Webpack** — `resolve.alias`.
- **Jest** — `jest.config.js`'da `moduleNameMapper`.
- **Node.js** — `tsconfig-paths` package yoki `tsx` (esbuild-based) avtomatik o'qiydi.
- **ts-node** — `tsconfig-paths/register`.

Library publish'da `paths` **xavfli**: emit qilingan `.d.ts` ham alias'larni saqlaydi (`import { x } from "@/utils"`), consumer'da `@/utils` resolve bo'lmaydi.

### Kod misol

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"]
    }
  }
}
```

```typescript
// src/app.ts
import { Button } from "@components/Button"; // TS type-check ✅
```

```javascript
// dist/app.js (emit'dan keyin)
const { Button } = require("@components/Button"); // ❌ Runtime crash
// Node.js: Error [ERR_MODULE_NOT_FOUND]
```

```typescript
// vite.config.ts — Vite uchun sozlash
import { defineConfig } from "vite";
import path from "node:path";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
      "@components": path.resolve(__dirname, "./src/components"),
    },
  },
});
```

```javascript
// jest.config.js — Jest uchun
module.exports = {
  moduleNameMapper: {
    "^@/(.*)$": "<rootDir>/src/$1",
    "^@components/(.*)$": "<rootDir>/src/components/$1",
  },
};
```

```bash
# Node.js'da tsconfig-paths bilan run
node -r tsconfig-paths/register dist/app.js

# tsx esbuild-based — paths avtomatik
npx tsx src/app.ts
```

### Edge Cases

- `vite-tsconfig-paths` plugin — `tsconfig.json` `paths`'ni avtomatik Vite alias'iga o'tkazadi (duplicate config kerak emas).
- Library publish'da yechim: build'da `tsc-alias` paketi bilan emit qilingan `.js`/`.d.ts` fayllarda alias'larni relative path'ga aylantirish.
- `baseUrl` `paths`'siz ham ishlatish mumkin: `baseUrl: "./src"` qo'yib, har import absolute (`import { x } from "utils/x"`). Lekin TS 5.0+ da `paths` ham `baseUrl`'siz ishlaydi (`baseUrl` faqat `paths` resolve uchun kerak emas).
- Monorepo workspace'da `paths` `references` bilan: type-check `paths` orqali, runtime `npm workspaces` bilan symlink — ikki yo'l birga.

### Follow-up savollar

1. "Nima uchun `tsc` import path'larini transform qilmaydi?" — `tsc` ko'p syntax transformation qiladi (downleveling, module format, JSX), ammo import specifier string'ini ataylab o'zgartirmaydi — module specifier'ni rewrite qilish bundler/runtime responsibility'sida qoldirilgan. `paths` faqat checker resolution uchun, emit'ga ta'sir qilmaydi.
2. "Library uchun `paths`'siz qanday qulay tartib?" — Workspace + relative import, yoki source'ni publish qilish (`exports.types` raw `.ts`'ga ko'rsatib). Build'da `tsc-alias` ham mumkin.

</details>

---

## Savol 7: `erasableSyntaxOnly` (TS 5.8+) nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`erasableSyntaxOnly: true` — faqat type-erasure'ga mos TS syntax'ga ruxsat beradi. Runtime semantic qo'shadigan construct'larni (`enum`, `namespace`, parameter property) taqiqlaydi. Node.js native `--experimental-strip-types` bilan moslik uchun.

### To'liq tushuntirish

Node.js 22.6+ `--experimental-strip-types` flag bilan TS fayllarni native run qila oladi: parser type annotation'larni shunchaki o'chiradi, transpilation yo'q. Bu yondashuv faqat **erasable syntax** uchun ishlaydi — type'larni yo'qotsa, qolgan JS valid bo'lishi kerak.

Quyidagi TS syntax'lar **erasable** (ruxsat etiladi):

- `type`, `interface` declaration'lar.
- Type annotation (`: string`, `as T`).
- Generic parameter (`<T>`).
- `satisfies`, `as const`.

Quyidagilar **erasable emas** (taqiqlanadi):

- `enum` (runtime'da object yaratadi, `Direction.Up` runtime'da kerak).
- `namespace` (function ichida property — runtime semantics).
- Parameter property (`constructor(public name: string)` — runtime'da `this.name = name` qo'shadi).
- `import = require()` syntax (TS-specific CJS interop).

### Kod misol

```typescript
// === Erasable (ruxsat etiladi) ===
type User = { name: string };           // ✅
interface Config { port: number }       // ✅
function add<T>(a: T, b: T): T { return a; } // ✅
const x = 42 as number;                  // ✅
const tuple = [1, 2] as const;          // ✅

// === Non-erasable (taqiqlanadi) ===

// ❌ enum runtime'da object yaratadi
enum Direction { Up, Down }

// ❌ namespace runtime'da IIFE
namespace Utils {
  export const PI = 3.14;
}

// ❌ Parameter property runtime'da this assignment
class User {
  constructor(public name: string, private age: number) {}
}

// === To'g'rilash ===
// enum o'rniga as const
const Direction = { Up: 0, Down: 1 } as const;
type Direction = typeof Direction[keyof typeof Direction];

// Parameter property o'rniga explicit
class User {
  name: string;
  private age: number;
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}
```

```json
// Mos tsconfig
{
  "compilerOptions": {
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true,
    "module": "Node16",
    "moduleResolution": "Node16",
    "target": "ES2022"
  }
}
```

```bash
node --experimental-strip-types src/app.ts
```

### Edge Cases

- `declare enum` — type-only declaration, erasable (`erasableSyntaxOnly` ruxsat etadi).
- `const enum` — `erasableSyntaxOnly` bilan taqiqlanadi, chunki TS uni runtime'da inline qiladi (transpilation qo'shadi).
- `enum`'dan oddiy `object as const` ga migration — runtime semantic farq: enum reverse mapping (`Direction[0]` → `"Up"`) yo'qoladi.
- Legacy decorator (`experimentalDecorators: true`, NestJS `@Injectable`) — `erasableSyntaxOnly` bilan **taqiqlanadi**: `__decorate`/`__metadata` runtime helper'lar emit qilinadi, bu pure type-erasure emas. Standart Stage-3 decorator'lar (TS 5.0+, flag'siz) ham runtime code emit qiladi — decorator umuman erasable construct emas, native strip-types decorator'larni qo'llab-quvvatlamaydi.

### Follow-up savollar

1. "Existing loyihada `erasableSyntaxOnly` yoqsam, qancha kod buziladi?" — Bog'liq: NestJS heavy parameter property ishlatadi, ko'p buziladi. Modern React + TS — kam ta'sirlanadi. Avval `tsc --noEmit` bilan tekshirish.
2. "Native TS strip-types stable bo'lganda `tsc` kerak bo'larmi?" — Ha, type-checking uchun. `--experimental-strip-types` faqat **runtime** (type'larni o'chirib run qilish), `tsc --noEmit` baribir CI'da kerak.

</details>

---

## Savol 8: `isolatedModules` constraints [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedModules: true` — har TS fayl mustaqil transpile qilinishi mumkin bo'lishini kafolatlaydi (SWC, esbuild, Babel — fayl-by-fayl ishlaydi). Ambient (import qilingan/`declare`) `const enum` access'i, ambiguous re-export, `export =` kabi cross-file context talab qiladigan construct'larni taqiqlaydi.

### To'liq tushuntirish

`tsc` butun loyihani bir vaqtning o'zida ko'radi, ammo SWC/esbuild har faylni alohida transpile qiladi (parallel, tez). Ayrim TS feature'lar cross-file ma'lumotni talab qiladi — bunday faylni izolyatsiyada to'g'ri transpile qilib bo'lmaydi.

Cheklovlar:

- **Ambient `const enum` access** — `tsc` enum access'ini inline qiladi (`Direction.Up` → `0`). Single-file transpiler bitta faylni ko'rganda boshqa fayldagi const enum qiymatlarini bila olmaydi → import qilib access qilinganda TS2748. Shu faylda lokal declare qilingan const enum esa cheklanmaydi (declaration ko'rinadi).
- **`export =`** / **`import = require()`** — TypeScript-specific CJS interop, modern bundler'lar bu syntax'ni qo'llab-quvvatlamaydi.
- **Ambiguous re-export** — `export { User } from "./types"` — `User` type yoki value ekanligi faylda ko'rinmasa, `export type { User }` (type-only) majburiy.
- **Non-module fayl** — `isolatedModules` `moduleDetection: "force"` bilan birga ishlatiladi, har fayl modul bo'lishi kerak (`import`/`export` bo'lishi shart).
- **Namespace merging** — fayllar bo'ylab namespace augmentation bundler bilan ishlamaydi.

Modern Vite/Next.js loyihalarda `isolatedModules: true` default — bundler `esbuild`/`SWC` ishlatadi.

### Kod misol

```typescript
// === ❌ ambient const enum access cross-file ===
// types.ts
export const enum Status { Active = "A", Inactive = "I" }

// app.ts
import { Status } from "./types";
console.log(Status.Active);
// ❌ tsc: TS2748 — Cannot access ambient const enums when 'isolatedModules' is enabled
// Sabab: tsc emit'da "A" inline qiladi, single-file transpiler (SWC/esbuild) esa
// boshqa fayldagi qiymatni ko'rmaydi → Status object runtime'da yo'q → ReferenceError

// ✅ Yechim — oddiy enum yoki const object
export const Status = { Active: "A", Inactive: "I" } as const;
export type Status = typeof Status[keyof typeof Status];

// === ❌ Ambiguous re-export ===
// types.ts
export type User = { name: string };
export const ADMIN_ID = 1;

// re-export.ts
export { User } from "./types";
// isolatedModules: bundler User type'mi value'mi bilmaydi → error

// ✅ Yechim — explicit type
export type { User } from "./types";

// === ❌ export = ===
// legacy.ts
class PaymentGateway { /* ... */ }
export = PaymentGateway; // ❌ isolatedModules error

// ✅ Yechim
export default class PaymentGateway { /* ... */ }
// yoki
export class PaymentGateway { /* ... */ }

// === ❌ Non-module fayl ===
// global-script.ts
const helper = () => "hi";
// isolatedModules + moduleDetection: "force" — error: file is not a module

// ✅ Yechim — export qo'shish
export const helper = () => "hi";
```

```json
{
  "compilerOptions": {
    "isolatedModules": true,
    "moduleDetection": "force",
    "verbatimModuleSyntax": true
  }
}
```

### Edge Cases

- `verbatimModuleSyntax: true` `isolatedModules` cheklovlarini yanada qattiqlashtirib, `import type`/`export type`'ni majburiy qiladi (type-only va value-only import'lar aniq farqlanadi).
- Test framework (Jest + ts-jest) `isolatedModules` ni o'z config'ida ham yoqishi mumkin — test transpilation tezligini sezilarli oshiradi (type-check'ni o'tkazib yuborib).
- `isolatedModules` `isolatedDeclarations` bilan birga ishlatish — transpilation **va** declaration emit ham fayl-by-fayl bo'ladi (eng tez parallel build).
- `const enum` + `preserveConstEnums: true` — emit'da object saqlanadi, lekin `isolatedModules` baribir cross-file inline'ni cheklaydi.

### Follow-up savollar

1. "`isolatedModules` ESLint'da emas, tsconfig'da bo'lishi kerakligini nima sabab?" — Bu emit constraint, type-check fazasi (`tsc --noEmit` ham buni tekshiradi).
2. "Mavjud loyihada `const enum`'ni qanday topish?" — `grep -r "const enum" src/` yoki ESLint rule `@typescript-eslint/prefer-as-const`. Migration: enum → `as const` object.

</details>

---

## Savol 9: Project References — composite va outDir qoidalari [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Project references — bir nechta TS sub-project'ni bog'lash. Har referenced project'da `composite: true` majburiy (avtomatik `declaration: true`, `incremental: true`, `outDir` cheklovlari yoqiladi). Circular reference taqiqlanadi (DAG bo'lishi shart).

### To'liq tushuntirish

Monorepo'da har package o'z `tsconfig.json`'iga ega — har birini alohida compile qilish (parallel build, incremental cache). Project references type-safe cross-package import'lar uchun:

```json
{ "references": [{ "path": "../core" }] }
```

`composite: true` qoidalari (referenced project'da majburiy):

1. **`declaration: true`** avtomatik yoqiladi — `.d.ts` fayllar emit (cross-project type info).
2. **`incremental: true`** avtomatik yoqiladi — `.tsbuildinfo` cache.
3. **`rootDir`** explicit yoki inferred (tsconfig directory).
4. **`outDir`** explicit (yoki default behavior'da source bilan birga emit qilinadi).
5. **`include`/`files`** explicit — har fayl ro'yxati ma'lum bo'lishi kerak (faqat `include`'siz emas).
6. **Circular reference** taqiqlanadi — A → B → A bo'lmasligi kerak.

Build:

- `tsc --build` (yoki `tsc -b`) — dependency tartibida compile.
- `tsc -b packages/api` — `api` va uning dependency'lari (transitive).
- `tsc -b --watch` — incremental watch.
- `tsc -b --force` — cache'ni e'tiborsiz qoldirish (to'liq rebuild).

### Kod misol

```
monorepo/
├── tsconfig.base.json
├── tsconfig.json
└── packages/
    ├── core/
    │   ├── src/
    │   └── tsconfig.json
    └── api/
        ├── src/
        └── tsconfig.json
```

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

```json
// packages/core/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src",
    "declarationMap": true
  },
  "include": ["src"]
}
```

```json
// packages/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "references": [{ "path": "../core" }]
}
```

```json
// tsconfig.json (root — solution)
{
  "files": [],
  "references": [
    { "path": "packages/core" },
    { "path": "packages/api" }
  ]
}
```

```bash
# Barcha project'larni dependency tartibida build
tsc --build

# Faqat api va uning dependency'lari
tsc -b packages/api

# Watch mode — har o'zgarishda incremental rebuild
tsc -b --watch

# Cache'ni tozalab to'liq rebuild
tsc -b --clean
tsc -b --force
```

```typescript
// packages/api/src/index.ts
import { User } from "@org/core"; // ✅ Type-safe cross-project import
```

### Edge Cases

- `composite: true` lekin `rootDir` ko'rsatilmagan — TS `include`'dagi eng past umumiy directory'dan infer qiladi. Murakkab loyihada noaniq xulq, explicit yozish tavsiya.
- Circular reference (A → B → A) — `tsc -b` xato beradi: "Project references may not form a circular graph". Refactoring kerak: shared kod alohida `common` package'ga.
- `tsc --build` `--watch` bilan har save'da incremental rebuild — `.tsbuildinfo` orqali faqat o'zgargan project'lar rebuild bo'ladi. Loyiha hajmiga qarab build tezligi sezilarli o'sadi.
- `outDir` referenced project ichida — consumer project `references` orqali `.d.ts`'ni `outDir`'dan o'qiydi. Source'dan o'qish faqat `declarationMap` yoqilgan bo'lsa IDE "Go to Definition" uchun ishlaydi.

### Follow-up savollar

1. "`composite`'siz `references` ishlatsam nima bo'ladi?" — `tsc -b` error: "Referenced project must have setting composite: true".
2. "Yarn workspaces / pnpm workspaces bilan project references qanday birga ishlaydi?" — Workspace package'lar (`@org/core`) symlink orqali `node_modules`'ga linked. Project references type-checking uchun, workspace runtime resolution uchun — parallel ishlaydi.

</details>

---

## Savol 10: Library tsconfig — 5 ta xato toping [Senior]

**Savol:** Bu config library publish uchun yozilgan. 5 ta muammo toping va tuzating.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "outDir": "dist",
    "declaration": true,
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"]
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

5 ta muammo: `rootDir` yo'q, `declarationMap` yo'q, `moduleResolution: "Bundler"` library uchun noto'g'ri, `paths` library'da xavfli, `skipLibCheck` yo'q.

### To'liq tushuntirish

**1. `rootDir` yo'q** — `outDir: "dist"`'ga `src/index.ts` → `dist/src/index.ts` deb yoziladi (qo'shimcha `src/` papka). `rootDir: "src"` yozilsa → `dist/index.ts`.

**2. `declarationMap` yo'q** — Consumer IDE'da "Go to Definition" `.d.ts` ga olib boradi, source `.ts`'ga emas. `declarationMap: true` `.d.ts.map` fayllar yaratadi, `sourceMap` bilan birga consumer source'ga navigate qila oladi.

**3. `moduleResolution: "Bundler"` library uchun noto'g'ri** — Consumer Node.js'da `tsc` yoki bundler'da ishlatishi mumkin. `"Bundler"` faqat bundler context'da ishlaydi. Library uchun `module: "Node16"`, `moduleResolution: "Node16"` universal.

**4. `paths` library'da xavfli** — Emit qilingan `.d.ts`'da `@/*` saqlanadi (`import { Helper } from "@/utils"`). Consumer'da `@/utils` resolve bo'lmaydi. Library'da relative import (`"./utils"`) ishlatish kerak. Agar `paths` zarur bo'lsa — `tsc-alias` paketi bilan emit'da relative'ga aylantirish.

**5. `skipLibCheck` yo'q** — Default `false` da `node_modules/**/*.d.ts` tekshiriladi. Library'da dependency'lar tipi tekshirilishi sekin va keraksiz (`@types/*` paketlar o'zlari tekshirilgan). `skipLibCheck: true` build tezligini sezilarli oshiradi.

### To'g'rilangan kod

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2020"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "stripInternal": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedDeclarations": true
  },
  "include": ["src"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

Qo'shimcha library-specific options:

- **`stripInternal: true`** — `@internal` JSDoc'li declaration'larni `.d.ts`'dan olib tashlaydi (private API yashirish).
- **`isolatedDeclarations: true`** (TS 5.5+) — har export'da explicit return type majburiy, fayl-by-fayl `.d.ts` emit (parallel build).
- **`target: "ES2020"`** — library uchun konservativ (eski Node.js consumer'lar qo'llab-quvvatlanadi).
- **`exclude` test'larni** — `.d.ts`'da test type'lari emit qilinmasligi uchun.

### Edge Cases

- Dual ESM/CJS publish (`exports` field `package.json`'da) uchun ikki `tsconfig` (ESM uchun va CJS uchun) yoki `tsup`/`tshy` build tool ishlatish.
- `isolatedDeclarations` yoqilganda — barcha export function'lar explicit return type talab qiladi. Bu majburiyat dasturchini "to'g'ri" yozishga undaydi va parallel build imkonini beradi.
- `declarationMap` lekin `sourceMap` yo'q — IDE source navigation qisman ishlaydi. Ikkalasi birga afzal.

### Follow-up savollar

1. "`tsup` library build uchun afzalmi yoki raw `tsc`?" — `tsup` (esbuild-based) dual ESM/CJS, minification, faster build. Lekin type-checking baribir `tsc --noEmit` qo'shilishi kerak.
2. "Library'da `target: "ES2022"` yozsam bo'ladimi?" — Bo'ladi, lekin consumer Node 14-16 ishlatsa — top-level await yoki private class field'lar buziladi. Library author auditoriyaga qarab tanlaydi (konservativ `"ES2020"` afzal).

<details>
<summary><strong>Deep Dive</strong></summary>

**Emit pipeline'da `outDir` + `rootDir` interaction:** `tsc` `rootDir`'ni "source root" deb hisoblaydi va har file'ning relative path'ini `outDir`'ga ko'chiradi. `rootDir` `include`'dan eng past umumiy directory'ga default infer qilinadi. Library publish'da explicit yozish — emit struktura predictable bo'ladi (no surprise nested folders).

**`declarationMap` ichida nima saqlanadi:** `.d.ts.map` source-map format (V3) bilan — `.d.ts`'ning har position'i source `.ts`'ga mapping. Consumer IDE "Go to Definition" `.d.ts` o'rniga source `.ts`'ga olib boradi. Bu yondashuv `sources` field'da relative path orqali ishlaydi — package publish'da `.ts` source ham include qilinishi kerak (`files` field).

**`isolatedDeclarations` performance trade-off:** Cross-file inference yo'qotiladi, ammo har `.d.ts` mustaqil, faqat o'sha faylga qarab generate qilinadi. TS 5.5'dan beri `isolatedDeclarations: true` constraint'i bilan SWC/oxc kabi parser-only tool'lar `.d.ts` emit'ini type checker'siz, parallel bajara oladi (`tsc`'ning o'zi baribir constraint'ni tekshiradi va `.d.ts` emit qila oladi). Bu library author'lar uchun kompromiss: verbose explicit return type'lar evaziga parallel, tezroq build.

**`stripInternal` implementation:** TS checker JSDoc `@internal` tag'ini topadi va declaration emit'da uni skip qiladi. Mexanizm fragile: parser nuance'lariga sezgir. ESM context'da `__INTERNAL__` prefix ham ishlatiladi (Vue.js style) — har ikkala usul bir-birini almashtirmaydi.

**Dual ESM/CJS publish challenges:** `package.json` `exports` field bilan ikki entry — `import` (ESM) va `require` (CJS). `tsc` bitta `module` setting bilan ikkalasini ham emit qila olmaydi — `tshy`/`tsup`/`unbuild` tool'lar ikki build'ni boshqaradi (ikki `tsconfig`, har biri uchun alohida `outDir`). Type duplication: `.d.ts` (ESM) + `.d.cts` (CJS) — TS 5.0+ `--moduleDetection` aniqlik beradi.

</details>

</details>

---

## Savol 11: Optimal tsconfig — React App / Node.js API / Library [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

React App (Vite): `module: "ESNext"`, `moduleResolution: "Bundler"`, `noEmit: true`, `jsx: "react-jsx"`. Node.js API: `module: "Node16"`, `moduleResolution: "Node16"`, `outDir`, `declaration`, `sourceMap`. Library: `target: "ES2020"`, `declarationMap`, `stripInternal`, `isolatedDeclarations`.

### To'liq tushuntirish

Har project turi turli build pipeline va consumer'larga ega — config farqlanadi.

**React App (Vite):**

- Vite (esbuild/Rollup) bundler — `tsc` emit kerak emas (`noEmit: true`).
- Bundler `module` interpret qilmaydi (faqat parse) — `ESNext` saqlash, bundler downlevel qiladi.
- `moduleResolution: "Bundler"` — `exports` field qo'llab-quvvatlanadi, `.js` extension majburiy emas.
- `jsx: "react-jsx"` — React 17+ automatic JSX runtime (`import React` kerak emas).
- `verbatimModuleSyntax: true` — `import type` strict, side-effect import aniq.
- `isolatedModules: true` — esbuild fayl-by-fayl transpile.

**Node.js API:**

- `tsc` build qiladi (yoki `tsx`/`swc` + `tsc --noEmit` CI'da).
- `module: "Node16"` + `moduleResolution: "Node16"` — Node.js modern resolution (`exports`, `imports` fields).
- `outDir`/`rootDir` — `dist`/`src` ajratish.
- `declaration: true` + `sourceMap: true` — production debugging.
- `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` — production safety.

**Library:**

- Universal consumer (Node, bundler, Deno) — `module: "Node16"` xavfsiz.
- `target: "ES2020"` — eski Node.js support.
- `declarationMap`/`sourceMap` — consumer IDE/debugger.
- `stripInternal` — private API yashirish.
- `isolatedDeclarations` (TS 5.5+) — parallel build, explicit return type.

### Kod misol

```json
// === React App (Vite + React 19) ===
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noEmit": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}

// === Node.js 20 API ===
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "incremental": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}

// === Library (publish to npm) ===
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2020"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "stripInternal": true,
    "isolatedDeclarations": true,
    "verbatimModuleSyntax": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

### Edge Cases

- React App + Next.js — Next.js o'z `tsconfig` plugin'ini (`"plugins": [{ "name": "next" }]`) talab qiladi (auto-incremental). `next dev` `tsc --noEmit`'dan ko'ra ko'proq qiladi (incremental file detection).
- Node.js API + Native ESM (`"type": "module"`) — relative import majburiy `.js` extension bilan (`./user.js`), `.ts` import error.
- Library uchun `paths` ishlatish xavfli — `tsc-alias` bilan post-process kerak, yoki umuman ishlatmaslik.
- Monorepo'da har package alohida tsconfig — `extends` orqali base config'ni share qilish, har biri o'z `module`/`outDir`'ini override qiladi.

### Follow-up savollar

1. "Next.js'da `noEmit: true` qo'yganimda Vercel deploy'da nima bo'ladi?" — Next.js o'z build pipeline'iga ega, `tsc` chaqirmaydi. `noEmit` faqat `tsc --noEmit` CI uchun.
2. "Library'da `target: "ES2022"` + private class fields ishlatsam ?" — Consumer Node 16+'da ishlaydi, eski Node 14 da syntax error. Auditoriyani bilish kerak — modern library uchun OK, legacy support kerak bo'lsa `"ES2020"`.

</details>

---

## Savol 12: `isolatedModules` xato — Output [Middle]

**Savol:** `isolatedModules: true` da bu kodda nechta xato bor?

```typescript
const enum Direction { Up = "UP", Down = "DOWN" }
export { Direction };
export { User } from "./types";
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedModules` o'zi yoqilgan holda bu faylda **bitta** xato: `export { User } from "./types"` — `User` `./types`'da type bo'lib chiqsa, TS re-export'ni `export type` bilan majburlaydi (TS1205). `const enum Direction` shu faylda **lokal** declare qilingan, undan keyin `export { Direction }` — bu faylda xato bermaydi: TS local const enum qiymatlarini ko'rib turibdi. Const enum muammosi boshqa modul uni **import qilib access** qilganda (TS2748) yoki shu faylga `verbatimModuleSyntax: true` ham qo'shilganda chiqadi.

### To'liq tushuntirish

**Xato: ambiguous type re-export (TS1205)**

```typescript
export { User } from "./types";
```

`isolatedModules` har faylni single-file transpiler (SWC/esbuild/Babel) mustaqil transpile qila olishini kafolatlaydi. Bu transpiler `./types`'ni o'qimaydi — faqat joriy faylni. `User` type bo'lsa, emit'da re-export butunlay yo'qotilishi kerak (type-erasure); value bo'lsa — saqlanishi kerak. `tsc` o'zi `./types`'ni resolve qilib `User` type ekanini aniqlaganda — single-file transpiler buni qila olmasligini bilib xato beradi:

```
TS1205: Re-exporting a type when 'isolatedModules' is enabled requires using 'export type'.
```

```typescript
// ✅ Yechim — explicit export type
export type { User } from "./types";
```

**Nega `const enum Direction` + `export { Direction }` bu faylda xato emas**

`tsc` `const enum` access'ini inline qiladi: `Direction.Up` → `"UP"`. Bu inlining uchun declaration ko'rinishi kerak. Declaration **shu faylda** bo'lganda — transpiler ham uni ko'radi, shu sababli local const enum `isolatedModules`'da xato bermaydi. Muammo declaration boshqa modulda bo'lib (ambient/imported), uni access qilganda chiqadi:

```typescript
// other.ts — Direction'ni import qilib access qilsa:
import { Direction } from "./this-file";
console.log(Direction.Up);
// ❌ TS2748: Cannot access ambient const enums when 'isolatedModules' is enabled.
```

Shu sababli `const enum`'ni **export qilish** xavfli: consumer faylda access TS2748 beradi va single-file transpiler inline qila olmaydi. Bu faylga `verbatimModuleSyntax: true` qo'shilsa — const enum export'i ham xato bo'ladi (const enum runtime value emas, shaffof emit imkonsiz).

**Const enum'dan qochish — barqaror muqobil**

Yechim 1 — oddiy enum (runtime'da object emit qilinadi, cross-file inline kerak emas):

```typescript
export enum Direction { Up = "UP", Down = "DOWN" }
```

Yechim 2 — `as const` object (modern, type-safe):

```typescript
export const Direction = { Up: "UP", Down: "DOWN" } as const;
export type Direction = typeof Direction[keyof typeof Direction];
```

### Kod misol

```typescript
// === Original ===
const enum Direction { Up = "UP", Down = "DOWN" }
export { Direction };                  // bu faylda xato emas (local const enum)
export { User } from "./types";        // ❌ TS1205 — User type bo'lsa, export type kerak

// === ✅ Yechim 1 — oddiy enum + type re-export ===
export enum Direction { Up = "UP", Down = "DOWN" }
export type { User } from "./types";

// === ✅ Yechim 2 — as const + type re-export (modern) ===
export const Direction = { Up: "UP", Down: "DOWN" } as const;
export type Direction = typeof Direction[keyof typeof Direction];
export type { User } from "./types";
```

### Edge Cases

- `enum` (const emas) ham SWC/esbuild'da IIFE'ga emit qilinadi — ishlaydi. Const enum esa access'da inline'ga tayanadi, shuning uchun import qilib access qilinganda single-file transpiler buziladi.
- TS2748 (ambient const enum access) faqat **import qilingan/`declare` qilingan** const enum'da chiqadi — shu faylda lokal declare qilingan const enum'da emas.
- `preserveConstEnums: true` flag — `const enum`'ni runtime object'ga ham emit qiladi, lekin `isolatedModules` baribir cross-file access'ni cheklaydi (single-file transpiler boshqa fayldagi object'ni ko'rmaydi).
- Library publish'da `const enum` export qilish xavfli — consumer cross-package access TS2748 beradi yoki single-file transpiler inline qila olmaydi.

### Follow-up savollar

1. "`enum`'ni umuman ishlatmaslik kerakmi?" — Modern stil — `as const` object afzal (`erasableSyntaxOnly` mos, runtime overhead yo'q, tree-shakeable).
2. "`export type` `export { type }`'dan farqi nima?" — Syntax farqi: `export type { User }` butun statement type-only, `export { type User, createUser }` (TS 5.0+) inline marker — har export uchun alohida (`User` type-only, `createUser` value).

</details>

---

## Savol 13: `noUncheckedIndexedAccess` — Compile xato [Middle]

**Savol:** Bu kod `noUncheckedIndexedAccess: true` da compile bo'ladimi? Output va xato matnini ayting.

```typescript
const config: Record<string, string> = {
  host: "localhost",
  port: "3000"
};

function getConfig(key: string): string {
  const value = config[key];
  return value;
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Compile bo'lmaydi. `config[key]` `string | undefined` type'ga ega, ammo return type `string` deb e'lon qilingan. TS error: `Type 'string | undefined' is not assignable to type 'string'`.

### To'liq tushuntirish

`noUncheckedIndexedAccess: true` `Record<string, string>` ning index access'iga `| undefined` qo'shadi — runtime'da `config["nonexistent"]` `undefined` qaytishi mumkin (TS bu reallikni type'da modellaydi).

Function signature `string` qaytarishni va'da qiladi, lekin `value` `string | undefined` — `undefined`'ni `string` deb qaytarish noto'g'ri.

To'g'rilash variantlari:

1. **Undefined check** — explicit guard.
2. **Default value** — fallback bilan.
3. **Return type kengaytirish** — `string | undefined` qaytarish.
4. **Throw** — key mavjud bo'lmaganda exception.

### Kod misol

```typescript
const config: Record<string, string> = {
  host: "localhost",
  port: "3000"
};

// === ❌ Original — compile error ===
function getConfig(key: string): string {
  const value = config[key];
  // value: string | undefined
  return value; // ❌ Type 'string | undefined' not assignable to 'string'
}

// === ✅ Yechim 1 — Undefined check + throw ===
function getConfigOrThrow(key: string): string {
  const value = config[key];
  if (value === undefined) {
    throw new Error(`Config key not found: ${key}`);
  }
  return value; // value: string (narrowed)
}

// === ✅ Yechim 2 — Default value ===
function getConfigOrDefault(key: string, defaultValue: string): string {
  return config[key] ?? defaultValue;
}

// === ✅ Yechim 3 — Return type kengaytirish ===
function getConfigOptional(key: string): string | undefined {
  return config[key];
  // Caller undefined handle qiladi
}

// Ishlatish:
const host = getConfigOrThrow("host"); // "localhost"
const proto = getConfigOrDefault("protocol", "http"); // "http" (default)
const xyz = getConfigOptional("xyz"); // undefined
```

### Edge Cases

- `as` assertion bilan flag'ni "bypass" qilish — `return config[key] as string;` — xavfli, runtime'da undefined.toUpperCase() crash bo'ladi. TS xatoni yashiradi, lekin bug saqlanadi.
- `!` non-null assertion — `return config[key]!;` — aynan shu xavfli pattern. Faqat siz haqiqatan key mavjudligiga ishonsangiz.
- `in` operator narrowing: `if (key in config)` — TS bu yerda hali `noUncheckedIndexedAccess` natijasini narrow qilmaydi (TypeScript 5.x).
- `Object.hasOwn(config, key)` ham type narrowing qilmaydi — explicit `value !== undefined` check kerak.

### Follow-up savollar

1. "`as string` bilan tuzatish OKmi?" — Yo'q, type-system'ni aldash. Lint rule (`@typescript-eslint/no-unnecessary-type-assertion`) bunday assertion'larni topadi.
2. "`Map<string, string>.get(key)` bu flag'dan ta'sirlanadimi?" — Yo'q. `Map.prototype.get` har doim `V | undefined` qaytaradi (lib.d.ts declaration), flag'siz ham.

</details>

---

## Savol 14: Module/moduleResolution juftlik + exclude — 2 xato [Middle]

**Savol:** Bu tsconfig'da nechta xato bor?

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "node",
    "strict": true,
    "outDir": "dist"
  },
  "include": ["src"],
  "exclude": ["src/internal"]
}
```

```typescript
// src/app.ts
import { helper } from "./internal/helper";
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ikki xato: `module: "Node16"` + `moduleResolution: "node"` (legacy node10) — `"Node16"` o'zining resolution algoritmini talab qiladi. `exclude: ["src/internal"]` import qilingan faylni to'xtatmaydi — `exclude` faqat initial file scan'ni filter qiladi.

### To'liq tushuntirish

**Xato 1: Module/moduleResolution mos kelmaydi**

`module: "Node16"` Node.js modern semantics (ESM/CJS hybrid, `package.json` `exports`/`imports`, `.mts`/`.cts` extension) talab qiladi. `moduleResolution: "node"` (alias `node10`) — eski algoritm, `exports` field qo'llab-quvvatlanmaydi. Natija: modern package'larni resolve qilishda xato, kutilmagan import xulqi.

To'g'ri juftliklar:

```json
{ "module": "Node16", "moduleResolution": "Node16" }       // Node.js
{ "module": "NodeNext", "moduleResolution": "NodeNext" }   // Node.js (always-latest)
{ "module": "ESNext", "moduleResolution": "Bundler" }      // Vite/Webpack
{ "module": "CommonJS", "moduleResolution": "node" }       // Legacy Node CJS
```

**Xato 2: `exclude` import'ni bloklamaydi**

`exclude` faqat `include` glob'dan chiqarib tashlaydi (initial fayl ro'yxati). Lekin agar `include`'dagi fayl excluded fayldan import qilsa — TypeScript baribir excluded faylni compile qiladi (transitive dependency).

`src/app.ts` `include`'da → compile. U `./internal/helper` import qiladi → `src/internal/helper.ts` ham resolve qilinadi va compile bo'ladi, garchi `exclude`'da bo'lsa ham.

Yechim — haqiqatan blok qilish kerak bo'lsa:

- ESLint rule (`no-restricted-imports`) — code-level blok.
- `paths` bilan internal'ni ham yoritmaslik.
- Architecture (`packages/internal` private package).

### Kod misol

```json
// === ❌ Original ===
{
  "compilerOptions": {
    "module": "Node16",
    "moduleResolution": "node",   // ← Xato 1
    "strict": true,
    "outDir": "dist"
  },
  "include": ["src"],
  "exclude": ["src/internal"]     // ← Xato 2 (import to'xtatmaydi)
}

// === ✅ To'g'rilangan ===
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16", // ← Mos juftlik
    "strict": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
  // exclude olib tashlandi yoki test/dist uchun ishlatildi
}
```

```javascript
// === ESLint bilan internal import bloklash ===
// eslint.config.mjs
export default [
  {
    rules: {
      "no-restricted-imports": ["error", {
        patterns: [{
          group: ["**/internal/**"],
          message: "Internal modules are private."
        }]
      }]
    }
  }
];
```

### Edge Cases

- `exclude` test fayllar uchun ishlatilsa — `**/*.test.ts` excluded, ammo `src/user.ts` `./user.test` import qilsa baribir compile bo'ladi. Test fayllar alohida `tsconfig.test.json`'da bo'lishi afzal.
- `module: "ESNext"` + `moduleResolution: "node"` — eski Webpack 4 loyihalarda ko'rinadi, ammo modern packages buziladi. Migration: `moduleResolution: "Bundler"`.
- `moduleResolution: "node"` `node10`'ga rename qilingan (TS 5.0+), ammo backward compat uchun ikkala nom ishlaydi.
- `module: "NodeNext"` har TS minor release'da xulqi o'zgarishi mumkin — production'da `"Node16"` stable.

### Follow-up savollar

1. "`moduleResolution: "node"` ni qachon ishlatish mumkin?" — Faqat legacy CJS loyihalarda (`module: "CommonJS"`), `exports` field ishlatmaydigan eski package'lar bilan. Yangi kodda — yo'q.
2. "Real internal modul'ni qanday yashirish?" — Monorepo: `packages/internal` private (`"private": true`), `exports` field'da faqat public surface'ni export qilish.

</details>

---

## Savol 15: `verbatimModuleSyntax` qachon va nima uchun? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`verbatimModuleSyntax: true` — `import type`/`export type` markirovkasini majburiy qiladi. Type-only va value import/export aniq farqlanadi, emit predictable bo'ladi (har import statement saqlanadi yoki butunlay olib tashlanadi).

### To'liq tushuntirish

Default'da TS "tushunadi" qaysi import type-only — emit fazasida olib tashlaydi. Bu xulq:

- Predictable emas (TS versiyaga qarab o'zgarishi mumkin).
- Side-effect import (`import "./styles.css"`)'ni noto'g'ri olib tashlashi mumkin (rare bug).
- Bundler (esbuild/SWC) bitta faylni ko'rib bu qarorni qila olmaydi (cross-file context kerak).

`verbatimModuleSyntax: true` qoidalari:

- **Value import**: `import { createUser } from "./user"` — `createUser` value bo'lishi shart, emit'da saqlanadi.
- **Type import**: `import type { User } from "./user"` — type-only, emit'da olib tashlanadi.
- **Mixed**: `import { createUser, type User } from "./user"` — inline type marker (TS 5.0+).
- **Side-effect**: `import "./styles.css"` — har doim saqlanadi (no specifier).

Emit qoidasi:

- Statement type-only bo'lsa — butunlay olib tashlanadi.
- Aks holda — to'liq saqlanadi.

`isolatedModules` bilan birga ishlatish odatiy (modern bundler stack).

### Kod misol

```typescript
// === ❌ verbatimModuleSyntax: true bilan xato ===
import { User } from "./types"; // User faqat type bo'lsa — error
// Error: 'User' is a type and must be imported using a type-only import

// === ✅ Type-only import ===
import type { User } from "./types";
// Emit'da butunlay olib tashlanadi

// === ✅ Mixed (TS 5.0+) ===
import { createUser, type User } from "./types";
// createUser value, User type — inline marker

// === ✅ Value import ===
import { createUser } from "./types";
// Emit'da saqlanadi

// === ✅ Side-effect ===
import "./styles.css";       // Saqlanadi
import "reflect-metadata";    // Saqlanadi

// === Export ===
export type { User } from "./types";
export { createUser } from "./types";

// === ❌ CJS module'da ESM syntax ===
// tsconfig: module: "CommonJS", verbatimModuleSyntax: true
import { readFile } from "fs";
// Error: ESM syntax is not allowed in CommonJS module
// Yechim: import readFile = require("fs"); yoki module: "Node16"
```

### Edge Cases

- `verbatimModuleSyntax: true` `module: "CommonJS"` bilan birga — ESM syntax taqiqlanadi, faqat `import = require()` ishlaydi. Modern loyihada `module: "Node16"` ga o'tish afzal.
- Decorators (`@Injectable`) — class metadata uchun runtime reflection talab qilinadi. `verbatimModuleSyntax` bilan `import` decorator-emitted metadata uchun saqlanishi shart. Tip-aware decorator (NestJS) bilan ehtiyot.
- Re-export pattern (`export { User } from "./user"`) — `User` type yoki value bo'lishi `verbatimModuleSyntax` bilan strict aniqlanishi kerak (`export type { User }` agar type).
- `importsNotUsedAsValues` va `preserveValueImports` (eski flag'lar) `verbatimModuleSyntax` bilan birlashtirilgan — yangi kodda `verbatim` ishlatish.

### Follow-up savollar

1. "`verbatimModuleSyntax` yoqilganda code o'zgarishi qancha bo'ladi?" — Mavjud value-only import'lar `import type` ga aylantirilishi kerak. TS auto-fix (VSCode quick action) yordam beradi, ESLint rule (`@typescript-eslint/consistent-type-imports`) ham.
2. "Bu flag tree-shaking'ga ta'sir qiladimi?" — Bilvosita ha — emit predictable bo'lsa, bundler dead code elimination aniqroq.

</details>

---

## Savol 16: Project References — Output [Middle+]

**Savol:** Bu monorepo setup'da `tsc --build packages/api` chaqirilganda nima bo'ladi? Build tartibini ayting.

```
monorepo/
├── packages/
│   ├── shared/  (composite: true, no references)
│   ├── auth/    (composite: true, references: [shared])
│   └── api/     (composite: true, references: [shared, auth])
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Build tartibi: `shared` → `auth` → `api`. `tsc --build` dependency DAG'ni topologik tartibda build qiladi. Har project `.tsbuildinfo` cache'ni saqlaydi — qayta build'da faqat o'zgarganlar rebuild qilinadi.

### To'liq tushuntirish

`tsc --build packages/api` qadamlari:

1. **Resolve references**: `api/tsconfig.json` o'qiladi, `references: [{path: "../shared"}, {path: "../auth"}]` topiladi.
2. **Transitive resolution**: `auth/tsconfig.json` o'qiladi → uning `references: [{path: "../shared"}]` topiladi.
3. **Topological sort**: DAG quriladi — `shared` (no deps) → `auth` (deps: shared) → `api` (deps: shared, auth).
4. **Per-project incremental check**: Har project'da `.tsbuildinfo` cache solishtiriladi. Hech qaysi `.ts` fayl o'zgarmagan bo'lsa — build skip.
5. **Build order**: O'zgarganlar topologik tartibda compile qilinadi.
6. **Emit `.d.ts`**: Har project'ning `outDir`'ga `.d.ts` emit (cross-project type info).

Birinchi build'dan keyin `.tsbuildinfo` saqlanadi:

```
packages/shared/dist/.tsbuildinfo
packages/auth/dist/.tsbuildinfo
packages/api/dist/.tsbuildinfo
```

`tsc -b --watch` mode'da: file watcher har save'da o'zgargan project'larni topadi va incremental rebuild qiladi.

### Kod misol

```bash
# === Birinchi build (cold) ===
$ tsc --build packages/api
# Order:
# 1. shared compile → dist/, .tsbuildinfo
# 2. auth compile → dist/, .tsbuildinfo
# 3. api compile → dist/, .tsbuildinfo

# === Hech narsa o'zgartirmasdan qayta ===
$ tsc --build packages/api
# Output: All projects up-to-date (skip)

# === auth/src/index.ts'ni o'zgartirib ===
$ tsc --build packages/api
# Order:
# 1. shared up-to-date (skip)
# 2. auth rebuild (file changed)
# 3. api rebuild (depends on auth)

# === shared/src/index.ts'ni o'zgartirib ===
$ tsc --build packages/api
# Order:
# 1. shared rebuild
# 2. auth rebuild (depends on shared)
# 3. api rebuild (depends on shared, auth)

# === Verbose mode ===
$ tsc --build packages/api --verbose
# "Project 'shared' is out of date because output file '...' does not exist"
# "Building project '.../shared/tsconfig.json'..."

# === Force rebuild ===
$ tsc --build packages/api --force
# Cache e'tiborsiz, hammasi rebuild

# === Cache tozalash ===
$ tsc --build packages/api --clean
# .tsbuildinfo va outDir tozalanadi
```

### Edge Cases

- Circular reference (A → B → A) — `tsc -b` error: "Project references may not form a circular graph". Refactoring: shared kod alohida `common` package'ga.
- `.tsbuildinfo` corruption — agar fayl noto'g'ri saqlangan bo'lsa, `tsc -b --clean && tsc -b` bilan tozalash.
- `outDir` o'zgartirilsa `.tsbuildinfo` invalid bo'ladi — `--clean` bilan rebuild.
- Workspace symlink (`pnpm`/`yarn workspaces`) — runtime resolution uchun ishlaydi, ammo TS `references` orqali type-check qiladi. Ikki mexanizm parallel.

### Follow-up savollar

1. "`composite: true` yoqilmagan project'ga reference qilsam?" — `tsc -b` error: "Referenced project must have setting composite: true". Referenced project bu flag'siz `.tsbuildinfo` yarata olmaydi.
2. "Parallel build qanday ishlaydi?" — `tsc -b` o'zi parallel emas (sequential topological). `pnpm` workspace + `concurrently` yoki `turborepo` parallel orchestration uchun. TS 5.6+ da `--watch` mode'da partial parallelism.

</details>

---

## Savol 17: `useDefineForClassFields` qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useDefineForClassFields: true` (TS 4.x+ default `target: ES2022+` da) class field'larni `Object.defineProperty` semantics bilan emit qiladi (ES2022 native behavior). `false` — eski TS assignment semantics. Inheritance va accessor override'da farq qiladi.

### To'liq tushuntirish

ES2022 `class fields` proposal definite semantics belgilab berdi: har field `Object.defineProperty` orqali `[[Define]]`'ed (yangi own property yaratadi, accessor'larni override qiladi).

Eski TS (TS 3.7'gacha) — `[[Set]]` semantics (assignment, accessor'lar ishga tushadi).

Farq aniq inheritance + accessor pattern'da:

```typescript
class Base {
  get x() { return 10; }
  set x(v: number) { console.log("set", v); }
}

class Child extends Base {
  x = 5;
}

new Child();
// useDefineForClassFields: false → console.log: "set 5" (setter ishga tushadi)
// useDefineForClassFields: true  → setter ishga tushmaydi (define orqali yangi own property)
```

`target: "ES2022"` yoki yangiroq — `useDefineForClassFields: true` avtomatik default. Bu native ES2022 semantics'ga moslik.

Eski legacy kod (decorator + setter pattern) `false`'ni talab qilishi mumkin — explicit yozish kerak (`useDefineForClassFields: false`).

### Kod misol

```typescript
// === Farq pattern ===
class Base {
  _x = 0;
  get x() { return this._x; }
  set x(v: number) { this._x = v * 2; }
}

class Child extends Base {
  x = 100;
}

const c = new Child();

// === useDefineForClassFields: false ===
// Child constructor: this.x = 100
// → Setter ishga tushadi → this._x = 200
// c.x → getter → 200

// === useDefineForClassFields: true ===
// Child constructor: Object.defineProperty(this, "x", { value: 100, writable: true, enumerable: true, configurable: true })
// → Yangi own data property
// c.x → own property → 100
// Setter "shadowed" (yo'qotilmagan, lekin override qilingan)

// === declare bilan setter saqlash ===
class ChildSafe extends Base {
  declare x: number; // Type-only, runtime'da emit qilinmaydi
  constructor() {
    super();
    this.x = 100; // Setter ishga tushadi
  }
}

// === Decorator pattern (NestJS, Angular) ===
class OrderService {
  @Inject() private orderRepository!: OrderRepository;
  // Decorator runtime'da this.orderRepository'ga property descriptor o'rnatadi
  // useDefineForClassFields: true bo'lsa — class field bu property'ni
  // undefined bilan qayta define qilib injection'ni o'chiradi (bug)
  // Yechim: declare ishlatish yoki useDefineForClassFields: false
}
```

### Edge Cases

- `target: "ES2022"` default `useDefineForClassFields: true` — modern loyihada explicit yozish kerak emas.
- Eski Angular / NestJS decorator pattern — `useDefineForClassFields: false` zarur bo'lishi mumkin (decorator metadata class field assignment'idan oldin o'rnatilgan deb taxmin qiladi).
- `declare` modifier — TS field type'ini bildiradi, lekin runtime emit qilmaydi. `useDefineForClassFields: true` bilan inheritance hot-fix.
- `accessor` keyword (TS 4.9+) — `accessor x = 5;` — `[[Define]]` semantics bilan getter/setter generate qiladi. Modern alternative.

### Follow-up savollar

1. "Mavjud loyiha `target: "ES2020"` da, agar `"ES2022"` ga ko'tarsam nima buziladi?" — `useDefineForClassFields` default `true`'ga aylanadi. Decorator yoki inheritance pattern'da behavior o'zgarishi mumkin — har class'ni audit qilish.
2. "`declare` keyword qachon kerak?" — Field declaration kerak (TS type), ammo runtime emit kerak emas. Decorator-injected, parent-set, framework-managed field'larda.

<details>
<summary><strong>Deep Dive</strong></summary>

**ES2022 class fields semantic asoslari:** Spec `[[DefineOwnProperty]]` operation'ni belgilab beradi — har class field constructor body'da `Object.defineProperty(this, name, { value, writable: true, enumerable: true, configurable: true })` orqali set qilinadi. Bu `[[Set]]` (assignment) operatsiyasidan farqli — accessor (`set`) chaqirilmaydi, prototype chain ignored.

**TS emit farq:**

```javascript
// useDefineForClassFields: true (ES2022 native)
class Child extends Base {
  constructor() {
    super();
    Object.defineProperty(this, "x", { value: 100, writable: true, enumerable: true, configurable: true });
  }
}

// useDefineForClassFields: false (legacy TS)
class Child extends Base {
  constructor() {
    super();
    this.x = 100; // assignment — setter chaqiriladi
  }
}
```

**Inheritance accessor "shadowing" muammosi:** Parent class accessor (`get x()`) child class data field bilan override qilinganda — `[[Define]]` semantics own property yaratadi va prototype'dagi accessor "shadowed". `[[Set]]` semantics — setter chaqiriladi, accessor active qoladi. Legacy decorator framework'lar (Angular DI, NestJS) `[[Set]]` semantics'ga tayanadi.

**`declare` modifier internal:** `declare x: number` TS AST'da `ClassDeclaration.modifiers`'ga `DeclareKeyword` qo'shadi. Emit fazasi bu node'ni butunlay skip qiladi — runtime'da hech qanday property o'rnatish bo'lmaydi. Type-only contract. Decorator-aware framework'da parent yoki decorator metadata property'ni o'rnatadi, TS faqat type kontekstni biladi.

**`accessor` keyword (TS 4.9+):** `accessor x = 5` — auto-generated getter/setter pair bilan private backing field. TC39 decorators proposal'ning bir qismi (auto-accessor, Stage 3 — hali biror ES editiona kiritilmagan). Decorator integration: `@reactive accessor count = 0` — decorator getter/setter'ga ulanishi mumkin. Modern alternative `useDefineForClassFields` define semantics'ga.

**Migration strategy `target: "ES2020"` → `"ES2022"`:** Inheritance audit (parent accessor + child field), decorator framework versiya (NestJS 10+ class fields'ga moslashgan), `declare` qo'shish (DI injection points). `tsc --strict` audit'da uncovered behaviors topiladi.

</details>

</details>

---

## Savol 18: `isolatedDeclarations` (TS 5.5+) — Bug fix [Senior]

**Savol:** Bu library kod `isolatedDeclarations: true` yoqilganda compile bo'lmaydi. Tuzating.

```typescript
export function createUser(name: string, age: number) {
  return { id: Date.now(), name, age };
}

export class UserService {
  getUser(id: number) {
    return { id, name: "Ali" };
  }
}

export const config = {
  port: 3000,
  host: "localhost"
};
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedDeclarations` har exported function/method/variable'da explicit type annotation talab qiladi. Har export'ga explicit return type (function, method) yoki type annotation (variable) qo'shish kerak.

### To'liq tushuntirish

`isolatedDeclarations: true` — har `.ts` faylni alohida ko'rib `.d.ts` emit qilish imkonini beradi (parallel build, faster). Bunga erishish uchun har export'ning type'i fayl ichida explicit ko'rinishi kerak — TS faqat joriy faylni o'qib `.d.ts` yoza olishi kerak.

Default'da TS return type'ni inference qiladi — butun loyihani ko'rib. `isolatedDeclarations` bu inference'ni cheklaydi: har exported entity'da explicit type majburiy.

Foyda:

- **Parallel build**: Har fayl mustaqil `.d.ts` emit (SWC/esbuild parallel).
- **Build tezligi**: Cross-file inference yo'q.
- **API surface aniq**: Library author har export type'ini ko'rsatishga majbur.

Bu flag library publish loyihalari uchun ideal — `tsc` butunlay olib tashlanishi mumkin (SWC/esbuild `.js` + `.d.ts` emit).

### Kod misol

```typescript
// === ❌ Original — isolatedDeclarations error ===
export function createUser(name: string, age: number) {
  // Error: Function must have an explicit return type annotation
  return { id: Date.now(), name, age };
}

export class UserService {
  getUser(id: number) {
    // Error: Method must have an explicit return type annotation
    return { id, name: "Ali" };
  }
}

export const config = {
  // Error: Variable must have an explicit type annotation
  port: 3000,
  host: "localhost"
};

// === ✅ To'g'rilangan ===
interface User {
  id: number;
  name: string;
  age?: number;
}

export function createUser(name: string, age: number): User {
  return { id: Date.now(), name, age };
}

export class UserService {
  getUser(id: number): User {
    return { id, name: "Ali" };
  }
}

interface Config {
  port: number;
  host: string;
}

export const config: Config = {
  port: 3000,
  host: "localhost"
};

// === Alternative — satisfies bilan ===
export const configAlt = {
  port: 3000,
  host: "localhost"
} as const satisfies Config;
// satisfies — type-check + literal saqlash, isolatedDeclarations bilan mos
```

### Edge Cases

- `private` method'lar `isolatedDeclarations` cheklovidan tashqarida (`.d.ts`'ga emit qilinmaydi). Faqat `public` (default) va `protected` method'lar uchun explicit type majburiy.
- `function expression` overloading — `isolatedDeclarations` har overload signature'ni explicit talab qiladi.
- Generic type parameter inference — `function id<T>(x: T)` da return type explicit kerak (`<T>(x: T): T`).
- `JSX` complex inference (`useState`, `useMemo`) — explicit type ko'pincha qulayroq, ammo verbose. Library author balance qiladi.

### Follow-up savollar

1. "`isolatedDeclarations` `isolatedModules`'dan qanday farq qiladi?" — Birinchi `.d.ts` emit uchun (declaration), ikkinchi `.js` emit uchun (transpilation). Birga ishlatiladi — har fayl mustaqil `.js` + `.d.ts` emit.
2. "Bu flag eski library kod uchun qancha o'zgarish talab qiladi?" — Bog'liq: implicit return type ko'p — to'liq audit kerak. VSCode quick action ("Add return type") yordam beradi. Yangi loyihalarda boshidan yoqish afzal.

<details>
<summary><strong>Deep Dive</strong></summary>

**Motivation va architectural design:** Default TS declaration emit cross-file inference talab qiladi — `function loadUser() { return fetchUser(); }`'da `fetchUser`'ning return type'i boshqa fayldan keladi. `tsc` butun project graph'ni quradi — bu sekin, parallel emit imkonsiz. `isolatedDeclarations` constraint: har export'ning type'i fayl ichida explicit ko'rinishi shart.

**Non-TS tool'lar uchun spec:** Bu flag yoqilganda `.d.ts` syntactic transformation orqali generate qilinishi mumkin — type checker kerak emas. SWC va oxc (`oxc-transform`'ning `isolatedDeclaration()`) kabi parser-only tool'lar `.d.ts` emit qila oladi. Babel `.d.ts` emit'ini qo'llab-quvvatlamaydi (AST codegen yetishmaydi). TS 5.5'dan beri `isolatedDeclarations` constraint'i rasmiy.

**Constraint rules formal:**

1. Har exported function/method: explicit return type.
2. Har exported variable: explicit type annotation, yoki literal expression (TS literal'dan type infer qila oladi sintaktik).
3. Har exported class: har public method explicit return type, har property explicit type.
4. Generic parameter: explicit inference signature (`<T>(x: T): T`).
5. Re-export: type-only marker (`export type { User }`).

**`satisfies` operator with isolated declarations:**

```typescript
// ❌ Implicit type inference
export const config = { port: 3000 };

// ✅ satisfies — type-check + literal saqlash + isolatedDeclarations OK
export const config = { port: 3000 } as const satisfies Config;

// ✅ Explicit type annotation
export const config: Config = { port: 3000 };
```

`satisfies` syntactic — TS faqat fayl ichidagi expression'ni ko'radi.

**Performance trade-off:** `tsc`'ning cross-file declaration emit'i butun project graph'ni quradi (sekin, parallel emas). `isolatedDeclarations` constraint'i ostida har `.d.ts` faqat o'z faylidan generate qilinadi — oxc/SWC kabi native tool'lar uni TS'dan ancha tezroq va parallel emit qila oladi. Aniq tezlik loyiha hajmi va fayl soniga bog'liq; native parallel emit `tsc`'dan sezilarli ustun.

**Library author perspective:** Explicit type — API surface dokumentatsiya sifatida. Implicit return type'lar API consumer uchun "noaniq" — `tsc` har versiya bilan inference o'zgarishi mumkin (breaking change semver bo'yicha). Explicit type — stable API contract.

**Limitation:** Generic'da return type baribir explicit yozilishi shart — `function identity<T>(x: T): T` ishlaydi, ammo `function identity<T>(x: T)` (return type'siz) `isolatedDeclarations` ostida xato beradi. Function overload — har signature alohida explicit.

</details>

</details>

---

## Xulosa

- `target` — JS output syntax, `lib` — type-checking uchun built-in API declaration'lar
- `strict` — meta-flag, 8 sub-flag birga (`noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`, `useUnknownInCatchVariables`)
- `noUncheckedIndexedAccess` — index access `T | undefined`, alohida yoqilishi kerak (`strict`'ga kirmaydi)
- `exactOptionalPropertyTypes` — `x?` (missing) va `x: T | undefined` (present, undefined) farqlanadi
- Module/moduleResolution juftliklari: Node16/Node16 (Node.js), ESNext/Bundler (Vite/Webpack), CJS/node10 (legacy)
- `paths` — faqat type resolution, runtime'da har bundler/tool alohida sozlash, library'da xavfli
- `erasableSyntaxOnly` — `enum`, `namespace`, parameter property taqiqlanadi (Node.js native TS strip uchun)
- `isolatedModules` — fayl-by-fayl transpilation (SWC/esbuild), `const enum` va ambiguous re-export taqiqlanadi
- `isolatedDeclarations` (TS 5.5+) — explicit return type majburiy, parallel `.d.ts` emit
- Project references — `composite: true` majburiy, DAG (no circular), `tsc --build` topological order
- `useDefineForClassFields` — class field `[[Define]]` vs `[[Set]]` semantics (inheritance/decorator pattern)
- Library tsconfig: `target: "ES2020"`, `declarationMap`, `stripInternal`, `isolatedDeclarations`, `paths`'siz
