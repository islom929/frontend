# Bo'lim 25: Type Compatibility va Variance

> Type compatibility — TypeScript type system'ning asosiy mexanizmi: bir type'ning ikkinchisiga assignable bo'lishini aniqlash. Structural typing, covariance, contravariance, bivariance, invariance — generic type'lar bir-biriga qanday mos kelishini belgilaydi. `in`/`out` variance modifier'lar (TS 4.7+) bilan explicit control. Variance — interview va kompleks type design'da kritik mavzu.

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

const p: Point = { x: 1, y: 2 };
const c: Coordinate = p; // ✅ — shakli bir xil, nomi farq qilmaydi
```

**Excess property checking** — fresh object literal'da qo'shimcha property bor bo'lsa, compiler error beradi. Bu typo'larni ushlash uchun mexanizm:

```typescript
interface User { name: string; age: number; }

const user: User = { name: "Ali", age: 25, email: "a@b.com" };
// ❌ — Object literal may only specify known properties, and 'email' does not exist
```

**Nima uchun faqat literal'da:** literal — programmer'ning intention'ini bevosita ifodalaydi (typo bo'lishi mumkin). Variable orqali assign — value boshqa kontekstda ishlatilgan bo'lishi mumkin, qo'shimcha property'lar normal:

```typescript
const obj = { name: "Ali", age: 25, email: "a@b.com" };
const user2: User = obj; // ✅ — variable orqali bypass (structural mos)
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

**Performance:** checker recursive compatibility'ni cache qiladi (`relationCache`). Bir xil juftlik (`A`, `B`) ikkinchi marta tekshirilmaydi — bu millionlab type comparison'larni tezlashtiradi.

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

**Covariance** — generic type'ning parametri subtype → supertype yo'nalishida o'zgarganda, generic type ham xuddi shu yo'nalishda assignable bo'lishi. Agar `Dog <: Animal` (Dog Animal'ning subtype'i), unda:

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

Algoritm `getVarianceOfTypeParameter` funksiyasida:
1. Type parameter'ni `marker` type bilan almashtirish (var A).
2. Type parameter'ni `marker`'ning supertype'i bilan almashtirish (var B).
3. `A` va `B` orasidagi assignability'ni har yo'nalishda tekshirish.
4. Natija: covariant (`A → B` ✅), contravariant (`B → A` ✅), bivariant (ikkalasi ✅), invariant (hech biri ✅).

**Variance caching:** `getVarianceOfTypeParameter` natijasi har generic type uchun cache qilinadi (`variances` field). Cross-file inference'da yana hisoblanmaydi.

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
interface BoxMethod { compare(other: Box): number; }     // BIVARIANT (method shorthand)
interface BoxProp   { compare: (other: Box) => number; } // CONTRAVARIANT (function property)
```

**Sabab — backward compatibility:** `Array.prototype.push`, DOM event handler'lar (`addEventListener("click", (e: MouseEvent) => {})`) — method shorthand bilan yozilgan, ko'p library'lar bivariance'ga bog'liq. `strictFunctionTypes` qo'shilganda contravariance method shorthand'ga qo'llanmadi (mavjud kodni buzmaslik uchun).

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
2. **Error messages** — kompilator yaxshiroq xato beradi
3. **Performance** — kompilator variance ni tekshirmasdan biladi

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
const animalWriter: WriteOnlyStore<Animal> = { set: (a) => {} };
const dogWriter: WriteOnlyStore<Dog> = animalWriter; // ✅

// Invariant:
const dogMutable: MutableStore<Dog> = { get: () => new Dog("X", "Y"), set: () => {} };
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
type F1 = (a: number) => string;
type F2 = (a: number, b: string) => string;
type F4 = () => string;
type F5 = (a: number) => "hello";
type F7 = (a: number | string) => string;

declare const f1: F1; declare const f2: F2;
declare const f4: F4; declare const f5: F5; declare const f7: F7;

const t1: F2 = f1;  // ✅ kamroq param (1 < 2)
const t2: F1 = f4;  // ✅ kamroq param (0 < 1)
const t3: F1 = f5;  // ✅ "hello" <: string (return covariant)
const t4: F1 = f7;  // ✅ number <: number|string (param contravariant)
// const t5: F1 = f2;  // ❌ ko'proq param (2 > 1)
// const t6: F5 = f1;  // ❌ string <: "hello" emas
// const t7: F7 = f1;  // ❌ number|string <: number emas
```

</details>

---

## Class va Enum Compatibility

### Nazariya

**Class** — structural compatibility'da boshqa type'lar bilan teng, lekin `private` yoki `protected` member bor bo'lsa **nominal behavior** ga o'tadi:

```typescript
class A { private x = 1; }
class B { private x = 1; }
// const a: A = new B(); // ❌ — private member har class'da unique identity
```

**Sabab:** `private`/`protected` — encapsulation kafolati. Agar ikki turli class'ning private property'lari structural mos kelsa, encapsulation buziladi (bir class boshqasining ichki holatiga kira oladi). Compiler har private member'ga class identity'sini bog'laydi.

**Inheritance bilan saqlanadi:** `class B extends A` — B'da A'ning private member'lari xuddi shu identity bilan inherit qilinadi, shuning uchun `A` va `B` mos keladi.

**Numeric enum** — enum'lar number'lar bo'yicha mos, lekin turli enum'lar bir-biriga mos emas (nominal):

```typescript
enum Color { Red, Green }
enum Fruit { Apple, Banana }

let c: Color = Color.Red;
// c = Fruit.Apple; // ❌ — turli enum'lar mos emas
c = 0; // ✅ — number → numeric enum (TS 5.0 dan ogohlantirish)
```

**String enum** — to'liq nominal: hatto bir xil string value bilan ham, boshqa enum yoki string literal mos kelmaydi:

```typescript
enum Direction { Up = "UP", Down = "DOWN" }
let d: Direction = Direction.Up;
// d = "UP"; // ❌ — string literal != enum member
```

**Sabab:** string enum identity'si — xuddi class private member'i kabi nominal. Bu refactoring'da xavfsizlik beradi (enum nomini o'zgartirsa, raw string ishlatilgan joylar topiladi).

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
type VoidFn = () => void;
const fn: VoidFn = () => 42; // ✅ — return value caller'da void
const result = fn();
// result: void — TypeScript result'ni ishlatishni taqiqlaydi

// Amaliy foyda — forEach callback:
[1, 2].forEach((v): boolean => v > 0);
// forEach signature: (value, index, array) => void
// Callback boolean qaytaradi, lekin void signature'ga mos — forEach ignore qiladi
```

**Sabab:** ko'p JavaScript API'lar callback'lar uchun `void` return ishlatadi (`forEach`, `addEventListener`), lekin developer'lar tasodifan qaytaruvchi function beradi (`(x) => x++`). Bu xato emas — caller qiymatni ishlatmaydi.

**Diqqat:** `Promise<void>` o'xshash ishlaydi: `() => Promise<number>` assignable to `() => Promise<void>`. Bu unhandled promise rejection'ga olib kelishi mumkin (caller `.then` chain'ini kutmasa).

### 4. Excess property check bypass — variable orqali

```typescript
interface Config { host: string; port: number; }

// Literal da — excess check
// const c: Config = { host: "x", port: 1, typo: true }; // ❌

// Variable orqali — bypass
const obj = { host: "x", port: 1, typo: true };
const c: Config = obj; // ✅ — excess property ruxsat
```

### 5. `in`/`out` modifier qo'yilmagan interface — implicit variance

```typescript
// in/out yo'q — TS variance'ni tip ichidan aniqlaydi
interface Box<T> { value: T; }
// `value: T` faqat read pozitsiyada ko'rinsa, TS uni covariant deb hisoblaydi

// Explicit — kompilatorga aniqroq sigal beradi (kuchli loyihalarda performance)
interface MutableBox<in out T> { value: T; }
// Invariant — mutable data uchun aniq belgilash
```

`in`/`out` annotation'lari to'g'ri ishlatilganda variance tekshiruvini tezlashtiradi va niyatni aniq qiladi. Implicit inference odatda to'g'ri ishlaydi, lekin annotation niyatni hujjatlashtiradi va kompilator complex tip'larda ham ishlay oladi.

---

## Common Mistakes

### ❌ Xato 1: Mutable array ni covariant ishlatish

```typescript
// ❌ — dogs da Animal paydo bo'ladi
const animals: Animal[] = dogs;
animals.push(new Animal("Cat"));

// ✅ — readonly bilan xavfsiz
const animals: readonly Animal[] = dogs;
```

### ❌ Xato 2: Method shorthand bilan bivariance ga tushish

```typescript
// ❌ — bivariant (xavfli)
interface Cmp { compare(other: Base): number; }

// ✅ — contravariant (xavfsiz)
interface Cmp { compare: (other: Base) => number; }
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

### ❌ Xato 5: String enum ni number ga assign qilish

```typescript
enum Color { Red = "RED", Green = "GREEN" }
// const c: Color = "RED"; // ❌ — string enum nominal!
const c: Color = Color.Red; // ✅ — faqat enum member orqali
```

---

## Amaliy Mashqlar

### Mashq 1: Structural Typing (Oson)

**Savol:** Qaysilari ishlaydi?

```typescript
interface Bird { fly(): void; layEggs(): void; }
interface Fish { swim(): void; layEggs(): void; }
interface FlyingFish { fly(): void; swim(): void; layEggs(): void; }

declare const bird: Bird; declare const fish: Fish; declare const ff: FlyingFish;

const a: Bird = ff;        // ?
const b: Fish = ff;        // ?
const c: FlyingFish = bird; // ?
const d: Bird & Fish = ff;  // ?
```

<details>
<summary>Javob</summary>

```typescript
const a: Bird = ff;        // ✅ — fly + layEggs bor
const b: Fish = ff;        // ✅ — swim + layEggs bor
const c: FlyingFish = bird; // ❌ — swim missing
const d: Bird & Fish = ff;  // ✅ — fly + swim + layEggs = FlyingFish
```

</details>

---

### Mashq 2: Variance (O'rta)

```typescript
class Animal { name = "a"; }
class Dog extends Animal { breed = "d"; }

type Producer<T> = () => T;
type Consumer<T> = (v: T) => void;

declare const pDog: Producer<Dog>; declare const pAnimal: Producer<Animal>;
declare const cDog: Consumer<Dog>; declare const cAnimal: Consumer<Animal>;

const a: Producer<Animal> = pDog;   // ?
const b: Producer<Dog> = pAnimal;   // ?
const c: Consumer<Dog> = cAnimal;   // ?
const d: Consumer<Animal> = cDog;   // ?
```

<details>
<summary>Javob</summary>

```typescript
const a: Producer<Animal> = pDog;   // ✅ Covariant
const b: Producer<Dog> = pAnimal;   // ❌ breed missing
const c: Consumer<Dog> = cAnimal;   // ✅ Contravariant
const d: Consumer<Animal> = cDog;   // ❌ breed Animal da yo'q
```

</details>

---

### Mashq 3: Function Compatibility (O'rta)

```typescript
type F1 = (a: number) => string;
type F2 = (a: number, b: string) => string;
type F3 = () => string;
type F4 = (a: number) => "hello";
type F5 = (a: number | string) => string;

declare const f1: F1;
declare const f2: F2;
declare const f3: F3;
declare const f4: F4;
declare const f5: F5;

// Qaysilari ishlaydi?
const t1: F2 = f1;
const t2: F1 = f3;
const t3: F1 = f4;
const t4: F1 = f5;
```

<details>
<summary>Javob</summary>

```
t1: ✅ kamroq param (1 < 2)
t2: ✅ kamroq param (0 < 1)
t3: ✅ "hello" <: string (return covariant)
t4: ✅ number <: number|string (param contravariant)
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

const dogRead: ReadOnlyStore<Dog> = { get: () => new Dog("X", "Y") };
const animalRead: ReadOnlyStore<Animal> = dogRead; // ✅ covariant

const animalWrite: WriteOnlyStore<Animal> = { set: () => {} };
const dogWrite: WriteOnlyStore<Dog> = animalWrite; // ✅ contravariant

const dogMut: MutableStore<Dog> = { get: () => new Dog("X", "Y"), set: () => {} };
// const animalMut: MutableStore<Animal> = dogMut; // ❌ invariant
```

</details>

---

### Mashq 5: Class Compatibility (O'rta)

```typescript
class A { x = 1; }
class B { x = 1; }
class C { private x = 1; }
class D { private x = 1; }

const a: A = new B(); // ?
const b: C = new D(); // ?
```

<details>
<summary>Javob</summary>

```typescript
const a: A = new B(); // ✅ — structural (public x: number ikkalasida)
const b: C = new D(); // ❌ — private member → nominal behavior
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
