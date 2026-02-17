# Bo'lim 5: Closures — Chuqur Tushuncha

> Closure — funksiya o'zi yaratilgan muhitni "eslab qolishi". JavaScript ning eng kuchli va eng ko'p so'raladigan tushunchasi.

---

## Mundarija

- [Closure Nima?](#closure-nima)
- [Closure Qanday Hosil Bo'ladi](#closure-qanday-hosil-boladi)
- [Lexical Environment va [[Environment]]](#lexical-environment-va-environment)
- [Use Cases](#use-cases)
- [Memory va Closures](#memory-va-closures)
- [Klassik Loop + Closure Muammosi](#klassik-loop--closure-muammosi)
- [Performance Considerations](#performance-considerations)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Closure Nima?

### Nazariya

Closure — JavaScript'ning eng kuchli va eng muhim tushunchalaridan biri. Closure bu funksiya va u yaratilgan **Lexical Environment** ning birikmasidir. Funksiya tashqi scope'dan qaytarilgandan keyin ham, tashqi funksiya tugagandan keyin ham, ichki funksiya tashqi scope'dagi o'zgaruvchilarni **ko'ra oladi va ishlatishi mumkin**.

Nima uchun bu g'ayrioddiy? Oddiy holda funksiya tugaganda uning barcha local o'zgaruvchilari yo'qoladi — Execution Context stack'dan olib tashlanadi, o'zgaruvchilar garbage collection uchun tayyor bo'ladi. Lekin closure bor bo'lganda bu qoida buziladi: ichki funksiya tashqi o'zgaruvchiga **reference** saqlaydi va shu sababli Garbage Collector bu o'zgaruvchini tozalay olmaydi. Bu xuddi **quti ichidagi kalitli sandiq**ga o'xshaydi — quti (tashqi funksiya) ochildi va yopildi, lekin sandiq (closure) kalitni (tashqi o'zgaruvchilarni) ichida saqlab qoldi.

Closure'lar nima uchun muhim? Zamonaviy JavaScript'dagi deyarli barcha muhim patternlar closure'ga asoslanadi: React Hooks (useState, useEffect), data privacy (private o'zgaruvchilar), factory function'lar, memoization, event handler'lar, callback'lar — bularning barchasi closure mexanizmi orqali ishlaydi. Closure'siz JavaScript bugungi ko'rinishidagi kuchli til bo'la olmasdi.

```javascript
function outer() {
  let count = 0;

  function inner() {
    count++;
    console.log(count);
  }

  return inner;
}

const fn = outer(); // outer() tugadi, lekin...
fn(); // 1 — count hali tirik!
fn(); // 2 — count oshyapti!
fn(); // 3 — closure count ni "eslab qoldi"
```

`outer()` tugadi — uning Execution Context stack'dan olib tashlandi. Lekin `count` yo'qolmadi. Nima uchun? Chunki `inner` funksiya `count` ga **closure** hosil qildi.

### MDN Ta'rif

> A closure is the combination of a function bundled together with references to its surrounding state (the lexical environment).

Closure = Funksiya + Uning Lexical Environment ga reference.

### Har Bir Funksiya Closure mi?

Texnik jihatdan — **ha**. JavaScript da har bir funksiya o'zi yaratilgan scope'ni reference qiladi. Lekin amalda "closure" deyilganda, odatda **tashqi scope o'zgaruvchilarini ishlatadigan** va **tashqi funksiyadan qaytariladigan** funksiyalar nazarda tutiladi.

```javascript
// Texnik jihatdan closure, lekin amalda "closure" deyilmaydi:
function simple() {
  let x = 1;
  console.log(x); // x faqat o'z scope'ida — maxsus narsa yo'q
}

// Amaliy closure — tashqi scope o'zgaruvchisini "eslab qoldi":
function makeCounter() {
  let count = 0;          // tashqi scope
  return function() {     // ichki funksiya — tashqi scope'ni eslab qoladi
    return ++count;
  };
}
```

---

## Closure Qanday Hosil Bo'ladi

### Qadam-Baqadam

```javascript
function createGreeter(greeting) {
  // 1. createGreeter() chaqirildi
  // 2. Yangi Execution Context yaratildi
  // 3. LexicalEnvironment: { greeting: "Salom" }

  return function(name) {
    // 4. Bu funksiya yaratilganda, u JORIY
    //    LexicalEnvironment ga reference oladi
    // 5. Bu reference [[Environment]] internal slot da saqlanadi
    return `${greeting}, ${name}!`;
  };

  // 6. createGreeter() tugadi, EC stack dan chiqdi
  // 7. LEKIN LexicalEnvironment yo'qolmadi —
  //    qaytarilgan funksiya unga reference qilyapti
}

const sayHi = createGreeter("Salom");
// 8. sayHi ning [[Environment]] = { greeting: "Salom" }

sayHi("Ali");   // "Salom, Ali!"
sayHi("Vali");  // "Salom, Vali!"
// 9. greeting = "Salom" hali tirik — closure saqlab turibdi
```

### Vizualizatsiya

```
1. createGreeter("Salom") chaqirildi:

   Call Stack:
   ┌────────────────────────────┐
   │  createGreeter() EC        │
   │  LexicalEnv:               │
   │    greeting: "Salom"       │◄─── Bu environment
   └────────────────────────────┘

2. return function(name){...} — ichki funksiya yaratildi:

   ┌──────────────────────┐
   │  function(name){...} │
   │  [[Environment]]: ───────► { greeting: "Salom" }
   └──────────────────────┘

3. createGreeter() tugadi, EC stack dan chiqdi:

   Call Stack:
   ┌────────────────────────────┐
   │  Global EC                  │
   └────────────────────────────┘

   Lekin HEAP da hali tirik:
   ┌──────────────────────┐      ┌─────────────────────┐
   │  sayHi (function)    │─────►│ { greeting: "Salom" }│
   │  [[Environment]]     │      │ (LexicalEnvironment) │
   └──────────────────────┘      └─────────────────────┘
                                   ↑ GC buni yo'qotmaydi —
                                     sayHi reference qilyapti

4. sayHi("Ali") chaqirilganda:

   Call Stack:
   ┌────────────────────────────┐
   │  sayHi() EC                │
   │  LexicalEnv:               │
   │    name: "Ali"             │
   │    outer: ─────────────────────► { greeting: "Salom" }
   └────────────────────────────┘
   │  Global EC                  │
   └────────────────────────────┘

   greeting → o'z scope'da yo'q → outer → { greeting: "Salom" } → TOPILDI!
```

---

## Lexical Environment va [[Environment]]

### Under the Hood

Har bir funksiya yaratilganda, engine uning **`[[Environment]]`** internal slot'iga joriy LexicalEnvironment ga reference saqlaydi. Bu **spec level** tushuncha.

```javascript
function outer() {
  let x = 10;
  // Bu yerning LexicalEnvironment: { x: 10, outer: → global }

  function inner() {
    console.log(x);
  }
  // inner.[[Environment]] = { x: 10, outer: → global }
  // inner YARATILGAN paytdagi environment SAQLANADI

  return inner;
}

const fn = outer();
// outer() tugadi, lekin:
// fn.[[Environment]] = { x: 10, outer: → global }
// Bu environment GC tomonidan yo'qotilmaydi — fn reference qilyapti

fn(); // 10 — x hali tirik!
```

### Bir Nechta Closure — Bitta Environment

Bir xil tashqi funksiyadan qaytarilgan bir nechta ichki funksiya **bitta** LexicalEnvironment ni share qiladi:

```javascript
function createPair() {
  let value = 0;
  // LexicalEnvironment A: { value: 0 }

  return {
    increment() { value++; },      // [[Env]] → A
    decrement() { value--; },      // [[Env]] → A
    getValue()  { return value; }  // [[Env]] → A
  };
  // Uchalasi ham BITTA A ga reference qiladi
}

const pair = createPair();
pair.increment();
pair.increment();
pair.increment();
console.log(pair.getValue()); // 3

pair.decrement();
console.log(pair.getValue()); // 2
// Uchalasi bitta "value" ni ko'radi va o'zgartiradi
```

```
┌────────────────────┐
│ increment()        │──┐
├────────────────────┤  │     ┌──────────────────┐
│ decrement()        │──┼────►│ { value: 2 }     │ (SHARED)
├────────────────────┤  │     └──────────────────┘
│ getValue()         │──┘
└────────────────────┘
```

### Har Bir Chaqiruv — Yangi Environment

```javascript
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const counter1 = makeCounter(); // Environment A: { count: 0 }
const counter2 = makeCounter(); // Environment B: { count: 0 }

counter1(); // 1 — A: { count: 1 }
counter1(); // 2 — A: { count: 2 }
counter2(); // 1 — B: { count: 1 } — ALOHIDA!
counter1(); // 3 — A: { count: 3 }
```

```
counter1 ──► Environment A: { count: 3 }
counter2 ──► Environment B: { count: 1 }
             ALOHIDA environment'lar!
```

---

## Use Cases

### 1. Data Privacy / Encapsulation

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance; // private — tashqaridan kirish imkoni yo'q

  return {
    deposit(amount) {
      if (amount <= 0) throw new Error("Musbat son kiriting");
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount > balance) throw new Error("Mablag' yetarli emas");
      balance -= amount;
      return balance;
    },
    getBalance() {
      return balance;
    }
  };
}

const account = createBankAccount(1000);
account.deposit(500);    // 1500
account.withdraw(200);   // 1300
account.getBalance();    // 1300

// balance ga to'g'ridan-to'g'ri KIRISH imkoni YO'Q:
console.log(account.balance); // undefined — private!
```

### 2. Factory Functions

```javascript
function createMultiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);
const tenX = createMultiplier(10);

double(5);  // 10
triple(5);  // 15
tenX(5);    // 50

// Har bir funksiya o'z "factor" ini eslab qoldi
```

### 3. Partial Application

```javascript
function createAPIClient(baseURL) {
  return function(endpoint) {
    return function(params) {
      return fetch(`${baseURL}${endpoint}`, params);
    };
  };
}

const api = createAPIClient("https://api.example.com");
const getUsers = api("/users");
const getPosts = api("/posts");

getUsers({ method: "GET" });
getPosts({ method: "GET" });

// Yoki currying style:
// createAPIClient("https://api.example.com")("/users")({ method: "GET" })
```

### 4. Module Pattern (Pre-ES6)

```javascript
const Calculator = (function() {
  // Private
  let history = [];

  function addToHistory(operation) {
    history.push(operation);
  }

  // Public API
  return {
    add(a, b) {
      const result = a + b;
      addToHistory(`${a} + ${b} = ${result}`);
      return result;
    },
    subtract(a, b) {
      const result = a - b;
      addToHistory(`${a} - ${b} = ${result}`);
      return result;
    },
    getHistory() {
      return [...history]; // copy qaytaradi, original emas
    }
  };
})();

Calculator.add(2, 3);      // 5
Calculator.subtract(10, 4); // 6
Calculator.getHistory();    // ["2 + 3 = 5", "10 - 4 = 6"]
// Calculator.history → undefined (private!)
// Calculator.addToHistory → undefined (private!)
```

### 5. Event Handlers

```javascript
function setupButton(buttonId, message) {
  const button = document.getElementById(buttonId);
  let clickCount = 0;

  button.addEventListener("click", function() {
    // closure: message va clickCount ni eslab qoldi
    clickCount++;
    console.log(`${message} — ${clickCount} marta bosildi`);
  });
}

setupButton("btn1", "Birinchi tugma");
setupButton("btn2", "Ikkinchi tugma");
// Har bir button o'z message va clickCount iga ega
```

### 6. Memoization

```javascript
function memoize(fn) {
  const cache = {}; // closure orqali saqlanadi

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache[key] !== undefined) {
      console.log("Cache dan olindi");
      return cache[key];
    }

    console.log("Hisoblanmoqda...");
    const result = fn(...args);
    cache[key] = result;
    return result;
  };
}

const expensiveCalc = memoize(function(n) {
  // Og'ir hisob-kitob
  let sum = 0;
  for (let i = 0; i < n; i++) sum += i;
  return sum;
});

expensiveCalc(1000000); // "Hisoblanmoqda..." → 499999500000
expensiveCalc(1000000); // "Cache dan olindi" → 499999500000 (tez!)
```

---

## Memory va Closures

### Nazariya

Closure kuchli mexanizm, lekin u **bepul kelmaydi** — har bir closure xotira resursini sarflaydi. Closure tashqi scope'ning LexicalEnvironment'iga reference saqlaydi va bu degani Garbage Collector bu environment'dagi o'zgaruvchilarni **yo'qota olmaydi** — chunki ularga hali tirik reference bor.

Bu xotira muammosi kichik dasturlarda sezilmaydi, lekin katta ilovalar — masalan, real-time dashboard'lar, chat ilovalar, yoki uzoq vaqt ochiq turadigan SPA (Single Page Application) larda jiddiy muammoga aylanishi mumkin. Agar closure'lar to'g'ri boshqarilmasa, ular **memory leak** (xotira oqishi) ga sabab bo'ladi: tozalanishi kerak bo'lgan katta ma'lumotlar (massivlar, ob'ektlar, DOM elementlari) closure orqali hayotda qoladi va xotira asta-sekin to'lib boradi.

Shuning uchun closure bilan ishlashda **xotira bo'shatish** haqida ham o'ylash kerak. Agar closure'ga endi ehtiyoj qolmasa — uning reference'ini `null` ga o'zgartiring yoki event listener'ni `removeEventListener` bilan olib tashlang. Zamonaviy engine'lar (V8) aqlli — agar closure faqat tashqi scope'ning **ba'zi** o'zgaruvchilarini ishlatsa, faqat shu o'zgaruvchilar saqlanadi (butun environment emas). Lekin bu optimizatsiya har doim ishlamaydi, shuning uchun dasturchi o'zi ham ehtiyot bo'lishi kerak.

### GC Qachon Tozalaydi

```javascript
function outer() {
  let bigData = new Array(1000000).fill("data"); // katta array

  return function inner() {
    console.log(bigData.length);
  };
}

let fn = outer();
// bigData HEAP da tirik — fn closure orqali reference qilyapti

fn(); // 1000000

fn = null;
// Endi fn yo'q → inner ga reference yo'q → bigData ga ham reference yo'q
// GC bigData ni tozalaydi ✅
```

### GC Qachon Tozalamaydi — Memory Leak

```javascript
// ❌ Memory Leak — closure keraksiz ma'lumotni ushlab turadi
function leak() {
  let hugeData = new Array(1000000).fill("x");

  return {
    getData() { return hugeData; },      // hugeData kerak
    getName() { return "Islom"; }         // hugeData KERAK EMAS, lekin...
  };
  // Ikkalasi ham BITTA environment share qiladi
  // getName() ishlatilsa ham, hugeData tozalanmaydi
}

const obj = leak();
// Faqat getName() ishlatamiz, lekin hugeData xotirada qoladi
obj.getName();
```

### Yechim — Keraksiz Reference'ni Tozalash

```javascript
// ✅ To'g'ri — keraksiz ma'lumotni null qilish
function noLeak() {
  let hugeData = new Array(1000000).fill("x");
  const result = hugeData.length; // kerakli ma'lumotni oldindan olish

  hugeData = null; // ← reference yo'q qilish — GC tozalaydi

  return {
    getSize() { return result; },
    getName() { return "Islom"; }
  };
}
```

```javascript
// ✅ Yoki — alohida scope'larga ajratish
function noLeak2() {
  const getData = (function() {
    let hugeData = new Array(1000000).fill("x");
    return function() { return hugeData; };
  })();
  // hugeData faqat getData ning closure'ida

  const getName = function() { return "Islom"; };
  // getName hugeData ni ko'rmaydi — alohida scope

  return { getData, getName };
}
```

### V8 Optimization — Dead Variable Elimination

Zamonaviy V8 closure'da **ishlatilmagan** o'zgaruvchilarni tozalay oladi:

```javascript
function outer() {
  let used = "kerak";
  let unused = new Array(1000000); // ishlatilmaydi

  return function() {
    console.log(used);
    // unused bu yerda ISHLATILMAYDI
    // V8 buni aniqlab, unused ni GC ga berishi MUMKIN
  };
}
```

Lekin bu optimizatsiya **kafolatlangan emas** — engine ga bog'liq. `eval()` ishlatilsa, V8 bu optimizatsiyani qila **olmaydi** (chunki eval ichida nima bo'lishi noma'lum):

```javascript
function outer() {
  let secret = "maxfiy";
  let unused = "keraksiz";

  return function(code) {
    eval(code); // ❌ V8: "eval bor — hech narsani tozalay olmayman"
    // unused ham xotirada qoladi
  };
}

const fn = outer();
fn("console.log(secret)"); // "maxfiy" — eval orqali kirdi
fn("console.log(unused)"); // "keraksiz" — eval orqali kirdi
```

---

## Klassik Loop + Closure Muammosi

Bu JavaScript'ning eng mashhur interview savoli va eng ko'p uchraydigan bug'i.

### Muammo

```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(function() {
    console.log(i);
  }, i * 100);
}
// Kutilgan: 0, 1, 2, 3, 4
// Haqiqiy:  5, 5, 5, 5, 5
```

### Nima Uchun 5, 5, 5, 5, 5?

```
1. var i → BITTA o'zgaruvchi (function/global scope)
2. Loop 5 marta aylanadi: i = 0, 1, 2, 3, 4, keyin i = 5 (loop tugaydi)
3. setTimeout callback'lari KEYINROQ ishlaydi (event loop)
4. Callback ishlash vaqtida loop ALLAQACHON tugagan, i = 5
5. BARCHA 5 ta callback BITTA i = 5 ni ko'radi

┌──────────────┐
│  var i = 5   │ ← BITTA o'zgaruvchi
├──────────────┤
│  callback 0 ─┤──► console.log(i) → 5
│  callback 1 ─┤──► console.log(i) → 5
│  callback 2 ─┤──► console.log(i) → 5
│  callback 3 ─┤──► console.log(i) → 5
│  callback 4 ─┤──► console.log(i) → 5
└──────────────┘
  Hammasi BITTA i ga ishora
```

### Yechim 1: let (Zamonaviy, Eng Oson)

```javascript
for (let i = 0; i < 5; i++) {
  setTimeout(function() {
    console.log(i);
  }, i * 100);
}
// ✅ 0, 1, 2, 3, 4
```

```
let → har iteratsiyada YANGI block scope, YANGI i nusxasi

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  let i = 0   │  │  let i = 1   │  │  let i = 2   │  ...
├──────────────┤  ├──────────────┤  ├──────────────┤
│  callback 0 ─┤  │  callback 1 ─┤  │  callback 2 ─┤
│  → i = 0     │  │  → i = 1     │  │  → i = 2     │
└──────────────┘  └──────────────┘  └──────────────┘
  Har biri O'Z i nusxasiga ega
```

### Yechim 2: IIFE (Eski Usul)

```javascript
for (var i = 0; i < 5; i++) {
  (function(j) {
    // j — i ning NUSXASI (har iteratsiyada yangi scope)
    setTimeout(function() {
      console.log(j);
    }, j * 100);
  })(i); // i ni argument sifatida berish
}
// ✅ 0, 1, 2, 3, 4
```

IIFE har iteratsiyada **yangi function scope** yaratadi. `j` = `i` ning shu paytdagi qiymati.

### Yechim 3: setTimeout ning 3-argumenti

```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(function(j) {
    console.log(j);
  }, i * 100, i); // 3-argument → callback ga parameter sifatida beriladi
}
// ✅ 0, 1, 2, 3, 4
```

### Yechim 4: bind

```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(console.log.bind(null, i), i * 100);
}
// ✅ 0, 1, 2, 3, 4
```

---

## Performance Considerations

### Closure Overhead

Closure qo'shimcha xotira talab qiladi — tashqi environment reference'ini saqlaydi:

```javascript
// ❌ Har bir object uchun yangi closure
function createUser(name) {
  return {
    getName() { return name; },
    setName(n) { name = n; }
  };
}

// 10,000 ta user = 10,000 ta closure + 10,000 ta environment
const users = Array.from({ length: 10000 }, (_, i) => createUser(`User ${i}`));
```

```javascript
// ✅ Prototype/Class ishlatish — method'lar share qilinadi
class User {
  constructor(name) {
    this.name = name; // property — closure emas
  }
  getName() { return this.name; }
  setName(n) { this.name = n; }
}

// 10,000 ta user, lekin getName/setName BITTA method (prototype da)
const users = Array.from({ length: 10000 }, (_, i) => new User(`User ${i}`));
```

### Qachon Closure, Qachon Class?

| Holat | Yechim |
|-------|--------|
| Data privacy kerak | Closure ✅ |
| Factory pattern | Closure ✅ |
| Ko'p instance, kam method | Class ✅ (memory tejash) |
| Inheritance kerak | Class ✅ |
| Simple utility | Closure ✅ |
| Memoization, debounce | Closure ✅ |
| React component state | Closure ✅ (hooks) |

Amalda closure overhead **juda kichik** — premature optimization qilmang. Faqat **ming-minglab** instance yaratayotganda class/prototype afzal.

---

## Common Mistakes

### ❌ Xato 1: Closure reference saqlashini tushunmaslik

```javascript
function createFunctions() {
  let result = [];
  for (var i = 0; i < 3; i++) {
    result.push(function() { return i; });
  }
  return result;
}

const fns = createFunctions();
console.log(fns[0]()); // 3 — 0 emas!
console.log(fns[1]()); // 3 — 1 emas!
console.log(fns[2]()); // 3 — 2 emas!
```

### ✅ To'g'ri usul:

```javascript
function createFunctions() {
  let result = [];
  for (let i = 0; i < 3; i++) { // let → har iteratsiyada yangi scope
    result.push(function() { return i; });
  }
  return result;
}

const fns = createFunctions();
console.log(fns[0]()); // 0 ✅
console.log(fns[1]()); // 1 ✅
console.log(fns[2]()); // 2 ✅
```

**Nima uchun:** Closure **qiymatni** emas, **reference**'ni saqlaydi. `var i` = bitta reference, hammasi 3 ko'radi. `let i` = har iteratsiyada yangi → har bir closure o'z nusxasiga ega.

---

### ❌ Xato 2: Closure orqali memory leak

```javascript
function setupHandler() {
  let hugeData = fetchHugeDataset(); // 100MB data

  document.getElementById("btn").addEventListener("click", function() {
    console.log("bosildi"); // hugeData ISHLATILMAYDI, lekin closure da
  });
  // hugeData xotirada qoladi — button mavjud ekan
}
```

### ✅ To'g'ri usul:

```javascript
function setupHandler() {
  let hugeData = fetchHugeDataset();
  const processedResult = processData(hugeData);

  hugeData = null; // ← xotirani bo'shatish

  document.getElementById("btn").addEventListener("click", function() {
    console.log(processedResult); // faqat kerakli natija
  });
}
```

**Nima uchun:** Event listener closure orqali tashqi scope'dagi barcha o'zgaruvchilarni reference qiladi. Katta ma'lumotlarni `null` ga o'zgartirish GC ga tozalash imkonini beradi.

---

### ❌ Xato 3: Closure ni noto'g'ri ishlatish — stale closure

```javascript
// React da ko'p uchraydi:
function Timer() {
  let count = 0;

  function startTimer() {
    setInterval(function() {
      count++;
      console.log(count); // ✅ ishlaydi — to'g'ridan-to'g'ri count ga yozmoqda
    }, 1000);
  }

  function getCount() {
    return count; // ❌ Muammo: qachon chaqirilganiga qarab farqli qiymat
  }

  return { startTimer, getCount };
}
```

### ✅ To'g'ri tushunish:

```javascript
// Stale closure — funksiya eski qiymatga yopishib qolishi
function createLogger() {
  let message = "boshlang'ich";

  const log = () => console.log(message);
  // log closure yaratdi — message ga reference

  message = "o'zgargan"; // message o'zgardi

  return log;
}

createLogger()(); // "o'zgargan" ✅ — closure REFERENCE saqlaydi, qiymat emas
// Bu holda to'g'ri. Lekin async kontekstda muammo bo'lishi mumkin.
```

**Nima uchun:** Closure **reference** saqlaydi. O'zgaruvchi o'zgarsa, closure **yangi qiymatni** ko'radi. Bu ba'zan kutilmagan natija beradi (ayniqsa React hooks'da — "stale closure" muammosi).

---

### ❌ Xato 4: Closure ichida this ni yo'qotish

```javascript
const user = {
  name: "Ali",
  friends: ["Vali", "Sami"],

  showFriends() {
    this.friends.forEach(function(friend) {
      console.log(`${this.name} ning do'sti: ${friend}`);
      // ❌ this.name = undefined — this yo'qoldi!
    });
  }
};
user.showFriends();
```

### ✅ To'g'ri usul:

```javascript
const user = {
  name: "Ali",
  friends: ["Vali", "Sami"],

  showFriends() {
    // Arrow function — this ni closure orqali oladi (lexical this)
    this.friends.forEach((friend) => {
      console.log(`${this.name} ning do'sti: ${friend}`); // ✅ "Ali ning do'sti: ..."
    });
  }
};
user.showFriends();
```

**Nima uchun:** Oddiy function o'zining `this` ini oladi. Arrow function `this` siz — tashqi scope'dagi `this` ni closure orqali ishlatadi. To'liq [10-this-keyword.md](10-this-keyword.md) da.

---

### ❌ Xato 5: eval + closure = optimization killer

```javascript
// ❌ eval closure optimization ni buzadi
function outer() {
  let a = 1;
  let b = 2;
  let c = new Array(1000000); // katta

  return function(code) {
    return eval(code);
    // V8 HECH NARSANI tozalay olmaydi — eval ichida nima bo'lishi noma'lum
    // a, b, c — hammasi xotirada qoladi
  };
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ eval ishlatmang. Hech qachon. Deyarli hech qachon.
function outer() {
  let a = 1;
  let b = 2;

  return function() {
    return a + b;
    // V8: "faqat a va b ishlatiladi, c ni tozalay olaman" ✅
  };
}
```

**Nima uchun:** `eval` runtime da ixtiyoriy kod bajaradi — V8 qaysi o'zgaruvchilar kerak bo'lishini **oldindan bilmaydi** → barcha closure o'zgaruvchilarini saqlashga majbur.

---

## Amaliy Mashqlar

### Mashq 1: Counter (Oson)

**Savol:** Closure ishlatib, `increment`, `decrement`, `getCount` methodlari bor counter yarating. Tashqaridan count ga to'g'ridan-to'g'ri kirish imkoni bo'lmasin.

<details>
<summary>Javob</summary>

```javascript
function createCounter(initial = 0) {
  let count = initial;

  return {
    increment() { return ++count; },
    decrement() { return --count; },
    getCount()  { return count; },
    reset()     { count = initial; return count; }
  };
}

const counter = createCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.decrement(); // 11
counter.getCount();  // 11
counter.reset();     // 10

console.log(counter.count); // undefined — private!
```

**Tushuntirish:** `count` `createCounter` ning scope'ida — tashqaridan kirish mumkin emas. Faqat qaytarilgan methodlar orqali boshqarish mumkin.
</details>

---

### Mashq 2: once() Funksiyasi (O'rta)

**Savol:** `once(fn)` funksiyasini yozing — u `fn` ni faqat **birinchi** chaqiruvda bajarsin, keyingi chaqiruvlarda birinchi natijani qaytarsin.

```javascript
const initialize = once(() => {
  console.log("Initialized!");
  return 42;
});

initialize(); // "Initialized!" → 42
initialize(); // → 42 (lekin "Initialized!" chiqmaydi)
initialize(); // → 42
```

<details>
<summary>Javob</summary>

```javascript
function once(fn) {
  let called = false;
  let result;

  return function(...args) {
    if (!called) {
      called = true;
      result = fn(...args);
    }
    return result;
  };
}
```

**Tushuntirish:** `called` va `result` closure orqali saqlanadi. Birinchi chaqiruvda `called = true` va natija saqlanadi. Keyingi chaqiruvlarda `fn` chaqirilmaydi, saqlangan `result` qaytariladi.
</details>

---

### Mashq 3: Private Counter + History (O'rta)

**Savol:** Counter yarating: har bir o'zgarishni history ga saqlash, undo qilish imkoniyati.

<details>
<summary>Javob</summary>

```javascript
function createCounterWithHistory(initial = 0) {
  let count = initial;
  let history = [initial];

  return {
    increment() {
      count++;
      history.push(count);
      return count;
    },
    decrement() {
      count--;
      history.push(count);
      return count;
    },
    undo() {
      if (history.length <= 1) return count;
      history.pop();
      count = history[history.length - 1];
      return count;
    },
    getCount()   { return count; },
    getHistory() { return [...history]; } // copy
  };
}

const c = createCounterWithHistory();
c.increment(); // 1
c.increment(); // 2
c.increment(); // 3
c.undo();      // 2
c.undo();      // 1
c.getHistory(); // [0, 1]
```

**Tushuntirish:** `count` va `history` ikkalasi ham private. `getHistory()` copy qaytaradi — tashqaridan original array ni o'zgartirib bo'lmaydi.
</details>

---

### Mashq 4: Debounce Implement Qilish (Qiyin)

**Savol:** `debounce(fn, delay)` funksiyasini yozing — oxirgi chaqiruvdan `delay` ms o'tgandan keyin `fn` ni bajarsin.

<details>
<summary>Javob</summary>

```javascript
function debounce(fn, delay) {
  let timerId = null; // closure — timer ID saqlaydi

  return function(...args) {
    clearTimeout(timerId); // oldingi timer bekor

    timerId = setTimeout(() => {
      fn.apply(this, args); // delay dan keyin bajar
    }, delay);
  };
}

// Ishlatish:
const search = debounce(function(query) {
  console.log("Qidirilmoqda:", query);
  // API call...
}, 300);

// Tez-tez chaqirilsa ham, faqat oxirgi chaqiruvdan 300ms keyin ishlaydi:
search("j");
search("ja");
search("jav");
search("java");
search("javas");
search("javascript");
// Faqat "javascript" uchun ishlaydi (300ms keyin)
```

**Tushuntirish:**
- `timerId` closure da saqlanadi — chaqiruvlar orasida o'zgarmaydi
- Har yangi chaqiruvda oldingi timer `clearTimeout` bilan bekor qilinadi
- Yangi timer o'rnatiladi — faqat oxirgisi ishlaydi
- `fn.apply(this, args)` — original `this` va argumentlarni saqlaydi
</details>

---

### Mashq 5: Curry Funksiyasi (Qiyin)

**Savol:** Universal `curry` funksiyasi yozing:

```javascript
function add(a, b, c) { return a + b + c; }
const curriedAdd = curry(add);

curriedAdd(1)(2)(3);   // 6
curriedAdd(1, 2)(3);   // 6
curriedAdd(1)(2, 3);   // 6
curriedAdd(1, 2, 3);   // 6
```

<details>
<summary>Javob</summary>

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }

    return function(...nextArgs) {
      return curried.apply(this, [...args, ...nextArgs]);
    };
    // ↑ closure: args saqlanadi, keyingi chaqiruvda birlashtriladi
  };
}

// Test:
function add(a, b, c) { return a + b + c; }
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3));   // 6
console.log(curriedAdd(1, 2)(3));   // 6
console.log(curriedAdd(1)(2, 3));   // 6
console.log(curriedAdd(1, 2, 3));   // 6
```

**Tushuntirish:**
- `fn.length` — original funksiya nechta argument kutadi
- Agar berilgan argumentlar yetarli → `fn` ni chaqir
- Agar kam → yangi funksiya qaytar, oldingi args ni closure da saqla
- Har bir qadam args yig'adi — `[...args, ...nextArgs]`
- Bu recursive closure — har bir qadam yangi closure yaratadi
</details>

---

## Xulosa

1. **Closure = Funksiya + Lexical Environment reference.** Funksiya o'zi yaratilgan muhitni "eslab qoladi".

2. **`[[Environment]]` internal slot** — har bir funksiyada bor. Yaratilgan paytdagi LexicalEnvironment ga reference.

3. **Closure reference saqlaydi, qiymat emas.** Tashqi o'zgaruvchi o'zgarsa — closure yangi qiymatni ko'radi.

4. **Bir nechta closure bitta environment share qiladi** — bitta tashqi funksiyadan qaytarilgan barcha ichki funksiyalar.

5. **Har bir tashqi funksiya chaqiruvi yangi environment yaratadi** — `makeCounter()` 2 marta chaqirilsa, 2 ta alohida closure.

6. **Use cases:** Data privacy, factory, partial application, module pattern, memoization, debounce/throttle, event handlers.

7. **Memory:** Closure tashqi environment'ni GC dan himoyalaydi. Keraksiz reference'larni `null` qiling. `eval` optimization ni buzadi.

8. **Loop + var muammosi** — classic interview savol. Yechim: `let` (yangi scope), IIFE, yoki bind.

---

> **Keyingi bo'lim:** [06-objects.md](06-objects.md) — Objects ichidan — property descriptors, getters/setters, freeze/seal, shallow vs deep copy.
