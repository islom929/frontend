# Bo'lim 1: TypeScript Nima va Nima Uchun

> TypeScript — Microsoft tomonidan yaratilgan, JavaScript ustiga qurilgan statik tiplangan dasturlash tili. TypeScript compiler (tsc) source code ni tahlil qilib, type-check qilib, oddiy JavaScript ga compile qiladi. Runtime da TypeScript yo'q — faqat JavaScript qoladi.

---

## Mundarija

- [TypeScript Nima](#typescript-nima)
- [TypeScript vs JavaScript](#typescript-vs-javascript)
- [TypeScript Compiler — tsc Pipeline](#typescript-compiler--tsc-pipeline)
- [Type Erasure — Runtime da Types Yo'q](#type-erasure--runtime-da-types-yoq)
- [Structural Typing vs Nominal Typing](#structural-typing-vs-nominal-typing)
- [Fayl Kengaytmalari — .ts, .tsx, .d.ts, .mts, .cts](#fayl-kengaytmalari--ts-tsx-dts-mts-cts)
- [tsconfig.json Asoslari](#tsconfigjson-asoslari)
- [strict: true Nima Qiladi](#strict-true-nima-qiladi)
- [TypeScript Playground](#typescript-playground)
- [TypeScript Versiyalar Tarixi](#typescript-versiyalar-tarixi)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## TypeScript Nima

### Nazariya

TypeScript — JavaScript ning **typed superset** i. "Superset" degani har qanday to'g'ri JavaScript kodi o'z-o'zidan TypeScript kodi hisoblanadi. TypeScript JavaScript ustiga **static type system** qo'shadi — o'zgaruvchilar, funksiya parametrlari, return qiymatlari va ifodalar uchun tiplar (types) belgilash imkonini beradi. Bu tiplar **compile-time** da tekshiriladi — ya'ni kod ishga tushmasdan oldin xatolar aniqlanadi.

TypeScript ni 2012-yil oktyabrda **Anders Hejlsberg** (C# va Delphi tillarining yaratuvchisi) rahbarligida Microsoft e'lon qilgan. TypeScript open-source loyiha bo'lib, Apache 2.0 litsenziyasi ostida GitHub da rivojlantiriladi. Bugungi kunda deyarli barcha yirik JavaScript loyihalar — Angular, React ecosystem (Next.js, Remix), Vue 3, NestJS, Deno, VS Code — TypeScript bilan yoziladi yoki TypeScript ni to'liq qo'llab-quvvatlaydi.

TypeScript ning asosiy maqsadlari:

1. **Type Safety** — kod yozilayotganda xatolarni erta aniqlash. `undefined is not a function`, `cannot read property of null` kabi runtime xatolar compile-time da ushlash
2. **Developer Experience** — IDE da autocomplete, refactoring, go-to-definition, inline documentation kabi tooling imkoniyatlarini oshirish
3. **Code as Documentation** — tiplar o'zi documentation vazifasini bajaradi. Funksiya nima qabul qiladi va nima qaytaradi — signature dan ko'rinadi
4. **Scalability** — katta codebase larda xavfsiz refactoring, yangi developer larning kodni tushunishi osonlashadi
5. **JavaScript Compatibility** — har qanday JavaScript kodi TypeScript da ishlaydi, bosqichma-bosqich migration qilish mumkin

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript — **compile-time til**. Runtime da TypeScript mavjud emas. TypeScript compiler (tsc) source code ni oddiy JavaScript ga aylantiradi. Bu jarayonda barcha type annotation lar, interface lar, type alias lar **to'liq o'chiriladi** — bu "type erasure" deb ataladi. Natijada brauzer yoki Node.js faqat sof JavaScript oladi.

TypeScript JavaScript ustiga qo'shadigan narsalarni ikki asosiy kategoriyaga bo'lish mumkin:

```
┌─────────────────────────────────────────────────────────┐
│              TypeScript qo'shimchalari                  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  1. FAQAT TYPE SYSTEM (compile-time, o'chiriladi) │  │
│  │                                                   │  │
│  │  - Type annotations: : string, : number           │  │
│  │  - Interfaces va type aliases                     │  │
│  │  - Generics: <T>                                  │  │
│  │  - Union/Intersection types                       │  │
│  │  - Conditional va mapped types                    │  │
│  │  - Type guards, assertions, satisfies             │  │
│  │  - `private`/`protected`/`readonly` modifiers     │  │
│  │  - `import type` / `export type`                  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  2. RUNTIME KOD HOSIL QILADI (JS ga compile)      │  │
│  │                                                   │  │
│  │  - `enum` → IIFE + reverse mapping                │  │
│  │  - `namespace` → IIFE (legacy)                    │  │
│  │  - Decorators → wrapper funksiyalar               │  │
│  │  - Constructor parameter properties               │  │
│  │    (`constructor(public x: number)`)              │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

Birinchi kategoriya — type system — `tsc` emitter tomonidan to'liq o'chiriladi. Ikkinchi kategoriya — TS-specific runtime construct'lar — JavaScript kodga aylantiriladi. TS 5.8 `erasableSyntaxOnly` flag ikkinchi kategoriyani taqiqlaydi (Node.js native TS support uchun).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

TypeScript va JavaScript bir xil vazifa — farqi statik tiplar va compile-time xato ushlash:

```javascript
// JavaScript — hech qanday type information yo'q
function greet(name) {
  return "Hello, " + name;
}

greet(42);       // Runtime da ishlaydi — "Hello, 42"
greet();         // Runtime da ishlaydi — "Hello, undefined"
greet("World");  // Runtime da ishlaydi — "Hello, World"
```

```typescript
// TypeScript — tiplar bilan xavfsiz
function greet(name: string): string {
  return "Hello, " + name;
}

greet(42);
// ❌ Argument of type 'number' is not assignable to parameter of type 'string'

greet();
// ❌ Expected 1 arguments, but got 0

greet("World"); // ✅ "Hello, World"
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

TypeScript kodi JavaScript ga compile bo'lganda type annotation lar butunlay o'chiriladi:

```typescript
// TypeScript source (target: ES2022)
interface User {
  name: string;
  age: number;
}

function greet(name: string): string {
  return "Hello, " + name;
}

const user: User = { name: "Ali", age: 25 };
greet(user.name);
```

```javascript
// Compiled JavaScript
// interface User — butunlay o'chirildi (type erasure)
// : string, : User, return type — hammasi o'chirildi
function greet(name) {
  return "Hello, " + name;
}

const user = { name: "Ali", age: 25 };
greet(user.name);
```

Runtime da TypeScript iz qoldirmaydi — `User` interface JavaScript kodda umuman mavjud emas.

</details>

---

## TypeScript vs JavaScript

### Nazariya

TypeScript va JavaScript ning fundamental farqi — **tiplarni qachon tekshirish**. JavaScript **dynamically typed** — tiplar runtime da aniqlanadi va tekshiriladi (masalan, `typeof` operatori runtime da ishlaydi). TypeScript **statically typed** — tiplar compile-time da aniqlanadi va tekshiriladi. Bu fundamental farq quyidagi amaliy oqibatlarga olib keladi:

**Type Safety.** JavaScript da tip xatolari faqat runtime da, kod bajarilayotgan paytda aniqlanadi. TypeScript da bu xatolar kod yozilayotganda yoki `tsc` ishga tushirilganda darhol ko'rinadi — ishga tushirishdan oldin.

**IDE Tooling.** TypeScript ning eng katta amaliy foydasi — **IDE tooling**. Type information tufayli VS Code (va boshqa IDE lar) quyidagilarni ta'minlaydi:

- **Autocomplete (IntelliSense)** — object ning qaysi property va method lari borligini aniq ko'rsatadi
- **Go to Definition** — funksiya, class yoki type ning ta'rifi joylashgan faylga o'tish
- **Rename Symbol** — identifier ni loyiha bo'ylab xavfsiz qayta nomlash
- **Find All References** — qaerda ishlatilganini topish
- **Inline Errors** — xatolarni yozayotgan paytda darhol ko'rsatish
- **Signature Help** — funksiya chaqiruvida parametr tiplari va tartibi

**Refactoring.** Katta codebase da funksiya signature sini o'zgartirish, property nomini almashtirish yoki data model ni qayta loyihalash JavaScript da xavfli operatsiya — barcha ishlatilgan joylarni qo'lda topish kerak. TypeScript da compiler barcha nomuvofiqliklarni darhol ko'rsatadi — refactoring xavfsiz bo'ladi.

**Taqqoslash jadvali:**

| Xususiyat | JavaScript | TypeScript |
|-----------|-----------|------------|
| Type checking | Runtime (dynamic) | Compile-time (static) |
| Type annotation syntax | Yo'q (JSDoc comment'lar bilan mumkin) | Built-in — `let x: number` |
| Compile/build step | Shart emas | Shart (tsc yoki esbuild/SWC) |
| Xato aniqlash vaqti | Runtime da | Compile-time da |
| IDE autocomplete | Cheklangan (tahmin) | To'liq, type asosida |
| Xato misoli | `TypeError: Cannot read property 'x' of undefined` (runtime) | `Object is possibly 'undefined'` (compile-time) |
| Ecosystem | npm packages | npm + `@types/*` packages |
| Brauzer/Node.js da | To'g'ridan-to'g'ri ishlaydi | `.js` ga compile kerak |
| Backward compatible | — | Har qanday JS kod = to'g'ri TS kod |

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript va JavaScript ning execution pipeline lari fundamental farq qiladi:

```
JavaScript Execution Pipeline:
┌──────────────┐    ┌────────────────┐    ┌──────────────┐
│  .js source  │ →  │  V8/SpiderMnky │ →  │   Runtime    │
│  (dynamic)   │    │  parse         │    │   execution  │
│              │    │  + JIT compile │    │   (xatolar   │
│              │    │                │    │    bu yerda) │
└──────────────┘    └────────────────┘    └──────────────┘

TypeScript Compilation + Execution Pipeline:
┌──────────────┐    ┌────────────────┐    ┌──────────────┐    ┌────────────────┐    ┌──────────────┐
│  .ts source  │ →  │  tsc: type     │ →  │  tsc: emit   │ →  │  V8/SpiderMnky │ →  │   Runtime    │
│  (static     │    │  checking      │    │  (type       │    │  parse         │    │   execution  │
│   types)     │    │  (xatolar      │    │   erasure)   │    │  + JIT compile │    │              │
│              │    │   bu yerda)    │    │  → .js       │    │                │    │              │
└──────────────┘    └────────────────┘    └──────────────┘    └────────────────┘    └──────────────┘
```

JavaScript da V8 engine source code ni to'g'ridan-to'g'ri parse qiladi va JIT (Just-In-Time) compiler orqali machine code ga aylantiradi. Type ma'lumoti yo'q — engine runtime da hidden class, inline cache, shape tracking orqali type larni taxmin qiladi.

TypeScript da `tsc` **oldindan** type check qiladi — barcha xatolar runtime dan oldin aniqlanadi. Keyin type annotation lar o'chiriladi va sof JS hosil bo'ladi. Runtime da V8 uchun hech qanday farq yo'q — u oddiy JavaScript ko'radi.

TypeScript ning type system **Turing-complete** — ya'ni type darajasida ixtiyoriy hisoblash bajarish mumkin. Bu conditional types, recursive types, template literal types orqali erishiladi. Lekin bu faqat compile-time da ishlaydi — runtime da iz qolmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

JavaScript da faqat runtime da aniqlanadigan xato — TypeScript buni compile-time da ushlab oladi:

```javascript
// JavaScript — xato faqat runtime da
function calculateArea(shape) {
  if (shape.type === "circle") {
    return Math.PI * shape.radius ** 2;
  } else if (shape.type === "rectangle") {
    return shape.width * shape.height;
  }
  // Aks holda — undefined qaytaradi
}

// Typo "cicle" — JavaScript hech narsa demaydi
const area = calculateArea({ type: "cicle", radius: 5 });
console.log(area); // undefined — debugging qiyin
```

```typescript
// TypeScript — compile-time da xato
type Shape =
  | { type: "circle"; radius: number }
  | { type: "rectangle"; width: number; height: number };

function calculateArea(shape: Shape): number {
  if (shape.type === "circle") {
    return Math.PI * shape.radius ** 2;
  } else if (shape.type === "rectangle") {
    return shape.width * shape.height;
  }
  // Exhaustive check — TS barcha holatlar qamrab olinganini tekshiradi
  const _exhaustive: never = shape;
  return _exhaustive;
}

// ❌ Type '"cicle"' is not assignable to type '"circle" | "rectangle"'
const area = calculateArea({ type: "cicle", radius: 5 });
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Discriminated union bilan yozilgan type-safe shape calculation — compile natijasida oddiy JavaScript:

```typescript
// TypeScript source (target: ES2022)
type Shape =
  | { type: "circle"; radius: number }
  | { type: "rectangle"; width: number; height: number };

function calculateArea(shape: Shape): number {
  if (shape.type === "circle") {
    return Math.PI * shape.radius ** 2;
  }
  return shape.width * shape.height;
}
```

```javascript
// Compiled JavaScript
// type Shape — butunlay o'chirildi
// : Shape, : number — annotation'lar o'chirildi
function calculateArea(shape) {
  if (shape.type === "circle") {
    return Math.PI * shape.radius ** 2;
  }
  return shape.width * shape.height;
}
// Discriminated union → oddiy if/else — runtime overhead nol
```

</details>

---

## TypeScript Compiler — tsc Pipeline

### Nazariya

TypeScript Compiler (`tsc`) — TypeScript source code ni JavaScript ga aylantiradigan dastur. `tsc` o'zi ham TypeScript da yozilgan va Node.js ustida ishlaydi. `npm install -g typescript` yoki loyihada `npm install --save-dev typescript` bilan o'rnatiladi. `tsc` ning ikki asosiy vazifasi bor:

1. **Type Checking** — barcha type annotation va ifodalarni tekshirish, xatolarni aniqlash
2. **Emit** — type annotation larni olib tashlab, JavaScript output hosil qilish

Bu ikki vazifa mustaqil — `noEmit: true` bilan faqat type check qilish mumkin (masalan, CI da). Teskarisi ham mumkin: faqat emit (type check qilmasdan) — bu esbuild, SWC, Babel va `ts.transpileModule()` API orqali amalga oshiriladi. Bu tool'lar `tsc` dan **sezilarli tez** ishlaydi, chunki ular faqat syntax parsing va type annotation o'chirish bilan shug'ullanadi — to'liq type check qilmaydi.

Amaliy loyihalarda odatda ikki xil vosita ishlatiladi: **esbuild/SWC** production build uchun (tez), **tsc** yoki IDE type service type checking uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

`tsc` ning ichki pipeline'i quyidagi bosqichlardan iborat:

```
┌──────────────┐    ┌─────────┐    ┌────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
│ Source Code  │ →  │ Scanner │ →  │ Parser │ →  │   AST    │ →  │  Binder  │ →  │ Checker │
│ (.ts fayl)   │    │ (Lexer) │    │        │    │          │    │          │    │         │
└──────────────┘    └─────────┘    └────────┘    └──────────┘    └──────────┘    └────┬────┘
                                                                                       │
                                                                                       ▼
                                                                                ┌──────────┐
                                                                                │ Emitter  │
                                                                                └────┬─────┘
                                                                                     │
                                                                                     ▼
                                                                              ┌──────────────┐
                                                                              │ .js fayllar  │
                                                                              │ .d.ts        │
                                                                              │ .js.map      │
                                                                              └──────────────┘
```

**Bosqich 1: Scanner (Lexer)**

Scanner source code matnini **token** larga ajratadi. Token — tilning eng kichik ma'noli birligi. Har bir belgi bir yoki bir necha tokenga aylanadi.

```typescript
// Source:
let age: number = 25;

// Scanner tomonidan hosil qilingan tokenlar:
// LetKeyword → Identifier("age") → ColonToken → NumberKeyword
// → EqualsToken → NumericLiteral(25) → SemicolonToken
```

Scanner whitespace va commentlarni o'tkazib yuboradi (yoki alohida saqlaydi — ular `trivia` deb ataladi). TS'ning scanner'i `src/compiler/scanner.ts` da joylashgan.

**Bosqich 2: Parser**

Parser tokenlar ketma-ketligidan **AST (Abstract Syntax Tree)** qurilmasini hosil qiladi. AST — kodning daraxt ko'rinishidagi tuzilmaviy tasviri.

```typescript
// Source:
let age: number = 25;

// AST (soddalashtirilgan):
// VariableStatement
//   └── VariableDeclarationList
//         └── VariableDeclaration
//               ├── name: Identifier("age")
//               ├── type: TypeReference("number")  ← TS qo'shimchasi
//               └── initializer: NumericLiteral(25)
```

Parser **recursive descent** algoritmi bilan ishlaydi — har bir grammatik qoida uchun alohida funksiya mavjud. TS parser yana bir muhim ish qiladi: **syntax error recovery**. Xato uchrasa ham parsing davom etadi — bu IDE da real-time xatolar ko'rsatish uchun zarur.

**Bosqich 3: Binder**

Binder AST bo'ylab yurib, **Symbol** larni yaratadi va scope larni aniqlaydi. Har bir declaration (o'zgaruvchi, funksiya, class, interface) uchun `Symbol` yaratiladi. Symbol bitta entity haqidagi barcha ma'lumotni birlashtiradigan obyekt. Binder scope tree qurib, har bir symbol qaysi scope ga tegishli ekanini belgilaydi.

**Bosqich 4: Checker (Type Checker)**

Checker `tsc` ning eng katta va murakkab qismi — `checker.ts` fayli juda katta hajmli (kompiler logikasining yarmidan ko'pi shu yerda). Checker barcha type tekshiruvlarni amalga oshiradi:

- Har bir ifoda type ini aniqlaydi (type inference)
- Annotation type'lar bilan haqiqiy type larni solishtiradi (assignability check)
- Generic type'larni resolve qiladi
- Function overload resolution qiladi
- Control flow asosida type narrowing ni hisoblaydi
- Diagnostic (xato xabar) larni hosil qiladi

Checker ning asosiy printsipi — **lazy evaluation**. Barcha type'larni oldindan hisoblamasdan, kerak bo'lganda hisoblaydi. Bu katta loyihalarda performance uchun muhim.

**Bosqich 5: Emitter**

Emitter AST dan JavaScript source code hosil qiladi. Bu jarayonda:

- Barcha type annotation lar o'chiriladi
- TS-specific runtime construct'lar (enum, namespace) JS ga aylantiriladi
- `target` ga qarab eski JS versiyasiga downlevel compile qilinadi (masalan, `async/await` → generator)
- Source map (`.js.map`) fayllar hosil qilinadi (debugging uchun)
- Declaration fayllar (`.d.ts`) hosil qilinadi (`declaration: true` bo'lsa)

Emitter faqat type check muvaffaqiyatli bo'lsa ishlaydi (default). `noEmitOnError: false` bilan hatto xato bo'lsa ham JS chiqariladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`tsc` ni terminal'da ishlatish:

```bash
# TypeScript loyihaga o'rnatish
npm install --save-dev typescript

# tsc versiyasini tekshirish
npx tsc --version  # Version 5.x.x

# tsconfig.json yaratish (minimal)
npx tsc --init

# tsconfig.json bilan loyihani compile qilish
npx tsc

# Bitta faylni compile qilish (tsconfig.json ni o'qimaydi)
npx tsc hello.ts

# Watch mode — fayl o'zgarganda avtomatik compile
npx tsc --watch

# Faqat type check, emit qilmaslik (CI uchun)
npx tsc --noEmit

# Target belgilash
npx tsc --target ES2022 hello.ts
```

`tsc` pipeline ni amalda ko'rish — TypeScript Compiler API orqali:

```typescript
// Scanner bosqichini ko'rish — tokenlar ro'yxati
import * as ts from "typescript";

const sourceCode = `let age: number = 25;`;

const scanner = ts.createScanner(
  ts.ScriptTarget.Latest,
  /* skipTrivia */ true
);
scanner.setText(sourceCode);

let token = scanner.scan();
while (token !== ts.SyntaxKind.EndOfFileToken) {
  console.log(ts.SyntaxKind[token], JSON.stringify(scanner.getTokenText()));
  token = scanner.scan();
}

// Output:
// LetKeyword    "let"
// Identifier    "age"
// ColonToken    ":"
// NumberKeyword "number"
// EqualsToken   "="
// NumericLiteral "25"
// SemicolonToken ";"
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

`tsc` pipeline ning to'liq natijasi — TS source dan JS output gacha:

```typescript
// TypeScript source (target: ES2022)
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  private prefix: string;
  constructor(prefix: string) {
    this.prefix = prefix;
  }
  log(message: string): void {
    console.log(`[${this.prefix}] ${message}`);
  }
}

const logger: Logger = new ConsoleLogger("APP");
logger.log("started");
```

```javascript
// Compiled JavaScript (target: ES2022)
// 1. interface Logger — butunlay o'chirildi (type erasure)
// 2. implements Logger — o'chirildi (compile-time constraint)
// 3. private keyword — o'chirildi, lekin this.prefix property qoladi
//    MUHIM: JS da instance.prefix runtime'da ochiq — himoya yo'q
// 4. : string, : void, : Logger — barcha annotation'lar o'chirildi
class ConsoleLogger {
  constructor(prefix) {
    this.prefix = prefix;
  }
  log(message) {
    console.log(`[${this.prefix}] ${message}`);
  }
}
const logger = new ConsoleLogger("APP");
logger.log("started");
```

> **Muhim:** TypeScript ning `private` modifier faqat compile-time himoya — runtime da hech qanday ta'sir qilmaydi. Haqiqiy runtime private uchun ES2022 `#` private fields kerak ([10-classes.md](10-classes.md) da batafsil).

</details>

---

## Type Erasure — Runtime da Types Yo'q

### Nazariya

Type erasure — TypeScript ning eng fundamental xususiyati: compile jarayonida barcha type ma'lumotlari **to'liq o'chiriladi**. Natijada hosil bo'lgan JavaScript da type annotation lar, interface lar, type alias lar, generic parametrlar — bularning hech biri yo'q. Runtime da TypeScript haqida hech qanday ma'lumot qolmaydi.

Bu nima uchun shunday? TypeScript ning dizayn maqsadi — **mavjud JavaScript runtime larini o'zgartirmaslik**. Brauzerlar va Node.js JavaScript ni bajaradi. TypeScript yangi runtime yaratish o'rniga, compile-time da foydali bo'lib, keyin "yo'qolib ketadi". Bu yondashuvning oqibatlari:

1. **Runtime da type tekshirish yo'q** — compile-time da xato bo'lmasa, runtime da tip xatosi chiqmaydi. Lekin `any` yoki type assertion bilan compiler ni aldash mumkin — u holda runtime xato bo'lishi mumkin.
2. **Runtime performance ta'siri yo'q** — type annotation lar JS ga tushmaydi, runtime da qo'shimcha ish yo'q.
3. **JavaScript interop** — compile qilingan TS kodi boshqa JS kod bilan to'g'ridan-to'g'ri ishlaydi — @types/* package'lar orqali JS kutubxonalari bilan mos keladi.
4. **Type-only checks runtime da ishlamaydi** — `if (value instanceof MyInterface)` ishlamaydi, chunki interface runtime da mavjud emas.

<details>
<summary><strong>Under the Hood</strong></summary>

Type erasure ning aniq qoidalari — qaysi construct JS ga tushadi, qaysi biri o'chiriladi:

```
O'CHIRILADI (JS da iz yo'q):             SAQLANADI (JS da qoladi):
──────────────────────────────           ──────────────────────────
: string, : number, : boolean            let, const, var declarations
interface User { ... }                   function bodies
type Alias = string | number             class bodies (methods, fields)
<T> generic type parameters              if/else, for, while logic
as Type (type assertion)                 import/export (value imports)
satisfies Type                           object literals, arrays
x! (non-null assertion)                  enum → IIFE (runtime kod)
import type { X }                        decorator → wrapper funksiya
readonly/public/private modifiers        constructor parameter → assign
```

Type erasure'ni `tsc` emitter amalga oshiradi. U AST bo'ylab yurib, type-related node'larni olib tashlaydi va qolgan JavaScript construct'larni output ga yozadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Type erasure ni amalda — har xil TypeScript construct'lar uchun:

```typescript
// TS SOURCE
interface Product {
  id: number;
  name: string;
  price: number;
}

type Catalog = readonly Product[];

function getTotal(items: Catalog): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

const products: Catalog = [
  { id: 1, name: "Laptop", price: 999 },
  { id: 2, name: "Mouse", price: 29 },
];

const total: number = getTotal(products);
```

Type erasure ning amaliy oqibati — runtime da interface bilan tip tekshirish **mumkin emas**:

```typescript
interface Cat {
  meow(): void;
}

interface Dog {
  bark(): void;
}

function makeSound(animal: Cat | Dog) {
  // ❌ ISHLAMAYDI — interface runtime da yo'q
  // if (animal instanceof Cat) { ... }
  // SyntaxError/ReferenceError: 'Cat' only refers to a type

  // ✅ Runtime da mavjud bo'lgan property ni tekshirish kerak
  if ("meow" in animal) {
    animal.meow(); // TS narrowing — animal: Cat
  } else {
    animal.bark(); // TS narrowing — animal: Dog
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Yuqoridagi `Product`/`Catalog` misoli compile bo'lganda:

```javascript
// Compiled JavaScript — type ma'lumotlari to'liq o'chirilgan
// interface Product — yo'q
// type Catalog — yo'q

function getTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

const products = [
  { id: 1, name: "Laptop", price: 999 },
  { id: 2, name: "Mouse", price: 29 },
];

const total = getTotal(products);
```

JS kodda faqat qiymatlar va logika qoladi. Type system butunlay "yo'qoladi" — compile vaqtida vazifasini bajargan va keyin kerak emas.

</details>

---

## Structural Typing vs Nominal Typing

### Nazariya

Dasturlash tillarida type compatibility (tiplar mosligini) aniqlashning ikki fundamental yondashuvi bor:

**Nominal Typing** (Java, C#, Swift, Rust):
Ikki tip mos keladi faqatgina ular **bir xil nom**ga ega bo'lsa yoki aniq **inheritance zanjiri** bilan bog'langan bo'lsa. Nomlar muhim — tuzilma bir xil bo'lsa ham, nom farq qilsa, tiplar mos kelmaydi.

```java
// Java — Nominal Typing
class UserId { int value; }
class ProductId { int value; }

// UserId va ProductId tuzilmasi bir xil, LEKIN
// ular turli class — shuning uchun mos kelmaydi
UserId userId = new ProductId(1); // ❌ Compile error!
```

**Structural Typing** (TypeScript, Go interfaces, OCaml object types):
Ikki tip mos keladi agar ularning **tuzilmasi (shape)** mos bo'lsa. Nomlar ahamiyatsiz — agar type A ning barcha kerakli property va method lari type B da mavjud bo'lsa, A ni B o'rnida ishlatish mumkin.

TypeScript aynan structural typing ishlatadi. Bu JavaScript ning tabiatiga mos keladi — JavaScript da object ning qaysi property va method lari borligiga qarab "type" aniqlanadi (bu duck typing deb ataladi: *"agar qush kabi yursa va qush kabi qichqirsa, demak qush"*). TypeScript shu yondashuvni compile-time da rasmiylashtiradi — class yoki interface nomi emas, **shape (tuzilma)** bo'yicha tip tekshiradi.

Structural typing'ning afzalligi — soddalik va JavaScript pattern'lari bilan tabiiy mos kelish. Kamchiligi — ba'zan **mantiqan farqli** lekin **tuzilmasi bir xil** tiplarni aralashtirish mumkin (masalan, `UserId` va `ProductId` ikkalasi ham `number` bo'lsa).

<details>
<summary><strong>Kod Misollari</strong></summary>

Structural typing amalda:

```typescript
interface Point {
  x: number;
  y: number;
}

// Bu class `Point` ni implement qilmagan —
// lekin tuzilmasi mos keladi
class Coordinate {
  constructor(public x: number, public y: number) {}
}

function logPoint(point: Point): void {
  console.log(`(${point.x}, ${point.y})`);
}

const coord = new Coordinate(10, 20);
logPoint(coord); // ✅ Ishlaydi — Coordinate shape'i Point'ga mos keladi

// Oddiy object ham ishlaydi
logPoint({ x: 5, y: 15 }); // ✅ Ishlaydi
```

Structural typing'ning mantiqiy kamchiligi:

```typescript
type UserId = number;
type ProductId = number;

function getUser(id: UserId) { /* ... */ }

const productId: ProductId = 42;
getUser(productId); // ✅ TS xato bermaydi — ikkalasi ham oddiy number
// Bu mantiqiy xato, lekin structural typing buni ushlamaydi
// Yechim: Branded types ([23-type-safe-patterns.md](23-type-safe-patterns.md) da batafsil)
```

> **Batafsil:** Structural typing, covariance, contravariance, bivariance va variance annotation'lar haqida to'liq ma'lumot [25-type-compatibility.md](25-type-compatibility.md) da yoritiladi.

</details>

---

## Fayl Kengaytmalari — .ts, .tsx, .d.ts, .mts, .cts

### Nazariya

TypeScript ekosistemida bir nechta fayl kengaytmalari mavjud, har birining aniq maqsadi bor:

**`.ts` — TypeScript source fayl**
Asosiy TypeScript fayl kengaytmasi. Type annotation lar, interface lar, generics — barcha TypeScript xususiyatlari yoziladi. `tsc` buni `.js` ga compile qiladi.

**`.tsx` — TypeScript + JSX**
React yoki boshqa JSX ishlatadigan framework larda TypeScript fayllari `.tsx` bo'ladi. `.tsx` faylda JSX syntax (`<Component />`) yozish mumkin. Muhim farq: `.tsx` da **angle-bracket type assertion** ishlamaydi — `<string>value` syntax JSX bilan konflikt qiladi. `.tsx` da faqat `value as string` ishlaydi.

**`.d.ts` — Declaration fayl**
Faqat type ma'lumotlarini saqlaydi — implementation yo'q. Maqsadlari:
- JavaScript kutubxonalar uchun type information berish (masalan, `@types/node`, `@types/lodash`)
- Compile qilingan TypeScript kutubxonaning type larini eksport qilish
- Global type declaration lar (ambient declarations, masalan `declare global { ... }`)

`.d.ts` fayllar hech qachon JavaScript ga compile qilinmaydi — ular faqat type checker uchun.

**`.mts` — TypeScript ES Module**
Node.js da ESM (ES Modules) formatidagi TypeScript fayl. Compile bo'lganda `.mjs` ga aylanadi. `.mts` fayl doim ESM sifatida qaraladi — `import`/`export` syntax ishlatiladi, fayl qaysi papkada ekani ahamiyatsiz.

**`.cts` — TypeScript CommonJS Module**
Node.js da CJS (CommonJS) formatidagi TypeScript fayl. Compile bo'lganda `.cjs` ga aylanadi. `.cts` fayl doim CJS sifatida qaraladi — `require`/`module.exports` ishlatiladi.

**Fayl kengaytma ↔ compile natijasi jadvali:**

| Source | Compile natijasi | Eslatma |
|--------|------------------|---------|
| `app.ts` | `app.js` | Standart TS fayl |
| `Button.tsx` | `Button.js` | JSX compile qilinadi |
| `types.d.ts` | (compile qilinmaydi) | Faqat type info |
| `server.mts` | `server.mjs` | ESM forced |
| `config.cts` | `config.cjs` | CJS forced |

> **Batafsil:** Module system, `moduleResolution` strategiyalari, ESM/CJS interop va `.mts`/`.cts` ishlatish paternlari [17-modules.md](17-modules.md) da yoritiladi.

---

## tsconfig.json Asoslari

### Nazariya

`tsconfig.json` — TypeScript loyihasining konfiguratsiya fayli. `tsc` shu faylni o'qib, qanday compile qilishni aniqlaydi. Odatda loyihaning root papkasida joylashadi. `tsc` ni hech qanday argumentsiz ishga tushirganingizda — u joriy papkada va ota papkalarda `tsconfig.json` ni qidiradi.

`tsconfig.json` ning asosiy bo'limlari:

1. **`compilerOptions`** — compiler sozlamalari (`target`, `module`, `strict`, va yuzlab boshqa flag'lar)
2. **`include`** — qaysi fayllar compile qilinadi (glob pattern, masalan `src/**/*.ts`)
3. **`exclude`** — qaysi fayllar tashlab ketiladi (odatda `node_modules`, `dist`)
4. **`files`** — aniq fayl ro'yxati (`include` o'rniga)
5. **`extends`** — boshqa tsconfig dan meros olish (masalan `@tsconfig/node20`)
6. **`references`** — project references (monorepo'lar uchun)

<details>
<summary><strong>Kod Misollari</strong></summary>

Minimal, lekin production-ready `tsconfig.json`:

```json
{
  "compilerOptions": {
    // === Target va Module ===
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",

    // === Output ===
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "sourceMap": true,

    // === Type Checking ===
    "strict": true,
    "noUncheckedIndexedAccess": true,

    // === Interop ===
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "isolatedModules": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

Turli loyiha turlari uchun qo'shimcha sozlamalar:

```json
// React (Vite) loyihasi
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "moduleResolution": "bundler"
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"]
}
```

```json
// Node.js library
{
  "compilerOptions": {
    "module": "NodeNext",
    "types": ["node"],
    "declaration": true,
    "declarationMap": true
  }
}
```

> **Batafsil:** `tsconfig.json` ning barcha opsiyalari, `extends` merge qoidalari, `references` monorepo setup va tez-tez uchraydigan konfiguratsiya pattern'lari [22-tsconfig.md](22-tsconfig.md) da yoritiladi.

</details>

---

## strict: true Nima Qiladi

### Nazariya

`strict: true` — `tsconfig.json` da bir qator strict type-checking flag larni **birdan yoqadigan** meta-flag. Yangi loyihada doim `strict: true` bilan boshlash tavsiya etiladi. `strict: true` quyidagi flag larning hammasini yoqadi:

| Flag | Nima qiladi |
|------|-------------|
| `strictNullChecks` | `null` va `undefined` ni har bir tipdan ajratadi. `string` ga `null` assign qilib bo'lmaydi — faqat `string \| null` ga |
| `strictFunctionTypes` | Funksiya **type syntax**'idagi (`(x: Dog) => void`) parametrlarini contravariant tekshiradi. **Method syntax** (`handle(x: Dog): void`) esa bivariant qoladi (backward compat) |
| `strictBindCallApply` | `.bind()`, `.call()`, `.apply()` parametrlarini to'g'ri tekshiradi |
| `strictPropertyInitialization` | Class property'lari constructor da initialize qilinganini tekshiradi |
| `noImplicitAny` | Tip aniqlanmasa `any` deb taxmin qilmaslik — aniq tip yozish kerak |
| `noImplicitThis` | `this` type'i noma'lum bo'lsa xato berish |
| `alwaysStrict` | Har bir faylga `"use strict"` qo'shish (ESM modullarda tushiriladi — ES modullar allaqachon strict rejimda) |
| `useUnknownInCatchVariables` | `catch(e)` da `e` ning type'i `unknown` (TS 4.0 da kiritilgan, TS 4.4 da `strict` ostiga qo'shilgan) |

`strict: true` yozib, individual flag'larni alohida o'chirish ham mumkin:

```json
{
  "compilerOptions": {
    "strict": true,
    "strictPropertyInitialization": false
    // strict dan faqat shu bitta flag o'chirildi
  }
}
```

> **Batafsil:** Har bir strict sub-flag'ning aniq xulqi, variance (covariance/contravariance/bivariance) va method syntax bivariance istisnosi [25-type-compatibility.md](25-type-compatibility.md) va [22-tsconfig.md](22-tsconfig.md) da yoritiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng ko'p ishlatiladigan uchta strict flag amalda:

**`strictNullChecks`:**

```typescript
// strict: false — null/undefined hech qayerda tekshirilmaydi
function getLength(str: string): number {
  return str.length; // str null bo'lsa → runtime crash
}
getLength(null); // Hech qanday xato — runtime'da crash

// strict: true — null alohida handle qilinishi SHART
function getLength(str: string | null): number {
  if (str === null) return 0; // ✅ null tekshirish majburiy
  return str.length;
}
getLength(null); // ✅ Xavfsiz — 0 qaytaradi
```

**`noImplicitAny`:**

```typescript
// strict: true — parameter type majburiy
function process(data) { // ❌ Parameter 'data' implicitly has an 'any' type
  return data.name;
}

function process(data: { name: string }) { // ✅ Aniq type
  return data.name;
}
```

**`useUnknownInCatchVariables`:**

```typescript
// strict: true — catch variable unknown
try {
  riskyOperation();
} catch (error) {
  // error: unknown — avval type ni tekshirish kerak
  if (error instanceof Error) {
    console.log(error.message); // ✅ Narrowing'dan keyin xavfsiz
  } else {
    console.log("Unknown error:", error);
  }
}
```

</details>

---

## TypeScript Playground

### Nazariya

TypeScript Playground — brauzerda TypeScript kodni yozib, compile natijasini real-time ko'rish imkonini beradigan rasmiy online muhit. URL: [typescriptlang.org/play](https://www.typescriptlang.org/play)

Playground ning asosiy imkoniyatlari:

1. **TS → JS** — chap tomonda TypeScript, o'ng tomonda compiled JavaScript (real-time)
2. **Compiler Options** — `tsconfig.json` sozlamalarini UI orqali o'zgartirish
3. **Errors panel** — type xatolarni inline va pastda ko'rish
4. **Types hover** — cursor ni o'zgaruvchi ustiga olib borganda inferred type ni ko'rish
5. **Share** — kodni URL orqali ulashish (butun kod URL'ga encode qilinadi)
6. **Versiya tanlash** — TypeScript ning har xil versiyalari bilan sinab ko'rish
7. **AST viewer** — kodning AST tuzilmasini ko'rish (plugin orqali)

Playground qachon ishlatiladi:

- Yangi TS feature'ni tezda sinab ko'rish (o'rnatishsiz)
- Type xatoni tushunish va debug qilish
- `target` flag ta'sirini ko'rish (ES5 vs ES2022 — qanday farq qiladi)
- StackOverflow, GitHub Issues'da TS muammosini boshqalarga ulashish
- Turli TypeScript versiyalarida kod xulqini taqqoslash

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript Playground brauzerda to'liq `tsc` ni ishga tushiradi — server-side hech narsa yo'q:

```
Playground Architecture:
┌─────────────────────────────────────────────────────┐
│  Browser                                            │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  Monaco Editor (VS Code ning editor core)     │  │
│  │  - Syntax highlighting                        │  │
│  │  - Autocomplete (IntelliSense)                │  │
│  │  - Inline error diagnostics                   │  │
│  └──────────────────┬────────────────────────────┘  │
│                     │ Har bir keystroke da          │
│                     ▼                               │
│  ┌───────────────────────────────────────────────┐  │
│  │  TypeScript Compiler (typescript.js)          │  │
│  │  - Bundle hajmi ~10 MB (compressed)           │  │
│  │  - Web Worker ichida ishlaydi                 │  │
│  │  - ts.createLanguageService() API orqali      │  │
│  │  - In-memory virtual file system              │  │
│  └──────────────────┬────────────────────────────┘  │
│                     │                               │
│                     ▼                               │
│  ┌───────────────────────────────────────────────┐  │
│  │  Output panels: .JS / .D.TS / Errors          │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

Monaco Editor — VS Code ning asosiy editor komponenti. U TypeScript Language Service bilan integratsiyalangan — shu sababli Playground da VS Code dagi kabi autocomplete, hover type info va inline diagnostics ishlaydi. `tsc` `Web Worker` ichida ishlaydi — bu main thread ni bloklamasdan type checking va emit qilish imkonini beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Playground da sinab ko'rish uchun klassik misol — enum compile natijasi:

```typescript
// Playground ga yozing — o'ng tomonda JS natijasini ko'ring

enum Direction {
  Up,
  Down,
  Left,
  Right
}

const dir: Direction = Direction.Up;
console.log(Direction[dir]); // "Up" (reverse mapping)
```

Playground o'ng panelda quyidagi JavaScript'ni ko'rsatadi:

```javascript
// Compiled JS — enum IIFE + reverse mapping
var Direction;
(function (Direction) {
    Direction[Direction["Up"] = 0] = "Up";
    Direction[Direction["Down"] = 1] = "Down";
    Direction[Direction["Left"] = 2] = "Left";
    Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));

const dir = Direction.Up;
console.log(Direction[dir]); // "Up"
```

Bu aynan enum'ning "runtime kod hosil qiladi" deganimizning amaliy namunasi — type erasure'dan farqli, enum JavaScript'da IIFE va reverse mapping ni qoldiradi.

</details>

---

## TypeScript Versiyalar Tarixi

### Nazariya

TypeScript 2012-yildan beri faol rivojlanib kelmoqda. Quyida major versiyalar va ularda kiritilgan muhim xususiyatlar:

| Versiya | Yil | Muhim xususiyatlar |
|---------|-----|-------------------|
| **1.0** | 2014 | Birinchi stable release — generics, classes, modules |
| **1.4** | 2015 | Union types (`string \| number`), `let`/`const` |
| **1.5** | 2015 | ES6 modules (`import`/`export`), destructuring, spread |
| **1.6** | 2016 | `.tsx` fayl kengaytmasi, intersection types (`&`), user-defined type guards |
| **2.0** | 2016 | **`strictNullChecks`**, `never` type, tagged (discriminated) unions, `readonly` |
| **2.1** | 2016 | `keyof`, mapped types, `Partial<T>`, `Readonly<T>` |
| **2.3** | 2017 | **`strict: true`** meta-flag, async iteration |
| **2.8** | 2018 | **Conditional types** (`T extends U ? X : Y`), `infer` keyword |
| **3.0** | 2018 | Project references, **`unknown`** type |
| **3.4** | 2019 | **`as const`** (const assertions), higher-order type inference |
| **3.7** | 2019 | Optional chaining (`?.`), nullish coalescing (`??`), assertion functions |
| **4.0** | 2020 | Variadic tuple types, labeled tuple elements |
| **4.1** | 2020 | **Template literal types**, key remapping in mapped types, recursive conditional types |
| **4.3** | 2021 | `override` keyword, separate getter/setter types |
| **4.4** | 2021 | `useUnknownInCatchVariables` `strict` ostiga qo'shildi |
| **4.7** | 2022 | `in`/`out` variance annotations, Node.js ESM support |
| **4.9** | 2022 | **`satisfies`** operator, auto-accessors (`accessor` keyword) |
| **5.0** | 2023 | **TC39 Decorators** (Stage 3), `const` type parameters, `moduleResolution: "bundler"` |
| **5.2** | 2023 | `using` declarations (Explicit Resource Management) |
| **5.4** | 2024 | `NoInfer<T>` utility type, preserved narrowing in closures |
| **5.5** | 2024 | `isolatedDeclarations`, inferred type predicates |
| **5.8** | 2025 | `erasableSyntaxOnly` — Node.js native TS support bilan integratsiya |

**Muhim milestone'lar:**

**TS 2.0 — `strictNullChecks`** (2016): Bu versiyadan oldin `null` va `undefined` har qanday type ga assign qilish mumkin edi — minglab runtime xatolarga sabab. `strictNullChecks` bu muammoni hal qildi: `string` va `string | null` endi farqli tip'lar.

**TS 2.8 — Conditional types** (2018): Type system'ga "if/else" kiritildi. `infer` keyword bilan birga type-level programming imkonini berdi. `Exclude`, `Extract`, `ReturnType` kabi utility'lar conditional types ustida qurilgan.

**TS 4.1 — Template literal types** (2020): String manipulation type darajasida — `` type EventName = `${string}Changed` ``. Type-safe routing, event system va CSS class validation uchun asos.

**TS 4.9 — `satisfies`** (2022): Oddiy type annotation literal type'larini widening qiladi, `as const` esa faqat o'zgarmas. `satisfies` bu ikki yondashuv o'rtasidagi bo'shliqni to'ldiradi — type check qiladi lekin inference saqlanadi.

**TS 5.0 — TC39 Decorators** (2023): Standart decorator'lar (Stage 3) qo'llab-quvvatlandi. Avvalgi `experimentalDecorators` legacy bo'lib qoldi (Angular, NestJS hozircha legacy'ni ishlatadi).

**TS 5.8 — `erasableSyntaxOnly`** (2025): Node.js 22.6+ `--experimental-strip-types` flag bilan TypeScript fayllarini to'g'ridan-to'g'ri bajara boshlagani sababli, faqat type erasure orqali o'chadigan syntax'ni majburlaydigan bu flag kiritildi. `enum`, `namespace`, constructor parameter properties, `experimentalDecorators` — bular runtime kod hosil qiladi, shuning uchun native TS support'da ishlamaydi. `erasableSyntaxOnly: true` ularni compile-time'da taqiqlaydi. Node.js 23.6'dan boshlab `--experimental-strip-types` default yoqilgan.

> **Batafsil:** TS 5.x yangiliklari, `satisfies`, `const` type parameters, `using` declarations, Decorators va boshqa zamonaviy xususiyatlar [26-ts-5x-features.md](26-ts-5x-features.md) da batafsil yoritiladi.

---

## Edge Cases va Gotchas

TypeScript ning asoslarini o'rganishda odatda ko'rinmas bo'lgan nozik holatlar. Bular ko'pincha production loyihalarda kutilmagan xatolar sifatida namoyon bo'ladi.

### 🕳 Gotcha 1: `instanceof` interface bilan ishlamaydi — type erasure oqibati

```typescript
interface Animal {
  name: string;
}

function check(value: unknown) {
  // ❌ ISHLAMAYDI
  // if (value instanceof Animal) { ... }
  // Compile-time xato: 'Animal' only refers to a type, but is being used as a value here

  // ✅ Shape check bilan narrowing
  if (typeof value === "object" && value !== null && "name" in value) {
    // value: object & Record<"name", unknown>
  }
}
```

**Sabab:** `interface` compile-time construct — runtime'da JavaScript'da mavjud emas. `instanceof` esa runtime operator bo'lib, class (constructor function) bilan ishlaydi. Type erasure'dan keyin interface yo'qoladi va `instanceof Animal` yozilsa, JavaScript `Animal` nomli qiymat topilmaydi deb xato beradi.

---

### 🕳 Gotcha 2: `strict: false → strict: true` — mavjud loyihada yuzlab xatolar

```json
// Eski loyiha
{ "compilerOptions": { "strict": false } }

// Yangilangan
{ "compilerOptions": { "strict": true } }
// ❌ Birdan 500+ ta type error paydo bo'ladi
```

**Sabab:** `strict: false` bilan TypeScript barcha o'zgaruvchi va parameter'lar `any` deb qaraydi (`noImplicitAny: false`), `null`/`undefined` har tipga assign bo'ladi (`strictNullChecks: false`), va hokazo. `strict: true` yoqilganda bu "noxushlik"lar barchasi bir zumda aniqlanadi.

**Yechim:** Bosqichma-bosqich migration. Avval faqat `strictNullChecks: true` yoqish, keyin `noImplicitAny: true`, va hokazo. Har flag alohida fix qilinadi. Yoki yangi fayllar uchun `// @ts-strict-ignore` / `// @ts-check` bilan mixed mode.

---

### 🕳 Gotcha 3: `tsc` versiyasi va VS Code versiyasi mos kelmasligi

```bash
# Loyihada
npm install --save-dev typescript@5.4

# Lekin VS Code built-in TypeScript 5.2 ishlatadi
# Natija: IDE'da bir xil xato, tsc'da boshqacha
```

**Sabab:** VS Code o'z ichida TypeScript bundle'ini saqlaydi. Loyihaning `node_modules/typescript` esa boshqa versiya bo'lishi mumkin. Natijada IDE feedback va `tsc` command-line natijasi farq qiladi.

**Yechim:** VS Code'da `Cmd+Shift+P` → "TypeScript: Select TypeScript Version" → "Use Workspace Version". Bundan keyin VS Code loyihadagi versiyani ishlatadi.

---

### 🕳 Gotcha 4: `skipLibCheck: false` bilan `node_modules` type xatolari

```json
{
  "compilerOptions": {
    "skipLibCheck": false
  }
}
```

`skipLibCheck: false` bilan `tsc` sizning kodingiz uchun ham, `node_modules` ichidagi barcha `.d.ts` fayllari uchun ham type check qiladi. Agar biror package'da type xato bo'lsa — sizning `tsc` commanda yuzlab xato ko'rinadi, holbuki sizning kodingiz to'g'ri.

**Sabab:** `@types/*` va library'lar o'z type'larida ba'zan xatolarga yo'l qo'yadi (ayniqsa eski versiyalarda). `skipLibCheck: true` (default tavsiya) bu tekshirishni o'tkazib yuboradi.

**Yechim:** Production loyihalarda doim `"skipLibCheck": true`. Library yozayotgan bo'lsangiz — o'z `.d.ts` fayllarini alohida test qiling.

---

### 🕳 Gotcha 5: JS loyihani TS'ga migratsiya — `allowJs` va `checkJs` chalkashligi

```json
{
  "compilerOptions": {
    "allowJs": true,    // JS fayllarni kompilyatsiyaga qo'shish
    "checkJs": false    // Lekin ularni type check qilmaslik
  }
}
```

`allowJs: true` bilan `.js` fayllar ham loyihaga kiradi, lekin `checkJs: false` bo'lsa ular tekshirilmaydi. `checkJs: true` yoqilsa — JS fayllarda ham type xatolar chiqa boshlaydi. Migration paytida bu ikki flagni noto'g'ri konfiguratsiya qilsangiz, loyiha ham ishlamaydi, ham to'liq tekshirilmagan bo'ladi.

**Yechim:** Migration bosqichlari:
1. `"allowJs": true, "checkJs": false` — JS fayllar compile bo'ladi
2. Asta-sekin `.js` → `.ts`/`.tsx` ga o'tkazish
3. Migration tugagach `allowJs: false` qilish

---

## Common Mistakes

### ❌ Xato 1: TypeScript runtime da type tekshiradi deb o'ylash

```typescript
interface User {
  name: string;
  age: number;
}

// API dan kelgan data ni TypeScript avtomatik tekshirmaydi
const data: User = await fetch("/api/user").then(r => r.json());
// ❌ XAVFLI — API boshqa format qaytarsa, runtime xato
// .json() any qaytaradi, TS hech narsa tekshirmaydi

// ✅ To'g'ri — runtime validation (Zod, io-ts yoki manual)
import { z } from "zod";

const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

const data = UserSchema.parse(
  await fetch("/api/user").then(r => r.json())
);
// Endi runtime'da ham type xavfsiz
```

**Nima uchun:** TypeScript faqat compile-time da type tekshiradi. Runtime'da API javob, user input, JSON parse, `localStorage` — bularning barchasi TypeScript nazoratidan tashqarida. Tashqi ma'lumotlar uchun runtime validation majburiy.

---

### ❌ Xato 2: `any` ni hamma joyda ishlatish

```typescript
// ❌ any — TypeScript'ni butunlay o'chiradi
function processData(data: any) {
  return data.users.map((u: any) => u.name.toUpperCase());
  // Hech qanday type check — runtime crash xavfi
}

// ✅ unknown — xavfsiz alternativa
function processData(data: unknown) {
  if (
    typeof data === "object" && data !== null &&
    "users" in data && Array.isArray((data as { users: unknown[] }).users)
  ) {
    // Narrowing'dan keyin xavfsiz operatsiyalar
  }
}

// ✅ Eng yaxshi — aniq type
interface ApiResponse {
  users: { name: string }[];
}

function processData(data: ApiResponse) {
  return data.users.map(u => u.name.toUpperCase());
  // ✅ To'liq type safety + autocomplete
}
```

**Nima uchun:** `any` bilan TypeScript'ning eng katta foydasi — type safety — yo'qoladi. `unknown` xavfsiz alternativa — majburiy narrowing talab qiladi. Yangi kodda `any` deyarli hech qachon kerak emas.

---

### ❌ Xato 3: Interface runtime'da mavjud deb o'ylash

```typescript
interface Serializable {
  serialize(): string;
}

// ❌ Kompilyatsiya xatosi
function save(obj: unknown) {
  // if (obj instanceof Serializable) { ... }
  // 'Serializable' only refers to a type, but is being used as a value here
}

// ✅ Type guard pattern — narrowing bir joyda
function isSerializable(obj: unknown): obj is Serializable {
  return (
    typeof obj === "object" && obj !== null &&
    "serialize" in obj &&
    typeof (obj as { serialize: unknown }).serialize === "function"
  );
}

function save(obj: unknown): string | null {
  return isSerializable(obj) ? obj.serialize() : null;
}
```

**Nima uchun:** `instanceof` operator faqat class (constructor function) bilan ishlaydi. Interface, type alias — compile-time construct'lar, type erasure'dan keyin JS'da mavjud emas. Type guard funksiya (`obj is Serializable`) bu pattern'ning eng toza yechimi.

---

### ❌ Xato 4: Yangi loyihani `strict: false` bilan boshlash

```json
// ❌ Yangi loyihada strict o'chirib boshlash
{ "compilerOptions": { "strict": false } }

// ✅ Doim strict: true bilan boshlash
{ "compilerOptions": { "strict": true } }
```

**Nima uchun:** `strict: false` bilan boshlasangiz, loyiha o'sganidan keyin `strict: true` ga o'tish juda qiyin bo'ladi — yuzlab xatolar birdan chiqadi (yuqoridagi Edge Cases Gotcha 2 ga qarang). Boshidan `strict: true` qo'yib, xavfsiz kod yozishga odatlanish yagona to'g'ri yo'l.

---

### ❌ Xato 5: Type assertion'ni ortiqcha ishlatish

```typescript
// ❌ as bilan compiler'ni aldash — runtime xato xavfi
const input = document.getElementById("name") as HTMLInputElement;
console.log(input.value);
// Agar element topilmasa yoki input bo'lmasa — runtime crash

// ✅ Avval mavjudligini va turini tekshirish
const element = document.getElementById("name");
if (element instanceof HTMLInputElement) {
  console.log(element.value); // ✅ Xavfsiz — type narrowing
}
```

**Nima uchun:** `as` type assertion — compiler'ga "men bilaman, sen bilmaysan" deyish. Agar aslida noto'g'ri bo'lsa, compiler sukut saqlaydi va xato faqat runtime'da chiqadi. Type guard (narrowing) yondashuv har doim xavfsizroq.

---

## Amaliy Mashqlar

### Mashq 1: Type Annotation lar (Oson)

**Savol:** Quyidagi JavaScript funksiyaga TypeScript type annotation lar qo'shing — parametrlar va return type:

```typescript
function createUser(name, age, isActive) {
  return {
    id: Math.random().toString(36).slice(2),
    name,
    age,
    isActive,
    createdAt: new Date()
  };
}
```

<details>
<summary>Javob</summary>

```typescript
interface User {
  id: string;
  name: string;
  age: number;
  isActive: boolean;
  createdAt: Date;
}

function createUser(name: string, age: number, isActive: boolean): User {
  return {
    id: Math.random().toString(36).slice(2),
    name,
    age,
    isActive,
    createdAt: new Date()
  };
}
```

**Tushuntirish:** Har parametr uchun `: string`, `: number`, `: boolean` annotation va return type uchun `User` interface. Return type'ni yozmaslik ham mumkin (inference), lekin public API funksiyalar uchun aniq yozish best practice.

</details>

---

### Mashq 2: Compile Output (O'rta)

**Savol:** Quyidagi TypeScript kodi JavaScript ga compile bo'lganda nimaga aylanadi?

```typescript
interface Config {
  host: string;
  port: number;
  debug?: boolean;
}

type Environment = "development" | "production" | "staging";

function createConfig(env: Environment): Config {
  const config: Config = {
    host: "localhost",
    port: env === "production" ? 443 : 3000,
  };

  if (env !== "production") {
    config.debug = true;
  }

  return config;
}

const cfg: Config = createConfig("development");
```

<details>
<summary>Javob</summary>

```javascript
// interface Config — butunlay o'chirildi
// type Environment — butunlay o'chirildi

function createConfig(env) {
  const config = {
    host: "localhost",
    port: env === "production" ? 443 : 3000,
  };

  if (env !== "production") {
    config.debug = true;
  }

  return config;
}

const cfg = createConfig("development");
```

**Tushuntirish:** `interface Config` va `type Environment` type erasure bilan to'liq o'chirildi — ular faqat compile-time tushunchalar. Barcha `: Config`, `: Environment`, `: string`, `: number` annotation'lar ham o'chirildi. Faqat JavaScript logikasi qoldi.

</details>

---

### Mashq 3: Structural Typing (Qiyin)

**Savol:** Quyidagi kodda har bir `greet` chaqiruvi uchun natija nimaga teng? Qaysi biri compile xato beradi?

```typescript
interface Greetable {
  name: string;
  greet(): string;
}

function greet(obj: Greetable): string {
  return obj.greet();
}

class Person {
  constructor(public name: string) {}
  greet() { return `Hi, I'm ${this.name}`; }
}

class Robot {
  name = "R2D2";
  greet() { return `Beep boop, I'm ${this.name}`; }
  recharge() { return "Charging..."; }
}

const plain = {
  name: "Anonymous",
  greet() { return "Hello!"; },
  extra: 42
};

// A
greet(new Person("Ali"));

// B
greet(new Robot());

// C
greet(plain);

// D
greet({ name: "Test", greet() { return "Hey"; }, extra: 42 });
```

<details>
<summary>Javob</summary>

```
A: "Hi, I'm Ali"         ✅ Person shape'i Greetable'ga mos
B: "Beep boop, I'm R2D2" ✅ Robot ham Greetable shape'ga mos
                            (qo'shimcha recharge() ta'sir qilmaydi)
C: "Hello!"              ✅ plain variable orqali — excess property
                            check YO'Q (variable orqali)
D: ❌ Compile error      Excess property 'extra' — object literal
                            to'g'ridan-to'g'ri berilganda TS
                            qo'shimcha property'larni tekshiradi
```

**Tushuntirish:** A, B, C variantlar structural typing tufayli ishlaydi. `Person` va `Robot` `Greetable`'ni `implements` qilmagan, lekin shape'i mos (`name` + `greet` bor). `plain` variable orqali berilgan — bu excess property check'ni o'tkazib yuboradi. D'da object literal to'g'ridan-to'g'ri argument — TypeScript **excess property checking** qiladi (`extra: 42` ortiqcha).

> **Batafsil:** Excess property checking va object type'lar [04-objects-interfaces.md](04-objects-interfaces.md) da yoritiladi.

</details>

---

## Xulosa

Bu bo'limda TypeScript ning asosiy tushunchalari bilan tanishdik:

- **TypeScript** — JavaScript ustiga qurilgan statik tiplangan til. Compile-time'da type xavfsizligini ta'minlaydi, runtime'da iz qoldirmaydi.
- **TypeScript vs JavaScript** — fundamental farqi static vs dynamic typing. TypeScript'ning asosiy foydasi — tooling (IDE autocomplete, refactoring, inline errors).
- **tsc Pipeline** — Scanner → Parser → AST → Binder → Checker → Emitter. Har bosqich o'z vazifasi: lexical analysis, syntax analysis, symbol binding, type checking, JavaScript emission.
- **Type Erasure** — TypeScript ning fundamental xususiyati. Kompile vaqtida barcha type ma'lumotlari o'chiriladi, runtime'da sof JavaScript qoladi.
- **Structural Typing** — TypeScript tip mosligini nom bo'yicha emas, **shape (tuzilma)** bo'yicha aniqlaydi. JavaScript'ning duck typing'ini compile-time'da rasmiylashtiradi.
- **Fayl kengaytmalari** — `.ts`, `.tsx`, `.d.ts`, `.mts`, `.cts` — har birining aniq maqsadi bor.
- **`tsconfig.json`** — TypeScript loyihasining konfiguratsiyasi. `strict: true` bilan boshlash majburiy.
- **`strict: true`** — bir nechta strict sub-flag'larni birdan yoqadigan meta-flag: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes` va boshqalar.
- **TypeScript Playground** — brauzerda TS kodni sinab ko'rish va compile natijasini real-time ko'rish uchun rasmiy tool.
- **Versiyalar tarixi** — TS 2.0 `strictNullChecks`, 2.8 conditional types, 4.1 template literals, 4.9 `satisfies`, 5.0 TC39 decorators — eng muhim milestone'lar.
- **Edge Cases** — `instanceof` interface bilan ishlamaydi, `strict` migration muammolari, VS Code/tsc versiya mos kelmasligi, `skipLibCheck`, `allowJs`/`checkJs` chalkashligi.

Keyingi bo'limda TypeScript ning type system ining asosiy qurilish bloklari — **primitive type lar**, type annotation syntax, type inference, maxsus type lar (`any`, `unknown`, `never`, `void`), literal types va `as const` bilan tanishamiz.

---

**Keyingi bo'lim:** [02-primitive-types.md](02-primitive-types.md) — Primitive Types va Type Annotations: type annotation syntax, type inference, `any` vs `unknown` vs `never` vs `void`, literal types, `as const`, type assertions, type widening va narrowing.
