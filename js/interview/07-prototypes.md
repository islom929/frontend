# Prototypal Inheritance — Interview Savollari

> Prototype chain, `[[Prototype]]`, constructor functions, `new` keyword, `instanceof`, property shadowing va prototype-based meros haqida interview savollari.

---

## Nazariy savollar

### 1. `__proto__` va `prototype` farqi nima? [Junior+]

<details>
<summary>Javob</summary>

Bu ikki tushuncha butunlay farqli narsalar:

- **`__proto__`** — har bir **object** da mavjud accessor. Object'ning `[[Prototype]]` internal slot'iga (ya'ni uning "ota" prototype object'iga) kirish imkonini beradi. Zamonaviy kodda `Object.getPrototypeOf()` ishlatish tavsiya etiladi.

- **`prototype`** — faqat **funksiya**larda mavjud oddiy property. `new` bilan funksiya chaqirilganda, yangi yaratilgan object'ning `[[Prototype]]` iga aynan shu `prototype` object bog'lanadi. Ya'ni u funksiyaning o'zi uchun emas, funksiya **yaratadigan instance'lar** uchun ota object.

```javascript
function Dog(name) { this.name = name; }
Dog.prototype.bark = function() { return "Hav!"; };

const rex = new Dog("Rex");

// rex.__proto__ — rex ning "otasi" = Dog.prototype
console.log(rex.__proto__ === Dog.prototype); // true ✅
console.log(Object.getPrototypeOf(rex) === Dog.prototype); // true ✅

// rex.prototype — MAVJUD EMAS (rex funksiya emas)
console.log(rex.prototype); // undefined
```

| | `__proto__` | `prototype` |
|-|-------------|-------------|
| **Kimda bor** | Har bir object | Faqat function'larda |
| **Nima** | `[[Prototype]]` ga accessor | `new` bilan yaratiladigan instance'lar uchun ota |
| **Zamonaviy API** | `Object.getPrototypeOf()` | `Fn.prototype.method = ...` |

</details>

### 2. Prototype chain nima va qanday ishlaydi? [Junior+]

<details>
<summary>Javob</summary>

Prototype chain — ob'ektlar bir-biriga `[[Prototype]]` orqali bog'langan zanjir. Object'da property topilmasa, engine bu zanjir bo'ylab **yuqoriga** qidiradi — `null` gacha.

Lookup algoritmi:
1. Object'ning **own property** larida qidiradi
2. Topilmasa — `[[Prototype]]` (ota) da qidiradi
3. Topilmasa — otaning otasida, va hokazo
4. `Object.prototype` → `null` ga yetilsa va topilmasa — `undefined`

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

// Chain: child → parent → grandparent → Object.prototype → null
```

V8 engine tezlashtirish uchun **inline cache** ishlatadi — birinchi lookup natijasini eslab qoladi, keyingilarda chain'ni qayta yurmaydi.

</details>

### 3. `new` keyword ichida nima sodir bo'ladi? Step-by-step tushuntiring. [Middle]

<details>
<summary>Javob</summary>

`new Constructor(args)` chaqirilganda engine 4 ta qadam bajaradi:

```javascript
function User(name) { this.name = name; }
User.prototype.greet = function() { return this.name; };

const user = new User("Ali");
```

**4 qadam:**

```javascript
// 1. Bo'sh object yaratish
const obj = {};

// 2. [[Prototype]] bog'lash
Object.setPrototypeOf(obj, User.prototype);

// 3. Constructor ni obj kontekstida chaqirish (this = obj)
const result = User.call(obj, "Ali");
// obj.name = "Ali"

// 4. Return logic:
//    - Constructor OBJECT qaytarsa → shu object return bo'ladi
//    - Primitive yoki hech narsa qaytarmasa → obj return bo'ladi
// result = undefined → return obj
```

Return override muhim edge case:

```javascript
// Primitive qaytarsa — IGNORED:
function Foo() { this.a = 1; return 42; }
new Foo(); // { a: 1 } — 42 ignored

// Object qaytarsa — this yo'qoladi:
function Bar() { this.a = 1; return { b: 2 }; }
new Bar(); // { b: 2 } — this.a yo'qoldi

// null — primitive sifatida ignored:
function Baz() { this.a = 1; return null; }
new Baz(); // { a: 1 }
```

</details>

### 4. `Object.create(null)` nima va nima uchun ishlatiladi? [Middle]

<details>
<summary>Javob</summary>

`Object.create(null)` `[[Prototype]]` i `null` bo'lgan ob'ekt yaratadi. Bu ob'ektda `Object.prototype` dan meros olinadigan hech qanday method (`toString`, `hasOwnProperty`, `valueOf`, `constructor`) **mavjud emas**.

```javascript
const dict = Object.create(null);
dict.key = "value";

console.log(dict.toString);       // undefined — Object.prototype yo'q
console.log(dict.hasOwnProperty); // undefined
console.log(dict.constructor);    // undefined

// Ammo Object.keys ishlaydi (static method — prototype'da emas):
console.log(Object.keys(dict)); // ["key"]
```

**Nima uchun ishlatiladi:**

1. **Toza dictionary/map** — `toString`, `constructor` kabi nomlar bilan key collision xavfi yo'q
2. **Prototype pollution** dan himoya — `__proto__` key xavfsiz oddiy property sifatida yoziladi
3. **Cache object** — faqat qo'shilgan key'lar mavjud

```javascript
// Oddiy object'da:
const obj = {};
obj["__proto__"] = { hacked: true };
// ⚠️ Bu obj'ning O'Z [[Prototype]] slot'ini o'zgartiradi (Object.prototype.__proto__
// setter orqali) — global Object.prototype'ga ta'sir qilmaydi. Lekin agar bu kod
// deep merge pattern ichida user input'dan kelsa (masalan: merge({}, userJSON)),
// recursive walk natijasida Object.prototype ga `isAdmin: true` kabi property
// inject bo'lishi mumkin — bu haqiqiy prototype pollution.

// Object.create(null) da xavfsiz:
const safeDict = Object.create(null);
safeDict["__proto__"] = "oddiy qiymat";
// [[Prototype]] yo'q → __proto__ accessor ham yo'q → oddiy data property bo'ladi
console.log(safeDict["__proto__"]); // "oddiy qiymat" ✅
```

</details>

### 5. `for...in`, `Object.keys`, `Object.getOwnPropertyNames` farqi nima? [Middle]

<details>
<summary>Javob</summary>

```javascript
function User(name) { this.name = name; }
User.prototype.role = "user";

const ali = new User("Ali");
Object.defineProperty(ali, "id", {
  value: 1,
  enumerable: false
});
```

| Method | Nima qaytaradi | Prototype | Non-enumerable |
|--------|---------------|-----------|----------------|
| `for...in` | Barcha enumerable | ✅ Ha | ❌ Yo'q |
| `Object.keys()` | Own enumerable | ❌ Yo'q | ❌ Yo'q |
| `Object.getOwnPropertyNames()` | Own (hammasi) | ❌ Yo'q | ✅ Ha |
| `Reflect.ownKeys()` | Own (hammasi + Symbol) | ❌ Yo'q | ✅ Ha |

```javascript
for (const key in ali) console.log(key);
// "name", "role" (id kirMAYDI — non-enumerable)

console.log(Object.keys(ali));
// ["name"]

console.log(Object.getOwnPropertyNames(ali));
// ["name", "id"]
```

</details>

### 6. Constructor inheritance qanday qilinadi? (ES5 usuli) [Middle+]

<details>
<summary>Javob</summary>

Constructor function'lar bilan inheritance 3 ta muhim qadamdan iborat:

```javascript
// Parent
function Shape(color) {
  this.color = color;
}
Shape.prototype.describe = function() {
  return `${this.color} ${this.type}`;
};

// Child
function Circle(color, radius) {
  Shape.call(this, color);  // QADAM 1: Parent constructor chaqirish
  this.radius = radius;
  this.type = "circle";
}

Circle.prototype = Object.create(Shape.prototype); // QADAM 2: Prototype chain ulash
Circle.prototype.constructor = Circle;              // QADAM 3: constructor tiklash

Circle.prototype.area = function() {
  return Math.PI * this.radius ** 2;
};

const c = new Circle("red", 5);
console.log(c.describe()); // "red circle"
console.log(c.area());     // 78.539...
console.log(c instanceof Circle); // true
console.log(c instanceof Shape);  // true
```

| Qadam | Nima qiladi | Agar qilmasak |
|-------|-------------|---------------|
| `Shape.call(this, color)` | Parent'ning instance property'larini olish | `this.color` yo'q bo'ladi |
| `Object.create(Shape.prototype)` | Prototype chain ulash | `instanceof Shape` false, Shape method'lari yo'q |
| `constructor = Circle` | constructor reference tiklash | `c.constructor === Circle` false |

**Deep Dive:**

`Object.create(Shape.prototype)` ishlatish sababi — `new Shape()` emas. `new Shape()` desak, Shape constructor side-effect'lari bo'lishi mumkin (API call, DOM manipulation, required parameter). `Object.create` faqat prototype chain'ni ulaydi, constructor'ni chaqirmaydi.

</details>

### 7. Prototype-based inheritance va class-based inheritance farqi nima? [Senior]

<details>
<summary>Javob</summary>

JavaScript **prototype-based** — ob'ektlar to'g'ridan-to'g'ri boshqa **ob'ekt**dan meros oladi. Java/C++ **class-based** — ob'ektlar **class** (blueprint) dan yaratiladi.

| | Prototype-based (JS) | Class-based (Java/C++) |
|-|---------------------|----------------------|
| **Meros** | Object → Object delegation | Class → Instance |
| **Runtime o'zgartirish** | Prototype runtime da o'zgartiriladi | Class compile-time da aniqlanadi |
| **Method qo'shish** | Istalgan vaqtda prototype'ga qo'shish mumkin | Recompile kerak |
| **Ko'p meros** | Yo'q (bitta [[Prototype]]), mixin bilan emulate | Interface / multiple inheritance |
| **Property saqlash** | Delegation — child'da yo'q, parent'dan so'raydi | Copy — har instance o'z nusxasiga ega |

```javascript
// JS da runtime da prototype'ga method qo'shish:
function User(name) { this.name = name; }
const ali = new User("Ali");

// ali.greet(); // TypeError — hozir yo'q

User.prototype.greet = function() { return this.name; };
console.log(ali.greet()); // "Ali" — delegation orqali ishlaydi
```

**Deep Dive:**

ES6 `class` sintaksisi kirganidan keyin JavaScript'da ham class-based ko'rinishi bor, lekin bu **syntactic sugar** — ichida xuddi shu prototype mexanizm ishlaydi. `class User { greet() {} }` engine ichida `User.prototype.greet = function() {}` ga aylanadi. `extends` esa `Object.create` va `Object.setPrototypeOf` kombinatsiyasi.

</details>

### 8. Prototype pollution nima va qanday himoya qilish? [Senior]

<details>
<summary>Javob</summary>

Prototype pollution — zararli data orqali `Object.prototype` ni o'zgartirish hujumi:

```javascript
// ❌ Xavfli — user input dan recursive merge
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] = target[key] || {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

// Attacker JSON:
const malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, malicious);

// Endi BARCHA object'lar:
const user = {};
console.log(user.isAdmin); // true! ❌
```

**Himoya usullari:**

```javascript
// 1. Xavfli key'larni filter qilish
function safeMerge(target, source) {
  for (const key of Object.keys(source)) { // for...in emas!
    if (key === "__proto__" || key === "constructor" || key === "prototype") {
      continue;
    }
    // ...
  }
}

// 2. Object.create(null) ishlatish
const config = Object.create(null);

// 3. Map ishlatish
const data = new Map();

// 4. Object.freeze(Object.prototype) — ehtiyot bilan
Object.freeze(Object.prototype);
```

**Deep Dive:**

Prototype pollution hujumi `[[Set]]` abstract operation orqali ishlaydi — `obj["__proto__"]["isAdmin"] = true` aslida `Object.getPrototypeOf(obj)` ga property qo'shadi, ya'ni `Object.prototype` ga. Spec bo'yicha `__proto__` maxsus accessor property (`Object.prototype.__proto__`) bo'lib, `[[Prototype]]` internal slot'ni o'zgartiradi. `Object.create(null)` bilan yaratilgan object'da bu accessor yo'q — shuning uchun `__proto__` oddiy data property sifatida yoziladi va xavfsiz.

</details>

### 9. Mixin pattern prototype bilan qanday qilinadi? [Senior]

<details>
<summary>Javob</summary>

JavaScript single inheritance — bitta `[[Prototype]]` chain. Mixin — bir nechta source'dan method'larni prototype'ga ko'chirish usuli:

```javascript
const Serializable = {
  serialize() { return JSON.stringify(this); }
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

// Mixin qo'shish:
Object.assign(User.prototype, Serializable, Validatable);

const user = new User("Ali", "ali@mail.com");
user.validate();  // true
user.serialize(); // '{"name":"Ali","email":"ali@mail.com"}'
```

**Deep Dive:**

Mixin muammolari: naming collision (ikki mixin bir xil method nomi), fragile base class problem. Zamonaviy JavaScript'da **composition** ko'pincha mixin'dan yaxshiroq. Bu haqda ko'proq [08-classes.md](08-classes.md) da.

</details>

### 10. Prototype method va instance method farqi nima? [Middle+]

<details>
<summary>Javob</summary>

```javascript
function User(name) {
  this.name = name;
  // ❌ Instance method — har bir instance da alohida funksiya:
  this.greetInstance = function() { return `Salom, ${this.name}`; };
}

// ✅ Prototype method — BITTA funksiya, barcha instance share:
User.prototype.greetProto = function() { return `Salom, ${this.name}`; };

const u1 = new User("Ali");
const u2 = new User("Vali");

console.log(u1.greetInstance === u2.greetInstance); // false — 2 ta funksiya!
console.log(u1.greetProto === u2.greetProto);       // true — 1 ta funksiya!
```

| | Instance Method | Prototype Method |
|-|----------------|-----------------|
| **Memory** | Har instance uchun yangi fn | Bitta fn, hammasi share |
| **1000 instance** | 1000 ta funksiya | 1 ta funksiya |
| **Access tezligi** | Inline cache orqali bir xil tezlikda (V8 IC) | Inline cache orqali bir xil tezlikda (V8 IC) |
| **Private data** | Closure orqali mumkin | Closure bilan mumkin emas |

**Modern V8 eslatma:** Eski materiallar "prototype access biroz sekinroq" deb yozadi, lekin **modern V8 inline caching** monomorphic load'da own va prototype property'ni deyarli bir xil tezlikda hal qiladi. Feedback vector cache'langanidan keyin ikkalasi ham bitta "load by offset" operation'ga kompilyatsiya qilinadi. Haqiqiy performance farqi — **memory allocation** (1000 instance = 1000 funksiya), access tezligi emas.

**Tavsiya:** Default'da prototype method ishlating. Instance method faqat closure orqali private data kerak bo'lganda — yoki method instance-specific state'ni capture qilishi kerak bo'lsa.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle]

**Savol:**

```javascript
function Animal(name) { this.name = name; }
Animal.prototype.speak = function() { return `${this.name} speaks`; };

function Dog(name) { Animal.call(this, name); }
Dog.prototype = Object.create(Animal.prototype);

const rex = new Dog("Rex");

console.log(rex.speak());
console.log(rex.constructor === Dog);
console.log(rex.constructor === Animal);
console.log(rex instanceof Dog);
console.log(rex instanceof Animal);
```

<details>
<summary>Javob</summary>

```
"Rex speaks"
false
true
true
true
```

```javascript
// rex.speak() — "Rex speaks"
// Dog.prototype = Object.create(Animal.prototype) orqali Animal.prototype'dagi
// speak method'ga kirish mumkin. this = rex, rex.name = "Rex".

// rex.constructor === Dog → FALSE
// Chunki Dog.prototype butunlay almashtirildi: Dog.prototype = Object.create(...)
// Yangi object'da constructor yo'q → chain bo'ylab Animal.prototype.constructor topiladi

// rex.constructor === Animal → TRUE
// Animal.prototype.constructor === Animal (default)

// rex instanceof Dog → TRUE — Dog.prototype chain'da bor
// rex instanceof Animal → TRUE — Animal.prototype chain'da bor
```

**Xulosa:** `Dog.prototype = Object.create(Animal.prototype)` qilganda `Dog.prototype.constructor = Dog` ni qayta qo'shish kerak, aks holda constructor yo'qoladi.

</details>

### 2. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
function A() {}
A.prototype.x = 1;

const a = new A();
console.log(a.x);

A.prototype = { x: 2 };

console.log(a.x);

const b = new A();
console.log(b.x);
```

<details>
<summary>Javob</summary>

```
1
1
2
```

```javascript
// a.x → 1
// a yaratilganda a.__proto__ = eski A.prototype = { x: 1 }

// A.prototype = { x: 2 } — yangi object ga almashtirildi
// LEKIN a.__proto__ hali ham ESKI A.prototype ga ishora qiladi!
// a.x → 1 (eski prototype'dan)

// b = new A() — b.__proto__ = YANGI A.prototype = { x: 2 }
// b.x → 2
```

**Xulosa:** `Constructor.prototype` ni almashtirganda, **oldin yaratilgan** instance'lar eski prototype'ga bog'liq bo'lib qoladi. Yangi instance'lar yangi prototype'ga ulangan bo'ladi.

</details>

### 3. Property shadowing nima? Quyidagi kodda nima sodir bo'ladi? [Middle]

**Savol:**

```javascript
const parent = { count: 0 };
const child = Object.create(parent);

child.count++;

console.log(child.count);
console.log(parent.count);
console.log(Object.hasOwn(child, "count"));
```

<details>
<summary>Javob</summary>

```
1
0
true
```

```javascript
// child.count++ bu aslida: child.count = child.count + 1

// O'ng tomondagi child.count — O'QISH (read):
// child'da own "count" yo'q → prototype bo'ylab parent.count = 0 topiladi

// Chap tomondagi child.count = — YOZISH (write):
// Yozish har doim o'z object'iga yoziladi (shadow yaratadi)
// child.count = 0 + 1 = 1

// Natija:
// child.count = 1 (o'zida — shadow property)
// parent.count = 0 (o'zgarmagan!)
// Object.hasOwn(child, "count") = true (child'da own property bor)
```

**Qoida:** Property o'qish — chain bo'ylab yuradi. Property yozish — faqat o'z object'iga yozadi (setter bo'lmasa). Bu shadowing.

</details>

### 4. Bu kodda nima xato? Qanday tuzatish kerak? [Middle]

**Savol:**

```javascript
function Team(name) {
  this.name = name;
}
Team.prototype.members = [];
Team.prototype.addMember = function(member) {
  this.members.push(member);
};

const alpha = new Team("Alpha");
const beta = new Team("Beta");

alpha.addMember("Ali");
console.log(beta.members);
```

<details>
<summary>Javob</summary>

`beta.members` → `["Ali"]` — bu bug!

Muammo: `members` array prototype'da aniqlangan, shuning uchun **barcha instance'lar bitta array'ni share qiladi**. `alpha.addMember("Ali")` — `this.members.push("Ali")` chaqiradi. `this.members` prototype'dagi array topiladi, `push` uni mutate qiladi. Shadow yaratilmaydi chunki bu assignment emas, mavjud array'ning mutatsiyasi.

```javascript
// ✅ To'g'ri usul — mutable data constructor ichida:
function Team(name) {
  this.name = name;
  this.members = []; // ✅ har instance o'z array'i
}
Team.prototype.addMember = function(member) {
  this.members.push(member);
};

const alpha = new Team("Alpha");
const beta = new Team("Beta");
alpha.addMember("Ali");
console.log(beta.members); // [] ✅
```

**Qoida:** Primitive qiymatlar prototype'da xavfsiz (yozishda shadow yaratiladi). Lekin **mutable reference type'lar** (array, object) faqat constructor ichida yarating.

</details>

### 5. `instanceof` ni implement qiling [Middle+]

**Savol:** `myInstanceof(obj, Constructor)` funksiyasini yozing.

<details>
<summary>Javob</summary>

`instanceof` prototype chain bo'ylab `Constructor.prototype` ni qidiradi:

```javascript
function myInstanceof(obj, Constructor) {
  // Primitive uchun false
  if (obj === null || (typeof obj !== "object" && typeof obj !== "function")) {
    return false;
  }

  let proto = Object.getPrototypeOf(obj);
  const target = Constructor.prototype;

  while (proto !== null) {
    if (proto === target) return true;
    proto = Object.getPrototypeOf(proto);
  }

  return false;
}

// Test:
function Parent() {}
function Child() {}
Child.prototype = Object.create(Parent.prototype);

const child = new Child();
console.log(myInstanceof(child, Child));   // true
console.log(myInstanceof(child, Parent));  // true
console.log(myInstanceof(child, Object));  // true
console.log(myInstanceof(child, Array));   // false
console.log(myInstanceof(42, Number));     // false (primitive)
console.log(myInstanceof(null, Object));   // false
```

**Deep Dive:**

ECMAScript spec'da `instanceof` aslida `OrdinaryHasInstance` abstract operation'ni chaqiradi. Agar constructor'da `Symbol.hasInstance` static method aniqlangan bo'lsa — u birinchi chaqiriladi:

```javascript
class EvenNumber {
  static [Symbol.hasInstance](num) {
    return typeof num === "number" && num % 2 === 0;
  }
}
console.log(4 instanceof EvenNumber);  // true
console.log(5 instanceof EvenNumber);  // false
```

</details>

### 6. `new` keyword polyfill yozing [Senior]

**Savol:** `myNew(Constructor, ...args)` funksiyasini yozing — `new` keyword'ning to'liq polyfill'i.

<details>
<summary>Javob</summary>

```javascript
function myNew(Constructor, ...args) {
  // Funksiya ekanligini tekshirish
  if (typeof Constructor !== "function") {
    throw new TypeError(`${Constructor} is not a constructor`);
  }

  // Qadam 1 + 2: Bo'sh object + prototype bog'lash
  const obj = Object.create(Constructor.prototype);

  // Qadam 3: Constructor ni chaqirish (this = obj)
  const result = Constructor.apply(obj, args);

  // Qadam 4: Return logic
  // Object qaytarsa → shu, primitive/undefined/null → obj
  return (result !== null && typeof result === "object") || typeof result === "function"
    ? result
    : obj;
}

// Test 1: Oddiy constructor
function Person(name) { this.name = name; }
Person.prototype.greet = function() { return `Hi ${this.name}`; };

const p = myNew(Person, "Ali");
console.log(p.name);              // "Ali"
console.log(p.greet());           // "Hi Ali"
console.log(p instanceof Person); // true ✅
console.log(p.constructor === Person); // true ✅

// Test 2: Object qaytaruvchi constructor
function Factory() {
  this.a = 1;
  return { b: 2 };
}
const f = myNew(Factory);
console.log(f); // { b: 2 } — this.a yo'qoldi ✅

// Test 3: Function qaytaruvchi constructor
function FnFactory() {
  this.a = 1;
  return function() { return 42; };
}
const fn = myNew(FnFactory);
console.log(typeof fn); // "function" ✅
console.log(fn());      // 42
```

**Deep Dive:**

Return logic'dagi noziklik: `typeof result === "function"` tekshiruvini qo'shdik, chunki function ham object hisoblanadi lekin `typeof function === "function"` (`"object"` emas). Standart `new` operator function qaytarilsa ham uni return qiladi.

**⚠️ Bu polyfill'ning cheklovlari:**
- **`[[IsConstructor]]` internal slot tekshirilmagan** — real `new` operator Constructor'ning `[[IsConstructor]]` slot'ini tekshiradi. Quyidagilar **constructor emas** va `new` bilan chaqirilsa `TypeError` beradi, lekin bu polyfill ularni qabul qiladi:
  - Arrow function: `const f = () => {}` → `new f()` TypeError
  - Method shorthand: `{ foo() {} }.foo` → `new obj.foo()` TypeError
  - Async function: `async function f() {}` → `new f()` TypeError
  - Generator: `function* f() {}` → `new f()` TypeError
- **`new.target` yo'q** — constructor ichida `new.target` har doim `undefined` bo'ladi (real `new` da Constructor'ga teng)

To'liqroq polyfill uchun `Reflect.construct(Constructor, args)` ishlatish mumkin — bu barcha yuqoridagi semantikani spec-accurate qoplaydi.

</details>

### 7. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
const obj = {
  a: 1,
  b: 2
};

const proto = {
  b: 3,
  c: 4
};

Object.setPrototypeOf(obj, proto);

console.log(obj.a);
console.log(obj.b);
console.log(obj.c);
console.log(Object.keys(obj));

for (const key in obj) {
  console.log(key);
}
```

<details>
<summary>Javob</summary>

```
1
2
4
["a", "b"]
a
b
c
```

```javascript
// obj.a = 1 (own property)
// obj.b = 2 (own property — prototype'dagi b:3 ni SHADOW qildi)
// obj.c = 4 (prototype'dan)
// Object.keys(obj) = ["a", "b"] — faqat own enumerable
// for...in — own + prototype enumerable: a, b, c
```

</details>

### 8. Prototype'da `writable: false` bo'lganda nima bo'ladi? [Senior]

**Savol:**

```javascript
const parent = {};
Object.defineProperty(parent, "x", {
  value: 10,
  writable: false,
  configurable: true
});

const child = Object.create(parent);
child.x = 20;

console.log(child.x);
console.log(Object.hasOwn(child, "x"));
```

<details>
<summary>Javob</summary>

```
10
false
```

```javascript
// parent.x writable: false
// ECMAScript spec bo'yicha: agar prototype chain'da writable:false
// property topilsa — child'da shadow yaratish ham TAQIQLANADI
// child.x = 20 → silent fail (strict mode: TypeError)
// child.x → 10 (parent'dan)
// Object.hasOwn(child, "x") → false (child'da own property yaratilMADI)
```

**Deep Dive:**

Bu ECMAScript spec'dagi `[[Set]]` algoritmi (OrdinarySet, qadam 4.d): agar inherited property data descriptor bo'lsa va `writable: false` bo'lsa — assignment `false` qaytaradi. Agar `Object.defineProperty(child, "x", { value: 20 })` desak — bu **ishlaydi**, chunki `defineProperty` `[[Set]]` ni emas, `[[DefineOwnProperty]]` ni chaqiradi.

</details>

### 9. Tuzatish — Prototype inheritance noto'g'ri ishlayapti [Middle]

**Savol:** Quyidagi kodda `Dog` `Animal` dan meros olishi kerak, lekin to'g'ri ishlamayapti. Xatolarni toping:

```javascript
function Animal(name) { this.name = name; }
Animal.prototype.speak = function() {
  return `${this.name} ovoz chiqaradi`;
};

function Dog(name, breed) {
  this.name = name;
  this.breed = breed;
}

Dog.prototype = Animal.prototype;

Dog.prototype.bark = function() {
  return `${this.name} vov-vov deydi`;
};

const dog = new Dog("Bobik", "Labrador");
const cat = new Animal("Mushuk");

console.log(cat.bark()); // Bu ishlamamasligi kerak edi!
```

<details>
<summary>Javob</summary>

3 ta xato:

**Xato 1:** `Dog.prototype = Animal.prototype` — bu ikki constructor'ning prototype'ini **bir xil object** qiladi. Dog'ga qo'shilgan `bark()` Animal'ga ham tushadi.

**Xato 2:** `Animal.call(this, name)` chaqirilmagan — parent constructor'ni chaqirish kerak.

**Xato 3:** `constructor` property qayta o'rnatilmagan.

```javascript
// ✅ To'g'ri kod
function Dog(name, breed) {
  Animal.call(this, name);  // ✅ Parent constructor
  this.breed = breed;
}

Dog.prototype = Object.create(Animal.prototype); // ✅ Yangi object
Dog.prototype.constructor = Dog;                  // ✅ constructor tiklash

Dog.prototype.bark = function() {
  return `${this.name} vov-vov deydi`;
};

const cat = new Animal("Mushuk");
console.log(cat.bark); // undefined ✅ — Animal'da bark yo'q
```

```
❌ Dog.prototype = Animal.prototype
   Dog.prototype ──→ [Animal.prototype]  ← BIR XIL object!

✅ Dog.prototype = Object.create(Animal.prototype)
   Dog.prototype ──→ [yangi object] ──→ [Animal.prototype]
```

</details>

### 10. Tuzatish — Prototype method instance property bilan shadow bo'lib qolgan [Middle+]

**Savol:** `getDiscount()` ba'zi hollarda noto'g'ri ishlayapti. Xatoni toping:

```javascript
function Product(name, price) {
  this.name = name;
  this.price = price;
  if (price > 100000) {
    this.getDiscount = function() {
      return this.price * 0.15;
    };
  }
}

Product.prototype.getDiscount = function() {
  return this.price * 0.05;
};

const laptop = new Product("Laptop", 500000);
console.log(laptop.getDiscount()); // 75000 (15%) ✅

laptop.price = 80000;
console.log(laptop.getDiscount()); // 12000 (15%) ❌ — 5% bo'lishi kerak!
```

<details>
<summary>Javob</summary>

Constructor ichida `this.getDiscount` instance property sifatida yaratilgan — bu prototype'dagi method'ni **shadow** qiladi. Narx o'zgarganda instance property o'chib ketmaydi — doim 15% qaytaradi.

```javascript
// ✅ Barcha logika prototype'da
function Product(name, price) {
  this.name = name;
  this.price = price;
}

Product.prototype.getDiscount = function() {
  return this.price > 100000
    ? this.price * 0.15
    : this.price * 0.05;
};

const laptop = new Product("Laptop", 500000);
console.log(laptop.getDiscount()); // 75000 (15%) ✅

laptop.price = 80000;
console.log(laptop.getDiscount()); // 4000 (5%) ✅
```

</details>

---
