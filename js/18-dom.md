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
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## DOM Nima

### Nazariya

DOM (Document Object Model) — brauzer HTML hujjatni parse qilib xotirada yaratadigan **daraxt (tree) strukturasi**. DOM **HTML ning o'zi emas** — HTML matn fayl, DOM esa brauzer xotirasidagi tirik (live) obyektlar daraxti. Brauzer HTML ni parse qilib DOM yaratadi, keyin JavaScript va CSS shu DOM bilan ishlaydi.

DOM — W3C standarti bo'lib, u **platform va til agnostic** interfeys beradi. JavaScript DOM bilan ishlash uchun eng ko'p ishlatiladigan til, lekin DOM API texnik jihatdan boshqa tillar bilan ham ishlatilishi mumkin. Brauzer kontekstida `document` global obyekt orqali DOM daraxtiga kirish mumkin.

Rendering pipeline: `HTML → Parser → DOM Tree + CSSOM → Render Tree → Layout → Paint → Composite → Screen`. Parse jarayoni **incremental** — brauzer hujjatni to'liq kutmaydi, kelgan qismlari bilan DOM qurishni boshlaydi. `<script>` tag'ga duch kelganda parser to'xtaydi (parser-blocking), shuning uchun script'larni `defer` yoki `async` bilan yoki `</body>` oldiga qo'yish tavsiya etiladi.

### Under the Hood

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

V8 (Chrome) da DOM node'lari C++ obyektlari sifatida yaratiladi va JavaScript wrapper orqali expose qilinadi. Har bir DOM node'iga JavaScript'dan kirish **bridge** orqali bo'ladi — bu narsa DOM operatsiyalarining JavaScript ichki operatsiyalardan sekinroq bo'lishining asosiy sababi.

### Kod Misollari

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

---

## Node Types

### Nazariya

DOM daraxtidagi har bir element **node**. Node'larning bir nechta turi bor, har biri `nodeType` raqami bilan farqlanadi. Amalda eng ko'p ishlatiladigan 5 ta tur: **Element** (1) — HTML tag'lar (`<div>`, `<p>`, `<a>`); **Text** (3) — element ichidagi matn; **Comment** (8) — HTML kommentlari (`<!-- ... -->`); **Document** (9) — daraxtning ildizi; **DocumentFragment** (11) — virtual container, DOM da ko'rinmaydi, lekin ichiga node'lar qo'shiladi.

Barcha node'lar `Node` interfeysidan meros oladi — `nodeType`, `nodeName`, `nodeValue`, `parentNode`, `childNodes`, `firstChild`, `lastChild`, `nextSibling`, `previousSibling` property'lari bor. Element node'lari qo'shimcha `Element` interfeysidan ham meros oladi — `children`, `className`, `id`, `getAttribute()`, `querySelector()` kabi.

### Kod Misollari

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

### Kod Misollari

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

### Kod Misollari

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

### Under the Hood

`getElementById` eng tez — brauzer DOM tree'dan alohida **ID hash map** yuritadi. Yangi ID li element qo'shilsa yoki o'chirilsa, map yangilanadi. `querySelector` esa CSS selector'ni avval parse qiladi, keyin DOM daraxtini depth-first preorder traversal bilan qidiradi — O(n). Lekin zamonaviy brauzerlar murakkab optimizatsiyalar qo'llaydi: Bloom filter, selector caching, indexed collections.

---

## Element Yaratish va Qo'shish

### Nazariya

DOM ga yangi element qo'shish — web dasturlarning asosiy vazifalaridan biri. `createElement()` bilan element yaratiladi, keyin turli usullar bilan daraxtga qo'shiladi. Zamonaviy API (`append`, `prepend`, `before`, `after`, `replaceWith`) va eski API (`appendChild`, `insertBefore`) farqi — zamonaviy usullar string ham qabul qiladi (avtomatik Text node yaratadi) va birda nechta argument oladi.

`insertAdjacentHTML/Element/Text` — 4 ta aniq pozitsiyani belgilash imkonini beradi: `beforebegin` (elementdan oldin), `afterbegin` (element ichiga boshiga), `beforeend` (element ichiga oxiriga), `afterend` (elementdan keyin). `insertAdjacentHTML` innerHTML'dan farqli ravishda mavjud elementlarni qayta parse qilmaydi — faqat yangi HTML qo'shadi.

Har safar DOM ga element qo'shilganda brauzer rendering pipeline'dan o'tadi, shuning uchun ko'p elementni birma-bir qo'shish sekin — `DocumentFragment` yoki batch usullar ishlatish kerak.

### Kod Misollari

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

---

## Element O'chirish va Almashtirish

### Nazariya

Element o'chirishning zamonaviy usuli — `element.remove()` — elementni DOM daraxtidan olib tashlaydi. Eski usul — `parent.removeChild(child)` — ota element orqali o'chiradi va o'chirilgan node'ni qaytaradi. `replaceWith()` — elementni boshqa element yoki string bilan almashtiradi. `replaceChildren()` — parent'ning barcha child'larini yangi node'lar bilan almashtiradi (tozalab, keyin qo'shadi).

Muhim nuans: DOM dan o'chirilgan element JavaScript xotirasida qoladi (agar unga reference bo'lsa). Bu memory leak sababchisi bo'lishi mumkin — event listener'lar, closure'lar orqali eski DOM elementlariga reference saqlanib qolsa, GC ularni tozalay olmaydi.

### Kod Misollari

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

---

## Cloning — cloneNode

### Nazariya

`cloneNode()` — mavjud DOM node'ning nusxasini yaratadi. Ikki rejimi bor: **shallow** (`cloneNode(false)` yoki `cloneNode()`) — faqat element o'zi klonlanadi, child'lari yo'q; **deep** (`cloneNode(true)`) — element va barcha ichki child'lari rekursiv klonlanadi.

Muhim nuans: `cloneNode` **event listener'larni klonlamaydi** (agar `addEventListener` bilan qo'shilgan bo'lsa). Inline event handler'lar (`onclick="..."`) HTML attribute sifatida klonlanadi. `id` attribute ham klonlanadi — shuning uchun klondan keyin ID ni o'zgartirish kerak (DOM da duplicate ID bo'lmasligi uchun).

### Kod Misollari

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

---

## Content O'zgartirish

### Nazariya

Element ichidagi contentni o'zgartirishning 3 ta asosiy usuli bor va ularning farqi muhim:

**`textContent`** — element va barcha child'larning **matn content'ini** o'qiydi/yozadi. Faqat matn — HTML parse qilinmaydi. Yozishda mavjud barcha child'lar o'chiriladi va bitta Text node bilan almashtiriladi. **XSS xavfi yo'q** — user input'ni xavfsiz ko'rsatish uchun eng yaxshi variant. Performance jihatdan eng tez — reflow trigger qilmaydi (faqat repaint).

**`innerHTML`** — element ichidagi **HTML markup'ni** string sifatida o'qiydi/yozadi. Yozishda string HTML sifatida parse qilinadi va DOM node'larga aylanadi. **XSS xavfi bor** — user input'ni innerHTML ga qo'yish xavfli (`<script>`, `onerror`, `onload` kabi). Yozishda mavjud child'larning barcha event listener'lari yo'qoladi (node'lar o'chiriladi va yangi yaratiladi).

**`innerText`** — element'ning **ko'rinadigan matn'ini** qaytaradi. `textContent` dan farqi: `display: none` bilan yashirilgan element matnini bermaydi, CSS layout'ni hisobga oladi, `<br>` ni yangi qatorga aylantiradi. Lekin bu uchun brauzer **reflow** qilishi kerak — shuning uchun `innerText` eng sekin.

### Kod Misollari

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

---

## Attributes va Dataset

### Nazariya

HTML element'larning ikki turdagi attribute'lari bor: **standart** (id, class, src, href, type, ...) va **custom** (`data-*`). Standart attribute'lar ko'pincha DOM property sifatida ham mavjud (`element.id`, `element.className`), lekin farqlar bor — attribute HTML dagi boshlang'ich qiymat, property esa JavaScript dagi joriy qiymat.

`getAttribute()`/`setAttribute()` — har qanday attribute bilan ishlaydi, doim **string** qaytaradi. `element.property` — faqat standart attribute'lar uchun, to'g'ri type qaytaradi (masalan, `input.checked` boolean).

`dataset` — `data-*` custom attribute'lar uchun maxsus API. `data-user-id="42"` → `element.dataset.userId` (camelCase ga o'tadi). Dataset orqali HTML dan JavaScript'ga ma'lumot uzatish — toza va standart usul.

### Kod Misollari

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

### Kod Misollari

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

---

## Style Manipulation

### Nazariya

CSS style'larni JavaScript orqali boshqarishning ikki usuli bor: **inline style** (`element.style`) va **computed style** (`getComputedStyle()`).

`element.style` — faqat **inline style** o'qiydi/yozadi. CSS property nomlari camelCase da yoziladi (`background-color` → `backgroundColor`). Bu usul faqat element'ga bevosita qo'yilgan inline style'ni ko'rsatadi — CSS fayldan kelgan style'larni ko'rsatmaydi.

`getComputedStyle(element)` — element'ga **haqiqatda qo'llangan** barcha style'larni qaytaradi (CSS fayl, inline, inherited — hammasi hisobga olinadi). Faqat o'qish mumkin — yozib bo'lmaydi. `getComputedStyle` chaqirilganda brauzer **reflow** qilishi mumkin — performance uchun kamroq ishlatish kerak.

CSS Custom Properties (`--var`) ni JavaScript orqali o'qish va yozish mumkin — bu theme switching, dynamic animation va boshqa holatlarda foydali.

### Kod Misollari

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

---

## Performance: Reflow va Repaint

### Nazariya

DOM manipulation'ning eng qimmat jihati — brauzerning rendering pipeline'ini trigger qilish. Ikki asosiy jarayon bor:

**Reflow (Layout)** — brauzer element'larning pozitsiyasi va o'lchamini qayta hisoblaydi. Bu **eng qimmat** operatsiya — bir elementning o'zgarishi butun sahifaga ta'sir qilishi mumkin (masalan, `div` kengaysa, pastdagi barcha elementlar pastga suriladi). Reflow trigger qiluvchi operatsiyalar: `width/height`, `padding/margin/border`, `display`, `position`, `font-size`, `offsetWidth/Height` (o'qish ham!), `getComputedStyle()`, `scrollTop`, `getBoundingClientRect()`.

**Repaint** — element'ning **ko'rinishi** o'zgaradi, lekin layout o'zgarmaydi. Reflow'dan arzon. Repaint trigger qiluvchi: `color`, `background`, `visibility`, `box-shadow`, `border-color`, `outline`.

**Layout thrashing** — JavaScript da DOM o'qish va yozish operatsiyalari aralash bajarilganda, har bir o'qishda brauzer avvalgi yozishlarni flush qilishi kerak. Bu ko'p reflow'ga olib keladi.

### Under the Hood

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

### Kod Misollari

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
// ❌ Har bir style alohida reflow:
element.style.width = "100px";
element.style.height = "100px";
element.style.margin = "10px";

// ✅ Bitta class bilan barcha style birdan:
element.classList.add("expanded"); // CSS da .expanded { width: 100px; height: 100px; margin: 10px; }

// ✅ cssText bilan:
element.style.cssText = "width: 100px; height: 100px; margin: 10px;";
```

---

## DocumentFragment

### Nazariya

`DocumentFragment` — DOM da ko'rinmaydigan **virtual container**. Unga element'lar qo'shilganda hech qanday reflow/repaint bo'lmaydi, chunki u hech qayerda ko'rinmaydi. Faqat fragment DOM ga append qilinganda bitta reflow sodir bo'ladi. Ko'p element qo'shishda performance uchun muhim.

Fragment DOM ga append qilinganda, uning **child'lari** ko'chiriladi, fragment o'zi qo'shilmaydi va bo'shalib qoladi. Bu `<div>` wrapper kerak emas degani.

### Kod Misollari

```javascript
// ❌ Har bir element alohida — 1000 ta reflow:
const list = document.getElementById("list");
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list.appendChild(li); // har safar reflow!
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

---

## requestAnimationFrame bilan DOM Update

### Nazariya

`requestAnimationFrame(callback)` — keyingi repaint oldidan callback ni bajaradi. Bu DOM animatsiyalari va vizual update'lar uchun eng to'g'ri usul — brauzer render cycle'iga sinxronlashtirilgan. `setTimeout`/`setInterval` bilan animatsiya qilish noto'g'ri — ular render cycle bilan sinxron emas, frame skip yoki jank (sekin, notekis animatsiya) bo'lishi mumkin.

`requestAnimationFrame` odatda 60fps (har 16.67ms) da ishlaydi, lekin brauzer tab background'da bo'lsa avtomatik to'xtaydi — batareya tejaydi. Callback'ga `DOMHighResTimeStamp` (performance.now() kabi) argument beriladi.

### Kod Misollari

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

---

## Virtual DOM Tushunchasi

### Nazariya

**Virtual DOM** — real DOM ning JavaScript object representatsiyasi. React, Vue kabi framework'lar real DOM bilan to'g'ridan-to'g'ri ishlamaydi — avval Virtual DOM da o'zgarishlarni qiladi, keyin eski va yangi Virtual DOM ni **diff** qiladi (farqlarni topadi), va faqat **o'zgargan qismlarni** real DOM da yangilaydi (patch). Bu minimal DOM manipulation kafolatlaydi.

Virtual DOM nima uchun kerak: real DOM operatsiyalari qimmat (C++ bridge, reflow/repaint). JavaScript object'lar bilan ishlash esa juda tez. 1000 ta element ichidan 3 tasi o'zgarganda — butun ro'yxatni qayta render qilish o'rniga faqat 3 ta DOM node yangilanadi.

Bu framework-specific texnologiya — native JavaScript da Virtual DOM ishlatilmaydi. Kichik dasturlarda direct DOM manipulation yetarli va ba'zan tezroq ham.

### Kod Misollari

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

**Keyingi bo'lim:** [19-events.md](19-events.md) — Events — event model, bubbling va capturing, event delegation, custom events, event listeners va memory, passive va once, AbortController pattern.

**Cross-references:** [17-type-coercion.md](17-type-coercion.md) (typeof, ToPrimitive), [16-memory.md](16-memory.md) (GC, memory leak — detached DOM), [11-event-loop.md](11-event-loop.md) (requestAnimationFrame, microtask/macrotask)
