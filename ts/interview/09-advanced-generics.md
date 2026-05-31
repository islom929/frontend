# Interview: Advanced Generics

> Conditional types, infer keyword, distributive conditional types, non-distributive conditional types, mapped types, template literal types, recursive types, variadic tuples, NoInfer, higher-kinded types workaround va advanced generic patterns bo'yicha interview savollari.

---

## Nazariy savollar

### Savol 1: Conditional type nima? Distributive conditional qachon ishga tushadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Conditional type — `T extends U ? X : Y` — type-level if/else. Type parameter naked (oddiy, bracket'siz) bo'lsa va union berilsa — TS har union member'ga alohida qo'llaydi (distributive). `[T] extends [U]` — non-distributive.

### To'liq tushuntirish

Conditional type ikki rejimda ishlaydi:

1. **Distributive** — `T extends U ? X : Y` bilan T naked type parameter va T union bo'lsa:
   - `(A | B | C) extends U ? X : Y` = `(A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y)`

2. **Non-distributive** — `[T] extends [U] ? X : Y` (tuple wrapper) bilan union bir butun deb tekshiriladi.

### Kod misol

```typescript
// Distributive
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">;             // true
type B = IsString<42>;                  // false
type C = IsString<string | number>;     // true | false = boolean — distributive!

// Non-distributive
type IsStringStrict<T> = [T] extends [string] ? true : false;
type D = IsStringStrict<string | number>; // false — non-distributive, union bir butun

// Real-world: Filter union
type ExtractString<T> = T extends string ? T : never;
type E = ExtractString<string | number | boolean>; // string

type ExcludeString<T> = T extends string ? never : T;
type F = ExcludeString<string | number | boolean>; // number | boolean

// Standard utility'lar shu pattern
type G = Extract<string | number, string>;  // string
type H = Exclude<string | number, string>;  // number

// Function type infer
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;
type I = ReturnTypeOf<() => number>;        // number
type J = ReturnTypeOf<string>;              // never
```

### Edge Cases

- **`never` distributive** — `never` = bo'sh union → 0 member → natija `never` (har distributive conditional'da).
- **`any` conditional** — `any extends U ? X : Y` = `X | Y` (har ikki branch).
- **Constraint vs conditional** — `<T extends string>` constraint generic'da, conditional type level'da. Farq: constraint compile-time check, conditional type transformation.
- **Distributive only at top** — `T extends U ? X[] : Y[]` distributive (T top-level), `Array<T extends U ? X : Y>` ham distributive (conditional ichkarida). Lekin `[T]` wrapper conditional'da non-distributive.

### Follow-up savollar

1. **"`T extends U` boolean qaytaradi degan tushuncha to'g'rimi?"** — Yo'q, type-level type qaytaradi (`X` yoki `Y`). `IsString<T>` `true | false` qaytarsa ham, bu literal type'lar, runtime boolean emas.
2. **"Conditional type'ning resolution algoritmi?"** — checker `T` (check type) va `U` (extends type) o'rtasida assignability tekshiradi. Generic'da `T` hali substitute bo'lmagan bo'lsa, conditional deferred holatda qoladi va faqat type argument berilganda resolve bo'ladi.

</details>

---

### Savol 2: `infer` keyword nima? Qayerda va qanday ishlatiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`infer X` — conditional type'da pattern matching: TS X position'idagi type'ni infer qiladi va X o'zgaruvchisiga binds. Faqat conditional type'ning `extends` qismida ishlaydi.

### To'liq tushuntirish

`infer` ikki rol bajaradi:

1. **Extract type from structure** — `T extends (...args: any) => infer R ? R : never`
2. **Multiple infer** — bir conditional'da bir nechta `infer`, har biri alohida pozitsiya

`infer` faqat conditional type'ning `extends` qismida ishlaydi. Generic constraint'da yoki oddiy type'da ishlatib bo'lmaydi.

### Kod misol

```typescript
// Return type — function signature'dan R infer
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type A = GetReturnType<() => string>;        // string
type B = GetReturnType<(x: number) => boolean>; // boolean

// Parameters — argument tuple infer
type GetParams<T> = T extends (...args: infer P) => any ? P : never;
type C = GetParams<(a: string, b: number) => void>; // [a: string, b: number]

// Array element — element type infer
type ArrayElement<T> = T extends (infer U)[] ? U : never;
type D = ArrayElement<string[]>;             // string

// Promise unwrap
type Awaited<T> = T extends Promise<infer U> ? U : T;
type E = Awaited<Promise<number>>;           // number

// Multiple infer
type Swap<T> = T extends [infer A, infer B] ? [B, A] : T;
type F = Swap<[string, number]>;             // [number, string]

// Constrained infer (TS 4.7+)
type GetNumber<T> = T extends `${infer N extends number}` ? N : never;
type G = GetNumber<"42">;                    // 42 (number literal)

// Nested infer
type DeepReturn<T> =
  T extends (...args: any) => infer R
    ? R extends Promise<infer U>
      ? U
      : R
    : never;
type H = DeepReturn<() => Promise<string>>;  // string

// Tuple head infer — bo'sh tuple uchun fallback branch
type FirstOrDefault<T> = T extends [infer F, ...any] ? F : never;
type I = FirstOrDefault<[1, 2, 3]>;          // 1
type J = FirstOrDefault<[]>;                 // never
```

### Edge Cases

- **Multiple matches** — `T extends { a: infer U; b: infer U }` — U barcha pozitsiya'lar union'i (homogeneous variance position) yoki intersection (contravariant position).
- **Constrained infer (TS 4.7+)** — `infer N extends number` — infer'da constraint, faqat constraint mos kelsa branch tanlanadi.
- **`infer` covariant vs contravariant** — return position (covariant) — union. Parameter position (contravariant) — intersection.
- **Recursive infer** — recursive conditional bilan kombinatsiya, depth limit'iga ehtiyot bo'lish kerak.
- **`infer` faqat extends'da** — `function fn<T extends infer U>` — xato. Faqat conditional type ichida.

### Follow-up savollar

1. **"Tuple destructuring infer bilan qanday?"** — `T extends [infer Head, ...infer Rest] ? ... : ...` — JavaScript destructuring'ning type-level ekvivalenti. Variadic tuple bilan.
2. **"`infer` performance ta'siri?"** — Murakkab `infer` zanjirlari TS checker'ni sekinlashtiradi. Recursive infer + variadic tuple — eng katta performance hit.

<details>
<summary><strong>Deep Dive</strong></summary>

**Variance positions for infer:**

`infer` joylashgan pozitsiya conditional type'ning natijasiga ta'sir qiladi:

- **Covariant position** (return type, property type, array element) — multiple `infer U` natijasi **union** bo'ladi
- **Contravariant position** (function parameter) — multiple `infer U` natijasi **intersection**

```typescript
// Covariant — return position
type ReturnUnion<T> = T extends { a: () => infer U; b: () => infer U } ? U : never;
type X = ReturnUnion<{ a: () => string; b: () => number }>; // string | number

// Contravariant — parameter position
type ParamIntersection<T> = T extends { a: (x: infer U) => void; b: (x: infer U) => void } ? U : never;
type Y = ParamIntersection<{ a: (x: { a: 1 }) => void; b: (x: { b: 2 }) => void }>;
// { a: 1 } & { b: 2 }
```

**Constrained infer (TS 4.7+) implementation:**

`infer N extends number` — TS template literal'dan number parse qiladi, constraint mos kelmasa branch fail. Bu compile-time numeric reasoning'ga yo'l ochdi (string '42' → number 42).

</details>

</details>

---

### Savol 3: `never` distributive conditional'da nima uchun tricky? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`never` — bo'sh union (0 member). Distributive conditional har union member'ga alohida qo'llaydi — 0 ta member uchun 0 ta natija — `never`. `[T]` wrapper bilan non-distributive qilsak, `never` oddiy type sifatida tekshiriladi.

### To'liq tushuntirish

`IsNever` paradox:

```typescript
type IsString<T> = T extends string ? true : false;
type A = IsString<never>;  // never — true ham false ham emas
```

Distributive conditional `never` ga 0 marta qo'llanadi → natija `never`.

Yechim — non-distributive wrapper:

```typescript
type IsNever<T> = [T] extends [never] ? true : false;
type B = IsNever<never>;     // true
type C = IsNever<string>;    // false
```

### Kod misol

```typescript
// Direct (generic emas) — never bottom type
type Direct = never extends string ? true : false; // true

// Generic naked — distributive
type Naked<T> = T extends string ? true : false;
type X = Naked<never>;       // never — bo'sh union → 0 natija

// Wrapped — non-distributive
type Wrapped<T> = [T] extends [string] ? true : false;
type Y = Wrapped<never>;     // true — never assignable to string

// IsNever utility
type IsNever<T> = [T] extends [never] ? true : false;
type Z = IsNever<never>;     // true

// Union'dan never'ni alohida filter qilishga hojat yo'q:
// never union'da avtomatik absorb bo'ladi
type Absorbed = string | never;   // string — never yo'qoladi

// Object property'dan never'ni o'chirish
type NonNeverKeys<T> = {
  [K in keyof T]: [T[K]] extends [never] ? never : K;
}[keyof T];

type Test = {
  a: string;
  b: never;
  c: number;
};
type Keys = NonNeverKeys<Test>; // "a" | "c"
```

### Edge Cases

- **`never extends X` direct** — true (never bottom). Generic'da naked T = never — distributive, natija never.
- **`never[]` array** — `never[]` element'larsiz array, har array'ga assignable.
- **`Promise<never>`** — har rejection bilan complete, hech qachon resolve qilmaydigan promise type.
- **Function return `never`** — funksiya hech qachon return qilmaydi (throw, infinite loop). Exhaustiveness check'da foydali.
- **`Record<string, never>`** — bo'sh string-keyed object (har string key uchun never qiymat — yaratib bo'lmaydi).

### Follow-up savollar

1. **"Discriminated union'da `never`'ning roli?"** — Exhaustiveness check: `switch (tag) { case "a": ...; default: const _: never = tag; }` — har case qamralganini compile-time'da tekshirish.
2. **"`never` va `void` farqi nima?"** — `void` — funksiya return qilmaydi (`undefined` qaytaradi). `never` — funksiya hech qachon tugamaydi yoki har doim throw qiladi.

</details>

---

### Savol 4: Mapped type nima? Modifier'lar (`readonly`, `?`, `-?`) qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Mapped type — `{ [K in Keys]: ValueType }` — har key uchun yangi property type yaratish. Modifier'lar: `readonly`/`-readonly` (mutability), `?`/`-?` (optionality). `-` prefix — modifier'ni olib tashlash.

### To'liq tushuntirish

Sintaksis: `{ [P in K]: T }` — K har element uchun T qiymatli property yaratiladi.

Modifier'lar:
- `readonly` — qo'shadi, `-readonly` — olib tashlaydi
- `?` — optional qiladi, `-?` — required qiladi

`as` clause (TS 4.1+) — key'ni qayta nomlash (template literal types bilan).

### Kod misol

```typescript
interface User { name: string; age: number; email: string; }

// Built-in utility'lar
type Partial<T> = { [K in keyof T]?: T[K] };
type Required<T> = { [K in keyof T]-?: T[K] };
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// Custom mapped type
type Nullable<T> = { [K in keyof T]: T[K] | null };
type N = Nullable<User>;
// { name: string | null; age: number | null; email: string | null }

// Pick va Omit
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

type UserName = Pick<User, "name">;          // { name: string }
type WithoutEmail = Omit<User, "email">;     // { name: string; age: number }

// Key remapping (TS 4.1+) — `as`
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// {
//   getName: () => string;
//   getAge: () => number;
//   getEmail: () => string;
// }

// Filter keys with `as` + conditional
type StringKeys<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type S = StringKeys<User>;
// { name: string; email: string } — age o'chirildi

// Recursive mapped + conditional
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};
```

### Edge Cases

- **Modifier'lar ham distributive emas** — mapped type union T uchun har member'ga qo'llanmaydi (homomorphic transformation).
- **Homomorphic mapped type** — `{ [K in keyof T]: ... }` (keyof T to'g'ridan-to'g'ri ishlatilsa) — T'ning modifier'larini saqlaydi (optional, readonly).
- **Non-homomorphic** — `{ [K in "a" | "b"]: ... }` — T'ning modifier'larini olmaydi.
- **Modifier index signature'da ham ishlaydi** — homomorphic mapped type T'dagi index signature'ning `readonly`/optional modifier'larini ham saqlaydi yoki `+`/`-` bilan o'zgartiradi (`type Mutable<T> = { -readonly [K in keyof T]: T[K] }` index signature'ni ham mutable qiladi).
- **Key remapping `never`** — `[K in keyof T as never]` — property o'chiriladi. Filter pattern.

### Follow-up savollar

1. **"Homomorphic mapped type avtomatik modifier inheritance qanday?"** — TS spec'da: `{ [P in keyof T]: ... }` shaklida T'ning har property'sining modifier'lari (readonly, optional) avtomatik saqlanadi.
2. **"`as never` bilan property filtering vs `Omit` farqi?"** — `as never` mapped type ichida real-time filter. `Omit` alohida wrapper. `as never` `as` + conditional bilan murakkab filter qila oladi.

</details>

---

### Savol 5: Template literal types — `${T}` qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Template literal type — string literal type'larni runtime string template'larga o'xshash yaratish. `${T}` interpolation, union'lar bilan distributive — `\`${"a" | "b"}-x\`` = `"a-x" | "b-x"`.

### To'liq tushuntirish

Template literal type — TS 4.1+. String literal'lardan compile-time string operations:

- Interpolation: `\`${T}\``
- Distributive — union argument'lar har biri bilan natija
- Built-in utilities: `Uppercase<T>`, `Lowercase<T>`, `Capitalize<T>`, `Uncapitalize<T>`

### Kod misol

```typescript
// Asoslari
type Greeting = `Hello, ${string}`;
const a: Greeting = "Hello, World";
// const b: Greeting = "Hi, World"; // ❌

// Distributive
type CSSUnit = "px" | "em" | "rem";
type Size = `${number}${CSSUnit}`;
const s1: Size = "10px";
const s2: Size = "1.5em";

// Combine union'lar
type Direction = "top" | "right" | "bottom" | "left";
type Position = "start" | "end";
type Combined = `${Direction}-${Position}`;
// "top-start" | "top-end" | "right-start" | ... (8 ta)

// Capitalize event names
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<"click">; // "onClick"

// Path-based access
type DotPath<T, P extends string = ""> = {
  [K in keyof T & string]: T[K] extends object
    ? DotPath<T[K], `${P}${K}.`>
    : `${P}${K}`;
}[keyof T & string];

interface AppConfig {
  api: { url: string; port: number };
  ui: { theme: string };
}

type ConfigPath = DotPath<AppConfig>;
// "api.url" | "api.port" | "ui.theme"

// Parse template literal
type SplitPath<S extends string> =
  S extends `${infer Head}.${infer Tail}`
    ? [Head, ...SplitPath<Tail>]
    : [S];

type P = SplitPath<"a.b.c">; // ["a", "b", "c"]

// HTTP method routing
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type Endpoint = `/${"users" | "products" | "orders"}`;
type Route = `${HttpMethod} ${Endpoint}`;
// "GET /users" | "POST /users" | ... (12 ta)
```

### Edge Cases

- **Constrained infer (TS 4.7+)** — `T extends \`${infer N extends number}\` ? N : never` — string'dan number parse.
- **`${string}` placeholder** — har string'ga mos. `${number}` — string-representation of number.
- **Recursive template literal** — depth limit'iga urishi mumkin (uzun string'lar uchun).
- **Empty string** — `\`\`` = `""` literal type.
- **Performance** — katta union'lar bilan template literal cartesian product yaratadi, TS checker'ga og'ir.

### Follow-up savollar

1. **"`${infer N extends number}` qanday ishlaydi?"** — TS template literal'dan substring olib, `number` constraint qo'llaydi. Constraint pass bo'lsa, N number literal sifatida saqlanadi.
2. **"Template literal types runtime'ga ta'sir qiladimi?"** — Yo'q, faqat compile-time. JS'ga compile'da type information o'chiriladi.

</details>

---

### Savol 6: Variadic tuple types nima? Spread tuple operations qanday? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Variadic tuple types (TS 4.0+) — tuple'da `...` spread. Tuple'larni birlashtirish, ajratish, transform qilish. Yo'naltirilgan operations: `Concat`, `Head`, `Tail`, `Push`, `Reverse`.

### To'liq tushuntirish

Variadic tuple — generic tuple'da rest element ishlatish:

- `[...T, ...U]` — birlashtirish
- `[infer H, ...infer Rest]` — destructuring
- `[...infer Init, infer Last]` — oxirgi element ajratish
- `[A, ...B, C]` — TS 4.2+ middle rest

### Kod misol

```typescript
// Tuple operations
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];
type A = Concat<[1, 2], [3, 4]>;        // [1, 2, 3, 4]

type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T extends unknown[]> = T extends [unknown, ...infer R] ? R : [];
type Last<T extends unknown[]> = T extends [...unknown[], infer L] ? L : never;
type Init<T extends unknown[]> = T extends [...infer I, unknown] ? I : [];

type B = Head<[1, 2, 3]>;               // 1
type C = Tail<[1, 2, 3]>;               // [2, 3]
type D = Last<[1, 2, 3]>;               // 3
type E = Init<[1, 2, 3]>;               // [1, 2]

// Push/Unshift
type Push<T extends unknown[], V> = [...T, V];
type Unshift<T extends unknown[], V> = [V, ...T];

type F = Push<[1, 2], 3>;               // [1, 2, 3]
type G = Unshift<[2, 3], 1>;            // [1, 2, 3]

// Reverse (recursive)
type Reverse<T extends unknown[]> =
  T extends [infer H, ...infer Rest]
    ? [...Reverse<Rest>, H]
    : [];
type H = Reverse<[1, 2, 3]>;            // [3, 2, 1]

// Tail-recursive Reverse (TS 4.5+) — accumulator
type ReverseTail<T extends unknown[], Acc extends unknown[] = []> =
  T extends [infer Head, ...infer Rest]
    ? ReverseTail<Rest, [Head, ...Acc]>
    : Acc;
type I = ReverseTail<[1, 2, 3]>;        // [3, 2, 1]

// Function call type-safe
function call<T extends unknown[], R>(
  fn: (...args: T) => R,
  ...args: T
): R {
  return fn(...args);
}

function greet(name: string, age: number): string {
  return `${name} is ${age}`;
}
call(greet, "Ali", 25);                  // ✅
// call(greet, "Ali");                    // ❌ Missing age
```

### Edge Cases

- **Tail-call evaluation (TS 4.5+)** — accumulator pattern bilan deeply recursive type'lar. Non-tail recursion depth limit'i past (bir necha o'nlab step), TS 4.5+ tail-recursive shape uchun hard limit `1000` (release notes).
- **Middle rest** — `[A, ...B[], C]` — `[A]` + array + `[C]` (TS 4.0+ variadic tuple). Labeled middle rest (`[a: A, ...b: B[], c: C]`) — TS 4.0+ labeled tuple.
- **Optional rest tuple** — `[A, B?]` — B optional. `[A, ...B[]]` — B array spread.
- **Tuple labels** — `[id: number, name: string]` — labels documentation, runtime'da yo'q.
- **Spread'lar tartibi** — `[...T, ...U, A, ...V]` — TS infer'da nazarda tutilgan tartib.

### Follow-up savollar

1. **"Tail recursion qanday optimize qilinadi?"** — TS 4.5+ checker recursive conditional type'ning oxirgi expression yana o'sha conditional bo'lsa (tail position), uni instantiation stack o'sishisiz iterativ tarzda hisoblaydi.
2. **"Variadic tuple va `arguments` farqi?"** — `arguments` — runtime old-style array-like. Variadic tuple — type-level spread, kerakli runtime — rest parameter (`...args`).

<details>
<summary><strong>Deep Dive</strong></summary>

**Variadic tuple inference:**

Tuple type bir nechta fixed element va eng ko'pi bilan bitta variadic (`...T`) element'dan iborat. `[A, ...B, C]` shaklida spread'dan element type infer qilishda checker quyidagicha ishlaydi:

1. Fixed prefix uzunligi aniqlanadi (`A`)
2. Fixed suffix uzunligi aniqlanadi (`C`)
3. Qolgan o'rta qism variadic element'ga (`B`) biriktiriladi

**Tail-call evaluation (TS 4.5+):**

Recursive conditional type'ning oxirgi expression (`Acc` pattern bilan) yana o'sha conditional bo'lsa, checker uni instantiation stack o'rniga iterativ tarzda hisoblaydi. Bu deep recursion'ga ruxsat beradi.

**Limit'lar:**

- Non-tail-recursive: har step yangi nested instantiation hosil qiladi, depth limit'iga tez urishadi (bir necha o'nlab element'dan keyin)
- Tail-recursive: hard limit 1000 (TS 4.5 release notes)
- Type instantiation count limit `5,000,000` — checker bu chegaradan oshganda `Type instantiation is excessively deep` (TS2589) xato beradi

</details>

</details>

---

### Savol 7: `NoInfer` (TS 5.4) nima? Qachon kerak? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`NoInfer<T>` (TS 5.4+) — type parameter ikki joyda ishlatilganda inference'ni bir tomondan bloklash. Caller'ga aniq type'ni argument'dan emas, contextual yoki explicit'dan olishni majbur qiladi.

### To'liq tushuntirish

Muammo: generic T ikki argument'da ishlatilsa, TS ikkalasidan ham inference qiladi va union'ga aylantirishi mumkin (yoki noto'g'ri narrowing).

Yechim: `NoInfer<T>` — TS bu pozitsiyadan T'ni infer qilmaydi, faqat boshqa pozitsiya'lardan oladi.

### Kod misol

```typescript
// Muammo
function setValue<T>(value: T, defaultValue: T): T {
  return value ?? defaultValue;
}

setValue("hello", "default"); // T = string — OK
setValue("hello", 42);          // T = string | number — kutilmagan inference

// Yechim: NoInfer
function setValueSafe<T>(value: T, defaultValue: NoInfer<T>): T {
  return value ?? defaultValue;
}

setValueSafe("hello", "default"); // T = string ✅
// setValueSafe("hello", 42);       // ❌ Type 'number' not assignable to 'string'

// Real-world: State machine
type StateMachine<States extends string> = {
  initialState: States;
  transitions: Record<States, States[]>;
};

// States transitions key'laridan infer qilinadi (barcha state'lar shu yerda).
// initialState NoInfer bilan — u inference manbai emas, shu union'ga tekshiriladi.
function createMachine<States extends string>(
  transitions: Record<States, States[]>,
  initialState: NoInfer<States>
): StateMachine<States> {
  return { initialState, transitions };
}

const machine = createMachine(
  {
    idle: ["running"],
    running: ["idle", "paused"],
    paused: ["running"],
  },
  "idle"
);
// States = "idle" | "running" | "paused" — transitions key'laridan
// initialState "idle" shu union'ga tekshiriladi (NoInfer)
// createMachine({ ... }, "stopped"); // ❌ "stopped" assignable emas

// Reducer pattern
function reducer<S, A extends { type: string }>(
  initialState: S,
  handlers: Record<A["type"], (state: NoInfer<S>, action: A) => S>
) {
  // ...
}
```

### Edge Cases

- **`NoInfer` faqat TS 5.4+** — undan oldingi versions'da workaround: helper type ishlatish.
- **`NoInfer` nested type'da** — `NoInfer<T[]>` — array element type'idan inference o'chiriladi.
- **Multiple `NoInfer`** — har joyda alohida ishlaydi, ikkalasi ham inference'siz.
- **`NoInfer` + constraint** — constraint hali ham qo'llanadi, faqat inference o'chiriladi.

### Follow-up savollar

1. **"`NoInfer` polyfill TS 5.4'gacha qanday yozilgan?"** — eng keng tarqalgani intersection trick (`T & {}`): TS bu position'dan to'g'ridan-to'g'ri inference qilmaydi. `[T] extends [infer U] ? U : never` kabi variantlar ishonchsiz — TS ko'pincha bu yerdan ham inference oladi. Native `NoInfer` esa hech bir edge case'da inference o'tkazmaydi.
2. **"`NoInfer` har generic'da ishlatish kerakmi?"** — Yo'q, faqat inference asymmetric kerak bo'lganda (caller bir argument'dan T'ni olishni xohlasa, boshqasi bog'liq).

<details>
<summary><strong>Deep Dive</strong></summary>

**`NoInfer` implementation — TS 5.4+:**

`NoInfer<T>` checker'da maxsus substitution type sifatida ifodalanadi (`TypeFlags.Substitution`, constraint `unknown` bilan — bu kombinatsiya boshqa hech qayerda uchramaydi). Inference paytida checker bu marker bilan belgilangan pozitsiyani inference candidate'lari uchun ko'rib chiqmaydi. Boshqa barcha kontekstda `T` va `NoInfer<T>` bir xil type. PR #56794 (Anders Hejlsberg) yangi type kind kiritmaslik uchun ataylab mavjud substitution type infrastructure'ini qayta ishlatadi — shu sababli `NoInfer` tooling ecosystem'ni buzmaydi va marker erasure mexanizmidan foydalanadi.

**Pre-5.4 workaround'lar — limitations:**

```typescript
type NoInferLegacy<T> = T & {};
// TS bu intersection position'idan to'g'ridan-to'g'ri inference qilmaydi
```

Intersection trick (`T & {}`) inference'ni qisman bloklaydi, lekin subtype relation'ga ta'sir qilishi mumkin (masalan primitive'larga `& {}` qo'shilishi). Modern TS 5.4+ — native `NoInfer` afzal, chunki u type'ni o'zgartirmasdan faqat inference'ni bloklaydi.

**Asymmetric inference pattern'lari:**

| Pattern | Misol | Foyda |
|---------|-------|-------|
| Anchor + constraint | `<T>(value: T, fallback: NoInfer<T>)` | Caller value'dan T olinadi, fallback uni cheklaydi |
| Discriminator | `<T>(items: T[], key: NoInfer<keyof T>)` | items'dan T inferred, key validated |
| State machine | `<S>(transitions: Record<S, S[]>, initial: NoInfer<S>)` | transitions key'laridan states, initial cross-check |

**Performance ta'siri:**

`NoInfer` checker time'ga minimal ta'sir — inference candidate filter'da bitta flag check. Lekin recursive type'larda har step'da check bo'ladi, juda murakkab generic'larda kichik overhead.

</details>

</details>

---

### Savol 8: Deferred conditional type — nima va qanday tuzatiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Deferred conditional — generic function body ichida T aniq emas, shuning uchun `T extends X ? Y : Z` resolve bo'lmaydi. Body'da `as` assertion yoki overload bilan tuzatiladi.

### To'liq tushuntirish

TS generic body'da T abstract — har possible type emas, faqat constraint'ga mos. Conditional type generic'da deferred holatda qoladi, body ichida runtime narrowing (`typeof`) TS-level conditional'ga ta'sir qilmaydi.

### Kod misol

```typescript
// ❌ Deferred conditional
function example<T extends string | number>(
  value: T
): T extends string ? string[] : number {
  if (typeof value === "string") {
    return value.split(","); // ❌ Type 'string[]' not assignable to 'T extends string ? string[] : number'
  }
  return 0; // ❌ Type 'number' not assignable to 'T extends string ? string[] : number'
}

// ✅ Yechim 1: Function overloads
function example1(value: string): string[];
function example1(value: number): number;
function example1(value: string | number): string[] | number {
  if (typeof value === "string") return value.split(",");
  return 0;
}

const a = example1("a,b");  // string[]
const b = example1(42);      // number

// ✅ Yechim 2: as assertion body'da
function example2<T extends string | number>(
  value: T
): T extends string ? string[] : number {
  type Result = T extends string ? string[] : number;
  if (typeof value === "string") {
    return value.split(",") as Result;
  }
  return 0 as Result;
}

// ✅ Yechim 3: Helper functions (clean implementation)
function parseString(s: string): string[] { return s.split(","); }
function parseNumber(n: number): number { return n; }

function example3<T extends string | number>(
  value: T
): T extends string ? string[] : number {
  return (typeof value === "string"
    ? parseString(value)
    : parseNumber(value as number)) as T extends string ? string[] : number;
}
```

### Edge Cases

- **Distributive deferred** — `T extends U ? X : Y` T union bo'lsa, har member alohida deferred bo'ladi.
- **Constraint narrowing body'da** — TS 4.7+ ba'zi narrowing'lar generic body'da T'ga ta'sir qiladi (qisman). Lekin conditional return type uchun yetarli emas.
- **Conditional return va inference** — caller side'da conditional resolve bo'ladi. Faqat body ichida deferred.
- **Generic'siz conditional** — non-generic conditional darhol resolve bo'ladi (deferred bo'lmaydi).

### Follow-up savollar

1. **"Deferred conditional TS bug emasmi?"** — Yo'q, intentional. Generic T ning haqiqiy qiymati call site'da aniq, body'da abstract. Strict checking sound type system uchun zarur.
2. **"Overload vs `as` assertion — qaysi biri afzal?"** — Overload — clean implementation, caller side aniq. `as` assertion — generic flexibility saqlaydi, lekin runtime safety dasturchining mas'uliyatida.

<details>
<summary><strong>Deep Dive</strong></summary>

**Why deferred:**

TS conditional type `T extends U ? X : Y` resolution `getConditionalType` da. Agar `T` yoki `U`'da hali substitute bo'lmagan generic type parameter mavjud bo'lsa, checker conditional'ni resolve qila olmaydi va uni deferred conditional type sifatida saqlaydi — type argument berilganda qaytadan hisoblanadi.

**Body narrowing limitation:**

JS-level `typeof`, `instanceof`, type predicate'lar — `value` parameter type'ini narrow qiladi (control flow analysis). Lekin generic `T` parameter'ni narrow qila olmaydi (T abstract — har call'da turli value bo'lishi mumkin).

**Workaround:**

Generic `T`'ni body ichida conditional return type uchun narrow qilishning to'g'ridan-to'g'ri yo'li hozircha yo'q. Amaldagi yechimlar — overload (caller side aniqlik) yoki body'da `as` assertion (dasturchining mas'uliyati).

</details>

</details>

---

### Savol 9: Recursive types — depth limit va tail-call elimination [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Recursive type — type o'zini o'zi reference qiladi. TS instantiation depth limit'i bor (`Type instantiation is excessively deep` xato). TS 4.5+ tail-call elimination — tail-recursive shape (accumulator pattern) kattaroq depth'da ishlay oladi.

### To'liq tushuntirish

Recursive type ikki shaklda:

1. **Non-tail-recursive** — TS har step'da yangi instantiation, depth limit oson urishi mumkin
2. **Tail-recursive** — accumulator parameter, oxirgi expression — recursive call, TS optimize qiladi

### Kod misol

```typescript
// Non-tail-recursive — Reverse
type Reverse<T extends unknown[]> =
  T extends [infer H, ...infer Rest]
    ? [...Reverse<Rest>, H]      // recursive call ichkarida — non-tail
    : [];

type A = Reverse<[1, 2, 3, 4, 5]>; // [5, 4, 3, 2, 1] ✅
// Reverse<[/* 100 element */]> — depth limit'ga urishi mumkin

// Tail-recursive — accumulator pattern
type ReverseTail<T extends unknown[], Acc extends unknown[] = []> =
  T extends [infer Head, ...infer Rest]
    ? ReverseTail<Rest, [Head, ...Acc]>   // recursive — oxirgi expression
    : Acc;

type B = ReverseTail<[1, 2, 3, 4, 5]>; // [5, 4, 3, 2, 1] ✅
// Kattaroq tuple'lar uchun ishlay oladi (TS 4.5+)

// JSON type — recursive union
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const data: JSONValue = {
  users: [
    { name: "Ali", age: 25, active: true, meta: null }
  ]
};

// Deep types — counter bilan depth limit
type DeepFlatten<T, D extends unknown[] = []> =
  D["length"] extends 10
    ? T                                    // 10 daraja limit
    : T extends (infer U)[]
      ? DeepFlatten<U, [...D, unknown]>
      : T;

type C = DeepFlatten<number[][][][]>; // number

// DeepReadonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]
    : T[K] extends object
      ? DeepReadonly<T[K]>
      : T[K];
};
```

### Edge Cases

- **`Type instantiation is excessively deep` xato** — TS instantiation budget tugaganda. Yechim: tail-rec, counter, simpler recursion.
- **`Type alias references itself` xato** — direct circular reference (`type A = A`). Indirect (interface, conditional bilan) ruxsat.
- **Recursive interface** — interface'lar default'da recursive ruxsat (`interface Tree { children: Tree[] }`).
- **Mutual recursion** — `type A = { b: B }`, `type B = { a: A }` — ruxsat, lekin TS forward declaration tartibida.
- **Recursive utility'lar performance** — checker time'ga ta'sir. Katta loyihalarda recursive `DeepPartial` kabi utility'lar build sekinlashtirishi mumkin.

### Follow-up savollar

1. **"Tail-call elimination TS'da JavaScript'dagi kabi optimization?"** — Yo'q, TS'da type system level optimization. Runtime'ga ta'sir yo'q. JS engine'larida tail-call elimination Safari'dan boshqa joyda implement qilinmagan.
2. **"`infinitely deep` xato'ni qanday debug qilamiz?"** — Counter bilan depth cheklash, simpler conditional, mutually recursive type'larni alohida ajratish. TS playground'da reproduce qilib step-by-step.

</details>

---

### Savol 10: Higher-Kinded Types (HKT) — TS'da qo'llab-quvvatlanadimi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

HKT — type parameter o'zi ham generic bo'lishi ("generic'ning generic'i"). TS to'g'ridan-to'g'ri qo'llab-quvvatlamaydi. Workaround pattern'lar: overload, URI-based encoding (fp-ts), conditional dispatch.

### To'liq tushuntirish

HKT misol — Haskell `Functor`:

```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

`f` o'zi generic — `Array`, `Maybe`, `Promise` kabi container'lar. TS'da `<F<_>>` syntax mavjud emas.

Workaround'lar:

1. **Overload** — har container uchun alohida
2. **URI pattern** (fp-ts) — string literal'ga map qilish
3. **Conditional dispatch** — type-level switch

### Kod misol

```typescript
// 1. Overload pattern
function mapValues<T, U>(container: T[], fn: (v: T) => U): U[];
function mapValues<T, U>(container: Promise<T>, fn: (v: T) => U): Promise<U>;
function mapValues<T, U>(
  container: T[] | Promise<T>,
  fn: (v: T) => U
): U[] | Promise<U> {
  if (Array.isArray(container)) return container.map(fn);
  return container.then(fn);
}

// 2. URI pattern (fp-ts style)
interface URIToKind<A> {
  Array: A[];
  Promise: Promise<A>;
  Maybe: A | null;
}

type URIS = keyof URIToKind<any>;
type Kind<URI extends URIS, A> = URIToKind<A>[URI];

interface Functor<F extends URIS> {
  map<A, B>(fa: Kind<F, A>, fn: (a: A) => B): Kind<F, B>;
}

const ArrayFunctor: Functor<"Array"> = {
  map: (fa, fn) => fa.map(fn),
};

const PromiseFunctor: Functor<"Promise"> = {
  map: (fa, fn) => fa.then(fn),
};

// 3. Conditional dispatch
type Container<T> = { type: "array"; value: T[] } | { type: "promise"; value: Promise<T> };

type MapResult<C, U> = C extends { type: "array" }
  ? { type: "array"; value: U[] }
  : C extends { type: "promise" }
    ? { type: "promise"; value: Promise<U> }
    : never;
```

### Edge Cases

- **TS open issue #1213** — HKT support'ga proposal, 2014'dan ochiq. Concrete plan yo'q.
- **fp-ts library** — URI pattern bilan to'liq FP ecosystem TS'da. Trade-off — explicit URI registration kerak.
- **Effect-TS** — yangi avlod, URI pattern + variance encoding'ni yaxshilangan.
- **Type-level computation kuchli** — TS conditional + mapped + template literal types HKT'siz ham ko'p narsani implement qila oladi.

### Follow-up savollar

1. **"TS HKT support'ni qachon qo'shadi?"** — Hozirgi roadmap'da yo'q. Performance va checker complexity tashvishlari. Amalda overload va URI pattern yetarli.
2. **"HKT'siz Monad pattern qanday yoziladi?"** — URI pattern + interface implementation. fp-ts shu yo'l bilan to'liq monadic API beradi.

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 11: Output — Flatten conditional [Middle+]

**Savol:** Har type'ning natijasini ayting:

```typescript
type Flatten<T> = T extends (infer U)[] ? U : T;

type A = Flatten<string[]>;
type B = Flatten<number>;
type C = Flatten<(string | number)[]>;
type D = Flatten<string[][]>;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type A = string;
type B = number;
type C = string | number;
type D = string[];
```

### To'liq tushuntirish

- `Flatten<string[]>` → `string[]` mos `(infer U)[]` → `U = string`
- `Flatten<number>` → mos emas → `T = number`
- `Flatten<(string | number)[]>` → mos → `U = string | number`
- `Flatten<string[][]>` → mos → `U = string[]` (faqat bitta daraja)

Deep flatten kerak bo'lsa recursive:

```typescript
type DeepFlatten<T> = T extends (infer U)[] ? DeepFlatten<U> : T;
type E = DeepFlatten<string[][][]>; // string
```

### Edge Cases

- **Tuple bilan** — `Flatten<[string, number]>` mos keladi (tuple — array sub-type), `U = string | number`.
- **Empty array** — `Flatten<[]>` — `U = never`.
- **Readonly array** — `Flatten<readonly string[]>` — `(infer U)[]` mos emas (readonly farq). `(infer U)[] | readonly (infer U)[]` kerak.

### Follow-up savollar

1. **"`Flatten<[1, "a", true]>` — natija?"** — `1 | "a" | true`. Tuple `(infer U)[]` pattern'iga mos keladi va `U` element type'larining union'i sifatida infer qilinadi (tuple'da `T[number]` = `1 | "a" | true`).

</details>

---

### Savol 12: Output — Distributive GetReturnType [Middle+]

**Savol:** Har type'ning natijasini ayting. E nima uchun shunday?

```typescript
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = GetReturnType<typeof Math.random>;
type B = GetReturnType<typeof parseInt>;
type C = GetReturnType<string>;
type D = GetReturnType<() => void>;
type E = GetReturnType<(() => string) | (() => number)>;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type A = number;
type B = number;
type C = never;
type D = void;
type E = string | number;
```

### To'liq tushuntirish

- `Math.random: () => number` → `R = number`
- `parseInt: (s: string, r?: number) => number` → `R = number`
- `string` — funksiya emas → false branch → `never`
- `() => void` → `R = void`
- E — distributive: T naked parameter, union T = `(() => string) | (() => number)`:
  ```
  GetReturnType<() => string> | GetReturnType<() => number>
  = string | number
  ```

### Edge Cases

- **Overloaded function** — `GetReturnType<typeof setTimeout>` — TS oxirgi overload'ni oladi (`ReturnType` utility ham shunday).
- **Generic function** — `<T>(x: T) => T` — `GetReturnType` = `unknown` (generic resolved without type argument).
- **Constructor signature** — `GetReturnType<typeof Date>` = `string` (`Date()` call signature). Constructor uchun `InstanceType`.

### Follow-up savollar

1. **"Distributive'ni qanday o'chiramiz?"** — `[T] extends [(...args: any[]) => infer R] ? R : never` — wrapper bilan. E natijasi `string | number` qoladi (chunki R covariant position'da union).

</details>

---

### Savol 13: Output — `never` tricky behavior [Senior]

**Savol:** Har type'ning natijasini ayting va tushuntiring:

```typescript
type A = never extends string ? true : false;
type B<T> = T extends string ? true : false;
type C = B<never>;
type D = [never] extends [string] ? true : false;
type E<T> = [T] extends [string] ? true : false;
type F = E<never>;
type G<T> = T extends any ? T[] : never;
type H = G<never>;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type A = true;    // never bottom type, har type'ga assignable
type C = never;   // distributive: never = bo'sh union → 0 natija
type D = true;    // [never] non-distributive, never assignable
type F = true;    // E<never> — wrapper, non-distributive
type H = never;   // G<never> distributive ham, lekin never = bo'sh union
```

### To'liq tushuntirish

| Type | Generic | Distributive | Natija |
|------|---------|--------------|--------|
| A | Yo'q (direct) | — | `true` |
| C (B\<never\>) | Ha (naked T) | Ha (never = bo'sh union) | `never` |
| D | Yo'q (direct) | — | `true` |
| F (E\<never\>) | Ha lekin `[T]` | Yo'q (wrapper) | `true` |
| H (G\<never\>) | Ha (naked T) | Ha (never = bo'sh union) | `never` |

**Kalit qoida:**
- Direct (generic'siz) — `never` bottom type, har type'ga assignable
- Generic naked T — `never` = bo'sh union → distributive → 0 natija → `never`
- Generic `[T]` wrapper — non-distributive → oddiy check

### Edge Cases

- **`any` parametr'da** — `T extends any ? T[] : never` — `any` har type'ga distribute bo'ladi. `G<string | number>` = `string[] | number[]`.
- **`unknown`** — `Wrapped<unknown>` — `[unknown] extends [string]` → `false` (unknown faqat unknown'ga assignable).
- **`never` mapped'da** — `{ [K in never]: V }` = `{}` (bo'sh object).

### Follow-up savollar

1. **"IsNever utility ni qanday yozamiz?"** — `type IsNever<T> = [T] extends [never] ? true : false`. Wrapper bilan distributive'ni o'chiramiz.

</details>

---

### Savol 14: Coding — Variadic tuple operations [Senior]

**Savol:** Tuple utility type'larini implement qiling: `Head`, `Last`, `Init`, `Reverse` (tail-recursive).

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Last<T extends unknown[]> = T extends [...unknown[], infer L] ? L : never;
type Init<T extends unknown[]> = T extends [...infer I, unknown] ? I : [];
type Reverse<T extends unknown[], Acc extends unknown[] = []> =
  T extends [infer Head, ...infer Rest]
    ? Reverse<Rest, [Head, ...Acc]>
    : Acc;
```

### To'liq tushuntirish

```typescript
type H1 = Head<[string, number, boolean]>;   // string
type H2 = Head<[]>;                          // never

type L1 = Last<[string, number, boolean]>;   // boolean
type L2 = Last<[42]>;                        // 42

type I1 = Init<[string, number, boolean]>;   // [string, number]
type I2 = Init<[42]>;                        // []

type R1 = Reverse<[1, 2, 3]>;                // [3, 2, 1]
type R2 = Reverse<[string, number]>;         // [number, string]
type R3 = Reverse<[]>;                       // []
```

- `infer H` + `...unknown[]` — tuple destructuring type-level
- `[...Acc, ...]` accumulator — tail-recursive optimization (TS 4.5+)
- Empty tuple base case — `[]` qaytaradi

### Edge Cases

- **Readonly tuple** — `Reverse<readonly [1, 2, 3]>` — `readonly` modifier yo'qoladi. `Reverse<T extends readonly unknown[]>` shaklida saqlash mumkin.
- **Optional tuple element** — `[string, number?]` — `Head` `string`, `Last` `number | undefined`.
- **Rest tuple** — `[string, ...number[]]` — `Head` `string`, `Last` `number`, lekin `Init` problematic (variable length).

### Follow-up savollar

1. **"Non-tail-recursive Reverse vs tail-recursive — performance farqi?"** — Non-tail TS instantiation stack o'sadi (depth limit'ga urishi mumkin). Tail-recursive iterative evaluation, kattaroq tuple'larga ruxsat.
2. **"`Reverse` typed runtime function bilan?"** — `function reverse<T extends unknown[]>(arr: [...T]): Reverse<T> { return [...arr].reverse() as Reverse<T> }` — `as` assertion deferred conditional uchun.

</details>

---

### Savol 15: Coding — Type-safe FormBuilder [Senior]

**Savol:** `FormBuilder` yozing — `addField` chaqirilganda yangi field type'ga qo'shilsin, `build()` barcha field'larning type'ini qaytarsin:

```typescript
// const form = new FormBuilder()
//   .addField("name", "")       → { name: string }
//   .addField("age", 0)         → { name: string; age: number }
//   .addField("active", true)   → { name: string; age: number; active: boolean }
//   .build();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
class FormBuilder<Fields extends Record<string, unknown> = {}> {
  private fields: Map<string, unknown> = new Map();

  addField<Name extends string, Type>(
    name: Name,
    defaultValue: Type
  ): FormBuilder<Fields & Record<Name, Type>> {
    this.fields.set(name, defaultValue);
    return this as unknown as FormBuilder<Fields & Record<Name, Type>>;
  }

  build(): Fields {
    return Object.fromEntries(this.fields) as Fields;
  }
}
```

### To'liq tushuntirish

- `Fields extends Record<string, unknown> = {}` — accumulator, default empty
- `addField<Name, Type>` — har chaqiruvda yangi parametr'lar
- Return: `FormBuilder<Fields & Record<Name, Type>>` — intersection bilan birlashtirish
- `as unknown as` — runtime instance type o'zgartirib bo'lmaydi, type-level assertion kerak

```typescript
const form = new FormBuilder()
  .addField("name", "")
  .addField("age", 0)
  .addField("active", true)
  .build();

form.name;     // string ✅
form.age;      // number ✅
form.active;   // boolean ✅
// form.email;   // ❌ Property 'email' does not exist
```

### Edge Cases

- **Duplicate field** — `addField("name", "").addField("name", 42)` — intersection `{ name: string } & { name: number }` = `{ name: never }`. Runtime'da oxirgi yutadi.
- **Empty build** — `new FormBuilder().build()` — `{}` (bo'sh object).
- **Type inference fail** — caller `.addField("x" as string, "")` — Name = `string` (literal yo'q), `Record<string, string>` ko'p key'lar uchun.
- **`<const T>` qo'shish** — `addField<const Name extends string, Type>` — caller `as const` yozmasa ham literal Name infer.

### Follow-up savollar

1. **"Validator qo'shish — type-safe qanday?"** — Generic'ga `Validators extends Record<keyof Fields, (v: Fields[K]) => boolean>` qo'shish. Builder method `addValidator<K extends keyof Fields>(key: K, validator: (v: Fields[K]) => boolean)`.
2. **"Bu pattern qaysi library'larda ishlatiladi?"** — Zod (schema builder), Drizzle ORM (query builder), tRPC (router builder), Hono (route builder).

<details>
<summary><strong>Deep Dive</strong></summary>

**Accumulator pattern type-level:**

Type-level "fold" / "reduce" — har step'da accumulator yangi state bilan yangilanadi. JS reduce equivalent type-level:

```typescript
// Type-level reduce
type Reduce<T extends unknown[], Acc, F extends { Acc: unknown; Item: unknown; Result: unknown }> =
  T extends [infer H, ...infer Rest]
    ? Reduce<Rest, (F & { Acc: Acc; Item: H })["Result"], F>
    : Acc;
```

**Variance and intersection:**

`Fields & Record<Name, Type>` — intersection. Method chain'da Fields covariantly o'sadi. Builder return type'da covariance saqlanadi.

**Runtime instance vs type:**

`this as unknown as FormBuilder<NewFields>` — runtime'da bir xil instance, type-level "morph". JS prototype chain o'zgarmaydi. Bu pattern monkey-patching emas, faqat type narrowing.

**Limitations:**

- `addField` qaytargan instance bir xil (mutable internal state)
- Immutable variant — `addField` new instance qaytaradi (memory cost)
- Method order matters for inference (chain order = type accumulation order)

</details>

</details>

---

### Savol 16: Bug — Deferred conditional fix [Middle+]

**Savol:** Bu kod compile xato beradi. Tuzating:

```typescript
function transform<T extends "json" | "text">(
  raw: string,
  mode: T
): T extends "json" ? object : string {
  if (mode === "json") {
    return JSON.parse(raw);
  }
  return raw;
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic body'da `T extends "json" ? object : string` deferred — TS T'ning aniq qiymatini bilmaydi. `as` assertion yoki overload bilan tuzatish.

### To'liq tushuntirish

```typescript
// Yechim 1: as assertion
function transform<T extends "json" | "text">(
  raw: string,
  mode: T
): T extends "json" ? object : string {
  type Result = T extends "json" ? object : string;
  if (mode === "json") {
    return JSON.parse(raw) as Result;
  }
  return raw as Result;
}

// Yechim 2: Overload (clean implementation)
function transformOverload(raw: string, mode: "json"): object;
function transformOverload(raw: string, mode: "text"): string;
function transformOverload(raw: string, mode: "json" | "text"): object | string {
  if (mode === "json") return JSON.parse(raw);
  return raw;
}

const a = transformOverload("{}", "json");   // object
const b = transformOverload("hello", "text"); // string
```

### Edge Cases

- **`unknown` return** — caller mode `"json" | "text"` (literal narrowing yo'q) — return `object | string`.
- **Runtime check va TS check** — `if (mode === "json")` — TS `mode` narrow qiladi, lekin conditional return type T'ga bog'liq, narrow bo'lmaydi.

### Follow-up savollar

1. **"Distributive conditional'da bir xil muammo paydo bo'ladimi?"** — Ha, generic body'da har conditional deferred. Caller side'da resolve bo'ladi.

</details>

---

## Xulosa

- Conditional type — type-level if/else, naked T + union → distributive
- `infer` — pattern matching, conditional'ning `extends` qismida. Multiple infer covariant union / contravariant intersection
- `never` distributive da bo'sh union → `never`. `[T]` wrapper non-distributive
- Mapped type — `{ [K in K]: V }`, modifier'lar `readonly`/`-readonly`, `?`/`-?`
- Template literal types — string literal'lardan compile-time strings
- Variadic tuple — `[...T, ...U]`, destructuring, transformation
- `NoInfer` (TS 5.4) — type parameter inference'ni asymmetric qilish
- Deferred conditional — generic body'da resolve bo'lmaydi, `as` yoki overload
- Recursive types — tail-call elimination (TS 4.5+), counter bilan depth control
- HKT — TS direct support'lamaydi, URI pattern (fp-ts) yoki overload

---