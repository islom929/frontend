# Hoisting — Interview Savollari

> Hoisting mexanizmi bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.

---

## Savol 1: Hoisting nima? [Junior+]

**Javob:**

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

Aslida "hoisting" — bu Creation Phase dagi early binding. Engine parse vaqtida declaration'larni topadi va ularni environment record'ga yozadi — kod bajarilishidan oldin.

---

## Savol 2: let va const hoist bo'ladimi? [Middle]

**Javob:**

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

---

## Savol 3: TDZ (Temporal Dead Zone) nima? [Middle]

**Javob:**

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

---

## Savol 4: Quyidagi kodning output'ini ayting [Middle]

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

**Javob:**

```
undefined
ReferenceError: Cannot access 'b' before initialization
```

Kod 2-qatorda to'xtaydi — ReferenceError tufayli qolgan qatorlar bajarilmaydi.

- `var a` → Creation Phase da `undefined` bilan initialize → console.log(a) = undefined
- `let b` → Creation Phase da `uninitialized` (TDZ) → console.log(b) = ReferenceError
- `c()` ga hech qachon yetmaydi

---

## Savol 5: Quyidagi kodning output'ini ayting [Middle+]

```javascript
var x = 1;

function foo() {
  console.log(x);
  var x = 2;
}

foo();
```

**Javob:**

```
undefined
```

`foo()` da local `var x` bor — bu function scope'ga hoist bo'ladi va `undefined` bilan initialize qilinadi. `console.log(x)` bajarilganda engine **local** `x` ni topadi (undefined) — global `x` (1) ga **bormaydi**. Bu variable shadowing — local scope global'ni yashiradi.

```
foo() Creation Phase:
  var x = undefined (local)

foo() Execution Phase:
  console.log(x) → local x = undefined (global x shadowed)
  x = 2 → local x endi 2
```

---

## Savol 6: Function declaration vs function expression hoisting farqi nima? [Junior+]

**Javob:**

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

---

## Savol 7: Quyidagi kodning output'ini ayting [Middle+]

```javascript
function test() {
  foo();
  bar();

  function foo() {
    console.log("foo");
  }

  var bar = function() {
    console.log("bar");
  };
}

test();
```

**Javob:**

```
foo
TypeError: bar is not a function
```

test() Creation Phase:
- `foo` → `function foo() {...}` (function declaration — to'liq hoist)
- `bar` → `undefined` (var hoist — function expression hoist bo'lmaydi)

Execution Phase:
- `foo()` → ✅ ishlaydi, "foo" chiqadi
- `bar()` → ❌ `undefined()` → TypeError: bar is not a function

---

## Savol 8: Quyidagi kodning output'ini ayting [Senior]

```javascript
console.log(foo);

var foo = 10;

function foo() {
  return 20;
}

var foo = 30;

console.log(foo);
```

**Javob:**

```
function foo() { return 20; }
30
```

Creation Phase:
1. `function foo()` qayta ishlanadi → `foo = function foo() {...}`
2. `var foo = 10` → foo allaqachon mavjud → **skip** (var override qilmaydi)
3. `var foo = 30` → foo allaqachon mavjud → **skip**

Execution Phase:
1. `console.log(foo)` → `function foo() {...}` (Creation Phase natijasi)
2. `foo = 10` (assignment — override)
3. (function declaration — Execution da hech narsa qilmaydi, allaqachon hoist bo'lgan)
4. `foo = 30` (assignment — override)
5. `console.log(foo)` → 30

Qoida: function declaration Creation Phase da `var` dan ustun. Lekin Execution Phase da assignment'lar tartib bo'yicha override qiladi.

---

## Savol 9: var vs let vs const farqlarini jadval bilan tushuntiring [Junior+]

**Javob:**

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

---

## Savol 10: Bu kodda xato bor. Toping va tuzating [Middle]

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

**Javob:**

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

---

## Savol 11: Quyidagi kodning output'ini ayting [Senior]

```javascript
let a = 1;

{
  console.log(a);
  let a = 2;
}
```

**Javob:**

```
ReferenceError: Cannot access 'a' before initialization
```

Block `{}` ichida `let a = 2` bor — bu block scope'da yangi `a` yaratadi. Bu yangi `a` hoist bo'ladi va TDZ ga tushadi. `console.log(a)` bajarilganda — block scope'dagi `a` topiladi (hoist tufayli), lekin TDZ da — shuning uchun ReferenceError.

Tashqi `let a = 1` ga bormaydi — chunki ichki `a` shadowing qiladi. Agar `let a = 2` bo'lmaganida — tashqi `a = 1` ko'rinar edi.

---

## Savol 12: Hoisting va class — output savol [Middle+]

```javascript
const instance = new MyClass();

class MyClass {
  constructor() {
    this.name = "test";
  }
}
```

**Javob:**

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

---
