# ES2024+ va Kelgusi Standartlar — Interview Savollari

> **Bo'lim 27** | Object.groupBy, Promise.withResolvers, Set methods, Iterator Helpers, using/await using, Import Attributes, Temporal, Decorators

---

## Savol 1: Object.groupBy() nima? reduce bilan farqi? [Junior+]

**Javob:**

`Object.groupBy()` — iterable elementlarini callback natijasiga qarab guruhlaydi. Har bir key uchun mos elementlar array'i bo'lgan object qaytaradi.

```javascript
const products = [
  { name: "Olma", type: "meva" },
  { name: "Sabzi", type: "sabzavot" },
  { name: "Banan", type: "meva" },
];

// ✅ Object.groupBy — bir qator
const grouped = Object.groupBy(products, p => p.type);
// {
//   meva: [{ name: "Olma", ... }, { name: "Banan", ... }],
//   sabzavot: [{ name: "Sabzi", ... }]
// }

// ❌ Eski usul — reduce bilan (ko'p boilerplate)
const oldGrouped = products.reduce((acc, p) => {
  (acc[p.type] ??= []).push(p);
  return acc;
}, {});
```

| | `Object.groupBy()` | `reduce` |
|--|---------------------|----------|
| Kod hajmi | 1 qator | 4-5 qator |
| O'qilishi | Oson | Murakkab |
| Natija | null-prototype object | Oddiy object |
| Standart | ES2024 | ES5+ |

**Muhim:** `Object.groupBy` natijasi — `Object.create(null)` — `Object.prototype` method'lari yo'q (`hasOwnProperty` ishlamaydi).

---

## Savol 2: Promise.withResolvers() nima? Qanday ishlatiladi? [Middle]

**Javob:**

`Promise.withResolvers()` — `{ promise, resolve, reject }` qaytaradi. Promise'ning `resolve`/`reject` funksiyalarini constructor tashqarisida olish imkonini beradi.

```javascript
// ❌ Eski usul — ko'p boilerplate
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});

// ✅ ES2024 — bir qator
const { promise, resolve, reject } = Promise.withResolvers();
```

Amaliy misol — event'ni await qilish:

```javascript
function waitForEvent(element, eventName) {
  const { promise, resolve } = Promise.withResolvers();
  element.addEventListener(eventName, resolve, { once: true });
  return promise;
}

// Ishlatish:
const event = await waitForEvent(button, "click");
```

Bu **Deferred pattern** — Promise'ni bir joyda yaratib, boshqa joyda resolve/reject qilish kerak bo'lganda ishlatiladi (event listener'lar, stream'lar, callback-based API'lar).

---

## Savol 3: Quyidagi kodning output'ini ayting [Middle]

```javascript
const nums = [1, 2, 3, 4, 5, 6];
const grouped = Object.groupBy(nums, n => n % 2 === 0 ? "even" : "odd");
console.log(grouped.even);
console.log(grouped.hasOwnProperty);
```

**Javob:**

```javascript
console.log(grouped.even);
// [2, 4, 6]

console.log(grouped.hasOwnProperty);
// undefined ❌
```

`Object.groupBy()` natijasi **null-prototype object** — `Object.create(null)` bilan yaratiladi. Shuning uchun `hasOwnProperty`, `toString`, `constructor` kabi `Object.prototype` method'lari yo'q.

```javascript
// ✅ To'g'ri tekshirish
"even" in grouped;             // true
Object.hasOwn(grouped, "even"); // true
```

---

## Savol 4: ES2025 Set method'lari haqida gapiring. Qanday method'lar bor? [Middle]

**Javob:**

ES2025 da `Set` ga 7 ta yangi method qo'shildi — matematik to'plam operatsiyalari:

```javascript
const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);

a.union(b);               // {1, 2, 3, 4, 5, 6} — birlashma
a.intersection(b);        // {3, 4} — kesishma
a.difference(b);          // {1, 2} — A da bor, B da yo'q
a.symmetricDifference(b); // {1, 2, 5, 6} — faqat bittasida

a.isSubsetOf(b);          // false — A ⊄ B
a.isSupersetOf(b);        // false — A ⊅ B
a.isDisjointFrom(b);      // false — umumiy element bor
```

| Method | Qaytaradi | Misol |
|--------|----------|-------|
| `union()` | Yangi Set | A ∪ B |
| `intersection()` | Yangi Set | A ∩ B |
| `difference()` | Yangi Set | A \ B |
| `symmetricDifference()` | Yangi Set | A △ B |
| `isSubsetOf()` | Boolean | A ⊆ B |
| `isSupersetOf()` | Boolean | A ⊇ B |
| `isDisjointFrom()` | Boolean | A ∩ B = ∅ |

Barcha method'lar asl Set'ni **mutate qilmaydi**.

---

## Savol 5: Iterator Helpers nima? Array method'lardan farqi? [Middle+]

**Javob:**

Iterator Helpers — `Iterator.prototype` ga qo'shilgan method'lar: `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `toArray`, `forEach`, `some`, `every`, `find`. Ularning asosiy farqi — **lazy evaluation**.

```javascript
// Array methods — EAGER (hammasi darhol bajariladi)
const result1 = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .map(n => n * 2)       // 10 ta element — yangi array
  .filter(n => n > 10)   // 10 ta tekshiruv — yangi array
  .slice(0, 3);          // 3 ta olish

// Iterator Helpers — LAZY (faqat kerak bo'lganda)
const result2 = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .values()
  .map(n => n * 2)       // hisob YO'Q — pipeline qurildi
  .filter(n => n > 10)   // hisob YO'Q
  .take(3)               // 3 ta topilguncha ishlaydi
  .toArray();             // [12, 14, 16] — SHU PAYTDA hisoblandi
```

| | Array methods | Iterator Helpers |
|--|--------------|-----------------|
| Evaluation | Eager | **Lazy** |
| Oraliq array | Har qadamda yangi | **Yo'q** |
| Cheksiz data | ❌ Ishlamaydi | ✅ Ishlaydi |
| `take(n)` | `slice(0, n)` | ✅ Built-in |
| `drop(n)` | `slice(n)` | ✅ Built-in |
| Qayerda | `Array.prototype` | `Iterator.prototype` |

```javascript
// Lazy ning kuchi — cheksiz generator bilan
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

// ❌ Array bilan cheksiz generator — crash (infinite loop)
// Array.from(naturals()).filter(...) // ❌ hech qachon tugamaydi

// ✅ Iterator Helpers — lazy, kerak bo'lganda to'xtaydi
naturals()
  .filter(n => n % 2 === 0)
  .take(5)
  .toArray();
// [2, 4, 6, 8, 10]
```

---

## Savol 6: using keyword nima? try/finally dan farqi? [Senior]

**Javob:**

`using` — resurslarni avtomatik tozalash uchun yangi syntax (ES2025). Resource object'da `Symbol.dispose` (sync) yoki `Symbol.asyncDispose` (async) bo'lishi kerak.

```javascript
// ❌ try/finally — ko'p boilerplate, xatolarga moyil
async function oldWay() {
  const db = await connectDB();
  try {
    const file = openFile("data.txt");
    try {
      await db.query("SELECT ...");
      file.write("result");
    } finally {
      file.close(); // fayl yopish
    }
  } finally {
    await db.close(); // connection yopish
  }
}

// ✅ using — ancha toza, xavfsiz
async function newWay() {
  await using db = await connectDB();
  using file = openFile("data.txt");

  await db.query("SELECT ...");
  file.write("result");
  // Scope tugaganda: avval file, keyin db — avtomatik yopiladi
  // Exception bo'lsa HAM yopiladi
}
```

Resource yaratish:

```javascript
class Connection {
  async [Symbol.asyncDispose]() {
    await this.close();
    console.log("Connection yopildi");
  }
}

class FileHandle {
  [Symbol.dispose]() {
    this.close();
    console.log("Fayl yopildi");
  }
}
```

| | `try/finally` | `using` |
|--|--------------|--------|
| Boilerplate | Ko'p | Kam |
| Nested resources | Nested try/finally | Flat — bir scope |
| Xato ehtimoli | finally'ni unutish | Avtomatik |
| Tartib | Qo'lda boshqarish | LIFO (teskari tartibda) |
| Exception safety | Qo'lda | Avtomatik (SuppressedError) |

**Deep Dive:** Agar main code xato bersa VA dispose ham xato bersa — `SuppressedError` yaratiladi: `{ error: dispose_xatosi, suppressed: asosiy_xato }`. Bu ikkala xatoni ham saqlaydi.

---

## Savol 7: Quyidagi kodning output'ini ayting [Middle+]

```javascript
function* gen() {
  yield 1;
  yield 2;
  yield 3;
  yield 4;
  yield 5;
}

const result = gen()
  .map(n => {
    console.log("map:", n);
    return n * 10;
  })
  .filter(n => {
    console.log("filter:", n);
    return n > 20;
  })
  .take(2)
  .toArray();

console.log(result);
```

**Javob:**

```
map: 1
filter: 10
map: 2
filter: 20
map: 3
filter: 30
map: 4
filter: 40
[30, 40]
```

Iterator Helper'lar **lazy** — element birma-bir pipeline'dan o'tadi:

1. `gen` → 1 → `map` → 10 → `filter` → 10 > 20? ❌ (o'tkazib yuborildi)
2. `gen` → 2 → `map` → 20 → `filter` → 20 > 20? ❌ (o'tkazib yuborildi)
3. `gen` → 3 → `map` → 30 → `filter` → 30 > 20? ✅ → `take` (1/2)
4. `gen` → 4 → `map` → 40 → `filter` → 40 > 20? ✅ → `take` (2/2) — **TO'XTADI**
5. `gen` → 5 — **HECH QACHON ISHLANMADI** (take 2 ta oldi, to'xtadi)

Bu eager Array method'lardan farqi — array'da barcha 5 ta element map va filter'dan o'tgan bo'lardi.

---

## Savol 8: Map.groupBy() va Object.groupBy() farqi nima? Qachon qaysi birini ishlatish kerak? [Middle+]

**Javob:**

```javascript
const users = [
  { name: "Ali", team: { id: 1, name: "Frontend" } },
  { name: "Vali", team: { id: 2, name: "Backend" } },
  { name: "Soli", team: { id: 1, name: "Frontend" } },
];

// Object.groupBy — key STRING bo'lishi kerak
const byTeamName = Object.groupBy(users, u => u.team.name);
// { Frontend: [...], Backend: [...] }

// Map.groupBy — key ISTALGAN qiymat bo'lishi mumkin
const teams = { frontend: { id: 1 }, backend: { id: 2 } };
const byTeamObj = Map.groupBy(users, u =>
  u.team.id === 1 ? teams.frontend : teams.backend
);
byTeamObj.get(teams.frontend); // [Ali, Soli]
byTeamObj.get(teams.backend);  // [Vali]
```

| | `Object.groupBy()` | `Map.groupBy()` |
|--|--------------------|-----------------|
| Natija | null-prototype Object | Map |
| Key turi | String/Symbol faqat | Istalgan (object, number...) |
| Key lookup | `grouped["key"]` | `grouped.get(key)` |
| Qachon | Key string bo'lganda | Key object/reference bo'lganda |

---

## Savol 9: Import Attributes nima? Nima uchun kerak? [Middle]

**Javob:**

Import Attributes — `import` ga qo'shimcha metadata berish. Asosan JSON va boshqa non-JS fayllarni xavfsiz import qilish uchun.

```javascript
// ✅ ES2025
import config from "./config.json" with { type: "json" };

// Dynamic import bilan
const data = await import("./data.json", { with: { type: "json" } });
```

**Nima uchun kerak?** Xavfsizlik:
- Server noto'g'ri Content-Type qaytarsa (JSON o'rniga `text/javascript`)
- `type: "json"` kafolatlaydi: fayl **faqat JSON** sifatida parse qilinadi
- JavaScript sifatida execute bo'lishiga **YO'L QO'YMAYDI**
- Supply-chain attack'lardan himoya

```javascript
// ⚠️ Eski syntax (deprecated)
import data from "./data.json" assert { type: "json" }; // ❌ eski (assert)
import data from "./data.json" with { type: "json" };    // ✅ yangi (with)
```

---

## Savol 10: isWellFormed() va toWellFormed() nima? Qachon kerak? [Middle+]

**Javob:**

JavaScript string'lari UTF-16. Ba'zan string'da **lone surrogate** — juftisiz surrogate code unit bo'lishi mumkin. Bu `encodeURIComponent()`, `TextEncoder`, `fetch` kabi API'larda xatolik yoki noto'g'ri natija beradi.

```javascript
// Lone surrogate
const bad = "test\uD800end";

bad.isWellFormed();   // false — lone surrogate bor
bad.toWellFormed();   // "test�end" — U+FFFD bilan almashtirdi

"hello".isWellFormed(); // true — problem yo'q
"😀".isWellFormed();    // true — to'g'ri surrogate pair

// Nima uchun kerak?
try {
  encodeURIComponent("\uD800"); // ❌ URIError!
} catch (e) {
  console.log("Xato:", e.message);
}

// ✅ Xavfsiz
encodeURIComponent("\uD800".toWellFormed()); // "%EF%BF%BD" — xavfsiz
```

Qachon ishlatiladi:
- Foydalanuvchi input'ini URL'ga qo'yishdan oldin
- `TextEncoder` bilan encode qilishdan oldin
- WebSocket/fetch orqali yuborishdan oldin

---

## Savol 11: Symbol.dispose va Symbol.asyncDispose farqi nima? [Senior]

**Javob:**

```javascript
// Symbol.dispose — sync resource uchun
class Lock {
  [Symbol.dispose]() {
    console.log("Lock released");
    // Sync tozalash
  }
}

// using — sync
{
  using lock = new Lock();
  // ...
} // "Lock released"

// Symbol.asyncDispose — async resource uchun
class DBConnection {
  async [Symbol.asyncDispose]() {
    await this.pool.end();
    console.log("DB connection closed");
  }
}

// await using — async
{
  await using db = new DBConnection();
  // ...
} // "DB connection closed"
```

| | `Symbol.dispose` | `Symbol.asyncDispose` |
|--|-----------------|----------------------|
| Keyword | `using` | `await using` |
| Dispose | Sync | Async (await) |
| Qaytaradi | void | Promise |
| Qachon | File, lock, timer | DB, network, stream |
| Kontekst | Istalgan joyda | Async function/module ichida |

Agar object'da ikkala Symbol ham bo'lsa — `await using` → `asyncDispose`, `using` → `dispose` chaqiradi.

---

## Savol 12: RegExp v flag u flag dan nima farq qiladi? [Senior]

**Javob:**

`v` flag — `u` ning kengaytirilgan versiyasi. Ikkalasini birga ishlatib bo'lmaydi.

```javascript
// u flag — basic Unicode support
/\p{Script=Latin}/u.test("a"); // true

// v flag — kengaytirilgan: set operations
// Difference (--)
/[\p{Emoji}--\p{ASCII}]/v.test("😀"); // true (ASCII bo'lmagan emoji)
/[\p{Emoji}--\p{ASCII}]/v.test("1");  // false

// Intersection (&&)
/[\p{Script=Latin}&&\p{Lowercase}]/v.test("a"); // true
/[\p{Script=Latin}&&\p{Lowercase}]/v.test("A"); // false

// String properties
/\p{RGI_Emoji_Flag_Sequence}/v.test("🇺🇿"); // true

// ❌ Birga ishlatib bo'lmaydi
// /test/uv → SyntaxError
```

| | `u` flag | `v` flag |
|--|---------|---------|
| Unicode property | ✅ | ✅ |
| Set difference `--` | ❌ | ✅ |
| Set intersection `&&` | ❌ | ✅ |
| Nested classes | ❌ | ✅ |
| String properties | ❌ | ✅ |
| Birga | ❌ `uv` xato | — |

---

## Savol 13: Array.fromAsync va Promise.all farqi nima? [Middle+]

**Javob:**

```javascript
const urls = ["/api/1", "/api/2", "/api/3"];

// Array.fromAsync — KETMA-KET (sequential)
const seq = await Array.fromAsync(urls, async url => {
  const res = await fetch(url);
  return res.json();
});
// /api/1 tugadi → /api/2 boshlanadi → /api/2 tugadi → /api/3 boshlanadi

// Promise.all — PARALLEL
const par = await Promise.all(urls.map(async url => {
  const res = await fetch(url);
  return res.json();
}));
// /api/1, /api/2, /api/3 — BARCHASI bir vaqtda boshlanadi
```

| | `Array.fromAsync` | `Promise.all` |
|--|-------------------|--------------|
| Bajarilish | **Sequential** | **Parallel** |
| Tezlik | Sekinroq | Tezroq |
| Xato | Birinchi xatoda to'xtaydi | Birinchi xatoda reject |
| Async iterable | ✅ | ❌ (faqat array of Promise) |
| Memory | Kam (birma-bir) | Ko'proq (barchasi bir vaqtda) |
| Qachon | Ketma-ket kerak bo'lganda, async iterable | Parallel kerak bo'lganda |

**Qoida:** Parallel imkoni bo'lsa — `Promise.all`. Ketma-ket kerak bo'lsa (rate limit, dependency) yoki async iterable bo'lsa — `Array.fromAsync`.

---

## Savol 14: TC39 process nima? Stage lar nima? [Junior+]

**Javob:**

TC39 — ECMAScript standartini boshqaradigan komitet. Har bir yangi feature quyidagi bosqichlardan o'tadi:

| Stage | Nomi | Ma'nosi | Ishlatish xavfsizmi? |
|-------|------|---------|---------------------|
| 0 | Strawperson | G'oya | ❌ Hali g'oya |
| 1 | Proposal | Muammo aniqlangan | ❌ O'zgarishi mumkin |
| 2 | Draft | Spec yozilgan | ⚠️ Syntax o'zgarishi mumkin |
| 2.7 | — | Spec tugallangan | ⚠️ Deyarli tayyor |
| 3 | Candidate | Browser'lar implement qilmoqda | ✅ Xavfsiz (polyfill bilan) |
| 4 | Finished | Standart! | ✅ Ishlating |

Har yil yanvar oyida Stage 4 ga yetganlar shu yilgi ECMAScript versiyasiga kiritiladi.

```
Misol: Object.groupBy() yoʻli
2021 — Stage 1 (proposal)
2023 — Stage 3 (browser'lar implement qilmoqda)
2024 — Stage 4 → ES2024 ga kiritildi
```

---

## Savol 15: Duplicate Named Capturing Groups nima? [Middle]

**Javob:**

ES2025 dan boshlab regex'da turli alternative branch'larda bir xil nomli capture group ishlatish mumkin.

```javascript
// ❌ Ilgari — har bir branch'da boshqa nom
const old = /(?<y>\d{4})-(?<m>\d{2})|(?<m2>\d{2})\/(?<y2>\d{4})/;

// ✅ ES2025 — bir xil nom
const dateRe = /(?<year>\d{4})-(?<month>\d{2})|(?<month>\d{2})\/(?<year>\d{4})/;

// ISO: 2024-03
dateRe.exec("2024-03").groups;
// { year: "2024", month: "03" }

// US: 03/2024
dateRe.exec("03/2024").groups;
// { year: "2024", month: "03" }

// Qaysi branch match bo'lsa — o'sha branch'dagi group qaytariladi
// Ikkalasida ham .groups.year va .groups.month ishlaydi
```

Bu turli formatdagi ma'lumotlarni parse qilishda juda foydali — bitta regex bilan bir nechta format'ni qo'llab-quvvatlash.

---

## Savol 16: DisposableStack nima? [Senior]

**Javob:**

`DisposableStack` — bir nechta disposable resource'ni birga boshqarish uchun container. `using` bilan ishlatilganda stack'dagi barcha resource'lar teskari tartibda (LIFO) dispose qilinadi.

```javascript
{
  using stack = new DisposableStack();

  const file1 = stack.use(openFile("a.txt"));
  const file2 = stack.use(openFile("b.txt"));
  const lock = stack.use(acquireLock("mutex"));

  // ... ishlash ...

  // Scope tugaganda teskari tartibda dispose:
  // 1. lock[Symbol.dispose]()
  // 2. file2[Symbol.dispose]()
  // 3. file1[Symbol.dispose]()
}

// AsyncDisposableStack — async versiya
async function process() {
  await using stack = new AsyncDisposableStack();

  const db = stack.use(await connectDB());
  const cache = stack.use(await connectRedis());

  // ... ishlash ...
  // Scope tugaganda: avval cache, keyin db — async dispose
}

// stack.defer() — callback qo'shish (Symbol.dispose bo'lmagan resource uchun)
{
  using stack = new DisposableStack();
  const timer = setInterval(() => {}, 1000);
  stack.defer(() => clearInterval(timer)); // dispose da clear qilinadi
}
```

`DisposableStack` method'lari:
- `use(resource)` — disposable resource qo'shadi, qaytaradi
- `adopt(value, onDispose)` — qiymat + cleanup callback
- `defer(callback)` — dispose vaqtida chaqiriladigan callback
- `move()` — resource'larni boshqa stack'ga ko'chiradi
