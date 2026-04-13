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
- [Dynamic Code Evaluation](#dynamic-code-evaluation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Execution Context Nima

### Nazariya

Execution Context (EC) — JavaScript engine har bir kod bo'lagini bajarishdan oldin yaratiladigan ichki environment. Shu environment ichida engine quyidagilarni aniqlaydi:

1. **Qaysi o'zgaruvchilar va funksiyalar mavjud** — o'zgaruvchilar e'lon qilinadi, funksiya declaration'lar to'liq saqlanadi
2. **Scope chain** — joriy scope'dan tashqi scope'larga yo'l (o'zgaruvchi qidiruv zanjiri)
3. **`this` qiymati** — joriy kontekstda `this` nimaga teng

JavaScript da har qanday kod faqat execution context ichida bajariladi. Engine uch xil EC yaratadi: **Global Execution Context (GEC)** — dastur boshlanganda bir marta yaratiladi, **Function Execution Context (FEC)** — har funksiya chaqirilganda yangi yaratiladi, **Eval Execution Context** — `eval()` chaqirilganda (amaliyotda ishlatmaslik tavsiya qilinadi). Har birining tafsilotlari quyidagi uch alohida bo'limda yoritiladi.

Execution context'ni tushunish nima uchun kerak: hoisting, scope chain, closure, `this` keyword — bularning barchasi execution context mexanizmiga bog'liq. Bu tushunchalarni chuqur anglash uchun avval EC'ning ichki tuzilmasini bilish zarur.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spetsifikatsiyasiga ko'ra, har bir execution context quyidagi komponentlarga ega:

```
┌──────────────────────────────────────┐
│        Execution Context             │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  LexicalEnvironment            │  │
│  │  • let, const, class binding'lari │
│  │  • OuterEnv → parent scope     │  │
│  │  (joriy scope, scope chain'ning │  │
│  │   asosiy entry point'i)        │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  VariableEnvironment           │  │
│  │  • var, function declaration'lar │
│  │  (function-level scope)        │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  ThisBinding                   │  │
│  │  • joriy this qiymati          │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

- **LexicalEnvironment** — joriy lexical scope. `let`, `const`, `class` binding'lari va parent scope'ga havola (`OuterEnv` / Outer Environment Reference) shu yerda. Scope chain qidiruvi shu havola orqali ishlaydi.
- **VariableEnvironment** — function-level scope. `var` va function declaration'lar saqlanadi. Funksiya boshida bu LexicalEnvironment bilan bir xil scope record'ga ishora qiladi.
- **ThisBinding** — joriy kontekstda `this` nimaga teng ekanligi.

> **Spec eslatmasi:** Modern ECMAScript spec'da `LexicalEnvironment` va `VariableEnvironment` — Execution Context'ning **slot**'lari, ular Environment Record object'lariga ishora qiladi. Outer reference esa Environment Record'ning `[[OuterEnv]]` slot'ida joylashadi. Yuqoridagi diagramma soddalashtirilgan model — pedagogik aniqlik uchun OuterEnv LexicalEnvironment qatorida ko'rsatilgan, chunki scope chain qidiruvi shu yerdan boshlanadi.

**Ikki alohida environment slot** nima uchun kerak: funksiya boshlanganda `LexicalEnvironment` va `VariableEnvironment` bir xil scope record'ga ishora qiladi. Lekin `{ }` block ichiga kirilganda `let`/`const` uchun yangi Environment Record yaratiladi va `LexicalEnvironment` slot unga ishora qilishga o'tkaziladi — `VariableEnvironment` esa function-level'da qoladi. Bu mexanizm tufayli `var` function-scoped, `let`/`const` esa block-scoped bo'lib ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Har bir kod bo'lagi execution context ichida bajariladi
// Global EC — dastur boshlanganda yaratiladi
var globalVar = "global scope";
console.log(this);  // globalThis (brauzer: window, Node.js: global)

function demo() {
  // Function EC — har chaqiruvda yangi yaratiladi
  var localVar = "function scope";
  console.log(globalVar); // scope chain orqali global'dan oladi
  console.log(this);      // Default binding — globalThis (non-strict)
}

demo();

// Har chaqiruv yangi EC yaratadi — local o'zgaruvchilar ajralgan
function counter() {
  let count = 0;
  count++;
  return count;
}
console.log(counter()); // 1
console.log(counter()); // 1 — yangi EC, yangi count
```

</details>

---

## Execution Context Turlari

### Nazariya

JavaScript da uch xil execution context mavjud. Ularning har biri turli holatlarda yaratiladi va turli qoidalarga bo'ysunadi:

1. **Global Execution Context (GEC)** — dastur boshlanganda birinchi bo'lib yaratiladi. Faqat **bitta** GEC bo'ladi va u dastur tugaguncha yashaydi. Global scope'dagi barcha kod shu EC ichida bajariladi. Global object (brauzerda `window`, Node.js da `global`), global `this`, va global o'zgaruvchilar shu yerda aniqlanadi.

2. **Function Execution Context (FEC)** — har bir funksiya **chaqirilganda** yangi FEC yaratiladi. Funksiya necha marta chaqirilsa — shuncha yangi FEC yaratiladi. Har bir FEC o'zining local o'zgaruvchilari, parameter'lari, `arguments` object'i va `this` qiymatiga ega. Funksiya tugaganda (return yoki exception) FEC stack'dan olib tashlanadi.

3. **Eval Execution Context** — `eval()` funksiyasi chaqirilganda yaratiladi. `eval()` string argument'ini JavaScript kod sifatida bajaradi va shu kod uchun yangi EC yaratadi. Amaliyotda `eval()`'dan voz kechish tavsiya qilinadi — xavfsizlik muammolari va JIT optimization'ni buzishi sababli (batafsil `Dynamic Code Evaluation` bo'limida).

```
┌────────────────────────────────────────────┐
│  Dastur boshlanganda:                      │
│                                            │
│  1. Global Execution Context yaratiladi    │
│     (bitta, dastur davomida yashaydi)      │
│                                            │
│  2. Funksiya chaqirilganda:                │
│     → Yangi Function EC yaratiladi         │
│     → Stack'ning tepasiga qo'shiladi       │
│     → Funksiya tugaganda stack'dan chiqadi │
│                                            │
│  3. eval() chaqirilganda:                  │
│     → Yangi Eval EC yaratiladi             │
│     (tavsiya etilmaydi)                    │
└────────────────────────────────────────────┘
```

Har bir tur'ning ichki tuzilmasi, yaratilish jarayoni, scope chain va `this` binding mexanizmi keyingi uch bo'limda alohida batafsil yoritiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

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
// Oldingi chaqiruvning EC si allaqachon yo'q — local o'zgaruvchilar alohida

// Har chaqiruvda yangi FEC — har chaqiruv 1 qaytaradi
function counter() {
  let count = 0;
  count++;
  return count;
}
console.log(counter()); // 1
console.log(counter()); // 1 — yangi FEC, yangi count
console.log(counter()); // 1 — yangi FEC, yangi count

// 3. Eval EC — amaliyotda ISHLATMANG
// eval("var x = 10"); // xavfsizlik va performance muammolari
```

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

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

> **Eslatma:** GEC — global scope uchun **maxsus** tuzilma. Bu yerda LexicalEnvironment va VariableEnvironment slot'lari **bir xil** composite GlobalEnvironmentRecord'ga ishora qiladi (yuqoridagi diagrammada faqat bittasi ko'rsatilgan, sodda holatga ko'ra). Function context'larda esa LexicalEnvironment va VariableEnvironment ko'pincha alohida Environment Record'larga ishora qiladi — bu farq `Variable Environment vs Lexical Environment` bo'limida batafsil ko'rib chiqiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

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
│  │  ├── arguments: { 0: 100, 1: 0.2 }    │  │
│  │  ├── price: 100                        │  │
│  │  ├── taxRate: 0.2                      │  │
│  │  ├── let finalPrice: <uninitialized>   │  │
│  │  │                                     │  │
│  │  OuterRef: → Global Environment        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  VariableEnvironment:                        │
│  ┌────────────────────────────────────────┐  │
│  │  DeclarativeEnvironmentRecord:         │  │
│  │  └── var discount: undefined           │  │
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

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

Har bir execution context (GEC, FEC yoki Eval EC) **ikki bosqichda** ishlaydi: avval **Creation Phase** (muhit yaratiladi — kod hali bajarilmaydi), keyin **Execution Phase** (kod qator-baqatar bajariladi). Bu ajralish JavaScript'ning eng fundamental mexanizmlaridan biri va ko'plab xususiyatlarning sababi — eng muhimi **hoisting**.

**Nima uchun bu ikki bosqich muhim:** hoisting xulq-atvori shu mexanizmga asoslanadi. Misol uchun, `console.log(x); var x = 10;` qatoridagi kod **xato bermaydi** va `undefined` chiqaradi — chunki Creation Phase'da `var x` allaqachon `undefined` qiymati bilan ro'yxatga olingan. Execution Phase boshlanganda `console.log(x)` bajarilganda `x` mavjud (lekin hali `10` qiymati berilmagan). Lekin `let` bilan xuddi shu kod `ReferenceError` beradi — chunki `let` Creation Phase'da "uninitialized" holatda, undan o'qish TDZ xatosini chiqaradi.

```
┌─────────────────────────────────────────────┐
│  Execution Context Lifecycle                │
│                                             │
│  1. CREATION PHASE (kod bajarilMAYDI):      │
│     ├── LexicalEnvironment yaratiladi       │
│     ├── VariableEnvironment yaratiladi      │
│     ├── this binding aniqlanadi             │
│     ├── var → undefined bilan initialize    │
│     ├── let/const → uninitialized (TDZ)     │
│     ├── function declaration → to'liq func  │
│     └── Scope chain quriladi (outer ref)    │
│                                             │
│  2. EXECUTION PHASE (kod bajariladi):       │
│     ├── Qator-baqatar kod bajariladi        │
│     ├── var → haqiqiy qiymat beriladi       │
│     ├── let/const → qiymat beriladi         │
│     │   (TDZ tugaydi)                       │
│     ├── Funksiyalar chaqiriladi             │
│     └── Expression'lar evaluate qilinadi    │
└─────────────────────────────────────────────┘
```

Har bir bosqich quyidagi `Creation Phase` va `Execution Phase` bo'limlarida alohida va batafsil yoritiladi — Creation Phase'da engine aniq qaysi qadamlarni bajaradi, Execution Phase'da kod qanday tartibda o'zgaruvchilarga qiymat berib funksiyalarni chaqiradi.

---

## Creation Phase

### Nazariya

**Creation Phase** — execution context'ning birinchi bosqichi. Bu bosqichda kod **hali bajarilmaydi**, faqat muhit tayyorlanadi. Engine source code'ni scan qiladi va quyidagilarni bajaradi:

**1. LexicalEnvironment yaratiladi:**
- `let`, `const` declaration'lar topiladi va **uninitialized** holatda ro'yxatga olinadi (TDZ boshlanadi)
- `class` declaration'lar topiladi va **uninitialized** holatda ro'yxatga olinadi (TDZ)

**2. VariableEnvironment yaratiladi:**
- `var` declaration'lar topiladi va **`undefined`** bilan initialize qilinadi
- `function` declaration'lar topiladi va **to'liq funksiya** bilan initialize qilinadi

**3. this binding aniqlanadi:**
- Global context'da → `globalThis` (window/global)
- Function context'da → chaqirilish usulga qarab

**4. Scope chain quriladi:**
- Outer environment reference o'rnatiladi — tashqi scope'ga havola

<details>
<summary><strong>Under the Hood</strong></summary>

Creation Phase ni qadam-baqadam ko'rsatadigan misol:

```javascript
// ⚠️ Har birini ALOHIDA faylda tekshiring — ReferenceError script'ni to'xtatadi!

// Test 1: var — ishlaydi (undefined)
console.log(x);       // undefined
var x = 10;

// Test 2: function — ishlaydi (to'liq hoist)
console.log(greet);   // function greet() {...}
function greet() { return "Hello"; }

// Test 3: let — ReferenceError (TDZ)
// console.log(y);    // ❌ ReferenceError: Cannot access 'y' before initialization
// let y = 20;

// Test 4: class — ReferenceError (TDZ)
// console.log(MyClass); // ❌ ReferenceError: Cannot access 'MyClass' before initialization
// class MyClass {}
```

Creation Phase da engine nima qiladi:

```
CREATION PHASE (kod bajarilmaydi):

VariableEnvironment:
  x: undefined          ← var → undefined bilan initialize
  greet: function() {}  ← function declaration → to'liq function

LexicalEnvironment:
  y: <uninitialized>    ← let → TDZ (Temporal Dead Zone)
  MyClass: <uninitialized> ← class → TDZ

EXECUTION PHASE (kod bajariladi — yuqoridagi Test 1 misolida):
  console.log(x)       → undefined (var allaqachon undefined bilan initialize)
  x = 10               → x endi 10

  Agar let/const e'londan OLDIN o'qilsa:
  console.log(y)       → ❌ ReferenceError! (y hali TDZ da — script to'xtaydi)

  Agar function declaration e'londan OLDIN o'qilsa:
  console.log(greet)   → function greet() {...} (to'liq hoist)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Execution Phase

### Nazariya

Execution Phase — Creation Phase tugagandan keyin boshlanaladigan ikkinchi bosqich. Bu yerda JavaScript kodi **qator-baqatar** bajariladi:

1. O'zgaruvchilarga haqiqiy qiymatlar beriladi (`var x = 10` → x endi 10)
2. `let`/`const` ning TDZ tugaydi (qiymat berilganda)
3. Funksiyalar chaqiriladi (har bir chaqiruv yangi FEC yaratadi)
4. Expression'lar evaluate qilinadi
5. Conditional va loop'lar bajariladi

<details>
<summary><strong>Kod Misollari</strong></summary>

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
  getInfo: function getInfo() {...}
LexicalEnvironment:
  age: <uninitialized>
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

</details>

---

## Variable Environment vs Lexical Environment

### Nazariya

Har bir execution context'da ikki alohida **slot** mavjud: `LexicalEnvironment` va `VariableEnvironment`. Ikkisi ham Environment Record'ga pointer'lar — ba'zan bir xil Environment Record'ga ko'rsatadi, ba'zan turlicha. Farq **qachon va nima saqlanishida**.

**VariableEnvironment:**
- `var` va `function` declaration'lari saqlanadigan **function-level** Environment Record'ga ishora qiladi
- Funksiya davomida o'zgarmaydi — doim funksiyaning asosiy environment'iga ko'rsatadi
- Block'lar (if, for, while) ichida e'lon qilingan `var` ham shu yerda — shuning uchun `var` "function-scoped"
- Creation Phase'da `var` → `undefined`, `function` → to'liq funksiya bilan initialize qilinadi

**LexicalEnvironment:**
- `let`, `const`, `class` declaration'lari saqlanadigan **joriy** Environment Record'ga ishora qiladi
- **O'zgaruvchan slot**: funksiya boshida VariableEnvironment bilan bir xil Environment Record'ga ko'rsatadi, lekin har block entry'da yangi Environment Record yaratiladi va shu slot unga ko'rsatish uchun o'zgartiriladi. Block'dan chiqqanda — oldingi qiymatga qaytadi.
- Creation Phase'da `let`/`const` **uninitialized** (TDZ)

**Nima uchun ikki slot kerak** — shunchaki `var` va `let`/`const` scope qoidalari boshqacha bo'lgani uchun. `var` function-level, shuning uchun u doim VariableEnvironment'da yashaydi (block ichida ham). `let`/`const` block-level, shuning uchun ular joriy LexicalEnvironment'da yashaydi — va bu slot block entry'da yangilanadi. Shu ikki slot'li dizayn block scoping va TDZ mexanizmini aniq implement qilish imkonini beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

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

  // ❌ Quyidagi ikkala qator ham ReferenceError beradi
  // (amalda birinchisida script to'xtaydi — ikkalasini alohida test qiling)
  console.log(d); // ReferenceError — let block-scoped
  console.log(e); // ReferenceError — const block-scoped
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

</details>

---

## Environment Record

### Nazariya

Environment Record — har bir LexicalEnvironment ichidagi o'zgaruvchilar va binding'lar saqlanadigan tuzilma. Bu ECMAScript spec'dagi termin bo'lib, aslida "scope'dagi barcha o'zgaruvchilar ro'yxati" degani.

Environment Record'ning ikki asosiy turi bor — **Declarative** va **Object**. Shulardan foydalanib spec yana ikkita maxsus composite turni aniqlaydi: **Global** (Object + Declarative birikmasi) va **Function** (Declarative subtype'i, `arguments` va `this` binding'lari bilan). ES2015'dan `Module Environment Record` ham qo'shildi — u ham Declarative subtype'i.

**1. Declarative Environment Record:**
- `let`, `const`, `var`, `function`, `class`, `import`, `catch` parameter binding'lari saqlanadi
- Funksiya va block scope'larda ishlatiladi
- O'zgaruvchilar to'g'ridan-to'g'ri record ichida saqlanadi — tez (hash lookup kerak emas)

**2. Object Environment Record:**
- Binding'lar biror object'ning property'lari sifatida saqlanadi — record "binding object" orqali ishlaydi
- Global scope'da `var` va function declaration'lar uchun ishlatiladi (binding object — `window`/`globalThis`)
- `with` statement ham ushbu turdan foydalanadi (strict mode'da taqiqlangan)

**3. Global Environment Record (composite):**
- Alohida "tur" emas — bu Object ER + Declarative ER birikmasi
- `var`, function declaration → Object ER qismi (global object'ga qo'shiladi)
- `let`, `const`, `class` → Declarative ER qismi (global object'ga qo'shilMAYDI)
- Shuning uchun global `var` brauzerda `window`'ga chiqadi, lekin global `let` chiqmaydi

**4. Function Environment Record (Declarative subtype):**
- Funksiya chaqirilganda yaratiladi — parameter'lar, local `var`/`let`/`const`, `arguments` (arrow function'da emas)
- Qo'shimcha: `[[ThisValue]]` va `[[NewTarget]]` slotlari (arrow function'da `this` lexical — bu slot'lar yo'q)

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

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

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Execution Context Stack

### Nazariya

Execution Context Stack — barcha execution context'larni LIFO (Last In, First Out) tartibida saqlaydigan stack. **Bu Call Stack bilan bir xil narsa** — terminologiya farqli, mexanizm bir xil. JavaScript single-threaded — bir vaqtda faqat bitta EC bajariladi (stack'ning eng tepasidagi).

Stack ichidagi har bir "frame" — bu to'liq **Execution Context**: o'z LexicalEnvironment, VariableEnvironment, ThisBinding va OuterRef bilan. GEC stack'ning tubida joylashadi (dastur boshlanganda qo'shiladi, tugaguncha qoladi). Har funksiya chaqirilganda engine yangi FEC yaratadi va uni stack'ning tepasiga qo'shadi. Funksiya `return` qilganda (yoki exception throw qilganda) shu FEC stack'dan olib tashlanadi, joriy ijro boshqaruvi oldingi EC'ga qaytadi.

Ishlash tartibi:
1. Dastur boshlanganda → GEC yaratiladi va stack'ga qo'shiladi
2. Funksiya chaqirilganda → yangi FEC yaratiladi va stack'ning tepasiga qo'shiladi
3. Funksiya tugaganda → uning FEC'i stack'dan olib tashlanadi
4. Boshqaruv oldingi EC'ga qaytadi (stack'ning yangi tepasiga)
5. Dastur tugaganda → GEC ham stack'dan chiqadi

Har EC mustaqil — o'z local o'zgaruvchilari bilan. Lekin `OuterRef` orqali lexical parent EC'ning environment'iga havola saqlaydi — shu havola scope chain'ni quradi (`Outer Environment Reference va Scope Chain` bo'limiga qarang).

> **Eslatma:** Call Stack'ning umumiy mexanizmi (LIFO, stack overflow, frame allocation) [01-js-engine.md](01-js-engine.md)'da yoritilgan. Bu bo'lim aynan shu stack'ning Execution Context bilan munosabatini va scope chain quruvchilik mexanizmini ko'rsatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Quyidagi misol orqali EC Stack'ning to'liq ishlashini ko'rsatamiz:

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
Boshlang'ich (dastur boshlanganda):
┌───────────┐
│  GEC      │ ← x: undefined → 10 (Creation Phase tugagach)
└───────────┘
ThisBinding: window
OuterRef: null

first() chaqirildi:
┌───────────┐
│  first()  │ ← Yangi FEC yaratildi
│  GEC      │    a: undefined → 20
└───────────┘    OuterRef: → GEC

second() chaqirildi (first ichidan):
┌───────────┐
│  second() │ ← Yangi FEC
│  first()  │    b: undefined → 30
│  GEC      │    OuterRef: → GEC (lexical parent — global'da yozilgan)
└───────────┘

third() chaqirildi (second ichidan):
┌───────────┐
│  third()  │ ← Yangi FEC
│  second() │    c: undefined → 40
│  first()  │    OuterRef: → GEC (lexical parent)
│  GEC      │
└───────────┘

third() console.log(40) → tugadi (return):
┌───────────┐
│  second() │ ← Stack'ning yangi tepasi
│  first()  │    boshqaruv second() ga qaytadi
│  GEC      │    third FEC tashlandi
└───────────┘

second() tugadi (return):
┌───────────┐
│  first()  │ ← Stack'ning yangi tepasi
│  GEC      │    boshqaruv first() ga qaytadi
└───────────┘

first() console.log(20) → tugadi (return):
┌───────────┐
│  GEC      │ ← Faqat GEC qoldi
└───────────┘    boshqaruv global ga qaytadi

console.log(10) → dastur tugadi:
┌───────────┐
│  (bo'sh)  │ ← GEC ham chiqdi
└───────────┘

Output: 40, 20, 10
```

Diqqat: `OuterRef` har funksiya uchun **lexical parent**'ga ko'rsatadi — ya'ni funksiya **yozilgan** scope'ga, **chaqirilgan** scope'ga emas. Yuqoridagi misolda `second()` `first()` ichidan chaqirildi, lekin `second()`'ning OuterRef'i GEC'ga ko'rsatadi (chunki `second()` global scope'da yozilgan), `first()` ga emas. Shu fundamental qoida lexical scoping deb ataladi (batafsil 04-scope.md va 05-closures.md da).

</details>

---

## Dynamic Code Evaluation

### Nazariya

JavaScript da string'ni kod sifatida bajarish imkonini beruvchi ikki asosiy mexanizm bor: `eval()` funksiyasi va `Function` constructor. Ikkala mexanizm ham **yangi execution context** yaratadi, lekin ularning scope bilan ishlashi tubdan farq qiladi.

**`eval()` nima?** — String argument qabul qilib, uni JavaScript kod sifatida **bajaradi**:

```javascript
eval("console.log('Salom')"); // "Salom" — string ni kod sifatida bajardi
eval("2 + 3"); // 5 — expression natijasini qaytaradi
```

**eval() xavflari** — nima uchun ISHLATMASLIK kerak:

1. **XSS (Cross-Site Scripting)** — foydalanuvchi kiritgan ma'lumotni eval'ga berish = **xavfsizlik teshigi**. Hacker ixtiyoriy kod bajara oladi
2. **Performance** — JIT compiler eval ichidagi kodni **optimize qila olmaydi**, chunki string runtime da keladi va compile-time da noma'lum
3. **Scope leaking** — eval local scope'dagi o'zgaruvchilarni o'zgartirishi mumkin — bu kutilmagan xatolarga olib keladi
4. **Debugging qiyinligi** — eval ichidagi xatolarni topish juda qiyin — stack trace'lar noaniq

**Direct eval vs Indirect eval** — bu execution context jihatidan muhim farq:

```
Direct eval:                        Indirect eval:
eval("code")                        (0, eval)("code")
                                    const e = eval; e("code")
                                    window.eval("code")

┌──────────────────────┐            ┌──────────────────────┐
│ Joriy scope'ga       │            │ GLOBAL scope'da      │
│ kirish huquqi BOR    │            │ ishlaydi             │
│                      │            │                      │
│ Local variable'larga │            │ Local variable'larga │
│ access BOR           │            │ access YO'Q          │
│                      │            │                      │
│ O'zgaruvchi yarata   │            │ Faqat global         │
│ oladi local scope'da │            │ scope'da ishlaydi    │
└──────────────────────┘            └──────────────────────┘
```

**`Function` constructor** — `new Function('param1', 'param2', 'body')`:

- eval() dan farqi: **faqat global scope'ga** access bor, local scope'ga **emas**
- Bu uni eval() dan **xavfsizroq** qiladi — local o'zgaruvchilarni buza olmaydi
- Use case: dynamic function yaratish (template engine'lar, formula evaluator'lar)

**Strict mode da eval() cheklangan:**
- `"use strict"` rejimida eval **o'z scope'ida** ishlaydi — tashqi scope'ga o'zgaruvchi qo'sha olmaydi
- Bu eval'ning eng xavfli xususiyatlaridan birini bloklaydi

**Xavfsiz alternativalar:**
- `JSON.parse()` — JSON string'ni parse qilish uchun (eval o'rniga)
- `Map` / Object lookup — dinamik property access uchun
- Template literals — dinamik string yaratish uchun
- `new URL()`, `new RegExp()` — maxsus parsing uchun

<details>
<summary><strong>Kod Misollari</strong></summary>

**eval() xavfi — nima uchun ISHLATMASLIK kerak:**

```javascript
// ❌ XAVFLI — hech qachon foydalanuvchi kiritgan ma'lumotni eval'ga bermang!
function unsafeCalculate(userInput) {
  return eval(userInput);
  // Hacker "document.cookie" yoki boshqa xavfli kod yuborishi mumkin
}

// Foydalanuvchi "2 + 3" yubordimi? → 5
// Hacker xavfli kod yubordimi? → Cookie o'g'irlandi!

// ✅ XAVFSIZ alternativa — ruxsat etilgan operatsiyalar bilan
function safeCalculate(a, operator, b) {
  const ops = {
    "+": (x, y) => x + y,
    "-": (x, y) => x - y,
    "*": (x, y) => x * y,
    "/": (x, y) => y !== 0 ? x / y : NaN
  };
  if (!(operator in ops)) throw new Error("Noto'g'ri operator");
  return ops[operator](a, b);
}

safeCalculate(2, "+", 3); // 5 — xavfsiz!
```

**Direct eval vs Indirect eval — scope farqi:**

```javascript
function testEval() {
  const secret = "mahfiy ma'lumot";

  // Direct eval — local scope'ga kiradi
  const directResult = eval("secret"); // ✅ "mahfiy ma'lumot" — local'ga access bor!

  // Indirect eval — global scope'da ishlaydi
  const indirectEval = eval;
  // const indirectResult = indirectEval("secret");
  // ❌ ReferenceError: secret is not defined — global scope'da "secret" yo'q

  // Indirect eval bilan global o'zgaruvchiga murojaat
  const indirectResult = indirectEval("typeof globalThis"); // ✅ "object"

  console.log(directResult);   // "mahfiy ma'lumot"
  console.log(indirectResult); // "object"
}

testEval();

// Direct eval scope'ga ta'sir qilishi:
function dangerousEval() {
  eval("var leaked = 'Oops!'"); // ❌ local scope'ga o'zgaruvchi qo'shdi!
  console.log(leaked); // "Oops!" — eval yaratgan o'zgaruvchi scope'da qoldi
}
```

**Function constructor — eval'dan xavfsizroq variant:**

```javascript
// Function constructor — faqat global scope'ga access
function testFunctionConstructor() {
  const localVar = "local";

  // ❌ eval — local scope'ga kiradi
  console.log(eval("localVar")); // "local"

  // ✅ Function constructor — local scope'ga kira OLMAYDI
  const fn = new Function("return typeof localVar");
  console.log(fn()); // "undefined" — localVar ko'rinmaydi!

  // Function constructor bilan dinamik funksiya yaratish
  const add = new Function("a", "b", "return a + b");
  console.log(add(2, 3)); // 5

  // Amaliy misol — formula evaluator
  function createFormula(expression, ...params) {
    return new Function(...params, "return " + expression);
  }

  const circleArea = createFormula("Math.PI * r * r", "r");
  console.log(circleArea(5)); // 78.53981633974483

  const hypotenuse = createFormula("Math.sqrt(a*a + b*b)", "a", "b");
  console.log(hypotenuse(3, 4)); // 5
}

testFunctionConstructor();
```

**JSON.parse — eval'ning xavfsiz alternativasi:**

```javascript
// ❌ Eski usul — eval bilan JSON parse (HECH QACHON bunday qilmang!)
const jsonString = '{"name": "Ali", "age": 25}';
// const data = eval("(" + jsonString + ")"); // ← XSS xavfi!

// ✅ To'g'ri usul — JSON.parse
const data = JSON.parse(jsonString); // { name: "Ali", age: 25 }

// JSON.parse noto'g'ri formatda xato beradi — xavfsiz
try {
  JSON.parse("bu JSON emas");
} catch (e) {
  console.log(e instanceof SyntaxError); // true — JSON.parse SyntaxError throw qiladi
  console.log(e.message); // Engine-specific xato matni — "Unexpected token..." shaklida
}

// JSON.parse reviver bilan — ma'lumotni transform qilish
const dateJSON = '{"created": "2025-01-15T10:30:00.000Z"}';
const withDate = JSON.parse(dateJSON, (key, value) => {
  if (key === "created") return new Date(value);
  return value;
});
console.log(withDate.created instanceof Date); // true
```

**Strict mode da eval cheklangan:**

```javascript
"use strict";

// Strict mode'da eval O'Z scope'ida ishlaydi
eval("var x = 10;");
// console.log(x); // ❌ ReferenceError — x faqat eval ichida yashadi

// Non-strict mode'da esa:
// eval("var x = 10;");
// console.log(x); // 10 — eval tashqi scope'ga x ni qo'shgan bo'lardi

// Strict mode eval — xavfsizroq, lekin baribir ISHLATMANG
// Alternativalar har doim yaxshiroq
```

</details>

---

## Edge Cases va Gotchas

### Module scope'da `this` — `undefined`, global emas

ES Module'larda (`<script type="module">` yoki `.mjs` fayl) top-level `this` — `undefined`. Script mode'dagidan farqli, module scope o'z execution context'iga ega va default `this` binding global object'ga emas, `undefined` ga o'rnatiladi.

```javascript
// Script mode (<script>):
console.log(this); // window (brauzer) yoki globalThis

// Module mode (<script type="module">):
console.log(this); // undefined

// Module ichidagi funksiya (modulelar har doim strict mode)
function regularFunc() {
  console.log(this); // undefined — strict mode default binding
}
regularFunc();
```

**Yechim:** Module'larda global reference kerak bo'lsa, `globalThis` ishlating — bu cross-environment global object'ga standart kirish usuli.

---

### Class declaration TDZ — function declaration'dan farqli

Function declaration'lar Creation Phase'da to'liq hoist bo'ladi, shuning uchun ularni e'londan oldin chaqirish mumkin. Class declaration'lar esa `let`/`const` kabi TDZ (Temporal Dead Zone)'da qoladi — e'londan oldin ishlatish ReferenceError beradi.

```javascript
// Function declaration — to'liq hoist
greet(); // ✅ "Hello" — ishlaydi
function greet() { return "Hello"; }

// Class declaration — TDZ'da
// new User("Ali"); // ❌ ReferenceError: Cannot access 'User' before initialization
class User {
  constructor(name) { this.name = name; }
}
const user = new User("Ali"); // ✅ e'londan keyin ishlaydi
```

**Yechim:** Class'larni doim ishlatishdan oldin deklaratsiya qiling. Modullarda import statement'lar har doim top'ga qo'yiladi — bu TDZ muammolarini oldini oladi.

---

### Block'da function declaration — Annex B legacy behavior

Non-strict mode'da `if`/`for`/`while` block ichidagi function declaration engine'larga qarab har xil ishlaydi. Strict mode'da esa function declaration faqat block ichida qoladi (lexical scope) — bu ECMAScript 2015'dan standartlashtirilgan.

```javascript
// Non-strict mode — engine'ga qarab farqli natija (Annex B legacy)
if (true) {
  function sneaky() { return 1; }
}
sneaky(); // Ba'zi engine'larda ishlaydi, ba'zilarida ReferenceError

// Strict mode — lexical (block-scoped), aniq behavior
"use strict";
if (true) {
  function scoped() { return 2; }
  console.log(scoped()); // 2
}
// console.log(scoped()); // ReferenceError — block tashqarisida yo'q
```

**Yechim:** Block ichida funksiya kerak bo'lsa `const fn = function() {}` yoki `const fn = () => {}` ishlating — bu cross-engine predictable behavior beradi.

---

### Method reference'ni `setTimeout` ga berish — `this` yo'qoladi

Method'ni callback sifatida berish — ayniqsa `setTimeout`, event listener yoki array method'lari uchun — method'ni o'z object'idan ajratadi. Natijada method chaqirilganda yangi execution context yaratiladi va `this` default binding oladi (global yoki undefined).

```javascript
const user = {
  name: "Ali",
  greet() {
    console.log("Hi, " + this.name);
  }
};

user.greet();               // "Hi, Ali" ✅ method call — this = user

setTimeout(user.greet, 100);
// Output: "Hi, undefined" ❌
// setTimeout ichida bog'liq bo'lmagan function chaqiradi
// this endi user emas — global/undefined

// ✅ Yechim 1: arrow function wrapper
setTimeout(() => user.greet(), 100); // "Hi, Ali"

// ✅ Yechim 2: bind
setTimeout(user.greet.bind(user), 100); // "Hi, Ali"
```

**Yechim:** Method'ni callback sifatida berayotganda doim `bind` yoki arrow wrapper ishlating. Class'larda constructor ichida `this.handleClick = this.handleClick.bind(this)` pattern'ini qo'llang.

---

### `var` try/catch blokda — function scope'ga oqib chiqadi

`try`/`catch` blokda e'lon qilingan `var` — catch parameter bundan mustasno — butun function scope'ga tegishli. Bu tasodifiy scope leak manbai bo'ladi.

```javascript
function processData() {
  try {
    var result = riskyOperation(); // function-scoped!
  } catch (error) {
    // error — catch parametri, faqat catch block'da
    console.log(error.message);
  }
  console.log(result); // ✅ Accessible — var function-scoped
  console.log(typeof error); // "undefined" — error faqat catch'da
}

function riskyOperation() {
  return "data";
}
```

**Yechim:** `try` dan oldin `let` bilan declare qiling: `let result; try { result = ... } catch {}`. Bu scope leak'ni oldini oladi va null-check imkonini beradi.

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
  d: function d() {}   ← function declaration → to'liq function
  e: undefined         ← var → undefined (function expression — faqat var hoist)

LexicalEnvironment:
  b: <uninitialized>   ← let → TDZ
  c: <uninitialized>   ← const → TDZ
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
ReferenceError: b is not defined
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
- **Variable Environment** — `var` va `function` declaration'lar saqlanadi (function-scoped)
- **Lexical Environment** — `let`, `const`, `class` saqlanadi (block-scoped)
- **Environment Record** — o'zgaruvchilar va binding'lar saqlanadigan tuzilma (Declarative va Object turlari)
- **Scope Chain** — outer environment reference orqali quriladi, o'zgaruvchi ichidan tashqariga qidiriladi
- **EC Stack** — Call Stack bilan bir xil — execution context'larni LIFO tartibida boshqaradi

Creation Phase'ni tushunish hoisting'ni tushunishning kaliti — keyingi bo'limda aynan hoisting mexanizmini chuqur o'rganamiz.

---

**Keyingi bo'lim:** [03-hoisting.md](03-hoisting.md) — Hoisting ichki mexanizmi, var/let/const hoisting farqlari, Temporal Dead Zone (TDZ), function vs variable hoisting priority.
