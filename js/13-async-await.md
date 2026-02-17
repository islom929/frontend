# Bo'lim 13: Async/Await

> Async/Await — Promise ustiga qurilgan syntactic sugar. Asinxron kodni sinxron ko'rinishda yozish imkonini beradi, lekin ichida hamma narsa Promise va microtask'lar bilan ishlaydi.

---

## Mundarija

- [Async Function Nima?](#async-function-nima)
- [Await Keyword](#await-keyword)
- [try/catch Bilan Error Handling](#trycatch-bilan-error-handling)
- [Parallel Execution — Promise.all + await](#parallel-execution--promiseall--await)
- [Sequential vs Parallel](#sequential-vs-parallel)
- [Top-Level Await (ES2022)](#top-level-await-es2022)
- [Async Patterns](#async-patterns)
  - [Retry Logic (Exponential Backoff)](#retry-logic-exponential-backoff)
  - [Timeout Pattern (Promise.race)](#timeout-pattern-promiserace)
  - [Concurrent Limit](#concurrent-limit)
  - [Queue Pattern](#queue-pattern)
- [for-await-of — Async Iteration](#for-await-of--async-iteration)
- [Under the Hood: Generator + Promise](#under-the-hood-generator--promise)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Async Function Nima?

### Nazariya

`async` keyword bilan belgilangan funksiya **doim Promise qaytaradi** — hatto siz oddiy qiymat return qilsangiz ham, u avtomatik `Promise.resolve()` ga o'raladi; `throw` qilsangiz esa `Promise.reject()` ga aylanadi. Bu shuni anglatadiki, async funksiya chaqirilganda natija hech qachon **bevosita** qaytmaydi — u doim Promise ichida keladi.

Async funksiya nima uchun yaratilgan? Promise chain'lari (`.then().then()`) callback hell'dan ancha yaxshi bo'lsa-da, murakkab asinxron logikada ular ham o'qilishi qiyin va debugging qiyinlashadi. `async/await` asinxron kodni **sinxron ko'rinishda** yozish imkonini beradi — lekin ichida hamma narsa Promise va microtask'lar bilan ishlaydi. Bu ES2017 da kiritilgan va zamonaviy JavaScript'da asinxron kodning **standart** yozish usuli hisoblanadi.

```javascript
// Oddiy funksiya
function getNumber() {
  return 42;
}
console.log(getNumber()); // 42

// Async funksiya — DOIM Promise qaytaradi
async function getNumberAsync() {
  return 42;
}
console.log(getNumberAsync()); // Promise {<fulfilled>: 42}

// Qiymatni olish uchun:
getNumberAsync().then(value => console.log(value)); // 42
```

Bu nimani anglatadi? Async funksiya chaqirilganda, natija hech qachon **bevosita** qaytmaydi — u doim Promise ichida keladi.

```javascript
// Hatto throw qilsak ham — rejected Promise qaytadi
async function willFail() {
  throw new Error("Xato!");
}
willFail().catch(err => console.log(err.message)); // "Xato!"

// Bu aslida quyidagicha:
function willFailDesugared() {
  return Promise.reject(new Error("Xato!"));
}
```

### Under the Hood

Async funksiya ichida nima sodir bo'ladi:

1. Funksiya chaqirilganda — **yangi Promise yaratiladi**
2. Funksiya tanasi (body) shu Promise ning **executor** sifatida ishlaydi
3. `return value` → `resolve(value)` ga aylanadi
4. `throw error` → `reject(error)` ga aylanadi
5. `await` uchrasa — funksiya **to'xtaydi** va boshqaruv chaqiruvchiga qaytadi

```javascript
// Biz yozamiz:
async function fetchUser() {
  const response = await fetch('/api/user');
  const data = await response.json();
  return data;
}

// Engine ko'radi (soddalashtirilgan):
function fetchUser() {
  return new Promise((resolve, reject) => {
    fetch('/api/user')
      .then(response => response.json())
      .then(data => resolve(data))
      .catch(err => reject(err));
  });
}
```

### Async Function Turlari

```javascript
// 1. Function declaration
async function fetchData() { /* ... */ }

// 2. Function expression
const fetchData = async function() { /* ... */ };

// 3. Arrow function
const fetchData = async () => { /* ... */ };

// 4. Method (object ichida)
const api = {
  async fetchData() { /* ... */ }
};

// 5. Class method
class ApiService {
  async fetchData() { /* ... */ }
}

// 6. IIFE (Immediately Invoked)
(async () => {
  const data = await fetchData();
  console.log(data);
})();
```

### Muhim Nuance: Promise Qaytarishda

```javascript
async function example() {
  // Agar Promise qaytarsangiz — ikki marta wrap qilinMAYDI
  return Promise.resolve(42);
}
// Natija: Promise {<fulfilled>: 42} — bitta Promise, ikkita emas!

// V8 buni optimallashtiradi — agar return value allaqachon Promise bo'lsa,
// yangi Promise yaratilmaydi (spec bo'yicha unwrap qilinadi)
```

---

## Await Keyword

### Nazariya

`await` — Promise resolve bo'lguncha funksiya bajarilishini **to'xtatadigan** (suspend) operator. Faqat `async` funksiya ichida yoki ES Module'ning top-level scope'ida ishlatiladi. `await` Promise'ni, thenable ob'ektni, yoki oddiy qiymatni qabul qiladi.

Eng muhim tushuncha: `await` funksiyani to'xtatadi, lekin **main thread ni BLOKLAMAYDI**. Funksiya to'xtaganda boshqaruv chaqiruvchiga qaytadi va Event Loop erkin qoladi. Promise resolve bo'lganda funksiya microtask orqali davom etadi. Shu sababli `await` dan keyingi kod aslida sinxron emas — u microtask sifatida bajariladi.

```javascript
async function demo() {
  console.log("1 — boshlanish");

  const result = await new Promise(resolve => {
    setTimeout(() => resolve("2 — await tugadi"), 1000);
  });

  console.log(result);     // "2 — await tugadi" (1 sekunddan keyin)
  console.log("3 — davom"); // await dan keyin — sinxron davom etadi
}

demo();
console.log("4 — tashqarida"); // Bu AVVAL chiqadi!

// Tartib: 1, 4, (1s keyin) 2, 3
```

### Await Nimani Kutadi?

```javascript
// 1. Promise kutadi — resolve yoki reject bo'lguncha
const data = await fetch('/api/data');

// 2. Thenable object — .then() metodi bor bo'lsa
const thenable = {
  then(resolve) {
    setTimeout(() => resolve(42), 1000);
  }
};
const val = await thenable; // 42 (1s keyin)

// 3. Non-Promise qiymat — darhol qaytaradi (Promise.resolve() orqali)
const num = await 42;       // 42 — darhol
const str = await "hello";  // "hello" — darhol
```

### Under the Hood: Await Qanday Ishlaydi

`await` uchrasa engine quyidagilarni qiladi:

```
async function example() {       ┌─────────────────────────────────┐
  const a = 1;                   │ 1. a = 1 — sinxron bajariladi   │
                                 └─────────────┬───────────────────┘
  const b = await fetch(url);                  │
       │                         ┌─────────────▼───────────────────┐
       │                         │ 2. fetch() chaqiriladi          │
       │                         │ 3. Promise qaytadi              │
       │                         │ 4. Funksiya SUSPEND bo'ladi     │
       │                         │ 5. Boshqaruv caller ga qaytadi  │
       │                         └─────────────┬───────────────────┘
       │                                       │ (Promise settle bo'lganda)
       │                         ┌─────────────▼───────────────────┐
       │                         │ 6. Microtask queue ga task qo'sh│
       │                         │ 7. Funksiya RESUME bo'ladi      │
       │                         │ 8. b = resolved value           │
       └─────────────────────── │ 9. Keyingi kodlar bajariladi    │
                                 └─────────────────────────────────┘
}
```

**Muhim:** `await` funksiyani to'xtatadi, lekin **main thread ni BLOCKLAMAS**. Event loop erkin qoladi — boshqa task'lar bajarilishi mumkin.

> **Eslatma:** Event loop haqida batafsil — [11-event-loop.md](11-event-loop.md), Promises haqida — [12-promises.md](12-promises.md)

### Await Ning Execution Tartibi

```javascript
async function order() {
  console.log("A"); // 1 — sinxron

  const x = await Promise.resolve("B");
  console.log(x);   // 3 — microtask

  console.log("C"); // 4 — await dan keyin sinxron (lekin aslida microtask ichida)
}

console.log("START"); // Eng birinchi
order();
console.log("END");   // 2 — order() await da to'xtadi

// Natija: START → A → END → B → C
```

Nima uchun `END` avval chiqadi? Chunki `await` funksiyani to'xtatib, **boshqaruvni chaqiruvchiga qaytaradi**. `order()` dan keyingi `console.log("END")` darhol bajariladi.

---

## try/catch Bilan Error Handling

### Nazariya

Async/await ning eng katta afzalliklaridan biri — error handling oddiy `try/catch` bilan ishlaydi, xuddi sinxron koddagidek. Promise'dagi `.catch()` zanjirlariga hojat yo'q — kod o'qilishi va tuzatilishi ancha osonlashadi.

Bundan tashqari, `try/catch` yordamida turli xato turlarini `instanceof` bilan ajratish, har bir `await` ni alohida try/catch da o'rash (granular error handling), yoki `await promise.catch()` pattern ishlatish mumkin. `finally` bloki esa resurslarni tozalash (spinner yashirish, connection yopish) uchun ishlatiladi — xuddi Promise'dagi `.finally()` kabi.

```javascript
// ❌ Promise bilan error handling — zanjir
function fetchUser(id) {
  return fetch(`/api/users/${id}`)
    .then(res => {
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    })
    .then(user => {
      return fetch(`/api/posts?userId=${user.id}`);
    })
    .then(res => res.json())
    .catch(err => {
      console.error("Xato:", err.message);
    });
}

// ✅ Async/await bilan — sinxron ko'rinishda
async function fetchUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);

    const user = await res.json();

    const postsRes = await fetch(`/api/posts?userId=${user.id}`);
    const posts = await postsRes.json();

    return { user, posts };
  } catch (err) {
    console.error("Xato:", err.message);
    // Xatoni qayta throw qilish yoki default qiymat qaytarish
    throw err;
  }
}
```

### Turli Xatolarni Ajratish

```javascript
async function processOrder(orderId) {
  try {
    const order = await fetchOrder(orderId);
    const payment = await processPayment(order);
    const shipment = await createShipment(order, payment);
    return shipment;
  } catch (err) {
    // Turli xatolarni turlicha handle qilish
    if (err instanceof NetworkError) {
      console.error("Tarmoq xatosi — qayta urinib ko'ring");
      return retry(() => processOrder(orderId));
    }
    if (err instanceof PaymentError) {
      console.error("To'lov xatosi:", err.message);
      await notifyUser(orderId, "To'lov amalga oshmadi");
      return null;
    }
    if (err instanceof ValidationError) {
      console.error("Validatsiya xatosi:", err.details);
      return { error: err.details };
    }
    // Kutilmagan xato
    throw err;
  }
}
```

### Har Bir Await ni Alohida Catch Qilish

Ba'zan har bir asinxron operatsiyani **alohida** handle qilish kerak:

```javascript
async function getUserData(userId) {
  // Har bir await o'z try/catch ichida
  let user;
  try {
    user = await fetchUser(userId);
  } catch (err) {
    console.error("User topilmadi:", err.message);
    return { error: "User not found" };
  }

  let posts;
  try {
    posts = await fetchPosts(userId);
  } catch (err) {
    console.warn("Postlar yuklanmadi, default ishlatamiz");
    posts = []; // graceful degradation
  }

  let notifications;
  try {
    notifications = await fetchNotifications(userId);
  } catch (err) {
    console.warn("Bildirishnomalar yuklanmadi");
    notifications = [];
  }

  return { user, posts, notifications };
}
```

### .catch() Alternativasi (try/catch siz)

```javascript
// await + .catch() — try/catch siz xatoni tutish
async function getUser(id) {
  const user = await fetchUser(id).catch(err => {
    console.error("Xato:", err);
    return null; // default qiymat
  });

  if (!user) return;

  // davom...
}
```

### Finally Block

```javascript
async function loadData() {
  const spinner = showSpinner();

  try {
    const data = await fetch('/api/data');
    return await data.json();
  } catch (err) {
    showError(err.message);
    return null;
  } finally {
    // Muvaffaqiyat bo'lsa ham, xato bo'lsa ham — DOIM ishlaydi
    hideSpinner(spinner);
  }
}
```

---

## Parallel Execution — Promise.all + await

### Nazariya

`await` ni ketma-ket ishlatish har bir operatsiya uchun oldingi tugashini kutadi. Agar operatsiyalar **bir-biriga bog'liq bo'lmasa** — bu behuda vaqt yo'qotish. `Promise.all()` bilan mustaqil operatsiyalarni **parallel** bajarish mumkin — natijada umumiy vaqt eng sekin operatsiya vaqtiga teng bo'ladi.

Eng muhim nuance: `Promise.all` **fail-fast** — bitta xato bo'lsa hammasi reject bo'ladi va muvaffaqiyatli natijalar yo'qoladi. Agar xatolarga chidamli bo'lish kerak bo'lsa — `Promise.allSettled` ishlatiladi, u barcha natijalarni (fulfilled va rejected) qaytaradi.

```javascript
// ❌ YOMON: ketma-ket — har biri oldingi tugashini kutadi
async function getPageData() {
  const user = await fetchUser();        // 1s kutadi
  const posts = await fetchPosts();      // yana 1s kutadi
  const comments = await fetchComments(); // yana 1s kutadi
  return { user, posts, comments };
  // Jami: ~3 sekund 😱
}

// ✅ YAXSHI: parallel — hammasi bir vaqtda boshlanadi
async function getPageData() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),        // |████| 1s
    fetchPosts(),       // |████| 1s
    fetchComments()     // |████| 1s
  ]);
  return { user, posts, comments };
  // Jami: ~1 sekund ✨ (eng sekin operatsiya vaqti)
}
```

### ASCII Diagram: Sequential vs Parallel

```
Sequential (ketma-ket):
──────────────────────────────────────────────────
Time:  0s        1s        2s        3s
       ├─────────┤
       fetchUser  ├─────────┤
                  fetchPosts ├─────────┤
                             fetchComments
Jami:  ═══════════════════════════════════ 3s

Parallel (Promise.all):
──────────────────────────────────────────────────
Time:  0s        1s
       ├─────────┤ fetchUser
       ├─────────┤ fetchPosts
       ├─────────┤ fetchComments
Jami:  ═══════════ 1s  (3x tezroq! 🚀)
```

### Promise.all — Fail Fast

`Promise.all` da **bitta xato** — hammasi reject bo'ladi:

```javascript
async function loadAll() {
  try {
    const [a, b, c] = await Promise.all([
      fetch('/api/a'),   // ✅ muvaffaqiyat
      fetch('/api/b'),   // ❌ xato — 500
      fetch('/api/c'),   // ✅ muvaffaqiyat (lekin natijasi yo'qoladi!)
    ]);
  } catch (err) {
    // Faqat BIRINCHI xato catch bo'ladi
    // b xato berdi — a va c natijalari YO'QOLDI
    console.error("Xato:", err);
  }
}
```

### Promise.allSettled — Hamma Natijani Olish

```javascript
async function loadAllSafe() {
  const results = await Promise.allSettled([
    fetch('/api/a'),
    fetch('/api/b'), // xato bersa ham
    fetch('/api/c'),
  ]);

  // Har bir natija: { status: 'fulfilled', value } yoki { status: 'rejected', reason }
  const successful = results
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value);

  const failed = results
    .filter(r => r.status === 'rejected')
    .map(r => r.reason);

  console.log(`${successful.length} muvaffaqiyat, ${failed.length} xato`);
  return { successful, failed };
}
```

### Kod Misollari: Real-World

```javascript
// Real-world: Dashboard uchun parallel data yuklash
async function loadDashboard(userId) {
  const startTime = performance.now();

  const [profile, orders, analytics, notifications] = await Promise.all([
    api.getProfile(userId),
    api.getOrders(userId, { limit: 10 }),
    api.getAnalytics(userId, { period: '30d' }),
    api.getNotifications(userId, { unread: true }),
  ]);

  const elapsed = performance.now() - startTime;
  console.log(`Dashboard yuklandi: ${elapsed.toFixed(0)}ms`);

  return { profile, orders, analytics, notifications };
}

// 🔍 Performance farqi:
// Sequential: 200ms + 300ms + 150ms + 100ms = 750ms
// Parallel:   max(200, 300, 150, 100) = 300ms — 2.5x tezroq!
```

---

## Sequential vs Parallel

### Nazariya

Sequential va parallel execution qachon ishlatishni bilish — async kodning **performance**ini belgilovchi eng muhim qaror. Qoida oddiy: keyingi operatsiya oldingi natijaga **bog'liq** bo'lsa — sequential; operatsiyalar **mustaqil** bo'lsa — parallel. Real-world da ko'pincha **aralash** pattern ishlatiladi: ba'zi operatsiyalar parallel, natijalar kelgandan keyin keyingi bosqich sequential davom etadi.

Bu farqni bilmaslik eng ko'p uchraydigan performance muammolardan biri. Masalan, 3 ta mustaqil API so'rovini ketma-ket qilish 3x sekinroq, parallel qilish esa tezlikni keskin oshiradi.

### Qaror Daraxti

```
Operatsiyalar bir-biriga bog'liqmi?
│
├─ HA → Sequential (ketma-ket await)
│   Masalan: userId olib → userId bilan postlarni olib → post bilan commentlarni ol
│
└─ YO'Q → Parallel (Promise.all)
    Masalan: user, posts, notifications — barchasi mustaqil
```

### Kod Misollari

```javascript
// ✅ Sequential — keyingisi oldingi natijaga BOG'LIQ
async function getOrderDetails(orderId) {
  // 1. Avval orderni olish kerak
  const order = await fetchOrder(orderId);

  // 2. Order ichidagi userId kerak
  const user = await fetchUser(order.userId);

  // 3. Order ichidagi productIds kerak
  const products = await Promise.all(
    order.productIds.map(id => fetchProduct(id))
  );
  // ☝️ Lekin productlar bir-biridan mustaqil — ularni parallel!

  return { order, user, products };
}
```

```javascript
// ❌ ANTI-PATTERN: mustaqil so'rovlar ketma-ket
async function getProfile(userId) {
  const user = await fetchUser(userId);       // 200ms
  const avatar = await fetchAvatar(userId);   // 150ms
  const settings = await fetchSettings(userId); // 100ms
  // Jami: 450ms — lekin nima uchun?! Ular mustaqil-ku!
  return { user, avatar, settings };
}

// ✅ TO'G'RI: mustaqil so'rovlarni parallel
async function getProfile(userId) {
  const [user, avatar, settings] = await Promise.all([
    fetchUser(userId),       // 200ms ─┐
    fetchAvatar(userId),     // 150ms ─┼─ parallel
    fetchSettings(userId),   // 100ms ─┘
  ]);
  // Jami: 200ms — 2x+ tezroq!
  return { user, avatar, settings };
}
```

### Aralash Pattern: Sequential + Parallel

```javascript
// Real-world: E-commerce checkout
async function checkout(cartId, userId) {
  // 1-qadam: Parallel — mustaqil ma'lumotlarni olish
  const [cart, user] = await Promise.all([
    fetchCart(cartId),
    fetchUser(userId),
  ]);

  // 2-qadam: Sequential — oldingi natijaga bog'liq
  const pricing = await calculatePricing(cart, user.discountLevel);

  // 3-qadam: Sequential — pricing kerak
  const payment = await processPayment(pricing.total, user.paymentMethod);

  // 4-qadam: Parallel — payment natijasiga bog'liq, lekin bir-biridan mustaqil
  const [receipt, shipment, notification] = await Promise.all([
    generateReceipt(payment),
    createShipment(cart, user.address),
    sendConfirmation(user.email, payment),
  ]);

  return { receipt, shipment, notification };
}
```

### ASCII Diagram: Aralash Pattern

```
Checkout Flow:
──────────────────────────────────────────────────────────
Time:  0        100ms    200ms    300ms    400ms    500ms

Step1: ├────────┤ fetchCart
       ├────────┤ fetchUser        (parallel)

Step2:          ├────────┤ calculatePricing (sequential — 1-qadamga bog'liq)

Step3:                   ├────────┤ processPayment (sequential — 2-qadamga bog'liq)

Step4:                            ├─────┤ generateReceipt
                                  ├─────┤ createShipment     (parallel)
                                  ├─────┤ sendConfirmation

       ════════════════════════════════════════════
       Jami: ~500ms (sequential bo'lganda ~800ms+ bo'lardi)
```

### Performance O'lchash

```javascript
// Performance farqini ko'rish uchun util
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function sequential() {
  console.time('sequential');
  await delay(1000);
  await delay(1000);
  await delay(1000);
  console.timeEnd('sequential'); // ~3000ms
}

async function parallel() {
  console.time('parallel');
  await Promise.all([
    delay(1000),
    delay(1000),
    delay(1000),
  ]);
  console.timeEnd('parallel'); // ~1000ms
}
```

---

## Top-Level Await (ES2022)

### Nazariya

ES2022 dan boshlab, **ES Modules** ichida `await` ni `async` funksiya siz ham ishlatish mumkin — bu **top-level await**. U modulning o'zini "async" qiladi — bu modulni import qilgan boshqa modullar ham uning tayyor bo'lishini kutadi.

Top-level await faqat ES Module'larda ishlaydi (`.mjs` yoki `"type": "module"` package.json da). CommonJS va oddiy script'larda ishlamaydi. U database connection, konfiguratsiya yuklash, dynamic import, va conditional module loading uchun juda qulay. Lekin ehtiyot bo'ling: top-level await sekin operatsiyani modulda ishlatish butun dependency tree'ni bloklab qo'yishi mumkin.

```javascript
// config.mjs (ES Module)
const response = await fetch('/api/config');
const config = await response.json();

export default config;
// Modul import qilinganda — config TAYYOR bo'ladi
```

### Qanday Ishlaydi?

```javascript
// database.mjs
const connection = await connectToDatabase();
export { connection };

// app.mjs
import { connection } from './database.mjs'; // Bu kutadi — database tayyor bo'lguncha

// Top-level await butun modulni "async" qiladi
// Import qilgan modul ham kutadi
```

### ASCII Diagram: Module Loading with Top-Level Await

```
Module Loading:
────────────────────────────────────────────────────

database.mjs:
├── connectToDatabase() ────────────────┤ (await)
                                        export { connection }

app.mjs:
├── import database ─── KUTISH ─────────┤ connection tayyor
                                        ├── import config ──┤
                                                            └── app boshlanadi

config.mjs:    (parallel yuklanyapti, lekin mustaqil)
├── fetch config ──┤ export config
```

### Muhim Qoidalar

```javascript
// ✅ Ishlaydi — ES Module (.mjs yoki "type": "module" package.json da)
// config.mjs
const data = await loadConfig();
export default data;

// ❌ ISHLAMAYDI — CommonJS (.cjs yoki oddiy .js)
// config.js
const data = await loadConfig(); // SyntaxError!

// ❌ ISHLAMAYDI — oddiy script (non-module)
// <script src="app.js"> ichida top-level await ISHLAMAYDI
// <script type="module"> kerak
```

### Use Cases

```javascript
// 1. Dynamic import
const locale = navigator.language;
const strings = await import(`./i18n/${locale}.mjs`);

// 2. Initialization
const db = await initDatabase();
export { db };

// 3. Fallback pattern
let jQuery;
try {
  jQuery = await import('https://cdn-a.example.com/jquery.mjs');
} catch {
  jQuery = await import('https://cdn-b.example.com/jquery.mjs');
}
export default jQuery;

// 4. Conditional module loading
const isProduction = process.env.NODE_ENV === 'production';
const logger = isProduction
  ? await import('./prod-logger.mjs')
  : await import('./dev-logger.mjs');
export default logger;
```

### Ogohlantirish

```javascript
// ⚠️ Top-level await modulni "bloklaydi" — ehtiyot bo'ling!
// Agar bu modul boshqa modullar tomonidan import qilinsa —
// BARCHASI shu await tugashini KUTADI.

// ❌ Yomon — sekin modul hamma narsani sekinlashtiradi
const hugeData = await fetchGigabyteOfData(); // 30 sekund
export { hugeData };
// Butun application 30 sekund bloklanadi!

// ✅ Yaxshi — lazy loading
let _cache;
export async function getHugeData() {
  if (!_cache) {
    _cache = await fetchGigabyteOfData();
  }
  return _cache;
}
```

---

## Async Patterns

Bu section da production-ready async patternlarni ko'rib chiqamiz.

### Retry Logic (Exponential Backoff)

#### Nazariya

Network so'rovlari muvaffaqiyatsiz bo'lganda qayta urinish — production dasturlarda eng ko'p ishlatiladigan patternlardan biri. **Exponential backoff** har safar kutish vaqtini 2x ga oshiradi (1s → 2s → 4s → 8s) — bu serverni ortiqcha yuklamaslik uchun muhim. Qo'shimcha **jitter** (random kechikish) esa barcha clientlar bir vaqtda retry qilmasligini ta'minlaydi.

```
Retry with Exponential Backoff:
───────────────────────────────────────────────────────────────

Urinish 1:   ├─── request ───┤ ❌ xato
              └── kutish: 1s ──┤

Urinish 2:                     ├─── request ───┤ ❌ xato
                                └── kutish: 2s ──────┤

Urinish 3:                                           ├─── request ───┤ ❌ xato
                                                      └── kutish: 4s ──────────┤

Urinish 4:                                                                     ├─── request ───┤ ✅ muvaffaqiyat!

Har safar kutish 2x oshadi: 1s → 2s → 4s → 8s → ...
+ Jitter (random qo'shimcha) — bir vaqtda barcha clientlar retry qilmaslik uchun
```

#### Kod

```javascript
async function retry(fn, options = {}) {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 30000,
    backoffFactor = 2,
    jitter = true,
    retryOn = () => true, // qaysi xatolarda retry qilish
    onRetry = () => {},    // har bir retry da callback
  } = options;

  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn(attempt);
    } catch (err) {
      lastError = err;

      // Oxirgi urinish bo'lsa — throw
      if (attempt === maxRetries) break;

      // Bu xatoni retry qilish kerakmi?
      if (!retryOn(err, attempt)) break;

      // Kutish vaqtini hisoblash
      let delay = Math.min(
        baseDelay * Math.pow(backoffFactor, attempt),
        maxDelay
      );

      // Jitter qo'shish (0.5x - 1.5x orasida random)
      if (jitter) {
        delay = delay * (0.5 + Math.random());
      }

      onRetry(err, attempt + 1, delay);

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

// Ishlatish:
const data = await retry(
  async (attempt) => {
    console.log(`Urinish ${attempt + 1}...`);
    const res = await fetch('https://api.example.com/data');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  },
  {
    maxRetries: 4,
    baseDelay: 1000,
    retryOn: (err) => {
      // Faqat network va 5xx xatolarda retry
      return err.message.includes('fetch') || err.message.includes('5');
    },
    onRetry: (err, attempt, delay) => {
      console.log(`Xato: ${err.message}. ${attempt}-urinish ${delay}ms dan keyin...`);
    },
  }
);
```

---

### Timeout Pattern (Promise.race)

#### Nazariya

Asinxron operatsiyaga vaqt chegarasi qo'yish — `Promise.race()` bilan bajariladigan muhim pattern. Agar operatsiya belgilangan vaqt ichida tugamasa, timeout xatosi qaytariladi. Bu ayniqsa tashqi API'lar bilan ishlashda muhim — javob kelmasa dastur cheksiz kutmasligi kerak.

```javascript
function withTimeout(promise, ms, message = 'Operation timed out') {
  const timeout = new Promise((_, reject) => {
    const id = setTimeout(() => {
      clearTimeout(id);
      reject(new Error(message));
    }, ms);
  });

  return Promise.race([promise, timeout]);
}

// Ishlatish:
async function fetchWithTimeout() {
  try {
    const data = await withTimeout(
      fetch('https://slow-api.example.com/data'),
      5000, // 5 sekund limit
      'API javob bermadi — 5s timeout'
    );
    return await data.json();
  } catch (err) {
    if (err.message.includes('timeout')) {
      console.error('API juda sekin!');
      return getCachedData(); // fallback
    }
    throw err;
  }
}
```

#### AbortController Bilan (To'g'ri Usul)

```javascript
// ✅ AbortController — so'rovni to'xtatadi (network resurs tejaydi)
async function fetchWithAbort(url, timeoutMs = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return await response.json();
  } catch (err) {
    if (err.name === 'AbortError') {
      throw new Error(`Request timed out after ${timeoutMs}ms`);
    }
    throw err;
  } finally {
    clearTimeout(timeoutId);
  }
}

// 🔍 Farq: Promise.race timeout da promise hali ishlayapti (resurs sarflaydi)
//          AbortController esa so'rovni TO'XTATADI — resurs tejaydi
```

---

### Concurrent Limit

#### Nazariya

1000 ta URL bor — hammasini bir vaqtda `Promise.all` qilsak, server 1000 ta parallel so'rov ko'radi va bu DDoS hisoblanadi. **Concurrent limit** — bir vaqtda faqat N ta so'rov yuborib, biri tugashi bilan navbatdagini boshlash. Bu server va client resurslarini tejaydi, rate-limiting'ga tushishdan saqlaydi.

```
Concurrent Limit (max 3):
──────────────────────────────────────────────────────────────

Slot 1: ├── req1 ──┤ ├── req4 ──────┤ ├── req7 ──┤ ├── req9 ───┤
Slot 2: ├── req2 ────────┤ ├── req5 ──┤ ├── req8 ────┤ ├── req10 ──┤
Slot 3: ├── req3 ──┤ ├── req6 ──────────┤

         Har doim max 3 ta so'rov parallel ishlaydi
         Biri tugasa — navbatdagi boshlanadi
```

#### Kod

```javascript
async function concurrentLimit(tasks, limit) {
  const results = [];
  const executing = new Set();

  for (const [index, task] of tasks.entries()) {
    // Task ni boshlash
    const promise = Promise.resolve().then(() => task());

    results[index] = promise;
    executing.add(promise);

    // Task tugaganda Set dan olib tashlash
    const cleanup = () => executing.delete(promise);
    promise.then(cleanup, cleanup);

    // Agar limit ga yetgan bo'lsa — bittasi tugashini kutish
    if (executing.size >= limit) {
      await Promise.race(executing);
    }
  }

  return Promise.all(results);
}

// Ishlatish:
const urls = Array.from({ length: 100 }, (_, i) => `https://api.example.com/item/${i}`);

const tasks = urls.map(url => () => fetch(url).then(r => r.json()));

// Bir vaqtda faqat 5 ta so'rov
const results = await concurrentLimit(tasks, 5);
console.log(`${results.length} ta natija olindi`);
```

#### Muqobil: p-limit Pattern

```javascript
// p-limit — sodda va ishonchli
function pLimit(concurrency) {
  const queue = [];
  let activeCount = 0;

  function next() {
    activeCount--;
    if (queue.length > 0) {
      queue.shift()();
    }
  }

  function run(fn, resolve, reject) {
    activeCount++;
    fn().then(resolve, reject).finally(next);
  }

  function enqueue(fn) {
    return new Promise((resolve, reject) => {
      if (activeCount < concurrency) {
        run(fn, resolve, reject);
      } else {
        queue.push(() => run(fn, resolve, reject));
      }
    });
  }

  return (fn) => enqueue(fn);
}

// Ishlatish:
const limit = pLimit(3); // max 3 ta parallel

const results = await Promise.all(
  urls.map(url =>
    limit(() => fetch(url).then(r => r.json()))
  )
);
```

---

### Queue Pattern

#### Nazariya

Queue pattern — task'lar ketma-ket (bitta-bitta) bajarilishi kerak bo'lganda ishlatiladigan pattern. Database yozish operatsiyalari, file system o'zgarishlari, yoki tartib muhim bo'lgan tranzaktsiyalar uchun ideal. U yangi task'larni navbatga qo'yadi va har bir task'ni oldingi tugagandan keyin bajaradi.

```javascript
class AsyncQueue {
  #queue = [];
  #processing = false;

  async enqueue(task) {
    return new Promise((resolve, reject) => {
      this.#queue.push({ task, resolve, reject });
      this.#process();
    });
  }

  async #process() {
    if (this.#processing) return; // Allaqachon ishlayapti
    this.#processing = true;

    while (this.#queue.length > 0) {
      const { task, resolve, reject } = this.#queue.shift();
      try {
        const result = await task();
        resolve(result);
      } catch (err) {
        reject(err);
      }
    }

    this.#processing = false;
  }

  get size() {
    return this.#queue.length;
  }
}

// Ishlatish:
const writeQueue = new AsyncQueue();

// Bu so'rovlar har qanday tartibda kelishi mumkin,
// lekin faylga KETMA-KET yoziladi
async function handleRequest(data) {
  const result = await writeQueue.enqueue(async () => {
    await writeToFile(data);
    return { success: true, timestamp: Date.now() };
  });
  return result;
}

// Parallel chaqirilsa ham — ketma-ket bajariladi
await Promise.all([
  handleRequest("data1"), // 1-chi
  handleRequest("data2"), // 2-chi (1-chi tugagandan keyin)
  handleRequest("data3"), // 3-chi (2-chi tugagandan keyin)
]);
```

#### Priority Queue

```javascript
class PriorityAsyncQueue {
  #queue = [];    // { task, resolve, reject, priority }
  #processing = false;

  async enqueue(task, priority = 0) {
    return new Promise((resolve, reject) => {
      this.#queue.push({ task, resolve, reject, priority });
      // Priority bo'yicha sort (katta = muhimroq)
      this.#queue.sort((a, b) => b.priority - a.priority);
      this.#process();
    });
  }

  async #process() {
    if (this.#processing) return;
    this.#processing = true;

    while (this.#queue.length > 0) {
      const { task, resolve, reject } = this.#queue.shift();
      try {
        resolve(await task());
      } catch (err) {
        reject(err);
      }
    }

    this.#processing = false;
  }
}

// Ishlatish:
const queue = new PriorityAsyncQueue();

queue.enqueue(() => sendEmail(user), 1);        // past priority
queue.enqueue(() => processPayment(order), 10); // yuqori priority — avval bajariladi
queue.enqueue(() => logAnalytics(event), 0);    // eng past
```

---

## for-await-of — Async Iteration

### Nazariya

`for-await-of` — asinxron iterable (async iterator) ustida iteratsiya qilish uchun maxsus tsikl. Stream, paginated API, va real-time data kabi scenariolarda ishlatiladi. U oddiy `for-of` ning asinxron versiyasi bo'lib, har bir iteratsiyada Promise resolve bo'lishini kutadi.

Bu pattern ayniqsa katta ma'lumotlarni bo'laklab o'qish (streaming), sahifalangan API natijalarini ketma-ket olish, yoki WebSocket/Server-Sent Events kabi real-time ma'lumot oqimlarini qayta ishlash uchun muhim.

> **Eslatma:** Iterator va Generator haqida batafsil — [14-iterators-generators.md](14-iterators-generators.md)

```javascript
// Oddiy for-of — sinxron iterable
for (const item of [1, 2, 3]) {
  console.log(item);
}

// for-await-of — asinxron iterable
async function processStream() {
  const stream = getAsyncStream();

  for await (const chunk of stream) {
    console.log("Chunk olindi:", chunk);
    await processChunk(chunk);
  }
}
```

### Async Generator

```javascript
// Async generator — yield + await bir joyda
async function* fetchPages(baseUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${baseUrl}?page=${page}`);
    const data = await response.json();

    yield data.items; // Har bir sahifani yield qilish

    hasMore = data.hasNextPage;
    page++;
  }
}

// Ishlatish:
async function getAllItems() {
  const allItems = [];

  for await (const items of fetchPages('https://api.example.com/products')) {
    allItems.push(...items);
    console.log(`${allItems.length} ta mahsulot yuklandi...`);
  }

  return allItems;
}
```

### Real-World: Paginated API

```javascript
// Paginated API — cursor-based pagination
async function* paginatedFetch(url, options = {}) {
  let cursor = null;

  do {
    const params = new URLSearchParams({ limit: '100' });
    if (cursor) params.set('cursor', cursor);

    const response = await fetch(`${url}?${params}`, options);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data = await response.json();
    cursor = data.nextCursor;

    yield data.results;
  } while (cursor);
}

// Ishlatish:
for await (const batch of paginatedFetch('https://api.example.com/users')) {
  for (const user of batch) {
    await processUser(user);
  }
}
```

### Real-World: Node.js Stream

```javascript
// Node.js readable stream — for-await-of bilan
import { createReadStream } from 'fs';

async function readFileChunked(filePath) {
  const stream = createReadStream(filePath, { encoding: 'utf-8' });
  let lineCount = 0;

  for await (const chunk of stream) {
    lineCount += chunk.split('\n').length;
  }

  console.log(`Faylda ${lineCount} qator bor`);
}
```

### Custom Async Iterable

```javascript
// Custom async iterable class
class EventPoller {
  #url;
  #intervalMs;
  #running = false;

  constructor(url, intervalMs = 1000) {
    this.#url = url;
    this.#intervalMs = intervalMs;
  }

  // Symbol.asyncIterator — for-await-of uchun
  async *[Symbol.asyncIterator]() {
    this.#running = true;

    while (this.#running) {
      try {
        const response = await fetch(this.#url);
        const events = await response.json();

        for (const event of events) {
          yield event; // Har bir event ni yield
        }
      } catch (err) {
        console.error('Polling xato:', err);
      }

      await new Promise(r => setTimeout(r, this.#intervalMs));
    }
  }

  stop() {
    this.#running = false;
  }
}

// Ishlatish:
const poller = new EventPoller('https://api.example.com/events', 5000);

for await (const event of poller) {
  console.log('Yangi event:', event);

  if (event.type === 'shutdown') {
    poller.stop();
  }
}
```

---

## Under the Hood: Generator + Promise

### Nazariya

`async/await` aslida **generator + promise** ning syntactic sugar versiyasi. ES2017 dan oldin Babel va boshqa transpiler'lar async/await ni generator+promise ga aylantirar edi. Generator'ning `yield` operatori funksiyani to'xtatadi, Promise resolve bo'lganda davom ettiradi — aynan `await` qilgan ishni bajaradi.

Bu ichki mexanizmni tushunish `async/await` ning nega microtask orqali ishlashini, nima uchun `await` dan keyingi kod darhol bajarilmasligini, va debug paytida call stack'da generator frame'larini ko'rish sabablarini tushuntirib beradi.

### Async/Await Qanday Desugar Bo'ladi

```javascript
// Biz yozamiz:
async function fetchUserPosts(userId) {
  const user = await fetch(`/api/users/${userId}`);
  const data = await user.json();
  const posts = await fetch(`/api/posts?userId=${data.id}`);
  return posts.json();
}

// Transpiler (Babel) buni generator + promise ga aylantiradi:
function fetchUserPosts(userId) {
  return spawn(function* () {
    const user = yield fetch(`/api/users/${userId}`);
    const data = yield user.json();
    const posts = yield fetch(`/api/posts?userId=${data.id}`);
    return yield posts.json();
  });
}

// spawn — generator ni boshqaruvchi funksiya:
function spawn(generatorFn) {
  return new Promise((resolve, reject) => {
    const generator = generatorFn();

    function step(nextFn) {
      let result;
      try {
        result = nextFn();
      } catch (err) {
        return reject(err);
      }

      if (result.done) {
        return resolve(result.value);
      }

      // yield qilingan qiymatni Promise ga o'rab, resolve bo'lganda davom
      Promise.resolve(result.value).then(
        value => step(() => generator.next(value)),
        err => step(() => generator.throw(err))
      );
    }

    step(() => generator.next());
  });
}
```

### ASCII Diagram: Desugared Flow

```
async/await (biz yozamiz):         generator + promise (engine ko'radi):
─────────────────────────────       ──────────────────────────────────────

async function foo() {              function foo() {
│                                     return spawn(function* () {
│  const a = await fetch(url1);  →      const a = yield fetch(url1);
│         │                                      │
│         │ [suspend]                             │ [yield — generator to'xtaydi]
│         │                                       │
│         │ [resume: a = result]                  │ [.next(result) — davom]
│         ▼                                       ▼
│  const b = await fetch(url2);  →      const b = yield fetch(url2);
│         │                                      │
│         │ [suspend]                             │ [yield]
│         │                                       │
│         │ [resume: b = result]                  │ [.next(result)]
│         ▼                                       ▼
│  return b;                     →      return b;
│                                     });
}                                   }

await   ↔  yield    (to'xtash)
resume  ↔  .next()  (davom etish)
reject  ↔  .throw() (xato tashish)
```

### V8 Optimization: Zero-Cost Async/Await

V8 v7.2+ (Node.js 12+, Chrome 72+) dan boshlab `async/await` oldingi versiyalarga qaraganda ancha tezroq. Sabab — **extra microtick** yo'qoldi.

```javascript
// Oldin (V8 < 7.2): 3 ta microtick kerak edi
// await har safar qo'shimcha Promise yaratardi

// Hozir (V8 >= 7.2): faqat 1 ta microtick
// Agar await qilingan qiymat allaqachon Promise bo'lsa —
// V8 uni to'g'ridan-to'g'ri ishlatadi, qo'shimcha wrap QILMAYDI

async function example() {
  // V8 7.2+: bu faqat 1 microtick oladi
  const result = await Promise.resolve(42);
  return result;
}

// Nima o'zgardi texnik jihatdan?
// Oldin:  await → PromiseResolve(promise) → yangi Promise → 2+ microtick
// Hozir:  await → promise allaqachon? → to'g'ridan-to'g'ri kutish → 1 microtick
```

```
V8 < 7.2 (sekin):                    V8 >= 7.2 (tez):
─────────────────                    ─────────────────

await somePromise                    await somePromise
     │                                    │
     ├─ PromiseResolve()                  ├─ somePromise allaqachon
     │    (yangi Promise yaratish)        │   resolved mi? → HA
     │                                    │
     ├─ microtick 1: resolve              ├─ microtick 1: resume
     ├─ microtick 2: handler              │   (to'g'ridan-to'g'ri!)
     ├─ microtick 3: resume               │
     │                                    │
     ▼                                    ▼
     Natija                               Natija

     3 microtick                          1 microtick ← 3x tezroq!
```

### Generator bilan Async Taqqoslash

```javascript
// 1. Generator pattern (eski usul — Babel transpile natijasi)
function* fetchDataGen() {
  try {
    const response = yield fetch('/api/data');
    const json = yield response.json();
    return json;
  } catch (err) {
    console.error(err);
  }
}

// 2. Async/Await (zamonaviy — ancha toza)
async function fetchDataAsync() {
  try {
    const response = await fetch('/api/data');
    const json = await response.json();
    return json;
  } catch (err) {
    console.error(err);
  }
}

// Farq:
// - async/await: native engine support, tezroq, stack trace yaxshiroq
// - generator: manual runner kerak (spawn/co), sekinroq, stack trace murakkab
// - Bugungi kunda generator pattern shart emas — async/await ishlatamiz
```

---

## Common Mistakes

### ❌ Xato 1: Keraksiz Ketma-Ket Await (Parallel Bo'lishi Kerak Edi)

```javascript
// ❌ Yomon — mustaqil so'rovlarni ketma-ket bajaryapti
async function getUserPage(userId) {
  const user = await fetchUser(userId);           // 300ms
  const notifications = await fetchNotifications(userId); // 200ms
  const feed = await fetchFeed(userId);           // 400ms
  return { user, notifications, feed };
  // Jami: 900ms 😱
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Yaxshi — mustaqil so'rovlarni parallel
async function getUserPage(userId) {
  const [user, notifications, feed] = await Promise.all([
    fetchUser(userId),           // 300ms ─┐
    fetchNotifications(userId),  // 200ms ─┼─ parallel
    fetchFeed(userId),           // 400ms ─┘
  ]);
  return { user, notifications, feed };
  // Jami: 400ms (eng sekin operatsiya vaqti) ✨
}
```

**Nima uchun:** Mustaqil operatsiyalarni ketma-ket bajarish vaqtni behuda sarflaydi. `Promise.all` ularni bir vaqtda boshlaydi — jami vaqt eng sekin operatsiyaga teng bo'ladi.

---

### ❌ Xato 2: forEach Ichida await (ISHLAMAYDI!)

```javascript
// ❌ Yomon — forEach callback ni KUTMAYDI
const userIds = [1, 2, 3, 4, 5];

userIds.forEach(async (id) => {
  const user = await fetchUser(id);
  console.log(user.name);
});

console.log("Tayyor!"); // Bu AVVAL chiqadi! forEach kutmaydi!
// forEach sinxron — async callback ni "ot va unut" qiladi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Variant 1: for...of — ketma-ket bajarish
for (const id of userIds) {
  const user = await fetchUser(id);
  console.log(user.name);
}
console.log("Tayyor!"); // Haqiqatan ham tayyor

// ✅ Variant 2: Promise.all + map — parallel bajarish
const users = await Promise.all(
  userIds.map(id => fetchUser(id))
);
users.forEach(user => console.log(user.name));
console.log("Tayyor!");
```

**Nima uchun:** `forEach` return qiymatini e'tiborsiz qoldiradi — async callback qaytargan Promise ni hech kim `await` qilmayapti. Natijada barcha so'rovlar "fire and forget" bo'ladi. `for...of` yoki `Promise.all` + `map` ishlatamiz.

---

### ❌ Xato 3: Loop Ichida await (Parallel Pattern Kerak)

```javascript
// ❌ Yomon — har bir so'rov oldingi tugashini kutadi
async function fetchAllUsers(ids) {
  const users = [];
  for (const id of ids) {
    const user = await fetchUser(id); // KETMA-KET — sekin!
    users.push(user);
  }
  return users;
  // 100 ta user × 200ms = 20 sekund 😱
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Yaxshi — barcha so'rovlarni parallel
async function fetchAllUsers(ids) {
  const users = await Promise.all(
    ids.map(id => fetchUser(id))
  );
  return users;
  // 100 ta user, lekin parallel = ~200ms ✨
}

// ✅ Yaxshiroq — concurrent limit bilan (server ni bosmaslik uchun)
async function fetchAllUsers(ids) {
  const limit = pLimit(10); // max 10 ta parallel
  const users = await Promise.all(
    ids.map(id => limit(() => fetchUser(id)))
  );
  return users;
  // 100 ta user, 10 ta parallel = ~2 sekund (server xavfsiz)
}
```

**Nima uchun:** Loop ichida `await` har bir iteratsiyani ketma-ket bajaradi. Agar so'rovlar mustaqil bo'lsa — `Promise.all` yoki concurrent limit ishlatamiz. Server himoyasi uchun concurrent limit eng yaxshi amaliyot.

---

### ❌ Xato 4: Error Handling Yo'q (Unhandled Rejection)

```javascript
// ❌ Yomon — xato catch qilinmagan
async function loadData() {
  const response = await fetch('/api/data'); // Network xato bo'lsa?
  const data = await response.json();        // Invalid JSON bo'lsa?
  return data;
}

// Agar loadData() ning qaytgan Promise ni catch qilmasangiz:
loadData(); // ⚠️ UnhandledPromiseRejection — dastur crash bo'lishi mumkin!
```

### ✅ To'g'ri usul:

```javascript
// ✅ Variant 1: try/catch ichida
async function loadData() {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return await response.json();
  } catch (err) {
    console.error('Data yuklashda xato:', err.message);
    return null; // yoki default qiymat, yoki throw
  }
}

// ✅ Variant 2: chaqirgan joyda catch
loadData()
  .then(data => render(data))
  .catch(err => showError(err));

// ✅ Variant 3: global handler (oxirgi himoya)
// Node.js:
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  // Log, alert, graceful shutdown
});
// Browser:
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled Rejection:', event.reason);
  event.preventDefault();
});
```

**Nima uchun:** Catch qilinmagan Promise rejection — dasturni crash qilishi mumkin (Node.js 15+ da default behavior). Doim error handling qo'shing — try/catch, .catch(), yoki global handler.

---

### ❌ Xato 5: Async Funksiyani Await Qilmaslik

```javascript
// ❌ Yomon — async funksiya chaqirildi, lekin await QILINMADI
async function saveToDatabase(data) {
  await db.insert(data);
  console.log("Saqlandi!");
}

async function handleRequest(req) {
  const data = parseRequest(req);
  saveToDatabase(data); // ⚠️ await YO'Q! — "fire and forget"
  // Xato bo'lsa — hech kim bilmaydi
  // Javob qaytariladi — lekin database ga yozish tugamagan bo'lishi mumkin
  return { success: true };
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ To'g'ri — await bilan kutish
async function handleRequest(req) {
  const data = parseRequest(req);
  await saveToDatabase(data); // ✅ Kutamiz — xato bo'lsa catch qilamiz
  return { success: true };
}

// Agar haqiqatan ham "fire and forget" kerak bo'lsa — hech bo'lmasa error log:
async function handleRequest(req) {
  const data = parseRequest(req);
  saveToDatabase(data).catch(err => {
    console.error("Background save failed:", err);
    // Alert, retry, yoki monitoring
  });
  return { success: true }; // Javobni kutmasdan qaytarish
}
```

**Nima uchun:** Async funksiyani `await` siz chaqirish — uning natijasini va xatolarini yo'qotish. Dastur "muvaffaqiyat" deb o'ylaydi, lekin database ga hech narsa yozilmagan bo'lishi mumkin.

---

### ❌ Xato 6: return await — Keraksiz Await

```javascript
// ❌ Ortiqcha — return await kerak emas (try/catch bo'lmasa)
async function getUser(id) {
  return await fetchUser(id); // await kerak emas — Promise shu holda qaytadi
}

// ✅ To'g'ri (try/catch bo'lmasa):
async function getUser(id) {
  return fetchUser(id); // Promise to'g'ridan-to'g'ri qaytadi
}

// ⚠️ LEKIN: try/catch ichida return await KERAK!
async function getUser(id) {
  try {
    return await fetchUser(id); // ✅ await KERAK — aks holda catch ishlamaydi
  } catch (err) {
    console.error("Xato:", err);
    return null;
  }
}

// ❌ Agar await qilmasangiz:
async function getUser(id) {
  try {
    return fetchUser(id); // ❌ Xato catch QILINMAYDI!
    // Promise shu holda qaytadi — try/catch ni "chetlab o'tadi"
  } catch (err) {
    console.error("Bu hech qachon ishlamaydi");
  }
}
```

**Nima uchun:** `return await` try/catch bo'lmasa keraksiz — Promise shu holda qaytaveradi. Lekin try/catch ichida **majburiy** — aks holda reject bo'lgan Promise catch blokini chetlab o'tadi.

---

### ❌ Xato 7: Async Constructor

```javascript
// ❌ ISHLAMAYDI — constructor async bo'la olmaydi
class Database {
  async constructor(url) { // ❌ SyntaxError!
    this.connection = await connect(url);
  }
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Factory pattern — static async method
class Database {
  #connection;

  constructor(connection) {
    this.#connection = connection;
  }

  static async create(url) {
    const connection = await connect(url);
    return new Database(connection);
  }

  async query(sql) {
    return this.#connection.execute(sql);
  }
}

// Ishlatish:
const db = await Database.create('postgres://localhost/mydb');
const users = await db.query('SELECT * FROM users');
```

**Nima uchun:** Constructor doim sinxron bo'lishi kerak va `this` qaytarishi kerak — Promise emas. Factory pattern (static async method) eng to'g'ri yechim.

---

## Amaliy Mashqlar

### Mashq 1: Retry with Exponential Backoff (O'rta)

**Savol:** `retryWithBackoff(fn, maxRetries, baseDelay)` funksiyasini yozing:
- `fn` — async funksiya
- Xato bo'lsa qayta urinish
- Har safar kutish vaqti 2x oshadi
- `maxRetries` dan keyin oxirgi xatoni throw qilish

```javascript
// Test:
const result = await retryWithBackoff(
  async () => {
    const res = await fetch('https://unreliable-api.example.com/data');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  },
  3,    // max 3 ta retry
  1000  // 1s dan boshlab: 1s, 2s, 4s
);
```

<details>
<summary>Javob</summary>

```javascript
async function retryWithBackoff(fn, maxRetries = 3, baseDelay = 1000) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      console.log(`Urinish ${attempt + 1} muvaffaqiyatsiz: ${err.message}`);

      if (attempt < maxRetries) {
        const delay = baseDelay * Math.pow(2, attempt);
        console.log(`${delay}ms kutish...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError;
}

// Test qilish uchun mock:
let callCount = 0;
const result = await retryWithBackoff(
  async () => {
    callCount++;
    if (callCount < 3) throw new Error("Server xato");
    return { data: "muvaffaqiyat!", attempts: callCount };
  },
  4,
  500
);
console.log(result); // { data: "muvaffaqiyat!", attempts: 3 }
```

**Tushuntirish:**
- `for` loop `maxRetries + 1` marta ishlaydi (birinchi urinish + retries)
- Har bir xatoda kutish vaqti `baseDelay * 2^attempt`: 1s → 2s → 4s → 8s
- Oxirgi urinishdan keyin `lastError` throw qilinadi
- `await new Promise(resolve => setTimeout(resolve, delay))` — kutish uchun

</details>

---

### Mashq 2: Concurrent Limit (Qiyin)

**Savol:** `mapWithLimit(items, limit, fn)` funksiyasini yozing:
- `items` — massiv
- `limit` — bir vaqtda nechta task parallel ishlashi
- `fn` — har bir item uchun async funksiya
- Natijalar tartibini saqlash

```javascript
// Test: 10 ta URL, bir vaqtda max 3 ta
const urls = Array.from({ length: 10 }, (_, i) => `https://api.example.com/item/${i}`);

const results = await mapWithLimit(urls, 3, async (url) => {
  const res = await fetch(url);
  return res.json();
});
// results[0] = item/0, results[1] = item/1, ... (tartib saqlangan)
```

<details>
<summary>Javob</summary>

```javascript
async function mapWithLimit(items, limit, fn) {
  const results = new Array(items.length);
  let currentIndex = 0;

  async function worker() {
    while (currentIndex < items.length) {
      const index = currentIndex++;
      try {
        results[index] = await fn(items[index], index);
      } catch (err) {
        results[index] = err; // yoki throw qilish mumkin
      }
    }
  }

  // limit ta worker ni parallel boshlash
  const workers = Array.from(
    { length: Math.min(limit, items.length) },
    () => worker()
  );

  await Promise.all(workers);
  return results;
}

// Test:
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const results = await mapWithLimit(
  [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
  3,
  async (num) => {
    const time = Math.random() * 1000;
    await delay(time);
    console.log(`Item ${num} tayyor (${time.toFixed(0)}ms)`);
    return num * 2;
  }
);

console.log(results); // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20] — tartib saqlangan!
```

**Tushuntirish:**
- `worker()` — bitta "ishchi" funksiya, navbatdan task oladi va bajaradi
- `currentIndex` — keyingi task ni aniqlash (shared state)
- `limit` ta worker parallel ishlaydi — biri tugasa, keyingi task ni oladi
- `results[index]` — natijalar original tartibda saqlanadi
- `Promise.all(workers)` — barcha workerlar tugashini kutish

</details>

---

### Mashq 3: Sequential va Parallel Refactoring (O'rta)

**Savol:** Quyidagi kodni optimallashtiring — parallel bo'ladigan joylarni parallel qiling:

```javascript
// ❌ Hammasi ketma-ket — sekin
async function getOrderSummary(orderId) {
  const order = await fetchOrder(orderId);
  const user = await fetchUser(order.userId);
  const products = [];
  for (const productId of order.productIds) {
    const product = await fetchProduct(productId);
    products.push(product);
  }
  const reviews = [];
  for (const productId of order.productIds) {
    const review = await fetchReviews(productId);
    reviews.push(review);
  }
  const shipping = await calculateShipping(order, user.address);
  return { order, user, products, reviews, shipping };
}
```

<details>
<summary>Javob</summary>

```javascript
// ✅ Optimallashtirilgan — parallel + sequential aralash
async function getOrderSummary(orderId) {
  // 1-qadam: order olish (boshqalar bunga bog'liq)
  const order = await fetchOrder(orderId);

  // 2-qadam: user va products/reviews PARALLEL (bir-biriga bog'liq emas)
  const [user, products, reviews] = await Promise.all([
    // user — order.userId ga bog'liq
    fetchUser(order.userId),

    // products — parallel fetch (bir-biriga bog'liq emas)
    Promise.all(order.productIds.map(id => fetchProduct(id))),

    // reviews — parallel fetch (bir-biriga bog'liq emas)
    Promise.all(order.productIds.map(id => fetchReviews(id))),
  ]);

  // 3-qadam: shipping — order va user.address ga bog'liq
  const shipping = await calculateShipping(order, user.address);

  return { order, user, products, reviews, shipping };
}

// 🔍 Performance taqqoslash:
// Aytaylik: fetchOrder=100ms, fetchUser=150ms, fetchProduct=200ms (x3),
//           fetchReviews=100ms (x3), calculateShipping=50ms
//
// ❌ Sequential: 100 + 150 + (200×3) + (100×3) + 50 = 1200ms
// ✅ Optimized:  100 + max(150, 200, 100) + 50 = 350ms — 3.4x tezroq!
```

**Tushuntirish:**
- `order` birinchi olinishi kerak — boshqalar unga bog'liq
- `user`, `products`, `reviews` — bir-biriga bog'liq emas → `Promise.all`
- `products` va `reviews` ichidagi itemlar ham mustaqil → ichki `Promise.all`
- `shipping` — `order` va `user` kerak → oxirida sequential

</details>

---

### Mashq 4: Async Error Handling Challenge (Qiyin)

**Savol:** Quyidagi kodni yozing:
1. 3 ta API dan parallel data oling
2. Har bir API xato berishi mumkin — xatoni alohida handle qiling
3. Agar hech biri ishlamasa — `AggregateError` throw qiling
4. Hech bo'lmasa bitta ishlaganda — natija qaytaring, ishlamagan joyga `null` qo'ying

```javascript
// Test:
const result = await resilientFetch({
  user: () => fetch('/api/user').then(r => r.json()),
  posts: () => fetch('/api/posts').then(r => r.json()),
  stats: () => fetch('/api/stats').then(r => r.json()),
});
// Agar posts xato bersa:
// result = { user: {...}, posts: null, stats: {...} }
// Agar hammasi xato bersa: AggregateError throw
```

<details>
<summary>Javob</summary>

```javascript
async function resilientFetch(sources) {
  const keys = Object.keys(sources);
  const errors = [];

  const results = await Promise.allSettled(
    keys.map(key => sources[key]())
  );

  const output = {};

  for (let i = 0; i < keys.length; i++) {
    const key = keys[i];
    const result = results[i];

    if (result.status === 'fulfilled') {
      output[key] = result.value;
    } else {
      console.warn(`${key} xato berdi:`, result.reason.message);
      output[key] = null;
      errors.push(result.reason);
    }
  }

  // Agar HAMMA xato bo'lsa — throw
  if (errors.length === keys.length) {
    throw new AggregateError(
      errors,
      `Barcha ${keys.length} ta so'rov muvaffaqiyatsiz`
    );
  }

  return output;
}

// Test:
function mockFetch(name, shouldFail = false, delay = 100) {
  return async () => {
    await new Promise(r => setTimeout(r, delay));
    if (shouldFail) throw new Error(`${name} server xato`);
    return { name, data: `${name} data` };
  };
}

// Aralash natija:
const result = await resilientFetch({
  user: mockFetch('user'),
  posts: mockFetch('posts', true), // xato beradi
  stats: mockFetch('stats'),
});
console.log(result);
// { user: { name: 'user', ... }, posts: null, stats: { name: 'stats', ... } }

// Hammasi xato:
try {
  await resilientFetch({
    user: mockFetch('user', true),
    posts: mockFetch('posts', true),
    stats: mockFetch('stats', true),
  });
} catch (err) {
  console.log(err instanceof AggregateError); // true
  console.log(err.message); // "Barcha 3 ta so'rov muvaffaqiyatsiz"
  console.log(err.errors.length); // 3
}
```

**Tushuntirish:**
- `Promise.allSettled` — hech qachon reject bo'lmaydi, barcha natijalarni qaytaradi
- Har bir natijani tekshiramiz: `fulfilled` → qiymat, `rejected` → null + xatoni saqlash
- Agar barcha so'rovlar xato bo'lsa → `AggregateError` throw (ES2021)
- Kamida bitta ishlaganda → graceful degradation (null bilan qaytarish)

</details>

---

### Mashq 5: Async Queue Implement Qilish (Qiyin)

**Savol:** `AsyncQueue` class yozing:
- `enqueue(task)` — task ni navbatga qo'shish, Promise qaytarish
- Task'lar ketma-ket bajarilishi kerak (bitta-bitta)
- Har bir task natijasi to'g'ri caller ga qaytishi kerak

```javascript
// Test:
const queue = new AsyncQueue();

// Bu 3 ta task parallel qo'shiladi, lekin KETMA-KET bajariladi
const [a, b, c] = await Promise.all([
  queue.enqueue(async () => {
    await delay(100);
    return "birinchi";
  }),
  queue.enqueue(async () => {
    await delay(50);
    return "ikkinchi";
  }),
  queue.enqueue(async () => {
    await delay(75);
    return "uchinchi";
  }),
]);
// a = "birinchi", b = "ikkinchi", c = "uchinchi"
// Bajarilish tartibi: birinchi → ikkinchi → uchinchi (ketma-ket)
```

<details>
<summary>Javob</summary>

```javascript
class AsyncQueue {
  #queue = [];
  #processing = false;

  enqueue(task) {
    return new Promise((resolve, reject) => {
      this.#queue.push({ task, resolve, reject });

      if (!this.#processing) {
        this.#processNext();
      }
    });
  }

  async #processNext() {
    if (this.#queue.length === 0) {
      this.#processing = false;
      return;
    }

    this.#processing = true;
    const { task, resolve, reject } = this.#queue.shift();

    try {
      const result = await task();
      resolve(result);
    } catch (err) {
      reject(err);
    }

    // Keyingi task ni boshlash (recursive, lekin async — stack overflow yo'q)
    this.#processNext();
  }

  get pending() {
    return this.#queue.length;
  }

  get isProcessing() {
    return this.#processing;
  }
}

// Test:
function delay(ms) {
  return new Promise(r => setTimeout(r, ms));
}

const queue = new AsyncQueue();
const log = [];

const [a, b, c] = await Promise.all([
  queue.enqueue(async () => {
    log.push('start-1');
    await delay(100);
    log.push('end-1');
    return "birinchi";
  }),
  queue.enqueue(async () => {
    log.push('start-2');
    await delay(50);
    log.push('end-2');
    return "ikkinchi";
  }),
  queue.enqueue(async () => {
    log.push('start-3');
    await delay(75);
    log.push('end-3');
    return "uchinchi";
  }),
]);

console.log(a, b, c); // "birinchi" "ikkinchi" "uchinchi"
console.log(log);
// ['start-1', 'end-1', 'start-2', 'end-2', 'start-3', 'end-3']
// ☝️ Ketma-ket! 2-chi faqat 1-chi tugagandan keyin boshlandi
```

**Tushuntirish:**
- Har bir `enqueue` chaqiruvi Promise yaratadi va `{ task, resolve, reject }` ni navbatga qo'shadi
- `#processNext()` navbatdan birinchi task ni oladi va bajaradi
- Task tugaganda `resolve/reject` chaqiriladi — enqueue chaqirgan joyga natija qaytadi
- `#processNext()` recursive chaqiriladi — keyingi task boshlanadi
- `#processing` flag — bir vaqtda faqat bitta task ishlaydi

</details>

---

## Xulosa

1. **`async` funksiya doim Promise qaytaradi.** `return value` → `resolve(value)`, `throw error` → `reject(error)`.

2. **`await` Promise resolve bo'lguncha funksiyani to'xtatadi** — lekin main thread ni BLOCKLAMAS. Event loop erkin qoladi.

3. **try/catch** — async/await bilan xatolarni sinxron ko'rinishda handle qilish. `.catch()` zanjirlariga hojat yo'q.

4. **Mustaqil operatsiyalar → `Promise.all`** bilan parallel bajaring. Performance farqi katta — N ta operatsiya vaqtini `max(operatsiya)` ga tushiradi.

5. **Sequential faqat bog'liqlik bo'lganda.** Keyingi operatsiya oldingi natijaga bog'liq bo'lsa — ketma-ket. Aks holda — parallel.

6. **Top-level await** — ES Modules ichida `async` funksiya siz `await` ishlatish. Modul loading ni "async" qiladi.

7. **Production patternlar:** retry (exponential backoff), timeout (AbortController), concurrent limit (server himoyasi), queue (tartibli bajarish).

8. **`for-await-of`** — async iterablelar ustida iteratsiya. Stream, paginated API, real-time data uchun.

9. **Under the hood:** async/await = generator + promise. V8 7.2+ da optimallashtirilgan — extra microtick yo'q.

10. **Eng katta xatolar:** forEach ichida await (ishlamaydi), keraksiz sequential await (parallel kerak), error handling yo'q (unhandled rejection), async constructor (mumkin emas).

---

> **Oldingi bo'lim:** [12-promises.md](12-promises.md) — Promises: then/catch/finally, chaining, Promise.all/allSettled/race/any
>
> **Keyingi bo'lim:** [14-iterators-generators.md](14-iterators-generators.md) — Iterators va Generators: Symbol.iterator, yield, async generators
