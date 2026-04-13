# Interview: Advanced Generics

> Recursive types, variadic tuples, generic inference edge cases, higher-kinded types va advanced generic patterns bo'yicha interview savollari. Conditional types, mapped types, template literal types — alohida bo'limlarda (12, 13, 14).

---

## Nazariy savollar

### 1. Recursive type nima? Qachon ishlatiladi?

<details>
<summary>Javob</summary>

Recursive type — type o'zini o'zi reference qilishi. Daraxt strukturalar, deeply nested object lar va type-level algoritmlar uchun kerak:

```typescript
// DeepReadonly — barcha nested property lar readonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends (...args: any[]) => any
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};

// JSON type — recursive union
type JSONValue =
  | string | number | boolean | null
  | JSONValue[]
  | { [key: string]: JSONValue };
```

**Depth limit:** TS non-tail-recursive type larda ~50, tail-recursive da ~1000 (TS 4.5+) daraja qo'llab-quvvatlaydi. Undan chuqur bo'lsa `Type instantiation is excessively deep` xatosi.

```typescript
// Counter tuple bilan depth limit
type DeepFlatten<T, D extends unknown[] = []> =
  D["length"] extends 10 ? T :
  T extends (infer U)[]
    ? DeepFlatten<U, [...D, unknown]>
    : T;
```

Real-world: `DeepPartial` — ORM da qisman update. `DeepReadonly` — immutable state (Redux, Zustand).

</details>

### 2. Variadic tuple types nima?

<details>
<summary>Javob</summary>

Variadic tuple types (TS 4.0+) — tuple type larda spread operator (`...`) ishlatish. Tuple larni birlashtirish, ajratish va transform qilish uchun:

```typescript
// Ikki tuple ni birlashtirish
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];
type A = Concat<[1, 2], [3, 4]>; // [1, 2, 3, 4]

// Tuple operatsiyalari
type Tail<T extends unknown[]> = T extends [unknown, ...infer Rest] ? Rest : [];
type Push<T extends unknown[], V> = [...T, V];
type Unshift<T extends unknown[], V> = [V, ...T];

type B = Tail<[1, 2, 3]>;    // [2, 3]
type C = Push<[1, 2], 3>;    // [1, 2, 3]
type D = Unshift<[2, 3], 1>; // [1, 2, 3]
```

Funksiya da type-safe concat:

```typescript
function concat<T extends unknown[], U extends unknown[]>(
  a: [...T], b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}

const result = concat([1, "hello"] as [number, string], [true] as [boolean]);
// result: [number, string, boolean]
```

</details>

### 3. `never` va distributive conditional types — nima uchun tricky?

<details>
<summary>Javob</summary>

`never` — **bo'sh union type**. Distributive conditional type da bo'sh union berilsa — hech qanday member yo'q — natija ham `never`:

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<never>;  // never — true ham false ham emas!
type B = IsString<string>; // true
```

Nima uchun? Distributive conditional type union ning **har bir member** iga alohida qo'llanadi. `never` = 0 ta member → 0 ta natija → `never`.

**Yechim — non-distributive** `[T]` wrapper:

```typescript
type IsNever<T> = [T] extends [never] ? true : false;

type C = IsNever<never>;  // true ✅
type D = IsNever<string>; // false ✅
```

Yana bir tricky holat: `never extends string` directly (generic emas) → `true` (never bottom type, barchaga assignable). Lekin `T extends string` generic da `T = never` → distributive → `never`.

Batafsil conditional types haqida — [interview/12](12-conditional-types.md).

</details>

### 4. Generic inference edge cases — deferred conditional nima?

<details>
<summary>Javob</summary>

Generic funksiya body'da conditional type **resolve bo'lmaydi** — TS generic T ning aniq qiymatini bilmasdan conditional ni evaluate qila olmaydi:

```typescript
function example<T extends string | number>(value: T): T extends string ? string[] : number {
  if (typeof value === "string") {
    return value.split(","); // ❌ TS buni string[] deb bilmaydi!
    // Error: Type 'string[]' is not assignable to type 'T extends string ? string[] : number'
  }
  return 0 as any; // as assertion kerak
}
```

Nima uchun? TS generic body'da `T` hali aniq emas — shuning uchun conditional type **deferred** (kechiktirilgan) holatda qoladi. Runtime `typeof` tekshiruvi TS ning type-level conditional ni resolve qilmaydi.

**Yechimlar:**

1. **Function overloads** — ko'p hollarda aniqroq:
```typescript
function example(value: string): string[];
function example(value: number): number;
function example(value: string | number): string[] | number {
  if (typeof value === "string") return value.split(",");
  return 0;
}
```

2. **Type assertion** — body ichida `as`:
```typescript
function example<T extends string | number>(
  value: T
): T extends string ? string[] : number {
  if (typeof value === "string") {
    return value.split(",") as T extends string ? string[] : number;
  }
  return 0 as T extends string ? string[] : number;
}
```

</details>

### 5. Higher-Kinded Types (HKT) — TS da qo'llab-quvvatlanadimi?

<details>
<summary>Javob</summary>

HKT — type parameter ning o'zi ham generic bo'lishi ("Generic ning generic-i"). Haskell, Scala da to'liq qo'llab-quvvatlanadi. TS da **to'g'ridan-to'g'ri qo'llab-quvvatlanmaydi**:

```typescript
// Bu yozib bo'lmaydi:
// type Functor<F<_>> = { map: <A, B>(fa: F<A>, fn: (a: A) => B) => F<B> }
// ❌ F<_> syntax yo'q
```

Workaround pattern lar:

```typescript
// 1. Overload — har bir container uchun alohida
function map<T, U>(container: T[], fn: (v: T) => U): U[];
function map<T, U>(container: Promise<T>, fn: (v: T) => U): Promise<U>;
function map<T, U>(container: T[] | Promise<T>, fn: (v: T) => U) {
  if (Array.isArray(container)) return container.map(fn);
  return container.then(fn);
}

// 2. Interface mapping (fp-ts pattern)
interface URItoKind<A> {
  Array: A[];
  Promise: Promise<A>;
}
type Kind<URI extends keyof URItoKind<any>, A> = URItoKind<A>[URI];
```

TS da HKT bo'yicha open issue bor (GitHub #1213, 2014 dan), lekin hali plan yo'q. Amalda overload va concrete type lar bilan ishlash yetarli.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Flatten — output savoli (Daraja: Middle+)

**Savol:** Har bir type ning natijasini ayting:

```typescript
type Flatten<T> = T extends (infer U)[] ? U : T;

type A = Flatten<string[]>;
type B = Flatten<number>;
type C = Flatten<(string | number)[]>;
type D = Flatten<string[][]>;
```

<details>
<summary>Yechim</summary>

```typescript
type A = string;
// string[] extends (infer U)[] → ✅ U = string

type B = number;
// number extends (infer U)[] → ❌ → T = number

type C = string | number;
// (string | number)[] extends (infer U)[] → ✅ U = string | number

type D = string[];
// string[][] extends (infer U)[] → ✅ U = string[]
// Faqat bitta daraja! string[][] ning element type i string[]
```

`Flatten` faqat bitta daraja flatten qiladi. Chuqur flatten kerak bo'lsa recursive:

```typescript
type DeepFlatten<T> = T extends (infer U)[] ? DeepFlatten<U> : T;
type E = DeepFlatten<string[][]>; // string
```

</details>

### 2. GetReturnType — output va distributive (Daraja: Middle+)

**Savol:** Har bir type ning natijasini ayting. E da nima uchun shunday?

```typescript
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = GetReturnType<typeof Math.random>;
type B = GetReturnType<typeof parseInt>;
type C = GetReturnType<string>;
type D = GetReturnType<() => void>;
type E = GetReturnType<(() => string) | (() => number)>;
```

<details>
<summary>Yechim</summary>

```typescript
type A = number;         // Math.random: () => number → R = number
type B = number;         // parseInt: (s: string, r?: number) => number → R = number
type C = never;          // string funksiya emas → false branch → never
type D = void;           // () => void → R = void
type E = string | number; // ❗ Distributive!
```

**E tushuntirish:** `(() => string) | (() => number)` — union. T naked type parameter, shuning uchun **distributive**:

```
GetReturnType<(() => string) | (() => number)>
= GetReturnType<() => string> | GetReturnType<() => number>
= string | number
```

Har bir union member alohida infer bo'ladi va natijalar qayta union qilinadi.

</details>

### 3. `never` tricky behavior — output savoli (Daraja: Senior)

**Savol:** Har bir type ning natijasini ayting va tushuntiring:

```typescript
type A = never extends string ? true : false;
type B<T> = T extends string ? true : false;
type C = B<never>;
type D = [never] extends [string] ? true : false;
type E<T> = [T] extends [string] ? true : false;
type F = E<never>;
```

<details>
<summary>Yechim</summary>

```typescript
type A = true;   // Direct (generic emas) — never bottom type, barchaga assignable
type C = never;  // B<never> — distributive: never = bo'sh union → 0 ta member → never
type D = true;   // Direct, [never] extends [string] — non-distributive, never assignable
type F = true;   // E<never> — [T] wrapper, non-distributive → [never] extends [string] → true
```

| | Generic | Distributive | Natija |
|---|---|---|---|
| A | ❌ direct | — | `true` |
| C (B\<never\>) | ✅ naked T | ✅ bo'sh union | `never` |
| D | ❌ direct | — | `true` |
| F (E\<never\>) | ✅ lekin `[T]` | ❌ non-dist | `true` |

**Kalit qoida:** `never` naked generic da = bo'sh union → distributive → `never`. `[T]` wrapper bilan non-distributive → oddiy check → `true`.

</details>

### 4. Variadic tuple operations (Daraja: Senior)

**Savol:** Quyidagi tuple utility type larini implement qiling:

```typescript
// Head<T> — birinchi element type
// Head<[string, number, boolean]> = string

// Last<T> — oxirgi element type
// Last<[string, number, boolean]> = boolean

// Init<T> — oxirgisiz (Tail ning teskari)
// Init<[string, number, boolean]> = [string, number]

// Reverse<T> — teskari tartib
// Reverse<[1, 2, 3]> = [3, 2, 1]
```

<details>
<summary>Yechim</summary>

```typescript
// Head — birinchi element
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;

type H1 = Head<[string, number, boolean]>; // string
type H2 = Head<[]>;                         // never

// Last — oxirgi element
type Last<T extends unknown[]> = T extends [...unknown[], infer L] ? L : never;

type L1 = Last<[string, number, boolean]>; // boolean
type L2 = Last<[42]>;                       // 42

// Init — oxirgisiz
type Init<T extends unknown[]> = T extends [...infer I, unknown] ? I : [];

type I1 = Init<[string, number, boolean]>; // [string, number]
type I2 = Init<[42]>;                       // []

// Reverse — teskari tartib (recursive)
type Reverse<T extends unknown[]> =
  T extends [infer H, ...infer Rest]
    ? [...Reverse<Rest>, H]
    : [];

type R1 = Reverse<[1, 2, 3]>;     // [3, 2, 1]
type R2 = Reverse<[string, number]>; // [number, string]
type R3 = Reverse<[]>;              // []
```

**Tushuntirish:**

- `infer H` + `...infer Rest` — tuple destructuring, type darajasida
- `[...Reverse<Rest>, H]` — variadic spread, recursive
- `Reverse` tail-recursive emas — katta tuple larda depth limit ga urishi mumkin
- TS 4.5+ da tail-recursive optimization bor — `type Reverse<T, Acc = []>` bilan 1000+ element ishlaydi

</details>

### 5. Type-safe FormBuilder pattern (Daraja: Senior)

**Savol:** `FormBuilder` yozing — `addField` chaqirilganda yangi field type ga qo'shilsin, `build()` barcha field larning type ini qaytarsin:

```typescript
// Kutilgan natija:
// new FormBuilder()
//   .addField("name", "")       → Fields = { name: string }
//   .addField("age", 0)         → Fields = { name: string; age: number }
//   .addField("active", true)   → Fields = { name: string; age: number; active: boolean }
//   .build()                    → { name: string; age: number; active: boolean }
```

<details>
<summary>Yechim</summary>

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

const form = new FormBuilder()
  .addField("name", "")
  .addField("age", 0)
  .addField("active", true)
  .build();

// form: { name: string; age: number; active: boolean }
form.name;    // string ✅
form.age;     // number ✅
form.active;  // boolean ✅
// form.email; // ❌ Error: 'email' does not exist
```

**Tushuntirish:**

1. `FormBuilder<Fields>` — generic class, accumulated type
2. Boshlang'ich `Fields = {}` (bo'sh object)
3. `addField<Name, Type>` → return: `FormBuilder<Fields & Record<Name, Type>>`
4. `&` intersection — yangi field ni mavjud Fields ga qo'shadi
5. `build()` — accumulated `Fields` type qaytaradi
6. `as unknown as` — TS class instance type ni runtime da o'zgartirib bo'lmaydi, shuning uchun assertion kerak

Bu "accumulator pattern" — functional fold/reduce ning type-level ekvivalenti. Zod, Drizzle ORM, tRPC shu pattern ishlatadi.

</details>

---

## Xulosa

- Recursive types — deeply nested data uchun. Depth limit: ~50 (non-tail), ~1000 (tail-recursive TS 4.5+)
- Variadic tuples — tuple spread, destructuring, transform. `infer` + `...` bilan
- `never` distributive da bo'sh union — `[T]` wrapper bilan oldini olish
- Deferred conditional — generic body da resolve bo'lmaydi, overload yoki assertion kerak
- HKT — TS da direct qo'llab-quvvatlanmaydi, overload yoki URI pattern workaround
- Conditional types batafsil — [interview/12](12-conditional-types.md)
- Mapped types batafsil — [interview/13](13-mapped-types.md)
- Template literal types — [interview/14](14-template-literal-types.md)

[Asosiy bo'limga qaytish →](../09-advanced-generics.md)
