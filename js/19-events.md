# Bo'lim 19: Events

> DOM Events — brauzerda foydalanuvchi harakatlari va tizim hodisalarini boshqarish mexanizmi.

---

## Mundarija

- [Event Model](#event-model)
- [Event Bubbling](#event-bubbling)
- [Event Capturing](#event-capturing)
- [Event Flow: Capturing → Target → Bubbling](#event-flow-capturing--target--bubbling)
- [addEventListener](#addeventlistener)
- [stopPropagation va stopImmediatePropagation](#stoppropagation-va-stopimmediatepropagation)
- [preventDefault](#preventdefault)
- [Event Delegation](#event-delegation)
- [event.target vs event.currentTarget](#eventtarget-vs-eventcurrenttarget)
- [Custom Events](#custom-events)
- [Event Listeners va Memory](#event-listeners-va-memory)
- [Passive Event Listeners](#passive-event-listeners)
- [Touch va Pointer Events](#touch-va-pointer-events)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Event Model

### Nazariya

Event (hodisa) — brauzerda sodir bo'ladigan har qanday narsa: click, klaviatura bosish, scroll, form submit, sahifa yuklash. JavaScript bu hodisalarni ushlash va javob berish imkonini beradi. Event bilan ishlashning 3 ta usuli bor.

### 1. HTML Attribute (eski, tavsiya qilinmaydi)

```html
<!-- HTML va JS aralashib ketadi — Separation of Concerns buziladi -->
<button onclick="alert('Bosildi!')">Bos</button>
<button onclick="handleClick(event)">Bos</button>
```

### 2. DOM Property (cheklangan)

```javascript
const btn = document.getElementById("btn");

btn.onclick = function(event) {
  console.log("Click 1");
};

btn.onclick = function(event) {
  console.log("Click 2");
};
// Faqat "Click 2" — oldingi OVERWRITE bo'ldi!
```

### 3. addEventListener (zamonaviy, to'g'ri)

```javascript
const btn = document.getElementById("btn");

btn.addEventListener("click", function(event) {
  console.log("Click 1");
});

btn.addEventListener("click", function(event) {
  console.log("Click 2");
});
// Ikkisi HAM ishlaydi: "Click 1", "Click 2"
```

| Usul | Ko'p handler | Remove | Options | Tavsiya |
|------|-------------|--------|---------|---------|
| HTML attribute | Yo'q | Yo'q | Yo'q | Yo'q |
| DOM property | Yo'q | Overwrite | Yo'q | Cheklangan |
| `addEventListener` | Ha | Ha | Ha | Ha |

### Under the Hood

Brauzer ichida har bir element `EventTarget` interface'ini implement qiladi. `addEventListener` chaqirilganda brauzer ichki listener ro'yxatiga `{ type, callback, capture, once, passive, signal }` yozuvi qo'shiladi. Event dispatch qilinganda brauzer bu ro'yxatni ko'rib chiqib, mos handler'larni chaqiradi.

---

## Event Bubbling

### Nazariya

Event Bubbling — hodisa **ichki** elementdan **tashqi** elementlarga qarab ko'tarilishi. Bu DOM ning default xulq-atvori — deyarli barcha event'lar bubble qiladi.

### Kod Misollari

```html
<div id="outer">
  <div id="middle">
    <button id="inner">Bos</button>
  </div>
</div>
```

```javascript
document.getElementById("outer").addEventListener("click", () => {
  console.log("1: OUTER");
});

document.getElementById("middle").addEventListener("click", () => {
  console.log("2: MIDDLE");
});

document.getElementById("inner").addEventListener("click", () => {
  console.log("3: INNER");
});

// Button bosilsa:
// "3: INNER"   ← target
// "2: MIDDLE"  ← yuqoriga ko'tarildi
// "1: OUTER"   ← yana yuqoriga
```

### ASCII Diagram

```
                    BUBBLING (ichdan tashqariga)

  ┌─────────────────────────────────────┐
  │ document                             │
  │  ┌──────────────────────────────┐   │
  │  │ #outer                    ③  │   │  ← 3-chi
  │  │  ┌───────────────────────┐   │   │
  │  │  │ #middle            ②  │   │   │  ← 2-chi
  │  │  │  ┌────────────────┐   │   │   │
  │  │  │  │ button      ①  │   │   │   │  ← 1-chi (target)
  │  │  │  │   [CLICK!]     │   │   │   │
  │  │  │  └────────────────┘   │   │   │
  │  │  └───────────────────────┘   │   │
  │  └──────────────────────────────┘   │
  └─────────────────────────────────────┘
```

### Under the Hood

Deyarli barcha event'lar bubble qiladi. Lekin ba'zilari **bubble qilmaydi**:

| Event | Bubbles? | Bubble versiyasi |
|-------|----------|------------------|
| `click`, `mousedown`, `mouseup` | Ha | — |
| `keydown`, `keyup` | Ha | — |
| `input`, `change`, `submit` | Ha | — |
| `focus`, `blur` | Yo'q | `focusin`, `focusout` |
| `mouseenter`, `mouseleave` | Yo'q | `mouseover`, `mouseout` |
| `load`, `error` | Yo'q | — |
| `scroll` | Ha | — |

---

## Event Capturing

### Nazariya

Event Capturing — hodisa **tashqi** elementdan **ichki** elementga qarab tushishi. Bubbling ning teskarisi. `addEventListener` ning uchinchi argumenti `true` yoki `{ capture: true }` qilib yoqiladi.

### Kod Misollari

```javascript
document.getElementById("outer").addEventListener("click", () => {
  console.log("OUTER (capture)");
}, true);

document.getElementById("middle").addEventListener("click", () => {
  console.log("MIDDLE (capture)");
}, { capture: true });

document.getElementById("inner").addEventListener("click", () => {
  console.log("INNER (target)");
});

// Button bosilsa:
// "OUTER (capture)"   ← tashqaridan (capturing phase)
// "MIDDLE (capture)"  ← ichkariga qarab
// "INNER (target)"    ← target element
```

---

## Event Flow: Capturing → Target → Bubbling

### Nazariya

Har bir DOM event **3 ta fazadan** o'tadi:
1. **Capturing Phase** — document dan target gacha (tashqaridan ichga)
2. **Target Phase** — target element o'zida
3. **Bubbling Phase** — target dan document gacha (ichdan tashqiga)

Target element'da capturing va bubbling handler'larning ishlash tartibi — **yozilgan tartibda** bajariladi (W3C DOM Level 2 Events spec).

### ASCII Diagram

```
         ┌────── CAPTURING ──────┐    ┌───── BUBBLING ─────┐
         │  (tashqaridan ichga)  │    │  (ichdan tashqariga) │
         ▼                       │    │                      ▲
  ┌──────┴──────────────────────┐│    │┌─────────────────────┴──┐
  │ document               ①  ││    ││                     ⑦ │
  │ ┌───────────────────────┐  ││    ││ ┌───────────────────┐  │
  │ │ <html>            ②  │  ││    ││ │               ⑥  │  │
  │ │ ┌─────────────────┐  │  ││    ││ │ ┌─────────────┐  │  │
  │ │ │ <body>       ③  │  │  ││    ││ │ │          ⑤  │  │  │
  │ │ │ ┌───────────┐  │  │  ││    ││ │ │ ┌────────┐  │  │  │
  │ │ │ │ <button>  │  │  │  │└────┘│ │ │ │        │  │  │  │
  │ │ │ │    ④      │  │  │  │      │ │ │ │        │  │  │  │
  │ │ │ │  TARGET   │  │  │  │      │ │ │ │        │  │  │  │
  │ │ │ └───────────┘  │  │  │      │ │ │ └────────┘  │  │  │
  │ │ └─────────────────┘  │  │      │ │ └─────────────┘  │  │
  │ └───────────────────────┘  │      │ └───────────────────┘  │
  └────────────────────────────┘      └────────────────────────┘
```

### To'liq Misol

```javascript
const outer = document.getElementById("outer");
const middle = document.getElementById("middle");
const inner = document.getElementById("inner");

// Capturing
outer.addEventListener("click", () => console.log("outer CAPTURE"), true);
middle.addEventListener("click", () => console.log("middle CAPTURE"), true);
inner.addEventListener("click", () => console.log("inner CAPTURE"), true);

// Bubbling
outer.addEventListener("click", () => console.log("outer BUBBLE"));
middle.addEventListener("click", () => console.log("middle BUBBLE"));
inner.addEventListener("click", () => console.log("inner BUBBLE"));

// inner button bosilsa:
// 1. "outer CAPTURE"   ← Capturing phase
// 2. "middle CAPTURE"  ← Capturing phase
// 3. "inner CAPTURE"   ← Target phase (yozilgan tartibda)
// 4. "inner BUBBLE"    ← Target phase (yozilgan tartibda)
// 5. "middle BUBBLE"   ← Bubbling phase
// 6. "outer BUBBLE"    ← Bubbling phase
```

---

## addEventListener

### Nazariya

`addEventListener` — event handler qo'shishning zamonaviy usuli. 3 ta argument: event type, handler, options.

### Sintaksis

```javascript
element.addEventListener(eventType, handler, options);

// options — boolean yoki object
// Boolean: true = capture, false = bubble (default)
// Object:
{
  capture: false,    // Capturing phase da ushlasinmi?
  once: true,        // Bir marta ishlab, avtomatik o'chirilsin
  passive: true,     // preventDefault() chaqirmasligini kafolatlash
  signal: controller.signal  // AbortController bilan o'chirish
}
```

### Kod Misollari

```javascript
// once — faqat bir marta
button.addEventListener("click", () => {
  console.log("Faqat bir marta!");
}, { once: true });

// AbortController bilan ko'p listener ni boshqarish
const controller = new AbortController();

button.addEventListener("click", () => {
  console.log("Click!");
}, { signal: controller.signal });

window.addEventListener("keydown", (e) => {
  console.log("Key:", e.key);
}, { signal: controller.signal });

// Bir marta abort() — IKKISI HAM o'chiriladi
controller.abort();
```

### removeEventListener

```javascript
// AYNI funksiya reference kerak!

// Ishlamaydi — boshqa funksiya
button.addEventListener("click", () => console.log("A"));
button.removeEventListener("click", () => console.log("A"));
// Bu BOSHQA funksiya — reference teng emas!

// Ishlaydi
function handler() { console.log("A"); }
button.addEventListener("click", handler);
button.removeEventListener("click", handler);

// capture/bubble mos kelishi kerak
button.addEventListener("click", handler, true);
button.removeEventListener("click", handler, true);  // capture true
button.removeEventListener("click", handler);         // ishlamaydi — default false
```

---

## stopPropagation va stopImmediatePropagation

### Nazariya

`stopPropagation()` — hodisa boshqa elementlarga **tarqalishini** to'xtatadi, lekin shu elementdagi boshqa handler'lar bajariladi. `stopImmediatePropagation()` — tarqalishni to'xtatadi **VA** shu elementdagi qolgan handler'lar ham to'xtatiladi.

### Kod Misollari

```javascript
// === stopPropagation ===
middle.addEventListener("click", (e) => {
  e.stopPropagation();
  console.log("MIDDLE — propagation to'xtatildi");
});

// inner bosilsa:
// "INNER"
// "MIDDLE — propagation to'xtatildi"
// "OUTER" — BAJARILMAYDI

// === stopImmediatePropagation ===
button.addEventListener("click", (e) => {
  console.log("Handler 1");
  e.stopImmediatePropagation();
});

button.addEventListener("click", (e) => {
  console.log("Handler 2"); // BAJARILMAYDI!
});

// Faqat "Handler 1"
```

### Farqi

```
stopPropagation:
  Button: [Handler1 ✅] [Handler2 ✅] — shu elementdagi barcha handler ishlaydi
  Parent: [Handler3 ❌]              — ota elementlarga o'tmaydi

stopImmediatePropagation:
  Button: [Handler1 ✅] [Handler2 ❌] — shu elementda HAM to'xtaydi
  Parent: [Handler3 ❌]              — ota elementlarga ham o'tmaydi
```

---

## preventDefault

### Nazariya

`preventDefault()` — brauzerning **standart xulq-atvorini** bekor qiladi (link navigatsiya, form submit, checkbox toggle, kontekst menyu). Muhim: u propagation'ni **TO'XTATMAYDI** — hodisa bubble qilishda davom etadi.

### Kod Misollari

```javascript
// Link navigatsiyani to'xtatish
link.addEventListener("click", (e) => {
  e.preventDefault();
  console.log("Navigatsiya yo'q");
});

// Form submit ni to'xtatish
form.addEventListener("submit", (e) => {
  e.preventDefault(); // sahifa reload bo'lmaydi
  const data = new FormData(form);
  console.log(Object.fromEntries(data));
});

// Kontekst menyu
document.addEventListener("contextmenu", (e) => {
  e.preventDefault(); // o'ng click menyu chiqmaydi
});
```

### preventDefault vs stopPropagation vs return false

```javascript
// preventDefault — default behavior to'xtatadi
// stopPropagation — propagation to'xtatadi
// Ular ALOHIDA narsa!

link.addEventListener("click", (e) => {
  e.preventDefault();    // navigatsiya yo'q
  e.stopPropagation();   // bubbling yo'q
});

// addEventListener da return false HECH NIMA QILMAYDI!
button.addEventListener("click", () => {
  return false; // hech narsa to'xtatmaydi
});

// Faqat onclick PROPERTY da ishlaydi
button.onclick = () => { return false; }; // preventDefault + stopPropagation
```

---

## Event Delegation

### Nazariya

Event Delegation — har bir elementga alohida handler qo'shish o'rniga, **ota elementga bitta** handler qo'yish va `event.target`/`closest()` orqali qaysi element bosilganini aniqlash. Bubbling asosida ishlaydi. Afzalliklari: memory tejash, dinamik elementlar uchun ishlaydi, kod sodda.

### Kod Misollari

```javascript
// 1000 ta handler o'rniga — bitta
document.getElementById("list").addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (!item) return;
  console.log("Item bosildi:", item.textContent);
});
// Yangi elementlar ham avtomatik ishlaydi!
```

### Amaliy Misol: Tab Component

```html
<div class="tabs">
  <button class="tab" data-tab="home">Home</button>
  <button class="tab" data-tab="about">About</button>
  <button class="tab" data-tab="contact">Contact</button>
</div>
<div class="tab-content"></div>
```

```javascript
const tabs = document.querySelector(".tabs");
const content = document.querySelector(".tab-content");

tabs.addEventListener("click", (e) => {
  const tab = e.target.closest(".tab");
  if (!tab) return;

  tabs.querySelectorAll(".tab").forEach(t => t.classList.remove("active"));
  tab.classList.add("active");

  content.textContent = `${tab.dataset.tab} sahifasi`;
});
```

### Under the Hood

```
1. Button bosildi
2. Event TARGET = button
3. Bubbling: button → li → ul → body → html → document
4. ul dagi handler ishlaydi
5. e.target = button (bosilgan element)
6. e.currentTarget = ul (handler qo'yilgan element)
7. closest(".item") bilan kerakli elementni topamiz
```

---

## event.target vs event.currentTarget

### Nazariya

`event.target` — hodisa **sodir bo'lgan** element (bosilgan), propagation davomida **o'zgarmaydi**. `event.currentTarget` — handler **qo'yilgan** element, har bosqichda o'zgaradi.

### Kod Misollari

```html
<div id="parent">
  <button id="child">Click me</button>
</div>
```

```javascript
document.getElementById("parent").addEventListener("click", (e) => {
  console.log("target:", e.target.id);        // "child"
  console.log("currentTarget:", e.currentTarget.id); // "parent"
});
```

```javascript
// Oddiy funksiyada this === currentTarget
parent.addEventListener("click", function(e) {
  console.log(this === e.currentTarget); // true
});

// Arrow function da this boshqa (outer scope)
parent.addEventListener("click", (e) => {
  console.log(this === e.currentTarget); // false
});
```

---

## Custom Events

### Nazariya

JavaScript orqali o'z hodisalarini yaratish va `dispatchEvent()` bilan yuborish mumkin. `CustomEvent` constructor'i `detail` property orqali data uzatadi. `bubbles: true` qo'yilsa bubble qiladi, `cancelable: true` qo'yilsa `preventDefault()` bilan bekor qilish mumkin.

### Kod Misollari

```javascript
// CustomEvent yaratish
const event = new CustomEvent("user-login", {
  detail: {
    userId: 42,
    username: "Ali",
    timestamp: Date.now()
  },
  bubbles: true,
  cancelable: true
});

// Tinglash
document.addEventListener("user-login", (e) => {
  console.log("Login:", e.detail);
});

// Dispatch
document.dispatchEvent(event);
```

### Event Bus Pattern

```javascript
class EventBus {
  constructor() {
    this.target = new EventTarget();
  }

  on(event, callback) {
    this.target.addEventListener(event, (e) => callback(e.detail));
  }

  off(event, callback) {
    this.target.removeEventListener(event, callback);
  }

  emit(event, data) {
    this.target.dispatchEvent(new CustomEvent(event, { detail: data }));
  }
}

const bus = new EventBus();

// Component A
bus.on("cart-update", (data) => {
  console.log("Cart:", data.items.length, "ta buyum");
});

// Component B
bus.emit("cart-update", { items: ["telefon", "quloqchin"] });
// "Cart: 2 ta buyum"
```

---

## Event Listeners va Memory

### Nazariya

Event listener'lar xotirada joy egallaydi. SPA (Single Page Application) larda sahifa qayta yuklanmagani uchun to'g'ri tozalanmasa memory leak bo'ladi. Element DOM dan o'chirilsa, unga qo'yilgan listener'lar element bilan yo'qoladi, **LEKIN** global listener'lar (window, document) alohida o'chirilishi kerak.

### Memory Leak Holatlari

```javascript
// Element o'chirilsa ham, global listener qoladi
function setupWidget(container) {
  const bigData = new Array(100000).fill("data");

  const handler = () => {
    console.log(bigData.length); // closure → bigData reference
  };

  window.addEventListener("scroll", handler);

  container.remove();
  // window scroll listener HALI QOLDI!
  // bigData ham GC tozalay olmaydi
}

// To'g'ri — cleanup funksiya
function setupWidget(container) {
  const bigData = new Array(100000).fill("data");

  const handler = () => console.log(bigData.length);
  window.addEventListener("scroll", handler);

  return function cleanup() {
    window.removeEventListener("scroll", handler);
  };
}

const cleanup = setupWidget(container);
cleanup(); // listener o'chirildi
```

### AbortController bilan Cleanup

```javascript
function setupComponent() {
  const controller = new AbortController();

  window.addEventListener("scroll", handleScroll, { signal: controller.signal });
  window.addEventListener("resize", handleResize, { signal: controller.signal });
  document.addEventListener("click", handleClick, { signal: controller.signal });

  return function cleanup() {
    controller.abort(); // BARCHA listener'lar bir marta bilan o'chiriladi
  };
}
```

### Element Remove va Listener'lar

```javascript
const element = document.getElementById("widget");
element.addEventListener("click", handler);
element.remove();
// Element dagi listener'lar element bilan birga yo'qoladi
// LEKIN: boshqa joyda element ga reference saqlansa — GC tozalamaydi

// Global listener'lar (window, document) — DOM remove bilan O'CHMAYDI
```

---

## Passive Event Listeners

### Nazariya

`passive: true` brauzerga kafolat beradi: "Bu handler `preventDefault()` chaqirmaydi." Scroll va touch event'lar uchun performance'ni sezilarli oshiradi — brauzer handler tugashini kutmasdan scroll'ni darhol boshlaydi.

### Kod Misollari

```javascript
// Brauzer kutishi kerak — sekin scroll
document.addEventListener("scroll", (e) => {
  // Og'ir hisoblash...
  // Brauzer: "preventDefault chaqiradimi? Kutaman..."
});

// passive: true — brauzer kutmaydi — tez scroll
document.addEventListener("scroll", (e) => {
  // Og'ir hisoblash...
  // Brauzer: "passive, scroll davom!"
  // e.preventDefault(); // Warning — passive da ishlamaydi!
}, { passive: true });
```

### Under the Hood

```
                     Passive: false (default)
  ┌──────────┐     ┌───────────────┐     ┌────────────┐
  │ Touch/   │────▶│ Handler       │────▶│ Scroll     │
  │ Scroll   │     │ (kutish)      │     │ (keyinroq) │
  └──────────┘     └───────────────┘     └────────────┘

                     Passive: true
  ┌──────────┐     ┌───────────────┐
  │ Touch/   │──┬─▶│ Handler       │     (parallel)
  │ Scroll   │  │  └───────────────┘
  └──────────┘  │  ┌────────────┐
                └─▶│ Scroll     │     (darhol!)
                   └────────────┘
```

Chrome 51+ dan boshlab `document`/`window` dagi `touchstart`, `touchmove`, `wheel` event'lari **default passive**.

---

## Touch va Pointer Events

### Nazariya

Mobile qurilmalarda `click` event 300ms kechikadi (double-tap zoom tekshirish uchun). Touch va Pointer event'lar bevosita sensorli ekran bilan ishlash uchun mo'ljallangan. Pointer Events — touch, mouse va pen ni **birlashtirgan** universal API.

### Touch Events

```javascript
const el = document.getElementById("swipeable");

el.addEventListener("touchstart", (e) => {
  const touch = e.touches[0]; // birinchi barmoq
  console.log("Start:", touch.clientX, touch.clientY);
  // e.touches     — hozir ekranda barcha barmoqlar
  // e.targetTouches — shu elementdagi barmoqlar
  // e.changedTouches — o'zgargan barmoqlar
});

el.addEventListener("touchmove", (e) => {
  const touch = e.touches[0];
  console.log("Move:", touch.clientX, touch.clientY);
}, { passive: true }); // scroll uchun passive

el.addEventListener("touchend", (e) => {
  const touch = e.changedTouches[0]; // touches bo'sh — changedTouches ishlatamiz
  console.log("End:", touch.clientX, touch.clientY);
});
```

### Pointer Events (Universal API)

```javascript
// Pointer Events — touch, mouse, pen barchasini birlashtiradi
const canvas = document.getElementById("canvas");

canvas.addEventListener("pointerdown", (e) => {
  console.log("Type:", e.pointerType); // "mouse", "touch", "pen"
  console.log("Pressure:", e.pressure); // 0..1 — pen/touch bosim
  console.log("Width:", e.width);       // barmoq/pen o'lchami
  console.log("isPrimary:", e.isPrimary); // asosiy pointer
  canvas.setPointerCapture(e.pointerId); // pointer capture
});

canvas.addEventListener("pointermove", (e) => {
  if (e.pressure > 0) { // faqat bosilganda
    draw(e.clientX, e.clientY, e.pressure);
  }
});

canvas.addEventListener("pointerup", (e) => {
  canvas.releasePointerCapture(e.pointerId);
});

// pointercancel — tizim tomonidan bekor qilinish (masalan, brauzer scroll qilsa)
canvas.addEventListener("pointercancel", (e) => {
  console.log("Pointer cancelled");
});
```

### Touch → Pointer moslik

| Touch Event | Pointer Event | Mouse Event |
|-------------|---------------|-------------|
| `touchstart` | `pointerdown` | `mousedown` |
| `touchmove` | `pointermove` | `mousemove` |
| `touchend` | `pointerup` | `mouseup` |
| `touchcancel` | `pointercancel` | — |

### Amaliy Misol: Swipe Detection

```javascript
function addSwipeDetection(element, callback) {
  let startX, startY;
  const THRESHOLD = 50; // minimum 50px

  element.addEventListener("pointerdown", (e) => {
    startX = e.clientX;
    startY = e.clientY;
    element.setPointerCapture(e.pointerId);
  });

  element.addEventListener("pointerup", (e) => {
    const diffX = e.clientX - startX;
    const diffY = e.clientY - startY;

    if (Math.abs(diffX) > Math.abs(diffY) && Math.abs(diffX) > THRESHOLD) {
      callback(diffX > 0 ? "right" : "left");
    } else if (Math.abs(diffY) > THRESHOLD) {
      callback(diffY > 0 ? "down" : "up");
    }
  });
}

addSwipeDetection(document.getElementById("slider"), (direction) => {
  console.log("Swipe:", direction);
});
```

### Under the Hood

Brauzer event tartibi touch qurilmalarda: `pointerdown → touchstart → pointerup → touchend → mousedown → mouseup → click`. Click 300ms kechikadi — `touch-action: manipulation` CSS bilan yoki `pointerdown` listener bilan oldini olish mumkin. Zamonaviy brauzerlar viewport `width=device-width` qo'yilganda 300ms kechikishni avtomatik o'chiradi.

---

## Common Mistakes

### Xato 1: removeEventListener da boshqa funksiya

```javascript
// Ishlamaydi — anonymous function
button.addEventListener("click", () => console.log("A"));
button.removeEventListener("click", () => console.log("A"));
// BOSHQA funksiya — reference teng emas

// To'g'ri:
const handler = () => console.log("A");
button.addEventListener("click", handler);
button.removeEventListener("click", handler);

// Yoki AbortController
const controller = new AbortController();
button.addEventListener("click", () => console.log("A"), { signal: controller.signal });
controller.abort();
```

---

### Xato 2: Event delegation da target ni noto'g'ri tekshirish

```javascript
// Noto'g'ri
list.addEventListener("click", (e) => {
  if (e.target.classList.contains("item")) { /* ... */ }
});
// <li class="item"><span>Text</span></li>
// span bosilsa → target = span, li emas!

// To'g'ri — closest()
list.addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (item) console.log(item.textContent);
});
```

---

### Xato 3: preventDefault va stopPropagation ni aralashtirish

```javascript
// Noto'g'ri
form.addEventListener("submit", (e) => {
  e.stopPropagation(); // Bu propagation to'xtatadi
  // Form hali ham submit bo'ladi!
});

// To'g'ri
form.addEventListener("submit", (e) => {
  e.preventDefault(); // Default behavior to'xtatadi
});
```

---

### Xato 4: var bilan loop ichida handler

```javascript
// Noto'g'ri — barcha buttonlar oxirgi qiymat
for (var i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => {
    console.log("Button:", i); // hammasi bir xil
  });
}

// To'g'ri — let yoki forEach
for (let i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => {
    console.log("Button:", i);
  });
}
```

---

### Xato 5: SPA da global listener tozalanmaydi

```javascript
// Noto'g'ri — har navigatsiyada yangi listener qo'shiladi
function initPage() {
  window.addEventListener("scroll", handleScroll); // leak!
}

// To'g'ri — AbortController
let pageController;
function initPage() {
  if (pageController) pageController.abort(); // oldingini o'chirish
  pageController = new AbortController();
  window.addEventListener("scroll", handleScroll, {
    signal: pageController.signal
  });
}
```

---

## Amaliy Mashqlar

### Mashq 1: Event Flow Tartibi (Oson)

Quyidagi kodda button bosilganda tartibni yozing:

```javascript
document.addEventListener("click", () => console.log("1: document BUBBLE"));
document.addEventListener("click", () => console.log("2: document CAPTURE"), true);
document.body.addEventListener("click", () => console.log("3: body BUBBLE"));
document.body.addEventListener("click", () => console.log("4: body CAPTURE"), true);
button.addEventListener("click", () => console.log("5: button BUBBLE"));
button.addEventListener("click", () => console.log("6: button CAPTURE"), true);
```

<details>
<summary>Javob</summary>

```
2: document CAPTURE
4: body CAPTURE
6: button CAPTURE
5: button BUBBLE
3: body BUBBLE
1: document BUBBLE
```

Capturing (tashqaridan): document → body. Target (yozilgan tartibda): button CAPTURE → BUBBLE. Bubbling (ichdan): body → document.

</details>

---

### Mashq 2: Event Delegation bilan Dropdown (O'rta)

`.menu` ga bitta handler bilan: bosilgan `.menu-item` ga `active` class, boshqalardan olib tashlang.

<details>
<summary>Javob</summary>

```javascript
const menu = document.querySelector(".menu");

menu.addEventListener("click", (e) => {
  const item = e.target.closest(".menu-item");
  if (!item || !menu.contains(item)) return;

  menu.querySelectorAll(".menu-item.active").forEach(el => {
    el.classList.remove("active");
  });
  item.classList.add("active");
});
```

</details>

---

### Mashq 3: EventEmitter Class (O'rta)

`on`, `off`, `emit`, `once` metodlari bilan EventEmitter yarating.

<details>
<summary>Javob</summary>

```javascript
class EventEmitter {
  constructor() {
    this.listeners = new Map();
  }

  on(event, callback) {
    if (!this.listeners.has(event)) this.listeners.set(event, []);
    this.listeners.get(event).push(callback);
    return this;
  }

  off(event, callback) {
    const handlers = this.listeners.get(event);
    if (!handlers) return this;
    const index = handlers.indexOf(callback);
    if (index !== -1) handlers.splice(index, 1);
    return this;
  }

  emit(event, ...args) {
    const handlers = this.listeners.get(event);
    if (!handlers) return false;
    handlers.forEach(h => h(...args));
    return true;
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    return this.on(event, wrapper);
  }
}
```

</details>

---

### Mashq 4: Debounced Search (Qiyin)

Input yozilganda 300ms kutib search chaqirilsin. AbortController bilan oldingi request bekor qilinsin.

<details>
<summary>Javob</summary>

```javascript
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

let currentController = null;

const performSearch = debounce(async (query) => {
  if (currentController) currentController.abort();
  currentController = new AbortController();

  if (!query.trim()) return;

  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal: currentController.signal
    });
    const data = await res.json();
    showResults(data);
  } catch (e) {
    if (e.name !== "AbortError") console.error(e);
  }
}, 300);

searchInput.addEventListener("input", (e) => {
  performSearch(e.target.value);
});
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Event Model** | `addEventListener` — zamonaviy, ko'p handler, options |
| **Bubbling** | Ichdan tashqariga (default) — deyarli barcha eventlar |
| **Capturing** | Tashqaridan ichga — `{ capture: true }` |
| **Event Flow** | Capturing → Target → Bubbling (3 faza) |
| **stopPropagation** | Boshqa elementlarga tarqalishni to'xtatadi |
| **preventDefault** | Brauzerning standart xulq-atvorini bekor qiladi |
| **Delegation** | Bitta ota handler + `closest()` = samarali |
| **target vs currentTarget** | target = bosilgan, currentTarget = handler qo'yilgan |
| **Custom Events** | `new CustomEvent("nom", { detail })` + `dispatchEvent` |
| **Memory** | `removeEventListener`, `AbortController`, `{ once: true }` |
| **Passive** | `{ passive: true }` — scroll/touch performance |
| **Pointer Events** | Universal API — touch + mouse + pen |

> **Keyingi bo'lim:** [Bo'lim 19.5: Browser APIs](19.5-browser-apis.md) — Fetch API, IntersectionObserver, Web Workers va boshqa brauzer imkoniyatlari.

> **Cross-references:** [18-dom.md](18-dom.md) (DOM Manipulation, querySelector, classList), [05-closures.md](05-closures.md) (closure va event handler), [16-memory.md](16-memory.md) (memory leak, GC — listener cleanup), [13-async-await.md](13-async-await.md) (AbortController), [09-functions.md](09-functions.md) (arrow vs regular — this in handlers)
