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
- [Keyboard Events Deep Dive](#keyboard-events-deep-dive)
- [Composed Events va Shadow DOM](#composed-events-va-shadow-dom)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
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

<details>
<summary><strong>Under the Hood</strong></summary>

Brauzer ichida har bir element `EventTarget` interface'ini implement qiladi. `addEventListener` chaqirilganda brauzer ichki listener ro'yxatiga `{ type, callback, capture, once, passive, signal }` yozuvi qo'shiladi. Event dispatch qilinganda brauzer bu ro'yxatni ko'rib chiqib, mos handler'larni chaqiradi.

</details>

---

## Event Bubbling

### Nazariya

Event Bubbling — hodisa **ichki** elementdan **tashqi** elementlarga qarab ko'tarilishi. Bu DOM ning default xulq-atvori — deyarli barcha event'lar bubble qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

Deyarli barcha event'lar bubble qiladi. Lekin ba'zilari **bubble qilmaydi**:

| Event | Bubbles? | Bubble versiyasi |
|-------|----------|------------------|
| `click`, `mousedown`, `mouseup` | Ha | — |
| `keydown`, `keyup` | Ha | — |
| `input`, `change`, `submit` | Ha | — |
| `focus`, `blur` | Yo'q | `focusin`, `focusout` |
| `mouseenter`, `mouseleave` | Yo'q | `mouseover`, `mouseout` |
| `load`, `error` | Yo'q | — |
| `scroll` | Yo'q* | — |

\* `scroll` event element'da bubble qilmaydi (spec: `bubbles: false`). Lekin `document`'da scroll bo'lganda event `window`'ga ham yetadi — bu "bubbling" emas, HTML spec'ning maxsus propagation'i (target aslida `document`, `window`'da listener ham ishlaydi). `scroll` uchun passive listener ishlatish tavsiya (performance).

</details>

---

## Event Capturing

### Nazariya

Event Capturing — hodisa **tashqi** elementdan **ichki** elementga qarab tushishi. Bubbling ning teskarisi. `addEventListener` ning uchinchi argumenti `true` yoki `{ capture: true }` qilib yoqiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

W3C DOM Level 2 Events specification bo'yicha event dispatch algoritmi capturing phase'ni birinchi bajaradi. Brauzer event target'ning ancestor chain'ini (document dan target gacha) qurib, har bir ancestor'dagi capture listener'larni **yuqoridan pastga** tartibda chaqiradi: `document → html → body → ... → target parent`.

Capture phase listener'lar faqat `{ capture: true }` yoki uchinchi argument `true` bilan ro'yxatdan o'tkazilganlari. Brauzer ichki listener ro'yxatida har bir yozuv `capture` flag saqlaydi — dispatch paytida faqat mos flag'li listener'lar chaqiriladi. Bitta elementda bir xil event type uchun capture va bubble listener'lar alohida ro'yxatda saqlanadi.

Target phase'da esa capture va bubble farqi yo'q — listener'lar **registration tartibida** (qo'shilgan ketma-ketlikda) bajariladi. Bu DOM Level 2 spec ning muhim nuansi — target node'da `capture: true` va `capture: false` listener'larning ishlash tartibi faqat `addEventListener` chaqirilgan ketma-ketlikka bog'liq.

Amalda capturing phase kam ishlatiladi — asosiy use case: event delegation da event'ni boshqa handler'lardan oldin ushlash yoki focus/blur kabi bubble qilmaydigan event'larni capturing orqali ota elementda eshitish.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

`EventTarget` interface'ining ichki implementatsiyasida har bir element **event listener list** saqlaydi. `addEventListener` chaqirilganda yangi yozuv qo'shiladi: `{ type, callback, capture, passive, once, signal }`. Agar ayni `type + callback + capture` kombinatsiyasi allaqachon mavjud bo'lsa, duplicate qo'shilmaydi (spec bo'yicha).

`once: true` qo'yilganda brauzer handler'ni wrapper'ga o'raydi — handler birinchi marta chaqirilgandan keyin avtomatik `removeEventListener` bajariladi. `passive: true` brauzer rendering thread'iga signal beradi — bu listener `preventDefault()` chaqirmasligi kafolatlangan, shuning uchun scroll/touch event'larda compositor thread handler tugashini kutmasdan scrollni boshlaydi.

`signal` option `AbortSignal` qabul qiladi. Signal abort bo'lganda brauzer ichki listener ro'yxatidan bu yozuvni o'chiradi — `removeEventListener` chaqirishning alternativ usuli. Bitta `AbortController` bilan bir nechta listener'ni boshqarish mumkin — `controller.abort()` signal'ga bog'langan barcha listener'larni bir vaqtda o'chiradi.

`removeEventListener` chaqirilganda brauzer listener ro'yxatidan `type + callback + capture` bo'yicha mos yozuvni topib o'chiradi. Callback **reference equality** bilan taqqoslanadi (`===`) — shuning uchun anonymous function'ni remove qilish imkonsiz.

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

Har bir `Event` object'da `cancelable` boolean flag bor. `preventDefault()` faqat `cancelable: true` bo'lgan event'larda ishlaydi — aks holda chaqirish hech narsa qilmaydi. `event.defaultPrevented` property `true` bo'ladi agar `preventDefault()` muvaffaqiyatli chaqirilgan bo'lsa. Brauzer event dispatch tugagandan keyin `defaultPrevented` flag'ni tekshiradi va `true` bo'lsa default action'ni bajarilmaydi.

`passive: true` bilan ro'yxatdan o'tkazilgan listener ichida `preventDefault()` chaqirilsa, brauzer uni **ignore** qiladi va console'ga warning chiqaradi. Bu spec da aniq belgilangan — passive listener `cancelable` event'ni cancel qilish huquqiga ega emas. Chrome 56+ da `document`/`window` dagi `touchstart`, `touchmove`, `wheel` event'lari default passive — shuning uchun bu event'larda `preventDefault()` ishlashi uchun `{ passive: false }` ni aniq ko'rsatish kerak.

Event'ning default action'lari: `click` — link navigatsiya, checkbox toggle; `submit` — form yuborish; `keydown` — matn kiritish; `wheel` — scroll; `contextmenu` — o'ng click menyu; `dragstart` — drag operatsiya. Ba'zi event'lar `cancelable: false` — masalan `scroll`, `load`, `focus`, `blur` — ularning default action'ni bekor qilish mumkin emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

// onclick PROPERTY (yoki HTML attribute) da return false:
// HTML spec bo'yicha — bu faqat MA'LUM event'lar uchun preventDefault() ga teng
// Asosan: <a>/<form> kabi default action'i bor bo'lgan elementlar
button.onclick = () => { return false; };
// - click on <a href="..."> → preventDefault() (navigatsiya yo'q) ✅
// - submit on <form>        → preventDefault() (sahifa reload yo'q) ✅
// - click on oddiy <button> (default action'i yo'q) → hech qanday effekt
// Hech qachon stopPropagation qilmaydi!
```

---

## Event Delegation

### Nazariya

Event Delegation — har bir elementga alohida handler qo'shish o'rniga, **ota elementga bitta** handler qo'yish va `event.target`/`closest()` orqali qaysi element bosilganini aniqlash. Bubbling asosida ishlaydi. Afzalliklari: memory tejash, dinamik elementlar uchun ishlaydi, kod sodda.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// 1000 ta handler o'rniga — bitta
document.getElementById("list").addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (!item) return;
  console.log("Item bosildi:", item.textContent);
});
// Yangi elementlar ham avtomatik ishlaydi!
```

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

```
1. Button bosildi
2. Event TARGET = button
3. Bubbling: button → li → ul → body → html → document
4. ul dagi handler ishlaydi
5. e.target = button (bosilgan element)
6. e.currentTarget = ul (handler qo'yilgan element)
7. closest(".item") bilan kerakli elementni topamiz
```

</details>

---

## event.target vs event.currentTarget

### Nazariya

`event.target` — hodisa **sodir bo'lgan** element (bosilgan), propagation davomida **o'zgarmaydi**. `event.currentTarget` — handler **qo'yilgan** element, har bosqichda o'zgaradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Custom Events

### Nazariya

JavaScript orqali o'z hodisalarini yaratish va `dispatchEvent()` bilan yuborish mumkin. `CustomEvent` constructor'i `detail` property orqali data uzatadi. `bubbles: true` qo'yilsa bubble qiladi, `cancelable: true` qo'yilsa `preventDefault()` bilan bekor qilish mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

### Event Bus Pattern

```javascript
class EventBus {
  constructor() {
    this.target = new EventTarget();
    this.listeners = new Map();
  }

  on(event, callback) {
    const wrapper = (e) => callback(e.detail);
    const key = `${event}::${callback}`;
    this.listeners.set(key, wrapper);
    this.target.addEventListener(event, wrapper);
  }

  off(event, callback) {
    const key = `${event}::${callback}`;
    const wrapper = this.listeners.get(key);
    if (wrapper) {
      this.target.removeEventListener(event, wrapper);
      this.listeners.delete(key);
    }
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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

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

Chrome 56+ dan boshlab `document`/`window` dagi `touchstart`, `touchmove`, `wheel` event'lari **default passive** (Chrome 51 faqat `passive` option'ni kiritdi, 56 da default qilindi).

</details>

---

## Touch va Pointer Events

### Nazariya

Mobile qurilmalarda tarixan `click` event ~300ms kechikishi bor edi — brauzer double-tap zoom'ni aniqlash uchun ikkinchi tap'ni kutishi kerak edi. **Zamonaviy brauzerlarda** (Chrome Android 32+, iOS Safari 9.3+) bu kechikish `<meta name="viewport" content="width=device-width">` yoki `touch-action: manipulation` CSS bilan avtomatik olib tashlangan — ya'ni to'g'ri viewport sozlanishi bor responsive saytlarda 300ms delay yo'q. Eski legacy sahifalar uchun esa muammo davom etadi. Touch va Pointer event'lar bevosita sensorli ekran bilan ishlash uchun mo'ljallangan va click'dan tezroq javob beradi. Pointer Events — touch, mouse va pen ni **birlashtirgan** universal API.

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

<details>
<summary><strong>Under the Hood</strong></summary>

Touch qurilmalarda bitta fizik tap bir nechta event'ni trigger qiladi — odatda: `pointerdown → touchstart → touchend → pointerup → (mousedown/mouseup synthesized) → click`. **Aniq tartib brauzer va platform'ga bog'liq** — Chrome, Safari, va Firefox'da biroz farqli bo'lishi mumkin (ayniqsa pointer va touch interleave tartibi). Mouse event'lari **compatibility layer** sifatida synthesized qilinadi — click listener'lari touch qurilmada ham ishlashini kafolatlash uchun. `touch-action: manipulation` CSS yoki `pointerdown` listener bilan bevosita javob berish orqali default click delay va compatibility mouse event sekvensidan chetlab o'tish mumkin. Pointer capture (`setPointerCapture`) — bir pointer'ni ma'lum elementga "bog'lash" imkonini beradi, drag-drop va gesture'lar uchun muhim.

</details>

---

## Keyboard Events Deep Dive

### Nazariya

Klaviatura event'lari — `keydown`, `keyup`, va tarixiy `keypress` (deprecated). `keydown` va `keyup` klavishaning **fizik** bosilishi va qo'yib yuborilishi, `keypress` esa matn kirituvchi (character-producing) klavishalarni bildiradi. `keypress` deprecated — yangi kodda ishlatilmasin, `keydown` + `beforeinput`/`input` event'larni ishlatish kerak.

Zamonaviy keyboard event'larda ikki muhim property bor: **`event.key`** va **`event.code`**. Ularni aralashtirish tez-tez xatolik manbai.

**`event.key`** — bosilgan klavishning **mantiqiy** (logical) qiymati: foydalanuvchi nima kiritmoqchiligi. Layout'ga bog'liq — QWERTY, AZERTY, Dvorak kabi turli layout'larda bir xil fizik klavisha turli `key` beradi. Modifier klavishalar (Shift) ta'sir qiladi — `a` vs `A`. Non-printable klavishalar esa nom sifatida keladi: `"Enter"`, `"Escape"`, `"ArrowLeft"`, `"Backspace"`, `"F1"`, `"Shift"`, `"Control"`, `"Meta"`, `"Tab"`.

**`event.code`** — bosilgan klavishning **fizik** (physical) pozitsiyasi: qaysi klavisha fizik bosildi, layout'ga bog'liq emas. QWERTY standart bo'yicha nomlanadi: `"KeyA"`, `"KeyS"`, `"Digit1"`, `"Space"`, `"Enter"`, `"ArrowLeft"`, `"ShiftLeft"` vs `"ShiftRight"` (chap/o'ng farqlanadi!). Shift bosilgan yoki yo'qligiga qaramay bir xil code.

Qachon qaysi birini ishlatish:
- **Shortcut'lar, tekstli input**: `event.key` — foydalanuvchi nima kiritmoqchiligi muhim
- **Game control, WASD, fizik pozitsiya**: `event.code` — layout'dan mustaqil bo'lishi kerak
- **Ikkalasi ham**: `Ctrl+S` kabi — `event.ctrlKey && event.key === "s"` yoki `event.code === "KeyS"`

<details>
<summary><strong>Under the Hood</strong></summary>

**Modifier klavishalar** alohida property'lar orqali: `event.shiftKey`, `event.ctrlKey`, `event.altKey`, `event.metaKey` (Mac'da Cmd, Windows'da Win kalit). **`metaKey`** cross-platform shortcut'lar uchun muhim — Mac'da `⌘+S`, Windows/Linux'da `Ctrl+S`.

**IME (Input Method Editor)** — Xitoy, Yapon, Koreys kabi tillar uchun klavishalar kombinatsiyasidan belgilar yig'iladi. IME composition paytida **`event.isComposing === true`** — bu holatda `keydown`'ga javob bermaslik kerak (foydalanuvchi hali yakuniy belgini kiritmagan). Yomon kodda bu muammo "IME users'da shortcut ishlamaydi" bug'iga olib keladi. Yechim: `beforeinput`/`input` event'larini ishlatish yoki `keydown`'da `event.isComposing` tekshirish.

**Auto-repeat** — klavisha bosib turilganda `keydown` qayta-qayta fire bo'ladi (OS repeat rate'iga qarab). **`event.repeat === true`** auto-repeat'ni aniqlaydi. Game loop yoki character counter kabi kontekstda bu muhim — birinchi bosishgina hisobga olinishi kerak bo'lsa `if (event.repeat) return` ishlatiladi.

**Dead key'lar** — aksent kiritish uchun klavishalar (masalan, International layout'da `^` + `a` → `â`). Birinchi bosish `event.key === "Dead"` beradi, keyingi bosish compose qilinadi. `beforeinput` event'i dead key compose bo'lgandan keyingi yakuniy matn bilan ishlash uchun yaxshiroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ─── key vs code farqi ───
document.addEventListener("keydown", (e) => {
  console.log(`key: "${e.key}", code: "${e.code}"`);
});
// QWERTY layout'da "A" bosilsa:
//   Shift'siz: key: "a", code: "KeyA"
//   Shift bilan: key: "A", code: "KeyA"
// AZERTY layout'da "A" fizik pozitsiyasida:
//   key: "q", code: "KeyA"  ← key layout'ga bog'liq, code emas

// ─── Shortcut — key usage ───
document.addEventListener("keydown", (e) => {
  // Cross-platform Ctrl/Cmd + S
  if ((e.ctrlKey || e.metaKey) && e.key === "s") {
    e.preventDefault(); // brauzer save dialog'ini to'xtatish
    saveDocument();
  }

  // Escape — modal close
  if (e.key === "Escape") {
    closeModal();
  }

  // Arrow keys — list navigation
  if (e.key === "ArrowDown") {
    e.preventDefault(); // scroll'ni to'xtatish
    selectNextItem();
  }
});

// ─── Game controls — code usage ───
const keysPressed = new Set();
document.addEventListener("keydown", (e) => {
  keysPressed.add(e.code);
  // WASD yoki arrow keys — har qanday layout'da ishlaydi
  if (e.code === "KeyW" || e.code === "ArrowUp")    movePlayer("up");
  if (e.code === "KeyS" || e.code === "ArrowDown")  movePlayer("down");
  if (e.code === "KeyA" || e.code === "ArrowLeft")  movePlayer("left");
  if (e.code === "KeyD" || e.code === "ArrowRight") movePlayer("right");
});
document.addEventListener("keyup", (e) => keysPressed.delete(e.code));

// ─── Chap/o'ng modifier ajratish ───
document.addEventListener("keydown", (e) => {
  if (e.code === "ShiftLeft")  console.log("Left Shift");
  if (e.code === "ShiftRight") console.log("Right Shift");
  // event.shiftKey — ikkalasi uchun true, farqlamaydi
});

// ─── Auto-repeat filter ───
document.addEventListener("keydown", (e) => {
  if (e.repeat) return; // klavisha bosib turilgan — skip
  // Birinchi bosish — bir marta bajariladi
  fireOnce();
});

// ─── IME awareness ───
const input = document.querySelector("input");
input.addEventListener("keydown", (e) => {
  if (e.isComposing || e.keyCode === 229) {
    // IME composition — hali yakuniy kiritish emas
    return;
  }
  // Normal keydown handling
});

// ✅ Zamonaviy — beforeinput IME bilan toza ishlaydi:
input.addEventListener("beforeinput", (e) => {
  console.log("Kiritilmoqchi:", e.data); // yakuniy composed matn
  if (e.data && !isValidInput(e.data)) {
    e.preventDefault(); // noto'g'ri input'ni block qilish
  }
});

// ─── Accessibility — ARIA pattern ───
// Custom button: <div role="button" tabindex="0">
const customBtn = document.querySelector('[role="button"]');
customBtn.addEventListener("click", handleAction);
customBtn.addEventListener("keydown", (e) => {
  // Native button uchun space/enter click trigger qiladi,
  // custom button'da qo'lda qilish kerak — accessibility uchun muhim
  if (e.key === "Enter" || e.key === " ") {
    e.preventDefault(); // space scroll'ni to'xtatish
    handleAction();
  }
});
```

</details>

---

## Composed Events va Shadow DOM

### Nazariya

Shadow DOM encapsulation'i event propagation'iga ham ta'sir qiladi. Shadow root ichida dispatch qilingan event'lar **default** Shadow boundary'dan tashqariga **chiqmaydi** — faqat `composed: true` flag'i bilan yaratilgan event'lar shadow tree'dan light (outer) DOM'ga "oqib" chiqadi. Bu encapsulation'ning bir qismi — komponent ichki event'larini tashqariga "leak" qilmaslik uchun.

**Native event'lar** (click, input, focus, keydown, ...) spec bo'yicha **default composed: true** — shadow DOM'dan tabiiy o'tadi. Custom event'lar esa default **`composed: false`** — agar komponent tashqariga signal yubormoqchi bo'lsa, `new CustomEvent("name", { composed: true, bubbles: true })` aniq belgilash kerak.

**`event.composedPath()`** — event'ning to'liq propagation yo'lini qaytaradi, **shadow DOM ichini ham o'z ichiga olib**. Oddiy `event.target` faqat shadow boundary'ning **retarget qilingan** targeti ko'rsatadi (komponent tashqaridagi kod uchun ichki node emas, host element ko'rinadi). Bu encapsulation'ning event tomoni.

<details>
<summary><strong>Under the Hood</strong></summary>

**Event retargeting** — shadow DOM'dan event tashqariga chiqqanda spec `target`'ni **shadow host**'ga o'zgartiradi. Ya'ni tashqi listener `event.target` deb tekshirsa — u shadow host'ni ko'radi, ichki element emas. Bu "seeing through" shadow'ning oldini oladi. Lekin `event.composedPath()` hali ham haqiqiy original target va to'liq yo'lni qaytaradi — debugging va intentional access uchun.

Event bir necha shadow tree'dan o'tganda har bir shadow boundary'da qayta retarget bo'ladi. Masalan, shadow A ichida shadow B bo'lsa va B ichidan event chiqsa — A ichida target = B host, A tashqarisida target = A host.

**`closed` shadow root** ichida dispatch qilingan event'lar ham tashqariga retarget bilan chiqishi mumkin — `mode: "closed"` faqat DOM API access'ni cheklaydi, event flow'ga ta'sir qilmaydi. Ya'ni closed shadow ichidagi click event hali ham outer document'da eshitiladi (agar composed: true bo'lsa).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ─── Custom Element — composed event dispatch ───
class FancyButton extends HTMLElement {
  connectedCallback() {
    const innerBtn = this.shadowRoot.getElementById("inner");
    innerBtn.addEventListener("click", (e) => {
      // ❌ bu event tashqariga chiqmaydi (default composed: false):
      // this.dispatchEvent(new CustomEvent("fancy-click"));

      // ✅ composed + bubbles — shadow boundary'dan o'tadi:
      this.dispatchEvent(new CustomEvent("fancy-click", {
        bubbles: true,
        composed: true,
        detail: { timestamp: Date.now() }
      }));
    });
  }
}

// ─── Tashqi listener ───
document.querySelector("fancy-button").addEventListener("fancy-click", (e) => {
  console.log("Received:", e.detail);
  console.log("target:", e.target);
  // <fancy-button> — host (retargeted)
  // Ichki <button> ko'rinmaydi — bu encapsulation

  console.log("composedPath:", e.composedPath());
  // [<button#inner>, <slot>, <shadow-root>, <fancy-button>, body, html, document, Window]
  // composedPath() — shadow DOM ichidagi haqiqiy click path'ni ochadi
});

// ─── Event retargeting misoli (ikki darajali shadow) ───
// <outer-component>
//   shadow: <inner-component>
//     shadow: <button>Deep Click</button>
//
// Ichki button bosilsa, event propagation:
// 1. Inner shadow'da: target = <button>, currentTarget chain ichida haqiqiy path
// 2. Inner host'ga chiqgach: target = <inner-component> (retargeted)
// 3. Outer shadow'da: target = <inner-component>
// 4. Outer host'ga chiqgach: target = <outer-component> (yana retarget)
// 5. document'da: target = <outer-component>

document.addEventListener("click", (e) => {
  console.log("Document - target:", e.target.tagName);
  // "OUTER-COMPONENT" — ikki marta retarget bo'lgan

  // composedPath() to'liq yo'lni ochadi:
  const path = e.composedPath().map(n => n.tagName || n.constructor.name);
  console.log("Full path:", path);
  // ["BUTTON", "INNER-COMPONENT", "OUTER-COMPONENT", "BODY", "HTML", "#document", "Window"]
});

// ─── Native vs Custom composed default ───
// Native eventlar (click, input, focus, keydown) — default composed: true:
shadowElement.addEventListener("click", () => {});
// Outer document'da ham eshitiladi — shadow boundary'dan tabiiy o'tadi

// Custom eventlar — default composed: false:
const hidden = new CustomEvent("my-event"); // composed: false
shadowElement.dispatchEvent(hidden);
// Outer document'da EHSHITILMAYDI — shadow ichida qoladi

const visible = new CustomEvent("my-event", { composed: true, bubbles: true });
shadowElement.dispatchEvent(visible);
// Outer document'da eshitiladi ✅

// ─── composedPath() bilan shadow-aware delegation ───
// Custom component'lardagi click'larni tashqi listener orqali handle qilish:
document.addEventListener("click", (e) => {
  // composedPath target'dan tashqari ichki node'larni ham ko'rsatadi
  const path = e.composedPath();
  for (const node of path) {
    if (node.matches && node.matches("[data-action]")) {
      handleAction(node.dataset.action);
      break;
    }
  }
});
```

</details>

---

## Edge Cases va Gotchas

Event handling bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha.

### Gotcha 1: `focus` va `blur` bubble qilmaydi — delegation ishlamaydi

`focus`, `blur`, va `load`/`error` event'lari **bubble qilmaydi**, shuning uchun ota elementda delegation bilan ushlab bo'lmaydi. Yechim: bubble qiluvchi alternativalar — `focusin`/`focusout`, yoki capture phase bilan parent'da eshitish.

```javascript
const form = document.getElementById("myForm");

// ❌ Ishlamaydi — focus bubble qilmaydi:
form.addEventListener("focus", (e) => {
  console.log("Focused:", e.target.name);
});
// Hech qachon chaqirilmaydi — focus form'ga yetmaydi

// ✅ Yechim 1: focusin / focusout (bubble qiladi):
form.addEventListener("focusin", (e) => {
  console.log("Focused:", e.target.name); // ishlaydi ✅
});
form.addEventListener("focusout", (e) => {
  console.log("Blurred:", e.target.name);
});

// ✅ Yechim 2: Capture phase — focus'ni tashqaridan ushlash:
form.addEventListener("focus", (e) => {
  console.log("Focused (capture):", e.target.name);
}, true); // { capture: true }
// focus capturing phase'da parent'ga yetadi, bubbling'da yetmaydi
```

**Nima uchun:** DOM Events spec'ning tarixiy qarori — `focus`/`blur` bubble qilmaydi, har field o'z focus'ini alohida boshqarishi uchun. Lekin real UI pattern'larda (form validation, auto-save) delegation kerak — shuning uchun `focusin`/`focusout` alternativa sifatida qo'shilgan.

**Yechim:** Form delegation uchun `focusin`/`focusout` afzal. `load`/`error` uchun image/iframe'da capture phase bilan eshitish kerak.

### Gotcha 2: `mouseenter` vs `mouseover` — nested element'larda behavior farq

`mouseover` va `mouseenter` o'xshash, lekin ikki muhim farq: (1) `mouseenter` bubble qilmaydi, `mouseover` bubble qiladi; (2) `mouseover` **nested child element'ga kursor kirganida qayta fire bo'ladi**, `mouseenter` faqat outer bounding'ga kirganda bir marta trigger bo'ladi.

```javascript
// Outer ichida Inner bor
const outer = document.getElementById("outer");

// ❌ mouseover — nested element'lar orasida ham fire bo'ladi:
outer.addEventListener("mouseover", (e) => {
  console.log("mouseover", e.target.id);
});
// Kursor outer'ga kirsa: "mouseover outer"
// Kursor inner'ga kirsa: "mouseover inner" (inner'dan bubble)
// Inner'dan outer'ga chiqsa: yana "mouseover outer"
// Har nested element orasida harakatlanish → ko'p event

// ✅ mouseenter — faqat outer bounding'ga kirganda:
outer.addEventListener("mouseenter", () => {
  console.log("mouseenter outer"); // faqat 1 marta outer'ga kirganda
});
outer.addEventListener("mouseleave", () => {
  console.log("mouseleave outer"); // faqat outer bounding'dan chiqqanda
});
// Inner'ga kirish-chiqish OUTER event'ga ta'sir qilmaydi ✅
```

**Nima uchun:** `mouseover`/`mouseout` — "element yoki uning har qanday child'iga hover" semantikasi, nested child'larga ham trigger bo'ladi. `mouseenter`/`mouseleave` — "element bounding'iga hover" — nested child'larni e'tiborga olmaydi. IE o'zining `mouseenter`'ini birinchi kiritgan, keyinchalik DOM spec'ga qo'shilgan.

**Yechim:** Hover effekti uchun odatda `mouseenter`/`mouseleave` (bir marta, toza). `mouseover`/`mouseout` — nested child'ga asoslangan nozik holatlar uchun, `relatedTarget` bilan "haqiqatan boshqa element'dan kirdimi?" tekshirish mumkin.

### Gotcha 3: Anonymous `removeEventListener` ishlamaydi — reference equality

`removeEventListener` callback'ni **reference equality** bilan topadi — xuddi shu funksiya object bo'lishi kerak. Arrow function yoki `.bind(this)` har chaqirilganda yangi funksiya yaratadi.

```javascript
// ❌ Arrow function — yangi funksiya har safar:
button.addEventListener("click", () => console.log("click"));
button.removeEventListener("click", () => console.log("click"));
// Ikki arrow function — turli reference, remove hech narsa qilmaydi

// ❌ .bind(this) — har chaqiriqda yangi bound function:
class Component {
  setup() {
    this.element.addEventListener("click", this.handleClick.bind(this));
  }
  cleanup() {
    this.element.removeEventListener("click", this.handleClick.bind(this));
    // .bind har safar yangi reference — remove ishlamaydi!
  }
  handleClick() { /* ... */ }
}

// ✅ Yechim 1: bind natijasini saqlash
class Component {
  setup() {
    this.boundHandler = this.handleClick.bind(this); // bir marta
    this.element.addEventListener("click", this.boundHandler);
  }
  cleanup() {
    this.element.removeEventListener("click", this.boundHandler);
  }
}

// ✅ Yechim 2 (eng yaxshi): AbortController
class Component {
  setup() {
    this.controller = new AbortController();
    this.element.addEventListener("click", () => this.handleClick(), {
      signal: this.controller.signal
    });
  }
  cleanup() {
    this.controller.abort(); // barcha signal listener'lar o'chiriladi
  }
}

// ✅ Yechim 3: Arrow class field — reference bir xil:
class Component {
  handleClick = () => { /* ... */ }; // class field, bir marta yaratiladi
  setup()   { this.element.addEventListener("click", this.handleClick); }
  cleanup() { this.element.removeEventListener("click", this.handleClick); }
}
```

**Nima uchun:** DOM spec `(type, callback, capture)` uchlik bilan listener'ni aniqlaydi, callback'ni `===` bilan taqqoslaydi. Har arrow function literal yangi function object yaratadi, `.bind()` ham yangi bound function yaratadi.

**Yechim:** `AbortController` — eng zamonaviy va toza yondashuv. Memory leak'ni oldini olish uchun har component unmount'da explicit cleanup kerak.

### Gotcha 4: `<div role="button">` keyboard'da ishlamaydi — native button NOT equivalent

`<button>` elementida Enter/Space bosilsa — brauzer avtomatik `click` event'ni sintez qiladi (accessibility). Lekin `<div role="button">` yaratsangiz — bu sintez **ishlamaydi**. Keyboard handler qo'shmasdan custom tugma aktsessability buzadi.

```javascript
// ✅ Native button — keyboard avtomatik ishlaydi:
document.querySelector(".native").addEventListener("click", () => {
  console.log("Native clicked");
  // Enter/Space bosilsa ham click fire bo'ladi — brauzer sintez qiladi
});

// ❌ Custom div — click listener keyboard'da ishlamaydi:
const custom = document.querySelector(".custom");
custom.addEventListener("click", () => {
  console.log("Custom clicked");
});
// Mouse click bilan ishlaydi, lekin Enter/Space bilan YO'Q!

// ✅ To'g'ri custom button pattern:
function makeCustomButton(element, handler) {
  element.setAttribute("role", "button");
  element.setAttribute("tabindex", "0"); // keyboard fokus uchun

  element.addEventListener("click", handler);
  element.addEventListener("keydown", (e) => {
    // Enter va Space — native button kabi:
    if (e.key === "Enter" || e.key === " ") {
      e.preventDefault(); // Space scroll'ni to'xtatish
      handler.call(element, e);
    }
  });
}
```

**Nima uchun:** HTML/ARIA spec bo'yicha `<button>` "interactive content" — brauzer default keyboard behavior'ni (Enter/Space → click) avtomatik handle qiladi. Custom element'lar uchun brauzer buni qilmaydi — element "maqsadi"ni bilmaydi. `role="button"` faqat assistive technology'larga semantik ma'lumot beradi, keyboard behavior'ni majburlamaydi.

**Yechim:** Iloji bo'lsa **native `<button>`** ishlating. Custom kerak bo'lsa — `role="button"` + `tabindex="0"` + keydown handler'da Enter/Space sintez qilish shart.

### Gotcha 5: Listener duplicate — `(type, callback, capture)` uchlik bo'yicha, options e'tiborsiz

DOM spec bo'yicha **aynan bir xil `type + callback + capture`** uchlik bo'lgan listener duplicate qo'shilmaydi — spec "silent skip" qiladi. Lekin `{ once: true }`, `{ passive: true }` kabi **boshqa options** duplicate'ga ta'sir qilmaydi — ya'ni ikkinchi chaqiriq yangi yozuv qo'shmaydi, **birinchi yozuvning options'lari saqlanib qoladi**.

```javascript
function handler() { console.log("clicked"); }

const button = document.querySelector("button");

// 1-chi add — ro'yxatga qo'shiladi
button.addEventListener("click", handler);

// 2-chi add — BIR XIL (type, callback, capture) — SKIP!
button.addEventListener("click", handler);
button.addEventListener("click", handler);
// Ro'yxatda faqat 1 ta yozuv bor

button.click(); // "clicked" — bir marta!

// ⚠️ options farq qilsa ham duplicate deb hisoblanadi:
button.addEventListener("click", handler, { once: true });
// Yuqoridagi bilan (type, callback, capture=false) bir xil — SKIP
// Birinchi listener once: false bilan qoladi!

// ✅ capture farqi — alohida listener:
button.addEventListener("click", handler, true); // capture: true
// Bu boshqa yozuv — ro'yxatga qo'shiladi

// ─── Common mistake: boshqa options bilan qayta add ───
element.addEventListener("scroll", handleScroll);
// Keyinroq "passive qilamiz":
element.addEventListener("scroll", handleScroll, { passive: true });
// REAL: hech narsa bo'lmadi — spec duplicate deb hisobladi
// handleScroll hali ham passive: false bilan ishlayapti!

// ✅ To'g'ri usul — avval remove, keyin yangi options bilan add:
element.removeEventListener("scroll", handleScroll);
element.addEventListener("scroll", handleScroll, { passive: true });
```

**Nima uchun:** Spec bo'yicha event listener list'i `(type, callback, capture)` uchlik bilan kalitlanadi — `once`, `passive`, `signal` options esa faqat listener yozuvi ichida saqlanadi. Ikkinchi `addEventListener` chaqiruvi "bor-yo'qligini" tekshiradi, mavjud bo'lsa silent skip qiladi.

**Yechim:** Listener options'ni o'zgartirish kerak bo'lsa — avval `removeEventListener`, keyin yangi options bilan `addEventListener`. Yoki `AbortController` bilan abort + yangi listener.

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
5: button BUBBLE
6: button CAPTURE
3: body BUBBLE
1: document BUBBLE
```

**Tushuntirish:**

1. **Capturing phase** (document → body, target'dan yuqoridagi ancestor'lar): faqat `capture: true` listener'lar ishlaydi → `2: document CAPTURE`, `4: body CAPTURE`.

2. **Target phase** (button'da): DOM spec bo'yicha target node'da **barcha listener'lar registration tartibida** ishlaydi — capture flag e'tiborga olinmaydi. Button'da listener'lar qo'shilish tartibi: line 936 `5 BUBBLE` birinchi, line 937 `6 CAPTURE` ikkinchi → `5: button BUBBLE`, `6: button CAPTURE`.

3. **Bubbling phase** (body → document): faqat `capture: false` (default) listener'lar ishlaydi → `3: body BUBBLE`, `1: document BUBBLE`.

**Muhim nuans:** Target phase'da capture/bubble tartibi **registration order** ga bog'liq, capture flag'iga emas. Agar button'da `CAPTURE` birinchi, `BUBBLE` ikkinchi qo'shilganda — natija teskari bo'lardi: `6: button CAPTURE`, so'ng `5: button BUBBLE`.

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
