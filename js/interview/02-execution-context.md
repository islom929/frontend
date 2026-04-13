# Execution Context — Interview Savollari

> Execution Context ichki mexanizmlari bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.

---

## Nazariy savollar

### 1. Execution Context nima va unda qanday komponentlar bor? [Junior+]

<details>
<summary>Javob</summary>

Execution Context — JavaScript engine har bir kod bo'lagini bajarishdan oldin yaratiladigan ichki muhit. Bu muhit quyidagi komponentlardan iborat:

1. **LexicalEnvironment** — `let`, `const`, `class` declaration'lar saqlanadi
2. **VariableEnvironment** — `var` va `function` declaration'lar saqlanadi
3. **ThisBinding** — `this` qiymati

Har bir EC ikki bosqichda yaratiladi:
- **Creation Phase** — muhit tayyorlanadi, o'zgaruvchilar e'lon qilinadi, lekin kod bajarilmaydi
- **Execution Phase** — kod qator-baqatar bajariladi

```javascript
// Creation Phase da nima sodir bo'ladi:
console.log(x);    // undefined — var Creation Phase da undefined
console.log(y);    // ReferenceError — let TDZ da
console.log(greet); // function — to'liq hoist

var x = 10;
let y = 20;
function greet() { return "Hi"; }
```

Bu mexanizm hoisting'ning asosi — `var` Creation Phase da `undefined` bilan initialize qilinadi, shuning uchun e'lon qilishdan oldin o'qish mumkin (lekin qiymati `undefined`).

</details>

### 2. Execution Context necha xil bo'ladi? [Junior+]

<details>
<summary>Javob</summary>

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
<summary>Javob</summary>

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
- **Creation Phase** (har EC yaratilganda) — parser'ning static ma'lumotlari asosida Environment Record'lar **runtime'da** to'ldiriladi: `CreateMutableBinding` va `InitializeBinding(undefined)` chaqiriladi (var uchun), let/const faqat `CreateMutableBinding` (uninitialized holatda).

Shuning uchun Creation Phase tez ishlaydi — engine source'ni qayta scan qilmaydi, faqat parser'ning metadata'sidan EC environment'ni yaratadi.

</details>

### 4. Variable Environment va Lexical Environment farqi nima? [Middle+]

<details>
<summary>Javob</summary>

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
<summary>Javob</summary>

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
// counter = 10; // ❌ SyntaxError: Assignment to constant variable
```

**Deep Dive:**

Declarative ER tezroq ishlashining sababi: engine compile vaqtida barcha binding'larning index'ini biladi — hash table lookup kerak emas. Object ER esa dynamic property lookup qiladi (window object orqali) — sekinroq va ko'p optimizatsiyalar uchun to'siq.

Module ER'ning "live binding" xususiyati — V8 spec'dagi `GetBindingValue` operatsiyasi har safar original exporting module'dan qiymatni oladi. Shuning uchun circular import'larda `import` qiymati keyinroq ham yangilanishi mumkin.

</details>

### 6. Global scope'da var bilan e'lon qilingan o'zgaruvchi let dan nima farq qiladi? [Middle]

<details>
<summary>Javob</summary>

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
<summary>Javob</summary>

`eval()` string sifatida berilgan JavaScript kodni parse va execute qiladi. Uni ishlatmaslik kerak chunki:

**1. Xavfsizlik:** Tashqi source'dan kelgan string'ni baholash code injection imkonini beradi:

```javascript
// ❌ Xavfli — foydalanuvchi input'ini baholash
const userInput = "alert('hacked')";
// ixtiyoriy kod bajariladi!
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

V8 ayrim hollarda **statik bo'sh** kod baholashni specially handle qiladi (constant folding orqali olib tashlanishi mumkin), lekin har qanday non-trivial yoki dinamik baholash scope escape trigger qiladi va kompilyator lokal o'zgaruvchilarni context (heap) ga ko'chirishga majbur bo'ladi.

**3. Direct vs Indirect — spec farqi va scope muammolari:**

Spec'da kod baholash ikki xil chaqiriladi, ular turli semantika beradi. Bu farq ko'pchilik developer'ga noma'lum va interview'larda muhim:

```javascript
// DIRECT — identifier orqali to'g'ridan-to'g'ri chaqiriladi
function direct() {
  // calling scope'ga inject qiladi!
}

// INDIRECT — boshqa expression orqali chaqiriladi
function indirect() {
  // calling scope'ga inject qilMAYDI
  // global scope'ga ketadi
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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

```
2
```

`inner()` funksiyasi `outer()` **ichida yozilgan** — shuning uchun uning outer reference `outer()`ning environment'iga bog'langan. `inner()` `anotherScope()` ichida **chaqirilgan** bo'lsa ham — JavaScript lexical scoping ishlatadi, dynamic emas.

Scope chain: `inner() → outer() → global`

`inner()` ichida `a` qidiriladi:
1. `inner()` scope'ida → yo'q
2. `outer()` scope'ida → `a = 2` ✅ topildi!

`anotherScope()` dagi `a = 3` ga hech qachon borilmaydi — chunki `inner()` `outer()` da yozilgan, `anotherScope()` da emas.

Bu **closure** mexanizmi — `inner()` `outer()` tugagandan keyin ham uning scope'iga kira oladi. Bu haqda to'liq [05-closures.md](../05-closures.md) da.

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
