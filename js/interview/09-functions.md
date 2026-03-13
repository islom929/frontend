# Functions — First-Class Citizens — Interview Savollari

> Function Declaration vs Expression vs Arrow, First-class functions, IIFE, HOF, Callbacks, Pure Functions, Currying, Partial Application, Composition, Memoization, Debounce, Throttle, arguments vs rest, default params, function name/length properties haqida interview savollari.
> **Savollar soni:** 25
> **Taqsimot:** Nazariy ~40% | Output ~30% | Coding ~20% | Tuzatish ~10%

---

## Savol 1: Function Declaration, Function Expression va Arrow Function — farqlari nima? [Junior+]

**Javob:**

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

**Deep Dive:**

Arrow function ichki `[[ThisMode]]: "lexical"` slotiga ega. Bu degani engine `this` ni resolve qilganda, arrow function ning o'zida `this` qidirmaydi — **tashqi scope** ga chiqadi. Shuning uchun `call`, `apply`, `bind` arrow function ning `this` ini o'zgartira olmaydi:

```javascript
const arrow = () => console.log(this);
arrow.call({ name: "Ali" }); // window/global — call e'tiborga olinmadi!
```

---

## Savol 2: Output nima bo'ladi? [Junior+]

```javascript
console.log(a());  // ?
console.log(b());  // ?
console.log(c());  // ?

function a() { return "A"; }
var b = function() { return "B"; };
const c = () => "C";
```

**Javob:**
- `a()` → `"A"` ✅ — Function Declaration to'liq hoist bo'ladi
- `b()` → ❌ `TypeError: b is not a function` — `var b` hoist bo'ladi lekin `undefined` qiymati bilan, funksiya tayinlanmagan
- `c()` ga yetib bormaydi — `b()` da xato bo'lganligi uchun dastur to'xtaydi. Agar alohida tekshirsak: ❌ `ReferenceError: Cannot access 'c' before initialization` — `const` TDZ da

🔍 **Tushuntirish:** Hoisting davomida `function a` to'liq ko'tariladi. `var b` faqat o'zgaruvchi sifatida `undefined` bilan ko'tariladi. `const c` esa TDZ (Temporal Dead Zone) da qoladi.

---

## Savol 3: First-class function nima degani? [Junior+]

**Javob:**

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

**Deep Dive:**

ECMAScript spec bo'yicha funksiya — **callable object**. `[[Call]]` internal method mavjudligi uni chaqiriladigan qiladi. Funksiya object bo'lganligi uchun unga property ham qo'shish mumkin:

```javascript
function validate(email) { return /@/.test(email); }
validate.errorMsg = "Email noto'g'ri"; // funksiyaga property qo'shish
console.log(validate.name);   // "validate"
console.log(validate.length); // 1 (parametr soni)
```

---

## Savol 4: IIFE nima? Nima uchun ishlatiladi? [Junior+]

**Javob:**

**IIFE** (Immediately Invoked Function Expression) — funksiya yaratilishi bilan darhol chaqiriladi. Scope izolyatsiyasi uchun ishlatiladi.

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

---

## Savol 5: Higher-Order Function (HOF) nima? Misol bering. [Junior+]

**Javob:**

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

---

## Savol 6: Callback nima? Callback hell nima? [Junior+]

**Javob:**

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

---

## Savol 7: Pure Function nima? Side Effect nima? [Middle]

**Javob:**

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

---

## Savol 8: Output nima bo'ladi? [Middle]

```javascript
function foo(a, b, c) {
  console.log(arguments.length);  // ?
  console.log(arguments[2]);      // ?

  arguments[0] = 100;
  console.log(a);                 // ?
}

foo(1, 2, 3);
```

**Javob:**
- `arguments.length` → `3` — 3 ta argument berilgan
- `arguments[2]` → `3` — uchinchi argument
- `a` → `100` ✅ — **non-strict mode** da `arguments` va named parameter lar **bog'langan** (linked). `arguments[0]` o'zgarganda `a` ham o'zgaradi.

🔍 **Muhim:** Strict mode da bu bog'lanish **buziladi**:

```javascript
"use strict";
function foo(a) {
  arguments[0] = 100;
  console.log(a); // 1 — bog'lanish yo'q!
}
foo(1);
```

Arrow function da `arguments` **umuman yo'q** — tashqi scope dan qidiradi.

---

## Savol 9: `arguments` vs Rest Parameters — farqi nima? [Middle]

**Javob:**

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

---

## Savol 10: Default Parameters qanday ishlaydi? [Middle]

**Javob:**

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

---

## Savol 11: Output nima bo'ladi? [Middle]

```javascript
const funcs = [];
for (var i = 0; i < 3; i++) {
  funcs.push(function() { return i; });
}
console.log(funcs[0]()); // ?
console.log(funcs[1]()); // ?
console.log(funcs[2]()); // ?
```

**Javob:** Hammasi `3` qaytaradi! ✅

🔍 **Tushuntirish:** `var` — function-scoped, bitta `i` bor. Loop tugaganda `i = 3`. Barcha 3 ta funksiya **bir xil** `i` ga closure qilgan — ularning hech biri o'z nusxasiga ega emas.

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

---

## Savol 12: Currying nima? Universal `curry` funksiyasini implement qiling. [Middle+]

**Javob:**

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

**Deep Dive:**

`fn.length` — funksiya kutayotgan parametrlar soni (rest va default parametrlar hisobga olinmaydi). Curry har safar argumentlar yetarli ekanini tekshiradi. Yetmasa — closure orqali avvalgi argumentlarni saqlab yangi funksiya qaytaradi. Bu **lazy evaluation** printsipi.

---

## Savol 13: Partial Application nima? Currying dan farqi? [Middle+]

**Javob:**

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

const double = partial(multiply, 2);
double(5); // 10
```

| | Currying | Partial Application |
|---|---------|---------------------|
| Argumentlar | Doim birma-bir | Bir nechtasini oldindan berish |
| Qaytaradi | Zanjir funksiyalar | Bitta yangi funksiya |
| `f(a,b,c)` | `f(a)(b)(c)` | `g = f(a, b); g(c)` |
| JavaScript da | Maxsus `curry` kerak | `bind` yoki wrapper |

---

## Savol 14: Function Composition nima? `pipe` va `compose` ni implement qiling. [Senior]

**Javob:**

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

**Deep Dive — amaliy misol:**

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

---

## Savol 15: `debounce` funksiyasini implement qiling. [Middle+]

**Javob:**

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

---

## Savol 16: `throttle` funksiyasini implement qiling. [Middle+]

**Javob:**

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

---

## Savol 17: `memoize` funksiyasini implement qiling. [Middle+]

**Javob:**

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
  return n * fact(n - 1);
});

factorial(100); // Hisoblaydi va cache ga saqlaydi
factorial(100); // Cache hit — darhol qaytaradi
factorial(99);  // Bu ham cache da! (100 hisoblanganda 99 ham hisoblangan)
```

**Deep Dive — maxSize bilan memoize:**

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

---

## Savol 18: Output nima bo'ladi? [Middle]

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

**Javob:**

- `test()` → `1 2 3` — hammasi default
- `test(5)` → `5 10 11` — `a=5`, `b=5*2=10`, `c=10+1=11`
- `test(5, undefined)` → `5 10 11` — `undefined` = default ishlatiladi
- `test(5, null)` → `5 null 1` — `null` **default emas!** `null + 1 = 1` (type coercion)
- `test(5, 0)` → `5 0 1` — `0` berildi, default emas. `c = 0 + 1 = 1`

🔍 **Muhim:** Default parameter faqat `undefined` da ishga tushadi. `null`, `0`, `""`, `false` — bularning barchasi "berilgan qiymat" hisoblanadi va default ishlamaydi.

---

## Savol 19: Quyidagi kodni tuzating (Fix the bug). [Middle]

```javascript
// ❌ Bug: har chaqiruvda oldingi natijalar qo'shiladi
function addToList(item, list = []) {
  list.push(item);
  return list;
}

console.log(addToList("a")); // ["a"]
console.log(addToList("b")); // ["b"] — kutilgan, lekin...
// Python da xatolik bo'lardi (mutable default), JS da bu TO'G'RI ishlaydi!

// ❌ HAQIQIY muammo — bu usulda:
const defaultList = [];
function addToList2(item, list = defaultList) {
  list.push(item);
  return list;
}

console.log(addToList2("a")); // ["a"]
console.log(addToList2("b")); // ["a", "b"] — ❌ oldingi "a" ham bor!
```

**Javob:**

```javascript
// ✅ Tuzatilgan — default parametrda tashqi reference ishlatmang
function addToList2(item, list) {
  // Har safar yangi array yaratish
  const result = list ? [...list] : [];
  result.push(item);
  return result;
}

// yoki sodda usul (JS da default = [] har safar YANGI array yaratadi):
function addToList3(item, list = []) {
  list.push(item);
  return list;
}
```

🔍 **Tushuntirish:** JavaScript da `function(list = [])` dagi default expression **har chaqiruvda qayta baholanadi** — ya'ni har safar yangi `[]` yaratiladi. Bu Python dan farq! Lekin agar tashqi o'zgaruvchiga reference qo'ysangiz (`list = defaultList`), u shared bo'ladi va mutation barcha chaqiruvlarga ta'sir qiladi.

---

## Savol 20: Output nima bo'ladi? [Middle+]

```javascript
var x = 10;

function foo() {
  console.log(x);      // ?
  var x = 20;
  console.log(x);      // ?
}

foo();
console.log(x);        // ?
```

**Javob:**

- Birinchi `console.log(x)` → `undefined` — `var x` hoist bo'ldi, lekin qiymati hali tayinlanmagan
- Ikkinchi `console.log(x)` → `20` — endi `x = 20` tayinlangan
- Uchinchi `console.log(x)` → `10` — global `x`, funksiya ichidagi `var x` faqat local scope da

🔍 **Tushuntirish:** `var x = 20` declaration hoist bo'ladi (`var x` yuqoriga ko'tariladi, `= 20` joyida qoladi). Shuning uchun funksiya ichida `x` **local** o'zgaruvchi — global `x` ga ta'sir qilmaydi. Bu "variable shadowing".

---

## Savol 21: Output nima bo'ladi? [Senior]

```javascript
function foo() {
  console.log(this);
}

const bar = () => {
  console.log(this);
};

const obj = { foo, bar };

obj.foo();           // ?
obj.bar();           // ?
foo.call({ a: 1 });  // ?
bar.call({ a: 1 });  // ?
new foo();           // ?
// new bar();        // ?
```

**Javob:**

- `obj.foo()` → `obj` ({foo: f, bar: f}) — implicit binding
- `obj.bar()` → `window`/`global` (yoki `{}` module da) — arrow function `this` ni **tashqi scope dan oladi**, implicit binding ishlamaydi
- `foo.call({ a: 1 })` → `{ a: 1 }` — explicit binding
- `bar.call({ a: 1 })` → `window`/`global` — arrow function da `call` **this ni o'zgartirmaydi**
- `new foo()` → `foo {}` — yangi bo'sh object (new binding)
- `new bar()` → ❌ `TypeError: bar is not a constructor` — arrow function `[[Construct]]` slotiga ega emas

🔍 Arrow function uchun **hech qanday** binding rule ishlamaydi — u doim tashqi lexical scope ning `this` ini ishlatadi.

---

## Savol 22: `pipe` funksiyasini yozing — har bir bosqichda log chiqarsin. [Senior]

**Javob:**

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

---

## Savol 23: Quyidagi kodni tuzating (Fix the bug). [Middle+]

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

**Javob:**

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

🔍 **Xato sababi:** Asl kodda `fn(...args)` natijasi `return` qilinmagan. Shuningdek, `this` konteksti `apply` orqali saqlanmagan.

---

## Savol 24: Output nima bo'ladi? [Senior]

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
// console.log(new D().x);  // ?
```

**Javob:**

- `new A().x` → `2` — constructor **object** qaytardi → shu object ishlatiladi, `this` tashlanadi
- `new B().x` → `1` — constructor **primitive** (`42`) qaytardi → ignore, `this` qaytariladi
- `new C().x` → `1` — `null` primitive hisoblanadi (garchi `typeof null === "object"`) → ignore, `this` qaytariladi
- `new D()` → ❌ `TypeError: D is not a constructor` — arrow function `[[Construct]]` yo'q

🔍 **Qoida:** `new` bilan chaqirilganda, agar constructor **non-null object** qaytarsa — o'sha object ishlatiladi. Aks holda `this` (yangi yaratilgan object) qaytariladi.

---

## Savol 25: Output nima bo'ladi? [Senior]

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

**Javob:**

- `inc()` → `1` ✅ — `increment` regular method (`this = obj`), ichidagi arrow function **lexical this** = `obj`. `obj.count` 0 dan 1 ga.
- `inc()` → `2` ✅ — yana `obj.count++`, 1 dan 2 ga.
- `dec()` → `NaN` ❌ — `decrement` **arrow function** (`this` = global/module, `obj` emas!). Ichidagi regular function **default binding** ishlatadi (`this` = global). `global.count` → `undefined`. `undefined - 1` → `NaN`.

🔍 **Tushuntirish:** `increment` da regular function + ichidagi arrow = **to'g'ri this zanjiri**. `decrement` da arrow function + ichidagi regular function = **ikkalasi ham this ni yo'qotadi**.

---

*Savollar soni: 25 | Nazariy: 10 (~40%) | Output: 8 (~32%) | Coding: 5 (~20%) | Tuzatish: 2 (~8%)*
