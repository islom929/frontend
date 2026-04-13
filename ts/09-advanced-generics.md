# Bo'lim 9: Advanced Generics

> Advanced Generics — TypeScript'ning type-level programming imkoniyatlari: conditional types bilan type darajasida shartli logic, `infer` bilan type extraction, mapped types bilan type transformation, template literal types bilan string-level typing, recursive types va variadic tuples. Bu tushunchalar TypeScript'da murakkab, lekin type-safe abstraction'lar yaratish uchun asos bo'ladi.
>
> Bu bo'lim — advanced type system'ning **sayohat xaritasi**. Har mavzu alohida bo'lim'larda (12, 13, 14) chuqur ko'riladi; bu yerda esa ularning umumiy kontekstdagi ishlatilishini, o'zaro munosabatini, va real-world pattern'larda qanday birlashishini o'rganasiz.

---

## Mundarija

- [Conditional Types](#conditional-types)
- [infer Keyword](#infer-keyword)
- [Distributive Conditional Types](#distributive-conditional-types)
- [Non-Distributive Conditional Types](#non-distributive-conditional-types)
- [Mapped Types](#mapped-types)
- [Template Literal Types](#template-literal-types)
- [Recursive Types](#recursive-types)
- [Variadic Tuple Types](#variadic-tuple-types)
- [Generic Constraints Best Practices](#generic-constraints-best-practices)
- [Real-World Advanced Patterns](#real-world-advanced-patterns)
- [Higher-Kinded Types](#higher-kinded-types)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Conditional Types

### Nazariya

Conditional type — type darajasida `if/else` logic. Sintaksis JavaScript ternary operatoriga (`? :`) o'xshaydi, lekin bu **type'lar** bilan ishlaydi, **qiymatlar** bilan emas:

```
T extends U ? TrueType : FalseType
```

**Ma'nosi:** "Agar T type U type'ga assignable bo'lsa — `TrueType`, aks holda `FalseType`."

Bu yerda `extends` — class inheritance emas, **structural subtyping** tekshiruvi. `T extends U` degani "T'ning barcha property'lari U'da bormi?" (yoki T U'ning subtype'imi?). Agar ha — `TrueType`.

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;    // true
type B = IsString<number>;    // false
type C = IsString<"hello">;   // true — "hello" string subtype
```

Conditional type'lar nima uchun kerak:

1. **Type dispatch** — type'ga qarab turli natija qaytarish
2. **Utility types** — `Exclude`, `Extract`, `NonNullable`, `Pick`, `ReturnType` barchasi conditional asosida
3. **Type-level programming** — string parsing, arithmetic, validation

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator conditional type'ni ikki xil usulda evaluate qiladi:

- **Resolved (darhol hisoblangan)** — `T` concrete type bo'lganda. Kompilator `T extends U` shartini darhol tekshiradi va true yoki false branch'ni tanlaydi. Natija — bitta aniq type.
- **Deferred (keyinga qoldirilgan)** — `T` hali noma'lum type parameter bo'lganda (generic funksiya ichida). Kompilator bu conditional type'ni resolve qilmaydi, keyinroq — `T` concrete type bilan instantiate bo'lganda hisoblaydi.

```
Conditional type evaluation:

IsString<string>  (T = string, concrete)
  ├─ T extends U ? → string extends string ? → true
  └─ Natija: true  (darhol resolved)

function f<T>(x: T): T extends string ? "yes" : "no"
  ├─ T noma'lum → DEFER (keyinga qoldiriladi)
  │
  f("hello")  ← T = string instantiate
  ├─ Endi hisoblash mumkin: string extends string → true
  └─ Natija: "yes"
```

**Caching:** TypeScript bir xil type argument'lar bilan ikkinchi marta bir xil conditional type'ni hisoblamaydi — natija cache'dan olinadi. Bu katta kod bazalarda performance uchun muhim.

**`extends` vs class inheritance:** Bu ikkita butunlay farqli ma'no:

- **Class context** — `class Dog extends Animal` — inheritance, prototype chain
- **Conditional context** — `T extends U` — assignability check (subtype question)

Bitta keyword, ikki xil ma'no — kontekstga qarab.

**Runtime'da iz yo'q:** Conditional type compile-time konstruksiyasi. Compiled JavaScript'da `T extends U ? X : Y` hech qanday kod emit qilmaydi — u faqat type checker uchun mavjud.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy IsString check
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;    // true
type B = IsString<number>;    // false
type C = IsString<"hello">;   // true

// 2. Type name olish — turli xil natija
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  T extends (...args: any[]) => any ? "function" :
  T extends unknown[] ? "array" :
  "object";

type T1 = TypeName<string>;      // "string"
type T2 = TypeName<number[]>;    // "array"
type T3 = TypeName<() => void>;  // "function"
type T4 = TypeName<{ a: 1 }>;    // "object"

// 3. Conditional type generic funksiyada
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

// 4. Nullable check
type IsNullable<T> = T extends null | undefined ? true : false;

type N1 = IsNullable<string>;        // false
type N2 = IsNullable<null>;           // true
type N3 = IsNullable<undefined>;      // true
type N4 = IsNullable<string | null>;  // boolean (distributive, pastda)

// 5. Function check
type IsFunction<T> = T extends (...args: any[]) => any ? true : false;

type F1 = IsFunction<() => void>;     // true
type F2 = IsFunction<string>;          // false

// 6. Array element extract
type ElementOf<T> = T extends (infer U)[] ? U : never;

type E1 = ElementOf<string[]>;   // string
type E2 = ElementOf<number[]>;   // number
type E3 = ElementOf<string>;     // never

// 7. Literal type check
type IsLiteral<T> =
  T extends string
    ? string extends T
      ? false      // `string` — literal emas
      : true        // `"hello"` — literal
    : false;

type L1 = IsLiteral<"hello">;  // true
type L2 = IsLiteral<string>;    // false
```

</details>

> **Chuqurroq:** Conditional types haqida to'liq batafsil — [12-conditional-types.md](12-conditional-types.md).

---

## infer Keyword

### Nazariya

`infer` — conditional type ichida type'ni **"olish"** (extract qilish) mexanizmi. `infer U` — "bu joyda qanday type bo'lsa — shu type'ni `U`'ga yoz" degan ma'no.

**Muhim qoidalar:**

1. `infer` faqat conditional type'ning `extends` qismida ishlaydi
2. `infer` bilan yaratilgan type variable faqat **true branch**'da ishlatilishi mumkin
3. `infer` pattern matching orqali ishlaydi — agar T pattern'ga mos kelsa, type olinadi

```typescript
// Array element type'ni olish
type ElementOf<T> = T extends (infer U)[] ? U : never;
//                              ^^^^^^^ pattern — array element type
//                                        ↓ true branch'da U ishlatiladi

type A = ElementOf<string[]>;  // string
type B = ElementOf<number[]>;  // number
type C = ElementOf<boolean>;   // never — boolean array emas
```

**`infer` nima uchun kerak?** Murakkab type'lar ichidan kerakli qismni ajratib olish uchun:

- Funksiya return type'ini olish (`ReturnType<T>`)
- Funksiya parameter type'larini olish (`Parameters<T>`)
- Promise ichidagi type'ni olish (`Awaited<T>`)
- Array element type, tuple element, Map value, Set element

<details>
<summary><strong>Under the Hood</strong></summary>

`infer` ishlash jarayoni — **pattern matching**:

1. Kompilator `T extends SomePattern<infer U>` ko'radi
2. T'ni SomePattern'ga moslashtiradi (structural check)
3. Agar T mos kelsa — `infer U` joylashgan pozitsiyada qanday type bo'lsa, U'ga assign qiladi
4. True branch'da U ishlatilsa — shu infer qilingan type bilan almashtiriladi
5. Agar T mos kelmasa — false branch'ga o'tiladi

```
Pattern matching vizualizatsiya:

T = Promise<string>
Pattern = Promise<infer U>

Moslashtirish:
  Promise<string> ↔ Promise<infer U>
  ✅ Mos keldi — U = string
  → true branch: U = string

T = number
Pattern = Promise<infer U>

Moslashtirish:
  number ↔ Promise<infer U>
  ❌ Mos kelmadi — false branch
```

**`infer` multi-position — variance farqi:** `infer U` bir nechta pozitsiyada ishlatilsa, kompilator natijani quyidagi qoidalar bo'yicha birlashtiradi:

- **Covariant position** (return type, read-only property) — natija **union** bo'ladi
- **Contravariant position** (parameter type) — natija **intersection** bo'ladi

```typescript
// Covariant — return type'larda
type GetReturns<T> = T extends {
  a: () => infer R;
  b: () => infer R;
} ? R : never;

type R1 = GetReturns<{ a: () => string; b: () => number }>;
// string | number — covariant, union

// Contravariant — parameter type'larda
type GetParams<T> = T extends {
  a: (x: infer P) => void;
  b: (x: infer P) => void;
} ? P : never;

type P1 = GetParams<{ a: (x: string) => void; b: (x: number) => void }>;
// string & number = never — contravariant, intersection
// Sabab: "string VA number bo'lgan type" — hech qaysi type bunday emas
```

**`string & number = never` sabab:** `string` va `number` — ikki disjoint (kesishmaydigan) type. `A & B` intersection "A ham, B ham" ma'nosini beradi — lekin hech qaysi qiymat bir vaqtda ham string, ham number bo'la olmaydi. Shuning uchun natija — "mumkin bo'lmagan type" = `never`.

**`infer extends` constrained inference (TS 4.7+):** `infer U extends SomeConstraint` — infer qilingan type ma'lum constraint'ga mos kelishini talab qilish:

```typescript
type FirstStringElement<T> = T extends [infer F extends string, ...any[]] ? F : never;

type F1 = FirstStringElement<["hello", 42, true]>;  // "hello"
type F2 = FirstStringElement<[42, "hello"]>;        // never — birinchi string emas
```

**Compiled JS'da iz yo'q:** `infer` sof compile-time feature. JavaScript output'da hech qanday iz qolmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Array element type
type ElementOf<T> = T extends (infer U)[] ? U : never;

type A = ElementOf<string[]>;               // string
type B = ElementOf<number[]>;               // number
type C = ElementOf<(string | number)[]>;     // string | number
type D = ElementOf<string>;                  // never

// 2. Promise unwrap — simplified Awaited
// MUHIM: Bu o'rganish uchun sodda misol.
// Real `Awaited<T>` (lib.es5.d.ts'da) thenable object'lar va null/undefined'ni ham handle qiladi
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;

type E = MyAwaited<Promise<string>>;             // string
type F = MyAwaited<Promise<Promise<number>>>;    // number (recursive)
type G = MyAwaited<string>;                       // string

// 3. Return type extraction — ReturnType'ning implementation'i
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type H = MyReturnType<() => string>;           // string
type I = MyReturnType<(x: number) => boolean>; // boolean
type J = MyReturnType<string>;                  // never

// 4. Parameters extraction — Parameters'ning implementation'i
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;

type K = MyParameters<(a: string, b: number) => void>; // [a: string, b: number]
type L = MyParameters<() => void>;                      // []

// 5. Multiple infer — bir nechta qism olish
type FunctionParts<T> = T extends (...args: infer P) => infer R
  ? { params: P; returnType: R }
  : never;

type Parts = FunctionParts<(name: string, age: number) => boolean>;
// { params: [name: string, age: number]; returnType: boolean }

// 6. Map value extraction
type MapValueType<T> = T extends Map<any, infer V> ? V : never;

type M1 = MapValueType<Map<string, number>>;  // number
type M2 = MapValueType<Map<string, User>>;     // User

interface User { name: string; }

// 7. Constructor parameter extraction
type ConstructorParams<T> = T extends new (...args: infer P) => any ? P : never;

class Person {
  constructor(public name: string, public age: number) {}
}

type PersonParams = ConstructorParams<typeof Person>;
// [name: string, age: number]

// 8. Tuple first element
type FirstElement<T> = T extends [infer F, ...any[]] ? F : never;

type First1 = FirstElement<[string, number, boolean]>; // string
type First2 = FirstElement<[]>;                         // never

// 9. infer extends (TS 4.7+) — constrained inference
type FirstNumber<T> = T extends [infer F extends number, ...any[]] ? F : never;

type N1 = FirstNumber<[42, "hello"]>;      // 42
type N2 = FirstNumber<["hello", 42]>;       // never — birinchi number emas
```

</details>

---

## Distributive Conditional Types

### Nazariya

Distributive conditional type — conditional type'ga **union type** berilganda, TypeScript uni union'ning **har bir member**'iga alohida qo'llaydi va natijalarni qayta union qiladi. Bu avtomatik sodir bo'ladi — faqat type parameter `T` **naked** (wrapper'siz) bo'lganda.

Matematik tarqatish (distribution) qoidasi bilan bir xil mantiq:

```
f(A | B | C) = f(A) | f(B) | f(C)
```

**Distributive bo'lish shartlari:**

1. Conditional type bo'lishi kerak — `T extends U ? X : Y`
2. Tekshirilayotgan type **naked type parameter** bo'lishi kerak — ya'ni to'g'ridan-to'g'ri `T`, `T[]` yoki `[T]` emas
3. Union type berilishi kerak

```typescript
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// = ToArray<string> | ToArray<number>
// = string[] | number[]
// ❗ (string | number)[] EMAS!
```

Distributive behavior — TypeScript'ning eng muhim built-in utility type'larining asosi:

- `Exclude<T, U>` — union'dan ma'lum type'larni olib tashlash
- `Extract<T, U>` — union'dan ma'lum type'larni tanlash
- `NonNullable<T>` — null va undefined'ni olib tashlash

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator conditional type'ni evaluate qilganda, avval check type'ning **naked type parameter** ekanligini tekshiradi. Agar `T` to'g'ridan-to'g'ri tursa — distribution mexanizmi ishga tushadi va union'ning har bir member'i uchun alohida evaluation qiladi.

```
Distributive jarayon:

type ToArray<T> = T extends any ? T[] : never;

ToArray<string | number>
  ↓
(string | number) ni "distribute" qilamiz:
  - string extends any ? string[] : never → string[]
  - number extends any ? number[] : never → number[]
  ↓
Natija: string[] | number[]
```

**Nima uchun distributive behavior standart?** Distributive behavior `Exclude`, `Extract`, `NonNullable` kabi utility'larni sodda qilish uchun ongli dizayn qaror:

```typescript
// Exclude — oddiy conditional type, lekin distributive
type MyExclude<T, U> = T extends U ? never : T;

type X = MyExclude<"a" | "b" | "c", "a">;
// Distribution:
// "a" extends "a" ? never : "a" → never
// "b" extends "a" ? never : "b" → "b"
// "c" extends "a" ? never : "c" → "c"
// Natija: never | "b" | "c" = "b" | "c"
```

**`never` bilan nozik holat:** `never` — empty union deb qaraladi. `T extends any ? T[] : never` ichida `T = never` bo'lsa, kompilator distribution'ga hech narsa bermaydi va natija `never` bo'ladi:

```typescript
type ToArray<T> = T extends any ? T[] : never;
type N = ToArray<never>; // never (bo'sh union'da nothing to distribute)
```

**`any` distributive:** `T extends any` har doim true — lekin `T` naked parameter bo'lsa, u hali ham distribute qilinadi. Bu pattern `ToArray<T> = T extends any ? T[] : never`'da ishlatiladi — `any` check "har qanday type" ma'nosini beradi, lekin union'ni parchalash uchun ishlatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Union dan array hosil qilish
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// string[] | number[] (distributive!)

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

type Ex2 = MyExtract<"a" | "b" | 1 | 2, string>;
// "a" | "b"

// 4. NonNullable implementation
type MyNonNullable<T> = T extends null | undefined ? never : T;

type N1 = MyNonNullable<string | null | undefined>;
// string

type N2 = MyNonNullable<number | null>;
// number

// 5. Distributive filter — only strings
type OnlyStrings<T> = T extends string ? T : never;

type O1 = OnlyStrings<"a" | "b" | 1 | 2>;
// "a" | "b"

// 6. Flatten nested arrays (simplified)
type Flatten<T> = T extends (infer U)[] ? U : T;

type F1 = Flatten<string[]>;           // string
type F2 = Flatten<(string | number)[]>; // string | number
type F3 = Flatten<number>;              // number

// 7. Distribution bilan kombinatsiya
type Stringify<T> = T extends string ? T : `${T & (string | number | boolean)}`;

type S1 = Stringify<"hello" | 42 | true>;
// "hello" | "42" | "true"
```

</details>

---

## Non-Distributive Conditional Types

### Nazariya

Ba'zan distributive xatti-harakatni **to'xtatish** kerak — ya'ni union type'ni butunlay, bo'laklarga ajratmasdan tekshirish kerak. Buning uchun type parameter'ni **wrapper** ichiga olish kerak — eng oddiy usul `[T]` tuple ichiga olish.

```typescript
// Distributive — har member alohida
type IsStringDist<T> = T extends string ? true : false;
type A = IsStringDist<string | number>;
// = (string extends string ? true : false) | (number extends string ? true : false)
// = true | false = boolean

// Non-distributive — butun union tekshiriladi
type IsStringNonDist<T> = [T] extends [string] ? true : false;
type B = IsStringNonDist<string | number>;
// = false — string | number butunlay string emas
```

**Qachon non-distributive kerak:**

1. **Type equality check** — "T va U aynan bir xil type'mi?"
2. **Union'ni butunlay tekshirish** — "butun union ma'lum type'ga mos keladimi?"
3. **Distribution'ni to'xtatish** — kutilmagan natijalardan qochish

**`[T]` wrapping qanday ishlaydi:** `[T]` — bir elementli tuple. TypeScript uni "naked type parameter" sifatida ko'rmaydi (wrapper ichida), shuning uchun distribution ishlamaydi. Union butunlay tuple ichida qoladi va bir butun sifatida tekshiriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator conditional type'ni evaluate qilganda avval check type'ning **naked type parameter** ekanligini tekshiradi. `[T] extends [string]` yozganda, kompilator `T`'ni endi naked emas deb biladi — u tuple ichida wrapped. Natijada distribution mexanizmi ishlamaydi.

```
Non-distributive evaluation:

type Check<T> = [T] extends [string] ? true : false;
Check<string | number>

1. T = string | number (union sifatida keladi)
2. [T] — tuple wrapper → T endi naked emas
3. Distribution SKIP — union bo'linmaydi
4. [string | number] extends [string] → false
5. Natija: false (bitta natija, union emas)
```

**Muhim nuance:** `[T]` wrapping faqat compile-time'da type checker behavior'ini o'zgartiradi. Runtime'da hech narsa o'zgarmaydi — bu purely type-level trick. Kompilator `[T] extends [U]`'ni internally tuple type'lar orasidagi subtyping sifatida tekshiradi.

**Boshqa wrapper'lar:** `[T]` eng keng tarqalgan, lekin istalgan wrapper ishlaydi:

- `[T] extends [U]` — tuple wrapper (standart)
- `{ t: T } extends { t: U }` — object wrapper (ishlatiladi, lekin og'irroq)
- `T[] extends U[]` — array wrapper (covariance tufayli natija boshqacha)

Amalda hamma **`[T] extends [U]`** pattern'ni ishlatadi — qisqa va aniq.

**Type equality trick:** TypeScript'da "ikki type aynan tengmi?" tekshiruvi non-distributive bilan amalga oshiriladi:

```typescript
type StrictEquals<T, U> =
  [T] extends [U]
    ? [U] extends [T]
      ? true
      : false
    : false;

// Ikki tomonga ham assignability tekshirish — faqat aynan teng bo'lsa true
```

Bu pattern'ning oddiy `T extends U` variantidan farqi — distribution va asymmetry.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Non-distributive check
type IsStringOnly<T> = [T] extends [string] ? true : false;

type A = IsStringOnly<string>;          // true
type B = IsStringOnly<string | number>;  // false — union butunlay string emas
type C = IsStringOnly<"hello">;          // true — literal subtype

// 2. Strict equality check
type StrictEquals<T, U> =
  [T] extends [U]
    ? [U] extends [T]
      ? true
      : false
    : false;

type E1 = StrictEquals<string, string>;          // true
type E2 = StrictEquals<string, number>;           // false
type E3 = StrictEquals<string | number, string>;  // false
type E4 = StrictEquals<string, string | number>;  // false
type E5 = StrictEquals<"a", "a">;                 // true

// 3. ToArray non-distributive versiyasi
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type F = ToArrayNonDist<string | number>;
// (string | number)[] — bitta array, ikkala type element

// Distributive variantda: string[] | number[]
// Non-distributive: (string | number)[]
// Bu ikki type farqli!

// 4. Check if type is never
type IsNever<T> = [T] extends [never] ? true : false;

type N1 = IsNever<never>;  // true
type N2 = IsNever<string>; // false
type N3 = IsNever<any>;    // false

// Distributive variant ishlamaydi:
// type IsNeverBad<T> = T extends never ? true : false;
// IsNeverBad<never> = never (distribution bo'sh union'da)

// 5. Strict union check
type ContainsString<T> = [T] extends [string] ? true : false;

type CS1 = ContainsString<string>;          // true
type CS2 = ContainsString<string | number>; // false — butunlay string emas
type CS3 = ContainsString<number>;          // false

// 6. Non-distributive bilan filter
type WrapIfAny<T> = [T] extends [never]
  ? []
  : [T][0] extends any
    ? [T]
    : never;

type W1 = WrapIfAny<string>;         // [string]
type W2 = WrapIfAny<string | number>; // [string | number]

// 7. Union detection
type IsUnion<T, U = T> =
  [T] extends [never]
    ? false
    : T extends unknown
      ? [U] extends [T]
        ? false
        : true
      : never;

type U1 = IsUnion<string>;          // false
type U2 = IsUnion<string | number>;  // true
type U3 = IsUnion<never>;            // false
```

</details>

---

## Mapped Types

### Nazariya

Mapped type — mavjud object type'ning har bir property'sini **transform** qilish mexanizmi. `{ [K in keyof T]: NewType }` — T'ning barcha key'larini iterate qiladi va har property uchun yangi type yaratadi. Bu `Array.map()`'ning type-level versiyasi.

```
Oddiy map: [1, 2, 3].map(x => x * 2) = [2, 4, 6]
Type map: { a: string; b: number } → { a: boolean; b: boolean }
```

**Asosiy sintaksis:**

```typescript
type Mapped<T> = {
  [K in keyof T]: /* T[K]'dan yangi type */;
};
```

**Property modifier'lar** — mapped type ichida `+?`, `-?`, `+readonly`, `-readonly` qo'shish yoki olib tashlash:

- `+?` yoki `?` — optional qo'shish
- `-?` — optional olib tashlash
- `+readonly` yoki `readonly` — readonly qo'shish
- `-readonly` — readonly olib tashlash

**Key remapping** (TS 4.1+) — `as` clause bilan key'ni qayta nomlash yoki filter qilish.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator mapped type'ni qayta ishlaganda, avval `keyof T` yoki `in` dan keyin kelgan type'ni resolve qilib, **string literal union** hosil qiladi. Keyin bu union'ning har bir member'i uchun yangi property yaratiladi.

```
Mapped type evaluation:

type MyPartial<T> = { [K in keyof T]?: T[K] };
MyPartial<{ name: string; age: number }>

1. keyof T → "name" | "age" (string literal union)
2. Iteration: K = "name" → property: name?: string
3. Iteration: K = "age"  → property: age?: number
4. Natija: { name?: string; age?: number }
```

**Homomorphic vs non-homomorphic mapped type:**

**Homomorphic** — `keyof T` ishlatgandagi mapped type. TypeScript property modifier'larni (readonly, optional) **avtomatik saqlaydi**:

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

type Nullable<T> = { [K in keyof T]: T[K] | null };
type NullableConfig = Nullable<Config>;
// { readonly host: string | null; port?: number | null }
// ✅ readonly va ? saqlandi
```

**Non-homomorphic** — `keyof T`'dan boshqa key source ishlatilgan. Modifier'lar **yo'qoladi**:

```typescript
type ManualNullable = {
  [K in "host" | "port"]: K extends keyof Config ? Config[K] | null : never;
};
// { host: string | null; port: number | null }
// ❌ readonly va ? yo'qoldi
```

Qoida: **doim `keyof T` ishlatish** — modifier'larni saqlash uchun.

**Key remapping `as`:** TS 4.1+ da mapped type ichida key'ni qayta nomlash yoki filter qilish mumkin. `never` remapping — property'ni butunlay olib tashlaydi:

```typescript
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
// never'ga remap qilingan property'lar butunlay olib tashlanadi
```

Bu pattern filter'lash uchun juda kuchli — utility type'larda (masalan, `PickByValue`) ishlatiladi.

**Compile-time operation:** Barcha mapped type transformatsiyalari compile-time'da sodir bo'ladi. Runtime'da oddiy JS object qoladi — hech qanday iz yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Partial implementation
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

interface User {
  name: string;
  age: number;
  email: string;
}

type PartialUser = MyPartial<User>;
// { name?: string; age?: number; email?: string }

// 2. Readonly implementation
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

type ReadonlyUser = MyReadonly<User>;
// { readonly name: string; readonly age: number; readonly email: string }

// 3. Required — optional olib tashlash
type MyRequired<T> = {
  [K in keyof T]-?: T[K];
};

type RequiredUser = MyRequired<MyPartial<User>>;
// { name: string; age: number; email: string }

// 4. Mutable — readonly olib tashlash
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type MutableUser = Mutable<ReadonlyUser>;
// { name: string; age: number; email: string }

// 5. Nullable — har property'ga null qo'shish
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type NullableUser = Nullable<User>;
// { name: string | null; age: number | null; email: string | null }

// 6. Key remapping — prefix qo'shish
type Prefixed<T, Prefix extends string> = {
  [K in keyof T as `${Prefix}${Capitalize<string & K>}`]: T[K];
};

type PrefixedUser = Prefixed<User, "get">;
// { getName: string; getAge: number; getEmail: string }

// 7. Filter properties by type
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type StringProps = OnlyStrings<User>;
// { name: string; email: string } — age (number) filtered

// 8. Pick implementation
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type UserSummary = MyPick<User, "name" | "email">;
// { name: string; email: string }

// 9. Record implementation
type MyRecord<K extends keyof any, V> = {
  [P in K]: V;
};

type StringMap = MyRecord<"a" | "b" | "c", number>;
// { a: number; b: number; c: number }

// 10. Getters generator
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; getEmail: () => string }
```

</details>

> **Chuqurroq:** Mapped types haqida to'liq batafsil — [13-mapped-types.md](13-mapped-types.md).

---

## Template Literal Types

### Nazariya

Template literal type — JavaScript template literal (`` `hello ${name}` ``)'ning type-level versiyasi. String literal type'lar bilan ishlaydi — yangi string pattern type'lar yaratadi. TS 4.1+ da qo'shilgan.

```typescript
type Greeting = `Hello, ${string}`;

const a: Greeting = "Hello, Ali";     // ✅
const b: Greeting = "Hello, World";   // ✅
// const c: Greeting = "Hi, Ali";     // ❌ "Hi,"'dan boshlanadi
```

**Union bilan kombinatsiya — Cartesian product:**

```typescript
type Color = "red" | "green" | "blue";
type Size = "sm" | "md" | "lg";
type ClassName = `${Color}-${Size}`;
// "red-sm" | "red-md" | "red-lg"
// | "green-sm" | "green-md" | "green-lg"
// | "blue-sm" | "blue-md" | "blue-lg"
// 3 × 3 = 9 kombinatsiya
```

**Intrinsic string manipulation types** — TS'da to'rtta built-in:

| Type | Ma'nosi |
|------|---------|
| `Uppercase<S>` | `"hello"` → `"HELLO"` |
| `Lowercase<S>` | `"HELLO"` → `"hello"` |
| `Capitalize<S>` | `"hello"` → `"Hello"` |
| `Uncapitalize<S>` | `"Hello"` → `"hello"` |

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator template literal type'ni compile-time'da to'liq construct qiladi. Har `${X}` placeholder uchun kompilator X'ning type'ini resolve qiladi — agar X string literal union bo'lsa, barcha kombinatsiyalar hosil qilinadi (Cartesian product).

```
Template literal evaluation:

type T = `${Color}-${Size}`;
Color = "red" | "green", Size = "sm" | "lg"

1. Kompilator Cartesian product hisoblaydi:
   "red" × "sm" → "red-sm"
   "red" × "lg" → "red-lg"
   "green" × "sm" → "green-sm"
   "green" × "lg" → "green-lg"
2. Natija: "red-sm" | "red-lg" | "green-sm" | "green-lg"
```

**`${string}` pattern** — agar placeholder'da literal emas `string` bo'lsa, kompilator concrete literal hosil qila olmaydi. Natija `` `prefix${string}` `` pattern type bo'lib qoladi. Bu pattern type structural matching'da ishlatiladi — har qanday string `prefix` bilan boshlansa mos keladi.

**Intrinsic types implementation:** `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — TS compiler'ning ichida maxsus kod sifatida implement qilingan. `lib.es5.d.ts`'da bu type'lar `type Uppercase<S extends string> = intrinsic;` deb declare qilingan — `intrinsic` keyword kompilatorga "bu type'ni mening ichki implementatsiyam orqali hisobla" deb buyuradi. Kompilator JavaScript'ning native `String.prototype.toUpperCase()` / `toLowerCase()`'ga o'xshash logika orqali string literal'larni transform qiladi.

Bu type'lar runtime'da mavjud emas — faqat compile-time'da string literal transformatsiya uchun ishlaydi. (Eslatma: TypeScript compiler'ning o'zi to'liq TypeScript'da yozilgan.)

**Template literal bilan `infer`:** TS 4.1+ da template literal type'lar ichida `infer` ishlatib string parsing qilish mumkin:

```typescript
type ParseRoute<T> = T extends `/${infer Segment}/${infer Rest}` ? [Segment, ...ParseRoute<`/${Rest}`>] : T extends `/${infer Last}` ? [Last] : [];

type R = ParseRoute<"/users/42/posts">;
// ["users", "42", "posts"]
```

Bu pattern type-level string parsing uchun ishlatiladi — route matching, SQL query builder, path type'lar.

**`${string}` empty string matching gotcha:** `${string}` pattern hatto **bo'sh string**'ni ham mos keladi. Bu kutilmagan bo'lishi mumkin:

```typescript
type Email = `${string}@${string}.${string}`;
const e: Email = "@." as Email; // ✅ TS allows (all parts empty)
```

Shuning uchun template literal type'lar runtime validation bilan kombinatsiya qilinishi kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy template literal
type Greeting = `Hello, ${string}`;

const a: Greeting = "Hello, Ali";    // ✅
const b: Greeting = "Hello, World";  // ✅
// const c: Greeting = "Hi";         // ❌

// 2. Union Cartesian product
type Color = "red" | "green" | "blue";
type Size = "sm" | "md" | "lg";
type ClassName = `${Color}-${Size}`;
// 9 kombinatsiya

const cls: ClassName = "red-sm"; // ✅

// 3. Intrinsic string types
type Upper = Uppercase<"hello">;       // "HELLO"
type Lower = Lowercase<"HELLO">;       // "hello"
type Cap = Capitalize<"hello">;        // "Hello"
type Uncap = Uncapitalize<"Hello">;    // "hello"

// 4. Event handler name generation
type EventName = "click" | "focus" | "blur";
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// 5. Mapped type bilan kombinatsiya — Getters
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }

// 6. Setters
type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type PersonSetters = Setters<Person>;
// { setName: (value: string) => void; setAge: (value: number) => void }

// 7. Path type (simple)
type CSSProperty = "color" | "background" | "border";
type CSSValue = `${number}px` | `${number}%` | `var(--${string})`;

const width: CSSValue = "100px";      // ✅
const height: CSSValue = "50%";       // ✅
const bg: CSSValue = "var(--primary)"; // ✅

// 8. Route pattern
type Method = "GET" | "POST" | "PUT" | "DELETE";
type Route = `${Method} /${string}`;

const r1: Route = "GET /users";           // ✅
const r2: Route = "POST /users/create";   // ✅
// const r3: Route = "FETCH /data";       // ❌

// 9. Template literal + infer — parsing
type ParseFirst<S> = S extends `${infer First}-${string}` ? First : never;

type P1 = ParseFirst<"red-sm">;    // "red"
type P2 = ParseFirst<"hello">;      // never (no dash)

// 10. Key renaming with template literal
type KeysAsEvents<T> = {
  [K in keyof T]: `on${Capitalize<string & K>}`;
};

type UserEvents = KeysAsEvents<User>;
// { name: "onName"; age: "onAge"; email: "onEmail" }

interface User {
  name: string;
  age: number;
  email: string;
}
```

</details>

> **Chuqurroq:** Template literal types haqida to'liq batafsil — [14-template-literal-types.md](14-template-literal-types.md).

---

## Recursive Types

### Nazariya

Recursive type — type o'zini o'zi reference qilishi. Bu daraxt (tree) strukturalar, chuqur nested object'lar, va type-level algoritmlar uchun kerak.

```typescript
// DeepReadonly — barcha nested property'larni readonly qilish
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]              // Funksiya — o'zgartirma
    : T[K] extends object
      ? DeepReadonly<T[K]>  // Object — recursive
      : T[K];               // Primitive — o'zi
};
```

**TypeScript recursive type'larda cheklov bor:**

- **Non-tail-recursive** — taxminan 50 daraja limit
- **Tail-recursive** (TS 4.5+ optimizatsiyasi) — taxminan 1000 daraja limit

**Tail-call optimization (TS 4.5+):** Agar conditional type'ning true branch'ida recursive call faqat "return position"'da tursa (boshqa type bilan wrap qilinmagan), kompilator uni tail position sifatida aniqlaydi va stack frame'ni qayta ishlatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator recursive type'ni evaluate qilganda har recursion step'da yangi type instantiation yaratadi. Instantiation depth counter bor — har recursive call'da oshadi. Agar bu counter limitga yetsa, kompilator "Type instantiation is excessively deep and possibly infinite" xato beradi.

**Tail-recursive vs non-tail-recursive:**

```typescript
// ❌ Non-tail — recursive call tuple ichida (boshqa type bilan wrap qilingan)
type NonTail<T extends string> =
  T extends `${infer C}${infer Rest}`
    ? [C, ...NonTail<Rest>]   // Recursive call [C, ...] ichida — tail emas
    : [];

// ✅ Tail — recursive call to'g'ridan-to'g'ri return (accumulator pattern)
type Tail<N extends number, Acc extends 0[] = []> =
  Acc["length"] extends N
    ? Acc                           // Base case
    : Tail<N, [...Acc, 0]>;          // Tail call — return pozitsiyasida
```

**Accumulator pattern** — functional programming'dan kelgan texnika. Recursive call har iteration'da javoni "to'playdi" va oxirgi iteration'da to'plangan natijani qaytaradi:

```
Tail<3> — accumulator pattern:

Call 1: Tail<3, []>         → Acc.length (0) !== 3 → Tail<3, [0]>
Call 2: Tail<3, [0]>        → Acc.length (1) !== 3 → Tail<3, [0, 0]>
Call 3: Tail<3, [0, 0]>     → Acc.length (2) !== 3 → Tail<3, [0, 0, 0]>
Call 4: Tail<3, [0, 0, 0]>  → Acc.length (3) === 3 → return [0, 0, 0]
```

Har qadamda natija accumulator ichida saqlanadi. Kompilator tail call'ni optimize qilib, stack frame'ni qayta ishlatadi — depth limit 1000'ga oshadi.

**Lazy evaluation:** Kompilator recursive type'ni **lazy** evaluate qiladi — ya'ni type faqat ishlatilganda (kerak bo'lganda) fully resolve qilinadi. Bu `JSONValue` kabi o'zini reference qiluvchi type alias'lar uchun muhim — agar eager evaluate qilsa, cheksiz loop bo'lardi.

**Depth limit bilan workaround:** Recursive type'ni counter tuple bilan chegaralash standart pattern:

```typescript
type DeepReadonly<T, Depth extends unknown[] = []> =
  Depth["length"] extends 10
    ? T  // 10 darajadan chuqur — to'xta
    : {
        readonly [K in keyof T]: T[K] extends object
          ? DeepReadonly<T[K], [...Depth, unknown]>
          : T[K];
      };
```

Counter tuple har recursion'da o'sadi — limit'ga yetganda evaluation to'xtaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. DeepReadonly — function check bilan
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]
    : T[K] extends object
      ? DeepReadonly<T[K]>
      : T[K];
};

interface Config {
  db: {
    host: string;
    port: number;
    options: {
      ssl: boolean;
      timeout: number;
    };
  };
  app: {
    name: string;
  };
}

type ReadonlyConfig = DeepReadonly<Config>;
// Barcha property'lar va nested property'lar readonly

// 2. DeepPartial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends (...args: any[]) => any
    ? T[K]
    : T[K] extends object
      ? DeepPartial<T[K]>
      : T[K];
};

const partial: DeepPartial<Config> = {
  db: { port: 5432 }, // ✅ faqat port berildi
};

// 3. JSON type — recursive union
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const data: JSONValue = {
  name: "Ali",
  age: 25,
  hobbies: ["coding", "reading"],
  address: {
    city: "Tashkent",
    location: {
      lat: 41.31,
      lng: 69.28,
    },
  },
};

// 4. Nested path type
type Paths<T> = T extends object
  ? {
      [K in keyof T & string]:
        | K
        | (T[K] extends object ? `${K}.${Paths<T[K]>}` : never);
    }[keyof T & string]
  : never;

type ConfigPaths = Paths<Config>;
// "db" | "db.host" | "db.port" | "db.options" | "db.options.ssl"
// | "db.options.timeout" | "app" | "app.name"

// 5. Tail-recursive — Length counting (accumulator pattern)
type Length<T extends unknown[]> = T["length"];

type L1 = Length<[1, 2, 3]>;  // 3
type L2 = Length<[]>;          // 0

// 6. Tail-recursive — Create tuple of N elements
type Repeat<N extends number, T, Acc extends T[] = []> =
  Acc["length"] extends N ? Acc : Repeat<N, T, [...Acc, T]>;

type R1 = Repeat<3, string>;  // [string, string, string]
type R2 = Repeat<5, 0>;        // [0, 0, 0, 0, 0]

// 7. Non-recursive with depth limit
type DeepReadonlyLimited<T, Depth extends unknown[] = []> =
  Depth["length"] extends 10
    ? T
    : {
        readonly [K in keyof T]: T[K] extends object
          ? DeepReadonlyLimited<T[K], [...Depth, unknown]>
          : T[K];
      };

// 8. Flatten nested arrays
type Flatten<T> = T extends (infer U)[]
  ? U extends unknown[]
    ? Flatten<U>
    : U
  : T;

type F1 = Flatten<number[][]>;      // number
type F2 = Flatten<string[][][]>;     // string

// 9. Tree type — recursive structure
interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[];
}

const tree: TreeNode<string> = {
  value: "root",
  children: [
    { value: "child1", children: [] },
    { value: "child2", children: [
      { value: "grandchild", children: [] },
    ]},
  ],
};
```

</details>

---

## Variadic Tuple Types

### Nazariya

Variadic tuple types (TS 4.0+) — tuple type'larda **spread operator** (`...`) ishlatish imkoniyati. Bu tuple'larni birlashtirish, ajratish va transform qilish uchun ishlatiladi.

```typescript
// Ikki tuple'ni birlashtirish
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];

type A = Concat<[1, 2], [3, 4]>;              // [1, 2, 3, 4]
type B = Concat<[string], [number, boolean]>;  // [string, number, boolean]
```

**Variadic tuple ishlatilgan joylar:**

- Tuple concatenation (`[...T, ...U]`)
- Tuple destructuring (`[infer First, ...infer Rest]`)
- Mixed position (`[string, ...T, number]`)
- Function parameter typing

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator variadic tuple type'ni compile-time'da resolve qiladi. `[...T, ...U]` ko'rganda, kompilator T va U tuple type'larining element type'larini chiqarib, yangi birlashtirilgan tuple type yaratadi.

```
Variadic tuple evaluation:

type Concat<T, U> = [...T, ...U];
Concat<[string, number], [boolean]>

1. T resolve → [string, number]
2. U resolve → [boolean]
3. Spread T → [string, number, ...]
4. Spread U → [..., boolean]
5. Concatenate → [string, number, boolean]
```

**`infer` bilan tuple destructuring:** `[infer First, ...infer Rest]` pattern orqali tuple'ning birinchi element'ini First'ga assign qilish va qolgan element'larni yangi tuple type sifatida Rest'ga assign qilish mumkin:

```typescript
type FirstAndRest<T> = T extends [infer F, ...infer R] ? [F, R] : never;

type F = FirstAndRest<[1, 2, 3]>;  // [1, [2, 3]]
```

**Mixed position — `[string, ...T, number]`:** Variadic element o'rtada ham turishi mumkin. Kompilator bu holatda T'ni "middle element"'lar sifatida infer qiladi — birinchi va oxirgi element'lar fixed, o'rtadagilar T'ga to'planadi.

```typescript
type SurroundWith<T extends unknown[]> = [string, ...T, number];
type S = SurroundWith<[boolean, boolean]>;  // [string, boolean, boolean, number]
```

**Runtime'da iz yo'q:** Barcha variadic tuple construct'lar compile-time'da yo'qoladi — faqat oddiy JavaScript array operatsiyalari qoladi. Tuple'lar runtime'da oddiy array, type'lari yo'q.

**Function parameter sifatida:** Variadic tuple'larning asosiy real-world ishlatilishi — funksiya parameter type'larini birlashtirish:

```typescript
// Parameters'ni boshqa parameter'lar bilan birlashtirish
function withCallback<T extends unknown[], U>(
  ...args: [...T, (result: U) => void]
): void {
  // args: [...T, callback]
}
```

Bu pattern `bind`, `curry`, `pipe` kabi higher-order funksiyalar uchun muhim.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Concat — ikki tuple'ni birlashtirish
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];

type A = Concat<[1, 2], [3, 4]>;    // [1, 2, 3, 4]
type B = Concat<[string], [number, boolean]>; // [string, number, boolean]

// 2. Function versiyasi
function concat<T extends unknown[], U extends unknown[]>(
  a: [...T],
  b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}

const result = concat([1, "hello"] as [number, string], [true] as [boolean]);
// [number, string, boolean]

// 3. Head/Tail — tuple dekompozitsiya
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T extends unknown[]> = T extends [unknown, ...infer R] ? R : [];

type H = Head<[1, 2, 3]>;  // 1
type T = Tail<[1, 2, 3]>;  // [2, 3]

// 4. Push/Unshift
type Push<T extends unknown[], V> = [...T, V];
type Unshift<T extends unknown[], V> = [V, ...T];

type P = Push<[1, 2], 3>;     // [1, 2, 3]
type U = Unshift<[2, 3], 1>;  // [1, 2, 3]

// 5. Length of tuple
type Length<T extends readonly unknown[]> = T["length"];

type L1 = Length<[1, 2, 3]>;         // 3
type L2 = Length<[]>;                // 0
type L3 = Length<readonly [string]>; // 1

// 6. Reverse tuple
type Reverse<T extends unknown[]> =
  T extends [infer F, ...infer R]
    ? [...Reverse<R>, F]
    : [];

type R1 = Reverse<[1, 2, 3]>;  // [3, 2, 1]

// 7. Middle variadic — surround pattern
type Wrap<T extends unknown[]> = [string, ...T, number];

type W = Wrap<[boolean, symbol]>;  // [string, boolean, symbol, number]

// 8. Type-safe event emitter with tuple payloads
type EventMap = {
  click: [x: number, y: number];
  input: [value: string];
  error: [code: number, message: string];
};

class TypedEmitter<Events extends Record<string, unknown[]>> {
  private handlers = new Map<keyof Events, Array<(...args: any[]) => void>>();

  on<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): void {
    const list = this.handlers.get(event) ?? [];
    list.push(handler);
    this.handlers.set(event, list);
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    this.handlers.get(event)?.forEach((fn) => fn(...args));
  }
}

const emitter = new TypedEmitter<EventMap>();

emitter.on("click", (x, y) => {
  // x: number, y: number
  console.log(`Click at ${x}, ${y}`);
});

emitter.emit("click", 100, 200);  // ✅
// emitter.emit("click", "100");  // ❌

// 9. Curry function type (simplified)
type Curry<P extends unknown[], R> =
  P extends [infer First, ...infer Rest]
    ? Rest extends []
      ? (arg: First) => R
      : (arg: First) => Curry<Rest, R>
    : R;

type CurriedAdd = Curry<[number, number, number], number>;
// (arg: number) => (arg: number) => (arg: number) => number
```

</details>

---

## Generic Constraints Best Practices

### Nazariya

Generic constraint yozishda ikkita xato tomon bor: **over-constraining** (haddan tashqari cheklash) va **under-constraining** (yetarli cheklamaslik). To'g'ri constraint — faqat **kerakli minimum**'ni cheklaydi.

**Over-constraining:** Constraint keraksiz shart'larni talab qiladi — natijada funksiya kam type bilan ishlaydi.

**Under-constraining:** Constraint yetarli emas — funksiya ichida type xatolari chiqadi yoki `any` kabi ishlashga majbur bo'ladi.

**Printsip: faqat ishlatiladigan xususiyatlarni cheklash.**

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator generic constraint'ni har `T` instantiation'da tekshiradi. Constraint qanchalik keng bo'lsa — shuncha ko'p type'lar qabul qilinadi.

**Over-constraining va inference:** Agar constraint juda qat'iy bo'lsa, TypeScript inference ham buziladi:

```typescript
// Over-constrained: faqat User interface'ni talab qiladi
function f<T extends User>(x: T): T {
  return x;
}

f({ name: "Ali" });
// ❌ Error: id, email, role yo'q
// Kompilator: Argument is not assignable to User

// Minimal constraint: faqat name kerak
function g<T extends { name: string }>(x: T): T {
  return x;
}

g({ name: "Ali" });                   // ✅
g({ name: "Ali", id: 1, extra: true }); // ✅
// T inferred as { name: string; id: number; extra: boolean }
```

**Under-constraining:** Agar constraint yo'q bo'lsa, funksiya body'da type xatolari chiqadi:

```typescript
// ❌ Under-constrained — constraint yo'q
function merge<T, U>(a: T, b: U): T & U {
  return { ...a, ...b }; // ❌ spread works on object types only
}

// ✅ Object constraint
function mergeFixed<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b } as T & U;
}
```

**Dependent constraint — `<U extends keyof T>`:** Constraint boshqa type parameter'ga reference qilishi mumkin. Bu holatda kompilator avval `T`'ni resolve qiladi, keyin `T`'dan olingan key union'ni `U`'ning constraint sifatida ishlatadi:

```typescript
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

`T` avval kelib, keyin `K`'ning constraint'ida ishlatiladi. Tartib muhim — `<K, T>`'da `K` uchun `T` hali mavjud emas.

**Pedagogic xulosa:** Constraint qoidasi oddiy — faqat **ishlatadigan xususiyatlarni** talab qiling. Bu principle "minimum coupling" ga asoslanadi — funksiya nima kerak bo'lsa, faqat shuni so'raydi, ortiqchasini emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Over-constraining — yomon pattern
interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

// ❌ Over-constrained
function getNameBad<T extends User>(obj: T): string {
  return obj.name;
}

// Muammo: faqat name kerak, lekin User'ning BARCHA property'lari talab qilinadi
// getNameBad({ name: "Ali" }); // ❌ Error

// ✅ Minimal constraint
function getName<T extends { name: string }>(obj: T): string {
  return obj.name;
}

getName({ name: "Ali" });                      // ✅
getName({ name: "Ali", id: 1, extra: true });  // ✅

// 2. Under-constraining — yomon pattern
// ❌ Spread primitive'larda ishlamaydi
function mergeBad<T, U>(a: T, b: U): T & U {
  return { ...a, ...b }; // ❌ Error
}

// ✅ Object constraint
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b } as T & U;
}

merge({ name: "Ali" }, { age: 25 }); // ✅

// 3. keyof bilan property access
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// 4. Multiple constraints — intersection
interface Identifiable {
  id: string;
}

interface Timestamped {
  createdAt: Date;
}

function logEntity<T extends Identifiable & Timestamped>(entity: T): void {
  console.log(`${entity.id} created at ${entity.createdAt}`);
}

logEntity({
  id: "1",
  createdAt: new Date(),
  extra: "data", // ✅ qo'shimcha OK
});

// 5. Constraint with default
function wrap<T extends object = {}>(value: T): { data: T } {
  return { data: value };
}

wrap({ name: "Ali" }); // { data: { name: string } }

// 6. Constructor constraint
function createInstance<T>(Ctor: new () => T): T {
  return new Ctor();
}

class Dog {}
const dog = createInstance(Dog); // Dog

// 7. Function constraint
function invoke<T extends () => unknown>(fn: T): ReturnType<T> {
  return fn() as ReturnType<T>;
}

invoke(() => 42);       // number
invoke(() => "hello");  // string

// 8. Iteratable constraint
function firstItem<T>(iterable: Iterable<T>): T | undefined {
  for (const item of iterable) {
    return item;
  }
  return undefined;
}

firstItem([1, 2, 3]);       // number | undefined
firstItem("hello");          // string | undefined
firstItem(new Set([1, 2])); // number | undefined
```

</details>

---

## Real-World Advanced Patterns

### Nazariya

Advanced generics production code'da eng ko'p quyidagi pattern'larda uchraydi: API response typing, state management, ORM-style query building, va type-safe DSL yaratish.

Bu pattern'lar ko'pincha **type-level** va **runtime-level** ni birlashtiradi — type'lar compile-time'da xavfsizlikni ta'minlaydi, runtime'da oddiy JavaScript logic ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Complex generic chain'larning cost'i:** Murakkab generic pattern'lar kompilator uchun **type instantiation cost** yaratadi. Har generic funksiya call'da kompilator yangi type instantiation yaratadi. Chuqur generic chain'larda (masalan, `api<Path, Method>` ichida nested conditional type'lar bilan) bu instantiation'lar ko'payib ketadi.

```
Type instantiation chain (API example):

api("/users", "GET")
  1. Path = "/users" → instantiate ApiEndpoints["/users"]
  2. Method = "GET" → instantiate ApiEndpoints["/users"]["GET"]
  3. Conditional: extends { body: infer B } → evaluate infer
  4. Return type: extends { response: infer R } → evaluate infer
  5. Natija: Promise<User[]>

Har qadam — yangi instantiation
```

**Pedagogic xulosa:** Murakkab generic chain'lar compile vaqtini sekinlashtiradi. Katta loyihalarda `tsc --diagnostics` bilan `Type Count` va `Instantiation count` metric'larini kuzatish muhim. Agar compilation sekin bo'lsa:

1. **Simplify constraint'lar** — keraksiz extends'larni olib tashlash
2. **Split types** — bir katta generic tiplagan o'rniga, bir nechta kichiklari
3. **Type aliases cache** — `type Cached<T> = ...` bilan takror hisoblashni kamaytirish
4. **Avoid deep recursion** — tail-recursive pattern'lar ishlatish

**Cache mechanism:** Kompilator bir xil type argument'lar bilan ikkinchi marta instantiation kerak bo'lganda, cache'dan oladi. Lekin agar har chaqiriqda yangi type argument kombinatsiyasi bo'lsa, cache miss bo'ladi — har biri qaytadan evaluate qilinadi.

**Builder pattern va type evolution:** Fluent builder pattern'da har method call yangi generic instantiation yaratadi (`QueryBuilder<T, K>` → `QueryBuilder<T, K2>`). Katta chain'larda bu compile vaqtini sezilarli oshirishi mumkin.

**Runtime'da murakkablik yo'q:** Barcha generic type'lar, conditional type'lar, `infer`'lar to'liq erase qilinadi. Faqat oddiy JavaScript function call'lari va object'lar qoladi. Type safety — faqat compile-time kafolat.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ═════════════════════════════════════════════
// 1. Type-Safe API Client
// ═════════════════════════════════════════════

interface User {
  id: number;
  name: string;
  email: string;
}

interface ApiEndpoints {
  "/users": {
    GET: { response: User[]; query: { page?: number } };
    POST: { response: User; body: { name: string; email: string } };
  };
  "/users/:id": {
    GET: { response: User; params: { id: string } };
    PUT: { response: User; params: { id: string }; body: Partial<User> };
    DELETE: { response: void; params: { id: string } };
  };
}

type HasBody<T> = T extends { body: infer B } ? { body: B } : Record<string, never>;
type GetResponse<T> = T extends { response: infer R } ? R : never;

async function api<
  Path extends keyof ApiEndpoints,
  Method extends keyof ApiEndpoints[Path]
>(
  path: Path,
  method: Method,
  options?: HasBody<ApiEndpoints[Path][Method]>
): Promise<GetResponse<ApiEndpoints[Path][Method]>> {
  const response = await fetch(path as string, {
    method: method as string,
    body: options && "body" in options ? JSON.stringify(options.body) : undefined,
  });
  return response.json();
}

// Ishlatish — to'liq type safety
// const users = await api("/users", "GET");        // User[]
// const newUser = await api("/users", "POST", {
//   body: { name: "Ali", email: "ali@test.com" },
// });                                                 // User
// api("/users", "PATCH");                          // ❌ PATCH yo'q

// ═════════════════════════════════════════════
// 2. Type-Safe State Management (Redux-style)
// ═════════════════════════════════════════════

type Action =
  | { type: "SET_USER"; payload: User }
  | { type: "SET_LOADING"; payload: boolean }
  | { type: "SET_ERROR"; payload: string | null };

// Action type'dan payload olish
type ActionPayload<T extends Action["type"]> =
  Extract<Action, { type: T }>["payload"];

type UserPayload = ActionPayload<"SET_USER">;       // User
type LoadingPayload = ActionPayload<"SET_LOADING">; // boolean

// Type-safe dispatch
function dispatch<T extends Action["type"]>(
  type: T,
  payload: ActionPayload<T>
): void {
  // dispatch logic
}

dispatch("SET_USER", { id: 1, name: "Ali", email: "ali@test.com" }); // ✅
dispatch("SET_LOADING", true);                                        // ✅
// dispatch("SET_LOADING", "invalid");                                // ❌

// ═════════════════════════════════════════════
// 3. Type-Safe Query Builder
// ═════════════════════════════════════════════

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

class QueryBuilder<T, Selected extends keyof T = keyof T> {
  private conditions: string[] = [];
  private selectedFields: string[] = [];

  constructor(private table: string) {}

  select<K extends keyof T>(...fields: K[]): QueryBuilder<T, K> {
    this.selectedFields = fields as string[];
    return this as unknown as QueryBuilder<T, K>;
  }

  where(condition: Partial<Pick<T, Selected>>): this {
    Object.entries(condition).forEach(([key, value]) => {
      this.conditions.push(`${key} = ${JSON.stringify(value)}`);
    });
    return this;
  }

  build(): { query: string; result: Pick<T, Selected> } {
    const fields = this.selectedFields.length > 0
      ? this.selectedFields.join(", ")
      : "*";
    const where = this.conditions.length > 0
      ? ` WHERE ${this.conditions.join(" AND ")}`
      : "";
    return {
      query: `SELECT ${fields} FROM ${this.table}${where}`,
      result: {} as Pick<T, Selected>,
    };
  }
}

const query = new QueryBuilder<Product>("products")
  .select("name", "price")         // Selected = "name" | "price"
  .where({ name: "Phone" })        // ✅
  .build();
// query.result: Pick<Product, "name" | "price">

// ═════════════════════════════════════════════
// 4. Validator DSL
// ═════════════════════════════════════════════

type ValidationResult<T> =
  | { valid: true; value: T }
  | { valid: false; errors: string[] };

type Validator<T> = (value: unknown) => ValidationResult<T>;

function string(): Validator<string> {
  return (value) =>
    typeof value === "string"
      ? { valid: true, value }
      : { valid: false, errors: ["Expected string"] };
}

function number(): Validator<number> {
  return (value) =>
    typeof value === "number"
      ? { valid: true, value }
      : { valid: false, errors: ["Expected number"] };
}

function object<T extends Record<string, Validator<unknown>>>(
  schema: T
): Validator<{ [K in keyof T]: T[K] extends Validator<infer U> ? U : never }> {
  return (value) => {
    if (typeof value !== "object" || value === null) {
      return { valid: false, errors: ["Expected object"] };
    }
    const errors: string[] = [];
    const result: Record<string, unknown> = {};

    for (const key in schema) {
      const validator = schema[key];
      const field = (value as Record<string, unknown>)[key];
      const validation = validator(field);

      if (validation.valid) {
        result[key] = validation.value;
      } else {
        errors.push(...validation.errors.map((e) => `${key}: ${e}`));
      }
    }

    return errors.length === 0
      ? { valid: true, value: result as any }
      : { valid: false, errors };
  };
}

// Ishlatish
const userValidator = object({
  name: string(),
  age: number(),
});

const result = userValidator({ name: "Ali", age: 25 });
if (result.valid) {
  console.log(result.value); // { name: string; age: number }
}
```

</details>

---

## Higher-Kinded Types

### Nazariya

Higher-kinded type (HKT) — type parameter'ning o'zi ham generic bo'lishi, ya'ni "generic'ning generic-i". Masalan, `F<T>` da `F` o'zi generic — `Array`, `Promise`, `Option` kabi. Haskell va Scala'da bu to'liq qo'llab-quvvatlanadi, lekin **TypeScript'da HKT to'g'ridan-to'g'ri qo'llab-quvvatlanmaydi**.

**Nima uchun kerak?** `map` funksiya yozmoqchisiz — `Array.map`, `Promise.then`, `Option.map` — barchasi bir xil pattern: `F<T>`'ni `F<U>`'ga transform qilish. HKT bilan bitta generic funksiya yozish mumkin bo'lardi.

```typescript
// Haskell da:
// class Functor f where
//   fmap :: (a -> b) -> f a -> f b

// TS da bunday yozib bo'lmaydi:
// function map<F, A, B>(fa: F<A>, fn: (a: A) => B): F<B>
//                          ^^^ — ❌ F is not generic
```

TypeScript'da HKT'ni **simulate qilish** uchun workaround'lar bor — lekin ular til darajasidagi feature emas, library darajasidagi abstraction.

<details>
<summary><strong>Under the Hood</strong></summary>

**Nima uchun TypeScript'da HKT yo'q?** TypeScript'ning type system'ida type parameter faqat **concrete type** (kind `*`) bo'lishi mumkin — ya'ni `T` o'zi `string`, `number`, `User` kabi to'liq type. HKT uchun type parameter'ning "kind" (type'ning type'i) `* -> *` bo'lishi kerak — ya'ni `F` o'zi generic, unga type berganingda concrete type hosil bo'ladi (`Array`, `Promise` kabi).

TypeScript'ning type system'i bunday "type constructor"'larni qo'llab-quvvatlamaydi. Bu dizayn qaroridan kelib chiqadi — TypeScript ko'proq structural va pragmatic bo'lish uchun yaratilgan, akademik type system emas. HKT qo'shilish kompilator arxitekturasini tubdan o'zgartirishni talab qiladi.

**Workaround yondashuvlari:**

1. **TypeMap pattern** — barcha generic type'larni bitta interface'ga yig'ib, string key orqali conditional type bilan dispatch qilish
2. **Defunctionalization** — `fp-ts` ishlatadigan pattern, har generic type uchun brand interface yaratib, `Kind<F, A>` utility type orqali resolve qilish
3. **Concrete overloads** — HKT o'rniga har type uchun alohida overload yozish

Ikkala workaround ham TypeScript'ning structural type system va conditional type'larini ishlatadi. Lekin bular compiler darajasida emas, library darajasidagi abstraction — shuning uchun har yangi generic type qo'shganda TypeMap yoki brand registry'ni manually kengaytirish kerak.

Amaliyotda ko'p TypeScript loyihalari HKT workaround'larini **ishlatmaydi** — oddiy overload'lar yoki concrete type'lar bilan cheklanadi. HKT kerak bo'lgan holatlar kam, va kod o'qilishi qiyin bo'lib qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. HKT workaround — TypeMap pattern
interface TypeMap {
  Array: unknown[];
  Promise: Promise<unknown>;
  Option: { value: unknown } | { value: null };
}

// Apply — TypeMap dagi F'ga T'ni qo'yish
type Apply<F extends keyof TypeMap, T> =
  F extends "Array" ? T[] :
  F extends "Promise" ? Promise<T> :
  F extends "Option" ? { value: T } | { value: null } :
  never;

// Functor interface
interface Functor<F extends keyof TypeMap, T> {
  map<U>(fn: (value: T) => U): Apply<F, U>;
}

// Bu cheklangan — yangi type qo'shish uchun TypeMap'ni o'zgartirish kerak

// 2. Amaliy yechim — overload bilan
function mapArr<T, U>(container: T[], fn: (value: T) => U): U[];
function mapArr<T, U>(container: Promise<T>, fn: (value: T) => U): Promise<U>;
function mapArr<T, U>(
  container: T[] | Promise<T>,
  fn: (value: T) => U
): U[] | Promise<U> {
  if (Array.isArray(container)) {
    return container.map(fn);
  }
  return container.then(fn);
}

const arr = mapArr([1, 2, 3], (n) => n.toString());      // string[]
const prom = mapArr(Promise.resolve(42), (n) => n * 2);  // Promise<number>

// 3. Option type simulation
type Option<T> = Some<T> | None;
interface Some<T> { kind: "some"; value: T; }
interface None { kind: "none"; }

function some<T>(value: T): Some<T> {
  return { kind: "some", value };
}

const none: None = { kind: "none" };

function mapOption<T, U>(option: Option<T>, fn: (value: T) => U): Option<U> {
  return option.kind === "some" ? some(fn(option.value)) : option;
}

const opt = some(42);
const doubled = mapOption(opt, (n) => n * 2);
// Option<number>

// 4. Real library — fp-ts style (simplified)
// fp-ts actual implementation juda murakkab — bu ishlashni ko'rsatadi
interface HKT<URI, A> {
  readonly _URI: URI;
  readonly _A: A;
}

interface URItoKind<A> {
  Array: A[];
  Promise: Promise<A>;
}

type URIS = keyof URItoKind<unknown>;
type Kind<URI extends URIS, A> = URItoKind<A>[URI];

// Bu pattern TypeScript'ning eng qiyin joylaridan biri
// Amaliy loyihalarda kam ishlatiladi
```

</details>

---

## Edge Cases va Gotchas

### 1. `never` va Distributive — Hech Narsa Qaytaradi

`never` — "bo'sh union" deb qaraladi. Distributive conditional type'da `never` berilsa, kompilator distribution'ga hech narsa bermaydi — natija `never`:

```typescript
type ToArray<T> = T extends any ? T[] : never;

type N = ToArray<never>; // never
// Kutilgan: never[]
// Haqiqiy: never — empty union'da nothing to distribute
```

**Nima uchun:** `never` — "hech qanday qiymat yo'q" ma'nosini beradi. Distributive behavior union'ning har member'i uchun alohida evaluation qiladi, lekin empty union'da hech qanday member yo'q — shuning uchun natija ham empty (`never`).

**Yechim** — non-distributive `[T] extends [any]` ishlatish:

```typescript
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type N2 = ToArrayNonDist<never>; // never[]
```

Bu pattern `never` bilan ishlaganda kutilmagan natijalarning oldini oladi.

### 2. `any` Distributive Behavior — Har Doim Distribute Qiladi

`any` type conditional type'da noodatiy xatti-harakatga ega. `any extends U` har doim ikkala branch'ni tanlaydi — natija union bo'ladi:

```typescript
type Check<T> = T extends string ? "yes" : "no";
type A = Check<any>; // "yes" | "no" (ikkala branch!)
```

**Sabab:** `any` — "istalgan type" ma'nosini beradi. Kompilator `any extends string` shartini tekshira olmaydi — chunki `any` bir vaqtda ham string bo'lishi mumkin, ham emas. Shuning uchun kompilator pessimistik yondashadi va ikkala branch'ni qo'shadi.

**Nima uchun muhim:** `any` parameter bilan conditional type kutilmagan natijalar beradi. Agar aniq natija kerak bo'lsa, `unknown` yoki specific type ishlatish kerak:

```typescript
type Check2<T> = T extends string ? "yes" : "no";
type U = Check2<unknown>; // "no" — unknown string emas
type S = Check2<string>;  // "yes" — aniq string
```

### 3. `infer` Multi-Position — Same Variable Different Positions

`infer U` bir nechta pozitsiyada ishlatilganda, kompilator variance qoidalariga asoslanib natijani union yoki intersection qiladi:

```typescript
// Return position — covariant → union
type GetReturns<T> = T extends { a: () => infer R; b: () => infer R } ? R : never;
type R1 = GetReturns<{ a: () => string; b: () => number }>;
// string | number

// Parameter position — contravariant → intersection
type GetParams<T> = T extends { a: (x: infer P) => void; b: (x: infer P) => void } ? P : never;
type P1 = GetParams<{ a: (x: string) => void; b: (x: number) => void }>;
// string & number = never
```

**Nima uchun intersection `never`'ga olib keladi:** `string & number` "string VA number" ma'nosini beradi — bunday qiymat mavjud emas. Shuning uchun intersection `never`. Bu odatda xohlanmagan natija — ikki parameter'ga bitta `infer P` ishlatish odatda mantiqiy xato.

**Yechim** — har parameter uchun alohida infer:

```typescript
type GetParamsEach<T> = T extends { a: (x: infer P1) => void; b: (x: infer P2) => void }
  ? [P1, P2]
  : never;

type P2 = GetParamsEach<{ a: (x: string) => void; b: (x: number) => void }>;
// [string, number]
```

### 4. Template Literal `${string}` — Bo'sh String Ham Mos Keladi

`${string}` pattern **bo'sh string**'ni ham mos keladi — bu kutilmagan bo'lishi mumkin:

```typescript
type Email = `${string}@${string}.${string}`;

const e: Email = "@.";         // ✅ TypeScript allows!
const e2: Email = "a@b.c";     // ✅
const e3: Email = "@b.c";      // ✅ (birinchi qism bo'sh)
```

**Sabab:** `${string}` — "har qanday string, shu jumladan bo'sh". Pattern faqat `@` va `.` belgilarini talab qiladi, qolgan qismlar istalgan bo'lishi mumkin.

**Muhimi:** Template literal type — **compile-time pattern matching**, to'liq validation emas. Real email format tekshiruvi runtime validator (`type guard`, regex, zod) talab qiladi.

**Yechim** — kamida bitta character talab qilish uchun `infer extends` ishlatish yoki runtime validation qo'shish:

```typescript
// Runtime validation type guard
function isEmail(value: string): value is Email {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
}

const input: string = "ali@test.com";
if (isEmail(input)) {
  // input: Email (type-safe + runtime-validated)
}
```

### 5. `as never` Key Remapping — Filter Pattern

Mapped type'da `as never` remapping — property'ni butunlay olib tashlaydi. Bu filter pattern sifatida ishlatiladi:

```typescript
// Function property'larni olib tashlash
type RemoveFunctions<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any ? never : K]: T[K];
};

interface User {
  name: string;
  age: number;
  greet: () => string;
  logout: () => void;
}

type UserData = RemoveFunctions<User>;
// { name: string; age: number }
// greet va logout olib tashlandi
```

**Nima uchun `as never` property'ni o'chiradi:** Mapped type'da key `never` bo'lsa, property "mavjud emas" sifatida qaraladi. Kompilator bu property'ni butunlay chiqarib tashlaydi. Bu filter uchun juda kuchli pattern.

**Gotcha:** Bu mexanizm faqat `keyof T` ishlatilgan mapped type'da ishlaydi. Oddiy union key source'da (`"a" | "b"`) `never` boshqacha ishlashi mumkin.

---

## Common Mistakes

### ❌ Xato 1: Distributive behavior'ni hisobga olmaslik

```typescript
type ToArray<T> = T extends any ? T[] : never;

// Kutilgan: (string | number)[]
// Haqiqiy: string[] | number[]
type Result = ToArray<string | number>;
```

**✅ To'g'ri usul:**

```typescript
type ToArray<T> = [T] extends [any] ? T[] : never;

type Result = ToArray<string | number>;
// (string | number)[] ✅
```

**Nima uchun:** Naked type parameter `T` union bilan conditional type'da har doim distributive bo'ladi. `[T]` bilan wrapping distribution'ni to'xtatadi.

---

### ❌ Xato 2: `infer`'ni noto'g'ri joyda ishlatish

```typescript
// ❌ infer faqat extends clause'da ishlaydi
// type Bad<T> = infer U; // Error

// ❌ infer false branch'da ishlatilmaydi
// type Wrong<T> = T extends string ? T : infer U;
```

**✅ To'g'ri usul:**

```typescript
// ✅ infer — extends clause'da, true branch'da ishlatiladi
type ElementOf<T> = T extends (infer U)[] ? U : never;

type Item = ElementOf<string[]>; // string
```

**Nima uchun:** `infer` — pattern matching operatori. U faqat `extends` clause'da pattern ichida joylashishi va faqat true branch'da murojaat qilinishi mumkin.

---

### ❌ Xato 3: Recursive type'da depth limit'ga uchrash

```typescript
// ❌ Juda chuqur nested bo'lsa xato beradi
type DeepFlatten<T> = T extends (infer U)[] ? DeepFlatten<U> : T;

type TooDeep = [[[[[[[[[[[[[[[[[[[[string]]]]]]]]]]]]]]]]]]]];
// type Flat = DeepFlatten<TooDeep>; // ❌ excessively deep
```

**✅ To'g'ri usul:**

```typescript
// ✅ Depth limit bilan counter tuple
type DeepFlatten<T, Depth extends unknown[] = []> =
  Depth["length"] extends 10
    ? T
    : T extends (infer U)[]
      ? DeepFlatten<U, [...Depth, unknown]>
      : T;

type Flat = DeepFlatten<[[[[string]]]]>; // string ✅
```

**Nima uchun:** Kompilator recursive type'larni chekli darajagacha qo'llab-quvvatlaydi. Counter tuple bilan chegaralash — standart workaround.

---

### ❌ Xato 4: Non-homomorphic mapped type'da modifier yo'qolishi

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

// ❌ Non-homomorphic — modifier'lar YO'QOLADI
type ManualNullable = {
  [K in "host" | "port"]: Config[K] | null;
};
// { host: string | null; port: number | null }
// readonly va ? yo'qoldi
```

**✅ To'g'ri usul:**

```typescript
// ✅ Homomorphic — keyof T ishlatish
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type NullableConfig = Nullable<Config>;
// { readonly host: string | null; port?: number | null }
// ✅ readonly va ? saqlandi
```

**Nima uchun:** `keyof T` bilan iterate qiluvchi mapped type — **homomorphic** deyiladi va modifier'larni avtomatik saqlaydi. Boshqa key source'da modifier'lar yo'qoladi. Doim `keyof T` ishlatish — safe default.

---

### ❌ Xato 5: Template literal type'da runtime validation yo'qligi

```typescript
type Email = `${string}@${string}.${string}`;

const userInput: string = "not-an-email";
// as Email bilan TS ruxsat beradi, lekin runtime xato:
// const email = userInput as Email;
```

**✅ To'g'ri usul:**

```typescript
type Email = `${string}@${string}.${string}`;

function isEmail(value: string): value is Email {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
}

function sendEmail(to: Email): void {
  fetch(`/api/send?to=${to}`);
}

const userInput = "ali@test.com";
if (isEmail(userInput)) {
  sendEmail(userInput); // ✅ Type-safe + runtime-safe
}
```

**Nima uchun:** Template literal types — compile-time pattern. JavaScript'ga compile bo'lganda yo'qoladi. Runtime xavfsizlik uchun **type guard** qo'shish kerak — ikkala daraja (compile + runtime)'da himoya.

---

## Amaliy Mashqlar

### Mashq 1: ReturnType'ni Implement Qiling (Oson)

**Savol:** `MyReturnType<T>` type'ni yozing — funksiya type'ning return type'ini olsin. `infer` ishlatish kerak.

```typescript
type MyReturnType<T> = /* yechim */;

type A = MyReturnType<() => string>;           // string
type B = MyReturnType<(x: number) => boolean>; // boolean
type C = MyReturnType<() => void>;             // void
```

<details>
<summary>Javob</summary>

```typescript
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = MyReturnType<() => string>;           // string ✅
type B = MyReturnType<(x: number) => boolean>; // boolean ✅
type C = MyReturnType<() => void>;             // void ✅
type D = MyReturnType<string>;                  // never — funksiya emas
```

**Tushuntirish:**

- `T extends (...args: any[]) => infer R` — T funksiya type'mi? Agar ha — return pozitsiyadagi type'ni R'ga yoz
- `? R` — true branch: R (infer qilingan return type)
- `: never` — false branch: T funksiya emas — never

Bu TypeScript'ning built-in `ReturnType<T>` utility type'ning aniq implementation'i.

</details>

---

### Mashq 2: DeepPartial'ni Implement Qiling (O'rta)

**Savol:** `DeepPartial<T>` type'ni yozing — barcha nested property'larni optional qilsin. Function type'larni o'zgartirmasin.

```typescript
interface Config {
  db: { host: string; port: number; options: { ssl: boolean } };
  app: { name: string };
}

type PartialConfig = DeepPartial<Config>;
// db?, db.host?, db.port?, db.options?, db.options.ssl?, app?, app.name?
```

<details>
<summary>Javob</summary>

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends (...args: any[]) => any
    ? T[K]              // Funksiya — o'zgartirma
    : T[K] extends object
      ? DeepPartial<T[K]> // Object — recursive
      : T[K];             // Primitive — o'zi
};

interface Config {
  db: { host: string; port: number; options: { ssl: boolean } };
  app: { name: string };
  onError: (e: Error) => void;
}

type PartialConfig = DeepPartial<Config>;
// {
//   db?: { host?: string; port?: number; options?: { ssl?: boolean } };
//   app?: { name?: string };
//   onError?: (e: Error) => void;  // ✅ Function saqlanadi
// }
```

**Tushuntirish:**

- `[K in keyof T]?:` — barcha property'lar optional (`?` qo'shildi) — homomorphic
- `T[K] extends (...args: any[]) => any` — Function check (proper callable signature)
- `T[K] extends object` — agar property object bo'lsa
- `DeepPartial<T[K]>` — object bo'lsa recursive chaqiruv
- Primitive yoki function bo'lsa — o'zgartirmasdan qo'yish

</details>

---

### Mashq 3: Exclude/Extract Output'ini Ayting (O'rta)

**Savol:**

```typescript
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;

type Union = "a" | "b" | "c" | 1 | 2 | true;

type A = MyExclude<Union, string>;
type B = MyExtract<Union, string>;
type C = MyExclude<Union, string | number>;
type D = MyExtract<Union, string | boolean>;
```

<details>
<summary>Javob</summary>

```typescript
type A = 1 | 2 | true;
// Har member uchun alohida:
// "a" extends string → never | "b" extends string → never | "c" extends string → never
// 1 extends string → 1 | 2 extends string → 2 | true extends string → true
// Natija: never | never | never | 1 | 2 | true = 1 | 2 | true

type B = "a" | "b" | "c";
// "a" extends string → "a" | "b" → "b" | "c" → "c"
// 1, 2, true extends string → never
// Natija: "a" | "b" | "c"

type C = true;
// string | number bilan ExcludE:
// "a", "b", "c" string → never
// 1, 2 number → never
// true — string ham, number ham emas → true
// Natija: true

type D = "a" | "b" | "c" | true;
// string | boolean bilan Extract:
// "a", "b", "c" string → saqlanadi
// 1, 2 — string ham, boolean ham emas → never
// true boolean → saqlanadi
// Natija: "a" | "b" | "c" | true
```

**Tushuntirish:** Distributive conditional types union'ning har member'ini alohida tekshiradi. `Exclude` mos kelmaganlarni qaytaradi, `Extract` mos kelganlarni. Bu TypeScript'ning built-in utility type'lari asosi.

</details>

---

### Mashq 4: Type-Safe Get Function (Qiyin)

**Savol:** `get(obj, path)` funksiyasining type'ini yozing — dot-notation path bilan nested property olsin.

```typescript
const data = {
  user: {
    name: "Ali",
    address: {
      city: "Tashkent",
    },
  },
};

get(data, "user.name");          // string
get(data, "user.address.city");  // string
get(data, "user.address");       // { city: string }
```

<details>
<summary>Javob</summary>

```typescript
type GetType<T, Path extends string> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? GetType<T[Key], Rest>    // Recursive: keyingi qismga
      : never
    : Path extends keyof T
      ? T[Path]                   // Oxirgi key
      : never;

function get<T extends Record<string, any>, Path extends string>(
  obj: T,
  path: Path
): GetType<T, Path> {
  return path.split(".").reduce((acc, key) => acc?.[key], obj as any);
}

const data = {
  user: {
    name: "Ali",
    address: {
      city: "Tashkent",
    },
  },
};

const name = get(data, "user.name");          // string ✅
const city = get(data, "user.address.city");  // string ✅
const addr = get(data, "user.address");       // { city: string } ✅
// const bad = get(data, "user.phone");       // never
```

**Tushuntirish:**

- `Path extends \`${infer Key}.${infer Rest}\`` — path'da dot bormi?
- `Key extends keyof T` — Key T'ning property'mi?
- `GetType<T[Key], Rest>` — recursive chaqiruv keyingi qismga
- Path'da dot yo'q — oxirgi key, `T[Path]` qaytaradi

Template literal types + `infer` + recursion — TypeScript'ning eng kuchli type-level tool'lari birgalikda. Bu pattern `lodash.get`, Zustand kabi library'larda ishlatiladi.

</details>

---

### Mashq 5: Type-Safe Route Parser (Qiyin)

**Savol:** Route string'dan parameter'lar object'ini chiqaruvchi type yozing. `/users/:id/posts/:postId` → `{ id: string; postId: string }`.

```typescript
type RouteParams<T extends string> = /* yechim */;

type P1 = RouteParams<"/users/:id">;                    // { id: string }
type P2 = RouteParams<"/users/:id/posts/:postId">;       // { id: string; postId: string }
type P3 = RouteParams<"/static">;                         // {}
```

<details>
<summary>Javob</summary>

```typescript
type RouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & RouteParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

// Test
type P1 = RouteParams<"/users/:id">;
// { id: string }

type P2 = RouteParams<"/users/:id/posts/:postId">;
// { id: string; postId: string }

type P3 = RouteParams<"/users/:id/posts/:postId/comments/:commentId">;
// { id: string; postId: string; commentId: string }

type P4 = RouteParams<"/static">;
// {}

// Real ishlatilish — type-safe route handler
function handleRoute<Path extends string>(
  path: Path,
  params: RouteParams<Path>
): void {
  console.log(path, params);
}

handleRoute("/users/:id", { id: "42" });                            // ✅
handleRoute("/users/:id/posts/:postId", { id: "1", postId: "2" }); // ✅
// handleRoute("/users/:id", {}); // ❌ id majburiy
```

**Tushuntirish:**

- Birinchi pattern: `${string}:${infer Param}/${infer Rest}` — parameter'dan keyin `/` bor (ko'proq parameter'lar bor)
- Ikkinchi pattern: `${string}:${infer Param}` — oxirgi parameter (keyin `/` yo'q)
- Base case: parameter yo'q — `{}`
- `{ [K in Param]: string }` — parameter nomidan object property yaratish
- `& RouteParams<...>` — recursive intersection bilan keyingi parameter'lar qo'shish

Bu pattern React Router v6, Next.js, Remix kabi framework'larda ishlatiladi. Template literal + `infer` + recursion + mapped type — TypeScript'ning hamma advanced feature'lari bir joyda.

</details>

---

## Xulosa

Bu bo'limda TypeScript'ning advanced type-level programming imkoniyatlarini o'rgandik:

**Asosiy tushunchalar:**

- **Conditional Types** — `T extends U ? X : Y` — type darajasida shartli logic, type transformation asosi
- **`infer` Keyword** — type ichidan type extraction, pattern matching, multi-position variance
- **Distributive Conditional Types** — union bilan avtomatik distribution, `Exclude`/`Extract`/`NonNullable` asosi
- **Non-Distributive** — `[T]` bilan wrapping orqali distribution to'xtatish, type equality check
- **Mapped Types** — `{ [K in keyof T]: ... }` — property transformation, modifier'lar, key remapping (`as`), homomorphic saving
- **Template Literal Types** — string pattern type'lar, intrinsic types, Cartesian product, template literal + `infer` parsing
- **Recursive Types** — self-reference, tail-call optimization (TS 4.5+), depth limit, accumulator pattern
- **Variadic Tuple Types** — `[...T, ...U]` — tuple composition, destructuring, mixed position
- **Generic Constraints Best Practices** — minimal constraint, over/under-constraining
- **Real-World Patterns** — API client, state management, query builder, validator DSL
- **Higher-Kinded Types** — TS'da yo'q, workaround pattern'lar (TypeMap, defunctionalization)

**Umumiy takeaway'lar:**

1. **Distributive behavior — naked parameter kerak.** `[T]` wrapping bilan to'xtatish.
2. **`keyof T` — modifier saqlash kaliti.** Homomorphic mapped type'lar automatic modifier preservation beradi.
3. **Type-level recursion cheklangan.** Counter tuple yoki tail-call optimization bilan chuqur recursion.
4. **Template literal — compile-time pattern, runtime validation emas.** Type guard bilan birga ishlatish.
5. **`never` va `any` — distributive context'da nozik.** `never` hech narsa beradi, `any` har doim ikkala branch.
6. **Minimal constraint.** Faqat ishlatadigan xususiyatlarni talab qilish — over-constraining'dan qochish.
7. **Complex generic'lar compile cost'ini oshiradi.** Katta loyihalarda `tsc --diagnostics` bilan kuzatish.

**Cross-references:**

- **[06-type-narrowing.md](06-type-narrowing.md)** — `typeof`, `instanceof`, user-defined type guards
- **[07-functions.md](07-functions.md)** — Function type, variance, contextual typing
- **[08-generics.md](08-generics.md)** — Generic'lar asos, `keyof`, index access
- **[12-conditional-types.md](12-conditional-types.md)** — Conditional types chuqur, `infer` advanced
- **[13-mapped-types.md](13-mapped-types.md)** — Mapped types chuqur, key remapping
- **[14-template-literal-types.md](14-template-literal-types.md)** — Template literal types chuqur, string parsing
- **[15-utility-types.md](15-utility-types.md)** — `Pick`, `Omit`, `Partial`, `ReturnType` va boshqa built-in utility'lar
- **[25-type-compatibility.md](25-type-compatibility.md)** — Variance chuqur (covariance, contravariance, bivariance)

---

**Keyingi bo'lim:** [10-classes.md](10-classes.md) — Classes TypeScript'da: access modifiers (`public`, `private`, `protected`), parameter properties, abstract classes, `implements` interface, `override` keyword, `this` type va fluent interfaces, generic classes, `accessor` keyword (TS 4.9+), ECMAScript native private fields (`#field`), va classes at runtime (JavaScript'ga compile).
