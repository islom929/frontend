# Bo'lim 12: Conditional Types Chuqur

> Conditional types — TypeScript type system'da **type-level if/else** logic. `T extends U ? X : Y` sintaksisi bilan type darajasida shartli qaror qabul qilish, `infer` keyword bilan murakkab type'lardan qism ajratib olish (type extraction), distributive behavior bilan union type'larni element-by-element qayta ishlash, va recursive conditional types bilan cheksiz chuqurlikdagi type transformation'lar.
>
> Bu bo'lim [09-advanced-generics.md](09-advanced-generics.md)'dagi asosiy tushunchalarni chuqurlashtirib, real-world pattern'lar, compiler'ning ichki mexanizmlari, va TypeScript type system'ning eng nozik joylarini ochib beradi.

---

## Mundarija

- [Conditional Type Asoslari](#conditional-type-asoslari)
- [`infer` Keyword Chuqur](#infer-keyword-chuqur)
- [Distributive Conditional Types](#distributive-conditional-types)
- [Non-Distributive Conditional Types](#non-distributive-conditional-types)
- [Recursive Conditional Types](#recursive-conditional-types)
- [Conditional Types va Function Overloads](#conditional-types-va-function-overloads)
- [Real-World Use Cases](#real-world-use-cases)
- [Conditional Types Limitations](#conditional-types-limitations)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Conditional Type Asoslari

### Nazariya

Conditional type — type darajasida shartli logic. Syntax JavaScript'ning ternary operatoriga o'xshaydi:

```
T extends U ? TrueType : FalseType
```

**Ma'nosi:** "Agar `T` type `U`'ga assignable bo'lsa — `TrueType`, aks holda — `FalseType`."

Bu ifoda uchta mustaqil qismni o'z ichiga oladi:

1. **`extends` tekshiruvi** — `T extends U` — "T'ni U'ga assign qilish mumkinmi?" (structural subtyping)
2. **True branch** — `extends` rost bo'lganda ishlatiladi, `T` narrowed
3. **False branch** — `extends` yolg'on bo'lganda ishlatiladi

**`extends` assignability misollari:**

| Ifoda | Natija | Sabab |
|-------|--------|-------|
| `string extends string` | ✅ | O'ziga o'zi assignable |
| `"hello" extends string` | ✅ | Literal type parent'ga assignable |
| `string extends "hello"` | ❌ | Keng type tor type'ga assign bo'lmaydi |
| `{ a: string; b: number } extends { a: string }` | ✅ | Ko'proq property = subtype |
| `string extends string \| number` | ✅ | Union member o'z union'ga assignable |
| `never extends T` | ✅ | `never` har qanday type'ga assignable |

**Nima uchun conditional types kerak:** Uchta asosiy muammoni hal qiladi:

1. **Input type'ga qarab output type'ni o'zgartirish** — funksiya/generic return type input'ga bog'liq
2. **Type filtering** — union type'dan ma'lum type'larni ajratish/olib tashlash
3. **Type extraction** — murakkab type ichidan kerakli qismni olish (`infer` bilan)

**Nested conditional — type-level `if/else if/else`:**

```typescript
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  T extends undefined ? "undefined" :
  T extends null ? "null" :
  T extends (...args: any[]) => any ? "function" :
  T extends symbol ? "symbol" :
  T extends bigint ? "bigint" :
  "object";
```

Har `extends` tekshiruvi ketma-ket — birinchi rost bo'lgan branch qaytariladi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori conditional type'ni quyidagi tartibda qayta ishlaydi:

```
1. T concrete type mi yoki generic mi?
   │
   ├── Concrete → darhol evaluate
   │   IsString<string> → string extends string ? true : false → true
   │
   └── Generic → DEFERRED (keyinga qoldiriladi)
       function foo<T>(x: T): IsString<T> { ... }
       → IsString<T> saqlanadi, T aniq bo'lganda evaluate bo'ladi

2. T union type mi?
   │
   ├── Ha + T naked → DISTRIBUTIVE
   │   IsString<string | number> → IsString<string> | IsString<number>
   │
   └── Yo'q (yoki wrapped) → ODDIY evaluate
       [T] extends [string] ? ... → butun T tekshiriladi

3. extends tekshiruvi — assignability check
   │
   ├── True → TrueType evaluate
   └── False → FalseType evaluate
```

**Deferred conditional types — narrowing gotcha:** Generic funksiya ichida `T` hali noma'lum, shuning uchun conditional type resolve bo'lmaydi. Bu — TypeScript'ning fundamental cheklovi.

```typescript
function processValue<T>(value: T): T extends string ? string[] : T {
  // Bu yerda T hali noma'lum
  if (typeof value === "string") {
    // TS: return type hali T extends string ? string[] : T
    // typeof narrowing runtime check, lekin conditional type resolve qilmaydi
    return value.split("") as unknown as T extends string ? string[] : T;
  }
  return value as T extends string ? string[] : T;
}

// Chaqiruvda T aniq bo'lganda — evaluate bo'ladi:
const a = processValue("hello"); // string[] — T = string
const b = processValue(42);      // number — T = number
```

**Runtime narrowing vs compile-time conditional:** Runtime'da `typeof value === "string"` ishlaydi va to'g'ri natija beradi. Lekin TypeScript kompilator generic `T`'ni runtime check orqali aniqlay olmaydi — conditional type compile-time'da, runtime'da esa type'lar yo'q. Bu farq ko'p developer'ni chalkashtiradi.

**Yechim'lar:** Function overloads (pastda batafsil) yoki type assertion.

**`never` va `any` — maxsus xatti-harakatlar:** Conditional type'da bu ikki type boshqalardan farqli ishlaydi (pastda Edge Cases'da batafsil):

- `never` — empty union, distributive'da "nothing" beradi
- `any` — ham true, ham false branch'ga mos keladi (ikkala branch evaluate bo'ladi)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;    // true
type B = IsString<number>;    // false
type C = IsString<"hello">;   // true

// 2. Type name — ketma-ket conditional
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  T extends (...args: any[]) => any ? "function" :
  T extends unknown[] ? "array" :
  "object";

type T1 = TypeName<string>;         // "string"
type T2 = TypeName<number[]>;       // "array"
type T3 = TypeName<() => void>;     // "function"

// 3. API response type — endpoint'ga qarab turli shape
interface User { id: string; name: string; }
interface Post { id: string; title: string; }

type ApiResponse<T extends string> =
  T extends "/users" ? User[] :
  T extends "/users/:id" ? User :
  T extends "/posts" ? Post[] :
  never;

declare function fetchApi<T extends string>(endpoint: T): Promise<ApiResponse<T>>;

// const users = await fetchApi("/users");    // User[]
// const user = await fetchApi("/users/:id"); // User
// const posts = await fetchApi("/posts");    // Post[]

// 4. Type filtering — union'dan ma'lum type'larni olib tashlash
type OnlyStrings<T> = T extends string ? T : never;

type Result = OnlyStrings<"a" | "b" | 1 | 2>;
// "a" | "b" — faqat string literal'lar

// 5. Conditional type generic funksiyada
function process<T extends string | number>(
  value: T
): T extends string ? string[] : number {
  if (typeof value === "string") {
    return value.split("") as T extends string ? string[] : number;
  }
  return ((value as number) * 2) as T extends string ? string[] : number;
}

const a = process("hello"); // string[]
const b = process(42);      // number

// 6. Nullable check
type IsNullable<T> = T extends null | undefined ? true : false;

type N1 = IsNullable<string>;        // false
type N2 = IsNullable<null>;           // true
type N3 = IsNullable<undefined>;      // true

// 7. Function check
type IsFunction<T> = T extends (...args: any[]) => any ? true : false;

type F1 = IsFunction<() => void>;  // true
type F2 = IsFunction<string>;       // false

// 8. Type-safe event payload extraction
interface EventMap {
  click: { x: number; y: number };
  keydown: { key: string; code: string };
  submit: { data: FormData };
}

type EventPayload<T extends keyof EventMap> = EventMap[T];

class TypedEventEmitter {
  private handlers = new Map<string, Array<(...args: any[]) => void>>();

  on<K extends keyof EventMap>(
    event: K,
    handler: (payload: EventPayload<K>) => void
  ): void {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as (...args: any[]) => void);
    this.handlers.set(event, list);
  }

  emit<K extends keyof EventMap>(event: K, payload: EventPayload<K>): void {
    const list = this.handlers.get(event) ?? [];
    list.forEach(h => h(payload));
  }
}

const emitter = new TypedEventEmitter();

emitter.on("click", (payload) => {
  // payload: { x: number; y: number } — avtomatik
  console.log(payload.x, payload.y);
});
```

</details>

---

## `infer` Keyword Chuqur

### Nazariya

`infer` — conditional type ichida type'ni **"olish"** (extract qilish) mexanizmi. `infer U` — "bu pozitsiyada qanday type bo'lsa — uni `U`'ga yoz" degan ma'no.

**`infer` qoidalari:**

1. **Faqat conditional type'ning `extends` clause'ida** ishlatiladi
2. Faqat **`true` branch**'da (`? ...`) ishlatiladi — `false` branch'da (`: ...`) emas
3. Bitta conditional type'da bir nechta `infer` bo'lishi mumkin
4. Bir xil nomli `infer` bir nechta joyda bo'lsa — pozitsiyaga qarab union yoki intersection

**`infer` pozitsiyalari va natijalar:**

| Pozitsiya | Misol | Bir xil nom = | Sabab |
|-----------|-------|:--------------:|-------|
| Return type (covariant) | `() => infer R` | Union | Covariant pozitsiyada union |
| Property (covariant) | `{ a: infer T }` | Union | Covariant pozitsiyada union |
| Parameter (contravariant) | `(x: infer P) => void` | Intersection | Contravariant pozitsiyada intersection |
| Constructor param (contravariant) | `new (x: infer P) => any` | Intersection | Contravariant pozitsiyada intersection |

**Constrained `infer` (TS 4.7+):** `infer U extends SomeType` — inferred type'ni cheklash. Agar moslashtirish natijasi constraint'ga mos kelmasa, false branch'ga tushadi.

```typescript
// TS 4.7'dan OLDIN — qo'shimcha tekshirish kerak
type OldGetString<T> = T extends [infer U, ...any[]]
  ? U extends string ? U : never
  : never;

// TS 4.7+ — infer constraint bilan
type NewGetString<T> = T extends [infer U extends string, ...any[]]
  ? U   // U albatta string
  : never;

type A = NewGetString<["hello", 42]>; // "hello"
type B = NewGetString<[42, "hello"]>; // never — 42 string emas
```

**TS 4.8+ yaxshilanish — literal parsing:** `infer extends number/boolean/bigint` bilan template literal'lardan literal type'larni parse qilish:

```typescript
type ParseInt<T> = T extends `${infer N extends number}` ? N : never;
type X = ParseInt<"42">;    // 42 (number literal!)
type Y = ParseInt<"hello">; // never

type ParseBool<T> = T extends `${infer B extends boolean}` ? B : never;
type Z = ParseBool<"true">; // true
```

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori `infer` ni pattern matching algoritmi bilan qayta ishlaydi:

```
Input: T extends Pattern<infer U> ? TrueType : FalseType

1. T ni Pattern'ga structural moslashtirish
2. infer U joylashgan pozitsiyada qaysi type bo'lsa — U'ga candidate sifatida yozish
3. Agar bir nechta candidate bo'lsa:
   - Covariant pozitsiya → candidates union = U
   - Contravariant pozitsiya → candidates intersection = U
4. U ni true branch'da ishlatish
```

**Co-variant vs contra-variant infer — sabab:**

```typescript
// COVARIANT — return type / property pozitsiya
type CoExample<T> = T extends {
  a: () => infer R;
  b: () => infer R;
} ? R : never;

type X = CoExample<{
  a: () => string;
  b: () => number;
}>;
// R candidates: string, number
// Covariant → UNION: string | number
```

**Nima uchun covariant union:** Agar object'ning `a` va `b` method'lari turli type qaytarsa, chaqiruv natijasi **har ikkisi ham** bo'lishi mumkin. Natija type `string | number` — "string yoki number".

```typescript
// CONTRAVARIANT — parameter pozitsiya
type ContraExample<T> = T extends {
  a: (x: infer P) => void;
  b: (x: infer P) => void;
} ? P : never;

type Y = ContraExample<{
  a: (x: string) => void;
  b: (x: number) => void;
}>;
// P candidates: string, number
// Contravariant → INTERSECTION: string & number = never
```

**Nima uchun contravariant intersection:** Funksiya parameter'i contravariant — funksiyani chaqirish uchun argument **barcha** candidate'larga mos bo'lishi kerak. `a`'ga `string` berish kerak, `b`'ga `number` — bitta qiymat ham string, ham number bo'lishi kerak. Bunday qiymat yo'q → `string & number = never`.

**Constrained infer ichki mexanizmi:**

```
T extends Pattern<infer U extends Constraint>

1-bosqich: Pattern matching — U candidate topish
2-bosqich: U candidate extends Constraint ? tekshirish
   ├── ✅ → U sifatida qabul qilish, true branch
   └── ❌ → false branch'ga o'tish
```

Bu `infer U` + `U extends Constraint ? ...` ga qaraganda yaxshiroq:

- Nested conditional kam
- Kompilator optimizatsiya qila oladi
- Type aniqroq

**Overloaded function `infer`:** TypeScript overloaded funksiya'ning **oxirgi overload signature**'ini oladi, barcha overload'lar union'ini emas:

```typescript
declare function overloaded(x: string): string;
declare function overloaded(x: number): number;

type R = ReturnType<typeof overloaded>; // number (oxirgi overload)
```

**Sabab:** TypeScript type system overload list'idagi **eng oxirgi** (eng keng) signature'ni implementation signature sifatida ishlatadi. Bu deterministik tanlov — lekin ko'p developer buni bilmaydi va tushunmaydi. Bu TS'ning ma'lum cheklovi.

**Runtime'da iz yo'q:** `infer` sof compile-time feature. Compiled JavaScript'da hech qanday iz qolmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Array element type — ElementOf
type ElementOf<T> = T extends (infer U)[] ? U : never;

type A = ElementOf<string[]>;               // string
type B = ElementOf<number[]>;               // number
type C = ElementOf<(string | number)[]>;     // string | number
type D = ElementOf<string>;                  // never

// 2. ReturnType — built-in implementation
type MyReturnType<T extends (...args: any[]) => any> =
  T extends (...args: any[]) => infer R ? R : any;

type R1 = MyReturnType<() => string>;          // string
type R2 = MyReturnType<(x: number) => boolean>; // boolean

// Overloaded function — oxirgi signature
declare function overloaded(x: string): string;
declare function overloaded(x: number): number;
type R3 = MyReturnType<typeof overloaded>; // number (oxirgi overload)

// 3. Parameters — tuple extraction
type MyParameters<T extends (...args: any[]) => any> =
  T extends (...args: infer P) => any ? P : never;

type P1 = MyParameters<(a: string, b: number) => void>; // [a: string, b: number]
type P2 = MyParameters<() => void>;                      // []

// 4. Birinchi parameter
type FirstParam<T extends (...args: any[]) => any> =
  T extends (first: infer F, ...rest: any[]) => any ? F : never;

type F1 = FirstParam<(name: string, age: number) => void>; // string

// 5. Oxirgi tuple element
type LastElement<T extends any[]> = T extends [...any[], infer L] ? L : never;

type L1 = LastElement<[string, number, boolean]>; // boolean

// 6. Awaited — recursive Promise unwrap
type MyAwaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any }
    ? F extends (value: infer V, ...args: infer _) => any
      ? MyAwaited<V>
      : never
    : T;

type A1 = MyAwaited<Promise<string>>;              // string
type A2 = MyAwaited<Promise<Promise<number>>>;     // number (recursive)
type A3 = MyAwaited<string>;                        // string

// 7. Constructor parameters va instance type
class User {
  constructor(public name: string, public age: number) {}
}

type ConstructorParams<T extends abstract new (...args: any[]) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;

type InstanceOf<T extends abstract new (...args: any[]) => any> =
  T extends abstract new (...args: any[]) => infer R ? R : never;

type UserParams = ConstructorParams<typeof User>; // [name: string, age: number]
type UserInstance = InstanceOf<typeof User>;      // User

// 8. Multiple infer — bir nechta qism olish
type FunctionParts<T> = T extends (...args: infer P) => infer R
  ? { params: P; returnType: R }
  : never;

type Parts = FunctionParts<(name: string, age: number) => boolean>;
// { params: [name: string, age: number]; returnType: boolean }

// 9. Constrained infer (TS 4.7+)
type FirstString<T extends any[]> =
  T extends [infer F extends string, ...any[]] ? F : never;

type S1 = FirstString<["hello", 42, true]>; // "hello"
type S2 = FirstString<[42, "hello"]>;       // never

// 10. Template literal infer (TS 4.8+)
type ParseIntLit<T> = T extends `${infer N extends number}` ? N : never;
type I1 = ParseIntLit<"42">;    // 42
type I2 = ParseIntLit<"3.14">;  // 3.14
type I3 = ParseIntLit<"hello">; // never

type ParseBoolLit<T> = T extends `${infer B extends boolean}` ? B : never;
type B1 = ParseBoolLit<"true">;  // true
type B2 = ParseBoolLit<"false">; // false
```

</details>

---

## Distributive Conditional Types

### Nazariya

Distributive conditional type — conditional type'ga **union type** berilganda, TypeScript uni union'ning **har bir member**'iga alohida qo'llaydi va natijalarni qayta union qiladi.

Matematik tarqatish qoidasi:

```
f(A | B | C) = f(A) | f(B) | f(C)
```

**Distributive bo'lish uchun BARCHA shartlar:**

1. Conditional type bo'lishi kerak — `T extends U ? X : Y`
2. Tekshirilayotgan `T` — **naked type parameter** bo'lishi kerak (wrapper'siz)
3. `T`'ga **union type** berilishi kerak

**"Naked type parameter" nima?** — `T` hech qanday wrapper yo'q holda, to'g'ridan-to'g'ri `extends`'dan oldin turadi:

```
✅ NAKED — distributive bo'ladi:
type Dist<T> = T extends string ? "yes" : "no";

❌ WRAPPED — non-distributive:
type NonDist1<T> = [T] extends [string] ? "yes" : "no";     // tuple wrap
type NonDist2<T> = T[] extends string[] ? "yes" : "no";     // array wrap
type NonDist3<T> = Promise<T> extends Promise<string> ? "yes" : "no"; // generic wrap
```

Distributive behavior — TypeScript'ning built-in `Exclude`, `Extract`, `NonNullable` utility type'larining asosi.

<details>
<summary><strong>Under the Hood</strong></summary>

Distributive conditional type'ning ichki mexanizmi:

```
type Dist<T> = T extends U ? X : Y;

Dist<A | B | C>

Kompilator jarayoni:
1. T = A | B | C — union detected
2. T naked type parameter — distribution triggered
3. Split: Dist<A> | Dist<B> | Dist<C>
4. Har birini alohida evaluate:
   - Dist<A> = A extends U ? X : Y → natija1
   - Dist<B> = B extends U ? X : Y → natija2
   - Dist<C> = C extends U ? X : Y → natija3
5. Combine: natija1 | natija2 | natija3
```

**`Exclude` va `Extract` ichidan:**

```typescript
// Exclude<T, U> = T extends U ? never : T

type Result = Exclude<"a" | "b" | "c" | "d", "a" | "c">;
// Step-by-step distributive:
// Exclude<"a", "a" | "c"> | Exclude<"b", "a" | "c"> | Exclude<"c", "a" | "c"> | Exclude<"d", "a" | "c">
// = ("a" extends "a" | "c" ? never : "a") | ("b" extends "a" | "c" ? never : "b") | ...
// = never | "b" | never | "d"
// = "b" | "d"
```

**`never` va `|` (union) identity:** `never` — "empty union" (`|` operatsiyada identity element). Union'dan `never` olib tashlansa — hech narsa o'zgarmaydi:

```
"a" | "b" | never = "a" | "b"
```

Shuning uchun `Exclude`'da `never` qaytarish — element'ni union'dan "o'chirish" degan ma'no. Bu distributive'ning eng kuchli pattern'i.

**`boolean` distributive surprise:** TypeScript'da `boolean = true | false`. Conditional type'da `boolean` distribute bo'ladi — `true` va `false` alohida tekshiriladi:

```typescript
type Check<T> = T extends true ? "yes" : "no";
type B = Check<boolean>;
// boolean distributive qilinadi:
// = Check<true> | Check<false>
// = "yes" | "no"
// ❗ Bu "yes" emas — chunki false ham tekshiriladi
```

**Naked type parameter visual contrast:**

```
Naked:           type F<T> = T extends X ? A : B
                          └─ T to'g'ridan-to'g'ri

Wrapped (tuple): type F<T> = [T] extends [X] ? A : B
                          └─ T tuple ichida

Wrapped (array): type F<T> = T[] extends X[] ? A : B
                          └─ T array element sifatida

Wrapped (generic): type F<T> = Box<T> extends Box<X> ? A : B
                            └─ T generic argument sifatida
```

Wrapper'lar distribution'ni to'xtatadi — union bitta unit sifatida tekshiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. ToArray — distributive
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// = string[] | number[] (distributive!)
// ❗ (string | number)[] emas

// 2. Exclude implementation
type MyExclude<T, U> = T extends U ? never : T;

type E1 = MyExclude<"a" | "b" | "c", "a">;
// "b" | "c"

type E2 = MyExclude<string | number | boolean, number>;
// string | boolean

// 3. Extract implementation
type MyExtract<T, U> = T extends U ? T : never;

type Ex1 = MyExtract<string | number | boolean, number | boolean>;
// number | boolean

// 4. NonNullable
type MyNonNullable<T> = T extends null | undefined ? never : T;

type N1 = MyNonNullable<string | null | undefined>;
// string

// 5. Property filter — KeysOfType
type KeysOfType<T, ValueType> = {
  [K in keyof T]: T[K] extends ValueType ? K : never;
}[keyof T];

interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  isActive: boolean;
}

type StringKeys = KeysOfType<User, string>;  // "name" | "email"
type NumberKeys = KeysOfType<User, number>;  // "id" | "age"
type BoolKeys = KeysOfType<User, boolean>;   // "isActive"

// 6. Discriminated union filter
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "keydown"; key: string }
  | { type: "scroll"; offset: number };

type ClickEvent = Extract<Event, { type: "click" }>;
// { type: "click"; x: number; y: number }

type MouseEvents = Extract<Event, { type: "click" | "scroll" }>;
// { type: "click"; ... } | { type: "scroll"; ... }

// 7. UnionToIntersection — famous trick
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void
    ? I
    : never;

// Ishlashi:
// 1. U distributive: (A extends any ? (x: A) => void : never) | (B extends any ? ...)
// 2. = ((x: A) => void) | ((x: B) => void)
// 3. Butun union (x: infer I) => void ga moslashtiradi
// 4. I contravariant pozitsiyada → intersection: A & B

type UI = UnionToIntersection<{ a: 1 } | { b: 2 }>;
// { a: 1 } & { b: 2 }

// 8. Distributive filter pattern
type Stringify<T> = T extends string | number | boolean ? `${T}` : never;

type S = Stringify<"hello" | 42 | true | { a: 1 }>;
// "hello" | "42" | "true" (object filtered)
```

</details>

---

## Non-Distributive Conditional Types

### Nazariya

Ba'zan distributive xatti-harakatni **to'xtatish** kerak — ya'ni union type'ni butunlay tekshirish. Buning uchun type parameter'ni **wrapper** ichiga olish kerak.

**Non-distributive qilish usullari:**

```typescript
// 1. Tuple wrap — eng ko'p ishlatiladigan (tavsiya)
type NonDist1<T> = [T] extends [string] ? "yes" : "no";

// 2. Array wrap
type NonDist2<T> = T[] extends string[] ? "yes" : "no";

// 3. Function wrap — ⚠️ contravariance ta'sir qiladi
type NonDist3<T> = ((x: T) => void) extends ((x: string) => void) ? "yes" : "no";

// 4. Object wrap
type NonDist4<T> = { value: T } extends { value: string } ? "yes" : "no";
```

**Tavsiya — tuple wrap `[T]`.** Eng oddiy va bashorat qilinadigan. Function wrap contravariance tufayli kutilmagan natija berishi mumkin (quyida).

**Qachon non-distributive kerak:**

1. **Union type'ni butunlay tekshirish** — individual member emas, butun union
2. **`never` tekshirish** — `never` empty union, distribute'da yo'qoladi
3. **Type equality check** — ikki type aynan bir xilmi
4. **Union member count** — nechta member bor

<details>
<summary><strong>Under the Hood</strong></summary>

Non-distributive conditional type'da kompilator union'ni split qilmaydi — butun union type'ni `extends` tekshiruviga beradi:

```
type NonDist<T> = [T] extends [string] ? true : false;

NonDist<string | number>
→ [string | number] extends [string] ?
→ Tekshirish: [string | number] assignable to [string]?
  → string | number assignable to string? ❌ (number string emas)
→ false

Taqqoslang distributive:
type Dist<T> = T extends string ? true : false;

Dist<string | number>
→ (string extends string ? true : false) | (number extends string ? true : false)
→ true | false
→ boolean
```

**Function wrap contravariance surprise:** Function wrap ishlatilganda parameter position contravariant — natija kutilgandan teskari bo'lishi mumkin:

```typescript
type FnDist<T> = ((x: T) => void) extends ((x: string) => void) ? "yes" : "no";

type A = FnDist<string>;        // "yes" (string extends string)
type B = FnDist<string | number>; // "yes" ❗ (kutilmagan!)
// Sabab: (x: string | number) => void extends (x: string) => void
// Parameter contravariant: keng accepted (string | number) tor expected (string)'ga assign bo'ladi
// Aslida: (x: string) => void argument hatto string | number'ga assign bo'ladi
```

Bu TypeScript variance qoidalarining samarasi — function wrap non-distributive uchun ishonchli emas. Tuple wrap yaxshiroq.

**`[T] extends [U]` ishonchliligi:** Tuple wrap'da parameter position yo'q — to'g'ridan-to'g'ri structural check. Shuning uchun variance effekti yo'q, natija bashorat qilinadi:

```typescript
type TupleDist<T> = [T] extends [string] ? "yes" : "no";

type X = TupleDist<string>;         // "yes"
type Y = TupleDist<string | number>; // "no" (butun union tekshiriladi)
type Z = TupleDist<number>;          // "no"
```

**Type equality check — famous pattern:**

```typescript
type Equals<A, B> =
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2)
    ? true
    : false;
```

Bu trick generic function type'larni solishtirish orqali type equality'ni aniqlaydi. Ikki deferred conditional type aynan bir xil bo'lsa, natija `true`.

**`IsUnion` detection:** Union type'ni aniqlash uchun distributive + non-distributive kombinatsiya:

```typescript
type IsUnion<T, C = T> =
  T extends T  // Distributive trigger — T har member'ga split
    ? [C] extends [T]  // Non-distributive: butun C ni current member T bilan taqqoslash
      ? false  // Agar C current member'ga assignable bo'lsa → bitta element
      : true   // Aks holda → ko'p element (union)
    : never;
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Non-distributive check
type IsStringOnly<T> = [T] extends [string] ? "yes" : "no";

type A = IsStringOnly<string>;          // "yes"
type B = IsStringOnly<string | number>; // "no" (butun union)
type C = IsStringOnly<"hello">;          // "yes" (literal subtype)

// 2. IsNever — must be non-distributive
type IsNever<T> = [T] extends [never] ? true : false;

type N1 = IsNever<never>;  // true ✅
type N2 = IsNever<string>; // false
type N3 = IsNever<any>;    // false

// Distributive variant ishlamaydi:
// type IsNeverBad<T> = T extends never ? true : false;
// IsNeverBad<never> = never (empty union distribute)

// 3. StrictEquals — ikki type aynan teng
type StrictEquals<T, U> =
  [T] extends [U]
    ? [U] extends [T]
      ? true
      : false
    : false;

type E1 = StrictEquals<string, string>;           // true
type E2 = StrictEquals<string, number>;            // false
type E3 = StrictEquals<string | number, string>;   // false
type E4 = StrictEquals<"a", "a">;                  // true

// 4. Deep Equals — famous pattern
type Equals<A, B> =
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2)
    ? true
    : false;

type DE1 = Equals<{ a: 1 }, { a: 1 }>;          // true
type DE2 = Equals<{ a: 1 }, { a: number }>;     // false

// 5. IsUnion detection
type IsUnion<T, C = T> =
  T extends T
    ? [C] extends [T]
      ? false
      : true
    : never;

type U1 = IsUnion<string>;          // false
type U2 = IsUnion<string | number>;  // true
type U3 = IsUnion<never>;            // false

// 6. Wrapped vs naked comparison
type NakedCheck<T> = T extends string ? "yes" : "no";
type WrappedCheck<T> = [T] extends [string] ? "yes" : "no";

type A1 = NakedCheck<string | number>;   // "yes" | "no" (distributive)
type A2 = WrappedCheck<string | number>; // "no" (non-distributive)

// 7. Filter with non-distributive
type AllExtendsString<T> = [T] extends [string] ? T : never;

type AE1 = AllExtendsString<"a" | "b">;     // "a" | "b" (butun union string)
type AE2 = AllExtendsString<"a" | 1>;       // never (1 string emas)

// 8. ToArray non-distributive
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type TAD1 = ToArrayNonDist<string | number>;
// (string | number)[] — bitta array, aralash element
// Distributive: string[] | number[] — ikki alohida
```

</details>

---

## Recursive Conditional Types

### Nazariya

TypeScript 4.1+'dan boshlab conditional type'lar **recursive** bo'lishi mumkin — type o'zini o'zi chaqiradi. Bu string manipulation, deep type transformation, va murakkab type computation'lar uchun kerak.

**Recursive conditional type asosiy pattern:**

```
type Recursive<T> =
  T extends BaseCase ? Result :
  T extends RecursiveCase<infer U> ? Transform<Recursive<U>> :
  Default;
```

Har recursive type'da **base case** (to'xtash sharti) **shart** — aks holda kompilator cheksiz loop'ga tushadi.

**Tail-call optimization (TS 4.5+):** Agar recursive call true branch'ida **return pozitsiyasida** tursa (boshqa type bilan wrap qilinmasdan), kompilator uni tail call sifatida aniqlaydi va stack frame'ni qayta ishlatadi. Bu depth limit'ni taxminan **1000**'ga oshiradi (odatdagi **~50**'dan).

**Accumulator pattern — tail-recursive bo'lishning qoidasi:**

```typescript
// ❌ Non-tail — recursive call tuple ichida
type NonTail<T extends string> =
  T extends `${infer C}${infer Rest}`
    ? [C, ...NonTail<Rest>]  // Recursive [C, ...] ichida — tail emas
    : [];

// ✅ Tail — accumulator pattern
type Tail<N extends number, Acc extends 0[] = []> =
  Acc["length"] extends N
    ? Acc                     // Base case
    : Tail<N, [...Acc, 0]>;    // Tail call — return pozitsiyasida
```

Accumulator pattern'da natija har iteration'da `Acc` parameter'ga to'planadi va oxirgi iteration'da qaytariladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Depth limit mexanizmi:** TypeScript kompilatori recursive conditional type'lar uchun cheklangan depth limit qo'ygan. Limit'ga yetganda `Type instantiation is excessively deep and possibly infinite` xatosi chiqadi — bu compile-time stack overflow'ni oldini olish uchun.

- **Non-tail-recursive** — taxminan **50 daraja** limit
- **Tail-recursive** (TS 4.5+) — taxminan **1000 daraja** limit (optimize qilingan)

**Tail-call optimization nima uchun ishlaydi:** Tail position'dagi recursive call kompilatorning "joriy stack frame'ni qayta ishlating" signalini beradi. Stack frame har yangi call'da yaratilmaydi — bitta frame qayta ishlatiladi. Bu functional programming til'larida (Haskell, Scala, Scheme) keng qo'llaniladi, TypeScript'da esa type-level recursion uchun kiritilgan.

```
Non-tail recursion (50 limit):

Recursive<[1, 2, 3]>
  → [1, ...Recursive<[2, 3]>]    ← recursive call [...] ichida
      → [2, ...Recursive<[3]>]
          → [3, ...Recursive<[]>]
              → []
  Har qadamda stack frame yaratiladi

Tail recursion (1000 limit):

Tail<3, []>
  → Tail<3, [0]>         ← recursive call bevosita return
      → Tail<3, [0, 0]>
          → Tail<3, [0, 0, 0]>
              → [0, 0, 0]
  Stack frame qayta ishlatiladi
```

**Accumulator pattern — functional programming texnikasi:** Accumulator "yig'uvchi" parameter — har iteration'da natija unga qo'shiladi. Oxirgi iteration'da (base case) accumulator qaytariladi. Bu pattern O(1) stack space ishlatadi, O(n) o'rniga.

**TypeScript'da tail-recursion tekshiruvi:** Kompilator AST'da recursive call'ning pozitsiyasini tekshiradi. Agar call true branch'ning return pozitsiyasida bo'lsa (boshqa type construct bilan wrap qilinmasa), tail call deb belgilanadi.

Tail qoida'lari:
- `T extends X ? Recursive<...> : Y` — ✅ tail
- `T extends X ? [Recursive<...>] : Y` — ❌ tail emas (tuple ichida)
- `T extends X ? Recursive<...> & Y : Z` — ❌ tail emas (intersection ichida)

**Depth limit'dan qochish:** Counter tuple bilan recursion depth'ni cheklash mumkin:

```typescript
type LimitedRecurse<T, Depth extends unknown[] = []> =
  Depth["length"] extends 10
    ? T  // 10'dan chuqur ketma
    : /* recursive logic */ LimitedRecurse<T, [...Depth, unknown]>;
```

Counter tuple har recursion'da o'sadi — limit'ga yetganda evaluation to'xtaydi. Bu infinite recursion'dan qochish uchun defensive pattern.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. DeepReadonly — nested readonly
type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :   // Funksiya — saqla
  T extends object ? {
    readonly [K in keyof T]: DeepReadonly<T[K]>;
  } :
  T;  // Primitive — base case

interface Config {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
}

type ReadonlyConfig = DeepReadonly<Config>;

// 2. DeepPartial
type DeepPartial<T> =
  T extends (...args: any[]) => any ? T :
  T extends object ? {
    [K in keyof T]?: DeepPartial<T[K]>;
  } :
  T;

const partial: DeepPartial<Config> = {
  server: { port: 3000 }, // ✅ qisman
};

// 3. String manipulation — Trim
type TrimLeft<S extends string> =
  S extends ` ${infer Rest}` ? TrimLeft<Rest> :
  S extends `\n${infer Rest}` ? TrimLeft<Rest> :
  S extends `\t${infer Rest}` ? TrimLeft<Rest> :
  S;

type TrimRight<S extends string> =
  S extends `${infer Rest} ` ? TrimRight<Rest> :
  S extends `${infer Rest}\n` ? TrimRight<Rest> :
  S extends `${infer Rest}\t` ? TrimRight<Rest> :
  S;

type Trim<S extends string> = TrimLeft<TrimRight<S>>;

type A = Trim<"  hello  ">;    // "hello"
type B = Trim<"\n  world\t">;  // "world"

// 4. Split string by delimiter
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Tail}`
    ? [Head, ...Split<Tail, D>]
    : S extends ""
      ? []
      : [S];

type C = Split<"a.b.c", ".">;  // ["a", "b", "c"]
type D = Split<"hello", "">;    // ["h", "e", "l", "l", "o"]

// 5. ReplaceAll — recursive
type ReplaceAll<S extends string, From extends string, To extends string> =
  From extends ""
    ? S
    : S extends `${infer Head}${From}${infer Tail}`
      ? ReplaceAll<`${Head}${To}${Tail}`, From, To>
      : S;

type E = ReplaceAll<"a-b-c-d", "-", "_">;  // "a_b_c_d"

// 6. Tuple Reverse
type Reverse<T extends unknown[]> =
  T extends [infer First, ...infer Rest]
    ? [...Reverse<Rest>, First]
    : [];

type R = Reverse<[1, 2, 3, 4]>;  // [4, 3, 2, 1]

// 7. Flatten — nested array
type Flatten<T extends any[]> =
  T extends [infer First, ...infer Rest]
    ? First extends any[]
      ? [...Flatten<First>, ...Flatten<Rest>]  // Recursive
      : [First, ...Flatten<Rest>]
    : [];

type F = Flatten<[1, [2, 3], [4, [5, 6]]]>;  // [1, 2, 3, 4, 5, 6]

// 8. Tail-recursive — BuildTuple bilan counting
type BuildTuple<N extends number, T extends unknown[] = []> =
  T["length"] extends N ? T : BuildTuple<N, [...T, unknown]>;

// Type-level Add
type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"] extends infer R extends number
    ? R
    : never;

type X = Add<3, 4>;  // 7
type Y = Add<10, 5>; // 15

// 9. Depth-limited recursion
type DeepPartialLimited<T, Depth extends unknown[] = []> =
  Depth["length"] extends 10
    ? T
    : T extends (...args: any[]) => any ? T :
      T extends object ? {
        [K in keyof T]?: DeepPartialLimited<T[K], [...Depth, unknown]>;
      } :
      T;
```

</details>

---

## Conditional Types va Function Overloads

### Nazariya

Funksiya return type'ini input'ga qarab o'zgartirish kerak bo'lganda ikki approach bor — **function overloads** va **conditional type**. Har birining o'z kuchli va zaif tomonlari bor.

**Function overloads:**

```typescript
function process(value: string): string[];
function process(value: number): number;
function process(value: string | number): string[] | number {
  if (typeof value === "string") return value.split("");
  return value * 2;
}

const a = process("hello"); // string[]
const b = process(42);      // number
```

**Conditional type:**

```typescript
type ProcessResult<T> = T extends string ? string[] : number;

function process<T extends string | number>(value: T): ProcessResult<T> {
  if (typeof value === "string") return value.split("") as ProcessResult<T>;
  return ((value as number) * 2) as ProcessResult<T>;
}
```

**Qachon qaysi birini ishlatish:**

| Mezon | Overloads | Conditional Type |
|-------|:---------:|:----------------:|
| Aniq, kam holat (2-3 ta) | ✅ Yaxshi | ❌ Ortiqcha |
| Ko'p holat (5+ ta) | ❌ Ko'p boilerplate | ✅ Yaxshi |
| Union argument bilan chaqirish | ❌ Eng keng overload | ✅ Distributive |
| IDE autocomplete | ✅ Har overload ko'rinadi | ⚠️ Conditional type ko'rinadi |
| Implementation'da narrowing | ✅ Ishlaydi | ❌ Type assertion kerak |
| Generic context | ❌ Murakkab | ✅ Natural |

**Pragmatic guide:**

- **Oddiy, kam holat** — overloads (IDE autocomplete, body'da narrowing)
- **Union argument bilan chaqiriladi** — conditional type (distributive)
- **Ko'p holat (5+)** — conditional type (boilerplate kam)
- **Generic context** — conditional type (natural integration)

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator overload resolution va conditional type evaluation'ni **butunlay farqli mexanizm**'larda bajaradi.

**Overload resolution:** Kompilator barcha overload signature'larni yuqoridan pastga tekshiradi. Birinchi mos keladigan overload tanlanadi. Union argument berilganda kompilator **bitta** overload tanlashga harakat qiladi — lekin union hech bir aniq overload'ga to'liq mos kelmaydi, shuning uchun xato bo'lishi mumkin.

**Conditional type evaluation:** Conditional type `T extends U ? X : Y` ni evaluate qiladi. Agar `T` naked parameter + union bo'lsa — distributive behavior yoqiladi va har member alohida evaluate bo'ladi:

```
Overload resolution:          Conditional type distribution:
process(string | number)      ProcessResult<string | number>
     │                             │
     v                        distribute over union
  overload 1: string → ❌        │
  overload 2: number → ❌    ProcessResult<string> | ProcessResult<number>
  implementation → ❌ Error      │                       │
                              string[]                 number
                                   \                  /
                                 string[] | number  ✅
```

**Overloaded function + `ReturnType` — nozik holat:** TypeScript overloaded funksiya uchun `ReturnType<typeof fn>` chaqirilganda **oxirgi** overload signature'ning return type'ini oladi. Bu deterministik tanlov, lekin ko'p developer'larni chalkashtiradi:

```typescript
declare function overloaded(x: string): string;
declare function overloaded(x: number): number;

type R = ReturnType<typeof overloaded>; // number (oxirgi)
// Kutilgan: string | number — lekin noto'g'ri
```

**Sabab:** TypeScript type system'da funksiya `typeof` orqali type olinganda, signature'larning oxirgisi (implementation signature'ga eng yaqini) tanlanadi. Bu eski design qarori — backward compatibility tufayli o'zgarmaydi.

**Yechim** — `infer` bilan barcha overload'larni olish (chekli):

```typescript
// TypeScript'ning standart yo'li yo'q — faqat oxirgi overload olinadi
// Custom trick mavjud, lekin 4 ta overload'gacha cheklangan
```

**Emit farqi yo'q:** Emitter ikkala yondashuvni (overloads va conditional) bir xil JavaScript'ga aylantiradi — runtime'da farq yo'q. Barcha tanlov — compile-time type safety va developer experience uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy 2-3 overload — overloads yaxshi
function formatValue(value: string): string;
function formatValue(value: number): string;
function formatValue(value: Date): string;
function formatValue(value: string | number | Date): string {
  if (value instanceof Date) return value.toISOString();
  return String(value);
}

formatValue("hello");      // string
formatValue(42);            // string
formatValue(new Date());    // string

// 2. Union argument — conditional type kerak
type ParseResult<T> = T extends string ? number : string;

function parseValue<T extends string | number>(value: T): ParseResult<T> {
  if (typeof value === "string") return parseInt(value) as ParseResult<T>;
  return (value as number).toString() as ParseResult<T>;
}

const input: string | number = Math.random() > 0.5 ? "42" : 42;
const result = parseValue(input);
// result: string | number (distributive)

// 3. Overloads bilan union argument — muammo
function parseOverload(value: string): number;
function parseOverload(value: number): string;
function parseOverload(value: string | number): number | string {
  if (typeof value === "string") return parseInt(value);
  return value.toString();
}

parseOverload("42");  // number
parseOverload(42);    // string
// parseOverload(input); // ❌ No overload matches

// 4. Ko'p holatli — conditional type yaxshi
type ExpandType<T> =
  T extends "small" ? { size: 10 } :
  T extends "medium" ? { size: 50 } :
  T extends "large" ? { size: 100 } :
  T extends "xlarge" ? { size: 200 } :
  T extends "xxlarge" ? { size: 500 } :
  never;

function expand<T extends "small" | "medium" | "large" | "xlarge" | "xxlarge">(
  size: T
): ExpandType<T> {
  const map = { small: 10, medium: 50, large: 100, xlarge: 200, xxlarge: 500 };
  return { size: map[size] } as ExpandType<T>;
}

const s1 = expand("small");  // { size: 10 }
const s2 = expand("large");  // { size: 100 }

// 5. Generic context — conditional natural
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

function unwrap<T>(value: T): UnwrapPromise<T> {
  if (value instanceof Promise) {
    throw new Error("Cannot unwrap Promise synchronously");
  }
  return value as UnwrapPromise<T>;
}

// 6. Combined — overloads + generic (advanced)
interface SelectOptions<T> {
  single: boolean;
  transform?: (item: T) => unknown;
}

function select<T>(items: T[], options: { single: true }): T | undefined;
function select<T>(items: T[], options: { single: false }): T[];
function select<T>(items: T[], options: { single: boolean }): T | T[] | undefined {
  return options.single ? items[0] : items;
}

const single = select([1, 2, 3], { single: true });  // number | undefined
const all = select([1, 2, 3], { single: false });    // number[]
```

</details>

---

## Real-World Use Cases

### Nazariya

Conditional types'ning amaliy qo'llanilishi — library/framework type safety, API type generation, deep object manipulation, va complex data transformation. Real-world'da conditional types ko'pincha mapped types, template literal types, va `infer` bilan birga ishlatiladi.

**Eng keng tarqalgan use case'lar:**

1. **Type-safe API client** — endpoint va method'ga qarab request/response type
2. **Deep path type** — dot notation bilan nested property type olish
3. **Discriminated union narrowing** — `Extract<T, { type: "..." }>`
4. **Utility type'lar** — `Pick`, `Omit`, `Record`, `Partial`, etc.
5. **State management** — Redux action → payload type mapping

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator conditional type'ni evaluate qilganda **type instantiation** jarayoni ishlaydi. Har conditional chaqiriqda yangi type object yaratiladi. Real-world pattern'larda chuqur recursion va nested conditional'lar bu jarayonni sezilarli sekinlashtirishi mumkin.

**Type caching:** Kompilator bir xil argument'lar bilan ikkinchi marta bir xil conditional type'ni hisoblamaydi — cache'dan oladi. Bu katta loyihalarda muhim optimization. Lekin har chaqiriqda yangi argument kombinatsiyasi bo'lsa, cache miss va qayta evaluation.

**Lazy evaluation:** Generic kontekstda conditional type **deferred** holatda qoladi — faqat concrete type bilan instantiate bo'lganda resolve bo'ladi. Bu pattern library kodda ko'p uchraydi:

```typescript
function fetchData<T>(url: string): Promise<ApiResult<T>> {
  // ApiResult<T> deferred — T aniq bo'lmaguncha evaluate bo'lmaydi
  return fetch(url).then(r => r.json());
}

// Call site:
const user = fetchData<User>("/api/user"); // ApiResult<User> resolve bo'ladi
```

**Performance monitoring:** TypeScript `tsc --generateTrace <dir>` flag'i bilan compilation trace'ni yozadi. Bu file'ni Chrome DevTools'ning Performance tab'ida ochib, qaysi type'lar eng ko'p vaqt olganini ko'rish mumkin. Real-world loyihalarda:

```
tsc --generateTrace ./trace-output
# trace.json va types.json fayllar yaratiladi
```

Agar bitta conditional type ko'p marta instantiate bo'lsa — bu bottleneck. Yechim'lar: type aliases cache, simpler conditional, fewer recursive calls.

**Trade-off:** Conditional types kuchli, lekin compile time va IDE responsiveness'ga ta'sir qiladi. Library kodda to'g'ri, application kodda oddiyroq pattern'lar afzal (overloads, union types, oddiy generic'lar).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Type-Safe API Client
interface ApiEndpoints {
  "/users": {
    GET: { response: User[]; query: { limit?: number } };
    POST: { response: User; body: { name: string; email: string } };
  };
  "/users/:id": {
    GET: { response: User; params: { id: string } };
    PUT: { response: User; params: { id: string }; body: Partial<User> };
    DELETE: { response: void; params: { id: string } };
  };
}

interface User {
  id: string;
  name: string;
  email: string;
}

// Conditional type'lar bilan type extraction
type EndpointConfig<
  Path extends keyof ApiEndpoints,
  Method extends keyof ApiEndpoints[Path]
> = ApiEndpoints[Path][Method];

type ResponseType<
  Path extends keyof ApiEndpoints,
  Method extends keyof ApiEndpoints[Path]
> = EndpointConfig<Path, Method> extends { response: infer R } ? R : never;

type HasBody<Config> = Config extends { body: infer B } ? B : never;

async function api<
  Path extends keyof ApiEndpoints,
  Method extends keyof ApiEndpoints[Path]
>(
  path: Path,
  method: Method,
  body?: HasBody<ApiEndpoints[Path][Method]>
): Promise<ResponseType<Path, Method>> {
  const response = await fetch(path as string, {
    method: method as string,
    body: body ? JSON.stringify(body) : undefined,
  });
  return response.json();
}

// Type-safe usage
// const users = await api("/users", "GET");              // User[]
// const user = await api("/users", "POST", { name: "Ali", email: "a@b.com" }); // User

// 2. Deep Path Type — dot notation
type DeepGet<T, Path extends string> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? DeepGet<T[Key], Rest>
      : never
    : Path extends keyof T
      ? T[Path]
      : never;

interface AppState {
  user: {
    profile: {
      name: string;
      age: number;
      address: {
        city: string;
        country: string;
      };
    };
    settings: {
      theme: "light" | "dark";
    };
  };
}

type A = DeepGet<AppState, "user.profile.name">;         // string
type B = DeepGet<AppState, "user.profile.address.city">; // string
type C = DeepGet<AppState, "user.settings.theme">;       // "light" | "dark"
type D = DeepGet<AppState, "user.nonexistent">;           // never

// 3. State Management (Redux-style)
type Action =
  | { type: "SET_USER"; payload: User }
  | { type: "SET_LOADING"; payload: boolean }
  | { type: "SET_ERROR"; payload: string | null };

type ActionPayload<T extends Action["type"]> =
  Extract<Action, { type: T }>["payload"];

type UserPayload = ActionPayload<"SET_USER">;       // User
type LoadingPayload = ActionPayload<"SET_LOADING">; // boolean
type ErrorPayload = ActionPayload<"SET_ERROR">;     // string | null

function dispatch<T extends Action["type"]>(
  type: T,
  payload: ActionPayload<T>
): void {
  // dispatch logic
}

dispatch("SET_USER", { id: "1", name: "Ali", email: "ali@test.com" }); // ✅
dispatch("SET_LOADING", true);                                          // ✅
// dispatch("SET_LOADING", "invalid"); // ❌

// 4. Property filter
type StringProperties<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type UserStrings = StringProperties<User>;
// { id: string; name: string; email: string }

// 5. Function return type based on config
type ConfigResult<T> =
  T extends { async: true } ? Promise<string> : string;

function processConfig<T extends { async: boolean }>(config: T): ConfigResult<T> {
  if (config.async) {
    return Promise.resolve("async result") as ConfigResult<T>;
  }
  return "sync result" as ConfigResult<T>;
}

const sync = processConfig({ async: false });   // string
const async = processConfig({ async: true });   // Promise<string>
```

</details>

---

## Conditional Types Limitations

### Nazariya

Conditional types juda qudratli, lekin cheklovlari ham bor. Bu cheklovlarni bilish — noto'g'ri ishlatishdan va kompilator muammolaridan qochish uchun muhim.

**Asosiy cheklovlar:**

1. **Deferred conditional types** — generic funksiya ichida narrowing ishlamaydi
2. **Recursion depth limit** — cheksiz recursion'ga ruxsat yo'q
3. **Performance** — katta union + complex conditional = sekin compilation
4. **Structural only** — `extends` structural, nominal type checking yo'q
5. **Implementation narrowing** — function body'da conditional return type bilan type assertion kerak

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript cheklovlari:**

Kompilator recursive conditional type'larda cheklangan depth limit'ga ega — bu compile-time stack overflow'ni oldini olish uchun. Limit'ga yetganda `Type instantiation is excessively deep and possibly infinite` xatosi chiqadi. Taxminiy qiymat'lar:

- **Non-tail-recursive:** cheklangan depth (~50 darajagacha)
- **Tail-recursive (TS 4.5+):** kengaytirilgan depth (~1000 darajagacha)

Aniq raqamlar compiler versiyasi va optimizatsiya'lariga bog'liq — o'zgarishi mumkin.

**Total type instantiation budget:** Kompilator butun compilation uchun cheklangan total instantiation count'ga ega. Bu limit aniq documented emas — lekin katta loyihalarda (distributive conditional + katta union) yetib bo'lishi mumkin. Belgi — compilation sekinlashishi yoki `--generateTrace` bilan ko'plab instantiation'lar.

**Union member limit:** Kompilator distributive conditional'da eksponensial instantiation'dan qochish uchun union member count'ga cheklov qo'ygan. Juda katta union'lar (masalan, 10,000+ member) kompilator'ni sekinlashtiradi yoki error beradi.

**Profiling:** `tsc --generateTrace <dir>` flag'i compilation trace'ni yozadi:

```
tsc --generateTrace ./trace-output
```

- `trace.json` — Chrome DevTools Performance tab'ida ochiladi
- `types.json` — barcha instantiate bo'lgan type'lar ro'yxati

Trace'da `checkSourceFile`, `resolveName`, `getContextualType` kabi event'larni kuzating. Agar bitta type minglab marta instantiate bo'lsa — bu bottleneck va refactoring kerak.

**Deferred narrowing muammo:** Generic funksiya ichida conditional return type'ni runtime check bilan resolve qilib bo'lmaydi. Kompilator `T`'ni bilmaydi, va conditional type faqat `T` aniq bo'lganda evaluate bo'ladi. Runtime'da `typeof value === "string"` ishlaydi, lekin TypeScript type system'ga ta'sir qilmaydi — bu sof runtime narrowing.

**Workaround'lar:**

1. **Function overloads** — har branch alohida signature
2. **Type assertion** — `as T extends string ? X : Y` (kamroq xavfsiz)
3. **Return discriminated union** — generic'dan voz kechib, oddiy union

**Structural vs nominal:** `extends` structural typing asosida ishlaydi — shape tekshiruvi. Nominal typing (class identity) uchun `extends` ishlamaydi. Branded type'lar bilan nominal typing'ni simulate qilish mumkin, lekin conditional type'da cheklov bor.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Deferred conditional narrowing — muammo
function example<T>(value: T): T extends string ? "yes" : "no" {
  if (typeof value === "string") {
    // ❌ Error: Type '"yes"' is not assignable to type 'T extends string ? "yes" : "no"'
    // return "yes";

    // ✅ Type assertion (kamroq xavfsiz)
    return "yes" as T extends string ? "yes" : "no";
  }
  return "no" as T extends string ? "yes" : "no";
}

// 2. Overload yechimi — yaxshiroq
function exampleOverload(value: string): "yes";
function exampleOverload(value: unknown): "no";
function exampleOverload(value: unknown): "yes" | "no" {
  return typeof value === "string" ? "yes" : "no";
}

// 3. Recursion depth limit
type DeepNested = {
  a: { b: { c: { d: { e: { f: { g: { h: { i: { j: string } } } } } } } } }
};

type ReadonlyDeep = DeepReadonly<DeepNested>; // OK, chuqur emas

// Juda chuqur bo'lsa — kompilator xato:
// type TooDeep = DeepReadonly<VeryVeryDeeplyNested>; // ❌ excessively deep

// 4. Depth-limited workaround
type DeepReadonlyLimited<T, Depth extends unknown[] = []> =
  Depth["length"] extends 10
    ? T
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonlyLimited<T[K], [...Depth, unknown]> }
      : T;

// 5. Performance — katta union
type BigUnion = "a" | "b" | "c" | "d" | "e" | "f" | "g";

type ComplexConditional<T> = T extends string
  ? { value: T; upper: Uppercase<T> }
  : never;

// Har member uchun alohida evaluation
type Result = ComplexConditional<BigUnion>;
// { value: "a"; upper: "A" } | { value: "b"; upper: "B" } | ...

// 6. Structural extends — not nominal
type Brand<T, B> = T & { __brand: B };
type USD = Brand<number, "USD">;
type EUR = Brand<number, "EUR">;

type Test = USD extends EUR ? true : false;
// false — chunki brand property farqli
// Lekin bu structural check, nominal emas

// 7. Extends vs subtype
type Animal = { name: string };
type Dog = { name: string; breed: string };

type T1 = Dog extends Animal ? true : false;  // true — Dog has all Animal props
type T2 = Animal extends Dog ? true : false;  // false — Animal missing breed

// 8. Deferred type + function
function identity<T>(value: T): T {
  return value;
}

function process<T extends string | number>(
  value: T,
  fn: (v: T) => T
): T extends string ? string : number {
  const processed = fn(value);
  return processed as T extends string ? string : number;
}
```

</details>

---

## Edge Cases va Gotchas

### 1. `never` Empty Union Distribute

`never` — empty union. Distributive conditional type'da `never` hech qanday member'ga ega emas, shuning uchun distribution natijasi ham `never`:

```typescript
type Check<T> = T extends string ? "yes" : "no";

type A = Check<never>;
// Kutilgan: "no" (never string emas)
// Haqiqiy: never (empty union distribute)
```

**Nima uchun:** Kompilator `never`'ni union sifatida split qiladi — lekin empty union'da hech qanday element yo'q. Distribution natijasi ham empty union = `never`.

**`IsNever` check:**

```typescript
// ❌ Distributive — ishlamaydi
type IsNeverBad<T> = T extends never ? true : false;
type B = IsNeverBad<never>; // never (true emas)

// ✅ Non-distributive — to'g'ri
type IsNever<T> = [T] extends [never] ? true : false;
type C = IsNever<never>;  // true
type D = IsNever<string>; // false
```

Non-distributive `[T]` wrap bilan `never` empty union sifatida ko'rilmaydi — butun `[never]` tuple `[never]`'ga extends mi degan savolga aylanadi.

### 2. `any` Ikki Branch'ga Ham Tushadi

`any` conditional type'da noodatiy xatti-harakatga ega. `any extends U` har doim `true` va `false` branch'larning ikkalasini ham tanlaydi:

```typescript
type Check<T> = T extends string ? "yes" : "no";

type A = Check<any>; // "yes" | "no" (ikkala branch!)
```

**Sabab:** `any` — "istalgan type" ma'nosini beradi. Kompilator `any extends string` shartini aniq tekshira olmaydi — chunki `any` bir vaqtda ham string bo'lishi mumkin, ham emas. Shuning uchun kompilator pessimistik yondashadi va ikkala branch'ni qo'shadi.

**`IsAny` detection — famous trick:**

```typescript
type IsAny<T> = 0 extends 1 & T ? true : false;

// Ishlashi:
// 1 & any = any → 0 extends any = true
// 1 & string = never → 0 extends never = false
// 1 & unknown = 1 → 0 extends 1 = false

type Y1 = IsAny<any>;     // true
type Y2 = IsAny<string>;   // false
type Y3 = IsAny<unknown>;  // false
type Y4 = IsAny<never>;    // false
```

### 3. Deferred Conditional + Runtime Narrowing

Generic funksiya ichida conditional return type'ni runtime `typeof` check bilan resolve qilib bo'lmaydi. Kompilator generic `T`'ni bilmaydi, conditional type deferred holatda qoladi:

```typescript
function process<T extends string | number>(
  value: T
): T extends string ? string[] : number {
  if (typeof value === "string") {
    return value.split(""); // ❌ Error
    // TS: T extends string ? string[] : number
    // Kompilator T'ni bilmaydi, string[] ni T extends ... ga assign qila olmaydi
  }
  return value * 2; // ❌ Error (same reason)
}
```

**Yechim'lar:**

1. **Function overloads** — har branch alohida signature, body type-safe
2. **Type assertion** — `as T extends string ? X : Y` (kamroq xavfsiz)
3. **Return union'ni oddiyroq qilish** — `string[] | number` (aniq type'dan voz kechish)

**Nima uchun TS shunday:** Compile-time type system runtime check'larni tushunmaydi. `typeof value === "string"` runtime statement, type system'ga ta'sir qilmaydi. Generic kontekstda `T` hali aniq emas — kompilator conditional'ni resolve qila olmaydi.

### 4. `boolean` Distributive — `true | false`

TypeScript'da `boolean = true | false`. Conditional type'da `boolean`'ga qo'llanganda, `true` va `false` alohida tekshiriladi:

```typescript
type Check<T> = T extends true ? "yes" : "no";

type B = Check<boolean>;
// boolean distributive qilinadi:
// = Check<true> | Check<false>
// = "yes" | "no"
// ❗ Bu "yes" emas
```

**Sabab:** `boolean` — aslida union `true | false`. Distributive conditional'da har member alohida evaluate bo'ladi. Bu ko'p developer'ni chalkashtiradi — `boolean` monolithic type emas.

**Yechim — non-distributive wrap:**

```typescript
type CheckNonDist<T> = [T] extends [true] ? "yes" : "no";

type C = CheckNonDist<boolean>;
// [boolean] extends [true]? → false
// = "no" ✅ (butun union tekshiriladi)
```

Bu nuance'ni bilish — `boolean`, `true`, `false` bilan ishlaganda muhim. Distributive bo'lish kerak yo'q bo'lsa, `[T] extends [true]` yoki `[T] extends [false]` ishlating.

### 5. Function Wrap Contravariance Surprise

Non-distributive qilish uchun function wrap `((x: T) => void) extends ((x: U) => void)` ishlatilsa, parameter position contravariant — natija kutilmagan bo'lishi mumkin:

```typescript
type FnDist<T> = ((x: T) => void) extends ((x: string) => void) ? "yes" : "no";

type A = FnDist<string>;         // "yes"
type B = FnDist<string | number>; // "yes" ❗ (kutilmagan!)
// Sabab contravariance: (x: string | number) => void extends (x: string) => void
// Keng parameter type tor parameter type'ga assign bo'ladi (contravariant)
```

**Nima uchun `"yes"`:** Function parameter position contravariant. Keng type (`string | number`) tor type (`string`) expected joyga mos keladi — chunki `(x: string) => void` funksiyasi hatto `string | number` bilan ham chaqirilishi mumkin (JS runtime qabul qiladi).

**Yechim — tuple wrap (ishonchli):**

```typescript
type TupleDist<T> = [T] extends [string] ? "yes" : "no";

type X = TupleDist<string>;         // "yes"
type Y = TupleDist<string | number>; // "no" (butun union)
```

Tuple wrap parameter position'siz — to'g'ridan-to'g'ri structural check. Variance effekti yo'q, natija bashorat qilinadi.

**Tavsiya:** Non-distributive uchun har doim **tuple wrap** (`[T]`) ishlatish. Function wrap nozik va ko'p joylarda kutilmagan natija beradi.

---

## Common Mistakes

### ❌ Xato 1: Deferred conditional'da runtime narrowing kutish

```typescript
function example<T extends string | number>(
  value: T
): T extends string ? string : number {
  if (typeof value === "string") {
    return value.toUpperCase(); // ❌ Error
  }
  return value * 2; // ❌ Error
}
```

**✅ To'g'ri usullar:**

```typescript
// 1. Function overloads
function exampleFn(value: string): string;
function exampleFn(value: number): number;
function exampleFn(value: string | number): string | number {
  if (typeof value === "string") return value.toUpperCase();
  return value * 2;
}

// 2. Type assertion (kamroq xavfsiz)
function exampleAssert<T extends string | number>(
  value: T
): T extends string ? string : number {
  if (typeof value === "string") {
    return value.toUpperCase() as T extends string ? string : number;
  }
  return ((value as number) * 2) as T extends string ? string : number;
}
```

**Nima uchun:** Generic funksiya ichida kompilator `T`'ni bilmaydi. Conditional type deferred, runtime narrowing type system'ga ta'sir qilmaydi.

---

### ❌ Xato 2: Recursive type'da base case unutish

```typescript
// ❌ Base case yo'q — excessively deep
type InfiniteFlatten<T> =
  T extends (infer U)[]
    ? InfiniteFlatten<U>  // Recursive, base case qanday yetadi?
    : T;
```

**✅ To'g'ri usul:**

```typescript
// Base case bor — agar T array emas, T'ni qaytaradi
type Flatten<T> =
  T extends (infer U)[]
    ? U extends unknown[]
      ? Flatten<U>          // U array → recursive
      : U                   // U array emas → base case
    : T;                    // T array emas → base case

type A = Flatten<number[][]>;      // number
type B = Flatten<string[][][]>;     // string
type C = Flatten<boolean>;          // boolean
```

**Nima uchun:** Recursive type'da base case bo'lmasa, kompilator infinite loop'ga tushadi va "excessively deep" xatosi chiqadi. Har recursive type'da `T extends ... ? recurse : basecase` pattern'i shart.

---

### ❌ Xato 3: Katta union + complex conditional — performance

```typescript
// ❌ 100+ member union + complex conditional
type BigUnion = /* 100+ literal types */;

type ComplexResult<T> =
  T extends string ? { value: T; upper: Uppercase<T>; lower: Lowercase<T> } :
  never;

type Result = ComplexResult<BigUnion>;
// Har member uchun alohida evaluation → juda sekin
```

**✅ To'g'ri usul:**

```typescript
// 1. Union'ni parcha-parcha tekshirish
type FirstHalf = ComplexResult<"a" | "b" | "c">;
type SecondHalf = ComplexResult<"d" | "e" | "f">;
// Har biri kichik, kombinatsiya oxirida

// 2. Type alias cache
type CachedResult = ComplexResult<BigUnion>;
// Bitta joyda hisoblanadi, qayta chaqiriladi

// 3. Conditional'ni soddalashtirish
type SimpleResult<T> = T extends string ? { value: T } : never;
// Kamroq field, tezroq
```

**Nima uchun:** Katta union + distributive conditional = eksponensial instantiation. Kompilator har member uchun alohida hisoblaydi. Real-world library kodda bu oddiy muammo — profiling va optimization kerak.

---

### ❌ Xato 4: `infer` `false` branch'da ishlatish

```typescript
// ❌ infer faqat true branch'da ishlatiladi
type Wrong<T> = T extends string ? T : infer U; // ❌ Error
// 'infer' declarations are only permitted in the 'extends' clause
```

**✅ To'g'ri usul:**

```typescript
// infer — extends clause'da, true branch'da
type GetArrayElement<T> = T extends (infer U)[] ? U : never;

type A = GetArrayElement<string[]>; // string
type B = GetArrayElement<number>;    // never

// Boshqa infer — boshqa conditional
type GetReturnOrArg<T> =
  T extends (...args: any[]) => infer R ? R :
  T extends { value: infer V } ? V :
  never;
```

**Nima uchun:** `infer` pattern matching operatori — pattern'ni `extends` clause'da belgilaydi va mos kelganda (true branch) type'ni ishlatadi. False branch'da pattern mos kelmagan, shuning uchun `infer U` aniq qiymat yo'q.

---

### ❌ Xato 5: `extends` structural, nominal emas

```typescript
type UserId = string & { __brand: "UserId" };
type ProductId = string & { __brand: "ProductId" };

type CheckUserId<T> = T extends UserId ? true : false;

type A = CheckUserId<string>;     // false — brand yo'q
type B = CheckUserId<UserId>;      // true
type C = CheckUserId<ProductId>;   // false — boshqa brand

// Lekin reverse:
type CheckString<T> = T extends string ? true : false;
type D = CheckString<UserId>;     // true ✅
// UserId structurally string, brand qo'shilgan
```

**✅ Nominal check (branded):**

```typescript
type StrictUserId<T> =
  T extends UserId
    ? T extends { __brand: "UserId" }
      ? true
      : false
    : false;

// Yoki branded helper
type IsBranded<T, B> = T extends { __brand: B } ? true : false;

type E = IsBranded<UserId, "UserId">;    // true
type F = IsBranded<ProductId, "UserId">; // false
```

**Nima uchun:** `extends` structural typing asosida ishlaydi — shape tekshiruvi. Brand'lar structural cheklov qo'shadi, lekin nominal typing simulate qilish uchun. Real nominal typing TypeScript'da yo'q — branded type'lar bilan simulate qilinadi.

---

## Amaliy Mashqlar

### Mashq 1: Type-Safe Event Emitter (O'rta)

Conditional type va `infer` bilan type-safe event emitter yozing. Har event nomi uchun payload type compile-time'da aniq bo'lsin.

```typescript
interface EventMap {
  "user:login": { userId: string; timestamp: Date };
  "user:logout": { userId: string };
  "item:add": { itemId: string; quantity: number };
  "item:remove": { itemId: string };
  "error": { message: string; code: number };
}
```

<details>
<summary>Javob</summary>

```typescript
interface EventMap {
  "user:login": { userId: string; timestamp: Date };
  "user:logout": { userId: string };
  "item:add": { itemId: string; quantity: number };
  "item:remove": { itemId: string };
  "error": { message: string; code: number };
}

type EventHandler<K extends keyof EventMap> = (payload: EventMap[K]) => void;

class TypedEmitter {
  private handlers = new Map<string, Array<(...args: any[]) => void>>();

  on<K extends keyof EventMap>(event: K, handler: EventHandler<K>): void {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as (...args: any[]) => void);
    this.handlers.set(event, list);
  }

  off<K extends keyof EventMap>(event: K, handler: EventHandler<K>): void {
    const list = this.handlers.get(event) ?? [];
    const index = list.indexOf(handler as (...args: any[]) => void);
    if (index >= 0) list.splice(index, 1);
  }

  emit<K extends keyof EventMap>(event: K, payload: EventMap[K]): void {
    const list = this.handlers.get(event) ?? [];
    list.forEach(h => h(payload));
  }
}

const emitter = new TypedEmitter();

emitter.on("user:login", (payload) => {
  // payload: { userId: string; timestamp: Date } — avtomatik
  console.log(payload.userId, payload.timestamp);
});

emitter.emit("user:login", { userId: "123", timestamp: new Date() }); // ✅
// emitter.emit("user:login", { wrong: true }); // ❌ Compile error
```

**Tushuntirish:**

- `EventMap` — event nomi → payload type mapping
- `EventHandler<K>` — indexed access bilan payload extraction
- `K extends keyof EventMap` — har method chaqiriqda K aniq bo'ladi
- `Function` o'rniga `(...args: any[]) => void` callable signature

</details>

---

### Mashq 2: DeepGet va DeepSet (O'rta)

Dot notation bilan nested property type olish va set qilish.

```typescript
interface State {
  user: {
    name: string;
    address: {
      city: string;
      zip: number;
    };
  };
  settings: {
    theme: "light" | "dark";
  };
}

// type A = DeepGet<State, "user.address.city">;    // string
// type B = DeepSet<State, "user.name", number>;    // State with user.name: number
```

<details>
<summary>Javob</summary>

```typescript
type DeepGet<T, Path extends string> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? DeepGet<T[Key], Rest>
      : never
    : Path extends keyof T
      ? T[Path]
      : never;

type DeepSet<T, Path extends string, Value> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? { [K in keyof T]: K extends Key ? DeepSet<T[K], Rest, Value> : T[K] }
      : never
    : Path extends keyof T
      ? { [K in keyof T]: K extends Path ? Value : T[K] }
      : never;

interface State {
  user: {
    name: string;
    address: {
      city: string;
      zip: number;
    };
  };
  settings: {
    theme: "light" | "dark";
  };
}

// Test
type A = DeepGet<State, "user.address.city">; // string
type B = DeepGet<State, "settings.theme">;    // "light" | "dark"
type C = DeepGet<State, "user.name">;          // string

type D = DeepSet<State, "user.name", number>;
// State with user.name: number

type E = DeepSet<State, "user.address.city", number>;
// State with user.address.city: number
```

**Tushuntirish:**

- `DeepGet` — path'ni `.` bo'yicha recursive split, har bosqichda `keyof T` tekshiruvi
- `DeepSet` — mapped type bilan original structure saqlanib, target property type o'zgartiriladi
- Base case: path'da `.` yo'q → to'g'ridan-to'g'ri property access/set
- Template literal + `infer` + recursion — TS'ning eng kuchli pattern'lari

</details>

---

### Mashq 3: Utility Type'larni Implement Qilish (O'rta)

Built-in utility type'larni conditional type + `infer` bilan o'zingiz yozing:

1. `MyReturnType<T>`
2. `MyParameters<T>`
3. `MyAwaited<T>`
4. `MyExclude<T, U>`
5. `MyExtract<T, U>`
6. `MyNonNullable<T>`

<details>
<summary>Javob</summary>

```typescript
// 1. ReturnType
type MyReturnType<T extends (...args: any[]) => any> =
  T extends (...args: any[]) => infer R ? R : any;

// 2. Parameters
type MyParameters<T extends (...args: any[]) => any> =
  T extends (...args: infer P) => any ? P : never;

// 3. Awaited (recursive)
type MyAwaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any }
    ? F extends (value: infer V, ...args: infer _) => any
      ? MyAwaited<V>
      : never
    : T;

// 4. Exclude (distributive)
type MyExclude<T, U> = T extends U ? never : T;

// 5. Extract (distributive)
type MyExtract<T, U> = T extends U ? T : never;

// 6. NonNullable
type MyNonNullable<T> = T extends null | undefined ? never : T;

// Test
type A = MyReturnType<() => string>;                          // string
type B = MyParameters<(a: string, b: number) => void>;       // [a: string, b: number]
type C = MyAwaited<Promise<Promise<string>>>;                // string
type D = MyExclude<"a" | "b" | "c", "a">;                   // "b" | "c"
type E = MyExtract<string | number | boolean, number | boolean>; // number | boolean
type F = MyNonNullable<string | null | undefined>;           // string
```

**Tushuntirish:**

- `ReturnType` — `infer R` return pozitsiyasida
- `Parameters` — `infer P` rest parameter pozitsiyasida → tuple
- `Awaited` — recursive: Promise ichidagi `then` callback'dan value olinadi
- `Exclude`/`Extract` — distributive: union har member alohida tekshiriladi
- `NonNullable` — null/undefined olib tashlash

</details>

---

### Mashq 4: Conditional Type Challenges (O'rta)

Output'ni oldindan bashorat qilib, keyin tekshiring:

```typescript
type A = string extends string | number ? "yes" : "no";
type B = string | number extends string ? "yes" : "no";
type C = never extends string ? "yes" : "no";

type IsNever<T> = T extends never ? true : false;
type D = IsNever<never>;

type Check<T> = T extends string ? true : false;
type E = Check<any>;
type F = Check<never>;
type G = Check<boolean>;

type Dist<T> = T extends string ? T[] : never;
type H = Dist<"a" | "b" | 1>;

type NonDist<T> = [T] extends [string] ? "yes" : "no";
type I = NonDist<"a" | "b">;
type J = NonDist<"a" | 1>;
```

<details>
<summary>Javob</summary>

```typescript
type A = string extends string | number ? "yes" : "no";
// "yes" — string assignable to string | number

type B = string | number extends string ? "yes" : "no";
// "no" — string | number assignable to string? ❌
// Bu distributive EMAS — string | number literal union, type parameter emas

type C = never extends string ? "yes" : "no";
// "yes" — never extends EVERYTHING
// Bu distributive emas — never literal

type IsNever<T> = T extends never ? true : false;
type D = IsNever<never>;
// never ❗ — empty union distribute → nothing → never

type Check<T> = T extends string ? true : false;
type E = Check<any>;
// boolean (true | false) — any ikki branch

type F = Check<never>;
// never — empty union distribute

type G = Check<boolean>;
// boolean distributive: Check<true> | Check<false> = false | false = false
// Wait: true extends string? false. false extends string? false.
// = false ❗ (boolean = true | false, ikkalasi string emas)

type Dist<T> = T extends string ? T[] : never;
type H = Dist<"a" | "b" | 1>;
// "a"[] | "b"[] (1 never, filtered)

type NonDist<T> = [T] extends [string] ? "yes" : "no";
type I = NonDist<"a" | "b">;
// "yes" — ["a" | "b"] extends [string]? ✅

type J = NonDist<"a" | 1>;
// "no" — ["a" | 1] extends [string]? ❌ (1 string emas)
```

</details>

---

### Mashq 5: Template Literal Parser (Qiyin)

`infer extends` (TS 4.8+) bilan URL path'dan parameter'larni extract qiluvchi type yozing.

```typescript
// type P1 = ExtractParams<"/users/:id">;                    // { id: string }
// type P2 = ExtractParams<"/users/:id/posts/:postId">;       // { id: string; postId: string }
// type P3 = ExtractParams<"/static">;                         // {}
```

<details>
<summary>Javob</summary>

```typescript
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & ExtractParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

// Test
type P1 = ExtractParams<"/users/:id">;
// { id: string }

type P2 = ExtractParams<"/users/:id/posts/:postId">;
// { id: string } & { postId: string }
// = { id: string; postId: string }

type P3 = ExtractParams<"/users/:id/posts/:postId/comments/:commentId">;
// { id: string; postId: string; commentId: string }

type P4 = ExtractParams<"/static">;
// {}

// Type-safe route handler
function handleRoute<Path extends string>(
  path: Path,
  params: ExtractParams<Path>
): void {
  console.log(path, params);
}

handleRoute("/users/:id", { id: "42" });
handleRoute("/users/:id/posts/:postId", { id: "1", postId: "2" });
// handleRoute("/users/:id", {}); // ❌ id majburiy
```

**Tushuntirish:**

- Birinchi pattern: `${string}:${infer Param}/${infer Rest}` — parameter'dan keyin `/` bor (ko'proq parameter'lar)
- Ikkinchi pattern: `${string}:${infer Param}` — oxirgi parameter (keyin `/` yo'q)
- Base case: parameter yo'q → `{}`
- `{ [K in Param]: string }` — parameter nom'idan object property yaratish
- `& ExtractParams<...>` — intersection bilan keyingi parameter'larni qo'shish

Bu pattern React Router, Next.js, Express.js kabi framework'larda ishlatiladi.

</details>

---

## Xulosa

Bu bo'limda conditional types'ning chuqur mexanizmlari o'rganildi:

**Asosiy tushunchalar:**

- **Syntax va semantics** — `T extends U ? X : Y`, structural assignability check, eager va deferred evaluation
- **`infer` keyword chuqur** — barcha pozitsiyalar (covariant union, contravariant intersection), tuple element extraction, constructor/instance type, constrained `infer` (TS 4.7+), template literal parsing (TS 4.8+)
- **Distributive conditional types** — naked type parameter + union, `Exclude`/`Extract`/`NonNullable` asosi, `never` empty union, `any` ikki branch
- **Non-distributive** — `[T] extends [U]` tuple wrap (tavsiya), `IsNever`, `StrictEquals`, `IsUnion`
- **Recursive conditional types** — base case kerak, tail-call optimization (TS 4.5+), accumulator pattern
- **Conditional types vs function overloads** — pragmatic guide (oddiy/union/ko'p holat)
- **Real-world use cases** — API client, DeepGet, state management
- **Limitations** — deferred narrowing, depth limit, performance, structural extends

**Umumiy takeaway'lar:**

1. **Deferred + narrowing cheklovi** — generic funksiya body'da conditional return type'ni `typeof` bilan resolve qilib bo'lmaydi. Overload yoki assertion yechim.
2. **`never` va `any` nozik** — `never` empty union (distribute'da yo'qoladi), `any` ikki branch'ga tushadi.
3. **Distributive faqat naked parameter** — `T` to'g'ridan-to'g'ri, wrapper yo'q. `[T]`, `T[]`, `Box<T>` distribution'ni to'xtatadi.
4. **Tuple wrap ishonchli** — non-distributive uchun `[T] extends [U]` afzal. Function wrap contravariance surprise beradi.
5. **Recursive base case shart** — infinite loop'dan qochish.
6. **Tail-recursion accumulator pattern** — chuqur recursion uchun (TS 4.5+).
7. **Overloaded `ReturnType`** — oxirgi overload signature'ni oladi, union emas. Bu nuance'ni bilish muhim.
8. **`boolean` distributive** — `true | false` alohida tekshiriladi.

**Cross-references:**

- **[06-type-narrowing.md](06-type-narrowing.md)** — `typeof`, `instanceof`, user-defined type guards
- **[07-functions.md](07-functions.md)** — Function overloads, variance, contextual typing
- **[08-generics.md](08-generics.md)** — Generic asos, `keyof`, index access
- **[09-advanced-generics.md](09-advanced-generics.md)** — Conditional types intro, infer basics
- **[13-mapped-types.md](13-mapped-types.md)** — Mapped types chuqur, key remapping
- **[14-template-literal-types.md](14-template-literal-types.md)** — Template literal chuqur, string parsing
- **[15-utility-types.md](15-utility-types.md)** — Built-in utility'lar (`Pick`, `Omit`, `ReturnType`, `Parameters`)
- **[25-type-compatibility.md](25-type-compatibility.md)** — Variance chuqur, covariance/contravariance

---

**Keyingi bo'lim:** [13-mapped-types.md](13-mapped-types.md) — Mapped Types chuqur: `{ [K in keyof T]: T[K] }` syntax, property modifier'lar (`+?`, `-?`, `+readonly`, `-readonly`), key remapping (`as`), homomorphic mapped types, va custom utility type'lar.
