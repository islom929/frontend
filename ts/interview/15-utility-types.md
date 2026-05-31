# Interview: Built-in Utility Types

> Partial, Required, Readonly, Record, Pick, Omit, Exclude, Extract, NonNullable, ReturnType, Parameters, ConstructorParameters, InstanceType, ThisParameterType, OmitThisParameter, Awaited, NoInfer — practical usage, gotchas, mexanizm va kombinatsiyalar bo'yicha interview savollari. Har javob mustaqil — kontekst javob ichida.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar) — 9 ta
- [Output savollari](#output-savollari) — 4 ta
- [Coding challenges](#coding-challenges) — 4 ta
- [Bug fix](#bug-fix) — 2 ta
- [Xulosa](#xulosa)

---

## Nazariy savollar

### Savol 1: `Partial<T>` va `Required<T>` — real-world use case. [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Partial<T>` — barcha property'larni optional qiladi. `Required<T>` — barcha optional'ni required qiladi. Asosiy use case — update DTO, form state, configuration merge.

### To'liq tushuntirish

`Partial<T>` mapped type ichida `+?` modifier qo'shadi, har property optional bo'ladi. `Required<T>` esa `-?` modifier bilan optional'ni olib tashlaydi (TS 2.8+).

Implementation:

```typescript
type Partial<T>  = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
```

Real-world use cases:
1. **Update DTO** — `updateUser(id, partial: Partial<User>)` — faqat o'zgartiriladigan field'lar
2. **Config merge** — default + override pattern
3. **Form state** — boshlang'ich bo'sh, asta-sekin to'ldiriladi
4. **Builder pattern** — har step'da yangi field qo'shiladi

### Kod misol

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;
}

// 1. Update DTO
function updateUser(id: number, updates: Partial<User>): User {
  const current = getUserById(id);
  return { ...current, ...updates };
}

updateUser(1, { name: "Ali" });           // ✅
updateUser(1, { email: "a@b.com", age: 25 }); // ✅
// updateUser(1, { invalid: true });      // ❌

// 2. Config merge
interface ApiConfig {
  baseUrl: string;
  timeout: number;
  retries: number;
}

const defaults: ApiConfig = {
  baseUrl: "https://api.example.com",
  timeout: 30000,
  retries: 3,
};

function createClient(overrides: Partial<ApiConfig>): ApiConfig {
  return { ...defaults, ...overrides };
}

// 3. Required — formdan submit qilishda
type FormDraft = Partial<User>;
type SubmittedForm = Required<Pick<User, "name" | "email">>;

function submitForm(draft: FormDraft): SubmittedForm | null {
  if (!draft.name || !draft.email) return null;
  return { name: draft.name, email: draft.email };
}
```

### Edge Cases

- `Partial<T>` va `Readonly<T>` **shallow** — faqat top-level property'lar. Nested object'larga ta'sir qilmaydi
- `Required<T>` `-?` faqat optional marker'ni olib tashlaydi va value type'dan `undefined`'ni chiqaradi. Lekin `phone: string | undefined` (marker'siz) — `-?` `undefined`'ni olib tashlamaydi
- Deep variant kerak bo'lsa — `DeepPartial<T>` recursive yozish, base case (function, array) bilan
- `Partial` har property'ga `undefined` qo'shadi — `exactOptionalPropertyTypes: true` flag bilan strict bo'ladi

### Follow-up savollar

1. **"`Partial<T>` deep variant qanday yoziladi?"** — Recursive: `T extends (...args) => any ? T : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T`.
2. **"`Partial<T>` bilan strict null check?"** — `exactOptionalPropertyTypes: true` flag — `name?: string` ga `undefined` explicit assign qilolmaslik.

</details>

### Savol 2: `Readonly<T>` — shallow ekanini qanday tushunish kerak? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Readonly<T>` faqat top-level property'larni `readonly` qiladi. Nested object'lar va array element'lari mutable qoladi. Deep immutability uchun `DeepReadonly<T>` recursive type kerak.

### To'liq tushuntirish

`Readonly<T> = { readonly [K in keyof T]: T[K] }` — `readonly` modifier qo'shadi, lekin value type'ni o'zgartirmaydi. Agar `T[K]` object yoki array bo'lsa — uning ichidagi property'lar mutable qoladi.

Bu **shallow vs deep** immutability farqi:
- **Shallow** — faqat tashqi qatlam immutable
- **Deep** — barcha nesting darajalar immutable

Compile-time check faqat — runtime'da `Object.freeze()` bilan birga ishlatish kerak agar true immutability talab qilinsa.

### Kod misol

```typescript
interface Config {
  port: number;
  server: {
    host: string;
    options: { ssl: boolean };
  };
  tags: string[];
}

type ReadonlyConfig = Readonly<Config>;
const config: ReadonlyConfig = {
  port: 3000,
  server: { host: "localhost", options: { ssl: true } },
  tags: ["dev", "test"],
};

// config.port = 4000;             // ❌ readonly
// config.server = {...};          // ❌ readonly (top-level)

config.server.host = "remote";     // ✅ — nested mutable!
config.server.options.ssl = false; // ✅ — deep nested mutable!
config.tags.push("prod");          // ✅ — array element mutable!

// Deep readonly — recursive
type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :
  T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

type DeepConfig = DeepReadonly<Config>;
const deepConfig: DeepConfig = {
  port: 3000,
  server: { host: "localhost", options: { ssl: true } },
  tags: ["dev"],
};

// deepConfig.server.host = "x";  // ❌ — readonly
// deepConfig.tags.push("y");     // ❌ — ReadonlyArray
```

### Edge Cases

- `ReadonlyArray<T>` — `push`, `pop`, `splice` ishlamaydi (compile error)
- Class instance `Readonly<T>` — instance property'lari readonly, lekin method'lar orqali state o'zgartirish mumkin
- `Object.freeze` runtime check beradi, `Readonly<T>` compile-time check beradi — ikkalasini birga ishlatish to'liq immutability uchun
- Mutable type'ga assign qilish — `Readonly<T>` → `T` allowed (type widening), ta'kid yo'q

### Follow-up savollar

1. **"`Readonly<T>` runtime'da ta'sir qiladi mi?"** — Yo'q, faqat compile-time. Runtime immutability uchun `Object.freeze()` yoki Immer/Immutable.js kerak.
2. **"`as const` va `Readonly<T>` farqi?"** — `as const` literal type generate qiladi (`{ x: 1 } as const` → `{ readonly x: 1 }`), `Readonly<T>` esa type-level utility.

</details>

### Savol 3: `Omit<T, K>` da qanday gotcha bor? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Omit<T, K>` typo'ni **tutmaydi** — `K extends keyof any` (loose constraint). Mavjud bo'lmagan key'ni o'tkazsa ham xato bermaydi. `StrictOmit<T, K extends keyof T>` xavfsizroq.

### To'liq tushuntirish

`Omit` implementation:

```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
// keyof any = string | number | symbol
```

`K extends keyof any` — har qanday `PropertyKey`. Bu generic code uchun moslashuvchanlik, lekin xavfli — typo compile error bermaydi.

Pick esa strict: `Pick<T, K extends keyof T>` — typo darhol tutiladi.

Nima uchun `Omit` loose: TS 3.5 da kiritilganda yangi key qo'shilishi mumkin bo'lgan generic scenarios uchun moslashuv kerak edi. Lekin production code'da xato manbai.

### Kod misol

```typescript
interface User {
  id: number;
  name: string;
  password: string;
}

// Pick — strict
// type Bad1 = Pick<User, "pasword">;
// ❌ Type '"pasword"' is not assignable to type 'keyof User'

// Omit — loose, typo tutmaydi
type Bad2 = Omit<User, "pasword">;
// ⚠️ Hech qanday xato!
// password property qoladi (typo tufayli olib tashlanmadi)

console.log({} as Bad2);
// { id: number; name: string; password: string }
// password hali ham bor — typo tufayli filter ishlamadi

// Yechim — StrictOmit
type StrictOmit<T, K extends keyof T> = Omit<T, K>;

// type Bad3 = StrictOmit<User, "pasword">;
// ❌ Type '"pasword"' is not assignable to type 'keyof User' ✅

type Good = StrictOmit<User, "password">;
// { id: number; name: string } ✅

// Real-world API DTO
type PublicUser = StrictOmit<User, "password">;
// Sensitive field guaranteed filterlanadi
```

### Edge Cases

- `Omit` **non-distributive** — union'ga qo'llanganda butun union'ga ishlaydi, har member'ga alohida emas. Misol: `Omit<{a: 1} | {a: 1, b: 2}, "b">` → `{a: 1}` (faqat umumiy property'lar qoladi)
- Discriminated union'da `Omit` — discriminant property'ni olib tashlasa narrowing buziladi. Distributive variant: `T extends any ? Omit<T, K> : never`
- `Omit<T, never>` → `T` (o'zgarishsiz), `Omit<T, keyof T>` → `{}`
- Branded type (`T & { __brand: "X" }`) bilan `Omit` — brand property'ni saqlash uchun ehtiyot bo'lish kerak (`__brand` keyof T'da bor)

### Follow-up savollar

1. **"Nima uchun `Omit` loose qilingan?"** — Generic moslashuvchanlik (Pick'da typo error ko'pincha noqulay edi). Lekin production'da `StrictOmit` afzal.
2. **"`Omit` discriminated union bilan qanday ishlaydi?"** — Distribution yo'q — butun union'ga Omit qo'llaniladi, discriminant property olib tashlansa narrowing buziladi. Yechim: `T extends any ? Omit<T, K> : never`.

</details>

### Savol 4: `Exclude<T, U>` va `Extract<T, U>` — qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Exclude<T, U>` — T'dan U bilan assignable bo'lgan member'larni olib tashlaydi. `Extract<T, U>` — T'dan U bilan assignable bo'lganlarni saqlaydi. Ikkala ham distributive conditional type — union har member'iga alohida qo'llaniladi.

### To'liq tushuntirish

Implementation:

```typescript
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
```

`T extends U` distributive conditional — union har member alohida tekshiriladi. Match bo'lsa Exclude'da `never`, Extract'da T qaytadi. `never` union'da yo'qoladi (`A | never = A`).

Use cases:
- **Exclude** — string literal union'dan ma'lum value'larni olib tashlash
- **Extract** — discriminated union'dan ma'lum variant'larni ajratish
- **NonNullable** — `Exclude<T, null | undefined>` ekvivalenti

### Kod misol

```typescript
// Exclude — olib tashlash
type Direction = "left" | "right" | "top" | "bottom";
type Horizontal = Exclude<Direction, "top" | "bottom">;
// "left" | "right"

// Extract — saqlash
type Vertical = Extract<Direction, "top" | "bottom">;
// "top" | "bottom"

// Discriminated union extraction
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "scroll"; delta: number }
  | { type: "input"; value: string };

type ClickEvent = Extract<Event, { type: "click" }>;
// { type: "click"; x: number; y: number }

type NonScrollEvent = Exclude<Event, { type: "scroll" }>;
// { type: "click"; ... } | { type: "input"; ... }

// Function-like'larni ajratish
// Eslatma: global `Function` type'i juda umumiy (typescript-eslint `no-unsafe-function-type`
// rule taqiqlaydi). Aniqroq usul — call signature bilan filter:
type Callable = ((x: string) => void) | ((x: number) => void) | string;
type Functions = Extract<Callable, (...args: any[]) => any>;
// ((x: string) => void) | ((x: number) => void)

// NonNullable ekvivalenti
type WithNull = string | number | null | undefined;
type NoNull = Exclude<WithNull, null | undefined>;
// string | number

// NonNullable<T> = T & {} (TS 4.8+, oldindan Exclude<T, null | undefined>)
type AlsoNoNull = NonNullable<WithNull>;
// string | number
```

### Edge Cases

- `Exclude<never, T>` → `never` (bo'sh union'dan chiqarish)
- `Exclude<T, never>` → T (hech narsa olib tashlanmaydi)
- `Extract<T, never>` → `never` (hech qaysisi mos kelmaydi)
- `Extract<any, T>` → T (any har narsaga mos)
- Structural matching — `Extract<{a: 1, b: 2} | {a: 1}, {a: 1}>` → `{a: 1, b: 2} | {a: 1}` (har ikkalasi ham match)

### Follow-up savollar

1. **"`Exclude` discriminated union uchun nima uchun ideal?"** — Type narrowing — `type` discriminant property bilan exact variant'ni ajratish.
2. **"`Exclude<T, U>` va `T extends U ? never : T` farqi?"** — Identik. Exclude — sintaktik shaker. Lekin generic type ichida `T extends U` distribution maxsus generic parameter behavior'iga ega.

</details>

### Savol 5: `ReturnType<T>` va `Parameters<T>` — qanday ishlaydi, limitlari nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`ReturnType<T>` — function type'ning return type'ini infer qiladi. `Parameters<T>` — parametr tuple'ini infer qiladi. Limitatsiyalar: overload bo'lsa faqat oxirgi signature, generic function uchun infer noma'lum (default qiymat bilan).

### To'liq tushuntirish

Implementation:

```typescript
type ReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : any;

type Parameters<T extends (...args: any) => any> =
  T extends (...args: infer P) => any ? P : never;
```

Conditional type + `infer` bilan function signature qismlarini ekstrakt qilish.

Limitatsiyalar:
1. **Overload** — faqat **oxirgi signature** olinadi (compiler limitation)
2. **Generic function** — type parameter'lar default'ga (`unknown`) instantiate qilinadi
3. **`typeof` zarur** — runtime value'dan type olish uchun

### Kod misol

```typescript
function getUser(id: number) {
  return { id, name: "Ali", email: "ali@test.com" };
}

type R = ReturnType<typeof getUser>;
// { id: number; name: string; email: string }

type P = Parameters<typeof getUser>;
// [id: number]

// Async function
async function fetchUser(): Promise<{ id: number }> {
  return { id: 1 };
}

type FR = ReturnType<typeof fetchUser>;
// Promise<{ id: number }> — ❗ Promise qoldi
// Unwrap uchun: Awaited<ReturnType<typeof fetchUser>>

// Overload limitation
function add(a: string, b: string): string;
function add(a: number, b: number): number;
function add(a: any, b: any): any { return a + b; }

type AddReturn = ReturnType<typeof add>;
// number — faqat oxirgi overload signature

type AddParams = Parameters<typeof add>;
// [a: number, b: number] — faqat oxirgi overload

// Generic function limitation
function identity<T>(x: T): T { return x; }
type IR = ReturnType<typeof identity>;
// unknown — T noma'lum, default instantiation

// Class method
class UserService {
  getUser(id: number) { return { id }; }
}

type UserReturn = ReturnType<UserService["getUser"]>;
// { id: number }
```

### Edge Cases

- **Overload** — barcha signature'larni olish uchun manual extraction kerak (complex utility)
- **Generic** — TS 4.7+ instantiation expression bilan parametrize qilish mumkin: `ReturnType<typeof identity<string>>` → `string`. Bunday yozuvsiz (`ReturnType<typeof identity>`) type parameter `unknown`'ga instantiate bo'ladi
- **Constructor** — `ReturnType` ishlamaydi, `InstanceType<typeof Class>` kerak
- **Method'lar** — `T["methodName"]` bilan accessing, so'ng ReturnType

### Follow-up savollar

1. **"Barcha overload signature'larni olish mumkinmi?"** — Murakkab utility kerak — multiple `infer` clause yoki TS 4.9+ tools bilan. Practical'da ko'pchilik manual yozadi.
2. **"`Parameters<typeof Class>` ishlaydi mi?"** — Yo'q — Class constructor uchun `ConstructorParameters<typeof Class>` kerak.

</details>

### Savol 6: `ConstructorParameters<T>` va `InstanceType<T>` — qachon kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`ConstructorParameters<T>` — class constructor parametr tuple'ini olish. `InstanceType<T>` — class instance type'ini olish. Generic factory funksiyalar uchun majburiy.

### To'liq tushuntirish

Implementation:

```typescript
type ConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;

type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

`abstract new` — TS 4.2+ syntax, abstract class'larni ham qabul qiladi.

Use cases:
1. **Factory funksiya** — class va argument'larni qabul qilib instance qaytarish
2. **Dependency injection** — type-safe constructor injection
3. **Generic wrapper** — class'ni qayta yaratish (proxy, decorator)

### Kod misol

```typescript
class UserService {
  constructor(private db: Database, private logger: Logger) {}
  getUser(id: number) { return { id }; }
}

class PostService {
  constructor(private db: Database) {}
  getPost(id: number) { return { id }; }
}

// Generic factory
function createService<T extends new (...args: any) => any>(
  ServiceClass: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new ServiceClass(...args);
}

declare const db: Database;
declare const logger: Logger;

const userService = createService(UserService, db, logger);
// userService: UserService ✅

const postService = createService(PostService, db);
// postService: PostService ✅

// Wrong arguments
// createService(PostService, db, logger);
// ❌ Expected 2 arguments, but got 3

// createService(UserService, db);
// ❌ Expected 3 arguments, but got 2

// Real-world DI container
class Container {
  resolve<T extends new (...args: any) => any>(
    ClassRef: T,
    ...args: ConstructorParameters<T>
  ): InstanceType<T> {
    return new ClassRef(...args);
  }
}
```

### Edge Cases

- **Abstract class** — `abstract new` constraint TS 4.2+'dan beri qo'llab-quvvatlanadi. Eski TS'da abstract class'da `ConstructorParameters` ishlamaydi
- **Private constructor** — `private constructor` bo'lgan singleton'da ishlamaydi (constructor signature accessible emas)
- **Overloaded constructor** — overloaded signature'larda — faqat oxirgi
- **Generic class** — `class Repository<T>` — parametrlanmasa T `unknown`'ga instantiate bo'ladi; TS 4.7+ instantiation expression bilan `ConstructorParameters<typeof Repository<string>>` aniq tip beradi

### Follow-up savollar

1. **"Singleton class bilan qanday ishlash kerak?"** — Private constructor'ga `ConstructorParameters` ishlamaydi. Static factory method ishlatish: `ReturnType<typeof Singleton.getInstance>`.
2. **"DI container generic factory uchun real-world misol?"** — InversifyJS, tsyringe — shu pattern asoslangan.

</details>

### Savol 7: `Awaited<T>` nima va nima uchun `ReturnType` async uchun yetarli emas? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Awaited<T>` (TS 4.5+) — recursive Promise unwrap qiladi (nested Promise'larni ham). `ReturnType` async funksiya uchun `Promise<T>` qaytaradi — unwrap qilmaydi. To'g'ri usul: `Awaited<ReturnType<typeof asyncFn>>`.

### To'liq tushuntirish

`Awaited<T>` TS 4.5'da kiritilgan — `async`/`await` semantics'ni type-level'da modellashtirish uchun. Recursive conditional type bilan `Promise<Promise<T>>` ni ham unwrap qiladi — runtime'dagi `await` chain'ning type-level ekvivalenti.

Implementation:

```typescript
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends ((value: infer V, ...args: infer _) => any) ?
      Awaited<V> :
      never :
  T;
```

Thenable object'larni ham qo'llab-quvvatlaydi (`Promise` emas, lekin `then` method'i bor object'lar).

### Kod misol

```typescript
async function getUser(): Promise<{ id: number; name: string }> {
  return { id: 1, name: "Ali" };
}

// ❌ ReturnType — Promise qoldi
type R1 = ReturnType<typeof getUser>;
// Promise<{ id: number; name: string }>

// ✅ Awaited bilan unwrap
type R2 = Awaited<ReturnType<typeof getUser>>;
// { id: number; name: string }

// Recursive unwrap
type A = Awaited<Promise<string>>;             // string
type B = Awaited<Promise<Promise<number>>>;    // number — nested unwrap
type C = Awaited<Promise<Promise<Promise<boolean>>>>; // boolean
type D = Awaited<boolean | Promise<string>>;   // boolean | string — union'da ham
type E = Awaited<string>;                      // string — non-Promise unchanged
type F = Awaited<null>;                        // null

// Thenable
type G = Awaited<{ then(cb: (v: number) => void): void }>;
// number

// Real-world — async function return type
type AsyncResult<T extends (...args: any) => any> = Awaited<ReturnType<T>>;

async function fetchData(): Promise<{ users: User[]; posts: Post[] }> {
  return { users: [], posts: [] };
}

type Data = AsyncResult<typeof fetchData>;
// { users: User[]; posts: Post[] }
```

### Edge Cases

- **TS 4.5 dan oldin** — `Awaited` yo'q edi, manual recursive type yozish kerak edi
- **Non-Promise** — Awaited o'zgarishsiz qaytaradi
- **Custom thenable** — `then` method'i bor object'lar — Awaited unwrap qiladi
- **Promise.all** result — `Awaited<Promise<[number, string]>>` → `[number, string]` (tuple unwrap)

### Follow-up savollar

1. **"Nima uchun nested Promise unwrap?"** — JS runtime semantics: Promise boshqa thenable bilan resolve qilinsa, uning state'ini adopt qiladi (`Promise<Promise<T>>` runtime'da mavjud emas). `await Promise.resolve(Promise.resolve(42))` → `42`. `Awaited` shu flatten behavior'ni type-level'da modellashtiradi.
2. **"`Awaited` bilan `Promise.all` qanday ishlaydi?"** — `Promise.all([p1, p2])` return type `Promise<[T1, T2]>`. `Awaited` tuple unwrap qiladi: `[T1, T2]`.

</details>

### Savol 8: `ThisParameterType<T>` va `OmitThisParameter<T>` — qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`ThisParameterType<T>` — function'ning explicit `this` parametr type'ini ekstrakt qiladi. `OmitThisParameter<T>` — function signature'dan `this` parametr'ni olib tashlaydi. Method binding va `call`/`apply` type'lari uchun ishlatiladi.

### To'liq tushuntirish

TypeScript funksiyalarda `this` parametr'i deklare qilish mumkin — bu virtual parameter, runtime'da emas, faqat type-level binding uchun:

```typescript
function greet(this: User, message: string) {
  return `${this.name}: ${message}`;
}
```

`this` har doim birinchi parameter, lekin argument list'da hisoblanmaydi (`greet.call(user, "hi")` — 2 ta argument).

Implementation:

```typescript
type ThisParameterType<T> =
  T extends (this: infer U, ...args: never) => any ? U : unknown;

type OmitThisParameter<T> =
  unknown extends ThisParameterType<T>
    ? T
    : T extends (...args: infer A) => infer R ? (...args: A) => R : T;
```

Use cases:
- Method extraction (class method'ni standalone function sifatida ishlatish)
- `bind` natija type'ini hisoblash
- Mixin pattern'lar

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
}

function describe(this: User, prefix: string): string {
  return `${prefix}: ${this.name}, ${this.age}`;
}

type ThisType = ThisParameterType<typeof describe>;
// User — this parameter type

type WithoutThis = OmitThisParameter<typeof describe>;
// (prefix: string) => string — this parametr'siz signature

const user: User = { name: "Ali", age: 25 };

// call/apply bilan ishlash
describe.call(user, "Info");
// "Info: Ali, 25"

// bind — this fixed
const bound: WithoutThis = describe.bind(user);
bound("Info"); // ✅ this avtomatik user

// Real-world: extract class method
class Calculator {
  value = 0;
  add(this: Calculator, x: number) {
    this.value += x;
    return this;
  }
}

type AddFn = typeof Calculator.prototype.add;
type AddThis = ThisParameterType<AddFn>;       // Calculator
type AddStandalone = OmitThisParameter<AddFn>; // (x: number) => Calculator

// Function without this — ThisParameterType → unknown
function noThis(x: number) { return x * 2; }
type NoThis = ThisParameterType<typeof noThis>;  // unknown
type NoThisOmit = OmitThisParameter<typeof noThis>; // typeof noThis (no change)
```

### Edge Cases

- Function'da explicit `this` parametr yo'q → `ThisParameterType` `unknown` qaytaradi
- Arrow function — `this` bind qilolmaydi, `this` parametr deklaratsiyasi xato
- Class method'da implicit `this` (`this` parametr yozilmagan) function type'ga kirmaydi — `ThisParameterType` `unknown` qaytaradi, instance type'ni emas. Instance type'ni olish uchun method'da explicit `this: Calculator` yozish kerak
- `noImplicitThis: true` flag — `this` annotatsiyasiz function body'da `this` ishlatish xato beradi (lekin method'da implicit `this` baribir instance type sifatida ishlaydi)

### Follow-up savollar

1. **"`bind` natija type'i qanday hisoblanadi?"** — Lib.es5: `bind` overload — `bind(this: T, thisArg: ThisParameterType<T>)` natija `OmitThisParameter<T>`.
2. **"Arrow function'da `this` parametr nima uchun ishlamaydi?"** — Arrow function lexical `this` ishlatadi, dynamic binding yo'q. `this` parametr declaration semantically noto'g'ri.

<details>
<summary><strong>Deep Dive</strong></summary>

**`this` parametr — fake parameter mexanizmi**

TypeScript compiler `this` parametr'ni AST'da birinchi parameter sifatida ko'radi, lekin emit paytida (JS chiqarishda) olib tashlaydi. Bu pure type-level construct:

```typescript
function describe(this: User, prefix: string) {
  return `${prefix}: ${this.name}`;
}

// JS emit:
function describe(prefix) {
  return `${prefix}: ${this.name}`;
}
```

`describe.length` JS'da `1` (faqat `prefix`), TS type-level'da ham 1-argument. `this` faqat call site'da binding tekshirish uchun.

**Type-level binding mexanizmi**

```typescript
declare function describe(this: User, prefix: string): string;

const user: User = { name: "Ali", age: 25 };
const wrongThis = { title: "admin" };

describe.call(user, "Info");       // ✅ user assignable to User
describe.call(wrongThis, "Info");  // ❌ wrongThis not assignable to User

// Method call sintaksisi
user.describe("Info");             // ✅ this === user (User type)
```

`strictBindCallApply: true` flag yoqilganda `call`/`apply`/`bind` argumentlari `ThisParameterType<T>` bilan tekshiriladi.

**`lib.es5.d.ts` `Function.prototype.bind` overload**

```typescript
interface CallableFunction extends Function {
  bind<T>(this: T, thisArg: ThisParameterType<T>): OmitThisParameter<T>;

  bind<T, A0, A extends any[], R>(
    this: (this: T, arg0: A0, ...args: A) => R,
    thisArg: T,
    arg0: A0
  ): (...args: A) => R;

  // ... 4-argument overload chain ...
}
```

`bind` natija type'i `OmitThisParameter<T>` — `this` parametr olib tashlangan signature. Bu type-level "currying" — `this` allaqachon bog'langan.

**Arrow function `this` parametr xato**

```typescript
const arrow = (this: User, name: string) => name;
// ❌ An arrow function cannot have a 'this' parameter.
```

Arrow function ECMAScript spec'iga ko'ra lexical `this` ishlatadi — `[[ThisMode]]` slot'i "lexical" (function declaration'da "global" yoki "strict"). Dynamic binding (`call`/`apply` thisArg'i) ignored. TS shu spec semantics'ni type-level'da reflect qiladi.

**`noImplicitThis: true` flag effekti**

```typescript
// noImplicitThis: false (default strict'siz)
function legacy(name) {
  return this.greeting + name;  // this: any
}

// noImplicitThis: true
function modern(name: string) {
  return this.greeting + name;
  // ❌ 'this' implicitly has type 'any' because it does not have a type annotation.
}

// To'g'ri:
function modern(this: { greeting: string }, name: string) {
  return this.greeting + name;
}
```

`strict: true` `noImplicitThis`'ni avtomatik yoqadi.

</details>

</details>

### Savol 9: Built-in utility type'lar qaysi mexanizmlar asosida qurilgan? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

3 ta asosiy mexanizm: (1) **Mapped types** — object transformation (`Partial`, `Required`, `Pick`), (2) **Conditional types + `infer`** — type extraction (`ReturnType`, `Parameters`, `Awaited`), (3) **Intrinsic** — compiler-implemented (`Uppercase`, `NoInfer`).

### To'liq tushuntirish

Har built-in utility quyidagi 3 mexanizmdan biri bilan implement qilingan:

**1. Mapped Types** — object structure transformation:

```typescript
Partial<T>  = { [P in keyof T]?: T[P] };     // homomorphic
Required<T> = { [P in keyof T]-?: T[P] };
Readonly<T> = { readonly [P in keyof T]: T[P] };
Pick<T, K>  = { [P in K]: T[P] };            // K extends keyof T
Record<K, T> = { [P in K]: T };              // non-homomorphic
Omit<T, K>  = Pick<T, Exclude<keyof T, K>>;
```

**2. Conditional Types + `infer`** — type-level "destructuring":

```typescript
Exclude<T, U>            = T extends U ? never : T;
Extract<T, U>            = T extends U ? T : never;
NonNullable<T>           = T & {};
ReturnType<T>            = T extends (...args: any) => infer R ? R : any;
Parameters<T>            = T extends (...args: infer P) => any ? P : never;
ConstructorParameters<T> = T extends abstract new (...args: infer P) => any ? P : never;
InstanceType<T>          = T extends abstract new (...args: any) => infer R ? R : any;
ThisParameterType<T>     = T extends (this: infer U, ...args: never) => any ? U : unknown;
OmitThisParameter<T>     = /* conditional + infer combination */;
Awaited<T>               = /* recursive conditional + infer */;
```

**3. Intrinsic** — compiler ichida hardcode qilingan (`.d.ts` da `= intrinsic`, haqiqiy logika `checker.ts` ichida):

```typescript
Uppercase<S>   = intrinsic;
Lowercase<S>   = intrinsic;
Capitalize<S>  = intrinsic;
Uncapitalize<S> = intrinsic;
NoInfer<T>     = intrinsic;  // TS 5.4+
```

### Kod misol

```typescript
// 1. Mapped — homomorphic
interface User { id: number; name: string }

type P1 = Partial<User>;     // { id?: number; name?: string }
type R1 = Required<User>;    // { id: number; name: string }
type Pick1 = Pick<User, "id">; // { id: number }

// 2. Conditional + infer
function getUser(id: number) { return { id, name: "Ali" } }
type R2 = ReturnType<typeof getUser>;  // { id: number; name: string }
type P2 = Parameters<typeof getUser>;  // [id: number]

// 3. Intrinsic
type U = Uppercase<"hello">;   // "HELLO"
type C = Capitalize<"hello">;  // "Hello"
```

### Categorization Table

| Kategoriya | Mexanizm | Utility types |
|-----------|---------|---------------|
| Object transform | Mapped types | Partial, Required, Readonly, Record, Pick, Omit |
| Union filter | Distributive conditional | Exclude, Extract, NonNullable |
| Function extract | Conditional + infer | ReturnType, Parameters, ConstructorParameters, InstanceType, ThisParameterType, OmitThisParameter |
| Async unwrap | Recursive conditional | Awaited |
| String manipulation | Intrinsic | Uppercase, Lowercase, Capitalize, Uncapitalize |
| Inference control | Intrinsic | NoInfer (TS 5.4+) |

### Edge Cases

- `NonNullable<T>` — TS 4.8'gacha `Exclude<T, null | undefined>` edi, hozir `T & {}` (empty object intersection — `null`/`undefined` ni filter qiladi)
- `Awaited<T>` recursive — Promise unwrap chain'ni to'liq qiladi
- Intrinsic type'lar custom code'da implement qilolmaydi — compiler-only

### Follow-up savollar

1. **"Nima uchun string manipulation type'lari intrinsic?"** — Pure type-level recursion har character uchun yangi conditional type instantiation hosil qiladi va recursion depth limit'ga tez uriladi. Intrinsic versiyada string transformatsiya compiler'ning string handling kodida to'g'ridan-to'g'ri bajariladi, hech qanday type instantiation yo'q.
2. **"`NoInfer<T>` qanday ishlaydi?"** — Compiler inference candidate'larni yig'ayotganda shu pozitsiyani skip qiladi. Generic type inference manual cheklash uchun.

</details>

---

## Output savollari

### Savol 10: Utility composition output [Middle]

**Savol:** Har type'ning natijasini ayting:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
}

type A = Pick<User, "name" | "email">;
type B = Omit<User, "id" | "active">;
type C = Partial<Pick<User, "name" | "age">>;
type D = Required<C>;
type E = Readonly<Omit<User, "active">> & { active: boolean };
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = {
  name: string;
  email: string;
};

type B = {
  name: string;
  email: string;
  age: number;
};

type C = {
  name?: string;
  age?: number;
};

type D = {
  name: string;
  age: number;
};
// Required Partial'ni qaytaradi — faqat Pick qilingan field'lar

type E = {
  readonly id: number;
  readonly name: string;
  readonly email: string;
  readonly age: number;
  active: boolean;  // readonly EMAS — intersection'da alohida berildi
};
```

### To'liq tushuntirish

- A, B — standard Pick/Omit
- C — Pick'dan keyin Partial, faqat shu field'lar optional
- D — C'dan Required, optional yo'qotiladi
- E — Readonly + intersection: Omit qilingan field'lar readonly, alohida qo'shilgan field readonly emas

### Edge Cases

- E variant tricky — Readonly faqat birinchi argument'iga qo'llaniladi, intersection part alohida
- C → D zanjirida Pick keyin Partial keyin Required = original Pick natijasi

</details>

### Savol 11: `Exclude`/`Extract` output [Middle]

**Savol:** Har type natijasi nima?

```typescript
type T1 = Exclude<"a" | "b" | "c", "a">;
type T2 = Extract<"a" | "b" | "c", "a" | "d">;
type T3 = Exclude<string | number | null, null>;
type T4 = Extract<{ type: "click" } | { type: "scroll" }, { type: "click" }>;
type T5 = Exclude<(() => void) | string, Function>;
type T6 = NonNullable<string | null | undefined | 0>;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type T1 = "b" | "c";
// "a" olib tashlandi

type T2 = "a";
// "a" "a"|"d" da bor → saqlandi
// "b", "c" yo'q → never

type T3 = string | number;
// null olib tashlandi

type T4 = { type: "click" };
// Faqat click variant

type T5 = string;
// Function (callable) olib tashlandi

type T6 = string | 0;
// null va undefined olib tashlandi
// 0 saqlanadi (falsy bo'lsa ham)
```

### To'liq tushuntirish

- T1, T3 — oddiy union filter
- T2 — Extract T'ning U bilan assignable member'larini saqlaydi; literal union'larda bu ikkala union'da ham bor element'lar to'plamini beradi (`"b"`, `"c"` `"a"|"d"` da yo'q → never)
- T4 — discriminated union variant extraction
- T5 — Function tip'i call signature bor type'lar (function literal'larni qamrab oladi)
- T6 — NonNullable faqat null/undefined ni olib tashlaydi, 0 va "" saqlanadi (falsy emas, nullish faqat)

### Edge Cases

- `Exclude<T, never>` → T (hech narsa olib tashlanmaydi)
- `Extract<T, never>` → never (hech qaysisi mos kelmaydi)
- `Extract<any, T>` → T (any har narsa bilan match)

</details>

### Savol 12: Function utility output [Middle+]

**Savol:** Natijani ayting:

```typescript
function fetchUser(id: number, options?: { cache: boolean }): Promise<{ name: string }> {
  return Promise.resolve({ name: "Ali" });
}

class Repository<T> {
  constructor(public name: string, public items: T[]) {}
}

type T1 = ReturnType<typeof fetchUser>;
type T2 = Parameters<typeof fetchUser>;
type T3 = Awaited<ReturnType<typeof fetchUser>>;
type T4 = ConstructorParameters<typeof Repository>;
type T5 = InstanceType<typeof Repository>;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type T1 = Promise<{ name: string }>;

type T2 = [id: number, options?: { cache: boolean } | undefined];
// Tuple labeled parameter, optional ham

type T3 = { name: string };
// Awaited unwrap qildi

type T4 = [name: string, items: unknown[]];
// Generic T noma'lum → unknown default

type T5 = Repository<unknown>;
// Generic instance — unknown default
```

### To'liq tushuntirish

- T1 — async function `Promise<T>` qaytaradi
- T2 — labeled tuple parameter'lar, optional `| undefined` bilan
- T3 — Awaited bilan unwrap
- T4 va T5 — generic class type parameter T noma'lum, default `unknown`

### Edge Cases

- Generic class concrete type bilan: TS 4.7+ instantiation expression — `ConstructorParameters<typeof Repository<string>>` → `[name: string, items: string[]]`. Bunday yozuvsiz `T` `unknown`'ga instantiate bo'ladi
- Optional parametr labeled tuple'da `?` bilan
- Async function'ning ReturnType har doim Promise — Awaited bilan unwrap

</details>

### Savol 13: `Awaited` deep unwrap output [Middle+]

**Savol:** Har type natijasi nima?

```typescript
type T1 = Awaited<Promise<string>>;
type T2 = Awaited<Promise<Promise<number>>>;
type T3 = Awaited<Promise<Promise<Promise<boolean>>>>;
type T4 = Awaited<number | Promise<string>>;
type T5 = Awaited<Promise<number> | Promise<string>>;
type T6 = Awaited<Promise<{ data: Promise<User[]> }>>;
type T7 = Awaited<{ then(cb: (value: number) => void): void }>;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type T1 = string;

type T2 = number;
// 2 ta Promise unwrap

type T3 = boolean;
// 3 ta Promise unwrap

type T4 = number | string;
// Union'da ham distribute

type T5 = number | string;

type T6 = { data: Promise<User[]> };
// Outer Promise unwrap, lekin inner Promise (data field'da) unwrap qilinmaydi
// Awaited faqat outer-level Promise chain'ni unwrap qiladi

type T7 = number;
// Thenable object — then method'i bilan
```

### To'liq tushuntirish

- T1-T3 — recursive unwrap, nested Promise to'liq tozalanadi
- T4, T5 — union distribution, har member alohida unwrap
- T6 — Awaited faqat top-level Promise chain'ni unwrap qiladi, nested object'ning property'lari ichidagi Promise tegmaydi
- T7 — thenable (Promise emas, lekin `then` method'i bor) — Awaited unwrap qiladi

### Edge Cases

- Object property ichidagi Promise — Awaited tegmaydi, alohida unwrap kerak
- Custom thenable — `then` method signature `(cb: (v: V) => void) => void` shaklida bo'lsa, V unwrap qilinadi

</details>

---

## Coding challenges

### Savol 14: `RequireFields<T, K>` — selective required [Middle+]

**Savol:** `T`'da `K` belgilangan field'larni majburiy, qolganlari optional qiladigan utility yozing:

```typescript
interface Options {
  debug?: boolean;
  timeout?: number;
  retries?: number;
}

// RequireFields<Options, "timeout"> → { debug?: boolean; retries?: number; timeout: number }
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`Omit<T, K>` qolgan field'larni saqlaydi (optional state'i bilan). `Required<Pick<T, K>>` belgilangan K field'larni majburiy qiladi. Intersection ikkalasini birlashtiradi.

### Kod misol

```typescript
type RequireFields<T, K extends keyof T> =
  Omit<T, K> & Required<Pick<T, K>>;

interface Options {
  debug?: boolean;
  timeout?: number;
  retries?: number;
}

type WithTimeout = RequireFields<Options, "timeout">;
// { debug?: boolean; retries?: number; timeout: number }

type WithTimeoutAndRetries = RequireFields<Options, "timeout" | "retries">;
// { debug?: boolean; timeout: number; retries: number }

// Real-world API — required ID, qolgan partial
interface UpdateUserDto {
  id: number;
  name?: string;
  email?: string;
  age?: number;
}

type SafeUpdate = RequireFields<UpdateUserDto, "id">;
// id majburiy, qolganlari optional

// Counterpart — PartialFields
type PartialFields<T, K extends keyof T> =
  Omit<T, K> & Partial<Pick<T, K>>;

interface FullUser {
  id: number;
  name: string;
  email: string;
}

type DraftUser = PartialFields<FullUser, "email">;
// { id: number; name: string; email?: string }
```

### Edge Cases

- `K` keyof T constraint — typo tutadi (strict)
- Optional field'ning `| undefined` value type'i — `Required` `-?` undefined ni ham olib tashlaydi
- Discriminated union — Pick distribution distributive (har member alohida), so'ng Required va Omit qo'llaniladi

### Follow-up savollar

1. **"`RequireFields` `null` value bilan qanday?"** — `Required<Pick<T, K>>` `null` value'ni saqlaydi (faqat optional marker va undefined olib tashlanadi). `null` ham filter qilish uchun `NonNullable<Pick<T, K>>` kerak.
2. **"`Optional<T, K>` qanday yoziladi?"** — Inverse — `Omit<T, K> & Partial<Pick<T, K>>`.

</details>

### Savol 15: Complex DTO patterns — utility combination [Senior]

**Savol:** `User` interface'dan quyidagi DTO type'larni yarating:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// CreateUserDto — id, createdAt, updatedAt yo'q (server yaratadi)
// UpdateUserDto — id majburiy, qolganlar optional, server field'larsiz
// PublicUser — password yo'q, readonly
// Merge<T, U> — ikki type birlashtirish, U ustun
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Real-world API design uchun klassik pattern'lar. Har utility combination'i ma'lum security yoki business invariant'ni type-level'da kafolatlaydi.

### Kod misol

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// 1. Create DTO — server-managed field'larsiz
type CreateUserDto = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; password: string; role: "admin" | "user" }

// 2. Update DTO — id majburiy, qolgan partial
type UpdateUserDto =
  Pick<User, "id"> &
  Partial<Omit<User, "id" | "createdAt" | "updatedAt">>;
// { id: number; name?: string; email?: string; password?: string; role?: ... }

// 3. Public response — password yo'q, readonly
type PublicUser = Readonly<Omit<User, "password">>;
// { readonly id: number; readonly name: string; ... readonly updatedAt: Date }

// 4. Merge — U field'lar T ni override qiladi
type Merge<T, U> = Omit<T, keyof U> & U;

type UpdatedUser = Merge<User, { email: string[]; verified: boolean }>;
// User + email type'i string[] ga o'zgardi + verified qo'shildi

// 5. Real API service signature
interface UserService {
  create(dto: CreateUserDto): Promise<PublicUser>;
  update(dto: UpdateUserDto): Promise<PublicUser>;
  findById(id: number): Promise<PublicUser | null>;
}

// Usage
declare const service: UserService;

const created = await service.create({
  name: "Ali",
  email: "ali@test.com",
  password: "secret",
  role: "user",
});
// created: PublicUser — password yo'q ✅

// const dto: CreateUserDto = { ... id: 1 ... };
// ❌ id ortiqcha (excess property check)
```

### Edge Cases

- `Omit<T, K>` discriminated union'da distribution yo'q — manual `T extends any ? Omit<T, K> : never` kerak
- `Readonly<T>` shallow — nested object readonly qilish uchun `DeepReadonly` kerak
- `Merge` da U bo'sh bo'lsa — T o'zgarishsiz qaytadi
- Password field — `PublicUser`'da yo'qligi guarantee, lekin runtime'da sanitization alohida kerak (compile-time check yetarli emas)

### Follow-up savollar

1. **"`Merge` deep variant qanday yoziladi?"** — Recursive — har nested key uchun, T va U'da bor bo'lsa recursive merge.
2. **"`Readonly<Omit<...>> & { ... }` — intersection mutable field qoldiradi mi?"** — Ha, intersection part alohida — readonly modifier qo'llanilmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

**DTO pattern'larning compile-time invariant'lari**

Har DTO type structural contract'ni kafolatlaydi:

- `CreateUserDto` — `id` field assign qilinmasligi compile-time'da bloklanadi (excess property check). Bu race condition'ni oldini oladi (client `id` yubormaydi).
- `UpdateUserDto` — `id` majburiy lekin qolgan property'lar optional. PATCH semantics modellanadi.
- `PublicUser` — `password` field'i type'da yo'q. Mistake'lar (e.g., `res.json(user)`) compile-time'da topiladi.

Lekin: type-level guarantee runtime sanitization'ni almashtirmaydi. Object'da haqiqiy `password` field bo'lishi mumkin (untyped JSON parse, type cast). Runtime'da explicit `delete user.password` yoki Zod schema parse zarur.

**`Merge<T, U>` semantics**

```typescript
type Merge<T, U> = Omit<T, keyof U> & U;
```

`keyof U` — U'da bor barcha key'lar. `Omit<T, keyof U>` — T'dan shu key'larni olib tashlaydi. So'ng U qo'shiladi. Natija: U property'lari T'ni override qiladi.

Edge case — optional vs required:

```typescript
type T = { a: string; b?: number };
type U = { b: string };
type R = Merge<T, U>;
// { a: string } & { b: string } = { a: string; b: string }
// b optional bo'lgan edi, U'da required bo'ldi — Merge required'ga aylantirdi
```

**Discriminated union va `Omit` distribution problem**

```typescript
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "scroll"; delta: number };

type WithoutType = Omit<Event, "type">;
// ❌ Kutilgan: { x: number; y: number } | { delta: number }
// ✅ Haqiqiy: {}
// Sabab: Omit<T, K> = Pick<T, Exclude<keyof T, K>>. Union'da keyof faqat
// UMUMIY key'larni beradi → keyof Event = "type". Exclude<"type", "type"> = never.
// Pick<Event, never> = {} — barcha variant-specific property yo'qoladi.

// Distributive version
type DistributiveOmit<T, K extends keyof any> =
  T extends any ? Omit<T, K> : never;

type Good = DistributiveOmit<Event, "type">;
// { x: number; y: number } | { delta: number } — har variant alohida
```

**`Readonly<T> & U` intersection — readonly inheritance**

```typescript
type PublicUser = Readonly<Omit<User, "password">> & { sessionId: string };
// {
//   readonly id: number; readonly name: string; ...
//   sessionId: string;  // ← readonly EMAS
// }
```

`Readonly` mapped type faqat o'z argumenti'ga qo'llaniladi. Intersection'ning ikkinchi qismi (`{ sessionId: string }`) alohida type — modifier'lar transitive emas. Agar barcha qismlar readonly bo'lishi kerak bo'lsa:

```typescript
type PublicUser = Readonly<Omit<User, "password"> & { sessionId: string }>;
// Avval intersection, keyin Readonly qo'llanadi
```

</details>

</details>

### Savol 16: `AsyncMethods<T>` — barcha method'larni async qilish [Senior]

**Savol:** Class interface'idan barcha method'larni async (Promise return) qiladigan type yozing:

```typescript
class UserService {
  getUser(id: number): User { ... }
  updateUser(id: number, data: Partial<User>): User { ... }
  deleteUser(id: number): boolean { ... }
  cache: Map<number, User>;  // method emas — skip
}

// AsyncMethods<UserService>:
// {
//   getUser: (id: number) => Promise<User>;
//   updateUser: (id: number, data: Partial<User>) => Promise<User>;
//   deleteUser: (id: number) => Promise<boolean>;
// }
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Mapped type + conditional + infer kombinatsiyasi. Function method'larni ajratish va return type'ni `Promise`'ga wrap qilish.

### Kod misol

```typescript
type AsyncMethods<T> = {
  [K in keyof T as T[K] extends Function ? K : never]:
    T[K] extends (...args: infer A) => infer R
      ? (...args: A) => Promise<R extends Promise<any> ? Awaited<R> : R>
      : never;
};

class UserService {
  getUser(id: number): User { return {} as User }
  updateUser(id: number, data: Partial<User>): User { return {} as User }
  deleteUser(id: number): boolean { return true }
  cache: Map<number, User> = new Map();  // method emas
}

type RemoteService = AsyncMethods<UserService>;
// {
//   getUser: (id: number) => Promise<User>;
//   updateUser: (id: number, data: Partial<User>) => Promise<User>;
//   deleteUser: (id: number) => Promise<boolean>;
// }
// cache field — filterlandi

// Real-world: RPC client wrapper
declare function createRpcClient<T>(url: string): AsyncMethods<T>;

const remoteUserService = createRpcClient<UserService>("https://api.example.com");
const user = await remoteUserService.getUser(1);
// user: User ✅

// Already async — double wrap yo'q
class AsyncRepository {
  fetch(): Promise<User[]> { return Promise.resolve([]) }
}

type RemoteRepo = AsyncMethods<AsyncRepository>;
// { fetch: () => Promise<User[]> }
// Promise<Promise<User[]>> emas — Awaited bilan flatten
```

### Edge Cases

- Already-async method'lar — double Promise wrap oldini olish uchun `Awaited` ishlatilgan
- Generic method'lar — generic parameter saqlash uchun maxsus syntax kerak (TS limit'i — generic preserve qiyin)
- Static method'lar — `typeof Class` orqali alohida ishlov
- Class private/protected — `keyof T` faqat public'larni qaytaradi

### Follow-up savollar

1. **"Generic method bilan qanday ishlatish kerak?"** — Generic preserve uchun manual instantiation yoki TS plugin (advanced).
2. **"Real RPC framework qaysi shu pattern'ni ishlatadi?"** — tRPC, gRPC-web TS, Hasura GraphQL codegen.

<details>
<summary><strong>Deep Dive</strong></summary>

**Key remapping `as` syntax (TS 4.1+)**

```typescript
type AsyncMethods<T> = {
  [K in keyof T as T[K] extends Function ? K : never]: ...
};
```

`as` clause TS 4.1'da kiritilgan — mapped type'da key'larni filter yoki rename qilish uchun. `T[K] extends Function ? K : never` distributive — har property uchun:

- Method bo'lsa: key saqlanadi (`K`).
- Field bo'lsa: `never` — key olib tashlanadi.

`as` clause `never` qaytarsa, mapped type o'sha key'ni natija type'idan butunlay tashlab yuboradi. Bu native TS filtering mexanizmi — alohida `FilterMethods<T>` yozish kerak emas.

**`Awaited` bilan double-wrap oldini olish**

```typescript
T[K] extends (...args: infer A) => infer R
  ? (...args: A) => Promise<R extends Promise<any> ? Awaited<R> : R>
  : never;
```

Conditional `R extends Promise<any>` tekshiradi:

- Method allaqachon async (`Promise<X>` qaytaradi) → `Awaited<R>` = `X`, natija `Promise<X>` (double wrap yo'q).
- Method sync (`User` qaytaradi) → conditional false, natija `Promise<User>`.

Sodda variant: `Promise<Awaited<R>>` — har holatda flatten qiladi (Awaited non-Promise'ga ta'sir qilmaydi).

**Generic method preservation problem**

```typescript
class Repo {
  find<T>(id: T): T { return id; }
}

type AsyncRepo = AsyncMethods<Repo>;
// { find: (id: unknown) => Promise<unknown> }
// ❌ Generic <T> yo'qoldi — mapped type'da generic preserve qilish TS limit'i
```

TypeScript mapped type'da `infer` clauses statik — generic parameter context'ini saqlay olmaydi. Higher-kinded types (TS'da yo'q) bo'lsa hal qilinishi mumkin edi.

Workaround — manual overload type:

```typescript
type AsyncRepo = {
  find<T>(id: T): Promise<T>;
};
```

**`keyof T` private/protected behavior**

```typescript
class UserService {
  public getUser() {}
  private validate() {}
  protected normalize() {}
}

type Keys = keyof UserService;
// "getUser" — faqat public
```

`keyof` TypeScript'ning `private`/`protected` visibility'sini hurmat qiladi — bu modifier'lar compile-time only construct (ECMAScript `#field` emas), shu sababli `keyof` ulardan faqat public member'larni qaytaradi. `AsyncMethods<UserService>` faqat public method'larni wrap qiladi. `private`/`protected` member'lar runtime'da oddiy property sifatida mavjud (TS emit ularni o'chirmaydi), lekin type-level'da `keyof`'ga kirmaydi.

</details>

</details>

### Savol 17: `Mutable<T>` va `DeepMutable<T>` [Senior]

**Savol:** `Readonly<T>` qarama-qarshisi — readonly modifier'ni olib tashlash:

```typescript
interface ReadonlyConfig {
  readonly port: number;
  readonly server: {
    readonly host: string;
    readonly options: { readonly ssl: boolean };
  };
  readonly tags: readonly string[];
}

// Mutable — shallow
// DeepMutable — recursive
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`-readonly` modifier (TS 2.8+) bilan readonly olib tashlanadi. Deep variant recursive — base case bilan (function, primitive).

### Kod misol

```typescript
// Shallow Mutable
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Deep Mutable — recursive
type DeepMutable<T> =
  T extends (...args: any[]) => any ? T :
  T extends ReadonlyArray<infer U> ? Array<DeepMutable<U>> :
  T extends object ? { -readonly [K in keyof T]: DeepMutable<T[K]> } :
  T;

interface ReadonlyConfig {
  readonly port: number;
  readonly server: {
    readonly host: string;
    readonly options: { readonly ssl: boolean };
  };
  readonly tags: readonly string[];
}

// Shallow
type ShallowMutable = Mutable<ReadonlyConfig>;
// {
//   port: number;
//   server: { readonly host: string; ... };  // nested readonly qoldi
//   tags: readonly string[];                  // ReadonlyArray qoldi
// }

// Deep
type FullMutable = DeepMutable<ReadonlyConfig>;
// {
//   port: number;
//   server: {
//     host: string;
//     options: { ssl: boolean };  // deep readonly olib tashlandi
//   };
//   tags: string[];                // mutable Array
// }

const config: FullMutable = {
  port: 3000,
  server: { host: "localhost", options: { ssl: true } },
  tags: ["dev"],
};

config.port = 4000;                  // ✅
config.server.host = "remote";       // ✅
config.tags.push("prod");            // ✅
config.server.options.ssl = false;   // ✅
```

### Edge Cases

- `ReadonlyArray<T>` → `Array<T>` — alohida branch zarur, aks holda `object` branch `ReadonlyArray`'ni structural object sifatida map qiladi va `length`/index property'larini transform qilib yuboradi
- Tuple `readonly [number, string]` → shallow `Mutable` tuple shape'ni saqlaydi, lekin `DeepMutable`'ning `ReadonlyArray<infer U>` branch'i tuple'ni `Array<number | string>`'ga aylantiradi (yuqoridagi Deep Dive ko'rsatilgan)
- Function — base case, o'zgarishsiz
- Class instance — method'lar saqlanadi, property'lar mutable bo'ladi

### Follow-up savollar

1. **"Nima uchun TS 2.8'da `-readonly` kiritilgan?"** — Original'da faqat `readonly` qo'shish edi. Olib tashlash uchun yangi syntax kerak edi — `Required<T>` va `Mutable<T>` implement qilish uchun.
2. **"`as const` `readonly` modifier'ini qanday olib tashlash kerak?"** — `Mutable<typeof config>` faqat `readonly` modifier'ni olib tashlaydi (literal type'lar saqlanadi: `{ readonly port: 3000 }` → `{ port: 3000 }`). Literal'ni primitive'ga kengaytirish alohida operatsiya — `Mutable` buni qilmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Mapping modifier syntax (TS 2.8+)**

TS 2.8 mapped type'larga ikki yangi modifier syntax kiritdi:

| Modifier | Semantika |
|----------|-----------|
| `readonly` yoki `+readonly` | readonly qo'shish (default behavior) |
| `-readonly` | readonly olib tashlash |
| `?` yoki `+?` | optional qo'shish |
| `-?` | optional olib tashlash |

`+` prefix odatda yozilmaydi (default), lekin `-` prefix homomorphic mapped type'da modifier olib tashlash uchun kerak.

**Homomorphic mapping va modifier preservation**

```typescript
type Identity<T> = { [K in keyof T]: T[K] };
// Homomorphic — readonly/optional modifier'lar avtomatik saqlanadi

type StripReadonly<T> = { -readonly [K in keyof T]: T[K] };
// -readonly explicit — modifier olib tashlanadi
```

Homomorphic mapped type (where keys come from `keyof T`):

- Modifier'lar default'da preserve qilinadi.
- `+` yoki `-` explicit modifier yozilsa override qilinadi.

Non-homomorphic mapped type (e.g., `{ [K in "a" | "b"]: T }`) modifier inherit qilmaydi.

**ReadonlyArray va Array branch**

```typescript
T extends ReadonlyArray<infer U> ? Array<DeepMutable<U>> : ...
```

`ReadonlyArray<T>` interface bilan `Array<T>`'dan strukturaviy farq qiladi — `push`, `pop`, `splice` method'lari yo'q. Mapped type bilan `-readonly` qo'llanganda array'ning element-level modifier'lariga ta'sir qiladi, lekin `ReadonlyArray` o'zining strukturasini saqlaydi.

Shu sababli explicit branch: `ReadonlyArray<U> → Array<DeepMutable<U>>` — interface'ni almashtirish.

Tuple holati:

```typescript
type Pair = readonly [number, string];

type M = Mutable<Pair>;
// [number, string] — homomorphic mapped type tuple shape'ni saqlaydi,
// -readonly tuple'ni mutable qildi

// Lekin DeepMutable:
type DM = DeepMutable<Pair>;
// (number | string)[] — ReadonlyArray<infer U> branch U = number | string
// infer qildi, natija Array<number | string>; tuple shape yo'qoldi
```

Shallow `Mutable<T>` — `{ -readonly [K in keyof T]: T[K] }` homomorphic, `keyof Pair` tuple index'larini (`0`, `1`, `length`, ...) bo'ylab map qiladi va tuple shape'ni saqlab, faqat `-readonly` modifier'ni qo'llaydi → mutable tuple.

`DeepMutable<T>` esa avval `T extends ReadonlyArray<infer U>` branch'iga tushadi: `readonly [number, string]` `ReadonlyArray<number | string>`'ga assignable, shuning uchun `U = number | string` infer qilinadi va natija `Array<number | string>` — tuple shape'i `infer U` orqali bitta union'ga yig'iladi. Tuple shape'ni saqlab deep mutable qilish uchun alohida tuple branch yoziladi:

```typescript
type DeepMutableTuple<T> =
  T extends readonly [...infer Rest]
    ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
    : T;
```

**Function va class instance — base case**

```typescript
T extends (...args: any[]) => any ? T : ...
```

Function — primitive type-like, structural transformation TAQIQ. Funksiya ichidagi parameter'lar va return type alohida transform qilinmaydi (semantically xato — function signature contract).

Class instance method'lari `keyof` orqali ko'rinadi. `DeepMutable<UserService>` method'larni saqlaydi, lekin instance field'larni mutable qiladi:

```typescript
class UserService {
  public readonly id: number = 1;
  greet() { return "hi"; }
}

type M = DeepMutable<UserService>;
// { id: number; greet: () => string; }
// readonly olib tashlandi, method saqlandi
```

</details>

</details>

---

## Bug fix

### Savol 18: `typeof` gotcha — ReturnType<getUser> [Middle]

**Savol:** Bu kodda xato bor. Toping va tuzating:

```typescript
function getUser() {
  return { id: 1, name: "Ali", email: "ali@test.com" };
}

type UserType = ReturnType<getUser>;
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`ReturnType<getUser>` — `getUser` runtime **value**, type emas. `typeof` operator runtime value'dan type olish uchun kerak.

### Kod misol

```typescript
// ❌ Error: 'getUser' refers to a value, but is being used as a type here
type UserType = ReturnType<getUser>;

// ✅ typeof bilan
type UserType = ReturnType<typeof getUser>;
// { id: number; name: string; email: string }

// Qoida: ReturnType, Parameters, InstanceType, ConstructorParameters
// hammasi TYPE qabul qiladi. Runtime value'dan type olish uchun typeof kerak

type R = ReturnType<typeof someFunction>;
type P = Parameters<typeof someFunction>;
type I = InstanceType<typeof SomeClass>;
type C = ConstructorParameters<typeof SomeClass>;

// Interface'lar uchun typeof shart emas
interface Fetcher {
  fetch(url: string): Promise<Response>;
}

type FetchReturn = ReturnType<Fetcher["fetch"]>;
// Promise<Response> — Fetcher interface, typeof shart emas
```

### Edge Cases

- Method'lar uchun — `T["methodName"]` indexing
- Class method (instance) — `InstanceType<typeof Class>["method"]` yoki `Class["prototype"]["method"]`
- Generic function — concrete type berib instantiate qilish kerak (avtomatik infer yo'q)

### Follow-up savollar

1. **"`typeof` qachon kerak emas?"** — Interface, type alias, class type'lari bilan (ular allaqachon type). Function value, class value (constructor), const value bilan kerak.

</details>

### Savol 19: `Required<T>` `| undefined` ni tozalamaydi [Middle+]

**Savol:** Nima uchun `Required<Form>` `phone` dan `undefined` ni olib tashlamaydi? Qanday tuzatish kerak?

```typescript
interface Form {
  name?: string;
  phone: string | undefined;
}

type RequiredForm = Required<Form>;
// phone hali ham string | undefined
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`Required<T>` `-?` faqat **optional marker** (`?`) ni olib tashlaydi. `phone: string | undefined` — marker yo'q, `undefined` esa union type'ning bir qismi. `-?` value type ichidagi `undefined` ni olib tashlamaydi.

### Kod misol

```typescript
interface Form {
  name?: string;                  // optional = marker + undefined
  phone: string | undefined;      // required, undefined union member
}

type RequiredForm = Required<Form>;
// {
//   name: string;                // ? va undefined ikkalasi ketdi
//   phone: string | undefined;   // ? yo'q edi, undefined qoldi
// }

// Sabab:
// name?: string → -? → name: string (marker + value type'dagi undefined ketadi)
// phone: string | undefined → -? → string | undefined (marker yo'q, hech narsa o'zgarmaydi)

// Yechim — StrictRequired
type StrictRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type StrictForm = StrictRequired<Form>;
// {
//   name: string;
//   phone: string;  // undefined va null tozalandi ✅
// }

// Faqat undefined tozalash (null saqlash)
type RequiredNoUndefined<T> = {
  [K in keyof T]-?: Exclude<T[K], undefined>;
};

interface WithNull {
  value?: string | null;
}

type Cleaned1 = StrictRequired<WithNull>;
// { value: string }  — null ham ketdi

type Cleaned2 = RequiredNoUndefined<WithNull>;
// { value: string | null }  — null saqlandi
```

### Edge Cases

- `NonNullable<T>` — null VA undefined ikkalasini olib tashlaydi
- `Exclude<T, undefined>` — faqat undefined olib tashlaydi
- `exactOptionalPropertyTypes: true` flag — `name?: string` ga `undefined` explicit assign qilolmaydi (strict)
- Deep version — recursive `StrictRequired` yozish kerak (DeepStrictRequired)

### Follow-up savollar

1. **"`exactOptionalPropertyTypes` flag nima qiladi?"** — `name?: string` ni faqat string yoki yo'q (key bo'lmasligi) bilan cheklaydi. `name: undefined` explicit yozish xato.
2. **"`Required<T>` TS 2.8'gacha qanday yozilardi?"** — `-?` modifier 2.8'da kiritilgan. Undan oldin Required mavjud emas edi.

</details>

---

## Xulosa

- **`Partial`/`Required`/`Readonly`** — **shallow**, faqat top-level. Deep variant'lar recursive
- **`Omit`** — `keyof any` (loose), typo tutmaydi. `StrictOmit<T, K extends keyof T>` xavfsizroq
- **`Required<T>`** `-?` — optional marker olib tashlaydi, lekin `phone: string | undefined` (marker'siz) tegmaydi. `NonNullable<T[K]>` bilan birga
- **`Exclude`/`Extract`** — distributive conditional, union har member alohida tekshiriladi
- **`ReturnType`/`Parameters`** — conditional + `infer`. Overload faqat oxirgi signature, generic function uchun `unknown` default
- **`ConstructorParameters`/`InstanceType`** — class factory uchun. `abstract new` TS 4.2+'dan
- **`ThisParameterType`/`OmitThisParameter`** — `this` parametr bilan method'lar uchun (bind, call/apply type)
- **`Awaited<T>`** — recursive Promise unwrap, `ReturnType` async uchun yetarli emas. TS 4.5+
- **`NoInfer<T>`** (TS 5.4+) — inference cheklash uchun, bitta argument'dan infer qilish
- **`typeof`** — utility type'lar TYPE qabul qiladi, runtime value'dan type olish uchun
- **3 ta mexanizm:** mapped types, conditional + infer, intrinsic
- **Zero cost** — barcha utility type'lar compile'da o'chiriladi

**Keyingi bo'lim:** [16-custom-utility-types.md](16-custom-utility-types.md) — Custom utility type'lar: `DeepPartial`, `DeepReadonly`, `Merge`, distributive conditional patterns.
