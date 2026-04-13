# Bo'lim 3: Arrays, Tuples va Enums

> TypeScript'ning **struktura tiplari** — array (bir xil tipdagi collection), tuple (fixed-length, heterogeneous), readonly variantlar va enum'lar. Har birining compile natijasi va amaliy qo'llanilishi. Zamonaviy TypeScript'da enum'ning alternativalari (`as const` object, union literal type) va qaysi biri qachon afzalroq ekanligi.

---

## Mundarija

- [Array Types — T[] va Array<T>](#array-types--t-va-arrayt)
- [Readonly Arrays](#readonly-arrays)
- [Tuples — Fixed-Length Tiplar](#tuples--fixed-length-tiplar)
- [Optional Tuple Elements](#optional-tuple-elements)
- [Rest Elements in Tuples](#rest-elements-in-tuples)
- [Named Tuples — Labelli Tuple Elementlari](#named-tuples--labelli-tuple-elementlari)
- [as const bilan Readonly Tuple](#as-const-bilan-readonly-tuple)
- [Enums — Numeric va String](#enums--numeric-va-string)
- [Const Enums — Inline Replace](#const-enums--inline-replace)
- [Enums Under the Hood — IIFE va Reverse Mapping](#enums-under-the-hood--iife-va-reverse-mapping)
- [Enum Pitfalls — Xavfli Holatlar](#enum-pitfalls--xavfli-holatlar)
- [Enum Alternativalari — Union Literal Types va as const](#enum-alternativalari--union-literal-types-va-as-const)
- [Decision Matrix — Enum vs Union vs as const](#decision-matrix--enum-vs-union-vs-as-const)
- [satisfies bilan Validation](#satisfies-bilan-validation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Array Types — T[] va Array<T>

### Nazariya

TypeScript'da array tipi ikki xil sintaksis bilan yoziladi — **ikkalasi bir xil** natija beradi, farq yo'q. Bu sof **stilistik** tanlov:

```typescript
// 1. Square bracket syntax — ko'proq ishlatiladi
let names: string[] = ["Ali", "Vali", "Soli"];
let ages: number[] = [25, 30, 35];

// 2. Generic syntax — Array<T>
let names2: Array<string> = ["Ali", "Vali", "Soli"];
let ages2: Array<number> = [25, 30, 35];
```

**Convention:** `string[]` ko'proq tarqalgan va o'qishga qulay. `Array<T>` sintaksisi ba'zan murakkab union yoki function type'lar bilan aniqroq bo'ladi:

```typescript
// Union array — bracket syntax'da qavslar kerak
let mixed: (string | number)[] = [1, "hello", 2];

// Generic syntax'da qavslar kerak emas
let mixed2: Array<string | number> = [1, "hello", 2];

// Nested array — bracket ham toza
let matrix: number[][] = [[1, 2], [3, 4]];
let matrix2: Array<Array<number>> = [[1, 2], [3, 4]]; // bir xil

// Murakkab tip — generic aniqroq
let handlers: Array<(event: Event) => void> = [];
let handlers2: ((event: Event) => void)[] = []; // bir xil, lekin o'qish qiyin
```

TypeScript array tipi **homogeneous** emas — union type bilan aralash array yaratish mumkin. Muhim: bo'sh array'ga **annotation kerak**, aks holda `any[]` (strict'da ham) yoki "evolving array" bo'ladi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript'da `T[]` va `Array<T>` ikkalasi ham **bir xil `Array<T>` interface**'iga resolve qilinadi — `lib.es5.d.ts`'da e'lon qilingan. `T[]` — sintaktik shakarlik (syntactic sugar), compiler uni darhol `Array<T>`'ga aylantiradi.

Checker'da array tipi `TypeFlags.Object` bilan belgilanadi (interface'dan keladi) va `ObjectFlags.Reference` + `ObjectFlags.InstantiatedTypeParameters` flag'lari bilan generic instansiyatsiya sifatida ko'riladi. `T` element type'i generic type argument sifatida saqlanadi.

**Inference asoslari:**

```typescript
// Best Common Type — barcha elementlardan umumiy tip
let arr = [1, "hello", true];
// Inferred: (string | number | boolean)[]
// Checker: union[string, number, boolean] → array element type

let nums = [1, 2, 3];
// Inferred: number[] (union'ning barcha a'zolari bir xil)

// Bo'sh array — evolving type (TS tracking)
let xs = []; // initially: any[]
xs.push(1);  // endi: number[]
xs.push("a"); // endi: (string | number)[]
```

Evolving array — TS kompileri bo'sh array'ga har `push`'dan keyin tipni kengaytirib boradi. Bu `noImplicitAny` strict mode'da ham ishlaydi, lekin murakkab kodda kutilmagandek xulqqa olib kelishi mumkin. Shuning uchun bo'sh array'larga **annotation** tavsiya etiladi.

**Array method'lari va return type'lar:**

```typescript
const names = ["Ali", "Vali", "Soli"]; // string[]

const upper = names.map(n => n.toUpperCase()); // string[]
const long = names.filter(n => n.length > 3);  // string[]
const first = names[0];                         // string (default)
```

`noUncheckedIndexedAccess: true` flag bilan `names[0]` tipi `string | undefined` bo'ladi — chunki index access safe emas (massiv bo'sh bo'lishi mumkin). Bu production kodda xavfsizroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Oddiy array deklaratsiyasi
let names: string[] = ["Ali", "Vali", "Soli"];
let ages: Array<number> = [25, 30, 35];
let flags: boolean[] = [true, false, true];

// Bo'sh array — annotation kerak
let scores: number[] = [];   // ✅ aniq number[]
scores.push(95);

// Annotation'siz — evolving (xavfli murakkab kodda)
let guesses = [];            // any[] boshida
guesses.push(1);             // TS: number[]
guesses.push("hello");       // TS: (string | number)[]

// Union array
let mixed: (string | number)[] = [1, "hello", 2, "world"];

// Nested (matrix)
const matrix: number[][] = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9],
];

// Function type array
const handlers: Array<(value: number) => number> = [
  x => x * 2,
  x => x + 1,
  x => x ** 2,
];

// Object array
interface User { id: number; name: string; }
const users: User[] = [
  { id: 1, name: "Ali" },
  { id: 2, name: "Vali" },
];

// Array method'lari — type inference ishlaydi
const doubled = users.map(u => u.id * 2);   // number[]
const filtered = users.filter(u => u.id > 1); // User[]
const found = users.find(u => u.id === 1);    // User | undefined
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
let names: string[] = ["Ali", "Vali"];
let ages: Array<number> = [25, 30];
const users: { id: number; name: string }[] = [
  { id: 1, name: "Ali" }
];
```

```javascript
// Compiled JS — type annotation'lar o'chirildi
let names = ["Ali", "Vali"];
let ages = [25, 30];
const users = [
  { id: 1, name: "Ali" }
];
// Array<T> va T[] — JS'da farq yo'q, oddiy array
```

TypeScript array tiplari compile-time konstruktsiya — JavaScript'da shunchaki array. Type inference va method signature'lar faqat compile-time'da ishlaydi.

</details>

---

## Readonly Arrays

### Nazariya

`readonly` array — elementlarni o'zgartirish, qo'shish yoki o'chirish mumkin bo'lmagan array. TypeScript'da uchta sintaksis mavjud, ularning barchasi **bir xil** natija beradi:

```typescript
// 1. readonly keyword + bracket (ko'proq ishlatiladi)
let names: readonly string[] = ["Ali", "Vali"];

// 2. ReadonlyArray<T> — built-in generic
let ages: ReadonlyArray<number> = [25, 30];

// 3. Readonly<T[]> — utility type
let scores: Readonly<number[]> = [90, 85];
```

`readonly` array'da **mutating method'lar** mavjud emas — `push`, `pop`, `splice`, `sort`, `reverse`, index assignment — hammasi cheklangan. Faqat **non-mutating** operatsiyalar ruxsat etiladi: `map`, `filter`, `find`, `slice`, `concat`, `length` o'qish, index o'qish.

**Muhim:** `readonly` faqat **compile-time** himoya. Runtime'da array oddiy mutable bo'lib qoladi — JavaScript'da hech qanday freeze yo'q. Haqiqiy runtime immutability uchun `Object.freeze()` yoki immutable library (Immer, Immutable.js) kerak.

**Nima uchun foydali:** funksiya parametrida `readonly` yozsangiz — bu funksiya array'ni **o'zgartirmasligi** kafolatlanadi. Calling code uchun ishonch. API dizayn qoidasi: **input array'lar doim `readonly`**, ichki implementation esa mutable ishlatishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

`readonly T[]` va `T[]` — type hierarchy'da alohida pozitsiyalar. `readonly T[]` — `T[]`'ning **supertype**'i (mutable → readonly yo'nalishi bo'yicha):

```
Type Hierarchy (array):

        readonly T[]  ← supertype (mutating method'lar yo'q)
             ▲
             │ T[] → readonly T[] (assignment OK)
             │ readonly T[] → T[] (assignment ❌)
             │
            T[]        ← subtype (mutating method'lar bor)
```

Sabab: `readonly T[]` fewer guarantees (kamroq kafolat) beradi — lekin array'ni o'zgartirish "yangi" qobiliyat (extra capability), shuning uchun mutable array readonly'ga assign bo'ladi (kamroq capability'ga yo'naltirish xavfsiz), lekin teskarisi emas.

Checker'da `readonly T[]` alohida `ReadonlyArray<T>` interface sifatida ifodalanadi — u `Array<T>`'ning **subset**'i (faqat non-mutating method'lar). Assignability check:
- `Array<T> → ReadonlyArray<T>` — OK (subset'ga assign)
- `ReadonlyArray<T> → Array<T>` — xato (`push` missing)

**Funksiya parametrida `readonly` — "io-variance" pattern:**

```typescript
// Producer (output): T[] qaytarish — readonly ishlatilmaydi
function createList(): string[] { return ["a", "b"]; }

// Consumer (input): readonly T[] qabul qilish
function logList(items: readonly string[]) {
  items.forEach(x => console.log(x));
}

// Mutable array readonly parametrga berish mumkin
const arr: string[] = ["x", "y"];
logList(arr); // ✅ T[] → readonly T[]
```

Bu Postel Qonuniga mos: *"be conservative in what you send, be liberal in what you accept"* — input'da keng (`readonly`), output'da aniq (`T[]`).

**Runtime'da:** `readonly` JavaScript'da iz qoldirmaydi. Compile vaqtida `tsc` `readonly` keyword'ni o'chirib tashlaydi. JS kodda `arr.push("x")` ishlaydi — TS'da xato edi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
const items: readonly string[] = ["a", "b", "c"];

// ❌ Mutating methods — hammasi xato
items.push("d");       // ❌ Property 'push' does not exist
items.pop();            // ❌
items.splice(0, 1);     // ❌
items[0] = "x";         // ❌ Index signature permits reading only
items.sort();           // ❌ sort mutates
items.reverse();        // ❌ reverse mutates
(items as any).length = 0; // ⚠️ any bilan aldash — xato signal yo'q

// ✅ Non-mutating methods — ishlaydi
const upper = items.map(x => x.toUpperCase());     // string[]
const filtered = items.filter(x => x !== "b");     // string[]
const found = items.find(x => x === "a");           // string | undefined
const joined = items.join(", ");                    // string
const sliced = items.slice(0, 2);                   // string[]
const len = items.length;                            // number
const first = items[0];                              // string (o'qish OK)
```

Funksiya parametrida `readonly` — best practice:

```typescript
// ✅ Input doim readonly — funksiya array'ni o'zgartirmaydi deb kafolatlash
function sum(numbers: readonly number[]): number {
  let total = 0;
  for (const n of numbers) total += n;
  return total;
  // numbers.push(0); // ❌ readonly — o'zgartirib bo'lmaydi
}

// Mutable va readonly array'larni ikkalasini ham qabul qiladi
const mut: number[] = [1, 2, 3];
const ro: readonly number[] = [4, 5, 6];

sum(mut);  // ✅
sum(ro);   // ✅

// Sort kerak bo'lsa — copy qilib sort
function sortedList(items: readonly string[]): string[] {
  return [...items].sort(); // ✅ spread copy + sort (original tegmaydi)
  // items.sort(); // ❌
}
```

`readonly` va `const` farqi:

```typescript
// const — variable binding'ni qotiradi (qayta assign yo'q)
const arr = [1, 2, 3];
// arr = [4, 5, 6]; // ❌ Cannot assign (binding)
arr.push(4);         // ✅ mutation OK — const elementlarni cheklamaydi

// readonly — elementlarni qotiradi
let roArr: readonly number[] = [1, 2, 3];
roArr = [4, 5, 6];   // ✅ binding OK (let)
// roArr.push(4);    // ❌ readonly (mutation)

// Ikkala himoya birga
const fullyImmutable: readonly number[] = [1, 2, 3];
// fullyImmutable = [4, 5, 6]; // ❌ const
// fullyImmutable.push(4);     // ❌ readonly
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
const items: readonly string[] = ["a", "b"];
const nums: ReadonlyArray<number> = [1, 2, 3];

function process(data: readonly number[]): number {
  return data.reduce((a, b) => a + b, 0);
}
```

```javascript
// Compiled JS — readonly to'liq o'chiriladi
const items = ["a", "b"];
const nums = [1, 2, 3];

function process(data) {
  return data.reduce((a, b) => a + b, 0);
}
// Runtime'da himoya YO'Q — items.push("c") JS'da ishlaydi
// readonly faqat compile-time tekshiruv
```

Runtime immutability kerak bo'lsa:

```javascript
// Object.freeze() — runtime immutability
const items = Object.freeze(["a", "b"]);
items.push("c"); // TypeError (strict mode) yoki sukut (sloppy mode)
```

</details>

---

## Tuples — Fixed-Length Tiplar

### Nazariya

Tuple — **aniq uzunlikdagi** array, har bir elementning **alohida tipi** bor. JavaScript'da tuple yo'q — oddiy array ishlatiladi. TypeScript tuple'ni **type system orqali** qo'llab-quvvatlaydi.

```typescript
// Tuple — har element alohida tip
let user: [string, number] = ["Ali", 25];
// user[0] type: string
// user[1] type: number

// Array — barcha element bir xil tip
let names: string[] = ["Ali", "Vali"];
// names[0] type: string (har doim)
```

Tuple va Array orasidagi asosiy farqlar:

| Xususiyat | Array `T[]` | Tuple `[T, U]` |
|-----------|-------------|----------------|
| Uzunlik | O'zgaruvchan | Fixed (compile-time) |
| Element tipi | Barcha bir xil | Har biri alohida |
| `length` tipi | `number` | Literal son (`2`, `3`, ...) |
| Index access | `T` (yoki `T \| undefined`) | Aniq tip per index |
| Mutating | Ruxsat | ⚠️ Ruxsat (gotcha — quyida) |

Tuple `length` tipi — **literal son** (masalan `2`). Bu tuple'ning compile-time'da "fixed-length" ekanligining asosiy belgisi. `lib.es5.d.ts`'da tuple interface'lar uchun `length` property'si literal type sifatida belgilangan.

**Muhim gotcha:** Tuple'da `push()` va boshqa mutating method'lar **ishlaydi** — TS to'liq himoya qilmaydi. Bu "fixed-length" kafolatining eng katta zaif nuqtasi. Yechim: `readonly` tuple.

<details>
<summary><strong>Under the Hood</strong></summary>

Tuple tipi TypeScript checker'da `ObjectFlags.Tuple` flag bilan belgilanadi. `Array<T>`'ning maxsus variantsiyasi — har element alohida tip bilan. Compile vaqtida tuple'lar `TupleType` class sifatida saqlanadi, `elementTypes: Type[]` va `minLength: number` bilan.

**Tuple `length` — literal type:**

```typescript
let point: [number, number] = [10, 20];
type PointLength = typeof point.length; // type: 2

let triplet: [string, number, boolean] = ["a", 1, true];
type TripletLength = typeof triplet.length; // type: 3
```

Nima uchun: tuple type'ning `length` property'si checker tomonidan literal number type sifatida qaytariladi. Oddiy array'da esa `length: number` (keng tip). Bu farq `ArrayType.typeArguments` va `TupleType.fixedLength` ni ajratish asosida.

**Tuple va push muammosi:**

Tuple `[string, number]` uchun TypeScript `Array<string | number>`'dan meros olingan method signature'larni ishlatadi. `push` method'ining parametr tipi — element'larning union'i (`string | number`). Shuning uchun `push("hi")` xato bermaydi:

```typescript
let pair: [string, number] = ["hello", 42];

// push signature (implicit): (...items: (string | number)[]) => number
pair.push("extra"); // ✅ TS to'xtatmaydi — "string | number" valid
pair.push(true);    // ❌ boolean — valid emas
```

Natija: runtime'da `pair = ["hello", 42, "extra"]` (3 element), lekin TS hali ham `[string, number]` (2 element) deb ko'radi. Bu **silent mismatch**:

```typescript
console.log(pair.length); // Runtime: 3, TS: 2
// pair[2]; // ❌ TS: Tuple type has no element at index 2
// (lekin runtime'da mavjud — pair[2] === "extra")
```

Yechim: **`readonly` tuple**. Readonly tuple'da `push` method mavjud emas — chunki `ReadonlyArray` interface'da mutating method'lar yo'q.

```typescript
let pair: readonly [string, number] = ["hello", 42];
// pair.push("extra"); // ❌ Property 'push' does not exist
// pair[0] = "world";  // ❌ Read-only property
```

**Runtime'da tuple:** JavaScript'da tuple tushunchasi yo'q. TS tuple runtime'da oddiy array. `const [a, b] = tuple` standart JS destructuring, tuple metadata faqat compile-time'da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tuple asoslari:

```typescript
// Tuple literal — aniq tiplar va aniq tartib
let point: [number, number] = [10, 20];
let entry: [string, number, boolean] = ["key", 42, true];

// Element'larning tipi aniq
const x: number = point[0]; // ✅ number
const y: number = point[1]; // ✅ number
// const z = point[2];      // ❌ Tuple type '[number, number]' has no element at index '2'

// Destructuring — har element to'g'ri tip
const [first, second] = entry;
// first: string, second: number
const [, , flag] = entry;
// flag: boolean

// Tuple length — literal
type PointLen = typeof point.length; // type: 2 (number emas)
```

Tuple real-world pattern'lari:

```typescript
// 1. React useState pattern — tuple return
function useState<T>(initial: T): [T, (next: T) => void] {
  let value = initial;
  const setValue = (next: T) => { value = next; };
  return [value, setValue];
}

const [count, setCount] = useState(0);
// count: number, setCount: (next: number) => void

// 2. Response tuple — [error, data] pattern (Go-style)
async function fetchUser(id: number): Promise<[Error | null, User | null]> {
  try {
    const user = await api.getUser(id);
    return [null, user];
  } catch (err) {
    return [err as Error, null];
  }
}

const [error, user] = await fetchUser(1);
if (error) { /* handle error */ }
else { /* use user */ }

// 3. Coordinate/vector — math operatsiyalar
type Vec2 = [number, number];
type Vec3 = [number, number, number];

function add2(a: Vec2, b: Vec2): Vec2 {
  return [a[0] + b[0], a[1] + b[1]];
}
```

Tuple push gotcha'si:

```typescript
let pair: [string, number] = ["hello", 42];

// ⚠️ push ishlaydi — TS to'xtatmaydi
pair.push("extra"); // string | number qabul qiladi
console.log(pair); // ["hello", 42, "extra"]
console.log(pair.length); // 3 (runtime)
// TS: pair.length === 2 deb hisoblaydi — silent mismatch

// ✅ Yechim: readonly tuple
let pair2: readonly [string, number] = ["hello", 42];
// pair2.push("extra"); // ❌ Property 'push' does not exist on 'readonly'
// pair2[0] = "world";  // ❌ Read-only
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
let point: [number, number] = [10, 20];
let entry: [string, number, boolean] = ["key", 42, true];
const [x, y] = point;
const [k, v, f] = entry;
```

```javascript
// Compiled JS — tuple oddiy array bo'lib qoladi
let point = [10, 20];
let entry = ["key", 42, true];
const [x, y] = point;
const [k, v, f] = entry;
// JS'da tuple va array farqi YO'Q — faqat TS compile-time tushunchasi
```

Tuple — pure type system constructi. Runtime'da TypeScript hech qanday "tuple wrapper" yaratmaydi.

</details>

---

## Optional Tuple Elements

### Nazariya

Tuple'da ba'zi element'lar **ixtiyoriy** (optional) bo'lishi mumkin — `?` bilan belgilanadi. Muhim cheklov: **optional element'lar faqat oxirida** joylashishi mumkin (required element'dan keyin optional, optional'dan keyin required — xato).

```typescript
// Optional element — oxirida
type Point2D = [number, number];
type Point3D = [number, number, number?]; // z ixtiyoriy

let p1: Point3D = [10, 20];          // ✅ 2 element
let p2: Point3D = [10, 20, 30];      // ✅ 3 element
```

Optional element'lar **funksiya parametrlari** uchun o'xshash (`function f(a, b, c?)`) — oxirida bo'lishi kerak. Sabab: o'rtada optional bo'lsa, qaysi element berilganini aniqlash ambiguous bo'ladi.

```typescript
// ❌ O'rtada optional mumkin emas
type Bad = [number?, string, boolean];
// Error: A required element cannot follow an optional element

// ✅ To'g'ri tartib
type Good = [number, string?, boolean?];
```

Optional element'ning tipi — `T | undefined`, ya'ni destructure qilinganda `undefined` bo'lishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Optional tuple element'lar checker'da `TupleType.elementFlags`'da `ElementFlags.Optional` flag bilan belgilanadi. Har tuple element'ining "minimum required count" va "maximum allowed count" hisoblanadi:

```typescript
type FlexTuple = [string, number?, boolean?];
// minLength: 1 (faqat string required)
// maxLength: 3 (barcha element'lar bo'lsa)
```

**Optional tuple va `length` literal type:**

```typescript
type FlexLength = FlexTuple["length"]; // type: 1 | 2 | 3
// Checker: minLength'dan maxLength'gacha barcha variantlar
```

Bu `1 | 2 | 3` union type — tuple uzunligi runtime'da shu 3 ta qiymatdan biri bo'ladi. Oddiy `[string, number]` tuple'da esa `length: 2` (bitta literal).

**Optional element'larni destructure qilish:**

```typescript
type Response = [number, string, object?]; // [status, msg, data?]

const res1: Response = [200, "OK"];
const [status, msg, data] = res1;
// status: number
// msg: string
// data: object | undefined (optional → undefined mumkin)
```

Checker destructuring paytida optional element'larning tipini `T | undefined`'ga kengaytiradi — chunki destructure qilayotganda qaysi variantdan olayotgani noma'lum.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Optional tuple — oxirgi element'lar
type Point3D = [number, number, number?];

let p1: Point3D = [10, 20];           // ✅ z yo'q
let p2: Point3D = [10, 20, 30];       // ✅ z bor

const [x1, y1, z1] = p1;
// x1: number, y1: number, z1: number | undefined

if (z1 !== undefined) {
  // z1: number (narrowed)
}

// Funksiya signature — optional tuple
type HttpResponse = [statusCode: number, message: string, body?: object];

function handleResponse(response: HttpResponse) {
  const [code, msg, body] = response;
  // code: number, msg: string, body: object | undefined

  if (body !== undefined) {
    console.log("Body:", body);
  } else {
    console.log(`${code}: ${msg}`);
  }
}

handleResponse([200, "OK", { id: 1 }]);  // ✅
handleResponse([404, "Not Found"]);      // ✅ body yo'q
handleResponse([500, "Error", { cause: "db" }]); // ✅

// Ketma-ket optional
type FlexTuple = [string, number?, boolean?];
let a: FlexTuple = ["hello"];            // ✅
let b: FlexTuple = ["hello", 42];        // ✅
let c: FlexTuple = ["hello", 42, true];  // ✅

// Length ham union
type FlexLen = FlexTuple["length"]; // 1 | 2 | 3

// ❌ Xato joylar
// type Bad1 = [number?, string]; // required after optional
// type Bad2 = [string, boolean?, number]; // required after optional
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
type Point3D = [number, number, number?];
let p1: Point3D = [10, 20];
let p2: Point3D = [10, 20, 30];

function handle(res: [number, string, object?]) {
  const [code, msg, body] = res;
}
```

```javascript
// Compiled JS — optional marker o'chiriladi
let p1 = [10, 20];
let p2 = [10, 20, 30];

function handle(res) {
  const [code, msg, body] = res;
  // body — undefined bo'lishi mumkin (JS destructuring standart xulqi)
}
// Optional tuple — faqat compile-time tekshiruv
// Runtime'da oddiy array, hech qanday length tekshiruv yo'q
```

</details>

---

## Rest Elements in Tuples

### Nazariya

Rest element — tuple ichida `...` bilan **qo'shimcha element'larni** belgilash. Bu tuple'ga "o'zgaruvchan uzunlik" beradi, lekin boshqa element'larning tiplari saqlanadi.

```typescript
// Rest element — oxirida
type StringAndNumbers = [string, ...number[]];

let a: StringAndNumbers = ["hello"];                 // ✅ 0 number
let b: StringAndNumbers = ["hello", 1, 2, 3];        // ✅ 3 number
let c: StringAndNumbers = ["hello", 1, 2, 3, 4, 5];  // ✅ 5 number
// let d: StringAndNumbers = [1, 2, 3];              // ❌ birinchisi string bo'lishi kerak
```

**TS 4.2+:** Rest element har pozitsiyada bo'lishi mumkin (boshida, o'rtada, oxirida):

```typescript
// Boshida — oxirgi element aniq
type TrailingString = [...number[], string];
let t: TrailingString = [1, 2, 3, "end"]; // ✅

// O'rtada — boshi va oxiri aniq
type Sandwich = [string, ...number[], string];
let s: Sandwich = ["start", 1, 2, 3, "end"]; // ✅

// Cheklov: bitta tuple'da faqat bitta rest element
// type Invalid = [...string[], ...number[]]; // ❌
```

Rest element'lar funksiya parametrlari bilan chambarchas bog'liq — `function log(msg: string, ...codes: number[])` signature'i aslida `[msg: string, ...codes: number[]]` tuple.

<details>
<summary><strong>Under the Hood</strong></summary>

Rest tuple element'lar checker'da `ElementFlags.Rest` bilan belgilanadi. `TupleType.elementFlags` array'ida har element uchun flag saqlanadi.

**Variadic tuple types (TS 4.0+):** Rest element generic type parameter'lar bilan ishlashi mumkin — bu TypeScript'ning eng kuchli type-level feature'laridan biri.

```typescript
// Variadic tuple — T va U o'zlari tuple
function concat<T extends unknown[], U extends unknown[]>(
  a: [...T],
  b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}

const result = concat([1, 2], ["a", "b"]);
// result type: [number, number, string, string]
// Har element aniq tipini saqlaydi
```

Checker bu yerda generic inference orqali `T = [number, number]` va `U = [string, string]`'ni aniqlaydi, keyin `[...T, ...U]` tuple'ni hisoblaydi.

**Rest element o'rtasida (TS 4.2+):**

Eski versiyalarda rest element faqat oxirida bo'lishi mumkin edi. TS 4.2'dan rest har joyda bo'lishi mumkin:

```typescript
type Sandwich<T> = [start: string, ...middle: T[], end: string];
// middle — rest, "labeled"

const s: Sandwich<number> = ["begin", 1, 2, 3, "finish"];
// start: "begin", middle: [1, 2, 3], end: "finish"
```

Bu "labeled rest" variadic tuple type'larini yanada kuchaytiradi — masalan, `Parameters<T>` utility bilan funksiya parametrlarini ajratish, `Head<T>`/`Tail<T>` recursive type'lar uchun asos.

**Runtime'da:** Rest tuple element'lar JS'da hech qanday maxsus konstruktsiyaga aylanmaydi — destructuring standart rest pattern'i (`const [a, ...rest] = arr`) kabi ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Rest tuple asosiy ishlatilishlari:

```typescript
// 1. Oxirida rest
type Args = [string, ...number[]];
function log(...args: Args) {
  const [label, ...nums] = args;
  console.log(label, nums);
}
log("total", 1, 2, 3);
log("count", 42);

// 2. Boshida rest
type ListWithLast = [...number[], string];
function process(items: ListWithLast) {
  const label = items[items.length - 1];
  // TS: label: string ✅
  const nums = items.slice(0, -1);
  return { label, nums };
}
process([1, 2, 3, "end"]);

// 3. O'rtada rest (TS 4.2+)
type Wrapped<T> = [prefix: string, ...content: T[], suffix: string];

function wrap<T>(items: Wrapped<T>) {
  return items;
}
wrap(["[", 1, 2, 3, "]"]);
```

Variadic tuple — real-world patterns:

```typescript
// concat — tuple'larni birlashtirish
function concat<T extends unknown[], U extends unknown[]>(
  a: [...T],
  b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}

const joined = concat([1, 2] as const, ["a", "b"] as const);
// joined type: [1, 2, "a", "b"]

// head va tail — first element va qolgani
function head<T>([first]: [T, ...unknown[]]): T {
  return first;
}

function tail<T extends unknown[]>([, ...rest]: [unknown, ...T]): T {
  return rest;
}

const h = head([1, 2, 3]);      // h: number (1)
const t = tail([1, "a", true]); // t: (string | boolean)[]

// Curry pattern
type Curried<Args extends unknown[], R> =
  Args extends [infer Head, ...infer Tail]
    ? (arg: Head) => Curried<Tail, R>
    : R;

// Funksiya parametrlarini tuple sifatida ishlatish
function apply<Args extends unknown[], R>(
  fn: (...args: Args) => R,
  args: Args
): R {
  return fn(...args);
}

function greet(name: string, age: number): string {
  return `${name} is ${age}`;
}

const result = apply(greet, ["Ali", 25]); // result: string
// apply(greet, ["Ali"]); // ❌ tuple to'liq kerak
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
type StringAndNumbers = [string, ...number[]];
let a: StringAndNumbers = ["hello", 1, 2, 3];

type Sandwich = [string, ...number[], string];
let b: Sandwich = ["start", 1, 2, "end"];

function concat<T extends unknown[], U extends unknown[]>(
  a: [...T], b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}
```

```javascript
// Compiled JS — rest element type'lari to'liq o'chiriladi
let a = ["hello", 1, 2, 3];
let b = ["start", 1, 2, "end"];

function concat(a, b) {
  return [...a, ...b];
}
// Generic va tuple type'lari JS'da iz yo'q
// Runtime'da oddiy spread operator
```

Rest tuple — pure type-level construct. JS'da faqat spread operator qoladi (`...`), lekin tuple type metadata yo'qoladi.

</details>

---

## Named Tuples — Labelli Tuple Elementlari

### Nazariya

Named tuples (TS 4.0+) — tuple element'lariga **nom** (label) berish. Bu **faqat documentation** uchun: runtime'da hech qanday ta'sir yo'q, hatto type identity ham o'zgarmaydi. Label'lar IDE tooling'da va error message'larda ko'rsatiladi.

```typescript
// Named tuple — element'larga label
type Point = [x: number, y: number];
type HttpResponse = [
  statusCode: number,
  body: string,
  headers?: Record<string, string>
];

// Label FAQAT compile-time metadata — runtime ta'sir yo'q
const point: Point = [10, 20];
// point.x;          // ❌ Label — property emas
// const [x, y] = point; // ✅ destructuring bilan olish

// IDE hover'da label'lar ko'rinadi:
function getUser(): [id: number, name: string, email: string] {
  return [1, "Ali", "ali@mail.com"];
}
```

**Constraint:** barcha element'lar **labeled** yoki barcha **unlabeled** bo'lishi kerak — qisman labeling taqiqlangan.

```typescript
// ❌ Qisman labeling — xato
type Mixed = [name: string, number];
// Error: Tuple members must all have names or all not have names

// ✅ Hammasi labeled
type AllNamed = [name: string, age: number];

// ✅ Hammasi unlabeled
type NoLabels = [string, number];
```

Named va unnamed tuple'lar **bir xil type identity**'ga ega — label'lar type system'ga ta'sir qilmaydi:

```typescript
type Named = [x: number, y: number];
type Unnamed = [number, number];

let a: Named = [10, 20];
let b: Unnamed = a; // ✅ bir-biriga assign bo'ladi — label farq qilmaydi
```

<details>
<summary><strong>Under the Hood</strong></summary>

Named tuple label'lari TypeScript checker'da **type system metadata** sifatida saqlanadi — ular `TupleType.labeledElementDeclarations` array'ida joylashadi. Label'lar type identity'ga ta'sir qilmaydi — checker `isTypeRelatedTo` funksiyasida faqat element tiplarini solishtiradi, nomlarini emas.

**Runtime'da label'lar butunlay yo'qoladi:** JavaScript'da tuple tushunchasi yo'q, shuning uchun named tuple element'lariga `.x` yoki `.y` bilan murojaat qilib bo'lmaydi. Ular oddiy array'ning indeksli element'lari. Label'lar faqat uch joyda ishlatiladi:

1. **IDE IntelliSense** — hover va autocomplete'da parametr nomlarini ko'rsatadi
2. **Error message'lar** — xato xabarlarida label nomi chiqadi (index raqami o'rniga)
3. **`Parameters<T>`** — funksiya parametrlarini tuple sifatida ajratganda label'lar parametr nomlariga aylanadi

```typescript
type Fn = (...args: [x: number, y: number]) => void;
// IDE: (x: number, y: number) => void — label'lar parametr nomi bo'ldi

type Args = Parameters<Fn>;
// Args = [x: number, y: number]
```

**Nima uchun qisman labeling taqiqlangan:**

Sabab: compiler'ning tuple representation'ida consistency. Agar `[name: string, number]` ruxsat etilganda, `concat`, `spread`, va `rest` operatsiyalarida label propagation noaniq bo'lib qolardi:

```typescript
// Agar qisman labeling ruxsat etilganda:
type A = [name: string, number];
type B = [...A, boolean]; // B = ? [name: string, number, boolean]?
// label propagation qoidasi belgisiz — shuning uchun to'liq taqiqlangan
```

Decision: hammasi labeled yoki hammasi unlabeled — bu "all-or-nothing" qoidasi compiler'ni soddalashtiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Named tuple — documentation
type GeoLocation = [latitude: number, longitude: number];
type UserInfo = [id: number, name: string, active: boolean];

const nyc: GeoLocation = [40.7128, -74.0060];
// IDE hover: [latitude: number, longitude: number]
// Destructure:
const [lat, lng] = nyc;

const user: UserInfo = [1, "Ali", true];
const [id, name, active] = user;

// Funksiya return type — named tuple
function getCoordinates(): [x: number, y: number, z: number] {
  return [10, 20, 30];
}

const [x, y, z] = getCoordinates();
// IDE yaxshi autocomplete: "x", "y", "z"

// Parameters<T> bilan label ekstraktsiyasi
type Handler = (userId: number, action: string) => void;
type HandlerArgs = Parameters<Handler>;
// type: [userId: number, action: string]

// Rest tuple + label
type LogArgs = [level: "info" | "warn" | "error", ...messages: string[]];

function log(...args: LogArgs) {
  const [level, ...messages] = args;
  console.log(`[${level}]`, ...messages);
}

log("info", "Server started");
log("error", "Connection failed", "retry in 5s");
```

Label ta'siri — xato xabarlarida:

```typescript
type Point = [x: number, y: number];

function distance(p: Point): number {
  return Math.sqrt(p[0] ** 2 + p[1] ** 2);
}

distance([1, "2"]);
// TS error: Type 'string' is not assignable to type 'number'
// (tuple element at position 1 — labeled "y")
// Error'da "y" parametri ekani ko'rinadi
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
type Point = [x: number, y: number];
const p: Point = [10, 20];
const [x, y] = p;

function getCoordinates(): [lat: number, lng: number] {
  return [40.7, -74.0];
}
```

```javascript
// Compiled JS — label'lar to'liq o'chiriladi
const p = [10, 20];
const [x, y] = p;

function getCoordinates() {
  return [40.7, -74.0];
}
// Named tuple — faqat TS type system va IDE uchun
// JS'da tuple oddiy array, label'lar yo'qoladi
```

Named tuple — zero-cost abstraction: compile-time'da documentation qulayligi, runtime'da nol overhead.

</details>

---

## as const bilan Readonly Tuple

### Nazariya

`as const` array'ga qo'llanilganda — natija **readonly tuple** bo'ladi. Bu `02-primitive-types.md`'da kirish sifatida ko'rilgan; bu yerda tuple kontekstida chuqurroq.

```typescript
// as const SIZ — oddiy array (widened)
const colors = ["red", "green", "blue"];
// type: string[]

// as const BILAN — readonly tuple (literal)
const colors2 = ["red", "green", "blue"] as const;
// type: readonly ["red", "green", "blue"]
// colors2[0] type: "red" (literal)
```

`as const` array'ga **uchta** effekt beradi:
1. **Literal preservation** — element'lar widen bo'lmaydi
2. **Tuple inference** — `T[]` emas, `readonly [T1, T2, ...]`
3. **Readonly marker** — `push`, index assignment va boshqa mutations taqiqlanadi

**Asosiy amaliy pattern:** array'dan **union type** yaratish — `typeof arr[number]` bilan:

```typescript
const ROLES = ["admin", "editor", "viewer"] as const;
type Role = typeof ROLES[number];
// type Role = "admin" | "editor" | "viewer"

// Runtime array + compile-time type — DRY
function checkRole(user: User, role: Role): boolean {
  return user.roles.includes(role);
}

// Runtime'da iterate qilish mumkin
ROLES.forEach(r => console.log(r));
```

Bu pattern enum'ga eng yaqin alternativa: **bitta manba** (`ROLES`), **ikkita ishlatilish** — runtime array va compile-time type.

<details>
<summary><strong>Under the Hood</strong></summary>

`as const` checker'da `checkAssertionExpression` orqali ishlanadi. Array literal'ga `as const` qo'llanilganda, checker quyidagi qadamlarni bajaradi:

**1. Element widening'ni to'xtatish:**
Odatda checker `["red", "green", "blue"]` array'ining element'larini `string`'ga widen qiladi. `as const` bilan bu widening o'tkazib yuboriladi — har element o'z literal tipida saqlanadi (`"red"`, `"green"`, `"blue"`).

**2. Tuple inference:**
Normal array literal `T[]` tipiga aylanadi. `as const` bilan esa `[T1, T2, T3]` tuple tipiga aylanadi — har element o'z pozitsiyasida. Checker `getArrayLiteralTupleTypeIfApplicable` funksiyasida `as const` flag'ni tekshiradi.

**3. Readonly modifier:**
Tuple'ga `ReadonlyArray` modifier qo'shiladi — mutating method'lar cheklanadi. `ObjectFlags.ArrayLiteral` bilan birga `ObjectFlags.Readonly` flag o'rnatiladi.

**`typeof arr[number]` indexed access:**

```typescript
const arr = [1, 2, 3] as const;
// type: readonly [1, 2, 3]

type Elem = typeof arr[number];
// type: 1 | 2 | 3
```

Bu nima uchun ishlaydi: `readonly [1, 2, 3]` tuple type'ida `[number]` indexed access operator — **tuple'ning barcha element tiplari union**'ini qaytaradi. Compiler `TupleType.elementTypes` array'ini olib, ularning union'ini hisoblaydi.

Oddiy array'da `number[]` → `string[number]` → `string` (element type). Tuple'da esa `readonly [1, 2, 3][number]` → `1 | 2 | 3` (har element alohida literal).

**Recursive `as const`:**

`as const` nested object va array'larga ham **rekursiv** ta'sir qiladi:

```typescript
const config = {
  name: "app",
  versions: ["1.0", "2.0"],
  meta: { author: "Ali" }
} as const;

// type:
// {
//   readonly name: "app";
//   readonly versions: readonly ["1.0", "2.0"];
//   readonly meta: { readonly author: "Ali" };
// }
```

Har daraja readonly va literal type'larga aylanadi — bu `as const`'ning "deep freeze" xususiyati (compile-time).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pattern 1 — constant'lar va union type yaratish:

```typescript
// Array + union — DRY pattern
const HTTP_METHODS = ["GET", "POST", "PUT", "DELETE", "PATCH"] as const;
type HttpMethod = typeof HTTP_METHODS[number];
// type: "GET" | "POST" | "PUT" | "DELETE" | "PATCH"

// Runtime list
console.log(HTTP_METHODS); // ["GET", "POST", ...]

// Compile-time type
function request(method: HttpMethod, url: string) { /* ... */ }
request("GET", "/api"); // ✅
// request("FETCH", "/api"); // ❌

// Iterate
for (const method of HTTP_METHODS) {
  console.log(method); // method: "GET" | "POST" | ...
}

// includes check (cast kerak — quyida tushuntirish)
function isHttpMethod(value: string): value is HttpMethod {
  return (HTTP_METHODS as readonly string[]).includes(value);
  // Sabab: HTTP_METHODS.includes parametr tipi "GET" | ... (literal union)
  // string argument'ni qabul qilmaydi — shuning uchun cast kerak
}
```

Pattern 2 — config object'lari:

```typescript
const CONFIG = {
  env: "production",
  port: 3000,
  features: ["auth", "billing", "analytics"],
  limits: { maxUsers: 1000, maxRequests: 100 }
} as const;

// type — to'liq readonly va literal
// CONFIG.env: "production"
// CONFIG.port: 3000
// CONFIG.features: readonly ["auth", "billing", "analytics"]
// CONFIG.limits.maxUsers: 1000

// CONFIG.port = 4000; // ❌ readonly
// CONFIG.features.push("x"); // ❌ readonly
```

Pattern 3 — nested type ekstraktsiyasi:

```typescript
const ACTIONS = {
  INCREMENT: { type: "increment", payload: 1 },
  DECREMENT: { type: "decrement", payload: 1 },
  RESET: { type: "reset", payload: 0 },
} as const;

// Barcha action type'larini ajratish
type ActionType = typeof ACTIONS[keyof typeof ACTIONS];
// type: { readonly type: "increment"; readonly payload: 1 } |
//       { readonly type: "decrement"; readonly payload: 1 } |
//       { readonly type: "reset"; readonly payload: 0 }

// Faqat discriminant'ni ajratish
type ActionName = typeof ACTIONS[keyof typeof ACTIONS]["type"];
// type: "increment" | "decrement" | "reset"
```

`readonly tuple` va `as const` tuple farqi:

```typescript
// readonly tuple — element'lar NARROW EMAS
let a: readonly [string, number] = ["hello", 42];
// a[0] type: string (literal emas!)

// as const — element'lar LITERAL
const b = ["hello", 42] as const;
// b[0] type: "hello" (literal!)

// Muhim farq:
// - readonly [string, number]: "element tiplari tor"
// - readonly ["hello", 42]: "element tiplari literal"
// as const ikkalasini ham qo'llaydi
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
const HTTP_METHODS = ["GET", "POST", "PUT"] as const;
type HttpMethod = typeof HTTP_METHODS[number];

const CONFIG = {
  env: "production",
  port: 3000,
} as const;
```

```javascript
// Compiled JS — as const to'liq o'chiriladi
const HTTP_METHODS = ["GET", "POST", "PUT"];
// type HttpMethod — yo'q (type erasure)

const CONFIG = {
  env: "production",
  port: 3000,
};
// readonly faqat compile-time — runtime'da array/object oddiy, mutable
// HTTP_METHODS.push("FETCH"); // JS'da ishlaydi, TS'da xato
```

Runtime immutability kerak bo'lsa — `Object.freeze()` bilan birga:

```javascript
const HTTP_METHODS = Object.freeze(["GET", "POST", "PUT"]);
HTTP_METHODS.push("FETCH"); // TypeError (strict mode)
```

</details>

---

## Enums — Numeric va String

### Nazariya

`enum` — TypeScript'ning **alohida runtime konstruksiyasi** (JavaScript'da yo'q). Enum **nomlangan konstantalar to'plami**ni e'lon qiladi va compile vaqtida JavaScript'ga IIFE (Immediately Invoked Function Expression) sifatida aylantiriladi. Enum ikki asosiy turga ega: **numeric** va **string**.

### Numeric Enum

Qiymatlar avtomatik ravishda 0'dan boshlab o'sib boradi:

```typescript
enum Direction {
  Up,     // 0 (default)
  Down,   // 1 (avtomatik)
  Left,   // 2
  Right,  // 3
}

let dir: Direction = Direction.Up; // 0
console.log(Direction.Right); // 3
```

Boshlang'ich qiymatni aniq berish mumkin:

```typescript
enum HttpStatus {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

// Qisman belgilash — keyingi member'lar avtomatik davom etadi
enum Priority {
  Low = 1,
  Medium,     // 2 (Low + 1)
  High,       // 3
  Critical = 10,
  Emergency,  // 11 (Critical + 1)
}
```

### String Enum

Har qiymat **aniq string** bilan berilishi shart — avtomatik increment yo'q:

```typescript
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

let color: Color = Color.Red; // "RED"

// ❌ Initializer majburiy
enum BadColor {
  Red = "RED",
  Green,  // ❌ Enum member must have initializer
}
```

### Heterogeneous Enum (Aralash) — TAVSIYA ETILMAYDI

```typescript
// ⚠️ Aralash — string + number — ishlatmang
enum Mixed {
  No = 0,
  Yes = "YES",
}
// Texnik jihatdan ishlaydi, lekin chalkash va xavfli
```

**String enum** va **numeric enum** orasidagi muhim farq:
- **Numeric enum** — runtime'da **reverse mapping** bor (sondan nomga qaytish)
- **String enum** — reverse mapping **yo'q** (faqat forward)

Bu farq `Enums Under the Hood` bo'limida batafsil.

<details>
<summary><strong>Under the Hood</strong></summary>

Enum TypeScript'da **dual nature**'ga ega — ham **type**, ham **value** sifatida mavjud:

```typescript
enum Status { Active, Inactive }

// VALUE sifatida — runtime'da object
console.log(Status.Active); // 0 — value namespace

// TYPE sifatida — compile-time type check
let s: Status = Status.Active; // type namespace
```

Shu sababli enum declaration merging va namespace merging bilan ishlashi mumkin — compiler har enum uchun ikkita namespace (type va value) parallel track qiladi.

**Numeric enum tip semantikasi:**

Numeric enum `number` subtype sifatida traktovka qilinadi — bu TypeScript'ning ataylab qilingan **"intentional unsoundness"** xususiyati. Har qanday `number` `Direction`'ga assign bo'la oladi:

```typescript
enum Direction { Up, Down, Left, Right }
let d: Direction = 42; // ⚠️ Ruxsat etilgan — xato yo'q
// 42 enum'da yo'q, lekin TS compile-time'da to'xtatmaydi
```

Bu TypeScript'ning **bit flags** pattern'ini qo'llab-quvvatlashi uchun ataylab qilingan:

```typescript
enum Permission {
  Read = 1,    // 001
  Write = 2,   // 010
  Execute = 4, // 100
}

const rw: Permission = Permission.Read | Permission.Write;
// rw = 3 (011) — enum'da yo'q, lekin valid bit flag
```

Agar TS numeric enum'ni qat'iy cheklasa, bitwise operatsiyalar natijalari enum'ga assign bo'lmaydi va bit flags pattern'i ishlamas edi.

**String enum tip semantikasi:**

String enum'ga string literal to'g'ridan-to'g'ri assign qilib bo'lmaydi — bu TypeScript'ning **nominal-like typing** simulatsiyasi:

```typescript
enum Color { Red = "RED" }

let c1: Color = Color.Red; // ✅ enum member orqali
let c2: Color = "RED";     // ❌ String literal ruxsat etilmaydi
// Error: Type '"RED"' is not assignable to type 'Color'
```

Sabab: string enum'lar "nominal identity"'ga ega — `Color.Red` tipi `Color` (enum tip), `"RED"` literal tipi esa `"RED"` (string literal). Ular structurally mos (ikkalasi ham `"RED"`), lekin TypeScript ularni ataylab ajratadi — chunki string enum'ning asosiy maqsadi aynan shu nominal tekshiruv.

**Compile natijasi (IIFE):**

Enum compile vaqtida IIFE orqali JS'ga aylantiriladi — har enum `var EnumName; (function(EnumName) { ... })(EnumName || (EnumName = {}));` shaklida. Batafsil keyingi bo'limda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Numeric enum:

```typescript
// Default 0'dan boshlanish
enum Direction {
  Up,     // 0
  Down,   // 1
  Left,   // 2
  Right,  // 3
}

// Boshlang'ich qiymat
enum HttpStatus {
  OK = 200,
  Created = 201,
  NotFound = 404,
  InternalError = 500,
}

// Computed values
enum FileAccess {
  None = 0,
  Read = 1 << 0,      // 1
  Write = 1 << 1,     // 2
  ReadWrite = Read | Write, // 3
}

// Ishlatish
function setPermission(access: FileAccess) {
  if (access & FileAccess.Read) {
    console.log("Can read");
  }
  if (access & FileAccess.Write) {
    console.log("Can write");
  }
}

setPermission(FileAccess.ReadWrite);
```

String enum:

```typescript
enum LogLevel {
  Debug = "DEBUG",
  Info = "INFO",
  Warning = "WARNING",
  Error = "ERROR",
}

function log(level: LogLevel, message: string) {
  console.log(`[${level}] ${message}`);
}

log(LogLevel.Info, "Server started");
// log(LogLevel.Info, "Server started"); → [INFO] Server started
// log("INFO", "test"); // ❌ string literal assign bo'lmaydi
```

Enum switch va exhaustiveness:

```typescript
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
  Pending = "PENDING",
}

function getLabel(status: Status): string {
  switch (status) {
    case Status.Active:
      return "User is active";
    case Status.Inactive:
      return "User is inactive";
    case Status.Pending:
      return "User is pending";
    default:
      // Exhaustive check — yangi member qo'shilsa bu yerda xato
      const _exhaustive: never = status;
      return _exhaustive;
  }
}
```

Enum iterate:

```typescript
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

// String enum — Object.values toza
const colors = Object.values(Color);
// ["RED", "GREEN", "BLUE"]

for (const color of colors) {
  console.log(color);
}

// Numeric enum — filter kerak (reverse mapping tufayli)
enum Direction { Up, Down, Left, Right }
const dirNames = Object.keys(Direction).filter(k => isNaN(Number(k)));
// ["Up", "Down", "Left", "Right"]
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Numeric enum:

```typescript
// TS source
enum Direction {
  Up,
  Down,
  Left,
  Right,
}
```

```javascript
// Compiled JS — IIFE pattern
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
  Direction[Direction["Left"] = 2] = "Left";
  Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));

// Natija object:
// Direction = {
//   Up: 0,       // forward
//   Down: 1,
//   Left: 2,
//   Right: 3,
//   "0": "Up",   // reverse mapping
//   "1": "Down",
//   "2": "Left",
//   "3": "Right",
// }
```

String enum:

```typescript
// TS source
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}
```

```javascript
// Compiled JS — reverse mapping YO'Q
var Color;
(function (Color) {
  Color["Red"] = "RED";
  Color["Green"] = "GREEN";
  Color["Blue"] = "BLUE";
})(Color || (Color = {}));

// Natija object:
// Color = {
//   Red: "RED",
//   Green: "GREEN",
//   Blue: "BLUE",
// }
// Faqat forward — reverse mapping yo'q (string collision xavfi tufayli)
```

</details>

---

## Const Enums — Inline Replace

### Nazariya

`const enum` — oddiy enum'dan farqli ravishda, compile vaqtida **to'liq inline** qilinadi. JavaScript'da hech qanday object yaratilmaydi — enum qiymatlari to'g'ridan-to'g'ri literal qiymat bilan almashtiriladi.

```typescript
const enum Direction {
  Up = 0,
  Down = 1,
  Left = 2,
  Right = 3,
}

let dir = Direction.Up;       // → compile: let dir = 0
console.log(Direction.Right);  // → compile: console.log(3)

// Direction object umuman YARATILMAYDI
```

**Afzalliklari:**
- **Bundle size** — enum object JS'ga tushmaydi, faqat literal qiymatlar
- **Performance** — runtime object lookup yo'q (inline value)
- **Zero cost abstraction** — compile-time nom, runtime hech narsa

**Cheklovlari — juda muhim:**

1. **`isolatedModules: true`** bilan — **boshqa fayldan** `const enum` import qilish cheklangan. Sababi: esbuild, SWC, Babel kabi single-file transpiler'lar har faylni alohida compile qiladi — ular boshqa fayllarni o'qish imkoniga ega emas va inline qila olmaydi. TypeScript bu holatda `TS2748` xato beradi: *"Cannot access ambient const enums when the '--isolatedModules' flag is provided."*

2. **`erasableSyntaxOnly: true`** (TS 5.8+) — Node.js native TypeScript support (`--experimental-strip-types`) bilan ishlash uchun `const enum` **bloklanadi**. Sabab: Node.js faqat type erasure qila oladi (annotation'larni o'chiradi), lekin enum qiymatlarini inline qilish uchun haqiqiy transformation kerak. Shuning uchun `const enum` "erasable" hisoblanmaydi.

3. **`preserveConstEnums: true`** — bu flag yoqilsa, compiler enum object'ni ham chiqaradi (const afzalligi yo'qoladi, lekin debugging oson).

4. **Runtime'da enum object yo'q** — `Object.keys(Direction)` ishlamaydi (`Direction` JS'da mavjud emas).

**Amaliy qoida:** `const enum` faqat **monorepo** yoki **single-project tsc build**'larida foydali. Agar loyihangiz esbuild/Vite/Bun kabi modern bundler ishlatayotgan bo'lsa — `const enum`'dan **voz keching**, `as const` object yoki union literal type ishlating.

<details>
<summary><strong>Under the Hood</strong></summary>

`const enum` compile vaqtida **inline substitution** qiladi — enum reference'ni uchratganda, uni to'g'ridan-to'g'ri literal qiymat bilan almashtiradi. Bu oddiy enum'dan tubdan farq qiladi:

**Oddiy enum — runtime object yaratadi:**

```typescript
enum Direction { Up = 0, Down = 1 }
let d = Direction.Up;
// → JS: let d = Direction.Up (object property lookup)
// Direction object yaratilgan, runtime'da mavjud
```

**Const enum — inline qiladi:**

```typescript
const enum Speed { Fast = 100, Slow = 10 }
let s = Speed.Fast;
// → JS: let s = 100 /* Speed.Fast */ (literal value inline)
// Speed object umuman YARATILMAYDI
```

Comment `/* Speed.Fast */` — `sourceMap` uchun debugging metadata. Asosiy qiymat — literal `100`.

**Compiler process tartibi:**

1. **Enum declaration** ni o'qib, barcha `member → value` mapping'ni xotirada saqlaydi
2. **Har reference**'da (masalan, `Speed.Fast`) mapping'dan qiymatni topadi
3. **Literal qiymat** bilan reference'ni almashtiradi (inline)
4. **Enum declaration**'ni emit qilmaydi — JS'da hech narsa qolmaydi

**`preserveConstEnums: true`:**

Agar bu flag yoqilsa, compiler ikkala ish ham qiladi:
- Reference'larni inline qiladi
- Enum object'ni ham emit qiladi (debugging uchun)

Bu holda bundle size afzalligi yo'qoladi, lekin runtime'da enum object mavjud bo'ladi va `Object.keys` ishlaydi.

**`isolatedModules` va `erasableSyntaxOnly` muammolari:**

Modern JavaScript bundler'lar (esbuild, SWC, Babel) har faylni alohida (isolated) compile qilishadi — ular boshqa fayllarni o'qimaydi. `const enum` import qilinganda, bundler qiymatlarni topish uchun source faylni o'qishi kerak bo'lardi — lekin u bunday qilolmaydi. Shuning uchun TS `isolatedModules: true` bilan const enum cross-file import'ni bloklaydi.

TS 5.8'dagi `erasableSyntaxOnly` — Node.js 22.6+'da `--experimental-strip-types` bilan TypeScript fayllarini to'g'ridan-to'g'ri ishga tushirish uchun. Node.js faqat type annotation'larni o'chiradi (type erasure), lekin enum inline qilish kabi transformation qila olmaydi. Shuning uchun `const enum` bu flag bilan ishlamaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Const enum asosiy ishlatilish:

```typescript
const enum HttpStatus {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

function handleResponse(status: HttpStatus) {
  if (status === HttpStatus.OK) {
    return "Success";
  } else if (status === HttpStatus.NotFound) {
    return "Not found";
  }
  return "Error";
}

handleResponse(HttpStatus.OK);
// Compile vaqtida:
// handleResponse(200 /* HttpStatus.OK */);
// HttpStatus object hech qayerda yo'q
```

Bit flags (performance-sensitive kodda const enum foydali):

```typescript
const enum Permissions {
  None = 0,
  Read = 1 << 0,   // 1
  Write = 1 << 1,  // 2
  Execute = 1 << 2, // 4
  All = Read | Write | Execute, // 7
}

function check(perm: Permissions, required: Permissions): boolean {
  return (perm & required) === required;
}

check(Permissions.All, Permissions.Read);
// Compile: check(7 /* Permissions.All */, 1 /* Permissions.Read */)
```

Runtime enum object yo'q — gotcha:

```typescript
const enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}

// ❌ Runtime'da ishlamaydi
// Object.keys(Status); // ReferenceError: Status is not defined
// Object.values(Status); // ReferenceError

// Faqat enum member'larning qiymatlari inline qilinadi:
const s = Status.Active; // → const s = "ACTIVE"
console.log(s); // "ACTIVE"
```

`isolatedModules` xatosi:

```typescript
// types.ts
export const enum Theme {
  Light = "LIGHT",
  Dark = "DARK",
}

// app.ts (isolatedModules: true)
import { Theme } from "./types";
// ❌ TS2748: Cannot access ambient const enums when '--isolatedModules' is enabled

// Yechimlar:
// 1. Oddiy enum
// export enum Theme { Light = "LIGHT", Dark = "DARK" }

// 2. as const object (eng yaxshi)
// export const THEME = { Light: "LIGHT", Dark: "DARK" } as const;
// export type Theme = typeof THEME[keyof typeof THEME];
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Const enum — hech qanday IIFE yo'q:

```typescript
// TS source
const enum Direction {
  Up = 0,
  Down = 1,
  Left = 2,
  Right = 3,
}

let dir = Direction.Up;
console.log(Direction.Right);
const arr = [Direction.Down, Direction.Left];
```

```javascript
// Compiled JS — hamma narsa inline
let dir = 0 /* Direction.Up */;
console.log(3 /* Direction.Right */);
const arr = [1 /* Direction.Down */, 2 /* Direction.Left */];

// Direction object YARATILMAGAN — JS'da hech qanday iz yo'q
// Comment'lar faqat sourceMap uchun
```

Solishtirish: oddiy enum vs const enum:

```javascript
// OLDINGI (oddiy enum):
// ~200+ bayt — IIFE + object + reverse mapping
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
  // ...
})(Direction || (Direction = {}));
let dir = Direction.Up;

// KEYINGI (const enum):
// ~20 bayt — faqat qiymat
let dir = 0 /* Direction.Up */;
```

Bundle size farqi enum hajmiga bog'liq — lekin const enum doimo kichikroq (object overhead yo'q).

</details>

---

## Enums Under the Hood — IIFE va Reverse Mapping

### Nazariya

TypeScript oddiy enum (const emas) JavaScript'ga **IIFE (Immediately Invoked Function Expression)** sifatida compile bo'ladi. Bu enum'ning runtime representatsiyasini tushunish uchun kalit tushuncha.

### Numeric Enum — IIFE va Reverse Mapping

```typescript
enum Direction {
  Up,     // 0
  Down,   // 1
  Left,   // 2
  Right,  // 3
}
```

Compile natijasi:

```javascript
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
  Direction[Direction["Left"] = 2] = "Left";
  Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));
```

Bitta satrni parchalaymiz: `Direction[Direction["Up"] = 0] = "Up"`

```javascript
// Qadam 1: Direction["Up"] = 0
// Object'ga "Up" key qo'shildi, qiymati 0
// JS assignment expression — natija assign qilingan qiymatni qaytaradi (0)

// Qadam 2: Direction[0] = "Up"
// Birinchi qadamning natijasi (0) index sifatida ishlatildi
// Object'ga 0 key qo'shildi, qiymati "Up" — bu REVERSE MAPPING
```

Natija object:

```javascript
Direction = {
  Up: 0,       // forward:  name → value
  Down: 1,
  Left: 2,
  Right: 3,
  "0": "Up",   // reverse:  value → name
  "1": "Down",
  "2": "Left",
  "3": "Right",
}
```

**Reverse mapping** — sondan nomga qaytish:

```typescript
enum Direction { Up, Down, Left, Right }

console.log(Direction.Up);    // 0 — nom → qiymat
console.log(Direction[0]);    // "Up" — qiymat → nom
console.log(Direction[2]);    // "Left"

// Faqat NUMERIC enum'da ishlaydi
// String enum'da reverse mapping YO'Q
```

### String Enum — Reverse Mapping YO'Q

```typescript
enum Color {
  Red = "RED",
  Green = "GREEN",
}
```

Compile natijasi:

```javascript
var Color;
(function (Color) {
  Color["Red"] = "RED";
  Color["Green"] = "GREEN";
})(Color || (Color = {}));

// Color = {
//   Red: "RED",     // forward faqat
//   Green: "GREEN", // forward faqat
//   // "RED": "Red" — YO'Q! Reverse mapping faqat numeric da
// }
```

**Nima uchun string enum'da reverse yo'q:** agar reverse mapping bo'lsa, enum value (masalan, `"RED"`) boshqa member nomi bilan **collision** qilishi mumkin. Masalan, `enum { Red = "RED", RED = "something" }` — compiler `Color["RED"]` muammosiga uchrar edi. TypeScript bu xavfni oldini olish uchun string enum'da reverse mapping qilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

### IIFE Pattern — Nima uchun

`var Direction; (function(Direction) { ... })(Direction || (Direction = {}));` pattern ikki maqsadni amalga oshiradi:

**1. Incremental assignment (namespace-style):**

`Direction || (Direction = {})` — agar `Direction` allaqachon mavjud bo'lsa, undan foydalanamiz, aks holda yangi object yaratamiz. Bu bitta enum'ni bir nechta joyda augment qilish imkonini beradi.

```javascript
// 1-marta: Direction = undefined
//   undefined || {} → {} (yangi bo'sh object)
// 2-marta: Direction = {Up: 0, 0: "Up", ...}
//   existing || {} → existing (mavjud object)
```

**2. Declaration merging:**

TypeScript enum'lar bir necha joyda **merge** bo'lishi mumkin — bir xil nom bilan ikkita enum deklaratsiya qilinsa, ular birlashadi:

```typescript
// File A
enum Direction { Up, Down }

// File B
enum Direction { Left = 2, Right = 3 }

// Natija: { Up: 0, Down: 1, Left: 2, Right: 3 }
```

JS compile natijasida ikki IIFE ketma-ket chaqiriladi, ikkinchisi birinchisi yaratgan object'ni augment qiladi:

```javascript
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
})(Direction || (Direction = {}));

(function (Direction) {
  Direction[Direction["Left"] = 2] = "Left";
  Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));
// Ikkinchi IIFE birinchisining natijasini augment qiladi
```

Bu pattern **ambient declarations** va **namespace merging** uchun ham ishlatiladi — TypeScript'ning ichki modularlik mexanizmining bir qismi.

### `Direction[Direction["Up"] = 0] = "Up"` — Assignment Expression

Bu qadama-qadam ko'rsak:

```
Assignment expression JS'da:
  x = value   →   assignment returns `value`
```

Shuning uchun `Direction["Up"] = 0` butun ifoda **`0`** qiymatini qaytaradi. Keyin tashqi assignment:

```
Direction[0] = "Up"
```

Bu `0` index'ga `"Up"` qiymatini qo'yadi. Ikkalasi birgalikda:
- `Direction.Up = 0` (forward mapping)
- `Direction[0] = "Up"` (reverse mapping)

**String enum nima uchun reverse qilmaydi:**

Agar string enum'ga reverse mapping qo'shilsa:

```typescript
enum Color { Red = "RED" }
// Xayoliy compile:
Color["Red"] = "RED";
Color["RED"] = "Red"; // reverse

// Muammo: Color object'da ikkita key
// "Red" → "RED"
// "RED" → "Red"
// Bu o'zaro collision yaratishi mumkin (masalan, enum member "RED" deb nomlangan bo'lsa)
```

Shuningdek, `for...in` iteration'da reverse key'lar chalkashlik yaratadi. TypeScript string enum'da bu chalkashlik'ni oldini olish uchun faqat forward mapping qoldiradi.

### Object.keys va Reverse Mapping

```typescript
enum Direction { Up, Down, Left, Right }

Object.keys(Direction);
// Result: ["0", "1", "2", "3", "Up", "Down", "Left", "Right"]
// 8 ta key — 4 forward + 4 reverse

// Faqat nomlarni olish uchun filter kerak:
const names = Object.keys(Direction).filter(k => isNaN(Number(k)));
// ["Up", "Down", "Left", "Right"]
```

Filter `isNaN(Number(k))` — reverse mapping key'lari raqamli string'lar (`"0"`, `"1"`, ...), ularni `Number()` bilan parse qilsak `NaN` emas. Forward key'lar esa `NaN` beradi.

String enum'da bu muammo yo'q:

```typescript
enum Color { Red = "RED", Green = "GREEN", Blue = "BLUE" }
Object.keys(Color);
// ["Red", "Green", "Blue"] — 3 ta, reverse mapping yo'q
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Numeric enum reverse mapping amaliy ishlatilish:

```typescript
enum HttpStatus {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

// Forward
console.log(HttpStatus.OK); // 200

// Reverse — sondan nomga
function statusName(code: number): string {
  return HttpStatus[code] || "Unknown";
}

console.log(statusName(200));  // "OK"
console.log(statusName(404));  // "NotFound"
console.log(statusName(999));  // "Unknown"

// Iterate faqat nomlarni
const names = Object.keys(HttpStatus).filter(k => isNaN(Number(k)));
// ["OK", "NotFound", "ServerError"]

// Qiymatlarni
const values = Object.values(HttpStatus).filter(v => typeof v === "number");
// [200, 404, 500]
```

String enum iterate:

```typescript
enum LogLevel {
  Debug = "DEBUG",
  Info = "INFO",
  Warning = "WARN",
  Error = "ERROR",
}

// Object.keys / values toza
Object.keys(LogLevel);
// ["Debug", "Info", "Warning", "Error"]

Object.values(LogLevel);
// ["DEBUG", "INFO", "WARN", "ERROR"]

// for...in
for (const key in LogLevel) {
  console.log(key, LogLevel[key as keyof typeof LogLevel]);
}
```

Declaration merging misoli:

```typescript
// File 1: types.ts
enum Color {
  Red = "#FF0000",
  Green = "#00FF00",
}

// File 2: same-or-other.ts (same module/project)
enum Color {
  Blue = "#0000FF",
  Yellow = "#FFFF00",
}

// Merge natijasi
Color.Red;     // "#FF0000"
Color.Green;   // "#00FF00"
Color.Blue;    // "#0000FF"
Color.Yellow;  // "#FFFF00"
```

Declaration merging faqat **same module/same file** ichida ishlatiladi, chunki TS modullar orasida avtomatik merge qilmaydi.

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Numeric enum to'liq compile:

```typescript
// TS source
enum Direction {
  Up,
  Down,
  Left,
  Right,
}
console.log(Direction.Up);
console.log(Direction[0]);
```

```javascript
// Compiled JS
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
  Direction[Direction["Left"] = 2] = "Left";
  Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));
console.log(Direction.Up);   // 0
console.log(Direction[0]);    // "Up"
```

Declaration merging:

```typescript
// TS source
enum Direction { Up, Down }
enum Direction { Left = 2, Right = 3 }
```

```javascript
// Compiled JS — ikki IIFE ketma-ket
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
})(Direction || (Direction = {}));

(function (Direction) {
  Direction[Direction["Left"] = 2] = "Left";
  Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));

// Birinchi IIFE: Direction = { Up: 0, 0: "Up", Down: 1, 1: "Down" }
// Ikkinchi IIFE: existing object augment qiladi
// Final: Direction = { Up: 0, 0: "Up", Down: 1, 1: "Down", Left: 2, 2: "Left", Right: 3, 3: "Right" }
```

</details>

---

## Enum Pitfalls — Xavfli Holatlar

### Nazariya

Enum'lar bir nechta xavfli holatga ega — bular TypeScript'ning enum dizayni bo'yicha eng ko'p tanqid qilinadigan jihatlari. Bu pitfall'larni bilib, ehtiyot bo'lish yoki **alternativa** ishlatish tavsiya etiladi.

### Pitfall 1: Numeric enum'ga har qanday son

```typescript
enum Direction { Up, Down, Left, Right }

let dir: Direction = Direction.Up; // ✅ 0
dir = 99;  // ⚠️ Hech qanday xato — TS ruxsat beradi
// 99 Direction'da yo'q, lekin compiler to'xtatmaydi

// String enum — bu muammo yo'q
enum Color { Red = "RED", Green = "GREEN" }
let c: Color = Color.Red;
// c = "RANDOM"; // ❌ String literal enum'ga assign bo'lmaydi
```

Sabab: TypeScript numeric enum'ni **bit flags** uchun mo'ljallagan, shuning uchun har qanday `number` ruxsat etiladi. Bit flags pattern'ida bitwise OR natijasi enum'da yo'q bo'lgan son bo'lishi mumkin — va bu normal.

### Pitfall 2: Reverse mapping chalkashligi

```typescript
enum Status { Active, Inactive }

// Mavjud bo'lmagan index bilan ham murojaat mumkin
console.log(Status[99]); // undefined — runtime, TS xato bermaydi

// Object.keys natijasi "ikki barobar"
Object.keys(Status);
// ["0", "1", "Active", "Inactive"] — 4 ta key
// Aslida 2 ta member, lekin reverse mapping tufayli 4 ta key
```

### Pitfall 3: `Object.keys()` bilan iterate qiyin

```typescript
enum Color {
  Red,    // 0
  Green,  // 1
  Blue,   // 2
}

// Naive iterate — reverse key'lar ham chiqadi
for (const key in Color) {
  console.log(key);
  // "0", "1", "2", "Red", "Green", "Blue"
}

// To'g'ri — filter kerak
const colorNames = Object.keys(Color).filter(k => isNaN(Number(k)));
// ["Red", "Green", "Blue"]

// Yoki Object.values + filter
const colorValues = Object.values(Color).filter(v => typeof v === "number");
// [0, 1, 2]
```

### Pitfall 4: `const enum` import muammosi

```typescript
// types.ts
export const enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}

// app.ts (isolatedModules: true)
import { Status } from "./types";
// ❌ TS2748: Cannot access ambient const enums when '--isolatedModules' is enabled
```

**Sabab:** `isolatedModules: true` har faylni alohida compile qiladi (esbuild, SWC, Babel stilida). `const enum` boshqa fayldan inline qilish uchun compiler'ga source fayllarga kirish kerak — lekin isolated compile'da bu mumkin emas.

**Yechimlar:**
1. Oddiy `enum` (const emas)
2. `as const` object (tavsiya)
3. Union literal type
4. `isolatedModules: false` + faqat `tsc` (modern bundler'larsiz)

### Pitfall 5: `erasableSyntaxOnly` (TS 5.8+) bilan enum ishlamaydi

```json
// tsconfig.json
{ "compilerOptions": { "erasableSyntaxOnly": true } }
```

```typescript
enum Direction { Up, Down } // ❌ Error: enum kod hosil qiladi, erasable emas
const enum Speed { Fast = 100 } // ❌ Ham xato
```

**Sabab:** Node.js 22.6+ `--experimental-strip-types` bilan TypeScript fayllarini to'g'ridan-to'g'ri ishga tushirishi uchun faqat **type erasure** (annotation'larni o'chirish) qila oladi. Enum esa haqiqiy JavaScript runtime kod (IIFE) hosil qilishi kerak — bu erasure emas, transformation.

`erasableSyntaxOnly: true` yoqilsa, compiler bu "runtime-producing" konstruksiyalarni bloklaydi: `enum`, `namespace`, `parameter properties` (`constructor(public x: number)`), `experimentalDecorators`. Bu flag Node.js native TS support bilan to'liq mos kelishi uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

Enum pitfall'larning sababi compiler'ning enum type representation'da:

**Numeric enum soundness:**

Numeric enum `number` bilan bi-directional assignability'ga ega — `number → EnumType` va `EnumType → number` ikkalasi ham ishlaydi. Bu TypeScript jamoasining **ataylab qilingan** qarori: bit flags pattern'i uchun zarur.

```typescript
enum Perm { Read = 1, Write = 2, Execute = 4 }

// Bitwise OR natijasi enum'da yo'q bo'lgan son
const combined = Perm.Read | Perm.Write; // 3
// 3 enum'da yo'q, lekin valid "bit combination"

// Agar numeric enum strict bo'lsa:
// combined: number (enum'dan tushib qolgan)
// Assign qilish: let p: Perm = combined; // xato

// Lekin bu yomon — bit flags ishlamaydi
// Shuning uchun TS numeric enum'ni keng qo'yadi
```

String enum esa nominal identity'ga ega (unique symbol kabi) — string literal'ni string enum'ga to'g'ridan-to'g'ri assign qilib bo'lmaydi:

```typescript
enum Color { Red = "RED" }

// Compiler type hierarchy:
// "RED" (string literal) ≠ Color.Red (enum member)
// Ular structurally mos, lekin TypeScript ularni alohida ko'radi
```

**Reverse mapping muammosi:**

Numeric enum object uchun compiler `index signature`'ni `string` deb ko'radi — chunki reverse mapping key'lar string (lekin raqam kabi ko'rinadi: `"0"`, `"1"`). Shuning uchun `Direction[99]` TS'da xato bermaydi — compiler bu valid index access deb o'ylaydi:

```typescript
enum Direction { Up, Down }

// TS view:
// Direction: { [key: string]: string | Direction, Up: 0, Down: 1, "0": "Up", "1": "Down" }
// Direction[99] → valid string access → string | Direction | undefined
```

Runtime'da `Direction[99]` `undefined`, lekin TS bu xavfni ko'rsatmaydi.

**`Object.keys()` muammosining ildizi:**

Numeric enum object'da N ta member uchun **2N** ta key mavjud bo'ladi (N forward + N reverse). Bu `Object.keys()` natijasini chalkashtirdi. String enum'da esa N ta member uchun **N** ta key — chunki reverse mapping yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pitfall'larni amaliy ko'rish:

```typescript
enum Direction { Up, Down, Left, Right }

// Pitfall 1: har qanday son
function move(dir: Direction) {
  switch (dir) {
    case Direction.Up: return "up";
    case Direction.Down: return "down";
    case Direction.Left: return "left";
    case Direction.Right: return "right";
  }
  return "unknown"; // 99 kelganda
}

move(99); // ⚠️ TS ruxsat beradi — "unknown" qaytaradi

// Pitfall 2: reverse mapping
console.log(Direction[99]); // undefined
// TS xato bermaydi — runtime'da silent undefined

// Pitfall 3: Object.keys chalkashlik
Object.keys(Direction);
// ["0", "1", "2", "3", "Up", "Down", "Left", "Right"]
// 4 member — 8 key!

// To'g'ri iterate — filter bilan
const directionNames = Object.keys(Direction)
  .filter(k => isNaN(Number(k)));
// ["Up", "Down", "Left", "Right"]

const directionValues = Object.values(Direction)
  .filter(v => typeof v === "number") as number[];
// [0, 1, 2, 3]
```

String enum — pitfall'larning aksariyati yo'q:

```typescript
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

// Pitfall 1 yo'q — string literal assign bo'lmaydi
let c: Color = "RED"; // ❌ Xato

// Pitfall 3 yo'q — Object.keys toza
Object.keys(Color); // ["Red", "Green", "Blue"]

// Lekin: Pitfall 4/5 — const enum va erasableSyntaxOnly muammolari bir xil
```

Alternativa — as const object (pitfall'lar yo'q):

```typescript
const Direction = {
  Up: 0,
  Down: 1,
  Left: 2,
  Right: 3,
} as const;

type Direction = typeof Direction[keyof typeof Direction];
// type: 0 | 1 | 2 | 3

// Pitfall yo'q
// let d: Direction = 99; // ❌ 99 assign bo'lmaydi (nominal yo'q, lekin literal union)
Object.keys(Direction); // ["Up", "Down", "Left", "Right"] — toza
Object.values(Direction); // [0, 1, 2, 3] — toza
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source — pitfall misoli
enum Direction { Up, Down, Left, Right }
let dir: Direction = 99; // TS ruxsat beradi
console.log(Direction[99]); // undefined
console.log(Object.keys(Direction));
```

```javascript
// Compiled JS
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
  Direction[Direction["Left"] = 2] = "Left";
  Direction[Direction["Right"] = 3] = "Right";
})(Direction || (Direction = {}));
let dir = 99; // TS tomonidan tekshirilmagan
console.log(Direction[99]); // undefined — runtime'da silent xato
console.log(Object.keys(Direction));
// Output: ["0", "1", "2", "3", "Up", "Down", "Left", "Right"]
```

</details>

---

## Enum Alternativalari — Union Literal Types va as const

### Nazariya

Zamonaviy TypeScript'da enum o'rniga **union literal types** va **`as const` objects** ko'proq ishlatilmoqda. Bu pattern'lar enum'ning barcha pitfall'laridan xoli va zamonaviy toolchain (esbuild, SWC, Bun, native Node.js TS support) bilan to'liq mos.

### Pattern 1: Union Literal Type

```typescript
// Enum o'rniga
// enum Direction { Up = "UP", Down = "DOWN", Left = "LEFT", Right = "RIGHT" }

// Union literal type
type Direction = "UP" | "DOWN" | "LEFT" | "RIGHT";

function move(dir: Direction): void {
  console.log(`Moving ${dir}`);
}

move("UP");    // ✅
// move("DIAGONAL"); // ❌ Type '"DIAGONAL"' is not assignable
```

**Afzalliklari:**
- ✅ **Bundle size 0** — type erasure, JS'ga hech narsa tushmaydi
- ✅ **Tree shaking** — muammo yo'q (hech qanday runtime kod yo'q)
- ✅ **isolatedModules mos** — oddiy type declaration
- ✅ **erasableSyntaxOnly mos** — pure type
- ✅ **Aniq qiymat** — numeric enum pitfall'lar yo'q
- ❌ **Runtime list yo'q** — `Object.values(Direction)` mumkin emas

### Pattern 2: as const Object

Runtime'da list/iterate kerak bo'lganda:

```typescript
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

// Union type yaratish
type Direction = typeof Direction[keyof typeof Direction];
// type: "UP" | "DOWN" | "LEFT" | "RIGHT"

// Runtime va compile-time
console.log(Direction.Up);           // "UP" — runtime
move(Direction.Up);                   // type safety — compile-time
Object.values(Direction);             // ["UP", "DOWN", "LEFT", "RIGHT"]
```

**Afzalliklari:**
- ✅ **Runtime object** (enum kabi)
- ✅ **Kichik bundle** — oddiy object literal (IIFE yo'q)
- ✅ **Tree shaking** — yaxshi
- ✅ **isolatedModules / erasableSyntaxOnly mos**
- ✅ **Iterate qulay** — `Object.keys`/`values` toza
- ❌ **Declaration merging yo'q** (enum bilan mumkin)

Ikki pattern kombinatsiyasi — enum'ga eng yaqin replacement:

```typescript
const STATUS = {
  Active: "active",
  Inactive: "inactive",
  Pending: "pending",
} as const;

type Status = typeof STATUS[keyof typeof STATUS];
// type: "active" | "inactive" | "pending"

// Runtime: STATUS.Active, Object.values(STATUS)
// Compile-time: Status (type-safe parameter)
// Zero-cost enum replacement
```

<details>
<summary><strong>Under the Hood</strong></summary>

Union type va `as const` object enum'dan farqli ravishda compiler'da qayta ishlanadi:

**Union literal type:**

```typescript
type Direction = "UP" | "DOWN" | "LEFT" | "RIGHT";
```

Compiler buni `UnionType` node sifatida saqlaydi, `types: [StringLiteralType, StringLiteralType, ...]` bilan. Bu pure type-level konstruktsiya — checker `TypeFlags.Union | TypeFlags.StringLiteral` flag'lari bilan track qiladi.

Compile vaqtida emit qilinmaydi — `tsc` type erasure orqali butunlay o'chiradi. Runtime'da hech qanday iz qolmaydi — `Direction` nomli qiymat JS'da mavjud emas.

**`as const` object:**

```typescript
const Direction = { Up: "UP", Down: "DOWN" } as const;
```

Compiler bu yerda oddiy `ObjectLiteralExpression` node'ni ishlatadi, keyin `as const` assertion'ni qo'llab, har property'ga `readonly` va literal type flag'larini qo'shadi. Emit vaqtida `as const` marker o'chiriladi, oddiy object literal JS'ga tushadi:

```javascript
const Direction = { Up: "UP", Down: "DOWN" }; // oddiy object
```

Hech qanday IIFE yo'q, wrapper yo'q, reverse mapping yo'q — minimal kod.

**`typeof X[keyof typeof X]` — pattern tahlili:**

Bu ikki pog'onali type ekstraktsiya:

1. `typeof Direction` → `{ readonly Up: "UP"; readonly Down: "DOWN"; ... }` (object type)
2. `keyof typeof Direction` → `"Up" | "Down" | "Left" | "Right"` (key'lar union'i)
3. `typeof Direction[keyof typeof Direction]` → `"UP" | "DOWN" | "LEFT" | "RIGHT"` (value'lar union'i)

Oxirgi qadamda TypeScript **indexed access type**'ni ishlatadi — union'dagi har key uchun qiymat tipini olib, ularning union'ini qaytaradi.

Bu pattern'ni soddalashtirish uchun yordamchi type ham yozish mumkin:

```typescript
type ValueOf<T> = T[keyof T];

const Direction = { Up: "UP", Down: "DOWN" } as const;
type Direction = ValueOf<typeof Direction>;
// "UP" | "DOWN"
```

**Tree shaking farqi:**

Bundler (webpack, esbuild, Rollup) enum IIFE'ni **side-effect** sifatida ko'radi — chunki `(function() { ... })()` immediately invoked. Agar enum ishlatilmasa ham, bundler uni o'chirishdan ehtiyot bo'ladi (IIFE boshqa side-effect qilishi mumkin deb o'ylaydi).

`as const` object — oddiy `const` declaration. Bundler unchalik ehtiyot bo'lmaydi — agar ishlatilmasa, osonlik bilan o'chiradi. Shuning uchun `as const` tree shaking uchun yaxshiroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Union literal type — ko'p hollarda eng yaxshi tanlov:

```typescript
// Simple union
type LogLevel = "debug" | "info" | "warn" | "error";

function log(level: LogLevel, msg: string) {
  console.log(`[${level.toUpperCase()}] ${msg}`);
}

log("info", "Server started");
// log("critical", "error"); // ❌

// Discriminated union — literal type bilan
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "rectangle": return s.width * s.height;
    case "triangle": return (s.base * s.height) / 2;
  }
}
```

`as const` object — runtime va compile-time:

```typescript
// Action types — Redux pattern
const ACTIONS = {
  INCREMENT: "counter/increment",
  DECREMENT: "counter/decrement",
  RESET: "counter/reset",
} as const;

type ActionType = typeof ACTIONS[keyof typeof ACTIONS];
// type: "counter/increment" | "counter/decrement" | "counter/reset"

// Runtime
const allActions = Object.values(ACTIONS);
// ["counter/increment", "counter/decrement", "counter/reset"]

// Type-safe dispatch
function dispatch(type: ActionType) { /* ... */ }

dispatch(ACTIONS.INCREMENT);         // ✅
dispatch("counter/increment");       // ✅ literal union'ga mos
// dispatch("counter/unknown");      // ❌

// Reverse lookup — kerak bo'lsa
function getName(value: ActionType): string | undefined {
  return Object.entries(ACTIONS).find(([, v]) => v === value)?.[0];
}
```

Const array — eng oddiy pattern:

```typescript
// Array + index access
const SIZES = ["xs", "sm", "md", "lg", "xl"] as const;
type Size = typeof SIZES[number];
// type: "xs" | "sm" | "md" | "lg" | "xl"

// Iterate
SIZES.forEach(size => console.log(size));

// Validation
function isValidSize(value: string): value is Size {
  return (SIZES as readonly string[]).includes(value);
}

// Type-safe usage
function getButtonClass(size: Size): string {
  return `btn-${size}`;
}
```

Enum vs alternative compile solishtirish:

```typescript
// TS source 1 — enum
enum Status1 {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}

// TS source 2 — union
type Status2 = "ACTIVE" | "INACTIVE";

// TS source 3 — as const
const STATUS3 = {
  Active: "ACTIVE",
  Inactive: "INACTIVE",
} as const;
type Status3 = typeof STATUS3[keyof typeof STATUS3];
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Uch yondashuv JS'ga qanday compile bo'ladi:

```javascript
// 1. Enum — IIFE (eng katta output)
var Status1;
(function (Status1) {
  Status1["Active"] = "ACTIVE";
  Status1["Inactive"] = "INACTIVE";
})(Status1 || (Status1 = {}));

// 2. Union type — 0 bytes (type erasure)
// (JS'da hech narsa yo'q — butunlay yo'qoldi)

// 3. as const object — oddiy object
const STATUS3 = {
  Active: "ACTIVE",
  Inactive: "INACTIVE",
};
// Type Status3 — yo'q (type erasure)
```

Bundle size solishtirish:
- **Enum:** IIFE + wrapper + property assignment (kattaroq)
- **Union:** 0 bayt (pure type)
- **as const:** oddiy object literal (enum'dan kichikroq, wrapper yo'q)

`as const` — enum'ga eng yaqin semantik, lekin sezilarli darajada kichikroq va bundler-friendly.

</details>

---

## Decision Matrix — Enum vs Union vs as const

### Nazariya

Qaysi yondashuv qachon afzalroq? Kontekstga qarab qaror qilinadi:

| Kriteriya | Enum | Union Type | as const Object |
|-----------|------|------------|-----------------|
| Runtime object kerak | ✅ Bor | ❌ Yo'q | ✅ Bor |
| Bundle size | ⚠️ Kattaroq (IIFE) | ✅ 0 | ✅ Kichik (oddiy object) |
| Tree shaking | ❌ Qiyin | ✅ Muammo yo'q | ✅ Yaxshi |
| `isolatedModules` | ⚠️ `const enum` ishlamaydi | ✅ Mos | ✅ Mos |
| `erasableSyntaxOnly` | ❌ Mos emas | ✅ Mos | ✅ Mos |
| Iterate (`Object.keys`) | ⚠️ Reverse mapping chalkash | ❌ Mumkin emas | ✅ Toza |
| Numeric bit flags | ✅ Yaxshi | ⚠️ Qiyin | ⚠️ Qiyin |
| Declaration merging | ✅ Mumkin | ❌ Yo'q | ❌ Yo'q |
| Reverse mapping | ✅ Numeric da | ❌ Yo'q | ❌ Yo'q |
| Namespace augmentation | ✅ Mumkin | ❌ Yo'q | ❌ Yo'q |
| Browser ecosystem | ⚠️ Bundler'ga bog'liq | ✅ Universal | ✅ Universal |
| Debugging | ✅ Toza (enum nomi) | ⚠️ Literal value | ✅ Object nomi |

**Xulosa (tavsiya):**

- **Union type** — ko'p hollarda **eng yaxshi** tanlov. Sodda, xavfsiz, bundle size 0, barcha zamonaviy toolchain'lar bilan mos.
- **`as const` object** — runtime'da iterate, validate yoki reverse lookup kerak bo'lganda. Enum'ga eng yaqin alternativa.
- **Enum** — faqat uch holat'da:
  1. **Legacy code** — eski kod base'da enum'lar mavjud va butunlay migratsiya qilish qiyin
  2. **Framework convention** — Angular, NestJS'da enum community pattern
  3. **Bit flags** — bitwise OR bilan kombinatsiya (masalan, `FilePermission.Read | FilePermission.Write`)

Yangi loyihada enum'dan voz kechish — zamonaviy TypeScript'ning asosiy tavsiyalaridan biri.

<details>
<summary><strong>Under the Hood</strong></summary>

Uch yondashuvning compiler va bundler'da qanday qayta ishlanishi:

**1. Enum — maxsus kompilyator mantig'i:**
- Emit: IIFE + object literal + reverse mapping logic (numeric bo'lsa)
- Bundler: side-effect (immediately invoked) — tree shake qiyin
- Runtime: object property lookup (`O(1)`, lekin indirection qatlami bor)

**2. Union type — pure type:**
- Emit: 0 bayt (type erasure — butunlay o'chiriladi)
- Bundler: hech qanday ta'sir (JS'da iz yo'q)
- Runtime: literal qiymatlar inline — hech qanday lookup

**3. `as const` object — oddiy literal:**
- Emit: oddiy `const` declaration + object literal
- Bundler: declaration optimizable — tree shake oson
- Runtime: object property lookup (enum bilan bir xil)

**Declaration merging — enum'ning noyob xususiyati:**

Faqat enum'da ishlaydi — chunki compiler `EnumDeclaration` node'larni same-named enum'lar uchun **merge** qilish logikasiga ega. Bu `||` operator pattern orqali (`(Dir || (Dir = {}))`) amalga oshadi. Union type va `as const` object'lar uchun bunday merging mexanizmi yo'q — ular oddiy type/value declaration.

```typescript
// Enum — faqat shu merge bo'ladi
enum Dir { Up }
enum Dir { Down }
// Dir = { Up: 0, 0: "Up", Down: 1, 1: "Down" }

// Union — merge yo'q
type Dir1 = "up";
type Dir1 = "down"; // ❌ Duplicate identifier

// as const — merge yo'q
const Dir2 = { Up: 0 } as const;
const Dir2 = { Down: 1 } as const; // ❌ Cannot redeclare
```

**`erasableSyntaxOnly` flag bilan mos keluvchilik:**

TS 5.8'dan boshlab Node.js native TS support (`--experimental-strip-types`) uchun shu flag mavjud. U faqat **type erasure** qilinadigan syntax'ni ruxsat etadi:

- ✅ Union type — pure type erasure
- ✅ `as const` — `as const` marker o'chiriladi, oddiy JS qoladi
- ❌ Enum — IIFE hosil qiladi (erasure emas, transformation)
- ❌ `const enum` — inline substitution (erasure emas)
- ❌ `namespace` — IIFE
- ❌ `experimentalDecorators` (legacy) — wrapper funksiyalar
- ❌ Constructor parameter properties (`constructor(public x: number)`) — assignment kerak

Bu flag zamonaviy TypeScript yo'nalishini ko'rsatadi — **pure type erasure**, runtime transformation minimal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Decision matrix amaliy qo'llash:

```typescript
// Case 1: Simple string constants — union type
type ButtonVariant = "primary" | "secondary" | "danger";
function Button({ variant }: { variant: ButtonVariant }) { /* ... */ }
// ✅ Eng yaxshi: bundle 0, sodda

// Case 2: Runtime iterate kerak — as const
const THEME_COLORS = {
  primary: "#007bff",
  secondary: "#6c757d",
  danger: "#dc3545",
} as const;
type ThemeColor = keyof typeof THEME_COLORS;
// Iterate mumkin:
for (const color in THEME_COLORS) { /* ... */ }
// ✅ Runtime + type-safe

// Case 3: Bit flags — enum
enum Permission {
  None = 0,
  Read = 1 << 0,
  Write = 1 << 1,
  Execute = 1 << 2,
  All = Read | Write | Execute,
}

function hasPermission(user: Permission, required: Permission): boolean {
  return (user & required) === required;
}

hasPermission(Permission.All, Permission.Read); // true
// ✅ Enum bit flags uchun toza

// Case 4: Validation with runtime list — as const array
const VALID_CURRENCIES = ["USD", "EUR", "GBP", "JPY"] as const;
type Currency = typeof VALID_CURRENCIES[number];

function isValidCurrency(code: string): code is Currency {
  return (VALID_CURRENCIES as readonly string[]).includes(code);
}
// ✅ Runtime validate + compile-time type
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS — uchta yondashuv
type Status1 = "active" | "inactive";
const STATUS2 = { Active: "active", Inactive: "inactive" } as const;
enum Status3 { Active = "active", Inactive = "inactive" }
```

```javascript
// JS compile natijasi
// Status1 — YO'Q (pure type erasure)

// STATUS2 — oddiy object
const STATUS2 = { Active: "active", Inactive: "inactive" };

// Status3 — IIFE
var Status3;
(function (Status3) {
  Status3["Active"] = "active";
  Status3["Inactive"] = "inactive";
})(Status3 || (Status3 = {}));
```

Hajm va runtime overhead solishtirish: union type < `as const` object < enum (IIFE wrapper overhead tufayli).

</details>

---

## satisfies bilan Validation

### Nazariya

`satisfies` operator (TS 4.9+) — qiymatning tipini tekshiradi, lekin tipni **kengaytirmaydi**. Bu enum va union type'lar bilan ishlaganda foydali — har qiymatning aniq tipi saqlanadi.

```typescript
type Color = "red" | "green" | "blue";

// ❌ Type annotation — literal type yo'qoladi
const palette: Record<Color, string> = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
};
// palette.red type: string (literal emas)

// ✅ satisfies — tip tekshiriladi, literal type saqlanadi
const palette2 = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
} satisfies Record<Color, string>;
// palette2.red type: "#ff0000" (literal type saqlandi)
```

`satisfies` asosiy foydasi:
- **Type check qiladi** — object shape'ini belgilangan tipga taqqoslaydi
- **Literal type saqlaydi** — `red` property'si `"#ff0000"` literal, `string` emas
- **Yetishmagan key'lar uchun xato** — agar biror `Color` member tushirib qoldirilsa

**Qo'llanish:** enum alternativalarida keng foydalaniladi — `as const` object'ni validate qilish uchun.

> **Batafsil:** `satisfies` operator haqida to'liq ma'lumot (`as const`, type annotation, `satisfies` farqi, narrowing effekti) [06-type-narrowing.md](06-type-narrowing.md)'da yoritiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Enum replacement — satisfies bilan
type Status = "active" | "inactive" | "pending";

const STATUS_LABELS = {
  active: "User is active",
  inactive: "User is inactive",
  pending: "Waiting for activation",
} satisfies Record<Status, string>;

// STATUS_LABELS.active type: "User is active" (literal)
// Agar biror status tushirib qoldirilsa:
// const bad = {
//   active: "...",
//   inactive: "...",
//   // pending yo'q
// } satisfies Record<Status, string>;
// ❌ Property 'pending' is missing in type

// Validation pattern
const ROUTES = {
  home: "/",
  about: "/about",
  contact: "/contact",
} satisfies Record<string, `/${string}`>;
// Har value `/${string}` template pattern'iga mos bo'lishi kerak
// ROUTES.home type: "/" (literal saqlanadi)
```

</details>

---

## Edge Cases va Gotchas

Array, tuple va enum bilan ishlashda uchraydigan nozik holatlar. Bular ko'pincha production kodda silent xato sifatida namoyon bo'ladi.

### 🕳 Gotcha 1: Tuple `push` length mismatch

```typescript
let pair: [string, number] = ["hello", 42];

// Push ishlaydi — TS to'xtatmaydi
pair.push("extra");
console.log(pair);        // Runtime: ["hello", 42, "extra"]
console.log(pair.length); // Runtime: 3

// Lekin TS hali ham length 2 deb hisoblaydi
// pair[2]; // ❌ TS: Tuple type has no element at index 2
// (runtime'da pair[2] === "extra" lekin TS ko'rmaydi)

// Yechim: readonly tuple
let pair2: readonly [string, number] = ["hello", 42];
// pair2.push("extra"); // ❌ Property 'push' does not exist
```

**Sabab:** Tuple `[string, number]` uchun `push` method signature element tiplarining union'i (`string | number`)'ni qabul qiladi. TypeScript runtime'da tuple uzunligini track qilolmaydi — shuning uchun `push`'dan keyin length mismatch yuzaga keladi. Bu "silent" xato — TS compile-time'da ko'rmaydi, runtime'da mavjud.

---

### 🕳 Gotcha 2: Numeric enum + `Object.keys()` ikki barobar key

```typescript
enum Direction { Up, Down, Left, Right }

Object.keys(Direction);
// ["0", "1", "2", "3", "Up", "Down", "Left", "Right"]
// 4 member — 8 key!

for (const key in Direction) {
  console.log(key);
  // "0", "1", "2", "3", "Up", "Down", "Left", "Right"
}

// Faqat nomlarni olish — filter
const names = Object.keys(Direction).filter(k => isNaN(Number(k)));
// ["Up", "Down", "Left", "Right"]
```

**Sabab:** Numeric enum'da **reverse mapping** — `Direction[0] === "Up"` va `Direction["Up"] === 0`. Object har ikkala yo'nalish uchun ham key saqlaydi, shuning uchun N member uchun 2N key. String enum'da bu muammo yo'q (reverse mapping yo'q).

---

### 🕳 Gotcha 3: `readonly T[]` va `Readonly<T[]>` farqi

```typescript
// Ikkalasi — readonly array, lekin internal representation farqli
type A = readonly string[];
type B = Readonly<string[]>;

// A: readonly string[] (native readonly modifier)
// B: readonly string[] (Readonly<T> utility type bilan)

// Ko'pincha bir xil, lekin generic constraint'larda farq bo'lishi mumkin
function test<T>(arr: T) {
  // T extends Readonly<any[]> — check bir xil
  // T extends readonly any[] — sintaktik farq
}

// MUHIM: nested readonly farq qiladi
type NestedA = readonly string[][];     // inner array mutable
type NestedB = Readonly<string[][]>;    // outer readonly, inner mutable
type NestedC = ReadonlyArray<ReadonlyArray<string>>; // ikkala daraja readonly

// Deep readonly kerak bo'lsa:
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

**Sabab:** `readonly T[]` va `Readonly<T[]>` sintaktik farqli lekin semantic bir xil — ikkalasi ham array'ning o'zini readonly qiladi. Lekin **nested** array yoki object'larda muhim farq bor: `readonly T[]` faqat tashqi darajani, `Readonly<T>` utility type esa faqat property darajasini ta'sirlaydi. Deep readonly uchun recursive utility type kerak.

---

### 🕳 Gotcha 4: Bo'sh array `const` va `never[]`

```typescript
// Bo'sh array const — contextual bo'lmasa muammo
const empty = []; // type: any[] (strict'da ham evolving)
empty.push(1); // type: number[] (evolved)

// Lekin function return'ida:
function getList() {
  return []; // return type: any[]
}

// Generic function da — `never[]` bo'lishi mumkin
function firstOrDefault<T>(arr: T[]): T | undefined {
  return arr[0];
}

const result = firstOrDefault([]); // result: undefined
// T inferred as unknown (TS 4.7+) or never (TS older)

// Aniq annotation yozish tavsiya:
const empty2: number[] = [];
const result2 = firstOrDefault<number>([]); // aniq generic
```

**Sabab:** Bo'sh array literal (`[]`) uchun TypeScript element tipini aniqlay olmaydi. Oddiy `let`/`const` da `any[]` yoki evolving array qo'yadi. Generic funksiya contekstida esa `never[]` yoki `unknown[]` infer qilishi mumkin — bu silent bug'larga olib keladi. Aniq type annotation har doim xavfsizroq.

---

### 🕳 Gotcha 5: Enum vs union type — `JSON.stringify` natijasi farqli

```typescript
// Enum — runtime object
enum Status1 {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}

JSON.stringify({ status: Status1.Active });
// {"status":"ACTIVE"} — enum qiymati saqlanadi

// Numeric enum — son saqlanadi
enum Direction { Up, Down, Left, Right }
JSON.stringify({ dir: Direction.Up });
// {"dir":0} — son, nom emas

// Union type — qiymat bevosita
type Status2 = "active" | "inactive";
const s: Status2 = "active";
JSON.stringify({ status: s });
// {"status":"active"} — literal qiymat
```

**Gotcha:** Numeric enum bilan JSON serialization'da **son** saqlanadi, **nom** emas. Frontend va backend'da enum member'lar mos kelmasa (masalan, versiyalar farqi) — silent bug. Yechim: string enum yoki union type ishlatish — ular runtime qiymati ham ma'nosi ham aniq.

---

## Common Mistakes

### ❌ Xato 1: Tuple kerak joyga array berish

```typescript
function setPoint(point: [number, number]): void { /* ... */ }

// ❌ number[] → [number, number] ga assign bo'lmaydi
const coords: number[] = [10, 20];
setPoint(coords);
// Error: Type 'number[]' is not assignable to type '[number, number]'

// ✅ Tuple sifatida e'lon
const coords2: [number, number] = [10, 20];
setPoint(coords2); // ✅

// ✅ Yoki as const (readonly tuple)
const coords3 = [10, 20] as const; // readonly [10, 20]
// setPoint(coords3); // ❌ readonly → mutable xato
// Yechim: function parametrini readonly qilish
function setPoint2(point: readonly [number, number]): void { /* ... */ }
setPoint2(coords3); // ✅
```

**Nima uchun:** `number[]` uzunligi noma'lum (0, 1, 100 element bo'lishi mumkin), `[number, number]` aniq 2 element. TypeScript bu ikkalasi orasida type safety ta'minlaydi — xato oldini oladi.

---

### ❌ Xato 2: Numeric enum'ga ixtiyoriy son

```typescript
enum Direction { Up, Down, Left, Right }

// ❌ Xavfli — 99 Direction'da yo'q
let dir: Direction = 99; // TS hech narsa demaydi!

function handleDirection(dir: Direction) {
  switch (dir) {
    case Direction.Up: /* ... */ break;
    case Direction.Down: /* ... */ break;
    case Direction.Left: /* ... */ break;
    case Direction.Right: /* ... */ break;
    // dir = 99 bo'lsa — hech qanday case'ga tushmaydi
  }
}

handleDirection(99); // ⚠️ Silent xato

// ✅ String enum yoki union type
type Direction2 = "UP" | "DOWN" | "LEFT" | "RIGHT";
let dir2: Direction2 = "RANDOM"; // ❌ Xato beradi
```

**Nima uchun:** Numeric enum bit flags pattern'ini qo'llab-quvvatlash uchun har qanday `number`'ni qabul qiladi. Bu ataylab qilingan "intentional unsoundness". Agar bit flags kerak bo'lmasa — string enum yoki union type ishlating.

---

### ❌ Xato 3: `const enum` ni `isolatedModules` bilan ishlatish

```typescript
// types.ts
export const enum Theme {
  Light = "LIGHT",
  Dark = "DARK",
}

// app.ts — isolatedModules: true
import { Theme } from "./types";
// ❌ TS2748: Cannot access ambient const enums when '--isolatedModules' is enabled

// ✅ Yechim 1 — oddiy enum
export enum Theme2 { Light = "LIGHT", Dark = "DARK" }

// ✅ Yechim 2 — as const (tavsiya)
export const THEME = {
  Light: "LIGHT",
  Dark: "DARK",
} as const;
export type Theme3 = typeof THEME[keyof typeof THEME];
```

**Nima uchun:** `isolatedModules: true` har faylni alohida compile qiladi (esbuild, SWC stilida). `const enum` boshqa fayldan import qilinganda inline substitution kerak — lekin alohida compile'da bu imkonsiz.

---

### ❌ Xato 4: `readonly` array'ni mutable funksiyaga berish

```typescript
const names: readonly string[] = ["Ali", "Vali"];

function sortNames(arr: string[]): string[] {
  return arr.sort(); // sort() array'ni mutates qiladi
}

sortNames(names);
// ❌ Type 'readonly string[]' is not assignable to type 'string[]'

// ✅ Yechim 1 — funksiya parametrini readonly
function sortNames2(arr: readonly string[]): string[] {
  return [...arr].sort(); // spread bilan copy, keyin sort
}

// ✅ Yechim 2 — caller'da copy
sortNames([...names]); // mutable copy yaratish
```

**Nima uchun:** `readonly string[]` → `string[]`'ga assign mumkin emas — chunki mutable funksiya original array'ni o'zgartirishi mumkin. TS bu xavfni compile-time'da to'xtatadi. Best practice: **input parameter'lar doim `readonly`**, ichki mutation uchun spread copy ishlating.

---

### ❌ Xato 5: Tuple push muammosini bilmaslik

```typescript
let pair: [string, number] = ["hello", 42];

// ⚠️ TS to'xtatmaydi — xavfli pattern
pair.push("extra");
console.log(pair); // ["hello", 42, "extra"] — runtime length 3
// pair[2]; // ❌ TS: tuple type has no element at index 2

// ✅ Yechim — readonly tuple
let pair2: readonly [string, number] = ["hello", 42];
// pair2.push("extra"); // ❌ Property 'push' does not exist
// pair2[0] = "world";  // ❌ Read-only
```

**Nima uchun:** TypeScript tuple'da `push` method'ini cheklay olmaydi — `Array.prototype.push` signature'i tuple'ga meros bo'ladi. Readonly tuple'da esa `ReadonlyArray` interface'idan meros bo'ladi, unda mutating method'lar yo'q.

---

## Amaliy Mashqlar

### Mashq 1: Array va Tuple tiplarini aniqlash (Oson)

**Savol:** Har o'zgaruvchining TypeScript tomonidan inferred tipini ayting:

```typescript
const a = [1, 2, 3];
const b = [1, "hello", true];
const c = [1, 2, 3] as const;
let d = [1, 2, 3];
const e: [number, string] = [1, "hello"];
const f = [];
```

<details>
<summary>Javob</summary>

```typescript
const a = [1, 2, 3];
// type: number[]

const b = [1, "hello", true];
// type: (string | number | boolean)[]

const c = [1, 2, 3] as const;
// type: readonly [1, 2, 3] — readonly tuple, har element literal

let d = [1, 2, 3];
// type: number[] — let/const farq yo'q array'da (faqat `as const` bilan farq bor)

const e: [number, string] = [1, "hello"];
// type: [number, string] — aniq annotation, tuple

const f = [];
// type: any[] — bo'sh array, annotation yo'q
// ⚠️ noImplicitAny: true bo'lsa ham TS bu yerda any[] qo'yadi
// Evolving: push bilan tipni kengaytirishi mumkin
```

**Tushuntirish:**
- `const` primitive'da literal type beradi, array'da property'lar widened
- `as const` — recursive: elementlar literal + tuple + readonly
- Bo'sh array annotation'siz — `any[]` (xavfli)

</details>

---

### Mashq 2: Enum Under the Hood (O'rta)

**Savol:** Quyidagi enum JavaScript'ga qanday compile bo'ladi? Natija object'ni to'liq yozing:

```typescript
enum Fruit {
  Apple,
  Banana = 5,
  Cherry,
}
```

<details>
<summary>Javob</summary>

```javascript
// Compiled JS
var Fruit;
(function (Fruit) {
  Fruit[Fruit["Apple"] = 0] = "Apple";
  Fruit[Fruit["Banana"] = 5] = "Banana";
  Fruit[Fruit["Cherry"] = 6] = "Cherry";
})(Fruit || (Fruit = {}));

// Natija object:
// Fruit = {
//   Apple: 0,
//   Banana: 5,
//   Cherry: 6,       // Banana + 1 = 6 (ketma-ket davom)
//   "0": "Apple",    // reverse mapping
//   "5": "Banana",
//   "6": "Cherry",
// }

// Tekshirish
console.log(Fruit.Apple);   // 0
console.log(Fruit.Banana);  // 5
console.log(Fruit.Cherry);  // 6
console.log(Fruit[0]);      // "Apple" (reverse)
console.log(Fruit[5]);      // "Banana"
console.log(Fruit[6]);      // "Cherry"
```

**Tushuntirish:**
- `Apple` — 0 (default)
- `Banana = 5` — aniq belgilangan
- `Cherry` — 6 (`Banana + 1`, ketma-ket davom)
- Numeric enum'da reverse mapping — sondan nomga qaytish
- `Fruit[Fruit["Apple"] = 0] = "Apple"` pattern — assignment natija pattern (assignment `0` qaytaradi, keyin `Fruit[0] = "Apple"`)

</details>

---

### Mashq 3: `as const` bilan union type yaratish (Qiyin)

**Savol:** Quyidagi kodda `HttpMethod` tipini yarating va `isValidMethod` funksiyasini implement qiling:

```typescript
const HTTP_METHODS = ["GET", "POST", "PUT", "DELETE", "PATCH"] as const;

type HttpMethod = /* ??? */;

function isValidMethod(method: string): method is HttpMethod {
  // TODO: implement
}
```

<details>
<summary>Javob</summary>

```typescript
const HTTP_METHODS = ["GET", "POST", "PUT", "DELETE", "PATCH"] as const;

// Union type — typeof + indexed access [number]
type HttpMethod = typeof HTTP_METHODS[number];
// type: "GET" | "POST" | "PUT" | "DELETE" | "PATCH"

// Type guard funksiya
function isValidMethod(method: string): method is HttpMethod {
  return (HTTP_METHODS as readonly string[]).includes(method);
  // Cast kerak: HTTP_METHODS: readonly ["GET", ...] — .includes() parametr tipi
  // literal union ("GET" | "POST" | ...), lekin `method: string`'ni qabul qilmaydi
}

// Ishlatish
const userInput: string = "GET";

if (isValidMethod(userInput)) {
  // Bu blokda: userInput: HttpMethod (narrowed)
  handleRequest(userInput); // ✅ type-safe
} else {
  console.log("Invalid HTTP method");
}

function handleRequest(method: HttpMethod) { /* ... */ }
```

**Tushuntirish:**
- `typeof HTTP_METHODS` → `readonly ["GET", "POST", ...]`
- `[number]` — tuple'ning barcha element tipi union'i
- Natija: `"GET" | "POST" | "PUT" | "DELETE" | "PATCH"`
- `.includes()` cast sababi: readonly tuple'ning `.includes` parametri literal union'ga cheklangan, `string` argument'ni qabul qilmaydi
- Type guard (`method is HttpMethod`) — narrowing uchun

</details>

---

## Xulosa

Bu bo'limda TypeScript'ning struktura tiplari bilan tanishdik:

- **Array Types** — `T[]` va `Array<T>` bir xil, convention bo'yicha `T[]` ko'proq ishlatiladi. Bo'sh array'ga annotation tavsiya.
- **Readonly Arrays** — `readonly T[]`, `ReadonlyArray<T>`, `Readonly<T[]>` — hammasi bir xil. Mutating method'lar taqiqlanadi. Faqat compile-time himoya. Funksiya parametrlari uchun best practice.
- **Tuples** — `[T, U]` fixed-length, har element alohida tip. `push` muammosi mavjud — yechim `readonly` tuple.
- **Optional Tuple** — `[T, U?]` faqat oxirida optional bo'ladi. Required after optional taqiqlangan.
- **Rest Tuple** — `[T, ...U[]]` va `[...T[], U]` (TS 4.2+). Variadic tuple types orqali generic inference (TS 4.0+).
- **Named Tuples** — `[x: number, y: number]` faqat documentation metadata. Runtime va type identity ta'sir yo'q.
- **`as const` tuple** — `readonly tuple` + literal types. `typeof arr[number]` bilan union type yaratish.
- **Numeric Enum** — IIFE + reverse mapping. Bit flags uchun mo'ljallangan, `number` bilan bidirectional assignability (unsoundness).
- **String Enum** — IIFE, reverse mapping yo'q. Nominal-like identity (string literal assign bo'lmaydi).
- **`const enum`** — inline substitution, bundle size 0. `isolatedModules` va `erasableSyntaxOnly` bilan muammo.
- **Enum Under the Hood** — `Direction[Direction["Up"] = 0] = "Up"` assignment pattern. Declaration merging `||` trick orqali.
- **Enum Pitfalls** — numeric'ga har qanday son, reverse mapping chalkashligi, `Object.keys` 2N key, `const enum` import muammosi, `erasableSyntaxOnly` mos emas.
- **Alternativalar** — **Union literal type** (ko'p hollarda eng yaxshi), **`as const` object** (runtime kerak bo'lsa).
- **Decision Matrix** — union type (eng yaxshi default) > `as const` object (runtime + iterate) > enum (bit flags, legacy, framework convention).
- **`satisfies`** — literal type'larni saqlab type check qilish. `as const` object bilan yaxshi juft.
- **Edge Cases** — tuple push mismatch, numeric enum Object.keys, `readonly` vs `Readonly<T>` nested farqi, bo'sh array widening, JSON.stringify enum.

Keyingi bo'limda TypeScript'ning **object va interface tiplari** — object type annotations, optional/readonly property'lar, index signatures, interface declaration, interface extending, interface merging, `interface` vs `type alias` farqi, excess property checking va recursive interface'lar bilan tanishamiz.

---

**Keyingi bo'lim:** [04-objects-interfaces.md](04-objects-interfaces.md) — Object Types va Interfaces: object type annotations, optional (`?`) va readonly property'lar, index signatures, `Record<K, V>` utility type, interface declaration va extending, declaration merging, interface vs type alias, excess property checking, recursive interfaces, `PropertyKey` type.
