# Bo'lim 4: Scope Chain

> Scope — o'zgaruvchilarning "ko'rinish sohasi". Engine o'zgaruvchini qayerdan qidirishini aynan scope belgilaydi.

---

## Mundarija

- [Scope Nima?](#scope-nima)
- [Global Scope](#global-scope)
- [Function Scope](#function-scope)
- [Block Scope](#block-scope)
- [Lexical Scope (Static Scope)](#lexical-scope-static-scope)
- [Dynamic Scope vs Lexical Scope](#dynamic-scope-vs-lexical-scope)
- [Scope Chain](#scope-chain)
- [Scope Chain va Execution Context](#scope-chain-va-execution-context)
- [Variable Shadowing](#variable-shadowing)
- [Variable Lookup](#variable-lookup)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Scope Nima?

### Nazariya

Scope — bu o'zgaruvchilar, funksiyalar va ob'ektlarning **ko'rinish (accessibility) sohasi**. Ya'ni kodning qaysi qismida qaysi o'zgaruvchiga kirish mumkinligi.

Oddiy qilib: scope = **"bu o'zgaruvchini qayerdan ko'rish mumkin?"**

```javascript
function greet() {
  let message = "Salom";  // message faqat greet() ICHIDA ko'rinadi
  console.log(message);    // ✅ "Salom"
}

greet();
console.log(message);      // ❌ ReferenceError — message bu yerda ko'rinMAYDI
```

### Scope Turlari

JavaScript da **3 xil** scope bor:

```
┌──────────────────────────────────────────────────┐
│  GLOBAL SCOPE                                     │
│  (hamma joydan ko'rinadi)                        │
│                                                   │
│   ┌──────────────────────────────────────────┐   │
│   │  FUNCTION SCOPE                           │   │
│   │  (faqat funksiya ichida)                  │   │
│   │                                           │   │
│   │   ┌──────────────────────────────────┐   │   │
│   │   │  BLOCK SCOPE                      │   │   │
│   │   │  (faqat {} ichida — let/const)    │   │   │
│   │   └──────────────────────────────────┘   │   │
│   └──────────────────────────────────────────┘   │
│                                                   │
└──────────────────────────────────────────────────┘
```

---

## Global Scope

### Nazariya

Global scope — eng tashqi scope. Bu yerda e'lon qilingan o'zgaruvchilar **hamma joydan** ko'rinadi — istalgan funksiya, istalgan block ichidan.

```javascript
// Global scope
var globalVar = "men global var man";
let globalLet = "men global let man";
const GLOBAL_CONST = "men global const man";

function anywhere() {
  // Funksiya ichidan ko'rinadi
  console.log(globalVar);    // ✅ "men global var man"
  console.log(globalLet);    // ✅ "men global let man"
  console.log(GLOBAL_CONST); // ✅ "men global const man"

  if (true) {
    // Block ichidan ham ko'rinadi
    console.log(globalVar);  // ✅
  }
}

anywhere();
```

### Global Scope Muammolari

Global scope kuchli, lekin **xavfli**:

```javascript
// ❌ Muammo 1: Name collision
var count = 0;

// ... 500 satr keyinroq yoki boshqa faylda ...
var count = "hello"; // Birinchi count yo'q bo'ldi!

// ❌ Muammo 2: Ifloslantirish (Global Namespace Pollution)
var temp = 1;
var data = [];
var result = null;
// Bular hammasi window ga tushadi — boshqa kutubxonalar bilan to'qnashishi mumkin

// ❌ Muammo 3: var → window property
var myApp = "config";
// Uchinchi tomon kutubxonasi ham window.myApp ishlatsa — MUAMMO
```

### Yechim — Scope ni cheklash

```javascript
// ✅ IIFE bilan scope yaratish (eski usul)
(function() {
  var privateVar = "xavfsiz";
  // bu yerda global scope ifloslanmaydi
})();

// ✅ Module bilan (zamonaviy usul)
// module.js
export const config = { port: 3000 };
// Bu global scope ga tushMAYDI

// ✅ Block scope bilan
{
  let temp = "xavfsiz";
  const data = [];
  // block tugaganda ular yo'qoladi
}
```

---

## Function Scope

### Nazariya

Function scope — funksiya ichida e'lon qilingan o'zgaruvchilar **faqat shu funksiya ichidan** ko'rinadi. Tashqaridan ko'rinmaydi.

`var`, `let`, `const` — **uchalasi** ham function scope'ni tushunadi. Lekin `var` **faqat** function scope ni tanimaydi (block scope'ni emas).

```javascript
function calculate() {
  var x = 10;       // function scope
  let y = 20;       // function scope (va block scope ham)
  const z = 30;     // function scope (va block scope ham)

  console.log(x, y, z); // ✅ 10, 20, 30
}

calculate();
console.log(x); // ❌ ReferenceError — x function ichida qoldi
console.log(y); // ❌ ReferenceError
console.log(z); // ❌ ReferenceError
```

### Har Bir Funksiya Chaqiruvi — Yangi Scope

```javascript
function counter() {
  var count = 0;  // HAR CHAQIRUVDA yangi scope, yangi count
  count++;
  return count;
}

counter(); // 1 — yangi scope: count = 0 → 1
counter(); // 1 — yana yangi scope: count = 0 → 1 (oldingi yo'q bo'lgan)
counter(); // 1 — har safar 1
```

### Nested Functions — Ichki Funksiyalar

Ichki funksiya tashqi funksiyaning o'zgaruvchilarini **ko'ra oladi**:

```javascript
function outer() {
  var outerVar = "tashqi";

  function inner() {
    var innerVar = "ichki";
    console.log(outerVar); // ✅ "tashqi" — ichki tashqini ko'radi
    console.log(innerVar); // ✅ "ichki"
  }

  inner();
  console.log(outerVar);   // ✅ "tashqi"
  console.log(innerVar);   // ❌ ReferenceError — tashqi ichkini ko'rmaydi!
}
```

```
outer() scope:
  outerVar: "tashqi"
  inner: function
  │
  └── inner() scope:
        innerVar: "ichki"
        outerVar ga kirish: ✅ (scope chain orqali)
```

**Qoida:** Ichki scope tashqini **ko'radi**. Tashqi scope ichkini **ko'rmaydi**. Faqat ichdan tashqariga — bir tomonlama.

---

## Block Scope

### Nazariya

Block scope — `{}` (curly braces) ichida yaratilgan scope. `let` va `const` block scope'ni tushunadi, `var` **tushunmaydi**.

Block: `if`, `for`, `while`, `switch`, yoki oddiy `{}`

```javascript
// if block
if (true) {
  var a = 1;     // ❌ var block dan CHIQADI (function scope)
  let b = 2;     // ✅ block ichida qoladi
  const c = 3;   // ✅ block ichida qoladi
}

console.log(a); // 1 — var chiqdi
console.log(b); // ❌ ReferenceError — let qoldi
console.log(c); // ❌ ReferenceError — const qoldi
```

### for Loop da

```javascript
// var bilan — BITTA instance
for (var i = 0; i < 3; i++) {
  // i butun function/global scope da
}
console.log(i); // 3 — var block dan chiqdi

// let bilan — HAR ITERATSIYADA yangi instance
for (let j = 0; j < 3; j++) {
  // j faqat bu iteration'ning block scope'ida
}
console.log(j); // ❌ ReferenceError — let block da qoldi
```

### Oddiy Block

```javascript
// Oddiy {} ham scope yaratadi (let/const uchun)
{
  let secret = "maxfiy";
  const API_KEY = "abc123";
  var exposed = "oshkor";   // ❌ var chiqib ketadi
}

console.log(secret);   // ❌ ReferenceError
console.log(API_KEY);  // ❌ ReferenceError
console.log(exposed);  // "oshkor" — var chiqdi
```

### switch da Block Scope

```javascript
// ❌ switch da scope muammosi
switch (action) {
  case "create":
    let result = create(); // result bu case da
    break;
  case "delete":
    let result = remove(); // ❌ SyntaxError: result allaqachon e'lon qilingan!
    break;
}

// ✅ To'g'ri usul — har bir case ni {} bilan o'rash
switch (action) {
  case "create": {
    let result = create(); // bu block scope'da
    break;
  }
  case "delete": {
    let result = remove(); // bu BOSHQA block scope'da — ✅
    break;
  }
}
```

---

## Lexical Scope (Static Scope)

### Nazariya

JavaScript **Lexical Scope** (yoki **Static Scope**) ishlatadi. Bu degani:

> O'zgaruvchining scope'i **kodda yozilgan joyiga** qarab aniqlanadi — **chaqirilgan joyiga** qarab emas.

Bu juda muhim tushuncha — closure'lar aynan shu prinsipga asoslanadi.

```javascript
let x = "global";

function outer() {
  let x = "outer";

  function inner() {
    console.log(x); // "outer" — inner YOZILGAN joyda x = "outer"
  }

  return inner;
}

const fn = outer();
fn(); // "outer" — CHAQIRILGAN joyda x = "global", lekin YOZILGAN joyda x = "outer"
```

### Vizualizatsiya

```
YOZILGAN joyga qaraydi (Lexical):

┌── Global Scope ──────────────────┐
│  let x = "global"                │
│                                  │
│  ┌── outer() Scope ──────────┐  │
│  │  let x = "outer"          │  │
│  │                            │  │
│  │  ┌── inner() Scope ───┐  │  │
│  │  │  console.log(x)    │  │  │
│  │  │  x yo'q → yuqoriga │──┘  │
│  │  │  x = "outer" ✅    │     │
│  │  └────────────────────┘     │
│  └─────────────────────────────┘│
└──────────────────────────────────┘

inner() qayerda CHAQIRILSA ham — u YOZILGAN joy bo'yicha scope chain quradi.
```

### Compile-Time da Aniqlanadi

Lexical scope — **compile-time** (yoki parse-time) da aniqlanadi. Engine kodni parse qilganda, har bir funksiyaning qaysi scope ichida **yozilganini** biladi.

```javascript
function a() {
  let x = 1;
  b(); // b() bu yerda CHAQIRILDI, lekin...
}

function b() {
  console.log(x); // ❌ ReferenceError
  // b() YOZILGAN joy — global scope
  // Global scope da x yo'q!
}

// b() a() ICHIDA chaqirilgan bo'lsa ham, a() ning x ini ko'rmaydi
// Chunki b() a() ichida YOZILMAGAN — global scope da yozilgan
let x; // ← faqat bu bo'lsa ko'radi
```

---

## Dynamic Scope vs Lexical Scope

### Nazariya

Ko'pchilik tillar (JavaScript, Python, C, Java) **Lexical Scope** ishlatadi. Ba'zi tillar (Bash, eski Perl, some Lisps) **Dynamic Scope** ishlatadi.

| Xususiyat | Lexical Scope (JS) | Dynamic Scope |
|-----------|-------------------|---------------|
| **Aniqlanadi** | Kod yozilganda (parse-time) | Kod chaqirilganda (runtime) |
| **Qaraydi** | Funksiya YOZILGAN joy | Funksiya CHAQIRILGAN joy |
| **Predictable** | Ha — kodni o'qib bilib olish mumkin | Yo'q — runtime da o'zgaradi |
| **Tillar** | JS, Python, C, Java, Go | Bash, old Perl, some Lisps |

### Farqni Ko'rsatish

```javascript
let x = "global";

function first() {
  let x = "first";
  second();
}

function second() {
  console.log(x);
}

first();
```

```
LEXICAL SCOPE (JavaScript):
  second() YOZILGAN joy = global scope
  x = "global" ✅

Agar DYNAMIC SCOPE bo'lganida:
  second() CHAQIRILGAN joy = first() ichida
  x = "first" bo'lardi
```

JavaScript **doim** lexical scope — `second()` qayerda chaqirilmasin, u **yozilgan** joydagi scope chain'ni ishlatadi.

> **Eslatma:** JavaScript da `this` keyword — dynamic scope'ga o'xshash behavior ko'rsatadi (chaqirilgan joyga qaraydi). Lekin bu scope emas, bu binding. To'liq [10-this-keyword.md](10-this-keyword.md) da.

---

## Scope Chain

### Nazariya

Scope Chain — bu engine o'zgaruvchini qidirish uchun yuradigan **zanjir**. Har bir scope o'z **tashqi (outer) scope**'iga havola qiladi. Engine o'zgaruvchini avval **o'z scope**'ida qidiradi, topmasa — **tashqi scope**'ga, keyin yana tashqiga... to global scope'gacha.

```
inner scope → outer scope → ... → global scope → topilmadi? → ReferenceError
```

### Misol

```javascript
let a = "global a";
let b = "global b";
let c = "global c";

function outer() {
  let b = "outer b";
  let d = "outer d";

  function inner() {
    let c = "inner c";

    console.log(a); // ? → o'z scope'da yo'q → outer da yo'q → GLOBAL da bor → "global a"
    console.log(b); // ? → o'z scope'da yo'q → OUTER da bor → "outer b"
    console.log(c); // ? → O'Z SCOPE da bor → "inner c"
    console.log(d); // ? → o'z scope'da yo'q → OUTER da bor → "outer d"
  }

  inner();
}

outer();
```

### Scope Chain Diagramma

```
┌── Global Scope ─────────────────────────────────┐
│  a: "global a"                                   │
│  b: "global b"                                   │
│  c: "global c"                                   │
│  outer: function                                 │
│                                                  │
│  ┌── outer() Scope ─────────────────────────┐   │
│  │  b: "outer b"  (global b ni shadow)       │   │
│  │  d: "outer d"                             │   │
│  │  inner: function                          │   │
│  │                                            │   │
│  │  ┌── inner() Scope ──────────────────┐   │   │
│  │  │  c: "inner c"  (global c ni shadow)│   │   │
│  │  │                                    │   │   │
│  │  │  a → yo'q → outer yo'q → GLOBAL ──│───│───│→ "global a"
│  │  │  b → yo'q → OUTER ────────────────│───│→ "outer b"
│  │  │  c → O'ZIDA ──────────────────────│→ "inner c"
│  │  │  d → yo'q → OUTER ────────────────│───│→ "outer d"
│  │  └───────────────────────────────────┘   │   │
│  └───────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

### Scope Chain FAQAT Yuqoriga Yuradi

```javascript
function parent() {
  let parentVar = "parent";

  function child() {
    let childVar = "child";
    console.log(parentVar); // ✅ yuqoriga → topdi
  }

  child();
  console.log(childVar); // ❌ ReferenceError — pastga yura olmaydi!
}
```

**Qoida:** Scope chain **faqat ichdan tashqariga** yuradi. Tashqaridan ichkariga kirish **mumkin emas**.

---

## Scope Chain va Execution Context

### Under the Hood

Scope Chain aslida Execution Context ning bir qismi. [02-execution-context.md](02-execution-context.md) da o'rganganimizdek, har bir EC ning **LexicalEnvironment** si bor, va har bir LexicalEnvironment **outer reference** ga ega:

```
┌─────────────────────────────────┐
│  inner() EC                     │
│  LexicalEnvironment:            │
│    c: "inner c"                 │
│    outer: ─────────────────────────→ outer() EC ning LexicalEnvironment
│                                 │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  outer() EC                     │
│  LexicalEnvironment:            │
│    b: "outer b"                 │
│    d: "outer d"                 │
│    outer: ─────────────────────────→ Global EC ning LexicalEnvironment
│                                 │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Global EC                      │
│  LexicalEnvironment:            │
│    a: "global a"                │
│    b: "global b"                │
│    c: "global c"                │
│    outer: null (eng tashqi)     │
│                                 │
└─────────────────────────────────┘
```

**Scope Chain** = LexicalEnvironment'larning **outer reference** lar zanjiri.

Engine o'zgaruvchini qidirganda:
1. Joriy EC ning LexicalEnvironment da qidiradi
2. Topmasa → `outer` reference orqali tashqi Environment ga
3. Topmasa → yana tashqiga
4. Global'gacha yetdi va topmasa → **ReferenceError**

---

## Variable Shadowing

### Nazariya

Variable shadowing — ichki scope'dagi o'zgaruvchi tashqi scope'dagini **"yashirishi"**. Ichki scope'da bir xil nomli o'zgaruvchi yaratilsa, tashqi scope'dagi ko'rinmay qoladi.

```javascript
let name = "Global";

function outer() {
  let name = "Outer"; // global name ni SHADOW qildi

  function inner() {
    let name = "Inner"; // outer name ni SHADOW qildi

    console.log(name); // "Inner" — eng yaqin scope
  }

  inner();
  console.log(name); // "Outer" — inner ning name buni ko'rmaydi
}

outer();
console.log(name); // "Global" — global o'zgarmagan
```

```
┌── Global: name = "Global" ──────────────┐
│                                          │
│  ┌── outer: name = "Outer" ──────────┐  │  ← Global "Global" ni yashirdi
│  │                                    │  │
│  │  ┌── inner: name = "Inner" ───┐  │  │  ← Outer "Outer" ni yashirdi
│  │  │  console.log(name)         │  │  │
│  │  │  → "Inner" ✅              │  │  │
│  │  └────────────────────────────┘  │  │
│  │                                    │  │
│  │  console.log(name) → "Outer" ✅  │  │
│  └────────────────────────────────────┘  │
│                                          │
│  console.log(name) → "Global" ✅        │
└──────────────────────────────────────────┘
```

### var bilan Shadowing Muammolari

```javascript
var x = 10;

function test() {
  console.log(x); // undefined — hoist bo'lgan LOCAL x ni ko'radi, global 10 ni emas!
  var x = 20;     // bu x global x ni shadow qildi
  console.log(x); // 20
}

test();
console.log(x); // 10 — global o'zgarmagan
```

Bu hoisting + shadowing kombinatsiyasi — ko'p interview savollarining asosi.

### let bilan var ni Shadow Qilish

```javascript
var x = 1;
{
  let x = 2;  // ✅ var x ni shadow qildi — bu legal
  console.log(x); // 2
}
console.log(x); // 1

// Lekin teskari — illegal:
let y = 1;
{
  var y = 2; // ❌ SyntaxError — var function scope, let bilan conflict
}
```

---

## Variable Lookup

### Nazariya

Engine o'zgaruvchini topish uchun **scope chain bo'ylab qidiradi**. Bu jarayon aniq qoidalar bo'yicha ishlaydi:

### Lookup Algoritmi

```
1. Joriy scope'da bor? → Ha → ISH TAMOM, qiymatini qaytir
                         → Yo'q → 2-qadamga

2. Tashqi (outer) scope bor? → Ha → outer scope'da qidir (1-qadamga qayit)
                                → Yo'q → 3-qadamga

3. Global scope'da bor? → Ha → qiymatini qaytir
                          → Yo'q → 4-qadamga

4. ReferenceError (strict mode)
   yoki undefined (non-strict, assignment da implicit global)
```

### Misol

```javascript
let x = "global";

function a() {
  let y = "a scope";

  function b() {
    let z = "b scope";

    function c() {
      // x ni qidirish:
      // c scope → yo'q
      // b scope → yo'q
      // a scope → yo'q
      // global scope → BOR! → "global"
      console.log(x); // "global"

      // y ni qidirish:
      // c scope → yo'q
      // b scope → yo'q
      // a scope → BOR! → "a scope"
      console.log(y); // "a scope"

      // z ni qidirish:
      // c scope → yo'q... ASLIDA: z... yo'q wait, z b scope da
      // c scope → yo'q
      // b scope → BOR! → "b scope"
      console.log(z); // "b scope"
    }
    c();
  }
  b();
}
a();
```

### Non-Strict Mode — Implicit Global (Xavfli!)

```javascript
// ❌ Non-strict mode da:
function danger() {
  oops = "global bo'lib qoldi!"; // var/let/const yo'q!
}
danger();
console.log(oops);       // "global bo'lib qoldi!" — global scope ga chiqdi!
console.log(window.oops); // "global bo'lib qoldi!" — window property bo'ldi!

// ✅ Strict mode da:
"use strict";
function safe() {
  oops = "xato"; // ❌ ReferenceError: oops is not defined
}
```

**Qoida:** Doim `"use strict"` ishlatish yoki module ishlatish (module'lar avtomatik strict). E'lon qilmasdan assign qilish — **xavfli implicit global** yaratadi.

---

## Common Mistakes

### ❌ Xato 1: Lexical scope o'rniga dynamic scope kutish

```javascript
let x = "global";

function printX() {
  console.log(x);
}

function callPrintX() {
  let x = "local";
  printX(); // "global" — "local" emas!
}

callPrintX();
```

### ✅ To'g'ri tushunish:

```javascript
// printX() YOZILGAN joyda x = "global"
// printX() CHAQIRILGAN joyda x = "local"
// JavaScript LEXICAL scope → YOZILGAN joy qaraydi → "global"

// Agar "local" chiqishini xohlasangiz — argument sifatida bering:
function printX(val) {
  console.log(val);
}

function callPrintX() {
  let x = "local";
  printX(x); // "local" ✅
}
```

**Nima uchun:** JS lexical scope — funksiya **yozilgan** joydagi scope chain saqlanadi, chaqirilgan joydagi emas.

---

### ❌ Xato 2: var ning block scope'siz ishlashini unutish

```javascript
function example() {
  for (var i = 0; i < 5; i++) {
    // ...
  }
  console.log(i); // 5 — for dan tashqarida!

  if (true) {
    var secret = "maxfiy";
  }
  console.log(secret); // "maxfiy" — if dan tashqarida!
}
```

### ✅ To'g'ri usul:

```javascript
function example() {
  for (let i = 0; i < 5; i++) {
    // ...
  }
  // console.log(i); // ❌ ReferenceError — let block ichida qoldi

  if (true) {
    const secret = "maxfiy";
  }
  // console.log(secret); // ❌ ReferenceError — const block ichida qoldi
}
```

**Nima uchun:** `var` faqat function scope'ni taniydi. `if`, `for`, `while` — bular function emas, block. `let`/`const` ishlating.

---

### ❌ Xato 3: Implicit global yaratish

```javascript
function oops() {
  myVar = "xavfli!"; // var/let/const yo'q!
}
oops();
console.log(myVar);       // "xavfli!" — global bo'lib qoldi
console.log(window.myVar); // "xavfli!" — window ga qo'shildi
```

### ✅ To'g'ri usul:

```javascript
"use strict"; // yoki module mode

function safe() {
  const myVar = "xavfsiz";
}
safe();
// console.log(myVar); // ❌ ReferenceError
```

**Nima uchun:** E'lon qilinmagan o'zgaruvchiga assign qilish (non-strict) global property yaratadi. `"use strict"` yoki ES module mode bu xatoni ReferenceError bilan ushlaydi.

---

### ❌ Xato 4: Shadowing + hoisting aralashmasi

```javascript
var x = 10;

function confusing() {
  console.log(x); // Ko'pchilik 10 kutadi
  if (false) {
    var x = 20;  // Bu satr HECH QACHON bajarilmaydi
  }
  console.log(x);
}

confusing();
```

### ✅ To'g'ri tushunish:

```javascript
// Output: undefined, undefined

// Nima uchun?
// var x = 20 — if(false) ichida bo'lsa ham, VAR HOIST bo'ladi!
// var block scope'ni tanimaydi — function scope'da hoist
// confusing() ning Creation Phase da: var x = undefined (LOCAL)
// Bu local x global x ni SHADOW qildi
// if(false) bajarilmaydi — x hech qachon 20 bo'lmaydi
// Ikki console.log ham LOCAL x = undefined ni ko'radi
```

**Nima uchun:** `var` block'dan chiqadi + hoist bo'ladi. `if (false)` ichidagi `var x` ham function scope'da hoist bo'ladi. `let` ishlatganingizda bu muammo bo'lmaydi.

---

### ❌ Xato 5: Closure + scope ni tushunmaslik

```javascript
function createFunctions() {
  var result = [];
  for (var i = 0; i < 3; i++) {
    result.push(function() {
      return i;
    });
  }
  return result;
}

var fns = createFunctions();
console.log(fns[0]()); // 3 — 0 emas!
console.log(fns[1]()); // 3 — 1 emas!
console.log(fns[2]()); // 3 — 2 emas!
```

### ✅ To'g'ri usul:

```javascript
// let ishlatish — har iteratsiyada yangi scope
function createFunctions() {
  const result = [];
  for (let i = 0; i < 3; i++) {
    result.push(function() {
      return i; // har bir callback O'ZINING i siga ega
    });
  }
  return result;
}

const fns = createFunctions();
console.log(fns[0]()); // 0 ✅
console.log(fns[1]()); // 1 ✅
console.log(fns[2]()); // 2 ✅
```

**Nima uchun:** `var i` — bitta instance. Barcha callback'lar **bitta** `i` ga reference qiladi. Loop tugaganda `i = 3`. `let i` — har iteratsiyada **yangi** scope va **yangi** `i`. Har bir callback o'zining `i` nusxasiga ega. Ko'proq [05-closures.md](05-closures.md) da.

---

## Amaliy Mashqlar

### Mashq 1: Scope Aniqlash (Oson)

**Savol:** Har bir `console.log` nima chiqaradi?

```javascript
let a = 1;
var b = 2;

function test() {
  let a = 10;
  var b = 20;

  {
    let a = 100;
    var c = 200;
    console.log(a); // ?
    console.log(b); // ?
  }

  console.log(a); // ?
  console.log(b); // ?
  console.log(c); // ?
}

test();
console.log(a); // ?
console.log(b); // ?
```

<details>
<summary>Javob</summary>

```javascript
console.log(a); // 100 — block scope'dagi let a
console.log(b); // 20 — var function scope, block ta'sir qilmaydi

console.log(a); // 10 — function scope'dagi let a
console.log(b); // 20
console.log(c); // 200 — var block dan chiqdi, function scope'da

console.log(a); // 1 — global let a
console.log(b); // 2 — global var b
```

**Tushuntirish:**
- `let a = 100` block ichida — faqat block scope
- `var c = 200` block ichida — lekin var function scope, block dan chiqadi
- Har bir scope o'zining `a` va `b` sini yaratgan — shadowing
</details>

---

### Mashq 2: Scope Chain Lookup (O'rta)

**Savol:**

```javascript
let x = "A";

function one() {
  let x = "B";

  function two() {
    let x = "C";

    function three() {
      console.log(x); // ?
    }
    three();
  }
  two();
}

one();
```

`three()` ichidagi `console.log(x)` "A", "B" yoki "C" chiqaradi?

Endi `three()` ichidagi `let x = "C"` ni olib tashlang — nima bo'ladi?
`two()` ichidagi `let x = "B"` ni ham olib tashlang — nima bo'ladi?

<details>
<summary>Javob</summary>

```javascript
// Original: console.log(x) → "C"
// three() → o'zida yo'q → two() da bor → "C"

// two() dan let x olib tashlansa:
// three() → o'zida yo'q → two() yo'q → one() da bor → "B"

// one() dan ham olib tashlansa:
// three() → yo'q → two() yo'q → one() yo'q → GLOBAL → "A"
```

Scope chain: `three → two → one → global` — ichdan tashqariga, yozilgan joylari bo'yicha.
</details>

---

### Mashq 3: Lexical vs Dynamic (O'rta)

**Savol:**

```javascript
let animal = "cat";

function getAnimal() {
  return animal;
}

function showAnimal() {
  let animal = "dog";
  console.log(getAnimal()); // ?
}

showAnimal();
```

<details>
<summary>Javob</summary>

```javascript
console.log(getAnimal()); // "cat"
```

**Tushuntirish:**

`getAnimal()` **yozilgan** joy = global scope. Global scope da `animal = "cat"`.
`showAnimal()` ichida `animal = "dog"` bor, lekin `getAnimal()` bu scope'ni ko'rmaydi — chunki u bu yerda **yozilmagan**, faqat **chaqirilgan**.

JavaScript = lexical scope = yozilgan joyga qaraydi.
</details>

---

### Mashq 4: Block Scope Puzzle (Qiyin)

**Savol:**

```javascript
function loop() {
  for (var i = 0; i < 3; i++) {
    setTimeout(function() {
      console.log("var:", i);
    }, 10);
  }

  for (let j = 0; j < 3; j++) {
    setTimeout(function() {
      console.log("let:", j);
    }, 10);
  }
}

loop();
```

<details>
<summary>Javob</summary>

```
var: 3
var: 3
var: 3
let: 0
let: 1
let: 2
```

**Tushuntirish:**

**var loop:**
- `var i` = function scope da **bitta** o'zgaruvchi
- Loop tugaganda `i = 3`
- setTimeout callback'lari loop tugagandan keyin ishlaydi
- Barchasi bitta `i = 3` ni ko'radi

**let loop:**
- `let j` = har iteratsiyada **yangi block scope**
- Har bir setTimeout callback o'zining `j` nusxasiga ega
- Callback #0 → j=0, Callback #1 → j=1, Callback #2 → j=2

Bu var va let ning eng muhim amaliy farqi.
</details>

---

### Mashq 5: Scope Chain Master (Qiyin)

**Savol:** Har bir `console.log` nima chiqaradi?

```javascript
var x = 1;
let y = 2;

function a() {
  var x = 10;
  let y = 20;

  function b() {
    var x = 100;

    function c() {
      console.log(x); // ?
      console.log(y); // ?
    }
    c();
  }
  b();
}

a();
```

<details>
<summary>Javob</summary>

```javascript
console.log(x); // 100
console.log(y); // 20
```

**Scope chain vizualizatsiya:**

```
c() uchun x qidirish:
  c() scope → yo'q
  b() scope → var x = 100 → TOPILDI ✅ → 100

c() uchun y qidirish:
  c() scope → yo'q
  b() scope → yo'q (b da y e'lon qilinmagan!)
  a() scope → let y = 20 → TOPILDI ✅ → 20
```

Kalit: `b()` da `x` bor lekin `y` yo'q. Shuning uchun `x` b() dan olinadi, `y` esa b() ni "sakrab" a() dan olinadi.
</details>

---

## Xulosa

1. **Scope** — o'zgaruvchining ko'rinish sohasi. 3 turi: Global, Function, Block.

2. **Global Scope** — hamma joydan ko'rinadi. Muammo: namespace pollution. Yechim: module, IIFE, block scope.

3. **Function Scope** — `var`, `let`, `const` hammasi taniydi. Funksiya ichida e'lon = tashqaridan ko'rinmaydi.

4. **Block Scope** — `let`/`const` taniydi, `var` **tanimaydi** (block'dan chiqadi).

5. **Lexical Scope** — JS kodda **yozilgan joyiga** qarab scope aniqlanadi, chaqirilgan joyiga emas. Bu closure'lar uchun asos.

6. **Scope Chain** — ichdan tashqariga zanjir: joriy scope → outer → ... → global → ReferenceError.

7. **Variable Shadowing** — ichki scope tashqi scope'dagi bir xil nomli o'zgaruvchini "yashiradi".

8. **Variable Lookup** — engine scope chain bo'ylab birinchi topgan qiymatni qaytaradi. `"use strict"` implicit global'larni oldini oladi.

---

> **Keyingi bo'lim:** [05-closures.md](05-closures.md) — Closures — lexical scope'ning eng kuchli natijasi. Funksiya yaratilganda scope saqlanadi.
