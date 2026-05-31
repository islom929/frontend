# Interview: Mapped Types

> Mapped type syntax, property modifiers, key remapping, homomorphic vs non-homomorphic, `Record`, recursive mapped types va custom utility patterns bo'yicha interview savollari. Har javob mustaqil — kontekst javob ichida.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar) — 8 ta
- [Output savollari](#output-savollari) — 5 ta
- [Coding challenges](#coding-challenges) — 4 ta
- [Bug fix](#bug-fix) — 2 ta
- [Xulosa](#xulosa)

---

## Nazariy savollar

### Savol 1: Mapped type nima va qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Mapped type — mavjud object type'ning har property'sini iterate qilib, yangi object type yaratish. Syntax: `{ [K in keyof T]: NewType }`.

### To'liq tushuntirish

Mapped type compile-time'da object type'ni transform qiladi. `keyof T` operator T'ning key'larini union sifatida beradi (`"name" | "age" | "email"`), `in` keyword shu union ustida iterate qiladi, har key uchun yangi property type generate qiladi.

Compiler bosqichlari:
1. `keyof T` — T'ning property name'larini union'ga aylantirish
2. Har `K` uchun yangi property descriptor yaratish
3. Natija type'larni single object type'da birlashtirish

Mapped type Array.prototype.map() ning type-level ekvivalenti — element o'rniga property, callback o'rniga type expression.

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

// Partial implementation — har property optional
type MyPartial<T> = { [K in keyof T]?: T[K] };
type PartialUser = MyPartial<User>;
// { name?: string; age?: number; email?: string }

// Readonly implementation — har property readonly
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };

// Value transformation — har value'ni Promise'ga wrap
type Promisified<T> = { [K in keyof T]: Promise<T[K]> };
type PromisifiedUser = Promisified<User>;
// { name: Promise<string>; age: Promise<number>; email: Promise<string> }

// Nullable — har property | null qabul qiladi
type Nullable<T> = { [K in keyof T]: T[K] | null };
```

### Edge Cases

- `keyof T` da `never` bo'lsa (`T = {}`) — natija `{}` (bo'sh object type)
- `T` primitive bo'lsa (`string`, `number`) — `keyof T` o'zining method'larini beradi (`"toString" | "valueOf" | ...`)
- Index signature mavjud bo'lsa — mapped type signature'ni ham iterate qiladi
- Symbol key'lar — `keyof T` symbol'larni ham qaytaradi, lekin template literal ichida ishlatish uchun `string & K` filter kerak

### Follow-up savollar

1. **"`{ [K in string]: number }` nima?"** — Index signature ekvivalenti. Mapped type sintaksisi orqali `Record<string, number>` ga teng.
2. **"Mapped type'da `T[K]` o'rniga boshqa expression yozish mumkinmi?"** — Ha, har qanday type expression (conditional, union, template literal) ishlaydi.

</details>

### Savol 2: Homomorphic va non-homomorphic mapped type — farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Homomorphic** — `keyof T` dan key oladigan mapped type, original modifier'larni (`readonly`, `?`) saqlaydi. **Non-homomorphic** — boshqa source ishlatadi (masalan `Record<K, V>`), modifier'larni yo'qotadi.

### To'liq tushuntirish

Homomorphic mapped type — `{ [K in keyof T]: ... }` shaklida bo'lib, T tipi instantiated bo'lganda compiler T'ning structural ma'lumotini (jumladan modifier'larni) tahlil qiladi va natija type'ga ko'chiradi. Bu special behavior — `Partial<T>`, `Required<T>`, `Readonly<T>`, `Pick<T, K>` shu mexanizmga asoslangan.

Non-homomorphic mapped type — `{ [P in K]: ... }` shaklida (masalan `Record<K, V>`) — K type T'dan kelmaydi, compiler structural ma'lumotni ko'chirmaydi, faqat yangi shape yaratadi.

### Kod misol

```typescript
interface Config {
  readonly host: string;
  readonly port: number;
  timeout?: number;
}

// Homomorphic — modifier'lar saqlanadi
type PartialConfig = Partial<Config>;
// { readonly host?: string; readonly port?: number; timeout?: number }
// ✅ readonly saqlandi, timeout optional qoldi

// Non-homomorphic — modifier'lar yo'qoladi
type RecordConfig = Record<keyof Config, string>;
// { host: string; port: string; timeout: string }
// ❌ readonly VA optional yo'qoldi

// Custom homomorphic — modifier saqlanadi
type Wrap<T> = { [K in keyof T]: { value: T[K] } };
type WrappedConfig = Wrap<Config>;
// { readonly host: { value: string }; readonly port: { value: number }; timeout?: { value: number } }
```

### Edge Cases

- `keyof T` o'rniga `K extends keyof T` qo'yilsa ham — homomorphic xususiyat saqlanadi (Pick shu sababli homomorphic)
- `T extends any ? { [K in keyof T]: T[K] } : never` — `T extends any` distribution union'larni alohida-alohida ishlatadi
- `Mapped<T> = { [K in keyof T & string]: T[K] }` — `& string` intersection homomorphism'ni buzadi, modifier'lar yo'qoladi
- `as` clause ishlatilganda ham homomorphic saqlanadi: `{ [K in keyof T as ...]: T[K] }`

### Follow-up savollar

1. **"Nima uchun `Record` non-homomorphic?"** — `Record<K, T> = { [P in K]: T }` — P type K dan oladi, lekin K T bilan bog'liq emas (K ko'pincha union literal). Compiler structural info ko'chirmaydi.
2. **"Homomorphism'ni saqlash uchun qoidalar?"** — Source `keyof T` yoki `keyof T & X` emas (faqat `keyof T`). `K in keyof T` shaklini saqlash kerak.

</details>

### Savol 3: Property modifier'lar — `+?`, `-?`, `+readonly`, `-readonly` qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`+` modifier'ni qo'shadi, `-` olib tashlaydi. `?` optional, `readonly` immutable. Default `+` (shuning uchun `?` = `+?`). `-?` optional marker'ni va u bilan implicit qo'shilgan `undefined`'ni olib tashlaydi (explicit `T | undefined` union member'iga ta'sir qilmaydi).

### To'liq tushuntirish

TS 2.8'gacha modifier faqat qo'shilardi (`?`, `readonly`). TS 2.8+ esa `-` operator olib tashlash imkonini berdi. Bu `Required<T>` va `Mutable<T>` utility'larni implement qilish uchun zarur.

Default behavior:
- `?` — `+?` ga teng (optional qo'shish)
- `readonly` — `+readonly` ga teng (readonly qo'shish)

`-?` semantics nuansi:
- `b?: string` — bu `b?: string` (marker bor, implicit `undefined` qo'shilgan). `-?` qo'llansa → `b: string` (marker va implicit undefined ketadi)
- `c: string | undefined` — marker yo'q, faqat union member sifatida explicit `undefined`. `-?` ta'sir qilmaydi — `c: string | undefined` qoladi. Explicit undefined'ni olib tashlash uchun `NonNullable<T[K]>` kerak

### Kod misol

```typescript
// Required — optional olib tashlash
type MyRequired<T> = { [K in keyof T]-?: T[K] };

// Mutable — readonly olib tashlash
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// Strict — hammasi required VA mutable
type Strict<T> = { -readonly [K in keyof T]-?: T[K] };

interface Example {
  a: string;
  b?: string;             // optional = string | undefined (marker bilan)
  c: string | undefined;  // required, lekin explicit | undefined union
  readonly d: number;
}

type Req = MyRequired<Example>;
// {
//   a: string;
//   b: string;              // ? marker va u bilan kelgan undefined ketdi
//   c: string | undefined;  // ❗ explicit undefined qoladi (-? ta'sir qilmaydi)
//   readonly d: number;     // readonly saqlandi
// }

// Explicit undefined'ni olib tashlash uchun:
type StrictReq = { [K in keyof Example]-?: NonNullable<Example[K]> };
// {
//   a: string;
//   b: string;
//   c: string;              // ✅ NonNullable undefined'ni olib tashladi
//   readonly d: number;
// }

type Mut = Mutable<Example>;
// {
//   a: string;
//   b?: string;           // optional saqlandi
//   c: string | undefined;
//   d: number;            // readonly olib tashlandi
// }
```

### Edge Cases

- `-?` faqat optional marker'ni va u bilan kelgan implicit `undefined`'ni olib tashlaydi. Explicit `T | undefined` ga ta'sir qilmaydi
- `-?` value type'dagi `null`'ni hech qachon olib tashlamaydi. `null` qoldirish uchun `NonNullable<T[K]>` kerak
- Modifier mapped type ichida ishlaydi, oddiy property declaration'da `-?` ishlamaydi
- `+?` va `?` — bir xil natija, lekin `+?` explicit (kod o'qish uchun foydali)

### Follow-up savollar

1. **"`Required<T>` `phone: string | undefined` ni nima uchun tozalamaydi?"** — `Required<T>` `-?` ishlatadi. Bu marker'ni olib tashlaydi. `phone: string | undefined` — marker'siz, `-?` undefined'ni olib tashlamaydi. `StrictRequired<T> = { [K in keyof T]-?: NonNullable<T[K]> }` kerak.
2. **"Nima uchun TS 2.8 dan oldin `Required` yo'q edi?"** — `-` modifier syntax 2.8'da kiritilgan. Undan oldin optional'ni olib tashlash mexanizmi yo'q edi.

</details>

### Savol 4: Key remapping (`as`) — nima beradi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Key remapping (TS 4.1+) — mapped type ichida `as` clause bilan key'ni qayta nomlash yoki filtrlash. `as never` qaytarilsa property butunlay olib tashlanadi.

### To'liq tushuntirish

Klassik mapped type `[K in keyof T]: V` — key'ni o'zgartirib bo'lmaydi. TS 4.1+ `as` clause kiritildi: `[K in keyof T as NewKey]: V`. NewKey har qanday `PropertyKey` (`string | number | symbol`) bo'lishi mumkin.

Asosiy ishlatish holatlari:
1. **Key qayta nomlash** — template literal bilan prefix/suffix qo'shish (`get${Capitalize<K>}`)
2. **Key filtrlash** — `K extends Condition ? K : never` — `never` qaytsa property olib tashlanadi
3. **Conditional remapping** — value type'ga qarab key'ni o'zgartirish

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

// 1. Key qayta nomlash — getter'lar generate
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; getEmail: () => string }

// 2. Key filtrlash — value type bo'yicha
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
type StringFields = OnlyStrings<User>;
// { name: string; email: string }
// age — never key bilan filterlandi

// 3. Multiple key'lar har property uchun
type GetterSetter<T> = {
  [K in keyof T as `get${Capitalize<string & K>}` | `set${Capitalize<string & K>}`]: (...args: any[]) => any;
};
```

### Edge Cases

- `string & K` — `Capitalize` faqat `string` qabul qiladi. `keyof T` esa `string | number | symbol`. `string & K` filter — number/symbol key'larni `never` orqali skip qiladi
- `as never` — property butunlay olib tashlanadi (yangi key yaratilmaydi)
- Template literal va `as` birga — `keyof T` da `symbol` bo'lsa, `string & K` bilan tozalash zarur
- Homomorphism saqlanadi — `as` ishlatilganda ham modifier'lar ko'chiriladi

### Follow-up savollar

1. **"`as never` ichki mexanizmi qanday?"** — Mapped type natijasida property key'i `never` bo'lsa, compiler shu property'ni butunlay yaratmaydi (chunki `never` qiymat type'sida mavjud emas).
2. **"`as K` (o'zgarishsiz) yozish — nima uchun foydali bo'lishi mumkin?"** — Explicit documentation uchun yoki conditional logic ichida bir branch'da key o'zgartirish, boshqasida saqlash.

</details>

### Savol 5: `Record<K, V>` qanday implement qilingan va mapped type'dan farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Record<K, V> = { [P in K]: V }` — non-homomorphic mapped type. K har qanday `PropertyKey` (`string | number | symbol`), V har property uchun bir xil. Homomorphic mapped type'dan farqi: original modifier'larni saqlamaydi.

### To'liq tushuntirish

`Record` `lib.es5.d.ts` da quyidagicha:

```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
// keyof any = string | number | symbol
```

`K extends keyof any` — har qanday valid property key qabul qiladi. Bu Pick'dan farqi — Pick'da `K extends keyof T` (faqat T'da mavjud key'lar).

Mexanizm: `[P in K]` — K union'ining har member'i uchun bir xil V type'li property yaratadi. K T bilan bog'liq emas, shuning uchun T'ning modifier'lari ko'chirilmaydi (non-homomorphic).

### Kod misol

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

// Record — non-homomorphic
type RecordResult = Record<keyof Config, boolean>;
// { host: boolean; port: boolean }
// readonly va optional yo'qoldi

// Mapped type — homomorphic
type MappedResult = { [K in keyof Config]: boolean };
// { readonly host: boolean; port?: boolean }
// modifier'lar saqlandi

// Record real use case — string union'dan object
type Role = "admin" | "user" | "guest";
type Permissions = Record<Role, string[]>;
// { admin: string[]; user: string[]; guest: string[] }

const perms: Permissions = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  guest: ["read"],
};

// Record + Pick — kombinatsiya
type StringRecord<K extends string> = Record<K, string>;
type ColorMap = StringRecord<"primary" | "secondary">;
// { primary: string; secondary: string }
```

### Edge Cases

- `Record<string, T>` — index signature ekvivalenti (`{ [key: string]: T }`)
- `Record<never, T>` → `{}` (bo'sh object)
- `keyof Record<string, T>` → `string` (number emas). Bu literal index signature'dan farq qiladi: `keyof { [key: string]: T }` → `string | number`. Sabab — `Record` mapped type (`[P in string]`) sifatida hisoblanadi, literal index signature esa `number` coercion'ni ham qo'shadi (intentional inconsistency)
- `Record<K, T>` da K literal union bo'lsa — exhaustive object talab qilinadi (har key kerak)

### Follow-up savollar

1. **"`Record<keyof T, V>` o'rniga `{ [K in keyof T]: V }` qachon afzal?"** — Original modifier'larni saqlash kerak bo'lganda (homomorphic).
2. **"`Record<K, V>` bilan dynamic object qanday yaratiladi?"** — `Record<string, V>` index signature beradi, lekin har string key valid bo'ladi (typo tutmaydi).

</details>

### Savol 6: `keyof` operator'i — turli type'lar bilan qanday natija beradi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`keyof T` — T object type'ning property name'larini union sifatida beradi. Index signature bilan natija index type'iga teng. `keyof unknown` → `never`, `keyof any` → `string | number | symbol`.

### To'liq tushuntirish

`keyof` natijasi quyidagi qoidalar bo'yicha hisoblanadi:
- **Object literal** — declared key'lar union
- **Index signature** — index type (declared property'lar yo'qoladi)
- **Union type** — common key'lar (intersection)
- **Intersection type** — barcha key'lar (union)
- **Primitive** — primitive'ning method nomlari
- **`any`** — `string | number | symbol`
- **`unknown`** — `never` (hech qaysi key access qilinmaydi)

### Kod misol

```typescript
// Oddiy object
type A = keyof { name: string; age: number };
// "name" | "age"

// Index signature
type B = keyof { [key: string]: any };
// string | number
// Sabab: JS da obj[0] === obj["0"] — number ham valid

type C = keyof { [key: number]: any };
// number

// Array
type D = keyof string[];
// number | "length" | "push" | "pop" | ... (Array method nomlari)

// Primitive
type E = keyof string;
// "toString" | "charAt" | "length" | ... (String prototype method'lari)

// any va unknown
type F = keyof any;      // string | number | symbol (PropertyKey)
type G = keyof unknown;  // never
type H = keyof never;    // string | number | symbol

// Union
type I = keyof ({ a: 1; b: 2 } | { a: 3; c: 4 });
// "a" — faqat umumiy key

// Intersection
type J = keyof ({ a: 1 } & { b: 2 });
// "a" | "b"

// Index signature + declared key
type WithIndex = { [key: string]: any; name: string };
type K = keyof WithIndex;
// string | number
// "name" yo'qoldi! Index signature mavjud bo'lganda declared key'lar union'ga qo'shilmaydi
```

### Edge Cases

- `keyof unknown` → `never` — unknown'da hech qaysi property access qilinmaydi
- `keyof any` → `PropertyKey` (`string | number | symbol`) — har qanday key valid
- Index signature mavjud bo'lganda declared key'lar union'ga qo'shilmaydi — TS limitation
- `keyof string` — String.prototype method nomlari (`"charAt"`, `"length"`, ...), `string` literal emas
- Symbol property'lar `keyof` natijasiga kiradi — template literal'da ishlatish uchun `string & K` filter zarur

### Follow-up savollar

1. **"Index signature bor object'da declared key'larni qanday olish mumkin?"** — `{ [K in keyof T]: string extends K ? never : number extends K ? never : K }[keyof T]` pattern bilan.
2. **"`keyof typeof obj` nima qiladi?"** — Runtime object'dan type olib, uning key'larini union qiladi. Const object bilan literal type beradi.

</details>

### Savol 7: Mapped + Conditional type — qanday birga ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Mapped type ichida har property uchun conditional type qo'llaniladi. Bu value type'ni filter qilish (`PickByType`) yoki transform qilish (function methods'ni `Promise`'ga aylantirish) uchun ishlatiladi.

### To'liq tushuntirish

Conditional type mapped type ichida ikki pozitsiyada qo'llaniladi:
1. **Value position** — `{ [K in keyof T]: T[K] extends X ? Y : Z }` — har value'ni shartli transform qilish
2. **Key remapping position** — `{ [K in keyof T as T[K] extends X ? K : never]: T[K] }` — value type bo'yicha property filter

Bu kombinatsiya advanced type-level programming asosi: function method'larini ajratish, async wrapper yaratish, type bo'yicha discriminated union yig'ish.

### Kod misol

```typescript
interface Service {
  id: number;
  name: string;
  start(): void;
  stop(): void;
  config: { port: number };
}

// 1. Function method'larni ajratish (key filter)
type Methods<T> = {
  [K in keyof T as T[K] extends Function ? K : never]: T[K];
};
type ServiceMethods = Methods<Service>;
// { start(): void; stop(): void }

// 2. Non-function property'lar (data fields)
type DataFields<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};
type ServiceData = DataFields<Service>;
// { id: number; name: string; config: { port: number } }

// 3. Method return'larni Promise'ga wrap qilish (value transform)
type Asyncified<T> = {
  [K in keyof T]: T[K] extends (...args: infer A) => infer R
    ? (...args: A) => Promise<R>
    : T[K];
};
type AsyncService = Asyncified<Service>;
// { id: number; name: string; start(): Promise<void>; stop(): Promise<void>; config: { port: number } }

// 4. Combined — function key'larni ajratib, ularni async qilish
type AsyncMethods<T> = {
  [K in keyof T as T[K] extends Function ? K : never]:
    T[K] extends (...args: infer A) => infer R ? (...args: A) => Promise<R> : never;
};
```

### Edge Cases

- `T[K] extends Function` — class method'larini ham qamrab oladi, lekin getter/setter (accessor) emas
- Value position'da conditional distribution — agar `T[K]` union bo'lsa, har member alohida tekshiriladi
- `extends Function` o'rniga `extends (...args: any) => any` — strictroq, faqat callable
- `never` value position'da — property type'i `never` bo'ladi (key qoladi, lekin qiymat berib bo'lmaydi)

### Follow-up savollar

1. **"`never` key position'da vs value position'da farqi?"** — Key position'da (`as never`) property butunlay olib tashlanadi. Value position'da (`: never`) property qoladi, lekin qiymat berib bo'lmaydi.
2. **"Method'larni ajratish uchun nima uchun `extends Function` yetarli emas?"** — `extends (...args: any) => any` strictroq, faqat call-signature'li type'lar; `Function` esa keng (har qanday function-like object).

</details>

### Savol 8: Recursive mapped type — qanday yoziladi va xavfi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Recursive mapped type — value type'ni rekursiv transform qiladi (`DeepReadonly`, `DeepPartial`). Cheksiz rekursiyani oldini olish uchun base case (primitive, function) tekshirish zarur. Non-tail recursive instantiation ~50 darajagacha, tail recursive (TS 4.5+) 1000 darajagacha ruxsat etiladi.

### To'liq tushuntirish

Recursive mapped type — value position'da conditional type orqali o'ziga murojaat qiladi:

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

Xavf — base case noto'g'ri bo'lsa noto'g'ri natija:
- **Function** `extends object` true. Homomorphic mapped type function type'ga qo'llanganda call signature yo'qoladi — `keyof (() => void)` = `never`, natija `{}` (bo'sh object, callable emas)
- **Array/Tuple** — homomorphic mapped type array'ga qo'llanganda compiler array shape'ni saqlaydi (element type transform qilinadi). Demak naive variant array'ni buzmaydi, lekin element'larni rekursiv transform qilish va `readonly` array'ga aylantirish nazoratini explicit `ReadonlyArray` branch beradi
- **Built-in object'lar** (Date, Map, Set) — `extends object` true, property'lari readonly bo'ladi, lekin internal slot'lar va method'lar (runtime mutable) shu holatda qoladi

To'g'ri pattern — function, array va built-in "leaf" type'larni base case'da alohida ishlash.

### Kod misol

```typescript
// ❌ Naive — function uchun noto'g'ri
type NaiveDeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? NaiveDeepReadonly<T[K]> : T[K];
};

interface Config {
  port: number;
  callback: () => void;
  tags: string[];
  nested: { host: string };
}

type Bad = NaiveDeepReadonly<Config>;
// {
//   readonly port: number;
//   readonly callback: {};                  // ❌ call signature yo'qoldi (keyof fn = never)
//   readonly tags: readonly string[];       // array homomorphic saqlandi
//   readonly nested: { readonly host: string };
// }

// ✅ Proper — function, array uchun alohida branch
type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :
  T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

type Good = DeepReadonly<Config>;
// {
//   readonly port: number;
//   readonly callback: () => void;        // function saqlandi
//   readonly tags: ReadonlyArray<string>; // array → ReadonlyArray
//   readonly nested: { readonly host: string };
// }

// DeepPartial — har darajada optional
type DeepPartial<T> =
  T extends (...args: any[]) => any ? T :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;
```

### Edge Cases

- **Rekursiya limit'i (~50 daraja non-tail)** — circular type yoki juda chuqur nesting'da `Type instantiation is excessively deep` error
- **`Date`, `RegExp`, `Map`, `Set`** — `extends object` true, lekin internal slot'lar bor. `T extends Date ? T : ...` qo'shish kerak
- **Tuple** — `[number, string]` `extends ReadonlyArray<U>` true, lekin element type'larini alohida transform qilish kerak
- **Union member** — `T extends object` distribution union'ni ajratadi, har member alohida traverse qilinadi

### Follow-up savollar

1. **"Cheksiz rekursiyani qanday topish mumkin?"** — Compiler `Type instantiation is excessively deep and possibly infinite` error chiqaradi.
2. **"`DeepReadonly` Map/Set bilan qanday ishlaydi?"** — Default behavior: Map/Set property'lari readonly bo'ladi, lekin `set()`, `add()` method'lari ishlaydi (runtime'da). To'liq immutability uchun `ReadonlyMap<K, V>`, `ReadonlySet<T>` ishlatish kerak.

<details>
<summary><strong>Deep Dive</strong></summary>

Compiler recursive mapped type'ni qanday instantiate qiladi: har rekursiv chaqiruvda yangi type instance generate qilinadi. Compiler bir xil type'ni bir xil type argument'lar bilan qayta instantiate qilmasligi uchun natijalarni cache'laydi. Lekin generic parameter har chaqiruvda yangi bo'lsa, cache hit yo'q, compile vaqt o'sadi.

Instantiation depth limit:
- Non-tail recursive: ~50 daraja (type instantiation depth)
- Tail recursive (TS 4.5+): 1000 daraja — conditional type oxirida yana conditional type bo'lsa, compiler resolution'ni loop'da bajaradi (qo'shimcha call stack ishlatmaydi)
- `Type instantiation is excessively deep and possibly infinite` error — depth yoki instantiation count limit'dan oshganda

Bypass strategiyalari:
- Conditional `T extends infer U ? ... : never` — `U` yangi binding sifatida compiler'ga yordam beradi (lazy evaluation)
- Manual depth limit — tuple length tracker (`Depth extends [...Depth, any]`)
- Tail-recursive refactor — recursive call eng oxirgi pozitsiyada

Depth limit type checker'ning ichki hisoblagichi — bu runtime call stack emas, balki instantiation chuqurligini kuzatuvchi counter. Limit'dan oshganda compiler `excessively deep and possibly infinite` error chiqaradi va instantiation'ni to'xtatadi. Compile time esa unique type instantiation soniga proportional o'sadi: cache hit bo'lganda qayta hisoblanmaydi, har chaqiruvda yangi type argument bo'lsa cache miss va qo'shimcha ish.

</details>

</details>

---

## Output savollari

### Savol 9: Modifier'lar output — bir nechta variant [Middle]

**Savol:** Har type'ning natijasini ayting:

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
type F = Record<keyof Data, boolean>;
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Har variant homomorphic mexanizmni va modifier interaction'ni tekshiradi. `F` non-homomorphic — modifier'lar yo'qoladi.

### Kod misol

```typescript
type A = {
  readonly id: number;
  name: string;
  tags?: string[];
};
// Homomorphic — modifier'lar saqlanadi

type B = {
  readonly id?: number;
  name?: string;
  tags?: string[];
};
// +? qo'shildi, readonly saqlandi

type C = {
  id: number;
  name: string;
  tags?: string[];
};
// -readonly olib tashlandi, optional saqlandi

type D = {
  readonly id: number;
  name: string;
  tags: string[];
};
// -? olib tashlandi, readonly saqlandi
// tags endi string[] (undefined ham ketdi)

type E = {
  readonly idInfo: number;
  nameInfo: string;
  tagsInfo?: string[];
};
// Key remapping — homomorphic xususiyat saqlanadi

type F = {
  id: boolean;
  name: boolean;
  tags: boolean;
};
// Record — non-homomorphic
// readonly VA optional yo'qoldi
```

### Edge Cases

- E variant — key remapping ishlatilganda ham `keyof T` source ekani uchun homomorphic
- D variant'da `tags: string[]` — `-?` undefined'ni ham value type'dan olib tashladi

### Follow-up savollar

1. **"F'da modifier saqlash uchun nima qilish kerak?"** — `Record` o'rniga `{ [K in keyof Data]: boolean }` yozish.

</details>

### Savol 10: `keyof` + index signature output [Middle]

**Savol:** Quyidagi type'lar uchun `keyof` natijasi nima?

```typescript
type T1 = { name: string; age: number };
type T2 = { [key: string]: any };
type T3 = { [key: string]: any; name: string };
type T4 = string[];
type T5 = keyof any;
type T6 = keyof unknown;
type T7 = keyof never;

type K1 = keyof T1;
type K2 = keyof T2;
type K3 = keyof T3;
type K4 = keyof T4;
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`keyof` behavior har holatga qarab farqlanadi. Index signature mavjud bo'lganda declared key'lar union'ga qo'shilmaydi — bu eng tricky qoida.

### Kod misol

```typescript
type K1 = keyof T1;
// "name" | "age"

type K2 = keyof T2;
// string | number
// Sabab: index signature [key: string] — JS'da number key'lar string'ga coerce qilinadi

type K3 = keyof T3;
// string | number
// ❗ "name" yo'qoldi — index signature mavjud bo'lganda declared key'lar union'ga qo'shilmaydi

type K4 = keyof T4;
// number | "length" | "push" | "pop" | "concat" | ... (Array method nomlari)

type T5 = keyof any;     // string | number | symbol
type T6 = keyof unknown; // never
type T7 = keyof never;   // string | number | symbol
```

### Edge Cases

- T3 holatida `name` key'ini olish uchun helper kerak: `{ [K in keyof T]: string extends K ? never : K }[keyof T]`
- T4 — `keyof array` faqat `number` emas, `length`, `push` va boshqalar
- `keyof Number`, `keyof String` — primitive prototype method'lari

### Follow-up savollar

1. **"Nima uchun `keyof unknown = never`?"** — unknown'da hech qaysi property access mumkin emas, shuning uchun key yo'q.
2. **"`keyof Function` nima qaytaradi?"** — `"apply" | "call" | "bind" | "prototype" | ...` — Function prototype method'lari.

</details>

### Savol 11: `as` clause output [Middle+]

**Savol:** Har type'ning natijasini ayting:

```typescript
interface Obj {
  name: string;
  age: number;
  email: string;
}

type A = { [K in keyof Obj as Uppercase<string & K>]: Obj[K] };
type B = { [K in keyof Obj as `_${string & K}`]: Obj[K] };
type C = { [K in keyof Obj as Obj[K] extends string ? K : never]: Obj[K] };
type D = { [K in keyof Obj as K extends "name" ? "fullName" : K]: Obj[K] };
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`as` clause har property uchun yangi key generate qiladi. `never` qaytsa property olib tashlanadi.

### Kod misol

```typescript
type A = {
  NAME: string;
  AGE: number;
  EMAIL: string;
};

type B = {
  _name: string;
  _age: number;
  _email: string;
};

type C = {
  name: string;
  email: string;
};
// age — never key bilan filterlandi (number, string emas)

type D = {
  fullName: string;
  age: number;
  email: string;
};
// Faqat "name" → "fullName" ga o'zgartirildi
```

### Edge Cases

- A va B variantlarda `string & K` — Uppercase va template literal faqat string qabul qiladi
- D variantda conditional remapping — bir key o'zgaradi, qolganlari saqlanadi

### Follow-up savollar

1. **"A variantda `string & K` o'rniga `K` yozsak nima bo'ladi?"** — Bu `Obj` (barcha key'lar string) uchun xatosiz compile bo'ladi, chunki `keyof Obj` = `"name" | "age" | "email"` string literal union, `Uppercase`'ning `extends string` constraint'iga mos. `string & K` faqat key'larda `number`/`symbol` bo'lishi mumkin bo'lgan generic `<T>` holatida zarur — u holatda `K` ga `Type 'K' does not satisfy the constraint 'string'` xatosi chiqadi.

</details>

### Savol 12: Recursive mapped output [Senior]

**Savol:** `DeepPartial<Config>` natijasi nima?

```typescript
type DeepPartial<T> =
  T extends (...args: any[]) => any ? T :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;

interface Config {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
  logger: (msg: string) => void;
  tags: string[];
}

type Result = DeepPartial<Config>;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type Result = {
  server?: {
    host?: string;
    port?: number;
    ssl?: {
      enabled?: boolean;
      cert?: string;
    };
  };
  logger?: (msg: string) => void;       // function — base case
  tags?: (string | undefined)[];        // array saqlandi, element'lar optional
};
```

### To'liq tushuntirish

- `server` — object, rekursiv traverse, har nested property optional
- `logger` — function (callable), base case branch — o'zi qaytariladi (faqat top-level optional)
- `tags` — array `extends object` true, mapped type array'ga qo'llanadi. Homomorphic mapped type compiler'da array uchun special-case: array shape saqlanadi, element type transform qilinadi. `?` modifier element type'ga `undefined` qo'shadi → `(string | undefined)[]`. Array obyektga aylanib buzilmaydi, lekin element'lar `undefined` qabul qilishi ko'pincha kutilmagan

Array elementiga `| undefined` qo'shilishi kerak bo'lmasa — Array uchun alohida branch:

```typescript
type DeepPartialFixed<T> =
  T extends (...args: any[]) => any ? T :
  T extends Array<infer U> ? Array<DeepPartialFixed<U>> :
  T extends object ? { [K in keyof T]?: DeepPartialFixed<T[K]> } :
  T;

type FixedResult = DeepPartialFixed<Config>;
// {
//   server?: { ... };
//   logger?: (msg: string) => void;
//   tags?: string[];           // ✅ element'lar undefined emas
// }
```

### Edge Cases

- Array branch element'lardan `| undefined`'ni olib tashlash uchun kerak — `T extends object` array'ni buzmaydi (homomorphic special-case), lekin `?` modifier element'larga `undefined` qo'shadi
- `ReadonlyArray<U>` uchun ham alohida branch kerak (yoki `readonly U[]` shaklini support)
- Tuple type'lar `Array<U>` ga match qiladi, lekin tuple shape (positional types) yo'qoladi

<details>
<summary><strong>Deep Dive</strong></summary>

Recursive mapped type'ning subtle nuance'lari:

**1. Function preservation:** Function ham `extends object` true. Plain function type'da `keyof (() => void)` = `never` (named member yo'q) — mapped type natijasi `{}`, callable signature yo'qoladi. Call/named property'lari bor function uchun esa o'sha property'lar mapped bo'ladi, lekin call signature baribir yo'qoladi. To'g'ri pattern — function branch birinchi.

**2. Array branch tartibi:** `T extends (...args: any[]) => any` birinchi, keyin `T extends Array<infer U>`, oxirida `T extends object`. Sabab: array ham object, function ham object — spetsifik tekshiruv umumiyga qadar.

**3. Distribution muammosi:** `T extends object` distributive (naked T). Union member'lar alohida tekshiriladi. `string[] | number[]` — har biri uchun alohida Array branch ishlaydi, natija `string[] | number[]` (saqlandi).

**4. ReadonlyArray nuance:** TS'da `readonly string[]` va `ReadonlyArray<string>` ekvivalent. Lekin tuple — `readonly [string, number]` `ReadonlyArray<U>` ga match qiladi, U union sifatida (`string | number`), tuple shape yo'qoladi. Tuple uchun explicit `T extends readonly [infer A, ...infer Rest]` recursive pattern kerak.

**5. Type predicate:** `obj is X` type guard funksiya — `extends (...args: any[]) => any` ga mos keladi (function), lekin return type `boolean` — narrow info yo'qoladi DeepPartial'da.

Production strategy: form library'larda (react-hook-form) DeepPartial pattern'ga "safe leaves" pattern qo'shiladi — `Date`, `File`, `RegExp` explicit branch'lar bilan. Aks holda runtime'da `Date.prototype` method'lari optional bo'lib silently buziladi.

</details>

</details>

### Savol 13: `extends Function` filter output [Middle+]

**Savol:** Natijani ayting:

```typescript
interface Mixed {
  id: number;
  name: string;
  start(): void;
  stop(): Promise<void>;
  data: { x: number };
}

type A = { [K in keyof Mixed as Mixed[K] extends Function ? K : never]: Mixed[K] };
type B = { [K in keyof Mixed as Mixed[K] extends Function ? never : K]: Mixed[K] };
type C = { [K in keyof Mixed]: Mixed[K] extends Function ? "method" : "data" };
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = {
  start(): void;
  stop(): Promise<void>;
};
// Faqat function'lar

type B = {
  id: number;
  name: string;
  data: { x: number };
};
// Function'larsiz

type C = {
  id: "data";
  name: "data";
  start: "method";
  stop: "method";
  data: "data";
};
// Har property — "method" yoki "data" literal
```

### Edge Cases

- A va B'da `as never` mexanizmi — property butunlay olib tashlanadi
- C'da key qoladi, value literal'ga aylanadi

</details>

---

## Coding challenges

### Savol 14: `Pick` va `Omit`'ni qo'lda implement qiling [Middle]

**Savol:** Mapped type ishlatib `MyPick` va `MyOmit` yozing. `Omit`'ni ikki xil usulda implement qiling:

```typescript
interface User { id: number; name: string; age: number; email: string }
// MyPick<User, "name" | "email"> → { name: string; email: string }
// MyOmit<User, "email"> → { id: number; name: string; age: number }
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`Pick` — tanlangan key'larni saqlash. `Omit` — ikki yondashuv: (1) Pick + Exclude kombinatsiyasi, (2) Key remapping bilan `never` filter.

### Kod misol

```typescript
// Pick — tanlangan key'larni olish
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit — variant 1: Pick + Exclude
type MyOmit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// Omit — variant 2: key remapping (TS 4.1+)
type MyOmit2<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

// Test
interface User {
  id: number;
  name: string;
  age: number;
  email: string;
}

type Picked = MyPick<User, "name" | "email">;
// { name: string; email: string }

type Omitted = MyOmit<User, "email">;
// { id: number; name: string; age: number }

type Omitted2 = MyOmit2<User, "email">;
// { id: number; name: string; age: number }
```

### Edge Cases

- `Pick`'da `K extends keyof T` — faqat T'da mavjud key'lar (typo tutadi)
- `Omit`'da `K extends keyof any` — har qanday key qabul qilinadi (typo tutmaydi — loose)
- `MyOmit2` homomorphic — modifier'lar saqlanadi
- `MyOmit` (variant 1) ham homomorphic (Pick orqali)

### Follow-up savollar

1. **"Qaysi variant afzal?"** — Standart `lib.es5.d.ts` Variant 1 (`Pick<T, Exclude<keyof T, K>>`) ishlatadi. Variant 2 (key remapping) TS 4.1+ kiritilgan, lekin standart implementation o'zgarmagan. Ikkalasi ham homomorphic, semantic teng.
2. **"`StrictOmit` qanday yoziladi?"** — `type StrictOmit<T, K extends keyof T> = Omit<T, K>` — `K extends keyof T` strict bound qo'shadi.

</details>

### Savol 15: `PickByType` va `OmitByType` implement qiling [Middle+]

**Savol:** Value type bo'yicha property'larni tanlash/olib tashlash:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
}

// PickByType<User, string> → { name: string; email: string }
// OmitByType<User, string> → { id: number; age: number; active: boolean }
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Key remapping + conditional type kombinatsiyasi. `as T[K] extends V ? K : never` — match bo'lsa key saqlanadi, aks holda olib tashlanadi.

### Kod misol

```typescript
type PickByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? K : never]: T[K];
};

type OmitByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? never : K]: T[K];
};

interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
}

type StringProps = PickByType<User, string>;
// { name: string; email: string }

type NumberProps = PickByType<User, number>;
// { id: number; age: number }

type NonStringProps = OmitByType<User, string>;
// { id: number; age: number; active: boolean }

type NonPrimitive = OmitByType<User, string | number | boolean>;
// {} (hammasi primitive bo'lgani uchun)
```

### Edge Cases

- `T[K] extends ValueType` — exact match emas, structural compatibility tekshiradi
- `T[K]` indexed access — naked type parameter emas, shuning uchun `T[K] extends V` **distribute qilmaydi**. `T[K]` union bo'lsa (`string | null`), butun union bir butun sifatida tekshiriladi: `(string | null) extends string` → false → property filterlanadi (match emas)
- Element distribution kerak bo'lsa naked param orqali: `type IsString<X> = X extends string ? true : false` — X naked bo'lgani uchun union har member'i alohida tekshiriladi
- Strict equality kerak bo'lsa: `T[K] extends V ? V extends T[K] ? K : never : never`

### Follow-up savollar

1. **"Optional property'lar bilan qanday ishlaydi?"** — `T[K]` optional bo'lsa value type'iga `undefined` qo'shilgan bo'ladi. `extends string` false bo'lishi mumkin.
2. **"Strict (exact) type matching qanday yozish kerak?"** — Mutual extends check: `T[K] extends V ? V extends T[K] ? K : never : never`.

</details>

### Savol 16: Object Diff — ikki type orasidagi farq [Senior]

**Savol:** `Diff<T, U>` va `Common<T, U>` utility'larni yozing. API versioning uchun `MigrationPlan` yarating:

```typescript
interface V1 { id: number; name: string; email: string }
interface V2 { id: number; name: string; phone: string; avatar: string }

// Diff<V2, V1> → { phone: string; avatar: string } (yangi field'lar)
// Diff<V1, V2> → { email: string } (o'chirilgan field'lar)
// Common<V1, V2> → { id: number; name: string } (umumiy)
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`Diff` — T'da bor, U'da yo'q key'larni olish. `Exclude<keyof T, keyof U>` — T key'laridan U key'larini chiqarib tashlash. `Common` — `Extract` bilan kesishuv.

### Kod misol

```typescript
// T da bor, U da yo'q key'lar
type Diff<T, U> = {
  [K in Exclude<keyof T, keyof U>]: T[K];
};

// T va U da umumiy key'lar
type Common<T, U> = {
  [K in Extract<keyof T, keyof U>]: T[K];
};

interface V1 { id: number; name: string; email: string }
interface V2 { id: number; name: string; phone: string; avatar: string }

type Added = Diff<V2, V1>;     // { phone: string; avatar: string }
type Removed = Diff<V1, V2>;   // { email: string }
type Shared = Common<V1, V2>;  // { id: number; name: string }

// API versioning migration plan
type MigrationPlan<Old, New> = {
  added: Diff<New, Old>;
  removed: Diff<Old, New>;
  shared: Common<Old, New>;
};

type Plan = MigrationPlan<V1, V2>;
// {
//   added: { phone: string; avatar: string };
//   removed: { email: string };
//   shared: { id: number; name: string };
// }

// Strict diff — value type ham mos kelishi shart
type StrictCommon<T, U> = {
  [K in Extract<keyof T, keyof U>]: T[K] extends U[K & keyof U]
    ? U[K & keyof U] extends T[K] ? T[K] : never
    : never;
};
```

### Edge Cases

- Optional key'lar — `email?: string` `Diff` da `email?: string` qoladi (modifier'lar saqlanadi — homomorphic)
- Value type mos kelmasa — `Common` faqat T'dan value oladi, type mismatch yashirin qoladi
- `Diff<{}, T>` → `{}` — bo'sh object'dan diff har doim bo'sh

### Follow-up savollar

1. **"Value type ham mos kelishini tekshirish uchun?"** — `StrictCommon` (yuqorida) — mutual extends bilan.
2. **"Nested diff qanday yoziladi?"** — Recursive — har nested key uchun rekursiv `Diff` chaqirish.

<details>
<summary><strong>Deep Dive</strong></summary>

API versioning real-world use case: REST API'da V1 → V2 migration. Backend va frontend bir vaqtda yangilanmaydi — har tomon o'z version'ini biladi. Type-level diff'lar:

- **Added fields:** yangi field'lar — old client backward compatibility uchun optional yoki default value beriladi
- **Removed fields:** old client deprecated field'larga murojaat qilishi mumkin — runtime'da no-op yoki warning
- **Modified fields:** type structure o'zgargan field'lar — `StrictCommon` bilan aniqlanadi

**Compile-time vs runtime:**
- Type-level diff faqat shape difference'ni aniqlaydi
- Runtime'da field validation alohida (Zod, Yup, io-ts schema)
- Type generation tools (OpenAPI codegen, GraphQL codegen) shu pattern'ga tayanadi

**Homomorphic preservation:** `Diff<T, U>` Pick orqali implement qilingan bo'lsa modifier'lar saqlanadi:
```typescript
type Diff<T, U> = Pick<T, Exclude<keyof T, keyof U>>;
// readonly va ? saqlanadi
```

Lekin `{ [K in Exclude<keyof T, keyof U>]: T[K] }` direct mapped — source `Exclude<keyof T, keyof U>`, bu `keyof T` emas — non-homomorphic, modifier'lar yo'qoladi. Bu subtle nuance — production code'da Pick variant afzal.

Real implementation: Stripe API SDK, GraphQL Codegen, Prisma migrations — barchasi DiffType pattern bilan version compatibility tekshiradi.

</details>

</details>

### Savol 17: Type-safe event bus — mapped type bilan [Senior]

**Savol:** Event map'dan `EventBus<T>` interface generate qiling — `on`, `off`, `emit` method'lari type-safe bo'lsin:

```typescript
interface AppEvents {
  userLogin: { userId: number; timestamp: number };
  userLogout: { userId: number };
  error: { code: number; message: string };
}

// EventBus<AppEvents>:
// on(event: "userLogin", handler: (data: { userId: number; timestamp: number }) => void): void
// emit(event: "userLogin", data: { userId: number; timestamp: number }): void
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Event map — key event nomi, value payload type. Mapped type generic constraint orqali har event uchun type-safe handler signature generate qilinadi.

### Kod misol

```typescript
interface AppEvents {
  userLogin: { userId: number; timestamp: number };
  userLogout: { userId: number };
  error: { code: number; message: string };
}

type EventBus<T> = {
  on<K extends keyof T>(event: K, handler: (data: T[K]) => void): void;
  off<K extends keyof T>(event: K, handler: (data: T[K]) => void): void;
  emit<K extends keyof T>(event: K, data: T[K]): void;
};

declare const bus: EventBus<AppEvents>;

bus.on("userLogin", (data) => {
  // data: { userId: number; timestamp: number }
  console.log(data.userId, data.timestamp);
});

bus.emit("userLogout", { userId: 42 });
// userLogout payload type — { userId: number }

bus.emit("error", { code: 500, message: "Server error" });

// bus.emit("userLogin", { userId: 1 });
// ❌ Property 'timestamp' is missing

// bus.on("unknown", () => {});
// ❌ "unknown" is not assignable to keyof AppEvents
```

### Edge Cases

- Event nomi `keyof T` bilan cheklangan — typo compile error
- Handler payload `T[K]` — exact match, qo'shimcha field'lar `excess property check`
- Off bilan handler reference — `===` bilan solishtiriladi (runtime), type emas
- Asinxron handler uchun `Promise<void>` return type qo'shish: `(data: T[K]) => void | Promise<void>`

### Follow-up savollar

1. **"Multiple handler'lar uchun qanday qilish kerak?"** — Internal storage `Map<keyof T, Set<Handler>>` bilan.
2. **"Wildcard event (`*`) qo'shish?"** — `on("*", (event, data) => {...})` — overload signature, `event: keyof T`, `data: T[keyof T]`.

<details>
<summary><strong>Deep Dive</strong></summary>

Compiler `on<K extends keyof T>` chaqirig'ida `K` ni qanday infer qiladi: contextual typing orqali, birinchi argument literal string'dan. Agar `bus.on("userLogin", ...)` chaqirilsa, `K = "userLogin"`, va `T[K] = { userId: number; timestamp: number }` — handler signature shu darajada cheklangan.

Inference algorithm bosqichlari:
1. Birinchi argument literal type'dan `K` candidate olinadi (`"userLogin"`)
2. `K extends keyof T` constraint tekshiriladi
3. `T[K]` resolve qilinadi va callback parameter type'iga assign qilinadi
4. Callback body'da `data` parametr aniq type bilan ishlaydi

Performance considerations: agar `keyof T` katta union bo'lsa (1000+ event), har `on` chaqirig'i constraint check va `T[K]` indexed access uchun compile vaqt talab qiladi. Real-world katta event bus'larda event'larni domain bo'yicha namespacing — har bus alohida `EventBus<DomainEvents>` instance bilan ishlash.

Type-safe pattern alternativalari:
- **Discriminated union:** `type Event = { type: "userLogin"; payload: ... } | { type: "userLogout"; ... }` — narrowing orqali
- **Conditional payload:** `emit<K>(event: K, payload: T[K])` — current pattern
- **Builder pattern:** `bus.event("userLogin").emit({ ... })` — fluent API

</details>

</details>

---

## Bug fix

### Savol 18: `Capitalize` xato — toping va tuzating [Middle+]

**Savol:** Bu kodda compile error bor. Xatoni toping va uchta usulda tuzating:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<K>}`]: () => T[K];
};
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`K` tipi `keyof T` = `string | number | symbol`. `Capitalize<S>` faqat `string` qabul qiladi:

```
Type 'K' does not satisfy the constraint 'string'.
```

### Kod misol

```typescript
// 1. string & K — intersection bilan
type Getters1<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
// string & number = never → skip, string & symbol = never → skip

// 2. Extract bilan
type Getters2<T> = {
  [K in Extract<keyof T, string> as `get${Capitalize<K>}`]: () => T[K];
};

// 3. keyof T & string
type Getters3<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
};

// Test
interface User {
  name: string;
  age: number;
  [Symbol.iterator]: () => Iterator<any>;
}

type Result = Getters1<User>;
// { getName: () => string; getAge: () => number }
// Symbol key — skip qilindi
```

### Edge Cases

- Variant 1 va 3 — homomorphic xususiyat saqlanadi (`keyof T` source)
- Variant 2 — `Extract<keyof T, string>` ham source `keyof T`'ga asoslangan, homomorphic
- Symbol va number key'lar uchchala variantda skip bo'ladi

### Follow-up savollar

1. **"Nima uchun TS template literal'da `number` ni qabul qilmaydi?"** — Aslida qabul qiladi (`` `${number}` ``), lekin `Capitalize` faqat string. Number key uchun: `K extends number ? `get${K}` : ...`.

</details>

### Savol 19: Recursive type infinite — toping va tuzating [Senior]

**Savol:** Bu kodda nima xato? Tuzating:

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: DeepReadonly<T[K]>;
};

interface Config {
  port: number;
  callback: () => void;
  nested: { host: string };
}

type Result = DeepReadonly<Config>;
// callback noto'g'ri ishlanadi
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

Base case yo'q — `extends object` shartisiz har value `BadDeepReadonly<T[K]>` orqali rekursiv chaqiriladi. Lekin natija primitive va function uchun bir xil emas:

- **Primitive** (`port: number`) — `BadDeepReadonly<number>` baribir `number` qaytaradi. Homomorphic mapped type (`{ [K in keyof T]: ... }`) non-object type'ga qo'llanganda compiler uni o'zgartirmaydi (identity behavior). `keyof number` Number.prototype method nomlarini bersa ham, mapped type ularni iterate qilmaydi — primitive shar holicha qoladi. Demak primitive value buzilmaydi.
- **Function** (`callback: () => void`) — `BadDeepReadonly<() => void>`. Function ham object, lekin homomorphic mapped type call signature'ni ko'chirmaydi: `keyof (() => void)` = `never`, natija `{}` — non-callable. `callback()` chaqirig'i `error TS2349: This expression is not callable` beradi.

Yagona buzilish — function. Base case'siz versiya primitive'ni emas, faqat function call signature'ni yo'qotadi.

### Kod misol

```typescript
// ❌ Noto'g'ri — base case yo'q
type BadDeepReadonly<T> = {
  readonly [K in keyof T]: BadDeepReadonly<T[K]>;
};

// ✅ To'g'ri — function, primitive, array uchun alohida branch
type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :
  T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

interface Config {
  port: number;
  callback: () => void;
  tags: string[];
  nested: { host: string };
}

type Result = DeepReadonly<Config>;
// {
//   readonly port: number;
//   readonly callback: () => void;
//   readonly tags: readonly string[];
//   readonly nested: { readonly host: string };
// }
```

### Edge Cases

- `Date`, `RegExp`, `Map`, `Set` uchun ham branch kerak — aks holda internal state buziladi
- Class instance — `extends object` true, lekin private field'lar accessibility o'zgaradi
- Tuple — `[number, string]` uchun `extends ReadonlyArray<U>` true, lekin tuple shape yo'qoladi

### Follow-up savollar

1. **"Class instance bilan qanday ishlaydi?"** — Class shape readonly bo'ladi, lekin method'larning return value'lari emas. Method'lar funksiya sifatida saqlanadi (base case).
2. **"TS limit'iga yetganda nima qilish kerak?"** — Depth tracker tuple ishlatish: `Depth extends [...Depth, any]` bilan manual cheklash.

<details>
<summary><strong>Deep Dive</strong></summary>

Recursive DeepReadonly'ning compile-time complexity:

**Branch ordering MAJBURIY tartibda:**
1. `T extends (...args: any[]) => any` — function birinchi (function ham object)
2. `T extends ReadonlyArray<infer U>` — array (array ham object)
3. `T extends object` — qolgan object'lar
4. fallback `T` (primitive)

Agar tartib noto'g'ri bo'lsa, umumiy `extends object` branch spetsifik branch'larni shadow qiladi.

**Built-in object'lar muammosi:**
- `Date extends object` true, lekin `[K in keyof Date]` iterate qilsa Date.prototype method'lari readonly bo'ladi (`getTime`, `setTime`, ...)
- `Map<K, V>` — har property readonly, lekin `Map.prototype.set` callable qoladi (runtime'da mutable)
- `Set<T>` — shu muammo

Real solution: `Readonly[Date|RegExp|Map|Set]` aniq branch'lar:
```typescript
type DeepReadonly<T> =
  T extends (...args: any[]) => any ? T :
  T extends Date ? T :
  T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>> :
  T extends Set<infer U> ? ReadonlySet<DeepReadonly<U>> :
  T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;
```

**Type vs runtime immutability:**
- Type-level: `readonly` modifier compile-time check
- Runtime: `Object.freeze()` shallow, `deep-freeze` library deep
- Type-level readonly runtime mutation'ni oldini olmaydi — `(obj as Mutable).x = 1` ishlaydi

Production library'lar (Immer, immutable-js) runtime structural sharing pattern bilan ishlaydi — type-level immutability + runtime persistence.

</details>

</details>

---

## Xulosa

- **Mapped type** — `{ [K in keyof T]: NewType }` — object type'ni transform qilish (Array.map type-level ekvivalenti)
- **Homomorphic** — `keyof T` source ishlatadi, modifier'lar (`readonly`, `?`) saqlanadi
- **Non-homomorphic** (`Record<K, V>`) — modifier'lar yo'qoladi
- **Modifier'lar:** `+?`/`?` (optional qo'shish), `-?` (optional + undefined olib tashlash, TS 2.8+), `+readonly`/`readonly`, `-readonly`
- **Key remapping (`as`, TS 4.1+):** key qayta nomlash (template literal), filtrlash (`never`)
- **`string & K`** — template literal'da number/symbol key'larni filterlash
- **Recursive mapped** — `DeepReadonly`, `DeepPartial` — base case (function, array, primitive) MAJBURIY
- **`keyof` + index signature** = `string | number` (declared key'lar yo'qoladi)
- **Conditional + mapped** — value type bo'yicha key filter (`PickByType`), value transform (`Asyncified`)
