# Bo'lim 11: Event Loop

> Event Loop — JavaScript ning yagona thread'da asinxron operatsiyalarni boshqaradigan mexanizmi. U Call Stack, Web APIs, Microtask Queue va Macrotask Queue o'rtasidagi aloqani tartibga soladi.

---

## Mundarija

- [JavaScript Single-Threaded](#javascript-single-threaded)
- [Runtime Architecture](#runtime-architecture)
- [Event Loop Algoritmi — Step by Step](#event-loop-algoritmi--step-by-step)
- [Macrotasks (Task Queue)](#macrotasks-task-queue)
- [Microtasks (Microtask Queue)](#microtasks-microtask-queue)
- [setTimeout(fn, 0) — Nima Uchun Darhol Bajarmaydi](#settimeoutfn-0--nima-uchun-darhol-bajarmaydi)
- [queueMicrotask()](#queuemicrotask)
- [requestAnimationFrame — Qayerda Turadi](#requestanimationframe--qayerda-turadi)
- [requestIdleCallback — Idle Vaqtda Ishlash](#requestidlecallback--idle-vaqtda-ishlash)
- [Node.js Event Loop Farqlari](#nodejs-event-loop-farqlari)
- [Starvation — Microtask Queue To'xtovsiz To'lsa](#starvation--microtask-queue-toxtovsiz-tolsa)
- [Real-World Misol: UI Blocking va Yechimi](#real-world-misol-ui-blocking-va-yechimi)
- [Output Tartibini Aniqlang — Tricky Misollar](#output-tartibini-aniqlang--tricky-misollar)
- [Scheduler API — scheduler.postTask](#scheduler-api--schedulerposttask)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## JavaScript Single-Threaded

### Nazariya

JavaScript **single-threaded** til — bir vaqtda faqat **bitta** operatsiyani bajaradi. Bitta Call Stack, bitta execution context, bitta code path. Har qanday paytda faqat bitta funksiya ishlab turadi.

JavaScript nima uchun single-threaded qilib yaratilgan? U dastlab browser uchun — DOM ni manipulatsiya qilish uchun yaratilgan. Agar ikkita thread bir vaqtda bitta DOM elementni o'zgartirmoqchi bo'lsa, **race condition** yuzaga keladi — natija oldindan aytib bo'lmas edi. Brendan Eich single-threaded dizaynni tanladi:

1. **DOM safety** — bir vaqtda bitta thread DOM ga yozadi, conflict yo'q
2. **Simplicity** — deadlock, mutex, lock kabi muammolar yo'q
3. **Predictability** — kod ketma-ketlikda bajariladi, debug qilish oson

Lekin single-threaded degani sekin degani emas. JavaScript o'zi single-threaded bo'lsa-da, u ishlaydigan muhit (browser yoki Node.js) **multi-threaded**. `setTimeout`, `fetch`, `fs.readFile` kabi operatsiyalar aslida browser/Node.js ning boshqa thread'larida bajariladi — JavaScript faqat natijalarni qaytarib oladi. Shu mexanizm tufayli JavaScript non-blocking tarzda ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da **"job"** tushunchasi bor — lekin Event Loop o'zi **spec dan tashqarida**. Event Loop browser uchun HTML spec da ta'riflangan, Node.js uchun esa libuv kutubxonasida implement qilingan.

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

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

JavaScript bitta thread da ishlaydi — kod ketma-ket bajariladi:

```javascript
console.log("1");
console.log("2");
console.log("3");

// Output (har doim shu tartibda):
// 1
// 2
// 3
```

Agar bitta operatsiya uzoq davom etsa, HAMMA narsa to'xtaydi — bu **blocking** deyiladi:

```javascript
// ❌ Blocking — og'ir hisoblash main thread'ni to'xtatadi
function heavyCalculation() {
  let sum = 0;
  for (let i = 0; i < 10_000_000_000; i++) {
    sum += i;
  }
  return sum;
}

console.log("Boshladik");
heavyCalculation(); // Bu yerda HAMMA narsa muzlab qoladi
console.log("Tugadi"); // Bu qator 10+ soniya kutadi
// Buttonlar bosilmaydi, animatsiyalar to'xtaydi, scroll ishlamaydi
```

Asinxron operatsiyalar orqali main thread bloklanmaydi:

```javascript
// ✅ Non-blocking — fetch boshqa thread'da ishlaydi
console.log("Boshladik");

fetch("https://api.example.com/data")
  .then(response => response.json())
  .then(data => console.log("Data keldi:", data));

console.log("Fetch yuborildi, boshqa ishlar davom etadi");

// Output:
// "Boshladik"
// "Fetch yuborildi, boshqa ishlar davom etadi"
// "Data keldi: {...}"  (bir necha millisekunddan keyin)
// fetch() browser ning network thread'ida ishlaydi
// JavaScript thread bloklanmaydi — boshqa kod davom etadi
```

</details>

---

## Runtime Architecture

### Nazariya

JavaScript o'zi single-threaded bo'lsa-da, u **yolg'iz ishlamaydi**. Browser yoki Node.js uni o'rab turgan muhitni ta'minlaydi — bu muhit **Runtime Environment** deyiladi. Runtime Environment beshta asosiy komponentdan tashkil topgan:

1. **Call Stack** — hozir bajarilayotgan funksiyalar (LIFO)
2. **Web APIs** — browser bergan asinxron API'lar (setTimeout, fetch, DOM Events)
3. **Macrotask Queue** (Task Queue) — tayyor callback'lar navbati (FIFO)
4. **Microtask Queue** — yuqori prioritetli callback'lar navbati (Promise.then)
5. **Event Loop** — bularning barchasini boshqaradigan koordinator

Ko'p dasturchilar "JavaScript asinxron til" deb o'ylaydi — aslida JavaScript **sinxron** til, lekin runtime environment asinxron imkoniyatlarni beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

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
│  │  LIFO (Last In   │                 │ Callback tayyor bo'lganda     │
│  │  First Out)      │                 ↓                               │
│  └──────────────────┘                                                │
│           ↑                                                          │
│           │         ┌────────────────────────────────────────┐       │
│           │         │          MICROTASK QUEUE                │       │
│           │         │  ┌──────────┬──────────┬─────────────┐ │       │
│           │         │  │ Promise  │ Promise  │ queueMicro  │ │       │
│           │         │  │ .then()  │ .catch() │  task()     │ │       │
│           │         │  └──────────┴──────────┴─────────────┘ │       │
│           │         └──────────────────┬─────────────────────┘       │
│           │                            │ YUQORI PRIORITY             │
│           │                            │                              │
│    ┌──────┴────────────────────────────┴────────────┐                │
│    │                  EVENT LOOP                     │                │
│    │                                                 │                │
│    │  1. Macrotask Queue dan bitta task ol → bajar    │                │
│    │  2. Microtask Queue ni to'liq bo'shat           │                │
│    │  3. Render (kerak bo'lsa)                       │                │
│    │  4. Qaytadan 1-dan boshla                       │                │
│    │                                                 │                │
│    └──────┬──────────────────────────────────────────┘                │
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

</details>

### Har Bir Komponent Haqida

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

#### 3. Macrotask Queue (Task Queue)

Web API'lar o'z ishini tugatganda, callback'ni **Macrotask Queue** ga qo'yadi. Bu FIFO (First In, First Out) navbat — kim avval kelsa, o'sha avval bajariladi.

#### 4. Microtask Queue

Microtask Queue — **yuqoriroq prioritetli** navbat. Promise `.then()`, `.catch()`, `.finally()`, `queueMicrotask()`, `MutationObserver` callback'lari shu yerga tushadi.

**Kritik farq:** Event Loop har bir macrotask'dan keyin **barcha** microtask'larni bajaradi. Macrotask Queue dan esa **bittadan** oladi.

#### 5. Event Loop

Event Loop — **cheksiz loop**. U Call Stack va Queue'larni kuzatib turadi va callback'larni to'g'ri tartibda bajarilishini ta'minlaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

setTimeout chaqirganda nima sodir bo'lishini qadam-baqadam ko'ramiz:

```javascript
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

To'liq runtime flow — barcha komponentlarni ko'rish:

```javascript
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

// Nima uchun bu tartibda?
// 1. Synchronous kodlar birinchi (Call Stack da darhol bajariladi)
// 2. Microtasks keyingi (Promise.then — yuqori priority)
// 3. Macrotasks eng oxirda (setTimeout — past priority)
```

</details>

---

## Event Loop Algoritmi — Step by Step

### Nazariya

Event Loop algoritmi — JavaScript asinxronligini tushunishning **kaliti**. Bu algoritmni bilish degani — istalgan asinxron kodning output tartibini oldindan aytib bera olish demak.

HTML Living Standard bo'yicha Event Loop algoritmining mohiyati quyidagi cheksiz sikldan iborat:
1. Macrotask Queue dan **bitta** task ol va bajar (yoki dastlabki script bajarilishini kut)
2. Microtask checkpoint — Microtask Queue ni **to'liq** bo'shat
3. Kerak bo'lsa render qil (requestAnimationFrame shu yerda)
4. Boshidan boshla

Bu algoritmning eng muhim nuqtasi — microtask va macrotask orasidagi **asimmetriya**: har bir macrotask'dan keyin microtask'lar **barchasi** bajariladi (ichidan yangi microtask qo'shilsa ham), lekin macrotask'dan faqat **bittasi** olinadi. Shu sababli Promise callback'lari doim setTimeout callback'laridan oldin ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

HTML Living Standard (spec) bo'yicha Event Loop quyidagicha ishlaydi:

**Qadam 1: Macrotask Queue dan bitta task olish**
- Eng eski (birinchi qo'shilgan) task ni olamiz (yoki dastlabki script bajarilishini kutamiz)
- Uni Call Stack ga qo'yamiz va bajaramiz
- **Faqat BITTA** macrotask olinadi — keyin microtask checkpoint bo'ladi

**Qadam 2: Microtask checkpoint**
- Microtask Queue dagi **BARCHA** task'larni ketma-ket bajaramiz
- Agar microtask bajarilayotganda yangi microtask qo'shilsa — **uni ham** bajaramiz
- Microtask Queue **butunlay bo'sh** bo'lguncha davom etamiz

**Qadam 3: Rendering**
- Browser ~16.6ms da bir marta render qiladi (60fps)
- Har bir Event Loop iteration'da render bo'lavermaydi — browser o'zi qaror qiladi
- `requestAnimationFrame` callback'lari shu bosqichda ishlaydi

**Qadam 4: Qaytadan boshdan**
- Loop to'xtamaydi — doimiy aylanadi
- Agar hech narsa yo'q bo'lsa — "idle" holatda kutadi

```
       ┌─────────────────────────────────────────┐
       │        EVENT LOOP ALGORITHM              │
       │                                          │
       │  ┌─────────────────────────────────┐     │
  ┌───→│  │ 1. Macrotask Queue dan          │     │
  │    │  │    BITTA task ol → bajar        │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓                            │
  │    │  ┌─────────────────────────────────┐     │
  │    │  │ 2. Microtask checkpoint         │     │
  │    │  │    → BARCHASINI bajar           │     │
  │    │  │    (yangi microtask qo'shilsa   │     │
  │    │  │     ularni ham bajar!)          │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓                            │
  │    │  ┌─────────────────────────────────┐     │
  │    │  │ 3. Rendering (agar kerak bo'lsa)│     │
  │    │  │    - Style hisoblash            │     │
  │    │  │    - Layout (reflow)            │     │
  │    │  │    - Paint                      │     │
  │    │  │    - requestAnimationFrame      │     │
  │    │  └──────────┬──────────────────────┘     │
  │    │             │                            │
  │    │             ↓                            │
  └────│─────── 1-qadamga qayt                    │
       │                                          │
       └─────────────────────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Event Loop algoritmini qadam-baqadam kuzatamiz:

```javascript
console.log("START");                         // Sync
setTimeout(() => console.log("TIMEOUT"), 0);  // → Macrotask Queue
Promise.resolve()
  .then(() => console.log("PROMISE 1"))       // → Microtask Queue
  .then(() => console.log("PROMISE 2"));      // → (1-microtask ichida) Microtask Queue
console.log("END");                           // Sync

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

4. Rendering (kerak bo'lsa)

5. Qaytadan boshlash → hech narsa yo'q → idle
```

Murakkabroq misol — microtask ichida microtask:

```javascript
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
    → queueMicrotask(F) → Microtask Queue: [F_callback]
    → E_callback resolve bo'ldi → G_callback queue ga
  → F_callback → "F" chiqadi  (avvalroq qo'shilgan)
  → G_callback → "G" chiqadi

  Microtask Queue bo'sh.

Macrotask Queue dan bitta: B_callback
  → "B" chiqadi
  → Promise.resolve().then(C) → Microtask Queue: [C_callback]

  Microtask checkpoint:
  → C_callback → "C" chiqadi

Macrotask Queue dan bitta: D_callback
  → "D" chiqadi
```

> **Eslatma:** `queueMicrotask(F)` sinxron tarzda microtask queue ga qo'shiladi. `.then(() => G)` esa faqat E callback resolve bo'lgandan keyin queue ga tushadi. Tartib: E → F → G.

</details>

---

## Macrotasks (Task Queue)

### Nazariya

Macrotask (yoki oddiy "task") — Event Loop ning **asosiy ish birligi**. Har bir Event Loop iteration'da faqat **bitta** macrotask olinadi va bajariladi — keyin esa microtask checkpoint bo'ladi.

Macrotask turlari:
- `setTimeout` callback'lari
- `setInterval` callback'lari
- I/O operatsiyalar (Node.js)
- UI rendering event'lari
- `MessageChannel` callback'lari
- `setImmediate` (faqat Node.js)

Macrotask nima uchun bittadan olinadi? Bu dizayn qaror browser'ning responsive bo'lishi uchun qilingan — agar barcha macrotask'lar bir yo'la bajarilsa, ular orasida render va microtask tekshiruvi bo'lmasdi. Bittadan olish orqali Event Loop har bir macrotask'dan keyin microtask'larni bajarishga, render qilishga va boshqa muhim ishlarga vaqt ajratadi.

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

0ms timeout ham darhol bajarmaydi — Macrotask Queue ga qo'yadi:

```javascript
setTimeout(() => console.log("timeout 1"), 0);
setTimeout(() => console.log("timeout 2"), 0);
setTimeout(() => console.log("timeout 3"), 0);
console.log("sync");

// Output:
// sync
// timeout 1
// timeout 2
// timeout 3

// Har bir setTimeout callback alohida macrotask
// Ular KETMA-KET, har bir Event Loop tick'da bittadan bajariladi
```

Macrotask'lar orasida microtask'lar bajariladi — bu eng muhim xususiyat:

```javascript
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

// Har bir macrotask'dan keyin microtask checkpoint bo'ladi
// Shuning uchun "Microtask from Macrotask 1" "Macrotask 2" dan OLDIN chiqadi
```

setInterval — takrorlanuvchi macrotask:

```javascript
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
// Interval: 1
// Interval: 2
// Interval: 3
// Interval to'xtatildi
```

</details>

---

## Microtasks (Microtask Queue)

### Nazariya

Microtask — macrotask'dan **kichikroq** va **yuqoriroq priority**ga ega task turi. Event Loop har bir macrotask'dan keyin Microtask Queue ni **TO'LIQ BO'SHATADI** — bu microtask'larning eng muhim xususiyati.

Microtask turlari:
- `Promise.then()` / `.catch()` / `.finally()` callback'lari
- `queueMicrotask()` callback'lari
- `MutationObserver` callback'lari
- Node.js da `process.nextTick()` (hatto oddiy microtask'lardan ham yuqoriroq priority bilan)

Microtask checkpoint algoritmining jiddiy oqibati — agar har bir microtask yangi microtask qo'shsa, bu loop **hech qachon tugamaydi** va macrotask'lar hamda rendering to'xtab qoladi (starvation — bu haqda keyinroq).

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da microtask'lar **"PromiseJobs"** deb ataladi. HTML spec'da esa **"microtask queue"** deb ataladi.

Microtask checkpoint algoritmi:

```
while (microtaskQueue.length > 0) {
    let task = microtaskQueue.shift();  // Birinchisini ol (FIFO)
    execute(task);                       // Bajir
    // Agar task ichida yangi microtask qo'shilsa
    // — loop davom etadi, uni HAM bajaradi
}

// XAVF: agar har bir microtask yangi microtask qo'shsa
// — bu loop HECH QACHON tugamaydi → STARVATION
```

Microtask Queue bo'shatilish davomida yangi microtask qo'shilsa — ular **shu** checkpoint'da bajariladi, keyingi tick'ga qoldirilmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Promise.then() — eng keng tarqalgan microtask:

```javascript
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

// .then() callback'lari microtask — sync koddan keyin, lekin macrotask'lardan oldin
```

Microtask ichida yangi microtask — recursive:

```javascript
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

// BARCHA microtask'lar (ichma-ich bo'lsa ham) macrotask'dan OLDIN tugaydi
// Chunki microtask checkpoint queue TO'LIQ bo'shguncha ishlaydi
```

MutationObserver — DOM o'zgarishlarini kuzatish (microtask sifatida ishlaydi):

```javascript
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

</details>

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

`setTimeout(fn, 0)` deganda ko'pchilik "0 millisekund = darhol" deb o'ylaydi. Lekin bu **noto'g'ri**. `setTimeout(fn, 0)` callback'ni **Macrotask Queue**ga qo'yadi, va u faqat:
1. Hozirgi sinxron kod tugagandan keyin
2. Barcha microtask'lar bajarilgandan keyin
3. Event Loop macrotask queue'ga navbat kelganda bajariladi

`setTimeout(fn, 0)` aslida "ushbu kodni joriy execution kontekstidan keyin, keyingi Event Loop tick'da bajarin" degani. Bu UI ni blokirovka qilmaslik uchun og'ir hisoblashlarni bo'laklarga ajratishda va render sikliga imkon berishda ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Aslida `setTimeout(fn, 0)` **0ms ham emas**:

1. **HTML spec** bo'yicha: agar nested setTimeout 5+ marta chaqirilgan bo'lsa, minimum delay **4ms** bo'ladi (clamping)
2. **Browser** — background tab'larda setTimeout minimum **1000ms** gacha throttle qilinishi mumkin
3. **Amalda** — hatto active tab'da ham OS scheduler tufayli ~1-4ms delay bo'ladi

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
console.log("A — sync");

setTimeout(() => {
  console.log("B — setTimeout(0)");
}, 0);

console.log("C — sync");

// Output:
// A — sync
// C — sync
// B — setTimeout(0)

// B eng oxirda! Chunki:
// 1. "A" — sync, darhol
// 2. setTimeout(B, 0) — callback ni Macrotask Queue ga QO'YADI (bajaramaydi!)
// 3. "C" — sync, darhol
// 4. Call Stack bo'sh → microtask queue (bo'sh) → macrotask queue → B
```

setTimeout(0) vs Promise — kim birinchi?

```javascript
setTimeout(() => console.log("setTimeout"), 0);
Promise.resolve().then(() => console.log("Promise"));
console.log("Sync");

// Output:
// Sync
// Promise
// setTimeout

// Promise (microtask) DOIM setTimeout (macrotask) dan oldin!
// 0ms delay ham buni o'zgartirmaydi
```

setTimeout minimum delay — nested call'larda 4ms clamping:

```javascript
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

// 6-chi nested setTimeout dan boshlab browser minimum 4ms delay qo'yadi
// HTML spec: "If nesting level is greater than 5, clamp timeout to at least 4ms"
// Ya'ni nesting level > 5 bo'lganda (6-chi va undan keyin) clamping boshlanadi
```

setTimeout(0) — real use case — UI unblocking:

```javascript
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

</details>

---

## queueMicrotask()

### Nazariya

`queueMicrotask(callback)` — microtask queue'ga to'g'ridan-to'g'ri callback qo'shish uchun API. U `Promise.resolve().then(callback)` bilan deyarli bir xil ishlaydi, lekin semantik jihatdan aniqroq va bir oz samaraliroq — chunki Promise object yaratilmaydi.

`queueMicrotask` qachon ishlatiladi:
- Microtask queue'ga callback qo'shish kerak bo'lganda, lekin Promise semantikasi kerak bo'lmaganda
- Sinxron koddan keyin, lekin rendering va macrotask'lardan oldin biror ishni bajarishda
- Bir nechta sinxron o'zgarishlarni batch'lab, bitta microtask'da qayta ishlashda

<details>
<summary><strong>Under the Hood</strong></summary>

`queueMicrotask` HTML spec da aniqlangan. U `Promise.resolve().then()` dan farqli ravishda yangi Promise object yaratmaydi — callback'ni to'g'ridan-to'g'ri microtask queue'ga qo'shadi. Bu ozgina performance ustunlik beradi.

Lekin **tartib** jihatidan `queueMicrotask` va `Promise.then` bir xil queue'da turadi — FIFO tartibida bajariladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
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

// queueMicrotask va Promise.then ikkalasi ham microtask
// Tartib: FIFO — qaysi biri avval queue ga tushsa, o'sha avval bajariladi
```

Real-world use case — batch DOM updates:

```javascript
let pendingUpdates = [];
let updateScheduled = false;

function scheduleUpdate(element, value) {
  pendingUpdates.push({ element, value });

  if (!updateScheduled) {
    updateScheduled = true;
    queueMicrotask(() => {
      // Barcha o'zgarishlarni bitta batch'da bajaramiz
      pendingUpdates.forEach(({ element, value }) => {
        element.textContent = value;
      });
      pendingUpdates = [];
      updateScheduled = false;
    });
  }
}

// Bir nechta update chaqiriladi — lekin DOM faqat BIR marta yangilanadi
scheduleUpdate(titleEl, "Yangi sarlavha");
scheduleUpdate(bodyEl, "Yangi matn");
scheduleUpdate(footerEl, "Yangi footer");
// Uchala o'zgarish bitta microtask'da batch'lanadi
```

</details>

---

## requestAnimationFrame — Qayerda Turadi

### Nazariya

`requestAnimationFrame(callback)` (qisqacha **rAF**) — na microtask, na macrotask. U Event Loop'ning **rendering bosqichida** ishlaydi — browser har gal ekranga chizishdan (paint) oldin rAF callback'larini chaqiradi. Bu uni animatsiyalar uchun ideal qiladi chunki u browser'ning render sikli bilan sinxronlashtirilgan (odatda 60fps = har 16.6ms).

`setTimeout` emas, `rAF` ishlatish kerak bo'lgan sabablar:
- `setTimeout` macrotask bo'lib, u render sikli bilan sinxronlashtirilmagan — natijada animatsiyalar "jitter" qilishi mumkin
- `rAF` browser o'zi render qilmoqchi bo'lganida aynan o'sha paytda chaqiriladi — silliq, 60fps animatsiya uchun optimal
- Tab ko'rinmayotganda rAF avtomatik to'xtaydi — battery va CPU tejaydi

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

<details>
<summary><strong>Under the Hood</strong></summary>

**rAF xususiyatlari:**
- Har frame'da **bir marta** chaqiriladi (~60fps = ~16.6ms)
- Agar tab background'da bo'lsa — **to'xtaydi** (battery/CPU tejash)
- Animation uchun **eng yaxshi** usul (vsync bilan sinxronlashgan)
- `setTimeout(fn, 16)` dan aniqroq — chunki browser ning render sikliga ulangan

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

rAF vs setTimeout — animatsiya uchun farq:

```javascript
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

rAF, setTimeout, Promise — tartibni aniqlash:

```javascript
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

// Tartib sababi:
// 1, 5 — synchronous (darhol)
// 4 — microtask (eng yuqori async priority)
// 3 — macrotask (microtask'lardan keyin)
// 2 — rAF (render bosqichida — macrotask'dan HAM keyin)
//
// DIQQAT: 3 va 2 tartibi browser va frame timing'ga qarab
// o'zgarishi mumkin. Agar render shu tick'da bo'lmasa,
// rAF keyingi render'gacha kutishi mumkin.
```

rAF bilan silliq scroll-to-top:

```javascript
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

</details>

---

## requestIdleCallback — Idle Vaqtda Ishlash

### Nazariya

`requestIdleCallback(callback)` — browser **bo'sh vaqtida** (idle) callback'ni bajaradigan API. U Event Loop'da barcha muhim ishlar (macrotask, microtask, render) tugagandan keyin, keyingi frame boshlanguncha qolgan vaqtda ishlaydi.

Bu API past priority'li ishlar uchun mo'ljallangan — analytics yuborish, prefetching, lazy initialization kabi foydalanuvchi tajribasiga bevosita ta'sir qilmaydigan operatsiyalar. `requestIdleCallback` ga berilgan callback `IdleDeadline` object oladi — unda `timeRemaining()` method bor, bu frame tugashigacha qancha vaqt qolganini (millisekund) ko'rsatadi.

`requestIdleCallback` `requestAnimationFrame` dan farqi: rAF har render oldidan ishlaydi (yuqori priority), `requestIdleCallback` esa faqat browser bo'sh bo'lgandagina ishlaydi (past priority). Agar browser band bo'lsa, `requestIdleCallback` umuman chaqirilmasligi ham mumkin — shuning uchun `timeout` option bilan cheklov qo'yish mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

```
Frame budget (16.6ms @ 60fps):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
│ Macrotask │ Microtasks │ rAF │ Style│Layout│Paint│ IDLE │
│   2ms     │   1ms      │ 1ms │ 2ms  │ 3ms  │ 2ms │~5.6ms│
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                                     ↑
                                            requestIdleCallback
                                            shu oraliqda ishlaydi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Past priority'li ishni idle vaqtda bajarish
requestIdleCallback((deadline) => {
  // deadline.timeRemaining() — frame oxirigacha qolgan vaqt (ms)
  while (deadline.timeRemaining() > 0) {
    // Past priority'li ish: analytics, prefetch, lazy init
    processNextAnalyticsEvent();
  }
});

// timeout bilan — maksimum 2 soniya kutadi, keyin majburiy bajaradi
requestIdleCallback((deadline) => {
  sendAnalytics(pendingData);
}, { timeout: 2000 });
```

```javascript
// Real-world use case: katta ro'yxatni idle vaqtda prefetch qilish
const itemsToPreload = [...longList];

function preloadBatch(deadline) {
  while (itemsToPreload.length > 0 && deadline.timeRemaining() > 5) {
    const item = itemsToPreload.shift();
    preloadImage(item.imageUrl); // Past priority — foydalanuvchi kutmaydi
  }

  if (itemsToPreload.length > 0) {
    requestIdleCallback(preloadBatch); // Qolganlari keyingi idle'da
  }
}

requestIdleCallback(preloadBatch);
```

</details>

---

## Node.js Event Loop Farqlari

### Nazariya

Node.js Event Loop browser'nikidan sezilarli farq qiladi. Node.js **libuv** kutubxonasini ishlatadi — bu C da yozilgan cross-platform asinxron I/O kutubxonasi. Browser'dagi oddiy "macrotask → microtask → render" siklidan farqli o'laroq, Node.js Event Loop **6 ta faza (phase)** dan iborat: Timers, Pending Callbacks, Idle/Prepare, Poll, Check, Close Callbacks.

Bu farqlarni bilish muhim chunki `setImmediate` vs `setTimeout(fn, 0)` ning farqi, `process.nextTick()` ning boshqa microtask'lardan yuqori priority'si, va Poll phase'ning I/O event'larni kutishi — bularning barchasi Node.js server kodining to'g'ri ishlashiga ta'sir qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

### Har Bir Faza Haqida

**1. Timers** — `setTimeout` va `setInterval` callback'lari. Timer expired bo'lgan callback'lar shu fazada bajariladi.

**2. Pending Callbacks** — ba'zi system-level I/O callback'lari (masalan, TCP `ECONNREFUSED`).

**3. Idle/Prepare** — faqat Node.js ichki ishlatadi.

**4. Poll** — eng muhim faza:
- Yangi I/O event'larini kutadi
- I/O callback'larini bajaradi (fs.readFile, http response)
- Agar poll queue bo'sh bo'lsa:
  - `setImmediate` bor → check faza ga o'tadi
  - Timer expired bor → timers faza ga o'tadi
  - Hech narsa yo'q → yangi event kutadi

**5. Check** — `setImmediate()` callback'lari. Bu **faqat Node.js** da bor.

**6. Close Callbacks** — `socket.on('close', ...)` kabi close event'lari.

### `process.nextTick()` vs `setImmediate()`

| Xususiyat | `process.nextTick()` | `setImmediate()` |
|-----------|---------------------|------------------|
| Qachon ishlaydi | **Har bir faza ORASIDA**, barchasidan oldin | **Check faza** da |
| Priority | Eng yuqori (microtask'dan ham oldin) | Macrotask (check phase) |
| Queue | nextTick Queue (alohida) | Check Queue |
| Starvation xavfi | **HA** — recursive nextTick loop'ni to'xtatmaydi | Yo'q |
| Nomlash | Aslida "immediate" ishlaydigan shu | Aslida "next tick" da ishlaydigan shu |

> `process.nextTick` va `setImmediate` nomlari teskari qo'yilgan. `nextTick` darhol ishlaydi, `setImmediate` esa keyingi fazada. Bu Node.js ning tarixiy xatosi — o'zgartirishning iloji yo'q (backward compatibility).

<details>
<summary><strong>Kod Misollari</strong></summary>

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
// 2 — setTimeout  }
// 3 — setImmediate}   ← BU IKKALASINING TARTIBI KAFOLATLANMAGAN (top-level da)
// Sababi: timer resolution ga bog'liq — ba'zan setTimeout oldin, ba'zan setImmediate

// Tartib: sync → nextTick → microtask → macrotask
```

I/O callback ichida — `setImmediate` DOIM `setTimeout(fn, 0)` dan oldin:

```javascript
const fs = require("fs");

fs.readFile(__filename, () => {
  setTimeout(() => console.log("setTimeout"), 0);
  setImmediate(() => console.log("setImmediate"));
});

// Output (HAR DOIM):
// setImmediate
// setTimeout

// I/O callback poll faza da bajariladi.
// Poll dan keyin → Check faza (setImmediate) → keyin loop boshiga → Timers (setTimeout)
// Shuning uchun I/O ichida setImmediate DOIM birinchi.
```

Top-level da — tartib **garanti emas**:

```javascript
setTimeout(() => console.log("setTimeout"), 0);
setImmediate(() => console.log("setImmediate"));

// Output (OLDINDAN AYTIB BO'LMAYDI):
// Ba'zan: setTimeout → setImmediate
// Ba'zan: setImmediate → setTimeout

// Main module top-level da, poll faza yo'q.
// setTimeout(0) aslida ≈1ms. Agar 1ms o'tgan bo'lsa → setTimeout birinchi.
// Agar hali o'tmagan bo'lsa → setImmediate birinchi.
// Timer resolution OS ga bog'liq.
```

</details>

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

**Starvation** — microtask queue hech qachon bo'sh bo'lmaydigan xavfli holat. Har bir microtask yangi microtask qo'shsa, Event Loop **hech qachon** macrotask'larga yoki rendering'ga o'tmaydi — natijada UI muzlab qoladi, setTimeout callback'lari ishlamaydi, va browser "Not Responding" holatiga tushadi.

Bu muammo aynan microtask'larning "barchasini bo'shat" qoidasidan kelib chiqadi. Macrotask'larda bu muammo yo'q chunki ular bittadan olinadi. Lekin microtask'lar rekursiv qo'shilsa — Event Loop ularni tugatishga urinib qoladi.

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

// Bu kod browser'ni muzlatadi!
// Microtask queue hech qachon bo'sh bo'lmaydi
// → Macrotask'lar bajarilmaydi → Rendering to'xtaydi → UI muzlab qoladi
```

```javascript
// ✅ XAVFSIZ — setTimeout bilan "breathing room" berish

function safeRecursion() {
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
// ✅ Node.js da — setImmediate bilan recursive ishlar
// process.nextTick bilan ham starvation bo'ladi:

// ❌ Node.js da process.nextTick bilan starvation:
function badNextTick() {
  process.nextTick(() => {
    badNextTick(); // I/O callback'lar HECH QACHON ishlamaydi
  });
}

// ✅ Node.js hujjatlari tavsiyasi: setImmediate ishlating
function goodRecursion() {
  setImmediate(() => {
    // setImmediate check fazada ishlaydi
    // Orada boshqa faza'lar (poll, timers) ishlay oladi
    goodRecursion();
  });
}
```

</details>

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

UI blocking — JavaScript ning single-threaded tabiatidan kelib chiqadigan **eng katta amaliy muammo**. Agar Call Stack da og'ir hisoblash bajarilayotgan bo'lsa — browser **hech narsani** qila olmaydi: button click'lar, scroll, animatsiyalar, render — HAMMA narsa to'xtaydi.

Bu muammoni hal qilishning bir nechta usuli bor:
1. **Batch processing** — ishni kichik bo'laklarga ajratish (`setTimeout` yoki `rAF` bilan)
2. **Web Worker** — haqiqiy parallel thread
3. **requestIdleCallback** — browser bo'sh paytida ishlash
4. **Debounce** — ortiqcha chaqiruvlarni kamaytirish

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Real-world muammo: Search filter 10,000 ta itemni sync filterlaydi

const searchInput = document.getElementById("search");
const resultsList = document.getElementById("results");
const items = Array.from({ length: 10000 }, (_, i) => `Mahsulot ${i + 1}`);

searchInput.addEventListener("input", (e) => {
  const query = e.target.value.toLowerCase();

  // 10,000 ta itemni filterlash + DOM update — hammasi sync
  resultsList.innerHTML = "";
  const filtered = items.filter(item => item.toLowerCase().includes(query));

  filtered.forEach(item => {
    const li = document.createElement("li");
    li.textContent = item;
    resultsList.appendChild(li);
  });
  // Natija: har bir harf yozganda UI 100-500ms muzlaydi
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
}, 300));
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
      setTimeout(nextBatch, 0); // Har batch orasida Event Loop ga "nafas" berish
    }
  }

  nextBatch();
}

processInBatches(hugeDataArray, 500, (item) => {
  renderItem(item); // Har 500 ta itemdan keyin UI yangilanadi
});
```

```javascript
// ✅ Yechim 3: Web Worker — og'ir ishni boshqa thread'ga uzatish

// main.js
const worker = new Worker("heavy-worker.js");

searchInput.addEventListener("input", debounce((e) => {
  worker.postMessage({ query: e.target.value, items: items });
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

// Web Worker alohida thread'da ishlaydi
// Main thread (UI) hech qachon bloklanmaydi
// Lekin: Worker da DOM ga access yo'q
```

</details>

---

## Output Tartibini Aniqlang — Tricky Misollar

Bu bo'lim interview'da eng ko'p so'raladigan savollar turini o'z ichiga oladi. Har bir misolni sinchiklab tahlil qiling.

### Misol 1: Asosiy Tartib

```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");

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

// Javob: A, F, C, E, D, B

// Tahlil:
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

// Javob: C, E, A, B, D

// Tahlil:
// Sync tugagandan keyin:
//   Macrotask Queue: [A, B]
//   Microtask Queue: [C_callback, E_callback]
//
// Microtask checkpoint:
//   C → "C" + setTimeout(D) → Macrotask Queue: [A, B, D]
//   E → "E"
//
// Macrotask: A → "A"
// Macrotask: B → "B"
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

// Javob: 1, 6, 2, 3, 4, 5

// Tahlil:
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

// Javob: A, 1, B, 2, 3

// async/await = Promise + .then() syntactic sugar
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

// Javob: 5, 6, 1, 2, 3, 4

// Tahlil:
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

### Misol 7: Super Tricky — Ko'p Qatlamli

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

// Javob: 1, 6, 10, 7, 9, 2, 3, 4, 8, 5

// Qadam-baqadam:
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

// Javob: 1, 2, 4, 3
// Promise constructor SYNC ishlaydi!
// resolve() Promise ni fulfilled qiladi, lekin .then() callback
// MICROTASK sifatida keyinroq ishlaydi
```

---

## Scheduler API — scheduler.postTask

### Nazariya

`scheduler.postTask()` — browser'ning yangi Scheduling API'si bo'lib, task'larni **priority** bilan queue ga qo'shish imkonini beradi. `setTimeout` va `requestIdleCallback` dan farqli o'laroq, bu API task'larga aniq priority berish va ularni runtime da boshqarish imkoniyatini taqdim etadi.

**3 ta priority darajasi:**
- **"user-blocking"** — eng yuqori priority. User interaction bilan bog'liq task'lar: input handling, animatsiya. Event Loop bu task'larni birinchi navbatda oladi
- **"user-visible"** — default priority. Foydalanuvchiga ko'rinadigan natija beradigan task'lar: ma'lumot render qilish, DOM yangilash
- **"background"** — eng past priority. Foydalanuvchi sezmaydigan task'lar: analytics, prefetch, cache yangilash. Browser bo'sh paytda bajaradi

**Event Loop bilan aloqasi:** Oddiy macrotask'lar (setTimeout) FIFO tartibda bajariladi — birinchi qo'shilgan birinchi bajariladi. `scheduler.postTask()` esa priority-based scheduling qiladi — "user-blocking" task "background" task'dan oldin bajariladi, qachon qo'shilganidan qat'iy nazar.

**AbortController** bilan task'ni cancel qilish mumkin — agar task hali bajarilmagan bo'lsa, uni queue'dan olib tashlaydi. Cancel qilingan task "AbortError" bilan reject bo'ladi.

**TaskController** — AbortController'ning kengaytmasi. Cancel qilishdan tashqari, task priority'sini **runtime da o'zgartirish** imkonini beradi. Masalan, foydalanuvchi scroll qilsa — background task'ni user-visible ga ko'tarish mumkin.

**setTimeout vs scheduler.postTask:**
- `setTimeout(fn, 0)` — priority yo'q, faqat FIFO, minimum 4ms delay (nested), cancel qilish uchun `clearTimeout` kerak
- `scheduler.postTask(fn)` — priority bor, Promise qaytaradi, AbortController bilan cancel, TaskController bilan dynamic priority

**Browser support:** Chrome 94+, Edge 94+. Firefox va Safari da hali yo'q (2024). Progressive enhancement sifatida ishlatish kerak — feature detection bilan.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Asosiy ishlatish — Priority bilan task qo'shish ===

// Eng yuqori — foydalanuvchi interaksiyasiga javob
scheduler.postTask(() => {
  console.log("user-blocking: input handling");
}, { priority: "user-blocking" });

// O'rta — ko'rinadigan natija
scheduler.postTask(() => {
  console.log("user-visible: render data");
}, { priority: "user-visible" });

// Past — fon ishi
scheduler.postTask(() => {
  console.log("background: analytics yuborish");
}, { priority: "background" });

// Output tartibi (priority bo'yicha):
// user-blocking: input handling
// user-visible: render data
// background: analytics yuborish

// scheduler.postTask Promise qaytaradi — natijani olish mumkin
const result = await scheduler.postTask(() => {
  return heavyCalculation();
}, { priority: "background" });
console.log("Natija:", result);
```

```javascript
// === AbortController bilan task cancel qilish ===

const controller = new AbortController();

scheduler.postTask(() => {
  console.log("Bu hech qachon bajarilmaydi");
  return fetchAnalytics();
}, {
  priority: "background",
  signal: controller.signal
}).catch(err => {
  if (err.name === "AbortError") {
    console.log("Task bekor qilindi");
  }
});

// Sahifa yopilmoqda — background task'larni cancel qilish
controller.abort();
// "Task bekor qilindi"
```

```javascript
// === TaskController — runtime da priority o'zgartirish ===

const taskController = new TaskController({ priority: "background" });

scheduler.postTask(() => {
  console.log("Data processing...");
  return processLargeDataset();
}, { signal: taskController.signal });

// Foydalanuvchi "Yuklanish" tugmasini bosdi — priority'ni ko'tarish
document.getElementById("loadBtn").addEventListener("click", () => {
  taskController.setPriority("user-blocking");
  console.log("Priority ko'tarildi: background → user-blocking");
});
```

```javascript
// === setTimeout vs scheduler.postTask taqqoslash ===

// setTimeout — FIFO, priority yo'q
setTimeout(() => console.log("setTimeout 1 — birinchi qo'shildi"), 0);
setTimeout(() => console.log("setTimeout 2 — ikkinchi qo'shildi"), 0);
// Doim: setTimeout 1, setTimeout 2 (tartib bo'yicha)

// scheduler.postTask — priority bo'yicha tartiblanadi
scheduler.postTask(
  () => console.log("background task — oldin qo'shildi"),
  { priority: "background" }
);
scheduler.postTask(
  () => console.log("user-blocking task — keyin qo'shildi"),
  { priority: "user-blocking" }
);
// Natija: user-blocking task keyin qo'shildi, user-blocking task birinchi ishlaydi

// === Feature detection — xavfsiz ishlatish ===
function scheduleTask(fn, priority = "user-visible") {
  if ("scheduler" in globalThis && "postTask" in scheduler) {
    return scheduler.postTask(fn, { priority });
  }
  // Fallback — oddiy setTimeout
  return new Promise(resolve => {
    setTimeout(() => resolve(fn()), 0);
  });
}
```

</details>

---

## Edge Cases va Gotchas

### `setInterval` drift — recursive `setTimeout` dan farqi

`setInterval(fn, 1000)` — fn ni har 1000ms'da chaqirishga **harakat qiladi**, lekin kafolat bermaydi. Agar fn 800ms davom etsa, keyingi call 200ms keyin (interval'ning qolgan qismi) sodir bo'ladi — fn tugashini **kutmaydi**. Agar fn interval'dan uzoqroq davom etsa, call'lar **to'planishi** yoki **overlap bo'lishi** mumkin. Recursive `setTimeout` esa har doim oldingi chaqiruv **tugaganidan keyin** delay boshlaydi.

```javascript
// ❌ setInterval — drift + overlap xavfi:
let count = 0;
const start = Date.now();

const interval = setInterval(() => {
  count++;
  // Og'ir ish — 500ms davom etadi
  const end = Date.now() + 500;
  while (Date.now() < end) {}
  console.log(`Interval ${count}: ${Date.now() - start}ms`);

  if (count === 3) clearInterval(interval);
}, 1000);

// Natija (taxminiy):
// Interval 1: 1000ms  (1000 delay + 500 ish)
// Interval 2: 2000ms  (keyingi interval 500ms o'tib darhol boshlandi!)
// Interval 3: 3000ms  (fn'lar "yetishadi" — drift kompensatsiyasi)

// ✅ Recursive setTimeout — barqaror delay:
let tickCount = 0;
const tickStart = Date.now();

function tick() {
  tickCount++;
  const end = Date.now() + 500;
  while (Date.now() < end) {}
  console.log(`Tick ${tickCount}: ${Date.now() - tickStart}ms`);

  if (tickCount < 3) setTimeout(tick, 1000);
}
setTimeout(tick, 1000);

// Natija:
// Tick 1: 1000ms   (1000 delay + 500 ish)
// Tick 2: 2500ms   (1500 + 1000 delay + 500 ish)
// Tick 3: 4000ms   (3000 + 1000 delay + 500 ish)
// Har tick orasida to'liq 1500ms — kafolatlangan
```

**Nima uchun:** `setInterval` browser ichida "schedule at fixed time points" modelida ishlaydi — `T+1000`, `T+2000`, `T+3000`. Agar callback oldingi tick'dan oshib ketsa, keyingisi darhol ishga tushadi. `setTimeout` recursion esa har safar "hozirdan +1000ms keyin" modelida ishlaydi — drift yo'q, lekin total time uzoqroq.

**Qoida:** Fixed intervals kerak bo'lsa → `setInterval` (masalan, real-time clock). Barqaror gap kerak bo'lsa → recursive `setTimeout` (masalan, polling API).

---

### Background tab throttling — `setTimeout` 1000ms'gacha clamped

Modern browser'lar foydalanuvchi tab'ni background'ga o'tkazganda `setTimeout`/`setInterval`'ni **sezilarli darajada** sekinlashtiradi — battery va CPU tejash uchun. Bu bir nechta darajada bo'ladi:

- **Chrome/Edge:** Background tab'da deeply nested setTimeout → minimum **1000ms** clamping
- **Firefox:** Background tab'da minimum ~1000ms (yoki 10000ms agar tab "throttled")
- **Safari:** Agressiv throttling, tab deactivate bo'lsa butun Event Loop sekinlashadi

```javascript
// Foreground tab (active):
setInterval(() => {
  console.log(`Tick: ${Date.now()}`);
}, 100); // Har ~100ms ishlaydi

// User tab'ni background'ga o'tkazsa (Cmd+Tab):
// Chrome: ~1000ms'gacha clamped
// Firefox: ~1000ms
// Safari: hatto ko'proq (~1s+)
// Natija: "Har 100ms" kod bo'lsa ham, background'da "Har 1000ms" ishlaydi

// Muhim holatlar:
// 1. Music/audio player — user experience buziladi
// 2. Real-time clock / timer — background'da noto'g'ri ko'rsatadi
// 3. WebSocket reconnect logic — delay uzaytiriladi

// Yechim 1: Web Audio API (throttle ta'sir qilmaydi)
// Yechim 2: Page Visibility API bilan holat sinxronizatsiyasi
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "visible") {
    // Background'dan foreground'ga qaytdi — holat'ni tiklash
    resyncState();
  }
});
```

**Nima uchun:** Modern browser'lar user agent ga performance va battery optimization uchun ruxsat beradi — HTML spec'da "The user agent may apply throttling to timers on documents that are not visible" deb belgilangan. Bu foydalanuvchi experience'i uchun muhim: background tab'lar battery'ni tejaydi, asosiy tab tez ishlaydi.

**Yechim:** Vaqtga bog'liq kod yozayotgan bo'lsangiz — `Date.now()` bilan actual elapsed time hisoblang, `setInterval` tick count'iga ishonmang.

---

### `MessageChannel` — `setTimeout(0)` clamping'ni aylanib o'tish

`setTimeout(fn, 0)` nested bo'lganda minimum **4ms** delay clamping bor (HTML spec). Agar sizga haqiqiy "keyingi tick'da ishlat" kerak bo'lsa (clamping'siz), `MessageChannel` API bilan zero-delay macrotask yaratish mumkin.

```javascript
// Standard setTimeout(0) — 4ms nested clamping:
function scheduleWithTimeout(fn) {
  setTimeout(fn, 0);
}

// MessageChannel — true 0ms, clamping yo'q:
function scheduleWithMessageChannel(fn) {
  const channel = new MessageChannel();
  channel.port1.onmessage = fn;
  channel.port2.postMessage(null);
}

// Benchmark:
console.time("setTimeout(0)");
scheduleWithTimeout(() => console.timeEnd("setTimeout(0)"));
// ~4ms (nested clamping)

console.time("MessageChannel");
scheduleWithMessageChannel(() => console.timeEnd("MessageChannel"));
// ~0.1-1ms (no clamping)

// Use case: high-frequency batched DOM updates
let pendingUpdates = [];
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  const updates = pendingUpdates;
  pendingUpdates = [];
  updates.forEach(applyUpdate);
};

function scheduleUpdate(update) {
  if (pendingUpdates.length === 0) {
    channel.port2.postMessage(null); // true 0ms schedule
  }
  pendingUpdates.push(update);
}
```

**Nima uchun:** HTML spec'ning "minimum timer interval" qoidasi faqat `setTimeout`/`setInterval`'ga tegishli — `MessageChannel` `postMessage` task source'i boshqa category'da, clamping qo'llanilmaydi. Bu fact high-performance scheduling library'lar tomonidan ishlatiladi (masalan React Scheduler implementation).

**Eslatma:** `MessageChannel` ham **macrotask** — rendering va microtask'lar undan oldin ishlaydi. Agar microtask tezlikda kerak bo'lsa, `queueMicrotask` ishlatish kerak.

---

### `await` inside `forEach` — serializatsiya qilmaydi

`forEach` **sinxron** iterator — callback'ning return qiymatini ko'rmaydi va `await` qilmaydi. Shu sababli `forEach` ichida `async` callback ishlatish async operation'larni **parallel** ishga tushiradi (serial emas) va `forEach` darhol qaytaradi.

```javascript
// ❌ await forEach ichida — SERIAL emas, PARALLEL:
async function fetchAllBroken(urls) {
  urls.forEach(async (url) => {
    const response = await fetch(url);
    const data = await response.json();
    console.log(data);
  });

  console.log("Done"); // ❌ Bu fetch'lar TUGAMAY BURUN chiqadi!
}

// fetchAllBroken(["/a", "/b", "/c"]):
// 1. forEach darhol qaytadi — async callback'lar parallel ishlaydi
// 2. "Done" — darhol chiqadi (fetchlar hali tugamagan)
// 3. Fetch natijalar keyin chiqadi (noto'g'ri tartibda bo'lishi mumkin)

// ✅ for...of — to'g'ri serializatsiya:
async function fetchAllSerial(urls) {
  for (const url of urls) {
    const response = await fetch(url);
    const data = await response.json();
    console.log(data);
  }
  console.log("Done"); // ✅ BARCHA fetch'lar tugagandan keyin
}

// ✅ Promise.all — parallel (intentional):
async function fetchAllParallel(urls) {
  const promises = urls.map(url => fetch(url).then(r => r.json()));
  const results = await Promise.all(promises);
  console.log("Done:", results); // ✅ hammasi tugaganda
}
```

**Nima uchun:** `Array.prototype.forEach` return value'ni e'tiborsiz qoldiradi — spec'da `callbackFn` ni sync chaqiradi. `async function` Promise qaytaradi, lekin `forEach` bu Promise'ni ko'rmaydi va `await` qilmaydi. Har async callback darhol chaqiriladi va Promise qaytaradi, forEach esa navbatdagi item'ga o'tadi — parallel execution.

**Qoida:** Async iteration uchun:
- **Serial** (keyingi oldingisini kutadi) → `for...of` + `await`
- **Parallel** (hammasi birga) → `Promise.all(array.map(...))`
- **forEach'da async ishlatmang** — har doim noto'g'ri behavior beradi

---

### `queueMicrotask` vs `Promise.resolve().then()` — xato boshqaruvi farqi

Ikkalasi ham microtask queue'ga callback qo'yadi va **bir xil priority**'da ishlaydi. Lekin **error handling** farqli: `queueMicrotask` ichidagi xato **uncaught exception** bo'lib chiqadi (global error handler orqali), `Promise.then` ichidagi xato esa **rejected promise** — `.catch()` bilan ushlash mumkin.

```javascript
// Promise.then — error caught:
Promise.resolve()
  .then(() => {
    throw new Error("promise error");
  })
  .catch(err => {
    console.log("Caught:", err.message); // ✅ "Caught: promise error"
  });

// queueMicrotask — error uncaught:
queueMicrotask(() => {
  throw new Error("microtask error");
  // ❌ "Uncaught (in microtask) Error: microtask error"
  // .catch() bilan ushlab bo'lmaydi — bu Promise emas
});

// Global error handler bilan ushlash mumkin:
window.addEventListener("error", (event) => {
  console.log("Global error:", event.error.message);
  // "Global error: microtask error"
});
```

**Execution order (ikkalasi bir xil queue):**

```javascript
console.log("1");

Promise.resolve().then(() => console.log("2 — Promise"));
queueMicrotask(() => console.log("3 — queueMicrotask"));
Promise.resolve().then(() => console.log("4 — Promise"));

console.log("5");

// Output:
// 1
// 5
// 2 — Promise
// 3 — queueMicrotask
// 4 — Promise
// FIFO order — qaysi biri avval queue'ga tushsa, o'sha avval
```

**Nima uchun:** `queueMicrotask` — HTML spec'ning low-level API'si, Promise semantikasi yo'q. Shu sababli error handling ham yo'q — xato darhol uncaught bo'lib chiqadi. `Promise.then` esa Promise state machine'ining bir qismi — xato rejected promise yaratadi, u esa chain'ning keyingi `.catch()`'igacha propagate qilinadi.

**Qachon qaysi birini tanlash:**
- **`queueMicrotask`** — yengil va oddiy microtask yaratish (error'lar bo'lmaydi yoki global handler bilan ushlash yetarli)
- **`Promise.resolve().then`** — error handling kerak yoki Promise chain bilan ishlash kerak bo'lsa

---

## Common Mistakes

### ❌ Xato 1: setTimeout(fn, 0) "darhol" ishlaydi deb o'ylash

```javascript
console.log("oldin");
setTimeout(() => console.log("setTimeout"), 0);
console.log("keyin");

// Ko'pchilik kutadi: oldin, setTimeout, keyin
// Aslida: oldin, keyin, setTimeout  ← ENG OXIRDA!
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
setTimeout(() => console.log("timeout"), 0);
Promise.resolve().then(() => console.log("promise"));

// Ko'pchilik kutadi: timeout, promise
// Aslida: promise, timeout
```

### ✅ To'g'ri usul:

```javascript
// Tartibni eslab qoling: Sync → Microtask → Macrotask

setTimeout(() => console.log("3 — macrotask"), 0);
Promise.resolve().then(() => console.log("2 — microtask"));
console.log("1 — sync");

// Output: 1, 2, 3  — DOIM shu tartibda!
```

**Nima uchun:** Microtask Queue **har doim** Macrotask Queue dan oldin bo'shatiladi. Bu Event Loop algoritmining asosiy qoidasi.

---

### ❌ Xato 4: Og'ir hisoblashni main thread'da qilish

```javascript
// ❌ UI bloklaydigan kod
button.addEventListener("click", () => {
  const sorted = hugeArray.sort((a, b) => complexCompare(a, b));
  renderTable(sorted);
  // UI 3-5 soniya muzlaydi
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ Web Worker bilan
const sortWorker = new Worker("sort-worker.js");

button.addEventListener("click", () => {
  showSpinner();
  sortWorker.postMessage(hugeArray);

  sortWorker.onmessage = (e) => {
    hideSpinner();
    renderTable(e.data);
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
async function example() {
  console.log("A");
  await fetch("/api/data");
  console.log("B");
}

example();
console.log("C");

// Ko'pchilik kutadi: A, B, C
// Aslida: A, C, B
```

### ✅ To'g'ri usul:

```javascript
async function example() {
  console.log("A");     // sync — darhol
  await Promise.resolve(); // Shu yerdan keyin — microtask
  console.log("B");     // ← microtask callback ichida
}

example();
console.log("C");       // ← Bu DARHOL ishlaydi!

// Output: A, C, B

// await — funksiya ichini "to'xtatadi", lekin TASHQI kodni to'xtatmaydi
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
7. `setTimeout(7)` → Macrotask Queue
8. `Promise.then(8)` → Microtask Queue: `[second_resume, then_8]`
9. **Microtask checkpoint:**
   - `second_resume` → `"4"` chiqadi. `second()` tugadi → `first` resume: `[then_8, first_resume]`
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
  Macrotask Q:   [timeout_callback]
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
  Microtask Q:   [promise_in_timeout]
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
      queueMicrotask(next); // Microtask — starvation!
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
      setTimeout(processBatch, 0); // macrotask — UI refresh imkoniyati beradi
    }
  }

  processBatch();
}

processItems(Array.from({ length: 100000 }, (_, i) => `item-${i}`));
```

**Tushuntirish:**
1. **Muammo:** `queueMicrotask` recursive chaqirilganda, microtask queue **hech qachon** bo'sh bo'lmaydi → browser muzlaydi.
2. **Yechim:** `setTimeout(fn, 0)` macrotask yaratadi. Har bir batch orasida Event Loop boshqa task'larni bajara oladi.
3. **Batch processing:** 1000 ta itemni sync bajaramiz, keyin setTimeout bilan keyingi batch'ga o'tamiz.

</details>

---

### Mashq 5: Node.js Event Loop (Qiyin)

**Savol:** Quyidagi Node.js kodining output tartibini aniqlang.

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

**Asosiy qoida:** I/O callback ichida `setImmediate` DOIM `setTimeout(fn, 0)` dan oldin ishlaydi.

</details>

---

## Xulosa

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

8. **requestIdleCallback** — browser bo'sh vaqtida past priority'li ishlarni bajarish uchun.

9. **Node.js** Event Loop 6 fazadan iborat, `process.nextTick` eng yuqori priority, `setImmediate` check fazada ishlaydi.

10. **Starvation** — recursive microtask'lar browser'ni muzlatadi. `setTimeout` yoki `setImmediate` bilan oldini oling.

---

> **Keyingi bo'lim:** [12-promises.md](12-promises.md) — Promises: state machine (pending → fulfilled | rejected), Promise constructor va executor semantikasi, `.then()`/`.catch()`/`.finally()` chaining va error propagation, static methods (`Promise.all`, `Promise.race`, `Promise.allSettled`, `Promise.any`, `Promise.withResolvers` ES2024), unhandled rejection detection, Promise internals va microtask queue bilan aloqasi (V8 `PromiseReactionJob`).
