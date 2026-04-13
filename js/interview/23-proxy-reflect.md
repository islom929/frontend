# Proxy va Reflect — Interview Savollari

> **Bo'lim 23** | Meta-programming, Proxy traps, Reflect API, receiver, invariants, revocable proxy, reactivity, use cases

---

## Nazariy savollar

### 1. Proxy nima va u qanday ishlaydi? [Junior+]

<details>
<summary>Javob</summary>

Proxy — JavaScript'ning meta-programming vositasi bo'lib, object ustidagi fundamental operatsiyalarni (property o'qish, yozish, o'chirish, funksiya chaqirish va boshqalar) **ushlash va qayta belgilash** imkonini beradi. `new Proxy(target, handler)` bilan yaratiladi.

`target` — asl object, `handler` — trap funksiyalar to'plami. ECMAScript spec'da har bir object uchun 13 ta internal method bor (`[[Get]]`, `[[Set]]`, `[[HasProperty]]` va boshqalar). Proxy shu internal method'larning har birini custom logic bilan almashtirishga imkon beradi. Agar handler'da tegishli trap yo'q bo'lsa — operatsiya to'g'ridan-to'g'ri target'ga forward qilinadi.

```javascript
const user = { name: "Ali", age: 25 };

const proxy = new Proxy(user, {
  get(target, prop, receiver) {
    console.log(`GET: ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`SET: ${prop} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  }
});

proxy.name;      // GET: name → "Ali"
proxy.age = 30;  // SET: age = 30
console.log(user.age); // 30 — target ham o'zgardi
```

Proxy va target bitta object'ga murojaat qiladi — proxy faqat oraliq qatlam. Proxy orqali yozilgan property target'da ham o'zgaradi.


</details>

### 2. Proxy ning 13 ta trap'ini sanab bering [Middle]

<details>
<summary>Javob</summary>

| # | Trap | Ushlaydi | Trigger |
|---|------|----------|---------|
| 1 | `get` | Property o'qish | `proxy.prop`, `proxy[key]` |
| 2 | `set` | Property yozish | `proxy.prop = val` |
| 3 | `has` | `in` operatori | `"prop" in proxy` |
| 4 | `deleteProperty` | `delete` operatori | `delete proxy.prop` |
| 5 | `apply` | Funksiya chaqirish | `proxy()`, `proxy.call()` |
| 6 | `construct` | `new` operator | `new proxy()` |
| 7 | `ownKeys` | Key'larni sanash | `Object.keys()`, `for...in` |
| 8 | `getOwnPropertyDescriptor` | Descriptor olish | `Object.getOwnPropertyDescriptor()` |
| 9 | `defineProperty` | Property belgilash | `Object.defineProperty()` |
| 10 | `getPrototypeOf` | Prototype olish | `Object.getPrototypeOf()` |
| 11 | `setPrototypeOf` | Prototype o'zgartirish | `Object.setPrototypeOf()` |
| 12 | `isExtensible` | Extensibility tekshirish | `Object.isExtensible()` |
| 13 | `preventExtensions` | Extensibility taqiqlash | `Object.preventExtensions()` |

`apply` va `construct` faqat funksiya target uchun ishlaydi. Qolgan 11 ta barcha object'lar uchun.


</details>

### 3. Reflect nima va nima uchun Proxy ichida ishlatish kerak? [Middle]

<details>
<summary>Javob</summary>

Reflect — har bir Proxy trap'iga 1:1 mos keladigan static method'lar to'plami. Proxy ichida `target[prop]` o'rniga `Reflect.get(target, prop, receiver)` ishlatish **best practice** — uchta sabab:

1. **receiver** — getter/setter'larda `this` to'g'ri bo'lishi. `target[prop]` qilsangiz getter'dagi `this` = `target`. `Reflect.get` bilan `this` = `receiver` (proxy yoki proxy'dan meros olgan object).

2. **Boolean qaytarish** — `Object.defineProperty()` muvaffaqiyatsiz bo'lsa `TypeError` tashlaydi, `Reflect.defineProperty()` esa `false` qaytaradi.

3. **Nomlar mos** — trap va Reflect metod nomlari bir xil: `get` → `Reflect.get`, `set` → `Reflect.set`.

```javascript
// ❌ target[prop] — prototype chain da getter bo'lsa this noto'g'ri
const handler = {
  get(target, prop) {
    return target[prop];
  }
};

// ✅ Reflect.get — receiver to'g'ri uzatiladi
const handler = {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
  }
};
```


</details>

### 4. set trap da nima uchun true qaytarish kerak? [Middle]

<details>
<summary>Javob</summary>

ECMAScript spetsifikatsiyasi bo'yicha `[[Set]]` internal method boolean qaytaradi — `true` (muvaffaqiyatli) yoki `false` (muvaffaqiyatsiz). Proxy'ning set trap'i ham shu qoidaga amal qiladi. Agar trap `false` yoki falsy qiymat (undefined dahil) qaytarsa va strict mode bo'lsa — engine `TypeError` tashlaydi.

```javascript
// ❌ return yo'q — undefined qaytaradi (falsy)
const proxy = new Proxy({}, {
  set(target, prop, value) {
    target[prop] = value;
    // implicit return undefined
  }
});
proxy.x = 1; // Strict mode: TypeError: 'set' on proxy: trap returned falsish

// ✅ Reflect.set ishlatish — avtomatik true qaytaradi
const proxy = new Proxy({}, {
  set(target, prop, value, receiver) {
    return Reflect.set(target, prop, value, receiver);
  }
});
```

ES Modules doim strict mode'da ishlaydi — shuning uchun zamonaviy loyihalarda `return true` yo'q bo'lsa deyarli har doim xato chiqadi.


</details>

### 5. Proxy invariants nima? Misol bilan tushuntiring [Senior]

<details>
<summary>Javob</summary>

Proxy invariant'lar — ECMAScript spec belgilagan qoidalar bo'lib, Proxy trap'larining buzishi mumkin bo'lmagan cheklovlar. Bu qoidalar JavaScript'ning xavfsizlik garantiyalarini saqlaydi — `Object.freeze()` yoki `Object.seal()` kabi operatsiyalar haqiqatan ham ishlashini ta'minlaydi.

Asosiy invariant'lar:

| Trap | Invariant |
|------|-----------|
| `get` | Non-writable, non-configurable property uchun target'dagi haqiqiy qiymatdan boshqa qaytarib bo'lmaydi |
| `set` | Non-writable, non-configurable property ga `true` qaytarib bo'lmaydi |
| `has` | Non-configurable property uchun `false` qaytarib bo'lmaydi |
| `ownKeys` | Non-configurable property'lar natijada bo'lishi **shart** |
| `isExtensible` | `target` bilan bir xil natija qaytarishi **shart** |

```javascript
const obj = {};
Object.defineProperty(obj, "id", {
  value: 42,
  writable: false,
  configurable: false
});

const proxy = new Proxy(obj, {
  get(target, prop) {
    if (prop === "id") return 999; // ❌ invariant buzilishi
    return target[prop];
  }
});

proxy.id;
// TypeError: 'get' on proxy: property 'id' is a read-only and
// non-configurable data property on the proxy target but the
// proxy did not return its actual value
```

Invariant'lar engine tomonidan **automatic** tekshiriladi — developer bu tekshiruvni o'chirib qo'ya olmaydi.

**Deep Dive:** ECMAScript spec'da har bir Proxy internal method (masalan `[[Get]]`) oxirida `ValidateGetTrap` kabi validation bosqichi bor — trap natijasi va target'ning property descriptor'i solishtiriladi. Bu tekshiruv `Object.getOwnPropertyDescriptor(target, prop)` chaqiradi — shuning uchun invariant enforcement'ning o'zi ham runtime cost qo'shadi. Proxy invariant'lari JavaScript'ning "integrity level" kafolatlarini himoya qiladi — `Object.freeze` va `Object.seal` semantic'lari proxy orqali ham buzilmasligini ta'minlaydi.

</details>

### 6. Proxy.revocable() nima va qachon ishlatiladi? [Middle+]

<details>
<summary>Javob</summary>

`Proxy.revocable(target, handler)` — bekor qilish mumkin bo'lgan proxy yaratadi. `{ proxy, revoke }` object qaytaradi. `revoke()` chaqirilgandan keyin proxy'ning barcha operatsiyalari `TypeError` tashlaydi — proxy butunlay ishlamay qoladi.

```javascript
const { proxy, revoke } = Proxy.revocable(
  { secret: "qiymat" },
  {}
);

console.log(proxy.secret); // "qiymat" ✅

revoke(); // proxy bekor qilindi

// proxy.secret; // ❌ TypeError: Cannot perform 'get' on a proxy that has been revoked
```

**Qachon ishlatiladi:**
- **Vaqtinchalik access** — third-party library'ga object'ni berish, keyin access'ni olish
- **Security** — foydalanuvchi sessiyasi tugaganda data'ga kirishni to'xtatish
- **Resource management** — proxy ushlab turgan reference'larni tozalash

`revoke()` chaqirilganda proxy ichki `[[Handler]]` va `[[Target]]` ni `null` ga o'zgartiradi — boshqa qayta tiklab bo'lmaydi.


</details>

### 7. Proxy bilan private field (#) muammosi nima? Qanday hal qilasiz? [Senior]

<details>
<summary>Javob</summary>

Private field'lar (`#`) brand check qiladi — faqat o'sha class instance'ida mavjud. Proxy boshqa object — shuning uchun proxy orqali method chaqirilganda `this` = proxy bo'ladi, lekin proxy'da `#field` yo'q → `TypeError`.

```javascript
class Wallet {
  #balance;
  constructor(amount) { this.#balance = amount; }
  getBalance() { return this.#balance; } // this = proxy bo'lganda xato
}

const wallet = new Wallet(1000);
const proxy = new Proxy(wallet, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
  }
});

// proxy.getBalance();
// ❌ TypeError: Cannot read private member #balance from an object
//    whose class did not declare it
```

**Yechim:** Method'larni target'ga bind qilish:

```javascript
const proxy = new Proxy(wallet, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    if (typeof value === "function") {
      return value.bind(target); // ✅ this = target (Wallet instance)
    }
    return value;
  }
});

proxy.getBalance(); // ✅ 1000
```

`bind(target)` bilan method chaqirilganda `this` = `target` (haqiqiy class instance) bo'ladi — private field'larga kira oladi.

**Deep Dive:** Private field'lar spec'da `[[PrivateName]]` internal slot orqali ishlaydi — engine object'da alohida "brand" tekshiradi. Bu `WeakMap` semantikasiga o'xshash, lekin engine darajasida optimallashtirilgan. TC39 da Proxy + private fields muammosi atayin hal qilinmagan — committee bu cheklovni xavfsizlik uchun zarur deb hisoblaydi, chunki proxy private data'ga kirish imkonini bermasa encapsulation kuchli qoladi.

</details>

### 8. Vue 3 reactivity Proxy asosida qanday ishlaydi? [Middle+]

<details>
<summary>Javob</summary>

Vue 3 har bir reactive object'ni Proxy bilan wrap qiladi. Asosiy mexanizm — **track** (dependency kuzatish) va **trigger** (effect qayta ishga tushirish):

- `get` trap'da — o'qilgan property'ni hozirgi active effect'ga **dependency** sifatida ro'yxatga oladi (track)
- `set` trap'da — property o'zgarganda shu property'ga bog'liq barcha effect'larni qayta ishga tushiradi (trigger)

```javascript
// Soddalashtirilgan model
let activeEffect = null;
const targetMap = new WeakMap(); // target → Map(key → Set(effects))

function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      // activeEffect mavjud bo'lsa — dependency ro'yxatga olish
      if (activeEffect) {
        let depsMap = targetMap.get(target) || new Map();
        targetMap.set(target, depsMap);
        let deps = depsMap.get(key) || new Set();
        depsMap.set(key, deps);
        deps.add(activeEffect);
      }
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const old = target[key];
      const result = Reflect.set(target, key, value, receiver);
      if (old !== value) {
        // bog'liq effect'larni trigger qilish
        const deps = targetMap.get(target)?.get(key);
        if (deps) deps.forEach(fn => fn());
      }
      return result;
    }
  });
}
```

Vue 2 `Object.defineProperty` ishlatardi — dynamic property qo'shish/o'chirish kuzatilmasdi (`Vue.set()` kerak edi), array index o'zgarishi ushlanmasdi. Vue 3 Proxy bilan bu cheklovlar yo'q.


</details>

### 9. Proxy performance haqida nima bilasiz? Qachon ishlatmaslik kerak? [Senior]

<details>
<summary>Javob</summary>

Proxy har bir operatsiyada qo'shimcha function call qo'shadi. V8 engine Proxy'ni oddiy object kabi optimize qila olmaydi — inline caching va hidden class optimization ishlamaydi. Hot path'da sezilarli sekinlashuv beradi (aniq overhead V8 versiyasi va trap murakkabligiga bog'liq).

| Ishlatish kerak | Ishlatmaslik kerak |
|-----------------|-------------------|
| Validation — runtime type check | Hot loop — million marta chaqiriladigan kod |
| Reactivity — framework (Vue 3) | Performance-critical path — rendering |
| Logging/Debug — development | Katta data set — har element uchun proxy |
| Access control — security | Oddiy getter/setter yetarli bo'lganda |

**Optimization pattern** — lazy proxy (faqat access bo'lganda nested object proxy qilish):

```javascript
// ❌ Barcha nested object'larni oldindan proxy qilish
function deepProxy(obj) {
  for (const key of Object.keys(obj)) {
    if (typeof obj[key] === "object") obj[key] = deepProxy(obj[key]);
  }
  return new Proxy(obj, handler);
}

// ✅ Lazy — faqat access bo'lganda proxy qilish
const proxy = new Proxy(obj, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    if (typeof value === "object" && value !== null) {
      return new Proxy(value, handler); // faqat kirgandagina
    }
    return value;
  }
});
```

Vue 3 ham shu lazy pattern ishlatadi — `reactive()` object faqat access bo'lgan property'larni recursive proxy qiladi.

**Deep Dive:** V8 da Proxy object'lari "exotic object" sifatida treat qilinadi — TurboFan optimizing compiler Proxy property access'ni inline cache (IC) bilan optimallashtira olmaydi, chunki har bir `get`/`set` trap ixtiyoriy JS kodi. Benchmark'larda Proxy property access oddiy object'dan ~5-10x sekin. MobX 5+ va Vue 3 Proxy ishlatadi, lekin Solid.js Proxy'dan qochadi va Signal pattern bilan reaktivlik ta'minlaydi — bu performance-critical rendering'da afzallik beradi.

</details>

### 10. Reflect.get va target[prop] farqi nima? [Middle+]

<details>
<summary>Javob</summary>

Asosiy farq — **receiver** argumenti. `Reflect.get(target, prop, receiver)` uchinchi argument bilan getter'dagi `this` ni belgilaydi. `target[prop]` esa doim `target` ni `this` qiladi.

```javascript
const base = {
  _x: 10,
  get x() { return this._x; }
};

// target[prop] bilan
const proxy1 = new Proxy(base, {
  get(target, prop) { return target[prop]; }
});
const child1 = Object.create(proxy1);
child1._x = 99;
console.log(child1.x); // 10 ❌ — this = base (target)

// Reflect.get bilan
const proxy2 = new Proxy(base, {
  get(target, prop, receiver) { return Reflect.get(target, prop, receiver); }
});
const child2 = Object.create(proxy2);
child2._x = 99;
console.log(child2.x); // 99 ✅ — this = child2 (receiver)
```

Ikkinchi farq — `Reflect.set` va `Reflect.defineProperty` xato o'rniga boolean qaytaradi:

```javascript
// Object.defineProperty — try/catch kerak
try {
  Object.defineProperty(frozenObj, "x", { value: 1 });
} catch (e) { /* handle */ }

// Reflect.defineProperty — if/else yetarli
if (!Reflect.defineProperty(frozenObj, "x", { value: 1 })) {
  // handle failure
}
```



</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Output nima? Proxy get trap [Middle]


```javascript
const handler = {
  get(target, prop) {
    return prop in target ? target[prop] : `"${prop}" topilmadi`;
  }
};

const obj = new Proxy({ x: 1, y: 2 }, handler);

console.log(obj.x);
console.log(obj.y);
console.log(obj.z);
console.log("x" in obj);
```

<details>
<summary>Javob</summary>

```
1
2
"z" topilmadi
true
```

`obj.x` va `obj.y` — target'da bor, shuning uchun haqiqiy qiymat qaytariladi. `obj.z` — target'da yo'q, `prop in target` → `false`, shuning uchun fallback string qaytariladi. `"x" in obj` — `has` trap yo'q, shuning uchun standart xatti-harakat: `true`.


</details>

### 2. Output nima? receiver muammosi [Senior]


```javascript
const parent = {
  _value: 10,
  get value() { return this._value; }
};

const child = Object.create(
  new Proxy(parent, {
    get(target, prop) {
      return target[prop]; // Reflect ishlatilmagan!
    }
  })
);
child._value = 20;

console.log(child.value);
```

<details>
<summary>Javob</summary>

```
10
```

`child.value` → prototype chain bo'ylab proxy'ga boradi → `get` trap chaqiriladi → `target[prop]` = `parent.value` getter → getter'dagi `this` = `parent` (target) → `parent._value` = `10`.

Agar `Reflect.get(target, prop, receiver)` ishlatilganida — `receiver` = `child` bo'lardi → getter'dagi `this` = `child` → `child._value` = `20` qaytardi.

**Deep Dive:**

`Reflect.get` ning uchinchi argumenti `receiver` — bu spec'dagi `[[Get]](propertyKey, Receiver)` ning ikkinchi argumenti. Getter funksiya chaqirilganda `this` shu receiver ga o'rnatiladi. `target[prop]` qilganda esa receiver har doim `target` o'zi bo'ladi.


</details>

### 3. Output nima? Proxy set + strict mode [Middle]


```javascript
"use strict";

const proxy = new Proxy({}, {
  set(target, prop, value) {
    if (prop === "age" && value < 0) {
      console.log("Rad etildi");
      return false;
    }
    target[prop] = value;
    return true;
  }
});

proxy.name = "Ali";
console.log(proxy.name);

try {
  proxy.age = -5;
} catch (e) {
  console.log(e.constructor.name);
}
```

<details>
<summary>Javob</summary>

```
Ali
Rad etildi
TypeError
```

`proxy.name = "Ali"` — `set` trap `true` qaytaradi → muvaffaqiyatli. `proxy.age = -5` — `console.log("Rad etildi")` chiqadi, keyin `return false` → strict mode'da engine `TypeError` tashlaydi. `e.constructor.name` = `"TypeError"`.


</details>

### 4. Negative array indexing qanday implement qilasiz? [Middle+]

<details>
<summary>Javob</summary>

```javascript
function createNegativeArray(arr) {
  return new Proxy(arr, {
    get(target, prop, receiver) {
      const index = Number(prop);
      if (Number.isInteger(index) && index < 0) {
        // manfiy index → target.length + index
        return target[target.length + index];
      }
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      const index = Number(prop);
      if (Number.isInteger(index) && index < 0) {
        target[target.length + index] = value;
        return true;
      }
      return Reflect.set(target, prop, value, receiver);
    }
  });
}

const arr = createNegativeArray([10, 20, 30, 40, 50]);
console.log(arr[-1]);  // 50 — oxirgi element
console.log(arr[-2]);  // 40
console.log(arr[0]);   // 10 — oddiy access ishlaydi
arr[-1] = 999;
console.log(arr[4]);   // 999 — o'zgardi
console.log(arr.length); // 5 — length ishlaydi
```

`Number(prop)` bilan property nomni raqamga aylantirish kerak. `Number.isInteger` tekshiruv — faqat butun sonlar uchun (length, push kabi boshqa property'larni buzmaslik uchun). `Reflect.get` fallback — `length`, `push`, `map` va boshqa array method/property'lar oddiydek ishlaydi.


</details>

### 5. Bu kodda nima xato? Proxy + frozen object [Senior]

```javascript
const config = Object.freeze({ apiUrl: "https://api.example.com" });

const proxy = new Proxy(config, {
  get(target, prop) {
    if (prop === "apiUrl") return "https://staging.example.com";
    return target[prop];
  }
});

console.log(proxy.apiUrl); // ?
```

<details>
<summary>Javob</summary>

**Xato:** Proxy invariant buzilishi — `TypeError` tashlaydi.

`Object.freeze()` barcha property'larni `writable: false`, `configurable: false` qiladi. `get` trap non-configurable, non-writable property uchun target'dagi qiymatdan boshqa narsa qaytara olmaydi. Engine invariant tekshiradi: trap `"https://staging.example.com"` qaytarmoqchi, lekin target'da `"https://api.example.com"` — nomuvofiq → `TypeError`.

**To'g'ri usul:** Frozen object'ni proxy qilmaslik yoki faqat haqiqiy qiymatni qaytarish:

```javascript
const proxy = new Proxy(config, {
  get(target, prop, receiver) {
    console.log(`Config o'qildi: ${String(prop)}`); // ✅ log qilish mumkin
    return Reflect.get(target, prop, receiver); // ✅ haqiqiy qiymat
  }
});
```

**Deep Dive:** `Object.freeze` spec'da `[[PreventExtensions]]` + barcha own property'larni `{writable: false, configurable: false}` qiladi. Proxy `get` trap chaqirilgandan keyin engine `Object.getOwnPropertyDescriptor(target, prop)` bilan descriptor tekshiradi — agar `configurable: false` va `writable: false` bo'lsa, trap qaytargan qiymat `SameValue` bilan target'dagi qiymatga teng bo'lishi shart. Shu sababli frozen object ustida Proxy faqat side-effect (logging) uchun foydali — qiymatni o'zgartirish mumkin emas.

</details>

### 6. createObservable implement qiling [Middle+]

<details>
<summary>Javob</summary>

`createObservable(obj)` — property o'zgarganda subscriber'larga xabar bersin:

```javascript
function createObservable(target) {
  const listeners = new Map();

  const proxy = new Proxy(target, {
    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      if (oldValue !== value) {
        // prop listener'lar
        const cbs = listeners.get(prop);
        if (cbs) cbs.forEach(cb => cb(value, oldValue, prop));
        // wildcard listener'lar
        const all = listeners.get("*");
        if (all) all.forEach(cb => cb(value, oldValue, prop));
      }
      return result;
    }
  });

  proxy.on = (prop, callback) => {
    if (!listeners.has(prop)) listeners.set(prop, new Set());
    listeners.get(prop).add(callback);
    // unsubscribe funksiyasi qaytarish
    return () => listeners.get(prop).delete(callback);
  };

  return proxy;
}

const state = createObservable({ count: 0 });
const unsub = state.on("count", (val, old) => console.log(`${old} → ${val}`));
state.count = 1; // "0 → 1"
unsub();
state.count = 2; // log yo'q — unsubscribe qilingan
```

Asosiy nuqtalar:
- `Reflect.set` bilan old/new value solishtirish
- `Map<string, Set<Function>>` — property → callbacks
- `"*"` wildcard — barcha o'zgarishlar uchun
- Unsubscribe — `Set.delete` qaytaruvchi funksiya


</details>
