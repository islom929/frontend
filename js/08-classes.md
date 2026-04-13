# Bo'lim 8: ES6 Classes

> ES6 `class` sintaksisi — prototypal inheritance ustiga qurilgan syntactic sugar. Ichida constructor function + prototype mexanizmi ishlaydi, lekin yozish va o'qish ancha qulay. Bu bo'limda class anatomy, inheritance (`extends`/`super`), private fields, static members, accessor'lar, mixins va composition vs inheritance chuqur yoritiladi.

---

## Mundarija

- [Class — Syntactic Sugar Over Prototypes](#class--syntactic-sugar-over-prototypes)
- [Class Anatomy](#class-anatomy)
- [Class Declarations vs Expressions](#class-declarations-vs-expressions)
- [Class Hoisting — TDZ](#class-hoisting--tdz)
- [Inheritance: extends va super](#inheritance-extends-va-super)
- [Private Fields va Methods](#private-fields-va-methods)
- [Public Class Fields](#public-class-fields)
- [Static Properties va Methods](#static-properties-va-methods)
- [Static Initialization Blocks](#static-initialization-blocks)
- [Accessor — Getter va Setter](#accessor--getter-va-setter)
- [instanceof Bilan Classes](#instanceof-bilan-classes)
- [Classes vs Prototypes — Ichki Farq](#classes-vs-prototypes--ichki-farq)
- [Mixins — Ko'p Merosxo'rlik Muammosi](#mixins--kop-merosxorlik-muammosi)
- [Composition vs Inheritance](#composition-vs-inheritance)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Class — Syntactic Sugar Over Prototypes

### Nazariya

ES6 da kiritilgan `class` sintaksisi — bu [07-prototypes.md](07-prototypes.md) da o'rgangan constructor function + prototype pattern'ining **yangi yozuv usuli**. Engine ichida class bilan yaratilgan kod aynan shu prototype mexanizmiga aylanadi. Ya'ni `class` yangi meros tizimi emas — u mavjud tizimga qulay sintaksis beradi.

Lekin `class` faqat "go'zal yozuv" emas — u bir nechta muhim farqlarga ega:
1. `class` ichidagi barcha method'lar **non-enumerable** (`enumerable: false`) — constructor function'da esa default `true`
2. `class` ichidagi barcha kod avtomatik **strict mode** da ishlaydi
3. `class` ni `new` siz chaqirib bo'lmaydi — `TypeError` beradi
4. `class` **function declaration kabi to'liq hoist emas** — u `let`/`const` kabi TDZ da turadi (binding scope boshida yaratiladi, lekin initialization evaluation vaqtiga kechiktiriladi). Function declaration esa to'liq hoist bo'ladi — e'londan oldin chaqirish mumkin.
5. `class` method'larida `[[Construct]]` internal slot yo'q — ya'ni method'larni `new` bilan chaqirib bo'lmaydi

<details>
<summary><strong>Under the Hood</strong></summary>

`class` yozganingizda engine ichida nima sodir bo'ladi:

```javascript
// class sintaksisi:
class User {
  constructor(name) {
    this.name = name;
  }
  greet() {
    return `Salom, ${this.name}`;
  }
}

// Engine ichida bu quyidagiga teng:
// 1. User funksiyasi yaratiladi (constructor)
// 2. User.prototype.greet qo'shiladi (enumerable: false!)
// 3. Strict mode yoqiladi
// 4. new siz chaqirish taqiqlanadi
```

```
class User { constructor(name) { ... }  greet() { ... } }
                    ↓ engine ichida ↓

┌─ User (function) ────────────────┐
│  [[IsClassConstructor]]: true    │ ← new siz chaqirish taqiq
│  prototype: ─────────────────────│──┐
└──────────────────────────────────┘  │
                                      ▼
                          ┌─ User.prototype ──────┐
                          │ greet() {enumerable:   │
                          │          false}        │
                          │ constructor: User      │
                          └────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Class va constructor function'ning teng ekanligini ko'rsatish:

```javascript
class User {
  constructor(name) { this.name = name; }
  greet() { return `Salom, ${this.name}`; }
}

// typeof — function:
console.log(typeof User); // "function"

// prototype chain bir xil:
const user = new User("Ali");
console.log(Object.getPrototypeOf(user) === User.prototype); // true

// Method prototype'da:
console.log(user.hasOwnProperty("greet")); // false
console.log(User.prototype.hasOwnProperty("greet")); // true

// LEKIN — muhim farq:
// Class method enumerable: false
console.log(Object.getOwnPropertyDescriptor(User.prototype, "greet").enumerable); // false

// Constructor function method enumerable: true (default)
function UserFn(name) { this.name = name; }
UserFn.prototype.greet = function() { return this.name; };
console.log(Object.getOwnPropertyDescriptor(UserFn.prototype, "greet").enumerable); // true
```

</details>

---

## Class Anatomy

### Nazariya

Class'ning tarkibiy qismlari:

1. **`constructor`** — maxsus method. `new` bilan class chaqirilganda avtomatik ishlaydi. Har bir class'da faqat **bitta** constructor bo'ladi. Yozilmasa — bo'sh default constructor qo'shiladi.

2. **Instance methods** — prototype'ga qo'shiladigan oddiy method'lar. Barcha instance'lar ularni share qiladi.

3. **Static methods** — `static` keyword bilan. Class'ning o'zida saqlanadi, instance'larda yo'q. Utility/factory funksiyalar uchun.

4. **Public class fields** — constructor'siz instance property e'lon qilish (ES2022).

5. **Private fields/methods** — `#` prefix bilan. Faqat class ichidan accessible (ES2022).

6. **Getter/Setter** — `get`/`set` keyword bilan computed property.

<details>
<summary><strong>Under the Hood</strong></summary>

Class body parse qilinganida engine **ClassDefinitionEvaluation** algorithm'ini ishlatadi. Bu jarayon quyidagi tartibda ishlaydi:

1. Yangi constructor function object yaratiladi — `[[IsClassConstructor]]: true` internal slot bilan
2. `ClassBody` dagi har bir `MethodDefinition` uchun **DefineMethod** / **PropertyDefinitionEvaluation** chaqiriladi
3. Instance method'lar `constructor.prototype` ga `Object.defineProperty` orqali qo'shiladi — `enumerable: false`, `writable: true`, `configurable: true`
4. Static method'lar constructor function'ning o'ziga own property sifatida qo'shiladi
5. Class field'lar (public va private) alohida **ClassFieldDefinition** record'lari sifatida saqlanadi — ular `new` chaqirilganida constructor oxirida evaluate bo'ladi

V8 da class yaratilganida engine `JSFunction` object yaratadi va unga `SharedFunctionInfo` bog'laydi. Prototype object `JSObject` sifatida allocate qilinadi va **Hidden Class** orqali shape tracking ishlaydi. (V8 source kodida Hidden Class `Map` deb nomlanadi — lekin bu JavaScript'dagi `Map` data structure'dan butunlay boshqa tushuncha.) Constructor'dagi `this.prop = value` assignment'lar V8 ning inline cache (IC) mexanizmi orqali optimallashtiriladi — har bir property qo'shilganida yangi Hidden Class transition yaratiladi.

Private field va method'lar uchun alohida `ClassScope` yaratiladi — bu scope tashqi koddan accessible emas. `#field` lar `PrivateName` record sifatida class evaluation vaqtida ro'yxatga olinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq class anatomy:

```javascript
class Product {
  // Public class field (har instance'da yaratiladi):
  currency = "UZS";

  // Private field:
  #discount = 0;

  // Constructor:
  constructor(name, price) {
    this.name = name;
    this.price = price;
  }

  // Instance method (prototype'da):
  getInfo() {
    return `${this.name}: ${this.finalPrice} ${this.currency}`;
  }

  // Getter (computed property):
  get finalPrice() {
    return this.price - this.price * this.#discount;
  }

  // Setter:
  set discountPercent(percent) {
    if (percent < 0 || percent > 100) throw new RangeError("0-100 orasida bo'lsin");
    this.#discount = percent / 100;
  }

  // Private method:
  #validate() {
    return this.price > 0 && this.name.length > 0;
  }

  // Static method:
  static compare(a, b) {
    return a.price - b.price;
  }

  // Static field:
  static category = "General";
}

const phone = new Product("iPhone", 15_000_000);
phone.discountPercent = 10;
console.log(phone.getInfo()); // "iPhone: 13500000 UZS"
console.log(Product.category); // "General"

// Static method:
const laptop = new Product("MacBook", 25_000_000);
const sorted = [laptop, phone].sort(Product.compare);
```

</details>

---

## Class Declarations vs Expressions

### Nazariya

Class'ni ikki usulda yaratish mumkin — declaration va expression. Farqi function declaration va function expression bilan bir xil pattern'ga amal qiladi, lekin **hoisting** jihatidan ikkalasi ham **TDZ** da turadi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da class declaration va class expression ikkalasi ham **ClassDefinitionEvaluation** orqali evaluate bo'ladi. Farq faqat binding mexanizmida:

- **Class Declaration** — `BindingIdentifier` joriy scope ning **lexical environment** ga `let`-semantika bilan qo'shiladi. Ya'ni hoisting bo'ladi (identifier scope boshida ro'yxatga olinadi), lekin initialization evaluation vaqtigacha kechiktiriladi — bu **TDZ (Temporal Dead Zone)** hosil qiladi.
- **Class Expression** — `BindingIdentifier` (agar berilgan bo'lsa) faqat class body ichidagi alohida **class scope** da mavjud bo'ladi. Tashqi scope ga qo'shilmaydi. Named class expression da ichki nom recursive reference uchun foydali — `const` semantikasi bilan bind qilinadi (o'zgartirib bo'lmaydi).

V8 parser class declaration ni uchratganida `VariableProxy` yaratadi va uni `MUST_USE_EXECUTION_CONTEXT` bilan belgilaydi. Class expression da esa anonymous class uchun V8 `SharedFunctionInfo` ning `inferred_name` field'iga o'zgaruvchi nomini yozadi — shuning uchun `const Foo = class {}` da `Foo.name === "Foo"` bo'ladi. Bu **name inference** mexanizmi `AssignmentExpression` evaluation vaqtida ishlaydi.

Function expression dan farqli o'laroq, class expression ham strict mode da evaluate bo'ladi va `[[IsClassConstructor]]: true` slot oladi — bu `new` siz chaqirishni taqiqlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Class Declaration:**

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} speaks`; }
}

const a = new Animal("Cat");
```

**Class Expression — named:**

```javascript
const Animal = class AnimalClass {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} speaks`; }
};

console.log(Animal.name);      // "AnimalClass" — ichki nom
// AnimalClass faqat class ichida accessible:
// console.log(AnimalClass);   // ❌ ReferenceError
```

**Class Expression — unnamed:**

```javascript
const Animal = class {
  constructor(name) { this.name = name; }
};

console.log(Animal.name); // "Animal" — o'zgaruvchi nomidan oladi
```

**Factory pattern bilan:**

```javascript
function createClass(baseGreeting) {
  return class {
    constructor(name) { this.name = name; }
    greet() { return `${baseGreeting}, ${this.name}`; }
  };
}

const UzbekGreeter = createClass("Assalomu alaykum");
const EnglishGreeter = createClass("Hello");

new UzbekGreeter("Ali").greet();    // "Assalomu alaykum, Ali"
new EnglishGreeter("John").greet(); // "Hello, John"
```

</details>

---

## Class Hoisting — TDZ

### Nazariya

Class declaration va class expression ikkalasi ham **hoisting** qilinadi, lekin **Temporal Dead Zone (TDZ)** da turadi — `let` va `const` bilan bir xil. Bu `function` declaration dan muhim farq: function declaration to'liq hoist bo'ladi (e'lon qilishdan oldin chaqirish mumkin), class esa e'londan oldin ishlatib bo'lmaydi.

Bu farq nima uchun bor? Class'da `extends` bilan inheritance bo'lishi mumkin — agar class hoist bo'lsa va parent class hali yaratilmagan bo'lsa, noto'g'ri prototype chain quriladi. TDZ bu muammoning oldini oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha class declaration `let` bilan bir xil **lexical binding** mexanizmidan foydalanadi. Compilation phase da engine quyidagilarni bajaradi:

1. **CreateMutableBinding** — identifier scope ga qo'shiladi, lekin **uninitialized** holatda
2. Shu nuqtadan to `class` declaration evaluate bo'lguncha — TDZ faol. Bu oraliqda identifier ga har qanday murojaat `ReferenceError` beradi
3. **ClassDefinitionEvaluation** tugagandan so'ng — **InitializeBinding** chaqiriladi va class function reference bind bo'ladi

V8 da TDZ implement qilish uchun `TheHole` degan maxsus sentinel value ishlatiladi. Binding yaratilganda uning qiymati `TheHole` ga set bo'ladi. Har bir variable access da V8 `TheHole` check qiladi — agar topsa `ReferenceError` throw qiladi. Bu check production code da JIT compiler tomonidan optimize qilinishi mumkin — agar V8 statik analiz orqali isbotlay olsa ki binding doim initialized bo'ladi, TDZ check ni olib tashlaydi.

`typeof` operatori ham TDZ da `ReferenceError` beradi — bu e'lon qilinMAGAN variable dan farq (`typeof undeclared === "undefined"`). Sababi: spec bo'yicha `typeof` avval **GetValue** chaqiradi, GetValue esa uninitialized binding uchun xato beradi. E'lon qilinmagan identifier uchun esa `typeof` maxsus yo'l bilan unresolvable reference ni handle qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Class — TDZ da, e'londan oldin ishlatib bo'lmaydi:
const user = new User("Ali"); // ❌ ReferenceError: Cannot access 'User' before initialization

class User {
  constructor(name) { this.name = name; }
}

// ✅ Function declaration — to'liq hoist:
const user2 = new UserFn("Ali"); // ✅ ishlaydi

function UserFn(name) { this.name = name; }
```

```javascript
// typeof ham TDZ da xato beradi:
console.log(typeof User); // ❌ ReferenceError (TDZ)

class User {}

// Function declaration da esa typeof ishlaydi:
console.log(typeof createUser); // "function" — to'liq hoist bo'lgan
function createUser() {}
console.log(typeof createUser); // "function"

// E'lon qilinMAGAN o'zgaruvchi uchun typeof xato bermaydi:
console.log(typeof Nonexistent); // "undefined" — xato emas
// Farq: e'lon qilinmagan → "undefined", TDZ da → ReferenceError
```

</details>

---

## Inheritance: extends va super

### Nazariya

`extends` keyword — child class parent class'dan meros olishini bildiradi. Ichida engine quyidagilarni bajaradi:
1. `Child.prototype.[[Prototype]]` = `Parent.prototype` (method meros)
2. `Child.[[Prototype]]` = `Parent` (static method meros)

`super` keyword ikki kontekstda ishlatiladi:
- **`super()`** — constructor ichida. Parent constructor'ni chaqiradi. `extends` ishlatilgan class'da `this` ga murojaat qilishdan **OLDIN** `super()` chaqirilishi **SHART**.
- **`super.method()`** — method ichida. Parent'ning prototype method'ini chaqiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`extends` engine ichida nima qiladi:

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} speaks`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);    // Parent constructor'ni chaqiradi (this shu yerda yaratiladi — Animal.call(this) dan farqli mexanizm)
    this.breed = breed;
  }
  bark() { return `${this.name} barks`; }
}
```

```
Engine ichida:

// 1. Prototype chain (instance method meros):
Object.setPrototypeOf(Dog.prototype, Animal.prototype);
// Dog.prototype.__proto__ === Animal.prototype

// 2. Constructor chain (static method meros):
Object.setPrototypeOf(Dog, Animal);
// Dog.__proto__ === Animal

Natija chain:
dog (instance)
  └── [[Prototype]] → Dog.prototype { bark() }
                          └── [[Prototype]] → Animal.prototype { speak() }
                                                  └── [[Prototype]] → Object.prototype
                                                                          └── null

Dog (constructor)
  └── [[Prototype]] → Animal (constructor)
                          └── [[Prototype]] → Function.prototype
```

</details>

### super() — Constructor Ichida

`extends` ishlatilgan (derived) class'da `this` ishlatishdan oldin `super()` chaqirish **majburiy**. Sababi: derived class'da `this` object'i parent constructor tomonidan yaratiladi — `super()` chaqirilmaguncha `this` mavjud emas.

```javascript
class Animal {
  constructor(name) {
    this.name = name;
    this.alive = true;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    // console.log(this); // ❌ ReferenceError: Must call super before accessing 'this'
    super(name);          // ✅ avval super — this yaratiladi
    this.breed = breed;   // endi this accessible
  }
}

const rex = new Dog("Rex", "Labrador");
console.log(rex); // { name: "Rex", alive: true, breed: "Labrador" }
```

Agar derived class'da constructor yozilmasa — default constructor avtomatik qo'shiladi:

```javascript
class Dog extends Animal {
  // Constructor yozilmasa, shu qo'shiladi:
  // constructor(...args) { super(...args); }
}
```

### super.method() — Method Ichida

`super` method ichida parent class'ning prototype method'iga murojaat qiladi:

```javascript
class Animal {
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  speak() {
    const parentResult = super.speak(); // Animal.prototype.speak chaqiriladi
    return `${parentResult} — specifically, barking!`;
  }
}

const rex = new Dog();
rex.name = "Rex"; // Dog va Animal da constructor yo'q, shuning uchun qo'lda
console.log(rex.speak());
// "Rex makes a sound — specifically, barking!"
```

### Ko'p Darajali Inheritance

```javascript
class Animal {
  constructor(name) { this.name = name; }
  eat() { return `${this.name} yemoqda`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }
  bark() { return `${this.name} havlayapti!`; }
}

class GuideDog extends Dog {
  constructor(name, breed, owner) {
    super(name, breed);
    this.owner = owner;
  }
  guide() { return `${this.name} ${this.owner}ni boshqaryapti`; }
}

const buddy = new GuideDog("Buddy", "Labrador", "Ali");
console.log(buddy.eat());    // "Buddy yemoqda" (Animal'dan)
console.log(buddy.bark());   // "Buddy havlayapti!" (Dog'dan)
console.log(buddy.guide());  // "Buddy Alini boshqaryapti" (GuideDog)
console.log(buddy instanceof GuideDog); // true
console.log(buddy instanceof Dog);      // true
console.log(buddy instanceof Animal);   // true
```

---

## Private Fields va Methods

### Nazariya

ES2022 dan beri JavaScript'da **haqiqiy private** field va method'lar mavjud — `#` prefix bilan. Bu [05-closures.md](05-closures.md) da ko'rsatilgan closure-based privacy dan farqli ravishda, **til darajasida** himoya qiladi: class tashqarisidan `#` field'ga kirish **SyntaxError** beradi.

Private field'lar nima uchun kerak? Encapsulation — ichki implementation detallarini yashirish. Bu refactoring ni osonlashtiradi (ichki field nomini o'zgartirsangiz, tashqi kod buzilmaydi) va xavfsizlikni oshiradi (tashqi kod ichki holatni o'zgartira olmaydi).

`#` field'lar `_convention` (underscore prefix) dan qanday farq qiladi? `_name` — bu faqat **convention** (kelishuv), hech qanday himoya yo'q — tashqi kod `obj._name` ga bemalol kiradi. `#name` — bu **til enforsementi**, tashqi kodda `obj.#name` deb yozish **SyntaxError**.

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine private field'larni internal data structure'da saqlaydi. Bu struktura konseptual ravishda **WeakMap-ga o'xshash** (per-class brand bilan instance → private value mapping) — aniq implementation V8 versiyalari bo'yicha evolyutsiyalangan, modern V8'da private symbol va object slot kombinatsiyasi ishlatiladi. Har bir class uchun **brand** (maxsus identifikator) yaratiladi. Private field'ga murojaat qilinganida engine avval object'ning brand'ini tekshiradi — agar mos kelmasa, `TypeError` beradi.

```javascript
class User {
  #name;
  #age;

  constructor(name, age) {
    this.#name = name;
    this.#age = age;
  }

  getInfo() {
    return `${this.#name}, ${this.#age}`;
  }

  // Private method:
  #validate() {
    return this.#name.length > 0 && this.#age > 0;
  }

  save() {
    if (!this.#validate()) throw new Error("Validation failed");
    return `${this.#name} saqlandi`;
  }
}

const user = new User("Ali", 25);
console.log(user.getInfo());    // "Ali, 25"
console.log(user.save());       // "Ali saqlandi"

// Tashqaridan kirish mumkin emas:
// console.log(user.#name); // ❌ SyntaxError
// user.#validate();        // ❌ SyntaxError
```

</details>

### Private Field Xususiyatlari

```javascript
class Counter {
  #count = 0;

  increment() { this.#count++; }
  get value() { return this.#count; }
}

const c = new Counter();
c.increment();
console.log(c.value); // 1

// 1. # field for...in, Object.keys da ko'rinmaydi:
console.log(Object.keys(c));                   // []
console.log(Object.getOwnPropertyNames(c));    // []

// 2. JSON.stringify ham ko'rmaydi:
console.log(JSON.stringify(c)); // "{}"

// 3. Reflect.ownKeys ham ko'rmaydi:
console.log(Reflect.ownKeys(c)); // []

// 4. Subclass ham kira olmaydi:
class DoubleCounter extends Counter {
  doubleIt() {
    // this.#count *= 2; // ❌ SyntaxError — child class ham kira olmaydi!
  }
}
```

### Private Static

```javascript
class Config {
  static #instance = null;

  static getInstance() {
    if (!Config.#instance) {
      Config.#instance = new Config();
    }
    return Config.#instance;
  }

  // Private constructor pattern:
  constructor() {
    if (Config.#instance) throw new Error("Singleton!");
  }
}

const c1 = Config.getInstance();
const c2 = Config.getInstance();
console.log(c1 === c2); // true — bitta instance
```

### in Operator bilan Private Field Tekshirish

ES2022 da `#field in obj` sintaksisi qo'shildi — object'da private field borligini tekshirish:

```javascript
class User {
  #name;
  constructor(name) { this.#name = name; }

  static isUser(obj) {
    return #name in obj; // ✅ brand check
  }
}

console.log(User.isUser(new User("Ali"))); // true
console.log(User.isUser({ name: "Ali" })); // false — #name yo'q
```

---

## Public Class Fields

### Nazariya

Public class field'lar — constructor'siz instance property e'lon qilish imkonini beradi. Har bir yangi instance'da bu field'lar alohida yaratiladi (prototype'da emas). Bu ES2022 standartining bir qismi.

<details>
<summary><strong>Under the Hood</strong></summary>

Public class field'lar spec da **Define semantics** ishlatadi — **Set semantics** emas. Bu muhim farq:

- **Define** (`Object.defineProperty`) — property ni to'g'ridan-to'g'ri object ga qo'shadi, prototype chain dagi setter'larni **chaqirmaydi**
- **Set** (`obj.prop = value`) — prototype chain bo'ylab setter qidiradi va topsa chaqiradi

Bu degani: agar parent class da `set name(v) { ... }` bo'lsa ham, child class dagi `name = "default"` field shu setter ni **bypass** qiladi va to'g'ridan-to'g'ri own property yaratadi. Bu TC39 tomonidan ataylab tanlangan — field'lar predictable bo'lishi uchun.

Engine ichida class field'lar **ClassFieldDefinition** record sifatida class evaluation vaqtida yig'iladi. Har bir field `[[Name]]` (property key) va `[[Initializer]]` (thunk function) dan iborat. `new` bilan constructor chaqirilganida, constructor body bajarilgandan **keyin** field initializer'lar tartib bo'yicha evaluate bo'ladi. Initializer ichida `this` yangi yaratilgan instance ga ishora qiladi.

V8 da class field'lar object ning **Hidden Class** transition chain'iga oldindan qo'shiladi. V8 class literal ni parse qilganida barcha field'larning nomlarini biladi va `InitialMap` ni yaratadi (V8 source'dagi "Map" — Hidden Class'ning ichki nomi, JS `Map` data structure emas) — bu instance creation ni tezlashtiradi chunki property'lar uchun shape transition'lar oldindan hisoblangan bo'ladi. Arrow function field'lar uchun har instance da yangi `JSFunction` allocate qilinadi — bu prototype method'ga nisbatan ko'proq memory ishlatadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class Button {
  // Public class fields — har instance'da yaratiladi:
  type = "button";
  disabled = false;
  clickCount = 0;

  constructor(label) {
    this.label = label;
  }

  click() {
    if (!this.disabled) {
      this.clickCount++;
    }
  }
}

const btn = new Button("Submit");
console.log(btn.type);       // "button"
console.log(btn.clickCount); // 0

// Field'lar instance'da, prototype'da emas:
console.log(btn.hasOwnProperty("type"));                  // true ✅
console.log(Button.prototype.hasOwnProperty("type"));     // false
console.log(Button.prototype.hasOwnProperty("click"));    // true — method prototype'da
```

</details>

### Arrow Function Field — this Binding

Class field'da arrow function aniqlash — `this` ni avtomatik bind qiladi. Bu event handler'lar uchun juda qulay:

```javascript
class Timer {
  count = 0;

  // Arrow function field — this doim Timer instance ga bog'liq:
  increment = () => {
    this.count++;
    console.log(this.count);
  };
}

const timer = new Timer();
const fn = timer.increment;
fn(); // 1 ✅ — this yo'qolmaydi

// Oddiy method bilan bo'lsa:
class Timer2 {
  count = 0;
  increment() { // prototype method
    this.count++;
    console.log(this.count);
  }
}
const timer2 = new Timer2();
const fn2 = timer2.increment;
// fn2(); // ❌ TypeError: Cannot read properties of undefined
```

**Lekin:** Arrow function field — har instance uchun alohida funksiya yaratadi (instance method). Prototype method emas — memory ko'proq ishlatadi. Faqat `this` binding muammosi bo'lganda ishlating (event handler'lar).

---

## Static Properties va Methods

### Nazariya

`static` keyword bilan aniqlangan property va method'lar **class'ning o'zida** saqlanadi, instance'larda emas. Ular `new` bilan yaratilgan object orqali emas, class nomi orqali chaqiriladi.

Static nima uchun kerak? Utility funksiyalar (data bilan bog'liq emas), factory method'lar (instance yaratish logikasi), va singleton kabi pattern'lar uchun.

`extends` bilan inheritance qilinganda static method'lar ham meros bo'ladi — chunki `Child.[[Prototype]] = Parent`.

<details>
<summary><strong>Under the Hood</strong></summary>

Static method va property'lar constructor function object'ning **own property** lari sifatida saqlanadi. Spec bo'yicha `static` keyword bilan aniqlangan element'lar `ClassDefinitionEvaluation` vaqtida constructor function'ga `Object.defineProperty` orqali qo'shiladi — instance method'lar `prototype` ga qo'shilganidek.

Static method'lar ham instance method'lar kabi `enumerable: false`, `writable: true`, `configurable: true` descriptor bilan define qilinadi. Static field'lar esa **Define semantics** bilan (Set emas) constructor function'ga own property sifatida qo'shiladi.

`extends` ishlatilganida `Object.setPrototypeOf(Child, Parent)` chaqiriladi. Natijada `Child.[[Prototype]] === Parent` bo'ladi. Shuning uchun `Child.staticMethod()` chaqirilganida — V8 avval `Child` ning own property'larini tekshiradi, topilmasa `[[Prototype]]` chain bo'ylab `Parent` da qidiradi. Bu oddiy prototype lookup — static method'lar kopiyalanmaydi, delegation orqali ishlaydi.

V8 ichida static method'lar constructor `JSFunction` object'ning property table'ida saqlanadi. `this` konteksti chaqirish vaqtida aniqlanadi — `Child.create()` da `this === Child`, `Parent.create()` da `this === Parent`. Bu factory method pattern da muhim — `new this()` chaqirilganida to'g'ri class instantiate bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class MathUtils {
  // Static method — class nomi bilan chaqiriladi:
  static add(a, b) { return a + b; }
  static multiply(a, b) { return a * b; }

  // Static property:
  static PI = 3.14159;
}

console.log(MathUtils.add(2, 3));    // 5
console.log(MathUtils.PI);           // 3.14159

// Instance'dan kirish mumkin emas:
const m = new MathUtils();
// m.add(2, 3); // ❌ TypeError: m.add is not a function
```

</details>

### Factory Pattern bilan Static Method

```javascript
class User {
  constructor(name, email, role) {
    this.name = name;
    this.email = email;
    this.role = role;
  }

  // Factory methods — turli xil User yaratish:
  static createAdmin(name, email) {
    return new User(name, email, "admin");
  }

  static createGuest() {
    return new User("Guest", "guest@temp.com", "guest");
  }

  // Data dan yaratish:
  static fromJSON(json) {
    const data = JSON.parse(json);
    return new User(data.name, data.email, data.role);
  }
}

const admin = User.createAdmin("Ali", "ali@mail.com");
const guest = User.createGuest();
const fromData = User.fromJSON('{"name":"Vali","email":"v@mail.com","role":"user"}');
```

### Static Inheritance

```javascript
class Animal {
  static kingdom = "Animalia";
  static create(name) { return new this(name); } // this = chaqirilgan class
}

class Dog extends Animal {
  bark() { return "Hav!"; }
}

// Static method meros bo'ladi:
console.log(Dog.kingdom);    // "Animalia" — Animal'dan
const rex = Dog.create("Rex"); // this = Dog, new Dog("Rex")
console.log(rex instanceof Dog); // true ✅
console.log(rex.bark());         // "Hav!"
```

---

## Static Initialization Blocks

### Nazariya

ES2022 da qo'shilgan `static { }` bloki — class yaratilganda (instance emas!) bir marta ishlaydigan **initialization kod**. Bu murakkab static property'larni yaratish uchun qulay — masalan, try/catch kerak bo'lganda yoki bir nechta static property'lar o'zaro bog'liq bo'lganda.

<details>
<summary><strong>Under the Hood</strong></summary>

Spec da static block **ClassStaticBlockDefinition** record sifatida ifodalanadi. Bu record `[[BodyFunction]]` field'iga ega — anonymous function bo'lib, class evaluation vaqtida call qilinadi. `this` qiymati class constructor'ning o'ziga set bo'ladi.

Static block'lar class body dagi boshqa static element'lar bilan **textual order** da evaluate bo'ladi. Ya'ni agar static field, keyin static block, keyin yana static field bo'lsa — aynan shu tartibda bajariladi. Bu ordering kafolati spec da aniq belgilangan va engine'lar uni to'g'ri implement qilishi shart.

Static block ichida `this` class constructor'ga teng bo'lgani uchun, private static field'larga ham murojaat qilish mumkin — `this.#privateField`. Bu static block'ning asosiy use case'laridan biri: private static member'larga class tashqarisidan kirish imkoni beruvchi "friend" pattern. Masalan, static block ichida tashqi variable'ga private field accessor'ni assign qilish mumkin.

V8 da static block parse vaqtida maxsus `ClassStaticBlock` AST node sifatida saqlanadi. Bytecode generation da u oddiy function call ga aylanadi. Static block ichidagi error (exception) class creation ni to'xtatadi — class object yaratilmay qoladi va binding uninitialized holatda qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class Database {
  static connection;
  static isConnected;

  // Static initialization block — class load bo'lganda ishlaydi:
  static {
    try {
      Database.connection = Database.#connectToDb();
      Database.isConnected = true;
    } catch (error) {
      Database.connection = null;
      Database.isConnected = false;
      console.error("DB connection failed:", error.message);
    }
  }

  static #connectToDb() {
    // Simulated connection
    return { host: "localhost", port: 5432 };
  }
}

console.log(Database.isConnected); // true
console.log(Database.connection);  // { host: "localhost", port: 5432 }
```

Bir nechta static block ham bo'lishi mumkin — tartib bo'yicha ishlaydi:

```javascript
class Config {
  static env;
  static debug;

  static {
    Config.env = "production";
  }

  static {
    Config.debug = Config.env !== "production";
  }
}

console.log(Config.env);   // "production"
console.log(Config.debug); // false
```

</details>

---

## Accessor — Getter va Setter

### Nazariya

`get` va `set` keyword'lar class ichida computed property yaratish imkonini beradi. Tashqaridan oddiy property kabi ko'rinadi, lekin ichida funksiya ishlaydi. Bu [06-objects.md](06-objects.md) da o'rgangan getter/setter bilan bir xil mexanizm — faqat class sintaksisi bilan.

Getter/setter nima uchun kerak? Validation (qiymat o'rnatishda tekshirish), computed property (boshqa field'lardan hisoblash), va encapsulation (ichki formatni yashirish).

<details>
<summary><strong>Under the Hood</strong></summary>

Class ichidagi `get` va `set` keyword'lar spec bo'yicha `Object.defineProperty` ning **accessor descriptor** formatiga compile bo'ladi. `get propName()` method uchun property descriptor `{ get: function, set: undefined, enumerable: false, configurable: true }` bo'ladi. Agar `get` va `set` ikkalasi ham aniqlangan bo'lsa — bitta property descriptor da birlashtiriladi.

Engine ichida accessor property oddiy data property dan farqli tarzda saqlanadi. V8 da Hidden Class property descriptor turini kuzatib boradi — accessor property `kAccessor` turi bilan belgilanadi. `obj.prop` o'qilganida V8 property lookup qilib, descriptor turini tekshiradi: agar `kAccessor` bo'lsa — `[[Get]]` function ni chaqiradi; agar `kData` bo'lsa — qiymatni to'g'ridan-to'g'ri qaytaradi.

Class body dagi getter/setter `prototype` object'ga qo'shiladi — ya'ni barcha instance'lar bitta getter/setter function'ni share qiladi. Bu `Object.getOwnPropertyDescriptor(ClassName.prototype, 'propName')` orqali tekshirish mumkin. Constructor ichidagi `this.prop = value` assignment accessor'ni trigger qiladi — chunki bu `Set` semantics, prototype chain dagi setter ni topadi va chaqiradi.

JIT optimization nuqtai nazaridan V8 accessor property'lar uchun inline cache (IC) yaratadi. Tez-tez chaqiriladigan getter/setter'lar monomorphic IC orqali optimallashtiriladi — lekin megamorphic holatlarda (ko'p turli shape'lar) deoptimization sodir bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class Temperature {
  #celsius;

  constructor(celsius) {
    this.celsius = celsius; // setter orqali — validation ishlaydi
  }

  // Getter — o'qish:
  get celsius() {
    return this.#celsius;
  }

  // Setter — yozish bilan validation:
  set celsius(value) {
    if (typeof value !== "number") throw new TypeError("Son bo'lishi kerak");
    if (value < -273.15) throw new RangeError("Absolyut noldan past bo'lmaydi");
    this.#celsius = value;
  }

  // Computed property — Fahrenheit:
  get fahrenheit() {
    return this.#celsius * 9 / 5 + 32;
  }

  set fahrenheit(f) {
    this.celsius = (f - 32) * 5 / 9; // celsius setter validation ishlatadi
  }
}

const temp = new Temperature(100);
console.log(temp.celsius);    // 100
console.log(temp.fahrenheit); // 212

temp.fahrenheit = 32;
console.log(temp.celsius);    // 0

// temp.celsius = -300; // ❌ RangeError: Absolyut noldan past bo'lmaydi
```

</details>

---

## instanceof Bilan Classes

### Nazariya

`instanceof` class bilan ham xuddi constructor function bilan bir xil ishlaydi — prototype chain bo'ylab `Class.prototype` ni qidiradi. Class inheritance'da (`extends`) child instance parent uchun ham `true` qaytaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`obj instanceof Constructor` chaqirilganida engine avval `Constructor[Symbol.hasInstance]` mavjudligini tekshiradi. Agar aniqlangan bo'lsa — shu method chaqiriladi. Agar yo'q bo'lsa — default **OrdinaryHasInstance** algorithm ishlaydi:

1. `Constructor.prototype` ni oladi (agar object bo'lmasa — `TypeError`)
2. `obj.[[Prototype]]` ni oladi (ya'ni `Object.getPrototypeOf(obj)`)
3. Prototype chain bo'ylab yuqoriga yuradi: har bir `[[Prototype]]` ni `Constructor.prototype` bilan **reference equality** (`===`) orqali taqqoslaydi
4. Agar topilsa — `true`, chain oxiriga (`null`) yetsa — `false`

Bu linear search — prototype chain uzun bo'lsa, `instanceof` sekinlashadi. Lekin amalda chain kamdan-kam 3-4 dan oshadi. V8 da `instanceof` uchun **InstanceOf** bytecode instruction bor va IC (inline cache) orqali optimize qilinadi — ma'lum constructor uchun natija cache'lanadi.

`Symbol.hasInstance` — well-known symbol bo'lib, `instanceof` behavior ni to'liq override qilish imkonini beradi. `Function.prototype[Symbol.hasInstance]` da default implementation `OrdinaryHasInstance` ni chaqiradi. Class da `static [Symbol.hasInstance](instance)` method aniqlansa — prototype chain walk o'rniga custom logic ishlaydi. Bu duck typing pattern uchun ishlatiladi.

`Object.setPrototypeOf` yoki `__proto__` orqali prototype chain o'zgartirilsa — `instanceof` natijasi ham o'zgaradi, chunki u runtime da chain ni walk qiladi, cache'lanmaydi (IC invalidation bo'ladi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

const rex = new Dog();

console.log(rex instanceof Dog);    // true
console.log(rex instanceof Animal); // true
console.log(rex instanceof Cat);    // false
console.log(rex instanceof Object); // true

// Symbol.hasInstance bilan customize:
class AbstractValidator {
  static [Symbol.hasInstance](instance) {
    return typeof instance.validate === "function";
  }
}

const form = { validate() { return true; } };
console.log(form instanceof AbstractValidator); // true — duck typing
```

</details>

---

## Classes vs Prototypes — Ichki Farq

### Nazariya

`class` sintaksisi constructor function + prototype pattern'ining qulay yozuvi, lekin u **faqat syntactic sugar emas** — bir nechta muhim **behavior farqlari** mavjud. Bu farqlar engine darajasida internal slot va semantika orqali qo'llaniladi:

- **`new` majburiyligi** — class konstruktorni `new` siz chaqirish darhol `TypeError` beradi (engine `[[IsClassConstructor]]` slot'ni tekshiradi). Constructor function esa `new` siz chaqirilsa, `this` globalThis (yoki strict mode'da undefined) bo'ladi — bu silent bug manbai.
- **Strict mode avtomatik** — class body har doim strict mode'da parse qilinadi, hatto tashqi kod sloppy bo'lsa ham. Constructor function'da `"use strict"` direktivasini qo'lda yozish kerak.
- **Method `enumerable: false`** — class method'lari prototype'da non-enumerable bo'lib define qilinadi. Constructor function'da `Fn.prototype.method = ...` esa default enumerable — `for...in` loop'lar method'larni ham iteratsiyaga oladi.
- **Method'lar constructable emas** — class method'larida `[[Construct]]` internal slot yo'q. Ya'ni `new instance.method()` darhol TypeError beradi. Constructor function method'lari esa texnik jihatdan `new` bilan chaqirilishi mumkin.
- **Static inheritance avtomatik** — `class Child extends Parent` ichida engine `Child.[[Prototype]] = Parent` bog'lanishini qiladi, shuning uchun static method'lar ham meros bo'ladi. Constructor function'da bu bog'lanish qo'lda sozlanadi (`Object.setPrototypeOf(Child, Parent)`).

Bu farqlarning hammasi `[[IsClassConstructor]]`, `[[FunctionKind]]` kabi internal slot'lar va class evaluation mexanizmi orqali engine tomonidan qo'llaniladi — Under the Hood'da ularning spec-level detallari.

<details>
<summary><strong>Under the Hood</strong></summary>

Engine darajasida class va constructor function o'rtasida bir nechta spec-level farq mavjud:

1. **`[[IsClassConstructor]]`** internal slot — class constructor'da `true`, oddiy function'da `false`. `[[Call]]` internal method chaqirilganida (ya'ni `new` siz) engine bu slot ni tekshiradi — `true` bo'lsa `TypeError` throw qiladi.
2. **`[[FunctionKind]]`** — class method'larida `"method"` qiymati bo'ladi. Bu method'larda `[[Construct]]` internal method yo'qligini bildiradi — shuning uchun `new obj.method()` xato beradi.
3. **Strict mode** — class body doim strict mode da parse qilinadi. Bu `FunctionBody` ning `[[Strict]]` field'ida `true` sifatida belgilanadi. Constructor function'da esa `"use strict"` direktivasi yozilmasa sloppy mode bo'ladi.
4. **Method definition** — class ichidagi method'lar `PropertyDefinitionEvaluation` orqali `enumerable: false` bilan define qilinadi. `Constructor.prototype.method = function(){}` esa default `enumerable: true` beradi.

Babel/TypeScript class'ni ES5 ga transpile qilganida bu farqlarni emulyatsiya qiladi: `_classCallCheck` funksiyasi `[[IsClassConstructor]]` ni, `_createClass` funksiyasi `Object.defineProperty` bilan non-enumerable method'larni, va `"use strict"` directive strict mode ni ta'minlaydi. Lekin native class V8 da to'g'ridan-to'g'ri `ClassLiteral` AST node sifatida optimallashtirilgan — transpiled versiyaga nisbatan kamroq overhead beradi.

</details>

### Taqqoslash

| Xususiyat | Constructor Function | Class |
|-----------|---------------------|-------|
| **Hoisting** | To'liq hoist | TDZ (e'londan oldin ishlatib bo'lmaydi) |
| **Strict mode** | Default sloppy | Har doim strict |
| **`new` siz chaqirish** | Ishlaydi (xavfli) | `TypeError` |
| **Method enumerable** | `true` (default) | `false` |
| **Method `new` bilan** | Mumkin | `TypeError` (construct yo'q) |
| **`typeof`** | `"function"` | `"function"` |
| **Static meros** | Qo'lda qilish kerak | `extends` avtomatik |

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// 1. new siz chaqirish:
function UserFn(name) { this.name = name; }
UserFn("Ali"); // ✅ ishlaydi — lekin this = globalThis (bug!)

class UserClass {
  constructor(name) { this.name = name; }
}
// UserClass("Ali"); // ❌ TypeError: Class constructor cannot be invoked without 'new'

// 2. Method enumerable:
function PersonFn(name) { this.name = name; }
PersonFn.prototype.greet = function() {};

class PersonClass {
  constructor(name) { this.name = name; }
  greet() {}
}

// for...in farqi:
const fn = new PersonFn("Ali");
const cls = new PersonClass("Ali");

for (const key in fn)  console.log(key); // "name", "greet" — method ham!
for (const key in cls) console.log(key); // "name" — method ko'rinmaydi ✅

// 3. Method ni new bilan chaqirish:
new PersonFn.prototype.greet(); // ✅ ishlaydi (xato, lekin ruxsat beriladi)
// new PersonClass.prototype.greet(); // ❌ TypeError: not a constructor
```

</details>

### Transpile — Class Engine Ichida

Babel yoki TypeScript class'ni ES5 ga transpile qilganda quyidagiga o'xshash natija beradi:

```javascript
// ES6 class:
class User {
  constructor(name) { this.name = name; }
  greet() { return this.name; }
  static create(name) { return new User(name); }
}

// Transpiled (soddalashtirilgan):
"use strict";
function User(name) {
  if (!(this instanceof User)) {
    throw new TypeError("Cannot call a class as a function");
  }
  this.name = name;
}

Object.defineProperty(User.prototype, "greet", {
  value: function() { return this.name; },
  writable: true,
  configurable: true,
  enumerable: false  // ← class method non-enumerable
});

User.create = function(name) { return new User(name); };
```

---

## Mixins — Ko'p Merosxo'rlik Muammosi

### Nazariya

JavaScript **single inheritance** — har bir class faqat bitta parent'dan `extends` qila oladi. Lekin ba'zan bir nechta independent capability (serialization, validation, event emitting) kerak bo'ladi. **Mixin** pattern bu muammoni hal qiladi — class'ga qo'shimcha method'lar "aralashtirish" (mix in) orqali qo'shiladi.

Mixin class'larda qanday implement qilinadi? **Subclass factory** pattern — funksiya qabul qiladigan class'ni argument sifatida olib, uni extend qilgan yangi class qaytaradi:

<details>
<summary><strong>Under the Hood</strong></summary>

Mixin pattern `extends` clause da dynamic expression bo'lishi mumkinligiga asoslanadi. Spec bo'yicha `class Child extends Expression` da `Expression` runtime da evaluate bo'ladi va natija constructor function bo'lishi kerak (`[[Construct]]` internal method bo'lishi shart, aks holda `TypeError`). Bu `extends` ni faqat statik class nomi bilan emas, funksiya chaqiruvi bilan ham ishlatish imkonini beradi.

`Serializable(Timestamped(Validatable(Object)))` chaining da nima sodir bo'ladi: har bir mixin funksiya chaqirilganida yangi anonymous class yaratiladi. Natijada prototype chain quyidagicha quriladi:

```
instance → MixinA.prototype → MixinB.prototype → MixinC.prototype → Object.prototype → null
```

Har bir mixin chaqiruvi `ClassDefinitionEvaluation` ni trigger qiladi va yangi `JSFunction` + `prototype` object pair yaratiladi. V8 nuqtai nazaridan bu prototype chain uzayishi demak — property lookup har bir chain element'ini tekshiradi. Lekin V8 ning inline cache (IC) mexanizmi tez-tez murojaat qilinadigan property'lar uchun lookup natijasini cache'laydi.

`Object.assign(Target.prototype, mixin)` alternativi ham ishlatiladi — bu prototype chain uzaytirmaydi, lekin `super` ishlamaydi va `instanceof` check ham mixin uchun `false` qaytaradi. Subclass factory pattern esa to'liq prototype chain quradi, `super` ishlaydi va `instanceof` to'g'ri natija beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Mixin yaratish — subclass factory pattern:
const Serializable = (Base) => class extends Base {
  serialize() {
    return JSON.stringify(this);
  }

  static deserialize(json) {
    return Object.assign(new this(), JSON.parse(json));
  }
};

const Timestamped = (Base) => class extends Base {
  constructor(...args) {
    super(...args);
    this.createdAt = new Date();
  }

  getAge() {
    return Date.now() - this.createdAt.getTime();
  }
};

const Validatable = (Base) => class extends Base {
  validate() {
    for (const [key, val] of Object.entries(this)) {
      if (val === null || val === undefined) {
        throw new Error(`${key} bo'sh bo'lmasligi kerak`);
      }
    }
    return true;
  }
};

// Mixin'larni qo'llash — chaining:
// Object ni base sifatida berish kerak — chunki har bir mixin factory
// argumentga "Base" class'ni kutadi va uni extend qiladi. Object barcha
// class'larning default parent'i, shuning uchun zanjir boshi sifatida ishlatiladi.
class User extends Serializable(Timestamped(Validatable(Object))) {
  constructor(name, email) {
    super();  // Serializable → Timestamped → Validatable → Object zanjiri bo'ylab
    this.name = name;
    this.email = email;
  }
}

const user = new User("Ali", "ali@mail.com");
console.log(user.serialize());    // JSON string
console.log(user.validate());     // true
console.log(user.createdAt);      // Date object
console.log(user instanceof User); // true
```

</details>

### Mixin Muammolari

1. **Naming collision** — ikki mixin bir xil method nomi bilan
2. **Diamond problem** — ikki mixin bir xil parent'dan meros olsa
3. **Constructor complexity** — mixin'lar super() chaqirilish tartibiga bog'liq
4. **Debugging qiyinligi** — method qaysi mixin'dan kelganini aniqlash qiyin

---

## Composition vs Inheritance

### Nazariya

**Inheritance** (meros) — "X **bu** Y" munosabati. `Dog extends Animal` — "Dog bu Animal".

**Composition** (tarkib) — "X **da** Y bor" munosabati. `Car` da `Engine` bor — lekin Car Engine emas.

Gang of Four (GoF) design patterns kitobidagi mashhur qoida: **"Favor composition over inheritance"** — imkon qadar inheritance o'rniga composition ishlating. Sababi: inheritance **tight coupling** yaratadi (parent o'zgarsa child buziladi), composition esa **loose coupling** beradi (komponentlar alohida o'zgartriladi).

<details>
<summary><strong>Under the Hood</strong></summary>

Inheritance va composition V8 ning object representation'iga turlicha ta'sir qiladi:

**Inheritance** da barcha instance'lar bir xil prototype chain'ga ega — V8 bu uchun bitta Hidden Class transition tree yaratadi. Bu monomorphic IC uchun ideal — property access tez bo'ladi. Lekin chuqur inheritance chain (ya'ni prototype chain uzun bo'lgan holatda) IC cache miss paytida property lookup'ni sekinlashtirishi mumkin — chunki har bir prototype level ketma-ket tekshiriladi. Aniq ta'sir cache holatiga (monomorphic/polymorphic/megamorphic) va kod pattern'iga bog'liq.

**Composition** da (`Object.assign` yoki spread bilan) barcha method'lar instance'ning **own property** lari bo'ladi. Bu V8 da boshqa `Map` shape yaratadi — har instance'da ko'proq property bo'lgani uchun `Map` kattaroq bo'ladi. Lekin lookup tez — chunki prototype chain yurish shart emas, faqat own property tekshiriladi.

Memory nuqtai nazaridan: inheritance da method'lar prototype'da bitta nusxada saqlanadi — 1000 ta instance uchun 1 ta function object. Composition da (`Object.assign` bilan factory pattern) har instance uchun method reference own property sifatida saqlanadi — 1000 ta instance uchun 1000 ta property entry (lekin function object'ning o'zi share bo'lishi mumkin, faqat property descriptor qo'shimcha memory oladi).

V8 ning object shape tracking tizimi inheritance pattern uchun yaxshiroq optimallashtirilgan — chunki barcha instance'lar bir xil `Map` ni share qiladi. Composition da har xil capability kombinatsiyalari turli `Map` lar hosil qiladi — bu **megamorphic** holat yaratib, IC performance ni tushirishi mumkin.

</details>

### Qachon Inheritance?

- Haqiqiy "is-a" munosabat bor: `Dog extends Animal`, `Circle extends Shape`
- Parent class barqaror va kamdan-kam o'zgaradi
- Child class parent'ning **barcha** xulq-atvorini meros olishi kerak

### Qachon Composition?

- "has-a" yoki "can-do" munosabat bor: `Car` da `Engine` bor, `User` `Serializable`
- Turli xil capability'larni aralashtirish kerak
- Runtime da xulq-atvorni o'zgartirish kerak

### Muammo: Inheritance Bilan Multiple Behavior

```javascript
// ❌ Inheritance orqali — rigid, tight coupling:
class FlyingSwimmingDuck extends FlyingAnimal {
  // Agar SwimmingAnimal dan ham meros olish kerak bo'lsa?
  // JavaScript da multiple inheritance yo'q!
}

// Inheritance bilan "Diamond Problem" yuzaga keladi:
//
//       Animal
//      /      \
//   Flying   Swimming
//      \      /
//       Duck ???  ← qaysi Animal.eat() ni oladi?
//
// JavaScript buni tuzatolmaydi — faqat BITTA extends mumkin.
// Yechim: Composition!
```

### 1. Object Composition (Functional Pattern)

Factory function'lar orqali behavior'larni object'ga qo'shish. Bu eng "toza" composition — **class yo'q, prototype yo'q, faqat object + function**:

```javascript
// ─── Behavior factory'lar (capability) ───
// Har biri state ni olib, method'lar qaytaradi:
const canFly = (state) => ({
  fly() {
    state.altitude += 10;
    return `${state.name} uchmoqda (${state.altitude}m)`;
  },
  land() {
    state.altitude = 0;
    return `${state.name} qo'ndi`;
  }
});

const canSwim = (state) => ({
  swim() {
    state.depth += 5;
    return `${state.name} suzmoqda (${state.depth}m chuqurlikda)`;
  },
  surface() {
    state.depth = 0;
    return `${state.name} suvdan chiqdi`;
  }
});

const canWalk = (state) => ({
  walk() {
    state.distance += 1;
    return `${state.name} yurmoqda (${state.distance}km)`;
  }
});

// ─── Entity factory'lar ───
// Kerakli behavior'larni spread bilan birlashtiradi:
function createDuck(name) {
  const state = { name, altitude: 0, depth: 0, distance: 0 };
  return {
    get name() { return state.name; },     // faqat o'qish
    getStats() { return { ...state }; },   // state nusxasi
    ...canFly(state),
    ...canSwim(state),
    ...canWalk(state)
  };
}

function createPenguin(name) {
  const state = { name, depth: 0, distance: 0 };
  return {
    get name() { return state.name; },
    getStats() { return { ...state }; },
    ...canSwim(state),
    ...canWalk(state)
    // canFly yo'q — penguin uchmaydi!
  };
}

function createFish(name) {
  const state = { name, depth: 0 };
  return {
    get name() { return state.name; },
    ...canSwim(state)
    // na yuradi, na uchadi
  };
}

// ─── Ishlatish ───
const duck = createDuck("Donald");
console.log(duck.fly());   // "Donald uchmoqda (10m)"
console.log(duck.swim());  // "Donald suzmoqda (5m chuqurlikda)"
console.log(duck.walk());  // "Donald yurmoqda (1km)"

const penguin = createPenguin("Tux");
console.log(penguin.swim()); // "Tux suzmoqda (5m chuqurlikda)"
console.log(penguin.walk()); // "Tux yurmoqda (1km)"
// penguin.fly — undefined, method yo'q

const fish = createFish("Nemo");
console.log(fish.swim());   // "Nemo suzmoqda (5m chuqurlikda)"
```

**Afzalliklari:**
- **True encapsulation** — `state` closure ichida yashiringan, tashqaridan o'zgartirib bo'lmaydi
- **Flexible** — istalgan behavior kombinatsiyasi, hech qanday cheklov yo'q
- **No `this` muammosi** — closure tufayli `this` kerak emas, callback'larda ham ishlaydi

**Kamchiliklari:**
- **`instanceof` ishlamaydi** — bu oddiy object, class emas
- **Memory** — har instance'da barcha method'lar own property (prototype sharing yo'q)
- **Method qo'shish** — keyinchalik yangi behavior qo'shish uchun factory'ni o'zgartirish kerak

### 2. Class Composition (Delegation Pattern)

Class ichida boshqa object/class'larni **property** sifatida saqlash va ularga **delegatsiya** qilish. OOP dunoyisida bu **"has-a"** munosabat:

```javascript
// ─── Alohida capability class'lar ───
class Engine {
  #horsepower;
  #running = false;

  constructor(hp) {
    this.#horsepower = hp;
  }

  start() {
    this.#running = true;
    return `Motor yondi (${this.#horsepower} HP)`;
  }

  stop() {
    this.#running = false;
    return "Motor o'chdi";
  }

  get isRunning() { return this.#running; }
  get power() { return this.#horsepower; }
}

class GPS {
  #location = { lat: 0, lng: 0 };

  navigate(lat, lng) {
    this.#location = { lat, lng };
    return `Navigatsiya: ${lat}, ${lng}`;
  }

  get location() { return { ...this.#location }; }
}

class MusicPlayer {
  #playlist = [];
  #current = null;

  addSong(song) {
    this.#playlist.push(song);
    return `"${song}" qo'shildi`;
  }

  play() {
    this.#current = this.#playlist[0] || null;
    return this.#current ? `♫ ${this.#current} ijro` : "Playlist bo'sh";
  }

  get nowPlaying() { return this.#current; }
}

// ─── Composition: Car "has-a" Engine, GPS, MusicPlayer ───
class Car {
  #name;
  #engine;    // Engine instance
  #gps;       // GPS instance
  #player;    // MusicPlayer instance

  constructor(name, hp) {
    this.#name = name;
    this.#engine = new Engine(hp);      // has-a Engine
    this.#gps = new GPS();              // has-a GPS
    this.#player = new MusicPlayer();   // has-a MusicPlayer
  }

  // Engine'ga delegation:
  start() { return `${this.#name}: ${this.#engine.start()}`; }
  stop() { return `${this.#name}: ${this.#engine.stop()}`; }

  // GPS'ga delegation:
  navigate(lat, lng) { return this.#gps.navigate(lat, lng); }

  // MusicPlayer'ga delegation:
  addSong(song) { return this.#player.addSong(song); }
  play() { return this.#player.play(); }

  // Composed behavior — bir nechta component'dan foydalanish:
  dashboard() {
    return {
      car: this.#name,
      engineRunning: this.#engine.isRunning,
      power: this.#engine.power,
      location: this.#gps.location,
      nowPlaying: this.#player.nowPlaying
    };
  }
}

const tesla = new Car("Tesla Model 3", 450);
console.log(tesla.start());              // "Tesla Model 3: Motor yondi (450 HP)"
console.log(tesla.navigate(41.31, 69.28)); // "Navigatsiya: 41.31, 69.28"
console.log(tesla.addSong("Bohemian Rhapsody"));
console.log(tesla.play());               // "♫ Bohemian Rhapsody ijro"
console.log(tesla.dashboard());
// {
//   car: "Tesla Model 3", engineRunning: true, power: 450,
//   location: { lat: 41.31, lng: 69.28 }, nowPlaying: "Bohemian Rhapsody"
// }
```

**Afzalliklari:**
- **`instanceof` ishlaydi** — `tesla instanceof Car` → `true`
- **Strong encapsulation** — private field'lar orqali component'lar himoyalangan
- **Alohida test** — har bir component (Engine, GPS) mustaqil test qilinadi
- **Runtime da almashtiriladi** — Engine ni boshqa Engine bilan o'zgartirish mumkin (Strategy Pattern)

**Kamchiliklari:**
- **Delegation boilerplate** — har method uchun "wrapper" yozish kerak
- **Type checking** — `tesla instanceof Engine` → `false` (semantik to'g'ri, lekin ba'zan noqulay)

### 3. Dynamic Composition — Runtime da Behavior O'zgartirish

Composition'ning eng kuchli tomoni — runtime da component'larni **almashtirish** (Strategy Pattern):

```javascript
// ─── Movement strategiyalari ───
class WalkMovement {
  move(name) { return `${name} yurmoqda 🚶`; }
  get speed() { return 5; }
}

class RunMovement {
  move(name) { return `${name} yugurmoqda 🏃`; }
  get speed() { return 15; }
}

class FlyMovement {
  move(name) { return `${name} uchmoqda 🦅`; }
  get speed() { return 100; }
}

class SwimMovement {
  move(name) { return `${name} suzmoqda 🏊`; }
  get speed() { return 8; }
}

// ─── Character class — movement strategiyasini runtime da o'zgartiradi ───
class Character {
  #name;
  #movement;  // delegation target — runtime da o'zgaradi

  constructor(name, movement = new WalkMovement()) {
    this.#name = name;
    this.#movement = movement;
  }

  // Movement'ga delegation:
  move() { return this.#movement.move(this.#name); }
  get speed() { return this.#movement.speed; }

  // ✨ Runtime da strategiyani almashtirish:
  setMovement(movement) {
    this.#movement = movement;
    return `${this.#name} harakatlanish usulini o'zgartirdi`;
  }
}

// ─── Ishlatish ───
const hero = new Character("Jasur");

console.log(hero.move());  // "Jasur yurmoqda 🚶"
console.log(hero.speed);   // 5

hero.setMovement(new RunMovement());
console.log(hero.move());  // "Jasur yugurmoqda 🏃"
console.log(hero.speed);   // 15

hero.setMovement(new FlyMovement());
console.log(hero.move());  // "Jasur uchmoqda 🦅"
console.log(hero.speed);   // 100

// Bu inheritance bilan mumkin emas —
// runtime da class'ni o'zgartira olmaysiz!
```

### Pattern'lar Taqqoslashi

| Xususiyat | Functional Composition | Class Composition (Delegation) |
|-----------|----------------------|-------------------------------|
| Yaratish | factory function | `new ClassName()` |
| Encapsulation | closure (kuchli) | private fields (kuchli) |
| `instanceof` | ❌ ishlamaydi | ✅ ishlaydi |
| `this` muammosi | ❌ yo'q (closure) | ⚠️ bor (lekin private field yordamida) |
| Memory | har instance'da own method | prototype sharing ishlaydi |
| Runtime o'zgartirish | spread bilan yangi object | property assignment bilan |
| Test qilish | oson (pure function) | oson (alohida component) |
| TypeScript bilan | interfeys qiyin | interfeys oson |
| Ishlatiladigan joy | utility, plugin, small object | domain model, OOP architecture |

---

## Edge Cases va Gotchas

### `class` body har doim strict mode

Class body hech qachon sloppy mode'da bo'lolmaydi — bu spec qoidasi. Hatto tashqi kod sloppy bo'lsa ham, class ichidagi barcha kod strict mode qoidalariga bo'ysunadi. Bu opt-out qilib bo'lmaydigan xususiyat.

```javascript
// Tashqi kod sloppy bo'lsa ham:
function sloppyOuter() {
  // "use strict" YOQ

  class Inside {
    test() {
      // ❌ Strict mode xatolari:
      // const n = 0777;          // Octal literal — SyntaxError
      // with ({a: 1}) { log(a); } // with statement — SyntaxError
      // undeclared = 10;         // Implicit global — ReferenceError

      // ✅ Method'da this:
      return this;  // strict mode: method context'da this = instance
                    // sloppy mode bo'lsa globalThis bo'lib qolardi
    }
  }

  return new Inside().test();
}
```

**Nima uchun:** Spec ataylab class body'ni strict mode'da majburiy qilgan — bu modern JavaScript'ning "clean slate" yondashuvi. Legacy sloppy mode xatolariga yo'l qo'yilmaydi, kod predictable bo'ladi.

---

### Class field initializer da `this` — yangi instance, class emas

Public va private field'lar initializer'laridagi `this` — **constructor body'dan OLDIN yaratilgan yangi instance**ga ishora qiladi. Constructor body field initializer'lardan **keyin** ishlaydi, shuning uchun field'lar constructor'dagi o'zgartirishlarni ko'rmaydi.

```javascript
class User {
  name = "default";

  // this = yangi yaratilayotgan instance (class emas)
  // Bu paytda name = "default" — constructor hali ishlamagan
  greeting = `Salom, ${this.name}`;

  constructor(name) {
    // Constructor body field init'dan KEYIN ishlaydi:
    this.name = name;
  }
}

const ali = new User("Ali");
console.log(ali.name);      // "Ali" — constructor'dan
console.log(ali.greeting);  // "Salom, default" ❌ — field init paytida name = "default"
                            // constructor argumentni ko'rmagan!
```

**Yechim:** Agar `greeting` constructor argumentga bog'liq bo'lsa — uni constructor body'da yozing, field'da emas:

```javascript
class User {
  constructor(name) {
    this.name = name;
    this.greeting = `Salom, ${name}`;  // ✅ constructor ichida
  }
}
```

---

### Static method'da `this` = chaqirilgan class (not defining class)

Static method ichidagi `this` — **metodni chaqirgan class**ga ishora qiladi, metod e'lon qilingan class emas. Bu `new this()` factory pattern'ining asosi va child class'da override bo'lmay ishlashiga imkon beradi.

```javascript
class Animal {
  static create(name) {
    return new this(name);  // this = chaqirilgan class
  }

  constructor(name) { this.name = name; }
}

class Dog extends Animal {
  bark() { return `${this.name}: Hav!`; }
}

const a = Animal.create("Cat"); // this = Animal → new Animal("Cat")
const d = Dog.create("Rex");    // this = Dog    → new Dog("Rex")

console.log(a instanceof Animal); // true
console.log(d instanceof Dog);    // true ✅ — Dog.create() ichida this = Dog
console.log(d instanceof Animal); // true (dog ham animal)
console.log(d.bark());            // "Rex: Hav!" — Dog method ishlaydi
```

**Nima uchun:** Static method'lar prototype chain orqali meros bo'ladi (`Dog.[[Prototype]] === Animal`). `Dog.create()` chaqirilganda engine `create` ni Animal'dan topadi, lekin `this` call site'ga bog'liq — `Dog.create()` → `this = Dog`. Bu factory pattern'da muhim — parent class yozgan factory child class instance'larini ham yarata oladi.

---

### `new.target` har doim chaqirilgan constructor'ni ko'rsatadi

`new.target` meta-property har bir constructor ichida **original `new` bilan chaqirilgan class**ga ishora qiladi — hatto `super()` orqali parent constructor ichida bo'lganda ham. Bu factory detection va abstract class pattern uchun ishlatiladi.

```javascript
class Parent {
  constructor() {
    // new.target = original class (Parent yoki Child)
    console.log("Yaratilgan class:", new.target.name);
  }
}

class Child extends Parent {
  constructor() {
    super();  // Parent constructor chaqiriladi, new.target = Child!
  }
}

new Parent();  // "Yaratilgan class: Parent"
new Child();   // "Yaratilgan class: Child" — super() ichida ham new.target = Child
```

Bu xususiyat **abstract class** pattern'ida qo'l keladi:

```javascript
class AbstractShape {
  constructor() {
    if (new.target === AbstractShape) {
      throw new TypeError("AbstractShape to'g'ridan-to'g'ri yaratilmaydi");
    }
  }

  area() { throw new Error("Subclass da implement qilish kerak"); }
}

class Circle extends AbstractShape {
  constructor(radius) {
    super();  // ✅ new.target = Circle, tekshiruvdan o'tadi
    this.radius = radius;
  }

  area() { return Math.PI * this.radius ** 2; }
}

// new AbstractShape();  // ❌ TypeError — abstract class
const c = new Circle(5); // ✅ ishlaydi
```

---

### Private brand check (`#x in obj`) — faqat o'z class'ida e'lon qilinsa

`#field in obj` sintaksisi (ES2022) faqat shu `#field` **e'lon qilingan class** ichida ishlaydi. Boshqa class ichida o'zi e'lon qilmagan private field'ni tekshirish — **SyntaxError**. Bu private'ning qat'iy himoyasining bir qismi.

```javascript
class User {
  #id;

  constructor(id) { this.#id = id; }

  // ✅ User ichida #id e'lon qilingan — brand check ishlaydi:
  static isUser(obj) {
    return #id in obj;
  }
}

class Admin {
  // Admin'da #id e'lon qilinmagan:
  static isUserLike(obj) {
    // return #id in obj;  // ❌ SyntaxError: Undeclared private field #id
    return User.isUser(obj);  // ✅ User orqali tekshirish
  }
}

const ali = new User(1);
console.log(User.isUser(ali));       // true ✅
console.log(User.isUser({ id: 1 })); // false — brand yo'q
console.log(Admin.isUserLike(ali));  // true (User.isUser orqali)
```

**Nima uchun:** Private field'lar "hard private" — class tashqarisida ularning mavjudligini tekshirish ham taqiqlanadi. Agar `#id in obj` boshqa class'da ishlaganida, private field'lar haqida information leak bo'lardi. Spec buni compile-time'da parser darajasida bloklaydi.

---

## Common Mistakes

### ❌ Xato 1: super() ni chaqirmasdan this ishlatish

```javascript
class Child extends Parent {
  constructor(name) {
    // ❌ super() chaqirilmagan:
    this.name = name; // ❌ ReferenceError: Must call super constructor
  }
}
```

### ✅ To'g'ri usul:

```javascript
class Child extends Parent {
  constructor(name) {
    super();         // ✅ avval super
    this.name = name; // keyin this
  }
}
```

**Nima uchun:** Derived class'da `this` object parent constructor tomonidan yaratiladi. `super()` chaqirilmaguncha `this` mavjud emas.

---

### ❌ Xato 2: Method ni callback sifatida berganda this yo'qolishi

```javascript
class Timer {
  count = 0;
  increment() {
    this.count++;
    console.log(this.count);
  }
}

const timer = new Timer();
// ❌ this yo'qoladi:
setTimeout(timer.increment, 1000); // NaN — this = globalThis (window), timer emas
document.addEventListener("click", timer.increment); // NaN — this = element, timer emas
```

### ✅ To'g'ri usul:

```javascript
// Variant 1: Arrow function field (tavsiya)
class Timer {
  count = 0;
  increment = () => { // arrow function — this bind
    this.count++;
    console.log(this.count);
  };
}

// Variant 2: bind
setTimeout(timer.increment.bind(timer), 1000);

// Variant 3: Wrapper arrow function
setTimeout(() => timer.increment(), 1000);
```

**Nima uchun:** Method callback sifatida berilganda `this` binding yo'qoladi. Arrow function field class field syntax orqali `this` ni avtomatik bind qiladi. Bu haqda ko'proq [10-this-keyword.md](10-this-keyword.md) da.

---

### ❌ Xato 3: Private field'ga subclass'dan kirish

```javascript
class Parent {
  #secret = 42;
  getSecret() { return this.#secret; }
}

class Child extends Parent {
  revealSecret() {
    // ❌ SyntaxError — private field faqat e'lon qilingan class'da accessible:
    return this.#secret;
  }
}
```

### ✅ To'g'ri usul:

```javascript
class Parent {
  #secret = 42;

  // Protected access uchun getter:
  getSecret() { return this.#secret; }
  // Yoki protected setter/getter pattern:
  _getSecret() { return this.#secret; }
}

class Child extends Parent {
  revealSecret() {
    return this.getSecret(); // ✅ public method orqali
  }
}
```

**Nima uchun:** `#` private field'lar **hard private** — faqat e'lon qilingan class body ichida accessible. Subclass ham kira olmaydi. JavaScript'da `protected` access modifier yo'q — convention bilan (underscore prefix) yoki getter/setter orqali emulyatsiya qilinadi.

---

### ❌ Xato 4: Arrow function field va prototype

```javascript
class User {
  // Arrow function field — har instance'da ALOHIDA funksiya:
  greet = () => { return `Hi, ${this.name}`; };
}

const u1 = new User();
const u2 = new User();
console.log(u1.greet === u2.greet); // false! — har biri alohida
console.log(User.prototype.hasOwnProperty("greet")); // false — prototype'da emas
```

**Nima uchun muammo:** Arrow function field har instance uchun yangi funksiya yaratadi — [07-prototypes.md](07-prototypes.md) da o'rgangan instance method muammosi. Memory ko'proq ishlatiladi. Faqat `this` binding kerak bo'lganda (event handler) ishlating, aks holda oddiy method yozing.

---

### ⚠️ Nozik nuqta: Class field'lar orasida tartib muhim

```javascript
class Config {
  baseUrl = "https://api.example.com";
  apiUrl = `${this.baseUrl}/v1`; // ✅ ishlaydi — baseUrl TEPADA aniqlangan
}

const c = new Config();
console.log(c.apiUrl); // "https://api.example.com/v1"

// ❌ Lekin TESKARI tartibda ishlamaydi:
class Broken {
  apiUrl = `${this.baseUrl}/v1`; // ❌ undefined — baseUrl HALI yaratilmagan
  baseUrl = "https://api.example.com";
}

const b = new Broken();
console.log(b.apiUrl); // "undefined/v1" — baseUrl hali mavjud emasganda evaluate bo'ldi
```

**Qoida:** Field'lar tartib bo'yicha (yuqoridan pastga) evaluate bo'ladi. Tepada aniqlangan field'ga murojaat qilish mumkin, lekin pastdagi field'ga murojaat qilish — `undefined` beradi.

---

## Amaliy Mashqlar

### Mashq 1: EventEmitter Class (Oson)

**Savol:** `on(event, handler)`, `emit(event, ...args)`, `off(event, handler)` method'lari bo'lgan `EventEmitter` class yozing.

<details>
<summary>Javob</summary>

```javascript
class EventEmitter {
  #handlers = {};

  on(event, handler) {
    if (!this.#handlers[event]) {
      this.#handlers[event] = [];
    }
    this.#handlers[event].push(handler);
    return this; // chaining uchun
  }

  off(event, handler) {
    if (!this.#handlers[event]) return this;
    this.#handlers[event] = this.#handlers[event].filter(h => h !== handler);
    return this;
  }

  emit(event, ...args) {
    if (!this.#handlers[event]) return false;
    this.#handlers[event].forEach(handler => handler(...args));
    return true;
  }
}

// Test:
const emitter = new EventEmitter();
const onMessage = (msg) => console.log(`Xabar: ${msg}`);

emitter.on("message", onMessage);
emitter.emit("message", "Salom!"); // "Xabar: Salom!"

emitter.off("message", onMessage);
emitter.emit("message", "Salom!"); // hech narsa — handler olib tashlandi
```

**Tushuntirish:** `#handlers` private field — tashqaridan o'zgartirish mumkin emas. `on` handler qo'shadi, `off` olib tashlaydi, `emit` barcha handler'larni chaqiradi. Method chaining uchun `this` qaytariladi.
</details>

---

### Mashq 2: extends + super Bilan Inheritance (O'rta)

**Savol:** `Shape` → `Rectangle` → `Square` inheritance chain yozing. Har birida `area()` va `describe()` method'lari bo'lsin.

<details>
<summary>Javob</summary>

```javascript
class Shape {
  constructor(name) {
    this.name = name;
  }

  area() {
    throw new Error("Subclass da implement qilish kerak");
  }

  describe() {
    return `${this.name}: area = ${this.area()}`;
  }
}

class Rectangle extends Shape {
  constructor(width, height) {
    super("Rectangle");
    this.width = width;
    this.height = height;
  }

  area() {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  constructor(side) {
    super(side, side); // Rectangle(side, side)
    this.name = "Square"; // override
  }
}

const rect = new Rectangle(5, 10);
console.log(rect.describe()); // "Rectangle: area = 50"

const sq = new Square(7);
console.log(sq.describe());  // "Square: area = 49"
console.log(sq.width);       // 7
console.log(sq.height);      // 7

console.log(sq instanceof Square);    // true
console.log(sq instanceof Rectangle); // true
console.log(sq instanceof Shape);     // true
```
</details>

---

### Mashq 3: Private Fields Bilan BankAccount (O'rta)

**Savol:** Private `#balance` field'li `BankAccount` class yozing: `deposit()`, `withdraw()`, `get balance`, `transfer(toAccount, amount)`.

<details>
<summary>Javob</summary>

```javascript
class BankAccount {
  #balance;
  #owner;

  constructor(owner, initialBalance = 0) {
    this.#owner = owner;
    this.#balance = initialBalance;
  }

  get balance() {
    return this.#balance;
  }

  get owner() {
    return this.#owner;
  }

  deposit(amount) {
    if (amount <= 0) throw new RangeError("Musbat son kiriting");
    this.#balance += amount;
    return this;
  }

  withdraw(amount) {
    if (amount <= 0) throw new RangeError("Musbat son kiriting");
    if (amount > this.#balance) throw new Error("Yetarli mablag' yo'q");
    this.#balance -= amount;
    return this;
  }

  transfer(toAccount, amount) {
    if (!(#balance in toAccount)) {
      throw new TypeError("Noto'g'ri account");
    }
    this.withdraw(amount);
    toAccount.deposit(amount);
    return this;
  }

  toString() {
    return `${this.#owner}: ${this.#balance} UZS`;
  }
}

const ali = new BankAccount("Ali", 1_000_000);
const vali = new BankAccount("Vali", 500_000);

ali.deposit(200_000).withdraw(100_000);
console.log(ali.balance); // 1_100_000

ali.transfer(vali, 300_000);
console.log(`${ali}`);  // "Ali: 800000 UZS"
console.log(`${vali}`); // "Vali: 800000 UZS"
```

**Tushuntirish:** `#balance in toAccount` — ES2022 private brand check. `toAccount` ning haqiqiy `BankAccount` ekanligini tekshiradi. Method chaining uchun `this` qaytariladi.
</details>

---

### Mashq 4: Mixin Pattern (Qiyin)

**Savol:** `Loggable`, `Cacheable`, `Comparable` mixin'larni yozing va ularni class'ga qo'llang.

<details>
<summary>Javob</summary>

```javascript
// Mixin'lar — subclass factory:
const Loggable = (Base) => class extends Base {
  log(message) {
    console.log(`[${this.constructor.name}] ${message}`);
  }
};

const Cacheable = (Base) => class extends Base {
  #cache = new Map();

  cached(key, computeFn) {
    if (this.#cache.has(key)) return this.#cache.get(key);
    const result = computeFn();
    this.#cache.set(key, result);
    return result;
  }

  clearCache() {
    this.#cache.clear();
  }
};

const Comparable = (Base) => class extends Base {
  compareTo(other) {
    throw new Error("Subclass da implement qilish kerak");
  }

  greaterThan(other) { return this.compareTo(other) > 0; }
  lessThan(other) { return this.compareTo(other) < 0; }
  equals(other) { return this.compareTo(other) === 0; }
};

// Barcha mixin'larni qo'llash:
class Product extends Loggable(Cacheable(Comparable(Object))) {
  constructor(name, price) {
    super();
    this.name = name;
    this.price = price;
  }

  compareTo(other) {
    return this.price - other.price;
  }

  getDescription() {
    return this.cached("description", () => {
      this.log("Description hisoblanmoqda...");
      return `${this.name}: ${this.price} UZS`;
    });
  }
}

const a = new Product("iPhone", 15_000_000);
const b = new Product("Samsung", 12_000_000);

console.log(a.greaterThan(b));    // true
console.log(a.getDescription());  // log + "iPhone: 15000000 UZS"
console.log(a.getDescription());  // cache'dan — log yo'q
```
</details>

---

### Mashq 5: Composition Pattern (Qiyin)

**Savol:** Inheritance o'rniga composition bilan `Logger`, `Validator`, `Serializer` capability'larini birlashtiring.

<details>
<summary>Javob</summary>

```javascript
// Capability factory'lar:
function createLogger(entity) {
  return {
    log(msg) { console.log(`[${entity.constructor?.name || "Object"}] ${msg}`); },
    warn(msg) { console.warn(`[WARN] ${msg}`); }
  };
}

function createValidator(entity, rules) {
  return {
    validate() {
      const errors = [];
      for (const [field, rule] of Object.entries(rules)) {
        if (rule.required && !entity[field]) {
          errors.push(`${field} majburiy`);
        }
        if (rule.minLength && entity[field]?.length < rule.minLength) {
          errors.push(`${field} kamida ${rule.minLength} ta belgi`);
        }
      }
      return { valid: errors.length === 0, errors };
    }
  };
}

function createSerializer(entity, fields) {
  return {
    toJSON() {
      const result = {};
      for (const field of fields) {
        result[field] = entity[field];
      }
      return result;
    },
    serialize() { return JSON.stringify(this.toJSON()); }
  };
}

// Composition bilan User:
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;

    // Capability'larni qo'shish:
    Object.assign(this,
      createLogger(this),
      createValidator(this, {
        name: { required: true, minLength: 2 },
        email: { required: true }
      }),
      createSerializer(this, ["name", "email"])
    );
  }
}

const user = new User("Ali", "ali@mail.com");
user.log("Yaratildi");          // "[User] Yaratildi"
console.log(user.validate());   // { valid: true, errors: [] }
console.log(user.serialize());  // '{"name":"Ali","email":"ali@mail.com"}'
```

**Tushuntirish:** Har bir capability alohida factory function. `Object.assign` orqali instance'ga qo'shiladi. Bu inheritance'dan ko'ra moslashuvchan — istalgan capability kombinatsiyasini yaratish mumkin.
</details>

---

## Xulosa

1. **Class = syntactic sugar** — ichida constructor function + prototype ishlaydi. Lekin strict mode, non-enumerable method'lar, `new` majburiyligi kabi muhim farqlar bor.

2. **Class hoisting** — TDZ da turadi (`let`/`const` kabi). Function declaration'dan farqli ravishda e'londan oldin ishlatib bo'lmaydi.

3. **`extends` va `super`** — inheritance chain quradi. `super()` derived class constructor'da `this` ishlatishdan oldin chaqirilishi SHART. `super.method()` parent method'ni chaqiradi.

4. **Private fields (`#`)** — til darajasida himoya. Tashqaridan va subclass'dan ham kirish mumkin emas. `#field in obj` bilan brand check.

5. **Public class fields** — constructor'siz instance property. Arrow function field `this` ni avtomatik bind qiladi (lekin har instance uchun alohida fn).

6. **Static** — class'ning o'zida saqlanadi. Factory method'lar, utility funksiyalar uchun. `extends` bilan meros bo'ladi.

7. **Static initialization blocks** — `static { }` — class yaratilganda bir marta ishlaydi. Murakkab initialization uchun.

8. **Getter/Setter** — computed property. Validation va encapsulation uchun.

9. **Mixins** — subclass factory pattern orqali ko'p capability qo'shish. Single inheritance cheklovi bilan kurashish usuli.

10. **Composition > Inheritance** — "has-a" munosabat uchun composition, "is-a" uchun inheritance. **Functional composition** (factory + closure) — encapsulation kuchli, `this` muammosi yo'q. **Class composition** (delegation) — `instanceof` ishlaydi, runtime da component almashtiriladi (Strategy Pattern).

---

> **Keyingi bo'lim:** [08.5-oop.md](08.5-oop.md) — Object-Oriented Programming paradigmasi: OOP 4 ustuni (Encapsulation, Abstraction, Inheritance, Polymorphism), prototypal vs classical OOP taqqoslash, SOLID printsiplari (S/O/L/I/D), Coupling va Cohesion, Law of Demeter, Tell Don't Ask, composition vs inheritance chuqur taqqoslash, delegation pattern (OLOO), method chaining, Symbol.hasInstance custom type checking, va Proxy bilan metaprogramming.
