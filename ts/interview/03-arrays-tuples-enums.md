# Interview: Arrays, Tuples va Enums

> Array types, readonly arrays, tuples (optional/rest/named elements), enum under the hood, const enum, enum alternativalari, satisfies bilan validation bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Output prediction savollari](#output-prediction-savollari)
- [Coding challenges](#coding-challenges)
- [Bug fix savollari](#bug-fix-savollari)

---

## Nazariy savollar

### Savol 1: TypeScript da array tipi qanday yoziladi? `string[]` va `Array<string>` farqi bormi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ikki sintaksis bir xil — ikkalasi `Array<T>` interface ga resolve bo'ladi. `string[]` qisqaroq, convention bo'yicha ko'proq ishlatiladi. Murakkab type lar uchun `Array<T>` aniqroq bo'lishi mumkin.

### To'liq tushuntirish

`T[]` va `Array<T>` semantically identical:

- `string[]` qisqa va o'qish oson
- `Array<string>` generic syntax, murakkab type lar uchun aniqroq

Convention:
- Simple type — `T[]` (`string[]`, `number[]`)
- Union type — qavslar kerak `(string | number)[]` yoki `Array<string | number>`
- Function type — qavslar chalkash, `Array<(e: Event) => void>` aniqroq

### Kod misol

```typescript
let names: string[] = ["Ali", "Vali"];
let names2: Array<string> = ["Ali", "Vali"];
// Ikkalasi bir xil type

// Union array
let mixed1: (string | number)[] = [1, "hello"];
let mixed2: Array<string | number> = [1, "hello"];

// Function array — Generic aniqroq
let handlers1: ((e: Event) => void)[] = [];  // qavslar chalkash
let handlers2: Array<(e: Event) => void> = []; // aniqroq

// Nested array
let matrix: number[][] = [[1, 2], [3, 4]];
let matrix2: Array<Array<number>> = [[1, 2], [3, 4]];
```

### Edge Cases

- `Array<T>` generic constraint da ishlatish kerak: `function f<T extends Array<unknown>>(arr: T)` — `T[]` bu yerda parse qilinmaydi
- `string[]` ASI (automatic semicolon insertion) muammosi: `array[0]` keyingi qatorda bo'lsa, parser chalkashishi mumkin
- Empty array literal — `const arr = []` `never[]` deb inference olinishi mumkin (const + strict); odatda `any[]` (let). Annotation kerak

### Follow-up savollar

1. **"`ReadonlyArray<T>` va `readonly T[]` bir xilmi?"** — Ha, semantically. `readonly T[]` qisqa, `ReadonlyArray<T>` generic syntax.

</details>

---

### Savol 2: `readonly` array nima va qachon ishlatish kerak? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`readonly` array — element larni o'zgartirish, qo'shish yoki o'chirish mumkin bo'lmagan array. Compile-time himoya (runtime ta'siri yo'q). Funksiya parametrlarida — best practice (immutability kafolati).

### To'liq tushuntirish

`readonly` array qoidalari:
- Mutating method lar (`push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`) — yo'q
- Index assignment yo'q (`arr[0] = "x"`)
- Read-only method lar (`map`, `filter`, `slice`, `concat`, `find`, `forEach`) — ishlaydi
- Variance: `T[]` `readonly T[]` ga assignable (covariant), teskari yo'q

Use case lar:

1. **Function parameter** — funksiya array ni o'zgartirmasligini kafolatlash
2. **Config / constant** — o'zgarmasligi kerak bo'lgan ma'lumot
3. **Return type** — caller mutate qila olmasligi uchun

### Kod misol

```typescript
const items: readonly string[] = ["a", "b", "c"];

items.push("d");    // ❌ Property 'push' does not exist
items[0] = "x";     // ❌ Index signature only permits reading
items.sort();       // ❌ sort mutates

items.map(x => x.toUpperCase()); // ✅ yangi array qaytaradi
items.filter(x => x !== "b");    // ✅ yangi array qaytaradi
const sorted = [...items].sort(); // ✅ copy + sort

// ✅ Best practice — parametrda readonly
function getTotal(prices: readonly number[]): number {
  return prices.reduce((sum, p) => sum + p, 0);
}

const prices: number[] = [10, 20, 30];
getTotal(prices); // ✅ number[] → readonly number[]

// ❌ Teskari yo'q
const frozen: readonly number[] = [1, 2, 3];
const arr: number[] = frozen; // ❌ readonly array is not assignable to mutable array
```

### Edge Cases

- `Object.freeze(arr)` runtime immutability beradi, lekin TypeScript bilan integratsiyalanmagan
- `readonly T[]` faqat top-level — element object property lar mutate bo'lishi mumkin
- `as const` array `readonly` tuple yaratadi (literal element lar)
- Variance qoidasi: `function f(arr: number[]): void` `readonly number[]` parameter qabul qila olmaydi — chunki funksiya mutate qilishi mumkin

### Follow-up savollar

1. **"`readonly` recursive bo'ladimi?"** — Yo'q. `readonly { x: number[] }` — `x` reference o'zgarmaydi, lekin `x` ichidagi array mutate bo'lishi mumkin. Recursive uchun `DeepReadonly<T>` utility yozish kerak.
2. **"`readonly` runtime'da ta'sir qiladimi?"** — Yo'q, faqat compile-time. Runtime immutability uchun `Object.freeze` yoki `Immutable.js`/`immer` library.

</details>

---

### Savol 3: Tuple nima va array dan farqi qaysi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Tuple — **aniq uzunlikdagi** array, har element ning **alohida tipi** bor (`[string, number]`). JavaScript da tuple yo'q — faqat TypeScript type system da. Array — bir xil tipli, o'zgaruvchan uzunlikdagi.

### To'liq tushuntirish

| Xususiyat | Array | Tuple |
|-----------|-------|-------|
| Uzunlik | O'zgaruvchan | Fixed (yoki rest bilan kengayuvchi) |
| Element tipi | Barcha bir xil | Har biri alohida |
| `length` type | `number` | Aniq son (yoki union, rest bilan) |
| `push` ishlatish | ✅ | ⚠️ Ishlaydi (TS to'xtatmaydi — mutable tuple pitfall) |
| Index access | `T \| undefined` (noUncheckedIndexedAccess) | Aniq element tipi |

Tuple use case:
- Function multiple return value: `[user, error]` pattern
- Coordinate, pair: `[x, y]`
- React `useState`: `[value, setter]`
- Map.entries: `[key, value][]`

### Kod misol

```typescript
let user: [string, number] = ["Ali", 25];

user[0];  // string
user[1];  // number
// user[2]; // ❌ Tuple has no element at index '2'
user.length; // 2 (aniq literal)

// React useState — tuple return
const [count, setCount] = useState(0);
// count: number, setCount: Dispatch<SetStateAction<number>>

// Multiple return value pattern
function parseInput(input: string): [boolean, string | null] {
  if (input.length === 0) return [false, null];
  return [true, input.trim()];
}

const [success, value] = parseInput("hello");

// ⚠️ Pitfall: mutable tuple da push ishlaydi
let pair: [string, number] = ["hello", 42];
pair.push("extra"); // ⚠️ TS xato bermaydi (TS issue #28508)
console.log(pair); // ["hello", 42, "extra"]

// ✅ Yechim: readonly tuple
let pair2: readonly [string, number] = ["hello", 42];
// pair2.push("extra"); // ❌ push yo'q
```

### Edge Cases

- Tuple push xavfsizlik muammosi — TypeScript known limitation. Readonly tuple ishlatish best practice
- Tuple destructuring — `[a, b, c] = tuple` ishlaydi, lekin `c` aslida `undefined` bo'lishi mumkin
- Tuple spread — `[...tuple, extra]` to'g'ri tuple yaratadi
- `as const` bilan: `["a", "b"] as const` → `readonly ["a", "b"]`

### Follow-up savollar

1. **"Tuple va object — qaysi biri afzal?"** — Object — field name bilan aniq (`{ name: string, age: number }`). Tuple — qisqa, destructuring oson, lekin order ahamiyatli va position bilan ma'no berish kerak.
2. **"`as const` har joyda ishlatish kerakmi?"** — Tuple ga aylanadi va literal type beradi — config va return value lar uchun foydali. Lekin har joyda kerak emas — performance ta'siri yo'q, lekin code noise oshadi.

</details>

---

### Savol 4: Optional tuple elements (`?`) qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Optional tuple elements — `?` belgisi bilan element ixtiyoriy bo'lishini ko'rsatadi (`[string, number?]`). Optional element lar tuple oxirida bo'lishi shart. Length type — union (`1 | 2` masalan). TS 3.0 da kiritilgan.

### To'liq tushuntirish

Qoidalari:

1. Optional element faqat **tuple oxirida** bo'lishi mumkin
2. Optional dan keyin required element bo'la olmaydi
3. Length — union literal (`[string, number?]` — `length: 1 | 2`)
4. Optional element type i: `T | undefined`

Use case lar:
- Function default parameter modeling
- Configuration tuple

### Kod misol

```typescript
type Coords = [number, number, number?];

const point2D: Coords = [10, 20];         // ✅
const point3D: Coords = [10, 20, 30];     // ✅
// const invalid: Coords = [10];          // ❌ Source has 1 element(s) but target requires 2

point2D[2]; // number | undefined
point2D.length; // 2 | 3

// Function pattern
function createUser(name: string, age: number, email?: string): User {
  return { name, age, email: email ?? "" };
}

type UserTuple = [name: string, age: number, email?: string];
const u1: UserTuple = ["Ali", 25];           // ✅
const u2: UserTuple = ["Ali", 25, "a@t.com"]; // ✅
```

### Edge Cases

- Optional element undefined qiymat bilan berilsa: `[10, 20, undefined]` — ✅
- Length union — `if (tuple.length === 3)` narrowing: TS tuple ni `[number, number, number]` deb biladi
- `?` faqat tuple element uchun — array elementlarda yo'q
- Rest element optional bilan birga: `[string, number?, ...boolean[]]` — optional avval, rest oxirida

### Follow-up savollar

1. **"Optional tuple element vs object optional property — qaysi biri afzal?"** — Object optional property aniqroq (field name bor). Tuple optional qachon order mantiqiy bo'lganda (`[x, y, z?]` koordinata).

</details>

---

### Savol 5: Rest elements in tuples qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Rest element — `...T[]` syntax bilan tuple ichida noma'lum sondagi element larni ifoda qiladi. TS 3.0 da end-position, TS 4.0 da har joyda (variadic tuple). Use case: variadic function, head/tail pattern.

### To'liq tushuntirish

TS 3.0 — rest faqat tuple oxirida:
```typescript
type T = [string, ...number[]]; // first string, then numbers
```

TS 4.0 — variadic tuple, rest har joyda:
```typescript
type T1 = [string, ...number[], boolean]; // first string, last boolean, middle numbers
type T2 = [...number[], string]; // last string
type T3<T extends unknown[]> = [...T, string]; // generic
```

Use case lar:
- Function signature: `function f(...args: [string, ...number[]])`
- React component event handler: `(event: Event, ...args: unknown[])`
- Generic type manipulation: `[Head, ...Tail<T>]`

### Kod misol

```typescript
// Rest oxirida
type StringNumbers = [string, ...number[]];
const a: StringNumbers = ["hello"];                   // ✅ (no rest)
const b: StringNumbers = ["hello", 1, 2, 3];          // ✅

// Variadic tuple (TS 4.0+)
type Endcap = [string, ...number[], boolean];
const e: Endcap = ["start", 1, 2, 3, true];           // ✅
const e2: Endcap = ["start", true];                    // ✅

// Generic head/tail
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T extends unknown[]> = T extends [unknown, ...infer R] ? R : never;

type H = Head<[1, 2, 3]>;  // 1
type T = Tail<[1, 2, 3]>;  // [2, 3]

// Function variadic args
function logFirst(...args: [string, ...number[]]): void {
  const [msg, ...nums] = args;
  console.log(msg, nums);
}

logFirst("count:", 1, 2, 3); // ✅
// logFirst(1, 2, 3); // ❌ first must be string
```

### Edge Cases

- Bir tuple da bitta rest element bo'lishi mumkin: `[...A, ...B]` ❌
- Rest element type — `T[]` (array), `T` emas
- Optional element rest bilan: `[string, number?, ...boolean[]]` — optional avval, rest oxirida
- Variadic tuple generic constraint — `T extends unknown[]` kerak

### Follow-up savollar

1. **"Variadic tuple qaysi advanced pattern lar uchun ishlatiladi?"** — Function composition (`pipe`, `compose`) type-safe implement qilish, `curry` function, Promise.all argument inference.

</details>

---

### Savol 6: Named (labeled) tuple elements nima va nima uchun foydali? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Named tuple elements (TS 4.0+) — tuple element lariga nom berish (`[name: string, age: number]`). Faqat IDE hint va documentation uchun — runtime ta'siri yo'q (type erasure). Function rest parameter labelling uchun foydali.

### To'liq tushuntirish

Sintaksis: `[label: Type]`. Label faqat documentation va IDE hint — type system uchun ahamiyatsiz.

Foydalari:
- IDE da `useState` return value uchun aniqroq hint: `[count: number, setCount: (n: number) => void]`
- Function signature labeled rest parameter
- Tuple destructuring aniqroq

Cheklov:
- Hammasi labeled bo'lishi shart yoki hech biri (mixed yo'q)
- Optional element label bilan: `[name: string, age?: number]`

### Kod misol

```typescript
// Labeled tuple
type UserTuple = [name: string, age: number, email?: string];

// IDE da destructuring vaqtida `name`, `age`, `email` hint chiqadi
const [name, age, email] = user;

// Function rest parameter labeling
function setRange(...range: [start: number, end: number]): void {
  console.log(`Range: ${range[0]} - ${range[1]}`);
}
setRange(0, 100);

// React useState — labeled return helpful
function useCounter(): [count: number, increment: () => void] {
  const [count, setCount] = useState(0);
  return [count, () => setCount(c => c + 1)];
}

// Generic with labeled tuple
type Event<T> = [type: string, payload: T];
const click: Event<{ x: number; y: number }> = ["click", { x: 10, y: 20 }];
```

### Edge Cases

- Label mixing taqiqlangan: `[string, age: number]` ❌
- Label runtime'da yo'q — `tuple[0]` ishlatish kerak, `tuple.name` ❌
- Optional element bilan: `[name: string, age?: number]` ✅
- Rest element bilan: `[first: string, ...rest: number[]]` ✅

### Follow-up savollar

1. **"Label tuple object dan farqi nimada?"** — Object — runtime'da haqiqiy property. Tuple — array sifatida (index access). Object name'siz buzilmaydi, tuple position bilan ishlaydi.

</details>

---

### Savol 7: Enum (numeric va string) qanday ishlaydi va JavaScript ga qanday compile bo'ladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Numeric enum — auto-increment integer (`Up = 0, Down = 1, ...`). String enum — aniq string value. Ikkalasi ham IIFE ga compile bo'ladi. Numeric enum reverse mapping ham hosil qiladi (qiymat → nom). String enum reverse mapping yo'q.

### To'liq tushuntirish

**Numeric enum**:

```typescript
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right, // 3
}
```

Auto-increment: birinchi member 0, keyingilari +1. Custom start: `Up = 10` → `Down = 11`.

Compile output:

```javascript
var Direction;
(function (Direction) {
    Direction[Direction["Up"] = 0] = "Up";
    Direction[Direction["Down"] = 1] = "Down";
    Direction[Direction["Left"] = 2] = "Left";
    Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));
```

Pattern: `Direction[Direction["Up"] = 0] = "Up"`:
1. `Direction["Up"] = 0` — forward mapping (`Up: 0`)
2. Assignment `0` qaytaradi
3. `Direction[0] = "Up"` — reverse mapping

**String enum**:

```typescript
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}
```

Compile output:

```javascript
var Status;
(function (Status) {
    Status["Active"] = "ACTIVE";
    Status["Inactive"] = "INACTIVE";
})(Status || (Status = {}));
// Reverse mapping YO'Q
```

### Kod misol

```typescript
// Numeric enum
enum Color { Red, Green, Blue }
console.log(Color.Red);    // 0
console.log(Color[0]);     // "Red" (reverse mapping)

// String enum
enum Method { GET = "GET", POST = "POST" }
console.log(Method.GET);   // "GET"
// Method["GET"] → "GET"
// Method["X"] → undefined (no reverse)

// Custom values
enum Status {
  Active = 1,
  Inactive = 5,
  Pending,   // auto-increment: 6
}

// Const initializer (computed)
enum Mixed {
  A = 1,
  B = 2 * A, // 2 (compile-time computed)
  // C = compute(), // ❌ if runtime function
}

// Object.keys behavior — numeric enum
Object.keys(Color); // ["0", "1", "2", "Red", "Green", "Blue"] (6 keys!)
Object.values(Method); // ["GET", "POST"] (string enum — clean)
```

### Edge Cases

- Numeric enum `Object.keys` ham reverse mapping ni qaytaradi — iterate qilish chalkash
- String enum reverse yo'q — `Method["X"]` `undefined`
- Mixed enum (numeric + string) — taqiqlanmagan, lekin chalkash
- `const enum` qiymat inline bo'ladi, IIFE yo'q
- TS 5.0+ pure literal enum union ga aylangan — `let d: Direction = 99` ❌ (avval ruxsat etilardi)

### Follow-up savollar

1. **"Enum membership runtime'da qanday tekshirish kerak?"** — Numeric uchun: `value in Color` (reverse mapping). String uchun: `Object.values(Status).includes(value)`.
2. **"Enum vs `as const` object — qaysi biri afzal?"** — Zamonaviy TypeScript da `as const` object — bundle size kichik, tree-shaking yaxshi, `isolatedModules`/`erasableSyntaxOnly` muammosiz. Enum faqat legacy yoki bit flags uchun.

<details>
<summary><strong>Deep Dive</strong></summary>

Numeric enum reverse mapping JS native object'da emas — alohida `value → key` mapping faylga compile bo'ladi. Bu pattern history: TypeScript birinchi versiyalarida enum C# style ishlashini xohladi, lekin JavaScript da plain object dan boshqa ifoda yo'q edi.

`const enum` Babel/esbuild bilan muammoli — chunki per-file compilation tools butun loyihani ko'ra olmaydi. TS 4.x da `--isolatedModules` flag `const enum` ni export qilishni cheklaydi.

TS 5.0 da pure literal enum union narrowing yaxshilandi — numeric enum `Color` literal union `0 | 1 | 2` deb qaraladi, type safety oshdi.

`enum` `erasableSyntaxOnly: true` (TS 5.8) bilan taqiqlanadi — Node.js native TypeScript bilan ishlamaydi.

</details>

</details>

---

### Savol 8: `const enum` nima va qachon ishlatish (yoki ishlatmaslik) kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`const enum` — qiymatlar compile time da inline bo'ladi, runtime object yaratilmaydi. Bundle size kichik, lekin `isolatedModules` bilan re-export muammosi va Babel/esbuild moslik muammosi bor. Library yozsangiz — `const enum` taqiqlangan.

### To'liq tushuntirish

`const enum` ta'sirlari:
- Runtime'da object yo'q — qiymatlar har joyda inline
- `Color[0]` (dynamic access) ishlamaydi — chunki object yo'q
- Bundle size 0
- TS native (per-file) `--isolatedModules` rejimida cheklov

Cheklovlar:
- Re-export `isolatedModules` bilan: `export const enum` taqiqlangan
- Babel/esbuild: butun loyihani ko'rmaydi, inline qilolmaydi
- Library: import qiluvchi paket boshqa `tsc` versiyada bo'lishi mumkin → broken inline

### Kod misol

`const enum` compile output:

```typescript
const enum Direction {
  Up = "UP",
  Down = "DOWN",
}

const dir = Direction.Up;
console.log(Direction.Down);

// Compile output (inline)
const dir = "UP";
console.log("DOWN");
// Direction object hosil bo'lmaydi
```

Re-export muammosi:

```typescript
// utils.ts
export const enum Theme { Light = "LIGHT", Dark = "DARK" }

// app.ts (isolatedModules: true)
import { Theme } from "./utils";
// ❌ Cannot access ambient const enums when 'isolatedModules' is enabled
```

`as const` object alternativasi:

```typescript
// const enum o'rniga
export const Theme = {
  Light: "LIGHT",
  Dark: "DARK",
} as const;

export type Theme = typeof Theme[keyof typeof Theme];

// Ishlatish
import { Theme } from "./utils";
const t: Theme = Theme.Light; // "LIGHT"
```

### Edge Cases

- `preserveConstEnums: true` flag — `const enum` ni oddiy `enum` kabi compile qiladi (runtime object hosil bo'ladi). Library yozish uchun yechim
- `const enum` faqat numeric/string literal qiymatlar qabul qiladi — `compute()` taqiqlangan
- TS 5.0+ `const enum` semantic o'zgarmagan, lekin `--isolatedModules` qattiqlashdi

### Follow-up savollar

1. **"Library yozaman, `const enum` ishlatish mumkinmi?"** — Yo'q, juda xavfli. Consumer turli tools (Babel, esbuild, swc) ishlatadi — inline ishlamaydi. `as const` object yoki oddiy `enum` afzal.

</details>

---

### Savol 9: Enum xavfli tomonlari qaysilar va alternativalari nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Asosiy xavflar: (1) numeric enum old "intentional unsoundness" (TS <5.0), (2) reverse mapping `Object.keys` chalkashlik, (3) `const enum` cross-package muammo, (4) `isolatedModules`/`erasableSyntaxOnly` mos kelmaslik, (5) tree-shaking qiyin. Alternativalar: union literal type, `as const` object.

### To'liq tushuntirish

**Xavfli holatlar:**

1. **Numeric enum unsoundness (TS < 5.0)** — `let d: Direction = 99` ruxsat etilardi (TS 5.0+ tuzatilgan)
2. **Computed member bilan** — `enum Mixed { A = 1, B = compute() }` — hali ham xavfli, `let m: Mixed = 999` ruxsat
3. **Reverse mapping iteration** — `Object.keys(Color)` 6 ta key qaytaradi (3 forward + 3 reverse)
4. **`const enum` cross-package** — Babel/esbuild bilan ishlamaydi
5. **`isolatedModules`** — `const enum` re-export taqiqlangan
6. **`erasableSyntaxOnly`** (TS 5.8) — enum butunlay taqiqlangan
7. **String enum literal assign** — `enum Status { A = "ACTIVE" }`, `const s: Status = "ACTIVE"` ❌
8. **Tree-shaking** — bundler IIFE ni dead code deb tushunmasligi mumkin

**Alternativalar:**

```typescript
// Variant 1: Union literal type
type Direction = "UP" | "DOWN" | "LEFT" | "RIGHT";

// Variant 2: as const object
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;
type Direction = typeof Direction[keyof typeof Direction];
```

### Kod misol

Enum issues:

```typescript
// 1. Reverse mapping iteration
enum Color { Red, Green, Blue }
console.log(Object.keys(Color));
// ["0", "1", "2", "Red", "Green", "Blue"] — 6 keys!

// 2. String enum strict assign
enum Status { Active = "ACTIVE" }
const s: Status = "ACTIVE"; // ❌ Type '"ACTIVE"' is not assignable

// 3. const enum + isolatedModules
// (different file)
export const enum Theme { Light = "LIGHT" }
// import { Theme } from "./theme"; // ❌ with isolatedModules

// 4. Bit flags pattern (legitimate enum use)
enum Permission {
  None = 0,
  Read = 1 << 0,  // 1
  Write = 1 << 1, // 2
  Delete = 1 << 2,// 4
  All = Read | Write | Delete, // 7
}
const userPerms = Permission.Read | Permission.Write; // 3
const canRead = (userPerms & Permission.Read) !== 0;  // true
```

`as const` object yechim (har holat uchun):

```typescript
const HTTP_METHOD = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
  DELETE: "DELETE",
} as const;

type HttpMethod = typeof HTTP_METHOD[keyof typeof HTTP_METHOD];
// "GET" | "POST" | "PUT" | "DELETE"

function makeRequest(method: HttpMethod): void {
  console.log(method);
}

makeRequest(HTTP_METHOD.GET);  // ✅
makeRequest("POST");            // ✅ string literal ham ishlaydi
// makeRequest("INVALID");      // ❌

// Iterate clean
Object.values(HTTP_METHOD);     // ["GET", "POST", "PUT", "DELETE"]
Object.keys(HTTP_METHOD);       // ["GET", "POST", "PUT", "DELETE"]
```

### Edge Cases

- Bit flags pattern uchun enum hali ham qulayroq (`Read | Write` numeric OR)
- Angular, NestJS legacy convention sifatida enum ishlatishadi — migration qiyin
- `const enum` `preserveConstEnums: true` bilan oddiy enum kabi compile bo'ladi
- TS 5.0+ pure literal union narrowing — numeric enum xavfsizroq bo'ldi

### Follow-up savollar

1. **"Qachon enum ni saqlash mantiqli?"** — (1) Bit flags (numeric OR), (2) Legacy Angular/NestJS codebase, (3) `as const` syntax cheklovlari (masalan namespace style organization).
2. **"`Object.values(asConstObject)` qanday tip qaytaradi?"** — `("GET" | "POST" | "PUT" | "DELETE")[]` — literal union array.

</details>

---

### Savol 10: `noUncheckedIndexedAccess` nima va nima uchun yoqish kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`noUncheckedIndexedAccess` — array va object index access natijasiga `| undefined` qo'shadi. `strict: true` ga **kirmaydi** — alohida yoqish kerak. Index out-of-bounds runtime crash larini oldini oladi.

### To'liq tushuntirish

Default xulq (`noUncheckedIndexedAccess: false`) — TypeScript optimistic:
- `arr[i]` type `T` (har doim mavjud deb hisoblaydi)
- Aslida `i >= arr.length` bo'lsa `undefined` qaytadi → runtime crash

`noUncheckedIndexedAccess: true` — TypeScript pessimistic:
- `arr[i]` type `T | undefined`
- Index access dan keyin narrowing kerak

Trade-off:
- Xavfsizroq (runtime crash kamayadi)
- Ko'proq narrowing kodi (`if (item !== undefined)`)
- For-of loop bilan yengilroq (`for (const item of arr)` — `item: T`)

### Kod misol

`noUncheckedIndexedAccess: false`:

```typescript
const names: string[] = ["Ali", "Vali"];
const tenth = names[99];
// tenth: string (lekin aslida undefined!)

tenth.toUpperCase(); // 💥 Runtime: Cannot read property 'toUpperCase' of undefined
```

`noUncheckedIndexedAccess: true`:

```typescript
const names: string[] = ["Ali", "Vali"];
const tenth = names[99];
// tenth: string | undefined

// tenth.toUpperCase(); // ❌ 'tenth' is possibly 'undefined'

if (tenth !== undefined) {
  tenth.toUpperCase(); // ✅
}

// For-of loop — type narrowed automatically
for (const name of names) {
  name.toUpperCase(); // ✅ name: string
}

// Optional chaining
const safe = names[99]?.toUpperCase() ?? "default";
```

Object index access:

```typescript
const map: Record<string, number> = { a: 1, b: 2 };
const value = map["c"];
// noUncheckedIndexedAccess: false → value: number (xavfli)
// noUncheckedIndexedAccess: true → value: number | undefined (xavfsiz)
```

### Edge Cases

- For-of loop type narrow qiladi: `for (const x of arr) { x: T }` (no `| undefined`)
- Tuple index access tekshirilmaydi: `[string, number][0]` har doim `string` (tuple bilan flag ta'sir qilmaydi)
- `Array.prototype.at()` ham `T | undefined` qaytaradi (default xulqi)
- `arr.length` check'dan keyin TypeScript narrow qila olmaydi: `if (i < arr.length) { arr[i] }` — hali ham `| undefined`

### Follow-up savollar

1. **"`noUncheckedIndexedAccess` nima uchun `strict` ichida emas?"** — Mavjud kodlarda yuzlab yangi xato chiqarishi mumkin (har array index access). Migration breaking change bo'lardi.
2. **"For-of vs traditional for loop — qaysi biri afzal?"** — For-of (`for (const x of arr)`) ko'p hollarda — `x` type aniq, off-by-one xato yo'q. Traditional `for (let i = 0; ...)` faqat index kerak bo'lganda.

</details>

---

### Savol 11: `satisfies` operator (TS 4.9) qachon ishlatiladi va `as const` dan farqi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`satisfies` (TS 4.9) — expression ning type ga mosligini tekshiradi, lekin inference saqlanadi (widening yo'q). `as const` literal va readonly qiladi (widening to'xtatadi, mutate'siz). `satisfies` type check + inference, `as const` immutability + literal.

### To'liq tushuntirish

| Aspect | `as Type` | `as const` | `satisfies` |
|--------|-----------|------------|-------------|
| Type check | ❌ Force cast | ✅ N/A | ✅ Validate |
| Inference saqlanadi | ❌ Cast'ga aylanadi | ✅ Literal | ✅ Literal |
| Readonly | ❌ | ✅ Recursive | ❌ |
| Use case | Force assertion | Immutable config | Validate + keep type |

`satisfies` pattern:
- Type validation kerak, lekin inference saqlash
- Object literal har property aniq tipga ega (inference yaxshi)
- Property qiymat literal saqlanadi

### Kod misol

```typescript
interface Color {
  rgb: [number, number, number];
  hex: string;
}

// ❌ as Type — inference yo'q
const red1 = {
  rgb: [255, 0, 0],
  hex: "#FF0000",
} as Color;
red1.hex; // string — literal yo'q

// ❌ Annotation — inference yo'q
const red2: Color = {
  rgb: [255, 0, 0],
  hex: "#FF0000",
};
red2.hex; // string — literal yo'q

// ✅ satisfies — inference saqlanadi
const red3 = {
  rgb: [255, 0, 0],
  hex: "#FF0000",
} satisfies Color;
red3.hex; // "#FF0000" — literal!
red3.rgb; // [number, number, number] — tuple, lekin widened element
```

`satisfies` + `as const` birga:

```typescript
const config = {
  theme: "dark",
  retries: 3,
  features: ["auth", "logging"],
} as const satisfies {
  theme: string;
  retries: number;
  features: readonly string[];
};

config.theme;    // "dark" (literal)
config.features; // readonly ["auth", "logging"]
// config.theme = "light"; // ❌ readonly
```

Discriminated union validation:

```typescript
type Action =
  | { type: "increment"; amount: number }
  | { type: "reset" };

const actions = {
  inc: { type: "increment", amount: 1 },
  reset: { type: "reset" },
} satisfies Record<string, Action>;

actions.inc.amount; // number (saqlandi, narrowing ishlaydi)
// actions.inc.type narrowed: "increment"
```

### Edge Cases

- `satisfies` excess property checking ishlaydi
- `satisfies` widening qilmaydi — `as const` bilan birga eng aniq inference
- `satisfies` keyin `as` ham mumkin: `({ ... } satisfies T) as TFull`
- TS 4.9 dan eski versiyalarda `satisfies` yo'q

### Follow-up savollar

1. **"`satisfies` ni har joyda ishlatish kerakmi?"** — Ko'p hollarda ha — annotation o'rniga. Lekin function parameter da kerak emas (parameter type allaqachon validatsiya qiladi).
2. **"`satisfies` va explicit annotation farqi qachon ahamiyatli?"** — Annotation widening qiladi (`hex: string` literal saqlamaydi). `satisfies` inference saqlaydi. Object literal config lar uchun `satisfies` afzal.

</details>

---

## Output prediction savollari

### Savol 12: Array va Tuple tiplari — output quiz [Junior+]

**Savol:** Har tipning `length` tipi nima? `C` da `push()` ishlaydi mi? `D` da-chi?

```typescript
type A = string[];
type B = readonly string[];
type C = [string, number];
type D = readonly [string, number];
type E = [string, ...number[]];
type F = [string, number?];
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A: `length: number` (mutable, push ✅). B: `length: number` (readonly, push ❌). C: `length: 2`, push ✅ (TS limitation). D: `length: 2` (readonly, push ❌). E: `length: number` (rest). F: `length: 1 | 2`.

### To'liq tushuntirish

```typescript
type A = string[];
// Mutable array. push, pop, sort ishlaydi
// A["length"] = number

type B = readonly string[];
// Immutable. Mutating method lar yo'q
// B["length"] = number

type C = [string, number];
// Mutable tuple — aniq 2 element
// C[0] = string, C[1] = number
// C["length"] = 2
// push() ISHLAYDI! ⚠️ TS known limitation

type D = readonly [string, number];
// Immutable tuple — to'liq himoyalangan
// D["length"] = 2

type E = [string, ...number[]];
// Rest element — 1+ element
// E["length"] = number (o'zgaruvchan)

type F = [string, number?];
// Optional tuple — 1 yoki 2 element
// F["length"] = 1 | 2
```

**Kalit tushuncha:** Mutable tuple da `push()` ishlaydi — TS limitation. Production'da `readonly` tuple ishlatish best practice.

### Edge Cases

- `D["length"]` — `readonly` faqat mutating method lar ni o'chiradi, lekin `length` semantic bir xil
- `E[0]` — string (har doim), `E[1]` — `number | undefined` (rest)
- `F[1]` — `number | undefined` (optional)
- Tuple length type narrowing — `if (tuple.length === 2)` bilan narrowing ishlaydi

### Follow-up savollar

1. **"Mutable tuple push muammosi qachon tuzatiladi?"** — TS issue #28508 ochiq. Hozircha workaround — `readonly` tuple ishlatish.

</details>

---

### Savol 13: Enum compile output — custom values [Middle+]

**Savol:** Bu enum JavaScript ga qanday compile bo'ladi? Har member qiymat nima?

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Active: 0`, `Inactive: 5`, `Pending: 6` (5+1, oldingi qiymatdan davom). `Status[5]` — `"Inactive"` (reverse mapping).

### Kod misol

Compiled JavaScript:

```javascript
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
Status.Pending  → 6  (5 + 1, oldingi qiymatdan davom)
Status[5]       → "Inactive" (reverse mapping)
```

### To'liq tushuntirish

Pattern `Status[Status["X"] = N] = "X"`:
1. `Status["X"] = N` — forward mapping, assignment N qaytaradi
2. `Status[N] = "X"` — reverse mapping

Auto-increment: `Active` default `0`, `Inactive` explicit `5`, `Pending` davom — `6`.

Agar `Inactive = 10` bo'lsa: `Pending = 11`.

### Edge Cases

- Computed member oldidan auto-increment ishlamaydi: `enum X { A = compute(), B }` ❌
- String va numeric aralash: birinchi string, keyingi numeric — ❌ (auto-increment kelishi mumkin emas)
- Negative number: `enum X { A = -1, B }` → B = 0
- Float: `enum X { A = 1.5, B }` → B = 2.5

### Follow-up savollar

1. **"`Object.keys(Status)` nima qaytaradi?"** — `["0", "5", "6", "Active", "Inactive", "Pending"]` — 6 ta key (forward + reverse).

</details>

---

## Coding challenges

### Savol 14: Tuple widening — xatoni toping va 3 xil usulda tuzating [Middle]

**Savol:** Bu kodda compile-time xato bor. Xatoni toping va 3 xil usulda tuzating:

```typescript
function processCoordinates(coords: [number, number]): void {
  console.log(`x: ${coords[0]}, y: ${coords[1]}`);
}

const point = [10, 20];
processCoordinates(point);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Xato: `point` type `number[]`, tuple `[number, number]` emas. TS array literal ni `const` bilan bo'lsa ham widening qiladi (element lar mutate bo'ladi). Yechim: aniq annotation, `as const`, yoki inline.

### Kod misol

```typescript
// ❌ Original
const point = [10, 20];        // number[]
processCoordinates(point);      // ❌ number[] not assignable to [number, number]

// ✅ Yechim 1: Aniq tuple annotation
const point1: [number, number] = [10, 20];
processCoordinates(point1);

// ✅ Yechim 2: as const (readonly tuple, parametr ham readonly bo'lishi shart)
const point2 = [10, 20] as const;
function processCoordinates2(coords: readonly [number, number]): void {
  console.log(`x: ${coords[0]}, y: ${coords[1]}`);
}
processCoordinates2(point2);

// ✅ Yechim 3: Inline
processCoordinates([10, 20]);  // Inline literal → tuple inference
```

### To'liq tushuntirish

TypeScript array literal ni variable ga assign qilganda `T[]` ga widening qiladi:

- `const a = [1, 2]` → `number[]` (mutable)
- `const a = [1, 2] as const` → `readonly [1, 2]` (literal tuple)
- `const a: [number, number] = [1, 2]` → `[number, number]` (mutable tuple)

`processCoordinates([10, 20])` — inline da TS contextual typing orqali tuple sifatida inference qiladi (parameter type tuple bo'lgani uchun).

### Edge Cases

- `processCoordinates([10, 20, 30])` — TS argument tuple uchun length tekshiradi (excess elements xato)
- `processCoordinates([10])` — length kam ham xato
- `as const` bilan parametr `readonly` bo'lishi shart — mutable parametr qabul qilmaydi readonly argument

### Follow-up savollar

1. **"Function ichida tuple ga aylantirish ham mumkinmi?"** — Generic constraint bilan: `function fn<T extends [number, number]>(coords: T)` — caller `[10, 20]` literal sifatida pass qilsa, T literal sifatida inferred bo'ladi.

</details>

---

### Savol 15: Enum dan `as const` ga refactoring [Middle+]

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

Enum ni `as const` object ga aylantirish, type ni `typeof obj[keyof typeof obj]` orqali olish. `isValidMethod` ni type guard ga aylantirish (`value is HttpMethod`).

### Kod misol

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

// Type guard — return type yaxshilandi
function isValidMethod(value: string): value is HttpMethod {
  return (Object.values(HTTP_METHOD) as string[]).includes(value);
}

// Ishlatish
makeRequest("/api/users", HTTP_METHOD.GET);  // ✅
makeRequest("/api/users", "GET");             // ✅ string literal ham

const userInput = "GET";
if (isValidMethod(userInput)) {
  makeRequest("/api", userInput);            // ✅ narrowed
}
```

### To'liq tushuntirish

Yaxshilanishlar:

1. **Bundle size** — IIFE o'rniga oddiy object (kichikroq)
2. **String literal flexibility** — `"GET"` to'g'ridan-to'g'ri pass qilish mumkin
3. **Type guard** — `value is HttpMethod` narrowing beradi (oddiy `boolean` emas)
4. **`isolatedModules`/`erasableSyntaxOnly`** — muammosiz
5. **Tree-shaking** — bundler hech ishlatilmagan method larni o'chirib tashlaydi

`typeof HTTP_METHOD[keyof typeof HTTP_METHOD]`:
- `typeof HTTP_METHOD` — `{ readonly GET: "GET"; ... }`
- `keyof` — `"GET" | "POST" | "PUT" | "DELETE" | "PATCH"`
- Indexed access — value union

### Edge Cases

- `Object.values` return type `string[]` (TS design choice) — cast kerak
- `as const` bilan property `readonly` — runtime'da `Object.freeze` qo'shish mumkin
- TS 5.5+ — `isValidMethod` funksiya tanasidan avtomatik type predicate (`value is HttpMethod`) inference bo'lishi mumkin

### Follow-up savollar

1. **"Enum vs `as const` — performance farqi bormi?"** — Runtime'da nol. Compile output da `as const` kichikroq (IIFE yo'q), lekin bu microbenchmark — production'da sezilmaydi.

</details>

---

### Savol 16: Type-safe config validation [Senior]

**Savol:** `as const` va generics ishlatib, type-safe configuration validator yozing. Validator faqat mavjud kalitlarni qabul qilsin va noto'g'ri qiymat tipini compile-time da ushlasin:

```typescript
const DEFAULTS = {
  theme: "light",
  fontSize: 14,
  darkMode: false,
  language: "en",
} as const;

// validateConfig({ theme: "dark" })   → ✅
// validateConfig({ fontSize: 18 })    → ✅
// validateConfig({ theme: 123 })      → ❌ number !== string
// validateConfig({ invalid: "x" })    → ❌ kalit yo'q
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`as const` bilan literal qiymatlar. Generic `Widen<T>` literal type ni base type ga aylantirish uchun. `Partial<WidenedConfig>` overrides accept qiladi.

### Kod misol

```typescript
const DEFAULTS = {
  theme: "light",
  fontSize: 14,
  darkMode: false,
  language: "en",
} as const;

type Config = typeof DEFAULTS;
type ConfigKey = keyof Config;

// Widening utility
type Widen<T> =
  T extends string ? string :
  T extends number ? number :
  T extends boolean ? boolean :
  T;

type WidenedConfig = { [K in ConfigKey]: Widen<Config[K]> };

function validateConfig(overrides: Partial<WidenedConfig>): WidenedConfig {
  return { ...DEFAULTS, ...overrides };
}

// ✅ To'g'ri
validateConfig({ theme: "dark" });
validateConfig({ fontSize: 18 });
validateConfig({ darkMode: true });

// ❌ Compile error
// validateConfig({ theme: 123 });        // number !== string
// validateConfig({ invalid: "x" });      // kalit yo'q
// validateConfig({ fontSize: "big" });   // string !== number
```

### To'liq tushuntirish

- `as const` — barcha qiymatlar literal (`"light"`, `14`, `false`)
- `typeof DEFAULTS` — `{ readonly theme: "light"; readonly fontSize: 14; ... }`
- `Widen<T>` — conditional type — literal'ni base type'ga kengaytiradi
- `Partial<WidenedConfig>` — barcha kalitlar optional, lekin tip to'g'ri bo'lishi shart
- Spread operator bilan defaults ustiga overrides

Agar `Widen` ishlatmasak, faqat `"light"` qabul qilinadi (literal), `"dark"` xato beradi.

### Edge Cases

- Nested object — `Widen` recursive qilish kerak
- Array — `Widen<readonly T[]>` `T[]` ga aylantirish kerak
- Generic constraint bilan `validateConfig<K extends ConfigKey>(overrides: { [P in K]: Widen<Config[P]> })` — aniq kalit signature

### Follow-up savollar

1. **"`satisfies` bilan bu pattern soddaroq bo'ladimi?"** — Ha, configuration definition uchun: `const DEFAULTS = { ... } satisfies ConfigShape` — type validation + literal inference birga. Override pattern uchun `Widen` hali ham kerak.

<details>
<summary><strong>Deep Dive</strong></summary>

`as const` semantics — `ConstAssertion` AST node. TypeScript checker uchun bu uch ta'sirga ega:
1. Object property'lar `readonly` modifier oladi (`{ readonly theme: "light"; ... }`)
2. String/number/boolean literal qiymatlar literal type sifatida saqlanadi (widening yo'q)
3. Array literal `readonly` tuple'ga aylanadi (`[1, 2]` → `readonly [1, 2]`)

`Widen<T>` conditional type — Distributive Conditional Type (DCT): union ustida iterate qiladi. `T extends string ? string : T extends number ? number : ...` — har literal type uchun base type'ga aylanadi.

Distributivity qoidasi: `T extends U ? X : Y` — agar `T` "naked type parameter" bo'lsa va union bo'lsa, har constituent uchun alohida hisoblanadi: `("a" | 1) extends string ? string : never` → `string | never` = `string`.

Mapped type `{ [K in ConfigKey]: Widen<Config[K]> }` — har key uchun `Widen` qo'llaniladi. `Config[K]` indexed access type — kalit qiymatining tipi.

Spec referensi: TypeScript Handbook "Mapped Types", "Conditional Types Distribution". TS 2.8+ joriy etilgan. `infer` Conditional Type ichida — pattern matching mexanizmi (TS 2.8+).

</details>

</details>

---

## Bug fix savollari

### Savol 17: Enum `Object.keys` chalkashlik [Middle]

**Savol:** Bu kod kutilmagan natija qaytaryapti. Sabab nima va qanday tuzatish kerak?

```typescript
enum Color { Red, Green, Blue }

const colorNames = Object.keys(Color);
console.log(colorNames);
// Expected: ["Red", "Green", "Blue"]
// Actual: ?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Numeric enum reverse mapping tufayli `Object.keys` 6 ta element qaytaradi: `["0", "1", "2", "Red", "Green", "Blue"]`. Yechim: faqat string key larni filter qilish, yoki `as const` object ishlatish.

### Kod misol

```typescript
enum Color { Red, Green, Blue }

console.log(Object.keys(Color));
// Actual: ["0", "1", "2", "Red", "Green", "Blue"]

// ✅ Yechim 1: Faqat NaN bo'lmagan key larni filter
const colorNames = Object.keys(Color).filter(key => isNaN(Number(key)));
console.log(colorNames); // ["Red", "Green", "Blue"]

// ✅ Yechim 2: Object.values (numeric enum uchun ham reverse mapping ko'rsatadi)
const colorValues = Object.values(Color).filter(v => typeof v === "number");
console.log(colorValues); // [0, 1, 2]

// ✅ Yechim 3: String enum — clean Object.keys
enum StringColor { Red = "RED", Green = "GREEN", Blue = "BLUE" }
console.log(Object.keys(StringColor)); // ["Red", "Green", "Blue"]

// ✅ Yechim 4 (BEST): as const object
const Color2 = { Red: 0, Green: 1, Blue: 2 } as const;
console.log(Object.keys(Color2)); // ["Red", "Green", "Blue"]
console.log(Object.values(Color2)); // [0, 1, 2]
```

### To'liq tushuntirish

Numeric enum runtime object ham forward ham reverse mapping ga ega:

```javascript
{
  "Red": 0, "Green": 1, "Blue": 2,
  "0": "Red", "1": "Green", "2": "Blue"
}
```

`Object.keys` har bir key ni qaytaradi — natija 6 ta element.

String enum reverse mapping yo'q, shuning uchun `Object.keys` clean.

`as const` object eng yaxshi — hech qanday IIFE va reverse mapping yo'q.

### Edge Cases

- `for...in` loop ham reverse mapping ni includes qiladi
- `Object.entries(enum)` — `[key, value]` pair lar 6 ta
- String + numeric aralash enum — chalkashroq

### Follow-up savollar

1. **"Enum nima uchun reverse mapping qiladi?"** — Tarixiy sabab: TypeScript birinchi versiyalarida C# style enum xohlandi, JavaScript object plain key-value, shuning uchun reverse mapping qilingan numeric uchun. String enum reverse yo'q chunki ambiguity (string key bilan kollision).

</details>

---

### Savol 18: `noUncheckedIndexedAccess` bilan kod buzilgan [Middle]

**Savol:** `noUncheckedIndexedAccess: true` yoqilgandan keyin bu kod xato beradi. Tuzating:

```typescript
const users = ["Ali", "Vali", "Sami"];

function logUsers(): void {
  for (let i = 0; i < users.length; i++) {
    const name = users[i];
    console.log(name.toUpperCase()); // ❌ 'name' is possibly 'undefined'
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`noUncheckedIndexedAccess: true` bilan `users[i]` type `string | undefined`. TS `i < users.length` check'dan keyin narrowing qila olmaydi (length check loop variable bilan bog'liq emas). Yechim: for-of loop, `!` assertion, yoki narrowing.

### Kod misol

```typescript
const users = ["Ali", "Vali", "Sami"];

// ✅ Yechim 1: for-of loop (eng yaxshi)
function logUsers1(): void {
  for (const name of users) {
    console.log(name.toUpperCase()); // name: string
  }
}

// ✅ Yechim 2: Narrowing
function logUsers2(): void {
  for (let i = 0; i < users.length; i++) {
    const name = users[i];
    if (name !== undefined) {
      console.log(name.toUpperCase());
    }
  }
}

// ⚠️ Yechim 3: Non-null assertion (riskli, lekin to'g'ri kontekstda OK)
function logUsers3(): void {
  for (let i = 0; i < users.length; i++) {
    const name = users[i]!; // We know i is valid
    console.log(name.toUpperCase());
  }
}

// ✅ Yechim 4: forEach (callback bilan)
function logUsers4(): void {
  users.forEach(name => {
    console.log(name.toUpperCase()); // name: string
  });
}
```

### To'liq tushuntirish

`noUncheckedIndexedAccess: true` array index access type ni `T | undefined` qiladi. Bu TypeScript ning conservative check — index out-of-bounds runtime crash larini oldini oladi.

For-of loop ishlatilganda TypeScript element type ni `T` deb biladi (length cheklov bilan emas, iterator protocol bilan).

`for (let i = 0; i < arr.length; i++)` pattern — TypeScript `i < length` check ni `arr[i]` ga bog'lamaydi (chunki control flow analysis bu pattern ni recognize qilmaydi).

### Edge Cases

- `arr.length` check'dan keyin `arr[arr.length - 1]` — hali ham `T | undefined`
- `Array.prototype.at()` ham `T | undefined`
- Tuple uchun ta'sir qilmaydi: `[string, number][0]` har doim `string`
- `forEach`, `map`, `filter` callback parametri `T` (no `| undefined`)

### Follow-up savollar

1. **"`noUncheckedIndexedAccess` har joyda yoqish kerakmi?"** — Yangi loyihalarda — ha. Mavjud loyihalarda — yuzlab xato chiqishi mumkin, bosqichma-bosqich migration.

</details>

---

## Xulosa

Bu bo'limdagi savollar TypeScript ning array va enum xususiyatlarini qamrab oldi:

- **Array types** — `T[]` vs `Array<T>`, semantic identical
- **Readonly arrays** — compile-time immutability, function parameter best practice
- **Tuples** — fixed-length, element-specific types, push pitfall, readonly tuple
- **Optional tuple elements** — `[T, U?]`, length union
- **Rest elements in tuples** — TS 3.0 end-position, TS 4.0 variadic (har joyda)
- **Named (labeled) tuples** — IDE hint, documentation, runtime ta'sir yo'q
- **Enums** — numeric/string, IIFE compile, reverse mapping
- **Const enums** — inline qiymatlar, `isolatedModules`/Babel cheklov
- **Enum dangers** — `Object.keys` chalkashlik, `erasableSyntaxOnly`, cross-package muammo
- **Enum alternatives** — union literal type, `as const` object
- **`noUncheckedIndexedAccess`** — `T | undefined`, for-of loop bilan yaxshi
- **`satisfies` (TS 4.9)** — type validation + inference saqlash, `as const` bilan birga
- **Bug fix patterns** — tuple widening, enum Object.keys, index access narrowing
