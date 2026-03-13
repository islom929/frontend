# Memory Management — Interview Savollari

> Stack vs Heap, Garbage Collection, memory leaks, WeakRef, WeakMap/WeakSet, performance optimization haqida interview savollari.

---

## Savol 1: Stack va Heap farqi nima? Qaysi ma'lumotlar qayerda saqlanadi? [Junior+]

**Javob:**

JavaScript engine xotirani ikki hududda boshqaradi:

| | Stack | Heap |
|---|-------|------|
| **Nima saqlanadi** | Primitives, pointers, call frames | Objects, Arrays, Functions |
| **Hajm** | Kichik, fixed (~1MB) | Katta, dynamic (GB gacha) |
| **Tezlik** | Juda tez (LIFO — pointer siljitish) | Sekinroq (allokatsiya + GC kerak) |
| **Boshqaruv** | Avtomatik (scope/funksiya tugasa tozalanadi) | Garbage Collector boshqaradi |
| **Tartib** | LIFO (Last In, First Out) | Tartibsiz |
| **Xato** | Stack Overflow (chuqur recursion) | Out of Memory (ko'p object) |

```javascript
let name = "Ali";         // "Ali" → STACK (primitive — value)
let age = 25;              // 25 → STACK (primitive — Smi)

let user = {               // pointer → STACK, object → HEAP
  name: "Ali",
  age: 25
};

let copy = user;           // ❌ FAQAT pointer nusxasi (HEAP dagi object BIR XIL)
copy.name = "Vali";
console.log(user.name);   // "Vali" — bitta object!
```

```
STACK                         HEAP
┌────────────────┐           ┌──────────────────┐
│ name: "Ali"    │           │                  │
│ age: 25        │           │  { name: "Ali",  │
│ user: ─────────────────→   │    age: 25 }     │
│ copy: ─────────────────→   │                  │
└────────────────┘           └──────────────────┘
```

V8 da heap bir nechta bo'limga bo'lingan: New Space (yangi object'lar — Scavenger GC), Old Space (yashagan object'lar — Mark-Compact GC), Large Object Space, Code Space (JIT compiled), Map Space (Hidden Classes).

---

## Savol 2: Copy by value va copy by reference farqi nima? [Junior+]

**Javob:**

Primitive turlari (number, string, boolean, undefined, null, symbol, bigint) **copy by value** — o'zgaruvchiga tayinlanganda qiymatning mustaqil nusxasi yaratiladi, birini o'zgartirish ikkinchisiga ta'sir qilmaydi.

Reference turlari (Object, Array, Function, Map, Set, ...) **copy by reference** — faqat heap'dagi object'ga pointer nusxalanadi, ikki o'zgaruvchi bitta object'ga ishora qiladi.

```javascript
// Primitive — copy by value:
let a = 42;
let b = a;    // mustaqil nusxa
b = 100;
console.log(a); // 42 — o'zgarmadi ✅

// Reference — copy by reference:
let arr1 = [1, 2, 3];
let arr2 = arr1;    // pointer nusxasi — bitta array!
arr2.push(4);
console.log(arr1);  // [1, 2, 3, 4] — o'zgardi!
console.log(arr1 === arr2); // true — bitta reference

// ✅ Mustaqil nusxa: spread yoki slice
let arr3 = [...arr1]; // shallow copy — yangi array
arr3.push(5);
console.log(arr1);  // [1, 2, 3, 4] — o'zgarmadi ✅
```

Bu farq React/Redux da muhim: state'ni mutate qilsangiz reference bir xil qoladi → `===` true → re-render bo'lmaydi. Immutable update (spread, map, filter) yangi reference yaratadi.

---

## Savol 3: Quyidagi kodning output'ini ayting [Middle]

```javascript
function modify(obj, num) {
  obj.x = 10;
  num = 10;
}

let myObj = { x: 1 };
let myNum = 1;

modify(myObj, myNum);

console.log(myObj.x); // ?
console.log(myNum);   // ?
```

**Javob:**

```
10
1
```

`myObj` — reference type. Funksiyaga pointer nusxasi beriladi, `obj.x = 10` orqali **original** object o'zgaradi. `myNum` — primitive. Funksiyaga **qiymat nusxasi** beriladi, `num = 10` faqat local nusxani o'zgartiradi, original `myNum` o'zgarmaydi.

---

## Savol 4: Garbage Collection qanday ishlaydi? Mark-and-Sweep nima? [Middle]

**Javob:**

JavaScript avtomatik memory management ishlatadi — Garbage Collector keraksiz object'larni topib tozalaydi.

Zamonaviy standart — **Mark-and-Sweep** algoritmi:
1. **Root** lardan boshlash: global object, call stack variables, active closures
2. **Mark**: root'dan reachable bo'lgan barcha object'larni belgilash (BFS/DFS)
3. **Sweep**: belgilanmagan (unreachable) object'larni tozalash

```
ROOT → A ✓ → B ✓ → C ✓
       ↓
       D ✓
              E ✗ ↔ F ✗   ← root dan yetib bo'lmaydi → TOZALANADI
```

Bu algoritm **circular reference** muammosini hal qiladi — ikkita object bir-biriga reference qilsa ham, root'dan reachable bo'lmasa tozalanadi. Eski Reference Counting algoritmi buni hal qila olmasdi.

V8 **Generational GC** ishlatadi:
- **Young Generation** (Scavenger, ~1-5ms) — yangi object'lar, tez tozalaydi
- **Old Generation** (Mark-Compact, ~50-200ms) — yashagan object'lar, chuqur tozalaydi

**Orinoco** tizimi — incremental, concurrent, parallel GC ni birlashtiradi, UI ~60fps saqlanadi.

---

## Savol 5: Memory leak nima? Eng keng tarqalgan turlari qaysilar? [Middle]

**Javob:**

Memory leak — dastur ishlayotganida xotira to'planib ketishi, lekin GC uni tozalay olmasligi. Sababi: object'ga biz unutgan/tashlab qo'ygan reference mavjud — GC uchun "kerakli".

**6 ta asosiy tur:**

| Leak Turi | Sababi | Yechimi |
|-----------|--------|---------|
| Global variables | `"use strict"` yo'q, var/let/const siz | `"use strict"`, doim `let`/`const` |
| Forgotten timers | `clearInterval`/`clearTimeout` qilmaslik | Timer ID saqlash, cleanup qilish |
| DOM references | Element o'chirildi, JS da reference qoldi | `ref = null` |
| Closures | Closure katta data ushlab turadi | Minimal data capture |
| Event listeners | `removeEventListener` qilmaslik | `AbortController` yoki explicit remove |
| Detached DOM trees | Butun sub-tree JS da reference bilan qoladi | `ref = null`, event delegation |

```javascript
// ❌ Forgotten timer — eng ko'p uchraydi SPA da:
class Component {
  mount() {
    this.timer = setInterval(() => this.update(), 1000);
  }
  // unmount() yo'q → timer va component xotirada qoladi!
}

// ✅ Cleanup:
class ComponentFixed {
  mount() { this.timer = setInterval(() => this.update(), 1000); }
  unmount() { clearInterval(this.timer); this.timer = null; }
}
```

---

## Savol 6: Quyidagi kodning output'ini ayting [Middle+]

```javascript
let a = { value: 1 };
let b = a;
let c = { value: 1 };

console.log(a === b);  // ?
console.log(a === c);  // ?

b.value = 2;
console.log(a.value);  // ?

b = { value: 3 };
console.log(a.value);  // ?
```

**Javob:**

```
true
false
2
2
```

- `a === b` → `true` — ikkalasi **bitta** object'ga ishora qiladi (pointer copy)
- `a === c` → `false` — tarkibi bir xil, lekin **turli** object'lar (heap'da turli joyda)
- `b.value = 2` → `a.value` ham 2 bo'ladi — bitta object (b pointer orqali mutate)
- `b = { value: 3 }` → b yangi object'ga ishora qildi, lekin `a` hali eski object'da → `a.value` hali 2

---

## Savol 7: WeakMap va Map farqi nima? Qachon WeakMap ishlatish kerak? [Middle+]

**Javob:**

| Xususiyat | `Map` | `WeakMap` |
|-----------|-------|-----------|
| Key turi | Har qanday (primitive + object) | Faqat **object** |
| GC | Key ni strong ushlab turadi | Key weak — GC tozalashi mumkin |
| Iterable | Ha (`for...of`, `keys()`, `values()`) | **Yo'q** |
| `size` | Ha | **Yo'q** |
| Use case | Umumiy key-value | Private data, DOM metadata, cache |

```javascript
const map = new Map();
const weakMap = new WeakMap();

let user = { name: "Islom" };
map.set(user, "data");
weakMap.set(user, "data");

user = null;
// Map: object hali MAP ichida — GC tozalamaydi ❌
// WeakMap: object GC tozalashi mumkin → entry yo'qoladi ✅
```

WeakMap ishlatish kerak bo'lgan holatlar:
1. **DOM element metadata** — element o'chirilganda data ham yo'qolsin
2. **Private data** — class instance bilan birga GC bo'lsin
3. **Memoization** — argument object GC bo'lganda cache ham tozalansin

---

## Savol 8: Bu kodda memory leak bormi? Toping va tuzating. [Middle+]

```javascript
function createCounter() {
  const history = [];
  let count = 0;

  return {
    increment() {
      count++;
      history.push({ value: count, timestamp: Date.now() });
    },
    getCount() { return count; },
    getHistory() { return history; }
  };
}

const counter = createCounter();
setInterval(() => counter.increment(), 100);
```

**Javob:**

**2 ta muammo:**
1. **`history` array cheksiz o'sadi** — har 100ms da yangi object qo'shiladi, hech qachon tozalanmaydi
2. **`setInterval` to'xtatilmaydi** — clearInterval yo'q

```javascript
function createCounterFixed(maxHistory = 1000) {
  const history = [];
  let count = 0;

  return {
    increment() {
      count++;
      history.push({ value: count, timestamp: Date.now() });
      if (history.length > maxHistory) {
        history.splice(0, history.length - maxHistory); // eski entry tozalash ✅
      }
    },
    getCount() { return count; },
    getHistory() { return history; },
    startAutoIncrement(ms = 100) {
      const id = setInterval(() => this.increment(), ms);
      return () => clearInterval(id); // cleanup function qaytarish ✅
    }
  };
}

const counter = createCounterFixed(500);
const stop = counter.startAutoIncrement(100);
// Kerak bo'lmaganda: stop();
```

---

## Savol 9: WeakRef nima? Qachon ishlatiladi? [Middle+]

**Javob:**

`WeakRef` (ES2021) — object'ga **kuchsiz reference** yaratadi. Oddiy (strong) reference GC ga "bu object kerak" deydi. WeakRef esa GC ga to'sqinlik qilmaydi — object'ga boshqa strong reference bo'lmasa, GC tozalashi mumkin.

```javascript
let obj = { data: "important" };
let weakRef = new WeakRef(obj);

obj = null; // strong reference yo'q

// WeakRef dan qiymat olish — har doim null check KERAK:
const value = weakRef.deref();
if (value) {
  console.log(value.data); // object hali tirik
} else {
  console.log("GC tozaladi!"); // object yo'q bo'ldi
}
```

Ishlatiladi: **cache** tizimlari (katta object'larni kerak bo'lganda GC tozalashi uchun), **observer pattern** (kuzatilayotgan object o'chirilganda avtomatik unsubscribe).

Muhim: GC **qachon** ishlashi noaniq — `deref()` istalgan paytda `undefined` qaytarishi mumkin. Dastur logikasini bunga asoslamang — faqat optimization uchun.

---

## Savol 10: Quyidagi kodning output'ini ayting [Middle+]

```javascript
function outer() {
  let large = new Array(1000000).fill("x");

  return function inner() {
    return large.length;
  };
}

const fn = outer();
console.log(fn()); // ?

// large GC tozalay oladimi?
```

**Javob:**

```
1000000
```

`large` GC tozalay **olmaydi** — `inner` funksiya closure orqali `large` ga reference ushlab turadi. `fn` o'zgaruvchisi tirik ekan, `large` (1M elementli array) xotirada qoladi.

GC tozalashi uchun: `fn = null` qilish kerak — shunda `inner` → `large` reference chain uziladi va GC tozalay oladi.

Yaxshiroq yondashuv — faqat kerakli ma'lumotni capture qilish:

```javascript
function outerFixed() {
  const large = new Array(1000000).fill("x");
  const length = large.length; // faqat kerakli data

  return function inner() {
    return length; // large reference qilmaydi → GC tozalaydi ✅
  };
}
```

---

## Savol 11: Chrome DevTools da memory leak qanday topiladi? [Middle+]

**Javob:**

**Comparison Workflow** — eng samarali usul:

1. DevTools → Memory tab → "Heap snapshot" → **Snapshot 1** (baseline)
2. Leak bo'lishi kerak bo'lgan amallarni bajaring (sahifa ochish-yopish, modal, navigate)
3. 🗑️ bosish — **Force GC** (keraksiz object'larni tozalash)
4. Yana "Heap snapshot" → **Snapshot 2**
5. Snapshot 2 da **"Comparison"** tanlash → Snapshot 1 bilan solishtirish
6. **"# Delta"** ustunida ortiq qolgan object'lar — potentsial leak
7. Object'ni bosib **Retainers** panelda ko'rish — kim ushlab turyapti?

**Performance Monitor** (`Ctrl+Shift+P` → "Show Performance Monitor") — real-time:
- JS heap size, DOM Nodes, JS event listeners raqamlari faqat **o'sib** borsa → leak bor

**Shallow Size** — object o'zi egallagan xotira. **Retained Size** — object + u ushlab turgan barcha sub-object'lar. Retained Size muhimroq — leak hajmini ko'rsatadi.

---

## Savol 12: V8 da Generational GC nima? Young va Old Generation farqi? [Senior]

**Javob:**

V8 **Generational Hypothesis** ga asoslanadi: ko'pchilik object'lar qisqa muddatli. Shuning uchun heap ikki avlodga bo'lingan:

**Young Generation (New Space, 1-8MB):**
- Barcha yangi object'lar shu yerda yaratiladi
- Ikki **semi-space** (From, To) — Cheney's copying algorithm
- **Scavenger** (Minor GC) — tez (~1-5ms), tez-tez ishlaydi
- Tirik object'larni From → To ko'chiradi, keyin almashadi
- 2 marta omon qolgan object → Old Generation ga **promote** bo'ladi

**Old Generation (Old Space, yuzlab MB):**
- Yashagan (promoted) object'lar shu yerda
- **Mark-Compact** (Major GC) — sekinroq (~50-200ms), kamroq ishlaydi
- 3 bosqich: Mark (reachable belgilash) → Sweep (unreachable tozalash) → Compact (fragmentation kamaytirish)

**Orinoco** — V8 ning zamonaviy GC tizimi:
- **Incremental marking** — mark bosqichini kichik bo'laklarga bo'ladi
- **Concurrent marking** — background thread da, JS to'xtamaydi
- **Parallel** — bir nechta GC thread birgalikda ishlaydi

Bu kombinatsiya tufayli Major GC ham main thread ni minimal to'xtatadi (~60fps saqlanadi).

---

## Savol 13: `structuredClone` va `JSON.parse(JSON.stringify())` farqi nima? [Middle]

**Javob:**

| Xususiyat | `JSON.parse(JSON.stringify())` | `structuredClone()` |
|-----------|-------------------------------|---------------------|
| Date | ❌ String ga aylanadi | ✅ Date saqlanadi |
| RegExp | ❌ Bo'sh object `{}` | ✅ RegExp saqlanadi |
| Map/Set | ❌ Yo'qoladi | ✅ Saqlanadi |
| Circular ref | ❌ TypeError | ✅ To'g'ri ishlaydi |
| Function | ❌ Yo'qoladi | ❌ Yo'qoladi |
| undefined | ❌ Yo'qoladi | ✅ Saqlanadi |
| Tezlik | Sekinroq (serialize+parse) | Tezroq (binary copy) |

```javascript
const obj = { date: new Date(), set: new Set([1,2,3]) };

const json = JSON.parse(JSON.stringify(obj));
console.log(json.date); // "2024-01-15T..." — STRING ❌
console.log(json.set);  // {} ❌

const clone = structuredClone(obj);
console.log(clone.date); // Date object ✅
console.log(clone.set);  // Set {1, 2, 3} ✅
```

Qoida: shallow copy → spread/`Object.assign` (eng tez). Deep copy → `structuredClone` (to'g'ri + tez). Serialize kerak → JSON (network/storage uchun).

---

## Savol 14: Object pooling nima? Qachon ishlatiladi? [Senior]

**Javob:**

Object pooling — ko'p object yaratish-yo'q qilish o'rniga, mavjud object'larni qayta ishlatish patterni. GC yukini kamaytiradi va performance oshiradi.

```javascript
class ObjectPool {
  #pool = [];
  #factory;
  #reset;
  #maxSize;

  constructor(factory, reset, maxSize = 100) {
    this.#factory = factory;
    this.#reset = reset;
    this.#maxSize = maxSize;
  }

  acquire() {
    const obj = this.#pool.pop() || this.#factory();
    return obj; // mavjud object qaytariladi — yangi yaratilmaydi
  }

  release(obj) {
    this.#reset(obj);
    if (this.#pool.length < this.#maxSize) this.#pool.push(obj);
  }
}
```

Ishlatiladi:
- **Game loop** — har frame da particle/bullet object (60fps × 100 object = 6000/sec)
- **Animation** — DOM element reuse
- **Server** — request/response object reuse
- **Hot loop** — performance-critical hisoblashlar

Qachon **kerak emas**: oddiy CRUD, kam object yaratish, bir martalik amallar — GC yetarli darajada samarali.

**Deep Dive:** V8 da Young Generation (Scavenger) allaqachon copying collector — qisqa muddatli object'lar uchun optimallashtirilgan. Object pooling asosan Old Generation ga promote bo'lishni oldini olish uchun foydali.

---

## Savol 15: FinalizationRegistry nima? Real-world use case ayting. [Senior]

**Javob:**

`FinalizationRegistry` (ES2021) — object GC tomonidan tozalanganda callback chaqirish imkonini beradi. Bu tashqi resurslarni cleanup qilish uchun foydali.

```javascript
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`Cleanup: ${heldValue}`);
  // Tashqi resource bo'shatish: file handle, network connection, ...
});

let resource = { handle: openFile("data.txt") };
registry.register(resource, "data-file"); // "data-file" = identifier

resource = null;
// ... GC ishlaganda ... Console: "Cleanup: data-file"
```

**Real-world use case — WeakRef + FinalizationRegistry cache:**

```javascript
class SmartCache {
  #entries = new Map();
  #registry;

  constructor() {
    this.#registry = new FinalizationRegistry((key) => {
      const ref = this.#entries.get(key);
      if (ref && !ref.deref()) this.#entries.delete(key);
    });
  }

  set(key, value) {
    this.#entries.set(key, new WeakRef(value));
    this.#registry.register(value, key);
  }

  get(key) {
    const ref = this.#entries.get(key);
    if (!ref) return undefined;
    const value = ref.deref();
    if (!value) { this.#entries.delete(key); return undefined; }
    return value;
  }
}
```

Muhim: callback **QACHON** chaqirilishi kafolatlanmagan — GC o'z vaqtida tozalaydi. Deterministik cleanup kerak bo'lsa — `try/finally` yoki ES2025 `using`/`Symbol.dispose` ishlatish kerak.

---

## Savol 16: `AbortController` event listener cleanup uchun qanday ishlatiladi? [Middle+]

**Javob:**

`addEventListener` ning `signal` optsiyasi orqali bir nechta listener'ni bir yo'la olib tashlash mumkin — `abort()` chaqirilganda barcha listener'lar avtomatik remove bo'ladi:

```javascript
class Component {
  constructor() {
    this.controller = new AbortController();
    const { signal } = this.controller;

    window.addEventListener("scroll", () => this.onScroll(), { signal });
    window.addEventListener("resize", () => this.onResize(), { signal });
    document.addEventListener("keydown", () => this.onKey(), { signal });
    // 3 ta listener — barchasi bitta signal bilan
  }

  destroy() {
    this.controller.abort();
    // BARCHA 3 ta listener bir yo'la olib tashlandi ✅
    // removeEventListener har biriga alohida chaqirish shart emas
  }
}
```

Bu pattern ayniqsa SPA component'larida foydali — `destroy()`/`unmount()` da bitta `abort()` chaqirish barcha listener'larni tozalaydi. `removeEventListener` bilan funksiya reference saqlash va har biriga alohida chaqirish kerak edi.

---

## Savol 17: `deepClone` funksiyasini implement qiling [Senior]

**Savol:** `structuredClone` ishlatmasdan, circular reference'larni ham qo'llab-quvvatlaydigan `deepClone` yozing.

**Javob:**

```javascript
function deepClone(obj, seen = new WeakMap()) {
  // Primitive yoki null — to'g'ridan-to'g'ri qaytarish
  if (obj === null || typeof obj !== "object") return obj;

  // Circular reference tekshirish
  if (seen.has(obj)) return seen.get(obj);

  // Maxsus turlar
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

  // Array yoki Object
  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone); // circular ref uchun oldindan saqlash

  for (const key of Reflect.ownKeys(obj)) {
    clone[key] = deepClone(obj[key], seen);
  }
  return clone;
}

// Test:
const obj = { a: 1, b: { c: 2 }, d: new Date(), e: new Set([1, 2]) };
obj.self = obj; // circular

const clone = deepClone(obj);
console.log(clone.b.c);        // 2
console.log(clone.d instanceof Date); // true
console.log(clone.self === clone);    // true — circular saqlanadi
console.log(clone === obj);           // false — turli object
```

**Deep Dive:** `WeakMap` circular reference'larni saqlash uchun ideal — allaqachon clone qilingan object'ni qayta uchratsak, WeakMap'dan olamiz. `Reflect.ownKeys` — Symbol property'larni ham o'z ichiga oladi (`Object.keys` faqat string key).

---

## Savol 18: React useEffect da memory leak qanday oldini olish mumkin? [Middle+]

**Javob:**

React `useEffect` da **cleanup function** qaytarish — component unmount bo'lganda yoki dependency o'zgarganda chaqiriladi:

```javascript
// ❌ Memory leak — cleanup yo'q:
useEffect(() => {
  const handler = () => console.log("scroll");
  window.addEventListener("scroll", handler);
  // Component unmount bo'lganda listener QOLADI!
});

// ✅ To'g'ri — cleanup function:
useEffect(() => {
  const handler = () => console.log("scroll");
  window.addEventListener("scroll", handler);

  return () => window.removeEventListener("scroll", handler); // CLEANUP ✅
}, []);

// ✅ AbortController bilan (ko'p listener uchun qulay):
useEffect(() => {
  const controller = new AbortController();

  window.addEventListener("scroll", onScroll, { signal: controller.signal });
  window.addEventListener("resize", onResize, { signal: controller.signal });
  const intervalId = setInterval(pollData, 5000);

  return () => {
    controller.abort();        // barcha listener'lar ✅
    clearInterval(intervalId); // timer ✅
  };
}, []);

// ❌ Keng tarqalgan xato — async fetch cleanup:
useEffect(() => {
  let cancelled = false;

  async function fetchData() {
    const res = await fetch("/api/data");
    if (!cancelled) setData(await res.json()); // unmount check ✅
  }
  fetchData();

  return () => { cancelled = true; }; // stale update oldini olish
}, []);
```

Cleanup kerak bo'lgan holatlar: `addEventListener`, `setInterval`/`setTimeout`, WebSocket, subscription, async fetch.

---

## Savol 19: Quyidagi kodning output'ini ayting [Middle+]

```javascript
const weakMap = new WeakMap();
let obj1 = { a: 1 };
let obj2 = { b: 2 };

weakMap.set(obj1, "first");
weakMap.set(obj2, "second");

console.log(weakMap.has(obj1)); // ?

obj1 = null;
// GC ishlagandan keyin:
console.log(weakMap.has(obj1)); // ?
```

**Javob:**

```
true
false
```

- Birinchi `weakMap.has(obj1)` → `true` — `obj1` hali tirik, WeakMap da entry bor
- `obj1 = null` dan keyin — `weakMap.has(obj1)` aslida `weakMap.has(null)` → `false`

Muhim nuance: `obj1 = null` qilganda WeakMap dan entry **ham** yo'qoladi (GC tozalaganda), lekin `weakMap.has(null)` ning o'zi `false` qaytaradi chunki `null` WeakMap key bo'la olmaydi. Hatto `obj1` ga yangi object tayinlasangiz ham — eski entry yo'q, chunki eski object'ga boshqa reference yo'q.

---

## Savol 20: V8 da xotirani monitoring qilish va profiling qanday qilinadi? (Node.js) [Senior]

**Javob:**

Node.js da V8 xotira holati `process.memoryUsage()` orqali ko'riladi:

```javascript
const mem = process.memoryUsage();
console.log({
  rss: (mem.rss / 1024 / 1024).toFixed(2) + " MB",           // Resident Set Size
  heapTotal: (mem.heapTotal / 1024 / 1024).toFixed(2) + " MB", // V8 heap ajratgan
  heapUsed: (mem.heapUsed / 1024 / 1024).toFixed(2) + " MB",   // V8 heap ishlatayotgan
  external: (mem.external / 1024 / 1024).toFixed(2) + " MB",   // C++ object'lar
});
```

**Heap snapshot olish:**

```javascript
// Node.js da --inspect flag bilan:
// node --inspect app.js
// Chrome da chrome://inspect ochib Memory tab ishlatish

// Yoki programmatik:
const v8 = require('v8');
const fs = require('fs');

const snapshotStream = v8.writeHeapSnapshot();
console.log(`Heap snapshot: ${snapshotStream}`);
// Natija .heapsnapshot fayl — Chrome DevTools da ochish mumkin
```

**--max-old-space-size** — Old Generation hajmini cheklash:

```bash
# Default ~1.5GB (64-bit), ko'paytirish:
node --max-old-space-size=4096 app.js  # 4GB

# Debug uchun GC loglarini ko'rish:
node --trace-gc app.js
```

**Leak aniqlash pattern:**
1. `setInterval(() => console.log(process.memoryUsage().heapUsed), 5000)` — heap o'sishini kuzatish
2. `heapUsed` faqat o'sib borsa → leak bor
3. `--inspect` bilan Chrome DevTools Heap Snapshot + Comparison workflow
