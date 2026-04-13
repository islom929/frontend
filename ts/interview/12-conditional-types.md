# Interview: Conditional Types

> Conditional type syntax, `infer` keyword, distributive behavior, recursive conditional types, deferred conditional, covariant/contravariant infer va real-world patterns bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Conditional type nima? `extends` bu yerda nima tekshiradi?

<details>
<summary>Javob</summary>

Conditional type — type darajasida `if/else`. Sintaksisi: `T extends U ? TrueType : FalseType`.

`extends` bu yerda inheritance **emas** — **assignability check**: "T ni U ga assign qilish mumkinmi?"

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">;  // true — "hello" string ga assignable
type B = IsString<number>;   // false

// Object — structural subtyping
type HasName<T> = T extends { name: string } ? true : false;
type C = HasName<{ name: string; age: number }>; // true — ko'proq property = subtype
type D = HasName<{ age: number }>;               // false — name yo'q
```

Nested conditional — chained `if/else if/else`:

```typescript
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  "object";
```

</details>

### 2. `infer` keyword nima qiladi? Qaysi pozitsiyalarda ishlatiladi?

<details>
<summary>Javob</summary>

`infer` — conditional type ichida type ni **extract** qilish (pattern matching). Faqat `extends` clause da ishlatiladi:

```typescript
// Return type olish
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Parameter type olish
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;

// Array element type
type ElementOf<T> = T extends (infer U)[] ? U : never;

// Promise unwrap
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;

// Tuple element lar
type First<T> = T extends [infer F, ...any[]] ? F : never;
type Last<T> = T extends [...any[], infer L] ? L : never;
```

**Muhim:** Bir xil nomli `infer` bir nechta joyda — pozitsiyaga qarab natija farqli:

| Pozitsiya | Natija | Sabab |
|-----------|--------|-------|
| Return type (covariant) | **Union** | Chiqish pozitsiyasi |
| Parameter (contravariant) | **Intersection** | Kirish pozitsiyasi |

```typescript
type CoInfer<T> = T extends { a: () => infer R; b: () => infer R } ? R : never;
type A = CoInfer<{ a: () => string; b: () => number }>;
// R = string | number — UNION

type ContraInfer<T> = T extends { a: (x: infer P) => void; b: (x: infer P) => void } ? P : never;
type B = ContraInfer<{ a: (x: string) => void; b: (x: number) => void }>;
// P = string & number = never — INTERSECTION
```

</details>

### 3. Distributive conditional type nima? Qachon distribute bo'ladi?

<details>
<summary>Javob</summary>

Conditional type ga **union type** berilganda, TS uni har bir member ga alohida qo'llaydi:

```typescript
type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>;
// = ToArray<string> | ToArray<number> = string[] | number[]
// (string | number)[] EMAS!
```

**Uchta sharti:**
1. Conditional type bo'lishi kerak
2. `T` — **naked type parameter** (wrapper yo'q)
3. Union type berilishi kerak

**To'xtatish** — `[T]` bilan wrapping:

```typescript
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type B = ToArrayNonDist<string | number>; // (string | number)[] — butun union
```

</details>

### 4. `infer extends` constraint (TS 4.7+) nima?

<details>
<summary>Javob</summary>

`infer U extends SomeType` — inferred type ni cheklash. Agar mos kelmasa — false branch:

```typescript
// String dan number parse
type ParseInt<T> = T extends `${infer N extends number}` ? N : never;
type A = ParseInt<"42">;    // 42 — number literal!
type B = ParseInt<"hello">; // never

// Boolean parse
type ParseBool<T> = T extends `${infer B extends boolean}` ? B : never;
type C = ParseBool<"true">;  // true
type D = ParseBool<"false">; // false
```

**Tuple dan faqat ma'lum type larni olish:**

```typescript
type NumbersOnly<T extends any[]> =
  T extends [infer H extends number, ...infer Rest]
    ? [H, ...NumbersOnly<Rest>]
    : T extends [any, ...infer Rest]
      ? NumbersOnly<Rest>
      : [];

type E = NumbersOnly<[1, "a", 2, "b", 3]>; // [1, 2, 3]
```

</details>

### 5. Deferred conditional type nima? Generic body da nima uchun ishlamaydi?

<details>
<summary>Javob</summary>

Generic funksiya body da `T` hali aniq emas — conditional type **evaluate bo'lmaydi**, deferred holda turadi:

```typescript
function process<T>(value: T): T extends string ? string[] : T {
  if (typeof value === "string") {
    return value.split(""); // ❌ Type assertion kerak!
    // T hali generic — conditional type resolve bo'lmagan
  }
  return value as any;
}
```

**Nima uchun?** `typeof value === "string"` value ni narrow qiladi, lekin **T type parameter** ni narrow qilmaydi. T hali generic — `string`, `number`, `string | number` — istalgan narsa bo'lishi mumkin.

**Yechim — overloads:**

```typescript
function process(value: string): string[];
function process<T>(value: T): T;
function process(value: unknown): unknown {
  if (typeof value === "string") return value.split("");
  return value;
}
```

Overload larda conditional type kerak emas — har signature mustaqil.

</details>

### 6. `infer` covariant vs contravariant pozitsiyada — nima farq?

<details>
<summary>Javob</summary>

Bir xil `infer R` bir nechta joyda bo'lsa — pozitsiyaga qarab natija farqli:

```typescript
// Covariant (return type) → UNION
type GetReturns<T> = T extends {
  a: () => infer R;
  b: () => infer R;
} ? R : never;

type A = GetReturns<{ a: () => string; b: () => number }>;
// R = string | number — union

// Contravariant (parameter) → INTERSECTION
type GetParams<T> = T extends {
  a: (x: infer P) => void;
  b: (x: infer P) => void;
} ? P : never;

type B = GetParams<{ a: (x: string) => void; b: (x: number) => void }>;
// P = string & number = never

type C = GetParams<{
  a: (x: { name: string }) => void;
  b: (x: { age: number }) => void;
}>;
// P = { name: string } & { age: number } = { name: string; age: number }
// Object intersection — property lar birlashadi, foydali natija!
```

**Nima uchun:**
- **Covariant** (chiqish) — funksiya string HAM number HAM qaytarishi mumkin → union
- **Contravariant** (kirish) — argument **barcha** candidate larga mos bo'lishi kerak → intersection

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `Exclude` va `Extract` qo'lda implement qiling (Daraja: Middle)

**Savol:** Distributive conditional type ishlatib `MyExclude` va `MyExtract` yozing. Har birining distributive jarayonini qadam-baqadam tushuntiring:

```typescript
// Implement qiling:
// MyExclude<"a" | "b" | "c", "a" | "c"> → "b"
// MyExtract<string | number | boolean, string | boolean> → string | boolean
```

<details>
<summary>Yechim</summary>

```typescript
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;

type A = MyExclude<"a" | "b" | "c", "a" | "c">;
// Distributive:
// = ("a" extends "a"|"c" ? never : "a") | ("b" extends "a"|"c" ? never : "b") | ("c" extends "a"|"c" ? never : "c")
// = never | "b" | never = "b" ✅

type B = MyExtract<string | number | boolean, string | boolean>;
// = (string extends string|boolean ? string : never) | (number extends ...) | (boolean extends ...)
// = string | never | boolean = string | boolean ✅
```

Distributive tufayli union ning **har bir member** alohida tekshiriladi. `never` union da yo'qoladi.

</details>

### 2. `ReturnType` va `Parameters` qo'lda implement qiling (Daraja: Middle+)

**Savol:** `infer` keyword ishlatib yozing. Overloaded function bilan nima bo'lishini ham tushuntiring:

```typescript
// Implement qiling:
// MyReturnType<() => string> → string
// MyParameters<(a: string, b: number) => void> → [a: string, b: number]
```

<details>
<summary>Yechim</summary>

```typescript
type MyReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : any;

type MyParameters<T extends (...args: any) => any> =
  T extends (...args: infer P) => any ? P : never;

type A = MyReturnType<() => string>;                    // string
type B = MyReturnType<(x: number) => boolean>;          // boolean
type C = MyParameters<(a: string, b: number) => void>;  // [a: string, b: number]
type D = MyParameters<() => void>;                       // []
```

**Overloaded function nuance:**

```typescript
declare function overloaded(x: string): string;
declare function overloaded(x: number): number;

type E = MyReturnType<typeof overloaded>;
// = number — faqat OXIRGI overload signature ishlatiladi!
```

**Qo'shimcha utility lar:**

```typescript
type FirstParam<T extends (...args: any) => any> =
  T extends (first: infer F, ...rest: any[]) => any ? F : never;

type MyInstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : never;
```

</details>

### 3. `never` va `any` trap lar — output savoli (Daraja: Middle+)

**Savol:** Har bir type ning natijasini ayting va **nima uchun** ekanini tushuntiring:

```typescript
type Test<T> = T extends string ? "yes" : "no";

type A = Test<string>;
type B = Test<number>;
type C = Test<never>;
type D = Test<any>;
type E = Test<string | number>;
```

Va `IsNever` va `IsAny` ni to'g'ri implement qiling.

<details>
<summary>Yechim</summary>

```typescript
type A = "yes";           // string extends string ✅
type B = "no";            // number extends string ❌
type C = never;           // ❗ never = "empty union" → 0 member → never
type D = "yes" | "no";   // ❗ any → ham true ham false branch
type E = "yes" | "no";   // Distributive: Test<string> | Test<number>
```

**`never`** — bo'sh union. Distributive da 0 ta member distribute bo'ladi → natija `never`.
**`any`** — maxsus: ham `string` ga assignable, ham emas → ikkala branch union.

**To'g'ri tekshirish:**

```typescript
// IsNever — non-distributive
type IsNever<T> = [T] extends [never] ? true : false;
type F = IsNever<never>;  // true ✅
type G = IsNever<string>; // false ✅

// IsAny — 1 & T trick
type IsAny<T> = 0 extends 1 & T ? true : false;
// 1 & any = any → 0 extends any = true
// 1 & string = never → 0 extends never = false
type H = IsAny<any>;     // true
type I = IsAny<string>;  // false
```

</details>

### 4. Recursive conditional — DeepAwaited va DeepFlatten (Daraja: Middle+)

**Savol:** Recursive conditional type ishlatib `DeepAwaited` va `DeepFlatten` yozing:

```typescript
// DeepAwaited<Promise<Promise<string>>> → string
// DeepFlatten<string[][][]> → string
```

<details>
<summary>Yechim</summary>

```typescript
type DeepAwaited<T> =
  T extends Promise<infer U>
    ? DeepAwaited<U>
    : T;

type A = DeepAwaited<Promise<string>>;                   // string
type B = DeepAwaited<Promise<Promise<number>>>;          // number
type C = DeepAwaited<Promise<Promise<Promise<boolean>>>>; // boolean
type D = DeepAwaited<string>;                            // string

type DeepFlatten<T> =
  T extends (infer U)[]
    ? DeepFlatten<U>
    : T;

type E = DeepFlatten<string[]>;     // string
type F = DeepFlatten<string[][]>;   // string
type G = DeepFlatten<string[][][]>; // string
type H = DeepFlatten<number>;       // number
```

**Tricky holatlar:**

```typescript
type I = DeepFlatten<(string | number)[]>;
// Distributive: DeepFlatten<string> | DeepFlatten<number> = string | number

type J = DeepFlatten<never[]>;
// never[] → U = never → DeepFlatten<never> → distributive → never

type K = DeepFlatten<any[]>;
// any → maxsus holat → any
```

TS 4.5+ da tail-recursive optimization — 1000+ daraja qo'llab-quvvatlaydi.

</details>

### 5. Distributive to'xtatish — IsNever, IsUnion (Daraja: Senior)

**Savol:** Non-distributive pattern ishlatib quyidagi utility type larni yozing:

```typescript
// IsNever<never> → true, IsNever<string> → false
// IsUnion<string | number> → true, IsUnion<string> → false
// NonDistributiveExclude — butun union ni tekshirish
```

<details>
<summary>Yechim</summary>

```typescript
// IsNever — [T] wrapper bilan non-distributive
type IsNever<T> = [T] extends [never] ? true : false;

type A = IsNever<never>;  // true
type B = IsNever<string>; // false

// IsUnion — distributive ichida non-distributive
type IsUnion<T, Copy = T> =
  T extends any
    ? [Copy] extends [T] ? false : true
    : never;

type C = IsUnion<string>;          // false
type D = IsUnion<string | number>; // true
// Qanday ishlaydi:
// T = string | number, Copy = string | number
// Distributive: T = string → [string | number] extends [string] ? false : true → true
// T = number → [string | number] extends [number] ? false : true → true
// Natija: true | true = true
//
// T = string, Copy = string
// T = string → [string] extends [string] ? false : true → false

// NonDistributiveExclude — butun union bitta unit sifatida
type NDExclude<T, U> = [T] extends [U] ? never : T;

type E = NDExclude<string | number, string | number>; // never — butun union mos
type F = NDExclude<string | number, string>;           // string | number — butun union mos emas
// Oddiy Exclude<string | number, string> = number (distributive)
```

**Qoida:** `[T]` wrapper — distribution ni to'xtatish. `T extends any` ichida `[Copy] extends [T]` — distributive ichida non-distributive tekshiruv.

</details>

### 6. `UnionToIntersection<U>` implement qiling (Daraja: Senior)

**Savol:** Contravariant `infer` pozitsiyasidan foydalanib, union type ni intersection ga aylantiring:

```typescript
// UnionToIntersection<{ a: 1 } | { b: 2 }> → { a: 1 } & { b: 2 }
```

<details>
<summary>Yechim</summary>

```typescript
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void
    ? I
    : never;

type A = UnionToIntersection<{ a: 1 } | { b: 2 }>;
// = { a: 1 } & { b: 2 } = { a: 1; b: 2 }
```

**Qadam-baqadam:**

```
1. U = { a: 1 } | { b: 2 }

2. Distributive: U extends any ? (x: U) => void : never
   = ((x: { a: 1 }) => void) | ((x: { b: 2 }) => void)

3. Bu union funksiya extends (x: infer I) => void
   infer I — contravariant pozitsiyada (parameter)

4. Candidates: { a: 1 }, { b: 2 }
   Contravariant → INTERSECTION: { a: 1 } & { b: 2 }
```

**Kalit tushuncha:** Funksiya parameter lari contravariant. Union funksiya lardan `infer` qilganda parameter type lari **intersection** hosil qiladi. Bu variance mexanizmiga asoslangan.

**Cheklov:** Primitive union da intersection `never` beradi (`string & number = never`). Faqat object type lar uchun foydali.

</details>

---

## Xulosa

- Conditional type — `T extends U ? X : Y`, `extends` = assignability check
- `infer` — type extraction, covariant da union, contravariant da intersection
- Distributive — union har member ga alohida qo'llanadi. `[T]` wrapper to'xtatadi
- `never` = bo'sh union → distributive da `never`. `any` → ikkala branch
- `infer extends` (TS 4.7+) — string dan number/boolean parse
- Deferred conditional — generic body da resolve bo'lmaydi, overload yechim
- `IsNever` → `[T] extends [never]`, `IsAny` → `0 extends 1 & T`
- `UnionToIntersection` — contravariant infer bilan union → intersection

[Asosiy bo'limga qaytish →](../12-conditional-types.md)
