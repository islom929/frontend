# Scope — Interview Savollari

> Scope — o'zgaruvchilarning accessibility chegarasi. Bu bo'limda scope turlari, scope chain, lexical scoping, variable shadowing, variable lookup mexanizmi va strict mode haqida interview savollari.

---

## Savol 1: Scope nima va JavaScript da qanday scope turlari mavjud? [Junior+]

**Javob:**

Scope — o'zgaruvchilar va funksiyalarning kodning qaysi qismida **accessible** (kirish mumkin) ekanligini belgilaydigan qoidalar to'plami. JavaScript da uchta asosiy scope turi bor:

1. **Global Scope** — script'ning eng tashqi qatlami, dasturning istalgan joyidan accessible
2. **Function Scope** — funksiya tanasi ichida yaratiladi, `var` faqat shu chegarani tan oladi
3. **Block Scope** — `{}` ichida `let`/`const` bilan yaratiladi (ES6+), `if`, `for`, `while` bloklarida ishlaydi

```javascript
const globalVar = "Global";       // Global scope

function example() {
  var funcVar = "Function";       // Function scope — faqat example() ichida
  let blockInFunc = "Also func";  // Function + block scope

  if (true) {
    let blockVar = "Block";       // ✅ Block scope — faqat if ichida
    var leaks = "Leaked";         // ❌ var block scope tanimaydi — function scope'ga chiqadi
    console.log(blockVar);        // ✅ "Block"
  }

  console.log(leaks);    // ✅ "Leaked" — var function scope'da
  // console.log(blockVar); // ❌ ReferenceError — let block scope'da qoldi
}
```

Scope'ning asosiy maqsadi — **encapsulation**: nomlar to'qnashuvining oldini olish va ma'lumotni kerakli chegarada saqlash.

**Deep Dive:**

ECMAScript spec'da scope **Environment Record** orqali implement qilingan. Global scope'da ikki xil record bor: Object Environment Record (`var`, `function` → `globalThis` property) va Declarative Environment Record (`let`, `const`, `class` → `globalThis` property EMAS). Shu sababli:

```javascript
var a = 1;
let b = 2;
console.log(globalThis.a); // 1 — var globalThis property
console.log(globalThis.b); // undefined — let globalThis property EMAS
```

---

## Savol 2: Quyidagi kodning output'i nima? [Middle]

```javascript
function test() {
  var x = 1;
  let y = 2;

  {
    var x = 10;
    let y = 20;
    console.log(x, y); // ?
  }

  console.log(x, y); // ?
}

test();
```

**Javob:**

```
10, 20
10, 2
```

Block ichida `var x = 10` — `var` block scope tanimaydi, u function scope'dagi `x` ni **qayta yozdi** (3 → 10). `let y = 20` esa block scope'da yangi `y` binding yaratdi (shadowing) — function scope'dagi `y` ga tegmadi.

Block tashqarisida: `x = 10` (var qayta yozilgan), `y = 2` (block scope'dagi let tugadi, function scope'dagi let qaytdi).

| Deklaratsiya | Block ichida | Block tashqarisida | Sabab |
|---|---|---|---|
| `var x = 10` | 10 | 10 | var block scope tanimaydi — function scope'dagi x ni o'zgartirdi |
| `let y = 20` | 20 | 2 | let block scope'da yangi binding — tashqi y ga tegmadi |

---

## Savol 3: Lexical scope nima va dynamic scope dan farqi? [Middle]

**Javob:**

Lexical scope (static scope) — scope funksiya **yozilgan** (define qilingan) joyga qarab aniqlanadi, **chaqirilgan** joyga qarab emas. JavaScript lexical scoping ishlatadi.

Dynamic scope'da esa scope funksiya chaqirilgan kontekstga qarab runtime'da aniqlanadi. Bash shell va ba'zi Lisp dialektlari dynamic scope ishlatadi.

```javascript
const value = "global";

function readValue() {
  console.log(value);
}

function callFromLocal() {
  const value = "local";
  readValue();
}

callFromLocal();
// Lexical scope (JS): "global" — readValue() global'da DEFINE qilingan
// Dynamic scope bo'lganida: "local" bo'lardi — readValue() callFromLocal() dan CHAQIRILGAN
```

| Xususiyat | Lexical Scope | Dynamic Scope |
|---|---|---|
| Aniqlanish | Compile-time (parse-time) | Runtime |
| Bog'liqlik | Yozilgan pozitsiya | Chaqirilgan kontekst |
| Predictability | Yuqori — kodni o'qib bilish mumkin | Past — runtime'da o'zgaradi |
| JavaScript da | Ha (standart) | Yo'q (`this` o'xshash lekin farqli) |

**Deep Dive:**

JavaScript da `this` keyword dynamic scope'ga o'xshash ishlaydi — u funksiya chaqirilgan kontekstga qarab o'zgaradi. Lekin `this` scope chain'ga tegishli emas — u Execution Context'ning alohida component'i. Arrow function'lar esa lexical `this` ishlatadi — bu aralashtirilgan model.

Texnik jihatdan lexical scope funksiya yaratilganda `[[Environment]]` internal slot'ga joriy LexicalEnvironment saqlash orqali implement qilingan. Bu slot bir marta o'rnatiladi va hech qachon o'zgarmaydi.

---

## Savol 4: Scope chain nima va qanday ishlaydi? [Middle]

**Javob:**

Scope chain — o'zgaruvchi qidirilganda engine bosib o'tadigan **scope'lar ketma-ketligi**. Qidiruv doim joriy (ichki) scope'dan boshlanadi va `[[OuterEnv]]` reference bo'ylab tashqariga (yuqoriga) qarab davom etadi — global scope'gacha.

```javascript
const a = 1;          // Global scope

function outer() {
  const b = 2;        // outer scope

  function inner() {
    const c = 3;      // inner scope
    console.log(a + b + c); // 6
  }

  inner();
}

outer();
```

```
inner() uchun scope chain:

inner Environment Record { c: 3 }
  └── [[OuterEnv]] → outer Environment Record { b: 2 }
                        └── [[OuterEnv]] → Global Environment Record { a: 1 }
                                              └── [[OuterEnv]] → null (zanjir tugadi)

console.log(a + b + c):
  c → inner'da topildi ✅
  b → inner'da yo'q → outer'da topildi ✅
  a → inner'da yo'q → outer'da yo'q → global'da topildi ✅
```

Muhim qoidalar:
- Faqat **ichdan tashqariga** — tashqi scope ichki scope'ni ko'rmaydi
- Birinchi topilgan joyda **to'xtaydi** — shadowing shuning uchun ishlaydi
- Scope chain **o'zgarmaydi** — funksiya yaratilgan paytda belgilanadi (lexical)
- Hech joyda topilmasa → `ReferenceError`

---

## Savol 5: Quyidagi kodning output'i nima? [Middle+]

```javascript
var x = 1;

function first() {
  console.log(x); // A
}

function second() {
  var x = 2;
  first();
  console.log(x); // B
}

second();
console.log(x); // C
```

**Javob:**

```
A: 1
B: 2
C: 1
```

Bu savol **lexical scope** ni tekshiradi. `first()` funksiyasi **global scope'da** define qilingan — uning scope chain: `first → global`. `second()` ichidan chaqirilgan bo'lsa ham, `first()` scope chain'i o'zgarmaydi.

- **A:** `first()` ichida `x` → `first` scope'da yo'q → global → `x = 1`
- **B:** `second()` ichida local `var x = 2` → `x = 2`
- **C:** Global scope'dagi `x = 1` — `second()` ichidagi `var x = 2` global `x` ga tegmadi

Agar JavaScript dynamic scope ishlatganida, A javob `2` bo'lardi — chunki `first()` `second()` kontekstidan chaqirilgan. Lekin lexical scope — yozilgan joy ahamiyatli.

---

## Savol 6: Variable shadowing nima? Quyidagi kodning output'ini ayting [Middle]

```javascript
let count = 100;

function outer() {
  let count = 0;

  function increment() {
    let count = count + 1;
    return count;
  }

  return increment();
}

console.log(outer()); // ?
```

**Javob:**

```
ReferenceError: Cannot access 'count' before initialization
```

Bu klassik **TDZ + shadowing** tuzoq. `increment()` ichida `let count = count + 1` qatori:

1. `let count` — yangi block scope binding yaratadi (shadowing)
2. `= count + 1` — bu yerda `count` endi **local** (shadow) `count` ga murojaat qiladi
3. Lekin local `count` hali **TDZ** da (Temporal Dead Zone) — `let` declare qilingan lekin initialize bo'lmagan
4. Natija: `ReferenceError`

Tuzatish usullari:

```javascript
// 1-usul: boshqa nom ishlatish
function increment() {
  let newCount = count + 1; // ✅ tashqi count o'qiladi
  return newCount;
}

// 2-usul: shadow qilmaslik
function increment() {
  count = count + 1; // ✅ tashqi count o'zgartiriladi
  return count;
}
```

Variable shadowing — ichki scope'da tashqi scope'dagi o'zgaruvchi bilan bir xil nomli yangi o'zgaruvchi e'lon qilish. Ichki scope ichida tashqi o'zgaruvchi "ko'rinmay qoladi". Bu o'z-o'zidan xato emas, lekin TDZ bilan birgalikda xatoga olib kelishi mumkin.

---

## Savol 7: `globalThis` nima va nima uchun kerak? [Junior+]

**Javob:**

`globalThis` — ES2020 da kiritilgan, **cross-environment global object** ga murojaat qilish uchun standart yo'l. Turli JavaScript environment'larda global object turli nomlarga ega:

| Environment | Global Object | `globalThis` |
|---|---|---|
| Browser | `window` | `globalThis === window` ✅ |
| Node.js | `global` | `globalThis === global` ✅ |
| Web Worker | `self` | `globalThis === self` ✅ |
| Deno | `globalThis` | ✅ |

```javascript
// ES2020 dan oldin — environment tekshirish kerak edi:
const globalObj =
  typeof window !== "undefined" ? window :
  typeof global !== "undefined" ? global :
  typeof self !== "undefined" ? self :
  this;

// ES2020 dan keyin — bir satr:
const globalObj = globalThis; // ✅ istalgan environment'da ishlaydi
```

`globalThis` ning muhim farqlari:
- `var` bilan global'da e'lon qilingan o'zgaruvchilar `globalThis` property bo'ladi
- `let`/`const` bilan e'lon qilinganlar `globalThis` property **bo'lMAYDI**

```javascript
var oldVar = "accessible";
let newLet = "not property";

console.log(globalThis.oldVar); // "accessible"
console.log(globalThis.newLet); // undefined
```

---

## Savol 8: Quyidagi kodning output'i nima? Nima uchun? [Middle+]

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log("var:", i), 0);
}

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log("let:", j), 0);
}
```

**Javob:**

```
var: 3
var: 3
var: 3
let: 0
let: 1
let: 2
```

Bu **scope + closure + event loop** ning kesishishi:

**`var` bilan:**
- `var i` function scope'da (yoki global'da) **bitta** binding yaratadi
- Barcha 3 ta `setTimeout` callback'lari shu **bitta** `i` ga closure hosil qiladi
- Loop tugagandan keyin `i = 3`
- Callback'lar event loop orqali ishlaganda — barchasi `i = 3` ni ko'radi

**`let` bilan:**
- `let j` har iteratsiyada **yangi** block scope binding yaratadi
- Har bir `setTimeout` callback **o'z** `j` binding'iga closure hosil qiladi
- Iteratsiya 0 da `j = 0`, iteratsiya 1 da `j = 1`, iteratsiya 2 da `j = 2`
- Har bir callback o'z qiymatini ko'radi

**Deep Dive:**

ECMAScript spec'da `for` loop bilan `let` ishlatilganida **ForBodyEvaluation** abstract operation har iteratsiya uchun `CreatePerIterationEnvironment` ni chaqiradi. Bu operation yangi **Declarative Environment Record** yaratadi va loop variable'ning joriy qiymatini shu yangi record'ga copy qiladi. `var` uchun bunday mexanizm yo'q — bitta VariableEnvironment binding.

---

## Savol 9: `var` ning block scope tanimasligini tushuntiring va bu qanday muammolarga olib keladi? [Junior+]

**Javob:**

`var` keyword faqat **function scope** ni tan oladi — `if`, `for`, `while`, `{}` kabi block konstruktsiyalar `var` uchun scope chegarasi hisoblanmaydi. `var` bilan block ichida e'lon qilingan o'zgaruvchi tashqi function scope'ga "chiqadi".

```javascript
function example() {
  console.log(status); // undefined — hoist bo'lgan (TDZ emas)

  if (false) {
    var status = "active";
    // ❌ if bloki bajarilMAYDI, lekin var hoist bo'ladi!
    // Creation Phase da: status = undefined
  }

  console.log(status); // undefined — assign bo'lmadi, lekin mavjud

  for (var i = 0; i < 5; i++) {
    // ...
  }

  console.log(i); // 5 — var for block'ni tanimaydi
}
```

Bu qanday muammolarga olib keladi:
1. **Kutilmagan o'zgaruvchi ko'rinishi** — block ichidagi o'zgaruvchi tashqarida ham accessible
2. **Loop + async muammosi** — barcha callback'lar bitta binding'ni ko'radi
3. **Hoisting confusion** — bajarilmaydigan code'dagi var ham hoist bo'ladi
4. **Debugging qiyinligi** — o'zgaruvchi qayerdan kelganini topish qiyin

Yechim: `let`/`const` ishlatish. Zamonaviy JavaScript'da `var` ishlatish uchun hech qanday texnik sabab qolmagan.

---

## Savol 10: Strict mode nima va u scope'ga qanday ta'sir ko'rsatadi? [Middle]

**Javob:**

Strict mode — `"use strict"` directive bilan yoqiladigan JavaScript'ning qat'iyroq rejimi. ES5 da kiritilgan. Scope va o'zgaruvchilarga tegishli asosiy ta'sirlari:

**1. Implicit global'lar taqiqlanadi:**
```javascript
"use strict";
undeclared = 10; // ❌ ReferenceError
// Non-strict: globalThis.undeclared = 10 bo'lardi
```

**2. `this` default binding'da `undefined`:**
```javascript
"use strict";
function show() { console.log(this); }
show(); // undefined (non-strict: globalThis)
```

**3. `eval` o'z scope'ini yaratadi:**
```javascript
"use strict";
eval("var x = 10");
// console.log(x); // ❌ ReferenceError
// Non-strict: x tashqi scope'ga chiqardi
```

**4. Duplicate parameter nomlari taqiqlanadi:**
```javascript
"use strict";
// function f(a, a) {} // ❌ SyntaxError
```

**5. `with` statement taqiqlanadi:**
```javascript
"use strict";
// with (obj) {} // ❌ SyntaxError — scope chain'ni buzadi
```

**6. `arguments.callee` taqiqlanadi**

ES6 **modullar** va **class** tanasi avtomatik strict mode'da ishlaydi — alohida `"use strict"` yozish shart emas. Zamonaviy loyihalarda modullar ishlatilgani uchun deyarli barcha kod strict mode'da.

| Xususiyat | Non-strict | Strict |
|---|---|---|
| Declare qilmagan assign | Global property yaratadi | ReferenceError |
| `this` (default binding) | `globalThis` | `undefined` |
| `eval` scope | Tashqi scope'ga chiqadi | O'z scope'ida qoladi |
| `with` statement | Ruxsat etilgan | SyntaxError |
| Duplicate params | Ruxsat (oxirgisi qoladi) | SyntaxError |

---

## Savol 11: Quyidagi kodda nima xato va qanday tuzatasiz? [Middle+]

```javascript
function createHandlers(elements) {
  var handlers = [];

  for (var i = 0; i < elements.length; i++) {
    handlers.push(function () {
      console.log(`Element ${i}: ${elements[i]}`);
    });
  }

  return handlers;
}

const handlers = createHandlers(["a", "b", "c"]);
handlers[0](); // ?
handlers[1](); // ?
handlers[2](); // ?
```

**Javob:**

Muammo: barcha handler'lar bir xil natija beradi:

```
Element 3: undefined
Element 3: undefined
Element 3: undefined
```

Sabab: `var i` function scope'da **bitta** binding. Loop tugaganda `i = 3`. Barcha closure'lar shu bitta `i` ga reference saqlaydi. `elements[3]` mavjud emas → `undefined`.

**3 ta tuzatish usuli:**

```javascript
// 1-usul: let ishlatish (eng oddiy va to'g'ri)
for (let i = 0; i < elements.length; i++) {
  handlers.push(function () {
    console.log(`Element ${i}: ${elements[i]}`);
    // ✅ Har iteratsiyada yangi i binding
  });
}

// 2-usul: IIFE bilan scope yaratish (ES5 usuli)
for (var i = 0; i < elements.length; i++) {
  (function (index) {
    handlers.push(function () {
      console.log(`Element ${index}: ${elements[index]}`);
      // ✅ IIFE har chaqiruvda yangi scope — index har birida alohida
    });
  })(i);
}

// 3-usul: forEach ishlatish (har callback o'z scope)
elements.forEach(function (element, index) {
  handlers.push(function () {
    console.log(`Element ${index}: ${element}`);
    // ✅ forEach callback'i har element uchun yangi scope yaratadi
  });
});
```

---

## Savol 12: `let`, `const`, `var` ning scope farqlarini jadvali bilan tushuntiring [Junior+]

**Javob:**

| Xususiyat | `var` | `let` | `const` |
|---|---|---|---|
| **Scope** | Function scope | Block scope | Block scope |
| **Hoisting** | Ha (undefined bilan) | Ha (TDZ bilan) | Ha (TDZ bilan) |
| **TDZ** | Yo'q | Ha | Ha |
| **Re-declaration** | Ha (bir scope'da) | Yo'q (SyntaxError) | Yo'q (SyntaxError) |
| **Re-assignment** | Ha | Ha | Yo'q (TypeError) |
| **Global scope'da** | `globalThis` property | `globalThis` da emas | `globalThis` da emas |
| **`for` loop'da** | Bitta binding | Har iteratsiya yangi | Har iteratsiya yangi |

```javascript
// Re-declaration
var a = 1;
var a = 2;    // ✅ ishlaydi — a qayta e'lon qilindi (2 bo'ldi)

let b = 1;
// let b = 2; // ❌ SyntaxError: Identifier 'b' has already been declared

// const va re-assignment
const c = { name: "Alice" };
// c = {};     // ❌ TypeError: Assignment to constant variable
c.name = "Bob"; // ✅ Object ICHIDAGI property o'zgaradi — reference o'zgarmadi
```

Zamonaviy JavaScript'da qoida: **default `const`**, faqat qiymat o'zgarishi kerak bo'lganda `let`. `var` ishlatmaslik.

---

## Savol 13: Scope chain va call stack farqi nima? [Senior]

**Javob:**

Bu ikki tushuncha ko'p aralashtiriladi, lekin ular **butunlay farqli** mexanizmlar:

**Call Stack** — funksiyalarning **chaqiruv tartibini** boshqaradi. LIFO (Last In, First Out) prinsipi. Funksiya chaqirilganda stack'ga tushadi, tugaganda stack'dan chiqadi.

**Scope Chain** — o'zgaruvchilarning **qidiruv yo'lini** belgilaydi. Lexical nesting (funksiya yozilgan joyga qarab). Runtime'da o'zgarmaydi.

```javascript
function a() {
  const x = "a";
  b();
}

function b() {
  const y = "b";
  console.log(x); // ❌ ReferenceError
}

a();
```

```
Call Stack:          Scope Chain:
┌─────────┐
│  b()    │         b() → global (b global'da define qilingan)
│  a()    │         a() → global (a global'da define qilingan)
│ global  │
└─────────┘

Call Stack bo'yicha: b() → a() → global (chaqiruv tartibi)
Scope Chain bo'yicha: b() → global (lexical nesting)

b() call stack'da a() ustida tursa ham, b() scope chain'ida
a() scope yo'q — chunki b() a() ICHIDA define qilinMAGAN.
```

| Xususiyat | Call Stack | Scope Chain |
|---|---|---|
| Nima boshqaradi | Chaqiruv tartibi | O'zgaruvchi qidirish |
| Qachon aniqlanadi | Runtime (dynamic) | Parse-time (static/lexical) |
| Tartib | Chaqirilgan ketma-ketlik | Yozilgan nesting ketma-ketlik |
| Tuzilishi | LIFO stack | `[[OuterEnv]]` linked list |

**Deep Dive:**

Closure bu ikki mexanizm orasidagi ko'prik. Funksiya call stack'dan chiqqandan keyin ham uning scope chain'i (Environment Record'lari) saqlanib qoladi — agar ichki funksiya ularga reference saqlasa. Call stack frame'i yo'qoladi, lekin scope data heap'da tirik qoladi.

---

## Savol 14: Quyidagi kodning output'ini ayting [Middle+]

```javascript
function outer() {
  var a = 1;
  let b = 2;

  function inner() {
    console.log(a); // A
    console.log(b); // B

    var a = 3;
    let b = 4;

    console.log(a); // C
    console.log(b); // D
  }

  inner();
}

outer();
```

**Javob:**

```
A: undefined
B: ReferenceError: Cannot access 'b' before initialization
```

C va D ga yetib bormaydi — B da xato bo'lib to'xtaydi.

Tahlil:

- **A: `undefined`** — `inner()` ichida `var a = 3` bor. `var a` hoist bo'ladi (Creation Phase da `a = undefined`). `console.log(a)` — local `a` ko'riladi (`undefined`), tashqi `a = 1` shadow qilingan.

- **B: `ReferenceError`** — `inner()` ichida `let b = 4` bor. `let b` hoist bo'ladi lekin TDZ da. `console.log(b)` — local `b` ni ko'radi (shadow), lekin u TDZ da → `ReferenceError`.

Bu savolda **hoisting + shadowing + TDZ** uchala tushuncha birgalikda tekshiriladi. `var` hoist bo'lganda `undefined` beradi, `let` hoist bo'lganda TDZ da qoladi — shu farq natijani belgilaydi.

---

## Savol 15: Module scope nima va global scope dan farqi? [Middle]

**Javob:**

ES Modules (`import`/`export`) da har bir fayl o'zining **module scope**'iga ega. Module scope'dagi o'zgaruvchilar boshqa modullardan **ko'rinmaydi** — faqat `export` qilinganlar accessible.

```javascript
// utils.js
const SECRET = "abc123";        // Module scope — tashqaridan ko'rinmaydi
export const API_URL = "/api";  // Export — import qilsa ko'rinadi

// app.js
import { API_URL } from "./utils.js";
console.log(API_URL);    // ✅ "/api"
// console.log(SECRET);  // ❌ ReferenceError — export qilinmagan
```

Module scope va global scope farqlari:

| Xususiyat | Global Scope (script) | Module Scope |
|---|---|---|
| `var` declaration | `globalThis` property | Module scope'da qoladi |
| `this` (top-level) | `globalThis` | `undefined` |
| Strict mode | Manual (`"use strict"`) | Avtomatik |
| O'zgaruvchi ko'rinishi | Barcha script'lardan | Faqat modul ichidan |
| Code execution | Bir necha marta | Faqat bir marta (cached) |

```javascript
// <script> tag'da (global scope):
var x = 10;
console.log(this);           // window (globalThis)
console.log(globalThis.x);   // 10

// <script type="module"> tag'da (module scope):
var y = 20;
console.log(this);           // undefined
console.log(globalThis.y);   // undefined — var ham globalThis'ga tushmaydi
```

---

## Savol 16: Scope bilimingizni ishlatib, quyidagi muammoni hal qiling [Senior]

**Savol:** `createPrivateCounter()` funksiyasini yozing. U `increment()`, `decrement()`, `getCount()`, `reset()` method'lariga ega object qaytarsin. `count` tashqaridan o'zgartirilishi mumkin bo'lmasin.

**Javob:**

```javascript
function createPrivateCounter(initialValue = 0) {
  // count faqat shu scope'da — tashqaridan accessible emas
  let count = initialValue;

  return {
    increment() {
      count++;
      return count;
    },
    decrement() {
      count--;
      return count;
    },
    getCount() {
      return count;
    },
    reset() {
      count = initialValue;
      return count;
    }
  };
}

const counter = createPrivateCounter(10);
console.log(counter.increment());  // 11
console.log(counter.increment());  // 12
console.log(counter.decrement());  // 11
console.log(counter.getCount());   // 11
console.log(counter.reset());      // 10

// count'ga to'g'ridan-to'g'ri kirish imkonsiz:
console.log(counter.count);        // undefined — property emas
// counter.count = 999;            // Bu yangi property yaratadi, lekin
//                                    ichki count'ga ta'sir qilmaydi
```

Bu pattern **scope-based encapsulation** deyiladi — function scope + closure yordamida private ma'lumotni saqlash. ES2022 da `#private` fields kiritilgan, lekin bu pattern hali ham keng ishlatiladi — ayniqsa class ishlatmaydigan functional code'da.

Scope chain: `increment()` → `createPrivateCounter()` scope (count bu yerda) → global. `count` `createPrivateCounter()` scope'da turadi — faqat shu scope'dan return qilingan method'lar uni ko'radi va o'zgartiradi.

---

## Savol 17: Quyidagi kodning output'i nima? [Middle+]

```javascript
let a = 1;
const b = 2;

function first() {
  console.log(a, b); // A

  if (true) {
    let a = 10;
    const b = 20;
    console.log(a, b); // B

    function second() {
      console.log(a, b); // C
    }
    second();
  }

  console.log(a, b); // D
}

first();
```

**Javob:**

```
A: 1, 2
B: 10, 20
C: 10, 20
D: 1, 2
```

Tahlil:
- **A:** `first()` scope'da local `a`/`b` yo'q → global'dan: `a=1, b=2`
- **B:** `if` block ichida `let a = 10`, `const b = 20` — shadowing. Block scope: `a=10, b=20`
- **C:** `second()` `if` block **ichida** define qilingan → lexical scope bo'yicha block scope'dagi `a=10, b=20` ni ko'radi
- **D:** `if` block tugadi — block scope o'zgaruvchilari ko'rinmaydi. Yana global: `a=1, b=2`

Diqqat: C — `second()` `if` block ichida define qilingan, shuning uchun uning scope chain `if block → first → global`. Block scope'dagi shadow o'zgaruvchilarni ko'radi.

---

## Savol 18: `with` statement nima uchun taqiqlangan va u scope chain'ga qanday ta'sir ko'rsatadi? [Senior]

**Javob:**

`with` statement object'ning property'larini scope chain'ga qo'shadi — object property'lariga to'g'ridan-to'g'ri nom bilan murojaat qilish imkonini beradi:

```javascript
const config = { host: "localhost", port: 3000, protocol: "https" };

// with bilan (strict mode'da TAQIQ):
with (config) {
  console.log(host);     // "localhost" — config.host
  console.log(port);     // 3000 — config.port
  console.log(protocol); // "https" — config.protocol
}
```

Nima uchun taqiqlangan:

1. **Scope chain buziladi** — engine compile-time da identifier qaysi scope'ga tegishli ekanini aniqlay olmaydi. `with` ichidagi `host` — bu local variable mi, tashqi scope variable mi, yoki object property mi? Runtime'gacha noma'lum.

2. **Optimizatsiya imkonsiz** — V8 scope analysis va variable caching qila olmaydi, chunki scope dynamic bo'lib qoladi.

3. **Bug'lar** — agar object'da kutilmagan property bo'lsa, boshqa scope'dagi o'zgaruvchi "yashirinadi":

```javascript
const obj = { toString: "custom" };
with (obj) {
  console.log(toString); // "custom" — Object.prototype.toString EMAS!
  // ❌ Kutilmagan shadowing
}
```

**Deep Dive:**

`with` statement yangi **Object Environment Record** yaratadi va uni scope chain'ning tepasiga qo'yadi. Bu record'ning binding object'i `with` ga berilgan object. Identifier resolve qilinganda avval shu object'da qidiriladi, topilmasa tashqi scope'ga o'tiladi. Bu mexanizm global scope'ning Object Environment Record'iga o'xshash — aslida global scope'ning `var` xulq-atvori ham shu orqali ishlaydi.

Strict mode'da `with` SyntaxError beradi — shu sababli zamonaviy JavaScript'da deyarli uchramaydi. Destructuring yaxshiroq alternativ:

```javascript
const { host, port, protocol } = config; // ✅ Aniq, optimizable
```

---
