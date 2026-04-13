# Bo'lim 7: Functions TypeScript'da

> TypeScript'da funksiyalar — parametrlar, return type, overload'lar, call/construct signature'lar, `this` parameter, generic funksiyalar va callback type'lar orqali to'liq type safety ta'minlanadigan asosiy qurilish bloklari. TypeScript funksiya type'larini compile-time'da tekshiradi va JS'ga compile bo'lganda barcha type annotation'lar o'chiriladi — lekin qolgan runtime xatti-harakati (default qiymatlar, rest parameter'lar, `new`) saqlanadi.
>
> Bu bo'limda funksiya type'larining har bir qirrasini ko'ramiz: oddiy annotation'lardan variance qoidalarigacha. Ayniqsa callback type'lar, event handler'lar, va `this` binding — TypeScript bilan ishlashning kundalik muammolarini hal qiladi.

---

## Mundarija

- [Function Type Annotations](#function-type-annotations)
- [Return Type Annotation va Inference](#return-type-annotation-va-inference)
- [Optional Parameters](#optional-parameters)
- [Default Parameters](#default-parameters)
- [Rest Parameters](#rest-parameters)
- [Function Overloads](#function-overloads)
- [Call Signatures](#call-signatures)
- [Construct Signatures](#construct-signatures)
- [this Parameter](#this-parameter)
- [void vs undefined](#void-vs-undefined)
- [Function Type Expressions](#function-type-expressions)
- [Generic Functions — Kirish](#generic-functions--kirish)
- [Callback Types](#callback-types)
- [Function Assignability](#function-assignability)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Function Type Annotations

### Nazariya

TypeScript'da funksiya yozishda parametrlar va return type'ga **type annotation** qo'yish mumkin. Bu annotation'lar funksiyaning **contract**'ini — qanday qiymatlar qabul qilishi va nima qaytarishini aniq belgilaydi. TypeScript shu contract'ni compile-time'da tekshiradi — noto'g'ri type berilsa xato beradi.

Function declaration, function expression va arrow function — uchala shaklda ham annotation qo'yiladi:

```typescript
// Function declaration
function add(a: number, b: number): number {
  return a + b;
}

// Function expression
const multiply = function (a: number, b: number): number {
  return a * b;
};

// Arrow function
const divide = (a: number, b: number): number => {
  return a / b;
};
```

Parametr type annotation'i noto'g'ri argument berilishini oldini oladi va parametr sonini qat'iy tekshiradi — JavaScript'dan farqli (JS'da ortiqcha argument'lar ignore qilinadi, kam argument'lar `undefined` bo'ladi, lekin TypeScript aniq shu sondagi argument talab qiladi).

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori funksiya chaqiruvini ko'rganda, har bir argument'ning type'ini funksiya signaturasidagi parametr type bilan solishtiradi. Bu **subtype** tekshiruvi — argument type parametr type'ning subtype'i bo'lsa mos keladi.

```
greet("Ali") tekshiruvi:
  argument: "Ali" — type: string literal "Ali"
  parameter: name: string
  "Ali" extends string? → ✅ mos

greet(42) tekshiruvi:
  argument: 42 — type: number literal 42
  parameter: name: string
  42 extends string? → ❌ xato
```

Kompilator har bir funksiya deklaratsiyasi uchun **`Signature`** object yaratadi — unda parametr type'lari, return type, `this` type, minimum required parametr soni, va maksimal parametr soni saqlanadi. Funksiya chaqiruvini tekshirishda kompilator `getResolvedSignature()` funksiyasi orqali mos keluvchi signature'ni topadi va argument'larni unga moslashtiradi.

**Parametr type annotation qo'yilmagan holat:** Agar parametr type'siz yozilsa, TypeScript odatda `any` beradi. Lekin `noImplicitAny` flag'i yoqilgan bo'lsa (yoki `strict: true`), kompilator xato beradi — har parametr uchun aniq type yoki contextual type (masalan, callback'da) kerak. Contextual type — funksiya tashqi contextdan type'ni "oladi", masalan `array.map((x) => x * 2)`'da `x` array element type'iga inferrans bo'ladi.

**Annotation vs inference:** Parametr type'lari uchun annotation kerak bo'lsa-da, return type uchun TypeScript ko'p hollarda inference qiladi. Shuning uchun `function add(a: number, b: number)` yozish ham mumkin — TypeScript return type `number` deb biladi. Lekin public API'larda explicit return type tavsiya etiladi (pastda batafsil).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy annotation
function greet(name: string): string {
  return `Hello, ${name}!`;
}

greet("Ali");   // ✅
// greet(42);   // ❌ Error: number not assignable to string
// greet();     // ❌ Error: Expected 1 argument, got 0

// 2. Object parameter
interface User {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
}

function formatUserName(user: User): string {
  return `${user.firstName} ${user.lastName}`;
}

function getUserEmail(user: User): string {
  return user.email.toLowerCase();
}

const user: User = {
  id: 1,
  firstName: "Ali",
  lastName: "Valiyev",
  email: "ALI@MAIL.COM",
};

formatUserName(user);  // "Ali Valiyev"
getUserEmail(user);    // "ali@mail.com"

// 3. Arrow function annotation
const sum = (a: number, b: number): number => a + b;

// 4. Annotation qisqartirilmasligi kerak joylar
function calculateTax(amount: number, rate: number): number {
  return amount * rate;
}

// calculateTax("100", "0.2"); // ❌ string emas number
// calculateTax(100);          // ❌ 2 ta argument kerak

// 5. Barcha argumentlar bilan aniq chaqiriq
calculateTax(100, 0.2); // ✅ 20

// 6. Nested funksiya annotation
function outer(x: number): (y: number) => number {
  return function inner(y: number): number {
    return x + y;
  };
}

const addFive = outer(5);
addFive(10); // 15
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function add(a: number, b: number): number {
  return a + b;
}

const multiply = (a: number, b: number): number => a * b;
```

```javascript
// Compiled JS (ES2016+ target) — barcha type annotation'lar o'chiriladi
function add(a, b) {
  return a + b;
}

const multiply = (a, b) => a * b;
```

Type annotation'lar — **sof compile-time** konstruksiya. Runtime'da hech qanday iz qolmaydi. Funksiya JavaScript'dagi oddiy funksiya — `typeof add === "function"`.

</details>

---

## Return Type Annotation va Inference

### Nazariya

TypeScript funksiyaning return type'ini **ikki usulda** aniqlaydi:

1. **Explicit annotation** — dasturchi o'zi yozadi: `function getUser(): string`
2. **Type inference** — TypeScript `return` statement'larini tahlil qilib, type'ni o'zi aniqlaydi

TypeScript inference mexanizmi kuchli — ko'p hollarda return type yozish shart emas. Lekin ba'zi holatlarda explicit annotation yozish **tavsiya etiladi** yoki **majburiy**:

| Holat | Yozish kerakmi? | Sabab |
|-------|-----------------|-------|
| Public API / library | ✅ Tavsiya | Consumer'lar uchun aniq contract, refactor xavfsizligi |
| Recursive funksiya | ✅ Majburiy (ba'zan) | TS o'zi aniqlay olmaydi, infinite loop xavfi |
| Murakkab return | ✅ Tavsiya | Kutilmagan type inference'ning oldini olish |
| Private / internal | ❌ Shart emas | Inference yetarli, kod toza qoladi |
| Oddiy arrow funksiya | ❌ Shart emas | `const sum = (a: number, b: number) => a + b` — aniq |

```typescript
// TS inference — return type yozilmagan, TS o'zi aniqlaydi
function createUser(name: string, age: number) {
  return { name, age, createdAt: new Date() };
  // TS infers: { name: string; age: number; createdAt: Date }
}

// Explicit annotation — library / public API uchun
function parseJSON(text: string): unknown {
  return JSON.parse(text);
  // Explicit "unknown" — consumer'ni type check qilishga majburlaydi
  // Agar "any" deb inference qildirsak — xavfsizlik yo'qoladi
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori funksiya body'dagi **barcha `return` statement'larini** yig'adi va ularning type'larini **union** qiladi. Agar hech qanday `return` yo'q bo'lsa — return type `void`.

```
function getValue(condition: boolean) {
  if (condition) return "hello"; // return type: "hello" (literal) → string
  return 42;                      // return type: 42 (literal) → number
}

Inference: string | number
```

Literal type'lar odatda `return` context'ida kengaytiriladi (widen) — `"hello"` → `string`, `42` → `number`. Bu `let` assignment'dagi widening bilan bir xil qoida — kompilator kengroq type tanlaydi, chunki boshqa holat'larda ham shu funksiya ishlatilishi mumkin.

**Recursive funksiyalar:** TypeScript rekursiv funksiyalarda return type'ni aniqlash uchun "type seed" (tiyin) kerak — funksiya o'zini chaqirsa, kompilator hali o'zini bilmaydi. Bu holatda explicit return type yozish kerak:

```typescript
// ❌ TS inference'ni aniqlay olmaydi
// function factorial(n: number) {
//   if (n <= 1) return 1;
//   return n * factorial(n - 1);  // factorial return type noma'lum
// }

// ✅ Explicit return type
function factorial(n: number): number {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}
```

**`isolatedDeclarations` (TS 5.5+).** TypeScript 5.5 da yangi `isolatedDeclarations` option qo'shildi — har bir faylni **alohida** declaration file (`.d.ts`) ga compile qilish imkonini beradi. Bu mode'da **export qilingan** funksiyalarda return type **majburiy**:

```typescript
// ❌ isolatedDeclarations mode'da xato
export function createUser(name: string) {
  return { name, createdAt: new Date() };
}

// ✅ Explicit return type
export function createUser(name: string): { name: string; createdAt: Date } {
  return { name, createdAt: new Date() };
}
```

Sabab: `isolatedDeclarations` rejimida TypeScript har bir faylni boshqa fayllardan **mustaqil** compile qilishi kerak. Return type inference uchun funksiya body'ni tahlil qilish kerak — lekin bu butun type checker'ni ishga tushirishni talab qiladi va parallel build'larda imkonsiz. Explicit return type bo'lsa — body'ni tahlil qilish shart emas, faqat signature'dan declaration file yaratiladi. Bu feature Deno, Bun, esbuild kabi tez build tool'lar bilan yaxshi ishlash uchun.

**Contextual return type:** Agar funksiya contextual type'da ishlatilsa (masalan, callback sifatida), TypeScript return type'ni tashqi context'dan ham oladi:

```typescript
type Mapper = (x: number) => string;

// TS contextual type'dan biladi: return type string bo'lishi kerak
const double: Mapper = (x) => x * 2 + ""; // OK, string qaytaradi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Return type inference
function createPoint(x: number, y: number) {
  return { x, y };
}
// TS infers: { x: number; y: number }

const p = createPoint(1, 2);
p.x; // number
p.y; // number

// 2. Union return type inference
function parse(input: string) {
  if (input === "") return null;
  if (input === "true") return true;
  if (input === "false") return false;
  return Number(input);
}
// TS infers: number | boolean | null

// 3. Explicit return type — public API
export function hashPassword(password: string): Promise<string> {
  return crypto.subtle
    .digest("SHA-256", new TextEncoder().encode(password))
    .then(buf => Array.from(new Uint8Array(buf))
      .map(b => b.toString(16).padStart(2, "0"))
      .join(""));
}

// 4. Recursive funksiya — explicit return type majburiy
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// 5. Mutual recursion
function isEven(n: number): boolean {
  if (n === 0) return true;
  return isOdd(n - 1);
}

function isOdd(n: number): boolean {
  if (n === 0) return false;
  return isEven(n - 1);
}

// 6. Never return type — funksiya hech qachon qaytarmaydi
function fail(message: string): never {
  throw new Error(message);
}

// 7. Contextual return type
type Validator = (value: unknown) => boolean;

const isNonEmpty: Validator = (value) => {
  // return type contextual: boolean
  return typeof value === "string" && value.length > 0;
};

// 8. Void return type — hech narsa qaytarmaydi
function log(message: string): void {
  console.log(message);
  // return undefined implicit
}
```

</details>

---

## Optional Parameters

### Nazariya

Optional parameter — `?` belgisi bilan e'lon qilingan, **berilishi shart bo'lmagan** parametr. Optional parameter'ning type'iga TypeScript avtomatik `| undefined` qo'shadi. Optional parametrlar doim **oxirida** bo'lishi kerak — required parametrdan keyin.

```typescript
function greet(name: string, greeting?: string): string {
  // greeting: string | undefined
  const g = greeting ?? "Hello"; // nullish coalescing bilan default
  return `${g}, ${name}!`;
}

greet("Ali");            // "Hello, Ali!" — greeting berilmadi (undefined)
greet("Ali", "Salom");   // "Salom, Ali!" — greeting berildi
// greet("Ali", 42);     // ❌ Error: number not assignable to string | undefined
```

**Optional (`?`) va explicit `| undefined` farqi.** TypeScript'da ikkita o'xshash lekin farqli pattern:

- `greeting?: string` — **parameter berilmasligi** mumkin, yoki `undefined` berilishi mumkin
- `greeting: string | undefined` — parameter **har doim berilishi shart**, lekin qiymat `undefined` bo'lishi mumkin

```typescript
function optional(x?: string) { }         // x?: string
function explicit(x: string | undefined) { }

optional();           // ✅ x berilmadi
optional(undefined);  // ✅ x = undefined
explicit();           // ❌ Error: Expected 1 argument
explicit(undefined);  // ✅ x = undefined
```

Bu farq `exactOptionalPropertyTypes` flag'i (TS 4.4+) bilan yanada kuchayadi. Bu option `strict` ostida **avtomatik yoqilmaydi** — alohida yoqilishi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori optional parameter'ni ko'rganda, funksiya chaqiruvida argument sonini quyidagicha tekshiradi:

```
function createOrder(product: string, quantity?: number, urgent?: boolean)

Minimal arguments: 1 (faqat required parametrlar soni)
Maximal arguments: 3 (barcha parametrlar soni)

createOrder("laptop")              → 1 argument, 1 ≤ 1 ≤ 3 ✅
createOrder("laptop", 5)           → 2 arguments, 1 ≤ 2 ≤ 3 ✅
createOrder("laptop", 5, true)     → 3 arguments, 1 ≤ 3 ≤ 3 ✅
createOrder()                      → 0 arguments, 0 < 1 ❌
createOrder("a", "b", "c", "d")    → 4 arguments, 4 > 3 ❌
```

**`Signature` object'da hisoblash:** Kompilator har funksiya uchun `Signature` object'da `minArgumentCount` va `parameters.length` (maksimal) saqlaydi. Optional parametr'lar `minArgumentCount`'ga qo'shilmaydi. Rest parameter bo'lsa, `maxArgumentCount` — cheksiz.

**`exactOptionalPropertyTypes` (TS 4.4+).** Bu flag optional property/parameter semantikasini o'zgartiradi. Oddiy holat'da `x?: string` = `string | undefined`. `exactOptionalPropertyTypes: true` bo'lsa, "optional" va "undefined" alohida tushunchalar:

```typescript
// exactOptionalPropertyTypes: true
interface Config {
  port?: number; // "port yo'q" yoki "port: number"
                 // LEKIN port: undefined RUXSAT ETILMAYDI
}

const c1: Config = {};             // ✅ port yo'q
const c2: Config = { port: 3000 }; // ✅ port berildi
// const c3: Config = { port: undefined }; // ❌ Error
```

**Muhim:** Bu flag `strict: true` ostida **avtomatik yoqilmaydi** — alohida `"exactOptionalPropertyTypes": true` yoki command-line flag kerak. Sabab: bu o'zgarish mavjud kod'ni buzishi mumkin (migration cost), shuning uchun opt-in qilib qoldirilgan.

**Optional parametrlar tuple bilan:** Rest parameter uchun tuple type'da optional element ishlatilishi mumkin:

```typescript
function fn(...args: [string, number?]): void { }
fn("a");        // ✅
fn("a", 42);    // ✅
fn("a", 42, 3); // ❌
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Asosiy optional parameter
function createUrl(base: string, path?: string): string {
  return path ? `${base}/${path}` : base;
}

createUrl("https://api.example.com");          // "https://api.example.com"
createUrl("https://api.example.com", "users"); // "https://api.example.com/users"

// 2. Ko'plab optional parametrlar
function createOrder(
  productId: number,
  quantity?: number,
  urgent?: boolean,
  notes?: string
): void {
  console.log({
    productId,
    quantity: quantity ?? 1,
    urgent: urgent ?? false,
    notes: notes ?? "",
  });
}

createOrder(1);                      // faqat product
createOrder(1, 5);                   // quantity qo'shildi
createOrder(1, 5, true);             // urgent qo'shildi
createOrder(1, 5, true, "rush");     // hammasi berildi

// 3. Optional vs explicit undefined
function optional(name?: string): void { }
function explicit(name: string | undefined): void { }

optional();            // ✅ argument berilmadi
optional(undefined);   // ✅ explicit undefined
// explicit();         // ❌ Error: 1 argument kerak
explicit(undefined);   // ✅ explicit undefined

// 4. Nullish coalescing bilan default
function fetchData(url: string, timeout?: number): void {
  const t = timeout ?? 5000; // default 5000
  console.log(`Fetching ${url} with timeout ${t}`);
}

// 5. Optional destructuring
interface Options {
  retries?: number;
  timeout?: number;
  cache?: boolean;
}

function request(url: string, options?: Options): void {
  const { retries = 3, timeout = 5000, cache = false } = options ?? {};
  console.log({ url, retries, timeout, cache });
}

request("/api/data");
request("/api/data", { retries: 5 });

// 6. ⚠️ Optional parameter division'da
function unsafeDivide(a: number, b?: number): number {
  return a / b!; // ❌ b undefined bo'lsa — NaN
}

// ✅ Default parameter bilan xavfsiz
function safeDivide(a: number, b: number = 1): number {
  return a / b;
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}!`;
}
```

```javascript
// Compiled JS — type annotation va ? belgisi o'chiriladi
function greet(name, greeting) {
  return `${greeting ?? "Hello"}, ${name}!`;
}
// JS da "optional parameter" tushunchasi yo'q —
// berilmagan argument har doim undefined bo'ladi
```

Optional parametrlar runtime'da "mavjud emas" — JS har qanday funksiyaga istalgan sondagi argument berish mumkin. Berilmagan parametrlar `undefined` bo'ladi.

</details>

---

## Default Parameters

### Nazariya

Default parameter — qiymat berilmasa ishlatadigan **standart qiymat** bilan e'lon qilingan parametr. TypeScript default qiymatdan type'ni **infer** qiladi — shuning uchun type annotation ko'pincha kerak emas.

```typescript
// Default parameter — type annotation kerak emas
function createPagination(page = 1, limit = 20): { page: number; limit: number } {
  return { page, limit };
}
// TS infers: page: number, limit: number (default qiymatlardan)

createPagination();        // { page: 1, limit: 20 }
createPagination(3);       // { page: 3, limit: 20 }
createPagination(3, 50);   // { page: 3, limit: 50 }

// undefined berilsa — default qiymat ishlatiladi
createPagination(undefined, 50); // { page: 1, limit: 50 }
```

**Default vs Optional:**

| | Optional (`?`) | Default (`= value`) |
|---|---|---|
| Type | `T \| undefined` | `T` (widened) |
| Berilmasa | `undefined` | default qiymat |
| `undefined` berilsa | `undefined` | default qiymat |
| Joylashuvi | Faqat oxirida | Istalgan joyda (lekin tavsiya emas) |

Default parameter o'rtada ham bo'lishi mumkin, lekin chaqirayotgan joyda uni "skip" qilish uchun `undefined` berish kerak — bu pattern'ning foydasi kam va tavsiya etilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori default parameter'ni ko'rganda ikki narsa sodir bo'ladi: **type inference** va **compiled output**'da default qiymatni saqlash.

**Type inference:** `page = 1` ko'rilganda, kompilator `1` ning type'ini aniqlab, `page` ga `number` type beradi. Bu **widening** — literal type `1` emas, balki `number` type assign bo'ladi. `let` va `var` deklaratsiyasi bilan bir xil widening qoidasi:

```
function createPagination(page = 1, limit = 20)

Compiler inference:
  page = 1    → typeof 1 → widen(1) → number → page: number
  limit = 20  → typeof 20 → widen(20) → number → limit: number

Call signature: (page?: number, limit?: number) => { page: number; limit: number }
```

Agar aniq literal type kerak bo'lsa, explicit annotation yoki `as const` ishlatish mumkin:

```typescript
function withLiteral(mode: "dev" | "prod" = "dev"): void { }
// mode: "dev" | "prod" — literal saqlanadi, chunki explicit type bor

function withAsConst(modes = ["a", "b", "c"] as const): void { }
// modes: readonly ["a", "b", "c"] — as const tufayli literal tuple
```

**Compiled output'da default value:** Bu — type annotation emas, runtime feature. Shuning uchun kompilator target ga qarab turlicha emit qiladi:

- **ES2015+ (ES6+):** `function f(x = 10)` — JS syntax'i to'g'ridan-to'g'ri qoldiriladi (native JS default parameter).
- **ES5:** `function f(x)` + body boshida `if (x === void 0) { x = 10; }` pattern.

**Default value evaluation:** Default qiymat **har chaqiriqda qayta evaluate qilinadi**:

```typescript
function appendId(list: number[] = []): number[] {
  list.push(Date.now());
  return list;
}

const a = appendId(); // [1700000000000]
const b = appendId(); // [1700000000001] — YANGI array
// a !== b — har chaqiriqda yangi [] yaratiladi
```

Bu Python'dan farqli — Python'da `def f(x=[])` default qiymat faqat bir marta yaratiladi (va bu klassik bug). JavaScript/TypeScript'da bu xavf yo'q — har chaqiriq'da yangi evaluation.

**Type compatibility check:** TypeScript default qiymatning type annotation bilan mos kelishini verify qiladi. `function f(x: string = 42)` — `42` ning type'i `number`, `string`'ga assign bo'lmaydi → compile-time xato.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy default
function greet(name: string, greeting: string = "Hello"): string {
  return `${greeting}, ${name}!`;
}

greet("Ali");              // "Hello, Ali!"
greet("Ali", "Salom");     // "Salom, Ali!"
greet("Ali", undefined);   // "Hello, Ali!" — undefined → default

// 2. Default bilan type inference
function retry(attempts = 3, delay = 1000): void {
  console.log(`Retrying ${attempts} times with ${delay}ms delay`);
}
// TS infers: attempts: number, delay: number

retry();        // 3, 1000
retry(5);       // 5, 1000
retry(5, 2000); // 5, 2000

// 3. Default object (har chaqiriqda yangi object)
interface Config {
  verbose: boolean;
  maxSize: number;
}

function process(items: number[], config: Config = { verbose: false, maxSize: 100 }): void {
  // config har chaqiriqda yangi object
  console.log(items.length, config);
}

process([1, 2, 3]);
process([1, 2, 3], { verbose: true, maxSize: 50 });

// 4. Default expression — evaluated har chaqiriqda
function createId(prefix: string, timestamp: number = Date.now()): string {
  return `${prefix}-${timestamp}`;
}

const id1 = createId("user"); // har xil timestamp
const id2 = createId("user"); // id1 dan keyin chaqirilgan — kattaroq timestamp

// 5. Oldingi parameter'dan foydalanuvchi default
function multiply(a: number, b: number = a): number {
  return a * b;
}

multiply(5);    // 25 (b = a = 5)
multiply(5, 3); // 15

// 6. Destructured default
function createUser({
  name,
  age = 18,
  role = "user",
}: {
  name: string;
  age?: number;
  role?: "admin" | "user";
}): void {
  console.log({ name, age, role });
}

createUser({ name: "Ali" });
createUser({ name: "Ali", age: 25 });
createUser({ name: "Ali", role: "admin" });

// 7. Default bilan complex type
type Logger = (msg: string) => void;

function createService(log: Logger = console.log): void {
  log("Service started");
}

createService();                      // console.log ishlatiladi
createService((m) => console.error(m)); // custom logger
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function greet(name: string, greeting: string = "Hello"): string {
  return `${greeting}, ${name}!`;
}
```

```javascript
// Compiled JS (ES2015+ target) — native default parameter saqlanadi
function greet(name, greeting = "Hello") {
  return `${greeting}, ${name}!`;
}
```

```javascript
// Compiled JS (ES5 target) — default parameter emulate qilinadi
function greet(name, greeting) {
  if (greeting === void 0) { greeting = "Hello"; }
  return greeting + ", " + name + "!";
}
```

ES2015+ target'da (zamonaviy browser'lar, Node.js 6+) JavaScript native default parameter'ni qo'llab-quvvatlaydi — TypeScript hech narsa qo'shmaydi. ES5 target'da esa `void 0` pattern bilan emulate qilinadi (`void 0` — `undefined`'ning xavfsiz qisqa shakli, chunki ba'zi eski JS engine'larda `undefined` global qayta yozilishi mumkin edi).

</details>

---

## Rest Parameters

### Nazariya

Rest parameter (`...`) — noaniq sondagi argumentlarni **array** sifatida qabul qiladi. TypeScript'da rest parameter type annotation'i **array type** bo'lishi kerak: `T[]`, `Array<T>` yoki tuple.

Rest parameter doim **eng oxirgi** parametr bo'lishi kerak — undan keyin parameter bo'lmaydi.

```typescript
// Oddiy rest parameter — number[]
function sum(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3);     // 6
sum(10, 20);      // 30
sum();            // 0 (bo'sh array)
// sum(1, "2", 3); // ❌ Error: string !== number

// Required + rest
function log(level: "info" | "warn" | "error", ...messages: string[]): void {
  console.log(`[${level.toUpperCase()}]`, ...messages);
}

log("info", "Server started");
log("error", "Failed to connect", "retrying...");
```

**Tuple rest parameter** — aniq parametr ketma-ketligini ta'minlaydi:

```typescript
function createEvent(...args: [name: string, timestamp: number, data?: unknown]): void {
  const [name, timestamp, data] = args;
  console.log(`Event: ${name} at ${timestamp}`, data);
}

createEvent("click", Date.now());             // data optional
createEvent("submit", Date.now(), { id: 1 });  // data berildi
// createEvent("click");                       // ❌ timestamp kerak
```

Tuple rest — **named tuple element'lar** (TS 4.0+) — har element'ga nom berish mumkin, bu developer experience'ni yaxshilaydi va IDE'da parameter hint'lari'da ko'rinadi.

**Variadic tuple types** (TS 4.0+) — rest parameter'ning eng kuchli forma:

```typescript
// Birinchi argument string, qolgani number
function withFirstString(...args: [string, ...number[]]): void { }

withFirstString("label", 1, 2, 3); // ✅
// withFirstString(1, 2, 3);        // ❌ birinchi string bo'lishi kerak
```

<details>
<summary><strong>Under the Hood</strong></summary>

Rest parameter — **runtime feature**, compile-time annotation emas. TypeScript kompilatori rest parameter'ni tuple yoki array type bilan bog'laydi va chaqiruv argument'larini tekshiradi.

```
function log(level: string, ...messages: string[])

Compiler internal:
  parameters: [level: string]
  rest parameter: messages: string[]
  minArgumentCount: 1
  maxArgumentCount: Infinity

log("info")                → 1 argument, ≥ 1 ✅
log("info", "a", "b")      → 3 arguments, ≥ 1 ✅
log()                      → 0 arguments, < 1 ❌
```

**Spread argument bilan tekshirish:** Spread argument (`func(...arr)`) ishlatilganda, kompilator array'ning **uzunligini bilishi kerak** — aks holda aniq argument count tekshiruvini qila olmaydi:

```
function add(a: number, b: number): number
const nums = [1, 2];        // type: number[] (uzunlik noma'lum)
const pair = [1, 2] as const; // type: readonly [1, 2] (uzunlik aniq)

add(...nums);  // ❌ Error: number[] uzunligi noma'lum
add(...pair);  // ✅ uzunlik 2, parameter count'ga mos
```

**Compiled output:** Rest parameter — ES2015+ (ES6+) native syntax, TypeScript uni to'g'ridan-to'g'ri qoldiradi:

```
ES2015+ output:
  function log(level, ...messages) { ... }

ES5 output (eski target):
  function log(level) {
    var messages = [];
    for (var _i = 1; _i < arguments.length; _i++) {
      messages[_i - 1] = arguments[_i];
    }
    ...
  }
```

ES5 target'da TypeScript `arguments` object'dan rest array'ni manually yaratadi (boshlang'ich indeks `1` — birinchi required parameter'dan keyingisidan).

**Variadic tuple types (TS 4.0+).** Bu feature tuple'larda ham rest element'ni qo'llab-quvvatlaydi:

```
type Strings = [string, string];
type Numbers = [number, number];
type Combined = [...Strings, ...Numbers];
// Combined = [string, string, number, number]
```

Bu pattern generic funksiyalarda kuchli — `concat`, `bind`, function composition kabi use case'lar uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy rest parameter
function max(...numbers: number[]): number {
  if (numbers.length === 0) throw new Error("No numbers");
  return numbers.reduce((a, b) => (a > b ? a : b));
}

max(1, 5, 3, 2);    // 5
max(42);            // 42

// 2. Mixed parameter turlari
function buildQuery(table: string, operation: "select" | "insert", ...columns: string[]): string {
  return `${operation.toUpperCase()} ${columns.join(", ")} FROM ${table}`;
}

buildQuery("users", "select", "id", "name", "email");
// "SELECT id, name, email FROM users"

// 3. Named tuple rest — parameter hint'lari yaxshilanadi
function makeRequest(...args: [method: "GET" | "POST", url: string, body?: unknown]): void {
  const [method, url, body] = args;
  console.log(method, url, body);
}

makeRequest("GET", "/api/users");
makeRequest("POST", "/api/users", { name: "Ali" });

// 4. Spread argument with tuple
function draw(x: number, y: number, color: string): void {
  console.log(`Drawing at (${x}, ${y}) in ${color}`);
}

const point = [10, 20, "red"] as const;
draw(...point); // ✅ tuple uzunligi 3

const dynamicPoint: number[] = [10, 20];
// draw(...dynamicPoint, "red"); // ❌ spread array

// 5. Variadic tuple types — kuchli pattern
function prepend<T, U extends unknown[]>(first: T, ...rest: U): [T, ...U] {
  return [first, ...rest];
}

const result = prepend("label", 1, 2, 3);
// result: [string, number, number, number]

// 6. Rest parameter bilan generic
function tuple<T extends unknown[]>(...args: T): T {
  return args;
}

const t1 = tuple(1, "a", true); // [number, string, boolean]
const t2 = tuple(10, 20);        // [number, number]

// 7. Function composition with variadic
function pipe<T>(...fns: Array<(arg: T) => T>): (value: T) => T {
  return (value) => fns.reduce((acc, fn) => fn(acc), value);
}

const transform = pipe<number>(
  (n) => n + 1,
  (n) => n * 2,
  (n) => n - 3,
);
transform(5); // ((5 + 1) * 2) - 3 = 9
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function log(level: "info" | "error", ...messages: string[]): void {
  console.log(`[${level}]`, ...messages);
}
```

```javascript
// Compiled JS (ES2015+ target) — native rest parameter
function log(level, ...messages) {
  console.log(`[${level}]`, ...messages);
}
```

Zamonaviy target'larda rest va spread ikkalasi ham native JavaScript feature — TypeScript o'zgarishsiz qoldiradi. Faqat ES5 yoki undan past target'larda `arguments` object'dan manually array yaratish pattern'i ishlatiladi.

</details>

---

## Function Overloads

### Nazariya

Function overload — bitta funksiyaga **bir nechta signature** yozish. Har bir overload signature funksiyaning qanday parametrlar bilan qanday type qaytarishini belgilaydi. Oxirida **implementation signature** yoziladi — bu haqiqiy funksiya body'si.

Overload uch qismdan iborat:

1. **Overload signature'lar** — tashqi dunyo ko'radigan signature'lar (1 dan ko'p)
2. **Implementation signature** — haqiqiy body yozilgan signature (faqat 1 ta)
3. **Implementation body** — funksiya kodi

**Muhim:** Implementation signature'ni tashqi dunyo **ko'rmaydi** — faqat overload signature'lar ko'rinadi.

```typescript
// Overload signature'lar — tashqi dunyo shu variant'larni ko'radi
function createElement(tag: "img"): HTMLImageElement;
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: "div"): HTMLDivElement;

// Implementation signature — barchani qamrab olishi kerak
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const img = createElement("img");     // return type: HTMLImageElement
const input = createElement("input"); // return type: HTMLInputElement
const div = createElement("div");     // return type: HTMLDivElement
// const p = createElement("p");      // ❌ Error: "p" overload'larda yo'q
```

**Overload resolution order:** TypeScript overload'larni **yuqoridan pastga** tekshiradi va birinchi mos kelganini tanlaydi. Shuning uchun **aniqroq** overload'lar birinchi yozilishi kerak.

**Overload vs Union Type — qachon qaysi biri:**

| Holat | Overload | Union |
|-------|----------|-------|
| Return type parametrga bog'liq | ✅ | ❌ |
| Parametr kombinatsiyalari farq qiladi | ✅ | ❌ |
| Barcha overload'lar bir xil return type | ❌ | ✅ |
| Oddiy parameter type farqi | ❌ | ✅ |

Overload'ni faqat return type parametrga qarab **o'zgarganda** ishlatish kerak — aks holda union type yetarli va soddaroq.

<details>
<summary><strong>Under the Hood</strong></summary>

Overload resolution TypeScript'ning `checker.ts`'dagi `chooseOverload()` va `getResolvedSignature()` funksiyalari orqali amalga oshiriladi. Algoritm:

```
Overload Resolution Algorithm:

1. Funksiya chaqiruvini topish
2. Kandidat signature'larni yig'ish (barcha overload'lar, implementation emas)
3. Har bir signature uchun parallel:
   a. Argument count'ni tekshir (min ≤ N ≤ max)
   b. Har bir argument'ni parameter type'ga assignability uchun tekshir
   c. Agar barcha argument'lar mos kelsa → "candidate"
4. Birinchi candidate'ni tanlash (yuqoridan pastga)
5. Agar hech bir candidate yo'q → error: "No overload matches this call"
```

**Shuning uchun tartib muhim.** TypeScript **birinchi mos keluvchini** tanlaydi, eng aniqni emas. Masalan:

```typescript
// ❌ Noto'g'ri tartib — keng overload birinchi
function processWrong(value: string | number): string[] | number;
function processWrong(value: string): string[];           // Hech qachon match bo'lmaydi
function processWrong(value: string | number): string[] | number {
  return typeof value === "string" ? value.split(",") : value * 2;
}

const result = processWrong("hello");
// result type: string[] | number — chunki birinchi overload (keng) match bo'ldi
// "string[]" overload'iga yetib kelmadi
```

```typescript
// ✅ To'g'ri tartib — aniq birinchi
function processRight(value: string): string[];
function processRight(value: number): number;
function processRight(value: string | number): string[] | number {
  return typeof value === "string" ? value.split(",") : value * 2;
}

const result = processRight("hello");
// result type: string[] ✅
```

**Implementation signature tashqi dunyodan yashirin:** TypeScript implementation signature'ni `getSignaturesOfType()`'dan chiqarib tashlaydi. Bu yashirinlik sabab — implementation signature ko'pincha ancha keng (`string | number`) bo'ladi, lekin caller'lar har doim bitta aniq variant bilan chaqiradi. Agar implementation ham "visible" bo'lsa, overload'lar keraksiz bo'lib qolardi.

**Overload vs generic function:** Ba'zi holatlarda overload o'rniga generic funksiya yaxshiroq. Masalan, `identity<T>(x: T): T` — har qanday type uchun ishlaydi. Overload faqat **dispatch** kerak bo'lganda (turli parameter turlari turli return type beradi) qo'llaniladi.

**Implementation signature caller'lar uchun ko'rinmasligi:** Agar 2 ta overload + implementation yozsangiz, caller'ga faqat 2 ta variant ko'rinadi. Implementation signature'dagi keng type (`string | number | boolean`) caller uchun emas — faqat body'ni yozish uchun kompilatorga yordam uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Return type parametrga bog'liq
function parseInput(input: string): string[];
function parseInput(input: number): number[];
function parseInput(input: string | number): string[] | number[] {
  if (typeof input === "string") return input.split(",");
  return String(input).split("").map(Number);
}

const words = parseInput("a,b,c");  // string[]
const digits = parseInput(12345);    // number[]

// 2. Resolution order matters
function format(value: string): string;
function format(value: number): string;
function format(value: Date): string;
function format(value: string | number | Date): string {
  if (value instanceof Date) return value.toISOString();
  return String(value);
}

format("hello");           // matches overload 1
format(42);                // matches overload 2
format(new Date());        // matches overload 3

// 3. Optional parameter + overload
function greet(name: string): string;
function greet(name: string, title: string): string;
function greet(name: string, title?: string): string {
  return title ? `${title} ${name}` : name;
}

greet("Ali");              // matches overload 1
greet("Ali", "Dr.");       // matches overload 2

// 4. DOM-style overload
function query(selector: "img"): HTMLImageElement | null;
function query(selector: "input"): HTMLInputElement | null;
function query(selector: "div"): HTMLDivElement | null;
function query(selector: string): HTMLElement | null {
  return document.querySelector(selector);
}

const img = query("img");    // HTMLImageElement | null
const input = query("input"); // HTMLInputElement | null

// 5. Configuration-based overload
function parseConfig(source: string): Record<string, string>;
function parseConfig(source: string, raw: true): string;
function parseConfig(source: string, raw?: boolean): Record<string, string> | string {
  if (raw) return source;
  const result: Record<string, string> = {};
  source.split("\n").forEach(line => {
    const [key, value] = line.split("=");
    if (key && value) result[key.trim()] = value.trim();
  });
  return result;
}

const parsed = parseConfig("key=value\nfoo=bar");   // Record<string, string>
const raw = parseConfig("key=value\nfoo=bar", true); // string

// 6. ❌ Keraksiz overload — union yetarli
function toStringBad(value: string): string;
function toStringBad(value: number): string;
function toStringBad(value: boolean): string;
function toStringBad(value: string | number | boolean): string {
  return String(value);
}

// ✅ Union — sodda va toza
function toStringGood(value: string | number | boolean): string {
  return String(value);
}
```

</details>

---

## Call Signatures

### Nazariya

Call signature — object type ichida funksiyani ifodalash usuli. Oddiy funksiya type emas, **property'lari ham bor** object'da ishlatiladi. Bu pattern JavaScript'da ko'p uchraydi — funksiya object sifatida property'larga ega bo'ladigan holatlar. Bu pattern **hybrid type** deb ataladi — type ham funksiya, ham object.

Real-world misol'lar: jQuery `$` (funksiya va `$.ajax` method), lodash `_` (funksiya va `_.map` method), Node.js `require` (funksiya va `require.cache` property).

```typescript
// Object type ichida call signature
type Logger = {
  (message: string): void;         // Call signature — funksiya sifatida chaqirish
  level: "info" | "warn" | "error"; // Property
  history: string[];                // Property
};

// Interface bilan call signature
interface Formatter {
  (value: unknown): string;        // Call signature
  locale: string;                  // Property
}
```

Call signature ikkita syntax'da yoziladi — `type` alias yoki `interface`. Ikkalasi ham bir xil ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Call signature — **faqat type system**'da mavjud bo'lgan konstruksiya. Compiled JavaScript'da call signature'ning hech qanday izi qolmaydi — butunlay type erasure qurboni.

TypeScript kompilatori ichida call signature `CallSignatureDeclaration` AST node sifatida saqlanadi. Bu node funksiya type'ini object type ichida ifodalaydi. Kompilator object type'ni tekshirganda, call signature bor-yo'qligini ko'radi — agar bor bo'lsa, shu object'ni funksiya sifatida chaqirish mumkinligini belgilaydi.

```
type Logger = {
  (message: string): void;    // CallSignatureDeclaration
  level: string;               // PropertySignature
};

Compiler tekshiruvi:
  logger("hello") → Logger'da call signature bormi? → ✅ (message: string): void
  logger.level     → Logger'da 'level' property bormi? → ✅ string
  logger(42)       → call signature: 42 extends string? → ❌ Error
```

**Hybrid type** — bu TypeScript community'da qabul qilingan nom. Call signature + property'lar birgalikda type'ni "ham funksiya, ham object" qiladi. JavaScript runtime'da bunday object oddiy JS function — `typeof hybridObj === "function"` qaytaradi, chunki JS funksiyalari ham object'dir (ularga property assign qilish mumkin).

**Arrow function type vs call signature:** Ikkalasi ham kompilator ichida bir xil `Signature` object'ga resolve bo'ladi. Farq faqat syntax'da:

```typescript
// Arrow syntax — faqat funksiya
type Fn1 = (x: number) => string;

// Call signature — funksiya + property qo'shish mumkin
type Fn2 = {
  (x: number): string;
  description: string;
};
```

Agar hech qanday property qo'shilmasa, arrow syntax afzal — qisqa va o'qishga oson. Call signature faqat hybrid type (funksiya + property) kerak bo'lganda ishlatiladi.

**Hybrid type yaratish — `Object.assign` pattern.** JavaScript'da hybrid object yaratishning eng toza usuli:

```typescript
type Counter = {
  (): number;
  reset(): void;
  count: number;
};

function createCounter(): Counter {
  const fn = () => ++counter.count;
  const counter = Object.assign(fn, {
    count: 0,
    reset() { counter.count = 0; },
  });
  return counter;
}
```

`Object.assign` funksiya object'ga property'larni qo'shadi va natija'ni qaytaradi. TypeScript 3.0'dan boshlab `Object.assign` return type'i barcha qo'shilgan property'lar bilan hisoblanadi — shuning uchun hybrid type'ga ishonchli cast qilish mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy hybrid type
type Greeter = {
  (name: string): string;
  language: "en" | "uz" | "ru";
};

const greetUz: Greeter = Object.assign(
  (name: string) => `Salom, ${name}!`,
  { language: "uz" as const }
);

greetUz("Ali");      // "Salom, Ali!"
greetUz.language;    // "uz"

// 2. Counter hybrid
type Counter = {
  (): number;
  reset(): void;
  count: number;
};

function createCounter(): Counter {
  const counter = Object.assign(
    function (): number {
      counter.count += 1;
      return counter.count;
    },
    {
      count: 0,
      reset() { counter.count = 0; },
    }
  );
  return counter;
}

const counter = createCounter();
counter();       // 1
counter();       // 2
counter.count;   // 2
counter.reset();
counter();       // 1

// 3. Service locator hybrid
interface Service {
  (): string;         // service'ni chaqirish
  name: string;
  start(): void;
  stop(): void;
}

function createService(name: string): Service {
  let running = false;
  const service = Object.assign(
    function (): string {
      return `Service ${name}: ${running ? "running" : "stopped"}`;
    },
    {
      name,
      start() { running = true; },
      stop() { running = false; },
    }
  );
  return service;
}

const db = createService("Database");
db.start();
db();      // "Service Database: running"
db.stop();
db();      // "Service Database: stopped"

// 4. Multiple call signatures
interface Parser {
  (input: string): number;
  (input: string, radix: number): number;
  defaultRadix: number;
}

const parse: Parser = Object.assign(
  (input: string, radix?: number) => parseInt(input, radix ?? parse.defaultRadix),
  { defaultRadix: 10 }
);

parse("ff", 16);  // 255
parse("123");     // 123 (radix 10)
parse.defaultRadix = 2;
parse("101");     // 5 (radix 2)

// 5. Function type vs call signature
type Fn1 = (x: number) => string;           // faqat funksiya
type Fn2 = { (x: number): string; desc: string }; // funksiya + property

const a: Fn1 = (x) => String(x);
// a.desc = "..."; // ❌ Fn1 da property yo'q

const b: Fn2 = Object.assign((x: number) => String(x), { desc: "converter" });
b.desc; // ✅ "converter"
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type Logger = {
  (message: string): void;
  level: "info" | "warn" | "error";
};

const logger: Logger = Object.assign(
  (message: string) => console.log(message),
  { level: "info" as const }
);
```

```javascript
// Compiled JS — type alias va call signature butunlay o'chiriladi
const logger = Object.assign(
  (message) => console.log(message),
  { level: "info" }
);
```

Call signature — sof type system konstruksiyasi. Compiled JavaScript'da faqat `Object.assign` chaqiriq qoladi — bu JS runtime feature. TypeScript hech qanday kod qo'shmaydi.

</details>

---

## Construct Signatures

### Nazariya

Construct signature — `new` keyword bilan chaqiriladigan funksiya type'ini ifodalaydi. Class constructor'larni type sifatida ifodalashda ishlatiladi — masalan, factory pattern'da class'ni parameter sifatida qabul qilish uchun. Call signature'ga `new` keyword qo'shiladi.

```typescript
// Construct signature
type UserConstructor = {
  new (name: string, age: number): User;
};

interface User {
  name: string;
  age: number;
}

class UserImpl implements User {
  constructor(public name: string, public age: number) {}
}

function createUser(ctor: UserConstructor, name: string, age: number): User {
  return new ctor(name, age);
}

const user = createUser(UserImpl, "Ali", 25);
```

**Call + Construct signature kombinatsiyasi.** Ba'zi JS object'lari ham funksiya sifatida, ham `new` bilan chaqirilishi mumkin. Built-in misollar: `Date`, `Array`, `String`.

```typescript
interface MyDateType {
  new (): Date;                   // new Date() — Date object
  new (value: number): Date;      // new Date(timestamp)
  new (dateString: string): Date; // new Date("2024-01-01")
  (): string;                     // Date() — hozirgi vaqt string sifatida
}
```

Bu pattern JavaScript'ning legacy API'larida uchraydi. Zamonaviy kod'da kam ishlatiladi — lekin declaration file'larda ko'p uchraydi (`lib.es5.d.ts`'dagi `DateConstructor`).

<details>
<summary><strong>Under the Hood</strong></summary>

Construct signature — type system'da `new` bilan chaqiriladigan funksiyani ifodalash usuli. Compiled JavaScript'da construct signature **butunlay o'chiriladi** — faqat compile-time tekshiruv uchun mavjud.

TypeScript kompilatori ichida construct signature `ConstructSignatureDeclaration` node sifatida AST'da saqlanadi. `new ctor(args)` expression ko'rilganda, kompilator `ctor`'ning type'ida construct signature borligini tekshiradi. Agar yo'q bo'lsa — `This expression is not constructable` xatosi beradi.

```
type UserConstructor = {
  new (name: string, age: number): User;
};

Compiler tekshiruvi:
  new ctor("Ali", 25) →
    1. ctor'ning type'ida construct signature bormi? → ✅
    2. Argumentlar mos keladi: "Ali" extends string, 25 extends number → ✅
    3. Return type: User
```

JavaScript runtime'da `new` keyword — JS'ning oddiy constructor chaqiruv mexanizmi: prototype chain, `this` binding, va object creation. TypeScript'ning construct signature bu mexanizmga **hech narsa qo'shmaydi** — faqat compile-time'da argument type'lari va return type to'g'ri ekanligini kafolatlaydi.

**Class'lar avtomatik construct signature'ga ega:** Kompilator class'ning `constructor` method'idan construct signature'ni hosil qiladi. Shuning uchun `new ClassName()` chaqiruvi type-safe bo'ladi. Class nomi ham type sifatida, ham value (constructor funksiya) sifatida ishlatiladi:

```typescript
class Point {
  constructor(public x: number, public y: number) {}
}

// Type sifatida
const p: Point = new Point(1, 2);

// Value sifatida (construct signature)
const PointCtor: typeof Point = Point;
const p2 = new PointCtor(3, 4);
```

**Abstract class va construct signature:** Abstract class'ni `new` bilan chaqirish taqiqlanadi — kompilator construct signature'ni "abstract" deb belgilaydi va `new AbstractClass()` xato beradi. Lekin abstract class concrete subclass'lar uchun constructor signature'ga ega — factory pattern'da ishlatish mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy construct signature
type PersonCtor = {
  new (name: string): Person;
};

class Person {
  constructor(public name: string) {}
  greet(): string { return `Hi, I'm ${this.name}`; }
}

function createPerson(Ctor: PersonCtor, name: string): Person {
  return new Ctor(name);
}

const ali = createPerson(Person, "Ali");
ali.greet(); // "Hi, I'm Ali"

// 2. Factory pattern with construct signature
interface Animal {
  name: string;
  sound(): string;
}

interface AnimalCtor {
  new (name: string): Animal;
}

class Dog implements Animal {
  constructor(public name: string) {}
  sound(): string { return "Woof!"; }
}

class Cat implements Animal {
  constructor(public name: string) {}
  sound(): string { return "Meow!"; }
}

function animalFactory(Ctor: AnimalCtor, name: string): Animal {
  return new Ctor(name);
}

const rex = animalFactory(Dog, "Rex");
const whiskers = animalFactory(Cat, "Whiskers");
rex.sound();       // "Woof!"
whiskers.sound();  // "Meow!"

// 3. Generic constructor
interface Constructor<T> {
  new (...args: any[]): T;
}

function createInstance<T>(Ctor: Constructor<T>, ...args: any[]): T {
  return new Ctor(...args);
}

const point = createInstance(Point, 5, 10);
const dog = createInstance(Dog, "Buddy");

// 4. typeof bilan construct signature (eng ko'p ishlatiladigan usul)
function registerClass<T>(Ctor: new (...args: any[]) => T): void {
  console.log(`Registered: ${Ctor.name}`);
}

registerClass(Person);
registerClass(Dog);

// 5. Call + Construct (like Date)
interface MyDateType {
  new (): Date;                   // new bilan
  new (timestamp: number): Date;  // new bilan timestamp
  (): string;                     // new siz — hozirgi vaqt string
}

// Foydalanish misolida — type cast bilan
const DateLike = Date as unknown as MyDateType;
const now = new DateLike();       // Date object
const nowStr = DateLike();         // string

// 6. Abstract class construct
abstract class Shape {
  abstract area(): number;
}

class Circle extends Shape {
  constructor(public radius: number) { super(); }
  area(): number { return Math.PI * this.radius ** 2; }
}

// Abstract class bilan ishlatish — concrete subclass'lar uchun
function createShape(Ctor: new (...args: any[]) => Shape, ...args: any[]): Shape {
  return new Ctor(...args);
}

const circle = createShape(Circle, 5);
// const shape = createShape(Shape); // ❌ Error: Cannot create instance of abstract class
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type UserConstructor = {
  new (name: string, age: number): { name: string; age: number };
};

function createUser(Ctor: UserConstructor, name: string, age: number) {
  return new Ctor(name, age);
}
```

```javascript
// Compiled JS — construct signature type butunlay o'chiriladi
function createUser(Ctor, name, age) {
  return new Ctor(name, age);
}
```

Construct signature faqat compile-time'da `new` chaqiruvining argument va return type'larini tekshiradi. JavaScript output'da faqat oddiy `new` expression qoladi — runtime type'ni tekshirmaydi.

</details>

---

## this Parameter

### Nazariya

TypeScript'da `this` parameter — funksiya ichida `this`'ning type'ini belgilaydigan **maxsus parametr**. U funksiya signaturasida birinchi parametr sifatida yoziladi, lekin **runtime'da mavjud emas** — faqat compile-time tekshiruvi uchun.

JavaScript'da `this` context'ga qarab o'zgaradi — bu ko'p bug'larga sabab bo'ladi. Method reference (`const fn = obj.method; fn();`) — `this` yo'qoladi. Callback'ga method berish — `this` yo'qoladi. TypeScript'ning `this` parameter shu muammoni type darajasida hal qiladi.

```typescript
interface UIElement {
  id: string;
  onClick: (this: UIElement, event: MouseEvent) => void;
}

const button: UIElement = {
  id: "submit-btn",
  onClick(this: UIElement, event: MouseEvent) {
    // this: UIElement — TS kafolatlaydi
    console.log(`Button ${this.id} clicked at (${event.clientX}, ${event.clientY})`);
  },
};

// ✅ To'g'ri chaqirish — this context to'g'ri
button.onClick(new MouseEvent("click"));

// ❌ Noto'g'ri chaqirish — this yo'qoladi
// const handler = button.onClick;
// handler(new MouseEvent("click"));
// Error: The 'this' context of type 'void' is not assignable to type 'UIElement'
```

**`noImplicitThis` option** (`strict: true` ostida avtomatik yoqilgan) — `this` type aniq bo'lmagan holatlarda xato beradi. Bu option'siz `this` avtomatik `any` bo'lib qoladi, bu type safety'ni yo'qotadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`this` parameter — TypeScript'ning **eng aniq type erasure** misollaridan biri. Compile bo'lganda birinchi `this` parameter **butunlay o'chiriladi** — JavaScript output'da u mavjud emas. Runtime'da `this` parameter hech qanday himoya bermaydi — faqat compile-time xavfsizlik.

TypeScript kompilatori `this` parameter'ni boshqa parametrlardan farqlaydi — birinchi parameter nomi `this` bo'lsa, uni **pseudo-parameter** sifatida belgilaydi. Parameter list'dan chiqarib tashlanadi va funksiya chaqiruvlarida argument count'ga qo'shilmaydi.

```
TS source:
function handleClick(this: HTMLButtonElement, event: MouseEvent): void { ... }

Compiler internal:
  Parameters: [this: HTMLButtonElement, event: MouseEvent]
  Actual JS parameters: [event]           // this o'chirildi
  Expected argument count: 1              // this hisoblanmaydi
  thisType: HTMLButtonElement             // alohida saqlanadi

Compiled JS:
function handleClick(event) { ... }
```

Kompilator `this` parameter turgan funksiya chaqiruvlarini tekshiradi — chaqiruv konteksti (method call, `.call()`, `.bind()`) orqali `this`'ning type'ini aniqlaydi va belgilangan type'ga mos kelishini verify qiladi. Agar funksiya detach qilinsa (variable'ga assign bo'lsa), `this` kontekst yo'qolishi haqida xato beradi.

**Arrow funksiya va `this`:** Arrow funksiyalarda `this` **lexical** — o'rab turgan scope'dan olinadi. Shuning uchun arrow funksiyada `this` parameter ishlatib bo'lmaydi — TypeScript buni taqiqlaydi:

```typescript
// ❌ Arrow function'da this parameter yo'q
// const fn = (this: MyType, x: number) => { }; // Error
```

Arrow funksiya method sifatida ishlatilganda (`obj.method = () => { ... }`), `this` obj'ga emas, tashqi scope'ga ishora qiladi. Bu ba'zan xohlangan (callback'da `this` yo'qolishini oldini olish), ba'zan muammo (class method sifatida `this` noto'g'ri ishora qiladi).

**React context:** React class component'larda `this` binding klassik muammo. JSX event handler `<button onClick={this.handleClick}>` — `this` yo'qoladi. Yechimlar:

1. Constructor'da `this.handleClick = this.handleClick.bind(this)` — bind
2. Arrow method property: `handleClick = () => { ... }` — arrow method
3. Inline arrow: `<button onClick={() => this.handleClick()}>` — inline capture

TypeScript'ning `this` parameter ikkinchi va uchinchi variantni type-safe qiladi.

**Compile-time xavfsizlik:** `this` parameter runtime'da hech narsa qilmaydi — lekin compile-time'da "wrong caller" xatosini ushlaydi. Bu — prevention over cure yondashuv. JavaScript'da bunday tekshiruv mumkin emas, chunki `this` dinamik.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Method type bilan this parameter
interface Toggle {
  active: boolean;
  toggle(this: Toggle): void;
}

const toggle: Toggle = {
  active: false,
  toggle(this: Toggle) {
    this.active = !this.active;
  },
};

toggle.toggle();     // ✅ active = true
toggle.toggle();     // ✅ active = false

// ❌ Method detach qilish xato beradi
// const detached = toggle.toggle;
// detached(); // Error: this: Toggle not assignable to this: void

// 2. DOM event handler with this parameter
function onButtonClick(this: HTMLButtonElement, event: MouseEvent): void {
  console.log(`Button ${this.textContent} clicked`);
  this.disabled = true;
}

// ✅ addEventListener orqali chaqirish — this = button
document.querySelector("button")?.addEventListener("click", onButtonClick);

// 3. Arrow method property — this lexical
class Counter {
  count = 0;

  // Method property — arrow function, this lexical
  increment = (): void => {
    this.count += 1;
  };

  // Regular method — this parameter kerak bo'lishi mumkin
  reset(this: Counter): void {
    this.count = 0;
  }
}

const counter = new Counter();

// Arrow method property — this doim Counter instance
const incRef = counter.increment;
incRef(); // ✅ count = 1, chunki arrow this lexical

// Regular method — detach qilinsa this yo'qoladi
// const resetRef = counter.reset;
// resetRef(); // ❌ Error: this: Counter not assignable

// 4. bind bilan this binding
class Logger {
  prefix: string;

  constructor(prefix: string) {
    this.prefix = prefix;
  }

  log(this: Logger, message: string): void {
    console.log(`[${this.prefix}] ${message}`);
  }
}

const logger = new Logger("APP");
logger.log("hello"); // "[APP] hello"

// bind bilan this'ni saqlash
const boundLog = logger.log.bind(logger);
boundLog("world"); // ✅ "[APP] world"

// 5. call va apply bilan this aniqlash
function introduce(this: { name: string }, greeting: string): string {
  return `${greeting}, ${this.name}`;
}

const user = { name: "Ali" };
introduce.call(user, "Salom");  // "Salom, Ali"
introduce.apply(user, ["Hi"]);  // "Hi, Ali"

// 6. React-style callback handler
type EventHandler<E extends Event, T> = (this: T, event: E) => void;

function attachHandler<T>(target: T, handler: EventHandler<Event, T>): void {
  // handler.call(target, event) shaklida chaqiriladi
  console.log("Handler attached");
}

attachHandler({ name: "button" }, function (event) {
  // this: { name: string }
  console.log(this.name);
});
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function handleClick(this: HTMLButtonElement, event: MouseEvent): void {
  console.log(this.textContent);
}
```

```javascript
// Compiled JS — this parameter butunlay o'chiriladi
function handleClick(event) {
  console.log(this.textContent);
}
// Runtime da this — JavaScript ning oddiy this qoidalari bo'yicha aniqlanadi
// (method call, .call, .bind, new, arrow function lexical)
```

`this` parameter — **type erasure** qurboni. JavaScript'ga compile bo'lganda u butunlay yo'qoladi. Lekin compile-time'da kompilator chaqiruv kontekstini tekshiradi va noto'g'ri usage'ni topadi.

</details>

---

## void vs undefined

### Nazariya

`void` va `undefined` — ikkalasi ham "hech narsa qaytarmaydi" ma'nosini beradi, lekin **semantik farqi** bor:

- **`void`** — funksiya return value'ni **ishlatmaslik kerak** degan signal. Funksiya aslida `undefined` qaytarishi mumkin, lekin consumer uni **e'tiborsiz qoldirishi kerak**.
- **`undefined`** — funksiya aniq `undefined` qaytaradi va shu qiymat ishlatilishi **mumkin**.

```typescript
// void — return value ishlatilmasligi kerak
function logMessage(msg: string): void {
  console.log(msg);
  // Implicit return undefined — lekin type void
}

// undefined — aniq undefined qaytaradi
function findItem(arr: number[], target: number): number | undefined {
  return arr.find((x) => x === target);
  // undefined qaytarishi mumkin va consumer buni tekshirishi kerak
}
```

**`void`'ning maxsus xatti-harakati callback context'da.** Kompilator `() => void` type'li callback uchun return type'ni **ignore** qiladi — callback istalgan qiymat qaytarishi mumkin, lekin u qiymat **o'qilmaydi**. Bu JavaScript'ning real pattern'ini support qilish uchun kiritilgan ongli dizayn qarori.

```typescript
// forEach callback type: (value: T) => void
const numbers = [1, 2, 3];

// ✅ push() number qaytaradi, lekin void callback bu'ni ignore qiladi
numbers.forEach((n) => numbers.push(n * 2));

// Bu ishlaydi — chunki JS'da `Array.forEach` callback'dan hech nima kutmaydi
// TS tilining dizayn qarori: real-world kod ishlashi uchun
```

**Muhim:** Bu xatti-harakat **faqat callback context**'da ishlaydi. Funksiya declaration'da `void` return type qo'yilsa — return qiymat berish **xato**:

```typescript
// Function declaration'da void — return qiymat bermaydi
function doSomething(): void {
  // return 42; // ❌ Error: Type 'number' not assignable to type 'void'
}

// Callback type'da void — return qiymat ignored
type Callback = () => void;
const cb: Callback = () => 42; // ✅ — void callback'da ruxsat
```

<details>
<summary><strong>Under the Hood</strong></summary>

`void` va `undefined` — JavaScript runtime'da **ikkalasi ham** `undefined` qiymatga resolve bo'ladi. Farq faqat TypeScript type system ichida mavjud — compiled JS'da ular orasida hech qanday farq yo'q.

TypeScript kompilatori ichida `void` va `undefined` **alohida type node**'lar sifatida saqlanadi. `void` — `VoidKeyword`, `undefined` — `UndefinedKeyword`. Checker ular orasida assignability qoidalarini quyidagicha belgilaydi:

```
Type assignability:
  undefined → void:  ✅ (undefined void'ga assign bo'ladi)
  void → undefined:  ❌ (void undefined'ga assign bo'lmaydi, strictNullChecks'da)
  number → void:     ❌ (hech qanday type void'ga assign bo'lmaydi, callback'dan tashqari)
```

**`void`'ning maxsus callback xatti-harakati:** `isTypeAssignableTo` funksiyasida maxsus case sifatida handle qilinadi. Agar target type `void` va expression "contextual type" (callback, method arg) da ishlatilsa, kompilator return type'ni tekshirmaydi. Agar funksiya deklaratsiyasi bo'lsa (standalone function yoki method body'si), return type tekshiriladi va `void` uchun return qiymat taqiqlanadi.

**Nima uchun bu asimmetriya?** JavaScript'da `Array.prototype.forEach`, `Array.prototype.map`, event handler'lar kabi ko'p API'lar callback'dan void kutadi. Lekin developer'lar ko'pincha return qiymatli expression yozadilar:

```javascript
// Keng tarqalgan JS pattern
[1, 2, 3].forEach(n => arr.push(n));  // push number qaytaradi
buttons.forEach(btn => btn.setAttribute("disabled", "true")); // setAttribute void
```

Agar TypeScript void callback'da return'ni taqiqlasa — bu pattern'lar ishlamaydi. Shuning uchun void callback **qaytarilgan qiymatni ignore qiladi** — caller o'sha qiymat'ga ulana olmaydi, chunki type'i `void`.

**`never` return type farqi:** `never` — funksiya **hech qachon qaytarmaydi** (throw, infinite loop, process.exit). `void` — funksiya return qiladi, lekin qiymat foydasiz. `undefined` — qiymat aniq `undefined`. Bular uchta farqli concept.

```typescript
function a(): void { }        // return qiladi, void
function b(): undefined { return undefined; } // return undefined aniq
function c(): never { throw new Error(); }    // hech qachon qaytmaydi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. void return — consumer ignore qiladi
function saveToDatabase(user: { id: number }): void {
  // DB call simulation
  console.log(`Saved user ${user.id}`);
  // Implicit return undefined, but type void
}

const result = saveToDatabase({ id: 1 });
// result: void — consumer ishlatmaslik kerak

// 2. undefined return — consumer tekshirishi kerak
function findUser(id: number): { name: string } | undefined {
  const users = [{ id: 1, name: "Ali" }];
  return users.find(u => u.id === id);
}

const user = findUser(2);
if (user) {
  console.log(user.name); // ✅ narrowed
}

// 3. void callback — return qiymat ignored
type Handler = (event: string) => void;

const handlers: Handler[] = [];

handlers.push((event) => {
  console.log(event);
  return event.length; // ✅ returned, lekin void callback'da ignored
});

// 4. forEach with void callback
const numbers = [1, 2, 3];
const collected: number[] = [];

numbers.forEach(n => {
  collected.push(n * 2); // push returns number — void callback'da OK
});

// 5. Function declaration'da void — return taqiqlanadi
function greet(): void {
  console.log("Hello");
  // return "hello"; // ❌ Error: string not assignable to void
}

// 6. void bilan method reference
interface Publisher {
  notify(event: string): void;
}

const pub: Publisher = {
  notify(event) {
    // Implementation return number ham qilsa, void consumer ishlatmaydi
    return event.length as unknown as void;
  },
};

// 7. never vs void vs undefined
function voidFn(): void {
  console.log("something");
  // return undefined (implicit)
}

function undefinedFn(): undefined {
  return undefined; // Explicit undefined return
}

function neverFn(): never {
  throw new Error("unreachable");
  // Yoki: while (true) { } — infinite loop
}

// 8. void callback'da error handling
type AsyncVoid = () => Promise<void>;

const task: AsyncVoid = async () => {
  const data = await fetch("/api/data");
  return data.status as unknown as void; // ignored by caller
};

// 9. Explicit undefined vs void parameter
function optional(x?: number): void {}  // x?: number → number | undefined
function explicit(x: number | undefined): void {} // x har doim berilishi kerak

optional();            // ✅
optional(42);          // ✅
// explicit();         // ❌
explicit(undefined);   // ✅
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function logMessage(msg: string): void {
  console.log(msg);
}

function findItem(arr: number[], target: number): number | undefined {
  return arr.find(x => x === target);
}
```

```javascript
// Compiled JS — void va undefined type annotation'lari o'chiriladi
function logMessage(msg) {
  console.log(msg);
}

function findItem(arr, target) {
  return arr.find(x => x === target);
}
```

`void` va `undefined` — ikkalasi ham runtime'da `undefined` qiymat. Type annotation'lar o'chiriladi, funksiya xatti-harakati o'zgarmaydi. Farq faqat TypeScript compile-time semantikasi'da.

</details>

---

## Function Type Expressions

### Nazariya

Function type expression — funksiyaning type'ini alohida type sifatida yozish. Bu callback'lar, higher-order funksiyalar, va type reuse uchun ishlatiladi.

```typescript
// Arrow syntax bilan function type
type Predicate<T> = (value: T) => boolean;
type Transformer<T, U> = (input: T) => U;
type ErrorHandler = (error: Error, context?: string) => void;

// Ishlatish
function filter<T>(arr: T[], predicate: Predicate<T>): T[] {
  return arr.filter(predicate);
}

const isPositive: Predicate<number> = (n) => n > 0;
const result = filter([1, -2, 3, -4], isPositive); // [1, 3]
```

Function type expression'lar **reusable** type yaratish uchun ishlatiladi — bitta joyda e'lon qilib, bir nechta joyda ishlatish. Bu DRY (Don't Repeat Yourself) printsipga mos.

<details>
<summary><strong>Under the Hood</strong></summary>

Function type expression — **butunlay type-level** konstruksiya. Compiled JS'da `type Predicate<T> = (value: T) => boolean` kabi deklaratsiyalar **to'liq o'chiriladi** — hech qanday JS kodi hosil bo'lmaydi.

TypeScript kompilatori ichida function type expression `FunctionTypeNode` AST node sifatida parse qilinadi. Bu node `TypeAliasDeclaration` ichida turadi (`type X = ...`). Kompilator type alias'ni **resolve** qilganda, function type'ning parametr type'lari va return type'ini `Signature` object'ga aylantiradi — bu real funksiya deklaratsiyasining signature'i bilan bir xil internal representation.

```
type Predicate<T> = (value: T) => boolean;

Compiler internal:
  TypeAlias "Predicate" → FunctionTypeNode
    TypeParameter: T
    Parameters: [value: T]
    ReturnType: boolean

Ishlatilganda:
  function filter<T>(arr: T[], predicate: Predicate<T>): T[]
  → Predicate<T> resolve bo'ladi → (value: T) => boolean
  → predicate parameter'ning type'i sifatida tekshiriladi
```

Generic function type (`Predicate<T>`) instantiate bo'lganda, kompilator `T`'ni aniq type bilan almashtiradi va yangi `Signature` hosil qiladi. Bu jarayon ham faqat compile-time'da sodir bo'ladi — runtime'da hech qanday iz yo'q.

**Function type vs call signature farqi:**

```typescript
// Arrow syntax — faqat funksiya, property yo'q
type Fn1 = (x: number) => string;

// Call signature — funksiya + property'lar
type Fn2 = {
  (x: number): string;
  description: string;
};
```

Farq: `Fn1`'da property qo'shib bo'lmaydi, `Fn2`'da property qo'shish mumkin. Ikkalasi ham kompilator ichida `Signature` object sifatida saqlanadi, lekin `Fn2`'da qo'shimcha property signature'lar ham bor. Agar faqat funksiya kerak bo'lsa, arrow syntax afzal — qisqa va o'qishga oson.

**Type alias vs interface uchun funksiya type:**

```typescript
type FnType = (x: number) => string;

interface FnInterface {
  (x: number): string;
}
```

Ikkalasi ham ishlaydi va teng. Farq: `interface`'da declaration merging mumkin (bir xil nomli ikkita interface birlashadi), `type alias`'da yo'q. Oddiy funksiya type uchun type alias tavsiya etiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy function type alias
type Comparator<T> = (a: T, b: T) => number;

function sortBy<T>(arr: T[], cmp: Comparator<T>): T[] {
  return [...arr].sort(cmp);
}

const numbers = sortBy([3, 1, 4, 1, 5], (a, b) => a - b);
// numbers: [1, 1, 3, 4, 5]

// 2. Async function type
type AsyncOperation<T> = (input: T) => Promise<T>;

const doubleAsync: AsyncOperation<number> = async (n) => n * 2;

// 3. Higher-order function type
type Middleware = (
  req: { url: string },
  res: { send: (data: string) => void },
  next: () => void
) => void;

const loggingMiddleware: Middleware = (req, res, next) => {
  console.log(`Request to ${req.url}`);
  next();
};

// 4. Event handler type
type EventHandler<E extends Event> = (event: E) => void;

function onClick(handler: EventHandler<MouseEvent>): void {
  document.addEventListener("click", handler);
}

onClick((event) => {
  console.log(event.clientX, event.clientY); // event: MouseEvent
});

// 5. Curried function type
type CurriedAdd = (a: number) => (b: number) => number;

const add: CurriedAdd = (a) => (b) => a + b;
const add5 = add(5);
add5(10); // 15

// 6. Function with this parameter
type BoundMethod<T, R> = (this: T) => R;

class User {
  constructor(public name: string) {}
}

const greet: BoundMethod<User, string> = function() {
  return `Hi, ${this.name}`;
};

const user = new User("Ali");
greet.call(user); // "Hi, Ali"

// 7. Validator function type
type Validator<T> = (value: T) => { valid: boolean; errors: string[] };

const emailValidator: Validator<string> = (email) => {
  const errors: string[] = [];
  if (!email.includes("@")) errors.push("Missing @");
  if (!email.includes(".")) errors.push("Missing dot");
  return { valid: errors.length === 0, errors };
};

// 8. Generic validator bilan type inference
function validate<T>(value: T, validator: Validator<T>): boolean {
  return validator(value).valid;
}

validate("ali@mail.com", emailValidator); // true
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type Predicate<T> = (value: T) => boolean;
type Transformer<T, U> = (input: T) => U;

const isPositive: Predicate<number> = (n) => n > 0;
const toStr: Transformer<number, string> = (n) => String(n);
```

```javascript
// Compiled JS — type alias'lar butunlay o'chiriladi
const isPositive = (n) => n > 0;
const toStr = (n) => String(n);
```

`type Predicate<T> = ...` va `type Transformer<T, U> = ...` — JS output'da hech qanday kod hosil qilmaydi. Type alias — **nol-cost abstraction** — faqat compile-time'da mavjud.

</details>

---

## Generic Functions — Kirish

### Nazariya

Generic function — **type parameter** qabul qiladigan funksiya. Type parameter funksiya ichida type placeholder sifatida ishlaydi — chaqirilganda aniq type bilan almashtiriladi. Bu "type o'zgaruvchisi" konseptsiyasi.

Generic funksiya yozishning asosiy sababi — **bir xil mantiq** turli type'lar bilan ishlashini ta'minlash, **type safety**'ni yo'qotmasdan.

```typescript
// ❌ Generic siz — type safety yo'q
function firstElementAny(arr: any[]): any {
  return arr[0];
}
const val1 = firstElementAny([1, 2, 3]); // any — type yo'qoldi

// ❌ Overload bilan — har type uchun alohida
function firstElementOverload(arr: string[]): string;
function firstElementOverload(arr: number[]): number;
function firstElementOverload(arr: (string | number)[]): string | number {
  return arr[0];
}
// Boolean uchun yana overload kerak... cheksiz

// ✅ Generic — bitta funksiya, barcha type'lar uchun
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const str = firstElement(["a", "b"]); // string | undefined
const num = firstElement([1, 2, 3]);  // number | undefined
const bool = firstElement([true]);    // boolean | undefined
```

**Generic Inference:** TypeScript ko'pincha type parameter'ni **o'zi aniqlaydi** (infer qiladi) — explicit berish shart emas.

**Generic Constraint:** `extends` keyword bilan type parameter'ni cheklash — faqat ma'lum shape'ga mos type'lar qabul qilinadi.

Chuqur generic'lar [08-generics.md](08-generics.md)'da — bu yerda intro.

<details>
<summary><strong>Under the Hood</strong></summary>

Generic type parameter'lar — **to'liq compile-time** mexanizm. Compiled JS'da `<T>`, `<T, U>` kabi type parameter'lar **butunlay o'chiriladi** — runtime'da generic'lar mavjud emas.

```
TS source:
function firstElement<T>(arr: T[]): T | undefined { return arr[0]; }

Compiled JS:
function firstElement(arr) { return arr[0]; }
// <T> va T[] type annotation'lar to'liq o'chirildi
```

TypeScript kompilatori generic funksiya chaqiruvini ko'rganda **type inference** jarayonini ishga tushiradi. Bu jarayon `checker.ts`'dagi `inferTypeArguments()` funksiyasida amalga oshiriladi. Kompilator argument'ning actual type'ini parametr type'idagi type variable bilan **unification** qiladi:

```
firstElement(["a", "b"]) chaqiruvida:
  1. Argument type: string[] (inferred from ["a", "b"])
  2. Parameter type: T[]
  3. Unification: T[] = string[] → T = string
  4. Return type: T | undefined → string | undefined
```

Agar kompilator type'ni infer qila olmasa, constraint type yoki `unknown`'ga fallback qiladi. Explicit type argument — `firstElement<number>([1, 2])` — inference'ni override qiladi.

**Generic constraint** (`extends`) — compile-time'da type parameter'ning upper bound'ini belgilaydi. `T extends { length: number }` deganda, kompilator `T` sifatida faqat `length: number` property'ga ega type'larni qabul qiladi. Bu constraint ham runtime'da yo'q — faqat checker tomonidan tekshiriladi.

**`keyof` bilan generic constraint** — juda foydali pattern:

```typescript
// K type parameter faqat T'ning key'laridan biri bo'lishi mumkin
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Ali", age: 25, email: "ali@mail.com" };
getProperty(user, "name"); // string
getProperty(user, "age");   // number
// getProperty(user, "phone"); // ❌ Error: "phone" keyof user'da yo'q
```

Bu pattern `08-generics.md` va `13-mapped-types.md`'da batafsil ko'rib chiqiladi. Hozircha `K extends keyof T` konstruksiyasi — "K, T'ning key'laridan biri" deganlikdir.

**`const` type parameter** (TS 5.0+) — type parameter'ga `const` qo'shib, inference'da literal type'larni saqlash:

```typescript
function asTuple<const T extends readonly unknown[]>(arr: T): T {
  return arr;
}

const t = asTuple([1, 2, 3]);
// TS 5.0+: t: readonly [1, 2, 3] (literal tuple)
// TS 5.0-: t: number[]
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy generic funksiya
function identity<T>(value: T): T {
  return value;
}

const str = identity("hello"); // string
const num = identity(42);       // number
const arr = identity([1, 2]);   // number[]

// 2. Ko'plab type parameter'lar
function pair<A, B>(a: A, b: B): [A, B] {
  return [a, b];
}

const p = pair("hello", 42); // [string, number]

// 3. Generic with constraint
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}

getLength("hello");     // ✅ string.length
getLength([1, 2, 3]);   // ✅ array.length
getLength({ length: 5, data: "x" }); // ✅ length property bor
// getLength(42);        // ❌ Error: number'da length yo'q

// 4. keyof bilan type-safe property access
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Ali", age: 25 };
const name = getProperty(user, "name"); // string
const age = getProperty(user, "age");    // number

// 5. Array methods — generic inference
function map<T, U>(arr: T[], fn: (value: T) => U): U[] {
  return arr.map(fn);
}

const lengths = map(["a", "bb", "ccc"], s => s.length);
// lengths: number[]

const doubled = map([1, 2, 3], n => n * 2);
// doubled: number[]

// 6. Default type parameter
function createArray<T = string>(length: number, value: T): T[] {
  return Array(length).fill(value);
}

const str1 = createArray(3, "hi");     // string[]
const num1 = createArray<number>(3, 0); // number[]

// 7. Generic with explicit type argument
function parse<T>(json: string): T {
  return JSON.parse(json);
}

interface User {
  name: string;
  age: number;
}

const user1 = parse<User>('{"name":"Ali","age":25}');
// user1: User

// 8. Inference with constraint
function pluck<T, K extends keyof T>(arr: T[], key: K): T[K][] {
  return arr.map(item => item[key]);
}

const users = [
  { name: "Ali", age: 25 },
  { name: "Vali", age: 30 },
];

const names = pluck(users, "name"); // string[]
const ages = pluck(users, "age");    // number[]
```

</details>

---

## Callback Types

### Nazariya

Callback — boshqa funksiyaga argument sifatida beriladigan funksiya. TypeScript'da callback'ning type'ini to'g'ri yozish — event handler'lar, Promise chain'lari, async pattern'lar, va higher-order funksiyalar uchun muhim.

Callback pattern'lar JavaScript/TypeScript'da hammajoyda ishlatiladi:

- **Event handler'lar** — `button.addEventListener("click", handler)`
- **Array method'lar** — `arr.map`, `arr.filter`, `arr.forEach`, `arr.reduce`
- **Promise chain'lar** — `.then(onFulfilled, onRejected)`
- **Node.js callback'lar** — `fs.readFile(path, (err, data) => {})`
- **Higher-order function'lar** — `retry`, `debounce`, `throttle`

**Contextual typing** — TypeScript'ning eng foydali callback feature'laridan biri. Callback inline yozilganda (arrow function sifatida), kompilator tashqi funksiya signaturasidan callback'ning parameter type'larini **infer** qiladi. Shuning uchun inline callback'larda parametr type yozish ko'pincha kerak emas.

```typescript
// addEventListener type: (handler: (event: MouseEvent) => void) => void
document.body.addEventListener("click", (event) => {
  // event: MouseEvent — TS contextual typing'dan infer qildi
  console.log(event.clientX, event.clientY);
});
```

**Type safety callback'larda:** Callback'ning return type'i ham ahamiyatli. `Array.map` callback'dan qaytgan qiymat'ni yig'adi, `Array.forEach` ignore qiladi (void callback).

<details>
<summary><strong>Under the Hood</strong></summary>

Callback type annotation'lari — boshqa type annotation'lar kabi **compile-time'da to'liq o'chiriladi**. Runtime'da callback funksiyalar oddiy JS function object sifatida beriladi — hech qanday type metadata saqlanmaydi.

TypeScript kompilatori callback type'larni tekshirishda **contextual typing** mexanizmidan foydalanadi. Callback inline yozilganda, kompilator tashqi funksiya signaturasidagi parametr type'idan callback'ning parametr type'larini infer qiladi:

```
addClickListener(document.body, (event) => { ... });

Contextual typing jarayoni:
  1. addClickListener'ning 2-parametri: handler: ClickHandler
  2. ClickHandler = (event: MouseEvent) => void
  3. Inline callback'da event parametr type berilmagan
  4. Contextual type'dan infer: event → MouseEvent
  5. Callback ichida event.clientX → MouseEvent'da bor → ✅
```

Bu **bidirectional type flow** — type ma'lumoti tashqi funksiyadan ichki callback'ga oqadi. Shuning uchun inline callback'larda parametr type yozish ko'pincha kerak emas — kompilator o'zi aniqlaydi.

**Callback type'lar uchun farqli contextual typing source'lari:**

1. **Type annotation** — `const fn: MyCallback = (x) => ...` — `MyCallback`'dan infer
2. **Funksiya argument** — `arr.map((x) => ...)` — `map`'ning parametr type'idan
3. **Property assignment** — `obj.handler = (x) => ...` — `obj.handler`'ning declared type'idan
4. **Return statement** — `return (x) => ...` — tashqi funksiyaning return type'idan

**Callback'da variance:** Callback parameter type'i **contravariant** (`strictFunctionTypes: true` bilan). Bu means — keng type'li callback'ni tor type'li joyga berish mumkin, lekin teskari emas. Bu Function Assignability bo'limida batafsil.

**Node.js style callback (error-first):** JavaScript'ning eng eski pattern'laridan biri — birinchi argument error, ikkinchi result:

```typescript
type NodeCallback<T> = (err: Error | null, result?: T) => void;

function readFile(path: string, cb: NodeCallback<string>): void {
  // implementation
}

readFile("/path", (err, data) => {
  if (err) return console.error(err);
  console.log(data); // data?: string
});
```

Zamonaviy kod'da Promise'lar afzal, lekin legacy code bilan ishlaganda bu pattern uchraydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. DOM event handler
type ClickHandler = (event: MouseEvent) => void;
type InputHandler = (event: InputEvent) => void;
type KeyHandler = (event: KeyboardEvent) => void;

function addClickListener(element: HTMLElement, handler: ClickHandler): void {
  element.addEventListener("click", handler);
}

// TS event type'ni tekshiradi
addClickListener(document.body, (event) => {
  // event: MouseEvent — contextual typing
  console.log(event.clientX, event.clientY);
});

// 2. Array method callbacks
const numbers = [1, 2, 3, 4, 5];

// map callback: (value: number, index: number, array: number[]) => U
const doubled = numbers.map((n) => n * 2);
// doubled: number[]

// filter callback: (value: number, index: number, array: number[]) => unknown
const evens = numbers.filter((n) => n % 2 === 0);
// evens: number[]

// reduce callback: (acc: T, value: number, index: number, array: number[]) => T
const sum = numbers.reduce((acc, n) => acc + n, 0);
// sum: number

// 3. Node.js style callback
type AsyncCallback<T> = (error: Error | null, result: T | null) => void;

function fetchData(url: string, callback: AsyncCallback<string>): void {
  try {
    const content = "file content"; // simulyatsiya
    callback(null, content);
  } catch (error) {
    callback(error as Error, null);
  }
}

fetchData("/api/data", (error, data) => {
  if (error) {
    console.error(error.message);
    return;
  }
  console.log(data?.toUpperCase());
});

// 4. Higher-order function with generic callback
function retry<T>(
  operation: () => Promise<T>,
  maxAttempts: number,
  onRetry?: (attempt: number, error: Error) => void
): Promise<T> {
  return (async () => {
    let lastError: Error | undefined;

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error as Error;
        if (attempt < maxAttempts && onRetry) {
          onRetry(attempt, lastError);
        }
      }
    }
    throw lastError; // TS biladi: loop oxirida error bo'lishi kerak
  })();
}

// Ishlatish — TS barcha callback type'larni infer qiladi
async function example() {
  const data = await retry(
    () => fetch("/api/data").then(r => r.json()),
    3,
    (attempt, error) => {
      // attempt: number, error: Error — inferred
      console.log(`Retry ${attempt}: ${error.message}`);
    }
  );
}

// 5. Promise callback with type-safe error
interface ApiError {
  code: number;
  message: string;
}

function fetchUser(id: number): Promise<{ name: string }> {
  return new Promise((resolve, reject) => {
    if (id < 0) reject({ code: 400, message: "Invalid ID" } as ApiError);
    else resolve({ name: "Ali" });
  });
}

fetchUser(1)
  .then((user) => console.log(user.name))  // user: { name: string }
  .catch((err: ApiError) => console.error(err.message));

// 6. Generic predicate callback
function findFirst<T>(arr: T[], predicate: (item: T) => boolean): T | undefined {
  for (const item of arr) {
    if (predicate(item)) return item;
  }
  return undefined;
}

const user = findFirst([{ id: 1 }, { id: 2 }], (u) => u.id === 2);
// user: { id: number } | undefined

// 7. Callback with multiple type parameters
function mapWithIndex<T, U>(
  arr: T[],
  fn: (item: T, index: number) => U
): U[] {
  return arr.map(fn);
}

const indexed = mapWithIndex(["a", "b", "c"], (s, i) => `${i}:${s}`);
// indexed: string[] — ["0:a", "1:b", "2:c"]
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type ClickHandler = (event: MouseEvent) => void;

function addListener(el: HTMLElement, handler: ClickHandler): void {
  el.addEventListener("click", handler);
}
```

```javascript
// Compiled JS — callback type alias'lar butunlay o'chiriladi
function addListener(el, handler) {
  el.addEventListener("click", handler);
}
```

Callback type annotation'lar — boshqa type'lar kabi **compile-time'da o'chiriladi**. Runtime'da callback oddiy JS function object sifatida beriladi — hech qanday type metadata saqlanmaydi. TypeScript'ning katta foydasi — bu metadata'ni compile-time'da tekshirish va xato'larni runtime'dan oldin topish.

</details>

---

## Function Assignability

### Nazariya

TypeScript'da funksiya type'larining bir-biriga **assign** bo'lishi (mos kelishi) maxsus qoidalarga bo'ysunadi — **variance** qoidalari. Bu qoidalar callback'lar, higher-order funksiyalar, va type compatibility'da muhim.

**Variance** — type'larning hierarchy'dagi mos kelish yo'nalishi. Funksiyalarda ikkita variance turi muhim:

1. **Covariance** (bir xil yo'nalish) — tor type keng type'ga assign bo'ladi. **Output positions** uchun (return type).
2. **Contravariance** (teskari yo'nalish) — keng type tor type'ga assign bo'ladi. **Input positions** uchun (parametr type).

Intuitiv tushunish: funksiya return qilganda nima **chiqadi** (output), parametr olganda nima **kiradi** (input). Output'lar tabiiy, input'lar teskari.

```typescript
class Animal { name = "animal"; }
class Dog extends Animal { breed = "labrador"; }

// Return type — COVARIANT (tabiy yo'nalish)
type ReturnsAnimal = () => Animal;
type ReturnsDog = () => Dog;

const getDog: ReturnsDog = () => new Dog();
const getAnimal: ReturnsAnimal = getDog;
// ✅ Dog qaytaradigan funksiya Animal qaytaradigan joyga mos
// Sabab: Dog — Animal subtype, Animal expected joyda Dog ishlaydi

// Parameter type — CONTRAVARIANT (teskari yo'nalish)
type AnimalHandler = (animal: Animal) => void;
type DogHandler = (dog: Dog) => void;

const handleAnimal: AnimalHandler = (a) => console.log(a.name);
const handleDog: DogHandler = handleAnimal;
// ✅ Animal handler Dog handler joyiga mos
// Sabab: Animal handler har qanday animal qabul qila oladi,
//        Dog ham Animal — xavfsiz

const dogOnly: DogHandler = (d) => console.log(d.breed);
// const bad: AnimalHandler = dogOnly;
// ❌ Error: Dog handler barcha Animal'larni handle qila olmaydi
// Chunki Cat berilsa — breed yo'q, crash
```

**Method shorthand bivariance:** `strictFunctionTypes: true` flag parametr contravariance'ni **function property** syntax'da yoqadi, lekin **method shorthand** syntax'da parametr **bivariant** (ikki yo'nalishda ishlaydi) bo'lib qoladi.

```typescript
interface A {
  handler: (arg: Animal) => void;   // Function property — CONTRAVARIANT
}

interface B {
  handler(arg: Animal): void;        // Method shorthand — BIVARIANT
}
```

Sabab: DOM API'lar (masalan, `Array<T>.push`, `addEventListener`) bilan **backward compatibility**. `Array<Dog>.push` `Array<Animal>.push`'ga assign bo'lishi kerak edi — lekin bu contravariance qoidasiga zid. Shuning uchun method shorthand'da bivariance qoldirildi.

Chuqur variance — [25-type-compatibility.md](25-type-compatibility.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

Function assignability — TypeScript type system'ning eng murakkab qismlaridan biri. Bu tekshiruvlar `checker.ts`'dagi `isSignatureAssignableTo()` va `compareSignaturesRelated()` funksiyalarida amalga oshiriladi.

```
Function Type Assignability Algorithm:

Source signature:  (sA: SA, sB: SB) => SR
Target signature:  (tA: TA, tB: TB) => TR

1. Argument count check:
   source.minArgumentCount ≤ target.maxArgumentCount
   Source kamroq parameter olishi mumkin (extra ignored)

2. Parameter types (CONTRAVARIANT):
   For each i: TA ⊆ SA (target parameter subtype of source)
   Yani: source's parameter must be wider than or equal to target's

3. Return type (COVARIANT):
   SR ⊆ TR (source return type subtype of target)

4. this type (if declared):
   Source this type compatible with target this type
```

**Input vs output positions:**

- **Input position** — funksiya parametri `f(x: T)`'da `T`, lekin `f(g: (x: T) => U)`'dagi `g`'ning input position'ida `T` **output position**'da!
- **Output position** — funksiya return type `f(): T`'da `T`

Bu farq nested function type'larda muhim:

```typescript
// Outer funksiya: callback'ni parametr sifatida qabul qiladi
// Callback'ning parametri — double negation → covariant (tabi yo'nalish)
type Outer = (callback: (arg: Animal) => void) => void;

// Bu teng:
type Equivalent = (callback: (arg: Dog) => void) => void;
// Outer → Equivalent assign bo'ladimi? Parametr contravariant, ichki parametr esa
// double contravariant = covariant. Lekin actual TS qoidasi murakkabroq.
```

Murakkab misollar uchun intuitiv qoida: **har "kirish-chiqish" (`→`) yo'nalishini o'zgartiradi**. Bir `→` — contravariant (input), ikki `→` — covariant qayta.

**Parameter count soni tekshiruvi:** Kompilator target funksiyaning `minArgumentCount`'ini source funksiyaning parameter count bilan solishtiradi. Source'da kamroq parameter bo'lishi mumkin — ortiqcha parametr'lar ignore bo'ladi (JS xatti-harakatiga mos).

```typescript
type TwoArgs = (a: number, b: string) => void;

const fn1: TwoArgs = (a) => console.log(a);     // ✅ 1 parameter
const fn2: TwoArgs = () => console.log("x");    // ✅ 0 parameter
// Array.forEach callback pattern: (value, index, array) => void
// Ko'pincha faqat value ishlatiladi
```

**`strictFunctionTypes` flag:** Faqat **function type expression** syntax'da (`property: (x: T) => U`) contravariance'ni yoqadi. **Method shorthand** syntax'da (`method(x: T): U`) parameter'lar bivariant bo'lib qoladi — bu ongli dizayn qarori.

```
strictFunctionTypes: true:
  Function property: (x: T) => U → CONTRAVARIANT x
  Method shorthand:  method(x: T): U → BIVARIANT x (backward compat)
```

Kompilator AST node type'iga qarab qaysi variance qoidasini ishlatishni aniqlaydi. `PropertySignature` + `FunctionType` → contravariant, `MethodSignature` → bivariant.

**Nima uchun bu farq muhim?** Real DOM API'da:

```typescript
interface Array<T> {
  push(...items: T[]): number;  // method shorthand — bivariant
}

// Bu ishlashini ta'minlash uchun:
const dogs: Dog[] = [];
const animals: Animal[] = dogs;  // ✅ array covariant (strictly unsafe but allowed)
animals.push(new Cat()); // runtime'da dogs'ga Cat qo'shildi!
```

Bu "holesome" — runtime'da xavfli, lekin TypeScript backward compatibility uchun qo'llab-quvvatlaydi. Agar bu xavf bo'lsa, `readonly Dog[]` ishlatish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Parameter count — kamroq mos
type TwoParams = (a: number, b: string) => void;

const fn1: TwoParams = (a) => console.log(a);       // ✅ 1 parameter
const fn2: TwoParams = () => console.log("nothing"); // ✅ 0 parameter

// forEach callback pattern
[1, 2, 3].forEach((value) => console.log(value));
// index va array ignored — to'g'ri

// 2. Parameter contravariance
class Shape { color = "red"; }
class Circle extends Shape { radius = 5; }

type ShapeHandler = (shape: Shape) => void;
type CircleHandler = (circle: Circle) => void;

const handleShape: ShapeHandler = (s) => console.log(s.color);
const handleCircle: CircleHandler = handleShape;
// ✅ Shape handler — Circle handle qila oladi (Circle — Shape)

const circleOnly: CircleHandler = (c) => console.log(c.radius);
// const bad: ShapeHandler = circleOnly;
// ❌ Error: Circle handler barcha Shape'larni handle qila olmaydi

// 3. Return type covariance
type ReturnsShape = () => Shape;
type ReturnsCircle = () => Circle;

const getCircle: ReturnsCircle = () => new Circle();
const getShape: ReturnsShape = getCircle; // ✅ Circle — Shape subtype

// 4. Callback pattern with variance
function applyToAll<T>(items: T[], fn: (item: T) => void): void {
  items.forEach(fn);
}

// Oddiy Animal handler ishlatish
const animals = [new Shape(), new Circle()];
applyToAll(animals, (item) => console.log(item.color));
// ✅ Ishlaydi, chunki Circle ham Shape

// 5. Method shorthand bivariance
interface Container<T> {
  add(item: T): void;      // method shorthand
  remove: (item: T) => void; // function property
}

const dogContainer: Container<Circle> = {
  add(item) { console.log(item.radius); },
  remove: (item) => console.log(item.radius),
};

// Bu ishlaydi method shorthand bivariance tufayli:
const shapeContainer: Container<Shape> = dogContainer;
// ❌ Aslida xavfli — shapeContainer.add(new Shape())
//    dogContainer.add'ga ulanadi, Circle kutilgan lekin Shape keladi
// Lekin method shorthand backward compat uchun ruxsat beradi

// 6. Array covariance (unsafe)
const circles: Circle[] = [new Circle()];
const shapes: Shape[] = circles; // ✅ allowed (unsafe)
// shapes.push(new Shape()); // runtime'da circles'ga Shape qo'shildi!

// readonly — xavfsiz covariance
const readOnlyShapes: readonly Shape[] = circles; // ✅ xavfsiz
// readOnlyShapes.push(...); // ❌ readonly

// 7. Generic constraint va variance
function compose<A, B, C>(
  f: (x: A) => B,
  g: (y: B) => C
): (x: A) => C {
  return (x) => g(f(x));
}

const toStr: (n: number) => string = (n) => String(n);
const toUpper: (s: string) => string = (s) => s.toUpperCase();

const pipeline = compose(toStr, toUpper);
pipeline(42); // "42"
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type AnimalHandler = (animal: Animal) => void;
type DogHandler = (dog: Dog) => void;

const handleAnimal: AnimalHandler = (a) => console.log(a.name);
const handleDog: DogHandler = handleAnimal; // contravariance
```

```javascript
// Compiled JS — assignability tekshiruvi butunlay yo'qoladi
const handleAnimal = (a) => console.log(a.name);
const handleDog = handleAnimal;
```

Function assignability — **faqat compile-time** tekshiruv. Runtime'da JavaScript hech qanday parameter type yoki return type tekshirmaydi — noto'g'ri type berilsa crash yoki unexpected behavior bo'ladi. TypeScript bu xatolarni **oldindan** topadi.

</details>

---

## Edge Cases va Gotchas

Bu bo'limda funksiya type'larining nozik va kutilmagan xatti-harakatlarini ko'ramiz.

### 1. `typeof fn === "function"` Class Constructor'larni Ham Qamrab Oladi

JavaScript'da class constructor'lar ham funksiya — `typeof MyClass === "function"`. TypeScript `typeof x === "function"` narrowing'ida ikkalasini bir xil handle qiladi.

```typescript
class User { constructor(public name: string) {} }

function handleCallable(value: ((x: number) => string) | User | string): void {
  if (typeof value === "function") {
    // value: ((x: number) => string) | typeof User
    // Lekin User class — callable, narrowing oddiy funksiya'ga toraytirmaydi
    // Ba'zida kompilator union'ni "callable" ga qisqartiradi, ba'zida — yo'q
  }
}
```

Bu gotcha [06-type-narrowing.md](06-type-narrowing.md)'da batafsil. Yechim — class'larni `instanceof` bilan narrow qilish, funksiyalarni alohida.

### 2. Arrow Function vs Function Declaration — `this` Binding

Arrow function'lar `this`'ni **lexically** oladi — tashqi scope'dan. Function declaration/expression esa `this`'ni call-site'ga qarab aniqlaydi.

```typescript
class Counter {
  count = 0;

  // ❌ Method — this call-site'ga bog'liq
  incrementBad() {
    this.count += 1;
  }

  // ✅ Arrow property — this doim Counter instance
  incrementGood = () => {
    this.count += 1;
  };
}

const counter = new Counter();

// Method detach — this yo'qoladi
const bad = counter.incrementBad;
// bad(); // Runtime error: cannot read 'count' of undefined

// Arrow property detach — this saqlanadi
const good = counter.incrementGood;
good(); // ✅ counter.count = 1
```

**Narx:** Arrow property har instance uchun yangi funksiya yaratadi (prototype'da emas). Ko'p instance bo'lsa — xotira iste'moli katta. Method shorthand esa prototype'da bo'lib, memory'da tejamkor.

### 3. Default Parameter Har Chaqiriqda Qayta Evaluate Qilinadi

JavaScript'da default parameter har funksiya chaqiriqda qayta hisoblanadi. Bu Python'dan farqli — Python'da default value bir marta hisoblanib saqlanadi (va bu klassik bug).

```typescript
function addToList(item: string, list: string[] = []): string[] {
  list.push(item);
  return list;
}

const a = addToList("x"); // ["x"]
const b = addToList("y"); // ["y"] — YANGI array, ["x", "y"] emas!
// a !== b

// Default expression ham qayta evaluate bo'ladi
function createId(prefix: string, timestamp: number = Date.now()): string {
  return `${prefix}-${timestamp}`;
}

createId("a"); // yangi Date.now()
createId("b"); // yana yangi Date.now() — a dan keyin
```

Bu xavfli emas, lekin TypeScript/JavaScript'dan boshqa tilga kelgan developer'lar uchun kutilmagan.

### 4. Callback `void` Return Type Funksiya Declaration'da Xato

Kompilator void return type'ni callback context'da "ignore" qiladi, lekin function declaration'da esa **taqiqlaydi** — asimmetrik xatti-harakat.

```typescript
// ❌ Function declaration da return taqiqlanadi
function doSomething(): void {
  // return 42; // ❌ Error: 'number' not assignable to 'void'
}

// ✅ Callback type da return ruxsat etiladi
type Cb = () => void;
const cb: Cb = () => 42; // ✅ return ignored

// Shu tufayli bu JS pattern ishlaydi:
[1, 2, 3].forEach((n) => arr.push(n));
// push returns number, forEach callback void kutadi
// Agar return taqiqlansa — pattern buziladi
```

**Gotcha:** Callback'ning return qiymatiga **ishonmaslik kerak** — caller uni ignore qiladi. Agar return qiymat kerak bo'lsa — callback type'ni `() => boolean` yoki boshqa aniq type qilish kerak.

### 5. Spread Argument — Tuple vs Array Uzunligi

Funksiya aniq parameter count kutsa, spread argument'i **tuple** bo'lishi kerak (uzunligi aniq). `array` type'i (`T[]`) uzunligi noma'lum, shuning uchun ishlamaydi.

```typescript
function add(a: number, b: number, c: number): number {
  return a + b + c;
}

// ❌ Array — uzunlik noma'lum
const dynamic = [1, 2, 3];
// add(...dynamic); // Error: A spread argument must have a tuple type

// ✅ Tuple — uzunlik aniq
const tuple: [number, number, number] = [1, 2, 3];
add(...tuple); // OK

// ✅ as const — readonly tuple
const constTuple = [1, 2, 3] as const;
add(...constTuple); // OK

// ✅ Yangi funksiya — rest parameter
function addAll(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0);
}
addAll(...dynamic); // OK — rest parameter array qabul qiladi
```

**Yechim:** Tuple type'ni `as const` bilan yoki explicit tuple annotation bilan yarating. Yoki funksiyani rest parameter bilan qayta yozing.

---

## Common Mistakes

### ❌ Xato 1: Implementation signature'ni tashqaridan chaqirish

```typescript
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}

// ❌ Implementation signature tashqi dunyoga ko'rinmaydi
// format(true); // Error: No overload matches this call
```

**✅ To'g'ri usul:**

```typescript
function format(value: string): string;
function format(value: number): string;
function format(value: boolean): string;
function format(value: string | number | boolean): string {
  return String(value);
}
format(true); // ✅ Endi ishlaydi
```

**Nima uchun:** Implementation signature — faqat body yozish uchun. Tashqi chaqiruvchilar uni ko'rmaydi — faqat overload signature'lar ko'rinadi.

---

### ❌ Xato 2: Optional parameter'ni division'da ishlatish

```typescript
function divide(a: number, b?: number): number {
  return a / b!; // ❌ b undefined bo'lishi mumkin — NaN natija
}

divide(10); // NaN — b = undefined, 10 / undefined = NaN
```

**✅ To'g'ri usul:**

```typescript
function divide(a: number, b: number = 1): number {
  return a / b; // ✅ b doim valid number
}

divide(10);    // 10 — b = 1
divide(10, 2); // 5
```

**Nima uchun:** Optional parameter `undefined` bo'lishi mumkin — runtime hisoblash NaN'ga olib keladi. Default parameter esa doim valid qiymat beradi.

---

### ❌ Xato 3: void callback'ning return qiymatiga ishonish

```typescript
type Handler = () => void;

const handlers: Handler[] = [
  () => true,
  () => false,
];

function processHandlers(): boolean {
  // ❌ void callback'ning return qiymatini ishlatish
  return handlers.every((h) => h());
  // h() returns void — JS'da void === undefined
  // undefined falsy — every har doim false qaytaradi
}
```

**✅ To'g'ri usul:**

```typescript
type Handler = () => boolean; // ✅ Aniq return type

const handlers: Handler[] = [
  () => true,
  () => false,
];

function processHandlers(): boolean {
  return handlers.every((h) => h()); // ✅ h() returns boolean
}
```

**Nima uchun:** `void` — "return value'ni ishlatma" degan signal. Agar return value kerak bo'lsa — aniq type yozing (`boolean`, `number`, `Promise<T>`).

---

### ❌ Xato 4: Ortiqcha overload yozish

```typescript
// ❌ Har type uchun alohida overload — keraksiz
function toString(value: string): string;
function toString(value: number): string;
function toString(value: boolean): string;
function toString(value: null): string;
function toString(value: undefined): string;
function toString(value: string | number | boolean | null | undefined): string {
  return String(value);
}
```

**✅ To'g'ri usul:**

```typescript
// ✅ Union type — sodda va toza
function toString(value: string | number | boolean | null | undefined): string {
  return String(value);
}
```

**Nima uchun:** Barcha overload'lar bir xil return type (`string`) qaytaradi — overload kerak emas. Overload faqat return type parametrga qarab **o'zgarganda** kerak.

---

### ❌ Xato 5: Method reference berishda `this` context yo'qolishi

```typescript
class UserService {
  users: string[] = [];

  addUser(name: string): void {
    this.users.push(name);
  }
}

const service = new UserService();

// ❌ Method reference — this yo'qoladi
const names = ["Ali", "Vali"];
// names.forEach(service.addUser);
// Runtime error: Cannot read 'users' of undefined
```

**✅ To'g'ri usullar:**

```typescript
// 1. Arrow function bilan bind
names.forEach((name) => service.addUser(name));

// 2. .bind() method
names.forEach(service.addUser.bind(service));

// 3. Class'da arrow property
class UserServiceFixed {
  users: string[] = [];
  addUser = (name: string): void => {
    this.users.push(name);
  };
}
const fixed = new UserServiceFixed();
names.forEach(fixed.addUser); // ✅ arrow this lexical
```

**Nima uchun:** JavaScript'da method reference (`obj.method`) berilganda `this` **yo'qoladi** — bu JS'ning klassik pitfall'laridan biri. Arrow function property yoki explicit `.bind()` bilan hal qilinadi. TypeScript `this` parameter bilan bu muammoni compile-time'da topadi.

---

## Amaliy Mashqlar

### Mashq 1: Function Type Inference (Oson)

**Savol:** Quyidagi funksiyalarning return type'ini ayting (TypeScript inference orqali):

```typescript
function fn1(x: number) {
  if (x > 0) return "positive";
  if (x < 0) return "negative";
  return 0;
}

function fn2(arr: string[]) {
  return arr.length > 0 ? arr[0] : undefined;
}

function fn3() {
  console.log("hello");
}

function fn4<T>(value: T) {
  return [value, value];
}
```

<details>
<summary>Javob</summary>

```typescript
fn1 → return type: string | number
// "positive", "negative" — string literal'lar, widened string'ga
// 0 — number literal, widened number'ga
// Union: string | number

fn2 → return type: string | undefined
// arr[0] — string, undefined — explicit
// Union: string | undefined

fn3 → return type: void
// Hech qanday return statement yo'q → void

fn4 → return type: T[]
// [value, value] — T[] array
// Generic inference: fn4("hi") → string[], fn4(42) → number[]
```

**Tushuntirish:** TypeScript barcha `return` statement'larning type'larini yig'ib union qiladi. Literal type'lar kengaytiriladi (widened). Generic funksiyada return type generic parameter'ni saqlaydi.

</details>

---

### Mashq 2: Overload vs Union Tanlash (O'rta)

**Savol:** Quyidagi funksiya uchun qaysi yondashuv yaxshi — overload yoki union type? Sababini tushuntiring va ikkala variantni yozing.

```typescript
// Funksiya qabul qiladi:
// - string → uppercase qaytaradi
// - number → formatted string qaytaradi
// - Date → ISO string qaytaradi
```

<details>
<summary>Javob</summary>

**Qaror: Union type yetarli**, chunki **barcha return type'lar bir xil (`string`)**. Overload faqat return type parametrga qarab o'zgarganda kerak.

**Union variant (tavsiya):**

```typescript
function format(value: string | number | Date): string {
  if (typeof value === "string") return value.toUpperCase();
  if (typeof value === "number") return value.toFixed(2);
  return value.toISOString(); // value: Date
}

format("hello");        // "HELLO"
format(3.14159);        // "3.14"
format(new Date());     // "2026-04-11T..."
```

**Overload variant (keraksiz, lekin to'g'ri):**

```typescript
function formatOverloaded(value: string): string;
function formatOverloaded(value: number): string;
function formatOverloaded(value: Date): string;
function formatOverloaded(value: string | number | Date): string {
  if (typeof value === "string") return value.toUpperCase();
  if (typeof value === "number") return value.toFixed(2);
  return value.toISOString();
}
```

**Farq:** Overload variant'i xuddi shu ishni qiladi, lekin kod ko'proq. Union type qisqa va bir xil type safety beradi. Qachon overload kerak: `parseInput(s: string): string[]; parseInput(n: number): number[];` — bu yerda return type parametrga bog'liq, overload kerak.

**Decision tree:**

1. Return type parametrga bog'liqmi? → Overload
2. Parametr kombinatsiyalari farqlimi (masalan, (a) vs (a, b))? → Overload
3. Aks holda → Union type

</details>

---

### Mashq 3: Generic Debounce (Qiyin)

**Savol:** `debounce` funksiyasining type'ini yozing — istalgan funksiyani qabul qilib, debounced versiyasini qaytarsin. Original funksiyaning parametr type'lari saqlanishi kerak.

```typescript
function debounce</* type parameters */>(
  fn: /* type */,
  delay: number
): /* return type */ {
  let timer: ReturnType<typeof setTimeout>;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

<details>
<summary>Javob</summary>

```typescript
function debounce<T extends (...args: any[]) => void>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout>;
  return (...args: Parameters<T>) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Ishlatish — original funksiya type'lari saqlanadi
function searchAPI(query: string, page: number): void {
  console.log(`Searching: ${query}, page: ${page}`);
}

const debouncedSearch = debounce(searchAPI, 300);
debouncedSearch("hello", 1);    // ✅ (query: string, page: number)
// debouncedSearch("hello");    // ❌ Expected 2 arguments
// debouncedSearch(42, 1);      // ❌ number !== string
```

**Tushuntirish:**

- `T extends (...args: any[]) => void` — `T` istalgan funksiya bo'lishi kerak
- `Parameters<T>` — `T`'ning parametr type'larini tuple sifatida oladi (utility type)
- Return type: `(...args: Parameters<T>) => void` — original parametrlar bilan void qaytaruvchi funksiya
- Bu pattern real-world'da `lodash.debounce`, `underscore.debounce` kabi library'larda ishlatiladi

**Muqobil yondashuv** — aniq funksiya signature bilan (kamroq flexible):

```typescript
function debounceSimple<A extends unknown[]>(
  fn: (...args: A) => void,
  delay: number
): (...args: A) => void {
  let timer: ReturnType<typeof setTimeout>;
  return (...args: A) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

Ikkalasi ham ishlaydi, lekin `Parameters<T>` yondashuvi zamonaviyroq va `T`'dan qo'shimcha ma'lumot olish mumkin (return type, this parameter).

</details>

---

### Mashq 4: `this` Parameter va Class Method (Qiyin)

**Savol:** Quyidagi kod nima uchun runtime'da xato beradi? Uchta yechim taklif qiling.

```typescript
class EventEmitter {
  private handlers: (() => void)[] = [];

  on(handler: () => void): void {
    this.handlers.push(handler);
  }

  emit(): void {
    this.handlers.forEach((h) => h());
  }
}

class Button {
  label = "Click me";

  constructor(emitter: EventEmitter) {
    emitter.on(this.handleClick);
  }

  handleClick(): void {
    console.log(this.label); // Runtime error
  }
}
```

<details>
<summary>Javob</summary>

**Muammo:** `this.handleClick` method reference sifatida berilganda `this` context **yo'qoladi**. Callback ichida `this` global yoki `undefined` bo'ladi (strict mode).

**Yechim 1 — Arrow function wrapper (eng oddiy):**

```typescript
class Button {
  label = "Click me";

  constructor(emitter: EventEmitter) {
    emitter.on(() => this.handleClick());
    //         ↑ arrow function lexical this
  }

  handleClick(): void {
    console.log(this.label); // ✅
  }
}
```

**Yechim 2 — `.bind()` (explicit):**

```typescript
class Button {
  label = "Click me";

  constructor(emitter: EventEmitter) {
    emitter.on(this.handleClick.bind(this));
    //                        ↑ bind this context
  }

  handleClick(): void {
    console.log(this.label); // ✅
  }
}
```

**Yechim 3 — Arrow method property (eng toza):**

```typescript
class Button {
  label = "Click me";

  constructor(emitter: EventEmitter) {
    emitter.on(this.handleClick); // ✅ arrow property this lexical
  }

  handleClick = (): void => {
    console.log(this.label); // ✅
  };
}
```

**TypeScript bilan compile-time tekshiruv — `this` parameter:**

```typescript
class Button {
  label = "Click me";

  handleClick(this: Button): void {
    console.log(this.label);
  }
}

const btn = new Button();
// emitter.on(btn.handleClick);
// ❌ Compile Error: 'this' context of type 'void' not assignable to 'Button'
```

**Taqqoslash:**

| Yechim | Memory | Readability | Type safety |
|---|---|---|---|
| Arrow wrapper | Har call yangi funksiya | Yaxshi | OK |
| `.bind()` | Har call yangi funksiya | O'rtacha | OK |
| Arrow property | Har instance yangi funksiya | Toza | Yaxshi |

**Qo'shimcha:** Arrow property method'ni **prototype'ga qo'shmaydi** — har instance'da alohida funksiya saqlanadi. Ko'p instance bo'lsa (1000+), memory iste'moli katta. Regular method shorthand esa prototype'da — bitta funksiya ko'p instance bilan share qilinadi.

</details>

---

### Mashq 5: Variadic Tuple va Function Composition (Qiyin)

**Savol:** `pipe` funksiya yozing — ko'p funksiyalarni birlashtirib, kirish argumenti dan chiqish argumentiga o'zgartiruvchi pipeline yaratsin. Type'lar har qadam'da saqlanib turishi kerak.

```typescript
// Misol ishlatish:
const pipeline = pipe(
  (x: number) => x + 1,
  (x: number) => x * 2,
  (x: number) => x.toString(),
);
pipeline(5); // "12" (((5 + 1) * 2) = 12)
```

<details>
<summary>Javob</summary>

```typescript
// Ikki funksiya uchun oddiy pipe
function pipe2<A, B, C>(
  fn1: (a: A) => B,
  fn2: (b: B) => C
): (a: A) => C {
  return (a) => fn2(fn1(a));
}

// Uch funksiya uchun
function pipe3<A, B, C, D>(
  fn1: (a: A) => B,
  fn2: (b: B) => C,
  fn3: (c: C) => D
): (a: A) => D {
  return (a) => fn3(fn2(fn1(a)));
}

// Overload bilan variable count
function pipeOverloaded<A, B>(f1: (a: A) => B): (a: A) => B;
function pipeOverloaded<A, B, C>(f1: (a: A) => B, f2: (b: B) => C): (a: A) => C;
function pipeOverloaded<A, B, C, D>(
  f1: (a: A) => B,
  f2: (b: B) => C,
  f3: (c: C) => D
): (a: A) => D;
function pipeOverloaded(...fns: Array<(x: unknown) => unknown>): (x: unknown) => unknown {
  return (x) => fns.reduce((acc, fn) => fn(acc), x);
}

// Ishlatish
const pipeline = pipeOverloaded(
  (x: number) => x + 1,
  (x: number) => x * 2,
  (x: number) => x.toString()
);
pipeline(5); // "12"

// Chaining type inference
const stringifyAndUpper = pipeOverloaded(
  (n: number) => n.toString(),
  (s: string) => s.toUpperCase()
);
stringifyAndUpper(42); // "42"
```

**Tushuntirish:**

- Oddiy approach'da overload'lar bilan har argument count uchun alohida signature
- Real library'lar (fp-ts, ramda) variadic tuple types bilan yozadi — 10+ overload qo'shadi
- `A, B, C, D` — generic parameter'lar har qadam'ning input va output type'ini ifodalaydi
- Har funksiya avvalgi funksiyaning output'ini input sifatida oladi

**Variadic tuple approach (TS 4.0+, murakkab):**

```typescript
type PipeFunction<A, B> = (a: A) => B;

type Chain<Fns extends readonly PipeFunction<any, any>[]> =
  Fns extends readonly [PipeFunction<infer A, any>, ...any[]]
    ? (a: A) => Fns extends readonly [...any[], PipeFunction<any, infer R>] ? R : never
    : never;

function pipeVariadic<Fns extends readonly PipeFunction<any, any>[]>(
  ...fns: Fns
): Chain<Fns> {
  return ((x: unknown) => fns.reduce((acc, fn) => fn(acc), x)) as Chain<Fns>;
}
```

Bu pattern — conditional types va inference bilan chuqur advanced. [12-conditional-types.md](12-conditional-types.md) va [13-mapped-types.md]'da batafsil.

</details>

---

## Xulosa

Bu bo'limda TypeScript'da funksiyalar bilan ishlashning barcha jihatlarini o'rgandik:

**Asosiy tushunchalar:**

- **Type annotation'lar** — parametr va return type'larni belgilash, kompilator inference mexanizmi, qachon explicit yozish kerak
- **Optional va default parameter'lar** — `?` va `= value` farqi, widening, har chaqiriqda qayta evaluate
- **Rest parameter'lar** — `...args: T[]`, tuple rest, named tuple elements, variadic tuple types (TS 4.0+)
- **Function overload'lar** — resolution order, implementation signature tashqi ko'rinmasligi, overload vs union qarori
- **Call/Construct signature'lar** — hybrid type, `Object.assign` pattern, class constructor signature
- **`this` parameter** — compile-time `this` binding tekshiruvi, arrow vs method farqi, React-style pattern'lar
- **`void` vs `undefined`** — callback'da void'ning maxsus xatti-harakati, function declaration'da return taqiqi
- **Function type expression'lar** — reusable type, type alias vs interface, generic function type
- **Generic funksiyalar (intro)** — type parameter, inference, constraint, `keyof` bilan property access
- **Callback type'lar** — contextual typing, Node.js style, Promise callback, higher-order function
- **Function assignability** — variance qoidalari, covariance/contravariance, input/output positions, method shorthand bivariance

**Variance intuitiv tushunish:**

- **Output (return type)** — tabi yo'nalish (covariance): `Dog → Animal` tabiy, Dog return Animal expected joyga mos
- **Input (parameter)** — teskari yo'nalish (contravariance): `Animal → Dog` teskari, keng input tor joyga mos
- **Method shorthand** — bivariance (ikkala yo'nalish) backward compatibility uchun

**Umumiy takeaway'lar:**

1. **Kompilator inference'ga ishonish** — oddiy funksiyalarda return type yozmaslik. Faqat public API, recursive funksiya, va murakkab return uchun explicit yozish.
2. **Union vs overload qarori** — qachon union yetarli, qachon overload kerak: return type parametrga bog'liqmi?
3. **Arrow vs method** — class'da `this` muammosini oldini olish uchun arrow property yoki `bind` ishlatish.
4. **Callback type safety** — contextual typing'dan foydalanish, inline callback'larda parametr type yozmaslik.
5. **Variance intuitiv** — input contravariant, output covariant. `strictFunctionTypes: true` bilan type safety yaxshiroq.
6. **`this` parameter** — class detachment bug'larini compile-time'da topish.

**Cross-references:**

- **[06-type-narrowing.md](06-type-narrowing.md)** — funksiya ichidagi return type narrowing, `typeof fn === "function"` narrowing
- **[08-generics.md](08-generics.md)** — generic function'lar chuqur, `keyof`, `typeof` type operator, mapped type bilan generic
- **[10-classes.md](10-classes.md)** — class'da `this` binding, arrow method property, constructor signature
- **[25-type-compatibility.md](25-type-compatibility.md)** — variance chuqur, structural typing, strictFunctionTypes ichki ishlashi

---

**Keyingi bo'lim:** [08-generics.md](08-generics.md) — Generics chuqur: type parameter'lar, constraint'lar (`extends`, `keyof`), index access types (`T[K]`), `typeof` type operator, multiple type parameter'lar, default va `const` type parameter'lar, generic utility pattern'lari.
