# Interview: Conditional Types

> Conditional type syntax, `infer` keyword, distributive behavior, recursive conditional types, deferred conditional, covariant/contravariant infer, `infer extends`, function overload nuance, recursion limit, real-world patterns bo'yicha interview savollari. Har javob mustaqil — kontekst javob ichida.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar) — 8 ta
- [Amaliy savollar](#amaliy-savollar-coding-challenges) — 7 ta
- [Bug fix savollar](#bug-fix-savollar) — 2 ta
- [Xulosa](#xulosa)

---

## Nazariy savollar

### Savol 1: Conditional type nima? `extends` bu yerda nima tekshiradi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Conditional type — type level'da `if/else`. Sintaksis: `T extends U ? TrueType : FalseType`. `extends` bu yerda inheritance emas — **assignability check** ("T ni U ga assign qilish mumkinmi?").

### To'liq tushuntirish

Conditional type asoschisi: type-level branching. Compile-time'da type checker `T extends U` ni evaluate qiladi:
- True branch — `T` `U` ga assignable bo'lsa
- False branch — aks holda
- Deferred — `T` hali generic bo'lsa (resolve qilinmasdan saqlanadi)

Assignability rules:
- Primitive: `"hello" extends string` → true
- Object: structural subtyping (ko'proq property = subtype)
- Function: bivariant param + covariant return (default), strict'da contravariant param

Nested conditional — chained `if/else if/else`:

### Kod misol

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">;       // true — "hello" string ga assignable
type B = IsString<number>;        // false

// Object — structural subtyping
type HasName<T> = T extends { name: string } ? true : false;
type C = HasName<{ name: string; age: number }>; // true — extra property OK
type D = HasName<{ age: number }>;               // false — name yo'q

// Nested conditional — multi-branch
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  T extends Function ? "function" :
  T extends undefined ? "undefined" :
  T extends null ? "null" :
  "object";

type E = TypeName<"hi">;        // "string"
type F = TypeName<42>;          // "number"
type G = TypeName<() => void>;  // "function"
type H = TypeName<{ x: 1 }>;    // "object"

// Real use case — discriminated narrowing
type EventPayload<E extends string> =
  E extends "click" ? { x: number; y: number } :
  E extends "submit" ? { formData: FormData } :
  E extends "load" ? { url: string } :
  never;

function handleEvent<E extends string>(event: E, payload: EventPayload<E>): void {
  /* type-safe payload */
}

handleEvent("click", { x: 10, y: 20 });    // ✅
// handleEvent("click", { formData: ... }); // ❌
```

### Edge Cases

- **`any` extends har narsa:** `any extends T` har doim ham true ham false branch'ni qaytaradi (union)
- **`never` extends:** distributive pozitsiyada `never` — bo'sh union (0 member), 0 marta distribute — natija `never`
- **Conditional in mapped type:** mapped type ichida conditional — `{ [K in keyof T]: T[K] extends Function ? ... : ... }`
- **Recursive conditional:** conditional o'z ichida o'zini referencing qila oladi (TS 4.1+)
- **Function param variance:** `strictFunctionTypes` yoqilgan paytda parameter contravariant — `(x: string) => void` `(x: string | number) => void` ga assignable emas. `strictFunctionTypes` o'chiq bo'lsa — bivariant (ikkala yo'nalishda assignable)

### Follow-up savollar

1. "`A extends B` va `A | B` farqi?" — `extends` — assignability check. `A | B` — union type (alternativa)
2. "Conditional type performance ta'siri?" — Murakkab nested conditional'lar compile time'ni sezilarli sekinlashtiradi

</details>

---

### Savol 2: `infer` keyword nima qiladi? Qaysi pozitsiyalarda ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`infer` — conditional type ichida type'ni **extract** qilish (pattern matching). Faqat conditional `extends` clause'ning right side'da ishlatiladi. Type checker pattern match qiladi va variable'ga bind qiladi.

### To'liq tushuntirish

`infer X` — "X type'ni shu pozitsiyadan oling". TS pattern matching algoritm orqali type'ni resolve qiladi va `X` ni true branch'da ishlatish mumkin.

Standard pattern'lar:

| Pattern | Use case |
|---------|----------|
| `(...args: infer P) => any` | Funksiya parameter tuple |
| `(...args: any) => infer R` | Funksiya return type |
| `(infer U)[]` | Array element type |
| `Promise<infer U>` | Promise wrapped type |
| `[infer H, ...any[]]` | Tuple birinchi element |
| `[...any[], infer L]` | Tuple oxirgi element |
| `Record<infer K, any>` | Index key type |

### Kod misol

```typescript
// Return type
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = MyReturnType<() => string>;       // string
type B = MyReturnType<(x: number) => boolean>; // boolean
type C = MyReturnType<string>;             // never (not a function)

// Parameter tuple
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;

type D = MyParameters<(a: string, b: number) => void>;
// [a: string, b: number]

// Array element
type ElementOf<T> = T extends (infer U)[] ? U : never;
type E = ElementOf<number[]>;              // number
type F = ElementOf<(string | boolean)[]>;  // string | boolean

// Promise unwrap (recursive)
type DeepAwaited<T> = T extends Promise<infer U> ? DeepAwaited<U> : T;
type G = DeepAwaited<Promise<Promise<string>>>; // string

// Tuple
type First<T> = T extends [infer F, ...any[]] ? F : never;
type Last<T> = T extends [...any[], infer L] ? L : never;
type H = First<[1, 2, 3]>;  // 1
type I = Last<[1, 2, 3]>;   // 3

// Record key
type KeyOf<T> = T extends Record<infer K, any> ? K : never;
type J = KeyOf<{ a: 1; b: 2 }>; // "a" | "b"
```

### Edge Cases

- **`infer` faqat `extends` right side:** `T extends infer U ? U : never` — ishlaydi, identity transformation
- **Multiple infer same name:** pozitsiyaga qarab union (covariant) yoki intersection (contravariant)
- **`infer` constraint (TS 4.7+):** `infer R extends number` — inferred type cheklash
- **Higher-order infer:** generic argument'da infer — `Array<infer U>` (rasmiy `(infer U)[]` afzal)
- **`infer` in nested conditional:** outer conditional false branch'da inner infer yo'qoladi

### Follow-up savollar

1. "`infer` overloaded function bilan qanday ishlaydi?" — Faqat **oxirgi** signature inferred bo'ladi (TS cheklov)
2. "`infer` constraint qachon kerak?" — Template literal'dan numeric/boolean parse qilish

</details>

---

### Savol 3: Distributive conditional type nima? Qachon distribute bo'ladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Conditional type'ga **union type** berilganda, TS uni har member'ga alohida qo'llaydi va natijalarni union qiladi. Buni "distributive conditional" deyiladi.

### To'liq tushuntirish

Distribution shartlari (uchchasi bajarilishi shart):
1. **Conditional type** bo'lishi kerak (`T extends U ? X : Y`)
2. **Naked type parameter** (wrapper'siz — `[T]`, `{ x: T }` yo'q)
3. **Union type** berilishi kerak (`string | number`)

`never` — bo'sh union (0 ta member) — distributive'da 0 marta distribute → natija `never`.

To'xtatish: `[T] extends [U]` — wrapper tuple. Yoki har qanday wrapper: `{ x: T } extends { x: U }`.

### Kod misol

```typescript
// Distributive
type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>;
// = ToArray<string> | ToArray<number>
// = string[] | number[]
// EMAS: (string | number)[]

// Non-distributive — wrapper bilan
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type B = ToArrayNonDist<string | number>; // (string | number)[]

// `never` — bo'sh union
type C = ToArray<never>; // never (0 marta distribute)

// Exclude / Extract — distributive misollari
type MyExclude<T, U> = T extends U ? never : T;
type D = MyExclude<"a" | "b" | "c", "a" | "c">;
// = ("a" extends "a"|"c" ? never : "a") | ("b" extends "a"|"c" ? never : "b") | ...
// = never | "b" | never = "b"

type MyExtract<T, U> = T extends U ? T : never;
type E = MyExtract<string | number | boolean, string | boolean>;
// = string | never | boolean = string | boolean

// Real use case — branded union'dan ma'lum brand'ni filter
type Pet = { kind: "dog"; bark(): void } | { kind: "cat"; meow(): void };
type DogPet = Extract<Pet, { kind: "dog" }>;
// = { kind: "dog"; bark(): void }
```

### Edge Cases

- **Inline conditional check:** `T extends any ? ... : ...` — distribute trigger (any har qanday union'ga mos)
- **Generic constraint distribution:** `function f<T extends string | number>(x: T)` — generic instance da distribute bo'lmaydi (T ma'lum)
- **`boolean` distribution:** `boolean = true | false` — distributive'da har biri uchun alohida
- **Tuple T:** tuple element distribute bo'lmaydi — tuple type non-distributive bo'lib qoladi
- **`unknown` vs `any`:** `unknown extends T` — har doim false branch (faqat unknown'ga assignable). `any` — ikkala branch

### Follow-up savollar

1. "Distribute bo'lishini xohlamasam?" — `[T] extends [U]` wrapper
2. "`never` distribution'ni qanday hisobga olaman?" — IsNever check: `[T] extends [never] ? true : false`

</details>

---

### Savol 4: `infer extends` constraint (TS 4.7+) nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`infer U extends SomeType` — inferred type'ni cheklash. Match topilsa va constraint'ga mos kelsa true branch; aks holda false branch.

### To'liq tushuntirish

Standart `infer U` — match topilgan har qanday type'ni qabul qiladi. `infer U extends Number` — type'ni number'ga aylantirib match qilish (template literal'dan).

Foydali use case'lar:
1. Template literal'dan typed value extraction: `"42"` → `42` (number literal)
2. Inferred type'ni constraint'ga rioya qilishini garantiya qilish
3. Recursive type'larda branch'ni progressiv toraytirish

### Kod misol

```typescript
// String'dan number literal parse
type ParseInt<T extends string> = T extends `${infer N extends number}` ? N : never;
type A = ParseInt<"42">;     // 42 (number literal!)
type B = ParseInt<"hello">;  // never
type C = ParseInt<"3.14">;   // 3.14

// Boolean parse
type ParseBool<T extends string> = T extends `${infer B extends boolean}` ? B : never;
type D = ParseBool<"true">;   // true
type E = ParseBool<"false">;  // false
type F = ParseBool<"maybe">;  // never

// Bigint parse
type ParseBigint<T extends string> = T extends `${infer N extends bigint}` ? N : never;
type G = ParseBigint<"100">;  // 100n

// Tuple'dan faqat ma'lum type'larni filter
type NumbersOnly<T extends any[]> =
  T extends [infer Head extends number, ...infer Rest]
    ? [Head, ...NumbersOnly<Rest>]
    : T extends [any, ...infer Rest]
      ? NumbersOnly<Rest>
      : [];

type H = NumbersOnly<[1, "a", 2, "b", 3]>; // [1, 2, 3]

// URL path param extraction
type ExtractRouteParams<T extends string> =
  T extends `${string}/:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<`/${Rest}`>
    : T extends `${string}/:${infer Param}`
      ? Param
      : never;

type I = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

### Edge Cases

- **TS 4.7 talab qiladi:** eski TS versiyalarida `infer X extends Y` ishlamaydi
- **Literal constraint:** `infer X extends 1` — faqat `1` literal'ga match qiladi (`"1"` template literal'dan)
- **Constraint sifatida union:** `infer X extends "a" | "b"` — har literal bilan match qilish
- **Constraint match bo'lmasa:** false branch (constraint qattiq filter sifatida)
- **Recursive type bilan:** tail position'da ishlatish — instantiation depth'ni saqlaydi

### Follow-up savollar

1. "`infer X extends Y` va `T extends Y` qachon farqi?" — `infer X extends Y` — match + cast. `T extends Y` — har T uchun check
2. "Template literal raqamlar uchun nima qiladi?" — `${infer N extends number}` template'dan numeric literal extract

</details>

---

### Savol 5: Deferred conditional type nima? Generic body'da nima uchun ishlamaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic funksiya body'da `T` hali generic parameter — concrete type ma'lum emas. Conditional type **evaluate bo'lmaydi**, deferred holatda qoladi. Funksiya implementation'da assertion yoki overload kerak.

### To'liq tushuntirish

`function process<T>(value: T): T extends string ? string[] : T` — bu signature TS uchun. Caller `T` ni aniq ma'lum qiladi: `process("hello")` da `T = "hello"`, return type `string[]`.

Lekin body ichida `T` hali abstract — `typeof value === "string"` value'ni narrow qiladi, lekin `T` type parameter'ni narrow qilmaydi. Compiler conditional type'ni resolve qila olmaydi → assertion zarur.

To'g'ri yondashuv — function overload:

### Kod misol

```typescript
// ❌ Deferred conditional — body'da resolve bo'lmaydi
function processConditional<T>(value: T): T extends string ? string[] : T {
  if (typeof value === "string") {
    return value.split("") as any; // ❌ assertion zarur
  }
  return value as any; // ❌
}

// ✅ Function overload — har signature mustaqil
function process(value: string): string[];
function process<T>(value: T): T;
function process(value: unknown): unknown {
  if (typeof value === "string") return value.split("");
  return value;
}

const a = process("hello"); // string[]
const b = process(42);       // 42
const c = process(true);     // true

// ❌ Yana bir deferred misol — distributive kerak
type StringOrNumber<T> = T extends string ? `s:${T}` : T extends number ? `n:${T}` : never;

function tag<T extends string | number>(value: T): StringOrNumber<T> {
  if (typeof value === "string") {
    return `s:${value}` as StringOrNumber<T>; // assertion zarur
  }
  return `n:${value}` as StringOrNumber<T>;
}

// ✅ Overload bilan
function tagOverload(value: string): `s:${string}`;
function tagOverload(value: number): `n:${number}`;
function tagOverload(value: string | number): string {
  return typeof value === "string" ? `s:${value}` : `n:${value}`;
}
```

### Edge Cases

- **Type narrowing limit:** TS variable type'ni narrow qiladi, generic type parameter'ni narrow qilmaydi
- **Generic constraint narrowing (TS 5.4+):** generic constraint'da control flow analysis qisman ishlaydi (kontrolflow ichida `T` constraint bo'yicha narrow bo'ladi), lekin conditional return type'ga ta'sir qilmaydi
- **Conditional return assignability:** body type'ni return type'ga assign — TS conservative, assertion zarur
- **Overload signature order:** specific signature avval, general — keyin

### Follow-up savollar

1. "Overload signature lar implementation signature'ga mos bo'lishi kerakmi?" — Implementation signature barcha overload'larni qamrash kerak (parameter va return type)
2. "Function overload va conditional type qachon ishlatish kerak?" — Caller API: conditional type. Implementation: overload + manual branching

</details>

---

### Savol 6: `infer` covariant vs contravariant pozitsiyada — nima farq? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bir xil `infer R` bir nechta joyda bo'lsa pozitsiyaga qarab natija farqli:
- **Return type pozitsiyasi** (covariant) → **UNION**
- **Parameter pozitsiyasi** (contravariant) → **INTERSECTION**

### To'liq tushuntirish

Variance qoidalari:
- **Covariant:** "output" pozitsiya — funksiya har xil narsalar qaytarishi mumkin. Candidate'lar union'ga birlashadi (har biri mumkin)
- **Contravariant:** "input" pozitsiya — funksiya har xil narsalarni qabul qilishi kerak. Candidate'lar intersection'ga birlashadi (barchasi mos kelishi shart)

Bu mexanizm `UnionToIntersection` utility'sining asosi.

### Kod misol

```typescript
// Covariant (return type) → UNION
type CoInfer<T> = T extends {
  a: () => infer R;
  b: () => infer R;
} ? R : never;

type A = CoInfer<{ a: () => string; b: () => number }>;
// R = string | number (union)

// Contravariant (parameter) → INTERSECTION
type ContraInfer<T> = T extends {
  a: (x: infer P) => void;
  b: (x: infer P) => void;
} ? P : never;

type B = ContraInfer<{ a: (x: string) => void; b: (x: number) => void }>;
// P = string & number = never (primitive intersection — bo'sh)

// Object intersection — foydali
type C = ContraInfer<{
  a: (x: { name: string }) => void;
  b: (x: { age: number }) => void;
}>;
// P = { name: string } & { age: number } = { name: string; age: number }

// UnionToIntersection — contravariant infer ishlatadi
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void
    ? I
    : never;

type D = UnionToIntersection<{ a: 1 } | { b: 2 }>;
// = { a: 1 } & { b: 2 } = { a: 1; b: 2 }

// Function overload — faqat oxirgi signature inferred
declare function overloaded(x: string): string;
declare function overloaded(x: number): number;

type E = MyReturnType<typeof overloaded>;
// number — faqat OXIRGI overload signature ishlatiladi

type MyReturnType<T> = T extends (...args: any) => infer R ? R : never;
```

### Edge Cases

- **`strictFunctionTypes: false`:** parameter bivariant — contravariant intersection ishlamasligi mumkin
- **Method vs function property:** method signature bivariant default'da, arrow function property contravariant
- **Multiple infer different positions:** mix covariant + contravariant — natija combined
- **Higher-rank polymorphism:** TS to'liq qo'llab-quvvatlamaydi — chegara

### Follow-up savollar

1. "Why this works for UnionToIntersection?" — Distributive conditional union'ni function family'ga aylantiradi, keyin contravariant infer intersection oladi
2. "Bivariant bilan nima farq?" — Bivariant — har ikki yo'nalishda assignable. Strict — covariant return, contravariant param

<details>
<summary><strong>Deep Dive</strong></summary>

TC39 type theory: function type `(A) → B` ning subtype relationship:
- `(A) → B <: (A') → B'` agar `A' <: A` (contravariant param) va `B <: B'` (covariant return)

TypeScript spec'da type checker bu qoidalarni `isTypeAssignableTo` checker'da implement qiladi. `inferTypes` algorithm har xil pozitsiyalarda candidates'ni yig'adi: union (covariant) yoki intersection (contravariant).

Real industry use case: function composition, lens'lar, optic'lar — variance to'g'ri ishlatilmasa type system noto'g'ri natija beradi.

</details>

</details>

---

### Savol 7: Recursive conditional type va recursion limit qanaqa? Tail recursion (TS 4.5+) nima beradi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Recursive conditional type — type o'z ichida o'zini referencing qiladi. TS recursion depth'ni 50 darajaga cheklaydi (instantiation excess). TS 4.5+ tail-recursive optimization — accumulator pattern bilan 1000+ darajagacha mumkin.

### To'liq tushuntirish

Recursion limit: 50 darajadan keyin `Type instantiation is excessively deep` xato. Sabab: infinite loop'ni oldini olish, compile time'ni cheklash.

Tail recursion: agar recursive call **eng oxirgi pozitsiya**da bo'lsa (accumulator pattern), TS optimization qiladi — har step uchun stack frame yaratmaydi, iterativ ishlaydi. Bu o'z chegarasini ko'taradi (1000+).

Non-tail recursion'da `[...PrevResult, T]` — eng oxirgi bo'lib qaytarilsa tail recursive. Lekin `T extends [...A, infer B] ? B[] : never` non-tail (B[] keyin processing).

### Kod misol

```typescript
// Non-tail recursive — 50 limit
type Reverse<T extends any[]> =
  T extends [infer First, ...infer Rest]
    ? [...Reverse<Rest>, First] // ❌ recursive call wrapper'da — non-tail
    : [];

// type LongTuple = [...50 elementdan ko'p...];
// type Reversed = Reverse<LongTuple>; // ❌ Excessively deep

// ✅ Tail recursive — accumulator
type ReverseTail<T extends any[], Acc extends any[] = []> =
  T extends [infer First, ...infer Rest]
    ? ReverseTail<Rest, [First, ...Acc]> // ✅ recursive call eng oxirgi pozitsiyada
    : Acc;

type A = ReverseTail<[1, 2, 3]>; // [3, 2, 1]
type B = ReverseTail<[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]>; // ✅ 1000+ ishlaydi

// DeepAwaited — tail recursive (TS optimize qiladi)
type DeepAwaited<T> =
  T extends Promise<infer U>
    ? DeepAwaited<U> // tail position
    : T;

type C = DeepAwaited<Promise<Promise<Promise<string>>>>; // string

// Range generator — accumulator pattern
type RangeHelper<N extends number, Acc extends number[]> =
  Acc["length"] extends N
    ? Acc
    : RangeHelper<N, [...Acc, Acc["length"]]>; // tail

type Range<N extends number> = RangeHelper<N, []>;
type D = Range<5>; // [0, 1, 2, 3, 4]

// String parser — accumulator
type SplitString<S extends string, Delimiter extends string, Acc extends string[] = []> =
  S extends `${infer Head}${Delimiter}${infer Tail}`
    ? SplitString<Tail, Delimiter, [...Acc, Head]> // tail
    : [...Acc, S];

type E = SplitString<"a,b,c,d", ",">; // ["a", "b", "c", "d"]
```

### Edge Cases

- **`Type instantiation is excessively deep and possibly infinite`:** klasik xato. Yechim: tail recursion yoki depth limit
- **Distributive recursion:** union member'lar ham distribute bo'ladi — har biri 50 limit
- **Cache:** TS type checker recursive type natijalarini cache qiladi — qayta call tezroq
- **Mutual recursion:** `A<T> = ... B<T> ...; B<T> = ... A<T> ...` — limit aralash
- **Instantiation depth:** non-tail recursion 50, tail-recursive (TS 4.5+) 1000 (`maxInstantiationCount`). Limit `tsc` source'da hardcoded — config orqali o'zgartirib bo'lmaydi

### Follow-up savollar

1. "50 limit'ni qanday bypass qilamiz?" — Tail recursion accumulator + iterativ approach
2. "Performance ta'siri qancha?" — Recursive type compile time'ni sezilarli oshiradi, build time'da o'lchanadi

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript 4.5 release: tail-call optimization. AST'da recursive instantiation eng oxirgi node bo'lsa, TS instantiation stack o'rniga iteration ishlatadi. Bu `Tuple<...>` manipulation'lar uchun kritik — fp-ts, ts-toolbelt, type-fest library'lar bu pattern'ga tayanadi.

Compiler implementation: `getConditionalType` har recursive call'ga `instantiationDepth` counter ko'taradi. Tail position aniqlanganda, counter inkrement qilinmaydi va iterativ resolve davom etadi.

Production tip: agar 50 limit'ga yaqinlashadigan recursive type bo'lsa — refactor majbur. Aks holda IDE intellisense break bo'ladi.

</details>

</details>

---

### Savol 8: Function overload va conditional type — qaysi qachon ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Overload** — har signature bir-biridan mustaqil, alohida documentation, body'da `unknown` ishlatish mumkin. **Conditional type** — generic API'da type relationship'ni ifoda etish, caller-side type inference.

### To'liq tushuntirish

Overload afzal qachon:
- Implementation aniq branch'larga bo'linsa
- Har signature mustaqil API hujjati
- Body'da manual type narrowing

Conditional type afzal qachon:
- Generic API, har T uchun mos return type
- Type relationship qonun (mapping)
- Caller `T` ni dynamic o'tkazadi

Combination: external API conditional, internal implementation overload.

### Kod misol

```typescript
// ❌ Conditional only — body'da assertion kerak
function getDefault1<T extends "string" | "number" | "boolean">(
  kind: T,
): T extends "string" ? string : T extends "number" ? number : boolean {
  if (kind === "string") return "" as any;
  if (kind === "number") return 0 as any;
  return false as any;
}

// ✅ Overload — clean implementation
function getDefault(kind: "string"): string;
function getDefault(kind: "number"): number;
function getDefault(kind: "boolean"): boolean;
function getDefault(kind: string): unknown {
  if (kind === "string") return "";
  if (kind === "number") return 0;
  return false;
}

const a = getDefault("string");  // string
const b = getDefault("number");  // number

// ✅ Conditional type — generic API
type RouteParams<Path extends string> =
  Path extends `${string}/:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof RouteParams<`/${Rest}`>]: string }
    : Path extends `${string}/:${infer Param}`
      ? { [K in Param]: string }
      : {};

function route<P extends string>(path: P, params: RouteParams<P>): string {
  return path;
}

route("/users/:userId/posts/:postId", { userId: "1", postId: "2" }); // ✅
// route("/users/:userId", { wrong: "1" }); // ❌

// Combined: overload + conditional type
type Awaitable<T> = T | Promise<T>;
function awaitValue<T>(value: T): T extends Promise<infer U> ? Promise<U> : T;
function awaitValue<T>(value: Awaitable<T>): Awaitable<T> {
  return value;
}
```

### Edge Cases

- **Overload limit:** TS spec'da yo'q, lekin 5-10 ortda chalkashlik. `infer` overloaded function'dan faqat **oxirgi** signature oladi
- **Conditional generic constraint:** `<T extends X>` constraint generic'da, body'da T bilan ishlash assertion talab qiladi
- **`as` assertion alternative:** type guard function — `is T` predicate
- **`satisfies` overload bilan:** har overload mustaqil — `satisfies` global validation uchun

### Follow-up savollar

1. "Overload signature lar va impl signature bog'liqligi qanday?" — Impl signature barcha overload'larni qamrashi shart (caller'ga ko'rinmaydi)
2. "Conditional type complexity qachon refactor signal?" — 3-4 nested conditional + recursive — refactor variant qidirish

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 9: `Exclude` va `Extract` qo'lda implement qiling [Middle]

**Savol:** Distributive conditional type ishlatib `MyExclude` va `MyExtract` yozing. Distribute jarayonini qadam-baqadam tushuntiring.

```typescript
// MyExclude<"a" | "b" | "c", "a" | "c"> → "b"
// MyExtract<string | number | boolean, string | boolean> → string | boolean
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
```

### To'liq tushuntirish

Naked type parameter + union → distribute. Har T member alohida tekshiriladi, `never` union'da yo'qoladi.

```typescript
type A = MyExclude<"a" | "b" | "c", "a" | "c">;
// Distribute step-by-step:
// ("a" extends "a"|"c" ? never : "a") | ("b" extends "a"|"c" ? never : "b") | ("c" extends "a"|"c" ? never : "c")
// = never | "b" | never
// = "b"

type B = MyExtract<string | number | boolean, string | boolean>;
// (string extends string|boolean ? string : never) | (number extends ...) | (boolean extends ...)
// = string | never | true | false
// = string | boolean
```

### Kod misol

```typescript
type Event = "click" | "submit" | "load" | "error";
type UserEvents = MyExclude<Event, "error" | "load">;
// = "click" | "submit"

type Primitive = string | number | boolean | null | undefined;
type Nullable = MyExtract<Primitive, null | undefined>;
// = null | undefined
```

### Edge Cases

- **`never` distribution:** `MyExclude<never, T>` = `never` (bo'sh distribute)
- **`any` qiyinligi:** `MyExclude<any, string>` = `any` (any har ikki branch)
- **Object union:** discriminated union'dan ma'lum variant'ni filter — `Extract<Pet, { kind: "dog" }>`

### Follow-up savollar

1. "`Exclude<T, never>` nima qaytaradi?" — `T` (never bo'sh union, hech kim mos kelmaydi)
2. "Non-distributive Exclude kerak bo'lsa?" — `[T] extends [U]` wrapper

</details>

---

### Savol 10: `ReturnType` va `Parameters` qo'lda implement qiling [Middle+]

**Savol:** `infer` keyword ishlatib yozing. Overloaded function bilan nima bo'lishini ham tushuntiring.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type MyReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : any;

type MyParameters<T extends (...args: any) => any> =
  T extends (...args: infer P) => any ? P : never;
```

### To'liq tushuntirish

Pattern matching: function signature shape'da `infer R` (return) yoki `infer P` (parameters tuple) bilan match qilamiz. Constraint `T extends (...args: any) => any` — generic'ni funksiya bilan cheklash.

### Kod misol

```typescript
type A = MyReturnType<() => string>;                     // string
type B = MyReturnType<(x: number) => boolean>;           // boolean
type C = MyParameters<(a: string, b: number) => void>;   // [a: string, b: number]
type D = MyParameters<() => void>;                       // []

// Overload — faqat OXIRGI signature
declare function overloaded(x: string): string;
declare function overloaded(x: number): number;

type E = MyReturnType<typeof overloaded>; // number — TS cheklov

// Qo'shimcha utility'lar
type FirstParam<T extends (...args: any) => any> =
  T extends (first: infer F, ...rest: any[]) => any ? F : never;

type F = FirstParam<(id: string, opts: object) => void>; // string

type MyInstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : never;

class UserAccount {}
type G = MyInstanceType<typeof UserAccount>; // UserAccount

type MyConstructorParameters<T extends new (...args: any) => any> =
  T extends new (...args: infer P) => any ? P : never;

class PaymentGateway { constructor(apiKey: string, timeout: number) {} }
type H = MyConstructorParameters<typeof PaymentGateway>; // [apiKey: string, timeout: number]
```

### Edge Cases

- **Overloaded function:** faqat oxirgi signature'dan `infer` ishlaydi — known limitation
- **Generic function:** `<T>(x: T) => T` — `MyReturnType` `T` deferred (generic preserve)
- **Method signature:** `{ method(): string }` — `MyReturnType<obj["method"]>` ishlaydi
- **`this` parameter:** `(this: Context, x: number) => void` — `MyParameters` tuple'ga `this` qo'shmaydi
- **Abstract constructor:** `abstract new (...) => T` — `MyInstanceType` `abstract` ham qabul qiladi (TS 4.2+)

### Follow-up savollar

1. "Overload'dan barcha signature'larni olish mumkinmi?" — Standard infer bilan yo'q. Recursive trick'lar (e.g. ts-toolbelt) qisman
2. "`Awaited<T>` utility qanday ishlaydi?" — Recursive `Promise<infer U> → Awaited<U>` tail recursion

</details>

---

### Savol 11: `never` va `any` trap'lar — output savoli [Middle+]

**Savol:** Har bir type natijasini ayting va sababini tushuntiring:

```typescript
type Test<T> = T extends string ? "yes" : "no";

type A = Test<string>;
type B = Test<number>;
type C = Test<never>;
type D = Test<any>;
type E = Test<string | number>;
```

Va `IsNever` va `IsAny` to'g'ri implement qiling.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type A = "yes";          // string extends string ✅
type B = "no";           // number extends string ❌
type C = never;          // never = bo'sh union → 0 distribute → never
type D = "yes" | "no";   // any → ham true ham false branch
type E = "yes" | "no";   // distribute: Test<string> | Test<number>
```

### To'liq tushuntirish

**`never`** — bo'sh union (0 ta member). Distributive'da 0 marta distribute → natija `never`. Bu sabab `IsNever<T> = T extends never ? true : false` ishlamaydi (`never` distribute bo'lib `never` qaytaradi).

**`any`** — maxsus type. Ham `string` ga assignable, ham emas — TS conservative natija: ikkala branch union.

### Kod misol

```typescript
// IsNever — non-distributive wrapper bilan
type IsNever<T> = [T] extends [never] ? true : false;

type F = IsNever<never>;  // true
type G = IsNever<string>; // false

// Distributive variant noto'g'ri
type IsNeverWrong<T> = T extends never ? true : false;
type Wrong = IsNeverWrong<never>; // never (kutilgan true emas!)

// IsAny — 0 extends 1 & T trick
type IsAny<T> = 0 extends 1 & T ? true : false;
// 1 & any = any → 0 extends any = true
// 1 & string = never → 0 extends never = false

type H = IsAny<any>;     // true
type I = IsAny<string>;  // false
type J = IsAny<unknown>; // false (unknown ≠ any)

// IsUnknown
type IsUnknown<T> = IsAny<T> extends true
  ? false
  : unknown extends T
    ? true
    : false;

type K = IsUnknown<unknown>; // true
type L = IsUnknown<any>;     // false
type M = IsUnknown<string>;  // false
```

### Edge Cases

- **`never` in mapped type:** `{ [K in never]: T }` = `{}` (bo'sh)
- **`any` propagation:** `any & T = any`, `any | T = any`
- **`unknown` vs `any`:** `unknown extends T` — har doim false branch (faqat unknown'ga assignable)
- **`void` distribution:** `void` distribute bo'ladi, lekin function return only — narrow use case

### Follow-up savollar

1. "Nima uchun `IsNever<T> = T extends never ? true : false` ishlamaydi?" — `never` bo'sh union, distribute 0 marta — natija `never`
2. "IsAny trick'ning sababi?" — `1 & any = any` (any har narsani yutadi), `1 & T = never` agar T = primitive

</details>

---

### Savol 12: Recursive conditional — DeepAwaited va DeepFlatten [Middle+]

**Savol:** Recursive conditional type ishlatib `DeepAwaited` va `DeepFlatten` yozing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type DeepAwaited<T> = T extends Promise<infer U> ? DeepAwaited<U> : T;
type DeepFlatten<T> = T extends (infer U)[] ? DeepFlatten<U> : T;
```

### To'liq tushuntirish

Recursive: agar T `Promise<U>` shape'iga mos kelsa, U bilan rekursiv chaqirish. Aks holda T qaytarish.

TS 4.5+ tail recursion optimize — recursive call eng oxirgi pozitsiyada bo'lganda iterativ ishlaydi.

### Kod misol

```typescript
type DeepAwaited<T> =
  T extends Promise<infer U>
    ? DeepAwaited<U>
    : T;

type A = DeepAwaited<Promise<string>>;                    // string
type B = DeepAwaited<Promise<Promise<number>>>;           // number
type C = DeepAwaited<Promise<Promise<Promise<boolean>>>>; // boolean
type D = DeepAwaited<string>;                             // string

// Built-in `Awaited<T>` (TS 4.5+) — similar semantics
type E = Awaited<Promise<string>>; // string

type DeepFlatten<T> =
  T extends (infer U)[]
    ? DeepFlatten<U>
    : T;

type F = DeepFlatten<string[]>;     // string
type G = DeepFlatten<string[][]>;   // string
type H = DeepFlatten<string[][][]>; // string
type I = DeepFlatten<number>;       // number

// Tricky cases
type J = DeepFlatten<(string | number)[]>;
// Distributive — har element type uchun alohida
// = DeepFlatten<string> | DeepFlatten<number>
// = string | number

type K = DeepFlatten<never[]>;
// never[] → U = never → DeepFlatten<never> → distributive 0 marta
// = never

type L = DeepFlatten<any[]>;
// any[] → U = any → DeepFlatten<any> → maxsus
// = any
```

### Edge Cases

- **TS 4.5+ tail optimization:** 1000+ nested Promise — ishlaydi
- **Eski TS:** 50 darajadan ko'p — `excessively deep` xato
- **Mix array + Promise:** `Promise<string[]>` — `DeepAwaited<Promise<string[]>>` = `string[]` (Promise unwrap, lekin array qoladi)
- **`PromiseLike` standard:** TS built-in `Awaited` `PromiseLike` ham qamraydi

### Follow-up savollar

1. "DeepFlatten readonly array bilan ishlaydimi?" — `T extends readonly (infer U)[]` qo'shish kerak
2. "Object'ni deep'da flatten qilish mumkinmi?" — Murakkab. Object'lar uchun mapped type + recursive

</details>

---

### Savol 13: Distributive to'xtatish — IsUnion, NonDistributiveExclude [Senior]

**Savol:** Non-distributive pattern ishlatib quyidagi utility'larni yozing:

```typescript
// IsNever<never> → true, IsNever<string> → false
// IsUnion<string | number> → true, IsUnion<string> → false
// NDExclude — butun union ni tekshirish
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type IsNever<T> = [T] extends [never] ? true : false;
type IsUnion<T, Copy = T> = T extends any ? ([Copy] extends [T] ? false : true) : never;
type NDExclude<T, U> = [T] extends [U] ? never : T;
```

### To'liq tushuntirish

**`[T]` wrapper** — distribution'ni to'xtatadi. Tuple kontekstida T butun union sifatida.

**`IsUnion` trick** — distribute ichida non-distribute tekshiruv:
- T ma'lum bir member (distribute orqali)
- Copy butun union (parameter sifatida saqlangan)
- Agar Copy butun T ga assignable — single member (not union)

### Kod misol

```typescript
type IsNever<T> = [T] extends [never] ? true : false;

type A = IsNever<never>;  // true
type B = IsNever<string>; // false

// IsUnion — distribute ichida non-distribute
type IsUnion<T, Copy = T> =
  T extends any
    ? [Copy] extends [T] ? false : true
    : never;

type C = IsUnion<string>;          // false
type D = IsUnion<string | number>; // true
type E = IsUnion<never>;           // never (degenerate case)

// Tushuntirish:
// T = string | number, Copy = string | number
// Distribute step 1: T = string → [string|number] extends [string]? → no → true
// Distribute step 2: T = number → [string|number] extends [number]? → no → true
// Natija: true | true = true

// T = string, Copy = string
// T = string → [string] extends [string]? → yes → false

// NonDistributiveExclude
type NDExclude<T, U> = [T] extends [U] ? never : T;

type F = NDExclude<string | number, string | number>;
// = never (butun union mos)

type G = NDExclude<string | number, string>;
// = string | number (butun union string ga to'liq mos emas)

// Distributive `Exclude` farqi:
type H = Exclude<string | number, string>;
// = number (har member alohida)

// Real use case
type RequireKey<T, K extends keyof T> = T & { [P in K]-?: NonNullable<T[P]> };
```

### Edge Cases

- **Wrapper alternativa:** `[T] extends [U]` o'rniga `{ x: T } extends { x: U }` ham ishlaydi (har qanday wrapper)
- **Function wrapper:** `(x: T) => void` — contravariant kontekst, boshqa semantics
- **`never` IsUnion:** `IsUnion<never>` = `never` — never bo'sh union, distribute 0 marta

### Follow-up savollar

1. "Wrapper tuple shape masala emasmi?" — Yo'q, distribution o'chirish uchun har qanday wrapper ishlaydi
2. "Performance ta'siri?" — Wrapper minimal overhead, lekin distribute o'chiqgani uchun ba'zi optimization yo'q

<details>
<summary><strong>Deep Dive</strong></summary>

Spec algoritmi (TC39 type theory adapted to TS):
- `T extends U ? X : Y` distributiv bo'lishi uchun `T` **naked type parameter** bo'lishi shart
- Compiler `isDistributionDependent` flag bilan ajratadi — naked yoki wrapped
- Wrapper tuple (`[T]`) — `instantiateType` da `TupleType` sifatida saqlanadi, distributiv bo'lmaydi

`IsUnion` trick'ning ichki mexanizmi: distributive conditional ichidan boshlangan har "iteration"da `Copy` saqlanadi (parameter default — bir marta resolve qilingan). Distribute har `T` member uchun `[Copy] extends [T]` non-distributive tekshiruv — `Copy` butun union, `T` individual member. Agar union > 1 member bo'lsa, hech qaysi `T` `Copy` ni qamrab olmaydi.

Real-world: ts-toolbelt, type-fest, fp-ts library'lar `IsUnion`, `UnionToTuple`, `IsLiteral` kabi utility'larni distribution boshqaruvi orqali implement qiladi. Performance: har wrapper minimal cost, lekin recursive type bilan birga ishlatilganda compile time'ga ta'sir qiladi.

</details>

</details>

---

### Savol 14: `UnionToIntersection<U>` implement qiling [Senior]

**Savol:** Contravariant `infer` pozitsiyasidan foydalanib, union type'ni intersection'ga aylantiring.

```typescript
// UnionToIntersection<{ a: 1 } | { b: 2 }> → { a: 1 } & { b: 2 }
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void
    ? I
    : never;
```

### To'liq tushuntirish

Algoritm:
1. Distributive: `U extends any ? (x: U) => void : never` — har U member uchun function type yaratish
2. Natija: `((x: A) => void) | ((x: B) => void)` (union of functions)
3. `infer I` parameter pozitsiyasida (contravariant) — barcha candidate'lar intersection sifatida birlashadi

Bu variance mexanizmiga asoslangan. Parameter contravariant — function har xil input'larni qabul qilishi kerak → barcha mos kelishi shart → intersection.

### Kod misol

```typescript
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void
    ? I
    : never;

type A = UnionToIntersection<{ a: 1 } | { b: 2 }>;
// = { a: 1 } & { b: 2 } = { a: 1; b: 2 }

type B = UnionToIntersection<string | number>;
// = string & number = never (primitive intersection bo'sh)

type C = UnionToIntersection<{ x: string } | { y: number } | { z: boolean }>;
// = { x: string; y: number; z: boolean }

// Real use case: tuple of objects'ni single object'ga merge
type MergeObjects<T extends object[]> = UnionToIntersection<T[number]>;

type D = MergeObjects<[
  { name: string },
  { age: number },
  { email: string }
]>;
// = { name: string } & { age: number } & { email: string }
// = { name: string; age: number; email: string }

// Discriminated union'dan barcha key'lar
type AllKeys<T> = UnionToIntersection<keyof T extends infer K ? { [P in keyof K]: K } : never>;
```

### Edge Cases

- **Primitive intersection:** `string & number = never` — primitive union'da foydasiz
- **Object property conflict:** `{ x: string } | { x: number }` → `{ x: string } & { x: number }` = `{ x: never }`
- **Function union:** `((x: A) => B) | ((x: C) => D)` → `(x: A & C) => B | D` (param intersect, return union)
- **`unknown` member:** `string | unknown` = `unknown` — union ko'pincha collapse

### Follow-up savollar

1. "Distribute o'chgan bo'lsa-chi?" — `[U] extends [any]` wrapper'da algoritm ishlamaydi
2. "Order saqlanadimi?" — Yo'q. TypeScript intersection order'ni saqlamaydi (canonical normalization)

<details>
<summary><strong>Deep Dive</strong></summary>

Variance theory: function `(A) → B` — A contravariant pozitsiya, B covariant. Inference algorithm har xil pozitsiyalardan candidate'larni yig'ib variance'ga ko'ra natija beradi.

Real industry use case: TypeScript ts-toolbelt, type-fest, fp-ts'da intensively ishlatiladi. `UnionToTuple<U>`, `IsUnion<U>` kabi advanced utility'lar shu mexanizmga tayanadi.

Limitation: TypeScript variance to'liq Hindley-Milner emas — local inference. Higher-rank polymorphism qisman qo'llab-quvvatlanadi.

</details>

</details>

---

### Savol 15: Conditional type bilan deep optional — TypeScript magic [Senior]

**Savol:** `DeepPartial<T>` yozing — har nested property optional bo'lsin:

```typescript
type Config = {
  api: {
    url: string;
    headers: { auth: string };
  };
  ui: { theme: "dark" | "light" };
};

// DeepPartial<Config> →
// {
//   api?: { url?: string; headers?: { auth?: string } };
//   ui?: { theme?: "dark" | "light" };
// }
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepPartial<T[K]>
    : T[K];
};
```

### To'liq tushuntirish

Mapped type + conditional:
1. `[K in keyof T]?` — har property optional
2. `T[K] extends object` — agar nested object, recursive DeepPartial
3. `T[K] extends Function` — Function ham object, lekin recurse qilmaslik
4. Primitive — `T[K]` o'zgarmagan

Edge case'lar: Array, Date, Map — turli yondashuv.

### Kod misol

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : T[K] extends Array<infer U>
        ? Array<DeepPartial<U>>
        : DeepPartial<T[K]>
    : T[K];
};

interface AppConfig {
  api: {
    url: string;
    timeout: number;
    headers: { auth: string; version: string };
  };
  ui: {
    theme: "dark" | "light";
    layouts: Array<{ name: string; cols: number }>;
  };
  callbacks: {
    onError(err: Error): void;
  };
}

type PartialConfig = DeepPartial<AppConfig>;

// Hammasi optional
const partial: PartialConfig = {
  api: {
    headers: { auth: "Bearer xxx" }, // version optional
  },
  ui: {
    layouts: [{ name: "main" }], // cols optional
  },
  // callbacks optional
};

// DeepRequired — teskari
type DeepRequired<T> = {
  [K in keyof T]-?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepRequired<T[K]>
    : T[K];
};

type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function ? T[K] : DeepReadonly<T[K]>
    : T[K];
};

// DeepMutable — readonly'ni o'chirish
type DeepMutable<T> = {
  -readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function ? T[K] : DeepMutable<T[K]>
    : T[K];
};
```

### Edge Cases

- **Function preservation:** `Function extends object` — true. Recurse qilsak signature buziladi. `extends Function` check zarur
- **Array element:** `Array<T>` ham `object` — `Array<infer U>` bilan extract, har element DeepPartial
- **Date, Map, Set:** `instanceof` objects — explicit branch yoki built-in type guard
- **Tuple:** `[string, number]` — DeepPartial'da har element optional bo'lmaydi (tuple structural)
- **`null` and `undefined`:** `object` extends'ga mos kelmaydi — primitive branch'da qoladi

### Follow-up savollar

1. "Recursion limit yo'qmi?" — 50 daraja — chuqur nested config refactor talab qiladi
2. "Performance ta'siri qancha?" — Murakkab DeepPartial application — sezilarli compile time

<details>
<summary><strong>Deep Dive</strong></summary>

`DeepPartial` recursion mexanizmi: compiler har `T[K]` uchun `extends object` check qiladi va recursive instantiation generate qiladi. `instantiationDepth` har step'da inkrement bo'ladi.

Function preservation muammosi: JS'da function ham `typeof === "function"`, lekin TS type checker `Function extends object` true qaytaradi (chunki function prototype `Object.prototype`'dan inherit qiladi). `[K in keyof T]` mapped — function'ning property'lari (`length`, `name`, `prototype`, `apply`, ...) ham iterate qilinadi, callable signature buziladi.

Built-in object'lar (`Date`, `RegExp`, `Map`, `Set`) — `extends object` true, lekin internal slot'lar (`[[DateValue]]`, `[[MapData]]`) type'ga ko'rinmaydi. `DeepPartial<Date>` natijasi — barcha `Date.prototype` method'lari optional, lekin runtime'da `new Date()` instance'i internal slot'larga ega bo'ladi.

Real-world: form library'lar (react-hook-form), config merging (Webpack, Vite), API DTO partial update — barchada DeepPartial pattern ishlatiladi. Production'da Function/Array/Date branch'lari MAJBURIY — aks holda runtime'da type-system silently buziladi.

</details>

</details>

---

## Bug fix savollar

### Savol 16: Distribute o'chmagan IsNever — toping va tuzating [Middle+]

**Savol:** Quyidagi kod `IsNever<never>` uchun `true` qaytarmaydi. Xatoni toping va tuzating:

```typescript
type IsNever<T> = T extends never ? true : false;

type A = IsNever<never>;   // kutilgan: true
type B = IsNever<string>;  // kutilgan: false
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`T extends never` — naked type parameter pozitsiyasida `never` distributive. `never` bo'sh union (0 member), 0 marta distribute — natija `never` (true emas, false ham emas). Conditional branch hech qachon evaluate qilinmaydi.

### Kod misol

```typescript
// ❌ Distribute trap
type BadIsNever<T> = T extends never ? true : false;

type A1 = BadIsNever<never>;   // never (kutilgan true emas)
type B1 = BadIsNever<string>;  // false

// ✅ Yechim — wrapper bilan distribution to'xtatish
type IsNever<T> = [T] extends [never] ? true : false;

type A2 = IsNever<never>;   // true ✅
type B2 = IsNever<string>;  // false ✅

// Boshqa wrapper variantlar (ham ishlaydi)
type IsNeverAlt<T> = (T extends T ? false : true) extends true ? true : false;
// T extends T — har T uchun (jumladan never uchun 0 marta), trick
```

### Edge Cases

- Wrapper tuple — eng oddiy va idiomatic
- Object wrapper `{ x: T } extends { x: never }` ham ishlaydi
- Function wrapper variance bilan murakkablashadi — afzal emas
- `IsNever<any>` — `[any] extends [never]` false → false (any never emas)

### Follow-up savollar

1. **"Nima uchun `never extends X` `true` emas?"** — Spec'da `never` har narsaga assignable (bottom type). Lekin distributive pozitsiyada bo'sh union sifatida qaraladi — distribute 0 marta.
2. **"`Exclude<T, U>` `never` bilan qanday ishlaydi?"** — `Exclude<never, X>` = `never` (distribute 0 marta). Bu bizning naive `IsNever` xatosi bilan bir xil sabab.

</details>

---

### Savol 17: Recursive type "excessively deep" — toping va tuzating [Senior]

**Savol:** Bu kod `Type instantiation is excessively deep and possibly infinite` xatosi beradi. Tuzating:

```typescript
type Reverse<T extends any[]> =
  T extends [infer First, ...infer Rest]
    ? [...Reverse<Rest>, First]
    : [];

type R = Reverse<[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20,
                  21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38,
                  39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52]>;
// Error: Type instantiation is excessively deep
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`Reverse<Rest>` recursive call **non-tail pozitsiyada** — natija `[...Reverse<Rest>, First]` ichida wrapper bor (tuple spread). TS 4.5+ tail-recursion optimization faqat recursive call eng oxirgi pozitsiyada bo'lganda ishlaydi. Non-tail uchun limit 50 daraja.

### Kod misol

```typescript
// ❌ Non-tail recursion — 50 daraja limit
type BadReverse<T extends any[]> =
  T extends [infer First, ...infer Rest]
    ? [...BadReverse<Rest>, First]  // ❌ recursive call spread ichida
    : [];

// ✅ Tail recursion — accumulator pattern
type Reverse<T extends any[], Acc extends any[] = []> =
  T extends [infer First, ...infer Rest]
    ? Reverse<Rest, [First, ...Acc]>  // ✅ recursive call eng oxirgi pozitsiyada
    : Acc;

type R1 = Reverse<[1, 2, 3]>;
// [3, 2, 1]

type R2 = Reverse<[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20,
                   21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38,
                   39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52]>;
// ✅ Ishlaydi — tail-recursive 1000 daraja limit
```

### Tail recursion qoidalari

Recursive call quyidagi pozitsiyalarda **tail** hisoblanadi:
- Conditional true/false branch'ning to'g'ridan-to'g'ri o'zi (`? Recursive<Rest> : ...`)
- Boshqa wrapper'siz return

Non-tail (optimize qilinmaydi):
- Tuple spread ichida (`[...Recursive<X>, Y]`)
- Union ichida (`Recursive<X> | Y`)
- Conditional kontekst (`Recursive<X> extends Y ? ...`)

### Edge Cases

- TS 4.5'gacha hech qanday tail optimization yo'q edi — har recursion 50 limit
- Accumulator pattern — fp'da klassik usul, har recursive type uchun mos
- `[...Acc, First]` — bu ham non-tail? Yo'q — bu **arg** sifatida tuple build qilish, recursive call'ning o'zi tail
- Mutual recursion (A → B → A) — TS optimize qilmaydi, har step instantiation count'ga kiradi

### Follow-up savollar

1. **"Tail recursion qachon ishlamaydi?"** — `[...Recursive<Rest>, X]` non-tail, `Union | Recursive<X>` non-tail, har qanday wrapper ichidagi recursive call non-tail.
2. **"50 va 1000 limit qaerda hardcoded?"** — `tsc` source code'da (`src/compiler/checker.ts` ichida `maxInstantiationDepth` va `maxInstantiationCount`). Config orqali o'zgartirib bo'lmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

TS 4.5 release tail-call optimization implementation: AST'da recursive conditional type aniqlanadi, agar `instantiation` eng oxirgi pozitsiyada bo'lsa, compiler iterative resolution path'ga o'tadi. Stack frame yaratmasdan, har step uchun bir xil instance qayta ishlatadi.

Algorithm detail (simplified):
1. Compiler `getConditionalType` chaqirig'ida `isTailRecursive(node)` tekshiradi
2. Tail recursive bo'lsa — `instantiationDepth` inkrement qilinmaydi
3. Iterative loop bilan resolve davom etadi
4. Non-tail — har step depth + 1, 50 darajada throw

Real-world impact:
- **ts-toolbelt, type-fest:** tuple manipulation library'lar accumulator pattern'ga to'liq tayanadi
- **react-hook-form:** path inference recursive — tail recursion shart
- **GraphQL codegen:** schema-to-types recursive — non-tail variantlar production'da ishlamaydi

Performance tip: agar tail-recursive refactor mumkin bo'lmasa, recursive type'ni split qilish: outer non-recursive, inner recursive tail variant. Yoki manual depth tracker (`Depth extends [...Depth, any]` — depth tuple grow qilib limit'ni manual qo'yish).

</details>

</details>

---

## Xulosa

- Conditional type — `T extends U ? X : Y`, assignability check (inheritance emas)
- `infer` — type extraction, faqat `extends` clause'da
- Distributive — naked T + union → har member alohida. `[T]` wrapper to'xtatadi
- `never` = bo'sh union → distribute 0 marta → `never`. `any` → ikkala branch union
- `infer extends` (TS 4.7+) — string'dan number/boolean parse, inferred type cheklash
- Deferred conditional — generic body'da resolve bo'lmaydi, function overload yechim
- `infer` covariant (return) → union, contravariant (parameter) → intersection
- Recursive conditional 50 daraja limit. Tail recursion (TS 4.5+) 1000+ darajagacha
- `IsNever` → `[T] extends [never]`, `IsAny` → `0 extends 1 & T`
- `UnionToIntersection` — contravariant infer bilan
- DeepPartial/DeepReadonly — mapped type + recursive conditional + Function branch
