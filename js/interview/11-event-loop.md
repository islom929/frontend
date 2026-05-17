# Event Loop — Interview Savollari

> Event Loop algoritmi, microtask/macrotask farqi, setTimeout(fn, 0), requestAnimationFrame, Node.js Event Loop, starvation, UI blocking, va output tartibini aniqlash haqida interview savollari.

---

## Nazariy savollar

### 1. JavaScript nima uchun single-threaded? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

JavaScript single-threaded dizaynining **ikki tarixiy sababi** bor:

1. **Implementation simplicity (asosiy)** — Netscape 1995-da Brendan Eich JavaScript prototipini juda qisqa muddat ichida yaratishi kerak edi. Multi-threaded runtime yaratish ancha murakkab va ko'p vaqt talab qilardi — single-threaded eng oddiy va tezkor yechim edi.

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
<summary><strong>Javob</strong></summary>

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

V8 engine Event Loop'ni implement qilmaydi — u faqat JavaScript kodni compile va execute qiladi. Event Loop browser uchun HTML Standard (WHATWG) "Event loops" bo'limida, Node.js uchun esa libuv kutubxonasida implement qilingan.

</details>

### 3. Microtask va Macrotask farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
- HTML Standard "Timers" bo'limi: nesting level 5+ bo'lganda minimum delay **4ms** (clamping)
- Background tab'larda browser'lar `setTimeout` ni throttle qiladi (Chrome — taxminan 1 soniya, Firefox — implementation-defined)
- Amalda OS scheduler resolution tufayli timer trigger 1-4ms kechikishi mumkin

</details>

### 5. Promise constructor sync ishlashini tushuntiring [Middle]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
| Spec | HTML Standard (WHATWG) | libuv (C library) |
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

`process.nextTick` va `setImmediate` nomlari teskari qo'yilgan — bu tarixiy noqulaylik. `nextTick` darhol (har faza orasida) ishlaydi, `setImmediate` esa check fazada (keyinroq) ishlaydi. Top-level'da `setTimeout(fn,0)` vs `setImmediate` tartibi **garanti emas** — OS timer resolution'ga bog'liq.

</details>

### 9. UI blocking muammosini qanday hal qilasiz? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

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

**4. `scheduler.postTask` — yangi browser API (Chromium 94+):**

```javascript
// Yangi standart: task'larga priority berish
scheduler.postTask(processItems, { priority: "user-blocking" }); // tezda
scheduler.postTask(prefetch, { priority: "background" });          // bo'sh vaqtda
```

Real production'da ko'pincha bir nechta usul birgalikda ishlatiladi: debounce + Web Worker + batch rendering + (kelajakda) `scheduler.postTask` priority hint'lari.

</details>

### 10. queueMicrotask() nima va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

Node.js rasmiy hujjatlari `queueMicrotask`'ni tavsiya qiladi — sabablari ikkita: u cross-platform (browser + Node.js), va `process.nextTick`'ning starvation xavfi yuqoriroq.

`process.nextTick` har bir Event Loop faza **orasida** ishlaydi. Recursive chaqirilganda — nextTick queue hech qachon bo'shamasa — `libuv` keyingi fazaga (timers, poll, check) **umuman o'tmaydi**. I/O callback'lar, timer callback'lari bloklanadi.

`queueMicrotask` esa V8'ning standart MicrotaskQueue'siga callback qo'shadi. Bu queue ham starvation berishi mumkin (Node 11+ har JS callback'dan keyin drain qilinadi), lekin `nextTick`'dan kamroq xavfli, chunki Node.js'da `nextTick` queue alohida (eng yuqori) priority'ga ega.

</details>

</details>

### 12. requestIdleCallback nima va requestAnimationFrame dan farqi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

`requestIdleCallback`'ga berilgan callback `IdleDeadline` object'ini argument sifatida oladi. `deadline.timeRemaining()` frame tugashigacha qolgan vaqtni millisekundlarda qaytaradi — spec bo'yicha maksimal 50ms. Browser bu deadline'ni harakatdagi frame budjeti asosida hisoblaydi.

Agar browser band bo'lsa va `timeout` parametri berilgan bo'lsa — timeout o'tgandan keyin `timeRemaining()` 0 qaytaradi, lekin callback baribir chaqiriladi. `deadline.didTimeout` flag esa `true` bo'ladi — kod shu holatni tekshirib, ish miqdorini moslab olish mumkin.

**Browser support:** Chrome 47+, Firefox 55+. Safari (WebKit) hozirgi vaqtda implement qilmagan — fallback sifatida `setTimeout(fn, 1)` yoki `MessageChannel` ishlatish kerak.

</details>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

`return Promise.resolve()` bilan return qilish muhim — bu `.then(9)`'ni faqat shu Promise resolve bo'lgandagina queue'ga qo'shadi. Oddiy `return undefined` bo'lganida `.then(9)` darhol microtask queue'ga tushardi.

`return Promise.resolve()` qo'shimcha microtask tick kutadi — spec bo'yicha `PromiseResolveThenableJob` yaratiladi (thenable resolve qilish uchun). Bu spec evolution:

- **Eski spec (ES2018 gacha):** 2 ta qo'shimcha microtask tick (`NewPromiseResolveThenableJob` + `PromiseReactionJob`)
- **ES2019+ (V8 7.2+):** 1 ta qo'shimcha microtask tick — optimization: native `Promise` resolve qilinayotganda `PromiseResolve` shortcut qo'llaniladi

Bu farq **shu savol natijasiga ta'sir qilmaydi** (tartib bir xil), lekin boshqa murakkab interleaving'larda muhim bo'lishi mumkin. Interview'da "spec version'ga qarab tick soni farq qilishi mumkin" deb eslatish bilim chuqurligini ko'rsatadi.

</details>

</details>

### 4. `MessageChannel` qanday ishlaydi va qachon ishlatiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`MessageChannel` — ikkita port (`port1`, `port2`) yaratadigan Web API. Bir port'ga `postMessage` chaqirilsa, ikkinchi port'ning `onmessage` handler'i **macrotask** sifatida ishga tushadi. `setTimeout(fn, 0)`'ga muqobil — lekin **4ms nested clamping**'siz, true zero-delay scheduling beradi.

### To'liq tushuntirish

HTML spec'ning "minimum timer interval" qoidasi faqat `setTimeout`/`setInterval` task source'iga tegishli. `MessageChannel.postMessage` boshqa task source'da bo'lgani uchun clamping qo'llanilmaydi. Bu fact high-performance scheduling library'lar (masalan React Scheduler) tomonidan ishlatiladi.

Muhim: `MessageChannel` callback'i ham **macrotask** — microtask'lar va rendering undan oldin ishlaydi. Microtask tezligida kerak bo'lsa `queueMicrotask` ishlatish kerak.

### Kod misol

```javascript
function scheduleWithMessageChannel(fn) {
  const channel = new MessageChannel();
  channel.port1.onmessage = fn;
  channel.port2.postMessage(null);
}

// setTimeout(0) — nested chain'da 4ms clamping
function scheduleWithTimeout(fn) {
  setTimeout(fn, 0);
}

console.time("setTimeout(0)");
scheduleWithTimeout(() => console.timeEnd("setTimeout(0)"));
// Nested 6+ darajada chaqirilganda: ~4ms

console.time("MessageChannel");
scheduleWithMessageChannel(() => console.timeEnd("MessageChannel"));
// Clamping yo'q — minimum delay sezilarli darajada past

// Real use case: batched DOM updates
const channel = new MessageChannel();
let pendingUpdates = [];

channel.port1.onmessage = () => {
  const updates = pendingUpdates;
  pendingUpdates = [];
  updates.forEach(applyUpdate);
};

function scheduleUpdate(update) {
  if (pendingUpdates.length === 0) {
    channel.port2.postMessage(null); // zero-delay schedule
  }
  pendingUpdates.push(update);
}
```

### Edge Cases

- **Cross-origin iframe'larda** — `MessageChannel` postMessage cross-context kommunikatsiya uchun mo'ljallangan, lekin scheduling pattern bitta context'da ham ishlaydi
- **Port transferring** — port `Transferable` interfeysni implement qiladi, `postMessage`'ning ikkinchi argumenti orqali boshqa context'ga uzatish mumkin
- **Garbage Collection** — port'larga reference saqlanmasa, channel GC qilinadi va callback'lar ishlamaydi

### Follow-up savollar

1. **MessageChannel macrotask'mi yoki microtask'mi?** — Macrotask. Rendering va microtask'lar undan oldin ishlaydi.
2. **React Scheduler nima uchun MessageChannel ishlatadi?** — Concurrent mode'da time-slicing uchun; `setTimeout` 4ms clamping va background tab throttle'iga uchraydi.

<details>
<summary><strong>Deep Dive</strong></summary>

`MessageChannel` HTML Standard'da "Message channels" bo'limida belgilangan. Har port `MessagePort` interfeysini implement qiladi va `Transferable` — `postMessage`'ning ikkinchi argumenti orqali boshqa context'ga (Worker, iframe) o'tkazilishi mumkin.

`postMessage(null)` callback `port.onmessage`'ga **"posted message" task source** orqali yetkaziladi. HTML spec timer clamping qoidasini faqat **"timer task source"**'ga (setTimeout/setInterval) qo'llaydi — bu boshqa task source bo'lgani uchun 4ms cheklovi yo'q.

React 18+ `Scheduler` paketi shu mexanizmni ishlatadi: `scheduler/src/forks/SchedulerDOM.js`'da `MessageChannel` orqali yield qilinadi (time-slicing). `MessageChannel` mavjud bo'lmagan environment'larda (eski Node.js, JSDOM ba'zi versiyalari) fallback sifatida `setTimeout(fn, 0)` ishlatiladi.

</details>

</details>

---

### 5. Quyidagi kodda nima xato va qanday tuzatasiz? [Middle+]

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
<summary><strong>Javob</strong></summary>

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

---

### 6. Node.js da quyidagi kodning output'ini ayting [Senior]

**Savol:**

```javascript
const fs = require("fs");

console.log("1");

setImmediate(() => console.log("2"));
process.nextTick(() => console.log("3"));
Promise.resolve().then(() => console.log("4"));

fs.readFile(__filename, () => {
  console.log("5");
  setImmediate(() => console.log("6"));
  process.nextTick(() => console.log("7"));
  Promise.resolve().then(() => console.log("8"));
});

setTimeout(() => console.log("9"), 0);

console.log("10");
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output: **1, 10, 3, 4, 9, 2, 5, 7, 8, 6**

`process.nextTick` har faza orasida birinchi, microtask ikkinchi. `setImmediate` check fazada, `setTimeout(0)` timers fazada. I/O callback (poll fazada) ichida `setImmediate` doim `setTimeout(0)` dan oldin.

### To'liq tushuntirish

```
SYNC: "1", "10"

Sync tugagandan keyin event loop boshlanadi.
Har faza orasida: nextTick queue → microtask queue:
  "3" — process.nextTick
  "4" — Promise.then

Timers fazasi:
  "9" — setTimeout(0)
  (top-level setImmediate va setTimeout tartibi OS timer resolution'ga bog'liq —
   bu yerda setTimeout oldin keladi deb hisoblaymiz)

Check fazasi:
  "2" — top-level setImmediate

Poll fazasi (I/O):
  "5" — fs.readFile callback
  Faza ichida sinxron tugagach, nextTick + microtask:
  "7" — process.nextTick
  "8" — Promise.then

Check fazasi (poll'dan keyin):
  "6" — fs.readFile ichida ro'yxatga olingan setImmediate
```

### Edge Cases

- **Top-level setTimeout(0) vs setImmediate** — tartib **garanti emas**, OS timer resolution'ga bog'liq. Test environment'da har ikki natija ham mumkin
- **I/O callback ichida** — `setImmediate` **DOIM** `setTimeout(0)` dan oldin (poll → check tartibi)
- **`process.nextTick` recursion** — I/O callback'larni umuman bloklab qo'yishi mumkin (starvation)

### Follow-up savollar

1. **Nima uchun top-level'da setTimeout vs setImmediate tartibi noaniq?** — `setTimeout(fn, 0)` aslida ~1ms (Node implementation). Agar event loop boshlanganda 1ms o'tgan bo'lsa — timer ready, oldin keladi. O'tmagan bo'lsa — setImmediate avval.

<details>
<summary><strong>Deep Dive</strong></summary>

`libuv` har fazani ketma-ket bajaradi: timers → pending → idle/prepare → poll → check → close. Har faza tugagach, **`uv_run` ichida** `process_nextTick_queue()` va microtask queue chaqiriladi.

`process.nextTick` Node.js'ning `_tickInfo` internal struktura orqali boshqariladi — bu **C++ darajadagi queue**, V8 microtask queue'sidan alohida. `_tickInfo.length > 0` bo'lganda V8 har JS callback'dan keyin nextTick'larni drain qiladi.

Microtask queue esa V8'ning standart `MicrotaskQueue` — Node.js V8 embedder API orqali `EnqueueMicrotask`/`PerformCheckpoint` chaqiradi. Node.js v11+ dan boshlab microtask checkpoint har JS callback'dan keyin amalga oshiriladi (oldin faza oxirida edi — bu xatti-harakat o'zgarishi).

</details>

</details>
