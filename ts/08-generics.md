# Bo'lim 8: Generics — Type Parameters

> Generics — TypeScript'da type-level abstraction mexanizmi. Funksiya, interface yoki class yaratishda aniq type o'rniga **type parameter** qo'yiladi va u faqat ishlatilgan paytda aniq type'ga aylanadi. Bu compile-time'da to'liq type safety'ni saqlab, reusable kod yozish imkonini beradi.
>
> Generics — TypeScript type system'ning **fundamenti**. Keyingi bo'limlardagi Conditional Types, Mapped Types, Utility Types — barchasi generics ustiga quriladi. Bu bo'limda generic'larning asosiy konseptsiyalarini, built-in operator'larni (`keyof`, `typeof`, index access), va real-world pattern'larni ko'ramiz.

---

## Mundarija

- [Generics Nima](#generics-nima)
- [Generic Functions](#generic-functions)
- [Generic Inference](#generic-inference)
- [Explicit Type Arguments](#explicit-type-arguments)
- [Generic Interfaces](#generic-interfaces)
- [Generic Classes](#generic-classes)
- [Generic Constraints](#generic-constraints)
- [keyof Bilan Generics](#keyof-bilan-generics)
- [Index Access Types](#index-access-types)
- [typeof Type Operator](#typeof-type-operator)
- [Multiple Type Parameters](#multiple-type-parameters)
- [Default Type Parameters](#default-type-parameters)
- [const Type Parameters](#const-type-parameters)
- [Generic Utility Patterns](#generic-utility-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Generics Nima

### Nazariya

Generics — type system'da **parametrizatsiya** mexanizmi. Oddiy funksiya qiymat (value) bilan ishlaydi — generics esa **type** bilan ishlaydi. `<T>` — bu type parameter, ya'ni "qaysi type kelishini hali bilmaymiz, lekin u kelganda barcha joyda bir xil type ishlatiladi" degan ma'no beradi.

Generics ikkita muammoni hal qiladi:

**1-muammo — Type safety yo'qolishi.** `any` ishlatilsa, type tekshiruv butunlay o'chiriladi:

```typescript
function firstElement(arr: any[]): any {
  return arr[0];
}

const result = firstElement([1, 2, 3]);
// result: any — TS hech narsa bilmaydi
// result.toUpperCase() — xato bo'lmaydi, lekin runtime'da crash
```

**2-muammo — Kod takrorlanishi.** Har type uchun alohida funksiya yozish (DRY buziladi):

```typescript
function firstString(arr: string[]): string { return arr[0]; }
function firstNumber(arr: number[]): number { return arr[0]; }
function firstUser(arr: User[]): User { return arr[0]; }
// Cheksiz funksiyalar...
```

**Generics ikki muammoni bir vaqtda hal qiladi:**

```typescript
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const str = firstElement(["a", "b"]);           // string | undefined
const num = firstElement([1, 2, 3]);             // number | undefined
const user = firstElement([{ name: "Ali" }]);    // { name: string } | undefined
```

Bitta funksiya — barcha type'lar uchun — to'liq type safety bilan.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori generic'larni quyidagi bosqichlarda qayta ishlaydi:

1. **Parsing** — `<T>` type parameter sifatida AST'ga qo'shiladi (`TypeParameterDeclaration` node)
2. **Type checking** — har chaqiruv joyida `T`'ning aniq type'ga **instantiation** qilinadi, argumentlardan inference yoki explicit berilgan type ishlatiladi
3. **Erasure** — JS'ga compile paytida `<T>` va barcha type annotation'lar butunlay o'chiriladi

```
TS Source:  function id<T>(x: T): T { return x; }
                         ↓ Type Check
Instantiation:  id<number>(42) → T = number → x: number, return: number ✅
                id<string>("a") → T = string → x: string, return: string ✅
                         ↓ Emit
JS Output:  function id(x) { return x; }   ← <T> yo'q
```

**Type parameter scope** — `<T>` funksiya, interface yoki class scope'ida mavjud. Har bir chaqiruv (call site) da `T` mustaqil instantiate bo'ladi — bir joyda `T = string` bo'lishi boshqa joyga ta'sir qilmaydi.

```
Generic Instantiation Flow:

  Source:  function id<T>(x: T): T

  ┌─────────────┐
  │ id<number>  │───── T = number ─────▶ x: number, return: number
  │   (42)      │
  └─────────────┘

  ┌─────────────┐
  │ id<string>  │───── T = string ─────▶ x: string, return: string
  │   ("hi")    │
  └─────────────┘

  ┌─────────────┐
  │ id(true)    │───── T = boolean ─────▶ x: boolean, return: boolean
  │ (inferred)  │     (argument'dan)
  └─────────────┘
                         │
                         ▼
                  JS Output:
             function id(x) { return x; }
             ← runtime'da generic iz yo'q
```

**Instantiation lazy**: kompilator barcha mumkin bo'lgan `T`'lar uchun oldindan tekshirmaydi — faqat haqiqiy call site'lardagi aniq type'lar uchun. Shuning uchun `id<number>` va `id<string>` ikki mustaqil instantiation — bir-biriga ta'sir qilmaydi.

**Runtime'da farq yo'q:** Generic funksiya compiled JS'da oddiy funksiya bo'lib qoladi. `instanceof`, `typeof`, reflect — hech biri type parameter'ni ko'ra olmaydi. Type safety faqat compile-time'da amal qiladi — shuning uchun external data (API response, JSON) bilan ishlashda runtime validation kerak bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy generic funksiya
function identity<T>(value: T): T {
  return value;
}

const a = identity("hello"); // string
const b = identity(42);       // number
const c = identity(true);     // boolean

// 2. Array bilan generic
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

firstElement(["a", "b", "c"]); // string | undefined
firstElement([1, 2, 3]);        // number | undefined

// 3. Transform pattern
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

pair("name", 25);      // [string, number]
pair(true, [1, 2, 3]); // [boolean, number[]]

// 4. Generic class
class Box<T> {
  constructor(public value: T) {}
  getValue(): T { return this.value; }
}

const strBox = new Box("hello"); // Box<string>
const numBox = new Box(42);       // Box<number>

// 5. Generic interface
interface Repository<T> {
  find(id: string): T | null;
  save(entity: T): void;
}
```

</details>

---

## Generic Functions

### Nazariya

Generic function — type parameter qabul qiladigan funksiya. Sintaksis: funksiya nomidan keyin `<T>` qo'yiladi. `T` — convention bo'yicha bitta harf (Type), lekin istalgan nom bo'lishi mumkin.

```
function functionName<TypeParam>(param: TypeParam): ReturnType { ... }
```

**Convention'lar — type parameter nomlari:**

| Nom | Ma'nosi | Qachon ishlatiladi |
|-----|---------|---------------------|
| `T` | Type (umumiy) | Bitta type parameter bo'lganda |
| `U`, `V` | Qo'shimcha type'lar | Ikkinchi, uchinchi parameter |
| `K` | Key | Object key type |
| `V` | Value | Object value type |
| `E` | Element | Array element |
| `R` | Return | Return type |
| `P` | Props (React) | Component props |

Function declaration, function expression va arrow funksiyalar — uchala shaklda ham generic yoziladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator generic funksiyani parse qilganda, `<T>` type parameter list `typeParameters` property'da saqlanadi — value parameter'lardan butunlay alohida AST branch.

Har call site'da kompilator `checker.ts` ichida **type parameter instantiation** jarayonini boshlaydi. `identity<number>(42)` chaqirilganda yangi type scope yaratiladi va `T = number` binding qo'yiladi. Signature'dagi barcha `T` o'rniga `number` qo'yilib tekshiriladi — parameter type, return type, body ichidagi operatsiyalar.

**Instantiation har call uchun alohida:** `identity<number>` va `identity<string>` — ikki mustaqil instantiation, bir-biriga ta'sir qilmaydi. Har chaqiriq o'z scope'ida T'ni boshqa type bilan bog'laydi.

**Emit bosqichida type parameter o'chiriladi:** `function identity<T>(value: T): T` → `function identity(value) { return value; }`. Runtime'da generic va non-generic funksiya orasida hech qanday farq yo'q.

**Arrow function + `<T>` vs JSX konflikti:** `.tsx` faylda `<T>` JSX tag sifatida parse qilinishi mumkin. Yechim: `<T,>` (trailing comma) yoki `<T extends unknown>` — bu faqat parser uchun disambiguation, emit natijasi bir xil:

```typescript
// .tsx faylda xato
// const fn = <T>(x: T) => x; // Error: JSX tag expected

// ✅ Yechim 1: trailing comma
const fn1 = <T,>(x: T) => x;

// ✅ Yechim 2: extends
const fn2 = <T extends unknown>(x: T) => x;
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Function declaration
function identity<T>(value: T): T {
  return value;
}

// 2. Function expression
const wrapInArray = function <T>(value: T): T[] {
  return [value];
};

// 3. Arrow function
const getFirst = <T>(arr: T[]): T | undefined => arr[0];

// 4. Array utility — oxirgi elementni olish
function lastElement<T>(arr: T[]): T | undefined {
  return arr[arr.length - 1];
}

lastElement([1, 2, 3]);      // number | undefined
lastElement(["a", "b"]);     // string | undefined

// 5. Ikki array'ni birlashtirish — bitta type
function concat<T>(a: T[], b: T[]): T[] {
  return [...a, ...b];
}

const nums = concat([1, 2], [3, 4]);       // number[]
const strs = concat(["a"], ["b", "c"]);    // string[]
// concat([1, 2], ["a"]); // ❌ Error: T bir xil bo'lishi kerak

// 6. Generic reverse
function reverse<T>(arr: T[]): T[] {
  return [...arr].reverse();
}

reverse([1, 2, 3]); // [3, 2, 1] — number[]

// 7. Wrap va unwrap pattern
function wrap<T>(value: T): { value: T } {
  return { value };
}

function unwrap<T>(container: { value: T }): T {
  return container.value;
}

const wrapped = wrap(42);      // { value: number }
const unwrapped = unwrap(wrapped); // number

// 8. Generic find
function find<T>(arr: T[], predicate: (item: T) => boolean): T | undefined {
  for (const item of arr) {
    if (predicate(item)) return item;
  }
  return undefined;
}

const user = find([{ name: "Ali" }, { name: "Vali" }], u => u.name === "Ali");
// user: { name: string } | undefined
```

</details>

---

## Generic Inference

### Nazariya

TypeScript ko'pincha type parameter'ni **o'zi aniqlaydi** — explicit berish shart emas. Inference argument'lardan amalga oshadi: kompilator argument'ning type'iga qarab `T`'ni aniqlaydi.

**Inference jarayoni:**

1. Funksiya chaqiriladi — `identity("hello")`
2. Kompilator argument type'ini aniqlaydi — `"hello"` → `string`
3. `T = string` deb belgilaydi
4. Barcha `T` ishlatilgan joylarni shu type bilan tekshiradi — return type ham `string`

```typescript
function identity<T>(value: T): T {
  return value;
}

const a = identity("hello"); // T = string (inference)
const b = identity(42);       // T = number (inference)
```

**Inference ko'plab argumentlardan:** Agar bir nechta argument bo'lsa, kompilator har biridan type candidate yig'adi va **best common type** algoritmi bilan yakuniy type'ni tanlaydi.

```typescript
function merge<T>(a: T, b: T): T {
  return { ...a, ...b };
}

merge({ x: 1 }, { y: "a" });
// T = { x: number } | { y: string } — common type
```

**Contextual inference** — callback'da parameter type avtomatik infer bo'ladi:

```typescript
function map<T, U>(arr: T[], fn: (value: T) => U): U[] {
  return arr.map(fn);
}

const result = map([1, 2, 3], (n) => n.toString());
// T = number (argument'dan), U = string (callback return'idan)
// result: string[]
```

<details>
<summary><strong>Under the Hood</strong></summary>

Type inference — kompilatorning eng murakkab algoritmlaridan biri. Umumiy jarayon:

```
Inference Pipeline:

1. Argument'larni yig'ish:
   identity("hello") → argument type: string

2. Type parameter'ga candidate qo'shish:
   T ← string (from argument 1)

3. Bir nechta candidate bo'lsa — best common type:
   merge({x: 1}, {y: "a"}) →
     T candidate 1: { x: number }
     T candidate 2: { y: string }
     Best common: { x: number } | { y: string }

4. Contextual inference (callback'lar uchun):
   map([1,2,3], (n) => ...) →
     map signature: <T, U>(arr: T[], fn: (value: T) => U) => U[]
     [1,2,3] → T = number (from argument 1)
     callback parameter `n` contextual type = T = number
     callback return `n.toString()` → U = string
```

**Best common type** algoritmi: kompilator candidate'lar orasidan eng keng type'ni tanlaydi. Agar ular hierarchy'da bog'liq bo'lsa — eng pastki ota tanlanadi (masalan, ikkala candidate `Dog` va `Cat` bo'lsa → `Dog | Cat` yoki `Animal` parent type mavjud bo'lsa).

**Contextual typing** (bidirectional type flow): Type ma'lumot tashqi funksiyadan ichki callback'ga oqadi. Kompilator callback parameter'larning type'ini tashqi signature'dan oladi — bu inline callback'larda parameter type yozmaslik imkonini beradi.

**Variance positions va inference:** Parameter position'da (contra-variant) inference boshqacha natija berishi mumkin. Kompilator buni internal priority flag'lari orqali boshqaradi.

**Inference muvaffaqiyatsiz:** Agar kompilator hech qanday candidate topa olmasa, `T = unknown` bo'ladi — xato emas, lekin foydali emas. Shuning uchun argumentsiz generic funksiyalarda explicit type berish tavsiya etiladi.

Runtime'da inference natijasi hech qanday iz qoldirmaydi — hammasi compile-time computation.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy inference
function identity<T>(value: T): T {
  return value;
}

const str = identity("hello"); // T = string
const num = identity(42);       // T = number

// 2. Array'dan T'ni infer qilish
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const n = first([1, 2, 3]);  // T = number
const s = first(["a", "b"]); // T = string

// 3. Callback'da contextual typing
function map<T, U>(arr: T[], fn: (value: T) => U): U[] {
  return arr.map(fn);
}

const lengths = map(["hello", "world"], (s) => s.length);
// T = string (argument'dan), U = number (callback return'idan)
// s: string — contextual type
// lengths: number[]

// 4. Multiple type parameter inference
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const p1 = pair("name", 25);          // [string, number]
const p2 = pair(true, { id: 1 });     // [boolean, { id: number }]

// 5. Best common type
function mostSpecific<T>(a: T, b: T): T {
  return a;
}

mostSpecific(1, 2);                // T = number
mostSpecific("a", "b");            // T = string
mostSpecific({ a: 1 }, { b: 2 });  // T = { a: number } | { b: number }

// 6. Partial<T> bilan inference
function update<T>(target: T, source: Partial<T>): T {
  return { ...target, ...source };
}

const updated = update(
  { name: "Ali", age: 25 },
  { age: 26 }
);
// T = { name: string; age: number } (birinchi argument'dan)
// source type: Partial<T> — qisman update'ni majburlaydi

// 7. Inference muvaffaqiyatsiz — argumentsiz funksiya
function createEmpty<T>(): T[] {
  return [];
}

// const arr = createEmpty(); // T = unknown — foydasiz
const arr = createEmpty<string>(); // Explicit: string[]

// 8. Generic'lar method chain'ida
const numbers = [1, 2, 3, 4, 5];
const result = numbers
  .filter((n) => n > 2)   // T = number (inferred from array)
  .map((n) => n * 2)       // U = number (inferred from callback)
  .map((n) => n.toString()); // V = string

// result: string[]
```

</details>

---

## Explicit Type Arguments

### Nazariya

Explicit type argument — chaqiruv paytida `<Type>`'ni qo'lda ko'rsatish. Uchta holatda kerak:

1. **Inference imkonsiz** — argumentlardan type aniqlanmaydi (argumentsiz funksiya)
2. **Inference noto'g'ri** — TypeScript kengroq type infer qiladi, torroq kerak
3. **Aniqlik uchun** — murakkab chaqiruvlarda o'qilishni yaxshilash

```typescript
function identity<T>(value: T): T {
  return value;
}

// Inference ishlaydi — explicit kerak emas
const a = identity("hello");           // T = string
const b = identity<string>("hello");   // Keraksiz, lekin valid

// Inference ishlamaydi — explicit kerak
function parseJSON<T>(json: string): T {
  return JSON.parse(json);
}

// const data = parseJSON('{"name":"Ali"}');  // T = unknown — foydasiz
const data = parseJSON<{ name: string }>('{"name":"Ali"}');
// data: { name: string }
```

**Partial type argument cheklovi:** TypeScript'da `fn<string>(...)` yozilganda, agar funksiya `<T, U>` bo'lsa — ikkala type ham explicit berilishi kerak, yoki **default type parameter** ishlatiladi. Ikkinchi type'ni alohida explicit berib, birinchisini inference'ga qoldirish hozircha mumkin emas.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `identity<number>(42)` yozilganda inference bosqichini butunlay o'tkazib yuboradi. Funksiya chaqiruv resolver'i avval explicit type argument'lar borligini tekshiradi — agar bor bo'lsa, inference ishga tushmaydi, to'g'ridan-to'g'ri berilgan type'lar ishlatiladi.

Angle bracket sintaksisi `<number>` faqat TypeScript parser uchun mavjud. AST'da `TypeArguments` node sifatida saqlanadi. Emit bosqichida bu node butunlay skip qilinadi — `identity<number>(42)` JS'da `identity(42)` bo'ladi. Ya'ni explicit yoki inferred — JS output **aynan bir xil**.

**Constraint tekshirish:** Kompilator explicit type'ni constraint'ga tekshiradi. `function fn<T extends string>(x: T)`'da `fn<number>(42)` yozilsa — `number extends string` false, compile error.

**Partial type argument cheklovi:** Bu TypeScript'ning hozirgi cheklovi — GitHub'da yillar davomida muhokama qilinmoqda. Yagona yechim — **default type parameter** ishlatish:

```typescript
function fn<T = string, U = number>(a: T, b: U): [T, U] {
  return [a, b];
}

// ❌ Partial: fn<boolean>("a", 42);
// ✅ Default bilan ikkalasini skip
fn("a", 42);                    // T = string (default), U = number (default)
fn<boolean, string>(true, "a"); // Ikkalasi explicit
```

**Type casting vs explicit generic:** `parseJSON<User>(json)` va `JSON.parse(json) as User` — ikkalasi ham compile-time'da `User` type beradi, lekin ikkisi ham **runtime'da tekshirmaydi**. Type safety faqat developer'ning berilgan type to'g'riligiga ishonchiga asoslangan. External data bilan ishlaganda validator (zod, io-ts) yoki type guard ishlatish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Argumentsiz funksiya — explicit shart
function createEmptyArray<T>(): T[] {
  return [];
}

const strings = createEmptyArray<string>();  // string[]
const users = createEmptyArray<User>();       // User[]

interface User {
  name: string;
}

// 2. JSON.parse pattern
function parseJSON<T>(json: string): T {
  return JSON.parse(json);
}

const config = parseJSON<{ host: string; port: number }>(
  '{"host":"localhost","port":3000}'
);
// config: { host: string; port: number }

// ⚠️ MUHIM: parseJSON runtime'da tekshirmaydi!
// Agar JSON noto'g'ri bo'lsa — TS bilmaydi, runtime xato yuzaga keladi
// Production'da zod/io-ts kabi validator ishlatish kerak

// 3. Factory pattern — type explicit
function createContainer<T>(initialValue: T): { get: () => T; set: (v: T) => void } {
  let state = initialValue;
  return {
    get: () => state,
    set: (v) => { state = v; },
  };
}

// Inference ishlaydi
const counter = createContainer(0); // T = number

// Union type uchun explicit kerak
const status = createContainer<"idle" | "loading" | "ready">("idle");
// Inference bo'lsa — T = "idle" (faqat birinchi qiymat), explicit'da union

// 4. Nested generic
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach((k) => { result[k] = obj[k]; });
  return result;
}

const user = { name: "Ali", age: 25, email: "ali@test.com" };

// Inference ishlaydi
const subset = pick(user, ["name", "email"]);
// subset: Pick<User, "name" | "email">

// Explicit variant — inference kuchini yo'qotadi, kam ishlatiladi
const subset2 = pick<typeof user, "name" | "email">(user, ["name", "email"]);

// 5. Generic React hook pattern
function useState<T>(initial: T): [T, (newValue: T) => void] {
  let state = initial;
  return [state, (v) => { state = v; }];
}

// Inference ishlaydi
const [count, setCount] = useState(0);       // T = number
const [name, setName] = useState("");         // T = string

// Union uchun explicit kerak
type Status = "idle" | "loading" | "ready";
const [status, setStatus] = useState<Status>("idle");

// 6. Default parameter bilan partial inference
function create<T = string, U = number>(a: T, b: U): { a: T; b: U } {
  return { a, b };
}

create("hello", 42);              // T = string (inferred), U = number (inferred)
create<boolean, string>(true, "x"); // Ikkalasi explicit
```

</details>

---

## Generic Interfaces

### Nazariya

Interface ham type parameter qabul qilishi mumkin — bu reusable data structure va contract'larni type-safe qiladi. Generic interface ishlatilganda aniq type **ko'rsatilishi shart** (funksiyadan farqli — inference yo'q, chunki interface'ning "chaqiruv" argumenti yo'q):

```typescript
interface Box<T> {
  value: T;
  getValue(): T;
  setValue(newValue: T): void;
}

// Ishlatishda aniq type beriladi
const numberBox: Box<number> = {
  value: 42,
  getValue() { return this.value; },
  setValue(v) { this.value = v; },
};

const stringBox: Box<string> = {
  value: "hello",
  getValue() { return this.value; },
  setValue(v) { this.value = v; },
};

// numberBox.setValue("oops"); // ❌ Error: string !== number
```

**Generic interface extend:**

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
}

interface PaginatedResponse<T> extends ApiResponse<T[]> {
  page: number;
  totalPages: number;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Generic interface — butunlay compile-time konstruksiya. JavaScript'da `interface` tushunchasi mavjud emas, shuning uchun generic interface emit bosqichida **to'liq o'chiriladi** — hech qanday runtime kod qolmaydi.

`interface Box<T> { value: T }` declaration qilinganda, kompilator `InterfaceDeclaration` node yaratadi va type parameter `T`'ni scope'ga qo'shadi. Bu faqat type checker uchun — emitter bu node'ni butunlay ignore qiladi. `Box<number>` ishlatilganda kompilator **type instantiation** qiladi — yangi type yaratadi va `T` o'rniga `number` qo'yadi. Bu instantiated type faqat checker memory'da yashaydi.

**Interface extends propagation:** `PaginatedResponse<T> extends ApiResponse<T[]>`'da kompilator parent interface'ning type parameter'larini child'ga propagate qiladi. `T` child'da `T[]`'ga map qilinadi. Bu mapping compile-time'da amalga oshadi.

**Declaration merging va generics:** Generic interface'lar merge qilish (declaration merging)'ni qo'llab-quvvatlaydi — lekin faqat **bir xil type parameter count** bo'lganda. Ikki `Box<T>` va `Box<T, U>` merge bo'lmaydi.

```typescript
interface Box<T> { value: T; }
interface Box<T> { timestamp: Date; } // Merge bilan Box<T> = { value: T; timestamp: Date }

// interface Box<T, U> { ... } // ❌ Merge bo'lmaydi — parameter count farqli
```

**Runtime'da iz yo'q:** `const box: Box<number> = { value: 42 }` shunchaki `const box = { value: 42 }` bo'ladi. `Box<number>` type annotation — `instanceof` yoki `typeof` bilan tekshirib bo'lmaydi. Interface faqat compile-time'da structural type checking uchun xizmat qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy generic interface
interface Container<T> {
  value: T;
  isEmpty(): boolean;
  clear(): void;
}

const numContainer: Container<number> = {
  value: 42,
  isEmpty() { return this.value === 0; },
  clear() { this.value = 0; },
};

// 2. API response pattern
interface ApiResponse<T> {
  data: T;
  status: number;
  timestamp: Date;
}

interface User {
  id: number;
  name: string;
}

const response: ApiResponse<User> = {
  data: { id: 1, name: "Ali" },
  status: 200,
  timestamp: new Date(),
};

// 3. Extended generic interface
interface PaginatedResponse<T> extends ApiResponse<T[]> {
  page: number;
  totalPages: number;
  totalItems: number;
}

interface Product {
  id: number;
  name: string;
  price: number;
}

const products: PaginatedResponse<Product> = {
  data: [
    { id: 1, name: "Phone", price: 999 },
    { id: 2, name: "Laptop", price: 1299 },
  ],
  status: 200,
  timestamp: new Date(),
  page: 1,
  totalPages: 5,
  totalItems: 50,
};

// 4. Callable generic interface
interface Transformer<T, U> {
  (input: T): U;
}

const stringToNumber: Transformer<string, number> = (s) => parseInt(s);
const numberToBoolean: Transformer<number, boolean> = (n) => n > 0;

// 5. Generic interface with constraints
interface Identifiable {
  id: string;
}

interface Store<T extends Identifiable> {
  items: Map<string, T>;
  add(item: T): void;
  findById(id: string): T | undefined;
}

// 6. Multiple type parameter
interface KeyValuePair<K, V> {
  key: K;
  value: V;
}

const entry: KeyValuePair<string, number> = {
  key: "age",
  value: 25,
};

// 7. Interface merging with generics
interface Config<T> {
  data: T;
}

interface Config<T> {
  timestamp: Date; // Merged
}

const config: Config<string> = {
  data: "hello",
  timestamp: new Date(),
};
```

</details>

---

## Generic Classes

### Nazariya

Class'da type parameter — class nomi yonida `<T>` qo'yiladi. Bu type parameter class ichidagi barcha property, method va constructor'larda ishlatilishi mumkin. Class'dan instance yaratilganda type aniqlanadi (inference yoki explicit).

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  get size(): number { return this.items.length; }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
// numberStack.push("3"); // ❌ Error: string !== number

const stringStack = new Stack<string>();
stringStack.push("hello");
```

**Instantiation expression** (TS 4.7+) — generic class'ni "pre-bind" qilish:

```typescript
const NumStack = Stack<number>; // Type — class constructor bilan T = number
const stack = new NumStack();   // Stack<number>
```

Bu pattern factory yoki dependency injection'da foydali.

<details>
<summary><strong>Under the Hood</strong></summary>

Generic class — interface'dan farqli ravishda, **class o'zi runtime'da saqlanadi**, lekin type parameter o'chiriladi. `class Stack<T>` → JS'da `class Stack` bo'ladi. Class body — constructor, method'lar, property'lar — hammasi qoladi, faqat type annotation'lar ketadi.

**Bitta class emit:** Kompilator `Stack<number>` va `Stack<string>` uchun **bitta class** emit qiladi. Java yoki C# dan farqli ravishda, TypeScript'da **type erasure** modeli ishlatiladi — runtime'da `Stack<number>` va `Stack<string>` orasida hech qanday farq yo'q. `new Stack<number>()` va `new Stack<string>()` bir xil constructor chaqiradi. `instanceof Stack` ikkala instance uchun `true` qaytaradi — generic parameter ko'rinmaydi.

**Static member'larda type parameter ishlatib bo'lmaydi:** Static member'lar class'ning o'ziga tegishli (prototype'ga emas), lekin type parameter instance-level concept. Kompilator bu cheklov'ni enforce qiladi:

```typescript
class Container<T> {
  // static default: T; // ❌ Error: Static members cannot reference class type parameters
  // static create(value: T): Container<T> {} // ❌ Error
}
```

Yechim: method'ni o'z type parameter'i bilan qilish (method-level generic, class'ning T'sidan mustaqil).

**Constructor inference:** `new Stack<number>()` — explicit. Lekin agar constructor argument qabul qilsa, kompilator inference qiladi: `new Box(42)` → `Box<number>`. Bu TypeScript'ning uzoq vaqtdan beri qo'llab-quvvatlaydigan xususiyati.

**Instantiation expressions** (TS 4.7+): `const NumStack = Stack<number>` — class tipini pre-binding qilish. Bu feature factory'larda va dependency injection'da foydali — siz bitta bog'langan class'ni variable sifatida yurita olasiz. Constructor inference (yuqoridagi `new Box(42)`)'dan farqli — bu yangi feature, 4.7 dan oldin yo'q edi.

**Private initializer emit:** `private items: T[] = []` emit qilinganida `items = []` bo'ladi — type annotation ketadi, lekin initialization expression saqlanadi. Runtime'da `items` oddiy array — hech qanday type enforcement yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy generic class — Stack
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  get size(): number { return this.items.length; }
  toArray(): T[] { return [...this.items]; }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
const top = numStack.pop(); // number | undefined

// 2. Constructor inference
class Box<T> {
  constructor(public value: T) {}
  get(): T { return this.value; }
  set(value: T): void { this.value = value; }
}

const strBox = new Box("hello"); // T inferred: string
const numBox = new Box(42);       // T inferred: number

// 3. Generic class with constraint — Repository pattern
interface Entity {
  id: string;
  createdAt: Date;
}

class Repository<T extends Entity> {
  private store = new Map<string, T>();

  save(entity: T): void {
    this.store.set(entity.id, entity);
  }

  findById(id: string): T | undefined {
    return this.store.get(id);
  }

  findAll(): T[] {
    return Array.from(this.store.values());
  }

  delete(id: string): boolean {
    return this.store.delete(id);
  }
}

interface User extends Entity {
  name: string;
  email: string;
}

const userRepo = new Repository<User>();
userRepo.save({
  id: "1",
  name: "Ali",
  email: "ali@test.com",
  createdAt: new Date(),
});

// 4. Generic method inside generic class
class Mapper<T> {
  constructor(private items: T[]) {}

  // Method-level generic — class T'sidan mustaqil
  map<U>(fn: (item: T) => U): U[] {
    return this.items.map(fn);
  }

  filter(predicate: (item: T) => boolean): T[] {
    return this.items.filter(predicate);
  }
}

const mapper = new Mapper([1, 2, 3, 4, 5]);
const strings = mapper.map((n) => n.toString()); // string[]
const evens = mapper.filter((n) => n % 2 === 0); // number[]

// 5. Instantiation expression (TS 4.7+)
const StringStack = Stack<string>;
const strStack = new StringStack();
strStack.push("hello");
strStack.push("world");

// Factory with instantiation expression
function createRepositoryFor<T extends Entity>(): Repository<T> {
  return new Repository<T>();
}

const productRepo = createRepositoryFor<Product>();

interface Product extends Entity {
  name: string;
  price: number;
}

// 6. Multiple type parameters in class
class KeyValueStore<K, V> {
  private map = new Map<K, V>();

  set(key: K, value: V): void { this.map.set(key, value); }
  get(key: K): V | undefined { return this.map.get(key); }
  has(key: K): boolean { return this.map.has(key); }
}

const userAges = new KeyValueStore<string, number>();
userAges.set("Ali", 25);
userAges.get("Ali"); // number | undefined
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}
```

```javascript
// Compiled JS (ES2015+)
class Stack {
  constructor() {
    this.items = [];
  }
  push(item) {
    this.items.push(item);
  }
  pop() {
    return this.items.pop();
  }
}
// <T>, private, T[], T | undefined — hammasi type erasure
// Runtime'da Stack<number> va Stack<string> bir xil class
```

</details>

---

## Generic Constraints

### Nazariya

Generic constraint — `extends` keyword bilan type parameter'ni cheklash. `<T extends SomeType>` — "T istalgan type emas, faqat SomeType'ga mos (yoki uning subtype) bo'lishi kerak" degan ma'no.

Constraint'siz `T` haqida hech narsa bilmaymiz — faqat `T` sifatida qaytarish mumkin, ichida hech qanday operation yo'q:

```typescript
// ❌ Constraint'siz — T haqida hech narsa ma'lum emas
function getLength<T>(value: T): number {
  // return value.length; // ❌ Error: 'length' does not exist on type 'T'
  return 0;
}

// ✅ Constraint bilan — T'da length borligiga kafolat
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}

getLength("hello");         // ✅ string.length bor
getLength([1, 2, 3]);        // ✅ array.length bor
getLength({ length: 10 });   // ✅ object'da length bor
// getLength(42);             // ❌ number'da length yo'q
```

**Structural constraint:** `extends` qoidasi structural typing asosida ishlaydi — constraint interface'da `{ length: number }` deyilsa, berilgan type'da shu property bo'lishi kifoya. Qolgan property'lar ahamiyatsiz.

<details>
<summary><strong>Under the Hood</strong></summary>

Constraint tekshiruvi compile-time'da amalga oshadi. Kompilator type parameter instantiate bo'lganda, berilgan type constraint'ga mos kelishini tekshiradi — structural typing qoidalari bo'yicha.

```
Generic Constraint Checking Flow:

  <T extends { length: number }>

  Call Site                Check                        Result
  ─────────                ─────                        ──────

  getLength("hello")       string.length: number ────▶ ✅ T = string
  getLength([1, 2, 3])    Array.length: number  ────▶ ✅ T = number[]
  getLength(42)            number.length? ──────────▶ ❌ Compile Error
  getLength({ len: 5 })   { len } has length? ─────▶ ❌ Compile Error
```

**Structural check algoritmi:**

1. `T` = berilgan argument type
2. Constraint = `{ length: number }`
3. `isTypeAssignableTo(T, Constraint)` tekshiradi:
   - Constraint'dagi har bir property `T`'da bormi?
   - Property type'lari mos keladimi?
4. Agar `true` → instantiation davom etadi
5. Agar `false` → compile error

**Constraint va narrowing:** Constraint type parameter'ning "pastki chegarasi"ni belgilaydi — `T`'ning value'si har doim constraint'ga mos keladi. Lekin `T` o'zi keng bo'lishi mumkin. `<T extends string>` — `T` `string`, yoki `"hello"`, yoki `string | "other"` bo'lishi mumkin.

**Constraint bilan generic constraint'ni hosil qilish:** Type parameter constraint'da boshqa type parameter'ni ishlatish mumkin:

```typescript
function prop<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

Bu yerda `K`'ning constraint'i `T`'ga bog'liq. Kompilator avval `T`'ni infer qiladi, keyin `keyof T`'ni hisoblaydi, keyin `K`'ni shu constraint'ga tekshiradi. Dependency order muhim — `T` avval bo'lishi kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Shape constraint
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}

getLength("hello");         // ✅
getLength([1, 2, 3]);        // ✅
getLength({ length: 10, data: "x" }); // ✅

// 2. Interface constraint
interface Serializable {
  serialize(): string;
}

function save<T extends Serializable>(item: T): void {
  const data = item.serialize();
  console.log("Saved:", data);
}

class User implements Serializable {
  constructor(public name: string) {}
  serialize(): string { return JSON.stringify({ name: this.name }); }
}

save(new User("Ali")); // ✅

// 3. Union constraint
function formatId<T extends string | number>(id: T): string {
  return `ID-${id}`;
}

formatId("abc");  // ✅
formatId(123);    // ✅
// formatId(true); // ❌

// 4. Class/constructor constraint
function createInstance<T>(ctor: new () => T): T {
  return new ctor();
}

class Dog {}
const dog = createInstance(Dog); // Dog

// 5. Constraint with default
function wrap<T extends object = {}>(value: T): { data: T } {
  return { data: value };
}

wrap({ name: "Ali" }); // { data: { name: string } }

// 6. Nested constraint
interface HasId {
  id: string;
}

interface HasName extends HasId {
  name: string;
}

function findByName<T extends HasName>(items: T[], name: string): T | undefined {
  return items.find((item) => item.name === name);
}

const users = [
  { id: "1", name: "Ali", age: 25 },
  { id: "2", name: "Vali", age: 30 },
];

const user = findByName(users, "Ali"); // { id: string; name: string; age: number } | undefined

// 7. Generic constraint with keyof (preview — batafsil pastda)
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// 8. Function constraint
function invoke<T extends () => unknown>(fn: T): ReturnType<T> {
  return fn() as ReturnType<T>;
}

invoke(() => 42);       // number
invoke(() => "hello");  // string
```

</details>

---

## keyof Bilan Generics

### Nazariya

`keyof` operator — object type'ning barcha key'larini **union type** sifatida oladi. `keyof T` — T type'ning barcha property nomlari.

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type UserKeys = keyof User; // "name" | "age" | "email"
```

Generic'lar bilan birga ishlatilganda kuchli pattern hosil bo'ladi: `<T, K extends keyof T>` — "T istalgan object, K esa T'ning key'laridan biri".

**Eng mashhur pattern — type-safe property access:**

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Ali", age: 25, email: "ali@test.com" };

const name = getProperty(user, "name"); // string
const age = getProperty(user, "age");    // number
// getProperty(user, "phone"); // ❌ Error: "phone" keyof User'da yo'q
```

**Muhim nuance — `keyof` va class visibility:** Class type'lar uchun `keyof T` faqat **public** member'larni beradi. `private`, `protected` TypeScript keyword'lari bilan belgilangan member'lar `keyof T` natijasiga **kirmaydi**. ECMAScript native private field'lar (`#field`) ham `keyof`'da ko'rinmaydi.

```typescript
class Account {
  public name: string = "";
  private balance: number = 0;
  protected userId: string = "";
  #internal: boolean = false;
}

type AccountKeys = keyof Account; // "name" (faqat public)
// "balance", "userId", "#internal" — hech biri kirmaydi
```

<details>
<summary><strong>Under the Hood</strong></summary>

`keyof` — compile-time operator, runtime'da mavjud emas. `keyof T` yozilganda kompilator `T`'ning barcha public property key'larini **string/number/symbol literal union** type sifatida hisoblaydi.

**Generic kontekstda `keyof T` deferred bo'ladi:** `T` hali aniq bo'lmaganda kompilator `keyof T`'ni evaluate qilmaydi, balki saqlab qo'yadi. Faqat `T` aniq type'ga instantiate bo'lganda (masalan, `T = User`), kompilator `keyof User`'ni `"name" | "age" | "email"`'ga resolve qiladi.

```
<T, K extends keyof T> pattern'ning ikki bosqichi:

1. T instantiate bo'ladi (inference yoki explicit'dan)
   T = User

2. keyof T hisoblanadi
   keyof User = "name" | "age" | "email"

3. K constraint'ga tekshiriladi
   getProperty(user, "name") → "name" ∈ keyof User? → ✅
   getProperty(user, "phone") → "phone" ∈ keyof User? → ❌
```

**Class visibility va `keyof`:** TypeScript class type'lar uchun `keyof T` faqat **public** member'larni qaytaradi. Bu accessibility filtering type-level'da amal qiladi.

- `public` member'lar — `keyof T`'da bor
- `private` keyword bilan belgilangan member'lar — `keyof T`'da **yo'q**
- `protected` keyword bilan belgilangan member'lar — `keyof T`'da **yo'q**
- `#field` (ECMAScript native private) — `keyof T`'da **yo'q**, chunki ular type system perspektivasidan oddiy property emas, balki maxsus syntactic construct

Bu dizayn qarori pedagogik jihatdan muhim: generic funksiyalar faqat public API bilan ishlashi kerak. Agar siz generic'ga private member'larga access kerak bo'lsa, siz encapsulation'ni buzyapsiz — refactor kerak.

**Runtime'da iz yo'q:** Emit bosqichida `keyof`'ning hech qanday izi qolmaydi. `function getProperty<T, K extends keyof T>(obj: T, key: K): T[K]` → JS'da `function getProperty(obj, key) { return obj[key]; }`. Runtime'da `key`'ning valid ekanligi tekshirilmaydi — `obj[key]` noto'g'ri key bilan `undefined` qaytaradi. Type safety faqat compile-time'da amal qiladi.

**`keyof any`:** Bu maxsus type — JavaScript'da object key bo'lishi mumkin bo'lgan type'lar: `string | number | symbol`. Foydali pattern: `Record<keyof any, unknown>` — istalgan key'li object.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Asosiy pattern — type-safe property access
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Ali", age: 25, email: "ali@test.com" };

const name = getProperty(user, "name"); // string
const age = getProperty(user, "age");    // number
// getProperty(user, "phone"); // ❌ Error

// 2. Type-safe setter
function setProperty<T, K extends keyof T>(obj: T, key: K, value: T[K]): void {
  obj[key] = value;
}

const config = { host: "localhost", port: 3000, debug: true };

setProperty(config, "host", "0.0.0.0");  // ✅ string → string
setProperty(config, "port", 8080);        // ✅ number → number
// setProperty(config, "port", "8080");   // ❌ string !== number

// 3. Multiple keys — Pick pattern
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach((key) => {
    result[key] = obj[key];
  });
  return result;
}

const subset = pick(user, ["name", "email"]);
// subset: { name: string; email: string }

// 4. Keys extraction
interface Config {
  host: string;
  port: number;
  debug: boolean;
}

type ConfigKeys = keyof Config; // "host" | "port" | "debug"

// 5. Class visibility demo — faqat public keyof'da
class Account {
  public name: string = "";
  private balance: number = 0;
  protected userId: string = "";
}

type AccountKeys = keyof Account; // "name"
// "balance" va "userId" — keyof da YO'Q

// 6. Bir nechta key bilan filter
function filterByKeys<T, K extends keyof T>(
  items: T[],
  keys: K[]
): Pick<T, K>[] {
  return items.map((item) => {
    const result = {} as Pick<T, K>;
    keys.forEach((key) => {
      result[key] = item[key];
    });
    return result;
  });
}

const users = [
  { id: 1, name: "Ali", email: "ali@test.com", age: 25 },
  { id: 2, name: "Vali", email: "vali@test.com", age: 30 },
];

const simplified = filterByKeys(users, ["name", "email"]);
// simplified: { name: string; email: string }[]

// 7. Record<keyof any, V> — universal key type
function safeGet<V>(obj: Record<keyof any, V>, key: string): V | undefined {
  return obj[key];
}

// 8. Typed event emitter pattern
interface EventMap {
  click: { x: number; y: number };
  keydown: { key: string };
  scroll: { delta: number };
}

function on<K extends keyof EventMap>(
  event: K,
  handler: (data: EventMap[K]) => void
): void {
  // handler chaqiruvi
}

on("click", (data) => {
  // data: { x: number; y: number }
  console.log(data.x, data.y);
});

on("keydown", (data) => {
  // data: { key: string }
  console.log(data.key);
});

// on("unknown", ...); // ❌ "unknown" keyof EventMap'da yo'q
```

</details>

---

## Index Access Types

### Nazariya

Index Access Type — `T[K]` sintaksisi bilan type'ning ma'lum property'sining type'ini olish. Bu JavaScript'dagi `obj["key"]`'ga o'xshash, lekin **type darajasida** ishlaydi — runtime emas, compile-time'da.

```typescript
interface User {
  name: string;
  age: number;
  address: {
    city: string;
    zip: string;
  };
}

type NameType = User["name"];           // string
type AgeType = User["age"];              // number
type AddressType = User["address"];      // { city: string; zip: string }
type CityType = User["address"]["city"]; // string (nested)
```

**Distributive behavior** — `T[K1 | K2]` union index'ga distribute bo'ladi:

```typescript
type NameOrAge = User["name" | "age"];
// = User["name"] | User["age"]
// = string | number
```

**Eng ko'p ishlatiladigan pattern'lar:**

- `T[keyof T]` — T'ning barcha value type'larining union
- `T[number]` — array/tuple element type
- `T[K]` generic kontekstda — property type'ni dynamically olish

<details>
<summary><strong>Under the Hood</strong></summary>

`T[K]` — kompilator ichida index access type node sifatida represent qilinadi. Bu node ikki qismdan iborat: objectType (`T`) va indexType (`K`). Kompilator `T` type'dagi `K` key'ning type'ini resolve qiladi.

**Non-generic holatda** — `User["name"]` — kompilator darhol resolve qiladi: `User` type'da `name` property'ni topadi va uning type'ini (`string`) qaytaradi. Bu compile-time lookup — JavaScript'dagi `user["name"]`'dan butunlay farqli. JS'da runtime'da object property'ga access bo'ladi, TS'da esa type system ichida type'ga access bo'ladi.

**Generic holatda** — `T[K]` — kompilator **deferred evaluation** qiladi. `T` va `K` hali aniq bo'lmaganda, index access type o'zgarishsiz saqlanadi. Faqat `T` va `K` aniq type'larga instantiate bo'lganda (masalan, `T = User`, `K = "name"`), kompilator `User["name"]` → `string` deb resolve qiladi.

**Distributive behavior — `T[K1 | K2]`:** Kompilator avval `K1 | K2` union'ni ko'radi va **har bir union member uchun alohida** `T[member]`'ni hisoblaydi, keyin natijalarni union qiladi:

```
User["name" | "age"]
  ↓ distribute
User["name"] | User["age"]
  ↓ resolve
string | number
```

Bu pattern `T[keyof T]` bilan juda kuchli — `keyof T` o'zi union, va `T[keyof T]` har key uchun value type'ni oladi:

```
Config = { host: string; port: number; debug: boolean }
keyof Config = "host" | "port" | "debug"
Config[keyof Config] = Config["host"] | Config["port"] | Config["debug"]
                     = string | number | boolean
```

**`T[number]` array/tuple'da:** Kompilator array type'ning element type'ini oladi. Tuple bilan esa ikki variant:
- `[string, number][number]` → `string | number` (union of all elements)
- `[string, number][0]` → `string` (literal index)
- `[string, number][1]` → `number`

**Runtime'da iz yo'q:** `T[K]`'ning hech qanday izi qolmaydi. `type NameType = User["name"]` → JS'da **hech narsa** emit qilinmaydi, chunki `type` declaration butunlay o'chiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Lookup types — T["key"]
interface User {
  name: string;
  age: number;
  address: {
    city: string;
    zip: string;
  };
}

type NameType = User["name"];      // string
type AgeType = User["age"];        // number
type AddressType = User["address"]; // { city: string; zip: string }

// Nested lookup
type CityType = User["address"]["city"]; // string

// 2. Union index — distributive
type NameOrAge = User["name" | "age"];
// = User["name"] | User["age"]
// = string | number

type AllValues = User[keyof User];
// = User["name"] | User["age"] | User["address"]
// = string | number | { city: string; zip: string }

// 3. Array element type — T[number]
const ROLES = ["admin", "user", "moderator"] as const;
type Role = (typeof ROLES)[number];
// = "admin" | "user" | "moderator"

// 4. Tuple element type
type Tuple = [string, number, boolean];
type TupleUnion = Tuple[number]; // string | number | boolean
type First = Tuple[0];            // string
type Second = Tuple[1];           // number
type Third = Tuple[2];            // boolean

// 5. Generic pluck — extract property from array
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}

const users = [
  { name: "Ali", age: 25 },
  { name: "Vali", age: 30 },
];

const names = pluck(users, "name"); // string[]
const ages = pluck(users, "age");    // number[]

// 6. Config value type extraction
interface AppConfig {
  host: string;
  port: number;
  debug: boolean;
  features: string[];
}

type ConfigKey = keyof AppConfig;          // "host" | "port" | "debug" | "features"
type ConfigValue = AppConfig[keyof AppConfig]; // string | number | boolean | string[]

// 7. Event payload lookup
interface EventMap {
  click: { x: number; y: number };
  input: { value: string };
  scroll: { delta: number };
}

type ClickPayload = EventMap["click"];  // { x: number; y: number }
type InputPayload = EventMap["input"];  // { value: string }

function handleEvent<K extends keyof EventMap>(
  type: K,
  handler: (payload: EventMap[K]) => void
): void {
  // handler chaqiruvi
}

handleEvent("click", (payload) => {
  console.log(payload.x, payload.y); // payload: { x: number; y: number }
});

// 8. Recursive lookup — deep property access
interface Organization {
  name: string;
  address: {
    street: string;
    city: string;
    country: {
      code: string;
      name: string;
    };
  };
}

type CountryCode = Organization["address"]["country"]["code"]; // string
type CountryName = Organization["address"]["country"]["name"]; // string

// 9. Readonly tuple bilan literal'larni saqlash
const HTTP_METHODS = ["GET", "POST", "PUT", "DELETE"] as const;
type HttpMethod = (typeof HTTP_METHODS)[number];
// "GET" | "POST" | "PUT" | "DELETE"

function request(method: HttpMethod, url: string): void {
  console.log(`${method} ${url}`);
}

request("GET", "/api/users");  // ✅
// request("PATCH", "/api/users"); // ❌
```

</details>

---

## typeof Type Operator

### Nazariya

`typeof` — **type context**'da ishlatilganda **qiymatdan type olish** operatori. Bu JavaScript'ning runtime `typeof` operatoridan **butunlay farqli** — TypeScript'ning `typeof` compile-time'da ishlaydi va to'liq TS type qaytaradi.

**Ikki kontekst:**

- **Expression context** (JS): `typeof x === "string"` — runtime type check, `"string" | "number" | ...` qaytaradi
- **Type context** (TS): `type T = typeof x` — compile-time type extraction, to'liq TypeScript type qaytaradi

```typescript
const config = {
  host: "localhost",
  port: 3000,
  endpoints: ["/api", "/health"],
};

// Type context — to'liq inferred type
type Config = typeof config;
// = { host: string; port: number; endpoints: string[] }

// Expression context (JS typeof) — runtime check
if (typeof config === "object") {
  // "object" runtime typeof natijasi
}
```

**Eng kuchli pattern — `ReturnType<typeof fn>`:** Funksiya return type'ini funksiyaning o'zidan olish. Agar funksiya o'zgarsa — type avtomatik yangilanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript'dagi type-level `typeof` va JavaScript'dagi runtime `typeof` — **butunlay ikki xil operator**. Parser ularni kontekst bo'yicha farqlaydi: type position'da (`: typeof x`, `type T = typeof x`) — TypeScript operator; expression position'da (`if (typeof x === "string")`) — JavaScript operator.

Type-level `typeof x` kompilator ichida type query node sifatida represent qilinadi. Kompilator `x` variable'ning type'ini oladi — bu variable'ga assign qilingan qiymatning to'liq inferred type'ini qaytaradi. `typeof config` → `{ host: string; port: number; endpoints: string[] }`. Bu oddiy `string` yoki `number` qaytaruvchi JS `typeof`'dan ancha batafsil.

**Cheklov — faqat identifier'larga:** Type-level `typeof` faqat **identifier** (variable, property) larga qo'llaniladi, arbitrary expression'larga emas. `type T = typeof fn()` yozib bo'lmaydi — buning uchun `ReturnType<typeof fn>` ishlatiladi.

**`as const` bilan birga eng kuchli:** `as const` bo'lmasa, widened type olinadi (`{ OK: number }`), `as const` bilan literal type olinadi (`{ readonly OK: 200 }`).

```
const STATUS = { OK: 200, NOT_FOUND: 404 };
typeof STATUS → { OK: number; NOT_FOUND: number } (widened)

const STATUS = { OK: 200, NOT_FOUND: 404 } as const;
typeof STATUS → { readonly OK: 200; readonly NOT_FOUND: 404 } (literal)
```

**Emit bosqichida yo'qoladi:** `type Config = typeof config` → JavaScript'da **hech narsa**. `const x: typeof config = ...` → JavaScript'da `const x = ...`. Lekin runtime `typeof` (expression context) saqlanadi — `if (typeof x === "string")` JS'da aynan shundayligicha qoladi.

**`keyof typeof X` pattern:** `typeof` va `keyof`'ni kombinatsiya qilib, object'ning key'larini union type sifatida olish mumkin. Bu ayniqsa `as const` bilan mashhur pattern — literal qiymat'lardan type yaratish:

```typescript
const STATUS = { OK: 200, NOT_FOUND: 404 } as const;
type StatusKey = keyof typeof STATUS;              // "OK" | "NOT_FOUND"
type StatusValue = (typeof STATUS)[keyof typeof STATUS]; // 200 | 404
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy typeof
const config = {
  host: "localhost",
  port: 3000,
  endpoints: ["/api", "/health"],
};

type Config = typeof config;
// Config = { host: string; port: number; endpoints: string[] }

// Boshqa qiymatni shu type'ga moslash
const anotherConfig: typeof config = {
  host: "0.0.0.0",
  port: 8080,
  endpoints: ["/v2"],
};

// 2. ReturnType bilan — eng mashhur pattern
function createUser(name: string, age: number) {
  return {
    id: Math.random().toString(36),
    name,
    age,
    createdAt: new Date(),
  };
}

type User = ReturnType<typeof createUser>;
// User = {
//   id: string;
//   name: string;
//   age: number;
//   createdAt: Date;
// }

function updateUser(user: User, changes: Partial<User>): User {
  return { ...user, ...changes };
}

// Agar createUser o'zgarsa — User type avtomatik yangilanadi

// 3. Parameters bilan
function sendRequest(url: string, method: "GET" | "POST", data?: unknown): void {
  console.log(url, method, data);
}

type SendRequestParams = Parameters<typeof sendRequest>;
// [string, "GET" | "POST", unknown | undefined]

// 4. as const bilan literal type'lar
const STATUS = {
  OK: 200,
  NOT_FOUND: 404,
  SERVER_ERROR: 500,
} as const;

type StatusMap = typeof STATUS;
// { readonly OK: 200; readonly NOT_FOUND: 404; readonly SERVER_ERROR: 500 }

type StatusKey = keyof typeof STATUS;
// "OK" | "NOT_FOUND" | "SERVER_ERROR"

type StatusCode = (typeof STATUS)[keyof typeof STATUS];
// 200 | 404 | 500

// 5. Enum o'rniga const object + typeof
const DIRECTION = {
  UP: "up",
  DOWN: "down",
  LEFT: "left",
  RIGHT: "right",
} as const;

type Direction = (typeof DIRECTION)[keyof typeof DIRECTION];
// "up" | "down" | "left" | "right"

function move(direction: Direction): void {
  console.log(`Moving ${direction}`);
}

move("up");    // ✅
// move("forward"); // ❌

// 6. Function type olish
const add = (a: number, b: number) => a + b;
type AddFn = typeof add; // (a: number, b: number) => number

const multiply: AddFn = (a, b) => a * b;
// AddFn type'ga mos keladi

// 7. Complex object — type inferance
const formConfig = {
  fields: [
    { name: "username", type: "text", required: true },
    { name: "email", type: "email", required: true },
    { name: "age", type: "number", required: false },
  ],
  submitUrl: "/api/register",
} as const;

type FormConfig = typeof formConfig;
// Barcha field'lar literal type bilan saqlanadi (as const tufayli)

type FieldType = (typeof formConfig.fields)[number]["type"];
// "text" | "email" | "number"
```

</details>

---

## Multiple Type Parameters

### Nazariya

Funksiya, interface yoki class'da bir nechta type parameter bo'lishi mumkin — `<T, U>`, `<T, U, V>` va hokazo. Har type parameter mustaqil — biri boshqasiga bog'liq bo'lishi shart emas (lekin constraint orqali bog'lash mumkin).

```typescript
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const p1 = pair("hello", 42);      // [string, number]
const p2 = pair(true, [1, 2]);     // [boolean, number[]]
```

**Bog'liq type parameter'lar** — biri boshqasiga constraint qo'yadi:

```typescript
// K — T'ning key'laridan biri (K T'ga bog'liq)
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

**Dependency order muhim** — `T` avval bo'lishi kerak, keyin `K`. Aks holda `K extends keyof T` evaluate bo'la olmaydi.

**Qoida: kamroq — yaxshi.** Har type parameter ma'no berishi kerak. Keraksiz type parameter kodni murakkablashtiradi va foyda bermaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Bir nechta type parameter bo'lganda kompilator har birini **mustaqil** inference qiladi. `pair<T, U>(first: T, second: U)`'da `pair("hello", 42)` chaqirilganda kompilator birinchi argument'dan `T = string`, ikkinchi argument'dan `U = number` deb alohida infer qiladi.

Inference'lar parallel emas, **sequential** — chapdan o'ngga argument tartibida. AST'da `<T, U>` bitta type parameter list node ichida ikki type parameter declaration child sifatida saqlanadi. Har type parameter o'zining scope binding'iga ega — `T` va `U` bir-biridan mustaqil symbol'lar.

**Bog'liq type parameter'larda dependency order:** `<T, K extends keyof T>`'da kompilator **dependency order**'ni hisobga oladi:

```
1. T infer bo'ladi (birinchi argument'dan)
   getProperty(user, "name") → T = typeof user

2. keyof T hisoblanadi
   keyof T = "name" | "age" | "email"

3. K shu constraint ichida tekshiriladi
   "name" ∈ keyof T → ✅

Agar T infer bo'lmasa, K'ning constraint'ini ham evaluate qilib bo'lmaydi
— shuning uchun ba'zan explicit type kerak.
```

**Order reverse qilib bo'lmaydi:** `<K extends keyof T, T>` — invalid, chunki `K` declaration paytida `T` hali ma'lum emas. Type parameter'lar chapdan o'ngga resolve bo'ladi.

**Emit bosqichida hammasi o'chiriladi:** `function pair<T, U>(first: T, second: U): [T, U]` → `function pair(first, second) { return [first, second]; }`. Nechta type parameter bo'lishidan qat'iy nazar, JavaScript output'da hech biri qolmaydi.

**Unused type parameter:** Kompilator keraksiz type parameter'larni aniqlamaydi — `<T, U>`'da `U` hech qayerda ishlatilmasa ham error bermaydi. Lekin ESLint'ning `no-unnecessary-type-parameters` rule'si buni aniqlaydi. Unused type parameter — type safety'ga hissa qo'shmaydi va faqat kodni murakkablashtiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Ikki mustaqil type parameter
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const p1 = pair("hello", 42);   // [string, number]
const p2 = pair(true, [1, 2]);  // [boolean, number[]]

// 2. Transform funksiya — kirish/chiqish type'lari farqli
function transform<T, U>(value: T, fn: (input: T) => U): U {
  return fn(value);
}

const length = transform("hello", (s) => s.length);     // number
const upper = transform("hello", (s) => s.toUpperCase()); // string
const parsed = transform("42", (s) => parseInt(s));      // number

// 3. Dependency order — K T'ga bog'liq
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// T avval infer bo'ladi (user'dan), keyin K tekshiriladi
const user = { name: "Ali", age: 25 };
const name = getProperty(user, "name"); // string

// 4. Uchta type parameter — map with index
function mapWithIndex<T, U, V>(
  arr: T[],
  fn: (item: T, index: number) => U,
  reducer: (acc: V, item: U) => V,
  initial: V
): V {
  return arr.reduce((acc, item, i) => reducer(acc, fn(item, i)), initial);
}

const result = mapWithIndex(
  ["a", "b", "c"],
  (s, i) => `${i}:${s}`,
  (acc: string[], item) => [...acc, item],
  [] as string[]
);
// result: string[]

// 5. Bog'liq type parameter — U T'ga constraint orqali
function assign<T extends object, U extends Partial<T>>(
  target: T,
  source: U
): T {
  return { ...target, ...source };
}

const merged = assign(
  { name: "Ali", age: 25 },
  { age: 26 }
);
// merged: { name: string; age: number }

// 6. Multi-parameter class
class HashMap<K, V> {
  private store = new Map<K, V>();

  set(key: K, value: V): void { this.store.set(key, value); }
  get(key: K): V | undefined { return this.store.get(key); }
  has(key: K): boolean { return this.store.has(key); }
  delete(key: K): boolean { return this.store.delete(key); }
}

const userAges = new HashMap<string, number>();
userAges.set("Ali", 25);
userAges.get("Ali"); // number | undefined

// 7. Bog'liq generic method
class Collection<T> {
  constructor(private items: T[]) {}

  // Method-level generic U class T'sidan mustaqil
  map<U>(fn: (item: T) => U): Collection<U> {
    return new Collection(this.items.map(fn));
  }

  filter(predicate: (item: T) => boolean): Collection<T> {
    return new Collection(this.items.filter(predicate));
  }
}

const numbers = new Collection([1, 2, 3]);
const strings = numbers.map((n) => n.toString()); // Collection<string>

// 8. ❌ Keraksiz type parameter — U ishlatilmagan
function bad<T, U>(value: T): T {
  return value;
}

// ✅ Har type parameter ma'noli rol o'ynaydi
function good<T, U>(value: T, transform: (v: T) => U): U {
  return transform(value);
}
```

</details>

---

## Default Type Parameters

### Nazariya

Type parameter'ga default qiymat berish mumkin — `<T = DefaultType>`. Agar type explicit berilmasa va inference ham ishlamasa — default ishlatiladi. Bu funksiya default parameter'ga o'xshash, lekin type darajasida.

```typescript
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}

// Type berilmasa — T = unknown
const errorResponse: ApiResponse = {
  data: null,
  status: 500,
};

// Type berilsa — T = User
const userResponse: ApiResponse<User> = {
  data: { name: "Ali" },
  status: 200,
};
```

**Default type parameter qoidalar:**

- Default faqat **keyingi** type parameter'dan keyin kelishi kerak (positional)
- Default oldingi type parameter'ga reference qilishi mumkin (`<T, U = T>`) — lekin teskari emas
- Default constraint'ga mos kelishi shart (`<T extends object = string>` — xato)

<details>
<summary><strong>Under the Hood</strong></summary>

Default type parameter — kompilator type parameter'ni resolve qilayotganda uch bosqichli fallback ketma-ketligi:

1. **Explicit type argument** berilganmi? — uni ishlatadi
2. **Inference** mumkinmi argument'lardan? — inferred type'ni ishlatadi
3. **Ikkalasi ham yo'q** — default type'ni ishlatadi

Default faqat uchinchi bosqichda amal qiladi.

**AST'da saqlanishi:** `<T = string>` type parameter declaration'ning `default` property'sida saqlanadi. Bu expression emas, type — shuning uchun runtime'da mavjud bo'lmaydi.

**Constraint'ga mos kelishi:** Default type ham constraint'ga mos kelishi **shart**. `<T extends object = string>` — compile error, chunki `string extends object` false.

```typescript
// ✅ Default constraint'ga mos
function fn<T extends { id: string } = { id: string }>(x: T): T {
  return x;
}

// ❌ Default constraint'ga mos emas
// function fn<T extends object = string>(x: T): T { ... }
// Error: Type 'string' does not satisfy the constraint 'object'
```

**Boshqa type parameter'ga reference:** Default type parameter oldin declared bo'lgan type parameter'ga reference qilishi mumkin:

```typescript
// ✅ U default = T — faqat oldingiga reference
function merge<T, U = T>(a: T, b: U): T & U { ... }

// ❌ T default = U — U hali declared emas
// function merge<T = U, U>(a: T, b: U): T & U { ... }
```

**Interface va class larda ayniqsa muhim:** Funksiyada inference mavjud, shuning uchun default kam kerak bo'ladi. Lekin interface/class'da inference yo'q — har chaqiriqda type argument talab qilinadi. Default bo'lsa — argument'siz ishlatish imkoniyati paydo bo'ladi.

**Emit bosqichida yo'qoladi:** `interface ApiResponse<T = unknown>` → JS'da hech narsa. `class EventEmitter<E = Record<string, any[]>>` → JS'da `class EventEmitter`. Default mechanism faqat compile-time type resolution uchun xizmat qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Default type — API response
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

// Type berilmasa — T = unknown
const genericResponse: ApiResponse = {
  data: { anything: "goes" },
  status: 200,
  message: "OK",
};

// Type berilsa — T = User
interface User {
  name: string;
  email: string;
}

const userResponse: ApiResponse<User> = {
  data: { name: "Ali", email: "ali@test.com" },
  status: 200,
  message: "User fetched",
};

// 2. Default reference oldingi parameter'ga
function merge<T, U = T>(a: T, b: U): T & U {
  return { ...a, ...b } as T & U;
}

// U berilmagan — U = T
merge({ name: "Ali" }, { name: "Vali" });

// U explicit
merge<{ name: string }, { age: number }>({ name: "Ali" }, { age: 25 });

// 3. Generic class default
class Cache<K = string, V = unknown> {
  private store = new Map<K, V>();

  set(key: K, value: V): void { this.store.set(key, value); }
  get(key: K): V | undefined { return this.store.get(key); }
}

// Default'lar ishlatiladi
const simpleCache = new Cache();
// Cache<string, unknown>

// Type'lar explicit
const userCache = new Cache<number, User>();
// Cache<number, User>

// 4. Event emitter with default
class EventEmitter<Events extends Record<string, unknown[]> = Record<string, unknown[]>> {
  private handlers = new Map<keyof Events, Array<(...args: any[]) => void>>();

  on<K extends keyof Events>(event: K, handler: (...args: Events[K]) => void): void {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as (...args: any[]) => void);
    this.handlers.set(event, list);
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    this.handlers.get(event)?.forEach((fn) => fn(...args));
  }
}

// Type'siz — istalgan event
const generic = new EventEmitter();

// Type bilan — aniq event'lar
interface AppEvents extends Record<string, unknown[]> {
  login: [username: string];
  logout: [];
  error: [code: number, message: string];
}

const app = new EventEmitter<AppEvents>();
app.on("login", (username) => {
  // username: string — inferred from AppEvents["login"]
  console.log(`${username} logged in`);
});

app.emit("login", "Ali");  // ✅
// app.emit("login", 42);    // ❌ number !== string

// 5. Default type with constraint
function createList<T extends { id: string } = { id: string; name: string }>(
  items: T[]
): T[] {
  return [...items];
}

// Default ishlatiladi
const list1 = createList([
  { id: "1", name: "Ali" },
  { id: "2", name: "Vali" },
]);

// Explicit
const list2 = createList<{ id: string; price: number }>([
  { id: "1", price: 100 },
]);

// 6. React component props pattern
interface ButtonProps<T = unknown> {
  label: string;
  onClick: (value?: T) => void;
  value?: T;
}

const simpleButton: ButtonProps = {
  label: "Click",
  onClick: () => console.log("clicked"),
};

const numberButton: ButtonProps<number> = {
  label: "Increment",
  onClick: (val) => console.log(val),
  value: 42,
};
```

</details>

---

## const Type Parameters

### Nazariya

`const` type parameter (TS 5.0+) — `<const T>` sintaksisi bilan type parameter'ga **literal type inference**'ni majbur qiladi. Oddiy generic'da TypeScript type'ni kengaytiradi (widening) — `"hello"` → `string`, `42` → `number`. `const` bilan TypeScript literal type'ni saqlab qoladi.

Bu `as const`'ning funksiya chaqiruvidagi muqobili — foydalanuvchi `as const` yozmasdan ham literal type oladi.

```typescript
// Oddiy generic — widening bo'ladi
function routes<T extends readonly string[]>(paths: T): T {
  return paths;
}

const r1 = routes(["home", "about", "contact"]);
// T = string[] — widening bo'ldi
// r1[0] — string (aniq qiymat yo'q)

// const type parameter — literal saqlanadi
function routesConst<const T extends readonly string[]>(paths: T): T {
  return paths;
}

const r2 = routesConst(["home", "about", "contact"]);
// T = readonly ["home", "about", "contact"] — literal tuple
// r2[0] — "home" (aniq literal type)
```

**Qachon ishlatish:**

- Configuration object'lar (route'lar, enum'lar, literal mapping)
- Literal tuple'lar (fixed length arrays)
- API'ga configuration sifatida beriladigan object'lar
- `as const` yozmaslik imkonini berish

<details>
<summary><strong>Under the Hood</strong></summary>

`const` modifier type parameter declaration'da internal flag o'rnatadi. Bu flag kompilatorning **type widening** algoritmini o'zgartiradi.

**Oddiy `<T>` da:** `routes(["home", "about"])` chaqirilganda kompilator `T = string[]` deb infer qiladi — literal `"home"` va `"about"` `string`'ga widen bo'ladi. Array ham `string[]` bo'ladi, `readonly` emas.

**`<const T>` da:** Widening butunlay o'chiriladi — kompilator `T = readonly ["home", "about"]` deb infer qiladi. Bundan tashqari, array literal'lar `readonly tuple`'ga aylanadi — `string[]` emas, `readonly ["home", "about"]` bo'ladi.

```
Widening Behavior:

Oddiy <T>:
  ["home", "about"] → string[] (widened)
  "hello" → string (widened)
  42 → number (widened)

const <T>:
  ["home", "about"] → readonly ["home", "about"] (literal)
  "hello" → "hello" (literal)
  42 → 42 (literal)
```

**`const` va constraint alohida mexanizm'lar:** `<const T extends readonly string[]>` ikkita alohida narsa:
- `const` — inference behavior'ni o'zgartiradi (widening off)
- `extends readonly string[]` — constraint (T'ning upper bound)

Ikkalasi birgalikda ishlaydi, lekin alohida qoidalar.

**Object literal'larda:** `defineEndpoints({ getUser: { method: "GET" } })`'da `const T` bilan `method`'ning type'i `"GET"` (literal) bo'ladi, `const` siz — `string` (widened). Bu `as const` assertion'ni funksiya signature'ga ko'chirish effekti.

**`as const` bilan taqqoslash:**

| | `as const` | `<const T>` |
|---|---|---|
| Qayerda yoziladi | Chaqiruv joyida | Funksiya signature'da |
| Kim eslab qoladi | Foydalanuvchi | Funksiya author |
| Backward compatibility | Har TS versiyasida | TS 5.0+ |
| Natija | Literal type | Literal type |

`<const T>` foydalanuvchi uchun qulayroq — ular `as const` yozishni unutmaydi.

**Emit bosqichida o'chiriladi:** `function routesConst<const T>(paths: T)` → JS'da `function routesConst(paths)`. Runtime'da literal type va widened type orasida farq yo'q — `["home", "about"]` har ikkala holatda ham oddiy JavaScript array. Farq faqat compile-time type checking'da ko'rinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy vs const — widening farqi
function normal<T>(value: T): T {
  return value;
}

function constGen<const T>(value: T): T {
  return value;
}

const n1 = normal("hello");     // string (widened)
const c1 = constGen("hello");   // "hello" (literal)

const n2 = normal([1, 2, 3]);   // number[]
const c2 = constGen([1, 2, 3]); // readonly [1, 2, 3]

// 2. Route definition — literal saqlash
function defineRoutes<const T extends readonly { path: string; component: string }[]>(
  routes: T
): T {
  return routes;
}

const routes = defineRoutes([
  { path: "/home", component: "HomePage" },
  { path: "/about", component: "AboutPage" },
  { path: "/contact", component: "ContactPage" },
]);

// routes[0].path: "/home" (literal, not string)
// routes[0].component: "HomePage" (literal)

// 3. API endpoint configuration
function defineEndpoints<
  const T extends Record<string, { method: string; path: string }>
>(endpoints: T): T {
  return endpoints;
}

const api = defineEndpoints({
  getUser: { method: "GET", path: "/users/:id" },
  createUser: { method: "POST", path: "/users" },
  deleteUser: { method: "DELETE", path: "/users/:id" },
});

// api.getUser.method: "GET" (literal type, string emas)
// api.createUser.path: "/users" (literal type)

// 4. Form field configuration
function defineForm<
  const T extends ReadonlyArray<{
    name: string;
    type: "text" | "number" | "email";
    required: boolean;
  }>
>(fields: T): T {
  return fields;
}

const form = defineForm([
  { name: "username", type: "text", required: true },
  { name: "age", type: "number", required: false },
  { name: "email", type: "email", required: true },
]);

// form[0].type: "text" (literal)
// form[1].required: false (literal, not boolean)

// 5. Enum-like pattern
function enumOf<const T extends readonly string[]>(...values: T): Record<T[number], T[number]> {
  return values.reduce(
    (acc, value) => ({ ...acc, [value]: value }),
    {} as Record<T[number], T[number]>
  );
}

const Color = enumOf("red", "green", "blue");
// Color: { red: "red"; green: "green"; blue: "blue" }
type ColorType = keyof typeof Color; // "red" | "green" | "blue"

// 6. as const bilan taqqoslash
const r1 = normal(["a", "b"] as const);
// T = readonly ["a", "b"] — as const tufayli

const r2 = constGen(["a", "b"]);
// T = readonly ["a", "b"] — const parameter tufayli
// Foydalanuvchi as const yozmagan!

// 7. Widening yo'q — literal preserved
function createState<const T>(initial: T): { value: T } {
  return { value: initial };
}

const state1 = createState({ status: "idle", count: 0 });
// state1.value.status: "idle" (literal)
// state1.value.count: 0 (literal)

// const siz
function createStateNormal<T>(initial: T): { value: T } {
  return { value: initial };
}

const state2 = createStateNormal({ status: "idle", count: 0 });
// state2.value.status: string (widened)
// state2.value.count: number (widened)
```

</details>

---

## Generic Utility Patterns

### Nazariya

Generic'larning asl kuchi — real-world'da qayta ishlatiladigan (reusable) pattern'lar yaratishda. Quyidagi pattern'lar production code'da eng ko'p uchraydi:

1. **Result / Either** — xato bilan ishlash type-safe
2. **Collection** — generic container'lar (Stack, Queue, List)
3. **Repository** — database CRUD abstraction
4. **Factory** — type-safe object creation
5. **Builder** — fluent API bilan object qurish

<details>
<summary><strong>Under the Hood</strong></summary>

**Result pattern discriminated union sifatida:** `Result<T, E>` discriminated union (yoki "tagged union") — har variant `ok` field bilan farqlanadi. Kompilator `ok` field'ni discriminant sifatida taniydi va narrowing qiladi. `if (result.ok)` yozilganda TypeScript narrowing orqali `result.value` yoki `result.error`'ga aniq type beradi.

**Nested generic method'lar:** `Collection<T>`'da `map<U>(fn: (item: T) => U): Collection<U>` — method-level type parameter `U` class-level `T`'dan mustaqil. Har chaqiriqda `U` alohida infer bo'ladi. Bu pattern TypeScript'ning eng foydali xususiyatlaridan biri — input type'dan boshqa output type'ga transform qilish.

**Repository constraint propagation:** `Repository<T extends { id: string }>`'da constraint class bo'ylab tarqaladi — barcha method'larda `T`'ning `id` property'si borligi kafolatlangan. `create(data: Omit<T, "id">)` yaratganda, kompilator `Omit` utility type'ni ishlatib `id`'ni chiqarib tashlaydi va qolgan property'larni talab qiladi.

**Factory va construct signature:** `new (...args: any[]) => T` — construct signature. Kompilator `new Ctor(...)` call'da `T`'ni infer qiladi. Bu pattern `typeof SomeClass` bilan teng — class'ning constructor type'i.

**Barcha pattern'larda bir xil qoida:** generic type parameter faqat compile-time'da mavjud — runtime'da `Result<User, ApiError>` va `Result<Product, string>` orasida hech qanday farq yo'q. Discriminated union narrowing JavaScript'da oddiy property check sifatida ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ═════════════════════════════════════════════
// 1. Result / Either Pattern
// ═════════════════════════════════════════════

type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) return err("Division by zero");
  return ok(a / b);
}

const result = divide(10, 2);
if (result.ok) {
  console.log(result.value); // number — narrowed
} else {
  console.log(result.error); // string — narrowed
}

// Async Result
interface ApiError {
  code: number;
  message: string;
}

async function fetchUser(id: string): Promise<Result<User, ApiError>> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      return err({ code: response.status, message: "Failed" });
    }
    const user = await response.json();
    return ok(user);
  } catch (e) {
    return err({ code: 0, message: "Network error" });
  }
}

interface User {
  id: string;
  name: string;
}

// ═════════════════════════════════════════════
// 2. Collection Pattern (Generic List)
// ═════════════════════════════════════════════

interface Collection<T> {
  add(item: T): void;
  get(index: number): T | undefined;
  find(predicate: (item: T) => boolean): T | undefined;
  filter(predicate: (item: T) => boolean): T[];
  map<U>(transform: (item: T) => U): Collection<U>;  // method-level generic
  size: number;
}

class ArrayList<T> implements Collection<T> {
  private items: T[] = [];

  add(item: T): void { this.items.push(item); }
  get(index: number): T | undefined { return this.items[index]; }

  find(predicate: (item: T) => boolean): T | undefined {
    return this.items.find(predicate);
  }

  filter(predicate: (item: T) => boolean): T[] {
    return this.items.filter(predicate);
  }

  map<U>(transform: (item: T) => U): Collection<U> {
    const result = new ArrayList<U>();
    this.items.forEach((item) => result.add(transform(item)));
    return result;
  }

  get size(): number { return this.items.length; }
}

const users = new ArrayList<User>();
users.add({ id: "1", name: "Ali" });
users.add({ id: "2", name: "Vali" });

// Method-level generic — U class T'dan mustaqil
const names = users.map((u) => u.name); // Collection<string>

// ═════════════════════════════════════════════
// 3. Repository Pattern
// ═════════════════════════════════════════════

interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;
  update(id: string, data: Partial<Omit<T, "id">>): Promise<T | null>;
  delete(id: string): Promise<boolean>;
}

class InMemoryRepository<T extends { id: string }> implements Repository<T> {
  private store = new Map<string, T>();
  private counter = 0;

  async findById(id: string): Promise<T | null> {
    return this.store.get(id) ?? null;
  }

  async findAll(): Promise<T[]> {
    return Array.from(this.store.values());
  }

  async create(data: Omit<T, "id">): Promise<T> {
    const id = String(++this.counter);
    const entity = { ...data, id } as T;
    this.store.set(id, entity);
    return entity;
  }

  async update(id: string, data: Partial<Omit<T, "id">>): Promise<T | null> {
    const existing = this.store.get(id);
    if (!existing) return null;
    const updated = { ...existing, ...data };
    this.store.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.store.delete(id);
  }
}

interface Product {
  id: string;
  name: string;
  price: number;
}

async function example(): Promise<void> {
  const productRepo = new InMemoryRepository<Product>();

  // create: Omit<Product, "id"> — id kerak emas
  const phone = await productRepo.create({ name: "Phone", price: 999 });
  // phone: Product (id avtomatik qo'shildi)

  // update: Partial<Omit<Product, "id">> — ixtiyoriy update
  await productRepo.update(phone.id, { price: 899 });
}

// ═════════════════════════════════════════════
// 4. Factory Pattern
// ═════════════════════════════════════════════

function createFactory<T>(
  Constructor: new (...args: any[]) => T,
  ...defaultArgs: any[]
): () => T {
  return () => new Constructor(...defaultArgs);
}

class DatabaseConnection {
  constructor(public host: string, public port: number) {}
  connect(): void {
    console.log(`Connecting to ${this.host}:${this.port}`);
  }
}

const createConnection = createFactory(DatabaseConnection, "localhost", 5432);
const conn = createConnection(); // DatabaseConnection
conn.connect();

// ═════════════════════════════════════════════
// 5. Builder Pattern (Type-safe)
// ═════════════════════════════════════════════

class QueryBuilder<T> {
  private filters: Array<(item: T) => boolean> = [];

  where(predicate: (item: T) => boolean): QueryBuilder<T> {
    this.filters.push(predicate);
    return this; // Fluent API
  }

  build(): (items: T[]) => T[] {
    return (items) => items.filter((item) => this.filters.every((fn) => fn(item)));
  }
}

interface UserRecord {
  name: string;
  age: number;
  role: "admin" | "user";
}

const query = new QueryBuilder<UserRecord>()
  .where((u) => u.age >= 18)
  .where((u) => u.role === "admin")
  .build();

const admins = query([
  { name: "Ali", age: 25, role: "admin" },
  { name: "Vali", age: 17, role: "user" },
]);
// admins: adult admin users
```

</details>

---

## Edge Cases va Gotchas

Bu bo'limda generic'larning nozik va kutilmagan xatti-harakatlarini ko'ramiz.

### 1. Generic Class'da `keyof T` Faqat Public Member'lar

TypeScript'ning `keyof T` operatori class type'lar uchun **faqat public** member'larni qaytaradi. `private`, `protected`, va `#field` (ECMAScript native private) `keyof T` natijasiga kirmaydi.

```typescript
class Account {
  public name: string = "";
  public email: string = "";
  private balance: number = 0;
  protected userId: string = "";
  #secretKey: string = "";
}

type AccountKeys = keyof Account;
// "name" | "email"
// "balance", "userId", "#secretKey" — hech biri yo'q
```

**Nima uchun bu muhim:** Generic funksiya class instance bilan ishlaganda faqat public API'ga kirish huquqiga ega. Bu encapsulation'ni ta'minlaydi — private implementation detail'lar tashqi kod uchun ko'rinmaydi.

```typescript
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const account = new Account();
getProp(account, "name");    // ✅ public
getProp(account, "email");   // ✅ public
// getProp(account, "balance"); // ❌ "balance" keyof Account'da yo'q
```

**Structural typing bilan bypass:** Agar generic'ga class'ning structural shape'ini bersangiz (class'ning o'zini emas), private member'larga access bo'ladi — lekin bu anti-pattern:

```typescript
interface AccountShape {
  name: string;
  balance: number; // private'ni structural type bilan expose qilish
}

// ❌ Anti-pattern — encapsulation buziladi
type BadKeys = keyof AccountShape; // "name" | "balance"
```

### 2. Distributive Index Access — `T[K1 | K2]` Union'ga Distribute Bo'ladi

Index access type union key bilan chaqirilganda, natija ham union bo'ladi — har key uchun alohida lookup qilinadi va natijalar union qilinadi.

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

type NameOrAge = User["name" | "age"];
// = User["name"] | User["age"]
// = string | number

type AllValues = User[keyof User];
// = User["name"] | User["age"] | User["email"]
// = string | number | string
// = string | number (simplified)
```

Bu pattern `keyof` bilan juda kuchli — `T[keyof T]` object'ning barcha value type'larini union sifatida beradi. Mapped type'lar va conditional type'larda ayniqsa ko'p uchraydi.

### 3. Type Parameter Dependency Order — `<T, K extends keyof T>` `T` Avval

Type parameter'larda dependency bor — keyingisi oldingisiga reference qilishi mumkin, lekin teskari emas. `<T, K extends keyof T>`'da `T` avval aniqlanishi kerak, keyin `K` shu constraint ichida tekshiriladi.

```typescript
// ✅ To'g'ri order
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// ❌ Invalid — K declaration paytida T hali ma'lum emas
// function getPropBad<K extends keyof T, T>(obj: T, key: K): T[K] { ... }
```

Bu cheklov **explicit type argument**'larda ham amal qiladi. Agar siz `K`'ni explicit berib, `T`'ni inference'ga qoldirishga urinsangiz — TypeScript'ning hozirgi versiyalarida bu ishlamaydi (**partial type argument inference** yo'q):

```typescript
const user = { name: "Ali", age: 25 };

// ❌ Xohlayotgan: T'ni inference'ga qoldirish, K explicit
// getProp<_, "name">(user, "name");

// ✅ Ikkalasi inference (oddiy yondashuv)
const name = getProp(user, "name"); // T va K ikkalasi inferred

// ✅ Ikkalasi explicit
const name2 = getProp<{ name: string; age: number }, "name">(user, "name");
```

### 4. `const` Type Parameter va Primitive Widening

`<const T>` type parameter (TS 5.0+) widening'ni o'chiradi, lekin natija **`readonly` tuple/object** bo'lishi mumkin. Bu kutilmagan bo'lishi mumkin — agar siz mutable natija kutayotgan bo'lsangiz.

```typescript
function asTuple<const T extends readonly unknown[]>(arr: T): T {
  return arr;
}

const t = asTuple([1, 2, 3]);
// t: readonly [1, 2, 3]
// t.push(4); // ❌ readonly'da push yo'q!
```

**Yechim:** Agar mutable natija kerak bo'lsa, `const` o'rniga oddiy generic ishlatish yoki natijani explicit cast qilish:

```typescript
function asNormalTuple<T extends readonly unknown[]>(arr: T): T {
  return arr;
}

const mutable = [...asTuple([1, 2, 3])]; // number[] — spread bilan mutable
mutable.push(4); // ✅
```

**Primitive value'larda ham o'zgaradi:**

```typescript
function createStatus<const T>(value: T): T {
  return value;
}

const s1 = createStatus("idle");      // s1: "idle" (literal)
const s2 = createStatus(42);           // s2: 42 (literal)

// const siz
function createStatusNormal<T>(value: T): T {
  return value;
}

const s3 = createStatusNormal("idle"); // s3: string (widened)
```

### 5. Generic Method vs Class Generic — Method O'zi Generic Bo'lishi Mumkin

Class'ning type parameter (`<T>`) va method'ning type parameter (`<U>`) — ikkita alohida parameter. Class T klass'da fixed, lekin method `<U>` har chaqiriqda yangi.

```typescript
class Container<T> {
  constructor(private value: T) {}

  // Class-level generic — T class'dan keladi
  getValue(): T {
    return this.value;
  }

  // Method-level generic — U class T'dan mustaqil
  transform<U>(fn: (value: T) => U): U {
    return fn(this.value);
  }

  // Ikkala: T class'dan, U method'dan
  mapToPair<U>(fn: (value: T) => U): [T, U] {
    return [this.value, fn(this.value)];
  }
}

const container = new Container(42); // T = number
const strValue = container.transform((n) => n.toString()); // U = string
const pair = container.mapToPair((n) => n > 10); // [number, boolean]
```

**Nima uchun muhim:** Agar method'ning o'z type parameter'i bo'lmasa, method output class'ning T'sidan boshqa type bo'la olmaydi. Method-level generic — output type'ni kirish type'dan farqli qilish imkonini beradi.

Bu pattern ayniqsa collection class'larda muhim — `map<U>()`, `filter()`, `reduce<U>()` kabi method'lar class T'dan boshqa type'ga transform qiladi.

---

## Common Mistakes

### ❌ Xato 1: Keraksiz type parameter

```typescript
// ❌ T faqat bitta joyda ishlatilgan — generic kerak emas
function greet<T extends string>(name: T): string {
  return `Hello, ${name}`;
}
```

**✅ To'g'ri usul:**

```typescript
// ✅ Generic kerak emas — oddiy string
function greet(name: string): string {
  return `Hello, ${name}`;
}

// ✅ Generic kerak — T bir nechta joyda ishlatiladi (input va output bog'liq)
function identity<T>(value: T): T {
  return value;
}
```

**Nima uchun:** Generic type parameter ma'noli faqat **ikki yoki undan ko'p joyda** ishlatilganda — input/output bog'liqligini ifodalaganda. Bitta joyda ishlatilsa — oddiy type yetarli.

---

### ❌ Xato 2: `T extends any` — constraint emas

```typescript
// ❌ T extends any — hech narsa cheklamaydi
function process<T extends any>(value: T): T {
  return value;
}
```

**✅ To'g'ri usul:**

```typescript
// ✅ Constraint kerak emas — constraintsiz yozing
function identity<T>(value: T): T {
  return value;
}

// ✅ Agar constraint kerak bo'lsa — aniq shape belgilang
function process<T extends { serialize(): string }>(value: T): string {
  return value.serialize();
}
```

**Nima uchun:** `extends any` — hech narsa cheklamaydi va constraintsiz `<T>` bilan teng. Bu keraksiz boilerplate. Agar constraint kerak bo'lsa — aniq type bering.

---

### ❌ Xato 3: `Object.keys()` va `keyof T` nomosligi

```typescript
interface Config {
  host: string;
  port: number;
}

function printConfig<T extends Config>(config: T): void {
  // ❌ Object.keys string[] qaytaradi, keyof T emas
  Object.keys(config).forEach((key) => {
    // key: string — keyof T emas
    // config[key]; // ❌ Element implicitly has an 'any' type
  });
}
```

**✅ To'g'ri usul:**

```typescript
function printConfig<T extends Config>(config: T): void {
  // ✅ Type assertion bilan
  (Object.keys(config) as Array<keyof T>).forEach((key) => {
    console.log(key, config[key]);
  });

  // ✅ Yoki explicit keys ro'yxati
  const keys: (keyof Config)[] = ["host", "port"];
  keys.forEach((key) => {
    console.log(key, config[key]);
  });
}
```

**Nima uchun:** `Object.keys()` har doim `string[]` qaytaradi — `(keyof T)[]` emas. Sabab: TypeScript structural typing ishlatadi — `T`'da `Config`'dan **ko'proq** property bo'lishi mumkin. `Object.keys` barcha runtime key'larni beradi — TS compile-time'da ularning hammasini bilmaydi.

---

### ❌ Xato 4: Generic narrowing'da return type yo'qolishi

```typescript
function process<T extends string | number>(value: T): T {
  if (typeof value === "string") {
    // ❌ value.toUpperCase() string qaytaradi, T emas
    return value.toUpperCase() as T;
    // Agar T = "hello" literal bo'lsa, "HELLO" qaytadi — T emas
  }
  return value;
}
```

**✅ To'g'ri usul:**

```typescript
// ✅ Overload yoki union return bilan
function process(value: string): string;
function process(value: number): number;
function process(value: string | number): string | number {
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  return value;
}
```

**Nima uchun:** Generic `T` narrowing'da muammo chiqaradi — `typeof value === "string"` qilganda TS `value: string` deb biladi, lekin `T` hali ham `string | number`. `string`'ni `T`'ga assign qilish xavfli. Generic narrowing kerak bo'lganda overload yoki conditional type ishlating.

---

### ❌ Xato 5: Ortiqcha murakkab generic

```typescript
// ❌ Haddan tashqari murakkab — o'qib bo'lmaydi
function getData<
  T extends Record<string, unknown>,
  K extends keyof T,
  V extends T[K],
  R extends V extends string ? string[] : V extends number ? number[] : never
>(obj: T, key: K): R {
  return [] as unknown as R;
}
```

**✅ To'g'ri usul:**

```typescript
// ✅ Sodda va tushunarli
function getData<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Agar murakkab logic kerak — overload ishlatish yaxshiroq
function toArray(value: string): string[];
function toArray(value: number): number[];
function toArray(value: string | number): string[] | number[] {
  return typeof value === "string" ? value.split("") : [value];
}
```

**Nima uchun:** Generic ko'p type parameter va nested conditional bilan tezda o'qib bo'lmaydigan bo'ladi. **Qoida:** agar generic'ni tushuntirish uchun 1 daqiqa kerak bo'lsa — soddalash kerak.

---

## Amaliy Mashqlar

### Mashq 1: Generic Pair (Oson)

**Savol:** `Pair<T, U>` interface va `makePair` funksiya yozing. `swap()` method ham bo'lsin — `T` va `U` o'rni almashsin.

```typescript
// const p = makePair("hello", 42);
// p.first → string
// p.second → number
// p.swap() → Pair<number, string>
```

<details>
<summary>Javob</summary>

```typescript
interface Pair<T, U> {
  first: T;
  second: U;
  swap(): Pair<U, T>;
}

function makePair<T, U>(first: T, second: U): Pair<T, U> {
  return {
    first,
    second,
    swap() {
      return makePair(second, first);
      // ✅ return type: Pair<U, T>
    },
  };
}

const p = makePair("hello", 42);
p.first;            // string
p.second;           // number
const swapped = p.swap();
swapped.first;      // number (42)
swapped.second;     // string ("hello")
```

**Tushuntirish:** `makePair` ikki type parameter qabul qiladi — `T` va `U`. `swap()` natijasi `Pair<U, T>` — type parameter'lar teskari tartibda. TypeScript inference orqali barcha type'larni to'g'ri aniqlaydi.

</details>

---

### Mashq 2: Type-Safe groupBy (Oson)

**Savol:** `groupBy` funksiyasining type'ini yozing — array element'larini berilgan key bo'yicha guruhlaydi.

```typescript
// function groupBy<???>(arr: ???, key: ???): ???

const users = [
  { name: "Ali", role: "admin" },
  { name: "Vali", role: "user" },
  { name: "Soli", role: "admin" },
];

const grouped = groupBy(users, "role");
```

<details>
<summary>Javob</summary>

```typescript
function groupBy<T, K extends keyof T>(
  arr: T[],
  key: K
): Record<string, T[]> {
  return arr.reduce((groups, item) => {
    const groupKey = String(item[key]);
    if (!groups[groupKey]) {
      groups[groupKey] = [];
    }
    groups[groupKey].push(item);
    return groups;
  }, {} as Record<string, T[]>);
}

const users = [
  { name: "Ali", role: "admin" },
  { name: "Vali", role: "user" },
  { name: "Soli", role: "admin" },
];

const grouped = groupBy(users, "role");
// grouped: Record<string, { name: string; role: string }[]>

// groupBy(users, "invalid"); // ❌ "invalid" keyof'da yo'q
```

**Tushuntirish:**

- `T` — array element type (inference orqali aniqlanadi)
- `K extends keyof T` — faqat T'ning mavjud key'lari qabul qilinadi
- Return: `Record<string, T[]>` — groups
- `String(item[key])` — key qiymati string'ga aylantiriladi

</details>

---

### Mashq 3: Generic Memoize (O'rta)

**Savol:** `memoize` funksiyasini generic bilan yozing — istalgan funksiyani qabul qilib, natijalarni cache qiladigan versiyasini qaytarsin. Original funksiyaning parameter va return type'lari saqlansin.

```typescript
function expensive(n: number): string {
  console.log("Computing...");
  return n.toString();
}

const memoized = memoize(expensive);
memoized(42); // "Computing..." → "42"
memoized(42); // "42" (cache'dan)
```

<details>
<summary>Javob</summary>

```typescript
function memoize<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  const cache = new Map<string, ReturnType<T>>();

  return (...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

function expensive(n: number): string {
  console.log("Computing...");
  return n.toString();
}

const memoized = memoize(expensive);
memoized(42); // Console: "Computing..." → "42"
memoized(42); // "42" (cache'dan)
memoized(10); // Console: "Computing..." → "10"
```

**Tushuntirish:**

- `T extends (...args: any[]) => any` — T istalgan funksiya bo'lishi kerak
- `Parameters<T>` — T'ning parameter type'larini tuple sifatida oladi
- `ReturnType<T>` — T'ning return type'ini oladi
- Cache key — `JSON.stringify(args)` (sodda, lekin object argument'larda circular reference bilan muammo bor)

Bu pattern `lodash.memoize`, React'ning `useMemo` kabi library'larda ishlatiladi.

</details>

---

### Mashq 4: Nested Generic Method (Qiyin)

**Savol:** `Collection<T>` class yozing — `map<U>()` method bilan. Method T'dan boshqa U type'ga transform qilsin. Class T'si saqlanmasdan, method'ning o'z U'si ishlatilsin.

```typescript
// const numbers = new Collection([1, 2, 3]);
// const strings = numbers.map(n => n.toString()); // Collection<string>
// const booleans = numbers.map(n => n > 1);        // Collection<boolean>
```

<details>
<summary>Javob</summary>

```typescript
class Collection<T> {
  constructor(private items: T[]) {}

  // Method-level generic — U class T'dan mustaqil
  map<U>(transform: (item: T) => U): Collection<U> {
    return new Collection(this.items.map(transform));
  }

  filter(predicate: (item: T) => boolean): Collection<T> {
    return new Collection(this.items.filter(predicate));
  }

  toArray(): T[] {
    return [...this.items];
  }

  get size(): number {
    return this.items.length;
  }
}

// Ishlatish
const numbers = new Collection([1, 2, 3, 4, 5]);

// T = number, method U = string
const strings = numbers.map((n) => n.toString());
// strings: Collection<string>

// T = number, method U = boolean
const bigs = numbers.map((n) => n > 2);
// bigs: Collection<boolean>

// Chain — har map'da yangi U
const processed = numbers
  .filter((n) => n % 2 === 0)  // Collection<number>
  .map((n) => n * 2)            // Collection<number>
  .map((n) => `value:${n}`);    // Collection<string>
```

**Tushuntirish:**

- **Class-level `T`** — class instance create paytida aniqlanadi (`new Collection([1, 2, 3])` → `T = number`)
- **Method-level `U`** — har `map()` chaqiriqda alohida infer bo'ladi
- **Method'ning o'z scope'i** — `U` class T'ga bog'liq emas, method signature'da alohida declared
- **Chain bo'ylab o'zgarish** — har transform'da type o'zgaradi: `number` → `number` → `string`

Bu pattern JavaScript'dagi `Array.prototype.map`, `Promise.prototype.then`, RxJS `Observable.pipe` kabi joylarda ishlatiladi. Nested generic — TypeScript'ning eng kuchli xususiyatlaridan biri.

</details>

---

### Mashq 5: Type-Safe Event Emitter (Qiyin)

**Savol:** `EventEmitter<Events>` class yozing — events map type sifatida berilsin. `on` va `emit` method'lar faqat belgilangan event'larga ruxsat bersin, va callback parameter'lari avtomatik infer bo'lsin.

```typescript
interface AppEvents {
  login: [username: string];
  logout: [];
  error: [code: number, message: string];
}

const emitter = new EventEmitter<AppEvents>();
emitter.on("login", (username) => console.log(username));  // username: string
emitter.emit("error", 500, "Server error");                 // ✅
// emitter.emit("login", 42);                               // ❌
// emitter.on("unknown", ...);                              // ❌
```

<details>
<summary>Javob</summary>

```typescript
type EventMap = Record<string, unknown[]>;

class EventEmitter<Events extends EventMap> {
  private handlers = new Map<keyof Events, Array<(...args: any[]) => void>>();

  on<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): void {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as (...args: any[]) => void);
    this.handlers.set(event, list);
  }

  emit<K extends keyof Events>(
    event: K,
    ...args: Events[K]
  ): void {
    const list = this.handlers.get(event);
    list?.forEach((fn) => fn(...args));
  }

  off<K extends keyof Events>(event: K): void {
    this.handlers.delete(event);
  }
}

interface AppEvents {
  login: [username: string];
  logout: [];
  error: [code: number, message: string];
}

const emitter = new EventEmitter<AppEvents>();

emitter.on("login", (username) => {
  // username: string — inferred from AppEvents["login"]
  console.log(`User ${username} logged in`);
});

emitter.on("error", (code, message) => {
  // code: number, message: string
  console.error(`Error ${code}: ${message}`);
});

emitter.emit("login", "Ali");              // ✅
emitter.emit("error", 500, "Not found");   // ✅
emitter.emit("logout");                     // ✅ (bo'sh tuple)

// emitter.emit("login", 42);                // ❌ 42 !== string
// emitter.emit("unknown");                   // ❌ "unknown" keyof'da yo'q
// emitter.emit("error", "bad");              // ❌ birinchi argument number bo'lishi kerak
```

**Tushuntirish:**

- `Events extends EventMap` — constraint, har event tuple type bilan bog'langan
- `K extends keyof Events` — faqat declared event'larga ruxsat
- `Events[K]` — index access type'dan event'ning parameter tuple'ini oladi
- `(...args: Events[K]) => void` — callback tuple'dan rest parameter sifatida ishlatiladi
- `tuple sintaksisi` — `[username: string]` — named tuple element (TS 4.0+)

Bu pattern Node.js `EventEmitter`'ning type-safe versiyasi. Real-world'da `mitt`, `typed-emitter` kabi library'larda ishlatiladi. Nested generic + keyof + index access — hamma TypeScript xususiyatlarini birlashtiradi.

</details>

---

## Xulosa

Bu bo'limda TypeScript Generics'ning barcha asosiy tushunchalarini o'rgandik:

**Asosiy tushunchalar:**

- **Generics nima** — type-level abstraction, `<T>` type parameter bilan reusable va type-safe kod yozish
- **Generic functions** — funksiya input/output type'larini bog'lash
- **Inference** — TypeScript argument'lardan type parameter'ni avtomatik aniqlaydi; contextual typing callback'larda
- **Explicit type arguments** — inference ishlamaganda yoki aniqlik uchun `<Type>` yozish
- **Generic interfaces va classes** — reusable data structure va contract'lar
- **Instantiation expression** (TS 4.7+) — `const NumStack = Stack<number>` pre-binding pattern
- **Constraints** — `extends` bilan type parameter'ni cheklash, `T` haqida kafolat olish
- **`keyof` + generics** — type-safe property access, `<T, K extends keyof T>` pattern
- **Index Access Types** — `T[K]`, `T[keyof T]`, `T[number]` — type darajasida property type olish
- **Distributive index access** — `T[K1 | K2]` → `T[K1] | T[K2]`
- **`typeof` operator** — qiymatdan type olish, `ReturnType<typeof fn>` pattern
- **Multiple va default type parameters** — murakkab generic'lar, default qiymatlar
- **`const` type parameters** (TS 5.0+) — literal type inference, widening'ni o'chirish
- **Utility patterns** — Result, Collection, Repository, Factory, Builder

**Umumiy takeaway'lar:**

1. **Kamroq — yaxshi.** Har type parameter ma'no berishi kerak. Bitta joyda ishlatilgan generic keraksiz.
2. **Constraint'lar — ishonch manbai.** `<T>` haqida hech narsa bilmaymiz; `<T extends X>` bilan `X`'ning barcha xususiyatlari mavjud.
3. **`keyof T` faqat public.** Class type'larda private/protected `keyof`'da ko'rinmaydi — encapsulation'ni ta'minlaydi.
4. **Inference → explicit → default.** Uch bosqichli fallback — TypeScript type'ni shu tartibda aniqlaydi.
5. **Runtime'da generic yo'q.** Barcha type parameter'lar compile-time'da o'chiriladi. External data'ni validate qilish kerak.
6. **Generic method vs class generic.** Method'ning o'z type parameter'i class T'sidan mustaqil — transform pattern'lar uchun muhim.
7. **`const` type parameter** — `as const` yozishni unutmaslik uchun.

**Cross-references:**

- **[06-type-narrowing.md](06-type-narrowing.md)** — Type guard'larda generic, `is` keyword, `asserts`
- **[07-functions.md](07-functions.md)** — Generic functions, callback type inference, variance
- **[09-advanced-generics.md](09-advanced-generics.md)** — Conditional types, `infer`, recursive generics
- **[10-classes.md](10-classes.md)** — Class visibility (`public`/`private`/`protected`/`#field`)
- **[12-conditional-types.md](12-conditional-types.md)** — `T extends U ? X : Y` chuqur
- **[13-mapped-types.md](13-mapped-types.md)** — `{ [K in keyof T]: ... }` chuqur
- **[15-utility-types.md](15-utility-types.md)** — `Pick`, `Omit`, `Partial`, `ReturnType`, `Parameters`
- **[25-type-compatibility.md](25-type-compatibility.md)** — Variance annotations `in`/`out` (TS 4.7+)

---

**Keyingi bo'lim:** [09-advanced-generics.md](09-advanced-generics.md) — Advanced Generics: conditional types (`T extends U ? X : Y`), `infer` keyword, distributive conditional types, recursive types, variadic tuple types, higher-kinded types workarounds, type-level programming.
