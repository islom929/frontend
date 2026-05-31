# Bo'lim 13: Mapped Types Chuqur

> Mapped types — TypeScript'da mavjud object type'ning har bir property'sini **transform** qilish mexanizmi. `{ [K in keyof T]: NewType }` sintaksisi bilan barcha key'larni iterate qilish, property modifier'lar (`+?`, `-?`, `+readonly`, `-readonly`) bilan optional/readonly holatini o'zgartirish, key remapping (`as`) bilan key'larni qayta nomlash yoki filtrlash, va conditional types bilan birgalikda qudratli type transformation'lar yaratish.
>
> Bu bo'lim [09-advanced-generics.md](09-advanced-generics.md)'dagi asosiy tushunchalarni chuqurlashtirib, `Partial`, `Required`, `Readonly`, `Record` kabi built-in utility type'larning ichki implementation'ini, homomorphic mapped type'lar tushunchasini, va real-world pattern'larni yoritadi.

---

## Mundarija

- [Mapped Type Asoslari](#mapped-type-asoslari)
- [`keyof` va Index Access — Mapped Type Building Blocks](#keyof-va-index-access--mapped-type-building-blocks)
- [Property Modifiers — `+?`, `-?`, `+readonly`, `-readonly`](#property-modifiers---readonly--readonly)
- [Built-in Utility Types Ichidan](#built-in-utility-types-ichidan)
- [Key Remapping — `as` Keyword (TS 4.1+)](#key-remapping--as-keyword-ts-41)
- [Mapped Types + Conditional Types](#mapped-types--conditional-types)
- [Custom Mapped Types — Real-World Patterns](#custom-mapped-types--real-world-patterns)
- [Homomorphic va Non-Homomorphic Mapped Types](#homomorphic-va-non-homomorphic-mapped-types)
- [Mapped Types va Union Types](#mapped-types-va-union-types)
- [Performance va Best Practices](#performance-va-best-practices)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Mapped Type Asoslari

### Nazariya

Mapped type — mavjud object type'ning har bir property'sini **transform** qilish mexanizmi. Asosiy sintaksis uchta qismdan iborat:

```typescript
{
  [K in Keys]: ValueType
//  │    │         │
//  │    │         └── Har property'ning yangi type'i
//  │    └── Iterate qilinadigan key union (odatda `keyof T`)
//  └── Har iteration'da joriy key
}
```

**Ma'nosi:** "Keys'dagi har bir K uchun — K nomli property yarat, type'i ValueType bo'lsin."

Bu JavaScript'ning `for...in` loop'ining type-level ekvivalenti:

```
Runtime:  for (const key in obj) { result[key] = transform(obj[key]); }
Type:     { [K in keyof T]:        TransformedType<T[K]> }
```

**Mapped type komponentlari:**

1. **Key source** — `K in keyof T` yoki `K in SomeUnion` — iterate qilinadigan key'lar
2. **Value mapping** — `:` dan keyingi type — `K` va `T[K]`'ga bog'liq bo'lishi mumkin
3. **Modifier'lar** — `readonly`, `?` — qo'shish yoki olib tashlash
4. **Key remapping** — `as NewKey` (TS 4.1+) — key'ni o'zgartirish

**Homomorphic pattern:** Agar mapped type `{ [K in keyof T]: ... }` shaklida bo'lsa (T — type parameter), bu **homomorphic** mapped type. Kompilator original type'ning modifier'larini (optional, readonly) avtomatik saqlaydi. Key source `keyof T` bo'lgan, yoki key source `T` constraint'i `keyof X` bo'lgan type parameter (`[P in K], K extends keyof X`) bo'lgan holatlar — ikkalasi ham homomorphic, shu sababli `Pick` ham modifier saqlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator mapped type'ni quyidagicha qayta ishlaydi:

```
Input: { [K in keyof T]: T[K] }
T = { name: string; age: number; active: boolean }

1. keyof T → "name" | "age" | "active"

2. Har key uchun property yaratish:
   K = "name"   → { name: T["name"] }   → { name: string }
   K = "age"    → { age: T["age"] }     → { age: number }
   K = "active" → { active: T["active"] } → { active: boolean }

3. Natijalarni birlashtirish:
   { name: string; age: number; active: boolean }
```

**Internal mexanizm:** Kompilator mapped type node'ni qayta ishlaganda, key source'ni union'ga resolve qiladi va har bir member uchun alohida property instantiation yaratadi. Har property uchun value type alohida evaluate bo'ladi — agar conditional type yoki generic ishlatilsa, har iteration'da mustaqil computation.

**Homomorphic detection:** Kompilator mapped type'ni qayta ishlayotganda key source'ni tekshiradi. Agar key source aynan `keyof T` bo'lsa (T — qandaydir type parameter) — yoki key source `keyof T` bilan cheklangan type parameter bo'lsa (`Pick`'dagi `[P in K], K extends keyof T`) — mapped type homomorphic deb belgilanadi va T'dagi modifier'lar (readonly, optional) natijaga tarqatiladi. Agar key source boshqa narsa bo'lsa (string literal union, `keyof any`, va boshqalar), non-homomorphic — modifier'lar yo'qoladi.

**Runtime'da iz yo'q:** Mapped types compile-time construct. JavaScript output'da hech qanday iz qolmaydi — barcha transform'lar type system ichida. Runtime immutability yoki validation uchun bu type'lar yaramaydi — faqat compile-time type safety.

**Mapped vs distributive conditional farqi:** Mapped type'ga union berilsa, distributive conditional'dan farqli ishlaydi. `{ [K in keyof (A | B)]: ... }` — kompilator `keyof (A | B)` ni hisoblaydi, bu **shared keys intersection**. Conditional type'dagi kabi union member'larga split qilinmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy identity mapped type
type Identity<T> = {
  [K in keyof T]: T[K];
};

interface User {
  name: string;
  age: number;
}

type UserCopy = Identity<User>;
// { name: string; age: number }

// 2. Value transformation — barcha Promise'ga wrap
type Promisified<T> = {
  [K in keyof T]: Promise<T[K]>;
};

type PromisifiedUser = Promisified<User>;
// { name: Promise<string>; age: Promise<number> }

// 3. Value type'ni o'zgartirish — barcha boolean
type BooleanFlags<T> = {
  [K in keyof T]: boolean;
};

type UserFlags = BooleanFlags<User>;
// { name: boolean; age: boolean }

// 4. Box pattern
type Boxed<T> = {
  [K in keyof T]: { value: T[K] };
};

type BoxedUser = Boxed<User>;
// { name: { value: string }; age: { value: number } }

// 5. Nullable
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type NullableUser = Nullable<User>;
// { name: string | null; age: number | null }

// 6. Function wrapper — har property uchun getter
type AsFunctions<T> = {
  [K in keyof T]: () => T[K];
};

type UserFns = AsFunctions<User>;
// { name: () => string; age: () => number }

// 7. Homomorphic — modifier preservation
interface Config {
  readonly host: string;
  port?: number;
}

type ConfigCopy = Identity<Config>;
// { readonly host: string; port?: number }
// ✅ readonly va ? saqlanadi (keyof T dan key)
```

</details>

---

## `keyof` va Index Access — Mapped Type Building Blocks

### Nazariya

`keyof` va `T[K]` — mapped type'larning asosiy building block'lari. Bu operator'lar [08-generics.md](08-generics.md)'da asos sifatida yoritilgan. Bu yerda esa ularning **mapped type kontekstida** ishlatilishiga fokus qilamiz.

**`keyof T` — type'ning barcha key'lari:**

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type UserKeys = keyof User; // "name" | "age" | "email"
```

**`T[K]` — type'ning K key'idagi property type'ini olish:**

```typescript
type NameType = User["name"];     // string
type AllValues = User[keyof User]; // string | number
```

**Mapped type kontekstida qo'llanilishi:**

```typescript
// keyof T — iteration source
type Transform<T> = { [K in keyof T]: T[K] };

// T[K] — value type access
type Values<T> = { [K in keyof T]: T[K] }[keyof T];
// Object → union of value types

// Kombinatsiya — common pattern
type KeyOfType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`keyof`'ning turli type'lar bilan xatti-harakati:**

```typescript
// Object type — declared key'lar
type A = keyof { a: 1; b: 2 };          // "a" | "b"

// Index signature bilan
type B = keyof { [key: string]: unknown }; // string | number
// Diqqat: string | number — JavaScript'da obj[0] === obj["0"]
// Number index'lar ham string'ga aylanadi

type C = keyof { [key: number]: unknown }; // number

// Array
type D = keyof string[]; // number | "length" | "toString" | ...
// Array methods + number index

// Primitive'lar
type E = keyof string;  // number | typeof Symbol.iterator | "length" | "toString" | ...
// String.prototype'ning methodlari

// never va unknown
type F = keyof never;   // string | number | symbol (bottom type)
type G = keyof unknown; // never (unknown'da hech narsa accessible emas)

// any
type H = keyof any;     // string | number | symbol
```

**Nima uchun index signature `string | number`:** JavaScript'da object key'lar string, number, yoki symbol bo'lishi mumkin. Number key'lar internal'da string'ga aylantiriladi (`obj[0]` va `obj["0"]` teng). TypeScript bu behavior'ni aks ettirish uchun `string | number` qaytaradi — developer number index ham ishlata olishi uchun.

**`T[keyof T]` pattern:** Object'ning barcha value type'larini union sifatida olish. Bu conditional type bilan birga ishlatilganda juda kuchli:

```typescript
type Values<T> = T[keyof T];
// { a: 1; b: 2 } → 1 | 2
```

**Array `T[number]`:** Array/tuple element type'ini olish uchun:

```typescript
type Arr = string[];
type El = Arr[number]; // string

type Tuple = [1, "a", true];
type AnyEl = Tuple[number]; // 1 | "a" | true (union)

// as const bilan literal saqlash
const colors = ["red", "green", "blue"] as const;
type Color = (typeof colors)[number]; // "red" | "green" | "blue"
```

**Tuple aniq index access:** Tuple'larda aniq index ishlatilishi mumkin:

```typescript
type Tuple = [string, number, boolean];
type First = Tuple[0];  // string
type Second = Tuple[1]; // number
```

Array'da aniq index ishlatilsa (`Arr[0]`) — element type qaytadi (`string[][0]` → `string`). Bu indexed access — `keyof`'dan farqli. `keyof string[]` esa `number | "length" | ...` beradi, chunki runtime'da `array[0]` va `array["0"]` aynan bir xil property (JavaScript number key'larni string'ga aylantiradi), shuning uchun `keyof` index qismida `number` ham keladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. keyof + mapped type — asosiy pattern
type Partial1<T> = {
  [K in keyof T]?: T[K];
};

// 2. Value type union olish
type UserValues<T> = T[keyof T];

interface User {
  name: string;
  age: number;
  isAdmin: boolean;
}

type AllUserValues = UserValues<User>; // string | number | boolean

// 3. Filter by value type — famous pattern
type KeysOfType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

type StringKeys = KeysOfType<User, string>;  // "name"
type NumberKeys = KeysOfType<User, number>;   // "age"
type BoolKeys = KeysOfType<User, boolean>;     // "isAdmin"

// 4. Pick by value type
type PickByValue<T, V> = Pick<T, KeysOfType<T, V>>;

type UserStrings = PickByValue<User, string>;
// { name: string }

// 5. Array index access
const roles = ["admin", "user", "guest"] as const;
type Role = (typeof roles)[number]; // "admin" | "user" | "guest"

// 6. Nested path
interface AppConfig {
  server: {
    host: string;
    port: number;
  };
  db: {
    url: string;
  };
}

type ServerType = AppConfig["server"];      // { host: string; port: number }
type HostType = AppConfig["server"]["host"]; // string

// 7. Enum-like constant
const STATUS = {
  pending: 0,
  active: 1,
  inactive: 2,
} as const;

type StatusKey = keyof typeof STATUS;           // "pending" | "active" | "inactive"
type StatusValue = (typeof STATUS)[StatusKey];  // 0 | 1 | 2

// 8. keyof + constraint — type-safe property access
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { name: "Ali", age: 25, isAdmin: false };
getProperty(user, "name"); // string (typed)
getProperty(user, "age");  // number (typed)
```

</details>

> **Cross-references:** `keyof` va index access asoslari — [08-generics.md](08-generics.md#keyof-bilan-generics). Conditional filtering — [12-conditional-types.md](12-conditional-types.md).

---

## Property Modifiers — `+?`, `-?`, `+readonly`, `-readonly`

### Nazariya

Mapped type'da property modifier'larni **qo'shish** (`+`) yoki **olib tashlash** (`-`) mumkin.

**To'rt modifier operatsiya:**

| Modifier | Ma'nosi | Misol |
|----------|---------|-------|
| `+?` yoki `?` | Optional qilish | `[K in keyof T]?: T[K]` |
| `-?` | Required qilish (optional olib tashlash) | `[K in keyof T]-?: T[K]` |
| `+readonly` yoki `readonly` | Readonly qilish | `readonly [K in keyof T]: T[K]` |
| `-readonly` | Mutable qilish | `-readonly [K in keyof T]: T[K]` |

**`+` default** — yozmasangiz `+` deb qabul qilinadi:

```typescript
[K in keyof T]?: T[K]         // = +? (optional qo'shish)
readonly [K in keyof T]: T[K] // = +readonly (readonly qo'shish)
```

**Muhim — `-?` va `| undefined` farqi:** `-?` (TS 2.8+)'da optional marker'ni va `undefined` union member'ni **ikkalasini ham** olib tashlaydi. Lekin property optional emas (faqat `| undefined` bor) bo'lsa, `-?` hech narsa qilmaydi — `| undefined` qoladi.

```typescript
interface Example {
  a: string;             // required
  b?: string;            // optional (= string | undefined)
  c: string | undefined; // required, lekin undefined bo'lishi mumkin
}

type Req = Required<Example>;
// {
//   a: string;
//   b: string;             // -? undefined ham olib tashladi
//   c: string | undefined; // -? faqat ? olib tashlaydi, | undefined qoladi
// }
```

<details>
<summary><strong>Under the Hood</strong></summary>

Modifier'lar kompilator ichida quyidagicha qayta ishlanadi:

```
Original: { name: string; age?: number; readonly id: number }

+? (Partial):
  { name?: string; age?: number; readonly id?: number }
  → Barcha property'lar optional bo'ldi

-? (Required):
  { name: string; age: number; readonly id: number }
  → age'dagi ? olib tashlandi

+readonly (Readonly):
  { readonly name: string; readonly age?: number; readonly id: number }
  → Barcha property'lar readonly bo'ldi

-readonly (Mutable):
  { name: string; age?: number; id: number }
  → id'dagi readonly olib tashlandi
```

**`-?` va `undefined` nuance'i:** TypeScript 2.8'dan boshlab `-?` modifier property type'idan `undefined`'ni ham olib tashlaydi. Oldin faqat optional marker'ni olib tashlar edi, lekin property type'da `| undefined` qolardi. Yangi behavior `Required<T>` kabi utility type'larning yaxshi ishlashi uchun kiritilgan.

```
TS 2.8+: -? → optional marker + "| undefined" ikkisi ham olib tashlanadi
TS 2.7 va oldin: -? → faqat optional marker olib tashlanadi
```

Lekin **faqat optional property'larda.** Agar property optional emas (explicit `: string | undefined`), `-?` hech narsa qilmaydi:

```typescript
type T1 = { a?: string };       // Required<T1> → { a: string }
type T2 = { a: string | undefined }; // Required<T2> → { a: string | undefined }
```

**Kombinatsiya — barcha modifier'lar:**

```typescript
type Strict<T> = {
  -readonly [K in keyof T]-?: T[K];
};

type Frozen<T> = {
  +readonly [K in keyof T]+?: T[K];
};
```

`Strict<T>` barcha readonly va optional'ni olib tashlaydi — to'liq mutable va required. `Frozen<T>` aksincha — hammasi immutable va optional.

**Modifier application order:** Kompilator modifier'larni quyidagi tartibda qo'llaydi:

1. `keyof T`'dan key'lar olinadi
2. Har key uchun property declaration yaratiladi (homomorphic bo'lsa — original modifier'lar ham)
3. Mapped type'da belgilangan modifier'lar qo'llaniladi (`+` qo'shadi, `-` olib tashlaydi)
4. Natija yangi mapped type

**Runtime'da iz yo'q:** Modifier'lar sof compile-time. JavaScript output'da hech qanday iz qolmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Partial — barcha optional
type Partial1<T> = {
  [K in keyof T]?: T[K];
};

interface Config {
  host: string;
  port: number;
  timeout: number;
}

type PartialConfig = Partial1<Config>;
// { host?: string; port?: number; timeout?: number }

// 2. Required — barcha required (optional olib tashlash)
type Required1<T> = {
  [K in keyof T]-?: T[K];
};

interface OptConfig {
  host?: string;
  port?: number;
  timeout?: number;
}

type RequiredConfig = Required1<OptConfig>;
// { host: string; port: number; timeout: number }

// 3. Readonly
type Readonly1<T> = {
  readonly [K in keyof T]: T[K];
};

type FrozenConfig = Readonly1<Config>;
// { readonly host: string; readonly port: number; readonly timeout: number }

// 4. Mutable — readonly olib tashlash
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

interface FrozenData {
  readonly id: number;
  readonly name: string;
}

type MutableData = Mutable<FrozenData>;
// { id: number; name: string }

// 5. Strict — hammasi required va mutable
type Strict<T> = {
  -readonly [K in keyof T]-?: T[K];
};

interface MixedType {
  readonly id: number;
  name?: string;
  readonly email?: string;
}

type StrictMixed = Strict<MixedType>;
// { id: number; name: string; email: string }
// readonly va ? ikkalasi ham olib tashlandi

// 6. Frozen — hammasi optional va readonly
type Frozen<T> = {
  readonly [K in keyof T]?: T[K];
};

type FrozenMixed = Frozen<Config>;
// { readonly host?: string; readonly port?: number; readonly timeout?: number }

// 7. -? va | undefined farqi
interface Example {
  a: string;
  b?: string;
  c: string | undefined;
}

type Req = Required<Example>;
// {
//   a: string;
//   b: string;             // -? optional + undefined olib tashladi
//   c: string | undefined; // -? hech narsa qilmaydi (optional emas)
// }

// Hammasini string qilish uchun NonNullable bilan
type FullyRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type FixedReq = FullyRequired<Example>;
// { a: string; b: string; c: string }
```

</details>

---

## Built-in Utility Types Ichidan

### Nazariya

TypeScript'ning built-in utility type'larining ko'pchiligi — mapped types bilan implement qilingan. Ularning ichini tushunish mapped types'ni chuqur o'zlashtirishga yordam beradi.

**Asosiy utility'lar:**

| Utility | Implementation | Homomorphic? |
|---------|----------------|:-------------:|
| `Partial<T>` | `{ [K in keyof T]?: T[K] }` | ✅ |
| `Required<T>` | `{ [K in keyof T]-?: T[K] }` | ✅ |
| `Readonly<T>` | `{ readonly [K in keyof T]: T[K] }` | ✅ |
| `Pick<T, K>` | `{ [P in K]: T[P] }` | ✅ |
| `Record<K, V>` | `{ [P in K]: V }` | ❌ |
| `Omit<T, K>` | `Pick<T, Exclude<keyof T, K>>` | ✅ (via Pick) |

**Homomorphic farqi:** `Partial`, `Required`, `Readonly`, `Pick` — homomorphic (modifier'lar saqlanadi). `Record` — non-homomorphic (modifier'lar yo'qoladi, chunki key source `K`, `keyof T` emas).

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator mapped type'ni evaluate qilganda, agar key source `keyof T` bo'lsa (homomorphic), T'dagi har property'ning modifier'lari natijaga tarqatiladi. Agar key source boshqa union bo'lsa (non-homomorphic), bu modifier'lar yo'qoladi.

```
Partial<User> evaluate jarayoni:

1. structured type members'ni oladi (User property'lari)
2. Har property uchun:
   K = "name" → mapped type body: { [K in keyof T]?: T[K] }
   → "name"?: string  (? qo'shiladi, homomorphic)
3. Natija: yangi ObjectType — barcha property'lar optional
```

**`Pick` va `Record` farqi:**

```typescript
// Pick — K keyof T extends, homomorphic
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Record — K keyof any, non-homomorphic
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

`Pick<T, K>`'da kompilator `K extends keyof T` constraint'ini ko'radi va T'dagi `K`'ga mos property'larning modifier'larini saqlaydi. `Record<K, V>`'da `K` — mustaqil type parameter, T bilan bog'liq emas — shuning uchun modifier propagation yo'q.

**Pedagogik muhim:** `Record<keyof T, V>` — developer'lar ko'p ishlatadigan pattern, lekin modifier'larni yo'qotadi:

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

type R = Record<keyof Config, string>;
// { host: string; port: string }
// ❌ readonly va ? yo'qoldi!

// Homomorphic alternative:
type H = { [K in keyof Config]: string };
// { readonly host: string; port?: string }
// ✅ modifier'lar saqlanadi
```

**`Omit` implementation:** `Omit<T, K>` — `Pick<T, Exclude<keyof T, K>>`'ga ekvivalent. Yoki to'g'ridan-to'g'ri:

```typescript
type MyOmit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
```

Bu yechim TS 4.1+ key remapping'dan foydalanadi — `as P extends K ? never : P` orqali K'ga mos key'larni filter qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Partial
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

// Real-world: update function
function updateUser(id: number, updates: MyPartial<User>): void {
  // ...
}
updateUser(1, { name: "Ali" }); // faqat name yangilash

// 2. Required
type MyRequired<T> = {
  [K in keyof T]-?: T[K];
};

interface OptUser {
  name?: string;
  age?: number;
}

type RequiredUser = MyRequired<OptUser>;
// { name: string; age: number }

// 3. Readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

type FrozenUser = MyReadonly<User>;
// { readonly name: string; readonly age: number; readonly email: string }

// 4. Pick
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type UserPreview = MyPick<User, "name" | "email">;
// { name: string; email: string }

// 5. Record
type MyRecord<K extends keyof any, T> = {
  [P in K]: T;
};

type StringMap = MyRecord<"a" | "b" | "c", number>;
// { a: number; b: number; c: number }

// 6. Omit (ikki implementation)
// Via Pick + Exclude
type MyOmit1<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// Via key remapping (TS 4.1+)
type MyOmit2<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

type UserWithoutEmail = MyOmit1<User, "email">;
// { name: string; age: number }

// 7. Homomorphic modifier preservation misol
interface StrictConfig {
  readonly host: string;
  port?: number;
}

// Pick — homomorphic, modifier saqlanadi
type HostOnly = Pick<StrictConfig, "host">;
// { readonly host: string }

// Record — non-homomorphic, modifier yo'qoladi
type RecordConfig = Record<keyof StrictConfig, string>;
// { host: string; port: string } — readonly va ? yo'q

// 8. Exclude/Extract — distributive conditional
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;

type A = MyExclude<"a" | "b" | "c", "a">; // "b" | "c"
type B = MyExtract<string | number, string>; // string
```

</details>

---

## Key Remapping — `as` Keyword (TS 4.1+)

### Nazariya

Key remapping — mapped type ichida key'ni **qayta nomlash**, **filtrlash**, yoki **transform** qilish. TS 4.1+'da `as` keyword bilan qo'shilgan. Bu mapped types'ning eng qudratli xususiyatlaridan biri.

**Syntax:**

```typescript
{
  [K in keyof T as NewKeyExpression]: T[K]
//                 ^^^^^^^^^^^^^^^^
//                 K asosida yangi key
}
```

**`as` dan keyingi expression:**

- **Template literal** — `` as `prefix${K}` `` — key'ni o'zgartirish
- **Conditional type** — `as K extends "id" ? never : K` — key filter
- **`never`** — property'ni butunlay o'chirish
- **Union** — bitta key'dan bir nechta key

**`as never` — property removal:** Key remapping'da `never` qaytarilsa, property butunlay o'chiriladi. Bu filter pattern'ning asosi:

```typescript
type OmitType<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};
```

**`string & K` intersection:** Template literal ichida `K` ishlatilganda, `K` `string | number | symbol` bo'lishi mumkin. `Capitalize`, `Uppercase`, `Lowercase` kabi string manipulation utility'lar faqat `string` qabul qiladi — shuning uchun `string & K` intersection kerak (`K`'ni faqat string subset'iga cheklaydi):

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
  //                             ^^^^^^^^^^ string bilan intersect
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator key remapping'ni quyidagicha qayta ishlaydi:

```
{ [K in keyof T as NewKey]: T[K] }

1. keyof T → union of keys
2. Har K uchun:
   a. NewKey = as expression evaluate
   b. Agar NewKey = never → property SKIP
   c. Agar NewKey = yangi nom → shu nom bilan property
3. Natija: yangi key'lar bilan yangi object type
```

**`never` filter mexanizmi:**

```
{ [K in keyof T as K extends "age" ? never : K]: T[K] }

K = "name"  → "name" extends "age" ? never : "name" = "name" ✅
K = "age"   → "age" extends "age" ? never : "age" = never ❌ SKIP
K = "email" → "email" extends "age" ? never : "email" = "email" ✅

Natija: { name: string; email: string } — age filtered
```

**Template literal + `Capitalize`:** String manipulation utility'lar (`Capitalize`, `Lowercase`, `Uppercase`, `Uncapitalize`) TypeScript compiler'da **intrinsic** sifatida implement qilingan — JavaScript'da emas, kompilator ichida string transformation qiladi. Bu utility'lar `lib.es5.d.ts`'da `type Uppercase<S extends string> = intrinsic;` deb declare qilingan — `intrinsic` keyword kompilatorga "bu type'ni mening ichki logikam bilan hisobla" deb buyuradi.

**Nima uchun `string & K` kerak:** `keyof T` natijasi `string | number | symbol` bo'lishi mumkin (index signature bilan, yoki symbol keyed property'lar). Template literal `` `prefix${K}` `` faqat `string | number | bigint | boolean | null | undefined`'ni qabul qiladi — `symbol` qo'shib bo'lmaydi. `string & K` intersection `K`'ni faqat string bo'lgan subset'iga cheklab, symbol va number'ni filter qiladi.

```typescript
// K: string | number | symbol
type Unsafe<T> = {
  [K in keyof T as `prefix${K}`]: T[K]; // ❌ Error agar K symbol bo'lsa
};

// K intersect string → faqat string key'lar
type Safe<T> = {
  [K in keyof T as `prefix${string & K}`]: T[K]; // ✅
};
```

**Capitalize intrinsic:** `Capitalize<S>` — birinchi harfni uppercase qiladi. Kompilator bu transform'ni to'g'ridan-to'g'ri string manipulation sifatida bajaradi, type system ichida. Runtime'da iz yo'q.

```
Capitalize<"name"> → "Name"
Capitalize<"email"> → "Email"
```

**Homomorphic xususiyat va key remapping:** `as` clause mapped type'ni homomorphic'likdan **chiqarmaydi** — key source hamon `keyof T` bo'lib qoladi, shuning uchun original modifier'lar (readonly, optional) yangi key'larga **ko'chiriladi**:

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

// Key remapping — modifier'lar saqlanadi
type Prefixed = {
  [K in keyof Config as `${string & K}Info`]: Config[K];
};
// { readonly hostInfo: string; portInfo?: number | undefined }
// ✅ readonly va ? yangi key'da ham bor
```

Kompilator har output property'ni o'zining source property'siga bog'lab, modifier'larni shu bog'lanish orqali ko'chiradi — key nomi o'zgarsa ham bog'lanish saqlanadi. Modifier yo'qolishi faqat key source `keyof T`'dan ajralganda (non-homomorphic, masalan `Record`) yuz beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Getter method type'lar yaratish
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
  email: string;
}

type UserGetters = Getters<User>;
// {
//   getName: () => string;
//   getAge: () => number;
//   getEmail: () => string;
// }

// 2. Setter method type'lar
type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type UserSetters = Setters<User>;
// {
//   setName: (value: string) => void;
//   setAge: (value: number) => void;
//   setEmail: (value: string) => void;
// }

// 3. Getter + Setter + original
type WithAccessors<T> = T & Getters<T> & Setters<T>;

// 4. Key filter — `as never` pattern
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

interface Mixed {
  name: string;
  age: number;
  email: string;
  active: boolean;
}

type StringProps = PickByValue<Mixed, string>;
// { name: string; email: string }

type NumberProps = PickByValue<Mixed, number>;
// { age: number }

// 5. Omit by value type
type OmitByValue<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};

type NonBoolProps = OmitByValue<Mixed, boolean>;
// { name: string; age: number; email: string }

// 6. Filter method property'lar
type OnlyMethods<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any ? K : never]: T[K];
};

interface Service {
  name: string;
  port: number;
  start(): void;
  stop(): void;
  getStatus(): string;
}

type ServiceMethods = OnlyMethods<Service>;
// { start: () => void; stop: () => void; getStatus: () => string }

// 7. Non-function property'lar
type OnlyProperties<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any ? never : K]: T[K];
};

type ServiceProps = OnlyProperties<Service>;
// { name: string; port: number }

// 8. CamelCase → snake_case transform
type SnakeCase<S extends string> =
  S extends `${infer C}${infer Rest}`
    ? C extends Uppercase<C>
      ? `_${Lowercase<C>}${SnakeCase<Rest>}`
      : `${C}${SnakeCase<Rest>}`
    : S;

type Snakified<T> = {
  [K in keyof T as K extends string ? SnakeCase<K> : K]: T[K];
};

interface CamelProps {
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
}

type SnakeProps = Snakified<CamelProps>;
// { first_name: string; last_name: string; date_of_birth: Date }

// 9. Change handler pattern
type WithChangeHandlers<T> = T & {
  [K in keyof T as `on${Capitalize<string & K>}Change`]: (
    newValue: T[K],
    oldValue: T[K]
  ) => void;
};

interface FormData {
  name: string;
  age: number;
}

type ReactiveForm = WithChangeHandlers<FormData>;
// {
//   name: string;
//   age: number;
//   onNameChange: (newValue: string, oldValue: string) => void;
//   onAgeChange: (newValue: number, oldValue: number) => void;
// }
```

</details>

---

## Mapped Types + Conditional Types

### Nazariya

Mapped types va conditional types **birgalikda** ishlaganda — eng qudratli type transformation'lar yaratish mumkin. Har property uchun type'i va key'iga qarab turli transform qo'llash.

**Umumiy pattern:**

```typescript
{
  [K in keyof T]: T[K] extends SomeCondition
    ? TransformA<T[K]>    // Shart rost
    : TransformB<T[K]>;   // Shart yolg'on
}
```

Bu kombinatsiya `DeepReadonly`, `DeepPartial`, object diff, type filtering kabi ko'p use case'larning asosi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator mapped type + conditional type'ni birgalikda evaluate qilganda ikki alohida mexanizm ketma-ket ishlaydi:

**1-bosqich: Mapped type iteration.** Kompilator `keyof T`'dan property key'larni oladi va har bir key uchun value type'ni hisoblashni boshlaydi.

**2-bosqich: Har property uchun conditional evaluation.** Mapped type value pozitsiyasida conditional type bo'lsa — har key uchun alohida conditional check ishlaydi.

```
DeepReadonly<Config>
  |
  Mapped type iteration(keyof Config)
  |
  K = "server" → T[K] = { host, port, ssl }
  |                extends Function? → no
  |                extends Array? → no
  |                extends object? → yes → recursive: DeepReadonly<{host,port,ssl}>
  |
  K = "features" → T[K] = string[]
                   extends Function? → no
                   extends Array<infer U>? → yes, U = string
                   → ReadonlyArray<DeepReadonly<string>>
                   → ReadonlyArray<string>
```

Har recursive call yangi mapped type instantiation yaratadi. `DeepReadonly` kabi recursive mapped + conditional type'larda kompilator instantiation depth'ni kuzatadi va juda chuqur ketganda `TS2589: Type instantiation is excessively deep and possibly infinite` xatosi bilan to'xtaydi. Nesting chuqurlashgan sari instantiation count tez o'sadi — har qatlam property soniga ko'paytiriladi.

**Caching nuance:** Kompilator instantiation natijalarini type identity bo'yicha cache qiladi — bir xil argument bilan bir xil mapped/conditional type qayta uchrasa, qayta hisoblanmaydi. Lekin generic `T` har chaqiriq'da yangi konkret type bo'lsa, har biri yangi instantiation — cache hit bo'lmaydi. Katta loyihalarda ko'p noyob instantiation performance bottleneck bo'lishi mumkin.

**`extends Function` pitfall:** Recursive mapped type + conditional'da `T[K] extends Function` check ishlatmaslik muhim — `Function` type anti-pattern. `T[K] extends (...args: any[]) => any` yoki `T[K] extends (...args: unknown[]) => unknown` ishlatish yaxshiroq:

```typescript
// ❌ Anti-pattern — Function type ko'p loose
type DeepReadonly<T> =
  T extends Function ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

// ✅ Proper callable signature
type DeepReadonlyFixed<T> =
  T extends (...args: any[]) => any ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonlyFixed<T[K]> } :
  T;
```

**Nima uchun `Function` anti-pattern:** `Function` type `lib.es5.d.ts`'da `{ apply, call, bind, name, length, ... }` shape'ida declare qilingan, lekin **inferable callable signature**'ga ega emas — ya'ni parametr va return type'lari noma'lum. Shu sababli `Function` type'idagi qiymatni chaqirsangiz, kompilator argument'larni tekshira olmaydi va return `any` deb hisoblaydi, type safety yo'qoladi. ESLint `@typescript-eslint/no-unsafe-function-type` rule'i bu pattern'ni taqiqlaydi (v8+). Tavsiya: aniq `(...args: any[]) => any` (yoki yanada aniq signature) ishlatish.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Deep Readonly — recursive mapped + conditional
type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :
  T extends Array<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

interface Config {
  server: {
    host: string;
    port: number;
    ssl: { enabled: boolean; cert: string };
  };
  features: string[];
}

type FrozenConfig = DeepReadonly<Config>;

// 2. Deep Partial — recursive
type DeepPartial<T> =
  T extends (...args: any[]) => any ? T :
  T extends Array<infer U> ? Array<DeepPartial<U>> :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;

type PartialConfig = DeepPartial<Config>;

// 3. Conditional value transformation
type Defaults<T> = {
  [K in keyof T]: T[K] extends string ? Lowercase<T[K] & string> :
                  T[K] extends number ? 0 :
                  T[K] extends boolean ? false :
                  T[K];
};

// 4. Nullable key extraction
type NullableKeys<T> = {
  [K in keyof T]: null extends T[K] ? K : never;
}[keyof T];

type NonNullableKeys<T> = {
  [K in keyof T]: null extends T[K] ? never : K;
}[keyof T];

interface Form {
  name: string;
  email: string;
  phone: string | null;
  address: string | null;
  age: number;
}

type FormNullable = NullableKeys<Form>;  // "phone" | "address"
type FormRequired = NonNullableKeys<Form>; // "name" | "email" | "age"

// 5. Object type diff
type Diff<T, U> = {
  [K in Exclude<keyof T, keyof U>]: T[K];
};

type Common<T, U> = {
  [K in Extract<keyof T, keyof U>]: T[K];
};

interface V1 {
  id: number;
  name: string;
  email: string;
}

interface V2 {
  id: number;
  name: string;
  phone: string;
}

type Added = Diff<V2, V1>;     // { phone: string }
type Removed = Diff<V1, V2>;    // { email: string }
type Shared = Common<V1, V2>;   // { id: number; name: string }

// 6. Async version of methods
type Asyncify<T> = {
  [K in keyof T]: T[K] extends (...args: infer P) => infer R
    ? (...args: P) => Promise<R>
    : T[K];
};

interface Service {
  name: string;
  start(port: number): void;
  getData(id: string): { value: number };
}

type AsyncService = Asyncify<Service>;
// {
//   name: string;
//   start: (port: number) => Promise<void>;
//   getData: (id: string) => Promise<{ value: number }>;
// }
```

</details>

---

## Custom Mapped Types — Real-World Patterns

### Nazariya

Real-world'da mapped type'lar library, framework, va application development'da keng ishlatiladi. Bu yerda eng ko'p uchraydigan pattern'larni ko'ramiz — `Getters`, `Setters`, `EventMap`, `Validators`, form state, API response.

Bu pattern'lar TypeScript type system'ning kuch'ini ko'rsatadi — boilerplate kod kam, type safety yuqori.

<details>
<summary><strong>Under the Hood</strong></summary>

Custom mapped type'lar kompilatorda oddiy mapped type sifatida qayta ishlanadi — hech qanday maxsus mexanizm yo'q. `as` clause bilan key remapping, template literal + `Capitalize` bilan key transform, conditional bilan value filter — barcha building block'lar.

**`Capitalize`, `Uppercase`, `Lowercase`, `Uncapitalize`** — TypeScript compiler'da **intrinsic** type'lar sifatida hardcode qilingan (hardcoded in the checker). `lib.es5.d.ts`'da ular `intrinsic` keyword bilan declare qilingan:

```typescript
type Uppercase<S extends string> = intrinsic;
type Lowercase<S extends string> = intrinsic;
type Capitalize<S extends string> = intrinsic;
type Uncapitalize<S extends string> = intrinsic;
```

`intrinsic` keyword kompilatorga "bu type'ni mening ichki logikam orqali hisobla" deb buyuradi. Compiler JavaScript'ning native `String.prototype.toUpperCase()` / `toLowerCase()`'ga o'xshash logika orqali string literal'larni transform qiladi. Bu type'lar runtime'da mavjud emas — faqat compile-time string manipulation uchun.

**Pattern design prinsiplari:**

1. **Consistency** — `Getters` bilan `Setters` bir xil pattern ishlatadi (Capitalize, prefix)
2. **Reusability** — `KeyOfType`, `WithChangeHandlers` — kichik building block'lar
3. **Composition** — `WithAccessors<T> = T & Getters<T> & Setters<T>`

Bu pattern'lar bir-biriga mos keladi va birga ishlatilishi mumkin. Library dizaynida bunday building block'lar muhim.

**Performance consideration:** Real-world pattern'larda key remapping + conditional + recursive kombinatsiya kompilator'ni sekinlashtirishi mumkin. `tsc --generateTrace` bilan profiling kerak bo'lishi mumkin. Intermediate type alias'lar cache'ga yordam beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Getters va Setters
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type WithAccessors<T> = T & Getters<T> & Setters<T>;

interface User {
  name: string;
  age: number;
  email: string;
}

type UserWithAccessors = WithAccessors<User>;
// User + getName/setName + getAge/setAge + getEmail/setEmail

// 2. Event Map — property → event handler
type EventMap<T> = {
  [K in keyof T as `on${Capitalize<string & K>}Change`]: (
    event: {
      property: K;
      oldValue: T[K];
      newValue: T[K];
    }
  ) => void;
};

interface UserState {
  name: string;
  age: number;
  active: boolean;
}

type UserEvents = EventMap<UserState>;
// {
//   onNameChange: (event: { property: "name"; oldValue: string; newValue: string }) => void;
//   onAgeChange: (event: { property: "age"; oldValue: number; newValue: number }) => void;
//   onActiveChange: (event: { property: "active"; oldValue: boolean; newValue: boolean }) => void;
// }

// 3. Validators — per-property validation
type Validators<T> = {
  [K in keyof T as `validate${Capitalize<string & K>}`]: (
    value: T[K]
  ) => { valid: boolean; error?: string };
};

type UserValidators = Validators<User>;
// {
//   validateName: (value: string) => { valid: boolean; error?: string };
//   validateAge: (value: number) => { valid: boolean; error?: string };
//   validateEmail: (value: string) => { valid: boolean; error?: string };
// }

// 4. Form state
type FormState<T> = {
  values: T;
  errors: { [K in keyof T]?: string };
  touched: { [K in keyof T]?: boolean };
  dirty: { [K in keyof T]?: boolean };
};

interface LoginForm {
  username: string;
  password: string;
  rememberMe: boolean;
}

type LoginFormState = FormState<LoginForm>;
// {
//   values: LoginForm;
//   errors: { username?: string; password?: string; rememberMe?: string };
//   touched: { username?: boolean; password?: boolean; rememberMe?: boolean };
//   dirty: { username?: boolean; password?: boolean; rememberMe?: boolean };
// }

// 5. API response wrapper
type ApiResponse<T> = {
  [K in keyof T]: T[K] | null;
} & {
  _meta: {
    fetched: boolean;
    loading: boolean;
    error: string | null;
  };
};

interface Product {
  id: number;
  name: string;
  price: number;
}

type ProductResponse = ApiResponse<Product>;
// {
//   id: number | null;
//   name: string | null;
//   price: number | null;
//   _meta: { fetched: boolean; loading: boolean; error: string | null };
// }

// 6. Method to promise converter
type Promisify<T> = {
  [K in keyof T]: T[K] extends (...args: infer P) => infer R
    ? (...args: P) => Promise<R>
    : T[K];
};

interface Api {
  baseUrl: string;
  get(url: string): string;
  post(url: string, body: unknown): boolean;
}

type AsyncApi = Promisify<Api>;
// {
//   baseUrl: string;
//   get: (url: string) => Promise<string>;
//   post: (url: string, body: unknown) => Promise<boolean>;
// }

// 7. Optional by condition
type OptionalByType<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]?: T[K];
} & {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};

interface Settings {
  host: string;
  port: number;
  debug: boolean;
  verbose: boolean;
}

type PartialBools = OptionalByType<Settings, boolean>;
// {
//   host: string;
//   port: number;
//   debug?: boolean;
//   verbose?: boolean;
// }
```

</details>

---

## Homomorphic va Non-Homomorphic Mapped Types

### Nazariya

Mapped type'lar ikki turga bo'linadi:

**1. Homomorphic mapped types** — original type'ning **strukturasini saqlaydi**. Original type'dagi optional (`?`), readonly modifier'lar va boshqa xususiyatlar yangi type'ga avtomatik o'tadi (agar aniq o'zgartirilmasa).

**Qoida:** Homomorphic mapped type — key source `keyof X` bo'lgan (X — type parameter), yoki key source `keyof X` bilan cheklangan type parameter bo'lgan mapped type. Birinchi shakl (`[K in keyof T]`) — modifier'larni `as` clause bilan ham, clause'siz ham saqlaydi. Ikkinchi shakl (`Pick`) — `[P in K]` bo'lib, `K extends keyof T` constraint orqali modifier source'i `T`'ga resolve bo'ladi.

```typescript
// ✅ Homomorphic — key source aynan keyof T
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };

// ✅ Homomorphic ham — K constraint'i keyof T, modifier source T
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
```

**2. Non-homomorphic mapped types** — original type strukturasini saqlamaydi. Key'lar mustaqil union type'dan olinadi, original type bilan bog'liq emas.

```typescript
// ❌ Non-homomorphic — key'lar keyof T dan emas
type MyRecord<K extends keyof any, T> = { [P in K]: T };
```

**Muhim:** `as` clause (key remapping) homomorphic xususiyatni **buzmaydi** — key source hamon `keyof T` bo'lsa, original modifier'lar yangi key'larga ko'chiriladi. Modifier yo'qolishi faqat non-homomorphic key source'da (masalan `Record`, mustaqil union) yuz beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Homomorphic detection:** Kompilator mapped type'ni qayta ishlaganda key source'ni tekshiradi:

```
Homomorphic shart (ikki shakl):
  1. { [K in keyof T]: ... }      → key source aynan keyof (type parameter)
  2. { [P in K]: ... }, K extends keyof T
                                  → K constraint'i keyof T → modifier source T

Non-homomorphic: { [K in SomeUnion]: ... }
                        ^^^^^^^^^^
                        Mustaqil union, type parameter'dan keyof emas
```

Kompilator buni `keyof X` shaklidagi constraint'ga ega type parameter mavjudligini tekshirib aniqlaydi — `X` esa modifier source bo'ladi.

Homomorphic pattern'da kompilator quyidagilarni bajaradi:

1. T'dagi har property uchun alohida declaration yaratadi
2. Original modifier'larni (`readonly`, `?`) natijaga tarqatadi
3. Mapped type'da belgilangan modifier'larni qo'shadi/olib tashlaydi

Non-homomorphic pattern'da:

1. Mustaqil union'ning har member uchun declaration yaratadi
2. Hech qanday modifier propagation yo'q
3. Faqat mapped type'da belgilangan modifier'lar qo'llaniladi

**Key remapping va modifier propagation:**

```
{ [K in keyof T as NewKey]: ... }
```

Key remapping bo'lganda key source hamon `keyof T` — kompilator har output property'ni o'zining source property'siga bog'lab qoldiradi. Shuning uchun modifier propagation `as` clause bilan ham ishlaydi:

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

// Homomorphic — modifier saqlanadi
type A = { [K in keyof Config]: Config[K] };
// { readonly host: string; port?: number }

// Key remapping — modifier hamon saqlanadi
type B = { [K in keyof Config as `${K & string}Info`]: Config[K] };
// { readonly hostInfo: string; portInfo?: number | undefined }
// readonly va ? yangi key'da ham bor
```

**Real-world implication:** `Record<keyof T, V>` ishlatish — juda keng tarqalgan xato. Developer'lar `Record`'ni shortcut sifatida ishlatadilar, lekin u non-homomorphic (key source mustaqil union) — modifier'lar yo'qoladi:

```typescript
// ❌ Modifier yo'qoladi
type AllStrings = Record<keyof Config, string>;
// { host: string; port: string } — readonly va ? yo'q

// ✅ Homomorphic alternative
type AllStringsHomo = { [K in keyof Config]: string };
// { readonly host: string; port?: string } — saqlanadi
```

Bu farq kam hollarda muhim, lekin immutability yoki optional tracking muhim bo'lgan kontekstda kritik.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface Config {
  readonly host: string;
  readonly port: number;
  timeout?: number;
}

// 1. Homomorphic — modifier saqlanadi
type PartialConfig = Partial<Config>;
// {
//   readonly host?: string;  ✅ readonly
//   readonly port?: number;  ✅ readonly
//   timeout?: number;
// }

type ReadonlyConfig = Readonly<Config>;
// Hammasi readonly (bor edi, qo'shildi)

// 2. Non-homomorphic — Record
type RecordConfig = Record<keyof Config, string>;
// {
//   host: string;    ❌ readonly yo'q
//   port: string;    ❌ readonly yo'q
//   timeout: string; ❌ ? yo'q
// }

// 3. Pick — homomorphic
type HostOnly = Pick<Config, "host">;
// { readonly host: string } — ✅ readonly

// 4. Identity — homomorphic
type Identity<T> = { [K in keyof T]: T[K] };
type IdConfig = Identity<Config>;
// { readonly host: string; readonly port: number; timeout?: number }

// 5. Key remapping — modifier hamon saqlanadi
type Prefixed<T> = {
  [K in keyof T as `${string & K}Prop`]: T[K];
};

type PrefixedConfig = Prefixed<Config>;
// {
//   readonly hostProp: string;       ✅ readonly ko'chdi
//   readonly portProp: number;       ✅ readonly ko'chdi
//   timeoutProp?: number;            ✅ ? ko'chdi
// }

// 6. Mapped + conditional — homomorphic xususiyat saqlanadi (keyof T ishlatilsa)
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type NullableConfig = Nullable<Config>;
// {
//   readonly host: string | null;  ✅ readonly
//   readonly port: number | null;  ✅ readonly
//   timeout?: number | null;        ✅ ?
// }
```

</details>

---

## Mapped Types va Union Types

### Nazariya

Mapped type'ga union type berilganda qanday ishlaydi? Bu conditional type'lardagi distributive behavior'dan **farq qiladi**.

**Mapped type va union — distribute qilmaydi:**

```typescript
type Boxed<T> = { [K in keyof T]: { value: T[K] } };

// Union berilganda:
type Result = Boxed<string | number>;
// keyof (string | number) = shared keys only
// Faqat string VA number da umumiy bo'lgan key'lar
```

**Mapped type ichida conditional distribution:**

```typescript
// Har property'ni conditional bilan transform
type TransformValues<T> = {
  [K in keyof T]: T[K] extends string
    ? T[K]
    : T[K] extends number
      ? `${T[K]}`
      : T[K];
};

interface Data {
  name: string;
  count: number;
  active: boolean;
}

type Transformed = TransformValues<Data>;
// { name: string; count: `${number}`; active: boolean }
```

<details>
<summary><strong>Under the Hood</strong></summary>

Mapped type va conditional type'ning distributive behavior'i **fundamental farq** qiladi.

**Conditional type distribution:** `T extends U ? X : Y`'da agar `T` naked type parameter bo'lsa va union berilsa — kompilator union'ni member'larga ajratadi va har birini alohida evaluate qiladi. Natijalar union'ga qayta birlashtiriladi.

**Mapped type:** `{ [K in keyof T]: ... }`'da kompilator `keyof T`'ni hisoblaydi. Union type'ga `keyof` qo'llansa — faqat **shared (umumiy) key'lar** qaytadi:

```
keyof (string | number)
  = (keyof string) & (keyof number)    ← intersection!
  = shared methods (valueOf, toString, ...)

Conditional type distribution (farqli):
  T extends X ? A : B where T = string | number
  = (string extends X ? A : B) | (number extends X ? A : B)
                                       ← union members alohida!
```

Shuning uchun `{ [K in T]: ... }` pattern'da `T extends string` constraint qo'yilsa (T — string literal union) — `K` mapped type'ning iteration variable bo'lib, union'ning har member'i bo'yicha aylanadi. Bu conditional distribution emas — bu mapped type'ning o'z iteration mexanizmi: `K` har iteration'da bitta literal'ga bog'lanadi.

**Union iteration pattern:** `{ [K in UnionType]: ... }` — union'ning har member'i K bo'lib turadi. Bu literal union'dan object yaratish uchun ishlatiladi:

```typescript
type Colors = "red" | "green" | "blue";
type ColorMap = { [K in Colors]: string };
// { red: string; green: string; blue: string }
```

Bu non-homomorphic mapped type — key source `keyof T`'dan emas, mustaqil union.

**Discriminated union flatten pattern:**

```typescript
type EventUnion =
  | { type: "click"; x: number; y: number }
  | { type: "keydown"; key: string };

// Event → lookup table
type EventLookup = {
  [E in EventUnion as E["type"]]: Omit<E, "type">;
};
// {
//   click: { x: number; y: number };
//   keydown: { key: string };
// }
```

Bu pattern'da `[E in EventUnion as E["type"]]` — union'ning har member'ini iterate qiladi va `as E["type"]` bilan key'ni type field'dan oladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Union → object map
type UnionToObject<T extends string> = {
  [K in T]: K;
};

type Colors = "red" | "green" | "blue";
type ColorMap = UnionToObject<Colors>;
// { red: "red"; green: "green"; blue: "blue" }

// 2. Union → flag object
type Flags<T extends string> = {
  [K in T]: boolean;
};

type ColorFlags = Flags<Colors>;
// { red: boolean; green: boolean; blue: boolean }

// 3. Discriminated union → lookup map
type EventUnion =
  | { type: "click"; x: number; y: number }
  | { type: "keydown"; key: string }
  | { type: "scroll"; offset: number };

type EventLookup = {
  [E in EventUnion as E["type"]]: Omit<E, "type">;
};
// {
//   click: { x: number; y: number };
//   keydown: { key: string };
//   scroll: { offset: number };
// }

// 4. Enum-like — literal union with defaults
type Status = "pending" | "active" | "inactive";

type StatusDefaults = {
  [S in Status]: { count: number; label: string };
};

// 5. Shared keys of union (nozik holat)
type SharedKeys<T, U> = keyof (T | U);
// = (keyof T) & (keyof U)

type Common = SharedKeys<
  { a: 1; b: 2; c: 3 },
  { b: 2; c: 4; d: 5 }
>;
// "b" | "c" — shared keys

// 6. Transform each union member
type EventHandlers = {
  [E in EventUnion as `handle${Capitalize<E["type"]>}`]: (
    payload: Omit<E, "type">
  ) => void;
};
// {
//   handleClick: (payload: { x: number; y: number }) => void;
//   handleKeydown: (payload: { key: string }) => void;
//   handleScroll: (payload: { offset: number }) => void;
// }
```

</details>

---

## Performance va Best Practices

### Nazariya

Murakkab mapped type'lar kompilator performance'iga ta'sir qilishi mumkin. Quyidagi best practice'lar bu ta'sirni kamaytirishga yordam beradi.

**Asosiy printsiplar:**

1. **Recursion depth cheklash** — counter tuple bilan recursion'ni belgilangan darajada to'xtatish (kompilator `TS2589: Type instantiation is excessively deep` xatosiga yetmasdan)
2. **Intermediate type alias'lar** — katta mapped type'larni kichik nomlangan bo'laklarga ajratish
3. **Pick bilan pre-filter** — katta type'dan faqat kerakli property'larni oldindan tanlash
4. **Per-property conditional'ni yengillashtirish** — har property uchun bajariladigan conditional zanjiri (`extends A ? ... : extends B ? ...`) property soniga ko'paytiriladi; chuqur/keng conditional'ni soddalashtirish evaluation cost'ini kamaytiradi

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator mapped type'ni evaluate qilganda har property uchun yangi type node yaratadi. Performance muammosi — property soni va har property uchun bajariladigan operatsiyalar sonining ko'paytirilishi.

```
Mapped type evaluation cost:

{ [K in keyof T]: Transform<T[K]> }

Cost = |keyof T| * cost(Transform)

Taxminiy hisob (DeepReadonly, 3 level deep):
  Level 0: N property  → N type nodes
  Level 1: N * M nested → N*M nodes
  Level 2: N*M * P nested → N*M*P nodes

Katta type'da tez o'sadi.
```

**Profiling:** `tsc --generateTrace <dir>` bilan compilation trace yoziladi — Chrome DevTools'ning Performance tab'ida ochish mumkin. Trace'da har type'ni tekshirish (type checking) qancha vaqt olganini ko'rib, eng qimmat type evaluation'larni topish mumkin. Agar bitta mapped type ko'p vaqt olsa — refactor kerak.

**Type caching:** Kompilator mapped type natijalarini cache qiladi. Bir xil `T` argument bilan bir xil mapped type qayta chaqirilsa — cache'dan olinadi. Lekin generic `T` har safar yangi type bo'lsa — cache ishlamaydi. Shuning uchun intermediate type alias'lar (`type A = X; type B = Y`) har alohida cache'ga tushadi — bu faydali optimization.

**`Pick` bilan pre-filter:** Katta type'da `DeepReadonly<HugeType>` — barcha property'lar uchun recursive computation. Lekin `DeepReadonly<Pick<HugeType, "a" | "b">>` — faqat ikki property — ancha tez.

```typescript
// ❌ Sekin — barcha property
type Slow = DeepReadonly<HugeType>;

// ✅ Tez — faqat kerakli
type Fast = DeepReadonly<Pick<HugeType, "needed1" | "needed2">>;
```

**Memory ta'siri:** Har noyob instantiation type system ichida saqlanadi — ko'p turli xil argument bilan ishlaydigan murakkab generic mapped type'lar compilation memory'sini oshiradi. Bu `tsc` jarayonining xotira sarfida ko'rinadi.

**Intermediate type aliases — pedagogical benefit:** Murakkab mapped type'ni kichik bo'laklarga ajratish nafaqat performance, balki kod o'qilishini ham yaxshilaydi. Hard-to-read nested mapped type o'rniga nomlangan intermediate type'lar — har birining maqsadi aniq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Recursion depth limit
type DeepReadonlyLimited<T, Depth extends number[] = []> =
  Depth["length"] extends 5 ? T :
  T extends (...args: any[]) => any ? T :
  T extends object ? {
    readonly [K in keyof T]: DeepReadonlyLimited<T[K], [...Depth, 0]>;
  } : T;

// 5 darajadan chuqur bo'lsa — to'xtaydi

// 2. Intermediate type alias'lar
type MethodKeys<T> = {
  [K in keyof T]: T[K] extends (...args: any[]) => any ? K : never;
}[keyof T];

type AsyncMethod<F> = F extends (...args: infer P) => infer R
  ? (...args: P) => Promise<R>
  : never;

type AsyncMethods<T> = {
  [K in MethodKeys<T> as `async${Capitalize<string & K>}`]: AsyncMethod<T[K]>;
};

// Kichik, readable, cacheable bo'laklar

// 3. Pick bilan pre-filter
interface HugeType {
  a: number;
  b: string;
  c: boolean;
  // ... 100+ property
}

// Katta type'da murakkab transformation — sekin
// type Slow = DeepReadonly<HugeType>;

// Kerakli qismni oldindan Pick
type Needed = Pick<HugeType, "a" | "b" | "c">;
type Fast = DeepReadonly<Needed>;

// 4. Cache bilan optimization
type StringKeys<T> = {
  [K in keyof T]: T[K] extends string ? K : never;
}[keyof T];

type NumberKeys<T> = {
  [K in keyof T]: T[K] extends number ? K : never;
}[keyof T];

// Alohida cache'lar — tezroq
type StringProps<T> = Pick<T, StringKeys<T>>;
type NumberProps<T> = Pick<T, NumberKeys<T>>;

// 5. Profiling workflow
// 1. tsc --generateTrace ./trace
// 2. Chrome DevTools → Performance tab → Load trace
// 3. Mapped type event'larini kuzatish
// 4. Bottleneck'ni topish → refactor
```

</details>

---

## Edge Cases va Gotchas

### 1. `keyof` Index Signature `string | number`

Index signature bilan object'da `keyof` `string | number` qaytaradi (symbol emas, lekin number ham).

```typescript
type IndexedObj = { [key: string]: unknown };
type Keys = keyof IndexedObj; // string | number
// Diqqat: number ham keladi — nima uchun?
```

**Sabab:** JavaScript'da object key'lar internal'da string'ga aylantiriladi — `obj[0]` va `obj["0"]` teng. TypeScript bu behavior'ni aks ettirish uchun `string | number` qaytaradi, developer number index ham ishlata olishi uchun.

```typescript
const obj: IndexedObj = {};
obj[0] = "a";    // ✅ number index
obj["0"] = "b";  // ✅ string index (aynan bir xil property)
console.log(obj[0]); // "b" — bir xil
```

**`{ [key: number]: unknown }`** — faqat `number` qaytaradi. `{ [key: symbol]: unknown }` — faqat `symbol`.

### 2. `-?` `undefined` Olib Tashlaydi (TS 2.8+), Lekin Faqat Optional'da

`Required<T>` `-?` bilan implement qilingan. TS 2.8+'da `-?` optional marker'ni va `| undefined`'ni ikkalasini ham olib tashlaydi. Lekin property optional emas bo'lsa — hech narsa qilmaydi:

```typescript
interface Example {
  a: string;
  b?: string;            // optional (= string | undefined)
  c: string | undefined; // required, lekin undefined bo'lishi mumkin
}

type Req = Required<Example>;
// {
//   a: string;
//   b: string;             // ✅ -? undefined ham olib tashladi
//   c: string | undefined; // Diqqat: -? hech narsa qilmadi, | undefined qoldi
// }
```

**Nima uchun:** `-?` faqat optional marker'ni tahlil qiladi. `c`'da optional marker yo'q — `-?` uchun hech narsa yo'q. `| undefined` union member — bu boshqa narsa, `-?` olib tashlamaydi.

**Yechim:** `NonNullable<T[K]>` bilan kombinatsiya:

```typescript
type FullyRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type Fixed = FullyRequired<Example>;
// { a: string; b: string; c: string }
```

### 3. `Record<keyof T, V>` Modifier'larni Yo'qotadi

`Record<K, V>` — non-homomorphic mapped type. `keyof T` ishlatilsa ham, kompilator original `T`'ni unutadi — faqat key'lar qoladi:

```typescript
interface Config {
  readonly host: string;
  port?: number;
  timeout?: number;
}

type R = Record<keyof Config, string>;
// {
//   host: string;    // ❌ readonly yo'q
//   port: string;    // ❌ ? yo'q
//   timeout: string; // ❌ ? yo'q
// }
```

**Sabab:** `Record`'ning implementation'i `{ [P in K]: T }` — `K` va `T` ikki mustaqil parameter. Kompilator `K`'ning qaerdan kelganini (`keyof Config`) hisobga olmaydi — u shunchaki union sifatida ishlatiladi. Original type'ning structure'i yo'qoladi.

**Homomorphic alternative:**

```typescript
type RHomo = {
  [K in keyof Config]: string;
};
// { readonly host: string; port?: string; timeout?: string }
// ✅ readonly va ? saqlanadi
```

Inline mapped type `keyof T` bilan — homomorphic. `Record` — non-homomorphic. Bu farq nozik, lekin immutability yoki optional tracking'da kritik.

### 4. `as never` — Property Removal Mexanizmi

Key remapping'da `as` clause'da `never` qaytarilsa, property butunlay o'chiriladi. Bu TypeScript'ning eng kuchli filter pattern'i:

```typescript
type OmitByType<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};

interface Mixed {
  name: string;
  age: number;
  active: boolean;
}

type NoStrings = OmitByType<Mixed, string>;
// { age: number; active: boolean }
```

**Nima uchun `never` property'ni o'chiradi:** Key remapping'da `as` clause natijasi property'ning yangi key type'ini belgilaydi. Agar bu type `never` bo'lsa, kompilator natijaviy object type'ni qurishda shu key'ni tashlab yuboradi — `never` hech qanday string/number/symbol literal'ni o'z ichiga olmaydi, shuning uchun unga mos property nomi yo'q. Bu type system mexanizmi (`never` — empty type), ECMAScript runtime cheklovi emas.

**Power pattern:** `PickByType`, `OmitByType`, `PickByValue`, `OmitByValue` — barchasi `as never` pattern'iga asoslangan. `Omit` utility type ham ichki `as never` bilan implement qilingan (TS 4.1+):

```typescript
type MyOmit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
```

### 5. Recursive Mapped + Function Check Shart

Recursive mapped type (`DeepReadonly`, `DeepPartial`) yozilayotganda Function check **majburiy** — aks holda function-valued property `object` deb topilib recursive mapped type'ga aylanadi, va call signature mapped type ichida iterate qilinmagani uchun yo'qoladi (qaytgan type endi callable emas).

```typescript
// ❌ Function check yo'q — call signature yo'qoladi
type BrokenDeep<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? BrokenDeep<T[K]>
    : T[K];
};

interface HasMethod {
  name: string;
  greet: () => string; // function type ham object
}

type Broken = BrokenDeep<HasMethod>;
// greet → object deb topiladi → BrokenDeep<() => string> ga recurse bo'ladi.
// `keyof (() => string)` = never (call signature mapped type'da iterate qilinmaydi),
// natijada greet `{}` ga aylanadi — call signature yo'qoladi.

declare const broken: Broken;
broken.greet(); // ❌ TS2349: This expression is not callable
```

**Yechim — Function check oldinda:**

```typescript
type SafeDeep<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K] // Function — o'zgartirma
    : T[K] extends object
      ? SafeDeep<T[K]> // Object — recursive
      : T[K]; // Primitive — o'zgartirma
};
```

**Nima uchun `Function` type ishlatmaslik:** `T[K] extends Function` TypeScript'da anti-pattern — `Function` type `{ apply, call, bind, ... }` ga ekvivalent, lekin callable signature yo'q. ESLint `@typescript-eslint/no-unsafe-function-type` rule'i buni taqiqlaydi (v8+; eski v7 da `ban-types` ichida edi). `(...args: any[]) => any` proper callable signature — aniqroq va tip-safe.

---

## Common Mistakes

### ❌ Xato 1: Homomorphic vs Record modifier loss

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

// ❌ Record — modifier yo'qoladi
type Bad = Record<keyof Config, boolean>;
// { host: boolean; port: boolean } — readonly va ? yo'q
```

**✅ To'g'ri usul:**

```typescript
type Good = { [K in keyof Config]: boolean };
// { readonly host: boolean; port?: boolean } — saqlanadi
```

**Nima uchun:** `Record<K, V>` — non-homomorphic, modifier'larni yo'qotadi. Inline `keyof T` mapped type — homomorphic, modifier'larni saqlaydi.

---

### ❌ Xato 2: Key remapping'da `string & K` unutish

```typescript
// ❌ Error: Type 'K' does not satisfy the constraint 'string'
type Broken<T> = {
  [K in keyof T as `get${Capitalize<K>}`]: T[K]; // ❌
};
// keyof T = string | number | symbol bo'lishi mumkin
// Capitalize faqat string oladi
```

**✅ To'g'ri usul:**

```typescript
type Fixed<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: T[K]; // ✅
};
// string & K — K'ni string subset'iga cheklaydi
```

**Nima uchun:** `keyof T` number va symbol ham bo'lishi mumkin. `Capitalize`, template literal faqat `string | number | bigint | boolean | null | undefined` qabul qiladi. `string & K` intersection symbol'larni filter qiladi.

---

### ❌ Xato 3: `-?` va `| undefined` farqini tushunmaslik

```typescript
interface Example {
  a: string;
  b?: string;
  c: string | undefined;
}

type Req = Required<Example>;
// {
//   a: string;
//   b: string;             // ✅ -? undefined ham olib tashladi
//   c: string | undefined; // ❌ -? faqat ? olib tashlaydi
// }
```

**✅ Hammasini string qilish:**

```typescript
type FullyRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type Fixed = FullyRequired<Example>;
// { a: string; b: string; c: string }
```

**Nima uchun:** `-?` faqat optional marker'ni tahlil qiladi. Explicit `| undefined` boshqa — union member. `NonNullable<T>` kombinatsiyasi ikkalasini ham olib tashlaydi.

---

### ❌ Xato 4: Recursive mapped type'da Function check yo'q

```typescript
// ❌ Function ham object — recursive bo'lib ketadi
type BrokenDeep<T> = {
  readonly [K in keyof T]: T[K] extends object ? BrokenDeep<T[K]> : T[K];
};

interface HasMethod {
  name: string;
  greet: () => string;
}

type Broken = BrokenDeep<HasMethod>;
// greet → BrokenDeep<() => string> → call signature yo'qoladi → endi callable emas
```

**✅ To'g'ri usul:**

```typescript
type FixedDeep<T> = {
  readonly [K in keyof T]: T[K] extends (...args: any[]) => any
    ? T[K]                      // Function — o'zgartirma
    : T[K] extends object
      ? FixedDeep<T[K]>          // Object — recursive
      : T[K];                    // Primitive — o'zgartirma
};
```

**Nima uchun:** function type `object`'ga extends qiladi, shuning uchun Function check'siz recursive mapped type function-valued property'ni `object` deb topib unga ham qo'llanadi — call signature mapped type ichida iterate qilinmaydi, natijada qaytgan type callable bo'lmay qoladi. `(...args: any[]) => any` proper callable signature check.

---

### ❌ Xato 5: Katta type'da performance

```typescript
// ❌ 100+ property li type'ga complex mapped type
interface HugeType { /* 100+ property */ }

type Slow = DeepReadonly<WithAccessors<Promisify<HugeType>>>;
// Har chaqiriq'da 100+ property uchun recursive transformation
// Instantiation soni property soniga ko'paytiriladi — compilation sekinlashadi
```

**✅ To'g'ri usullar:**

```typescript
// 1. Pick bilan pre-filter
type Needed = Pick<HugeType, "id" | "name" | "status">;
type Fast = DeepReadonly<Needed>;

// 2. Intermediate type alias'lar
type WithMethods = WithAccessors<HugeType>;
type WithAsync = Promisify<WithMethods>;
type FinalReadonly = DeepReadonly<WithAsync>;

// 3. Depth limit
type SafeDeep<T, D extends number[] = []> =
  D["length"] extends 3 ? T :
  T extends object ? { readonly [K in keyof T]: SafeDeep<T[K], [...D, 0]> } :
  T;
```

**Nima uchun:** Kompilator har mapped type'ni yangi instantiation'ga aylantiradi. Katta type'da bu eksponensial o'sishi mumkin. Pick, alias, depth limit — performance best practice'lar.

---

## Amaliy Mashqlar

### Mashq 1: `Mutable<T>` va `DeepMutable<T>` (O'rta)

`Readonly<T>` teskarisini yarating — barcha `readonly` modifier'larni olib tashlash. Recursive versiyasi ham.

<details>
<summary>Javob</summary>

```typescript
// Shallow Mutable
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Recursive DeepMutable
type DeepMutable<T> =
  T extends (...args: any[]) => any ? T :
  T extends Array<infer U> ? Array<DeepMutable<U>> :
  T extends object ? {
    -readonly [K in keyof T]: DeepMutable<T[K]>;
  } :
  T;

interface FrozenConfig {
  readonly host: string;
  readonly port: number;
  readonly ssl: {
    readonly enabled: boolean;
    readonly cert: string;
  };
}

type A = Mutable<FrozenConfig>;
// Shallow — ssl ichidagi readonly saqlanadi

type B = DeepMutable<FrozenConfig>;
// Barcha darajada readonly olib tashlandi
```

**Tushuntirish:**

- `-readonly` — readonly modifier'ni olib tashlaydi
- `DeepMutable` — Function va Array alohida handle
- `(...args: any[]) => any` — proper callable signature

</details>

---

### Mashq 2: `PickByType<T, V>` va `OmitByType<T, V>` (O'rta)

Value type'ga qarab property'larni filter qilish.

<details>
<summary>Javob</summary>

```typescript
type PickByType<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

type OmitByType<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};

interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
  createdAt: Date;
}

type StringProps = PickByType<User, string>;
// { name: string; email: string }

type NumberProps = PickByType<User, number>;
// { id: number; age: number }

type NonStringProps = OmitByType<User, string>;
// { id: number; age: number; active: boolean; createdAt: Date }

type NonPrimitive = OmitByType<User, string | number | boolean>;
// { createdAt: Date }
```

**Tushuntirish:**

- `as T[K] extends V ? K : never` — key remapping bilan filter
- `never` — property butunlay o'chiriladi
- `OmitByType` — `PickByType`'ning teskari

</details>

---

### Mashq 3: Result Prediction (O'rta)

```typescript
interface Data {
  readonly id: number;
  name: string;
  tags?: string[];
}

type A = { [K in keyof Data]: Data[K] };
type B = { [K in keyof Data]?: Data[K] };
type C = { -readonly [K in keyof Data]: Data[K] };
type D = { [K in keyof Data]-?: Data[K] };
type E = { [K in keyof Data as `${K & string}Info`]: Data[K] };
type F = { [K in keyof Data as Data[K] extends string ? K : never]: Data[K] };
type G = Record<keyof Data, boolean>;
```

<details>
<summary>Javob</summary>

```typescript
type A = { [K in keyof Data]: Data[K] };
// { readonly id: number; name: string; tags?: string[] }
// Homomorphic — modifier saqlanadi

type B = { [K in keyof Data]?: Data[K] };
// { readonly id?: number; name?: string; tags?: string[] }
// +? qo'shildi, readonly saqlandi

type C = { -readonly [K in keyof Data]: Data[K] };
// { id: number; name: string; tags?: string[] }
// -readonly olib tashladi, ? saqlandi

type D = { [K in keyof Data]-?: Data[K] };
// { readonly id: number; name: string; tags: string[] }
// -? olib tashladi, readonly saqlandi
// tags endi string[] (undefined ham olib tashlandi)

type E = { [K in keyof Data as `${K & string}Info`]: Data[K] };
// { readonly idInfo: number; nameInfo: string; tagsInfo?: string[] }
// Key remapping — modifier'lar yangi key'larga ko'chadi (readonly, ?)

type F = { [K in keyof Data as Data[K] extends string ? K : never]: Data[K] };
// { name: string }
// Filter — faqat string value

type G = Record<keyof Data, boolean>;
// { id: boolean; name: boolean; tags: boolean }
// Non-homomorphic — readonly va ? yo'qoldi
```

</details>

---

### Mashq 4: `DeepRequired<T>` (Qiyin)

Recursive required — barcha nested level'da optional olib tashlash.

<details>
<summary>Javob</summary>

```typescript
type DeepRequired<T> =
  T extends (...args: any[]) => any ? T :
  T extends Array<infer U> ? Array<DeepRequired<U>> :
  T extends object ? {
    [K in keyof T]-?: DeepRequired<T[K]>;
  } :
  T;

interface Config {
  host?: string;
  port?: number;
  ssl?: {
    enabled?: boolean;
    cert?: string;
    options?: {
      ttl?: number;
      retries?: number;
    };
  };
}

type RequiredConfig = DeepRequired<Config>;
// Barcha darajada optional olib tashlandi
// {
//   host: string;
//   port: number;
//   ssl: {
//     enabled: boolean;
//     cert: string;
//     options: { ttl: number; retries: number };
//   };
// }

// Nullable olib tashlash uchun:
type DeepRequiredNonNull<T> =
  T extends (...args: any[]) => any ? T :
  T extends Array<infer U> ? Array<DeepRequiredNonNull<U>> :
  T extends object ? {
    [K in keyof T]-?: DeepRequiredNonNull<NonNullable<T[K]>>;
  } :
  T;
```

**Tushuntirish:**

- `-?` har darajada optional olib tashlaydi
- `DeepRequired<T[K]>` recursive
- Function va Array alohida handle (infinite recursion'dan qochish)
- `NonNullable<T[K]>` variant — null/undefined ham olib tashlash

</details>

---

### Mashq 5: Type-Safe Mapped Event Bus (Qiyin)

Event bus uchun type-safe mapping yarating — event nomi + payload type dan handler tiplar.

<details>
<summary>Javob</summary>

```typescript
// type alias (interface emas) — index signature'siz interface
// `Record<string, object>` constraint'iga assign bo'lmaydi (TS2344)
type AppEvents = {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "item:add": { itemId: string; quantity: number };
  "error": { message: string; code: number };
};

// Event handler map
type EventHandlers<T> = {
  [K in keyof T]: (payload: T[K]) => void;
};

// Unsubscribe function
type Unsubscribe = () => void;

// Event bus class
class TypedEventBus<T extends Record<string, object>> {
  private handlers = new Map<keyof T, Array<(...args: any[]) => void>>();

  on<K extends keyof T>(event: K, handler: (payload: T[K]) => void): Unsubscribe {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as (...args: any[]) => void);
    this.handlers.set(event, list);

    return () => {
      const current = this.handlers.get(event) ?? [];
      const filtered = current.filter(h => h !== handler);
      this.handlers.set(event, filtered);
    };
  }

  emit<K extends keyof T>(event: K, payload: T[K]): void {
    const list = this.handlers.get(event) ?? [];
    list.forEach(h => h(payload));
  }

  // All registered events as union
  getEventNames(): (keyof T)[] {
    return Array.from(this.handlers.keys());
  }
}

// Usage
const bus = new TypedEventBus<AppEvents>();

const unsubscribe = bus.on("user:login", (payload) => {
  // payload: { userId: string; timestamp: number } — auto typed
  console.log(`User ${payload.userId} logged in at ${payload.timestamp}`);
});

bus.emit("user:login", { userId: "123", timestamp: Date.now() });
bus.emit("error", { message: "Failed", code: 500 });

// ❌ Type errors:
// bus.emit("user:login", { userId: 123 }); // userId must be string
// bus.on("unknown", ...); // "unknown" not in AppEvents

unsubscribe(); // Stop listening
```

**Tushuntirish:**

- `T extends Record<string, object>` — event map constraint
- `K extends keyof T` — generic event name
- `T[K]` — event payload type avto-extract
- `Unsubscribe` — cleanup callback
- Mapped type'siz qilolmas edi — har method o'z generic'iga ega

</details>

---

## Xulosa

Bu bo'limda mapped types'ning chuqur mexanizmlari o'rganildi:

**Asosiy tushunchalar:**

- **Syntax va semantics** — `{ [K in keyof T]: T[K] }`, kompilator ichida iteration jarayoni
- **`keyof` va index access** — mapped type building block'lari, `keyof` turli type'lar bilan (index signature, primitive, array)
- **Property modifiers** — `+?`, `-?`, `+readonly`, `-readonly` — kombinatsiyalari, `-?` va `| undefined` farqi
- **Built-in utility types** — `Partial`, `Required`, `Readonly`, `Pick`, `Record`, `Omit` — mapped type implementation'lari
- **Key remapping (`as`)** — key rename, filter (`as never`), template literal + `Capitalize`, `string & K` intersection
- **Mapped + conditional** — `DeepReadonly`, `DeepPartial`, object diff, nullable keys extraction
- **Custom patterns** — `Getters`, `Setters`, `EventMap`, `Validators`, form state, API response
- **Homomorphic vs non-homomorphic** — `keyof T` → homomorphic (modifier saqlanadi), `Record` → non-homomorphic (yo'qoladi)
- **Mapped + union** — shared keys intersection, union iteration, discriminated union flatten
- **Performance** — recursion depth limit, intermediate alias'lar, Pick pre-filter

**Umumiy takeaway'lar:**

1. **`keyof T` homomorphic.** Modifier'lar (readonly, optional) avtomatik saqlanadi. `Record<keyof T, V>` non-homomorphic — modifier yo'qoladi.
2. **`as never` property removal.** Filter pattern'ning asosi — `PickByType`, `OmitByType`, `Omit` barchasi shu pattern bilan.
3. **`-?` TS 2.8+ undefined ham olib tashlaydi.** Lekin faqat optional property'larda — `| undefined` standalone qoladi.
4. **`string & K` template literal'da.** `keyof T`'da symbol va number bo'lishi mumkin — template literal'da filter kerak.
5. **Recursive mapped + Function check.** `T[K] extends (...args: any[]) => any` shart, `Function` type anti-pattern.
6. **Performance — intermediate alias, Pick pre-filter.** Katta type'larda murakkab mapped type'lar sekin.
7. **Key remapping modifier saqlaydi.** `as` clause key source `keyof T` bo'lsa homomorphic'likni buzmaydi — readonly/optional yangi key'larga ko'chadi. Modifier yo'qolishi faqat non-homomorphic key source'da (`Record`, mustaqil union).
8. **Index signature `keyof` `string | number` beradi.** JavaScript'ning number/string key equivalence'i.

**Cross-references:**

- **[07-functions.md](07-functions.md)** — Function type, callable signature
- **[08-generics.md](08-generics.md)** — `keyof` asos, index access
- **[09-advanced-generics.md](09-advanced-generics.md)** — Mapped types intro
- **[12-conditional-types.md](12-conditional-types.md)** — Conditional chuqur, `infer`, distributive
- **[14-template-literal-types.md](14-template-literal-types.md)** — Template literal chuqur, string manipulation
- **[15-utility-types.md](15-utility-types.md)** — Built-in utility'lar to'liq
- **[16-custom-utility-types.md](16-custom-utility-types.md)** — Custom pattern'lar
- **[25-type-compatibility.md](25-type-compatibility.md)** — Variance, assignability

---

**Keyingi bo'lim:** [14-template-literal-types.md](14-template-literal-types.md) — Template Literal Types chuqur: string pattern matching, union distribution, `Uppercase`/`Lowercase`/`Capitalize`/`Uncapitalize` intrinsic types, type-safe routing, event naming patterns.
