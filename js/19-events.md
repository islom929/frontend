# Bo'lim 19: Events

> DOM Events — brauzerda foydalanuvchi harakatlari va tizim hodisalarini boshqarish mexanizmi.

---

## Mundarija

- [Event Model](#event-model)
- [Event Bubbling](#event-bubbling)
- [Event Capturing](#event-capturing)
- [Event Flow: Capturing → Target → Bubbling](#event-flow)
- [addEventListener](#addeventlistener)
- [stopPropagation va stopImmediatePropagation](#stoppropagation-va-stopimmediatepropagation)
- [preventDefault](#preventdefault)
- [Event Delegation](#event-delegation)
- [event.target vs event.currentTarget](#eventtarget-vs-eventcurrenttarget)
- [Custom Events](#custom-events)
- [Event Listeners va Memory](#event-listeners-va-memory)
- [Passive Event Listeners](#passive-event-listeners)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Event Model

### Nazariya

Event (hodisa) — brauzerda sodir bo'ladigan har qanday narsa: foydalanuvchi click qilishi, klaviatura bosishi, sahifa yuklanganligi, scroll, formani jo'natishi va hokazo. JavaScript orqali biz bu hodisalarni ushlash va ularga javob berish mumkin.

Event bilan ishlashning 3 ta usuli bor:

### 1. HTML Attribute (eski, tavsiya qilinmaydi)

```html
<!-- ❌ Noto'g'ri — HTML va JS aralashib ketadi -->
<button onclick="alert('Bosildi!')">Bos</button>
<button onclick="handleClick(event)">Bos</button>
```

### 2. DOM Property (cheklangan)

```javascript
const btn = document.getElementById("btn");

// 🟡 Ishlaydi, lekin faqat BITTA handler
btn.onclick = function(event) {
  console.log("Click 1");
};

btn.onclick = function(event) {
  console.log("Click 2");
};
// Faqat "Click 2" ishlaydi — oldingi overwrite bo'ldi!
```

### 3. addEventListener (zamonaviy, to'g'ri)

```javascript
const btn = document.getElementById("btn");

// ✅ Ko'p handler qo'shish mumkin
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
| HTML attribute | ❌ | ❌ | ❌ | ❌ |
| DOM property | ❌ | Overwrite | ❌ | 🟡 |
| `addEventListener` | ✅ | ✅ | ✅ | ✅ |

---

## Event Bubbling

### Nazariya

Event Bubbling — hodisa **ichki** elementdan **tashqi** elementlarga qarab ko'tarilishi. Bu DOM ning default xulq-atvori.

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
// "3: INNER"   ← bosilgan element (target)
// "2: MIDDLE"  ← yuqoriga ko'tarildi
// "1: OUTER"   ← yana yuqoriga
```

### ASCII Diagram

```
                    BUBBLING (ichdan tashqariga)
                    
  ┌─────────────────────────────────────┐
  │ document                             │
  │  ┌──────────────────────────────┐   │
  │  │ #outer                    ③  │   │  ← 3-chi handler ishlaydi
  │  │  ┌───────────────────────┐   │   │
  │  │  │ #middle            ②  │   │   │  ← 2-chi handler ishlaydi
  │  │  │  ┌────────────────┐   │   │   │
  │  │  │  │ button      ①  │   │   │   │  ← 1-chi (target)
  │  │  │  │   [CLICK!]     │   │   │   │
  │  │  │  └────────────────┘   │   │   │
  │  │  └───────────────────────┘   │   │
  │  └──────────────────────────────┘   │
  └─────────────────────────────────────┘
```

### Under the Hood

Deyarli barcha event'lar bubble qiladi. Lekin ba'zi event'lar **bubble qilmaydi**:

| Event | Bubbles? |
|-------|----------|
| `click`, `mousedown`, `mouseup` | ✅ |
| `keydown`, `keyup`, `keypress` | ✅ |
| `input`, `change` | ✅ |
| `submit` | ✅ |
| `scroll` | ✅ (lekin capturing da ushlanishi qiyin) |
| `focus`, `blur` | ❌ (bubbles emas) |
| `focusin`, `focusout` | ✅ (focus/blur ning bubble versiyasi) |
| `mouseenter`, `mouseleave` | ❌ |
| `mouseover`, `mouseout` | ✅ |
| `load`, `unload`, `error` | ❌ |

---

## Event Capturing

### Nazariya

Event Capturing (yoki "trickling") — hodisa **tashqi** elementdan **ichki** elementga qarab tushishi. Bu bubbling ning teskarisi. Capturing faqat `addEventListener` ning uchinchi argumenti orqali yoqiladi.

### Kod Misollari

```javascript
// Capturing yoqish — 3-argument: true yoki { capture: true }
document.getElementById("outer").addEventListener("click", () => {
  console.log("OUTER (capture)");
}, true); // ← capture mode

document.getElementById("middle").addEventListener("click", () => {
  console.log("MIDDLE (capture)");
}, { capture: true }); // ← capture mode

document.getElementById("inner").addEventListener("click", () => {
  console.log("INNER (target)");
});

// Button bosilsa:
// "OUTER (capture)"   ← tashqaridan boshlanadi (capturing phase)
// "MIDDLE (capture)"  ← ichkariga qarab tushadi
// "INNER (target)"    ← target element
```

---

## Event Flow

### Nazariya

Har bir DOM event **3 ta fazadan** o'tadi:

1. **Capturing Phase** — `document` dan target element gacha (tashqaridan ichga)
2. **Target Phase** — target element o'zida
3. **Bubbling Phase** — target dan `document` gacha (ichdan tashqiga)

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

// Capturing handler'lar
outer.addEventListener("click", () => console.log("outer CAPTURE"), true);
middle.addEventListener("click", () => console.log("middle CAPTURE"), true);
inner.addEventListener("click", () => console.log("inner CAPTURE"), true);

// Bubbling handler'lar
outer.addEventListener("click", () => console.log("outer BUBBLE"));
middle.addEventListener("click", () => console.log("middle BUBBLE"));
inner.addEventListener("click", () => console.log("inner BUBBLE"));

// inner button bosilsa, tartib:
// 1. "outer CAPTURE"   ← Capturing phase
// 2. "middle CAPTURE"  ← Capturing phase
// 3. "inner CAPTURE"   ← Target phase (yozilgan tartibda)
// 4. "inner BUBBLE"    ← Target phase (yozilgan tartibda)
// 5. "middle BUBBLE"   ← Bubbling phase
// 6. "outer BUBBLE"    ← Bubbling phase
```

### Under the Hood

Target element da capturing va bubbling handler'larning ishlash tartibi — **yozilgan tartibda** (spec bo'yicha). Ba'zi eski brauzerlar buni noto'g'ri qilardi, lekin zamonaviy brauzerlar to'g'ri implement qilgan.

---

## addEventListener

### Nazariya

`addEventListener` — event handler qo'shishning zamonaviy va mos usuli.

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
// Oddiy handler
function handleClick(event) {
  console.log("Bosildi!", event.type);
}
button.addEventListener("click", handleClick);

// once — faqat bir marta ishlaydi
button.addEventListener("click", () => {
  console.log("Faqat bir marta!");
}, { once: true });
// Birinchi click dan keyin avtomatik removeEventListener bo'ladi

// AbortController bilan o'chirish
const controller = new AbortController();

button.addEventListener("click", () => {
  console.log("Click!");
}, { signal: controller.signal });

window.addEventListener("keydown", (e) => {
  console.log("Key:", e.key);
}, { signal: controller.signal });

// Bir marta controller.abort() chaqirsak — IKKISI HAM o'chiriladi!
controller.abort();
// Bu ko'p listener'larni bir vaqtda tozalash uchun juda qulay
```

### removeEventListener

```javascript
// ⚠️ removeEventListener uchun AYNI funksiya reference kerak!

// ❌ Ishlamaydi — yangi funksiya
button.addEventListener("click", () => console.log("A"));
button.removeEventListener("click", () => console.log("A"));
// Bu BOSHQA funksiya — reference teng emas!

// ✅ Ishlaydi — BIR XIL funksiya reference
function handler() { console.log("A"); }
button.addEventListener("click", handler);
button.removeEventListener("click", handler);

// ⚠️ capture/bubble mos kelishi kerak
button.addEventListener("click", handler, true);  // capture
button.removeEventListener("click", handler, true); // ✅ capture ham true
button.removeEventListener("click", handler);       // ❌ ishlamaydi — default false
```

---

## stopPropagation va stopImmediatePropagation

### Nazariya

- `stopPropagation()` — hodisa boshqa elementlarga **tarqalishini** to'xtatadi (bubble/capture to'xtaydi)
- `stopImmediatePropagation()` — tarqalishni to'xtatadi VA shu elementdagi **qolgan** handler'lar ham bajarilmaydi

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
// "OUTER" — ❌ BAJARILMAYDI (propagation to'xtatilgan)


// === stopImmediatePropagation ===
button.addEventListener("click", (e) => {
  console.log("Handler 1");
  e.stopImmediatePropagation();
});

button.addEventListener("click", (e) => {
  console.log("Handler 2"); // ❌ BAJARILMAYDI!
});

// Button bosilsa: faqat "Handler 1"
// Handler 2 shu elementda, lekin stopImmediate tozaladi
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

`preventDefault()` — brauzerning **standart xulq-atvorini** bekor qiladi. Lekin hodisa propagation sini TO'XTATMAYDI — hodisa davom etadi.

### Kod Misollari

```javascript
// === Link ni to'xtatish ===
const link = document.querySelector("a");
link.addEventListener("click", (e) => {
  e.preventDefault(); // sahifa boshqa URL ga o'tmaydi
  console.log("Link bosildi, lekin navigatsiya yo'q");
});

// === Form submit ni to'xtatish ===
const form = document.querySelector("form");
form.addEventListener("submit", (e) => {
  e.preventDefault(); // sahifa reload bo'lmaydi
  
  const data = new FormData(form);
  console.log("Form data:", Object.fromEntries(data));
  // AJAX bilan yuborish mumkin
});

// === Kontekst menyu ni to'xtatish ===
document.addEventListener("contextmenu", (e) => {
  e.preventDefault(); // o'ng click menyu chiqmaydi
  // Custom menyu ko'rsatish mumkin
});

// === Checkbox ni to'xtatish ===
const checkbox = document.querySelector("input[type='checkbox']");
checkbox.addEventListener("click", (e) => {
  e.preventDefault(); // checkbox holati o'zgarmaydi
});
```

### preventDefault vs stopPropagation vs return false

```javascript
// preventDefault — faqat default behavior to'xtatadi
// stopPropagation — faqat propagation to'xtatadi
// Ular ALOHIDA ishlar qiladi!

link.addEventListener("click", (e) => {
  e.preventDefault();    // navigatsiya yo'q
  e.stopPropagation();   // bubbling yo'q
  // Ikkisi birga — na default, na propagation
});

// ⚠️ return false
// addEventListener da return false hech nima qilMAYDI!
button.addEventListener("click", () => {
  return false; // ❌ hech narsa to'xtatmaydi
});

// Faqat onclick PROPERTY da return false ishlaydi (eski jQuery da ham)
button.onclick = () => { return false; }; // preventDefault + stopPropagation
```

---

## Event Delegation

### Nazariya

Event Delegation — har bir elementga alohida handler qo'shish o'rniga, **ota elementga bitta** handler qo'yish va `event.target` orqali qaysi element bosilganini aniqlash. Bu pattern bubbling asosida ishlaydi.

### Nima Uchun Kerak?

```javascript
// ❌ NOTO'G'RI — 1000 ta handler qo'shish
const items = document.querySelectorAll(".item"); // 1000 ta element
items.forEach(item => {
  item.addEventListener("click", (e) => {
    console.log("Item bosildi:", item.textContent);
  });
});
// 1000 ta funksiya yaratildi — memory isrof!
// Yangi qo'shilgan elementlar ISHLAMAYDI — ular keyin qo'shilgan

// ✅ TO'G'RI — bitta handler (delegation)
document.getElementById("list").addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (!item) return; // .item bosilmagan
  console.log("Item bosildi:", item.textContent);
});
// 1 ta funksiya — 1000 ta element uchun ishlaydi
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

// Delegation — barcha tab'lar uchun bitta handler
tabs.addEventListener("click", (e) => {
  const tab = e.target.closest(".tab");
  if (!tab) return;

  // Active class
  tabs.querySelectorAll(".tab").forEach(t => t.classList.remove("active"));
  tab.classList.add("active");

  // Content o'zgartirish
  const tabName = tab.dataset.tab;
  content.textContent = `${tabName} sahifasi`;
});
```

### Amaliy Misol: Dynamic Todo List

```javascript
const list = document.getElementById("todo-list");

// Delegation — hali yaratilmagan elementlar uchun ham ishlaydi
list.addEventListener("click", (e) => {
  // Delete button
  if (e.target.closest(".delete-btn")) {
    const li = e.target.closest("li");
    li.remove();
    return;
  }

  // Edit button
  if (e.target.closest(".edit-btn")) {
    const li = e.target.closest("li");
    const span = li.querySelector(".text");
    const newText = prompt("Yangi matn:", span.textContent);
    if (newText) span.textContent = newText;
    return;
  }

  // Toggle done
  if (e.target.closest("li")) {
    e.target.closest("li").classList.toggle("done");
  }
});

// Yangi element qo'shish — delegation tufayli avtomatik ishlaydi
function addTodo(text) {
  const li = document.createElement("li");
  li.innerHTML = `
    <span class="text">${text}</span>
    <button class="edit-btn">✏️</button>
    <button class="delete-btn">🗑️</button>
  `;
  list.appendChild(li);
}
```

### Under the Hood

Event delegation nima uchun ishlaydi:

```
1. Button bosildi
2. Event TARGET = button
3. Bubbling boshlanadi: button → li → ul → body → html → document
4. ul dagi handler ishlaydi
5. e.target = button (bosilgan element)
6. e.currentTarget = ul (handler qo'yilgan element)
7. closest(".item") bilan kerakli elementni topamiz
```

---

## event.target vs event.currentTarget

### Nazariya

| Property | Ma'no | O'zgaradimi? |
|----------|-------|-------------|
| `event.target` | Hodisa **sodir bo'lgan** element (click qilingan) | ❌ O'zgarmaydi |
| `event.currentTarget` | Handler **qo'yilgan** element | ✅ Har bosqichda o'zgaradi |

### Kod Misollari

```html
<div id="parent">
  <button id="child">Click me</button>
</div>
```

```javascript
document.getElementById("parent").addEventListener("click", (e) => {
  console.log("target:", e.target.id);        // "child" — bosilgan
  console.log("currentTarget:", e.currentTarget.id); // "parent" — handler shu yerda
});

// Button bosilganda target = child, currentTarget = parent
// Parent o'zi bosilganda target = parent, currentTarget = parent
```

```javascript
// this = currentTarget (oddiy funksiyada)
parent.addEventListener("click", function(e) {
  console.log(this === e.currentTarget); // true
  // ⚠️ Arrow function da this boshqa
});

parent.addEventListener("click", (e) => {
  console.log(this === e.currentTarget); // false — arrow da this = outer scope
});
```

---

## Custom Events

### Nazariya

JavaScript orqali o'z hodisalarimizni yaratish va dispatch qilish mumkin. Bu component'lar orasidagi kommunikatsiya uchun juda foydali — ayniqsa loosely coupled arxitekturada.

### Kod Misollari

```javascript
// === CustomEvent yaratish ===
const event = new CustomEvent("user-login", {
  detail: {
    userId: 42,
    username: "Ali",
    timestamp: Date.now()
  },
  bubbles: true,      // bubble qilsinmi?
  cancelable: true,    // preventDefault mumkinmi?
  composed: false      // Shadow DOM dan chiqsinmi?
});

// === Tinglash ===
document.addEventListener("user-login", (e) => {
  console.log("Login event:", e.detail);
  // { userId: 42, username: "Ali", timestamp: ... }
});

// === Dispatch qilish ===
document.dispatchEvent(event);

// Yoki element dan:
const loginForm = document.getElementById("login-form");
loginForm.dispatchEvent(event); // bubbles: true bo'lsa, yuqoriga ko'tariladi
```

### Real-World Pattern: Event Bus

```javascript
// Mini Event Bus — component'lar aro kommunikatsiya
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
  console.log("Cart yangilandi:", data.items.length, "ta buyum");
});

// Component B
bus.emit("cart-update", { items: ["telefon", "quloqchin"] });
// "Cart yangilandi: 2 ta buyum"
```

---

## Event Listeners va Memory

### Nazariya

Event listener'lar xotirada joy egallaydi. Agar to'g'ri tozalanmasa — memory leak bo'lishi mumkin. Ayniqsa **SPA** (Single Page Application) larda bu katta muammo.

### Memory Leak Holatlari

```javascript
// ❌ Element o'chirilsa ham, listener qoladi (agar closure reference bor bo'lsa)
function setupWidget(container) {
  const bigData = new Array(100000).fill("data"); // katta massiv

  // Handler closure orqali bigData ga reference saqlaydi
  const handler = () => {
    console.log(bigData.length);
  };

  window.addEventListener("scroll", handler);

  // Widget o'chirilganda:
  container.remove();
  // ⚠️ window scroll listener HALI QOLDI!
  // bigData ham GC tozalay olmaydi — handler reference saqlayotgan
}

// ✅ To'g'ri — cleanup funksiyasi
function setupWidget(container) {
  const bigData = new Array(100000).fill("data");

  const handler = () => {
    console.log(bigData.length);
  };

  window.addEventListener("scroll", handler);

  // Cleanup qaytarish
  return function cleanup() {
    window.removeEventListener("scroll", handler);
    // Endi bigData ham GC tozalay oladi
  };
}

const cleanup = setupWidget(container);
// Keyin:
cleanup(); // listener o'chirildi, memory bo'shatildi
```

### AbortController bilan Cleanup

```javascript
// ✅ Eng zamonaviy usul — AbortController
function setupComponent() {
  const controller = new AbortController();

  window.addEventListener("scroll", handleScroll, { signal: controller.signal });
  window.addEventListener("resize", handleResize, { signal: controller.signal });
  document.addEventListener("click", handleClick, { signal: controller.signal });

  return function cleanup() {
    controller.abort(); // BARCHA listener'lar bir marta bilan o'chiriladi!
  };
}
```

### Element Remove va Listener'lar

```javascript
// Element DOM dan o'chirilsa:
const element = document.getElementById("widget");
element.addEventListener("click", handler);
element.remove();
// Element dagi listener'lar element bilan birga yo'qoladi
// LEKIN: agar boshqa joyda element ga reference saqlansa — GC tozalamaydi

// Global listener'lar (window, document) alohida — ular DOM remove bilan o'chMAYDI!
```

---

## Passive Event Listeners

### Nazariya

`passive: true` brauzerga aytadi: "Bu handler `preventDefault()` chaqirmaydi." Bu scroll va touch event'lar uchun performanceni sezilarli oshiradi.

### Nima Uchun Kerak?

```javascript
// Muammo: scroll event handler ichida preventDefault bo'lishi MUMKIN
// Shuning uchun brauzer handler tugashini KUTADI — scroll sekinlashadi

// ❌ Brauzer kutishi kerak — sekin scroll
document.addEventListener("scroll", (e) => {
  // Og'ir hisoblash...
  // Brauzer: "preventDefault chaqiradimi? Kutaman..."
  // Scroll 100ms kechikadi
});

// ✅ passive: true — brauzer kutmaydi — tez scroll
document.addEventListener("scroll", (e) => {
  // Og'ir hisoblash...
  // Brauzer: "passive = true, preventDefault bo'lmaydi, scroll davom!"
  // e.preventDefault(); // ❌ Warning — passive da ishlamaydi!
}, { passive: true });
```

### Under the Hood

```
                     Passive: false (default)
  ┌──────────┐     ┌───────────────┐     ┌────────────┐
  │ Touch/   │────▶│ Handler       │────▶│ Scroll     │
  │ Scroll   │     │ (kutish)      │     │ (keyinroq) │
  └──────────┘     └───────────────┘     └────────────┘
                   Brauzer kutadi:
                   "preventDefault
                    chaqiladimi?"

                     Passive: true
  ┌──────────┐     ┌───────────────┐
  │ Touch/   │──┬─▶│ Handler       │     (parallel)
  │ Scroll   │  │  └───────────────┘
  └──────────┘  │  ┌────────────┐
                └─▶│ Scroll     │     (darhol!)
                   │ (tezkor)   │
                   └────────────┘
```

### Brauzer Defaults

```javascript
// Chrome 51+ dan boshlab:
// document-level touchstart/touchmove -> default passive: true
// Bu sahifada scroll tezligini oshiradi

// Agar haqiqatan ham preventDefault kerak bo'lsa:
document.addEventListener("touchmove", handler, { passive: false });
// Lekin bu scrollni sekinlashtiradi — haqiqatan kerak bo'lgandagina
```

---

## Common Mistakes

### ❌ Xato 1: removeEventListener da boshqa funksiya berish

```javascript
// ❌ Noto'g'ri — anonymous function remove bo'lmaydi!
button.addEventListener("click", () => console.log("A"));
button.removeEventListener("click", () => console.log("A")); // Ishlamaydi!
// Bu BOSHQA funksiya — reference teng emas
```

### ✅ To'g'ri usul:

```javascript
// ✅ Named function yoki o'zgaruvchi
const handler = () => console.log("A");
button.addEventListener("click", handler);
button.removeEventListener("click", handler); // Ishlaydi!

// ✅ Yoki { once: true }
button.addEventListener("click", () => console.log("A"), { once: true });

// ✅ Yoki AbortController
const controller = new AbortController();
button.addEventListener("click", () => console.log("A"), { signal: controller.signal });
controller.abort(); // O'chirildi
```

**Nima uchun:** JavaScript da har bir `() => {}` yangi funksiya yaratadi. Ular tashqi ko'rinishi bir xil bo'lsa ham, xotirada farqli obyektlar. `removeEventListener` aynan o'sha reference ni kutadi.

---

### ❌ Xato 2: Event delegation da target ni noto'g'ri tekshirish

```javascript
// ❌ Noto'g'ri — nested element bosilsa target boshqa bo'ladi
list.addEventListener("click", (e) => {
  if (e.target.classList.contains("item")) {
    console.log("Item bosildi");
  }
});

// HTML: <li class="item"><span>Text</span></li>
// span bosilsa → target = span, li emas → ISHLAMAYDI!
```

### ✅ To'g'ri usul:

```javascript
// ✅ closest() ishlatish — yuqoriga qarab qidiradi
list.addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (item) {
    console.log("Item bosildi:", item.textContent);
  }
});
```

**Nima uchun:** `e.target` eng **ichki** bosilgan element. Agar `<li>` ichida `<span>` bo'lsa, span bosilganda target = span. `closest()` yuqoriga qarab eng yaqin mos elementni topadi.

---

### ❌ Xato 3: preventDefault va stopPropagation ni aralashtirish

```javascript
// ❌ Noto'g'ri — "tugma ishlamayapti" deb stopPropagation ishlatish
form.addEventListener("submit", (e) => {
  e.stopPropagation(); // ❌ Bu FAQAT propagation to'xtatadi
  // Form hali ham submit bo'ladi! Sahifa reload bo'ladi!
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ preventDefault — brauzer default behavior ni to'xtatadi
form.addEventListener("submit", (e) => {
  e.preventDefault(); // ✅ Sahifa reload bo'lmaydi
  // AJAX bilan form yuborish
});
```

**Nima uchun:** `stopPropagation` boshqa elementlarga tarqalishni to'xtatadi. `preventDefault` brauzerning o'z xulq-atvorini (navigate, submit, scroll) to'xtatadi. Bular ikki xil narsa.

---

### ❌ Xato 4: Loop ichida handler qo'shishda closure muammosi

```javascript
// ❌ Event listener loop ichida noto'g'ri closure
const buttons = document.querySelectorAll("button");
for (var i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => {
    console.log("Button:", i); // Hammasi oxirgi qiymatni ko'rsatadi!
  });
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ let ishlatish
for (let i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => {
    console.log("Button:", i); // Har biri o'z qiymatini ko'rsatadi
  });
}

// ✅ Yoki forEach
buttons.forEach((btn, i) => {
  btn.addEventListener("click", () => console.log("Button:", i));
});

// ✅ Yoki delegation (eng yaxshi)
const container = document.querySelector(".buttons-container");
container.addEventListener("click", (e) => {
  const btn = e.target.closest("button");
  if (btn) console.log("Button:", btn.dataset.index);
});
```

**Nima uchun:** `var` function scope — bitta `i` barchaga umumiy. `let` block scope — har iteratsiya yangi `i`.

---

## Amaliy Mashqlar

### Mashq 1: Event Flow Tartibi (Oson)

**Savol:** Quyidagi kodda button bosilganda console da nima chiqadi? Tartibini yozing.

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

**Tushuntirish:**
1. **Capturing phase** (tashqaridan ichga): document CAPTURE → body CAPTURE
2. **Target phase** (yozilgan tartibda): button CAPTURE → button BUBBLE
3. **Bubbling phase** (ichdan tashqariga): body BUBBLE → document BUBBLE

</details>

---

### Mashq 2: Delegation bilan Dropdown Menu (O'rta)

**Savol:** Event delegation ishlatib dropdown menu yarating:
- `.menu` elementga bitta handler qo'ying
- Qaysi `menu-item` bosilganini aniqlang
- Bosilgan itemga `active` class qo'ying, boshqalardan olib tashlang

<details>
<summary>Javob</summary>

```javascript
const menu = document.querySelector(".menu");

menu.addEventListener("click", (e) => {
  const item = e.target.closest(".menu-item");
  if (!item) return;
  if (!menu.contains(item)) return; // xavfsizlik — faqat shu menu ichida

  // Barchadan active ni olib tashlash
  menu.querySelectorAll(".menu-item.active").forEach(el => {
    el.classList.remove("active");
  });

  // Bosilganga active qo'shish
  item.classList.add("active");

  console.log("Tanlandi:", item.dataset.value);
});
```

**Tushuntirish:** `closest(".menu-item")` bosilgan elementdan yuqoriga qarab eng yaqin `.menu-item` ni topadi. `menu.contains(item)` — faqat shu menu ichidagi element ekanligini tekshiradi (nested menu muammosini oldini olish).

</details>

---

### Mashq 3: Custom Event System (O'rta)

**Savol:** `EventEmitter` class yarating:
- `on(event, callback)` — listener qo'shish
- `off(event, callback)` — listener o'chirish
- `emit(event, ...args)` — event trigger qilish
- `once(event, callback)` — faqat bir marta ishlash

<details>
<summary>Javob</summary>

```javascript
class EventEmitter {
  constructor() {
    this.listeners = new Map();
  }

  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(callback);
    return this; // chaining uchun
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
    
    handlers.forEach(handler => handler(...args));
    return true;
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    this.on(event, wrapper);
    return this;
  }
}

// Test:
const emitter = new EventEmitter();

emitter.on("data", (msg) => console.log("A:", msg));
emitter.once("data", (msg) => console.log("B (once):", msg));

emitter.emit("data", "salom");
// A: salom
// B (once): salom

emitter.emit("data", "yana");
// A: yana
// (B ishlamaydi — once edi)
```

**Tushuntirish:** Bu Node.js ning `EventEmitter` ning soddalashtirilgan versiyasi. `once` wrapper funksiya orqali ishlaydi — birinchi chaqiruvda callback ni bajarib, o'zini o'chiradi.

</details>

---

### Mashq 4: Debounced Search (Qiyin)

**Savol:** Input da yozilganda search funksiya chaqirilsin, lekin faqat foydalanuvchi 300ms yozmasa. Event delegation va AbortController ishlatib implement qiling.

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

const searchContainer = document.getElementById("search-container");

const performSearch = debounce(async (query, signal) => {
  if (!query.trim()) {
    showResults([]);
    return;
  }

  try {
    const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal
    });
    const results = await response.json();
    showResults(results);
  } catch (error) {
    if (error.name !== "AbortError") {
      console.error("Search xatosi:", error);
    }
  }
}, 300);

let currentController = null;

// Event delegation — search container ga bitta handler
searchContainer.addEventListener("input", (e) => {
  const input = e.target.closest("input[type='search']");
  if (!input) return;

  // Oldingi request ni bekor qilish
  if (currentController) currentController.abort();
  currentController = new AbortController();

  performSearch(input.value, currentController.signal);
});

function showResults(results) {
  const list = document.getElementById("results");
  list.innerHTML = results.map(r => `<li>${r.title}</li>`).join("");
}
```

**Tushuntirish:**
- `debounce` — foydalanuvchi 300ms to'xtamaguncha funksiya chaqirilmaydi
- `AbortController` — yangi search boshlanganda eski fetch bekor qilinadi
- Event delegation — search container ga bitta handler, input fieldni `closest` bilan topamiz

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Event Model** | `addEventListener` — zamonaviy, ko'p handler, options |
| **Bubbling** | Ichdan tashqariga (default) — deyarli barcha eventlar |
| **Capturing** | Tashqaridan ichga — `{ capture: true }` bilan |
| **Event Flow** | Capturing → Target → Bubbling (3 faza) |
| **stopPropagation** | Boshqa elementlarga tarqalishni to'xtatadi |
| **preventDefault** | Brauzerning standart xulq-atvorini bekor qiladi |
| **Delegation** | Bitta ota handler + `closest()` = memory samarali |
| **target vs currentTarget** | target = bosilgan, currentTarget = handler qo'yilgan |
| **Custom Events** | `new CustomEvent("nom", { detail })` + `dispatchEvent` |
| **Memory** | Listener tozalash! `removeEventListener`, `AbortController`, `{ once: true }` |
| **Passive** | `{ passive: true }` — scroll/touch performance uchun |

> **Keyingi bo'lim:** [Bo'lim 19.5: Browser APIs](19.5-browser-apis.md) — Fetch API, IntersectionObserver, Web Workers va boshqa brauzer imkoniyatlari.

> **Cross-references:** [18-dom.md](18-dom.md) (DOM Manipulation, querySelector, classList), [10-closures.md](10-closures.md) (closure va event handler), [16-memory.md](16-memory.md) (memory leak, GC — listener cleanup), [12-async.md](12-async.md) (Promise, async/await — AbortController), [09-functions.md](09-functions.md) (arrow vs regular function — this in event handlers)
