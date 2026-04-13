# Event Loop — Interview Savollari

> Event Loop algoritmi, microtask/macrotask farqi, setTimeout(fn, 0), requestAnimationFrame, Node.js Event Loop, starvation, UI blocking, va output tartibini aniqlash haqida interview savollari.

---

## Nazariy savollar

### 1. JavaScript nima uchun single-threaded? [Junior+]

<details>
<summary>Javob</summary>

JavaScript single-threaded dizaynining **ikki tarixiy sababi** bor:

1. **Implementation simplicity (asosiy)** — Netscape 1995-da Brendan Eich'ga JavaScript'ni 10 kun ichida yozishni buyurgan. Multi-threaded runtime yaratish murakkab va ko'p vaqt talab qilardi — single-threaded eng oddiy va tezkor yechim edi.

2. **DOM concurrency safety (post-hoc justification)** — keyinchalik muhim: agar ikkita thread bir vaqtda bitta DOM elementni o'zgartirmoqchi bo'lsa, **race condition** yuzaga keladi. Single-threaded model DOM API dizaynini ham ancha soddalashtirdi.

Single-threaded dizaynning afzalliklari:
- **DOM safety** — bir vaqtda bitta thread DOM ga yozadi
- **Simplicity** — deadlock, mutex muammolari yo'q
- **Predictability** — kod ketma-ket bajariladi

```javascript
// JavaScript single-threaded — kod KETMA-KET bajariladi
console.log("1");
console.log("2");
console.log("3");
// Output har doim: 1, 2, 3 — parallel execution yo'q
```

Lekin JavaScript o'zi single-threaded bo'lsa-da, runtime (browser/Node.js) **multi-threaded**. `setTimeout`, `fetch` kabi operatsiyalar browser'ning boshqa thread'larida bajariladi.

</details>

### 2. Event Loop nima va qanday ishlaydi? [Middle]

<details>
<summary>Javob</summary>

Event Loop — Call Stack va Queue'lar o'rtasidagi aloqani boshqaradigan cheksiz loop. U quyidagi algoritmni takrorlaydi:

1. **Call Stack bo'shmi?** — agar yo'q, kutamiz
2. **Microtask Queue** ni **TO'LIQ** bo'shat (yangi qo'shilganlarni ham)
3. **Macrotask Queue** dan **BITTA** task ol va bajir
4. **Render** (kerak bo'lsa — ~16.6ms oraliqda)
5. 1-qadamga qayt

```javascript
console.log("sync");                          // 1. Call Stack
setTimeout(() => console.log("macro"), 0);    // → Macrotask Queue
Promise.resolve().then(() => console.log("micro")); // → Microtask Queue

// Output: sync, micro, macro
// Sync → Microtask → Macrotask — har doim shu tartibda
```

Eng muhim nuqta — **asimmetriya**: microtask'lar **barchasi** bajariladi, macrotask'dan esa **bittasi** olinadi.

**Deep Dive:**

V8 engine Event Loop ni implement qilmaydi — u faqat JavaScript kodni compile va execute qiladi. Event Loop browser uchun HTML spec da, Node.js uchun libuv kutubxonasida implement qilingan.

</details>

### 3. Microtask va Macrotask farqi nima? [Middle]

<details>
<summary>Javob</summary>

| Xususiyat | Microtask | Macrotask |
|-----------|-----------|-----------|
| **Misollar** | Promise.then, queueMicrotask, MutationObserver | setTimeout, setInterval, I/O, MessageChannel |
| **Nechta bajariladi** | **BARCHASI** (har tick'da) | **BITTADAN** (har tick'da) |
| **Priority** | Yuqori | Past |
| **Render ga ta'siri** | Render dan **OLDIN** | Render bilan **NAVBATMA** |
| **Starvation xavfi** | **BOR** (recursive microtask) | **YO'Q** |

```javascript
setTimeout(() => {
  console.log("Macrotask 1");
  Promise.resolve().then(() => console.log("Microtask from Macro 1"));
}, 0);

setTimeout(() => {
  console.log("Macrotask 2");
}, 0);

// Output:
// Macrotask 1
// Microtask from Macro 1    ← Macrotask 2 dan OLDIN!
// Macrotask 2

// Har bir macrotask'dan keyin microtask checkpoint bo'ladi
```

</details>

### 4. setTimeout(fn, 0) nima uchun darhol bajarmaydi? [Middle]

<details>
<summary>Javob</summary>

`setTimeout(fn, 0)` callback'ni **Macrotask Queue** ga qo'yadi. U faqat:
1. Sync kod tugagandan keyin
2. Barcha microtask'lar bajarilgandan keyin
3. Event Loop macrotask queue'ga navbat kelganda bajariladi

```javascript
setTimeout(() => console.log("setTimeout"), 0);
Promise.resolve().then(() => console.log("Promise"));
console.log("Sync");

// Output: Sync, Promise, setTimeout
// Promise (microtask) DOIM setTimeout (macrotask) dan oldin
```

Qo'shimcha nuqtalar:
- HTML spec bo'yicha nested setTimeout 5+ marta chaqirilganda minimum delay **4ms** (clamping)
- Background tab'larda minimum **1000ms** gacha throttle qilinishi mumkin
- Amalda OS scheduler tufayli ~1-4ms delay bo'ladi

</details>

### 5. Promise constructor sync ishlashini tushuntiring [Middle]

<details>
<summary>Javob</summary>

`new Promise(executor)` dagi `executor` funksiya **synchronous** tarzda darhol chaqiriladi. Faqat `.then()`, `.catch()`, `.finally()` callback'lari async (microtask sifatida) ishlaydi.

```javascript
const p = new Promise((resolve) => {
  console.log("1 — sync!");     // ← Darhol ishlaydi
  resolve("done");
  console.log("2 — sync!");     // ← resolve() funksiyani TO'XTATMAYDI
});

p.then(val => console.log("3 — async:", val));
console.log("4 — sync");

// Output: 1, 2, 4, 3
// Promise constructor: SYNC
// resolve(): Promise holatini o'zgartiradi, lekin funksiyani to'xtatmaydi
// .then(): MICROTASK — sync koddan keyin
```

</details>

### 6. requestAnimationFrame Event Loop'da qayerda turadi? [Middle+]

<details>
<summary>Javob</summary>

`requestAnimationFrame` (rAF) na microtask, na macrotask — u Event Loop'ning **rendering bosqichida** ishlaydi.

```
Event Loop tick:
1. [Macrotask] — bitta
2. [Microtask Queue] — barchasini bajir
3. [Rendering]:
   a. rAF callbacks  ← SHU YERDA
   b. Style recalculation
   c. Layout
   d. Paint
4. Keyingi tick ga qayt
```

```javascript
console.log("sync");
requestAnimationFrame(() => console.log("rAF"));
setTimeout(() => console.log("setTimeout"), 0);
Promise.resolve().then(() => console.log("promise"));

// Ko'p hollarda: sync, promise, setTimeout, rAF
// rAF render bosqichida — macrotask'dan ham keyin
// Lekin tartib browser va frame timing'ga qarab o'zgarishi mumkin
```

rAF animatsiya uchun ideal:
- Browser render sikli bilan sinxronlashgan (~60fps)
- Background tab'da avtomatik to'xtaydi (battery tejaydi)
- `setTimeout(fn, 16)` dan aniqroq

</details>

### 7. Starvation nima va qanday oldini olish mumkin? [Middle+]

<details>
<summary>Javob</summary>

**Starvation** — microtask queue hech qachon bo'sh bo'lmaydigan holat. Har bir microtask yangi microtask qo'shsa, Event Loop macrotask'larga va rendering'ga **hech qachon** o'tmaydi — UI muzlab qoladi.

```javascript
// ❌ STARVATION — browser muzlaydi
function bad() {
  Promise.resolve().then(() => {
    bad(); // Har safar yangi microtask → cheksiz loop
  });
}
bad();
// setTimeout, render, click — HECH NARSA ishlamaydi

// ✅ XAVFSIZ — setTimeout bilan
function good() {
  setTimeout(() => {
    good(); // Macrotask — orada render va boshqa ishlar bajariladi
  }, 0);
}
```

Oldini olish qoidalari:
- Recursive `Promise.then()` / `queueMicrotask` ishlatmang
- Uzoq loop'larni `setTimeout(fn, 0)` bilan batch'lang
- Node.js da `setImmediate` ishlating (`process.nextTick` emas)

</details>

### 8. Node.js Event Loop browser'nikidan qanday farq qiladi? [Middle+]

<details>
<summary>Javob</summary>

Node.js Event Loop **6 ta faza (phase)** dan iborat (browser'da faqat macrotask → microtask → render):

1. **Timers** — setTimeout, setInterval
2. **Pending Callbacks** — system-level I/O
3. **Idle/Prepare** — ichki
4. **Poll** — I/O events (eng muhim faza)
5. **Check** — setImmediate
6. **Close** — socket.on('close')

**Har bir faza orasida:** `process.nextTick` → microtask queue bo'shatiladi.

| Xususiyat | Browser | Node.js |
|-----------|---------|---------|
| Spec | HTML Spec | libuv (C) |
| Fazalar | 1 (macrotask→micro→render) | 6 ta faza |
| rAF | ✅ | ❌ (DOM yo'q) |
| setImmediate | ❌ | ✅ |
| process.nextTick | ❌ | ✅ (eng yuqori priority) |

```javascript
const fs = require("fs");

// I/O callback ichida — setImmediate DOIM setTimeout dan oldin
fs.readFile(__filename, () => {
  setTimeout(() => console.log("setTimeout"), 0);
  setImmediate(() => console.log("setImmediate"));
});
// HAR DOIM: setImmediate, setTimeout
// Poll faza → Check faza (setImmediate) → Timers faza (setTimeout)
```

**Deep Dive:**

`process.nextTick` va `setImmediate` nomlari teskari qo'yilgan — tarixiy xato. `nextTick` darhol (har faza orasida) ishlaydi, `setImmediate` esa check fazada (keyinroq) ishlaydi. Top-level da `setTimeout(fn,0)` vs `setImmediate` tartibi **garanti emas** (OS timer resolution ga bog'liq).

</details>

### 9. UI blocking muammosini qanday hal qilasiz? [Middle+]

<details>
<summary>Javob</summary>

UI blocking — og'ir hisoblash main thread'ni to'xtatganda yuzaga keladi. Yechimlar:

**1. Batch processing — setTimeout bilan:**

```javascript
function processInBatches(items, batchSize, fn) {
  let i = 0;
  function next() {
    const end = Math.min(i + batchSize, items.length);
    while (i < end) fn(items[i++]);
    if (i < items.length) setTimeout(next, 0);
    // Har batch orasida render va event'lar ishlaydi
  }
  next();
}
```

**2. Web Worker — alohida thread:**

```javascript
// main.js
const worker = new Worker("worker.js");
worker.postMessage(data);
worker.onmessage = (e) => renderResults(e.data);

// worker.js — alohida thread, UI bloklanmaydi
self.onmessage = (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};
```

**3. Debounce — ortiqcha chaqiruvlarni kamaytirish:**

```javascript
function debounce(fn, delay) {
  let id;
  return (...args) => {
    clearTimeout(id);
    id = setTimeout(() => fn(...args), delay);
  };
}

input.addEventListener("input", debounce(filterList, 300));
```

Real production'da ko'pincha uchala usul birgalikda ishlatiladi: debounce + Web Worker + batch rendering.

</details>

### 10. queueMicrotask() nima va qachon ishlatiladi? [Middle]

<details>
<summary>Javob</summary>

`queueMicrotask(callback)` — microtask queue'ga to'g'ridan-to'g'ri callback qo'shish uchun API. `Promise.resolve().then(callback)` bilan deyarli bir xil, lekin Promise object yaratmaydi.

```javascript
queueMicrotask(() => console.log("qM"));
Promise.resolve().then(() => console.log("Promise"));
console.log("sync");

// Output: sync, qM, Promise
// Ikkalasi ham microtask — FIFO tartibda
```

Qachon ishlatiladi:
- Microtask queue'ga callback qo'shish kerak, lekin Promise semantikasi kerak emas
- Bir nechta sinxron o'zgarishlarni batch'lab bitta microtask'da qayta ishlash
- Sync koddan keyin, lekin rendering va macrotask'lardan oldin ish bajarish

</details>

### 11. process.nextTick() va queueMicrotask() farqi nima? (Node.js) [Senior]

<details>
<summary>Javob</summary>

| Xususiyat | `process.nextTick()` | `queueMicrotask()` |
|-----------|---------------------|---------------------|
| **Platform** | Faqat Node.js | Browser + Node.js |
| **Priority** | **Eng yuqori** — microtask'dan HAM oldin | Oddiy microtask priority |
| **Queue** | nextTick Queue (alohida) | Microtask Queue |
| **Starvation** | Xavfliroq — barcha fazalarni to'xtatadi | Xavfli, lekin kamroq |
| **Spec** | Node.js specific | HTML/ECMA spec |

```javascript
// Node.js da
process.nextTick(() => console.log("nextTick"));
queueMicrotask(() => console.log("queueMicrotask"));
Promise.resolve().then(() => console.log("Promise"));

// Output:
// nextTick          ← eng birinchi (har faza orasida)
// queueMicrotask    ← microtask queue (FIFO)
// Promise           ← microtask queue (FIFO)
```

**Deep Dive:**

Node.js hujjatlari o'zi `queueMicrotask` ni tavsiya qiladi — chunki u cross-platform (browser + Node.js) va `process.nextTick` ning starvation xavfi yuqoriroq. `process.nextTick` har bir Event Loop faza **orasida** ishlagani uchun, recursive chaqirilganda I/O callback'lar umuman bajarilmasligi mumkin.

</details>

### 12. requestIdleCallback nima va requestAnimationFrame dan farqi? [Senior]

<details>
<summary>Javob</summary>

`requestIdleCallback` — browser **bo'sh vaqtida** (idle) callback bajaradigan API. `requestAnimationFrame` dan farqli o'laroq, u **past priority'li** ishlar uchun — analytics, prefetching, lazy initialization.

| | `requestAnimationFrame` | `requestIdleCallback` |
|---|---|---|
| **Qachon** | Har render **oldidan** | Browser **bo'sh** bo'lganda |
| **Priority** | Yuqori | Past |
| **Maqsad** | Animatsiya, DOM update | Analytics, prefetch, lazy init |
| **Garanti** | Har frame'da (~60fps) | **Garanti yo'q** — band bo'lsa chaqirilmasligi mumkin |
| **timeout** | Yo'q | Bor — `{ timeout: 2000 }` |

```javascript
// rAF — har frame'da, yuqori priority
requestAnimationFrame(() => {
  element.style.transform = `translateX(${pos}px)`;
});

// rIC — bo'sh vaqtda, past priority
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 5) {
    sendAnalyticsEvent(); // Foydalanuvchi kutmaydi
  }
}, { timeout: 3000 }); // 3 soniya ichida garanti
```

**Deep Dive:**

`requestIdleCallback` ga berilgan callback `IdleDeadline` object oladi. `deadline.timeRemaining()` frame tugashigacha qolgan vaqtni (ms) qaytaradi. Odatda bu 0-50ms oraliqda. Agar browser band bo'lsa va `timeout` berilgan bo'lsa — timeout o'tgandan keyin `timeRemaining()` 0 qaytaradi, lekin callback baribir chaqiriladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve()
  .then(() => console.log("C"))
  .then(() => console.log("D"));
Promise.resolve().then(() => console.log("E"));
console.log("F");
```

<details>
<summary>Javob</summary>

Output: **A, F, C, E, D, B**

```
Sync phase: "A", "F"

Microtask Queue: [C_callback, E_callback]
→ C bajariladi → "C" → .then(D) queue ga: [E_callback, D_callback]
→ E bajariladi → "E"
→ D bajariladi → "D"

Macrotask: B bajariladi → "B"
```

C va E **ikkalasi ham birinchi** .then() callback — ular FIFO tartibida bajariladi. D esa C bajarilgandan keyin queue ga tushadi, shuning uchun E dan keyin.

</details>

### 2. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
async function runAsync() {
  console.log("1");
  await Promise.resolve();
  console.log("2");
}

console.log("3");
runAsync();
console.log("4");
setTimeout(() => console.log("5"), 0);
Promise.resolve().then(() => console.log("6"));
```

<details>
<summary>Javob</summary>

Output: **3, 1, 4, 2, 6, 5**

```
Sync: "3"
runAsync() chaqirildi:
  "1" — sync (async funksiyaning await gacha bo'lgan qismi sync)
  await Promise.resolve() — runAsync to'xtaydi → microtask: [runAsync_resume]
"4" — sync (main script davom etadi)

setTimeout(5) → Macrotask Queue
Promise.then(6) → Microtask Queue: [runAsync_resume, then_6]

Microtask checkpoint:
  runAsync_resume → "2"
  then_6 → "6"

Macrotask: "5"
```

`await` faqat o'sha async funksiya ichini to'xtatadi — tashqi sync kod davom etadi. `await` dan keyingi kod `.then()` callback bilan teng.

</details>

### 3. Quyidagi murakkab kodning output'ini ayting [Senior]

**Savol:**

```javascript
console.log("1");

setTimeout(() => {
  console.log("2");
  new Promise(resolve => {
    console.log("3");
    resolve();
  }).then(() => console.log("4"));
  setTimeout(() => console.log("5"), 0);
}, 0);

new Promise(resolve => {
  console.log("6");
  resolve();
}).then(() => {
  console.log("7");
  setTimeout(() => console.log("8"), 0);
  return Promise.resolve();
}).then(() => {
  console.log("9");
});

console.log("10");
```

<details>
<summary>Javob</summary>

Output: **1, 6, 10, 7, 9, 2, 3, 4, 8, 5**

```
SYNC: "1", "6" (Promise constructor sync!), "10"

Microtask: then_7 → "7"
  → setTimeout(8) → Macrotask: [setTimeout_2, setTimeout_8]
  → return Promise.resolve() → then_9 microtask queue ga
Microtask: then_9 → "9"

Macrotask: setTimeout_2 → "2"
  → Promise constructor → "3" (sync)
  → .then(4) → Microtask: [then_4]
  → setTimeout(5) → Macrotask: [setTimeout_8, setTimeout_5]
Microtask: then_4 → "4"

Macrotask: setTimeout_8 → "8"
Macrotask: setTimeout_5 → "5"
```

**Deep Dive:**

`return Promise.resolve()` bilan return qilish muhim — bu `.then(9)` ni faqat shu Promise resolve bo'lgandagina queue ga qo'shadi. Oddiy `return undefined` bo'lganida `.then(9)` darhol queue ga tushardi.

`return Promise.resolve()` esa qo'shimcha microtask tick kutadi — spec bo'yicha `PromiseResolveThenableJob` yaratiladi (thenable resolve qilish uchun). Bu **spec evolution** bor:
- **Eski spec (ES2018 gacha):** 2 extra microtask tick (NewPromiseResolveThenableJob + PromiseReactionJob)
- **ES2019+ (V8 7.2+):** 1 extra microtask tick — optimization: native `Promise` resolve qilinayotganda `PromiseResolve` shortcut qo'llaniladi

Bu farq **shu savol natijasiga ta'sir qilmaydi** (tartib bir xil), lekin boshqa murakkab interleaving'larda muhim bo'lishi mumkin — shuning uchun interview'da "spec version'ga qarab tick soni farq qilishi mumkin" deb eslatish yaxshi.

</details>

### 4. Quyidagi kodda nima xato va qanday tuzatasiz? [Middle+]

**Savol:**

```javascript
// Foydalanuvchi "Load" tugmasini bosganda 50,000 ta yozuvni ko'rsatish
loadBtn.addEventListener("click", () => {
  const records = fetchRecordsSync(); // 50,000 ta yozuv

  records.forEach(record => {
    const row = document.createElement("tr");
    row.textContent = `${record.name} | ${record.email}`;
    table.appendChild(row);
  });
});
```

<details>
<summary>Javob</summary>

**Muammo:** 50,000 ta DOM element yaratish va qo'shish — hammasi **sync** va **bitta frame** ichida. Bu UI ni bir necha soniya muzlatadi.

**Xatolar:**
1. `fetchRecordsSync()` — sync data fetch (I/O blocking)
2. `table.appendChild` har safar chaqirilganda reflow trigger qiladi
3. Hamma ish bitta macrotask ichida — render imkoniyati yo'q

```javascript
// ✅ Tuzatilgan versiya
loadBtn.addEventListener("click", async () => {
  const records = await fetchRecords(); // ✅ Async fetch

  const BATCH = 500;
  let i = 0;

  function renderBatch() {
    const fragment = document.createDocumentFragment(); // ✅ Batch DOM
    const end = Math.min(i + BATCH, records.length);

    while (i < end) {
      const row = document.createElement("tr");
      const td1 = document.createElement("td");
      td1.textContent = records[i].name;
      const td2 = document.createElement("td");
      td2.textContent = records[i].email;
      row.appendChild(td1);
      row.appendChild(td2);
      fragment.appendChild(row);
      i++;
    }

    table.appendChild(fragment); // ✅ Bir marta DOM ga qo'shish

    if (i < records.length) {
      requestAnimationFrame(renderBatch); // ✅ Har frame'da batch
    }
  }

  renderBatch();
});
```

</details>
