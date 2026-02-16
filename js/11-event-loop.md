# Bo'lim 11: Event Loop — Miyaning Asosi

> Event Loop — JavaScript ning yagona thread'da asinxron operatsiyalarni boshqaradigan mexanizmi. Async JS ni tushunish shu yerdan boshlanadi.

---

## Mundarija

- [JavaScript Single-Threaded](#javascript-single-threaded)
- [Runtime Architecture — To'liq Diagramma](#runtime-architecture--toliq-diagramma)
- [Event Loop Algoritmi — Step by Step](#event-loop-algoritmi--step-by-step)
- [Macrotasks (Task Queue)](#macrotasks-task-queue)
- [Microtasks (Microtask Queue)](#microtasks-microtask-queue)
- [setTimeout(fn, 0) — Nima Uchun Darhol Bajarmaydi](#settimeoutfn-0--nima-uchun-darhol-bajarmaydi)
- [requestAnimationFrame — Qayerda Turadi](#requestanimationframe--qayerda-turadi)
- [Node.js Event Loop Farqlari](#nodejs-event-loop-farqlari)
- [Starvation — Microtask Queue To'xtovsiz To'lsa](#starvation--microtask-queue-toxtovsiz-tolsa)
- [Real-World Misol: UI Blocking va Yechimi](#real-world-misol-ui-blocking-va-yechimi)
- [Output Tartibini Aniqlang — Tricky Misollar](#output-tartibini-aniqlang--tricky-misollar)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## JavaScript Single-Threaded

### Nazariya

JavaScript **single-threaded** til — ya'ni bir vaqtda faqat **bitta** operatsiyani bajaradi. Bitta Call Stack, bitta execution context, bitta code path.

Oddiy qilib aytganda: JavaScript bir qo'lli oshpaz. U bir vaqtda faqat bitta idishni tayyorlay oladi. Lekin aqlli tarzda — bitta narsa pishib turganida, boshqa ingredientlarni tayyorlashga o'tadi.

**Nima uchun single-threaded?**

JavaScript dastlab browser uchun yaratilgan — DOM ni manipulatsiya qilish uchun. Agar ikkita thread bir vaqtda bitta DOM element ni o'zgartirmoqchi bo'lsa — **race condition** yuzaga keladi. Brendan Eich JavaScript ni single-threaded qilgan, chunki:

1. **DOM safety** — bir vaqtda bitta thread DOM ga yozadi, conflict yo'q
2. **Simplicity** — deadlock, mutex, lock kabi muammolar yo'q
3. **Predictability** — kod ketma-ketlikda bajariladi, debug qilish oson

```javascript
// JavaScript bitta thread da ishlaydi
// Bu kod KETMA-KET bajariladi, parallel emas

console.log("1");
console.log("2");
console.log("3");

// Output (har doim shu tartibda):
// 1
// 2
// 3
```

### Under the Hood

ECMAScript spec da **"job"** tushunchasi bor — lekin Event Loop o'zi **spec dan tashqarida**, HTML spec da ta'riflangan (browser uchun) va libuv kutubxonasida (Node.js uchun).

V8 engine o'zi Event Loop ni implement qilmaydi. V8 faqat JavaScript kodni compile va execute qiladi. Event Loop **runtime environment** (browser yoki Node.js) tomonidan ta'minlanadi.

```
┌─────────────────────────────────────────────────┐
│                JavaScript Engine (V8)             │
│                                                   │
│   ┌─────────────┐       ┌──────────────────┐     │
│   │  Call Stack  │       │   Memory Heap    │     │
│   │  (Execution) │       │  (Object storage)│     │
│   └─────────────┘       └──────────────────┘     │
│                                                   │
│   Engine FAQAT shu ikkitani beradi.               │
│   Event Loop, Timer, HTTP — bularni               │
│   runtime environment beradi.                     │
└─────────────────────────────────────────────────┘
```

**Muhim farq:**
- **JavaScript (til)** — single-threaded
- **Browser/Node.js (runtime)** — multi-threaded! Web APIs background thread'larda ishlaydi
- Biz yozgan JavaScript kod bitta thread da ishlaydi, lekin `setTimeout`, `fetch`, `fs.readFile` kabi operatsiyalar **boshqa thread'larda** bajariladi

```javascript
// Biz buni yozamiz:
setTimeout(() => console.log("Timer!"), 1000);

// Lekin aslida:
// 1. V8: setTimeout() ni bajaradi → Web API ga uzatadi
// 2. Browser (C++ thread): 1000ms kutadi
// 3. 1000ms o'tgach: callback ni Task Queue ga qo'yadi
// 4. Event Loop: Call Stack bo'sh bo'lganda callback ni stack ga ko'taradi
// 5. V8: callback ni bajaradi → "Timer!" chiqaradi
```

### Kod Misollari

```javascript
// ❌ Single-threaded muammo — blocking
// Agar bitta operatsiya uzoq davom etsa, HAMMA narsa to'xtaydi

function heavyCalculation() {
  let sum = 0;
  for (let i = 0; i < 10_000_000_000; i++) {
    sum += i;
  }
  return sum;
}

console.log("Boshladik");
heavyCalculation(); // 🔴 Bu yerda HAMMA narsa muzlab qoladi
console.log("Tugadi"); // Bu qator 10+ soniya kutadi

// Buttonlar bosilmaydi, animatsiyalar to'xtaydi, scroll ishlamaydi
// Chunki JavaScript single-threaded — bitta narsa bajarilayotganda boshqasi kutadi
```

```javascript
// ✅ Yechim — async operatsiyalar orqali non-blocking
// Og'ir ishlarni boshqa thread'ga (Web API) uzatamiz

console.log("Boshladik");

fetch("https://api.example.com/data")
  .then(response => response.json())
  .then(data => console.log("Data keldi:", data));

console.log("Fetch yuborildi, boshqa ishlar davom etadi");

// Output:
// "Boshladik"
// "Fetch yuborildi, boshqa ishlar davom etadi"
// "Data keldi: {...}"  (bir necha millisekunddan keyin)

// 🔍 fetch() browser ning network thread'ida ishlaydi
// JavaScript thread bloklanmaydi — boshqa kod davom etadi
```

---

## Runtime Architecture — To'liq Diagramma

### Nazariya

JavaScript o'zi single-threaded bo'lsa-da, u **yolg'iz ishlamaydi**. Browser yoki Node.js uni o'rab turgan muhitni ta'minlaydi — bu muhit **Runtime Environment** deyiladi.

Runtime Environment quyidagilardan tashkil topgan:

1. **Call Stack** — hozir bajarilayotgan funksiyalar stack'i (LIFO)
2. **Web APIs / C++ APIs** — browser yoki Node.js bergan asinxron API'lar
3. **Callback Queue (Task Queue / Macrotask Queue)** — tayyor bo'lgan callback'lar navbati
4. **Microtask Queue** — promise va boshqa microtask'lar navbati (yuqori priority)
5. **Event Loop** — bularning barchasini boshqaradigan "dirigyor"

### Under the Hood

```
┌──────────────────────────────────────────────────────────────────────┐
│                        BROWSER RUNTIME                               │
│                                                                      │
│  ┌──────────────────┐      ┌──────────────────────────────────────┐  │
│  │                  │      │          WEB APIs                    │  │
│  │    CALL STACK    │      │     (Browser Thread Pool)            │  │
│  │                  │      │                                      │  │
│  │  ┌────────────┐  │      │  ┌────────────┐  ┌──────────────┐   │  │
│  │  │ func3()    │  │ ───→ │  │ setTimeout │  │ DOM Events   │   │  │
│  │  ├────────────┤  │      │  ├────────────┤  ├──────────────┤   │  │
│  │  │ func2()    │  │      │  │ fetch/XHR  │  │ Geolocation  │   │  │
│  │  ├────────────┤  │      │  ├────────────┤  ├──────────────┤   │  │
│  │  │ func1()    │  │      │  │ setInterval│  │ Web Workers  │   │  │
│  │  ├────────────┤  │      │  └────────────┘  └──────────────┘   │  │
│  │  │  global()  │  │      │                                      │  │
│  │  └────────────┘  │      └──────────┬───────────────────────────┘  │
│  │                  │                 │                               │
│  │  ← LIFO (Last   │                 │ Callback tayyor bo'lganda     │
│  │     In First Out)│                 ↓                               │
│  └──────────────────┘                                                │
│           ↑                                                          │
│           │         ┌────────────────────────────────────────┐       │
│           │         │          MICROTASK QUEUE                │       │
│           │         │  ┌──────────┬──────────┬─────────────┐ │       │
│           │         │  │ Promise  │ Promise  │ queueMicro  │ │       │
│           │         │  │ .then()  │ .catch() │  task()     │ │       │
│           │         │  └──────────┴──────────┴─────────────┘ │       │
│           │         └──────────────────┬─────────────────────┘       │
│           │                            │ ⚡ YUQORI PRIORITY           │
│           │                            │                              │
│    ┌──────┴────────────────────────────┴────────────┐                │
│    │                  EVENT LOOP                     │                │
│    │                                                 │                │
│    │  1. Call Stack bo'shmi? → Microtask Queue       │                │
│    │  2. Microtask Queue bo'sh → Macrotask Queue     │                │
│    │  3. Macrotask dan bitta ol → Call Stack ga       │                │
│    │  4. Render (kerak bo'lsa)                       │                │
│    │  5. Qaytadan 1-dan boshla                       │                │
│    └──────┬────────────────────────────────────────-─┘                │
│           │                            ↑                              │
│           │         ┌──────────────────┴─────────────────────┐       │
│           │         │       MACROTASK QUEUE (Task Queue)      │       │
│           │         │  ┌──────────┬───────────┬────────────┐ │       │
│           │         │  │setTimeout│setInterval│ I/O events │ │       │
│           │         │  │ callback │ callback  │ callback   │ │       │
│           │         │  └──────────┴───────────┴────────────┘ │       │
│           │         └────────────────────────────────────────┘       │
│           │                                                          │
│           └──────────── Call Stack ga yuboriladi                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Har Bir Qism Haqida Batafsil

#### 1. Call Stack

Call Stack — LIFO (Last In, First Out) ma'lumot strukturasi. Har bir funksiya chaqirilganda yangi **Execution Context** yaratiladi va stack'ga push qilinadi. Funksiya tugaganda — pop qilinadi. (Batafsil: [01-js-engine.md](01-js-engine.md) va [02-execution-context.md](02-execution-context.md))

```javascript
function birinchi() {
  console.log("birinchi");
  ikkinchi();
}

function ikkinchi() {
  console.log("ikkinchi");
  uchinchi();
}

function uchinchi() {
  console.log("uchinchi");
}

birinchi();
```

```
Call Stack holati (har bir qadam):

Qadam 1:        Qadam 2:        Qadam 3:        Qadam 4:        Qadam 5:
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│            │  │            │  │ uchinchi()  │  │            │  │            │
│            │  │ ikkinchi() │  │ ikkinchi()  │  │ ikkinchi() │  │            │
│ birinchi() │  │ birinchi() │  │ birinchi()  │  │ birinchi() │  │ birinchi() │
│  global()  │  │  global()  │  │  global()   │  │  global()  │  │  global()  │
└────────────┘  └────────────┘  └────────────┘  └────────────┘  └────────────┘
 birinchi()      ikkinchi()      uchinchi()      uchinchi()      ikkinchi()
  chaqirildi     chaqirildi      chaqirildi       tugadi          tugadi
```

#### 2. Web APIs (Browser) / C++ APIs (Node.js)

Bu API'lar JavaScript engine'dan **tashqarida** ishlaydi. Browser yoki Node.js o'z thread'larida bu ishlarni bajaradi:

| Browser Web APIs | Node.js C++ APIs (libuv) |
|-----------------|-------------------------|
| `setTimeout()` / `setInterval()` | `setTimeout()` / `setInterval()` |
| `fetch()` / `XMLHttpRequest` | `http`, `fs`, `net` modullar |
| DOM Events (`click`, `scroll`) | File System operatsiyalari |
| `requestAnimationFrame()` | `crypto`, `dns` operatsiyalar |
| `Geolocation`, `WebSocket` | `child_process` |
| `IntersectionObserver` | OS-level I/O (libuv) |

```javascript
// setTimeout chaqirganda nima sodir bo'ladi:
console.log("A");

setTimeout(function timerCallback() {
  console.log("B");
}, 2000);

console.log("C");

// Qadam-baqadam:
// 1. console.log("A") → Call Stack da bajariladi → "A" chiqadi
// 2. setTimeout() → Call Stack da bajariladi → Web API ga timerCallback + 2000ms uzatiladi
//    → setTimeout() Call Stack dan chiqadi (ISHINI TUGADI)
// 3. console.log("C") → Call Stack da bajariladi → "C" chiqadi
// 4. ... 2000ms o'tadi (Web API thread'da) ...
// 5. timerCallback → Macrotask Queue ga qo'yiladi
// 6. Event Loop → Call Stack bo'sh → timerCallback ni stack ga ko'taradi
// 7. timerCallback bajariladi → "B" chiqadi

// Output: A, C, B
```

#### 3. Callback Queue (Macrotask Queue / Task Queue)

Web API lar o'z ishini tugatganda, callback'ni **Macrotask Queue** ga qo'yadi. Bu FIFO (First In, First Out) navbat — kim avval kelsa, o'sha avval bajariladi.

#### 4. Microtask Queue

Microtask Queue — **yuqoriroq prioritetli** navbat. Promise `.then()`, `.catch()`, `.finally()`, `queueMicrotask()`, `MutationObserver` callback'lari shu yerga tushadi.

**Kritik farq:** Event Loop har bir macrotask'dan keyin **barcha** microtask'larni bajaradi. Macrotask Queue dan esa **bittadan** oladi.

#### 5. Event Loop

Event Loop — bu **cheksiz loop** (while true). U Call Stack va Queue'larni kuzatib turadi va callback'larni to'g'ri tartibda bajarilishini ta'minlaydi.

```
┌─────────────────────────────────┐
│          EVENT LOOP             │
│                                 │
│    ┌───────────────────┐        │
│    │ Call Stack bo'shmi?│←──────┐│
│    └────────┬──────────┘       ││
│             │ Ha               ││
│             ↓                  ││
│    ┌───────────────────┐       ││
│    │ Microtask Queue   │       ││
│    │ da task bormi?    │       ││
│    └───┬──────────┬────┘       ││
│     Ha │          │ Yo'q       ││
│        ↓          ↓            ││
│   ┌─────────┐ ┌────────────┐  ││
│   │ BARCHA  │ │ Macrotask  │  ││
│   │ micro-  │ │ Queue dan  │  ││
│   │ task ni │ │ BITTA ol   │  ││
│   │ bajir   │ └─────┬──────┘  ││
│   └────┬────┘       │         ││
│        │            ↓         ││
│        │     ┌────────────┐   ││
│        │     │ Render     │   ││
│        │     │ (agar kerak│   ││
│        │     │  bo'lsa)   │   ││
│        │     └─────┬──────┘   ││
│        │           │          ││
│        └───────────┴──────────┘│
│                                 │
└─────────────────────────────────┘
```

### Kod Misollari

```javascript
// To'liq runtime flow — barcha qismlarni ko'rish

console.log("1: Synchronous — Call Stack");

setTimeout(() => {
  console.log("2: Macrotask — setTimeout callback");
}, 0);

Promise.resolve().then(() => {
  console.log("3: Microtask — Promise.then callback");
});

console.log("4: Synchronous — Call Stack");

// Output:
// "1: Synchronous — Call Stack"
// "4: Synchronous — Call Stack"
// "3: Microtask — Promise.then callback"
// "2: Macrotask — setTimeout callback"

// 🔍 Nima uchun bu tartibda?
// 1. Synchronous kodlar birinchi (Call Stack da darhol bajariladi)
// 2. Microtasks keyingi (Promise.then — yuqori priority)
// 3. Macrotasks eng oxirda (setTimeout — past priority)
```

---

## Event Loop Algoritmi — Step by Step

### Nazariya

Event Loop algoritmi quyidagi tartibda ishlaydi. Bu algoritmni to'liq tushunish — JavaScript asinxronligini tushunish demak.

### Event Loop Sikli (To'liq)

```
       ┌─────────────────────────────────────────┐
       │        EVENT LOOP ALGORITHM              │
       │                                          │
       │  ┌─────────────────────────────────┐     │
  ┌───→│  │ 1. Call Stack bo'shmi?          │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓ Ha                         │
  │    │  ┌─────────────────────────────────┐     │
  │    │  │ 2. Microtask Queue ni tekshir   │     │
  │    │  │    → BARCHASINI bajir           │     │
  │    │  │    (yangi microtask qo'shilsa   │     │
  │    │  │     ularni ham bajir!)          │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓                            │
  │    │  ┌─────────────────────────────────┐     │
  │    │  │ 3. Macrotask Queue dan          │     │
  │    │  │    BITTA task ol → bajir        │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓                            │
  │    │  ┌─────────────────────────────────┐     │
  │    │  │ 4. Rendering (agar kerak bo'lsa)│     │
  │    │  │    - Style hisoblash            │     │
  │    │  │    - Layout (reflow)            │     │
  │    │  │    - Paint                      │     │
  │    │  │    - requestAnimationFrame      │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓                            │
  └────│─────── 1-qadamga qayt ◄─────────────    │
       │                                          │
       └─────────────────────────────────────────┘
```

### Under the Hood

HTML Living Standard (spec) bo'yicha Event Loop quyidagicha ishlaydi:

**Qadam 1: Call Stack bo'shligini tekshir**
- Agar Call Stack da hali funksiya bajarilayotgan bo'lsa — kutamiz
- Stack bo'sh bo'lganda keyingi qadamga o'tamiz

**Qadam 2: Microtask checkpoint**
- Microtask Queue dagi **BARCHA** task'larni ketma-ket bajaramiz
- Agar microtask bajarilayotganda yangi microtask qo'shilsa — **uni ham** bajaramiz
- Microtask Queue **butunlay bo'sh** bo'lguncha davom etamiz
- Shuning uchun microtask'lar macrotask'lardan oldin ishlaydi

**Qadam 3: Macrotask Queue dan bitta task olish**
- Eng eski (birinchi qo'shilgan) task ni olamiz
- Uni Call Stack ga qo'yamiz va bajaramiz
- **Faqat BITTA** macrotask olinadi — keyin yana microtask checkpoint bo'ladi

**Qadam 4: Rendering**
- Browser ~16.6ms da bir marta render qiladi (60fps)
- Har bir Event Loop iteration'da render bo'lavermaydi
- Browser o'zi qaror qiladi — render kerakmi yoki yo'q
- `requestAnimationFrame` callback'lari shu bosqichda ishlaydi

**Qadam 5: Qaytadan boshdan**
- Loop to'xtamaydi — doimiy aylanadi
- Agar hech narsa yo'q bo'lsa — "idle" holatda kutadi

### Kod Misollari

```javascript
// Event Loop algoritmini qadam-baqadam kuzatamiz:

console.log("START");                         // Qadam 1: Sync

setTimeout(() => console.log("TIMEOUT"), 0);  // → Macrotask Queue

Promise.resolve()
  .then(() => console.log("PROMISE 1"))       // → Microtask Queue
  .then(() => console.log("PROMISE 2"));      // → (1-microtask ichida) Microtask Queue

console.log("END");                           // Qadam 2: Sync

// OUTPUT:
// START
// END
// PROMISE 1
// PROMISE 2
// TIMEOUT
```

**Qadam-baqadam tahlil:**

```
1. Call Stack: console.log("START") → bajarildi → "START"
   Call Stack: setTimeout(...) → Web API ga uzatildi (0ms timer)
   Call Stack: Promise.resolve().then(...) → Microtask Queue ga: [PROMISE_1_callback]
   Call Stack: console.log("END") → bajarildi → "END"

2. Call Stack bo'sh. Microtask Queue ni tekshir:
   Microtask Queue: [PROMISE_1_callback]
   → PROMISE_1_callback bajarildi → "PROMISE 1"
   → .then(() => "PROMISE 2") → yangi microtask: [PROMISE_2_callback]
   → PROMISE_2_callback bajarildi → "PROMISE 2"
   Microtask Queue: [] (bo'sh)

3. Macrotask Queue ni tekshir:
   Macrotask Queue: [TIMEOUT_callback]
   → TIMEOUT_callback bajarildi → "TIMEOUT"
   Macrotask Queue: [] (bo'sh)

4. Rendering (kerak bo'lsa)

5. Qaytadan boshlash → hech narsa yo'q → idle
```

```javascript
// Murakkabroq misol — microtask ichida microtask

console.log("A");

setTimeout(() => {
  console.log("B");
  Promise.resolve().then(() => console.log("C"));
}, 0);

setTimeout(() => {
  console.log("D");
}, 0);

Promise.resolve()
  .then(() => {
    console.log("E");
    queueMicrotask(() => console.log("F"));
  })
  .then(() => console.log("G"));

console.log("H");

// OUTPUT:
// A
// H
// E
// F
// G
// B
// C
// D
```

**Tahlil:**

```
Sync phase:
  → "A" chiqadi
  → setTimeout(B...) → Macrotask Queue: [B_callback]
  → setTimeout(D...) → Macrotask Queue: [B_callback, D_callback]
  → Promise.then(E...) → Microtask Queue: [E_callback]
  → "H" chiqadi

Call Stack bo'sh → Microtask Queue ni tekshir:
  → E_callback → "E" chiqadi
    → queueMicrotask(F) → Microtask Queue: [G_callback, F_callback]
  → G_callback → "G" ... KUTIB TURING!

  Aslida .then(() => "G") — bu E_callback .then() zinjiridagi keyingi qadam.
  E_callback resolve bo'lganda "G" microtask sifatida qo'shiladi.
  Shuning uchun:
  → F_callback → "F" chiqadi  (queueMicrotask avvalroq qo'shilgan)
  → G_callback → "G" chiqadi  (.then() keyinroq resolve bo'lgan)

  Microtask Queue bo'sh.

Macrotask Queue dan bitta: B_callback
  → "B" chiqadi
  → Promise.resolve().then(C) → Microtask Queue: [C_callback]

  Microtask checkpoint:
  → C_callback → "C" chiqadi

Macrotask Queue dan bitta: D_callback
  → "D" chiqadi
```

> **Eslatma:** Yuqoridagi misolda F va G tartibini tushunish muhim. `queueMicrotask(F)` sinxron tarzda microtask queue ga qo'shiladi. `.then(() => G)` esa faqat E callback resolve bo'lgandan keyin queue ga tushadi — lekin ikkalasi ham microtask, shuning uchun tartib: E → F → G.

---

## Macrotasks (Task Queue)

### Nazariya

Macrotask (yoki oddiy "task") — bu Event Loop ning asosiy ish birligi. Har bir Event Loop iteration'da **bitta** macrotask olinadi va bajariladi.

**Macrotask turlari:**

| Macrotask | Qayerdan keladi |
|-----------|----------------|
| `setTimeout(callback, delay)` | Timer Web API |
| `setInterval(callback, delay)` | Timer Web API |
| `setImmediate(callback)` | Node.js only |
| I/O operatsiyalar | File system, network |
| UI rendering events | Browser |
| `MessageChannel` `onmessage` | Browser/Worker |
| `requestIdleCallback` | Browser (idle vaqtda) |

### Under the Hood

Macrotask Queue aslida **bitta** queue emas — browser'da bir nechta **task source** lar bo'lishi mumkin. Browser o'zi qaysi source dan task olishni tanlaydi (user interaction task'lari odatda yuqoriroq priority oladi).

Lekin eng muhim qoida o'zgarmaydi: **har bir Event Loop tick'da faqat BITTA macrotask bajariladi**, keyin barcha microtask'lar bajariladi.

```
Event Loop Tick #1:          Event Loop Tick #2:
┌──────────────────┐         ┌──────────────────┐
│ 1 Macrotask      │         │ 1 Macrotask      │
│ ALL Microtasks   │         │ ALL Microtasks   │
│ Render (maybe)   │         │ Render (maybe)   │
└──────────────────┘         └──────────────────┘

Macrotask Queue: [T1, T2, T3, T4]
                  ↑               
                  Tick #1 da faqat T1 olinadi
                      ↑
                      Tick #2 da T2 olinadi
```

### Kod Misollari

```javascript
// setTimeout — eng ko'p ishlatiladigan macrotask

// 0ms timeout ham DARHOL bajarmaydi — Macrotask Queue ga qo'yadi
setTimeout(() => console.log("timeout 1"), 0);
setTimeout(() => console.log("timeout 2"), 0);
setTimeout(() => console.log("timeout 3"), 0);

console.log("sync");

// Output:
// sync
// timeout 1
// timeout 2
// timeout 3

// 🔍 Har bir setTimeout callback alohida macrotask
// Ular KETMA-KET, har bir Event Loop tick'da bittadan bajariladi
// Lekin oralarda microtask queue tekshiriladi
```

```javascript
// setInterval — takrorlanuvchi macrotask

let counter = 0;

const intervalId = setInterval(() => {
  counter++;
  console.log(`Interval: ${counter}`);

  if (counter >= 3) {
    clearInterval(intervalId);
    console.log("Interval to'xtatildi");
  }
}, 1000);

// Output (har 1 soniyada):
// Interval: 1   (1s da)
// Interval: 2   (2s da)
// Interval: 3   (3s da)
// Interval to'xtatildi
```

```javascript
// Macrotask'lar orasida microtask'lar bajariladi:

setTimeout(() => {
  console.log("Macrotask 1");
  Promise.resolve().then(() => console.log("Microtask from Macrotask 1"));
}, 0);

setTimeout(() => {
  console.log("Macrotask 2");
  Promise.resolve().then(() => console.log("Microtask from Macrotask 2"));
}, 0);

// Output:
// Macrotask 1
// Microtask from Macrotask 1
// Macrotask 2
// Microtask from Macrotask 2

// 🔍 Har bir macrotask'dan keyin microtask checkpoint bo'ladi
// Shuning uchun "Microtask from Macrotask 1" "Macrotask 2" dan OLDIN chiqadi
```

---

## Microtasks (Microtask Queue)

### Nazariya

Microtask — macrotask'dan **kichikroq** va **tezroq** bajariladigan task. Microtask Queue macrotask'larga nisbatan **yuqori priority**ga ega.

**Asosiy qoida: Event Loop har bir macrotask'dan keyin Microtask Queue ni TO'LIQ BO'SHATADI.**

**Microtask turlari:**

| Microtask | Qayerdan keladi |
|-----------|----------------|
| `Promise.then()` callback | Promise resolve/reject |
| `Promise.catch()` callback | Promise reject |
| `Promise.finally()` callback | Promise settle |
| `queueMicrotask(callback)` | Maxsus API |
| `MutationObserver` callback | DOM o'zgarishlarini kuzatish |
| `process.nextTick()` | Node.js only (microtask'dan ham yuqori!) |

### Under the Hood

ECMAScript spec da microtask'lar **"PromiseJobs"** deb ataladi. HTML spec'da esa **"microtask queue"** deb ataladi.

Microtask checkpoint algoritmini batafsil ko'ramiz:

```
Microtask Checkpoint Algoritmi:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

while (microtaskQueue.length > 0) {
    let task = microtaskQueue.shift();  // Birinchisini ol (FIFO)
    execute(task);                       // Bajir
    // Agar task ichida yangi microtask qo'shilsa
    // — loop davom etadi, uni HAM bajaradi
}

// ⚠️ XAVF: agar har bir microtask yangi microtask qo'shsa
// — bu loop HECH QACHON tugamaydi → STARVATION
```

**Muhim:** Microtask Queue bo'shatilish davomida yangi microtask qo'shilsa — ular **shu** checkpoint'da bajariladi, keyingi tick'ga qoldirilmaydi!

### Kod Misollari

```javascript
// Promise.then() — eng keng tarqalgan microtask

const promise = Promise.resolve("natija");

promise.then(value => {
  console.log("then 1:", value);
});

promise.then(value => {
  console.log("then 2:", value);
});

console.log("sync");

// Output:
// sync
// then 1: natija
// then 2: natija

// 🔍 .then() callback'lari microtask — sync koddan keyin, lekin macrotask'lardan oldin
```

```javascript
// queueMicrotask() — to'g'ridan-to'g'ri microtask qo'shish

console.log("1");

queueMicrotask(() => {
  console.log("2 — queueMicrotask");
});

Promise.resolve().then(() => {
  console.log("3 — Promise.then");
});

console.log("4");

// Output:
// 1
// 4
// 2 — queueMicrotask
// 3 — Promise.then

// 🔍 queueMicrotask va Promise.then ikkalasi ham microtask
// Tartib: FIFO — qaysi biri avval queue ga tushsa, o'sha avval bajariladi
```

```javascript
// Microtask ichida yangi microtask — recursive

queueMicrotask(() => {
  console.log("micro 1");
  queueMicrotask(() => {
    console.log("micro 2 (micro 1 ichidan yaratildi)");
    queueMicrotask(() => {
      console.log("micro 3 (micro 2 ichidan yaratildi)");
    });
  });
});

setTimeout(() => console.log("macrotask"), 0);

// Output:
// micro 1
// micro 2 (micro 1 ichidan yaratildi)
// micro 3 (micro 2 ichidan yaratildi)
// macrotask

// 🔍 BARCHA microtask'lar (ichma-ich bo'lsa ham) macrotask'dan OLDIN tugaydi
// Chunki microtask checkpoint queue TO'LIQ bo'shguncha ishlaydi
```

```javascript
// MutationObserver — DOM o'zgarishlarini kuzatish (microtask)

const targetNode = document.getElementById("app");

const observer = new MutationObserver((mutations) => {
  console.log("MutationObserver: DOM o'zgardi!");
  mutations.forEach(mutation => {
    console.log("  Turi:", mutation.type);
  });
});

observer.observe(targetNode, { childList: true, subtree: true });

// DOM o'zgarishi sodir bo'lganda — callback microtask sifatida ishlaydi
targetNode.appendChild(document.createElement("div"));
// MutationObserver callback shu frame'da, microtask sifatida ishlaydi
```

### Microtask vs Macrotask — To'liq Taqqoslash

```
┌──────────────────────┬──────────────────────────┐
│     MICROTASK        │      MACROTASK           │
├──────────────────────┼──────────────────────────┤
│ Promise.then/catch   │ setTimeout               │
│ queueMicrotask       │ setInterval              │
│ MutationObserver     │ setImmediate (Node)      │
│ process.nextTick*    │ I/O callbacks            │
│                      │ UI rendering events      │
│                      │ MessageChannel           │
├──────────────────────┼──────────────────────────┤
│ BARCHASI bajariladi  │ BITTADAN bajariladi      │
│ (har tick'da)        │ (har tick'da)            │
├──────────────────────┼──────────────────────────┤
│ Yuqori priority      │ Past priority            │
├──────────────────────┼──────────────────────────┤
│ Render dan OLDIN     │ Render bilan NAVBATMA    │
├──────────────────────┼──────────────────────────┤
│ Starvation xavfi bor │ Starvation xavfi yo'q    │
└──────────────────────┴──────────────────────────┘

* process.nextTick() Node.js da microtask queue dan HAM oldinroq ishlaydi
```

---

## setTimeout(fn, 0) — Nima Uchun Darhol Bajarmaydi

### Nazariya

`setTimeout(fn, 0)` deganda ko'pchilik "0 millisekund = darhol" deb o'ylaydi. Lekin bu **NOTO'G'RI**. `setTimeout(fn, 0)` callback'ni **Macrotask Queue** ga qo'yadi, va u faqat:

1. Hozirgi synchronous kod tugaganda
2. Barcha microtask'lar bajarilganda
3. Event Loop macrotask queue ga navbat kelganda

bajariladi.

### Under the Hood

Aslida `setTimeout(fn, 0)` **0ms ham emas**:

1. **HTML spec** bo'yicha: agar nested setTimeout 5+ marta chaqirilgan bo'lsa, minimum delay **4ms** bo'ladi (clamping)
2. **Browser** — background tab'larda setTimeout minimum **1000ms** gacha throttle qilinishi mumkin
3. **Amalda** — hatto active tab'da ham OS scheduler tufayli ~1-4ms delay bo'ladi

```javascript
// setTimeout(fn, 0) aslida nima qiladi:

console.log("A — sync");

setTimeout(() => {
  console.log("B — setTimeout(0)");
}, 0);

console.log("C — sync");

// Output:
// A — sync
// C — sync
// B — setTimeout(0)

// 🔍 B eng oxirda! Chunki:
// 1. "A" — sync, darhol
// 2. setTimeout(B, 0) — callback ni Macrotask Queue ga QO'YADI (bajaramaydi!)
// 3. "C" — sync, darhol
// 4. Call Stack bo'sh → microtask queue (bo'sh) → macrotask queue → B
```

### Kod Misollari

```javascript
// setTimeout(0) vs Promise — kim birinchi?

setTimeout(() => console.log("setTimeout"), 0);
Promise.resolve().then(() => console.log("Promise"));
console.log("Sync");

// Output:
// Sync
// Promise
// setTimeout

// 🔍 Promise (microtask) DOIM setTimeout (macrotask) dan oldin!
// 0ms delay ham buni o'zgartirmaydi
```

```javascript
// setTimeout(0) real use case — UI unblocking

// ❌ Noto'g'ri — UI bloklanadi
function renderHeavyList(items) {
  items.forEach(item => {
    const el = document.createElement("div");
    el.textContent = item.name;
    document.body.appendChild(el);
  });
  // 10,000 element qo'shilguncha UI muzlab qoladi
}

// ✅ To'g'ri — setTimeout(0) bilan batch'larga bo'lish
function renderHeavyListBatched(items) {
  const batchSize = 100;
  let index = 0;

  function renderBatch() {
    const batch = items.slice(index, index + batchSize);
    batch.forEach(item => {
      const el = document.createElement("div");
      el.textContent = item.name;
      document.body.appendChild(el);
    });
    index += batchSize;

    if (index < items.length) {
      setTimeout(renderBatch, 0); // Keyingi batch — macrotask
      // Bu orada browser render qila oladi, UI responsive bo'ladi
    }
  }

  renderBatch();
}
```

```javascript
// setTimeout minimum delay — nested call larda 4ms clamping

let start = Date.now();

setTimeout(function tick() {
  let elapsed = Date.now() - start;
  console.log(`${elapsed}ms o'tdi`);

  if (elapsed < 50) {
    setTimeout(tick, 0); // Har safar 0ms bilan chaqiramiz
  }
}, 0);

// Output (taxminiy):
// 1ms o'tdi    — birinchi chaqiruv
// 2ms o'tdi    — ikkinchi
// 3ms o'tdi    — uchinchi
// 4ms o'tdi    — to'rtinchi
// 8ms o'tdi    — beshinchi (4ms clamping boshlandi!)
// 12ms o'tdi   — oltinchi
// 16ms o'tdi   — yettinchi
// ...

// 🔍 5-chi nested setTimeout dan boshlab browser minimum 4ms delay qo'yadi
// HTML spec: "If nesting level is greater than 5, clamp timeout to at least 4ms"
```

---

## requestAnimationFrame — Qayerda Turadi

### Nazariya

`requestAnimationFrame(callback)` (qisqacha **rAF**) — na microtask, na macrotask. U **rendering bosqichida** ishlaydi — browser har gal ekranga chizishdan (paint) oldin rAF callback'larini chaqiradi.

**rAF Event Loop'da qayerda turadi:**

```
┌────────────────────────────────────────────────┐
│              EVENT LOOP TICK                    │
│                                                │
│  1. [Macrotask] ← bitta                       │
│  2. [Microtask Queue] ← barchasini bajir      │
│  3. [Rendering Pipeline]:                      │
│     a. requestAnimationFrame callbacks  ← rAF  │
│     b. Style recalculation                     │
│     c. Layout (reflow)                         │
│     d. Paint                                   │
│  4. Keyingi tick ga qayt                       │
│                                                │
│  * Rendering har tick'da bo'lmaydi —           │
│    browser ~16.6ms (60fps) oraliqda render     │
│    qiladi. Oraliq tick'larda render skip.      │
└────────────────────────────────────────────────┘
```

### Under the Hood

**rAF xususiyatlari:**
- Har frame'da **bir marta** chaqiriladi (~60fps = ~16.6ms)
- Agar tab background'da bo'lsa — **to'xtaydi** (battery/CPU tejash)
- Animation uchun **eng yaxshi** usul (vsync bilan sinxronlashgan)
- `setTimeout(fn, 16)` dan aniqroq — chunki browser ning render sikliga ulangan

```javascript
// rAF vs setTimeout — animation uchun farq

// ❌ Noto'g'ri — setTimeout bilan animation
function animateWithTimeout(element) {
  let position = 0;

  function move() {
    position += 2;
    element.style.transform = `translateX(${position}px)`;
    if (position < 300) {
      setTimeout(move, 16); // ~60fps? Yo'q — aniq emas!
    }
  }

  move();
}
// Muammo: setTimeout 16ms aniq emas (4ms clamping, throttling)
// Natija: silliqligi past, jank/stutter bo'lishi mumkin

// ✅ To'g'ri — requestAnimationFrame bilan animation
function animateWithRAF(element) {
  let position = 0;

  function move() {
    position += 2;
    element.style.transform = `translateX(${position}px)`;
    if (position < 300) {
      requestAnimationFrame(move); // Browser render sikliga ulangan
    }
  }

  requestAnimationFrame(move);
}
// Natija: silliq, 60fps ga sinxron, battery-friendly
```

### Kod Misollari

```javascript
// rAF, setTimeout, Promise — tartibni aniqlang

console.log("1 — sync");

requestAnimationFrame(() => {
  console.log("2 — requestAnimationFrame");
});

setTimeout(() => {
  console.log("3 — setTimeout(0)");
}, 0);

Promise.resolve().then(() => {
  console.log("4 — Promise.then");
});

console.log("5 — sync");

// Ko'p hollarda output:
// 1 — sync
// 5 — sync
// 4 — Promise.then
// 3 — setTimeout(0)
// 2 — requestAnimationFrame

// 🔍 Tartib sababi:
// 1, 5 — synchronous (darhol)
// 4 — microtask (eng yuqori async priority)
// 3 — macrotask (microtask'lardan keyin)
// 2 — rAF (render bosqichida — macrotask'dan HAM keyin)
//
// ⚠️ DIQQAT: 3 va 2 tartibi browser va frame timing'ga qarab
// o'zgarishi mumkin. Agar render shu tick'da bo'lmasa,
// rAF keyingi render'gacha kutishi mumkin.
```

```javascript
// rAF bilan silliq DOM update

function smoothScrollToTop() {
  const scrollY = window.scrollY;
  if (scrollY > 0) {
    window.scrollTo(0, scrollY - scrollY / 8);
    requestAnimationFrame(smoothScrollToTop);
  }
}

// Har bir frame'da scroll position ni kamaytiramiz
// Browser render sikliga sinxron — silliq animation
document.getElementById("topBtn").addEventListener("click", () => {
  requestAnimationFrame(smoothScrollToTop);
});
```

---

## Node.js Event Loop Farqlari

### Nazariya

Node.js Event Loop browser'nikidan farq qiladi. Node.js **libuv** kutubxonasini ishlatadi — bu C da yozilgan cross-platform asinxron I/O kutubxonasi. Node.js Event Loop **6 ta faza (phase)** dan iborat.

### Under the Hood

```
   ┌───────────────────────────────────────────────┐
   │              NODE.JS EVENT LOOP                │
   │                                                │
   │    ┌─────────────────────────────┐             │
┌──│──→ │    1. TIMERS                │             │
│  │    │    setTimeout, setInterval  │             │
│  │    │    callback'lari            │             │
│  │    └──────────────┬──────────────┘             │
│  │                   │                            │
│  │    ┌──────────────▼──────────────┐             │
│  │    │    2. PENDING CALLBACKS     │             │
│  │    │    System-level callback    │             │
│  │    │    (TCP errors, etc.)       │             │
│  │    └──────────────┬──────────────┘             │
│  │                   │                            │
│  │    ┌──────────────▼──────────────┐             │
│  │    │    3. IDLE / PREPARE        │             │
│  │    │    (ichki ishlatiladi)      │             │
│  │    └──────────────┬──────────────┘             │
│  │                   │                            │
│  │    ┌──────────────▼──────────────┐             │
│  │    │    4. POLL                  │  ← Asosiy!  │
│  │    │    I/O events kutish        │             │
│  │    │    (fs, network, etc.)      │             │
│  │    │    callback'larni bajarish  │             │
│  │    └──────────────┬──────────────┘             │
│  │                   │                            │
│  │    ┌──────────────▼──────────────┐             │
│  │    │    5. CHECK                 │             │
│  │    │    setImmediate()           │             │
│  │    │    callback'lari            │             │
│  │    └──────────────┬──────────────┘             │
│  │                   │                            │
│  │    ┌──────────────▼──────────────┐             │
│  │    │    6. CLOSE CALLBACKS       │             │
│  │    │    socket.on('close', ...)  │             │
│  │    └──────────────┬──────────────┘             │
│  │                   │                            │
│  └───────────────────┘  ← Qaytadan Timers ga     │
│                                                   │
│  ══════════════════════════════════════════════   │
│  HAR BIR FAZA ORASIDA:                           │
│  ┌─────────────────────────────────────────┐     │
│  │  process.nextTick() queue  ← ENG BIRINCHI│     │
│  │  Microtask queue (Promises) ← IKKINCHI   │     │
│  └─────────────────────────────────────────┘     │
│  ══════════════════════════════════════════════   │
└───────────────────────────────────────────────────┘
```

### Har Bir Faza Haqida

**1. Timers** — `setTimeout` va `setInterval` callback'lari. Timer expired bo'lgan callback'lar shu fazada bajariladi.

**2. Pending Callbacks** — ba'zi system-level I/O callback'lari (masalan, TCP `ECONNREFUSED`).

**3. Idle/Prepare** — faqat Node.js ichki ishlatadi. Biz uchun ahamiyati yo'q.

**4. Poll** — eng muhim faza:
- Yangi I/O event'larini kutadi
- I/O callback'larini bajaradi (fs.readFile, http response, etc.)
- Agar poll queue bo'sh bo'lsa:
  - `setImmediate` bor → check faza ga o'tadi
  - Timer expired bor → timers faza ga o'tadi
  - Hech narsa yo'q → yangi event kutadi

**5. Check** — `setImmediate()` callback'lari. Bu **faqat Node.js** da bor.

**6. Close Callbacks** — `socket.on('close', ...)` kabi close event'lari.

### `process.nextTick()` vs `setImmediate()`

Bu ikkalasi Node.js ga xos va ko'pchilikni chalkashtiradigan API'lar:

| Xususiyat | `process.nextTick()` | `setImmediate()` |
|-----------|---------------------|------------------|
| Qachon ishlaydi | **Har bir faza ORASIDA**, barchasidan oldin | **Check faza** da |
| Priority | Eng yuqori (microtask'dan ham oldin) | Macrotask (check phase) |
| Queue | nextTick Queue (alohida) | Check Queue |
| Starvation xavfi | **HA** — recursive nextTick loop'ni to'xtatmaydi | Yo'q |
| Nomlash | Aslida "immediate" ishlaydigan shu | Aslida "next tick" da ishlaydigan shu 😅 |

> **Qiziq fakt:** `process.nextTick` va `setImmediate` nomlari teskari qo'yilgan. `nextTick` darhol ishlaydi, `setImmediate` esa keyingi fazada. Bu Node.js ning tarixiy xatosi — o'zgartirishning iloji yo'q (backward compatibility).

### Kod Misollari

```javascript
// Node.js — process.nextTick vs setImmediate vs setTimeout

console.log("1 — sync");

setTimeout(() => console.log("2 — setTimeout"), 0);

setImmediate(() => console.log("3 — setImmediate"));

process.nextTick(() => console.log("4 — nextTick"));

Promise.resolve().then(() => console.log("5 — Promise"));

console.log("6 — sync");

// Output (Node.js):
// 1 — sync
// 6 — sync
// 4 — nextTick       ← har bir fazadan OLDIN
// 5 — Promise         ← microtask (nextTick dan keyin)
// 2 — setTimeout      ← timers faza
// 3 — setImmediate    ← check faza

// 🔍 Tartib: sync → nextTick → microtask → macrotask
// Node.js da: nextTick > Promise > setTimeout >= setImmediate
```

```javascript
// setTimeout(0) vs setImmediate — I/O ichida

const fs = require("fs");

// I/O callback ICHIDA — setImmediate DOIM birinchi
fs.readFile(__filename, () => {
  setTimeout(() => console.log("setTimeout"), 0);
  setImmediate(() => console.log("setImmediate"));
});

// Output (HAR DOIM):
// setImmediate
// setTimeout

// 🔍 I/O callback poll faza da bajariladi.
// Poll dan keyin → Check faza (setImmediate) → keyin loop boshiga → Timers (setTimeout)
// Shuning uchun I/O ichida setImmediate DOIM birinchi.
```

```javascript
// Top-level da — tartib GARANTI EMAS

setTimeout(() => console.log("setTimeout"), 0);
setImmediate(() => console.log("setImmediate"));

// Output (OLDINDAN AYTIB BO'LMAYDI — system timer resolution ga bog'liq):
// Ba'zan: setTimeout → setImmediate
// Ba'zan: setImmediate → setTimeout

// 🔍 Nima uchun? Main module top-level da, poll faza yo'q.
// setTimeout(0) aslida ≈1ms. Agar 1ms o'tgan bo'lsa → setTimeout birinchi.
// Agar hali o'tmagan bo'lsa → setImmediate birinchi.
// Timer resolution OS ga bog'liq.
```

### Browser vs Node.js Event Loop Taqqoslash

```
┌────────────────────────┬────────────────────────────┐
│      BROWSER           │       NODE.JS              │
├────────────────────────┼────────────────────────────┤
│ HTML Spec              │ libuv (C kutubxona)        │
├────────────────────────┼────────────────────────────┤
│ Bitta macrotask →      │ 6 ta faza (phase) bor      │
│ barcha microtask →     │ Har faza orasida:          │
│ render (agar kerak)    │ nextTick → microtask       │
├────────────────────────┼────────────────────────────┤
│ requestAnimationFrame  │ rAF yo'q (DOM yo'q)        │
├────────────────────────┼────────────────────────────┤
│ queueMicrotask ✅       │ queueMicrotask ✅           │
├────────────────────────┼────────────────────────────┤
│ MutationObserver ✅     │ MutationObserver ❌         │
├────────────────────────┼────────────────────────────┤
│ setImmediate ❌         │ setImmediate ✅             │
├────────────────────────┼────────────────────────────┤
│ process.nextTick ❌     │ process.nextTick ✅         │
├────────────────────────┼────────────────────────────┤
│ UI Rendering bor       │ UI Rendering yo'q          │
├────────────────────────┼────────────────────────────┤
│ Background tab throttle│ Throttle yo'q              │
└────────────────────────┴────────────────────────────┘
```

---

## Starvation — Microtask Queue To'xtovsiz To'lsa

### Nazariya

**Starvation** — bu microtask queue hech qachon bo'sh bo'lmaydigan holat. Har bir microtask yangi microtask qo'shsa, Event Loop **hech qachon** macrotask'larga yoki rendering'ga o'tmaydi.

Buni eslang: Event Loop **barcha** microtask'larni bajaradi macrotask'ga o'tishdan oldin. Agar microtask'lar cheksiz qo'shilsa — macrotask'lar va rendering **ochliqdan o'ladi** (starve).

### Under the Hood

```
Normal holat:
━━━━━━━━━━━━━
[Sync] → [Micro 1, Micro 2] → [Macro 1] → [Micro 3] → [Render] → [Macro 2] → ...
                                                          ↑
                                                     UI yangilanadi ✅

Starvation holati:
━━━━━━━━━━━━━━━━━━
[Sync] → [Micro 1 → Micro 2 → Micro 3 → Micro 4 → Micro 5 → ...∞...]
                                                                    ↑
                                           HECH QACHON Macro va Render ga yetmaydi ❌
                                           UI muzlab qoladi!
                                           setTimeout callback'lari bajarmaydi!
```

### Kod Misollari

```javascript
// ❌ XAVFLI — Microtask starvation (browser muzlab qoladi!)
// BU KODNI ISHLATMANG — faqat tushunish uchun

function starvation() {
  Promise.resolve().then(() => {
    console.log("microtask"); // Cheksiz chiqadi
    starvation();             // Yangi microtask qo'shadi
  });
}

starvation();
// setTimeout(() => console.log("HECH QACHON ishlamaydi"), 0);

// 🔴 Bu kod browser'ni muzlatadi!
// Microtask queue hech qachon bo'sh bo'lmaydi
// → Macrotask'lar (setTimeout) bajarilmaydi
// → Rendering to'xtaydi → UI muzlab qoladi
// → Tab "Not Responding" bo'ladi
```

```javascript
// ❌ XAVFLI — queueMicrotask bilan ham xuddi shunday

function infiniteMicrotasks() {
  queueMicrotask(() => {
    console.log("yana microtask...");
    infiniteMicrotasks(); // cheksiz loop
  });
}

infiniteMicrotasks();
// 🔴 Huddi Promise starvation kabi — browser muzlaydi
```

```javascript
// ✅ XAVFSIZ — setTimeout bilan "breathing room" berish

function safeRecursion() {
  // Har bir iteration — macrotask
  // Event Loop boshqa task'larni HAM bajara oladi
  setTimeout(() => {
    console.log("iteration");
    safeRecursion();
  }, 0);
}

safeRecursion();
// Bu xavfsiz: har bir iteration macrotask
// Orada microtask'lar, render, boshqa macrotask'lar bajarilib turadi
```

```javascript
// ✅ To'g'ri — process.nextTick starvation (Node.js)

// ❌ Node.js da process.nextTick bilan ham starvation bo'ladi:
function badNextTick() {
  process.nextTick(() => {
    // Bu cheksiz davom etsa — I/O callback'lar HECH QACHON ishlamaydi
    badNextTick();
  });
}

// ✅ Node.js hujjatlari tavsiyasi:
// Recursive holatlarda setImmediate() ishlating, nextTick() emas
function goodRecursion() {
  setImmediate(() => {
    // setImmediate check fazada ishlaydi
    // Orada boshqa faza'lar (poll, timers) ishlay oladi
    goodRecursion();
  });
}
```

### Starvation Prevention Qoidalari

| Qoida | Tushuntirish |
|-------|-------------|
| Recursive microtask yaratmang | `Promise.then` yoki `queueMicrotask` ichida recursive chaqirmang |
| Uzoq loop'larni macrotask'ga bo'ling | `setTimeout(fn, 0)` yoki `setImmediate` bilan batch'lang |
| Node.js da `setImmediate` ishlating | `process.nextTick` o'rniga `setImmediate` (recursive holatda) |
| `requestAnimationFrame` ishlating | Animation uchun — render sikliga ulangan, starvation qilmaydi |

---

## Real-World Misol: UI Blocking va Yechimi

### Nazariya

UI blocking — bu JavaScript ning single-threaded tabiatidan kelib chiqadigan eng katta muammo. Agar Call Stack da og'ir hisoblash bajarilayotgan bo'lsa — browser **hech narsani** qila olmaydi: button click'lar, scroll, animation, render — HAMMA narsa to'xtaydi.

### Kod Misollari

```javascript
// ❌ Real-world muammo: Search filter 10,000 ta itemni sync filterlaydi

const searchInput = document.getElementById("search");
const resultsList = document.getElementById("results");
const items = Array.from({ length: 10000 }, (_, i) => `Mahsulot ${i + 1}`);

searchInput.addEventListener("input", (e) => {
  const query = e.target.value.toLowerCase();

  // 🔴 10,000 ta itemni filterlash + DOM update — hammasi sync
  resultsList.innerHTML = "";
  const filtered = items.filter(item => item.toLowerCase().includes(query));

  filtered.forEach(item => {
    const li = document.createElement("li");
    li.textContent = item;
    resultsList.appendChild(li);
  });

  // Natija: har bir harf yozganda UI 100-500ms muzlaydi
  // Foydalanuvchi yozayotganda kursor "qotib qoladi"
});
```

```javascript
// ✅ Yechim 1: Debounce + requestAnimationFrame

function debounce(fn, delay) {
  let timeoutId;
  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

searchInput.addEventListener("input", debounce((e) => {
  const query = e.target.value.toLowerCase();
  const filtered = items.filter(item => item.toLowerCase().includes(query));

  // requestAnimationFrame — render sikliga sinxron
  requestAnimationFrame(() => {
    resultsList.innerHTML = "";
    const fragment = document.createDocumentFragment();
    filtered.forEach(item => {
      const li = document.createElement("li");
      li.textContent = item;
      fragment.appendChild(li);
    });
    resultsList.appendChild(fragment); // Bir marta DOM ga qo'shish
  });
}, 300)); // 300ms kutish — foydalanuvchi yozishni to'xtatganda ishlaydi
```

```javascript
// ✅ Yechim 2: Og'ir hisoblashni batch'larga bo'lish

function processInBatches(items, batchSize, processFn) {
  let index = 0;

  function nextBatch() {
    const batch = items.slice(index, index + batchSize);
    batch.forEach(processFn);
    index += batchSize;

    if (index < items.length) {
      // Har bir batch orasida Event Loop ga "nafas olish" imkoniyati
      setTimeout(nextBatch, 0);
    }
  }

  nextBatch();
}

// Ishlatish:
processInBatches(hugeDataArray, 500, (item) => {
  // Har 500 ta itemdan keyin UI yangilanadi
  renderItem(item);
});
```

```javascript
// ✅ Yechim 3: Web Worker — og'ir ishni boshqa thread'ga uzatish

// main.js
const worker = new Worker("heavy-worker.js");

searchInput.addEventListener("input", debounce((e) => {
  worker.postMessage({
    query: e.target.value,
    items: items
  });
}, 200));

worker.onmessage = (e) => {
  const filtered = e.data;
  requestAnimationFrame(() => {
    renderResults(filtered); // UI thread'da faqat render
  });
};

// heavy-worker.js (alohida thread!)
self.onmessage = (e) => {
  const { query, items } = e.data;
  // Og'ir filter — bu yerda UI bloklanmaydi!
  const filtered = items.filter(item =>
    item.toLowerCase().includes(query.toLowerCase())
  );
  self.postMessage(filtered);
};

// 🔍 Web Worker alohida thread'da ishlaydi
// Main thread (UI) hech qachon bloklanmaydi
// Lekin: Worker da DOM ga access yo'q
```

### UI Blocking Diagrammasi

```
❌ BLOCKING (og'ir ish sync):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│█████████████████████████│ render │ click │ render │
│  Og'ir hisoblash (500ms)│       │       │        │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
0ms                     500ms    516ms   520ms   536ms
     ↑ BU ORADA UI MUZLAGAN


✅ NON-BLOCKING (batch'larga bo'lingan):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│batch│render│click│batch│render│batch│render│batch│
│ 50ms│      │     │50ms │      │50ms │      │50ms │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
0    50     66    70   120    136   186   202   252ms
      ↑ UI responsive — har batch orasida render va event'lar ishlaydi
```

---

## Output Tartibini Aniqlang — Tricky Misollar

Bu bo'lim eng muhim — interview'da eng ko'p so'raladigan savollar shu turdagi. Har bir misolni sinchiklab o'rganing.

### Misol 1: Asosiy Tartib

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");

// Output nima?
// ...
// ...
// ...
// Javob: 1, 4, 3, 2
// Sync (1, 4) → Microtask (3) → Macrotask (2)
```

### Misol 2: Promise Chain

```javascript
console.log("A");

setTimeout(() => console.log("B"), 0);

Promise.resolve()
  .then(() => console.log("C"))
  .then(() => console.log("D"));

Promise.resolve()
  .then(() => console.log("E"));

console.log("F");

// Output nima?
// ...
// ...
// ...
// Javob: A, F, C, E, D, B

// 🔍 Tahlil:
// Sync: A, F
// Microtask Queue (sync tugagandan keyin): [C_callback, E_callback]
// → C bajariladi → .then(D) queue ga qo'shiladi: [E_callback, D_callback]
// → E bajariladi → [D_callback]
// → D bajariladi → []
// Macrotask: B
```

### Misol 3: setTimeout Ketma-ketligi

```javascript
setTimeout(() => console.log("A"), 0);
setTimeout(() => console.log("B"), 0);

Promise.resolve()
  .then(() => {
    console.log("C");
    setTimeout(() => console.log("D"), 0);
  });

Promise.resolve()
  .then(() => console.log("E"));

// Output nima?
// ...
// ...
// ...
// Javob: C, E, A, B, D

// 🔍 Tahlil:
// Sync tugagandan keyin:
//   Macrotask Queue: [A, B]
//   Microtask Queue: [C_callback, E_callback]
//
// Microtask checkpoint:
//   C → "C" + setTimeout(D) → Macrotask Queue: [A, B, D]
//   E → "E"
//
// Macrotask: A → "A"
// Microtask checkpoint (bo'sh)
// Macrotask: B → "B"
// Microtask checkpoint (bo'sh)
// Macrotask: D → "D"
```

### Misol 4: Nested Promises

```javascript
new Promise((resolve) => {
  console.log("1"); // ← Sync! Promise constructor ichida
  resolve();
}).then(() => {
  console.log("2");
  return new Promise((resolve) => {
    console.log("3"); // ← Sync! (then callback ichida)
    resolve();
  }).then(() => {
    console.log("4");
  });
}).then(() => {
  console.log("5");
});

console.log("6");

// Output nima?
// ...
// ...
// ...
// Javob: 1, 6, 2, 3, 4, 5

// 🔍 Tahlil:
// Sync: 1 (Promise constructor sync ishlaydi!), 6
// Microtask: then_2 callback
//   → "2" chiqadi
//   → ichida yangi Promise("3" — sync) va .then(4)
//   → "3" chiqadi (sync)
//   → then_4 microtask queue ga
//   → then_2 return qilgan Promise resolve bo'lganda then_5 queue ga tushadi
// Microtask: then_4 → "4"
// Microtask: then_5 → "5"
```

### Misol 5: async/await Desugaring

```javascript
async function asyncFunc() {
  console.log("1");
  await Promise.resolve();
  console.log("2"); // ← Bu aslida .then() ichida!
  await Promise.resolve();
  console.log("3"); // ← Bu ham .then() ichida!
}

console.log("A");
asyncFunc();
console.log("B");

// Output nima?
// ...
// ...
// ...
// Javob: A, 1, B, 2, 3

// 🔍 async/await = Promise + .then() syntactic sugar
// Yuqoridagi kod quyidagiga teng:
//
// console.log("A");
// console.log("1");          // async func ning sync qismi
// Promise.resolve()
//   .then(() => {
//     console.log("2");
//     return Promise.resolve();
//   })
//   .then(() => {
//     console.log("3");
//   });
// console.log("B");
```

### Misol 6: setTimeout + Promise Aralashmasi

```javascript
setTimeout(() => {
  console.log("1");
  Promise.resolve().then(() => console.log("2"));
}, 0);

setTimeout(() => {
  console.log("3");
  Promise.resolve().then(() => console.log("4"));
}, 0);

Promise.resolve()
  .then(() => console.log("5"))
  .then(() => console.log("6"));

// Output nima?
// ...
// ...
// ...
// Javob: 5, 6, 1, 2, 3, 4

// 🔍 Tahlil:
// Microtask Queue: [then_5]
// Macrotask Queue: [setTimeout_1, setTimeout_3]
//
// Microtask checkpoint:
//   5 → "5" → then_6 queue ga
//   6 → "6"
//
// Macrotask: setTimeout_1 → "1"
//   → Promise.then(2) → Microtask Queue: [then_2]
// Microtask checkpoint:
//   2 → "2"
//
// Macrotask: setTimeout_3 → "3"
//   → Promise.then(4) → Microtask Queue: [then_4]
// Microtask checkpoint:
//   4 → "4"
```

### Misol 7: queueMicrotask + Promise + setTimeout

```javascript
console.log("START");

queueMicrotask(() => {
  console.log("queueMicrotask 1");
  queueMicrotask(() => console.log("queueMicrotask 2"));
});

setTimeout(() => console.log("setTimeout 1"), 0);

Promise.resolve()
  .then(() => console.log("Promise 1"))
  .then(() => console.log("Promise 2"));

setTimeout(() => console.log("setTimeout 2"), 0);

console.log("END");

// Output nima?
// ...
// ...
// ...
// Javob:
// START
// END
// queueMicrotask 1
// Promise 1
// queueMicrotask 2
// Promise 2
// setTimeout 1
// setTimeout 2

// 🔍 Tahlil:
// Sync: START, END
// Microtask Queue: [qM1_callback, P1_callback]
//   qM1 → "queueMicrotask 1" → qM2 queue ga: [..., P1_callback, qM2_callback]
//   P1  → "Promise 1" → P2 queue ga: [qM2_callback, P2_callback]
//   qM2 → "queueMicrotask 2"
//   P2  → "Promise 2"
// Macrotask: setTimeout 1, setTimeout 2
```

### Misol 8: Promise Constructor — Sync Trap

```javascript
const promise = new Promise((resolve, reject) => {
  console.log("1 — Promise constructor sync!");
  resolve("done");
  console.log("2 — resolve dan keyin ham ishlaydi!");
  // resolve() funksiyani to'xtatmaydi — faqat Promise holatini o'zgartiradi
});

promise.then(val => console.log("3 — then:", val));

console.log("4 — sync");

// Output nima?
// ...
// ...
// ...
// Javob: 1, 2, 4, 3

// 🔍 Muhim: Promise constructor SYNC ishlaydi!
// resolve() Promise ni fulfilled qiladi, lekin .then() callback
// MICROTASK sifatida keyinroq ishlaydi
```

### Misol 9: Super Tricky — Ko'p Qatlamli

```javascript
console.log("1");

setTimeout(() => {
  console.log("2");

  new Promise(resolve => {
    console.log("3");
    resolve();
  }).then(() => {
    console.log("4");
  });

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

// Output nima?
// ...
// ...
// ...
// Javob: 1, 6, 10, 7, 9, 2, 3, 4, 8, 5

// 🔍 Qadam-baqadam:
// SYNC: "1", "6" (Promise constructor), "10"
//
// Microtask: then_7 → "7"
//   → setTimeout(8) → Macrotask Queue: [setTimeout_2, setTimeout_8]
//   → return Promise.resolve() → then_9 queue ga
// Microtask: then_9 → "9"
//
// Macrotask: setTimeout_2 → "2"
//   → Promise constructor: "3" (sync)
//   → .then(4) → Microtask Queue: [then_4]
//   → setTimeout(5) → Macrotask Queue: [setTimeout_8, setTimeout_5]
// Microtask: then_4 → "4"
//
// Macrotask: setTimeout_8 → "8"
// Macrotask: setTimeout_5 → "5"
```

---

## Common Mistakes

### ❌ Xato 1: setTimeout(fn, 0) "darhol" ishlaydi deb o'ylash

```javascript
// Ko'pchilik buni "instant" deb o'ylaydi
console.log("oldin");
setTimeout(() => console.log("setTimeout"), 0);
console.log("keyin");

// "setTimeout" darhol chiqadi deb kutiladi
// Lekin aslida:
// oldin
// keyin
// setTimeout  ← ENG OXIRDA!
```

### ✅ To'g'ri usul:

```javascript
// setTimeout(fn, 0) — macrotask queue ga qo'yadi
// U faqat sync kod va microtask'lar tugagandan keyin ishlaydi

// Agar darhol kerak bo'lsa — sync yozing:
console.log("oldin");
console.log("darhol"); // Darhol ishlaydi
console.log("keyin");

// Agar microtask kerak bo'lsa — queueMicrotask ishlating:
queueMicrotask(() => console.log("microtask"));
// Bu setTimeout(fn,0) dan OLDIN ishlaydi
```

**Nima uchun:** `setTimeout(fn, 0)` callback'ni Web API ga uzatadi, keyin Macrotask Queue ga qo'yadi. U faqat Call Stack bo'sh BO'LGANDAN KEYIN va barcha microtask'lar bajarilgandan keyin ishlaydi.

---

### ❌ Xato 2: Promise constructor sync ekanini bilmaslik

```javascript
console.log("A");

new Promise((resolve) => {
  console.log("B"); // Bu ASYNC deb o'ylash ← XATO!
  resolve();
});

console.log("C");

// Ko'pchilik kutadi: A, C, B
// Aslida: A, B, C
```

### ✅ To'g'ri usul:

```javascript
console.log("A");

new Promise((resolve) => {
  console.log("B"); // ✅ Sync — Promise constructor darhol ishlaydi
  resolve();
}).then(() => {
  console.log("D"); // ✅ Async — .then() callback microtask
});

console.log("C");

// Output: A, B, C, D
// Promise constructor: SYNC
// .then() callback: MICROTASK (async)
```

**Nima uchun:** `new Promise(executor)` dagi `executor` funksiya **synchronous** tarzda, darhol chaqiriladi. Faqat `.then()`, `.catch()`, `.finally()` callback'lari microtask sifatida async ishlaydi.

---

### ❌ Xato 3: Microtask va Macrotask tartibini bilmaslik

```javascript
// "setTimeout birinchi" deb o'ylash ← XATO
setTimeout(() => console.log("timeout"), 0);
Promise.resolve().then(() => console.log("promise"));

// Kutiladi: timeout, promise
// Aslida: promise, timeout
```

### ✅ To'g'ri usul:

```javascript
// Tartibni eslab qoling: Sync → Microtask → Macrotask

setTimeout(() => console.log("3 — macrotask"), 0);
Promise.resolve().then(() => console.log("2 — microtask"));
console.log("1 — sync");

// Output: 1, 2, 3
// DOIM shu tartibda!
```

**Nima uchun:** Microtask Queue **har doim** Macrotask Queue dan oldin bo'shatiladi. Bu Event Loop algoritmining asosiy qoidasi.

---

### ❌ Xato 4: Og'ir hisoblashni main thread'da qilish

```javascript
// ❌ UI bloklaydigan kod
button.addEventListener("click", () => {
  // 50,000 ta elementni sortlash — UI 3-5 soniya muzlaydi
  const sorted = hugeArray.sort((a, b) => {
    // Murakkab comparison logic
    return complexCompare(a, b);
  });
  renderTable(sorted);
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ Web Worker bilan
const sortWorker = new Worker("sort-worker.js");

button.addEventListener("click", () => {
  // Loading ko'rsatish
  showSpinner();

  // Og'ir ishni Worker ga yuborish
  sortWorker.postMessage(hugeArray);

  sortWorker.onmessage = (e) => {
    hideSpinner();
    renderTable(e.data); // Tayyor natijani render qilish
  };
});

// sort-worker.js
self.onmessage = (e) => {
  const sorted = e.data.sort((a, b) => complexCompare(a, b));
  self.postMessage(sorted);
};
```

**Nima uchun:** JavaScript single-threaded. Agar main thread'da og'ir hisoblash bajarilsa — UI, event'lar, animation'lar **hammasi** to'xtaydi. Web Worker alohida thread'da ishlaydi va main thread'ni bloklamaydi.

---

### ❌ Xato 5: async/await ning Event Loop'dagi o'rnini tushunmaslik

```javascript
// Ko'pchilik await "to'xtatadi" deb o'ylaydi
async function example() {
  console.log("A");
  await fetch("/api/data"); // "Hamma narsa shu yerda to'xtaydi" ← XATO tushuncha
  console.log("B");
}

example();
console.log("C");

// Ko'pchilik kutadi: A, B, C
// Aslida: A, C, B
```

### ✅ To'g'ri usul:

```javascript
// await FAQAT async funksiya ICHIDA "kutadi"
// Tashqi kod DAVOM ETADI

async function example() {
  console.log("A");     // sync — darhol
  await Promise.resolve(); // Shu yerdan keyin — microtask
  console.log("B");     // ← microtask callback ichida
}

example();              // async funksiyani chaqiramiz
console.log("C");       // ← Bu DARHOL ishlaydi!

// Output: A, C, B

// 🔍 await — funksiya ichini "to'xtatadi", lekin TASHQI kodni to'xtatmaydi
// await = .then() ning syntactic sugar'i
// await dan keyingi kod = .then() callback ichidagi kod
```

**Nima uchun:** `await` faqat o'sha async funksiyaning bajarilishini to'xtatadi. Call Stack'dan async funksiya chiqadi va boshqa sync kod davom etadi. `await` dan keyingi kod aslida `.then()` callback'i — ya'ni microtask.

---

## Amaliy Mashqlar

### Mashq 1: Output Tartibini Aniqlang (Oson)

**Savol:** Quyidagi kod nima chiqaradi?

```javascript
console.log("A");

setTimeout(() => console.log("B"), 0);

Promise.resolve()
  .then(() => console.log("C"))
  .then(() => console.log("D"));

queueMicrotask(() => console.log("E"));

console.log("F");
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// A
// F
// C
// E
// D
// B
```

**Tushuntirish:**
1. **Sync:** `A`, `F` — darhol bajariladi
2. **Microtask Queue:** `[C_callback, E_callback]` — .then(C) va queueMicrotask(E) ketma-ket queue ga tushadi
3. `C` bajariladi → `.then(D)` microtask queue ga qo'shiladi: `[E_callback, D_callback]`
4. `E` bajariladi
5. `D` bajariladi
6. **Macrotask:** `B` — setTimeout callback eng oxirda

</details>

---

### Mashq 2: Murakkab Tartib (O'rta)

**Savol:** Quyidagi kod nima chiqaradi?

```javascript
async function first() {
  console.log("1");
  await second();
  console.log("2");
}

async function second() {
  console.log("3");
  await Promise.resolve();
  console.log("4");
}

console.log("5");
first();
console.log("6");
setTimeout(() => console.log("7"), 0);
Promise.resolve().then(() => console.log("8"));
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// 5
// 1
// 3
// 6
// 4
// 8
// 2
// 7
```

**Tushuntirish:**
1. `"5"` — sync
2. `first()` chaqirildi → `"1"` — sync (first ichida)
3. `await second()` → `second()` chaqirildi → `"3"` — sync (second ichida)
4. `await Promise.resolve()` — second to'xtaydi, microtask queue ga: `[second_resume]`
5. `second()` qaytmadi hali, shuning uchun `first` ham kutib turadi
6. `"6"` — sync (main script davom etadi)
7. `setTimeout(7)` → Macrotask Queue: `[timeout_7]`
8. `Promise.then(8)` → Microtask Queue: `[second_resume, then_8]`
9. **Microtask checkpoint:**
   - `second_resume` → `"4"` chiqadi. `second()` tugadi → `first` resume bo'ladi: `[then_8, first_resume]`
   - `then_8` → `"8"` chiqadi
   - `first_resume` → `"2"` chiqadi
10. **Macrotask:** `"7"`

</details>

---

### Mashq 3: Event Loop Visualizer (O'rta)

**Savol:** Quyidagi kodning har bir qadamida Call Stack, Microtask Queue, va Macrotask Queue holatini yozing.

```javascript
console.log("sync 1");

setTimeout(() => {
  console.log("timeout");
  Promise.resolve().then(() => console.log("promise in timeout"));
}, 0);

Promise.resolve().then(() => {
  console.log("promise 1");
  queueMicrotask(() => console.log("microtask in promise"));
});

console.log("sync 2");
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// sync 1
// sync 2
// promise 1
// microtask in promise
// timeout
// promise in timeout
```

**Qadam-baqadam vizualizatsiya:**

```
Qadam 1: console.log("sync 1")
  Call Stack:    [global, console.log]
  Microtask Q:   []
  Macrotask Q:   []
  Output:        "sync 1"

Qadam 2: setTimeout(...)
  Call Stack:    [global, setTimeout]
  Microtask Q:   []
  Macrotask Q:   [timeout_callback]  ← Web API dan keladi
  Output:        —

Qadam 3: Promise.resolve().then(...)
  Call Stack:    [global]
  Microtask Q:   [promise1_callback]
  Macrotask Q:   [timeout_callback]
  Output:        —

Qadam 4: console.log("sync 2")
  Call Stack:    [global, console.log]
  Microtask Q:   [promise1_callback]
  Macrotask Q:   [timeout_callback]
  Output:        "sync 2"

Qadam 5: Microtask — promise1_callback
  Call Stack:    [promise1_callback]
  Microtask Q:   [microtask_in_promise]  ← yangi microtask qo'shildi
  Macrotask Q:   [timeout_callback]
  Output:        "promise 1"

Qadam 6: Microtask — microtask_in_promise
  Call Stack:    [microtask_in_promise]
  Microtask Q:   []
  Macrotask Q:   [timeout_callback]
  Output:        "microtask in promise"

Qadam 7: Macrotask — timeout_callback
  Call Stack:    [timeout_callback]
  Microtask Q:   [promise_in_timeout]  ← yangi microtask
  Macrotask Q:   []
  Output:        "timeout"

Qadam 8: Microtask — promise_in_timeout
  Call Stack:    [promise_in_timeout]
  Microtask Q:   []
  Macrotask Q:   []
  Output:        "promise in timeout"
```

</details>

---

### Mashq 4: Starvation Tuzatish (Qiyin)

**Savol:** Quyidagi kod browser'ni muzlatadi. Uni tuzating — natija bir xil bo'lsin, lekin UI bloklanmasin.

```javascript
// ❌ Bu kod browser'ni muzlatadi (starvation)
function processItems(items) {
  let index = 0;

  function next() {
    if (index < items.length) {
      console.log(`Processing: ${items[index]}`);
      index++;
      // Microtask — starvation!
      queueMicrotask(next);
    }
  }

  next();
}

processItems(Array.from({ length: 100000 }, (_, i) => `item-${i}`));
```

<details>
<summary>Javob</summary>

```javascript
// ✅ Yechim: Macrotask (setTimeout) + batch processing

function processItems(items) {
  const BATCH_SIZE = 1000;
  let index = 0;

  function processBatch() {
    const end = Math.min(index + BATCH_SIZE, items.length);

    while (index < end) {
      console.log(`Processing: ${items[index]}`);
      index++;
    }

    if (index < items.length) {
      // setTimeout — macrotask, UI refresh imkoniyati beradi
      setTimeout(processBatch, 0);
    }
  }

  processBatch();
}

processItems(Array.from({ length: 100000 }, (_, i) => `item-${i}`));
```

**Tushuntirish:**
1. **Muammo:** `queueMicrotask` recursive chaqirilganda, microtask queue **hech qachon** bo'sh bo'lmaydi. Event Loop macrotask'larga va rendering'ga o'ta olmaydi → browser muzlaydi.
2. **Yechim:** `setTimeout(fn, 0)` macrotask yaratadi. Har bir batch orasida Event Loop boshqa task'larni (rendering, click event'lar) bajara oladi.
3. **Batch processing:** 1000 ta itemni sync bajaramiz (tez), keyin setTimeout bilan keyingi batch'ga o'tamiz. Bu "breathing room" beradi.

</details>

---

### Mashq 5: Node.js Event Loop (Qiyin)

**Savol:** Quyidagi Node.js kodining output tartibini aniqlang va nima uchun shunday ekanini tushuntiring.

```javascript
const fs = require("fs");

console.log("1");

fs.readFile(__filename, () => {
  console.log("2");
  setTimeout(() => console.log("3"), 0);
  setImmediate(() => console.log("4"));
  process.nextTick(() => console.log("5"));
  Promise.resolve().then(() => console.log("6"));
});

setTimeout(() => console.log("7"), 0);
setImmediate(() => console.log("8"));
process.nextTick(() => console.log("9"));
Promise.resolve().then(() => console.log("10"));

console.log("11");
```

<details>
<summary>Javob</summary>

```javascript
// Output (Node.js):
// 1
// 11
// 9
// 10
// 7       (yoki 8 — top-level da tartib garanti emas)
// 8       (yoki 7)
// 2
// 5
// 6
// 4
// 3
```

**Tushuntirish:**

**Sync phase:** `1`, `11`

**Har bir faza orasida:** nextTick → microtask queue
- `9` — `process.nextTick` (eng yuqori priority)
- `10` — `Promise.then` (microtask)

**Timers/Check faza (top-level):**
- `7` va `8` — tartib garanti emas (timer resolution ga bog'liq)

**I/O callback (poll faza) ichida:**
- `2` — fs.readFile callback (poll faza da)
- `5` — process.nextTick (faza orasida, eng birinchi)
- `6` — Promise.then (microtask, nextTick dan keyin)
- `4` — setImmediate (check faza — poll dan keyin)
- `3` — setTimeout (timers faza — keyingi loop'da)

**Asosiy qoida:** I/O callback ichida `setImmediate` DOIM `setTimeout(fn, 0)` dan oldin ishlaydi, chunki poll fazadan keyin check faza keladi.

</details>

---

## Xulosa

### Asosiy Fikrlar

1. **JavaScript single-threaded** — bitta Call Stack, bitta execution context. Lekin runtime (browser/Node.js) multi-threaded.

2. **Event Loop** — Call Stack va Queue'larni boshqaradigan cheksiz loop:
   - Call Stack bo'shmi? → Microtask Queue ni TO'LIQ bo'shat → Macrotask Queue dan BITTA ol → Render → Qayta boshla

3. **Priority tartib:**

```
Sync kod → process.nextTick (Node) → Microtask (Promise) → Macrotask (setTimeout) → rAF → Render
```

4. **Microtask'lar** (Promise.then, queueMicrotask) **har doim** macrotask'lardan (setTimeout, setInterval) oldin bajariladi.

5. **Macrotask Queue** dan **bittadan**, Microtask Queue dan **barchasini** oladi.

6. **setTimeout(fn, 0)** darhol bajarmaydi — macrotask queue ga qo'yadi, sync va microtask'lardan keyin ishlaydi.

7. **requestAnimationFrame** — render sikliga ulangan, animation uchun eng yaxshi usul.

8. **Node.js** Event Loop 6 fazadan iborat, `process.nextTick` eng yuqori priority, `setImmediate` check fazada ishlaydi.

9. **Starvation** — recursive microtask'lar browser'ni muzlatadi. `setTimeout` yoki `setImmediate` bilan oldini oling.

10. **UI blocking** — og'ir hisoblashni batch'larga bo'ling, Web Worker ishlating, yoki requestAnimationFrame bilan render sikliga sinxronlang.

### Event Loop Mental Model

```
┌────────────────────────────────────────────────────────┐
│                    EVENT LOOP                          │
│                                                        │
│   🔄 Cheksiz Loop:                                    │
│                                                        │
│   [Sync Code]                     ← Birinchi: HAMMASI │
│        ↓                                               │
│   [Microtask Queue]               ← BARCHASI          │
│   (Promise, queueMicrotask)          bajariladi        │
│        ↓                                               │
│   [Macrotask Queue]               ← BITTASI           │
│   (setTimeout, setInterval)          bajariladi        │
│        ↓                                               │
│   [Render]                        ← Kerak bo'lsa      │
│   (rAF, style, layout, paint)                          │
│        ↓                                               │
│   [Microtask Queue]               ← Yana tekshir      │
│        ↓                                               │
│   ... takrorlanadi ...                                 │
└────────────────────────────────────────────────────────┘
```

### Keyingi Qadam

Event Loop ni tushunganingizdan keyin, keyingi bo'limlarda biz async JavaScript'ning asosiy qurollarini o'rganamiz:
- [Bo'lim 12: Promises](12-promises.md) — async operatsiyalarni boshqarish
- [Bo'lim 13: Async/Await](13-async-await.md) — zamonaviy async syntax

Shuningdek, qayta ko'rib chiqing:
- [Bo'lim 1: JS Engine](01-js-engine.md) — Call Stack va Memory Heap haqida
