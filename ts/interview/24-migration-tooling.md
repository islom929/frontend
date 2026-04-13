# Interview: Migration va Tooling

> JS → TS migration strategiyasi, allowJs/checkJs, JSDoc, @ts-expect-error, SWC/esbuild, tsx, Node.js strip-types, type-coverage bo'yicha interview savollari.

---

## Nazariy savollar

### 1. JS → TS migration bosqichlari? "Big bang" vs "gradual"?

<details>
<summary>Javob</summary>

Gradual (bosqichma-bosqich) — yagona to'g'ri yondashuv:

1. `tsconfig.json` + `allowJs: true` — hech narsa o'zgarmaydi, TS tooling ishlaydi
2. `checkJs: true` — JS fayllarni TS tekshiradi (JSDoc orqali)
3. Muhim fayllarni `.js` → `.ts`
4. Yangi kod faqat `.ts` da
5. Eski fayllarni ketma-ket migrate
6. `allowJs: false` — butun loyiha TS
7. `strict: true` — to'liq type safety

**"Big bang"** — bir PR da hamma narsa. 500+ fayl, review mumkin emas, merge conflict, katta rollback. Har bosqich **alohida PR** bo'lishi kerak.

</details>

### 2. `allowJs` va `checkJs` farqi?

<details>
<summary>Javob</summary>

**`allowJs: true`** — `.js` va `.ts` birgalikda import/export. **Tekshiruv yo'q**.
**`checkJs: true`** — JS fayllarni ham **type check** qiladi (inference + JSDoc).

**Per-file override:**
- `// @ts-check` — shu faylni tekshir (opt-in)
- `// @ts-nocheck` — shu faylni o'tkazib yubor

**Strategiya:** `checkJs: false` + tayyor fayllarga `// @ts-check` — "opt-in" model.

</details>

### 3. "Leaf-first" migration nima?

<details>
<summary>Javob</summary>

Dependency tree ning **leaf** fayllaridan boshlash:

```
models/user.js      # 1-BIRINCHI (leaf — import yo'q)
utils/helpers.js    # 1-BIRINCHI (leaf)
services/user.js    # 2-keyinroq
routes/userRoutes.js # 3-keyinroq
app.js              # OXIRGI (entry point)
```

**Nima uchun leaf dan:** dependency yo'q — mustaqil migrate, boshqa fayllarni buzmaydi, kichik PR lar. Entry point dan boshlash — cascade xatolar, barchasi buziladi.

</details>

### 4. `@ts-ignore` va `@ts-expect-error` farqi?

<details>
<summary>Javob</summary>

| | `@ts-ignore` | `@ts-expect-error` |
|---|---|---|
| Xatoni yashiradi | ✅ | ✅ |
| Xato **bo'lmasa** | Hech narsa demaydi | **Yangi xato beradi** |
| Ishlatish | ❌ Xavfli | ✅ To'g'ri |

```typescript
// @ts-expect-error — xato tuzatilganda ESLATADI
const w: number = 42; // ❌ "Unused '@ts-expect-error' directive"
```

Doim `@ts-expect-error` + izoh: `// @ts-expect-error — TODO: JIRA-1234 fix types`

</details>

### 5. SWC/esbuild — "transpile fast, check separately"?

<details>
<summary>Javob</summary>

**SWC** (Rust) — tsc dan 20-70x tez. **esbuild** (Go) — bundler + transpiler. Ikkalasi **type check qilmaydi**:

```json
{
  "scripts": {
    "build": "swc src -d dist",
    "type-check": "tsc --noEmit",
    "ci": "npm run type-check && npm run build"
  }
}
```

| Tool | Vaqt (1000 fayl) |
|------|-----------------|
| tsc (check + emit) | ~15s |
| SWC (transpile) | ~0.3s |
| esbuild (transpile) | ~0.1s |

**Muhim:** SWC/esbuild ishlatib type check ni **unutmaslik** — CI da `tsc --noEmit` albatta.

</details>

### 6. `ts-node`, `tsx`, Node.js strip-types farqi?

<details>
<summary>Javob</summary>

| Tool | Type check | Tezlik | Mechanism |
|------|:----------:|--------|-----------|
| `ts-node` | ✅ | Sekin | tsc API |
| `tsx` | ❌ | Tez | esbuild |
| Node.js strip-types | ❌ | Eng tez | Annotation olib tashlash |

```bash
npx ts-node src/index.ts           # to'liq, sekin
npx tsx src/index.ts                # tez
node --experimental-strip-types src/index.ts  # eng tez
```

Node.js strip-types cheklovi: `enum`, `namespace`, parameter property **ishlamaydi**. Node 22.18+/23.6+ da flag siz ishlaydi.

**Tavsiya:** Dev da `tsx`, CI/prod da `tsc` build + `node`.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. JSDoc bilan type checking (Daraja: Middle)

**Savol:** `// @ts-check` bilan JS faylda type-safe kod yozing — typedef, generic, import type:

<details>
<summary>Yechim</summary>

```javascript
// @ts-check

/** @typedef {{ id: number; name: string; email: string }} User */

/**
 * @param {string} name
 * @param {string} email
 * @returns {User}
 */
function createUser(name, email) {
  return { id: Date.now(), name, email };
}

/** @template T
 * @param {T[]} arr
 * @param {(item: T) => boolean} predicate
 * @returns {T | undefined}
 */
function findItem(arr, predicate) { return arr.find(predicate); }

/** @type {import("./models/user").User} */
let currentUser;

const canvas = /** @type {HTMLCanvasElement} */ (document.getElementById("c"));
```

`checkJs: true` bo'lganda TS JSDoc ni type information sifatida o'qiydi — `.ts` ga o'tkazmasdan type safety.

</details>

### 2. Migration tsconfig xatolar (Daraja: Middle)

**Savol:** Migration uchun bu tsconfig da 3 ta xato bor:

```json
{
  "compilerOptions": {
    "module": "Node16",
    "allowJs": true, "checkJs": true,
    "strict": true,
    "skipLibCheck": false,
    "incremental": false
  }
}
```

<details>
<summary>Yechim</summary>

1. **`strict: true` + migration boshida** — yuzlab xato beradi. `strict: false`, kerakli flag larni alohida yoqish
2. **`skipLibCheck: false`** — node_modules `.d.ts` tekshiradi, sekin va keraksiz
3. **`incremental: false`** — har safar to'liq recompile, sekin

```json
{
  "module": "Node16", "moduleResolution": "Node16",
  "allowJs": true, "checkJs": true,
  "strict": false, "noImplicitAny": true,
  "skipLibCheck": true, "incremental": true
}
```

</details>

### 3. `const enum` + SWC xato (Daraja: Middle)

**Savol:** Bu build nima uchun ishlamaydi?

```json
{ "scripts": { "build": "swc src -d dist" } }
```

```typescript
const enum Direction { Up, Down, Left, Right }
export function move(dir: Direction): string { return `Moving ${dir}`; }
```

<details>
<summary>Yechim</summary>

1. **`const enum` SWC bilan ishlamaydi** — SWC fayl-by-fayl transpile, `const enum` cross-file inline kerak
2. **`type-check` script yo'q** — SWC type check qilmaydi

```typescript
// const enum → oddiy enum yoki as const
const Direction = { Up: 0, Down: 1, Left: 2, Right: 3 } as const;
type Direction = (typeof Direction)[keyof typeof Direction];
```

```json
{ "scripts": { "build": "swc src -d dist", "type-check": "tsc --noEmit" } }
```

</details>

### 4. `type-coverage` — migration progress (Daraja: Middle)

**Savol:** `type-coverage` nima? CI da qanday ishlatiladi?

<details>
<summary>Yechim</summary>

Loyihadagi qancha identifier aniq type ga ega — `any` coverage ni tushiradi:

```bash
npx type-coverage              # 85.23% (4521/5305)
npx type-coverage --detail     # Qayerda any bor
npx type-coverage --at-least 80 # CI da minimum
```

| Coverage | Daraja |
|----------|--------|
| < 50% | Migration boshlanmagan |
| 70-85% | Yaxshi |
| 95%+ | Production-ready |

**`any` bosqichma-bosqich kamaytirish:**
1. `input: any` → `input: string` (input type)
2. `return any` → `return ParsedData` (output type)
3. `JSON.parse` → `Schema.parse` (runtime validation)

</details>

### 5. `isolatedModules` vs `isolatedDeclarations` (Daraja: Middle+)

**Savol:** Farqini tushuntiring — qachon qaysi biri?

<details>
<summary>Yechim</summary>

**`isolatedModules`** — har fayl mustaqil **transpile**. SWC/esbuild uchun:
- `const enum` taqiq
- Type-only re-export `export type` kerak

**`isolatedDeclarations` (TS 5.5+)** — har fayl mustaqil **declaration emit**. Export da explicit type kerak:

```typescript
// isolatedModules xato:
const enum Dir { Up } // ❌
export { User };       // ❌ agar faqat type

// isolatedDeclarations xato:
export function add(a: number, b: number) { return a + b; } // ❌ return type kerak
export function add(a: number, b: number): number { return a + b; } // ✅
```

`isolatedModules` → transpilation uchun. `isolatedDeclarations` → declaration emit uchun. Ikkalasi parallel build va 3rd party tool lar uchun.

</details>

---

## Xulosa

- Gradual migration — leaf-first, bosqichma-bosqich PR lar
- `allowJs` → accept, `checkJs` → type check JS fayllar
- `@ts-expect-error` > `@ts-ignore` — xato tuzatilganda eslatadi
- SWC/esbuild — transpile tez, type check alohida (`tsc --noEmit`)
- `tsx` — dev uchun eng qulay, Node.js strip-types — eng tez
- `type-coverage` — migration progress o'lchash, CI da threshold
- `const enum` — SWC/esbuild bilan ishlamaydi, `as const` object afzal

[Asosiy bo'limga qaytish →](../24-migration-tooling.md)
