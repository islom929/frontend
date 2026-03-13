# Objects — Interview Savollari

> Object — key-value juftliklardan iborat ma'lumot tuzilmasi. Bu bo'limda property descriptors, immutability, shallow/deep copy, enumeration va zamonaviy object metodlari haqida savollar.

---

## Savol 1: Object yaratishning qanday usullari bor? [Junior+]

**Javob:**

JavaScript da object yaratishning to'rtta asosiy usuli:

```javascript
// 1. Object Literal — eng ko'p ishlatiladigan
const user1 = { name: "Alice", age: 25 };

// 2. Constructor Function — new bilan
function User(name) { this.name = name; }
const user2 = new User("Alice");

// 3. Object.create() — berilgan prototype bilan
const proto = { greet() { return "Hi"; } };
const user3 = Object.create(proto);

// 4. Class (ES6) — constructor function'ning syntactic sugar'i
class UserClass {
  constructor(name) { this.name = name; }
}
const user4 = new UserClass("Alice");
```

| Usul | Prototype | Use case |
|------|-----------|----------|
| Literal `{}` | `Object.prototype` | Oddiy object'lar |
| Constructor | `Constructor.prototype` | ES5 da OOP |
| `Object.create(proto)` | berilgan proto | Custom prototype chain |
| Class | `Class.prototype` | Zamonaviy OOP |

---

## Savol 2: Property descriptor nima? [Middle]

**Javob:**

Har bir object property'si meta-ma'lumotlarga ega — bu **property descriptor**. Uch xil flag bor:

- **`writable`** — qiymat o'zgartirilishi mumkinmi
- **`enumerable`** — `for...in`, `Object.keys()` da ko'rinadimi
- **`configurable`** — descriptor qayta sozlanishi yoki property o'chirilishi mumkinmi

```javascript
const obj = { name: "Alice" };

// Default descriptor (literal bilan yaratilganda):
console.log(Object.getOwnPropertyDescriptor(obj, "name"));
// { value: "Alice", writable: true, enumerable: true, configurable: true }

// Custom descriptor:
Object.defineProperty(obj, "id", {
  value: 1,
  writable: false,     // o'zgarmas
  enumerable: false,   // ko'rinmas
  configurable: false  // qaytib bo'lmas
});

obj.id = 999;          // ❌ silent fail (strict: TypeError)
console.log(Object.keys(obj)); // ["name"] — id ko'rinmaydi
delete obj.id;         // ❌ o'chirilmaydi
```

`Object.defineProperty` bilan yaratilgan property'larda default qiymatlar **`false`** — oddiy assign bilan yaratilganda esa **`true`**.

---

## Savol 3: `Object.freeze`, `Object.seal`, `Object.preventExtensions` farqi nima? [Middle]

**Javob:**

| | Yangi property | O'zgartirish | O'chirish |
|---|---|---|---|
| `preventExtensions` | ❌ | ✅ | ✅ |
| `seal` | ❌ | ✅ | ❌ |
| `freeze` | ❌ | ❌ | ❌ |

```javascript
// freeze — eng kuchli
const obj = Object.freeze({ a: 1, nested: { b: 2 } });
obj.a = 99;           // ❌ o'zgarmaydi
obj.nested.b = 99;    // ✅ ISHLAYDI — shallow freeze!
```

Barchasi **shallow** — nested object'larga ta'sir qilmaydi. Deep freeze uchun recursive funksiya kerak:

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
```

---

## Savol 4: Shallow copy va deep copy farqi nima? Qanday usullar bor? [Middle]

**Javob:**

**Shallow copy** — birinchi daraja copy, nested object'lar reference bo'lib qoladi:
```javascript
const a = { x: 1, inner: { y: 2 } };
const b = { ...a }; // shallow
b.inner.y = 99;
console.log(a.inner.y); // 99 — ❌ original buzildi
```

**Deep copy** — barcha darajalar to'liq copy:
```javascript
const c = structuredClone(a); // deep
c.inner.y = 99;
console.log(a.inner.y); // 2 — ✅ original saqlanadi
```

| Usul | Tur | Circular ref | Date/Map/Set | Function |
|------|-----|-------------|-------------|----------|
| `{ ...obj }` / `Object.assign` | Shallow | ❌ | ❌ ref | ❌ ref |
| `JSON.parse(JSON.stringify())` | Deep | ❌ Error | ❌ yo'qoladi | ❌ yo'qoladi |
| `structuredClone()` (ES2022) | Deep | ✅ | ✅ | ❌ Error |
| Recursive function | Deep | ✅ (WeakMap) | ✅ | ⚠️ ref |

Zamonaviy kod'da `structuredClone` eng yaxshi tanlov — function copy kerak bo'lmasa.

---

## Savol 5: Quyidagi kodning output'i nima? [Middle+]

```javascript
const a = { x: 1, y: { z: 2 } };
const b = Object.assign({}, a);
const c = { ...a };

b.x = 10;
c.y.z = 30;

console.log(a.x);   // ?
console.log(a.y.z);  // ?
console.log(b.y.z);  // ?
```

**Javob:**

```
1
30
30
```

- `b.x = 10` → `a.x` o'zgarmaydi (birinchi daraja copy)
- `c.y.z = 30` → `a.y.z` **HAM** o'zgaradi (shallow — `y` reference bo'lib qolgan)
- `b.y.z = 30` → b ham c ham a bilan **bir xil** `y` object'ga reference saqlaydi
- Barchasi bitta `y` object: `a.y === b.y === c.y`

---

## Savol 6: `for...in` vs `Object.keys()` vs `Reflect.ownKeys()` farqi nima? [Middle]

**Javob:**

| Metod | Own | Inherited | Non-enumerable | Symbol |
|-------|-----|-----------|---------------|--------|
| `for...in` | ✅ | ✅ | ❌ | ❌ |
| `Object.keys()` | ✅ | ❌ | ❌ | ❌ |
| `Reflect.ownKeys()` | ✅ | ❌ | ✅ | ✅ |

```javascript
const parent = { inherited: true };
const obj = Object.create(parent);
obj.visible = 1;
Object.defineProperty(obj, "hidden", { value: 2, enumerable: false });
obj[Symbol("id")] = 3;

// for...in: "visible", "inherited" (inherited ham!)
// Object.keys: ["visible"]
// Reflect.ownKeys: ["visible", "hidden", Symbol(id)]
```

Zamonaviy kod'da `for...in` o'rniga `Object.keys()` / `Object.entries()` ishlatish tavsiya etiladi — inherited property'lardan kutilmagan xulq bo'lmaydi.

---

## Savol 7: `Object.hasOwn()` nima va `hasOwnProperty` dan farqi? [Junior+]

**Javob:**

`Object.hasOwn()` (ES2022) — property object'ning **o'ziga** tegishli ekanini tekshiradi. `hasOwnProperty` ning xavfsiz versiyasi:

```javascript
// hasOwnProperty muammolari:
const obj = Object.create(null); // prototype yo'q
// obj.hasOwnProperty("key"); // ❌ TypeError — metod mavjud emas

const tricky = { hasOwnProperty: () => false }; // override
tricky.hasOwnProperty("hasOwnProperty"); // false — noto'g'ri!

// Object.hasOwn — har doim ishlaydi:
Object.hasOwn(obj, "key");    // ✅ ishlaydi — statik metod
Object.hasOwn(tricky, "hasOwnProperty"); // true — to'g'ri
```

Zamonaviy kod'da doim `Object.hasOwn()` ishlatish kerak.

---

## Savol 8: Getter va Setter nima? Qachon ishlatiladi? [Middle]

**Javob:**

Getter/Setter — property o'qilganda/yozilganda avtomatik chaqiriladigan funksiyalar. Tashqaridan oddiy property, ichida logika:

```javascript
const user = {
  _age: 25,

  get age() {
    return this._age;
  },

  set age(value) {
    if (value < 0 || value > 150) throw new Error("Invalid age");
    this._age = value;
  }
};

console.log(user.age);  // 25 — getter chaqirildi (funksiya kabi emas!)
user.age = 30;           // ✅ setter — validation o'tdi
// user.age = -5;        // ❌ Error: Invalid age
```

Use cases:
1. **Validation** — qiymat yozilganda tekshirish
2. **Computed properties** — `fullName` = `firstName` + `lastName`
3. **Lazy initialization** — birinchi o'qilganda hisoblash, keyingilarida cache
4. **Backward compatibility** — eski property nomini saqlab, ichida yangi logika

---

## Savol 9: `deepClone` funksiyasini implement qiling [Senior]

**Javob:**

```javascript
function deepClone(obj, seen = new WeakMap()) {
  // Primitive va null
  if (obj === null || typeof obj !== "object") return obj;

  // Circular reference himoya
  if (seen.has(obj)) return seen.get(obj);

  // Special types
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

  // Array va Object
  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone);

  for (const key of Reflect.ownKeys(obj)) {
    clone[key] = deepClone(obj[key], seen);
  }
  return clone;
}

// Test:
const original = {
  date: new Date(),
  nested: { a: [1, 2, { b: 3 }] },
  map: new Map([["x", 1]]),
  regex: /test/gi
};
original.circular = original; // circular reference

const cloned = deepClone(original);
console.log(cloned.nested.a[2].b); // 3
console.log(cloned.date instanceof Date); // true
console.log(cloned.circular === cloned);  // true (circular resolved)
console.log(cloned !== original); // true (alohida object)
```

`WeakMap` circular reference'ni handle qiladi: object birinchi ko'rilganda `seen` ga yoziladi, qayta uchrasa — saqlangan clone qaytariladi.

---

## Savol 10: `deepEqual` funksiyasini implement qiling [Senior]

**Javob:**

```javascript
function deepEqual(a, b) {
  if (a === b) return true;
  if (a === null || b === null) return false;
  if (typeof a !== "object" || typeof b !== "object") return false;
  if (a.constructor !== b.constructor) return false;

  if (Array.isArray(a)) {
    if (a.length !== b.length) return false;
    return a.every((item, i) => deepEqual(item, b[i]));
  }

  if (a instanceof Date) return a.getTime() === b.getTime();
  if (a instanceof RegExp) return a.source === b.source && a.flags === b.flags;

  if (a instanceof Map) {
    if (a.size !== b.size) return false;
    for (const [key, val] of a) {
      if (!b.has(key) || !deepEqual(val, b.get(key))) return false;
    }
    return true;
  }

  if (a instanceof Set) {
    if (a.size !== b.size) return false;
    for (const val of a) {
      if (!b.has(val)) return false;
    }
    return true;
  }

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key =>
    Object.hasOwn(b, key) && deepEqual(a[key], b[key])
  );
}

// Test:
deepEqual({ a: [1, { b: 2 }] }, { a: [1, { b: 2 }] }); // true
deepEqual({ a: 1 }, { a: 2 });                           // false
deepEqual(new Date(0), new Date(0));                      // true
deepEqual([1, 2], [1, 2, 3]);                             // false
```

---

## Savol 11: `Object.groupBy()` nima va qanday ishlaydi? [Junior+]

**Javob:**

`Object.groupBy()` (ES2024) — array elementlarini callback natijasi bo'yicha guruhlaydi:

```javascript
const users = [
  { name: "Alice", role: "admin" },
  { name: "Bob", role: "user" },
  { name: "Charlie", role: "admin" },
  { name: "Diana", role: "user" }
];

const byRole = Object.groupBy(users, user => user.role);
// {
//   admin: [{ name: "Alice", role: "admin" }, { name: "Charlie", role: "admin" }],
//   user: [{ name: "Bob", role: "user" }, { name: "Diana", role: "user" }]
// }
```

ES2024 dan oldin bu `reduce` bilan qo'lda qilinardi:

```javascript
const byRole = users.reduce((groups, user) => {
  const key = user.role;
  (groups[key] ??= []).push(user);
  return groups;
}, {});
```

`Map.groupBy()` ham bor — key sifatida non-string qiymat kerak bo'lsa (masalan object).

---

## Savol 12: `structuredClone` nima va JSON hack dan farqi? [Middle]

**Javob:**

`structuredClone()` (ES2022) — browser'ning Structured Clone Algorithm'i asosida deep copy yaratadi:

| Xususiyat | `JSON.parse(JSON.stringify())` | `structuredClone()` |
|---|---|---|
| Date | ❌ → string | ✅ Date saqlanadi |
| Map/Set | ❌ → {} | ✅ saqlanadi |
| RegExp | ❌ → {} | ✅ saqlanadi |
| undefined | ❌ yo'qoladi | ✅ saqlanadi |
| NaN/Infinity | ❌ → null | ✅ saqlanadi |
| Circular ref | ❌ TypeError | ✅ ishlaydi |
| Function | ❌ yo'qoladi | ❌ DataCloneError |
| Symbol keys | ❌ yo'qoladi | ❌ copy qilmaydi |
| DOM Node | ❌ Error | ❌ Error |

```javascript
const obj = {
  date: new Date(),
  set: new Set([1, 2, 3]),
  undef: undefined,
  nan: NaN
};
obj.self = obj; // circular

const clone = structuredClone(obj);
console.log(clone.date instanceof Date); // true
console.log(clone.set instanceof Set);   // true
console.log(clone.self === clone);       // true — circular resolved
```

---

## Savol 13: Quyidagi kodning output'i nima? [Middle+]

```javascript
const obj = { a: 1 };

Object.defineProperty(obj, "b", {
  value: 2,
  enumerable: false,
  writable: true
});

obj[Symbol("c")] = 3;

console.log(Object.keys(obj));              // ?
console.log(Object.getOwnPropertyNames(obj)); // ?
console.log(Reflect.ownKeys(obj));           // ?
console.log(JSON.stringify(obj));            // ?
```

**Javob:**

```javascript
Object.keys(obj);                // ["a"]
// ✅ Faqat own + enumerable string keys

Object.getOwnPropertyNames(obj); // ["a", "b"]
// ✅ Own string keys (enumerable + non-enumerable), Symbol yo'q

Reflect.ownKeys(obj);            // ["a", "b", Symbol(c)]
// ✅ BARCHA own keys: string + Symbol, enumerable + non-enumerable

JSON.stringify(obj);             // '{"a":1}'
// ✅ Faqat enumerable string keys, Symbol keys va non-enumerable yo'q
```

---

## Savol 14: Optional chaining (`?.`) qanday ishlaydi va qachon ishlatiladi? [Junior+]

**Javob:**

Optional chaining — chuqur nested property'ga xavfsiz murojaat. Agar zanjirdagi biror qism `null` yoki `undefined` bo'lsa — `TypeError` o'rniga `undefined` qaytaradi:

```javascript
const user = {
  profile: {
    address: { city: "NYC" }
  }
};

// Optional chaining yo'q — manual tekshirish:
const city1 = user && user.profile && user.profile.address && user.profile.address.city;

// Optional chaining bilan:
const city2 = user?.profile?.address?.city;       // "NYC"
const zip = user?.profile?.address?.zip;          // undefined (xato emas)
const phone = user?.contact?.phone;               // undefined (xato emas)

// Method chaqirish:
user?.profile?.toString?.();  // "[object Object]"
user?.missing?.method?.();    // undefined

// Index access:
const first = user?.items?.[0]; // undefined
```

Qachon ishlatish kerak: tashqi API'dan kelgan data, optional config, nullable type'lar bilan ishlashda.

Qachon ishlatMASLIK kerak: doim mavjud bo'lishi kerak bo'lgan property'lar uchun — `?.` xatoni yashiradi, debugging qiyinlashadi.

---

## Savol 15: V8 Hidden Class nima va performance'ga qanday ta'sir qiladi? [Senior]

**Javob:**

V8 har bir object uchun **Hidden Class** (ichki nomi "Map") yaratadi — bu object'ning "shakli" (shape). Bir xil tartibda bir xil property'lar qo'shilgan object'lar bitta Hidden Class'ni share qiladi.

Hidden Class'ning maqsadi — **inline caching**. Agar V8 bilsa ki ikkita object bir xil shape'da — birinchi object'da property access optimizatsiya qilinsa, ikkinchisida ham shu optimizatsiya ishlaydi.

```javascript
// ✅ Yaxshi — bir xil tartibda property qo'shish
function createPoint(x, y) {
  const p = {};
  p.x = x;  // Hidden Class: C0 → C1 (x qo'shildi)
  p.y = y;  // Hidden Class: C1 → C2 (y qo'shildi)
  return p;
}
const p1 = createPoint(1, 2); // Shape: C2
const p2 = createPoint(3, 4); // Shape: C2 — SHARE ✅

// ❌ Yomon — turli tartibda property qo'shish
const a = {}; a.x = 1; a.y = 2; // Shape: A
const b = {}; b.y = 2; b.x = 1; // Shape: B — BOSHQA shape!
// V8 inline caching foyda bermaydi
```

Performance qoidalari:
1. Object'larga property'larni **bir xil tartibda** qo'shing
2. Object yaratgandan keyin yangi property **qo'shMANG** (constructor'da hammasi bo'lsin)
3. Property **o'chirMANG** (`delete`) — shape buziladi
4. Property type'ini **o'zgartirMANG** (number → string) — deoptimization

---
