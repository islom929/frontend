# Bo'lim 25: Type Compatibility va Variance

> Type compatibility — TypeScript type system'ning asosiy mexanizmi: bir type'ning ikkinchisiga assignable bo'lishini aniqlash. Structural typing, covariance, contravariance, bivariance, invariance — generic type'lar bir-biriga qanday mos kelishini belgilaydi. `in`/`out` variance modifier'lar (TS 4.7+) bilan explicit control. Variance — interview va complex type design'da kritik mavzu.

---

## Mundarija

- [Structural Typing](#structural-typing)
- [Type Compatibility Rules](#type-compatibility-rules)
- [Covariance — Bir Yo'nalishda Moslik](#covariance--bir-yonalishda-moslik)
- [Contravariance — Teskari Yo'nalishda](#contravariance--teskari-yonalishda)
- [Bivariance va Invariance](#bivariance-va-invariance)
- [`in`/`out` Variance Modifiers (TS 4.7+)](#inout-variance-modifiers-ts-47)
- [Function Compatibility](#function-compatibility)
- [Class va Enum Compatibility](#class-va-enum-compatibility)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Structural Typing

### Nazariya

TypeScript **structural typing** (duck typing) ishlatadi — type'ning **nomi** emas, **shakli** (shape — property'lar to'plami va type'lari) muhim. Agar A type'ning barcha required property'lari B type'da mos type bilan bor bo'lsa — A ni B ga **assignable** deb hisoblanadi.

Bu **nominal typing** (Java, C#, Rust) dan farqli — u yerda ikki class bir xil shape'ga ega bo'lsa ham, agar nomi yoki inheritance'da bog'liqlik bo'lmasa — assignable emas. Structural — kompozitsiya va refactoring uchun yengil, duck-typed JavaScript bilan moslashuvchan.

```typescript
interface Point { x: number; y: number; }
interface Coordinate { x: number; y: number; }

const point: Point = { x: 1, y: 2 };
const coordinate: Coordinate = point; // ✅ — shakli bir xil, nomi farq qilmaydi
```

**Excess property checking** — fresh object literal'da qo'shimcha property bor bo'lsa, compiler error beradi. Bu typo'larni ushlash uchun mexanizm:

```typescript
interface User { name: string; age: number; }

const user: User = { name: "Ali", age: 25, email: "a@b.com" };
// ❌ — Object literal may only specify known properties, and 'email' does not exist
```

**Nima uchun faqat literal'da:** literal — programmer'ning intention'ini bevosita ifodalaydi (typo bo'lishi mumkin). Variable orqali assign — value boshqa kontekstda ishlatilgan bo'lishi mumkin, qo'shimcha property'lar normal:

```typescript
const rawUser = { name: "Ali", age: 25, email: "a@b.com" };
const user2: User = rawUser; // ✅ — variable orqali bypass (structural mos)
```

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript checker'da structural compatibility `isTypeAssignableTo(source, target)` funksiyasi orqali aniqlanadi. Algoritm:

1. **Identity check** — agar source va target bir xil type bo'lsa, true.
2. **Property-by-property** — target'dagi har required property uchun:
   - Source'da shu nom bilan property bormi?
   - Source property type'i target property type'iga assignable mi (recursive call)?
3. **Optional property** — agar target'da optional, source'da yo'q bo'lsa, OK.
4. **Excess property check** — agar source fresh literal bo'lsa, source'dagi har property target'da mavjudligini tekshirish.

**Performance:** checker assignability natijalarini type juftligi bo'yicha cache qiladi (assignable relation). Bir xil `(source, target)` juftligi ikkinchi marta to'liq tekshirilmaydi — bu katta loyihalarda ko'p type comparison'ni tezlashtiradi.

**Variance — kontekstga qarab:** property type comparison covariant (readonly position), contravariant (function parameter), yoki invariant (mutable position) bo'lishi mumkin. Bu boshqa section'da batafsil.

</details>

---

## Type Compatibility Rules

### Nazariya

| Qoida | Misol |
|-------|-------|
| **Object** — kerakli property lar bor bo'lsa mos | `{ x, y, z }` → `{ x, y }` ✅ |
| **Function params** — kamroq parametr mos | `(a) => void` → `(a, b) => void` ✅ |
| **Return type** — subtype mos (covariant) | `() => Dog` → `() => Animal` ✅ |
| **Optional** — optional required ga mos | `{ x: string }` → `{ x?: string }` ✅ |
| **`void` return** — har qanday return mos | `() => number` → `() => void` ✅ |

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Object compatibility ===
interface Animal { name: string; }
interface Dog extends Animal { breed: string; }

let animal: Animal = { name: "Rex" };
let dog: Dog = { name: "Rex", breed: "Labrador" };

animal = dog;  // ✅ — Dog da name bor (ko'proq property OK)
// dog = animal; // ❌ — Animal da breed yo'q

// === Function parameter — kamroq param mos ===
type Callback = (a: string, b: number) => void;
const fn: Callback = (a) => console.log(a); // ✅ — 1 param (2 dan kam)
// Nima uchun? [1, 2].forEach((v, i) => ...) — 3-arg callback ga 1-arg fn berish xavfsiz

// === Return type — subtype mos (covariant) ===
type GetAnimal = () => Animal;
const getDog: GetAnimal = () => ({ name: "Rex", breed: "Lab" }); // ✅ — Dog <: Animal

// === void return — hamma mos ===
type VoidFn = () => void;
const numFn: VoidFn = () => 42; // ✅ — return value IGNORE qilinadi
```

</details>

---

## Covariance — Bir Yo'nalishda Moslik

### Nazariya

**Covariance** — generic type'ning parametri subtype → supertype yo'nalishida o'zgarganda, generic type ham aynan shu yo'nalishda assignable bo'lishi. Agar `Dog <: Animal` (Dog Animal'ning subtype'i), unda:

- `Dog[]` → `Animal[]` ✅ (array covariant — TS pragmatic ruxsat)
- `() => Dog` → `() => Animal` ✅ (return type covariant)
- `Promise<Dog>` → `Promise<Animal>` ✅
- `readonly Dog[]` → `readonly Animal[]` ✅ (xavfsiz)

**Nima uchun "covariant"** so'zi: type parameter'ning variance'i base type bilan **bir xil yo'nalishda** ("co-" = together). Agar `T` covariant bo'lsa, `Box<T>` ning subtyping `T` ning subtyping'iga ergashadi.

**Xavfsizlik:** covariance faqat **output (read) pozitsiyada** xavfsiz. Agar `T` type'i faqat qaytariladigan bo'lsa (`() => T`, `readonly T[]`), unda subtype qaytarilsa, uni supertype sifatida ishlatish to'g'ri (Liskov substitution).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
class Animal { constructor(public name: string) {} }
class Dog extends Animal { constructor(name: string, public breed: string) { super(name); } }

// Return type — covariant
type Producer<T> = () => T;
const getDog: Producer<Dog> = () => new Dog("Rex", "Lab");
const getAnimal: Producer<Animal> = getDog; // ✅ Dog <: Animal

// Array — covariant (o'qish uchun xavfsiz)
const dogs: Dog[] = [new Dog("Rex", "Lab")];
const animals: readonly Animal[] = dogs; // ✅ readonly covariant xavfsiz

// ⚠️ Mutable array — covariant lekin XAVFLI (unsound)
const mutableAnimals: Animal[] = dogs; // ✅ TS ruxsat beradi
mutableAnimals.push(new Animal("Cat"));
// Runtime: dogs array'da Animal (Dog emas)
// Lekin TS dogs[1] ni Dog deb hisoblaydi — type system buzilgan
// Bu TS ning bilgan unsoundness'i — array covariance ergonomics uchun saqlangan
```

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript checker'da variance type parameter'ning **ishlatilish pozitsiyasi**ga qarab aniqlanadi:

- **Output pozitsiya** (return type, getter, readonly property) → covariant
- **Input pozitsiya** (function parameter, setter) → contravariant
- **Ikkalasi** (mutable property, method bilan get+set) → invariant

Variance compiler tomonidan **marker type**'lar yordamida hisoblanadi:
1. Type parameter'ni bitta marker type bilan almashtirib, generic type'ning bir instance'i olinadi (`super`).
2. Shu type parameter'ni marker'ning subtype'i bilan almashtirib, ikkinchi instance olinadi (`sub`).
3. `sub` va `super` orasidagi assignability har ikki yo'nalishda tekshiriladi.
4. Natija: covariant (`sub → super` ✅), contravariant (`super → sub` ✅), bivariant (ikkalasi ✅), invariant (hech biri ✅).

**Variance caching:** hisoblangan variance har generic type uchun cache qilinadi. Bir xil type qayta tekshirilganda variance qaytadan hisoblanmaydi — bu katta loyihalarda type comparison'ni tezlashtiradi.

**Array covariance unsoundness:** `T[]` ham `push`, `pop` (input pozitsiya) ham `[i]` access (output pozitsiya) bor — texnik jihatdan invariant bo'lishi kerak. TS pragmatic sabablar uchun covariant qilgan (`readonly T[]` esa to'g'ri covariant).

</details>

---

## Contravariance — Teskari Yo'nalishda

### Nazariya

**Contravariance** — generic type'ning parametri subtype → supertype yo'nalishida o'zgarganda, generic type **teskari** yo'nalishda assignable bo'lishi. Function parameter pozitsiyasi contravariant (`strictFunctionTypes: true` bilan).

Agar `Dog <: Animal`, unda:
- `(animal: Animal) => void` assignable to `(dog: Dog) => void` ✅
- `(dog: Dog) => void` assignable to `(animal: Animal) => void` ❌

**Nima uchun** "contra-" = qarama-qarshi: type parameter'ning variance'i base type'ga **teskari yo'nalishda** ergashadi.

**Liskov substitution intuition:** Agar function `Dog` handler joyga ishlatilsa, lekin u `Animal` ham handle qila olsa — bu yetarli (Animal har qanday Animal'ni biladi, shu jumladan Dog'ni). Teskari — xavfli: `Dog` handler `Animal`'ni handle qila olmaydi (boshqa subtype'larni — masalan `Cat` — bilmaydi).

**`strictFunctionTypes: true` ahamiyati:** bu flag'siz function parameter'lar bivariant (ikki yo'nalishda mos) — bu unsound. Strict mode bilan parameter'lar contravariant bo'ladi (method shorthand'dan tashqari — backward compat).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Consumer<T> = (value: T) => void;

const feedAnimal: Consumer<Animal> = (a) => console.log(a.name);
const feedDog: Consumer<Dog> = (d) => console.log(d.breed);

const handler: Consumer<Dog> = feedAnimal; // ✅ contravariant
// const handler2: Consumer<Animal> = feedDog; // ❌ — breed Animal da yo'q
```

</details>

---

## Bivariance va Invariance

### Nazariya

**Bivariance** — type parameter ikki yo'nalishda ham (sub → super va super → sub) assignable. Bu **unsound** — runtime xato'larga olib keladi, lekin ergonomic. TS'da faqat method shorthand syntax'da `strictFunctionTypes` bilan ham bivariant qoladi:

```typescript
interface MethodComparer { compare(other: Animal): number; }     // BIVARIANT (method shorthand)
interface PropertyComparer { compare: (other: Animal) => number; } // CONTRAVARIANT (function property)
```

**Sabab — backward compatibility:** `Array<T>.push`, `Array<T>.indexOf` kabi built-in method'lar parameter pozitsiyasida `T` ishlatadi. Agar method shorthand contravariant bo'lganda, `number[]` ni `(number | string)[]` ga assign qilish buzilardi. `strictFunctionTypes` (TS 2.6) qo'shilganda contravariance faqat function property'larga qo'llandi, method shorthand'ga emas — mavjud kodni buzmaslik uchun.

**Invariance** — hech qanday yo'nalishda mos emas (faqat aniq teng bo'lsa). Mutable container'lar (get + set ikkalasini ham qila oladi) invariant bo'lishi kerak — bu xavfsizlik talabi:

```typescript
interface Box<T> { get(): T; set(v: T): void; }
// Box<Dog> va Box<Animal> — bir-biriga assignable EMAS:
// - get() Dog qaytarishi kerak, Animal yetmaydi (covariance kerak)
// - set(Animal) qabul qilishi kerak, Dog yetmaydi (contravariance kerak)
// Ikki talab qarama-qarshi → invariant
```

**TS 4.7+ variance modifiers** — niyatni explicit ifodalash:

```typescript
interface Producer<out T> { get(): T; }                    // Covariant
interface Consumer<in T> { accept(value: T): void; }       // Contravariant
interface MutableBox<in out T> { get(): T; set(v: T): void; } // Invariant
```

---

## `in`/`out` Variance Modifiers (TS 4.7+)

### Nazariya

Variance modifiers — generic type ning **intended variance** ni explicit belgilash:

| Modifier | Variance | Ma'nosi | Misol |
|----------|----------|---------|-------|
| `out T` | Covariant | T faqat **output** da | `Producer<out T>` |
| `in T` | Contravariant | T faqat **input** da | `Consumer<in T>` |
| `in out T` | Invariant | T ikkalasida | `MutableBox<in out T>` |

Nima uchun kerak:
1. **Documentation** — niyatni aniq qilish
2. **Error messages** — compiler yaxshiroq xato beradi
3. **Performance** — compiler variance'ni qaytadan hisoblamasdan biladi

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface ReadonlyStore<out T> {
  get(): T;
  // set(value: T): void; // ❌ — out T ni input da ishlatish mumkin emas
}

interface WriteOnlyStore<in T> {
  set(value: T): void;
  // get(): T; // ❌ — in T ni output da ishlatish mumkin emas
}

interface MutableStore<in out T> {
  get(): T;
  set(value: T): void;
}

// Covariant:
const dogStore: ReadonlyStore<Dog> = { get: () => new Dog("Rex", "Lab") };
const animalStore: ReadonlyStore<Animal> = dogStore; // ✅

// Contravariant:
const animalWriter: WriteOnlyStore<Animal> = { set: (animal) => {} };
const dogWriter: WriteOnlyStore<Dog> = animalWriter; // ✅

// Invariant:
const dogMutable: MutableStore<Dog> = { get: () => new Dog("Rex", "Labrador"), set: () => {} };
// const animalMutable: MutableStore<Animal> = dogMutable; // ❌
```

</details>

---

## Function Compatibility

### Nazariya

Function compatibility qoidalari:

1. **Parameter soni** — kamroq param mos (ko'proq emas)
2. **Parameter type** — contravariant (`strictFunctionTypes` bilan)
3. **Return type** — covariant (subtype mos)
4. **`void` return** — har qanday return mos
5. **Optional/rest** — optional param required ga mos

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type OneParam = (id: number) => string;
type TwoParam = (id: number, label: string) => string;
type NoParam = () => string;
type LiteralReturn = (id: number) => "hello";
type WideParam = (id: number | string) => string;

declare const oneParam: OneParam; declare const twoParam: TwoParam;
declare const noParam: NoParam; declare const literalReturn: LiteralReturn;
declare const wideParam: WideParam;

const c1: TwoParam = oneParam;       // ✅ kamroq param (1 < 2)
const c2: OneParam = noParam;        // ✅ kamroq param (0 < 1)
const c3: OneParam = literalReturn;  // ✅ "hello" <: string (return covariant)
const c4: OneParam = wideParam;      // ✅ number <: number|string (param contravariant)
// const c5: OneParam = twoParam;       // ❌ ko'proq param (2 > 1)
// const c6: LiteralReturn = oneParam;  // ❌ string <: "hello" emas
// const c7: WideParam = oneParam;      // ❌ number|string <: number emas
```

</details>

---

## Class va Enum Compatibility

### Nazariya

**Class** — structural compatibility'da boshqa type'lar bilan teng, lekin `private` yoki `protected` member bor bo'lsa **nominal behavior** ga o'tadi:

```typescript
class Account { private balance = 1; }
class Wallet { private balance = 1; }
// const account: Account = new Wallet(); // ❌ — private member har class'da unique identity
```

**Sabab:** `private`/`protected` — encapsulation kafolati. Agar ikki turli class'ning private property'lari structural mos kelsa, encapsulation buziladi (bir class boshqasining ichki holatiga kira oladi). Compiler har private member'ga class identity'sini bog'laydi.

**Inheritance bilan saqlanadi:** `class B extends A` — B'da A'ning private member'lari aynan shu identity bilan inherit qilinadi, shuning uchun `A` va `B` mos keladi.

**Numeric enum** — enum'lar number'lar bo'yicha mos, lekin turli enum'lar bir-biriga mos emas (nominal):

```typescript
enum Color { Red, Green }
enum Fruit { Apple, Banana }

let color: Color = Color.Red;
// color = Fruit.Apple; // ❌ — turli enum'lar mos emas
color = 0; // ✅ — 0 = Color.Red member value, raw number assignable
// color = 5; // ❌ TS 5.0 dan: enum union (Color.Red | Color.Green) ga out-of-range number mos emas
```

**String enum** — to'liq nominal: hatto bir xil string value bilan ham, boshqa enum yoki string literal mos kelmaydi:

```typescript
enum Direction { Up = "UP", Down = "DOWN" }
let direction: Direction = Direction.Up;
// direction = "UP"; // ❌ — string literal != enum member
```

**Sabab:** string enum member'i nominal identity'ga ega — class'ning `private` member'i bilan bir xil prinsipda, type identity faqat o'sha enum declaration'iga bog'lanadi. Bu refactoring'da xavfsizlik beradi (enum nomini o'zgartirsa, raw string ishlatilgan joylar topiladi).

---

## Edge Cases va Gotchas

### 1. Mutable array covariance — xavfli

```typescript
const dogs: Dog[] = [new Dog("Rex", "Lab")];
const animals: Animal[] = dogs; // ✅ TS ruxsat beradi
animals.push(new Animal("Cat")); // ❌ Runtime: dogs da Animal!
// TS bu xavfni biladi lekin pragmatik sabablarga ko'ra ruxsat beradi
// Yechim: readonly Animal[] ishlatish
```

### 2. Method shorthand bivariance

```typescript
interface Handler {
  handle(event: MouseEvent): void; // Method shorthand — BIVARIANT
}

interface SafeHandler {
  handle: (event: MouseEvent) => void; // Function property — CONTRAVARIANT
}

// Method shorthand da xavfli coercion ruxsat beriladi (backward compatibility)
// strictFunctionTypes bu holda HAM bivariant qoldiriladi
```

### 3. `void` return — substitutability rule

`void` return type — "caller return value'ni ishlatmaydi" degan kontrakt. Compiler bu signature'ga har qanday qaytaruvchi function'ni assignable qiladi (return value'ni o'qib bo'lmaydi):

```typescript
type LogHandler = () => void;
const logHandler: LogHandler = () => 42; // ✅ — return value caller'da void
const result = logHandler();
// result: void — qiymat sifatida ishlatib bo'lmaydi (number deb hisoblanmaydi)

// Amaliy foyda — forEach callback:
[1, 2].forEach((value): boolean => value > 0);
// forEach signature: (value, index, array) => void
// Callback boolean qaytaradi, lekin void signature'ga mos — forEach ignore qiladi
```

**Sabab:** ko'p JavaScript API'lar callback'lar uchun `void` return ishlatadi (`forEach`, `addEventListener`), lekin developer'lar tasodifan qaytaruvchi function beradi (`(x) => x++`). Bu xato emas — caller qiymatni ishlatmaydi.

**Diqqat:** bu rule faqat bare `void` ga tegishli, `Promise<void>` ga **kengaymaydi**. `() => Promise<number>` `() => Promise<void>` ga assignable EMAS — chunki `Promise<number>` ni `Promise<void>` ga solishtirganda `number` `void` pozitsiyasida tekshiriladi va mos kelmaydi (TS issue [#49755](https://github.com/microsoft/TypeScript/issues/49755), hozircha ochiq). Sync `boolean → void` ishlaydi, lekin async `Promise<boolean> → Promise<void>` ishlamaydi:

```typescript
type SyncLog = () => void;
const syncLog: SyncLog = () => 42; // ✅ — bare void: return ignore qilinadi

type AsyncLog = () => Promise<void>;
// const asyncLog: AsyncLog = async () => 42; // ❌ — Promise<number> != Promise<void>
const asyncLog: AsyncLog = async () => { await Promise.resolve(); }; // ✅
```

### 4. Excess property check bypass — variable orqali

```typescript
interface Config { host: string; port: number; }

// Literal'da — excess check
// const config: Config = { host: "localhost", port: 8080, typo: true }; // ❌

// Variable orqali — bypass
const rawConfig = { host: "localhost", port: 8080, typo: true };
const config: Config = rawConfig; // ✅ — excess property ruxsat
```

### 5. `in`/`out` modifier qo'yilmagan interface — implicit variance

```typescript
// in/out yo'q — TS variance'ni type ichidan aniqlaydi
interface Container<T> { readonly value: T; }
// `readonly value: T` faqat read pozitsiyada — TS uni covariant deb hisoblaydi

// Mutable property — read va write ikkalasi → invariant (implicit)
interface MutableBox<T> { value: T; }

// Explicit — compiler'ga aniqroq signal beradi (katta loyihalarda performance)
interface AnnotatedBox<in out T> { value: T; }
// Invariant — mutable data uchun aniq belgilash
```

`in`/`out` annotation'lari to'g'ri ishlatilganda variance tekshiruvini tezlashtiradi va niyatni aniq qiladi. Implicit inference odatda to'g'ri ishlaydi, lekin annotation niyatni hujjatlashtiradi va compiler variance'ni structural expansion'siz biladi.

---

## Common Mistakes

### ❌ Xato 1: Mutable array ni covariant ishlatish

```typescript
// ❌ — mutable covariance: dogs array'ga Animal qo'shiladi
const mutableAnimals: Animal[] = dogs;
mutableAnimals.push(new Animal("Cat"));

// ✅ — readonly bilan xavfsiz (push mavjud emas)
const safeAnimals: readonly Animal[] = dogs;
```

### ❌ Xato 2: Method shorthand bilan bivariance ga tushish

```typescript
// ❌ — bivariant (xavfli)
interface BivariantComparer { compare(other: Product): number; }

// ✅ — contravariant (xavfsiz)
interface SafeComparer { compare: (other: Product) => number; }
```

### ❌ Xato 3: Function param soni tekshirmaslik

```typescript
// ❌ — kamroq param mos — lekin bu feature, xato emas!
[1, 2, 3].map((x) => x * 2); // (value, index, array) kutiladi, 1 param berildi — ✅
// Bu JavaScript callback convention — optional param skip
```

### ❌ Xato 4: `strictFunctionTypes` o'chirilganda xavfni bilmaslik

```typescript
// strictFunctionTypes: false — barcha function param bivariant
// Bu xavfli — runtime type error larga olib keladi
// Doim strict: true ishlatish!
```

### ❌ Xato 5: String enum'ga raw string literal assign qilish

```typescript
enum Color { Red = "RED", Green = "GREEN" }
// const color: Color = "RED"; // ❌ — string enum nominal, raw string mos emas
const color: Color = Color.Red; // ✅ — faqat enum member orqali
```

---

## Amaliy Mashqlar

### Mashq 1: Structural Typing (Oson)

**Savol:** Qaysilari ishlaydi?

```typescript
interface Bird { fly(): void; layEggs(): void; }
interface Fish { swim(): void; layEggs(): void; }
interface FlyingFish { fly(): void; swim(): void; layEggs(): void; }

declare const bird: Bird; declare const fish: Fish; declare const flyingFish: FlyingFish;

const asBird: Bird = flyingFish;          // ?
const asFish: Fish = flyingFish;          // ?
const asFlyingFish: FlyingFish = bird;    // ?
const asBoth: Bird & Fish = flyingFish;   // ?
```

<details>
<summary>Javob</summary>

```typescript
const asBird: Bird = flyingFish;          // ✅ — fly + layEggs bor
const asFish: Fish = flyingFish;          // ✅ — swim + layEggs bor
const asFlyingFish: FlyingFish = bird;    // ❌ — swim missing
const asBoth: Bird & Fish = flyingFish;   // ✅ — fly + swim + layEggs = FlyingFish
```

</details>

---

### Mashq 2: Variance (O'rta)

```typescript
class Animal { name = "Rex"; }
class Dog extends Animal { breed = "Labrador"; }

type Producer<T> = () => T;
type Consumer<T> = (value: T) => void;

declare const pDog: Producer<Dog>; declare const pAnimal: Producer<Animal>;
declare const cDog: Consumer<Dog>; declare const cAnimal: Consumer<Animal>;

const produceAnimal: Producer<Animal> = pDog;   // ?
const produceDog: Producer<Dog> = pAnimal;      // ?
const consumeDog: Consumer<Dog> = cAnimal;      // ?
const consumeAnimal: Consumer<Animal> = cDog;   // ?
```

<details>
<summary>Javob</summary>

```typescript
const produceAnimal: Producer<Animal> = pDog;   // ✅ Covariant
const produceDog: Producer<Dog> = pAnimal;      // ❌ breed missing
const consumeDog: Consumer<Dog> = cAnimal;      // ✅ Contravariant
const consumeAnimal: Consumer<Animal> = cDog;   // ❌ breed Animal da yo'q
```

</details>

---

### Mashq 3: Function Compatibility (O'rta)

```typescript
type OneParam = (id: number) => string;
type TwoParam = (id: number, label: string) => string;
type NoParam = () => string;
type LiteralReturn = (id: number) => "hello";
type WideParam = (id: number | string) => string;

declare const oneParam: OneParam;
declare const twoParam: TwoParam;
declare const noParam: NoParam;
declare const literalReturn: LiteralReturn;
declare const wideParam: WideParam;

// Qaysilari ishlaydi?
const check1: TwoParam = oneParam;
const check2: OneParam = noParam;
const check3: OneParam = literalReturn;
const check4: OneParam = wideParam;
```

<details>
<summary>Javob</summary>

```
check1: ✅ kamroq param (1 < 2)
check2: ✅ kamroq param (0 < 1)
check3: ✅ "hello" <: string (return covariant)
check4: ✅ number <: number|string (param contravariant)
```

</details>

---

### Mashq 4: in/out Modifiers (Qiyin)

**Savol:** `ReadOnlyStore<out T>`, `WriteOnlyStore<in T>`, `MutableStore<in out T>` yarating va mos keladigan assignment larni ko'rsating.

<details>
<summary>Javob</summary>

```typescript
interface ReadOnlyStore<out T> { get(): T; }
interface WriteOnlyStore<in T> { set(v: T): void; }
interface MutableStore<in out T> { get(): T; set(v: T): void; }

const dogRead: ReadOnlyStore<Dog> = { get: () => new Dog("Rex", "Labrador") };
const animalRead: ReadOnlyStore<Animal> = dogRead; // ✅ covariant

const animalWrite: WriteOnlyStore<Animal> = { set: () => {} };
const dogWrite: WriteOnlyStore<Dog> = animalWrite; // ✅ contravariant

const dogMut: MutableStore<Dog> = { get: () => new Dog("Rex", "Labrador"), set: () => {} };
// const animalMut: MutableStore<Animal> = dogMut; // ❌ invariant
```

</details>

---

### Mashq 5: Class Compatibility (O'rta)

```typescript
class PublicPoint { x = 1; }
class PublicVector { x = 1; }
class PrivateAccount { private x = 1; }
class PrivateWallet { private x = 1; }

const point: PublicPoint = new PublicVector();    // ?
const account: PrivateAccount = new PrivateWallet(); // ?
```

<details>
<summary>Javob</summary>

```typescript
const point: PublicPoint = new PublicVector();       // ✅ — structural (public x: number ikkalasida)
const account: PrivateAccount = new PrivateWallet(); // ❌ — private member → nominal behavior
```

</details>

---

## Xulosa

| Variance | Qoida | Qaerda |
|----------|-------|--------|
| **Covariant** | Sub → Super ✅ | Return type, readonly, Promise |
| **Contravariant** | Super → Sub ✅ | Function params (strict) |
| **Bivariant** | Ikkalasi ✅ ⚠️ | Method shorthand (legacy) |
| **Invariant** | Faqat o'zi ✅ | Mutable read+write data |

**Modifiers (TS 4.7+):** `out T` = covariant, `in T` = contravariant, `in out T` = invariant.

**Qoidalar:** `strictFunctionTypes: true` doim. Function property > method shorthand. `readonly` array param larda.

**Bog'liq:** [Bo'lim 6: Type Narrowing](06-type-narrowing.md), [Bo'lim 16: Brand](16-custom-utility-types.md), [Bo'lim 22: tsconfig](22-tsconfig.md).

---

**Keyingi bo'lim:** [26-ts-5x-features.md](26-ts-5x-features.md) — TypeScript 5.0 dan 5.8 gacha barcha yangiliklar.
