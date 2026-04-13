# Bo'lim 18: DOM Manipulation

> DOM (Document Object Model) — brauzer HTML hujjatni parse qilib, JavaScript orqali boshqarish imkonini beradigan daraxt (tree) strukturasi. Har bir element, matn, komment — barchasi node. Bu bo'limda DOM traversal, element yaratish/o'zgartirish/o'chirish, attribute va style boshqaruv, va performance optimizatsiyasi (reflow/repaint, DocumentFragment, requestAnimationFrame) to'liq yoritiladi.

---

## Mundarija

- [DOM Nima](#dom-nima)
- [Node Types](#node-types)
- [DOM Traversal](#dom-traversal)
- [Element Tanlash (Selecting)](#element-tanlash-selecting)
- [Element Yaratish va Qo'shish](#element-yaratish-va-qoshish)
- [Element O'chirish va Almashtirish](#element-ochirish-va-almashtirish)
- [Cloning — cloneNode](#cloning--clonenode)
- [Content O'zgartirish](#content-ozgartirish)
- [Attributes va Dataset](#attributes-va-dataset)
- [ClassList API](#classlist-api)
- [Style Manipulation](#style-manipulation)
- [Performance: Reflow va Repaint](#performance-reflow-va-repaint)
- [DocumentFragment](#documentfragment)
- [requestAnimationFrame bilan DOM Update](#requestanimationframe-bilan-dom-update)
- [Virtual DOM Tushunchasi](#virtual-dom-tushunchasi)
- [Web Components](#web-components)
- [Observer APIs — Mutation, Intersection, Resize](#observer-apis--mutation-intersection-resize)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## DOM Nima

### Nazariya

DOM (Document Object Model) — brauzer HTML hujjatni parse qilib xotirada yaratadigan **daraxt (tree) strukturasi**. DOM **HTML ning o'zi emas** — HTML matn fayl, DOM esa brauzer xotirasidagi tirik (live) obyektlar daraxti. Brauzer HTML ni parse qilib DOM yaratadi, keyin JavaScript va CSS shu DOM bilan ishlaydi.

DOM — W3C standarti bo'lib, u **platform va til agnostic** interfeys beradi. JavaScript DOM bilan ishlash uchun eng ko'p ishlatiladigan til, lekin DOM API texnik jihatdan boshqa tillar bilan ham ishlatilishi mumkin. Brauzer kontekstida `document` global obyekt orqali DOM daraxtiga kirish mumkin.

Rendering pipeline: `HTML → Parser → DOM Tree + CSSOM → Render Tree → Layout → Paint → Composite → Screen`. Parse jarayoni **incremental** — brauzer hujjatni to'liq kutmaydi, kelgan qismlari bilan DOM qurishni boshlaydi. `<script>` tag'ga duch kelganda parser to'xtaydi (parser-blocking), shuning uchun script'larni `defer` yoki `async` bilan yoki `</body>` oldiga qo'yish tavsiya etiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Brauzer HTML parse qilganda tokenizer HTML ni token'larga ajratadi, keyin tree construction algoritmi DOM daraxtini quradi. DOM daraxt `document` node dan boshlanadi:

```
document (Document node — nodeType: 9)
└── html (Element node — nodeType: 1)
    ├── head
    │   ├── title
    │   │   └── "Sahifa nomi"  (Text node — nodeType: 3)
    │   └── meta
    └── body
        ├── h1
        │   └── "Salom"  (Text node)
        ├── p
        │   └── "Matn..."  (Text node)
        └── <!-- komment -->  (Comment node — nodeType: 8)
```

```
Rendering Pipeline:
┌──────┐    ┌───────┐    ┌───────────┐    ┌────────┐    ┌──────┐    ┌───────────┐
│ HTML │ →  │ Parse │ →  │ DOM Tree  │ →  │ Render │ →  │Layout│ →  │   Paint   │
│      │    │       │    │ + CSSOM   │    │  Tree  │    │      │    │ Composite │
└──────┘    └───────┘    └───────────┘    └────────┘    └──────┘    └───────────┘
```

V8 (Chrome) da DOM node'lari C++ obyektlari sifatida yaratiladi va JavaScript wrapper orqali expose qilinadi (Blink binding qatlami). Tarixan DOM operatsiyalari "sekinroq" deb hisoblangan — chunki har chaqiriq C++ tomoniga o'tish va uning effektlarini (layout, style recalculation) trigger qilish mumkin edi. Zamonaviy Blink va V8 inline caches, fast path'lar, va direct property access optimizatsiyalarini qo'llaydi — shuning uchun ko'p DOM getter/setter chaqiriqlari **deyarli native** tezlikda ishlaydi. Lekin **layout'ni trigger qiluvchi operatsiyalar** (`offsetWidth`, `getBoundingClientRect`, `getComputedStyle`) hanuzgacha qimmat qoladi — bu narsa "DOM sekinligi" ning haqiqiy sababi, JavaScript↔C++ bridge emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// document — DOM daraxtining ildizi
console.log(document.nodeType);     // 9 (DOCUMENT_NODE)
console.log(document.nodeName);     // "#document"
console.log(document.documentElement); // <html> — root element
console.log(document.head);         // <head>
console.log(document.body);         // <body>
console.log(document.title);        // sahifa sarlavhasi (o'qish/yozish)

// DOM live — o'zgartirilsa darhol ekranda ko'rinadi:
document.title = "Yangi Sarlavha"; // brauzer tab sarlavhasi o'zgaradi
```

</details>

---

## Node Types

### Nazariya

DOM daraxtidagi har bir element **node**. Node'larning bir nechta turi bor, har biri `nodeType` raqami bilan farqlanadi. Amalda eng ko'p ishlatiladigan 5 ta tur: **Element** (1) — HTML tag'lar (`<div>`, `<p>`, `<a>`); **Text** (3) — element ichidagi matn; **Comment** (8) — HTML kommentlari (`<!-- ... -->`); **Document** (9) — daraxtning ildizi; **DocumentFragment** (11) — virtual container, DOM da ko'rinmaydi, lekin ichiga node'lar qo'shiladi.

Barcha node'lar `Node` interfeysidan meros oladi — `nodeType`, `nodeName`, `nodeValue`, `parentNode`, `childNodes`, `firstChild`, `lastChild`, `nextSibling`, `previousSibling` property'lari bor. Element node'lari qo'shimcha `Element` interfeysidan ham meros oladi — `children`, `className`, `id`, `getAttribute()`, `querySelector()` kabi.

<details>
<summary><strong>Under the Hood</strong></summary>

DOM specification da `nodeType` 12 ta konstanta bilan aniqlangan (1 dan 12 gacha), lekin bir qismi **deprecated** yoki faqat XML kontekstida ishlatiladi. HTML DOM da amalda 5 ta asosiy tur ishlatiladi: **Element (1)**, **Text (3)**, **Comment (8)**, **Document (9)**, **DocumentFragment (11)**. Deprecated: `ATTRIBUTE_NODE (2)`, `ENTITY_REFERENCE_NODE (5)`, `ENTITY_NODE (6)`, `NOTATION_NODE (12)`. Kam ishlatiladigan lekin hali spec'da: `CDATA_SECTION_NODE (4)`, `PROCESSING_INSTRUCTION_NODE (7)`, `DOCUMENT_TYPE_NODE (10)` — asosan XML/XHTML hujjatlarda. Har bir node type o'z interface'iga ega va ular inheritance hierarchy bo'ylab joylashgan:

```
EventTarget
  └── Node (nodeType, childNodes, parentNode...)
        ├── Document (nodeType: 9) — getElementById, createElement
        ├── DocumentFragment (nodeType: 11) — virtual container
        ├── CharacterData
        │     ├── Text (nodeType: 3) — matn content
        │     └── Comment (nodeType: 8) — HTML kommentlar
        └── Element (nodeType: 1) — querySelector, classList, attributes
              ├── HTMLElement — style, dataset, offsetWidth
              │     ├── HTMLDivElement
              │     ├── HTMLInputElement — value, checked, files
              │     ├── HTMLAnchorElement — href, download
              │     └── ... (har bir HTML tag uchun subclass)
              └── SVGElement
```

`childNodes` property `NodeList` qaytaradi — bu **live** collection. DOM ga child qo'shilsa yoki o'chirilsa, `childNodes` avtomatik yangilanadi. `children` esa `HTMLCollection` qaytaradi — faqat Element node'lar (Text, Comment filtrlangan). Brauzer ichida `childNodes` node'larni linked list yoki array sifatida saqlaydi — engine implementatsiyasiga bog'liq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// nodeType konstantalari:
Node.ELEMENT_NODE                // 1
Node.TEXT_NODE                   // 3
Node.COMMENT_NODE                // 8
Node.DOCUMENT_NODE               // 9
Node.DOCUMENT_FRAGMENT_NODE      // 11

// Element ichidagi barcha child node'lar:
// <div id="box">Salom<!-- komment -->Dunyo</div>
const box = document.getElementById("box");

console.log(box.childNodes.length); // 3
// childNodes[0] — Text node: "Salom" (nodeType: 3)
// childNodes[1] — Comment node: " komment " (nodeType: 8)
// childNodes[2] — Text node: "Dunyo" (nodeType: 3)

console.log(box.children.length); // 0
// children — faqat Element node'lar (Text va Comment kiritilmaydi)

// Node turini tekshirish:
function isElementNode(node) {
  return node.nodeType === Node.ELEMENT_NODE; // 1
}
```

</details>

---

## DOM Traversal

### Nazariya

DOM traversal — daraxt bo'ylab yuqoriga, pastga va yonga harakatlanish. Ikki guruh property bor: **Node-based** (barcha node'lar — Text, Comment ham kiradi) va **Element-based** (faqat Element node'lar). Amalda deyarli doim Element-based property'lar ishlatiladi — Text va Comment node'lar odatda kerak emas.

| Node-based | Element-based | Farq |
|------------|---------------|------|
| `parentNode` | `parentElement` | Deyarli bir xil (document uchun farq) |
| `childNodes` | `children` | childNodes: Text+Comment ham; children: faqat Element |
| `firstChild` | `firstElementChild` | firstChild: Text bo'lishi mumkin |
| `lastChild` | `lastElementChild` | lastChild: Text bo'lishi mumkin |
| `nextSibling` | `nextElementSibling` | nextSibling: Text/Comment bo'lishi mumkin |
| `previousSibling` | `previousElementSibling` | previousSibling: Text/Comment bo'lishi mumkin |

`closest(selector)` — o'zidan yuqoriga qarab CSS selector bo'yicha birinchi mos ota elementni topadi (o'zini ham tekshiradi). `matches(selector)` — element berilgan CSS selector'ga mos keladimi tekshiradi. Bu ikki metod event delegation da muhim.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// <ul id="list"><li>Bir</li><li>Ikki</li><li>Uch</li></ul>
const list = document.getElementById("list");

// children — faqat Element node'lar
console.log(list.children.length);       // 3
console.log(list.firstElementChild);     // <li>Bir</li>
console.log(list.lastElementChild);      // <li>Uch</li>

// Aka-uka (sibling)
const second = list.children[1];
console.log(second.nextElementSibling);     // <li>Uch</li>
console.log(second.previousElementSibling); // <li>Bir</li>

// Ota element
console.log(second.parentElement); // <ul id="list">

// closest() — o'zidan yuqoriga qarab qidiradi
const link = document.querySelector(".nav-item a");
const nav = link.closest("nav");         // eng yaqin <nav> ota
const card = link.closest(".card");      // eng yaqin .card ota
const absent = link.closest(".absent");  // null — topilmadi

// matches() — selector ga mos keladimi
const div = document.querySelector("div");
console.log(div.matches(".active"));     // true/false

// closest + matches — event delegation da:
document.addEventListener("click", (e) => {
  const button = e.target.closest("button");
  if (button) {
    console.log("Button bosildi:", button.textContent);
  }
});
```

</details>

---

## Element Tanlash (Selecting)

### Nazariya

DOM dan element tanlashning bir necha usuli bor. `getElementById` eng tez — brauzer ID lar uchun **hash map** yuritadi, lookup O(1). `querySelector`/`querySelectorAll` CSS selector bilan ishlaydi — murakkab selector yozish mumkin, lekin CSS parser'dan o'tgani uchun biroz sekinroq.

Muhim farq: `getElementsByClassName`/`getElementsByTagName` **live** HTMLCollection qaytaradi (DOM o'zgarganda avtomatik yangilanadi), `querySelectorAll` esa **static** NodeList qaytaradi (snapshot — DOM o'zgarsa yangilanmaydi). Live collection loop ichida xavfli — DOM o'zgarsa collection ham o'zgaradi va cheksiz loop yoki elementlarni o'tkazib yuborish mumkin.

| Usul | Qaytaradi | Live? | Tezlik |
|------|-----------|-------|--------|
| `getElementById("id")` | Element \| null | — | ⚡ eng tez (O(1)) |
| `getElementsByClassName("cls")` | HTMLCollection | ✅ live | tez |
| `getElementsByTagName("tag")` | HTMLCollection | ✅ live | tez |
| `querySelector(".css")` | Element \| null | — | o'rta |
| `querySelectorAll(".css")` | NodeList | ❌ static | o'rta |

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// getElementById — unikal element uchun, eng tez
const header = document.getElementById("main-header");

// querySelector — CSS selector bilan birinchi topilgan element
const firstCard = document.querySelector(".card");
const activeLink = document.querySelector("nav a.active");
const emailInput = document.querySelector("input[type='email']");

// querySelectorAll — barcha mos elementlar (static NodeList)
const allCards = document.querySelectorAll(".card");
const evenRows = document.querySelectorAll("tr:nth-child(even)");

// NodeList da forEach ishlaydi:
allCards.forEach((card, i) => console.log(`Card ${i}:`, card.textContent));

// HTMLCollection da forEach YO'Q — Array ga aylantirib ishlatish kerak:
const divs = document.getElementsByTagName("div");
// divs.forEach(...) // ❌ TypeError!
Array.from(divs).forEach(div => console.log(div)); // ✅
[...divs].forEach(div => console.log(div));         // ✅

// Live vs Static farqi:
const liveList = document.getElementsByClassName("item");
const staticList = document.querySelectorAll(".item");
console.log(liveList.length);  // 3
console.log(staticList.length); // 3

// Yangi element qo'shamiz
const newItem = document.createElement("div");
newItem.className = "item";
document.body.appendChild(newItem);

console.log(liveList.length);  // 4 ← o'zgardi! (live)
console.log(staticList.length); // 3 ← o'zgarmadi (static snapshot)
```

</details>

<details>
<summary><strong>Under the Hood</strong></summary>

`getElementById` eng tez — brauzer DOM tree'dan alohida **ID hash map** yuritadi. Yangi ID li element qo'shilsa yoki o'chirilsa, map yangilanadi. `querySelector` esa CSS selector'ni avval parse qiladi, keyin DOM daraxtini depth-first preorder traversal bilan qidiradi — O(n). Lekin zamonaviy brauzerlar murakkab optimizatsiyalar qo'llaydi: Bloom filter, selector caching, indexed collections.

</details>

---

## Element Yaratish va Qo'shish

### Nazariya

DOM ga yangi element qo'shish — web dasturlarning asosiy vazifalaridan biri. `createElement()` bilan element yaratiladi, keyin turli usullar bilan daraxtga qo'shiladi. Zamonaviy API (`append`, `prepend`, `before`, `after`, `replaceWith`) va eski API (`appendChild`, `insertBefore`) farqi — zamonaviy usullar string ham qabul qiladi (avtomatik Text node yaratadi) va birda nechta argument oladi.

`insertAdjacentHTML/Element/Text` — 4 ta aniq pozitsiyani belgilash imkonini beradi: `beforebegin` (elementdan oldin), `afterbegin` (element ichiga boshiga), `beforeend` (element ichiga oxiriga), `afterend` (elementdan keyin). `insertAdjacentHTML` innerHTML'dan farqli ravishda mavjud child elementlarni o'chirmaydi va ularning event listener'larini saqlab qoladi — faqat yangi HTML ni ko'rsatilgan pozitsiyaga parse qilib qo'shadi.

Har safar DOM ga element qo'shilganda brauzer rendering pipeline'dan o'tadi, shuning uchun ko'p elementni birma-bir qo'shish sekin — `DocumentFragment` yoki batch usullar ishlatish kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Element yaratish
const div = document.createElement("div");
div.className = "card";
div.id = "card-1";
div.textContent = "Yangi karta";

// Text node yaratish
const text = document.createTextNode("Oddiy matn");

// ─── Qo'shish usullari ───

// appendChild — oxiriga qo'shadi (eski API, faqat node)
const container = document.getElementById("container");
container.appendChild(div);

// append — oxiriga, string ham oladi, bir nechta argument
container.append(div, " va matn", document.createElement("span"));

// prepend — boshiga qo'shadi
container.prepend(document.createElement("header"));

// before / after — elementdan oldin/keyin
const ref = document.getElementById("reference");
ref.before(document.createElement("hr"));  // ref dan oldin
ref.after(document.createElement("hr"));   // ref dan keyin

// insertBefore — reference element oldiga (eski API)
const newItem = document.createElement("li");
const list = document.getElementById("list");
const thirdItem = list.children[2];
list.insertBefore(newItem, thirdItem); // thirdItem oldiga qo'yadi

// ─── insertAdjacentHTML — 4 ta pozitsiya ───
const card = document.querySelector(".card");
card.insertAdjacentHTML("beforebegin", "<hr>"); // card dan OLDIN
card.insertAdjacentHTML("afterbegin", "<h2>Sarlavha</h2>"); // card ICHIGA boshiga
card.insertAdjacentHTML("beforeend", "<footer>Oxiri</footer>"); // card ICHIGA oxiriga
card.insertAdjacentHTML("afterend", "<hr>"); // card dan KEYIN

// Pozitsiyalar vizualizatsiyasi:
// <!-- beforebegin -->
// <div class="card">
//   <!-- afterbegin -->
//   ... mavjud content ...
//   <!-- beforeend -->
// </div>
// <!-- afterend -->
```

</details>

---

## Element O'chirish va Almashtirish

### Nazariya

Element o'chirishning zamonaviy usuli — `element.remove()` — elementni DOM daraxtidan olib tashlaydi. Eski usul — `parent.removeChild(child)` — ota element orqali o'chiradi va o'chirilgan node'ni qaytaradi. `replaceWith()` — elementni boshqa element yoki string bilan almashtiradi. `replaceChildren()` — parent'ning barcha child'larini yangi node'lar bilan almashtiradi (tozalab, keyin qo'shadi).

Muhim nuans: DOM dan o'chirilgan element JavaScript xotirasida qoladi (agar unga reference bo'lsa). Bu memory leak sababchisi bo'lishi mumkin — event listener'lar, closure'lar orqali eski DOM elementlariga reference saqlanib qolsa, GC ularni tozalay olmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// remove() — zamonaviy, sodda
const element = document.getElementById("old-item");
element.remove(); // DOM dan o'chirildi

// removeChild() — eski API, parent kerak
const list = document.getElementById("list");
const firstItem = list.firstElementChild;
const removed = list.removeChild(firstItem); // o'chirilgan node qaytadi
// removed hali xotirada bor — boshqa joyga qo'shish mumkin

// replaceWith() — element o'rniga boshqasini qo'yish
const oldCard = document.querySelector(".old-card");
const newCard = document.createElement("div");
newCard.className = "new-card";
newCard.textContent = "Yangi karta";
oldCard.replaceWith(newCard);
// oldCard endi DOM da yo'q, newCard uning o'rnida

// replaceChildren() — barcha child'larni almashtirish
const container = document.getElementById("container");
const newHeader = document.createElement("h1");
newHeader.textContent = "Yangi content";
container.replaceChildren(newHeader); // eski child'lar tozalandi, yangi qo'shildi

// Barcha child'larni tozalash:
container.replaceChildren(); // hech narsa bermasa — tozalaydi
// Eski usul:
container.innerHTML = ""; // ishlaydi, lekin event listener'larni ham yo'qotadi
```

</details>

---

## Cloning — cloneNode

### Nazariya

`cloneNode()` — mavjud DOM node'ning nusxasini yaratadi. Ikki rejimi bor: **shallow** (`cloneNode(false)` yoki `cloneNode()`) — faqat element o'zi klonlanadi, child'lari yo'q; **deep** (`cloneNode(true)`) — element va barcha ichki child'lari rekursiv klonlanadi.

Muhim nuans: `cloneNode` **event listener'larni klonlamaydi** (agar `addEventListener` bilan qo'shilgan bo'lsa). Inline event handler'lar (`onclick="..."`) HTML attribute sifatida klonlanadi. `id` attribute ham klonlanadi — shuning uchun klondan keyin ID ni o'zgartirish kerak (DOM da duplicate ID bo'lmasligi uchun).

<details>
<summary><strong>Under the Hood</strong></summary>

DOM spec bo'yicha `cloneNode` algoritmi ikki bosqichda ishlaydi. **Shallow clone** (`deep=false`): faqat node o'zi va uning attribute'lari (NamedNodeMap) nusxalanadi — child node'lar nusxalanmaydi. **Deep clone** (`deep=true`): node va barcha descendant'lari rekursiv nusxalanadi — har bir child node uchun `cloneNode(true)` chaqiriladi.

`addEventListener` orqali qo'shilgan event listener'lar klonlanmaydi — chunki ular DOM attribute emas, EventTarget ning ichki listener ro'yxatida saqlanadi. Bu ro'yxat node ning clone operatsiyasiga kiritilmagan. Inline handler'lar (`onclick="..."`) esa HTML attribute sifatida `NamedNodeMap` da saqlanadi, shuning uchun klonlanadi.

Clone qilingan node hali DOM tree'ga qo'shilmagan — `ownerDocument` o'rnatiladi, lekin `parentNode` `null` bo'ladi. DOM ga `appendChild`/`append` bilan qo'shilgunga qadar render pipeline'ga ta'sir qilmaydi. `<template>` element'ining `content` property'si (`DocumentFragment`) ni klonlash — Web Components va template pattern'da ko'p ishlatiladigan usul.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// <div id="card" class="card"><h2>Title</h2><p>Text</p></div>
const card = document.getElementById("card");

// Shallow clone — faqat <div>, ichidagi <h2> va <p> YO'Q
const shallow = card.cloneNode(false);
console.log(shallow.innerHTML);     // "" — bo'sh
console.log(shallow.className);     // "card" — attribute'lar bor

// Deep clone — element va barcha child'lar
const deep = card.cloneNode(true);
console.log(deep.innerHTML);        // "<h2>Title</h2><p>Text</p>"
deep.id = "card-copy"; // ⚠️ ID ni o'zgartiring — duplicate bo'lmasin!

document.body.appendChild(deep);

// Template pattern — bitta elementdan ko'p nusxa:
const template = document.createElement("li");
template.className = "list-item";
template.innerHTML = '<span class="text"></span><button class="delete">X</button>';

const items = ["Olma", "Nok", "Uzum"];
const fragment = document.createDocumentFragment();
items.forEach(item => {
  const li = template.cloneNode(true); // har safar deep clone
  li.querySelector(".text").textContent = item;
  fragment.appendChild(li);
});
document.getElementById("list").appendChild(fragment);
```

</details>

---

## Content O'zgartirish

### Nazariya

Element ichidagi contentni o'zgartirishning 3 ta asosiy usuli bor va ularning farqi muhim:

**`textContent`** — element va barcha child'larning **matn content'ini** o'qiydi/yozadi. Faqat matn — HTML parse qilinmaydi. Yozishda mavjud barcha child'lar o'chiriladi va bitta Text node bilan almashtiriladi. **XSS xavfi yo'q** — user input'ni xavfsiz ko'rsatish uchun eng yaxshi variant. Performance jihatdan eng tez — HTML parse qilmaydi, computed style hisoblamaydi.

**`innerHTML`** — element ichidagi **HTML markup'ni** string sifatida o'qiydi/yozadi. Yozishda string HTML sifatida parse qilinadi va DOM node'larga aylanadi. **XSS xavfi bor** — user input'ni innerHTML ga qo'yish xavfli (`<script>`, `onerror`, `onload` kabi). Yozishda mavjud child'larning barcha event listener'lari yo'qoladi (node'lar o'chiriladi va yangi yaratiladi).

**`innerText`** — element'ning **ko'rinadigan matn'ini** qaytaradi. `textContent` dan farqi: `display: none` bilan yashirilgan element matnini bermaydi, CSS layout'ni hisobga oladi, `<br>` ni yangi qatorga aylantiradi. Lekin bu uchun brauzer **reflow** qilishi kerak — shuning uchun `innerText` eng sekin.

<details>
<summary><strong>Under the Hood</strong></summary>

`textContent` o'qishda brauzer DOM subtree'ni traverse qilib barcha Text node'larning `nodeValue` qiymatlarini concatenate qiladi — layout engine ishtirok etmaydi. Yozishda esa barcha child node'lar o'chiriladi va bitta yangi Text node yaratiladi. Bu operatsiya arzon — HTML parser ishlamaydi.

`innerHTML` yozishda brauzer to'liq HTML parser'ni ishga tushiradi: tokenization → tree construction → DOM node creation. Mavjud child node'lar avval recursive o'chiriladi (event listener'lar ham yo'qoladi), keyin yangi DOM subtree qo'shiladi. Bu **XSS attack vector** — user input'dagi `<img onerror="...">`, `<svg onload="...">` kabi markup bajariladi. `<script>` tag'lar `innerHTML` orqali kiritilganda bajarilmaydi (HTML5 spec), lekin event handler attribute'lari (`onerror`, `onload`) bajariladi.

`innerText` o'qishda brauzer layout hisoblashi kerak — `getComputedStyle` har bir descendant uchun ko'rib chiqiladi, `display: none` elementlar filtrlanadi, shu sababli **forced synchronous layout (reflow)** trigger bo'ladi. CSS `visibility: hidden` elementlar ham chiqarilmaydi. Shuning uchun `innerText` performance jihatdan eng qimmat — aniq tezlik farqi DOM chuqurligi, pending layout invalidation holati va element soniga bog'liq, lekin "read-only" operatsiya sifatida silent reflow beradi, bu layout thrashing manbai bo'lishi mumkin. Faqat haqiqatan "ko'rinadigan matn" kerak bo'lgan joyda ishlating — ma'lumot ko'chirish, text extraction uchun `textContent` tanlang.

`outerHTML` o'qishda element o'zi ham serialization'ga kiritiladi. Yozishda esa element o'zi DOM dan o'chiriladi va yangi HTML parse qilingan node'lar uning o'rniga qo'yiladi — eski element'ga JavaScript reference bo'lsa, u endi DOM tree'da emas, lekin xotirada qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// <div id="box">
//   <span>Salom</span>
//   <span style="display:none">Yashirin</span>
// </div>
const box = document.getElementById("box");

// textContent — barcha matn (yashirinlari ham)
console.log(box.textContent); // "  Salom  Yashirin  " (whitespace ham)

// innerText — faqat ko'rinadigan matn
console.log(box.innerText);   // "Salom" (display:none olib tashlandi)

// innerHTML — HTML markup string
console.log(box.innerHTML);   // '<span>Salom</span><span style="display:none">Yashirin</span>'

// ─── Yozish ───

// ✅ textContent — xavfsiz, tez
box.textContent = "<script>alert('xss')</script>";
// Ekranda KO'RINADI: <script>alert('xss')</script>
// Script BAJARILMAYDI — faqat matn sifatida chiqadi

// ❌ innerHTML — XSS xavfi
box.innerHTML = "<img onerror='alert(1)' src='x'>";
// ⚠️ Bu alert(1) ni bajaradi! User input uchun ISHLATMANG

// ✅ Xavfsiz HTML qo'shish:
const userInput = "<b>Salom</b>";
const safe = document.createElement("div");
safe.textContent = userInput; // matn sifatida
// Yoki DOMPurify kutubxonasi bilan sanitize qilish

// outerHTML — element o'zini ham o'z ichiga oladi:
const p = document.querySelector("p");
console.log(p.outerHTML); // "<p class='info'>Matn</p>"
```

</details>

---

## Attributes va Dataset

### Nazariya

HTML element'larning ikki turdagi attribute'lari bor: **standart** (id, class, src, href, type, ...) va **custom** (`data-*`). Standart attribute'lar ko'pincha DOM property sifatida ham mavjud (`element.id`, `element.className`), lekin farqlar bor — attribute HTML dagi boshlang'ich qiymat, property esa JavaScript dagi joriy qiymat.

`getAttribute()`/`setAttribute()` — har qanday attribute bilan ishlaydi, doim **string** qaytaradi. `element.property` — faqat standart attribute'lar uchun, to'g'ri type qaytaradi (masalan, `input.checked` boolean).

`dataset` — `data-*` custom attribute'lar uchun maxsus API. `data-user-id="42"` → `element.dataset.userId` (camelCase ga o'tadi). Dataset orqali HTML dan JavaScript'ga ma'lumot uzatish — toza va standart usul.

<details>
<summary><strong>Under the Hood</strong></summary>

Element attribute'lari ichki ravishda `NamedNodeMap` collection'ida saqlanadi — `element.attributes` orqali kirish mumkin. Har bir attribute `Attr` node (nodeType: 2) sifatida mavjud bo'lib, `name` va `value` property'lariga ega. `getAttribute()`/`setAttribute()` shu `NamedNodeMap` bilan ishlaydi va doim string qaytaradi.

DOM property va HTML attribute sinxronizatsiyasi murakkab. Ba'zi attribute'lar **one-way sync** — `input.value` property o'zgartirilsa, `getAttribute("value")` eski boshlang'ich qiymatni qaytaradi. Ba'zilari **two-way sync** — `id`, `className` property o'zgartirilsa, attribute ham yangilanadi. `checked`, `selected`, `disabled` kabi boolean attribute'lar uchun attribute mavjudligi muhim (qiymati emas) — `setAttribute("disabled", "")` va `setAttribute("disabled", "false")` ikkalasi ham elementni disabled qiladi.

`dataset` property `DOMStringMap` interface'ini implement qiladi. `data-*` attribute'larning nomlari kebab-case dan camelCase ga mapping qilinadi: `data-user-id` → `dataset.userId`, `data-x` → `dataset.x`. Bu mapping spec da aniq belgilangan — har bir `-` dan keyingi harf uppercase ga aylanadi. `DOMStringMap` orqali yangi property qo'shilsa, DOM da `data-*` attribute yaratiladi. Barcha qiymatlar doim string — type conversion qo'lda qilish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// <input id="email" type="email" value="test@mail.com" disabled data-validation="required" data-max-length="50">
const input = document.getElementById("email");

// getAttribute/setAttribute — string ishlaydi
console.log(input.getAttribute("type"));   // "email"
console.log(input.getAttribute("disabled")); // "" (mavjud — bo'sh string)
input.setAttribute("placeholder", "Email kiriting");
input.removeAttribute("disabled");
console.log(input.hasAttribute("disabled")); // false

// Property vs Attribute farqi:
// Attribute — HTML dagi boshlang'ich qiymat
// Property — JavaScript dagi joriy qiymat
input.value = "yangi@mail.com";
console.log(input.value);                  // "yangi@mail.com" — property (joriy)
console.log(input.getAttribute("value"));  // "test@mail.com" — attribute (boshlang'ich)

// checked — boolean property, string attribute:
// <input type="checkbox" checked>
const checkbox = document.querySelector("input[type=checkbox]");
console.log(checkbox.checked);               // true (boolean)
console.log(checkbox.getAttribute("checked")); // "" (string)

// ─── dataset — data-* attribute'lar ───
// data-validation="required" → dataset.validation
// data-max-length="50" → dataset.maxLength (camelCase!)
console.log(input.dataset.validation);  // "required"
console.log(input.dataset.maxLength);   // "50" (string!)

// Yangi data attribute qo'shish:
input.dataset.errorMessage = "Email noto'g'ri";
// HTML da: data-error-message="Email noto'g'ri" (kebab-case)

// dataset orqali component'lar arasi ma'lumot uzatish:
// <button data-action="delete" data-item-id="42">O'chirish</button>
document.addEventListener("click", (e) => {
  const btn = e.target.closest("[data-action]");
  if (btn) {
    const { action, itemId } = btn.dataset;
    console.log(action, itemId); // "delete", "42"
  }
});
```

</details>

---

## ClassList API

### Nazariya

`classList` — element'ning CSS class'larini boshqarish uchun API. `className` (string) bilan ishlashdan ancha qulay va xavfsiz — `className` da class qo'shganda mavjud class'larni tasodifan o'chirib yuborish mumkin, `classList` da bunday xavf yo'q.

| Metod | Tavsif |
|-------|--------|
| `add("cls1", "cls2")` | Class qo'shadi (mavjud bo'lsa o'tkazib yuboradi) |
| `remove("cls1", "cls2")` | Class o'chiradi (yo'q bo'lsa xato bermaydi) |
| `toggle("cls")` | Bor → o'chiradi, yo'q → qo'shadi. Boolean qaytaradi |
| `toggle("cls", force)` | `force=true` → doim qo'shadi, `false` → doim o'chiradi |
| `contains("cls")` | Class bormi? Boolean qaytaradi |
| `replace("old", "new")` | Bir classni boshqasiga almashtiradi |

<details>
<summary><strong>Under the Hood</strong></summary>

`classList` property `DOMTokenList` interface'ini qaytaradi. `DOMTokenList` ichki ravishda `className` string'ni token'larga (space-separated) ajratib boshqaradi. `add()`, `remove()`, `toggle()` chaqirilganda `DOMTokenList` avval token mavjudligini tekshiradi (Set semantikasi — duplicate qo'shilmaydi), keyin `className` attribute'ni yangilaydi.

`className` direct assignment (`element.className = "new"`) bilan solishtirganda `classList` operatsiyalari aniqroq va xavfsizroq — `className` butun string'ni overwrite qiladi, `classList.add()` esa mavjud class'larga zarar bermasdan yangi token qo'shadi. Performance jihatdan `classList.add("a", "b")` bitta DOM attribute mutation trigger qiladi — ikki marta `classList.add()` chaqirish esa ikkita mutation.

`classList` operatsiyalari DOM attribute'ni o'zgartirgani uchun `MutationObserver` bilan kuzatish mumkin (`attributeFilter: ["class"]`). Brauzer rendering pipeline'da class o'zgarishi CSS selector recalculation'ni trigger qiladi — agar o'zgargan class layout property'larni o'zgartirsa reflow, faqat visual property'larni o'zgartirsa repaint bo'ladi. Shuning uchun animatsiyalar uchun `transform` va `opacity` kabi compositor-only property'lar bilan ishlaydigan class'lar afzal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const card = document.querySelector(".card");

// Class qo'shish/o'chirish
card.classList.add("active", "highlighted"); // ikkalasini qo'shadi
card.classList.remove("old", "deprecated");  // ikkalasini o'chiradi

// toggle — dark mode switch
const body = document.body;
const isDark = body.classList.toggle("dark-mode");
console.log(isDark); // true (qo'shildi) yoki false (o'chirildi)

// toggle bilan force — conditional class
const isValid = form.checkValidity();
input.classList.toggle("error", !isValid);   // xato bo'lsa qo'shadi
input.classList.toggle("success", isValid);  // to'g'ri bo'lsa qo'shadi

// contains — class tekshirish
if (card.classList.contains("active")) {
  console.log("Karta faol");
}

// replace — almashtirish
card.classList.replace("loading", "loaded");

// ❌ className bilan xavfli:
card.className = "new-class"; // ⚠️ barcha eski class'lar O'CHDI!
// ✅ className bilan to'g'ri:
card.className += " new-class"; // mavjud class'larga qo'shadi (lekin classList yaxshiroq)
```

</details>

---

## Style Manipulation

### Nazariya

CSS style'larni JavaScript orqali boshqarishning ikki usuli bor: **inline style** (`element.style`) va **computed style** (`getComputedStyle()`).

`element.style` — faqat **inline style** o'qiydi/yozadi. CSS property nomlari camelCase da yoziladi (`background-color` → `backgroundColor`). Bu usul faqat element'ga bevosita qo'yilgan inline style'ni ko'rsatadi — CSS fayldan kelgan style'larni ko'rsatmaydi.

`getComputedStyle(element)` — element'ga **haqiqatda qo'llangan** barcha style'larni qaytaradi (CSS fayl, inline, inherited — hammasi hisobga olinadi). Faqat o'qish mumkin — yozib bo'lmaydi. `getComputedStyle` chaqirilganda brauzer **reflow** qilishi mumkin — performance uchun kamroq ishlatish kerak.

CSS Custom Properties (`--var`) ni JavaScript orqali o'qish va yozish mumkin — bu theme switching, dynamic animation va boshqa holatlarda foydali.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const card = document.querySelector(".card");

// Inline style yozish:
card.style.backgroundColor = "#f0f0f0"; // background-color → camelCase
card.style.padding = "20px";
card.style.borderRadius = "8px";        // border-radius → camelCase
card.style.display = "none";            // yashirish

// Bir nechta style birdan — cssText:
card.style.cssText = "background: red; padding: 10px; color: white;";
// ⚠️ cssText mavjud inline style'larni TO'LIQ ALMASHTIRADI

// Inline style o'chirish:
card.style.display = ""; // bo'sh string — inline style o'chadi, CSS dagi qo'llanadi

// getComputedStyle — haqiqiy qo'llangan style:
const computed = getComputedStyle(card);
console.log(computed.backgroundColor); // "rgb(240, 240, 240)"
console.log(computed.padding);         // "20px"
console.log(computed.fontSize);        // CSS dan kelgan qiymat ham ko'rinadi

// CSS Custom Properties (CSS variables):
// CSS: :root { --primary: #3498db; --spacing: 16px; }
const root = document.documentElement;

// O'qish:
const primary = getComputedStyle(root).getPropertyValue("--primary");
console.log(primary); // "#3498db"

// Yozish — theme switch:
root.style.setProperty("--primary", "#e74c3c"); // qizilga o'zgardi
root.style.setProperty("--spacing", "24px");

// Pseudo-element style o'qish:
const beforeStyle = getComputedStyle(card, "::before");
console.log(beforeStyle.content); // pseudo-element content'i
```

</details>

---

## Performance: Reflow va Repaint

### Nazariya

DOM manipulation'ning eng qimmat jihati — brauzerning rendering pipeline'ini trigger qilish. Ikki asosiy jarayon bor:

**Reflow (Layout)** — brauzer element'larning pozitsiyasi va o'lchamini qayta hisoblaydi. Bu **eng qimmat** operatsiya — bir elementning o'zgarishi butun sahifaga ta'sir qilishi mumkin (masalan, `div` kengaysa, pastdagi barcha elementlar pastga suriladi). Reflow trigger qiluvchi operatsiyalar: `width/height`, `padding/margin/border`, `display`, `position`, `font-size`, `offsetWidth/Height` (o'qish ham!), `getComputedStyle()`, `scrollTop`, `getBoundingClientRect()`.

**Repaint** — element'ning **ko'rinishi** o'zgaradi, lekin layout o'zgarmaydi. Reflow'dan arzon. Repaint trigger qiluvchi: `color`, `background`, `visibility`, `box-shadow`, `border-color`, `outline`.

**Layout thrashing** — JavaScript da DOM o'qish va yozish operatsiyalari aralash bajarilganda, har bir o'qishda brauzer avvalgi yozishlarni flush qilishi kerak. Bu ko'p reflow'ga olib keladi.

<details>
<summary><strong>Under the Hood</strong></summary>

```
Reflow va Repaint farqi:
┌─────────────────────────────────────────────────────────────┐
│ DOM O'zgardi                                                 │
│     │                                                        │
│     ▼                                                        │
│ Layout o'zgardimi? (size, position, display...)              │
│     │YES                    │NO                              │
│     ▼                       ▼                                │
│  REFLOW                  Ko'rinish o'zgardimi?               │
│  (Layout qayta hisob)    (color, background...)              │
│     │                       │YES           │NO               │
│     ▼                       ▼              ▼                 │
│  REPAINT                 REPAINT         Hech narsa          │
│  (Piksellar qayta chizish)                                   │
│     │                       │                                │
│     ▼                       ▼                                │
│  COMPOSITE (layerlarni birlashtirish)                        │
└─────────────────────────────────────────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Layout thrashing — har bir iteratsiyada reflow:
const items = document.querySelectorAll(".item");
items.forEach(item => {
  const width = item.offsetWidth;    // O'QISH → reflow trigger!
  item.style.width = width * 2 + "px"; // YOZISH
  // Keyingi o'qishda yana reflow — chunki yozish bo'ldi
});

// ✅ To'g'ri — avval hamma o'qish, keyin hamma yozish:
const widths = [];
items.forEach(item => {
  widths.push(item.offsetWidth); // Barcha O'QISHlarni avval
});
items.forEach((item, i) => {
  item.style.width = widths[i] * 2 + "px"; // Barcha YOZISHlarni keyin
});

// ✅ CSS class bilan — bitta reflow:
// ⚠️ Ishlaydi, lekin CSS class bilan yanada toza:
element.style.width = "100px";
element.style.height = "100px";
element.style.margin = "10px";
// (Modern browser'lar buni batch qiladi — bitta reflow bo'ladi,
//  lekin orada offsetWidth kabi o'qish bo'lsa — layout thrashing!)

// ✅ Bitta class bilan barcha style birdan:
element.classList.add("expanded"); // CSS da .expanded { width: 100px; height: 100px; margin: 10px; }

// ✅ cssText bilan:
element.style.cssText = "width: 100px; height: 100px; margin: 10px;";
```

</details>

---

## DocumentFragment

### Nazariya

`DocumentFragment` — DOM da ko'rinmaydigan **virtual container**. Unga element'lar qo'shilganda hech qanday reflow/repaint bo'lmaydi, chunki u hech qayerda ko'rinmaydi. Faqat fragment DOM ga append qilinganda bitta reflow sodir bo'ladi. Ko'p element qo'shishda performance uchun muhim.

Fragment DOM ga append qilinganda, uning **child'lari** ko'chiriladi, fragment o'zi qo'shilmaydi va bo'shalib qoladi. Bu `<div>` wrapper kerak emas degani.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ⚠️ Ishlaydi, lekin DocumentFragment bilan samaraliroq:
const list = document.getElementById("list");
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list.appendChild(li); // Modern browser'lar batch qiladi, lekin overhead ko'p
}

// ✅ DocumentFragment bilan — 1 ta reflow:
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li); // fragment DOM da emas — reflow yo'q
}
list.appendChild(fragment); // bitta reflow — 1000 ta element birdan

// ✅ Alternativa — innerHTML bilan:
const html = Array.from({ length: 1000 }, (_, i) =>
  `<li>Item ${i}</li>`
).join("");
list.innerHTML = html; // bitta parse, bitta reflow

// ✅ Yoki append bilan ko'p argument:
const items = Array.from({ length: 1000 }, (_, i) => {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  return li;
});
list.append(...items); // bitta operatsiya
```

</details>

---

## requestAnimationFrame bilan DOM Update

### Nazariya

`requestAnimationFrame(callback)` — keyingi repaint oldidan callback ni bajaradi. Bu DOM animatsiyalari va vizual update'lar uchun eng to'g'ri usul — brauzer render cycle'iga sinxronlashtirilgan. `setTimeout`/`setInterval` bilan animatsiya qilish noto'g'ri — ular render cycle bilan sinxron emas, frame skip yoki jank (sekin, notekis animatsiya) bo'lishi mumkin.

`requestAnimationFrame` display'ning refresh rate'iga mos ishlaydi — standart 60Hz monitorlarda har ~16.67ms, lekin 90Hz/120Hz/144Hz/240Hz kabi high refresh rate displaylarda mos ravishda tez-tez chaqiriladi (~11ms, ~8ms, ~7ms, ~4ms). Tab background'da bo'lsa avtomatik sekinlashadi yoki to'xtaydi — batareya tejaladi. Callback'ga `DOMHighResTimeStamp` (performance.now() bilan bir xil monotonic timer) argument beriladi — delta-time hisoblashlar uchun bu argumentni ishlatish kerak, wall-clock emas.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ✅ requestAnimationFrame bilan smooth animatsiya:
let position = 0;
const box = document.getElementById("box");

function animate(timestamp) {
  position += 2;
  box.style.transform = `translateX(${position}px)`;

  if (position < 300) {
    requestAnimationFrame(animate); // keyingi frame uchun qayta chaqir
  }
}
requestAnimationFrame(animate);

// ✅ DOM batch update — requestAnimationFrame bilan:
function batchDOMUpdates(updates) {
  requestAnimationFrame(() => {
    updates.forEach(fn => fn()); // barcha update'lar bitta frame'da
  });
}

batchDOMUpdates([
  () => { header.textContent = "Yangi sarlavha"; },
  () => { card.classList.add("active"); },
  () => { footer.style.display = "none"; }
]);

// ❌ setTimeout bilan animatsiya — noto'g'ri:
// setInterval(() => { box.style.left = pos++ + "px"; }, 16);
// Frame rate aniq emas, jank bo'lishi mumkin

// ✅ Cancel qilish:
const animId = requestAnimationFrame(animate);
cancelAnimationFrame(animId); // animatsiyani to'xtatish
```

</details>

---

## Virtual DOM Tushunchasi

### Nazariya

**Virtual DOM** — real DOM ning JavaScript object representatsiyasi. React, Vue kabi framework'lar real DOM bilan to'g'ridan-to'g'ri ishlamaydi — avval Virtual DOM da o'zgarishlarni qiladi, keyin eski va yangi Virtual DOM ni **diff** qiladi (farqlarni topadi), va faqat **o'zgargan qismlarni** real DOM da yangilaydi (patch). Bu minimal DOM manipulation kafolatlaydi.

Virtual DOM nima uchun kerak: real DOM operatsiyalari qimmat (C++ bridge, reflow/repaint). JavaScript object'lar bilan ishlash esa juda tez. 1000 ta element ichidan 3 tasi o'zgarganda — butun ro'yxatni qayta render qilish o'rniga faqat 3 ta DOM node yangilanadi.

Bu framework-specific texnologiya — native JavaScript da Virtual DOM ishlatilmaydi. Kichik dasturlarda direct DOM manipulation yetarli va ba'zan tezroq ham.

<details>
<summary><strong>Kod Misollari</strong></summary>

Virtual DOM konsepti — soddalashtirilgan implementatsiya:

```javascript
// Virtual DOM node — oddiy JavaScript object:
function createElement(tag, props, ...children) {
  return { tag, props: props || {}, children };
}

// Virtual DOM → Real DOM:
function render(vnode) {
  if (typeof vnode === "string") {
    return document.createTextNode(vnode);
  }
  const el = document.createElement(vnode.tag);
  Object.entries(vnode.props).forEach(([key, val]) => {
    if (key.startsWith("on")) {
      el.addEventListener(key.slice(2).toLowerCase(), val);
    } else {
      el.setAttribute(key === "className" ? "class" : key, val);
    }
  });
  vnode.children.forEach(child => el.appendChild(render(child)));
  return el;
}

// Ishlatish:
const app = createElement("div", { className: "app" },
  createElement("h1", {}, "Virtual DOM"),
  createElement("button", {
    onClick: () => alert("Bosildi!")
  }, "Bos"),
  createElement("ul", {},
    ...[1, 2, 3].map(n =>
      createElement("li", {}, `Item ${n}`)
    )
  )
);
document.body.appendChild(render(app));
```

</details>

---

## Web Components

### Nazariya

**Web Components** — brauzer-native komponent tizimi bo'lib, qayta ishlatiluvchi, encapsulated UI elementlarni **hech qanday framework'siz** yaratish imkonini beradi. React, Vue, Angular kabi framework'lar o'z komponent tizimlariga ega, lekin Web Components **brauzer standarti** — barcha zamonaviy brauzerlarda ishlaydi va framework agnostic.

Web Components 3 ta asosiy texnologiyadan iborat:

```
Web Components Arxitekturasi:
┌─────────────────────────────────────────────────────────┐
│                   Web Components                         │
│                                                          │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ Custom Elements │  │  Shadow DOM  │  │  HTML       │  │
│  │                 │  │              │  │  Templates  │  │
│  │ O'z HTML        │  │ Encapsulated │  │             │  │
│  │ tag'larini      │  │ DOM tree +   │  │ <template>  │  │
│  │ yaratish        │  │ Style        │  │ <slot>      │  │
│  │                 │  │ isolation    │  │             │  │
│  └─────────────────┘  └──────────────┘  └────────────┘  │
│           │                   │                │         │
│           └───────────────────┼────────────────┘         │
│                               │                          │
│                    ┌──────────▼──────────┐               │
│                    │  Reusable Custom    │               │
│                    │  HTML Element       │               │
│                    │  <my-component>     │               │
│                    └─────────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

**1. Custom Elements** — o'z HTML teglarini yaratish mexanizmi. `customElements.define()` bilan yangi element ro'yxatdan o'tkaziladi. Element nomi **albatta tire (-) bo'lishi kerak** (`my-card`, `app-header`) — bu brauzer native elementlardan ajratish uchun. Ikki turi bor:
- **Autonomous elements** — to'liq yangi element (`extends HTMLElement`)
- **Customized built-in elements** — mavjud elementni kengaytirish (`extends HTMLButtonElement`, `is="my-button"`) — Safari qo'llab-quvvatlamaydi

**2. Shadow DOM** — element ichida encapsulated DOM tree yaratish. Shadow DOM ichidagi style'lar tashqariga ta'sir qilmaydi, tashqi style'lar ichkariga ta'sir qilmaydi — **to'liq style isolation**. Bu komponentlar bir-birining CSS ini buzmasligini kafolatlaydi.

**3. HTML Templates** — `<template>` va `<slot>` elementlari. `<template>` ichidagi content brauzer tomonidan render qilinmaydi — JavaScript orqali clone qilib ishlatiladi. `<slot>` — tashqaridan content kiritish mexanizmi (React'dagi `children` va `props.children` ga o'xshash mexanizm).

**Lifecycle callbacks** — Custom Element'ning hayot sikli:

```
Custom Element Lifecycle:
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  constructor()          Element yaratildi (new yoki parse)        │
│       │                 ⚠️ DOM ga tegmang! Shadow DOM attach      │
│       │                 qilish va event listener'lar o'rnatish OK │
│       ▼                                                           │
│  connectedCallback()    DOM ga qo'shildi                          │
│       │                 ✅ DOM bilan ishlash, fetch, render        │
│       │                 Har safar DOM ga qo'shilganda chaqiriladi │
│       ▼                                                           │
│  attributeChangedCallback(name, oldVal, newVal)                   │
│       │                 Kuzatilgan attribute o'zgardi              │
│       │                 Faqat observedAttributes dagi attribute'lar│
│       ▼                                                           │
│  disconnectedCallback() DOM dan olib tashlandi                    │
│       │                 ✅ Cleanup: listener, timer, subscription │
│       ▼                                                           │
│  adoptedCallback()      Boshqa document ga ko'chirildi (kam)      │
│                         iframe orasida ko'chirganda               │
└──────────────────────────────────────────────────────────────────┘
```

**Shadow DOM vs Light DOM:**

```
┌─ <my-card> ──────────────────────────────────────┐
│                                                    │
│  Light DOM (tashqi, foydalanuvchi yozadi):          │
│  ┌──────────────────────────────────────┐          │
│  │ <span slot="title">Salom</span>     │          │
│  │ <p>Content...</p>                    │          │
│  └──────────────────────────────────────┘          │
│                                                    │
│  Shadow DOM (ichki, komponent yaratuvchi yozadi):   │
│  ┌──────────────────────────────────────┐          │
│  │ #shadow-root                         │          │
│  │ ├── <style> ... </style>             │  ← Isolated │
│  │ ├── <header>                         │          │
│  │ │   └── <slot name="title"></slot>   │  ← Light DOM dan oladi │
│  │ └── <div class="body">              │          │
│  │     └── <slot></slot>               │  ← Default slot │
│  └──────────────────────────────────────┘          │
└────────────────────────────────────────────────────┘
```

**CSS va Shadow DOM:**
- **CSS Custom Properties** (`--var`) Shadow DOM chegarasini kesib o'tadi — theming uchun asosiy mexanizm
- `:host` — shadow host elementini ichkaridan style qilish
- `:host-context(.dark)` — ota elementning class'iga qarab style
- `::slotted(selector)` — slot'ga tushgan light DOM elementlarini style qilish
- Tashqi CSS selektorlar shadow DOM ichiga kirmaydi

**Events va Shadow DOM:** Shadow DOM ichidan dispatch qilingan eventlar `composed: true` bo'lsa shadow boundary'ni kesib chiqadi. `event.composedPath()` event'ning to'liq yo'lini ko'rsatadi (shadow DOM ichini ham). Native eventlar (click, input, focus) default `composed: true`. Custom eventlar uchun `composed: true` ni qo'lda belgilash kerak.

**Form-associated custom elements** — `ElementInternals` API orqali custom element'lar `<form>` bilan integratsiya qila oladi: validation, form data, submit — native `<input>` kabi ishlaydi.

**Framework vs Web Components:**
- Web Components — **UI primitives** uchun ideal (design system, shareable widgets)
- Framework'lar — **application logic** uchun kuchli (state management, routing, SSR)
- Eng yaxshi amaliyot — ikkalasini birga ishlatish: Web Components design system, framework app logic

<details>
<summary><strong>Under the Hood</strong></summary>

Brauzer Custom Element ni qanday ro'yxatdan o'tkazadi:

```
customElements.define() jarayoni:
┌───────────────────┐     ┌──────────────────────┐
│ customElements    │     │  Custom Element       │
│ .define(          │ →   │  Registry             │
│   "my-card",      │     │  ┌────────┬─────────┐ │
│   MyCard          │     │  │  name  │  class   │ │
│ )                 │     │  ├────────┼─────────┤ │
│                   │     │  │my-card │ MyCard   │ │
│                   │     │  │my-btn  │ MyButton │ │
└───────────────────┘     │  └────────┴─────────┘ │
                          └──────────────────────┘
                                    │
                                    ▼
                          HTML Parser topganda:
                          <my-card> → Registry'dan
                          MyCard class ni oladi →
                          new MyCard() → upgrade
```

Element **upgrade** mexanizmi: Agar HTML da `<my-card>` yozilgan bo'lsa, lekin JavaScript hali `customElements.define()` chaqirmagan bo'lsa — brauzer uni `HTMLUnknownElement` sifatida yaratadi. `define()` chaqirilganda barcha mavjud `<my-card>` elementlari **upgrade** qilinadi — class almashtiriladi va `constructor` + `connectedCallback` chaqiriladi. `customElements.whenDefined('my-card')` — element define bo'lguncha kutadigan Promise qaytaradi.

Shadow DOM attachment — `attachShadow({ mode: 'open' })` chaqirilganda brauzer element ichida alohida DOM tree yaratadi. `mode: 'open'` — `element.shadowRoot` public property orqali tashqaridan kirish mumkin. `mode: 'closed'` — `element.shadowRoot` `null` qaytaradi va standart DOM API orqali ichkariga kirish mumkin emas. Lekin bu **soft encapsulation**: determined kod `attachShadow` method'ini `Element.prototype` darajasida patch qilib, har bir closed root reference'ni sirtqi `WeakMap`'ga saqlashi mumkin — ya'ni "xavfsizlik" chegarasi emas, faqat accidental access'ga qarshi. Ko'p hollarda `open` afzal — debugging va DevTools integratsiyasi oddiyroq, va closed mode'ning haqiqiy encapsulation kafolati yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ═══════════════════════════════════════════════════════
// 1. SIMPLE CUSTOM ELEMENT — Counter Component
// ═══════════════════════════════════════════════════════

class MyCounter extends HTMLElement {
  constructor() {
    super(); // ⚠️ Albatta super() birinchi!
    this._count = 0;

    // Shadow DOM yaratish
    this.attachShadow({ mode: "open" });

    // Template
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: inline-block;
          font-family: sans-serif;
        }
        .counter {
          display: flex;
          align-items: center;
          gap: 12px;
          padding: 12px;
          border: 2px solid #3498db;
          border-radius: 8px;
        }
        button {
          padding: 6px 14px;
          font-size: 18px;
          cursor: pointer;
          border: 1px solid #ccc;
          border-radius: 4px;
          background: #ecf0f1;
        }
        button:hover { background: #3498db; color: white; }
        .value {
          font-size: 24px;
          font-weight: bold;
          min-width: 40px;
          text-align: center;
        }
      </style>
      <div class="counter">
        <button id="dec">−</button>
        <span class="value">0</span>
        <button id="inc">+</button>
      </div>
    `;
  }

  // DOM ga qo'shilganda
  connectedCallback() {
    this.shadowRoot.getElementById("inc")
      .addEventListener("click", () => this._update(1));
    this.shadowRoot.getElementById("dec")
      .addEventListener("click", () => this._update(-1));

    // Boshlang'ich qiymatni attribute dan olish
    if (this.hasAttribute("value")) {
      this._count = parseInt(this.getAttribute("value")) || 0;
      this._render();
    }
  }

  _update(delta) {
    this._count += delta;
    this._render();
    // Custom event dispatch — tashqi kod eshitishi mumkin
    this.dispatchEvent(new CustomEvent("change", {
      detail: { value: this._count },
      bubbles: true,
      composed: true // Shadow DOM dan chiqadi
    }));
  }

  _render() {
    this.shadowRoot.querySelector(".value").textContent = this._count;
  }
}

customElements.define("my-counter", MyCounter);

// HTML da ishlatish:
// <my-counter value="5"></my-counter>

// JavaScript dan ishlatish:
const counter = document.createElement("my-counter");
counter.setAttribute("value", "10");
document.body.appendChild(counter);

counter.addEventListener("change", (e) => {
  console.log("Counter qiymati:", e.detail.value);
});
```

```javascript
// ═══════════════════════════════════════════════════════
// 2. SHADOW DOM — Style Isolation
// ═══════════════════════════════════════════════════════

class StyledCard extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: "open" });

    shadow.innerHTML = `
      <style>
        /* :host — komponent elementining o'zini style qiladi */
        :host {
          display: block;
          margin: 10px;
        }

        /* :host() — shartli style (host da class bo'lsa) */
        :host(.highlighted) {
          border: 3px solid gold;
        }

        /* :host-context() — ota elementga qarab style */
        :host-context(.dark-theme) .card {
          background: #2c3e50;
          color: #ecf0f1;
        }

        /* Bu style FAQAT shadow DOM ichida ishlaydi */
        /* Tashqi .card class'larga TA'SIR QILMAYDI */
        .card {
          padding: 20px;
          border-radius: 12px;
          background: white;
          box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        }

        h2 {
          margin-top: 0;
          color: #2c3e50;
        }

        /* ::slotted() — slot ga tushgan elementlarni style qiladi */
        ::slotted(p) {
          color: #7f8c8d;
          line-height: 1.6;
        }

        ::slotted([slot="footer"]) {
          border-top: 1px solid #eee;
          padding-top: 10px;
          font-size: 14px;
        }
      </style>
      <div class="card">
        <h2><slot name="title">Default Title</slot></h2>
        <div class="body">
          <slot></slot>
        </div>
        <slot name="footer"></slot>
      </div>
    `;
  }
}

customElements.define("styled-card", StyledCard);

// HTML ishlatish:
// <styled-card class="highlighted">
//   <span slot="title">Mening Kartam</span>
//   <p>Bu karta ichidagi matn. Tashqi CSS bu matn ga ta'sir qilmaydi.</p>
//   <p>Ikkinchi paragraf.</p>
//   <small slot="footer">Yaratildi: 2024</small>
// </styled-card>

// ✅ Tashqi CSS shadow DOM ichiga KIRMAYDI:
// <style>
//   h2 { color: red; }     /* Shadow DOM ichidagi h2 ga ta'sir QILMAYDI */
//   .card { padding: 0; }  /* Shadow DOM ichidagi .card ga ta'sir QILMAYDI */
// </style>

// ✅ CSS Custom Properties shadow DOM orqali O'TADI:
// <style>
//   :root { --primary-color: #e74c3c; }
// </style>
// Shadow DOM ichida: color: var(--primary-color); /* Ishlaydi! */
```

```javascript
// ═══════════════════════════════════════════════════════
// 3. TEMPLATE + SLOT — Content Projection
// ═══════════════════════════════════════════════════════

// HTML da template e'lon qilish:
// <template id="user-card-template">
//   <style>
//     .user-card { display: flex; gap: 16px; padding: 16px;
//                  border: 1px solid #ddd; border-radius: 8px; }
//     .avatar { width: 60px; height: 60px; border-radius: 50%;
//               object-fit: cover; }
//     .info { flex: 1; }
//     ::slotted(h3) { margin: 0 0 4px; }
//     ::slotted(.role) { color: #3498db; font-weight: bold; }
//   </style>
//   <div class="user-card">
//     <slot name="avatar">
//       <img class="avatar" src="default-avatar.png" alt="avatar">
//     </slot>
//     <div class="info">
//       <slot name="name"><h3>Noma'lum</h3></slot>
//       <slot name="role"><span class="role">Foydalanuvchi</span></slot>
//       <slot></slot>
//     </div>
//   </div>
// </template>

class UserCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });

    // Template ni clone qilib ishlatish
    const template = document.getElementById("user-card-template");
    // ⚠️ cloneNode(true) — deep clone, har bir instance uchun yangi nusxa
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}

customElements.define("user-card", UserCard);

// HTML da ishlatish (named slots bilan):
// <user-card>
//   <img slot="avatar" src="ali.jpg" alt="Ali" style="width:60px;height:60px;border-radius:50%">
//   <h3 slot="name">Ali Valiyev</h3>
//   <span slot="role" class="role">Senior Developer</span>
//   <p>JavaScript va Web Components mutaxassisi</p>   ← default slot ga tushadi
// </user-card>

// Slot tayinlanmagan content — default (nomsiz) slot ga tushadi
// Slot mavjud bo'lmasa — fallback content ko'rsatiladi ("Noma'lum")

// ─── JavaScript orqali template yaratish (HTML kerak emas) ───
class AlertBox extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: "open" });

    // Template ni JavaScript da yaratish
    const template = document.createElement("template");
    template.innerHTML = `
      <style>
        .alert {
          padding: 12px 20px;
          border-radius: 6px;
          margin: 8px 0;
          font-family: sans-serif;
        }
        :host([type="success"]) .alert {
          background: #d4edda; color: #155724; border: 1px solid #c3e6cb;
        }
        :host([type="error"]) .alert {
          background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb;
        }
        :host([type="warning"]) .alert {
          background: #fff3cd; color: #856404; border: 1px solid #ffeeba;
        }
        .close-btn {
          float: right; cursor: pointer; font-size: 18px;
          background: none; border: none; color: inherit;
        }
      </style>
      <div class="alert">
        <button class="close-btn">&times;</button>
        <slot></slot>
      </div>
    `;
    shadow.appendChild(template.content.cloneNode(true));
  }

  connectedCallback() {
    this.shadowRoot.querySelector(".close-btn")
      .addEventListener("click", () => {
        this.dispatchEvent(new CustomEvent("close", { bubbles: true, composed: true }));
        this.remove();
      });
  }
}

customElements.define("alert-box", AlertBox);

// Ishlatish:
// <alert-box type="success">Muvaffaqiyatli saqlandi!</alert-box>
// <alert-box type="error">Xatolik yuz berdi.</alert-box>
// <alert-box type="warning">Disk joy tugamoqda!</alert-box>
```

```javascript
// ═══════════════════════════════════════════════════════
// 4. attributeChangedCallback — Reactive Updates
// ═══════════════════════════════════════════════════════

class ProgressBar extends HTMLElement {
  // ⚠️ Static getter — faqat shu attribute'lar kuzatiladi
  static get observedAttributes() {
    return ["value", "max", "color", "label"];
  }

  constructor() {
    super();
    this.attachShadow({ mode: "open" });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          margin: 8px 0;
          font-family: sans-serif;
        }
        .container {
          background: #ecf0f1;
          border-radius: 10px;
          overflow: hidden;
          height: 24px;
          position: relative;
        }
        .bar {
          height: 100%;
          border-radius: 10px;
          transition: width 0.3s ease, background-color 0.3s ease;
          background-color: var(--bar-color, #3498db);
        }
        .label {
          position: absolute;
          top: 50%;
          left: 50%;
          transform: translate(-50%, -50%);
          font-size: 12px;
          font-weight: bold;
          color: #333;
        }
        .title {
          margin-bottom: 4px;
          font-size: 14px;
          color: #555;
        }
      </style>
      <div class="title"></div>
      <div class="container">
        <div class="bar"></div>
        <span class="label"></span>
      </div>
    `;
  }

  connectedCallback() {
    this._render();
  }

  // Har safar kuzatilgan attribute o'zgarganda chaqiriladi
  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue === newValue) return; // O'zgarish yo'q
    console.log(`Attribute "${name}": "${oldValue}" → "${newValue}"`);
    this._render();
  }

  _render() {
    const value = parseFloat(this.getAttribute("value")) || 0;
    const max = parseFloat(this.getAttribute("max")) || 100;
    const color = this.getAttribute("color");
    const label = this.getAttribute("label") || "";

    const percent = Math.min(100, Math.max(0, (value / max) * 100));

    const bar = this.shadowRoot.querySelector(".bar");
    const labelEl = this.shadowRoot.querySelector(".label");
    const titleEl = this.shadowRoot.querySelector(".title");

    bar.style.width = `${percent}%`;
    if (color) bar.style.setProperty("--bar-color", color);
    labelEl.textContent = `${Math.round(percent)}%`;
    titleEl.textContent = label;
  }

  // ─── Property ↔ Attribute sinxronizatsiya ───
  get value() { return parseFloat(this.getAttribute("value")) || 0; }
  set value(val) { this.setAttribute("value", val); } // → attributeChangedCallback

  get max() { return parseFloat(this.getAttribute("max")) || 100; }
  set max(val) { this.setAttribute("max", val); }
}

customElements.define("progress-bar", ProgressBar);

// HTML:
// <progress-bar value="35" max="100" color="#2ecc71" label="Yuklash"></progress-bar>

// JavaScript bilan dynamic update:
const bar = document.querySelector("progress-bar");

// Attribute orqali — attributeChangedCallback chaqiriladi:
bar.setAttribute("value", "75");

// Property orqali — setter setAttribute chaqiradi → callback:
bar.value = 90;
bar.max = 200;

// Animatsiya:
let progress = 0;
const interval = setInterval(() => {
  progress += 2;
  bar.value = progress;
  if (progress >= 100) clearInterval(interval);
}, 50);
```

```javascript
// ═══════════════════════════════════════════════════════
// 5. FORM-ASSOCIATED Custom Element (Basic)
// ═══════════════════════════════════════════════════════

class StarRating extends HTMLElement {
  // Form bilan bog'liq ekanligini bildiradi
  static formAssociated = true;

  static get observedAttributes() {
    return ["value", "max"];
  }

  constructor() {
    super();
    // ElementInternals — form integration uchun
    this._internals = this.attachInternals();
    this._value = 0;
    this._max = 5;

    this.attachShadow({ mode: "open" });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: inline-block;
          cursor: pointer;
          font-size: 24px;
          user-select: none;
        }
        .stars { display: flex; gap: 2px; }
        .star { transition: color 0.15s; color: #ddd; }
        .star.active { color: #f39c12; }
        .star:hover { transform: scale(1.2); }
        :host([disabled]) {
          pointer-events: none;
          opacity: 0.5;
        }
      </style>
      <div class="stars" role="radiogroup" aria-label="Yulduz reytingi"></div>
    `;
  }

  connectedCallback() {
    this._max = parseInt(this.getAttribute("max")) || 5;
    this._renderStars();
    this._internals.setFormValue(this._value);
  }

  attributeChangedCallback(name, oldVal, newVal) {
    if (name === "value") {
      this._value = parseInt(newVal) || 0;
      this._updateStars();
      this._internals.setFormValue(this._value);
    }
    if (name === "max") {
      this._max = parseInt(newVal) || 5;
      this._renderStars();
    }
  }

  _renderStars() {
    const container = this.shadowRoot.querySelector(".stars");
    container.innerHTML = "";
    for (let i = 1; i <= this._max; i++) {
      const star = document.createElement("span");
      star.className = "star";
      star.textContent = "★";
      star.dataset.value = i;
      star.addEventListener("click", () => this._select(i));
      container.appendChild(star);
    }
    this._updateStars();
  }

  _select(val) {
    this._value = val;
    this.setAttribute("value", val);
    // Form qiymatini yangilash
    this._internals.setFormValue(val);
    // Validation
    if (val === 0) {
      this._internals.setValidity(
        { valueMissing: true },
        "Iltimos reyting tanlang",
        this.shadowRoot.querySelector(".stars")
      );
    } else {
      this._internals.setValidity({});
    }
    this.dispatchEvent(new CustomEvent("change", {
      detail: { value: val }, bubbles: true, composed: true
    }));
  }

  _updateStars() {
    const stars = this.shadowRoot.querySelectorAll(".star");
    stars.forEach((star, i) => {
      star.classList.toggle("active", i < this._value);
    });
  }

  // Form reset qilinganda chaqiriladi
  formResetCallback() {
    this._value = 0;
    this.removeAttribute("value");
    this._updateStars();
    this._internals.setFormValue(0);
  }

  // Form state restore (bfcache, form autofill)
  formStateRestoreCallback(state, mode) {
    this._value = parseInt(state) || 0;
    this._updateStars();
  }

  // Form disabled state
  formDisabledCallback(disabled) {
    if (disabled) this.setAttribute("disabled", "");
    else this.removeAttribute("disabled");
  }

  // Property access
  get value() { return this._value; }
  set value(val) { this.setAttribute("value", val); }
}

customElements.define("star-rating", StarRating);

// Form ichida ishlatish — native input kabi ishlaydi:
// <form id="review-form">
//   <label>Reyting:</label>
//   <star-rating name="rating" max="5" required></star-rating>
//   <button type="submit">Yuborish</button>
//   <button type="reset">Tozalash</button>
// </form>

const form = document.getElementById("review-form");
form.addEventListener("submit", (e) => {
  e.preventDefault();
  const formData = new FormData(form);
  console.log("Reyting:", formData.get("rating")); // "4" — native form kabi!
});

// ─── Events va Shadow DOM ───
// composed: true — event shadow boundary dan chiqadi
document.querySelector("star-rating").addEventListener("change", (e) => {
  console.log("Tanlangan:", e.detail.value);
  console.log("Composed path:", e.composedPath()); // to'liq yo'l
});
```

```javascript
// ═══════════════════════════════════════════════════════
// QO'SHIMCHA: customElements API
// ═══════════════════════════════════════════════════════

// Element define bo'lganini tekshirish
console.log(customElements.get("my-counter"));
// → MyCounter class (yoki undefined agar define qilinmagan bo'lsa)

// Element define bo'lguncha kutish
customElements.whenDefined("my-counter").then(() => {
  console.log("my-counter tayyor!");
});

// ─── Upgrade mexanizmi ───
// HTML da <my-widget> bor, lekin hali define qilinmagan:
const widget = document.querySelector("my-widget");
console.log(widget instanceof HTMLElement); // true (hali upgrade qilinmagan — HTMLElement sifatida)
// Eslatma: tire(-) bor custom element'lar HTMLElement bo'ladi, HTMLUnknownElement EMAS
// HTMLUnknownElement faqat tire'siz noma'lum tag'lar uchun (<foo> kabi)

class MyWidget extends HTMLElement {
  connectedCallback() {
    this.textContent = "Widget tayyor!";
  }
}
customElements.define("my-widget", MyWidget);
// Endi barcha mavjud <my-widget> lar avtomatik upgrade bo'ladi

// ─── Lazy loading bilan ───
// HTML: <my-heavy-component></my-heavy-component>
customElements.whenDefined("my-heavy-component").then(() => {
  // Component tayyor bo'lganda UI yangilash
  document.querySelector(".loading-spinner").remove();
});

// Alohida faylda: my-heavy-component.js
// import("./my-heavy-component.js"); — kerak bo'lganda yuklash

// ─── CSS :defined pseudo-class ───
// Upgrade bo'lmagan elementlarni style qilish:
// my-widget:not(:defined) {
//   opacity: 0.5;
//   pointer-events: none;
// }
// my-widget:defined {
//   opacity: 1;
// }
```

</details>

---

## Observer APIs — Mutation, Intersection, Resize

### Nazariya

Zamonaviy web dasturlarda "qachondir nimadir o'zgarsa — reaksiya qil" pattern'i juda keng tarqalgan: element ekranga kirdimi (lazy loading), DOM o'zgardimi (reactive sync), element hajmi o'zgardimi (responsive layout). Eski davrda bu muammolar polling (`setInterval` bilan tekshirish) yoki `scroll`/`resize` event'lariga tayanish bilan hal qilinardi — ikkalasi ham performance'ga yomon ta'sir qiladi (main thread busy, layout thrashing).

Modern browser'lar **Observer API'lari** oilasini taqdim etadi — ular **off-main-thread** yoki **lazy** ishlaydi va callback'lar **microtask**/**idle time**'da chaqiriladi. Bu API'lar barcha polling va scroll listener'larga nisbatan samaraliroq.

**3 ta asosiy Observer:**

| Observer | Nimani kuzatadi | Use case |
|----------|-----------------|----------|
| `MutationObserver` | DOM tree o'zgarishlari (child, attribute, text) | Reactive UI sync, third-party script monitoring, form auto-save |
| `IntersectionObserver` | Element viewport yoki ancestor bilan kesishuvi | Lazy loading, infinite scroll, visibility tracking, ad impression |
| `ResizeObserver` | Element hajmi (border-box yoki content-box) o'zgarishi | Responsive components, chart re-render, container queries polyfill |

Barchasi umumiy pattern: `new Observer(callback)` → `.observe(target, options)` → callback'da entries array keladi → `.disconnect()` yoki `.unobserve(target)` bilan to'xtatish.

<details>
<summary><strong>Under the Hood</strong></summary>

**MutationObserver** — DOM spec tomonidan aniqlangan. Brauzer har DOM mutation'da (child qo'shish/o'chirish, attribute o'zgartirish, Text node qiymati) o'zgarishni **mutation record**'ga yozadi. Rendering pipeline'ning oxirida (microtask phase'da) barcha to'plangan record'lar callback'ga **batch** qilib yuboriladi — bu narsa DOM o'zgarishlar seriyasi paytida callback ko'p marta chaqirilishining oldini oladi. `MutationRecord` ichida `type`, `target`, `addedNodes`, `removedNodes`, `attributeName`, `oldValue` bor.

**IntersectionObserver** — W3C spec. Brauzer compositor thread'da element pozitsiyalarini kuzatadi — **layout thread'ni band qilmaydi**. Har target uchun intersection ratio hisoblanadi va `threshold` ga yetganda callback chaqiriladi. Threshold — 0 (element ko'rina boshlaganda), 0.5 (yarmi ko'rinsa), 1 (to'liq ko'rinsa), yoki bir nechta qiymat bilan array. `root` parameter — default viewport, lekin ixtiyoriy scroll container bo'lishi mumkin. `rootMargin` — root'ning margin'i (CSS syntax bilan), masalan `"100px 0px"` — 100px oldin trigger qilish (preloading uchun).

**ResizeObserver** — CSS WG spec. Element hajmi o'zgarganda (CSS, content, window resize, parent layout change — barchasi) trigger bo'ladi. `scroll`/`resize` event'lardan farqli ravishda — faqat kuzatilayotgan element'ga tegishli o'zgarishlarda chaqiriladi, butun window resize'ga emas. Callback rendering cycle'ning **layout phase'idan keyin** ishga tushadi — bu `ResizeObserver` ichida DOM o'qish xavfsiz ekanligini anglatadi. **ResizeObserver loop** — agar callback ichida kuzatilayotgan element hajmini o'zgartirsangiz va bu zanjirli effekt bersa — brauzer "ResizeObserver loop limit exceeded" warning chiqaradi va o'zini to'xtatadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**MutationObserver — form auto-save:**

```javascript
const form = document.querySelector("#editor");

const observer = new MutationObserver((mutations) => {
  // mutations — MutationRecord array
  for (const mutation of mutations) {
    console.log(`${mutation.type} on`, mutation.target);
    if (mutation.type === "characterData") {
      console.log(`Matn: "${mutation.oldValue}" → "${mutation.target.data}"`);
    }
  }
  // Barcha o'zgarishlar batch qilib yuboriladi — har o'zgarish uchun
  // alohida save qilmaslik kerak:
  debouncedSave(form.innerHTML);
});

// Kuzatish boshlash — nimani kuzatish kerakligini aniq ko'rsatish majburiy:
observer.observe(form, {
  childList: true,      // child qo'shish/o'chirish
  subtree: true,        // descendant'larni ham
  attributes: true,     // attribute o'zgarishlari
  attributeOldValue: true, // eski attribute qiymatini ham olish
  characterData: true,  // Text node content
  characterDataOldValue: true,
  // attributeFilter: ["class", "data-state"] // faqat shu attribute'larni
});

// To'xtatish — component unmount da majburiy!
// observer.disconnect();

// Bir martalik takeRecords — disconnect oldidan barcha pending record'lar:
const remaining = observer.takeRecords();
console.log("To'plangan record'lar:", remaining.length);
```

**IntersectionObserver — lazy image loading:**

```javascript
const imageObserver = new IntersectionObserver((entries, observer) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src; // data-src → src (lazy load)
      img.removeAttribute("data-src");
      observer.unobserve(img); // ko'rilgach kuzatishni to'xtatish
    }
  }
}, {
  root: null,              // null = viewport
  rootMargin: "200px 0px", // 200px oldin yuklay boshlash
  threshold: 0.01          // 1% ko'ringach trigger
});

// Barcha lazy image'larni kuzatish:
document.querySelectorAll("img[data-src]").forEach(img => {
  imageObserver.observe(img);
});

// ─── Infinite scroll ───
const sentinel = document.querySelector(".scroll-sentinel");
const loadMoreObserver = new IntersectionObserver(async ([entry]) => {
  if (entry.isIntersecting) {
    const newItems = await fetchNextPage();
    appendToList(newItems);
  }
}, { rootMargin: "400px" });

loadMoreObserver.observe(sentinel);

// ─── Visibility tracking (ad impression) ───
const adObserver = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.intersectionRatio >= 0.5) {
      // Reklama 50% ko'rinadi — impression log
      trackImpression(entry.target.dataset.adId);
    }
  }
}, { threshold: [0, 0.25, 0.5, 0.75, 1] });
```

**ResizeObserver — responsive chart:**

```javascript
const chart = document.querySelector("#chart");

const resizeObserver = new ResizeObserver((entries) => {
  for (const entry of entries) {
    // contentBoxSize/borderBoxSize — zamonaviy API (array)
    const { width, height } = entry.contentRect; // eski API (legacy, lekin simple)

    // Aniqroq — box size arrays:
    // const { inlineSize, blockSize } = entry.contentBoxSize[0];

    chart.width = width;
    chart.height = height;
    redrawChart(); // faqat kerak bo'lgan hajmga
  }
});

resizeObserver.observe(chart);

// Box type tanlash:
// resizeObserver.observe(el, { box: "border-box" });    // default
// resizeObserver.observe(el, { box: "content-box" });
// resizeObserver.observe(el, { box: "device-pixel-content-box" });

// ─── Container queries polyfill pattern ───
// Element kengligi threshold'lardan oshganda class o'zgartirish:
const cardObserver = new ResizeObserver((entries) => {
  for (const { target, contentRect } of entries) {
    target.classList.toggle("narrow", contentRect.width < 300);
    target.classList.toggle("wide",   contentRect.width >= 600);
  }
});

document.querySelectorAll(".responsive-card").forEach(el => cardObserver.observe(el));

// ─── Cleanup — component o'chganda majburiy ───
// resizeObserver.disconnect();  // barcha observer'larni to'xtatish
// resizeObserver.unobserve(el); // bittasini to'xtatish
```

**Observer'lar va Web Components — cleanup pattern:**

```javascript
class ObservedElement extends HTMLElement {
  #mutationObs;
  #resizeObs;
  #intersectionObs;

  connectedCallback() {
    // DOM ga qo'shilganda boshlash
    this.#mutationObs = new MutationObserver(this.#onMutate.bind(this));
    this.#mutationObs.observe(this, { childList: true, subtree: true });

    this.#resizeObs = new ResizeObserver(this.#onResize.bind(this));
    this.#resizeObs.observe(this);

    this.#intersectionObs = new IntersectionObserver(this.#onIntersect.bind(this));
    this.#intersectionObs.observe(this);
  }

  disconnectedCallback() {
    // DOM dan olingach — cleanup MAJBURIY!
    this.#mutationObs?.disconnect();
    this.#resizeObs?.disconnect();
    this.#intersectionObs?.disconnect();
  }

  #onMutate(mutations) { /* ... */ }
  #onResize(entries)   { /* ... */ }
  #onIntersect(entries){ /* ... */ }
}
```

**Observer vs eski usul — performance farqi:**

```javascript
// ❌ Eski usul — scroll event + manual check:
window.addEventListener("scroll", () => {
  document.querySelectorAll("img[data-src]").forEach(img => {
    const rect = img.getBoundingClientRect(); // ⚠️ forced reflow har img uchun!
    if (rect.top < window.innerHeight) {
      img.src = img.dataset.src;
    }
  });
});
// 60 Hz scroll → har frame'da 100 ta img uchun getBoundingClientRect →
// massive layout thrashing, janky scroll, battery drain

// ✅ IntersectionObserver — compositor thread da:
const obs = new IntersectionObserver((entries) => {
  entries.filter(e => e.isIntersecting).forEach(e => {
    e.target.src = e.target.dataset.src;
    obs.unobserve(e.target);
  });
});
document.querySelectorAll("img[data-src]").forEach(img => obs.observe(img));
// Scroll event listener yo'q, main thread bo'sh, kerakli paytda chaqiriladi
```

</details>

---

## Edge Cases va Gotchas

DOM manipulatsiyasi bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha. Har biri spec/browser behavior darajasida sabab bilan.

### Gotcha 1: HTML property `+=` barcha child event listener'larni yo'q qiladi

`element` ning HTML property'siga `+=` orqali qo'shish — ko'pchilik "append" deb o'ylaydi, lekin aslida bu **read + concat + write** sikli. Yozish bosqichida brauzer barcha mavjud child node'larni o'chirib, string'ni qayta parse qiladi va yangi DOM node'lar yaratadi. **Eski node'lardagi barcha event listener'lar yo'qoladi.**

```javascript
const list = document.getElementById("list");
// 3 ta li bor, har birida click listener:
list.querySelectorAll("li").forEach(li => {
  li.addEventListener("click", () => console.log("Clicked:", li.textContent));
});

// ❌ Listener'lar yo'qoladi:
// list.innerHTML = list.innerHTML + "<li>Yangi</li>";  (`+=` ham shu)
// Endi hech qaysi li click listener'iga javob bermaydi!
// Sabab: setter → eski li'lar o'chirildi va yangi parse qilindi

// ✅ To'g'ri usul — yangi node qo'shish:
const newLi = document.createElement("li");
newLi.textContent = "Yangi element";
list.appendChild(newLi); // eski listener'lar saqlanadi ✅

// ✅ Yoki insertAdjacentHTML — sibling'larga tegmaydi:
list.insertAdjacentHTML("beforeend", "<li>Yangi element</li>");
// Eski li'lar va ularning listener'lari saqlanadi ✅
// (lekin yangi li'da listener yo'q — event delegation tavsiya)
```

**Nima uchun:** HTML setter parser'ni ishga tushiradi va barcha descendant'larni yangi node'lar bilan to'liq almashtiradi. Spec darajasida bu "read current HTML → parse new HTML → replace all descendants" sekvensi. `addEventListener` orqali qo'shilgan listener'lar DOM attribute emas, EventTarget'ning ichki ro'yxatida — bu ro'yxat eski node bilan birga yo'qoladi.

**Yechim:** Hech qachon HTML setter'ni `+=` bilan ishlatmang. `append()`, `insertAdjacentHTML()`, `appendChild()` — barchasi sibling'larga tegmaydi. Katta ro'yxatlar uchun event delegation (parent'da bitta listener) va `DocumentFragment` ishlating.

### Gotcha 2: `<template>` content — to'g'ridan-to'g'ri `querySelector` ishlamaydi

`<template>` element'ining ichki content'i **render qilinmaydi** va **parent DOM tree'da ko'rinmaydi** — u `template.content` degan `DocumentFragment` ichida yashaydi. `document.querySelector` bu fragment ichiga kirmaydi. Bu Web Components'da keng tarqalgan xatolik manbai.

```html
<template id="card-template">
  <div class="card">
    <h3 class="title"></h3>
    <p class="body"></p>
  </div>
</template>
```

```javascript
// ❌ Template ichidagi elementga to'g'ridan-to'g'ri kirish:
const title = document.querySelector("#card-template .title");
console.log(title); // null — topilmadi!
// Sabab: document.querySelector template content'ga kirmaydi

// ❌ Template element'ning children property'si ham bo'sh ko'rinadi:
const tpl = document.getElementById("card-template");
console.log(tpl.children.length); // 0 — template.children YO'Q
console.log(tpl.firstElementChild); // null

// ✅ To'g'ri yo'l — .content ichidan olish:
const tplContent = document.getElementById("card-template").content;
console.log(tplContent instanceof DocumentFragment); // true
const titleEl = tplContent.querySelector(".title"); // ✅ ishlaydi

// ✅ Har instance uchun clone qilish:
function createCard(data) {
  const clone = tplContent.cloneNode(true); // deep clone DocumentFragment
  clone.querySelector(".title").textContent = data.title;
  clone.querySelector(".body").textContent = data.body;
  return clone;
  // clone — DocumentFragment, append qilinganda uning children ko'chiriladi
}

document.body.appendChild(createCard({ title: "Salom", body: "Matn" }));
```

**Nima uchun:** HTML spec'da `<template>` element'i "inert" deb belgilangan — brauzer uni parse qiladi, lekin **active document'ga biriktirmaydi**. Uning children'lari `template.content` degan alohida `DocumentFragment`'da saqlanadi. Bu DocumentFragment **passive** — script'lar ishlamaydi, resource'lar yuklanmaydi, CSS qo'llanmaydi. Bu "template pattern" uchun maxsus yaratilgan — HTML'ni rendering'siz parse qilish.

**Yechim:** Template bilan ishlaganda har doim `.content` orqali kiring. `cloneNode(true)` bilan har instance uchun yangi nusxa yarating (template'ni qayta ishlatish uchun). Clone'ni o'zgartirib, keyin DOM'ga qo'shing.

### Gotcha 3: `dataset` qiymatlari doim **string** — `"false"` truthy, `"0"` ham truthy

`data-*` attribute'lar HTML'da yoziladi → DOM'da doim string sifatida saqlanadi → `dataset` ham string qaytaradi. Common xatolik: `data-active="false"` → `dataset.active` → `"false"` (string) → **truthy** (bo'sh string emas). Boolean, number semantikasini kutib hisob-kitob qilganda silent bug hosil bo'ladi.

```html
<div id="widget"
     data-active="false"
     data-count="0"
     data-visible="true"
     data-config='{"theme":"dark"}'>
</div>
```

```javascript
const widget = document.getElementById("widget");

// ❌ Boolean sifatida tekshirish:
if (widget.dataset.active) {
  console.log("Faol!"); // BU BLOK ISHLAYDI!
  // Sabab: dataset.active === "false" (string) — truthy
}

// ❌ Number sifatida:
const count = widget.dataset.count;
console.log(count + 1); // "01" — string concatenation!
console.log(count === 0); // false — string !== number
if (!count) console.log("Bo'sh"); // chiqmaydi — "0" truthy!

// ✅ Explicit parse qilish kerak:
const isActive = widget.dataset.active === "true"; // string compare
const isVisible = widget.dataset.visible === "true";
const count2 = Number(widget.dataset.count);       // 0 (number)
const count3 = +widget.dataset.count;              // 0 (unary +)

// ✅ JSON uchun ham parse kerak:
const config = JSON.parse(widget.dataset.config); // { theme: "dark" }

// ✅ Boolean-like attribute pattern (attribute mavjudligi):
// HTML: <div data-active></div>  (qiymatsiz) yoki yo'q
if (widget.hasAttribute("data-active")) { /* mavjudlik tekshiruv */ }

// ✅ Utility helper:
function parseDataset(element) {
  return {
    active: element.dataset.active === "true",
    count: Number(element.dataset.count) || 0,
    visible: element.dataset.visible !== "false",
    config: element.dataset.config ? JSON.parse(element.dataset.config) : null,
  };
}
```

**Nima uchun:** HTML attribute qiymatlari har doim string — bu HTML spec'ning fundamental qoidasi. `dataset` faqat `data-*` attribute'lar uchun "qulay accessor" — u `getAttribute("data-foo")` bilan ekvivalent, faqat kebab-case → camelCase mapping qiladi. Type konversiya qilmaydi. `"false"`, `"0"`, `"null"` — barchasi truthy string'lar.

**Yechim:** `dataset` qiymatlarini doim explicit parse qiling — `Number()`, `=== "true"`, `JSON.parse()`. Boolean-like attribute'lar uchun `hasAttribute()` ishlatish ma'no jihatdan toza (HTML boolean attribute semantikasi bilan mos).

### Gotcha 4: `getBoundingClientRect()` read-only emas — u forced synchronous layout trigger qiladi

Method nomi "get" deb boshlangani va Promise qaytarmagani uchun ko'pchilik uni arzon sinxron "getter" deb tasavvur qiladi. Aslida **har chaqiriq brauzerga pending DOM mutations'ni flush qilishga majbur qiladi** — chunki aniq pozitsiya/hajm olish uchun to'liq layout hisobi kerak.

```javascript
const items = document.querySelectorAll(".item");

// ❌ Layout thrashing — har iteratsiyada yozish + o'qish:
items.forEach(item => {
  item.style.transform = "translateX(10px)"; // YOZISH (pending)
  const rect = item.getBoundingClientRect(); // O'QISH → FORCED REFLOW!
  // Har getBoundingClientRect chaqirig'ida brauzer:
  // 1. Pending style changes'ni flush qiladi
  // 2. Layout qayta hisoblaydi (reflow)
  // 3. Aniq rect qaytaradi
  console.log(rect.top);
});
// 1000 item uchun → 1000 reflow → main thread qotib qoladi

// ✅ Avval yozish, keyin o'qish — bitta reflow:
items.forEach(item => {
  item.style.transform = "translateX(10px)";
});
items.forEach(item => {
  const rect = item.getBoundingClientRect(); // bitta reflow hammasi uchun
  console.log(rect.top);
});

// ⚠️ Qaysi API'lar forced synchronous layout trigger qiladi:
// - offsetTop/Left/Width/Height
// - clientTop/Left/Width/Height
// - scrollTop/Left/Width/Height
// - getBoundingClientRect(), getClientRects()
// - getComputedStyle()
// - innerText (read!)
// - window.innerWidth/innerHeight (ba'zi holatlarda)
// - focus() (some cases)
```

**Nima uchun:** Brauzer rendering optimizatsiyasi uchun DOM mutations'ni batch qilib, bitta reflow bilan apply qiladi (style/layout invalidation tracking). Lekin `getBoundingClientRect` **aniq hozirgi** pozitsiyani so'raydi — pending mutations bilan hisoblash notekis bo'lardi, shuning uchun brauzer hamma narsani flush qilib layout'ni sync hisoblaydi. Shu mexanizm barcha "layout-reading" DOM API'larga taalluqli.

**Yechim:** **Read/write separation pattern** — avval barcha o'qishlarni bajarish, keyin barcha yozishlarni. Sync layout API'larni loop ichida ishlatmang — loop oldidan kerakli ma'lumotlarni olib qo'ying. Agar real-time observer kerak bo'lsa — `IntersectionObserver` yoki `ResizeObserver` ishlating (ular layout-aware lekin forced sync emas).

### Gotcha 5: `classList.toggle(cls, force)` — force argument toggle qilmaydi, u yangi holatni **majbur qiladi**

`classList.toggle("active", true)` — ko'pchilik "agar true bo'lsa toggle qil" deb tushunadi. Aslida bu "**doim true qil**" — ya'ni `classList.add("active")` bilan ekvivalent. `force: false` → doim olib tashlaydi. Argument nomi `force` "toggle conditionally" emas, "force this state".

```javascript
const el = document.querySelector(".item");
el.classList.add("active");

// ❌ "Agar true bo'lsa toggle qil" deb tasavvur qilish:
const someCondition = true;
el.classList.toggle("active", someCondition);
// Kutilgan: toggle (active → inactive)
// Haqiqiy: active qo'shadi (allaqachon bor edi — o'zgarishsiz)

// ✅ Argument'ning haqiqiy semantikasi:
el.classList.toggle("active", true);   // === add("active")
el.classList.toggle("active", false);  // === remove("active")
el.classList.toggle("active");         // true/false ga qarab almashtiradi

// Return value:
const newState = el.classList.toggle("active", true);
console.log(newState); // true — yangi holat (qo'shildi yoki mavjud)

// ─── Amaliy use: conditional class ───
// ❌ Noto'g'ri pattern:
if (isValid) {
  el.classList.add("error");
} else {
  el.classList.remove("error");
}

// ✅ Qisqa — force bilan:
el.classList.toggle("error", !isValid); // xato bo'lsa qo'shadi, yo'q bo'lsa o'chiradi

// ✅ Haqiqiy "toggle if" pattern:
if (someCondition) {
  el.classList.toggle("active"); // faqat condition true bo'lsa toggle
}

// ─── return value qaytishi gotcha'si ───
// force berilganda — return value force ga teng (add ham, bo'lgan ham):
el.classList.remove("old");
console.log(el.classList.toggle("old", false)); // false (bo'lmagan ham false)
console.log(el.classList.toggle("old", true));  // true (qo'shildi)
console.log(el.classList.toggle("old", true));  // true (bor edi — baribir true)
```

**Nima uchun:** DOM spec'da `toggle(token, force)` method'i "conditional add/remove" sifatida aniqlangan. `force` undefined bo'lsa — toggle; `force === true` bo'lsa — majburan add (class mavjud bo'lsa o'z holicha qoldiradi); `force === false` bo'lsa — majburan remove. Bu design `toggle("cls", condition)` iborasini `if (cond) add else remove` uchun qisqartma sifatida ishlatish uchun — "conditional toggle" emas.

**Yechim:** `classList.toggle(cls, bool)` ni **conditional add/remove shortcut** sifatida ishlating (`toggle("error", !isValid)` — bu idiomatik). Haqiqiy "toggle agar condition bo'lsa" uchun `if (cond) el.classList.toggle(cls);` yozish kerak. Return value'ni tekshirish kerak bo'lsa — `force`siz versiyani ishlating (u haqiqiy toggle qiladi va yangi state qaytaradi).

---

## Common Mistakes

### 1. innerHTML bilan XSS

```javascript
// ❌ User input ni innerHTML ga qo'yish — XSS xavfi:
const userInput = '<img onerror="stealCookies()" src="x">';
element.innerHTML = userInput; // ⚠️ Script bajariladi!

// ✅ textContent — xavfsiz:
element.textContent = userInput; // faqat matn sifatida ko'rsatadi
```

### 2. Layout Thrashing

```javascript
// ❌ O'qish va yozish aralash — har safar reflow:
elements.forEach(el => {
  el.style.width = el.offsetWidth * 2 + "px"; // o'qish → yozish → reflow
});

// ✅ Avval hamma o'qish, keyin hamma yozish:
const widths = [...elements].map(el => el.offsetWidth);
elements.forEach((el, i) => {
  el.style.width = widths[i] * 2 + "px";
});
```

### 3. Live Collection Loop Muammosi

```javascript
// ❌ Live collection loop da xavfli:
const items = document.getElementsByClassName("remove-me");
for (let i = 0; i < items.length; i++) {
  items[i].remove(); // ⚠️ Collection o'zgaradi, elementlar o'tkazib yuboriladi!
}

// ✅ Static collection yoki reverse loop:
const items2 = document.querySelectorAll(".remove-me"); // static
items2.forEach(item => item.remove()); // xavfsiz

// ✅ Yoki reverse:
const live = document.getElementsByClassName("remove-me");
for (let i = live.length - 1; i >= 0; i--) {
  live[i].remove();
}
```

### 4. Event Listener Memory Leak

```javascript
// ❌ Element o'chirildi, lekin tashqi reference va listener qoldi:
let card = document.getElementById("card");
const handler = () => console.log("click");
card.addEventListener("click", handler);
card.remove();
// card o'zgaruvchisi hali reference ushlab turadi → GC tozalamaydi

// ✅ Listener ni olib tashlang va reference ni tozalang:
card.removeEventListener("click", handler);
card.remove();
card = null; // reference tozalandi → GC tozalaydi
```

### 5. querySelectorAll Natijasi Array Emas

```javascript
// ❌ NodeList da array metodlari yo'q:
const items = document.querySelectorAll(".item");
items.map(item => item.textContent); // ❌ TypeError!

// ✅ Array ga aylantiring:
const texts = [...items].map(item => item.textContent); // ✅
const texts2 = Array.from(items, item => item.textContent); // ✅
```

---

## Amaliy Mashqlar

### Mashq 1: DOM Tree Traversal (Oson)

**Savol:** Berilgan element'ning barcha ota (ancestor) element'larini array sifatida qaytaring:

```javascript
// getAncestors(element) → [parent, grandparent, ..., html, document]
```

<details>
<summary>Javob</summary>

```javascript
function getAncestors(element) {
  const ancestors = [];
  let current = element.parentNode;
  while (current) {
    ancestors.push(current);
    current = current.parentNode;
  }
  return ancestors;
}

// Test:
const li = document.querySelector("li");
console.log(getAncestors(li));
// [ul, div, body, html, document]
```

</details>

---

### Mashq 2: Safe innerHTML (O'rta)

**Savol:** `safeHTML(element, html)` funksiyasini yozing — HTML qo'shadi, lekin `<script>` va `on*` event handler'larni olib tashlaydi:

<details>
<summary>Javob</summary>

```javascript
function safeHTML(element, html) {
  const template = document.createElement("template");
  template.innerHTML = html;

  // Script taglarni o'chirish
  template.content.querySelectorAll("script").forEach(s => s.remove());

  // on* attribute'larni o'chirish (onclick, onerror, ...)
  template.content.querySelectorAll("*").forEach(el => {
    [...el.attributes].forEach(attr => {
      if (attr.name.startsWith("on")) {
        el.removeAttribute(attr.name);
      }
    });
  });

  element.innerHTML = "";
  element.appendChild(template.content);
}

// Test:
safeHTML(document.getElementById("output"),
  '<p>Salom</p><script>alert("xss")</script><img onerror="alert(1)" src="x">'
);
// Natija: <p>Salom</p><img src="x"> — script va onerror o'chirildi
```

</details>

---

### Mashq 3: Batch DOM Builder (O'rta)

**Savol:** `buildList(items)` funksiyasini yozing — string array'dan `<ul>` yaratadi, DocumentFragment ishlatib performance optimal bo'lsin:

<details>
<summary>Javob</summary>

```javascript
function buildList(items) {
  const ul = document.createElement("ul");
  const fragment = document.createDocumentFragment();

  items.forEach(text => {
    const li = document.createElement("li");
    li.textContent = text;
    fragment.appendChild(li); // fragment da — reflow yo'q
  });

  ul.appendChild(fragment); // bitta operatsiya
  return ul;
}

// Test:
const list = buildList(["Olma", "Nok", "Uzum", "Gilos"]);
document.body.appendChild(list);
```

</details>

---

### Mashq 4: DOM Diff (Qiyin)

**Savol:** `updateList(container, oldItems, newItems)` funksiyasini yozing — minimal DOM operatsiyalar bilan ro'yxatni yangilaydi:

<details>
<summary>Javob</summary>

```javascript
function updateList(container, oldItems, newItems) {
  const children = container.children;

  // O'chirish kerak — ortiqcha elementlar
  while (children.length > newItems.length) {
    container.lastElementChild.remove();
  }

  // Mavjud elementlarni yangilash
  for (let i = 0; i < children.length; i++) {
    if (children[i].textContent !== newItems[i]) {
      children[i].textContent = newItems[i]; // faqat o'zgargan
    }
  }

  // Yangi elementlar qo'shish
  const fragment = document.createDocumentFragment();
  for (let i = children.length; i < newItems.length; i++) {
    const li = document.createElement("li");
    li.textContent = newItems[i];
    fragment.appendChild(li);
  }
  if (fragment.childNodes.length > 0) {
    container.appendChild(fragment);
  }
}

// Test:
const ul = document.getElementById("list");
updateList(ul, ["A", "B", "C"], ["A", "X", "C", "D"]);
// B → X o'zgardi (textContent update), D qo'shildi (append)
// Minimal DOM operatsiyalar — faqat 2 ta o'zgarish
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **DOM nima** | HTML → Parser → DOM Tree — tirik, o'zgaruvchan daraxt |
| **Node types** | Element (1), Text (3), Comment (8), Document (9), Fragment (11) |
| **Traversal** | `children` (elementlar), `childNodes` (barchasi), `closest()`, `matches()` |
| **Selecting** | `querySelector/All` — universal; `getElementById` — eng tez (O(1)) |
| **Yaratish** | `createElement`, `createTextNode`, `createDocumentFragment` |
| **Qo'shish** | `append/prepend` (zamonaviy), `insertAdjacentHTML` (4 pozitsiya) |
| **O'chirish** | `remove()` (zamonaviy), `replaceChildren()`, `replaceWith()` |
| **Cloning** | `cloneNode(true)` — deep, `cloneNode(false)` — shallow |
| **Content** | `textContent` (xavfsiz, tez), `innerHTML` (XSS xavfi), `innerText` (sekin) |
| **Attributes** | `getAttribute/setAttribute`, `dataset` (data-* uchun) |
| **ClassList** | `add`, `remove`, `toggle`, `contains`, `replace` |
| **Styles** | `style.*` (inline), `getComputedStyle()` (hisoblangan), CSS variables |
| **Reflow/Repaint** | Layout o'zgarishi = Reflow (qimmat), Ko'rinish = Repaint (arzonroq) |
| **DocumentFragment** | Virtual container — ko'p elementni bitta reflow bilan qo'shish |
| **rAF** | `requestAnimationFrame` — render cycle bilan sinxron animatsiya |
| **Virtual DOM** | DOM ning JS object nusxasi — diff/patch bilan minimal DOM update |

---

> **Keyingi bo'lim:** [19-events.md](19-events.md) — Events — Event model (registration, propagation, handling), Event Bubbling va Capturing, Event Flow (capturing → target → bubbling), `addEventListener` 3-argument (`once`, `passive`, `signal` / AbortController), `stopPropagation` va `stopImmediatePropagation`, `preventDefault`, Event Delegation pattern, `event.target` vs `event.currentTarget`, Custom Events va `dispatchEvent`, memory leak va cleanup, passive scroll performance, Touch va Pointer events.

**Cross-references:** [17-type-coercion.md](17-type-coercion.md) (typeof, ToPrimitive), [16-memory.md](16-memory.md) (GC, memory leak — detached DOM, MutationObserver batching), [11-event-loop.md](11-event-loop.md) (requestAnimationFrame, microtask/macrotask)
