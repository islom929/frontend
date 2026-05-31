# Hoisting — Interview Savollari

> Hoisting mexanizmi bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.

---

## Nazariy savollar

### 1. Hoisting nima? [Junior+]

<details>
<summary>Javob</summary>

Hoisting — JavaScript engine'ning Creation Phase da declaration'larni scope'da registration qilish mexanizmi. Kod fizik ravishda "ko'tarilmaydi" — engine execution context yaratilganda barcha declaration'larni topadi va environment record'ga yozadi.

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
// const — default tanlovi (reassignment kerak bo'lmasa)
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

**Qoida:** `FunctionDeclarationInstantiation` algoritmida function declaration'lar var declaration'lardan **keyin** qayta ishlanadi va ularni override qiladi (`functionsToInitialize` ro'yxati var binding'lar yaratilgandan so'ng `SetMutableBinding` bilan o'tkaziladi). Execution Phase'da esa assignment'lar tartib bo'yicha oddiy `PutValue` orqali ishlaydi — hoisting yo'q.

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
<summary><strong>Javob</strong></summary>

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

### 8. Default parameter'larda TDZ qanday ishlaydi? [Middle+]

```javascript
function createConfig(timeout = retries * 1000, retries = 3) {
  return { timeout, retries };
}

console.log(createConfig());
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob
`ReferenceError: Cannot access 'retries' before initialization` — parameter'lar chapdan o'ngga initialize bo'ladi, har parameter o'z TDZ'siga ega.

### To'liq tushuntirish

Function parameter'lar **chapdan o'ngga** ketma-ket initialize qilinadi. Har bir parameter o'z TDZ zonasiga ega: undan **oldingi** parameter'lar mavjud va initialize qilingan, **keyingi** parameter'lar esa hali `uninitialized` (TDZ).

`createConfig()` chaqirilganda:
1. Engine `timeout` default'ini evaluate qiladi: `retries * 1000`
2. `retries` hali initialize qilinmagan — keyingi parameter
3. TDZ'dagi `retries` ga murojaat → `ReferenceError`

### Kod misol

```javascript
// ❌ Noto'g'ri tartib — keyingi parameter'ga reference
function bad(a = b, b = 1) {
  return [a, b];
}
bad(); // ❌ ReferenceError: Cannot access 'b' before initialization

// ✅ To'g'ri tartib — dependent parameter keyin
function good(b = 1, a = b) {
  return [a, b];
}
good(); // [1, 1] ✅
good(10); // [10, 10] — b = 10, a = b = 10
good(10, 20); // [20, 10]
```

### Edge Cases

- Parameter scope va function body — alohida scope qatlamlari. Body'dagi `let`/`const` parameter default'da ko'rinmaydi:
  ```javascript
  function example(x = y) {
    const y = 10; // body scope'da
    return x;
  }
  example(); // ❌ ReferenceError: y is not defined
  // (TDZ emas — y umuman parameter scope'da yo'q)
  ```
- Default parameter expression'da `this`, `arguments` ishlatish mumkin (parameter scope'dan accessible)
- Default `undefined` argument'ni trigger qiladi: `bad(undefined, 5)` ham xato beradi — `undefined` default'ni faollashtiradi

### Follow-up savollar

1. "Default parameter `var` declaration bilan kolleziya bo'lsa?" — Parameter Creation Phase'da binding sifatida yaratiladi. Body'dagi `var x` mavjud binding'ni topadi va o'sha qiymatni saqlaydi (assignment yo'q bo'lsa).
2. "Rest parameter TDZ'ga ega?" — Ha, `(a, ...rest)` da `rest` `a` ga reference qila olmaydi (oldingi parameter'lardan keyin keladi).

<details>
<summary><strong>Deep Dive</strong></summary>

Spec `10.2.11 FunctionDeclarationInstantiation`: agar funksiya non-simple parameter'larga ega (default/rest/destructuring), engine alohida **ParameterEnvironment** yaratadi. Har parameter `IteratorBindingInitialization` orqali chapdan o'ngga initialize qilinadi: `CreateMutableBinding` + agar argument berilgan bo'lsa `InitializeBinding`, aks holda default expression evaluate qilinadi va `InitializeBinding`. Keyingi parameter binding'lari hali `uninitialized` — TDZ holatida. `ResolveBinding` `uninitialized` binding'ga uchrasa `ReferenceError` throw qiladi.

</details>

</details>

### 9. Block ichida function declaration — strict vs non-strict farqi [Senior]

```javascript
"use strict";

if (true) {
  function helper() { return "strict"; }
  console.log(helper());
}
console.log(typeof helper);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob
Strict mode'da: `"strict"` keyin `"undefined"`. Non-strict mode'da: `"strict"` keyin `"function"` (Annex B web compatibility tufayli).

### To'liq tushuntirish

ES2015 dan boshlab block ichidagi function declaration spec'da aniq belgilangan, lekin xulq-atvor strict mode'ga bog'liq:

- **Strict mode** (yoki ES Module): function declaration faqat **block-scoped** binding yaratadi (`let`-like). Block tashqarisida o'zgaruvchi mavjud emas — `typeof` `"undefined"` qaytaradi.
- **Non-strict mode**: ECMAScript Annex B "Block-Level Function Declarations Web Legacy Compatibility Semantics" qoidalariga ko'ra ikki binding yaratiladi — block-scoped + function-scoped. Spec'gacha bu xulq-atvor engine'larda turlicha implement qilingan edi va Annex B uni keyin standartlashtirgan; web compatibility uchun saqlangan, lekin engine'lararo to'liq bir xil emas (masalan, block hech qachon bajarilmasa, function-scoped binding qiymati `undefined` yoki funksiya bo'lishi engine'ga bog'liq).

### Kod misol

```javascript
// Strict mode (yuqoridagi kod):
"use strict";
if (true) {
  function helper() { return "strict"; }
  console.log(helper()); // "strict" ✅
}
console.log(typeof helper); // "undefined" — block tashqarisida YO'Q

// Non-strict mode (classic script, top-level):
if (true) {
  function helper() { return "non-strict"; }
}
helper(); // "non-strict" ✅ — function-scoped binding mavjud (Annex B)
console.log(typeof helper); // "function"
```

### Edge Cases

- ES Modules **doim strict** — module'da block function declaration faqat block-scoped
- Dynamic code evaluation ichida strict mode'da bir xil qoida
- Block function declaration hoisting: strict'da `let`-like (TDZ), non-strict'da function-scoped `var`-like binding `undefined` bilan boshlanadi, block bajarilganda funksiya qiymati assign bo'ladi

### Follow-up savollar

1. "Nima uchun block function declaration ishlatish noaniq?" — Strict/non-strict farqi va engine'lararo legacy farqlar. Aniq behavior uchun `const fn = function() {}` yoki arrow function tavsiya etiladi.
2. "Loop ichida function declaration?" — Bir xil qoidaga amal qiladi: strict'da block-scoped, non-strict'da Annex B.

<details>
<summary><strong>Deep Dive</strong></summary>

ECMAScript Annex B "Additional ECMAScript Features for Web Browsers" — bu standart qism web compatibility uchun saqlangan legacy behavior'larni belgilaydi. "Block-Level Function Declarations Web Legacy Compatibility Semantics" non-strict mode'da block function declaration uchun ikkita binding yaratishni belgilaydi: block scope'da `let`-like binding (`InstantiateFunctionObject` natijasi) va function scope'da `var`-like binding. Block ichidagi har bir assignment function-scoped binding'ni ham yangilaydi. Strict mode'da Annex B qoidalari **qo'llanilmaydi** — faqat block-scoped binding qoladi.

</details>

</details>

### 10. Parameter va var nomi to'qnashganda nima sodir bo'ladi? [Senior]

```javascript
function showValue(price) {
  var price;
  console.log(price);

  var price = 999;
  console.log(price);
}

showValue(100);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob
Output: `100` keyin `999`. Parameter va `var` bir xil scope'ga kiradi; assignment'siz `var price` mavjud binding'ni saqlaydi, lekin `var price = 999` assignment'i Execution Phase'da uni override qiladi.

### To'liq tushuntirish

Function parameter'lar va body'dagi `var` declaration'lar bir xil function-level scope'ga (VariableEnvironment) kiradi. Creation Phase'da:

1. Parameter `price` binding sifatida yaratiladi va argument qiymati (`100`) bilan initialize qilinadi
2. `var price` declaration qayta ishlanadi — binding allaqachon mavjud → mavjud qiymat saqlanadi (spec `VarDeclaredNames` qoidasi)

Execution Phase'da:

1. `console.log(price)` → `100` (parameter qiymati)
2. `var price = 999` — declaration qismi no-op, lekin `= 999` assignment bajariladi → `price = 999`
3. `console.log(price)` → `999`

### Kod misol

```javascript
function example(userId) {
  var userId; // ✅ assignment yo'q — parameter qiymati saqlanadi
  console.log(userId); // → argument qiymati

  var userId = "new"; // ❌ assignment override qiladi
  console.log(userId); // → "new"
}
example(42);
// Output:
// 42
// "new"
```

### Edge Cases

- `let`/`const` parameter bilan kolleziya — SyntaxError: `function fn(x) { let x; }` parse-time xato
- Strict mode'da bir nomli parameter'lar (`function fn(x, x)`) — SyntaxError
- Default parameter va body var: `function fn(x = 10) { var x; }` — `x` parameter qiymatini saqlaydi (10)
- Arrow function parameter'lari — bir xil qoidalarga bo'ysunadi

### Follow-up savollar

1. "Function parameter va Function declaration body'da bir xil nomda bo'lsa?" — Function declaration parameter binding'ni override qiladi (function declaration var'dan keyin Creation Phase'da qayta ishlanadi).
2. "Bu pattern qachon foydali?" — Hech qachon. Modern kodda `let`/`const` ishlatib parameter va body nomlarini ajratish — clarity beradi va SyntaxError bilan kolleziyalarning oldini oladi.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec `FunctionDeclarationInstantiation` (10.2.11) qadamlari:

1. `parameterNames` ro'yxati olinadi
2. `varNames` (function body'dagi) ro'yxati olinadi
3. Har parameter uchun `CreateMutableBinding` + `InitializeBinding(argument)` (yoki default)
4. `var` declaration'lar: agar nom `parameterNames` da bo'lsa va binding allaqachon initialize qilingan — `CreateMutableBinding` skip qilinadi (`HasBinding` returns true)
5. Function declaration'lar — `SetMutableBinding` orqali mavjud binding'ni override qiladi (parameter ham var ham)

Shuning uchun: parameter + assignment'siz var = parameter qiymati saqlanadi. Parameter + function declaration = function override.

</details>

</details>
