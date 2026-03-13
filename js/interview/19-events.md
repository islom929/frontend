# Events — Interview Savollari

> **Bo'lim 19** | Event model, bubbling/capturing, delegation, custom events, performance

---

## Savol 1: Event Bubbling va Capturing nima? Tartibini ayting [Junior+]

**Javob:**

DOM da har bir event 3 fazadan o'tadi:
1. **Capturing** — tashqaridan ichga (document → target)
2. **Target** — target elementda
3. **Bubbling** — ichdan tashqariga (target → document)

```javascript
const outer = document.getElementById("outer");
const inner = document.getElementById("inner");

outer.addEventListener("click", () => console.log("outer CAPTURE"), true);
outer.addEventListener("click", () => console.log("outer BUBBLE"));
inner.addEventListener("click", () => console.log("inner CAPTURE"), true);
inner.addEventListener("click", () => console.log("inner BUBBLE"));

// inner bosilganda:
// "outer CAPTURE"  → capturing phase
// "inner CAPTURE"  → target phase (yozilgan tartibda)
// "inner BUBBLE"   → target phase (yozilgan tartibda)
// "outer BUBBLE"   → bubbling phase
```

**Deep Dive:**

Target element da handler'lar **yozilgan tartibda** ishlaydi (capture/bubble ahamiyati yo'q). Ba'zi eventlar bubble qilmaydi: `focus`/`blur` (o'rniga `focusin`/`focusout`), `mouseenter`/`mouseleave`, `load`, `error`.

---

## Savol 2: Output nima? Event flow tartibi [Middle]

**Javob:**

```javascript
document.addEventListener("click", () => console.log("1: document BUBBLE"));
document.addEventListener("click", () => console.log("2: document CAPTURE"), true);
document.body.addEventListener("click", () => console.log("3: body BUBBLE"));
document.body.addEventListener("click", () => console.log("4: body CAPTURE"), true);
button.addEventListener("click", () => console.log("5: button BUBBLE"));
button.addEventListener("click", () => console.log("6: button CAPTURE"), true);

// button bosilganda:
```

**Javob:**

```
2: document CAPTURE
4: body CAPTURE
6: button CAPTURE
5: button BUBBLE
3: body BUBBLE
1: document BUBBLE
```

Capturing (tashqaridan): document → body. Target (yozilgan tartibda): button CAPTURE → BUBBLE. Bubbling (ichdan): body → document.

---

## Savol 3: `stopPropagation()` vs `preventDefault()` farqi nima? [Middle]

**Javob:**

| | `stopPropagation()` | `preventDefault()` |
|---|---|---|
| Nima qiladi | Event tarqalishini to'xtatadi | Brauzer default xulqini to'xtatadi |
| Misol | Parent handler ishlamaydi | Link navigate qilmaydi |
| Event handler | Shu elementdagi handler'lar ishlaydi | Event tarqalishi davom etadi |

```javascript
// stopPropagation — parent'lar eshitmaydi
inner.addEventListener("click", (e) => {
  e.stopPropagation();
  console.log("inner"); // ishlaydi
});
outer.addEventListener("click", () => {
  console.log("outer"); // ISHLAMAYDI
});

// preventDefault — brauzer xulqi to'xtaydi
form.addEventListener("submit", (e) => {
  e.preventDefault(); // sahifa reload bo'lmaydi
});
```

`stopImmediatePropagation()` — shu elementdagi **qolgan** handler'lar ham to'xtaydi:

```javascript
button.addEventListener("click", (e) => {
  e.stopImmediatePropagation();
  console.log("birinchi"); // ishlaydi
});
button.addEventListener("click", () => {
  console.log("ikkinchi"); // ISHLAMAYDI
});
```

---

## Savol 4: Event Delegation nima? Qanday implement qilinadi? [Middle]

**Javob:**

Ota elementga bitta handler qo'yish — bubbling orqali bolalar eventini ushlash:

```javascript
// 1000 ta handler o'rniga — bitta
const list = document.getElementById("list");
list.addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (!item) return;
  if (!list.contains(item)) return; // xavfsizlik
  console.log("Bosildi:", item.textContent);
});
```

| Afzallik | Tushuntirish |
|----------|-------------|
| Memory tejash | 1 handler vs N handler |
| Dinamik elementlar | Keyinroq qo'shilgan ham ishlaydi |
| Sodda cleanup | Bitta handler o'chirish kifoya |

**Deep Dive:**

`closest()` kerak: `e.target` eng ichki element. `<li class="item"><span>Text</span></li>` — span bosilsa `e.target = span`. `closest(".item")` yuqoriga qarab `li` ni topadi.

---

## Savol 5: `event.target` vs `event.currentTarget` farqi [Junior+]

**Javob:**

```javascript
const list = document.getElementById("list");
list.addEventListener("click", function(e) {
  console.log("target:", e.target);         // bosilgan li yoki span
  console.log("currentTarget:", e.currentTarget); // doim list
  console.log("this:", this === e.currentTarget); // true (regular fn)
});
```

| | `target` | `currentTarget` |
|---|---------|-----------------|
| Kim | Event trigger qilgan element | Handler qo'yilgan element |
| Delegation da | Turli elementlar | Doim bir xil |
| Arrow fn da | O'zgarmaydi | `this !== currentTarget` |

---

## Savol 6: Xato toping — delegation da target tekshirish [Middle]

**Javob:**

```javascript
document.body.addEventListener("click", (e) => {
  if (e.target.classList.contains("delete-btn")) {
    const row = e.target.parentElement.parentElement;
    row.remove();
  }
});
```

**Muammo:**

```html
<button class="delete-btn">
  <svg class="icon">...</svg>  <!-- bosilsa target = svg! -->
</button>
```

`e.target` ichki element (svg) bo'lishi mumkin. `parentElement.parentElement` — qattiq coded, DOM o'zgarsa buziladi.

**To'g'ri:**

```javascript
document.body.addEventListener("click", (e) => {
  const deleteBtn = e.target.closest(".delete-btn");
  if (!deleteBtn) return;
  const row = deleteBtn.closest("tr");
  if (row) row.remove();
});
```

---

## Savol 7: Custom Events qanday yaratiladi? [Middle+]

**Javob:**

```javascript
const event = new CustomEvent("user-login", {
  detail: { userId: 42, name: "Ali" },
  bubbles: true,
  cancelable: true
});

document.addEventListener("user-login", (e) => {
  console.log("Login:", e.detail.name);
});

document.dispatchEvent(event);

// Cancel tekshirish
const cancelled = !element.dispatchEvent(event);
if (cancelled) console.log("preventDefault chaqirildi");
```

**Use cases:** component'lar aro kommunikatsiya, plugin tizimlari, Event Bus pattern.

---

## Savol 8: Output nima? Event handler va microtask/macrotask [Middle+]

**Javob:**

```javascript
button.addEventListener("click", () => {
  console.log("1: Click handler");

  Promise.resolve().then(() => console.log("2: Microtask"));

  setTimeout(() => console.log("3: Timeout"), 0);

  console.log("4: Sync after");
});

// Button bosilganda:
```

**Javob:**

```
1: Click handler
4: Sync after
2: Microtask
3: Timeout
```

Handler synchronous kod to'liq ishlaydi, keyin microtask'lar (Promise), keyin macrotask'lar (setTimeout).

---

## Savol 9: `removeEventListener` nima uchun ishlamayapti? [Junior+]

**Javob:**

```javascript
button.addEventListener("click", () => console.log("A"));
button.removeEventListener("click", () => console.log("A"));
// Handler o'chirilmadi! Nima uchun?
```

Har bir `() => {}` yangi funksiya — reference teng emas.

```javascript
// To'g'ri:
const handler = () => console.log("A");
button.addEventListener("click", handler);
button.removeEventListener("click", handler); // bir xil reference

// Yoki:
button.addEventListener("click", handler, { once: true }); // avtomatik o'chiriladi

// Yoki AbortController:
const controller = new AbortController();
button.addEventListener("click", handler, { signal: controller.signal });
controller.abort(); // o'chirildi
```

`capture` qiymati ham mos kelishi kerak:
```javascript
button.addEventListener("click", handler, true);
button.removeEventListener("click", handler, true);  // ishlaydi
button.removeEventListener("click", handler);         // ISHLAMAYDI — default false
```

---

## Savol 10: Passive event listener nima? Nima uchun kerak? [Middle+]

**Javob:**

`{ passive: true }` — brauzerga "preventDefault chaqirilmaydi" kafolati. Scroll/touch uchun performance oshiradi — brauzer handler kutmaydi.

```javascript
// Sekin — brauzer kutadi
document.addEventListener("scroll", handleScroll);

// Tez — brauzer kutmaydi
document.addEventListener("scroll", handleScroll, { passive: true });

// passive da preventDefault = warning, ishlamaydi
```

Chrome 51+ da document-level `touchstart`, `touchmove`, `wheel` — **default passive: true**. Agar `preventDefault` haqiqatan kerak — `{ passive: false }` aniq yozish kerak.

---

## Savol 11: Event listener va memory leak. Qanday oldini olish mumkin? [Middle]

**Javob:**

```javascript
// Memory leak — global listener remove qilinmagan
function setupWidget(container) {
  const bigData = new Array(100000).fill("data");
  const handler = () => console.log(bigData.length);
  window.addEventListener("scroll", handler);
  container.remove();
  // window listener QOLDI + bigData GC tozalay olmaydi
}

// To'g'ri — AbortController
function setupWidget(container) {
  const controller = new AbortController();

  window.addEventListener("scroll", handleScroll, { signal: controller.signal });
  window.addEventListener("resize", handleResize, { signal: controller.signal });
  document.addEventListener("click", handleClick, { signal: controller.signal });

  return () => controller.abort(); // BARCHA bir marta o'chiriladi
}
```

**Qoidalar:**
- Element DOM dan o'chirilsa — unga qo'yilgan listener'lar yo'qoladi
- Global listener'lar (window, document) — alohida o'chirilishi kerak
- SPA da har navigatsiyada cleanup qilish shart

---

## Savol 12: Keyboard events: `e.key` vs `e.code` farqi [Junior+]

**Javob:**

```javascript
document.addEventListener("keydown", (e) => {
  console.log("key:", e.key);   // "a" — tugma BELGI (layout ga bog'liq)
  console.log("code:", e.code); // "KeyA" — tugma JOYI (layout ga bog'liq emas)
});
```

| Property | QWERTY da "A" | AZERTY da "A" |
|----------|---------------|---------------|
| `e.key` | "a" | "q" |
| `e.code` | "KeyA" | "KeyA" |

```javascript
// Shortcut uchun key ishlatiladi
if (e.ctrlKey && e.key === "s") {
  e.preventDefault();
  saveDocument();
}

// Game uchun code ishlatiladi (WASD)
if (e.code === "KeyW") moveUp();
```

`keypress` — **deprecated**, ishlatmang. `keydown` + `keyup` yetarli.

---

## Savol 13: Touch va Pointer Events farqi [Senior]

**Javob:**

```javascript
// Touch events — faqat sensorli ekranlar
element.addEventListener("touchstart", (e) => {
  const touch = e.touches[0];
  console.log(touch.clientX, touch.clientY);
});

// Pointer events — universal (mouse + touch + pen)
element.addEventListener("pointerdown", (e) => {
  console.log("Type:", e.pointerType); // "mouse", "touch", "pen"
  console.log("Pressure:", e.pressure); // 0..1
});
```

| Touch Event | Pointer Event | Mouse Event |
|-------------|---------------|-------------|
| `touchstart` | `pointerdown` | `mousedown` |
| `touchmove` | `pointermove` | `mousemove` |
| `touchend` | `pointerup` | `mouseup` |

Mobile da event tartibi: `pointerdown → touchstart → pointerup → touchend → mousedown → mouseup → click`.

**Tavsiya:** Pointer Events ishlatish — barcha qurilmalar uchun bitta API.

---

## Savol 14: Coding: Debounced search handler [Middle+]

**Javob:**

```javascript
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

let controller = null;

const handleSearch = debounce(async (query) => {
  controller?.abort();
  controller = new AbortController();

  if (query.length < 2) return;

  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal: controller.signal
    });
    const data = await res.json();
    renderResults(data);
  } catch (e) {
    if (e.name !== "AbortError") console.error(e);
  }
}, 300);

searchInput.addEventListener("input", (e) => {
  handleSearch(e.target.value);
});
```

**Kalit nuqtalar:**
- `debounce` — 300ms kutish
- `AbortController` — eski request cancel
- `encodeURIComponent` — URL xavfsizligi

---

## Savol 15: addEventListener options to'liq ro'yxatini ayting [Senior]

**Javob:**

```javascript
element.addEventListener("click", handler, {
  capture: false,              // capturing phase da
  once: true,                  // bir marta, avtomatik remove
  passive: true,               // preventDefault chaqirilmaslik kafolati
  signal: controller.signal    // AbortController bilan remove
});
```

| Option | Default | Vazifasi |
|--------|---------|----------|
| `capture` | `false` | Capturing fazada ushlash |
| `once` | `false` | Bir marta ishlab o'chirilish |
| `passive` | `false`* | preventDefault bloklash |
| `signal` | — | AbortController bilan cleanup |

*`touchstart`/`touchmove`/`wheel` da document/window uchun Chrome default `passive: true` qilgan.

---

## Savol 16: Xato toping — `var` bilan loop ichida handler [Middle]

**Javob:**

```javascript
const buttons = document.querySelectorAll("button");
for (var i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => {
    console.log("Button:", i);
  });
}
// Barcha buttonlar oxirgi qiymat ko'rsatadi!
```

`var` function scope — bitta `i` barcha handler'larga umumiy. Loop tugaganda `i` = `buttons.length`.

**To'g'ri:**

```javascript
// let — block scope
for (let i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => {
    console.log("Button:", i);
  });
}

// Yoki delegation
container.addEventListener("click", (e) => {
  const btn = e.target.closest("button");
  if (btn) console.log("Button:", btn.dataset.index);
});
```

---

## Savol 17: `return false` addEventListener da ishlaydimi? [Junior+]

**Javob:**

```javascript
// addEventListener da return false HECH NIMA QILMAYDI!
link.addEventListener("click", () => {
  return false; // hech narsa to'xtatmaydi — link navigate qiladi
});

// Faqat onclick PROPERTY da ishlaydi
link.onclick = () => { return false; }; // preventDefault + stopPropagation
```

**To'g'ri:**
```javascript
link.addEventListener("click", (e) => {
  e.preventDefault();    // navigatsiya to'xtaydi
  e.stopPropagation();   // bubbling to'xtaydi
});
```

---

## Savol 18: Coding: EventEmitter implement qiling [Middle+]

**Javob:**

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

// Test:
const emitter = new EventEmitter();
emitter.on("data", (msg) => console.log("A:", msg));
emitter.once("data", (msg) => console.log("B:", msg));

emitter.emit("data", "salom");
// A: salom
// B: salom

emitter.emit("data", "yana");
// A: yana
// (B ishlamaydi — once)
```

---

## Savol 19: SPA da event listener cleanup pattern [Senior]

**Javob:**

```javascript
// Har navigatsiyada yangi listener qo'shiladi — LEAK
function initPage() {
  window.addEventListener("scroll", handleScroll); // har safar +1 listener!
}

// To'g'ri — AbortController lifecycle pattern
class PageComponent {
  constructor() {
    this.controller = new AbortController();
  }

  mount() {
    const { signal } = this.controller;

    window.addEventListener("scroll", this.handleScroll, { signal });
    window.addEventListener("resize", this.handleResize, { signal });

    document.addEventListener("keydown", (e) => {
      if (e.key === "Escape") this.close();
    }, { signal });
  }

  unmount() {
    this.controller.abort(); // barcha listener'lar o'chiriladi
  }

  handleScroll = () => { /* ... */ };
  handleResize = () => { /* ... */ };
}

// Ishlatish:
const page = new PageComponent();
page.mount();

// Boshqa sahifaga o'tganda:
page.unmount(); // 3 ta listener bir marta o'chirildi
```

**Deep Dive:**

React da `useEffect` cleanup, Vue da `onUnmounted` — ichida xuddi shu pattern. AbortController zamonaviy va xavfsiz — `removeEventListener` da reference saqlash muammosi yo'q.

---

## Savol 20: Coding: Event delegation bilan Tab Component [Middle]

**Javob:**

```javascript
function createTabs(container) {
  const tabs = container.querySelector(".tabs");
  const content = container.querySelector(".tab-content");

  tabs.addEventListener("click", (e) => {
    const tab = e.target.closest("[data-tab]");
    if (!tab || !tabs.contains(tab)) return;

    // Active class
    tabs.querySelectorAll("[data-tab]").forEach(t => {
      t.classList.remove("active");
    });
    tab.classList.add("active");

    // Content o'zgartirish
    const panels = content.querySelectorAll("[data-panel]");
    panels.forEach(p => {
      p.hidden = p.dataset.panel !== tab.dataset.tab;
    });

    // Custom event
    container.dispatchEvent(new CustomEvent("tab-change", {
      detail: { tab: tab.dataset.tab },
      bubbles: true
    }));
  });
}

// Tinglash
document.addEventListener("tab-change", (e) => {
  console.log("Tab o'zgardi:", e.detail.tab);
});
```

**Kalit nuqtalar:**
- Delegation — bitta handler, dinamik tab'lar ham ishlaydi
- `closest("[data-tab]")` — nested elementlar uchun xavfsiz
- `contains()` — faqat shu tabs ichidagi elementlar
- Custom event — boshqa component'lar xabardor bo'ladi

---

> **Keyingi bo'lim:** [Browser APIs — Interview Savollari](19.5-browser-apis.md)
