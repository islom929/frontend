# Bo'lim 5: Closures — Chuqur Tushuncha

> Closure — funksiya va uning yaratilgan paytdagi lexical environment ning birikmasi. Funksiya tashqi scope o'zgaruvchilariga reference saqlab, tashqi funksiya tugagandan keyin ham ularga murojaat qila oladi.

---

## Mundarija

- [Closure Nima](#closure-nima)
- [Lexical Environment va Closure Aloqasi](#lexical-environment-va-closure-aloqasi)
- [Closure Qanday Hosil Bo'ladi](#closure-qanday-hosil-boladi)
- [V8 da Closure — Under the Hood](#v8-da-closure--under-the-hood)
- [Use Cases](#use-cases)
  - [Data Privacy / Encapsulation](#data-privacy--encapsulation)
  - [Factory Functions](#factory-functions)
  - [Partial Application](#partial-application)
  - [Module Pattern](#module-pattern)
  - [Event Handlers](#event-handlers)
  - [Memoization](#memoization)
  - [Iterators (Stateful)](#iterators-stateful)
- [Memory va Closures](#memory-va-closures)
- [Memory Leaks va Oldini Olish](#memory-leaks-va-oldini-olish)
- [Klassik Loop + Closure Muammosi](#klassik-loop--closure-muammosi)
- [Performance Considerations](#performance-considerations)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Closure Nima

### Nazariya

Closure — funksiyaning o'zi **yaratilgan** (define qilingan) paytdagi lexical environment'ga **reference** saqlab qolishi. Natijada funksiya tashqi scope'dagi o'zgaruvchilarga murojaat qila oladi — hatto tashqi funksiya **tugab**, call stack'dan **chiqib ketgan** bo'lsa ham.

Closure ikki qismdan iborat:
1. **Funksiya** — ichki funksiya (yoki istalgan funksiya)
2. **Lexical Environment** — shu funksiya yaratilgan paytdagi scope'dagi barcha o'zgaruvchilar

Closure hosil bo'lishining **zaruriy sharti** — funksiya tashqi scope'dagi kamida bitta o'zgaruvchiga murojaat qilishi. Agar funksiya faqat o'z local o'zgaruvchilarini ishlatsa — texnik jihatdan closure mavjud, lekin amaliy ta'siri yo'q (engine optimizatsiya qilib, keraksiz reference'larni olib tashlashi mumkin).

Closure'ning amaliy ta'siri shundaki, funksiya **o'zi bilan birga** tashqi scope ma'lumotlarini olib yuradi. Bu ma'lumotlar boshqa hech kimga ko'rinmaydi — faqat shu funksiya (va uning "aka-uka" funksiyalari — bir xil tashqi scope'da yaratilganlar) murojaat qila oladi.

### Kod Misollari

Eng oddiy closure — ichki funksiya tashqi funksiyaning o'zgaruvchisiga murojaat qiladi:

```javascript
function createGreeting(greeting) {
  // greeting — createGreeting scope'da

  return function (name) {
    // Bu funksiya greeting ga closure hosil qildi
    // createGreeting tugagandan keyin ham greeting accessible
    return `${greeting}, ${name}!`;
  };
}

const hello = createGreeting("Hello");
const salom = createGreeting("Salom");

// createGreeting() allaqachon tugagan — call stack'dan chiqqan
// Lekin greeting qiymati hali tirik — closure uni ushlab turadi

console.log(hello("Alice"));  // "Hello, Alice!"
console.log(salom("Bob"));    // "Salom, Bob!"
// ✅ Har bir chaqiruvda alohida closure — alohida greeting qiymati
```

---

## Lexical Environment va Closure Aloqasi

### Nazariya

Closure'ni tushunish uchun [04-scope.md](04-scope.md) da o'rganilgan **Lexical Environment** tushunchasini chuqurlashtirish kerak.

Har bir Execution Context ichida **LexicalEnvironment** component bor. Bu component ikki qismdan iborat:
1. **Environment Record** — joriy scope'dagi barcha binding'lar (o'zgaruvchi-qiymat juftliklari)
2. **`[[OuterEnv]]`** — tashqi scope'ning LexicalEnvironment'iga reference

Har bir funksiya yaratilganda (define qilinganda) engine uning **`[[Environment]]`** internal slot'iga **joriy LexicalEnvironment** ni saqlaydi. Bu slot funksiyaning "tug'ilgan joyi" — uning lexical scope'ining snapshot'i.

Closure bu mexanizmning **oqibati**:
- Funksiya `[[Environment]]` orqali tashqi scope'ga reference saqlaydi
- Tashqi funksiya tugasa ham, agar ichki funksiya hali mavjud bo'lsa — tashqi scope'ning Environment Record'i **GC tomonidan tozalanmaydi** (chunki reference bor)
- Natija: ichki funksiya tashqi scope o'zgaruvchilariga murojaat qila oladi

Bu **reference** saqlanadi, **copy** emas. Ya'ni closure tashqi o'zgaruvchining **joriy qiymatini** ko'radi — closure yaratilgan paytdagi qiymatini emas.

### Under the Hood

```
function outer() {
  let count = 0;

  function increment() {
    count++;
    return count;
  }

  return increment;
}

const inc = outer();

─── outer() CHAQIRILGANDA ───

1. Yangi Execution Context yaratiladi:
   outer EC:
     LexicalEnvironment:
       Environment Record: { count: 0, increment: function }
       [[OuterEnv]]: Global Environment

2. increment funksiyasi YARATILADI:
   increment.[[Environment]] = outer EC ning LexicalEnvironment
   // ✅ Bu reference — outer scope'ga "ip bog'landi"

3. increment RETURN qilinadi → inc = increment

─── outer() TUGADI ───

4. outer EC call stack'dan CHIQDI
5. LEKIN: outer ning Environment Record yo'qolMADI!
   Sabab: inc (increment) hali unga reference ushlab turadi
   → GC buni tozalay olmaydi

─── inc() CHAQIRILGANDA ───

6. Yangi Execution Context yaratiladi:
   inc EC:
     LexicalEnvironment:
       Environment Record: { } (local o'zgaruvchi yo'q)
       [[OuterEnv]]: increment.[[Environment]]
                     = outer ning LexicalEnvironment
                       { count: 0 }

7. count++ bajariladi:
   count → inc scope'da yo'q
         → [[OuterEnv]] → outer scope → count = 0 → count = 1
   ✅ Tashqi scope'dagi count o'zgartirildi (reference, copy emas)
```

### Kod Misollari

Closure reference saqlashini ko'rsatuvchi misol:

```javascript
function createCounter() {
  let count = 0;

  return {
    increment() { return ++count; },
    decrement() { return --count; },
    getCount()  { return count; }
  };
}

const counter = createCounter();

console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.decrement()); // 1
console.log(counter.getCount());  // 1

// ✅ increment, decrement, getCount — barchasi BIR XIL count ga
//    closure hosil qilgan. Ular "aka-uka" — bir scope'da yaratilgan.
//    count o'zgartirsa — hammasi yangi qiymatni ko'radi (reference)
```

---

## Closure Qanday Hosil Bo'ladi

### Nazariya

Closure hosil bo'lish jarayoni qadam-baqadam:

**1-qadam: Tashqi funksiya chaqiriladi**
Engine yangi Execution Context va LexicalEnvironment yaratadi. Bu environment'da tashqi funksiyaning barcha local o'zgaruvchilari saqlanadi.

**2-qadam: Ichki funksiya define qilinadi**
Tashqi funksiya tanasi bajarilayotganda ichki funksiya yaratiladi. Engine ichki funksiyaning `[[Environment]]` slot'iga joriy (tashqi funksiyaning) LexicalEnvironment'ni yozadi.

**3-qadam: Ichki funksiya tashqariga "chiqadi"**
Ichki funksiya return qilinadi, yoki tashqi o'zgaruvchiga assign bo'ladi, yoki callback sifatida beriladi — qanday bo'lmasin, u tashqi funksiya scope'idan tashqarida ham mavjud bo'lib qoladi.

**4-qadam: Tashqi funksiya tugaydi**
Tashqi funksiyaning Execution Context call stack'dan chiqadi. Lekin uning Environment Record'i **yo'qolmaydi** — ichki funksiya hali reference ushlab turadi.

**5-qadam: Ichki funksiya chaqiriladi**
Ichki funksiyaning yangi EC yaratilganda, uning `[[OuterEnv]]` reference'i `[[Environment]]` slot'idagi saqlangan environment'ga ulanadi. Natija: tashqi o'zgaruvchilarga kirish mumkin.

### Kod Misollari

Har bir qadam vizual ko'rsatilgan misol:

```javascript
function makeAdder(x) {
  // Qadam 1: makeAdder chaqirildi
  // Environment Record: { x: 5 } (masalan x = 5)

  const adder = function (y) {
    // Qadam 2: adder funksiyasi yaratildi
    // adder.[[Environment]] = makeAdder Environment { x: 5 }
    return x + y;
  };

  // Qadam 3: adder return qilinadi
  return adder;
}
// Qadam 4: makeAdder tugadi — lekin Environment Record tirik

const add5 = makeAdder(5);
const add10 = makeAdder(10);
// Har bir chaqiruv ALOHIDA Environment yaratdi:
// add5.[[Environment]] → { x: 5 }
// add10.[[Environment]] → { x: 10 }

// Qadam 5: Chaqirish
console.log(add5(3));   // 8  — x=5, y=3
console.log(add10(3));  // 13 — x=10, y=3
// ✅ Har bir closure o'z environment'ini ko'radi
```

Closure har doim ichki funksiya bo'lishi shart emas — **istalgan funksiya** tashqi scope'ga reference saqlasa, closure hosil bo'ladi:

```javascript
let globalHandler;

function setup() {
  const config = { debug: true, version: "2.0" };

  globalHandler = function () {
    // Bu funksiya setup() scope'idagi config ga closure hosil qildi
    return config;
  };
}

setup();
// setup() tugadi, lekin config tirik — globalHandler ushlab turadi
console.log(globalHandler()); // { debug: true, version: "2.0" }
```

---

## V8 da Closure — Under the Hood

### Nazariya

V8 engine closure'larni qanday implement qiladi va optimizatsiya qiladi — bu bilim production'da performance-critical kod yozishda foydali.

**Context Object**: V8 da closure hosil bo'lganda, tashqi scope'dagi **faqat closure tomonidan ishlatiladigan** o'zgaruvchilar maxsus **Context** object'iga ko'chiriladi. Bu object heap'da saqlanadi (stack'da emas). Closure ishlatiLMAYDIGAN o'zgaruvchilar Context'ga tushMAYDI — ular stack frame bilan birga yo'qoladi.

```
function outer() {
  const used = "saqlanadi";      // ✅ closure ishlatadi → Context'ga tushadi
  const unused = "yo'qoladi";    // ❌ closure ishlatmaydi → stack'da qoladi
  let alsoUsed = 0;              // ✅ closure ishlatadi → Context'ga tushadi

  return function inner() {
    console.log(used);
    alsoUsed++;
  };
}

V8 ichida:

Stack frame (outer):
  ├── unused: "yo'qoladi"   ← stack'da, outer tugaganda yo'qoladi
  └── [Context pointer] ──→ Heap:
                              Context Object {
                                used: "saqlanadi"
                                alsoUsed: 0
                              }

inner.[[Environment]] → Context Object (heap'da)
```

**Optimizatsiya**: V8 parse-time da scope analysis qiladi — qaysi o'zgaruvchilar closure tomonidan ishlatilishini aniqlaydi. Faqat kerakli o'zgaruvchilar Context'ga ko'chiriladi. Bu "context allocation" deyiladi. Agar `eval()` ishlatilsa — V8 qaysi o'zgaruvchilar kerak bo'lishini aniqlay olmaydi va **barcha** o'zgaruvchilarni Context'ga ko'chiradi. Shu sababli `eval()` closure performance'ga salbiy ta'sir qiladi.

### Kod Misollari

V8 optimizatsiyasini ko'rsatuvchi misol:

```javascript
function createProcessor() {
  const smallConfig = { timeout: 5000 };           // closure ishlatadi
  const hugeData = new Array(1_000_000).fill(0);   // closure ishlatMAYDI

  return function process(input) {
    // faqat smallConfig ishlatiladi
    return { ...input, timeout: smallConfig.timeout };
    // ✅ V8: faqat smallConfig Context'ga tushadi
    // ✅ hugeData stack'da qoladi → outer tugaganda GC tozalaydi
  };
}

const process = createProcessor();
// hugeData allaqachon GC tomonidan tozalangan (closure ishlatmagan)
// faqat smallConfig heap'da tirik — process closure ushlab turadi
```

`eval` ning ta'siri:

```javascript
function createBad() {
  const small = "kerak";
  const huge = new Array(1_000_000).fill(0); // ❌ eval tufayli saqlanib qoladi

  return function () {
    eval(""); // ❌ eval bor — V8 barcha o'zgaruvchilarni Context'ga ko'chiradi
    return small;
  };
}
// ❌ huge ham Context'da saqlanadi — eval qaysi o'zgaruvchini
//    ishlatishini V8 oldindan bilmaydi
```

---

## Use Cases

### Data Privacy / Encapsulation

Closure'ning eng asosiy use case'i — ma'lumotni tashqi dunyodan **yashirish**. JavaScript da ES2022 gacha (private fields `#`) class'larda haqiqiy private data yo'q edi. Closure yagona ishonchli encapsulation usuli bo'lgan.

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;
  const transactions = [];

  function recordTransaction(type, amount) {
    transactions.push({
      type,
      amount,
      balance,
      timestamp: Date.now()
    });
  }

  return {
    deposit(amount) {
      if (amount <= 0) throw new Error("Amount must be positive");
      balance += amount;
      recordTransaction("deposit", amount);
      return balance;
    },

    withdraw(amount) {
      if (amount <= 0) throw new Error("Amount must be positive");
      if (amount > balance) throw new Error("Insufficient funds");
      balance -= amount;
      recordTransaction("withdrawal", amount);
      return balance;
    },

    getBalance() {
      return balance;
    },

    getTransactions() {
      return [...transactions]; // copy qaytaradi — original himoyalangan
    }
  };
}

const account = createBankAccount(1000);
account.deposit(500);    // 1500
account.withdraw(200);   // 1300
console.log(account.getBalance()); // 1300

// balance va transactions tashqaridan accessible EMAS:
console.log(account.balance);       // undefined
console.log(account.transactions);  // undefined
// ✅ Faqat public method'lar orqali kirish mumkin
```

---

### Factory Functions

Factory function — har chaqiruvda yangi object (yoki funksiya) yaratuvchi funksiya. Har bir yaratilgan object o'z closure scope'iga ega — biri ikkinchisiga ta'sir qilmaydi.

```javascript
function createLogger(prefix, level = "info") {
  let logCount = 0;

  return {
    log(message) {
      logCount++;
      console.log(`[${prefix}] [${level}] #${logCount}: ${message}`);
    },
    warn(message) {
      logCount++;
      console.warn(`[${prefix}] [WARN] #${logCount}: ${message}`);
    },
    getCount() {
      return logCount;
    }
  };
}

const dbLogger = createLogger("DB", "debug");
const apiLogger = createLogger("API");

dbLogger.log("Connected");       // [DB] [debug] #1: Connected
dbLogger.log("Query executed");  // [DB] [debug] #2: Query executed
apiLogger.log("Request received"); // [API] [info] #1: Request received

console.log(dbLogger.getCount());  // 2
console.log(apiLogger.getCount()); // 1
// ✅ Har bir logger o'z logCount va prefix ga ega — alohida closure
```

---

### Partial Application

Partial application — funksiyaning ba'zi argumentlarini oldindan "qotirish" va qolganlarini keyinroq qabul qiluvchi yangi funksiya yaratish.

```javascript
function request(baseURL, method, endpoint, data) {
  return fetch(`${baseURL}${endpoint}`, {
    method,
    headers: { "Content-Type": "application/json" },
    body: data ? JSON.stringify(data) : undefined
  });
}

// baseURL ni qotirish — partial application:
function createAPI(baseURL) {
  return {
    get(endpoint) {
      return request(baseURL, "GET", endpoint);
      // ✅ baseURL closure orqali saqlanadi
    },
    post(endpoint, data) {
      return request(baseURL, "POST", endpoint, data);
    },
    put(endpoint, data) {
      return request(baseURL, "PUT", endpoint, data);
    },
    delete(endpoint) {
      return request(baseURL, "DELETE", endpoint);
    }
  };
}

const api = createAPI("https://api.example.com");
// Endi har safar baseURL yozish shart emas:
api.get("/users");
api.post("/users", { name: "Alice" });
```

---

### Module Pattern

ES6 modullardan oldin IIFE + closure module pattern sifatida ishlatilgan — private state + public API:

```javascript
const UserModule = (function () {
  // Private — tashqaridan accessible emas
  const users = [];
  let nextId = 1;

  function validateEmail(email) {
    return email.includes("@");
  }

  // Public API — return qilingan object
  return {
    addUser(name, email) {
      if (!validateEmail(email)) {
        throw new Error("Invalid email");
      }
      const user = { id: nextId++, name, email };
      users.push(user);
      return user;
    },

    getUser(id) {
      return users.find(u => u.id === id);
    },

    getUserCount() {
      return users.length;
    }
  };
})();
// IIFE — darhol bajariladi, natija UserModule ga assign

UserModule.addUser("Alice", "alice@mail.com");
console.log(UserModule.getUserCount()); // 1
// console.log(UserModule.users);       // undefined — private
// console.log(UserModule.validateEmail); // undefined — private
```

---

### Event Handlers

DOM event handler'larida closure tez-tez ishlatiladi — handler yaratilgandagi kontekst ma'lumotlarini saqlash uchun:

```javascript
function setupButtons(buttonConfigs) {
  buttonConfigs.forEach(config => {
    const button = document.createElement("button");
    button.textContent = config.label;

    button.addEventListener("click", () => {
      // ✅ config closure orqali saqlanadi
      // Har bir button o'z config'iga ega
      console.log(`${config.label} clicked, action: ${config.action}`);
      config.handler();
    });

    document.body.appendChild(button);
  });
}

setupButtons([
  { label: "Save", action: "save", handler: () => console.log("Saving...") },
  { label: "Delete", action: "delete", handler: () => console.log("Deleting...") }
]);
// ✅ Har bir button o'z config closure'iga ega — to'g'ri label va handler
```

---

### Memoization

Memoization — funksiya natijasini cache'lash. Bir xil argument bilan chaqirilganda qayta hisoblash o'rniga cache'dan javob berish. Cache closure ichida saqlanadi:

```javascript
function memoize(fn) {
  const cache = new Map();
  // ✅ cache closure ichida — tashqaridan accessible emas

  return function (...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      console.log(`Cache hit: ${key}`);
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

function expensiveCalculation(n) {
  console.log(`Computing for ${n}...`);
  let result = 0;
  for (let i = 0; i < n * 1000000; i++) result += i;
  return result;
}

const memoized = memoize(expensiveCalculation);
memoized(100);  // "Computing for 100..." — hisoblaydi
memoized(100);  // "Cache hit: [100]" — cache'dan
memoized(200);  // "Computing for 200..." — yangi argument, hisoblaydi
```

---

### Iterators (Stateful)

Closure yordamida stateful iterator yaratish — har chaqiruvda keyingi qiymatni qaytaruvchi funksiya:

```javascript
function createRangeIterator(start, end, step = 1) {
  let current = start;

  return {
    next() {
      if (current > end) {
        return { value: undefined, done: true };
      }
      const value = current;
      current += step;
      // ✅ current closure ichida — har chaqiruvda oshadi
      return { value, done: false };
    }
  };
}

const iter = createRangeIterator(1, 5);
console.log(iter.next()); // { value: 1, done: false }
console.log(iter.next()); // { value: 2, done: false }
console.log(iter.next()); // { value: 3, done: false }
console.log(iter.next()); // { value: 4, done: false }
console.log(iter.next()); // { value: 5, done: false }
console.log(iter.next()); // { value: undefined, done: true }
```

---

## Memory va Closures

### Nazariya

Closure **memory** bilan bevosita bog'liq — tashqi scope'ning Environment Record'i heap'da saqlanadi va GC (Garbage Collector) uni closure reference mavjud ekan tozalay olmaydi.

Closure'ning memory lifecycle'i:

1. **Yaratilish** — tashqi funksiya chaqirilganda Environment Record yaratiladi
2. **Bog'lanish** — ichki funksiya `[[Environment]]` orqali shu record'ga reference oladi
3. **Tashqi funksiya tugashi** — EC call stack'dan chiqadi, lekin Environment Record heap'da qoladi
4. **Closure ishlash davri** — closure funksiya mavjud ekan, record tirik
5. **Tozalanish** — closure funksiyaga hech qanday reference qolmaganida, GC Environment Record'ni tozalaydi

GC **mark-and-sweep** algoritmi bilan ishlaydi: root'dan (global scope, call stack, register'lar) boshlab barcha accessible object'larni belgilaydi. Belgisiz qolganlar — tozalanadi. Closure'ning Environment Record'i closure funksiya orqali root'dan accessible — shuning uchun saqlanadi.

### Under the Hood

```
function createClosure() {
  const data = { items: [1, 2, 3] };  // Heap'da yaratildi
  let count = 0;                       // Context object'ga ko'chadi

  return function () {
    count++;
    return data;
  };
}

let fn = createClosure();

// Memory holati:
// Root → fn → fn.[[Environment]] → Context { count: 0, data: → Object { items: [...] } }
// ✅ Barcha zanjir root'dan accessible → GC tozalamaydi

fn();  // count: 1
fn();  // count: 2

fn = null;
// fn ga reference yo'q bo'ldi
// Root → ??? → Context { count, data } ga yo'l yo'q
// ✅ GC Context object'ni VA data object'ni tozalaydi
```

### Kod Misollari

Memory'ni nazorat qilish:

```javascript
function heavyOperation() {
  const largeArray = new Array(1_000_000).fill("data");
  let processCount = 0;

  const processor = function () {
    processCount++;
    return largeArray.length; // ✅ largeArray ga reference — heap'da saqlanadi
  };

  // Tozalash funksiyasi
  const cleanup = function () {
    // largeArray ni bo'shatish imkonsiz — closure reference saqlaydi
    // Yagona yo'l — processor va cleanup reference'larini null qilish
  };

  return { processor, cleanup };
}

let ops = heavyOperation();
ops.processor(); // ishlaydi, largeArray heap'da

// Tozalash — barcha reference'larni yo'q qilish:
ops = null;
// ✅ GC endi largeArray ni tozalashi mumkin
```

---

## Memory Leaks va Oldini Olish

### Nazariya

Memory leak — keraksiz bo'lib qolgan memory'ning tozalanmasligi. Closure'lar bilan memory leak quyidagi hollarda sodir bo'ladi:

**1. Keraksiz reference saqlash:**
Closure tashqi scope'dagi katta object'ga reference saqlaydi, lekin aslida faqat kichik qismini ishlatadi. V8 optimizatsiya qiladi (faqat ishlatilgan o'zgaruvchilarni Context'ga ko'chiradi), lekin `eval` yoki `debugger` ishlatilsa bu optimizatsiya o'chadi.

**2. Event listener'lar:**
DOM element'ga closure bilan event listener qo'shiladi, lekin element olib tashlanganida listener remove qilinmaydi — closure (va u ushlab turgan scope) GC bo'lmaydi.

**3. Timer'lar:**
`setInterval` closure'si to'xtatilmasa, u ushlab turgan scope abadiy tirik qoladi.

**4. Aylanma (circular) reference:**
Closure va DOM element bir-biriga reference saqlasa — ikkalasi ham GC bo'lmaydi (zamonaviy engine'larda bu muammo deyarli hal qilingan, lekin eski IE da katta muammo edi).

### Kod Misollari

Leak va uni tuzatish:

```javascript
// ❌ LEAK: setInterval to'xtatilmasa closure abadiy tirik
function startPolling(url) {
  const results = [];
  // results closure ichida — tozalanmaydi

  setInterval(async () => {
    const data = await fetch(url).then(r => r.json());
    results.push(data); // ❌ results cheksiz o'sadi
  }, 5000);
}
startPolling("/api/status");
// setInterval hech qachon to'xtatilmaydi — results abadiy o'sadi

// ✅ TO'G'RI: intervalId saqlash va tozalash
function startPolling(url) {
  const results = [];
  const MAX_RESULTS = 100;

  const intervalId = setInterval(async () => {
    const data = await fetch(url).then(r => r.json());
    results.push(data);
    if (results.length > MAX_RESULTS) results.shift(); // ✅ limit
  }, 5000);

  // Tozalash funksiyasi qaytarish
  return function stopPolling() {
    clearInterval(intervalId);
    results.length = 0; // ✅ array bo'shatish
  };
}

const stop = startPolling("/api/status");
// Kerak bo'lganda:
stop(); // ✅ interval to'xtaydi, results bo'shatiladi, GC tozalaydi
```

Event listener leak va yechimi:

```javascript
// ❌ LEAK: listener remove qilinmasa closure tirik qoladi
function setupHandler(element) {
  const heavyData = loadHeavyData();

  element.addEventListener("click", function handler() {
    process(heavyData); // closure — heavyData saqlanadi
  });

  // element DOM'dan olib tashlansa ham, handler va heavyData tirik
}

// ✅ TO'G'RI: AbortController bilan
function setupHandler(element) {
  const heavyData = loadHeavyData();
  const controller = new AbortController();

  element.addEventListener("click", function handler() {
    process(heavyData);
  }, { signal: controller.signal }); // ✅ AbortController bilan

  // Tozalash:
  return () => controller.abort(); // ✅ listener ham, closure ham tozalanadi
}
```

---

## Klassik Loop + Closure Muammosi

### Nazariya

Bu JavaScript'dagi eng mashhur closure muammosi — `var` bilan for loop ichida closure hosil qilganda barcha closure'lar **bitta o'zgaruvchiga** reference saqlaydi.

Muammoning ildizi: `var` block scope yaratMAYDI — bitta function scope binding mavjud. Barcha iteratsiyalardagi closure'lar shu bitta binding'ga murojaat qiladi. Loop tugaganda o'zgaruvchi oxirgi qiymatga ega — barcha closure'lar shu qiymatni ko'radi.

### Under the Hood

```
// var bilan:
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}

Memory model:
Function Environment Record: { i: 0 → 1 → 2 → 3 }
                                            ↑
setTimeout callback 0 ─────────────────────┘
setTimeout callback 1 ─────────────────────┘  (BARCHASI BITTA i)
setTimeout callback 2 ─────────────────────┘

Loop tugaganda i = 3, barcha callback'lar 3 ni ko'radi.

─────────────────────────────────────

// let bilan:
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}

Memory model:
Iteratsiya 0: Block Environment Record { i: 0 } ← callback 0
Iteratsiya 1: Block Environment Record { i: 1 } ← callback 1
Iteratsiya 2: Block Environment Record { i: 2 } ← callback 2

Har bir callback O'Z block scope'idagi i ni ko'radi.
```

### Kod Misollari

Muammo va barcha yechimlar:

```javascript
// ❌ MUAMMO:
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3

// ✅ YECHIM 1: let ishlatish (eng oddiy, zamonaviy)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2

// ✅ YECHIM 2: IIFE bilan yangi scope yaratish (ES5 usuli)
for (var i = 0; i < 3; i++) {
  (function (j) {
    // j — IIFE ning local parametri, i ning COPY'si
    setTimeout(() => console.log(j), 100);
  })(i);
}
// Output: 0, 1, 2

// ✅ YECHIM 3: setTimeout ning uchinchi argumenti
for (var i = 0; i < 3; i++) {
  setTimeout((j) => console.log(j), 100, i);
  // setTimeout 3-argument callback'ga pass qilinadi
}
// Output: 0, 1, 2

// ✅ YECHIM 4: bind bilan
for (var i = 0; i < 3; i++) {
  setTimeout(console.log.bind(console, i), 100);
}
// Output: 0, 1, 2
```

---

## Performance Considerations

### Nazariya

Closure'lar kuchli, lekin ularning performance ta'sirini bilish kerak:

**1. Memory overhead:**
Har bir closure tashqi scope'ning Environment Record'iga reference saqlaydi — bu qo'shimcha memory. Agar minglab closure yaratilsa va har biri katta scope'ga reference saqlasa — memory usage oshadi.

**2. Scope chain length:**
Chuqur nested closure'larda o'zgaruvchi qidirish uzoqroq scope chain bo'ylab yuradi. V8 scope caching bilan buni optimizatsiya qiladi, lekin extremely deep nesting'da ta'sir sezilishi mumkin.

**3. Engine optimizatsiyasi:**
V8 closure'larni yaxshi optimizatsiya qiladi — faqat ishlatilgan o'zgaruvchilarni heap'ga ko'chiradi. Lekin `eval`, `with`, `debugger` statement'lari bu optimizatsiyani buzadi.

**4. Prototype method vs Closure method:**
Class/prototype method'lari barcha instance'lar orasida **share** qilinadi (bitta funksiya). Closure method'lari har instance uchun **alohida** funksiya yaratadi — ko'proq memory.

### Kod Misollari

```javascript
// ❌ Har instance uchun alohida funksiya — ko'proq memory
function createUser(name) {
  return {
    getName() { return name; },        // har user uchun yangi funksiya
    greet() { return `Hi, ${name}`; }  // har user uchun yangi funksiya
  };
}
// 10000 user = 20000 funksiya object

// ✅ Prototype method'lari share qilinadi — kam memory
class User {
  #name; // ES2022 private field
  constructor(name) { this.#name = name; }
  getName() { return this.#name; }       // barcha instance'lar SHARE qiladi
  greet() { return `Hi, ${this.#name}`; } // barcha instance'lar SHARE qiladi
}
// 10000 user = 2 funksiya object (prototype'da)
```

Amalda bu farq katta hajmdagi object'lar uchun muhim. Kichik miqdor uchun closure method'lari yaxshi ishlaydi va encapsulation afzalligiga ega.

---

## Common Mistakes

### ❌ Xato 1: Closure Reference Saqlashini Tushunmaslik

```javascript
function createFunctions() {
  let value = 1;

  const getValue = () => value;
  const setValue = (v) => { value = v; };

  return { getValue, setValue };
}

const obj = createFunctions();
console.log(obj.getValue()); // 1
obj.setValue(42);
console.log(obj.getValue()); // 42 — 1 EMAS!
```

### ✅ Tushuntirish:

Closure **copy** emas, **reference** saqlaydi. `getValue` va `setValue` bir xil `value` ga murojaat qiladi. `setValue` qiymatni o'zgartirsa — `getValue` yangi qiymatni ko'radi.

**Nima uchun:** Ikkalasi bir scope'da yaratilgan — bir Environment Record'dagi bitta binding'ga reference ushlab turibdi. Bu shunchaki pointer — copy emas.

---

### ❌ Xato 2: Closure Ichida Tashqi O'zgaruvchini Keraksiz Ushlab Turish

```javascript
function processLargeData() {
  const hugeArray = fetchMillionRecords(); // 100MB data

  const summary = computeSummary(hugeArray);

  return function getSummary() {
    return summary;
    // ❌ getSummary faqat summary ishlatadi
    // Lekin scope'da hugeArray ham bor — u ham saqlanib qolishi MUMKIN
  };
}
```

### ✅ To'g'ri usul:

```javascript
function processLargeData() {
  const hugeArray = fetchMillionRecords();
  const summary = computeSummary(hugeArray);

  // hugeArray ni scope'dan chiqarish — alohida funksiyada ishlash
  return function getSummary() {
    return summary;
    // ✅ V8 faqat summary ni Context'ga ko'chiradi
    // hugeArray closure tomonidan ishlatilmaydi → GC tozalaydi
  };
}

// Yoki aniqroq bo'lishi uchun — scope ajratish:
function processLargeData() {
  const summary = (function () {
    const hugeArray = fetchMillionRecords();
    return computeSummary(hugeArray);
    // hugeArray IIFE scope'da — tashqariga chiqmaydi
  })();

  return function getSummary() {
    return summary; // ✅ faqat summary closure'da
  };
}
```

**Nima uchun:** V8 odatda faqat ishlatilgan o'zgaruvchilarni Context'ga ko'chiradi. Lekin `eval` yoki `debugger` bo'lsa — barcha o'zgaruvchilar saqlanadi. Xavfsiz yondashuv — katta data'ni alohida scope'da ishlash.

---

### ❌ Xato 3: Stale Closure — Eskirgan Qiymat

```javascript
function createTimer() {
  let seconds = 0;

  setInterval(() => {
    seconds++;
  }, 1000);

  return function getTime() {
    return seconds;
    // ✅ Bu aslida TO'G'RI ishlaydi — reference, har doim yangi qiymat
  };
}

// Lekin React da bu muammo bor — stale closure:
// function Counter() {
//   const [count, setCount] = useState(0);
//
//   useEffect(() => {
//     const id = setInterval(() => {
//       console.log(count); // ❌ Doim 0 — closure eskirgan count ni ko'radi
//     }, 1000);
//     return () => clearInterval(id);
//   }, []); // [] — faqat bir marta, yangi count closure hosil BO'LMAYDI
// }
```

### ✅ To'g'ri usul (React context'ida):

```javascript
// useEffect dependency array'ga count qo'shish:
// useEffect(() => {
//   const id = setInterval(() => {
//     console.log(count); // ✅ Har count o'zgarganda yangi closure
//   }, 1000);
//   return () => clearInterval(id);
// }, [count]); // count o'zgarganda effect qayta ishlaydi
```

**Nima uchun:** React hook'lari har render'da yangi closure yaratadi. Agar dependency array to'g'ri bo'lmasa — eski render'dagi closure ishlaydi, yangi state'ni ko'rmaydi. Bu "stale closure" muammosi.

---

### ❌ Xato 4: for...in + Closure Muammosi

```javascript
const config = { host: "localhost", port: 3000, protocol: "https" };
const getters = {};

for (const key in config) {
  getters[key] = () => config[key];
  // ✅ Bu aslida TO'G'RI ishlaydi — const har iteratsiyada yangi binding
}

// Lekin var bilan:
for (var key in config) {
  getters[key] = () => config[key]; // ❌ key doim oxirgi qiymat
}
```

### ✅ To'g'ri usul:

```javascript
// let yoki const ishlatish — har iteratsiya yangi binding:
for (const key in config) {
  getters[key] = () => config[key]; // ✅ har biri o'z key
}

// Yoki Object.entries bilan:
Object.entries(config).forEach(([key, value]) => {
  getters[key] = () => value; // ✅ har callback o'z value
});
```

**Nima uchun:** `const`/`let` `for...in` da ham har iteratsiyada yangi binding yaratadi. `var` esa bitta binding — loop tugaganda oxirgi key qoladi.

---

## Amaliy Mashqlar

### Mashq 1: Oddiy Closure (Oson)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
function outer() {
  let x = 10;

  function inner() {
    x++;
    console.log(x);
  }

  inner();
  inner();
  return inner;
}

const fn = outer();
fn();
fn();
```

<details>
<summary>Javob</summary>

```
11
12
13
14
```

```javascript
// outer() chaqirildi — x = 10
// inner() 1-chaqiruv: x++ → x = 11, console.log(11)
// inner() 2-chaqiruv: x++ → x = 12, console.log(12)
// outer() tugadi, fn = inner (closure hosil bo'ldi)
// fn() 1-chaqiruv: x++ → x = 13, console.log(13)
// fn() 2-chaqiruv: x++ → x = 14, console.log(14)
// ✅ x reference sifatida saqlanadi — barcha chaqiruvlar bitta x ni ko'radi
```

</details>

---

### Mashq 2: Alohida Closure'lar (O'rta)

**Savol:** Quyidagi kodning output'ini ayting:

```javascript
function makeCounter() {
  let count = 0;
  return () => ++count;
}

const counterA = makeCounter();
const counterB = makeCounter();

console.log(counterA()); // ?
console.log(counterA()); // ?
console.log(counterB()); // ?
console.log(counterA()); // ?
```

<details>
<summary>Javob</summary>

```
1
2
1
3
```

```javascript
// makeCounter() 1-chaqiruv → counterA closure: { count: 0 }
// makeCounter() 2-chaqiruv → counterB closure: { count: 0 } — YANGI scope

// counterA() → count: 0 → 1 (counterA scope'da)
// counterA() → count: 1 → 2 (counterA scope'da)
// counterB() → count: 0 → 1 (counterB scope'da — alohida!)
// counterA() → count: 2 → 3 (counterA scope'da — davom etadi)
// ✅ Har bir makeCounter() chaqiruvi yangi scope — alohida count
```

</details>

---

### Mashq 3: once() Implement Qilish (O'rta)

**Savol:** `once(fn)` funksiyasini yozing. U berilgan `fn` ni faqat **bir marta** chaqiradi. Keyingi chaqiruvlarda birinchi chaqiruvning natijasini qaytaradi.

```javascript
function once(fn) {
  // Sizning kodingiz
}

const initialize = once(() => {
  console.log("Initializing...");
  return { status: "ready" };
});

console.log(initialize()); // "Initializing..." + { status: "ready" }
console.log(initialize()); // { status: "ready" } — "Initializing..." chiqmaydi
console.log(initialize()); // { status: "ready" }
```

<details>
<summary>Javob</summary>

```javascript
function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
      // ✅ fn faqat bir marta chaqiriladi
      // ✅ apply — this va args to'g'ri pass qilish uchun
    }
    return result;
    // ✅ Keyingi chaqiruvlarda cache'langan result qaytadi
  };
}
```

**Tushuntirish:** `called` va `result` closure ichida saqlanadi. Birinchi chaqiruvda `called = true` bo'ladi va `fn` bajariladi. Keyingi chaqiruvlarda `called` allaqachon `true` — `fn` chaqirilmaydi, saqlangan `result` qaytariladi. Bu pattern initialization, expensive computation, va singleton-like behavior uchun ishlatiladi.

</details>

---

### Mashq 4: Private State bilan Stack (Qiyin)

**Savol:** Closure yordamida `createStack()` funksiyasini yozing. `push()`, `pop()`, `peek()`, `size()`, `isEmpty()`, `toArray()` method'lariga ega bo'lsin. Ichki array tashqaridan accessible bo'lmasin.

<details>
<summary>Javob</summary>

```javascript
function createStack() {
  const items = []; // ✅ Private — tashqaridan accessible emas

  return {
    push(element) {
      items.push(element);
      return items.length; // yangi size qaytaradi
    },

    pop() {
      if (items.length === 0) {
        throw new Error("Stack is empty");
      }
      return items.pop();
    },

    peek() {
      if (items.length === 0) return undefined;
      return items[items.length - 1];
    },

    size() {
      return items.length;
    },

    isEmpty() {
      return items.length === 0;
    },

    toArray() {
      return [...items]; // ✅ copy qaytaradi — original himoyalangan
    }
  };
}

const stack = createStack();
stack.push(1);
stack.push(2);
stack.push(3);
console.log(stack.peek());    // 3
console.log(stack.pop());     // 3
console.log(stack.size());    // 2
console.log(stack.toArray()); // [1, 2]
console.log(stack.isEmpty()); // false

// items tashqaridan accessible emas:
console.log(stack.items); // undefined ✅
```

**Tushuntirish:** `items` array `createStack()` scope'da — faqat return qilingan method'lar closure orqali unga murojaat qila oladi. `toArray()` copy qaytaradi (`[...items]`) — tashqaridan original array o'zgartirilishi mumkin emas. Bu pattern class'dagi `#private` field'larga alternativ.

</details>

---

### Mashq 5: Pipe + Closure (Qiyin)

**Savol:** Quyidagi kodning output'ini ayting va har bir qadamni tushuntiring:

```javascript
function pipe(...fns) {
  return function (value) {
    return fns.reduce((acc, fn) => fn(acc), value);
  };
}

function add(x) {
  return function (y) {
    return x + y;
  };
}

function multiply(x) {
  return function (y) {
    return x * y;
  };
}

const transform = pipe(add(10), multiply(2), add(-5));
console.log(transform(5)); // ?
```

<details>
<summary>Javob</summary>

```
25
```

```javascript
// add(10) → closure { x: 10 }, returns (y) => 10 + y
// multiply(2) → closure { x: 2 }, returns (y) => 2 * y
// add(-5) → closure { x: -5 }, returns (y) => -5 + y

// pipe(add(10), multiply(2), add(-5)):
// fns = [(y) => 10 + y, (y) => 2 * y, (y) => -5 + y]
// pipe closure hosil qildi — fns saqlanadi

// transform(5):
// reduce boshlanadi, acc = 5 (initial value)
// 1-qadam: fn = add(10),     acc = 10 + 5 = 15
// 2-qadam: fn = multiply(2), acc = 2 * 15 = 30
// 3-qadam: fn = add(-5),     acc = -5 + 30 = 25
// Natija: 25
```

**Tushuntirish:** Bu misolda **ikki qatlam** closure bor:
1. `add(10)` → `x = 10` closure, `multiply(2)` → `x = 2` closure
2. `pipe(...)` → `fns` array closure

`pipe` funksiyasi funksiyalarni chapdan o'ngga ketma-ket bajaradi — har birining natijasi keyingisiga argument bo'ladi. Bu functional programming'da `pipe` pattern — data transformation pipeline.

</details>

---

## Xulosa

Bu bo'limda closure'ning barcha qatlamlari yoritildi:

- **Closure** — funksiya + uning yaratilgan paytdagi lexical environment. Reference saqlaydi, copy emas.
- **`[[Environment]]` internal slot** — funksiya yaratilganda joriy LexicalEnvironment shu slot'ga saqlanadi.
- **Closure hosil bo'lish** — 5 qadam: tashqi EC yaratilish → ichki funksiya define → ichki funksiya tashqariga chiqish → tashqi EC tugash → ichki funksiya chaqirilish.
- **V8 optimizatsiyasi** — faqat closure ishlatgan o'zgaruvchilar Context object'iga ko'chiriladi. `eval` bu optimizatsiyani buzadi.
- **Use cases** — data privacy, factory functions, partial application, module pattern, event handlers, memoization, stateful iterators.
- **Memory** — closure tashqi Environment Record'ni heap'da tirik saqlaydi. Barcha reference'lar yo'qolganda GC tozalaydi.
- **Memory leaks** — timer'lar, event listener'lar, keraksiz reference'lar. Yechim: cleanup funksiyalari, AbortController, scope ajratish.
- **Loop muammosi** — `var` bitta binding, `let` har iteratsiya yangi binding.
- **Performance** — closure method vs prototype method (memory tradeoff), scope chain depth, engine optimizatsiyalari.

Scope va closure — JavaScript'ning fundamental tushunchalari. Keyingi bo'limlardagi objects, prototypes, va ayniqsa `this` keyword closure bilan chambarchas bog'liq.

---

**Keyingi bo'lim:** [06-objects.md](06-objects.md) — Objects ichidan: object creation patterns, property descriptors (writable, enumerable, configurable), getters/setters, immutability (`Object.freeze`, `Object.seal`), deep/shallow copy, `structuredClone`, property enumeration.
