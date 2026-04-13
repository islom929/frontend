# Interview: Type Compatibility va Variance

> Covariance, contravariance, bivariance, invariance, `in`/`out` modifiers, function compatibility, array covariance hole bo'yicha interview savollari. Structural typing — [interview/01](01-ts-intro.md), [interview/04](04-objects-interfaces.md).

---

## Nazariy savollar

### 1. Covariance va contravariance nima?

<details>
<summary>Javob</summary>

**Variance** — generic type larda subtype munosabatining yo'nalishi. `Dog <: Animal` bo'lganda:

**Covariant (bir yo'nalish)** — return type, readonly property:

```typescript
class Animal { name = "" }
class Dog extends Animal { breed = "" }

type AnimalFactory = () => Animal;
type DogFactory = () => Dog;

const makeDog: DogFactory = () => new Dog();
const makeAnimal: AnimalFactory = makeDog; // ✅ Dog qaytaruvchi Animal o'rnida ishlaydi
```

**Contravariant (teskari yo'nalish)** — function parameter (`strictFunctionTypes`):

```typescript
type AnimalHandler = (a: Animal) => void;
type DogHandler = (d: Dog) => void;

const logName: AnimalHandler = (a) => console.log(a.name);
const logDog: DogHandler = logName; // ✅ Animal handler Dog ni ham handle qiladi

// TESKARI — xavfli:
// const handler: AnimalHandler = (d: Dog) => console.log(d.breed);
// ❌ breed faqat Dog da — Animal berilsa crash
```

**Nima uchun contravariant xavfsiz?** `logName` faqat `name` ishlatadi — Dog da ham bor. Teskari da `breed` ishlatilsa — oddiy Animal da yo'q.

</details>

### 2. Bivariance nima? Method shorthand vs function property?

<details>
<summary>Javob</summary>

`strictFunctionTypes: true` da:

| Syntax | Variance |
|--------|----------|
| Method shorthand: `foo(x: T): void` | **Bivariant** (ikki yo'nalish) |
| Function property: `foo: (x: T) => void` | **Contravariant** (xavfsiz) |

```typescript
interface Handler1 { handle(a: Animal): void }     // bivariant
interface Handler2 { handle: (a: Animal) => void }  // contravariant

const dogHandler = { handle(d: Dog) { console.log(d.breed) } };

const h1: Handler1 = dogHandler; // ✅ bivariant — lekin XAVFLI
// const h2: Handler2 = dogHandler; // ❌ contravariant — xavfsiz

h1.handle(new Cat()); // Runtime: Cat da breed yo'q → undefined!
```

**Nima uchun bivariant qoldirilgan?** DOM API (`addEventListener`) va Node.js (`EventEmitter`) bivariant method larga tayanadi. Contravariant qilinsa — ecosystem buziladi.

**Tavsiya:** ESLint `@typescript-eslint/method-signature-style: "property"`.

</details>

### 3. Invariance nima? `in`/`out` modifiers?

<details>
<summary>Javob</summary>

**Invariant** — T ham input ham output da → hech qanday yo'nalishda moslik yo'q:

```typescript
interface MutableBox<T> {
  get: () => T;            // covariant (output)
  set: (value: T) => void; // contravariant (input)
  // Ikkalasi → INVARIANT
}
```

**`in`/`out` modifiers (TS 4.7+)** — compiler ga variance ni explicit aytish:

```typescript
interface Producer<out T> { produce(): T }         // Covariant
interface Consumer<in T> { consume(v: T): void }    // Contravariant
interface Processor<in out T> { process(v: T): T }  // Invariant
```

Noto'g'ri position da xato:

```typescript
interface Wrong<out T> {
  consume(value: T): void; // ❌ Error — T input da, lekin 'out' belgilangan
}
```

**Foyda:** ~5-10% tezroq compilation, self-documenting, noto'g'ri ishlatish compile error.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Structural typing — qaysi compile bo'ladi? (Daraja: Middle)

**Savol:**

```typescript
interface User { id: number; name: string }
interface Employee { id: number; name: string; department: string }

const emp: Employee = { id: 1, name: "Ali", department: "Eng" };
const user: User = emp;                                    // A
// const emp2: Employee = user;                            // B
const obj = { id: 1, name: "Ali", department: "Eng", salary: 5000 };
const user2: User = obj;                                   // C
const user3: User = { id: 1, name: "Ali", extra: true };  // D
```

<details>
<summary>Yechim</summary>

- **A: ✅** — `Employee` da `id`, `name` bor, extra `department` OK (structural)
- **B: ❌** — `user` da `department` yo'q, `Employee` talab qiladi
- **C: ✅** — variable orqali — excess property check **yo'q**
- **D: ❌** — **object literal** da excess property check ishlaydi

</details>

### 2. Array covariance hole (Daraja: Middle+)

**Savol:** Output ni ayting. Nima uchun xavfli?

```typescript
class Animal { name = "Animal" }
class Dog extends Animal { breed = "Lab" }

const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;
animals.push(new Animal());

console.log(dogs.length);
console.log(dogs[1].breed);
```

<details>
<summary>Yechim</summary>

```
2
undefined
```

`dogs` va `animals` **bir xil reference**. `animals.push(new Animal())` aslida `dogs` ga `Animal` qo'shdi. `dogs[1]` — TS `Dog` deydi, lekin runtime da oddiy `Animal` (`breed` yo'q) → `undefined`.

**Array covariance — TS ning soundness hole.** Xavfsiz variant: `readonly Dog[]` → `readonly Animal[]` (push yo'q).

</details>

### 3. Function variance — qaysi compile bo'ladi? (Daraja: Middle+)

**Savol:** `strictFunctionTypes: true` da:

```typescript
class Animal { name = "" }
class Dog extends Animal { breed = "" }

type AnimalHandler = (a: Animal) => void;
type DogHandler = (d: Dog) => void;

const logName: AnimalHandler = (a) => console.log(a.name);
const logBreed: DogHandler = (d) => console.log(d.breed);

const a: DogHandler = logName;     // A
const b: AnimalHandler = logBreed; // B
```

<details>
<summary>Yechim</summary>

- **A: ✅** — contravariant. `AnimalHandler` → `DogHandler` — Animal handler Dog ni ham handle qiladi (`name` bor)
- **B: ❌** — `DogHandler` → `AnimalHandler` xavfli. `logBreed` `breed` ishlatadi — oddiy `Animal` da `breed` yo'q

Function parameter lar **contravariant** — kengroq type qabul qiluvchi torlroq joyga mos keladi. Teskari — xavfli.

</details>

### 4. Variance — GUIDE #19: tushuntirish + misol (Daraja: Senior)

**Savol:** `Producer<T>`, `Consumer<T>`, `Processor<T>` uchun qaysi assign ishlaydi? `in`/`out` bilan yozing:

```typescript
class Animal { name = "" }
class Dog extends Animal { breed = "" }

// dogProducer: Producer<Dog> → animalProducer: Producer<Animal>?
// animalConsumer: Consumer<Animal> → dogConsumer: Consumer<Dog>?
// dogProcessor: Processor<Dog> → animalProcessor: Processor<Animal>?
```

<details>
<summary>Yechim</summary>

```typescript
interface Producer<out T> { produce(): T }
interface Consumer<in T> { consume(v: T): void }
interface Processor<in out T> { process(v: T): T }

declare const dogProducer: Producer<Dog>;
declare const animalConsumer: Consumer<Animal>;
declare const dogProcessor: Processor<Dog>;

const ap: Producer<Animal> = dogProducer;  // ✅ Covariant — Dog producer Animal o'rnida
const dc: Consumer<Dog> = animalConsumer;  // ✅ Contravariant — Animal consumer Dog o'rnida

// const aProc: Processor<Animal> = dogProcessor; // ❌ Invariant — hech qanday yo'nalishda mos emas
// const dProc: Processor<Dog> = animalProcessor; // ❌ Invariant
```

| Type | Variance | Dog → Animal | Animal → Dog |
|------|----------|:------------:|:------------:|
| `Producer<out T>` | Covariant | ✅ | ❌ |
| `Consumer<in T>` | Contravariant | ❌ | ✅ |
| `Processor<in out T>` | Invariant | ❌ | ❌ |

</details>

---

## Xulosa

- Covariant — return type, readonly. `Dog <: Animal` → `F<Dog> <: F<Animal>`
- Contravariant — parameter. `Dog <: Animal` → `F<Animal> <: F<Dog>` (teskari!)
- Bivariant — method shorthand (xavfli, tarixiy sabab)
- Invariant — ham input ham output → hech qanday yo'nalish
- `in`/`out` modifiers (TS 4.7+) — explicit variance, tezroq compile
- Array covariance — TS ning soundness hole. `readonly` bilan xavfsiz
- Method shorthand bivariant, function property contravariant — `@typescript-eslint/method-signature-style`

[Asosiy bo'limga qaytish →](../25-type-compatibility.md)
