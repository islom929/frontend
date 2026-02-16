# Bo'lim 7: Prototypal Inheritance

> JavaScript da klassik inheritance yo'q — uning o'rniga prototypal inheritance bor. Har bir object boshqa object'dan meros oladi.

---

## Mundarija

- [[[Prototype]] Internal Slot](#prototype-internal-slot)
- [__proto__ vs prototype](#__proto__-vs-prototype)
- [Prototype Chain](#prototype-chain)
- [Object.create()](#objectcreate)
- [Constructor Functions](#constructor-functions)
- [new Keyword Ichidan](#new-keyword-ichidan)
- [instanceof Qanday Ishlaydi](#instanceof-qanday-ishlaydi)
- [Object.getPrototypeOf va setPrototypeOf](#objectgetprototypeof-va-setprototypeof)
- [Property Shadowing](#property-shadowing)
- [Performance: Prototype vs Instance Method](#performance-prototype-vs-instance-method)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## [[Prototype]] Internal Slot

### Nazariya

JavaScript da har bir object'ning ichki **`[[Prototype]]`** slot'i bor. Bu slot boshqa object'ga (yoki `null` ga) ishora qiladi. Biz bu object'ga **prototype** deymiz.

Object'da biror property topilmasa, engine **`[[Prototype]]`** bo'ylab yuqoriga qarab qidiradi.

```javascript
const animal = {
  eats: true,
  walk() { return "Yurish..."; }
};

const rabbit = {
  jumps: true
};

// rabbit ning [[Prototype]] ini animal ga bog'lash:
Object.setPrototypeOf(rabbit, animal);

console.log(rabbit.jumps); // true — o'zida bor
console.log(rabbit.eats);  // true — prototype dan oldi!
console.log(rabbit.walk()); // "Yurish..." — prototype dan
```

```
rabbit                    animal                  Object.prototype
┌────────────────┐       ┌────────────────┐      ┌──────────────────┐
│ jumps: true    │       │ eats: true     │      │ toString()       │
│                │       │ walk()         │      │ hasOwnProperty() │
│ [[Prototype]]──│──────►│ [[Prototype]]──│─────►│ [[Prototype]]:null│
└────────────────┘       └────────────────┘      └──────────────────┘
```

---

## __proto__ vs prototype

Bu ikki tushunchani aralashtirib yuborish — eng keng tarqalgan xato.

### `__proto__` (dunderpropro)

- Har bir **object** da bor
- `[[Prototype]]` internal slot ga **accessor** (getter/setter)
- Object ning **meros ota**siga ishora
- Zamonaviy kodda ishlatmang — `Object.getPrototypeOf()` ishlating

```javascript
const obj = {};
console.log(obj.__proto__ === Object.prototype); // true
// obj ning "otasi" — Object.prototype
```

### `prototype` property

- Faqat **funksiya**larda bor (arrow function'larda yo'q)
- `new` bilan chaqirilganda, yangi object'ning `[[Prototype]]`i shu object bo'ladi
- **Funksiyaning** o'zi emas, **yaratadigan instance'lar** uchun

```javascript
function Dog(name) {
  this.name = name;
}
Dog.prototype.bark = function() { return "Hav!"; };

const rex = new Dog("Rex");
// rex.__proto__ === Dog.prototype ✅
// rex.bark() — Dog.prototype dan olindi
```

### Farq — Diagramma

```
                Dog (function)
                ┌──────────────────┐
                │ prototype: ──────│──────┐
                └──────────────────┘      │
                                          ▼
        ┌─── rex ───────┐      ┌── Dog.prototype ──┐
        │ name: "Rex"   │      │ bark()            │
        │ __proto__: ───│─────►│ constructor: Dog  │
        └───────────────┘      │ __proto__: ───────│──► Object.prototype
                               └───────────────────┘
        
__proto__  = object'ning MEROS OTASI    (har bir object da)
prototype  = funksiya YARATADIGAN       (faqat funksiyalarda)
             instance'lar uchun ota
```

### Taqqoslash

| | `__proto__` | `prototype` |
|-|-------------|-------------|
| **Kimda bor** | Har bir object | Faqat function'larda |
| **Nima** | [[Prototype]] ga accessor | new bilan yaratiladigan instance'lar uchun ota |
| **Ishlatish** | `Object.getPrototypeOf()` | Constructor function / class |
| **O'zgartirish** | `Object.setPrototypeOf()` | `Fn.prototype.method = ...` |

---

## Prototype Chain

### Nazariya

Object'da property topilmasa, engine `[[Prototype]]` zanjiri bo'ylab **yuqoriga** qidiradi — to `null` gacha.

```javascript
const grandparent = { a: 1 };
const parent = Object.create(grandparent);
parent.b = 2;
const child = Object.create(parent);
child.c = 3;

console.log(child.c); // 3 — o'zida
console.log(child.b); // 2 — parent da
console.log(child.a); // 1 — grandparent da
console.log(child.d); // undefined — hech qayerda yo'q
```

### Lookup Jarayoni

```
child.a qidirish:

child → { c: 3 }         a bor? YO'Q → yuqoriga
  ↓
parent → { b: 2 }        a bor? YO'Q → yuqoriga
  ↓
grandparent → { a: 1 }   a bor? HA! → return 1
```

```
child.d qidirish:

child → { c: 3 }              d bor? YO'Q
  ↓
parent → { b: 2 }             d bor? YO'Q
  ↓
grandparent → { a: 1 }        d bor? YO'Q
  ↓
Object.prototype → {toString, hasOwnProperty, ...}  d bor? YO'Q
  ↓
null → ZANJIR TUGADI → return undefined
```

### Har Bir Object'ning Oxirgi Otasi — Object.prototype

```javascript
const obj = { x: 1 };

// Prototype chain:
// obj → Object.prototype → null

console.log(obj.toString());        // "[object Object]" — Object.prototype dan
console.log(obj.hasOwnProperty("x")); // true — Object.prototype dan

// Object.prototype ning prototypi — null
console.log(Object.getPrototypeOf(Object.prototype)); // null
```

### Maxsus: Object.create(null)

```javascript
// Prototype chain'siz object — "dictionary"
const dict = Object.create(null);
dict.key = "value";

console.log(dict.toString);         // undefined — Object.prototype yo'q!
console.log(dict.hasOwnProperty);   // undefined

// Use case: sof key-value storage, method collision bo'lmaydi
// Object.keys(dict) ishlaydi, lekin for...in da prototype'dan keluvchi property yo'q
```

---

## Object.create()

### Nazariya

`Object.create(proto)` — yangi object yaratadi va uning `[[Prototype]]`ini `proto` ga bog'laydi.

```javascript
const vehicle = {
  type: "noma'lum",
  describe() {
    return `Bu ${this.type}`;
  }
};

const car = Object.create(vehicle);
car.type = "avtomobil";
car.wheels = 4;

console.log(car.describe()); // "Bu avtomobil"
console.log(car.wheels);      // 4 — o'zida
console.log(car.type);        // "avtomobil" — o'zida (vehicle.type ni shadow)
```

### Property Descriptors bilan

```javascript
const car = Object.create(vehicle, {
  type: {
    value: "avtomobil",
    writable: true,
    enumerable: true,
    configurable: true
  },
  wheels: {
    value: 4,
    writable: false,
    enumerable: true,
    configurable: false
  }
});
```

### Prototype Chain Qurish

```javascript
// 3 darajali chain:
const living = { alive: true };
const animal = Object.create(living);
animal.eats = true;
const dog = Object.create(animal);
dog.barks = true;

// dog → animal → living → Object.prototype → null
console.log(dog.barks);  // true (o'zida)
console.log(dog.eats);   // true (animal)
console.log(dog.alive);  // true (living)
```

---

## Constructor Functions

### Nazariya

ES6 dan oldin class yo'q edi. Object yaratish uchun **constructor function** ishlatilgan — oddiy funksiya, `new` bilan chaqiriladi.

```javascript
function Person(name, age) {
  // new bilan chaqirilganda:
  // 1. this = {} (yangi bo'sh object)
  // 2. this.__proto__ = Person.prototype
  this.name = name;
  this.age = age;
  // 3. return this (avtomatik)
}

// Method'lar prototype da — barcha instance'lar share qiladi
Person.prototype.greet = function() {
  return `Salom, men ${this.name}, ${this.age} yoshdaman`;
};

Person.prototype.isAdult = function() {
  return this.age >= 18;
};

const ali = new Person("Ali", 25);
const vali = new Person("Vali", 17);

ali.greet();    // "Salom, men Ali, 25 yoshdaman"
vali.isAdult(); // false

// Method'lar SHARE qilingan:
console.log(ali.greet === vali.greet); // true — bitta funksiya!
```

### constructor Property

Har bir `prototype` object'da `constructor` property bor — u yaratuvchi funksiyaga ishora qiladi:

```javascript
console.log(Person.prototype.constructor === Person); // true
console.log(ali.constructor === Person);               // true (prototype dan)

// Yangi instance yaratish:
const cloneAli = new ali.constructor("Ali Clone", 25);
```

---

## new Keyword Ichidan

### Nazariya

`new` keyword nima qiladi — **4 qadam**:

```javascript
function User(name) {
  this.name = name;
}
User.prototype.greet = function() { return this.name; };

const user = new User("Islom");
```

### Step-by-Step

```javascript
// new User("Islom") ichidan:

// 1. Bo'sh object yaratish
const obj = {};

// 2. [[Prototype]] ni bog'lash
Object.setPrototypeOf(obj, User.prototype);
// obj.__proto__ = User.prototype

// 3. Constructor ni obj bilan chaqirish (this = obj)
const result = User.call(obj, "Islom");
// obj.name = "Islom"

// 4. Return logic:
//    - Agar constructor OBJECT qaytarsa → shu object
//    - Agar primitive yoki hech narsa qaytarmasa → obj
//    result = undefined (constructor hech narsa qaytarmadi)
//    → return obj
```

### Diagramma

```
new User("Islom")

Step 1:  obj = {}
Step 2:  obj.[[Prototype]] = User.prototype
Step 3:  User.call(obj, "Islom") → obj.name = "Islom"
Step 4:  return obj

Natija:
┌──────────────┐        ┌─────────────────┐
│ user         │        │ User.prototype  │
│ name: "Islom"│        │ greet()         │
│ __proto__: ──│───────►│ constructor: User│
└──────────────┘        └─────────────────┘
```

### Return Override

```javascript
// Primitive qaytarsa — IGNORED, this qaytariladi
function Foo() {
  this.a = 1;
  return 42; // ← ignored! primitive
}
console.log(new Foo()); // { a: 1 }

// Object qaytarsa — shu object qaytariladi
function Bar() {
  this.a = 1;
  return { b: 2 }; // ← object! this o'rniga shu qaytadi
}
console.log(new Bar()); // { b: 2 } — this yo'qoldi!

// null — primitive hisoblanadi:
function Baz() {
  this.a = 1;
  return null; // ← ignored!
}
console.log(new Baz()); // { a: 1 }
```

### new ni O'zimiz Yozamiz

```javascript
function myNew(Constructor, ...args) {
  // 1. Bo'sh object, prototype bog'lash
  const obj = Object.create(Constructor.prototype);

  // 2. Constructor chaqirish
  const result = Constructor.apply(obj, args);

  // 3. Return logic
  return (result !== null && typeof result === "object") ? result : obj;
}

// Test:
function Person(name) { this.name = name; }
Person.prototype.greet = function() { return this.name; };

const p = myNew(Person, "Ali");
console.log(p.name);    // "Ali"
console.log(p.greet()); // "Ali"
console.log(p instanceof Person); // true ✅
```

---

## instanceof Qanday Ishlaydi

### Nazariya

`instanceof` operator object'ning prototype chain'ida berilgan constructor'ning `prototype`'i borligini tekshiradi.

```javascript
function Animal() {}
function Dog() {}
Dog.prototype = Object.create(Animal.prototype);

const rex = new Dog();

console.log(rex instanceof Dog);    // true
console.log(rex instanceof Animal); // true
console.log(rex instanceof Object); // true
```

### Ichki Algoritm

```javascript
// rex instanceof Dog
// Engine quyidagini tekshiradi:

rex.__proto__ === Dog.prototype?
// Ha → TRUE

// rex instanceof Animal:
rex.__proto__ === Animal.prototype?           // Yo'q (Dog.prototype)
rex.__proto__.__proto__ === Animal.prototype?  // Ha → TRUE

// rex instanceof Object:
// ... chain bo'ylab Object.prototype gacha yuradi → TRUE
```

```
rex.__proto__                = Dog.prototype       ✅ instanceof Dog
rex.__proto__.__proto__      = Animal.prototype     ✅ instanceof Animal
rex.__proto__.__proto__.__proto__ = Object.prototype ✅ instanceof Object
rex.__proto__.__proto__.__proto__.__proto__ = null   ← chain tugadi
```

### instanceof Muammolari

```javascript
// 1. Prototype o'zgarsa:
function Foo() {}
const foo = new Foo();
console.log(foo instanceof Foo); // true

Foo.prototype = {}; // prototype ALMASHTIRILDI
console.log(foo instanceof Foo); // false! — eski prototype endi chain da yo'q

// 2. Har xil realm (iframe):
// iframe dagi Array !== asosiy sahifadagi Array
// arr instanceof Array — false bo'lishi mumkin
// Yechim: Array.isArray(arr)
```

---

## Object.getPrototypeOf va setPrototypeOf

```javascript
const proto = { greet() { return "Salom"; } };
const obj = Object.create(proto);

// O'qish:
Object.getPrototypeOf(obj) === proto; // true

// O'zgartirish (tavsiya etilMAYDI — performance muammo):
const newProto = { greet() { return "Hello"; } };
Object.setPrototypeOf(obj, newProto);
obj.greet(); // "Hello"
```

⚠️ **`Object.setPrototypeOf`** ni production da ishlatmang! U juda **sekin** — V8 inline cache va hidden class optimizatsiyalarini buzadi. O'rniga `Object.create` bilan boshidanoq to'g'ri prototype chain quring.

---

## Property Shadowing

### Nazariya

Agar object'da va uning prototype'ida **bir xil nomli** property bo'lsa, object'ning o'zi **yutadi** (shadow qiladi).

```javascript
const parent = {
  name: "Parent",
  greet() { return `Salom, ${this.name}`; }
};

const child = Object.create(parent);
child.name = "Child"; // parent.name ni SHADOW qildi

console.log(child.name);    // "Child" — o'zida bor, prototype ga yetmaydi
console.log(parent.name);   // "Parent" — o'zgarmagan
console.log(child.greet()); // "Salom, Child" — this = child
```

### Shadowing va Write

```javascript
const parent = { count: 0 };
const child = Object.create(parent);

// O'qish — prototype'dan:
console.log(child.count); // 0 (parent dan)

// YOZISH — child'da YANGI property yaratiladi:
child.count = 10;
console.log(child.count);  // 10 (o'zida — shadow)
console.log(parent.count); // 0 — o'zgarmagan!
```

**Qoida:** Property **o'qish** prototype chain bo'ylab yuradi. Property **yozish** faqat o'z object'iga yozadi (agar setter bo'lmasa).

### Shadowing Edge Case — writable: false

```javascript
const parent = {};
Object.defineProperty(parent, "x", { value: 1, writable: false });

const child = Object.create(parent);
child.x = 10; // ❌ Silent fail! (strict: TypeError)
console.log(child.x); // 1 — parent'dan, child'da yaratilMADI

// Nima uchun? Agar prototype'dagi property writable:false bo'lsa,
// child'da shadow yaratish HAM taqiqlanadi.
// Bu kutilmagan behavior — ko'pchilik bilmaydi.
```

---

## Performance: Prototype vs Instance Method

```javascript
// Variant 1: Instance method — har bir object'da yangi funksiya
function UserA(name) {
  this.name = name;
  this.greet = function() { return this.name; }; // ← har instance uchun yangi!
}

// Variant 2: Prototype method — barcha instance share qiladi
function UserB(name) {
  this.name = name;
}
UserB.prototype.greet = function() { return this.name; };
```

```
UserA — 1000 ta instance:
┌──────────┐ ┌──────────┐ ┌──────────┐
│name:"Ali" │ │name:"Bob"│ │name:"Kim"│  × 1000
│greet: fn1 │ │greet: fn2│ │greet: fn3│  ← 1000 ta alohida funksiya!
└──────────┘ └──────────┘ └──────────┘

UserB — 1000 ta instance:
┌──────────┐ ┌──────────┐ ┌──────────┐
│name:"Ali" │ │name:"Bob"│ │name:"Kim"│  × 1000
│__proto__──│►│__proto__──│►│__proto__──│► UserB.prototype
└──────────┘ └──────────┘ └──────────┘   │ greet: fn │ ← BITTA funksiya!
                                          └───────────┘
```

| | Instance Method | Prototype Method |
|-|----------------|-----------------|
| **Memory** | Har instance uchun yangi fn | Bitta fn, hammasi share |
| **Tezlik (access)** | Biroz tezroq (o'zida) | Biroz sekinroq (chain lookup) |
| **Tezlik (yaratish)** | Sekin (har safar fn yaratish) | Tez |
| **Private access** | Closure orqali mumkin | Mumkin emas |

**Tavsiya:** Prototype/class method — default tanlov. Instance method faqat closure (private data) kerak bo'lganda.

---

## Common Mistakes

### ❌ Xato 1: __proto__ va prototype ni aralashtirib yuborish

```javascript
function Dog(name) { this.name = name; }

const rex = new Dog("Rex");

// ❌ NOTO'G'RI tushunish:
// "rex.prototype da method'lar bor"
console.log(rex.prototype); // undefined! — instance'da prototype yo'q!

// ✅ TO'G'RI:
// "rex.__proto__ (ya'ni Dog.prototype) da method'lar bor"
console.log(rex.__proto__ === Dog.prototype); // true
console.log(Object.getPrototypeOf(rex) === Dog.prototype); // true
```

**Nima uchun:** `prototype` faqat **funksiyalarda** bor. Instance (object)'da `__proto__` bor — bu uning prototype'iga ishora.

---

### ❌ Xato 2: Prototype ni to'liq almashtirganda constructor yo'qolishi

```javascript
function User(name) { this.name = name; }

// ❌ Prototype ni to'liq almashtiryapmiz:
User.prototype = {
  greet() { return this.name; }
};

const user = new User("Ali");
console.log(user.constructor === User); // false! ❌
console.log(user.constructor === Object); // true — constructor yo'qoldi
```

### ✅ To'g'ri usul:

```javascript
// Variant 1: constructor ni qaytarish
User.prototype = {
  constructor: User, // ← qayta qo'shish
  greet() { return this.name; }
};

// Variant 2: prototype ga qo'shish, almashmaslik (tavsiya)
User.prototype.greet = function() { return this.name; };
// constructor saqlanadi
```

**Nima uchun:** Prototype ni `= {}` bilan almashtirganda, yangi object'da `constructor` yo'q. Default `Object.prototype.constructor` = `Object` bo'lib qoladi.

---

### ❌ Xato 3: Prototype da reference type (object/array)

```javascript
function Team(name) { this.name = name; }
Team.prototype.members = []; // ❌ Barcha instance SHARE qiladi!

const team1 = new Team("Alpha");
const team2 = new Team("Beta");

team1.members.push("Ali");
console.log(team2.members); // ["Ali"] — team2 ga ham tushdi! 😱
```

### ✅ To'g'ri usul:

```javascript
function Team(name) {
  this.name = name;
  this.members = []; // ✅ Har instance o'zining array'i
}

Team.prototype.addMember = function(name) {
  this.members.push(name);
};

const team1 = new Team("Alpha");
const team2 = new Team("Beta");

team1.addMember("Ali");
console.log(team2.members); // [] ✅ — alohida
```

**Nima uchun:** Prototype'dagi object/array **shared** — barcha instance bitta reference orqali ko'radi. Mutable data har doim **constructor ichida** (instance'da) yarating.

---

### ❌ Xato 4: setPrototypeOf ishlatish (performance)

```javascript
// ❌ Runtime da prototype o'zgartirish
const obj = { a: 1 };
const proto = { greet() { return "hi"; } };
Object.setPrototypeOf(obj, proto); // SEKIN — V8 optimizatsiya buziladi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Boshidanoq Object.create bilan
const proto = { greet() { return "hi"; } };
const obj = Object.create(proto);
obj.a = 1;
```

**Nima uchun:** `setPrototypeOf` V8 ning hidden class va inline cache optimizatsiyalarini buzadi. MDN ham "extremely slow operation" deb yozgan.

---

### ❌ Xato 5: for...in bilan prototype property olish

```javascript
function User(name) { this.name = name; }
User.prototype.role = "user";

const ali = new User("Ali");

const copy = {};
for (const key in ali) {
  copy[key] = ali[key]; // role ham kiradi!
}
console.log(copy); // { name: "Ali", role: "user" } — keraksiz!
```

### ✅ To'g'ri usul:

```javascript
// Object.keys — faqat own property
const copy = {};
for (const key of Object.keys(ali)) {
  copy[key] = ali[key];
}
// Yoki spread:
const copy2 = { ...ali }; // { name: "Ali" } ✅
```

**Nima uchun:** `for...in` prototype chain'ni ham yuradi. `Object.keys` va spread faqat own enumerable property'larni oladi.

---

## Amaliy Mashqlar

### Mashq 1: Prototype Chain (Oson)

**Savol:** Quyidagi kodda `d.toString()` chaqirilganda, engine qanday qidiradi?

```javascript
const a = { x: 1 };
const b = Object.create(a);
b.y = 2;
const c = Object.create(b);
c.z = 3;
```

<details>
<summary>Javob</summary>

```
c.toString() qidirish:

c → { z: 3 }                      toString bor? YO'Q
  ↓
b → { y: 2 }                      toString bor? YO'Q
  ↓
a → { x: 1 }                      toString bor? YO'Q
  ↓
Object.prototype → { toString, ... }  toString bor? HA! → TOPILDI

Prototype chain: c → b → a → Object.prototype → null
```
</details>

---

### Mashq 2: new Keyword ni Implement Qiling (O'rta)

**Savol:** `myNew(Constructor, ...args)` yozing.

<details>
<summary>Javob</summary>

```javascript
function myNew(Constructor, ...args) {
  const obj = Object.create(Constructor.prototype);
  const result = Constructor.apply(obj, args);
  return (result !== null && typeof result === "object") ? result : obj;
}

// Test:
function Car(brand) { this.brand = brand; }
Car.prototype.drive = function() { return `${this.brand} haydash`; };

const car = myNew(Car, "BMW");
console.log(car.brand);            // "BMW"
console.log(car.drive());          // "BMW haydash"
console.log(car instanceof Car);   // true
console.log(car.constructor === Car); // true
```
</details>

---

### Mashq 3: Inheritance Chain (O'rta)

**Savol:** `Animal → Dog → GuideDog` prototype chain yarating. Har birida method'lar, GuideDog barcha ota method'larini ishlatsin.

<details>
<summary>Javob</summary>

```javascript
function Animal(name) {
  this.name = name;
}
Animal.prototype.eat = function() {
  return `${this.name} yemoqda`;
};

function Dog(name, breed) {
  Animal.call(this, name); // super()
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function() {
  return `${this.name} havlayapti!`;
};

function GuideDog(name, breed, owner) {
  Dog.call(this, name, breed);
  this.owner = owner;
}
GuideDog.prototype = Object.create(Dog.prototype);
GuideDog.prototype.constructor = GuideDog;
GuideDog.prototype.guide = function() {
  return `${this.name} ${this.owner}ni boshqaryapti`;
};

const buddy = new GuideDog("Buddy", "Labrador", "Ali");
console.log(buddy.eat());    // "Buddy yemoqda" (Animal'dan)
console.log(buddy.bark());   // "Buddy havlayapti!" (Dog'dan)
console.log(buddy.guide());  // "Buddy Alini boshqaryapti" (GuideDog)
console.log(buddy instanceof GuideDog); // true
console.log(buddy instanceof Dog);      // true
console.log(buddy instanceof Animal);   // true
```
</details>

---

### Mashq 4: instanceof Implement Qilish (Qiyin)

**Savol:** `myInstanceof(obj, Constructor)` yozing.

<details>
<summary>Javob</summary>

```javascript
function myInstanceof(obj, Constructor) {
  if (obj === null || typeof obj !== "object") return false;

  let proto = Object.getPrototypeOf(obj);

  while (proto !== null) {
    if (proto === Constructor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }

  return false;
}

// Test:
function A() {}
function B() {}
B.prototype = Object.create(A.prototype);

const b = new B();
console.log(myInstanceof(b, B));      // true
console.log(myInstanceof(b, A));      // true
console.log(myInstanceof(b, Object)); // true
console.log(myInstanceof(b, Array));  // false
console.log(myInstanceof(42, Number)); // false (primitive)
```

**Tushuntirish:** Prototype chain bo'ylab yuqoriga yurib, `Constructor.prototype` ni qidiramiz. Topsak — `true`, `null` ga yetsak — `false`.
</details>

---

### Mashq 5: Mixin Pattern (Qiyin)

**Savol:** Bir nechta object'lardan method'larni birlashtiradigan `mixin` funksiyasi yozing.

<details>
<summary>Javob</summary>

```javascript
function mixin(target, ...sources) {
  for (const source of sources) {
    // Own enumerable property'larni olish (Symbol ham)
    for (const key of Reflect.ownKeys(source)) {
      if (key === "constructor") continue; // constructor skip

      const desc = Object.getOwnPropertyDescriptor(source, key);
      Object.defineProperty(target, desc ? key : key, desc || { value: source[key] });
    }
  }
  return target;
}

// Use case:
const Serializable = {
  serialize()   { return JSON.stringify(this); },
  deserialize(json) { return Object.assign(this, JSON.parse(json)); }
};

const Validatable = {
  validate() {
    for (const [key, val] of Object.entries(this)) {
      if (val === null || val === undefined) {
        throw new Error(`${key} bo'sh bo'lmasligi kerak`);
      }
    }
    return true;
  }
};

function User(name, email) {
  this.name = name;
  this.email = email;
}

mixin(User.prototype, Serializable, Validatable);

const user = new User("Ali", "ali@mail.com");
console.log(user.serialize());    // '{"name":"Ali","email":"ali@mail.com"}'
console.log(user.validate());     // true
```

**Tushuntirish:** JavaScript single inheritance — faqat bitta prototype. Mixin pattern orqali bir nechta "trait" / "capability" ni qo'shish mumkin. Bu [08-classes.md](08-classes.md) da ko'proq.
</details>

---

## Xulosa

1. **`[[Prototype]]`** — har bir object'ning internal slot'i. Boshqa object yoki null ga ishora. Property topilmasa shu zanjir bo'ylab qidiriladi.

2. **`__proto__` vs `prototype`:** `__proto__` = object'ning otasi (har kimda bor). `prototype` = funksiya yaratadigan instance'lar uchun ota (faqat function'da).

3. **Prototype Chain:** object → prototype → prototype → ... → Object.prototype → null. Property o'qishda chain bo'ylab yuradi, yozishda faqat o'z object'iga.

4. **`new` keyword 4 qadam:** bo'sh object → prototype bog'lash → constructor chaqirish → return (object yoki this).

5. **`instanceof`:** prototype chain bo'ylab `Constructor.prototype` ni qidiradi.

6. **Property Shadowing:** o'zidagi property prototype'dagi bir xil nomlini "yashiradi".

7. **Performance:** Prototype method'lar memory tejaydi (shared). `setPrototypeOf` sekin — boshidanoq `Object.create` ishlating.

8. **Prototype'da mutable data qo'ymang** (array, object). Constructor ichida yarating.

---

> **Keyingi bo'lim:** [08-classes.md](08-classes.md) — ES6 Classes — syntactic sugar over prototypes, extends, super, private fields.
