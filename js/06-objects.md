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
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
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

// Prototype tekshirish — zamonaviy usul:
console.log(Object.getPrototypeOf(user) === userProto); // true ✅
// Eski usul `user.__proto__ === userProto` ham ishlaydi, lekin
// `__proto__` legacy deb belgilangan — Object.getPrototypeOf() afzal
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

<details>
<summary><strong>Under the Hood</strong></summary>

V8 da object yaratilganda engine **Hidden Class** yaratadi. (V8 source kodida bu internal strukturaga `Map` deb nom beriladi — lekin bu JavaScript'dagi `Map` data structure'dan butunlay boshqa tushuncha, ular shunchaki bir xil so'zni ishlatadi.) Hidden Class object'ning "shakli" (shape) — qaysi property'lar bor va ular memory'da qanday joylashgan. Bir xil ketma-ketlikda bir xil property'lar qo'shilgan object'lar **bitta** Hidden Class'ni share qiladi — bu inline caching va optimizatsiya uchun muhim (batafsil [01-js-engine.md](01-js-engine.md) da).

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha har bir property aslida **Property Descriptor Record** — bu `{[[Value]], [[Writable]], [[Enumerable]], [[Configurable]]}` (data descriptor) yoki `{[[Get]], [[Set]], [[Enumerable]], [[Configurable]]}` (accessor descriptor) fieldlardan iborat internal record.

`Object.defineProperty(obj, key, desc)` chaqirilganda engine ichida **[[DefineOwnProperty]](key, desc)** internal method ishga tushadi. Bu method quyidagi qadamlarni bajaradi:

1. `ToPropertyDescriptor(desc)` — berilgan JS objectni Property Descriptor Record ga aylantiradi
2. `ValidateAndApplyPropertyDescriptor` — yangi descriptor mavjud descriptor bilan solishtiriladi
3. Agar property `configurable: false` bo'lsa, faqat cheklangan o'zgarishlar ruxsat etiladi (`writable: true -> false` mumkin, lekin `false -> true` mumkin emas)
4. Data va accessor descriptor aralashtirib bo'lmaydi — agar mavjud property data descriptor bo'lsa va accessor descriptor berilsa, engine avval eski descriptor ni to'liq o'chiradi

V8 ichida property descriptor'lar **PropertyDetails** structurasida bit flaglar sifatida saqlanadi. `writable`, `enumerable`, `configurable` har biri bitta bit egallaydi. `Object.defineProperty` bilan yaratilgan property'larda spec bo'yicha default qiymatlar `false` (oddiy assign bilan yaratilganda esa `true`). Bu farq spec ning `CreateDataProperty` (assign/literal) vs `DefinePropertyOrThrow` (`Object.defineProperty`) abstract operation'lari orasidagi farqdan kelib chiqadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Getters va Setters

### Nazariya

Getter va setter — property o'qilganda yoki yozilganda avtomatik chaqiriladigan funksiyalar. Tashqaridan oddiy property kabi ko'rinadi, lekin ichida logika bajariladi.

Getter/setter ikkita asosiy maqsadga xizmat qiladi:
1. **Computed properties** — har o'qilganda qiymat hisoblash
2. **Validation** — qiymat yozilganda tekshirish

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha getter/setter — bu **accessor property**. Accessor property data property dan farqli ravishda `[[Value]]` va `[[Writable]]` o'rniga **`[[Get]]`** va **`[[Set]]`** internal slotlariga ega. `[[Get]]` — property o'qilganda chaqiriladigan funksiya (yoki `undefined`), `[[Set]]` — yozilganda chaqiriladigan funksiya.

Property access (`obj.prop`) bo'lganda engine **[[Get]](prop, receiver)** internal method ni chaqiradi. Bu method ichida `OrdinaryGet` abstract operation ishlaydi:
1. Property topiladi va uning descriptor olinadi
2. Agar descriptor accessor bo'lsa — `[[Get]]` funksiya `Call(getter, receiver)` orqali chaqiriladi
3. Agar data descriptor bo'lsa — `[[Value]]` qaytariladi

Assign (`obj.prop = val`) bo'lganda **[[Set]](prop, value, receiver)** chaqiriladi va `OrdinarySet` ishlaydi — accessor bo'lsa `Call(setter, receiver, value)` bajariladi.

V8 da accessor property'lar `AccessorPair` internal objectda saqlanadi. Bu objectda `getter` va `setter` funksiya pointer'lari bor. V8 inline cache (IC) accessor property'larni ham cache qiladi, lekin data property'larga nisbatan sekinroq — chunki har safar funksiya chaqiruvi (function call overhead) bo'ladi. Shu sababli performance-critical kodda getter/setter o'rniga oddiy property + explicit method chaqiruvi afzalroq bo'lishi mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da object immutability uchta **integrity level** orqali boshqariladi. Har biri object ning internal method'larini cheklaydi:

**`Object.preventExtensions()`** — object ning **`[[IsExtensible]]`** internal slot ini `false` ga o'rnatadi. Bu `[[DefineOwnProperty]]` ni chaqirganda yangi property qo'shishni bloklaydi. `OrdinaryDefineOwnProperty` ichida `IsExtensible(O)` tekshiriladi — `false` bo'lsa va property mavjud bo'lmasa, `false` qaytadi (strict mode da TypeError).

**`Object.seal()`** — spec da `SetIntegrityLevel(O, "sealed")` abstract operation chaqiriladi. Bu operation:
1. `[[PreventExtensions]](O)` — yangi property bloklash
2. Barcha own property'larning `[[Configurable]]` flagini `false` ga o'rnatish (`[[DefineOwnProperty]]` orqali)
3. `configurable: false` bo'lganda property o'chirib bo'lmaydi va descriptor turi o'zgartirilmaydi

**`Object.freeze()`** — `SetIntegrityLevel(O, "frozen")` chaqiriladi:
1. `[[PreventExtensions]](O)` + barcha property'lar `configurable: false`
2. Data property'lar uchun `writable: false` ham qo'shiladi
3. Accessor property'lar (getter/setter) **freeze bo'lmaydi** — `[[Get]]/[[Set]]` funksiyalar o'zgartirilmaydi, lekin ular chaqirilishi davom etadi

V8 ichida freeze/seal operatsiyalari object ning Hidden Class (Map) ini almashtiradi — yangi Map yaratiladi, unda `frozen`/`sealed` flag `true` bo'ladi. Bu keyingi property access operatsiyalarida tez tekshirish imkonini beradi (IC fast path).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

  Reflect.ownKeys(obj).forEach(prop => {
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

</details>

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
// ❌ Function, Symbol, DOM node → DataCloneError tashlaydi

// 2. JSON hack — cheklovlar bor
const deep2 = JSON.parse(JSON.stringify(original));
// ❌ undefined, Function, Symbol yo'qoladi; Infinity, NaN → null ga aylanadi
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

<details>
<summary><strong>Under the Hood</strong></summary>

`structuredClone()` ichida HTML spec ning **StructuredSerializeInternal / StructuredDeserialize** algoritmi ishlaydi — bu `postMessage()` va IndexedDB da ham ishlatiladigan algoritm.

Circular reference handling: algoritm **`memory`** deb nomlangan internal `Map<original, clone>` saqlaydi. Har bir object clone qilinayotganda avval `memory` da tekshiriladi — agar oldin clone qilingan bo'lsa, shu clone qaytariladi. Bu circular reference va shared reference muammosini hal qiladi:

```
A → B → C → A  (circular)
1. Clone(A): memory = {A: A'}, A.b = Clone(B)
2. Clone(B): memory = {A: A', B: B'}, B.c = Clone(C)
3. Clone(C): memory = {A: A', B: B', C: C'}, C.a = Clone(A) → memory da A' topildi → A' qaytariladi
Natija: A' → B' → C' → A' (circular saqlanadi, yangi graph)
```

Algoritm ichida har bir type uchun alohida serialization logic bor: `Date` uchun `[[DateValue]]` internal slot copy qilinadi, `RegExp` uchun `[[RegExpMatcher]]` + `[[OriginalSource]]` + `[[OriginalFlags]]`, `Map`/`Set` uchun entry'lar recursive clone qilinadi. `Function`, `Symbol`, `WeakMap`/`WeakRef` esa **non-serializable** — `DataCloneError` DOMException throw qilinadi.

V8 da `structuredClone` native C++ da implement qilingan (`v8::ValueSerializer`/`v8::ValueDeserializer`), shuning uchun JS-level recursive clone'dan tezroq ishlashi kutiladi. `JSON.parse(JSON.stringify())` usuli esa faqat JSON-safe (non-circular, primitive qiymatlar, plain object/array) data uchun ishlaydi — V8 ning JSON parser'i juda optimallashtirilgan bo'lgani uchun bu yondashuv ham amaliyotda samarali. Aniq tezlik farqlari workload va object strukturasiga bog'liq — production kodda benchmark bilan tekshirish tavsiya etiladi.

Copy usullarini taqqoslash:

| Xususiyat | Spread / assign | JSON hack | `structuredClone` | Recursive |
|---|---|---|---|---|
| Nested objects | Shallow (faqat birinchi level) | ✅ Deep | ✅ Deep | ✅ Deep |
| Circular ref | ❌ shallow | ❌ Error | ✅ | ✅ (WeakMap orqali) |
| Date | reference copy | → string bo'lib qoladi | ✅ saqlanadi | ✅ (manual handling) |
| Map/Set | reference copy | yo'qoladi (empty obj) | ✅ saqlanadi | ✅ (manual handling) |
| Function | reference copy | yo'qoladi | ❌ DataCloneError | reference copy |
| RegExp | reference copy | yo'qoladi (empty obj) | ✅ saqlanadi | ✅ (manual handling) |
| undefined | ✅ saqlanadi | yo'qoladi | ✅ saqlanadi | ✅ saqlanadi |
| Symbol keys | ✅ (spread) | yo'qoladi | yo'qoladi (silent drop) | ✅ (Reflect.ownKeys orqali) |
| Complexity | Juda oddiy | Oddiy | Oddiy | Yuqori (qo'lda yozish) |

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da property enumeration uchun ikki asosiy internal method va bir nechta abstract operation mavjud:

**`[[OwnPropertyKeys]]()`** — object ning barcha own property key'larini qaytaradi. Spec bo'yicha key'lar qat'iy tartibda qaytariladi:
1. Integer index property'lar — ascending numeric order (0, 1, 2, ...)
2. String property'lar — yaratilish (insertion) tartibida
3. Symbol property'lar — yaratilish tartibida

`Reflect.ownKeys()` to'g'ridan-to'g'ri `[[OwnPropertyKeys]]()` ni chaqiradi — shuning uchun u BARCHA key'larni (string + symbol, enumerable + non-enumerable) qaytaradi.

**`EnumerableOwnProperties(O, kind)`** abstract operation esa `Object.keys()`, `Object.values()`, `Object.entries()` uchun ishlatiladi. Bu operation:
1. `[[OwnPropertyKeys]]()` ni chaqiradi
2. Har bir key uchun `[[GetOwnProperty]](key)` bilan descriptor oladi
3. Faqat `enumerable: true` bo'lgan property'larni filtrlaydi
4. `kind` parametriga qarab key, value, yoki [key, value] qaytaradi

`for...in` loop uchun esa spec da `EnumerateObjectProperties(O)` abstract operation ishlatiladi — bu **prototype chain** bo'ylab ham yuradi. Engine `[[GetPrototypeOf]]()` orqali prototype chain ni traverse qiladi va har bir level dagi enumerable string key'larni yig'adi (Symbol key'lar skip qilinadi). Lekin spec tartibni kafolatlamaydi — V8 amalda `[[OwnPropertyKeys]]` tartibiga amal qiladi, lekin inherited property'lar tartibi implementation-defined.

V8 da `Object.keys()` kabi operatsiyalar uchun **enum cache** mavjud — agar object ning Hidden Class (Map) o'zgarmagan bo'lsa, oldingi enumeration natijasi cache dan qaytariladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Zamonaviy Object Metodlari

### Nazariya

ES2022 va ES2024 da `Object` global obyektiga ikki muhim static method qo'shildi: `Object.hasOwn()` va `Object.groupBy()`. Ular eski API'larning kamchiliklarini hal qiladi va zamonaviy JavaScript da keng qo'llaniladi.

**`Object.hasOwn(obj, key)`** — `hasOwnProperty` ning xavfsiz almashtiruvchisi. ESLint ham `no-prototype-builtins` qoidasi orqali instance method o'rniga shu static method'ni tavsiya qiladi.

**`Object.groupBy(items, keyFn)`** — array elementlarini callback natijasi bo'yicha guruhlash. Ilgari `reduce` bilan qo'lda yozish kerak edi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`Object.hasOwn` nima uchun yaxshiroq**:

ES2022 dan oldin, `hasOwnProperty` instance method sifatida `Object.prototype` da edi. Bu ikki muammoni keltirib chiqaradi:

1. **`Object.create(null)`**: Prototype'siz object'larda `hasOwnProperty` mavjud emas — `TypeError`
2. **Override**: Object o'z `hasOwnProperty` ni override qilishi mumkin — natija noto'g'ri

`Object.hasOwn` static method sifatida shu muammolarni hal qiladi: u har doim `Object` orqali chaqiriladi, hech qachon override qilinmaydi.

Spec implementatsiyasi:
```
Object.hasOwn(obj, key):
  1. O = ToObject(obj)
  2. P = ToPropertyKey(key)
  3. return HasOwnProperty(O, P)
```

`HasOwnProperty` abstract operation `[[GetOwnProperty]]` internal method'ni chaqiradi va prototype chain'ga **kirmaydi** — faqat object'ning own property'larini tekshiradi.

**`Object.groupBy` algoritmi**:

```
Object.groupBy(items, callbackFn):
  1. groups = Object.create(null)  ← prototype'siz object
  2. for each item in items (with index):
     a. key = callbackFn(item, index)
     b. propertyKey = ToPropertyKey(key)
     c. if groups[propertyKey] does not exist:
        groups[propertyKey] = []
     d. groups[propertyKey].push(item)
  3. return groups
```

Eng muhimi: `groups` — `Object.create(null)` bilan yaratilgan, **prototype'siz** object. Bu degani:
- `__proto__`, `toString`, `constructor` kabi inherited property'lar yo'q
- Xavfsiz key sifatida ishlatilishi mumkin — hech qanday property collision yo'q
- `for...in` ham toza ishlaydi

`Map.groupBy()` ham mavjud — natija `Map` qaytaradi (key'lar object yoki primitive bo'lishi mumkin).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`Object.hasOwn()` (ES2022)** — `hasOwnProperty` ning zamonaviy versiyasi:

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

**`Object.groupBy()` (ES2024)** — array elementlarini callback natijasi bo'yicha guruhlash:

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da computed property name **ComputedPropertyName** grammar production orqali parse qilinadi: `[ AssignmentExpression ]`. Object literal evaluate bo'lganda har bir property uchun **PropertyDefinitionEvaluation** abstract operation chaqiriladi.

Computed property uchun jarayon:
1. `[ ]` ichidagi expression **runtime** da evaluate qilinadi (compile-time emas)
2. Natija `ToPropertyKey()` abstract operation orqali property key ga aylantiriladi
3. `ToPropertyKey` ichida `ToPrimitive(argument, "string")` chaqiriladi — agar natija Symbol bo'lsa Symbol qaytadi, aks holda `ToString()` qo'llaniladi
4. Hosil bo'lgan key bilan `CreateDataPropertyOrThrow(object, key, value)` chaqiriladi

Bu degani computed property name sifatida ixtiyoriy expression ishlatish mumkin — `Symbol()`, funksiya chaqiruvi, ternary operator, hatto `await` expression ham. Lekin key har safar object literal evaluate bo'lganda **qayta hisoblanadi**.

V8 da object literal ichidagi computed property'lar **boilerplate optimization** ni buzadi. Oddiy (non-computed) property'lar uchun V8 compile-time da `ObjectBoilerplateDescription` yaratadi va keyin `CloneObjectIC` orqali tez clone qiladi. Computed property bor bo'lsa bu optimization ishlamaydi — har safar property'lar runtime da bitta-bitta qo'shiladi, bu esa yangi Hidden Class transition'lar ketma-ketligiga olib keladi.

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

</details>

---

## Edge Cases va Gotchas

### `structuredClone` class instance'larni "plain object"ga aylantiradi

`structuredClone` HTML structured clone algoritmi bilan ishlaydi — u object'ning **ma'lumotini** clone qiladi, lekin **prototype**'ni saqlamaydi. Class instance'larni clone qilganda prototype yo'qoladi va natija oddiy `Object` bo'ladi — method'lar ishlamaydi.

```javascript
class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  greet() { return `Hi, I'm ${this.name}`; }
}

const alice = new User("Alice", 25);
alice.greet(); // "Hi, I'm Alice" ✅

const cloned = structuredClone(alice);
console.log(cloned.name); // "Alice" — ma'lumot saqlanadi
console.log(cloned instanceof User); // false — prototype YO'Q
// cloned.greet(); // ❌ TypeError: cloned.greet is not a function
```

**Yechim:** Agar class instance clone qilish kerak bo'lsa — constructor'ni qayta chaqirish: `new User(cloned.name, cloned.age)`. Yoki class ichida `clone()` method yozish — `return new User(this.name, this.age)`.

---

### `Object.freeze` va `const` — farqi

Ko'p dasturchilar bularni bir xil deb o'ylaydi. Aslida ular boshqa darajada ishlaydi: `const` **variable binding**'ni o'zgartirilmas qiladi (o'zgaruvchini qayta assign qilib bo'lmaydi), `Object.freeze` esa **object ma'lumotini** (content'ini) o'zgartirilmas qiladi.

```javascript
// const — binding immutable, lekin content mutable
const user = { name: "Alice" };
user.name = "Bob";        // ✅ Ishlaydi — object content o'zgartirish mumkin
// user = { name: "Bob" }; // ❌ TypeError — binding o'zgartirib bo'lmaydi

// Object.freeze — content immutable, lekin binding (agar let bo'lsa) mutable
let frozen = Object.freeze({ name: "Alice" });
frozen.name = "Bob";       // ❌ silent fail / strict TypeError — content lock
frozen = { name: "Bob" };  // ✅ Ishlaydi — let binding o'zgartirish mumkin

// Ikkalasini birga ishlatish — to'liq immutable:
const truly = Object.freeze({ name: "Alice" });
truly.name = "Bob";  // ❌ content lock
// truly = {};       // ❌ binding lock
```

**Yechim:** "Fully immutable" ma'lumot kerak bo'lsa — `const` + `Object.freeze` birgalikda. Nested object'lar uchun `deepFreeze()` kerak.

---

### Symbol keys `JSON.stringify` va `for...in`'da ko'rinmaydi

Symbol-keyed property'lar oddiy enumeration va serialization method'laridan yashirin. Bu ba'zan foydali (metadata uchun), ba'zan kutilmagan bug.

```javascript
const id = Symbol("id");
const obj = {
  name: "Alice",
  [id]: 12345
};

// for...in — faqat string keys
for (const key in obj) {
  console.log(key); // "name" — Symbol key yo'q
}

// Object.keys — faqat string keys
console.log(Object.keys(obj)); // ["name"]

// JSON.stringify — Symbol keys butunlay ignore qilinadi
console.log(JSON.stringify(obj)); // '{"name":"Alice"}'

// Symbol keyga kirish — alohida API kerak:
console.log(Object.getOwnPropertySymbols(obj)); // [Symbol(id)]
console.log(obj[id]); // 12345
console.log(Reflect.ownKeys(obj)); // ["name", Symbol(id)] — hammasi
```

**Yechim:** Symbol keys metadata uchun mukammal (serialization'dan yashirin). Lekin agar barcha data iterate qilish kerak bo'lsa — `Reflect.ownKeys()` ishlating, u string va Symbol keys'ni birga qaytaradi.

---

### `in` operator prototype chain bo'ylab qidiradi

`in` operator object'da property mavjudmi yoki yo'qligini tekshiradi, LEKIN u **prototype chain**'ni ham qamrab oladi. Bu `Object.hasOwn()` dan asosiy farqi.

```javascript
const parent = { inherited: "from parent" };
const child = Object.create(parent);
child.own = "child property";

// "in" operator — own + inherited
console.log("own" in child);       // true
console.log("inherited" in child); // true ← prototype chain'dan
console.log("toString" in child);  // true ← Object.prototype'dan!

// Object.hasOwn — faqat own
console.log(Object.hasOwn(child, "own"));       // true
console.log(Object.hasOwn(child, "inherited")); // false ← own emas
console.log(Object.hasOwn(child, "toString"));  // false
```

**Yechim:** "Object'ning o'zida bu property bormi?" degan savol uchun `Object.hasOwn()` ishlating. "Bu property'ga murojaat qilsam u ishlaydimi?" degan savol uchun `in` ishlatish mumkin.

---

### Integer-like string keys numeric order'da enumerate qilinadi

JavaScript object property tartibi ko'rinadigandek — insertion order bo'yicha. LEKIN **integer-like string keys** (ya'ni `"0"`, `"1"`, `"100"` kabi) har doim **numeric ascending order**'da keladi, qolgan string keys'dan **oldin**. Bu kutilmagan tartib'ga olib keladi.

```javascript
const obj = {
  "10": "ten",
  "1": "one",
  "banana": "fruit",
  "2": "two",
  "apple": "fruit"
};

console.log(Object.keys(obj));
// ["1", "2", "10", "banana", "apple"]
// ❌ Insertion tartibida emas!
// ✅ Integer-like keys avval (1, 2, 10 — ascending numeric)
// ✅ Keyin string keys insertion tartibida ("banana", "apple")

// Bu spec tomonidan kafolatlangan tartib:
// 1. Integer index keys — ascending
// 2. String keys — insertion order
// 3. Symbol keys — insertion order
```

**Nima uchun:** ECMAScript spec bo'yicha `[[OwnPropertyKeys]]()` internal method shu tartibda qaytaradi. Bu arrays bilan consistency uchun kiritilgan (array index'lar ham string: `arr["0"]`, `arr["1"]`).

**Yechim:** Agar aniq tartib kerak bo'lsa — `Map` ishlating (insertion order kafolatlangan, har xil key turlari orasida tartib buzilmaydi) yoki `Object.keys()` natijasini manually sort qiling.

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
// ❌ original ichida function bor — structuredClone BUTUN ob'ektni rad etadi:
try {
  const copy = structuredClone(original);
} catch (e) {
  console.log(e); // DOMException (DataCloneError) — fn tufayli BUTUN clone rad etiladi
}

// ✅ Function'siz ob'ektda structuredClone to'liq ishlaydi:
const safe = {
  date: new Date(),
  pattern: /test/gi,
  undef: undefined,              // ✅ structuredClone undefined'ni SAQLAYDI
  map: new Map([["a", 1]]),
  set: new Set([1, 2, 3])
};
const copy = structuredClone(safe);
// ✅ date → Date object saqlanadi
// ✅ pattern → RegExp saqlanadi
// ✅ undef → undefined sifatida saqlanadi (JSON bilan yo'qolar edi)
// ✅ map → Map saqlanadi
// ✅ set → Set saqlanadi
```

**Nima uchun:** JSON faqat JSON-safe qiymatlarni qo'llab-quvvatlaydi. `undefined`, `Function`, `Symbol` yo'qoladi; `Infinity`, `NaN` → `null` ga aylanadi; `Date` → string ga aylanadi; `Map`, `Set`, `RegExp` → bo'sh object ga aylanadi. `structuredClone` esa HTML spec algoritmi bilan ishlaydi — u `undefined`, `Date`, `Map`, `Set`, `RegExp` va circular reference'larni yaxshi saqlaydi, lekin `Function`, `Symbol` va DOM node'larni rad etadi (`DataCloneError`).

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

**Savol:** `deepEqual(a, b)` funksiyasini yozing. Ikki qiymatning chuqur tengligini tekshiradi.

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
