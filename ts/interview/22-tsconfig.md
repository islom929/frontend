# Interview: tsconfig.json

> target vs lib, paths/baseUrl, exactOptionalPropertyTypes, erasableSyntaxOnly, project references, common mistakes va optimal config bo'yicha interview savollari. strict — [interview/01](01-ts-intro.md), moduleResolution — [interview/17](17-modules.md).

---

## Nazariy savollar

### 1. `target` va `lib` orasidagi farq nima?

<details>
<summary>Javob</summary>

**`target`** — qaysi JS versiyasiga **compile** qiladi (output syntax).
**`lib`** — qaysi built-in **type** lar mavjud (compile-time da).

```json
// ES5 ga compile, lekin Promise polyfill bor
{ "target": "ES5", "lib": ["ES2015", "DOM"] }

// Browser
{ "target": "ES2022", "lib": ["ES2022", "DOM", "DOM.Iterable"] }

// Node.js — DOM kerak EMAS
{ "target": "ES2022", "lib": ["ES2022"] }
```

`lib` ko'rsatilmasa — `target` ga mos default. `target: "ES5"` da `async/await` generators ga downlevel, `ES2017+` da saqlanadi. **Farq:** `target` = JS output, `lib` = qaysi API lar "bor".

</details>

### 2. `exactOptionalPropertyTypes` nima?

<details>
<summary>Javob</summary>

`exactOptionalPropertyTypes: true` — `x?: string` va `x: string | undefined` **farqli**:

- `x?: string` — property **mavjud bo'lmasligi** mumkin (missing)
- `x: string | undefined` — property **mavjud**, qiymati `undefined`

```typescript
interface User { name: string; nickname?: string; }

const u1: User = { name: "Ali" };                       // ✅ nickname yo'q
const u2: User = { name: "Ali", nickname: "Al" };        // ✅
// const u3: User = { name: "Ali", nickname: undefined }; // ❌ Error!
// nickname?: string = yo'q bo'lishi mumkin, undefined EMAS
```

Farq muhim: `Object.keys`, `"key" in obj`, database da "field yo'q" vs "field null" — semantic farq bor. `strict: true` ga **kirmaydi** — alohida yoqish kerak.

</details>

### 3. `paths` va `baseUrl` — eng keng tarqalgan xato?

<details>
<summary>Javob</summary>

`paths` — import alias. Lekin faqat **TS type resolution** uchun:

```json
{ "baseUrl": ".", "paths": { "@/*": ["src/*"] } }
```

```typescript
import { Button } from "@components/Button"; // TS OK ✅
```

**Eng katta xato:** `paths` runtime da **ishlamaydi**! JS output da alias **o'zgarmaydi**:

```javascript
import { Button } from "@components/Button"; // Runtime CRASH ❌
```

**Har bir tool da alohida sozlash kerak:**
- Vite: `resolve.alias`
- Webpack: `resolve.alias`
- Jest: `moduleNameMapper`
- Node.js: `tsconfig-paths` yoki `tsx`

Library publish da `paths` **xavfli** — `.d.ts` da resolve qilinmagan alias qoladi.

</details>

### 4. `erasableSyntaxOnly` (TS 5.8+) nima?

<details>
<summary>Javob</summary>

Faqat **o'chirib tashlanadigan** TS syntax ga ruxsat. Runtime semantics qo'shadigan construct larni taqiqlaydi:

```typescript
type User = { name: string };         // ✅ erasable
interface Config { port: number }      // ✅ erasable

// enum Direction { Up, Down }         // ❌ Runtime semantics
// namespace Utils { }                  // ❌ Runtime semantics
// class A { constructor(public x: number) {} } // ❌ Parameter property
```

**Nima uchun kerak:** Node.js 22+ `--experimental-strip-types` — TS fayllarni type larni o'chirib to'g'ridan-to'g'ri run qiladi. Lekin faqat "type olib tashlash" — `enum`, `namespace`, parameter property handle qila olmaydi. `erasableSyntaxOnly` bu mos kelishni kafolatlaydi.

```json
{ "erasableSyntaxOnly": true, "verbatimModuleSyntax": true }
```

</details>

### 5. Project References — monorepo da qanday ishlaydi?

<details>
<summary>Javob</summary>

Har package o'z `tsconfig.json` ga ega, `references` bilan dependency e'lon qilinadi:

```
monorepo/
├── packages/shared/   ← boshqalar depend
├── packages/api/      ← shared ga depend
└── tsconfig.json      ← root
```

```json
// packages/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "composite": true, "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../shared" }]
}
```

```bash
tsc --build           # Barcha to'g'ri tartibda
tsc -b packages/api   # Bitta + dependency lari
```

**`composite: true`** majburiy — `incremental: true` + `declaration: true` implicit yoqiladi. `.tsbuildinfo` da oldingi build saqlanadi, faqat o'zgarganlar rebuild — **50-80%** tezroq. Circular reference taqiqlangan (DAG bo'lishi shart).

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `isolatedModules` xato — toping (Daraja: Middle)

**Savol:** `isolatedModules: true` da bu kodda nechta xato bor?

```typescript
const enum Direction { Up = "UP", Down = "DOWN" }
export { Direction };
export { User } from "./types";
```

<details>
<summary>Yechim</summary>

**Ikki xato:**

1. **`const enum` taqiqlangan** — bundler lar boshqa fayldagi `const enum` qiymatlarini bilmaydi:
```typescript
enum Direction { Up = "UP", Down = "DOWN" } // ✅ oddiy enum
```

2. **`export { User }` — type mi, value mi noma'lum** — bundler bitta faylni ko'rganda bilmaydi:
```typescript
export type { User } from "./types"; // ✅ explicit type re-export
```

`isolatedModules` har qanday bundler (Vite, esbuild, SWC) da yoqish kerak — ular faylni izolyatsiyada compile qiladi.

</details>

### 2. `noUncheckedIndexedAccess` — compile xato (Daraja: Middle+)

**Savol:** Bu kod `noUncheckedIndexedAccess: true` da compile bo'ladimi?

```typescript
const config: Record<string, string> = { host: "localhost", port: "3000" };

function getConfig(key: string): string {
  const value = config[key];
  return value;
}
```

<details>
<summary>Yechim</summary>

**Compile bo'lmaydi:**

```
Error: Type 'string | undefined' is not assignable to type 'string'.
```

`noUncheckedIndexedAccess: true` da `config[key]` type i `string | undefined`. Lekin return type `string`.

```typescript
// ✅ Tuzatish 1 — undefined check
function getConfig(key: string): string {
  const value = config[key];
  if (value === undefined) throw new Error(`Key not found: ${key}`);
  return value;
}

// ✅ Tuzatish 2 — default value
function getConfig(key: string, defaultValue = ""): string {
  return config[key] ?? defaultValue;
}
```

</details>

### 3. tsconfig juftlik xato — toping (Daraja: Middle)

**Savol:** Bu tsconfig da nechta xato bor?

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
<summary>Yechim</summary>

**Ikki xato:**

1. **`module: "Node16"` + `moduleResolution: "node"` nomutanosib.** `"node"` — eski `node10`. To'g'risi:
```json
{ "module": "Node16", "moduleResolution": "Node16" }
```

2. **`exclude` import ni TO'XTATMAYDI.** `exclude: ["src/internal"]` faqat initial file scan ni filter qiladi. `src/app.ts` dan `src/internal/helper.ts` import qilinsa — **baribir compile bo'ladi**. `exclude` `import` ni bloklamaydi.

Mos juftliklar: `Node16`/`Node16`, `ESNext`/`Bundler`, `NodeNext`/`NodeNext`.

</details>

### 4. Library tsconfig xatolarini toping (Daraja: Senior)

**Savol:** Bu config library publish uchun. 5 ta muammo toping:

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
<summary>Yechim</summary>

1. **`rootDir` yo'q** — output tuzilmasi oldindan aytib bo'lmaydi. `"rootDir": "src"` kerak.

2. **`declarationMap` yo'q** — "Go to Definition" `.d.ts` ga tushadi, `.ts` ga emas. `"declarationMap": true` kerak.

3. **`moduleResolution: "Bundler"` library uchun noto'g'ri** — consumer Node.js da bundler siz ishlatishi mumkin. Library uchun `"module": "Node16", "moduleResolution": "Node16"`.

4. **`paths` library da xavfli** — `.d.ts` da resolve qilinmagan `@/*` qoladi. Consumer da broken path. Library da `paths` ishlatmaslik yaxshi.

5. **`skipLibCheck` yo'q** — compile tezligini 30-50% oshiradi: `"skipLibCheck": true`.

</details>

### 5. Optimal tsconfig — project turiga qarab (Daraja: Middle+)

**Savol:** React App (Vite), Node.js API, va Library uchun optimal tsconfig ning asosiy farqlari?

<details>
<summary>Yechim</summary>

**React App (Vite):**
```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "ESNext", "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"], "jsx": "react-jsx",
    "strict": true, "noUncheckedIndexedAccess": true,
    "noEmit": true,                   // ← Vite compile qiladi
    "verbatimModuleSyntax": true, "skipLibCheck": true
  }
}
```

**Node.js API:**
```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "Node16", "moduleResolution": "Node16",
    "lib": ["ES2022"],                // ← DOM yo'q
    "strict": true, "noUncheckedIndexedAccess": true,
    "outDir": "dist", "rootDir": "src",
    "declaration": true, "sourceMap": true, "incremental": true, "skipLibCheck": true
  }
}
```

**Library:**
```json
{
  "compilerOptions": {
    "target": "ES2020", "module": "Node16", "moduleResolution": "Node16",
    "strict": true,
    "outDir": "dist", "rootDir": "src",
    "declaration": true, "declarationMap": true, // ← Go to Definition
    "sourceMap": true, "stripInternal": true,     // ← @internal yashirish
    "skipLibCheck": true
  }
}
```

**Asosiy farqlar:** React da `noEmit` (bundler compile), Node.js da `module: "Node16"`, Library da `declarationMap` + `stripInternal`.

</details>

---

## Xulosa

- `target` = JS output syntax, `lib` = qaysi API type lar mavjud
- `exactOptionalPropertyTypes` — `x?` va `x | undefined` farqli. `strict` ga kirmaydi
- `paths` faqat type resolution — runtime da ishlamaydi, har tool da alohida sozlash
- `erasableSyntaxOnly` — Node.js native TS uchun, enum/namespace/parameter property taqiqlanadi
- Project References — monorepo da incremental build, `composite: true` majburiy
- Library da `moduleResolution: "Node16"`, `declarationMap`, `paths` dan qochish
- `noUncheckedIndexedAccess` — `strict` ga kirmaydi, alohida yoqish tavsiya
- `strict`, `moduleResolution`, `isolatedModules`, `verbatimModuleSyntax` — batafsil [interview/01](01-ts-intro.md), [interview/17](17-modules.md)

[Asosiy bo'limga qaytish →](../22-tsconfig.md)
