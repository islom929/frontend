# Bo'lim 25: Type Compatibility va Variance

> Type compatibility — TypeScript type system ning **asosiy mexanizmi**. Structural typing, covariance, contravariance, bivariance, invariance — generic type lar qanday bir-biriga mos kelishini tushunish. `in`/`out` variance modifiers bilan explicit control. Bu interview larda eng ko'p so'raladigan va eng kam tushunilgan mavzulardan biri.

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

TypeScript **structural typing** ishlatadi — type ning **nomi** emas, **shakli** (shape) muhim. Agar A type ning barcha property lari B type da bor bo'lsa — A B ga **assignable**.

Bu **nominal typing** (Java, C#) dan farqli — u yerda type nomi mos kelishi kerak.

```typescript
interface Point { x: number; y: number; }
interface Coordinate { x: number; y: number; }

const p: Point = { x: 1, y: 2 };
const c: Coordinate = p; // ✅ — shakli bir xil, nomi farq qilmaydi
```

**Excess property checking** — faqat **fresh object literal** da ishlaydi (typo catcher):

```typescript
interface User { name: string; age: number; }

const user: User = { name: "Ali", age: 25, email: "a@b.com" };
// ❌ — excess property "email" (literal da)

const obj = { name: "Ali", age: 25, email: "a@b.com" };
const user2: User = obj; // ✅ — variable orqali bypass
```

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

**Covariant** — subtype → supertype yo'nalishida moslik. Agar `Dog extends Animal`, unda:

- `Dog[]` → `Animal[]` ✅ (array covariant)
- `() => Dog` → `() => Animal` ✅ (return type covariant)
- `Promise<Dog>` → `Promise<Animal>` ✅
- `readonly Dog[]` → `readonly Animal[]` ✅

Covariance **o'qish** uchun xavfsiz — agar Dog qaytarilsa, uni Animal sifatida o'qish mumkin.

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

// ⚠️ Mutable array — covariant lekin XAVFLI
const mutableAnimals: Animal[] = dogs; // ✅ TS ruxsat beradi
mutableAnimals.push(new Animal("Cat")); // ❌ Runtime: dogs array da Animal!
// Bu TS ning bilgan cheklovi — strictness vs ergonomics trade-off
```

</details>

---

## Contravariance — Teskari Yo'nalishda

### Nazariya

**Contravariant** — supertype → subtype yo'nalishida moslik. Function **parameter** lari contravariant (`strictFunctionTypes: true` bilan):

Agar `Dog extends Animal`, unda:
- `(animal: Animal) => void` → `(dog: Dog) => void` ✅ (contravariant)
- `(dog: Dog) => void` → `(animal: Animal) => void` ❌

Nima uchun? Agar `Dog` handler kutilsa va `Animal` handler berilsa — bu xavfsiz (Animal har qanday Animal ni handle qiladi, shu jumladan Dog). Lekin teskari — xavfli (Dog handler faqat Dog biladi, boshqa Animal ni handle qilolmaydi).

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

**Bivariant** — ikki yo'nalishda mos (xavfli!). Method shorthand syntax da `strictFunctionTypes` bilan ham bivariant qoladi:

```typescript
interface Box {
  // Method shorthand — BIVARIANT (xavfli)
  compare(other: Box): number;

  // Function property — CONTRAVARIANT (xavfsiz)
  compare: (other: Box) => number;
}
```

**Invariant** — hech qanday yo'nalishda mos emas. Mutable data (read + write) invariant bo'lishi kerak.

**TS 4.7+ variance modifiers:**

```typescript
interface Producer<out T> { get(): T; }          // Covariant
interface Consumer<in T> { accept(value: T): void; } // Contravariant
interface MutableBox<in out T> { get(): T; set(value: T): void; } // Invariant
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

**Class** — structural, lekin `private`/`protected` member bor bo'lsa nominal behavior:

```typescript
class A { private x = 1; }
class B { private x = 1; }
// const a: A = new B(); // ❌ — private member lar turli class larda nominal!
```

**Enum** — numeric enum lar number ga mos, lekin turli enum lar bir-biriga mos EMAS:

```typescript
enum Color { Red, Green }
enum Fruit { Apple, Banana }

let c: Color = Color.Red;
// c = Fruit.Apple; // ❌ — turli enum lar mos emas
c = 0; // ✅ — number → numeric enum
```

String enum lar **hech qanday** boshqa type ga mos emas (nominal behavior).

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

### 3. `void` return — kutilmagan behavior

```typescript
type VoidFn = () => void;
const fn: VoidFn = () => 42; // ✅ — return value IGNORE qilinadi
const result = fn();
// result: void — 42 ni ishlatib bo'lmaydi

// Lekin forEach callback da:
[1, 2].forEach(() => true); // ✅ — void callback, return ignore
```

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
// in/out yo'q — TS o'zi aniqlaydi (slow, ba'zan noto'g'ri)
interface Box<T> { value: T; }
// TS buni covariant deb aniqlaydi (value faqat read ko'rinadi)

// Explicit — tezroq va aniqroq
interface Box<in out T> { value: T; }
// Invariant — mutable data uchun TO'G'RI
```

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

// Qaysilari ishlaydi?
const t1: F2 = f1; const t2: F1 = f3; const t3: F1 = f4; const t4: F1 = f5;
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
