# Interview: Generics

> Type parameters, generic functions, interfaces, classes, constraints, keyof, index access types, const type parameters va generic utility patterns bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Generics nima va nima uchun kerak? `any` dan farqi nima?

<details>
<summary>Javob</summary>

Generics — type-level abstraction. Funksiya, interface yoki class da aniq type o'rniga **type parameter** (`<T>`) qo'yiladi. U faqat ishlatilgan paytda aniq type ga aylanadi.

```typescript
// ❌ any — type safety yo'q
function firstAny(arr: any[]): any {
  return arr[0];
}
const a = firstAny([1, 2, 3]);
a.toUpperCase(); // TS xato bermaydi, lekin runtime crash

// ✅ Generic — to'liq type safety
function firstGeneric<T>(arr: T[]): T | undefined {
  return arr[0];
}
const b = firstGeneric([1, 2, 3]); // number | undefined
// b.toUpperCase(); // ❌ Compile-time error
```

| | `any` | Generic `<T>` |
|---|---|---|
| Type safety | ❌ | ✅ |
| Inference | Hamma narsa `any` | Aniq type qaytaradi |
| IDE support | ❌ | ✅ To'liq IntelliSense |

Generic ikki muammoni hal qiladi: **type safety yo'qolishi** va **kod takrorlanishi**.

</details>

### 2. Generic constraint nima? `extends` qanday ishlaydi?

<details>
<summary>Javob</summary>

Generic constraint — `<T extends SomeType>` bilan type parameter ni cheklash. Constraintsiz T ga hech qanday operation qilib bo'lmaydi.

```typescript
// ❌ Constraintsiz
function getLength<T>(value: T): number {
  // return value.length; // ❌ 'length' does not exist on type 'T'
}

// ✅ Constraint bilan
function getLength<T extends { length: number }>(value: T): number {
  return value.length; // ✅
}

getLength("hello");   // ✅ string.length bor
getLength([1, 2, 3]); // ✅ array.length bor
// getLength(42);      // ❌ number da length yo'q
```

Constraint lar type parameter lar orasida **bog'liqlik** yaratish uchun ham ishlatiladi:

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]; // K — T ning key laridan biri bo'lishi KERAK
}
```

</details>

### 3. `keyof` nima va generics bilan qanday ishlatiladi?

<details>
<summary>Javob</summary>

`keyof` — object type ning barcha key larini **string literal union** sifatida oladi. Generics bilan birga type-safe property access yaratadi.

```typescript
interface User { name: string; age: number; email: string; }

type UserKeys = keyof User; // "name" | "age" | "email"

function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { name: "Ali", age: 25, email: "ali@test.com" };
const name = getProperty(user, "name");  // string
const age = getProperty(user, "age");    // number
// getProperty(user, "phone");            // ❌ "phone" keyof User da yo'q
```

`keyof` turli type lar bilan:

```typescript
keyof { a: 1; b: 2 }         // "a" | "b"
keyof { [k: string]: number } // string | number (index signature)
keyof any                     // string | number | symbol
```

`keyof` va index signature: `{ [k: string]: T }` da `keyof` = `string | number` — JS da `obj[0]` va `obj["0"]` bir xil.

</details>

### 4. Index Access Types — `T[K]`, `T[keyof T]`, `T[number]` nima?

<details>
<summary>Javob</summary>

Index Access Type — type darajasida property type olish. JS dagi `obj["key"]` ga o'xshash, lekin compile-time da.

```typescript
interface User { name: string; age: number; roles: string[]; }

type NameType = User["name"];         // string
type UserValues = User[keyof User];   // string | number | string[]
type NameOrAge = User["name" | "age"]; // string | number
```

`T[number]` — array/tuple element type:

```typescript
const COLORS = ["red", "green", "blue"] as const;
type Color = (typeof COLORS)[number]; // "red" | "green" | "blue"

type Tuple = [string, number, boolean];
type First = Tuple[0];    // string
type All = Tuple[number]; // string | number | boolean
```

Real-world — nested type olish:

```typescript
interface ApiResponse {
  data: { users: { id: number; name: string }[]; };
}
type UserFromApi = ApiResponse["data"]["users"][number];
// { id: number; name: string }
```

</details>

### 5. Generic class va generic function farqi nima?

<details>
<summary>Javob</summary>

Asosiy farq — **type parameter scope** va **instantiation** vaqti:

```typescript
// Generic function — har bir CHAQIRUVDA T aniqlanadi
function identity<T>(value: T): T { return value; }
identity("hello"); // T = string
identity(42);      // T = number — yangi instantiation

// Generic class — YARATILGAN paytda T aniqlanadi
class Box<T> {
  constructor(public value: T) {}
  getValue(): T { return this.value; }
  setValue(v: T): void { this.value = v; }
}
const box = new Box("hello"); // T = string — barcha method larda string
// box.setValue(42); // ❌ T = string qotib qoldi
```

| | Generic Function | Generic Class |
|---|---|---|
| T aniqlanish vaqti | Har bir chaqiruvda | Instance yaratilganda |
| T scope | Faqat shu funksiya | Barcha member lar |
| Har safar yangi T | ✅ | ❌ qotib qoladi |

Class ichida alohida method ning o'z type parameter i ham bo'lishi mumkin:

```typescript
class Converter<T> {
  constructor(public value: T) {}
  to<U>(transform: (v: T) => U): Converter<U> {
    return new Converter(transform(this.value));
  }
}
```

</details>

### 6. `const` type parameter (TS 5.0) nima? Qachon ishlatish kerak?

<details>
<summary>Javob</summary>

`<const T>` — type parameter ga **literal type inference** ni majbur qiladi. Oddiy generic da TS widening qiladi (`"hello"` → `string`), `const` bilan literal saqlanadi.

```typescript
// Oddiy — widening bo'ladi
function define<T>(config: T): T { return config; }
const cfg1 = define({ env: "prod", port: 3000 });
// T = { env: string; port: number }

// const T — widening bo'lmaydi
function defineConst<const T>(config: T): T { return config; }
const cfg2 = defineConst({ env: "prod", port: 3000 });
// T = { readonly env: "prod"; readonly port: 3000 }
```

`as const` dan farqi: `as const` chaqiruv joyida yoziladi (foydalanuvchi eslab qolishi kerak), `<const T>` funksiya definition da yoziladi (foydalanuvchi bilmasa ham ishlaydi). Library yozayotganda `<const T>` afzal.

</details>

### 7. `Object.keys()` generic bilan nima uchun muammo chiqaradi?

<details>
<summary>Javob</summary>

`Object.keys()` har doim `string[]` qaytaradi — `(keyof T)[]` **emas**. Sabab — structural typing:

```typescript
interface User { name: string; age: number; }

function logUser<T extends User>(user: T): void {
  Object.keys(user).forEach((key) => {
    // key: string — keyof T emas!
    // console.log(user[key]); // ❌ implicitly 'any'
  });
}
```

Nima uchun? `T extends User` degani T da **qo'shimcha property lar ham bo'lishi mumkin**:

```typescript
interface Admin extends User { role: string; }
const admin: Admin = { name: "Ali", age: 25, role: "admin" };
logUser(admin); // Object.keys = ["name", "age", "role"]
// keyof User = "name" | "age" — role yo'q! Shuning uchun xavfsiz emas
```

Yechimlar:

```typescript
// 1. Type assertion
(Object.keys(user) as Array<keyof T>).forEach(key => console.log(user[key]));

// 2. Aniq key lar
const keys: (keyof User)[] = ["name", "age"];
keys.forEach(key => console.log(user[key]));
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Generic wrap — output savoli (Daraja: Middle)

**Savol:** Output va TS type larini ayting:

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
<summary>Yechim</summary>

```
number
HELLO
3
number
```

- `wrap(42)` → T = number → `{ data: 42, type: "number" }` — `a.type` = `"number"`
- `wrap("hello")` → T = string → `b.data.toUpperCase()` = `"HELLO"`
- `wrap([1, 2, 3])` → T = number[] → `c.data.length` = `3` (lekin `c.type` = `"object"` — JS da `typeof []` = `"object"`)
- `typeof a.data` — runtime JS `typeof` → `"number"`

TS type lari: `a.data: number`, `b.data: string`, `c.data: number[]`. Runtime `typeof` va TS type farq qilishi mumkin (`number[]` vs `"object"`).

</details>

### 2. merge bitta T — xatoni toping (Daraja: Middle)

**Savol:** Bu kodda nima xato? Qanday tuzatiladi?

```typescript
function merge<T>(a: T, b: T): T {
  return { ...a, ...b };
}

merge({ name: "Ali" }, { age: 25 });
```

<details>
<summary>Yechim</summary>

**Xato:** Ikkinchi argument `{ age: 25 }` birinchi bilan **bir xil type** bo'lishi kerak (ikkalasi T). Birinchi dan T = `{ name: string }` infer bo'ladi — ikkinchida `name` yo'q.

```typescript
// ❌ T = { name: string } — { age: 25 } mos emas
merge({ name: "Ali" }, { age: 25 }); // Error

// ✅ Yechim 1: Ikki alohida type parameter
function merge<T, U>(a: T, b: U): T & U {
  return { ...a, ...b };
}
merge({ name: "Ali" }, { age: 25 }); // ✅ { name: string } & { age: number }

// ✅ Yechim 2: Object constraint bilan Partial
function merge2<T extends object>(a: T, b: Partial<T>): T {
  return { ...a, ...b };
}
```

**Dars:** Bitta `T` ikki parametrda ishlatilsa — ikkalasi **bir xil type** bo'lishi kerak. Farqli type lar kutilsa — `<T, U>` ishlatish kerak.

</details>

### 3. identity widening — output savoli (Daraja: Middle+)

**Savol:** Har bir o'zgaruvchining TS type ini ayting:

```typescript
function identity<T>(value: T): T {
  return value;
}

const a = identity("hello");
const b = identity<"hello">("hello");
const c = identity(42 as const);

type A = typeof a; // ?
type B = typeof b; // ?
type C = typeof c; // ?
```

<details>
<summary>Yechim</summary>

```typescript
type A = string;   // "hello" → string ga widening
type B = "hello";  // Explicit literal type berildi
type C = 42;       // as const — literal saqlanadi
```

- `identity("hello")` — TS `T = string` deb infer qiladi (widening)
- `identity<"hello">("hello")` — explicit `T = "hello"` — literal saqlanadi
- `identity(42 as const)` — argument type `42` literal — `T = 42`

TS 5.0+ da `<const T>` bilan widening ni to'xtatish mumkin:

```typescript
function identityConst<const T>(value: T): T { return value; }
const d = identityConst("hello");
type D = typeof d; // "hello" — <const T> literal saqlab qoladi
```

</details>

### 4. Type-safe `pick` funksiyasi (Daraja: Middle+)

**Savol:** `pick(obj, keys)` — object dan faqat berilgan key larni olib yangi object qaytaradi. Key lar compile-time da tekshirilsin:

```typescript
interface User {
  name: string;
  age: number;
  email: string;
  role: string;
}

// pick funksiyasini yozing:
// pick(user, ["name", "email"]) → { name: string; email: string }
// pick(user, ["phone"])          → ❌ compile error
```

<details>
<summary>Yechim</summary>

```typescript
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach((key) => {
    result[key] = obj[key];
  });
  return result;
}

const user: User = { name: "Ali", age: 25, email: "ali@test.com", role: "admin" };

const subset = pick(user, ["name", "email"]);
// subset: Pick<User, "name" | "email"> = { name: string; email: string }

console.log(subset);     // { name: "Ali", email: "ali@test.com" }
// subset.age;            // ❌ 'age' Pick da yo'q
// pick(user, ["phone"]); // ❌ "phone" keyof User da yo'q
```

**Tushuntirish:**

- `K extends keyof T` — faqat T ning key lari qabul qilinadi
- `Pick<T, K>` built-in utility — `{ [P in K]: T[P] }` mapped type
- Return type **aniq** — faqat tanlangan key lar bor
- TS inference: `pick(user, ["name", "email"])` → `K = "name" | "email"`

</details>

### 5. firstElement empty array — gotcha (Daraja: Senior)

**Savol:** Bu kodda compile-time xato yo'q. Lekin runtime da crash bo'ladi. Nima uchun? Qanday tuzatiladi?

```typescript
function firstElement<T>(arr: T[]): T {
  return arr[0];
}

const result = firstElement([]);
console.log(result.toString());
```

<details>
<summary>Yechim</summary>

**Muammo:** `firstElement([])` — T = `never` (bo'sh array `never[]` infer). `arr[0]` runtime da `undefined` qaytaradi. Lekin return type `T` = `never` — TS xato bermaydi chunki `never` har qanday type ga assignable (bottom type).

`undefined.toString()` — **runtime TypeError**.

```typescript
// ❌ TS xato bermaydi — never.toString() valid
const result = firstElement([]); // type: never
result.toString(); // TS: ✅, Runtime: 💥 TypeError

// ✅ Tuzatish: return type T | undefined
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const result = firstElement([]); // type: undefined
// result.toString(); // ❌ Compile error: possibly undefined

if (result !== undefined) {
  result.toString(); // ✅ xavfsiz
}
```

**Dars:** Array dan element olayotganda har doim `T | undefined` qaytaring. `never` bottom type sifatida TS ning "unsound" joylaridan biri — compile-time da xato bermaydi, lekin runtime da crash bo'lishi mumkin.

</details>

---

## Xulosa

- Generics — type-safe abstraction, `any` dan farqli ravishda type ni saqlaydi
- Constraint (`extends`) — T ga operatsiya qilish uchun kerak, constraintsiz T unknown dek
- `keyof` + `T[K]` — type-safe property access pattern
- Generic class — T instance yaratilganda qotib qoladi, function da har chaqiruvda yangi
- `<const T>` (TS 5.0) — literal type saqlab qoladi, library API lar uchun foydali
- `Object.keys()` `string[]` qaytaradi — structural typing sababi
- Generics compile da o'chiriladi (type erasure) — runtime da generic yo'q
- Covariance/Contravariance — batafsil [interview/25 — Type Compatibility](25-type-compatibility.md) da
- Result\<T,E\> pattern — batafsil [interview/21 — Design Patterns](21-design-patterns.md) da

[Asosiy bo'limga qaytish →](../08-generics.md)
