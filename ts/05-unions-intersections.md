# Bo'lim 5: Type Aliases va Union/Intersection Types

> Type alias — tipga yangi nom berish mexanizmi. **Union types** (`|`) "bu **yoki** u" mantiqini, **Intersection types** (`&`) "ham bu, **ham** u" mantiqini type darajasida ifodalaydi. **Discriminated union**'lar esa type-safe state management va pattern matching'ning asosi — Redux, state machines, API responses, AST node'lar — barchasi shu pattern ustiga quriladi.

---

## Mundarija

- [Type Aliases](#type-aliases)
- [Union Types](#union-types)
- [Narrowing Union Types](#narrowing-union-types)
- [typeof Type Guard](#typeof-type-guard)
- [instanceof Type Guard](#instanceof-type-guard)
- [in Operator Type Guard](#in-operator-type-guard)
- [Intersection Types](#intersection-types)
- [Intersection Bilan Conflict](#intersection-bilan-conflict)
- [Discriminated Unions](#discriminated-unions)
- [Custom Type Guards](#custom-type-guards)
- [Exhaustive Checking](#exhaustive-checking)
- [assertNever Pattern](#assertnever-pattern)
- [Union Types va Function Overloads](#union-types-va-function-overloads)
- [satisfies bilan Validation](#satisfies-bilan-validation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Type Aliases

### Nazariya

Type alias — mavjud tipga **yangi nom berish**. `type` keyword bilan yaratiladi. Muhim: type alias **yangi tip yaratmaydi** — faqat mavjud tipga muqobil nom (alias) beradi. Lekin bu nom kodni o'qishni osonlashtiradi va murakkab tiplarni bir joyda define qilib, ko'p joyda qayta ishlatish imkonini beradi.

Type alias **har qanday** tip uchun ishlatiladi:

```typescript
// Primitive type alias — semantik ma'no
type UserID = string;
type Age = number;
type IsActive = boolean;

// Union type — interface'da mumkin emas
type StringOrNumber = string | number;
type Status = "idle" | "loading" | "success" | "error";

// Object shape — interface ham ishlaydi
type User = {
  id: UserID;
  name: string;
  age: Age;
};

// Function type — callback signature
type Formatter = (input: string) => string;
type Predicate<T> = (value: T) => boolean;

// Tuple type — interface'da mumkin emas
type Coordinate = [number, number];
type Entry<K, V> = [K, V];

// Generic type alias
type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
};

// Mapped type — interface'da mumkin emas
type Optional<T> = { [K in keyof T]?: T[K] };

// Conditional type — interface'da mumkin emas
type NonNull<T> = T extends null | undefined ? never : T;
```

**Type alias va interface — asosiy farq:**

| Xususiyat | Type Alias | Interface |
|-----------|------------|-----------|
| Object shape | ✅ | ✅ (afzal) |
| Union types | ✅ | ❌ |
| Tuple/primitive | ✅ | ❌ |
| Mapped/conditional | ✅ | ❌ |
| Declaration merging | ❌ | ✅ |
| `extends` | `&` bilan | `extends` |

**Qoida:** object shape uchun — **interface**. Union, tuple, primitive alias, mapped, conditional — **type alias**. Batafsil: [04-objects-interfaces.md](04-objects-interfaces.md).

<details>
<summary><strong>Under the Hood</strong></summary>

`tsc` type alias'ni `TypeAliasDeclaration` AST node sifatida parse qiladi. Checker bosqichida type alias'ni ishlatilgan joyda **resolve** qiladi — alias'ning o'zini emas, uning **target type**'ini ishlatadi.

```
Type Alias Resolution:

Source                     Checker                        Emitted JS
──────                     ───────                        ──────────
type UserID = string   →   TypeAlias {                    (hech narsa)
                             name: "UserID",
                             target: StringType
                           }
const id: UserID       →   id resolves to StringType      const id = "abc"
```

Type alias compile-time'da `SymbolTable`'ga yoziladi — har alias o'z scope'ida unique symbol oladi. Emit bosqichida alias **butunlay o'chiriladi** — JS output'da alias'ning hech qanday izi qolmaydi.

**Generic type alias** (`type ApiResponse<T> = {...}`) checker'da `GenericTypeAlias` sifatida saqlanadi. Har safar `ApiResponse<User>` ishlatilganda, compiler yangi **instantiated type** yaratadi — `T`'ni `User` bilan almashtiradi. Bu instantiation compile-time'da, runtime'da generic'ning izi yo'q.

**Type alias vs interface — ichki farq:**

- **Type alias:** `TypeAliasDeclaration` — type expression'ni lazy evaluate qiladi (zamonaviy TS'da cached). Declaration merging yo'q.
- **Interface:** `InterfaceType` — `SymbolTable` bilan eager built. Declaration merging `binder.ts`'da `mergeSymbol()` orqali.

Zamonaviy TS'da (4.x+) ikkalasi ham cached — performance farqi kam. Asosiy farq **semantic expressiveness**: type alias kengroq constructs'ni qo'llab-quvvatlaydi (union, mapped, conditional), interface esa object shape va class implements'ga ixtisoslashgan.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Primitive aliases — semantik ma'no
type Email = string;
type UserId = number;
type Age = number;

function sendEmail(to: Email, subject: string) { /* ... */ }
// sendEmail'da "to: Email" yozilishi "to: string"'dan aniqroq

// Object shape — interface ham ishlaydi
type Product = {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
};

// Union — faqat type alias
type PaymentMethod = "card" | "paypal" | "cash";
type ID = string | number;

// Function type
type Handler<E> = (event: E) => void;
type Validator<T> = (value: T) => boolean | string;

const isValidEmail: Validator<string> = (email) =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email) || "Invalid email";

// Tuple
type Point2D = [number, number];
type Point3D = [number, number, number];

const origin: Point2D = [0, 0];

// Generic alias
type Box<T> = { value: T };
type Pair<K, V> = { key: K; value: V };

const stringBox: Box<string> = { value: "hello" };
const entry: Pair<string, number> = { key: "age", value: 25 };

// Recursive type alias
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

// Composition
type WithTimestamps = {
  createdAt: Date;
  updatedAt: Date;
};

type UserWithTimestamps = User & WithTimestamps;
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
type UserID = string;
type User = { id: UserID; name: string };
type Status = "active" | "inactive";

const user: User = { id: "u1", name: "Ali" };
const status: Status = "active";
```

```javascript
// Compiled JS — type alias'lar TO'LIQ o'chiriladi
const user = { id: "u1", name: "Ali" };
const status = "active";

// Type alias — compile-time tushuncha, JS'da hech qanday iz yo'q
// Runtime'da UserID, User, Status — mavjud emas
```

Type alias pure type erasure — JavaScript'da hech qanday wrapper, metadata yoki runtime kod yo'q. Bu TypeScript'ning fundamental prinsipi.

</details>

---

## Union Types

### Nazariya

Union type — qiymat **bir nechta tipdan birortasi** bo'lishi mumkinligini bildiradi. `|` (pipe) operator bilan yoziladi. Mantiqiy jihatdan "yoki" (OR).

Union type yaratilganida, shu tipdagi qiymat **faqat bitta** member tipga tegishli bo'ladi (bir vaqtda bir nechtasiga emas). TypeScript compiler qiymatning qaysi member tipga tegishli ekanligini aniqlay olmaydi — buning uchun **narrowing** kerak.

```typescript
// Oddiy union
type ID = string | number;

let userId: ID = "abc-123";  // ✅ string
userId = 42;                  // ✅ number
// userId = true;             // ❌ boolean union'da yo'q

// Literal union — enum alternativa
type Direction = "up" | "down" | "left" | "right";
type HttpStatus = 200 | 201 | 400 | 401 | 403 | 404 | 500;
type EventType = "click" | "hover" | "focus" | "blur";

let dir: Direction = "up";
// dir = "diagonal"; // ❌ "diagonal" union'da yo'q
```

**Muhim cheklov:** Union tipida faqat **barcha member'larga umumiy** bo'lgan property va metod'larni ishlatish mumkin. Masalan, `string | number`'da faqat ikkalasida ham bor bo'lgan `.toString()`, `.valueOf()` ishlaydi. `.toUpperCase()` (faqat string) yoki `.toFixed()` (faqat number) — narrowing'siz ishlamaydi.

```typescript
function printValue(value: string | number): void {
  console.log(value.toString());    // ✅ toString ikkalasida ham bor
  console.log(value.valueOf());     // ✅ valueOf ikkalasida ham bor

  // value.toUpperCase(); // ❌ faqat string'da
  // value.toFixed(2);    // ❌ faqat number'da
}
```

Union — TypeScript type system'ining eng ko'p ishlatiladigan konstruksiyalaridan biri. API response'lar, parameter variants, state types, discriminated unions — barchasi union ustiga quriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`checker.ts`'da union type `UnionType` object sifatida yaratiladi — `getUnionType()` funksiyasi orqali. Bu funksiya member'larni **flatten** qiladi (nested union'larni yoyadi), **duplicate**'larni olib tashlaydi va **subtype**'larni eliminate qiladi:

```
getUnionType([string, number]):

1. flattenUnionType()     →  nested union'larni yoyish
                              A | (B | C) → A | B | C

2. removeDuplicates()     →  takroriy tiplarni olib tashlash
                              string | string | number → string | number

3. reduceUnionType()      →  subtype'larni olib tashlash
                              "hello" | string → string (literal string'ga kiradi)

4. Natija:
   UnionType {
     flags: TypeFlags.Union,
     types: [StringType, NumberType],
     objectFlags: ObjectFlags.None
   }
```

**Widening gotcha:**

TypeScript literal tiplarni keng tipga **widening** qiladi `let`/parameter context'ida:

```typescript
let x = "hello";  // x type: string (widened from "hello")
const y = "hello"; // y type: "hello" (literal)

// Union'da ham
let id = condition ? "abc" : 42;
// id type: string | number (widened)

const id2 = condition ? "abc" : 42;
// id2 type: string | number (ham widened — conditional expression)

// const bilan literal:
const id3 = "abc" as const; // type: "abc"
```

**Property access'da common members:**

Union'da property access'ida `checker.ts` `getPropertyOfUnionOrIntersectionType()` funksiyasini chaqiradi. Bu funksiya **barcha** member tiplarda mavjud bo'lgan property'lar topadi — faqat umumiy property'lar accessible.

**Union size:**

Union member'lar soni texnik jihatdan cheklanmagan, lekin amaliy cheklov bor:
- Distribution paytida union size cheklanadi (taxminan 100'lar atrofida)
- Juda katta union'lar performance'ga ta'sir qiladi — checker O(n) iteratsiya qiladi
- TS team recommend: 100'dan kam member (lekin `tsc` ko'p ko'proq member'larni qo'llab-quvvatlaydi)

Runtime'da union type'ning izi yo'q — faqat `typeof`, `instanceof`, `in` kabi JS tekshiruvlar qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Simple union
type StringOrNumber = string | number;

function format(value: StringOrNumber): string {
  if (typeof value === "string") return value;
  return value.toString();
}

// Literal union
type Status = "idle" | "loading" | "success" | "error";
type Role = "admin" | "moderator" | "user" | "guest";

function getLabel(status: Status): string {
  const labels = {
    idle: "Ready",
    loading: "Loading...",
    success: "Done",
    error: "Failed",
  };
  return labels[status];
}

// Numeric literal union
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

function rollDice(): DiceRoll {
  return (Math.floor(Math.random() * 6) + 1) as DiceRoll;
}

// Object union
type Response =
  | { success: true; data: string }
  | { success: false; error: string };

// Null/undefined bilan
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Maybe<T> = T | null | undefined;

function find(id: number): Nullable<User> {
  return users.find(u => u.id === id) ?? null;
}

// Union va array
type MixedArray = (string | number)[];
type StringArrayOrNumberArray = string[] | number[];

// Farq:
const mixed: MixedArray = ["a", 1, "b", 2];           // ✅ aralash
// const strict: StringArrayOrNumberArray = ["a", 1]; // ❌ bir tip bo'lishi kerak
const strings: StringArrayOrNumberArray = ["a", "b"];  // ✅
const numbers: StringArrayOrNumberArray = [1, 2, 3];    // ✅
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
type ID = string | number;
type Status = "idle" | "loading" | "success";

function format(id: ID, status: Status): string {
  if (typeof id === "string") return `${status}:${id.toUpperCase()}`;
  return `${status}:${id}`;
}
```

```javascript
// Compiled JS — union type'lar butunlay o'chiriladi
function format(id, status) {
  if (typeof id === "string") return `${status}:${id.toUpperCase()}`;
  return `${status}:${id}`;
}

// Union, literal type'lar — compile-time
// Runtime'da faqat typeof tekshiruvi qoladi
```

</details>

---

## Narrowing Union Types

### Nazariya

Narrowing — union tipni **aniqroq** (specific) tipga **toraytirish** jarayoni. Union yaratgandan keyin, TypeScript o'z-o'zidan qaysi member tip ekanligini bilmaydi. Uni aniqlash uchun **runtime tekshiruvlar** ishlatiladi — `typeof`, `instanceof`, `in`, equality check, user-defined type guards.

TypeScript compiler bu tekshiruvlarni tanib, type'ni avtomatik toraytiradi — bu **control flow analysis (CFA)** deyiladi. Narrowing'siz union type deyarli foydasiz — umumiy member'lardan boshqa hech narsaga kirish mumkin emas.

**Asosiy narrowing usullari:**

1. **`typeof`** — primitive tiplar (`string`, `number`, `boolean`, `undefined`, `object`, `function`, `bigint`, `symbol`)
2. **`instanceof`** — class instance'lar
3. **`in` operator** — property mavjudligi
4. **Equality** — `===`, `!==`, `==`, `!=`
5. **Truthiness** — `if (x)` — falsy qiymatlarni chiqarish
6. **Discriminant property** — discriminated union'larda
7. **Custom type guards** — `is` keyword bilan

Har bir usul quyida alohida bo'limlarda batafsil ko'rib chiqiladi.

> **Batafsil:** Control Flow Analysis, assertion functions, `satisfies` operator, closure'larda narrowing, narrowing limitation'lari [06-type-narrowing.md](06-type-narrowing.md) da yoritiladi. Bu bobda union type kontekstida asosiy narrowing usullari keltirilgan.

<details>
<summary><strong>Under the Hood</strong></summary>

Narrowing `checker.ts`'da **Control Flow Graph (CFG)** orqali amalga oshiriladi. Compiler kodni parse qilgandan keyin `binder.ts`'da CFG yaratadi — har `if`, `switch`, `return`, `throw`, `&&`, `||` yangi branch node hosil qiladi.

`checker.ts`'dagi `getFlowTypeOfReference()` funksiyasi har code nuqtasida o'zgaruvchining aniq tipini hisoblaydi:

```
Narrowing Pipeline:

Source → Parser → AST → Binder (CFG) → Checker (flow types)

        Control Flow Graph
        ┌────────────────────┐
        │ x: string | number │
        └────────┬───────────┘
                 │
        ┌────────▼────────────┐
        │ typeof x === "string"│
        └──┬──────────────┬───┘
      true │              │ false
    ┌──────▼────┐   ┌─────▼──────┐
    │ x: string │   │ x: number  │
    └───────────┘   └────────────┘
```

Har CFG node'da `getTypeAtFlowNode()` joriy tipni oldingi node'dagi tipdan va shart natijasidan hisoblaydi. Agar shart `typeof x === "string"` bo'lsa — `filterType()` funksiyasi union'dan `string`'ni ajratib oladi (true branch) yoki olib tashlaydi (false branch).

**Narrowing compile-time — runtime overhead yo'q:**

Narrowing butunlay compile-time'da sodir bo'ladi. Runtime'da faqat JavaScript tekshiruvlari (`typeof`, `instanceof`, `in`, `===`) qoladi — TypeScript hech qanday qo'shimcha runtime kod qo'shmaydi. Mavjud JS tekshiruvlari type information uchun ishlatilinadi.

</details>

---

## typeof Type Guard

### Nazariya

`typeof` operator — JavaScript'ning runtime operator'i bo'lib, qiymatning **primitive tip**'ini string sifatida qaytaradi. TypeScript buni **type guard** sifatida taniydi — `typeof` tekshiruvidan keyin blok ichida tip avtomatik toraytiriladi.

`typeof` qaytarishi mumkin bo'lgan qiymatlar:
- `"string"` — string primitive
- `"number"` — number (NaN, Infinity ham)
- `"bigint"` — bigint primitive
- `"boolean"` — `true` yoki `false`
- `"symbol"` — symbol primitive
- `"undefined"` — undefined
- `"object"` — object (+ **null** ham!)
- `"function"` — function

**Muhim gotcha:** `typeof null === "object"` — bu JavaScript'ning 1995-yildan beri mavjud **legacy bug**'i. TypeScript narrowing algoritmi bu bug'ni aks ettiradi — `typeof x === "object"` bilan narrowing'da `null` **chiqarilmaydi**. Shuning uchun doim `val !== null` qo'shimcha tekshiruv kerak.

```typescript
function processInput(input: string | number | boolean): string {
  if (typeof input === "string") {
    return input.toUpperCase(); // ✅ input: string
  }
  if (typeof input === "number") {
    return input.toFixed(2);    // ✅ input: number
  }
  return input ? "yes" : "no";  // ✅ input: boolean
}
```

**`typeof object` va null — ehtiyot:**

```typescript
function check(val: unknown) {
  if (typeof val === "object") {
    // val: object | null — null HAM chiqarilmaydi!
    // val.toString(); // ❌ 'val' is possibly 'null'
  }

  if (typeof val === "object" && val !== null) {
    // val: object — null endi chiqarilgan
    val.toString(); // ✅
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`typeof` — JS runtime operator, TypeScript hech qanday o'zgartirmaydi. Compiled JS'da `typeof` aynan saqlanadi. TypeScript faqat compile-time'da `typeof` natijasini tahlil qilib, keyingi block'dagi tipni toraytiradi.

`checker.ts`'da `typeof x === "string"` ko'rilganda `narrowTypeByTypeof()` funksiyasi chaqiriladi. Bu funksiya union tipi'ning member'larini iteratsiya qiladi va har member'ning `typeof` natijasini tekshiradi:

```
narrowTypeByTypeof("string", string | number | boolean):

Member: string  → typeof "" === "string"      → true  → KEEP
Member: number  → typeof 0 === "string"       → false → REMOVE
Member: boolean → typeof false === "string"   → false → REMOVE

Natija (true branch):  string
Natija (false branch): number | boolean
```

Compiler `typeof` qaytarishi mumkin bo'lgan qiymatlarni **hardcode** qilib biladi — `"string"`, `"number"`, `"bigint"`, `"boolean"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`. Agar `typeof x === "array"` yozilsa, TS syntax xato bermaydi (chunki JS syntax to'g'ri), lekin narrowing **ishlamaydi** — chunki `"array"` hech qachon `typeof` natijasi emas.

**`typeof null === "object"` aniq ishlov:**

Checker `narrowTypeByTypeof()`'da `"object"` case'ida `null`'ni chiqarmaydi — bu TypeScript'ning ataylab qilingan qaror. Sabab: runtime'da `null` haqiqatan ham `typeof === "object"` beradi, va narrowing runtime xulqni aks ettirishi kerak. `null`'ni chiqarish kerak bo'lsa, developer aniq tekshirishi kerak: `val !== null`.

**`typeof function` maxsus holat:**

`typeof` `"function"` faqat callable object'lar uchun ishlaydi. Arrow function, class, regular function — barchasi `"function"` qaytaradi. Class instance'lari esa `"object"` qaytaradi (constructor'ning o'zi `"function"`).

```typescript
class User {}
const user = new User();

typeof User;  // "function" — class constructor
typeof user;  // "object" — instance
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Basic narrowing
function stringify(value: string | number | boolean): string {
  if (typeof value === "string") return `"${value}"`;
  if (typeof value === "number") return value.toString();
  return value ? "true" : "false";
}

// Array check
function process(input: string | string[]): string[] {
  if (typeof input === "string") {
    return [input]; // input: string
  }
  return input; // input: string[]
}

// null with "object" type
function safeLog(val: unknown): void {
  if (typeof val === "object" && val !== null) {
    console.log(Object.keys(val)); // val: object
  }
}

// Callable check
type MaybeCallback = (() => void) | string;

function execute(cb: MaybeCallback): void {
  if (typeof cb === "function") {
    cb(); // cb: () => void
  } else {
    console.log(cb); // cb: string
  }
}

// Switch with typeof
function valueType(v: string | number | boolean | null | undefined): string {
  switch (typeof v) {
    case "string": return "text";
    case "number": return "number";
    case "boolean": return "bool";
    case "undefined": return "undefined";
    case "object": return v === null ? "null" : "object";
    default: return "unknown";
  }
}

// typeof yoki instanceof
function describe(value: string | Date | RegExp): string {
  if (typeof value === "string") {
    return `String: ${value}`;
  }
  if (value instanceof Date) {
    return `Date: ${value.toISOString()}`;
  }
  return `RegExp: ${value.source}`;
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function processInput(input: string | number): string {
  if (typeof input === "string") {
    return input.toUpperCase();
  }
  return input.toFixed(2);
}
```

```javascript
// Compiled JS — typeof AYNAN qoladi (runtime operator)
function processInput(input) {
  if (typeof input === "string") {
    return input.toUpperCase();
  }
  return input.toFixed(2);
}

// Type annotation va union'lar o'chirildi
// typeof — JS native operator, o'zgarmagan holda saqlandi
```

</details>

---

## instanceof Type Guard

### Nazariya

`instanceof` operator — object'ning qaysi **class** (yoki constructor function)'dan yaratilganligini tekshiradi. TypeScript buni type guard sifatida taniydi.

`instanceof` **prototype chain** bo'ylab tekshiradi: `obj instanceof Cls` — `Cls.prototype` `obj`'ning prototype chain'ida bormi?

```typescript
class ApiError {
  constructor(public statusCode: number, public message: string) {}
}

class ValidationError {
  constructor(public field: string, public message: string) {}
}

function handleError(error: ApiError | ValidationError): string {
  if (error instanceof ApiError) {
    return `API ${error.statusCode}: ${error.message}`;
    // error: ApiError
  }
  return `Field "${error.field}": ${error.message}`;
  // error: ValidationError
}
```

**Muhim cheklov:** `instanceof` faqat **class'lar** va **constructor function**'lar bilan ishlaydi. Interface va type alias bilan **ishlamaydi** — chunki ular runtime'da mavjud emas:

```typescript
interface Printable {
  print(): void;
}

function process(item: Printable) {
  // if (item instanceof Printable) {} // ❌ 'Printable' only refers to a type
  // Interface runtime'da yo'q
}
```

Built-in class'lar bilan ishlaydi:

```typescript
function formatDate(input: string | Date): string {
  if (input instanceof Date) {
    return input.toISOString();
  }
  return new Date(input).toISOString();
}

function processCollection(
  data: Array<number> | Map<string, number> | Set<number>
): void {
  if (data instanceof Array) {
    console.log(data.reduce((a, b) => a + b, 0));
  } else if (data instanceof Map) {
    data.forEach((v, k) => console.log(`${k}: ${v}`));
  } else {
    console.log(data.size);
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`instanceof` — JS runtime operator. Compiled JS'da aynan saqlanadi. Runtime'da prototype chain'ni tekshiradi: `obj.__proto__.__proto__... === Constructor.prototype`.

`checker.ts`'da `instanceof` ko'rilganda `narrowTypeByInstanceof()` funksiyasi ishlaydi. Bu funksiya union tipi'ning har member'ini tekshiradi — member `Constructor`'ning instance tipiga assignable bo'lsa, `true` branch'ga qo'shiladi:

```
narrowTypeByInstanceof(ApiError, ApiError | ValidationError):

true branch:
  ApiError → ApiError.prototype chain'da? → HA → KEEP
  ValidationError → ApiError.prototype chain'da? → YO'Q → REMOVE
  Natija: ApiError

false branch:
  ApiError → REMOVE (instanceof true bo'lishi mumkin emas)
  ValidationError → KEEP
  Natija: ValidationError
```

`instanceof` faqat **runtime'da mavjud** bo'lgan constructor'lar bilan ishlaydi. Checker `instanceof`'ning o'ng tomonidagi qiymatni tekshiradi — u **value** (runtime entity) bo'lishi kerak, **type** (compile-time entity) emas.

Interface va type alias runtime'da mavjud emas — shuning uchun `instanceof` bilan ishlamaydi. Class'lar runtime'da constructor function sifatida mavjud — shuning uchun `instanceof` ishlaydi.

**Narrowing va prototype chain:**

`instanceof` prototype chain bo'ylab ishlaganligi sababli, **inheritance** bilan ham ishlaydi:

```typescript
class Animal {}
class Dog extends Animal {}

const dog = new Dog();
dog instanceof Dog;    // true
dog instanceof Animal; // true (prototype chain'da Animal bor)
```

TypeScript bu xulqni to'g'ri track qiladi — base class'ga `instanceof` narrowing ham subclass instance'larini qamrab oladi.

**Iframe/cross-realm gotcha:**

Real-world'da `instanceof` ba'zan iframe yoki cross-realm (VM) context'larda **ishlamasligi** mumkin — chunki har realm o'z `Array`, `Map`, `Date` constructor'ini saqlaydi. `arr instanceof Array` false bo'lishi mumkin agar `arr` boshqa realm'da yaratilgan bo'lsa. Bu holda `Array.isArray(arr)` yoki `Object.prototype.toString.call(arr)` ishlatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Custom error class'lar
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

class ValidationError extends Error {
  constructor(public field: string, message: string) {
    super(message);
  }
}

function handleError(error: unknown): string {
  if (error instanceof ApiError) {
    return `API ${error.statusCode}: ${error.message}`;
  }
  if (error instanceof ValidationError) {
    return `Field ${error.field}: ${error.message}`;
  }
  if (error instanceof Error) {
    return `Error: ${error.message}`;
  }
  return "Unknown error";
}

// Collection types
function sum(items: number[] | Set<number>): number {
  if (items instanceof Set) {
    return [...items].reduce((a, b) => a + b, 0);
  }
  return items.reduce((a, b) => a + b, 0);
}

// DOM elements
function setupInput(element: HTMLElement): void {
  if (element instanceof HTMLInputElement) {
    element.value = "default";
    element.addEventListener("change", () => console.log(element.value));
  } else if (element instanceof HTMLTextAreaElement) {
    element.value = "default text";
  }
}

// Class hierarchy
abstract class Shape {
  abstract area(): number;
}

class Circle extends Shape {
  constructor(public radius: number) { super(); }
  area() { return Math.PI * this.radius ** 2; }
}

class Rectangle extends Shape {
  constructor(public width: number, public height: number) { super(); }
  area() { return this.width * this.height; }
}

function describeShape(shape: Shape): string {
  if (shape instanceof Circle) {
    return `Circle with radius ${shape.radius}`;
  }
  if (shape instanceof Rectangle) {
    return `Rectangle ${shape.width}x${shape.height}`;
  }
  return "Unknown shape";
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class ApiError {
  constructor(public statusCode: number, public message: string) {}
}
class ValidationError {
  constructor(public field: string, public message: string) {}
}

function handleError(e: ApiError | ValidationError): string {
  if (e instanceof ApiError) return `${e.statusCode}: ${e.message}`;
  return `${e.field}: ${e.message}`;
}
```

```javascript
// Compiled JS — instanceof AYNAN qoladi, class'lar o'zgarmagan
class ApiError {
  constructor(statusCode, message) {
    this.statusCode = statusCode;
    this.message = message;
  }
}
class ValidationError {
  constructor(field, message) {
    this.field = field;
    this.message = message;
  }
}

function handleError(e) {
  if (e instanceof ApiError) return `${e.statusCode}: ${e.message}`;
  return `${e.field}: ${e.message}`;
}

// instanceof JS runtime operator — o'zgartirilmaydi
```

</details>

---

## in Operator Type Guard

### Nazariya

`in` operator — object'da ma'lum bir **property mavjudligini** tekshiradi. TypeScript buni type guard sifatida taniydi — agar property faqat bitta union member'da bo'lsa, tip shu member'ga toraytiriladi.

`"prop" in obj` — `obj` yoki uning prototype chain'ida `prop` nomli property bormi? `true` yoki `false` qaytaradi.

```typescript
type Fish = { swim: () => void; name: string };
type Bird = { fly: () => void; name: string };

function move(animal: Fish | Bird): void {
  if ("swim" in animal) {
    animal.swim(); // animal: Fish
  } else {
    animal.fly();  // animal: Bird
  }
}
```

`in` operator ayniqsa **interface'lar** bilan narrowing'da foydali — `instanceof` interface'larda ishlamaydi, lekin `in` ishlaydi (runtime'da property mavjudligini tekshiradi):

```typescript
interface HttpResponse {
  status: number;
  data: unknown;
}

interface HttpError {
  status: number;
  error: string;
  retryAfter?: number;
}

function handle(response: HttpResponse | HttpError): void {
  if ("error" in response) {
    console.log(response.error); // response: HttpError
  } else {
    console.log(response.data);  // response: HttpResponse
  }
}
```

**TS 4.9'gacha qat'iy cheklov:** `in` operator faqat **discriminating property** bo'lganda narrowing qilardi — ya'ni property faqat bitta union member'da bo'lsa. **TS 4.9+ dan boshlab** `in` operator yanada kuchli — object'ni narrow qilish uchun property mavjudligi yetarli, hatto discriminating bo'lmasa ham.

```typescript
// TS 4.9+ yaxshilanish
type A = { shared: string };
type B = { shared: string; onlyB: boolean };

function process(value: A | B): void {
  if ("onlyB" in value) {
    // TS 4.9+: value: B (narrowed)
    // Eski TS: narrowing ishlamas edi
    console.log(value.onlyB);
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`in` — JS runtime operator, compiled JS'da aynan saqlanadi. Runtime'da `"prop" in obj` object'ning own property va prototype chain'ini tekshiradi. TypeScript hech qanday qo'shimcha kod qo'shmaydi.

`checker.ts`'da `"prop" in obj` ko'rilganda `narrowTypeByInKeyword()` funksiyasi chaqiriladi. Bu funksiya union'ning har member tipini tekshiradi — shu member'da `prop` nomli property bormi yoki yo'qmi:

```
narrowTypeByInKeyword("swim", Fish | Bird):

Fish type:  "swim" property bormi? → HA  → true branch
Bird type:  "swim" property bormi? → YO'Q → false branch

true:  Fish
false: Bird
```

**TS 4.9+ yaxshilanishi:**

Eski TS'da `in` operator faqat discriminating property bilan narrowing qilardi. TS 4.9'dan boshlab checker yanada kuchli algoritm ishlatadi: agar property faqat bitta member'da **qo'shimcha** sifatida bo'lsa ham (masalan, ikki tip bir xil asosga ega, lekin biri qo'shimcha property'ga ega), narrowing ishlaydi.

```
narrowTypeByInKeyword("extra", A | B) — TS 4.9+:

Type A = { shared: string }
Type B = { shared: string; extra: boolean }

"extra" in value:
  A: "extra" bormi? → YO'Q → false branch (A qoladi)
  B: "extra" bormi? → HA → true branch (B qoladi)

true:  B (eski TS'da value o'zgarmas edi)
false: A
```

**Optional property bilan ishlash:**

Agar `type A = { x?: number }` bo'lsa, `"x" in obj` hali ham A'ga narrow qiladi. Checker optional property'ni ham "mavjud bo'lishi mumkin" deb hisoblaydi. Runtime'da `in` operator property'ning mavjudligini tekshiradi (qiymati `undefined` bo'lsa ham `true` qaytaradi — property declared, qiymat undefined).

```typescript
const obj = { x: undefined };
"x" in obj; // true — property mavjud (qiymati undefined)

const obj2 = {};
"x" in obj2; // false — property yo'q
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Interface'lar bilan (instanceof ishlamaydi)
interface LoginResponse {
  token: string;
  user: { id: number; name: string };
}

interface ErrorResponse {
  error: string;
  code: number;
}

function handleAuth(response: LoginResponse | ErrorResponse): void {
  if ("token" in response) {
    // response: LoginResponse
    console.log(`Logged in as ${response.user.name}`);
  } else {
    // response: ErrorResponse
    console.log(`Error ${response.code}: ${response.error}`);
  }
}

// Discriminating property
type Admin = { role: "admin"; permissions: string[] };
type User = { role: "user"; email: string };

function greet(person: Admin | User): string {
  if ("permissions" in person) {
    return `Admin with ${person.permissions.length} permissions`;
  }
  return `User ${person.email}`;
}

// Complex shape
type EventWithData = { type: string; data: unknown };
type SimpleEvent = { type: string };

function processEvent(event: EventWithData | SimpleEvent): void {
  if ("data" in event) {
    console.log("Data:", event.data);
  } else {
    console.log("Simple event:", event.type);
  }
}

// `in` operator bilan type guard
function isUserInput(value: unknown): value is { username: string; password: string } {
  return (
    typeof value === "object" &&
    value !== null &&
    "username" in value &&
    "password" in value
  );
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type Fish = { swim: () => void; name: string };
type Bird = { fly: () => void; name: string };

function move(animal: Fish | Bird): void {
  if ("swim" in animal) {
    animal.swim();
  } else {
    animal.fly();
  }
}
```

```javascript
// Compiled JS — "in" operator AYNAN qoladi
function move(animal) {
  if ("swim" in animal) {
    animal.swim();
  } else {
    animal.fly();
  }
}

// Type alias'lar o'chirildi
// "in" — JS runtime operator, o'zgarmagan
```

</details>

---

## Intersection Types

### Nazariya

Intersection type — bir nechta tipni **birlashtiradi** (combine). `&` (ampersand) operator bilan yoziladi. Mantiqiy jihatdan "va" (AND). Natijadagi tip **barcha** member tiplar'ning property'larini o'z ichiga oladi.

Union "bu **yoki** u" desa, Intersection "**ham** bu, **ham** u" deydi.

```typescript
type HasName = { name: string };
type HasAge = { age: number };
type HasEmail = { email: string };

// Intersection — uchala tipni birlashtiradi
type Person = HasName & HasAge & HasEmail;
// { name: string; age: number; email: string }

const person: Person = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
  // Uch property ham majburiy
};
```

**Real-world — type composition:**

```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type SoftDeletable = {
  deletedAt: Date | null;
  isDeleted: boolean;
};

type Identifiable = {
  id: string;
};

// Composition — kichik tiplardan katta tiplar
type User = Identifiable & Timestamped & {
  name: string;
  email: string;
  role: "admin" | "user";
};

type Product = Identifiable & Timestamped & SoftDeletable & {
  title: string;
  price: number;
  stock: number;
};
```

**Primitive tiplar bilan intersection:**

```typescript
type Impossible = string & number; // never
// Hech qanday qiymat bir vaqtda string ham, number ham bo'lolmaydi

type AlsoImpossible = string & boolean; // never
```

Bu **type theory**'da "bo'sh intersection" (empty intersection) — TypeScript buni **bottom type** `never` sifatida modellashtiradi. Intuitiv: `string` va `number` ikki alohida "to'plam" — ularning umumiy qismi bo'sh, shuning uchun natija `never`.

**Literal tiplar bilan intersection — filter sifatida:**

```typescript
type A = "admin" | "user" | "guest";
type B = "admin" | "moderator";

type Common = A & B; // "admin"
// Faqat ikkala union'da ham bor bo'lgan literal qoladi
```

<details>
<summary><strong>Under the Hood</strong></summary>

Intersection type checker'da `IntersectionType` object sifatida yaratiladi — `getIntersectionType()` funksiyasi orqali. Algoritm:

```
getIntersectionType([A, B, C]):

1. flattenIntersectionType()    →  nested intersection'larni yoyish
                                    A & (B & C) → A & B & C

2. removeDuplicates()            →  takroriy tiplarni olib tashlash
                                    A & A & B → A & B

3. Primitive conflict check:
   - string & number → never
   - true & false → never
   - "a" & "b" → never

4. Object types — property merging:
   {x: string} & {y: number} → {x: string; y: number}

5. Recursive property intersection:
   {a: {x: 1}} & {a: {y: 2}} → {a: {x: 1; y: 2}}
```

**Type theory asosi:**

Intersection — ikki tip'ning "umumiy elementlari" to'plami. Agar:
- `A = {x: 1, 2, 3}` va `B = {2, 3, 4}` → `A & B = {2, 3}`
- `A = string` va `B = number` → umumiy element yo'q → `never` (bo'sh to'plam)
- `A = "a"` va `B = "a" | "b"` → `A & B = "a"`
- `A = {name: string}` va `B = {age: number}` → `A & B = {name: string, age: number}` (har ikki property'ga ega bo'lgan object'lar)

Object'larda intersection "property union" kabi ko'rinadi — chunki object'ning ikkala constraint'ga mos kelish uchun **har ikkala** property'ga ega bo'lishi kerak.

**`never` kengayishi:**

Primitive conflict'da `never` alohida property uchun bo'lishi mumkin, lekin object butunlay `never` bo'lmaydi:

```typescript
type A = { status: string; data: number };
type B = { status: number; data: string };

type AB = A & B;
// {
//   status: string & number; // never
//   data: number & string;   // never
// }
// AB "technically" exists, lekin hech qanday qiymat mos kelmaydi
// Har property'ga qiymat qo'yib bo'lmaydi
```

**Structural typing + intersection:**

Intersection structural typing bilan tabiiy birga ishlaydi — `A & B`'ga mos object `A`'ning va `B`'ning **barcha** property'larini qondirishi kerak. Bu object literal'lar uchun oddiy tekshirish. Lekin class'lar yoki complex type'lar bilan intersection ba'zan kutilmagandek bo'lishi mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Basic composition
type Timestamps = { createdAt: Date; updatedAt: Date };
type Entity = { id: string };

type Post = Entity & Timestamps & {
  title: string;
  content: string;
  authorId: string;
};

const post: Post = {
  id: "p1",
  createdAt: new Date(),
  updatedAt: new Date(),
  title: "Hello",
  content: "World",
  authorId: "u1",
};

// Mixin pattern
type Printable = { print(): void };
type Serializable = { serialize(): string };
type Cloneable<T> = { clone(): T };

type Document = Printable & Serializable & Cloneable<Document> & {
  content: string;
};

// Props composition
type BaseProps = { id: string; className?: string };
type InteractiveProps = { onClick: () => void; disabled?: boolean };
type StyledProps = { style?: Record<string, string> };

type ButtonProps = BaseProps & InteractiveProps & StyledProps & {
  label: string;
};

// Literal intersection — filter
type AllRoles = "admin" | "moderator" | "user" | "guest";
type PrivilegedRoles = "admin" | "moderator" | "superuser";

type Common = AllRoles & PrivilegedRoles;
// "admin" | "moderator"

// Primitive intersection — never
type Impossible = string & number; // never
// Amalda ishlatib bo'lmaydi

// Branded types — nominal typing simulation
type Brand<T, B> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

function createUserId(id: string): UserId {
  return id as UserId;
}

function createOrderId(id: string): OrderId {
  return id as OrderId;
}

const uid = createUserId("u1");
const oid = createOrderId("o1");

function getUser(id: UserId) { /* ... */ }
getUser(uid);  // ✅
// getUser(oid); // ❌ Type 'OrderId' is not assignable to 'UserId'
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type HasName = { name: string };
type HasAge = { age: number };
type Person = HasName & HasAge;

const p: Person = { name: "Ali", age: 25 };
```

```javascript
// Compiled JS — intersection o'chiriladi
const p = { name: "Ali", age: 25 };

// Intersection type — compile-time tushuncha
// JS'da oddiy object literal qoladi
```

</details>

---

## Intersection Bilan Conflict

### Nazariya

Agar ikki tip'ning **bir xil nomdagi** property'lari turli tipga ega bo'lsa — intersection'da **conflict** yuzaga keladi. Natija property tipiga bog'liq:

**1. Object property — rekursiv intersection:**

```typescript
type A = { config: { theme: string; debug: boolean } };
type B = { config: { theme: string; lang: string } };

type AB = A & B;
// config: { theme: string; debug: boolean } & { theme: string; lang: string }
// = config: { theme: string; debug: boolean; lang: string }
// Object property'lar ham intersection — recursive
```

**2. Primitive property conflict — `never`:**

```typescript
type X = { status: string };
type Y = { status: number };

type XY = X & Y;
// status: string & number = never
// Property mavjud, lekin qiymat qo'yib bo'lmaydi

// const xy: XY = { status: ??? }; // ❌ Hech qanday qiymat mos emas
```

**3. Literal type conflict — mos kelsa qoladi, mos kelmasa `never`:**

```typescript
type P = { role: "admin" | "user" };
type Q = { role: "admin" | "moderator" };

type PQ = P & Q;
// role: ("admin" | "user") & ("admin" | "moderator") = "admin"
// Faqat umumiy literal — "admin" qoldi

const pq: PQ = { role: "admin" }; // ✅
// const pq2: PQ = { role: "user" }; // ❌
```

<details>
<summary><strong>Under the Hood</strong></summary>

Intersection type hisoblash jarayoni:

```
A & B hisoblash:

1. Ikkala tipdagi barcha property'larni yig':
   - Faqat A'da: A'dan olish
   - Faqat B'da: B'dan olish
   - Ikkalasida ham: property tiplarini intersection qilish

2. Property type intersection:
   { a: string } & { a: string }     → { a: string }        (bir xil — qoladi)
   { a: string } & { a: number }     → { a: never }          (conflict — never)
   { a: {x: 1} } & { a: {y: 2} }     → { a: {x: 1; y: 2} }  (recursive)
   { a: "x" | "y" } & { a: "y" | "z" } → { a: "y" }         (literal filter)

3. Agar natijada `never` property bo'lsa:
   - Type o'zi never bo'lmaydi
   - Lekin shu property'ga qiymat assign qilib bo'lmaydi
   - Amalda tip ishlatib bo'lmaydigan holat
```

**Type theory — nega `string & number = never`:**

Tip'lar "qiymatlar to'plami" deb qaralsa:
- `string` = barcha string qiymatlari
- `number` = barcha number qiymatlari
- `string ∩ number` = ikkalasida ham bor bo'lgan qiymatlar = **bo'sh to'plam** = `never`

`never` — **bottom type**. TypeScript type theory'da empty set'ni ifodalaydi. "Hech qanday qiymat mavjud emas" degan ma'noni beradi. Bu mantiqiy to'g'ri: hech qanday qiymat bir vaqtda string ham number ham bo'la olmaydi (bigint'dan tashqari — alohida primitive).

**Real-world ogohlantiruvchi:**

```typescript
type ServerConfig = { port: number };
type AppConfig = { port: string }; // boshqa kodda — har xil semantic

type MergedConfig = ServerConfig & AppConfig;
// port: never — silent muammo

// const cfg: MergedConfig = { port: ??? }; // hech qachon assign bo'lmaydi
```

Bu dizayn xatosining belgisi — ikki kodbaza'ning bir xil nomli lekin farqli semantic'dagi tip'larini merge qilish. Yechim: property nomlarini farqlash (`httpPort`, `appPort`).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Case 1: Same type (no conflict)
type A = { x: string };
type B = { x: string };
type AB = A & B; // { x: string }
const ab: AB = { x: "hello" }; // ✅

// Case 2: Object property (recursive merge)
type C = { config: { host: string; port: number } };
type D = { config: { host: string; timeout: number } };
type CD = C & D;
// { config: { host: string; port: number; timeout: number } }
const cd: CD = {
  config: { host: "localhost", port: 3000, timeout: 5000 },
};

// Case 3: Primitive conflict (never)
type E = { status: string };
type F = { status: number };
type EF = E & F; // { status: never }
// const ef: EF = { status: ??? }; // impossible

// Case 4: Literal filter
type G = { role: "admin" | "user" };
type H = { role: "admin" | "moderator" };
type GH = G & H; // { role: "admin" }
const gh: GH = { role: "admin" }; // ✅

// Case 5: Subtype intersection
type I = { x: number };
type J = { x: 42 };
type IJ = I & J;
// { x: number & 42 } = { x: 42 } (42 ⊂ number)
const ij: IJ = { x: 42 }; // ✅

// Practical: avoiding conflict
type WithTimestamp = { createdAt: Date };
type WithModified = { modifiedAt: Date }; // farqli nom
type Entity = WithTimestamp & WithModified & {
  id: string;
};
// Sof merge, conflict yo'q
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type A = { config: { theme: string; debug: boolean } };
type B = { config: { theme: string; lang: string } };
type AB = A & B;

const ab: AB = {
  config: { theme: "dark", debug: true, lang: "uz" },
};
```

```javascript
// Compiled JS — intersection type butunlay o'chirildi
const ab = {
  config: { theme: "dark", debug: true, lang: "uz" },
};

// Merged properties — oddiy nested object
// Runtime'da hech qanday validation yo'q
```

</details>

---

## Discriminated Unions

### Nazariya

Discriminated union (tagged union, algebraic data type) — union'ning har member'ida umumiy **discriminant property** (tag) bo'ladi. Bu property **literal tipga ega** va har member'da **farqli qiymat** oladi. TypeScript compiler shu discriminant bo'yicha `switch` yoki `if` orqali member'ni aniq aniqlaydi.

Discriminated union — TypeScript'ning **eng kuchli** pattern'laridan biri. Redux action'lar, API response'lar, state machines, AST node'lar — barchasi shu pattern ustiga quriladi.

**Discriminated union'ning uchta sharti:**
1. Union'ning har member'ida **umumiy property** (discriminant)
2. Bu property **literal tipga ega** (`"circle"`, `"square"`, `42`, `true`)
3. `switch` yoki `if` orqali narrowing mumkin

```typescript
// Klassik misol — Shape
type Circle = { kind: "circle"; radius: number };
type Square = { kind: "square"; side: number };
type Rectangle = { kind: "rectangle"; width: number; height: number };

type Shape = Circle | Square | Rectangle;

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // shape: Circle
    case "square":
      return shape.side ** 2;              // shape: Square
    case "rectangle":
      return shape.width * shape.height;    // shape: Rectangle
  }
}
```

**Discriminant sifatida ishlatilishi mumkin:**
- **String literal:** `kind: "circle"`
- **Number literal:** `code: 404`
- **Boolean literal:** `success: true`
- **`null` / `undefined`:** `data: null`
- **Enum member:** `status: Status.Active`

**Discriminant bo'la olmaydi:**
- Keng tip'lar: `string`, `number`
- Non-literal union'lar
- Generic tip'lar

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript compiler discriminated union'ni qanday taniydi:

```
Discriminated Union aniqlash jarayoni:

1. Union type'ning barcha member'larini ko'radi
2. Umumiy property'larni topadi (har member'da bor)
3. Shu property har member'da LITERAL tipga ega ekanligini tekshiradi
4. Agar shartlar bajarilsa — bu property "discriminant" deb belgilanadi
5. switch/if'da shu property tekshirilganda, TS har case'da
   type'ni avtomatik toraytiradi
```

**`narrowTypeByDiscriminantProperty()`:**

Checker `switch (shape.kind)` ko'rilganda, har `case` label'iga qarab union'dan mos member'ni ajratadi:

```
switch (shape.kind) {
  case "circle":  → filter union where kind === "circle" → Circle
  case "square":  → filter union where kind === "square" → Square
  case "rectangle": → filter union where kind === "rectangle" → Rectangle
  default:  → union'dan barcha handled member'lar olib tashlangan → never
}
```

**Literal type requirement:**

Discriminant property **literal** bo'lishi kerak, chunki narrowing uchun compiler aniq `===` tekshiruvini qilishi kerak. Keng tiplar bilan bu mumkin emas:

```typescript
// ❌ Discriminant sifatida ishlamaydi
type A = { status: string; data: number };
type B = { status: string; error: string };

function handle(x: A | B) {
  if (x.status === "success") {
    // x: A | B (narrowing ishlamadi — "success" compile-time'da A yoki B bilan bog'liq emas)
  }
}

// ✅ Ishlaydi
type A2 = { status: "success"; data: number };
type B2 = { status: "error"; error: string };

function handle2(x: A2 | B2) {
  if (x.status === "success") {
    // x: A2 (narrowed) — literal type bilan
  }
}
```

**Exhaustiveness inference:**

Compiler `switch` statement'da barcha case'lar handle qilinganligini avtomatik aniqlaydi. `default` case'da qolgan union `never` bo'lishi kerak — agar biror case tushirilgan bo'lsa, `never` assignment xato beradi. Bu exhaustive checking'ning asosi (keyingi bo'limda batafsil).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

API Response pattern:

```typescript
type ApiSuccess<T> = {
  status: "success";
  data: T;
  timestamp: number;
};

type ApiError = {
  status: "error";
  error: { code: number; message: string };
  timestamp: number;
};

type ApiLoading = {
  status: "loading";
};

type ApiResponse<T> = ApiSuccess<T> | ApiError | ApiLoading;

function handleResponse<T>(response: ApiResponse<T>): void {
  switch (response.status) {
    case "success":
      console.log("Data:", response.data);
      break;
    case "error":
      console.log(`Error ${response.error.code}: ${response.error.message}`);
      break;
    case "loading":
      console.log("Loading...");
      break;
  }
}
```

State machine:

```typescript
type ConnectionState =
  | { status: "disconnected" }
  | { status: "connecting"; startedAt: Date }
  | { status: "connected"; connectedAt: Date; socket: WebSocket }
  | { status: "error"; error: Error; retryCount: number };

function getStatusMessage(state: ConnectionState): string {
  switch (state.status) {
    case "disconnected":
      return "Not connected";
    case "connecting":
      return `Connecting since ${state.startedAt.toISOString()}`;
    case "connected":
      return `Connected via ${state.socket.url}`;
    case "error":
      return `Error: ${state.error.message} (retry #${state.retryCount})`;
  }
}
```

Redux-style actions:

```typescript
type Action =
  | { type: "ADD_TODO"; payload: { text: string } }
  | { type: "TOGGLE_TODO"; payload: { id: number } }
  | { type: "DELETE_TODO"; payload: { id: number } }
  | { type: "SET_FILTER"; payload: { filter: "all" | "active" | "completed" } };

type Todo = { id: number; text: string; completed: boolean };

function todoReducer(state: Todo[], action: Action): Todo[] {
  switch (action.type) {
    case "ADD_TODO":
      return [...state, {
        id: Date.now(),
        text: action.payload.text,
        completed: false,
      }];
    case "TOGGLE_TODO":
      return state.map(t =>
        t.id === action.payload.id ? { ...t, completed: !t.completed } : t
      );
    case "DELETE_TODO":
      return state.filter(t => t.id !== action.payload.id);
    case "SET_FILTER":
      console.log("Filter:", action.payload.filter);
      return state;
  }
}
```

Result type (Rust-style):

```typescript
type Ok<T> = { ok: true; value: T };
type Err<E> = { ok: false; error: E };
type Result<T, E = Error> = Ok<T> | Err<E>;

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { ok: false, error: "Division by zero" };
  }
  return { ok: true, value: a / b };
}

const result = divide(10, 2);
if (result.ok) {
  console.log(result.value); // 5
} else {
  console.log(result.error);
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
  }
}
```

```javascript
// Compiled JS — switch/case va kind tag AYNAN qoladi
function area(shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
  }
}

// Type alias o'chirildi
// Discriminant property ("kind") oddiy JS string — runtime'da saqlanadi
// Narrowing — faqat compile-time, JS'da oddiy property access
```

</details>

---

## Custom Type Guards

### Nazariya

Ba'zan `typeof`, `instanceof`, `in` yetarli emas — murakkab tip tekshiruvi kerak bo'ladi. **Custom type guard** — `is` keyword bilan funksiya yaratish. Bu funksiya `true` qaytarsa, TypeScript parameter tipini belgilangan tipga toraytiradi.

**Syntax:**
```typescript
function isX(value: SomeType): value is SpecificType {
  return /* boolean expression */;
}
```

```typescript
type Fish = { swim: () => void; name: string };
type Bird = { fly: () => void; name: string };

function isFish(animal: Fish | Bird): animal is Fish {
  return "swim" in animal;
}

function move(animal: Fish | Bird): void {
  if (isFish(animal)) {
    animal.swim(); // animal: Fish (narrowed)
  } else {
    animal.fly();  // animal: Bird
  }
}
```

Custom type guard'ning afzalligi — **murakkab tekshiruv mantig'ini funksiyaga ajratish**. Bu kodni toza va qayta ishlatiladigan qiladi:

```typescript
type User = {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user";
};

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value &&
    "email" in value &&
    "role" in value &&
    typeof (value as User).id === "string" &&
    typeof (value as User).name === "string" &&
    typeof (value as User).email === "string" &&
    ((value as User).role === "admin" || (value as User).role === "user")
  );
}

function processApiData(data: unknown): void {
  if (isUser(data)) {
    console.log(data.name, data.role); // data: User
  } else {
    console.log("Invalid user data");
  }
}
```

**Muhim ogohlantiruv:** Custom type guard'da TypeScript **funksiya ichidagi mantiqni tekshirmaydi** — `is` keyword bilan siz TS'ga "menga ishon" deysiz. Agar mantiq noto'g'ri bo'lsa — runtime xato, lekin TS ogohlantirmaydi:

```typescript
// ⚠️ Mantiqiy xato, lekin TS xato bermaydi
function isString(value: unknown): value is string {
  return typeof value === "number"; // ❌ Noto'g'ri
}

const val: unknown = 42;
if (isString(val)) {
  val.toUpperCase(); // Runtime Error! val aslida number
}
```

> **Batafsil:** Custom type guard'lar va assertion functions haqida chuqur ma'lumot [06-type-narrowing.md](06-type-narrowing.md) da yoritiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`checker.ts`'da `is` predicate `TypePredicateKind.Identifier` sifatida parse qilinadi. Funksiya return type'i `value is Fish` bo'lganda, compiler uni oddiy `boolean` return emas, balki **type predicate** deb saqlaydi.

```
Custom Type Guard — checker.ts jarayoni:

Source                              Internal state
──────                              ──────────────
function isFish(a): a is Fish   →   FunctionDeclaration {
                                      returnType: TypePredicate {
                                        kind: Identifier,
                                        parameterName: "a",
                                        parameterIndex: 0,
                                        type: FishType
                                      }
                                    }

if (isFish(animal)):
  getTypePredicateOfSignature()
  → parameterIndex: 0 → animal topildi
  → true branch:  animal → FishType
  → false branch: animal → union'dan Fish olib tashlash → BirdType
```

**Compiler trust model:**

Checker funksiya **ichidagi mantiqni tekshirmaydi** — `is` predicate'ga ishonadi. Bu "trust-based" contract: developer type guard'ning to'g'ri implement qilinganligiga javobgar.

Agar mantiq noto'g'ri bo'lsa (masalan, `return typeof x === "number"` lekin return type `x is string`), TypeScript bu yerda xato bermaydi. Runtime'da esa, funksiya `true` qaytargan holatda narrowing'dan keyin xato kod ishlaydi — `string` method'lari `number` qiymatda chaqiriladi va crash bo'ladi.

Bu **type safety gap** — TypeScript bilan bo'lgan kelishuv'ning bir qismi. Custom type guard'lar kuchli, lekin ehtiyot bilan yozilishi kerak.

**Runtime shape check patterns:**

Yaxshi custom type guard'lar odatda bosqichma-bosqich tekshiruvni amalga oshiradi:

1. `typeof === "object"` va `!== null`
2. Har kerakli property mavjudligi (`"x" in value`)
3. Har property'ning tipi to'g'ri ekanligi (`typeof (value as X).x === "string"`)
4. Literal union'lar (`(value as X).role === "admin" || ...`)

Murakkabroq validation uchun **Zod**, **io-ts**, **runtypes** kabi library'lar ishlatiladi — ular schema'dan type guard va runtime validator'larni avtomatik yaratadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Simple discriminating guard
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function isFish(animal: Fish | Bird): animal is Fish {
  return "swim" in animal;
}

// Complex object validation
interface Product {
  id: string;
  name: string;
  price: number;
  category: "electronics" | "clothing" | "food";
  inStock: boolean;
}

function isProduct(value: unknown): value is Product {
  if (typeof value !== "object" || value === null) return false;
  const v = value as Record<string, unknown>;
  return (
    typeof v.id === "string" &&
    typeof v.name === "string" &&
    typeof v.price === "number" &&
    (v.category === "electronics" || v.category === "clothing" || v.category === "food") &&
    typeof v.inStock === "boolean"
  );
}

// Array type guard
function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every(item => typeof item === "string");
}

// Generic type guard
function isArrayOf<T>(
  value: unknown,
  check: (item: unknown) => item is T
): value is T[] {
  return Array.isArray(value) && value.every(check);
}

// Usage
const data: unknown = [{ id: "1", name: "Apple", price: 100, category: "food", inStock: true }];
if (isArrayOf(data, isProduct)) {
  data.forEach(p => console.log(p.name)); // data: Product[]
}

// Result type guard
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function isOk<T, E>(result: Result<T, E>): result is { ok: true; value: T } {
  return result.ok;
}

const res: Result<number, string> = { ok: true, value: 42 };
if (isOk(res)) {
  console.log(res.value * 2); // res: { ok: true; value: number }
}

// Using Zod (real-world)
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

type User = z.infer<typeof UserSchema>;

function parseUser(data: unknown): User {
  return UserSchema.parse(data); // throws if invalid
}

// Or type guard
function isUser(data: unknown): data is User {
  return UserSchema.safeParse(data).success;
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function isFish(animal: Fish | Bird): animal is Fish {
  return "swim" in animal;
}
function move(animal: Fish | Bird): void {
  if (isFish(animal)) {
    animal.swim();
  } else {
    animal.fly();
  }
}
```

```javascript
// Compiled JS — "animal is Fish" predicate o'chiriladi
function isFish(animal) {
  return "swim" in animal;
}
function move(animal) {
  if (isFish(animal)) {
    animal.swim();
  } else {
    animal.fly();
  }
}

// Type predicate faqat compile-time
// Runtime'da oddiy boolean-qaytaruvchi funksiya
```

</details>

---

## Exhaustive Checking

### Nazariya

Exhaustive checking — union tipi'ning **barcha** member'lari handle qilinganligini **compile-time'da** tekshirish. `switch` yoki `if-else` zanjirida barcha holatlar ko'rib chiqilgandan keyin, qolgan tip `never` bo'lishi kerak. Agar yangi member qo'shilsa va handle qilinmasa — TypeScript **compile-time xato** beradi.

Bu pattern production'da juda muhim: yangi variant qo'shganda, barcha ishlov beruvchi joylarni topib yangilash — **compiler** ta'minlaydi. Silent bug'lar o'rniga kompilyatsiya xatosi.

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default: {
      // Barcha holatlar handle qilingan — shape bu yerda `never`
      const _exhaustiveCheck: never = shape;
      throw new Error(`Unhandled shape: ${JSON.stringify(shape)}`);
      //                                              ^^^^^^
      //                             runtime'da haqiqiy qiymatni chiqarish
      //                             (_exhaustiveCheck type'i `never`, lekin runtime'da = shape)
    }
  }
}
```

**Yangi variant qo'shish — muhim misol:**

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number }
  | { kind: "pentagon"; side: number; count: 5 }; // ← YANGI

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default: {
      const _exhaustiveCheck: never = shape;
      // ❌ Type '{ kind: "pentagon"; ... }' is not assignable to type 'never'
      // TS ogohlantiradi: "pentagon" handle qilinmagan!
      throw new Error(`Unhandled shape: ${JSON.stringify(shape)}`);
    }
  }
}
```

Bu **compile-time** xato. Siz yangi variant qo'shdingiz va handle qilishni **unutdingiz** — TypeScript bu xatoni topadi. Production kodda bu juda qimmatli: silent bug'lar o'rniga CI/build'da to'xtash.

<details>
<summary><strong>Under the Hood</strong></summary>

**`never` type — bottom type:**

`never` TypeScript type system'da **bo'sh to'plam** (empty set). Hech qanday qiymat `never` tipga tegishli emas. Union tipida barcha member'lar eliminate qilingandan keyin — hech narsa qolmaydi, ya'ni `never`.

```
Exhaustive check jarayoni:

shape: Circle | Square | Triangle

case "circle":  → shape: Square | Triangle    (Circle olib tashlandi)
case "square":  → shape: Triangle              (Square olib tashlandi)
case "triangle": → shape: never                (Triangle olib tashlandi, bo'sh)

default: shape = never ✅ — barcha holatlar handle qilingan
```

**`const _exhaustiveCheck: never = shape` mexanizmi:**

Bu pattern ikki qavat xavfsizlik beradi:

1. **Compile-time:** `shape`'ni `never`'ga assign qilib bo'lmasligi — TS xato. Bu faqat barcha case'lar handle qilingan holatda ishlaydi, aks holda `shape` hali ham union'ning qolgan qismi.

2. **Runtime:** `throw new Error(...)` — agar runtime'da kutilmagan qiymat kelsa (masalan, API dan noto'g'ri payload), xato throw bo'ladi. Bu safety net.

**Muhim nuance:** `_exhaustiveCheck` o'zgaruvchi type'i `never`, lekin runtime'da `shape`'ning qiymatini saqlaydi. `never` faqat compile-time konstruksiya — runtime'da variable oddiy JavaScript holat.

Shuning uchun xato xabari'da `_exhaustiveCheck` emas, `JSON.stringify(shape)` ishlatish kerak — `shape` runtime'da haqiqiy qiymatni saqlaydi.

**`throw` importance:**

Ba'zi misollarda `return _exhaustiveCheck` ishlatiladi, lekin bu **kam xavfsiz**:

```typescript
// Kam xavfsiz
default:
  const _check: never = shape;
  return _check; // runtime'da `_check` undefined qaytaradi
```

Agar runtime'da kutilmagan qiymat kelsa (masalan, `compileTime` va `runtime` sinxron emas), funksiya `undefined` qaytaradi — silent bug. `throw` bilan esa aniq error paydo bo'ladi.

**Best practice:**

```typescript
default: {
  const _exhaustiveCheck: never = shape;
  throw new Error(`Unhandled case: ${JSON.stringify(shape)}`);
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Basic exhaustive check
type Status = "active" | "inactive" | "pending" | "deleted";

function getStatusLabel(status: Status): string {
  switch (status) {
    case "active": return "Active";
    case "inactive": return "Inactive";
    case "pending": return "Pending";
    case "deleted": return "Deleted";
    default: {
      const _exhaustive: never = status;
      throw new Error(`Unhandled status: ${status}`);
    }
  }
}

// State machine exhaustive
type AppState =
  | { screen: "home" }
  | { screen: "profile"; userId: string }
  | { screen: "settings"; tab: "general" | "security" }
  | { screen: "error"; message: string };

function renderScreen(state: AppState): string {
  switch (state.screen) {
    case "home":
      return "Welcome home!";
    case "profile":
      return `Profile of user ${state.userId}`;
    case "settings":
      return `Settings: ${state.tab}`;
    case "error":
      return `Error: ${state.message}`;
    default: {
      const _exhaustive: never = state;
      throw new Error(`Unknown screen: ${JSON.stringify(state)}`);
    }
  }
}

// Reducer exhaustive
type Action =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "SET"; payload: number }
  | { type: "RESET" };

function counterReducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT":
      return state + 1;
    case "DECREMENT":
      return state - 1;
    case "SET":
      return action.payload;
    case "RESET":
      return 0;
    default: {
      const _exhaustive: never = action;
      throw new Error(`Unknown action: ${JSON.stringify(action)}`);
    }
  }
}

// If-else exhaustive (alternative)
type Theme = "light" | "dark" | "auto";

function applyTheme(theme: Theme): string {
  if (theme === "light") return "#fff";
  if (theme === "dark") return "#000";
  if (theme === "auto") return "system";
  const _exhaustive: never = theme;
  throw new Error(`Unknown theme: ${theme}`);
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    default: {
      const _exhaustiveCheck: never = shape;
      throw new Error(`Unhandled: ${JSON.stringify(shape)}`);
    }
  }
}
```

```javascript
// Compiled JS — ": never" o'chiriladi, throw saqlanadi
function getArea(shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    default: {
      const _exhaustiveCheck = shape;
      throw new Error(`Unhandled: ${JSON.stringify(shape)}`);
    }
  }
}

// MUHIM: throw bo'lmasa, default branch silently undefined qaytarib
// runtime'da topish qiyin bug'larga olib kelishi mumkin
// Exhaustive check pattern'da DOIM throw qiling
```

</details>

---

## assertNever Pattern

### Nazariya

`assertNever` — exhaustive checking'ni **qayta ishlatiladigan funksiya** shaklida yozish. Funksiya `never` tipli parameter qabul qiladi va `throw` qiladi. TypeScript `never` parameter'ni faqat barcha case'lar handle qilingan holda qabul qiladi — shuning uchun bu compile-time safety.

```typescript
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

type PaymentMethod =
  | { type: "credit_card"; cardNumber: string; cvv: string }
  | { type: "paypal"; email: string }
  | { type: "bank_transfer"; iban: string };

function processPayment(method: PaymentMethod): void {
  switch (method.type) {
    case "credit_card":
      console.log(`Card: ${method.cardNumber}`);
      break;
    case "paypal":
      console.log(`PayPal: ${method.email}`);
      break;
    case "bank_transfer":
      console.log(`IBAN: ${method.iban}`);
      break;
    default:
      assertNever(method);
      // Yangi payment method qo'shilsa va case qo'shilmasa — TS xato
      // Runtime'da kutilmagan kirsa — throw Error
  }
}
```

`assertNever`'ning `never` qaytarish tipi — uni **expression** sifatida ham ishlatish imkonini beradi:

```typescript
function getPaymentLabel(method: PaymentMethod): string {
  switch (method.type) {
    case "credit_card": return "Credit Card";
    case "paypal": return "PayPal";
    case "bank_transfer": return "Bank Transfer";
    default: return assertNever(method);
    // `never` qaytaradi — har qanday tipga assign qilish mumkin
  }
}
```

Bu pattern discriminated union'lar bilan eng ko'p ishlatiladigan **exhaustiveness guarantee** mexanizmi. Kodbazalarning `utils.ts` yoki `type-helpers.ts` faylida `assertNever` odatda birinchi yoziladigan helper.

<details>
<summary><strong>Under the Hood</strong></summary>

`assertNever` pattern **ikki darajada** ishlaydi — compile-time type checking va runtime error throwing.

**Compile-time check:**

```
switch (method.type):
  case "credit_card": → method: PayPal | BankTransfer
  case "paypal":       → method: BankTransfer
  case "bank_transfer": → method: never (barcha handled)

default:
  assertNever(method) → method: never assignable to parameter: never → HA ✅

Agar "crypto" case qo'shilsa lekin handle qilinmasa:
default:
  assertNever(method) → method: { type: "crypto"; ... } assignable to never → YO'Q ❌
```

Compiler `assertNever(method)` chaqiruvini ko'rganda, `method`'ning joriy tipini `never` parameter tipiga assign qilib bo'ladimi tekshiradi. Agar barcha case'lar handle qilingan bo'lsa — `method` `never`, assign ishlaydi. Aks holda — compile error.

**Runtime behavior:**

`assertNever` oddiy JS funksiya — `throw new Error(...)` qiladi. Compiled JS'da funksiya sifatida qoladi, type annotation'lar o'chiriladi, lekin `throw` statement ishlaydi.

**`never` return type:**

`assertNever` return type'i `never`. Bu checker uchun maxsus holat: `never` qaytaruvchi funksiya chaqirilgandan keyin, compiler keyingi kodni **unreachable** deb belgilaydi. Shuning uchun:

```typescript
function getLabel(x: PaymentMethod): string {
  switch (x.type) {
    case "credit_card": return "CC";
    case "paypal": return "PP";
    case "bank_transfer": return "BT";
    default: return assertNever(x); // ✅ ishlaydi
    // `never` ham `string`'ga assign bo'ladi (bottom type)
  }
}
```

`never` **har qanday** tipga assign bo'ladi — shuning uchun `return assertNever(x)` har xil return type'larda ishlaydi.

**Arrow function variant:**

```typescript
const assertNever = (value: never): never => {
  throw new Error(`Unexpected: ${value}`);
};
```

Ikkalasi ham bir xil ishlaydi — funksiya declaration'iga qarshi arrow'ni afzal ko'rish stilistik qaror.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Utility file: type-helpers.ts
export function assertNever(value: never, message?: string): never {
  throw new Error(
    message ?? `Exhaustive check failed: ${JSON.stringify(value)}`
  );
}

// Usage: state machine
type AuthState =
  | { status: "unauthenticated" }
  | { status: "authenticating"; credentials: { email: string } }
  | { status: "authenticated"; user: { id: string; name: string } }
  | { status: "error"; error: Error };

function getAuthLabel(state: AuthState): string {
  switch (state.status) {
    case "unauthenticated":
      return "Please log in";
    case "authenticating":
      return `Logging in as ${state.credentials.email}`;
    case "authenticated":
      return `Welcome, ${state.user.name}`;
    case "error":
      return `Error: ${state.error.message}`;
    default:
      return assertNever(state);
  }
}

// Usage: reducer
type Event =
  | { type: "mouseClick"; x: number; y: number }
  | { type: "keyPress"; key: string }
  | { type: "scroll"; delta: number };

function handleEvent(event: Event): void {
  switch (event.type) {
    case "mouseClick":
      console.log(`Click at ${event.x}, ${event.y}`);
      break;
    case "keyPress":
      console.log(`Key: ${event.key}`);
      break;
    case "scroll":
      console.log(`Scroll: ${event.delta}`);
      break;
    default:
      assertNever(event);
  }
}

// Usage: component renderer
type ComponentType =
  | { kind: "button"; label: string }
  | { kind: "input"; placeholder: string }
  | { kind: "image"; src: string; alt: string };

function renderComponent(component: ComponentType): string {
  switch (component.kind) {
    case "button":
      return `<button>${component.label}</button>`;
    case "input":
      return `<input placeholder="${component.placeholder}" />`;
    case "image":
      return `<img src="${component.src}" alt="${component.alt}" />`;
    default:
      return assertNever(component);
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function assertNever(value: never, msg?: string): never {
  throw new Error(msg ?? `Unexpected value: ${JSON.stringify(value)}`);
}

type Method = { type: "card"; num: string } | { type: "paypal"; email: string };

function process(m: Method): void {
  switch (m.type) {
    case "card": console.log(m.num); break;
    case "paypal": console.log(m.email); break;
    default: assertNever(m);
  }
}
```

```javascript
// Compiled JS — assertNever funksiya sifatida qoladi
function assertNever(value, msg) {
  throw new Error(msg ?? `Unexpected value: ${JSON.stringify(value)}`);
}

function process(m) {
  switch (m.type) {
    case "card": console.log(m.num); break;
    case "paypal": console.log(m.email); break;
    default: assertNever(m);
  }
}

// ": never" o'chirildi, lekin funksiya va throw — runtime safety net sifatida qoldi
// Runtime'da kutilmagan qiymat kelsa — throw bilan aniq xato
```

</details>

---

## Union Types va Function Overloads

### Nazariya

Ba'zan bir funksiya turli parameter tiplari uchun turli return tiplar qaytarishi kerak. Bu holda ikki yondashuv bor:

1. **Union type** — agar return type **har doim bir xil** yoki parameter'ga bog'liq emas
2. **Function overloads** — agar return type **parameter'ga bog'liq**

**Union type — yetarli bo'lgan holat:**

```typescript
function formatValue(value: string | number): string {
  if (typeof value === "string") return value.toUpperCase();
  return value.toFixed(2);
}

// Return type har doim string — union yetarli
const a = formatValue("hello"); // string
const b = formatValue(42);      // string
```

**Function overloads — parameter'ga bog'liq return:**

```typescript
// Overload signatures
function parseInput(input: string): string[];
function parseInput(input: number): number[];
// Implementation signature
function parseInput(input: string | number): string[] | number[] {
  if (typeof input === "string") {
    return input.split(",");
  }
  return [input, input * 2];
}

const strings = parseInput("a,b,c"); // type: string[] — aniq
const numbers = parseInput(42);       // type: number[] — aniq
// Union bo'lganida ikkalasi ham string[] | number[] bo'lardi
```

**Overload resolution — "first match wins":**

Compiler overload'larni **yuqoridan pastga** iteratsiya qiladi va birinchi mos keluvchi'ni tanlaydi. Shuning uchun **eng spetsifik overload'lar birinchi yozilishi kerak**:

```typescript
// ✅ To'g'ri tartib — spetsifik birinchi
function fn(x: "hello"): 1;
function fn(x: string): 2;
function fn(x: string | "hello"): 1 | 2 {
  return x === "hello" ? 1 : 2;
}

fn("hello"); // type: 1 (birinchi mos)
fn("world"); // type: 2 (ikkinchi)

// ❌ Noto'g'ri tartib — umumiy birinchi
function fn2(x: string): 2;
function fn2(x: "hello"): 1;
// fn2("hello") → type: 2 (umumiy string birinchi mos keldi)
// Ikkinchi "hello" overload'ga hech qachon yetib bormaydi
```

| Holat | Yechim |
|-------|--------|
| Return type parametrga bog'liq emas | Union type |
| Return type parametrga bog'liq | Function overloads |
| 1-2 oddiy variant | Union type (sodda) |
| 3+ variant, har biri farqli return | Overloads yoki generic conditional |
| Input-output mapping kerak | Overloads |
| Type-level mapping | Generic conditional type |

> **Batafsil:** Function overloads, generic, conditional return types [07-functions.md](07-functions.md) da batafsil yoritiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Union type va function overloads — `checker.ts`'da **butunlay boshqa** mexanizmlar.

**Union type resolution:**

Union type bilan funksiya chaqirilganda, checker oddiy type checking qiladi — argument union type'ga assignable bo'lsa, call ishlaydi. Return type **har doim** deklaratsiya qilingan return type (masalan `string`). Compiler argument'ning aniq tipini return type'ni aniqlash uchun **ishlatmaydi**.

```
formatValue("hello"):
  - parameter type: string | number
  - argument: "hello" (string literal)
  - "hello" assignable to string | number? → HA
  - return type: string (deklaratsiyadan)
```

**Overload resolution algoritmi:**

```
parseInput("hello") chaqirilganda:

1. Signature: parseInput(input: string): string[]
   → "hello" assignable to string? → HA → TANLANDI ✅
   → Return type: string[]

2. Keyingi signature tekshirilmaydi (first match)

parseInput(42) chaqirilganda:

1. Signature: parseInput(input: string): string[]
   → 42 assignable to string? → YO'Q → keyingisiga

2. Signature: parseInput(input: number): number[]
   → 42 assignable to number? → HA → TANLANDI ✅
   → Return type: number[]
```

**Overload eshigi:**

Har overload signature **alohida** yoziladi, oxirida esa **implementation signature** (implementation'ning haqiqiy type'i). Implementation signature caller'larga **ko'rinmaydi** — faqat internal. Caller'lar faqat overload signature'larni ko'radi.

```typescript
// Bu "public API" — caller'lar ko'radi
function parseInput(input: string): string[];
function parseInput(input: number): number[];

// Bu "implementation" — caller'lar ko'rmaydi (private)
function parseInput(input: string | number): string[] | number[] {
  // ...
}
```

Shuning uchun implementation signature **kengroq** (union) va return type'lari **kengroq** (union) bo'ladi — u barcha overload'ni qoplashi kerak.

**Compiled JS — overload'lar yo'qoladi:**

Compiled JS'da **faqat implementation** qoladi — overload signature'lar butunlay o'chiriladi. JavaScript'da overloading tushunchasi yo'q. Runtime'da bitta funksiya ishlaydi, ichida `typeof` yoki boshqa tekshiruvlar orqali argument type aniqlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Case 1: Union yetarli
function format(value: string | number): string {
  if (typeof value === "string") return value.trim();
  return value.toString();
}

// Case 2: Overloads kerak — return bog'liq
function parse(input: string): string[];
function parse(input: number): number[];
function parse(input: boolean): boolean;
function parse(input: string | number | boolean): string[] | number[] | boolean {
  if (typeof input === "string") return input.split(",");
  if (typeof input === "number") return [input];
  return !input;
}

const s = parse("a,b,c");  // string[]
const n = parse(42);        // number[]
const b = parse(true);      // boolean

// Case 3: DOM API (real-world)
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "span"): HTMLSpanElement;
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const div = createElement("div");    // HTMLDivElement
const input = createElement("input"); // HTMLInputElement
const custom = createElement("foo");  // HTMLElement (fallback)

// Case 4: Generic conditional (advanced)
type ParseResult<T> =
  T extends string ? string[] :
  T extends number ? number[] :
  T extends boolean ? boolean :
  never;

function parseGeneric<T extends string | number | boolean>(input: T): ParseResult<T> {
  // implementation
  if (typeof input === "string") return input.split(",") as ParseResult<T>;
  if (typeof input === "number") return [input] as ParseResult<T>;
  return !input as ParseResult<T>;
}

// Generic bilan ham return type aniq:
const gs = parseGeneric("a,b,c"); // string[]
const gn = parseGeneric(42);       // number[]
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS — overload signatures + implementation
function parseInput(input: string): string[];
function parseInput(input: number): number[];
function parseInput(input: string | number): string[] | number[] {
  if (typeof input === "string") return input.split(",");
  return [input, input * 2];
}
```

```javascript
// Compiled JS — overload signatures butunlay o'chirildi
function parseInput(input) {
  if (typeof input === "string") return input.split(",");
  return [input, input * 2];
}

// Ikki overload signature yo'qoldi — JS'da bitta funksiya qoldi
// Runtime'da typeof orqali argument turi aniqlanadi
```

</details>

---

## satisfies bilan Validation

### Nazariya

`satisfies` operator (TS 4.9+) — qiymatning tipini **tekshiradi**, lekin tipni **kengaytirmaydi**. Bu union va literal type'lar bilan ishlaganda juda foydali — har qiymatning aniq literal tipi saqlanadi.

```typescript
type Color = "red" | "green" | "blue";

// ❌ Type annotation — literal type yo'qoladi
const palette: Record<Color, string> = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
};
// palette.red type: string (literal emas)

// ✅ satisfies — tip tekshiriladi, literal saqlanadi
const palette2 = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
} satisfies Record<Color, string>;
// palette2.red type: "#ff0000" (literal type saqlandi!)
```

**Uch yondashuv farqi:**

```typescript
// 1. Type annotation — literal yo'qoladi
const a: Record<string, string> = { x: "hello" };
// a.x type: string

// 2. as const — literal, lekin tip tekshirilmaydi
const b = { x: "hello" } as const;
// b.x type: "hello", lekin Record<string, string> constraint'ga mos ekanligi tekshirilmagan

// 3. satisfies — ikkalasi ham: literal saqlanadi VA constraint tekshiriladi
const c = { x: "hello" } satisfies Record<string, string>;
// c.x type: "hello" (literal) — va Record'ga mos ekanligi tekshirilgan
```

**`satisfies` enum/union validation uchun:**

```typescript
type Status = "active" | "inactive" | "pending";

// Majburiy: har Status uchun label
const STATUS_LABELS = {
  active: "User is active",
  inactive: "User is inactive",
  pending: "Waiting for activation",
} satisfies Record<Status, string>;

// Agar biror status tushirib qoldirilsa:
// const bad = {
//   active: "...",
//   inactive: "...",
//   // pending yo'q
// } satisfies Record<Status, string>;
// ❌ Property 'pending' is missing

// Literal type saqlanadi:
type ActiveLabel = typeof STATUS_LABELS["active"];
// type: "User is active" (literal)
```

> **Batafsil:** `satisfies` operator, `as const`, type annotation farqi, narrowing effekti [06-type-narrowing.md](06-type-narrowing.md) da yoritiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`satisfies` operator compiler'da **ikki bosqichli** tekshiruv amalga oshiradi:

1. **Infer** — avval expression'ning "natural type"'ini aniqlaydi (widening qilmasdan)
2. **Check** — inferred tipni `satisfies` dan keyingi tipga **assignability** tekshiradi
3. **Preserve** — check o'tsa, inferred tipni saqlaydi (annotation emas)

```
Annotation vs satisfies — compiler behavior:

Annotation:
  checkAssignability(value, annotationType) → variable has annotationType
  a: Record<string, string> = { x: "hello" }
  → a.x type: string (annotation'dan)

satisfies:
  checkAssignability(value, satisfiesType) → variable has inferredType
  b = { x: "hello" } satisfies Record<string, string>
  → b.x type: "hello" (inferred'dan)
```

**Union narrowing bilan `satisfies`:**

```typescript
type Theme = "light" | "dark";
const config = { theme: "light" } satisfies { theme: Theme };
// config.theme type: "light" (narrowed literal, Theme emas!)
```

Bu holat juda kuchli — `satisfies` ham validation (Theme'ga mos), ham narrowing (aniq "light") beradi.

**Runtime:** `satisfies` JS'da iz qoldirmaydi — pure type-level operator. Emit bosqichida butunlay o'chiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Enum replacement with satisfies
type Status = "active" | "inactive" | "pending";

const STATUS_LABELS = {
  active: "User is active",
  inactive: "User is inactive",
  pending: "Waiting for activation",
} satisfies Record<Status, string>;

// Validation pattern
type Route = `/${string}`;
const ROUTES = {
  home: "/",
  about: "/about",
  contact: "/contact",
} satisfies Record<string, Route>;

// Har value `/...` template pattern'iga mos bo'lishi kerak
// ROUTES.home type: "/" (literal)

// Config object
type ConfigKey = "host" | "port" | "timeout";

const config = {
  host: "localhost",
  port: 3000,
  timeout: 5000,
} satisfies Record<ConfigKey, string | number>;

// config.host type: "localhost" (literal)
// config.port type: 3000 (literal)

// Discriminated union validation
type Action = { type: "ADD" } | { type: "REMOVE" } | { type: "UPDATE" };

const actions = {
  add: { type: "ADD" },
  remove: { type: "REMOVE" },
  update: { type: "UPDATE" },
} satisfies Record<string, Action>;

// actions.add type: { type: "ADD" } — narrowed!
// Agar biror action tipi noto'g'ri bo'lsa:
// const bad = {
//   add: { type: "WRONG" }, // ❌
// } satisfies Record<string, Action>;
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
const palette = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
} satisfies Record<"red" | "green" | "blue", string>;
```

```javascript
// Compiled JS — satisfies butunlay o'chiriladi
const palette = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
};

// satisfies faqat compile-time validation
// Runtime'da oddiy object
```

</details>

---

## Edge Cases va Gotchas

### 🕳 Gotcha 1: Union narrowing closure ichida yo'qoladi

```typescript
type Shape = { kind: "circle"; radius: number } | { kind: "square"; side: number };

function process(shape: Shape) {
  if (shape.kind === "circle") {
    // shape: Circle (narrowed)
    setTimeout(() => {
      // ⚠️ Closure ichida: shape hali ham Shape (narrowing yo'qoladi!)
      // shape.radius; // ❌ Property 'radius' does not exist on type 'Shape'
    }, 100);
  }
}

// Yechim: variable'ga saqlash
function processFixed(shape: Shape) {
  if (shape.kind === "circle") {
    const circle = shape; // narrowed type saqlanadi
    setTimeout(() => {
      console.log(circle.radius); // ✅ circle: Circle
    }, 100);
  }
}
```

**Sabab:** TypeScript control flow analysis closure chegarasidan o'ta olmaydi. Callback ichida o'zgaruvchi o'zgarishi mumkin deb hisoblaydi — shuning uchun narrowing "eskiradi". Batafsil [06-type-narrowing.md](06-type-narrowing.md) da.

---

### 🕳 Gotcha 2: `typeof null === "object"` narrowing'da chiqarilmaydi

```typescript
function check(val: string | object | null) {
  if (typeof val === "object") {
    // val: object | null — null HAM chiqarilmadi!
    // val.toString(); // ❌ 'val' is possibly 'null'
  }

  if (typeof val === "object" && val !== null) {
    // val: object (endi null emas)
    val.toString(); // ✅
  }
}
```

**Sabab:** JavaScript'ning 1995-yildan beri mavjud legacy bug — `typeof null === "object"`. TypeScript narrowing bu xulqni aks ettiradi va `null`'ni `"object"` case'idan chiqarmaydi. Har doim `val !== null` qo'shimcha check kerak.

---

### 🕳 Gotcha 3: Intersection — primitive va object aralash

```typescript
type A = string;
type B = { length: number };

type AB = A & B;
// Texnik jihatdan: string bilan { length: number }
// string da length bor — compile qiladi
// Lekin amalda object yaratib bo'lmaydi:
// const x: AB = ??? // impossible
```

**Sabab:** Primitive va object'ni intersection qilish noaniq holat. `string` primitive, `{ length: number }` object — ikkalasi bir vaqtda bo'la olmaydi. Compiler bu narsa'ni "technically valid" deb qabul qiladi (chunki `string` o'zi `{ length: number }` shape'iga ega), lekin runtime'da hech qanday qiymat bunday tipda bo'lmaydi. Bu "unsoundness" holati.

---

### 🕳 Gotcha 4: Discriminated union — discriminant property yo'q

```typescript
type A = { status: "success"; data: string };
type B = { status: "error"; error: string };
type C = { data: number }; // ⚠️ status property yo'q

type Response = A | B | C;

function handle(r: Response) {
  if (r.status === "success") {
    // ❌ Property 'status' does not exist on type 'C'
    // C'da status yo'q — switch discriminant qurilmaydi
  }
}
```

**Sabab:** Discriminated union ishlashi uchun **har** member'da bir xil discriminant property bo'lishi shart. Bitta member'da property yo'q bo'lsa, narrowing butunlay ishlamaydi. Yechim: har member'ga discriminant qo'shish yoki pattern'ni qayta ko'rib chiqish.

---

### 🕳 Gotcha 5: Non-discriminated union — faqat umumiy property'lar

```typescript
type Book = { title: string; pages: number; author: string };
type Movie = { title: string; duration: number; director: string };

type Media = Book | Movie;

function describe(media: Media) {
  console.log(media.title); // ✅ umumiy
  // console.log(media.pages); // ❌ faqat Book'da
  // console.log(media.duration); // ❌ faqat Movie'da

  // Narrowing kerak — discriminant yo'q, "in" bilan
  if ("pages" in media) {
    console.log(media.pages);   // media: Book
  } else {
    console.log(media.duration); // media: Movie
  }
}
```

**Sabab:** Discriminant (tagged) bo'lmasa, union'da faqat **umumiy** property'larga kirish mumkin. Qolgan property'lar uchun `in` operator yoki custom type guard bilan narrowing kerak. Bu yerda discriminated union (`type: "book"` va `type: "movie"`) afzalroq — har member'ning property'lari darhol accessible.

---

## Common Mistakes

### ❌ Xato 1: Union'da narrowing'siz member-specific property ishlatish

```typescript
function process(value: string | number): void {
  // ❌ toUpperCase faqat string'da
  console.log(value.toUpperCase());
  // Error: Property 'toUpperCase' does not exist on type 'number'
}

// ✅ Narrowing bilan
function processFixed(value: string | number): void {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
```

**Nima uchun:** Union'da TS faqat **barcha member'larga umumiy** property'larni ko'radi. Specific property uchun avval narrowing shart.

---

### ❌ Xato 2: Intersection conflict'ni sezmaslik

```typescript
type Config = { port: string } & { port: number };
// Config.port: string & number = never
// Type mavjud, lekin amalda ishlatib bo'lmaydi

// ✅ Property nomlarini farqlang
type ServerConfig = { httpPort: number };
type AppConfig = { appPort: string };
type Config2 = ServerConfig & AppConfig;
const cfg: Config2 = { httpPort: 3000, appPort: "8080" };
```

**Nima uchun:** Bir xil nomli property'lar turli tipga ega bo'lsa, intersection ularni ham intersect qiladi — natija `never`. Property ga qiymat qo'yib bo'lmaydi.

---

### ❌ Xato 3: Discriminated union'da discriminant'ni literal type qilmaslik

```typescript
// ❌ Keng tip — narrowing ishlamaydi
type Result =
  | { status: string; data: unknown }
  | { status: string; error: string };

function handle(r: Result) {
  if (r.status === "success") {
    // r hali ham Result — narrowing ishlamadi
  }
}

// ✅ Literal type
type Result2 =
  | { status: "success"; data: unknown }
  | { status: "error"; error: string };

function handle2(r: Result2) {
  if (r.status === "success") {
    console.log(r.data); // ✅ narrowed
  }
}
```

**Nima uchun:** TS discriminant sifatida faqat **literal type**'larni taniydi. `string` keng tip — discriminant bo'la olmaydi.

---

### ❌ Xato 4: Exhaustive check qo'ymaslik

```typescript
type Status = "active" | "inactive" | "suspended";

// ❌ "suspended" case yo'q — TS xato bermaydi
function getLabel(s: Status): string {
  if (s === "active") return "Active";
  if (s === "inactive") return "Inactive";
  return "Unknown"; // "suspended" bu yerga tushadi — noto'g'ri
}

// ✅ Exhaustive check
function getLabelFixed(s: Status): string {
  switch (s) {
    case "active": return "Active";
    case "inactive": return "Inactive";
    case "suspended": return "Suspended";
    default: {
      const _check: never = s;
      throw new Error(`Unhandled: ${s}`);
    }
  }
}
```

**Nima uchun:** Exhaustive check'siz yangi variant qo'shganda TS ogohlantirmaydi — bug yashirinib qoladi.

---

### ❌ Xato 5: Custom type guard'da noto'g'ri mantiq

```typescript
type Cat = { meow: () => void };
type Dog = { bark: () => void };

// ❌ Mantiqiy xato — TS ishonadi
function isCat(animal: Cat | Dog): animal is Cat {
  return "bark" in animal; // "bark" bor = Dog, lekin Cat deb qaytarmoqda
}

// ✅ To'g'ri
function isCatFixed(animal: Cat | Dog): animal is Cat {
  return "meow" in animal;
}
```

**Nima uchun:** `is` keyword bilan siz TS'ga "menga ishon" deysiz. Compiler funksiya ichidagi mantiqni tekshirmaydi — noto'g'ri guard runtime xato'lariga olib keladi.

---

## Amaliy Mashqlar

### Mashq 1: Type Alias va Union (Oson)

**Savol:** Quyidagi kodning natijasini ayting:

```typescript
type Result = "success" | "error" | "pending";

function getIcon(result: Result): string {
  if (result === "success") return "✅";
  if (result === "error") return "❌";
  return "⏳";
}

console.log(getIcon("success"));
console.log(getIcon("pending"));
// console.log(getIcon("warning")); // Bu qator nimaga olib keladi?
```

<details>
<summary>Javob</summary>

```
✅
⏳
// "warning" qatori: TS compile-time xato beradi:
// Argument of type '"warning"' is not assignable to parameter of type 'Result'
```

**Tushuntirish:** Literal union type faqat belgilangan qiymatlarni qabul qiladi. `"warning"` `Result`'da yo'q — TS compile-time'da to'xtaydi. Bu type safety — noto'g'ri qiymat runtime'ga yetmaydi.

</details>

---

### Mashq 2: Intersection Composition (O'rta)

**Savol:** `createTimestampedUser` funksiyasini yozing — `User` va `Timestamps`'ni birlashtirgan object qaytarsin.

```typescript
type User = { name: string; email: string };
type Timestamps = { createdAt: Date; updatedAt: Date };

function createTimestampedUser(name: string, email: string): User & Timestamps {
  // implement
}
```

<details>
<summary>Javob</summary>

```typescript
function createTimestampedUser(name: string, email: string): User & Timestamps {
  const now = new Date();
  return {
    name,
    email,
    createdAt: now,
    updatedAt: now,
  };
}
```

**Tushuntirish:** Intersection `User & Timestamps` barcha property'larni talab qiladi. Return object'da `name`, `email` (User'dan) va `createdAt`, `updatedAt` (Timestamps'dan) — to'rttasi ham bo'lishi shart.

</details>

---

### Mashq 3: Discriminated Union + Exhaustive Check (Qiyin)

**Savol:** `Notification` type uchun `renderNotification` funksiyasini yozing. Har tur uchun tegishli xabar qaytarsin va **exhaustive check** bo'lsin.

```typescript
type Notification =
  | { type: "email"; to: string; subject: string; body: string }
  | { type: "sms"; phone: string; message: string }
  | { type: "push"; deviceId: string; title: string; body: string };

function renderNotification(n: Notification): string {
  // implement — exhaustive check bilan
}
```

<details>
<summary>Javob</summary>

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected: ${JSON.stringify(value)}`);
}

function renderNotification(n: Notification): string {
  switch (n.type) {
    case "email":
      return `📧 To: ${n.to} | Subject: ${n.subject}`;
    case "sms":
      return `📱 To: ${n.phone} | ${n.message}`;
    case "push":
      return `🔔 Device: ${n.deviceId} | ${n.title}`;
    default:
      return assertNever(n);
  }
}
```

**Tushuntirish:** `assertNever` compile-time va runtime safety beradi. Agar yangi notification type qo'shilsa (`"webhook"`), TS compile xato beradi.

</details>

---

### Mashq 4: Custom Type Guard (Qiyin)

**Savol:** `isAdmin` type guard funksiyasini yozing.

```typescript
type Admin = { role: "admin"; name: string; permissions: string[] };
type RegularUser = { role: "user"; name: string };
type User = Admin | RegularUser;

function isAdmin(user: User): user is Admin {
  // implement
}

function showDashboard(user: User): void {
  if (isAdmin(user)) {
    console.log(`Admin: ${user.name}, Permissions: ${user.permissions.join(", ")}`);
  } else {
    console.log(`User: ${user.name}`);
  }
}
```

<details>
<summary>Javob</summary>

```typescript
function isAdmin(user: User): user is Admin {
  return user.role === "admin";
}
```

**Tushuntirish:** Discriminated union bo'lgani uchun oddiy `if` ham ishlaydi — lekin type guard funksiya qayta ishlatilishi mumkin (DRY principle). Type guard mantiqini bitta joyda saqlash kodni toza qiladi.

</details>

---

### Mashq 5: Intersection Conflict (Qiyin)

**Savol:** Quyidagi kodni tushuntiring — qaysi qator xato beradi va nima uchun?

```typescript
type A = { x: number; y: string };
type B = { x: number; y: number; z: boolean };

type C = A & B;

const c1: C = { x: 1, y: "hello", z: true };
const c2: C = { x: 1, y: 42, z: true };
const c3: C = { x: 1, z: true };
```

<details>
<summary>Javob</summary>

```typescript
const c1: C = { x: 1, y: "hello", z: true };
// ❌ y: string & number = never, "hello" never'ga mos kelmaydi

const c2: C = { x: 1, y: 42, z: true };
// ❌ y: string & number = never, 42 ham never'ga mos kelmaydi

const c3: C = { x: 1, z: true };
// ❌ y property majburiy (bor, lekin never type), va obyektda yo'q

// C = { x: number; y: never; z: boolean }
// y property: string & number = never
// Hech qanday qiymat assign qilib bo'lmaydi
// C type amalda ishlatib bo'lmaydi
```

**Tushuntirish:** `A.y: string` va `B.y: number` intersection'da `never` bo'ladi. `never`'ga hech qanday qiymat assign qilib bo'lmaydi — natija: C tip'i mavjud, lekin ishlatib bo'lmaydi.

</details>

---

## Xulosa

Bu bo'limda type system'ning eng asosiy qurilish bloklarini o'rgandik:

- **Type Aliases** — murakkab tiplarga nom berish. Object shape uchun `interface`, boshqa holatlarda `type alias` (union, tuple, mapped, conditional, primitive alias, function type)
- **Union Types (`|`)** — "bu yoki u" mantiq. Qiymat bir nechta tipdan birortasi. Narrowing bilan ishlash — faqat umumiy member'lar accessible
- **Narrowing** — `typeof`, `instanceof`, `in`, custom type guard orqali union'ni aniq tipga toraytirish
- **`typeof`** — primitive tiplar uchun. `null` "object" gotcha'si — qo'shimcha tekshiruv kerak
- **`instanceof`** — class instance'lar uchun. Interface'lar bilan ishlamaydi
- **`in` operator** — property mavjudligi. TS 4.9+ da yanada kuchli (non-discriminated union'larda ham)
- **Intersection Types (`&`)** — "ham bu, ham u". Type composition, mixin pattern, branded types
- **Intersection Conflict** — bir xil nomli farqli tiplar — `never` (type theory: empty set). Object'lar rekursiv merge, primitive'lar conflict
- **Discriminated Unions** — umumiy discriminant literal property bilan. Redux, state machines, API responses — hammasi shu pattern ustida
- **Custom Type Guards** — `is` keyword bilan. Compile-time trust — TS mantiqni tekshirmaydi
- **Exhaustive Checking** — `never` bilan barcha case'larni kafolatlash. Yangi variant qo'shganda compile xato
- **`assertNever`** — reusable exhaustive check helper. Ikki qavat safety: compile + runtime
- **Union vs Overloads** — return type parameter'ga bog'liq bo'lsa — overloads, aks holda union
- **`satisfies`** — type check + literal type saqlash. Enum replacement, config validation uchun kuchli
- **Edge Cases** — closure narrowing, `typeof null`, intersection primitive+object, discriminant yo'qligi, non-discriminated union

Discriminated union'lar — TypeScript'ning eng kuchli pattern'i. `assertNever` bilan birga, yangi variant qo'shganda compiler sizni himoyalaydi. Production kodda har bir union type uchun exhaustive check qo'yish — best practice.

---

**Keyingi bo'lim:** [06-type-narrowing.md](06-type-narrowing.md) — Type Narrowing chuqur: Control Flow Analysis to'liq, truthiness/equality narrowing, assertion functions (`asserts x is T`), `satisfies` operator chuqur, closure'larda narrowing muammolari, narrowing limitations va best practices.
