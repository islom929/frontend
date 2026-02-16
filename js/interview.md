# JavaScript Interview Savollari

> Senior-level javoblar bilan — nafaqat "nima" balki "nima uchun" va "qanday"

---

## Mundarija

- [JS Engine va Asoslar](#js-engine-va-asoslar)
- [Execution Context](#execution-context)
- [Hoisting](#hoisting)
- [Scope](#scope)
- [Closures](#closures)
- [Objects](#objects)
- [Prototypes](#prototypes)
- [Classes](#classes)
- [Functions](#functions)
- [`this` Keyword](#this-keyword)
- [Event Loop](#event-loop)
- [Promises](#promises)
- [Async/Await](#asyncawait)
- [Iterators va Generators](#iterators-va-generators)
- [Modules](#modules)
- [Memory Management](#memory-management)
- [Type Coercion va Equality](#type-coercion-va-equality)
- [DOM Manipulation](#dom-manipulation)
- [Events](#events)
- [Browser APIs](#browser-apis)
- [Error Handling](#error-handling)
- [Modern JavaScript](#modern-javascript)
- [Array Methods](#array-methods)
- [Proxy va Reflect](#proxy-va-reflect)
- [Design Patterns](#design-patterns)

---

## JS Engine va Asoslar

### Savol 1: JavaScript Engine nima va u qanday ishlaydi?

**Javob:**

JavaScript Engine — bu JS kodini olib, mashina tushunadigan instruksiyalarga aylantirib bajaradigan dastur. Masalan, V8 (Chrome, Node.js), SpiderMonkey (Firefox), JavaScriptCore (Safari).

Zamonaviy enginelar **JIT (Just-In-Time) Compilation** ishlatadi:

1. **Parser** source code ni token'larga ajratadi va **AST** (Abstract Syntax Tree) quradi
2. **Interpreter (Ignition)** AST ni **bytecode** ga aylantiradi va darhol bajaradi
3. **Profiler** qaysi kod "hot" (ko'p ishlatiladigan) ekanini kuzatadi
4. **Optimizing Compiler (TurboFan)** hot code ni optimized **machine code** ga compile qiladi
5. Agar type taxminlari noto'g'ri bo'lsa — **deoptimization** — bytecode ga qaytadi

```javascript
function add(a, b) { return a + b; }

// Dastlab Ignition bytecode orqali bajaradi
// Ko'p chaqirilsa → TurboFan machine code ga optimize qiladi
for (let i = 0; i < 100000; i++) {
  add(i, i + 1); // hot code → optimized
}
```

**Deep Dive:**

V8 da bu **lazy compilation** deyiladi — hamma narsa birdaniga compile bo'lmaydi. Faqat zarur bo'lganda va faqat hot code optimize qilinadi. Bu startup vaqtini qisqartiradi — odatiy web sahifadagi kodning ~30-50% hech qachon bajarilmaydi.

---

### Savol 2: Call Stack nima? Stack Overflow qanday sodir bo'ladi?

**Javob:**

Call Stack — funksiya chaqiruvlarini kuzatuvchi **LIFO** (Last In, First Out) ma'lumot tuzilmasi. Har bir funksiya chaqirilganda yangi **Execution Context** stack tepasiga qo'yiladi, funksiya tugaganda olib tashlanadi.

```javascript
function first() {
  second();
}
function second() {
  third();
}
function third() {
  console.log("hello");
}
first();

// Call Stack:
// 1. [global]
// 2. [global, first]
// 3. [global, first, second]
// 4. [global, first, second, third]  ← tepada
// 5. [global, first, second]         ← third tugadi
// 6. [global, first]                 ← second tugadi
// 7. [global]                        ← first tugadi
```

**Stack Overflow** — recursive funksiya base case siz chaqirilganda stack limiti (odatda ~10,000-25,000 frame) oshib ketadi:

```javascript
function infinite() { infinite(); }
infinite(); // RangeError: Maximum call stack size exceeded
```

**Deep Dive:**

Call Stack = Execution Context Stack. JavaScript **single-threaded** — bitta call stack bor. Shuning uchun bitta og'ir funksiya butun UI ni bloklashi mumkin. Bu muammoni Event Loop ([11-event-loop.md](11-event-loop.md)) hal qiladi.

---

### Savol 3: Quyidagi kodning output ini ayting va nima uchun shunday ekanini tushuntiring:

```javascript
function foo() {
  console.log("foo");
  bar();
  console.log("foo end");
}

function bar() {
  console.log("bar");
  throw new Error("xato!");
}

try {
  foo();
} catch(e) {
  console.log("caught:", e.message);
}
console.log("davom");
```

**Javob:**

```
foo
bar
caught: xato!
davom
```

**Tushuntirish:**

1. `foo()` chaqirildi → "foo" chop etildi
2. `bar()` chaqirildi → "bar" chop etildi
3. `throw new Error("xato!")` — xato tashlanadi
4. Error call stack bo'ylab **yuqoriga** (unwind) ko'tariladi: `bar` → `foo` → `try/catch`
5. `catch` blok xatoni ushladi → "caught: xato!" chop etildi
6. Dastur davom etadi → "davom" chop etildi

"foo end" **chop etilmaydi** — chunki `bar()` ichida throw bo'lganda, `foo` ning qolgan qismi bajarilmaydi, stack darhol unwind bo'ladi.

**Deep Dive:**

Error propagation Call Stack bilan to'g'ridan-to'g'ri bog'liq. Error tashlanganida, engine stack'dagi har bir frame'ni tekshiradi — `try/catch` bor frame'ni topguncha. Topmasa — "Uncaught Error" va dastur to'xtaydi.

---

### Savol 4: V8 da Hidden Classes nima? Qanday qilib kodni V8 uchun optimize qilish mumkin?

**Javob:**

V8 har bir object uchun ichki **Hidden Class** (rasmiy nomi **Map**) yaratadi. Bu object ning "shape" ini belgilaydi — qaysi property'lar bor va qaysi tartibda.

```javascript
// Bir xil tartib = bir xil hidden class = TEZKOR
let a = { x: 1, y: 2 };
let b = { x: 3, y: 4 }; // a va b BIR hidden class

// Farqli tartib = farqli hidden class = SEKIN
let c = { y: 1, x: 2 }; // c BOSHQA hidden class (y avval)
```

**Optimization qoidalari:**

```javascript
// 1. Doim bir xil tartibda property qo'shing
// ❌
let obj = {};
if (cond) obj.a = 1;
obj.b = 2;

// ✅
let obj = { a: cond ? 1 : null, b: 2 };

// 2. delete ishlatmang
// ❌
delete obj.a;
// ✅
obj.a = undefined;

// 3. Funksiyalarga doim bir xil type bering
// ❌
calc(5); calc("5"); calc(true);
// ✅
calc(5); calc(10); calc(42);
```

**Deep Dive:**

Hidden class'lar **inline caching** imkonini beradi. V8 property access ni birinchi marta bajarganida, hidden class va offset ni cache'laydi. Keyingi access'larda agar hidden class bir xil bo'lsa — to'g'ridan-to'g'ri memory offset bilan oladi (hash table lookup emas). Bu katta performance farq.

Monomorphic (1 type) > Polymorphic (2-4 type) > Megamorphic (5+ type) — V8 optimizatsiya darajasi.

---

### Savol 5: JIT Compilation nima? Oddiy Interpreter va Compiler dan qanday farqi bor?

**Javob:**

| Yondashuv | Ishlash | Afzallik | Kamchilik |
|-----------|---------|----------|-----------|
| **Interpreter** | Satr-bosatr o'qib bajaradi | Tez boshlaydi | Bajarish sekin |
| **Compiler** | Butun kodni oldin compile qiladi | Bajarish tez | Boshlash sekin |
| **JIT** | Avval interpret, keyin hot code compile | Ikkalasining yaxshisi | Murakkab implementation |

```javascript
// JIT amalda:
function calculate(n) {
  return n * 2 + 1;
}

// 1-chi chaqiruv: Ignition (interpreter) bajiradi — tez boshlash
calculate(5);

// 100-chi chaqiruv: hali Ignition, lekin profiling yig'moqda
// "n doim number ekan..."

// ~2000-chi chaqiruv: TurboFan optimize qildi!
// Machine code: faqat integer multiply + add
// Hech qanday type check yo'q — juda tez

// 50,000-chi chaqiruv: optimized machine code — C++ tezligiga yaqin
```

**Deep Dive:**

V8 ning JIT pipeline bir necha **tier** dan iborat. Ignition (Tier 0, bytecode), Sparkplug (Tier 1, fast compile non-optimized), Maglev (Tier 2, mid-tier optimized), TurboFan (Tier 3, full optimization). Har bir tier sekinroq compile qiladi, lekin tezroq bajaradi. Kod qanchalik "hot" bo'lsa, shuncha yuqori tier'ga ko'tariladi.

---

### Savol 6: Stack va Heap xotira farqi nima? Qaysi ma'lumotlar qayerda saqlanadi?

**Javob:**

```javascript
// STACK — primitives (copy by value)
let a = 42;        // stack da
let b = a;         // b ga 42 ning NUSXASI
b = 100;
console.log(a);    // 42 — a o'zgarmadi

// HEAP — reference types (copy by reference)
let arr1 = [1, 2, 3];  // heap da, arr1 da reference
let arr2 = arr1;        // arr2 da AYNI reference
arr2.push(4);
console.log(arr1);      // [1, 2, 3, 4] — arr1 HAM o'zgardi!
```

| | Stack | Heap |
|-|-------|------|
| **Nima** | Primitives, references, execution context | Objects, arrays, functions |
| **Tezlik** | Juda tez (CPU cache friendly) | Sekinroq (random access) |
| **Hajm** | Kichik (~1-8MB) | Katta (~1.5GB) |
| **Tozalash** | Avtomatik (scope tugasa) | Garbage Collector |

**Deep Dive:**

Stack tez ishlashiga sabab — u **contiguous memory** (yonma-yon xotira) va CPU cache ga yaxshi tushadi. Heap esa **fragmented** — object'lar tarqoq joylashadi. V8 heap ni **New Space** (yangi, kichik, tez GC) va **Old Space** (eski, katta, sekin GC) ga bo'ladi — bu Generational GC strategiyasi.

---

## Execution Context

### Savol 7: Execution Context nima? Qanday yaratiladi?

**Javob:**

Execution Context — JavaScript engine kodni bajarish uchun yaratadigan muhit. Ichida o'zgaruvchilar, scope chain va `this` qiymati saqlanadi.

Har bir EC **2 phase** dan o'tadi:

1. **Creation Phase** — xotira ajratish:
   - `var` → `undefined` bilan initialize
   - `let`/`const` → TDZ da (uninitialized)
   - `function declaration` → to'liq hoist
   - `this` binding aniqlanadi
   - Scope chain quriladi

2. **Execution Phase** — kod satr-bosatr bajariladi

```javascript
console.log(x); // undefined — var hoist + init
console.log(y); // ReferenceError — let TDZ da
console.log(fn); // function fn(){} — to'liq hoist

var x = 1;
let y = 2;
function fn() { return 3; }
```

3 turi bor: Global EC (1 ta, dastur boshida), Function EC (har chaqiruvda yangi), Eval EC.

**Deep Dive:**

EC ichida ikkita environment bor:
- **VariableEnvironment** — `var` uchun (function scope)
- **LexicalEnvironment** — `let`/`const`/`function` uchun (block scope)

Global EC da VariableEnvironment **Object Environment Record** ishlatadi (shuning uchun `var` → `window.property`), LexicalEnvironment esa **Declarative Environment Record** (shuning uchun `let`/`const` window da yo'q).

---

### Savol 8: Quyidagi kodning output ini ayting:

```javascript
var a = 1;

function outer() {
  console.log(a);
  var a = 2;

  function inner() {
    console.log(a);
    var a = 3;
  }

  inner();
  console.log(a);
}

outer();
console.log(a);
```

**Javob:**

```
undefined
undefined
2
1
```

**Tushuntirish:**

1. `outer()` chaqirildi. Uning Creation Phase da **local** `var a = undefined` hoist bo'ladi. Shuning uchun birinchi `console.log(a)` = `undefined` (global `a=1` ni emas, o'z scope'idagi hoist qilingan `a` ni ko'radi)
2. `a = 2` — outer ning `a` endi 2
3. `inner()` chaqirildi. Uning ham **local** `var a = undefined`. Shuning uchun `console.log(a)` = `undefined` (outer ning `a=2` ni emas)
4. `inner()` tugadi, outer davom: `console.log(a)` = `2` (outer ning `a`)
5. `outer()` tugadi, global davom: `console.log(a)` = `1` (global `a`, hech kim o'zgartirmagan)

Har bir `var a` o'z scope da **yangi `a`** yaratadi — tashqi `a` ni **shadow** qiladi.

**Deep Dive:**

Bu **variable shadowing** — ichki scope'dagi o'zgaruvchi tashqi scope'dagini "yashiradi". Engine avval o'z scope'ida qidiradi, topsa — tashqariga chiqmaydi. Scope chain faqat o'z scope'ida **topilmasa** tashqariga yuradi.

---

### Savol 9: VariableEnvironment va LexicalEnvironment farqi nima?

**Javob:**

| | VariableEnvironment | LexicalEnvironment |
|-|--------------------|--------------------|
| **Nima** | `var` declarations | `let`, `const`, `function` |
| **Scope** | Function scope | Block scope |
| **Hoisting** | `undefined` bilan | TDZ (uninitialized) |
| **Global** | `window` ga qo'shadi | `window` ga qo'shMAYDI |

```javascript
function example() {
  if (true) {
    var x = 1;   // VariableEnvironment → function scope → if dan chiqadi
    let y = 2;   // LexicalEnvironment → block scope → if ichida qoladi
  }
  console.log(x); // 1 ✅
  console.log(y); // ReferenceError ❌
}
```

**Deep Dive:**

Har bir block (`if`, `for`, `while`, `{}`) yangi **LexicalEnvironment** yaratadi, lekin **VariableEnvironment** o'zgarmaydi. Shuning uchun `var` block'dan chiqadi (function scope ga tegishli), `let`/`const` esa chiqmaydi (block'ning LexicalEnvironment da qoladi).

---

### Savol 10: Nima uchun global `var` `window` da ko'rinadi, lekin `let` ko'rinmaydi?

**Javob:**

```javascript
var a = 1;
let b = 2;

console.log(window.a); // 1
console.log(window.b); // undefined
```

Sababi: Global EC da ikki xil **Environment Record** bor:

- `var` → **Object Environment Record** — bu global object (window) ning o'zi. `var a = 1` deganda `window.a = 1` bo'ladi.
- `let`/`const` → **Declarative Environment Record** — bu alohida ichki ma'lumotlar bazasi, window bilan aloqasi yo'q.

**Deep Dive:**

Bu ECMAScript spec'dagi dizayn qaror. ES6 da `let`/`const` qo'shilganda, global object'ni ifloslashtirishni to'xtatish maqsad edi. Eski `var` behavior backward compatibility uchun qoldi. Shuning uchun zamonaviy kodda global scope'da `let`/`const` ishlatish tavsiya qilinadi — window property'lariga tasodifiy yozishning oldi olinadi.

---

## Hoisting

### Savol 11: Hoisting nima? "let hoist bo'lmaydi" degan gap to'g'rimi?

**Javob:**

Hoisting — bu Creation Phase da engine barcha declaration'larni topib, oldindan xotiraga yozishi. Kod **ko'tarilmaydi**, engine **oldindan tayyorlaydi**.

`let` va `const` **HOIST BO'LADI**. Lekin `undefined` bilan initialize qilinmaydi — TDZ (Temporal Dead Zone) da turadi.

```javascript
// ISBOT: let hoist bo'lishini ko'rsatish
let x = "global";

function test() {
  console.log(x); // ReferenceError — "global" emas!
  let x = "local";
}
test();
// Agar let hoist bo'lmaganida → global x = "global" chiqishi kerak edi
// Lekin ReferenceError berdi → demak engine LOCAL x borligini BILADI → hoist bo'lgan!
```

| Keyword | Hoist | Initialize | Kirish mumkinmi? |
|---------|-------|-----------|-----------------|
| `var` | ✅ | `undefined` | ✅ (undefined qaytaradi) |
| `let` | ✅ | ❌ (TDZ) | ❌ (ReferenceError) |
| `const` | ✅ | ❌ (TDZ) | ❌ (ReferenceError) |
| `function` | ✅ | To'liq body | ✅ (chaqirish mumkin) |

**Deep Dive:**

Texnik jihatdan `var` va `let`/`const` farqi — `var` Creation Phase da **CreateBinding + InitializeBinding** ikkalasi ham bo'ladi (undefined bilan). `let`/`const` esa faqat **CreateBinding** — InitializeBinding Execution Phase da, e'lon satriga yetganda bo'ladi. Oradagi gap = TDZ.

---

### Savol 12: Quyidagi kodning output ini ayting:

```javascript
var a = 1;
function b() {
  a = 10;
  return;
  function a() {}
}
b();
console.log(a);
```

**Javob:**

```
1
```

**Tushuntirish:**

Bu klassik interview trick:

1. `b()` ning Creation Phase da `function a(){}` **hoist** bo'ladi → **local** `a` yaratiladi
2. Execution Phase da `a = 10` — bu **local** `a` ni o'zgartiradi (global emas!)
3. `return` — funksiya tugadi
4. Global `a` hali `1` — o'zgarmagan

Kalit: `function a(){}` `b()` ichida **local scope** da `a` yaratdi. `a = 10` shu local `a` ga yozildi.

**Deep Dive:**

Bu xuddi quyidagiga teng:
```javascript
function b() {
  var a = function() {}; // hoist natijasi
  a = 10;               // local a ni o'zgartirdi
  return;
}
```

---

### Savol 13: TDZ (Temporal Dead Zone) nima? Qachon sodir bo'ladi?

**Javob:**

TDZ — `let`/`const` o'zgaruvchining hoist bo'lgan nuqtasidan to e'lon qilingan satrgacha bo'lgan zona. Bu zona ichida o'zgaruvchiga kirish **taqiqlangan**.

```javascript
{
  // ─── TDZ boshlanadi ───
  console.log(x); // ❌ ReferenceError
  // ─── hali TDZ ───
  let x = 10;     // ← TDZ TUGADI
  console.log(x); // ✅ 10
}
```

TDZ **temporal** (vaqtga asoslangan):

```javascript
// Ishlaydi — chaqirilgan vaqtda x tayyor
const fn = () => console.log(x);
let x = 10;
fn(); // 10 ✅

// Ishlamaydi — chaqirilgan vaqtda x TDZ da
const fn2 = () => console.log(y);
fn2(); // ❌ ReferenceError
let y = 20;
```

**Deep Dive:**

TDZ `typeof` ni ham buzadi — bu oddiy "e'lon qilinmagan" o'zgaruvchidan farqi:

```javascript
typeof undeclared;  // "undefined" — xavfsiz
typeof tdzVariable; // ❌ ReferenceError — TDZ!
let tdzVariable;
```

---

### Savol 14: Quyidagi kod nima beradi va nima uchun?

```javascript
console.log(typeof foo);
var foo = "hello";
function foo() { return "world"; }
console.log(typeof foo);
```

**Javob:**

```
"function"
"string"
```

**Tushuntirish:**

Creation Phase priority:
1. `var foo` → `undefined`
2. `function foo(){}` → function (var ni **overwrite** qildi)

Shuning uchun birinchi `typeof foo` = `"function"`.

Execution Phase:
1. `foo = "hello"` — string bilan overwrite

Shuning uchun ikkinchi `typeof foo` = `"string"`.

**Deep Dive:**

Function declaration var declaration dan **priority** oladi. Lekin Execution Phase dagi **assignment** hammasini overwrite qila oladi. Shuning uchun `var foo = "hello"` ning assignment qismi (`= "hello"`) function ni yengdi.

---

## Scope

### Savol 15: Scope nima? JavaScript da nechta scope turi bor?

**Javob:**

Scope — o'zgaruvchining ko'rinish sohasi. JS da **3 turi** bor:

1. **Global Scope** — hamma joydan ko'rinadi
2. **Function Scope** — funksiya ichida (`var`, `let`, `const`)
3. **Block Scope** — `{}` ichida (faqat `let`/`const`, `var` tanimaydi)

```javascript
let global = "G";          // global scope

function outer() {
  var funcScope = "F";     // function scope
  if (true) {
    let blockScope = "B";  // block scope
    var notBlock = "NB";   // var — function scope!
  }
  console.log(notBlock);   // "NB" ✅ — var block dan chiqdi
  console.log(blockScope); // ❌ ReferenceError
}
```

**Deep Dive:**

`var` faqat function scope taniydi, `let`/`const` ikkalasini ham (function + block). Shuning uchun `var` `if`, `for`, `while` block'laridan chiqadi — bu ko'p buglarning sababi. Zamonaviy kodda `let`/`const` ishlatish standart.

---

### Savol 16: Lexical scope nima? Dynamic scope dan farqi?

**Javob:**

**Lexical (Static) Scope** — o'zgaruvchining scope'i kodda **yozilgan joyiga** qarab aniqlanadi. JavaScript shu usulni ishlatadi.

```javascript
let x = "global";

function foo() { console.log(x); }

function bar() {
  let x = "local";
  foo(); // "global" — foo YOZILGAN joyda x = "global"
}

bar();
```

**Dynamic scope** bo'lganida `foo()` ni `bar()` ichida **chaqirilgani** uchun `x = "local"` ko'rardi. Lekin JS lexical — shuning uchun `"global"`.

**Deep Dive:**

Lexical scope compile-time (parse-time) da aniqlanadi. Engine kodni parse qilganda har bir funksiyaning **`[[Environment]]`** internal slot'iga o'sha paytdagi scope reference'ini saqlaydi. Bu closure'larning asosi — [05-closures.md](05-closures.md).

---

### Savol 17: Scope Chain qanday ishlaydi? Quyidagi kodning output ini ayting:

```javascript
var x = 10;

function a() {
  console.log(x); // ?
  var x = 20;

  function b() {
    console.log(x); // ?
  }
  b();
}

a();
console.log(x); // ?
```

**Javob:**

```
undefined
20
10
```

**Tushuntirish:**

1. `a()` ichida `var x = 20` bor → **hoist** → local `x = undefined` → global `x = 10` ni shadow qildi → birinchi log = `undefined`
2. `x = 20` assign bo'ldi → `b()` chaqirildi → b() da x yo'q → scope chain: b() → a() → `x = 20` topdi
3. Global'da `x = 10` — a() ning local x global'ni o'zgartirmagan

**Deep Dive:**

Scope chain lookup: engine avval o'z scope'ida qidiradi, topmasa outer'ga, topmasa global'ga. Topilmasa — ReferenceError. Birinchi topgan qiymatni qaytaradi va qidirishni to'xtatadi (shadowing shuning uchun ishlaydi).

---

## Closures

### Savol 18: Closure nima? Misol bilan tushuntiring.

**Javob:**

Closure — funksiya + uning tashqi scope o'zgaruvchilariga reference. Funksiya yaratilgan muhitni "eslab qoladi" — tashqi funksiya tugagandan keyin ham.

```javascript
function outer() {
  let count = 0;
  return function() {
    return ++count;
  };
}

const counter = outer(); // outer() tugadi
counter(); // 1 — count hali tirik!
counter(); // 2
counter(); // 3
```

`outer()` tugadi, EC stack'dan chiqdi. Lekin `count` yo'qolmadi — qaytarilgan funksiya unga **closure** hosil qildi.

**Deep Dive:**

Har bir funksiya yaratilganda, engine `[[Environment]]` internal slot ga joriy LexicalEnvironment reference'ini saqlaydi. Funksiya chaqirilganda, yangi EC ning `outer` reference'i shu `[[Environment]]` ga bog'lanadi. Shuning uchun tashqi o'zgaruvchilarga kirish mumkin — ular heap'da tirik, GC tozalamagan (chunki reference bor).

---

### Savol 19: Quyidagi kodning output ini ayting va tushuntiring:

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
```

**Javob:**

```
3
3
3
```

`var i` = bitta o'zgaruvchi (function/global scope). Loop tugaganda `i = 3`. Barcha setTimeout callback'lari **bitta** `i` ga reference qiladi → hammasi `3`.

**Yechim:**

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// 0, 1, 2 — let har iteratsiyada yangi block scope yaratadi
```

**Deep Dive:**

`let` bilan for loop'da engine har iteratsiyada **yangi LexicalEnvironment** yaratadi va `i` ning joriy qiymatini shu environment ga ko'chiradi. Har bir callback o'z environment'iga closure qiladi → o'z `i` nusxasiga ega.

---

### Savol 20: Closure ishlatib private variable qanday yaratiladi?

**Javob:**

```javascript
function createUser(name) {
  let _loginCount = 0; // private — tashqaridan kirish yo'q

  return {
    getName()      { return name; },
    getLoginCount() { return _loginCount; },
    login() {
      _loginCount++;
      return `${name} logged in (${_loginCount})`;
    }
  };
}

const user = createUser("Ali");
user.login();           // "Ali logged in (1)"
user.login();           // "Ali logged in (2)"
user.getLoginCount();   // 2
user._loginCount;       // undefined — private!
user.name;              // undefined — private!
```

**Deep Dive:**

Bu **Module Pattern** ning asosi. ES2022 da `class` ning `#private` field'lari qo'shildi, lekin closure based privacy hali ham ko'p ishlatiladi — ayniqsa functional programming va React hooks'da. Farqi: class private = spec-level enforcement, closure private = scope-level encapsulation.

---

### Savol 21: debounce funksiyasini implement qiling.

**Javob:**

```javascript
function debounce(fn, delay) {
  let timerId;

  return function(...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Test:
const log = debounce((msg) => console.log(msg), 300);
log("a"); log("ab"); log("abc");
// 300ms keyin faqat "abc" chop etiladi
```

**Tushuntirish:** `timerId` closure da saqlanadi. Har yangi chaqiruvda oldingi timer bekor qilinadi (`clearTimeout`), yangi timer o'rnatiladi. Faqat oxirgi chaqiruvdan `delay` ms o'tganda `fn` ishlaydi.

**Deep Dive:**

Real production debounce'da qo'shimcha xususiyatlar kerak:
- `immediate` / `leading` option — birinchi chaqiruvda darhol bajarish
- `cancel()` method — timer ni bekor qilish
- `flush()` method — darhol bajarish
- Return value (Promise bilan)

Lodash'ning `_.debounce` shu barcha xususiyatlarga ega.

---

## Objects

### Savol 22: Shallow copy va deep copy farqi nima? Qaysi usulni qachon ishlatish kerak?

**Javob:**

**Shallow copy** — birinchi daraja nusxalanadi, ichki object'lar reference bo'lib qoladi.
**Deep copy** — hamma narsa to'liq nusxalanadi, hech qanday shared reference yo'q.

```javascript
const original = { a: 1, b: { c: 2 } };

// Shallow:
const shallow = { ...original };
shallow.b.c = 99;
console.log(original.b.c); // 99 ❌ — original o'zgardi!

// Deep:
const deep = structuredClone(original);
deep.b.c = 99;
console.log(original.b.c); // 2 ✅ — alohida
```

| Usul | Chuqurlik | Tavsiya |
|------|-----------|---------|
| Spread / Object.assign | Shallow | Flat object uchun |
| structuredClone | Deep | Standart deep copy |
| JSON hack | Deep | Oddiy data (function/Date yo'q) |
| Lodash cloneDeep | Deep | Complex case |

**Deep Dive:**

`structuredClone` **structured clone algorithm** ishlatadi — bu Web Workers'da `postMessage` bilan data yuborishda ham ishlatiladigan algoritm. U circular reference'ni handle qiladi, Date/Map/Set/ArrayBuffer ni to'g'ri nusxalaydi, lekin function va DOM node'larni quvvatlamaydi.

---

### Savol 23: Object.freeze, Object.seal va Object.preventExtensions farqi nima?

**Javob:**

| | preventExtensions | seal | freeze |
|-|---|---|---|
| Yangi property | ❌ | ❌ | ❌ |
| O'chirish | ✅ | ❌ | ❌ |
| O'zgartirish | ✅ | ✅ | ❌ |

```javascript
const obj = { a: 1 };
Object.freeze(obj);
obj.a = 2;    // ❌ o'zgarmaydi
obj.b = 3;    // ❌ qo'shilmaydi
delete obj.a; // ❌ o'chirilmaydi
```

⚠️ Barchasi **shallow** — ichki object'lar hali mutable!

**Deep Dive:**

`seal` = `preventExtensions` + barcha property'lar `configurable: false`. `freeze` = `seal` + barcha property'lar `writable: false`. Bular descriptor flag'larining kombinatsiyasi.

---

### Savol 24: Property descriptor nima? `defineProperty` va oddiy assignment farqi?

**Javob:**

Har bir property 4 ta flag ga ega: `value`, `writable`, `enumerable`, `configurable`.

```javascript
// Oddiy assignment — barchasi true:
obj.x = 1;
// { value: 1, writable: true, enumerable: true, configurable: true }

// defineProperty — barchasi default false:
Object.defineProperty(obj, "y", { value: 2 });
// { value: 2, writable: false, enumerable: false, configurable: false }
```

Real use case: read-only property, hidden property (enumerable: false), tamper-proof property (configurable: false).

**Deep Dive:**

`enumerable: false` property `Object.keys`, `for...in`, `JSON.stringify`, spread operator da **ko'rinmaydi**. Lekin `Object.getOwnPropertyNames` va `Reflect.ownKeys` da ko'rinadi. Bu "private-like" behavior beradi.

---

## Prototypes

### Savol 25: `__proto__` va `prototype` farqi nima?

**Javob:**

| | `__proto__` | `prototype` |
|-|---|---|
| **Kimda** | Har bir object | Faqat function'larda |
| **Nima** | Object'ning prototype'iga accessor | `new` bilan yaratiladigan instance uchun ota |

```javascript
function Dog(name) { this.name = name; }
Dog.prototype.bark = function() { return "Hav!"; };

const rex = new Dog("Rex");
rex.__proto__ === Dog.prototype;           // true
Object.getPrototypeOf(rex) === Dog.prototype; // true (zamonaviy usul)
```

`__proto__` — legacy. `Object.getPrototypeOf()` ishlating.

**Deep Dive:**

`__proto__` aslida `Object.prototype` da joylashgan getter/setter. `Object.create(null)` bilan yaratilgan object'da `__proto__` yo'q (chunki Object.prototype zanjirda yo'q).

---

### Savol 26: `new` keyword ichidan nima sodir bo'ladi?

**Javob:**

4 qadam:

```javascript
function User(name) { this.name = name; }

// new User("Ali") =
// 1. const obj = {};
// 2. obj.__proto__ = User.prototype;
// 3. const result = User.call(obj, "Ali"); → obj.name = "Ali"
// 4. return (typeof result === "object" && result !== null) ? result : obj;
```

Agar constructor **object** qaytarsa — shu object qaytadi. **Primitive** yoki hech narsa qaytarmasa — yangi yaratilgan object qaytadi.

**Deep Dive:**

```javascript
// Object return → this O'RNIGA shu object:
function Trick() {
  this.a = 1;
  return { b: 2 };
}
new Trick(); // { b: 2 } — this yo'qoldi!

// null return → this qaytadi (null = primitive):
function Safe() { this.a = 1; return null; }
new Safe(); // { a: 1 }
```

---

### Savol 27: Prototype chain da property lookup qanday ishlaydi?

**Javob:**

Object'da property topilmasa, engine `[[Prototype]]` chain bo'ylab yuqoriga qidiradi:

```javascript
const a = { x: 1 };
const b = Object.create(a); // b.__proto__ = a
const c = Object.create(b); // c.__proto__ = b

c.x; // c da yo'q → b da yo'q → a da BOR → 1
c.y; // c → b → a → Object.prototype → null → undefined
```

**Write** farq qiladi: `c.x = 10` — bu a.x ni o'zgartirMAYDI, c da yangi property yaratadi (shadowing).

**Deep Dive:**

Shadowing exception: agar prototype'dagi property `writable: false` bo'lsa, child'da shadow yaratish ham **taqiqlanadi** (strict mode da TypeError). Bu kutilmagan behavior.

---

## Classes

### Savol 28: Class "syntactic sugar" deyiladi — aslida ichida nima ishlaydi?

**Javob:**

Class ichida **prototype** ishlaydi:

```javascript
class User {
  constructor(name) { this.name = name; }
  greet() { return this.name; }
  static create(n) { return new User(n); }
}

// Xuddi shu:
// typeof User === "function"
// User.prototype.greet = function(){...}  (non-enumerable!)
// User.create = function(){...}
```

Lekin class qo'shimcha himoyalarga ega: TDZ (hoist bo'lmaydi), `new` siz chaqirib bo'lmaydi (TypeError), avtomatik strict mode, method'lar non-enumerable.

**Deep Dive:**

Class'ning ichki `[[IsClassConstructor]]: true` flag'i bor. Bu tufayli `new` siz chaqirish TypeError beradi. Oddiy constructor function'da bu flag yo'q — `new` siz ham chaqiriladi (va bugga olib keladi).

---

### Savol 29: `extends` va `super` qanday ishlaydi?

**Javob:**

`extends` — prototype chain quradi. `super()` — parent constructor'ni chaqiradi.

```javascript
class Animal {
  constructor(name) { this.name = name; }
  eat() { return `${this.name} yemoqda`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);        // ← MAJBURIY, this dan oldin
    this.breed = breed;
  }
  bark() { return "Hav!"; }
}

// Chain: dog → Dog.prototype → Animal.prototype → Object.prototype
```

`extends` da `this` ni **parent constructor** yaratadi. `super()` chaqirilmaguncha `this` mavjud emas — shuning uchun `super()` birinchi bo'lishi shart.

**Deep Dive:**

`super.method()` — parent prototype'dagi method'ni chaqiradi. Static method'larda ham ishlaydi. Ichida `[[HomeObject]]` internal property ishlatiladi — bu method qaysi object'ga tegishli ekanini saqlaydi.

---

### Savol 30: Private `#` field va closure privacy farqi nima?

**Javob:**

```javascript
// # Private:
class A { #x = 1; getX() { return this.#x; } }

// Closure:
function createA() { let x = 1; return { getX() { return x; } }; }
```

| | `#` Private | Closure |
|-|---|---|
| **Haqiqiy private** | ✅ (spec-level) | ✅ (scope-level) |
| **instanceof** | ✅ | ❌ |
| **Subclass ko'radimi** | ❌ | Scope'ga bog'liq |
| **Performance** | Tezroq | O'rtacha |
| **DevTools** | Ko'rinadi | Ko'rinmaydi |

`#` — class ecosystem uchun yaxshiroq (inheritance, instanceof, typing). Closure — functional pattern uchun.

---

## Functions

### Savol 31: Function Declaration, Function Expression va Arrow Function farqi nima?

**Javob:**

```javascript
// Declaration — hoist bo'ladi, o'zining this/arguments bor
function greet(name) { return `Hello, ${name}`; }

// Expression — hoist bo'lmaydi (faqat variable hoist, TDZ)
const greet = function(name) { return `Hello, ${name}`; };

// Arrow — hoist bo'lmaydi, this/arguments yo'q (lexical)
const greet = (name) => `Hello, ${name}`;
```

| Xususiyat | Declaration | Expression | Arrow |
|-----------|------------|------------|-------|
| Hoisting | ✅ To'liq | ❌ (TDZ) | ❌ (TDZ) |
| `this` | Dynamic | Dynamic | Lexical (tashqi) |
| `arguments` | ✅ | ✅ | ❌ |
| `new` bilan | ✅ | ✅ | ❌ |
| `prototype` | ✅ | ✅ | ❌ |
| Nom (debugging) | ✅ doim | Ixtiyoriy | Ixtiyoriy |

**Deep Dive:**

Arrow function — `[[ThisMode]]: lexical` bo'lgani uchun o'zining `this` binding'i umuman yo'q. Engine tashqi scope'ning `this` ni ishlatadi. Shuning uchun `bind/call/apply` arrow function'da `this` ni o'zgartirmaydi.

---

### Savol 32: First-class function nima degani?

**Javob:**

JavaScript'da funksiya — **birinchi darajali fuqaro (first-class citizen)**. Ya'ni funksiya boshqa qiymatlar kabi:

```javascript
// 1. O'zgaruvchiga assign
const fn = function() { return 42; };

// 2. Argument sifatida berish
[1, 2, 3].map(n => n * 2);

// 3. Funksiyadan qaytarish
function multiplier(factor) {
  return (num) => num * factor;
}
const double = multiplier(2);
double(5); // 10

// 4. Object property sifatida
const obj = { run: () => console.log('running') };

// 5. Array elementida
const pipeline = [trim, lowercase, validate];
```

**Deep Dive:**

Spec'da funksiya — `callable object`. `[[Call]]` internal method'i bor bo'lgan har qanday object chaqirilishi mumkin. Shu internal method mavjudligi uni "first-class" qiladi.

---

### Savol 33: Pure function nima? Nima uchun muhim?

**Javob:**

Pure function — **ikki shartni** bajaruvchi funksiya:
1. Bir xil input → doim bir xil output (deterministic)
2. Side effect yo'q (tashqi state'ni o'zgartirmaydi)

```javascript
// ✅ Pure
function add(a, b) {
  return a + b;
}

// ❌ Impure — tashqi state ga bog'liq
let tax = 0.2;
function calcPrice(price) {
  return price + price * tax; // tax o'zgarsa, natija o'zgaradi
}

// ❌ Impure — side effect (tashqi state'ni o'zgartiradi)
let total = 0;
function addToTotal(amount) {
  total += amount; // mutation
  return total;
}
```

**Nima uchun muhim:** Testlash oson, debug qilish oson, memoize qilsa bo'ladi, parallellashtirsa bo'ladi. React, Redux, functional programming — hammasi pure function'larga asoslangan.

---

### Savol 34: Currying nima? Implement qiling.

**Javob:**

Currying — ko'p argumentli funksiyani **birma-bir argument qabul qiluvchi** funksiyalar ketma-ketligiga aylantirish.

```javascript
// Oddiy
function add(a, b, c) { return a + b + c; }

// Curried
function curriedAdd(a) {
  return function(b) {
    return function(c) {
      return a + b + c;
    };
  };
}
curriedAdd(1)(2)(3); // 6
```

**Universal curry implementation:**

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);    // 6
add(1, 2)(3);    // 6
add(1)(2, 3);    // 6
add(1, 2, 3);    // 6
```

**Deep Dive:**

`fn.length` — funksiya kutayotgan parametrlar soni. Curry har safar argumentlar yetarli ekanini tekshiradi. Yetmasa — yangi funksiya qaytaradi (closure orqali oldingi argumentlar saqlanadi).

---

### Savol 35: Debounce va Throttle farqi nima? Implement qiling.

**Javob:**

| | Debounce | Throttle |
|---|---------|----------|
| **Qachon ishlaydi** | Oxirgi chaqiruvdan N ms keyin | Har N ms da bir marta |
| **Use case** | Search input, resize | Scroll, mousemove |
| **Natija** | Faqat oxirgisi | Tekis taqsimlangan |

```javascript
// Debounce — oxirgi chaqiruvdan keyin kutadi
function debounce(fn, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle — oraliqda faqat bir marta
function throttle(fn, limit) {
  let lastCall = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastCall >= limit) {
      lastCall = now;
      return fn.apply(this, args);
    }
  };
}
```

```
Debounce (300ms):
Chaqiruvlar: ──x──x──x──x──────────|──> fn()
                                    ^ 300ms keyin

Throttle (300ms):
Chaqiruvlar: ──x──x──x──x──x──x──x──x──>
Bajariladi:  ──x────────x────────x───────>
             ^          ^        ^
             Har 300ms da bir
```

---

### Savol 36: Memoization nima? Implement qiling.

**Javob:**

Memoization — oldingi natijalarni **cache'lash** orqali takroriy hisoblashlarni oldini olish.

```javascript
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const factorial = memoize(function f(n) {
  if (n <= 1) return 1;
  return n * f(n - 1);
});

factorial(100); // Hisoblaydi
factorial(100); // Cache'dan — darhol
```

**Deep Dive:**

`JSON.stringify` — oddiy key uchun yetadi, lekin object/circular ref uchun muammo. Production'da `WeakMap` (object key), yoki custom hash function ishlatiladi. React'da `useMemo`, `React.memo` — xuddi shu printsip.

---

### Savol 37: Output nima?

**Javob:**

```javascript
const funcs = [];
for (var i = 0; i < 3; i++) {
  funcs.push(function() { return i; });
}
console.log(funcs[0]()); // ?
console.log(funcs[1]()); // ?
console.log(funcs[2]()); // ?
```

**Javob:** Hammasi `3` qaytaradi.

`var` — function-scoped, bitta `i` bor. Loop tugaganda `i = 3`. Barcha funksiyalar bir xil `i` ga closure qilgan.

**Yechimlar:**
```javascript
// 1. let ishlatish (block scope)
for (let i = 0; i < 3; i++) { ... }

// 2. IIFE bilan
for (var i = 0; i < 3; i++) {
  funcs.push((function(j) { return function() { return j; }; })(i));
}
```

---

## `this` Keyword

### Savol 38: `this` qanday aniqlanadi? Binding qoidalari nima?

**Javob:**

`this` — **runtime'da**, funksiya **qanday chaqirilganiga** qarab aniqlanadi (qayerda e'lon qilinganiga emas).

**4 ta qoida (priority tartibida):**

```javascript
// 1. new binding (ENG YUQORI)
function User(name) { this.name = name; }
const u = new User('Ali'); // this → yangi object

// 2. Explicit binding
function greet() { console.log(this.name); }
greet.call({ name: 'Vali' }); // this → { name: 'Vali' }

// 3. Implicit binding
const obj = {
  name: 'Sami',
  greet() { console.log(this.name); }
};
obj.greet(); // this → obj

// 4. Default binding (ENG PAST)
function show() { console.log(this); }
show(); // this → globalThis (yoki undefined strict mode'da)
```

**Deep Dive:**

Engine har bir funksiya chaqiruvida `[[Call]]` internal method'ni ishlatadi. `thisValue` argument sifatida beriladi. Priority: agar `new` → yangi object; agar `call/apply/bind` → berilgan value; agar `obj.method()` → obj; aks holda → global/undefined.

---

### Savol 39: `call`, `apply`, `bind` farqi nima?

**Javob:**

```javascript
function introduce(greeting, punctuation) {
  console.log(`${greeting}, men ${this.name}${punctuation}`);
}

const user = { name: 'Ali' };

// call — argumentlar alohida
introduce.call(user, 'Salom', '!');

// apply — argumentlar array'da
introduce.apply(user, ['Salom', '!']);

// bind — yangi funksiya qaytaradi, darhol chaqirmaydi
const boundFn = introduce.bind(user, 'Salom');
boundFn('!'); // keyinroq chaqirish
```

| | `call` | `apply` | `bind` |
|---|--------|---------|--------|
| **Bajaradi** | Darhol | Darhol | Yo'q (qaytaradi) |
| **Argumentlar** | Alohida | Array | Alohida (partial) |
| **Qaytaradi** | Natija | Natija | Yangi funksiya |

---

### Savol 40: Output nima?

**Javob:**

```javascript
const obj = {
  name: 'Test',
  getName: () => this.name,
  getNameRegular() { return this.name; }
};

console.log(obj.getName());        // ?
console.log(obj.getNameRegular()); // ?
```

**Javob:**
- `obj.getName()` → `undefined` — arrow function `this`'ni tashqi scope'dan oladi (global/module), obj'dan emas
- `obj.getNameRegular()` → `"Test"` — implicit binding ishlaydi

---

### Savol 41: Output nima?

**Javob:**

```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }
  start() {
    setInterval(function() {
      this.seconds++;
      console.log(this.seconds);
    }, 1000);
  }
}
new Timer().start(); // Nima bo'ladi?
```

**Javob:** `NaN` chiqadi. `setInterval` callback'i — oddiy function. `this` → `globalThis`. `globalThis.seconds` → `undefined`. `undefined + 1` → `NaN`.

**Yechim:**
```javascript
start() {
  // Arrow function — lexical this
  setInterval(() => {
    this.seconds++;
    console.log(this.seconds); // 1, 2, 3...
  }, 1000);
}
```

---

### Savol 42: `this` yo'qotish muammosi va yechimi

**Javob:**

```javascript
class UserService {
  constructor(name) {
    this.name = name;
  }
  greet() {
    console.log(`Salom, ${this.name}`);
  }
}

const service = new UserService('Ali');

// ❌ this yo'qoladi
const fn = service.greet;
fn(); // TypeError: Cannot read properties of undefined

// ❌ Callback sifatida
setTimeout(service.greet, 100); // undefined

// ✅ Yechim 1: bind
setTimeout(service.greet.bind(service), 100);

// ✅ Yechim 2: arrow wrapper
setTimeout(() => service.greet(), 100);

// ✅ Yechim 3: class field arrow (eng yaxshi)
class UserService {
  constructor(name) { this.name = name; }
  greet = () => {
    console.log(`Salom, ${this.name}`);
  };
}
```

**Deep Dive:**

Method'ni reference sifatida olganingizda, `this` binding yo'qoladi — chunki chaqiruv call-site'da `obj.method()` formatida emas. Class arrow field — har bir instance uchun yangi funksiya yaratadi (memory trade-off), lekin `this` doim to'g'ri bo'ladi.

---

## Event Loop

### Savol 43: Event Loop nima? JavaScript qanday qilib single-threaded bo'la turib async ishlaydi?

**Javob:**

JavaScript — **single-threaded**, ya'ni bir vaqtda faqat bitta ish bajaradi. Lekin **Event Loop** mexanizmi tufayli async operatsiyalar "parallel" ko'rinadi.

```
┌───────────────────────────────────────────┐
│              JavaScript Runtime            │
├──────────┬────────────────────────────────┤
│Call Stack │  Web APIs (setTimeout, fetch)  │
│          │  ↓                             │
│          │  Callback/Macrotask Queue      │
│          │  Microtask Queue               │
│          │  ← Event Loop ←               │
└──────────┴────────────────────────────────┘
```

**Algoritm:**
1. Call Stack bo'shmi? → yo'q bo'lsa kutadi
2. Microtask Queue → **barchasini** bajir (Promise.then, queueMicrotask)
3. Macrotask Queue → **bittasini** bajir (setTimeout, setInterval)
4. Render (kerak bo'lsa)
5. 1-ga qayt

**Deep Dive:**

Microtask'lar **doim** macrotask'lardan oldin bajariladi. Shuning uchun `Promise.then()` har doim `setTimeout(fn, 0)` dan oldin ishlaydi. Agar microtask ichida yangi microtask qo'shilsa — u ham shu cycle'da bajariladi (starvation xavfi).

---

### Savol 44: Output tartibini aniqlang

**Javob:**

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

queueMicrotask(() => console.log('4'));

console.log('5');
```

**Javob:** `1`, `5`, `3`, `4`, `2`

**Tushuntirish:**
1. `console.log('1')` — synchronous → darhol
2. `setTimeout` — macrotask queue'ga
3. `Promise.then` — microtask queue'ga
4. `queueMicrotask` — microtask queue'ga
5. `console.log('5')` — synchronous → darhol
6. Call Stack bo'sh → microtask'lar: `3`, `4`
7. Keyin macrotask: `2`

---

### Savol 45: Output tartibini aniqlang (murakkab)

**Javob:**

```javascript
setTimeout(() => console.log('A'), 0);

Promise.resolve()
  .then(() => {
    console.log('B');
    setTimeout(() => console.log('C'), 0);
  })
  .then(() => console.log('D'));

Promise.resolve().then(() => console.log('E'));

console.log('F');
```

**Javob:** `F`, `B`, `E`, `D`, `A`, `C`

**Tushuntirish:**
1. `F` — sync
2. Microtask'lar: birinchi Promise'ning birinchi `.then()` → `B` (ichida `setTimeout('C')` macrotask'ga qo'shiladi), ikkinchi Promise'ning `.then()` → `E`
3. `B` tugagandan keyin ikkinchi `.then()` microtask'ga qo'shiladi → `D`
4. Macrotask'lar tartib bo'yicha: `A` (oldin qo'shilgan), `C` (keyin qo'shilgan)

---

### Savol 46: `setTimeout(fn, 0)` nima uchun 0ms da ishlamaydi?

**Javob:**

`setTimeout(fn, 0)` aslida "0ms dan keyin bajir" degani **emas**. U "macrotask queue'ga qo'y, call stack bo'shaganda va barcha microtask'lar tugaganda bajir" degani.

```javascript
const start = Date.now();
setTimeout(() => {
  console.log(Date.now() - start); // 4-5ms (minimum 4ms browser clamp)
}, 0);
```

Sabablar:
- Browser'da minimum delay **4ms** (nested setTimeout uchun)
- Call Stack bo'shishi kerak
- Microtask'lar birinchi bajariladi
- Render cycle kutishi mumkin

**Deep Dive:** HTML spec bo'yicha, 5+ marta nested `setTimeout` da minimum 4ms clamp qo'llanadi. Node.js da `setTimeout(fn, 0)` → `setTimeout(fn, 1)` ga aylanadi.

---

## Promises

### Savol 47: Promise nima? State'lari qanday?

**Javob:**

Promise — asinxron operatsiya **natijasini ifodalovchi** object. State machine:

```
         resolve(value)
pending ──────────────→ fulfilled
   │                        
   │      reject(reason)    
   └──────────────────→ rejected
```

- **pending** — kutilmoqda (boshlang'ich holat)
- **fulfilled** — muvaffaqiyatli tugadi (value bor)
- **rejected** — xato bilan tugadi (reason bor)

**Muhim:** State faqat **bir marta** o'zgaradi. Fulfilled/rejected ga o'tgandan keyin — qayta o'zgarmaydi (settled).

```javascript
const p = new Promise((resolve, reject) => {
  resolve('birinchi');
  resolve('ikkinchi');  // Ignored — allaqachon settled
  reject('xato');       // Ignored
});
p.then(v => console.log(v)); // 'birinchi'
```

---

### Savol 48: `Promise.all` vs `Promise.allSettled` vs `Promise.race` vs `Promise.any` farqi?

**Javob:**

```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.reject('err');
const p3 = Promise.resolve(3);
```

| Method | Qachon resolve | Qachon reject | Natija |
|--------|---------------|---------------|--------|
| `all` | Hammasi fulfilled | Bitta reject | `[1, 'err' throws, 3]` → reject |
| `allSettled` | Hammasi settled | **Hech qachon reject** | `[{status,value}, {status,reason}, ...]` |
| `race` | Birinchi settled | Birinchi settled (reject) | Birinchi natija |
| `any` | Birinchi fulfilled | Hammasi reject | Birinchi fulfilled yoki AggregateError |

```javascript
// all — bitta xato = hammasi fail
Promise.all([p1, p2, p3]).catch(e => e); // 'err'

// allSettled — hammasi natija beradi
Promise.allSettled([p1, p2, p3]);
// [{status:'fulfilled',value:1}, {status:'rejected',reason:'err'}, {status:'fulfilled',value:3}]

// race — birinchi tugagan (fulfilled yoki rejected)
Promise.race([
  new Promise(r => setTimeout(() => r('sekin'), 100)),
  Promise.reject('tez xato')
]); // reject: 'tez xato'

// any — birinchi muvaffaqiyatli
Promise.any([p2, p1, p3]); // 1
```

---

### Savol 49: Promise.all ni implement qiling

**Javob:**

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;
    const items = Array.from(promises);

    if (items.length === 0) return resolve([]);

    items.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completed++;
          if (completed === items.length) {
            resolve(results);
          }
        })
        .catch(reject); // birinchi reject — hammasi reject
    });
  });
}
```

**Kalit nuqtalar:**
- `Promise.resolve(promise)` — non-promise value'larni ham qo'llab-quvvatlash
- `results[index]` — tartibni saqlash (forEach parallel, lekin index saqlaydi)
- Birinchi `reject` — darhol reject (qolganlarni kutmaydi)

---

### Savol 50: Promise chaining da `.catch()` qayerda qo'yish kerak?

**Javob:**

```javascript
// ❌ Har bir then ga catch — ortiqcha
fetch('/api/users')
  .then(r => r.json()).catch(handleErr)
  .then(data => process(data)).catch(handleErr)
  .then(result => save(result)).catch(handleErr);

// ✅ Oxirida bitta catch — error propagation ishlaydi
fetch('/api/users')
  .then(r => r.json())
  .then(data => process(data))
  .then(result => save(result))
  .catch(handleErr); // qayerda xato bo'lsa ham ushlaydi

// ✅ Recovery kerak bo'lsa — o'rtada catch
fetch('/api/users')
  .then(r => r.json())
  .catch(() => getCachedUsers()) // fallback → davom etadi
  .then(data => render(data));
```

**Deep Dive:** `.catch()` dan keyin chain davom etadi (chunki `.catch()` yangi fulfilled Promise qaytaradi). Shuning uchun error recovery pattern ishlaydi.

---

## Async/Await

### Savol 51: `async/await` nima? Promise bilan farqi?

**Javob:**

`async/await` — Promise ustiga qurilgan **syntactic sugar**. Async kod'ni sync ko'rinishda yozish imkonini beradi.

```javascript
// Promise chain
function fetchUser(id) {
  return fetch(`/api/users/${id}`)
    .then(r => r.json())
    .then(user => fetch(`/api/posts/${user.id}`))
    .then(r => r.json());
}

// async/await — xuddi shu narsa, lekin o'qish oson
async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  const user = await res.json();
  const postsRes = await fetch(`/api/posts/${user.id}`);
  return await postsRes.json();
}
```

**Farqi:** Funksionallik bir xil. `async` doim Promise qaytaradi. `await` — Promise resolve bo'lguncha "kutadi" (aslida microtask queue orqali davom etadi).

**Deep Dive:** Under the hood `async/await` — generator + Promise pattern'ga teng. V8 7.2+ dan beri `await` optimized — qo'shimcha microtick yaratmaydi.

---

### Savol 52: Sequential vs Parallel — qachon qaysi biri?

**Javob:**

```javascript
// ❌ SEQUENTIAL — 3 ta request ketma-ket (jami: 3 * request_time)
async function slow() {
  const users = await fetch('/api/users').then(r => r.json());
  const posts = await fetch('/api/posts').then(r => r.json());
  const comments = await fetch('/api/comments').then(r => r.json());
  return { users, posts, comments };
}

// ✅ PARALLEL — 3 ta request bir vaqtda (jami: max(request_time))
async function fast() {
  const [users, posts, comments] = await Promise.all([
    fetch('/api/users').then(r => r.json()),
    fetch('/api/posts').then(r => r.json()),
    fetch('/api/comments').then(r => r.json()),
  ]);
  return { users, posts, comments };
}
```

**Qoida:** Agar request'lar bir-biriga bog'liq **bo'lmasa** → `Promise.all`. Agar keyingisi oldingisinig natijasiga bog'liq bo'lsa → sequential.

---

### Savol 53: `await` forEach ichida nima uchun ishlamaydi?

**Javob:**

```javascript
// ❌ Ishlamaydi — forEach callback natijasini kutmaydi
async function processItems(items) {
  items.forEach(async (item) => {
    await saveToDb(item); // fire-and-forget!
  });
  console.log('Tayyor'); // Bu OLDIN ishlaydi!
}

// ✅ for...of — ketma-ket kutadi
async function processItems(items) {
  for (const item of items) {
    await saveToDb(item);
  }
  console.log('Tayyor'); // Haqiqatan tayyor
}

// ✅ Parallel kerak bo'lsa
async function processItems(items) {
  await Promise.all(items.map(item => saveToDb(item)));
  console.log('Tayyor');
}
```

**Nima uchun:** `forEach` — callback'ning return qiymatini ignore qiladi. `async` callback Promise qaytaradi, lekin forEach uni `await` qilmaydi. Natijada barcha callback'lar "fire-and-forget" bo'ladi.

---

### Savol 54: Retry with exponential backoff implement qiling

**Javob:**

```javascript
async function retry(fn, maxRetries = 3, baseDelay = 1000) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxRetries) throw err;

      const delay = baseDelay * Math.pow(2, attempt);
      const jitter = delay * Math.random() * 0.1; // 10% jitter
      console.log(`Retry ${attempt + 1}/${maxRetries} in ${delay}ms`);
      await new Promise(r => setTimeout(r, delay + jitter));
    }
  }
}

// Ishlatish:
const data = await retry(
  () => fetch('/api/data').then(r => {
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return r.json();
  }),
  3,    // max 3 retry
  1000  // 1s, 2s, 4s
);
```

**Deep Dive:**

Exponential backoff — har safar kutish vaqti 2x oshadi (1s → 2s → 4s). Jitter qo'shiladi — ko'p client'lar bir vaqtda retry qilmasligi uchun (thundering herd problem).

---

## Iterators va Generators

### Savol 55: Iterable va Iterator farqi nima?

**Javob:**

**Iterable** — `Symbol.iterator` method'i bor object. **Iterator** — `next()` method'i bor, `{ value, done }` qaytaruvchi object.

```javascript
const arr = [1, 2, 3]; // Iterable — Symbol.iterator bor

const iterator = arr[Symbol.iterator](); // Iterator olish
iterator.next(); // { value: 1, done: false }
iterator.next(); // { value: 2, done: false }
iterator.next(); // { value: 3, done: false }
iterator.next(); // { value: undefined, done: true }
```

| | Iterable | Iterator |
|---|---------|----------|
| **Method** | `[Symbol.iterator]()` | `next()` |
| **Qaytaradi** | Iterator | `{ value, done }` |
| **Kim** | Array, String, Map, Set | `[Symbol.iterator]()` natijasi |
| **Ishlatadi** | `for...of`, spread, destructuring | Engine ichida |

**Deep Dive:**

`for...of` — avval `[Symbol.iterator]()` chaqiradi, keyin `done: true` bo'lguncha `next()` ni takrorlaydi. Shuning uchun har qanday object'ni iterable qilsa bo'ladi — faqat `Symbol.iterator` implement qiling.

---

### Savol 56: Generator nima? Oddiy funksiyadan farqi?

**Javob:**

Generator — `function*` bilan yaratilgan maxsus funksiya. U **to'liq bajarmaydi** — `yield` da **to'xtab turadi** va keyingi `next()` chaqiruvida davom etadi.

```javascript
function* counter() {
  console.log('Boshlandi');
  yield 1;
  console.log('Davom');
  yield 2;
  console.log('Tugadi');
  return 3;
}

const gen = counter(); // Hech narsa bajarmaydi!
gen.next(); // 'Boshlandi' → { value: 1, done: false }
gen.next(); // 'Davom'    → { value: 2, done: false }
gen.next(); // 'Tugadi'   → { value: 3, done: true }
```

| | Oddiy funksiya | Generator |
|---|---------------|-----------|
| **Run-to-completion** | ✅ Ha | ❌ Yo'q |
| **Pause/Resume** | ❌ | ✅ `yield` da to'xtaydi |
| **Qaytaradi** | Qiymat | Generator object (iterator) |
| **State** | Saqlamaydi | Ichki state saqlanadi |
| **Lazy** | ❌ | ✅ Kerak bo'lganda hisoblaydi |

**Deep Dive:**

Generator object — ham iterable, ham iterator. Shuning uchun `for...of` bilan ishlaydi. `return` qiymati `for...of` da ko'rinmaydi (faqat `done: true` bo'lganda to'xtaydi).

---

### Savol 57: `next(value)` bilan generator'ga qiymat berish qanday ishlaydi?

**Javob:**

```javascript
function* calculator() {
  const a = yield 'Birinchi son?';
  const b = yield 'Ikkinchi son?';
  return a + b;
}

const calc = calculator();
calc.next();      // { value: 'Birinchi son?', done: false }
calc.next(10);    // a = 10, { value: 'Ikkinchi son?', done: false }
calc.next(20);    // b = 20, { value: 30, done: true }
```

`next(value)` — `yield` expression'ning **qaytarish qiymatini** belgilaydi. Birinchi `next()` dagi argument ignore bo'ladi (chunki hali `yield` ga yetib bormagan).

---

### Savol 58: Infinite sequence generator yozing (Fibonacci)

**Javob:**

```javascript
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Faqat kerak bo'lganini olish — lazy evaluation
const fib = fibonacci();
fib.next().value; // 0
fib.next().value; // 1
fib.next().value; // 1
fib.next().value; // 2
fib.next().value; // 3

// Birinchi 10 ta
function take(gen, n) {
  const result = [];
  for (const val of gen) {
    result.push(val);
    if (result.length >= n) break;
  }
  return result;
}
take(fibonacci(), 10); // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

**Deep Dive:** `while (true)` — xotira muammo emas, chunki generator **lazy** — faqat `next()` chaqirilganda hisoblaydi. Memory O(1).

---

## Modules

### Savol 59: CommonJS va ES Modules farqi nima?

**Javob:**

| Xususiyat | CommonJS | ES Modules |
|-----------|----------|------------|
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | Synchronous (runtime) | Asynchronous (compile-time parse) |
| **Binding** | Value copy | Live binding (reference) |
| **Evaluation** | Dynamic (shart ichida `require` bo'ladi) | Static (top-level, hoisted) |
| **`this`** | `module.exports` | `undefined` |
| **Strict mode** | Yo'q (default) | Doim strict |
| **File ext** | `.js` (default) | `.mjs` yoki `"type": "module"` |
| **Tree shaking** | ❌ Qiyin | ✅ Static analysis tufayli |
| **Browser** | ❌ (bundler kerak) | ✅ `<script type="module">` |

```javascript
// CommonJS — value copy
let count = require('./counter').count; // 0
require('./counter').increment();
console.log(count); // 0 — hali ham eski qiymat!

// ESM — live binding
import { count, increment } from './counter.js';
console.log(count); // 0
increment();
console.log(count); // 1 — yangilangan!
```

**Deep Dive:** ESM — 3 bosqichli: **Parse** (static import/export topiladi) → **Instantiate** (memory allocate, live binding) → **Evaluate** (kod bajariladi). CJS — faqat runtime'da `require()` chaqirilganda fayl bajariladi.

---

### Savol 60: Dynamic import nima? Qachon ishlatiladi?

**Javob:**

```javascript
// Static import — doim load bo'ladi
import { heavyModule } from './heavy.js';

// Dynamic import — kerak bo'lganda load
const button = document.getElementById('report');
button.addEventListener('click', async () => {
  const { generateReport } = await import('./report-generator.js');
  generateReport();
});
```

**Use cases:**
- **Code splitting** — route-based (React.lazy, Vue async components)
- **Conditional loading** — faqat kerak bo'lganda
- **Performance** — initial bundle hajmini kamaytirish

```javascript
// React.lazy
const AdminPanel = React.lazy(() => import('./AdminPanel'));

// Feature flag
if (user.isPremium) {
  const { PremiumFeatures } = await import('./premium.js');
}

// Polyfill
if (!window.IntersectionObserver) {
  await import('intersection-observer');
}
```

**Deep Dive:** `import()` — Promise qaytaradi. Module **bir marta** evaluate bo'ladi (cached). Bundler'lar `import()` ni ko'rganda avtomatik **chunk** qiladi (code splitting).

---

### Savol 61: Tree shaking nima? Nima uchun ESM kerak?

**Javob:**

Tree shaking — **ishlatilmagan** export'larni final bundle'dan **olib tashlash**.

```javascript
// math.js
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export function multiply(a, b) { return a * b; }

// app.js — faqat add ishlatiladi
import { add } from './math.js';
console.log(add(2, 3));
// subtract va multiply bundle'ga TUSHMAYDI
```

**Nima uchun ESM kerak:** ESM import/export — **static** (compile-time ma'lum). Bundler qaysi export ishlatilganini aniq biladi. CJS `require()` — dynamic (runtime), shuning uchun tree shake qilish qiyin.

```javascript
// ❌ CJS — tree shake qilib bo'lmaydi
const utils = require('./utils'); // qaysi property kerakligini bilmaydi

// ✅ ESM — tree shake ishlaydi
import { debounce } from './utils.js'; // aniq nima kerakligi ma'lum
```

**`sideEffects`:** `package.json` da `"sideEffects": false` — bundler'ga "bu modul side effect'siz, xavfsiz tree shake qil" deydi.

---

## Memory Management

### Savol 62: Stack vs Heap farqi nima?

**Javob:**

| | Stack | Heap |
|---|-------|------|
| **Nima saqlanadi** | Primitives, references (pointers) | Objects, arrays, functions |
| **Hajm** | Kichik, fixed (~1MB) | Katta, dynamic |
| **Tezlik** | Juda tez (LIFO) | Sekinroq (lookup kerak) |
| **Boshqaruv** | Avtomatik (scope tugasa tozalanadi) | Garbage Collector |
| **Tartib** | LIFO (Last In, First Out) | Tartibsiz |

```javascript
let name = 'Ali';     // 'Ali' → STACK da
let age = 25;          // 25 → STACK da

let user = {           // pointer → STACK, object → HEAP
  name: 'Ali',
  age: 25
};

let copy = user;       // FAQAT pointer copy bo'ladi (HEAP dagi object bir xil)
copy.name = 'Vali';
console.log(user.name); // 'Vali' — bir xil object!
```

```
STACK                    HEAP
┌──────────┐            ┌──────────────────┐
│ name: 'Ali' │         │                  │
│ age: 25     │         │  { name: 'Ali',  │
│ user: ─────────────→  │    age: 25 }     │
│ copy: ─────────────→  │                  │
└──────────┘            └──────────────────┘
```

---

### Savol 63: Garbage Collection qanday ishlaydi? Mark-and-Sweep nima?

**Javob:**

JavaScript — **avtomatik** memory management. Garbage Collector (GC) ishlatilmayotgan object'larni topib o'chiradi.

**Mark-and-Sweep algoritm:**
1. **Root** lardan boshlash (global object, stack variables, closures)
2. Root'dan reachable bo'lgan barcha object'larni **mark** qilish
3. Mark **bo'lmagan** object'larni **sweep** (o'chirish)

```
Boshlang'ich:                  Mark:                    Sweep:
root → A → B                   root → A✓ → B✓          root → A → B
       ↓                              ↓
       C → D                          C✓ → D✓
                                                        
       E → F (root dan unreachable)   E   F             [E, F o'chirildi]
```

**V8 Generational GC:**
- **Young Generation** (Scavenger) — yangi object'lar, tez-tez tozalanadi (~1-8MB)
- **Old Generation** (Mark-Compact) — uzoq yashaganlar, kamroq tozalanadi
- Yangi object → Young. Agar 2 ta GC cycle'dan omon qolsa → Old ga ko'chadi

---

### Savol 64: Eng ko'p uchraydigan memory leak'lar qaysilar?

**Javob:**

```javascript
// 1. ❌ Accidental global
function process() {
  result = computeHeavy(); // 'var/let/const' yo'q → global!
}

// 2. ❌ Forgotten timer
const id = setInterval(() => {
  const data = getHugeData();
  updateUI(data);
}, 1000);
// clearInterval(id) hech qachon chaqirilmaydi

// 3. ❌ DOM reference
const elements = [];
function addElement() {
  const el = document.createElement('div');
  document.body.appendChild(el);
  elements.push(el); // Array ushlab turadi
}
function removeElement() {
  document.body.removeChild(elements[0]);
  // elements array dan O'CHIRILMADI — detached DOM node
}

// 4. ❌ Closure katta data ni ushlab turish
function createHandler() {
  const hugeData = new Array(1000000).fill('x');
  return function() {
    console.log(hugeData.length); // hugeData GC qila olmaydi
  };
}

// 5. ❌ Event listener remove qilmasa
element.addEventListener('click', handler);
// element o'chirildi, lekin listener hali bor → leak
```

**Yechim:** `clearInterval`, `removeEventListener`, WeakMap/WeakRef, scope'ni tor qiling.

---

### Savol 65: WeakMap nima? Map dan farqi?

**Javob:**

```javascript
// Map — key'ni kuchli ushlab turadi (GC o'chira olmaydi)
const map = new Map();
let obj = { data: 'heavy' };
map.set(obj, 'metadata');
obj = null; // Object hali HAM map ichida — GC tozalamaydi!

// WeakMap — key'ni kuchsiz ushlab turadi (GC o'chira oladi)
const weakMap = new WeakMap();
let obj2 = { data: 'heavy' };
weakMap.set(obj2, 'metadata');
obj2 = null; // GC object'ni va WeakMap entry'ni tozalaydi ✅
```

| | Map | WeakMap |
|---|-----|---------|
| **Key turi** | Har qanday | Faqat object |
| **GC** | Key'ni ushlab turadi | Key GC bo'lishi mumkin |
| **Iterable** | ✅ `for...of` | ❌ |
| **`.size`** | ✅ | ❌ |
| **Use case** | General cache | Private data, DOM metadata |

---

## Type Coercion va Equality

### Savol 66: `==` va `===` farqi nima? Qachon qaysi birini ishlatish kerak?

**Javob:**

`===` — **strict equality** — type conversion yo'q. Type ham, value ham teng bo'lishi shart.
`==` — **loose equality** — avval type coercion, keyin taqqoslash.

```javascript
1 === 1     // true (type: number, value: 1)
1 === '1'   // false (type farq: number vs string)
1 == '1'    // true ('1' → 1 ga convert, keyin 1 == 1)

null === undefined  // false (type farq)
null == undefined   // true (spec da maxsus qoida)
null == 0           // false (null faqat undefined bilan == true)
```

**Qoida:** Doim `===` ishlatilsin. `==` faqat bitta holda foydali: `x == null` (null va undefined ikkalasini tekshiradi).

---

### Savol 67: Output nima?

**Javob:**

```javascript
console.log([] == false);    // ?
console.log([] == ![]);      // ?
console.log('' == false);    // ?
console.log('0' == false);   // ?
console.log(null == 0);      // ?
console.log(NaN == NaN);     // ?
```

**Javoblar:**
- `[] == false` → **true** — `[]` → `""` → `0`, `false` → `0`, `0 == 0`
- `[] == ![]` → **true** — `![]` → `false`, keyin `[] == false` → yuqoridagidek
- `'' == false` → **true** — `""` → `0`, `false` → `0`
- `'0' == false` → **true** — `'0'` → `0`, `false` → `0`
- `null == 0` → **false** — `null` faqat `undefined` bilan loose equal
- `NaN == NaN` → **false** — NaN hech narsaga teng emas, o'ziga ham

---

### Savol 68: `typeof` ning barcha natijalari nima? `typeof null` nima uchun `'object'`?

**Javob:**

```javascript
typeof undefined   // 'undefined'
typeof null        // 'object'  ← BUG!
typeof true        // 'boolean'
typeof 42          // 'number'
typeof 'hello'     // 'string'
typeof Symbol()    // 'symbol'
typeof 42n         // 'bigint'
typeof {}          // 'object'
typeof []          // 'object'  ← array ham object
typeof function(){} // 'function'
```

**`typeof null === 'object'` — tarixiy bug.** JavaScript'ning birinchi implementatsiyasida har bir value **type tag** bilan saqlangan. Object'lar tag `000` edi. `null` — **null pointer** (`0x00`) — tag ham `000`. Shuning uchun `typeof null` → `'object'`. Bu bug 1995-yildan beri bor, tuzatish backward compatibility ni buzadi.

---

### Savol 69: Truthy va Falsy values to'liq ro'yxat

**Javob:**

**Falsy (8 ta — yodlash kerak):**
```javascript
false, 0, -0, 0n, "", null, undefined, NaN
```

**Truthy kutilmagan holatlar:**
```javascript
Boolean([])        // true  ← bo'sh array truthy!
Boolean({})        // true  ← bo'sh object truthy!
Boolean('0')       // true  ← '0' string truthy!
Boolean('false')   // true  ← 'false' string truthy!
Boolean(new Number(0))  // true ← wrapper object truthy!
Boolean(-1)        // true
Boolean(Infinity)  // true
```

**Deep Dive:** `document.all` — yagona **falsy object** (`typeof document.all === 'undefined'`). Bu legacy web compatibility uchun maxsus spec exception.

---

### Savol 70: Map vs Object — qachon qaysi biri?

**Javob:**

| | Object | Map |
|---|--------|-----|
| **Key turi** | String/Symbol | Har qanday (object, function, etc.) |
| **Tartib** | Murakkab qoidalar | Insertion order (doim) |
| **Hajm** | `Object.keys().length` | `.size` (O(1)) |
| **Iteration** | `for...in` + hasOwnProperty | `for...of` natively |
| **Performance** | Kichik data uchun tezroq | Ko'p qo'shish/o'chirish uchun tezroq |
| **Default keys** | Prototype keys bor | Bo'sh |
| **JSON** | ✅ Native | ❌ Manual conversion |

```javascript
// Object — config, DTO, simple key-value
const config = { theme: 'dark', lang: 'uz' };

// Map — frequent add/delete, non-string keys, counting
const clickCount = new Map();
clickCount.set(buttonElement, 0);     // DOM element as key!
clickCount.set(userObject, { clicks: 5 }); // Object as key!
```

**Qoida:** Static config/data → Object. Dynamic key-value, non-string keys, frequent CRUD → Map.

---

## DOM Manipulation

### Savol 56: DOM nima? HTML bilan farqi nima?

**Javob:**

DOM (Document Object Model) — brauzer HTML hujjatni parse qilib, xotirada yaratadigan **daraxt (tree) strukturasi**. HTML — bu matn fayl, DOM — bu tirik (live) JavaScript obyekti.

```javascript
// DOM va HTML farqi — brauzer DOM ni o'zgartiradi
// HTML: <table><tr><td>Test</td></tr></table>
// DOM:  table → tbody → tr → td
// Brauzer avtomatik <tbody> qo'shadi — HTML da yo'q, DOM da bor

console.log(document.querySelector("table").children[0].tagName);
// "TBODY" — HTML da yozmagansiz, lekin DOM da bor!
```

DOM ga kirish:
```javascript
console.log(document.documentElement); // <html>
console.log(document.head);            // <head>
console.log(document.body);            // <body>
```

**Deep Dive:**

Brauzer rendering pipeline: `HTML → Parser → DOM Tree → CSSOM → Render Tree → Layout → Paint → Composite → Screen`. DOM va CSSOM birlashib Render Tree hosil qiladi. JavaScript DOM ni o'zgartirsa — yana Layout/Paint bosqichlari qayta ishlaydi (reflow/repaint).

---

### Savol 57: `textContent`, `innerHTML`, `innerText` farqi nima? Qaysi birini ishlatish kerak?

**Javob:**

```html
<div id="demo">
  <span>Ko'rinadigan</span>
  <span style="display:none">Yashirin</span>
</div>
```

```javascript
const el = document.getElementById("demo");

console.log(el.textContent); // "Ko'rinadigan Yashirin" — barchasi, HTML siz
console.log(el.innerText);   // "Ko'rinadigan" — faqat ko'rinadigan matn
console.log(el.innerHTML);   // "<span>Ko'rinadigan</span>..." — HTML bilan
```

| Xususiyat | HTML parse? | Yashirinlarni | Performance | XSS xavf |
|-----------|-------------|---------------|-------------|-----------|
| `textContent` | ❌ | Ko'rsatadi | ⚡ Tez | ✅ Xavfsiz |
| `innerText` | ❌ | Ko'rsatmaydi | 🔴 Sekin | ✅ Xavfsiz |
| `innerHTML` | ✅ | Ko'rsatadi | 🟡 O'rta | ⚠️ XSS xavfi |

```javascript
// ❌ XSS hujum — hech qachon foydalanuvchi input'ini innerHTML ga qo'ymang!
const userInput = '<img src=x onerror="alert(document.cookie)">';
element.innerHTML = userInput; // 💀 Cookie o'g'irlanishi mumkin!

// ✅ Xavfsiz
element.textContent = userInput; // Oddiy matn sifatida ko'rsatadi
```

**Deep Dive:**

`innerText` sekinligining sababi — u **reflow trigger qiladi**. CSS `display`, `visibility`, `opacity` kabi property'larni hisoblashi kerak — qaysi matn ko'rinadi, qaysi yo'q. `textContent` esa faqat DOM tree'dagi text node'larni yig'adi — CSS hech qanday ahamiyat bermaydi.

---

### Savol 58: Reflow va Repaint nima? Layout Thrashing nima va qanday oldini olish mumkin?

**Javob:**

- **Reflow (Layout)** — element'larning o'lchami/pozitsiyasi qayta hisoblanadi. Qimmat.
- **Repaint** — faqat ko'rinish (rang, shadow) qayta chiziladi. Arzonroq.

```javascript
// ❌ Layout Thrashing — har safar o'qish + yozish = har safar reflow!
const items = document.querySelectorAll(".item");
items.forEach(item => {
  const width = item.offsetWidth;     // O'QISH → reflow!
  item.style.width = width * 2 + "px"; // YOZISH → layout invalid
  // Keyingi offsetWidth → YANA reflow!
});

// ✅ To'g'ri — avval hammasi O'QI, keyin hammasi YOZ
const widths = [...items].map(item => item.offsetWidth); // bitta reflow
items.forEach((item, i) => {
  item.style.width = widths[i] * 2 + "px"; // bitta batch write
});
```

Reflow-free property'lar:
```javascript
// transform va opacity GPU da ishlaydi — Reflow/Repaint yo'q!
element.style.transform = "translateX(100px)"; // Layout ta'sir qilmaydi
element.style.opacity = "0.5";                  // Paint ta'sir qilmaydi
```

**Deep Dive:**

Brauzer style yozishlarni **batch** qiladi va keyingi frame'da bitta reflow qiladi. Lekin `offsetWidth`, `clientHeight`, `getBoundingClientRect()`, `getComputedStyle()` kabi **o'qish** operatsiyalari brauzerga **majburan** layout hisoblashni buyuradi — chunki to'g'ri qiymat qaytarishi kerak. Shuning uchun o'qish va yozishni aralash qilish ko'p reflow'ga olib keladi.

---

### Savol 59: `getElementsByClassName` va `querySelectorAll` farqi nima?

**Javob:**

```javascript
const live = document.getElementsByClassName("item");   // HTMLCollection (LIVE)
const static_ = document.querySelectorAll(".item");     // NodeList (STATIC)

console.log(live.length);    // 3
console.log(static_.length); // 3

// Yangi element qo'shamiz
const div = document.createElement("div");
div.className = "item";
document.body.appendChild(div);

console.log(live.length);    // 4 ← avtomatik yangilandi!
console.log(static_.length); // 3 ← o'zgarmadi (static snapshot)
```

| | `getElementsByClassName` | `querySelectorAll` |
|-|--------------------------|--------------------|
| **Qaytaradi** | HTMLCollection | NodeList |
| **Live?** | ✅ Live | ❌ Static |
| **forEach** | ❌ Yo'q | ✅ Bor |
| **CSS selector** | ❌ Yo'q | ✅ Har qanday |
| **Tezlik** | ⚡ Tezroq | 🟡 Sekinroq |

```javascript
// ⚠️ Live collection bilan loop muammosi
const items = document.getElementsByClassName("item");
// ❌ Loop da o'chirish — elementlar "sakraydi"!
for (let i = 0; i < items.length; i++) {
  items[i].remove(); // items[1] endi items[0] bo'ldi → skip!
}
// Natija: faqat yarmi o'chirildi!

// ✅ querySelectorAll bilan
document.querySelectorAll(".item").forEach(el => el.remove()); // hammasi o'chirildi
```

**Deep Dive:**

`getElementById` eng tez — brauzer ID lar uchun hash map saqlaydi (O(1) lookup). `querySelector` esa CSS selector parser orqali o'tadi va DOM daraxtini traverse qiladi. Lekin zamonaviy brauzerlar optimizatsiya qilgan — farq kichik sahifalarda sezilmaydi.

---

### Savol 60: DocumentFragment nima? Virtual DOM dan farqi nima?

**Javob:**

**DocumentFragment** — DOM ning yengil, virtual container'i. Unga element qo'shish reflow trigger qilmaydi:

```javascript
// ❌ 1000 ta reflow
for (let i = 0; i < 1000; i++) {
  list.appendChild(createElement("li")); // har safar reflow
}

// ✅ 1 ta reflow
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  fragment.appendChild(createElement("li")); // reflow yo'q
}
list.appendChild(fragment); // BITTA reflow
```

**Virtual DOM** — DOM ning **JavaScript object** sifatidagi nusxasi (React, Vue):

| | DocumentFragment | Virtual DOM |
|-|-----------------|-------------|
| **Nima** | Haqiqiy DOM node | JS object |
| **Diffing** | ❌ Yo'q | ✅ Bor |
| **Framework** | Native API | React, Vue |
| **Maqsad** | Batch insert | Minimal DOM update |

```javascript
// Virtual DOM tushunchasi
const vdom = {
  tag: "div",
  props: { className: "app" },
  children: [
    { tag: "h1", props: {}, children: ["Title"] }
  ]
};
// O'zgarish bo'lganda: eski VDOM vs yangi VDOM → diff → faqat farqlarni DOM ga apply
```

**Deep Dive:**

Virtual DOM doim ham tezroq emas. Kichik o'zgarishlar uchun to'g'ridan-to'g'ri DOM manipulation tezroq. Virtual DOM ning kuchi — **developer experience** va **katta, murakkab UI'lar** uchun avtomatik optimizatsiya. Svelte buni compile-time da hal qiladi — runtime Virtual DOM siz.

---

## Events

### Savol 61: Event Bubbling va Capturing nima? Farqi nima?

**Javob:**

DOMda har bir event 3 fazadan o'tadi:
1. **Capturing** — tashqaridan ichga (document → target)
2. **Target** — target elementda
3. **Bubbling** — ichdan tashqariga (target → document)

```javascript
const outer = document.getElementById("outer");
const inner = document.getElementById("inner");

outer.addEventListener("click", () => console.log("outer CAPTURE"), true);
outer.addEventListener("click", () => console.log("outer BUBBLE"));
inner.addEventListener("click", () => console.log("inner CAPTURE"), true);
inner.addEventListener("click", () => console.log("inner BUBBLE"));

// inner bosilsa:
// "outer CAPTURE" → "inner CAPTURE" → "inner BUBBLE" → "outer BUBBLE"
```

**Deep Dive:**

Target element da capturing va bubbling handler'lar **yozilgan tartibda** ishlaydi. Ba'zi eventlar bubble qilmaydi: `focus`/`blur` (o'rniga `focusin`/`focusout`), `mouseenter`/`mouseleave`, `load`/`unload`.

---

### Savol 62: Event Delegation nima? Nima uchun ishlatiladi?

**Javob:**

Event Delegation — har bir elementga alohida handler o'rniga, **ota elementga bitta handler** qo'yib, `event.target` yoki `closest()` bilan bosilgan elementni aniqlash.

```javascript
// ❌ 1000 ta handler — memory isrof
items.forEach(item => item.addEventListener("click", handle));

// ✅ 1 ta handler — delegation
list.addEventListener("click", (e) => {
  const item = e.target.closest(".item");
  if (!item) return;
  console.log("Bosildi:", item.textContent);
});
```

Afzalliklari:
- **Memory tejash** — 1 handler vs N handler
- **Dinamik elementlar** — keyinroq qo'shilgan elementlar ham ishlaydi
- **Sodda cleanup** — bitta handler o'chirish kifoya

**Deep Dive:**

`closest()` nima uchun kerak: `e.target` eng **ichki** bosilgan element. `<li class="item"><span>Text</span></li>` da span bosilsa target = span. `closest(".item")` yuqoriga qarab li ni topadi.

---

### Savol 63: `stopPropagation()` va `preventDefault()` farqi nima?

**Javob:**

| | `stopPropagation()` | `preventDefault()` |
|-|--------------------|-----------------------|
| **Nima** | Event tarqalishini to'xtatadi | Brauzer default xulqini to'xtatadi |
| **Misol** | Parent handler ishlamaydi | Link navigate qilmaydi, form submit bo'lmaydi |
| **Event davom** | Element dagi handler'lar ishlaydi | Event tarqalishi davom etadi |

```javascript
// stopPropagation — parent handler'lar to'xtaydi
inner.addEventListener("click", (e) => {
  e.stopPropagation();
  console.log("inner"); // ishlaydi
});
outer.addEventListener("click", () => {
  console.log("outer"); // ❌ ISHLAMAYDI
});

// preventDefault — brauzer xulqi to'xtaydi
form.addEventListener("submit", (e) => {
  e.preventDefault();  // sahifa reload bo'lmaydi
  // AJAX orqali yuborish
});
```

⚠️ `stopImmediatePropagation()` — shu elementdagi qolgan handler'lar ham to'xtaydi.

**Deep Dive:**

`addEventListener` ichida `return false` **hech nima qilmaydi**. Faqat eski `onclick` property da ishlaydi (preventDefault + stopPropagation). jQuery da ham `return false` ikkisini birga qiladi.

---

### Savol 64: Custom Events qanday yaratiladi? Qachon ishlatiladi?

**Javob:**

```javascript
// Yaratish
const event = new CustomEvent("user-login", {
  detail: { userId: 42, name: "Ali" },
  bubbles: true,
  cancelable: true
});

// Tinglash
document.addEventListener("user-login", (e) => {
  console.log("Login:", e.detail.name); // "Ali"
});

// Dispatch
document.dispatchEvent(event);
```

Ishlatish holatlari:
- Component'lar orasidagi loose kommunikatsiya
- Pub/Sub pattern
- Plugin/extension tizimlar

**Deep Dive:**

`CustomEvent` `Event` dan meros oladi va `detail` property qo'shadi. `bubbles: true` qo'ysak — event delegation ishlaydi. `cancelable: true` qo'ysak — `preventDefault()` bilan bekor qilish mumkin (event handler'da `e.defaultPrevented` tekshirib).

---

### Savol 65: Passive event listener nima? Qachon ishlatiladi?

**Javob:**

`{ passive: true }` brauzerga aytadi: "Bu handler `preventDefault()` chaqirmaydi" — shuning uchun brauzer handler tugashini **kutmaydi**, scroll/touch darhol ishlaydi.

```javascript
// ❌ Sekin scroll — brauzer handler ni kutadi
document.addEventListener("scroll", handleScroll);

// ✅ Tez scroll — brauzer kutmaydi
document.addEventListener("scroll", handleScroll, { passive: true });
// e.preventDefault() chaqirilsa — warning, ishlamaydi
```

| | passive: false | passive: true |
|-|----------------|---------------|
| **Scroll** | Handler kutiladi | Darhol scroll |
| **preventDefault** | ✅ Ishlaydi | ❌ Warning |
| **Performance** | 🔴 Sekin | ⚡ Tez |

**Deep Dive:**

Chrome 51+ da document-level `touchstart` va `touchmove` **default passive: true**. `wheel` event ham Chrome 73+ da default passive. Agar haqiqatan `preventDefault` kerak bo'lsa — `{ passive: false }` aniq yozish kerak, lekin bu scroll performanceni pasaytiradi.

---

## Browser APIs

### Savol 66: Fetch da HTTP xatolarni qanday to'g'ri boshqarish kerak?

**Javob:**

```javascript
// ⚠️ Fetch faqat TARMOQ xatosida reject bo'ladi
// 404, 500 kabi HTTP xatolar REJECT BO'LMAYDI!

// ❌ Noto'g'ri
try {
  const data = await fetch("/api/users").then(r => r.json());
  // 404 ham success — HTML error sahifasini parse qilishga harakat qiladi
} catch (e) { /* faqat tarmoq xatosi */ }

// ✅ To'g'ri
async function apiFetch(url, options) {
  const response = await fetch(url, options);
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  return response.json();
}
```

**Deep Dive:**

Fetch spec: "A fetch() promise only rejects when a network error is encountered." HTTP 4xx/5xx server javob qaytargan — network muvaffaqiyatli. `response.ok` = `status` 200-299 oralig'ida ekanligini tekshiradi.

---

### Savol 67: IntersectionObserver nima? Scroll event dan qanday farqi bor?

**Javob:**

| | `scroll` event | IntersectionObserver |
|-|---------------|---------------------|
| **Trigger** | Har scroll da | Element viewport ga kirganda |
| **Performance** | Main thread da, throttle kerak | Brauzer optimized, async |
| **Threshold** | Manual hisoblash | Built-in (0-1) |
| **Memory** | Har element uchun handler | Bitta observer, ko'p target |

```javascript
// Lazy loading misol
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.src = entry.target.dataset.src;
      observer.unobserve(entry.target);
    }
  });
}, { rootMargin: "200px" }); // 200px oldindan yuklash

images.forEach(img => observer.observe(img));
```

**Deep Dive:**

Scroll event har frame da fire bo'ladi (60fps = 60 call/s). IntersectionObserver brauzer tomonidan optimized — idle time da, batched callback'lar bilan. `rootMargin` bilan preloading amalda har doim ishlatiladi.

---

### Savol 68: Web Worker nima? Qachon ishlatiladi?

**Javob:**

Web Worker — JavaScript ni **alohida thread** da ishlatish. Main thread (UI) bloklanmaydi.

```javascript
// main.js
const worker = new Worker("worker.js");
worker.postMessage(100000); // data yuborish
worker.onmessage = (e) => console.log("Natija:", e.data);

// worker.js
self.onmessage = (e) => {
  const result = heavyCalculation(e.data);
  self.postMessage(result); // natija qaytarish
};
```

Cheklov: Worker da **DOM ga kirish mumkin emas** (`document`, `window` yo'q). Communication faqat `postMessage` orqali (Structured Clone algorithmasi bilan serialize qilinadi).

Ishlatish: Og'ir hisob-kitoblar, data parsing, rasm qayta ishlash, kriptografiya.

**Deep Dive:**

`postMessage` data ni **copy** qiladi (Structured Clone). Katta data uchun **Transferable Objects** (ArrayBuffer) ishlatilgan — copy emas, ownership ko'chirish (zero-copy).

---

### Savol 69: localStorage va sessionStorage farqi nima? Cheklovlari?

**Javob:**

| | localStorage | sessionStorage |
|-|-------------|----------------|
| **Umr** | Abadiy (brauzer yopilsa ham) | Tab yopilsa o'chadi |
| **Scope** | Domain (barcha tab) | Faqat shu tab |
| **Hajm** | ~5-10MB | ~5-10MB |
| **API** | Sinxron (main thread bloklanadi) | Sinxron |

```javascript
// ❌ Keng tarqalgan xato
localStorage.setItem("user", { name: "Ali" }); // "[object Object]"!

// ✅ To'g'ri
localStorage.setItem("user", JSON.stringify({ name: "Ali" }));
const user = JSON.parse(localStorage.getItem("user"));
```

Cheklovlar: faqat string, sinxron (blocking), 5-10MB limit, private/incognito da o'chishi mumkin.

**Deep Dive:**

Katta yoki murakkab data uchun **IndexedDB** (250MB+, asinxron, structured data). `storage` event orqali boshqa tab'lardagi o'zgarishlarni eshitish mumkin (faqat boshqa tab'lardan, o'z tab dan emas).

---

### Savol 70: AbortController nima? Qanday ishlatiladi?

**Javob:**

AbortController — asinxron operatsiyalarni **bekor qilish** uchun mexanizm.

```javascript
const controller = new AbortController();

// Fetch bilan
fetch(url, { signal: controller.signal })
  .catch(e => {
    if (e.name === "AbortError") console.log("Bekor qilindi");
  });

// Event listener bilan
button.addEventListener("click", handler, { signal: controller.signal });

// Hammasi bir marta bilan bekor
controller.abort(); // fetch cancel + listener remove
```

Use cases:
- Request timeout
- Foydalanuvchi sahifadan chiqsa — pending request'larni cancel
- Search debounce — eski request cancel, yangi boshlash
- Component cleanup — barcha listener'larni bir marta bilan o'chirish

**Deep Dive:**

`AbortSignal.timeout(ms)` (yangi) — avtomatik timeout signal yaratadi: `fetch(url, { signal: AbortSignal.timeout(5000) })`. `AbortSignal.any([signal1, signal2])` — bir nechtasini birlashtirish.

---

## Error Handling

### Savol 71: try/catch/finally da finally ning xususiyatlari nima?

**Javob:**

`finally` bloki **doim** ishlaydi — xato bo'lsa ham, bo'lmasa ham, hatto `return` bo'lsa ham.

```javascript
// ⚠️ finally ichidagi return — try/catch dagi return ni override qiladi!
function tricky() {
  try {
    return "try";
  } finally {
    return "finally"; // ❌ "try" yo'qoldi!
  }
}
console.log(tricky()); // "finally"

// ⚠️ finally ichidagi return — throw ni ham yutadi!
function trickier() {
  try {
    throw new Error("muhim xato!");
  } finally {
    return "finally"; // ❌ Error yo'qoldi!
  }
}
console.log(trickier()); // "finally" — xato ko'rinmaydi
```

**Deep Dive:**

finally da faqat cleanup (resource yopish, connection tugatish) qilish kerak. finally ichida `return` yoki `throw` yozmang — bu try/catch natijalarini yo'qotadi. Resource management uchun (fayl, DB connection) finally ideal.

---

### Savol 72: Custom Error class qanday yaratiladi va nima uchun kerak?

**Javob:**

Error dan extend qilib, o'z error class larimizni yaratamiz:

```javascript
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
    this.statusCode = 400;
  }
}

// instanceof bilan tekshirish
try {
  throw new ValidationError("Email noto'g'ri", "email");
} catch (error) {
  if (error instanceof ValidationError) {
    console.log(`[${error.field}]: ${error.message}`);
  }
}
```

**Deep Dive:**

Custom Error kerak chunki: (1) `instanceof` bilan aniq tekshirish mumkin, (2) qo'shimcha ma'lumot saqlash (field, statusCode), (3) error hierarchy qurib, turli qatlamlarda turli xatolarni ushlash mumkin. `this.name = this.constructor.name` ishlatish tavsiya.

---

### Savol 73: async/await da xatolarni qanday to'g'ri boshqarish kerak?

**Javob:**

```javascript
// ✅ try/catch — eng keng tarqalgan
async function loadData() {
  try {
    const user = await fetchUser();
    const posts = await fetchPosts(user.id);
    return { user, posts };
  } catch (error) {
    console.error("Xato:", error);
    return null;
  }
}

// ⚠️ Muhim: async funksiyani catch siz chaqirmaslik!
async function risky() { throw new Error("xato"); }
risky(); // ❌ UnhandledPromiseRejection!
risky().catch(console.error); // ✅

// Promise.allSettled — barchasi natijasini olish
const results = await Promise.allSettled([fetchA(), fetchB()]);
results.forEach(r => {
  if (r.status === "rejected") console.log("Xato:", r.reason);
});
```

**Deep Dive:**

`Promise.all` bitta xato bo'lsa barchasini reject qiladi. `Promise.allSettled` har birining natijasini beradi. `Promise.any` birinchi muvaffaqiyatni kutadi, barchasi fail bo'lsa `AggregateError` tashlaydi.

---

### Savol 74: Error propagation qanday ishlaydi?

**Javob:**

Xato funksiya zanjiri bo'ylab yuqoriga ko'tariladi — birinchi `catch` blokgacha:

```javascript
function c() { throw new Error("xato"); } // ① throw
function b() { c(); }                     // ② catch yo'q → yuqoriga
function a() { b(); }                     // ③ catch yo'q → yuqoriga

try {
  a();                                     // ④ SHU YERDA ushlandi
} catch (e) {
  console.log(e.stack); // stack trace: c → b → a
}
```

**Deep Dive:**

Re-throwing pattern muhim: kerakli xatolarni ushlab, qolganlarini `throw error` bilan qayta ko'tarish. Error Translation pattern: past darajadagi xatoni yuqori darajadagi business xatoga aylantirish (HTTP error → NotFoundError).

---

### Savol 75: Error handling best practices ro'yxatingiz qanday?

**Javob:**

1. **Doim `Error` obyekti throw qiling** — string emas (stack trace uchun)
2. **Bo'sh catch yozmang** — kamida `console.error` qiling
3. **Specific error ushlang** — `instanceof` bilan
4. **Async xatolarni doim ushlang** — `.catch()` yoki `try/catch`
5. **finally da return/throw yozmang**
6. **Error cause ishlatish** — `new Error("msg", { cause: originalError })` (ES2022)
7. **Global handler qo'ying** — `unhandledrejection`, `window.onerror`

```javascript
// ✅ Ideal pattern:
async function safeOperation() {
  try {
    return await riskyOperation();
  } catch (error) {
    if (error instanceof NetworkError) {
      return await fallbackOperation(); // graceful degradation
    }
    throw new AppError("Operation failed", { cause: error });
  } finally {
    cleanup(); // doim ishlaydi
  }
}
```

**Deep Dive:**

Operational errors (kutilgan — network xato, validation) vs Programming errors (bug — TypeError, null reference) farqlash muhim. Operational xatolarni handle qilish mumkin, programming errors esa fix qilish kerak.

---

## Modern JavaScript

### Savol 76: Destructuring da default qiymat va alias qanday ishlaydi?

**Javob:**

```javascript
const user = { name: "Ali", age: 25 };

// Alias — nom o'zgartirish
const { name: userName } = user;
console.log(userName); // "Ali"
// console.log(name); // ❌ ReferenceError — faqat userName bor

// Default — yo'q bo'lsa
const { role = "user" } = user;
console.log(role); // "user"

// Alias + Default birga
const { role: userRole = "user" } = user;
console.log(userRole); // "user"

// ⚠️ Default faqat undefined da ishlaydi, null da EMAS
const { val = 10 } = { val: null };
console.log(val); // null — default ishlamadi!
```

**Deep Dive:**

Nested destructuring da default berish muhim: `const { address: { city } = {} } = user` — agar address undefined bo'lsa, TypeError o'rniga bo'sh object destructure qilinadi.

---

### Savol 77: `||` va `??` operatorlari orasidagi farq nima?

**Javob:**

```javascript
// || — barcha FALSY qiymatlar da default beradi
// ?? — faqat NULL va UNDEFINED da default beradi

console.log(0 || 10);   // 10 — 0 falsy
console.log(0 ?? 10);   // 0  — 0 null/undefined emas

console.log("" || "default");  // "default" — "" falsy
console.log("" ?? "default");  // ""       — "" null emas

console.log(false || true);  // true  — false falsy
console.log(false ?? true);  // false — false null emas
```

**Deep Dive:**

`??` ko'p holatlarda `||` dan xavfsizroq — 0, "", false kabi lekin ahamiyatli qiymatlarni yo'qotmaydi. `??` ni `||` yoki `&&` bilan qavsarsiz aralashtirib bo'lmaydi — `SyntaxError`.

---

### Savol 78: Spread operator shallow copy qilishini tushuntiring.

**Javob:**

```javascript
const original = {
  name: "Ali",
  address: { city: "Toshkent" } // nested object
};

const copy = { ...original };
copy.name = "Vali";         // ✅ original o'zgarmaydi
copy.address.city = "Buxoro"; // ❌ original HAM o'zgaradi!

console.log(original.name);          // "Ali" — OK
console.log(original.address.city);  // "Buxoro" — ❌ o'zgardi!
```

**Deep Dive:**

Spread 1-chi darajadagi property'larni nusxalaydi, nested object/array lar uchun reference copy qiladi. Deep copy uchun `structuredClone()` (ES2022) ishlatish kerak. `JSON.parse(JSON.stringify())` ham ishlaydi, lekin Date, Map, Set, Function, circular reference bilan muammo.

---

### Savol 79: Optional chaining `?.` qanday ishlaydi va qachon ishlatiladi?

**Javob:**

```javascript
// ?. — null/undefined bo'lsa xato bermaydi, undefined qaytaradi
const user = { name: "Ali" };

console.log(user?.address?.city);  // undefined (xato yo'q)
console.log(user?.getName?.());    // undefined (method yo'q)
console.log(users?.[0]?.name);     // undefined (array element)

// ⚠️ Short-circuit — null/undefined bo'lsa DAVOM ETMAYDI
null?.a.b.c; // undefined — .a ga ham bormaydi

// ⚠️ Yozish (assignment) da ishlatib BO'LMAYDI
// user?.name = "Vali"; // ❌ SyntaxError
```

**Deep Dive:**

`?.` API response yoki konfiguratsiya kabi noaniq tuzilmalar bilan ishlashda ideal. Lekin ortiqcha ishlatmaslik — `a?.b?.c?.d?.e` ko'p bo'lsa, data model ni qayta ko'rib chiqish kerak.

---

### Savol 80: Tagged template literals nima va qanday ishlaydi?

**Javob:**

```javascript
function tag(strings, ...values) {
  // strings = statik qismlar massivi
  // values = interpolated qiymatlar
  console.log(strings); // ["Salom, ", "! Yosh: ", ""]
  console.log(values);  // ["Ali", 25]
  return strings.reduce((r, s, i) =>
    r + s + (values[i] !== undefined ? `<b>${values[i]}</b>` : ""), ""
  );
}

const name = "Ali", age = 25;
tag`Salom, ${name}! Yosh: ${age}`;
// "Salom, <b>Ali</b>! Yosh: <b>25</b>"
```

**Deep Dive:**

Real-world ishlatish: `styled-components` (CSS-in-JS), `lit-html` (HTML templates), `graphql-tag` (GraphQL queries), SQL injection himoyasi. `String.raw` ham tagged template — escape sequence larni qayta ishlamaslik uchun.

---

## Array Methods

### Savol 81: `map`, `filter`, `reduce` farqi nima va qachon qaysi birini ishlatish kerak?

**Javob:**

- **`map`** — har bir elementni o'zgartirish, yangi array qaytaradi (1:1)
- **`filter`** — shartga mos elementlarni ajratish, yangi array qaytaradi (1:0 yoki 1:1)
- **`reduce`** — bar array ni bitta qiymatga kamaytirish (N:1)

```javascript
const nums = [1, 2, 3, 4, 5];
nums.map(n => n * 2);           // [2, 4, 6, 8, 10]
nums.filter(n => n > 3);        // [4, 5]
nums.reduce((s, n) => s + n, 0); // 15
```

**Deep Dive:**

`reduce` eng kuchli — `map` va `filter` ni ham reduce orqali yozish mumkin. Lekin readability uchun aniq metodni ishlatish yaxshiroq: transformatsiya = map, filtrlash = filter, agregatsiya = reduce.

---

### Savol 82: `sort()` nima uchun xavfli va qanday to'g'ri ishlatish kerak?

**Javob:**

```javascript
// 1. sort() MUTATES — original array o'zgaradi!
const arr = [3, 1, 2];
arr.sort();
console.log(arr); // [1, 2, 3] — o'zgargan!

// 2. Default sort STRING sifatida!
[10, 9, 2, 100].sort(); // [10, 100, 2, 9] — noto'g'ri!

// ✅ To'g'ri
const sorted = [...arr].sort((a, b) => a - b); // immutable + comparator
// Yoki ES2023: arr.toSorted((a, b) => a - b)
```

**Deep Dive:**

ES2019 dan beri barcha brauzerlar **stable sort** qiladi (TimSort). ES2023 da `toSorted`, `toReversed`, `toSpliced`, `with` — immutable variantlar qo'shildi.

---

### Savol 83: `find` vs `filter`, `some` vs `every` farqi nima?

**Javob:**

- `find` — birinchi mos **element** | `filter` — barcha mos **elementlar**
- `some` — kamida bitta mos bor? (**OR**) | `every` — barchasi mos? (**AND**)

```javascript
const nums = [1, 2, 3, 4, 5];
nums.find(n => n > 3);    // 4 (birinchi mos)
nums.filter(n => n > 3);  // [4, 5] (barcha mos)
nums.some(n => n > 4);    // true
nums.every(n => n > 0);   // true

// ⚠️ Bo'sh array
[].some(() => true);  // false
[].every(() => false); // true (vacuously true)
```

**Deep Dive:**

`find` va `some` birinchi mos topilganda **to'xtaydi** (short-circuit). `filter` va `every` barcha elementlarni ko'radi. Performance jihatdan `find`/`some` tezroq.

---

### Savol 84: `flatMap` nima va qachon ishlatiladi?

**Javob:**

```javascript
// flatMap = map + flat(1)
const sentences = ["salom dunyo", "javascript zo'r"];

// map → nested
sentences.map(s => s.split(" ")); // [["salom","dunyo"],["javascript","zo'r"]]

// flatMap → tekis
sentences.flatMap(s => s.split(" ")); // ["salom","dunyo","javascript","zo'r"]

// Filter + transform birga
[1,2,3,4].flatMap(n => n % 2 === 0 ? [n, n*10] : []);
// [2, 20, 4, 40]
```

**Deep Dive:**

`flatMap` faqat 1 daraja tekislaydi. Agar chuqurroq kerak — `flat(Infinity)` ishlatish kerak. `flatMap` bir elementdan ko'p yoki nol element qaytarish uchun ideal (1:N mapping).

---

### Savol 85: Immutable array operatsiyalari qanday qilinadi?

**Javob:**

```javascript
const arr = [1, 2, 3];

// push → spread
const added = [...arr, 4];           // [1, 2, 3, 4]

// pop → slice
const removed = arr.slice(0, -1);    // [1, 2]

// sort → toSorted (ES2023) yoki spread+sort
const sorted = arr.toSorted((a,b) => b-a); // [3, 2, 1]

// splice → toSpliced (ES2023)
const spliced = arr.toSpliced(1, 1, 99);   // [1, 99, 3]

// update by index → with (ES2023) yoki map
const updated = arr.with(0, 100);          // [100, 2, 3]
```

**Deep Dive:**

ES2023 immutable metodlari: `toSorted`, `toReversed`, `toSpliced`, `with`. Eski usul: `[...arr].sort()`, `arr.slice()`, `arr.map()`. React/Redux kabi framework'larda immutability muhim — state ni to'g'ridan-to'g'ri o'zgartirish bug'lar keltirib chiqaradi.

---

## Proxy va Reflect

### Savol 86: Proxy nima va qanday ishlaydi?

**Javob:**

Proxy — object ga wrapper qo'yib, uning operatsiyalarini (o'qish, yozish, o'chirish) ushlash imkonini beradi:

```javascript
const user = { name: "Ali" };
const proxy = new Proxy(user, {
  get(target, prop) {
    console.log(`GET ${prop}`);
    return Reflect.get(target, prop);
  },
  set(target, prop, value) {
    console.log(`SET ${prop} = ${value}`);
    return Reflect.set(target, prop, value);
  }
});
proxy.name;        // "GET name" → "Ali"
proxy.age = 25;    // "SET age = 25"
```

**Deep Dive:**

13 ta trap mavjud: get, set, has, deleteProperty, apply, construct, ownKeys va boshqalar. Proxy revocable bo'lishi mumkin: `Proxy.revocable()` — kerak bo'lmasa bekor qilish mumkin.

---

### Savol 87: Reflect nima uchun kerak va Proxy ichida ishlatish nima uchun tavsiya?

**Javob:**

```javascript
// ❌ target[prop] — ba'zi edge case larda xato
const handler = {
  get(target, prop) {
    return target[prop]; // prototype getter da this noto'g'ri
  }
};

// ✅ Reflect.get — receiver bilan to'g'ri this
const handler = {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);
  }
};
```

**Deep Dive:**

Reflect har bir Proxy trap ga mos keladigan statik metod beradi. `Reflect.set` boolean qaytaradi (exception emas), `Reflect.ownKeys` Symbol va non-enumerable ham beradi (`Object.keys` dan farqli).

---

### Savol 88: Proxy bilan validation qanday qilinadi?

**Javob:**

```javascript
const user = new Proxy({}, {
  set(target, prop, value) {
    if (prop === "age" && (typeof value !== "number" || value < 0)) {
      throw new TypeError("age musbat raqam bo'lishi kerak");
    }
    if (prop === "email" && !value.includes("@")) {
      throw new Error("email noto'g'ri");
    }
    return Reflect.set(target, prop, value);
  }
});
user.age = 25;              // ✅
// user.age = "yigirma";    // ❌ TypeError
```

**Deep Dive:**

Proxy validation runtime da ishlaydi — TypeScript compile-time. Ikkalasi birgalikda ishlatilsa — eng kuchli. Proxy validation form input, API data, config fayllar uchun foydali.

---

### Savol 89: Vue 3 reactivity Proxy asosida qanday ishlaydi?

**Javob:**

Vue 3 `reactive()` Proxy orqali:
- **get trap** — property o'qilganda **track** (qaysi effect bu property ga bog'liq? saqlash)
- **set trap** — property yozilganda **trigger** (bog'liq effect'larni qayta ishga tushirish)

```javascript
// Soddalashtirilgan:
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key);    // dependency saqlash
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver);
      trigger(target, key);  // UI yangilash
      return result;
    }
  });
}
```

**Deep Dive:**

Vue 2 `Object.defineProperty` ishlatardi — yangi property qo'shish, array index o'zgartirish bilan muammo. Vue 3 Proxy bilan bu muammolar yo'q: dynamic props, array, Map/Set — barchasi reaktiv.

---

### Savol 90: Proxy da Revocable Proxy nima?

**Javob:**

```javascript
const { proxy, revoke } = Proxy.revocable({ name: "Ali" }, {
  get(target, prop) { return Reflect.get(target, prop); }
});

console.log(proxy.name); // "Ali" — ishlaydi
revoke();                // Proxy bekor qilindi!
// proxy.name;           // ❌ TypeError: Cannot perform 'get' on a revoked proxy
```

**Deep Dive:**

Revocable proxy xavfsizlik uchun: third-party kodga vaqtinchalik access berish va keyin bekor qilish. API token, session, resource access control kabi holatlarda foydali. Revoke qilingandan keyin hech qanday operatsiya bajarib bo'lmaydi.

---

## Design Patterns

### Savol 91: Observer (Pub/Sub) pattern nima va qanday ishlatiladi?

**Javob:**

Observer — bir object o'zgarganda, barcha subscriber'larni avtomatik xabardor qilish:

```javascript
class EventEmitter {
  #events = new Map();
  on(event, cb) {
    if (!this.#events.has(event)) this.#events.set(event, new Set());
    this.#events.get(event).add(cb);
    return () => this.#events.get(event)?.delete(cb); // unsub
  }
  emit(event, ...args) {
    this.#events.get(event)?.forEach(cb => cb(...args));
  }
}

const bus = new EventEmitter();
const unsub = bus.on("update", data => console.log(data));
bus.emit("update", { count: 1 }); // { count: 1 }
unsub(); // listener tozalash
```

**Deep Dive:**

DOM `addEventListener`, Node.js `EventEmitter`, Redux, Vue reactivity — barchasi Observer pattern. Memory leak oldini olish uchun `unsub` chaqirish **majburiy**.

---

### Savol 92: Factory pattern nima va qachon ishlatiladi?

**Javob:**

Factory — object yaratish logikasini markazlashtirish:

```javascript
function createUser(type, data) {
  const roles = {
    admin: { permissions: ["read","write","delete"], dashboard: "/admin" },
    editor: { permissions: ["read","write"], dashboard: "/editor" },
    viewer: { permissions: ["read"], dashboard: "/" }
  };
  if (!roles[type]) throw new Error(`Unknown: ${type}`);
  return { ...data, role: type, ...roles[type] };
}

const admin = createUser("admin", { name: "Ali" });
```

**Deep Dive:**

Factory client kodni yaratish logikasidan ajratadi. Yangi tur qo'shish oson — faqat factory ichini o'zgartirish kerak. `document.createElement()` ham Factory pattern.

---

### Savol 93: Singleton pattern JS da qanday qilinadi?

**Javob:**

```javascript
// 1. Class bilan
class Config {
  static #instance = null;
  constructor() {
    if (Config.#instance) return Config.#instance;
    this.settings = {};
    Config.#instance = this;
  }
}
new Config() === new Config(); // true

// 2. ES Module — eng oddiy singleton!
// config.js
export default { theme: "light" };
// Har qanday import = BIR XIL object
```

**Deep Dive:**

ES Module o'zi singleton — birinchi importda cache'lanadi. Shuning uchun alohida Singleton class kamdan-kam kerak. Test uchun reset mexanizmi qo'shish tavsiya.

---

### Savol 94: Strategy pattern nima va real-world misol keltiring.

**Javob:**

Strategy — algoritmni alohida qilib, runtime da almashtirish:

```javascript
const sorters = {
  name: (a, b) => a.name.localeCompare(b.name),
  age: (a, b) => a.age - b.age,
  date: (a, b) => new Date(a.date) - new Date(b.date)
};

function sortUsers(users, strategy = "name") {
  return [...users].sort(sorters[strategy]);
}

sortUsers(users, "age");  // yosh bo'yicha
sortUsers(users, "name"); // ism bo'yicha
```

**Deep Dive:**

Strategy pattern if/else zanjirini yo'qotadi. Yangi strategiya qo'shish = bitta function qo'shish. Validatsiya, narx hisoblash, autentifikatsiya kabi joylarda keng ishlatiladi.

---

### Savol 95: Middleware pattern nima va qanday ishlaydi?

**Javob:**

Middleware — so'rovni zanjir bo'ylab qayta ishlash (Express.js, Redux asosi):

```javascript
class Pipeline {
  #fns = [];
  use(fn) { this.#fns.push(fn); return this; }
  async run(ctx) {
    let i = 0;
    const next = async () => {
      if (i < this.#fns.length) await this.#fns[i++](ctx, next);
    };
    await next();
    return ctx;
  }
}

const app = new Pipeline();
app.use(async (ctx, next) => { console.log("→", ctx.url); await next(); });
app.use(async (ctx, next) => { ctx.status = 200; await next(); });
app.run({ url: "/api" });
```

**Deep Dive:**

`next()` chaqirmaslik = zanjirni to'xtatish (auth fail kabi). Har bir middleware `next()` dan oldin va keyin kod bajara oladi (onion model). Express, Koa, Redux middleware — shu pattern.
