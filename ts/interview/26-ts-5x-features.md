# Interview: TypeScript 5.x Yangiliklari

> TC39 decorators (TS 5.0), `const` type parameters (TS 5.0), `satisfies` operator (TS 4.9), `using`/`await using` declarations (TS 5.2), `NoInfer<T>` (TS 5.4), inferred type predicates (TS 5.5), `isolatedDeclarations` (TS 5.5), `erasableSyntaxOnly` (TS 5.8) va boshqa TS 5.x feature lari bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Output savollar](#output-savollar)
- [Amaliy savollar (Coding)](#amaliy-savollar-coding)
- [Bug fix savollar](#bug-fix-savollar)

---

## Nazariy savollar

### Savol 1: `satisfies` operator (TS 4.9) nima va `as` dan farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`satisfies` — expression target type'ga mos kelishini tekshiradi, **lekin** literal/narrow inferred type'ni saqlaydi. `as` (type assertion) compile-time'da type'ni majburiy o'zgartiradi — contract tekshirilmaydi.

### To'liq tushuntirish

`as T` — programmer "bu T tipida" deydi. Compiler bidirectional assignability tekshiradi (qiymat T ga yoki T qiymatga assignable bo'lishi shart), lekin natija type — `T` (qiymatning inferred shape'i yo'qoladi). Mos kelmagan cast'ga `unknown` orqali ikkala assertion (`as unknown as T`) bypass'i mavjud — runtime crash xavfi.

`satisfies T` — compiler "expression T ga mos keladimi?" deb tekshiradi. Mos kelmasa — error. Lekin expression'ning **inferred type**'ini saqlaydi (literal, narrow, specific). Bu `as const` bilan birgalikda yaxshi ishlaydi.

Asosiy use case: object literal aniq shape'ini saqlash, lekin contract'ni tekshirish. Routes, config, theme'lar uchun keng tarqalgan pattern.

### Kod misol

```typescript
type Config = Record<string, string | number>;

// `as` — type override, narrow ma'lumot yo'qoladi
const cfg1 = { host: "localhost", port: 3000 } as Config;
cfg1.host;  // string | number (literal yo'qoldi)

// `satisfies` — contract check + narrow saqlash
const cfg2 = { host: "localhost", port: 3000 } satisfies Config;
cfg2.host;  // string (literal narrow saqlandi)
cfg2.port;  // number

// Routes — keys narrow
type RouteHandler = (req: { url: string }) => Response;
const routes = {
  "/users": (req) => new Response("users"),
  "/posts": (req) => new Response("posts"),
} satisfies Record<string, RouteHandler>;

type RoutePath = keyof typeof routes;  // "/users" | "/posts"
```

### Edge Cases

- **`as const` bilan birga** — `{ x: 1 } as const satisfies { x: number }` — `readonly { x: 1 }`
- **Function return** — `function f() { return { x: 1 } satisfies T; }` ruxsat
- **Generic constraint check** — `satisfies` da type parameter infer bo'lmaydi, lekin constraint tekshiriladi
- **Excess property** rejected — `{ a: 1, extra: 2 } satisfies { a: number }` error
- **Runtime kod yo'q** — emit'da to'liq olib tashlanadi (type erasure)

### Follow-up savollar

1. **"`satisfies` ni har joyda ishlatish kerakmi?"** — Yo'q. Faqat narrow type kerak + contract check ikkalasi kerak bo'lganda. Function parameter type allaqachon contract beradi.
2. **"`satisfies` Pick/Partial bilan qanday ishlaydi?"** — Pick/Partial result satisfies'da contract sifatida ishlatiladi, lekin output type literal saqlanadi.

</details>

---

### Savol 2: `const` type parameter (TS 5.0) qachon kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<const T>` — generic argument'ni `as const` qilingandek infer qiladi. Tuple va readonly literal type saqlanadi. Caller `as const` yozish majburiyatidan ozod bo'ladi.

### To'liq tushuntirish

Default generic inference array'ni widen qiladi: `f([1, 2, 3])` natijasi `T = number[]`. Literal va tuple saqlanmaydi.

`<const T>` modifier compiler'ga "argument'ni xuddi `as const` qilingandek tekshir" deyish. Natijada:
- Array → readonly tuple (`readonly [1, 2, 3]`)
- Object literal → readonly properties with literal values
- String literal → narrow literal type

Use case'lar: routing config, state machine transition, theme tokens, form validation schema — caller har joyda `as const` yozmaslik uchun.

### Kod misol

```typescript
// Default — widening
function routes<T extends readonly { path: string }[]>(r: T) { return r; }
const r1 = routes([{ path: "/" }, { path: "/about" }]);
// r1: readonly { path: string }[] — path widened

// `const` modifier — literal saqlanadi
function constRoutes<const T extends readonly { path: string }[]>(r: T) { return r; }
const r2 = constRoutes([{ path: "/" }, { path: "/about" }]);
// r2: readonly [{ readonly path: "/" }, { readonly path: "/about" }]

type Path = (typeof r2)[number]["path"];  // "/" | "/about"

// State machine
function defineMachine<const T extends readonly string[]>(states: T) {
  return { states };
}
const m = defineMachine(["idle", "loading", "success", "error"]);
type State = (typeof m.states)[number];  // "idle" | "loading" | "success" | "error"
```

### Edge Cases

- **Constraint bilan kombinatsiya** — `<const T extends string[]>` ishlaydi, T literal string tuple bo'ladi
- **Nested object** — har level'da readonly + literal preserve
- **Function parameter widening** — primitive (non-object) argument ham narrow saqlanadi
- **Inference fallback** — agar `const` qo'yib chiqarib bo'lmasa (variable orqali), widening qoladi
- **Runtime ta'siri yo'q** — faqat type level, emit'da olib tashlanadi

### Follow-up savollar

1. **"`as const` va `<const T>` bir vaqtda kerakmi?"** — Yo'q. `<const T>` `as const`'ni callerda majburiyatdan ozod qiladi. Ortiqcha `as const` qo'shilsa zarari yo'q, lekin keraksiz.
2. **"`<const T>` runtime'ga ta'siri bormi?"** — Yo'q, faqat type level. Emit'da olib tashlanadi.

</details>

---

### Savol 3: `using` va `await using` declarations (TS 5.2) qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`using` — TC39 Explicit Resource Management. Resource scope tugaganda **avtomatik** `[Symbol.dispose]()` chaqirilib resource ozod bo'ladi. `await using` — async dispose (`[Symbol.asyncDispose]()`). `try/finally`'ga replacement.

### To'liq tushuntirish

`using` deklaratsiya — block scope (function body, if/for block) tugaganda LIFO tartibda dispose chaqiriladi. Hatto exception throw bo'lsa ham. Manual `try/finally` boilerplate'siz.

Implementation:
- **Sync resource** — `Symbol.dispose()` method qaytaruvchi obyekt (`Disposable` interface)
- **Async resource** — `Symbol.asyncDispose()` Promise qaytaruvchi (`AsyncDisposable` interface). Faqat `await using` bilan ishlatiladi

Compile target'ga qarab TS down-level emit qiladi (`try/finally`'ga aylanadi). Runtime'da `Symbol.dispose` va `Symbol.asyncDispose` global'lari kerak (ES2023+ runtime yoki polyfill).

Resource lifetime aniq scope bilan bog'lanadi — file handle, DB connection, lock, transaction kabi cleanup kerak resurslar uchun. C# `using`, Python `with`, Java `try-with-resources` ekvivalenti.

### Kod misol

```typescript
// Sync — Disposable
class FileHandle implements Disposable {
  #path: string;
  constructor(path: string) {
    this.#path = path;
    console.log(`Opened: ${path}`);
  }
  read(): string { return "file content"; }
  [Symbol.dispose](): void {
    console.log(`Closed: ${this.#path}`);
  }
}

function processFile(path: string) {
  using handle = new FileHandle(path);
  return handle.read();
  // Scope tugaganda [Symbol.dispose]() avtomatik
  // throw bo'lsa ham
}

// Async — AsyncDisposable
class DbConnection implements AsyncDisposable {
  constructor(private url: string) {}
  async query(sql: string) { return []; }
  async [Symbol.asyncDispose](): Promise<void> {
    console.log(`Disconnecting: ${this.url}`);
  }
}

async function fetchUsers() {
  await using db = new DbConnection("postgres://localhost/app");
  return await db.query("SELECT * FROM users");
  // await db[Symbol.asyncDispose]() avtomatik
}
```

### Edge Cases

- **`null`/`undefined`** — `using x = null` ruxsat, dispose chaqirilmaydi
- **Error in dispose** — `SuppressedError` paydo bo'ladi (original error + dispose error)
- **`DisposableStack`** — runtime API, conditional dispose registration
- **`for (using x of iter)`** — har iteration'da dispose
- **Class field** — `using` faqat local variable, class field uchun emas

### Follow-up savollar

1. **"`using` va `try/finally` farqi nima?"** — `using` deklarativ, LIFO multi-resource, exception chain. `try/finally` imperative, manual order, nested boilerplate.
2. **"Symbol.dispose qaysi runtime'da mavjud?"** — Node.js 20+, Bun, Deno. Browser polyfill kerak (esnext.disposable lib).

</details>

---

### Savol 4: `NoInfer<T>` (TS 5.4) qanday muammoni hal qiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`NoInfer<T>` — generic inference'da type parameter ni o'zining position'idan **infer qilinmasligi**ni belgilaydi. Bir nechta parameter dan T infer qilinishi kerak bo'lganda, ba'zilarini "passive" qilib boshqalardan inference'ga ishonadi.

### To'liq tushuntirish

Default generic inference — har parameter T'ni infer qilishga urinadi. Bu ba'zan istalmagan: birinchi argument'dan T'ni aniqlash, qolganlari faqat T'ga mos kelishini tekshirish kerak.

`NoInfer<T>` parameter ni "passive" qiladi — T faqat boshqa parameter'lardan infer qilinadi. Type check baribir ishlaydi (mos kelmasa error), lekin inference faqat boshqa parameter'lardan.

Use case'lar: default value, fallback, override parameter (ko'pincha "tweak" parameter T'ni aniqlamasligi kerak).

### Kod misol

```typescript
// Asl muammo
function createStore<T>(initial: T, fallback: T): T { return initial ?? fallback; }
createStore(42, "hello");
// T inferred: number | string (union) — istalmagan

// NoInfer bilan
function createStoreFixed<T>(initial: T, fallback: NoInfer<T>): T {
  return initial ?? fallback;
}
createStoreFixed(42, 0);
// T = number (faqat initial'dan)
// createStoreFixed(42, "hello");  // ❌ string number'ga mos emas

// React-like API
function defineComponent<P>(props: P, defaultProps: NoInfer<P>) {
  return { props, defaultProps };
}
defineComponent({ name: "Ali", age: 25 }, { name: "", age: 0 });

// Configurable factory
function createValidator<T>(
  schema: T,
  errors: { [K in keyof T as NoInfer<K>]?: string }
) {
  return { schema, errors };
}
```

### Edge Cases

- **Faqat inference'ni bloklaydi, type check ishlaydi** — agar `NoInfer<T>` parameter'i type mos kelmasa error
- **Nested generic** — `NoInfer<Array<T>>` inner T'ni ham bloklaydi
- **Constraint bilan** — `<T extends U>` T inference NoInfer bilan birga
- **TS 5.4'dan oldin** — `[T][T extends any ? 0 : never]` hack mavjud edi
- **Default type parameter** — `<T = string>` bilan birga ishlaydi

### Follow-up savollar

1. **"`NoInfer<T>` runtime ta'siri bormi?"** — Yo'q, type-level marker. Emit'da yo'qoladi.
2. **"Nima uchun NoInfer ba'zan kutilgan inference'ga teskari natija beradi?"** — `T` faqat oddiy parameter'dan infer qilinishi kerak; agar oddiy parameter'da union bo'lsa, T union'ga widen bo'ladi.

</details>

---

### Savol 5: Inferred type predicates (TS 5.5) nima va qaerda ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TS 5.5'dan `Array.filter` (va shunga o'xshash method'lar) callback `boolean` qaytarganda, compiler avtomatik **type predicate** infer qiladi. Manual `(v): v is T` yozish kerak emas — narrowing avtomatik.

### To'liq tushuntirish

TS 5.4'gacha `[1, null].filter(x => x !== null)` natijasi `(number | null)[]` qoladi. Manual predicate kerak edi: `.filter((x): x is number => x !== null)`.

TS 5.5'da compiler callback ichidagi logic'ni tahlil qilib type narrowing predicate ni avtomatik chiqaradi. Quyidagi shartlar bajarilsa:
- Callback type predicate'ga "rasshifrovka" qilinadi (`typeof x === "string"`, `x !== null`, `x instanceof Cls`, va h.k.)
- Callback boshqa kompleks logic ishlatmaydi
- Narrowing aniq

Boshqa method'lar ham ishlaydi: `Array.find`, `Array.some`, `Array.every`. Lekin asosiy use case `filter`.

### Kod misol

```typescript
// TS 5.4 va undan oldin
const values1: (string | null | undefined)[] = ["a", null, "b", undefined];
const filtered1 = values1.filter((v): v is string => v != null);
// filtered1: string[] (manual predicate)

// TS 5.5+
const values2: (string | null | undefined)[] = ["a", null, "b", undefined];
const filtered2 = values2.filter(v => v != null);
// filtered2: string[] — avtomatik infer

// typeof bilan
const mixed: (string | number)[] = ["a", 1, "b", 2];
const strings = mixed.filter(v => typeof v === "string");
// strings: string[]

// instanceof bilan
class Cat { meow() {} }
class Dog { bark() {} }
const animals: (Cat | Dog)[] = [new Cat(), new Dog()];
const cats = animals.filter(a => a instanceof Cat);
// cats: Cat[]

// `.find` da ham
const firstString = mixed.find(v => typeof v === "string");
// firstString: string | undefined
```

### Edge Cases

- **Murakkab callback** — agar callback if/else, multiple return bilan complex bo'lsa, infer ishlamasligi mumkin
- **Negated check** — `x => !someCheck(x)` ba'zan negation predicate infer
- **External function call** — `arr.filter(isUser)` — `isUser` o'zi predicate bo'lishi kerak
- **Array of objects** — property check bilan ham ishlaydi (`x => x.id != null`)
- **`!= null` vs `!== null`** — `!= null` ikkala `null` va `undefined`'ni olib tashlaydi (loose equality)

### Follow-up savollar

1. **"Murakkab callback'da explicit predicate yana kerak?"** — Ha. Compiler infer qila olmasa, `(v): v is T` explicit yozish kerak.
2. **"`some` va `every`'da narrowing nima ma'noda?"** — `some` — element T tipi bormi yo'qmi (boolean), `every` — barcha element T tipida (array narrowing).

</details>

---

### Savol 6: TC39 Stage 3 decorators (TS 5.0) legacy decoratordan qanday farq qiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TC39 Stage 3 decorators — yangi ECMAScript standart (TS 5.0). `experimentalDecorators` flag kerak emas. Yangi API: decorator funksiyalari `(value, context)` qabul qiladi, har element turi uchun aniq context. Legacy decoratorlar — eski TS implementation, `experimentalDecorators: true` talab qiladi va Stage 1 proposal'ga asoslangan.

### To'liq tushuntirish

**Legacy decorator** (TS <5.0):
- `experimentalDecorators: true` kerak
- Signature: `(target, propertyKey, descriptor)` — `Reflect.metadata` ishlatiladi
- Parameter decorator mavjud (Stage 3'da yo'q)
- ECMA spec'ga rasman kirmagan — TypeScript-only

**Stage 3 decorator** (TS 5.0+):
- Flag kerak emas (default qoidlangan)
- Signature: `(value, context)` — `context` aniq turi (`ClassMethodDecoratorContext`, `ClassFieldDecoratorContext`)
- `Symbol.metadata` orqali metadata (TS 5.2+)
- ECMAScript standart bo'lib bormoqda — kelajakda runtime native

Asosiy farq'lar:
1. **Parameter decorator yo'q** — TC39'da rad etildi (Stage 3'da)
2. **Field decorator** — accessor'ga aylanadi, initializer return qiladi
3. **Method decorator** — function return, replace qila oladi
4. **Class decorator** — class return, replace qila oladi

Migration: legacy → Stage 3 — ko'pincha API qayta yozish kerak. Frameworks (Angular, NestJS) hozircha legacy'da qoldi.

### Kod misol

```typescript
// Stage 3 — method decorator
function log<This, Args extends unknown[], Return>(
  value: (this: This, ...args: Args) => Return,
  context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => Return>
) {
  const methodName = String(context.name);
  return function (this: This, ...args: Args): Return {
    console.log(`Calling ${methodName}(${JSON.stringify(args)})`);
    const result = value.apply(this, args);
    console.log(`Result: ${JSON.stringify(result)}`);
    return result;
  };
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

new Calculator().add(2, 3);
// → Calling add([2,3])
// → Result: 5
```

### Edge Cases

- **Initializer hook** — `context.addInitializer(fn)` — class yaratilganda chaqiriladi
- **Private name access** — `context.access.get/set` — private maydonga decorator'dan kirish
- **Metadata API** — `context.metadata` — `Symbol.metadata` orqali shared metadata
- **Static vs instance** — `context.static` boolean
- **Accessor decorator** — `get`/`set` ikkalasi uchun alohida transformatsiya

<details>
<summary><strong>Deep Dive</strong></summary>

Stage 3 decorators ECMAScript spec'ning Stage 3 (Candidate) bosqichida. Bu bosqichda API to'liq belgilangan, lekin oz o'zgarishlar bo'lishi mumkin (Stage 4'gacha). TS 5.0 Stage 3'ga muvofiq implement qildi.

Legacy va Stage 3 bir vaqtda ishlatish mumkin emas — `experimentalDecorators: true` Stage 3 syntax'ni rad etadi. `experimentalDecorators: false` (default) Stage 3 ishlaydi.

Framework migration holati:
- **NestJS**: hozircha legacy, Stage 3 plan'da
- **Angular**: legacy + reflect-metadata, Stage 3 migration aktiv
- **TypeORM, class-validator**: legacy + reflect-metadata

`reflect-metadata` polyfill — legacy decoratorlar uchun runtime metadata storage. Stage 3 da `Symbol.metadata` native — polyfill kerak emas.

`emitDecoratorMetadata` — faqat legacy bilan. Type info'ni runtime'da reflect-metadata orqali saqlash (`design:type`, `design:paramtypes`, `design:returntype`). Stage 3 da bunday emit yo'q — manual annotation kerak.

</details>

</details>

---

### Savol 7: `isolatedDeclarations` (TS 5.5) nima uchun kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`isolatedDeclarations` — har file'ning `.d.ts` declaration emit'i **faqat shu file** type info'sidan hosil bo'lishini majburlaydi. Inferred type'lar ruxsat etilmaydi — har export'da explicit type annotation kerak. Bu parallel build'ni tezlashtiradi.

### To'liq tushuntirish

TS default `.d.ts` emit'da function/class return type'larini infer qiladi — bu boshqa file'lardagi type'larga bog'liqlikni keltirib chiqaradi (import chain). Katta loyihada bir file o'zgarsa, bog'liq file'lar declaration emit qayta hisoblanadi.

`isolatedDeclarations: true` — har file mustaqil tahlil qilinishini majburlaydi:
- Public export'lar uchun explicit return type
- Class/method explicit return type
- Re-export explicit type

Foyda: build tool (Bun, esbuild, swc) `.d.ts` emit'ni TypeScript compiler'siz ham bajara oladi — har file alohida parse qilinib type yoziladi.

Trade-off: developer'ga explicit annotation majburi, lekin build performance va parallelizm sezilarli yaxshilanadi (monorepo, large codebase).

### Kod misol

```typescript
// isolatedDeclarations: true talab qiladi
export function getUser(id: number): { id: number; name: string } {
  return { id, name: "Ali" };
}

// ❌ — return type infer
// export function badUser(id: number) {
//   return { id, name: "Ali" };
// }

export class UserService {
  // ✅ explicit return
  getUser(id: number): User { return { id, name: "Ali" }; }
}

// Type alias OK
export type User = { id: number; name: string };
```

### Edge Cases

- **Class field** — `private name = "x"` — initializer'dan infer ruxsat (private — declaration'ga chiqmaydi)
- **Const variable** — `const x = 1` literal type infer (declaration'da `const x: 1`)
- **Conditional type return** — explicit annotation kerak
- **Generic instantiation** — `function f<T>(x: T): T` — T explicit
- **Build tool integration** — parallel `.d.ts` emit qila oladi

### Follow-up savollar

1. **"`isolatedDeclarations` kichik loyihada kerakmi?"** — Yo'q. Foyda monorepo va parallel build tool ishlatadigan loyihalarda sezilarli.
2. **"`isolatedModules` bilan farqi?"** — `isolatedModules` — har file mustaqil transpilation (TS → JS). `isolatedDeclarations` — har file mustaqil `.d.ts` emit. Ikkalasi orthogonal.

</details>

---

### Savol 8: `erasableSyntaxOnly` (TS 5.8) nima va qaysi syntax taqiqlanadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`erasableSyntaxOnly` — faqat **type erasure** bilan o'chiriladigan syntax ruxsat. `enum`, `namespace`, parameter property (constructor `public/private x`), legacy decorators — TAQIQ. Node.js'ning `--experimental-strip-types` flag'iga mo'ljallangan.

### To'liq tushuntirish

Node.js 22+ native `.ts` file'larni `--experimental-strip-types` flag bilan ishlatadi. Bu strip-only — TS syntax JS syntax'ga aylanmaydi, faqat type annotation'lar olib tashlanadi. Shuning uchun runtime kod hosil qiluvchi TS feature'lar mos kelmaydi.

Taqiqlanadigan syntax (runtime kod hosil qiladi):
- **`enum`** — JS'da yo'q, TS object hosil qiladi
- **`namespace`** (non-type) — IIFE hosil qiladi
- **Parameter property** — `constructor(public name: string)` — body'ga `this.name = name` qo'shadi
- **Legacy decorator emit'i** — runtime decorator behavior

Ruxsat (type erasure):
- Type annotation, interface, type alias
- Generic parameter
- `satisfies`, `as`, type assertion
- `import type`, `export type`
- TC39 Stage 3 decorators (JS standart)

Migration: enum → `as const` object, namespace → object yoki ES module, parameter property → manual assign.

### Kod misol

```typescript
// ❌ erasableSyntaxOnly da TAQIQ
// enum Status { Active, Inactive }

// ✅ muqobil
const Status = { Active: 0, Inactive: 1 } as const;
type Status = (typeof Status)[keyof typeof Status];

// ❌ namespace TAQIQ
// namespace StringUtils {
//   export function capitalize(s: string) { return s.toUpperCase(); }
// }

// ✅ ES module
export const StringUtils = {
  capitalize: (s: string) => s.toUpperCase(),
};

// ❌ Parameter property
// class Logger {
//   constructor(public name: string, private level: number) {}
// }

// ✅ Explicit assign
class Logger {
  public name: string;
  private level: number;
  constructor(name: string, level: number) {
    this.name = name;
    this.level = level;
  }
}

// ✅ Pure type — ruxsat
type Config = { url: string; port: number };
interface User { id: number; name: string; }
```

### Edge Cases

- **`const enum`** — taqiq, lekin inlined bo'lgani uchun ko'pincha re-implement oson
- **Declaration merging namespace** — type-only namespace ruxsat (faqat type member'lar)
- **Re-export'da default qiymat** — class re-export OK (type info erased)
- **TC39 decorators** — Stage 3 decoratorlar runtime'da JS standart bo'lgani uchun ruxsat (kelajakda)
- **Node 22 strip-types** — `erasableSyntaxOnly` bilan birga ishlatish tavsiya

### Follow-up savollar

1. **"`enum` ni butunlay tashlash kerakmi?"** — `erasableSyntaxOnly` ishlatadigan loyihada — ha. Aks holda — opsiyaviy.
2. **"`reflect-metadata` `erasableSyntaxOnly`'ga ta'sir qiladimi?"** — Ha. Legacy decorator + emitDecoratorMetadata'ga tayanadi — taqiqlanadi.

</details>

---

## Output savollar

### Savol 9: `const` type parameter natijasi nima? [Middle]

```typescript
function defineColors<const T extends readonly string[]>(colors: T) {
  return colors;
}

const palette = defineColors(["red", "green", "blue"]);
type PaletteItem = (typeof palette)[number];

function defineColorsNoConst<T extends readonly string[]>(colors: T) {
  return colors;
}

const palette2 = defineColorsNoConst(["red", "green", "blue"]);
type PaletteItem2 = (typeof palette2)[number];
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`palette`: `readonly ["red", "green", "blue"]`, `PaletteItem`: `"red" | "green" | "blue"`. `palette2`: `string[]`, `PaletteItem2`: `string`.

### To'liq tushuntirish

`<const T>` modifier argument'ni `as const` qilingandek infer qiladi — array tuple readonly bo'lib, har element literal saqlanadi. `(typeof palette)[number]` index access bilan union narrow olinadi.

`defineColorsNoConst` da `T extends readonly string[]` constraint bor, lekin `const` modifier yo'q — array `string[]` ga widen bo'ladi (literal yo'qoladi). `PaletteItem2` `string` umumiy type bo'ladi.

### Edge Cases

`as const` callerda yozilsa, `<const T>` bilan bir xil natija beradi: `defineColorsNoConst(["red", "green", "blue"] as const)` — `readonly ["red", "green", "blue"]`. `<const T>` shu boilerplate'ni olib tashlaydi.

</details>

---

### Savol 10: `satisfies` va `as` output farqi [Middle]

```typescript
type ColorConfig = Record<string, { hex: string; rgb: readonly number[] }>;

const c1 = {
  primary: { hex: "#ff0000", rgb: [255, 0, 0] as const },
  secondary: { hex: "#00ff00", rgb: [0, 255, 0] as const },
} as ColorConfig;

const c2 = {
  primary: { hex: "#ff0000", rgb: [255, 0, 0] as const },
  secondary: { hex: "#00ff00", rgb: [0, 255, 0] as const },
} satisfies ColorConfig;

const x1 = c1.primary;
const x2 = c2.primary;
const y1 = c1.nonExistent;
const y2 = c2.nonExistent;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`x1`: `{ hex: string; rgb: readonly number[] }` (widened). `x2`: narrow inferred type (`readonly [255, 0, 0]` saqlanadi). `y1`: type OK (Record'da har key ruxsat) lekin runtime'da `undefined`. `y2`: ❌ compile error — `nonExistent` key'i yo'q.

### To'liq tushuntirish

`as ColorConfig` natijasi `Record<string, ...>` — har string key ruxsat, narrow key'lar yo'qoladi.

`satisfies ColorConfig` natijasi inferred type — `{ primary: {...}; secondary: {...} }`. Faqat shu ikki key bor. `nonExistent` access'da compile error. Tuple `readonly [255, 0, 0]` ham saqlanadi.

</details>

---

### Savol 11: `using` LIFO tartibi — output [Middle]

```typescript
class Resource implements Disposable {
  constructor(private id: string) { console.log(`open ${id}`); }
  [Symbol.dispose]() { console.log(`close ${this.id}`); }
}

function run() {
  using a = new Resource("A");
  using b = new Resource("B");
  using c = new Resource("C");
  console.log("work");
}

run();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
open A
open B
open C
work
close C
close B
close A
```

### To'liq tushuntirish

Resource'lar deklaratsiya tartibida open bo'ladi. Scope tugaganda **LIFO** (Last-In-First-Out) tartibda dispose — `C → B → A`. Bu nested resource'lar uchun to'g'ri: oxirgi olingan resurs avval ozod bo'ladi. C# `using` va Java `try-with-resources` ham xuddi shu tartibni qo'llaydi.

### Edge Cases

Dispose chaqirig'ida exception throw bo'lsa, qolgan resource'lar baribir dispose qilinadi. Birinchi exception saqlanib, keyingi exception bilan `SuppressedError` chain hosil bo'ladi.

</details>

---

### Savol 12: Inferred type predicates natijasi [Middle]

```typescript
interface Product { id: number; name: string; price: number; }

const products: (Product | null | undefined)[] = [
  { id: 1, name: "Phone", price: 500 },
  null,
  { id: 2, name: "Laptop", price: 1500 },
  undefined,
];

// TS 5.5+
const valid = products.filter(p => p != null);
const expensive = valid.filter(p => p.price > 1000);

console.log(valid.length);
console.log(expensive[0].name);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`valid`: `Product[]`, length `2`. `expensive`: `Product[]`, `expensive[0].name`: `"Laptop"`.

### To'liq tushuntirish

TS 5.5+ `filter` callback'dan type predicate avtomatik infer qiladi:
- `p != null` — `p is Product` (null va undefined olib tashlanadi)
- `p.price > 1000` — narrowing yo'q (boolean predicate, type allaqachon Product)

Output: `valid.length` → `2`, `expensive[0].name` → `"Laptop"`.

</details>

---

## Amaliy savollar (Coding)

### Savol 13: `using` bilan database pool implement [Senior]

**Savol:** `DatabasePool` class yozing — `await using` orqali pool destroy. `acquire()` `using` orqali connection auto-release.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`PooledConnection` `Disposable` (sync release), `DatabasePool` `AsyncDisposable` (async cleanup). Scope tugaganda avtomatik release/destroy.

### Kod misol

```typescript
class PooledConnection implements Disposable {
  constructor(
    public readonly id: number,
    private onRelease: (id: number) => void
  ) {
    console.log(`Connection ${id} acquired`);
  }

  query(sql: string): string {
    return `[Conn ${this.id}] ${sql}`;
  }

  [Symbol.dispose](): void {
    console.log(`Connection ${this.id} released`);
    this.onRelease(this.id);
  }
}

class DatabasePool implements AsyncDisposable {
  private available: number[] = [];

  private constructor(private size: number) {
    for (let i = 1; i <= size; i++) this.available.push(i);
  }

  static async create(size: number): Promise<DatabasePool> {
    console.log(`Pool created (size=${size})`);
    return new DatabasePool(size);
  }

  acquire(): PooledConnection {
    const id = this.available.pop();
    if (id === undefined) throw new Error("Pool exhausted");
    return new PooledConnection(id, (released) => {
      this.available.push(released);
    });
  }

  async [Symbol.asyncDispose](): Promise<void> {
    console.log(`Pool destroyed (${this.available.length} connections)`);
  }
}

async function main() {
  await using pool = await DatabasePool.create(3);

  {
    using conn = pool.acquire();
    console.log(conn.query("SELECT * FROM users"));
  }  // conn dispose

  using conn2 = pool.acquire();
  console.log(conn2.query("SELECT * FROM products"));
}  // conn2 dispose, then pool dispose

await main();
// → Pool created (size=3)
// → Connection 3 acquired
// → [Conn 3] SELECT * FROM users
// → Connection 3 released
// → Connection 3 acquired
// → [Conn 3] SELECT * FROM products
// → Connection 3 released
// → Pool destroyed (3 connections)
```

### Edge Cases

- `available.pop()` LIFO — oxirgi released avval qayta acquire
- `acquire()` exhausted bo'lsa exception — ko'pincha queueing yoki async wait pattern qo'shiladi
- Pool destroy paytida active connection bo'lsa — `DisposableStack` orqali tracking yaxshi
- Async dispose error bo'lsa — `SuppressedError` chain bilan

<details>
<summary><strong>Deep Dive</strong></summary>

**Down-level emit semantics.** TS `using` deklaratsiyani target'ga qarab transform qiladi. ES2022 va undan past target'larda compiler `try/finally` bilan implementation kod hosil qiladi — har `using` declaration yashirin `try` block ochadi, scope tugaganda `finally`'da `[Symbol.dispose]()` chaqiriladi. Ko'p `using` deklaratsiyalar nested `try/finally` ga aylanadi (LIFO kafolatlash uchun).

**`SuppressedError` semantikasi.** ECMAScript spec qoidasi: agar function body throw qilsa va dispose chaqirig'i ham throw qilsa, dispose error original error'ni "suppress" qiladi. `SuppressedError` class'i ikki error'ni saqlaydi: `error` (suppressing) va `suppressed` (original). Stack trace ikkalasini ham ko'rsatadi.

**`DisposableStack` runtime API.** Conditional resource registration — `stack.use(resource)` resurs scope'ga qo'shadi, `stack.adopt(value, onDispose)` arbitrary value + callback. `stack.defer(callback)` arbitrary cleanup. `stack.move()` ownership transfer — partial init'da useful (constructor halfway throw bo'lsa qisman initialized resource'lar dispose).

**Async dispose ordering kafolat.** `await using` bo'lsa, dispose chain to'liq awaited. Ammo sync `using` ichida async resource bo'lsa (`await using` o'rniga `using`) — TS compile-time'da bunday mismatch'ni rad etadi (`AsyncDisposable` faqat `await using` bilan).

**Pool exhaustion strategy.** Production pool'larda `acquire()` Promise qaytaradi — bo'sh slot bo'lmasa wait queue'ga qo'shiladi (event loop). `Symbol.asyncDispose` bilan birga `await using conn = await pool.acquire()` pattern — connection kelguncha kutish va auto-release.

**Spec stage va runtime support.** TC39 Explicit Resource Management — Stage 3 (TS 5.2 release vaqtida). Node.js 22+, Bun 1.0+, Deno 1.40+ native qo'llaydi. Browser support hozircha cheklangan — polyfill (`esnext.disposable` lib option, `core-js`) kerak.

</details>

</details>

---

### Savol 14: Type-safe routing — `const` + `satisfies` [Senior]

**Savol:** Routing config yozing. Path'lar va parameter'lar type-safe — runtime'da string, compile-time'da narrow.

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type RouteParam<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | RouteParam<`/${Rest}`>
    : Path extends `${string}:${infer Param}`
    ? Param
    : never;

type RouteDefinition = {
  path: string;
  handler: (params: Record<string, string>) => Response;
};

function defineRoutes<const T extends readonly RouteDefinition[]>(routes: T) {
  return {
    routes,
    navigate<P extends T[number]["path"]>(
      path: P,
      params: Record<RouteParam<P>, string>
    ): string {
      let url = path as string;
      for (const [key, value] of Object.entries(params)) {
        url = url.replace(`:${key}`, value);
      }
      return url;
    },
  };
}

const router = defineRoutes([
  { path: "/users/:userId", handler: () => new Response("user") },
  { path: "/posts/:postId/comments/:commentId", handler: () => new Response("comment") },
]);

// Type-safe navigate
router.navigate("/users/:userId", { userId: "42" });
// → "/users/42"

router.navigate("/posts/:postId/comments/:commentId", {
  postId: "1",
  commentId: "10",
});
// → "/posts/1/comments/10"

// Compile error: noto'g'ri path
// router.navigate("/invalid", {});  // ❌

// Compile error: missing parameter
// router.navigate("/users/:userId", {});  // ❌ userId required
```

### To'liq tushuntirish

`<const T>` paths'ni literal saqlaydi. `RouteParam<P>` template literal type bilan parameter nomlarini ajratib oladi. `Record<RouteParam<P>, string>` — har parameter uchun majburi key. Compiler typo va missing parameter'larni rad etadi.

### Edge Cases

- Optional parameter (`:param?`) qo'shilsa — `RouteParam` yangi conditional branch kerak
- Query string — alohida type bilan
- Nested groups — template literal recursion bilan
- Path collision — runtime detection (compile-time qiyin)

<details>
<summary><strong>Deep Dive</strong></summary>

**Template literal type recursion mexanizmi.** `RouteParam<P>` conditional type recursive — `Path extends ${string}:${infer Param}/${infer Rest}` pattern segment'ni ajratib oladi (`:userId/comments/:commentId` → `Param = "userId"`, `Rest = "comments/:commentId"`). Recursion `RouteParam<\`/${Rest}\`>` bilan davom etadi. Base case — `:${infer Param}` (oxirgi parameter). Tail recursion paterni — TS optimizer recursion'ni unroll qiladi (TS 4.5+ tail call optimization for conditional types).

**`<const T>` inference deep effect.** Routes array `[{path: "/users/:userId"}, ...]` — default inference `{path: string}[]` widening qiladi. `<const T>` har element'ni `as const` qilingandek: `readonly [{readonly path: "/users/:userId"}, ...]`. `T[number]["path"]` indexed access narrow union beradi: `"/users/:userId" | "/posts/:postId/comments/:commentId"`.

**`Record<RouteParam<P>, string>` exhaustiveness.** Recursive `RouteParam<P>` union qaytaradi (`"postId" | "commentId"`). `Record<Union, string>` har key majburi (TS missing key error). Bu compile-time exhaustiveness — runtime'da `replace` qilingan key parametrlar to'liq berilgan.

**Recursion depth limit.** TS conditional type recursion default 50 level (`tsc --noErrorTruncation` da error: "Type instantiation is excessively deep"). Long path (`/a/:b/c/:d/...`) recursion limit'ga yaqinlashishi mumkin. Workaround: accumulator pattern (`RouteParam<Path, Acc = never>`) tail-recursive form.

**Wildcards va optional params.** `:param?` (optional) — conditional branch'da `Path extends ${string}:${infer Param}?` qo'shish (`?` literal). Wildcard (`/*`) — `${string}*` pattern. Production router'lar (`react-router`, `expo-router`) bu pattern'larni keng ishlatadi — schema'dan generated tipga tayanadi.

**Compile-time vs runtime sync.** Type-level routing fully compile-time — runtime kod oddiy string replace. `defineRoutes` runtime'da array storage, `navigate` runtime'da `replace` chaqiradi. Type system bu runtime invariant'ga (parameter key route path'ga mos) compile-time'da kafolat beradi.

</details>

</details>

---

### Savol 15: TS 5.x feature versiya quiz [Middle]

**Savol:** Har feature qaysi TS versiyada qo'shilgan?

```
A. satisfies operator
B. TC39 Stage 3 decorators
C. using / await using declarations
D. NoInfer<T>
E. Inferred type predicates
F. isolatedDeclarations
G. erasableSyntaxOnly
H. const type parameters
I. Symbol.metadata
J. moduleResolution: "bundler"
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
A. satisfies operator        → TS 4.9
B. TC39 Stage 3 decorators   → TS 5.0
C. using / await using       → TS 5.2
D. NoInfer<T>                → TS 5.4
E. Inferred type predicates  → TS 5.5
F. isolatedDeclarations      → TS 5.5
G. erasableSyntaxOnly        → TS 5.8
H. const type parameters     → TS 5.0
I. Symbol.metadata           → TS 5.2
J. moduleResolution bundler  → TS 5.0
```

</details>

---

## Bug fix savollar

### Savol 16: `using` ishlatishda xato — toping va tuzating [Middle+]

**Savol:** Quyidagi kod compile bo'lmaydi. Sabab nima?

```typescript
class Connection {
  constructor(private url: string) {}
  close() { console.log(`Closed: ${this.url}`); }
}

function processData() {
  using conn = new Connection("postgres://localhost");
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Connection` `Disposable` interface implement qilmaydi — `[Symbol.dispose]()` method'i yo'q. `using` faqat `Symbol.dispose` (yoki `Symbol.asyncDispose`)ni taniydi, `close()` method emas.

### To'liq tushuntirish

`using` deklaratsiya runtime'da resource scope tugaganda `[Symbol.dispose]()` chaqiradi. Compiler bu method mavjudligini tekshiradi. `close()` — odatiy method nomi, lekin `using` uchun maxsus emas.

### Kod misol (tuzatilgan)

```typescript
class Connection implements Disposable {
  constructor(private url: string) {}

  [Symbol.dispose](): void {
    console.log(`Closed: ${this.url}`);
  }
}

function processData() {
  using conn = new Connection("postgres://localhost");
  // Scope tugaganda [Symbol.dispose]() avtomatik
}

// Yoki — close() ham saqlash:
class Connection2 implements Disposable {
  constructor(private url: string) {}
  close(): void { console.log(`Closed: ${this.url}`); }
  [Symbol.dispose](): void { this.close(); }
}
```

### Edge Cases

- `AsyncDisposable` interface — `[Symbol.asyncDispose]()` Promise qaytaradi, `await using` bilan
- `Symbol.dispose` global'i mavjud bo'lishi kerak (Node 20+, lib `esnext.disposable`)
- Class library — Stage 3 standart, fallback polyfill kerak

</details>

---

### Savol 17: `NoInfer` notog'ri ishlatish [Senior]

**Savol:** Quyidagi function `setState(0, p => p + 1)` chaqirig'ida nima xato? Tuzating.

```typescript
function setState<T>(initial: NoInfer<T>, update: (prev: T) => T): T {
  return update(initial);
}

const result = setState(0, (prev) => prev + 1);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`NoInfer<T>` `initial` parameter'ida — T `initial`'dan infer bo'lmaydi. `update` parameter'idan T infer qilinishi kerak, lekin `(prev) => prev + 1`'dan `prev` annotation'i yo'q — T `unknown`'ga qoladi. Yechim: `NoInfer`'ni `update`'ga ko'chirish.

### To'liq tushuntirish

`NoInfer<T>` "passive" parameter belgilaydi. T faqat **boshqa** parameter'lardan infer bo'ladi. Bu yerda `update: (prev: T) => T` — agar `prev` annotation'i berilmasa, compiler T'ni infer qila olmaydi (callback parameter'dan teskari inference odatda ishlamaydi).

Tuzatish: `initial`'dan T'ni infer qilish, `update`'ni passive qilish:

### Kod misol

```typescript
// ✅ Tuzatilgan
function setState<T>(initial: T, update: (prev: NoInfer<T>) => NoInfer<T>): T {
  return update(initial);
}

const result1 = setState(0, (prev) => prev + 1);
// T = number (initial'dan), update callback'i T'ga mos
// result1: number

// ❌ String berib ko'rish
// setState(0, (prev: string) => "x");  // ❌ string number'ga mos emas

// Boshqa pattern — callback'dan infer
function setStateAlt<T>(initial: NoInfer<T>, defaultValue: T): T {
  return initial ?? defaultValue;
}
const v = setStateAlt(0, 42);  // T = number (defaultValue'dan)
```

### Edge Cases

- **Callback parameter inference** — TS reverse inference cheklangan, explicit annotation kerak
- **`NoInfer` har joyda ishlatmaslik** — faqat aniq use case'da (default value, fallback)
- **Type widening** — primitive literal'lar T = literal vs general type — context'ga qarab

<details>
<summary><strong>Deep Dive</strong></summary>

**`NoInfer<T>` implementation.** TS standard library'da:

```typescript
type NoInfer<T> = [T][T extends any ? 0 : never];
```

Bu hack (TS 5.4'dan oldin manual ishlatilgan) — indexed access tuple'dan T'ni "extract" qiladi, lekin TS inference engine bu position'da T'ni candidate sifatida qabul qilmaydi. TS 5.4'da `NoInfer` rasman lib type — internal flag `INTRINSIC` orqali inference'dan chetlatadi (`intrinsicTypeKinds.NoInfer`).

**Inference algorithm interaction.** TS generic inference candidate collection bosqichida har parameter'dan T uchun nomzod chiqaradi (`InferTypeArguments`). `NoInfer<T>` position'i bu collection'dan istisno. Lekin keyingi assignability check'da `NoInfer<T>` parameter T'ga mos kelishi tekshiriladi — type check ishlaydi, inference o'tkazib yuboriladi.

**Callback parameter reverse inference chegarasi.** `update: (prev: T) => T`'da `prev` annotation'siz callback yozilsa (`(prev) => prev + 1`), TS contextual typing orqali `prev`'ni `T` deb qabul qiladi — lekin T birinchi navbatda boshqa parameter'lardan aniqlanishi kerak. Agar boshqa parameter `NoInfer<T>` bo'lsa, T uchun candidate yo'q → T `unknown` yoki constraint default'ga tushadi.

**`prev + 1` semantikasi.** `T = unknown` bo'lsa `prev + 1` — `unknown + number` error. TS arithmetic operator'lar `number | bigint`'ga toraytirilgan operand kutadi. Bu xato chain'da paydo bo'ladi.

**Reverse inference: declaration position vs use position.** `<T>` parameter generic function call'da har use site uchun mustaqil aniqlanadi. `setState(0, (p) => p + 1)`'da TS birinchi parameter'dan `T = number` nomzod chiqarmoqchi bo'ladi, ammo `NoInfer<T>` bu nomzodni rad etadi. Ikkinchi parameter `(p: T) => T` — `p` annotation'siz, T uchun candidate berolmaydi.

**Tuzatish yondashuvi.** Variant 1: `NoInfer`'ni ko'chirish (yuqorida). Variant 2: callback parameter'iga explicit annotation (`(prev: number) => prev + 1`). Variant 3: `NoInfer` umuman olib tashlash — agar T'ni har ikki parameter'dan infer qilish maqbul (union widening bo'lmasa).

**Spec status.** `NoInfer<T>` TS 5.4 release (2024-03-06)'da kiritildi. ECMAScript standartiga mansub emas — TS-specific utility (Java/C#/Rust ekvivalenti yo'q, ammo Scala `=:=` constraint'lar shu maqsadda).

</details>

</details>

---

## Xulosa

- **`satisfies` (TS 4.9)** — qiymat target type'ga moslashuvini tekshiradi, lekin inferred type (literal/narrow) saqlaydi. `as`'dan farqli — type'ni override qilmaydi
- **`const` type parameters (TS 5.0)** — generic argumentni `as const` qilingandek infer (tuple, readonly, literal). Caller'da `as const` boilerplate'siz
- **TC39 Stage 3 decorators (TS 5.0)** — yangi standart, `(value, context)` signature, `experimentalDecorators` flag kerak emas. Parameter decorator yo'q
- **`using`/`await using` (TS 5.2)** — Explicit Resource Management. `[Symbol.dispose]()` LIFO avtomatik, `try/finally`'ga replacement
- **`NoInfer<T>` (TS 5.4)** — generic parameter'ni inference'dan chetlatadi, type check saqlanadi. Default/fallback parameter pattern
- **Inferred type predicates (TS 5.5)** — `Array.filter` callback'dan predicate avtomatik. Manual `(v): v is T` kerak emas (sodda case'larda)
- **`isolatedDeclarations` (TS 5.5)** — har file mustaqil `.d.ts` emit, public export'da explicit return type majburiy. Build tool parallelizmi
- **`erasableSyntaxOnly` (TS 5.8)** — type erasure bilan o'chiriladigan syntax cheklash. `enum`, `namespace`, parameter property TAQIQ. Node.js `--experimental-strip-types` integrasi
- **Migration roadmap** — legacy decorator → Stage 3 (framework bog'liq), `enum` → `as const` object, `namespace` → ES module, parameter property → explicit assign

