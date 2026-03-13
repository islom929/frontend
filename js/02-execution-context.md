# Bo'lim 2: Execution Context

> Execution Context — JavaScript engine kod bajarishdan oldin yaratiladigan ichki muhit bo'lib, o'zgaruvchilar, funksiyalar, scope chain va `this` qiymati qanday aniqlanishini belgilaydigan tuzilma.

---

## Mundarija

- [Execution Context Nima](#execution-context-nima)
- [Execution Context Turlari](#execution-context-turlari)
- [Global Execution Context (GEC)](#global-execution-context-gec)
- [Function Execution Context (FEC)](#function-execution-context-fec)
- [Eval Execution Context](#eval-execution-context)
- [Execution Context Phases](#execution-context-phases)
- [Creation Phase](#creation-phase)
- [Execution Phase](#execution-phase)
- [Variable Environment vs Lexical Environment](#variable-environment-vs-lexical-environment)
- [Environment Record](#environment-record)
- [Outer Environment Reference va Scope Chain](#outer-environment-reference-va-scope-chain)
- [this Binding](#this-binding)
- [Execution Context Stack](#execution-context-stack)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Execution Context Nima

### Nazariya

Execution Context (EC) — JavaScript engine har bir kod bo'lagini bajarishdan oldin yaratiladigan ichki muhit (environment). Bu muhit ichida engine quyidagilarni aniqlaydi:

1. **Qaysi o'zgaruvchilar va funksiyalar mavjud** — o'zgaruvchilar e'lon qilinadi, funksiya declaration'lar to'liq saqlanadi
2. **Scope chain** — joriy scope'dan tashqi scope'larga yo'l (o'zgaruvchi qidiruv zanjiri)
3. **`this` qiymati** — joriy kontekstda `this` nimaga teng

JavaScript da har qanday kod — global scope'dagi kod, funksiya ichidagi kod, yoki `eval` ichidagi kod — faqat execution context ichida bajariladi. Hech qanday kod execution context siz ishlamaydi.

Execution context'ni tushunish nima uchun kerak: hoisting, scope chain, closure, `this` keyword — bularning barchasi execution context mexanizmiga bog'liq. Bu tushunchalarni chuqur anglash uchun avval execution context'ning ichki tuzilmasini bilish zarur.

### Under the Hood

ECMAScript spetsifikatsiyasiga ko'ra, har bir execution context quyidagi komponentlarga ega:

```
┌──────────────────────────────────────┐
│        Execution Context             │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  LexicalEnvironment            │  │
│  │  (let, const, function, class) │  │
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

- **LexicalEnvironment** — `let`, `const`, `function`, `class` declaration'lar saqlanadigan muhit
- **VariableEnvironment** — `var` declaration'lar saqlanadigan muhit
- **ThisBinding** — joriy kontekstda `this` nimaga teng ekanligi

ES6 dan oldin LexicalEnvironment va VariableEnvironment bir xil edi. ES6 da `let`/`const` kiritilgandan keyin ular ajraldi — bu block scoping va TDZ (Temporal Dead Zone) mexanizmini ta'minlash uchun.

---

## Execution Context Turlari

### Nazariya

JavaScript da uch xil execution context mavjud:

1. **Global Execution Context (GEC)** — dastur boshlanganda birinchi bo'lib yaratiladi. Faqat **bitta** GEC bo'ladi
2. **Function Execution Context (FEC)** — har bir funksiya **chaqirilganda** yangi FEC yaratiladi. Har bir chaqiruv uchun alohida
3. **Eval Execution Context** — `eval()` funksiyasi chaqirilganda yaratiladi

```
┌───────────────────────────────────────────┐
│  Dastur boshlanganda:                     │
│                                           │
│  1. Global Execution Context yaratiladi   │
│     (faqat bitta, dastur davomida yashaydi)│
│                                           │
│  2. Funksiya chaqirilganda:               │
│     → Yangi Function EC yaratiladi        │
│     → Stack'ga qo'shiladi                 │
│     → Funksiya tugaganda stack'dan chiqadi │
│                                           │
│  3. eval() chaqirilganda:                 │
│     → Yangi Eval EC yaratiladi            │
│     (ishlatish tavsiya etilMAYDI)         │
└───────────────────────────────────────────┘
```

### Kod Misollari

```javascript
// 1. Global Execution Context — dastur boshlanganda yaratiladi
var globalVar = "global";
let globalLet = "global let";

// 2. Function Execution Context — har bir chaqiruvda yangi EC
function greet(name) {
  // greet() chaqirilganda yangi FEC yaratiladi
  var greeting = "Hello";
  return greeting + ", " + name;
}

greet("Ali");   // FEC #1 yaratildi → tugadi → stack'dan chiqdi
greet("Vali");  // FEC #2 yaratildi → tugadi → stack'dan chiqdi
// ✅ Har bir chaqiruv YANGI execution context yaratadi
// Oldingi chaqiruvning EC si allaqachon yo'q

// 3. Eval — ISHLATMANG
// eval("var x = 10"); // ❌ xavfsizlik va performance muammolari
```

---

## Global Execution Context (GEC)

### Nazariya

Global Execution Context — JavaScript dasturi boshlanganda eng birinchi yaratiladigan execution context. Butun dastur davomida faqat **bitta** GEC mavjud — u dastur tugaguncha yashaydi.

GEC yaratilganda quyidagilar sodir bo'ladi:

1. **Global object yaratiladi:**
   - Brauzerda → `window` object
   - Node.js da → `global` object
   - Har qanday muhitda → `globalThis` (ES2020) orqali accessible
2. **`this` qiymati global object'ga bog'lanadi** (non-strict mode da)
3. **`var` bilan e'lon qilingan o'zgaruvchilar** global object'ning property'si bo'ladi
4. **`let`/`const` bilan e'lon qilinganlar** global object'ga qo'shilMAYDI — ular alohida declarative environment record'da saqlanadi

### Under the Hood

GEC'ning ichki tuzilmasi:

```
Global Execution Context:
┌──────────────────────────────────────────────┐
│  LexicalEnvironment:                         │
│  ┌────────────────────────────────────────┐  │
│  │  GlobalEnvironmentRecord:              │  │
│  │  ├── ObjectEnvironmentRecord:          │  │
│  │  │   └── [[BindingObject]]: window     │  │
│  │  │       ├── var globalVar: undefined  │  │
│  │  │       ├── function greet: <func>    │  │
│  │  │       └── (built-ins: console, etc) │  │
│  │  │                                     │  │
│  │  └── DeclarativeEnvironmentRecord:     │  │
│  │      ├── let globalLet: <uninitialized>│  │
│  │      └── const PI: <uninitialized>     │  │
│  │                                        │  │
│  │  OuterRef: null (global — eng tashqi)  │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ThisBinding: window (globalThis)            │
└──────────────────────────────────────────────┘
```

Global scope'da ikki xil environment record ishlaydi:
- **Object Environment Record** — `var` va function declaration'lar `window` (global object) ga property sifatida qo'shiladi
- **Declarative Environment Record** — `let`, `const`, `class` declaration'lar alohida saqlanadi, `window`'ga qo'shilMAYDI

### Kod Misollari

```javascript
var name = "Ali";
let age = 25;
const PI = 3.14;

function greet() {
  return "Hello";
}

// var — global object'ning property'si bo'ladi
console.log(window.name);     // "Ali" ✅
console.log(globalThis.name); // "Ali" ✅

// let/const — global object'ga qo'shilMAYDI
console.log(window.age);      // undefined ❌
console.log(window.PI);       // undefined ❌
// ✅ Lekin global scope'da accessible:
console.log(age);             // 25
console.log(PI);              // 3.14

// function declaration — global object'ga qo'shiladi
console.log(window.greet);    // function greet() {...} ✅

// this — global scope'da window'ga teng (non-strict)
console.log(this === window); // true (brauzerda)
```

---

## Function Execution Context (FEC)

### Nazariya

Function Execution Context — har bir funksiya **chaqirilganda** (invoke/call) yangi yaratiladigan execution context. Funksiya necha marta chaqirilsa — shuncha yangi FEC yaratiladi. Har bir FEC o'zining alohida:

- Local o'zgaruvchilari (arguments, parameters, local variables)
- Scope chain'i (tashqi scope'ga havola)
- `this` qiymati

FEC yaratilish tartibi GEC bilan bir xil — avval Creation Phase, keyin Execution Phase. Farqi — FEC da qo'shimcha ravishda:
- **Arguments object** yaratiladi (arrow function'dan tashqari)
- **Parameter'lar** local o'zgaruvchi sifatida e'lon qilinadi
- **`this`** chaqirilish usulga qarab aniqlanadi (GEC da esa doim global object)

### Under the Hood

```javascript
function calculateTotal(price, taxRate) {
  var discount = 0.1;
  let finalPrice = price * (1 + taxRate) * (1 - discount);
  return finalPrice;
}

calculateTotal(100, 0.2);
```

`calculateTotal(100, 0.2)` chaqirilganda yaratiladigan FEC:

```
Function Execution Context (calculateTotal):
┌──────────────────────────────────────────────┐
│  LexicalEnvironment:                         │
│  ┌────────────────────────────────────────┐  │
│  │  DeclarativeEnvironmentRecord:         │  │
│  │  ├── let finalPrice: <uninitialized>   │  │
│  │  │                                     │  │
│  │  OuterRef: → Global Environment        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  VariableEnvironment:                        │
│  ┌────────────────────────────────────────┐  │
│  │  DeclarativeEnvironmentRecord:         │  │
│  │  ├── var discount: undefined           │  │
│  │  ├── arguments: { 0: 100, 1: 0.2 }    │  │
│  │  ├── price: 100                        │  │
│  │  └── taxRate: 0.2                      │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ThisBinding: window (default binding)       │
└──────────────────────────────────────────────┘
```

Creation Phase da:
- `var discount` → `undefined` bilan initialize qilinadi
- `let finalPrice` → **uninitialized** (TDZ da) — hali qiymat berilmagan
- `price`, `taxRate` → argument qiymatlari bilan initialize
- `arguments` object yaratiladi

Execution Phase da:
- `discount = 0.1` — qiymat beriladi
- `finalPrice = 100 * 1.2 * 0.9` — hisob bajariladi, qiymat yoziladi
- `return finalPrice` — natija qaytariladi, FEC stack'dan chiqadi

### Kod Misollari

```javascript
// Har bir chaqiruv YANGI FEC yaratadi
function counter() {
  var count = 0;
  count++;
  return count;
}

console.log(counter()); // 1 — FEC #1: count = 0 → 1
console.log(counter()); // 1 — FEC #2: count = 0 → 1 (yangi EC, yangi count)
console.log(counter()); // 1 — FEC #3: count = 0 → 1

// ✅ Har safar yangi EC — har safar count 0 dan boshlanadi
// Bu closure bo'lmagani uchun oldingi count saqlanmaydi
```

```javascript
// Nested function — ichki FEC tashqi FEC ga outer reference orqali bog'lanadi
function outer() {
  var x = 10;

  function inner() {
    var y = 20;
    console.log(x + y); // ✅ x — outer scope'dan (scope chain orqali)
  }

  inner();
}

outer(); // 30

// inner() ning FEC:
// LexicalEnvironment:
//   y: 20
//   OuterRef: → outer() ning Environment
//                x: 10
//                OuterRef: → Global Environment
```

---

## Eval Execution Context

### Nazariya

`eval()` funksiyasi chaqirilganda yangi execution context yaratiladi. `eval()` string sifatida berilgan JavaScript kodni parse qiladi va bajaradi.

```javascript
eval("var x = 10; console.log(x);"); // 10
```

**`eval()` ni ishlatmaslik kerak.** Sabablari:

1. **Xavfsizlik** — tashqi source'dan kelgan string'ni eval qilish code injection xavfini yaratadi
2. **Performance** — eval ichidagi kod engine tomonidan optimize qilinMAYDI. Engine eval mavjud bo'lgan scope'dagi barcha optimization'larni bekor qiladi, chunki eval ixtiyoriy o'zgaruvchilarni yaratishi/o'zgartirishi mumkin
3. **Debugging** — eval ichidagi kod stack trace'da noma'lum ko'rinadi
4. **Strict mode** — strict mode'da eval o'z scope'ini yaratadi (calling scope'ga ta'sir qilmaydi), lekin bu ham yetarli emas

```javascript
// ❌ eval ishlatmang
eval("var injected = 'xavfli kod'");
console.log(injected); // "xavfli kod" — calling scope'ga inject bo'ldi!

// ✅ O'rniga: JSON.parse (data uchun), Function constructor (oxirgi chora), yoki boshqa pattern
const data = JSON.parse('{"name": "Ali"}');
```

---

## Execution Context Phases

### Nazariya

Har bir execution context (GEC yoki FEC) **ikki bosqichda** yaratiladi va bajariladi:

1. **Creation Phase** — EC yaratilish bosqichi. Kod hali bajarilMAYDI. Bu bosqichda engine muhitni tayyorlaydi
2. **Execution Phase** — Kod qator-baqatar bajariladi. O'zgaruvchilarga qiymatlar beriladi, funksiyalar chaqiriladi

Bu ikki bosqichli mexanizm **hoisting** ning asosiy sababi. Creation Phase da `var` o'zgaruvchilar `undefined` bilan, function declaration'lar to'liq function bilan initialize qilinadi — shuning uchun ularni e'lon qilishdan oldin ishlatish mumkin.

```
┌─────────────────────────────────────────────┐
│  Execution Context Lifecycle                │
│                                             │
│  1. CREATION PHASE (kod bajarilMAYDI):      │
│     ├── LexicalEnvironment yaratiladi       │
│     ├── VariableEnvironment yaratiladi       │
│     ├── this binding aniqlanadi             │
│     ├── var → undefined bilan initialize    │
│     ├── let/const → uninitialized (TDZ)     │
│     ├── function declaration → to'liq func  │
│     └── Scope chain quriladi (outer ref)    │
│                                             │
│  2. EXECUTION PHASE (kod bajariladi):       │
│     ├── Qator-baqatar kod bajariladi        │
│     ├── var → haqiqiy qiymat beriladi       │
│     ├── let/const → qiymat beriladi (TDZ    │
│     │   tugaydi)                            │
│     ├── Funksiyalar chaqiriladi             │
│     └── Expression'lar evaluate qilinadi    │
└─────────────────────────────────────────────┘
```

---

## Creation Phase

### Nazariya

Creation Phase — execution context'ning birinchi bosqichi. Bu bosqichda kod **hali bajarilmaydi**, faqat muhit tayyorlanadi. Engine source code'ni scan qiladi va quyidagilarni bajaradi:

**1. LexicalEnvironment yaratiladi:**
- `let`, `const` declaration'lar topiladi va **uninitialized** holatda ro'yxatga olinadi (TDZ boshlanadi)
- `function` declaration'lar topiladi va **to'liq funksiya** bilan initialize qilinadi
- `class` declaration'lar topiladi va **uninitialized** holatda ro'yxatga olinadi (TDZ)

**2. VariableEnvironment yaratiladi:**
- `var` declaration'lar topiladi va **`undefined`** bilan initialize qilinadi

**3. this binding aniqlanadi:**
- Global context'da → `globalThis` (window/global)
- Function context'da → chaqirilish usulga qarab

**4. Scope chain quriladi:**
- Outer environment reference o'rnatiladi — tashqi scope'ga havola

### Under the Hood

Creation Phase ni qadam-baqadam ko'rsatadigan misol:

```javascript
console.log(x);       // undefined (var — creation da undefined)
console.log(y);       // ReferenceError (let — TDZ da)
console.log(greet);   // function greet() {...} (to'liq hoist)
console.log(MyClass); // ReferenceError (class — TDZ da)

var x = 10;
let y = 20;
function greet() { return "Hello"; }
class MyClass {}
```

Creation Phase da engine nima qiladi:

```
CREATION PHASE (kod bajarilmaydi):

VariableEnvironment:
  x: undefined          ← var → undefined bilan initialize

LexicalEnvironment:
  y: <uninitialized>    ← let → TDZ (Temporal Dead Zone)
  greet: function() {}  ← function declaration → to'liq function
  MyClass: <uninitialized> ← class → TDZ

EXECUTION PHASE (kod bajariladi):
  console.log(x)       → undefined (var allaqachon undefined)
  console.log(y)       → ReferenceError! (y hali TDZ da)
  console.log(greet)   → function greet() {...} (allaqachon to'liq)
  console.log(MyClass) → ReferenceError! (hali TDZ da)
  x = 10               → x endi 10
  y = 20               → y endi 20, TDZ tugadi
  MyClass = class {}   → MyClass endi class, TDZ tugadi
```

### Kod Misollari

```javascript
// Creation Phase'ni tushunish uchun muhim misol:
var a = 1;
function b() {
  return 2;
}
var c = function() {
  return 3;
};

// Creation Phase tugaganda, Execution Phase boshlanishidan OLDIN:
// a = undefined       (var → undefined)
// b = function b(){}  (function declaration → to'liq)
// c = undefined       (var → undefined, function EXPRESSION hoist bo'lmaydi)
```

```javascript
// Function declaration vs Function expression hoisting farqi:
console.log(sum(2, 3));      // ✅ 5 — function declaration to'liq hoist
console.log(multiply(2, 3)); // ❌ TypeError: multiply is not a function
// multiply hozir undefined (var hoist), function emas

function sum(a, b) {
  return a + b;
}

var multiply = function(a, b) {
  return a * b;
};
// ✅ sum — function declaration: Creation Phase da to'liq funksiya sifatida saqlanadi
// ❌ multiply — var + function expression: Creation Phase da faqat var → undefined
```

---

## Execution Phase

### Nazariya

Execution Phase — Creation Phase tugagandan keyin boshlanaladigan ikkinchi bosqich. Bu yerda JavaScript kodi **qator-baqatar** bajariladi:

1. O'zgaruvchilarga haqiqiy qiymatlar beriladi (`var x = 10` → x endi 10)
2. `let`/`const` ning TDZ tugaydi (qiymat berilganda)
3. Funksiyalar chaqiriladi (har bir chaqiruv yangi FEC yaratadi)
4. Expression'lar evaluate qilinadi
5. Conditional va loop'lar bajariladi

### Kod Misollari

Creation va Execution Phase'larni birga ko'rsatish:

```javascript
var name = "Ali";
let age = 25;

function getInfo() {
  var result = name + " - " + age;
  return result;
}

var info = getInfo();
console.log(info);
```

```
=== GLOBAL EC — CREATION PHASE ===
VariableEnvironment:
  name: undefined
  info: undefined
LexicalEnvironment:
  age: <uninitialized>
  getInfo: function getInfo() {...}
ThisBinding: window

=== GLOBAL EC — EXECUTION PHASE ===
Qator 1: name = "Ali"        → name endi "Ali"
Qator 2: age = 25            → age endi 25 (TDZ tugadi)

Qator 8: getInfo() chaqirildi
  → Yangi FEC yaratiladi:

    === getInfo FEC — CREATION PHASE ===
    VariableEnvironment:
      result: undefined
    LexicalEnvironment:
      OuterRef: → Global Environment
    ThisBinding: window

    === getInfo FEC — EXECUTION PHASE ===
    result = "Ali" + " - " + 25  → result = "Ali - 25"
    return "Ali - 25"
    → FEC stack'dan chiqadi

Qator 8 davomi: info = "Ali - 25"
Qator 9: console.log("Ali - 25")  → "Ali - 25"
```

---

## Variable Environment vs Lexical Environment

### Nazariya

ES6 dan boshlab har bir execution context'da **ikkita** environment bor:

**VariableEnvironment:**
- Faqat `var` declaration'lar saqlanadi
- `var` function-scoped — block'lar (if, for, while) ichida e'lon qilingan `var` ham shu yerda
- Creation Phase da `undefined` bilan initialize qilinadi

**LexicalEnvironment:**
- `let`, `const`, `function`, `class` declaration'lar saqlanadi
- `let`/`const` block-scoped — har bir block uchun yangi LexicalEnvironment yaratilishi mumkin
- Creation Phase da `let`/`const` **uninitialized** (TDZ)
- `function` declaration → to'liq funksiya bilan initialize

Nima uchun ikki alohida environment kerak: `var` function-scoped, `let`/`const` block-scoped. Agar hammasi bitta environment'da bo'lsa — block scoping mexanizmini implement qilish murakkablashadi. Alohida environment'lar tufayli engine block ichida yangi LexicalEnvironment yaratib, `let`/`const`'ni shu block'ga cheklash oson.

### Under the Hood

```javascript
function example() {
  var a = 1;
  let b = 2;

  if (true) {
    var c = 3;   // ← function scope (VariableEnvironment)
    let d = 4;   // ← block scope (yangi LexicalEnvironment)
    const e = 5; // ← block scope (yangi LexicalEnvironment)
  }

  console.log(a); // 1 ✅
  console.log(b); // 2 ✅
  console.log(c); // 3 ✅ — var function-scoped, if block cheklamaydi
  console.log(d); // ❌ ReferenceError — let block-scoped, if dan tashqarida yo'q
  console.log(e); // ❌ ReferenceError — const block-scoped
}
```

Ichki tuzilma:

```
example() FEC:

VariableEnvironment (function scope):
┌──────────────────────┐
│  a: undefined → 1    │
│  c: undefined → 3    │  ← if ichidagi var ham shu yerda!
└──────────────────────┘

LexicalEnvironment (function scope):
┌──────────────────────┐
│  b: <uninitialized>  │  → 2
│  OuterRef: → Global  │
└──────────────────────┘

if block LexicalEnvironment (block scope):
┌──────────────────────┐
│  d: <uninitialized>  │  → 4
│  e: <uninitialized>  │  → 5
│  OuterRef: → Function│  ← tashqi function scope'ga
│             LexEnv   │     bog'lanadi
└──────────────────────┘
```

if block tugaganda, block LexicalEnvironment yo'qoladi — shuning uchun `d` va `e` block tashqarisida accessible emas.

---

## Environment Record

### Nazariya

Environment Record — har bir LexicalEnvironment ichidagi o'zgaruvchilar va binding'lar saqlanadigan tuzilma. Bu ECMAScript spec'dagi termin bo'lib, aslida "scope'dagi barcha o'zgaruvchilar ro'yxati" degani.

Environment Record ikki asosiy turga bo'linadi:

**1. Declarative Environment Record:**
- `let`, `const`, `var`, `function`, `class`, `import`, `catch` parameter binding'lari saqlanadi
- Funksiya va block scope'larda ishlatiladi
- O'zgaruvchilar to'g'ridan-to'g'ri record ichida saqlanadi — tez

**2. Object Environment Record:**
- Biror object'ning property'larini binding sifatida ko'rsatadi
- Global scope'da `var` va function declaration'lar `window` object orqali — Object Environment Record
- `with` statement'da ishlatiladi (taqiqlangan)

**3. Global Environment Record** (maxsus):
- Object Environment Record + Declarative Environment Record birikmasi
- `var` va function → Object ER (window'ga qo'shiladi)
- `let`, `const`, `class` → Declarative ER (window'ga qo'shilMAYDI)

### Under the Hood

```
Global Environment Record:
┌──────────────────────────────────────────┐
│                                          │
│  Object Environment Record:              │
│  ┌────────────────────────────────────┐  │
│  │  [[BindingObject]]: window         │  │
│  │  ├── var name: "Ali"               │  │
│  │  ├── function greet: <func>        │  │
│  │  ├── parseInt: <built-in>          │  │
│  │  ├── console: <built-in>           │  │
│  │  └── ... (window properties)       │  │
│  └────────────────────────────────────┘  │
│                                          │
│  Declarative Environment Record:         │
│  ┌────────────────────────────────────┐  │
│  │  let age: 25                       │  │
│  │  const PI: 3.14                    │  │
│  │  class User: <class>               │  │
│  └────────────────────────────────────┘  │
│                                          │
└──────────────────────────────────────────┘
```

Function scope'larda faqat Declarative Environment Record ishlatiladi — bu engine uchun optimizatsiya imkonini beradi (o'zgaruvchi indexi oldindan ma'lum, hash lookup kerak emas).

### Kod Misollari

```javascript
// Global scope'da var vs let farqi — Environment Record turlari tufayli
var globalVar = "var bilan";
let globalLet = "let bilan";

// var → Object Environment Record → window property
console.log(window.globalVar);    // "var bilan" ✅
console.log('globalVar' in window); // true

// let → Declarative Environment Record → window'da yo'q
console.log(window.globalLet);    // undefined
console.log('globalLet' in window); // false

// Lekin ikkisi ham global scope'da accessible:
console.log(globalVar); // "var bilan" ✅
console.log(globalLet); // "let bilan" ✅
```

---

## Outer Environment Reference va Scope Chain

### Nazariya

Har bir LexicalEnvironment'da **Outer Environment Reference** (outer ref) mavjud — bu tashqi (parent) scope'ning LexicalEnvironment'iga havola. Shu havola orqali **Scope Chain** quriladi.

O'zgaruvchi qidirilganda engine quyidagi tartibda ishlaydi:

1. Joriy Environment Record'da qidiradi
2. Topilmasa → outer ref bo'ylab tashqi scope'ga o'tadi
3. Tashqida ham topilmasa → yana tashqiga
4. Global scope'gacha davom etadi
5. Global scope'da ham topilmasa → `ReferenceError`

Outer reference **statik** (lexical) — funksiya **yozilgan** joyga qarab aniqlanadi, **chaqirilgan** joyga emas. Bu lexical scoping'ning asosi.

### Under the Hood

```javascript
var globalX = "global";

function outer() {
  var outerX = "outer";

  function middle() {
    var middleX = "middle";

    function inner() {
      var innerX = "inner";
      console.log(innerX);   // "inner"   — o'z scope'ida
      console.log(middleX);  // "middle"  — 1 ta tashqi
      console.log(outerX);   // "outer"   — 2 ta tashqi
      console.log(globalX);  // "global"  — 3 ta tashqi (global)
    }

    inner();
  }

  middle();
}

outer();
```

Scope Chain vizualizatsiya:

```
inner() Environment Record:
  innerX: "inner"
  outer ref ─────→ middle() Environment Record:
                     middleX: "middle"
                     outer ref ─────→ outer() Environment Record:
                                        outerX: "outer"
                                        outer ref ─────→ Global Environment Record:
                                                           globalX: "global"
                                                           outer ref: null (tugadi)
```

`console.log(outerX)` bajarilganda:
1. `inner()` → outerX yo'q
2. `middle()` → outerX yo'q
3. `outer()` → outerX = "outer" ✅ — topildi!

---

## this Binding

### Nazariya

Har bir execution context yaratilganda `this` qiymati aniqlanadi. `this` **compile-time** da emas, **runtime** da — funksiya **qanday chaqirilganiga** qarab belgilanadi (arrow function bundan mustasno).

Bu bo'limda faqat asosiy tushuncha beriladi — to'liq `this` keyword mexanizmi [10-this-keyword.md](10-this-keyword.md) da yoritiladi.

**Global Execution Context da this:**
- Non-strict mode: `this === globalThis` (window/global)
- Strict mode: `this === globalThis`
- Module scope: `this === undefined`

**Function Execution Context da this:**
- Default binding: `this === globalThis` (non-strict) yoki `undefined` (strict)
- Method call: `this === object` (method chaqirgan object)
- `new` binding: `this === yangi yaratilgan object`
- Explicit binding: `call`/`apply`/`bind` bilan belgilangan qiymat
- Arrow function: o'zining `this` i yo'q — tashqi scope'dan oladi (lexical this)

### Kod Misollari

```javascript
// Global context'da this
console.log(this === window); // true (brauzer, non-strict)

// Function context'da this
function showThis() {
  console.log(this);
}

showThis();           // window (non-strict) / undefined (strict)

const user = {
  name: "Ali",
  greet() {
    console.log(this.name); // ✅ "Ali" — method call, this = user
  }
};

user.greet(); // "Ali"

// Arrow function — this ni tashqi scope'dan oladi
const team = {
  name: "Dev Team",
  getMembers() {
    const arrow = () => {
      console.log(this.name); // ✅ "Dev Team" — tashqi scope (getMembers) dan
    };
    arrow();
  }
};

team.getMembers(); // "Dev Team"
```

---

## Execution Context Stack

### Nazariya

Execution Context Stack (Call Stack bilan bir xil narsa) — barcha execution context'larni LIFO tartibida saqlaydigan stack. JavaScript single-threaded — bir vaqtda faqat bitta EC bajariladi (stack'ning eng tepasidagi).

Ishlash tartibi:
1. Dastur boshlanganda → GEC yaratiladi va stack'ga qo'shiladi
2. Funksiya chaqirilganda → yangi FEC yaratiladi va stack'ning tepasiga qo'shiladi
3. Funksiya tugaganda → uning FEC'i stack'dan olib tashlanadi
4. Boshqaruv oldingi EC'ga qaytadi
5. Dastur tugaganda → GEC ham stack'dan chiqadi

### Under the Hood

```javascript
var x = 10;

function first() {
  var a = 20;
  second();
  console.log(a);
}

function second() {
  var b = 30;
  third();
}

function third() {
  var c = 40;
  console.log(c);
}

first();
console.log(x);
```

EC Stack holati qadam-baqadam:

```
Boshlang'ich:
┌───────────┐
│  GEC      │ ← x: undefined → 10
└───────────┘

first() chaqirildi:
┌───────────┐
│  first()  │ ← a: undefined → 20
│  GEC      │
└───────────┘

second() chaqirildi (first ichidan):
┌───────────┐
│  second() │ ← b: undefined → 30
│  first()  │
│  GEC      │
└───────────┘

third() chaqirildi (second ichidan):
┌───────────┐
│  third()  │ ← c: undefined → 40
│  second() │
│  first()  │
│  GEC      │
└───────────┘

third() console.log(40) → tugadi:
┌───────────┐
│  second() │
│  first()  │
│  GEC      │
└───────────┘

second() tugadi:
┌───────────┐
│  first()  │
│  GEC      │
└───────────┘

first() console.log(20) → tugadi:
┌───────────┐
│  GEC      │
└───────────┘

console.log(10) → dastur tugadi:
┌───────────┐
│  (bo'sh)  │
└───────────┘

Output: 40, 20, 10
```

**Execution Context Stack = Call Stack.** [01-js-engine.md](01-js-engine.md) da call stack haqida asosiy tushuncha berilgan edi — bu bo'limda EC tafsilotlari qo'shildi.

---

## Common Mistakes

### ❌ Xato 1: var hoisting'ni tushunmaslik

```javascript
// ❌ Noto'g'ri kutish — "x hali e'lon qilinmagan, xato beradi"
console.log(x); // undefined — xato emas!
var x = 5;
// Ko'pchilik ReferenceError kutadi, lekin aslida undefined chiqadi
```

### ✅ To'g'ri tushunish:

```javascript
// ✅ Creation Phase da var x undefined bilan initialize qilinadi
// Shuning uchun console.log(x) → undefined (xato emas)
// Execution Phase da x = 5 beriladi

// Agar let/const bo'lsa — ReferenceError berardi:
console.log(y); // ❌ ReferenceError: Cannot access 'y' before initialization
let y = 5;
// let Creation Phase da <uninitialized> — TDZ da
```

**Nima uchun:** Creation Phase da `var` → `undefined`, `let`/`const` → `uninitialized` (TDZ). `undefined` o'qish mumkin, `uninitialized` o'qish `ReferenceError` beradi.

---

### ❌ Xato 2: var'ni block-scoped deb o'ylash

```javascript
// ❌ Noto'g'ri — var block scope'da cheklanadi deb kutish
if (true) {
  var secret = "hidden";
}
console.log(secret); // "hidden" — ❌ ko'pchilik undefined yoki xato kutadi
```

### ✅ To'g'ri tushunish:

```javascript
// ✅ var function-scoped — faqat function chegarasida qoladi, block emas
// if, for, while ichidagi var → function scope'ga ko'tariladi

if (true) {
  let blockScoped = "hidden";
}
console.log(blockScoped); // ✅ ReferenceError — let block-scoped
```

**Nima uchun:** `var` VariableEnvironment'da saqlanadi — bu function-level. `let`/`const` LexicalEnvironment'da — block-level. If block o'z VariableEnvironment'ini yaratMAYDI, faqat LexicalEnvironment yaratadi.

---

### ❌ Xato 3: Function expression'ni declaration kabi hoist bo'ladi deb o'ylash

```javascript
// ❌ Noto'g'ri kutish
greet(); // ✅ ishlaydi
hello(); // ❌ TypeError: hello is not a function

function greet() { console.log("Hi"); }     // declaration
var hello = function() { console.log("Hello"); }; // expression
```

### ✅ To'g'ri tushunish:

```javascript
// ✅ Creation Phase da:
// greet → function greet() {...}  (to'liq hoist — function declaration)
// hello → undefined               (faqat var hoist — expression emas)

// shuning uchun greet() ishlaydi, hello() esa TypeError beradi
// TypeError, ReferenceError emas — chunki hello MAVJUD (undefined), lekin function emas
```

**Nima uchun:** Function declaration'lar Creation Phase da to'liq funksiya sifatida saqlanadi. Function expression'lar — faqat o'zgaruvchi sifatida (`var` → `undefined`), funksiya qiymati Execution Phase da beriladi.

---

### ❌ Xato 4: Global scope'da let/const window'da bor deb o'ylash

```javascript
// ❌ Noto'g'ri kutish
let apiKey = "abc123";
console.log(window.apiKey); // undefined — ❌ topilmaydi!
```

### ✅ To'g'ri tushunish:

```javascript
// ✅ let/const Global Environment Record'ning
// Declarative qismida saqlanadi — Object (window) da emas

var oldStyle = "window da";
let newStyle = "window da emas";

console.log(window.oldStyle); // "window da" ✅
console.log(window.newStyle); // undefined ❌
// ✅ Lekin ikkisi ham global scope'da accessible:
console.log(oldStyle); // "window da" ✅
console.log(newStyle); // "window da emas" ✅
```

**Nima uchun:** Global scope'da `var` → Object Environment Record (window ga property bo'ladi). `let`/`const` → Declarative Environment Record (window ga qo'shilMAYDI). Bu spec'da belgilangan farq.

---

### ❌ Xato 5: Execution context va scope'ni adashtirib yuborish

```javascript
// ❌ Noto'g'ri tushuncha: "funksiya chaqirilgan joyning scope'i ishlatiladi"
var x = "global";

function showX() {
  console.log(x);
}

function wrapper() {
  var x = "wrapper";
  showX(); // "global" — "wrapper" emas!
}

wrapper();
```

### ✅ To'g'ri tushunish:

```javascript
// ✅ JavaScript LEXICAL scoping ishlatadi
// showX() YOZILGAN joyda global scope'da — outer ref: Global Environment
// showX() CHAQIRILGAN joy (wrapper ichida) ahamiyatsiz

// Creation Phase da showX ning outer ref = Global Environment
// Execution Phase da x qidiradi: showX scope'ida yo'q → Global da x = "global"
// wrapper'dagi x ga hech qachon bormaydi
```

**Nima uchun:** Scope chain funksiya **yozilgan** joyga qarab quriladi (lexical scoping), **chaqirilgan** joyga emas (dynamic scoping). JavaScript lexical scoping ishlatadi.

---

## Amaliy Mashqlar

### Mashq 1: Creation Phase (Oson)

**Savol:** Quyidagi kodda Creation Phase tugaganda har bir o'zgaruvchining holati qanday bo'ladi?

```javascript
var a = 1;
let b = 2;
const c = 3;
function d() { return 4; }
var e = function() { return 5; };
```

<details>
<summary>Javob</summary>

```
Creation Phase tugaganda (kod hali bajarilMAGAN):

VariableEnvironment:
  a: undefined         ← var → undefined
  e: undefined         ← var → undefined (function expression — faqat var hoist)

LexicalEnvironment:
  b: <uninitialized>   ← let → TDZ
  c: <uninitialized>   ← const → TDZ
  d: function d() {}   ← function declaration → to'liq function
```

**Tushuntirish:**
- `var` → `undefined` bilan initialize
- `let`/`const` → `uninitialized` (TDZ da — o'qilsa ReferenceError)
- function declaration → to'liq funksiya
- function expression (`var e = function...`) → faqat `var e = undefined`
</details>

---

### Mashq 2: Execution Order (Oson)

**Savol:** Output nima bo'ladi?

```javascript
console.log(typeof greet);
console.log(typeof hello);

function greet() { return "Hi"; }
var hello = function() { return "Hello"; };

console.log(typeof greet);
console.log(typeof hello);
```

<details>
<summary>Javob</summary>

```
"function"
"undefined"
"function"
"function"
```

**Tushuntirish:**
- 1-qator: `greet` — function declaration, Creation Phase da to'liq hoist → `typeof` = `"function"`
- 2-qator: `hello` — `var` bilan, Creation Phase da `undefined` → `typeof undefined` = `"undefined"`
- 3-qator: hech narsa o'zgarmadi — `greet` hali ham function → `"function"`
- 4-qator: Execution Phase da `hello = function(){}` assign bo'ldi → `typeof` = `"function"`
</details>

---

### Mashq 3: Scope Chain (O'rta)

**Savol:** Har bir `console.log` ning natijasini ayting.

```javascript
var x = 10;

function outer() {
  var x = 20;

  function inner() {
    console.log(x); // ?
  }

  inner();
}

outer();
console.log(x); // ?
```

<details>
<summary>Javob</summary>

```
20
10
```

**Tushuntirish:**
- `inner()` ichida `x` qidiriladi → inner'da yo'q → outer ref bo'ylab outer'ga → `x = 20` topildi
- Global scope'da `console.log(x)` → global `x = 10`
- inner() outer() ichida **yozilgan** — shuning uchun outer scope'ga bog'langan (lexical scoping)
</details>

---

### Mashq 4: var vs let block scope (O'rta)

**Savol:** Output nima bo'ladi?

```javascript
function test() {
  console.log(a); // ?
  console.log(b); // ?

  if (true) {
    var a = 1;
    let b = 2;
  }

  console.log(a); // ?
  console.log(b); // ?
}

test();
```

<details>
<summary>Javob</summary>

```
undefined
ReferenceError: Cannot access 'b' before initialization
```

Kod 2-qatorda to'xtaydi — ReferenceError tufayli 3 va 4-qatorgacha yetmaydi.

Agar 2-qatordagi `console.log(b)` o'chirilsa:

```
undefined     ← var a hoist bo'ldi, undefined
1             ← if ichidagi var a = 1 function scope'da
ReferenceError ← let b block scope'da — if tashqarisida yo'q
```

**Tushuntirish:**
- `var a` function scope'ga hoist bo'ladi → if tashqarisida ham accessible (undefined, keyin 1)
- `let b` block scope → faqat if ichida accessible, tashqarida ReferenceError

Birinchi `console.log(b)` da: `b` function scope'da yo'q. Test function scope'ining LexicalEnvironment'ida `b` qidiriladi — topilmaydi. Global'da ham yo'q → ReferenceError.
</details>

---

### Mashq 5: Execution Context Stack (Qiyin)

**Savol:** EC Stack'ning holatini har bir qadam uchun chizing. Output nima?

```javascript
var result = [];

function first() {
  result.push("first start");
  second();
  result.push("first end");
}

function second() {
  result.push("second start");
  third();
  result.push("second end");
}

function third() {
  result.push("third");
}

first();
console.log(result);
```

<details>
<summary>Javob</summary>

```javascript
["first start", "second start", "third", "second end", "first end"]
```

EC Stack holati:

```
1. first()  chaqirildi  → Stack: [GEC, first]    → "first start"
2. second() chaqirildi  → Stack: [GEC, first, second] → "second start"
3. third()  chaqirildi  → Stack: [GEC, first, second, third] → "third"
4. third()  tugadi      → Stack: [GEC, first, second] → "second end"
5. second() tugadi      → Stack: [GEC, first] → "first end"
6. first()  tugadi      → Stack: [GEC]
```

**Tushuntirish:** Call stack (EC Stack) LIFO — eng oxirgi qo'shilgan birinchi tugaydi. `third()` birinchi tugaydi, keyin `second()`, keyin `first()`. Har bir funksiya o'zidan keyingi kodni faqat chaqirgan funksiya qaytgandan keyin davom ettiradi.
</details>

---

## Xulosa

Bu bo'limda Execution Context'ning ichki tuzilmasini o'rgandik:

- **Execution Context** — JavaScript engine har bir kod bo'lagini bajarish uchun yaratiladigan muhit. Uch turi bor: Global, Function, Eval
- **Creation Phase** — kod bajarilmasdan oldin muhit tayyorlanadi: `var` → `undefined`, `let`/`const` → TDZ, function declaration → to'liq function
- **Execution Phase** — kod qator-baqatar bajariladi, o'zgaruvchilarga qiymat beriladi
- **Variable Environment** — `var` declaration'lar saqlanadi (function-scoped)
- **Lexical Environment** — `let`, `const`, `function`, `class` saqlanadi (block-scoped)
- **Environment Record** — o'zgaruvchilar va binding'lar saqlanadigan tuzilma (Declarative va Object turlari)
- **Scope Chain** — outer environment reference orqali quriladi, o'zgaruvchi ichidan tashqariga qidiriladi
- **EC Stack** — Call Stack bilan bir xil — execution context'larni LIFO tartibida boshqaradi

Creation Phase'ni tushunish hoisting'ni tushunishning kaliti — keyingi bo'limda aynan hoisting mexanizmini chuqur o'rganamiz.

---

**Keyingi bo'lim:** [03-hoisting.md](03-hoisting.md) — Hoisting ichki mexanizmi, var/let/const hoisting farqlari, Temporal Dead Zone (TDZ), function vs variable hoisting priority.
