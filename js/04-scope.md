# Bo'lim 4: Scope Chain

> Scope — o'zgaruvchilar va funksiyalarning kodning qaysi qismida accessible ekanligini belgilaydigan qoidalar to'plami. JavaScript lexical scoping ishlatadi — scope kod **yozilgan** joyga qarab, compile-time da aniqlanadi.

---

## Mundarija

- [Scope Nima](#scope-nima)
- [Global Scope](#global-scope)
- [Function Scope](#function-scope)
- [Block Scope](#block-scope)
- [Lexical Scope (Static Scope)](#lexical-scope-static-scope)
- [Dynamic Scope vs Lexical Scope](#dynamic-scope-vs-lexical-scope)
- [Scope Chain](#scope-chain)
- [Scope Chain va Execution Context Bog'liqligi](#scope-chain-va-execution-context-bogliqligi)
- [Variable Shadowing](#variable-shadowing)
- [Variable Lookup](#variable-lookup)
- [Strict Mode](#strict-mode)
- [Labeled Statements](#labeled-statements)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Scope Nima

### Nazariya

Scope — o'zgaruvchilar, funksiyalar va class'larning **accessibility (kirish mumkinlik) chegarasi**. Har bir o'zgaruvchi qandaydir scope ichida e'lon qilinadi va faqat shu scope (va uning ichki scope'lari) dan ko'rinadi.

Scope'ning mavjud bo'lishiga sabab — **encapsulation** (ma'lumotni ajratish). Agar barcha o'zgaruvchilar hamma joydan ko'rinadigan bo'lsa, nomlar to'qnashuvi (name collision) muqarrar bo'ladi. Scope tizimi har bir kod blokiga o'z "xotira maydoni" beradi — bu maydon ichidagi nomlar tashqariga chiqmaydi.

JavaScript da uchta asosiy scope turi mavjud:

1. **Global Scope** — script'ning eng yuqori darajasi, hamma joydan accessible
2. **Function Scope** — funksiya tanasi ichida yaratiladi, faqat shu funksiya ichidan ko'rinadi
3. **Block Scope** — `{}` qavslar ichida `let`/`const` bilan yaratiladi (ES6+)

Bu uch tur bir-birining ichiga joylashishi (nesting) mumkin — natijada **scope chain** (scope zanjiri) hosil bo'ladi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spetsifikatsiyasida scope tushunchasi **Environment Record** orqali implement qilingan. Har bir scope uchun yangi Environment Record yaratiladi. Bu record ichida shu scope'dagi barcha binding'lar (o'zgaruvchi-qiymat juftliklari) saqlanadi.

```
Scope turlari va Environment Record aloqasi:

Global Scope     →  Global Environment Record (Object + Declarative)
Function Scope   →  Function Environment Record (Declarative)
Block Scope      →  Declarative Environment Record
Module Scope     →  Module Environment Record (Declarative)
```

Global Environment Record ikki qismdan iborat:
- **Object Environment Record** — `var` va `function` declaration'lar bu yerda, global object (`window`/`globalThis`) ning property'lari sifatida
- **Declarative Environment Record** — `let`, `const`, `class` declaration'lar bu yerda, global object'ga tushMAYDI

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Uchta scope turini bir misolda ko'rsatadigan kod:

```javascript
// ── Global Scope ──
const appName = "MyApp";

function processUser(name) {
  // ── Function Scope ──
  const prefix = "User";

  if (name.length > 0) {
    // ── Block Scope ──
    const fullName = `${prefix}: ${name}`;
    console.log(fullName); // ✅ "User: Alice" — block ichida accessible
  }

  // console.log(fullName);
  // ❌ ReferenceError — fullName block scope'da, bu yerda ko'rinmaydi

  console.log(prefix); // ✅ "User" — function scope ichida
}

processUser("Alice");
// console.log(prefix);
// ❌ ReferenceError — prefix function scope'da, global'da ko'rinmaydi
```

</details>

---

## Global Scope

### Nazariya

Global scope — script'ning eng tashqi qatlami. Bu yerda e'lon qilingan o'zgaruvchilar dasturning **istalgan joyidan** accessible — barcha funksiyalar, barcha bloklar, barcha modullar (module scope'dan tashqari) ichidan ko'rinadi.

Global scope'da ikki xil narsa saqlanadi:

1. **`var` va `function` declaration'lar** — global object (`window` browser'da, `global` Node.js da) ning property'lariga aylanadi
2. **`let`, `const`, `class` declaration'lar** — global scope'da mavjud, lekin global object'ning property'si bo'lMAYDI

`var` bilan e'lon qilingan global o'zgaruvchi `window.variableName` orqali ham accessible, `let`/`const` bilan e'lon qilingan esa faqat identifier orqali.

**`globalThis` (ES2020)** — cross-environment global object'ga murojaat qilish uchun standart yo'l. Browser'da `globalThis === window`, Node.js da `globalThis === global`, Web Worker'da `globalThis === self`. ES2020 dan oldin har bir environment uchun alohida nom ishlatish kerak edi.

<details>
<summary><strong>Under the Hood</strong></summary>

Global scope'ning Environment Record tuzilishi:

```
Global Environment Record
├── Object Environment Record (bindingObject = globalThis)
│   ├── var message = "hello"     → globalThis.message = "hello"
│   ├── function greet() {...}    → globalThis.greet = function(){...}
│   └── [built-ins: parseInt, Math, JSON, ...]
│
├── Declarative Environment Record
│   ├── let count = 0             → faqat identifier orqali accessible
│   ├── const PI = 3.14           → faqat identifier orqali accessible
│   └── class User {...}          → faqat identifier orqali accessible
│
└── [[GlobalThisValue]] → globalThis object
```

`var` global declaration nima uchun `globalThis` property bo'ladi? ECMAScript spec bo'yicha global code'dagi `var` statement **CreateGlobalVarBinding** abstract operation ni chaqiradi — bu operation global object'ga property qo'shadi. `let`/`const` esa **CreateGlobalLetBinding** ni chaqiradi — bu global object'ga tegmaydi, faqat declarative record'ga yozadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`var` vs `let`/`const` ning global scope'dagi farqi:

```javascript
var oldWay = "var bilan";
let newWay = "let bilan";
const alsoNew = "const bilan";

console.log(globalThis.oldWay);   // ✅ "var bilan" — window property
console.log(globalThis.newWay);   // ✅ undefined — window property EMAS
console.log(globalThis.alsoNew);  // ✅ undefined — window property EMAS

// globalThis turli environment larda
console.log(globalThis === window);  // ✅ true (browser)
console.log(globalThis === global);  // ✅ true (Node.js)
console.log(globalThis === self);    // ✅ true (Web Worker)
```

Global scope pollution muammosi — uchinchi tomon kutubxona ham global'da nom e'lon qilsa, to'qnashuv bo'ladi:

```javascript
// kutubxona-A.js
var utils = { format: () => "A" };

// kutubxona-B.js
var utils = { format: () => "B" };
// ❌ utils o'zgaruvchisi qayta yozildi — kutubxona-A ishlashdan to'xtadi

// Yechim: modullar yoki IIFE orqali scope ajratish
// (batafsil 15-modules.md da)
```

</details>

---

## Function Scope

### Nazariya

Function scope — funksiya tanasi (`{ }`) ichida yaratiladi. Funksiya ichida `var` bilan e'lon qilingan o'zgaruvchilar **faqat shu funksiya ichidan** accessible. Funksiya tashqarisidan ularga murojaat qilish imkonsiz — `ReferenceError` beradi.

`var` keyword **faqat function scope** ni tan oladi — block scope (`if`, `for`, `while`) ni tanimaydi. Bu `var` ning eng asosiy xususiyati va ko'plab xatolarga sabab bo'ladigan xulq-atvori.

Funksiya parametrlari ham function scope'ga tegishli — ular shu funksiyaning local o'zgaruvchilari hisoblanadi.

Har bir funksiya chaqiruvi **yangi scope** yaratadi. Bitta funksiya 10 marta chaqirilsa — 10 ta alohida function scope hosil bo'ladi, har birida o'z local o'zgaruvchilari.

<details>
<summary><strong>Under the Hood</strong></summary>

Funksiya chaqirilganda engine yangi **Function Execution Context** yaratadi. Bu context'ning **VariableEnvironment** component'ida `var` declaration'lar, **LexicalEnvironment** component'ida `let`/`const` declaration'lar saqlanadi.

```
processOrder() chaqirilganda:

Function Execution Context
├── VariableEnvironment (Function Environment Record)
│   ├── arguments: Arguments object
│   ├── orderId: undefined → keyin 42
│   └── var bilan e'lon qilingan boshqa o'zgaruvchilar
│
├── LexicalEnvironment (Function Environment Record)
│   ├── let/const bilan e'lon qilinganlar
│   └── [[OuterEnv]] → Global Environment Record
│
└── ThisBinding → (chaqiruv kontekstiga qarab)
```

`var` ning function scope xulq-atvori tufayli, agar `if` block ichida `var` ishlatilsa, u function scope'ga "ko'tariladi" (hoist bo'ladi):

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`var` ning function scope xulq-atvori:

```javascript
function checkAge(age) {
  if (age >= 18) {
    var status = "adult";
    // ✅ var → function scope, if block uni cheklamaydi
  }

  console.log(status);
  // ✅ "adult" — var function scope'da, if tashqarisida ham ko'rinadi
  // agar age < 18 bo'lsa → undefined (hoist bo'lgan, assign bo'lmagan)
}

checkAge(25); // "adult"
checkAge(15); // undefined — xato EMAS, faqat undefined
```

Har bir funksiya chaqiruvi alohida scope yaratadi:

```javascript
function counter() {
  var count = 0;
  count++;
  return count;
}

console.log(counter()); // 1
console.log(counter()); // 1 — har chaqiruvda YANGI scope, yangi count
// ✅ oldingi chaqiruvdagi count yo'q bo'lgan — yangi EC yaratildi
```

</details>

---

## Block Scope

### Nazariya

Block scope — `{}` qavslar (curly braces) ichida `let` yoki `const` bilan e'lon qilingan o'zgaruvchilar **faqat shu block ichida** accessible bo'lishi. Bu ES6 (2015) da kiritilgan — undan oldin JavaScript da block scope mavjud emas edi, faqat function scope bor edi.

Block scope yaratadigan konstruktsiyalar:
- `if / else if / else` bloklari
- `for`, `for...in`, `for...of` loop'lari
- `while`, `do...while` loop'lari
- `switch` statement
- `try / catch / finally` bloklari
- Oddiy `{}` — standalone block (block statement)

`var` keyword block scope'ni **tanimaydi** — u faqat function scope ni ko'radi. Shu sababli `let`/`const` ning block scope'da ishlashi `var` dan tubdan farq qiladi.

`for` loop'da `let` ishlatilganida har bir iteratsiya uchun **yangi block scope** yaratiladi — bu closure bilan ishlashda juda muhim farq (batafsil [05-closures.md](05-closures.md) da).

<details>
<summary><strong>Under the Hood</strong></summary>

Engine block scope uchun yangi **Declarative Environment Record** yaratadi. Bu record joriy execution context'ning LexicalEnvironment'iga ulanadi. Block tugaganda bu record yo'qoladi (GC tomonidan tozalanadi, agar closure reference saqlamasa).

```
for (let i = 0; i < 3; i++) — har bir iteratsiyada:

Iteratsiya 0:
  Block Environment Record { i: 0 }
    └── [[OuterEnv]] → Function/Global Environment Record

Iteratsiya 1:
  Block Environment Record { i: 1 }  ← YANGI record
    └── [[OuterEnv]] → Function/Global Environment Record

Iteratsiya 2:
  Block Environment Record { i: 2 }  ← YANGI record
    └── [[OuterEnv]] → Function/Global Environment Record
```

`var` bilan `for` da faqat **bitta** binding yaratiladi — barcha iteratsiyalar shu bitta `i` ni ko'radi. `let` bilan har iteratsiya **yangi** binding oladi — har biri o'z `i` qiymatiga ega.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Block scope ning amaliy farqi:

```javascript
function showItems(items) {
  for (let i = 0; i < items.length; i++) {
    const item = items[i];
    console.log(item);
  }

  // console.log(i);
  // ❌ ReferenceError — i faqat for block ichida (let)

  // console.log(item);
  // ❌ ReferenceError — item faqat for body ichida (const)
}
```

`var` vs `let` — loop + async muammosi:

```javascript
// ❌ var bilan — barcha callback'lar bitta i ni ko'radi
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3
// ✅ Sabab: var function-scoped, bitta binding — loop tugaganda i = 3

// ✅ let bilan — har iteratsiya o'z i binding'iga ega
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2
// ✅ Sabab: let block-scoped, har iteratsiyada yangi binding yaratiladi
```

Standalone block statement:

```javascript
{
  const temp = computeValue();
  processResult(temp);
}
// temp bu yerda ko'rinmaydi — scope tugadi
// ✅ Foyda: vaqtinchalik o'zgaruvchilar tashqi scope'ni ifloslamaydi
```

</details>

---

## Lexical Scope (Static Scope)

### Nazariya

Lexical scope (yoki static scope) — scope'ning **kod yozilgan pozitsiyaga** qarab aniqlanishi. JavaScript lexical scoping ishlatadi — funksiyaning scope'i u **define qilingan** joyga bog'liq, **chaqirilgan** joyga emas.

"Lexical" so'zi "leksik analiz" (lexical analysis) dan olingan — bu compiler/interpreter ning source code'ni tokenize qilish bosqichi. Scope aynan shu bosqichda, ya'ni **parse-time** da (kod o'qilayotganda) aniqlanadi, runtime da emas.

Bu shuni anglatadi: funksiya tanasiga qarab, uning qaysi tashqi o'zgaruvchilarga kirishi mumkinligini **kodni o'qib** aniqlash mumkin. Runtime da scope o'zgarmaydi (bir nechta istisnolardan tashqari, masalan `eval()` va `with` — ikkalasi ham zamonaviy JavaScript da ishlatish tavsiya etilmaydi).

Lexical scope'ning muhim oqibati — funksiya qayerda chaqirilmasin, u o'zi yaratilgan scope'dagi o'zgaruvchilarga murojaat qiladi. Bu xulq-atvor **closure** ning asosi (batafsil [05-closures.md](05-closures.md) da).

<details>
<summary><strong>Under the Hood</strong></summary>

Har bir funksiya yaratilganda (define qilinganda — e'lon yoki expression bilan), engine uning **`[[Environment]]`** internal slot'iga **joriy LexicalEnvironment** ni saqlaydi. Bu slot funksiyaning "tug'ilgan joyi" ni eslab qoladi.

```
const x = 10;

function outer() {
  const y = 20;

  function inner() {
    console.log(x, y);
  }

  return inner;
}

inner yaratilgan paytda:
  inner.[[Environment]] → outer() ning LexicalEnvironment
    ├── y: 20
    └── [[OuterEnv]] → Global LexicalEnvironment
                         └── x: 10

inner() qayerda chaqirilmasin — uning [[Environment]] o'zgarmaydi.
Doim outer() scope'iga, keyin global scope'ga murojaat qiladi.
```

Bu mexanizm compile-time (parse-time) da belgilanadi — engine AST ni traverse qilib, har bir identifier qaysi scope'ga tegishli ekanini **statik** aniqlaydi. V8 da bu jarayon "scope analysis" deyiladi va bytecode generation paytida sodir bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Lexical scope ishini ko'rsatuvchi misol — funksiya boshqa scope'da chaqirilsa ham o'z scope'ini ishlatadi:

```javascript
const language = "JavaScript";

function getLanguage() {
  return language;
  // ✅ language → lexical scope bo'yicha global'dagi "JavaScript"
}

function wrapper() {
  const language = "TypeScript";
  // Bu local language getLanguage() ga hech qanday ta'sir ko'rsatmaydi
  return getLanguage();
}

console.log(wrapper()); // "JavaScript" — "TypeScript" EMAS
// ✅ getLanguage() global scope'da define qilingan
// ✅ Uning [[Environment]] global scope'ga ishora qiladi
// ✅ wrapper() ichidagi language ga murojaat qilmaydi
```

Nested lexical scope:

```javascript
function createMultiplier(factor) {
  // factor → createMultiplier scope'da
  return function multiply(number) {
    // multiply bu yerda define qilingan →
    // uning [[Environment]] = createMultiplier scope
    return number * factor;
    // ✅ factor lexical scope bo'yicha createMultiplier dan olinadi
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5));  // 10 — factor = 2 (birinchi chaqiruvdan)
console.log(triple(5));  // 15 — factor = 3 (ikkinchi chaqiruvdan)
// ✅ Har bir chaqiruv alohida scope yaratdi, har birida o'z factor qiymati
```

</details>

---

## Dynamic Scope vs Lexical Scope

### Nazariya

Scope aniqlashning ikkita fundamental yondashuvi bor:

1. **Lexical (Static) Scope** — scope kod **yozilgan** joyga qarab aniqlanadi (compile-time). JavaScript, C, Java, Python, Go — barchasi lexical scope ishlatadi.

2. **Dynamic Scope** — scope funksiya **chaqirilgan** joyga qarab aniqlanadi (runtime). Bash shell, ba'zi Lisp dialektlari, Perl (maxsus `local` bilan) — dynamic scope'ga misol.

Asosiy farq:

| Xususiyat | Lexical Scope | Dynamic Scope |
|-----------|--------------|---------------|
| **Aniqlanish vaqti** | Compile-time (parse-time) | Runtime (execution-time) |
| **Nimaga bog'liq** | Kod yozilgan pozitsiya | Funksiya chaqirilgan kontekst |
| **Predictability** | Yuqori — kodni o'qib bilsa bo'ladi | Past — runtime da o'zgaradi |
| **Debugging** | Oson | Qiyin |
| **Performance** | Tezroq — statik analiz mumkin | Sekinroq — har chaqiruvda qidirish |
| **JS da** | Ha — standart | Yo'q (lekin `this` o'xshash xulq ko'rsatadi) |

JavaScript da `this` keyword dynamic scope'ga **yuzaki o'xshaydi** — u funksiya **chaqirilgan** kontekstga qarab o'zgaradi (batafsil [10-this-keyword.md](10-this-keyword.md) da). Lekin bu aynan dynamic scope emas: `this` chaqiruvchi kod tomonidan aniqlangan **yagona binding**, u scope chain orqali o'zgaruvchilarni qidirmaydi. Haqiqiy dynamic scope'da barcha identifier'lar (nafaqat `this`) chaqiruv zanjiri bo'ylab qidirilgan bo'lar edi. JavaScript da esa o'zgaruvchi lookup doim lexical scope — faqat `this` binding call-site'ga bog'liq.

<details>
<summary><strong>Kod Misollari</strong></summary>

Agar JavaScript dynamic scope ishlatganida nima bo'lishini ko'rsatuvchi solishtirma:

```javascript
const value = "global";

function readValue() {
  console.log(value);
}

function callWithLocal() {
  const value = "local";
  readValue();
}

callWithLocal();

// LEXICAL SCOPE (JavaScript haqiqiy xulqi):
// Output: "global"
// ✅ readValue() global scope'da define qilingan →
//    uning [[Environment]] global scope → value = "global"

// AGAR DYNAMIC SCOPE BO'LGANIDA:
// Output: "local" bo'lardi
// ❌ readValue() callWithLocal() ichidan chaqirilgan →
//    callWithLocal() scope'idagi value = "local" ishlatilardi
```

</details>

---

## Scope Chain

### Nazariya

Scope chain — o'zgaruvchi qidirilganda engine bosib o'tadigan **scope'lar ketma-ketligi**. Qidiruv doim **ichki scope'dan boshlanadi** va tashqariga (yuqoriga) qarab davom etadi — eng tashqi scope global scope.

Scope chain quyidagicha quriladi:

1. Joriy scope (funksiya yoki block) — birinchi tekshiriladi
2. Tashqi (parent) scope — joriy scope'ning `[[OuterEnv]]` reference'i orqali
3. Undan ham tashqi scope — yana `[[OuterEnv]]` orqali
4. ... va hokazo ...
5. Global scope — zanjirning eng tashqi halqasi
6. Global scope'da ham topilmasa → `ReferenceError`

Scope chain'ning muhim qoidalari:
- **Faqat ichdan tashqariga** — tashqi scope ichki scope'dagi o'zgaruvchini ko'rMAYDI
- **Birinchi topilgan qiymat** — qidiruv birinchi topilgan joyda to'xtaydi (shadowing)
- **Scope chain o'zgarmaydi** — u funksiya yaratilgan paytda belgilanadi (lexical)

<details>
<summary><strong>Under the Hood</strong></summary>

Scope chain fizik ravishda **Environment Record'larning `[[OuterEnv]]` zanjiri** orqali implement qilingan. Har bir Environment Record o'zining tashqi (parent) Environment Record'iga reference saqlaydi.

```
function a() {
  let x = 1;
  function b() {
    let y = 2;
    function c() {
      let z = 3;
      console.log(x + y + z); // 6
    }
    c();
  }
  b();
}
a();

Scope Chain vizualizatsiya (c() ichidan):

c() Environment Record
  ├── z: 3
  └── [[OuterEnv]] ──→ b() Environment Record
                          ├── y: 2
                          └── [[OuterEnv]] ──→ a() Environment Record
                                                 ├── x: 1
                                                 └── [[OuterEnv]] ──→ Global Environment Record
                                                                        ├── a: function
                                                                        └── [[OuterEnv]] → null

console.log(x + y + z) bajarilganda:
1. z → c() da topildi ✅ (z = 3)
2. y → c() da yo'q → b() da topildi ✅ (y = 2)
3. x → c() da yo'q → b() da yo'q → a() da topildi ✅ (x = 1)
```

V8 engine'da scope chain traversal optimizatsiya qilingan: agar engine parse-time da identifier qaysi scope'ga tegishli ekanini aniqlay olsa, u **to'g'ridan-to'g'ri** shu scope'ga murojaat qiladi — zanjir bo'ylab ketma-ket qidirmaydi. Bu "scope resolution" deyiladi va bytecode'da index orqali amalga oshiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Chuqur scope chain misoli:

```javascript
const config = { debug: true };

function createApp(name) {
  const version = "1.0";

  function createRouter(prefix) {
    const routes = [];

    function addRoute(path, handler) {
      // addRoute ichida barcha tashqi scope o'zgaruvchilari accessible:
      routes.push({
        fullPath: `${prefix}${path}`,        // ✅ prefix — createRouter scope
        handler,
        app: name,                            // ✅ name — createApp scope
        appVersion: version,                  // ✅ version — createApp scope
        isDebug: config.debug                 // ✅ config — global scope
      });
    }

    return { addRoute, routes };
  }

  return createRouter;
}

const router = createApp("MyApp")("/api");
router.addRoute("/users", () => {});
console.log(router.routes[0]);
// { fullPath: "/api/users", handler: fn, app: "MyApp", appVersion: "1.0", isDebug: true }
```

</details>

---

## Scope Chain va Execution Context Bog'liqligi

### Nazariya

Scope chain va Execution Context bir-biriga chambarchas bog'langan — scope chain Execution Context'ning **LexicalEnvironment** component'i orqali quriladi.

[02-execution-context.md](02-execution-context.md) da o'rganganimizdek, har bir Execution Context ikkita environment component'ga ega:

1. **VariableEnvironment** — `var` declaration'lar va `function` declaration'lar saqlanadi. Funksiya scope'ni ifodalaydi.
2. **LexicalEnvironment** — `let`, `const`, `class` declaration'lar saqlanadi. Block scope'ni ifodalaydi.

Ikkalasida ham **`[[OuterEnv]]`** (outer reference) bor — shu reference scope chain'ni hosil qiladi.

Aloqani aniq ko'rsatuvchi jarayon:

1. Funksiya **yaratilganda** — `[[Environment]]` slot'iga joriy LexicalEnvironment saqlanadi
2. Funksiya **chaqirilganda** — yangi Execution Context yaratiladi
3. Yangi EC'ning LexicalEnvironment'ining `[[OuterEnv]]` reference'i → funksiyaning `[[Environment]]` slot'idagi qiymatga o'rnatiladi
4. Natija: yangi EC ichidan tashqi scope'ga yo'l ochiladi — scope chain tayyor

<details>
<summary><strong>Under the Hood</strong></summary>

```
// Source code:
function outer() {
  const x = 10;
  function inner() {
    console.log(x);
  }
  inner();
}
outer();

// Runtime jarayoni:

1. outer() CHAQIRILDI:
   ┌─ Execution Context: outer ──────────────────┐
   │  LexicalEnvironment:                         │
   │    Environment Record: { x: 10, inner: fn }  │
   │    [[OuterEnv]]: Global Environment           │
   └──────────────────────────────────────────────┘

2. inner() YARATILDI (outer ichida):
   inner.[[Environment]] = outer'ning LexicalEnvironment

3. inner() CHAQIRILDI:
   ┌─ Execution Context: inner ──────────────────────────┐
   │  LexicalEnvironment:                                 │
   │    Environment Record: { } (local o'zgaruvchi yo'q)  │
   │    [[OuterEnv]]: outer'ning LexicalEnvironment ──────┼──→ { x: 10 }
   └──────────────────────────────────────────────────────┘

4. console.log(x):
   inner EC → x yo'q → [[OuterEnv]] → outer EC → x = 10 ✅
```

Call Stack va Scope Chain **bir-biridan mustaqil**. Call Stack funksiyalarning **chaqiruv tartibini** boshqaradi (LIFO), Scope Chain esa o'zgaruvchilarning **qidiruv yo'lini** belgilaydi (lexical nesting). Funksiya call stack'dan chiqsa ham, uning scope'i closure orqali saqlanib qolishi mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Execution Context va Scope Chain'ning birga ishlashini ko'rsatuvchi misol:

```javascript
let step = 0;

function first() {
  let a = 1;
  step++;
  console.log(`Step ${step}: first() — a=${a}`);
  second();
}

function second() {
  let b = 2;
  step++;
  console.log(`Step ${step}: second() — b=${b}`);
  // console.log(a);
  // ❌ ReferenceError — a first() scope'da
  // ✅ second() global scope'da define qilingan
  //    uning scope chain: second → global
  //    first() scope'ga yo'l YO'Q (lexical, call-based emas)
}

first();
// Step 1: first() — a=1
// Step 2: second() — b=2
```

</details>

---

## Variable Shadowing

### Nazariya

Variable shadowing — ichki scope'da tashqi scope'dagi o'zgaruvchi bilan **bir xil nomli** yangi o'zgaruvchi e'lon qilish. Bu holda ichki scope'da yangi (local) o'zgaruvchi tashqi o'zgaruvchini "yashiradi" — ichki scope ichidan tashqi o'zgaruvchiga murojaat qilish imkonsiz bo'ladi.

Shadowing qoidalari:

- `let` / `const` **shadow qila oladi** — tashqi scope'dagi `var`, `let`, `const` yoki function parameter'ni yashirishi mumkin (ichki block'da yangi binding orqali)
- `var` **shadow qila oladi** tashqi function scope'dagi `var` ni (lekin bu block ichida emas, yangi function scope'da amalga oshadi)
- `var` **to'qnashadi** bir xil function scope'dagi `let`/`const` bilan — bu SyntaxError beradi, chunki `var` function scope'ga ko'tariladi va u yerdagi `let`/`const` binding bilan kolleziya qiladi
- Shadowed tashqi o'zgaruvchining original qiymati o'zgarmaydi — ichki scope'dagi o'zgaruvchi mutlaqo alohida binding'dir

Shadow bo'lgan o'zgaruvchiga qayta murojaat qilishning **yagona yo'li** — global scope'da `globalThis.variableName` orqali (faqat `var` yoki global property bo'lsa).

<details>
<summary><strong>Kod Misollari</strong></summary>

Shadowing'ning turli holatlari:

```javascript
const name = "Global";

function greet() {
  const name = "Function";
  // ✅ Shadowing — global name "yashirildi"

  if (true) {
    const name = "Block";
    // ✅ Shadowing — function name ham "yashirildi"
    console.log(name); // "Block"
  }

  console.log(name); // "Function" — block scope tugadi, function scope tiklanadi
}

greet();
console.log(name); // "Global" — function scope tugadi, global tiklanadi
```

`var` va `let` shadowing muammosi:

```javascript
function example() {
  let x = 10;

  if (true) {
    // let x = 20; // ✅ Bu ishlaydi — let block scope'da shadow
    // var x = 20; // ❌ SyntaxError: Identifier 'x' has already been declared
    // Sabab: var function scope'ga ko'tariladi va let bilan to'qnashadi
  }
}
```

</details>

---

## Variable Lookup

### Nazariya

Variable lookup — engine'ning o'zgaruvchi qiymatini topish jarayoni. Identifier (o'zgaruvchi nomi) ishlatilganda engine quyidagi algoritmni bajaradi:

1. **Joriy Environment Record** — identifier shu record'da bormi? Agar ha — qiymatini qaytar
2. **`[[OuterEnv]]` bo'ylab tashqariga** — joriy record'da yo'q bo'lsa, outer reference bo'ylab tashqi scope'ga o't
3. **Jarayon takrorlanadi** — har bir tashqi scope'da identifier qidiriladi
4. **Global scope** — eng tashqi scope, bu yerda ham topilmasa:
   - **Strict mode'da** → `ReferenceError: x is not defined`
   - **Non-strict mode'da** → agar bu **assignment** bo'lsa (masalan `x = 10`), engine global object'ga yangi property qo'shadi (implicit global) — bu **xato** va strict mode'da taqiqlangan

Variable lookup ikki kontekstda farq qiladi:

- **Read (o'qish)** — `console.log(x)` — qiymatni olish, topilmasa `ReferenceError`
- **Write (yozish)** — `x = 10` — assign qilish, strict mode'da topilmasa `ReferenceError`, non-strict'da implicit global

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec'da variable lookup **ResolveBinding** abstract operation orqali amalga oshiriladi:

```
ResolveBinding(name, env):
  1. Agar env berilmagan bo'lsa → env = running EC ning LexicalEnvironment
  2. Agar env === null → ReferenceError (global scope'dan ham o'tdi)
  3. exists = env.HasBinding(name)
  4. Agar exists === true → return env.GetBindingValue(name)
  5. Agar exists === false → return ResolveBinding(name, env.[[OuterEnv]])

Bu recursive jarayon — har bir scope'da tekshirib, topilmasa tashqariga o'tadi.
```

V8 optimizatsiyasi: V8 parse-time da har bir identifier uchun scope depth va index ni aniqlaydi. Runtime da zanjir bo'ylab qidirmasdan, to'g'ridan-to'g'ri kerakli scope'dagi index'ga murojaat qiladi. Bu **scope caching** deyiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Lookup jarayoni qadam-baqadam:

```javascript
const globalVar = "G";

function outer() {
  const outerVar = "O";

  function inner() {
    const innerVar = "I";

    // 1-lookup: innerVar
    console.log(innerVar);
    // inner scope → topildi ✅ ("I")

    // 2-lookup: outerVar
    console.log(outerVar);
    // inner scope → yo'q → outer scope → topildi ✅ ("O")

    // 3-lookup: globalVar
    console.log(globalVar);
    // inner → yo'q → outer → yo'q → global → topildi ✅ ("G")

    // 4-lookup: unknown
    // console.log(unknown);
    // inner → yo'q → outer → yo'q → global → yo'q → ReferenceError ❌
  }

  inner();
}

outer();
```

Implicit global muammosi (non-strict mode):

```javascript
function leaky() {
  // "use strict" yo'q
  leaked = "Oops!";
  // ❌ let/const/var yo'q — engine scope chain bo'ylab qidiradi
  // Hech joyda topilmaydi → global object'ga property qo'shadi
  // globalThis.leaked = "Oops!"
}

leaky();
console.log(leaked); // "Oops!" — global scope iflos bo'ldi
console.log(globalThis.leaked); // "Oops!" — globalThis property
```

</details>

---

## Strict Mode

### Nazariya

Strict mode — JavaScript'ning **qat'iyroq** rejimi bo'lib, `"use strict"` directive bilan yoqiladi. ES5 (2009) da kiritilgan. Strict mode bir qator xavfli va xatoga olib keladigan xulq-atvorlarni taqiqlaydi, code'ni xavfsizroq va optimizatsiyaga qulayroq qiladi.

Strict mode ikki darajada yoqiladi:
1. **Script darajasida** — fayl boshiga `"use strict";` yoziladi, butun fayl uchun amal qiladi
2. **Function darajasida** — funksiya tanasining birinchi qatoriga yoziladi, faqat shu funksiya uchun

ES6 **modullar** (`import`/`export`) va **class** tanasi avtomatik strict mode'da ishlaydi — alohida `"use strict"` yozish shart emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**`"use strict"` — directive prologue**:

`"use strict"` — bu maxsus syntactic construct emas, balki **string literal**. Engine uni alohida tushunadi faqat agar:
1. Script yoki function tanasining **birinchi statement**'i bo'lsa
2. Faqat string literal (ifoda emas — `"use" + "strict"` ishlamaydi)
3. **Directive prologue** ichida (eng yuqori positions)

```javascript
function strict() {
  "use strict";  // ✅ birinchi statement — directive
  // ...
}

function notStrict() {
  console.log("hello");
  "use strict"; // ❌ Birinchi emas — oddiy string, ignored
  // ...
}
```

**Engine perspektivasidan strict mode**:

V8 va boshqa engine'lar parsing paytida `"use strict"` directive'ini topib, qolgan kodni "strict mode parser" bilan parse qiladi. Bu degani:

1. **Parse-time errors**: Syntax xatolari (duplicate params, with statement) parser'da reject qilinadi
2. **Runtime semantics**: `this` binding, `delete`, `eval` xatti-harakatlari boshqacha
3. **Optimizatsiya imkoniyati**: Strict code'ni V8 tezroq optimize qila oladi, chunki xavfli operatsiyalar yo'q

**Modullarda implicit strict mode**:

ES6 modullar (`<script type="module">`, `.mjs` fayllar, `import`/`export`) **avtomatik strict mode**'da. Class tanalari ham. Bu spec qarori — modern JavaScript code default'da strict bo'lishi kerak.

```javascript
// module.mjs
function example() {
  undeclaredVar = 10; // ❌ ReferenceError — implicit strict
}

// regular.js  
function example() {
  undeclaredVar = 10; // ✅ ishlaydi — strict yo'q
}
```

**V8 da strict mode optimizatsiyasi**:

V8 strict functions'ni alohida flag bilan belgilaydi va quyidagi optimizatsiyalarni qila oladi:

1. **No `arguments.caller`**: caller chain saqlash kerak emas
2. **No `with`**: scope chain analysis statik
3. **No silent assign failures**: read-only property'lar darhol throw qiladi
4. **Better inlining**: this binding aniq

Bu optimizatsiyalar tufayli strict mode funksiyalar ko'pincha non-strict funksiyalarga nisbatan samaraliroq compile va execute bo'ladi — aniq tezlik farqi loyiha va workload'ga bog'liq.

**Engine uchun strict mode ning umumiy foydasi:**

1. **Scope optimizatsiyasi** — `with` va (strict) `eval` ning scope'ni buzish imkoniyati yo'q bo'lgani uchun engine compile-time da scope'ni to'liq aniqlay oladi → tezroq variable lookup
2. **TDZ enforcement** — `let`/`const` hoisting bilan ishlashda TDZ check'lari aniqroq ishlaydi
3. **`this` security** — tasodifan global object'ni o'zgartirish xavfi yo'qoladi — method'lar kutilgan kontekstda chaqirilishga majbur
4. **Hidden class stability (V8)** — `with` yo'qligi object shape'larni barqarorroq qiladi → inline caching samaraliroq ishlaydi

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Strict mode'da taqiqlangan va o'zgartirilgan xulq-atvorlar:

**1. Implicit global'lar taqiqlanadi:**

```javascript
"use strict";
undeclaredVar = 10;
// ❌ ReferenceError: undeclaredVar is not defined
// Non-strict'da bu global property yaratgan bo'lardi
```

**2. `this` → `undefined` (default binding da):**

Non-strict mode'da funksiya oddiy chaqirilganda (method emas) `this` global object'ga (`window`/`globalThis`) ishora qiladi. Strict mode'da bu `undefined` bo'ladi:

```javascript
"use strict";

function showThis() {
  console.log(this);
}

showThis(); // undefined
// Non-strict'da: globalThis (window) bo'lardi
// ✅ Bu xatolikdan himoya — tasodifan global object'ni o'zgartirmaslik uchun
```

**3. Duplicate parameter nomlari taqiqlanadi:**

```javascript
"use strict";

// ❌ SyntaxError: Duplicate parameter name not allowed
function sum(a, a) {
  return a + a;
}

// Non-strict'da bu ishlaydi — ikkinchi a birinchini yozib tashlaydi
```

**4. `with` statement taqiqlanadi:**

```javascript
"use strict";

const obj = { x: 10, y: 20 };
// ❌ SyntaxError: Strict mode code may not include a with statement
with (obj) {
  console.log(x + y);
}

// with scope chain'ni buzadi — engine compile-time da scope'ni
// aniqlay olmaydi, shuning uchun optimizatsiya imkonsiz bo'ladi
```

**5. `arguments.callee` taqiqlanadi:**

```javascript
"use strict";

function factorial(n) {
  if (n <= 1) return 1;
  // ❌ TypeError: 'caller', 'callee', and 'arguments' properties
  //    may not be accessed on strict mode functions
  return n * arguments.callee(n - 1);
}

// ✅ To'g'ri usul — named function expression ishlatish:
const factorial2 = function fact(n) {
  if (n <= 1) return 1;
  return n * fact(n - 1); // ✅ nom orqali recursive chaqiruv
};
```

**6. Octal literal syntax taqiqlanadi:**

```javascript
"use strict";

// ❌ SyntaxError: Octal literals are not allowed in strict mode
const num = 010; // non-strict'da 8 ga teng (octal)

// ✅ Agar octal kerak bo'lsa — ES6 prefix ishlatish:
const octal = 0o10; // 8 — aniq va tushunarli
```

**7. `delete` cheklovlari:**

```javascript
"use strict";

let x = 10;
// ❌ SyntaxError: Delete of an unqualified identifier in strict mode
delete x;

// Non-strict'da bu silently fail qiladi (false qaytaradi)
```

**8. `eval` cheklovlari:**

```javascript
"use strict";

eval("var x = 10");
// console.log(x);
// ❌ ReferenceError — strict mode'da eval o'z scope'ini yaratadi
// Non-strict'da x tashqi scope'ga chiqardi
```

**9. Production'da strict mode — modullar va class orqali avtomatik:**

Zamonaviy loyihalarda `"use strict"` yozish deyarli kerak emas — modullar va class tanalari avtomatik strict mode'da:

```javascript
// module.js — avtomatik strict mode
export function processOrder(order) {
  // "use strict" yozish shart emas — module ichida
  // quantity = order.qty; // ❌ ReferenceError — implicit global
  const quantity = order.qty; // ✅ To'g'ri
  return quantity * order.price;
}

// class ichida — avtomatik strict mode
class UserService {
  getUser() {
    // "use strict" yozish shart emas — class ichida
    return this; // strict mode qoidalari amal qiladi
  }
}
```

</details>

---

## Labeled Statements

### Nazariya

Label — identifier va colon (`:`) dan iborat bo'lib, loop yoki block'ga **nom berish** imkonini beradi. Labellar nested (ichma-ich) loop'larda `break` va `continue` ni **tashqi loop'ga** yo'naltirish uchun ishlatiladi. Oddiy `break` faqat eng ichki loop'ni to'xtatadi — labeled `break` esa istalgan tashqi loop'ni to'xtatishi mumkin.

```
Label sintaksisi:

labelName: statement

Loop label:          Block label:
outer: for (...)     myBlock: {
  inner: for (...)     // ... kod ...
    break outer;       break myBlock;  ← block'dan chiqish
                     }
```

Label'lar bilan ishlaydigan statement'lar:

1. **`break label`** — berilgan label'li loop yoki block'ni **to'liq to'xtatadi**
2. **`continue label`** — berilgan label'li loop'ning **keyingi iteration'iga** o'tadi (`continue` faqat loop'larda ishlaydi, block'larda emas)
3. **Block label** — `label: { ... break label; }` — block'dan erta chiqish uchun. Bu funksiya yaratmasdan "early exit" qilish imkonini beradi

```
Nested loop'da break vs break label:

outer: for (let i = 0; i < 3; i++) {
  inner: for (let j = 0; j < 3; j++) {
    break;          ← faqat inner loop to'xtaydi, outer davom etadi
    break inner;    ← bir xil natija — inner loop to'xtaydi
    break outer;    ← IKKALA loop ham to'xtaydi — tashqariga chiqadi
  }
}

continue bilan:
outer: for (let i = 0; i < 3; i++) {
  inner: for (let j = 0; j < 3; j++) {
    continue;       ← inner loop'ning keyingi iteratsiyasi
    continue outer; ← outer loop'ning keyingi iteratsiyasi (j loop to'xtaydi)
  }
}
```

**Nima uchun kam ishlatiladi?** Label'lar kod'ni o'qishni qiyinlashtiradi. Ko'pchilik dasturchilar buning o'rniga funksiya ajratib, `return` ishlatishni afzal ko'radi. Lekin ba'zi hollarda — masalan, murakkab matrix qidiruv yoki ko'p darajali ma'lumot qayta ishlashda — label performance va qulaylik jihatidan yaxshiroq bo'lishi mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

**Nested loop'da break outer — elementni topib, ikkala loop'ni to'xtatish:**

```javascript
const matrix = [
  [1,  2,  3,  4],
  [5,  6,  7,  8],
  [9, 10, 11, 12]
];

let targetRow = -1;
let targetCol = -1;
const target = 7;

// Label siz — flag o'zgaruvchi kerak bo'lardi
// let found = false;
// for (...) { for (...) { if (found) break; } if (found) break; }

// ✅ Label bilan — ancha toza
search: for (let i = 0; i < matrix.length; i++) {
  for (let j = 0; j < matrix[i].length; j++) {
    if (matrix[i][j] === target) {
      targetRow = i;
      targetCol = j;
      break search; // ← ikkala loop ham to'xtaydi!
    }
  }
}

console.log(`${target} topildi: [${targetRow}][${targetCol}]`);
// "7 topildi: [1][2]"
```

**Continue outer — tashqi loop'ning keyingi iteratsiyasiga o'tish:**

```javascript
// Har bir foydalanuvchining birinchi faol buyurtmasini topish
const users = [
  { name: "Ali",  orders: [{ id: 1, status: "cancelled" }, { id: 2, status: "active" }] },
  { name: "Vali", orders: [{ id: 3, status: "cancelled" }, { id: 4, status: "cancelled" }] },
  { name: "Gani", orders: [{ id: 5, status: "active" }] }
];

const firstActiveOrders = [];

userLoop: for (const user of users) {
  for (const order of user.orders) {
    if (order.status === "active") {
      firstActiveOrders.push({ user: user.name, orderId: order.id });
      continue userLoop; // ← bu user uchun topildi, keyingi user'ga o'tish
    }
  }
  // Bu yerga faqat hech qanday active order topilmagan user'lar uchun keladi
  console.log(`${user.name} — faol buyurtma yo'q`);
}

// Console: "Vali — faol buyurtma yo'q"
// firstActiveOrders: [{ user: "Ali", orderId: 2 }, { user: "Gani", orderId: 5 }]
```

**Block label — funksiyasiz early exit:**

```javascript
// Block label — murakkab shartli logika uchun
function processData(data) {
  let result = null;

  validation: {
    if (!data) {
      console.log("Data yo'q");
      break validation; // ← block'dan chiqish
    }
    if (!data.type) {
      console.log("Type yo'q");
      break validation;
    }
    if (!data.value) {
      console.log("Value yo'q");
      break validation;
    }
    // Hamma validation o'tdi
    result = `${data.type}: ${data.value}`;
  }

  return result; // validation'dan break qilinganda null qaytadi
}

processData(null);                    // "Data yo'q" → null
processData({ type: "name" });        // "Value yo'q" → null
processData({ type: "name", value: "Ali" }); // "name: Ali"
```

</details>

---

## Edge Cases va Gotchas

### `for` loop'dagi `let` — har iteratsiya yangi binding, `var` bitta

Bu klassik JavaScript gotcha: `for (let i = 0; ...)` ichida har bir iteratsiyada **yangi `i` binding** yaratiladi. Engine block'ni har iteratsiya boshida qayta yaratadi. `var` bilan esa faqat **bitta** binding bo'ladi — barcha iteratsiyalar shu bitta o'zgaruvchini share qiladi. Bu closure bilan ishlashda juda katta farq keltiradi.

```javascript
// let bilan — har iteratsiya yangi i
const letFns = [];
for (let i = 0; i < 3; i++) {
  letFns.push(() => i);
}
console.log(letFns.map(fn => fn())); // [0, 1, 2] — har closure o'z i'sini eslaydi

// var bilan — bitta i, loop tugaganda i = 3
const varFns = [];
for (var i = 0; i < 3; i++) {
  varFns.push(() => i);
}
console.log(varFns.map(fn => fn())); // [3, 3, 3] — hamma bitta i ga reference
```

**Yechim:** Zamonaviy kodda doim `let`/`const` ishlating. `for` loop'da `let` closure bilan doim to'g'ri ishlaydi.

---

### Named function expression nomi tashqarida ko'rinmaydi

`var fn = function myName() { ... }` — bu **named function expression**. `myName` identifikatori **faqat funksiya tanasi ichida** accessible (recursion yoki debugging uchun). Tashqaridan uni `myName()` deb chaqirib bo'lmaydi — faqat `fn()` orqali.

```javascript
const factorial = function fact(n) {
  if (n <= 1) return 1;
  return n * fact(n - 1); // ✅ fact funksiya ichida accessible
};

console.log(factorial(5)); // 120 ✅
// console.log(fact(5)); // ❌ ReferenceError — tashqarida yo'q
```

**Nima uchun:** Named function expression'ning nomi uning o'z scope'iga biriktirilgan — bu ichki scope faqat funksiya body'sini o'z ichiga oladi. Tashqi scope bu nomni ko'rmaydi. Bu spec qoida: `FunctionExpression` tanasi yangi Environment Record'ga ega bo'lib, unda faqat funksiya nomi binding sifatida saqlanadi.

**Foyda:** Recursion ichkaridan ishonchli ishlaydi (hatto tashqi o'zgaruvchi qayta belgilansa ham), va stack trace'da funksiya nomi ko'rinadi — debugging'ni osonlashtiradi.

---

### `try`/`catch` parameter faqat `catch` scope'da — alohida block scope

`catch (error)` ning `error` parametri o'ziga xos **block scope**'ga ega — faqat `catch` block ichidan accessible. Bu ichki block'da `error` nomini qayta ishlatish mumkin (shadow), lekin tashqi scope bilan to'qnashmaydi.

```javascript
try {
  throw new Error("outer error");
} catch (error) {
  console.log(error.message); // "outer error"

  try {
    throw new Error("inner error");
  } catch (error) {
    // ✅ Yangi catch — yangi error binding (shadow)
    console.log(error.message); // "inner error"
  }

  console.log(error.message); // "outer error" — tashqi catch tiklandi
}

// console.log(error); // ❌ ReferenceError — catch scope tugadi
```

**Yechim:** Oddatda nested catch'da `err` yoki `e` kabi boshqa nom ishlating — chalkashlik kamayadi. Yoki ikki darajali error handling kerak bo'lsa, alohida funksiyalar ajrating.

---

### `with` statement — strict mode'da parse-time SyntaxError

Non-strict mode'da `with` scope chain'ga object property'larni "qo'shadi" — bu engine optimization'larini butunlay buzadi. Strict mode'da esa `with` **parse-time SyntaxError**'ga olib keladi — hatto kod umuman run bo'lmaydi.

```javascript
// Non-strict — ishlaydi, lekin engine optimization'lar buziladi
const obj = { x: 10, y: 20 };
with (obj) {
  console.log(x + y); // 30 — scope chain'ga obj qo'shildi
}

// Strict mode — parse-time'da rad etiladi
"use strict";
with ({ x: 10 }) { // ❌ SyntaxError: Strict mode code may not include a with statement
  console.log(x);
}
```

**Nima uchun parse-time:** Oddiy runtime ReferenceError'dan farqli, `with` strict mode bilan mos kelmasligini engine **parsing bosqichida** aniqlaydi — script umuman yuklanmaydi. Bu sabab: modullar va class tanalari avtomatik strict mode'da bo'lgani uchun `with` zamonaviy JavaScript'da amalda taqiqlangan.

**Yechim:** `with` o'rniga destructuring yoki oddiy o'zgaruvchi ishlating: `const { x, y } = obj;`.

---

### Global `var` qayta yozilishi mumkin, global `let`/`const` esa — yo'q

Global scope'da `var` bilan e'lon qilingan o'zgaruvchi `globalThis` object'ga **writable property** sifatida qo'shiladi — uni `globalThis.name = newValue` orqali o'zgartirish mumkin. Global `let`/`const` esa declarative environment record'da saqlanadi va `globalThis`'ga qo'shilmaydi — uni tashqi kod `globalThis` orqali o'zgartira olmaydi.

```javascript
// Global scope
var varGlobal = "initial";
let letGlobal = "initial";

// Tashqi kod (masalan boshqa script):
globalThis.varGlobal = "hacked"; // ✅ var — writable property, o'zgardi
globalThis.letGlobal = "hacked"; // ✅ yangi property qo'shildi
                                  // LEKIN asl letGlobal o'zgarmadi

console.log(varGlobal); // "hacked" — asl binding o'zgardi
console.log(letGlobal); // "initial" — asl binding o'z joyida
console.log(globalThis.letGlobal); // "hacked" — yangi alohida property
```

**Nima uchun:** `var` `globalThis`'ga binding object orqali bog'liq — property o'zgarishi binding'ni o'zgartiradi. `let`/`const` esa declarative record'da alohida, `globalThis` property'lari bilan hech qanday aloqasi yo'q.

**Yechim:** Global state'ni har doim `const` bilan e'lon qiling (immutable reference) va modullar orqali ajratilgan namespace ishlating. Global `var` xavfli — uchinchi tomon kod uni o'zgartirishi mumkin.

---

## Common Mistakes

### ❌ Xato 1: Implicit Global O'zgaruvchilar

```javascript
function calculate() {
  result = 42; // ❌ let/const/var yo'q
  return result;
}

calculate();
console.log(result); // 42 — global scope iflos bo'ldi!
```

### ✅ To'g'ri usul:

```javascript
"use strict"; // yoki module ishlatish

function calculate() {
  const result = 42; // ✅ scope ichida qoladi
  return result;
}

calculate();
// console.log(result); // ❌ ReferenceError — to'g'ri xulq
```

**Nima uchun:** Non-strict mode'da assign qilingan lekin declare qilinmagan o'zgaruvchi global property bo'ladi. Bu xatolikni topish qiyin va boshqa kod bilan conflict yaratishi mumkin.

---

### ❌ Xato 2: `var` ning Block Scope'ni Tanimaslik Muammosi

```javascript
function processItems(items) {
  for (var i = 0; i < items.length; i++) {
    var item = items[i]; // ❌ var function scope'ga ko'tariladi
  }

  console.log(i);    // items.length — loop tugagandan keyin ham accessible
  console.log(item);  // oxirgi element — kutilmagan natija
}
```

### ✅ To'g'ri usul:

```javascript
function processItems(items) {
  for (let i = 0; i < items.length; i++) {
    const item = items[i]; // ✅ block scope ichida qoladi
  }

  // console.log(i);    // ❌ ReferenceError — to'g'ri!
  // console.log(item);  // ❌ ReferenceError — to'g'ri!
}
```

**Nima uchun:** `var` faqat function scope'ni tan oladi — `for` loop body block scope hisoblanmaydi. `let`/`const` ishlatish bu muammoni hal qiladi.

---

### ❌ Xato 3: Scope Chain ni Noto'g'ri Tushunish

```javascript
function outer() {
  const secret = "abc123";
}

function reader() {
  console.log(secret); // ❌ ReferenceError
  // reader() va outer() bir-birining ichida emas
  // scope chain: reader → global (outer scope emas!)
}
```

### ✅ To'g'ri usul:

```javascript
function outer() {
  const secret = "abc123";

  function reader() {
    console.log(secret); // ✅ "abc123"
    // reader() outer() ICHIDA define qilingan
    // scope chain: reader → outer → global
  }

  reader();
}
```

**Nima uchun:** Scope chain lexical — **yozilgan joyga** qarab aniqlanadi. Funksiyalar bir-birini chaqirishi scope chain'ga ta'sir qilmaydi, faqat bir-birining **ichida** define bo'lishi ta'sir qiladi.

---

### ❌ Xato 4: Loop + var + Closure Muammosi

```javascript
const buttons = document.querySelectorAll(".btn");

for (var i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", function () {
    console.log(`Button ${i} clicked`);
    // ❌ Doim "Button [buttons.length] clicked" chiqaradi
    // var bitta binding — loop tugaganda i = buttons.length
  });
}
```

### ✅ To'g'ri usul:

```javascript
const buttons = document.querySelectorAll(".btn");

for (let i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", function () {
    console.log(`Button ${i} clicked`);
    // ✅ Har iteratsiyada yangi i binding — to'g'ri raqam chiqaradi
  });
}
```

**Nima uchun:** `var` bitta binding yaratadi, barcha closure'lar shu bitta `i` ga reference saqlaydi. `let` har iteratsiyada yangi binding yaratadi — har bir closure o'z `i` qiymatiga ega.

---

### ❌ Xato 5: Shadow Qilingan O'zgaruvchini O'zgartirmoqchi Bo'lish

```javascript
let count = 0;

function increment() {
  let count = count + 1;
  // ❌ ReferenceError: Cannot access 'count' before initialization
  // Sabab: let count — shadowing yaratdi, yangi count TDZ da
  // "count + 1" dagi count — yangi (shadow) count, hali initialize bo'lmagan
  return count;
}
```

### ✅ To'g'ri usul:

```javascript
let count = 0;

function increment() {
  count = count + 1; // ✅ tashqi count'ni o'zgartiradi (shadow yo'q)
  return count;
}

// Yoki boshqa nom ishlatish:
function increment2() {
  let newCount = count + 1; // ✅ boshqa nom — shadowing muammosi yo'q
  return newCount;
}
```

**Nima uchun:** `let count = count + 1` da chap tomondagi `let count` shadowing yaratadi. O'ng tomondagi `count` endi yangi (local) `count` ga murojaat qiladi — lekin u hali TDZ da (initialize bo'lmagan).

---

## Amaliy Mashqlar

### Mashq 1: Scope Aniqlash (Oson)

**Savol:** Quyidagi kodning output'ini ayting va har bir natijani tushuntiring:

```javascript
let a = 1;

function outer() {
  let b = 2;

  function inner() {
    let c = 3;
    console.log(a, b, c);
  }

  inner();
  console.log(a, b);
}

outer();
console.log(a);
```

<details>
<summary>Javob</summary>

```javascript
// inner() ichida:
console.log(a, b, c); // 1, 2, 3
// a → inner → yo'q → outer → yo'q → global → 1 ✅
// b → inner → yo'q → outer → 2 ✅
// c → inner → 3 ✅

// outer() ichida (inner() tugagandan keyin):
console.log(a, b); // 1, 2
// a → outer → yo'q → global → 1 ✅
// b → outer → 2 ✅
// c → outer da yo'q — lekin console.log(c) ham yo'q

// global'da:
console.log(a); // 1
// a → global → 1 ✅
```

**Tushuntirish:** Scope chain ichdan tashqariga ishlaydi. `inner()` uchta scope'ni ko'radi (inner → outer → global), `outer()` ikkitasini (outer → global), global faqat o'zini.

</details>

---

### Mashq 2: Shadowing (O'rta)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
const x = "global";

function first() {
  const x = "first";

  function second() {
    console.log(x); // A
  }

  function third() {
    const x = "third";
    second();
    console.log(x); // B
  }

  third();
  console.log(x); // C
}

first();
console.log(x); // D
```

<details>
<summary>Javob</summary>

```
A: "first"
B: "third"
C: "first"
D: "global"
```

```javascript
// A: second() first() ichida define qilingan
//    second() scope chain: second → first → global
//    x → second'da yo'q → first'da topildi → "first"
//    DIQQAT: second() third() ichidan chaqirilgan bo'lsa ham,
//    lexical scope bo'yicha first() scope'iga qaraydi

// B: third() ichidagi const x = "third" — shadowing
//    console.log(x) → third scope'dagi x → "third"

// C: first() ichida const x = "first"
//    third() tugab bo'lgan, first() scope'ga qaytdik → "first"

// D: global scope'dagi const x = "global" → "global"
```

**Tushuntirish:** A — eng muhim qism. `second()` funksiyasi `third()` ichidan **chaqirilgan** bo'lsa ham, u `first()` ichida **define qilingan**. Lexical scope: chaqirilgan joy emas, yozilgan joy ahamiyatli.

</details>

---

### Mashq 3: var vs let vs Scope (O'rta)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
function test() {
  var a = 1;
  let b = 2;

  {
    var a = 3;
    let b = 4;
    console.log(a, b); // A
  }

  console.log(a, b); // B
}

test();
```

<details>
<summary>Javob</summary>

```
A: 3, 4
B: 3, 2
```

```javascript
// Block ichida:
// var a = 3 — var block scope tanimaydi, function scope'dagi a NI QAYTA YOZDI
// let b = 4 — let block scope'da yangi b yaratdi (shadowing)
// A: a = 3, b = 4

// Block tashqarisida:
// a = 3 — var a = 3 function scope'dagi a ni 3 ga o'zgartirgan edi
// b = 2 — block'dagi let b = 4 tugadi, function scope'dagi b = 2 qaytdi
// B: a = 3, b = 2
```

**Tushuntirish:** `var a = 3` yangi o'zgaruvchi yaratmadi — mavjud function scope'dagi `a` ni qayta yozdi. `let b = 4` esa yangi block scope binding yaratdi — function scope'dagi `b` ga tegmadi.

</details>

---

### Mashq 4: Scope Chain + Closure (Qiyin)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
function createCounter(initial) {
  let count = initial;

  return {
    increment() {
      count++;
      return count;
    },
    getCount() {
      return count;
    }
  };
}

const counter1 = createCounter(0);
const counter2 = createCounter(10);

console.log(counter1.increment()); // A
console.log(counter1.increment()); // B
console.log(counter2.increment()); // C
console.log(counter1.getCount());  // D
console.log(counter2.getCount());  // E
```

<details>
<summary>Javob</summary>

```
A: 1
B: 2
C: 11
D: 2
E: 11
```

```javascript
// createCounter(0) chaqirilganda:
//   Yangi scope yaratildi: { count: 0, initial: 0 }
//   increment va getCount shu scope'ga closure hosil qildi

// createCounter(10) chaqirilganda:
//   YANGI scope yaratildi: { count: 10, initial: 10 }
//   Boshqa increment va getCount — boshqa scope

// counter1.increment() → count: 0 → 1, return 1       (A)
// counter1.increment() → count: 1 → 2, return 2       (B)
// counter2.increment() → count: 10 → 11, return 11    (C)
// counter1.getCount()  → count = 2                     (D)
// counter2.getCount()  → count = 11                    (E)
```

**Tushuntirish:** Har bir `createCounter()` chaqiruvi yangi scope (Environment Record) yaratadi. `counter1` va `counter2` turli scope'larga closure hosil qilgan — ular bir-biriga ta'sir qilmaydi. Bu scope + closure pattern'ining asosi — data privacy.

</details>

---

### Mashq 5: Strict Mode Muammolari (Qiyin)

**Savol:** Quyidagi kodda qaysi qatorlar xato beradi va nima uchun?

```javascript
"use strict";

function processData(data, data) {  // Qator A
  result = data * 2;                // Qator B
  var undefined = 5;                // Qator C
  let NaN = 10;                     // Qator D
  delete data;                      // Qator E
  return result;
}
```

<details>
<summary>Javob</summary>

```javascript
// Qator A: ❌ SyntaxError — strict mode'da duplicate parameter taqiq
// Qator B: ❌ ReferenceError — strict mode'da implicit global taqiq (let/const/var yo'q)
// Qator C: 🟡 Texnik jihatdan ishlaydi — `undefined` reserved word EMAS, u global
//          property. `var undefined = 5` local variable yaratadi va global
//          undefined'ni shadow qiladi. Lekin bu juda yomon amaliyot va reader'ni
//          chalkashtiradi — hech qachon qilmang.
// Qator D: 🟡 Texnik jihatdan ishlaydi — `NaN` ham reserved word emas, global
//          property. `let NaN = 10` local shadow yaratadi. Lekin ham yomon amaliyot.
// Qator E: ❌ SyntaxError — strict mode'da identifier'ni delete qilish taqiq
```

**Tushuntirish:** Strict mode bir nechta xavfli pattern'larni taqiqlaydi. **Haqiqiy xatolar** (A, B, E) — duplicate parameters, implicit globals va identifier `delete`. Bular compile-time da aniqlanadi. **Texnik jihatdan ishlaydigan lekin yomon amaliyot** (C, D) — `undefined` va `NaN` reserved word emas, shuning uchun ularni shadow qilish mumkin, lekin hech qachon bunday qilmaslik kerak — kod chalkash va xavfli bo'ladi.

**Amalda:** Agar bu kodni ishga tushirsangiz, parser Qator A'da birinchi SyntaxError'ni topib to'xtaydi — script umuman run bo'lmaydi. SyntaxError'lar parse-time'da aniqlanadi, shuning uchun B va E gacha hech qachon yetmaydi.

</details>

---

## Xulosa

Bu bo'limda scope tizimining barcha qatlamlari yoritildi:

- **Scope** — o'zgaruvchilarning accessibility chegarasi. Uchta asosiy turi: global, function, block.
- **Global Scope** — `var`/`function` → `globalThis` property, `let`/`const` → faqat identifier orqali.
- **Function Scope** — `var` faqat function chegarasini tan oladi, block'ni emas.
- **Block Scope** — `let`/`const` `{}` ichida qoladi. `for` loop'da har iteratsiya yangi binding.
- **Lexical Scope** — scope kod **yozilgan** joyga qarab aniqlanadi, **chaqirilgan** joyga emas.
- **Scope Chain** — `[[OuterEnv]]` reference'lar zanjiri, ichdan tashqariga qidirish.
- **Variable Shadowing** — ichki scope tashqi scope'dagi bir xil nomli o'zgaruvchini yashiradi.
- **Variable Lookup** — ResolveBinding algoritmi, scope chain bo'ylab recursive qidirish.
- **Strict Mode** — implicit globals, duplicate params, `with`, `arguments.callee` taqiqlaydi; `this` → `undefined`.

Scope tushunchasi keyingi bo'limdagi **closure** ning asosi — closure scope chain'ning "tirik qolishi" natijasida hosil bo'ladi.

---

**Keyingi bo'lim:** [05-closures.md](05-closures.md) — Closures chuqur tushuncha: lexical environment va closure aloqasi, `[[Environment]]` internal slot, closure hosil bo'lish jarayoni qadam-baqadam, use cases (data privacy, factory functions, memoization), memory va closures, klassik loop + closure muammosi.
