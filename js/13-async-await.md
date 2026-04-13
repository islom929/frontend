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
  - [Timeout Pattern (Promise.race va AbortController)](#timeout-pattern-promiserace-va-abortcontroller)
  - [Concurrent Limit](#concurrent-limit)
  - [Queue Pattern](#queue-pattern)
- [for-await-of — Async Iteration](#for-await-of--async-iteration)
- [Under the Hood: Generator + Promise](#under-the-hood-generator--promise)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Async Function Nima?

### Nazariya

`async` keyword bilan belgilangan funksiya **doim Promise qaytaradi** — hatto siz oddiy qiymat return qilsangiz ham, u avtomatik `Promise.resolve()` ga o'raladi; `throw` qilsangiz esa `Promise.reject()` ga aylanadi. Bu shuni anglatadiki, async funksiya chaqirilganda natija hech qachon **bevosita** qaytmaydi — u doim Promise ichida keladi.

Async funksiya nima muammoni hal qiladi? Promise chain'lari (`.then().then()`) callback hell'dan ancha yaxshi bo'lsa-da, murakkab asinxron logikada ular ham o'qilishi qiyin va debugging qiyinlashadi. `async/await` asinxron kodni **sinxron ko'rinishda** yozish imkonini beradi — lekin ichida hamma narsa Promise va microtask'lar bilan ishlaydi. Bu ES2017 (ES8) da kiritilgan va zamonaviy JavaScript'da asinxron kodning **standart** yozish usuli hisoblanadi.

Async funksiyaning uchta muhim xususiyati:
1. **Doim Promise qaytaradi** — `return 42` → `Promise.resolve(42)`, `throw err` → `Promise.reject(err)`
2. **`await` ishlatish imkonini beradi** — faqat async funksiya ichida (yoki ES Module top-level'da)
3. **Error handling `try/catch` bilan ishlaydi** — sinxron koddagidek

<details>
<summary><strong>Under the Hood</strong></summary>

Async funksiya ichida engine quyidagilarni bajaradi:

1. Funksiya chaqirilganda — **yangi Promise yaratiladi**
2. Funksiya tanasi (body) shu Promise ning executori sifatida bajariladi
3. `return value` → `resolve(value)` ga aylanadi
4. `throw error` → `reject(error)` ga aylanadi
5. `await` uchrasa — funksiya **to'xtaydi** (suspend) va boshqaruv chaqiruvchiga qaytadi

```
async function fetchUser() {     Engine ichida:
  const a = await getProfile();→  1. yangi Promise yaratiladi
  return a;                       2. getProfile() chaqiriladi, Promise qaytadi
}                                 3. fetchUser() SUSPEND — boshqaruv caller ga qaytadi
                                  4. getProfile() resolve bo'lganda — fetchUser() RESUME
                                  5. return a → resolve(a)
```

ECMAScript spec bo'yicha `async` funksiya `AsyncFunction` turida bo'lib, `AsyncFunctionCreate` abstract operation orqali yaratiladi. Ordinary function'dan farqi — funksiya chaqirilganda spec'dagi **AsyncFunctionStart** abstract operation ishga tushadi va generator-ga o'xshash suspend/resume mexanizmini boshqaradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Oddiy funksiya va async funksiya farqini ko'rsatadigan misol:

```javascript
// Oddiy funksiya — bevosita qiymat qaytaradi
function getNumber() {
  return 42;
}
console.log(getNumber()); // 42

// Async funksiya — DOIM Promise qaytaradi
async function getNumberAsync() {
  return 42;
}
console.log(getNumberAsync()); // Promise {<fulfilled>: 42}

// Qiymatni olish uchun .then() yoki await kerak:
getNumberAsync().then(value => console.log(value)); // 42
```

`throw` qilsak — rejected Promise qaytadi:

```javascript
async function willFail() {
  throw new Error("Xato!");
}
willFail().catch(err => console.log(err.message)); // "Xato!"

// Bu aslida quyidagicha ishlaydi:
function willFailDesugared() {
  return new Promise((resolve, reject) => {
    try {
      throw new Error("Xato!");
    } catch (e) {
      reject(e);
    }
  });
}
```

Async funksiyaning barcha turlarini ko'rsatadigan misol:

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

Promise qaytarishda muhim nuance — ikki marta wrap qilinmaydi:

```javascript
async function example() {
  return Promise.resolve(42);
}
// Natija: Promise {<fulfilled>: 42} — bitta qiymat, ikki marta wrap qilinmaydi!

// Muhim: async function DOIM yangi Promise qaytaradi (identity saqlanmaydi).
// Lekin value ikki marta wrap bo'lmaydi — spec bo'yicha unwrap qilinadi.
// Ya'ni: example() === Promise.resolve(42) → false (yangi Promise object)
```

</details>

---

## Await Keyword

### Nazariya

`await` — Promise resolve bo'lguncha funksiya bajarilishini **to'xtatadigan** (suspend) operator. Faqat `async` funksiya ichida yoki ES Module'ning top-level scope'ida ishlatiladi. `await` Promise'ni, thenable ob'ektni, yoki oddiy qiymatni qabul qiladi.

Eng muhim tushuncha: `await` funksiyani to'xtatadi, lekin **main thread ni BLOKLAMAYDI**. Funksiya to'xtaganda boshqaruv chaqiruvchiga qaytadi va Event Loop erkin qoladi. Promise resolve bo'lganda funksiya microtask orqali davom etadi. Shu sababli `await` dan keyingi kod aslida sinxron emas — u microtask sifatida bajariladi.

`await` uchta turdagi qiymat bilan ishlaydi:
1. **Promise** — resolve yoki reject bo'lguncha kutadi
2. **Thenable** — `.then()` metodi bor ob'ekt (duck typing)
3. **Non-Promise qiymat** — darhol qaytaradi (`Promise.resolve()` orqali wrap qilib)

<details>
<summary><strong>Under the Hood</strong></summary>

`await` uchrasa engine quyidagi qadamlarni bajaradi:

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
       └─────────────────────────│ 9. Keyingi kodlar bajariladi    │
                                 └─────────────────────────────────┘
}
```

Spec bo'yicha `await` operatori ichida **PromiseResolve** abstract operation chaqiriladi. Agar operand allaqachon Promise bo'lsa — uni to'g'ridan-to'g'ri ishlatadi (V8 7.2+ optimizatsiyasi). Agar oddiy qiymat bo'lsa — `Promise.resolve()` bilan wrap qiladi. Keyin `.then()` handler'i orqali funksiya davom ettiriladi — bu handler microtask queue'ga joylashadi.

> **Eslatma:** Event loop mexanizmi haqida batafsil — [11-event-loop.md](11-event-loop.md), Promises haqida — [12-promises.md](12-promises.md)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`await` ning asosiy xatti-harakati — funksiyani to'xtatadi, lekin main thread erkin qoladi:

```javascript
async function demo() {
  console.log("1 — boshlanish");

  const result = await new Promise(resolve => {
    setTimeout(() => resolve("2 — await tugadi"), 1000);
  });

  console.log(result);     // "2 — await tugadi" (1 sekunddan keyin)
  console.log("3 — davom"); // await dan keyin davom etadi
}

demo();
console.log("4 — tashqarida"); // Bu AVVAL chiqadi!

// Tartib: 1, 4, (1s keyin) 2, 3
```

`await` turli qiymatlar bilan ishlashini ko'rsatadigan misol:

```javascript
// 1. Promise kutadi — resolve yoki reject bo'lguncha
const data = await fetch('/api/data'); // network so'rov tugashini kutadi

// 2. Thenable object — .then() metodi bor bo'lsa
const thenable = {
  then(resolve) {
    setTimeout(() => resolve(42), 1000);
  }
};
const val = await thenable; // 42 (1s keyin)

// 3. Non-Promise qiymat — darhol qaytaradi (Promise.resolve() orqali)
const num = await 42;       // 42 — darhol (lekin hali ham microtask!)
const str = await "hello";  // "hello" — darhol
```

Execution tartibini tushunish uchun muhim misol:

```javascript
async function order() {
  console.log("A"); // 1 — sinxron

  const x = await Promise.resolve("B");
  console.log(x);   // 3 — microtask

  console.log("C"); // 4 — await dan keyin (lekin aslida microtask ichida)
}

console.log("START"); // Eng birinchi
order();
console.log("END");   // 2 — order() await da to'xtadi

// Natija: START → A → END → B → C
```

Nima uchun `END` avval chiqadi? Chunki `await` funksiyani to'xtatib, **boshqaruvni chaqiruvchiga qaytaradi**. `order()` dan keyingi `console.log("END")` darhol bajariladi. `await` dan keyingi kod esa faqat microtask navbatida bajariladi.

</details>

---

## try/catch Bilan Error Handling

### Nazariya

Async/await ning eng katta afzalliklaridan biri — error handling oddiy `try/catch` bilan ishlaydi, sinxron koddagi kabi oddiy sintaksis bilan. Promise'dagi `.catch()` zanjirlariga hojat yo'q — kod o'qilishi va tuzatilishi ancha osonlashadi.

`try/catch` bilan ishlashning uchta asosiy strategiyasi mavjud:
1. **Yagona try/catch** — butun funksiya uchun bitta catch blok
2. **Granular try/catch** — har bir `await` uchun alohida try/catch (har bir xatoni alohida handle qilish kerak bo'lganda)
3. **`await promise.catch()` pattern** — try/catch siz inline error handling

`finally` bloki resurslarni tozalash uchun ishlatiladi — xato bo'lsa ham, muvaffaqiyat bo'lsa ham bajariladi. Spinner yashirish, connection yopish, cleanup — `finally` uchun ideal use case.

<details>
<summary><strong>Kod Misollari</strong></summary>

Promise chain bilan solishtirganda async/await error handling qanday soddalashtiradi:

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
    throw err; // xatoni qayta throw qilish — caller ham bilsin
  }
}
```

Turli xato turlarini `instanceof` bilan ajratib handle qilish:

```javascript
async function processOrder(orderId) {
  try {
    const order = await fetchOrder(orderId);
    const payment = await processPayment(order);
    const shipment = await createShipment(order, payment);
    return shipment;
  } catch (err) {
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
    // Kutilmagan xato — qayta throw
    throw err;
  }
}
```

Har bir `await` ni alohida try/catch bilan o'rash — granular error handling:

```javascript
async function getUserData(userId) {
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

`await + .catch()` pattern — try/catch siz xatoni inline tutish:

```javascript
async function getUser(id) {
  const user = await fetchUser(id).catch(err => {
    console.error("Xato:", err);
    return null; // default qiymat
  });

  if (!user) return;
  // davom...
}
```

`finally` bloki bilan resurslarni tozalash:

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

</details>

---

## Parallel Execution — Promise.all + await

### Nazariya

`await` ni ketma-ket ishlatish har bir operatsiya uchun oldingi tugashini kutadi. Agar operatsiyalar **bir-biriga bog'liq bo'lmasa** — bu behuda vaqt yo'qotish. `Promise.all()` bilan mustaqil operatsiyalarni **parallel** bajarish mumkin — natijada umumiy vaqt eng sekin operatsiya vaqtiga teng bo'ladi.

`Promise.all` ning muhim xususiyati — **fail-fast** semantikasi. Bitta Promise reject bo'lsa — `Promise.all` darhol reject bo'ladi va qolgan muvaffaqiyatli natijalar yo'qoladi. Agar xatolarga chidamli bo'lish kerak bo'lsa — `Promise.allSettled` ishlatiladi, u barcha natijalarni (fulfilled va rejected) qaytaradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Sequential va parallel execution farqini ko'rsatadigan misol:

```javascript
// ❌ YOMON: ketma-ket — har biri oldingi tugashini kutadi
async function getPageData() {
  const user = await fetchUser();        // 1s kutadi
  const posts = await fetchPosts();      // yana 1s kutadi
  const comments = await fetchComments(); // yana 1s kutadi
  return { user, posts, comments };
  // Jami: ~3 sekund
}

// ✅ YAXSHI: parallel — hammasi bir vaqtda boshlanadi
async function getPageData() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),        // |████| 1s ─┐
    fetchPosts(),       // |████| 1s ─┼─ parallel
    fetchComments()     // |████| 1s ─┘
  ]);
  return { user, posts, comments };
  // Jami: ~1 sekund (eng sekin operatsiya vaqti)
}
```

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
Jami:  ═══════════ 1s  (3x tezroq!)
```

`Promise.all` fail-fast xatti-harakati:

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

`Promise.allSettled` — xatolarga chidamli variant:

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

Real-world misol — dashboard uchun parallel data yuklash:

```javascript
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

// Performance farqi:
// Sequential: 200ms + 300ms + 150ms + 100ms = 750ms
// Parallel:   max(200, 300, 150, 100) = 300ms — 2.5x tezroq!
```

</details>

---

## Sequential vs Parallel

### Nazariya

Sequential va parallel execution qachon ishlatishni bilish — async kodning **performance**ini belgilovchi eng muhim qaror. Qoida oddiy: keyingi operatsiya oldingi natijaga **bog'liq** bo'lsa — sequential; operatsiyalar **mustaqil** bo'lsa — parallel.

Real-world da ko'pincha **aralash** pattern ishlatiladi: ba'zi operatsiyalar parallel bajariladi, natijalar kelgandan keyin keyingi bosqich sequential davom etadi. Bu farqni bilmaslik eng ko'p uchraydigan performance muammolardan biri.

<details>
<summary><strong>Under the Hood</strong></summary>

```
Qaror daraxti:

Operatsiyalar bir-biriga bog'liqmi?
│
├─ HA → Sequential (ketma-ket await)
│   Masalan: userId → postlarni ol → post bilan commentlarni ol
│
└─ YO'Q → Parallel (Promise.all)
    Masalan: user, posts, notifications — barchasi mustaqil
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sequential — keyingisi oldingi natijaga bog'liq:

```javascript
async function getOrderDetails(orderId) {
  // 1. Avval orderni olish kerak
  const order = await fetchOrder(orderId);

  // 2. Order ichidagi userId kerak — oldingi natijaga BOG'LIQ
  const user = await fetchUser(order.userId);

  // 3. Order ichidagi productIds kerak — lekin productlar mustaqil!
  const products = await Promise.all(
    order.productIds.map(id => fetchProduct(id))
  );
  // ☝️ Productlar bir-biridan mustaqil — ularni parallel qilamiz

  return { order, user, products };
}
```

Anti-pattern va to'g'ri usul:

```javascript
// ❌ ANTI-PATTERN: mustaqil so'rovlar ketma-ket
async function getProfile(userId) {
  const user = await fetchUser(userId);       // 200ms
  const avatar = await fetchAvatar(userId);   // 150ms
  const settings = await fetchSettings(userId); // 100ms
  // Jami: 450ms — lekin ular mustaqil!
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

Aralash pattern — real-world e-commerce checkout misoli:

```javascript
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

Performance farqini o'lchash uchun yordamchi funksiya:

```javascript
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

</details>

---

## Top-Level Await (ES2022)

### Nazariya

ES2022 dan boshlab, **ES Modules** ichida `await` ni `async` funksiya siz ham ishlatish mumkin — bu **top-level await**. U modulning o'zini "async" qiladi — bu modulni import qilgan boshqa modullar ham uning tayyor bo'lishini kutadi.

Top-level await faqat ES Module'larda ishlaydi — `.mjs` fayllar yoki `"type": "module"` ko'rsatilgan `package.json` dagi `.js` fayllar. CommonJS (`require`) va oddiy script'larda (non-module `<script>`) ishlamaydi.

Top-level await quyidagi scenariolarda ishlatiladi:
1. **Module initialization** — database connection, konfiguratsiya yuklash
2. **Dynamic import** — shart bo'yicha modul yuklash
3. **Fallback pattern** — birinchi CDN ishlamasa ikkinchisini sinash
4. **Conditional module loading** — environment ga qarab turli modul

Ehtiyot bo'lish kerak: top-level await sekin operatsiyani modulda ishlatish butun dependency tree'ni bloklab qo'yishi mumkin. Bu modulni import qilgan **barcha** modullar shu await tugashini kutadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Asosiy ishlatish — modul initialization:

```javascript
// config.mjs (ES Module)
const response = await fetch('/api/config');
const config = await response.json();

export default config;
// Modul import qilinganda — config TAYYOR bo'ladi
```

Module loading mexanizmi:

```javascript
// database.mjs
const connection = await connectToDatabase();
export { connection };

// app.mjs
import { connection } from './database.mjs';
// Bu kutadi — database tayyor bo'lguncha
```

```
Module Loading:
────────────────────────────────────────────────────

database.mjs:
├── connectToDatabase() ────────────────┤ (await)
                                        export { connection }

app.mjs:
├── import database ─── KUTISH ─────────┤ connection tayyor
                                        ├── app boshlanadi

config.mjs:    (parallel yuklanyapti, lekin mustaqil)
├── fetch config ──┤ export config
```

Qayerda ishlaydi va qayerda ishlamaydi:

```javascript
// ✅ Ishlaydi — ES Module (.mjs yoki "type": "module" package.json da)
const data = await loadConfig();
export default data;

// ❌ ISHLAMAYDI — CommonJS (.cjs yoki oddiy .js)
const data = await loadConfig(); // SyntaxError!

// ❌ ISHLAMAYDI — oddiy script (non-module)
// <script src="app.js"> ichida top-level await ISHLAMAYDI
// <script type="module"> kerak
```

Practical use cases:

```javascript
// 1. Dynamic import — locale bo'yicha
const locale = navigator.language;
const strings = await import(`./i18n/${locale}.mjs`);

// 2. Fallback pattern — CDN ishlamasa boshqasini sinash
let jQuery;
try {
  jQuery = await import('https://cdn-a.example.com/jquery.mjs');
} catch {
  jQuery = await import('https://cdn-b.example.com/jquery.mjs');
}
export default jQuery;

// 3. Conditional module loading
const isProduction = process.env.NODE_ENV === 'production';
const logger = isProduction
  ? await import('./prod-logger.mjs')
  : await import('./dev-logger.mjs');
export default logger;
```

Top-level await bilan **ehtiyot bo'lish** — sekin modul hamma narsani bloklab qo'yadi:

```javascript
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

</details>

---

## Async Patterns

Quyidagi bo'limda production-ready async patternlar — har biri real-world muammoni hal qiladi.

### Retry Logic (Exponential Backoff)

#### Nazariya

Network so'rovlari muvaffaqiyatsiz bo'lganda qayta urinish — production dasturlarda eng ko'p ishlatiladigan patternlardan biri. **Exponential backoff** har safar kutish vaqtini 2x ga oshiradi (1s → 2s → 4s → 8s) — bu serverni ortiqcha yuklamaslik uchun muhim. Qo'shimcha **jitter** (random kechikish) esa barcha clientlar bir vaqtda retry qilmasligini ta'minlaydi — bu "thundering herd" muammosining oldini oladi.

```
Retry with Exponential Backoff:
───────────────────────────────────────────────────────────────

Urinish 1:   ├─── request ───┤ ❌ xato
              └── kutish: 1s ──┤

Urinish 2:                     ├─── request ───┤ ❌ xato
                                └── kutish: 2s ──────┤

Urinish 3:                                           ├─── request ───┤ ❌ xato
                                                      └── kutish: 4s ──────────┤

Urinish 4:                                                                     ├─── request ───┤ ✅

Har safar kutish 2x oshadi: 1s → 2s → 4s → 8s → ...
+ Jitter (random qo'shimcha) — thundering herd oldini olish uchun
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Exponential backoff formulasi**: `delay(attempt) = min(maxDelay, baseDelay × 2^attempt)`. Har retry oldingisidan 2 marta uzoqroq: 1s → 2s → 4s → 8s → 16s.

**Jitter nima uchun kerak**: 1000 client bir vaqtda 503 xato olib, 1 sekund'dan keyin qayta urinsa — server qayta crash qiladi (**thundering herd**). Jitter random komponent qo'shadi:
```
Full jitter:  delay = random(0, baseDelay * 2^attempt)
Equal jitter: delay = (baseDelay * 2^attempt) / 2 + random(...)
```
AWS/Google Cloud **equal jitter**'ni tavsiya qiladi.

**Idempotency muhim**: Retry faqat idempotent operatsiyalar uchun xavfsiz:
- ✅ GET, PUT, DELETE — idempotent
- ❌ POST — duplicate yaratishi mumkin, **idempotency key** kerak (Stripe pattern)

**Status code'larga qarab**:
- 4xx (client error) — retry mantiqsiz (404, 401, 403)
- 408 (timeout), 429 (rate limit) — retry mumkin, `Retry-After` header hurmat qiling
- 5xx (server error) — retry oqilona (vaqtinchalik xato)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
async function retry(fn, options = {}) {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 30000,
    backoffFactor = 2,
    jitter = true,
    retryOn = () => true,  // qaysi xatolarda retry qilish
    onRetry = () => {},     // har bir retry da callback
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
      // Faqat network va 5xx xatolarda retry qilish
      // 4xx (client error) da retry qilish mantiqsiz
      return err.message.includes('fetch') || err.message.includes('5');
    },
    onRetry: (err, attempt, delay) => {
      console.log(`Xato: ${err.message}. ${attempt}-urinish ${delay}ms dan keyin...`);
    },
  }
);
```

</details>

---

### Timeout Pattern (Promise.race va AbortController)

#### Nazariya

Asinxron operatsiyaga vaqt chegarasi qo'yish — production da muhim pattern. Agar tashqi API belgilangan vaqt ichida javob bermasa, dastur cheksiz kutmasligi kerak. Bu ikki usulda amalga oshiriladi:

1. **`Promise.race()`** — operatsiya va timeout Promise'ni "poyga"ga qo'yish. Qaysi biri birinchi settle bo'lsa — shu natija.
2. **`AbortController`** — so'rovni to'g'ridan-to'g'ri bekor qilish. `Promise.race` dan farqli — so'rov haqiqatan ham to'xtatiladi va network resurs tejaladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Race vs Cancel — eng muhim farq**:
```javascript
// ❌ Promise.race — request DAVOM ETADI background'da
await Promise.race([fetch('/slow'), timeoutPromise(5000)]);
// Timeout bo'lsa ham fetch tugaydi, natija ignored (memory waste)

// ✅ AbortController — request HAQIQATAN to'xtatiladi
const ctrl = new AbortController();
setTimeout(() => ctrl.abort(), 5000);
await fetch('/slow', { signal: ctrl.signal });
// Timeout bo'lsa — network connection yopiladi, resurs tejaladi
```

`Promise.race` faqat birinchi settled Promise'ni qaytaradi, boshqalar davom etaveradi. `AbortController` esa **haqiqiy cancellation** beradi — fetch API'ning signal'iga subscribe bo'ladi va `abort` event'ida network connection yopiladi.

**`AbortSignal.timeout()` (ES2024)** — alohida controller yaratish keraksiz:
```javascript
fetch(url, { signal: AbortSignal.timeout(5000) });
```

**`AbortSignal.any()`** — bir nechta signal'ni birlashtirish (user cancel + timeout + page unload):
```javascript
const signal = AbortSignal.any([userCancel, AbortSignal.timeout(10000)]);
fetch(url, { signal });
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`Promise.race` bilan timeout:

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

`AbortController` bilan — so'rovni to'xtatadi (to'g'ri usul):

```javascript
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
    clearTimeout(timeoutId); // timeout ni tozalash
  }
}

// Farq: Promise.race timeout da so'rov hali ishlayapti (resurs sarflaydi)
//        AbortController esa so'rovni TO'XTATADI — resurs tejaydi
```

</details>

---

### Concurrent Limit

#### Nazariya

1000 ta URL bor — hammasini bir vaqtda `Promise.all` qilsak, server 1000 ta parallel so'rov ko'radi va bu DDoS hisoblanadi. **Concurrent limit** — bir vaqtda faqat N ta so'rov yuborib, biri tugashi bilan navbatdagini boshlash. Bu server va client resurslarini tejaydi va rate-limiting'ga tushishdan saqlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Worker pool pattern**: N ta "slot" bilan ishlaydi. Har slot bo'shashganda navbatdagi task'ni oladi. Bu **producer-consumer** pattern — backpressure avtomatik (agar consumer sekin bo'lsa, queue to'ladi lekin crash bo'lmaydi).

**Nima uchun kerak**: `Promise.all([1...1000000].map(fn))` — 1M Promise bir vaqtda:
- 1M async operation start
- 1M callback va state object'lar memory'da
- Network: 1M request navbatda
- Crash xavfi

Concurrent limit shu muammoni hal qiladi — faqat N ta parallel, qolgani navbatda kutadi.

**Real-world library'lar**: **p-limit** (eng mashhur, minimal), **p-queue** (priority + concurrency), **bottleneck** (rate limit + concurrency), **fastq** (performance optimized).

```
Concurrent Limit (max 3):
──────────────────────────────────────────────────────────────

Slot 1: ├── req1 ──┤ ├── req4 ──────┤ ├── req7 ──┤ ├── req9 ───┤
Slot 2: ├── req2 ────────┤ ├── req5 ──┤ ├── req8 ────┤ ├── req10 ──┤
Slot 3: ├── req3 ──┤ ├── req6 ──────────┤

         Har doim max 3 ta so'rov parallel ishlaydi
         Biri tugasa — navbatdagi boshlanadi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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
const urls = Array.from(
  { length: 100 },
  (_, i) => `https://api.example.com/item/${i}`
);

const tasks = urls.map(url => () => fetch(url).then(r => r.json()));

// Bir vaqtda faqat 5 ta so'rov
const results = await concurrentLimit(tasks, 5);
console.log(`${results.length} ta natija olindi`);
```

`pLimit` pattern — sodda va ishonchli implementatsiya:

```javascript
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

</details>

---

### Queue Pattern

#### Nazariya

Queue pattern — task'lar ketma-ket (bitta-bitta) bajarilishi kerak bo'lganda ishlatiladigan pattern. Database yozish operatsiyalari, file system o'zgarishlari, yoki tartib muhim bo'lgan tranzaktsiyalar uchun ideal. U yangi task'larni navbatga qo'yadi va har bir task'ni oldingi tugagandan keyin bajaradi. Concurrent limit'ning `limit = 1` maxsus holati deb ham qarash mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**FIFO guarantee**: Queue task'larni **enqueue tartibi** bo'yicha bajaradi. Bu database transaction'lar uchun kritik:
```javascript
// ❌ Race condition
async function transfer(from, to, amount) {
  const balance = await db.get(from);
  await db.set(from, balance - amount);
}

// ✅ Queue bilan — sequential
await txQueue.enqueue(() => transfer(from, to, amount));
```

**Promise resolver pattern**: Har enqueued task yangi Promise yaratadi, resolver/rejecter closure orqali saqlanadi. ES2024 da `Promise.withResolvers()` buni soddalashtiradi:
```javascript
enqueue(task) {
  const { promise, resolve, reject } = Promise.withResolvers();
  this.queue.push({ task, resolve, reject });
  this.process();
  return promise;
}
```

**Error isolation**: Har task uchun alohida `try/catch` — bitta task fail bo'lsa ham, queue ishlashda davom etadi (fault isolation).

**Compact pattern** — 3 qatorli queue:
```javascript
let chain = Promise.resolve();
function enqueue(task) {
  chain = chain.then(task, task);
  return chain;
}
```
Memory leak xavfi bor (chain o'sib boradi), production'da to'liq queue class afzal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

Priority qo'llab-quvvatlaydigan queue:

```javascript
class PriorityAsyncQueue {
  #queue = [];
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
queue.enqueue(() => processPayment(order), 10); // yuqori — avval bajariladi
queue.enqueue(() => logAnalytics(event), 0);    // eng past
```

</details>

---

## for-await-of — Async Iteration

### Nazariya

`for-await-of` — asinxron iterable (async iterator) ustida iteratsiya qilish uchun maxsus tsikl. Stream, paginated API, va real-time data kabi scenariolarda ishlatiladi. U oddiy `for-of` ning asinxron versiyasi bo'lib, har bir iteratsiyada Promise resolve bo'lishini kutadi.

Bu pattern ayniqsa katta ma'lumotlarni bo'laklab o'qish (streaming), sahifalangan API natijalarini ketma-ket olish, yoki WebSocket/Server-Sent Events kabi real-time ma'lumot oqimlarini qayta ishlash uchun muhim. Async iteration **Symbol.asyncIterator** protocol'iga asoslanadi — bu oddiy **Symbol.iterator** ning async versiyasi.

> **Eslatma:** Iterator va Generator haqida batafsil — [14-iterators-generators.md](14-iterators-generators.md)

<details>
<summary><strong>Under the Hood</strong></summary>

Oddiy iterator `{ value, done }` qaytaradi. Async iterator esa `Promise<{ value, done }>` qaytaradi. `for-await-of` har bir iteratsiyada bu Promise resolve bo'lishini kutadi, keyin keyingi iteratsiyaga o'tadi.

```
Oddiy iterator:            Async iterator:
──────────────             ──────────────────
next() → { value, done }  next() → Promise<{ value, done }>
                                    │
                                    └── await bilan kutiladi
```

Async generator — `async function*` — `yield` va `await` ni birgalikda ishlatish imkonini beradi. Har bir `yield` qilingan qiymat Promise ichida keladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Oddiy for-of va for-await-of farqi:

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

Async generator bilan paginated API:

```javascript
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

Cursor-based pagination — real-world pattern:

```javascript
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

Node.js readable stream bilan:

```javascript
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

Custom async iterable class — event polling:

```javascript
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

</details>

---

## Under the Hood: Generator + Promise

### Nazariya

`async/await` aslida **generator + promise** ning syntactic sugar versiyasi. ES2017 dan oldin Babel va boshqa transpiler'lar async/await ni generator+promise ga aylantirar edi. Generator'ning `yield` operatori funksiyani to'xtatadi, Promise resolve bo'lganda davom ettiradi — aynan `await` qilgan ishni bajaradi.

Bu ichki mexanizmni tushunish quyidagi savollarni javob beradi:
- `await` dan keyingi kod nima uchun darhol bajarilmaydi?
- Nima uchun async funksiya microtask orqali davom etadi?
- Debug paytida call stack'da generator frame'larini ko'rish sababi nima?

<details>
<summary><strong>Under the Hood</strong></summary>

Async/await qanday desugar (transpile) bo'ladi:

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

// spawn — generator ni boshqaruvchi funksiya (runner):
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

```
async/await (biz yozamiz):         generator + promise (engine ko'radi):
─────────────────────────────       ──────────────────────────────────────

async function fetchUser() {        function fetchUser() {
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

</details>

### V8 Optimization: Zero-Cost Async/Await

V8 v7.2+ (Node.js 12+, Chrome 72+) dan boshlab `async/await` oldingi versiyalarga qaraganda ancha tezroq. Sabab — **extra microtick** yo'qoldi.

```javascript
// Oldin (V8 < 7.2): 3 ta microtick kerak edi
// await har safar qo'shimcha Promise yaratardi

// Hozir (V8 >= 7.2): faqat 1 ta microtick
// Agar await qilingan qiymat allaqachon Promise bo'lsa —
// V8 uni to'g'ridan-to'g'ri ishlatadi, qo'shimcha wrap QILMAYDI

async function example() {
  const result = await Promise.resolve(42);
  return result;
}
// V8 7.2+: bu faqat 1 microtick oladi
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

<details>
<summary><strong>Kod Misollari</strong></summary>

Generator va async/await taqqoslash:

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
// Bu patternni ishlatish uchun runner (spawn) funksiya kerak

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
// Runner kerak emas — engine o'zi boshqaradi

// Farq:
// - async/await: native engine support, tezroq, stack trace yaxshiroq
// - generator: manual runner kerak, sekinroq, stack trace murakkab
// - Bugungi kunda generator pattern shart emas — async/await ishlatamiz
```

</details>

---

## Edge Cases va Gotchas

### Sync `throw` async function ichida — rejected Promise, sync catch ushlamaydi

Async funksiya ichida `throw` qilinsa — hatto funksiya boshida `await` bo'lmasa ham — xato **sync throw emas**, **rejected Promise** bo'lib qaytadi. Bu sync try/catch bilan ushlash mumkin emas: chaqirgan joy ham async bo'lishi va `await` qilishi kerak.

```javascript
async function broken() {
  throw new Error("xato"); // ← sync throw bo'lib ko'rinadi, lekin Promise.reject'ga aylanadi
}

// ❌ Sync try/catch ushlamaydi:
try {
  broken(); // Promise.reject qaytaradi — sync throw emas
} catch (e) {
  console.log("never caught"); // ← BU chaqirilmaydi!
}
// Console'da: "Unhandled Promise rejection: xato"

// ✅ await bilan — ushlanadi:
async function main() {
  try {
    await broken();
  } catch (e) {
    console.log("caught:", e.message); // "caught: xato"
  }
}

// ✅ Yoki .catch bilan:
broken().catch(e => console.log("caught:", e.message));
```

**Nima uchun:** Spec bo'yicha async funksiya chaqirilganda darhol Promise yaratiladi. Body ichida har qanday `throw` — shu Promise'ni reject qiladi, sync execution'ga tegmaydi. Bu async funksiyaning **asosiy kontrakti**: "returns a Promise" kafolati — throw bo'lgan bo'lsa ham, mic-mic yoki sync'da ham.

**Yechim:** Async funksiyani har doim `await` yoki `.catch()` bilan chaqiring. Sync chaqiruv (`broken();` alohida) — unhandledrejection event'ini trigger qiladi (Node.js 15+ da process crash).

---

### `await` on non-Promise value — hali ham microtask suspension

`await` faqat Promise emas, **har qanday qiymat** bilan ishlaydi — number, string, object, va hokazo. Lekin non-Promise qiymat bilan ham `await` funksiyani **suspend** qiladi va keyingi kod **microtask** sifatida bajariladi. Bu intuitively sinxron ko'rinishi mumkin, lekin aslida emas.

```javascript
async function test() {
  console.log("1 — boshlanish");
  await 42; // ← Number, Promise emas
  console.log("2 — await dan keyin");
}

test();
console.log("3 — tashqarida");

// Output:
// 1 — boshlanish
// 3 — tashqarida   ← BU avval chiqadi!
// 2 — await dan keyin
```

**Nima uchun:** `await value` spec bo'yicha `value` ni `Promise.resolve(value)` bilan wrap qiladi (yoki agar u Promise bo'lsa — to'g'ridan-to'g'ri ishlatadi). Keyin funksiya **suspend** bo'ladi va continuation microtask queue'ga qo'yiladi. Even if value is instantly "resolved", the continuation **always** goes through one microtask tick — bu JavaScript'ning "consistent async scheduling" kafolati.

**Amaliy ta'sir:**
```javascript
// ❌ Intuitive lekin noto'g'ri:
async function getValue() {
  return cache.has(key) ? cache.get(key) : await fetchValue(key);
}
// Cache hit holatida ham `return` "async return" — caller microtask'dan keyin oladi

// ✅ Agar sync path kerak bo'lsa:
function getValue() {
  return cache.has(key) ? cache.get(key) : fetchValue(key);
  // Oddiy function qaytaradi: sync value yoki Promise
}
```

**Muhim:** Zalgo anti-pattern'ni oldini olish uchun async API'lar **doim** microtask orqali qaytarishi yaxshi — consistent behavior. Lekin hot path'da bu ozgina overhead.

---

### Top-level await + circular imports — deadlock xavfi

Top-level `await` ES Module'larda modul'ni **async** qiladi — uni import qilgan boshqa modullar shu await tugashini kutadi. Agar circular import bilan birga ishlatsangiz, **deadlock** yuzaga kelishi mumkin: A modul B'ni kutadi, B esa A ni kutadi.

```javascript
// ❌ a.mjs
import { b } from './b.mjs'; // B modulini kutadi
export const a = await initA(); // sekin initialization

// ❌ b.mjs
import { a } from './a.mjs'; // A modulini kutadi
export const b = await initB(a); // a ga bog'liq — lekin a hali tayyor emas!

// Yuklash: import a.mjs
// → a.mjs: import b.mjs kutadi
// → b.mjs: import a.mjs kutadi (allaqachon loading)
// → circular detected, lekin a.mjs awaits hali complete emas
// → deadlock: a waits for b, b waits for a's awaited value
```

**Nima uchun:** ESM spec'da top-level await modul'ning "async graph"'ini yaratadi. Loader modullar o'rtasidagi dependency'ni topoligik tartibda hal qilishga urinadi, lekin circular + async kombinatsiyasi **partial initialization** holatiga olib keladi — A modul hali awaiting, B'ga eksport qilingan qiymat hali `undefined`.

**Best practices:**
1. **Circular import'lardan saqlaning** — bu har doim design smell
2. **Top-level await'ni kamdan-kam ishlating** — oddiy export afzalroq bo'lsa
3. **Shared state'ni alohida modulga ajrating** — A va B o'rniga A, B, shared.mjs
4. **Lazy initialization** — top-level await o'rniga lazy getter

```javascript
// ✅ Lazy pattern — circular deadlock'dan saqlanadi
let _a;
export async function getA() {
  if (!_a) _a = await initA();
  return _a;
}
```

---

### Async generator `finally` — `break` da ham ishlaydi (cleanup semantics)

Async generator (`async function*`) ichidagi `try/finally` bloki **consumer `break` qilganda ham** ishlaydi — iterator'ning `return()` method'i chaqirilishi orqali. Bu JavaScript'ning "graceful cleanup" pattern'i — resource management uchun muhim.

```javascript
async function* dataStream() {
  const connection = await openConnection();
  console.log("Connection opened");

  try {
    for (let i = 0; i < 1000; i++) {
      const chunk = await connection.read();
      yield chunk;
    }
  } finally {
    // ✅ Iterator break qilinsa ham ishlaydi!
    await connection.close();
    console.log("Connection closed");
  }
}

async function main() {
  for await (const chunk of dataStream()) {
    console.log("Got chunk:", chunk);
    if (shouldStop(chunk)) {
      break; // ← iterator.return() chaqiriladi → finally ishlaydi
    }
  }
}

// Output (masalan 5 chunk'dan keyin break):
// Connection opened
// Got chunk: ...
// Got chunk: ... (x5)
// Connection closed  ← Cleanup bajarildi, hatto break'dan keyin ham
```

**Nima uchun:** `for await...of` tsikli `break`, `return`, yoki `throw` bilan to'xtaganda, engine iterator'ning `return()` method'ini chaqiradi. Async generator'da bu method joriy yield position'dan davom etadi va **finally bloki ishlaydi**. Bu stream'lar, database connection'lar, va WebSocket kabi resurslar uchun kritik.

**Muhim:** `break` bilan cleanup avtomatik. Lekin **exception** bilan bo'lsa (masalan `throw` iterator ichida), ham `finally` ishlaydi. Buni spec'da "Completion Records" orqali formal belgilangan.

**Gotcha:** Agar siz `finally` ichida `await` qilsangiz — `for await...of` loop'ning `break`'i bu cleanup tugaguncha **kutadi**. Sekin cleanup = sekin break.

---

### `Promise.all` reject — boshqa promise'lar background'da davom etadi (cancellation yo'q)

`Promise.all` bitta Promise reject bo'lsa **darhol** reject qiladi, lekin **boshqa promise'larni to'xtatmaydi** — ular background'da davom etadi va resurs sarflaydi. Bu "cancellation illusion" — xato ushlangandek, lekin ishlar hali ham davom etadi.

```javascript
async function task(name, duration, fail = false) {
  console.log(`${name} started`);
  await new Promise(r => setTimeout(r, duration));
  if (fail) throw new Error(`${name} failed`);
  console.log(`${name} completed`); // ← Bu hali ham ishlaydi!
  return name;
}

async function main() {
  try {
    await Promise.all([
      task("A", 3000),           // 3s — uzoq
      task("B", 500, true),      // 0.5s — darhol fail
      task("C", 2000),           // 2s — uzoq
    ]);
  } catch (e) {
    console.log("Caught:", e.message);
  }

  console.log("Main function done");
}

main();

// Output:
// A started
// B started
// C started
// Caught: B failed     ← 500ms (Promise.all rejected)
// Main function done   ← Darhol keyin
// C completed          ← ❌ 2000ms — HALI ISHLAYDI!
// A completed          ← ❌ 3000ms — HALI ISHLAYDI!
```

**Nima uchun:** Promise'lar spec'da **cancelable emas** — ular state machine, bir marta start bo'lgandan keyin "abort" qilish imkoniyati yo'q. `Promise.all` reject bo'lishi — bu faqat "aggregate Promise" state'ini o'zgartirish, source Promise'larni to'xtatish emas. Backgroundda ular microtask queue'larida ishlaydi, network request'lar tugaydi, timer'lar firing bo'ladi.

**Xavf:**
- **Resurs waste** — bekor qilingan ishlar hali ham CPU/network/memory sarflaydi
- **Side effects** — fail'dan keyin ham boshqa task'lar database yozishi, email yuborishi mumkin
- **Race conditions** — logic "hammasi bekor" deb o'ylaydi, lekin background hali ishlaydi

**Yechim — `AbortController` bilan haqiqiy cancellation:**

```javascript
async function task(name, duration, signal, fail = false) {
  console.log(`${name} started`);
  try {
    await new Promise((resolve, reject) => {
      const timer = setTimeout(resolve, duration);
      signal.addEventListener("abort", () => {
        clearTimeout(timer);
        reject(new Error("Aborted"));
      });
    });
    if (fail) throw new Error(`${name} failed`);
    console.log(`${name} completed`);
  } catch (e) {
    if (e.message === "Aborted") {
      console.log(`${name} aborted`); // ← Cleanup'ga ishora
    } else {
      throw e;
    }
  }
}

async function main() {
  const controller = new AbortController();
  try {
    await Promise.all([
      task("A", 3000, controller.signal),
      task("B", 500, controller.signal, true),
      task("C", 2000, controller.signal),
    ]);
  } catch (e) {
    controller.abort(); // ✅ Boshqa task'larni to'xtatish
    console.log("Caught:", e.message);
  }
}

// Output:
// A started
// B started
// C started
// A aborted    ← ✅ B failed'dan keyin darhol
// C aborted    ← ✅
// Caught: B failed
```

**Muhim:** Har bir async operation `AbortSignal` qabul qilishi kerak. `fetch`, `setTimeout` (custom wrapper bilan), va modern API'lar bu pattern'ni qo'llab-quvvatlaydi.

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
  // Jami: 900ms
}
```

### ✅ To'g'ri usul:

```javascript
async function getUserPage(userId) {
  const [user, notifications, feed] = await Promise.all([
    fetchUser(userId),           // 300ms ─┐
    fetchNotifications(userId),  // 200ms ─┼─ parallel
    fetchFeed(userId),           // 400ms ─┘
  ]);
  return { user, notifications, feed };
  // Jami: 400ms (eng sekin operatsiya vaqti)
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

**Nima uchun:** `forEach` return qiymatini e'tiborsiz qoldiradi — async callback qaytargan Promise ni hech kim `await` qilmayapti. Natijada barcha so'rovlar "fire and forget" bo'ladi.

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
  // 100 ta user × 200ms = 20 sekund
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Yaxshi — barcha so'rovlarni parallel
async function fetchAllUsers(ids) {
  return Promise.all(ids.map(id => fetchUser(id)));
  // 100 ta user, lekin parallel = ~200ms
}

// ✅ Yaxshiroq — concurrent limit bilan (server ni bosmaslik uchun)
async function fetchAllUsers(ids) {
  const limit = pLimit(10); // max 10 ta parallel
  return Promise.all(
    ids.map(id => limit(() => fetchUser(id)))
  );
  // 100 ta user, 10 ta parallel = ~2 sekund (server xavfsiz)
}
```

**Nima uchun:** Loop ichida `await` har bir iteratsiyani ketma-ket bajaradi. Agar so'rovlar mustaqil bo'lsa — `Promise.all` yoki concurrent limit ishlatamiz.

---

### ❌ Xato 4: Error Handling Yo'q (Unhandled Rejection)

```javascript
// ❌ Yomon — xato catch qilinmagan
async function loadData() {
  const response = await fetch('/api/data'); // Network xato bo'lsa?
  const data = await response.json();        // Invalid JSON bo'lsa?
  return data;
}

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
    return null;
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
});
// Browser:
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled Rejection:', event.reason);
  event.preventDefault();
});
```

**Nima uchun:** Catch qilinmagan Promise rejection dasturni crash qilishi mumkin (Node.js 15+ da default behavior). Doim error handling qo'shing.

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
  });
  return { success: true }; // Javobni kutmasdan qaytarish
}
```

**Nima uchun:** Async funksiyani `await` siz chaqirish — uning natijasini va xatolarini yo'qotish. Dastur "muvaffaqiyat" deb o'ylaydi, lekin database ga hech narsa yozilmagan bo'lishi mumkin.

---

### ❌ Xato 6: return await — Kerak/Keraksiz

```javascript
// ❌ Ortiqcha — return await kerak emas (try/catch bo'lmasa)
async function getUser(id) {
  return await fetchUser(id); // await kerak emas
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
  async constructor(url) { // SyntaxError!
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
        results[index] = err;
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
async function getOrderSummary(orderId) {
  // 1-qadam: order olish (boshqalar bunga bog'liq)
  const order = await fetchOrder(orderId);

  // 2-qadam: user va products/reviews PARALLEL (bir-biriga bog'liq emas)
  const [user, products, reviews] = await Promise.all([
    fetchUser(order.userId),
    Promise.all(order.productIds.map(id => fetchProduct(id))),
    Promise.all(order.productIds.map(id => fetchReviews(id))),
  ]);

  // 3-qadam: shipping — order va user.address ga bog'liq
  const shipping = await calculateShipping(order, user.address);

  return { order, user, products, reviews, shipping };
}

// Performance taqqoslash:
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

**Savol:** `resilientFetch(sources)` funksiyasini yozing:
1. Object'dagi barcha async funksiyalarni parallel chaqiring
2. Har bir xatoni alohida handle qiling (xato bo'lsa `null` qo'ying)
3. Agar hammasi xato bersa — `AggregateError` throw qiling
4. Kamida bitta ishlaganda — natija qaytaring

```javascript
// Test:
const result = await resilientFetch({
  user: () => fetch('/api/user').then(r => r.json()),
  posts: () => fetch('/api/posts').then(r => r.json()),
  stats: () => fetch('/api/stats').then(r => r.json()),
});
// Agar posts xato bersa:
// result = { user: {...}, posts: null, stats: {...} }
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
function mockFetch(name, shouldFail = false, delayMs = 100) {
  return async () => {
    await new Promise(r => setTimeout(r, delayMs));
    if (shouldFail) throw new Error(`${name} server xato`);
    return { name, data: `${name} data` };
  };
}

// Aralash natija:
const result = await resilientFetch({
  user: mockFetch('user'),
  posts: mockFetch('posts', true),
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
// Ketma-ket! 2-chi faqat 1-chi tugagandan keyin boshlandi
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

2. **`await` Promise resolve bo'lguncha funksiyani to'xtatadi** — lekin main thread ni BLOKLAMAYDI. Event loop erkin qoladi.

3. **try/catch** — async/await bilan xatolarni sinxron ko'rinishda handle qilish. `.catch()` zanjirlariga hojat yo'q.

4. **Mustaqil operatsiyalar → `Promise.all`** bilan parallel bajaring. Performance farqi katta — N ta operatsiya vaqtini `max(operatsiya)` ga tushiradi.

5. **Sequential faqat bog'liqlik bo'lganda.** Keyingi operatsiya oldingi natijaga bog'liq bo'lsa — ketma-ket. Aks holda — parallel.

6. **Top-level await** — ES Modules ichida `async` funksiya siz `await` ishlatish. Modul loading ni "async" qiladi.

7. **Production patternlar:** retry (exponential backoff), timeout (AbortController), concurrent limit (server himoyasi), queue (tartibli bajarish).

8. **`for-await-of`** — async iterablelar ustida iteratsiya. Stream, paginated API, real-time data uchun.

9. **Under the hood:** async/await = generator + promise. V8 7.2+ da optimallashtirilgan — extra microtick yo'q.

10. **Eng katta xatolar:** forEach ichida await (ishlamaydi), keraksiz sequential await (parallel kerak), error handling yo'q (unhandled rejection), async constructor (mumkin emas).

---

> **Oldingi bo'lim:** [12-promises.md](12-promises.md) — Promises: state machine, constructor, chaining, static methods, microtask queue.
>
> **Keyingi bo'lim:** [14-iterators-generators.md](14-iterators-generators.md) — Iterators va Generators: Symbol.iterator, yield, async generators, lazy evaluation.
