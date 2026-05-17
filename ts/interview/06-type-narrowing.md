# Interview: Type Narrowing

> Type narrowing, control flow analysis, truthiness narrowing, equality narrowing, `typeof`/`instanceof`/`in` guards, assignment narrowing, non-null assertion (`!`), user-defined type guards (`is`), assertion functions (`asserts`), `satisfies` operator, closure narrowing limitations bo'yicha interview savollari.

---

## Mundarija

**Nazariy savollar**
- Savol 1: Type narrowing va control flow analysis `[Junior+]`
- Savol 2: Truthiness narrowing va falsy traps `[Middle]`
- Savol 3: Equality narrowing — `==` va `===` `[Middle]`
- Savol 4: `typeof` operator va narrowing `[Middle]`
- Savol 5: User-defined guard (`is`) vs assertion (`asserts`) `[Middle+]`
- Savol 6: `satisfies` operator vs `:` annotation `[Middle+]`
- Savol 7: Closure ichida narrowing yo'qolishi `[Senior]`

**Output savollar**
- Savol 8: Control flow narrowing — har nuqtada tip `[Junior+]`
- Savol 9: `typeof null` xavfli pattern `[Middle]`

**Coding savollar**
- Savol 10: `satisfies` bilan type-safe routing config `[Middle+]`
- Savol 11: Custom assertion function — API validator `[Middle+]`
- Savol 12: Type-safe event emitter type guard'lar bilan `[Senior]`

**Bug fix savollar**
- Savol 13: Narrowing cheklovlari — 3 ta muammo va yechim `[Senior]`
- Savol 14: Non-null assertion (`!`) ortiqcha ishlatish `[Middle]`

---

## Nazariy savollar

### Savol 1: Type narrowing nima? Control flow analysis qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type narrowing — TypeScript compiler runtime tekshiruvlarni (`typeof`, `instanceof`, `in`, equality) tahlil qilib, union type'ni har code branch'da aniqroq tipga toraytirishi. Control flow analysis — kompilyator kodni yuqoridan pastga oqim grafi bo'yicha tahlil qilib, har nuqtada o'zgaruvchi tipini track qilishi.

### To'liq tushuntirish

Union type'da faqat barcha member'larga umumiy operatsiyalar ishlaydi. Specific property yoki method uchun avval narrowing qilish kerak — kompilyatorga "shu branch'da tip aniqroq" deb signal berish. TypeScript bu signallarni `if`, `switch`, `?:`, `&&`, `||`, `return`, `throw`, `break` orqali oladi.

**Control flow analysis bosqichlari:**

1. Kompilyator parsing'dan keyin syntax tree quradi.
2. Har scope uchun flow graph (block, branch, loop) yaratiladi.
3. Har node'da `getFlowTypeOfReference` o'zgaruvchining hozirgi tipini hisoblaydi.
4. Branching ifoda (`if`, `switch`) `narrowType*` funksiyalari orqali tipni eliminate qiladi.
5. Branch tugagandan keyin (return/throw) qolgan tip continuation'da saqlanadi.

### Kod misol

```typescript
function format(value: string | number): string {
  // value: string | number
  // value.toUpperCase(); // ❌ number'da yo'q

  if (typeof value === "string") {
    return value.toUpperCase(); // value: string — narrowing
  }
  return value.toFixed(2);      // value: number — string olib tashlandi
}

// Multi-step narrowing
function process(x: string | number | null | undefined): void {
  // x: string | number | null | undefined

  if (x == null) {
    return;                     // null va undefined olib tashlandi
  }
  // x: string | number

  if (typeof x === "string") {
    x.toUpperCase();            // x: string
    return;
  }
  // x: number — string ham olib tashlandi

  x.toFixed(2);                 // x: number
}

// `&&` short-circuit bilan narrowing
function getLengthBad(value: string | null): number {
  return (value && value.length) || 0;
  // value && — null olib tashlanadi
  // "" ham falsy — 0 qaytaradi (xatti-harakat)
}

// `??` (nullish coalescing) — faqat null/undefined
function getLengthGood(value: string | null): number {
  return value?.length ?? 0;
}
```

### Edge Cases

- **`==` vs `===`**: `== null` ikkala `null` va `undefined` ni oladi, `=== null` faqat `null`.
- **Falsy values**: `0`, `""`, `false` truthy check'da olib tashlanadi — `null/undefined` ga e'tibor.
- **Block scope**: `if` ichida `const` bilan reassign yo'q — narrowing saqlanadi.
- **Function call'dan keyin**: object property narrowing yo'qoladi (mutation ehtimoli).
- **Loop ichida**: narrowing har iteratsiyada qayta hisoblanadi.

### Follow-up savollar

1. **"Nima uchun union type bilan to'g'ridan-to'g'ri ishlash mumkin emas?"** — Union qiymat har bir member'ning property/method'iga ega bo'lmaydi — faqat umumiy interface'ga.
2. **"Narrowing nominal type system'larda qanday?"** — Java, C# da `instanceof` bor, lekin TypeScript'dagi `typeof`, `in`, equality narrowing'i — flow-sensitive typing.
3. **"`assertNever` narrowing'da qaerda ishlatiladi?"** — Exhaustive check — barcha case handle qilingandan keyin `never` deb ishlatish.

</details>

---

### Savol 2: Truthiness narrowing qanday ishlaydi? Falsy qiymatlar bilan xatosi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Truthiness narrowing — `if (value)` orqali `null`, `undefined`, `0`, `""`, `false`, `NaN` qiymatlarini olib tashlash. Xato — string yoki number tipida `0`/`""` ham truthy check'dan o'tmaydi — bu valid qiymatlarni yo'qotadi.

### To'liq tushuntirish

JavaScript'da har value `Boolean(value)` orqali truthy yoki falsy ga konvert bo'ladi. Falsy qiymatlar: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Qolgan barcha qiymatlar truthy (jumladan `"false"`, `[]`, `{}`, `0n` dan tashqari bigint, function).

Truthiness narrowing'ning xatosi: `string | null` tipida `if (value)` `null`'ni olib tashlaydi, lekin `""` (bo'sh string) ham truthy emas — agar `""` valid qiymat bo'lsa, false-positive yo'qotish. Xuddi shunday `number | null` da `0` valid bo'lsa — yo'qoladi.

**To'g'ri pattern**: aniq tekshiruv (`value !== null`, `value !== undefined`) yoki `?.`/`??` operatorlari.

### Kod misol

```typescript
// ❌ Truthiness — silent bug
function getLengthBad(s: string | null | undefined): number {
  if (s) {
    return s.length;
  }
  return 0;
  // Problem: "" valid string, lekin truthy emas — 0 qaytadi
  // Aslida "" length 0, lekin pattern bug bilan ham 0 — silent fail
}

// ❌ Number'da yanada xavfli
function isValid(n: number | null): boolean {
  if (n) {                  // ❌ 0 ham null kabi handle qilinadi
    return n > 0;
  }
  return false;
}
console.log(isValid(0));    // false — lekin 0 valid number
console.log(isValid(null)); // false — to'g'ri

// ✅ Aniq tekshiruv
function getLengthGood(s: string | null | undefined): number {
  if (s !== null && s !== undefined) {
    return s.length;  // "" ham handle qilinadi (length 0)
  }
  return 0;
}

// ✅ Optional chaining + nullish coalescing
function getLengthBest(s: string | null | undefined): number {
  return s?.length ?? 0;
}

// ✅ Number uchun — aniq tekshiruv
function isValidNumber(n: number | null): boolean {
  return n !== null && n > 0;
}

// Truthiness object union'da
type User = { name: string };
type Empty = null;

function greet(user: User | Empty): string {
  if (user) {       // ✅ object truthy, null falsy
    return user.name;
  }
  return "Anonim";
}

// Falsy qiymatlar to'liq ro'yxat
type Falsy = false | 0 | -0 | 0n | "" | null | undefined; // NaN ham, lekin tipi number
```

### Edge Cases

- **`""` falsy**: bo'sh string ko'pchilik form input'larda valid qiymat — truthiness yo'qotadi.
- **`0` falsy**: counter, index, score kabi field'larda 0 valid — truthiness yo'qotadi.
- **`NaN` falsy**: arithmetic xato natijasi, lekin truthiness'da bilinmaydi — alohida `Number.isNaN`.
- **Empty array/object truthy**: `[]`, `{}` truthy — `if (arr)` arr mavjudligini tekshiradi, length emas.
- **`Boolean()` constructor**: `Boolean(value)` truthiness'ni explicit ko'rsatadi.

### Follow-up savollar

1. **"`??` va `||` farqi qachon muhim?"** — `||` har falsy ga default beradi, `??` faqat `null/undefined` ga. `value ?? "default"` — 0 va "" saqlaydi.
2. **"ESLint qoidasi bormi?"** — `@typescript-eslint/strict-boolean-expressions` — implicit truthy/falsy check'larni majbur qilib aniq tekshiruvga o'tkazadi.
3. **"`Boolean(value)` `!!value` bilan farq nima?"** — Bir xil natija. `Boolean()` aniqroq o'qiladi, `!!` keng tarqalgan idiom.

</details>

---

### Savol 3: Equality narrowing qanday ishlaydi? `==` va `===` farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Equality narrowing — `===`, `!==`, `==`, `!=` orqali aniq qiymat bo'yicha narrowing. `===` strict equality — tip va qiymat. `== null` maxsus — `null` va `undefined` ikkalasini tekshiradi (TypeScript buni biladi).

### To'liq tushuntirish

`===` strict equality — tip konvertatsiyasi yo'q. `==` loose equality — JavaScript abstrak operatsiyasi `==` qoidalari bo'yicha tip konvertatsiya qiladi. TypeScript ko'pgina holatlarda `===` ni majbur qiladi (`@typescript-eslint/eqeqeq`).

**Maxsus holat — `== null`**: JavaScript `==` qoidalari bo'yicha `null == undefined === true`, qolgan tip'lar bilan `false`. TypeScript bu pattern'ni alohida biladi — `value == null` `null | undefined` bo'lsa, ikkalasini birga olib tashlaydi (yoki saqlaydi, branch'ga qarab).

Discriminated union'da equality narrowing fundamental: `switch (value.kind)` har case'da literal qiymat tekshiruvi — `kind === "circle"` narrow qiladi.

### Kod misol

```typescript
// 1. Strict equality — literal narrowing
type Status = "active" | "inactive" | "banned";

function getColor(status: Status): string {
  if (status === "active")   return "green";
  if (status === "inactive") return "gray";
  if (status === "banned")   return "red";
  // status: never — exhaustive
  return assertNever(status);
}

function assertNever(value: never): never {
  throw new Error(`Unhandled: ${value}`);
}

// 2. `== null` — null va undefined birga
function process(value: string | number | null | undefined): void {
  if (value == null) {
    return; // null va undefined olib tashlandi
  }
  // value: string | number
  console.log(value);
}

// `=== null` — faqat null
function processStrict(value: string | null | undefined): void {
  if (value === null) {
    return; // faqat null
  }
  // value: string | undefined — undefined hali bor
}

// 3. Inequality narrowing
type Color = "red" | "green" | "blue";

function isPrimary(color: Color): boolean {
  return color !== "blue"; // red yoki green
}

// 4. Switch equality
function area(shape: { kind: "circle"; r: number } | { kind: "square"; s: number }): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.r ** 2;
    case "square": return shape.s ** 2;
  }
}

// 5. Literal type narrowing
function check(value: 1 | 2 | 3): string {
  if (value === 1) return "one";
  if (value === 2) return "two";
  return "three";
}

// 6. `==` xavfli misol
function badCheck(value: unknown): boolean {
  return value == 0;  // "0", false, 0 — barchasi true qaytaradi
}
console.log(badCheck("0"));    // true (string "0" == 0)
console.log(badCheck(false));  // true (false == 0)
console.log(badCheck(""));     // true ("" == 0)
```

### Edge Cases

- **`NaN === NaN`**: false! `Number.isNaN(value)` ishlatish kerak.
- **Object equality**: `{} === {}` false — reference comparison.
- **`-0 === +0`**: true, lekin `Object.is(-0, +0)` false.
- **Symbol equality**: unique — `Symbol("x") !== Symbol("x")`.
- **String literal vs string**: `"x" === value` value tipi `"x"` ga narrow bo'ladi (literal).

### Follow-up savollar

1. **"`==` qachon ishlatish mumkin?"** — Faqat `value == null` patternda. Qolgan barcha holatlar `===`.
2. **"`Object.is` `===` dan qanday farq qiladi?"** — `Object.is(NaN, NaN)` true, `Object.is(-0, +0)` false. Math operatsiyalarida foydali.
3. **"Discriminated union'da `===` literal bilan narrowing qanday?"** — `value.kind === "circle"` literal tip'ni member'ga match qiladi — TypeScript shu member tipini saqlaydi.

</details>

---

### Savol 4: `typeof` operator qaysi qiymatlarni qaytaradi? TypeScript qanday narrow qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`typeof` 8 ta natija qaytaradi: `"string"`, `"number"`, `"bigint"`, `"boolean"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`. TypeScript har birini tegishli primitive tipga narrow qiladi. `typeof null === "object"` — JavaScript bug, alohida `=== null` tekshiruvi kerak.

### To'liq tushuntirish

JavaScript'dagi `typeof` operator — value'ning eng asosiy tip yorlig'ini qaytaradi. Bu primitive tip'larni ajratish uchun yaratilgan, lekin `null` JavaScript 1.0'dan beri `"object"` qaytaradi (tarixiy bug, backward compatibility uchun tuzatilmaydi).

TypeScript narrowing semantikasi:
- `typeof === "string"` → `string`
- `typeof === "number"` → `number`
- `typeof === "boolean"` → `boolean`
- `typeof === "object"` → object tip'lar **va `null`** (TS biladi, alohida tekshirish kerak)
- `typeof === "function"` → callable (function, class constructor)

| `typeof` natijasi | Qaysi qiymat | TS narrowing |
|---|---|---|
| `"string"` | string primitive | ✅ string |
| `"number"` | number, NaN, Infinity | ✅ number |
| `"bigint"` | bigint primitive | ✅ bigint |
| `"boolean"` | true, false | ✅ boolean |
| `"symbol"` | symbol | ✅ symbol |
| `"undefined"` | undefined | ✅ undefined |
| `"object"` | object, array, **null** | ⚠️ null ham kiradi |
| `"function"` | function, class | ✅ Function |

### Kod misol

```typescript
function describe(value: unknown): string {
  if (typeof value === "string") {
    return value.toUpperCase();           // value: string
  }
  if (typeof value === "number") {
    return value.toFixed(2);              // value: number
  }
  if (typeof value === "boolean") {
    return value ? "yes" : "no";          // value: boolean
  }
  if (typeof value === "function") {
    return value.name;                    // value: Function
  }
  if (typeof value === "object" && value !== null) {
    return JSON.stringify(value);         // value: object (null safe)
  }
  return String(value);
}

// `typeof null` muammosi
function processObject(value: object | null): void {
  // ❌ Xavfli
  // if (typeof value === "object") {
  //   value.toString(); // value: object | null — null'ga toString crash
  // }

  // ✅ Xavfsiz
  if (value !== null && typeof value === "object") {
    value.toString();  // value: object
  }
}

// `typeof` array bilan
function lengthOf(value: string | number[]): number {
  if (typeof value === "string") {
    return value.length;
  }
  return value.length;
  // Array'da typeof === "object" — alohida narrowing yo'q
  // Lekin union member'lari farq qilgani uchun TS biladi
}

// `Array.isArray` bilan aniq tekshiruv
function flatten(value: number | number[]): number[] {
  if (Array.isArray(value)) {
    return value;       // value: number[]
  }
  return [value];       // value: number
}

// `typeof` function bilan
type Callback = () => void;
type Value = number;

function execute(input: Callback | Value): void {
  if (typeof input === "function") {
    input();            // input: Callback
  } else {
    console.log(input); // input: Value
  }
}
```

### Edge Cases

- **`typeof null === "object"`**: JavaScript 1.0 bug — backward compatibility uchun tuzatilmaydi.
- **`typeof undeclared`**: ReferenceError'siz `"undefined"` qaytaradi (security probing).
- **Array `typeof`**: `"object"` — `Array.isArray()` aniq tekshiruv.
- **Class instance**: `"object"` — `instanceof` aniq tekshiruv.
- **`typeof Symbol`**: yangi qiymat, ES6'da qo'shilgan.

### Follow-up savollar

1. **"`typeof null === "object"` nima uchun tuzatilmaydi?"** — Eski JavaScript kodlar bunga bog'liq — fix backward incompatible bo'lar edi (TC39 reject qilgan).
2. **"`typeof` runtime vs `:` annotation qanday farq qiladi?"** — `typeof` runtime operator, `: typeof variable` — TypeScript type query operator (compile-time).
3. **"BigInt qachon qo'shilgan?"** — ES2020. `typeof 1n === "bigint"`.

</details>

---

### Savol 5: User-defined type guard (`is`) va assertion function (`asserts`) farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type guard `value is T` — boolean qaytaradi, `if (isX(val))` branch'ida narrow qiladi. Assertion `asserts value is T` — void qaytaradi, throw qilmasa keyingi kodda narrow qiladi. Guard — branching, assertion — preconditions.

### To'liq tushuntirish

Ikkalasi ham TypeScript'ga "value tipi shunday" deb signal beradi, lekin chaqirilish patterni va semantikasi farq qiladi:

**Type Guard (`is`)**:
- Return tipi `boolean` + `value is T` predicate.
- `if (isX(value))` ichida true branch'da narrow.
- `else` branch'da type olib tashlanadi (negation narrowing).
- Use case: filtering, branching logic.

**Assertion Function (`asserts`)**:
- Return tipi `void`.
- Body throw qiladi yoki muvaffaqiyatli return.
- Throw bo'lmasa — assertion'dan keyin narrow.
- Use case: precondition, input validation.

### Kod misol

```typescript
// 1. Type guard — boolean predicate
function isString(value: unknown): value is string {
  return typeof value === "string";
}

const input: unknown = getUserInput();
if (isString(input)) {
  console.log(input.toUpperCase()); // ✅ input: string
} else {
  // input: unknown - string olib tashlangan
}

// 2. Assertion function — throw qiladi
function assertString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}

const input2: unknown = getUserInput();
assertString(input2);
input2.toUpperCase(); // ✅ Keyingi kod — input2: string

// 3. Custom assertNever
function assertNever(value: never): never {
  throw new Error(`Unexpected: ${value}`);
}

// 4. `asserts` condition variant
function assert(condition: unknown, message?: string): asserts condition {
  if (!condition) throw new Error(message ?? "Assertion failed");
}

function processUser(user: User | null) {
  assert(user !== null, "User required");
  console.log(user.name); // ✅ user: User
}

// 5. Filtering bilan type guard
function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

const mixed: (string | null)[] = ["a", null, "b"];
const filtered: string[] = mixed.filter(isNonNullable);

// 6. Object struktura tekshiruv
interface User {
  id: number;
  name: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as Record<string, unknown>).id === "number" &&
    typeof (value as Record<string, unknown>).name === "string"
  );
}

// Class hierarchy
class AppError extends Error {
  constructor(public code: number, message: string) {
    super(message);
  }
}

function isAppError(error: unknown): error is AppError {
  return error instanceof AppError;
}

try {
  // ...
} catch (error: unknown) {
  if (isAppError(error)) {
    console.log(error.code); // ✅ error: AppError
  }
}
```

### Edge Cases

- **Body logic xato**: TS predicate'ni body bilan compare qilmaydi — guard noto'g'ri yozilsa runtime crash.
- **`asserts value` (predicate'siz)**: faqat condition truthy ekanligini ta'minlaydi (`asserts condition`).
- **`asserts` va arrow function**: arrow function'da `asserts` ishlaydi, lekin syntax: `(x: T): asserts x is U => {...}`.
- **`asserts` class method'da**: `this is T` predicate ham mumkin.
- **`is` `else` branch**: negation narrowing — `if (!isX(value))` branch'da type olib tashlanadi.

### Follow-up savollar

1. **"Generic type guard qachon ishlatiladi?"** — `isNonNullable<T>` kabi utility — har tip uchun.
2. **"`asserts` Node.js `assert` module bilan bog'liqmi?"** — Pattern o'xshash, lekin TypeScript `asserts` faqat type narrowing — runtime tekshiruvini siz yozasiz.
3. **"`zod` schema qanday ishlatadi?"** — Schema'ni `parse` qiladi (throws) yoki `safeParse` (Result tipida qaytaradi). TypeScript guard'larni avtomatik infer qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Type predicate ABI**: TypeScript compiler `is` predicate'ni `getCheckFlags` orqali track qiladi. `narrowTypeByCallExpression` chaqiriq natijasini analyze qilib, argumentni narrow qiladi. `asserts` esa `narrowTypeByAssertion` orqali — return statement'dan keyingi flow'da type'ni update qiladi.

**`asserts` vs `throw`**: `asserts` body to'g'ridan-to'g'ri throw qilishi shart. `return` qilsa — predicate ishlamaydi (TypeScript trust qiladi, runtime'da assertion ishlamasligi mumkin).

**Predicate variance**: predicate parameter contravariant — function argument tipi narrowing chaqiriqlarida muhim. Generic guard'lar `<T>` bilan har tip uchun yagona implementatsiya.

</details>

</details>

---

### Savol 6: `satisfies` operator qanday ishlaydi? `:` annotation farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`satisfies` (TS 4.9+) — value target tipga mosligini tekshiradi, lekin **inferred** tipni saqlab qoladi. `:` annotation esa inferred tipni target'ga widen qiladi — aniq tip yo'qoladi.

### To'liq tushuntirish

TypeScript'da type system ikki yo'nalishda ishlaydi:

1. **Inferred type** — value'dan kelib chiqib eng tor tip (literal'larni saqlash).
2. **Declared type** (annotation) — target tip (widen qilingan).

`:` annotation declared type'ni majbur qiladi — inferred type yo'qoladi. `satisfies` esa "value declared'ga mos kelishini tekshir, lekin inferred'ni qaytar" semantikasi bilan ishlaydi.

Bu farq:
- Config object'larida (literal property'larni saqlash kerak).
- Theme/route definition'larida (aniq enum-like qiymatlar).
- Tagged template literal'larida (const pattern'lar).

### Kod misol

```typescript
type Config = Record<string, string | number>;

// 1. `:` annotation — widen
const config1: Config = { port: 3000, host: "localhost" };
config1.port;     // string | number — aniq tip yo'qoldi
config1.invalid;  // string | number — har string ruxsat (Record open)

// 2. `satisfies` — saqlash
const config2 = { port: 3000, host: "localhost" } satisfies Config;
config2.port;     // number — aniq
config2.host;     // string — aniq
// config2.invalid; // ❌ Property 'invalid' does not exist
// Shape closed qoldi — faqat declared property'lar

// 3. `as const` + `satisfies` — eng aniq
const routes = {
  home: { path: "/",    method: "GET" },
  api:  { path: "/api", method: "POST" },
} as const satisfies Record<string, { path: string; method: string }>;

routes.home.path;    // "/" — literal
routes.api.method;   // "POST" — literal

// 4. Union discriminated bilan
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "RESET" };

const inc = { type: "INCREMENT", amount: 5 } satisfies Action;
inc.type;     // "INCREMENT" — aniq
inc.amount;   // number

const reset: Action = { type: "RESET" };
reset.type;   // "INCREMENT" | "RESET" — widen

// 5. Function return — refine
function buildRoutes() {
  return {
    users: "/users",
    posts: "/posts",
    health: "/health",
  } satisfies Record<string, string>;
}
const r = buildRoutes();
r.users; // string — type narrow

// 6. Validation + autocomplete
type Color = "red" | "green" | "blue";

const palette = {
  primary: "red",
  secondary: "green",
  // accent: "purple", // ❌ "purple" Color'da yo'q (satisfies tekshirdi)
} satisfies Record<string, Color>;

palette.primary; // "red" — literal
```

### Edge Cases

- **`as` vs `satisfies`**: `as` unsafe assertion (tekshirmaydi), `satisfies` type-safe validation.
- **Excess property check**: `satisfies` annotation kabi excess property'larni ushlaydi.
- **Function position**: `(x) => x satisfies T` — arrow function'da ham ishlaydi.
- **Generic constraint**: generic parametr sifatida emas, lekin generic value bilan.
- **Performance**: bir marta tekshiriladi, runtime'da yo'q.

### Follow-up savollar

1. **"Qachon `:` annotation aniq kerak?"** — Variable widen tip kerak bo'lganda (function param sifatida boshqa joyda).
2. **"`as const satisfies` qanday ishlaydi?"** — Operator order: avval `as const` (literal saqlaydi), keyin `satisfies` (validation). Eng tor + validated.
3. **"`satisfies` performance'ga ta'sir qiladimi?"** — Compile-time'da bir marta. JavaScript output'da yo'q.

</details>

---

### Savol 7: Closure ichida narrowing nima uchun yo'qoladi? Qanday saqlash mumkin? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`let` bilan e'lon qilingan o'zgaruvchi narrowing'dan keyin callback (closure) ichida ishlatilsa — TypeScript narrowing'ni yo'qotadi, chunki callback qachon chaqirilishi noma'lum (variable o'zgargan bo'lishi mumkin). TS 5.4+ `let` reassign bo'lmasa narrowing'ni saqlaydi. Yechim: `const` ga capture.

### To'liq tushuntirish

TypeScript control flow analysis intra-procedural — bir funksiya ichida ishlaydi. Callback'da o'zgaruvchi ishlatilsa, compiler:
1. Callback qachon chaqirilishini bilmaydi (sync, async, never).
2. Callback chaqirilguncha variable reassign bo'lishi mumkin.
3. Xavfsiz yo'l — variable'ni declared tipga qaytarish (widen).

Bu cheklov "false positive" emas — haqiqiy bug oldini oladi. Misol: narrow'dan keyin `value = 42`, setTimeout esa keyin chaqiriladi va eski narrow'ga ishonib crash.

TS 5.4+ flow-sensitive `let` analysis — agar `let` narrowing'dan keyin reassign qilinmasa, closure'da ham narrow saqlanadi. Lekin reassign bo'lsa — yo'qoladi.

Yechimlar:
1. **`const` ga capture** — universal va eski TS'larda ishlaydi.
2. **Function parameter** — explicit pass.
3. **TS 5.4+** — automatic (faqat reassign bo'lmasa).

### Kod misol

```typescript
// Muammo
function example1(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    // TS 5.3-: value: string — narrowing
    setTimeout(() => {
      // TS 5.3-: value: string | number — yo'qoldi
      // TS 5.4+: value: string (agar reassign bo'lmasa)
      value.toUpperCase(); // ❌ TS 5.3- da error
    }, 100);

    value = 42; // ← Reassign — TS 5.4+ ham yo'qotadi
  }
}

// Yechim 1: const ga capture
function example2(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    const captured = value; // const — immutable binding, narrow saqlanadi
    setTimeout(() => {
      captured.toUpperCase(); // ✅ captured: string
    }, 100);
    value = 42;
  }
}

// Yechim 2: parameter sifatida pass
function example3(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    runAsync(value);
    value = 42;
  }
}

function runAsync(s: string): void {
  setTimeout(() => {
    s.toUpperCase(); // ✅ s: string (parameter)
  }, 100);
}

// Muammo: object property mutation
function example4(obj: { name: string | null }): void {
  if (obj.name !== null) {
    externalCall(obj);
    // obj.name narrowed? — TS BILMAYDI
    // externalCall mutate qilgan bo'lishi mumkin
    obj.name.toUpperCase(); // ❌ Possibly null
  }
}

declare function externalCall(o: object): void;

// Yechim: local const
function example5(obj: { name: string | null }): void {
  const name = obj.name; // local snapshot
  if (name !== null) {
    externalCall(obj);
    name.toUpperCase(); // ✅ name: string (local, immutable)
  }
}

// Aliased conditions (TS 4.4+)
function example6(x: string | number): void {
  const isString = typeof x === "string";
  if (isString) {
    x.toUpperCase(); // ✅ TS 4.4+ — aliased predicate ishlaydi
  }
}
```

### Edge Cases

- **TS 5.4+ improvement**: `let` reassign bo'lmasa closure'da narrowing saqlanadi.
- **`const` bilan har doim**: const reassign mumkin emas — narrowing har doim saqlanadi.
- **Method chain**: `obj.x.y` chain'idagi har segment alohida narrow (`obj`, `obj.x`, `obj.x.y`).
- **`this` narrowing**: class method'da `this` narrowing — `this is T` predicate'lar bilan.
- **Loop closures**: loop ichida narrow har iteratsiyada qayta hisoblanadi.

### Follow-up savollar

1. **"TS 5.4 qachon release bo'lgan va nima yaxshilanibdi?"** — 2024-03. `let` reassign bo'lmasa flow'da narrowing saqlanishi, IIFE va callback'larda yaxshi.
2. **"Object property narrowing nima uchun saqlanmaydi function call'dan keyin?"** — Function reference orqali mutation qilishi mumkin — TypeScript conservative.
3. **"`readonly` modifier bilan saqlanadimi?"** — Ha, `readonly` property narrow qilingach saqlanadi (mutation imkonsiz).

<details>
<summary><strong>Deep Dive</strong></summary>

**Intra-procedural cheklov**: TypeScript inter-procedural analysis qilmaydi (function chaqiruvlar orasidagi side effect'lar). Bu compiler performance va undecidability sababli. Compiler `getEffectiveTypeRoots` orqali function boundary'larda type'ni reset qiladi.

**TS 5.4 mexanizmi**: `getNarrowedTypeWorker` ichida `lastAssignmentLeavesNarrowedType` — agar `let` reassign bo'lmasa, closure context'da ham narrowing inherit qilinadi. Reassign bo'lsa — `getUnionType` declared tip bilan join qilinadi.

**Use site narrowing**: TS narrowing call site'da ishlaydi — declaration site'da emas. Bu `useDefineForClassFields` va `definite assignment` analysis'da murakkablashadi.

</details>

</details>

---

## Output savollari

### Savol 8: Control flow narrowing — har nuqtada tip [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har `if` va `return` branch'idan keyin tip eliminate bo'ladi. `== null` ikkala null va undefined ni oladi.

### To'liq tushuntirish

TypeScript control flow analysis kodni yuqoridan pastga o'qib, har nuqtada o'zgaruvchining hozirgi tipini hisoblaydi. `if (condition)` branch'ida — narrowing qo'llanadi, `return`/`throw` esa shu branch'dagi tipni keyingi kod oqimidan butunlay olib tashlaydi. `==` operator JavaScript loose equality qoidalariga binoan `null == undefined === true` — TypeScript bu pattern'ni alohida biladi va `value == null` orqali ikkala tipni birga eliminate qiladi. Negation operatorlari (`!==`, `typeof !==`) — narrowing'da olib tashlanadigan tipni belgilaydi, qolgan member'lar saqlanadi.

### Kod misol

```typescript
function mystery(value: string | number | boolean | null | undefined): void {
  console.log(value);  // A: string | number | boolean | null | undefined

  if (value == null) return;
  console.log(value);  // B: string | number | boolean — null va undefined olib tashlandi

  if (typeof value === "boolean") {
    console.log(value); // C: boolean
    return;
  }
  console.log(value);  // D: string | number — boolean olib tashlandi (return)

  if (typeof value !== "number") {
    console.log(value); // E: string — number olib tashlandi
  }
}
```

**Tushuntirish**:

| Nuqta | Tip | Sabab |
|---|---|---|
| A | `string \| number \| boolean \| null \| undefined` | hali narrowing yo'q |
| B | `string \| number \| boolean` | `== null` ikkalasini olib tashladi |
| C | `boolean` | `typeof === "boolean"` |
| D | `string \| number` | boolean branch return qildi |
| E | `string` | `typeof !== "number"` negation |

### Edge Cases

- **`== null` vs `=== null`**: birinchi ikkalasini, ikkinchi faqat null.
- **`return` narrowing**: return'dan keyin shu branch tipi olib tashlanadi.
- **Negation `!==`**: tip olib tashlanadi (rest qoladi).

### Follow-up savollar

1. **"`==` qachon `===` ga teng emas?"** — `null == undefined` true. `null === undefined` false.
2. **"Boolean narrowing'da `else if` farqi bormi?"** — Yo'q, `if` chain bilan bir xil. Order muhim.

</details>

---

### Savol 9: `typeof null` xavfli pattern [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`typeof null === "object"` JavaScript bug — string|null tipida ishlaydi, lekin union'ga object qo'shilsa silent fail. `=== null` bilan aniq tekshirish kerak.

### To'liq tushuntirish

JavaScript 1.0 (1995)'dan beri `typeof null` `"object"` qaytaradi — bu original implementation bug, lekin backward compatibility tufayli TC39 fix'ni rad etgan. `string | null` union'da `typeof === "object"` faqat `null`'ga match qiladi (string emas), shuning uchun ishlaydi. Lekin union'ga `object` qo'shilganda — `typeof === "object"` ikkala `null` va `object`'ni match qiladi — narrowing'da `null` qoldiriladi va keyingi method chaqiruv runtime'da `TypeError: Cannot read properties of null` beradi. To'g'ri pattern: `value === null` bilan explicit tekshirish yoki optional chaining (`value?.length`).

### Kod misol

```typescript
// ❌ Original — xavfli pattern
function getLength(value: string | null): number {
  if (typeof value === "object") {
    return 0; // null holat
  }
  return value.length;
}

// Hozir ishlaydi, lekin tip kengaytirilsa buziladi
function getLength2(value: string | object | null): number {
  if (typeof value === "object") {
    return 0; // null + object ikkalasi — bug
  }
  return value.length;
}

// ✅ Yechim 1 — aniq null check
function getLengthSafe(value: string | null): number {
  if (value === null) return 0;
  return value.length;
}

// ✅ Yechim 2 — optional chaining
function getLengthShort(value: string | null): number {
  return value?.length ?? 0;
}

// ✅ Yechim 3 — kengaytirilgan union ham xavfsiz
function getLengthExtended(value: string | object | null): number {
  if (value === null) return 0;
  if (typeof value === "string") return value.length;
  return JSON.stringify(value).length;
}
```

### Edge Cases

- **`typeof null` history**: JavaScript 1.0 (1995)'dan beri bug — fix backward incompatible.
- **`typeof undefined` correct**: `"undefined"` — to'g'ri.
- **`Array.isArray(null)`**: false — array tekshiruvi xavfsiz.

### Follow-up savollar

1. **"Nima uchun JS engine'lar `null` ni `"null"` deb qaytarmaydi?"** — TC39 backward compatibility uchun reject qilgan.
2. **"`typeof` qanday yo'l bilan reliable yozish kerak?"** — Discriminated union yoki `instanceof`/`Array.isArray` kombinatsiyasi.

</details>

---

## Coding savollar

### Savol 10: `satisfies` bilan type-safe routing config [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`satisfies RouteConfig` har route mosligini tekshiradi, lekin `handler` return tipini aniq saqlaydi. `as const` literal property'larni o'zgarmas qiladi.

### To'liq tushuntirish

Routing config — har route uchun method va handler. `:` annotation bilan declare qilinsa, `handler` return tipi `unknown` (RouteConfig'dagi umumiy tip) ga widen qilinadi — call site'da `as` assertion kerak. `satisfies` esa har handler'ning inferred return tipini saqlab, validation qo'shadi — method literal (`"GET"`/`"POST"`) ham aniq saqlanadi. `as const` qo'shilsa — har string property literal sifatida frozen bo'ladi (`"GET"` `string` ga widen qilinmaydi). Bu kombinatsiya tRPC, Hono, Fastify schema-based router'lar uchun fundamental — type-safe RPC. `keyof typeof routes` bilan har route key inferred, `Parameters`/`ReturnType` utility'lari bilan handler signature decompose qilinadi.

### Kod misol

```typescript
type RouteConfig = Record<string, {
  method: "GET" | "POST" | "PUT" | "DELETE";
  handler: (...args: any[]) => unknown;
}>;

// ✅ satisfies bilan — validation + inferred type
const routes = {
  listUsers: {
    method: "GET",
    handler: () => ["Aziz", "Vali", "Soli"],
  },
  createUser: {
    method: "POST",
    handler: (name: string) => ({ id: Math.random(), name }),
  },
  health: {
    method: "GET",
    handler: () => ({ status: "ok" }),
  },
} as const satisfies RouteConfig;

// Inferred type'lar saqlangan:
const users = routes.listUsers.handler();       // string[]
const newUser = routes.createUser.handler("Aziz"); // { id: number; name: string }
const health = routes.health.handler();         // { status: string }

// Method literal saqlangan (`as const` tufayli):
routes.listUsers.method;   // "GET" (literal)
routes.createUser.method;  // "POST" (literal)

// ❌ `:` annotation bilan — aniq tip yo'qoldi
const routesBad: RouteConfig = {
  listUsers: { method: "GET", handler: () => ["x"] },
  createUser: { method: "POST", handler: () => ({}) },
};
routesBad.listUsers.handler(); // unknown — TS bilmaydi
routesBad.listUsers.method;    // "GET" | "POST" | "PUT" | "DELETE" — widen

// Real-world: type-safe API client
type RouteKey = keyof typeof routes;

function callRoute<K extends RouteKey>(
  key: K,
  ...args: Parameters<typeof routes[K]["handler"]>
): ReturnType<typeof routes[K]["handler"]> {
  return routes[key].handler(...args) as ReturnType<typeof routes[K]["handler"]>;
}

const u = callRoute("listUsers");     // string[]
const n = callRoute("createUser", "Aziz"); // { id: number; name: string }
```

### Edge Cases

- **`as const` shart**: literal property uchun (`"GET"` → `string` widen oldini olish).
- **Generic constraint**: `K extends RouteKey` mapped access uchun.
- **`unknown` vs `any`**: `RouteConfig` `unknown` qaytaradi — safer, lekin assertion kerak.

### Follow-up savollar

1. **"`tRPC` qanday qilib bunday type-safety beradi?"** — Procedure'larni `satisfies`-like pattern bilan deklare qiladi, client side'da generic mapped type'lar.
2. **"`Parameters`/`ReturnType` qanday ishlaydi?"** — Utility types: `infer` bilan function signature'ni decompose qiladi.

</details>

---

### Savol 11: Custom assertion function — API response validator [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`asserts value is ApiResponse` predicate bilan — har property tekshiriladi, xato bo'lsa throw, muvaffaqiyatli o'tsa keyingi kodda type narrow.

### To'liq tushuntirish

`fetch().then(r => r.json())` natijasi `unknown` (yoki `any`) — runtime tipini compile-time'da kafolatlash mumkin emas. To'g'ri pattern: API boundary'da validation — har property recursive tekshiriladi. Assertion function `asserts value is ApiResponse` predicate'i — agar throw bo'lmasa, keyingi kod context'ida `value` `ApiResponse` deb narrow qilinadi. Har property uchun: type tekshiruvi (`typeof`/`Array.isArray`), null check, recursive iteration (massivlar uchun). Manual assertion ko'p kod yozadi — production'da `zod`/`io-ts`/`valibot` schema library'lari afzal: schema definition'dan tip va validation function ikkalasi avtomatik infer qilinadi.

### Kod misol

```typescript
interface ApiResponse {
  status: number;
  data: { users: { id: number; name: string }[] };
}

function assertValidResponse(value: unknown): asserts value is ApiResponse {
  if (typeof value !== "object" || value === null) {
    throw new Error("Response must be an object");
  }

  const obj = value as Record<string, unknown>;

  if (typeof obj.status !== "number") {
    throw new Error(`Response.status must be number, got ${typeof obj.status}`);
  }

  if (typeof obj.data !== "object" || obj.data === null) {
    throw new Error("Response.data must be an object");
  }

  const data = obj.data as Record<string, unknown>;

  if (!Array.isArray(data.users)) {
    throw new Error("Response.data.users must be an array");
  }

  for (const [index, user] of data.users.entries()) {
    if (typeof user !== "object" || user === null) {
      throw new Error(`User #${index} must be an object`);
    }
    const u = user as Record<string, unknown>;
    if (typeof u.id !== "number") {
      throw new Error(`User #${index}.id must be number`);
    }
    if (typeof u.name !== "string") {
      throw new Error(`User #${index}.name must be string`);
    }
  }
}

// Usage
async function fetchUsers(): Promise<string[]> {
  const raw: unknown = await fetch("/api/users").then(r => r.json());
  assertValidResponse(raw);
  // raw: ApiResponse — bu nuqtadan keyin type-safe
  return raw.data.users.map(u => u.name);
}

// Real-world bilan zod
// import { z } from "zod";
// const schema = z.object({
//   status: z.number(),
//   data: z.object({
//     users: z.array(z.object({ id: z.number(), name: z.string() })),
//   }),
// });
// const parsed = schema.parse(raw); // throws on invalid, narrows on success
```

### Edge Cases

- **Boilerplate**: manual assertion ko'p kod yozadi — `zod`/`io-ts`/`valibot` library afzal.
- **Error message localization**: error xabarlari foydalanuvchi tilida bo'lishi mumkin.
- **Recursive validation**: nested object'lar uchun rekursiv assertion.
- **Partial validation**: ba'zi property'larni skip qilish — flexibility uchun.

### Follow-up savollar

1. **"`zod` `parse` vs `safeParse` farqi?"** — `parse` throws, `safeParse` `Result<T, E>` qaytaradi.
2. **"Runtime validation va TypeScript type birga sinxron qanday bo'ladi?"** — `z.infer<typeof schema>` — schema'dan type'ni infer qiladi.

</details>

---

### Savol 12: Type-safe event emitter type guard'lar bilan [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Event map type bilan har event uchun aniq payload tip. Type guard funksiyalar har event ni narrow qiladi. Listener — generic constraint bilan.

### To'liq tushuntirish

Native Node `EventEmitter` type-unsafe — har event uchun handler `(...args: any[])` qabul qiladi, payload tipi kafolatlanmaydi. Type-safe wrapper: `EventMap` interface har event nomini specific payload tipiga map qiladi. `on<E extends EventName>(event, handler)` generic — har chaqirilishda specific `E` literal'i orqali handler payload tipi narrow qilinadi. `emit<E>` ham generic — wrong event nomi yoki noto'g'ri payload compile-time'da bloklanadi. Untrusted source'lardan (`postMessage`, `WebSocket`) kelgan string'larni dispatch qilish uchun — type guard `isEventName(value): value is EventName` va payload validator'lar (`isUserLoginPayload`) kerak. Bu pattern Redux store, FSA actions, EventBus library'larida fundamental.

### Kod misol

```typescript
// Event map — discriminated union'ga ekvivalent
interface EventMap {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "data:update": { key: string; value: unknown };
}

type EventName = keyof EventMap;
type EventPayload<E extends EventName> = EventMap[E];

class TypedEmitter {
  private listeners: {
    [K in EventName]?: Array<(payload: EventPayload<K>) => void>;
  } = {};

  on<E extends EventName>(event: E, handler: (payload: EventPayload<E>) => void): void {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    (this.listeners[event] as Array<(payload: EventPayload<E>) => void>).push(handler);
  }

  emit<E extends EventName>(event: E, payload: EventPayload<E>): void {
    const handlers = this.listeners[event];
    if (handlers) {
      (handlers as Array<(payload: EventPayload<E>) => void>).forEach(h => h(payload));
    }
  }
}

// Type guard — runtime event verification
function isEventName(value: unknown): value is EventName {
  return (
    typeof value === "string" &&
    (value === "user:login" || value === "user:logout" || value === "data:update")
  );
}

// Usage
const emitter = new TypedEmitter();

emitter.on("user:login", (payload) => {
  console.log(payload.userId);   // ✅ string
  console.log(payload.timestamp); // ✅ number
});

emitter.on("user:logout", (payload) => {
  console.log(payload.userId);   // ✅ string
  // payload.timestamp;          // ❌ Property 'timestamp' does not exist
});

emitter.emit("user:login", { userId: "u1", timestamp: Date.now() });
// emitter.emit("user:login", { userId: "u1" }); // ❌ timestamp missing
// emitter.emit("invalid", {});                  // ❌ Not in EventMap

// Untrusted source bilan
function dispatchFromString(raw: string, payload: unknown): void {
  if (isEventName(raw)) {
    // raw: EventName — narrow
    // Lekin payload'ni alohida validate qilish kerak
    if (raw === "user:login" && isUserLoginPayload(payload)) {
      emitter.emit(raw, payload);
    }
  }
}

function isUserLoginPayload(value: unknown): value is { userId: string; timestamp: number } {
  return (
    typeof value === "object" && value !== null &&
    typeof (value as Record<string, unknown>).userId === "string" &&
    typeof (value as Record<string, unknown>).timestamp === "number"
  );
}
```

### Edge Cases

- **Event map sync**: schema o'zgarsa har joyda update kerak — single source of truth muhim.
- **Generic constraint**: `E extends EventName` bilan har chaqiruvda aniq tip.
- **`as` assertion'lar**: storage'ning generic'lik tabiati tufayli kerak — alternativa `Map` ishlatish.

### Follow-up savollar

1. **"Node EventEmitter type-safe qanday qilinadi?"** — `@types/node` da generic interface yo'q, lekin override yoki wrapper class bilan.
2. **"`mitt` library qanday ishlaydi?"** — Minimal tiny emitter, generic event map qabul qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Mapped type + indexed access mexanizmi**: `EventMap[E]` indexed access `E` generic parametri orqali compile-time'da resolve qilinadi. TypeScript `getIndexedAccessType` chaqiruvi orqali har specific `E` literal uchun tegishli payload tipini hisoblaydi. Storage `{ [K in EventName]?: ... }` mapped type — har key uchun array, lekin generic context'da `Array<(payload: EventPayload<E>) => void>` exact tipni saqlamaydi (variance loss).

**`as` assertion'lar sababi**: storage `{[K in EventName]?: ...}` mapped type'da har key uchun **invariant** array. Lekin runtime'da access — generic `E` orqali, compiler array tipini specific `E` ga match qila olmaydi. `as Array<...>` ishonchli — invariant ichki kodda `K` bilan kafolatlangan. Alternative: storage'ni `Map<EventName, Array<unknown>>` qilish + cast on access.

**Distributive conditional + EventMap**: agar `match` pattern kerak bo'lsa, distributive conditional bilan har case uchun handler tipi: `{ [K in EventName]: (payload: EventMap[K]) => R }` — union'da har handler aniq payload tipini biladi.

**`emit` overload alternative**: generic `emit<E>` o'rniga function overload signatures yozish ham mumkin: `emit(e: "user:login", p: {userId: string; timestamp: number}): void` — har event uchun alohida signature. Trade-off: overload'lar declarative, generic — DRY.

</details>

</details>

---

## Bug fix savollar

### Savol 13: Narrowing cheklovlari — 3 ta muammo va yechim [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

(1) `typeof null === "object"` — `=== null` alohida. (2) Closure'da narrowing yo'qoladi — `const` capture. (3) Object property narrowing function call'dan keyin yo'qoladi — local snapshot.

### To'liq tushuntirish

TypeScript narrowing — control flow analysis, intra-procedural (bir funksiya ichida). Compiler bir nechta scenario'da conservative bo'ladi: (1) `typeof null === "object"` — JavaScript spec bug, union'ga `object` qo'shilganda null hali ham ko'rinmas tarzda kiradi; (2) closure (callback, setTimeout) ichida `let` reassign bo'lishi mumkin — closure chaqirilguncha — narrowing yo'qoladi; (3) function chaqiruv side-effect orqali object property'ni o'zgartirishi mumkin — keyingi access'da narrowing reset. Universal yechim — `const` lokal o'zgaruvchiga snapshot olish: `const x = obj.prop; if (x !== null) { ... use x ... }`. Const immutable binding — reassign mumkin emas, narrowing har joyda saqlanadi. TS 5.4+ flow-sensitive `let` improvement: `let` narrowing'dan keyin reassign bo'lmasa, closure'da ham saqlanadi.

### Kod misol

```typescript
// Muammo 1: typeof null
function process1Bad(val: object | null): string {
  if (typeof val === "object") {
    return val.toString(); // ❌ val: object | null — null'ga crash
  }
  return "not object";
}

// ✅ Tuzatish 1
function process1Good(val: object | null): string {
  if (val !== null && typeof val === "object") {
    return val.toString(); // ✅ val: object
  }
  return "not object";
}

// Muammo 2: Closure narrowing yo'qoladi (TS 5.3-)
function process2Bad(): void {
  let x: string | number = "hello";
  if (typeof x === "string") {
    setTimeout(() => {
      console.log(x.toUpperCase()); // ❌ x reassign bo'lgan bo'lishi mumkin
    }, 100);
    x = 42; // reassign — closure'da xavfli
  }
}

// ✅ Tuzatish 2 — const capture
function process2Good(): void {
  let x: string | number = "hello";
  if (typeof x === "string") {
    const captured = x;
    setTimeout(() => {
      console.log(captured.toUpperCase()); // ✅ captured: string
    }, 100);
    x = 42;
  }
}

// Muammo 3: Object property + external mutation
function process3Bad(obj: { name: string | null }): void {
  if (obj.name !== null) {
    externalMutate(obj);
    // External mutation obj.name'ni null qilishi mumkin
    console.log(obj.name.toUpperCase()); // ❌ Possibly null
  }
}

declare function externalMutate(o: { name: string | null }): void;

// ✅ Tuzatish 3 — local snapshot
function process3Good(obj: { name: string | null }): void {
  const name = obj.name;
  if (name !== null) {
    externalMutate(obj);
    console.log(name.toUpperCase()); // ✅ name: string — local
  }
}

// Umumiy yechim — const ga capture
function processGeneric<T>(value: T | null): void {
  const captured = value;
  if (captured !== null) {
    // Har joyda captured ishlatilsa narrowing saqlanadi
    use(captured);
  }
}

declare function use<T>(value: T): void;
```

### Edge Cases

- **TS 5.4+ improvement**: `let` reassign bo'lmasa closure narrowing saqlanadi.
- **`readonly` property**: `readonly name: string | null` narrowing function call'dan keyin saqlanadi (mutation imkonsiz).
- **Aliased conditions**: `const isStr = typeof x === "string"` — TS 4.4+ predicate alias ishlaydi.

### Follow-up savollar

1. **"Bu cheklovlar nima uchun bor?"** — Inter-procedural analysis'siz compiler conservative bo'lishi shart — soundness uchun.
2. **"Yagona universal yechim qaysi?"** — Local `const` ga capture — har TS versiyada ishlaydi, har scenario'da xavfsiz.

<details>
<summary><strong>Deep Dive</strong></summary>

**Soundness vs completeness trade-off**: TypeScript control flow analysis intentionally **unsound** ba'zi joylarda (covariant array, bivariant method) — ergonomics uchun. Lekin narrowing'da soundness afzal — function call'dan keyin object property narrowing reset qilinadi, chunki callee mutation qilishi mumkin. Compiler `getNarrowedTypeWorker` ichida har reference uchun `lastAssignmentLeavesNarrowedType` flag'ini tracking qiladi.

**`getFlowTypeOfReference` algoritmi**: TypeScript har `Reference` (variable, property access) uchun flow node graf'idan boshlab teskari yo'nalishda walk qiladi. Har `FlowAssignment`, `FlowCondition`, `FlowCall` node'i tipni narrowing/widening yo'li bilan o'zgartiradi. Closure scope cross-bo'lganda — local flow context yo'qoladi, declared type qaytariladi.

**TS 5.4 mexanizmi — `lastAssignmentLeavesNarrowedType`**: agar `let` o'zgaruvchi narrowing'dan keyin shu branch'da reassign bo'lmasa — compiler closure'da ham narrow tipni saqlaydi. Implementation `getTypeAtFlowNode` ichida — `FlowFlags.Unreachable` va `FlowFlags.Assignment` flag'larini tracking qiladi.

**`readonly` interaction**: `readonly` property mutation imkonsiz bo'lgani uchun — function call'dan keyin ham narrowing saqlanadi. Bu pattern API design'da muhim: immutable data — predictable type narrowing.

</details>

</details>

---

### Savol 14: Non-null assertion (`!`) ortiqcha ishlatish [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`value!` operator runtime tekshiruvsiz null/undefined olib tashlaydi. Xavfli — runtime crash. Aniq narrowing yoki optional chaining afzal.

### To'liq tushuntirish

Non-null assertion (`!`) — TypeScript compiler'ga "bu qiymat hech qachon `null`/`undefined` emas" deb signal beradi. Runtime tekshiruv qo'shilmaydi — agar qiymat haqiqatda `null` bo'lsa, keyingi property access JavaScript'da `TypeError` beradi. Bu pattern xavfli, chunki invariant kod o'zgarganda buzilishi mumkin (compiler ushlay olmaydi). To'g'ri pattern: (1) explicit null check va early return/throw — runtime kafolatli, (2) optional chaining + nullish coalescing — default qiymat bilan, (3) assertion function (`asserts value is T`) — markazlashgan invariant tekshiruv. `!` ruxsat etiladigan kam holatlar: definite assignment (constructor'dan tashqarida init qilinadigan class field), va invariant matematik kafolatlangan joylarda.

### Kod misol

```typescript
// ❌ Ortiqcha ishlatish — xavfli
function processUser(id: string): string {
  const user = findUser(id);
  return user!.name; // ❌ Agar findUser null qaytarsa — runtime crash
}

declare function findUser(id: string): { name: string } | null;

// ✅ Tuzatish 1 — aniq tekshiruv
function processUserGood(id: string): string {
  const user = findUser(id);
  if (user === null) {
    throw new Error(`User ${id} not found`);
  }
  return user.name;
}

// ✅ Tuzatish 2 — default value
function processUserDefault(id: string): string {
  const user = findUser(id);
  return user?.name ?? "Unknown";
}

// ✅ Tuzatish 3 — assertion function
function assertNotNull<T>(value: T | null, message?: string): asserts value is T {
  if (value === null) {
    throw new Error(message ?? "Expected non-null");
  }
}

function processUserAssert(id: string): string {
  const user = findUser(id);
  assertNotNull(user, `User ${id} not found`);
  return user.name; // ✅ Narrow
}

// `!` qachon ruxsat etiladi: invariant garantiya bilan
class Container {
  private data!: string; // Definite assignment assertion (constructor'da set qilinadi)

  constructor() {
    this.initialize();
  }

  private initialize(): void {
    this.data = "initialized";
  }

  getData(): string {
    return this.data; // OK — initialize'da set qilingan
  }
}

// DOM access — element mavjudligi kafolatlangan
const root = document.getElementById("root")!; // ❌ Yoki:

// ✅ Aniqroq
const rootSafe = document.getElementById("root");
if (rootSafe === null) {
  throw new Error("Root element not found");
}
rootSafe.appendChild(/* ... */);
```

### Edge Cases

- **Definite assignment**: class field `!:` constructor'dan tashqarida set qilinadi — runtime'da initialized bo'lishi kerak.
- **DOM elements**: `getElementById!` keng tarqalgan, lekin xavfli — runtime'da element mavjudligi guarantee yo'q.
- **Array index**: `arr[0]!` — `noUncheckedIndexedAccess` bilan kerak bo'ladi, lekin runtime undefined bo'lishi mumkin.
- **`ts-essentials` `Optional<T>`**: optional handling utility'lari.

### Follow-up savollar

1. **"`!` qachon ruxsat etiladi?"** — Invariant matematik kafolatlangan bo'lsa (constructor'da init, kompilyator buni ushlay olmaydi).
2. **"ESLint qoidasi bormi?"** — `@typescript-eslint/no-non-null-assertion` — bloklaydi yoki ogohlantiradi.
3. **"`noUncheckedIndexedAccess` bilan qanday?"** — Array/Object access har doim `T | undefined` qaytaradi — `!` yoki aniq tekshiruv.

</details>

---

## Xulosa

- Type narrowing — union tipni `typeof`/`instanceof`/`in`/equality bilan toraytirish.
- Control flow analysis — kompilyator kodni yuqoridan pastga oqim grafi bo'yicha tahlil qiladi.
- Truthiness narrowing — `0`/`""` falsy — string/number tipida xavfli, aniq `!== null` afzal.
- Equality narrowing — `===` strict, `== null` ikkala null va undefined.
- `typeof null === "object"` — JS bug, `=== null` alohida.
- Type guard (`is`) — branching, assertion (`asserts`) — preconditions.
- `satisfies` — type validation + inferred saqlash. `:` annotation widen qiladi.
- Closure va external mutation — narrowing yo'qolishi mumkin, `const` capture universal yechim.
- Non-null assertion (`!`) — invariant kafolatsiz xavfli, aniq narrowing afzal.
