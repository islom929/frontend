# Bo'lim 6: Objects Ichidan

> JavaScript da deyarli hamma narsa object — ularni chuqur tushunish butun tilni tushunish demak.

---

## Mundarija

- [Object Creation Patterns](#object-creation-patterns)
- [Property Descriptors](#property-descriptors)
- [Getters va Setters](#getters-va-setters)
- [Object Immutability](#object-immutability)
- [Object Copying](#object-copying)
- [Property Enumeration](#property-enumeration)
- [Computed Properties va Optional Chaining](#computed-properties-va-optional-chaining)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Object Creation Patterns

### 1. Object Literal (Eng Ko'p Ishlatiladi)

```javascript
const user = {
  name: "Islom",
  age: 25,
  greet() {
    return `Salom, men ${this.name}man`;
  }
};
```

### 2. Constructor Function

```javascript
function User(name, age) {
  this.name = name;
  this.age = age;
}
User.prototype.greet = function() {
  return `Salom, men ${this.name}man`;
};

const user = new User("Islom", 25);
```

### 3. Object.create()

```javascript
const userProto = {
  greet() {
    return `Salom, men ${this.name}man`;
  }
};

const user = Object.create(userProto);
user.name = "Islom";
user.age = 25;

// yoki bir qadamda:
const user2 = Object.create(userProto, {
  name: { value: "Ali", writable: true, enumerable: true, configurable: true },
  age:  { value: 30, writable: true, enumerable: true, configurable: true }
});
```

### 4. Class (ES6)

```javascript
class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  greet() {
    return `Salom, men ${this.name}man`;
  }
}

const user = new User("Islom", 25);
```

### Taqqoslash

| Pattern | Qachon ishlatish |
|---------|-----------------|
| **Literal** | Oddiy, bir martalik object'lar |
| **Constructor** | `new` bilan ko'p instance, ES5 kod |
| **Object.create** | Aniq prototype chain kerak bo'lganda |
| **Class** | OOP, inheritance, zamonaviy kod |

---

## Property Descriptors

### Nazariya

JavaScript'da biz object property'siga qiymat berganimizda (`obj.name = "Islom"`), bu oddiy ko'rinadigan operatsiya ortida murakkab mexanizm yashiringan. Har bir property aslida oddiy qiymatdan **ancha ko'proq** ma'lumotga ega — u **Property Descriptor** deb ataluvchi meta-ma'lumot ob'ekti bilan birga saqlanadi.

Property Descriptor — bu property xulq-atvorini belgilaydigan to'rtta flag dan iborat: `value` (qiymat), `writable` (o'zgartirish mumkinmi), `enumerable` (`for...in` da ko'rinadimi) va `configurable` (o'chirib yoki qayta sozlash mumkinmi). Buni **fayl tizimi ruxsatnomasi**ga o'xshatish mumkin: faylda faqat mazmun emas, balki "o'qish mumkin", "yozish mumkin", "bajarish mumkin" kabi meta-ma'lumotlar ham bor. Property descriptor ham xuddi shunday — qiymat va uning ustida nima qilish mumkinligini belgilaydi.

Nima uchun bu muhim? Real-world dasturlashda siz ba'zan property'ni faqat o'qish uchun (read-only) qilishingiz kerak (masalan, ob'ekt `id` si), ba'zan esa property'ni `Object.keys()` va `for...in` dan yashirishingiz kerak (masalan, ichki `_private` property'lar). Framework'lar (Vue.js ning reactivity tizimi, MobX) ham descriptor'lar orqali ishlaydi — ular `Object.defineProperty` yordamida property'larga getter/setter o'rnatib, qiymat o'zgarganda UI ni yangilaydi.

| Flag | Default | Ma'nosi |
|------|---------|---------|
| **`value`** | `undefined` | Property qiymati |
| **`writable`** | `true` | Qiymatni o'zgartirish mumkinmi |
| **`enumerable`** | `true` | `for...in` / `Object.keys` da ko'rinadimi |
| **`configurable`** | `true` | O'chirib yuborish yoki descriptor'ni o'zgartirish mumkinmi |

### Object.getOwnPropertyDescriptor()

```javascript
const user = { name: "Islom", age: 25 };

console.log(Object.getOwnPropertyDescriptor(user, "name"));
// {
//   value: "Islom",
//   writable: true,
//   enumerable: true,
//   configurable: true
// }

// Barcha property'lar uchun:
console.log(Object.getOwnPropertyDescriptors(user));
// {
//   name: { value: "Islom", writable: true, enumerable: true, configurable: true },
//   age:  { value: 25, writable: true, enumerable: true, configurable: true }
// }
```

### Object.defineProperty()

```javascript
const user = {};

Object.defineProperty(user, "id", {
  value: 1,
  writable: false,      // o'zgartirib bo'lmaydi
  enumerable: false,     // for...in da ko'rinmaydi
  configurable: false    // o'chirib bo'lmaydi, descriptor o'zgarmaydi
});

console.log(user.id);     // 1
user.id = 2;               // ❌ Silent fail (strict mode da TypeError)
console.log(user.id);     // 1 — o'zgarmadi

console.log(Object.keys(user)); // [] — enumerable: false

delete user.id;            // ❌ Silent fail
console.log(user.id);     // 1 — o'chirilmadi
```

⚠️ **Muhim:** `Object.defineProperty` bilan yaratilgan property'larda `writable`, `enumerable`, `configurable` default **`false`**! Oddiy assignment (`.` yoki `[]`) bilan yaratilganda esa default **`true`**.

### Object.defineProperties()

```javascript
const product = {};

Object.defineProperties(product, {
  name: {
    value: "Laptop",
    writable: true,
    enumerable: true,
    configurable: true
  },
  _price: {
    value: 1000,
    writable: true,
    enumerable: false,    // "private" — ko'rinmaydi
    configurable: false
  },
  id: {
    value: "PRD-001",
    writable: false,      // read-only
    enumerable: true,
    configurable: false
  }
});

console.log(Object.keys(product));  // ["name", "id"] — _price yo'q (enumerable: false)
product.name = "MacBook";           // ✅
product.id = "PRD-002";             // ❌ writable: false
```

### Under the Hood

Aslida object'dagi **har bir** property descriptor bilan saqlanadi — biz ko'rmaymiz, lekin engine ichida mavjud. Oddiy assignment:

```javascript
obj.x = 5;
// Huddi shu:
Object.defineProperty(obj, "x", {
  value: 5,
  writable: true,
  enumerable: true,
  configurable: true
});
```

---

## Getters va Setters

### Nazariya

Getter va Setter — bu **accessor properties** (kirish xossalari) bo'lib, ular tashqi ko'rinishda oddiy property kabi ishlaydi, lekin ichida funksiya bajariladi. `get` — property o'qilganda, `set` — property ga qiymat yozilganda chaqiriladi.

Nima uchun getter/setter kerak? Oddiy property'da qiymatni o'qish va yozish cheklanmagan — istalgan qiymat berilishi mumkin. Lekin real-world da ko'pincha **validatsiya** (masalan, yosh manfiy bo'lishi mumkin emas), **computed values** (masalan, `fullName = firstName + lastName`) yoki **side effects** (masalan, qiymat o'zgarganda log yozish) kerak bo'ladi. Getter va Setter aynan shu ehtiyojlarni qondiradi — ular property'ga **aqlli interfeys** beradi.

Bu pattern zamonaviy JavaScript ekotizimida keng tarqalgan. Vue.js 2.x butun reaktivlik tizimini `Object.defineProperty` orqali getter/setter'lar bilan qurgan. Class'larda private property'larga xavfsiz kirish uchun getter/setter ishlatiladi. Node.js stream'larida, Express middleware'larida — hamma joyda accessor pattern'lar uchraydi.

```javascript
const user = {
  firstName: "Islom",
  lastName: "Karimov",

  // Getter — o'qish
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  },

  // Setter — yozish
  set fullName(value) {
    const parts = value.split(" ");
    this.firstName = parts[0];
    this.lastName = parts[1];
  }
};

// Oddiy property kabi ishlatiladi:
console.log(user.fullName);        // "Islom Karimov" (getter chaqirildi)
user.fullName = "Ali Valiyev";     // setter chaqirildi
console.log(user.firstName);       // "Ali"
console.log(user.lastName);        // "Valiyev"
```

### Getter/Setter Descriptor

Accessor property'ning descriptor'i boshqacha:

```javascript
// Data property:     { value, writable, enumerable, configurable }
// Accessor property: { get, set, enumerable, configurable }
// Ikkalasi BIRGA bo'lmaydi! value/writable va get/set — o'zaro exclusive.

const desc = Object.getOwnPropertyDescriptor(user, "fullName");
// {
//   get: function fullName() {...},
//   set: function fullName(value) {...},
//   enumerable: true,
//   configurable: true
// }
```

### Validation bilan Setter

```javascript
const account = {
  _balance: 0,

  get balance() {
    return this._balance;
  },

  set balance(value) {
    if (typeof value !== "number") {
      throw new TypeError("Balance raqam bo'lishi kerak");
    }
    if (value < 0) {
      throw new RangeError("Balance manfiy bo'lishi mumkin emas");
    }
    this._balance = value;
  }
};

account.balance = 100;        // ✅
console.log(account.balance); // 100
account.balance = -50;        // ❌ RangeError
account.balance = "yuz";      // ❌ TypeError
```

### defineProperty bilan Getter/Setter

```javascript
const obj = { _temp: 36.6 };

Object.defineProperty(obj, "temperature", {
  get() {
    return `${this._temp}°C`;
  },
  set(celsius) {
    this._temp = celsius;
  },
  enumerable: true,
  configurable: true
});

obj.temperature = 38;
console.log(obj.temperature); // "38°C"
```

---

## Object Immutability

JavaScript da object'ni "muzlatish" uchun 3 daraja bor:

### 1. Object.preventExtensions()

Yangi property **qo'shib bo'lmaydi**, lekin mavjudlarini o'zgartirish/o'chirish mumkin.

```javascript
const obj = { a: 1, b: 2 };
Object.preventExtensions(obj);

obj.c = 3;          // ❌ Silent fail (strict: TypeError)
console.log(obj.c); // undefined — qo'shilmadi

obj.a = 10;         // ✅ O'zgartirish mumkin
delete obj.b;       // ✅ O'chirish mumkin

console.log(Object.isExtensible(obj)); // false
```

### 2. Object.seal()

Yangi property **qo'shib bo'lmaydi**, mavjudlarni **o'chirib bo'lmaydi**, lekin qiymatini **o'zgartirish mumkin**.

```javascript
const obj = { a: 1, b: 2 };
Object.seal(obj);

obj.c = 3;          // ❌ qo'shilmaydi
delete obj.b;       // ❌ o'chirilmaydi
obj.a = 10;         // ✅ qiymat o'zgartiriladi

console.log(Object.isSealed(obj)); // true
```

Ichki implementatsiya: `preventExtensions` + barcha property'larga `configurable: false`.

### 3. Object.freeze()

**Hech narsa** o'zgarmaydi — yangi property yo'q, o'chirish yo'q, o'zgartirish yo'q.

```javascript
const obj = { a: 1, b: 2 };
Object.freeze(obj);

obj.c = 3;          // ❌
delete obj.b;       // ❌
obj.a = 10;         // ❌

console.log(Object.isFrozen(obj)); // true
```

Ichki implementatsiya: `seal` + barcha property'larga `writable: false`.

### Taqqoslash Jadvali

| Xususiyat | preventExtensions | seal | freeze |
|-----------|-------------------|------|--------|
| **Yangi property** | ❌ | ❌ | ❌ |
| **Property o'chirish** | ✅ | ❌ | ❌ |
| **Qiymat o'zgartirish** | ✅ | ✅ | ❌ |
| **Descriptor o'zgartirish** | ✅ | ❌ | ❌ |

### Muammo: Shallow Freeze

```javascript
const config = {
  db: { host: "localhost", port: 5432 },
  api: { url: "https://api.example.com" }
};

Object.freeze(config);

config.db = {};               // ❌ o'zgarmaydi
config.db.host = "remote";    // ✅ O'ZGARADI! 😱
console.log(config.db.host);  // "remote" — ichki object freeze bo'lmagan
```

`Object.freeze` **shallow** — faqat birinchi daraja. Ichki (nested) object'lar freeze bo'lmaydi.

### Deep Freeze

```javascript
function deepFreeze(obj) {
  Object.freeze(obj);

  Object.getOwnPropertyNames(obj).forEach(prop => {
    const value = obj[prop];
    if (value !== null && typeof value === "object" && !Object.isFrozen(value)) {
      deepFreeze(value);
    }
  });

  return obj;
}

const config = deepFreeze({
  db: { host: "localhost", port: 5432 },
  api: { url: "https://api.example.com" }
});

config.db.host = "remote"; // ❌ Endi o'zgarmaydi!
```

---

## Object Copying

### Shallow Copy — Sayoz Nusxa

Birinchi daraja nusxalanadi, ichki object'lar **reference** bo'lib qoladi.

#### Spread Operator

```javascript
const original = { a: 1, b: { c: 2 } };
const copy = { ...original };

copy.a = 10;
console.log(original.a); // 1 ✅ — alohida

copy.b.c = 20;
console.log(original.b.c); // 20 ❌ — bir xil reference!
```

#### Object.assign()

```javascript
const original = { a: 1, b: { c: 2 } };
const copy = Object.assign({}, original);

// Xuddi spread kabi — shallow
copy.b.c = 20;
console.log(original.b.c); // 20 — bir xil reference
```

```
Shallow Copy:

original:                  copy:
┌──────────────┐          ┌──────────────┐
│ a: 1         │          │ a: 1         │  ← alohida
│ b: ──────────│────┐     │ b: ──────────│────┐
└──────────────┘    │     └──────────────┘    │
                    ▼                          ▼
              ┌──────────┐  ← BIR XIL object!
              │ { c: 2 } │
              └──────────┘
```

### Deep Copy — Chuqur Nusxa

#### structuredClone() (Zamonaviy, Tavsiya)

```javascript
const original = {
  a: 1,
  b: { c: 2 },
  d: [1, 2, 3],
  e: new Date(),
  f: new Map([["key", "value"]]),
  g: new Set([1, 2, 3])
};

const copy = structuredClone(original);

copy.b.c = 20;
console.log(original.b.c); // 2 ✅ — to'liq alohida!

// structuredClone QUVVATLAMAYDIGAN narsalar:
// ❌ Functions
// ❌ DOM Nodes
// ❌ Symbols (property key sifatida)
// ❌ Property descriptors (writable, enumerable, etc. saqlanmaydi)
// ❌ Prototype chain
```

#### JSON.parse + JSON.stringify (Eski Hack)

```javascript
const original = { a: 1, b: { c: 2 } };
const copy = JSON.parse(JSON.stringify(original));

copy.b.c = 20;
console.log(original.b.c); // 2 ✅

// ❌ Muammolari:
// undefined → yo'qoladi
// Function → yo'qoladi
// Date → string ga aylanadi
// RegExp → bo'sh object
// Map, Set → bo'sh object
// Infinity, NaN → null
// Circular reference → Error!

const bad = {
  fn: () => {},           // yo'qoladi
  date: new Date(),       // string bo'ladi
  undef: undefined,       // yo'qoladi
  regex: /hello/gi,       // {} bo'ladi
};
JSON.parse(JSON.stringify(bad));
// { date: "2026-02-06T...", regex: {} } — fn va undef yo'qoldi
```

#### Recursive Deep Copy

```javascript
function deepClone(obj, seen = new WeakMap()) {
  // Primitive yoki null
  if (obj === null || typeof obj !== "object") return obj;

  // Circular reference
  if (seen.has(obj)) return seen.get(obj);

  // Date
  if (obj instanceof Date) return new Date(obj);

  // RegExp
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);

  // Array yoki Object
  const copy = Array.isArray(obj) ? [] : {};
  seen.set(obj, copy);

  for (const key of Reflect.ownKeys(obj)) {
    copy[key] = deepClone(obj[key], seen);
  }

  return copy;
}

// Circular reference ham ishlaydi:
const obj = { a: 1 };
obj.self = obj;
const cloned = deepClone(obj);
console.log(cloned.self === cloned); // true ✅ (o'ziga ishora, original emas)
```

### Taqqoslash

| Method | Chuqurlik | Circular | Function | Date | Performance |
|--------|-----------|----------|----------|------|-------------|
| Spread / Object.assign | Shallow | — | ✅ | ✅ (ref) | Eng tez |
| structuredClone | Deep | ✅ | ❌ | ✅ | Tez |
| JSON hack | Deep | ❌ | ❌ | ❌ | O'rta |
| Recursive | Deep | ✅ | ✅ | ✅ | Sekin |

**Tavsiya:** `structuredClone` — aksariyat holatlarda yetarli. Function yoki class instance kerak bo'lsa — recursive yoki kutubxona (lodash `_.cloneDeep`).

---

## Property Enumeration

### for...in

Barcha **enumerable** property'larni ko'rsatadi — **prototype chain** bo'ylab ham!

```javascript
const parent = { inherited: true };
const child = Object.create(parent);
child.own = "mine";

for (const key in child) {
  console.log(key);
}
// "own"
// "inherited" ← prototype dan!

// Faqat o'zining property'lari uchun:
for (const key in child) {
  if (child.hasOwnProperty(key)) {
    console.log(key); // faqat "own"
  }
}
```

### Object.keys / values / entries

Faqat **own** + **enumerable** property'lar:

```javascript
const user = { name: "Islom", age: 25 };
Object.defineProperty(user, "secret", {
  value: "hidden",
  enumerable: false
});

Object.keys(user);    // ["name", "age"] — secret yo'q
Object.values(user);  // ["Islom", 25]
Object.entries(user);  // [["name", "Islom"], ["age", 25]]
```

### hasOwnProperty vs in

```javascript
const obj = Object.create({ inherited: true });
obj.own = "mine";

"own" in obj;             // true — own + prototype
"inherited" in obj;       // true — prototype da bor

obj.hasOwnProperty("own");       // true
obj.hasOwnProperty("inherited"); // false — prototype'da, own emas

// Xavfsiz variant (hasOwnProperty override qilingan bo'lsa):
Object.hasOwn(obj, "own");       // true (ES2022)
```

### Enumeration Methodlar Taqqoslash

| Method | Own only | Enumerable only | Prototype | Symbol |
|--------|----------|-----------------|-----------|--------|
| `for...in` | ❌ | ✅ | ✅ | ❌ |
| `Object.keys()` | ✅ | ✅ | ❌ | ❌ |
| `Object.getOwnPropertyNames()` | ✅ | ❌ | ❌ | ❌ |
| `Object.getOwnPropertySymbols()` | ✅ | ❌ | ❌ | ✅ |
| `Reflect.ownKeys()` | ✅ | ❌ | ❌ | ✅ |

```javascript
const sym = Symbol("id");
const obj = { a: 1, [sym]: 2 };
Object.defineProperty(obj, "hidden", { value: 3, enumerable: false });

Object.keys(obj);                       // ["a"]
Object.getOwnPropertyNames(obj);        // ["a", "hidden"]
Object.getOwnPropertySymbols(obj);      // [Symbol(id)]
Reflect.ownKeys(obj);                   // ["a", "hidden", Symbol(id)]
```

---

## Computed Properties va Optional Chaining

### Computed Property Names

```javascript
const field = "name";
const prefix = "user";

const obj = {
  [field]: "Islom",                    // obj.name = "Islom"
  [`${prefix}Age`]: 25,               // obj.userAge = 25
  [`get${field.charAt(0).toUpperCase() + field.slice(1)}`]() {
    return this[field];
  }  // obj.getName()
};

console.log(obj.name);        // "Islom"
console.log(obj.userAge);     // 25
console.log(obj.getName());   // "Islom"
```

### Dynamic Property Access

```javascript
function getProperty(obj, path) {
  return path.split(".").reduce((acc, key) => acc?.[key], obj);
}

const data = {
  user: {
    address: {
      city: "Toshkent"
    }
  }
};

getProperty(data, "user.address.city"); // "Toshkent"
getProperty(data, "user.phone.number"); // undefined (xato emas)
```

### Optional Chaining (`?.`)

```javascript
const user = {
  name: "Islom",
  address: {
    city: "Toshkent"
  }
};

// ❌ Eski usul — uzun tekshirish
const zip = user && user.address && user.address.zip;

// ✅ Optional chaining
const zip2 = user?.address?.zip;         // undefined (xato emas)
const city = user?.address?.city;        // "Toshkent"

// Method chaqirishda:
user?.greet?.();          // undefined (greet yo'q — xato emas)

// Bracket notation da:
const key = "name";
user?.[key];              // "Islom"

// Array da:
const arr = null;
arr?.[0];                 // undefined (xato emas)
```

---

## Common Mistakes

### ❌ Xato 1: Shallow copy bilan deep copy ni aralashtirib yuborish

```javascript
const config = {
  server: { host: "localhost", port: 3000 }
};

const backup = { ...config }; // SHALLOW!
backup.server.port = 4000;

console.log(config.server.port); // 4000 — original O'ZGARDI!
```

### ✅ To'g'ri usul:

```javascript
const backup = structuredClone(config);
backup.server.port = 4000;
console.log(config.server.port); // 3000 ✅ — alohida
```

**Nima uchun:** Spread faqat birinchi darajani nusxalaydi. Ichki object'lar reference bo'lib qoladi — ikkalasi bitta object'ga ishora qiladi.

---

### ❌ Xato 2: Object.freeze shallow ekanini unutish

```javascript
const settings = Object.freeze({
  theme: { color: "dark" }
});

settings.theme.color = "light"; // ✅ O'zgaradi! Freeze shallow.
```

### ✅ To'g'ri usul:

```javascript
function deepFreeze(obj) {
  Object.freeze(obj);
  Object.getOwnPropertyNames(obj).forEach(prop => {
    const val = obj[prop];
    if (val && typeof val === "object" && !Object.isFrozen(val)) {
      deepFreeze(val);
    }
  });
  return obj;
}

const settings = deepFreeze({ theme: { color: "dark" } });
settings.theme.color = "light"; // ❌ Endi o'zgarmaydi
```

**Nima uchun:** `Object.freeze` birinchi darajani muzlatadi. Ichki object hali **mutable**. Deep freeze kerak bo'lsa — recursive.

---

### ❌ Xato 3: for...in bilan prototype property ko'rish

```javascript
function User(name) { this.name = name; }
User.prototype.role = "user";

const user = new User("Ali");

const data = {};
for (const key in user) {
  data[key] = user[key]; // role ham kiradi!
}
console.log(data); // { name: "Ali", role: "user" } — noto'g'ri!
```

### ✅ To'g'ri usul:

```javascript
// Variant 1: hasOwnProperty
for (const key in user) {
  if (Object.hasOwn(user, key)) {
    data[key] = user[key];
  }
}

// Variant 2: Object.keys (tavsiya)
const data2 = {};
for (const key of Object.keys(user)) {
  data2[key] = user[key];
}

// Variant 3: Spread (eng oson)
const data3 = { ...user }; // faqat own enumerable
```

**Nima uchun:** `for...in` prototype chain bo'ylab ham yuradi. `Object.keys` yoki spread faqat own property'larni oladi.

---

### ❌ Xato 4: defineProperty default'larini bilmaslik

```javascript
const obj = {};
Object.defineProperty(obj, "x", { value: 10 });

obj.x = 20;            // ❌ O'zgarmaydi! (writable default false)
console.log(obj.x);    // 10
delete obj.x;          // ❌ O'chirilmaydi! (configurable default false)
console.log(Object.keys(obj)); // [] (enumerable default false)
```

### ✅ To'g'ri usul:

```javascript
// defineProperty default: writable:false, enumerable:false, configurable:false
// Oddiy assignment default: writable:true, enumerable:true, configurable:true

Object.defineProperty(obj, "x", {
  value: 10,
  writable: true,
  enumerable: true,
  configurable: true
});
// Endi oddiy property kabi ishlaydi
```

**Nima uchun:** `defineProperty` default'lari `false` — bu "xavfsiz" default. Oddiy `.` assignment esa `true`. Ko'p dasturchilar bu farqni bilmaydi.

---

### ❌ Xato 5: Object equality tekshirish

```javascript
const a = { x: 1 };
const b = { x: 1 };

console.log(a === b);  // false ❌ — farqli reference!
console.log(a == b);   // false ❌ — farqli reference!

// Object'lar REFERENCE bo'yicha taqqoslanadi, qiymat bo'yicha emas
```

### ✅ To'g'ri usul:

```javascript
// Shallow comparison:
function shallowEqual(a, b) {
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every(key => a[key] === b[key]);
}

// Deep comparison:
function deepEqual(a, b) {
  if (a === b) return true;
  if (a == null || b == null) return false;
  if (typeof a !== "object" || typeof b !== "object") return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => deepEqual(a[key], b[key]));
}

// Yoki oddiy holatlarda:
JSON.stringify(a) === JSON.stringify(b); // ⚠️ key tartibi muhim
```

**Nima uchun:** JS da object'lar **reference** bo'yicha taqqoslanadi. Ikki alohida object bir xil qiymatga ega bo'lsa ham — `===` `false`. Qiymat bo'yicha taqqoslash uchun maxsus funksiya kerak.

---

## Amaliy Mashqlar

### Mashq 1: Property Descriptor (Oson)

**Savol:** Object yarating: `id` read-only va enumerable, `_secret` enumerable bo'lmagan.

<details>
<summary>Javob</summary>

```javascript
const obj = {};

Object.defineProperties(obj, {
  id: {
    value: "USR-001",
    writable: false,
    enumerable: true,
    configurable: false
  },
  _secret: {
    value: "maxfiy-kalit",
    writable: true,
    enumerable: false,
    configurable: false
  }
});

console.log(obj.id);              // "USR-001"
obj.id = "boshqa";                // ❌ o'zgarmaydi
console.log(Object.keys(obj));    // ["id"] — _secret ko'rinmaydi
console.log(obj._secret);         // "maxfiy-kalit" — to'g'ridan-to'g'ri kirish mumkin
```
</details>

---

### Mashq 2: Deep Freeze (O'rta)

**Savol:** `deepFreeze` yozing — nested object'larni ham muzlating.

<details>
<summary>Javob</summary>

```javascript
function deepFreeze(obj) {
  Object.freeze(obj);

  for (const key of Object.getOwnPropertyNames(obj)) {
    const value = obj[key];
    if (value !== null && typeof value === "object" && !Object.isFrozen(value)) {
      deepFreeze(value);
    }
  }

  return obj;
}

// Test:
const config = deepFreeze({
  db: { host: "localhost", port: 5432 },
  cache: { ttl: 300, nested: { deep: true } }
});

config.db.host = "remote";           // ❌
config.cache.nested.deep = false;     // ❌
console.log(config.db.host);          // "localhost" ✅
console.log(config.cache.nested.deep); // true ✅
```
</details>

---

### Mashq 3: Shallow vs Deep Equal (O'rta)

**Savol:** `shallowEqual(a, b)` va `deepEqual(a, b)` funksiyalarini yozing.

<details>
<summary>Javob</summary>

```javascript
function shallowEqual(a, b) {
  if (a === b) return true;
  if (!a || !b || typeof a !== "object" || typeof b !== "object") return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => a[key] === b[key]);
}

function deepEqual(a, b) {
  if (a === b) return true;
  if (a == null || b == null) return false;
  if (typeof a !== typeof b) return false;
  if (typeof a !== "object") return a === b;

  if (Array.isArray(a) !== Array.isArray(b)) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => deepEqual(a[key], b[key]));
}

// Test:
shallowEqual({ a: 1, b: 2 }, { a: 1, b: 2 });               // true
shallowEqual({ a: { x: 1 } }, { a: { x: 1 } });             // false (nested)
deepEqual({ a: { x: 1 } }, { a: { x: 1 } });                // true
deepEqual({ a: [1, 2] }, { a: [1, 2] });                     // true
deepEqual({ a: [1, 2] }, { a: [1, 3] });                     // false
```
</details>

---

### Mashq 4: Observable Object (Qiyin)

**Savol:** Object yarating — har qanday property o'zgarganda callback chaqirilsin.

<details>
<summary>Javob</summary>

```javascript
function observable(target, onChange) {
  return new Proxy(target, {
    set(obj, prop, value) {
      const oldValue = obj[prop];
      obj[prop] = value;
      if (oldValue !== value) {
        onChange(prop, value, oldValue);
      }
      return true;
    },
    deleteProperty(obj, prop) {
      const oldValue = obj[prop];
      delete obj[prop];
      onChange(prop, undefined, oldValue);
      return true;
    }
  });
}

const user = observable({ name: "Ali", age: 25 }, (prop, newVal, oldVal) => {
  console.log(`${prop}: ${oldVal} → ${newVal}`);
});

user.name = "Vali";    // "name: Ali → Vali"
user.age = 30;         // "age: 25 → 30"
delete user.age;       // "age: 30 → undefined"
```

**Tushuntirish:** Proxy orqali har bir `set` va `delete` operatsiyasini ushlab, callback chaqiramiz. Bu Vue 3 reactivity ning asosiy prinsipi. Proxy haqida to'liq [23-proxy-reflect.md](23-proxy-reflect.md) da.
</details>

---

### Mashq 5: structuredClone Polyfill (Qiyin)

**Savol:** `deepClone` funksiyasini yozing — circular reference'ni ham to'g'ri handle qilsin.

<details>
<summary>Javob</summary>

```javascript
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (obj instanceof Date) return new Date(obj.getTime());
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);
  if (obj instanceof Map) {
    const map = new Map();
    seen.set(obj, map);
    obj.forEach((val, key) => map.set(deepClone(key, seen), deepClone(val, seen)));
    return map;
  }
  if (obj instanceof Set) {
    const set = new Set();
    seen.set(obj, set);
    obj.forEach(val => set.add(deepClone(val, seen)));
    return set;
  }

  // Circular reference tekshirish
  if (seen.has(obj)) return seen.get(obj);

  const copy = Array.isArray(obj) ? [] : {};
  seen.set(obj, copy);

  for (const key of Reflect.ownKeys(obj)) {
    copy[key] = deepClone(obj[key], seen);
  }

  return copy;
}

// Test:
const obj = { a: 1, b: { c: 2 }, d: new Date(), e: /hello/gi };
obj.self = obj; // circular!

const cloned = deepClone(obj);
console.log(cloned.b.c);         // 2
console.log(cloned.d instanceof Date); // true
console.log(cloned.self === cloned);   // true ✅ (circular handled)
console.log(cloned.self === obj);      // false ✅ (alohida)
```

**Tushuntirish:** `WeakMap` bilan allaqachon klonlangan object'larni kuzatamiz. Circular reference topilsa — saqlangan nusxani qaytaramiz. `Reflect.ownKeys` — string + symbol key'larni oladi.
</details>

---

## Xulosa

1. **Object yaratish:** Literal (oddiy), Constructor (ko'p instance), Object.create (aniq prototype), Class (OOP).

2. **Property Descriptors:** `value`, `writable`, `enumerable`, `configurable`. `defineProperty` default'lari `false`, oddiy assignment `true`.

3. **Getters/Setters:** Accessor property — o'qishda funksiya, yozishda validation. Data property bilan aralashtirib bo'lmaydi.

4. **Immutability:** `preventExtensions` < `seal` < `freeze`. Barchasi **shallow** — deep freeze uchun recursive funksiya kerak.

5. **Copying:** Spread/assign = shallow. `structuredClone` = deep (tavsiya). JSON hack = limited. Recursive = universal.

6. **Enumeration:** `for...in` (prototype ham), `Object.keys` (own + enumerable), `Reflect.ownKeys` (hammasi).

7. **Optional Chaining** (`?.`) — xavfsiz property access. Null/undefined da xato bermaydi.

8. **Object equality** — reference bo'yicha (`===`). Qiymat bo'yicha taqqoslash uchun shallowEqual/deepEqual kerak.

---

> **Keyingi bo'lim:** [07-prototypes.md](07-prototypes.md) — Prototypal Inheritance — `[[Prototype]]`, prototype chain, `new` keyword ichidan.
