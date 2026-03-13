# Bo'lim 6: Objects Ichidan

> Object — key-value juftliklardan iborat ma'lumot tuzilmasi. JavaScript da deyarli barcha murakkab qiymatlar (array, function, date, regex) object'ning maxsus ko'rinishlari. Object'larni chuqur tushunish — butun tilni tushunish.

---

## Mundarija

- [Object Creation Patterns](#object-creation-patterns)
- [Property Descriptors](#property-descriptors)
- [Getters va Setters](#getters-va-setters)
- [Object Immutability](#object-immutability)
- [Object Copying — Shallow va Deep](#object-copying--shallow-va-deep)
- [Property Enumeration](#property-enumeration)
- [Zamonaviy Object Metodlari](#zamonaviy-object-metodlari)
- [Computed Properties va Shorthand](#computed-properties-va-shorthand)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Object Creation Patterns

### Nazariya

JavaScript da object yaratishning to'rtta asosiy usuli bor. Har birining o'z use case'i va xususiyatlari mavjud:

**1. Object Literal** — eng ko'p ishlatiladigan va eng oddiy usul:

```javascript
const user = {
  name: "Alice",
  age: 25,
  greet() {
    return `Hi, I'm ${this.name}`;
  }
};
```

**2. Constructor Function** — `new` keyword bilan (ES6 dan oldingi asosiy pattern):

```javascript
function User(name, age) {
  this.name = name;
  this.age = age;
}
User.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};

const user = new User("Alice", 25);
```

**3. `Object.create()`** — berilgan prototype bilan yangi object yaratadi:

```javascript
const userProto = {
  greet() { return `Hi, I'm ${this.name}`; }
};

const user = Object.create(userProto);
user.name = "Alice";
user.age = 25;
// user.__proto__ === userProto ✅
```

**4. Class (ES6)** — constructor function'ning syntactic sugar'i:

```javascript
class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  greet() {
    return `Hi, I'm ${this.name}`;
  }
}
const user = new User("Alice", 25);
```

### Under the Hood

V8 da object yaratilganda engine **Hidden Class** (Map deyiladi) yaratadi. Hidden Class object'ning "shakli" (shape) — qaysi property'lar bor va ular memory'da qanday joylashgan. Bir xil ketma-ketlikda bir xil property'lar qo'shilgan object'lar **bitta** Hidden Class'ni share qiladi — bu inline caching va optimizatsiya uchun muhim.

```
const a = { x: 1, y: 2 };
const b = { x: 10, y: 20 };
// a va b bitta Hidden Class share qiladi — bir xil shape

const c = { y: 2, x: 1 };
// c boshqa Hidden Class — property qo'shilish TARTIBI boshqa

const d = { x: 1, y: 2, z: 3 };
// d ham boshqa Hidden Class — property soni farq qiladi
```

V8 optimizatsiyasi uchun: object'larga property'larni **bir xil tartibda** qo'shish — Hidden Class'larni share qilishga yordam beradi, bu esa property access'ni tezlashtiradi.

---

## Property Descriptors

### Nazariya

Har bir object property'si faqat qiymatdan emas, balki **meta-ma'lumotlar** dan ham iborat. Bu meta-ma'lumotlar **property descriptor** deyiladi va property'ning xulq-atvorini boshqaradi.

Ikki xil descriptor turi bor:

**Data Descriptor:**
- `value` — property qiymati
- `writable` — qiymatni o'zgartirish mumkinmi (`true`/`false`)
- `enumerable` — `for...in` va `Object.keys()` da ko'rinadimi
- `configurable` — descriptor o'zgartirilishi yoki property o'chirilishi mumkinmi

**Accessor Descriptor:**
- `get` — property o'qilganda chaqiriladigan funksiya
- `set` — property yozilganda chaqiriladigan funksiya
- `enumerable` — yuqoridagi kabi
- `configurable` — yuqoridagi kabi

Data va Accessor descriptor aralashtirilMAYDI — bitta property'da `value`/`writable` va `get`/`set` birga bo'lishi mumkin emas.

Object literal yoki oddiy assign bilan yaratilgan property'larning default descriptor'i:

```javascript
const user = { name: "Alice" };

console.log(Object.getOwnPropertyDescriptor(user, "name"));
// {
//   value: "Alice",
//   writable: true,       ← qiymat o'zgartirsa bo'ladi
//   enumerable: true,     ← for...in da ko'rinadi
//   configurable: true    ← o'chirish/qayta configure mumkin
// }
```

### Under the Hood

`Object.defineProperty()` bilan property'ning xulq-atvorini batafsil sozlash mumkin. `Object.defineProperty` bilan yaratilgan property'larda default qiymatlar **`false`** (`writable`, `enumerable`, `configurable` hammasi `false`):

```javascript
const config = {};

Object.defineProperty(config, "API_KEY", {
  value: "secret-123",
  writable: false,      // ❌ o'zgartirish mumkin emas
  enumerable: false,    // ❌ for...in / Object.keys da ko'rinmaydi
  configurable: false   // ❌ delete qilib bo'lmaydi, descriptor o'zgarmaydi
});

config.API_KEY = "hacked"; // ❌ silent fail (strict mode'da TypeError)
console.log(config.API_KEY); // "secret-123" — o'zgarmadi

delete config.API_KEY; // ❌ silent fail
console.log(Object.keys(config)); // [] — enumerable: false

// Bir nechta property birdan:
Object.defineProperties(config, {
  HOST: { value: "localhost", enumerable: true },
  PORT: { value: 3000, enumerable: true }
});
```

### Kod Misollari

`configurable: false` ning ta'siri — qaytib bo'lmaydiganlik:

```javascript
const obj = {};

Object.defineProperty(obj, "permanent", {
  value: 42,
  configurable: false
});

// Qayta define qilish mumkin emas:
// Object.defineProperty(obj, "permanent", { configurable: true });
// ❌ TypeError: Cannot redefine property: permanent

// LEKIN: writable true → false o'tkazish mumkin (bir tomonlama):
Object.defineProperty(obj, "semiLocked", {
  value: 10,
  writable: true,
  configurable: false
});

obj.semiLocked = 20; // ✅ ishlaydi — writable: true

Object.defineProperty(obj, "semiLocked", { writable: false });
// ✅ writable: true → false mumkin (configurable: false bo'lsa ham)

obj.semiLocked = 30; // ❌ endi o'zgarmaydi
// Object.defineProperty(obj, "semiLocked", { writable: true });
// ❌ TypeError — false → true qaytarish mumkin emas
```

---

## Getters va Setters

### Nazariya

Getter va setter — property o'qilganda yoki yozilganda avtomatik chaqiriladigan funksiyalar. Tashqaridan oddiy property kabi ko'rinadi, lekin ichida logika bajariladi.

Getter/setter ikkita asosiy maqsadga xizmat qiladi:
1. **Computed properties** — har o'qilganda qiymat hisoblash
2. **Validation** — qiymat yozilganda tekshirish

### Kod Misollari

```javascript
const user = {
  firstName: "Alice",
  lastName: "Smith",

  // Getter — o'qilganda hisoblanadi
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  },

  // Setter — yozilganda validation + parsing
  set fullName(value) {
    const parts = value.split(" ");
    if (parts.length < 2) {
      throw new Error("Full name must include first and last name");
    }
    this.firstName = parts[0];
    this.lastName = parts.slice(1).join(" ");
  }
};

console.log(user.fullName); // "Alice Smith" — getter chaqirildi
user.fullName = "Bob Johnson"; // setter chaqirildi
console.log(user.firstName); // "Bob"
console.log(user.lastName);  // "Johnson"
```

`Object.defineProperty` bilan getter/setter:

```javascript
function createTemperature(celsius) {
  const temp = { _celsius: celsius };

  Object.defineProperty(temp, "fahrenheit", {
    get() {
      return this._celsius * 9 / 5 + 32;
    },
    set(f) {
      this._celsius = (f - 32) * 5 / 9;
    },
    enumerable: true
  });

  return temp;
}

const t = createTemperature(100);
console.log(t.fahrenheit); // 212 — getter: 100 * 9/5 + 32
t.fahrenheit = 32;
console.log(t._celsius);   // 0 — setter: (32-32) * 5/9
```

---

## Object Immutability

### Nazariya

JavaScript da object'ni "o'zgarmas" qilishning uchta darajasi bor — har biri oldigidan kuchliroq:

| Metod | Yangi property | O'zgartirish | O'chirish |
|-------|---------------|-------------|-----------|
| `Object.preventExtensions()` | ❌ | ✅ | ✅ |
| `Object.seal()` | ❌ | ✅ | ❌ |
| `Object.freeze()` | ❌ | ❌ | ❌ |

Tekshirish metodlari: `Object.isExtensible()`, `Object.isSealed()`, `Object.isFrozen()`.

Muhim: **barcha uchta faqat SHALLOW** ishlaydi — ichki (nested) object'larga ta'sir qilmaydi.

### Kod Misollari

```javascript
// preventExtensions — yangi property qo'shib bo'lmaydi
const obj1 = { a: 1 };
Object.preventExtensions(obj1);
obj1.b = 2;       // ❌ silent fail (strict: TypeError)
obj1.a = 10;      // ✅ mavjud property o'zgartirish mumkin
delete obj1.a;    // ✅ o'chirish mumkin

// seal — yangi property yo'q, o'chirish yo'q, o'zgartirish mumkin
const obj2 = { a: 1, b: 2 };
Object.seal(obj2);
obj2.c = 3;       // ❌ yangi property
obj2.a = 10;      // ✅ mavjud property o'zgartirish
delete obj2.b;    // ❌ o'chirish

// freeze — hech narsa o'zgarmaydi
const obj3 = { a: 1, nested: { x: 10 } };
Object.freeze(obj3);
obj3.a = 99;           // ❌ o'zgarmaydi
obj3.nested.x = 99;   // ✅ ISHLAYDI! — shallow freeze, nested object'ga tegmaydi
console.log(obj3.nested.x); // 99
```

Deep freeze implement qilish:

```javascript
function deepFreeze(obj) {
  Object.freeze(obj);

  Object.getOwnPropertyNames(obj).forEach(prop => {
    const value = obj[prop];
    if (value !== null && typeof value === "object" && !Object.isFrozen(value)) {
      deepFreeze(value);
      // ✅ recursive — ichki object'lar ham freeze bo'ladi
      // Object.isFrozen check — circular reference'dan himoya
    }
  });

  return obj;
}

const config = deepFreeze({
  db: { host: "localhost", port: 5432 },
  api: { timeout: 5000 }
});

config.db.host = "hacked"; // ❌ o'zgarmaydi — deep freeze
console.log(config.db.host); // "localhost"
```

---

## Object Copying — Shallow va Deep

### Nazariya

JavaScript da object'lar **reference** bo'yicha uzatiladi. `const b = a` deganda `b` yangi object emas — `a` bilan bir xil object'ga reference. Object'ni haqiqiy copy qilish uchun maxsus usullar kerak.

**Shallow Copy** — faqat birinchi daraja copy bo'ladi. Ichki (nested) object'lar hali reference bo'lib qoladi:

```javascript
const original = { name: "Alice", address: { city: "NYC" } };

// Shallow copy usullari:
const copy1 = { ...original };                    // Spread
const copy2 = Object.assign({}, original);         // Object.assign

copy1.name = "Bob";          // ✅ original.name o'zgarmaydi
copy1.address.city = "LA";   // ❌ original.address.city HAM o'zgaradi!
// Sabab: address object reference copy bo'ldi, object o'zi emas
console.log(original.address.city); // "LA" — original buzildi
```

**Deep Copy** — barcha darajalar to'liq copy bo'ladi. Hech qanday reference qolmaydi:

```javascript
// 1. structuredClone (ES2022) — eng yaxshi zamonaviy usul
const deep1 = structuredClone(original);
// ✅ Circular reference qo'llab-quvvatlaydi
// ✅ Date, Map, Set, ArrayBuffer, RegExp copy qiladi
// ❌ Function, Symbol, DOM node copy qilmaydi

// 2. JSON hack — cheklovlar bor
const deep2 = JSON.parse(JSON.stringify(original));
// ❌ undefined, Function, Symbol, Infinity, NaN yo'qoladi
// ❌ Date → string ga aylanadi (qaytmaydi)
// ❌ Map, Set, RegExp yo'qoladi
// ❌ Circular reference → TypeError

// 3. Recursive deep clone — to'liq nazorat
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (seen.has(obj)) return seen.get(obj); // circular ref himoya

  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);
  if (obj instanceof Map) {
    const map = new Map();
    seen.set(obj, map);
    obj.forEach((v, k) => map.set(deepClone(k, seen), deepClone(v, seen)));
    return map;
  }
  if (obj instanceof Set) {
    const set = new Set();
    seen.set(obj, set);
    obj.forEach(v => set.add(deepClone(v, seen)));
    return set;
  }

  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone);

  for (const key of Reflect.ownKeys(obj)) {
    clone[key] = deepClone(obj[key], seen);
  }
  return clone;
}
```

### Under the Hood

`structuredClone()` browser'ning **Structured Clone Algorithm** ini ishlatadi — bu xuddi `postMessage()` (Web Worker'larga data yuborish) da ishlatiladigan algoritm. U object'ni serialize qilib, keyin yangi object sifatida deserialize qiladi.

Copy usullarini taqqoslash:

| Xususiyat | Spread / assign | JSON hack | `structuredClone` | Recursive |
|---|---|---|---|---|
| Nested objects | ❌ Shallow | ✅ Deep | ✅ Deep | ✅ Deep |
| Circular ref | ❌ | ❌ Error | ✅ | ✅ (WeakMap) |
| Date | ❌ ref | ❌ → string | ✅ | ✅ |
| Map/Set | ❌ ref | ❌ yo'qoladi | ✅ | ✅ |
| Function | ❌ ref | ❌ yo'qoladi | ❌ Error | ⚠️ ref |
| RegExp | ❌ ref | ❌ → {} | ✅ | ✅ |
| undefined | ✅ | ❌ yo'qoladi | ✅ | ✅ |
| Symbol keys | ✅ (spread) | ❌ | ❌ | ✅ (Reflect.ownKeys) |
| Performance | Eng tez | O'rta | O'rta | Sekin |

---

## Property Enumeration

### Nazariya

Object property'larini sanab o'tishning bir nechta usuli bor — har biri turli property'larni ko'rsatadi:

| Metod | Own | Inherited | Enumerable | Non-enum | Symbol |
|-------|-----|-----------|------------|----------|--------|
| `for...in` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `Object.keys()` | ✅ | ❌ | ✅ | ❌ | ❌ |
| `Object.values()` | ✅ | ❌ | ✅ | ❌ | ❌ |
| `Object.entries()` | ✅ | ❌ | ✅ | ❌ | ❌ |
| `Object.getOwnPropertyNames()` | ✅ | ❌ | ✅ | ✅ | ❌ |
| `Object.getOwnPropertySymbols()` | ✅ | ❌ | ✅ | ✅ | ✅ (faqat) |
| `Reflect.ownKeys()` | ✅ | ❌ | ✅ | ✅ | ✅ |

### Kod Misollari

```javascript
const parent = { inherited: true };
const obj = Object.create(parent);

obj.enumProp = "visible";
Object.defineProperty(obj, "hiddenProp", {
  value: "hidden",
  enumerable: false
});
obj[Symbol("id")] = 123;

// for...in — own + inherited, faqat enumerable
for (const key in obj) {
  console.log(key); // "enumProp", "inherited"
  // ✅ inherited ham ko'rinadi — shuning uchun hasOwn tekshirish kerak
}

// Object.keys — own, faqat enumerable
console.log(Object.keys(obj)); // ["enumProp"]

// Reflect.ownKeys — HAMMASI (own, non-enum, symbol)
console.log(Reflect.ownKeys(obj)); // ["enumProp", "hiddenProp", Symbol(id)]
```

`Object.fromEntries()` — entries array'dan object yaratish (reverse of `Object.entries`):

```javascript
const entries = [["name", "Alice"], ["age", 25]];
const obj = Object.fromEntries(entries);
// { name: "Alice", age: 25 }

// Real-world: URL search params → object
const params = new URLSearchParams("name=Alice&age=25");
const query = Object.fromEntries(params);
// { name: "Alice", age: "25" }

// Real-world: object transform pipeline
const prices = { apple: 1.5, banana: 0.75, cherry: 3.0 };
const doubled = Object.fromEntries(
  Object.entries(prices).map(([fruit, price]) => [fruit, price * 2])
);
// { apple: 3, banana: 1.5, cherry: 6 }
```

---

## Zamonaviy Object Metodlari

### `Object.hasOwn()` (ES2022)

`hasOwnProperty` ning zamonaviy, xavfsiz versiyasi:

```javascript
const obj = Object.create(null); // prototype yo'q — hasOwnProperty metodi yo'q
obj.key = "value";

// obj.hasOwnProperty("key"); // ❌ TypeError — metod mavjud emas
Object.hasOwn(obj, "key");     // ✅ true — statik metod, har doim ishlaydi

// Nima uchun Object.hasOwn yaxshiroq:
// 1. Object.create(null) bilan yaratilgan object'larda ishlaydi
// 2. hasOwnProperty override qilingan bo'lsa ham xavfsiz
const tricky = { hasOwnProperty: () => false };
tricky.hasOwnProperty("hasOwnProperty"); // false — noto'g'ri!
Object.hasOwn(tricky, "hasOwnProperty"); // true — to'g'ri
```

### `Object.groupBy()` (ES2024)

Array elementlarini callback natijasi bo'yicha guruhlash:

```javascript
const products = [
  { name: "Apple", category: "fruit", price: 1.5 },
  { name: "Banana", category: "fruit", price: 0.75 },
  { name: "Carrot", category: "vegetable", price: 1.0 },
  { name: "Broccoli", category: "vegetable", price: 2.0 },
];

const grouped = Object.groupBy(products, product => product.category);
// {
//   fruit: [{ name: "Apple", ... }, { name: "Banana", ... }],
//   vegetable: [{ name: "Carrot", ... }, { name: "Broccoli", ... }]
// }

// Narx bo'yicha guruhlash:
const byPrice = Object.groupBy(products, p =>
  p.price > 1 ? "expensive" : "cheap"
);
```

---

## Computed Properties va Shorthand

### Nazariya

ES6 da object literal'ga kiritilgan qulayliklar:

**Computed Property Names** — property nomini expression orqali hisoblash:

```javascript
const field = "email";
const prefix = "user";

const obj = {
  [field]: "alice@mail.com",          // ✅ "email": "alice@mail.com"
  [`${prefix}Name`]: "Alice",         // ✅ "userName": "Alice"
  [`get${field[0].toUpperCase() + field.slice(1)}`]() {
    return this[field];               // ✅ "getEmail" metodi
  }
};

console.log(obj.email);     // "alice@mail.com"
console.log(obj.userName);  // "Alice"
console.log(obj.getEmail()); // "alice@mail.com"
```

**Shorthand Property Names** — o'zgaruvchi nomi va property nomi bir xil bo'lsa:

```javascript
const name = "Alice";
const age = 25;

// ES5:
const user1 = { name: name, age: age };
// ES6 shorthand:
const user2 = { name, age }; // ✅ bir xil natija
```

**Shorthand Methods** — `function` keyword'siz:

```javascript
// ES5:
const obj1 = { greet: function() { return "Hi"; } };
// ES6:
const obj2 = { greet() { return "Hi"; } }; // ✅ qisqaroq
```

**Optional Chaining (`?.`)** bilan xavfsiz object traversal:

```javascript
const user = {
  profile: {
    address: { city: "NYC" }
  }
};

// Xavfsiz chuqur access:
const city = user?.profile?.address?.city; // "NYC"
const zip = user?.profile?.address?.zip;   // undefined (xato emas)
const phone = user?.contact?.phone;        // undefined (xato emas)

// Method chaqirish:
const result = user?.profile?.toString?.(); // "[object Object]"
const missing = user?.nonExistent?.method?.(); // undefined

// Index access:
const arr = user?.items?.[0]; // undefined
```

---

## Common Mistakes

### ❌ Xato 1: Shallow Copy ni Deep Copy deb o'ylash

```javascript
const original = { settings: { theme: "dark" } };
const copy = { ...original }; // ❌ shallow copy

copy.settings.theme = "light";
console.log(original.settings.theme); // "light" — original BUZILDI!
```

### ✅ To'g'ri usul:

```javascript
const copy = structuredClone(original); // ✅ deep copy
copy.settings.theme = "light";
console.log(original.settings.theme); // "dark" — original saqlanadi
```

**Nima uchun:** Spread operator faqat birinchi darajani copy qiladi. Nested object'lar reference bo'lib qoladi. `structuredClone` to'liq deep copy yaratadi.

---

### ❌ Xato 2: `for...in` da hasOwn Tekshirmaslik

```javascript
const parent = { type: "parent" };
const child = Object.create(parent);
child.name = "Alice";

for (const key in child) {
  console.log(key, child[key]);
  // "name" "Alice"
  // "type" "parent" — ❌ inherited property ham ko'rinadi!
}
```

### ✅ To'g'ri usul:

```javascript
// Object.keys — faqat own enumerable
for (const key of Object.keys(child)) {
  console.log(key, child[key]); // faqat "name" "Alice"
}

// Yoki hasOwn bilan:
for (const key in child) {
  if (Object.hasOwn(child, key)) {
    console.log(key, child[key]); // faqat "name" "Alice"
  }
}
```

**Nima uchun:** `for...in` prototype chain bo'ylab inherited property'larni ham sanab o'tadi. Faqat own property kerak bo'lsa — `Object.keys()` yoki `Object.hasOwn()` ishlatish kerak.

---

### ❌ Xato 3: `Object.freeze` ni Deep Freeze deb o'ylash

```javascript
const config = Object.freeze({
  db: { host: "localhost", port: 5432 }
});

config.db.host = "hacked"; // ✅ ISHLAYDI! — nested object freeze emas
console.log(config.db.host); // "hacked"
```

### ✅ To'g'ri usul:

```javascript
function deepFreeze(obj) {
  Object.freeze(obj);
  Object.getOwnPropertyNames(obj).forEach(prop => {
    const val = obj[prop];
    if (val !== null && typeof val === "object" && !Object.isFrozen(val)) {
      deepFreeze(val);
    }
  });
  return obj;
}

const config = deepFreeze({ db: { host: "localhost" } });
config.db.host = "hacked"; // ❌ o'zgarmaydi
```

**Nima uchun:** `Object.freeze()` faqat shallow — birinchi daraja property'lar freeze bo'ladi. Nested object'larni ham freeze qilish uchun recursive yondashuv kerak.

---

### ❌ Xato 4: JSON.stringify/parse bilan Deep Copy Muammolari

```javascript
const original = {
  date: new Date(),
  pattern: /test/gi,
  fn: () => "hello",
  undef: undefined,
  map: new Map([["a", 1]])
};

const copy = JSON.parse(JSON.stringify(original));
console.log(copy);
// {
//   date: "2024-01-01T00:00:00.000Z" — ❌ string bo'lib qoldi
//   pattern: {}                        — ❌ bo'sh object
//   fn: [yo'q]                        — ❌ yo'qoldi
//   undef: [yo'q]                     — ❌ yo'qoldi
//   map: {}                           — ❌ bo'sh object
// }
```

### ✅ To'g'ri usul:

```javascript
const copy = structuredClone(original);
// ✅ date → Date object saqlanadi
// ✅ pattern → RegExp saqlanadi
// ❌ fn — structuredClone ham function copy qilmaydi (DataCloneError)
// ✅ map → Map saqlanadi
```

**Nima uchun:** JSON faqat JSON-safe qiymatlarni qo'llab-quvvatlaydi. `Date`, `RegExp`, `Map`, `Set`, `undefined`, `Function`, `Symbol`, `Infinity`, `NaN` — barchasi yo'qoladi yoki noto'g'ri convert bo'ladi.

---

## Amaliy Mashqlar

### Mashq 1: Property Descriptor (Oson)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
const obj = {};
Object.defineProperty(obj, "secret", {
  value: 42,
  enumerable: false
});
obj.visible = "yes";

console.log(Object.keys(obj));
console.log(obj.secret);
console.log("secret" in obj);
```

<details>
<summary>Javob</summary>

```javascript
console.log(Object.keys(obj));  // ["visible"]
// ✅ Object.keys faqat enumerable property'larni qaytaradi
// secret enumerable: false — ko'rinmaydi

console.log(obj.secret);       // 42
// ✅ enumerable: false faqat sanab o'tishda yashiradi
// to'g'ridan-to'g'ri access ishlaydi

console.log("secret" in obj);  // true
// ✅ "in" operatori enumerable'ga qaramaydi — mavjudligini tekshiradi
```

</details>

---

### Mashq 2: Shallow vs Deep Copy (O'rta)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
const a = { x: 1, inner: { y: 2 } };
const b = { ...a };
const c = structuredClone(a);

b.x = 10;
b.inner.y = 20;
c.inner.y = 30;

console.log(a.x);       // ?
console.log(a.inner.y);  // ?
```

<details>
<summary>Javob</summary>

```
1
20
```

- `b.x = 10` → `a.x` o'zgarmaydi (birinchi daraja copy bo'lgan)
- `b.inner.y = 20` → `a.inner.y` **HAM** o'zgaradi! (shallow copy — inner reference)
- `c.inner.y = 30` → `a.inner.y` o'zgarmaydi (structuredClone deep copy)
- `a.inner.y = 20` — b tomonidan o'zgartirilgan

</details>

---

### Mashq 3: deepEqual Implement Qilish (Qiyin)

**Savol:** `deepEqual(a, b)` funksiyasini yozing. Ikki qiymatning chuqur tenglgini tekshiradi.

<details>
<summary>Javob</summary>

```javascript
function deepEqual(a, b) {
  // 1. Strict equality — primitive va bir xil reference
  if (a === b) return true;

  // 2. null yoki non-object
  if (a === null || b === null) return false;
  if (typeof a !== "object" || typeof b !== "object") return false;

  // 3. Turli constructor
  if (a.constructor !== b.constructor) return false;

  // 4. Array
  if (Array.isArray(a)) {
    if (a.length !== b.length) return false;
    return a.every((item, i) => deepEqual(item, b[i]));
  }

  // 5. Date
  if (a instanceof Date) {
    return a.getTime() === b.getTime();
  }

  // 6. RegExp
  if (a instanceof RegExp) {
    return a.source === b.source && a.flags === b.flags;
  }

  // 7. Object
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key =>
    Object.hasOwn(b, key) && deepEqual(a[key], b[key])
  );
}

// Test:
console.log(deepEqual({ a: 1, b: { c: 2 } }, { a: 1, b: { c: 2 } })); // true
console.log(deepEqual({ a: 1 }, { a: 2 }));  // false
console.log(deepEqual([1, [2]], [1, [2]]));   // true
console.log(deepEqual(new Date(0), new Date(0))); // true
```

</details>

---

### Mashq 4: Immutable Config (Qiyin)

**Savol:** `createConfig(defaults)` funksiyasini yozing. Qaytarilgan object deep freeze bo'lsin va `get(path)` metodi dot notation bilan ishlashi kerak.

<details>
<summary>Javob</summary>

```javascript
function createConfig(defaults) {
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

  const data = deepFreeze(structuredClone(defaults));

  return {
    get(path) {
      return path.split(".").reduce((obj, key) => obj?.[key], data);
      // ✅ "db.host" → data.db.host
      // ✅ optional chaining — yo'q bo'lsa undefined
    },
    toJSON() {
      return structuredClone(data); // copy qaytaradi
    }
  };
}

const config = createConfig({
  db: { host: "localhost", port: 5432 },
  api: { timeout: 5000, retries: 3 }
});

console.log(config.get("db.host"));     // "localhost"
console.log(config.get("api.timeout")); // 5000
console.log(config.get("missing.key")); // undefined
```

</details>

---

## Xulosa

Bu bo'limda object'larning ichki mexanizmlari yoritildi:

- **Creation Patterns** — literal, constructor, `Object.create()`, class. V8 Hidden Class optimizatsiyasi.
- **Property Descriptors** — `writable`, `enumerable`, `configurable`. `Object.defineProperty()` bilan batafsil sozlash.
- **Getters/Setters** — computed properties va validation. Tashqaridan oddiy property, ichida logika.
- **Immutability** — `preventExtensions` → `seal` → `freeze`. Barchasi shallow — deep freeze recursive kerak.
- **Copying** — Spread/assign (shallow), JSON hack (cheklovlar), `structuredClone` (deep, zamonaviy), recursive (to'liq nazorat).
- **Enumeration** — `for...in` (inherited ham), `Object.keys` (own enumerable), `Reflect.ownKeys` (hammasi).
- **Zamonaviy** — `Object.hasOwn()` (ES2022), `Object.groupBy()` (ES2024), `Object.fromEntries()`.
- **Computed/Shorthand** — dynamic property names, qisqa yozuv, optional chaining.

Object'lar keyingi bo'limdagi **prototypal inheritance** ning asosi — object'lar prototype chain orqali bir-biridan method va property meros oladi.

---

**Keyingi bo'lim:** [07-prototypes.md](07-prototypes.md) — Prototypal Inheritance: `[[Prototype]]` internal slot, `__proto__` vs `prototype`, prototype chain, `Object.create()`, constructor functions, `new` keyword ichidan step-by-step, `instanceof` mexanizmi.
