# Bo'lim 8: ES6 Classes

> Class — prototypal inheritance ustiga qurilgan "syntactic sugar". Yozish oson, ichida esa prototype ishlaydi.

---

## Mundarija

- [Class = Syntactic Sugar](#class--syntactic-sugar)
- [Class Anatomy](#class-anatomy)
- [Class Expressions vs Declarations](#class-expressions-vs-declarations)
- [Inheritance: extends va super](#inheritance-extends-va-super)
- [Private Fields va Methods](#private-fields-va-methods)
- [Public Class Fields](#public-class-fields)
- [Static Properties va Methods](#static-properties-va-methods)
- [instanceof bilan Classes](#instanceof-bilan-classes)
- [Classes vs Prototypes — Detallar](#classes-vs-prototypes--detallar)
- [Mixins](#mixins)
- [Composition vs Inheritance](#composition-vs-inheritance)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Class = Syntactic Sugar

### Nazariya

ES6 `class` — bu **yangi narsa emas**, balki constructor function + prototype pattern'ining **chiroyli yozilishi** (syntactic sugar). Lekin bu "shunchaki qand" emas — class bir nechta muhim yaxshilanishlar olib keldi: avtomatik strict mode, `new` siz chaqirishdan himoya, method'larning non-enumerable bo'lishi, va TDZ (Temporal Dead Zone) orqali ishonchli hoisting xulq-atvori.

Nima uchun class kerak bo'ldi? ES5 da constructor function va prototype bilan ishlash juda ko'p boilerplate kod talab qilardi va yangi dasturchilar uchun tushunarsiz edi. Java, C#, Python dan kelgan dasturchilar uchun prototype-based OOP g'alati ko'rinardi. `class` sintaksisi bu muammoni hal qildi — u tanish OOP interfeys beradi, lekin ichida JavaScript'ning kuchli prototype mexanizmi ishlaydi.

Muhim tushuncha: class **prototype**ni **almashtirmaydi** — u prototype ustiga qurilgan **abstraksiya**. `typeof MyClass` — `"function"`, method'lar `MyClass.prototype` da saqlanadi, va `new` keyword ichida xuddi shu 4 qadam bajariladi. Shu sababli [07-prototypes.md](07-prototypes.md) dagi bilimlar class'larni chuqur tushunish uchun zarur.

```javascript
// ES5 — Constructor Function
function UserES5(name, age) {
  this.name = name;
  this.age = age;
}
UserES5.prototype.greet = function() {
  return `Salom, ${this.name}`;
};

// ES6 — Class (xuddi shu natija)
class UserES6 {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  greet() {
    return `Salom, ${this.name}`;
  }
}
```

### Isboti

```javascript
console.log(typeof UserES6);                    // "function" — class = function!
console.log(UserES6.prototype.greet);           // function — method prototype'da
console.log(UserES6.prototype.constructor === UserES6); // true

const u1 = new UserES5("Ali", 25);
const u2 = new UserES6("Ali", 25);

// Ikkalasi ham xuddi bir xil tuzilmaga ega:
// u1.__proto__ === UserES5.prototype
// u2.__proto__ === UserES6.prototype
```

### Lekin Ba'zi Farqlar Bor

Class **pure syntactic sugar emas** — ba'zi muhim farqlar bor:

| Xususiyat | Constructor Function | Class |
|-----------|---------------------|-------|
| **Hoisting** | Ha (function declaration) | Yo'q (TDZ) |
| **`new` siz chaqirish** | Ishlaydi (xato bermaydi) | TypeError |
| **Strict mode** | Manual qo'shish kerak | Avtomatik strict |
| **Enumerable methods** | Ha (default) | Yo'q (prototype method'lar non-enumerable) |
| **`[[IsClassConstructor]]`** | Yo'q | Ha |

```javascript
// new siz chaqirish farqi:
function OldUser(name) { this.name = name; }
OldUser("Ali"); // xato bermaydi, lekin this = window (bug!)

class NewUser {
  constructor(name) { this.name = name; }
}
NewUser("Ali"); // ❌ TypeError: Class constructor cannot be invoked without 'new'
```

---

## Class Anatomy

```javascript
class Product {
  // 1. Public class field (instance property)
  currency = "USD";

  // 2. Private field
  #discount = 0;

  // 3. Static property
  static TAX_RATE = 0.12;

  // 4. Constructor
  constructor(name, price) {
    this.name = name;
    this.price = price;
  }

  // 5. Instance method (prototype'da)
  getPrice() {
    const discounted = this.price * (1 - this.#discount);
    return discounted * (1 + Product.TAX_RATE);
  }

  // 6. Getter
  get info() {
    return `${this.name}: ${this.currency} ${this.getPrice().toFixed(2)}`;
  }

  // 7. Setter
  set discount(value) {
    if (value < 0 || value > 1) throw new RangeError("0-1 oralig'ida bo'lsin");
    this.#discount = value;
  }

  // 8. Private method
  #calculateTax() {
    return this.price * Product.TAX_RATE;
  }

  // 9. Static method
  static compare(a, b) {
    return a.price - b.price;
  }
}
```

### Qayerda Nima Saqlanadi

```
Product (function/class)
├── TAX_RATE: 0.12              ← static (class'ning o'zida)
├── compare()                    ← static method (class'ning o'zida)
│
└── Product.prototype
    ├── constructor: Product
    ├── getPrice()               ← instance method (prototype'da)
    └── get info / set discount  ← accessor (prototype'da)

new Product("Laptop", 1000)
├── name: "Laptop"              ← instance property (constructor'da)
├── price: 1000                 ← instance property
├── currency: "USD"             ← public class field (instance'da)
├── #discount: 0                ← private field (instance'da)
└── __proto__ → Product.prototype
```

---

## Class Expressions vs Declarations

### Class Declaration

```javascript
class Animal {
  constructor(name) { this.name = name; }
}

// ❌ Hoist bo'lmaydi (TDZ):
const a = new MyClass(); // ReferenceError
class MyClass {}
```

### Class Expression

```javascript
// Anonymous:
const Animal = class {
  constructor(name) { this.name = name; }
};

// Named:
const Animal2 = class AnimalClass {
  constructor(name) { this.name = name; }
  whoAmI() {
    return AnimalClass.name; // Faqat class ICHIDAN ko'rinadi
  }
};

console.log(Animal2.name);      // "AnimalClass"
console.log(typeof AnimalClass); // "undefined" — tashqaridan ko'rinmaydi
```

---

## Inheritance: extends va super

### extends

`extends` orqali child class parent class'dan meros oladi:

```javascript
class Animal {
  constructor(name) {
    this.name = name;
    this.alive = true;
  }

  eat() {
    return `${this.name} yemoqda`;
  }

  sleep() {
    return `${this.name} uxlamoqda`;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);        // ← Animal.constructor chaqirish SHART
    this.breed = breed;
  }

  bark() {
    return `${this.name} havlayapti!`;
  }
}

const rex = new Dog("Rex", "German Shepherd");
rex.eat();   // "Rex yemoqda" — Animal'dan meros
rex.bark();  // "Rex havlayapti!" — Dog'ning o'zi
rex.sleep(); // "Rex uxlamoqda" — Animal'dan meros
```

### Prototype Chain

```
rex
├── name: "Rex"
├── breed: "German Shepherd"
├── alive: true
└── __proto__ → Dog.prototype
                ├── bark()
                ├── constructor: Dog
                └── __proto__ → Animal.prototype
                                ├── eat()
                                ├── sleep()
                                ├── constructor: Animal
                                └── __proto__ → Object.prototype
```

### super() Constructor'da

`extends` ishlatganda, child constructor'da `super()` chaqirish **MAJBURIY** — `this` ishlatishdan **oldin**:

```javascript
class Child extends Parent {
  constructor(x, y) {
    // ❌ this.x = x; // ReferenceError — super() hali chaqirilmagan!
    super(x);          // ✅ avval super
    this.y = y;        // ✅ keyin this
  }
}
```

**Nima uchun?** `extends` da `this` object'ni **parent constructor** yaratadi. `super()` chaqirilmaguncha `this` mavjud emas.

Agar child'da constructor yozmasangiz — avtomatik qo'shiladi:

```javascript
class Dog extends Animal {
  // constructor yo'q = implicit:
  // constructor(...args) { super(...args); }
}
```

### super.method() — Parent Method Chaqirish

```javascript
class Animal {
  speak() {
    return `${this.name} ovoz chiqaryapti`;
  }
}

class Dog extends Animal {
  speak() {
    const parentResult = super.speak(); // Animal.speak()
    return `${parentResult}: Hav-hav!`;
  }
}

const rex = new Dog();
rex.name = "Rex";
rex.speak(); // "Rex ovoz chiqaryapti: Hav-hav!"
```

### Ko'p Darajali Inheritance

```javascript
class Animal {
  eat() { return "yeyapti"; }
}

class Dog extends Animal {
  bark() { return "havlayapti"; }
}

class GuideDog extends Dog {
  guide() { return "boshqaryapti"; }
}

const buddy = new GuideDog();
buddy.eat();   // ✅ Animal'dan
buddy.bark();  // ✅ Dog'dan
buddy.guide(); // ✅ GuideDog'dan

// Chain: buddy → GuideDog.prototype → Dog.prototype → Animal.prototype → Object.prototype
```

---

## Private Fields va Methods

### Private Fields (#)

ES2022 da **haqiqiy private** field'lar qo'shildi — `#` prefiksi bilan:

```javascript
class BankAccount {
  #balance;       // private field declaration
  #pin;

  constructor(owner, initialBalance, pin) {
    this.owner = owner;           // public
    this.#balance = initialBalance; // private
    this.#pin = pin;              // private
  }

  #verifyPin(pin) {              // private method
    return this.#pin === pin;
  }

  withdraw(amount, pin) {
    if (!this.#verifyPin(pin)) throw new Error("Noto'g'ri PIN");
    if (amount > this.#balance) throw new Error("Mablag' yetarli emas");
    this.#balance -= amount;
    return this.#balance;
  }

  getBalance(pin) {
    if (!this.#verifyPin(pin)) throw new Error("Noto'g'ri PIN");
    return this.#balance;
  }
}

const acc = new BankAccount("Ali", 5000, "1234");
acc.getBalance("1234");       // 5000
acc.withdraw(1000, "1234");   // 4000

acc.#balance;                  // ❌ SyntaxError — private!
acc.#verifyPin("1234");        // ❌ SyntaxError — private!
acc["#balance"];               // undefined — bu boshqa narsa
```

### Private vs Closure Privacy

```javascript
// Closure privacy (eski usul):
function createAccount(balance) {
  return {
    getBalance() { return balance; }
  };
}
// ✅ Private, lekin: typeof, instanceof ishlamaydi, performance past

// # Private (yangi usul):
class Account {
  #balance;
  constructor(b) { this.#balance = b; }
  getBalance() { return this.#balance; }
}
// ✅ Private, class ecosystem bilan integrate, spec-level enforcement
```

| | `#` Private | Closure | `_` Convention |
|-|-------------|---------|---------------|
| **Haqiqiy private** | ✅ | ✅ | ❌ (faqat convention) |
| **instanceof** | ✅ | ❌ | ✅ |
| **Performance** | Tez | O'rta | Tez |
| **Inheritance** | Child ko'rmaydi | Scope'ga bog'liq | Ko'radi |
| **DevTools** | Ko'rinadi | Ko'rinmaydi | Ko'rinadi |

### Private va Inheritance

```javascript
class Parent {
  #secret = 42;

  getSecret() {
    return this.#secret;
  }
}

class Child extends Parent {
  tryAccess() {
    // return this.#secret; // ❌ SyntaxError — child ko'rmaydi!
    return this.getSecret(); // ✅ public method orqali
  }
}

const c = new Child();
c.tryAccess(); // 42 — public getter orqali
```

---

## Public Class Fields

```javascript
class Config {
  // Public instance fields — har instance'da alohida
  host = "localhost";
  port = 3000;
  debug = false;

  // Computed / dynamic default
  createdAt = Date.now();

  // Arrow function field — this avtomatik bind
  handleClick = () => {
    console.log(this.host); // ✅ this = instance (doim)
  };

  constructor(overrides = {}) {
    Object.assign(this, overrides);
  }
}

const c = new Config({ port: 8080 });
console.log(c.host); // "localhost"
console.log(c.port); // 8080 — override bo'ldi
```

### Muhim: Arrow Function Field = Instance Method

```javascript
class Button {
  label = "Click";
  
  // Arrow field — har instance uchun YANGI funksiya (instance method)
  onClick = () => {
    console.log(this.label);
  };

  // Oddiy method — prototype'da, BITTA funksiya (shared)
  onHover() {
    console.log(this.label);
  }
}

const b1 = new Button();
const b2 = new Button();
b1.onClick === b2.onClick;   // false — har birida alohida
b1.onHover === b2.onHover;   // true — prototype'da bitta
```

Arrow field qachon kerak? **Event handler** sifatida berishda — `this` yo'qolmasligi uchun:

```javascript
const btn = new Button();
document.addEventListener("click", btn.onClick);   // ✅ this = btn
document.addEventListener("hover", btn.onHover);   // ❌ this = document!
```

---

## Static Properties va Methods

### Nazariya

`static` keyword — bu class'ning **o'ziga** (constructor funksiyasiga) tegishli property va method'larni belgilash uchun ishlatiladi. Static a'zolar instance'larda mavjud emas — ular `new` qilmasdan to'g'ridan-to'g'ri class orqali chaqiriladi.

Buni **fabrika binosi** ga o'xshatish mumkin: fabrika (class) ning o'zi "bu fabrikada nechta mahsulot ishlab chiqarilgan" degan hisobni yuritadi (static property), lekin har bir mahsulot (instance) bu hisob haqida bilmaydi. `Math.random()`, `Date.now()`, `Array.isArray()` — bularning barchasi static method'lar.

Static method'lar qachon kerak? Utility/helper funksiyalar (masalan, `MathUtils.clamp()`), factory method'lar (masalan, `User.fromJSON()`), singleton pattern, va konfiguratsiya konstantalari uchun. Muhim: static method ichida `this` class'ning o'ziga ishora qiladi (instance'ga emas), va static a'zolar ham `extends` orqali meros bo'ladi.

```javascript
class MathUtils {
  static PI = 3.14159;

  static square(x) {
    return x * x;
  }

  static clamp(value, min, max) {
    return Math.min(Math.max(value, min), max);
  }
}

MathUtils.PI;            // 3.14159
MathUtils.square(5);     // 25
MathUtils.clamp(15, 0, 10); // 10

const m = new MathUtils();
m.square(5); // ❌ TypeError — static method instance'da yo'q
```

### Factory Pattern bilan Static

```javascript
class User {
  constructor(name, role) {
    this.name = name;
    this.role = role;
  }

  // Factory methods:
  static createAdmin(name) {
    return new User(name, "admin");
  }

  static createGuest() {
    return new User("Guest", "guest");
  }

  static fromJSON(json) {
    const data = JSON.parse(json);
    return new User(data.name, data.role);
  }
}

const admin = User.createAdmin("Ali");
const guest = User.createGuest();
const fromApi = User.fromJSON('{"name":"Vali","role":"editor"}');
```

### Static Inheritance

```javascript
class Animal {
  static category = "Living";
  static describe() { return `Category: ${this.category}`; }
}

class Dog extends Animal {
  static category = "Pet";
}

Animal.describe(); // "Category: Living"
Dog.describe();    // "Category: Pet" — static HAM meros bo'ladi!
```

---

## instanceof bilan Classes

```javascript
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

const rex = new Dog();

rex instanceof Dog;    // true
rex instanceof Animal; // true
rex instanceof Cat;    // false
rex instanceof Object; // true

// Symbol.hasInstance bilan customize:
class Even {
  static [Symbol.hasInstance](num) {
    return typeof num === "number" && num % 2 === 0;
  }
}

4 instanceof Even;  // true
3 instanceof Even;  // false
```

---

## Classes vs Prototypes — Detallar

### Class Transpile Qilganda

Babel class'ni ES5 ga qanday aylantiradi:

```javascript
// ES6 Class:
class User {
  constructor(name) {
    this.name = name;
  }
  greet() {
    return `Salom, ${this.name}`;
  }
  static create(name) {
    return new User(name);
  }
}

// ES5 ga transpile:
"use strict";

function User(name) {
  // new siz chaqirishni taqiqlash:
  if (!(this instanceof User)) {
    throw new TypeError("Cannot call a class as a function");
  }
  this.name = name;
}

// Instance method — prototype'da, non-enumerable
Object.defineProperty(User.prototype, "greet", {
  value: function greet() {
    return "Salom, " + this.name;
  },
  writable: true,
  configurable: true,
  enumerable: false    // ← class method'lari non-enumerable!
});

// Static method — constructor'da
Object.defineProperty(User, "create", {
  value: function create(name) {
    return new User(name);
  },
  writable: true,
  configurable: true,
  enumerable: false
});
```

---

## Mixins

### Nazariya

JavaScript **single inheritance** tildir — class faqat **bitta** class'dan `extends` qila oladi. Lekin real-world dasturlashda ko'pincha bitta ob'ektga bir nechta mustaqil "qobiliyat" berish kerak bo'ladi: masalan, ob'ekt ham serializable, ham validatable, ham timestamped bo'lishi kerak. Bu muammoning yechimi — **Mixin** pattern.

Mixin — bu class'ga qo'shimcha funksionallik qo'shadigan funksiya. JavaScript'da mixin'lar odatda **higher-order function** sifatida amalga oshiriladi: ular Base class'ni qabul qiladi va yangi class qaytaradi. Bu pattern aslida **function composition** ga asoslanadi va u juda moslashuvchan: istalgan kombinatsiyada istalgan qobiliyatlarni birlashtirish mumkin.

Mixin'lar zamonaviy JavaScript ekotizimida keng tarqalgan. React'ning eski versiyalarida `React.createClass` bilan mixin'lar ishlatilgan. LitElement (Web Components) va boshqa framework'larda hozir ham mixin pattern faol ishlatiladi. TypeScript'da esa `Mixin` pattern rasmiy documentation'da tavsiya etilgan.

### Mixin Pattern

```javascript
// Mixin'lar — oddiy funksiyalar:
const Serializable = (Base) => class extends Base {
  serialize() {
    return JSON.stringify(this);
  }

  static deserialize(json) {
    return Object.assign(new this(), JSON.parse(json));
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

const Timestamped = (Base) => class extends Base {
  constructor(...args) {
    super(...args);
    this.createdAt = new Date();
    this.updatedAt = new Date();
  }

  touch() {
    this.updatedAt = new Date();
  }
};

// Ishlatish — chain qilish:
class BaseUser {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

class User extends Serializable(Validatable(Timestamped(BaseUser))) {
  // User → Serializable → Validatable → Timestamped → BaseUser
}

const user = new User("Ali", "ali@mail.com");
user.validate();       // true ✅
user.serialize();      // '{"name":"Ali","email":"ali@mail.com","createdAt":...}'
user.touch();          // updatedAt yangilandi
user.createdAt;        // Date object ✅
```

### Prototype Chain

```
user → User.prototype
       → Serializable(mix).prototype
         → Validatable(mix).prototype
           → Timestamped(mix).prototype
             → BaseUser.prototype
               → Object.prototype
```

---

## Composition vs Inheritance

### Muammo: Gorilla-Banana

> "Siz faqat banan xohladingiz, lekin gorilla bilan butun o'rmonni oldingiz."

Inheritance'da child parent'ning **hamma narsasini** oladi — kerak bo'lmasa ham.

### Inheritance (Tight Coupling)

```javascript
// ❌ Chuqur inheritance — fragile, rigid
class Animal {
  eat() { /* ... */ }
  sleep() { /* ... */ }
}

class FlyingAnimal extends Animal {
  fly() { /* ... */ }
}

class SwimmingAnimal extends Animal {
  swim() { /* ... */ }
}

// Uchib suzadigan hayvon kerak — MUAMMO! (JS da multiple extends yo'q)
// class Duck extends FlyingAnimal, SwimmingAnimal {} // ❌ SyntaxError!
```

### Composition (Loose Coupling)

```javascript
// ✅ Composition — qobiliyatlarni birlashtirish
const canEat = (state) => ({
  eat() { return `${state.name} yemoqda`; }
});

const canSleep = (state) => ({
  sleep() { return `${state.name} uxlamoqda`; }
});

const canFly = (state) => ({
  fly() { return `${state.name} uchmoqda`; }
});

const canSwim = (state) => ({
  swim() { return `${state.name} suzmoqda`; }
});

// Duck — uchi ham, suza ham oladi:
function createDuck(name) {
  const state = { name };
  return {
    ...state,
    ...canEat(state),
    ...canSleep(state),
    ...canFly(state),
    ...canSwim(state)
  };
}

const duck = createDuck("Donald");
duck.eat();   // "Donald yemoqda"
duck.fly();   // "Donald uchmoqda"
duck.swim();  // "Donald suzmoqda"
```

### Qachon Qaysi Biri?

| | Inheritance | Composition |
|-|-------------|-------------|
| **Qachon** | "is-a" munosabat (Dog IS Animal) | "has-a" / "can-do" munosabat |
| **Hierarxiya** | Aniq, chuqur emas (2-3 daraja max) | Flat, moslashuvchan |
| **Reuse** | Vertical (parent → child) | Horizontal (mix & match) |
| **Coupling** | Tight (parent o'zgarsa, child buziladi) | Loose |

**Tavsiya:** "Prefer composition over inheritance" — lekin dogmatik bo'lmang. 1-2 darajali inheritance to'g'ri holatlarda juda yaxshi ishlaydi.

---

## Common Mistakes

### ❌ Xato 1: super() ni unutish

```javascript
class Animal {
  constructor(name) { this.name = name; }
}

class Dog extends Animal {
  constructor(name, breed) {
    this.breed = breed; // ❌ ReferenceError: Must call super constructor
    // super(name); kerak edi!
  }
}
```

### ✅ To'g'ri usul:

```javascript
class Dog extends Animal {
  constructor(name, breed) {
    super(name);         // ✅ avval super
    this.breed = breed;  // ✅ keyin this
  }
}
```

**Nima uchun:** `extends` da `this` ni parent constructor yaratadi. `super()` chaqirilmaguncha `this` mavjud emas.

---

### ❌ Xato 2: Method ni callback sifatida berganda this yo'qolishi

```javascript
class Timer {
  seconds = 0;

  tick() {
    this.seconds++;           // this = ??? 
    console.log(this.seconds);
  }

  start() {
    setInterval(this.tick, 1000); // ❌ this = window/undefined
  }
}
```

### ✅ To'g'ri usul:

```javascript
class Timer {
  seconds = 0;

  // Variant 1: Arrow field (tavsiya)
  tick = () => {
    this.seconds++;
    console.log(this.seconds);
  };

  // Variant 2: bind
  start() {
    setInterval(this.tick.bind(this), 1000);
  }

  // Variant 3: arrow wrapper
  start2() {
    setInterval(() => this.tick(), 1000);
  }
}
```

**Nima uchun:** Method callback sifatida berilganda `this` konteksti yo'qoladi. Arrow function yoki `.bind()` buni hal qiladi. To'liq [10-this-keyword.md](10-this-keyword.md) da.

---

### ❌ Xato 3: Class hoist bo'ladi deb o'ylash

```javascript
const user = new User("Ali"); // ❌ ReferenceError

class User {
  constructor(name) { this.name = name; }
}
```

### ✅ To'g'ri usul:

```javascript
class User {
  constructor(name) { this.name = name; }
}

const user = new User("Ali"); // ✅
```

**Nima uchun:** Class **hoist bo'ladi** lekin **TDZ** da — xuddi `let`/`const` kabi. E'londan oldin kirish — ReferenceError.

---

### ❌ Xato 4: Private field ga subclass'dan kirish

```javascript
class Parent {
  #data = 42;
}

class Child extends Parent {
  getData() {
    return this.#data; // ❌ SyntaxError — private parent'da!
  }
}
```

### ✅ To'g'ri usul:

```javascript
class Parent {
  #data = 42;

  // Protected-like — public getter orqali
  getData() {
    return this.#data;
  }

  // Yoki protected pattern:
  _getDataForSubclass() {
    return this.#data;
  }
}

class Child extends Parent {
  getData() {
    return super.getData(); // ✅ public method orqali
  }
}
```

**Nima uchun:** `#` private — faqat **o'sha class body** ichidan ko'rinadi. Subclass ham ko'rmaydi. JS da `protected` keyword yo'q — convention (`_` prefix) yoki public getter ishlatiladi.

---

### ❌ Xato 5: Constructor da arrow method qaytarish

```javascript
class User {
  constructor(name) {
    this.name = name;
    // ❌ constructor'da method qaytarish — har instance uchun yangi fn
    this.greet = () => `Salom, ${this.name}`;
  }
}
// 1000 ta User = 1000 ta greet funksiya (xotira isrof)
```

### ✅ To'g'ri usul:

```javascript
class User {
  constructor(name) {
    this.name = name;
  }
  // ✅ prototype'da — bitta funksiya, hammasi share
  greet() {
    return `Salom, ${this.name}`;
  }
}
// 1000 ta User = 1 ta greet funksiya

// Faqat callback kerak bo'lganda arrow field:
class Button {
  label = "ok";
  onClick = () => console.log(this.label); // callback uchun kerak
}
```

**Nima uchun:** Arrow field/constructor method = har instance uchun yangi funksiya = ko'p xotira. Prototype method = bitta, shared = tejamkor. Arrow faqat this binding kerak joyda.

---

## Amaliy Mashqlar

### Mashq 1: Oddiy Class (Oson)

**Savol:** `Rectangle` class yarating: `width`, `height`, `area()` getter, `perimeter()` method. `Square extends Rectangle` — faqat `side` qabul qiladi.

<details>
<summary>Javob</summary>

```javascript
class Rectangle {
  constructor(width, height) {
    this.width = width;
    this.height = height;
  }

  get area() {
    return this.width * this.height;
  }

  perimeter() {
    return 2 * (this.width + this.height);
  }

  toString() {
    return `Rectangle(${this.width}x${this.height})`;
  }
}

class Square extends Rectangle {
  constructor(side) {
    super(side, side);
  }

  toString() {
    return `Square(${this.width})`;
  }
}

const rect = new Rectangle(5, 3);
rect.area;        // 15
rect.perimeter();  // 16

const sq = new Square(4);
sq.area;           // 16
sq.perimeter();    // 16
sq instanceof Square;    // true
sq instanceof Rectangle; // true
```
</details>

---

### Mashq 2: Private + Validation (O'rta)

**Savol:** `Temperature` class: #celsius private, getter/setter bilan Fahrenheit va Celsius, validation bilan.

<details>
<summary>Javob</summary>

```javascript
class Temperature {
  #celsius;

  constructor(celsius) {
    this.celsius = celsius; // setter orqali — validation ishlaydi
  }

  get celsius() {
    return this.#celsius;
  }

  set celsius(value) {
    if (typeof value !== "number" || isNaN(value)) {
      throw new TypeError("Raqam bo'lishi kerak");
    }
    if (value < -273.15) {
      throw new RangeError("Absolyut noldan past bo'lishi mumkin emas");
    }
    this.#celsius = value;
  }

  get fahrenheit() {
    return this.#celsius * 9/5 + 32;
  }

  set fahrenheit(f) {
    this.celsius = (f - 32) * 5/9; // setter orqali — validation
  }

  toString() {
    return `${this.#celsius.toFixed(1)}°C / ${this.fahrenheit.toFixed(1)}°F`;
  }

  static fromFahrenheit(f) {
    return new Temperature((f - 32) * 5/9);
  }
}

const t = new Temperature(100);
t.toString();        // "100.0°C / 212.0°F"
t.fahrenheit = 32;
t.celsius;            // 0
t.celsius = -300;     // ❌ RangeError

const t2 = Temperature.fromFahrenheit(98.6);
t2.celsius;           // 37
```
</details>

---

### Mashq 3: EventEmitter Class (O'rta)

**Savol:** Oddiy `EventEmitter` class yarating: `on(event, fn)`, `off(event, fn)`, `emit(event, ...args)`, `once(event, fn)`.

<details>
<summary>Javob</summary>

```javascript
class EventEmitter {
  #listeners = new Map();

  on(event, fn) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, []);
    }
    this.#listeners.get(event).push(fn);
    return this; // chaining uchun
  }

  off(event, fn) {
    const fns = this.#listeners.get(event);
    if (!fns) return this;
    this.#listeners.set(event, fns.filter(f => f !== fn && f._original !== fn));
    return this;
  }

  emit(event, ...args) {
    const fns = this.#listeners.get(event);
    if (!fns) return false;
    fns.forEach(fn => fn(...args));
    return true;
  }

  once(event, fn) {
    const wrapper = (...args) => {
      fn(...args);
      this.off(event, wrapper);
    };
    wrapper._original = fn; // off() da topish uchun
    return this.on(event, wrapper);
  }
}

// Test:
const emitter = new EventEmitter();

emitter.on("data", (msg) => console.log("A:", msg));
emitter.once("data", (msg) => console.log("B (once):", msg));

emitter.emit("data", "salom");
// A: salom
// B (once): salom

emitter.emit("data", "dunyo");
// A: dunyo
// (B endi yo'q — once edi)
```
</details>

---

### Mashq 4: Mixin Yarating (Qiyin)

**Savol:** `Loggable`, `Cacheable` mixin'lar yozing. Class ikkala qobiliyatga ega bo'lsin.

<details>
<summary>Javob</summary>

```javascript
const Loggable = (Base) => class extends Base {
  log(message) {
    console.log(`[${this.constructor.name}] ${message}`);
  }

  logMethod(methodName, ...args) {
    this.log(`${methodName}(${args.join(", ")})`);
  }
};

const Cacheable = (Base) => class extends Base {
  #cache = new Map();

  getCached(key, computeFn) {
    if (this.#cache.has(key)) {
      return this.#cache.get(key);
    }
    const value = computeFn();
    this.#cache.set(key, value);
    return value;
  }

  clearCache() {
    this.#cache.clear();
  }
};

class DataService extends Loggable(Cacheable(class {})) {
  fetchUser(id) {
    return this.getCached(`user:${id}`, () => {
      this.log(`Fetching user ${id}...`);
      return { id, name: `User ${id}` };
    });
  }
}

const service = new DataService();
service.fetchUser(1); // [DataService] Fetching user 1...  → { id: 1, name: "User 1" }
service.fetchUser(1); // Cache'dan — log yo'q
service.fetchUser(2); // [DataService] Fetching user 2...
```
</details>

---

### Mashq 5: LinkedList Class (Qiyin)

**Savol:** `LinkedList` class yarating: `push`, `pop`, `get(index)`, `[Symbol.iterator]` (for...of uchun).

<details>
<summary>Javob</summary>

```javascript
class Node {
  constructor(value) {
    this.value = value;
    this.next = null;
  }
}

class LinkedList {
  #head = null;
  #size = 0;

  get size() { return this.#size; }

  push(value) {
    const node = new Node(value);
    if (!this.#head) {
      this.#head = node;
    } else {
      let current = this.#head;
      while (current.next) current = current.next;
      current.next = node;
    }
    this.#size++;
    return this;
  }

  pop() {
    if (!this.#head) return undefined;
    if (!this.#head.next) {
      const val = this.#head.value;
      this.#head = null;
      this.#size--;
      return val;
    }
    let current = this.#head;
    while (current.next.next) current = current.next;
    const val = current.next.value;
    current.next = null;
    this.#size--;
    return val;
  }

  get(index) {
    if (index < 0 || index >= this.#size) return undefined;
    let current = this.#head;
    for (let i = 0; i < index; i++) current = current.next;
    return current.value;
  }

  *[Symbol.iterator]() {
    let current = this.#head;
    while (current) {
      yield current.value;
      current = current.next;
    }
  }

  toString() {
    return [...this].join(" → ");
  }
}

const list = new LinkedList();
list.push(1).push(2).push(3);
list.size;     // 3
list.get(1);   // 2
list.pop();    // 3
list.toString(); // "1 → 2"

for (const val of list) {
  console.log(val); // 1, 2
}
```
</details>

---

## Xulosa

1. **Class = syntactic sugar** — ichida prototype ishlaydi. Lekin qo'shimcha himoyalar bor: TDZ, `new` majburiy, strict mode, non-enumerable methods.

2. **Class anatomy:** constructor, instance method (prototype), static method/property, getter/setter, public field, private `#` field/method.

3. **`extends`** — prototype chain quradi. **`super()`** — parent constructor chaqirish (majburiy, `this` dan oldin). **`super.method()`** — parent method.

4. **Private `#`** — haqiqiy encapsulation. Faqat class body ichidan ko'rinadi, subclass ham ko'rmaydi.

5. **Static** — class'ning o'zida, instance'da emas. Factory method, utility, constant uchun.

6. **Transpile:** class → constructor function + `Object.defineProperty` (non-enumerable methods) + strict checks.

7. **Mixins** — multiple inheritance muammosini hal qiladi. Funksiya class qabul qilib, kengaytirib qaytaradi.

8. **Composition > Inheritance** — moslashuvchan, loose coupling. Lekin 1-2 darajali inheritance o'z joyida to'g'ri.

---

> **Keyingi bo'lim:** [09-functions.md](09-functions.md) — Functions First-Class Citizens — declaration vs expression vs arrow, IIFE, currying, memoization, debounce/throttle.
