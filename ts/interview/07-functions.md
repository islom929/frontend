# Interview: Functions TypeScript da

> Function type annotations, overloads, call/construct signatures, this parameter, void vs undefined, function assignability, rest/spread, callback types bo'yicha interview savollari.

---

## Nazariy savollar

### Savol 1: TS da funksiya return type ni yozish kerakmi yoki inference ga qo'yish kerakmi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Internal funksiyada inference yetarli, public API (export, library boundary)'da explicit return type tavsiya — refactor safety va aniq contract uchun.

### To'liq tushuntirish

TS funksiya body'dan return type ni inference qiladi. Bu kod tozaligi uchun yaxshi, lekin ba'zi holatlarda explicit yozish foydali:

- **Public API / library export** — type signature contract, body o'zgarsa ham public type tekshiriladi
- **Recursive function** — TS rekursiv chaqiruvni inference qila olmaydi (ba'zan circular)
- **`any` qaytaruvchi API** (`JSON.parse`, `fetch().then(r => r.json())`) — explicit type `any` propagation'ni cheklaydi
- **`isolatedDeclarations` mode** (TS 5.5+) — export funksiyalarda return type majburiy (DTS faylni body'siz generate qilish uchun)

### Kod misol

```typescript
// Internal — inference yetarli
function add(a: number, b: number) {
  return a + b; // inferred: number
}

// Public API — explicit
export function parseConfig(raw: string): Config {
  return JSON.parse(raw) as Config;
}

// Recursive — TS ba'zan inference qila olmaydi
function factorial(n: number): number {
  return n <= 1 ? 1 : n * factorial(n - 1);
}

interface Config {
  port: number;
  host: string;
}
```

### Edge Cases

- **Arrow function const** — `const fn = (x: number) => x * 2;` — type `(x: number) => number` inferred. Public API'da `const fn: (x: number) => number = ...` explicit.
- **Object method shorthand** — `{ greet(name: string) { return `Hi ${name}`; } }` — return type inferred (string).
- **Async function** — body `string` qaytarsa, signature `Promise<string>` inferred.
- **Generator function** — return type `Generator<Y, R, N>` murakkab, ko'pincha explicit yozish kerak.

### Follow-up savollar

1. **"`isolatedDeclarations` nima uchun kerak?"** — DTS faylni TypeScript checker'siz, faqat parser bilan generate qilish (build performance). Babel, swc kabi tool'lar bunda yordam beradi.
2. **"Return type explicit yozsam type narrowing yo'qoladimi?"** — Yo'q. Body ichida narrowing TS checker'da, return type signature contract. Internal flow analysis o'zgarmaydi.

</details>

---

### Savol 2: Optional parameter (`?`) va default parameter (`= value`) farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Optional `?` — berilmasa `undefined`, type `T | undefined`. Default `= value` — berilmasa standart qiymat va `undefined` explicit berilsa ham default ishlatiladi, type `T`.

### To'liq tushuntirish

| Holat | Optional `?` | Default `= value` |
|-------|--------------|-------------------|
| Berilmasa | `undefined` | default qiymat |
| `undefined` explicit berilsa | `undefined` qoladi | **default ishlatiladi** |
| Parameter type | `T \| undefined` | `T` |
| Tartib | Faqat oxirgi parameter'da | Istalgan joyda (lekin oldingi default'lar `undefined` orqali skip qilinadi) |

JS spec ECMA-262 spec'ida — default parameter `undefined` berilganda ishga tushadi (`IsDefaultParameterUsedHandled`).

### Kod misol

```typescript
function greetOptional(name: string, prefix?: string): string {
  return `${prefix ?? "Hello"}, ${name}!`;
}
greetOptional("Ali");             // "Hello, Ali!"
greetOptional("Ali", undefined);  // "Hello, Ali!" — undefined → ?? "Hello"

function greetDefault(name: string, prefix: string = "Hello"): string {
  return `${prefix}, ${name}!`;
}
greetDefault("Ali");              // "Hello, Ali!"
greetDefault("Ali", undefined);   // "Hello, Ali!" — default ishlatildi
greetDefault("Ali", "Hi");        // "Hi, Ali!"

// Type farqi:
function f1(x?: number) { return x; }       // x: number | undefined
function f2(x: number = 0) { return x; }    // x: number (default bor)
```

### Edge Cases

- **`null` berilganda** — default ishlamaydi (faqat `undefined` trigger qiladi). `greetDefault("Ali", null as any)` → `"null, Ali!"`.
- **Default qiymat sifatida boshqa parameter** — `function f(a: number, b: number = a * 2)` — keyingi parameter oldingisiga reference berishi mumkin (chap-to-o'ng).
- **Destructuring default** — `function f({ x = 0 } = {})` — ikki daraja default (object yo'qsa `{}`, x yo'qsa `0`).
- **`function.length`** — birinchi default yoki rest parameter'gacha bo'lgan parameter'larni sanaydi. `function f(a: number, b: number = 0, c: number)` — `f.length === 1`. Optional `?` ham `function.length`'ga sanalmaydi.

### Follow-up savollar

1. **"Public API'da qaysi biri afzal?"** — Default parameter — caller `undefined` yuborsa ham backward-compatible behavior. Optional — caller'ga "men `undefined` qabul qilaman" signal beradi.
2. **"Optional va default birga ishlatib bo'ladimi?"** — `function f(x?: number = 0)` — TS xato beradi: "Parameter cannot have question mark and initializer". `function f(x: number = 0)` allaqachon optional behavior beradi.

</details>

---

### Savol 3: Function overload nima? Resolution order qanday? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Function overload — bitta funksiyaga bir nechta call signature yozish, implementation alohida. TS yuqoridan pastga birinchi mos kelgan overload'ni tanlaydi — shuning uchun eng aniq overload birinchi yoziladi.

### To'liq tushuntirish

Overload kerak bo'lgan holat — **return type parametrga qarab o'zgaradi** (yoki parameter count/type kombinatsiyasi turli signature beradi).

Sintaksis:

1. **Overload signatures** (call signatures) — public API
2. **Implementation signature** — barchasini qamrab oluvchi, caller'ga ko'rinmaydi

Resolution order — TS overload list'ni **declaration tartibida** tekshiradi, **birinchi assignable** signature'ni tanlaydi. Implementation signature caller'ga ko'rinmaydi.

### Kod misol

```typescript
// Overload signatures (eng aniqdan eng kenggacha)
function format(input: string): string[];
function format(input: number): string;
function format(input: string | number): string | string[] {
  if (typeof input === "string") return input.split(",");
  return String(input);
}

const a = format("a,b,c"); // string[]
const b = format(42);       // string

// ❌ Noto'g'ri tartib — keng overload birinchi
function bad(input: string | number): string;
function bad(input: string): string[];
function bad(input: string | number): string | string[] {
  if (typeof input === "string") return input.split(",");
  return String(input);
}
const c = bad("a,b,c"); // string — string[] EMAS, chunki birinchi overload yutib yubordi
```

### Edge Cases

- **Implementation signature caller uchun ko'rinmaydi** — `format(true as any)` chaqirilmaydi (implementation `string | number` qabul qilsa ham).
- **Overload va default parameter** — implementation'da default berib bo'ladi, overload signatures'da optional sifatida ko'rsatiladi.
- **`this` parameter overload'da** — har bir overload'da alohida yozilishi kerak.
- **Generic overload** — har overload o'z type parameter'iga ega bo'lishi mumkin.
- **Type guard overload** — `function isString(x: unknown): x is string` — predicate overload da ishlaydi.

### Follow-up savollar

1. **"Overload o'rniga union return type yetarli emasmi?"** — Agar return type parametrga bog'liq bo'lmasa (har doim bir xil union qaytsa) — union yetarli. Bog'liq bo'lsa — overload narrowing beradi (`format("x")` → `string[]`, union'da har doim `string | string[]`).
2. **"Method overload class'da qanday?"** — Bir xil sintaksis, lekin method body bitta. Constructor overload ham mavjud (`constructor(x: string); constructor(x: number); constructor(x: string | number) {...}`).

<details>
<summary><strong>Deep Dive</strong></summary>

TS spec'ida overload resolution — `getCandidateSignatures` algoritmi. Har overload uchun:
1. Argument count check (optional/rest hisobga olinadi)
2. Each argument assignable to corresponding parameter?
3. Generic inference — type arguments infer qilinadi
4. Constraint check

Birinchi `true` qaytargan overload tanlanadi. Agar hech biri mos kelmasa — implementation signature tekshiriladi (lekin uning return type caller'ga ko'rinmaydi, faqat parameter compatibility uchun).

**Method dispatch'siz** — overload faqat TS type system'da, runtime'da bitta funksiya. JS'ga compile'da overload signatures o'chiriladi, faqat implementation qoladi.

</details>

</details>

---

### Savol 4: Call signature va construct signature farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Call signature — funksiya chaqiruv signature'i (`(x: T) => U`). Construct signature — `new` operator bilan chaqiruv signature'i (`new (x: T) => U`). Construct signature `new` orqali instance yaratadigan funksiyalar uchun (constructor, class).

### To'liq tushuntirish

**Call signature** — interface yoki type'da funksiya chaqiruvi:

```typescript
interface Logger {
  (message: string): void;            // call signature
  prefix: string;                      // property
  log(message: string): void;          // method
}
```

**Construct signature** — `new` bilan chaqiriladigan:

```typescript
interface UserConstructor {
  new (name: string, age: number): User;  // construct signature
}

class User {
  constructor(public name: string, public age: number) {}
}

const Ctor: UserConstructor = User;
const user = new Ctor("Ali", 25);
```

Class type'i o'zi construct signature'ga ega — `typeof User` = `new (name: string, age: number) => User`.

### Kod misol

```typescript
// Real-world: factory function call/construct ikkalasini qabul qiladi
interface ApiClientFactory {
  // Call — function-style yaratish
  (config: { baseUrl: string }): ApiClient;
  // Construct — class-style yaratish
  new (config: { baseUrl: string }): ApiClient;
}

class ApiClient {
  constructor(public config: { baseUrl: string }) {}
  get(path: string) {
    return fetch(`${this.config.baseUrl}${path}`);
  }
}

const createClient: ApiClientFactory = function(config) {
  return new ApiClient(config);
} as any;

const a = createClient({ baseUrl: "/api" });        // call
const b = new createClient({ baseUrl: "/api" });    // construct

// Abstract construct signature — abstract class
abstract class Shape {
  abstract area(): number;
}
type ShapeCtor = abstract new () => Shape; // abstract construct signature
// new ShapeCtor(); // ❌ Cannot instantiate abstract class
```

### Edge Cases

- **`abstract new`** — abstract class type'i `abstract new (...) => T`. Bunday type instance yaratib bo'lmaydi.
- **Call + construct birga** — bitta interface ikkala signature'ga ega bo'lishi mumkin (`Date` shunday: `Date()` string qaytaradi, `new Date()` Date instance).
- **Generic construct signature** — `interface ListCtor { new <T>(): T[] }` — instantiate'da type parameter beriladi.
- **`InstanceType<T>`** — construct signature'dan instance type'ni oladi. `InstanceType<typeof User>` = `User`.

### Follow-up savollar

1. **"`typeof Class` nima uchun construct signature?"** — Class declaration ikki narsa yaratadi: instance type (`User`) va constructor type (`typeof User`). Constructor type — static method'lar + construct signature.
2. **"Constructor parameter type'larini qanday olamiz?"** — `ConstructorParameters<typeof User>` — tuple qaytaradi (`[string, number]`).

</details>

---

### Savol 5: `this` parameter nima? Qanday ishlaydi va JS'ga compile'da nima bo'ladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`this` parameter — funksiya signature'ida birinchi parameter sifatida `this: T` yoziladi, `this` ning type'ini belgilaydi. JS'ga compile'da butunlay o'chiriladi (type-only).

### To'liq tushuntirish

JS'da `this` dynamic — funksiya qanday chaqirilishiga bog'liq (`obj.method()` da `this = obj`, `const f = obj.method; f()` da `this = undefined` strict mode'da). TS `this` parameter compile-time'da `this` context'ni tekshiradi.

Sintaksis: `function method(this: T, arg1: U, arg2: V)` — `this` haqiqiy parameter emas, faqat type annotation. Call'da `arg1` birinchi position'da.

### Kod misol

```typescript
class Timer {
  seconds = 0;
  tick(this: Timer): void {
    this.seconds++;
  }
}

const timer = new Timer();
timer.tick(); // ✅ this = timer

const detached = timer.tick;
// detached(); // ❌ 'this' context of type 'void' is not assignable to 'Timer'

setInterval(timer.tick, 1000);
// ❌ Argument of type '() => void' not assignable — this lost

setInterval(() => timer.tick(), 1000); // ✅ arrow binds this lexically
setInterval(timer.tick.bind(timer), 1000); // ✅ explicit bind

// Non-class function ham `this` parameter ishlatishi mumkin
interface User {
  name: string;
  greet(this: User): string;
}

function greet(this: User): string {
  return `Hello, ${this.name}`;
}

const user: User = { name: "Ali", greet };
user.greet();              // ✅ this = user
// greet.call({ name: 5 }); // ❌ name must be string
```

### Edge Cases

- **`this: void`** — funksiya `this` ishlatmasligini majbur qiladi. Callback uchun foydali: `function callback(this: void, x: number)` — `this` ishlatib bo'lmaydi.
- **Arrow function**'da `this` parameter — TS xato beradi (`An arrow function cannot have a 'this' parameter`). Arrow lexical this.
- **`ThisParameterType<T>` va `OmitThisParameter<T>`** — utility type'lar. `ThisParameterType<typeof greet>` = `User`. `OmitThisParameter` `this` parameter'ni olib tashlaydi.
- **Class method'da `this` parameter overload** — har overload'da alohida yozilishi shart.
- **`this` parameter va `bind`** — `greet.bind(user)` qaytaradi `OmitThisParameter<typeof greet>` type (this allaqachon bound).

### Follow-up savollar

1. **"Method reference muammosini boshqa qanday tuzatamiz?"** — Arrow method (`tick = () => { this.seconds++; }`) — class field, har instance'da o'z copy'si. Yoki constructor'da `this.tick = this.tick.bind(this)`.
2. **"`strictBindCallApply` flag'i nima?"** — `bind`/`call`/`apply` parameter type'larini strict tekshiradi. TS 3.2'da kiritilgan. `--strict` flag yoqilganda u ham yoqiladi. Bunsiz `this`/`call` arguments `any` edi.

</details>

---

### Savol 6: Function assignability — kamroq parametrli funksiya nima uchun mos keladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JS callback pattern (`forEach((value, index, array) => ...)`) — caller hamma argument yuboradi, callee faqat kerakli'larini ishlatadi. TS bu pattern'ni qo'llab-quvvatlash uchun kamroq parametrli funksiya'ni ko'p parametrli signature'ga mos deb hisoblaydi.

### To'liq tushuntirish

Function assignability uch qoidaga bo'ysunadi:

**1. Parameter count — kamroq mos (covariant in count):**

`(x: T) => U` ni `(x: T, y: V) => U` joyiga bersa bo'ladi. Sabab — JS'da qo'shimcha argument'lar ignore qilinadi.

**2. Parameter types — contravariant (`strictFunctionTypes: true`):**

`(x: Animal) => U` ni `(x: Dog) => U` joyiga bersa bo'ladi (Animal handler Dog'ni ham qabul qiladi, Dog ⊂ Animal). Lekin teskari emas.

**3. Return type — covariant:**

`() => Dog` ni `() => Animal` joyiga bersa bo'ladi (Dog ⊂ Animal). Lekin teskari emas.

### Kod misol

```typescript
type ThreeParamCallback = (a: number, b: string, c: boolean) => void;

// ✅ Kamroq parametr — mos
const fn1: ThreeParamCallback = (a) => console.log(a);
const fn2: ThreeParamCallback = () => {};

// ❌ Ko'proq parametr — mos emas
// const fn3: ThreeParamCallback = (a, b, c, d) => {};
// Parameter 'd' has no corresponding argument

// Contravariance — strictFunctionTypes
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }

type AnimalHandler = (a: Animal) => void;
type DogHandler = (d: Dog) => void;

const animalHandler: AnimalHandler = (a) => console.log(a.name);
const dogHandler: DogHandler = animalHandler;
// ✅ animalHandler `DogHandler` joyiga mos — contravariance:
// caller Dog yuboradi (Dog ⊂ Animal), animalHandler uni qabul qila oladi

// const wrongHandler: AnimalHandler = (d: Dog) => console.log(d.breed);
// ❌ Dog-specific handler Animal handler joyiga mos emas:
// caller har Animal yuborishi mumkin (Cat ham), Dog handler'ga unsafe

// Return covariance
type GetAnimal = () => Animal;
type GetDog = () => Dog;
const getDog: GetDog = () => new Dog();
const getAnimal: GetAnimal = getDog; // ✅ Dog Animal'ga assignable
```

### Edge Cases

- **Method bivariance** — class method'lar (`method(x: T): U` shorthand) `strictFunctionTypes`'da ham **bivariant** (tarixiy sabab, Array compat). Property arrow (`method: (x: T) => U`) — contravariant.
- **Rest parameter** — `(...args: T[]) => U` istalgan count'ni qabul qiladi.
- **Optional parameter** — `(x?: T) => U` ham `() => U` joyiga ham `(x: T) => U` joyiga mos.
- **Excess parameter implementation'da OK** — overload signature'larda kamroq parameter ko'rsatib, implementation'da ko'proq qabul qilish mumkin (caller faqat overload count'ini ko'radi).

### Follow-up savollar

1. **"`strictFunctionTypes: false`'da nima farq?"** — Parameter type bivariance — TS 2.6'gacha default edi. Hozir strict mode default. Eski kodda noto'g'ri Dog handler Animal joyiga o'tib ketardi.
2. **"Promise.then callback nega kam parametr qabul qiladi?"** — `then(onfulfilled?: (value: T) => U)` — single-parameter callback expected. JS spec'i shunday. TS shu signature'ni follow qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Variance theory:**

- **Covariant** — sub-type relation saqlanadi (`Dog ⊂ Animal` → `Array<Dog> ⊂ Array<Animal>` readonly bo'lsa).
- **Contravariant** — sub-type relation teskari (`Dog ⊂ Animal` → `(Animal) => U ⊂ (Dog) => U`).
- **Bivariant** — ikkala yo'nalishda mos (unsafe, lekin amaliy uchun foydali).
- **Invariant** — hech qanday yo'nalishda mos kelmaydi.

TS function type'larida default:
- Parameter — contravariant (strict mode)
- Return — covariant
- Method shorthand — bivariant (parameter)

**`strictFunctionTypes` implementation:**

TS checker `checkTypeRelatedTo` function'ida — function types comparing. Parameter type uchun source/target almashtiriladi (contravariance). Method shorthand `MethodSignature` flag bilan belgilanadi va bivariance saqlanadi.

</details>

</details>

---

### Savol 7: `void` va `undefined` farqi nima? Callback'da `void` nima uchun maxsus? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`void` — "return qiymati ahamiyatsiz" signal. `undefined` — aniq `undefined` qiymat. Callback'da `void` qaytarish — istalgan qiymat ruxsat (TS type system'da, runtime'da qiymat haqiqiy qaytariladi). Function declaration'da `void` strict.

### To'liq tushuntirish

`void` ikki context'da farq qiladi:

**1. Function declaration / return type annotation** — strict:

```typescript
function noReturn(): void {
  return 42; // ❌ Type 'number' is not assignable to type 'void'
}
```

**2. Callback / function type parameter** — assignability:

```typescript
type Callback = () => void;
const cb: Callback = () => 42; // ✅ — return qiymati ignored
```

Bu **JS callback pattern**'ni qo'llab-quvvatlash uchun: `Array.prototype.forEach`, `addEventListener` — callback'lar qaytargan qiymat ignore qilinadi. Agar `void` strict bo'lganida — `[1,2,3].forEach(n => arr.push(n))` ishlamasdi (`push` `number` qaytaradi).

### Kod misol

```typescript
type VoidCallback = () => void;

const fn1: VoidCallback = () => 42;          // ✅ runtime: 42
const fn2: VoidCallback = () => "hello";     // ✅ runtime: "hello"
const fn3: VoidCallback = () => undefined;   // ✅ runtime: undefined

const result = fn1();
// TS type: void — result ishlatib bo'lmaydi
// console.log(result.toFixed(2)); // ❌ Property 'toFixed' does not exist on type 'void'

// Lekin runtime'da qiymat bor:
console.log(fn1()); // 42

// forEach pattern
const numbers: number[] = [];
[1, 2, 3].forEach((n) => numbers.push(n));
// push returns number, forEach expects void callback — OK

// Function declaration strict
function strictReturn(): void {
  // return 42; // ❌
  return; // ✅
  // implicit return ham OK
}

// undefined explicit
function returnsUndefined(): undefined {
  return undefined; // ✅
  // return; // ✅ ham OK
}
```

### Edge Cases

- **`undefined` type strict** — `function f(): undefined { return 42; }` xato. `undefined` literal type, faqat `undefined` qaytarish kerak.
- **`void` operator (JS)** — TS `void` type'i bilan farq. `void 0` — `undefined` qiymat ekspresion JS'da.
- **`void` in generic position** — `Promise<void>` — `then` callback `value` parameter `void` (ishlatib bo'lmaydi).
- **Empty body callback `() => {}`** — bo'sh block (object literal emas), `undefined` qaytaradi. Object qaytarish uchun `() => ({})` qavs.
- **`void 0` vs `undefined`** — JS minification trick. ES5+ da global `undefined` non-writable, lekin local scope'da `let undefined = 1` mumkin edi (sloppy mode ES3). `void 0` har doim `undefined` qiymatga aylanadi (semantic guarantee).

### Follow-up savollar

1. **"Nima uchun `() => void` ga `() => Promise<void>` mos kelmaydi (default)?"** — Async funksiya `Promise<undefined>` qaytaradi. Caller `void` deb ignore qilsa, unhandled promise rejection bo'lishi mumkin. TS bu xavfni ko'rsatadi.
2. **"`noImplicitReturns` qachon void'ga ta'sir qiladi?"** — Funksiya'da ba'zi branch'lar return qilmasa xato beradi. `void` return type bilan bu flag triggered bo'lmaydi (har branch implicit `undefined` qaytaradi).

</details>

---

### Savol 8: Rest parameter va spread argument — TS qanday tekshiradi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Rest parameter — `...args: T[]` yoki `...args: [A, B, C]` (tuple) sintaksis. Spread argument — `fn(...arr)` chaqiruvda. TS tuple rest bilan exact count va type'ni tekshiradi, array rest bilan faqat type.

### To'liq tushuntirish

**Rest parameter ikki shakl:**

1. **Array rest** — `...args: number[]` — istalgan count number qabul qiladi.
2. **Tuple rest** — `...args: [string, number, boolean]` — exact tuple shape.

**Variadic tuple** (TS 4.0+) — rest tuple ichida boshqa rest'larni birlashtirish.

### Kod misol

```typescript
// Array rest — istalgan count
function sum(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3);          // ✅
sum(...[1, 2, 3]);     // ✅ spread argument

// Tuple rest — exact shape
function logEvent(...args: [event: string, timestamp: number, source: string]): void {
  console.log(args);
}
logEvent("click", Date.now(), "button");
// logEvent("click", "now", "button"); // ❌ timestamp must be number

// Variadic tuple — birlashtirish
function call<T extends unknown[], R>(
  fn: (...args: T) => R,
  ...args: T
): R {
  return fn(...args);
}

function greet(name: string, age: number): string {
  return `${name} is ${age}`;
}

const result = call(greet, "Ali", 25); // ✅ args type'i [string, number]'dan infer

// Spread heterogeneous
function curry<A, B, R>(
  fn: (a: A, b: B) => R,
  ...partial: [A]
): (b: B) => R {
  return (b) => fn(...partial, b);
}

const addPartial = curry((a: number, b: number) => a + b, 5);
addPartial(3); // 8
```

### Edge Cases

- **Rest parameter faqat oxirgi** — `function f(...args: number[], last: string)` xato. JS spec qoidasi.
- **Spread iterable'larni** — `Set`, `Map`, generator ham spread bo'ladi. TS `Iterable<T>` interface'iga qaytadi.
- **Tuple rest middle** — `[string, ...number[], boolean]` — TS 4.0+ qo'llab-quvvatlanadi (variadic tuple types). Labeled tuple (`[event: string, ...nums: number[]]`) — TS 4.0+.
- **Spread function call'da type narrowing** — `fn(...tuple)` da TS tuple shape'ini funksiya parametr'lariga moslaydi.
- **`Parameters<T>` rest bilan** — `Parameters<typeof fn>` tuple sifatida qaytaradi, rest qismi `[...string[]]` shaklida.

### Follow-up savollar

1. **"Spread argument count compile-time'da aniqlanmasa nima?"** — Tuple rest sifatida yozilgan bo'lsa TS shape'ni biladi. Array rest spread'da count noma'lum, TS faqat element type'ni tekshiradi.
2. **"`apply` va spread farqi?"** — JS'da `fn.apply(this, arr)` va `fn(...arr)` ekvivalent. TS ikkalasi uchun ham type checking qiladi.

</details>

---

### Savol 9: Generic function va overload — qachon qaysi birini ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic — bir xil mantiq turli type'lar bilan (`first<T>(arr: T[]): T | undefined`). Overload — har xil mantiq + return type parametr'ga qarab o'zgarsa (`serialize(string) → string` vs `serialize(Date) → string`). Avval generic'ni sinab ko'ring, agar logic farq qilsa overload'ga o'ting.

### To'liq tushuntirish

| Holat | Generic | Overload |
|-------|---------|----------|
| Mantiq bir xil, type farq | Mos | Ortiqcha |
| Mantiq turli, return type parametr'ga bog'liq | Conditional return bilan mumkin | Tabiiy |
| Yangi type qo'shish | Hech narsa o'zgartirmaydi | Yangi overload kerak |
| O'qish oson | Mos | Ko'p signature murakkab |
| Inference quality | Yaxshi | Yaxshi |

### Kod misol

```typescript
// Generic — bir xil mantiq
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first([1, 2, 3]);          // number | undefined
first(["a", "b"]);         // string | undefined

// Overload — har xil mantiq
function serialize(value: string): string;
function serialize(value: number): string;
function serialize(value: Date): string;
function serialize(value: string | number | Date): string {
  if (typeof value === "string") return JSON.stringify(value);
  if (typeof value === "number") return value.toFixed(2);
  return value.toISOString();
}

// Conditional return type — generic bilan
function parseInput<T extends "json" | "text">(
  raw: string,
  mode: T
): T extends "json" ? object : string {
  if (mode === "json") return JSON.parse(raw);
  return raw as T extends "json" ? object : string;
}

const obj = parseInput('{"x":1}', "json");  // object
const str = parseInput("hello", "text");     // string
```

### Edge Cases

- **Generic + overload kombinatsiyasi** — har overload o'z generic'ga ega bo'lishi mumkin (`function f<T>(x: T): T[]; function f<T, U>(x: T, y: U): [T, U]`).
- **Overload limit** — TS amalda overload count'ga chegara qo'ymaydi, lekin checker performance'i pasayadi.
- **Generic conditional vs overload trade-off** — conditional implementation ichida `as` assertion talab qiladi (deferred conditional resolve bo'lmaydi). Overload — clean body.
- **Generic'da inference fail** — generic argument berilmasa va inference fail bo'lsa, T constraint'ga teng (default `unknown` TS 3.5+). TS 3.4 va undan oldin — default `{}` edi. Strict mode'da bu xato beradi (`noImplicitAny`'ga o'xshash).

### Follow-up savollar

1. **"Overload o'rniga `Parameters` utility ishlatib bo'ladimi?"** — Ha, agar caller side type'larini olishni xohlasangiz: `function wrap<T extends (...args: any[]) => any>(fn: T): (...args: Parameters<T>) => ReturnType<T>`.
2. **"Generic constraint vs overload — qaysi performance jihatdan tezroq?"** — Generic — bitta signature, TS instantiate qiladi har call'da. Overload — har biri alohida tekshiriladi (resolution loop). Katta union'larda generic afzal.

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 10: Output — Overload resolution [Middle]

**Savol:** Output va TS type'larini ayting:

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
["a", "b", "c"]
x,y
object
string
```

### To'liq tushuntirish

- `process("a,b,c")` — overload 1 mos (`string` → `string[]`), `"a,b,c".split(",")` = `["a", "b", "c"]`
- `process(["x", "y"])` — overload 2 mos (`string[]` → `string`), `["x", "y"].join(",")` = `"x,y"`
- TS type'lari: `a: string[]`, `b: string`
- Runtime `typeof`: `typeof ["a","b","c"]` = `"object"` (array — JS'da object), `typeof "x,y"` = `"string"`

Overload resolution yuqoridan pastga — birinchi mos kelgan signature.

### Edge Cases

- `typeof []` har doim `"object"` JS'da. Array'ni aniqlash uchun `Array.isArray()` kerak.
- Agar `process(["a"] as string[])` o'rniga `process(["a"])` yozilsa ham overload 2 tanlanadi.

### Follow-up savollar

1. **"Agar overload tartibi teskari bo'lsa (string[] birinchi) — nima o'zgaradi?"** — `process("a,b,c")` da TS string array'ga assignable emas, shuning uchun overload 2 birinchi tekshiriladi va o'tib ketadi, overload 1 mos keladi. Natija o'zgarmaydi.

</details>

---

### Savol 11: Output — void callback gotcha [Middle]

**Savol:** Output va sababini tushuntiring:

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
[false, false, true, true]
```

### To'liq tushuntirish

`Callback = () => void` — void callback istalgan qiymat qaytarishi mumkin (TS type system'da). Runtime'da haqiqiy qiymat qaytariladi:

- `() => 42` — runtime'da `42`, `42 === undefined` → `false`
- `() => "hello"` — `"hello"`, → `false`
- `() => undefined` — `undefined`, → `true`
- `() => {}` — **bo'sh block body** (object literal emas!). Implicit `undefined` qaytaradi → `true`

**Gotcha:** `() => {}` — bu **bo'sh statement block**. Bo'sh object qaytarish uchun qavs ichida: `() => ({})`.

### Edge Cases

- `cb()` natijasi TS'da `void` type — `result === undefined` ga compare qilish to'g'ri ishlaydi (runtime'da JS'da `void` mavjud emas).
- `runCallbacks` return type `boolean[]` — `void` callback'lardan haqiqiy qiymat olish strict TS'da type assertion talab qilishi mumkin.

### Follow-up savollar

1. **"Nima uchun TS `cb()` natijasini ishlatishga ruxsat bermaydi?"** — Strict mode'da `void` qiymatni ishlatish (masalan `.toString()`) xato beradi. Comparison (`=== undefined`) maxsus holat — TS ruxsat beradi.
2. **"`() => ({})` va `() => {}` farqi runtime'da?"** — Birinchi `{}` (bo'sh object) qaytaradi, ikkinchi `undefined`. Arrow body parser ambiguity — `{}` block sifatida interpret qilinadi.

</details>

---

### Savol 12: Bug — Overload return type mismatch [Middle+]

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

`user2.age` — compile-time xato. Ikkinchi overload return type `{ name: string }` — `age` property yo'q.

### To'liq tushuntirish

```typescript
console.log(user1.age); // ✅ 25 — overload 1: { name: string; age: number }
console.log(user2.age); // ❌ Property 'age' does not exist on type '{ name: string }'
```

Overload kuchi shunda — return type parametr count'ga qarab **farqlanadi**. Caller `createUser("Vali")` chaqirsa, TS biladi `age` yo'q.

Agar overload bo'lmaganida:

```typescript
function createUserSimple(name: string, age?: number) {
  return age !== undefined ? { name, age } : { name };
}
// Return: { name: string; age: number } | { name: string }
// user.age — narrowing kerak (yomonroq DX)
```

### Edge Cases

- **Tuzatish (agar `age` ga doim kirish kerak bo'lsa)**:

```typescript
function createUser(name: string, age: number): { name: string; age: number };
function createUser(name: string): { name: string; age?: number };
function createUser(name: string, age?: number) {
  return age !== undefined ? { name, age } : { name };
}
const user2 = createUser("Vali");
console.log(user2.age); // ✅ undefined (optional)
```

### Follow-up savollar

1. **"Discriminated union return type yaxshiroqmi?"** — `{ tag: "with-age", name, age } | { tag: "no-age", name }` — narrowing aniq, lekin caller'ga `tag` tekshirishni majbur qiladi. Overload — DX yaxshiroq.

</details>

---

### Savol 13: Coding — Overload to conditional return [Middle+]

**Savol:** Overloaded funksiyani bitta generic funksiya bilan qayta yozing, conditional return type ishlatib:

```typescript
function parse(input: string): number;
function parse(input: number): string;
function parse(input: string | number): string | number {
  if (typeof input === "string") return parseInt(input, 10);
  return String(input);
}

const a = parse("42");  // number
const b = parse(42);    // string

// Generic conditional versiyasini yozing:
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
function parse<T extends string | number>(
  input: T
): T extends string ? number : string {
  type Result = T extends string ? number : string;
  if (typeof input === "string") {
    return parseInt(input, 10) as Result;
  }
  return String(input) as Result;
}
```

### To'liq tushuntirish

`T extends string ? number : string` — distributive conditional. Caller side'da type narrowing aniq:

```typescript
const a = parse("42");  // number ✅
const b = parse(42);    // string ✅
const c = parse(Math.random() > 0.5 ? "x" : 42);
// Type: number | string (distributive union)
```

Trade-off jadvali:

| | Overload | Conditional return |
|---|---|---|
| O'qish oson | Mos | Murakkab |
| `as` assertion body'da | Kerak emas | Kerak (deferred conditional) |
| Generic kombinatsiya | Mos emas | Mos |
| Inference quality | Aniq | Aniq |

### Edge Cases

- **Deferred conditional** — TS generic body'da `T` aniq emas, conditional resolve bo'lmaydi. `as` assertion kerak.
- **Distributive union** — `parse(value: string | number)` chaqirsa, return type `number | string` (har union member alohida resolve).
- **`NoInfer` bilan** (TS 5.4) — `function parse<T extends string | number>(input: T): NoInfer<T> extends string ? number : string` — return type'da inference yo'q.

### Follow-up savollar

1. **"`NoInfer` qachon kerak?"** — Generic type parameter ikki joyda ishlatilganda, faqat bittasidan inference qilinishi kerak bo'lsa. Misol: `function as<T>(value: T, fallback: NoInfer<T>)` — fallback'dan T infer bo'lmaydi.
2. **"Conditional return implementation'ga ta'sir qiladimi?"** — Faqat caller side'da farq. Body ichida har doim `as` kerak (TS generic body'da conditional ni resolve qilmaydi).

<details>
<summary><strong>Deep Dive</strong></summary>

**Deferred conditional types** — TS spec terminology. Conditional type `T extends U ? X : Y` agar T yoki U generic parameter bo'lsa, **deferred** holatga o'tadi. Resolve faqat T va U aniq bo'lganda bo'ladi (caller side'da).

Body ichida `typeof value === "string"` JS-level narrowing — TS bunga `value` parameter type'ini narrow qiladi, lekin generic T parameter'ni narrow qila olmaydi (T abstract).

**Implementation rationale:** TS conditional types'ni structural emas, **lazy** evaluate qiladi. Bu performance va correctness uchun. Strict resolve agar T multiple narrowing bo'ladigan bo'lsa ko'p hollarda noto'g'ri javob berishi mumkin.

**Workaround patternlar:**
- `as` assertion (eng oddiy)
- Helper function (private overload, public generic)
- Single conditional output (avoid multi-branch implementation)

</details>

</details>

---

### Savol 14: Coding — Type-safe EventEmitter [Senior]

**Savol:** `EventEmitter<Events>` yozing — `on`, `off`, `emit` method'lari Events map'iga qarab type-safe bo'lsin:

```typescript
type AppEvents = {
  login: { userId: string; timestamp: number };
  logout: { userId: string };
  error: Error;
};

// const emitter = new EventEmitter<AppEvents>();
// emitter.on("login", (data) => data.userId);     // data: { userId; timestamp }
// emitter.emit("login", { userId: "1", timestamp: 0 });
// emitter.on("unknown", () => {});                  // ❌ unknown event
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type EventListener<T> = (payload: T) => void;

class EventEmitter<Events extends Record<string, unknown>> {
  private listeners: {
    [K in keyof Events]?: EventListener<Events[K]>[];
  } = {};

  on<K extends keyof Events>(event: K, listener: EventListener<Events[K]>): void {
    (this.listeners[event] ??= []).push(listener);
  }

  off<K extends keyof Events>(event: K, listener: EventListener<Events[K]>): void {
    const arr = this.listeners[event];
    if (!arr) return;
    const idx = arr.indexOf(listener);
    if (idx >= 0) arr.splice(idx, 1);
  }

  emit<K extends keyof Events>(event: K, payload: Events[K]): void {
    this.listeners[event]?.forEach((listener) => listener(payload));
  }
}
```

### To'liq tushuntirish

- `Events extends Record<string, unknown>` — event map constraint
- `K extends keyof Events` — har method event name'ni cheklaydi
- `Events[K]` — index access, ushbu event'ning payload type'i
- Listener storage — mapped type `{ [K in keyof Events]?: EventListener<Events[K]>[] }`

```typescript
type AppEvents = {
  login: { userId: string; timestamp: number };
  logout: { userId: string };
  error: Error;
};

const emitter = new EventEmitter<AppEvents>();

emitter.on("login", (data) => {
  console.log(data.userId, data.timestamp); // ✅ aniq type
});

emitter.emit("login", { userId: "1", timestamp: Date.now() });
// emitter.emit("login", { userId: "1" });  // ❌ timestamp missing
// emitter.on("unknown", () => {});           // ❌ "unknown" keyof AppEvents'da yo'q
```

### Edge Cases

- **Multiple listener** — har event'ga ko'p listener qo'shish mumkin (`listeners[event]` array).
- **Once handler** — qo'shimcha method: `once(event, listener)` — bir marta ishlaydi, keyin auto-removed.
- **Listener inference** — TS `(data) => ...` da `data` aniq type olishi uchun event name string literal bo'lishi kerak (constant yoki `as const`).
- **Wildcard event (`*`)** — Events map'ga `*: unknown` qo'shish kerak, yoki alohida `onAny` method.

### Follow-up savollar

1. **"`emit` payload optional bo'lishi mumkinmi?"** — `Events[K] extends void ? [] : [Events[K]]` rest tuple bilan: `emit<K>(event: K, ...args: Events[K] extends void ? [] : [Events[K]])`.
2. **"`off` listener'siz qanday — barchasini olib tashlash?"** — Overload qo'shish: `off(event)` va `off(event, listener)`. Birinchi listener array'ni tozalaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Mapped type'da optional**: `{ [K in keyof Events]?: ... }` — har property optional. Bu `listeners[event]` access'da `T[K] | undefined` qaytaradi, `??=` operator bilan lazy init.

**Variance considerations:**
- Listener parameter — contravariant. Lekin shu type'da invariant ishlatishimiz uchun listener'lar bir xil event'ga aynan bir xil payload type qabul qiladi.
- Real-world EventEmitter (Node.js) — bivariant API, type safety yo'q. Bizning version — strict.

**Memory leak risks:** Listener'lar `EventEmitter` instance lifecycle'iga bog'liq. Cleanup uchun `off` doim kerak. Modern API'lar `AbortController` + `signal` ishlatadi (DOM `addEventListener` kabi).

**Generic constraint qoidalari:**
- `Events extends Record<string, unknown>` — har event payload bor (`void` ham unknown).
- `K extends keyof Events` — string literal narrowing
- `EventListener<Events[K]>` — index access generic'ga distribute bo'ladi

</details>

</details>

---

### Savol 15: Bug — `this` parameter detached method [Senior]

**Savol:** Bu kod compile xato beradi. Bu xato JS'da qachon paydo bo'lardi va TS uni qanday topadi?

```typescript
class Counter {
  count = 0;
  increment(this: Counter): void {
    this.count++;
  }
}

const counter = new Counter();
const increment = counter.increment;
increment();

setInterval(counter.increment, 1000);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ikkala chaqiruvda `this` Counter instance'ga bog'lanmaydi. `this: Counter` parameter — TS uchun marker, JS'da detach'da `this = undefined` (strict) yoki `globalThis` (non-strict). TS compile-time xato beradi.

### To'liq tushuntirish

```typescript
const increment = counter.increment;
// increment(); // ❌ 'this' context of type 'void' is not assignable to method's 'this' of type 'Counter'

setInterval(counter.increment, 1000);
// ❌ Same error — callback'da this lost
```

JS runtime'da bu xato faqat method body ichida `this.count++` chaqirilganda paydo bo'lardi (`Cannot read property 'count' of undefined`). TS `this` parameter bilan compile-time'da topadi.

**Tuzatishlar:**

```typescript
// 1. Arrow function — lexical this
setInterval(() => counter.increment(), 1000);

// 2. .bind()
setInterval(counter.increment.bind(counter), 1000);

// 3. Arrow class field — har instance'da bound copy
class CounterArrow {
  count = 0;
  increment = (): void => {
    this.count++;
  };
}
```

### Edge Cases

- **Arrow class field memory cost** — har instance'da method copy. Prototype method — sinf bo'yicha bir marta saqlanadi.
- **`.bind()` performance** — har chaqiruvda yangi bound function. Kashlash kerak bo'lsa, constructor'da: `this.increment = this.increment.bind(this)`.
- **TypeScript Decorator `@autobind`** — `@autobind` decorator method'larni avtomatik bind qiladi (3rd-party).
- **`globalThis` non-strict** — `script` tag'larda (strict mode emas) detach method'da `this = window`. `count` `window`'da yo'q → `undefined++` → `NaN`.

### Follow-up savollar

1. **"Prototype method vs arrow class field — qaysi biri afzal?"** — Prototype — memory efficient, lekin `this` bog'lanmagan. Arrow field — `this` bound, lekin har instance'da copy. UI handler'lar (React class component) odatda arrow afzal.
2. **"`this: void` parameter qachon ishlatiladi?"** — Callback'lar `this` ishlatmasligini majbur qilish. Misol: `array.sort((a, b) => a - b)` — comparator `this` ishlatmaslik kerak.

<details>
<summary><strong>Deep Dive</strong></summary>

**`this` parameter — JS'ga compile:**

TS'da `function method(this: T, x: U)` — `this` faqat type annotation. JS'ga compile'da `this` parameter olib tashlanadi, `function method(x)` qoladi. Argument tartibi `arguments[0]` = `x` (this position 0 hisoblanmaydi).

**Compile-time check algoritmi:**

TS checker `checkMethodReference` — method'ga reference olinganda `this: T` declared bo'lsa, target type tekshiriladi. Detached call'da `this` `void` deb hisoblanadi (`this: void` parametriga assignable, lekin `this: Counter` parametriga assignable emas).

**Arrow class field — emit farqi:**

```typescript
// Prototype method (transpile)
class Counter { increment(): void { /* ... */ } }
// → Counter.prototype.increment = function() { ... };

// Arrow class field (transpile)
class CounterArrow { increment = (): void => { /* ... */ }; }
// → constructor() { this.increment = () => { ... }; }
```

Arrow field — har constructor call'da yangi function instance yaratiladi (`new CounterArrow()` × 1000 = 1000 ta function object). Prototype — bitta shared function.

**`useDefineForClassFields` flag (TS 3.7+):**

ES2022 class field semantics'ga mos kelish uchun. Yoqilganda — class field'lar `Object.defineProperty` bilan declare qilinadi (constructor body'da assign emas). Inherited method'larni override qilishda kichik farq bor.

**Decorator-based binding (TS 5.0+ Stage 3 decorators):**

```typescript
function autobind(value: Function, context: ClassMethodDecoratorContext) {
  context.addInitializer(function() {
    (this as any)[context.name] = value.bind(this);
  });
}

class Counter {
  count = 0;
  @autobind increment() { this.count++; }
}
```

Decorator runtime'da har instance'da `bind` qiladi (memory cost arrow field bilan teng), lekin syntax prototype method'ga o'xshash.

</details>

</details>

---

### Savol 16: Output — Function assignability variance [Senior]

**Savol:** Har qator uchun compile xato yoki OK? `strictFunctionTypes: true`.

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }
class Cat extends Animal { color = ""; }

type AnimalHandler = (a: Animal) => Animal;
type DogHandler = (d: Dog) => Dog;

const animalHandler: AnimalHandler = (a) => a;
const dogHandler: DogHandler = (d) => d;

// 1
const x: AnimalHandler = dogHandler;
// 2
const y: DogHandler = animalHandler;
// 3
const z: () => Animal = () => new Dog();
// 4
const w: () => Dog = () => new Animal();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
1. ❌ Error — parameter contravariance fail
2. ❌ Error — return covariance fail
3. ✅ OK — return covariance pass
4. ❌ Error — return covariance fail
```

### To'liq tushuntirish

- **1. `AnimalHandler = dogHandler` — ❌**: Parameter contravariance fail. `AnimalHandler` har Animal sub-type'ni qabul qilishi kerak (shu jumladan Cat'ni), lekin `DogHandler` faqat Dog qabul qiladi. `strictFunctionTypes: true` da bu xato.

- **2. `DogHandler = animalHandler` — ❌**: Parameter contravariance OK (Animal handler Dog qabul qiladi, Dog ⊂ Animal). Lekin return type Animal `Dog`'ga assignable emas (`breed` property yo'q). Return covariance fail.

- **3. `() => Animal = () => new Dog()` — ✅**: Return covariance. Dog ⊂ Animal, shuning uchun `() => Dog` ni `() => Animal` joyiga berib bo'ladi.

- **4. `() => Dog = () => new Animal()` — ❌**: Return covariance fail. Animal Dog'ga assignable emas.

### Edge Cases

- **Method bivariance**: agar `AnimalHandler` interface method shorthand bo'lsa (`interface { handle(a: Animal): Animal }`), variant 1 ishlardi (bivariant parameter). Type alias va arrow — strict contravariance.
- **Top type `unknown` parameter** — `(x: unknown) => Animal` — har handler'ga mos (unknown — universal supertype).
- **Bottom type `never` return** — `() => never` har return type'ga assignable (never bottom).

### Follow-up savollar

1. **"`strictFunctionTypes: false` da 1 ishlaydimi?"** — Ha, parameter bivariant bo'ladi. Lekin runtime'da Cat berilsa `breed` access crashes.
2. **"Method bivariance nima uchun saqlandi?"** — Array.prototype.forEach va boshqa built-in API'lar method shorthand'lar bilan declarated. Strict bo'lsa eski kod buzilardi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Subtype relation va variance matrix:**

Function type `(P) => R` ikki pozitsiyaga ega. Sub-typing relation `A ⊂ B` (A is assignable to B):

| Pozitsiya | Variance | Sub-type qoidasi |
|-----------|----------|------------------|
| Parameter `P` | Contravariant | `A ⊂ B` → `(B) => R ⊂ (A) => R` |
| Return `R` | Covariant | `A ⊂ B` → `(P) => A ⊂ (P) => B` |
| Method param (shorthand) | Bivariant | Ikki yo'nalishda mos (unsafe legacy) |

**`strictFunctionTypes` implementation:**

TS checker `compareSignaturesRelated` funksiyasida har parameter `compareTypes(target, source)` (contravariant — almashtirilgan) bilan tekshiriladi. Return type `compareTypes(source.return, target.return)` (covariant). Method shorthand'da `SignatureFlags.IsMethod` flag — bu flag bilan parameter compare bivariant (`compareTypes` ikki yo'nalishda).

**Method bivariance — tarixiy sabab:**

TS 2.6'gacha barcha function parameter'lar bivariant edi. Eski kod migration uchun (`Array.prototype.forEach((value: number) => ...)` — `number` `unknown`'dan derive bo'lsa contravariance buzgan bo'lardi) — TS team method shorthand uchun bivariance saqladi. Arrow property syntax (`method: (x: T) => U`) — strict contravariance.

**Bottom type `never` ning roli:**

`() => never` har return type'ga assignable (Liskov substitution — sub-type sub-type return berishi mumkin, `never` har type'ning sub-type). `throw` yoki infinite loop qaytaruvchi funksiya har joyga mos.

**Top type `unknown` parameter:**

`(x: unknown) => U` — parameter contravariance bo'yicha har handler joyiga mos (caller har type yuborishi mumkin, callee uni narrow qilib ishlaydi). `any` ham mos, lekin type safety yo'qotadi.

</details>

</details>

---

## Xulosa

- Return type — public API'da explicit, internal'da inference
- Optional `?` → `undefined`, default `= value` → standart qiymat (explicit `undefined` ham trigger qiladi default'ni)
- Overload — return type parametr'ga bog'liq bo'lganda, eng aniq overload birinchi
- Call signature funksiya chaqiruv, construct signature `new` bilan chaqiruv
- `this` parameter — compile-time `this` context tekshiruvi, JS'ga tushmaydi
- `void` callback'da assignability uchun maxsus (har return ruxsat), function declaration'da strict
- Rest parameter tuple bilan exact shape, array bilan istalgan count
- Function assignability — parameter contravariant, return covariant, count kamroq mos
- Generic mantiq bir xil, overload mantiq farq + return parametr'ga bog'liq

---
