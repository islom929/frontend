# Bo'lim 23: Proxy va Reflect

> Proxy — object ning fundamental operatsiyalarini (property o'qish, yozish, o'chirish, funksiya chaqirish, prototype bilan ishlash va boshqalar) ushlash va qayta belgilash imkonini beradigan meta-programming vositasi. Reflect — shu operatsiyalarni standart va xavfsiz usulda bajarish uchun API.

---

## Mundarija

- [Meta-Programming va Proxy Tushunchasi](#meta-programming-va-proxy-tushunchasi)
- [Proxy Yaratish](#proxy-yaratish)
- [get va set Trap'lar](#get-va-set-traplar)
- [has, deleteProperty, ownKeys](#has-deleteproperty-ownkeys)
- [apply va construct](#apply-va-construct)
- [Descriptor va Prototype Trap'lar](#descriptor-va-prototype-traplar)
- [Reflect API](#reflect-api)
- [Reflect + Proxy — receiver va Invariants](#reflect--proxy--receiver-va-invariants)
- [Revocable Proxy](#revocable-proxy)
- [Amaliy Use Cases](#amaliy-use-cases)
- [Performance Considerations](#performance-considerations)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Meta-Programming va Proxy Tushunchasi

### Nazariya

Meta-programming — dastur o'zining tuzilishi yoki xatti-harakatini runtime da tekshirish va o'zgartirish imkoniyati. JavaScript da meta-programming uchta darajada mavjud:

1. **Introspection** — object tuzilishini o'qish (tekshirish). `Object.keys()`, `typeof`, `instanceof` — bular introspection. Dastur o'zining tuzilishini ko'radi, lekin o'zgartirmaydi.

2. **Self-modification** — dastur o'zini runtime da o'zgartiradi. `Object.defineProperty()` bilan property xatti-harakatini qayta belgilash, `delete` bilan property o'chirish — bular self-modification.

3. **Intercession** — dasturning fundamental operatsiyalarini **ushlash va qayta belgilash**. Proxy aynan shu darajada ishlaydi — object ustidagi har qanday operatsiyani intercept qilish imkonini beradi.

Proxy intercession uchun yaratilgan. ECMAScript spetsifikatsiyasi har bir object uchun 13 ta **essential internal method** belgilaydi: `[[Get]]`, `[[Set]]`, `[[HasProperty]]`, `[[Delete]]`, `[[Call]]`, `[[Construct]]` va boshqalar. Oddiy object'larda bu metodlar standart xatti-harakatga ega. Proxy esa bu internal method'larning har birini **custom logic** bilan almashtirishga imkon beradi.

### Under the Hood

ECMAScript spec bo'yicha har bir object — bu internal method'lar to'plami. Ordinary object'larda bu metodlar standart algoritm bo'yicha ishlaydi. Proxy esa **exotic object** — uning internal method'lari handler'dagi trap funksiyalarni chaqiradi.

```
ECMAScript Internal Methods → Proxy Traps:

[[Get]]                    → handler.get()
[[Set]]                    → handler.set()
[[HasProperty]]            → handler.has()
[[Delete]]                 → handler.deleteProperty()
[[Call]]                   → handler.apply()
[[Construct]]              → handler.construct()
[[OwnPropertyKeys]]       → handler.ownKeys()
[[GetOwnProperty]]        → handler.getOwnPropertyDescriptor()
[[DefineOwnProperty]]     → handler.defineProperty()
[[GetPrototypeOf]]        → handler.getPrototypeOf()
[[SetPrototypeOf]]        → handler.setPrototypeOf()
[[IsExtensible]]          → handler.isExtensible()
[[PreventExtensions]]     → handler.preventExtensions()
```

Agar handler'da tegishli trap bo'lmasa — Proxy operatsiyani to'g'ridan-to'g'ri target object'ga yo'naltiradi (transparent forwarding). Bu degani trap'siz Proxy asl object kabi ishlaydi.

---

## Proxy Yaratish

### Nazariya

`new Proxy(target, handler)` — Proxy constructor ikki argument oladi:

- **target** — proxy qiladigan asl object. Bu oddiy object, array, funksiya, boshqa proxy — har qanday object bo'lishi mumkin. Proxy bu object ga **o'rab olinadi** (wrap qilinadi).
- **handler** — trap'lar (interceptor funksiyalar) to'plami. Bu oddiy object bo'lib, ichida 13 ta mumkin bo'lgan trap metodlaridan keraklilari belgilanadi.

Proxy yaratilgandan keyin — barcha operatsiyalar proxy orqali o'tadi. Proxy'ga yozilgan property target'da ham o'zgaradi. Proxy va target bitta object'ga murojaat qiladi — proxy faqat **oraliq qatlam**.

### Kod Misollari

Eng oddiy proxy — handler bo'sh object. Barcha operatsiyalar to'g'ridan-to'g'ri target'ga yo'naltiriladi:

```javascript
const target = { name: "Ali", age: 25 };
const proxy = new Proxy(target, {}); // bo'sh handler — transparent proxy

proxy.name;       // "Ali" — to'g'ridan-to'g'ri target'dan o'qiladi
proxy.age = 30;   // target.age ham 30 bo'ladi
console.log(target.age); // 30 — proxy va target bitta object
```

Handler'da trap qo'shish — operatsiyani ushlash:

```javascript
const user = { name: "Ali", age: 25 };

const proxy = new Proxy(user, {
  get(target, property, receiver) {
    console.log(`GET: ${property}`);
    return target[property];
  },
  set(target, property, value, receiver) {
    console.log(`SET: ${property} = ${value}`);
    target[property] = value;
    return true; // set trap true qaytarishi shart
  }
});

proxy.name;       // GET: name → "Ali"
proxy.age = 26;   // SET: age = 26
console.log(user.age); // 26 — target ham o'zgardi
```

### Under the Hood

```
Proxy arxitekturasi:

               Proxy Object
Kod ────────▶ ┌──────────────┐ ──────▶ Target Object
              │   Handler    │
              │              │
proxy.name    │ get() trap   │────▶ target.name
proxy.x = 1   │ set() trap   │────▶ target.x = 1
delete proxy.y│ deleteProperty()│──▶ delete target.y
"z" in proxy  │ has() trap   │────▶ "z" in target
proxy()       │ apply() trap │────▶ target()
new proxy()   │ construct()  │────▶ new target()
              │              │
              │ Trap yo'q?   │────▶ To'g'ridan-to'g'ri target ga
              └──────────────┘
```

V8 engine'da Proxy — `JSProxy` degan maxsus internal type. U oddiy `JSObject` dan farqli ravishda har bir property access'da handler'ni tekshiradi. Shuning uchun Proxy oddiy object'dan **sekinroq** — har bir operatsiyada qo'shimcha function call bo'ladi.

---

## get va set Trap'lar

### Nazariya

`get` va `set` — eng ko'p ishlatiladigan trap'lar. Object'dan property o'qish va yozishni ushlaydi.

**get(target, property, receiver):**
- `target` — asl object
- `property` — o'qilayotgan property nomi (string yoki Symbol)
- `receiver` — proxy o'zi yoki proxy'dan meros olgan object. Getter'larda `this` sifatida ishlatiladi.

**set(target, property, value, receiver):**
- `target` — asl object
- `property` — yozilayotgan property nomi
- `value` — yangi qiymat
- `receiver` — proxy o'zi yoki meros olgan object
- **Return:** `true` qaytarishi **shart**. `false` yoki `undefined` qaytarsa strict mode'da `TypeError`.

### Kod Misollari

Property o'qishda default qiymat berish — yo'q property uchun `undefined` o'rniga fallback:

```javascript
const defaults = { color: "black", size: "M", quantity: 1 };

const config = new Proxy({}, {
  get(target, prop) {
    // target da bor bo'lsa — target dan, yo'q bo'lsa — defaults dan
    return prop in target ? target[prop] : defaults[prop];
  }
});

console.log(config.color);    // "black" — default
config.color = "red";
console.log(config.color);    // "red" — yangi qiymat
console.log(config.size);     // "M" — hali default
console.log(config.missing);  // undefined — defaults da ham yo'q
```

Property yozishda validatsiya — noto'g'ri qiymatni rad etish:

```javascript
const user = new Proxy({ name: "", age: 0, email: "" }, {
  set(target, prop, value) {
    if (prop === "age") {
      if (typeof value !== "number") {
        throw new TypeError(`age raqam bo'lishi kerak, ${typeof value} berildi`);
      }
      if (value < 0 || value > 150) {
        throw new RangeError(`age 0-150 orasida bo'lishi kerak, ${value} berildi`);
      }
    }
    if (prop === "email") {
      if (typeof value !== "string" || !value.includes("@")) {
        throw new Error(`email noto'g'ri format: "${value}"`);
      }
    }
    target[prop] = value;
    return true;
  }
});

user.name = "Ali";              // ✅
user.age = 25;                  // ✅
user.email = "ali@example.com"; // ✅
// user.age = "yigirma";        // ❌ TypeError: age raqam bo'lishi kerak
// user.age = -5;               // ❌ RangeError: age 0-150 orasida
// user.email = "invalid";      // ❌ Error: email noto'g'ri format
```

### Under the Hood

`get` trap ishga tushish ketma-ketligi:

1. `proxy.prop` yoki `proxy["prop"]` yoziladi
2. Engine proxy'ning `[[Get]]` internal method'ini chaqiradi
3. `[[Get]]` handler object'dan `get` trap'ni qidiradi
4. Agar `get` trap bor — `get(target, "prop", receiver)` chaqiriladi
5. Agar `get` trap yo'q — `Reflect.get(target, "prop", receiver)` chaqiriladi (standart xatti-harakat)

`set` trap uchun ham xuddi shunday — lekin `set` trap'dan qaytgan qiymat tekshiriladi. `false` yoki falsy qiymat qaytsa va strict mode bo'lsa — `TypeError` tashlaydi.

```
get trap flow:

proxy.name
  │
  ▼
[[Get]]("name", proxy)
  │
  ├── handler.get mavjud? ──▶ handler.get(target, "name", proxy)
  │                              │
  │                              └── Custom qiymat qaytaradi
  │
  └── handler.get yo'q? ──────▶ Reflect.get(target, "name", proxy)
                                    │
                                    └── target.name qaytaradi
```

---

## has, deleteProperty, ownKeys

### Nazariya

Bu trap'lar property **mavjudligini tekshirish**, **o'chirish** va **sanab chiqish** operatsiyalarini ushlaydi.

**has(target, property):**
- `in` operatori ishga tushganda chaqiriladi: `"prop" in proxy`
- `true` yoki `false` qaytarishi kerak
- `for...in` loop ham `has` trap'ni chaqiradi

**deleteProperty(target, property):**
- `delete proxy.prop` ishga tushganda chaqiriladi
- `true` qaytarsa — o'chirish muvaffaqiyatli (lekin aslida o'chirish developer ixtiyorida)
- `false` qaytarsa strict mode'da `TypeError`

**ownKeys(target):**
- `Object.keys()`, `Object.getOwnPropertyNames()`, `Object.getOwnPropertySymbols()`, `for...in`, `JSON.stringify()` ishga tushganda chaqiriladi
- String va Symbol'lardan iborat array qaytarishi kerak

### Kod Misollari

Private property'larni yashirish — `_` bilan boshlanadigan property'larga tashqaridan kirishni taqiqlash:

```javascript
const data = {
  _id: "abc123",
  _token: "secret-jwt-token",
  name: "Ali",
  age: 25
};

const safeData = new Proxy(data, {
  // in operatorida private field'larni yashirish
  has(target, prop) {
    if (typeof prop === "string" && prop.startsWith("_")) {
      return false; // "_id" in safeData → false
    }
    return Reflect.has(target, prop);
  },

  // Object.keys() da private field'larni yashirish
  ownKeys(target) {
    return Reflect.ownKeys(target).filter(
      key => typeof key !== "string" || !key.startsWith("_")
    );
  },

  // get — private field o'qishni bloklash
  get(target, prop, receiver) {
    if (typeof prop === "string" && prop.startsWith("_")) {
      return undefined;
    }
    return Reflect.get(target, prop, receiver);
  },

  // delete — private field o'chirishni taqiqlash
  deleteProperty(target, prop) {
    if (typeof prop === "string" && prop.startsWith("_")) {
      throw new Error(`"${prop}" private field — o'chirib bo'lmaydi`);
    }
    return Reflect.deleteProperty(target, prop);
  }
});

console.log("name" in safeData);   // true
console.log("_id" in safeData);    // false — yashirilgan
console.log(Object.keys(safeData)); // ["name", "age"] — _id, _token yo'q
console.log(safeData._token);      // undefined — kirib bo'lmaydi
// delete safeData._id;            // ❌ Error: private field
delete safeData.age;               // ✅ age o'chirildi
```

### Under the Hood

`ownKeys` trap qaytargan massiv `Object.keys()` va `Object.getOwnPropertyNames()` uchun har xil filtrlangan. `Object.keys()` faqat **enumerable** property'larni qaytaradi — shuning uchun `ownKeys` natijasi `getOwnPropertyDescriptor` trap orqali yana tekshiriladi. Agar `getOwnPropertyDescriptor` trap berilmagan bo'lsa — target'dagi haqiqiy descriptor ishlatiladi.

```
ownKeys trap ishga tushishi:

Object.keys(proxy)
  │
  ▼
[[OwnPropertyKeys]]()
  │
  ▼
handler.ownKeys(target) → ["name", "age"]
  │
  ▼ (har bir key uchun)
[[GetOwnProperty]](key)
  │
  ▼
Faqat enumerable: true bo'lganlar → Object.keys() natijasi
```

---

## apply va construct

### Nazariya

Bu trap'lar faqat **funksiya** target'lar uchun ishlaydi. Oddiy object target'da bu trap'lar chaqirilmaydi (chunki oddiy object callable emas).

**apply(target, thisArg, argumentsList):**
- Funksiyani chaqirganda ishga tushadi: `proxy()`, `proxy.call()`, `proxy.apply()`
- `target` — asl funksiya
- `thisArg` — `this` konteksti
- `argumentsList` — argumentlar massivi

**construct(target, argumentsList, newTarget):**
- `new proxy()` yozilganda ishga tushadi
- `target` — asl constructor funksiya yoki class
- `argumentsList` — argumentlar massivi
- `newTarget` — asl chaqirilgan constructor (extends bilan ishlashda muhim)
- **Object qaytarishi shart** — primitive qaytarsa `TypeError`

### Kod Misollari

Funksiya chaqiruvini ushlash — logging, timing, argument validation:

```javascript
function fetchData(url, options = {}) {
  return fetch(url, options).then(r => r.json());
}

// apply trap bilan har bir chaqiruvni loglash va timing qilish
const trackedFetch = new Proxy(fetchData, {
  apply(target, thisArg, args) {
    const [url] = args;
    console.log(`[API] ${url} ga so'rov yuborilmoqda...`);
    const start = performance.now();

    const result = Reflect.apply(target, thisArg, args);

    // Promise bo'lgani uchun .then bilan timing
    return result.then(data => {
      const duration = (performance.now() - start).toFixed(2);
      console.log(`[API] ${url} — ${duration}ms da javob keldi`);
      return data;
    });
  }
});

// trackedFetch("/api/users");
// [API] /api/users ga so'rov yuborilmoqda...
// [API] /api/users — 234.56ms da javob keldi
```

Constructor chaqiruvini ushlash — singleton pattern, instance tracking:

```javascript
class Database {
  constructor(connectionString) {
    this.connectionString = connectionString;
    this.connected = false;
  }
  connect() {
    this.connected = true;
    console.log(`${this.connectionString} ga ulandi`);
  }
}

// construct trap bilan singleton: faqat bitta instance
let instance = null;
const SingletonDB = new Proxy(Database, {
  construct(target, args, newTarget) {
    if (instance) {
      console.log("Mavjud instance qaytarilmoqda");
      return instance; // yangi yaratmay, mavjudni qaytaradi
    }
    instance = Reflect.construct(target, args, newTarget);
    console.log("Yangi instance yaratildi");
    return instance;
  }
});

const db1 = new SingletonDB("postgres://localhost/mydb");
// "Yangi instance yaratildi"

const db2 = new SingletonDB("postgres://localhost/other");
// "Mavjud instance qaytarilmoqda"

console.log(db1 === db2); // true — bitta instance
```

### Under the Hood

`apply` trap faqat `target` callable bo'lganda ishlaydi. ECMAScript spec bo'yicha object'ning `[[Call]]` internal method'i bo'lishi kerak — bu faqat funksiyalarda bor. Oddiy object'ni proxy qilib `apply` trap qo'ysangiz — `proxy()` chaqirganda `TypeError: proxy is not a function` xatosi chiqadi, chunki target'da `[[Call]]` yo'q.

`construct` trap ham xuddi shunday — faqat `target` da `[[Construct]]` internal method bor bo'lsa ishlaydi. Arrow function'larda `[[Construct]]` yo'q — shuning uchun ularni `new` bilan chaqirib bo'lmaydi, proxy qilsangiz ham.

---

## Descriptor va Prototype Trap'lar

### Nazariya

Qolgan 6 ta trap kamroq ishlatiladi, lekin to'liq meta-programming uchun kerak:

| Trap | Ushlaydi | Qachon |
|------|----------|--------|
| `getOwnPropertyDescriptor(target, prop)` | Property descriptor olish | `Object.getOwnPropertyDescriptor(proxy, prop)` |
| `defineProperty(target, prop, descriptor)` | Property descriptor belgilash | `Object.defineProperty(proxy, prop, desc)` |
| `getPrototypeOf(target)` | Prototype olish | `Object.getPrototypeOf(proxy)`, `proxy.__proto__` |
| `setPrototypeOf(target, proto)` | Prototype o'zgartirish | `Object.setPrototypeOf(proxy, proto)` |
| `isExtensible(target)` | Kengaytirish tekshirish | `Object.isExtensible(proxy)` |
| `preventExtensions(target)` | Kengaytirishni taqiqlash | `Object.preventExtensions(proxy)` |

### Kod Misollari

`getOwnPropertyDescriptor` va `defineProperty` — immutable object yaratish, property'lar o'zgartirilishini bloklash:

```javascript
const config = { apiUrl: "https://api.example.com", debug: false };

const frozenConfig = new Proxy(config, {
  // property descriptor o'zgartirishni bloklash
  defineProperty(target, prop, descriptor) {
    throw new Error(`Config property "${prop}" ni qayta define qilib bo'lmaydi`);
  },

  // prototype o'zgartirishni bloklash
  setPrototypeOf(target, proto) {
    throw new Error("Config ning prototype ini o'zgartirib bo'lmaydi");
  },

  // property yozishni bloklash
  set(target, prop, value) {
    throw new Error(`Config property "${prop}" read-only`);
  },

  // property o'chirishni bloklash
  deleteProperty(target, prop) {
    throw new Error(`Config property "${prop}" ni o'chirib bo'lmaydi`);
  }
});

console.log(frozenConfig.apiUrl); // ✅ "https://api.example.com"
// frozenConfig.apiUrl = "x";     // ❌ Error: read-only
// delete frozenConfig.debug;     // ❌ Error: o'chirib bo'lmaydi
```

`ownKeys` bilan `getOwnPropertyDescriptor` birgalikda — virtual property'lar yaratish:

```javascript
// ownKeys trap qaytargan key'lar uchun getOwnPropertyDescriptor
// ham mos descriptor qaytarishi kerak — aks holda key ko'rinmaydi
const virtualObj = new Proxy({}, {
  ownKeys() {
    return ["virtual1", "virtual2", "virtual3"];
  },
  getOwnPropertyDescriptor(target, prop) {
    // virtual property'lar uchun descriptor qaytarish
    return {
      value: `Qiymati: ${prop}`,
      enumerable: true,
      configurable: true,
      writable: true
    };
  },
  get(target, prop) {
    return `Qiymati: ${prop}`;
  }
});

console.log(Object.keys(virtualObj));
// ["virtual1", "virtual2", "virtual3"] — real property mavjud emas, lekin ko'rinadi
console.log(virtualObj.virtual1); // "Qiymati: virtual1"
```

---

## Reflect API

### Nazariya

`Reflect` — global object bo'lib, Proxy trap'lariga mos keladigan **13 ta static metod** beradi. Reflect constructor emas — `new Reflect()` qilish mumkin emas. `Math` kabi faqat static method'lar uchun namespace.

Reflect nima uchun kerak:

1. **Standart xatti-harakatni chaqirish.** Proxy trap ichida target'ning default operatsiyasini bajarish uchun. `target[prop]` o'rniga `Reflect.get(target, prop, receiver)` — receiver'ni to'g'ri uzatish.

2. **Xavfsiz qaytarish qiymati.** `Object.defineProperty()` xato tashlaydi (muvaffaqiyatsiz bo'lsa), `Reflect.defineProperty()` esa `false` qaytaradi. Try/catch yozish shart emas.

3. **Receiver — to'g'ri `this` konteksti.** Prototype chain'da getter/setter'lar bo'lganda `this` to'g'ri ishlashi uchun `receiver` argumenti juda muhim.

4. **1:1 mos kelish.** Har bir Proxy trap uchun Reflect metod bor — nomlar va argumentlar bir xil. Bu Proxy ichida default behavior'ga "forward" qilishni osonlashtiradi.

### Reflect Metodlari

```javascript
const obj = { name: "Ali", age: 25 };

// === Reflect.get(target, propertyKey [, receiver]) ===
Reflect.get(obj, "name");  // "Ali"

// === Reflect.set(target, propertyKey, value [, receiver]) ===
Reflect.set(obj, "age", 30); // true — muvaffaqiyatli
Reflect.set(obj, "role", "admin"); // true — yangi property qo'shildi

// === Reflect.has(target, propertyKey) ===
// in operatorning funksional shakli
Reflect.has(obj, "name");    // true
Reflect.has(obj, "missing"); // false

// === Reflect.deleteProperty(target, propertyKey) ===
// delete operatorning funksional shakli
Reflect.deleteProperty(obj, "role"); // true

// === Reflect.ownKeys(target) ===
// Object.keys + non-enumerable + Symbol key'lar — hammasi
Reflect.ownKeys(obj); // ["name", "age"]

// === Reflect.defineProperty(target, propertyKey, attributes) ===
// Object.defineProperty dan farqi: throw o'rniga false qaytaradi
const success = Reflect.defineProperty(obj, "id", {
  value: 1,
  writable: false,
  enumerable: true,
  configurable: false
});
console.log(success); // true

// Qayta define — writable:false va configurable:false bo'lgani uchun
const fail = Reflect.defineProperty(obj, "id", { value: 2 });
console.log(fail); // false — xato TASHLMASDAN muvaffaqiyatsizlik bildirdi

// === Reflect.getOwnPropertyDescriptor(target, propertyKey) ===
Reflect.getOwnPropertyDescriptor(obj, "id");
// { value: 1, writable: false, enumerable: true, configurable: false }

// === Reflect.apply(target, thisArgument, argumentsList) ===
// Function.prototype.apply ning xavfsiz varianti
Reflect.apply(Math.max, null, [1, 5, 3]); // 5
Reflect.apply(String.prototype.toUpperCase, "hello", []); // "HELLO"

// === Reflect.construct(target, argumentsList [, newTarget]) ===
// new operator ning funksional shakli
function User(name) { this.name = name; }
const user = Reflect.construct(User, ["Ali"]);
console.log(user.name); // "Ali"
console.log(user instanceof User); // true

// === Reflect.getPrototypeOf(target) ===
Reflect.getPrototypeOf(user); // User.prototype

// === Reflect.setPrototypeOf(target, proto) ===
Reflect.setPrototypeOf(user, null); // true — prototype o'chirildi

// === Reflect.isExtensible(target) ===
Reflect.isExtensible(obj); // true

// === Reflect.preventExtensions(target) ===
Reflect.preventExtensions(obj); // true
Reflect.isExtensible(obj); // false — endi yangi property qo'shib bo'lmaydi
```

### Under the Hood

Reflect va Object metod'larining asosiy farqi — xatolarga munosabat:

| Operatsiya | Object | Reflect |
|-----------|--------|---------|
| Non-configurable property'ni qayta define | `TypeError` tashlaydi | `false` qaytaradi |
| Non-object'da ishlash | `TypeError` tashlaydi | `TypeError` tashlaydi |
| `preventExtensions` qilingan object'ga property qo'shish | `TypeError` tashlaydi | `false` qaytaradi |
| `defineProperty` muvaffaqiyatli | Object qaytaradi | `true` qaytaradi |
| `deleteProperty` non-configurable | `TypeError` (strict) | `false` qaytaradi |

Reflect metod'lari **boolean** qaytargani uchun `if` bilan tekshirish oson:

```javascript
// ❌ Object.defineProperty — try/catch kerak
try {
  Object.defineProperty(obj, "id", { value: 2 });
} catch (e) {
  console.log("Define bo'lmadi");
}

// ✅ Reflect.defineProperty — if bilan tekshirish
if (!Reflect.defineProperty(obj, "id", { value: 2 })) {
  console.log("Define bo'lmadi");
}
```

---

## Reflect + Proxy — receiver va Invariants

### Nazariya

Proxy trap ichida `target[prop]` o'rniga `Reflect.get(target, prop, receiver)` ishlatish — **best practice**. Buning sababi — **receiver** argumenti. Receiver — getter/setter funksiyalarda `this` qiymatini belgilaydi. `target[prop]` qilsangiz, getter'dagi `this` doim `target` ga ishora qiladi. Lekin agar proxy'dan meros olingan bo'lsa — `this` receiver (ya'ni proxy yoki proxy child) bo'lishi kerak.

**ECMAScript Proxy Invariants** — trap'lar ECMAScript spec qoidalariga (invariant'larga) rioya qilishi **shart**. Bu qoidalar buzilsa engine `TypeError` tashlaydi. Asosiy invariant'lar:

| Trap | Invariant |
|------|-----------|
| `get` | Non-writable, non-configurable property uchun target'dagi qiymatdan boshqa qiymat qaytarib bo'lmaydi |
| `set` | Non-writable, non-configurable property ga yozish `true` qaytara olmaydi |
| `has` | Non-configurable property uchun `false` qaytarib bo'lmaydi |
| `deleteProperty` | Non-configurable property uchun `true` qaytarib bo'lmaydi |
| `ownKeys` | Target'dagi non-configurable property'lar natijada bo'lishi **shart** |
| `getPrototypeOf` | Non-extensible object uchun haqiqiy prototype'dan boshqa qiymat qaytarib bo'lmaydi |
| `isExtensible` | Target bilan bir xil bo'lishi **shart** |

### Kod Misollari

`receiver` muammosi — prototype chain'da getter bilan:

```javascript
const parent = {
  _name: "Parent",
  get name() {
    return this._name; // this — kim chaqirsa, shu
  }
};

// ❌ Noto'g'ri — target[prop] ishlatish
const badProxy = new Proxy(parent, {
  get(target, prop) {
    return target[prop]; // this = target (parent), receiver emas
  }
});

const child = Object.create(badProxy);
child._name = "Child";
console.log(child.name); // "Parent" ❌ — this = parent bo'lib qoldi!

// ✅ To'g'ri — Reflect.get bilan receiver uzatish
const goodProxy = new Proxy(parent, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver); // this = receiver (child)
  }
});

const child2 = Object.create(goodProxy);
child2._name = "Child";
console.log(child2.name); // "Child" ✅ — this = child2
```

Invariant buzilishi — engine `TypeError` tashlaydi:

```javascript
const obj = {};
Object.defineProperty(obj, "id", {
  value: 42,
  writable: false,
  configurable: false
});

const proxy = new Proxy(obj, {
  get(target, prop) {
    if (prop === "id") return 999; // invariant buzilishi!
    return Reflect.get(target, prop);
  }
});

// proxy.id; // ❌ TypeError: 'get' on proxy: property 'id' is a read-only
//           // and non-configurable data property on the proxy target
//           // but the proxy did not return its actual value
```

### Under the Hood

Invariant'lar nima uchun kerak? Ular JavaScript'ning **xavfsizlik** va **bashorat qilish** garantiyalarini saqlaydi. Agar kod `Object.freeze(obj)` bilan object'ni muzlatsa — hech kim (hatto Proxy ham) bu kafolatni buza olmaydi. Bu `Object.freeze`, `Object.seal`, `Object.preventExtensions` kabi operatsiyalarning ishonchliligini ta'minlaydi.

```
Invariant tekshiruv flow:

proxy.id  (non-writable, non-configurable property)
  │
  ▼
handler.get(target, "id", receiver)
  │
  ▼ (return 999)
  │
  ▼
Engine invariant check:
  ├── target.id = 42
  ├── writable: false, configurable: false
  ├── trap qaytargan qiymat: 999
  ├── 999 !== 42 → INVARIANT BUZILDI!
  └── TypeError tashlaydi
```

---

## Revocable Proxy

### Nazariya

`Proxy.revocable(target, handler)` — vaqtinchalik proxy yaratadi. Qaytargan object'da ikkita property bor: `proxy` (Proxy o'zi) va `revoke` (bekor qilish funksiyasi). `revoke()` chaqirilgandan keyin proxy'ning **barcha** trap'lari `TypeError` tashlaydi — proxy butunlay ishlamay qoladi. Bu mexanizm vaqtinchalik access berish, security, va resource management uchun ishlatiladi.

Revoke qilingan proxy'ni qayta tiklab bo'lmaydi — bu **bir tomonlama** operatsiya. `revoke()` chaqirilgandan keyin proxy ichki `[[Handler]]` va `[[Target]]` reference'larini `null` ga o'zgartiradi — target object'ga boshqa proxy orqali kirib bo'lmaydi.

### Kod Misollari

Vaqtinchalik API access — ma'lum vaqtdan keyin bekor qilish:

```javascript
function createTimedAccess(target, durationMs) {
  const { proxy, revoke } = Proxy.revocable(target, {
    get(target, prop, receiver) {
      console.log(`[access] ${String(prop)} o'qildi`);
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      console.log(`[access] ${String(prop)} = ${value} yozildi`);
      return Reflect.set(target, prop, value, receiver);
    }
  });

  // belgilangan vaqtdan keyin access bekor qilinadi
  setTimeout(() => {
    revoke();
    console.log("[access] Muddat tugadi — access bekor qilindi");
  }, durationMs);

  return proxy;
}

const secretData = { apiKey: "sk-123", balance: 50000 };
const tempAccess = createTimedAccess(secretData, 5000);

tempAccess.apiKey;     // ✅ [access] apiKey o'qildi → "sk-123"
tempAccess.balance;    // ✅ [access] balance o'qildi → 50000

// 5 sekunddan keyin:
// [access] Muddat tugadi — access bekor qilindi
// tempAccess.apiKey;  // ❌ TypeError: Cannot perform 'get' on a proxy
//                     //    that has been revoked
```

Third-party library'ga object berish — keyin access ni olish:

```javascript
function giveTemporaryAccess(sensitiveData) {
  const { proxy, revoke } = Proxy.revocable(sensitiveData, {});

  // third-party library proxy bilan ishlaydi
  thirdPartyLib.process(proxy);

  // library tugagandan keyin access bekor qilinadi
  revoke();
  // endi thirdPartyLib proxy orqali data'ga kira olmaydi
}
```

---

## Amaliy Use Cases

### 1. Validation Proxy — Schema-Based

```javascript
function createValidator(rules) {
  return new Proxy({}, {
    set(target, prop, value) {
      const rule = rules[prop];
      if (!rule) {
        throw new Error(`"${prop}" schema da mavjud emas. Ruxsat etilgan: ${Object.keys(rules).join(", ")}`);
      }

      const { type, min, max, required, pattern, custom } = rule;

      if (required && (value === null || value === undefined || value === "")) {
        throw new Error(`"${prop}" majburiy maydon`);
      }
      if (type && typeof value !== type) {
        throw new TypeError(`"${prop}" ${type} bo'lishi kerak, ${typeof value} berildi`);
      }
      if (typeof value === "number") {
        if (min !== undefined && value < min) {
          throw new RangeError(`"${prop}" kamida ${min} bo'lishi kerak`);
        }
        if (max !== undefined && value > max) {
          throw new RangeError(`"${prop}" ko'pi bilan ${max} bo'lishi kerak`);
        }
      }
      if (pattern && !pattern.test(String(value))) {
        throw new Error(`"${prop}" format noto'g'ri`);
      }
      if (custom && !custom(value)) {
        throw new Error(`"${prop}" custom validatsiyadan o'tmadi`);
      }

      return Reflect.set(target, prop, value);
    }
  });
}

const user = createValidator({
  name: { type: "string", required: true },
  age: { type: "number", min: 0, max: 150 },
  email: { type: "string", pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ },
  role: { type: "string", custom: v => ["admin", "user", "editor"].includes(v) }
});

user.name = "Ali";              // ✅
user.age = 25;                  // ✅
user.email = "ali@example.com"; // ✅
user.role = "admin";            // ✅
// user.age = -5;               // ❌ RangeError: kamida 0
// user.email = "invalid";      // ❌ format noto'g'ri
// user.role = "superadmin";    // ❌ custom validatsiyadan o'tmadi
// user.phone = "+998";         // ❌ schema da mavjud emas
```

### 2. Logging / Debug Proxy

```javascript
function createLogger(target, label = "Object") {
  return new Proxy(target, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      if (typeof value === "function") {
        // method chaqiruvlarni ushlash
        return new Proxy(value, {
          apply(fn, thisArg, args) {
            console.log(`[${label}] ${String(prop)}(${args.map(a => JSON.stringify(a)).join(", ")})`);
            const result = Reflect.apply(fn, thisArg, args);
            console.log(`[${label}] ${String(prop)} → ${JSON.stringify(result)}`);
            return result;
          }
        });
      }
      console.log(`[${label}] GET ${String(prop)} → ${JSON.stringify(value)}`);
      return value;
    },
    set(target, prop, value, receiver) {
      const old = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      console.log(`[${label}] SET ${String(prop)}: ${JSON.stringify(old)} → ${JSON.stringify(value)}`);
      return result;
    },
    deleteProperty(target, prop) {
      console.log(`[${label}] DELETE ${String(prop)}`);
      return Reflect.deleteProperty(target, prop);
    }
  });
}

const cart = createLogger({
  items: [],
  add(item) {
    this.items.push(item);
    return this.items.length;
  }
}, "Cart");

cart.items;              // [Cart] GET items → []
cart.add("Telefon");     // [Cart] add("Telefon") → 1
```

### 3. Caching / Memoization Proxy

```javascript
function memoize(fn) {
  const cache = new Map();
  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);
      if (cache.has(key)) {
        return cache.get(key);
      }
      const result = Reflect.apply(target, thisArg, args);
      cache.set(key, result);
      return result;
    }
  });
}

const factorial = memoize(function fact(n) {
  console.log(`Hisoblash: fact(${n})`);
  if (n <= 1) return 1;
  return n * fact(n - 1); // recursive chaqiruvlar cache'lanmaydi (fact, memoized emas)
});

factorial(5);  // Hisoblash: fact(5), fact(4), fact(3), fact(2), fact(1) → 120
factorial(5);  // 120 — cache'dan, hisoblash yo'q
factorial(3);  // Hisoblash: fact(3), fact(2), fact(1) → 6
               // (factorial(3) cache da yo'q — faqat factorial(5) bor)
```

### 4. Negative Array Index

```javascript
function createNegativeArray(arr) {
  return new Proxy(arr, {
    get(target, prop, receiver) {
      const index = Number(prop);
      if (Number.isInteger(index) && index < 0) {
        // manfiy index — oxiridan sanash
        const realIndex = target.length + index;
        if (realIndex < 0) return undefined; // chegaradan tashqarida
        return target[realIndex];
      }
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      const index = Number(prop);
      if (Number.isInteger(index) && index < 0) {
        const realIndex = target.length + index;
        if (realIndex < 0) {
          throw new RangeError(`Index ${index} chegaradan tashqarida`);
        }
        target[realIndex] = value;
        return true;
      }
      return Reflect.set(target, prop, value, receiver);
    }
  });
}

const arr = createNegativeArray([10, 20, 30, 40, 50]);
console.log(arr[-1]);  // 50 — oxirgi element
console.log(arr[-2]);  // 40 — oxiridan ikkinchi
console.log(arr[0]);   // 10 — oddiy access ishlaydi
console.log(arr.length); // 5 — length ham ishlaydi
arr[-1] = 999;
console.log(arr[4]);   // 999 — o'zgardi
```

### 5. Reactive System (Vue 3 Pattern)

Vue 3 reactivity tizimining soddalashtirilgan versiyasi — property o'qilganda **track** (dependency kuzatish), yozilganda **trigger** (effect qayta ishga tushirish):

```javascript
let activeEffect = null;
const targetMap = new WeakMap(); // target → Map(key → Set(effects))

function track(target, key) {
  if (!activeEffect) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    depsMap = new Map();
    targetMap.set(target, depsMap);
  }
  let deps = depsMap.get(key);
  if (!deps) {
    deps = new Set();
    depsMap.set(key, deps);
  }
  deps.add(activeEffect);
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const effects = depsMap.get(key);
  if (effects) {
    effects.forEach(fn => fn());
  }
}

function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key); // dependency ni kuzatishga olish
      const value = Reflect.get(target, key, receiver);
      // nested object'lar uchun recursive reactive
      if (typeof value === "object" && value !== null) {
        return reactive(value);
      }
      return value;
    },
    set(target, key, value, receiver) {
      const oldValue = target[key];
      const result = Reflect.set(target, key, value, receiver);
      if (oldValue !== value) {
        trigger(target, key); // bog'liq effect'larni qayta ishga tushirish
      }
      return result;
    }
  });
}

function effect(fn) {
  activeEffect = fn;
  fn(); // birinchi marta ishlaydi — get trap'lar orqali dependency'larni track qiladi
  activeEffect = null;
}

// === Ishlatish ===
const state = reactive({ count: 0, user: { name: "Ali" } });

effect(() => {
  console.log(`Count: ${state.count}`);
});
// "Count: 0" — darhol ishladi

state.count = 1;  // "Count: 1" — avtomatik trigger
state.count = 5;  // "Count: 5" — avtomatik trigger

effect(() => {
  console.log(`User: ${state.user.name}`);
});
// "User: Ali"

state.user.name = "Vali"; // "User: Vali" — nested reactivity
```

**Vue 2 vs Vue 3 farqi:**

```
Vue 2 (Object.defineProperty):          Vue 3 (Proxy):
├── Property oldindan aniqlangan        ├── Dynamic property qo'shish ✅
│   bo'lishi kerak                      ├── Property o'chirish kuzatiladi ✅
├── Array index kuzatilmaydi            ├── Array o'zgarishlari to'liq ✅
├── Vue.set() kerak yangi prop uchun    ├── Map, Set qo'llab-quvvat ✅
├── Har bir prop uchun alohida          ├── Bitta proxy butun object ✅
│   getter/setter                       ├── Tezroq va kamroq memory ✅
└── Sekinroq — katta object'larda       └── Lazy — faqat kirish bo'lganda
```

### 6. Data Binding — Model-View Sync

```javascript
function createBoundModel(initialData, renderFn) {
  const proxy = new Proxy(initialData, {
    set(target, prop, value, receiver) {
      const result = Reflect.set(target, prop, value, receiver);
      // property o'zgarganda view ni qayta render qilish
      renderFn(proxy);
      return result;
    }
  });

  // boshlang'ich render
  renderFn(proxy);
  return proxy;
}

// Ishlatish (console-based demo)
const model = createBoundModel(
  { title: "Salom", count: 0 },
  (data) => {
    console.log(`[View] Title: ${data.title} | Count: ${data.count}`);
  }
);
// [View] Title: Salom | Count: 0

model.count = 1; // [View] Title: Salom | Count: 1
model.title = "Xayr"; // [View] Title: Xayr | Count: 1
```

---

## Performance Considerations

### Nazariya

Proxy har bir operatsiyada qo'shimcha function call qo'shadi — bu **overhead** yaratadi. V8 engine Proxy'ni oddiy object'dek optimize qila olmaydi — inline caching va hidden class optimization Proxy uchun ishlamaydi.

**Benchmark natijalar (taxminiy):**

| Operatsiya | Oddiy Object | Proxy (bo'sh handler) | Proxy (trap bilan) |
|-----------|-------------|---------------------|-------------------|
| Property o'qish | ~1x | ~3-5x sekinroq | ~5-10x sekinroq |
| Property yozish | ~1x | ~3-5x sekinroq | ~5-10x sekinroq |
| Funksiya chaqirish | ~1x | ~2-3x sekinroq | ~3-5x sekinroq |

**Qachon Proxy ishlatish kerak:**
- Validation — development va runtime type checking
- Reactivity — framework'lar (Vue 3)
- Logging/Debug — development va monitoring
- Access control — security-sensitive operatsiyalar
- Virtual object'lar — API wrapper, ORM

**Qachon ishlatmaslik kerak:**
- Hot loop'lar ichida — million marta chaqiriladigan kod
- Performance-critical path'lar — rendering, animation
- Oddiy getter/setter yetarli bo'lsa — `Object.defineProperty` tezroq
- Katta data set'lar ustida — har bir element uchun proxy yaratish

### Kod Misollari

```javascript
// ❌ Har bir array element uchun proxy — juda sekin
const bigArray = Array.from({ length: 100000 }, (_, i) => ({ id: i, value: i * 2 }));
const proxiedItems = bigArray.map(item => new Proxy(item, handler)); // 100K proxy!

// ✅ Faqat asosiy container uchun proxy — kerak bo'lganda nested proxy
const proxiedArray = new Proxy(bigArray, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    // faqat element access bo'lganda proxy qilish (lazy)
    if (typeof prop === "string" && !isNaN(prop) && typeof value === "object") {
      return new Proxy(value, handler);
    }
    return value;
  }
});
```

---

## Common Mistakes

### ❌ Xato 1: set trap da true qaytarmaslik

```javascript
const proxy = new Proxy({}, {
  set(target, prop, value) {
    target[prop] = value;
    // return true yo'q — undefined qaytaradi
  }
});

// Strict mode da (modules, class ichida):
proxy.name = "Ali"; // ❌ TypeError: 'set' on proxy: trap returned falsish
```

### ✅ To'g'ri usul:

```javascript
const proxy = new Proxy({}, {
  set(target, prop, value, receiver) {
    return Reflect.set(target, prop, value, receiver); // ✅ true qaytaradi
  }
});
```

**Nima uchun:** ECMAScript spec bo'yicha `[[Set]]` internal method `true` qaytarilishini kutadi. `false` yoki `undefined` (falsy) qaytarsa va strict mode bo'lsa — `TypeError`.

---

### ❌ Xato 2: Reflect ishlatmasdan target[prop] bilan ishlash

```javascript
const parent = {
  _name: "Parent",
  get fullName() {
    return this._name.toUpperCase();
  }
};

const proxy = new Proxy(parent, {
  get(target, prop) {
    return target[prop]; // ❌ getter'dagi this = target, receiver emas
  }
});

const child = Object.create(proxy);
child._name = "Child";
console.log(child.fullName); // "PARENT" ❌ — child._name emas, parent._name
```

### ✅ To'g'ri usul:

```javascript
const proxy = new Proxy(parent, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver); // ✅ this = receiver
  }
});

const child = Object.create(proxy);
child._name = "Child";
console.log(child.fullName); // "CHILD" ✅
```

**Nima uchun:** `Reflect.get` receiver'ni getter'ga `this` sifatida uzatadi. `target[prop]` bilan `this` doim `target` bo'lib qoladi.

---

### ❌ Xato 3: Proxy invariant'larini buzish

```javascript
const frozen = Object.freeze({ id: 1, name: "Ali" });

const proxy = new Proxy(frozen, {
  get(target, prop) {
    return "hacked!"; // ❌ non-configurable, non-writable property uchun
  }
});

proxy.id; // ❌ TypeError — invariant buzildi
```

### ✅ To'g'ri usul:

```javascript
const frozen = Object.freeze({ id: 1, name: "Ali" });

const proxy = new Proxy(frozen, {
  get(target, prop, receiver) {
    // non-configurable property'lar uchun original qiymatni qaytarish kerak
    const descriptor = Object.getOwnPropertyDescriptor(target, prop);
    if (descriptor && !descriptor.configurable && !descriptor.writable) {
      return Reflect.get(target, prop, receiver); // ✅ haqiqiy qiymat
    }
    // configurable property'lar uchun custom logic
    return Reflect.get(target, prop, receiver);
  }
});
```

**Nima uchun:** ECMAScript spec invariant: non-writable, non-configurable property uchun trap target'dagi qiymatdan boshqa narsa qaytara olmaydi.

---

### ❌ Xato 4: this yo'qotilishi — proxy'ni destructure qilganda

```javascript
const obj = {
  value: 42,
  getValue() { return this.value; }
};

const proxy = new Proxy(obj, {
  get(target, prop, receiver) {
    console.log(`GET: ${String(prop)}`);
    return Reflect.get(target, prop, receiver);
  }
});

// ❌ Destructure qilganda method proxy dan ajralib chiqadi
const { getValue } = proxy;
getValue(); // GET: value loglanmaydi! this = undefined (strict) yoki global
```

### ✅ To'g'ri usul:

```javascript
// ✅ Method ni proxy orqali chaqirish
proxy.getValue(); // GET: getValue, GET: value — ikkalasi ham loglanadi

// ✅ yoki bind qilish
const getValue = proxy.getValue.bind(proxy);
getValue(); // GET: value — ishlaydi
```

**Nima uchun:** Destructure qilganda method oddiy funksiya reference bo'lib qoladi — proxy ortidagi trap'lar faqat `proxy.prop` orqali access qilgandagina ishlaydi.

---

### ❌ Xato 5: Proxy ni private field (#) bilan ishlatish

```javascript
class User {
  #password;
  constructor(name, password) {
    this.name = name;
    this.#password = password;
  }
  checkPassword(input) {
    return this.#password === input; // this = proxy bo'lganda xato
  }
}

const user = new User("Ali", "secret123");
const proxy = new Proxy(user, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
  }
});

// proxy.checkPassword("secret123");
// ❌ TypeError: Cannot read private member #password from an object
//    whose class did not declare it
```

### ✅ To'g'ri usul:

```javascript
const proxy = new Proxy(user, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    // method'larni target'ga bind qilish — this = target (user) bo'ladi
    if (typeof value === "function") {
      return value.bind(target);
    }
    return value;
  }
});

proxy.checkPassword("secret123"); // ✅ true — this = target (User instance)
```

**Nima uchun:** Private field'lar (`#`) brand check qiladi — faqat class instance'ida ishlaydi. Proxy boshqa object — `#password` ga kira olmaydi. Method'ni target'ga bind qilish bu muammoni hal qiladi.

---

## Amaliy Mashqlar

### Mashq 1: createObservable (O'rta)

**Savol:** `createObservable(obj)` funksiyasi yarating. Property o'zgarganda subscriber'larga xabar bersin. `on(prop, callback)` va `on("*", callback)` qo'llab-quvvatlansin. `on` unsubscribe funksiyasi qaytarsin.

<details>
<summary>Javob</summary>

```javascript
function createObservable(target) {
  const listeners = new Map();

  const proxy = new Proxy(target, {
    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      if (oldValue !== value) {
        // prop-specific listener'lar
        const propListeners = listeners.get(prop);
        if (propListeners) {
          propListeners.forEach(cb => cb(value, oldValue, prop));
        }
        // wildcard listener'lar
        const allListeners = listeners.get("*");
        if (allListeners) {
          allListeners.forEach(cb => cb(value, oldValue, prop));
        }
      }
      return result;
    }
  });

  proxy.on = (prop, callback) => {
    if (!listeners.has(prop)) listeners.set(prop, new Set());
    listeners.get(prop).add(callback);
    // unsubscribe funksiyasi
    return () => listeners.get(prop).delete(callback);
  };

  return proxy;
}

// Test
const state = createObservable({ count: 0, name: "Ali" });
const unsub = state.on("count", (val, old) => console.log(`count: ${old} → ${val}`));
state.on("*", (val, old, prop) => console.log(`[any] ${prop}: ${old} → ${val}`));

state.count = 1;
// count: 0 → 1
// [any] count: 0 → 1

unsub(); // count listener ni olib tashlash
state.count = 2;
// [any] count: 1 → 2  (faqat wildcard ishlaydi)
```

**Tushuntirish:** `set` trap'da eski va yangi qiymat solishtiriladi. O'zgarish bo'lsa — prop-specific va wildcard listener'lar chaqiriladi. `on` metod `Set` ga callback qo'shadi va `delete` qiladigan unsubscribe funksiyasi qaytaradi.

</details>

---

### Mashq 2: createTyped — Schema-Based Type System (Qiyin)

**Savol:** `createTyped(schema)` funksiyasi yarating. Schema'da belgilangan type va validatsiya qoidalariga mos kelmaydigan qiymatlar rad etilsin. Schema'da yo'q property'lar ham rad etilsin.

<details>
<summary>Javob</summary>

```javascript
function createTyped(schema) {
  return new Proxy({}, {
    set(target, prop, value) {
      if (!(prop in schema)) {
        throw new Error(`"${prop}" schema da yo'q. Ruxsat: ${Object.keys(schema).join(", ")}`);
      }
      const rule = schema[prop];
      // rule string bo'lsa — typeof tekshirish
      if (typeof rule === "string") {
        if (typeof value !== rule) {
          throw new TypeError(`"${prop}" ${rule} bo'lishi kerak, ${typeof value} berildi`);
        }
      }
      // rule funksiya bo'lsa — custom validator
      if (typeof rule === "function") {
        if (!rule(value)) {
          throw new Error(`"${prop}" validatsiyadan o'tmadi`);
        }
      }
      // rule object bo'lsa — murakkab qoidalar
      if (typeof rule === "object" && rule !== null) {
        if (rule.type && typeof value !== rule.type) {
          throw new TypeError(`"${prop}" ${rule.type} bo'lishi kerak`);
        }
        if (rule.enum && !rule.enum.includes(value)) {
          throw new Error(`"${prop}" faqat: ${rule.enum.join(", ")}`);
        }
        if (rule.min !== undefined && value < rule.min) {
          throw new RangeError(`"${prop}" kamida ${rule.min}`);
        }
        if (rule.max !== undefined && value > rule.max) {
          throw new RangeError(`"${prop}" ko'pi bilan ${rule.max}`);
        }
      }
      return Reflect.set(target, prop, value);
    },
    get(target, prop, receiver) {
      return Reflect.get(target, prop, receiver);
    }
  });
}

// Test
const product = createTyped({
  name: "string",
  price: { type: "number", min: 0 },
  category: { type: "string", enum: ["electronics", "clothing", "food"] },
  inStock: "boolean",
  rating: (v) => typeof v === "number" && v >= 0 && v <= 5
});

product.name = "Telefon";           // ✅
product.price = 5000;               // ✅
product.category = "electronics";   // ✅
product.inStock = true;             // ✅
product.rating = 4.5;               // ✅

// product.price = -100;            // ❌ RangeError
// product.category = "toys";       // ❌ Error: faqat electronics, clothing, food
// product.rating = 6;              // ❌ Error: validatsiyadan o'tmadi
// product.color = "red";           // ❌ Error: schema da yo'q
```

</details>

---

### Mashq 3: createDeepReactive — Nested Reactivity (Qiyin)

**Savol:** `createDeepReactive(obj, onChange)` yarating. Nested object'lar ham kuzatilsin. `onChange(path, oldValue, newValue)` — to'liq path bilan callback.

<details>
<summary>Javob</summary>

```javascript
function createDeepReactive(target, onChange, path = "") {
  return new Proxy(target, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      if (typeof value === "object" && value !== null && typeof prop === "string") {
        // nested object uchun recursive proxy
        const nestedPath = path ? `${path}.${prop}` : prop;
        return createDeepReactive(value, onChange, nestedPath);
      }
      return value;
    },
    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      if (oldValue !== value) {
        const fullPath = path ? `${path}.${prop}` : String(prop);
        onChange(fullPath, oldValue, value);
      }
      return result;
    },
    deleteProperty(target, prop) {
      const oldValue = target[prop];
      const result = Reflect.deleteProperty(target, prop);
      if (result) {
        const fullPath = path ? `${path}.${prop}` : String(prop);
        onChange(fullPath, oldValue, undefined);
      }
      return result;
    }
  });
}

// Test
const state = createDeepReactive(
  {
    user: { name: "Ali", address: { city: "Toshkent" } },
    settings: { theme: "dark" }
  },
  (path, oldVal, newVal) => {
    console.log(`[${path}] ${JSON.stringify(oldVal)} → ${JSON.stringify(newVal)}`);
  }
);

state.user.name = "Vali";
// [user.name] "Ali" → "Vali"

state.user.address.city = "Samarqand";
// [user.address.city] "Toshkent" → "Samarqand"

state.settings.theme = "light";
// [settings.theme] "dark" → "light"
```

</details>

---

### Mashq 4: Revocable Access Token (Qiyin)

**Savol:** `createAccessToken(data, permissions)` yarating. `permissions` massividagi property'largagina access berilsin. `revoke()` bilan butunlay bekor qilinsin.

<details>
<summary>Javob</summary>

```javascript
function createAccessToken(data, permissions) {
  const { proxy, revoke } = Proxy.revocable(data, {
    get(target, prop, receiver) {
      if (typeof prop === "symbol" || prop === "then") {
        return Reflect.get(target, prop, receiver);
      }
      if (!permissions.includes(prop)) {
        throw new Error(`"${prop}" ga access yo'q. Ruxsat: ${permissions.join(", ")}`);
      }
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      if (!permissions.includes(prop)) {
        throw new Error(`"${prop}" ga yozish taqiqlangan`);
      }
      return Reflect.set(target, prop, value, receiver);
    },
    has(target, prop) {
      return permissions.includes(prop) && Reflect.has(target, prop);
    },
    ownKeys() {
      return permissions;
    },
    getOwnPropertyDescriptor(target, prop) {
      if (!permissions.includes(prop)) return undefined;
      return Reflect.getOwnPropertyDescriptor(target, prop) || {
        value: target[prop],
        writable: true,
        enumerable: true,
        configurable: true
      };
    }
  });

  return { token: proxy, revoke };
}

// Test
const userData = { name: "Ali", email: "ali@mail.com", password: "secret", role: "admin" };
const { token, revoke } = createAccessToken(userData, ["name", "email"]);

console.log(token.name);     // ✅ "Ali"
console.log(token.email);    // ✅ "ali@mail.com"
// token.password;            // ❌ Error: "password" ga access yo'q
// token.role;                // ❌ Error: "role" ga access yo'q
console.log(Object.keys(token)); // ["name", "email"]

revoke();
// token.name;               // ❌ TypeError: proxy revoked
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Proxy** | Object'ning 13 ta fundamental operatsiyasini ushlash va qayta belgilash. Meta-programming vositasi |
| **Handler + Traps** | get, set, has, deleteProperty, apply, construct, ownKeys, getOwnPropertyDescriptor, defineProperty, getPrototypeOf, setPrototypeOf, isExtensible, preventExtensions |
| **Reflect** | Proxy trap'lariga 1:1 mos methodlar. Boolean qaytaradi (throw o'rniga). Proxy ichida **best practice** |
| **receiver** | Getter/setter'larda to'g'ri `this`. `Reflect.get(target, prop, receiver)` — prototype chain to'g'ri ishlaydi |
| **Invariants** | Proxy trap'lar ECMAScript invariant'lariga rioya qilishi shart. Non-configurable property uchun target qiymatidan boshqa qaytarib bo'lmaydi |
| **Revocable** | `Proxy.revocable()` — vaqtinchalik access. `revoke()` dan keyin barcha operatsiyalar `TypeError` |
| **Use Cases** | Validation, Logging, Caching, Negative indexing, Reactivity (Vue 3), Data binding, Access control |
| **Performance** | Proxy oddiy object'dan 3-10x sekin. Hot path'larda ishlatmaslik. Lazy proxy qilish |

> **Keyingi bo'lim:** [24-design-patterns.md](24-design-patterns.md) — JavaScript design patterns: Factory, Singleton, Builder, Module, Decorator, Observer, Pub/Sub, Strategy, Command, Mediator, Chain of Responsibility. Har bir pattern muammo-yechim-real misol bilan.

> **Cross-references:** [06-objects.md](06-objects.md) (property descriptors, Object.defineProperty — Proxy invariant'lari bilan bog'liq), [07-prototypes.md](07-prototypes.md) (prototype chain — receiver va getPrototypeOf trap), [08-classes.md](08-classes.md) (private fields #  — Proxy bilan ishlash muammosi), [16-memory.md](16-memory.md) (WeakMap — Vue reactivity targetMap, GC)
