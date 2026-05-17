# Prototypal Inheritance — Interview Savollari

> Prototype chain, `[[Prototype]]`, constructor functions, `new` keyword, `instanceof`, property shadowing va prototype-based meros haqida interview savollari.

---

## Nazariy savollar

### 1. `__proto__` va `prototype` farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

Prototype chain — ob'ektlar bir-biriga `[[Prototype]]` orqali bog'langan zanjir. Object'da property topilmasa, engine bu zanjir bo'ylab **yuqoriga** qidiradi — `null` gacha.

Lookup algoritmi:
1. Object'ning **own property** larida qidiradi
2. Topilmasa — `[[Prototype]]` da qidiradi
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
<summary><strong>Javob</strong></summary>

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
function Product() { this.name = "Laptop"; return 42; }
new Product(); // { name: "Laptop" } — 42 ignored

// Object qaytarsa — this yo'qoladi:
function Config() { this.debug = true; return { env: "production" }; }
new Config(); // { env: "production" } — this.debug yo'qoldi

// null — primitive sifatida ignored:
function Settings() { this.theme = "dark"; return null; }
new Settings(); // { theme: "dark" }
```

</details>

### 4. `Object.create(null)` nima va nima uchun ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

`Object.create(Shape.prototype)` ishlatish sababi — `new Shape()` emas. `new Shape()` desak, Shape constructor side-effect'lari bo'lishi mumkin (API call, DOM manipulation, required parameter validation). `Object.create` faqat `[[Prototype]]` chain'ni ulaydi (`OrdinaryObjectCreate` orqali) — constructor body'ni chaqirmaydi.

Spec darajasida (`Object.create(proto)`, ECMA-262 §20.1.2.3): yangi ordinary object yaratiladi va uning `[[Prototype]]` slot'iga `proto` o'rnatiladi. `new Shape()` esa `Construct(Shape, args)` chaqiradi — bu `[[Construct]]` internal method'ni ishga tushiradi, `OrdinaryCreateFromConstructor` bilan instance yaratadi va keyin Shape body'ni execute qiladi. Inheritance setup paytida body execution kerak emas, faqat chain.

</details>

</details>

### 7. Prototype-based inheritance va class-based inheritance farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

JavaScript **prototype-based** — ob'ektlar to'g'ridan-to'g'ri boshqa **ob'ekt**dan meros oladi. Java/C++ **class-based** — ob'ektlar `class` blueprint'idan yaratiladi.

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

<details>
<summary><strong>Deep Dive</strong></summary>

ES6 `class` sintaksisi kirganidan keyin JavaScript'da ham class-based ko'rinishi bor, lekin bu asosan **syntactic sugar** — ichida xuddi shu prototype mexanizm ishlaydi. `class User { greet() {} }` engine ichida `User.prototype.greet = function() {}` ga aylanadi (method'lar prototype'ga, static'lar constructor function'ga). `extends` esa `Object.setPrototypeOf(Child, Parent)` (static inheritance uchun) + `Child.prototype.[[Prototype]] = Parent.prototype` (instance method inheritance) kombinatsiyasi.

Lekin class'da ba'zi xulq-atvor sugar emas — alohida semantika: constructor'ni `new` bilan chaqirish majburiy (`[[IsClassConstructor]]` slot tekshiriladi, oddiy call'da TypeError), class body'ning ichi har doim strict mode, class declaration'lar hoist bo'lmaydi (TDZ'da turadi), `super` syntax faqat class va object literal method'larda ishlaydi (`HomeObject` slot'iga bog'liq), va `#private` field'lar faqat class'da bor.

</details>

</details>

### 8. Prototype pollution nima va qanday himoya qilish? [Senior]

<details>
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

Prototype pollution hujumi recursive merge orqali `[[Set]]` abstract operation ketma-ketligini ekspluatatsiya qiladi. `merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'))` ichida loop iteratsiya `target["__proto__"]` ni read qiladi — bu `Object.prototype.__proto__` getter accessor orqali `Object.prototype` ni qaytaradi. Recursive call `merge(Object.prototype, { isAdmin: true })` ga aylanadi — keyingi iteratsiyada `Object.prototype["isAdmin"] = true` set qilinadi va global ta'sir bo'ladi.

Spec bo'yicha `__proto__` Annex B.2.2.1 da `Object.prototype` da accessor property sifatida aniqlangan: get `[[GetPrototypeOf]]`, set `[[SetPrototypeOf]]` chaqiradi. `Object.create(null)` bilan yaratilgan object'da bu accessor inherit qilinmaydi — `__proto__` oddiy data property sifatida yoziladi (`[[DefineOwnProperty]]`). Node.js 22+ va modern browser'lar `JSON.parse` paytida `__proto__` key'ni `[[Prototype]]` slot'iga emas, oddiy data property sifatida set qiladi — lekin `merge` funksiyasidagi keyingi `[[Set]]` accessor'ga tushadi va pollution sodir bo'ladi. Himoya: `for...of Object.keys` ishlatish (own enumerable string keys), yoki `Object.create(null)` ni intermediate object sifatida ishlatish.

</details>

</details>

### 9. Mixin pattern prototype bilan qanday qilinadi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

Mixin muammolari: naming collision (ikki mixin bir xil method nomi bo'lsa keyingisi oldingisini override qiladi), fragile base class problem (mixin yangilanganda barcha consumer'lar sinishi mumkin), kuchsiz instanceof (mixin object instanceof bilan tekshirilmaydi — alohida brand check kerak). Zamonaviy JavaScript'da **composition** ko'pincha mixin'dan yaxshiroq — mustaqil object'lar method'larini delegatsiya bilan birlashtirish.

```javascript
// ✅ Composition — mixin o'rniga
function createValidatedUser(name, email) {
  const user = { name, email };
  const validator = {
    validate() { return !!user.email && user.email.includes("@"); }
  };
  const serializer = {
    toJSON() { return JSON.stringify(user); }
  };
  return Object.assign({}, user, validator, serializer);
}
```

`Object.assign(target, ...sources)` ichida `[[Set]]` semantics ishlatiladi — bu degani target prototype'da accessor (`set` defined) bo'lsa, mixin assignment unga tushadi va kutilmagan side effect bo'lishi mumkin. Spec-accurate "data-only" mixin uchun `Object.defineProperties` bilan barcha descriptor'larni ko'chirish kerak: `Object.defineProperties(User.prototype, Object.getOwnPropertyDescriptors(Mixin))` — bu enumerable, non-enumerable, getter/setter va Symbol key'larni ham to'g'ri ko'chiradi.

</details>

</details>

### 10. Prototype method va instance method farqi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

ECMAScript spec'da `instanceof` operator (ECMA-262 §13.10.2) `InstanceofOperator(V, target)` abstract operation'ni chaqiradi. Bu birinchi `GetMethod(target, @@hasInstance)` orqali `Symbol.hasInstance` mavjudligini tekshiradi — agar bor bo'lsa, uni `target` ga `V` bilan chaqiradi va boolean qaytaradi. Aks holda `OrdinaryHasInstance(target, V)` chaqiriladi — bu prototype chain bo'ylab `target.prototype` ni qidiradi (bound function'lar uchun `[[BoundTargetFunction]]` recursive unwrap qilinadi).

```javascript
class EvenNumber {
  static [Symbol.hasInstance](num) {
    return typeof num === "number" && num % 2 === 0;
  }
}
console.log(4 instanceof EvenNumber);  // true
console.log(5 instanceof EvenNumber);  // false
```

Bizning `myInstanceof` polyfill `Symbol.hasInstance` override'ni handle qilmaydi va arrow function'larda fail bo'ladi (arrow function'da `prototype` property yo'q — `target.prototype === undefined`). To'liq spec-accurate polyfill `Symbol.hasInstance` check'ni boshida qo'shishi va `prototype` property mavjudligini validate qilishi kerak.

</details>

</details>

### 6. `new` keyword polyfill yozing [Senior]

**Savol:** `myNew(Constructor, ...args)` funksiyasini yozing — `new` keyword'ning to'liq polyfill'i.

<details>
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

Return logic'dagi noziklik: `typeof result === "function"` tekshiruvini qo'shdik, chunki function ham object hisoblanadi (spec'da `Object` tipi callable bo'lishi mumkin), lekin `typeof function === "function"` (`"object"` emas). Standart `new` operator function qaytarilsa ham uni return qiladi — spec test `Type(result) is Object` (function ham Object type'iga kiradi).

Real `new` spec algoritmi (`[[Construct]]`, ECMA-262 §10.2.2):

1. `OrdinaryCreateFromConstructor(newTarget, "%Object.prototype%")` — `[[Prototype]]` `newTarget.prototype` ga teng ordinary object yaratiladi
2. `PrepareForOrdinaryCall(F, newTarget)` — yangi calleeContext yaratiladi, `[[ThisValue]]` instance'ga bind qilinadi
3. Function body bajariladi, result olinadi
4. Agar result `Object` type bo'lsa — uni qaytaradi, aks holda yangi instance'ni

`new.target` — calleeContext'ning `[[NewTarget]]` slot'idan o'qiladi (function chaqirilganda `[[Call]]` bilan oddiy chaqiruvda `undefined`, `[[Construct]]` bilan chaqirilganda constructor reference). Bu xulq-atvor `Reflect.construct(target, args, newTarget)` orqali kuzatiladi.

**⚠️ Bu polyfill'ning cheklovlari:**
- **`[[IsConstructor]]` internal slot tekshirilmagan** — real `new` operator Constructor'ning `[[IsConstructor]]` slot'ini tekshiradi. Quyidagilar **constructor emas** va `new` bilan chaqirilsa `TypeError` beradi, lekin bu polyfill ularni qabul qiladi:
  - Arrow function: `const f = () => {}` → `new f()` TypeError
  - Method shorthand: `{ greet() {} }.greet` → `new obj.greet()` TypeError
  - Async function: `async function f() {}` → `new f()` TypeError
  - Generator: `function* f() {}` → `new f()` TypeError
- **`new.target` yo'q** — constructor ichida `new.target` har doim `undefined` bo'ladi (real `new` da Constructor'ga teng)

To'liqroq polyfill uchun `Reflect.construct(Constructor, args)` ishlatish mumkin — bu barcha yuqoridagi semantikani spec-accurate qoplaydi.

</details>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

Bu ECMAScript spec'dagi `OrdinarySet` algoritmi (ECMA-262 §10.1.9.2) bilan tushuntiriladi. `OrdinarySetWithOwnDescriptor(O, P, V, Receiver, ownDesc)` ichida agar `ownDesc` `undefined` bo'lsa (own property yo'q), parent ga delegatsiya qiladi. Parent'da property data descriptor va `[[Writable]]` `false` bo'lsa — algoritm `false` qaytaradi (yoki strict mode'da TypeError). Sabab: assignment semantikasi inherited read-only invariant'ni saqlash kerak — agar parent'da read-only bo'lsa, child shadow yarata olmaydi.

Lekin `Object.defineProperty(child, "x", { value: 20 })` **ishlaydi**, chunki `defineProperty` `[[Set]]` ni emas, `[[DefineOwnProperty]]` ni chaqiradi (ECMA-262 §10.1.6). `[[DefineOwnProperty]]` parent'ning writable atributini tekshirmaydi — faqat o'z object'idagi mavjud property'ni va `[[Extensible]]` flag'ni hisobga oladi. Bu farq sintaktik shadowing va explicit descriptor definition orasidagi semantik chegarani belgilaydi.

</details>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

### 11. `Object.getPrototypeOf` vs `__proto__` — qaysi biri afzal va nima uchun? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Ikkalasi ham object'ning `[[Prototype]]` internal slot'ini o'qiydi, lekin standart va xulq-atvor jihatdan farqli:

| | `Object.getPrototypeOf(obj)` | `obj.__proto__` |
|-|----------------------------|-----------------|
| **Standart** | ECMAScript core | Annex B (legacy web compat) |
| **`Object.create(null)`** | ✅ ishlaydi (`null` qaytaradi) | ❌ `undefined` — accessor yo'q |
| **Override mumkin** | ❌ static method | ✅ ha (`Object.prototype.__proto__` accessor) |
| **Performance** | Bir xil — ikkalasi `[[GetPrototypeOf]]` ni chaqiradi | Bir xil |
| **Type system (TS)** | Aniq tip | `any` |

```javascript
const obj = { name: "Alice" };

// ✅ Tavsiya — har doim ishlaydi:
console.log(Object.getPrototypeOf(obj) === Object.prototype); // true

// ⚠️ Legacy — Object.create(null)'da yo'q:
console.log(obj.__proto__ === Object.prototype); // true

const bare = Object.create(null);
console.log(Object.getPrototypeOf(bare)); // null ✅
console.log(bare.__proto__);              // undefined ❌
```

**Yana bir farq** — `__proto__` accessor `this` binding'ga sezgir:

```javascript
const desc = Object.getOwnPropertyDescriptor(Object.prototype, "__proto__");
// desc.get / desc.set — accessor functions
// Object.create(null) bilan yaratilgan object'da bu accessor inherit qilinmaydi
```

**Qoida:** Zamonaviy kodda doim `Object.getPrototypeOf()` va `Object.setPrototypeOf()` ishlating. `__proto__` faqat eski browser compatibility uchun.

<details>
<summary><strong>Deep Dive</strong></summary>

`__proto__` accessor `Object.prototype` da getter/setter sifatida aniqlangan (ECMA-262 Annex B.2.2.1). Annex B `normative-optional` bo'lim — non-browser implementation'lar uchun majburiy emas, lekin web compatibility tufayli har bir engine (V8, SpiderMonkey, JSC) implement qiladi. `Object.getPrototypeOf` esa core spec'da (§20.1.2.12) — har doim mavjud. Performance jihatidan ikkalasi ham V8'da `[[GetPrototypeOf]]` internal method'ni chaqiradi, IC bilan optimallashadi — sezilarli farq yo'q.

Asosiy tanlov sabablari: **xavfsizlik** (`Object.create(null)` da `__proto__` ishlamaydi — lookup `undefined` qaytaradi va silent bug yaratishi mumkin), **portability** (Annex B'siz environment'larda yo'q bo'lishi mumkin), va **TypeScript** (`__proto__` ning tipi `any`, `Object.getPrototypeOf` esa `object | null` qaytaradi — type-safe).

</details>

</details>

### 12. Object.prototype'da nima metodlar bor? `isPrototypeOf` qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`Object.prototype` — barcha (default prototype chain'li) object'larning eng yuqori ajdodi. Unda quyidagi method'lar bor:

- **`toString()`** — `"[object Object]"` qaytaradi (override mumkin)
- **`valueOf()`** — object'ning primitive konvertatsiyasi
- **`hasOwnProperty(key)`** — property own ekanligini tekshiradi
- **`isPrototypeOf(obj)`** — `obj` ning prototype chain'ida `this` bormi
- **`propertyIsEnumerable(key)`** — property enumerable own ekanligini
- **`__proto__`** (Annex B) — `[[Prototype]]` accessor

```javascript
const animal = { eats: true };
const dog = Object.create(animal);
const puppy = Object.create(dog);

// isPrototypeOf — instance method, this'dan boshlab chain bo'ylab qidiradi:
console.log(animal.isPrototypeOf(puppy));  // true (animal → dog → puppy)
console.log(dog.isPrototypeOf(puppy));     // true (bevosita parent)
console.log(puppy.isPrototypeOf(animal));  // false — teskari emas

// `instanceof` bilan farq — instanceof Constructor.prototype'ni qidiradi:
console.log(animal.isPrototypeOf(puppy)); // animal object'ni qidiradi
// instanceof bilan ekvivalenti yo'q (instanceof faqat Constructor function bilan)
```

**Algoritm** (`isPrototypeOf` ichida — spec OrdinaryHasInstance ga o'xshash):

```javascript
function isPrototypeOf(target) {
  let proto = Object.getPrototypeOf(target);
  while (proto !== null) {
    if (proto === this) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}
```

**`instanceof` bilan asosiy farq:**

```javascript
function Animal() {}
const a = new Animal();

// instanceof Constructor.prototype ni qidiradi:
console.log(a instanceof Animal);  // true — Animal.prototype chain'da

// isPrototypeOf — bevosita object'ni qidiradi:
console.log(Animal.prototype.isPrototypeOf(a));  // true — bir xil natija
console.log(Animal.isPrototypeOf(a));            // false! — Animal function, .prototype emas
```

**Use case:** `isPrototypeOf` constructor'siz prototype chain bilan ishlaganda foydali (`Object.create` pattern, polymorphic API'lar).

</details>

### 13. Quyidagi kodda nima sodir bo'ladi? [Senior]

**Savol:**

```javascript
const parent = {
  greet() { return `Salom, ${this.name}`; }
};

const child = Object.create(parent);
child.name = "Alice";

const greet = child.greet;
console.log(greet());
console.log(child.greet());
console.log(child.greet.call({ name: "Bob" }));
```

<details>
<summary><strong>Javob</strong></summary>

```
"Salom, undefined"  (yoki TypeError strict mode'da)
"Salom, Alice"
"Salom, Bob"
```

**Tushuntirish:**

```javascript
// 1. const greet = child.greet — method'ni "ajratib oldik"
//    Bu prototype chain bo'ylab parent.greet'ni topadi, lekin
//    method bilan birga this bog'lanmagan — faqat function reference
//
// 2. greet() — bog'lanmagan function call, this = undefined (strict) yoki globalThis
//    globalThis.name undefined → "Salom, undefined"
//    (Browser'da globalThis.name = "" bo'lishi mumkin → "Salom, ")
//
// 3. child.greet() — to'g'ri method call, this = child
//    parent.greet topiladi (delegation), this = child, child.name = "Alice"
//    Natija: "Salom, Alice"
//
// 4. child.greet.call({ name: "Bob" }) — explicit this binding
//    this = { name: "Bob" }, natija: "Salom, Bob"
```

**Asosiy qoida:** Method "ajratib olinganda" `this` binding yo'qoladi. Bu prototype'dan kelgan method bilan ham ishlaydi — chunki delegation faqat lookup'ga ta'sir qiladi, `this` binding esa call-site bilan belgilanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

Bu xulq-atvor JavaScript'dagi `this` binding'ning dynamic ekanligidan kelib chiqadi. Spec bo'yicha `EvaluateCall(func, ref, arguments, tailPosition)` (ECMA-262 §13.3.6.2) ichida: agar `ref` Reference Record bo'lsa va base Object — `thisValue = GetThisValue(ref)` (base object), aks holda `thisValue = undefined`. `child.greet` member access expression Reference Record qaytaradi (`{ Base: child, ReferencedName: "greet" }`), shuning uchun `child.greet()` → `thisValue = child`. Lekin `const greet = child.greet` `GetValue` chaqiradi va Reference Record'ni function value'ga aylantiradi — `greet()` chaqiruvida hech qanday base yo'q, `thisValue = undefined`.

Method shorthand (`greet() {}`) va `function` expression ikkalasi ham bu xulq-atvor'ga ega — `[[ThisMode]]` `"strict"` yoki `"global"`. Arrow function esa farq qiladi — `[[ThisMode]]` `"lexical"`, `OrdinaryCallBindThis` no-op qiladi, va `this` resolving outer Environment Record'ga delegatsiya qiladi. Shu sababli arrow function'ni `call`/`apply`/`bind` bilan o'zgartirib bo'lmaydi.

Real-world muammoda yechim:
- **Explicit bind:** `const greet = child.greet.bind(child);` — `BoundFunctionCreate` orqali yangi function object yaratiladi, `[[BoundThis]]` slot'iga `child` saqlanadi
- **Arrow wrapper:** `const greet = () => child.greet();` — closure orqali `child` ushlab qoladi
- **Class field arrow** (instance method): `class Foo { greet = () => {...} }` — har instance uchun bound copy (memory cost'i bor)

</details>

</details>

---
