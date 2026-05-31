# Interview: Type Compatibility va Variance

> Structural compatibility, assignability qoidalari, variance (covariance, contravariance, bivariance, invariance), `strictFunctionTypes`, method bivariance, `in`/`out` modifiers, fresh object literal types va excess property check bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Output savollar](#output-savollar)
- [Amaliy savollar (Coding)](#amaliy-savollar-coding)
- [Bug fix savollar](#bug-fix-savollar)

---

## Nazariy savollar

### Savol 1: Structural typing nima va nominal typing dan qanday farq qiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Structural typing — TypeScript ikki type'ni **shape** (property to'plami) bo'yicha taqqoslaydi, nomi muhim emas. Java va C# nominal typing — type nomi mos kelishi shart.

### To'liq tushuntirish

TypeScript da type identity **shape** bilan aniqlanadi. A type B ga **assignable** bo'lishi uchun A B ning barcha required member'larini taklif qilishi shart (member nomi va type'i mos kelishi bilan). A qo'shimcha member'larga ega bo'lishi mumkin — bu mos kelishni buzmaydi (lekin fresh object literal'da excess property check alohida qoida). Duck typing printsipiga asoslangan: type'ning identity'si nomi bilan emas, balki taklif qiladigan member to'plami bilan aniqlanadi.

Nominal typing (Java, C#, Rust) — type identity nom orqali aniqlanadi. `class Point` va `class Coordinate` bir xil maydonlarga ega bo'lsa ham — bir-biriga mos emas, chunki nomi farqli. TypeScript bu yondashuvni faqat `private`/`protected` member'li class'lar uchun simulyatsiya qiladi.

Structural typing — interface va `type` alias uchun farq yo'q. Ikkalasi shape orqali tekshiriladi. Qachon class structural emas — `private`/`protected` member bor bo'lganda. Bunda compiler member origin (qaysi class declaration'dan kelgan) ni tekshiradi.

### Kod misol

```typescript
interface Point { x: number; y: number; }
interface Coordinate { x: number; y: number; }

const p: Point = { x: 1, y: 2 };
const c: Coordinate = p;  // ✅ — shape mos, nom muhim emas

// Class structural — public member only
class CityA { name = ""; }
class CityB { name = ""; }
const cityA: CityA = new CityB();  // ✅ — shape mos

// Class nominal — private/protected member
class SecureA { private token = ""; }
class SecureB { private token = ""; }
// const secure: SecureA = new SecureB();  // ❌ private member origin'i farqli
```

### Edge Cases

- **`private` member'lar nominal behavior beradi** — bir xil nomli private maydon turli class declaration'lardan kelganda compatible emas
- **`protected` member** ham nominal behavior keltiradi (lekin inheritance ichida ishlaydi)
- **Empty type `{}`** — har qanday non-null value'ga assign bo'ladi (faqat `null` va `undefined` rad etiladi)
- **Object literal** — fresh literal excess property check'ka uchraydi, lekin variable orqali bypass bo'ladi
- **Type alias vs interface** — assignability uchun farq yo'q, lekin declaration merging faqat interface uchun

### Follow-up savollar

1. **"Branding (nominal type emulation) qanday qilinadi?"** — `type UserId = string & { __brand: "UserId" }` — intersection bilan unique tag qo'shish, runtime'da bo'sh, compile-time'da nominal behavior beradi.
2. **"Empty interface `{}` nima ma'noni anglatadi?"** — har qanday non-null/non-undefined value'ga assign bo'ladi. `Object` type bilan deyarli ekvivalent, lekin primitive'lar ham mos keladi.

</details>

---

### Savol 2: Excess property check nima va qachon ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Excess property check — **fresh object literal** target type'da mavjud bo'lmagan property qo'shganda compile error. Faqat to'g'ridan-to'g'ri literal assignment'da ishlaydi, variable orqali bypass bo'ladi.

### To'liq tushuntirish

Structural typing'da "ko'proq property" odatda mos keladi (Dog → Animal). Lekin bu typo'larni yashiradi: `{ name: "Ali", emial: "a@b.com" }` — `emial` typo, TS struktural qoida bo'yicha rad etmaydi. Buni oldini olish uchun TS **fresh object literal** uchun qat'iyroq qoida qo'llaydi — target type'da yo'q property — error.

Fresh literal — to'g'ridan-to'g'ri yozilgan `{...}` (variable'ga bog'lanmagan). Variable'ga assign qilingach, "freshness" yo'qoladi va oddiy structural compatibility ishlaydi.

Bypass usullari: variable orqali (`const obj = {...}; const x: T = obj`), type assertion (`as T`), index signature qo'shish (`[key: string]: unknown`).

### Kod misol

```typescript
interface UserConfig {
  name: string;
  age: number;
}

// ❌ — fresh literal, excess property "email"
// const user1: UserConfig = { name: "Ali", age: 25, email: "a@b.com" };

// ✅ — variable bypass
const data = { name: "Ali", age: 25, email: "a@b.com" };
const user2: UserConfig = data;  // excess property ignore qilinadi

// ✅ — index signature
interface FlexConfig {
  name: string;
  [key: string]: unknown;
}
const user3: FlexConfig = { name: "Ali", anyExtra: 123 };  // ruxsat

// Function argument — literal — check ishlaydi
function process(cfg: UserConfig) {}
// process({ name: "Ali", age: 25, role: "admin" });  // ❌
```

### Edge Cases

- **Spread operator** — `{ ...other, extra: 1 }` natijasi fresh, lekin spread qismi widened
- **Return type** — function return'da fresh literal qaytarilsa check ishlaydi
- **Array of objects** — har element fresh literal sifatida tekshiriladi
- **Conditional/ternary** — `cond ? {...} : {...}` natijasi union, fresh saqlanadi
- **Optional property typo** — `{ nme?: string }` — nominal name mos kelmasa, target type'da ham yo'q, error

### Follow-up savollar

1. **"`as const` excess property check'ka qanday ta'sir qiladi?"** — `as const` literal type'larni mustahkamlaydi, fresh holatini saqlaydi, lekin readonly qo'shadi.
2. **"Generic function'da literal argument'ga excess check qo'llaniladimi?"** — Ha, lekin generic inference oldin ishlaydi, keyin check.

</details>

---

### Savol 3: Function parameter contravariance nima? `strictFunctionTypes` qanday ta'sir qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Function parameter type **contravariant** — agar `Dog <: Animal`, unda `(a: Animal) => void` `(d: Dog) => void` ga assignable (teskari yo'nalish). `strictFunctionTypes: true` (TS 2.6+) bu qoidani majburiy qiladi; o'chirilganda parameter bivariant bo'lib qoladi (xavfli, lekin legacy mos).

### To'liq tushuntirish

Contravariance soundness uchun zarur: agar `DogHandler` (faqat Dog ni biluvchi) `AnimalHandler` o'rnida ishlatilsa, callee Animal yuborishi mumkin — DogHandler `breed` ga qaraydi, Animal'da yo'q → crash. Aksincha, `AnimalHandler` `DogHandler` o'rnida — callee Dog yuboradi, Animal handler faqat `name` ishlatadi (Dog'da bor) → xavfsiz.

`strictFunctionTypes: true` — function type literal va function signature'lar uchun contravariance majburlanadi. **Method shorthand** (`handle(x: T): void`) bundan istisno — bivariant qoladi. Sabab: legacy DOM/Node.js API'lar bivariant method'larga tayanadi.

Soundness vs ergonomics trade-off: TS pragmatik tanlov qildi — `Array<Dog>` `Array<Animal>` ga mos (covariant, lekin mutable array'da unsound), method bivariant (legacy compat).

### Kod misol

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }

type AnimalConsumer = (a: Animal) => void;
type DogConsumer = (d: Dog) => void;

const consumeAnimal: AnimalConsumer = (a) => console.log(a.name);
const consumeDog: DogConsumer = (d) => console.log(d.breed);

// ✅ Contravariant — Animal handler Dog o'rnida xavfsiz
const handler1: DogConsumer = consumeAnimal;

// ❌ — DogConsumer faqat Dog ni biladi, Animal o'rnida xavfli
// const handler2: AnimalConsumer = consumeDog;
// strictFunctionTypes: false bo'lsa ✅ (bivariant) — lekin runtime crash

// strictFunctionTypes ta'siri:
// true:  parameter contravariant — xavfsiz
// false: parameter bivariant — xavfli (covariant + contravariant ikkalasi)
```

### Edge Cases

- **Method shorthand istisno** — `interface EventTarget { handle(event: Dog): void }` bivariant qoladi `strictFunctionTypes` da ham
- **Constructor parameter** — class constructor parameter ham bivariant (method bilan bir xil sabab)
- **Parameter count** — kamroq parametrli function ko'proq kutadigan type'ga mos (qolgan argument'lar ignore qilinadi); teskari emas. Bu count qoidasi, har bir parametr type'i esa contravariant tekshiriladi
- **Rest parameter** — `(...args: T[])` array contravariance qoidalari
- **`this` parameter** — alohida contravariance check'ka uchraydi

### Follow-up savollar

1. **"Nima uchun method shorthand bivariant qoldirilgan?"** — DOM `addEventListener` va Node `EventEmitter` ko'p yillik API'lar — strictga o'tsa ecosystem buziladi. Kompromis.
2. **"ESLint orqali bivariance'dan qanday himoyalanish?"** — `@typescript-eslint/method-signature-style: ["error", "property"]` — method shorthand'ni function property'ga aylantirish majburi.

</details>

---

### Savol 4: Bivariance va invariance nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Bivariant** — ikki yo'nalishda mos (sub→super va super→sub). Method shorthand `strictFunctionTypes` da ham bivariant. **Invariant** — hech qanday yo'nalishda mos emas (T ham input ham output position'da).

### To'liq tushuntirish

Bivariance — soundness buzilgan, lekin pragmatik. `interface H { on(e: Event): void }` da `on(e: MouseEvent): void` ga mos keladi (subtype) va `on(e: UIEvent): void` ga ham mos (supertype, agar Event <: UIEvent bo'lsa). Real ecosystem'da bu DOM event handler'lar uchun foydali, lekin tip xavfsizligi cheklovi.

Invariance — type parameter T ham covariant ham contravariant position'da bo'lganda paydo bo'ladi. Read+write data (mutable container) invariant bo'lishi shart, aks holda runtime crash bo'ladi:

```
MutableBox<T> { get(): T; set: (v: T) => void }
- get(): T          — covariant position (output)
- set: (v: T)=>void — contravariant position (input, function property)
- Birga             — INVARIANT
```

`set` **function property** bo'lishi muhim: method shorthand (`set(v: T): void`) bo'lsa parameter bivariant o'lchanadi va container invariant emas, covariant bo'lib qoladi. Agar `MutableBox<Dog>` `MutableBox<Animal>`'ga assign bo'lsa, `set(new Cat())` chaqirib, `get()`'da Dog kutiladi — runtime crash.

### Kod misol

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }
class Cat extends Animal { whiskers = 0; }

// Bivariant — method shorthand
interface EventHandler {
  handle(event: Animal): void;  // method — BIVARIANT
}

const dogOnly = { handle(d: Dog) { console.log(d.breed); } };
const h: EventHandler = dogOnly;  // ✅ bivariant ruxsat, lekin xavfli
h.handle(new Cat());  // Runtime: Cat.breed → undefined

// Contravariant — function property
interface SafeHandler {
  handle: (event: Animal) => void;  // function property — CONTRAVARIANT
}
// const safe: SafeHandler = dogOnly;  // ❌ contravariant — Dog → Animal yo'q

// Invariant — read+write. set FUNCTION PROPERTY bo'lishi shart:
// method shorthand (set(value: T)) bo'lsa parameter bivariant o'lchanadi va
// container invariant emas, covariant'ga aylanib qoladi (soundness hole)
interface MutableContainer<T> {
  get(): T;
  set: (value: T) => void;  // function property — contravariant param
}
declare const dogContainer: MutableContainer<Dog>;
// const animalContainer: MutableContainer<Animal> = dogContainer;  // ❌ INVARIANT
```

### Edge Cases

- **Bivariance backward compat** — TS 2.0 da function parameter umuman bivariant edi (strictFunctionTypes yo'q edi)
- **Constructor signature** — class constructor parameter bivariant qoladi
- **JSX prop callback** — React event handler'lar ko'pincha method shorthand orqali bivariance ga tayanadi
- **Invariance + generic constraint** — `<T extends U>` da T va U variance position'lariga qarab tekshiriladi
- **Promise<T>** — T bo'yicha covariant (faqat `then`/`await` orqali o'qiladi); reject reason T emas, shuning uchun variance'ga ta'sir qilmaydi

### Follow-up savollar

1. **"`@typescript-eslint/method-signature-style` qachon `error`, qachon `warn`?"** — yangi loyihada `error`, legacy migration'da `warn` (asta-sekin tuzatish).
2. **"`Array<T>` invariant bo'lishi kerakmidi?"** — Soundness bo'yicha ha, lekin TS pragmatik covariant qildi (ergonomics). `readonly T[]` xavfsiz covariant.

</details>

---

### Savol 5: `in`/`out` variance modifiers (TS 4.7+) nima uchun kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`in`/`out` modifiers — generic type parameter ning **kutilgan variance**'ni explicit declare qilish. `out T` — covariant, `in T` — contravariant, `in out T` — invariant. TS 4.7'da qo'shilgan. Compiler variance inference'ni skip qilib tezroq tekshiradi, niyatni hujjatlashtiradi.

### To'liq tushuntirish

Modifier qo'yilmagan generic — TS variance'ni tip ichidagi position'lardan inference qiladi (har instantiation uchun). Modifier qo'yilganda compiler bu inference'ni skip qilib, assignability'ni to'g'ridan-to'g'ri annotation bo'yicha o'lchaydi (recursive generic'da compile speed yaxshilanadi). Lekin annotation **tekshiriladi**, ko'r-ko'rona qabul qilinmaydi: compiler annotated variance'ni T ning struktural usage'idan kelib chiqadigan variance bilan solishtiradi, va ular zid bo'lsa **declaration site'ning o'zida** `TS2636` beradi.

**`out T`** — covariant deb e'lon qiladi (`Box<Dog>` `Box<Animal>`'ga subtype). T faqat output position'da bo'lishi kerak (return type, readonly property).

**`in T`** — contravariant deb e'lon qiladi (`Box<Animal>` `Box<Dog>`'ga subtype). T faqat input position'da bo'lishi kerak (function parameter).

**`in out T`** — invariant deb e'lon qiladi. T ikkala position'da bo'lganda (mutable container) to'g'ri tanlov.

T ning struktural variance'i annotation'dan torroq bo'lsa — real contravariant usage `out` ostida, yoki real covariant usage `in` ostida — error **declaration'da** chiqadi: `"Type 'X<sub-T>' is not assignable to type 'X<super-T>' as implied by variance annotation"`. Diqqat: **method shorthand parameter bivariant o'lchanadi**, shuning uchun `out T` interface'ida `consume(value: T): void` method-shorthand'i contravariant usage sifatida hisoblanmaydi va declaration error bermaydi — xato faqat parametr **function property** (`consume: (value: T) => void`) bo'lganda yuzaga keladi. Modifier'lar runtime'ga ta'sir qilmaydi — faqat type checker uchun belgi, hamda self-documenting kod.

### Kod misol

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }

// out — covariant
interface Producer<out T> {
  produce(): T;          // ✅ output position — annotation'ga mos
  // consume(value: T): void;     // method shorthand — param bivariant o'lchanadi,
  //                              // contravariant usage emas → declaration error YO'Q
  // consume: (value: T) => void; // function property — real contravariant usage,
  //                              // 'out'ga zid → declaration'da TS2636
}

// in — contravariant
interface Consumer<in T> {
  consume(value: T): void;  // ✅ input position — annotation'ga mos
}

// in out — invariant
interface Container<in out T> {
  get(): T;
  set(value: T): void;
}

declare const dogProducer: Producer<Dog>;
declare const animalConsumer: Consumer<Animal>;
declare const dogContainer: Container<Dog>;

const ap: Producer<Animal> = dogProducer;  // ✅ covariant
const dc: Consumer<Dog> = animalConsumer;  // ✅ contravariant
// const ac: Container<Animal> = dogContainer;  // ❌ invariant
```

### Edge Cases

- **Annotation real usage'ga zid kelsa** — `TS2636` "...as implied by variance annotation" **declaration site'da** chiqadi (T struktural variance'i annotation'dan torroq bo'lganda). Method-shorthand parameter bivariant o'lchanib zid usage hisoblanmaydi; function-property parameter esa real contravariant usage
- **Invariant container assignment** — `in out T` da unsound assignment plain `TS2322` beradi (struktural check buziladi), `TS2636` emas; `TS2636` faqat annotation type'ni struktura ruxsat bergandan kengroq qilganda chiqadi
- **Variance annotation faqat class/interface/type alias'da** — boshqa joyda (masalan generic function'da) yozilsa `TS1274` "'in'/'out' modifier can only appear on a type parameter of a class, interface or type alias"
- **Type parameter constraint bilan birga** — `<in T extends Comparable>` ruxsat
- **Implicit variance to'g'ri bo'lsa** — modifier qo'yish majburi emas, lekin loyiha standart sifatida foydali
- **Nested generic** — `Producer<Producer<T>>` da inner va outer variance alohida hisoblanadi
- **Default type parameter bilan** — `<out T = Animal>` ruxsat

### Follow-up savollar

1. **"`in out T` va modifier yo'q — farqi nima?"** — Modifier yo'q — TS inference qiladi, ko'pincha `in out`'ga ekvivalent. Lekin explicit aniqroq error msg va tezroq compile.
2. **"Standart lib type'larining variance'i annotation bilanmi yoki inference bilanmi aniqlanadi?"** — Inference bilan. `Array`/`ReadonlyArray`/`Promise` lib declaration'larida explicit `in`/`out` yozilmagan; compiler ularning position usage'idan variance'ni o'zi chiqaradi. TS 4.7 release'da `ReadonlyArray<out T>` faqat illustrativ misol edi.

</details>

---

### Savol 6: Array covariance va soundness hole nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Dog[]` `Animal[]` ga assignable (covariant), lekin `Array<T>` mutable — `push()` invariant bo'lishi kerak. TS pragmatik tanlovi: covariant ruxsat, lekin runtime'da Dog array'ga Animal qo'shish mumkin. Bu **soundness hole** — type system kafolat bera olmaydi. Yechim: `readonly T[]`.

### To'liq tushuntirish

Soundness — agar type checker xato bermasa, runtime ham xato bermaslik kafolati. TS bu kafolatni ataylab buzgan joylar bor (pragmatik sabablar):

1. **Array covariance** — `Dog[] → Animal[]` ruxsat. Lekin `animals.push(new Cat())` aslida Dog array'ga Cat qo'shadi.
2. **Function parameter bivariance** (method shorthand) — yuqorida.
3. **Any** — har qanday yo'nalishda mos.

Array covariance ergonomics uchun saqlanadi — ko'p kod `Animal[]` qabul qiladigan funksiyaga `Dog[]` yuboradi, agar bu ishlamasa har joyda generic kerak bo'ladi.

Xavfsiz yechim: **`readonly T[]`** covariant va sound — push/pop yo'q, faqat o'qish. `ReadonlyArray<T>` (lib type) ham aynan shu type.

### Kod misol

```typescript
class Animal { name = "Animal"; }
class Dog extends Animal { breed = "Labrador"; }
class Cat extends Animal { whiskers = 12; }

const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;   // ✅ covariant — soundness hole

animals.push(new Cat());           // Type OK, lekin dogs'ga Cat qo'shildi!
console.log(dogs[1].breed);        // Runtime: undefined (Cat.breed yo'q)

// Xavfsiz — readonly
const safeDogs: Dog[] = [new Dog()];
const safeAnimals: readonly Animal[] = safeDogs;  // ✅ covariant va sound
// safeAnimals.push(new Cat());  // ❌ push readonly'da yo'q

// Generic funksiyada
function logNames(items: readonly Animal[]) {
  items.forEach(a => console.log(a.name));
  // items.push(...) — ❌ readonly
}
logNames(dogs);  // ✅ Dog[] readonly Animal[] ga mos
```

### Edge Cases

- **`as const`** — `[1, 2, 3] as const` literal `readonly [1, 2, 3]` tuple beradi (covariant va sound)
- **Tuple covariance** — `[Dog, Dog]` `[Animal, Animal]` ga mos, lekin element soni shart
- **Map<K, V>** — V Array bilan bir xil **covariant soundness hole**: lib'da `set(key: K, value: V): this` method-shorthand bo'lgani uchun `value` parametri bivariant o'lchanadi, V esa effektiv covariant chiqadi. `Map<string, Dog>` `Map<string, Animal>`'ga **assignable** (keyin `animalMap.set("x", new Cat())` orqali dog map'ni buzish mumkin)
- **Set<T>** — T ham covariant soundness hole: `add(value: T): this` method-shorthand → bivariant param. `Set<Dog>` `Set<Animal>`'ga assignable (`animalSet.add(new Cat())` unsound)
- **Generic constraint** — `<T extends readonly Animal[]>` covariant input

### Follow-up savollar

1. **"Flow va Reason kabi tillarda array variance qanday?"** — Flow ham covariant qildi (pragmatik), OCaml/Reason mutable array invariant qildi (sound).
2. **"`as const` covariance ga qanday ta'sir qiladi?"** — `as const` array'ni `readonly tuple` ga aylantiradi, covariance saqlanadi va sound bo'ladi.

</details>

---

### Savol 7: Class va enum compatibility qoidalari? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Class** odatda structural, lekin `private`/`protected` member borligi nominal behavior keltiradi. **Numeric enum** number'ga va aksincha mos, lekin turli numeric enum'lar bir-biriga mos emas. **String enum** to'liq nominal — faqat enum member orqali assign.

### To'liq tushuntirish

Class compatibility:
- Faqat **public** member'lar bo'lsa — structural (boshqa class instance ham mos kelishi mumkin)
- `private` yoki `protected` member bo'lsa — TS member ning **origin** (qaysi class declaration'dan) ni tekshiradi. Bir xil nomli private maydon turli class'larda — incompatible
- Inheritance subtype relationship beradi: `class B extends A` → `B <: A`
- Static side va instance side alohida tekshiriladi

Enum compatibility:
- **Numeric enum**: enum member ↔ number ikkala yo'nalishda mos. Turli numeric enum'lar bir-biriga mos emas.
- **String enum**: faqat enum member orqali assign — string literal ham, boshqa string enum ham mos emas (to'liq nominal)
- **Const enum**: compile-time inline, runtime'da yo'q (`isolatedModules` bilan cheklov)

### Kod misol

```typescript
// Class — public structural
class PointA { x = 0; y = 0; }
class PointB { x = 0; y = 0; }
const pa: PointA = new PointB();  // ✅ structural

// Class — private nominal
class SecureA { private token = ""; data = 0; }
class SecureB { private token = ""; data = 0; }
// const sa: SecureA = new SecureB();  // ❌ private origin farqli

// Inheritance — subtype
class Vehicle { speed = 0; }
class Car extends Vehicle { wheels = 4; }
const v: Vehicle = new Car();  // ✅ Car <: Vehicle

// Numeric enum
enum Status { Active, Inactive }
enum Role { User, Admin }
let s: Status = Status.Active;
s = 0;                  // ✅ number → numeric enum
// s = Role.User;       // ❌ turli numeric enum

// String enum — to'liq nominal
enum Color { Red = "RED", Green = "GREEN" }
let c: Color = Color.Red;
// c = "RED";           // ❌ string literal string enum'ga mos emas
declare const raw: string;
// c = raw;             // ❌ string string enum'ga mos emas
c = raw as Color;       // assertion bilan ruxsat (runtime'da xavfli)
```

### Edge Cases

- **Abstract class** — instance'ga mos kelmaydi, faqat concrete subclass instance'lari
- **Static side compatibility** — `typeof MyClass` orqali constructor signature tekshiriladi
- **Mixin pattern** — structural type composition, runtime'da prototype zanjir
- **`declare class`** — faqat type info, runtime kod yo'q (lib type'lar uchun)
- **Const enum erasure** — `const enum` member'lari compile-time'da literal qiymatga inline qilinadi, runtime'da enum object emit qilinmaydi (kichikroq output). `isolatedModules` ostida ambient `const enum` (`declare const enum`) ishlatilsa `TS2748` chiqadi

### Follow-up savollar

1. **"Nima uchun string enum nominal qilingan?"** — Brand emulation uchun (`ID: "USER_ID"`), runtime safety. Lekin ergonomics qurboni — har joyda enum member yozish kerak.
2. **"Class va interface bir xil shape — assignable bo'ladimi?"** — Ha, agar class public-only va structural mos kelsa. Interface implementation explicit emas.

</details>

---

### Savol 8: `void` return type maxsus xatti-harakati nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`void` return type **maxsus** — har qanday return type qaytaruvchi funksiya `void` qaytaruvchi tip'ga mos keladi. Callee return value'ni ignore qiladi. Bu callback API'lar uchun ergonomics (`forEach`, `addEventListener`).

### To'liq tushuntirish

Oddiy covariance bo'yicha `() => number` `() => undefined` ga mos kelmasligi kerak (`number` `undefined`'ga subtype emas). Lekin TS `void` return uchun maxsus qoida: **return value e'tibordan chetda**. Sabab — callback ergonomics:

```typescript
[1, 2, 3].forEach(x => x * 2);  // map callback void kutadi, lekin map natijani ishlatmaydi
```

Bu istisno faqat **return position**'da `void` bo'lganda. Variable type'ida `void` o'zgaruvchi ko'p qiymat olishi mumkin emas (undefined faqat).

Muhim: callee return value'ni `void` deb biladi va ishlatmaydi — agar callee yashirin return ishlatsa, soundness buziladi.

### Kod misol

```typescript
type VoidCallback = () => void;

// ✅ — return value ignore qilinadi
const numFn: VoidCallback = () => 42;
const result = numFn();
// result: void — 42 ni ishlatib bo'lmaydi

// forEach holatida foydali
const items = [1, 2, 3];
items.forEach(x => x);  // callback `void` kutadi, lekin `x` return — OK

// addEventListener
declare const button: HTMLButtonElement;
button.addEventListener("click", (e) => {
  return false;  // ✅ — handler void kutadi, false ignore qilinadi
});

// Maxsus istisno faqat return position'da
type VoidArg = (v: void) => void;
// const fn: VoidArg = (v: number) => {};  // ❌ — parameter position'da farq
```

### Edge Cases

- **`Promise<void>` void special'dan tashqari** — void return istisnosi `Promise<>` ichiga kirmaydi: `() => Promise<number>` `() => Promise<void>`'ga **assignable emas** (TS#49755). Bu floating-promise xatolarini ushlash uchun ataylab
- **Generator function** — `Generator<T, void, U>` — return type void, lekin yield T
- **Method override** — parent void return, child non-void return — TS ruxsat (return widening)
- **Type predicate** — `(x): x is T` return type void emas, narrowing predicate
- **`undefined` vs `void` parameter** — parameter `void` deyarli yo'q (faqat dummy), `undefined` aniq

### Follow-up savollar

1. **"Nima uchun callback'da return ignore qilish xavfli emas?"** — Callee API contract'da return ishlatmasligi belgilangan. Bu TS'ning maxsus kelishuvi.
2. **"`Promise<void>` async function uchun nima ma'noni anglatadi?"** — Function resolve qiladi lekin meaningful value qaytarmaydi. `await` natijasi `undefined`.

</details>

---

## Output savollar

### Savol 9: Quyidagi assignment'lardan qaysilari compile bo'ladi? [Middle]

```typescript
interface User { id: number; name: string; }
interface Employee { id: number; name: string; department: string; }

declare const employee: Employee;
declare const userVar: User;

const u1: User = employee;                                       // A
const u2: User = { id: 1, name: "Ali", role: "admin" };          // B
const data = { id: 1, name: "Ali", role: "admin" };
const u3: User = data;                                            // C
const e1: Employee = userVar;                                     // D
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A: ✅, B: ❌, C: ✅, D: ❌

### To'liq tushuntirish

- **A: ✅** — `employee`da `id` va `name` bor, `User` shape'ni qondiradi. Qo'shimcha `department` structural mosligida muammo emas (variable orqali — excess check yo'q).
- **B: ❌** — fresh object literal, `User` da `role` yo'q — excess property check error.
- **C: ✅** — variable `data` orqali assignment, fresh emas — excess check bypass.
- **D: ❌** — `User` da `department` yo'q, `Employee` shape qondirilmaydi.

### Edge Cases

Fresh literal qoida nuancelari: spread operator (`{ ...userVar, extra: 1 }`) natijasi fresh, lekin spread qismi widened. Function argument'da literal bo'lsa — excess check ishlaydi.

</details>

---

### Savol 10: Variance — quyidagi assignment'lar qaysi compile bo'ladi? [Middle+]

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }

type Reader<T> = () => T;
type Writer<T> = (v: T) => void;

declare const readDog: Reader<Dog>;
declare const readAnimal: Reader<Animal>;
declare const writeDog: Writer<Dog>;
declare const writeAnimal: Writer<Animal>;

const a: Reader<Animal> = readDog;   // A
const b: Reader<Dog> = readAnimal;   // B
const c: Writer<Dog> = writeAnimal;  // C
const d: Writer<Animal> = writeDog;  // D
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A: ✅ (covariant return), B: ❌, C: ✅ (contravariant param), D: ❌

### To'liq tushuntirish

Return type **covariant**: `Reader<Dog>` `Reader<Animal>` ga mos (Dog Animal'ga subtype, va Animal kutilgan joyda Dog qaytarish xavfsiz). Teskari xavfli — Animal qaytaruvchi Dog kutilgan joyda → `breed` yo'q.

Parameter **contravariant** (`strictFunctionTypes: true`): `Writer<Animal>` `Writer<Dog>` ga mos (Animal handler Dog ni ham handle qiladi, faqat `name` ishlatadi). Teskari xavfli — Dog handler Animal kutilgan joyda → `breed`'ga qaraydi, oddiy Animal'da yo'q.

</details>

---

### Savol 11: Method shorthand vs function property — output? [Middle+]

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }
class Cat extends Animal { whiskers = 0; }

interface HandlerShorthand {
  handle(value: Animal): void;
}
interface HandlerProperty {
  handle: (value: Animal) => void;
}

const dogOnly = { handle(d: Dog) { console.log(d.breed); } };

const h1: HandlerShorthand = dogOnly;  // A
const h2: HandlerProperty = dogOnly;   // B
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A: ✅ (bivariant), B: ❌ (contravariant)

### To'liq tushuntirish

`HandlerShorthand.handle` — method shorthand syntax — `strictFunctionTypes: true` bo'lsa ham **bivariant** qoladi. Dog handler Animal handler o'rnida ruxsat, lekin xavfli: `h1.handle(new Cat())` chaqirilsa runtime'da `Cat.breed` → `undefined`.

`HandlerProperty.handle` — function property — **contravariant**. Dog handler Animal handler o'rnida rad etiladi. Bu xavfsiz: type check soundness ta'minlaydi.

### Edge Cases

ESLint `@typescript-eslint/method-signature-style: ["error", "property"]` qoidasi method shorthand'ni avtomatik function property'ga aylantirishni majburlaydi — bivariance hole'dan himoyalanish.

</details>

---

## Amaliy savollar (Coding)

### Savol 12: `Producer<T>`, `Consumer<T>`, `Processor<T>` — variance modifiers bilan yozing [Senior]

**Savol:** `in`/`out` modifier bilan uch interface yarating va har biri uchun assignability misollarini ko'rsating.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Producer<out T>` — covariant. `Consumer<in T>` — contravariant. `Processor<in out T>` — invariant.

### To'liq tushuntirish

Har modifier kutilgan variance'ni e'lon qiladi. `out T` — covariant (T output position'da: return type), `in T` — contravariant (T input position'da: parameter), `in out T` — invariant (ikkala position'da). Compiler annotation'ni T ning struktural usage'i bilan solishtiradi: zid bo'lsa (real contravariant usage `out` ostida yoki real covariant usage `in` ostida) `TS2636` declaration site'da chiqadi.

### Kod misol

```typescript
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }

interface Producer<out T> {
  produce(): T;          // output position — 'out'ga mos (covariant)
}

interface Consumer<in T> {
  consume(value: T): void;  // input position — 'in'ga mos (contravariant)
}

interface Processor<in out T> {
  process(value: T): T;  // ikki position'da — 'in out' (invariant)
}

declare const dogProducer: Producer<Dog>;
declare const animalConsumer: Consumer<Animal>;
declare const dogProcessor: Processor<Dog>;
declare const animalProcessor: Processor<Animal>;

// Covariant — Dog producer Animal o'rnida xavfsiz (faqat o'qish)
const ap: Producer<Animal> = dogProducer;       // ✅

// Contravariant — Animal consumer Dog o'rnida xavfsiz (kengroq qabul qiladi)
const dc: Consumer<Dog> = animalConsumer;       // ✅

// Invariant — hech qanday yo'nalishda mos emas
// const aProc: Processor<Animal> = dogProcessor;     // ❌
// const dProc: Processor<Dog> = animalProcessor;     // ❌
```

### Edge Cases

| Type | Variance | Dog → Animal | Animal → Dog |
|------|----------|:------------:|:------------:|
| `Producer<out T>` | Covariant | ✅ | ❌ |
| `Consumer<in T>` | Contravariant | ❌ | ✅ |
| `Processor<in out T>` | Invariant | ❌ | ❌ |

<details>
<summary><strong>Deep Dive</strong></summary>

Modifier'siz TS har generic instantiation'da position analysis qiladi (T qaerda — covariant/contravariant/invariant?). Recursive type'lar va katta loyihalarda bu compile time'ni oshiradi. Modifier qo'yilganda compiler assignability'ni to'g'ridan-to'g'ri annotation bo'yicha o'lchaydi va bu analysis'ni qisqartiradi. Lekin annotation tekshiriladi: agar T ning struktural usage'i annotation'dan torroq variance talab qilsa (masalan `out T` ostida function-property parameter — real contravariant usage), compiler `TS2636`'ni **declaration site'da** beradi. Method-shorthand parameter bivariant o'lchangani uchun zid usage hisoblanmaydi.

Standart lib type'larining variance'i compiler tomonidan inference qilinadi — TS 4.7 annotation'i `Array`/`ReadonlyArray`/`Promise`'ga explicit yozilmagan. `ReadonlyArray<T>` covariant (T faqat o'qiladi, shuning uchun 4.7 release'da `<out T>` illustrativ misol sifatida keltirildi). `Promise<T>` covariant (T `then` natijasida o'qiladi). `Array<T>` esa position bo'yicha invariant bo'lishi kerak edi (T `push`'da input, indexing'da output), lekin TS ataylab uni covariant deb belgilab, mashhur soundness hole'ni qoldirgan (Savol 6'dagi `Dog[]` → `Animal[]`).

`out` va `in` keyword'lari type parameter nomidan oldin yoziladi, `extends`'dan ham oldin: `<out T extends Animal>`. `default` bilan birga ham: `<out T = Animal>`.

</details>

</details>

---

### Savol 13: Function compatibility — quyidagi function'lar mos keladimi? [Middle+]

**Savol:** `strictFunctionTypes: true` da har assignment compile bo'ladimi?

```typescript
type F1 = (a: number) => string;
type F2 = (a: number, b: string) => string;
type F3 = () => string;
type F4 = (a: number) => "hello";
type F5 = (a: number | string) => string;
type F6 = (a: number) => void;

declare const f1: F1; declare const f2: F2; declare const f3: F3;
declare const f4: F4; declare const f5: F5; declare const f6: F6;

const t1: F2 = f1;  // A
const t2: F1 = f3;  // B
const t3: F1 = f4;  // C
const t4: F1 = f5;  // D
const t5: F6 = f1;  // E
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A: ✅, B: ✅, C: ✅, D: ✅, E: ✅

### To'liq tushuntirish

- **A** (F1 → F2): kamroq parametr ko'proqga mos. Callee 2 arg yuboradi, callee 1 arg ishlatadi — qolgani ignore. Xavfsiz.
- **B** (F3 → F1): 0 param 1 param'ga mos. Sabab bir xil.
- **C** (F4 → F1): return type `"hello"` `string` ga subtype (literal). Covariant — ✅.
- **D** (F5 → F1): parameter contravariant. `number | string` `number` ga supertype (kengroq). Callee `number` yuboradi, F5 callback `number | string` ni boshqaradi — `number` mos. ✅.
- **E** (F1 → F6): return type `string` `void` ga mos (void special — return ignore). ✅.

### Edge Cases

- `(a: number) => "hello"` → `(a: number) => string` ✅ literal subtype
- `(a: number) => string` → `(a: number) => "hello"` ❌ — return'da literal kutiladi
- `(a: number | string) => string` → `(a: number) => string` ✅ — contravariant param
- `(a: number) => string` → `(a: number | string) => string` ❌ — toraytirilgan param

</details>

---

### Savol 14: Class compatibility — private member [Middle]

**Savol:** Qaysi assignment'lar compile bo'ladi?

```typescript
class PublicA { x = 0; }
class PublicB { x = 0; }
class PrivateA { private token = ""; data = 0; }
class PrivateB { private token = ""; data = 0; }
class Child extends PrivateA { extra = 0; }

const a1: PublicA = new PublicB();    // A
const a2: PrivateA = new PrivateB();  // B
const a3: PrivateA = new Child();     // C
const a4: PublicA = new PrivateA();   // D (faqat x check)
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A: ✅, B: ❌, C: ✅, D: ❌

### To'liq tushuntirish

- **A**: public-only class'lar structural — `x: number` ikkalasida — mos.
- **B**: `private token` har class declaration'da unique nominal identifier. `PrivateA` va `PrivateB` member'lari turli origin'lardan — incompatible.
- **C**: Inheritance — `Child extends PrivateA` → `Child <: PrivateA` (private member origin saqlanadi).
- **D**: `PublicA` `{ x }` shape kutadi, `PrivateA` da `x` yo'q (`token`, `data` bor) — shape mos kelmaydi.

</details>

---

## Bug fix savollar

### Savol 15: Quyidagi kodda type error qayerda? Tuzating [Middle+]

**Savol:**

```typescript
class Vehicle { speed = 0; }
class Car extends Vehicle { wheels = 4; }
class Truck extends Vehicle { capacity = 1000; }

interface EventBus {
  emit(vehicle: Vehicle): void;
  on(handler: (v: Vehicle) => void): void;
}

const bus: EventBus = {
  emit(c: Car) { console.log(c.wheels); },
  on(handler: (t: Truck) => void) { handler({ speed: 0, capacity: 100 } as Truck); }
};

bus.emit(new Truck());  // compile OK, lekin runtime'da c.wheels → undefined
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Hech qaerda compile error **yo'q** — aynan shu muammo. `EventBus.emit` va `on` ikkalasi ham **method shorthand** (`emit(...)`, `on(...)`), shuning uchun parameter'lari **bivariant** tekshiriladi (`strictFunctionTypes`'da ham). Xavfli `emit(c: Car)` va `on(handler: (t: Truck) => void)` bivariance hole tufayli o'tib ketadi. Tuzatish: ikkala member'ni **function property** syntax'iga aylantirish — shunda parameter contravariance majburlanadi va xavfli implementation'lar compile'da rad etiladi.

### To'liq tushuntirish

`emit(c: Car)` method shorthand bivariance tufayli ruxsat oladi — type checker xato bermaydi. Lekin `bus.emit(new Truck())` runtime'da `c.wheels`'ni o'qiydi, Truck'da `wheels` yo'q → `undefined`.

`on(handler: (t: Truck) => void)` ham aynan shu hole'dan o'tadi: `on` method shorthand bo'lgani uchun uning `handler` parameter'i bivariant tekshiriladi, shuning uchun `(t: Truck) => void` `(v: Vehicle) => void` o'rnida qabul qilinadi. Lekin implementation `handler`'ni `{ ... } as Truck` bilan chaqiradi — `as` assertion soundness'ni majburan buzadi. Real callee oddiy `Vehicle` (masalan `Car`) yuborsa, `handler` `capacity`'ga qaraydi → `undefined`.

Demak xato — "qayerda compile error?" emas, balki "nega compile error YO'Q?": method shorthand bivariance hole ikkala unsound assignment'ni yashiradi. Yechim quyida.

### Kod misol

```typescript
// Tuzatilgan versiya
interface EventBus {
  // Function property — contravariance majburi
  emit: (vehicle: Vehicle) => void;
  on: (handler: (v: Vehicle) => void) => void;
}

const bus: EventBus = {
  // ✅ — Vehicle handler har qanday subclass'ni (Car/Truck) ham qabul qiladi
  emit(vehicle: Vehicle) { console.log(vehicle.speed); },
  on(handler) { handler(new Car()); }
};

bus.emit(new Car());    // ✅
bus.emit(new Truck());  // ✅
```

### Edge Cases

Method shorthand'dan function property'ga o'tish — bivariance hole'dan himoyalanish standart. ESLint `@typescript-eslint/method-signature-style` qoidasi shu maqsadda.

</details>

---

### Savol 16: Array covariance crash — sababini toping [Middle+]

**Savol:**

```typescript
class Bird { fly(): string { return "flying"; } }
class Penguin extends Bird { fly(): string { throw new Error("Penguins can't fly!"); } }

function letBirdsFly(birds: Bird[]) {
  birds.forEach(b => b.fly());
}

const penguins: Penguin[] = [new Penguin()];
letBirdsFly(penguins);  // ?
```

> `fly()`'da explicit `: string` return type yozilgan. Aks holda faqat `throw` qiluvchi method body return type'ni `void` deb inference qiladi, bu base'ning `() => string`'iga mos kelmaydi (`TS2416`) va kod umuman compile bo'lmaydi — demak bu yerda compile error variance'dan emas, override return type'idan kelib chiqardi.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Compile OK (array covariant — `Penguin[]` `Bird[]` ga mos), lekin runtime'da `Error: Penguins can't fly!`. Liskov Substitution Principle buzilgan: subclass parent contract'ni buzdi.

### To'liq tushuntirish

TS `Penguin[]` `Bird[]` ga assignable deb hisoblaydi (covariant). Lekin `Penguin.fly()` parent `Bird.fly()` contract'ni buzadi — exception throw qiladi. Bu type system buzilishi emas (Penguin extends Bird), Liskov printsipi buzilishi.

Yechim — domain modelni qayta ko'rib chiqish. Yoki `readonly Bird[]` ishlatish (lekin bu LSP'ni hal qilmaydi, faqat array mutation'ni cheklaydi).

### Kod misol

```typescript
// Yaxshiroq model
abstract class Bird {
  abstract canFly: boolean;
  fly(): string {
    if (!this.canFly) throw new Error("This bird can't fly");
    return "flying";
  }
}

class Sparrow extends Bird { canFly = true; }
class Penguin extends Bird { canFly = false; }

// Yoki interface bilan
interface FlyingBird { fly(): string; }
interface SwimmingBird { swim(): string; }

class Sparrow2 implements FlyingBird { fly() { return "flying"; } }
class Penguin2 implements SwimmingBird { swim() { return "swimming"; } }
```

</details>

---

## Xulosa

- **Structural typing** — TS type identity shape bo'yicha (member nomi va type'i), nominal emas. `private`/`protected` member nominal behavior beradi
- **Excess property check** — fresh object literal'ga maxsus qoida (target type'da yo'q property error). Variable orqali bypass
- **Variance qoidalari**: return type covariant, function parameter contravariant (`strictFunctionTypes: true`), method shorthand bivariant (legacy compat), mutable container invariant
- **Bivariance hole** — method shorthand syntax `(x: Sub) => void` ↔ `(x: Super) => void` ikkala yo'nalishda mos. ESLint `@typescript-eslint/method-signature-style` himoya
- **`in`/`out` modifiers (TS 4.7+)** — generic variance'ni explicit declaration. Compiler annotation'ni T struktural usage'i bilan tekshiradi; zid bo'lsa (real contravariant usage `out` ostida) `TS2636` declaration site'da chiqadi. Compile speed va niyatni hujjatlash
- **Array covariance soundness hole** — `Dog[]` `Animal[]` ga mos, `push` invariant kerak — TS pragmatik tanlov. `readonly T[]` xavfsiz
- **`void` return special** — har qanday return value ignore qilinadi callback ergonomics uchun (`forEach`, `addEventListener`)
- **Class compatibility** — public structural, private/protected origin-based nominal. Numeric enum number'ga mos, string enum to'liq nominal
- Liskov Substitution: type system covariant ruxsat bersa ham subclass parent contract'ni buzsa runtime crash — domain model qayta ko'rib chiqish

