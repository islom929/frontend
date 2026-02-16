# Bo'lim 18: DOM Manipulation

> DOM — brauzer HTML ni JavaScript orqali boshqarish imkonini beradigan dasturlash interfeysi.

---

## Mundarija

- [DOM Nima?](#dom-nima)
- [Node Types](#node-types)
- [DOM Traversal](#dom-traversal)
- [Element Tanlash (Selecting)](#element-tanlash-selecting)
- [Element Yaratish va Qo'shish](#element-yaratish-va-qoshish)
- [Element O'chirish](#element-ochirish)
- [Content O'zgartirish](#content-ozgartirish)
- [Attributes va Dataset](#attributes-va-dataset)
- [ClassList API](#classlist-api)
- [Style Manipulation](#style-manipulation)
- [Performance: Reflow va Repaint](#performance-reflow-va-repaint)
- [DocumentFragment](#documentfragment)
- [Virtual DOM Tushunchasi](#virtual-dom-tushunchasi)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## DOM Nima?

### Nazariya

DOM (Document Object Model) — brauzer HTML hujjatni parse qilib, undan yaratadigan **daraxt (tree) strukturasi**. Bu daraxtdagi har bir element, matn, komment — barchasi **node** deb ataladi. JavaScript orqali biz bu daraxtni o'qiymiz, o'zgartiramiz, yangi elementlar qo'shamiz, eskilarini o'chiramiz.

DOM **HTML ning o'zi emas**. HTML — bu matn fayl, DOM — bu brauzer xotirasidagi tirik (live) obyekt. Brauzer HTML ni parse qilib DOM yaratadi, keyin JavaScript va CSS shu DOM bilan ishlaydi.

```
HTML Source Code → Parser → DOM Tree → Render Tree → Layout → Paint → Screen
```

### Under the Hood

Brauzer HTML parse qilganda **incremental parsing** ishlatadi — hujjatni to'liq kutmaydi, kelgan qismlari bilan DOM qurishni boshlaydi. DOM daraxt `document` node dan boshlanadi:

```
document
└── html
    ├── head
    │   ├── title
    │   │   └── "Sahifa nomi"  (Text node)
    │   └── meta
    └── body
        ├── h1
        │   └── "Salom"  (Text node)
        ├── p
        │   └── "Matn..."  (Text node)
        └── div
            ├── span
            └── a
```

### Kod Misollari

```javascript
// DOM daraxtiga kirish
console.log(document);                  // butun hujjat
console.log(document.documentElement);  // <html>
console.log(document.head);             // <head>
console.log(document.body);             // <body>

// DOM tirik (live) — o'zgarishlar darhol aks etadi
document.body.style.background = "lightblue"; // sahifa rangi o'zgardi
```

```javascript
// DOM va HTML farqi — oddiy misol
// HTML:  <table><tr><td>Test</td></tr></table>
// DOM:   brauzer avtomatik <tbody> qo'shadi!
// Shuning uchun DOM da: table → tbody → tr → td
// HTML source da tbody yo'q, lekin DOM da bor
```

---

## Node Types

### Nazariya

DOM daraxtidagi har bir element — **node**. Har bir node ning turi (type) bor va har biri boshqacha xususiyatlarga ega.

### Asosiy Node Turlari

| Node Type | nodeType | Tavsif | Misol |
|-----------|----------|--------|-------|
| `Element` | 1 | HTML element | `<div>`, `<p>`, `<span>` |
| `Text` | 3 | Matn content | `"Salom dunyo"` |
| `Comment` | 8 | HTML izoh | `<!-- izoh -->` |
| `Document` | 9 | Butun hujjat | `document` |
| `DocumentType` | 10 | DOCTYPE | `<!DOCTYPE html>` |
| `DocumentFragment` | 11 | Virtual container | `document.createDocumentFragment()` |

### Kod Misollari

```javascript
const div = document.createElement("div");
const text = document.createTextNode("Salom");
const comment = document.createComment("Bu izoh");

console.log(div.nodeType);     // 1 (Element)
console.log(text.nodeType);    // 3 (Text)
console.log(comment.nodeType); // 8 (Comment)
console.log(document.nodeType); // 9 (Document)
```

```javascript
// Node vs Element — farqi
const parent = document.getElementById("container");

// childNodes — BARCHA node lar (text, comment ham)
console.log(parent.childNodes); // NodeList [text, element, text, comment, ...]

// children — faqat Element node lar
console.log(parent.children); // HTMLCollection [element, element, ...]
```

### Under the Hood

Barcha node'lar `Node` class'idan meros oladi. Element'lar esa qo'shimcha `Element` class'idan:

```
EventTarget
  └── Node
      ├── Document
      ├── DocumentFragment
      ├── Element
      │   └── HTMLElement
      │       ├── HTMLDivElement
      │       ├── HTMLParagraphElement
      │       ├── HTMLInputElement
      │       └── ...
      ├── Text
      └── Comment
```

```javascript
const div = document.createElement("div");
console.log(div instanceof HTMLDivElement); // true
console.log(div instanceof HTMLElement);    // true
console.log(div instanceof Element);       // true
console.log(div instanceof Node);          // true
console.log(div instanceof EventTarget);   // true
```

---

## DOM Traversal

### Nazariya

DOM traversal — daraxtdagi node'lar orasida harakatlanish. Ikki xil navigatsiya mavjud:

1. **Node navigatsiyasi** — barcha node'lar (text, comment ham)
2. **Element navigatsiyasi** — faqat element node'lar

### Node Navigatsiyasi vs Element Navigatsiyasi

| Node (barchasi) | Element (faqat) | Tavsif |
|-----------------|-----------------|--------|
| `parentNode` | `parentElement` | Ota node/element |
| `childNodes` | `children` | Barcha bolalar |
| `firstChild` | `firstElementChild` | Birinchi bola |
| `lastChild` | `lastElementChild` | Oxirgi bola |
| `nextSibling` | `nextElementSibling` | Keyingi aka-uka |
| `previousSibling` | `previousElementSibling` | Oldingi aka-uka |

### Kod Misollari

```html
<ul id="list">
  <li>Birinchi</li>
  <li>Ikkinchi</li>
  <li>Uchinchi</li>
</ul>
```

```javascript
const list = document.getElementById("list");

// Node navigatsiyasi — text node'lar ham kiradi (whitespace)
console.log(list.childNodes);
// NodeList(7): [text, li, text, li, text, li, text]
// text node'lar — HTML dagi bo'sh joylar va yangi qatorlar

// Element navigatsiyasi — faqat elementlar
console.log(list.children);
// HTMLCollection(3): [li, li, li]

// Birinchi bola
console.log(list.firstChild);        // #text (whitespace)
console.log(list.firstElementChild); // <li>Birinchi</li>

// Oxirgi bola
console.log(list.lastChild);         // #text (whitespace)
console.log(list.lastElementChild);  // <li>Uchinchi</li>

// Aka-uka (sibling)
const second = list.children[1];
console.log(second.nextElementSibling);     // <li>Uchinchi</li>
console.log(second.previousElementSibling); // <li>Birinchi</li>

// Ota element
console.log(second.parentElement); // <ul id="list">
```

### `closest()` va `matches()`

```javascript
// closest() — o'zidan yuqoriga qarab CSS selector bo'yicha qidiradi
const link = document.querySelector(".nav-item a");
const nav = link.closest("nav");         // eng yaqin <nav> ota element
const card = link.closest(".card");      // eng yaqin .card class li ota
const nothing = link.closest(".absent"); // null — topilmadi

// matches() — element selector ga mos keladimi tekshiradi
const div = document.querySelector("div");
console.log(div.matches(".active"));       // true/false
console.log(div.matches("div.container")); // true/false

// Amaliy misol — event delegation da closest ishlatish
document.addEventListener("click", (e) => {
  const button = e.target.closest("button");
  if (button) {
    console.log("Button bosildi:", button.textContent);
  }
});
```

---

## Element Tanlash (Selecting)

### Nazariya

DOM dan element tanlashning bir necha usuli bor. Zamonaviy JS da asosan `querySelector` va `querySelectorAll` ishlatiladi.

### Usullar Jadvali

| Usul | Qaytaradi | Live? | Tezlik |
|------|-----------|-------|--------|
| `getElementById("id")` | Element \| null | — | ⚡ eng tez |
| `getElementsByClassName("cls")` | HTMLCollection | ✅ live | 🟡 tez |
| `getElementsByTagName("tag")` | HTMLCollection | ✅ live | 🟡 tez |
| `querySelector(".css")` | Element \| null | — | 🟠 o'rta |
| `querySelectorAll(".css")` | NodeList | ❌ static | 🟠 o'rta |

### Live vs Static Collection

```javascript
const liveList = document.getElementsByClassName("item");
const staticList = document.querySelectorAll(".item");

console.log(liveList.length);  // 3
console.log(staticList.length); // 3

// Yangi element qo'shamiz
const newItem = document.createElement("div");
newItem.className = "item";
document.body.appendChild(newItem);

// Live collection avtomatik yangilandi!
console.log(liveList.length);  // 4 ← o'zgardi!
console.log(staticList.length); // 3 ← o'zgarmadi (static)
```

### Kod Misollari

```javascript
// getElementById — eng tez, unikal element uchun
const header = document.getElementById("main-header");

// querySelector — CSS selector bilan birinchi topilgan element
const firstCard = document.querySelector(".card");
const activeLink = document.querySelector("nav a.active");
const input = document.querySelector("input[type='email']");

// querySelectorAll — barcha mos elementlar
const allCards = document.querySelectorAll(".card");
const evenRows = document.querySelectorAll("tr:nth-child(even)");

// querySelectorAll NodeList qaytaradi — forEach ishlaydi
allCards.forEach((card, index) => {
  console.log(`Card ${index}:`, card.textContent);
});

// HTMLCollection da forEach YO'Q — avval Array ga aylantirish kerak
const divs = document.getElementsByTagName("div");
// divs.forEach(...) // ❌ TypeError!
Array.from(divs).forEach(div => console.log(div)); // ✅

// Yoki spread operator
[...divs].forEach(div => console.log(div)); // ✅
```

### Under the Hood

`getElementById` eng tez — chunki brauzer ID lar uchun **hash map** yuritadi, lookup O(1). `querySelector` esa CSS selector parser'dan o'tadi va DOM daraxtini qidirishi kerak — O(n). Lekin zamonaviy brauzerlar querySelector ni ham optimize qilgan, farq kichik sahifalarda sezilmaydi.

```javascript
// Performance comparison (amalda)
// 100,000 ta element bor sahifada:
console.time("getElementById");
for (let i = 0; i < 10000; i++) {
  document.getElementById("test");
}
console.timeEnd("getElementById"); // ~2-5ms

console.time("querySelector");
for (let i = 0; i < 10000; i++) {
  document.querySelector("#test");
}
console.timeEnd("querySelector"); // ~5-15ms
```

---

## Element Yaratish va Qo'shish

### Nazariya

DOM ga yangi element qo'shish — web dasturlarning asosiy vazifalaridan biri. Element yaratib, uni daraxtga qo'shishning bir necha usuli bor.

### Element Yaratish

```javascript
// createElement — yangi HTML element
const div = document.createElement("div");
div.className = "card";
div.id = "card-1";
div.textContent = "Yangi karta";

// createTextNode — matn node
const text = document.createTextNode("Oddiy matn");

// createDocumentFragment — virtual container (keyinroq batafsil)
const fragment = document.createDocumentFragment();
```

### Qo'shish Usullari

| Usul | Tavsif | Qayerga |
|------|--------|---------|
| `appendChild(node)` | Oxiriga qo'shadi | Parent ning ichiga |
| `insertBefore(new, ref)` | Reference oldiga | Parent ning ichiga |
| `append(...nodes)` | Oxiriga (ko'p) | Parent ning ichiga |
| `prepend(...nodes)` | Boshiga (ko'p) | Parent ning ichiga |
| `after(...nodes)` | Element dan keyin | Element ning yoniga |
| `before(...nodes)` | Element dan oldin | Element ning yoniga |
| `replaceChild(new, old)` | Almashtirib qo'yadi | Parent ning ichiga |
| `replaceWith(...nodes)` | O'zini almashtiradi | Element o'rniga |

### Kod Misollari

```javascript
const container = document.getElementById("container");

// === appendChild — oxiriga qo'shish ===
const p = document.createElement("p");
p.textContent = "Yangi paragraf";
container.appendChild(p);

// === insertBefore — ma'lum elementdan oldin ===
const newItem = document.createElement("li");
newItem.textContent = "Yangi element";
const referenceItem = container.children[2]; // 3-chi element
container.insertBefore(newItem, referenceItem);
// referenceItem null bo'lsa — appendChild kabi oxiriga qo'shadi

// === append vs appendChild ===
// append — ko'p node va string qabul qiladi
container.append(
  document.createElement("div"),
  document.createElement("span"),
  "Oddiy matn"  // string ham bersa bo'ladi
);

// appendChild — faqat bitta node
// container.appendChild("matn"); // ❌ TypeError

// === prepend — boshiga qo'shish ===
const header = document.createElement("h2");
header.textContent = "Sarlavha";
container.prepend(header); // container ning eng boshiga

// === after / before — yoniga qo'shish ===
const target = document.querySelector(".target");
const afterEl = document.createElement("p");
afterEl.textContent = "Target dan keyin";
target.after(afterEl);

const beforeEl = document.createElement("p");
beforeEl.textContent = "Target dan oldin";
target.before(beforeEl);

// === replaceWith — element ni almashtirish ===
const oldElement = document.querySelector(".old");
const newElement = document.createElement("div");
newElement.textContent = "Yangi element";
oldElement.replaceWith(newElement);
```

### `insertAdjacentHTML`, `insertAdjacentElement`, `insertAdjacentText`

Bu usullar **aniq pozitsiya** ni belgilash imkonini beradi:

```javascript
const div = document.querySelector(".container");

// 4 ta pozitsiya:
// beforebegin — element DAN OLDIN
// afterbegin  — element ICHIDA, BOSHIDA
// beforeend   — element ICHIDA, OXIRIDA
// afterend    — element DAN KEYIN

/*
  <!-- beforebegin -->
  <div class="container">
    <!-- afterbegin -->
    mavjud content
    <!-- beforeend -->
  </div>
  <!-- afterend -->
*/

// insertAdjacentHTML — HTML string bilan
div.insertAdjacentHTML("beforeend", "<p>Yangi <strong>paragraf</strong></p>");

// insertAdjacentElement — element bilan
const span = document.createElement("span");
div.insertAdjacentElement("afterbegin", span);

// insertAdjacentText — oddiy matn bilan
div.insertAdjacentText("beforeend", "Qo'shimcha matn");
```

### Under the Hood

Har safar DOM ga element qo'shilganda brauzer bir necha bosqichdan o'tadi:

```
DOM o'zgarishi → Style hisoblash → Layout (Reflow) → Paint → Composite
```

Shuning uchun ko'p elementni birma-bir qo'shish — **sekin**. Buning o'rniga **DocumentFragment** yoki **innerHTML** ishlatish samaraliroq (performance bo'limida batafsil).

---

## Element O'chirish

### Nazariya

DOM dan element o'chirish ham bir necha usulda amalga oshiriladi. Zamonaviy `remove()` methodi eng qulay.

### Kod Misollari

```javascript
// === remove() — zamonaviy, eng oddiy ===
const element = document.querySelector(".remove-me");
element.remove(); // Tamom — element DOM dan chiqdi

// === removeChild() — eski usul, parent kerak ===
const parent = document.getElementById("list");
const child = parent.children[0];
parent.removeChild(child); // qaytaradi: o'chirilgan element

// === Barcha bolalarni o'chirish ===

// 1-usul: innerHTML bilan (tez, lekin event listener'lar ham yo'qoladi)
container.innerHTML = "";

// 2-usul: Loop bilan (xavfsizroq)
while (container.firstChild) {
  container.removeChild(container.firstChild);
}

// 3-usul: replaceChildren() — zamonaviy (ES2020+)
container.replaceChildren(); // barcha bolalarni o'chiradi
container.replaceChildren(newChild1, newChild2); // va yangilarini qo'yadi
```

### ⚠️ Muhim: O'chirilgan element xotirada

```javascript
const removed = document.querySelector(".item");
removed.remove(); // DOM dan chiqdi

// LEKIN: agar JavaScript da reference saqlansa — GC tozalamaydi!
// removed o'zgaruvchisi hali element ga ishora qilmoqda
// Bu memory leak bo'lishi mumkin — katta dasturlarda diqqatli bo'ling

// Yechim: reference ni tozalang
// removed = null; // agar kerak bo'lmasa
```

---

## Content O'zgartirish

### Nazariya

Element ichidagi contentni o'zgartirishning 3 xil yo'li bor. Har birining farqi va xavfini tushunish muhim.

### `textContent` vs `innerHTML` vs `innerText`

| Xususiyat | Nima qiladi | HTML parse? | Xavfsiz? | Performance |
|-----------|-------------|-------------|----------|-------------|
| `textContent` | Barcha matn (yashirin ham) | ❌ | ✅ Xavfsiz | ⚡ Tez |
| `innerHTML` | HTML content | ✅ Parse qiladi | ⚠️ XSS xavfi | 🟡 O'rta |
| `innerText` | Ko'rinadigan matn | ❌ | ✅ Xavfsiz | 🔴 Sekin |

### Kod Misollari

```html
<div id="demo">
  <span>Ko'rinadigan</span>
  <span style="display:none">Yashirin</span>
</div>
```

```javascript
const el = document.getElementById("demo");

// textContent — BARCHA matn, HTML teg'larsiz
console.log(el.textContent); // "\n  Ko'rinadigan\n  Yashirin\n"
// Yashirin element ham kiradi!

// innerText — faqat ko'rinadigan matn
console.log(el.innerText); // "Ko'rinadigan"
// display:none bo'lgan matn kiritilMAYDI

// innerHTML — HTML sifatida
console.log(el.innerHTML);
// "\n  <span>Ko'rinadigan</span>\n  <span style=\"display:none\">Yashirin</span>\n"
```

### ⚠️ innerHTML va XSS

```javascript
// ❌ XAVFLI — foydalanuvchi kiritgan ma'lumotni innerHTML ga qo'ymang!
const userInput = '<img src=x onerror="alert(\'hacked!\')">';
element.innerHTML = userInput; // 💀 XSS hujum!

// ✅ XAVFSIZ — textContent ishlatamiz
element.textContent = userInput; // Oddiy matn sifatida ko'rsatadi, xavfsiz

// ✅ XAVFSIZ — agar HTML kerak bo'lsa, sanitize qiling
// DOMPurify kutubxonasi bilan:
// element.innerHTML = DOMPurify.sanitize(userInput);
```

### `outerHTML`

```javascript
const el = document.querySelector(".card");

// outerHTML — element NI O'ZINI HAM beradi
console.log(el.outerHTML); // "<div class='card'>Content</div>"
console.log(el.innerHTML); // "Content"

// outerHTML ga yozish — element o'zi almashadi
el.outerHTML = "<section>Yangi element</section>";
// Diqqat: eski el variable hali eski element ga ishora qiladi!
// el endi DOM da yo'q, lekin JS da reference saqlangan
```

---

## Attributes va Dataset

### Nazariya

HTML element'larning attribute'lari DOM orqali o'qiladi va o'zgartiriladi. Standart attribute'lar property sifatida ham mavjud, lekin umumiy usullar ham bor.

### Standart Attribute Usullari

```javascript
const link = document.querySelector("a");

// === getAttribute / setAttribute ===
link.getAttribute("href");           // "https://example.com"
link.setAttribute("href", "https://google.com");
link.setAttribute("target", "_blank");

// === hasAttribute / removeAttribute ===
link.hasAttribute("target");         // true
link.removeAttribute("target");

// === attributes — barcha attribute'lar ===
console.log(link.attributes);        // NamedNodeMap
for (const attr of link.attributes) {
  console.log(`${attr.name} = ${attr.value}`);
}
```

### Property vs Attribute

```javascript
const input = document.querySelector("input");

// Attribute — HTML da yozilgan boshlang'ich qiymat
// Property — DOM object dagi hozirgi qiymat

input.setAttribute("value", "boshlang'ich"); // HTML attribute
input.value = "hozirgi";                      // DOM property

console.log(input.getAttribute("value")); // "boshlang'ich" — attribute o'zgarmadi
console.log(input.value);                 // "hozirgi" — property o'zgardi

// Farq: value, checked (checkbox), selected (option) kabilar uchun
// attribute = boshlang'ich qiymat (HTML da yozilgan)
// property = hozirgi amaliy qiymat (user yoki JS o'zgartirgan)
```

### `data-*` Attributes va `dataset`

Custom ma'lumotlarni element da saqlash uchun `data-*` attribute'lar ishlatiladi:

```html
<div id="user-card"
  data-user-id="42"
  data-user-name="Ali"
  data-is-active="true"
  data-created-at="2024-01-15">
  Ali — Profil
</div>
```

```javascript
const card = document.getElementById("user-card");

// dataset — barcha data-* attribute'lar
console.log(card.dataset);
// DOMStringMap {
//   userId: "42",        ← data-user-id → camelCase
//   userName: "Ali",     ← data-user-name → camelCase
//   isActive: "true",
//   createdAt: "2024-01-15"
// }

// O'qish
console.log(card.dataset.userId);   // "42" (string!)
console.log(card.dataset.userName); // "Ali"

// Yozish — yangi data attribute qo'shish
card.dataset.role = "admin";
// HTML da: data-role="admin" paydo bo'ladi

// O'chirish
delete card.dataset.isActive;
// HTML dan: data-is-active o'chirildi

// ⚠️ Diqqat: dataset qiymatlari doim STRING
// "42" ni number sifatida ishlatish uchun parseInt yoki Number() kerak
const userId = Number(card.dataset.userId); // 42

// CSS da data attribute lar bilan ishlash
// [data-is-active="true"] { color: green; }
```

---

## ClassList API

### Nazariya

Element'larning CSS class'larini boshqarish uchun `classList` API ishlatiladi. Bu `className` property'siga qaraganda ancha qulay va xavfsiz.

### Kod Misollari

```javascript
const button = document.querySelector("button");

// === className — eski usul, butun string ===
button.className = "btn primary large";
// ⚠️ Muammo: oldingi class'larni o'chirib tashlaydi!
button.className += " active"; // " active" bilan qo'shish — xato bo'lishi mumkin

// === classList — zamonaviy, xavfsiz ===

// add — class qo'shish (bir yoki bir nechta)
button.classList.add("active");
button.classList.add("btn", "primary", "large"); // bir nechta

// remove — class o'chirish
button.classList.remove("large");
button.classList.remove("btn", "primary"); // bir nechta

// toggle — bor bo'lsa o'chiradi, yo'q bo'lsa qo'shadi
button.classList.toggle("active");
// Qaytaradi: true (qo'shildi) yoki false (o'chirildi)

// toggle — force parametri bilan
button.classList.toggle("active", true);  // doim qo'shadi
button.classList.toggle("active", false); // doim o'chiradi
// Bu if/else o'rniga qulay:
const isActive = someCondition;
button.classList.toggle("active", isActive);

// contains — class borligini tekshirish
if (button.classList.contains("active")) {
  console.log("Active!");
}

// replace — bir class ni boshqasiga almashtirish
button.classList.replace("btn-primary", "btn-secondary");
// Qaytaradi: true (almashdi) yoki false (eski class topilmadi)

// item — index bo'yicha class olish
console.log(button.classList.item(0)); // birinchi class nomi

// length — nechtа class bor
console.log(button.classList.length); // 3

// forEach — barcha class'lar bo'ylab iteratsiya
button.classList.forEach(cls => console.log(cls));
```

### Amaliy Misol: Theme Switcher

```javascript
function toggleTheme() {
  document.body.classList.toggle("dark-theme");
  const isDark = document.body.classList.contains("dark-theme");
  localStorage.setItem("theme", isDark ? "dark" : "light");
}

// Sahifa yuklanganda theme ni tiklash
const savedTheme = localStorage.getItem("theme");
if (savedTheme === "dark") {
  document.body.classList.add("dark-theme");
}
```

---

## Style Manipulation

### Nazariya

Element'larning CSS stillarini JavaScript orqali ikki usulda boshqarish mumkin: **inline styles** (to'g'ridan-to'g'ri) va **computed styles** (hisoblangan).

### Inline Styles

```javascript
const box = document.querySelector(".box");

// O'qish va yozish — camelCase!
box.style.backgroundColor = "blue";     // background-color → backgroundColor
box.style.fontSize = "18px";            // font-size → fontSize
box.style.marginTop = "20px";           // margin-top → marginTop
box.style.zIndex = "10";                // z-index → zIndex
box.style.display = "flex";

// CSS o'zgaruvchi (custom property) bilan
box.style.setProperty("--main-color", "#3498db");
box.style.getPropertyValue("--main-color"); // "#3498db"
box.style.removeProperty("--main-color");

// Stil o'chirish — bo'sh string
box.style.backgroundColor = ""; // inline style o'chirildi, CSS ga qaytadi

// cssText — butun inline style ni birdan
box.style.cssText = "color: red; font-size: 20px; padding: 10px;";
// ⚠️ Oldingi inline style'larni TO'LIQ o'chiradi
```

### Computed Styles

```javascript
const box = document.querySelector(".box");

// getComputedStyle — hisoblangan (final) stil qiymatlari
const computed = getComputedStyle(box);
console.log(computed.width);           // "300px" — haqiqiy kengligi
console.log(computed.backgroundColor); // "rgb(52, 152, 219)"
console.log(computed.fontSize);        // "16px"
console.log(computed.display);         // "flex"

// Pseudo-element stillarini olish
const beforeStyles = getComputedStyle(box, "::before");
console.log(beforeStyles.content); // '"Some text"'

// ⚠️ getComputedStyle FAQAT O'QISH uchun
// computed.width = "500px"; // ❌ Ishlamaydi!
```

### Under the Hood

```javascript
// style property faqat INLINE style'larni ko'rsatadi
const div = document.querySelector("div");
// CSS faylda: div { color: red; }
console.log(div.style.color);               // "" — inline yo'q
console.log(getComputedStyle(div).color);    // "rgb(255, 0, 0)" — CSS dan

// getComputedStyle — final qiymat hisoblaydi:
// 1. Browser default styles
// 2. External CSS files
// 3. <style> blocks
// 4. Inline styles (eng yuqori priority, !important dan tashqari)
// Barchasi cascade qoidasi bo'yicha birlashtiriladi
```

---

## Performance: Reflow va Repaint

### Nazariya

DOM o'zgartirishlar brauzer rendering pipeline'iga ta'sir qiladi. Ikki asosiy tushuncha bor:

- **Reflow (Layout)** — element'larning o'lchami yoki pozitsiyasi qayta hisoblanadi. **Qimmat** operatsiya.
- **Repaint** — element'larning ko'rinishi (rang, shadow) qayta chiziladi. Reflow dan **arzonroq**.

### Rendering Pipeline

```
DOM/CSS o'zgarishi → Style → Layout (Reflow) → Paint (Repaint) → Composite → Screen
```

### Nima Reflow Trigger Qiladi?

```javascript
// ❌ Reflow trigger qiluvchi OPERATSIYALAR:
element.offsetWidth;      // o'qish
element.offsetHeight;     // o'qish
element.clientWidth;      // o'qish
element.clientHeight;     // o'qish
element.scrollTop;        // o'qish
element.getBoundingClientRect(); // o'qish
getComputedStyle(element);       // o'qish

// ❌ Reflow trigger qiluvchi STYLE O'ZGARISHLAR:
element.style.width = "100px";
element.style.height = "200px";
element.style.margin = "10px";
element.style.padding = "5px";
element.style.display = "block";
element.style.position = "absolute";
element.style.fontSize = "14px";
element.style.fontFamily = "Arial";
// ... va boshqa layout ga ta'sir qiluvchi property'lar
```

### Nima Faqat Repaint Trigger Qiladi?

```javascript
// 🟡 Faqat Repaint (Reflow siz):
element.style.color = "red";
element.style.backgroundColor = "blue";
element.style.visibility = "hidden";    // Reflow YO'Q (joy saqlaydi)
element.style.boxShadow = "0 0 5px #000";
element.style.outline = "1px solid red";
element.style.opacity = "0.5";
```

### ❌ Layout Thrashing — Eng Katta Muammo

```javascript
const items = document.querySelectorAll(".item");

// ❌ NOTO'G'RI — Layout Thrashing
// Har bir iteratsiyada O'QISH + YOZISH = har safar reflow!
items.forEach(item => {
  const height = item.offsetHeight;    // O'QISH → reflow!
  item.style.height = height * 2 + "px"; // YOZISH → invalidate
  // Keyingi iteratsiyada offsetHeight → QAYTA reflow!
});

// ✅ TO'G'RI — Avval hammasini O'QI, keyin hammasini YOZ
const heights = [];
items.forEach(item => {
  heights.push(item.offsetHeight); // Avval barchasini O'QISH (bitta reflow)
});
items.forEach((item, i) => {
  item.style.height = heights[i] * 2 + "px"; // Keyin barchasini YOZISH
});
// Jami: bitta reflow + bitta reflow = 2 ta
// Noto'g'ri usulda: N ta reflow!
```

### ✅ Performance Best Practices

```javascript
// 1. Class orqali o'zgartirish — bitta reflow
// ❌
element.style.width = "100px";
element.style.height = "200px";
element.style.margin = "10px";
element.style.padding = "5px";
// 4 ta style o'zgartirish — brauzer optimize qilsa ham, xavfli

// ✅
element.classList.add("new-layout");
// CSS da: .new-layout { width: 100px; height: 200px; margin: 10px; padding: 5px; }
// Bitta reflow!

// 2. cssText bilan birdan o'zgartirish
element.style.cssText = "width: 100px; height: 200px; margin: 10px;";

// 3. DOM dan chiqarib, o'zgartirib, qaytarish
const parent = element.parentNode;
const next = element.nextSibling;
parent.removeChild(element);
// ... ko'p o'zgartirishlar ...
parent.insertBefore(element, next);

// 4. requestAnimationFrame bilan DOM update
requestAnimationFrame(() => {
  element.style.transform = `translateX(${x}px)`;
  // Brauzer keyingi frame da bajaradi — smooth animation
});

// 5. transform va opacity — Reflow/Repaint siz!
// Bu ikki property GPU (compositor) da ishlaydi
element.style.transform = "translateX(100px)"; // Layout ta'sir qilmaydi
element.style.opacity = "0.5";                  // Paint ta'sir qilmaydi
```

---

## DocumentFragment

### Nazariya

`DocumentFragment` — bu **yengil, virtual DOM container**. U haqiqiy DOM ning bir qismi emas, shuning uchun unga element qo'shish **reflow trigger qilmaydi**. Barcha elementlarni fragment ga yig'ib, keyin bir marta DOM ga qo'shish — samarali usul.

### Kod Misollari

```javascript
// ❌ NOTO'G'RI — har bir qo'shish reflow trigger qiladi
const list = document.getElementById("list");
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list.appendChild(li); // har safar reflow!
}

// ✅ TO'G'RI — DocumentFragment bilan
const list = document.getElementById("list");
const fragment = document.createDocumentFragment();

for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li); // fragment ga — reflow YO'Q
}

list.appendChild(fragment); // BITTA reflow!
// fragment endi bo'sh — bolalari list ga ko'chdi
```

### Fragment vs innerHTML

```javascript
// innerHTML — string sifatida (tez, lekin xavf bor)
list.innerHTML = items.map(item =>
  `<li class="item">${item.name}</li>`
).join("");
// ⚠️ item.name da HTML bo'lsa — XSS xavfi!
// ⚠️ Mavjud event listener'lar yo'qoladi!

// fragment — xavfsiz va event listener'lar saqlanadi
const fragment = document.createDocumentFragment();
items.forEach(item => {
  const li = document.createElement("li");
  li.className = "item";
  li.textContent = item.name; // XSS xavfsiz
  fragment.appendChild(li);
});
list.appendChild(fragment);

// <template> elementi ham fragment sifatida ishlaydi
const template = document.getElementById("item-template");
const clone = template.content.cloneNode(true); // DocumentFragment
// template.content = DocumentFragment
```

### Under the Hood

```
                    DOM Tree
                    ┌───────┐
                    │  body │
                    └───┬───┘
                        │
                    ┌───┴───┐
                    │  ul   │ ← bu DOM da
                    └───────┘

    DocumentFragment              
    ┌───────────────┐             
    │   Fragment    │ ← bu DOM da EMAS
    │  ┌────┐      │   Reflow yo'q!
    │  │ li │      │
    │  ├────┤      │
    │  │ li │      │
    │  ├────┤      │
    │  │ li │      │
    │  └────┘      │
    └──────────────┘

    ul.appendChild(fragment)    
    ┌───────┐                   
    │  ul   │ ← BITTA reflow    
    │ ┌────┐│                   
    │ │ li ││                   
    │ ├────┤│ Fragment bo'sh —  
    │ │ li ││ bolalari ko'chdi  
    │ ├────┤│                   
    │ │ li ││                   
    │ └────┘│                   
    └───────┘                   
```

---

## Virtual DOM Tushunchasi

### Nazariya

Virtual DOM — bu DOM ning **JavaScript object** sifatidagi engil nusxasi. React, Vue kabi framework'lar ishlatadi. Maqsad: DOM operatsiyalarini minimallashtirib, performance ni oshirish.

### Nima Uchun Kerak?

```
Muammo:
1. DOM operatsiyalari qimmat (reflow, repaint)
2. Ko'p o'zgarishlar bo'lsa — brauzer qotishi mumkin
3. Qaysi element o'zgarganligi ni tracking qilish qiyin

Yechim: Virtual DOM
1. Haqiqiy DOM ning JS object nusxasini saqlash
2. O'zgarish bo'lganda — yangi virtual DOM qurish
3. Eski va yangi virtual DOM ni solishtirish (diffing)
4. Faqat farqlarni haqiqiy DOM ga qo'llash (patching)
```

### Soddalashtirilgan Virtual DOM Misoli

```javascript
// Virtual DOM — oddiy JavaScript object
const virtualDOM = {
  tag: "div",
  props: { className: "container" },
  children: [
    {
      tag: "h1",
      props: {},
      children: ["Salom"]
    },
    {
      tag: "p",
      props: { id: "text" },
      children: ["Bu paragraf"]
    }
  ]
};

// Virtual element yaratuvchi funksiya
function createElement(tag, props = {}, ...children) {
  return { tag, props, children: children.flat() };
}

// Ishlatish
const vdom = createElement("div", { className: "app" },
  createElement("h1", {}, "Title"),
  createElement("p", { id: "desc" }, "Description")
);

// Virtual DOM ni haqiqiy DOM ga aylantirish
function render(vnode) {
  // Matn node
  if (typeof vnode === "string") {
    return document.createTextNode(vnode);
  }

  const el = document.createElement(vnode.tag);

  // Props / attributes
  for (const [key, value] of Object.entries(vnode.props)) {
    if (key === "className") {
      el.className = value;
    } else if (key.startsWith("on")) {
      el.addEventListener(key.slice(2).toLowerCase(), value);
    } else {
      el.setAttribute(key, value);
    }
  }

  // Bolalar
  vnode.children.forEach(child => {
    el.appendChild(render(child));
  });

  return el;
}

// DOM ga qo'shish
document.body.appendChild(render(vdom));
```

### Diffing Algorithm (Soddalashtirilgan)

```javascript
// Eski va yangi VDOM orasidagi farqlarni topish
function diff(oldVNode, newVNode) {
  // Node o'chirilgan
  if (!newVNode) {
    return { type: "REMOVE" };
  }

  // Matn o'zgargan
  if (typeof oldVNode === "string" || typeof newVNode === "string") {
    if (oldVNode !== newVNode) {
      return { type: "REPLACE", newVNode };
    }
    return null; // O'zgarmagan
  }

  // Tag o'zgargan — butun element almashtirish
  if (oldVNode.tag !== newVNode.tag) {
    return { type: "REPLACE", newVNode };
  }

  // Faqat props yoki children o'zgargan
  return {
    type: "UPDATE",
    props: diffProps(oldVNode.props, newVNode.props),
    children: diffChildren(oldVNode.children, newVNode.children)
  };
}
```

### Under the Hood

```
Haqiqiy DOM bilan ishlash:
┌──────────────┐     ┌──────────────┐
│ O'zgarish 1  │────▶│    Reflow    │
├──────────────┤     ├──────────────┤
│ O'zgarish 2  │────▶│    Reflow    │  Har biri alohida
├──────────────┤     ├──────────────┤
│ O'zgarish 3  │────▶│    Reflow    │
└──────────────┘     └──────────────┘

Virtual DOM bilan ishlash:
┌──────────────┐     ┌──────────────┐     ┌──────┐     ┌────────┐
│ O'zgarish 1  │     │              │     │      │     │        │
├──────────────┤────▶│  Diff/Patch  │────▶│ Bitta│────▶│ Bitta  │
│ O'zgarish 2  │     │              │     │ DOM  │     │ Reflow │
├──────────────┤     │              │     │update│     │        │
│ O'zgarish 3  │     │              │     │      │     │        │
└──────────────┘     └──────────────┘     └──────┘     └────────┘
```

> **Eslatma:** Virtual DOM doim ham tezroq emas. Kichik, oddiy DOM o'zgarishlar uchun to'g'ridan-to'g'ri manipulation tezroq bo'lishi mumkin. Virtual DOM katta va murakkab UI'lar uchun optimaldir — React, Vue, Svelte kabi framework'larning tanloviga sabab shu.

---

## Common Mistakes

### ❌ Xato 1: innerHTML bilan foydalanuvchi ma'lumotini qo'yish (XSS)

```javascript
// ❌ Noto'g'ri
const username = getUserInput(); // "<script>alert('hacked')</script>"
element.innerHTML = `Salom, ${username}!`;
// 💀 XSS hujum — script bajariladi!
```

### ✅ To'g'ri usul:

```javascript
// ✅ textContent — HTML parse qilmaydi
element.textContent = `Salom, ${username}!`;
// Ko'rsatadi: "Salom, <script>alert('hacked')</script>!"
// Script sifatida bajarilMAYDI
```

**Nima uchun:** `innerHTML` string ni HTML sifatida parse qiladi. Agar foydalanuvchi zararli HTML/JS kiritsa — brauzer uni bajaradi.

---

### ❌ Xato 2: Loop ichida DOM ni birma-bir o'zgartirish

```javascript
// ❌ Noto'g'ri — 1000 ta reflow
const list = document.getElementById("list");
for (let i = 0; i < 1000; i++) {
  list.innerHTML += `<li>Item ${i}</li>`;
  // Har safar innerHTML QAYTA parse bo'ladi!
  // Mavjud bolalar yo'q qilinib, yangidan yaratiladi
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Fragment bilan — bitta reflow
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);
}
list.appendChild(fragment);

// Yoki innerHTML ni birdan
list.innerHTML = Array.from({ length: 1000 },
  (_, i) => `<li>Item ${i}</li>`
).join("");
```

**Nima uchun:** `innerHTML +=` har safar eski content ni parse qilib, yangisini qo'shadi — O(n²) operatsiya. 1000 ta element uchun bu juda sekin.

---

### ❌ Xato 3: Layout thrashing — o'qish va yozishni aralash qilish

```javascript
// ❌ Noto'g'ri
elements.forEach(el => {
  el.style.width = el.offsetWidth + 10 + "px"; // read + write har safar!
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ Avval o'qi, keyin yoz
const widths = [...elements].map(el => el.offsetWidth);
elements.forEach((el, i) => {
  el.style.width = widths[i] + 10 + "px";
});
```

**Nima uchun:** Brauzer style o'zgartirganda layout ni "dirty" deb belgilaydi. Keyingi `offsetWidth` o'qish uchun **majburan** layout hisoblanadi. Loop ichida bu har iteratsiyada takrorlanadi.

---

### ❌ Xato 4: Live collection bilan loop da element o'chirish

```javascript
// ❌ Noto'g'ri — elementlar "sakraydi"
const items = document.getElementsByClassName("item"); // LIVE!
for (let i = 0; i < items.length; i++) {
  items[i].remove(); // items[0] o'chirildi, items[1] endi items[0] bo'ldi!
  // Natija: faqat yarmi o'chiriladi
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Variant 1: Orqadan boshlab o'chirish
for (let i = items.length - 1; i >= 0; i--) {
  items[i].remove();
}

// ✅ Variant 2: while loop
while (items.length > 0) {
  items[0].remove();
}

// ✅ Variant 3: Static list (querySelectorAll)
document.querySelectorAll(".item").forEach(el => el.remove());
```

**Nima uchun:** `getElementsByClassName` **live** HTMLCollection qaytaradi. Element o'chirilsa — collection darhol o'zgaradi, index'lar siljiydi.

---

### ❌ Xato 5: `innerText` ni performance-critical joyda ishlatish

```javascript
// ❌ Sekin — innerText reflow trigger qiladi!
const text = element.innerText; // Layout hisoblanishi kerak (visibility uchun)

// ✅ Tez — textContent reflow qilmaydi
const text = element.textContent; // Faqat DOM node larni o'qiydi
```

**Nima uchun:** `innerText` CSS ni hisobga oladi (display:none, visibility) — shuning uchun reflow kerak. `textContent` esa DOM node'larning matnini to'g'ridan-to'g'ri oladi — ancha tez.

---

## Amaliy Mashqlar

### Mashq 1: Element Counter (Oson)

**Savol:** Sahifadagi barcha `<p>` elementlar sonini hisoblang va natijani `<h1>` ichiga yozing.

<details>
<summary>Javob</summary>

```javascript
const paragraphs = document.querySelectorAll("p");
const heading = document.querySelector("h1");
heading.textContent = `Sahifada ${paragraphs.length} ta paragraf bor`;
```

**Tushuntirish:** `querySelectorAll` static NodeList qaytaradi. `.length` bilan sonni olamiz. `textContent` xavfsiz yozish uchun.

</details>

---

### Mashq 2: Todo List (O'rta)

**Savol:** Oddiy todo list yarating:
- Input va button bor
- Button bosilganda input dagi matn yangi `<li>` sifatida `<ul>` ga qo'shiladi
- Har bir `<li>` da "O'chirish" tugmasi bo'lsin

<details>
<summary>Javob</summary>

```javascript
const input = document.getElementById("todo-input");
const button = document.getElementById("add-btn");
const list = document.getElementById("todo-list");

button.addEventListener("click", () => {
  const text = input.value.trim();
  if (!text) return;

  const li = document.createElement("li");
  li.textContent = text;

  const deleteBtn = document.createElement("button");
  deleteBtn.textContent = "O'chirish";
  deleteBtn.addEventListener("click", () => li.remove());

  li.appendChild(deleteBtn);
  list.appendChild(li);
  input.value = "";
  input.focus();
});
```

**Tushuntirish:** `createElement` bilan element yaratamiz, `addEventListener` bilan interaktivlik qo'shamiz, `remove()` bilan o'chiramiz.

</details>

---

### Mashq 3: Performant List Builder (O'rta)

**Savol:** 10,000 ta element yarating va DOM ga qo'shing. Ikki usulni solishtiring:
1. `appendChild` loop ichida
2. `DocumentFragment` bilan

`console.time` / `console.timeEnd` bilan vaqtni o'lchang.

<details>
<summary>Javob</summary>

```javascript
// 1-usul: To'g'ridan-to'g'ri
console.time("direct");
const list1 = document.getElementById("list1");
for (let i = 0; i < 10000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list1.appendChild(li);
}
console.timeEnd("direct"); // ~50-200ms

// 2-usul: Fragment bilan
console.time("fragment");
const list2 = document.getElementById("list2");
const fragment = document.createDocumentFragment();
for (let i = 0; i < 10000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);
}
list2.appendChild(fragment);
console.timeEnd("fragment"); // ~20-80ms (tezroq!)
```

**Tushuntirish:** Fragment bilan reflow bitta marta sodir bo'ladi. To'g'ridan-to'g'ri usulda brauzer har safar reflow'ni optimize qilishga harakat qiladi (batching), lekin fragment kafolatli bitta reflow.

</details>

---

### Mashq 4: DOM Traversal Challenge (Qiyin)

**Savol:** Quyidagi funksiyani yozing — berilgan elementning barcha bolalarini (recursive, hamma darajada) topib, faqat `Element` turidagilarini qaytaradi.

```javascript
function getAllDescendantElements(element) {
  // sizning kodingiz
}
```

<details>
<summary>Javob</summary>

```javascript
function getAllDescendantElements(element) {
  const result = [];

  function traverse(node) {
    for (const child of node.children) {
      result.push(child);
      traverse(child); // recursive — bolalarning bolalari
    }
  }

  traverse(element);
  return result;
}

// Test:
const container = document.getElementById("container");
const allElements = getAllDescendantElements(container);
console.log(`Jami ${allElements.length} ta element topildi`);

// Yoki TreeWalker API bilan (brauzer built-in):
function getAllDescendantsWalker(element) {
  const walker = document.createTreeWalker(
    element,
    NodeFilter.SHOW_ELEMENT // faqat elementlar
  );
  const result = [];
  while (walker.nextNode()) {
    result.push(walker.currentNode);
  }
  return result;
}
```

**Tushuntirish:**
- 1-variant: oddiy recursion — `children` faqat element'larni beradi
- 2-variant: `TreeWalker` — brauzerning built-in API, katta daraxtlar uchun samaraliroq

</details>

---

### Mashq 5: Mini Virtual DOM (Qiyin)

**Savol:** Soddalashtirilgan virtual DOM render funksiyasini yozing:
- `createElement(tag, props, ...children)` — virtual node yaratsin
- `render(vnode)` — haqiqiy DOM element qaytarsin
- props ichida `className`, `id`, va `on*` event'lar ishlashi kerak

<details>
<summary>Javob</summary>

```javascript
function createElement(tag, props = {}, ...children) {
  return { tag, props, children: children.flat() };
}

function render(vnode) {
  if (typeof vnode === "string" || typeof vnode === "number") {
    return document.createTextNode(String(vnode));
  }

  const el = document.createElement(vnode.tag);

  for (const [key, value] of Object.entries(vnode.props)) {
    if (key === "className") {
      el.className = value;
    } else if (key.startsWith("on")) {
      const event = key.slice(2).toLowerCase(); // onClick → click
      el.addEventListener(event, value);
    } else {
      el.setAttribute(key, value);
    }
  }

  vnode.children.forEach(child => {
    el.appendChild(render(child));
  });

  return el;
}

// Test:
const app = createElement("div", { className: "app", id: "root" },
  createElement("h1", {}, "Virtual DOM"),
  createElement("button", {
    className: "btn",
    onClick: () => alert("Bosildi!")
  }, "Bos meni"),
  createElement("ul", {},
    ...[1, 2, 3].map(n =>
      createElement("li", {}, `Item ${n}`)
    )
  )
);

document.body.appendChild(render(app));
```

**Tushuntirish:** Bu React-ning `createElement` va `ReactDOM.render` ning soddalashtirilgan versiyasi. Real framework'larda diff/patch algoritmi, fiber architecture, va boshqa murakkab optimizatsiyalar bor.

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **DOM nima** | HTML → Parser → DOM Tree — tirik, o'zgaruvchan daraxt |
| **Node types** | Element (1), Text (3), Comment (8), Document (9), Fragment (11) |
| **Traversal** | `children` (elementlar), `childNodes` (barchasi), `closest()`, `matches()` |
| **Selecting** | `querySelector/All` — universal; `getElementById` — eng tez |
| **Yaratish** | `createElement`, `createTextNode`, `createDocumentFragment` |
| **Qo'shish** | `append/prepend` (zamonaviy), `appendChild` (eski), `insertAdjacentHTML` |
| **O'chirish** | `remove()` (zamonaviy), `removeChild()` (eski), `replaceChildren()` |
| **Content** | `textContent` (xavfsiz, tez), `innerHTML` (XSS xavfi), `innerText` (sekin) |
| **Attributes** | `getAttribute/setAttribute`, `dataset` (data-* uchun) |
| **ClassList** | `add`, `remove`, `toggle`, `contains`, `replace` |
| **Styles** | `style.*` (inline), `getComputedStyle()` (hisoblangan, faqat o'qish) |
| **Reflow/Repaint** | Layout o'zgarishi = Reflow (qimmat), Rang o'zgarishi = Repaint (arzon) |
| **DocumentFragment** | Virtual container — ko'p elementni bitta reflow bilan qo'shish |
| **Virtual DOM** | DOM ning JS object nusxasi — diff/patch bilan minimal DOM update |

> **Keyingi bo'lim:** [Bo'lim 19: Events](19-events.md) — Event model, bubbling, capturing, delegation va boshqa DOM event mexanizmlari.

> **Cross-references:** [17-type-coercion.md](17-type-coercion.md) (typeof, ToPrimitive), [16-memory.md](16-memory.md) (GC, memory leak — element remove), [06-objects.md](06-objects.md) (property access, Object methods), [10-closures.md](10-closures.md) (event handler closures)
