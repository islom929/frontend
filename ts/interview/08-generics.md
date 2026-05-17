# Interview: Generics

> Type parameters, generic functions, generic inference, explicit type arguments, generic interfaces, generic classes, constraints, keyof, index access types, multiple type parameters, default type parameters, const type parameters va generic utility patterns bo'yicha interview savollari.

---

## Nazariy savollar

### Savol 1: Generics nima va nima uchun kerak? `any` dan farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generics — type-level abstraction. Funksiya/interface/class'da aniq type o'rniga type parameter `<T>` qo'yiladi, ishlatilgan paytda aniq type'ga aylanadi. `any` type safety'ni o'chiradi, generic — type'ni saqlaydi.

### To'liq tushuntirish

Generic ikki muammoni hal qiladi: **type safety yo'qolishi** va **kod takrorlanishi**. `any` bilan return type ham `any` — IDE inference yo'q. Generic bilan caller'ga aniq type qaytadi.

### Kod misol

```typescript
// any — type safety yo'q
function firstAny(arr: any[]): any {
  return arr[0];
}
const a = firstAny([1, 2, 3]);
a.toUpperCase(); // TS xato bermaydi, lekin runtime'da crash

// Generic — type safety
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const b = first([1, 2, 3]); // number | undefined
// b.toUpperCase(); // ❌ Compile-time error
const c = first(["a", "b"]); // string | undefined
c?.toUpperCase(); // ✅

// Generic preserve type'ni
interface ApiResponse<T> {
  data: T;
  status: number;
}

const userResponse: ApiResponse<{ name: string }> = {
  data: { name: "Ali" },
  status: 200,
};
userResponse.data.name; // ✅ string
```

### Edge Cases

- **`unknown` vs generic** — `unknown` type-safe `any`, lekin caller'ga type information bermaydi. Generic caller'ning input type'ini saqlaydi.
- **Generic erasure** — JS'ga compile'da type parameter'lar o'chiriladi (type-only). Runtime'da generic information yo'q.
- **`any[]` vs `unknown[]` vs `T[]`** — `any[]` har operation ruxsat (xavfli). `unknown[]` element'ga operation taqiq. `T[]` — caller side aniq type.

### Follow-up savollar

1. **"Generic constraint'siz T da nima qilib bo'ladi?"** — Faqat universal operation'lar: assign, compare reference, pass-through. `T.length`, `T.toString()` constraint kerak.
2. **"Generic inference qachon fail bo'ladi?"** — Argument T'ga bog'liq emas, return type'da ishlatilsa. Yoki ambiguous union argument. Bunda explicit type argument kerak.

</details>

---

### Savol 2: Generic inference qanday ishlaydi? Qachon explicit type argument kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TS argument'lar type'idan generic parametr'ni infer qiladi. Inference fail bo'ladigan holatlar: T faqat return type'da, ambiguous union, contextual type yo'q. Bunda `fn<Type>(args)` explicit beriladi.

### To'liq tushuntirish

**Inference qoidalari:**

1. **Parameter position** — T argument'da ishlatilsa, TS argument type'idan infer qiladi.
2. **Return-only T** — T faqat return type'da ishlatilsa, inference yo'q (default `unknown`).
3. **Multi-position T** — bir nechta parameter'da T ishlatilsa, TS ularning union'ini oladi.
4. **Contextual type** — caller side'dan kutilgan type ham infer'ga ta'sir qiladi.

### Kod misol

```typescript
// Inference: argument'dan
function identity<T>(value: T): T { return value; }
identity(42);           // T = number (inferred)
identity("hello");      // T = string

// Multi-position — union
function pair<T>(a: T, b: T): T[] {
  return [a, b];
}
pair(1, "hello");       // T = string | number (inferred union)

// Return-only T — inference fail
function parse<T>(raw: string): T {
  return JSON.parse(raw);
}
const data = parse('{"x":1}'); // T = unknown (no inference source)
const typed = parse<{ x: number }>('{"x":1}'); // ✅ explicit

// Contextual type
const users: number[] = identity([1, 2, 3]);
// T inferred as number[] from contextual type

// Multi-position constraint
function getValue<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
getValue({ name: "Ali", age: 25 }, "name"); // T inferred, K = "name"
```

### Edge Cases

- **Excess type argument** — `identity<string, number>(42)` xato (TS expecting 1, got 2).
- **Partial type argument** — TS 4.7+ — `<T1 = number>` default berilsa, qolganini infer qilish mumkin.
- **`NoInfer<T>`** (TS 5.4) — type parameter ikki joyda ishlatilganda inference'ni bir tomondan o'chiradi.
- **Inference + literal widening** — `identity("hello")` → `T = string` (widening). `identity("hello" as const)` yoki `identity<const T>("hello")` → `T = "hello"`.

### Follow-up savollar

1. **"`NoInfer` qachon foydali?"** — `function fn<T>(value: T, fallback: NoInfer<T>)` — fallback inference uchun ishlatilmaydi, faqat value'dan T infer bo'ladi. Asymmetric API'lar uchun.
2. **"Inference algoritmi qanday?"** — TS `inferFromTypes` algoritmi: source/target type'larini juftlikda solishtiradi, candidate'larni yig'adi, oxirida `getCommonSupertype` (yoki union) bilan birlashtiradi.

</details>

---

### Savol 3: Generic constraint nima? `extends` qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic constraint — `<T extends SomeType>` bilan type parametrini cheklash. Constraint'siz T da hech qanday operation qilib bo'lmaydi (TS `T` ni unknown deb hisoblaydi).

### To'liq tushuntirish

Constraint type system'ga T qanday shape'ga ega ekanligini aytadi:

- **Object constraint** — `T extends { length: number }` — T'da `length` property bor
- **Union constraint** — `T extends string | number` — T faqat shu type'lardan biri
- **Multi-parameter constraint** — `<T, K extends keyof T>` — K T'ga bog'liq

### Kod misol

```typescript
// ❌ Constraintsiz
function getLength<T>(value: T): number {
  // return value.length; // ❌ 'length' does not exist on type 'T'
  return 0;
}

// ✅ Constraint bilan
function getLengthSafe<T extends { length: number }>(value: T): number {
  return value.length;
}

getLengthSafe("hello");       // ✅
getLengthSafe([1, 2, 3]);     // ✅
getLengthSafe({ length: 5 }); // ✅
// getLengthSafe(42);           // ❌ number da length yo'q

// Multi-parameter constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Ali", age: 25 };
getProperty(user, "name"); // string
getProperty(user, "age");  // number
// getProperty(user, "phone"); // ❌ "phone" keyof user'da yo'q

// Union constraint
function format<T extends string | number>(value: T): string {
  return String(value);
}
format("hello");  // ✅
format(42);       // ✅
// format(true);    // ❌
```

### Edge Cases

- **Constraint default ham bo'lishi mumkin** — `<T extends string = "default">`. Caller hech narsa bermasa default ishlatiladi.
- **Recursive constraint** — `<T extends Tree<T>>` — T o'zini reference qilish.
- **Distributive over constraint** — `T extends U` constraint distributive emas (faqat conditional type'da distributive).
- **`any` constraint** — `<T extends any>` va `<T>` (constraint'siz) farq qiladi: constraint'siz — T body'da `unknown` kabi (operatsiyalar cheklangan); `extends any` — T body'da `any` kabi (har operatsiya ruxsat, type safety yo'q). Default constraint TS 3.5+ — `unknown`, avval `{}` edi.

### Follow-up savollar

1. **"`extends keyof T` bilan property union'ni qanday olamiz?"** — `T[keyof T]` — barcha value type'larining union'i. `<T, K extends keyof T>` da `K` bitta key.
2. **"Constraint juda strict bo'lsa qanday?"** — Constraint juda keng (`object`) — type safety past. Juda strict (`{ length: number; charAt: ... }`) — caller kam. Optimal — kerakli minimum shape.

</details>

---

### Savol 4: `keyof` operator nima va generics bilan qanday ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`keyof T` — T object type'ning barcha key'larini string literal union sifatida oladi. Generic'da type-safe property access pattern uchun: `<T, K extends keyof T>(obj: T, key: K)`.

### To'liq tushuntirish

`keyof` ikki kontekstda:

1. **Object type** — `keyof { a: 1; b: 2 }` = `"a" | "b"`
2. **Index signature** — `keyof { [k: string]: T }` = `string | number` (JS'da `obj[0]` = `obj["0"]`)

`keyof any` = `string | number | symbol` (har JS object key type).

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type UserKeys = keyof User; // "name" | "age" | "email"

function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { name: "Ali", age: 25, email: "ali@example.com" };
const name = getProperty(user, "name");  // string
const age = getProperty(user, "age");    // number
// getProperty(user, "phone");            // ❌ "phone" UserKeys'da yo'q

// keyof index signature
type StringMap = { [k: string]: number };
type StringMapKeys = keyof StringMap; // string | number

// keyof tuple
type Tuple = [string, number, boolean];
type TupleKeys = keyof Tuple;
// number | "0" | "1" | "2" | "length" | "concat" | ... (array method'lari)

// keyof class
class Person {
  constructor(public name: string, public age: number) {}
  greet() { return `Hi ${this.name}`; }
}
type PersonKeys = keyof Person; // "name" | "age" | "greet"
```

### Edge Cases

- **`keyof T` boshqa generic'da** — `<T, K extends keyof T>` — TS infer K'ni T'ning aniq key'iga (`"name"` literal, `keyof User` emas).
- **Optional property** — `keyof { a?: 1; b: 2 }` = `"a" | "b"` (optional ham key sifatida).
- **Private/protected access** — `keyof Person` private/protected member'larni qamramaydi (faqat public).
- **`keyof never`** = `string | number | symbol` (`never` har type'ga assignable).

### Follow-up savollar

1. **"`Object.keys()` nega `(keyof T)[]` qaytarmaydi?"** — Structural typing: T'da qo'shimcha property'lar bo'lishi mumkin (`extends` bilan kelgan). `Object.keys()` runtime'da ko'p key qaytarishi mumkin → unsafe assertion.
2. **"`keyof typeof obj` qanday?"** — `typeof obj` runtime value'dan type'ni oladi, `keyof` shu type key'larini. Constant object'lardan literal union yaratish uchun.

</details>

---

### Savol 5: Index Access Types — `T[K]`, `T[keyof T]`, `T[number]` nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Index Access Type — type darajasida property type olish, JS'dagi `obj["key"]` ga o'xshash. `T[K]` — bitta property type, `T[keyof T]` — barcha value type'lari union'i, `T[number]` — array/tuple element type.

### To'liq tushuntirish

Index access type — compile-time'da property lookup. Real-world: nested API response'lardan type olish, mapped type yaratish, type-level transformation.

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
  roles: string[];
}

type NameType = User["name"];           // string
type AllValues = User[keyof User];      // string | number | string[]
type NameOrAge = User["name" | "age"];  // string | number

// T[number] — array/tuple element
const COLORS = ["red", "green", "blue"] as const;
type Color = (typeof COLORS)[number];   // "red" | "green" | "blue"

type Tuple = [string, number, boolean];
type First = Tuple[0];                  // string
type AllTuple = Tuple[number];          // string | number | boolean

// Nested access
interface ApiResponse {
  data: {
    users: { id: number; name: string }[];
    meta: { total: number };
  };
}

type UserFromApi = ApiResponse["data"]["users"][number];
// { id: number; name: string }
type MetaTotal = ApiResponse["data"]["meta"]["total"]; // number

// Generic'da
function getValue<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

### Edge Cases

- **Index doesn't exist** — `User["phone"]` xato (TS 2.1+).
- **`T[string]` index signature'da** — agar T'da `[k: string]: V` bo'lsa, `T[string]` = `V`.
- **`T[number]` non-array** — `{ name: string }[number]` xato (TS index access fail).
- **Conditional index** — `T extends { id: infer Id } ? Id : never` — index access alternative.
- **Recursive index** — `T extends (infer U)[] ? U : T[keyof T]` — recursive structures.

### Follow-up savollar

1. **"`Pick<T, K>` qanday implement qilingan?"** — `type Pick<T, K extends keyof T> = { [P in K]: T[P] }` — mapped type + index access.
2. **"`T[K]` da K union bo'lsa nima?"** — TS distributive: `User["name" | "age"]` = `User["name"] | User["age"]` = `string | number`.

</details>

---

### Savol 6: Generic class — instance instantiation va method generic farqi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic class — instance yaratilganda T aniqlanadi va barcha method'larda qotib qoladi. Method'da alohida generic — har chaqiruvda yangi U aniqlanadi (class T'ga ta'sir qilmaydi).

### To'liq tushuntirish

| | Generic class T | Method generic U |
|---|---|---|
| Aniqlanish vaqti | Instance yaratilganda | Method chaqirilganda |
| Scope | Barcha class member'lar | Faqat shu method |
| Har chaqiruvda yangi | Yo'q (qotib) | Ha |

### Kod misol

```typescript
// Generic class — T instance bo'yicha
class Box<T> {
  constructor(public value: T) {}
  getValue(): T { return this.value; }
  setValue(v: T): void { this.value = v; }
}

const box = new Box("hello");  // T = string
box.setValue("world");          // ✅
// box.setValue(42);            // ❌ T = string qotgan

const numBox = new Box(42);     // T = number (alohida instance, alohida T)

// Method generic — har chaqiruvda yangi
class Converter<T> {
  constructor(public value: T) {}

  to<U>(transform: (v: T) => U): Converter<U> {
    return new Converter(transform(this.value));
  }
}

const c = new Converter("42");                       // T = string
const c2 = c.to((s) => parseInt(s));                  // U = number, new Converter<number>
const c3 = c2.to((n) => n > 0);                       // U = boolean

// Generic class + generic method
class Repository<T> {
  private items: T[] = [];

  add(item: T): void { this.items.push(item); }

  findBy<K extends keyof T>(key: K, value: T[K]): T | undefined {
    return this.items.find((item) => item[key] === value);
  }
}

interface User { id: number; name: string; }
const repo = new Repository<User>();
repo.add({ id: 1, name: "Ali" });
repo.findBy("name", "Ali"); // K = "name", value: string
```

### Edge Cases

- **Generic class inheritance** — `class Sub<T> extends Base<T[]>` — type parameter forward qilinadi.
- **Static method generic** — static method'da class T ishlatib bo'lmaydi (faqat o'z generic'i): `class Box<T> { static create<U>(value: U): Box<U> { return new Box(value); } }`. Caller: `Box.create(42)` — `Box<number>` qaytaradi.
- **Abstract generic class** — `abstract class Repo<T> { abstract find(id: number): T; }` — abstract bilan birga.
- **Type parameter shadowing** — method'da `<T>` yozish — class T'ni shadow qiladi (xavfli, agar atayin emas bo'lsa).

### Follow-up savollar

1. **"Static method'da class T nima uchun ishlatib bo'lmaydi?"** — Static method instance'siz chaqiriladi, T aniq emas. Faqat `Box.create<number>(42)` kabi explicit beriladi.
2. **"Generic class type predicate ishlatish?"** — `function isBox<T>(x: unknown): x is Box<T>` — type guard generic'ga distribute bo'lmaydi (runtime'da T noma'lum, faqat structural check).

</details>

---

### Savol 7: Multiple type parameters — qachon va qanday ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Funksiya turli type'larni qabul qiladi yoki ularning bog'liqligini ifodalaydi. Misol: `<T, U>` mustaqil, `<T, K extends keyof T>` bog'liq.

### To'liq tushuntirish

**Mustaqil type parameter'lar** — `<T, U>`: ikki argument turli type'lar.

**Bog'liq type parameter'lar** — `<T, K extends keyof T>`: K T'ga bog'liq.

**Default + bog'liq** — `<T, U = T>`: U default qiymat T'dan oladi.

### Kod misol

```typescript
// Mustaqil
function pair<T, U>(a: T, b: U): [T, U] {
  return [a, b];
}
pair("hello", 42); // [string, number]

// Bog'liq
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// 3 parameter — Map function
function mapEntries<K extends string, V, R>(
  obj: Record<K, V>,
  fn: (key: K, value: V) => R
): Record<K, R> {
  const result = {} as Record<K, R>;
  for (const key in obj) {
    result[key] = fn(key, obj[key]);
  }
  return result;
}

const prices = { apple: 100, banana: 50 };
const formatted = mapEntries(prices, (k, v) => `${k}: ${v}$`);
// Record<"apple" | "banana", string>

// merge — ikki object
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
const user = merge({ name: "Ali" }, { age: 25 });
// { name: string } & { age: number }
```

### Edge Cases

- **Single T ikki argument** — `function merge<T>(a: T, b: T)` — TS T'ni union qilib infer qiladi yoki xato beradi (literal'lar uchun). Farqli object'lar uchun `<T, U>` afzal.
- **`<T, U extends T>`** — U T'ning sub-type'i bo'lishi shart.
- **Type parameter order** — caller side'da explicit type argument tartibi muhim. `pair<string, number>("a", 1)` — birinchi T, ikkinchi U.
- **Reusing type parameter** — `<T>(a: T, b: T) => T[]` — TS source/target'larni union qilib oladi.

### Follow-up savollar

1. **"4-5 type parameter'li funksiya — yomon practice'mi?"** — Ko'p type parameter — caller side complexity oshadi. Object parameter (named type'lar) yoki interface bilan group qilish afzal.
2. **"Type parameter va parameter order farqi?"** — Type parameter declaration order — `<T, U>`. Function parameter order — `(a: T, b: U)`. Ikkalasi mustaqil.

</details>

---

### Savol 8: Default type parameter — `<T = number>` qachon kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Default type parameter — caller type argument bermasa qaysi type ishlatilishini belgilaydi. `Promise<T = void>` kabi. Default constraint ham bo'lishi mumkin: `<T extends string = "default">`.

### To'liq tushuntirish

Default type parameter — backward compatibility va caller convenience uchun. Generic'ga yangi parameter qo'shganda eski code'larni buzmaslik uchun ham foydali.

### Kod misol

```typescript
// Default — caller default ishlatishi mumkin
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}

const a: ApiResponse = { data: null, status: 200 };           // T = unknown
const b: ApiResponse<{ name: string }> = {
  data: { name: "Ali" },
  status: 200,
};

// Default'lar boshqa parameter'ga bog'liq
interface Container<T, U = T[]> {
  value: T;
  history: U;
}

const c: Container<number> = { value: 42, history: [40, 41, 42] };
// U inferred as number[] from T

// Backward compatibility — yangi parameter qo'shish
// Avval:
// interface Form<T> { values: T; }
// Hozir:
interface Form<T, V = T> { values: T; validators: Partial<Record<keyof T, (v: V) => boolean>>; }

// Constraint + default
function loadConfig<T extends { env: string } = { env: "prod" }>(
  override?: Partial<T>
): T {
  return { env: "prod", ...override } as T;
}
const cfg = loadConfig(); // T = { env: "prod" }
```

### Edge Cases

- **Default constraint'ga mos bo'lishi shart** — `<T extends string = number>` xato (`number` `string`'ga assignable emas).
- **Default tartibi** — default'li parameter'lar default'siz parameter'lardan keyin: `<T, U = string>` OK, `<T = string, U>` xato.
- **Function generic default** — TS 2.3+ qo'llab-quvvatlanadi. Lekin function generic'da default kam ishlatiladi (inference odatda yetarli).
- **`= unknown` vs `= any`** — Default `unknown` strict type safety, `any` — har operatsiya ruxsat. Default'da `unknown` afzal.

### Follow-up savollar

1. **"Default va inference bir vaqtda?"** — Inference birinchi, default'siz parameter'lar inference'dan kelganda — argument'lardan, default'lilar inference'siz qolsa default ishlatiladi.
2. **"`unknown` default qaysi holatda foydali?"** — `Promise<T = unknown>` — caller `Promise` deb yozsa generic argument bermasdan, T = unknown bo'ladi (default'siz xato berardi).

</details>

---

### Savol 9: `const` type parameter (TS 5.0) nima? Qachon ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<const T>` — type parameter'ga literal type inference'ni majbur qiladi. Oddiy generic'da TS widening qiladi (`"hello"` → `string`), `const` bilan literal saqlanadi (`"hello"` → `"hello"`).

### To'liq tushuntirish

`as const` chaqiruv joyida yoziladi (caller eslab qolishi kerak), `<const T>` funksiya definition'da yoziladi (caller bilmasa ham ishlaydi). Library API'lar uchun afzal.

### Kod misol

```typescript
// Oddiy — widening
function define<T>(config: T): T { return config; }
const cfg1 = define({ env: "prod", port: 3000 });
// T = { env: string; port: number }

// const T — widening yo'q
function defineConst<const T>(config: T): T { return config; }
const cfg2 = defineConst({ env: "prod", port: 3000 });
// T = { readonly env: "prod"; readonly port: 3000 }

// Array — literal tuple
function createTuple<const T extends readonly unknown[]>(items: T): T {
  return items;
}
const t1 = createTuple(["a", "b", "c"]);
// T = readonly ["a", "b", "c"]

// Real-world: route definition
function createRoute<const Path extends string>(path: Path): { path: Path } {
  return { path };
}
const r = createRoute("/users/:id");
// r.path: "/users/:id" — literal saqlandi

// const T vs as const
const cfg3 = define({ env: "prod" } as const);    // T = { readonly env: "prod" }
const cfg4 = defineConst({ env: "prod" });          // bir xil natija, caller'siz
```

### Edge Cases

- **`const T` mutable type'ga mos kelmasligi** — `<const T>` readonly modifier qo'shadi. Caller mutable type kutsa, `Readonly<...>` cast kerak.
- **`const T` + `extends`** — `<const T extends string[]>` ishlaydi, lekin `<const T extends string>` kam foydali (string allaqachon literal infer bo'ladi explicit'da).
- **Object property modifier** — `<const T>` har property'ga `readonly` qo'shadi. Method shorthand'lar — `readonly` ta'sir qilmaydi (callable bo'lib qoladi).
- **Performance** — `const T` bilan TS literal type ko'p saqlaydi (memory), katta object'larda checker secondary'ga ta'sir qilishi mumkin.

### Follow-up savollar

1. **"`<const T>` qachon ishlamaydi?"** — Caller side'da type assertion (`as Foo`) bo'lsa, literal inference o'tib ketadi. Variable type explicit annotation (`const x: string = "hello"`) — `string` qoladi.
2. **"`<const T>` runtime'ga ta'sir qiladimi?"** — Yo'q, faqat type system. Object'lar runtime'da mutable qoladi, faqat TS readonly belgilaydi.

</details>

---

### Savol 10: `Object.keys()` generic bilan nima uchun muammo chiqaradi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Object.keys()` `string[]` qaytaradi, `(keyof T)[]` emas. Sabab — structural typing: `T` da qo'shimcha property'lar bo'lishi mumkin (subtype'larda), `Object.keys()` runtime'da ko'proq key qaytarishi mumkin → unsafe.

### To'liq tushuntirish

TS structural type system'ida `T extends User` degani T'da User'ning barcha property'lari bor, **lekin qo'shimcha ham bo'lishi mumkin**. `Object.keys()` runtime'da haqiqiy property'larni qaytaradi, shu sababli `(keyof T)[]` deb belgilash unsafe.

### Kod misol

```typescript
interface User { name: string; age: number; }

function logUser<T extends User>(user: T): void {
  Object.keys(user).forEach((key) => {
    // key: string — keyof T emas
    // console.log(user[key]); // ❌ Element implicitly has an 'any' type
  });
}

// Misol — qo'shimcha property
interface Admin extends User { role: string; }
const admin: Admin = { name: "Ali", age: 25, role: "admin" };
logUser(admin); // Object.keys = ["name", "age", "role"]
// keyof User = "name" | "age" — role yo'q! unsafe

// Yechim 1: type assertion (caller javobgar)
function logUserSafe<T extends User>(user: T): void {
  (Object.keys(user) as Array<keyof T>).forEach((key) => {
    console.log(user[key]);
  });
}

// Yechim 2: aniq key list
const KNOWN_KEYS = ["name", "age"] as const;
KNOWN_KEYS.forEach((key) => console.log(admin[key]));

// Yechim 3: for...in (lekin type bir xil — string)
for (const key in admin) {
  // key: string
}
```

### Edge Cases

- **`Object.entries()`** — `[string, any][]` qaytaradi (worse than keys).
- **`Object.fromEntries()`** — type information yo'qoladi. TS 4.0+ overload qo'shilgan, lekin literal tuple kerak.
- **`for...in` loop** — JS spec'ga ko'ra enumerable property'lar, inheritance ham qamraydi. Type ham `string`.
- **`Record<K, V>` bilan ishlash** — `Record<"a" | "b", number>` da `Object.keys()` `(keyof typeof obj)[]` deb cast qilish nisbatan xavfsiz (literal record'da extra property'lar yo'q).

### Follow-up savollar

1. **"`as const` object bilan `Object.keys()` qanday?"** — `const obj = { a: 1, b: 2 } as const` — `Object.keys(obj)` hali `string[]`. Cast yoki utility function kerak.
2. **"Helper utility — type-safe keys?"** — `function keys<T extends object>(obj: T): (keyof T)[] { return Object.keys(obj) as (keyof T)[]; }` — caller responsibility'ni encapsulate qiladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 11: Output — Generic wrap inference [Middle]

**Savol:** Output va TS type'larini ayting:

```typescript
function wrap<T>(value: T): { data: T; type: string } {
  return { data: value, type: typeof value };
}

const a = wrap(42);
const b = wrap("hello");
const c = wrap([1, 2, 3]);

console.log(a.type);
console.log(b.data.toUpperCase());
console.log(c.data.length);
console.log(typeof a.data);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
number
HELLO
3
number
```

### To'liq tushuntirish

- `wrap(42)` — T = `number` → `{ data: 42, type: "number" }`. `a.type` = `"number"` (string runtime).
- `wrap("hello")` — T = `string` → `b.data.toUpperCase()` = `"HELLO"`.
- `wrap([1, 2, 3])` — T = `number[]` → `c.data.length` = `3`. Lekin `c.type` = `"object"` (JS'da `typeof []` = `"object"`).
- `typeof a.data` runtime — `"number"`.

TS type'lari va runtime `typeof` farq qilishi mumkin (`number[]` TS — `"object"` runtime).

### Edge Cases

- `typeof null` = `"object"` JS bug — backward compatibility.
- `typeof undefined` = `"undefined"`.
- `wrap(null)` — T = `null` (literal), `a.type` runtime = `"object"`.

### Follow-up savollar

1. **"TS T va runtime typeof bir xil bo'lishini qanday majbur qilamiz?"** — Discriminated union return: `{ data: number; type: "number" } | { data: string; type: "string" } | ...`. Conditional return bilan: `T extends number ? { data: T; type: "number" } : ...`.

</details>

---

### Savol 12: Bug — `merge` bitta T parameter [Middle]

**Savol:** Bu kodda nima xato? Qanday tuzatiladi?

```typescript
function merge<T>(a: T, b: T): T {
  return { ...a, ...b };
}

merge({ name: "Ali" }, { age: 25 });
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bitta T ikki parameter'da — ikkalasi bir xil shape bo'lishi kerak. Birinchi argument'dan T = `{ name: string }` infer, ikkinchida `name` yo'q → xato.

### To'liq tushuntirish

```typescript
// ❌ T = { name: string } — { age: 25 } mos emas
merge({ name: "Ali" }, { age: 25 });

// ✅ Yechim 1: ikki alohida type parameter
function merge<T, U>(a: T, b: U): T & U {
  return { ...a, ...b };
}
merge({ name: "Ali" }, { age: 25 });
// { name: string } & { age: number }

// ✅ Yechim 2: Partial<T> (a strict, b qisman)
function merge2<T extends object>(a: T, b: Partial<T>): T {
  return { ...a, ...b };
}
```

### Edge Cases

- **TS ba'zan T'ni union qilib infer qiladi** — `merge({ name: "Ali" }, { age: 25 })` ba'zi versions'da `{ name: string } | { age: number }` infer qilishi mumkin (TS evolution).
- **Spread va `&` farqi** — `{ ...a, ...b }` runtime'da o'zgaruvchini birlashtiradi (oxirgi yutadi). `T & U` type system'da union.
- **Key collision** — agar T va U'da bir xil key bor bo'lsa, intersection type bir xil property bo'lsa OK, har xil bo'lsa `never`.

### Follow-up savollar

1. **"Type parameter bittasidan ikki argument'da infer qilingani — TS'ning xatosi?"** — Yo'q, intentional. `<T>(a: T, b: T)` API'da ikkalasi bir xil type bo'lishini ifodalaydi (`Array.prototype.includes(value: T)`).
2. **"Bir xil T va `<T, U extends T>` farqi?"** — `<T>` ikkalasini union qiladi yoki xato beradi. `<T, U extends T>` U strictly T'ning sub-type.

</details>

---

### Savol 13: Output — Generic widening + `<const T>` [Middle+]

**Savol:** Har o'zgaruvchining TS type'ini ayting:

```typescript
function identity<T>(value: T): T { return value; }
function identityConst<const T>(value: T): T { return value; }

const a = identity("hello");
const b = identity<"hello">("hello");
const c = identity(42 as const);
const d = identityConst("hello");
const e = identityConst({ env: "prod" });

type A = typeof a;
type B = typeof b;
type C = typeof c;
type D = typeof d;
type E = typeof e;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type A = string;                            // widening: "hello" → string
type B = "hello";                           // explicit literal
type C = 42;                                // as const literal
type D = "hello";                           // <const T> literal
type E = { readonly env: "prod" };          // <const T> readonly + literal
```

### To'liq tushuntirish

- `identity("hello")` — argument string literal, T `string` infer (widening default)
- `identity<"hello">("hello")` — explicit literal type
- `identity(42 as const)` — `as const` chaqiruvda literal
- `identityConst("hello")` — `<const T>` widening yo'q, literal saqlanadi
- `identityConst({ env: "prod" })` — har property `readonly` + literal

### Edge Cases

- **`<const T>` array** — `identityConst([1, 2, 3])` → `readonly [1, 2, 3]` (tuple literal).
- **`<const T>` + variable annotation** — `const x: string = identityConst("hello")` — `string` annotation override qiladi, lekin function side'da T = `"hello"` infer.
- **`as const` + `<const T>` birga** — natija bir xil (literal). Ortiqcha.

### Follow-up savollar

1. **"Widening qachon kerak (`as const` ishlatmaslik)?"** — Mutable konfiguratsiya, dynamic values, runtime'da o'zgaradigan state. `as const` immutable kontekst.

</details>

---

### Savol 14: Coding — Type-safe `pick` [Middle+]

**Savol:** `pick(obj, keys)` — object'dan faqat berilgan key'larni olib yangi object qaytaradi. Key'lar compile-time'da tekshirilsin:

```typescript
interface User {
  name: string;
  age: number;
  email: string;
  role: string;
}

// pick(user, ["name", "email"]) → { name: string; email: string }
// pick(user, ["phone"]) → ❌ compile error
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach((key) => {
    result[key] = obj[key];
  });
  return result;
}
```

### To'liq tushuntirish

- `K extends keyof T` — faqat T'ning key'lari qabul qilinadi
- `Pick<T, K>` built-in utility = `{ [P in K]: T[P] }` mapped type
- Return type aniq — faqat tanlangan key'lar

```typescript
const user: User = {
  name: "Ali",
  age: 25,
  email: "ali@example.com",
  role: "admin",
};

const subset = pick(user, ["name", "email"]);
// subset: { name: string; email: string }

console.log(subset);
// subset.age;            // ❌ 'age' Pick'da yo'q
// pick(user, ["phone"]);  // ❌ "phone" keyof User'da yo'q
```

### Edge Cases

- **Duplicate keys** — `pick(user, ["name", "name"])` — TS xato bermaydi, runtime'da ham OK (oxirgi yutadi).
- **Empty keys array** — `pick(user, [])` — `Pick<User, never>` = `{}` (bo'sh object).
- **Optional property pick** — `Pick<T, K>` optional modifier'ni saqlaydi (`Pick<{ x?: number }, "x">` = `{ x?: number }`).
- **Nested key pick** — bu funksiya faqat top-level. Nested uchun path-based generic: `pick(user, "address.city")` template literal types kerak.

### Follow-up savollar

1. **"`omit` qanday implement qilinadi?"** — `Omit<T, K> = Pick<T, Exclude<keyof T, K>>`. Implementation'da `Object.keys` orqali filter va cast.
2. **"`pick` runtime'da unknown key bilan?"** — TS compile-time'da bloklaydi. Runtime'da unknown property — `undefined` qaytadi (JS behavior).

</details>

---

### Savol 15: Bug — `firstElement` empty array [Senior]

**Savol:** Bu kodda compile-time xato yo'q, lekin runtime crash. Nima uchun? Qanday tuzatiladi?

```typescript
function firstElement<T>(arr: T[]): T {
  return arr[0];
}

const result = firstElement([]);
console.log(result.toString());
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`firstElement([])` — T = `never` (bo'sh array `never[]` infer). `arr[0]` runtime'da `undefined`. Return type `T` = `never` — har type'ga assignable (bottom type), TS xato bermaydi. Runtime'da `undefined.toString()` crash.

### To'liq tushuntirish

```typescript
// ❌ TS xato bermaydi
const result = firstElement([]); // type: never
result.toString(); // TS: OK, runtime: TypeError

// ✅ Tuzatish: T | undefined
function firstElementSafe<T>(arr: T[]): T | undefined {
  return arr[0];
}

const result2 = firstElementSafe([]); // type: undefined
// result2.toString(); // ❌ Object is possibly 'undefined'

if (result2 !== undefined) {
  result2.toString(); // ✅
}

// `noUncheckedIndexedAccess` flag (tsconfig)
// arr[0] avtomatik T | undefined qaytaradi
```

### Edge Cases

- **`noUncheckedIndexedAccess: true`** — array index access avtomatik `T | undefined`. Strict safety, lekin kod ko'p narrowing talab qiladi.
- **Tuple bilan farq** — `function first<T extends unknown[]>(arr: [...T]): T[0]` — tuple shape'ni saqlaydi. Bo'sh tuple xato beradi.
- **`never` bottom type** — TS'da har type'ning sub-type'i. `never` qiymat yaratib bo'lmaydi (faqat `throw`, infinite loop, exhaustiveness).
- **`T[0]` tuple type — tuple'ning birinchi element** — `[]` ga `T[0]` `undefined` qaytaradi (`noUncheckedIndexedAccess` mustaqil). Bu non-empty tuple constraint uchun yaxshi pattern.

### Follow-up savollar

1. **"Tuple bilan empty array xato qaytadimi?"** — `function first<T extends readonly [unknown, ...unknown[]]>(arr: T): T[0]` — non-empty constraint. `first([])` — xato (empty `T`'ga assignable emas).
2. **"`noUncheckedIndexedAccess` flag'ini production'da ishlatish kerakmi?"** — Yangi loyihalarda majburiy tavsiya, eski loyihalar uchun migration ko'p qator narrowing talab qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**`never` semantics:**

- Bottom type — har type'ning sub-type
- `never` value mavjud emas — faqat `throw`, `while(true)`, recursive `never` qaytaruvchi function
- `never` har joyga assignable (`const x: string = throwError()`)
- Hech narsa `never`'ga assignable emas (`unknown`'dan tashqari `never extends unknown`)

**Bottom type'ning xavfli unsoundness:**

- TS strict mode'da ham `never.toString()` xato bermaydi
- Sabab — `never` "this can't happen" promise, lekin compiler runtime'ni tekshirmaydi
- `firstElement([])` — empty array uchun "T = never" intentional (no element type to infer), lekin function body'da hali ham `arr[0]` ishlaydi va `undefined` qaytaradi

**`noUncheckedIndexedAccess` flag'ining implementation:**

- Array/tuple/record access'da TS avtomatik `T | undefined` qaytaradi
- Performance ta'siri yo'q (faqat type system)
- Migration cost — runtime null check'lar ko'p qator qo'shilishini majbur qiladi

</details>

</details>

---

### Savol 16: Coding — Type-safe `groupBy` [Senior]

**Savol:** `groupBy(items, keyFn)` — array'ni `keyFn` natijasi bo'yicha guruhlar. Return type aniq bo'lsin (literal key'lar saqlanadi):

```typescript
const users = [
  { id: 1, role: "admin" as const, name: "Ali" },
  { id: 2, role: "user" as const, name: "Vali" },
  { id: 3, role: "admin" as const, name: "Salim" },
];

// groupBy(users, u => u.role) → { admin: User[]; user: User[] }
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
function groupBy<T, K extends PropertyKey>(
  items: readonly T[],
  keyFn: (item: T) => K
): Record<K, T[]> {
  const result = {} as Record<K, T[]>;
  for (const item of items) {
    const key = keyFn(item);
    (result[key] ??= []).push(item);
  }
  return result;
}
```

### To'liq tushuntirish

- `K extends PropertyKey` — `string | number | symbol` (har object key turi)
- `Record<K, T[]>` — har K key uchun `T[]` qiymat
- Caller `keyFn` literal qaytarsa, K literal union infer bo'ladi
- `??=` (TS 4.0+, ES2021) — lazy initialization

```typescript
const users = [
  { id: 1, role: "admin" as const, name: "Ali" },
  { id: 2, role: "user" as const, name: "Vali" },
];

const grouped = groupBy(users, (u) => u.role);
// Type: Record<"admin" | "user", { id: number; role: "admin" | "user"; name: string }[]>

grouped.admin;    // ✅ array
grouped.user;     // ✅ array
// grouped.guest; // ❌ "guest" Record key'da yo'q
```

### Edge Cases

- **`keyFn` `string` qaytarsa (non-literal)** — K = `string`, return `Record<string, T[]>` — har string key, type safety past.
- **Empty array** — `groupBy([], fn)` — return `{}` (bo'sh Record). `Record<K, T[]>` da K hech qachon ishlatilmagan.
- **`keyFn` `undefined`/`null` qaytarsa** — runtime'da `result[undefined]` = `"undefined"` string key. TS bunga ruxsat bermaydi (`PropertyKey` constraint).
- **`<const K>` (TS 5.0+)** — caller `as const` yozmasa ham literal saqlash uchun.

### Follow-up savollar

1. **"Numeric key bilan ishlash?"** — `keyFn: (item: T) => number` — K extends number, `Record<number, T[]>`. JS'da `obj[1]` = `obj["1"]` (string coercion). TS aniq tekshiradi.
2. **"`Map` ishlatish afzalmi?"** — Ha, `Map<K, T[]>` — key type strict (string coercion yo'q), insertion order saqlanadi, `Map<object, T[]>` ham mumkin. Lekin JSON serialization murakkab.

<details>
<summary><strong>Deep Dive</strong></summary>

**`PropertyKey` type:**

TS lib'da `type PropertyKey = string | number | symbol` — har JS object key turi. Generic constraint sifatida — caller'ga keng ruxsat, lekin K narrowing literal'lar uchun.

**Record vs Map type-level:**

`Record<K, V>` — `{ [P in K]: V }` (mapped type). K literal union bo'lsa, har key required. `Map<K, V>` — class, runtime structure, key types JS strict (no string coercion).

**Inference + `as const`:**

Caller `as const` yozmasa, `u.role` type `string` infer bo'ladi (literal widening). `as const` array literal'larini `readonly` qiladi va string'larni literal saqlaydi. Alternative — `keyFn` parametr'ga `<const K>` (TS 5.0+) qo'shish:

```typescript
function groupByConst<T, const K extends PropertyKey>(
  items: readonly T[],
  keyFn: (item: T) => K
): Record<K, T[]> { /* ... */ }
```

`<const K>` — caller hech narsa o'zgartirmasdan literal K infer bo'ladi.

**Lodash comparison:**

Lodash `_.groupBy` — runtime'da bir xil, lekin TS type'da `Dictionary<T[]>` qaytaradi (har string key uchun T[], literal narrowing yo'q). Modern TS implementation strict typing beradi.

</details>

</details>

---

## Xulosa

- Generics — type-safe abstraction, `any` type'ni saqlaydi
- Inference — argument'dan T infer, return-only T uchun explicit kerak
- Constraint (`extends`) — T'ga operatsiya qilish uchun
- `keyof` + `T[K]` — type-safe property access pattern
- Multiple parameter — mustaqil yoki bog'liq, `<T, K extends keyof T>` keng tarqalgan
- Default type parameter — backward compatibility, caller convenience
- `<const T>` (TS 5.0) — literal type saqlaydi, library API'lar uchun
- Generic class T instance bo'yicha, method generic har chaqiruvda yangi
- `Object.keys()` `string[]` — structural typing sababi, type assertion kerak

---