# Bo'lim 2: Execution Context

> JavaScript kodni bajarishdan oldin "sahna tayyorlaydi" — Execution Context aynan shu sahna.

---

## Mundarija

- [Execution Context Nima?](#execution-context-nima)
- [Execution Context Turlari](#execution-context-turlari)
- [Global Execution Context (GEC)](#global-execution-context-gec)
- [Function Execution Context (FEC)](#function-execution-context-fec)
- [Execution Context Phases](#execution-context-phases)
- [Variable Environment vs Lexical Environment](#variable-environment-vs-lexical-environment)
- [Environment Record](#environment-record)
- [this Binding](#this-binding)
- [Execution Context Stack](#execution-context-stack)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Execution Context Nima?

### Nazariya

Execution Context — bu JavaScript engine kodni bajarish uchun yaratadigan **muhit (environment)** bo'lib, u dasturning istalgan nuqtasida qaysi o'zgaruvchilar mavjud, ularning qiymatlari nima, qaysi scope'larga kirish mumkin va `this` nimaga ishora qilayotganini belgilaydi.

Buni **teatr sahnasi**ga o'xshatish mumkin. Har bir sahna (Execution Context) o'zining dekoratsiyalari (o'zgaruvchilar), aktyorlari (funksiyalar) va rejissyori (engine) ga ega. Yangi sahna o'ynalsa — yangi dekoratsiya o'rnatiladi. Eski sahna tugasa — dekoratsiya yig'ishtiriladi. Global kod — bu asosiy sahna, har bir funksiya chaqiruvi esa yangi kichik sahna ochadi.

Execution Context tushunchasi nima uchun muhim? Chunki JavaScript'dagi eng ko'p uchraydigan tushunmovchiliklar — hoisting, scope, closure, `this` ning kutilmagan qiymati — bularning barchasi Execution Context mexanizmini bilmaslikdan kelib chiqadi. Agar siz EC qanday yaratilishini, uning ichida nima borligini va qachon yo'q qilinishini tushunsangiz, bu tushunchalarning barchasi o'z joyiga tushadi.

ECMAScript spetsifikatsiyasiga ko'ra har bir Execution Context uchta asosiy komponentdan iborat: **LexicalEnvironment** (`let`, `const`, funksiya e'lonlari uchun), **VariableEnvironment** (`var` e'lonlari uchun) va **ThisBinding** (`this` qiymati). Bu komponentlarning har biri alohida va muhim rol o'ynaydi — ularni keyingi bo'limlarda batafsil ko'rib chiqamiz.

### Under the Hood

ECMAScript spec bo'yicha Execution Context quyidagi komponentlardan iborat:

```
┌──────────────────────────────────────┐
│        EXECUTION CONTEXT             │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  LexicalEnvironment            │  │
│  │  (let, const, functions)       │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  VariableEnvironment           │  │
│  │  (var declarations)            │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  ThisBinding                   │  │
│  │  (this qiymati)               │  │
│  └────────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

---

## Execution Context Turlari

JavaScript da **3 xil** Execution Context bor:

| Turi | Qachon yaratiladi | Nechta bo'ladi |
|------|-------------------|----------------|
| **Global Execution Context (GEC)** | Dastur boshlanganda | Faqat **1 ta** |
| **Function Execution Context (FEC)** | Har bir funksiya chaqirilganda | **Cheksiz** (har bir call uchun yangi) |
| **Eval Execution Context** | `eval()` chaqirilganda | Kam ishlatiladi, e'tiborsiz qoldiramiz |

---

## Global Execution Context (GEC)

### Nazariya

GEC (Global Execution Context) — JavaScript dasturi ishga tushganda eng birinchi yaratiladigan Execution Context. U butun dastur davomida yashaydi va faqat **bitta** bo'ladi — dastur tugaguncha. GEC barcha global kodning "uyi" hisoblanadi.

GEC nima uchun muhim? Chunki u dasturning ishlash uchun zarur bo'lgan dastlabki muhitni tayyorlaydi. GEC yaratilganda uchta muhim ish sodir bo'ladi: birinchidan, **Global Object** yaratiladi (brauzerda `window`, Node.js da `global`, universal `globalThis`), ikkinchidan, `this` global ob'ektga bog'lanadi, uchinchidan, barcha global `var` e'lonlari va function declaration'lar **hoist** qilinadi — ya'ni xotiraga oldindan yoziladi.

Buni **ofis binosi**ga o'xshatish mumkin. GEC — bu binoning asosiy qavati (lobby). Bino qurilganda avval lobby tayyorlanadi. Lobby doimiy — bino yashayotgan ekan u bor. Barcha umumiy resurslar (receptsiya, lift, yo'laklar) shu yerda. Har bir xona (funksiya) o'z ichki muhitiga ega, lekin lobby'dagi resurslardan foydalanishi mumkin. Xuddi shunday, global scope'dagi o'zgaruvchi va funksiyalar istalgan joydan ko'rinadi.

Amaliy nuqtai nazardan, GEC bilan ishlashda ikkita muhim narsani bilish kerak: `var` bilan global scope'da e'lon qilingan o'zgaruvchilar global ob'ektning property'siga aylanadi (`window.myVar`), lekin `let` va `const` bilan e'lon qilinganlar global scope'da mavjud bo'lsa-da, global ob'ektga **qo'shilmaydi**. Bu farq ko'p xatolarga sabab bo'ladi.

```javascript
// Bu kod hali BIRORTA ham funksiya ichida emas
// Shuning uchun Global Execution Context ichida bajariladi

var name = "Islom";          // GEC ning Variable Environment da
let age = 25;                // GEC ning Lexical Environment da
const ROLE = "developer";    // GEC ning Lexical Environment da

function greet() {           // GEC da hoist qilingan
  return "Salom!";
}

console.log(this);           // window (browser) / global (Node.js)
console.log(this === window); // true (browser da)
```

### GEC Yaratilish Jarayoni

```
┌─────────────────────────────────────────────┐
│          GLOBAL EXECUTION CONTEXT           │
│                                             │
│  ── Creation Phase ──                       │
│  1. Global Object yaratildi (window)        │
│  2. this = window                           │
│  3. var name = undefined  (hoist)           │
│  4. greet = function(){...}  (hoist)        │
│  5. let age — TDZ da  (hoist, lekin init ❌)│
│  6. const ROLE — TDZ da                     │
│                                             │
│  ── Execution Phase ──                      │
│  1. name = "Islom"                          │
│  2. age = 25  (TDZ tugadi)                  │
│  3. ROLE = "developer"  (TDZ tugadi)        │
│  4. greet() chaqirilsa — yangi FEC          │
│                                             │
└─────────────────────────────────────────────┘
```

### Global Object bilan munosabat

```javascript
var x = 10;
let y = 20;

console.log(window.x); // 10 — var global object ga property bo'lib qo'shiladi
console.log(window.y); // undefined — let global object ga qo'shilMAYDI

// Shuning uchun:
console.log(x === window.x); // true
console.log(y === window.y); // false
```

Bu muhim farq — `var` global scope da `window` ning property'si bo'ladi, `let`/`const` esa yo'q. Ular global scope da bor, lekin global **object** da emas.

---

## Function Execution Context (FEC)

### Nazariya

Har safar funksiya **chaqirilganda** (e'lon qilinganda emas, aynan **chaqirilganda**) yangi Function Execution Context yaratiladi. Bu JavaScript'ning eng fundamental mexanizmlaridan biri — har bir funksiya chaqiruvi mustaqil, izolyatsiyalangan muhitda bajariladi.

Buni **restoran buyurtmasi**ga o'xshatish mumkin. Oshpaz (engine) har bir buyurtma (funksiya chaqiruvi) uchun yangi ish stolini tayyorlaydi: ingredientlarni qo'yadi (parametrlar), idishlarni joylashtiradi (local o'zgaruvchilar), retseptni ko'radi (kod). Taom tayyor bo'lganda (funksiya tugaganda) — stol tozalanadi va yangi buyurtma uchun bo'shatiladi.

Nima uchun har bir chaqiruv uchun yangi EC kerak? Chunki bir xil funksiya turli argumentlar bilan chaqirilishi mumkin. `calculate(3, 7)` va `calculate(10, 20)` turli natija beradi — har birining o'z muhiti bo'lishi kerak. Shuningdek, rekursiv funksiyalarda bir xil funksiya o'zini qayta-qayta chaqiradi — har bir chaqiruv o'z alohida EC'siga ega bo'lishi shart, aks holda o'zgaruvchilar bir-birini buzib yuboradi.

```javascript
function sayHi(name) {
  let greeting = "Salom, " + name;
  return greeting;
}

// Har bir chaqiruv = yangi FEC
sayHi("Ali");   // FEC #1 yaratildi → bajarildi → yo'q qilindi
sayHi("Vali");  // FEC #2 yaratildi → bajarildi → yo'q qilindi
```

### FEC Yaratilish Jarayoni

```javascript
function calculate(a, b) {
  var result = a + b;
  let doubled = result * 2;
  return doubled;
}

calculate(3, 7);
```

```
┌─────────────────────────────────────────────┐
│    FUNCTION EXECUTION CONTEXT: calculate()  │
│                                             │
│  ── Creation Phase ──                       │
│  1. arguments = { 0: 3, 1: 7, length: 2 }  │
│  2. a = 3, b = 7  (parametrlar)            │
│  3. var result = undefined  (hoist)         │
│  4. let doubled — TDZ da                    │
│  5. Scope Chain = [local scope, global]     │
│  6. this = window (non-strict)              │
│                                             │
│  ── Execution Phase ──                      │
│  1. result = 3 + 7 = 10                    │
│  2. doubled = 10 * 2 = 20  (TDZ tugadi)   │
│  3. return 20                               │
│  4. FEC stack dan olib tashlanadi           │
│                                             │
└─────────────────────────────────────────────┘
```

### Nested Function Calls

Har bir ichki funksiya chaqiruvi ham o'z FEC ini yaratadi:

```javascript
function outer() {
  var x = 10;

  function inner() {
    var y = 20;
    return x + y; // x ni outer ning scope idan oladi
  }

  return inner();
}

outer(); // 30
```

```
Execution Context Stack:

1)                  2)                   3)
┌──────────┐       ┌──────────┐        ┌──────────┐
│          │       │  inner() │        │          │
│  outer() │       │  outer() │        │  outer() │
│  global  │       │  global  │        │  global  │
└──────────┘       └──────────┘        └──────────┘
 outer()            inner()             inner() tugadi
 chaqirildi         chaqirildi          pop, outer davom

4)
┌──────────┐
│          │
│          │
│  global  │
└──────────┘
 outer() tugadi
 pop, return 30
```

---

## Execution Context Phases

Har bir Execution Context (GEC va FEC) **2 bosqichdan** o'tadi:

### Phase 1: Creation Phase (Yaratish Bosqichi)

Kod **hali bajarilmagan** — engine faqat "tayyorgarlik" ko'radi:

1. **LexicalEnvironment** yaratiladi
2. **VariableEnvironment** yaratiladi
3. **`this`** binding aniqlanadi
4. O'zgaruvchilar uchun xotira ajratiladi:
   - `var` → `undefined` bilan initialize
   - `let`/`const` → TDZ da (uninitialized)
   - `function declaration` → to'liq funksiya bilan initialize (to'liq hoist)

Bu bosqich **hoisting** deb ataladigan hodisaning sababi — keyingi bo'limda ([03-hoisting.md](03-hoisting.md)) batafsil.

### Phase 2: Execution Phase (Bajarish Bosqichi)

Kod **satr-bosatr** bajariladi:

1. O'zgaruvchilarga qiymat beriladi
2. Funksiyalar chaqiriladi
3. Ifodalar hisoblanadi

### Misol bilan Ko'rsatish

```javascript
console.log(a);    // ?
console.log(b);    // ?
console.log(c);    // ?
console.log(greet); // ?

var a = 1;
let b = 2;
const c = 3;
function greet() { return "hi"; }
```

**Creation Phase da:**

```
┌────────────────────────────────────────────────────┐
│  CREATION PHASE                                     │
│                                                     │
│  VariableEnvironment:                               │
│    a → undefined          (var: hoist + init)       │
│                                                     │
│  LexicalEnvironment:                                │
│    b → <uninitialized>    (let: hoist, TDZ da)     │
│    c → <uninitialized>    (const: hoist, TDZ da)   │
│    greet → function(){...} (function: to'liq hoist) │
│                                                     │
│  ThisBinding: window                                │
└────────────────────────────────────────────────────┘
```

**Execution Phase da (satr-bosatr):**

```javascript
console.log(a);     // undefined — var hoist bo'lgan, lekin qiymat hali berilmagan
console.log(b);     // ❌ ReferenceError — TDZ da!
// console.log(c);  // ❌ ReferenceError — TDZ da! (bu satrga yetmaydi)
// console.log(greet); // function greet(){...} — to'liq hoist

var a = 1;          // a = 1 (endi undefined emas)
let b = 2;          // b = 2 (TDZ tugadi)
const c = 3;        // c = 3 (TDZ tugadi)
function greet() { return "hi"; } // allaqachon hoist bo'lgan
```

---

## Variable Environment vs Lexical Environment

### Nazariya

ES6 (ECMAScript 2015) dan boshlab Execution Context ichida ikkita alohida environment mavjud — **VariableEnvironment** va **LexicalEnvironment**. Bu ikki muhitning mavjudligi `var` va `let`/`const` ning tubdan farqli ishlash mexanizmini ta'minlaydi.

Nima uchun ikkita environment kerak bo'ldi? ES6 dan oldin JavaScript'da faqat `var` bor edi va u faqat function scope'ni tan olardi. Block scope (`if`, `for` ichida) tushunchasi umuman yo'q edi. ES6 da `let` va `const` block scope bilan kelganda, engine ichida muammo paydo bo'ldi: eski `var` behavior'ini buzmasdan, yangi block scope mexanizmini qanday qo'shish kerak? Javob — ikkita alohida muhit yaratish: `var` uchun VariableEnvironment (eski funksional scope qoidalari bilan), `let`/`const` uchun LexicalEnvironment (yangi block scope qoidalari bilan).

Bu tushunchani bilish nima uchun amaliy jihatdan muhim? Chunki ko'pgina murakkab xatolar — masalan, `var` ning loop'dan chiqib ketishi, `let` ning block ichida qolishi, TDZ (Temporal Dead Zone) — bularning barchasi shu ikki environment ning turli ishlash qoidalaridan kelib chiqadi. Agar siz VariableEnvironment va LexicalEnvironment ning farqini tushunsangiz, `var` vs `let`/`const` bilan bog'liq muammolarni tezda aniqlash va hal qilishingiz mumkin.

| Xususiyat | VariableEnvironment | LexicalEnvironment |
|-----------|--------------------|--------------------|
| **Nima saqlaydi** | `var` declarations | `let`, `const`, `function` declarations |
| **Scope** | Function scope | Block scope |
| **Hoisting** | `undefined` bilan | TDZ (uninitialized) |
| **Global da** | `window` ga qo'shadi | `window` ga qo'shMAYDI |

### Nima Uchun Ikki Environment Kerak?

```javascript
function example() {
  var x = 1;     // VariableEnvironment — function scope
  let y = 2;     // LexicalEnvironment — block scope

  if (true) {
    var x2 = 10;  // VariableEnvironment — function scope (block dan chiqadi!)
    let y2 = 20;  // YANGI LexicalEnvironment — faqat bu block ichida
    const z = 30; // YANGI LexicalEnvironment — faqat bu block ichida

    console.log(x);  // 1 ✅
    console.log(y);  // 2 ✅
    console.log(y2); // 20 ✅
  }

  console.log(x2); // 10 ✅ — var function scope, block dan chiqdi
  console.log(y2); // ❌ ReferenceError — let block scope, if ichida qoldi
}
```

### Under the Hood — Diagramma

```
function example() {
  var a = 1;
  let b = 2;

  if (true) {
    var c = 3;
    let d = 4;
    ...
  }
}

┌─────────────────────────────────────────────────────────┐
│  FUNCTION EXECUTION CONTEXT: example()                   │
│                                                          │
│  VariableEnvironment (function scope):                   │
│  ┌──────────────────────────────┐                        │
│  │  a: 1                        │                        │
│  │  c: 3  ← var, block dan chiqdi                       │
│  └──────────────────────────────┘                        │
│                                                          │
│  LexicalEnvironment (function scope):                    │
│  ┌──────────────────────────────┐                        │
│  │  b: 2                        │                        │
│  │  outer: → Global Environment │                        │
│  │                              │                        │
│  │  ┌──────────────────────┐   │  ← if block scope      │
│  │  │  d: 4                │   │                         │
│  │  │  outer: → function   │   │                         │
│  │  └──────────────────────┘   │                         │
│  └──────────────────────────────┘                        │
│                                                          │
│  ThisBinding: window                                     │
└─────────────────────────────────────────────────────────┘
```

**Muhim:** Har bir block (`if`, `for`, `while`, oddiy `{}`) yangi **LexicalEnvironment** yaratadi, lekin **VariableEnvironment** o'zgarmaydi (funksiya scope da qoladi).

---

## Environment Record

### Nazariya

Environment Record — bu LexicalEnvironment va VariableEnvironment ning **ichki ma'lumotlar bazasi**. Har bir Environment o'z ichida Environment Record saqlaydi va aynan shu record o'zgaruvchilar, funksiyalar, parametrlar haqidagi barcha ma'lumotlarni o'z ichiga oladi. Buni **ma'lumotlar bazasining jadvali**ga o'xshatish mumkin: environment — bu baza, record — bu jadval, o'zgaruvchilar — bu jadval satrlari.

ECMAScript spetsifikatsiyasiga ko'ra Environment Record ikki asosiy turga bo'linadi: **Declarative Environment Record** va **Object Environment Record**. Bu bo'linish sun'iy emas — u JavaScript'ning ichki arxitekturasidan kelib chiqadi. Funksiya va block scope'larda o'zgaruvchilar **to'g'ridan-to'g'ri** (optimallashtirilgan holda) saqlanadi — bu Declarative ER. Global scope'da esa `var` o'zgaruvchilari global ob'ekt (`window`) ning property'siga aylanadi — bu Object ER. Shu sababli `window.myVar` ishlaydi, lekin `window.myLet` ishlamaydi.

Bu farqni tushunish production kodda amaliy ahamiyatga ega. Masalan, kutubxona global `var` orqali o'zgaruvchi e'lon qilsa, u `window` ga tushadi va boshqa kutubxonalar bilan **nom to'qnashuvi** xavfi bor. `let`/`const` esa global ob'ektga qo'shilmaydi — bu muammoni bartaraf etadi. Shuning uchun zamonaviy kod doimo `let`/`const` ishlatadi.

ECMAScript spec da ikki turi bor:

### 1. Declarative Environment Record

Funksiya va block scope'lar uchun. `let`, `const`, `var`, `function`, `class`, `import` — barcha declaration'lar shu yerda.

```javascript
function greet(name) {
  // Declarative Environment Record:
  // {
  //   name: "Ali",          ← parameter
  //   greeting: undefined,  ← var (creation phase)
  //   LANG: <TDZ>,          ← const (creation phase)
  // }
  var greeting = "Salom";
  const LANG = "uz";
  return `${greeting}, ${name}! (${LANG})`;
}

greet("Ali");
```

### 2. Object Environment Record

**Faqat** Global Execution Context da `var` uchun ishlatiladi. `var` bilan e'lon qilingan global o'zgaruvchilar aslida global object (`window`) ning property'si bo'ladi.

```javascript
var globalVar = "salom";
// Object Environment Record:
// window.globalVar = "salom"

let globalLet = "dunyo";
// Declarative Environment Record:
// globalLet = "dunyo" (window da emas!)
```

### Under the Hood — Global EC ning Ikki Record'i

```
┌──────────────────────────────────────────────────┐
│         GLOBAL EXECUTION CONTEXT                  │
│                                                   │
│  VariableEnvironment:                             │
│  ┌─────────────────────────────────────────┐      │
│  │  Object Environment Record              │      │
│  │  (global object = window)               │      │
│  │  ┌───────────────────────────────────┐  │      │
│  │  │  globalVar: "salom"               │  │      │
│  │  │  myFunc: function(){...}          │  │      │
│  │  │  (window.globalVar bilan bir xil) │  │      │
│  │  └───────────────────────────────────┘  │      │
│  └─────────────────────────────────────────┘      │
│                                                   │
│  LexicalEnvironment:                              │
│  ┌─────────────────────────────────────────┐      │
│  │  Declarative Environment Record         │      │
│  │  ┌───────────────────────────────────┐  │      │
│  │  │  globalLet: "dunyo"               │  │      │
│  │  │  GLOBAL_CONST: 42                 │  │      │
│  │  │  (window da EMAS)                 │  │      │
│  │  └───────────────────────────────────┘  │      │
│  └─────────────────────────────────────────┘      │
│                                                   │
│  ThisBinding: window                              │
└──────────────────────────────────────────────────┘
```

---

## this Binding

### Nazariya

`this` — JavaScript'ning eng ko'p chalkashlik keltiradigan tushunchalaridan biri. Har bir Execution Context yaratilganda `this` qiymati ham aniqlanadi. Bu **Creation Phase** da sodir bo'ladi va `this` qiymati **funksiya qanday chaqirilganiga** bog'liq — qayerda yozilganiga emas.

Bu yerda `this` haqida qisqacha umumiy ko'rinish beramiz, chunki u Execution Context'ning tarkibiy qismi. Lekin `this` ning to'liq mexanizmi, binding qoidalari, priority tartibi va amaliy jihatlari juda keng mavzu — shuning uchun alohida [10-this-keyword.md](10-this-keyword.md) bo'limida batafsil yoritilgan.

`this` qanday aniqlanishi **kontekstga** bog'liq:

| Kontekst | `this` qiymati |
|----------|---------------|
| **Global EC** (non-strict) | `window` (browser), `global` (Node.js) |
| **Global EC** (strict mode) | `window` (global code da strict ham `window`) |
| **Function EC** (non-strict) | `window` (oddiy chaqiruv) |
| **Function EC** (strict mode) | `undefined` (oddiy chaqiruv) |
| **Method chaqiruv** | Object o'zi (`.` oldidagi object) |
| **Arrow function** | O'zining `this` i yo'q — tashqi scope dan oladi |
| **`new` keyword** | Yangi yaratilgan object |
| **`call`/`apply`/`bind`** | Berilgan object |

```javascript
// Global
console.log(this); // window

// Function (non-strict)
function show() {
  console.log(this); // window
}
show();

// Function (strict)
"use strict";
function showStrict() {
  console.log(this); // undefined
}
showStrict();

// Method
const user = {
  name: "Ali",
  greet() {
    console.log(this); // user object
  }
};
user.greet();

// Arrow function
const obj = {
  name: "Vali",
  greet: () => {
    console.log(this); // window — arrow function o'z this'i yo'q
  }
};
obj.greet();
```

Bu mavzu juda keng — to'liq [10-this-keyword.md](10-this-keyword.md) da.

---

## Execution Context Stack

### Nazariya

Execution Context Stack (yoki **Call Stack**) — bu engine'ning barcha Execution Context'larni tartib bilan saqlash va boshqarish uchun ishlatiladigan asosiy tuzilmasi. [01-js-engine.md](01-js-engine.md) da Call Stack haqida gaplangan edik — Execution Context Stack ham **aynan o'sha narsa**, faqat boshqa nom bilan.

```
Execution Context Stack = Call Stack
```

Bu tushunchani to'liq anglash muhim, chunki JavaScript'ning **single-threaded** tabiati aynan shu stack orqali namoyon bo'ladi. Engine har doim stack'ning **eng tepasidagi** context'ni bajaradi. Yangi funksiya chaqirilganda — uning EC'si tepaga qo'yiladi (push), funksiya tugaganda — olib tashlanadi (pop). Global EC har doim stack'ning eng pastida turadi va dastur yakunlanguncha yo'qolmaydi.

### To'liq Misol

```javascript
var language = "JavaScript";

function first() {
  var a = 10;
  second();
  var d = 40;
}

function second() {
  var b = 20;
  third();
  var e = 50;
}

function third() {
  var c = 30;
  console.log(c);
}

first();
console.log(language);
```

### Qadam-Baqadam Vizualizatsiya

**1-qadam: Dastur boshlandi — GEC yaratildi**

```
┌─────────────────────────────────────┐
│  GLOBAL EC                          │
│                                     │
│  Creation Phase:                    │
│    language: undefined              │
│    first: function(){...}           │
│    second: function(){...}          │
│    third: function(){...}           │
│    this: window                     │
│                                     │
│  Execution Phase:                   │
│    language = "JavaScript"          │
└─────────────────────────────────────┘

Stack: [Global EC]
```

**2-qadam: `first()` chaqirildi — FEC yaratildi**

```
┌─────────────────────────────┐
│  first() EC                 │
│  Creation: a=undefined,     │
│            d=undefined      │
│  Execution: a=10            │
│  → second() chaqirildi...   │
└─────────────────────────────┘
┌─────────────────────────────┐
│  GLOBAL EC                  │
│  language: "JavaScript"     │
└─────────────────────────────┘

Stack: [Global EC, first EC]
```

**3-qadam: `second()` chaqirildi**

```
┌─────────────────────────────┐
│  second() EC                │
│  Creation: b=undefined,     │
│            e=undefined      │
│  Execution: b=20            │
│  → third() chaqirildi...    │
└─────────────────────────────┘
┌─────────────────────────────┐
│  first() EC                 │
│  a=10, d=undefined          │
└─────────────────────────────┘
┌─────────────────────────────┐
│  GLOBAL EC                  │
│  language: "JavaScript"     │
└─────────────────────────────┘

Stack: [Global EC, first EC, second EC]
```

**4-qadam: `third()` chaqirildi**

```
┌─────────────────────────────┐
│  third() EC                 │
│  Creation: c=undefined      │
│  Execution: c=30            │
│  console.log(30) → "30"     │
└─────────────────────────────┘
┌─────────────────────────────┐
│  second() EC  (b=20)        │
└─────────────────────────────┘
┌─────────────────────────────┐
│  first() EC   (a=10)        │
└─────────────────────────────┘
┌─────────────────────────────┐
│  GLOBAL EC                  │
└─────────────────────────────┘

Stack: [Global EC, first EC, second EC, third EC]
```

**5-qadam: `third()` tugadi — pop**

```
Stack: [Global EC, first EC, second EC]
→ second() davom etadi: e = 50
```

**6-qadam: `second()` tugadi — pop**

```
Stack: [Global EC, first EC]
→ first() davom etadi: d = 40
```

**7-qadam: `first()` tugadi — pop**

```
Stack: [Global EC]
→ console.log("JavaScript") bajariladi
```

**8-qadam: Dastur tugadi**

```
Stack: [] (bo'sh — dastur yakunlandi)
```

Output:
```
30
JavaScript
```

---

## Common Mistakes

### ❌ Xato 1: var va let/const ning scope farqini bilmaslik

```javascript
function test() {
  if (true) {
    var x = 10;   // function scope — if dan tashqarida ham bor
    let y = 20;   // block scope — faqat if ichida
  }
  console.log(x); // 10 ✅
  console.log(y); // ❌ ReferenceError: y is not defined
}
```

### ✅ To'g'ri tushunish:

```javascript
function test() {
  // var → VariableEnvironment (function scope)
  // Shuning uchun if block'dan chiqadi

  // let/const → LexicalEnvironment (block scope)
  // Shuning uchun if block ichida qoladi

  let y; // Agar tashqarida kerak bo'lsa — tashqarida e'lon qiling
  if (true) {
    var x = 10;
    y = 20;
  }
  console.log(x); // 10
  console.log(y); // 20
}
```

**Nima uchun:** `var` VariableEnvironment ga tushadi (function scope), `let`/`const` LexicalEnvironment ga (block scope). If blokning o'z LexicalEnvironment'i bor, lekin VariableEnvironment tashqi funksiyadan keladi.

---

### ❌ Xato 2: Creation Phase da TDZ ni tushunmaslik

```javascript
// Kop odamlar "let hoist bo'lmaydi" deb o'ylaydi — bu NOTO'G'RI
console.log(a); // undefined (var hoist + undefined init)
console.log(b); // ❌ ReferenceError (let hoist qiladi, lekin TDZ da)

var a = 1;
let b = 2;
```

### ✅ To'g'ri tushunish:

```javascript
// let va const HAM hoist bo'ladi!
// Lekin ular TDZ (Temporal Dead Zone) da turadi
// — ya'ni hoist bo'lgan, lekin initialize qilinMAGAN

// Creation Phase:
// a → undefined (var: hoist + init)
// b → <uninitialized> (let: hoist, init YO'Q → TDZ)

// Execution Phase:
// console.log(a) → undefined (a mavjud, qiymati undefined)
// console.log(b) → ReferenceError (b mavjud, lekin TDZ da — kirish TAQIQLANGAN)
```

**Nima uchun:** Bu ko'proq [03-hoisting.md](03-hoisting.md) da, lekin asosiy tushuncha — TDZ "o'zgaruvchi yo'q" degani emas, "o'zgaruvchi bor, lekin hali kirish mumkin emas" degani.

---

### ❌ Xato 3: Global var va let farqini bilmaslik

```javascript
var globalVar = "salom";
let globalLet = "dunyo";

console.log(window.globalVar); // "salom" ✅
console.log(window.globalLet); // undefined ❌ (kutilganday "dunyo" emas!)
```

### ✅ To'g'ri tushunish:

```javascript
// var global scope da → Object Environment Record → window ga qo'shiladi
var globalVar = "salom";
console.log(window.globalVar);       // "salom"
console.log(globalVar);              // "salom"
console.log(window.globalVar === globalVar); // true

// let global scope da → Declarative Environment Record → window ga QO'SHILMAYDI
let globalLet = "dunyo";
console.log(window.globalLet);       // undefined
console.log(globalLet);              // "dunyo"
console.log(window.globalLet === globalLet); // false
```

**Nima uchun:** Global EC da ikki xil Environment Record bor. `var` → Object ER (window bilan bog'liq), `let`/`const` → Declarative ER (window bilan aloqasi yo'q).

---

### ❌ Xato 4: Funksiya har chaqirilganda yangi EC yaratilishini unutish

```javascript
function counter() {
  var count = 0;
  count++;
  return count;
}

console.log(counter()); // 1
console.log(counter()); // 1 — 2 emas!
console.log(counter()); // 1 — 3 emas!
```

### ✅ To'g'ri tushunish:

```javascript
// Har bir counter() chaqiruvi YANGI EC yaratadi
// Creation Phase: count = undefined
// Execution Phase: count = 0, count++, return 1
// EC yo'q qilinadi — count yo'qoladi

// Keyingi chaqiruvda yana YANGI EC — yana count = 0 dan boshlanadi

// Agar holatni saqlamoqchi bo'lsangiz — CLOSURE ishlatish kerak:
function makeCounter() {
  var count = 0;
  return function() {
    count++;
    return count;
  };
}

const counter2 = makeCounter();
console.log(counter2()); // 1
console.log(counter2()); // 2 ← closure tufayli count saqlanadi
console.log(counter2()); // 3
```

**Nima uchun:** Har bir funksiya chaqiruvi yangi FEC yaratadi, yangi VariableEnvironment bilan. Oldingi chaqiruvning o'zgaruvchilari yo'qoladi. Closure bu muammoni hal qiladi — [05-closures.md](05-closures.md) da batafsil.

---

### ❌ Xato 5: Execution order va async ni aralashtirib yuborish

```javascript
console.log("1");

setTimeout(function() {
  console.log("2");
}, 0);

console.log("3");

// Ko'pchilik kutadi: 1, 2, 3
// Aslida:           1, 3, 2
```

### ✅ To'g'ri tushunish:

```javascript
// 1. console.log("1") — Call Stack da bajariladi → "1"
// 2. setTimeout — Web API ga beriladi (0ms bo'lsa ham!)
//    Callback Queue ga qo'yiladi
// 3. console.log("3") — Call Stack da bajariladi → "3"
// 4. Call Stack bo'shadi
// 5. Event Loop: "Stack bo'shmi? Ha. Queue da bor? Ha."
//    → callback Stack ga qo'yiladi → "2"
```

**Nima uchun:** `setTimeout(fn, 0)` darhol bajarmaydi — u Web API orqali Callback Queue ga tushadi, faqat Call Stack to'liq bo'shagandan keyin bajariladi. Bu Event Loop mavzusi — [11-event-loop.md](11-event-loop.md) da to'liq.

---

## Amaliy Mashqlar

### Mashq 1: Creation Phase ni Aniqlang (Oson)

**Savol:** Quyidagi kodning Creation Phase da VariableEnvironment va LexicalEnvironment nimaga teng?

```javascript
function test() {
  var a = 10;
  let b = 20;
  const c = 30;
  function inner() { return "hi"; }
}
```

<details>
<summary>Javob</summary>

```
Creation Phase:

VariableEnvironment:
  a → undefined

LexicalEnvironment:
  b → <uninitialized> (TDZ)
  c → <uninitialized> (TDZ)
  inner → function() { return "hi"; }
```

**Tushuntirish:**
- `var a` → VariableEnvironment ga tushadi, `undefined` bilan initialize
- `let b`, `const c` → LexicalEnvironment ga, TDZ da (uninitialized)
- `function inner` → LexicalEnvironment ga, to'liq funksiya bilan (function hoisting)
</details>

---

### Mashq 2: Output ni Aniqlang (O'rta)

**Savol:** Quyidagi kodning output ini aniqlang:

```javascript
var x = 1;

function outer() {
  console.log(x); // ?
  var x = 2;
  console.log(x); // ?

  function inner() {
    console.log(x); // ?
    var x = 3;
    console.log(x); // ?
  }

  inner();
  console.log(x); // ?
}

outer();
console.log(x); // ?
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// undefined
// 2
// undefined
// 3
// 2
// 1
```

**Tushuntirish:**

1. `outer()` ning Creation Phase da **local** `var x = undefined` (hoist). Shuning uchun birinchi `console.log(x)` = `undefined` (global `x=1` ni emas, o'zining hoist qilingan `x` ini ko'radi — variable shadowing)
2. `x = 2` — outer ning `x` endi 2
3. `inner()` da ham **local** `var x = undefined` (hoist). Shuning uchun `console.log(x)` = `undefined`
4. `x = 3` — inner ning `x` endi 3
5. `inner()` tugadi — outer'ga qaytamiz. Outer ning `x` hali 2
6. Global'ga qaytamiz. Global `x` hali 1

```
Global EC:        outer() EC:       inner() EC:
x: 1              x: 2              x: 3
```

Har bir scope o'zining `x` ini yaratdi — bir-biriga ta'sir qilmaydi.
</details>

---

### Mashq 3: Global Object bilan Ishlash (O'rta)

**Savol:** Quyidagi kodning har bir `console.log` natijasini ayting:

```javascript
var a = "hello";
let b = "world";
const c = "!";

console.log(window.a); // ?
console.log(window.b); // ?
console.log(window.c); // ?
console.log(a);        // ?
console.log(b);        // ?

window.d = "test";
console.log(d);        // ?
```

<details>
<summary>Javob</summary>

```javascript
console.log(window.a); // "hello" — var global object ga qo'shiladi
console.log(window.b); // undefined — let qo'shilMAYDI
console.log(window.c); // undefined — const qo'shilMAYDI
console.log(a);        // "hello"
console.log(b);        // "world"

window.d = "test";
console.log(d);        // "test" — window.d = global property
```

**Tushuntirish:**
- `var` → Object Environment Record → `window` ning property'si
- `let`/`const` → Declarative Environment Record → `window` da yo'q
- `window.d = "test"` to'g'ridan-to'g'ri global object ga property qo'shadi — `var d` deb e'lon qilmasdan ham global da ko'rinadi
</details>

---

### Mashq 4: EC Stack Vizualizatsiya (Qiyin)

**Savol:** Quyidagi kodning output ini ayting va har bir qadam da EC Stack ni chizing:

```javascript
function a() {
  console.log("a start");
  b();
  console.log("a end");
}

function b() {
  console.log("b start");
  c();
  console.log("b end");
}

function c() {
  console.log("c");
  throw new Error("boom");
}

try {
  a();
} catch(e) {
  console.log("error caught");
}
console.log("done");
```

<details>
<summary>Javob</summary>

```
a start
b start
c
error caught
done
```

**EC Stack vizualizatsiya:**

```
1. [Global]                     → try { a(); }
2. [Global, a()]                → "a start"
3. [Global, a(), b()]           → "b start"
4. [Global, a(), b(), c()]      → "c"
5. throw Error("boom")!
   → c() unwind (pop)
   → b() unwind (pop) — "b end" BAJARILMAYDI
   → a() unwind (pop) — "a end" BAJARILMAYDI
   → Global'dagi catch ushladi
6. [Global]                     → "error caught"
7. [Global]                     → "done"
```

**Tushuntirish:**
- `throw` bo'lganda engine stack bo'ylab yuqoriga qarab `try/catch` qidiradi
- `c()`, `b()`, `a()` — hech birida `try/catch` yo'q, barchasi pop bo'ladi
- Global'dagi `try/catch` topildi — error ushlandi
- `b()` va `a()` ning qolgan kodlari ("b end", "a end") **bajarilmaydi**
</details>

---

### Mashq 5: Murakkab Hoisting + EC (Qiyin)

**Savol:** Quyidagi kodning output ini aniqlang:

```javascript
var x = 10;

function foo() {
  console.log(x);
  console.log(y);
  var x = 20;
  var y = 30;
  console.log(x);

  function bar() {
    console.log(x);
    var x = 40;
    console.log(x);
  }

  bar();
  console.log(x);
}

foo();
console.log(x);
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// undefined
// undefined
// 20
// undefined
// 40
// 20
// 10
```

**Qadam-baqadam:**

```
1. Global EC Creation:
   x = undefined, foo = function(){...}
   Global EC Execution:
   x = 10

2. foo() EC Creation:
   x = undefined (LOCAL var, global x ni shadow qiladi)
   y = undefined
   bar = function(){...}

3. foo() EC Execution:
   console.log(x) → undefined (local x hali assign bo'lmagan)
   console.log(y) → undefined (local y hali assign bo'lmagan)
   x = 20
   y = 30
   console.log(x) → 20

4. bar() EC Creation:
   x = undefined (LOCAL var, foo ning x ini shadow qiladi)

5. bar() EC Execution:
   console.log(x) → undefined (local x hali assign bo'lmagan)
   x = 40
   console.log(x) → 40

6. bar() tugadi, foo() davom:
   console.log(x) → 20 (foo ning x i, bar ta'sir qilmagan)

7. foo() tugadi, Global davom:
   console.log(x) → 10 (global x, foo ta'sir qilmagan)
```

**Kalit tushuncha:** Har bir `var x` o'z scope da yangi `x` yaratadi — tashqi `x` ni o'zgartirmaydi (variable shadowing).
</details>

---

## Xulosa

1. **Execution Context** — kod bajarilish muhiti: o'zgaruvchilar, scope chain, `this` haqidagi ma'lumotlar.

2. **3 turi** bor: Global EC (1 ta), Function EC (har bir chaqiruvda yangi), Eval EC.

3. **2 Phase:** Creation Phase (xotira ajratish, hoisting) va Execution Phase (kod bajarish).

4. **VariableEnvironment** — `var` uchun (function scope). **LexicalEnvironment** — `let`/`const`/`function` uchun (block scope).

5. **Environment Record** — o'zgaruvchilar haqiqatda shu yerda. Global da ikki xil: Object ER (`var` → window) va Declarative ER (`let`/`const`).

6. **`this`** Creation Phase da aniqlanadi — kontekstga bog'liq (global, function, method, arrow, new, call/apply/bind).

7. **EC Stack = Call Stack** — LIFO. Engine doim tepada turgan EC ni bajaradi.

8. Har bir funksiya chaqiruvi **yangi EC** yaratadi — o'z o'zgaruvchilari, o'z scope'i. Funksiya tugaganda EC yo'q qilinadi.

---

> **Keyingi bo'lim:** [03-hoisting.md](03-hoisting.md) — Hoisting — Creation Phase ning amaliy natijasi. var, let, const, function — kim qanday hoist bo'ladi.
