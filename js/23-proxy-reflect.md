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
- [Membrane Pattern — Advanced Isolation](#membrane-pattern--advanced-isolation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
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

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

---

## Proxy Yaratish

### Nazariya

`new Proxy(target, handler)` — Proxy constructor ikki argument oladi:

- **target** — proxy qiladigan asl object. Bu oddiy object, array, funksiya, boshqa proxy — har qanday object bo'lishi mumkin. Proxy bu object ga **o'rab olinadi** (wrap qilinadi).
- **handler** — trap'lar (interceptor funksiyalar) to'plami. Bu oddiy object bo'lib, ichida 13 ta mumkin bo'lgan trap metodlaridan keraklilari belgilanadi.

Proxy yaratilgandan keyin — barcha operatsiyalar proxy orqali o'tadi. Proxy'ga yozilgan property target'da ham o'zgaradi. Proxy va target bitta object'ga murojaat qiladi — proxy faqat **oraliq qatlam**.

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

---

## has, deleteProperty, ownKeys

### Nazariya

Bu trap'lar property **mavjudligini tekshirish**, **o'chirish** va **sanab chiqish** operatsiyalarini ushlaydi.

**has(target, property):**
- `in` operatori ishga tushganda chaqiriladi: `"prop" in proxy`
- `true` yoki `false` qaytarishi kerak
- **Diqqat:** `for...in` loop `has` trap'ni **chaqirmaydi** — u `ownKeys` va `getOwnPropertyDescriptor` trap'larni ishlatadi

**deleteProperty(target, property):**
- `delete proxy.prop` ishga tushganda chaqiriladi
- `true` qaytarsa — o'chirish muvaffaqiyatli (lekin aslida o'chirish developer ixtiyorida)
- `false` qaytarsa strict mode'da `TypeError`

**ownKeys(target):**
- `Object.keys()`, `Object.getOwnPropertyNames()`, `Object.getOwnPropertySymbols()`, `for...in`, `JSON.stringify()` ishga tushganda chaqiriladi
- String va Symbol'lardan iborat array qaytarishi kerak

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

`apply` trap faqat `target` callable bo'lganda ishlaydi. ECMAScript spec bo'yicha object'ning `[[Call]]` internal method'i bo'lishi kerak — bu faqat funksiyalarda bor. Oddiy object'ni proxy qilib `apply` trap qo'ysangiz — `proxy()` chaqirganda `TypeError: proxy is not a function` xatosi chiqadi, chunki target'da `[[Call]]` yo'q.

`construct` trap ham xuddi shunday — faqat `target` da `[[Construct]]` internal method bor bo'lsa ishlaydi. Arrow function'larda `[[Construct]]` yo'q — shuning uchun ularni `new` bilan chaqirib bo'lmaydi, proxy qilsangiz ham.

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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
Reflect.ownKeys(obj); // ["name", "age", "id"]

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

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

---

## Revocable Proxy

### Nazariya

`Proxy.revocable(target, handler)` — vaqtinchalik proxy yaratadi. Qaytargan object'da ikkita property bor: `proxy` (Proxy o'zi) va `revoke` (bekor qilish funksiyasi). `revoke()` chaqirilgandan keyin proxy'ning **barcha** trap'lari `TypeError` tashlaydi — proxy butunlay ishlamay qoladi. Bu mexanizm vaqtinchalik access berish, security, va resource management uchun ishlatiladi.

Revoke qilingan proxy'ni qayta tiklab bo'lmaydi — bu **bir tomonlama** operatsiya. `revoke()` chaqirilgandan keyin proxy ichki `[[Handler]]` va `[[Target]]` reference'larini `null` ga o'zgartiradi — target object'ga boshqa proxy orqali kirib bo'lmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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
│   getter/setter (init'da walk)        ├── Lazy init — kirish bo'lganda
└── Init sekin — katta object'larda     └── Init tez, per-access overhead
    O(n) getter o'rnatish                    mavjud lekin nested lazy
```

**Trade-off:** Vue 3 yondashuvi **initialization** jihatidan tezroq va xotira-samarali — Vue 2 har property uchun getter/setter o'rnatardi va butun object tree'ni walk qilardi. Vue 3 esa faqat kirilgan property'lar uchun lazy proxy yaratadi. Lekin **per-access** jihatidan Proxy Object.defineProperty'dan sekinroq (har `get`/`set` trap function call oshiradi). Asl afzallik: feature'lar (dynamic prop, array, Map/Set) va initialization performance, raw access speed emas.

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

Proxy har bir operatsiyada qo'shimcha function call qo'shadi — bu **overhead** yaratadi. V8 engine Proxy'ni oddiy object'dek optimize qila olmaydi — inline caching va hidden class optimization Proxy uchun to'liq ishlamaydi (zamonaviy V8 ba'zi Proxy pattern'lar uchun optimizatsiyalar qo'shgan, lekin oddiy property access'ga yetmaydi).

**Umumiy tendentsiya:**

| Operatsiya | Oddiy Object | Proxy (bo'sh handler) | Proxy (trap bilan) |
|-----------|-------------|---------------------|-------------------|
| Property o'qish | Eng tez (inline cache) | Sezilarli sekinroq | Yana sekinroq (trap body + engine roundtrip) |
| Property yozish | Eng tez | Sezilarli sekinroq | Yana sekinroq |
| Funksiya chaqirish | Eng tez | Kamroq overhead | Trap body'ga bog'liq |

Aniq tezlik farqi V8 versiyasi, trap complexity, va operation turiga bog'liq. Hot path'da (game loop, animation frame, katta collection iteration) farq ko'p marta bo'lishi mumkin — oddiy business logic'da esa deyarli sezilmaydi.

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Membrane Pattern — Advanced Isolation

### Nazariya

**Membrane** (membrana) — Proxy'ning eng ilg'or pattern'laridan biri. Asosiy g'oya: ikki object graph (masalan, "trusted" va "untrusted" kod) o'rtasida **to'liq chegara** yaratish va chegaradan o'tayotgan barcha object'larni avtomatik ravishda **wrap/unwrap** qilish.

Oddiy Proxy bitta object'ni wrap qiladi, lekin shu object ichida **boshqa object'lar** bo'lsa — ular wrap qilinmagan holatda tashqariga "oqib" chiqishi mumkin. Masalan: `proxy.user` — user object'i **original** (proxied emas). Agar untrusted kod `proxy.user.password = "pwned"` deb yozsa — sizning security proxy'ingiz buni ko'rmaydi, chunki `user` wrap qilinmagan.

**Membrane yechimi:** chegaradan o'tayotgan har qanday **object** (return qiymati, argument) — avtomatik ravishda shu chegaraning Proxy'siga wrap qilinadi. Va aksincha — chegaradan tashqariga qaytayotgan object unwrap qilinadi. Bu **transitive isolation** — butun object graph izolyatsiya qilinadi.

**Ishlatish joylari:**
- **Security sandboxing** — untrusted JavaScript'ni izolyatsiya qilish (masalan, plugin, user script, iframe o'rnini bosuvchi)
- **Revocable permissions** — bitta revoke butun grafga ta'sir qiladi, recursive
- **Read-only enforcement** — "frozen" membrane: barcha nested object'lar ham read-only
- **SES (Secure ECMAScript)** va **Realms API** proposal'larida membrane foydalaniladi
- **Caja** (Google), **Figma** plugin sandbox, **MetaMask Snaps** — real dunyo misollar

<details>
<summary><strong>Under the Hood</strong></summary>

Membrane'ning asosiy murakkabligi — **identity preservation**: agar siz bir xil target object'ni ikki marta membrane orqali olsangiz, ikkala marta **bir xil proxy**'ni qaytaring. Aks holda `===` taqqoslash, `WeakMap` key'lar, `Set` membership — hammasi buziladi.

**Algoritm:**
1. `wetToDry` va `dryToWet` — ikki `WeakMap`, har yo'nalish uchun cache
2. Har object chegarani kesib o'tganda — cache'dan proxy'ni olish yoki yangi yaratish
3. Return qiymati va argumentlar ham recursively wrap/unwrap
4. Primitive'lar (string, number, boolean, null, undefined) transparent o'tadi

**Soddalashtirilgan semantika:**
- "wet side" — asl object'lar (internal)
- "dry side" — wrap qilingan proxy'lar (external, ishlatuvchi tomon)
- Dry tomon'ga chiqayotgan object — `wetToDry` cache'da qidiriladi, yo'q bo'lsa yangi proxy yaratiladi
- Wet tomon'ga kirayotgan object (argument sifatida) — `dryToWet` cache'da qidiriladi, yoki unwrap qilinadi

Membrane'ning haqiqiy implementatsiyasi `WeakMap` identity, symmetric wrap/unwrap, function va prototype chain handling — juda nozik. Quyida soddalashtirilgan versiya ko'rsatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Soddalashtirilgan read-only membrane:**

```javascript
function createReadOnlyMembrane(wetObject) {
  // Identity cache — bir xil target ikki marta wrap qilinmasligi uchun
  const wetToDry = new WeakMap();

  function wrap(value) {
    // Primitive'lar transparent
    if (value === null || (typeof value !== "object" && typeof value !== "function")) {
      return value;
    }

    // Cache'dan qidirish
    if (wetToDry.has(value)) {
      return wetToDry.get(value);
    }

    // Yangi proxy yaratish
    const proxy = new Proxy(value, {
      get(target, prop, receiver) {
        const result = Reflect.get(target, prop, receiver);
        return wrap(result); // ← chegaradan chiqayotgan narsa wrap qilinadi
      },
      set() {
        throw new Error("Read-only membrane: mutation taqiqlangan");
      },
      deleteProperty() {
        throw new Error("Read-only membrane: delete taqiqlangan");
      },
      defineProperty() {
        throw new Error("Read-only membrane: defineProperty taqiqlangan");
      },
      setPrototypeOf() {
        throw new Error("Read-only membrane: setPrototypeOf taqiqlangan");
      }
    });

    wetToDry.set(value, proxy);
    return proxy;
  }

  return wrap(wetObject);
}

// Test — nested object'lar ham avtomatik himoyalangan
const internal = {
  user: {
    name: "Ali",
    address: {
      city: "Toshkent",
      zip: "100000"
    }
  },
  settings: ["dark", "uz"]
};

const readonly = createReadOnlyMembrane(internal);

// ✅ O'qish ishlaydi
console.log(readonly.user.name);           // "Ali"
console.log(readonly.user.address.city);   // "Toshkent"
console.log(readonly.settings[0]);         // "dark"

// ❌ Barcha mutation'lar bloklanadi — istalgan darajada
try { readonly.user.name = "Vali"; } catch (e) { console.log(e.message); }
// "Read-only membrane: mutation taqiqlangan"

try { readonly.user.address.city = "X"; } catch (e) { console.log(e.message); }
// "Read-only membrane: mutation taqiqlangan" — nested ham bloklangan!

try { readonly.settings.push("en"); } catch (e) { console.log(e.message); }
// "Read-only membrane: mutation taqiqlangan" — array ham bloklangan!

// Identity preservation — bir xil object ikki marta wrap qilinmaydi
const addr1 = readonly.user.address;
const addr2 = readonly.user.address;
console.log(addr1 === addr2); // true ✅ — cache tufayli bir xil proxy
```

**Revocable Membrane — butun graph uchun bir yo'la revoke:**

```javascript
function createRevocableMembrane(wetObject, handler = {}) {
  const wetToDry = new WeakMap();
  const revokers = []; // barcha revoker'larni to'plash

  function wrap(value) {
    if (value === null || (typeof value !== "object" && typeof value !== "function")) {
      return value;
    }

    if (wetToDry.has(value)) {
      return wetToDry.get(value);
    }

    const { proxy, revoke } = Proxy.revocable(value, {
      get(target, prop, receiver) {
        if (handler.get) {
          const result = handler.get(target, prop, receiver);
          return wrap(result);
        }
        return wrap(Reflect.get(target, prop, receiver));
      },
      set(target, prop, val, receiver) {
        if (handler.set) return handler.set(target, prop, val, receiver);
        return Reflect.set(target, prop, val, receiver);
      },
      apply(target, thisArg, args) {
        // Argument'lar ichkariga kirayotganda — unwrap (soddalashtirilgan)
        const result = Reflect.apply(target, thisArg, args);
        return wrap(result); // result tashqariga chiqyapti — wrap
      }
    });

    wetToDry.set(value, proxy);
    revokers.push(revoke);
    return proxy;
  }

  const membraned = wrap(wetObject);

  return {
    proxy: membraned,
    revoke() {
      // Barcha proxy'larni bir yo'la revoke qilish
      revokers.forEach(r => r());
      revokers.length = 0;
    }
  };
}

// Use case: third-party plugin'ga access berish
function loadUntrustedPlugin(pluginFn, appData) {
  const { proxy: safeData, revoke } = createRevocableMembrane(appData, {
    // Faqat ruxsat etilgan field'larga access
    get(target, prop) {
      const allowed = ["publicConfig", "userName", "theme"];
      if (!allowed.includes(prop)) {
        throw new Error(`Access denied to "${prop}"`);
      }
      return target[prop];
    }
  });

  try {
    pluginFn(safeData); // plugin membraned data bilan ishlaydi
  } finally {
    // Plugin tugadi — butun access bir yo'la revoke qilinadi
    revoke();
    // Endi plugin saqlab qo'ygan har qanday reference (nested ham!) TypeError beradi
  }
}

// Untrusted plugin kod
function maliciousPlugin(appData) {
  console.log(appData.userName); // ✅ ruxsat etilgan

  // Plugin nested object'ga reference saqlashga harakat qiladi
  const stolenRef = appData.publicConfig;

  setTimeout(() => {
    // 1 sekund keyin access qilishga harakat — REVOKED
    try {
      console.log(stolenRef.secretKey); // ❌ TypeError: revoked
    } catch (e) {
      console.log("Plugin access denied after revoke:", e.message);
    }
  }, 1000);

  // try { appData.password; } catch (e) { ... } // ❌ Access denied
}

loadUntrustedPlugin(maliciousPlugin, {
  publicConfig: { theme: "dark" },
  userName: "Ali",
  password: "secret"
});
```

**Membrane semantikasining muhim jihatlari:**

1. **Identity preservation** — `WeakMap` orqali bir xil target bir xil proxy'ni qaytaradi, `===` ishlaydi
2. **Transitive wrapping** — har qaytarilgan object/function avtomatik wrap qilinadi, chuqurlik cheklanmagan
3. **Symmetric wrap/unwrap** — membrane ikki tomonlama: out'ga chiqayotgan narsa wrap, in'ga kirayotgan argument unwrap (to'liq implementatsiyada)
4. **Revocation cascading** — bitta `revoke()` butun object graph'ni o'lik qiladi, nested reference'lar ham yaramaydi
5. **Primitive'lar transparent** — string, number, boolean membrane'dan bemalol o'tadi (ular immutable va reference'siz)

**Haqiqiy production implementatsiyalar:**
- **ShadowRealm API** (Stage 3 TC39 proposal) — Realms va membrane pattern standart'ga kiritilmoqda
- **SES (Secure ECMAScript)** — Agoric/MetaMask tomonidan ishlab chiqilgan secure JavaScript runtime
- **MetaMask Snaps** — extension plugin'lar uchun isolated execution environment
- **Figma plugins** — plugin'lar iframe + membrane bilan izolyatsiyalangan

Oddiy loyihalarda membrane kerak emas — bu **high-security** yoki **plugin system** quruvchilar uchun. Lekin pattern'ni tushunish Proxy'ning haqiqiy kuchini ko'rsatadi: oddiy proxy bitta object'ni wrap qiladi, membrane esa **butun object universe**'ni izolyatsiya qiladi.

</details>

---

## Edge Cases va Gotchas

Proxy bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha.

### Gotcha 1: `proxy === target` har doim `false` — identity equality

Proxy va target **ikki alohida reference**, garchi ular bir xil xulq-atvor ko'rsatsa ham. Bu `===`, `WeakMap`/`WeakSet` membership, `Set.has`, `Map` keys — hammasiga ta'sir qiladi. Object identity Proxy orqali "transparent" emas.

```javascript
const target = { name: "Ali" };
const proxy = new Proxy(target, {});

// ❌ Identity equality — har doim false
console.log(proxy === target); // false
console.log(target === proxy); // false

// ❌ WeakMap — proxy va target ikki alohida key
const wm = new WeakMap();
wm.set(target, "original");
console.log(wm.get(proxy)); // undefined — proxy boshqa key!
wm.set(proxy, "proxied");
console.log(wm.size); // konceptual: 2 ta yozuv

// ❌ Set membership
const seen = new Set();
seen.add(target);
console.log(seen.has(proxy)); // false

// ❌ Array.includes (reference search)
const arr = [target];
console.log(arr.includes(proxy)); // false
console.log(arr.indexOf(proxy));  // -1

// ─── Real-world implication: cache/dedupe ───
// Cache key sifatida ishlatishda target yoki proxy ni consistent tanlash kerak
const cache = new WeakMap();

function getOrCompute(obj, computeFn) {
  if (cache.has(obj)) return cache.get(obj);
  const result = computeFn(obj);
  cache.set(obj, result);
  return result;
}

// ❌ Noto'g'ri pattern: gohi proxy, gohi target
const result1 = getOrCompute(target, o => o.name.length);
const result2 = getOrCompute(proxy, o => o.name.length);
// Ikki marta compute bo'ladi — cache miss!

// ✅ To'g'ri: consistent reference ishlatish
// Yoki hamma joyda proxy, yoki hamma joyda target

// ─── Helper: "unwrap" pattern ───
// Agar bo'lsa ham proxy'ni original target'ga map qilish kerak bo'lsa
const targetRegistry = new WeakMap(); // proxy → target

function makeProxy(target, handler) {
  const proxy = new Proxy(target, handler);
  targetRegistry.set(proxy, target);
  return proxy;
}

function unwrap(obj) {
  return targetRegistry.get(obj) ?? obj; // proxy bo'lsa target, bo'lmasa o'zi
}

const p = makeProxy(target, {});
console.log(unwrap(p) === target); // true ✅ — explicit unwrap
```

**Nima uchun:** JavaScript'da object identity `===` native reference comparison — ikki o'zgaruvchi bir xil heap address'ga ishora qiladimi. Proxy spec'da Proxy **alohida exotic object** sifatida yaratiladi — u target'dan farqli object hisoblanadi, garchi internal method'lari target'ga forward qilsa ham. Bu design qarori — aks holda Proxy introspection (masalan, "bu proxy'mi?" degan check) imkonsiz bo'lardi.

**Yechim:** Agar kodingizda cache, dedup, identity-based logic bo'lsa — **consistent reference** ishlating (hamma joyda proxy yoki hamma joyda target). Original'ni olish kerak bo'lsa — alohida `WeakMap` registry orqali explicit unwrap pattern. "Proxy bo'lsa target'ga tarjima qil" uchun library (Vue 3 `toRaw()`) ishlating.

### Gotcha 2: `typeof proxy` target turiga bog'liq — function target "function" qaytaradi

`typeof` operator Proxy'ni **target turiga qarab** baholaydi: object target → `"object"`, function target → `"function"`. Proxy'ning mavjudligi `typeof` natijasini o'zgartirmaydi. Bu "is this a proxy?" detection'ni qiyinlashtiradi.

```javascript
// Object target
const objProxy = new Proxy({}, {});
console.log(typeof objProxy); // "object"

// Function target
function greet() { return "Hello"; }
const fnProxy = new Proxy(greet, {});
console.log(typeof fnProxy); // "function" ✅

// ⚠️ Oddiy object'ni function'ga "o'xshatib" bo'lmaydi
const fakeCallable = new Proxy({}, {
  apply() { return "called"; }
});
console.log(typeof fakeCallable); // "object" — target object, "function" bo'lmaydi
// fakeCallable(); // ❌ TypeError: fakeCallable is not a function
// apply trap faqat callable target bilan ishlaydi

// ✅ Callable proxy faqat function target bilan
const realCallable = new Proxy(function() {}, {
  apply(target, thisArg, args) {
    return `called with ${args}`;
  }
});
console.log(typeof realCallable); // "function" ✅
console.log(realCallable(1, 2));  // "called with 1,2"

// ─── "Bu proxy'mi?" detection — JavaScript'da mumkin emas ───
// util.types.isProxy(value) faqat Node.js internal API, browser'da yo'q
// typeof, instanceof, Object.prototype.toString — hech biri Proxy'ni aniqlay olmaydi!

function isProxy(obj) {
  // Bu FUNDAMENTAL'dan mumkin emas — Proxy har qanday operatsiyani intercept qila oladi
  // Shu jumladan detection urinishlarini ham
  return false; // ❓
}

// ─── Faqat internal tracking bilan ───
const knownProxies = new WeakSet();

function trackedProxy(target, handler) {
  const proxy = new Proxy(target, handler);
  knownProxies.add(proxy);
  return proxy;
}

const p = trackedProxy({}, {});
console.log(knownProxies.has(p)); // true ✅ — o'zimiz track qilganlar
```

**Nima uchun:** `typeof` spec bo'yicha object'ning `[[Call]]` internal method'i borligini tekshiradi — bor bo'lsa "function", yo'q bo'lsa "object" (yoki primitive type). Proxy spec'da Proxy o'zining `[[Call]]`'ini **faqat target'da `[[Call]]` bor bo'lsa** meros qiladi. Shuning uchun object target bilan proxy — callable emas, function target bilan proxy — callable.

**Yechim:** "Callable proxy" kerak bo'lsa — target sifatida function ishlating (hatto bo'sh function `() => {}` bo'lsa ham). Proxy detection kerak bo'lsa — tashqaridan `WeakSet` registry orqali. Proxy'ni tashqaridan "aniqlab bo'lmaydi" — bu uning asosiy feature'i, xavfsizlik uchun muhim (untrusted kod proxy'ni detect qila olmasligi kerak).

### Gotcha 3: `JSON.stringify(proxy)` har property uchun `get` trap chaqiradi — side effect'lar trigger bo'ladi

`JSON.stringify` object'ning har property'sini iteratsiya qiladi va har biri uchun `get` trap chaqirilishi kerak. Shu jumladan u **`toJSON`** property'sini ham tekshiradi. Side-effect'li `get` trap'da (masalan, reactive dependency tracking yoki logging) bu kutilmagan ko'p trigger'ga olib keladi.

```javascript
let getCount = 0;
const target = { name: "Ali", age: 25, city: "Toshkent" };

const proxy = new Proxy(target, {
  get(target, prop, receiver) {
    getCount++;
    console.log(`GET: ${String(prop)}`);
    return Reflect.get(target, prop, receiver);
  }
});

// JSON.stringify — har property uchun get trap + toJSON check
const json = JSON.stringify(proxy);
console.log(getCount);
// Output:
// GET: toJSON          ← JSON.stringify avval toJSON mavjudligini tekshiradi
// GET: name
// GET: age
// GET: city
// getCount: 4

// ⚠️ Reactive system'da — har stringify butun object'ni "access qilgan" deb sanaladi
// Dependency tracking bu stringify call'ni "user shu property'larni o'qidi" deb bilib qoladi
// Natijada shu component keraksiz re-render bo'lishi mumkin

// ─── Real Vue 3 muammosi: reactive + JSON.stringify ───
// const state = reactive({ user: { name: "Ali" }, count: 0 });
// const json = JSON.stringify(state); // barcha nested property'lar "track" bo'ladi
// Effect endi BARCHA property o'zgarishida re-run bo'ladi — kutilmagan

// ✅ Yechim 1: toRaw() — reactive'dan original'ga (Vue 3 pattern)
function toRaw(proxy) {
  return targetMap.get(proxy) ?? proxy;
}
// JSON.stringify(toRaw(state)) — trap chaqirilmaydi

// ✅ Yechim 2: Stringify oldidan dependency tracking'ni pauza qilish
let trackingEnabled = true;
const proxyWithPause = new Proxy(target, {
  get(target, prop, receiver) {
    if (trackingEnabled) {
      trackDependency(target, prop);
    }
    return Reflect.get(target, prop, receiver);
  }
});

// Serialization paytida tracking'ni o'chirish
function safeStringify(obj) {
  trackingEnabled = false;
  try {
    return JSON.stringify(obj);
  } finally {
    trackingEnabled = true;
  }
}

// ✅ Yechim 3: Custom toJSON
const proxyWithToJSON = new Proxy(target, {
  get(target, prop, receiver) {
    if (prop === "toJSON") {
      // Custom serialization — side-effect'lar yo'q
      return () => ({ ...target });
    }
    trackDependency(target, prop);
    return Reflect.get(target, prop, receiver);
  }
});
```

**Nima uchun:** `JSON.stringify` spec'da quyidagicha ishlaydi: (1) `toJSON` method mavjudligini tekshirish (Date kabi built-in object'lar uchun), (2) agar bor bo'lsa chaqirish va natijani serialize qilish, (3) aks holda enumerable own property'larni iteratsiya qilish. Har qadam `[[Get]]` internal method chaqiradi — Proxy'da bu `get` trap'ga map bo'ladi. Har trap call engine'ning side-effect orientirovkasi bo'lmagan — spec to'liq transparent forwarding'ni taxmin qiladi.

**Yechim:** Reactive/tracking'li proxy'larda `JSON.stringify` dan oldin **tracking'ni vaqtincha o'chirish**, yoki `toRaw()` bilan original'ni olish, yoki `toJSON` trap'da side-effect'siz path yaratish. Vue 3 `toRaw()` aynan shu muammoni hal qiladi.

### Gotcha 4: Accidental Thenable — `Promise.resolve(proxy)` `then` trap'ga yopishib qoladi

JavaScript Promise'lar **duck typing** orqali "thenable" aniqlaydi — object'da `then` property'si funksiya bo'lsa, u thenable hisoblanadi. Agar proxy'ning `get` trap'i **har qanday property** uchun function qaytarsa (masalan, logger proxy) — Promise uni thenable deb oladi va infinite loop yoki xato beradi.

```javascript
// ❌ Logger proxy — har property uchun function qaytaradi
const logger = new Proxy({}, {
  get(target, prop) {
    return (...args) => {
      console.log(`[${String(prop)}]`, ...args);
    };
  }
});

logger.info("test");  // [info] test ✅
logger.error("oops"); // [error] oops ✅

// ⚠️ Lekin Promise bilan ishlatishga urinish:
async function run() {
  const result = await logger; // ⚠️ await logger.then chaqiradi!
  // Proxy get('then') → function(resolve, reject) qaytaradi
  // Promise machinery bu function'ni chaqiradi: then(resolve, reject)
  // Logger esa "[then]" log qiladi va undefined qaytaradi
  // → await natijasi undefined
  console.log("Result:", result); // undefined
}
run();

// ─── Ko'proq xavfli holat — infinite loop ───
const trickier = new Proxy({}, {
  get(target, prop) {
    // Har property uchun yangi thenable qaytaradi
    return {
      then(resolve) {
        resolve(trickier); // ← yana proxy'ni resolve qiladi!
      }
    };
  }
});

// await trickier — trickier.then(resolve) chaqiradi
// resolve(trickier) — yana Promise trickier.then'ni chaqiradi
// Infinite recursion yoki hang

// ✅ Yechim 1: then property'ni explicit handle qilish
const safeLogger = new Proxy({}, {
  get(target, prop) {
    // Symbol va "then"/"catch"/"finally" uchun undefined qaytarish
    if (prop === "then" || prop === "catch" || prop === "finally") {
      return undefined;
    }
    if (typeof prop === "symbol") {
      return undefined; // Symbol.toPrimitive, Symbol.iterator va h.k.
    }
    return (...args) => console.log(`[${prop}]`, ...args);
  }
});

async function runSafe() {
  const result = await safeLogger; // ✅ then === undefined → not thenable
  console.log(result); // safeLogger o'zi (Proxy object)
}

// ✅ Yechim 2: Promise.resolve bilan explicit wrap
const wrappedLogger = Promise.resolve({ logger: safeLogger });
// Endi logger Promise ichida, thenable resolution yo'q
const { logger: lg } = await wrappedLogger;
lg.info("test"); // ✅

// ─── Real-world: API client proxy ───
// ❌ Har method call uchun proxy — await bilan muammo
const api = new Proxy({}, {
  get(target, endpoint) {
    return (...args) => fetch(`/api/${endpoint}`, ...args);
  }
});

// await api; // ⚠️ api.then() chaqiriladi → fetch("/api/then", resolve, reject)
// HTTP request "/api/then" ga jo'natiladi — bug!
```

**Nima uchun:** ECMAScript Promise spec "thenable" ni duck typing bilan aniqlaydi: `typeof obj.then === "function"`. `await` va `Promise.resolve()` bu check'ni o'tkazadi. Proxy'ning `get` trap'i har qanday property'ga javob beradi — shu jumladan `then`'ga ham. Agar trap function qaytarsa, engine proxy'ni thenable deb qabul qiladi va `then(resolve, reject)` chaqiradi. Bu "structural subtyping" ning chegarasi — JavaScript runtime explicit "this is a thenable" deklaratsiyasini talab qilmaydi.

**Yechim:** Proxy yaratganda `get` trap'da `then` (va boshqa Promise method'lari `catch`, `finally`) uchun **`undefined` qaytarish**. Shuningdek `Symbol` property'lar uchun ham (masalan `Symbol.toPrimitive` — `String(proxy)` chaqirilganda muammo bo'ladi). "Virtual API" pattern'larida bu **majburiy defensive check**.

### Gotcha 5: Chained proxies — execution order **outer → inner**

`new Proxy(new Proxy(target, inner), outer)` — ikki proxy zanjiri. Property access'da **outer handler birinchi** chaqiriladi, inner keyinroq. Stack trace chuqur va confusing — debugging juda qiyin. Middleware pattern yaratayotganda bu order muhim.

```javascript
const target = { value: 42 };

const inner = new Proxy(target, {
  get(target, prop, receiver) {
    console.log(`[INNER] get ${String(prop)}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`[INNER] set ${String(prop)} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  }
});

const outer = new Proxy(inner, {
  get(target, prop, receiver) {
    console.log(`[OUTER] get ${String(prop)}`);
    return Reflect.get(target, prop, receiver);
    // ← Reflect.get(inner, ...) — inner'ning get trap'ini trigger qiladi
  },
  set(target, prop, value, receiver) {
    console.log(`[OUTER] set ${String(prop)} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  }
});

outer.value;
// Output order:
// [OUTER] get value   ← outer birinchi
// [INNER] get value   ← outer forwarding qilgach, inner
// Natija: 42

outer.value = 100;
// [OUTER] set value = 100
// [INNER] set value = 100
// target.value = 100

// ─── Middleware pattern — order matter ───
// Loglash, caching, validation middleware'larni chain qilish
function loggingMiddleware(target) {
  return new Proxy(target, {
    get(t, p, r) {
      console.log(`[log] GET ${String(p)}`);
      return Reflect.get(t, p, r);
    }
  });
}

function cachingMiddleware(target) {
  const cache = new Map();
  return new Proxy(target, {
    get(t, p, r) {
      if (cache.has(p)) {
        console.log(`[cache] HIT ${String(p)}`);
        return cache.get(p);
      }
      console.log(`[cache] MISS ${String(p)}`);
      const value = Reflect.get(t, p, r);
      cache.set(p, value);
      return value;
    }
  });
}

// Order matters!
const data = { result: "expensive computation" };

// Order 1: log → cache
const withCache1 = cachingMiddleware(loggingMiddleware(data));
withCache1.result;
// [cache] MISS result    ← outer (cache) birinchi
// [log] GET result       ← inner (log) keyin — har miss'da log
withCache1.result;
// [cache] HIT result    ← cache hit — log ishlamaydi (short-circuit)

// Order 2: cache → log
const withCache2 = loggingMiddleware(cachingMiddleware(data));
withCache2.result;
// [log] GET result       ← outer (log) birinchi — har chaqiruvda
// [cache] MISS result    ← inner (cache)
withCache2.result;
// [log] GET result       ← yana log (har chaqiruvda)
// [cache] HIT result     ← cache hit

// Kutilgan behavior'ga qarab order tanlanadi:
// - "Har chaqiruvda log" → log OUTER
// - "Faqat cache miss'da log" → log INNER
```

**Nima uchun:** JavaScript engine proxy operatsiyasini xuddi oddiy property access kabi evaluate qiladi — tashqi object'dan ichki object'ga. `outer.prop` → `outer` ning `[[Get]]` chaqiriladi → bu `outer.handler.get(inner, prop, outer)` — inner proxy target sifatida beriladi. Handler ichidagi `Reflect.get(inner, prop)` → `inner.handler.get(target, prop, inner)`. Ya'ni chain outer-to-inner execution order'ida.

**Yechim:** Middleware chain'larni tuzayotganda execution order'ni **aniq hujjatlashtiring**. Trap ichida `Reflect.*` orqali "next" middleware'ga forward qilish — Express-style middleware pattern'iga o'xshash. Debugging uchun har middleware o'z label'iga ega bo'lishi kerak (console.log prefix). Test case'lar order-sensitive bo'lganda yaxshi test'lar yozing.

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
