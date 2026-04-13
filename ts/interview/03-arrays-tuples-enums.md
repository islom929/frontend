# Interview: Arrays, Tuples va Enums

> Array types, readonly arrays, tuples, enum under the hood, const enum, enum alternativalari bo'yicha interview savollari.

---

## Nazariy savollar

### 1. TypeScript da array tipi qanday yoziladi? `string[]` va `Array<string>` farqi bormi?

<details>
<summary>Javob</summary>

Ikki xil sintaksis — ikkalasi **bir xil**:

```typescript
let names: string[] = ["Ali", "Vali"];
let names2: Array<string> = ["Ali", "Vali"];
```

Farq yo'q — ikkalasi `Array<T>` interface ga resolve bo'ladi. Convention bo'yicha `string[]` ko'proq ishlatiladi. Lekin murakkab tiplar uchun `Array<T>` ba'zan aniqroq:

```typescript
// Union array — bracket da qavslar kerak
let mixed: (string | number)[] = [1, "hello"];

// Generic da qavslar kerak emas
let mixed2: Array<string | number> = [1, "hello"];

// Funksiya array — Generic aniqroq
let handlers: Array<(e: Event) => void> = [];
let handlers2: ((e: Event) => void)[] = []; // qavslar chalkash
```

</details>

### 2. Readonly array nima? Qachon ishlatish kerak?

<details>
<summary>Javob</summary>

`readonly` array — elementlarni o'zgartirish, qo'shish yoki o'chirish mumkin bo'lmagan array. Compile-time himoya — runtime da ta'siri yo'q.

```typescript
const items: readonly string[] = ["a", "b", "c"];

items.push("d");    // ❌ Property 'push' does not exist
items[0] = "x";     // ❌ Index signature only permits reading
items.sort();       // ❌ sort mutates

items.map(x => x.toUpperCase()); // ✅ yangi array qaytaradi
items.filter(x => x !== "b");    // ✅ yangi array qaytaradi
```

Qachon ishlatish kerak:

1. **Funksiya parametrlari** — funksiya array ni o'zgartirmasligini kafolatlash
2. **Config/constant data** — o'zgarmasligi kerak bo'lgan ma'lumotlar

```typescript
// ✅ Best practice — parametrda readonly
function getTotal(prices: readonly number[]): number {
  return prices.reduce((sum, p) => sum + p, 0);
}

// Mutable → readonly ga berish mumkin
const prices: number[] = [10, 20, 30];
getTotal(prices); // ✅ number[] → readonly number[]

// Teskari — readonly → mutable mumkin EMAS
const frozen: readonly number[] = [1, 2, 3];
const arr: number[] = frozen; // ❌
```

</details>

### 3. Tuple nima? Array dan farqi nima?

<details>
<summary>Javob</summary>

Tuple — **aniq uzunlikdagi** array bo'lib, har bir elementning **alohida tipi** bor. JavaScript da tuple yo'q — faqat TypeScript type system da mavjud.

```typescript
let user: [string, number] = ["Ali", 25];
user[0]; // string
user[1]; // number
// user[2]; // ❌ Tuple has no element at index '2'
```

| Xususiyat | Array | Tuple |
|-----------|-------|-------|
| Uzunlik | O'zgaruvchan | Fixed |
| Element tipi | Barcha bir xil | Har biri alohida |
| `length` tipi | `number` | Aniq son (2, 3...) |

**Muhim pitfall:** Tuple da `push()` ishlaydi — TS to'xtatmaydi:

```typescript
let pair: [string, number] = ["hello", 42];
pair.push("extra"); // ⚠️ Xato bermaydi!

// ✅ Yechim — readonly tuple
let pair2: readonly [string, number] = ["hello", 42];
// pair2.push("extra"); // ❌ push yo'q
```

</details>

### 4. Enum ning xavfli tomonlari qaysilar?

<details>
<summary>Javob</summary>

**1. Numeric enum ga har qanday son assign mumkin:**

```typescript
enum Direction { Up, Down, Left, Right }
let dir: Direction = 99; // ⚠️ Xato bermaydi!
```

**2. Reverse mapping chalkashlik:**

```typescript
enum Color { Red, Green, Blue }
Object.keys(Color);
// ["0", "1", "2", "Red", "Green", "Blue"] — 6 ta key!
```

**3. const enum import muammosi:**

```typescript
// isolatedModules: true rejimda
export const enum Theme { Light = "LIGHT", Dark = "DARK" }
import { Theme } from "./types"; // ❌ Cannot access ambient const enums
```

**4. String enum ga string assign mumkin emas:**

```typescript
enum Status { Active = "ACTIVE" }
const s: Status = "ACTIVE"; // ❌ Enum faqat member orqali
```

**Yechim:** Union literal type yoki `as const` object ishlatish xavfsizroq:

```typescript
type Direction = "UP" | "DOWN" | "LEFT" | "RIGHT";
// ✅ Numeric pitfalls yo'q, bundle size 0, isolatedModules muammo yo'q
```

</details>

### 5. `noUncheckedIndexedAccess` nima va nima uchun yoqish kerak?

<details>
<summary>Javob</summary>

Array va object index access natijasiga `| undefined` qo'shadigan tsconfig flag. `strict: true` ga **kirmaydi** — alohida yoqish kerak.

```typescript
// noUncheckedIndexedAccess: false (default)
const names: string[] = ["Ali", "Vali"];
const tenth = names[99]; // type: string — lekin aslida undefined!
tenth.toUpperCase();      // 💥 Runtime crash

// noUncheckedIndexedAccess: true
const tenth2 = names[99]; // type: string | undefined
if (tenth2 !== undefined) {
  tenth2.toUpperCase(); // ✅ narrowing dan keyin xavfsiz
}
```

Kamchiligi — ba'zan ortiqcha tekshiruv kerak:

```typescript
for (let i = 0; i < names.length; i++) {
  const name = names[i]; // string | undefined — lekin biz bilamiz mavjud
  name!.toUpperCase();    // ! kerak
}
```

</details>

### 6. Enum vs union literal vs `as const` — qachon qaysi biri?

<details>
<summary>Javob</summary>

| Kriteriya | Enum | Union | as const |
|-----------|------|-------|----------|
| Runtime da mavjud | ✅ IIFE object | ❌ O'chiriladi | ✅ Oddiy object |
| Bundle size | Katta | 0 | Kichik |
| Tree shaking | Qiyin | Muammo yo'q | Yaxshi |
| `isolatedModules` | ⚠️ const enum ishlamaydi | ✅ | ✅ |
| `erasableSyntaxOnly` | ❌ | ✅ | ✅ |
| Iterate mumkin | ⚠️ Reverse mapping | ❌ | ✅ |
| String literal assign | ❌ | ✅ | ✅ |
| Numeric bit flags | ✅ | Qiyin | Qiyin |

**Tavsiya:**

- **Union type** — ko'p holat uchun eng yaxshi (`type Status = "active" | "inactive"`)
- **`as const` object** — runtime da qiymatlar kerak bo'lganda
- **Enum** ��� faqat legacy code, Angular/NestJS convention, yoki bit flags

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Array va Tuple tiplari — output quiz (Daraja: Junior+)

**Savol:** Har bir tipning `length` tipi nima? `C` ga `push()` ishlaydi mi? `D` ga-chi?

```typescript
type A = string[];
type B = readonly string[];
type C = [string, number];
type D = readonly [string, number];
type E = [string, ...number[]];
type F = [string, number?];
```

<details>
<summary>Yechim</summary>

```typescript
type A = string[];
// Mutable string array. push, pop, sort ishlaydi
// A["length"] = number (o'zgaruvchan)

type B = readonly string[];
// Immutable string array. push, pop, sort YO'Q
// B["length"] = number

type C = [string, number];
// Mutable tuple — aniq 2 element. C[0] = string, C[1] = number
// C["length"] = 2
// push() ishlaydi! ⚠️ TS to'xtatmaydi — bu xavfli

type D = readonly [string, number];
// Immutable tuple — push YO'Q, to'liq himoyalangan
// D["length"] = 2

type E = [string, ...number[]];
// Rest element li tuple — 1+ element
// ["hello"] ✅, ["hello", 1, 2, 3] ✅
// E["length"] = number (o'zgaruvchan)

type F = [string, number?];
// Optional tuple — 1 yoki 2 element
// ["hello"] ✅, ["hello", 42] ✅
// F["length"] = 1 | 2
```

**Kalit tushuncha:** Mutable tuple da `push()` ishlaydi — bu TS ning ma'lum cheklovi. Xavfsizlik uchun `readonly` tuple ishlatish kerak.

</details>

### 2. Tuple widening — xatoni toping (Daraja: Middle)

**Savol:** Bu kodda compile-time xato bor. Xatoni toping va 3 xil usulda tuzating:

```typescript
function processCoordinates(coords: [number, number]): void {
  console.log(`x: ${coords[0]}, y: ${coords[1]}`);
}

const point = [10, 20];
processCoordinates(point);
```

<details>
<summary>Yechim</summary>

**Xato:** `point` ning tipi `number[]` — tuple `[number, number]` emas. `const` bilan e'lon qilinsa ham array elementlari widened bo'ladi. `number[]` uzunligi noma'lum, shuning uchun `[number, number]` ga assign mumkin emas.

```typescript
// ❌ Type 'number[]' is not assignable to type '[number, number]'
const point = [10, 20];        // number[]
processCoordinates(point);      // ❌

// ✅ Yechim 1: aniq tuple annotation
const point1: [number, number] = [10, 20];
processCoordinates(point1);     // ✅

// ✅ Yechim 2: as const + readonly parametr
const point2 = [10, 20] as const; // readonly [10, 20]
function processCoordinates2(coords: readonly [number, number]): void {
  console.log(`x: ${coords[0]}, y: ${coords[1]}`);
}
processCoordinates2(point2);    // ✅

// ✅ Yechim 3: inline
processCoordinates([10, 20]);   // ✅ Inline literal → tuple deb aniqlaydi
```

**Tushuntirish:** TypeScript array literal ni variable ga assign qilganda `T[]` ga widen qiladi. Tuple sifatida saqlash uchun aniq annotation, `as const`, yoki inline berish kerak.

</details>

### 3. Enum compile output — custom values (Daraja: Middle+)

**Savol:** Bu enum JavaScript ga qanday compile bo'ladi? Har bir member ning qiymatini ayting:

```typescript
enum Status {
  Active,
  Inactive = 5,
  Pending,
}

console.log(Status.Active);    // ?
console.log(Status.Inactive);  // ?
console.log(Status.Pending);   // ?
console.log(Status[5]);        // ?
```

<details>
<summary>Yechim</summary>

```javascript
// Compiled JavaScript:
var Status;
(function (Status) {
  Status[Status["Active"] = 0] = "Active";
  Status[Status["Inactive"] = 5] = "Inactive";
  Status[Status["Pending"] = 6] = "Pending";
})(Status || (Status = {}));
```

```
Status.Active   → 0
Status.Inactive → 5
Status.Pending  → 6  (5 + 1, oldingi qiymatdan davom etadi)
Status[5]       → "Inactive" (reverse mapping)
```

**Pattern tushuntirish:**

```javascript
Status[Status["Active"] = 0] = "Active";
// 1-qadam: Status["Active"] = 0 → Active: 0 qo'shildi
//          assignment 0 ni qaytaradi
// 2-qadam: Status[0] = "Active" → reverse mapping qo'shildi
```

**Muhim:** `Pending` qiymati 6 — TypeScript oldingi member qiymatidan (`5`) `+1` qiladi. Agar `Inactive = 10` bo'lsa, `Pending = 11` bo'ladi.

</details>

### 4. Enum dan `as const` ga refactoring (Daraja: Middle+)

**Savol:** Bu enum asosidagi kodni `as const` object ga o'tkazing. Type safety va funksionallik saqlansin:

```typescript
enum HttpMethod {
  GET = "GET",
  POST = "POST",
  PUT = "PUT",
  DELETE = "DELETE",
  PATCH = "PATCH",
}

function makeRequest(url: string, method: HttpMethod): void {
  fetch(url, { method });
}

function isValidMethod(value: string): boolean {
  return Object.values(HttpMethod).includes(value as HttpMethod);
}

makeRequest("/api/users", HttpMethod.GET);
```

<details>
<summary>Yechim</summary>

```typescript
const HTTP_METHOD = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
  DELETE: "DELETE",
  PATCH: "PATCH",
} as const;

type HttpMethod = typeof HTTP_METHOD[keyof typeof HTTP_METHOD];
// "GET" | "POST" | "PUT" | "DELETE" | "PATCH"

function makeRequest(url: string, method: HttpMethod): void {
  fetch(url, { method });
}

// ✅ Type guard bilan — return type yaxshilandi
function isValidMethod(value: string): value is HttpMethod {
  return (Object.values(HTTP_METHOD) as string[]).includes(value);
}

// Ishlatish:
makeRequest("/api/users", HTTP_METHOD.GET); // ✅ object orqali
makeRequest("/api/users", "GET");            // ✅ string literal ham ishlaydi

const userInput = "GET";
if (isValidMethod(userInput)) {
  makeRequest("/api", userInput); // ✅ narrowed to HttpMethod
}
```

**Yaxshilanishlar:**

1. **Bundle size** — IIFE o'rniga oddiy object
2. **String literal** — `"GET"` to'g'ridan-to'g'ri berish mumkin
3. **Type guard** — `value is HttpMethod` bilan narrowing
4. **`isolatedModules`/`erasableSyntaxOnly`** — muammosiz

</details>

### 5. Type-safe config validation (Daraja: Senior)

**Savol:** `as const` va generics ishlatib, type-safe configuration validator yozing. Validator faqat mavjud kalitlarni qabul qilsin va noto'g'ri qiymat tipini compile-time da ushlasin:

```typescript
const DEFAULTS = {
  theme: "light",
  fontSize: 14,
  darkMode: false,
  language: "en",
} as const;

// validateConfig funksiyasini yozing:
// validateConfig({ theme: "dark" })         → ✅
// validateConfig({ fontSize: 18 })          → ✅
// validateConfig({ theme: 123 })            → ❌ number !== string
// validateConfig({ invalid: "x" })          → ❌ kalit yo'q
```

<details>
<summary>Yechim</summary>

```typescript
const DEFAULTS = {
  theme: "light",
  fontSize: 14,
  darkMode: false,
  language: "en",
} as const;

type Config = typeof DEFAULTS;
type ConfigKey = keyof Config;

// Widened type map — literal dan base type ga
type Widen<T> =
  T extends string ? string :
  T extends number ? number :
  T extends boolean ? boolean :
  T;

type WidenedConfig = { [K in ConfigKey]: Widen<Config[K]> };

function validateConfig(
  overrides: Partial<WidenedConfig>
): WidenedConfig {
  return { ...DEFAULTS, ...overrides };
}

// ✅ Ishlaydi
validateConfig({ theme: "dark" });          // string ✅
validateConfig({ fontSize: 18 });           // number ✅
validateConfig({ darkMode: true });         // boolean ✅

// ❌ Compile error
// validateConfig({ theme: 123 });          // number !== string
// validateConfig({ invalid: "x" });        // kalit yo'q
// validateConfig({ fontSize: "big" });     // string !== number
```

**Tushuntirish:**

- `as const` — barcha qiymatlar literal (`"light"`, `14`, `false`)
- `Widen<T>` — literal tipni base tipga kengaytiradi (`"light"` → `string`)
- `Partial<WidenedConfig>` — barcha kalitlar optional, lekin tipi to'g'ri bo'lishi shart
- Agar `Widen` ishlatmasak, faqat `"light"` qabul qilinadi, `"dark"` xato beradi

</details>

---

## Xulosa

- `string[]` va `Array<string>` — bir xil, convention bo'yicha `T[]` afzal
- `readonly` array — funksiya parametrlarida doim ishlatish tavsiya etiladi
- Tuple — fixed-length array, lekin `push()` ishlaydi — `readonly` tuple xavfsizroq
- Enum xavflari — numeric assign, reverse mapping chalkashlik, `isolatedModules` muammo
- `noUncheckedIndexedAccess` — `strict` ga kirmaydi, alohida yoqish kerak
- Zamonaviy TS da union type yoki `as const` — enum ga qaraganda afzal

[Asosiy bo'limga qaytish →](../03-arrays-tuples-enums.md)
