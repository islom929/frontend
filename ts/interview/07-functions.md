# Interview: Functions TypeScript da

> Function type annotations, overloads, this parameter, void vs undefined, function assignability, rest/spread bo'yicha interview savollari.

---

## Nazariy savollar

### 1. TS da funksiya return type ni yozish kerakmi yoki inference ga qo'yish kerakmi?

<details>
<summary>Javob</summary>

TS funksiya return type ni **o'zi aniqlaydi** (infer qiladi). Ko'p hollarda explicit yozish shart emas. Lekin ba'zi holatlarda yozish **tavsiya** etiladi:

| Holat | Yozish kerakmi? | Sabab |
|-------|-----------------|-------|
| Public API / library export | ✅ Ha | Aniq contract, refactor xavfsizligi |
| Recursive function | ✅ Ba'zan | TS infer qila olmaydi |
| JSON.parse, any qaytaruvchi API | ✅ Ha | any ni cheklash |
| Internal / private function | ❌ Yo'q | Inference yetarli |
| Oddiy arrow function | ❌ Yo'q | Code toza qoladi |

```typescript
// ❌ Inference yetarli
function add(a: number, b: number) { return a + b; } // return: number

// ✅ Public API — yozish tavsiya
export function parseConfig(raw: string): Config {
  return JSON.parse(raw); // JSON.parse any qaytaradi — explicit type kerak
}
```

`isolatedDeclarations` (TS 5.5+) mode da **export** funksiyalarda return type majburiy.

</details>

### 2. Optional parameter (`?`) va default parameter (`= value`) farqi nima?

<details>
<summary>Javob</summary>

```typescript
// Optional — berilmasa undefined
function greet(name: string, prefix?: string): string {
  return `${prefix ?? "Hello"}, ${name}!`;
}
greet("Ali"); // prefix = undefined → "Hello, Ali!"

// Default — berilmasa standart qiymat
function greetDefault(name: string, prefix: string = "Hello"): string {
  return `${prefix}, ${name}!`;
}
greetDefault("Ali"); // prefix = "Hello" → "Hello, Ali!"
```

| | Optional (`?`) | Default (`= value`) |
|---|---|---|
| Berilmasa | `undefined` | default qiymat |
| `undefined` berilsa | `undefined` qoladi | **default ishlatiladi** |
| Type | `T \| undefined` | `T` |

Muhim farq — `undefined` explicit berilganda:

```typescript
function test(x?: number) { return x; }
test(undefined); // undefined

function testDefault(x: number = 42) { return x; }
testDefault(undefined); // 42 — default ishlatiladi
```

</details>

### 3. Function overload nima? Qachon ishlatish kerak?

<details>
<summary>Javob</summary>

Function overload — bitta funksiyaga **bir nechta signature** yozish. Overload kerak bo'lgan holat: **return type parametrga qarab o'zgarsa**.

```typescript
// Overload signatures
function getElementById(id: string): HTMLElement;
function getElementById(id: string, strict: false): HTMLElement | null;
// Implementation
function getElementById(id: string, strict: boolean = true): HTMLElement | null {
  const el = document.getElementById(id);
  if (strict && !el) throw new Error(`#${id} not found`);
  return el;
}

const el1 = getElementById("app");        // HTMLElement
const el2 = getElementById("app", false); // HTMLElement | null
```

**Qoida:** Agar barcha overload lar **bir xil return type** qaytarsa → overload kerak emas, union ishlatilsin. Agar return type **parametrga bog'liq** o'zgarsa → overload kerak.

</details>

### 4. `void` va `undefined` farqi nima? Callback da void nima uchun maxsus ishlaydi?

<details>
<summary>Javob</summary>

`void` — return value ni **ishlatmaslik kerak** degan signal. Callback context da void ning maxsus xususiyati: **void return type li callback istalgan qiymat qaytarishi mumkin**.

```typescript
type VoidCallback = () => void;

const fn1: VoidCallback = () => 42;      // ✅ 42 qaytarildi, lekin ignored
const fn2: VoidCallback = () => "hello"; // ✅ string ham OK
```

Bu nima uchun bunday:

```typescript
[1, 2, 3].forEach((n) => numbers.push(n));
// push() returns number — lekin forEach void callback kutadi
// Agar void strict bo'lganida — bu pattern ishlamaydi
```

**Lekin** function declaration da `void` strict:

```typescript
function doSomething(): void {
  // return 42; // ❌ Error — function declaration da void strict
}
```

</details>

### 5. `this` parameter nima? JS da compile bo'lganda nima bo'ladi?

<details>
<summary>Javob</summary>

`this` parameter — funksiya signaturasida birinchi parametr sifatida yoziladi va `this` ning type ini belgilaydi. JS ga compile bo'lganda **butunlay o'chiriladi**.

```typescript
class Timer {
  seconds = 0;
  tick(this: Timer): void {
    this.seconds++;
  }
}

const timer = new Timer();
// setInterval(timer.tick, 1000);
// ❌ Error: 'this' context of type 'void' is not assignable to 'Timer'

// ✅ Arrow function this ni bind qiladi
setInterval(() => timer.tick(), 1000);
```

`this` parameter nima uchun kerak — method reference berilganda `this` yo'qolishi muammosini **compile-time** da topish. JS da bu xato faqat **runtime** da ko'rinadi.

</details>

### 6. Function assignability qanday ishlaydi? Kamroq parametrli funksiya nima uchun mos keladi?

<details>
<summary>Javob</summary>

TS da funksiya assignability uchta qoida:

**1. Parameter soni — kamroq mos:**

```typescript
type ThreeParams = (a: number, b: string, c: boolean) => void;
const fn: ThreeParams = (a) => console.log(a); // ✅ 1 param — mos
// JS da callback lar ortiqcha parametrlarni ignore qiladi
// forEach((value, index, array) => ...) — ko'pchilik faqat value ishlatadi
```

**2. Parameter types — contravariant (`strictFunctionTypes`):**

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }

type AnimalFn = (a: Animal) => void;
type DogFn = (d: Dog) => void;

const animalFn: AnimalFn = (a) => console.log(a.name);
const dogFn: DogFn = animalFn; // ✅ Animal handler Dog ni ham qabul qiladi
```

**3. Return type — covariant:**

```typescript
type ReturnAnimal = () => Animal;
type ReturnDog = () => Dog;
const getAnimal: ReturnAnimal = () => new Dog(); // ✅ Dog ⊂ Animal
```

</details>

### 7. Generic function va overload — qachon qaysi birini ishlatish kerak?

<details>
<summary>Javob</summary>

**Generic** — bir xil mantiq turli type lar bilan:

```typescript
function first<T>(arr: T[]): T | undefined {
  return arr[0]; // Bir xil mantiq: string[], number[] — barchasi uchun arr[0]
}
```

**Overload** — har xil mantiq turli type lar bilan:

```typescript
function serialize(value: string): string;
function serialize(value: number): string;
function serialize(value: Date): string;
function serialize(value: string | number | Date): string {
  if (typeof value === "string") return `"${value}"`;
  if (typeof value === "number") return value.toFixed(2);
  return value.toISOString(); // Date uchun boshqa mantiq
}
```

| Holat | Generic | Overload |
|-------|---------|----------|
| Mantiq bir xil, type farq | ✅ | ❌ |
| Mantiq va return type farq | ❌ | ✅ |
| Yangi type qo'shish | Hech narsa o'zgarmaydi | Yangi overload kerak |
| Soddalik | ✅ | ❌ |

**Qoida:** Avval generic bilan yechishga harakat qiling. Agar return type parametrga qarab farq qilsa — overload ishlating.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Overload — output savoli (Daraja: Middle)

**Savol:** Output va TS type larini ayting:

```typescript
function process(value: string): string[];
function process(value: string[]): string;
function process(value: string | string[]): string | string[] {
  if (typeof value === "string") return value.split(",");
  return value.join(",");
}

const a = process("a,b,c");
const b = process(["x", "y"]);

console.log(a);
console.log(b);
console.log(typeof a);
console.log(typeof b);
```

<details>
<summary>Yechim</summary>

```
["a", "b", "c"]
x,y
object
string
```

- `process("a,b,c")` — Overload 1 match: `string` → `string[]` → `"a,b,c".split(",")` → `["a", "b", "c"]`
- `process(["x", "y"])` — Overload 2 match: `string[]` → `string` → `["x", "y"].join(",")` → `"x,y"`
- TS type lari: `a: string[]`, `b: string`
- `typeof ["a","b","c"]` → `"object"` (array = object), `typeof "x,y"` → `"string"`

Overload resolution yuqoridan pastga ishlaydi — birinchi mos kelgan signature tanlanadi.

</details>

### 2. void callback — output savoli (Daraja: Middle)

**Savol:** Output ni ayting va nima uchun ekanini tushuntiring:

```typescript
type Callback = () => void;

function runCallbacks(callbacks: Callback[]): boolean[] {
  return callbacks.map((cb) => {
    const result = cb();
    return result === undefined;
  });
}

const cbs: Callback[] = [
  () => 42,
  () => "hello",
  () => undefined,
  () => {},
];

console.log(runCallbacks(cbs));
```

<details>
<summary>Yechim</summary>

```
[false, false, true, false]
```

`Callback` type i `() => void` — void callback **istalgan qiymat qaytarishi mumkin** (TS taqiqlamaydi). Lekin void faqat **TS type system** da — **runtime ga ta'sir qilmaydi**:

- `() => 42` — runtime da `42` qaytaradi → `42 === undefined` → `false`
- `() => "hello"` — `"hello"` → `false`
- `() => undefined` — `undefined` → `true`
- `() => {}` — bo'sh object `{}` → `false`

**Muhim:** `void` — "return value **ahamiyatsiz**" degani, "hech narsa qaytarmaydi" degani **emas**. TS void callback ga istalgan return ruxsat beradi (callback assignability uchun), lekin runtime da haqiqiy qiymat qaytariladi.

</details>

### 3. Overload xato — toping va tuzating (Daraja: Middle+)

**Savol:** Bu kodda compile-time xato bor. Toping va tuzating:

```typescript
function createUser(name: string, age: number): { name: string; age: number };
function createUser(name: string): { name: string };
function createUser(name: string, age?: number) {
  return age !== undefined ? { name, age } : { name };
}

const user1 = createUser("Ali", 25);
const user2 = createUser("Vali");

console.log(user1.age);
console.log(user2.age);
```

<details>
<summary>Yechim</summary>

`user2.age` da **compile-time xato** — ikkinchi overload return type `{ name: string }` da `age` property yo'q.

```typescript
console.log(user1.age); // ✅ 25 — Overload 1: { name: string; age: number }
console.log(user2.age); // ❌ Property 'age' does not exist on type '{ name: string }'
```

Bu overload ning kuchi — return type parametrga qarab **farqlanadi**. Agar overload bo'lmaganida:

```typescript
function createUserSimple(name: string, age?: number) {
  return age !== undefined ? { name, age } : { name };
}
// Return type: { name: string; age: number } | { name: string }
// user.age uchun narrowing kerak — yomonroq DX
```

</details>

### 4. Overload vs conditional return type (Daraja: Middle+)

**Savol:** Quyidagi overloaded funksiyani **bitta** generic funksiya bilan qayta yozing. Conditional return type ishlatib, bir xil type safety saqlang:

```typescript
// Overloaded versiya
function parse(input: string): number;
function parse(input: number): string;
function parse(input: string | number): string | number {
  if (typeof input === "string") return parseInt(input, 10);
  return String(input);
}

const a = parse("42");  // number
const b = parse(42);    // string

// Generic conditional versiyasini yozing:
// function parse<T extends string | number>(input: T): ???
```

<details>
<summary>Yechim</summary>

```typescript
// Conditional return type bilan
function parse<T extends string | number>(
  input: T
): T extends string ? number : string {
  if (typeof input === "string") {
    return parseInt(input, 10) as T extends string ? number : string;
  }
  return String(input) as T extends string ? number : string;
}

const a = parse("42");  // number ✅
const b = parse(42);    // string ✅
```

**Trade-off tahlili:**

| | Overload | Conditional Return |
|---|---|---|
| O'qilishi | ✅ Oson | ❌ Murakkab |
| `as` assertion kerak | ❌ | ✅ (implementation da) |
| Yangi variant qo'shish | Yangi overload | Conditional kengaytirish |
| Inferred type | ✅ Aniq | ✅ Aniq |
| Generic bilan ishlashi | ❌ | ✅ |

**Qachon qaysi biri:**

- **Overload** — 2-3 variant, o'qilishi muhim, generic shart emas
- **Conditional return** — generic kontekst, dynamic type mapping, ko'p variant

**Muhim:** Conditional return type da implementation ichida `as` assertion kerak — TS generic body da conditional type ni resolve qila olmaydi (deferred conditional). Bu ma'lum cheklov.

</details>

### 5. Overload resolution order — xatoni toping (Daraja: Senior)

**Savol:** Bu overload lar nima uchun kutilganday ishlamaydi? Tuzating:

```typescript
function format(input: string | number): string;
function format(input: string): string[];
function format(input: string | number): string | string[] {
  if (typeof input === "string") return input.split(",");
  return String(input);
}

const result = format("a,b,c");
// Kutilgan: string[] (split natijasi)
// Haqiqiy TS type: ???
```

<details>
<summary>Yechim</summary>

**Muammo:** TS overload larni **yuqoridan pastga** tekshiradi va **birinchi mos kelganini** tanlaydi. `format("a,b,c")` — birinchi overload `(input: string | number): string` ga mos keladi (`string ⊂ string | number`). Shuning uchun return type `string` — `string[]` emas.

```typescript
const result = format("a,b,c"); // TS type: string — string[] EMAS!
```

**Tuzatish — aniq overload ni birinchi qo'yish:**

```typescript
function format(input: string): string[];           // Aniq — BIRINCHI
function format(input: number): string;              // Aniq
function format(input: string | number): string | string[] {
  if (typeof input === "string") return input.split(",");
  return String(input);
}

const result = format("a,b,c"); // ✅ string[] — to'g'ri
const result2 = format(42);      // ✅ string — to'g'ri
```

**Qoida:** Doim **eng aniq** overload ni birinchi, **eng keng** ni oxirida yozing. Keng overload birinchi tursa — barcha chaqiruvlarni "yutib yuboradi".

</details>

---

## Xulosa

- Return type — public API da yozish, internal da inference ga ishonish
- Optional (`?`) → `undefined`, default (`= value`) → standart qiymat (`undefined` berilsa ham)
- Overload — return type parametrga qarab farq qilganda. Union yetarli bo'lsa overload kerak emas
- `void` callback — istalgan return ruxsat (assignability uchun). Declaration da strict
- `this` parameter — compile-time da `this` yo'qolish xatoni topadi, JS ga tushmaydi
- Function assignability — kamroq param mos (JS callback pattern), param contravariant, return covariant
- Generic vs overload — mantiq bir xil → generic, mantiq farq → overload
- Type-safe EventEmitter — batafsil [interview/21 — Design Patterns](21-design-patterns.md) da

[Asosiy bo'limga qaytish →](../07-functions.md)
