# Bo'lim 15: Built-in Utility Types

> Built-in utility types — TypeScript ning type tizimi uchun "standard library" si. Bular tayyor type transformation lar bo'lib, tez-tez uchraydigan type operatsiyalarini qisqacha yozish imkonini beradi. Barchasi oldingi bo'limlarda o'rganilgan tushunchalar asosida qurilgan: [mapped types](13-mapped-types.md) (`Partial`, `Required`, `Readonly`, `Record`, `Pick`, `Omit`), [conditional types](12-conditional-types.md) (`Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `Awaited`), va [template literal types](14-template-literal-types.md) (`Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — [Bo'lim 14](14-template-literal-types.md#string-manipulation-types) da yoritilgan). Har bir utility type uchun: **nima qiladi**, **ichki implementatsiya**, **real-world use case**, va **gotchas**.

---

## Mundarija

- [Object Transformation Types](#object-transformation-types)
  - [`Partial<T>`](#partialt)
  - [`Required<T>`](#requiredt)
  - [`Readonly<T>`](#readonlyt)
  - [`Record<K, V>`](#recordk-v)
  - [`Pick<T, K>`](#pickt-k)
  - [`Omit<T, K>`](#omitt-k)
- [Union Transformation Types](#union-transformation-types)
  - [`Exclude<T, U>`](#excludet-u)
  - [`Extract<T, U>`](#extractt-u)
  - [`NonNullable<T>`](#nonnullablet)
- [Function Types](#function-types)
  - [`ReturnType<T>`](#returntypet)
  - [`Parameters<T>`](#parameterst)
  - [`ConstructorParameters<T>`](#constructorparameterst)
  - [`InstanceType<T>`](#instancetypet)
- [This Types](#this-types)
  - [`ThisParameterType<T>`](#thisparametertypet)
  - [`OmitThisParameter<T>`](#omitthisparametert)
- [Async Types](#async-types)
  - [`Awaited<T>`](#awaitedt)
- [`NoInfer<T>` (TS 5.4+)](#noinfert-ts-54)
- [Utility Types Kombinatsiyasi](#utility-types-kombinatsiyasi)
- [Performance va Best Practices](#performance-va-best-practices)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Object Transformation Types

### Nazariya

Bu guruh — [mapped types](13-mapped-types.md) asosida qurilgan utility type lar. Object type ning property larini transform qiladi: optional/required, readonly/mutable, key tanlash/olib tashlash. Barchasi [homomorphic mapped type](13-mapped-types.md) — original type ning modifier larini saqlab qoladi (`Record` bundan mustasno — u non-homomorphic).

---

### `Partial<T>`

`Partial<T>` — object type `T` ning **barcha property larini optional** (`?`) qiladi. Ya'ni har bir property berilmasligi mumkin.

**Qachon ishlatiladi:** Eng ko'p `update` funksiyalarda — faqat o'zgargan field larni berish uchun.

**Implementation:**

```typescript
// lib.es5.d.ts da:
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

Bu homomorphic mapped type — original type ning `readonly` modifier larini saqlab qoladi.

**Gotcha:** Shallow — faqat top-level property lar optional. Nested object lar to'liq qoladi. Deep partial uchun custom type kerak ([Bo'lim 16](16-custom-utility-types.md)).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

// Update funksiyasi — faqat o'zgargan field lar
function updateUser(id: number, updates: Partial<User>): User {
  const current = getUserById(id);
  return { ...current, ...updates };
}

updateUser(1, { name: "Ali" });         // ✅ faqat name
updateUser(1, { age: 30, email: "a" }); // ✅ ikki field
updateUser(1, {});                      // ✅ hech narsa

// ❗ Partial barcha field larni optional qiladi, shu jumladan kerak bo'lganlarni
type PartialUser = Partial<User>;
const user: PartialUser = {}; // ✅ — id ham yo'q! Xavfli bo'lishi mumkin

// Yechim — ba'zi field larni required qoldirib, qolganini optional qilish:
type UpdateUser = Partial<User> & Pick<User, "id">;
// id majburiy, qolganlar optional

const update: UpdateUser = { id: 1, name: "Ali" }; // ✅
// const bad: UpdateUser = { name: "Ali" };          // ❌ — id kerak

// Shallow xususiyati:
interface Config {
  db: { host: string; port: number };
}

type PartialConfig = Partial<Config>;
// { db?: { host: string; port: number } }
// db optional — lekin agar berilsa, host VA port IKKALASI kerak!
```

</details>

---

### `Required<T>`

`Required<T>` — `Partial` ning teskari amali. **Barcha optional property larni required** qiladi. `-?` modifier bilan optional marker ni olib tashlaydi.

**Qachon ishlatiladi:** Default qiymatlar qo'yilgandan keyin, config object ning to'liq ekanini kafolatlash uchun.

**Implementation:**

```typescript
type Required<T> = {
  [P in keyof T]-?: T[P];
};
```

`-?` — optional marker ni olib tashlaydi. `readonly` modifier saqlanadi (homomorphic).

**Gotcha:** `-?` faqat optional MARKER (`?`) ni olib tashlaydi — value type dagi `| undefined` ni OLIB TASHLAMAYDI.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface Options {
  debug?: boolean;
  timeout?: number;
  retries?: number;
  baseUrl?: string;
}

// Default lar bilan merge qilgandan keyin — hammasi required
function createClient(opts: Options): Required<Options> {
  return {
    debug: false,
    timeout: 5000,
    retries: 3,
    baseUrl: "https://api.example.com",
    ...opts,
  };
}

const client = createClient({ timeout: 10000 });
client.debug;    // boolean — optional emas, aniq mavjud
client.retries;  // number — aniq mavjud

// ❗ -? va | undefined farqi
interface Form {
  name?: string;              // optional (? marker)
  phone: string | undefined;  // required lekin undefined bo'lishi mumkin
}

type RequiredForm = Required<Form>;
// {
//   name: string;                // ✅ ? olib tashlandi, undefined ham ketdi
//   phone: string | undefined;   // ❗ qoldi — | undefined union da edi, ? yo'q edi
// }
```

</details>

---

### `Readonly<T>`

`Readonly<T>` — barcha property larga `readonly` modifier qo'shadi. Property qiymatlari faqat o'qilishi mumkin, o'zgartirib bo'lmaydi.

**Qachon ishlatiladi:** Immutable data pattern larda, state management da (Redux store), config object larda.

**Implementation:**

```typescript
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

**Gotcha:** Shallow — faqat top-level property lar readonly. Nested object va array lar mutable qoladi. Deep readonly uchun custom type kerak ([Bo'lim 16](16-custom-utility-types.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

`Readonly<T>` va `as const` farqi muhim:

| Xususiyat | `Readonly<T>` | `as const` |
|-----------|--------------|------------|
| Depth | Shallow | Deep (recursive) |
| Literal types | Saqlanmaydi | Saqlanadi |
| Array | Property `readonly`, lekin element lar mutable | `readonly ["a", "b"]` tuple |
| Runtime | ❌ | ❌ |

`Readonly` compile-time construct — runtime da hech qanday himoya yo'q. `Object.freeze()` bilan birgalikda ishlatish runtime himoya beradi, lekin `Object.freeze` ham shallow.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface State {
  count: number;
  items: string[];
}

function reducer(state: Readonly<State>, action: Action): State {
  // state.count = 5;        // ❌ — readonly
  // state.items.push("x");  // ⚠️ — COMPILE bo'ladi! (shallow readonly)

  return {
    ...state,
    count: state.count + 1,
    items: [...state.items, "new"],
  };
}

// Shallow xususiyati:
interface Config {
  db: { host: string; port: number };
  tags: string[];
}

const config: Readonly<Config> = {
  db: { host: "localhost", port: 5432 },
  tags: ["prod"],
};

// config.db = { host: "x", port: 1 };  // ❌ — readonly
config.db.host = "newhost";              // ⚠️ ✅ — nested mutable!
config.tags.push("staging");              // ⚠️ ✅ — array mutable!
```

</details>

---

### `Record<K, V>`

`Record<K, V>` — berilgan key type (`K`) va value type (`V`) dan object type yaratadi. Bu mapped type, lekin **non-homomorphic** — ya'ni original type ning modifier larini saqlamaydi.

**Qachon ishlatiladi:** Dictionary/map yaratish, enum dan object yaratish, key-value mapping.

**Implementation:**

```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
// keyof any = string | number | symbol
```

`keyof T` dan emas, mustaqil `K` dan key oladi — shuning uchun non-homomorphic.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Dictionary pattern ===
type UserMap = Record<string, User>;
const users: UserMap = {
  "user-1": { id: 1, name: "Ali", email: "a@b.com", age: 25 },
  "user-2": { id: 2, name: "Vali", email: "v@b.com", age: 30 },
};

// === Enum dan config ===
type Theme = "light" | "dark" | "system";
type ThemeConfig = Record<Theme, { bg: string; fg: string }>;

const themes: ThemeConfig = {
  light:  { bg: "#fff", fg: "#000" },
  dark:   { bg: "#000", fg: "#fff" },
  system: { bg: "auto", fg: "auto" },
};
// ❗ Har bir Theme uchun config MAJBURIY — birini tushirib qoldirsa xato

// === Status mapping ===
type StatusCode = 200 | 404 | 500;
type StatusMessage = Record<StatusCode, string>;

const messages: StatusMessage = {
  200: "OK",
  404: "Not Found",
  500: "Internal Server Error",
};

// === Record vs Index Signature ===

// Index signature — hech qanday key majburiy emas
interface Dict { [key: string]: number; }

// Record<string, number> — semantik jihatdan bir xil
type Dict2 = Record<string, number>;

// Lekin aniq union key bilan:
type Specific = Record<"a" | "b" | "c", number>;
// { a: number; b: number; c: number }
// ❗ Barcha key lar MAJBURIY
```

</details>

---

### `Pick<T, K>`

`Pick<T, K>` — object type `T` dan faqat tanlangan property larni (`K`) oladi. Yangi type faqat shu key larga ega.

**Implementation:**

```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

`K extends keyof T` — faqat `T` da mavjud key larni qabul qiladi (strict).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// API response dan password ni olib tashlash
type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: number; name: string; email: string }

// Form uchun kerakli field lar
type LoginForm = Pick<User, "email" | "password">;
// { email: string; password: string }

// Component props
type UserCardProps = Pick<User, "name" | "email"> & {
  onClick: () => void;
};

// === Pick vs Omit — Qachon Qaysi Biri ===

// 10 ta field dan 2 tasini OLMOQCHI — Pick qulayroq
type Short = Pick<User, "id" | "name">;

// 10 ta field dan 2 tasini OLIB TASHLAMOQCHI — Omit qulayroq
type WithoutSensitive = Omit<User, "password" | "createdAt">;
```

</details>

---

### `Omit<T, K>`

`Omit<T, K>` — `Pick` ning teskari amali. Object type dan tanlangan property larni **olib tashlaydi**. Qolgan barcha property lar saqlanadi.

**Implementation:**

```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

Ichida `Pick` va `Exclude` ishlatiladi: avval `keyof T` dan `K` ni chiqaradi, keyin qolgan key lar bilan `Pick` qiladi.

**Muhim farq:** `Pick` da `K extends keyof T` (faqat mavjud key lar), `Omit` da `K extends keyof any` (mavjud bo'lmagan key ham qabul qilinadi — typo tutmaydi!).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Sensitive field larni olib tashlash
type SafeUser = Omit<User, "password">;
// { id: number; name: string; email: string; createdAt: Date }

// Create uchun — id va createdAt server da yaratiladi
type CreateUserDto = Omit<User, "id" | "createdAt">;
// { name: string; email: string; password: string }

// Extending — mavjud type ni kengaytirish
type Admin = Omit<User, "createdAt"> & {
  role: "admin" | "superadmin";
  permissions: string[];
};

// ❗ Omit mavjud bo'lmagan key ni ham qabul qiladi — typo xavfi
type Wrong = Omit<User, "pasword">;  // ✅ — xato bermaydi, lekin typo!

// Yechim — strict Omit yaratish:
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
// type Wrong2 = StrictOmit<User, "pasword">; // ❌ — xato beradi ✅
```

</details>

---

## Union Transformation Types

### Nazariya

Bu guruh — [conditional types](12-conditional-types.md) asosida qurilgan. Union type dan member larni filtrlash va chiqarish uchun ishlatiladi. Barchasi **distributive conditional type** — union ning har bir member i alohida tekshiriladi.

---

### `Exclude<T, U>`

`Exclude<T, U>` — union type `T` dan `U` ga assignable bo'lgan member larni **chiqaradi** (olib tashlaydi). Qolgan member lar qoladi.

**Implementation:**

```typescript
type Exclude<T, U> = T extends U ? never : T;
```

Distributive conditional type — union har bir member i alohida tekshiriladi. Agar member `U` ga extend qilsa → `never` (chiqariladi), aks holda → qoladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Distribution jarayoni:

```
Exclude<"a" | "b" | "c", "a" | "b">

1. "a" extends "a" | "b" ? never : "a" → never
2. "b" extends "a" | "b" ? never : "b" → never
3. "c" extends "a" | "b" ? never : "c" → "c"

Natija: never | never | "c" → "c"
```

Discriminated union bilan ham ishlaydi — structural matching orqali:

```
Exclude<Circle | Square | Triangle, { kind: "circle" }>

1. Circle extends { kind: "circle" } → ✅ → never
2. Square extends { kind: "circle" } → ❌ → Square
3. Triangle extends { kind: "circle" } → ❌ → Triangle

Natija: Square | Triangle
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Asosiy ishlatish ===
type T1 = Exclude<"a" | "b" | "c", "a">;
// "b" | "c"

type T2 = Exclude<"a" | "b" | "c", "a" | "b">;
// "c"

type T3 = Exclude<string | number | boolean, string>;
// number | boolean

// === Real-world: mavjud event lardan ba'zilarini chiqarish ===
type AllEvents = "click" | "scroll" | "keypress" | "mousemove" | "resize";
type MouseEvents = "click" | "mousemove";

type NonMouseEvents = Exclude<AllEvents, MouseEvents>;
// "scroll" | "keypress" | "resize"

// === Discriminated union da ===
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

type NonCircle = Exclude<Shape, { kind: "circle" }>;
// { kind: "square" } | { kind: "triangle" }
```

</details>

---

### `Extract<T, U>`

`Extract<T, U>` — `Exclude` ning teskari amali. Union type `T` dan **faqat `U` ga assignable bo'lgan** member larni ajratib oladi.

**Implementation:**

```typescript
type Extract<T, U> = T extends U ? T : never;
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type T1 = Extract<"a" | "b" | "c", "a" | "c">;
// "a" | "c"

type T2 = Extract<string | number | boolean, string | boolean>;
// string | boolean

// === Discriminated union da ma'lum shape larni olish ===
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

type RoundShapes = Extract<Shape, { kind: "circle" }>;
// { kind: "circle"; radius: number }

// === Function type larni ajratish ===
type Mixed = string | number | (() => void) | ((x: number) => string);

type Functions = Extract<Mixed, (...args: any[]) => any>;
// (() => void) | ((x: number) => string)
```

</details>

---

### `NonNullable<T>`

`NonNullable<T>` — type dan `null` va `undefined` ni chiqaradi. Aslida `Exclude` ning maxsus holati.

**Implementation:**

```typescript
// TS 5.0+ da ichki optimizatsiya:
type NonNullable<T> = T & {};

// Semantik ekvivalent:
// type NonNullable<T> = Exclude<T, null | undefined>;
```

**Qanday ishlaydi:** `T & {}` — `null` va `undefined` ni chiqaradi chunki ular `{}` ga assignable emas. `null & {}` → `never`, `undefined & {}` → `never`, `string & {}` → `string`.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type T1 = NonNullable<string | null | undefined>;
// string

type T2 = NonNullable<number | null>;
// number

// === Real-world: optional chaining natijasini kafolatlash ===
interface Config {
  db?: { host: string; port: number };
}

function getHost(config: Config): string {
  const db = config.db;
  if (!db) throw new Error("DB config required");

  // Bu yerda db: NonNullable<Config["db"]>
  // Ya'ni: { host: string; port: number }
  return db.host;
}

// === Array.filter bilan ===
const values = [1, null, 2, undefined, 3] as (number | null | undefined)[];

const clean: number[] = values.filter(
  (v): v is NonNullable<typeof v> => v != null
);
// [1, 2, 3] — type-safe filter
```

</details>

---

## Function Types

### Nazariya

Bu guruh — [conditional types va `infer` keyword](12-conditional-types.md) asosida qurilgan. Function type dan turli qismlarni extract qiladi: return type, parametrlar, constructor parametrlari, instance type.

**Muhim:** Bu utility type lar **`typeof`** bilan ishlatiladi — ular TYPE qabul qiladi, VALUE emas. Funksiya VALUE dan type olish uchun `typeof fn` kerak.

---

### `ReturnType<T>`

`ReturnType<T>` — funksiya type dan **return type** ni extract qiladi.

**Implementation:**

```typescript
type ReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : any;
```

<details>
<summary><strong>Under the Hood</strong></summary>

`ReturnType` bilan ishlashda bilish kerak bo'lgan holatlar:

1. **`typeof` kerak** — `ReturnType<typeof fn>` (fn value, typeof fn type)
2. **Generic funksiya** — `ReturnType<typeof identity>` → `unknown` (generic T resolve bo'lmaydi)
3. **Async funksiya** — `ReturnType<typeof asyncFn>` → `Promise<T>` (unwrap QILMAYDI). Unwrap uchun: `Awaited<ReturnType<typeof asyncFn>>`
4. **Overloaded funksiya** — faqat **oxirgi** overload ning return type i olinadi. Bu kutilmagan natija berishi mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
function createUser(name: string, age: number) {
  return { id: Math.random(), name, age, createdAt: new Date() };
}

type NewUser = ReturnType<typeof createUser>;
// { id: number; name: string; age: number; createdAt: Date }

// ❗ typeof kerak
// ReturnType<createUser>        // ❌ — value, type emas
// ReturnType<typeof createUser> // ✅

// === Generic funksiya ===
function identity<T>(arg: T): T { return arg; }
type IdReturn = ReturnType<typeof identity>;
// unknown — generic T resolve bo'lmaydi

// === Async funksiya ===
async function fetchUser(): Promise<User> { return {} as User; }
type FetchReturn = ReturnType<typeof fetchUser>;
// Promise<User> — unwrap QILMAYDI

type SyncReturn = Awaited<ReturnType<typeof fetchUser>>;
// User — Awaited bilan unwrap

// === Overloaded funksiya ===
declare function parse(input: string): string;
declare function parse(input: number): number;
type ParseReturn = ReturnType<typeof parse>;
// number — OXIRGI overload ning return type i
```

</details>

---

### `Parameters<T>`

`Parameters<T>` — funksiya type dan **parametrlar tuple** ini extract qiladi.

**Implementation:**

```typescript
type Parameters<T extends (...args: any) => any> =
  T extends (...args: infer P) => any ? P : never;
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
function greet(name: string, age: number, active: boolean): string {
  return `${name}, ${age}`;
}

type GreetParams = Parameters<typeof greet>;
// [name: string, age: number, active: boolean]

// Har bir parametrni olish
type FirstParam = GreetParams[0];   // string
type SecondParam = GreetParams[1];  // number
type ThirdParam = GreetParams[2];   // boolean

// === Wrapper funksiya yaratish ===
function withLogging<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  return (...args) => {
    console.log("Calling with:", args);
    return fn(...args);
  };
}

const loggedGreet = withLogging(greet);
loggedGreet("Ali", 25, true);   // ✅ — type-safe
// loggedGreet("Ali");            // ❌ — argumentlar yetarli emas

// === Event handler ===
type ClickHandler = (event: MouseEvent, target: HTMLElement) => void;
type ClickParams = Parameters<ClickHandler>;
// [event: MouseEvent, target: HTMLElement]
```

</details>

---

### `ConstructorParameters<T>`

`ConstructorParameters<T>` — class constructor ning parametrlari tuple ini extract qiladi. `Parameters` ning constructor versiyasi.

**Implementation:**

```typescript
type ConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;
```

`abstract new (...)` — abstract class constructorlarni ham qabul qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
class UserService {
  constructor(
    private db: Database,
    private cache: CacheService,
    private logger: Logger
  ) {}
}

type UserServiceParams = ConstructorParameters<typeof UserService>;
// [db: Database, cache: CacheService, logger: Logger]

// === Factory funksiya ===
function createInstance<T extends new (...args: any[]) => any>(
  ctor: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new ctor(...args);
}

const service = createInstance(UserService, db, cache, logger);
// service: UserService — type-safe

// === Built-in class lar ===
// Overloaded constructor lar uchun faqat OXIRGI overload signature olinadi
type DateParams = ConstructorParameters<typeof Date>;
// [value: string | number | Date]

type ErrorParams = ConstructorParameters<typeof Error>;
// [message?: string, options?: ErrorOptions]

type RegExpParams = ConstructorParameters<typeof RegExp>;
// [pattern: string | RegExp, flags?: string]
```

</details>

---

### `InstanceType<T>`

`InstanceType<T>` — constructor type dan **instance type** ini extract qiladi. Ya'ni `new` bilan yaratilgan object ning type i.

**Implementation:**

```typescript
type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
class User {
  constructor(public name: string, public age: number) {}
  greet() { return `Hi, ${this.name}`; }
}

type UserInstance = InstanceType<typeof User>;
// User — class ning instance type i

// ❗ typeof kerak — InstanceType constructor TYPE qabul qiladi
// InstanceType<User>         — ❌ User bu instance type, constructor emas
// InstanceType<typeof User>  — ✅ typeof User bu constructor type

// === Generic factory pattern ===
function create<T extends new (...args: any[]) => any>(
  ctor: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new ctor(...args);
}

const user = create(User, "Ali", 25);
// user: User — TS biladi bu User instance

// === Abstract class bilan ===
abstract class Shape {
  abstract area(): number;
}

class Circle extends Shape {
  constructor(public radius: number) { super(); }
  area() { return Math.PI * this.radius ** 2; }
}

type ShapeInstance = InstanceType<typeof Shape>;
// Shape — abstract class ham ishlaydi
```

</details>

---

## This Types

### Nazariya

`this` parameter bilan ishlovchi utility type lar. TypeScript da funksiya birinchi parametr sifatida `this` ni declare qilishi mumkin — bu runtime da mavjud emas, faqat type checking uchun.

---

### `ThisParameterType<T>`

`ThisParameterType<T>` — funksiya type dan `this` parameter ning type ini extract qiladi. Agar `this` parameter bo'lmasa — `unknown` qaytaradi.

**Implementation:**

```typescript
type ThisParameterType<T> =
  T extends (this: infer U, ...args: never) => any ? U : unknown;
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
function toHex(this: Number) {
  return this.toString(16);
}

type ThisType = ThisParameterType<typeof toHex>;
// Number

// this parametr bo'lmasa:
function add(a: number, b: number): number { return a + b; }
type NoThis = ThisParameterType<typeof add>;
// unknown

// === DOM event handler ===
function handleClick(this: HTMLButtonElement, event: MouseEvent) {
  this.disabled = true;
}

type HandlerThis = ThisParameterType<typeof handleClick>;
// HTMLButtonElement
```

</details>

---

### `OmitThisParameter<T>`

`OmitThisParameter<T>` — funksiya type dan `this` parameter ni **olib tashlaydi**. `this` siz yangi funksiya type qaytaradi.

**Implementation:**

```typescript
type OmitThisParameter<T> =
  unknown extends ThisParameterType<T>
    ? T
    : T extends (...args: infer A) => infer R
      ? (...args: A) => R
      : T;
```

Agar `this` parameter bo'lmasa — o'zgarishsiz qaytaradi. Aks holda — `this` siz yangi funksiya type yaratadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
function toHex(this: Number) {
  return this.toString(16);
}

type ToHexFn = typeof toHex;
// (this: Number) => string

type BoundToHex = OmitThisParameter<typeof toHex>;
// () => string — this olib tashlandi

// Real-world: .bind() natijasi
const boundHex: BoundToHex = toHex.bind(42 as Number);
boundHex(); // "2a"

// === Callback sifatida o'tkazish ===
function withThis(this: HTMLElement, event: Event) {
  this.classList.add("active");
}

type Callback = OmitThisParameter<typeof withThis>;
// (event: Event) => void
```

</details>

---

## Async Types

### Nazariya

Promise va async/await bilan ishlovchi utility type.

---

### `Awaited<T>`

`Awaited<T>` — `Promise` (yoki nested Promise lar) ni **recursive unwrap** qiladi. `Promise<T>` dan `T` ni oladi. Nested `Promise<Promise<T>>` bo'lsa — to'liq unwrap qiladi. TS 4.5 da qo'shilgan.

**Implementation:**

```typescript
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends ((value: infer V, ...args: infer _) => any) ?
      Awaited<V> :
      never :
  T;
```

Bu implementation thenable object larni ham qo'llab-quvvatlaydi — faqat native `Promise` emas, `then` methodi bor har qanday object.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type A = Awaited<Promise<string>>;
// string

type B = Awaited<Promise<Promise<number>>>;
// number — nested Promise to'liq unwrap

type C = Awaited<boolean | Promise<string>>;
// boolean | string — union da ham ishlaydi

type D = Awaited<string>;
// string — Promise emas, o'zgarishsiz

// === Promise.all bilan ===
async function fetchData() {
  const [users, posts] = await Promise.all([
    fetchUsers(),   // Promise<User[]>
    fetchPosts(),   // Promise<Post[]>
  ]);
  // users: User[], posts: Post[] — Awaited avtomatik unwrap
}

// === ReturnType bilan kombinatsiya ===
async function getUser(): Promise<User> {
  return {} as User;
}

type AsyncReturn = ReturnType<typeof getUser>;
// Promise<User>

type SyncReturn = Awaited<ReturnType<typeof getUser>>;
// User — Promise unwrap qilindi
```

</details>

---

## `NoInfer<T>` (TS 5.4+)

### Nazariya

`NoInfer<T>` — TypeScript ning **type inference** ni ma'lum pozitsiyada **to'xtatadi**. Ya'ni kompilator shu pozitsiyadagi qiymatdan type ni infer qilmasligi kerakligini bildiradi.

**Muammo:** Generic funksiyalarda ba'zan TypeScript noto'g'ri joydan type ni infer qiladi. Masalan, default qiymat dan infer qilish kerak emas — faqat asosiy argument dan.

**Implementation:**

```typescript
// lib.es5.d.ts da:
type NoInfer<T> = intrinsic;
```

Bu `Uppercase`/`Lowercase` kabi **intrinsic** type — kompilator ichida maxsus qayta ishlanadi. Runtime da hech qanday ta'siri yo'q (type erasure).

<details>
<summary><strong>Under the Hood</strong></summary>

`NoInfer` kompilator ning inference algoritmida qanday ishlaydi:

```
function createStore<T>(initial: T, defaultValue: NoInfer<T>): T

Call: createStore(42, "fallback")

NoInfer siz:
1. T ni initial dan infer: T = number (42 dan)
2. T ni defaultValue dan infer: T = string ("fallback" dan)
3. Birlashtiriladi: T = number | string
→ Natija: string | number — KUTILMAGAN

NoInfer bilan:
1. T ni initial dan infer: T = number (42 dan)
2. defaultValue: NoInfer<T> — inference BLOKLANADI
3. T = number (faqat initial dan)
4. defaultValue: number — "fallback" mos kelmaydi
→ ❌ Error — TO'G'RI XATO
```

Qachon ishlatish kerak:
- **Default/fallback qiymat** — asosiy argument dan infer bo'lsin
- **Callback parametri** — funksiya signature dan infer bo'lsin
- **Validation** — asosiy type dan infer, validator mos kelsin

Qachon kerak EMAS:
- Barcha argument lar teng darajada muhim va infer bo'lishi kerak bo'lsa

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Muammo — NoInfer siz ===

function createStoreLoose<T>(initial: T, defaultValue: T): T {
  return initial ?? defaultValue;
}

const looseStore = createStoreLoose(42, "fallback");
// T = string | number — ikkala argument dan infer qildi!

// === Yechim — NoInfer bilan ===

function createStoreStrict<T>(initial: T, defaultValue: NoInfer<T>): T {
  return initial ?? defaultValue;
}

const strictStoreError = createStoreStrict(42, "fallback");
// ❌ Error: '"fallback"' is not assignable to 'number'

const strictStoreOk = createStoreStrict(42, 0);
// ✅ T = number, defaultValue = 0

// === Event emitter pattern ===

function on<T extends string>(
  event: T,
  callback: (data: NoInfer<T>) => void
): void {
  // implementation
}

on("click", (data) => {
  // data: "click" — aniq literal type
});

// === Configuration pattern ===

interface Theme {
  primary: string;
  secondary: string;
  accent: string;
}

function createTheme<T extends Theme>(
  theme: T,
  fallback: NoInfer<T>
): T {
  return { ...fallback, ...theme };
}
```

</details>

---

## Utility Types Kombinatsiyasi

### Nazariya

Utility type larning haqiqiy kuchi — ularni **kombinatsiya** qilib ishlatishda. Bir nechta utility type ni birga qo'llab murakkab type transformatsiyalar yaratish mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user" | "moderator";
  createdAt: Date;
  updatedAt: Date;
}

// === 1. Create DTO — id, timestamps server yaratadi ===
type CreateUserDto = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; password: string; role: ... }

// === 2. Update DTO — id kerak, qolganlar optional ===
type UpdateUserDto = Pick<User, "id"> & Partial<Omit<User, "id">>;
// { id: number } & { name?: string; email?: string; ... }

// === 3. Public User — sensitive field larsiz, readonly ===
type PublicUser = Readonly<Omit<User, "password">>;
// { readonly id: number; readonly name: string; ... }

// === 4. Faqat string field larni olish ===
type StringFields<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
type UserStrings = StringFields<User>;
// { name: string; email: string; password: string }

// === 5. Required faqat ma'lum field lar uchun ===
type RequireFields<T, K extends keyof T> =
  Omit<T, K> & Required<Pick<T, K>>;

interface Options {
  debug?: boolean;
  timeout?: number;
  retries?: number;
}

type WithTimeout = RequireFields<Options, "timeout">;
// { debug?: boolean; retries?: number; timeout: number }

// === 6. Partial faqat ma'lum field lar uchun ===
type PartialFields<T, K extends keyof T> =
  Omit<T, K> & Partial<Pick<T, K>>;

type FlexibleUser = PartialFields<User, "email" | "password">;
// email va password optional, qolganlar required

// === 7. Async versiya — barcha method lar Promise qaytaradi ===
type Async<T> = {
  [K in keyof T]: T[K] extends (...args: infer A) => infer R
    ? (...args: A) => Promise<Awaited<R>>
    : T[K];
};

interface UserRepo {
  findById(id: number): User | null;
  findAll(): User[];
  save(user: User): void;
}

type AsyncUserRepo = Async<UserRepo>;
// {
//   findById(id: number): Promise<User | null>;
//   findAll(): Promise<User[]>;
//   save(user: User): Promise<void>;
// }
```

</details>

---

## Performance va Best Practices

### Nazariya

Utility type lar — kompilator type check jarayonida hisoblash talab qiladi. Chuqur nesting va murakkab kombinatsiyalar performance ga ta'sir qilishi mumkin.

**Qoidalar:**

1. **Intermediate type alias** ishlatish — chuqur nesting o'rniga bosqichma-bosqich
2. **Generic constraints** ichida murakkab utility qo'ymaslik — type alias bilan ajratish
3. **Recursive utility** da depth limit qo'shish

**Utility Type Tanlash Jadvali:**

| Kerak narsa | Utility Type |
|-------------|-------------|
| Barcha optional | `Partial<T>` |
| Barcha required | `Required<T>` |
| Barcha readonly | `Readonly<T>` |
| Key-value map | `Record<K, V>` |
| Ba'zi property lar | `Pick<T, K>` |
| Ba'zi property larsiz | `Omit<T, K>` |
| Union dan chiqarish | `Exclude<T, U>` |
| Union dan ajratish | `Extract<T, U>` |
| null/undefined chiqarish | `NonNullable<T>` |
| Funksiya return type | `ReturnType<T>` |
| Funksiya parametrlari | `Parameters<T>` |
| Constructor parametrlari | `ConstructorParameters<T>` |
| Class instance type | `InstanceType<T>` |
| Promise unwrap | `Awaited<T>` |
| Inference bloklash | `NoInfer<T>` |
| this parameter type | `ThisParameterType<T>` |
| this olib tashlash | `OmitThisParameter<T>` |

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ❌ Yomon — chuqur nesting
type Bad = Partial<Required<Readonly<Omit<Pick<User, "id" | "name">, "id">>>>;

// ✅ Yaxshi — intermediate type alias lar
type PickedUser = Pick<User, "id" | "name">;
type ReadonlyPicked = Readonly<PickedUser>;
type Good = Partial<ReadonlyPicked>;

// ❌ Yomon — generic constraints ichida murakkab utility
function bad<T extends Partial<Required<Readonly<SomeType>>>>() {}

// ✅ Yaxshi — type alias bilan
type ProcessedType = Partial<Required<Readonly<SomeType>>>;
function good<T extends ProcessedType>() {}

// ❌ Yomon — recursive utility types chuqur nesting
type DeepPartial<T> = { [K in keyof T]?: DeepPartial<T[K]> };
// 10+ level deep object da sekin bo'lishi mumkin

// ✅ Yaxshi — depth limit qo'shish
type BoundedDeepPartial<T, D extends unknown[] = []> =
  D["length"] extends 5 ? T :
  T extends object ? { [K in keyof T]?: BoundedDeepPartial<T[K], [...D, unknown]> } : T;
```

</details>

---

## Edge Cases va Gotchas

### 1. `Exclude<never, T>` → `never` — bo'sh union dan chiqarish

```typescript
type T1 = Exclude<never, string>;
// never — never bo'sh union, hech qanday member yo'q
// Distributive conditional type — bo'sh union da hech narsa iterate bo'lmaydi

type T2 = Extract<never, string>;
// never — xuddi shunday sabab
```

### 2. `Partial<T>` + `exactOptionalPropertyTypes` — `undefined` assignability o'zgaradi

```typescript
// tsconfig: "exactOptionalPropertyTypes": true

interface User {
  name: string;
  age: number;
}

type PartialUser = Partial<User>;
// { name?: string; age?: number }

// exactOptionalPropertyTypes YOQILGAN bo'lsa:
const user: PartialUser = { name: undefined };
// ❌ Error! ? — "property yo'q bo'lishi mumkin" degani,
// "undefined bo'lishi mumkin" EMAS

// exactOptionalPropertyTypes O'CHIRILGAN bo'lsa (default):
const user2: PartialUser = { name: undefined };
// ✅ — default da ? va | undefined bir xil
```

### 3. `Parameters` va `ReturnType` overloaded funksiya bilan — faqat oxirgi overload

```typescript
declare function parse(input: string): string;
declare function parse(input: number): number;
declare function parse(input: boolean): boolean;

type Params = Parameters<typeof parse>;
// [input: boolean] — OXIRGI overload!

type Return = ReturnType<typeof parse>;
// boolean — OXIRGI overload!

// ❗ Birinchi yoki ikkinchi overload kerak bo'lsa —
// alohida type extraction yoki conditional type kerak
```

### 4. `keyof Record<string, T>` → `string` (emas `string | number`)

```typescript
type Keys1 = keyof Record<string, unknown>;
// string — faqat string

type Keys2 = keyof { [key: string]: unknown };
// string | number — index signature!

// Nima uchun farq? Record<string, T> mapped type:
// { [P in string]: T } — bu index signature bilan bir xil EMAS
// Record faqat string key lar, index signature esa
// number key larni ham qabul qiladi (JS da obj[0] === obj["0"])
```

### 5. `Readonly<T>` va array — `push`/`pop` hali ham ishlaydi

```typescript
interface State {
  items: string[];
}

const state: Readonly<State> = { items: ["a", "b"] };

// state.items = [];        // ❌ — readonly (reassign mumkin emas)
state.items.push("c");      // ⚠️ ✅ — COMPILE BO'LADI!
state.items[0] = "changed"; // ⚠️ ✅ — COMPILE BO'LADI!

// Readonly faqat STATE.ITEMS reassign ni bloklaydi,
// lekin items ICHIDAGI mutatsiyani YO'Q.
// Yechim: ReadonlyArray<string> yoki readonly string[]
interface SafeState {
  readonly items: readonly string[];
}
```

---

## Common Mistakes

### ❌ Xato 1: `Partial` va `Readonly` ning shallow ekanini unutish

```typescript
interface Config {
  db: { host: string; port: number };
  tags: string[];
}

const config: Readonly<Config> = {
  db: { host: "localhost", port: 5432 },
  tags: ["prod"],
};

config.db.host = "remote"; // ⚠️ ✅ — nested mutable!
config.tags.push("dev");    // ⚠️ ✅ — array mutable!
```

**Nima uchun:** `Readonly<T>` faqat top-level property larga `readonly` qo'shadi. Deep readonly uchun custom type kerak ([Bo'lim 16](16-custom-utility-types.md)).

### ❌ Xato 2: `Omit` da typo ni tutmaslik

```typescript
interface User { id: number; name: string; password: string; }

type SafeUser = Omit<User, "pasword">;  // Typo! password QOLDI
// ⚠️ Xato BERMAYDI

// ✅ Strict Omit:
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
// StrictOmit<User, "pasword"> // ❌ Error — typo tutildi
```

**Nima uchun:** `Omit` da `K extends keyof any` — har qanday string qabul qilinadi.

### ❌ Xato 3: `ReturnType` ga value berish (`typeof` unutish)

```typescript
function getUser() { return { id: 1, name: "Ali" }; }

// type T = ReturnType<getUser>;         // ❌ — value, type emas
type T = ReturnType<typeof getUser>;     // ✅
```

**Nima uchun:** `ReturnType<T>` type parameter qabul qiladi. `getUser` — runtime value. `typeof getUser` — shu value ning type i.

### ❌ Xato 4: `Required` `| undefined` ni olib tashlamaydi

```typescript
interface Form {
  name?: string;              // optional (? marker)
  phone: string | undefined;  // required lekin undefined bo'lishi mumkin
}

type RequiredForm = Required<Form>;
// { name: string; phone: string | undefined }
// ❗ phone dagi | undefined QOLDI — -? faqat ? ni olib tashlaydi
```

### ❌ Xato 5: `Record` ning non-homomorphic ekanini bilmaslik

```typescript
interface Config {
  readonly host: string;
  readonly port: number;
  timeout?: number;
}

// ❌ Record bilan — modifier lar yo'qoladi
type RecordConfig = Record<keyof Config, string>;
// { host: string; port: string; timeout: string }
// readonly YO'QOLDI, optional YO'QOLDI!

// ✅ Mapped type bilan — modifier lar saqlanadi
type MappedConfig = { [K in keyof Config]: string };
// { readonly host: string; readonly port: string; timeout?: string }
```

**Nima uchun:** `Record` — [non-homomorphic](13-mapped-types.md) mapped type. Key larni `keyof T` dan emas, mustaqil union dan oladi.

---

## Amaliy Mashqlar

### Mashq 1: `PublicUser` type yarating (Oson)

**Savol:** Quyidagi `User` type dan `password` va `secretKey` ni olib tashlab, qolganlarini `readonly` qiling.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  secretKey: string;
  role: "admin" | "user";
}

type PublicUser = /* sizning yechimingiz */;
```

<details>
<summary>Javob</summary>

```typescript
type PublicUser = Readonly<Omit<User, "password" | "secretKey">>;
// {
//   readonly id: number;
//   readonly name: string;
//   readonly email: string;
//   readonly role: "admin" | "user";
// }
```

**Tushuntirish:** Avval `Omit` bilan sensitive field larni olib tashlash, keyin `Readonly` bilan barcha qolgan field larni immutable qilish.

</details>

---

### Mashq 2: `RequireFields` type yarating (O'rta)

**Savol:** Faqat berilgan field larni required qilib, qolganlarini o'zgarishsiz qoldiradigan type yozing.

```typescript
type RequireFields<T, K extends keyof T> = /* ? */;

// Test:
interface Options {
  debug?: boolean;
  timeout?: number;
  retries?: number;
  baseUrl?: string;
}

type WithTimeout = RequireFields<Options, "timeout" | "baseUrl">;
// { debug?: boolean; retries?: number; timeout: number; baseUrl: string }
```

<details>
<summary>Javob</summary>

```typescript
type RequireFields<T, K extends keyof T> =
  Omit<T, K> & Required<Pick<T, K>>;
```

**Tushuntirish:**
1. `Omit<T, K>` — K field larni olib tashlash (ular optional qoladi)
2. `Required<Pick<T, K>>` — K field larni olish va required qilish
3. `&` (intersection) — ikkalasini birlashtirish

</details>

---

### Mashq 3: `AsyncMethods<T>` type yarating (O'rta)

**Savol:** Object type ning barcha method larini `Promise` qaytaradigan qilib, oddiy property larni o'zgarishsiz qoldiradigan type yozing.

```typescript
type AsyncMethods<T> = /* ? */;

// Test:
interface UserService {
  name: string;
  findById(id: number): User | null;
  findAll(): User[];
  create(data: CreateUserDto): User;
  count: number;
}

type AsyncUserService = AsyncMethods<UserService>;
// {
//   name: string;
//   findById(id: number): Promise<User | null>;
//   findAll(): Promise<User[]>;
//   create(data: CreateUserDto): Promise<User>;
//   count: number;
// }
```

<details>
<summary>Javob</summary>

```typescript
type AsyncMethods<T> = {
  [K in keyof T]: T[K] extends (...args: infer A) => infer R
    ? (...args: A) => Promise<Awaited<R>>
    : T[K];
};
```

**Tushuntirish:**
1. Mapped type — har bir property ni iterate
2. Conditional type — agar property funksiya bo'lsa: `infer A` (argument lar), `infer R` (return type), `Promise<Awaited<R>>` (Promise ga wrap, double wrap dan himoya)
3. Funksiya bo'lmasa — o'zgarishsiz (`T[K]`)

</details>

---

### Mashq 4: `StrictOmit` va `PickByType` yozing (Qiyin)

**Savol:**

1. `StrictOmit<T, K>` — `Omit` ning strict versiyasi (faqat mavjud key larni qabul qiladi)
2. `PickByType<T, ValueType>` — object type dan faqat ma'lum value type ga ega property larni oladi

```typescript
type StrictOmit<T, K extends keyof T> = /* ? */;
type PickByType<T, ValueType> = /* ? */;

// Test:
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
}

// StrictOmit<User, "typo">; // ❌ — xato berishi kerak
type WithoutId = StrictOmit<User, "id">;
// { name: string; email: string; age: number; active: boolean }

type StringProps = PickByType<User, string>;
// { name: string; email: string }

type NumberProps = PickByType<User, number>;
// { id: number; age: number }
```

<details>
<summary>Javob</summary>

```typescript
type StrictOmit<T, K extends keyof T> = Omit<T, K>;

type PickByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? K : never]: T[K];
};
```

**Tushuntirish:**
- `StrictOmit` — `K extends keyof T` constraint qo'shildi. Mavjud bo'lmagan key berilsa compile-time xato.
- `PickByType` — key remapping (`as`) bilan filtrlash: agar `T[K]` berilgan `ValueType` ga extend qilsa — key saqlanadi, aks holda `never`.

</details>

---

### Mashq 5: `MergeTypes<T, U>` yozing (Qiyin)

**Savol:** Ikki object type ni birlashtiruvchi type yozing. Agar ikkala type da bir xil key bo'lsa — `U` (ikkinchi) ning type i ustun bo'lsin.

```typescript
type MergeTypes<T, U> = /* ? */;

// Test:
interface Base {
  id: number;
  name: string;
  role: "user";
}

interface Override {
  name: string | null;
  role: "admin" | "user";
  permissions: string[];
}

type Merged = MergeTypes<Base, Override>;
// {
//   id: number;
//   name: string | null;     // U dan
//   role: "admin" | "user";  // U dan
//   permissions: string[];   // U dan
// }
```

<details>
<summary>Javob</summary>

```typescript
type MergeTypes<T, U> = Omit<T, keyof U> & U;
```

**Tushuntirish:**
1. `Omit<T, keyof U>` — T dan U da mavjud key larni olib tashlash (conflict larni yo'q qilish)
2. `& U` — U ni to'liq qo'shish
3. Natija: T ning unique field lari + U ning barcha field lari (conflict da U ustun)

Bu JavaScript ning `{ ...base, ...override }` operatsiyasining type-level versiyasi.

</details>

---

## Xulosa

Bu bo'limda TypeScript ning barcha built-in utility type lari chuqur o'rganildi:

**Object transformation (mapped types asosida):**
- **`Partial<T>`** — barcha optional. Gotcha: shallow, nested to'liq qoladi.
- **`Required<T>`** — barcha required (`-?`). Gotcha: `| undefined` union ni olib tashlamaydi.
- **`Readonly<T>`** — barcha readonly. Gotcha: shallow, nested mutation mumkin.
- **`Record<K, V>`** — key-value map. Gotcha: non-homomorphic — modifier lar saqlanmaydi.
- **`Pick<T, K>`** — tanlangan key lar. `K extends keyof T` — strict.
- **`Omit<T, K>`** — key larni olib tashlash. `K extends keyof any` — typo tutmaydi.

**Union transformation (conditional types asosida):**
- **`Exclude<T, U>`** — union dan chiqarish. Distributive conditional type.
- **`Extract<T, U>`** — union dan ajratish. `Exclude` ning teskari amali.
- **`NonNullable<T>`** — `null | undefined` chiqarish. TS 5.0+ da `T & {}` bilan.

**Function types (`infer` asosida):**
- **`ReturnType<T>`** — return type. `typeof` kerak. Overload da oxirgi signature.
- **`Parameters<T>`** — parametrlar tuple. Wrapper funksiya lar uchun.
- **`ConstructorParameters<T>`** — constructor parametrlari. Factory pattern uchun.
- **`InstanceType<T>`** — instance type. `typeof` kerak.

**Boshqa:**
- **`Awaited<T>`** (TS 4.5+) — recursive Promise unwrap. Thenable ham qo'llab-quvvatlanadi.
- **`NoInfer<T>`** (TS 5.4+) — inference bloklash. Default/fallback qiymatlarda.
- **`ThisParameterType<T>`** / **`OmitThisParameter<T>`** — `this` parameter bilan ishlash.

**Bog'liq bo'limlar:**
- [Bo'lim 12: Conditional Types](12-conditional-types.md) — `infer`, distributive behavior — utility type lar asosi
- [Bo'lim 13: Mapped Types](13-mapped-types.md) — homomorphic/non-homomorphic, modifier lar
- [Bo'lim 14: Template Literal Types](14-template-literal-types.md) — `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`
- [Bo'lim 16: Custom Utility Types](16-custom-utility-types.md) — `DeepPartial`, `DeepReadonly`, `Brand`, `PathKeys`

Asosiy insight: barcha utility type lar — oldingi bo'limlardagi tushunchalar (mapped types, conditional types, `infer`, `keyof`) ning tayyor kombinatsiyalari. Ularning ichki implementatsiyasini tushunish — o'z custom utility type laringizni yaratish uchun asos.

---

**Keyingi bo'lim:** [16-custom-utility-types.md](16-custom-utility-types.md) — O'z custom utility type laringizni yaratish: `DeepPartial`, `DeepReadonly`, `Mutable`, `Brand`/`Opaque` (nominal types), `PathKeys`/`PathValue` (dotted path), type-level programming patterns, va recursive type optimization.
