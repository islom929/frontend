# Execution Context — Interview Savollari

> Execution Context ichki mexanizmlari bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.


---

## Savol 1: Execution Context nima va unda qanday komponentlar bor? [Junior+]

**Javob:**

Execution Context — JavaScript engine har bir kod bo'lagini bajarishdan oldin yaratiladigan ichki muhit. Bu muhit quyidagi komponentlardan iborat:

1. **LexicalEnvironment** — `let`, `const`, `function`, `class` declaration'lar saqlanadi
2. **VariableEnvironment** — `var` declaration'lar saqlanadi
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

---

## Savol 2: Execution Context necha xil bo'ladi? [Junior+]

**Javob:**

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

---

## Savol 3: Creation Phase va Execution Phase da nima sodir bo'ladi? [Middle]

**Javob:**

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

Creation Phase'da engine aslida source code'ni ikkinchi marta scan qilmaydi — parsing bosqichida (AST yaratishda) allaqachon barcha declaration'lar aniqlangan. Creation Phase parser'ning ma'lumotlari asosida environment record'larni to'ldiradi.

---

## Savol 4: Variable Environment va Lexical Environment farqi nima? [Middle+]

**Javob:**

ES6 dan boshlab har bir EC'da ikkita alohida environment bor:

| Xususiyat | VariableEnvironment | LexicalEnvironment |
|-----------|--------------------|--------------------|
| Nima saqlanadi | `var` | `let`, `const`, `function`, `class` |
| Scope | Function scope | Block scope |
| Hoisting | `undefined` bilan | `uninitialized` (TDZ) |
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

---

## Savol 5: Quyidagi kodning output'ini ayting [Middle]

```javascript
var x = 1;

function foo() {
  console.log(x);
  var x = 2;
  console.log(x);
}

foo();
console.log(x);
```

**Javob:**

```
undefined
2
1
```

Qadam-baqadam:

```
foo() FEC Creation Phase:
  VariableEnvironment: x = undefined (var hoist — local x!)

foo() Execution Phase:
  console.log(x)  → undefined (local x hali undefined)
  x = 2            → local x endi 2
  console.log(x)  → 2

Global:
  console.log(x)  → 1 (global x o'zgarmagan)
```

Asosiy nuqta: `foo()` ichida `var x = 2` bor — bu **local** `x` yaratadi. Creation Phase da local `x` = `undefined` bo'ladi. Shuning uchun birinchi `console.log(x)` local `x` ni topadi (undefined), global `x` (1) ni emas. Bu **variable shadowing** — local scope global scope'ni "yashiradi".

---

## Savol 6: Quyidagi kodning output'ini ayting [Middle+]

```javascript
console.log(foo);
console.log(bar);

var foo = "Hello";
function foo() { return "World"; }
var bar = function() { return "!"; };

console.log(foo);
console.log(bar);
```

**Javob:**

```
function foo() { return "World"; }
undefined
"Hello"
function() { return "!"; }
```

Creation Phase:
1. `var foo` → `undefined`
2. `function foo()` → **function declaration function bilan override qiladi** → `foo = function foo(){...}`
3. `var bar` → `undefined`

Function declaration `var` dan yuqori priority'ga ega — ikkalasi bir xil nomda bo'lsa, function declaration yutadi.

Execution Phase:
1. `console.log(foo)` → `function foo(){...}` (Creation Phase natijasi)
2. `console.log(bar)` → `undefined` (var hoist)
3. `foo = "Hello"` → endi foo string
4. `bar = function(){...}` → endi bar function
5. `console.log(foo)` → `"Hello"`
6. `console.log(bar)` → `function(){ return "!"; }`

---

## Savol 7: Environment Record turlari va farqlari nima? [Senior]

**Javob:**

ECMAScript spec'da Environment Record — scope'dagi barcha binding'lar saqlanadigan tuzilma. Uch asosiy turi bor:

**1. Declarative Environment Record:**
- `let`, `const`, `var`, `function`, `class`, `import`, `catch` parameter'lar saqlanadi
- Function va block scope'larda ishlatiladi
- O'zgaruvchilar to'g'ridan-to'g'ri record ichida — **tez** (index-based access)

**2. Object Environment Record:**
- Biror object'ning property'larini binding sifatida ko'rsatadi
- Global scope'da `var` va function declaration'lar `window` orqali → Object ER
- `with` statement uchun ishlatiladi

**3. Global Environment Record** (maxsus — ikkisining birikmasi):
- Object ER (`var`, function → window property bo'ladi)
- Declarative ER (`let`, `const`, `class` → window'da yo'q)

```javascript
var name = "Ali";       // Object ER → window.name = "Ali"
let age = 25;           // Declarative ER → window.age = undefined

console.log(window.name); // "Ali" ✅
console.log(window.age);  // undefined ❌ — Declarative ER da, window da emas
console.log(age);          // 25 ✅ — scope'da accessible
```

**Deep Dive:**

Declarative ER tezroq ishlashining sababi: engine compile vaqtida barcha binding'larning index'ini biladi — hash table lookup kerak emas. Object ER esa dynamic property lookup qiladi (window object orqali) — sekinroq.

---

## Savol 8: Quyidagi kodning output'ini ayting [Middle+]

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

**Javob:**

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

---

## Savol 9: Global scope'da var bilan e'lon qilingan o'zgaruvchi let dan nima farq qiladi? [Middle]

**Javob:**

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
delete window.x;              // ✅ o'chirish mumkin (configurable: true)
// let o'zgaruvchini delete bilan o'chirib bo'lmaydi
```

Bu farqning sababi: `var` va `let` turli Environment Record'larda saqlanadi. `var` → Object ER (window object orqali), `let` → Declarative ER (alohida, window'dan mustaqil).

---

## Savol 10: Quyidagi kodda EC Stack qanday o'zgaradi? Output nima? [Middle]

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

**Javob:**

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

---

## Savol 11: Bu kodda xato bor. Toping va tuzating [Middle]

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

**Javob:**

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

---

## Savol 12: Quyidagi kodning output'ini ayting [Senior]

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

**Javob:**

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

---

## Savol 13: eval() nima uchun ishlatmaslik kerak? [Middle]

**Javob:**

`eval()` string sifatida berilgan JavaScript kodni parse va execute qiladi. Uni ishlatmaslik kerak chunki:

**1. Xavfsizlik:** Tashqi source'dan kelgan string'ni eval qilish code injection imkonini beradi:

```javascript
// ❌ Xavfli — foydalanuvchi input'ini eval qilish
const userInput = "alert('hacked')";
eval(userInput); // ❌ ixtiyoriy kod bajariladi!
```

**2. Performance:** Engine eval mavjud scope'dagi barcha optimization'larni bekor qiladi:

```javascript
function optimized() {
  var x = 10;
  // V8 x ni register'da saqlashi mumkin (tez)
  return x + 1;
}

function notOptimized() {
  var x = 10;
  eval(""); // ❌ hatto bo'sh eval ham optimization'ni buzadi!
  // V8 x ni heap'da saqlashga majbur — chunki eval x ni o'zgartirishi MUMKIN
  return x + 1;
}
```

**3. Scope muammolari:** Non-strict mode'da eval calling scope'ga o'zgaruvchi inject qilishi mumkin:

```javascript
function test() {
  eval("var injected = 'surprise'");
  console.log(injected); // "surprise" — ❌ eval scope'ga inject qildi!
}
```

**O'rniga ishlatish kerak:** `JSON.parse()` (data uchun), `new Function()` (oxirgi chora), yoki boshqa pattern'lar.

---
