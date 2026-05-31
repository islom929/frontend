# Functions — First-Class Citizens — Interview Savollari

> Function Declaration vs Expression vs Arrow, First-class functions, IIFE, HOF, Callbacks, Pure Functions, Currying, Partial Application, Composition, Memoization, Debounce, Throttle, arguments vs rest, default params, function name/length properties haqida interview savollari.
> **Savollar soni:** 29
> **Taqsimot:** Nazariy ~45% | Output ~28% | Coding ~21% | Tuzatish ~7%

---

## Nazariy savollar

### 1. Function Declaration, Function Expression va Arrow Function — farqlari nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

JavaScript da funksiya yaratishning **3 ta asosiy usuli** bor. Ularning farqi hoisting, `this` binding, `arguments`, va `new` bilan chaqirish imkoniyatlarida namoyon bo'ladi.

```javascript
// 1. Function Declaration — to'liq hoist bo'ladi
function greet(name) {
  return `Salom, ${name}`;
}

// 2. Function Expression — hoist bo'lmaydi (TDZ yoki undefined)
const greet2 = function(name) {
  return `Salom, ${name}`;
};

// 3. Arrow Function — hoist bo'lmaydi, this/arguments yo'q
const greet3 = (name) => `Salom, ${name}`;
```

| Xususiyat | Declaration | Expression | Arrow |
|-----------|------------|------------|-------|
| Hoisting | ✅ To'liq (nom + tana) | ❌ TDZ (`const`/`let`) yoki `undefined` (`var`) | ❌ TDZ |
| `this` | Dynamic (call-site ga bog'liq) | Dynamic | Lexical (tashqi scope dan) |
| `arguments` | ✅ Bor | ✅ Bor | ❌ Yo'q |
| `new` bilan | ✅ Constructor bo'la oladi | ✅ Constructor bo'la oladi | ❌ TypeError |
| `prototype` | ✅ Bor | ✅ Bor | ❌ Yo'q |

**Qo'shimcha — arrow function va `this`:**

Arrow function ichki `[[ThisMode]]: "lexical"` slotiga ega. Bu degani engine `this` ni resolve qilganda, arrow function ning o'zida `this` qidirmaydi — **tashqi scope** ga chiqadi. Shuning uchun `call`, `apply`, `bind` arrow function ning `this` ini o'zgartira olmaydi:

```javascript
const arrow = () => console.log(this);
arrow.call({ name: "Ali" }); // module top-level: undefined; script: globalThis — call e'tiborga olinmadi!
```

</details>

### 2. First-class function nima degani? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

JavaScript da funksiyalar — **birinchi darajali fuqaro (first-class citizen)**. Bu degani funksiyalar boshqa qiymatlar (`number`, `string`, `object`) kabi muomala qilinadi:

```javascript
// 1. O'zgaruvchiga saqlash
const fn = function() { return 42; };

// 2. Argument sifatida berish (callback)
[1, 2, 3].map(n => n * 2); // [2, 4, 6]

// 3. Funksiyadan qaytarish (factory/closure)
function multiplier(factor) {
  return (num) => num * factor; // funksiya qaytaryapti
}
const double = multiplier(2);
double(5); // 10

// 4. Object property sifatida
const obj = { run: () => console.log("running") };

// 5. Array elementida saqlash
const pipeline = [Math.abs, Math.sqrt, Math.round];
```

**Spec — callable object:**

ECMAScript spec bo'yicha funksiya — **callable object**. `[[Call]]` internal method mavjudligi uni chaqiriladigan qiladi. Funksiya object bo'lganligi uchun unga property ham qo'shish mumkin:

```javascript
function validate(email) { return /@/.test(email); }
validate.errorMsg = "Email noto'g'ri"; // funksiyaga property qo'shish
console.log(validate.name);   // "validate"
console.log(validate.length); // 1 (parametr soni)
```

</details>

### 3. IIFE nima? Nima uchun ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

**IIFE** (Immediately Invoked Function Expression) — funksiya yaratilishi bilan darhol chaqiriladi. Scope isolation'i uchun ishlatiladi.

```javascript
// Klassik IIFE
(function() {
  const secret = "maxfiy";
  console.log(secret); // "maxfiy"
})();
// console.log(secret); // ❌ ReferenceError — tashqaridan ko'rinmaydi

// Arrow IIFE
(() => {
  console.log("Arrow IIFE ishladi!");
})();

// Parametrli IIFE
(function(name) {
  console.log(`Salom, ${name}!`);
})("Ali"); // "Salom, Ali!"
```

**Qachon kerak:**
1. **Legacy kod** — `var` davrida global scope ni himoya qilish
2. **Module Pattern** — public/private ajratish
3. **Async initialization** — top-level await yo'q bo'lsa:

```javascript
(async () => {
  const data = await fetch("/api/config");
  const config = await data.json();
  initApp(config);
})();
```

**Zamonaviy kodda** ES modules va `let`/`const` block scope IIFE ning ko'p use case larini almashtirdi.

</details>

### 4. Higher-Order Function (HOF) nima? Misol bering. [Junior+]

<details>
<summary><strong>Javob</strong></summary>

HOF — boshqa funksiyani **argument sifatida qabul qiladigan** yoki **funksiyani qaytaradigan** funksiya.

```javascript
// 1. Funksiyani QABUL qiladigan HOF (callback pattern)
function repeat(n, action) {
  for (let i = 0; i < n; i++) action(i);
}
repeat(3, console.log); // 0, 1, 2

// 2. Funksiyani QAYTARADIGAN HOF (factory pattern)
function greeter(greeting) {
  return function(name) {
    return `${greeting}, ${name}!`;
  };
}
const salom = greeter("Salom");
salom("Ali"); // "Salom, Ali!"

// 3. Ikkalasi ham — Decorator pattern
function withLogging(fn) {       // funksiyani qabul qiladi
  return function(...args) {      // yangi funksiya qaytaradi
    console.log(`Chaqirildi: ${fn.name}(${args})`);
    const result = fn(...args);
    console.log(`Natija: ${result}`);
    return result;
  };
}

const add = (a, b) => a + b;
const loggedAdd = withLogging(add);
loggedAdd(2, 3);
// Chaqirildi: add(2,3)
// Natija: 5
```

Built-in HOF lar: `map`, `filter`, `reduce`, `forEach`, `sort`, `setTimeout`, `addEventListener`, `Promise.then`.

</details>

### 5. Callback nima? Callback hell nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Callback — boshqa funksiyaga **argument sifatida beriladigan** funksiya. U HOF tomonidan kerak bo'lganda chaqiriladi.

```javascript
// Sinxron callback
const numbers = [3, 1, 4, 1, 5];
numbers.sort((a, b) => a - b); // callback: taqqoslash qoidasi

// Asinxron callback
setTimeout(() => {
  console.log("2 sekunddan keyin");
}, 2000);
```

**Callback Hell** — ichma-ich callback lar o'qilmaydigan kod hosil qiladi:

```javascript
// ❌ Callback hell — "pyramid of doom"
getUser(id, function(user) {
  getOrders(user.id, function(orders) {
    getProducts(orders[0].id, function(products) {
      getReviews(products[0].id, function(reviews) {
        console.log(reviews); // 4 qavat chuqurlik
      });
    });
  });
});

// ✅ Yechim — Promise chain yoki async/await
async function getData(id) {
  const user = await getUser(id);
  const orders = await getOrders(user.id);
  const products = await getProducts(orders[0].id);
  const reviews = await getReviews(products[0].id);
  return reviews;
}
```

</details>

### 6. Pure Function nima? Side Effect nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Pure function **ikki shartni** bajaradi:
1. **Deterministic** — bir xil input doim bir xil output beradi
2. **No side effects** — tashqi state ni o'zgartirmaydi

```javascript
// ✅ Pure function
function add(a, b) {
  return a + b; // faqat inputlarga bog'liq, hech narsani o'zgartirmaydi
}

// ✅ Pure function
function formatName(first, last) {
  return `${first} ${last}`; // yangi string qaytaradi
}

// ❌ Impure — tashqi state ga BOG'LIQ
let taxRate = 0.2;
function calcPrice(price) {
  return price + price * taxRate; // taxRate o'zgarsa — natija o'zgaradi
}

// ❌ Impure — tashqi state ni O'ZGARTIRADI (side effect)
let total = 0;
function addToTotal(amount) {
  total += amount;  // mutation — side effect!
  return total;
}

// ❌ Impure — side effect (console, DOM, network, random, date)
function logResult(x) {
  console.log(x);  // side effect — I/O
  return x;
}
```

**Side Effect turlari:**
- Tashqi o'zgaruvchini o'zgartirish (mutation)
- `console.log`, DOM manipulation, network request
- `Math.random()`, `Date.now()` — nondeterministic
- File system, database yozish

**Nima uchun muhim:** Pure function larni test qilish oson, memoize qilsa bo'ladi, parallellashtirsa bo'ladi. **React, Redux, functional programming** — hammasi pure function larga asoslangan.

</details>

### 7. `arguments` vs Rest Parameters — farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

```javascript
// arguments — array-LIKE object (eski usul)
function oldWay() {
  console.log(arguments);         // { 0: "a", 1: "b", length: 2 }
  console.log(Array.isArray(arguments)); // false
  // arguments.map(...) — ❌ TypeError! map metodi yo'q

  // Array ga aylantirish kerak:
  const arr = Array.from(arguments); // yoki [...arguments]
  arr.map(x => x.toUpperCase()); // ✅ endi ishlaydi
}

// rest parameters — haqiqiy Array (zamonaviy usul)
function newWay(...args) {
  console.log(args);              // ["a", "b"]
  console.log(Array.isArray(args)); // true
  args.map(x => x.toUpperCase()); // ✅ darhol ishlaydi
}

// Rest bilan boshqa parametrlarni ajratish
function logMessage(level, ...messages) {
  // level = "error"
  // messages = ["Server down", "DB timeout"] — qolganlari
  messages.forEach(msg => console.log(`[${level}] ${msg}`));
}
logMessage("error", "Server down", "DB timeout");
```

| Xususiyat | `arguments` | Rest `...args` |
|-----------|-------------|----------------|
| Turi | Array-like Object | Haqiqiy Array |
| Arrow function | ❌ Yo'q | ✅ Ishlaydi |
| Array methods | ❌ (`map`, `filter` yo'q) | ✅ Hammasi bor |
| Boshqa params bilan | ❌ Ajratib bo'lmaydi | ✅ `(a, b, ...rest)` |
| Destructuring | ❌ | ✅ |

**Xulosa:** Doim `...rest` parametrlarini ishlating. `arguments` — faqat legacy kod da uchraydi.

</details>

### 8. Default Parameters qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

ES6 dan beri parametrlarga default qiymat berish mumkin. Default qiymat faqat argument `undefined` bo'lganda ishlatiladi (`null` emas!).

```javascript
function createUser(name, role = "user", active = true) {
  return { name, role, active };
}

createUser("Ali");                    // { name: "Ali", role: "user", active: true }
createUser("Ali", "admin");           // { name: "Ali", role: "admin", active: true }
createUser("Ali", undefined, false);  // { name: "Ali", role: "user", active: false }
createUser("Ali", null);              // { name: "Ali", role: null, active: true }
//                 ^^^^ null !== undefined, shuning uchun default ISHLAMAYDI
```

**Tricky case — default qiymatda expression:**

```javascript
// Default qiymat EXPRESSION bo'lishi mumkin — har chaqiruvda baholanadi
function addItem(item, list = []) {
  list.push(item);
  return list;
}

console.log(addItem("a"));       // ["a"]      — yangi array
console.log(addItem("b"));       // ["b"]      — YANA yangi array (har safar!)
// Python dan farq! Python da default mutable object bir marta yaratiladi

// Default qiymatda oldingi parametrga murojaat
function greet(name, greeting = `Salom, ${name}!`) {
  return greeting;
}
greet("Ali"); // "Salom, Ali!"
```

</details>

### 9. Currying nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Currying — ko'p argumentli funksiyani **birma-bir argument qabul qiluvchi** funksiyalar zanjiriga aylantirish: `f(a, b, c)` → `f(a)(b)(c)`.

```javascript
// Oddiy funksiya
function add(a, b, c) { return a + b + c; }
add(1, 2, 3); // 6

// Qo'lda curried versiya
function curriedAdd(a) {
  return function(b) {
    return function(c) {
      return a + b + c;
    };
  };
}
curriedAdd(1)(2)(3); // 6
```

| | Currying | Partial Application |
|---|---------|---------------------|
| Argumentlar | Doim birma-bir | Bir nechtasini oldindan berish |
| Qaytaradi | Zanjir funksiyalar | Bitta yangi funksiya |
| `f(a,b,c)` | `f(a)(b)(c)` | `g = f(a, b); g(c)` |
| JavaScript da | Maxsus `curry` kerak | `bind` yoki wrapper |

</details>

### 10. Partial Application nima? Currying dan farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

**Partial Application** — funksiyaning **ba'zi argumentlarini oldindan berish** va qolganlarini keyinroq kutuvchi yangi funksiya olish.

```javascript
// Currying: f(a, b, c) → f(a)(b)(c) — DOIM birma-bir
// Partial:  f(a, b, c) → g(c) — ba'zilarini oldindan berib yangi fn olish

// bind bilan partial application
function greet(greeting, name) {
  return `${greeting}, ${name}!`;
}
const sayHello = greet.bind(null, "Hello"); // greeting = "Hello" qotirildi
sayHello("Ali");  // "Hello, Ali!"
sayHello("Vali"); // "Hello, Vali!"

// O'z partial funksiyamiz
function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

const multiply = (a, b) => a * b;
const double = partial(multiply, 2); // a = 2 qotirildi
double(5); // 10
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Output nima bo'ladi? [Junior+]

```javascript
console.log(a());  // ?
console.log(b());  // ?
console.log(c());  // ?

function a() { return "A"; }
var b = function() { return "B"; };
const c = () => "C";
```

<details>
<summary><strong>Javob</strong></summary>
- `a()` → `"A"` ✅ — Function Declaration to'liq hoist bo'ladi
- `b()` → ❌ `TypeError: b is not a function` — `var b` hoist bo'ladi lekin `undefined` qiymati bilan, funksiya tayinlanmagan
- `c()` ga yetib bormaydi — `b()` da xato bo'lganligi uchun dastur to'xtaydi. Agar alohida tekshirsak: ❌ `ReferenceError: Cannot access 'c' before initialization` — `const` TDZ da

**Tushuntirish:** Hoisting davomida `function a` to'liq ko'tariladi. `var b` faqat o'zgaruvchi sifatida `undefined` bilan ko'tariladi. `const c` esa TDZ (Temporal Dead Zone) da qoladi.

</details>

### 2. Output nima bo'ladi? [Middle]

```javascript
function showArgs(a, b, c) {
  console.log(arguments.length);  // ?
  console.log(arguments[2]);      // ?

  arguments[0] = 100;
  console.log(a);                 // ?
}

showArgs(1, 2, 3);
```

<details>
<summary><strong>Javob</strong></summary>
- `arguments.length` → `3` — 3 ta argument berilgan
- `arguments[2]` → `3` — uchinchi argument
- `a` → `100` ✅ — **non-strict mode** va **simple parameter list** (default/rest/destructuring yo'q) bilan `arguments` va named parameter lar **bog'langan** (ParameterMap orqali). `arguments[0]` o'zgarganda `a` ham o'zgaradi.

**Muhim:** ParameterMap (bog'lanish) quyidagi holatlarda **buziladi**:

```javascript
// 1. Strict mode
"use strict";
function f1(a) {
  arguments[0] = 100;
  console.log(a); // 1 — bog'lanish yo'q
}

// 2. Non-strict mode, lekin default parameter bor
function f2(a = 0) {  // default → ParameterMap bo'sh
  arguments[0] = 100;
  console.log(a); // 1 — bog'lanish yo'q, hatto non-strict'da ham
}

// 3. Non-strict mode, lekin rest parameter bor
function f3(a, ...rest) {
  arguments[0] = 100;
  console.log(a); // 1 — bog'lanish yo'q
}

// 4. Non-strict mode, lekin destructuring parameter bor
function f4({a}) {
  arguments[0] = 100;
  console.log(a); // original qiymat — bog'lanish yo'q
}
```

Spec bo'yicha: **har qanday ES6 parameter feature** (default, rest, destructuring) ParameterMap'ni o'chiradi — hatto non-strict mode'da ham. Faqat klassik `function(a, b, c)` simple list non-strict'da bog'lanishga ega.

Arrow function da `arguments` **umuman yo'q** — tashqi scope dan qidiradi.

</details>

### 3. Output nima bo'ladi? [Middle]

```javascript
const funcs = [];
for (var i = 0; i < 3; i++) {
  funcs.push(function() { return i; });
}
console.log(funcs[0]()); // ?
console.log(funcs[1]()); // ?
console.log(funcs[2]()); // ?
```

<details>
<summary><strong>Javob</strong></summary>

Hammasi `3` qaytaradi.

`var` — function-scoped, bitta `i` bor. Loop tugaganda `i = 3`. Barcha 3 ta funksiya **bir xil** `i` ga closure qilgan — ularning hech biri o'z nusxasiga ega emas.

**Yechimlar:**

```javascript
// ✅ Yechim 1: let ishlatish (block scope — har iteratsiyada yangi i)
for (let i = 0; i < 3; i++) {
  funcs.push(function() { return i; });
}
// funcs[0]() → 0, funcs[1]() → 1, funcs[2]() → 2

// ✅ Yechim 2: IIFE bilan (har iteratsiyada yangi scope)
for (var i = 0; i < 3; i++) {
  funcs.push((function(j) {
    return function() { return j; };
  })(i));
}
```


</details>

### 4. Universal `curry` funksiyasini implement qiling. [Middle+]

<details>
<summary><strong>Javob</strong></summary>

**Universal `curry` implementation:**

```javascript
function curry(fn) {
  return function curried(...args) {
    // Argumentlar yetarli bo'lsa — asl funksiyani chaqir
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    // Yetmasa — yangi funksiya qaytar, oldingi args ni eslab qol (closure)
    return function(...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

// Ishlatish
const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);     // 6
add(1, 2)(3);     // 6
add(1)(2, 3);     // 6
add(1, 2, 3);     // 6 — oddiy chaqiruv ham ishlaydi

// Amaliy misol
const multiply = curry((a, b) => a * b);
const double = multiply(2);  // partial application
const triple = multiply(3);
double(5); // 10
triple(5); // 15
```

**Tushuntirish:**

`fn.length` — funksiya kutayotgan parametrlar soni (rest va default parametrlar hisobga olinmaydi). Curry har safar argumentlar yetarli ekanini tekshiradi. Yetmasa — closure orqali avvalgi argumentlarni saqlab yangi funksiya qaytaradi. Bu **lazy evaluation** printsipi.

</details>

### 5. Function Composition nima? `pipe` va `compose` ni implement qiling. [Senior]

<details>
<summary><strong>Javob</strong></summary>

Function Composition — bir necha kichik funksiyalarni **birlashtirish** orqali yangi funksiya hosil qilish. `compose(f, g)(x)` = `f(g(x))`.

```javascript
// Oddiy misol — qo'lda composition
const trim = str => str.trim();
const toLower = str => str.toLowerCase();
const split = str => str.split(" ");

// Qo'lda
const result = split(toLower(trim("  SALOM DUNYO  ")));
// ["salom", "dunyo"]

// compose — O'NGDAN CHAPGA (matematikadagi kabi)
function compose(...fns) {
  return function(value) {
    return fns.reduceRight((acc, fn) => fn(acc), value);
  };
}

const processText = compose(split, toLower, trim);
processText("  SALOM DUNYO  "); // ["salom", "dunyo"]
// Bajarilish: trim → toLower → split (o'ngdan chapga o'qiladi)

// pipe — CHAPDAN O'NGGA (o'qishga qulay)
function pipe(...fns) {
  return function(value) {
    return fns.reduce((acc, fn) => fn(acc), value);
  };
}

const processText2 = pipe(trim, toLower, split);
processText2("  SALOM DUNYO  "); // ["salom", "dunyo"]
// Bajarilish: trim → toLower → split (chapdan o'ngga, tabiiy tartib)
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Amaliy misol — API response pipeline:**

```javascript
// API dan kelgan data ni qayta ishlash pipeline
const parseJSON = str => JSON.parse(str);
const extractUsers = data => data.users;
const filterActive = users => users.filter(u => u.active);
const sortByName = users => [...users].sort((a, b) => a.name.localeCompare(b.name));
const formatNames = users => users.map(u => `${u.name} (${u.role})`);

const processResponse = pipe(
  parseJSON,
  extractUsers,
  filterActive,
  sortByName,
  formatNames
);

// processResponse('{"users": [...]}') → ["Ali (admin)", "Vali (user)"]
```

**Spec va internal:**

`pipe` va `compose` pattern ECMAScript spec'da maxsus qo'llab-quvvatlanmaydi — bu `Array.prototype.reduce`/`reduceRight` ustiga qurilgan userland abstraktsiya. Reduce har step natijasini accumulator'ga o'tkazadi, callback `(acc, fn) => fn(acc)` zanjirni quradi.

**TC39 Pipeline operator (`|>`)** — Hack-style proposal Stage 2 da (faol harakat sust, lekin spec ochiq). Sintaksis: `value |> fn1(%) |> fn2(%)` — bu expression-based, `%` placeholder joriy value'ni bildiradi. Unary function emas, istalgan expression ishlaydi: `5 |> double(%) |> Math.sqrt(%)`.

</details>

</details>

### 6. `debounce` funksiyasini implement qiling. [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Debounce — funksiyani **oxirgi chaqiruvdan N ms o'tgandan keyin** bajaradi. Agar vaqt tugashidan oldin yana chaqirilsa — taymer qaytadan boshlanadi.

```
Debounce (300ms):
Chaqiruvlar: ──x──x──x──x──────────|──> fn()
                                    ^ 300ms o'tdi, endi bajaring
```

**Use case:** Search input (foydalanuvchi yozishni to'xtatganda qidirish)

```javascript
function debounce(fn, delay) {
  let timerId;

  return function(...args) {
    // Oldingi taymer bo'lsa — bekor qil
    clearTimeout(timerId);

    // Yangi taymer o'rnat
    timerId = setTimeout(() => {
      fn.apply(this, args); // this va argumentlarni saqlab qo'yish
    }, delay);
  };
}

// Ishlatish
const search = debounce(function(query) {
  console.log(`API chaqirildi: ${query}`);
}, 300);

// Foydalanuvchi tez-tez yozadi:
search("J");     // ❌ bekor qilindi
search("Ja");    // ❌ bekor qilindi
search("Jav");   // ❌ bekor qilindi
search("Java");  // ✅ 300ms kutib — "API chaqirildi: Java"
```

**Kengaytirilgan versiya (leading + cancel):**

```javascript
function debounce(fn, delay, { leading = false } = {}) {
  let timerId;
  let isLeadingInvoked = false;

  function debounced(...args) {
    // Leading: birinchi chaqiruvda darhol bajar
    if (leading && !isLeadingInvoked) {
      fn.apply(this, args);
      isLeadingInvoked = true;
    }

    clearTimeout(timerId);
    timerId = setTimeout(() => {
      if (!leading) fn.apply(this, args);
      isLeadingInvoked = false;
    }, delay);
  }

  debounced.cancel = function() {
    clearTimeout(timerId);
    isLeadingInvoked = false;
  };

  return debounced;
}
```

</details>

### 7. `throttle` funksiyasini implement qiling. [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Throttle — funksiyani **har N ms da bir marta** bajaradi. Oraliqda qancha chaqirilishidan qat'i nazar, faqat bitta bajariladi.

```
Throttle (300ms):
Chaqiruvlar: ──x──x──x──x──x──x──x──x──>
Bajariladi:  ──x────────x────────x───────>
             ^          ^        ^
             Har 300ms da bir marta
```

**Use case:** Scroll event, mousemove, resize

```javascript
function throttle(fn, limit) {
  let lastCall = 0;

  return function(...args) {
    const now = Date.now();

    // Oxirgi chaqiruvdan beri yetarli vaqt o'tdimi?
    if (now - lastCall >= limit) {
      lastCall = now;
      return fn.apply(this, args);
    }
  };
}

// Ishlatish
const onScroll = throttle(function() {
  console.log("Scroll handled:", window.scrollY);
}, 200);

window.addEventListener("scroll", onScroll);
// 200ms da bir marta handler ishlaydi, qancha scroll bo'lishidan qat'i nazar
```

**Kengaytirilgan versiya (trailing call):**

```javascript
function throttle(fn, limit) {
  let lastCall = 0;
  let timerId = null;

  return function(...args) {
    const now = Date.now();
    const remaining = limit - (now - lastCall);

    if (remaining <= 0) {
      // Yetarli vaqt o'tdi — darhol bajaring
      if (timerId) {
        clearTimeout(timerId);
        timerId = null;
      }
      lastCall = now;
      fn.apply(this, args);
    } else if (!timerId) {
      // Trailing call — oxirgi chaqiruvni ham bajarish
      timerId = setTimeout(() => {
        lastCall = Date.now();
        timerId = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}
```

</details>

### 8. `memoize` funksiyasini implement qiling. [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Memoization — funksiya natijalarini **cache lash** orqali takroriy hisoblashlarni oldini olish. Pure function lar uchun ideal.

```javascript
function memoize(fn) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    // Cache da bor bo'lsa — qaytarish (hisoblash kerak emas)
    if (cache.has(key)) {
      console.log(`Cache hit: ${key}`);
      return cache.get(key);
    }

    // Yo'q bo'lsa — hisoblash va saqlash
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Ishlatish — qimmat hisoblash
const factorial = memoize(function fact(n) {
  if (n <= 1) return 1;
  return n * fact(n - 1); // ⚠️ `fact` — INNER name, memoized wrapper emas!
});

factorial(100); // Hisoblaydi va cache ga saqlaydi (faqat n=100 uchun)
factorial(100); // Cache hit — darhol qaytaradi
factorial(99);  // Cache da EMAS! Qayta hisoblaydi

// ⚠️ NIMA UCHUN `factorial(99)` cache'da yo'q?
// `fact(n - 1)` memoized `factorial` ni emas, INNER `fact` ni chaqiradi
// (named function expression ichki scope'da faqat shu nom ko'rinadi)
// → Recursive call'lar memoize wrapper'ni chetlab o'tadi
// → Faqat top-level chaqiruvlar (factorial(100), factorial(99)) cache'ga tushadi

// ✅ Barcha intermediate natijalarni cache'lash uchun — let-binding trick:
let memoFactorial;
memoFactorial = memoize(function (n) {
  if (n <= 1) return 1;
  return n * memoFactorial(n - 1); // ← memoized wrapper'ga murojaat
});

memoFactorial(100); // 100, 99, 98, ..., 1 hammasi cache'ga tushadi
memoFactorial(99);  // Cache hit — darhol qaytaradi
memoFactorial(50);  // Cache hit — chunki 100 hisoblanganda 50 ham saqlangan
```

**Kengaytirilgan versiya — maxSize (LRU eviction):**

```javascript
function memoize(fn, { maxSize = 100 } = {}) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      // LRU: olib qaytadan qo'yish (oxiriga ko'chirish)
      const val = cache.get(key);
      cache.delete(key);
      cache.set(key, val);
      return val;
    }

    const result = fn.apply(this, args);
    cache.set(key, result);

    // maxSize dan oshsa — eng eski (birinchi) ni o'chirish
    if (cache.size > maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    return result;
  };
}
```

⚠️ `JSON.stringify` — oddiy key uchun yetadi, lekin circular reference yoki function argument uchun muammo. Production da custom hash yoki `WeakMap` ishlatiladi.

</details>

### 9. Output nima bo'ladi? [Middle]

```javascript
function test(a = 1, b = a * 2, c = b + 1) {
  console.log(a, b, c);
}

test();              // ?
test(5);             // ?
test(5, undefined);  // ?
test(5, null);       // ?
test(5, 0);          // ?
```

<details>
<summary><strong>Javob</strong></summary>

- `test()` → `1 2 3` — hammasi default
- `test(5)` → `5 10 11` — `a=5`, `b=5*2=10`, `c=10+1=11`
- `test(5, undefined)` → `5 10 11` — `undefined` = default ishlatiladi
- `test(5, null)` → `5 null 1` — `null` **default emas!** `null + 1 = 1` (type coercion)
- `test(5, 0)` → `5 0 1` — `0` berildi, default emas. `c = 0 + 1 = 1`

**Muhim:** Default parameter faqat `undefined` da ishga tushadi. `null`, `0`, `""`, `false` — bularning barchasi "berilgan qiymat" hisoblanadi va default ishlamaydi.

</details>

### 10. JavaScript va Python default parameter — farqi nima? Quyidagi kodda bug qayerda? [Middle]

**Savol 1:** `addToList` Python da mashhur "mutable default" bug'iga uchraydi. JS da ham shundaymi?

```javascript
function addToList(item, list = []) {
  list.push(item);
  return list;
}

console.log(addToList("a")); // ?
console.log(addToList("b")); // ?
```

**Savol 2:** Quyidagi variantda haqiqiy bug bor. Aniqlang va tuzating:

```javascript
const defaultList = [];
function addToList2(item, list = defaultList) {
  list.push(item);
  return list;
}

console.log(addToList2("a")); // ?
console.log(addToList2("b")); // ?
```

<details>
<summary><strong>Javob</strong></summary>

**Javob 1 — `addToList`:** `["a"]` va `["b"]` — **bug yo'q, JS da to'g'ri ishlaydi**.

```javascript
addToList("a"); // ["a"]  — yangi array
addToList("b"); // ["b"]  — YANA yangi array
```

**Javob 2 — `addToList2`:** `["a"]` va `["a", "b"]` — **bu haqiqiy bug**.

```javascript
addToList2("a"); // ["a"]       — defaultList ga push
addToList2("b"); // ["a", "b"]  — ❌ oldingi "a" hali ham bor!
```

**Python vs JavaScript farqi:**

| | Python | JavaScript |
|---|--------|------------|
| Default expression | Funksiya **definition** vaqtida **bir marta** baholanadi | Har **chaqiruvda qayta baholanadi** |
| `def f(x=[])` | Shared mutable default — klassik tuzoq | ✅ Muammo yo'q, har chaqiruvda yangi `[]` |
| Tashqi reference | Shunga o'xshash tuzoq | `list = defaultList` bo'lsa shared — **bug** |

**Tuzatish (`addToList2` uchun):**

```javascript
// ✅ 1-usul — default'da inline expression ishlating
function addToList2(item, list = []) {
  list.push(item);
  return list;
}

// ✅ 2-usul — default'da copy yaratish (shared reference'dan himoya)
function addToList3(item, list) {
  const result = list ? [...list] : [];
  result.push(item);
  return result;
}
```

**Xulosa:** JS'da `function(list = [])` expression'i har chaqiruvda yangi `[]` yaratadi (Python'dan farqli). Tuzoq faqat default'da **tashqi o'zgaruvchi** ga reference qilingan bo'lsa paydo bo'ladi — shunda mutation shared bo'ladi.

</details>

### 11. Output nima bo'ladi? [Middle+]

```javascript
var x = 10;

function showValue() {
  console.log(x);      // ?
  var x = 20;
  console.log(x);      // ?
}

showValue();
console.log(x);        // ?
```

<details>
<summary><strong>Javob</strong></summary>

- Birinchi `console.log(x)` → `undefined` — `var x` hoist bo'ldi, lekin qiymati hali tayinlanmagan
- Ikkinchi `console.log(x)` → `20` — endi `x = 20` tayinlangan
- Uchinchi `console.log(x)` → `10` — global `x`, funksiya ichidagi `var x` faqat local scope da

**Tushuntirish:** `var x = 20` declaration hoist bo'ladi (`var x` yuqoriga ko'tariladi, `= 20` joyida qoladi). Shuning uchun funksiya ichida `x` **local** o'zgaruvchi — global `x` ga ta'sir qilmaydi. Bu "variable shadowing".

</details>

### 12. Output nima bo'ladi? [Senior]

```javascript
function greet() {
  console.log(this);
}

const announce = () => {
  console.log(this);
};

const obj = { greet, announce };

obj.greet();              // ?
obj.announce();           // ?
greet.call({ a: 1 });     // ?
announce.call({ a: 1 });  // ?
new greet();              // ?
// new announce();        // ?
```

<details>
<summary><strong>Javob</strong></summary>

- `obj.greet()` → `obj` ({greet: f, announce: f}) — implicit binding
- `obj.announce()` → `announce` definition vaqtidagi tashqi `this` (script'da `globalThis`, ES module'da `undefined`) — arrow function `this` ni **tashqi scope dan oladi**, implicit binding ishlamaydi
- `greet.call({ a: 1 })` → `{ a: 1 }` — explicit binding
- `announce.call({ a: 1 })` → definition vaqtidagi tashqi `this` (script'da `globalThis`, ES module'da `undefined`) — arrow function da `call` **this ni o'zgartirmaydi**
- `new greet()` → `greet {}` — yangi bo'sh object (new binding)
- `new announce()` → ❌ `TypeError: announce is not a constructor` — arrow function `[[Construct]]` slotiga ega emas

Arrow function uchun **hech qanday** binding rule ishlamaydi — u doim tashqi lexical scope ning `this` ini ishlatadi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Spec'dagi internal slot'lar:**

Arrow function `[[ThisMode]]: "lexical"` ichki slotiga ega. `[[Construct]]` internal method'i yo'q — shuning uchun `new` bilan chaqirilsa `TypeError: <name> is not a constructor` tashlaydi (`Construct` abstract operation `[[Construct]]` mavjudligini tekshiradi).

**`this` resolution:**

`[[Call]]` chaqirilganda `OrdinaryCallBindThis` qadami `[[ThisMode]] === "lexical"` bo'lsa skip qilinadi — funksiya scope'da yangi `this` binding yaratilmaydi. `this` ga murojaat `ResolveThisBinding` orqali `GetThisEnvironment()` tashqi Environment Record'gacha yuradi — birinchi `HasThisBinding()` `true` qaytarsa o'sha Environment'dan `[[ThisValue]]` olinadi.

**`arguments` object:**

Arrow function'da `arguments` object ham yaratilmaydi — `FunctionDeclarationInstantiation` da `[[ThisMode]] === "lexical"` bo'lsa `arguments` binding skip qilinadi. Shuning uchun arrow ichida `arguments` tashqi function'ning `arguments`'iga (yoki global scope'da `ReferenceError`'ga) ishora qiladi.

**`call`/`apply`/`bind` arrow'da:**

Method'lar arrow function'da chaqirilsa, birinchi argument (thisArg) ignore qilinadi — chunki `OrdinaryCallBindThis` skip qilinadi. Lekin `bind` partial application uchun ishlatilsa bo'ladi (qo'shimcha argumentlarni qotirish).

</details>

</details>

### 13. `pipe` funksiyasini yozing — har bir bosqichda log chiqarsin. [Senior]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function pipeWithLog(...fns) {
  return function(value) {
    return fns.reduce((acc, fn) => {
      const result = fn(acc);
      console.log(`${fn.name || "anonymous"}: ${JSON.stringify(acc)} → ${JSON.stringify(result)}`);
      return result;
    }, value);
  };
}

// Ishlatish
const double = x => x * 2;
const addTen = x => x + 10;
const square = x => x * x;

const transform = pipeWithLog(double, addTen, square);
transform(3);
// double: 3 → 6
// addTen: 6 → 16
// square: 16 → 256
// Natija: 256
```

**Async pipe (Promise support):**

```javascript
function asyncPipe(...fns) {
  return function(value) {
    return fns.reduce(
      (promise, fn) => promise.then(fn),
      Promise.resolve(value)
    );
  };
}

// Ishlatish
const fetchUser = async (id) => { /* API call */ };
const extractName = (user) => user.name;
const toUpperCase = (str) => str.toUpperCase();

const getUserName = asyncPipe(fetchUser, extractName, toUpperCase);
await getUserName(1); // "ALI"
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Sync vs async pipe internal:**

Sync `pipe` `Array.prototype.reduce` ustiga qurilgan — har step natijasi accumulator'ga o'tkaziladi. Async variant `Promise.resolve(initial)` bilan boshlanadi va har step `.then(fn)` orqali qo'shiladi — har fn promise yoki value qaytarishi mumkin (`.then` ikkalasini ham handle qiladi).

**Error handling async pipe'da:**

```javascript
function asyncPipe(...fns) {
  return (value) => fns.reduce(
    (promise, fn) => promise.then(fn).catch(err => {
      console.error(`Pipeline failed at ${fn.name}:`, err);
      throw err;
    }),
    Promise.resolve(value)
  );
}
```

**TC39 Pipeline operator (`|>`):**

Hack-style proposal Stage 2 da. Sintaksis: `value |> fn1(%) |> fn2(%)` — `%` placeholder joriy value'ni bildiradi. Faqat unary function emas, istalgan expression ishlaydi:

```javascript
// Hozir (userland)
const result = pipe(double, addTen, square)(3); // 256

// Pipeline operator bilan
const result = 3 |> double(%) |> addTen(%) |> square(%); // 256
```

Expression-based bo'lgani uchun `await`, ternary, method call ham ishlaydi: `data |> await fetch(%) |> (%).json()`.

</details>

</details>

### 14. Quyidagi kodni tuzating (Fix the bug). [Middle+]

```javascript
// ❌ Bug: once funksiyasi to'g'ri ishlamayapti
function once(fn) {
  let called = false;
  return function(...args) {
    if (!called) {
      called = true;
      fn(...args);
    }
  };
}

const pay = once(function(amount) {
  console.log(`To'lov: ${amount} so'm`);
  return amount;
});

const result = pay(50000);
console.log(result); // undefined ❌ — kutilgan: 50000
pay(100000);         // chaqirilmadi ✅ — bu to'g'ri
```

<details>
<summary><strong>Javob</strong></summary>

```javascript
// ✅ Tuzatilgan — return qo'shish kerak!
function once(fn) {
  let called = false;
  let result;
  return function(...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args); // this ni ham saqlash + return
      return result;
    }
    return result; // takroriy chaqiruvlarda oldingi natijani qaytarish
  };
}

const pay = once(function(amount) {
  console.log(`To'lov: ${amount} so'm`);
  return amount;
});

console.log(pay(50000));  // To'lov: 50000 so'm → 50000 ✅
console.log(pay(100000)); // → 50000 (oldingi natija qaytarildi, funksiya qayta chaqirilmadi) ✅
```

**Xato sababi:** Asl kodda `fn(...args)` natijasi `return` qilinmagan. Shuningdek, `this` konteksti `apply` orqali saqlanmagan.

</details>

### 15. Output nima bo'ladi? [Senior]

```javascript
function A() {
  this.x = 1;
  return { x: 2 };
}

function B() {
  this.x = 1;
  return 42;
}

function C() {
  this.x = 1;
  return null;
}

console.log(new A().x);  // ?
console.log(new B().x);  // ?
console.log(new C().x);  // ?

const D = () => { this.x = 1; };
console.log(new D().x);  // ?
```

<details>
<summary><strong>Javob</strong></summary>

- `new A().x` → `2` — constructor **object** qaytardi → shu object ishlatiladi, `this` tashlanadi
- `new B().x` → `1` — constructor **primitive** (`42`) qaytardi → ignore, `this` qaytariladi
- `new C().x` → `1` — `null` primitive hisoblanadi (garchi `typeof null === "object"`) → ignore, `this` qaytariladi
- `new D()` → ❌ `TypeError: D is not a constructor` — arrow function `[[Construct]]` yo'q

**Qoida:** `new` bilan chaqirilganda, agar constructor **non-null object** qaytarsa — o'sha object ishlatiladi. Aks holda `this` (yangi yaratilgan object) qaytariladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**`[[Construct]]` algoritmi spec'da:**

`OrdinaryCreateFromConstructor` birinchi yangi object yaratadi (`thisArgument`), prototype'ni constructor'ning `prototype` property'sidan oladi. Keyin constructor body `[[Call]]` orqali `this = thisArgument` bilan bajariladi (`OrdinaryCallBindThis` `this` ni yangi object'ga binding qiladi).

**Return qiymat tekshiruvi:**

Spec'da aniq qadam — `Construct` operation'i:
```
If result.[[Type]] is normal and Type(result.[[Value]]) is Object:
  Return result.[[Value]]
If result.[[Type]] is normal:
  Return thisArgument
```

`Type(null)` — spec'da `Null`, `Object` emas. Shuning uchun `return null` `this` ni saqlab qoladi. `Type(42)` — `Number`. `Type({})` — `Object`. `Type(function(){})` — `Object` (function spec'da object).

**Edge case — return function:**

```javascript
function F() {
  this.x = 1;
  return function() { return "from return"; };
}
const f = new F();
typeof f;  // "function"
f();       // "from return" — function ham Object hisoblanadi
```

**Arrow function va `[[Construct]]`:**

Arrow function spec'da `[[ConstructorKind]]` slot yo'q va `[[Construct]]` internal method o'rnatilmaydi. `Construct(F)` chaqirilganda spec birinchi `IsConstructor(F)` tekshiradi — `false` qaytsa `TypeError`. Class constructor'lari `[[ConstructorKind]] = "derived"` yoki `"base"` bo'ladi.

</details>

</details>

### 16. Output nima bo'ladi? [Senior]

```javascript
const obj = {
  count: 0,
  increment: function() {
    return () => {
      this.count++;
      return this.count;
    };
  },
  decrement: () => {
    return function() {
      this.count--;
      return this.count;
    };
  }
};

const inc = obj.increment();
console.log(inc());  // ?
console.log(inc());  // ?

const dec = obj.decrement();
console.log(dec());  // ?
```

<details>
<summary><strong>Javob</strong></summary>

- `inc()` → `1` ✅ — `increment` regular method (`this = obj`), ichidagi arrow function **lexical this** = `obj`. `obj.count` 0 dan 1 ga.
- `inc()` → `2` ✅ — yana `obj.count++`, 1 dan 2 ga.
- `dec()` → `NaN` ❌ — `decrement` **arrow function** (`this` = global/module, `obj` emas!). Ichidagi regular function **default binding** ishlatadi (`this` = global). `global.count` → `undefined`. `undefined - 1` → `NaN`.

**Tushuntirish:** `increment` da regular function + ichidagi arrow = **to'g'ri this zanjiri**. `decrement` da arrow function + ichidagi regular function = **ikkalasi ham this ni yo'qotadi**.

<details>
<summary><strong>Deep Dive</strong></summary>

**`[[ThisMode]]` internal slot:**

Spec'da har function'ning `[[ThisMode]]` slot'i 3 qiymatdan birini oladi:
- `"global"` — non-strict regular function (default — `this = undefined` bo'lsa global object'ga coerce qilinadi)
- `"strict"` — strict mode regular function (`this = undefined` saqlanadi, coerce qilinmaydi)
- `"lexical"` — arrow function (`this` resolve qilinmaydi, lexical scope'dan olinadi)

**`increment` ish zanjiri:**

1. `obj.increment()` — `obj` PropertyReference bo'lib `Call`'ga uzatiladi
2. `OrdinaryCallBindThis` `[[ThisMode]] !== "lexical"` — `this = obj` binding qiladi function Environment Record'da
3. Ichidagi arrow function chaqirilganda `[[ThisMode]] === "lexical"` — `OrdinaryCallBindThis` skip
4. Arrow ichida `this` — `ResolveThisBinding()` orqali tashqi (`increment`) Environment'dan `obj` olinadi

**`decrement` ish zanjiri:**

1. `decrement` arrow function — definition vaqtida lexical scope (module/global) `this` ni ushlaydi
2. Object literal'ning o'z `this` binding'i YO'Q — arrow tashqi scope'ga chiqadi
3. Module'da `this = undefined`, script'da `this = globalThis`
4. Ichidagi regular function `obj.decrement()()` — `()` chaqiruvi PropertyReference'siz, `this = undefined` (strict) yoki `globalThis` (non-strict)
5. `globalThis.count` — `undefined` → `undefined - 1 = NaN`

**Module vs script muhim:**

ES modules avtomatik strict — top-level `this = undefined`. `<script>` tag'da yoki Node.js CommonJS'da `this = globalThis`. Shuning uchun bir xil kod muhitga qarab `NaN` yoki `TypeError: Cannot read property 'count' of undefined` berishi mumkin.

</details>

</details>

### 17. Named Function Expression — ichki ism qayerda visible? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Named function expression (`const f = function innerName() {...}`)'da `innerName` **faqat function body ichida** mavjud — recursive self-reference uchun safe pattern.

```javascript
const factorial = function fact(n) {
  if (n <= 1) return 1;
  return n * fact(n - 1); // ✅ ichki "fact" — recursive call
};

console.log(factorial(5));   // 120
console.log(factorial.name); // "fact"

// Tashqi scope'da "fact" YO'Q:
// console.log(fact(5)); // ❌ ReferenceError

// Pattern afzalligi — tashqi reference o'zgarsa ham ishlaydi:
const fn = factorial;
// factorial = null; // bekor qilsak ham
// fn(5) → 120 ishlaydi, chunki ichki "fact" tashqi o'zgarmaydi

// Anonymous expression — tashqi reference ga bog'liq:
const broken = function(n) {
  if (n <= 1) return 1;
  return n * broken(n - 1); // ❌ tashqi "broken" ga bog'liq
};
```

**Spec — function expression scope:**

Spec'da Named Function Expression ichki nomni alohida **function expression scope** Environment Record'da bind qiladi — read-only binding, faqat function body ichida visible. Bu `FunctionExpression` runtime semantics'ida belgilangan. Use case'lar: recursive function expression, debugging-friendly stack trace (DevTools'da nom ko'rinadi), library'larda named pattern.

</details>

### 18. Default parameter scope — TDZ va parametr orasidagi murojaat [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Default parameter'lar o'z scope'iga ega — har parameter o'zidan **oldingi** parametrlarga murojaat qilishi mumkin, lekin **keyingi**larga TDZ tufayli kira olmaydi.

```javascript
// ✅ Oldingi parametrga murojaat — ishlaydi
function createElement(tag, className = tag + "-element") {
  return { tag, className };
}
createElement("div"); // { tag: "div", className: "div-element" }

// ❌ Keyingi parametrga murojaat — TDZ
function test(a = b, b = 1) {
  return [a, b];
}
test();  // ❌ ReferenceError: Cannot access 'b' before initialization
test(5); // [5, 1] ✅ — a berildi, b'ga murojaat yo'q

// Murakkab zanjir
function chain(a = 1, b = a * 2, c = a + b) {
  return [a, b, c];
}
chain();    // [1, 2, 3]
chain(5);   // [5, 10, 15]
chain(5,3); // [5, 3, 8]

// Required parameter trick
function required(name) {
  throw new Error(`"${name}" majburiy`);
}
function transfer(from = required("from"), amount = required("amount")) {
  return { from, amount };
}
transfer("Ali", 100); // ✅
// transfer("Ali"); // ❌ throw Error
```

**Spec — parameter scope va TDZ:**

Spec'da har parameter alohida **declarative Environment Record** binding bo'ladi — TDZ qoidalari amal qiladi. Default expression `IteratorBindingInitialization` algorithm'ida lazy evaluate qilinadi (faqat argument `undefined` bo'lsa). Bu Python'dan farqli — Python default function definition vaqtida bir marta evaluate bo'ladi (mutable default tuzog'i), JavaScript har chaqiruvda qayta evaluate qiladi.

</details>

### 19. Function `length` vs `arguments.length` — farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`function.length` — **e'lon qilingan** majburiy parametrlar soni (definition-time, default/rest dan oldingi). `arguments.length` — **chaqiruvda** berilgan argumentlar soni (call-time).

```javascript
function test(a, b, c) {
  return { declared: test.length, passed: arguments.length };
}
test(1, 2);       // { declared: 3, passed: 2 }
test(1, 2, 3, 4); // { declared: 3, passed: 4 }

// length default/rest dan oldingi parametrlarni hisoblaydi
function fn1(a, b, c) {}              // length: 3
function fn2(a, b, ...rest) {}        // length: 2 — rest hisoblanmaydi
function fn3(a, b = 1, c) {}          // length: 1 — b default, undan keyingi c ham hisoblanmaydi
function fn4(a = 1, b, c) {}          // length: 0 — birinchi default, keyingilar ham
function fn5(...args) {}              // length: 0

// bind length'ga ta'sir qiladi:
function add(a, b, c) { return a + b + c; }
console.log(add.length);              // 3
console.log(add.bind(null).length);   // 3
console.log(add.bind(null, 1).length); // 2 — bir partial arg berildi
console.log(add.bind(null, 1, 2).length); // 1
```

**Use case — curry'da:**

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) { // ← fn.length kerakli son
      return fn.apply(this, args);
    }
    return (...more) => curried(...args, ...more);
  };
}
```

**Spec — `length` property:**

Spec'da `length` property — `FunctionDeclarationInstantiation` vaqtida o'rnatiladi: `ExpectedArgumentCount` — non-rest, non-default parameter'lar soni. Birinchi default/rest position'dan keyin barchasi exclude. `arguments.length` esa `OrdinaryCallEvaluateBody` vaqtida actual passed argument count. `length` `writable: false, configurable: true` — `Object.defineProperty` orqali o'zgartirilishi mumkin (testing/mocking uchun).

</details>

---

*Savollar soni: 29 | Nazariy: 13 (~45%) | Output: 8 (~28%) | Coding: 6 (~21%) | Tuzatish: 2 (~7%)*
