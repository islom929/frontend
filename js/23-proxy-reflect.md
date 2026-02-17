# Bo'lim 23: Proxy va Reflect

> Proxy va Reflect — JavaScript ning meta-programming vositalari. Object xulq-atvorini ushlash, o'zgartirish va kuzatish imkonini beradi.

---

## Mundarija

- [Proxy Asoslari](#proxy-asoslari)
- [Proxy Traps](#proxy-traps)
- [Reflect API](#reflect-api)
- [Amaliy Use Cases](#amaliy-use-cases)
- [Vue.js Reactivity Qanday Ishlaydi](#vuejs-reactivity)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Proxy Asoslari

### Nazariya

`Proxy` — JavaScript ning meta-programming vositasi bo'lib, object ga "o'rash" (wrapper) qo'yib, uning asosiy operatsiyalarini (o'qish, yozish, o'chirish, funksiya chaqirish, `in` tekshirish va boshqalar) **ushlash va qayta belgilash** imkonini beradi. `new Proxy(target, handler)` bilan yaratiladi: `target` — asl object, `handler` — trap'lar (13 ta operatsiyani ushlash mumkin). Proxy orqali validatsiya, logging, caching, computed property'lar, va reactivity tizimlarini (Vue 3) qurish mumkin.

### Sintaksis

```javascript
const proxy = new Proxy(target, handler);
// target — asl object
// handler — trap'lar (operatsiyalarni ushlash uchun)
```

### Kod Misollari

```javascript
const user = { name: "Ali", age: 25 };

const proxy = new Proxy(user, {
  // get trap — property o'qish ushlanadi
  get(target, property, receiver) {
    console.log(`"${property}" o'qilmoqda`);
    return target[property];
  },

  // set trap — property yozish ushlanadi
  set(target, property, value, receiver) {
    console.log(`"${property}" = ${value} yozilmoqda`);
    target[property] = value;
    return true; // true qaytarish kerak (strict mode uchun)
  }
});

proxy.name;       // "name" o'qilmoqda → "Ali"
proxy.age = 26;   // "age" = 26 yozilmoqda
proxy.role = "admin"; // "role" = admin yozilmoqda

console.log(user.age);  // 26 — asl object HAM o'zgardi
console.log(user.role); // "admin"
```

### Under the Hood

```
               Proxy
  Kod ──────▶ ┌──────────┐ ──────▶ Target Object
              │ Handler  │
  proxy.name  │ get()    │ ──▶ target.name
  proxy.x = 1 │ set()    │ ──▶ target.x = 1
  delete proxy │ delete() │ ──▶ delete target
  proxy()     │ apply()  │ ──▶ target()
              └──────────┘
```

---

## Proxy Traps

### Barcha Trap'lar

| Trap | Operatsiya | Qachon ishga tushadi |
|------|-----------|---------------------|
| `get` | Property o'qish | `proxy.prop`, `proxy[key]` |
| `set` | Property yozish | `proxy.prop = val` |
| `has` | `in` operatori | `"prop" in proxy` |
| `deleteProperty` | `delete` | `delete proxy.prop` |
| `apply` | Funksiya chaqirish | `proxy()`, `proxy.call()` |
| `construct` | `new` operator | `new proxy()` |
| `getPrototypeOf` | Prototype olish | `Object.getPrototypeOf(proxy)` |
| `setPrototypeOf` | Prototype o'zgartirish | `Object.setPrototypeOf(proxy, proto)` |
| `ownKeys` | Kalitlarni sanash | `Object.keys(proxy)`, `for...in` |
| `getOwnPropertyDescriptor` | Descriptor olish | `Object.getOwnPropertyDescriptor()` |
| `defineProperty` | Property belgilash | `Object.defineProperty()` |
| `isExtensible` | Kengaytirish tekshirish | `Object.isExtensible()` |
| `preventExtensions` | Kengaytirishni to'xtatish | `Object.preventExtensions()` |

### get va set Trap'lar (Batafsil)

```javascript
const handler = {
  // get(target, property, receiver)
  get(target, prop, receiver) {
    if (prop in target) {
      return target[prop];
    }
    throw new ReferenceError(`"${prop}" property mavjud emas`);
  },

  // set(target, property, value, receiver)
  set(target, prop, value, receiver) {
    if (prop === "age" && (typeof value !== "number" || value < 0)) {
      throw new TypeError("age musbat raqam bo'lishi kerak");
    }
    target[prop] = value;
    return true;
  }
};

const user = new Proxy({}, handler);
user.name = "Ali";     // ✅
user.age = 25;         // ✅
// user.age = -5;      // ❌ TypeError: age musbat raqam bo'lishi kerak
// user.age = "yigirma"; // ❌ TypeError
// console.log(user.missing); // ❌ ReferenceError
```

### has, deleteProperty, ownKeys

```javascript
const privateFields = new Proxy({ _secret: "123", name: "Ali", age: 25 }, {
  // has — "in" operatorida private field'larni yashirish
  has(target, prop) {
    if (prop.startsWith("_")) return false;
    return prop in target;
  },

  // ownKeys — Object.keys() da private field'larni yashirish
  ownKeys(target) {
    return Object.keys(target).filter(key => !key.startsWith("_"));
  },

  // get — private field o'qishni taqiqlash
  get(target, prop) {
    if (prop.startsWith("_")) return undefined;
    return target[prop];
  },

  // deleteProperty — private field o'chirishni taqiqlash
  deleteProperty(target, prop) {
    if (prop.startsWith("_")) {
      throw new Error("Private field o'chirilmaydi");
    }
    delete target[prop];
    return true;
  }
});

console.log("name" in privateFields);   // true
console.log("_secret" in privateFields); // false — yashirilgan!
console.log(Object.keys(privateFields)); // ["name", "age"] — _secret yo'q
console.log(privateFields._secret);      // undefined
```

### apply va construct

```javascript
// === apply — funksiya chaqirishni ushlash ===
function sum(a, b) { return a + b; }

const loggedSum = new Proxy(sum, {
  apply(target, thisArg, args) {
    console.log(`sum(${args.join(", ")}) chaqirildi`);
    const result = target.apply(thisArg, args);
    console.log(`Natija: ${result}`);
    return result;
  }
});
loggedSum(3, 5); // "sum(3, 5) chaqirildi" → "Natija: 8" → 8

// === construct — new bilan yaratishni ushlash ===
class User {
  constructor(name) { this.name = name; }
}

const TrackedUser = new Proxy(User, {
  construct(target, args, newTarget) {
    console.log(`Yangi User: ${args[0]}`);
    return new target(...args);
  }
});
const u = new TrackedUser("Ali"); // "Yangi User: Ali"
```

---

## Reflect API

### Nazariya

`Reflect` — object operatsiyalarini **standart va xavfsiz** usulda bajarish uchun API. Har bir Proxy trap'iga mos Reflect metodi bor. Proxy trap'lari ichida `Reflect` ishlatish — **best practice**, chunki `target[prop]` bilan ishlash prototype chain, getter/setter va `receiver` (to'g'ri `this` konteksti) bilan bog'liq edge case'larda xato berishi mumkin. Reflect metodlari boolean qaytaradi (muvaffaqiyat/muvaffaqiyatsizlik), `Object` metodlari esa xato tashlaydi.

### Nima Uchun Reflect?

```javascript
// ❌ target [][] bilan ishlash — ba'zi edge case larda xato
const handler = {
  get(target, prop) {
    return target[prop]; // ❌ prototype chain ni to'g'ri handle qilmasligi mumkin
  }
};

// ✅ Reflect ishlatish — barcha edge case lar to'g'ri ishlaydi
const handler = {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
    // receiver — to'g'ri this kontekstini saqlaydi
  }
};
```

### Reflect Metodlari

```javascript
const obj = { name: "Ali", age: 25 };

// get / set
Reflect.get(obj, "name");           // "Ali"
Reflect.set(obj, "age", 30);        // true — muvaffaqiyatli
Reflect.set(obj, "role", "admin");   // true

// has (in operator)
Reflect.has(obj, "name");           // true
Reflect.has(obj, "missing");        // false

// deleteProperty
Reflect.deleteProperty(obj, "age"); // true

// ownKeys
Reflect.ownKeys(obj); // ["name", "role"]
// Object.keys dan farqi: Symbol va non-enumerable ham beradi

// defineProperty — boolean qaytaradi (throw qilmaydi)
const success = Reflect.defineProperty(obj, "id", {
  value: 1,
  writable: false
});
// Object.defineProperty xato beradi, Reflect false qaytaradi

// apply
Reflect.apply(Math.max, null, [1, 2, 3]); // 3
```

### Reflect + Proxy Best Practice

```javascript
// ✅ IDEAL — Proxy ichida Reflect ishlatish
function createReactiveObject(target, onChange) {
  return new Proxy(target, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      // Nested object lar uchun recursive proxy
      if (typeof value === "object" && value !== null) {
        return createReactiveObject(value, onChange);
      }
      return value;
    },
    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      if (oldValue !== value) {
        onChange(prop, oldValue, value);
      }
      return result;
    },
    deleteProperty(target, prop) {
      const result = Reflect.deleteProperty(target, prop);
      if (result) onChange(prop, target[prop], undefined);
      return result;
    }
  });
}

const state = createReactiveObject({ count: 0, user: { name: "Ali" } }, 
  (prop, oldVal, newVal) => {
    console.log(`${prop}: ${oldVal} → ${newVal}`);
  }
);

state.count = 5;          // "count: 0 → 5"
state.user.name = "Vali"; // "name: Ali → Vali"
```

---

## Amaliy Use Cases

### 1. Validation Proxy

```javascript
function createValidator(rules) {
  return {
    set(target, prop, value) {
      const rule = rules[prop];
      if (rule) {
        const { type, min, max, required, pattern } = rule;
        if (required && (value === null || value === undefined || value === "")) {
          throw new Error(`${prop} majburiy`);
        }
        if (type && typeof value !== type) {
          throw new TypeError(`${prop} ${type} bo'lishi kerak, ${typeof value} berildi`);
        }
        if (min !== undefined && value < min) {
          throw new RangeError(`${prop} kamida ${min} bo'lishi kerak`);
        }
        if (max !== undefined && value > max) {
          throw new RangeError(`${prop} ko'pi bilan ${max} bo'lishi kerak`);
        }
        if (pattern && !pattern.test(String(value))) {
          throw new Error(`${prop} noto'g'ri formatda`);
        }
      }
      return Reflect.set(target, prop, value);
    }
  };
}

const user = new Proxy({}, createValidator({
  name: { type: "string", required: true },
  age: { type: "number", min: 0, max: 150 },
  email: { type: "string", pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ }
}));

user.name = "Ali";           // ✅
user.age = 25;               // ✅
// user.age = -5;            // ❌ RangeError
// user.email = "invalid";   // ❌ Error: noto'g'ri format
```

### 2. Logging / Debug Proxy

```javascript
function createLogger(target, label = "Object") {
  return new Proxy(target, {
    get(target, prop, receiver) {
      console.log(`[${label}] GET ${String(prop)}`);
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      console.log(`[${label}] SET ${String(prop)} = ${JSON.stringify(value)}`);
      return Reflect.set(target, prop, value, receiver);
    },
    deleteProperty(target, prop) {
      console.log(`[${label}] DELETE ${String(prop)}`);
      return Reflect.deleteProperty(target, prop);
    }
  });
}

const debugUser = createLogger({ name: "Ali" }, "User");
debugUser.name;        // [User] GET name
debugUser.age = 25;    // [User] SET age = 25
delete debugUser.age;  // [User] DELETE age
```

### 3. Caching / Memoization Proxy

```javascript
function memoize(fn) {
  const cache = new Map();
  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);
      if (cache.has(key)) {
        console.log("Cache hit:", key);
        return cache.get(key);
      }
      const result = Reflect.apply(target, thisArg, args);
      cache.set(key, result);
      return result;
    }
  });
}

const expensiveCalc = memoize((n) => {
  console.log("Hisoblash...");
  return n * n;
});

expensiveCalc(5);  // "Hisoblash..." → 25
expensiveCalc(5);  // "Cache hit: [5]" → 25 (qayta hisoblanmaydi)
```

### 4. Negative Array Index

```javascript
function createNegativeArray(arr) {
  return new Proxy(arr, {
    get(target, prop, receiver) {
      const index = Number(prop);
      if (!isNaN(index) && index < 0) {
        return target[target.length + index];
      }
      return Reflect.get(target, prop, receiver);
    }
  });
}

const arr = createNegativeArray([10, 20, 30, 40, 50]);
console.log(arr[-1]); // 50 — Python kabi!
console.log(arr[-2]); // 40
console.log(arr[0]);  // 10 — oddiy access ishlaydi
```

---

## Vue.js Reactivity

### Nazariya

Vue 3 reactivity tizimi `Proxy` ga asoslangan. Property o'qilganda **track** (dependency sifatida kuzatishga olish), yozilganda **trigger** (bog'liq effect'larni qayta ishga tushirish) qilinadi. Bu Vue 2 ning `Object.defineProperty` asosidagi tizimidan ustun: dynamic property qo'shish/o'chirish, array index o'zgarishi, Map/Set qo'llab-quvvatlanadi, va tezroq ishlaydi.

### Soddalashtirilgan Vue Reactivity

```javascript
// Vue 3 reactivity tizimining soddalashtirilgan versiyasi
let activeEffect = null;
const targetMap = new WeakMap(); // target → Map(key → Set(effects))

// Track — property o'qilganda
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

// Trigger — property yozilganda
function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const effects = depsMap.get(key);
  if (effects) {
    effects.forEach(effect => effect());
  }
}

// reactive — Proxy bilan kuzatiladigan object
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key);
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver);
      trigger(target, key);
      return result;
    }
  });
}

// effect — reaktiv ta'sirni ro'yxatga olish
function effect(fn) {
  activeEffect = fn;
  fn(); // birinchi marta ishlash — get trap'lar orqali track qilish
  activeEffect = null;
}

// === Ishlatish ===
const state = reactive({ count: 0, message: "Salom" });

// effect — count o'zgarganda avtomatik ishlaydi
effect(() => {
  console.log(`Count: ${state.count}`);
});
// "Count: 0" — darhol ishladi

state.count = 1;  // "Count: 1" — avtomatik!
state.count = 5;  // "Count: 5" — avtomatik!
state.message = "Xayr"; // hech narsa — bu effect count ni kuzatadi
```

### Vue 2 vs Vue 3

```
Vue 2 (Object.defineProperty):
├── Barcha property lar oldidan aniqlangan bo'lishi kerak
├── Array index o'zgarishini ushlolmaydi
├── Property qo'shish/o'chirish kuzatilmaydi → Vue.set() kerak
└── Sekinroq — har bir property ga alohida getter/setter

Vue 3 (Proxy):
├── Dynamic property qo'shish/o'chirish kuzatiladi ✅
├── Array o'zgarishlarini to'liq ushlaydi ✅
├── Map, Set, WeakMap bilan ishlaydi ✅
└── Tezroq va soddaroq ✅
```

---

## Common Mistakes

### ❌ Xato 1: set trap da true qaytarmaslik

```javascript
// ❌ Strict mode da TypeError
const proxy = new Proxy({}, {
  set(target, prop, value) {
    target[prop] = value;
    // return true yo'q!
  }
});
proxy.name = "Ali"; // ❌ TypeError (strict mode)
```

### ✅ To'g'ri usul:

```javascript
// ✅ return true yoki Reflect.set
const proxy = new Proxy({}, {
  set(target, prop, value, receiver) {
    return Reflect.set(target, prop, value, receiver); // ✅ true qaytaradi
  }
});
```

**Nima uchun:** ECMAScript spec bo'yicha set trap `true` qaytarishi kerak. `false` yoki `undefined` qaytarsa strict mode da `TypeError` tashlaydi.

---

### ❌ Xato 2: Reflect ishlatmasdan target bilan ishlash

```javascript
// ❌ Prototype chain muammolari
const parent = { greeting: "Salom" };
const child = Object.create(parent);

const proxy = new Proxy(child, {
  get(target, prop) {
    return target[prop]; // ❌ agar child da getter bo'lsa — this noto'g'ri
  }
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ Reflect.get receiver bilan
const proxy = new Proxy(child, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver); // ✅ to'g'ri this
  }
});
```

**Nima uchun:** `Reflect.get` uchinchi argumenti `receiver` — getter funksiyalarda `this` to'g'ri bo'lishini ta'minlaydi. `target[prop]` bilan bu guarantee yo'q.

---

## Amaliy Mashqlar

### Mashq 1: createObservable (O'rta)

**Savol:** `createObservable(obj)` funksiyasi yarating. Property o'zgarganda callback chaqirilsin.

<details>
<summary>Javob</summary>

```javascript
function createObservable(target) {
  const listeners = new Map();

  const proxy = new Proxy(target, {
    set(target, prop, value, receiver) {
      const old = target[prop];
      const result = Reflect.set(target, prop, value, receiver);
      if (old !== value) {
        const cbs = listeners.get(prop) || [];
        cbs.forEach(cb => cb(value, old, prop));
        const allCbs = listeners.get("*") || [];
        allCbs.forEach(cb => cb(value, old, prop));
      }
      return result;
    }
  });

  proxy.on = (prop, callback) => {
    if (!listeners.has(prop)) listeners.set(prop, []);
    listeners.get(prop).push(callback);
    return () => {
      const cbs = listeners.get(prop);
      const i = cbs.indexOf(callback);
      if (i > -1) cbs.splice(i, 1);
    };
  };

  return proxy;
}

const state = createObservable({ count: 0, name: "Ali" });
const unsub = state.on("count", (val, old) => console.log(`count: ${old} → ${val}`));
state.on("*", (val, old, prop) => console.log(`[any] ${prop} o'zgardi`));

state.count = 1; // "count: 0 → 1" + "[any] count o'zgardi"
unsub();
state.count = 2; // faqat "[any] count o'zgardi"
```

</details>

---

### Mashq 2: Type-Safe Object (Qiyin)

**Savol:** `createTyped(schema)` funksiyasi yarating. Object property'lari schema'ga mos kelishi kerak.

<details>
<summary>Javob</summary>

```javascript
function createTyped(schema) {
  return new Proxy({}, {
    set(target, prop, value) {
      if (!(prop in schema)) {
        throw new Error(`"${prop}" schemada yo'q. Faqat: ${Object.keys(schema).join(", ")}`);
      }
      const rule = schema[prop];
      if (typeof rule === "string" && typeof value !== rule) {
        throw new TypeError(`"${prop}" ${rule} bo'lishi kerak, ${typeof value} berildi`);
      }
      if (typeof rule === "function" && !rule(value)) {
        throw new Error(`"${prop}" validatsiyadan o'tmadi`);
      }
      return Reflect.set(target, prop, value);
    },
    get(target, prop) {
      if (prop in schema && !(prop in target)) {
        return undefined;
      }
      return Reflect.get(target, prop);
    }
  });
}

const user = createTyped({
  name: "string",
  age: "number",
  email: (v) => typeof v === "string" && v.includes("@")
});

user.name = "Ali";           // ✅
user.age = 25;               // ✅
user.email = "ali@mail.com"; // ✅
// user.name = 123;          // ❌ TypeError
// user.role = "admin";      // ❌ Error: schemada yo'q
// user.email = "invalid";   // ❌ validatsiyadan o'tmadi
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Proxy** | Object operatsiyalarini ushlash va qayta belgilash. 13 ta trap mavjud |
| **Reflect** | Object operatsiyalarini standart va xavfsiz bajarish. Proxy ichida ishlatish — best practice |
| **Validation** | set trap bilan runtime type checking va validatsiya |
| **Logging** | get/set trap bilan debug va monitoring |
| **Reactivity** | Vue 3 Proxy asosida — track (get) + trigger (set) pattern |
| **Vue 2 vs 3** | defineProperty → Proxy. Dynamic prop, array, Map/Set qo'llab-quvvatlash |
| **Invariants** | Proxy trap'lar ECMAScript invariant'lariga rioya qilishi kerak |

> **Keyingi bo'lim:** [24-design-patterns.md](24-design-patterns.md) — Factory, Singleton, Observer, Module, Strategy, Pub/Sub pattern'lar.

> **Cross-references:** [06-objects.md](06-objects.md) (property descriptors, Object.defineProperty), [07-prototypes.md](07-prototypes.md) (prototype chain — Reflect va receiver), [08-classes.md](08-classes.md) (class, private fields — Proxy bilan privacy), [16-memory.md](16-memory.md) (WeakMap — targetMap, GC)
