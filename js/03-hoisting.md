# Bo'lim 3: Hoisting — Ichki Mexanizm

> Hoisting — bu o'zgaruvchi va funksiyalarning "ko'tarilishi" emas, balki Creation Phase da xotiraga yozilishi.

---

## Mundarija

- [Hoisting Nima?](#hoisting-nima)
- [Aslida Nima Sodir Bo'layotgani](#aslida-nima-sodir-bolayotgani)
- [var Hoisting](#var-hoisting)
- [let va const Hoisting](#let-va-const-hoisting)
- [Temporal Dead Zone (TDZ)](#temporal-dead-zone-tdz)
- [Function Declaration Hoisting](#function-declaration-hoisting)
- [Function Expression Hoisting](#function-expression-hoisting)
- [Hoisting Priority](#hoisting-priority)
- [var vs let vs const — To'liq Taqqoslash](#var-vs-let-vs-const--toliq-taqqoslash)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Hoisting Nima?

### Nazariya

Hoisting — bu JavaScript'ning o'ziga xos xususiyati bo'lib, o'zgaruvchi va funksiya e'lonlarini kodda yozilgan joyidan **oldin** ishlatish imkoniyatini beradi. Bu xususiyat boshqa ko'plab dasturlash tillarida (Python, Java, C++) mavjud emas va shuning uchun JavaScript'ga o'tgan dasturchilarni ko'pincha chalkashtirib yuboradi.

Ko'pchilik "hoisting = declaration yuqoriga ko'tariladi" deb tushunadi. Bu **oddiy mental model** — boshlang'ich darajada tushunish uchun foydali, lekin texnik jihatdan **mutlaqo noto'g'ri**. Hech qanday kod jismonan ko'tarilmaydi yoki qayta joylashtirilmaydi. Aslida sodir bo'layotgan narsa — bu [02-execution-context.md](02-execution-context.md) da o'rgangan **Creation Phase** mexanizmimiz. Engine kodni bajarishdan **oldin** avval butun scope'ni skanerlab, barcha e'lonlarni topadi va ularni xotiraga yozadi. Shu sababli e'londan oldingi satrlarda ham o'zgaruvchi yoki funksiyaga murojaat qilish mumkin.

Nima uchun bu muhim? Real-world kodda hoisting'ni tushunmaslik kutilmagan xatolarga olib keladi. Masalan, `var` bilan e'lon qilingan o'zgaruvchi e'londan oldin `undefined` qaytaradi (xato bermaydi!), `let`/`const` esa `ReferenceError` tashlab dasturni to'xtatadi. Function declaration'lar to'liq hoist bo'ladi va e'londan oldin chaqirish mumkin, lekin function expression'lar faqat o'zgaruvchi sifatida hoist bo'ladi. Bu farqlarni bilmaslik React, Express va boshqa framework'larda nozik bug'larga sabab bo'lishi mumkin.

```javascript
console.log(name); // undefined — hali e'lon qilinmagan ko'rinadi, lekin xato bermaydi!
var name = "Islom";
```

Ko'pchilik buni shunday tasavvur qiladi (lekin bu noto'g'ri!):

```javascript
// ❌ "Go'yoki kod shunday qayta yoziladi" — bu NOTO'G'RI model
var name;              // "yuqoriga ko'tarildi"
console.log(name);     // undefined
name = "Islom";
```

Aslida hech narsa **ko'tarilmaydi**. Hech qanday kod qayta yozilmaydi.

---

## Aslida Nima Sodir Bo'layotgani

### Under the Hood

Hoisting — bu **Creation Phase** ning natijasi. Oldingi bo'limda ([02-execution-context.md](02-execution-context.md)) o'rganganimizdek, har bir Execution Context yaratilganda **ikki bosqich** bor:

1. **Creation Phase** — engine kodni **o'qiydi** (bajarMAYDI!) va barcha declaration'larni topib, xotiraga yozadi
2. **Execution Phase** — kod satr-bosatr bajariladi

```javascript
// Biz yozgan kod:
console.log(x);
var x = 5;

// Engine ichida nima sodir bo'ladi:

// ═══ CREATION PHASE ═══
// Engine butun kodni skanerlaydi va declaration'larni topadi:
// "var x topildi → VariableEnvironment ga x: undefined yozaman"

// ═══ EXECUTION PHASE ═══
// Satr 1: console.log(x) → x ni qidiraman → VariableEnvironment da x: undefined bor → undefined
// Satr 2: x = 5 → VariableEnvironment dagi x ni 5 ga o'zgartiraman
```

```
┌──────────────────────────────────────────────────┐
│  "HOISTING" aslida:                              │
│                                                  │
│  Kod SILJIMAYDI. Ko'tarilMAYDI.                  │
│  Engine Creation Phase da OLDINDAN xotira         │
│  ajratadi — shuning uchun kod bajarilganda        │
│  o'zgaruvchi allaqachon "mavjud" bo'ladi.        │
│                                                  │
│  var   → xotiraga yoziladi + undefined bilan init│
│  let   → xotiraga yoziladi + init QILINMAYDI     │
│  const → xotiraga yoziladi + init QILINMAYDI     │
│  function declaration → xotiraga TO'LIQ yoziladi │
│  function expression → faqat variable hoist      │
└──────────────────────────────────────────────────┘
```

---

## var Hoisting

### Nazariya

`var` — JavaScript'ning eng qadimgi o'zgaruvchi e'lon qilish usuli bo'lib, uning hoisting xulq-atvori boshqa e'lon turlaridan tubdan farq qiladi. `var` bilan e'lon qilingan o'zgaruvchi Creation Phase da xotiraga yoziladi **va** darhol `undefined` qiymati bilan **initialize** qilinadi. Shuning uchun e'londan oldingi satrlarda ham `var` o'zgaruvchisiga murojaat qilsangiz, xato emas, `undefined` qaytaradi.

Bu xulq-atvor ko'pincha nozik va topish qiyin bo'lgan bug'larga sabab bo'ladi. Tasavvur qiling, katta loyihada 200-verstalik faylda biror joyda `var total` e'lon qilingan, lekin siz kodni o'qiyotib yuqori qismda `total` ni ishlatmoqchisiz — `undefined` qaytaradi va hech qanday xato bermaydi. Bu **jim xato** (silent bug) — dastur ishlab turadi, lekin noto'g'ri natija beradi. Aynan shu muammo `let`/`const` ni yaratishga sabab bo'lgan asosiy motivatsiyalardan biri edi.

```javascript
console.log(a); // undefined — mavjud, lekin hali qiymat berilmagan
var a = 10;
console.log(a); // 10 — endi qiymat bor

console.log(b); // undefined
var b;          // faqat declaration, qiymatsiz
console.log(b); // undefined — qiymat berilmagan
```

### var — Faqat Declaration Hoist Bo'ladi, Assignment Emas

```javascript
// Bu:
console.log(x); // undefined
var x = 5;

// Engine uchun XUDDI SHU:
// Creation Phase: x → undefined
// Execution Phase:
//   console.log(x) → undefined
//   x = 5
```

Bu muhim farq: **declaration** (`var x`) hoist bo'ladi, lekin **assignment** (`= 5`) hoist **bo'lmaydi**.

### var — Function Scope

`var` block scope ni **tanimaydi** — faqat function scope:

```javascript
function test() {
  console.log(x); // undefined — hoist bo'ldi

  if (true) {
    var x = 10; // var — function scope, if block emas!
  }

  console.log(x); // 10 — if dan chiqdi
}
test();
```

```javascript
// for loop da ham:
for (var i = 0; i < 3; i++) {
  // ...
}
console.log(i); // 3 — var function/global scope, for block emas
```

---

## let va const Hoisting

### Nazariya

`let` va `const` hoisting mavzusida eng keng tarqalgan noto'g'ri tushuncha: **"let va const hoist bo'lmaydi"**. Bu **noto'g'ri**. `let` va `const` ham xuddi `var` kabi **hoist bo'ladi** — ya'ni Creation Phase da engine ularning mavjudligini biladi va xotiraga yozadi. Lekin muhim farq shundaki — ular `undefined` bilan **initialize qilinmaydi**. Ular e'lon qilingan satrgacha **TDZ (Temporal Dead Zone)** holatida turadi va bu vaqt ichida ularga murojaat qilish `ReferenceError` tashlanishiga sabab bo'ladi.

Bu farq sun'iy emas — u dizayn bo'yicha ataylab qilingan qaror. `var` ning `undefined` bilan initialize bo'lishi ko'p yillar davomida **jim xatolar**ga (silent bugs) sabab bo'ldi. Dasturchi o'zgaruvchini e'lon qilishni unutardi yoki noto'g'ri joyda ishlatardi, lekin JavaScript xato bermasdi — `undefined` qaytarib dasturchini chalg'itardi. `let`/`const` ning TDZ mexanizmi aynan shu muammoni hal qiladi: agar o'zgaruvchini e'londan oldin ishlatsangiz, JavaScript darhol **xato beradi** va siz muammoni deployment'ga chiqishidan oldin topasiz.

Quyidagi misol `let` ning hoist bo'lishini **isbotlaydi**:

```javascript
// ❌ Agar let hoist bo'lmaganida, bu global x ni ko'rishi kerak edi:
let x = "global";

function test() {
  console.log(x); // ❌ ReferenceError — TDZ!
  let x = "local";
}
test();
```

Bu isbot: agar `let x = "local"` hoist bo'lmaganida, `console.log(x)` global `x = "global"` ni ko'rishi kerak edi. Lekin ReferenceError berdi — demak engine bu scope da `x` **borligini biladi**, lekin unga **kirishga ruxsat bermayapti** (TDZ).

### Creation Phase da:

```
var   → xotiraga yoziladi + undefined bilan INITIALIZE qilinadi
        ↳ Shuning uchun kirish mumkin (undefined qaytaradi)

let   → xotiraga yoziladi + INITIALIZE QILINMAYDI (TDZ)
        ↳ Shuning uchun kirish TAQIQLANGAN (ReferenceError)

const → xotiraga yoziladi + INITIALIZE QILINMAYDI (TDZ)
        ↳ Shuning uchun kirish TAQIQLANGAN (ReferenceError)
```

---

## Temporal Dead Zone (TDZ)

### Nazariya

TDZ — **Temporal Dead Zone** ("vaqtinchalik o'lik zona") — bu `let` va `const` o'zgaruvchining scope'ga kirgan nuqtasidan (hoist bo'lgan joydan) to e'lon qilingan satrgacha bo'lgan **xavfli hudud**. Bu hudud ichida o'zgaruvchi xotiraga yozilgan (engine uning mavjudligini biladi), lekin hali initialize qilinmagan — shuning uchun har qanday murojaat `ReferenceError` bilan tugaydi.

"Temporal" so'zi bu yerda kalit: TDZ **vaqtga** asoslangan, **koddagi pozitsiyaga** emas. Bu nozik, lekin juda muhim farq. Funksiya ichida `let x` dan oldin `x` ga murojaat qilgan bo'lsangiz ham, agar bu funksiya `let x` dan **keyin** chaqirilsa — xato bo'lmaydi, chunki chaqirilish vaqtida `x` allaqachon initialize bo'lgan. Aksincha, funksiya `let x` dan oldin chaqirilsa — TDZ ishlaydi va `ReferenceError` tashlanadi. Shu sababli "temporal" — ya'ni vaqtga bog'liq.

TDZ ni tushunish amaliy jihatdan muhim, chunki zamonaviy JavaScript kodda `let`/`const` standart hisoblanadi va TDZ xatolari production'da eng ko'p uchraydigan runtime xatolardan biri. TypeScript ham TDZ ni tekshiradi va compile vaqtida ogohlantiradi.

### Vizualizatsiya

```javascript
// ──────────── TDZ boshlanadi (scope boshlanishi) ────────────
//  |
//  |  console.log(x); // ❌ ReferenceError — TDZ ichida!
//  |  console.log(x); // ❌ ReferenceError — hali TDZ ichida!
//  |
//  ↓
let x = 10;  // ← TDZ TUGAYDI — x initialize bo'ldi
//
console.log(x); // ✅ 10 — TDZ dan chiqdik
```

```
┌─────────────────────────────┐
│  Block/Function boshlanishi │ ← TDZ BOSHLANADI
│          ⚠️ DANGER ZONE     │
│  let x;  hoist bo'ldi       │
│  lekin initialize YO'Q      │
│  x ga kirish = ReferenceError│
│          ...                │
│          ...                │
│  let x = 10;               │ ← TDZ TUGAYDI
│          ✅ SAFE ZONE       │
│  x ga kirish = 10           │
│          ...                │
└─────────────────────────────┘
```

### TDZ — Vaqtga Asoslangan, Pozitsiyaga Emas

TDZ **temporal** (vaqtga bog'liq), pozitsiyaga emas:

```javascript
// Bu ishlaydi! Funksiya CHAQIRILGANDA x allaqachon initialize bo'lgan
function useLater() {
  console.log(x); // ✅ 10 — chaqirilgan vaqtda x tayyor
}

let x = 10;  // x initialize bo'ldi
useLater();  // funksiya ENDI chaqirildi — x allaqachon bor
```

```javascript
// Bu ishlamaydi! Funksiya CHAQIRILGAN vaqtda x hali TDZ da
function useEarly() {
  console.log(x); // ❌ ReferenceError
}

useEarly(); // funksiya chaqirildi — lekin x hali TDZ da!
let x = 10;
```

### const va TDZ

`const` ham xuddi `let` kabi TDZ ga ega. Lekin `const` da qo'shimcha qoida bor — e'lon qilganda **darhol** qiymat berish **SHART**:

```javascript
const a;        // ❌ SyntaxError: Missing initializer in const declaration
const b = 10;   // ✅

let c;          // ✅ — let da qiymatsiz e'lon mumkin
c = 20;         // ✅ — keyinroq qiymat berish mumkin
```

### TDZ typeof da

```javascript
// var bilan — xavfsiz
console.log(typeof undeclaredVar);  // "undefined" — xato bermaydi
console.log(typeof declaredVar);    // "undefined"
var declaredVar = 1;

// let bilan — TDZ ishlaydi
console.log(typeof y); // ❌ ReferenceError — TDZ!
let y = 2;

// Umuman e'lon qilinmagan — xavfsiz
console.log(typeof neverDeclared); // "undefined" — xato bermaydi
```

`typeof` odatda xavfsiz — e'lon qilinmagan o'zgaruvchi uchun "undefined" qaytaradi. Lekin **TDZ** da turgan o'zgaruvchi uchun ReferenceError beradi. Bu `let`/`const` ning "hoist bo'ladi" ekanligining yana bir isboti.

---

## Function Declaration Hoisting

### Nazariya

Function declaration — bu hoisting'ning eng **kuchli** turi. Creation Phase da funksiya to'liq — tanasi, parametrlari va hamma narsasi bilan birga xotiraga yoziladi. Shuning uchun function declaration'ni kodda e'lon qilingan joyidan **oldin** chaqirish mumkin — engine allaqachon uni biladi.

Bu xususiyat amaliy jihatdan juda qulay: kodni yuqoridan pastga o'qiyotganda avval **nima qilinayotganini** (funksiya chaqiruvlari) ko'rib, keyin pastda **qanday qilinayotganini** (funksiya tanasi) o'qish mumkin. Ko'p tajribali dasturchilar aynan shu sababli fayl yuqori qismida asosiy logikani, pastda esa yordamchi funksiyalarni joylashtiradi.

Lekin muhim ogohlantiradigan narsa bor: bu xususiyat **faqat function declaration** uchun ishlaydi. Function expression'lar (`var f = function() {}`) va arrow function'lar (`const f = () => {}`) bu yerda **istisno** — ular o'zgaruvchi hoisting qoidalariga bo'ysunadi (ya'ni `var` bilan `undefined`, `let`/`const` bilan TDZ). Bu farqni bilmaslik ko'pincha `TypeError: ... is not a function` xatosiga olib keladi.

```javascript
// Funksiyani e'lon qilishdan OLDIN chaqirish mumkin:
greet("Islom"); // "Salom, Islom!" ✅

function greet(name) {
  console.log("Salom, " + name + "!");
}
```

Creation Phase da:

```
VariableEnvironment / LexicalEnvironment:
  greet → function(name) { console.log("Salom, " + name + "!"); }
  ↑ TO'LIQ funksiya, undefined emas!
```

### Bu Faqat Function DECLARATION uchun!

```javascript
// ✅ Function Declaration — to'liq hoist
sayHi(); // "Hi!" ✅
function sayHi() { console.log("Hi!"); }

// ❌ Function Expression — faqat variable hoist
sayHello(); // TypeError: sayHello is not a function
var sayHello = function() { console.log("Hello!"); };

// ❌ Arrow Function — faqat variable hoist
sayBye(); // TypeError: sayBye is not a function
var sayBye = () => console.log("Bye!");

// ❌ let/const bilan — TDZ
greet(); // ReferenceError — TDZ!
const greet = function() { console.log("Greet!"); };
```

---

## Function Expression Hoisting

### Nazariya

Function expression — bu funksiyani o'zgaruvchiga assign qilish (`var add = function() {}` yoki `const add = () => {}`). Bu holda hoisting qoidasi **funksiyaga** emas, **o'zgaruvchiga** taalluqli bo'ladi. Ya'ni, `var` bilan yozilgan function expression'da faqat o'zgaruvchi nomi hoist bo'ladi (`undefined` bilan), funksiyaning o'zi esa Execution Phase da assign qilinadi.

Bu farq amaliy jihatdan katta ahamiyatga ega. Ko'p dasturchilar function declaration va function expression'ni bir xil deb o'ylaydi, lekin ularning hoisting xulq-atvori butunlay farqli. `var` bilan yozilgan function expression'ni e'londan oldin chaqirsangiz `TypeError` olasiz (chunki `undefined` ni funksiya sifatida chaqirib bo'lmaydi), `let`/`const` bilan yozilganda esa `ReferenceError` (TDZ) olasiz. Zamonaviy JavaScript kodda `const` bilan function expression yozish tavsiya etiladi — bu TDZ orqali xatoni darhol ko'rsatadi va `undefined` ning "jim" muammosidan qochadi.

### var bilan Function Expression

```javascript
console.log(add);       // undefined — var hoist, lekin funksiya emas
console.log(typeof add); // "undefined"
add(1, 2);              // ❌ TypeError: add is not a function

var add = function(a, b) {
  return a + b;
};

console.log(add);       // function(a, b) { return a + b; }
add(1, 2);              // 3 ✅
```

Creation Phase:
```
add → undefined   // var hoist — faqat variable, funksiya emas!
```

Execution Phase:
```
console.log(add) → undefined
add(1, 2) → TypeError (undefined ni chaqirib bo'lmaydi!)
add = function(a, b) { return a + b; }  // endi assign bo'ldi
add(1, 2) → 3
```

### let/const bilan Function Expression

```javascript
add(1, 2);              // ❌ ReferenceError — TDZ!

const add = function(a, b) {
  return a + b;
};

add(1, 2);              // 3 ✅
```

### Named Function Expression

```javascript
var calc = function multiply(a, b) {
  return a * b;
};

console.log(calc(2, 3));     // 6 ✅
console.log(multiply(2, 3)); // ❌ ReferenceError — multiply faqat funksiya ICHIDA mavjud
```

Named function expression'ning nomi faqat funksiya ichidan ko'rinadi — recursion uchun foydali.

---

## Hoisting Priority

### Nazariya

Agar bir xil nomda `var` va `function declaration` e'lon qilinsa, kim yutadi? Bu savol intervyu'da tez-tez so'raladi va uning javobi hoisting mexanizmini qanchalik chuqur tushunishingizni ko'rsatadi.

**Qoida oddiy: Function declaration har doim `var` declaration'dan ustun turadi.** Creation Phase da engine avval barcha `var` declaration'larni xotiraga yozadi (`undefined` bilan), keyin function declaration'larni yozadi (to'liq funksiya bilan). Agar nomlar bir xil bo'lsa — function declaration ikkinchi bo'lib yoziladi va oldingi `undefined` qiymatini **qayta yozadi** (overwrite). Shuning uchun aynan bitta bosqich ichida function declaration va `var` bir xil nomga ega bo'lsa, function har doim g'alaba qozonadi.

Bu qoidani bilish nima uchun muhim? Real-world loyihalarda, ayniqsa katta jamoalarda, turli dasturchilar bir xil fayl yoki global scope'da bir xil nomli o'zgaruvchi va funksiya yaratishi mumkin. Bu holda hoisting priority'si qaysi qiymat qolishini belgilaydi. Zamonaviy kodda bu muammo kam uchraydi (chunki `let`/`const` va modullar ishlatiladi), lekin legacy code bilan ishlashda bu bilim juda zarur.

**Qoida: Function declaration > var declaration**

```javascript
console.log(foo); // function foo() { return "function"; }

var foo = "variable";
function foo() { return "function"; }

console.log(foo); // "variable"
```

### Nima uchun?

Creation Phase qadam-baqadam:

```
1. var foo → undefined (VariableEnvironment ga yozildi)
2. function foo → function(){...} (LexicalEnvironment — lekin OVERWRITE qiladi!)
   ↳ Function declaration var dan USTUN — shuning uchun foo = function

Execution Phase:
1. console.log(foo) → function foo(){...} (Creation Phase natijasi)
2. foo = "variable" — ENDI overwrite bo'ldi
3. console.log(foo) → "variable"
```

### Bir nechta Function Declaration

```javascript
foo(); // "ikkinchi" — oxirgi e'lon yutadi

function foo() { console.log("birinchi"); }
function foo() { console.log("ikkinchi"); }
```

Bir xil nomdagi function declaration'lar — oxirgisi oldingilarini **overwrite** qiladi.

### let/const bilan — Xato

```javascript
let x = 1;
let x = 2; // ❌ SyntaxError: Identifier 'x' has already been declared

const y = 1;
var y = 2; // ❌ SyntaxError

function z() {}
let z = 1; // ❌ SyntaxError
```

`let`/`const` — qayta e'lon qilishga **RUXSAT BERMAYDI**. Bu `var` ning muammolaridan biri edi.

---

## var vs let vs const — To'liq Taqqoslash

| Xususiyat | `var` | `let` | `const` |
|-----------|-------|-------|---------|
| **Scope** | Function scope | Block scope | Block scope |
| **Hoist** | Ha + `undefined` init | Ha, lekin TDZ | Ha, lekin TDZ |
| **TDZ** | Yo'q | Ha | Ha |
| **Qayta e'lon** | Ha (xato bermaydi) | Yo'q (SyntaxError) | Yo'q (SyntaxError) |
| **Qayta assign** | Ha | Ha | Yo'q (TypeError) |
| **Global scope** | `window` property | `window` da emas | `window` da emas |
| **Init kerakmi** | Yo'q | Yo'q | Ha (SHART) |
| **for loop** | Bitta instance | Har iteratsiyada yangi | — |

### Tavsiya

```javascript
// ✅ Default: const — o'zgarmas qiymatlar uchun
const API_URL = "https://api.example.com";
const config = { port: 3000 }; // object o'zi const, lekin ichki property o'zgarishi mumkin
config.port = 4000; // ✅ — const object REFERENCE ni himoyalaydi, contentni emas

// ✅ O'zgarishi kerak bo'lganda: let
let count = 0;
count++;

// ❌ Hech qachon: var — zamonaviy kodda shart emas
var oldStyle = "ishlatmang"; // scope muammolari, hoisting confusion
```

**Zamonaviy qoida:** `const` by default → `let` kerak bo'lganda → `var` hech qachon.

---

## Common Mistakes

### ❌ Xato 1: "let hoist bo'lmaydi" deb o'ylash

```javascript
let x = "global";

function test() {
  console.log(x); // ❌ ReferenceError, "global" emas!
  let x = "local";
}
test();
```

### ✅ To'g'ri tushunish:

```javascript
// let HOIST BO'LADI — shuning uchun engine local x ni "ko'radi"
// Lekin u TDZ da — shuning uchun ReferenceError
// Agar hoist bo'lmaganida — global x = "global" chiqishi kerak edi

// Isbot: agar let olib tashlansak:
let x = "global";
function test() {
  console.log(x); // "global" ✅ — endi local x yo'q, global ko'rinadi
}
test();
```

**Nima uchun:** `let x` Creation Phase da hoist bo'ladi va o'z scope'ida `x` ni "egallaydi". Tashqi `x` shadow qilinadi. Lekin TDZ tufayli kirish mumkin emas — ReferenceError.

---

### ❌ Xato 2: Function Expression ni Declaration kabi ishlatish

```javascript
// ❌ TypeError!
calculate(5, 3);

var calculate = function(a, b) {
  return a + b;
};
```

### ✅ To'g'ri usul:

```javascript
// Variant 1: Function Declaration ishlating
function calculate(a, b) {
  return a + b;
}
calculate(5, 3); // 8 ✅

// Variant 2: Avval e'lon, keyin chaqiring
const calculate2 = function(a, b) {
  return a + b;
};
calculate2(5, 3); // 8 ✅
```

**Nima uchun:** `var calculate` hoist bo'lganda `undefined` bo'ladi. `undefined(5, 3)` — TypeError. Function **declaration** esa to'liq hoist bo'ladi.

---

### ❌ Xato 3: var for loop muammosi

```javascript
// ❌ Classic closure + var bug
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 100);
}
// Output: 3, 3, 3 — 0, 1, 2 emas!
```

### ✅ To'g'ri usul:

```javascript
// ✅ let ishlatish — har iteratsiyada YANGI scope
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 100);
}
// Output: 0, 1, 2 ✅

// Nima uchun? var — bitta instance butun loop uchun (function scope)
//            let — har iteratsiyada YANGI instance (block scope)
```

**Nima uchun:** `var i` function scope da **bitta** o'zgaruvchi. Loop tugaganda `i = 3`. Barcha setTimeout callback'lari **bitta** `i` ga ishora qiladi → hammasi 3. `let i` esa har iteratsiyada **yangi block scope** yaratadi → har bir callback o'z `i` nusxasiga ega. Bu haqida ko'proq [05-closures.md](05-closures.md) da.

---

### ❌ Xato 4: Hoisting priority ni bilmaslik

```javascript
var x = 1;
function x() { return 2; }
console.log(x); // ?
```

### ✅ To'g'ri tushunish:

```javascript
// Creation Phase:
// 1. var x → undefined
// 2. function x → function(){...} — var ni OVERWRITE qildi

// Execution Phase:
// x = 1 — function ni OVERWRITE qildi

console.log(x); // 1

// Agar assignment bo'lmasa:
console.log(y); // function y(){}
var y;
function y() {}
// function declaration var dan ustun
```

**Nima uchun:** Creation Phase da function declaration var dan priority oladi. Lekin Execution Phase da `x = 1` assignment function ni overwrite qiladi.

---

### ❌ Xato 5: Block ichida function declaration

```javascript
// ❌ Behavior browserlar orasida FARQ QILADI!
if (true) {
  function sayHi() { console.log("Hi!"); }
}
sayHi(); // Ba'zi browserlarda ishlaydi, ba'zilarida ishlamaydi

// ❌ Yanada xavfli:
if (false) {
  function neverDeclared() { console.log("?"); }
}
neverDeclared(); // ??? — browser ga bog'liq!
```

### ✅ To'g'ri usul:

```javascript
// ✅ Function expression ishlatish block ichida
let sayHi;
if (true) {
  sayHi = function() { console.log("Hi!"); };
}
sayHi(); // "Hi!" ✅ — predictable behavior
```

**Nima uchun:** Block ichidagi function declaration behavior spec da aniq emas (legacy reasons) va browserlar turlicha implement qiladi. Shuning uchun block ichida function **expression** ishlatish xavfsiz.

---

## Amaliy Mashqlar

### Mashq 1: Output ni Aniqlang (Oson)

**Savol:**

```javascript
console.log(a);
console.log(b);
console.log(c);

var a = 1;
let b = 2;
const c = 3;
```

<details>
<summary>Javob</summary>

```javascript
console.log(a); // undefined — var hoist + undefined init
console.log(b); // ❌ ReferenceError: Cannot access 'b' before initialization
// Keyingi satrlar bajarilMAYDI (xato tufayli)
```

**Tushuntirish:**
- `var a` → hoist + `undefined` bilan init → kirish mumkin
- `let b` → hoist lekin TDZ da → ReferenceError → dastur to'xtaydi
- `const c` ga yetmay qoldi
</details>

---

### Mashq 2: Function Hoisting (Oson)

**Savol:**

```javascript
console.log(foo);
console.log(bar);

function foo() { return 1; }
var bar = function() { return 2; };
```

<details>
<summary>Javob</summary>

```javascript
console.log(foo); // function foo() { return 1; }
console.log(bar); // undefined
```

**Tushuntirish:**
- `function foo` → function declaration → **to'liq** hoist
- `var bar` → faqat variable hoist → `undefined` (funksiya emas!)
</details>

---

### Mashq 3: Tricky TDZ (O'rta)

**Savol:**

```javascript
let x = 10;

function test() {
  if (false) {
    let x = 20; // Bu satr HECH QACHON bajarilmaydi
  }
  console.log(x); // ?
}

test();
```

<details>
<summary>Javob</summary>

```javascript
console.log(x); // 10 ✅
```

**Tushuntirish:**

`let x = 20` **if block** ichida — u o'zining block scope'iga ega. `if (false)` tufayli bu block bajarilmaydi, lekin bu muhim emas — chunki `let x = 20` faqat **if block** ning scope'ida mavjud.

Funksiya scope'ida `x` yo'q → scope chain orqali global `x = 10` topiladi.

Agar `let x = 20` if block'siz bo'lsa:
```javascript
function test() {
  console.log(x); // ❌ ReferenceError — TDZ!
  let x = 20;
}
```
Bu holda TDZ ishlardi, chunki `let x` funksiya scope'ida.
</details>

---

### Mashq 4: Priority Puzzle (Qiyin)

**Savol:**

```javascript
console.log(typeof foo); // ?

var foo = "hello";
function foo() { return "world"; }

console.log(typeof foo); // ?
console.log(foo);         // ?
```

<details>
<summary>Javob</summary>

```javascript
console.log(typeof foo); // "function"
// ...
console.log(typeof foo); // "string"
console.log(foo);         // "hello"
```

**Tushuntirish:**

Creation Phase:
1. `var foo` → `undefined`
2. `function foo` → `function(){...}` — **overwrite** qildi

Shuning uchun birinchi `typeof foo` = `"function"`

Execution Phase:
1. `foo = "hello"` — endi string bilan overwrite
2. `typeof foo` = `"string"`, `foo` = `"hello"`
</details>

---

### Mashq 5: Kompleks Hoisting (Qiyin)

**Savol:** Har bir `console.log` ning natijasini aniqlang:

```javascript
var a = 1;
function b() {
  a = 10;
  return;
  function a() {}
}
b();
console.log(a); // ?
```

<details>
<summary>Javob</summary>

```javascript
console.log(a); // 1
```

**Tushuntirish:**

Bu klassik interview trick. Qadam-baqadam:

1. Global: `var a = 1`
2. `b()` chaqirildi
3. `b()` ning **Creation Phase** da: `function a(){}` hoist bo'ldi → **local** `a` = function
4. `b()` ning Execution Phase da: `a = 10` — bu **LOCAL** `a` ni o'zgartiradi (global emas!)
5. `return` — funksiya tugadi
6. `function a(){}` — bu allaqachon Creation Phase da hoist bo'lgan, bu satr execution da hech narsa qilmaydi
7. Global `a` hali **1** — o'zgarmagan!

```
b() EC:
  Creation Phase: a → function(){} (LOCAL — function declaration hoist)
  Execution Phase: a = 10 (LOCAL a ni o'zgartirdi, global a emas)

Global EC:
  a: 1 (o'zgarmagan!)
```

**Kalit:** `function a(){}` `b()` ichida **local** `a` yaratdi. `a = 10` shu local `a` ga yozildi, global `a` ga tegmadi.
</details>

---

## Xulosa

1. **Hoisting = Creation Phase natijasi.** Kod ko'tarilmaydi — engine oldindan xotira ajratadi.

2. **`var`** — hoist + `undefined` bilan initialize. Shuning uchun e'londan oldin `undefined` qaytaradi.

3. **`let`/`const`** — hoist bo'ladi, lekin **TDZ** da. E'londan oldin kirish = ReferenceError.

4. **TDZ (Temporal Dead Zone)** — scope boshlanishidan e'lon satrigacha. Vaqtga asoslangan, pozitsiyaga emas.

5. **Function Declaration** — to'liq hoist (body bilan birga). E'londan oldin chaqirish mumkin.

6. **Function Expression** — faqat variable hoist. `var` bilan = undefined, `let`/`const` bilan = TDZ.

7. **Priority:** Function Declaration > var. Bir xil nom bo'lsa, function yutadi (Creation Phase da).

8. **`var` muammolari:** function scope (block'dan chiqadi), qayta e'lon ruxsat, global scope da window property. Shuning uchun zamonaviy kodda `const` > `let` > ~~`var`~~.

---

> **Keyingi bo'lim:** [04-scope.md](04-scope.md) — Scope Chain — o'zgaruvchilar qanday qidiriladi, lexical scope, variable shadowing.
