# Execution Context — Interview Savollari

> Execution Context ichki mexanizmlari bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.

---

## Nazariy savollar

### 1. Execution Context nima va unda qanday komponentlar bor? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Execution Context — JavaScript engine har bir kod bo'lagini bajarishdan oldin yaratiladigan ichki muhit. Spec'da EC'ning state component'lari quyidagilar:

1. **LexicalEnvironment** — `let`, `const`, `class` declaration'lar saqlanadigan Environment Record'ga pointer
2. **VariableEnvironment** — `var` va `function` declaration'lar saqlanadigan Environment Record'ga pointer
3. **PrivateEnvironment** — class'ning `#private` field'lari (PrivateEnvironment Record), class tashqarisida `null`

`this` qiymati alohida EC component emas — u eng yaqin Function/Global Environment Record'ning `[[ThisValue]]`/`[[GlobalThisValue]]` slot'ida saqlanadi va `ResolveThisBinding` orqali olinadi.

Har bir EC ikki bosqichda yaratiladi:
- **Creation Phase** — muhit tayyorlanadi, o'zgaruvchilar e'lon qilinadi, lekin kod bajarilmaydi
- **Execution Phase** — kod qator-baqatar bajariladi

```javascript
// Creation Phase natijalari (har qator alohida — birinchi error throw qilsa keyingilari bajarilmaydi):
console.log(x);     // undefined — var Creation Phase'da undefined bilan initialize
console.log(greet); // [Function: greet] — function declaration to'liq hoist
console.log(y);     // ReferenceError — let TDZ'da (uninitialized)

var x = 10;
let y = 20;
function greet() { return "Hi"; }
```

Bu mexanizm hoisting'ning asosi — `var` Creation Phase da `undefined` bilan initialize qilinadi, shuning uchun e'lon qilishdan oldin o'qish mumkin (lekin qiymati `undefined`).

</details>

### 2. Execution Context necha xil bo'ladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Uch xil:

| Tur | Qachon yaratiladi | Nechta bo'ladi |
|-----|-------------------|----------------|
| **Global EC** | Dastur boshlanganda | Faqat 1 ta |
| **Function EC** | Funksiya chaqirilganda | Har chaqiruvda yangi |
| **Eval EC** | `eval()` chaqirilganda | Har chaqiruvda yangi |

```javascript
// Global EC — dastur boshlanganda yaratiladi
var x = 10; // GEC ichida

function calculate(a, b) {
  // calculate() chaqirilganda yangi FEC yaratiladi
  var result = a + b; // FEC ichida
  return result;
}

calculate(1, 2);  // FEC #1 yaratildi → tugadi → yo'qoldi
calculate(3, 4);  // FEC #2 yaratildi → tugadi → yo'qoldi
// ✅ Har bir chaqiruv YANGI FEC — oldingi EC bilan aloqasi yo'q
```

Barcha EC'lar **Execution Context Stack** (Call Stack) da boshqariladi — LIFO tartibida.

</details>

### 3. Creation Phase va Execution Phase da nima sodir bo'ladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

**Creation Phase** — kod bajarilmaydi, faqat muhit tayyorlanadi:
- `var` declaration'lar topiladi → `undefined` bilan initialize
- `let`/`const` topiladi → `uninitialized` (TDZ boshlanadi)
- function declaration'lar topiladi → to'liq funksiya bilan initialize
- `this` binding aniqlanadi
- Scope chain quriladi (outer reference)

**Execution Phase** — kod qator-baqatar bajariladi:
- O'zgaruvchilarga haqiqiy qiymatlar beriladi
- Funksiyalar chaqiriladi (yangi FEC yaratiladi)
- Expression'lar evaluate qilinadi

```javascript
// Amaliy misol — Creation va Execution:
var a = 1;
let b = 2;
function sum() { return a + b; }
var result = sum();

// CREATION PHASE:
// a: undefined (var)
// b: <uninitialized> (let — TDZ)
// sum: function sum() {...} (to'liq hoist)
// result: undefined (var)

// EXECUTION PHASE:
// a = 1
// b = 2 (TDZ tugadi)
// result = sum() → sum FEC yaratiladi → 3 qaytaradi → result = 3
```

**Deep Dive:**

Muhim farq: **Parsing** va **Creation Phase** — bular ikki alohida bosqich:
- **Parsing** (engine source code'dan AST yaratishda) — declaration'lar static ravishda topiladi va saqlanadi (qaysi scope'da qaysi `var`/`let`/`const`/`function` bor). Bu faqat bir marta bajariladi.
- **Creation Phase** (har EC yaratilganda) — parser'ning static ma'lumotlari asosida Environment Record'lar **runtime'da** to'ldiriladi: `var` uchun `CreateMutableBinding` + `InitializeBinding(undefined)` chaqiriladi; `let` uchun `CreateMutableBinding` (uninitialized — TDZ); `const` uchun `CreateImmutableBinding` (uninitialized — TDZ). Initialize keyinroq, Execution Phase'da declaration'ga yetganda bo'ladi.

Shuning uchun Creation Phase tez ishlaydi — engine source'ni qayta scan qilmaydi, faqat parser'ning metadata'sidan EC environment'ni yaratadi.

</details>

### 4. Variable Environment va Lexical Environment farqi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

ES6 dan boshlab har bir EC'da ikkita alohida environment bor:

| Xususiyat | VariableEnvironment | LexicalEnvironment |
|-----------|--------------------|--------------------|
| Nima saqlanadi | `var`, `function` | `let`, `const`, `class` |
| Scope | Function scope | Block scope |
| Hoisting | `undefined` bilan (`function` — to'liq) | `uninitialized` (TDZ) |
| Block ichida | Block yangi VE yaratMAYDI | Block yangi LE yaratadi |

```javascript
function example() {
  var x = 1;    // VariableEnvironment (function scope)
  let y = 2;    // LexicalEnvironment (function scope)

  if (true) {
    var z = 3;  // VariableEnvironment (function scope! — block emas)
    let w = 4;  // Yangi block LexicalEnvironment
    const v = 5; // Yangi block LexicalEnvironment
  }

  console.log(x); // 1 ✅
  console.log(y); // 2 ✅
  console.log(z); // 3 ✅ — var function-scoped
  console.log(w); // ❌ ReferenceError — let block-scoped
}
```

Nima uchun ikki alohida environment kerak: `var` (ES5 dan) function-scoped. ES6 da `let`/`const` kiritilganda block scoping kerak bo'ldi. Alohida environment'lar tufayli engine block ichida yangi LexicalEnvironment yaratib block scope'ni ta'minlaydi — VariableEnvironment'ga ta'sir qilmaydi.

</details>

### 5. Environment Record turlari va farqlari nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

ECMAScript spec'da Environment Record — scope'dagi barcha binding'lar saqlanadigan tuzilma. Spec da **5 turi** bor (ko'p manbalar faqat 3 ta aytadi, bu chala):

**1. Declarative Environment Record:**
- `let`, `const`, `function`, `class`, `catch` parameter, va **function scope'dagi `var`**'lar saqlanadi
- Block va function scope'larda ishlatiladi
- O'zgaruvchilar to'g'ridan-to'g'ri record ichida — **tez** (index-based access)
- Eslatma: **global scope'dagi `var`** esa Object ER'ga ketadi (pastda)

**2. Function Environment Record** (Declarative ER'ning kengaytmasi):
- Declarative ER'ning barcha funksiyasi +
- `[[ThisValue]]` — funksiya `this` binding'i
- `[[NewTarget]]` — `new.target` qiymati
- `[[HomeObject]]` — `super` resolution uchun method'ning uy-object'i
- `[[FunctionObject]]` — joriy funksiyaning o'ziga reference
- Har function chaqiruvi yangi Function ER yaratadi

**3. Module Environment Record** (ES6+ ES Modules uchun):
- Declarative ER'ning kengaytmasi
- `import` qilingan binding'lar **immutable** va **indirect** (original module'dagi binding'ga reference, nusxa emas)
- Import value'lari exporting module'da o'zgartirilsa — importing module ham yangilanadi (live bindings)

**4. Object Environment Record:**
- Biror object'ning property'larini binding sifatida ko'rsatadi
- Global scope'da `var` va function declaration'lar Object ER'ga ketadi (`globalThis`/`window` orqali)
- `with` statement uchun ishlatiladi (faqat shu ikki holat)

**5. Global Environment Record** (maxsus — composite):
- Object ER qismi: `var`, function declaration'lar → `globalThis` property bo'ladi
- Declarative ER qismi: `let`, `const`, `class` → alohida, `globalThis`'da yo'q

```javascript
// Global scope
var name = "Ali";       // Global ER → Object ER qismi → window.name = "Ali"
let age = 25;           // Global ER → Declarative ER qismi (window.age yo'q)

console.log(window.name); // "Ali" ✅
console.log(window.age);  // undefined ❌ — Declarative ER'da, window'da emas
console.log(age);          // 25 ✅ — scope resolve orqali topiladi

// Function scope — var bu yerda Declarative ER'ga ketadi!
function example() {
  var x = 1;    // Function ER (Declarative ER kengaytmasi) — window.x YO'Q
  console.log(globalThis.x); // undefined ✅
}

// ES Module scope
import { counter } from "./module.js";
// counter — Module Environment Record'da, immutable indirect binding
// counter = 10; // ❌ early SyntaxError (parse vaqtida, kod ishga tushishidan oldin)
//                  V8 message: "Assignment to constant variable."
```

**Edge Cases:**
- **`with` statement**: Object ER yaratiladi va scope chain'ga qo'shiladi — strict mode'da taqiq, lookup juda sekin (dynamic property check har binding uchun)
- **`catch` parameter**: O'ziga xos Declarative ER yaratiladi — `try { ... } catch(e) { ... }` ichida `e` binding faqat shu blok'da mavjud
- **`import.meta`**: Module Environment Record'da emas, Source Text Module Record'ning `[[ImportMeta]]` field'ida saqlanadi — birinchi murojaatda yaratiladi (`module.[[ImportMeta]]` undefined bo'lsa, `HostGetImportMetaProperties` bilan to'ldiriladi), keyin cache qilinadi. Har module uchun unique metadata (URL, va boshqa)

**Follow-up savollar:**
1. **Global scope'da `var foo` va `globalThis.foo = ...` farqi nima?** — `var foo` Global ER'ning Object ER qismida configurable: false bilan property yaratadi (delete bilan o'chmaydi). `globalThis.foo = ...` configurable: true bilan oddiy property (delete mumkin).
2. **Module ER'da `var` ishlatilsa nima bo'ladi?** — Module top-level'dagi `var` Module ER'ga ketadi (Object ER emas) — `globalThis.varName` da yo'q.

<details>
<summary><strong>Deep Dive</strong></summary>

Declarative ER tezroq ishlashining sababi: engine compile vaqtida barcha binding'larning index'ini biladi — hash table lookup kerak emas, direct slot access ishlaydi. Object ER esa dynamic property lookup qiladi (window object orqali, `[[Get]]` internal method) — sekinroq va ko'p JIT optimization'lar uchun to'siq (inline caching murakkab).

Module ER'ning "live binding" xususiyati spec'dagi `GetBindingValue` operatsiyasi orqali implement qilingan: har safar original exporting module'dan qiymatni `ResolveBinding` orqali oladi. Shuning uchun circular import'larda `import` qiymati keyinroq ham yangilanishi mumkin — bu CommonJS'dagi static snapshot semantikasidan farq qiluvchi xususiyat.

V8 implementation: Environment Record'lar `Context` object sifatida heap'da saqlanadi. Har Context'da `previous` slot — bu `[[OuterEnv]]`. Escape analysis bilan agar variable'lar inner funksiyalar tomonidan capture qilinmagan bo'lsa — Context allocation skip qilinadi, register/stack'da saqlanadi.

</details>

</details>

### 6. Global scope'da var bilan e'lon qilingan o'zgaruvchi let dan nima farq qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

| Farq | `var` (global) | `let` (global) |
|------|---------------|----------------|
| Environment Record | Object ER (window) | Declarative ER |
| `window` property | Ha ✅ | Yo'q ❌ |
| Hoisting | `undefined` bilan | TDZ (uninitialized) |
| Qayta e'lon | Mumkin | ❌ SyntaxError |
| Block scope | Yo'q (function-scoped) | Ha ✅ |

```javascript
var x = 1;
let y = 2;

// window property
console.log(window.x);        // 1 ✅
console.log(window.y);        // undefined ❌

// Qayta e'lon
var x = 10;                   // ✅ xato yo'q
// let y = 20;                // ❌ SyntaxError: Identifier 'y' has already been declared

// Hoisting
console.log(a);               // undefined
console.log(b);               // ReferenceError: Cannot access 'b' before initialization
var a = 1;
let b = 2;

// delete
delete window.x;              // ❌ false qaytaradi (configurable: false)
// var ham let ham global scope'da delete bilan o'chirilmaydi
// Faqat window.x = 1 (var'siz) qilingan property delete bo'ladi
```

Bu farqning sababi: `var` va `let` turli Environment Record'larda saqlanadi. `var` → Object ER (window object orqali), `let` → Declarative ER (alohida, window'dan mustaqil).

</details>

### 7. Kod baholash operatsiyasi nima uchun ishlatmaslik kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`eval()` string sifatida berilgan JavaScript kodni parse va execute qiladi. Uni ishlatmaslik kerak chunki:

**1. Xavfsizlik:** Tashqi source'dan kelgan string'ni baholash code injection imkonini beradi:

```javascript
// ❌ Xavfli — foydalanuvchi input'ini baholash
const userInput = "alert('hacked')";
window["ev"+"al"](userInput); // ixtiyoriy kod bajariladi — XSS, code injection
```

**2. Performance:** Non-trivial kod baholash mavjud scope'dagi ko'p optimization'larni bekor qiladi:

```javascript
function optimized() {
  var x = 10;
  // V8 x ni register'da saqlashi mumkin (tez)
  return x + 1;
}

function notOptimized() {
  var x = 10;
  // ❌ scope escape — x heap'ga
  // V8 x ni heap'da saqlashga majbur — chunki baholash x ni o'qishi/o'zgartirishi MUMKIN
  return x + 1;
}
```

V8 ayrim hollarda **statik bo'sh** kod baholashni specially handle qiladi (constant folding orqali olib tashlanishi mumkin), lekin har qanday non-trivial yoki dinamik baholash scope escape trigger qiladi va compiler lokal o'zgaruvchilarni context (heap) ga ko'chirishga majbur bo'ladi.

**3. Direct vs Indirect — spec farqi va scope muammolari:**

Spec'da kod baholash ikki xil chaqiriladi, ular turli semantika beradi. Bu farq ko'pchilik developer'ga noma'lum va interview'larda muhim:

```javascript
// DIRECT — to'g'ridan-to'g'ri identifier orqali
function direct() {
  const e = window["ev"+"al"]; // identifier reference
  e("var x = 10");             // calling scope'ga inject — local x yaratadi
}

// INDIRECT — boshqa expression yoki alias orqali
function indirect() {
  const alias = window["ev"+"al"];
  alias("var x = 10");         // global scope'da bajariladi — local x emas
  (0, window["ev"+"al"])("..."); // comma operator ham indirect
}
```

**Spec farqi:** Direct chaqiruv calling scope context'ida bajariladi. Indirect esa har doim **global scope context'ida** ishlaydi.

**Amaliy ta'siri:**
- Kutubxonalar qasddan indirect variant ishlatadi ("safe" global code evaluation uchun)
- Direct variant **scope pollution** va **optimization destruction** ni yaratadi
- Indirect variant faqat **global injection** xavfi — lekin lokal scope xavfsiz qoladi

**O'rniga ishlatish kerak:** `JSON.parse()` (ma'lumot uchun), yoki dizayn pattern'lar (strategy, command map, factory) — bular statik kod bilan ishlaydi va hech qanday runtime parsing/execution kerak emas.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle]

```javascript
var x = 1;

function showValue() {
  console.log(x);
  var x = 2;
  console.log(x);
}

showValue();
console.log(x);
```

<details>
<summary><strong>Javob</strong></summary>

```
undefined
2
1
```

Qadam-baqadam:

```
showValue() FEC Creation Phase:
  VariableEnvironment: x = undefined (var hoist — local x!)

showValue() Execution Phase:
  console.log(x)  → undefined (local x hali undefined)
  x = 2            → local x endi 2
  console.log(x)  → 2

Global:
  console.log(x)  → 1 (global x o'zgarmagan)
```

Asosiy nuqta: `showValue()` ichida `var x = 2` bor — bu **local** `x` yaratadi. Creation Phase da local `x` = `undefined` bo'ladi. Shuning uchun birinchi `console.log(x)` local `x` ni topadi (undefined), global `x` (1) ni emas. Bu **variable shadowing** — local scope global scope'ni "yashiradi".

</details>

### 2. Quyidagi kodning output'ini ayting [Middle+]

```javascript
console.log(greeting);
console.log(format);

var greeting = "Hello";
function greeting() { return "World"; }
var format = function() { return "!"; };

console.log(greeting);
console.log(format);
```

<details>
<summary><strong>Javob</strong></summary>

```
[Function: greeting]
undefined
Hello
[Function (anonymous)]
```

> **Eslatma:** Node.js/brauzer console'da function qiymatlar `[Function: name]` yoki `[Function (anonymous)]` sifatida ko'rsatiladi, to'liq tanasi bilan emas. Quyida real bajarilish tartibi tushuntiriladi.

Creation Phase:
1. `var greeting` → `undefined`
2. `function greeting()` → **function declaration function bilan override qiladi** → `greeting = function greeting(){...}`
3. `var format` → `undefined`

Function declaration `var` dan yuqori priority'ga ega — ikkalasi bir xil nomda bo'lsa, function declaration yutadi.

Execution Phase:
1. `console.log(greeting)` → `function greeting(){...}` (Creation Phase natijasi)
2. `console.log(format)` → `undefined` (var hoist)
3. `greeting = "Hello"` → endi greeting string
4. `format = function(){...}` → endi format function
5. `console.log(greeting)` → `"Hello"`
6. `console.log(format)` → `function(){ return "!"; }`

</details>

### 3. Quyidagi kodning output'ini ayting [Middle+]

```javascript
var a = 1;

function outer() {
  var a = 2;

  function inner() {
    console.log(a);
  }

  return inner;
}

var fn = outer();

function anotherScope() {
  var a = 3;
  fn();
}

anotherScope();
```

<details>
<summary><strong>Javob</strong></summary>

```
2
```

`inner()` funksiyasi `outer()` **ichida yozilgan** — shuning uchun uning outer reference `outer()`ning environment'iga bog'langan. `inner()` `anotherScope()` ichida **chaqirilgan** bo'lsa ham — JavaScript lexical scoping ishlatadi, dynamic emas.

Scope chain: `inner() → outer() → global`

`inner()` ichida `a` qidiriladi:
1. `inner()` scope'ida → yo'q
2. `outer()` scope'ida → `a = 2` ✅ topildi!

`anotherScope()` dagi `a = 3` ga hech qachon borilmaydi — chunki `inner()` `outer()` da yozilgan, `anotherScope()` da emas.

Bu **closure** mexanizmi — `inner()` `outer()` tugagandan keyin ham uning scope'iga kira oladi. Spec'da bu function object'ning `[[Environment]]` internal slot orqali implement qilingan: funksiya yaratilganda (`OrdinaryFunctionCreate`) joriy LexicalEnvironment shu slot'ga saqlanadi, va chaqirilganda yangi FEC'ning `[[OuterEnv]]` qiymati shu slot'dan olinadi. Shuning uchun outer function'ning Environment Record garbage collect bo'lmaydi — inner function reference'i ushlab turadi.

</details>

### 4. Quyidagi kodda EC Stack qanday o'zgaradi? Output nima? [Middle]

```javascript
function a() {
  console.log("a");
  b();
}

function b() {
  console.log("b");
}

function c() {
  console.log("c start");
  a();
  console.log("c end");
}

c();
```

<details>
<summary><strong>Javob</strong></summary>

```
c start
a
b
c end
```

EC Stack holati:

```
1. c() chaqirildi     → Stack: [GEC, c]           → "c start"
2. a() chaqirildi     → Stack: [GEC, c, a]        → "a"
3. b() chaqirildi     → Stack: [GEC, c, a, b]     → "b"
4. b() tugadi         → Stack: [GEC, c, a]
5. a() tugadi         → Stack: [GEC, c]            → "c end"
6. c() tugadi         → Stack: [GEC]
```

Muhim nuqta: `a()` ichidagi `b()` tugamaguncha `c()` `console.log("c end")` ga yeta olmaydi. JavaScript single-threaded — bir vaqtda faqat stack tepasidagi EC bajariladi.

</details>

### 5. Bu kodda xato bor. Toping va tuzating [Middle]

```javascript
function getUserRole(userId) {
  if (userId === 1) {
    var role = "admin";
    var permissions = ["read", "write", "delete"];
  } else {
    var role = "user";
    var permissions = ["read"];
  }

  return { role, permissions };
}
```

<details>
<summary><strong>Javob</strong></summary>

Kod ishlaydi, lekin **potentsial xato** bor: `var` function-scoped bo'lgani uchun if/else tashqarisida ham accessible. Hozir ishlaydi, lekin bu xavfli pattern:

1. `var role` ikki marta e'lon qilingan — birinchi e'lon ikkinchisi tomonidan "yashiriladi"
2. Agar kelajakda if/else shartini o'zgartirsangiz — `role` va `permissions` `undefined` bo'lib qolishi mumkin (agar hech bir branch bajarilmasa)

```javascript
// ✅ Tuzatilgan — let bilan block scope
function getUserRole(userId) {
  let role;
  let permissions;

  if (userId === 1) {
    role = "admin";
    permissions = ["read", "write", "delete"];
  } else {
    role = "user";
    permissions = ["read"];
  }

  return { role, permissions };
}

// ✅ Yoki yanada yaxshi — early return pattern
function getUserRole(userId) {
  if (userId === 1) {
    return { role: "admin", permissions: ["read", "write", "delete"] };
  }
  return { role: "user", permissions: ["read"] };
}
```

`var` bilan if ichida e'lon qilish — bu niyatni noaniq qiladi. O'quvchi "bu faqat if ichida ishlaydi" deb o'ylashi mumkin, aslida function scope bo'ylab accessible. `let` bilan yozish niyatni aniq ko'rsatadi.

</details>

### 6. Quyidagi kodning output'ini ayting [Senior]

```javascript
var x = 1;

function outer() {
  console.log(x);    // ?
  var x = 2;

  function inner() {
    console.log(x);  // ?
    var x = 3;
    console.log(x);  // ?
  }

  inner();
  console.log(x);    // ?
}

outer();
console.log(x);      // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
undefined
undefined
3
2
1
```

Qadam-baqadam:

```
outer() Creation Phase:
  var x = undefined (local x — global x ni shadow qiladi)
  function inner = <func>

outer() Execution Phase:
  console.log(x) → undefined (local x hali undefined)
  x = 2

  inner() Creation Phase:
    var x = undefined (inner'ning local x'i — outer x ni shadow qiladi)

  inner() Execution Phase:
    console.log(x) → undefined (inner'ning local x'i hali undefined)
    x = 3
    console.log(x) → 3 (inner'ning local x'i)

  inner() tugadi
  console.log(x) → 2 (outer'ning x'i — inner ning x'i aralashmaydi)

outer() tugadi
console.log(x) → 1 (global x — hech o'zgarmagan)
```

Har bir funksiyada `var x` bor — har biri **alohida** local `x` yaratadi. Inner'ning `x` i outer'ning `x` iga ta'sir qilmaydi, outer'niki global'ga ta'sir qilmaydi. Bu **variable shadowing** — ichki scope'dagi o'zgaruvchi tashqisini "yashiradi".

**Deep Dive:**

ECMAScript spec'da har bir `var` declaration VariableEnvironment'ning Environment Record'iga `CreateMutableBinding` va `InitializeBinding(undefined)` orqali yoziladi. Identifier resolution esa `ResolveBinding` abstract operation orqali — joriy LexicalEnvironment'dan boshlab, `[[OuterEnv]]` chain bo'ylab `null` gacha qidiriladi. Shuning uchun inner scope'dagi binding topilsa — outer scope'ga bormaydi.

</details>

### 7. Outer Environment Reference va Scope Chain qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob
Har bir LexicalEnvironment'da `[[OuterEnv]]` slot mavjud — bu lexical parent scope'ning Environment Record'iga havola. Scope Chain shu havolalar zanjirini bo'ylab quriladi va identifier resolution shu zanjir orqali ishlaydi.

### To'liq tushuntirish

`[[OuterEnv]]` reference funksiya **yozilgan** (chaqirilgan emas) joydagi scope'ga ko'rsatadi — bu **lexical scoping**'ning asosi. Engine identifier'ni qidirganda quyidagi tartibni qo'llaydi:

1. Joriy Environment Record'da binding mavjudligini tekshiradi (`HasBinding`)
2. Topilmasa → `[[OuterEnv]]` bo'ylab parent scope'ga o'tadi
3. Tashqida ham topilmasa → yana yuqori scope'ga
4. Global scope'da ham topilmasa → `ReferenceError`

Outer reference funksiya yaratilganda (Creation Phase'da emas, funksiya object yaratilganda) o'rnatiladi va keyin o'zgarmaydi.

### Kod misol

```javascript
const company = "TechCorp";

function outer() {
  const department = "Engineering";

  function inner() {
    const role = "Developer";
    console.log(role);        // o'z scope'i: "Developer"
    console.log(department);  // outer ref: "Engineering"
    console.log(company);     // outer ref → global: "TechCorp"
  }

  inner();
}

outer();
```

Scope chain quyidagicha quriladi:

```
inner Environment Record:
  role: "Developer"
  [[OuterEnv]] ─→ outer Environment Record:
                    department: "Engineering"
                    [[OuterEnv]] ─→ Global Environment Record:
                                      company: "TechCorp"
                                      [[OuterEnv]]: null
```

### Edge Cases

- **Lexical vs Dynamic scoping**: JavaScript lexical ishlatadi — funksiya **yozilgan** joyga qarab `[[OuterEnv]]` o'rnatiladi:
  ```javascript
  const value = "global";
  function show() { console.log(value); }
  function wrapper() {
    const value = "wrapper";
    show(); // "global" — show global'da yozilgan, wrapper'da emas
  }
  wrapper();
  ```
- **Closure** — funksiya tugagandan keyin ham `[[OuterEnv]]` saqlanadi: nested funksiya reference saqlasa, parent EC'ning Environment Record heap'da yashashda davom etadi.
- **`with` statement** — Object Environment Record'ni scope chain'ga qo'shadi (strict mode'da taqiq). Dynamic property lookup tufayli ko'p optimization'larni buzadi.

### Follow-up savollar

1. "OuterEnv qachon o'rnatiladi?" — Funksiya object yaratilganda (declaration/expression evaluate qilinganda). Funksiya `[[Environment]]` internal slot'iga joriy LexicalEnvironment yoziladi. Har chaqiruvda yangi FEC'ning `[[OuterEnv]]`'i shu slot'dan olinadi.
2. "Scope chain qancha chuqur bo'lishi mumkin?" — Texnik chegara yo'q, lekin har qatlam lookup vaqtini oshiradi (linear search). Chuqur nested funksiyalar V8 da optimization uchun "context allocation" qiladi — faqat ishlatiladigan variable'lar parent context'da saqlanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec'da `ResolveBinding(name, env)` abstract operation:
1. `env` argument berilmagan bo'lsa — joriy `LexicalEnvironment` olinadi
2. `GetIdentifierReference(env, name, strict)` chaqiriladi
3. Ichida: `env.HasBinding(name)` → true bo'lsa Reference qaytariladi, false bo'lsa `env.OuterEnv` bilan rekursiv chaqiriladi
4. `env === null` bo'lsa — Reference (base=unresolvable) qaytariladi → `GetValue` `ReferenceError` throw qiladi

V8 implementation: har Environment Record `Context` object sifatida saqlanadi (heap'da). Har Context'da `previous` slot — bu `[[OuterEnv]]`. JIT optimization: V8 escape analysis bilan agar variable'lar inner funksiyalar tomonidan capture qilinmagan bo'lsa — Context allocation umuman qilinmaydi, register/stack'da saqlanadi. Bu "context-allocated" vs "stack-allocated" variable farqi `--print-bytecode` flag bilan ko'rinadi.

</details>

</details>

### 8. Execution Context'da `this` binding qanday aniqlanadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob
`this` qiymati EC yaratilganda **funksiya qanday chaqirilganiga** qarab aniqlanadi (arrow function bundan mustasno — u outer scope'dan oladi). Global EC'da `this` — `globalThis` (module'da `undefined`).

### To'liq tushuntirish

`this` runtime'da chaqiruv usuli (call site) bo'yicha aniqlanadi:

| Chaqiruv usuli | `this` qiymati |
|----------------|---------------|
| Global scope (script) | `globalThis` (window/global) |
| Global scope (module) | `undefined` |
| Funksiya chaqiruvi (non-strict) | `globalThis` |
| Funksiya chaqiruvi (strict) | `undefined` |
| Method chaqiruvi (`obj.method()`) | `obj` |
| `new Constructor()` | yangi yaratilgan object |
| `fn.call(ctx)` / `fn.apply(ctx)` | `ctx` |
| `fn.bind(ctx)()` | `ctx` (permanent) |
| Arrow function | Outer scope'dan (lexical) |
| Event handler (DOM) | Element |
| `setTimeout` callback (browser non-strict) | `globalThis` |

Modern spec'da `this` Function Environment Record'ning `[[ThisValue]]` slot'ida saqlanadi (alohida EC slot emas). `ResolveThisBinding()` algoritmi joriy execution context'dan boshlab eng yaqin function/module Environment Record'gacha ko'tariladi va `[[ThisValue]]`'ni oladi.

### Kod misol

```javascript
"use strict";

const order = {
  id: 1001,
  total: 0,
  calculate(price, tax) {
    this.total = price * (1 + tax);
    console.log(this.id, this.total);
  },
  getCalculator() {
    // Arrow function — this lexical (outer = order)
    return (price, tax) => {
      this.total = price * (1 + tax);
    };
  }
};

// Method call — this = order
order.calculate(100, 0.2); // 1001 120

// Method ajratilsa — this yo'qoladi
const fn = order.calculate;
// fn(100, 0.2); // ❌ TypeError: Cannot set properties of undefined
// strict mode'da this = undefined

// Explicit binding
fn.call(order, 100, 0.2); // 1001 120 ✅

// Arrow function — this saqlanadi
const calc = order.getCalculator();
calc(200, 0.1); // order.total = 220 — lexical this
```

### Edge Cases

- **Constructor return**: `new` bilan chaqirilganda agar constructor object qaytarsa — `new` shu object'ni qaytaradi, primitive qaytarsa — `this`'ni qaytaradi.
- **Arrow function va `bind`**: Arrow function'da `bind`/`call`/`apply` `this`'ga ta'sir qilmaydi (faqat argument'larni o'rnatishi mumkin).
- **Method shorthand vs arrow class field**:
  ```javascript
  class Counter {
    count = 0;
    inc() { this.count++; }          // method — har chaqiruvda this aniqlanadi
    incArrow = () => { this.count++; } // arrow class field — this permanent (instance bound)
  }
  ```
- **`bind` chain**: `fn.bind(a).bind(b)()` da `this = a` (birinchi bind yutadi — keyingilar e'tibordan chetda).

### Follow-up savollar

1. "Strict mode `this`'ga qanday ta'sir qiladi?" — Function chaqiruvida non-strict `globalThis`, strict `undefined`. Method call, `new`, explicit binding'ga ta'sir qilmaydi. Global EC'ning `this`'iga ham ta'sir qilmaydi.
2. "Arrow function nima uchun `this`'ga ega emas?" — Spec'da arrow function'ning Function Environment Record'i `[[ThisBindingStatus]] = "lexical"`. `ResolveThisBinding()` shu status'ni ko'rib `[[ThisValue]]`'ni o'z record'idan emas, outer environment'dan oladi.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec'da `ResolveThisBinding()` (9.4.4):
1. `envRec` = `GetThisEnvironment()` — joriy lexical environment'dan eng yaqin function/module/global ER'ni topish
2. `envRec.GetThisBinding()` chaqiriladi:
   - Function ER: `[[ThisBindingStatus]] = "uninitialized"` bo'lsa `ReferenceError`, `"lexical"` bo'lsa parent'dan oladi, `"initialized"` bo'lsa `[[ThisValue]]` qaytariladi
   - Global ER: `[[GlobalThisValue]]` qaytariladi
   - Module ER: doim `undefined`

`new Constructor()` chaqiruvida `[[Construct]]` internal method ishlaydi: yangi object yaratiladi → constructor'ning FEC yaratiladi → `[[ThisBindingStatus]] = "initialized"`, `[[ThisValue]] = newObject`. Agar constructor explicit return qilsa va return qiymati object bo'lsa — `[[Construct]]` shu object'ni qaytaradi, aks holda `[[ThisValue]]`'ni.

</details>

</details>

### 9. Quyidagi kodning output'ini va EC Stack'ni ko'rsating [Senior]

```javascript
const value = "global";

function a() {
  const value = "a";
  b();
}

function b() {
  const value = "b";
  console.log(value);
  c();
}

function c() {
  console.log(value);
}

a();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob
Output: `"b"` keyin `"global"`. Lexical scoping tufayli `c()` global scope'da yozilgan — `value`'ni global'dan oladi, `a()`/`b()` chaqiruvlari ta'sir qilmaydi.

### To'liq tushuntirish

EC Stack holati va scope chain har funksiya uchun:

```
1. a() chaqirildi:
   Stack: [GEC, a]
   a EC: value = "a", [[OuterEnv]] → GEC (a global'da yozilgan)

2. b() chaqirildi (a ichidan):
   Stack: [GEC, a, b]
   b EC: value = "b", [[OuterEnv]] → GEC (b global'da yozilgan, a EMAS)

3. console.log(value) ichida b:
   b scope'da value = "b" topildi → output: "b"

4. c() chaqirildi (b ichidan):
   Stack: [GEC, a, b, c]
   c EC: [[OuterEnv]] → GEC (c global'da yozilgan)

5. console.log(value) ichida c:
   c scope'da value yo'q → [[OuterEnv]] = GEC → value = "global" → output: "global"
```

Asosiy nuqta: `c()` ning `[[OuterEnv]]` `b` ga emas, **global**'ga ko'rsatadi. Bu **lexical scoping** — funksiyaning chaqiruvchisi (caller) ahamiyatsiz, yozilgan joy muhim.

### Kod misol — dynamic scoping bilan farq

```javascript
// JavaScript: lexical scoping
const value = "global";
function a() { console.log(value); }
function b() {
  const value = "b";
  a(); // "global" — a global'da yozilgan
}
b();

// Agar JS dynamic scoping ishlatganda — "b" chiqar edi
// (Bash, Emacs Lisp kabi tillarda)
```

### Edge Cases

- Direct dynamic code evaluation calling EC'ning lexical environment'iga inject qiladi — bu lexical scoping'ni "yumshatadi", lekin scope chain'ning o'zi o'zgarmaydi.
- Dynamic function constructor (`Function` global) — har doim global scope'da yozilgan deb hisoblanadi (calling scope'ga access yo'q).
- Closure: `c()` `b` ichida **yaratilganda** (yozilganda, declare qilinganda) — uning `[[OuterEnv]]` `b` ga ko'rsatadi. Lekin yuqoridagi misolda `c()` global'da yozilgan, faqat `b` ichidan **chaqirilgan** — bu farq kritik.

### Follow-up savollar

1. "Agar `c()` `b()` ichida yozilgan bo'lsa, output qanday o'zgaradi?" — `c`'ning `[[OuterEnv]]` `b`'ga ko'rsatardi: `value = "b"` topilardi → output `"b"` `"b"`.
2. "Bu pattern qachon foyda beradi?" — Module pattern'lar, closure-based encapsulation, factory function'lar — yozilgan joy muhitiga bog'liq xulq-atvor uchun. JavaScript'ning eng kuchli features'i shu lexical scoping'ga asoslanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

Function object yaratilganda `[[Environment]]` internal slot'iga joriy LexicalEnvironment yoziladi (`OrdinaryFunctionCreate` abstract operation: `F.[[Environment]] = env`). Har funksiya chaqiruvida `[[Call]]` internal method `PrepareForOrdinaryCall` chaqiradi → bu `NewFunctionEnvironment(F, newTarget)` orqali yangi Function ER (`localEnv`) yaratadi va `localEnv.[[OuterEnv]] = F.[[Environment]]` o'rnatadi → keyin `calleeContext.LexicalEnvironment = localEnv`. Shuning uchun yangi FEC'ning `[[OuterEnv]]` funksiya **yaratilgan** paytdagi LexicalEnvironment'ga ko'rsatadi, chaqirgan funksiyaning EC'iga emas.

Bu mexanizm "closure" deb ataladigan tushunchaning fundamental implementation'i: funksiya o'zining lexical environment'ini hamroh sifatida olib yuradi.

</details>

</details>
