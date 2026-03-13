# Bo'lim 8: ES6 Classes

> ES6 `class` sintaksisi — prototypal inheritance ustiga qurilgan syntactic sugar. Ichida xuddi constructor function + prototype mexanizmi ishlaydi, lekin yozish va o'qish ancha qulay. Bu bo'limda class anatomy, inheritance (`extends`/`super`), private fields, static members, accessor'lar, mixins va composition vs inheritance chuqur yoritiladi.

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
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Class — Syntactic Sugar Over Prototypes

### Nazariya

ES6 da kiritilgan `class` sintaksisi — bu [07-prototypes.md](07-prototypes.md) da o'rgangan constructor function + prototype pattern'ining **yangi yozuv usuli**. Engine ichida class bilan yaratilgan kod xuddi shu prototype mexanizmiga aylanadi. Ya'ni `class` yangi meros tizimi emas — u mavjud tizimga qulay sintaksis beradi.

Lekin `class` faqat "go'zal yozuv" emas — u bir nechta muhim farqlarga ega:
1. `class` ichidagi barcha method'lar **non-enumerable** (`enumerable: false`) — constructor function'da esa default `true`
2. `class` ichidagi barcha kod avtomatik **strict mode** da ishlaydi
3. `class` ni `new` siz chaqirib bo'lmaydi — `TypeError` beradi
4. `class` **hoisting** qilinmaydi (TDZ da turadi) — function declaration esa to'liq hoist bo'ladi
5. `class` method'larida `[[Construct]]` internal slot yo'q — ya'ni method'larni `new` bilan chaqirib bo'lmaydi

### Under the Hood

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

### Kod Misollari

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

### Kod Misollari

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

---

## Class Declarations vs Expressions

### Nazariya

Class'ni ikki usulda yaratish mumkin — declaration va expression. Farqi function declaration va function expression bilan bir xil pattern'ga amal qiladi, lekin **hoisting** jihatidan ikkalasi ham **TDZ** da turadi.

### Kod Misollari

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

---

## Class Hoisting — TDZ

### Nazariya

Class declaration va class expression ikkalasi ham **hoisting** qilinadi, lekin **Temporal Dead Zone (TDZ)** da turadi — xuddi `let` va `const` kabi. Bu `function` declaration dan muhim farq: function declaration to'liq hoist bo'ladi (e'lon qilishdan oldin chaqirish mumkin), class esa e'londan oldin ishlatib bo'lmaydi.

Bu farq nima uchun bor? Class'da `extends` bilan inheritance bo'lishi mumkin — agar class hoist bo'lsa va parent class hali yaratilmagan bo'lsa, noto'g'ri prototype chain quriladi. TDZ bu muammoning oldini oladi.

### Kod Misollari

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

// Function da esa typeof ishlaydi:
console.log(typeof Foo); // "undefined" (hoist bo'lgan, lekin...)
function Foo() {}
console.log(typeof Foo); // "function"
```

---

## Inheritance: extends va super

### Nazariya

`extends` keyword — child class parent class'dan meros olishini bildiradi. Ichida engine quyidagilarni bajaradi:
1. `Child.prototype.[[Prototype]]` = `Parent.prototype` (method meros)
2. `Child.[[Prototype]]` = `Parent` (static method meros)

`super` keyword ikki kontekstda ishlatiladi:
- **`super()`** — constructor ichida. Parent constructor'ni chaqiradi. `extends` ishlatilgan class'da `this` ga murojaat qilishdan **OLDIN** `super()` chaqirilishi **SHART**.
- **`super.method()`** — method ichida. Parent'ning prototype method'ini chaqiradi.

### Under the Hood

`extends` engine ichida nima qiladi:

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} speaks`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);    // Animal.call(this, name) ga teng
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

const rex = new Dog("Rex");
// super orqali speak() oldin yozilmagan bo'lsa ham ishlaydi,
// chunki Dog constructorsiz default constructor super() chaqiradi
rex.name = "Rex";
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

### Under the Hood

V8 engine private field'larni internal **WeakMap-like** tuzilmada saqlaydi. Har bir class uchun "brand" (maxsus identifikator) yaratiladi. Private field'ga murojaat qilinganida engine avval object'ning brand'ini tekshiradi — agar mos kelmasa, `TypeError` beradi.

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

### Kod Misollari

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

### Kod Misollari

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

### Kod Misollari

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

---

## Accessor — Getter va Setter

### Nazariya

`get` va `set` keyword'lar class ichida computed property yaratish imkonini beradi. Tashqaridan oddiy property kabi ko'rinadi, lekin ichida funksiya ishlaydi. Bu [06-objects.md](06-objects.md) da o'rgangan getter/setter bilan bir xil mexanizm — faqat class sintaksisi bilan.

Getter/setter nima uchun kerak? Validation (qiymat o'rnatishda tekshirish), computed property (boshqa field'lardan hisoblash), va encapsulation (ichki formatni yashirish).

### Kod Misollari

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

---

## instanceof Bilan Classes

### Nazariya

`instanceof` class bilan ham xuddi constructor function bilan bir xil ishlaydi — prototype chain bo'ylab `Class.prototype` ni qidiradi. Class inheritance'da (`extends`) child instance parent uchun ham `true` qaytaradi.

### Kod Misollari

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

---

## Classes vs Prototypes — Ichki Farq

### Nazariya

`class` va constructor function o'rtasidagi farqlar faqat sintaktik emas — bir nechta muhim behavior farqlari bor:

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

### Kod Misollari

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

### Kod Misollari

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
class User extends Serializable(Timestamped(Validatable(Object))) {
  constructor(name, email) {
    super();
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

### Qachon Inheritance?

- Haqiqiy "is-a" munosabat bor: `Dog extends Animal`, `Circle extends Shape`
- Parent class barqaror va kamdan-kam o'zgaradi
- Child class parent'ning **barcha** xulq-atvorini meros olishi kerak

### Qachon Composition?

- "has-a" yoki "can-do" munosabat bor: `Car` da `Engine` bor, `User` `Serializable`
- Turli xil capability'larni aralashtirish kerak
- Runtime da xulq-atvorni o'zgartirish kerak

### Kod Misollari

```javascript
// ❌ Inheritance orqali — rigid, tight coupling:
class FlyingSwimmingDuck extends FlyingAnimal {
  // Agar SwimmingAnimal dan ham meros olish kerak bo'lsa?
  // JavaScript da multiple inheritance yo'q!
}

// ✅ Composition orqali — flexible, loose coupling:
const canFly = (state) => ({
  fly() { return `${state.name} uchmoqda`; }
});

const canSwim = (state) => ({
  swim() { return `${state.name} suzmoqda`; }
});

const canWalk = (state) => ({
  walk() { return `${state.name} yurmoqda`; }
});

function createDuck(name) {
  const state = { name };
  return {
    ...state,
    ...canFly(state),
    ...canSwim(state),
    ...canWalk(state)
  };
}

function createPenguin(name) {
  const state = { name };
  return {
    ...state,
    ...canSwim(state),
    ...canWalk(state)
    // canFly yo'q — penguin uchmayd!
  };
}

const duck = createDuck("Donald");
console.log(duck.fly());  // "Donald uchmoqda"
console.log(duck.swim()); // "Donald suzmoqda"

const penguin = createPenguin("Tux");
console.log(penguin.swim()); // "Tux suzmoqda"
// penguin.fly();             // ❌ undefined — uchmayd
```

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
setTimeout(timer.increment, 1000); // NaN — this = undefined
document.addEventListener("click", timer.increment); // NaN
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

### ❌ Xato 5: Class field'da this orqali boshqa field'ga murojaat

```javascript
class Config {
  baseUrl = "https://api.example.com";
  // ❌ Bu ISHLAMAYDI — field'lar tartib bo'yicha evaluate bo'ladi,
  // lekin this.baseUrl hali mavjud emas degan xato bermaydi,
  // chunki class field'lar constructor bilan bir vaqtda ishlaydi:
  apiUrl = `${this.baseUrl}/v1`; // ✅ aslida ishlaydi!
}

// Lekin static field'larda ehtiyot:
class App {
  static version = "1.0";
  // static fullVersion = `App v${this.version}`; // ✅ ishlaydi
  // Tartibga bog'liq — tepadagi field oldin evaluate bo'ladi
}
```

**Qoida:** Field'lar tartib bo'yicha (yuqoridan pastga) evaluate bo'ladi. Birinchi field ichida ikkinchi field'ga murojaat qilish mumkin emas (chunki hali yaratilmagan).

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

10. **Composition > Inheritance** — "has-a" munosabat uchun composition, "is-a" uchun inheritance. Composition loose coupling beradi.

---

> **Keyingi bo'lim:** [09-functions.md](09-functions.md) — Functions First-Class Citizens: function declaration vs expression vs arrow, HOF, currying, partial application, memoization, debounce/throttle, IIFE, pure functions.
