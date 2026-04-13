# Bo'lim 27: ES2024+ va Kelgusi Standartlar

> ECMAScript har yili yangilanadi — TC39 komiteti proposal'larni Stage 0 dan Stage 4 gacha olib boradi. Stage 4 ga yetgan feature'lar rasmiy standartga qo'shiladi. Bu bo'limda ES2024 va ES2025 yangiliklari hamda kelgusi muhim proposal'lar batafsil yoritiladi.

---

## Mundarija

- [TC39 Process](#tc39-process)
- [Object.groupBy va Map.groupBy (ES2024)](#objectgroupby-va-mapgroupby-es2024)
- [Promise.withResolvers (ES2024)](#promisewithresolvers-es2024)
- [String isWellFormed va toWellFormed (ES2024)](#string-iswellformed-va-towellformed-es2024)
- [RegExp v Flag — Unicode Sets (ES2024)](#regexp-v-flag--unicode-sets-es2024)
- [Array.fromAsync (ES2024)](#arrayfromasync-es2024)
- [Atomics.waitAsync (ES2024)](#atomicswaitasync-es2024)
- [Set Methods (ES2025)](#set-methods-es2025)
- [Iterator Helpers (ES2025)](#iterator-helpers-es2025)
- [Explicit Resource Management — using (ES2025)](#explicit-resource-management--using-es2025)
- [Import Attributes (ES2025)](#import-attributes-es2025)
- [RegExp Pattern Modifiers (ES2025)](#regexp-pattern-modifiers-es2025)
- [Duplicate Named Capturing Groups (ES2025)](#duplicate-named-capturing-groups-es2025)
- [Kelgusi Proposals](#kelgusi-proposals)
- [Common Mistakes](#common-mistakes)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## TC39 Process

### Nazariya

TC39 — ECMAScript standartini boshqaradigan komitet. Har bir yangi feature quyidagi bosqichlardan o'tadi:

| Stage | Nomi | Ma'nosi |
|-------|------|---------|
| **0** | Strawperson | G'oya taklif qilingan, rasmiy hujjat yo'q |
| **1** | Proposal | Muammo aniqlangan, yechim taklif qilingan, champion tayinlangan |
| **2** | Draft | Spec matni yozilgan, syntax va semantika aniq |
| **2.7** | — | Spec matni tugallangan, lekin testlar va implementation'lar hali to'liq emas |
| **3** | Candidate | Spec tayyor, browser'lar implement qilmoqda, polyfill'lar bor |
| **4** | Finished | Kamida 2 ta implementation, Test262 testlari o'tgan — standartga qo'shiladi |

Stage 3+ feature'larni ishlatish xavfsiz — ular deyarli o'zgarmaydi. Stage 2 va undan pastlari hali o'zgarishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript har yilgi yangi versiyasi **iyun** oyida Ecma General Assembly tomonidan tasdiqlanadi. Feature Stage 4 ga yetishi uchun uning kutish chegarasi — o'sha yilgi **mart/aprel TC39 plenary** yig'ilishi (odatda iyundan 2-3 oy oldin). Shu cut-off'gacha Stage 4 ga yetgan proposal'lar keyingi ES spec'ga kiritiladi. Masalan: ES2024 cut-off = 2024-mart TC39 plenary, rasmiy nashr = 2024-iyun Ecma GA.

Browser vendor'lar (Chrome/V8, Firefox/SpiderMonkey, Safari/JavaScriptCore) odatda **Stage 3** dan implement qilishni boshlaydi — shuning uchun ES2025 feature'larining ko'pi allaqachon production browser'larda ishlaydi (Stage 3 = "Candidate", syntax va semantika muzlatilgan, asosan bug fix kutiladi).

</details>

---

## Object.groupBy va Map.groupBy (ES2024)

### Nazariya

`Object.groupBy()` — iterable elementlarini callback natijasiga qarab guruhlaydi. Natija oddiy object — har bir key uchun mos elementlar array'i. `Map.groupBy()` xuddi shunday, lekin natija `Map` bo'ladi — key sifatida istalgan qiymat (object, number, symbol) ishlatiladi.

Bu method'lar `Array.prototype.reduce()` bilan qo'lda guruhlash o'rniga qisqa, o'qilishi oson yechim beradi.

**Muhim farq:** `Object.groupBy()` — `Array.prototype` da emas, `Object` static method. Chunki natija — null-prototype object (`Object.create(null)` kabi), `Object.prototype` method'lari chaqirilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

```
Object.groupBy(items, callback) algoritmi:

1. items ni iterate qiladi
2. Har bir element uchun callback(element, index) chaqiradi
3. Callback qaytargan qiymat → key (PropertyKey ga coerce qilinadi)
4. null-prototype object yaratadi: Object.create(null)
5. Har bir key uchun elementlarni array ga to'playdi
6. Natija object'ni qaytaradi

Map.groupBy xuddi shunday, faqat Map qaytaradi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const products = [
  { name: "Olma", category: "meva", price: 5000 },
  { name: "Sabzi", category: "sabzavot", price: 3000 },
  { name: "Banan", category: "meva", price: 8000 },
  { name: "Kartoshka", category: "sabzavot", price: 4000 },
  { name: "Uzum", category: "meva", price: 12000 },
];

// Object.groupBy — category bo'yicha guruhlash
const grouped = Object.groupBy(products, product => product.category);
// {
//   meva: [
//     { name: "Olma", category: "meva", price: 5000 },
//     { name: "Banan", category: "meva", price: 8000 },
//     { name: "Uzum", category: "meva", price: 12000 },
//   ],
//   sabzavot: [
//     { name: "Sabzi", category: "sabzavot", price: 3000 },
//     { name: "Kartoshka", category: "sabzavot", price: 4000 },
//   ],
// }

// Narx bo'yicha guruhlash — arzon/qimmat
const byPrice = Object.groupBy(products, p =>
  p.price > 5000 ? "qimmat" : "arzon"
);
// { arzon: [...], qimmat: [...] }

// Map.groupBy — object key sifatida
const users = [
  { name: "Ali", role: { id: 1, title: "admin" } },
  { name: "Vali", role: { id: 2, title: "user" } },
  { name: "Soli", role: { id: 1, title: "admin" } },
];

const roles = {
  admin: { id: 1, title: "admin" },
  user: { id: 2, title: "user" },
};

// Map.groupBy — object reference'lar key bo'la oladi
const byRole = Map.groupBy(users, u =>
  u.role.id === 1 ? roles.admin : roles.user
);

byRole.get(roles.admin);
// [{ name: "Ali", ... }, { name: "Soli", ... }]

// ❌ Eski usul — reduce bilan qo'lda (ko'p kod)
const oldWay = products.reduce((acc, product) => {
  const key = product.category;
  (acc[key] ??= []).push(product);
  return acc;
}, {});

// ✅ Yangi usul — bir qator
const newWay = Object.groupBy(products, p => p.category);
```

</details>

---

## Promise.withResolvers (ES2024)

### Nazariya

`Promise.withResolvers()` — Promise constructor'ni ishlatmasdan, `resolve` va `reject` funksiyalarini tashqarida olish imkonini beradi. Bu **Deferred pattern** ning standart implementatsiyasi.

Ilgari bu pattern uchun quyidagi boilerplate kerak edi:

```javascript
// ❌ Eski usul — ko'p boilerplate
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});
```

`Promise.withResolvers()` buni bitta qatorda hal qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`Promise.withResolvers()` ichida aynan yuqoridagi eski pattern bajariladigan. Method yangi Promise yaratadi, executor'dan `resolve` va `reject` ni oladi, va uchala qiymatni bitta object sifatida qaytaradi.

```
Promise.withResolvers() qaytaradi:
{
  promise: Promise<T>,   ← yangi pending Promise
  resolve: (value) => void,  ← shu promise'ni resolve qilish
  reject: (reason) => void   ← shu promise'ni reject qilish
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ✅ ES2024 usul — toza, qisqa
const { promise, resolve, reject } = Promise.withResolvers();

// promise — pending holatda
// resolve/reject — istalgan joyda chaqirish mumkin

// Use case 1: Event-based async
function waitForClick(element) {
  const { promise, resolve } = Promise.withResolvers();
  element.addEventListener("click", resolve, { once: true });
  return promise;
}
// const event = await waitForClick(button);

// Use case 2: Timeout bilan promise
function promiseWithTimeout(asyncFn, ms) {
  const { promise, resolve, reject } = Promise.withResolvers();

  asyncFn().then(resolve, reject);
  setTimeout(() => reject(new Error("Timeout")), ms);

  return promise;
}

// Use case 3: Stream ni promise ga aylantirish
function readStream(stream) {
  const { promise, resolve, reject } = Promise.withResolvers();
  const chunks = [];

  stream.on("data", chunk => chunks.push(chunk));
  stream.on("end", () => resolve(Buffer.concat(chunks)));
  stream.on("error", reject);

  return promise;
}

// Use case 4: Manual trigger — test'larda foydali
function createDeferred() {
  return Promise.withResolvers();
}

const deferred = createDeferred();
// ... boshqa joyda, boshqa vaqtda:
deferred.resolve("tayyor!");
// ... kutayotgan joyda:
const result = await deferred.promise; // "tayyor!"
```

</details>

---

## String isWellFormed va toWellFormed (ES2024)

### Nazariya

JavaScript string'lari UTF-16 formatida saqlanadi. UTF-16 da ba'zi belgilar (emoji, CJK) **surrogate pair** — ikkita 16-bit code unit bilan ifodalanadi. Agar string'da **lone surrogate** (juftisiz surrogate) bo'lsa, bu "well-formed" emas — `encodeURIComponent()`, `TextEncoder`, JSON serialization kabi API'larda xatolik yoki noto'g'ri natija berishi mumkin.

- `isWellFormed()` — string'da lone surrogate borligini tekshiradi
- `toWellFormed()` — lone surrogate'larni `U+FFFD` (replacement character) bilan almashtiradi

<details>
<summary><strong>Under the Hood</strong></summary>

UTF-16 surrogate'lar:
- **High surrogate**: `0xD800` – `0xDBFF`
- **Low surrogate**: `0xDC00` – `0xDFFF`
- **Surrogate pair**: high + low = to'liq Unicode code point

Lone surrogate — high surrogate'dan keyin low surrogate kelmasa, yoki low surrogate high surrogate'siz tursa.

```
UTF-16 encoding:

"A"  → [0x0041]                    ← BMP (Basic Multilingual Plane)
"😀" → [0xD83D, 0xDE00]            ← surrogate pair (to'g'ri)
"\uD83D" → [0xD83D]                ← lone high surrogate (noto'g'ri!)
"\uDE00" → [0xDE00]                ← lone low surrogate (noto'g'ri!)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Well-formed string — oddiy belgilar va to'g'ri surrogate pair'lar
"Hello".isWellFormed();   // true
"😀".isWellFormed();      // true (to'g'ri surrogate pair)

// Lone surrogate — well-formed emas
"\uD800".isWellFormed();         // false (lone high surrogate)
"\uDFFF".isWellFormed();         // false (lone low surrogate)
"abc\uD800def".isWellFormed();   // false (o'rtada lone surrogate)

// toWellFormed — lone surrogate'ni almashtiradi
"\uD800".toWellFormed();         // "\uFFFD" (replacement character: �)
"abc\uD800def".toWellFormed();   // "abc�def"

// Nima uchun kerak? — encodeURIComponent() lone surrogate'da xatolik beradi
try {
  encodeURIComponent("\uD800"); // ❌ URIError: URI malformed
} catch (e) {
  console.log(e.message); // "URI malformed"
}

// ✅ Xavfsiz encoding
function safeEncode(str) {
  return encodeURIComponent(str.toWellFormed());
}
safeEncode("test\uD800"); // "test%EF%BF%BD" (xavfsiz)

// TextEncoder ham lone surrogate'ni qabul qilmaydi
const encoder = new TextEncoder();
encoder.encode("\uD800"); // Uint8Array [0xEF, 0xBF, 0xBD] — avtomatik almashtiradi

// Real-world: foydalanuvchi input'ini tozalash
function sanitizeInput(input) {
  if (!input.isWellFormed()) {
    return input.toWellFormed();
  }
  return input;
}
```

</details>

---

## RegExp v Flag — Unicode Sets (ES2024)

### Nazariya

`v` flag — `u` flag'ning kuchaytirilgan versiyasi. Unicode property'lar bilan ishlashni kengaytiradi va to'plam operatsiyalari (set operations) imkonini beradi: **difference** (`--`), **intersection** (`&&`), **union** (ichma-ich `[...]`).

`v` flag `u` flag'ni almashtirishga mo'ljallangan — ikkalasini birga ishlatib bo'lmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ u flag — cheklangan
/\p{Script=Greek}/u.test("α"); // true — basic Unicode property

// ✅ v flag — kengaytirilgan imkoniyatlar
/\p{Script=Greek}/v.test("α"); // true — v ham ishlaydi

// Set operations — difference (--)
// Barcha emoji'lardan raqam emoji'larni chiqarib tashlash
/[\p{Emoji}--\p{ASCII}]/v.test("😀"); // true
/[\p{Emoji}--\p{ASCII}]/v.test("1");  // false (ASCII chiqarildi)

// Intersection (&&) — ikkala to'plamda ham bor elementlar
// Lotin harflari VA kichik harflar
/[\p{Script=Latin}&&\p{Lowercase}]/v.test("a"); // true
/[\p{Script=Latin}&&\p{Lowercase}]/v.test("A"); // false (kichik emas)

// Nested classes
/[[\p{ASCII}]&&[\p{Alpha}]]/v.test("a"); // true
/[[\p{ASCII}]&&[\p{Alpha}]]/v.test("1"); // false (Alpha emas)

// String properties — multi-character match
/\p{RGI_Emoji_Flag_Sequence}/v.test("🇺🇿"); // true (O'zbekiston bayrog'i)

// Practical: faqat harflar (Unicode-safe, raqamsiz, belgisiz)
const letterOnly = /^[\p{Letter}]+$/v;
letterOnly.test("Hello");   // true
letterOnly.test("Привет");  // true
letterOnly.test("你好");     // true
letterOnly.test("Hello123"); // false
letterOnly.test("Hello!");   // false

// u va v farqi
// /[a-z]/u  — ishlaydi
// /[a-z]/v  — ishlaydi
// /[a-z]/uv — ❌ SyntaxError (ikkalasini birga bo'lmaydi)
```

</details>

---

## Array.fromAsync (ES2024)

### Nazariya

`Array.fromAsync()` — async iterable yoki async mapping'dan array yaratadi. `Array.from()` ning async versiyasi. `for await...of` loop yozish o'rniga bitta method chaqiruvi bilan array hosil qilish mumkin.

Qabul qiladigan argumentlar:
1. **Async iterable** — `Symbol.asyncIterator` ga ega object
2. **Iterable with async mapping** — oddiy iterable + async mapFn
3. **Array-like with async mapping** — `{ length, 0: ..., 1: ... }` + async mapFn

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Async generator → array
async function* asyncRange(start, end) {
  for (let i = start; i <= end; i++) {
    await new Promise(r => setTimeout(r, 100));
    yield i;
  }
}

const nums = await Array.fromAsync(asyncRange(1, 5));
// [1, 2, 3, 4, 5]

// Array.from analog — async mapFn bilan
const urls = ["/api/users/1", "/api/users/2", "/api/users/3"];

const users = await Array.fromAsync(urls, async url => {
  const res = await fetch(url);
  return res.json();
});
// [{ id: 1, ... }, { id: 2, ... }, { id: 3, ... }]
// ⚠️ KETMA-KET bajariladi, parallel EMAS!

// ✅ Parallel uchun — Promise.all ishlatish
const usersParallel = await Promise.all(urls.map(async url => {
  const res = await fetch(url);
  return res.json();
}));

// ReadableStream → array
async function streamToArray(stream) {
  return Array.fromAsync(stream);
}

// Promise'lar array'i — har biri resolve bo'lishini kutadi
const promises = [
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3),
];
const results = await Array.fromAsync(promises);
// [1, 2, 3]

// Array-like + async mapping
const mapped = await Array.fromAsync({ length: 3 }, async (_, i) => {
  const res = await fetch(`/api/item/${i}`);
  return res.json();
});
```

</details>

---

## Atomics.waitAsync (ES2024)

### Nazariya

`Atomics.waitAsync()` — `Atomics.wait()` ning **non-blocking** versiyasi. SharedArrayBuffer bilan ishlaydigan ko'p thread'li (Worker) dasturlarda thread'lar orasida sinxronizatsiya uchun ishlatiladi. `Atomics.wait()` main thread'da ishlamaydi (blocking), lekin `waitAsync()` istalgan thread'da ishlaydi — chunki Promise qaytaradi.

Bu API asosan **low-level concurrent programming** uchun — Web Workers orasida shared memory bilan ishlashda.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// SharedArrayBuffer — thread'lar orasidagi umumiy memory
// ⚠️ Cross-origin isolation majburiy: COOP: same-origin + COEP: require-corp
// (Spectre mitigation 2018+, aks holda SharedArrayBuffer mavjud emas)
const sab = new SharedArrayBuffer(1024);
const int32 = new Int32Array(sab);

// Main thread'da — async kutish
async function waitForSignal() {
  // Atomics.waitAsync qaytaradi: { async: boolean, value: Promise | string }
  // - async: false → value = "not-equal" (hozirgi qiymat expectedValue'ga teng emas, darhol qaytdi)
  // - async: true  → value = Promise, u "ok" (notify) yoki "timed-out" ga resolve bo'ladi
  const result = await Atomics.waitAsync(int32, 0, 0).value;
  // result: "not-equal" | "ok" | "timed-out"
  console.log("Signal holati:", result);
}

waitForSignal();

// Worker thread'da — signal yuborish
// Atomics.store(int32, 0, 1);
// Atomics.notify(int32, 0, 1);

// Timeout bilan va ikki case'ni to'g'ri boshqarish
const { value, async: isAsync } = Atomics.waitAsync(int32, 0, 0, 5000);
if (isAsync) {
  // Haqiqiy async kutish — Promise resolve bo'lishini kutamiz
  value.then(res => console.log("Async natija:", res));
  // 5000ms ichida notify bo'lmasa — "timed-out"
} else {
  // Darhol qaytdi — value aslida "not-equal" string
  console.log("Sync natija:", value);
}

// Farq: wait vs waitAsync
// Atomics.wait()      — BLOCKING (Worker'da ishlaydi, Main thread'da TypeError)
// Atomics.waitAsync() — NON-BLOCKING (Main thread + Worker, Promise qaytaradi)
```

</details>

---

## Set Methods (ES2025)

### Nazariya

ES2025 da `Set` ga matematik to'plam operatsiyalari qo'shildi. Ilgari bu amallarni qo'lda (filter, loop) yozish kerak edi. Yangi method'lar:

| Method | Nima qiladi | Matematik analog |
|--------|------------|-----------------|
| `union(other)` | Birlashma — ikkala to'plam | A ∪ B |
| `intersection(other)` | Kesishma — ikkala to'plamda bor | A ∩ B |
| `difference(other)` | Farq — A da bor, B da yo'q | A \ B |
| `symmetricDifference(other)` | Simmetrik farq — faqat bittasida bor | A △ B |
| `isSubsetOf(other)` | A to'liq B ichida | A ⊆ B |
| `isSupersetOf(other)` | B to'liq A ichida | A ⊇ B |
| `isDisjointFrom(other)` | Umumiy element yo'q | A ∩ B = ∅ |

Barcha method'lar **yangi Set qaytaradi** (mutate qilmaydi) yoki boolean qaytaradi (is* method'lari). Argument sifatida har qanday **Set-like object** qabul qiladi — `size` property va `has()`, `keys()` method'lari bo'lsa yetarli.

<details>
<summary><strong>Under the Hood</strong></summary>

```
Set operatsiyalari vizualizatsiya:

A = {1, 2, 3, 4, 5}
B = {3, 4, 5, 6, 7}

union(B):                {1, 2, 3, 4, 5, 6, 7}
intersection(B):         {3, 4, 5}
difference(B):           {1, 2}          ← A da bor, B da yo'q
symmetricDifference(B):  {1, 2, 6, 7}   ← faqat bittasida
isSubsetOf(B):           false           ← A ⊄ B
isDisjointFrom(B):       false           ← umumiy element bor (3,4,5)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const frontend = new Set(["HTML", "CSS", "JavaScript", "TypeScript"]);
const backend = new Set(["JavaScript", "TypeScript", "Python", "Go"]);

// Union — barcha texnologiyalar
frontend.union(backend);
// Set {"HTML", "CSS", "JavaScript", "TypeScript", "Python", "Go"}

// Intersection — ikkala sohada ham bor
frontend.intersection(backend);
// Set {"JavaScript", "TypeScript"}

// Difference — faqat frontend da bor
frontend.difference(backend);
// Set {"HTML", "CSS"}

// Symmetric difference — faqat bittasida bor
frontend.symmetricDifference(backend);
// Set {"HTML", "CSS", "Python", "Go"}

// Subset/superset tekshirish
const jsOnly = new Set(["JavaScript"]);
jsOnly.isSubsetOf(frontend);    // true — JS frontend ichida
frontend.isSupersetOf(jsOnly);  // true — frontend JS ni o'z ichiga oladi

// Disjoint — umumiy element yo'qmi?
const devops = new Set(["Docker", "Kubernetes", "Terraform"]);
frontend.isDisjointFrom(devops); // true — hech bir umumiy element yo'q
frontend.isDisjointFrom(backend); // false — JS va TS umumiy

// ❌ Eski usul — qo'lda yozish (ko'p kod)
function oldUnion(a, b) {
  return new Set([...a, ...b]);
}
function oldIntersection(a, b) {
  return new Set([...a].filter(x => b.has(x)));
}
function oldDifference(a, b) {
  return new Set([...a].filter(x => !b.has(x)));
}

// ✅ Yangi usul — bir qator
frontend.union(backend);
frontend.intersection(backend);
frontend.difference(backend);

// Real-world: foydalanuvchi ruxsatlari
const userPermissions = new Set(["read", "write", "delete"]);
const requiredPermissions = new Set(["read", "write"]);

requiredPermissions.isSubsetOf(userPermissions); // true — ruxsat bor
```

</details>

---

## Iterator Helpers (ES2025)

### Nazariya

Iterator Helpers — `Iterator.prototype` ga qo'shilgan method'lar. Array method'lariga o'xshash (`map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `toArray`, `forEach`, `some`, `every`, `find`), lekin muhim farqi bor — ular **lazy** (dangasa). Ya'ni elementlar faqat kerak bo'lganda hisoblandi — katta yoki cheksiz collection'lar bilan ishlashda juda samarali.

Array method'larida har bir qadam butun array ni qayta ishlaydi va yangi array yaratadi. Iterator helper'larda esa zanjir (pipeline) quriladi va elementlar birma-bir, kerak bo'lganda o'tadi.

| Method | Nima qiladi | Lazy? |
|--------|------------|-------|
| `.map(fn)` | Har bir elementni o'zgartiradi | ✅ Ha |
| `.filter(fn)` | Shartga mos elementlar | ✅ Ha |
| `.take(n)` | Birinchi n ta element | ✅ Ha |
| `.drop(n)` | Birinchi n tani o'tkazib yuboradi | ✅ Ha |
| `.flatMap(fn)` | map + flatten (1 daraja) | ✅ Ha |
| `.reduce(fn, init)` | Yig'ish | ❌ Yo'q (consuming) |
| `.toArray()` | Iterator → Array | ❌ Yo'q (consuming) |
| `.forEach(fn)` | Har bir elementga fn | ❌ Yo'q (consuming) |
| `.some(fn)` | Kamida bittasi true? | ❌ Yo'q (consuming) |
| `.every(fn)` | Hammasi true? | ❌ Yo'q (consuming) |
| `.find(fn)` | Birinchi mos element | ❌ Yo'q (consuming) |

<details>
<summary><strong>Under the Hood</strong></summary>

```
Lazy evaluation vizualizatsiya:

Array method'lar (eager):
  [1, 2, 3, 4, 5]
    .map(x => x * 2)      → [2, 4, 6, 8, 10]  ← yangi array
    .filter(x => x > 5)   → [6, 8, 10]          ← yana yangi array
    .take(2)               → ❌ bunday method yo'q

Iterator helper'lar (lazy):
  [1, 2, 3, 4, 5].values()
    .map(x => x * 2)      → iterator (hali hisob yo'q)
    .filter(x => x > 5)   → iterator (hali hisob yo'q)
    .take(2)               → iterator (hali hisob yo'q)
    .toArray()             → [6, 8] ← FAQAT SHU PAYTDA hisoblandi

  Qancha element ishlandi? Faqat 4 ta (1→2, 2→4, 3→6✅, 4→8✅ — 2 ta topildi, to'xtadi)
  Array usulida? 5 + 5 = 10 ta operatsiya + 2 ta oraliq array
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Asosiy ishlatish — Array iterator'dan
const result = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .values()                      // iterator olish
  .filter(n => n % 2 === 0)      // juft sonlar
  .map(n => n ** 2)              // kvadratga
  .take(3)                       // birinchi 3 tasi
  .toArray();
// [4, 16, 36]

// Cheksiz iterator bilan — lazy bo'lgani uchun ishlaydi!
function* naturals() {
  let n = 1;
  while (true) yield n++;  // cheksiz ketma-ketlik
}

// Birinchi 5 ta tub sonni topish
function isPrime(n) {
  if (n < 2) return false;
  for (let i = 2; i * i <= n; i++) {
    if (n % i === 0) return false;
  }
  return true;
}

const primes = naturals()
  .filter(isPrime)
  .take(5)
  .toArray();
// [2, 3, 5, 7, 11] — cheksiz generator'dan faqat 5 ta olindi

// drop — birinchi n tani o'tkazib yuborish
naturals()
  .drop(5)     // 1-5 ni o'tkazib yuboradi
  .take(3)     // keyin 3 tani oladi
  .toArray();
// [6, 7, 8]

// flatMap — har bir elementdan bir nechta element
[1, 2, 3].values()
  .flatMap(n => [n, n * 10])
  .toArray();
// [1, 10, 2, 20, 3, 30]

// reduce — yig'ish (lazy emas — to'liq iterate qiladi)
[1, 2, 3, 4, 5].values()
  .filter(n => n % 2 === 0)
  .reduce((sum, n) => sum + n, 0);
// 6 (2 + 4)

// forEach — side effect'lar uchun
[1, 2, 3].values()
  .map(n => n * 2)
  .forEach(n => console.log(n));
// 2, 4, 6

// some / every / find
[1, 2, 3, 4, 5].values().some(n => n > 3);  // true
[1, 2, 3, 4, 5].values().every(n => n > 0);  // true
[1, 2, 3, 4, 5].values().find(n => n > 3);   // 4

// Map va Set iterator'lari bilan
const map = new Map([["a", 1], ["b", 2], ["c", 3]]);
map.values()
  .filter(v => v > 1)
  .toArray();
// [2, 3]

const set = new Set([1, 2, 3, 4, 5]);
set.values()
  .map(n => n * 10)
  .take(3)
  .toArray();
// [10, 20, 30]

// Iterator.from() — iterable → iterator (helper method'lar bilan)
Iterator.from({ [Symbol.iterator]() {
  let i = 0;
  return { next() { return { value: i++, done: i > 5 }; } };
}})
  .map(n => n * 2)
  .toArray();
// [0, 2, 4, 6, 8]

// Performance qiyoslash — katta array'da
const bigArray = Array.from({ length: 1_000_000 }, (_, i) => i);

// ❌ Eager — 3 ta oraliq array yaratiladi
bigArray
  .map(n => n * 2)       // 1M yangi array
  .filter(n => n > 100)  // ~1M yangi array
  .slice(0, 10);         // natija

// ✅ Lazy — oraliq array YARATILMAYDI
bigArray.values()
  .map(n => n * 2)
  .filter(n => n > 100)
  .take(10)
  .toArray();
// Faqat ~60 ta element ishlandi (10 ta topilguncha)
```

</details>

---

## Explicit Resource Management — using (ES2025)

### Nazariya

`using` va `await using` — resurslarni **avtomatik tozalash** uchun yangi syntax. Fayl handle'lar, database connection'lar, lock'lar kabi resurslarni ochib, ishlatib, keyin **majburiy yopish** kerak bo'lganda ishlatiladi.

C# dagi `using`, Python dagi `with`, va Java dagi `try-with-resources` ga o'xshash. JavaScript'da ilgari `try/finally` bilan yozilgan edi — yangi syntax buni ancha soddalashtiradi.

Ishlash uchun resource object'da **`Symbol.dispose`** (sync) yoki **`Symbol.asyncDispose`** (async) method bo'lishi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

```
using da nima sodir bo'ladi:

using resource = getResource();
// ... resource ishlatish ...
// Scope tugaganda AVTOMATIK:
// resource[Symbol.dispose]() chaqiriladi

Bu aslida quyidagiga kompilyatsiya qilinadi:

const resource = getResource();
try {
  // ... resource ishlatish ...
} finally {
  resource[Symbol.dispose]();
}
```

```
Lifecycle:

1. using x = expr    → expr baholanadi
2. x scope ichida ishlatiladi
3. Scope tugaganda (normal yoki exception):
   → x[Symbol.dispose]() chaqiriladi
   → Agar dispose xato bersa — SuppressedError yaratiladi

await using uchun:
   → x[Symbol.asyncDispose]() chaqiriladi (await bilan)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Symbol.dispose — sync resource
class FileHandle {
  #path;
  #content = "";

  constructor(path) {
    this.#path = path;
    console.log(`Fayl ochildi: ${path}`);
  }

  write(data) {
    this.#content += data;
  }

  [Symbol.dispose]() {
    console.log(`Fayl yopildi: ${this.#path}`);
    // Fayl yopish logikasi
  }
}

// using — scope tugaganda avtomatik yopiladi
{
  using file = new FileHandle("/tmp/data.txt");
  file.write("Hello");
  file.write(" World");
  // Scope tugadi → [Symbol.dispose]() chaqirildi
  // Console: "Fayl ochildi: /tmp/data.txt"
  //          "Fayl yopildi: /tmp/data.txt"
}

// Symbol.asyncDispose — async resource
class DatabaseConnection {
  #url;

  constructor(url) {
    this.#url = url;
  }

  async query(sql) {
    // ... query bajarish
  }

  async [Symbol.asyncDispose]() {
    console.log("Connection yopilmoqda...");
    // await connection.close();
  }
}

// await using — async dispose
async function getUser(id) {
  await using db = new DatabaseConnection("postgres://...");
  const user = await db.query(`SELECT * FROM users WHERE id = ${id}`);
  return user;
  // Function tugaganda → await db[Symbol.asyncDispose]() chaqiriladi
}

// DisposableStack — bir nechta resource birga boshqarish
{
  using stack = new DisposableStack();

  const file1 = stack.use(new FileHandle("/tmp/a.txt"));
  const file2 = stack.use(new FileHandle("/tmp/b.txt"));

  file1.write("data1");
  file2.write("data2");

  // Scope tugaganda — ikkalasi ham yopiladi (teskari tartibda)
}

// AsyncDisposableStack — async versiya
async function processFiles() {
  await using stack = new AsyncDisposableStack();

  const db = stack.use(await connectDB());
  const cache = stack.use(await connectRedis());

  // ... ishlash ...
  // Scope tugaganda: avval cache, keyin db yopiladi
}

// ❌ Eski usul — try/finally (ko'p boilerplate)
async function oldWay() {
  const db = await connectDB();
  try {
    const cache = await connectRedis();
    try {
      // ... ishlash ...
    } finally {
      await cache.close();
    }
  } finally {
    await db.close();
  }
}

// ✅ Yangi usul — using bilan
async function newWay() {
  await using db = await connectDB();
  await using cache = await connectRedis();
  // ... ishlash ...
  // Avtomatik yopiladi — teskari tartibda
}

// SuppressedError — asosiy xato va dispose xatosi birga
// Agar try block xato bersa VA dispose ham xato bersa
// → SuppressedError yaratiladi:
// {
//   error: dispose xatosi,
//   suppressed: asosiy xato
// }
```

</details>

---

## Import Attributes (ES2025)

### Nazariya

Import Attributes — `import` statement'ga qo'shimcha metadata berish imkonini beradi. Asosan **JSON modul'lar** va boshqa non-JavaScript fayllarni import qilishda tur ko'rsatish uchun ishlatiladi. Bu xavfsizlik uchun muhim — server noto'g'ri Content-Type qaytarsa ham, import qilingan fayl faqat ko'rsatilgan tur sifatida ishlanadi.

Oldingi nomlanishi "Import Assertions" edi — keyinchalik "Import Attributes" ga o'zgartirildi (semantik farq: assertion faqat tekshiradi, attribute import xatti-harakatini o'zgartirishi mumkin).

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// JSON import — type attribute bilan
import data from "./config.json" with { type: "json" };
console.log(data.apiUrl); // JSON object sifatida ishlatiladi

// Dynamic import bilan
const config = await import("./config.json", {
  with: { type: "json" },
});

// CSS import (future — hali standartda emas, lekin yo'nalish shu)
// import styles from "./styles.css" with { type: "css" };

// Nima uchun kerak? — Xavfsizlik
// Agar server noto'g'ri Content-Type qaytarsa (masalan, JSON o'rniga JS):
// type: "json" — faqat JSON sifatida parse qilinishini kafolatlaydi
// JavaScript sifatida execute bo'lishiga YO'L QO'YMAYDI

// Re-export bilan
export { default as config } from "./config.json" with { type: "json" };

// ⚠️ Eski syntax (assert) — deprecated
// import data from "./config.json" assert { type: "json" }; // ❌ eski
// import data from "./config.json" with { type: "json" };    // ✅ yangi
```

</details>

---

## RegExp Pattern Modifiers (ES2025)

### Nazariya

RegExp Pattern Modifiers — regex ichida **ma'lum bir qism** uchun flag'larni yoqish yoki o'chirish imkonini beradi. `(?flags:pattern)` syntax'i bilan — butun regex uchun emas, faqat bir qism uchun flag o'zgartirish.

| Syntax | Ma'nosi |
|--------|---------|
| `(?i:pattern)` | Shu qism uchun case-insensitive |
| `(?-i:pattern)` | Shu qism uchun case-sensitive (i ni o'chirish) |
| `(?m:pattern)` | Shu qism uchun multiline |
| `(?s:pattern)` | Shu qism uchun dotAll (`.` = `\n` ham) |

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Faqat bir qismda case-insensitive
const re = /hello (?i:world)/;
re.test("hello World"); // true — "World" case-insensitive
re.test("Hello world"); // false — "Hello" case-sensitive!

// Amaliy misol — HTTP header parsing
// Header nomi case-insensitive, qiymati case-sensitive
const headerRe = /^(?i:content-type): (.+)$/;
headerRe.test("Content-Type: text/html");  // true
headerRe.test("CONTENT-TYPE: text/html");  // true
headerRe.test("content-type: TEXT/HTML");  // true — qiymat saqlanadi

// Flag'ni o'chirish
const re2 = /(?-i:EXACT) match/i;
// Butun regex case-insensitive (i flag)
// Lekin "EXACT" qismi case-SENSITIVE (i o'chirildi)
re2.test("EXACT Match");  // true
re2.test("exact Match");  // false — "exact" ≠ "EXACT"
re2.test("EXACT match");  // true
```

</details>

---

## Duplicate Named Capturing Groups (ES2025)

### Nazariya

ES2025 dan boshlab regex'da turli **alternative** (`|`) branch'larda **bir xil nomli** capture group ishlatish mumkin. Ilgari har bir nomli group nomi unikal bo'lishi kerak edi — bu alternation pattern'larda qo'shimcha murakkablik yaratardi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Ilgari — har bir branch'da boshqa nom kerak edi
const oldRe = /(?<year>\d{4})-(?<month>\d{2})|(?<month2>\d{2})\/(?<year2>\d{4})/;
// "month" va "month2", "year" va "year2" — chalkash

// ✅ ES2025 — bir xil nom, farqli branch'larda
const dateRe = /(?<year>\d{4})-(?<month>\d{2})|(?<month>\d{2})\/(?<year>\d{4})/;

// ISO format: 2024-03
const isoMatch = dateRe.exec("2024-03");
isoMatch.groups.year;  // "2024"
isoMatch.groups.month; // "03"

// US format: 03/2024
const usMatch = dateRe.exec("03/2024");
usMatch.groups.year;  // "2024"
usMatch.groups.month; // "03"

// Ikkalasida ham .groups.year va .groups.month ishlaydi!
// Qaysi branch match bo'lsa — o'sha branch'dagi group qaytariladi

// Amaliy misol — turli vaqt formatlarini parse qilish
const timeRe = /(?<h>\d{1,2}):(?<m>\d{2})(?::(?<s>\d{2}))?|(?<h>\d{1,2})h(?<m>\d{2})/;

timeRe.exec("14:30").groups;    // { h: "14", m: "30", s: undefined }
timeRe.exec("14:30:45").groups; // { h: "14", m: "30", s: "45" }
timeRe.exec("14h30").groups;    // { h: "14", m: "30" }
```

</details>

---

## Kelgusi Proposals

### Nazariya

Quyidagi proposal'lar turli bosqichlarda — ba'zilari Stage 3 da (browser'lar implement qilmoqda), ba'zilari esa hali ertaroq bosqichlarda (syntax o'zgarishi mumkin).

### Temporal API

Date object'ni almashtirishga mo'ljallangan yangi API — immutable, 1-indexed oy, timezone-safe, nanosecond aniqlik. Batafsil [26-date-intl.md](26-date-intl.md) da yoritilgan.

```javascript
const date = Temporal.PlainDate.from("2024-03-13");
date.add({ months: 1 }); // 2024-04-13 (immutable, yangi object)
date.month; // 3 (1-indexed!)
```

### Decorators

Class va class member'lariga meta-behavior qo'shish uchun standart syntax. TypeScript allaqachon o'z decorator'larini implement qilgan, lekin TC39 standarti farqli.

```javascript
// Stage 3 Decorators — method decorator signature
// Birinchi parameter (value): decorated bo'layotgan method (function)
// Ikkinchi parameter (context): { kind, name, static, private, addInitializer, access, metadata }
function logged(value, context) {
  if (context.kind !== "method") return value;
  const methodName = context.name;
  return function (...args) {
    console.log(`${methodName} chaqirildi:`, args);
    return value.apply(this, args); // asl method'ni this bilan chaqirish
  };
}

class Calculator {
  @logged
  add(a, b) {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// Console: "add chaqirildi: [2, 3]"
// Return: 5

// ⚠️ Muhim: Stage 3 (TC39) decorators va TypeScript "experimental decorators"
// (tsconfig `experimentalDecorators`) — BOSHQA-BOSHQA proposal'lar.
// TS 5+ `experimentalDecorators: false` bilan Stage 3 syntax'ga o'tgan.
```

### Pattern Matching

`switch` ning kuchliroq versiyasi — qiymatlarni murakkab pattern'lar bilan solishtirish.

```javascript
// ⚠️ Hali Stage 1 — syntax o'zgarishi mumkin
// Taxminiy syntax:
match (response) {
  when ({ status: 200, body }) -> handleSuccess(body),
  when ({ status: 404 }) -> handleNotFound(),
  when ({ status: 500, body: { message } }) -> handleError(message),
  default -> handleUnknown(),
}
```

### Record va Tuple

Immutable va deeply comparable data structure'lar — primitive sifatida.

```javascript
// ⚠️ Status (2026-04): Proposal Stage 2 da ko'p yil qolib ketdi — 2024-25 da
// championship o'zgardi va engine implementation'dagi murakkablik tufayli
// asl "new primitive" yondashuvi qayta ko'rib chiqilmoqda. TC39'da alternativ
// variant — "Composites" yoki "Keyed Collections" kabi — muhokama qilinmoqda.
// Quyidagi `#{...}` syntax'i oxirgi shakl bo'lmasligi mumkin.

const record = #{ name: "Ali", age: 25 };
const tuple = #[1, 2, 3];

// Deep equality — primitive kabi ===
#{ a: 1 } === #{ a: 1 }; // true (object'da false bo'lardi!)
#[1, 2] === #[1, 2];     // true

// Immutable
// record.name = "Vali"; // ❌ TypeError
```

**Hozirgi alternativ (production'da):** `Object.freeze()` deep + custom structural equality (lodash `isEqual`), yoki Immutable.js / Immer kutubxonalari.

---

## Common Mistakes

### ❌ Xato 1: Array.fromAsync ketma-ket bajarilishini bilmaslik

```javascript
// ❌ Parallel deb o'ylash — aslida ketma-ket
const results = await Array.fromAsync(urls, async url => {
  const res = await fetch(url);
  return res.json();
});
// Har bir fetch OLDINGI fetch tugashini KUTADI

// ✅ Parallel bajarish uchun — Promise.all
const results = await Promise.all(
  urls.map(async url => {
    const res = await fetch(url);
    return res.json();
  })
);
```

**Nima uchun:** `Array.fromAsync` elementlarni birma-bir qayta ishlaydi — har bir element uchun avval `await` qiladi, keyin keyingisiga o'tadi. Bu lazy/sequential by design.

---

### ❌ Xato 2: Iterator helper'lar Array'da to'g'ridan-to'g'ri ishlamaydi

```javascript
// ❌ Array'da helper method'lar YO'Q
[1, 2, 3].take(2); // ❌ TypeError: .take is not a function

// ✅ Avval iterator olish kerak
[1, 2, 3].values().take(2).toArray(); // ✅ [1, 2]

// Yoki Iterator.from()
Iterator.from([1, 2, 3]).take(2).toArray(); // ✅ [1, 2]
```

**Nima uchun:** Iterator helper'lar `Iterator.prototype` da, `Array.prototype` da emas. `.values()`, `.keys()`, `.entries()` yoki `Iterator.from()` bilan iterator olish kerak.

---

### ❌ Xato 3: Set method'lar asl Set'ni mutate qiladi deb o'ylash

```javascript
const a = new Set([1, 2, 3]);
const b = new Set([3, 4, 5]);

const union = a.union(b);

console.log(a); // Set {1, 2, 3} — O'ZGARMAGAN ✅
console.log(union); // Set {1, 2, 3, 4, 5} — YANGI Set
```

**Nima uchun:** Barcha Set method'lari (union, intersection, difference, ...) **yangi Set qaytaradi** — asl Set'ni mutate qilmaydi.

---

### ❌ Xato 4: using scope'ni tushunmaslik

```javascript
// ❌ using scope tashqarisida resource'ga murojaat
function process() {
  {
    using file = openFile("data.txt");
    file.write("test");
  }
  // Shu yerda file allaqachon DISPOSE bo'lgan!
  // file.write("more"); // ❌ Resource closed
}

// ✅ using scope ichida barcha ishni tugatish
function process() {
  using file = openFile("data.txt");
  file.write("test");
  file.write("more");
  // Function tugaganda dispose bo'ladi
}
```

---

### ❌ Xato 5: Object.groupBy natijasida Object.prototype method'lar kutish

```javascript
const grouped = Object.groupBy([1, 2, 3], n => n > 1 ? "big" : "small");

// ❌ Object.prototype method'lari yo'q
grouped.hasOwnProperty("big"); // ❌ TypeError!

// Nima uchun? grouped = Object.create(null)
// null-prototype object — prototype CHAIN yo'q

// ✅ To'g'ri tekshirish
"big" in grouped;         // true ✅
Object.hasOwn(grouped, "big"); // true ✅
```

---

## Edge Cases va Gotchas

Ushbu bo'lim Common Mistakes'da qamrab olinmagan, lekin ES2024+ feature'larni production'da ishlatishda uchraydigan nozik holatlarni tavsiflaydi.

### Gotcha 1: `Object.groupBy` key coercion — object kalitlar bir guruhga tushadi

`Object.groupBy` callback qaytargan qiymatni `ToPropertyKey` orqali **string'ga coerce qiladi**. Bu har qanday non-string/non-symbol qiymat string ifodasi sifatida saqlanishiga olib keladi. Object'lar esa `"[object Object]"` ga aylanadi — natijada **barcha object kalitlar bir guruhga tushib ketadi**.

```javascript
const data = [
  { type: { id: 1 }, value: "a" },
  { type: { id: 2 }, value: "b" },
  { type: { id: 1 }, value: "c" },
];

// ❌ Object.groupBy — object kalitlar "[object Object]" ga coerce bo'ladi
const wrong = Object.groupBy(data, item => item.type);
// {
//   "[object Object]": [
//     { type: { id: 1 }, value: "a" },
//     { type: { id: 2 }, value: "b" },  // ← ikkala id aralashib ketdi!
//     { type: { id: 1 }, value: "c" },
//   ]
// }

// ❌ Boolean ham noto'g'ri — string'ga coerce bo'ladi
Object.groupBy([1, 2, 3], n => n > 1);
// { "false": [1], "true": [2, 3] } — boolean emas, string kalit!

// ✅ Object kalit kerak bo'lsa — Map.groupBy ishlating
const right = Map.groupBy(data, item => item.type);
// Map { { id: 1 } → [...], { id: 2 } → [...] }
// Lekin eslatma: Map.groupBy identity-based — ikki har xil { id: 1 } object
// alohida kalit hisoblanadi. Normalizatsiya uchun avval mapping qiling.

// ✅ String kalitni majburiy qilish — JSON.stringify bilan
const byIdString = Object.groupBy(data, item => JSON.stringify(item.type));
// { '{"id":1}': [...], '{"id":2}': [...] }
```

**Nima uchun:** `Object.groupBy` spec'da callback natijasi `ToPropertyKey` abstract operatsiyasidan o'tadi — bu string yoki symbol'ga coerce qiladi. `Map.groupBy` esa SameValueZero taqqoslash ishlatadi, shuning uchun object identity saqlanadi.

**Yechim:** Primitive (string/number) kalit uchun `Object.groupBy`, object yoki Symbol kalit uchun `Map.groupBy`. Composite kalit uchun `JSON.stringify()` yoki explicit ID maydoni.

---

### Gotcha 2: Iterator helper pipeline — single-use semantics

Iterator helper'lar yaratgan pipeline **bir marta ishlatiladi**. Siz `.toArray()` chaqirganingizdan yoki for-of bilan consume qilganingizdan keyin iterator **exhausted** holatga o'tadi. Qayta chaqirish bo'sh natija beradi. Bu array method'laridan tub farqi — array har safar boshidan boshlanadi.

```javascript
const pipeline = [1, 2, 3, 4, 5]
  .values()
  .map(n => n * 2)
  .filter(n => n > 4);

// ✅ Birinchi consume — kutilganidek ishlaydi
const first = pipeline.toArray();
// [6, 8, 10]

// ❌ Ikkinchi consume — bo'sh!
const second = pipeline.toArray();
// [] — iterator allaqachon exhausted

// Worse: for-of dan keyin ham yo'qoladi
const iter = [1, 2, 3].values().map(n => n * 2);
for (const x of iter) console.log(x); // 2, 4, 6
for (const x of iter) console.log(x); // — hech narsa

// ✅ Qayta ishlatish kerak bo'lsa — har safar yangi iterator
function makeIter() {
  return [1, 2, 3, 4, 5].values().map(n => n * 2).filter(n => n > 4);
}
makeIter().toArray(); // [6, 8, 10]
makeIter().toArray(); // [6, 8, 10] ✅

// ✅ Yoki natijani array'ga saqlab qo'yish
const cached = makeIter().toArray();
cached; // qayta ishlatish mumkin
```

**Nima uchun:** Iterator protocol'ida `next()` state'ni oldinga siljitadi — bu fundamental one-way traversal. Helper'lar ichki iterator'ni wrap qiladi, lekin state'ni reset qila olmaydi (asl manba reusable bo'lsa ham). Bu lazy evaluation'ning narxi.

**Yechim:** Pipeline'ni funksiya ichida yarating (har chaqiruvda yangi), yoki oxirgi natijani array'ga `toArray()` bilan saqlang.

---

### Gotcha 3: `using` va `return` bilan qiymat qaytarish — dispose vaqti

`using` declaration'li funksiyada `return resource` yoki `return resource.value` yozganda, qaytariladigan qiymat **dispose'dan oldin** hisoblanadi, lekin resource dispose'dan **keyin** chaqiruvchiga qaytariladi. Bu "yopilgan resource'ga ega bo'lgan reference" xavfini yaratadi.

```javascript
class Connection {
  #closed = false;

  query(sql) {
    if (this.#closed) throw new Error("Connection yopilgan");
    return { rows: [/* ... */] };
  }

  [Symbol.dispose]() {
    this.#closed = true;
    console.log("Connection yopildi");
  }
}

// ❌ Xavfli pattern — resource reference'ni qaytarish
function getConnection() {
  using conn = new Connection();
  return conn; // dispose chaqiriladi, lekin reference tashqariga chiqadi
}

const c = getConnection();
// Console: "Connection yopildi"
c.query("SELECT *"); // ❌ Error: Connection yopilgan

// ❌ Yanada xavfli — qaytarilgan method chaqiriladi
function getQueryFn() {
  using conn = new Connection();
  return () => conn.query("SELECT *"); // closure — conn dispose bo'lgan!
}

const fn = getQueryFn();
fn(); // ❌ Error: Connection yopilgan

// ✅ Natijani scope ichida oling
function fetchData() {
  using conn = new Connection();
  return conn.query("SELECT *"); // natija (POJO) — reference emas
  // dispose — OK, chunki biz primitive/POJO qaytarmoqdamiz
}

fetchData(); // { rows: [...] } ✅ — connection allaqachon yopilgan, lekin bizga kerak emas

// ✅ Yoki scope'ni manual boshqarish — tashqariga reference kerak bo'lsa
async function processWithConn(task) {
  using conn = new Connection();
  return await task(conn); // task scope ichida tugaydi
}

await processWithConn(async conn => {
  const result = conn.query("...");
  return result; // POJO — conn dispose'dan keyin ham mavjud
});
```

**Nima uchun:** `using` spec-level `try/finally` kabi kompilyatsiya bo'ladi — `finally` blokidagi dispose `return` qiymat hisoblanganidan **keyin**, lekin chaqiruvchi uni olishdan **oldin** bajariladi. Bu mexanizm scope-bound lifecycle'ni kafolatlaydi.

**Yechim:** `using` scope'idan faqat **primitive** yoki **resource'ga bog'liq bo'lmagan ma'lumot** qaytaring. Resource'ning o'zini yoki uning method'ini hech qachon tashqariga bermang.

---

### Gotcha 4: `Promise.withResolvers` detached reference — uzoq pending xavfi

`Promise.withResolvers()` `resolve`/`reject` funksiyalarini scope'dan chiqarib beradi. Agar bu funksiyalar hech qachon chaqirilmasa, yoki chaqiruvchi object GC bo'lib ketsa ham, **promise pending holatda qoladi** va barcha `.then()` handler'lari xotirada saqlanadi — memory leak.

```javascript
// ❌ Xavfli pattern — event source'siz pending promise
class EventBus {
  #waiters = new Map();

  waitFor(event) {
    const { promise, resolve } = Promise.withResolvers();
    this.#waiters.set(event, resolve);
    return promise;
    // ⚠️ Agar event hech qachon emit bo'lmasa — promise abadiy pending
  }

  emit(event, data) {
    const resolve = this.#waiters.get(event);
    if (resolve) {
      resolve(data);
      this.#waiters.delete(event);
    }
  }
}

const bus = new EventBus();

// Ko'p joylardan kutish
async function handler1() {
  const data = await bus.waitFor("ready");
  // ... data ishlatish
}

handler1(); // "ready" hech qachon emit bo'lmasa — handler1 abadiy pending
// + microtask queue'da catch/then chain xotirani ushlab turadi

// ✅ Timeout bilan majburiy cleanup
function waitWithTimeout(bus, event, ms) {
  const { promise, resolve, reject } = Promise.withResolvers();

  const timer = setTimeout(() => {
    reject(new Error(`Timeout: ${event}`));
    bus.off(event, resolve); // listener'ni olib tashlash
  }, ms);

  bus.once(event, data => {
    clearTimeout(timer);
    resolve(data);
  });

  return promise;
}

// ✅ Yoki AbortSignal bilan cancellation
function waitForCancellable(bus, event, signal) {
  const { promise, resolve, reject } = Promise.withResolvers();

  if (signal?.aborted) {
    reject(new DOMException("Aborted", "AbortError"));
    return promise;
  }

  const onEvent = data => {
    signal?.removeEventListener("abort", onAbort);
    resolve(data);
  };
  const onAbort = () => {
    bus.off(event, onEvent);
    reject(new DOMException("Aborted", "AbortError"));
  };

  bus.once(event, onEvent);
  signal?.addEventListener("abort", onAbort);

  return promise;
}
```

**Nima uchun:** Promise'ning lifecycle'i undan foydalanuvchi kodga emas, resolve/reject chaqiruviga bog'liq. `Promise.withResolvers` bu nazoratni tashqariga beradi — lekin "nazorat" = "mas'uliyat". Eski `new Promise(executor)` pattern'ida executor tugagach engine resolve/reject'ga reference'ni boshqa ushlab turmas edi (agar developer saqlamagan bo'lsa).

**Yechim:** Har bir `Promise.withResolvers()` uchun **uch savolga javob bering**: (1) Resolve qachon chaqiriladi? (2) Timeout/cancellation bormi? (3) Abandoned holatda cleanup mavjudmi? Javoblar aniq bo'lmasa — `setTimeout` yoki `AbortSignal` bilan cheklang.

---

### Gotcha 5: RegExp `v` flag strict character class — eski `u` pattern'lar xato beradi

`v` flag `u` flag'ning yangi versiyasi, lekin **strict character class parsing** bilan. Ayrim `u` flag'da ruxsat etilgan pattern'lar `v` flag'da SyntaxError beradi. Bu migration'da nozik tuzoq — "shunchaki `u` ni `v` bilan almashtirish" ishlamaydi.

```javascript
// ❌ v flag'da SyntaxError — reserved punctuation
/[(){}]/v;
// SyntaxError: Invalid regular expression: /[(){}]/: Invalid character in character class

// Sabab: v flag'da ( ) { } [ ] / - \ | belgilar character class
// ichida escape qilinishi SHART — set operations uchun reserved
// ✅ v flag uchun escape
/[\(\)\{\}]/v; // OK

// ❌ u flag da ishlagan — v da yo'q
/[a-z&&]/u; // u flag: "a-z va &" — oddiy character class
/[a-z&&]/v; // v flag: SyntaxError — && intersection syntax reservation!

// ✅ v flag da escape qilish
/[a-z\&\&]/v; // OK, literal &&

// ❌ Subtraction/intersection bilan multi-char set — faqat string properties
/[\q{ab}--\q{a}]/v;        // ✅ \q{} string property — OK
/[[ab]--[a]]/v;             // ❌ character class ichida multi-char literal emas
// Sabab: v flag character class ichida faqat single characters va
// string properties (\q{...}, \p{RGI_Emoji}, va h.k.)

// ✅ Migration strategy — `u` kodini v ga o'tkazishda avval test
function upgradeRegex(pattern) {
  try {
    // Avval v flag bilan sinab ko'ring
    return new RegExp(pattern, "v");
  } catch (e) {
    if (e instanceof SyntaxError) {
      console.warn(`v flag bilan mos kelmaydi: ${pattern}`, e.message);
      return new RegExp(pattern, "u"); // fallback
    }
    throw e;
  }
}

// ❌ Umumiy xato: u va v ni birga ishlatish
/[a-z]/uv;
// SyntaxError: Invalid regular expression flags
// u va v MUTUALLY EXCLUSIVE
```

**Nima uchun:** `v` flag (Unicode Sets) ECMA-262 da yangi character class syntax kiritdi — set operations (`--`, `&&`) uchun punctuation'lar reserved qilindi. Bu backward-incompatible — ba'zi `u` pattern'lar `v` da invalid. Spec'da bu "strict mode" tariqasida atay qilingan — kelajak kengaytmalar uchun ambiguity'dan saqlanish.

**Yechim:** Codebase'da RegExp'larni `u`→`v` ga o'tkazayotganda har birini alohida test qiling. Statik pattern'lar uchun SyntaxError compile-time'da chiqadi. Dynamic pattern'lar (user input'dan) uchun har doim `try/catch` bilan o'rang.

---

## Amaliy Mashqlar

### Mashq 1: Object.groupBy bilan guruhlash (Oson)

**Savol:** Berilgan massivni yoshga qarab "yosh" (< 30) va "katta" (>= 30) guruhlariga ajrating.

```javascript
const people = [
  { name: "Ali", age: 25 },
  { name: "Vali", age: 35 },
  { name: "Soli", age: 28 },
  { name: "Karim", age: 42 },
];
```

<details>
<summary>Javob</summary>

```javascript
const groups = Object.groupBy(people, person =>
  person.age < 30 ? "yosh" : "katta"
);

// {
//   yosh: [{ name: "Ali", age: 25 }, { name: "Soli", age: 28 }],
//   katta: [{ name: "Vali", age: 35 }, { name: "Karim", age: 42 }]
// }
```

**Tushuntirish:** `Object.groupBy` callback natijasini key sifatida ishlatadi. Har bir element uchun "yosh" yoki "katta" qaytariladi — shunga qarab guruhlanadi.

</details>

---

### Mashq 2: Iterator helper'lar bilan pipeline (O'rta)

**Savol:** Cheksiz Fibonacci generator yarating. Undan birinchi 10 ta juft Fibonacci sonini oling.

<details>
<summary>Javob</summary>

```javascript
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const evenFibs = fibonacci()
  .filter(n => n % 2 === 0)  // faqat juft
  .drop(1)                    // 0 ni o'tkazib yuborish
  .take(10)                   // birinchi 10 ta
  .toArray();

// [2, 8, 34, 144, 610, 2584, 10946, 46368, 196418, 832040]
```

**Tushuntirish:** Fibonacci cheksiz generator — lekin iterator helper'lar lazy bo'lgani uchun faqat kerak bo'lgan elementlar hisoblandi. `filter` → `drop` → `take` pipeline'i birma-bir element o'tkazadi.

</details>

---

### Mashq 3: Set operations bilan ruxsat tekshirish (O'rta)

**Savol:** `hasAllPermissions(user, required)` va `getMissingPermissions(user, required)` funksiyalarini yozing. Set method'lar ishlatsin.

<details>
<summary>Javob</summary>

```javascript
function hasAllPermissions(userPerms, requiredPerms) {
  return requiredPerms.isSubsetOf(userPerms);
}

function getMissingPermissions(userPerms, requiredPerms) {
  return requiredPerms.difference(userPerms);
}

// Test
const user = new Set(["read", "write", "comment"]);
const required = new Set(["read", "write", "delete"]);

hasAllPermissions(user, required);
// false — "delete" yo'q

getMissingPermissions(user, required);
// Set {"delete"} — faqat "delete" yetishmayapti
```

**Tushuntirish:** `isSubsetOf` — barcha required permission'lar user'da bormi. `difference` — required'da bor, lekin user'da yo'q permission'larni topadi.

</details>

---

### Mashq 4: using bilan resource management (Qiyin)

**Savol:** `Lock` class yarating — `Symbol.dispose` implementatsiya qilsin. `using` bilan ishlatilganda avtomatik unlock bo'lsin.

<details>
<summary>Javob</summary>

```javascript
class Lock {
  #name;
  #locked = false;

  constructor(name) {
    this.#name = name;
    this.#locked = true;
    console.log(`Lock acquired: ${name}`);
  }

  get isLocked() {
    return this.#locked;
  }

  [Symbol.dispose]() {
    if (this.#locked) {
      this.#locked = false;
      console.log(`Lock released: ${this.#name}`);
    }
  }
}

function acquireLock(name) {
  return new Lock(name);
}

// Ishlatish
{
  using lock = acquireLock("database");
  console.log(lock.isLocked); // true
  // ... critical section ...
}
// Console: "Lock acquired: database"
//          true
//          "Lock released: database"

// Agar exception bo'lsa ham — dispose chaqiriladi
try {
  using lock = acquireLock("cache");
  throw new Error("Xatolik!");
} catch (e) {
  // Lock allaqachon released
}
// Console: "Lock acquired: cache"
//          "Lock released: cache"
```

**Tushuntirish:** `Symbol.dispose` — using scope tugaganda (normal yoki exception) avtomatik chaqiriladi. Bu resource leak'larni oldini oladi.

</details>

---

## Xulosa

| Feature | Versiya | Kategoriya | Asosiy foyda |
|---------|---------|-----------|-------------|
| `Object.groupBy` | ES2024 | Collection | reduce boilerplate o'rniga bir qator guruhlash |
| `Promise.withResolvers` | ES2024 | Async | Deferred pattern — resolve/reject tashqarida |
| `isWellFormed/toWellFormed` | ES2024 | String | Lone surrogate xavfsizligi |
| RegExp `v` flag | ES2024 | RegExp | Set operatsiyalari, Unicode property'lar |
| `Array.fromAsync` | ES2024 | Async | Async iterable → Array |
| `Atomics.waitAsync` | ES2024 | Concurrency | Non-blocking shared memory wait |
| Set methods | ES2025 | Collection | union, intersection, difference — to'plam matematikasi |
| Iterator Helpers | ES2025 | Iteration | Lazy map/filter/take — katta data uchun samarali |
| `using` / `await using` | ES2025 | Resource | Avtomatik resource cleanup — leak prevention |
| Import Attributes | ES2025 | Modules | JSON import xavfsizligi |
| RegExp modifiers | ES2025 | RegExp | Qisman flag control |
| Duplicate named groups | ES2025 | RegExp | Alternative branch'larda bir xil nom |
| Temporal | Stage 3 | Date | Date replacement — immutable, timezone-safe |
| Decorators | Stage 3 | Class | Meta-programming, AOP |

---

> **Kurs tugadi!** Bu 27 bo'limda JavaScript'ning barcha asosiy va advanced tushunchalari — engine ichidan zamonaviy standartlargacha — yoritildi. Interview tayyorgarlik uchun [interview/](interview/) papkasidagi savol-javoblarni ko'ring.
