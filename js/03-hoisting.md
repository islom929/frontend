# Bo'lim 3: Hoisting — Ichki Mexanizm

> Hoisting — JavaScript engine'ning Creation Phase da o'zgaruvchi va funksiya declaration'larni scope'ning boshiga registratsiya qilish mexanizmi. Kod fizik ravishda "ko'tarilmaydi" — engine shunchaki declaration'larni kod bajarilishidan oldin qayta ishlaydi.

---

## Mundarija

- [Hoisting Nima](#hoisting-nima)
- [Hoisting Aslida Nima Sodir Bo'layotgani](#hoisting-aslida-nima-sodir-bolayotgani)
- [var Hoisting](#var-hoisting)
- [let va const Hoisting](#let-va-const-hoisting)
- [Temporal Dead Zone (TDZ)](#temporal-dead-zone-tdz)
- [Function Declaration Hoisting](#function-declaration-hoisting)
- [Function Expression Hoisting](#function-expression-hoisting)
- [var vs let vs const — To'liq Taqqoslash](#var-vs-let-vs-const--toliq-taqqoslash)
- [Hoisting Priority — Function vs Variable](#hoisting-priority--function-vs-variable)
- [Class Hoisting](#class-hoisting)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Hoisting Nima

### Nazariya

Hoisting — JavaScript'da o'zgaruvchi va funksiya declaration'larning kod bajarilishidan **oldin** scope'da mavjud bo'lishi hodisasi. Ko'pchilik buni "declaration'lar fayl tepasiga ko'tariladi" deb tushuntiradi, lekin bu texnik jihatdan noto'g'ri. Kod hech qayerga ko'chirilmaydi — engine shunchaki **Creation Phase** da declaration'larni qayta ishlaydi.

Hoisting terminining o'zi noto'g'ri tushunchaga olib keladi — aslida sodir bo'layotgan narsa Creation Phase dagi **early binding** (erta bog'lash). Engine source code'ni parse qilganida barcha declaration'larni topadi va ularni execution context'ning environment record'iga yozadi. Shu sababli kod bajarilishida (Execution Phase) bu declaration'lar allaqachon mavjud.

Qanday narsalar hoist bo'ladi:
- `var` declaration'lar → `undefined` bilan initialize
- `function` declaration'lar → to'liq funksiya bilan initialize
- `let` / `const` declaration'lar → hoist bo'ladi, lekin **initialize qilinMAYDI** (TDZ)
- `class` declaration'lar → hoist bo'ladi, lekin **initialize qilinMAYDI** (TDZ)
- `import` declaration'lar → hoist bo'ladi

### Kod Misollari

```javascript
// Hoisting tufayli bu kod ishlaydi:
console.log(message); // undefined — xato emas!
var message = "Hello";
// ✅ var Creation Phase da undefined bilan initialize — shuning uchun o'qish mumkin

greet(); // "Hi!" — xato emas!
function greet() {
  console.log("Hi!");
}
// ✅ function declaration Creation Phase da to'liq hoist — chaqirish mumkin
```

---

## Hoisting Aslida Nima Sodir Bo'layotgani

### Nazariya

"Hoisting" atamasi oddiylashtirish — aslida bu [02-execution-context.md](02-execution-context.md) da o'rganilgan **Creation Phase** ning natijasi. Kod tepaga ko'chirilmaydi — engine Creation Phase da environment record'ga binding'larni yozadi.

Jarayon qadam-baqadam:

1. Engine source code'ni **parse** qiladi va AST hosil qiladi
2. Execution context yaratiladi — **Creation Phase** boshlanadi
3. Engine AST ni scan qilib barcha **declaration**'larni topadi:
   - `var` topilsa → VariableEnvironment'ga yoziladi, `undefined` bilan initialize
   - `function declaration` topilsa → LexicalEnvironment'ga yoziladi, to'liq funksiya bilan initialize
   - `let`/`const` topilsa → LexicalEnvironment'ga yoziladi, lekin **initialize qilinMAYDI**
4. **Execution Phase** boshlanadi — kod qator-baqatar bajariladi
5. `var x = 10` ga yetganda → `x` allaqachon mavjud (undefined), endi `10` assign bo'ladi
6. `let y = 20` ga yetganda → `y` allaqachon mavjud (lekin TDZ da), endi `20` assign bo'ladi va TDZ tugaydi

### Under the Hood

```javascript
console.log(a); // undefined
console.log(b); // ReferenceError: Cannot access 'b' before initialization
var a = 10;
let b = 20;
```

Engine ichida nima sodir bo'ladi:

```
PARSING → AST yaratiladi

CREATION PHASE:
  VariableEnvironment:
    a → undefined          ← var topildi, undefined bilan initialize

  LexicalEnvironment:
    b → <uninitialized>    ← let topildi, lekin initialize qilinMADI (TDZ)

EXECUTION PHASE:
  console.log(a)  → a mavjud, qiymati undefined → output: undefined
  console.log(b)  → b mavjud, lekin <uninitialized> → ReferenceError!
                     "Cannot access 'b' before initialization"
  a = 10           → a endi 10
  b = 20           → b endi 20, TDZ tugadi
```

Muhim farq: `var` va `let` ikkalasi ham hoist bo'ladi — farqi **initialization** da. `var` darhol `undefined` bilan initialize qilinadi, `let`/`const` esa initialize qilinMAYDI (TDZ holatida qoladi).

`let` hoist bo'lishining isboti:

```javascript
let x = "global";

function test() {
  console.log(x); // ReferenceError — "global" emas!
  let x = "local";
}

test();
// Agar let hoist bo'lmaganida — console.log(x) global "global" ni ko'rsatar edi
// Lekin ReferenceError chiqdi — demak engine local let x ni BILADI
// (hoist bo'lgan), lekin hali initialize qilinMAGAN (TDZ)
```

---

## var Hoisting

### Nazariya

`var` bilan e'lon qilingan o'zgaruvchilar Creation Phase da **`undefined`** qiymati bilan initialize qilinadi. Bu degani `var` o'zgaruvchini e'lon qilishdan oldin o'qish mumkin — lekin qiymati `undefined` bo'ladi (haqiqiy qiymat emas).

`var` hoisting'ning xususiyatlari:
- `var` **function-scoped** — faqat funksiya chegarasida qoladi
- Block scope (if, for, while) `var` ni cheklamaydi
- `var` declaration hoist bo'ladi, lekin **assignment** (qiymat berish) hoist bo'lMAYDI
- Bir xil nomda bir necha marta `var` e'lon qilish mumkin — xato bermaydi (re-declaration)

### Kod Misollari

```javascript
// var declaration hoist bo'ladi, assignment bo'lmaydi:
console.log(name);  // undefined — declaration hoist bo'ldi
var name = "Ali";   // assignment faqat shu qatorda bajariladi
console.log(name);  // "Ali"

// Engine buni shunday ko'radi:
// var name;           ← Creation Phase da (undefined bilan)
// console.log(name);  ← Execution: undefined
// name = "Ali";       ← Execution: assignment
// console.log(name);  ← Execution: "Ali"
```

```javascript
// var function-scoped — block scope cheklamaydi:
function example() {
  console.log(x); // undefined — hoist bo'lgan

  if (false) {
    var x = 10; // ❌ Bu kod HECH QACHON bajarilmaydi
    // Lekin var declaration Creation Phase da hoist bo'ladi!
  }

  console.log(x); // undefined — assignment bajarilmagan
}

example();
// ✅ if (false) ichidagi var ham hoist bo'ladi
// Chunki hoist Creation Phase da — runtime shart (if) tekshirilmaydi
```

```javascript
// var re-declaration — xato bermaydi:
var count = 1;
var count = 2; // ✅ xato yo'q — ikkinchi e'lon birinchini override qiladi
console.log(count); // 2

// let bilan bu xato beradi:
// let count = 1;
// let count = 2; // ❌ SyntaxError: Identifier 'count' has already been declared
```

---

## let va const Hoisting

### Nazariya

`let` va `const` ham **hoist bo'ladi** — lekin `var` dan farqli ravishda ular Creation Phase da **initialize qilinMAYDI**. Ular `<uninitialized>` holatda qoladi — bu **Temporal Dead Zone (TDZ)** deb ataladi.

TDZ — bu scope boshlanganidan `let`/`const` declaration'ga yetguncha bo'lgan oraliq. Bu oraliqda o'zgaruvchini o'qishga yoki yozishga urinish `ReferenceError` beradi.

`let` va `const` nima uchun TDZ da: bu dizayn qaror — dasturchilarni o'zgaruvchini e'lon qilishdan oldin ishlatishdan himoya qilish. `var` dagi `undefined` muammosi ko'p bug'larga sabab bo'lgan — `let`/`const` TDZ orqali buni oldini oladi.

`let` va `const` farqi:
- `let` — TDZ tugagandan keyin qayta qiymat berish mumkin
- `const` — TDZ tugaganda qiymat beriladi, keyin qayta berish **mumkin emas** (re-assignment taqiq). Lekin const object/array **mutate** bo'lishi mumkin

### Kod Misollari

```javascript
// let — hoist bo'ladi, lekin TDZ da
console.log(age); // ❌ ReferenceError: Cannot access 'age' before initialization
let age = 25;
// let age scope boshida hoist bo'lgan, lekin initialize qilinmagan
// console.log gacha TDZ — o'qish taqiq

// const — xuddi let kabi TDZ
console.log(PI); // ❌ ReferenceError
const PI = 3.14;
```

```javascript
// const re-assignment taqiq, lekin mutation mumkin:
const user = { name: "Ali" };
user.name = "Vali";      // ✅ mutation — object content o'zgaradi
console.log(user.name);  // "Vali"

// user = { name: "New" }; // ❌ TypeError: Assignment to constant variable
// ✅ const reference'ni o'zgartirish taqiq — boshqa object'ga point qilib bo'lmaydi
// ✅ Lekin object ICHIDAGI property'larni o'zgartirish mumkin

const numbers = [1, 2, 3];
numbers.push(4);          // ✅ mutation — array content o'zgaradi
console.log(numbers);     // [1, 2, 3, 4]
// numbers = [5, 6];      // ❌ TypeError
```

---

## Temporal Dead Zone (TDZ)

### Nazariya

Temporal Dead Zone (TDZ) — `let`, `const` va `class` declaration'lar uchun scope boshlanganidan declaration qatoriga yetguncha bo'lgan zona. Bu zonada o'zgaruvchiga har qanday murojaat (o'qish yoki yozish) `ReferenceError` beradi.

TDZ **vaqt** (temporal) bo'yicha aniqlanadi — kod bajarilish tartibi bo'yicha, pozitsiya bo'yicha emas. Ya'ni, agar biror kod declaration'dan oldin **bajarilsa** — TDZ xatosi chiqadi, hatto kod declaration'dan **keyin** joylashgan bo'lsa ham.

TDZ qachon boshlanadi va tugaydi:
- **Boshlanadi:** Scope yaratilganda (function yoki block boshida)
- **Tugaydi:** `let`/`const` declaration qatori bajarilganda (qiymat berilganda)

### Under the Hood

```
let x = 10; satri bajarilish jarayoni:

1. Scope yaratilganda: x → <uninitialized> (TDZ BOSHLANADI)
2. ... (TDZ zonasi — x ga murojaat → ReferenceError) ...
3. let x = 10; qatori bajariladi:
   a. x → 10 (initialize + assign)
   b. TDZ TUGADI
4. ... (x erkin ishlatiladi) ...
```

TDZ vaqt bo'yicha ishlashini ko'rsatuvchi misol:

```javascript
// TDZ vaqt (temporal) bo'yicha, pozitsiya emas:
function example() {
  // TDZ boshlanadi (scope boshi)

  const getX = () => x; // ✅ Bu FUNCTION — hali bajarilMAYAPTI
  // getX TANASI ichida x bor, lekin funksiya hali CHAQIRILMAGAN
  // Shuning uchun hozir xato yo'q

  let x = 10; // TDZ TUGADI

  console.log(getX()); // ✅ 10 — endi getX chaqirilganda x allaqachon initialize
}

example();
```

```javascript
// TDZ typeof bilan ham ishlaydi:
console.log(typeof undeclaredVar); // "undefined" — e'lon qilinmagan → xato yo'q
console.log(typeof tdzVar);       // ❌ ReferenceError!
let tdzVar = 10;
// typeof odatda xato bermaydi (e'lon qilinmagan o'zgaruvchi uchun)
// Lekin TDZ dagi o'zgaruvchi uchun — ReferenceError beradi
// Bu let hoist bo'lishining yana bir isboti
```

### Kod Misollari

```javascript
// TDZ — amaliy misol, ko'p uchraydigan xato:
let count = 0;

function increment() {
  count++; // ❌ ReferenceError!
  let count = 1;
  // ✅ Sabab: local let count hoist bo'ldi
  // count++ bajarilganda local count TDZ da
  // Global count ga bormaydi — local count shadowing qiladi
}

// ✅ Tuzatish — local count ni oldin e'lon qilish:
function incrementFixed() {
  let count = 0;
  count++;
  return count; // 1
}
```

```javascript
// TDZ — default parameter'larda ham ishlaydi:
function example(a = b, b = 1) {
  return [a, b];
}

example();  // ❌ ReferenceError: Cannot access 'b' before initialization
// a = b bajarilganda b hali TDZ da — parameter'lar chapdan o'ngga initialize bo'ladi

// ✅ Tuzatish — tartibni o'zgartirish:
function exampleFixed(b = 1, a = b) {
  return [a, b];
}

exampleFixed(); // [1, 1] ✅
```

---

## Function Declaration Hoisting

### Nazariya

Function declaration'lar Creation Phase da **to'liq funksiya** bilan initialize qilinadi. Bu `var` (undefined) va `let`/`const` (uninitialized) dan farq qiladi — function declaration darhol chaqirishga tayyor.

Bu nima uchun shunday: JavaScript'ning dastlabki dizaynida funksiyalarni faylning istalgan joyida e'lon qilib, istalgan joyida chaqirish imkoniyati kerak edi. Shu sababli function declaration'lar to'liq hoist bo'ladi.

Qoidalar:
- **Function declaration** → to'liq hoist (e'lon qilishdan oldin chaqirish mumkin)
- **Function expression** → faqat o'zgaruvchi hoist bo'ladi (var → undefined, let/const → TDZ)
- **Arrow function** → function expression bilan bir xil (faqat o'zgaruvchi hoist)

### Kod Misollari

```javascript
// ✅ Function declaration — to'liq hoist
console.log(sum(2, 3)); // 5 — e'lon qilishdan oldin chaqirish mumkin

function sum(a, b) {
  return a + b;
}
// Creation Phase da: sum → function sum(a, b) { return a + b; }
```

```javascript
// ❌ Function expression — faqat var hoist
console.log(multiply(2, 3)); // TypeError: multiply is not a function

var multiply = function(a, b) {
  return a * b;
};
// Creation Phase da: multiply → undefined (var hoist)
// Execution da: multiply(2, 3) → undefined(2, 3) → TypeError!
// multiply MAVJUD (undefined), lekin function emas
```

```javascript
// ❌ Arrow function — function expression bilan bir xil
console.log(divide(10, 2)); // TypeError: divide is not a function

var divide = (a, b) => a / b;
// Creation Phase da: divide → undefined (var hoist)
```

```javascript
// ❌ let/const bilan function expression — TDZ
console.log(greet("Ali")); // ReferenceError: Cannot access 'greet'

const greet = function(name) {
  return "Hello, " + name;
};
// Creation Phase da: greet → <uninitialized> (const — TDZ)
```

---

## Function Expression Hoisting

### Nazariya

Function expression — bu o'zgaruvchiga assign qilingan funksiya. Hoisting nuqtai nazaridan, function expression **faqat o'zgaruvchi sifatida** hoist bo'ladi — funksiya qismi hoist bo'lMAYDI.

```javascript
var getName = function() { return "Ali"; };     // var → undefined hoist
let getName = function() { return "Ali"; };     // let → TDZ hoist
const getName = function() { return "Ali"; };   // const → TDZ hoist
const getName = () => "Ali";                     // const → TDZ hoist
```

Barcha hollarda o'zgaruvchi hoist bo'ladi, lekin funksiya qiymati faqat Execution Phase da, declaration qatoriga yetganda assign bo'ladi.

### Kod Misollari

```javascript
// Named function expression — nom faqat funksiya ICHIDA accessible:
var calculate = function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1); // ✅ factorial — funksiya ichida accessible
};

console.log(calculate(5));   // 120 ✅
// console.log(factorial(5)); // ❌ ReferenceError — tashqarida yo'q!
// Named function expression nomi funksiya tashqarisida ko'rinmaydi
// Bu faqat debugging va recursion uchun foydali
```

---

## var vs let vs const — To'liq Taqqoslash

### Nazariya

| Xususiyat | `var` | `let` | `const` |
|-----------|-------|-------|---------|
| **Scope** | Function | Block | Block |
| **Hoist bo'ladimi** | Ha | Ha | Ha |
| **Initialize** | `undefined` | `uninitialized` (TDZ) | `uninitialized` (TDZ) |
| **TDZ** | Yo'q | Ha | Ha |
| **Re-declaration** | Mumkin ✅ | Taqiq ❌ | Taqiq ❌ |
| **Re-assignment** | Mumkin ✅ | Mumkin ✅ | Taqiq ❌ |
| **Global object property** | Ha (`window.x`) | Yo'q | Yo'q |
| **Block ichida** | Function scope'ga chiqadi | Block'da qoladi | Block'da qoladi |

### Kod Misollari

```javascript
// Scope farqi:
function scopeTest() {
  if (true) {
    var a = 1;    // function scope — if tashqarisida ham bor
    let b = 2;    // block scope — faqat if ichida
    const c = 3;  // block scope — faqat if ichida
  }

  console.log(a); // 1 ✅
  console.log(b); // ❌ ReferenceError
  console.log(c); // ❌ ReferenceError
}
```

```javascript
// Loop da var vs let — klassik muammo:
// ❌ var bilan — barcha callback'lar bitta i ni ko'radi
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3
// ❌ Sabab: var function-scoped, bitta i — loop tugaganda i = 3
// setTimeout callback'lari 100ms keyin bajarilganda — i allaqachon 3

// ✅ let bilan — har iteratsiyada yangi i
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2
// ✅ Sabab: let block-scoped, har bir iteratsiya yangi block = yangi i
// Har bir callback o'z iteratsiyasidagi i ni eslab qoladi (closure)
```

```javascript
// Zamonaviy yondashuv — qachon nimani ishlatish:
// const — default tanlovi. Qayta assign kerak bo'lmaguncha const
const API_URL = "https://api.example.com";
const user = { name: "Ali" }; // object mumkin, reference o'zgarmaydi

// let — faqat qayta assign kerak bo'lganda
let count = 0;
count++;

// var — ISHLATMANG (legacy kod bilan ishlashdan tashqari)
// var function scope va hoisting muammolari keltirib chiqaradi
```

---

## Hoisting Priority — Function vs Variable

### Nazariya

Agar bir xil nomda `function declaration` va `var` e'lon qilinsa — **function declaration yuqori priority**ga ega. Function declaration `var` dan oldin environment record'ga yoziladi.

Qoidalar:
1. Function declaration'lar birinchi qayta ishlanadi
2. Keyin var declaration'lar qayta ishlanadi
3. Agar nom allaqachon mavjud bo'lsa — var uni **override qilMAYDI** (function qoladi)
4. Lekin Execution Phase da assignment barini override qilishi mumkin

### Under the Hood

```javascript
console.log(foo); // ?

var foo = "string";
function foo() { return "function"; }

console.log(foo); // ?
```

Creation Phase qadamlari:

```
1. Function declaration qayta ishlanadi:
   foo → function foo() { return "function"; }

2. var declaration qayta ishlanadi:
   foo allaqachon mavjud (function) → var uni override qilMAYDI
   (var faqat yangi binding yaratadi, mavjud bo'lsa skip qiladi)

3. Creation Phase natijasi:
   foo → function foo() { return "function"; }

EXECUTION PHASE:
  console.log(foo) → function foo() {...}
  foo = "string"   → foo endi "string" (assignment override qiladi!)
  console.log(foo) → "string"
```

### Kod Misollari

```javascript
// Function vs var — function yutadi:
console.log(x); // function x() { return 1; }

var x = 10;
function x() { return 1; }

console.log(x); // 10 — assignment override qildi

// Creation Phase: x = function (function > var)
// Execution: console.log(x) → function
// Execution: x = 10 (assignment)
// Execution: console.log(x) → 10
```

```javascript
// Ikki function declaration — oxirgi yutadi:
console.log(greet()); // "Second"

function greet() { return "First"; }
function greet() { return "Second"; }
// ✅ Ikkinchi function declaration birinchini override qiladi
// Ikkalasi ham Creation Phase da qayta ishlanadi — oxirgisi saqlanadi
```

---

## Class Hoisting

### Nazariya

`class` declaration'lar `let`/`const` kabi hoist bo'ladi — ya'ni engine scope boshida class'ni biladi, lekin **TDZ** da qoladi. Class'ni e'lon qilishdan oldin ishlatish `ReferenceError` beradi.

Bu function declaration'dan farq qiladi — function declaration'ni e'lon qilishdan oldin chaqirish mumkin, class'ni esa mumkin emas.

Class expression'lar ham xuddi function expression kabi — faqat o'zgaruvchi hoist bo'ladi.

### Kod Misollari

```javascript
// ❌ Class declaration — TDZ
const user = new User("Ali"); // ReferenceError: Cannot access 'User'

class User {
  constructor(name) {
    this.name = name;
  }
}
// class hoist bo'ladi, lekin TDZ da — function declaration kabi emas!
```

```javascript
// ✅ To'g'ri tartib — avval class, keyin ishlatish
class Product {
  constructor(name, price) {
    this.name = name;
    this.price = price;
  }
}

const phone = new Product("iPhone", 999); // ✅
```

```javascript
// Class expression — function expression bilan bir xil:
const car = new Vehicle("BMW"); // ❌ ReferenceError (const — TDZ)

const Vehicle = class {
  constructor(brand) {
    this.brand = brand;
  }
};
```

Bu haqda to'liq [08-classes.md](08-classes.md) da.

---

## Common Mistakes

### ❌ Xato 1: var hoisting'ga ishonib o'zgaruvchi e'lon qilishdan oldin ishlatish

```javascript
// ❌ Noto'g'ri — var hoisting'dan foydalanish
function getDiscount(price) {
  if (price > 100) {
    discount = 0.2; // ❌ var hoist tufayli ishlaydi, lekin xavfli
  }
  var discount = 0;
  return price * (1 - discount);
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ O'zgaruvchini scope boshida e'lon qilish
function getDiscount(price) {
  let discount = 0;
  if (price > 100) {
    discount = 0.2;
  }
  return price * (1 - discount);
}
```

**Nima uchun:** var hoisting'dan foydalanish kodni o'qib tushunishni qiyinlashtiradi. O'zgaruvchini ishlatishdan oldin e'lon qilish — kodni aniq va predictable qiladi.

---

### ❌ Xato 2: let/const hoist bo'lmaydi deb o'ylash

```javascript
// ❌ Noto'g'ri tushuncha: "let hoist bo'lmaydi, shuning uchun global x ko'rinadi"
let x = "global";

function test() {
  console.log(x); // ❌ ReferenceError — "global" EMAS!
  let x = "local";
}
// Agar let hoist bo'lmaganida — console.log global x ni ko'rsatar edi
```

### ✅ To'g'ri tushunish:

```javascript
// ✅ let HOIST BO'LADI, lekin TDZ da — shuning uchun ReferenceError
// Engine local let x ni BILADI (hoist) — shuning uchun global x ga bormaydi
// Lekin local x hali initialize qilinMAGAN (TDZ) — shuning uchun xato

let x = "global";

function test() {
  let x = "local"; // ✅ avval e'lon qilish, keyin ishlatish
  console.log(x);  // "local"
}
```

**Nima uchun:** `let`/`const` hoist bo'ladi (engine scope boshida ularni biladi), lekin `undefined` bilan emas, `uninitialized` bilan. Bu TDZ ni yaratadi — declaration qatoriga yetguncha murojaat taqiq.

---

### ❌ Xato 3: Function expression'ni declaration kabi chaqirish

```javascript
// ❌ Noto'g'ri — expression hoist bo'lmaydi
calculate(10, 20);

var calculate = function(a, b) {
  return a + b;
};
// TypeError: calculate is not a function
// Creation Phase da: calculate → undefined (var hoist)
// undefined(10, 20) → TypeError
```

### ✅ To'g'ri usul:

```javascript
// ✅ Function declaration ishlatish (agar hoist kerak bo'lsa):
calculate(10, 20); // ✅ 30

function calculate(a, b) {
  return a + b;
}

// ✅ Yoki expression'ni chaqirishdan OLDIN e'lon qilish:
const calculate = (a, b) => a + b;
console.log(calculate(10, 20)); // 30
```

**Nima uchun:** Function expression'larda faqat o'zgaruvchi hoist bo'ladi (`undefined`), funksiya emas. Declaration kerak yoki chaqirishdan oldin yozish kerak.

---

### ❌ Xato 4: for loop da var ishlatish

```javascript
// ❌ Noto'g'ri — var function-scoped, bitta i barcha callback'lar uchun
for (var i = 0; i < 5; i++) {
  setTimeout(function() {
    console.log(i); // 5, 5, 5, 5, 5
  }, i * 100);
}
// ❌ Loop tugaganda i = 5, barcha callback'lar shu bitta i ni ko'radi
```

### ✅ To'g'ri usul:

```javascript
// ✅ let ishlatish — har iteratsiyada yangi binding
for (let i = 0; i < 5; i++) {
  setTimeout(function() {
    console.log(i); // 0, 1, 2, 3, 4
  }, i * 100);
}
// ✅ let block-scoped — har bir iteratsiya yangi i yaratadi
// Har bir setTimeout callback o'z i sini eslab qoladi (closure)
```

**Nima uchun:** `var` function-scoped — butun loop bitta `i`. `let` block-scoped — har bir iteratsiya block yangi `i` yaratadi. setTimeout callback'lari keyinroq bajarilganda — `var` bilan oxirgi qiymatni, `let` bilan o'z iteratsiyasidagi qiymatni ko'radi.

---

### ❌ Xato 5: const bilan TDZ'da default parameter ishlatish

```javascript
// ❌ Noto'g'ri — parameter'lar chapdan o'ngga initialize bo'ladi
function createConfig(timeout = retries * 1000, retries = 3) {
  return { timeout, retries };
}

createConfig(); // ❌ ReferenceError: Cannot access 'retries' before initialization
// timeout = retries * 1000 bajarilganda retries hali TDZ da
```

### ✅ To'g'ri usul:

```javascript
// ✅ Parameter tartibini to'g'irlash
function createConfig(retries = 3, timeout = retries * 1000) {
  return { timeout, retries };
}

createConfig(); // { timeout: 3000, retries: 3 } ✅
// retries avval initialize bo'ladi, keyin timeout undan foydalana oladi
```

**Nima uchun:** Default parameter'lar chapdan o'ngga evaluate qilinadi. Har bir parameter oldingilariga murojaat qilishi mumkin, lekin keyingilariga — TDZ tufayli — mumkin emas.

---

## Amaliy Mashqlar

### Mashq 1: Hoisting Aniqlash (Oson)

**Savol:** Har bir console.log ning natijasini ayting:

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

```
undefined
ReferenceError: Cannot access 'b' before initialization
```

Kod 2-qatorda ReferenceError tufayli to'xtaydi — 3-qator bajarilmaydi.

- `var a` → Creation Phase da `undefined` bilan initialize → o'qish mumkin
- `let b` → Creation Phase da `uninitialized` (TDZ) → o'qish `ReferenceError`
- `const c` → shu qatorga yetmaydi (oldingi xato tufayli)
</details>

---

### Mashq 2: Function vs Variable Hoisting (O'rta)

**Savol:** Output nima?

```javascript
console.log(typeof foo);
var foo = "bar";
function foo() {}
console.log(typeof foo);
```

<details>
<summary>Javob</summary>

```
"function"
"string"
```

Creation Phase:
1. `function foo()` → foo = function (function declaration birinchi)
2. `var foo` → foo allaqachon mavjud → **skip** (override qilmaydi)

Execution Phase:
1. `typeof foo` → `"function"` (Creation Phase natijasi)
2. `foo = "bar"` → foo endi string
3. `typeof foo` → `"string"`
</details>

---

### Mashq 3: TDZ (O'rta)

**Savol:** Bu kod ishlaydimi? Nima uchun?

```javascript
let x = x + 1;
console.log(x);
```

<details>
<summary>Javob</summary>

```
ReferenceError: Cannot access 'x' before initialization
```

`let x = x + 1;` bajarilganda:
- `=` ning o'ng tomoni avval evaluate qilinadi: `x + 1`
- Bu paytda `x` hali initialize qilinmagan (TDZ da)
- `x` ni o'qishga urinish → `ReferenceError`

`var` bilan bo'lganida:
```javascript
var x = x + 1;
console.log(x); // NaN
// var x Creation Phase da undefined
// x + 1 = undefined + 1 = NaN
```
</details>

---

### Mashq 4: Scope va Hoisting Birgalikda (Qiyin)

**Savol:** Output nima?

```javascript
var x = 1;

function test() {
  console.log(x);

  if (true) {
    var x = 2;
  }

  console.log(x);
}

test();
console.log(x);
```

<details>
<summary>Javob</summary>

```
undefined
2
1
```

- `test()` Creation Phase: `var x = undefined` (function scope'ga hoist — if ichidagi var)
- `console.log(x)` → local `x` = undefined (global `x` shadowed)
- `if (true)` ichida: `x = 2` → local `x` endi 2
- `console.log(x)` → 2
- Global `console.log(x)` → 1 (global `x` o'zgarmagan)

if ichidagi `var x = 2` — bu function scope'ga hoist bo'ladi. Shuning uchun test() ichida birinchi console.log global `x` (1) ni emas, local `x` (undefined) ni ko'radi.
</details>

---

### Mashq 5: Murakkab Hoisting (Qiyin)

**Savol:** Output nima?

```javascript
function outer() {
  console.log(a);
  console.log(foo());

  var a = 1;

  function foo() {
    return 2;
  }

  console.log(a);
}

outer();
```

<details>
<summary>Javob</summary>

```
undefined
2
1
```

outer() Creation Phase:
- `var a = undefined`
- `function foo = function foo() { return 2; }`

Execution Phase:
1. `console.log(a)` → undefined (var hoist)
2. `console.log(foo())` → foo() chaqirildi → 2 qaytardi → 2 (function to'liq hoist)
3. `a = 1` → a endi 1
4. `console.log(a)` → 1
</details>

---

## Xulosa

Bu bo'limda Hoisting mexanizmini chuqur o'rgandik:

- **Hoisting** — Creation Phase da declaration'larning scope'da registratsiya qilinishi. Kod fizik ko'chirilmaydi
- **var** → `undefined` bilan initialize (o'qish mumkin, lekin qiymati undefined)
- **let/const** → `uninitialized` (TDZ da — o'qish ReferenceError)
- **function declaration** → to'liq funksiya bilan initialize (chaqirishga tayyor)
- **function expression** → faqat o'zgaruvchi hoist (var → undefined, let/const → TDZ)
- **TDZ** — scope boshidan declaration qatoriga qadar zona, murojaat taqiq
- **Priority** — function declaration > var (bir xil nomda bo'lsa function yutadi)
- **class** → let/const kabi TDZ da (function declaration'dan farqli)
- **Zamonaviy yondashuv** — `const` default, `let` kerak bo'lganda, `var` ishlatmang

Hoisting mexanizmini tushunish scope chain'ni tushunish uchun asos — keyingi bo'limda scope chain va lexical scoping'ni chuqur o'rganamiz.

---

**Keyingi bo'lim:** [04-scope.md](04-scope.md) — Scope Chain, lexical vs dynamic scoping, variable lookup mexanizmi, strict mode, variable shadowing.
