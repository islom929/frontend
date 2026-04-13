# ES6 Classes — Interview Savollari

> ES6 class sintaksisi, inheritance (`extends`/`super`), private fields, static members, accessor'lar, mixins va composition vs inheritance haqida interview savollari.

---

## Nazariy savollar

### 1. Class nima va prototype bilan qanday bog'liq? [Junior+]

<details>
<summary>Javob</summary>

ES6 `class` — prototypal inheritance ustiga qurilgan **syntactic sugar**. Engine ichida class xuddi constructor function + prototype mexanizmi sifatida ishlaydi:

```javascript
class User {
  constructor(name) { this.name = name; }
  greet() { return `Hi, ${this.name}`; }
}

// typeof — function:
console.log(typeof User); // "function"

// Method prototype'da:
console.log(User.prototype.hasOwnProperty("greet")); // true

// Instance prototype chain:
const user = new User("Ali");
console.log(Object.getPrototypeOf(user) === User.prototype); // true
```

Lekin `class` faqat "yozuv qulayligi" emas — muhim farqlari bor: barcha method'lar `enumerable: false`, ichki kod **strict mode** da, `new` siz chaqirish **TypeError** beradi, va **TDZ** da turadi (hoist bo'ladi lekin declaration gacha ishlatib bo'lmaydi).

</details>

### 2. Class va constructor function o'rtasidagi farqlar nima? [Middle]

<details>
<summary>Javob</summary>

| Xususiyat | Constructor Function | Class |
|-----------|---------------------|-------|
| **Hoisting** | To'liq hoist | TDZ |
| **Strict mode** | Default sloppy | Har doim strict |
| **`new` siz** | Ishlaydi (xavfli) | TypeError |
| **Method enumerable** | `true` | `false` |
| **Method'ni `new` bilan** | Mumkin | TypeError |

```javascript
// 1. new siz chaqirish farqi:
function UserFn(name) { this.name = name; }
UserFn("Ali"); // ✅ ishlaydi — lekin this = globalThis (bug!)

class UserClass {
  constructor(name) { this.name = name; }
}
// UserClass("Ali"); // ❌ TypeError

// 2. Method enumerable farqi:
function PersonFn(name) { this.name = name; }
PersonFn.prototype.greet = function() {};

class PersonCls {
  constructor(name) { this.name = name; }
  greet() {}
}

for (const key in new PersonFn("Ali"))  console.log(key); // "name", "greet"
for (const key in new PersonCls("Ali")) console.log(key); // "name" — greet yo'q
```

</details>

### 3. `extends` va `super` qanday ishlaydi? [Middle]

<details>
<summary>Javob</summary>

`extends` — child class parent'dan meros olishini bildiradi. Engine ichida ikki prototype chain quradi:

1. `Child.prototype.[[Prototype]]` = `Parent.prototype` — instance method meros
2. `Child.[[Prototype]]` = `Parent` — static method meros

`super` ikki kontekstda ishlatiladi:
- **`super(args)`** constructor ichida — parent constructor chaqiradi
- **`super.method()`** method ichida — parent prototype method chaqiradi

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
  static kingdom() { return "Animalia"; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);     // Animal constructor chaqiriladi
    this.breed = breed;
  }
  speak() {
    return super.speak() + " — Hav!"; // parent method + qo'shimcha
  }
}

const rex = new Dog("Rex", "Labrador");
console.log(rex.speak()); // "Rex makes a sound — Hav!"
console.log(Dog.kingdom()); // "Animalia" — static meros

// Derived class'da super() majburiy:
// super() chaqirilmasdan this ishlatish → ReferenceError
```

</details>

### 4. Private fields (`#`) va underscore convention (`_`) farqi nima? [Middle]

<details>
<summary>Javob</summary>

`_` prefix — bu faqat **convention** (kelishuv), hech qanday himoya yo'q. `#` prefix — bu **til darajasidagi enforcement**, tashqi kodda kirish **SyntaxError** beradi.

```javascript
class UserA {
  _name; // convention — tashqaridan kira oladi
  constructor(name) { this._name = name; }
}
const a = new UserA("Ali");
console.log(a._name); // "Ali" — hech qanday himoya yo'q!

class UserB {
  #name; // hard private — tashqaridan kira olmaydi
  constructor(name) { this.#name = name; }
  getName() { return this.#name; }
}
const b = new UserB("Ali");
// console.log(b.#name); // ❌ SyntaxError
console.log(b.getName()); // "Ali" — faqat public method orqali
```

| | `_convention` | `#private` |
|-|--------------|-----------|
| **Himoya** | Yo'q (faqat kelishuv) | Til enforsementi |
| **Tashqaridan kirish** | Mumkin | SyntaxError |
| **Subclass kirishi** | Mumkin | ❌ Mumkin emas |
| **`Object.keys`** | Ko'rinadi | Ko'rinmaydi |
| **Reflection** | Mavjud | Yo'q |

</details>

### 5. Static method nima va qachon ishlatiladi? [Middle]

<details>
<summary>Javob</summary>

`static` method class'ning o'zida saqlanadi, instance'larda emas. Instance yaratmasdan chaqiriladi.

```javascript
class MathUtils {
  static clamp(value, min, max) {
    return Math.min(Math.max(value, min), max);
  }

  static randomBetween(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }
}

console.log(MathUtils.clamp(15, 0, 10)); // 10
console.log(MathUtils.randomBetween(1, 6)); // 1-6

// Instance'dan kirish mumkin emas:
const m = new MathUtils();
// m.clamp(15, 0, 10); // ❌ TypeError
```

**Asosiy use case'lar:**
1. **Factory method'lar** — `User.fromJSON(json)`, `User.createAdmin()`
2. **Utility funksiyalar** — `MathUtils.clamp()`, `Validator.isEmail()`
3. **Singleton** — `Config.getInstance()`
4. **Comparison** — `Product.compare(a, b)` — sort uchun

Static method'lar `extends` bilan meros bo'ladi:

```javascript
class Animal {
  constructor(name) { this.name = name; }
  static create(name) { return new this(name); }
  //                         ↑ "this" static method'da = chaqirilgan class'ning o'zi
}
class Dog extends Animal {}

const rex = Dog.create("Rex");
// Dog.create chaqirilganda this = Dog
// new this(name) → new Dog("Rex") → Animal constructor chaqiriladi (meros orqali)
// Natija: Dog instance, name = "Rex"

console.log(rex instanceof Dog);  // true ✅
console.log(rex.name);            // "Rex" ✅
```

Bu pattern — **polymorphic factory** — parent'da yozilgan factory method child class'lar uchun ham to'g'ri instance yaratadi (chunki `this` dinamik).

</details>

### 6. Class field'lar (public va arrow function) qanday ishlaydi? [Middle]

<details>
<summary>Javob</summary>

Public class field'lar — har instance'da **own property** sifatida yaratiladi (prototype'da emas):

```javascript
class Button {
  type = "button";    // public field → instance'da
  onClick = () => {}; // arrow function field → instance'da, this bind

  click() {}          // oddiy method → prototype'da
}

const btn = new Button();

// Field'lar instance'da:
console.log(btn.hasOwnProperty("type"));     // true
console.log(btn.hasOwnProperty("onClick"));  // true

// Method prototype'da:
console.log(btn.hasOwnProperty("click"));                // false
console.log(Button.prototype.hasOwnProperty("click"));   // true
```

Arrow function field — `this` ni avtomatik bind qiladi, lekin **har instance uchun alohida funksiya** yaratadi:

```javascript
class Timer {
  count = 0;
  // Arrow function — this bind, lekin har instance alohida fn:
  increment = () => { this.count++; };
}

const t1 = new Timer();
const t2 = new Timer();
console.log(t1.increment === t2.increment); // false — 2 ta alohida fn!

// Oddiy method — bitta fn, barcha share:
// t1.click === t2.click → true
```

**Tavsiya:** Arrow function field faqat `this` binding kerak bo'lganda (event handler) ishlating. Aks holda oddiy method yozing — memory tejaydi.

</details>

### 7. Getter va setter class'da qanday ishlaydi? [Middle]

<details>
<summary>Javob</summary>

`get`/`set` — computed property yaratadi. Tashqaridan oddiy property kabi ko'rinadi, ichida funksiya ishlaydi:

```javascript
class User {
  #firstName;
  #lastName;

  constructor(first, last) {
    this.#firstName = first;
    this.#lastName = last;
  }

  // Getter — o'qish:
  get fullName() {
    return `${this.#firstName} ${this.#lastName}`;
  }

  // Setter — yozish bilan validation:
  set fullName(value) {
    const parts = value.split(" ");
    if (parts.length < 2) throw new Error("Ism va familiya kiriting");
    this.#firstName = parts[0];
    this.#lastName = parts.slice(1).join(" ");
  }
}

const user = new User("Ali", "Valiyev");
console.log(user.fullName);        // "Ali Valiyev" — getter
user.fullName = "Vali Karimov";    // setter
console.log(user.fullName);        // "Vali Karimov"
// user.fullName = "Madonna";      // ❌ Error: Ism va familiya kiriting
```

Getter/setter prototype'da **accessor descriptor** sifatida saqlanadi:

```javascript
const desc = Object.getOwnPropertyDescriptor(User.prototype, "fullName");
console.log(typeof desc.get); // "function"
console.log(typeof desc.set); // "function"
console.log(desc.enumerable); // false — class method
```

</details>

### 8. Static initialization block nima? [Middle+]

<details>
<summary>Javob</summary>

ES2022 da qo'shilgan `static { }` — class yaratilganda bir marta ishlaydi. Murakkab static initialization uchun:

```javascript
class Config {
  static env;
  static debug;
  static apiUrl;

  static {
    Config.env = globalThis.process?.env?.NODE_ENV || "development";
    Config.debug = Config.env !== "production";
    Config.apiUrl = Config.debug
      ? "http://localhost:3000"
      : "https://api.example.com";
  }
}

console.log(Config.env);    // "development"
console.log(Config.debug);  // true
console.log(Config.apiUrl); // "http://localhost:3000"
```

Nima uchun kerak? Static field'da try/catch ishlatib bo'lmaydi, bir nechta o'zaro bog'liq field'larni initialize qilish qiyin. `static { }` bloki to'liq JavaScript kodi yozish imkonini beradi.

</details>

### 9. Mixin pattern class'larda qanday qilinadi? [Senior]

<details>
<summary>Javob</summary>

JavaScript single inheritance — bitta `extends`. Mixin — **subclass factory** pattern orqali ko'p capability qo'shish:

```javascript
// Mixin — funksiya bo'lib, class oladi va extend qilgan class qaytaradi:
const Serializable = (Base) => class extends Base {
  serialize() { return JSON.stringify(this); }
};

const Timestamped = (Base) => class extends Base {
  constructor(...args) {
    super(...args);
    this.createdAt = new Date();
  }
};

// Qo'llash — chaining:
class User extends Serializable(Timestamped(Object)) {
  constructor(name) {
    super();
    this.name = name;
  }
}

const user = new User("Ali");
console.log(user.serialize());  // '{"createdAt":"...","name":"Ali"}'
console.log(user.createdAt);    // Date object
```

**Muammolari:**
1. Naming collision — ikki mixin bir xil method nomi
2. Constructor tartib — super() chaqirilish ketma-ketligi muhim
3. Debugging — method qaysi mixin'dan kelganini aniqlash qiyin

**Deep Dive:**

Mixin pattern spec'da maxsus qo'llab-quvvatlanmaydi — bu oddiy `extends` expression'ning dynamic xususiyatidan foydalanadi. `class C extends Mixin(Base)` da `extends` operandi runtime'da evaluate qilinadigan expression. Engine `ClassDefinitionEvaluation` algoritmida `superclass` ni evaluate qiladi va uning `prototype` property'sini yangi class'ning `[[Prototype]]` chain'iga ulaydi. Shuning uchun mixin chain'da har bir mixin alohida class yaratadi va prototype chain'ga qo'shiladi.

</details>

### 10. Composition vs inheritance — qachon qaysi biri? [Senior]

<details>
<summary>Javob</summary>

**Inheritance** — "is-a" munosabat. `Dog extends Animal` — "Dog **bu** Animal".
**Composition** — "has-a" munosabat. `Car` da `Engine` **bor**.

```javascript
// ❌ Inheritance muammosi — rigid hierarchy:
class Animal { move() {} }
class FlyingAnimal extends Animal { fly() {} }
class SwimmingAnimal extends Animal { swim() {} }
// FlyingSwimmingAnimal? Multiple inheritance yo'q!

// ✅ Composition — flexible (closure-based):
const canFly = (state) => ({
  fly() { return `${state.name} uchmoqda`; }
});
const canSwim = (state) => ({
  swim() { return `${state.name} suzmoqda`; }
});

function createDuck(name) {
  const state = { name };
  return { ...state, ...canFly(state), ...canSwim(state) };
}

// ⚠️ Subtle closure behavior: method'lar `state.name` ni closure orqali tutadi,
// natija object'idagi `name` bilan bog'liq emas:
const duck = createDuck("Donald");
duck.name = "Daffy";       // object property o'zgardi
duck.fly();                // "Donald uchmoqda" — closure `state.name` o'zgarmadi!

// ✅ Alternative — `this` bilan composition (tashqi o'zgarish'ga sezgir):
const canFly2 = {
  fly() { return `${this.name} uchmoqda`; }
};
const canSwim2 = {
  swim() { return `${this.name} suzmoqda`; }
};

function createDuck2(name) {
  return Object.assign({ name }, canFly2, canSwim2);
}

const duck2 = createDuck2("Donald");
duck2.name = "Daffy";
duck2.fly();               // "Daffy uchmoqda" ✅ — `this.name` dinamik
```

**Qaysi usulni tanlash:** Closure-based ko'proq "private state" illyuziyasini beradi (o'zgarmas snapshot), `this`-based esa dinamik — real OOP class semantikasiga yaqin.

| Criteria | Inheritance | Composition |
|----------|------------|-------------|
| **Munosabat** | "is-a" | "has-a" / "can-do" |
| **Coupling** | Tight | Loose |
| **Flexibility** | Rigid hierarchy | Runtime o'zgartirish mumkin |
| **Reuse** | Single chain | Istalgan kombinatsiya |

**Qoida:** Default'da composition ishlating. Inheritance faqat haqiqiy "is-a" munosabat va barqaror parent class bo'lganda.

**Deep Dive:**

JavaScript spec'da faqat single prototype chain (`[[Prototype]]`) qo'llab-quvvatlanadi — shuning uchun multiple inheritance to'g'ridan-to'g'ri mumkin emas. Composition bu cheklovdan qochadi — object'lar bir-biriga delegation emas, balki property copying orqali bog'lanadi (`Object.assign` yoki spread). V8 nuqtai nazaridan composition'da yaratilgan object'lar predictable hidden class'ga ega bo'ladi, chunki barcha property'lar yaratish vaqtida aniq — bu inline caching uchun qulay.

</details>

### 11. `#private in obj` tekshiruvi nima? [Middle+]

<details>
<summary>Javob</summary>

ES2022 da `#field in obj` sintaksisi — **brand check**. Object'da ma'lum class'ning private field'i borligini tekshiradi:

```javascript
class User {
  #id;
  constructor(id) { this.#id = id; }

  static isUser(obj) {
    return #id in obj; // brand check
  }

  equals(other) {
    if (!(#id in other)) return false; // boshqa class instance emas
    return this.#id === other.#id;
  }
}

const u1 = new User(1);
const u2 = new User(1);
const fake = { id: 1 };

console.log(User.isUser(u1));   // true
console.log(User.isUser(fake)); // false — #id yo'q
console.log(u1.equals(u2));     // true
console.log(u1.equals(fake));   // false
```

Bu `instanceof` dan kuchli — `instanceof` prototype chain tekshiradi (o'zgartirish mumkin), `#field in` esa haqiqiy class brand tekshiradi.

</details>

### 12. Class'da `new.target` nima? [Senior]

<details>
<summary>Javob</summary>

`new.target` — constructor ichida qaysi class `new` bilan chaqirilganini ko'rsatadi. Abstract class (instance yaratish taqiqlangan) yaratish uchun ishlatiladi:

```javascript
class Shape {
  constructor(name) {
    if (new.target === Shape) {
      throw new Error("Shape — abstract class, to'g'ridan-to'g'ri yaratib bo'lmaydi");
    }
    this.name = name;
  }

  area() {
    throw new Error("Subclass da implement qilish kerak");
  }
}

class Circle extends Shape {
  constructor(radius) {
    super("Circle");
    this.radius = radius;
  }
  area() { return Math.PI * this.radius ** 2; }
}

// new Shape("test"); // ❌ Error: abstract class
const c = new Circle(5); // ✅ new.target = Circle, Shape emas
console.log(c.area()); // 78.539...
```

`new.target` inheritance chain'da child class'ga ishora qiladi — parent constructor ichida ham:

```javascript
class A {
  constructor() { console.log(new.target.name); }
}
class B extends A {}
class C extends B {}

new A(); // "A"
new B(); // "B" — A constructor ichida, lekin new.target = B
new C(); // "C"
```

**Deep Dive:**

Spec'da `new.target` aslida `NewTarget` deb ataladi va u `GetNewTarget()` abstract operation orqali olinadi. Bu qiymat funksiya `[[Construct]]` internal method orqali chaqirilganda `newTarget` argument sifatida uzatiladi. Derived class'da `super()` chaqirilganda parent constructor'ga `newTarget` sifatida child class uzatiladi — shuning uchun parent ichida ham `new.target` child'ga teng. Oddiy funksiya chaqiruvida (`[[Call]]`) `new.target` `undefined` bo'ladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
class A {
  constructor() {
    this.name = "A";
    console.log(this.greet());
  }
  greet() { return "Hello from A"; }
}

class B extends A {
  constructor() {
    super();
    this.name = "B";
  }
  greet() { return "Hello from B"; }
}

const b = new B();
console.log(b.name);
```

<details>
<summary>Javob</summary>

```
"Hello from B"
"B"
```

```javascript
// new B() chaqirilganda:
// 1. B constructor ishlaydi → super() chaqiriladi
// 2. super() → A constructor ishlaydi
// 3. A constructor ichida: this.name = "A"
// 4. A constructor ichida: console.log(this.greet())
//    this = b instance, b.greet() → B ning greet() chaqiriladi (override!)
//    → "Hello from B"
// 5. A constructor tugadi, B constructor'ga qaytadi
// 6. this.name = "B" — A da yozilgan "A" ni override qiladi
// 7. console.log(b.name) → "B"
```

**Muhim nuqta:** `super()` ichida ham `this` child instance'ga ishora qiladi. Shuning uchun `this.greet()` child'ning override method'ini chaqiradi — parent'nikini emas.

</details>

### 2. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
class Counter {
  #count = 0;

  increment() {
    this.#count++;
    return this;
  }

  get value() { return this.#count; }
}

const c = new Counter();
console.log(c.increment().increment().increment().value);
console.log(Object.keys(c));
console.log(JSON.stringify(c));
```

<details>
<summary>Javob</summary>

```
3
[]
"{}"
```

```javascript
// c.increment().increment().increment()
// Har bir increment() this qaytaradi → chaining mumkin
// #count: 0 → 1 → 2 → 3
// .value → getter → 3

// Object.keys(c) → []
// #count private — ko'rinmaydi. Oddiy own property yo'q.

// JSON.stringify(c) → "{}"
// Private field'lar JSON.stringify da ko'rinmaydi
// Public own property'lar yo'q → bo'sh object
```

</details>

### 3. Quyidagi kodning output'ini ayting [Senior]

**Savol:**

```javascript
class Parent {
  static x = 1;
  static getX() { return this.x; }
}

class Child extends Parent {
  static x = 2;
}

console.log(Parent.getX());
console.log(Child.getX());
console.log(Child.hasOwnProperty("getX"));
console.log(Child.getX === Parent.getX);
```

<details>
<summary>Javob</summary>

```
1
2
false
true
```

```javascript
// Parent.getX() → this = Parent, Parent.x = 1 → 1
// Child.getX() → this = Child, Child.x = 2 → 2
//   (getX Child'da yo'q — Parent'dan meros. Lekin this = Child)

// Child.hasOwnProperty("getX") → false
//   (getX Child'ning o'zida yo'q, Parent'dan meros bo'lgan)

// Child.getX === Parent.getX → true
//   (bitta funksiya — Child.[[Prototype]] = Parent orqali topiladi)
```

**Muhim:** Static method'larda `this` — chaqirilgan class'ning o'zi. `Child.getX()` da `this` = `Child`, shuning uchun `this.x` = `Child.x` = 2.

**Deep Dive:**

`extends` keyword engine ichida ikkita prototype chain quradi: `Object.setPrototypeOf(Child.prototype, Parent.prototype)` (instance method meros) va `Object.setPrototypeOf(Child, Parent)` (static method meros). Shuning uchun `Child.getX` aslida `Child.[[Prototype]]` ya'ni `Parent` dan topiladi. Lekin `this` standart implicit binding qoidasi bo'yicha `Child` ga bog'lanadi — chunki `Child.getX()` shaklida chaqirilgan.

</details>

### 4. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
class A {
  x = 1;
  method() { return this.x; }
}

class B extends A {
  x = 2;
}

const b = new B();
console.log(b.method());
console.log(b.x);
console.log(Object.keys(b));
```

<details>
<summary>Javob</summary>

```
2
2
["x"]
```

```javascript
// Class field'lar tartib bo'yicha initialize bo'ladi:
// 1. super() chaqiriladi (B da default constructor)
// 2. A ning field'lari: this.x = 1
// 3. B ning field'lari: this.x = 2 (A dagi x ni OVERRIDE qildi)

// b.method() → this = b, this.x = 2 → 2
// b.x → 2
// Object.keys(b) → ["x"] — bitta own property (ikkinchi assign birinchini yozdi)
```

</details>

### 5. Bu kodda nima xato? [Middle]

**Savol:**

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} speaks`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    this.breed = breed;
    super(name);
  }
}

const rex = new Dog("Rex", "Labrador");
```

<details>
<summary>Javob</summary>

`ReferenceError: Must call super constructor in derived class before accessing 'this'`

```javascript
// ❌ this.breed = breed → super() dan OLDIN this ishlatilgan!
// Derived class'da this faqat super() chaqirilgandan keyin mavjud

// ✅ To'g'ri:
class Dog extends Animal {
  constructor(name, breed) {
    super(name);       // ✅ avval super
    this.breed = breed; // keyin this
  }
}
```

**Sababi:** Derived class'da `this` object parent constructor tomonidan yaratiladi. `super()` chaqirilmaguncha `this` mavjud emas — bu JavaScript'ning prototype-based meros mexanizmining tabiiy cheklovi.

</details>

### 6. Bu kodda nima xato? [Middle]

**Savol:**

```javascript
class Logger {
  prefix = "[LOG]";

  log(message) {
    console.log(`${this.prefix} ${message}`);
  }
}

const logger = new Logger();
const log = logger.log;
log("test");
```

<details>
<summary>Javob</summary>

`TypeError: Cannot read properties of undefined (reading 'prefix')` — `this` yo'qoldi.

```javascript
// log = logger.log — method reference olingan
// log("test") — this = undefined (strict mode, class ichida)

// ✅ Yechimlar:

// 1. Arrow function field:
class Logger {
  prefix = "[LOG]";
  log = (message) => {
    console.log(`${this.prefix} ${message}`);
  };
}

// 2. bind:
const log = logger.log.bind(logger);

// 3. Wrapper:
const log = (msg) => logger.log(msg);
```

Bu class method'larning ko'p uchraydigan muammosi — callback sifatida berilganda `this` binding yo'qoladi. Ko'proq [10-this-keyword.md](10-this-keyword.md) da.

</details>

---
