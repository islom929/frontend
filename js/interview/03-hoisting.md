# Hoisting — Interview Savollari

> Hoisting mexanizmi bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.

---

## Nazariy savollar

### 1. Hoisting nima? [Junior+]

<details>
<summary>Javob</summary>

Hoisting — JavaScript engine'ning Creation Phase da declaration'larni scope'da registratsiya qilish mexanizmi. Kod fizik ravishda "ko'tarilmaydi" — engine execution context yaratilganda barcha declaration'larni topadi va environment record'ga yozadi.

`var` → `undefined` bilan initialize (o'qish mumkin), `let`/`const` → `uninitialized` (TDZ — o'qish ReferenceError), function declaration → to'liq funksiya bilan initialize (chaqirishga tayyor).

```javascript
console.log(x);   // undefined — var hoist, undefined bilan
console.log(y);   // ReferenceError — let hoist, lekin TDZ da
greet();           // "Hello" — function declaration to'liq hoist

var x = 10;
let y = 20;
function greet() { console.log("Hello"); }
```

Aslida "hoisting" — bu Creation Phase dagi early binding. Muhim farq: **parsing** bosqichida (AST yaratishda) engine declaration'larni static topadi va saqlab qo'yadi, keyin **Creation Phase**'da (har EC yaratilganda) parser'ning metadata'si asosida Environment Record'larni runtime'da to'ldiradi (`CreateMutableBinding` + `InitializeBinding` chaqiriladi). Ikki bosqich alohida: parsing bir marta bajariladi, Creation Phase har EC uchun.

</details>

### 2. let va const hoist bo'ladimi? [Middle]

<details>
<summary>Javob</summary>

Ha, `let` va `const` **hoist bo'ladi** — lekin `var` dan farqli ravishda ular `undefined` bilan emas, `uninitialized` holatda saqlanadi. Bu **Temporal Dead Zone (TDZ)** deb ataladi — scope boshidan declaration qatoriga qadar o'zgaruvchiga murojaat taqiq.

```javascript
// let hoist bo'lishining isboti:
let x = "global";

function test() {
  console.log(x); // ❌ ReferenceError — "global" emas!
  let x = "local";
}

test();
// Agar let hoist bo'lmaganida — global x ko'rinar edi
// Lekin ReferenceError — demak engine local let x ni BILADI (hoist)
// Lekin hali initialize qilinMAGAN (TDZ) — shuning uchun xato
```

| | `var` | `let` / `const` |
|---|-------|-----------------|
| Hoist | Ha | Ha |
| Initialize | `undefined` | `uninitialized` (TDZ) |
| E'londan oldin o'qish | `undefined` qaytaradi | `ReferenceError` beradi |

</details>

### 3. TDZ (Temporal Dead Zone) nima? [Middle]

<details>
<summary>Javob</summary>

TDZ — scope boshlanganidan `let`/`const`/`class` declaration qatoriga qadar bo'lgan zona. Bu zonada o'zgaruvchiga har qanday murojaat (o'qish yoki yozish) `ReferenceError` beradi.

TDZ **vaqt** (temporal) bo'yicha aniqlanadi — kod bajarilish tartibi bo'yicha:

```javascript
// TDZ vaqt bo'yicha — pozitsiya emas:
function example() {
  const getVal = () => value; // ✅ funksiya TANASI — hali bajarilmaydi

  let value = 42;             // TDZ tugadi

  console.log(getVal());      // 42 ✅ — endi bajarilganda value tayyor
}

example();
```

TDZ typeof bilan ham ishlaydi:
```javascript
console.log(typeof x);         // "undefined" — e'lon qilinmagan → xato yo'q
console.log(typeof tdzVar);    // ❌ ReferenceError!
let tdzVar = 10;
// typeof odatda xato bermaydi, lekin TDZ dagi o'zgaruvchi uchun beradi
```

</details>

### 4. Function declaration vs function expression hoisting farqi nima? [Junior+]

<details>
<summary>Javob</summary>

**Function declaration** Creation Phase da to'liq funksiya bilan initialize qilinadi — e'lon qilishdan oldin chaqirish mumkin. **Function expression** da faqat o'zgaruvchi hoist bo'ladi — funksiya Execution Phase da assign bo'ladi.

```javascript
// ✅ Function declaration — to'liq hoist
console.log(sum(2, 3)); // 5

function sum(a, b) {
  return a + b;
}

// ❌ Function expression — faqat var/let/const hoist
console.log(multiply(2, 3)); // TypeError: multiply is not a function

var multiply = function(a, b) {
  return a * b;
};
// Creation Phase: multiply = undefined (var hoist)
// multiply(2, 3) → undefined(2, 3) → TypeError
```

Muhim farq: `var` bilan expression **TypeError** beradi (undefined mavjud, lekin function emas), `let`/`const` bilan esa **ReferenceError** beradi (TDZ).

</details>

### 5. var vs let vs const farqlarini jadval bilan tushuntiring [Junior+]

<details>
<summary>Javob</summary>

| Xususiyat | `var` | `let` | `const` |
|-----------|-------|-------|---------|
| Scope | Function | Block | Block |
| Hoist | Ha (`undefined`) | Ha (TDZ) | Ha (TDZ) |
| Re-declaration | ✅ Mumkin | ❌ SyntaxError | ❌ SyntaxError |
| Re-assignment | ✅ Mumkin | ✅ Mumkin | ❌ TypeError |
| `window` property | ✅ Ha | ❌ Yo'q | ❌ Yo'q |

```javascript
// Zamonaviy standard:
// const — default tanlovi (90%+ hollarda)
const API_URL = "https://api.example.com";
const users = []; // ✅ array content o'zgarishi mumkin

// let — faqat qayta assign kerak bo'lganda
let count = 0;
count++; // ✅

// var — ishlatMANG (faqat legacy kod bilan)
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle]

```javascript
console.log(a);
console.log(b);

var a = 1;
let b = 2;

function c() {
  console.log(a);
  console.log(b);
}

c();
```

<details>
<summary>Javob</summary>

```
undefined
ReferenceError: Cannot access 'b' before initialization
```

Kod 2-qatorda to'xtaydi — ReferenceError tufayli qolgan qatorlar bajarilmaydi.

- `var a` → Creation Phase da `undefined` bilan initialize → console.log(a) = undefined
- `let b` → Creation Phase da `uninitialized` (TDZ) → console.log(b) = ReferenceError
- `c()` ga hech qachon yetmaydi

</details>

### 2. Quyidagi kodning output'ini ayting [Middle+]

```javascript
var x = 1;

function showValue() {
  console.log(x);
  var x = 2;
}

showValue();
```

<details>
<summary>Javob</summary>

```
undefined
```

`showValue()` da local `var x` bor — bu function scope'ga hoist bo'ladi va `undefined` bilan initialize qilinadi. `console.log(x)` bajarilganda engine **local** `x` ni topadi (undefined) — global `x` (1) ga **bormaydi**. Bu variable shadowing — local scope global'ni yashiradi.

```
showValue() Creation Phase:
  var x = undefined (local)

showValue() Execution Phase:
  console.log(x) → local x = undefined (global x shadowed)
  x = 2 → local x endi 2
```

</details>

### 3. Quyidagi kodning output'ini ayting [Middle+]

```javascript
function test() {
  greet();
  notify();

  function greet() {
    console.log("greet");
  }

  var notify = function() {
    console.log("notify");
  };
}

test();
```

<details>
<summary>Javob</summary>

```
greet
TypeError: notify is not a function
```

test() Creation Phase:
- `greet` → `function greet() {...}` (function declaration — to'liq hoist)
- `notify` → `undefined` (var hoist — function expression hoist bo'lmaydi)

Execution Phase:
- `greet()` → ✅ ishlaydi, "greet" chiqadi
- `notify()` → ❌ `undefined()` → TypeError: notify is not a function

</details>

### 4. Quyidagi kodning output'ini ayting [Senior]

```javascript
console.log(value);

var value = 10;

function value() {
  return 20;
}

var value = 30;

console.log(value);
```

<details>
<summary>Javob</summary>

```
[Function: value]
30
```

> **Eslatma:** Birinchi `console.log` natijasi Node.js'da `[Function: value]`, Chrome DevTools'da `ƒ value() { return 20; }` sifatida ko'rinadi. Quyida bajarilish tartibi batafsil tushuntiriladi.

Creation Phase (spec `FunctionDeclarationInstantiation` algoritmi):
1. **Var declaration'lar avval** qayta ishlanadi:
   - `var value = 10` → `CreateMutableBinding(value)` + `InitializeBinding(undefined)` → `value = undefined`
   - `var value = 30` → binding allaqachon mavjud → **skip** (duplicate var ignored)
2. **Function declaration'lar keyin** qayta ishlanadi:
   - `function value()` → mavjud `value` binding'ni `SetMutableBinding` bilan **override** qiladi → `value = function value() {...}`

Creation Phase oxiri: `value = function value() {...}` (function wins)

Execution Phase:
1. `console.log(value)` → `function value() {...}` (Creation Phase natijasi)
2. `value = 10` (assignment — oddiy `PutValue`, override)
3. Function declaration — Execution'da hech narsa qilmaydi (allaqachon Creation'da initialize bo'lgan)
4. `value = 30` (assignment — oddiy `PutValue`, override)
5. `console.log(value)` → 30

**Qoida:** `FunctionDeclarationInstantiation` algoritmida function declaration'lar var declaration'lardan **keyin** qayta ishlanadi va ularni override qiladi (spec 29.f-g qadamlar). Execution Phase'da esa assignment'lar tartib bo'yicha oddiy `PutValue` orqali ishlaydi — hoisting yo'q.

**Deep Dive:**

ECMAScript spec'dagi `FunctionDeclarationInstantiation` algoritmida function declaration'lar `var` declaration'lardan **keyin** qayta ishlanadi — shuning uchun bir xil nomdagi binding'ni override qiladi. Spec'da aniq yozilgan: agar allaqachon mavjud binding bo'lsa va u function declaration emas — function declaration uni yozib tashlaydi (`SetMutableBinding`). Lekin Execution Phase'da assignment'lar oddiy `PutValue` orqali tartib bo'yicha ishlaydi.

</details>

### 5. Bu kodda xato bor. Toping va tuzating [Middle]

```javascript
function getItems() {
  var items = [];

  for (var i = 0; i < 3; i++) {
    items.push(function() {
      return i;
    });
  }

  return items;
}

const fns = getItems();
console.log(fns[0]()); // kutilgan: 0
console.log(fns[1]()); // kutilgan: 1
console.log(fns[2]()); // kutilgan: 2
```

<details>
<summary>Javob</summary>

Haqiqiy output: `3, 3, 3` — kutilgan emas!

Muammo: `var i` function-scoped — bitta `i` barcha callback'lar uchun. Loop tugaganda `i = 3`. Barcha callback'lar shu bitta `i` ga reference qiladi.

```javascript
// ✅ Tuzatilgan — let ishlatish
function getItems() {
  const items = [];

  for (let i = 0; i < 3; i++) {
    // ✅ let block-scoped — har iteratsiyada yangi i
    items.push(function() {
      return i; // har bir callback o'z i sini eslab qoladi
    });
  }

  return items;
}

const fns = getItems();
console.log(fns[0]()); // 0 ✅
console.log(fns[1]()); // 1 ✅
console.log(fns[2]()); // 2 ✅
```

Bu klassik closure + var muammosi — `let` block-scoped bo'lgani uchun har iteratsiyada yangi binding yaratadi va har bir callback o'z qiymatini eslab qoladi.

</details>

### 6. Quyidagi kodning output'ini ayting [Senior]

```javascript
let a = 1;

{
  console.log(a);
  let a = 2;
}
```

<details>
<summary>Javob</summary>

```
ReferenceError: Cannot access 'a' before initialization
```

Block `{}` ichida `let a = 2` bor — bu block scope'da yangi `a` yaratadi. Bu yangi `a` hoist bo'ladi va TDZ ga tushadi. `console.log(a)` bajarilganda — block scope'dagi `a` topiladi (hoist tufayli), lekin TDZ da — shuning uchun ReferenceError.

Tashqi `let a = 1` ga bormaydi — chunki ichki `a` shadowing qiladi. Agar `let a = 2` bo'lmaganida — tashqi `a = 1` ko'rinar edi.

**Deep Dive:**

Spec bo'yicha `let`/`const` binding'lar `CreateMutableBinding` (yoki `CreateImmutableBinding`) bilan yaratiladi, lekin `InitializeBinding` **chaqirilmaydi** — declaration qatoriga yetgunicha. TDZ davomida binding mavjud, lekin `uninitialized` holatda. `GetBindingValue` chaqirilganda binding hali initialized bo'lmasa — spec aniq `ReferenceError` throw qilishni talab qiladi.

</details>

### 7. Hoisting va class — output savol [Middle+]

```javascript
const instance = new MyClass();

class MyClass {
  constructor() {
    this.name = "test";
  }
}
```

<details>
<summary>Javob</summary>

```
ReferenceError: Cannot access 'MyClass' before initialization
```

`class` declaration `let`/`const` kabi TDZ da hoist bo'ladi — function declaration kabi to'liq hoist bo'lMAYDI. Class'ni e'lon qilishdan oldin ishlatish ReferenceError beradi.

```javascript
// ✅ To'g'ri tartib:
class MyClass {
  constructor() {
    this.name = "test";
  }
}

const instance = new MyClass(); // ✅
```

Bu function declaration bilan asosiy farq — function declaration to'liq hoist bo'ladi, class esa TDZ da.

</details>
