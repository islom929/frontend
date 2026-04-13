# Interview: Declaration Files

> `.d.ts` fayllar, `declare` keyword, declaration generate qilish, `isolatedDeclarations`, DefinitelyTyped, `typeRoots`/`types`, library publishing bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Declaration file (`.d.ts`) nima? Nima uchun kerak?

<details>
<summary>Javob</summary>

`.d.ts` — JavaScript kodiga **type information** beradigan fayl. Faqat type lar, signature lar — implementation (body) **yo'q**:

```typescript
// utils.ts — to'liq implementation
export function formatDate(date: Date): string {
  return date.toISOString().split("T")[0];
}

// utils.d.ts — faqat type
export declare function formatDate(date: Date): string;
```

**Qachon kerak:**
1. **JS kutubxona uchun type** — kutubxona JS da, TS da ishlatmoqchisiz
2. **Library publish** — TS library `.js` + `.d.ts` sifatida distribute
3. **Global type lar** — `window`, `process` kabi
4. **Non-TS fayllar** — CSS, image import qilganda

`.d.ts` JS ga **compile qilinmaydi** — faqat type-checking uchun o'qiladi.

</details>

### 2. `declare` keyword nima? Qanday ishlatiladi?

<details>
<summary>Javob</summary>

`declare` — "bu narsa **boshqa joyda** mavjud, men faqat type ini aytayapman". JS ga **emit qilinmaydi**:

```typescript
declare const __DEV__: boolean;             // global variable
declare function structuredClone<T>(value: T): T; // global function
declare class EventEmitter {                // tashqi class
  on(event: string, listener: (...args: any[]) => void): this;
}
declare module "lodash" {                   // module type
  export function chunk<T>(array: T[], size: number): T[][];
}
declare global {                            // global kengaytirish
  interface Window { __STORE__: any; }
}
```

| Construct | Ishlatilish |
|-----------|------------|
| `declare const/let/var` | Tashqi variable |
| `declare function` | Tashqi function |
| `declare class` | Tashqi class |
| `declare module` | Module type |
| `declare namespace` | Type lar guruhi |
| `declare global` | Global scope kengaytirish |

</details>

### 3. Library publish qilganda `.d.ts` qanday generate qilinadi?

<details>
<summary>Javob</summary>

`tsconfig.json` da `declaration: true`:

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./dist/types",
    "declarationMap": true,
    "emitDeclarationOnly": true
  }
}
```

**`package.json` da:**

```json
{
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

**Muhim:** `exports` da `"types"` **birinchi** bo'lishi kerak — TS birinchi mos condition ni oladi.

</details>

### 4. DefinitelyTyped nima? `@types` package lar qanday ishlaydi?

<details>
<summary>Javob</summary>

**DefinitelyTyped** — community type definitions repositoriyasi. JS kutubxona o'zida type bo'lmasa — `@types/` da install:

```bash
npm install --save-dev @types/express @types/lodash @types/node
```

**Type qidiruv tartibi:**
1. Package o'zida → `package.json` `"types"` field
2. `@types` package → `node_modules/@types/`
3. `typeRoots` → tsconfig da belgilangan papkalar

**Versiya:** Major + minor versiya mos bo'lishi kerak. Agar library o'zida type lar bo'lsa — `@types` **kerak emas** (conflict yaratadi).

</details>

### 5. `typeRoots` va `types` tsconfig da farqi?

<details>
<summary>Javob</summary>

| Option | Nima qiladi | Default |
|--------|-------------|---------|
| `typeRoots` | Type qidiriladigan **papkalar** | `["./node_modules/@types"]` |
| `types` | Include qilinadigan aniq **package lar** | Barcha `@types/*` |

```json
// typeRoots — qo'shimcha papka qo'shish
{ "typeRoots": ["./node_modules/@types", "./custom-types"] }

// types — faqat shu package lar
{ "types": ["node", "vitest/globals"] }
// @types/jest EXCLUDE — conflict oldini olish
```

`types` belgilasangiz — faqat shu package lar include, qolganlari ignore. Global scope pollution oldini oladi.

</details>

### 6. `isolatedDeclarations` (TS 5.5+) nima?

<details>
<summary>Javob</summary>

Har fayl `.d.ts` sini **boshqa fayllar type ma'lumotisiz** generate qilish. `isolatedModules` ning declaration versiyasi.

**Cheklov — exported code da explicit types kerak:**

```typescript
// ❌ return type inference kerak
export function add(a: number, b: number) { return a + b; }

// ✅ explicit return type
export function add(a: number, b: number): number { return a + b; }

// ❌ variable type inference
export const config = { host: "localhost", port: 3000 };

// ✅ explicit type
export const config: { host: string; port: number } = { host: "localhost", port: 3000 };
```

Local (unexported) code cheklanmaydi.

| | `declaration: true` | `isolatedDeclarations: true` |
|---|--------------------|-----------------------------|
| Tezlik | Sekin (butun project) | Tez (fayl-bo'yicha) |
| Parallelization | ❌ | �� |
| Explicit types | Ixtiyoriy | Majburiy (exported) |

**Qachon:** Katta monorepo, esbuild/SWC bilan build — fayl-bo'yicha parallel declaration emit.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `.d.ts` da qaysi biri to'g'ri? (Daraja: Junior+)

**Savol:** Qaysilari `.d.ts` da to'g'ri, qaysilari xato?

```typescript
// A
export function greet(name: string): string {
  return `Hello, ${name}`;
}

// B
export declare function greet(name: string): string;

// C
export interface User { id: number; name: string; }

// D
declare module "my-lib" {
  export function greet(name: string): string;
}
```

<details>
<summary>Yechim</summary>

- **A — ❌** `.d.ts` da function body bo'lishi mumkin emas
- **B — ✅** Function declaration, `declare` bilan, faqat signature
- **C — ✅** Interface — zaten faqat type, body yo'q
- **D — ✅** Module declaration

`.d.ts` da **implementation yo'q** — faqat type signature. Interface, type alias — doim to'g'ri. Function, class, variable — `declare` bilan, body siz.

</details>

### 2. `declarationMap` nima uchun kerak? (Daraja: Middle)

**Savol:** `declarationMap: true` va `false` da "Go to Definition" qayerga olib boradi?

<details>
<summary>Yechim</summary>

```
declarationMap: false (default):
  "Go to Definition" → dist/utils.d.ts (faqat signature)

declarationMap: true:
  "Go to Definition" → src/utils.ts (to'liq source code)
```

`declarationMap` `.d.ts.map` fayl generate qiladi — IDE ga `.d.ts` dan original `.ts` ga map beradi.

**Kerak:** Library development, monorepo (package lar orasida navigate), debugging.
**Kerak emas:** Closed-source library, production build (qo'shimcha hajm).

</details>

### 3. `package.json` types xato — tartib (Daraja: Middle+)

**Savol:** Bu konfiguratsiyada nima xato? Type lar topilmaydi:

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

<details>
<summary>Yechim</summary>

**Xato:** `"types"` oxirda — `"import"` birinchi mos keladi. TS `.mjs` ni type sifatida o'qishga urinadi — topilmaydi!

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

`"types"` **doim birinchi**. TS `exports` da birinchi mos condition ni oladi. Tekshirish: `npx @arethetypeswrong/cli --pack .`

</details>

### 4. `.d.ts` output yozing (Daraja: Middle)

**Savol:** Bu TS kodning `.d.ts` output ini yozing:

```typescript
export type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

export interface RequestConfig {
  url: string;
  method: HttpMethod;
  headers?: Record<string, string>;
}

export async function request<T>(config: RequestConfig): Promise<T> {
  const response = await fetch(config.url, { method: config.method });
  return response.json();
}

export class ApiClient {
  private baseUrl: string;
  constructor(baseUrl: string) { this.baseUrl = baseUrl; }
  get<T>(path: string): Promise<T> {
    return request({ url: `${this.baseUrl}${path}`, method: "GET" });
  }
}
```

<details>
<summary>Yechim</summary>

```typescript
export type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

export interface RequestConfig {
  url: string;
  method: HttpMethod;
  headers?: Record<string, string>;
}

export declare function request<T>(config: RequestConfig): Promise<T>;

export declare class ApiClient {
  private baseUrl: string;
  constructor(baseUrl: string);
  get<T>(path: string): Promise<T>;
}
```

**O'zgarganlar:**
- Function/method body → o'chirildi
- `declare` keyword → avtomatik qo'shildi
- `async` → yo'q (return type `Promise<T>` allaqachon bor)
- `type`, `interface` → o'zgarishsiz
- `private baseUrl: string` → saqlanadi (tsc private type emit qiladi)

</details>

### 5. Qo'lda `.d.ts` xatolari — toping (Daraja: Middle+)

**Savol:** Bu qo'lda yozilgan `.d.ts` da 3 ta xato bor. Toping:

```typescript
declare function fetch(input: string | Request, init?: RequestInit): Promise<Response>;
declare function fetch(input: string): Promise<Response>;

declare class Builder {
  setName(name: string): Builder;
}

declare function get<T>(obj: T, key: string): any;
```

<details>
<summary>Yechim</summary>

**Xato 1 — Overload tartibi noto'g'ri:** Umumiy signature birinchi — doim mos keladi, aniq signature hech qachon tanlanmaydi:

```typescript
// ✅ Aniqroq birinchi
declare function fetch(input: string): Promise<Response>;
declare function fetch(input: string | Request, init?: RequestInit): Promise<Response>;
```

**Xato 2 — `this` return type yo'q:** Method chaining buziladi subclass da:

```typescript
// ✅ this type
declare class Builder {
  setName(name: string): this;
}
```

**Xato 3 — Generic juda keng:** `key: string` + `any` return — type safety yo'q:

```typescript
// ✅ Type-safe
declare function get<T, K extends keyof T>(obj: T, key: K): T[K];
```

</details>

---

## Xulosa

- `.d.ts` — faqat type, implementation yo'q. JS ga compile qilinmaydi
- `declare` — "boshqa joyda mavjud, faqat type aytayapman". Emit qilinmaydi
- `declaration: true` — tsc avtomatik `.d.ts` generate qiladi
- `isolatedDeclarations` (TS 5.5+) — fayl-bo'yicha parallel, explicit types majburiy
- DefinitelyTyped / `@types/*` — JS kutubxona larga community type lar
- `typeRoots` — type papkalar, `types` — aniq package lar (conflict oldini olish)
- `package.json` `exports` da `"types"` **doim birinchi**
- `declarationMap` — "Go to Definition" da source code ga o'tish
- Module augmentation — batafsil [interview/17 — Modules](17-modules.md)

[Asosiy bo'limga qaytish →](../18-declaration-files.md)
