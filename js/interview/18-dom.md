# DOM Manipulation — Interview Savollari

> **Bo'lim 18** | DOM tree, node types, traversal, manipulation, performance, Virtual DOM

---

## Savol 1: DOM nima? HTML bilan farqi nima? [Junior+]

**Javob:**

DOM (Document Object Model) — brauzer HTML hujjatni parse qilib xotirada yaratadigan **daraxt strukturasi**. HTML — matn fayl (source code), DOM — tirik JavaScript object tree.

```javascript
// HTML: <table><tr><td>Test</td></tr></table>
// DOM:  table → tbody → tr → td
// Brauzer avtomatik <tbody> qo'shadi — HTML da yo'q, DOM da bor

console.log(document.querySelector("table").children[0].tagName);
// "TBODY"
```

```javascript
console.log(document.documentElement); // <html>
console.log(document.head);            // <head>
console.log(document.body);            // <body>
```

**Deep Dive:**

Rendering pipeline: `HTML → Parser → DOM Tree → CSSOM → Render Tree → Layout → Paint → Composite`. JavaScript DOM ni o'zgartirsa Layout/Paint qayta ishlaydi (reflow/repaint). Parser incremental — hujjat to'liq kutilmaydi, kelgan qismi bilan DOM qurishni boshlaydi.

---

## Savol 2: `childNodes` va `children` farqi nima? Output nima? [Junior+]

**Javob:**

```html
<ul id="list">
  <li>Birinchi</li>
  <li>Ikkinchi</li>
  <li>Uchinchi</li>
</ul>
```

```javascript
const list = document.getElementById("list");

console.log(list.firstChild.nodeName);        // ?
console.log(list.firstElementChild.nodeName); // ?
console.log(list.childNodes.length);          // ?
console.log(list.children.length);            // ?
```

**Javoblar:**

```
"#text"  // firstChild = whitespace text node (yangi qator)
"LI"     // firstElementChild = birinchi element
7        // childNodes: [text, li, text, li, text, li, text]
3        // children: [li, li, li] — faqat elementlar
```

`childNodes` — BARCHA node turlari (text, comment ham). `children` — faqat Element node'lar.

**Deep Dive:**

Node types: Element (1), Text (3), Comment (8), Document (9), DocumentFragment (11). Node inheritance chain: `EventTarget → Node → Element → HTMLElement → HTMLDivElement`.

---

## Savol 3: `querySelector` va `getElementById` farqi nima? [Junior+]

**Javob:**

| Usul | Qaytaradi | Live? | Tezlik |
|------|-----------|-------|--------|
| `getElementById("id")` | Element \| null | — | eng tez (hash map, O(1)) |
| `getElementsByClassName("cls")` | HTMLCollection | Live | tez |
| `querySelector(".css")` | Element \| null | — | o'rta |
| `querySelectorAll(".css")` | NodeList | Static | o'rta |

```javascript
const live = document.getElementsByClassName("item");   // LIVE
const static_ = document.querySelectorAll(".item");     // STATIC

document.body.appendChild(Object.assign(
  document.createElement("div"), { className: "item" }
));

console.log(live.length);    // 4 — avtomatik yangilandi
console.log(static_.length); // 3 — o'zgarmadi (snapshot)
```

**Deep Dive:**

`getElementById` eng tez — brauzer ID lar uchun hash map saqlaydi (O(1)). `querySelector` CSS selector parser orqali DOM traverse qiladi — O(n). HTMLCollection da `forEach` yo'q — `Array.from()` kerak.

---

## Savol 4: `textContent`, `innerHTML`, `innerText` farqi nima? [Junior+]

**Javob:**

```html
<div id="demo">
  <span>Ko'rinadigan</span>
  <span style="display:none">Yashirin</span>
</div>
```

```javascript
const el = document.getElementById("demo");

console.log(el.textContent); // "Ko'rinadigan Yashirin" — barchasi
console.log(el.innerText);   // "Ko'rinadigan" — faqat ko'rinadigan
console.log(el.innerHTML);   // "<span>Ko'rinadigan</span>..."
```

| Xususiyat | HTML parse? | Yashirin | Performance | XSS xavf |
|-----------|-------------|----------|-------------|-----------|
| `textContent` | Yo'q | Ko'rsatadi | Tez | Xavfsiz |
| `innerText` | Yo'q | Ko'rsatmaydi | Sekin (reflow) | Xavfsiz |
| `innerHTML` | Ha | Ko'rsatadi | O'rta | XSS xavfi |

```javascript
// XSS hujum
const userInput = '<img src=x onerror="alert(document.cookie)">';
element.innerHTML = userInput;    // XSS!
element.textContent = userInput;  // Xavfsiz — oddiy matn
```

**Deep Dive:**

`innerText` sekin chunki CSS `display`, `visibility` hisoblashi kerak — reflow trigger qiladi. `textContent` faqat DOM tree'dagi text node'larni yig'adi.

---

## Savol 5: `closest()` va `matches()` nima? [Middle]

**Javob:**

- `closest(selector)` — elementdan **yuqoriga** qarab eng yaqin mos ota elementni topadi (o'zini ham tekshiradi)
- `matches(selector)` — element CSS selector'ga mos keladimi (boolean)

```javascript
const link = document.querySelector(".nav-item a");
link.closest("nav");          // eng yaqin <nav> ota
link.closest(".absent");      // null — topilmadi

const div = document.querySelector("div");
div.matches(".active");       // true yoki false
```

Event delegation da `closest()` muhim:
```javascript
// Xato — nested element bosilsa target noto'g'ri
list.addEventListener("click", (e) => {
  if (e.target.classList.contains("item")) { /* ... */ }
  // <li class="item"><span>Text</span></li>
  // span bosilsa target = span, li emas!
});

// To'g'ri
list.addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (!item) return;
  console.log(item.textContent);
});
```

---

## Savol 6: `data-*` attributes va `dataset` qanday ishlaydi? [Junior+]

**Javob:**

```html
<div id="card" data-user-id="42" data-user-name="Ali" data-is-active="true">
</div>
```

```javascript
const card = document.getElementById("card");

// O'qish — camelCase
console.log(card.dataset.userId);   // "42" (data-user-id → userId)
console.log(card.dataset.userName); // "Ali"

// Yozish
card.dataset.role = "admin"; // HTML: data-role="admin"

// O'chirish
delete card.dataset.isActive;

// Qiymatlar DOIM string!
const id = Number(card.dataset.userId); // "42" → 42
```

**Attribute vs Property:**
```javascript
const input = document.querySelector("input");
input.setAttribute("value", "boshlang'ich"); // HTML attribute
input.value = "hozirgi";                      // DOM property

console.log(input.getAttribute("value")); // "boshlang'ich" — o'zgarmadi
console.log(input.value);                 // "hozirgi" — JS o'zgartirgan
```

**Deep Dive:**

Attribute — HTML dagi boshlang'ich qiymat. Property — DOM object dagi hozirgi qiymat. Ba'zilari sinxron (id, href), ba'zilari emas (value). CSS da `[data-state="active"] { color: green; }` ishlatiladi.

---

## Savol 7: `classList` API ni tushuntiring [Junior+]

**Javob:**

```javascript
const btn = document.querySelector("button");

// className — butun string, oldingilarni o'chirishi mumkin
btn.className = "btn primary";

// classList — xavfsiz
btn.classList.add("active");
btn.classList.add("btn", "primary", "large"); // bir nechta
btn.classList.remove("large");
btn.classList.toggle("active");               // bor → o'chiradi, yo'q → qo'shadi
btn.classList.toggle("active", true);         // doim qo'shadi (force)
btn.classList.contains("active");             // boolean
btn.classList.replace("btn-primary", "btn-secondary");
```

---

## Savol 8: Xato toping — Live collection bilan loop [Middle]

**Javob:**

```javascript
// Bu kod barcha elementlarni o'chirmaydi — nima uchun?
const items = document.getElementsByClassName("item"); // LIVE!
for (let i = 0; i < items.length; i++) {
  items[i].remove();
}
```

**Xato:** `getElementsByClassName` live HTMLCollection qaytaradi. `items[0]` o'chirilganda `items[1]` endi `items[0]` bo'ladi — index siljiydi. Faqat **yarmi** o'chiriladi.

```javascript
// Variant 1: orqadan
for (let i = items.length - 1; i >= 0; i--) {
  items[i].remove();
}

// Variant 2: while
while (items.length > 0) {
  items[0].remove();
}

// Variant 3: Static list (eng yaxshi)
document.querySelectorAll(".item").forEach(el => el.remove());

// Variant 4: replaceChildren()
container.replaceChildren(); // barcha bolalarni o'chiradi
```

---

## Savol 9: Reflow va Repaint nima? Layout Thrashing qanday bo'ladi? [Middle+]

**Javob:**

- **Reflow (Layout)** — element o'lchami/pozitsiyasi qayta hisoblanadi. Qimmat operatsiya.
- **Repaint** — faqat ko'rinish (rang, shadow) qayta chiziladi. Arzonroq.

```javascript
// Layout Thrashing — har safar O'QISH + YOZISH = har safar reflow!
const items = document.querySelectorAll(".item");
items.forEach(item => {
  const width = item.offsetWidth;       // O'QISH → forced reflow!
  item.style.width = width * 2 + "px";  // YOZISH → layout invalidate
});

// To'g'ri — avval hammasi O'QI, keyin hammasi YOZ
const widths = [...items].map(item => item.offsetWidth); // bitta reflow
items.forEach((item, i) => {
  item.style.width = widths[i] * 2 + "px"; // bitta batch write
});
```

Reflow trigger qiluvchi property'lar:
```javascript
element.offsetWidth;             // reflow
element.offsetHeight;            // reflow
element.getBoundingClientRect(); // reflow
getComputedStyle(element);       // reflow
```

Reflow-free:
```javascript
// transform va opacity GPU da ishlaydi
element.style.transform = "translateX(100px)"; // Layout ta'sir qilmaydi
element.style.opacity = "0.5";                  // Paint ta'sir qilmaydi
```

**Deep Dive:**

Brauzer style yozishlarni batch qiladi, lekin `offsetWidth`, `clientHeight`, `getBoundingClientRect()` kabi o'qish operatsiyalari **forced reflow** qiladi — to'g'ri qiymat qaytarishi kerak. Best practice: `classList.add()` (bitta reflow) yoki `requestAnimationFrame()`.

---

## Savol 10: DocumentFragment nima? Nima uchun kerak? [Middle]

**Javob:**

DocumentFragment — DOM ning yengil virtual container'i. Unga element qo'shish reflow trigger qilmaydi.

```javascript
// 1000 ta reflow
const list = document.getElementById("list");
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list.appendChild(li); // har safar reflow
}

// 1 ta reflow — DocumentFragment bilan
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li); // fragment ga — reflow yo'q
}
list.appendChild(fragment); // BITTA reflow
// fragment bo'sh qoladi — bolalari list ga ko'chdi
```

```javascript
// <template> elementi ham DocumentFragment
const template = document.getElementById("item-template");
const clone = template.content.cloneNode(true); // DocumentFragment
```

**Deep Dive:**

Fragment DOM ga qo'shilganda "eritiladi" — faqat bolalari qo'shiladi, fragment o'zi DOM da qolmaydi. `innerHTML` ham tez, lekin XSS xavfi bor va mavjud event listener'lar yo'qoladi.

---

## Savol 11: `requestAnimationFrame` nima? `setInterval` dan farqi? [Middle]

**Javob:**

`requestAnimationFrame(callback)` — brauzerga keyingi repaint oldidan callback bajarishni aytadi. Animatsiyalar uchun optimal.

```javascript
// setInterval bilan — frame drop, battery drain
setInterval(() => {
  element.style.left = parseInt(element.style.left) + 1 + "px";
}, 16); // ~60fps lekin KAFOLAT YO'Q

// requestAnimationFrame bilan — brauzer sync
let x = 0;
function animate() {
  element.style.transform = `translateX(${x++}px)`;
  if (x < 300) {
    requestAnimationFrame(animate);
  }
}
requestAnimationFrame(animate);
```

```javascript
// DOM batch update
requestAnimationFrame(() => {
  el1.style.width = "100px";
  el2.style.height = "200px";
  el3.classList.add("active");
  // barcha o'zgarishlar bitta frame da
});
```

**Deep Dive:**

`rAF` rendering pipeline boshida ishlaydi — Style/Layout/Paint oldidan. `setInterval` macrotask queue'da — frame timing bilan sync emas. `rAF` tab background'da to'xtaydi (battery tejash), `setInterval` davom etadi. `cancelAnimationFrame(id)` bilan bekor qilish mumkin.

---

## Savol 12: `cloneNode` qanday ishlaydi? Event listener'lar nusxalanadimi? [Middle]

**Javob:**

```javascript
const original = document.querySelector(".template");

// Shallow clone — faqat element o'zi
const shallow = original.cloneNode(false);
console.log(shallow.children.length); // 0

// Deep clone — element + barcha bolalari
const deep = original.cloneNode(true);

// cloneNode event listener'larni NUSXALAMAYDI!
// Faqat HTML attribute (onclick="...") nusxalanadi
// addEventListener bilan qo'shilganlar — YO'Q
```

```javascript
// outerHTML — element ni o'zini ham qaytaradi
console.log(el.outerHTML);  // "<div class='card'>Content</div>"
console.log(el.innerHTML);  // "Content"

// outerHTML ga yozish — element almashadi
el.outerHTML = "<section>Yangi</section>";
// el variable hali eski element ga ishora qilmoqda (detached)
```

---

## Savol 13: Output nima? `innerHTML +=` muammosi [Middle]

**Javob:**

```javascript
const list = document.getElementById("list");
list.innerHTML = '<li id="first">Birinchi</li>';

const firstLi = document.getElementById("first");
firstLi.addEventListener("click", () => console.log("Bosildi!"));

// firstLi click — ishlaydi
list.innerHTML += '<li>Ikkinchi</li>';
// firstLi click — nima bo'ladi?
```

**Javob:** Click handler **ISHLAMAYDI**.

`innerHTML +=` aslida:
1. Eski innerHTML o'qiladi (string)
2. Yangi string qo'shiladi
3. **BUTUN** ichidagi DOM yo'q qilinadi
4. Qayta DOM yaratiladi — lekin event listener'lar yo'qoldi!

```javascript
// firstLi endi DETACHED element ga ishora qilmoqda

// To'g'ri usul:
list.insertAdjacentHTML("beforeend", "<li>Ikkinchi</li>");
// Mavjud elementlar va listener'lar saqlanadi
```

---

## Savol 14: Xato toping — XSS xavfli kod [Middle+]

**Javob:**

```javascript
function showProfile(user) {
  const container = document.getElementById("profile");
  container.innerHTML = `
    <h2>${user.name}</h2>
    <p>Bio: ${user.bio}</p>
    <a href="${user.website}">Website</a>
  `;
}

showProfile({
  name: '<script>alert("hacked")</script>',
  bio: '<img src=x onerror="document.cookie">',
  website: 'javascript:alert(1)'
});
```

**3 ta XSS xavfi:**
1. `innerHTML` bilan user input — script/img injection
2. `href` ga user input — `javascript:` protocol
3. Hech qanday sanitization yo'q

**To'g'ri:**
```javascript
function showProfileSafe(user) {
  const container = document.getElementById("profile");

  const h2 = document.createElement("h2");
  h2.textContent = user.name; // XSS xavfsiz

  const p = document.createElement("p");
  p.textContent = `Bio: ${user.bio}`;

  const a = document.createElement("a");
  const url = new URL(user.website, location.origin);
  if (url.protocol === "http:" || url.protocol === "https:") {
    a.href = url.href; // faqat http/https
  }
  a.textContent = "Website";

  container.replaceChildren(h2, p, a);
}
```

---

## Savol 15: Coding: 10,000 elementni performance-optimal qo'shing [Senior]

**Javob:**

```javascript
// === Usul 1: To'g'ridan-to'g'ri appendChild ===
console.time("direct");
const list1 = document.getElementById("list1");
for (let i = 0; i < 10000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list1.appendChild(li);
}
console.timeEnd("direct"); // ~50-200ms

// === Usul 2: DocumentFragment ===
console.time("fragment");
const list2 = document.getElementById("list2");
const fragment = document.createDocumentFragment();
for (let i = 0; i < 10000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);
}
list2.appendChild(fragment);
console.timeEnd("fragment"); // ~20-80ms

// === Usul 3: innerHTML ===
console.time("innerHTML");
const list3 = document.getElementById("list3");
list3.innerHTML = Array.from({ length: 10000 },
  (_, i) => `<li>Item ${i}</li>`
).join("");
console.timeEnd("innerHTML"); // ~10-40ms
// XSS xavfi, listener'lar yo'qoladi

// === Usul 4: Off-screen element ===
console.time("offscreen");
const list4 = document.getElementById("list4");
const clone = list4.cloneNode(false); // bo'sh nusxa (DOM da emas)
for (let i = 0; i < 10000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  clone.appendChild(li); // off-screen — reflow yo'q
}
list4.parentNode.replaceChild(clone, list4); // bitta reflow
console.timeEnd("offscreen"); // ~15-50ms
```

**Deep Dive:**

Usul 4 eng samarali: `cloneNode(false)` DOM'dan uzilgan element yaratadi — reflow/repaint trigger qilmaydi. `replaceChild` bitta atomic operatsiya. Virtual scrolling (faqat ko'rinadigan elementlarni render qilish) — 10,000+ element uchun eng to'g'ri yechim.

---

## Savol 16: Virtual DOM nima? DocumentFragment dan farqi? [Senior]

**Javob:**

Virtual DOM — haqiqiy DOM ning JavaScript object sifatidagi engil nusxasi. React, Vue kabi framework'lar ishlatadi.

| | DocumentFragment | Virtual DOM |
|-|-----------------|-------------|
| Nima | Haqiqiy DOM node | JS object |
| Diffing | Yo'q | Bor (eski vs yangi taqqoslash) |
| Framework | Native API | React, Vue |
| Maqsad | Batch insert | Minimal DOM update |

```javascript
// Virtual DOM — oddiy JS object
const vdom = {
  tag: "div",
  props: { className: "app" },
  children: [
    { tag: "h1", props: {}, children: ["Title"] },
    { tag: "p", props: { id: "desc" }, children: ["Text"] }
  ]
};

// Jarayon:
// 1. State o'zgardi → yangi VDOM yaratiladi
// 2. Eski va yangi VDOM solishtiriladi (diffing)
// 3. Faqat FARQLAR haqiqiy DOM ga qo'llaniladi (reconciliation)
```

```
Haqiqiy DOM:               Virtual DOM:
O'zgarish 1 → Reflow      O'zgarish 1 ┐
O'zgarish 2 → Reflow      O'zgarish 2 ├→ Diff → 1 DOM update → 1 Reflow
O'zgarish 3 → Reflow      O'zgarish 3 ┘
```

**Deep Dive:**

Virtual DOM doim ham tezroq emas. Kichik o'zgarishlar uchun to'g'ridan-to'g'ri manipulation tezroq (VDOM diff/patch overhead). VDOM ning kuchi — developer experience (declarative UI) va katta UI'lar uchun optimizatsiya. Svelte compile-time da hal qiladi — runtime VDOM ishlatmaydi.

---

## Savol 17: `insertAdjacentHTML` qanday ishlaydi? 4 ta pozitsiyani ayting [Middle]

**Javob:**

```javascript
/*
  <!-- beforebegin -->
  <div class="container">
    <!-- afterbegin -->
    mavjud content
    <!-- beforeend -->
  </div>
  <!-- afterend -->
*/

const el = document.querySelector(".container");

el.insertAdjacentHTML("beforebegin", "<p>Oldin</p>");   // el dan oldin
el.insertAdjacentHTML("afterbegin", "<p>Boshiga</p>");  // el ichiga, boshiga
el.insertAdjacentHTML("beforeend", "<p>Oxiriga</p>");   // el ichiga, oxiriga
el.insertAdjacentHTML("afterend", "<p>Keyin</p>");      // el dan keyin
```

```javascript
// insertAdjacentElement — tayyor element qo'shish
el.insertAdjacentElement("beforeend", document.createElement("div"));

// insertAdjacentText — text node qo'shish (XSS xavfsiz)
el.insertAdjacentText("beforeend", "Xavfsiz matn");
```

**Deep Dive:**

`insertAdjacentHTML` mavjud elementlarni buzmasdan HTML qo'shadi — `innerHTML +=` dan farqli. Lekin user input bilan XSS xavfi bor. User input uchun `insertAdjacentText` xavfsiz variant.

---

## Savol 18: `getComputedStyle` va `style` farqi nima? [Middle]

**Javob:**

```javascript
const box = document.querySelector(".box");

// style — faqat INLINE stillarni o'qiydi/yozadi
box.style.backgroundColor = "blue";     // camelCase
box.style.setProperty("--main-color", "#3498db"); // CSS variable

// CSS faylda: div { color: red; }
console.log(box.style.color);            // "" — inline yo'q
console.log(getComputedStyle(box).color); // "rgb(255, 0, 0)" — CSS dan

// getComputedStyle — hisoblangan (final) qiymat, FAQAT O'QISH
const computed = getComputedStyle(box);
console.log(computed.width);           // "300px"
console.log(computed.backgroundColor); // "rgb(52, 152, 219)"

// Pseudo-element
const before = getComputedStyle(box, "::before");
console.log(before.content);
```

**Deep Dive:**

`getComputedStyle` CSS cascade final natijasini qaytaradi. `style` faqat inline. Ko'p stil o'zgartirish uchun `classList.add()` (bitta reflow) afzal — har bir `style.property` ham batch qilinsa ham, class orqali aniqroq va tezroq.

---

## Savol 19: Element yaratish va qo'shish usullarini solishtiring [Junior+]

**Javob:**

```javascript
const container = document.getElementById("container");

// appendChild — oxiriga, faqat 1 node, eski usul
container.appendChild(document.createElement("div"));

// append — oxiriga, ko'p node + string
container.append(
  document.createElement("span"),
  "Oddiy matn"
);

// prepend — boshiga
container.prepend(document.createElement("h2"));

// before / after — yoniga (tashqarida)
const target = document.querySelector(".target");
target.after(document.createElement("p"));
target.before(document.createElement("p"));

// replaceWith — o'rniga
const old = document.querySelector(".old");
old.replaceWith(document.createElement("div"));

// replaceChildren — barcha bolalarni almashtirish
container.replaceChildren(
  document.createElement("h1"),
  document.createElement("p")
);
```

| Usul | Argument | String? | Return |
|------|----------|---------|--------|
| `appendChild` | 1 Node | Yo'q | Node |
| `append` | Ko'p Node/string | Ha | undefined |
| `prepend` | Ko'p Node/string | Ha | undefined |
| `replaceChildren` | Ko'p Node/string | Ha | undefined |

---

## Savol 20: Coding: Todo list — DOM manipulation bilan [Middle+]

**Javob:**

```javascript
function createTodoApp(containerId) {
  const container = document.getElementById(containerId);

  container.innerHTML = `
    <input type="text" id="todo-input" placeholder="Yangi vazifa...">
    <button id="add-btn">Qo'shish</button>
    <ul id="todo-list"></ul>
    <p id="counter">0 ta vazifa</p>
  `;

  const input = container.querySelector("#todo-input");
  const addBtn = container.querySelector("#add-btn");
  const list = container.querySelector("#todo-list");
  const counter = container.querySelector("#counter");

  function updateCounter() {
    const total = list.children.length;
    const done = list.querySelectorAll(".done").length;
    counter.textContent = `${done}/${total} ta vazifa bajarildi`;
  }

  function addTodo(text) {
    if (!text.trim()) return;

    const li = document.createElement("li");
    li.className = "todo-item";

    const span = document.createElement("span");
    span.textContent = text; // textContent — XSS xavfsiz

    const doneBtn = document.createElement("button");
    doneBtn.textContent = "Done";
    doneBtn.className = "done-btn";

    const deleteBtn = document.createElement("button");
    deleteBtn.textContent = "Delete";
    deleteBtn.className = "delete-btn";

    li.append(span, doneBtn, deleteBtn);
    list.appendChild(li);
    updateCounter();
  }

  // Event delegation
  list.addEventListener("click", (e) => {
    const li = e.target.closest(".todo-item");
    if (!li) return;

    if (e.target.closest(".delete-btn")) {
      li.remove();
      updateCounter();
      return;
    }

    if (e.target.closest(".done-btn")) {
      li.classList.toggle("done");
      updateCounter();
    }
  });

  addBtn.addEventListener("click", () => {
    addTodo(input.value);
    input.value = "";
    input.focus();
  });

  input.addEventListener("keydown", (e) => {
    if (e.key === "Enter") {
      addTodo(input.value);
      input.value = "";
    }
  });
}
```

**Kalit nuqtalar:**
- `textContent` — XSS xavfsiz
- Event delegation — bitta handler, dinamik elementlar uchun ishlaydi
- `classList.toggle()` — toggle pattern
- `closest()` — nested elementlarda to'g'ri target topish

---

> **Keyingi bo'lim:** [Events — Interview Savollari](19-events.md)
