# Bo'lim 20: Error Handling

> Xatolar — dasturning tabiiy qismi. Ularni to'g'ri boshqarish — professional dasturchi va boshlang'ich o'rtasidagi asosiy farq.

---

## Mundarija

- [Error Types](#error-types)
- [try/catch/finally](#trycatchfinally)
- [Error Object](#error-object)
- [Custom Errors](#custom-errors)
- [Error Propagation](#error-propagation)
- [Async Error Handling](#async-error-handling)
- [Global Error Handlers](#global-error-handlers)
- [Error Handling Patterns](#error-handling-patterns)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Error Types

### Nazariya

JavaScript da bir nechta built-in error turlari mavjud. Har biri turli xil muammoni bildiradi:

### Built-in Error Turlari

```
┌─────────────────────────────────────────────────────────┐
│                    Error Hierarchy                        │
│                                                          │
│  Error (base class)                                      │
│   ├── SyntaxError     — kod yozilishi noto'g'ri          │
│   ├── ReferenceError  — mavjud bo'lmagan o'zgaruvchi     │
│   ├── TypeError       — noto'g'ri type da operatsiya     │
│   ├── RangeError      — qiymat ruxsat etilgan oraliqdan  │
│   ├── URIError        — noto'g'ri URI funksiya ishlatish │
│   ├── EvalError       — eval() bilan bog'liq (eski)      │
│   └── AggregateError  — bir necha xatoni birlashtirish   │
└─────────────────────────────────────────────────────────┘
```

### Kod Misollari

```javascript
// === SyntaxError — kod noto'g'ri yozilgan ===
// eval("if("); // SyntaxError: Unexpected end of input
// JSON.parse("{invalid}"); // SyntaxError: Unexpected token i
JSON.parse("{'key': 'value'}"); // SyntaxError — single quotes!

// === ReferenceError — o'zgaruvchi topilmadi ===
// console.log(notDefined); // ReferenceError: notDefined is not defined
// ⚠️ typeof ishlatsa — xato bermaydi:
typeof notDefined; // "undefined" — xatosiz

// === TypeError — noto'g'ri type ===
// null.toString();           // TypeError: Cannot read properties of null
// undefined.name;            // TypeError: Cannot read properties of undefined
// "string"();                // TypeError: "string" is not a function
// const x = 5; x = 10;      // TypeError: Assignment to constant variable

// === RangeError — qiymat chegaradan tashqari ===
// new Array(-1);             // RangeError: Invalid array length
// (1).toFixed(200);          // RangeError: toFixed() digits argument must be between 0 and 100
// function f() { f(); } f(); // RangeError: Maximum call stack size exceeded (stack overflow)

// === URIError — URI funksiyalari ===
// decodeURIComponent("%");   // URIError: URI malformed

// === AggregateError — bir necha xato (ES2021) ===
const errors = [
  new TypeError("type xato"),
  new RangeError("range xato")
];
const aggError = new AggregateError(errors, "Ko'p xato");
console.log(aggError.errors); // [TypeError, RangeError]
// Promise.any() AggregateError tashlashi mumkin
```

### Under the Hood

```javascript
// Barcha Error lar Error.prototype dan meros oladi
const err = new TypeError("xato");

console.log(err instanceof TypeError); // true
console.log(err instanceof Error);     // true — barcha errorlar Error dan meros

// Error zanjiri:
// TypeError.prototype → Error.prototype → Object.prototype → null

// Error obyektining xususiyatlari:
console.log(err.name);    // "TypeError"
console.log(err.message); // "xato"
console.log(err.stack);   // Stack trace — debugging uchun
```

---

## try/catch/finally

### Nazariya

`try/catch/finally` — xatolarni ushlab, dastur to'xtamasdan davom etish imkonini beradi.

### Sintaksis va Ishlash Tartibi

```javascript
try {
  // ① Xato bo'lishi mumkin bo'lgan kod
  riskyOperation();
} catch (error) {
  // ② Xato bo'lsa — shu yerga tushadi
  console.error("Xato:", error.message);
} finally {
  // ③ DOIM ishlaydi — xato bo'lsa ham, bo'lmasa ham
  cleanup();
}
```

### Kod Misollari

```javascript
// === Asosiy ishlatish ===
try {
  const data = JSON.parse("invalid json");
} catch (error) {
  console.log(error.name);    // "SyntaxError"
  console.log(error.message); // "Unexpected token i in JSON at position 0"
}
console.log("Dastur davom etadi!"); // ✅ To'xtamaydi

// === finally — doim ishlaydi ===
function readFile() {
  const file = openFile("data.txt");
  try {
    const data = file.read();
    return data;
  } catch (error) {
    console.error("Fayl o'qishda xato:", error);
    return null;
  } finally {
    file.close(); // ✅ DOIM yopiladi — xato bo'lsa ham, return bo'lsa ham
  }
}

// === catch siz try/finally ===
function alwaysCleanup() {
  const resource = acquire();
  try {
    useResource(resource);
  } finally {
    resource.release(); // xato bo'lsa throw davom etadi, lekin cleanup ishlaydi
  }
}

// === Specific error type tekshirish ===
try {
  someOperation();
} catch (error) {
  if (error instanceof TypeError) {
    console.log("Type xatosi:", error.message);
  } else if (error instanceof RangeError) {
    console.log("Range xatosi:", error.message);
  } else if (error instanceof SyntaxError) {
    console.log("Sintaksis xatosi:", error.message);
  } else {
    throw error; // kutilmagan xato — qayta throw
  }
}
```

### Under the Hood — finally va return

```javascript
// ⚠️ finally ichida return — catch ni qaytaradi!
function tricky() {
  try {
    return "try";
  } finally {
    return "finally"; // ⚠️ "try" ni OVERRIDE qiladi!
  }
}
console.log(tricky()); // "finally" — try dagi return yo'qoldi!

// ⚠️ finally ichida return — throw ni ham yo'qotadi!
function trickier() {
  try {
    throw new Error("xato!");
  } finally {
    return "finally"; // ⚠️ Error yo'qoldi — XAVFLI!
  }
}
console.log(trickier()); // "finally" — error hech qachon tashqariga chiqmaydi

// ✅ QOIDA: finally ichida HECH QACHON return yozmang!
```

### ASCII Diagram — try/catch/finally Flow

```
  try bloku
    │
    ├── Xato YO'Q ──────────────────┐
    │                               │
    ├── Xato BOR ──┐                │
    │              ▼                │
    │          catch bloku          │
    │              │                │
    │              ├── throw ──┐    │
    │              │           │    │
    │              ▼           │    │
    │         (handled)        │    │
    │              │           │    │
    └──────────────┼───────────┼────┘
                   │           │
                   ▼           ▼
              finally bloku (DOIM)
                   │           │
                   ▼           ▼
              return        throw
             (normal)      (re-throw)
```

---

## Error Object

### Nazariya

Error obyekti xato haqida to'liq ma'lumot beradi. Uni yaratish va tashlab (throw) mumkin.

### Kod Misollari

```javascript
// Error yaratish
const error = new Error("Nimadir noto'g'ri ketdi");
console.log(error.name);     // "Error"
console.log(error.message);  // "Nimadir noto'g'ri ketdi"
console.log(error.stack);    // Stack trace — qayerda sodir bo'lgani

// Stack trace misoli:
/*
Error: Nimadir noto'g'ri ketdi
    at calculateTotal (app.js:15:11)
    at processOrder (app.js:42:5)
    at main (app.js:67:3)
*/

// throw — har qanday narsa throw qilish mumkin (lekin Error tavsiya!)
throw new Error("xato xabari");
throw new TypeError("type xato");
throw new RangeError("range xato");

// ⚠️ Texnik jihatdan string ham throw qilish mumkin — lekin YOMON AMALIYOT
// throw "xato!";          // ❌ stack trace yo'q
// throw 42;               // ❌ stack trace yo'q
// throw { code: 500 };    // ❌ stack trace yo'q
// DOIM Error yoki uning subclass ini throw qiling!
```

### Error cause (ES2022)

```javascript
// cause — xatoning asl sababini saqlash
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    // Asl xatoni cause sifatida saqlash
    throw new Error(`Foydalanuvchi ${id} yuklanmadi`, {
      cause: error // ← ES2022
    });
  }
}

try {
  await fetchUser(42);
} catch (error) {
  console.log(error.message);     // "Foydalanuvchi 42 yuklanmadi"
  console.log(error.cause);        // Error: HTTP 404
  console.log(error.cause.message); // "HTTP 404"
}
```

---

## Custom Errors

### Nazariya

O'z xato class'larimizni yaratish — katta dasturlarda xatolarni kategoriyalash va boshqarish uchun juda muhim.

### Kod Misollari

```javascript
// === Custom Error class ===
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

class NotFoundError extends Error {
  constructor(resource, id) {
    super(`${resource} #${id} topilmadi`);
    this.name = "NotFoundError";
    this.resource = resource;
    this.id = id;
    this.statusCode = 404;
  }
}

class AuthorizationError extends Error {
  constructor(message = "Ruxsat yo'q") {
    super(message);
    this.name = "AuthorizationError";
    this.statusCode = 403;
  }
}

// === Ishlatish ===
function validateUser(data) {
  if (!data.name || data.name.length < 2) {
    throw new ValidationError("Ism kamida 2 ta harf bo'lishi kerak", "name");
  }
  if (!data.email || !data.email.includes("@")) {
    throw new ValidationError("Email noto'g'ri formatda", "email");
  }
  if (data.age < 0 || data.age > 150) {
    throw new RangeError("Yosh 0-150 orasida bo'lishi kerak");
  }
}

// === Error ni ushlash ===
try {
  validateUser({ name: "A", email: "invalid", age: 200 });
} catch (error) {
  if (error instanceof ValidationError) {
    console.log(`Validatsiya xatosi [${error.field}]: ${error.message}`);
    // Form da shu field ni qizil qilish
  } else if (error instanceof RangeError) {
    console.log("Qiymat chegaradan tashqari:", error.message);
  } else {
    throw error; // boshqa xatolarni qayta throw
  }
}
```

### Error Hierarchy Pattern

```javascript
// Katta loyihalar uchun xato hierarchy
class AppError extends Error {
  constructor(message, statusCode = 500, cause) {
    super(message, { cause });
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = true; // operational vs programming error
  }
}

class ClientError extends AppError {
  constructor(message, statusCode = 400, cause) {
    super(message, statusCode, cause);
  }
}

class ServerError extends AppError {
  constructor(message, cause) {
    super(message, 500, cause);
    this.isOperational = false; // server xatosi — critical
  }
}

// Aniq error turlari
class BadRequestError extends ClientError {
  constructor(message) { super(message, 400); }
}

class UnauthorizedError extends ClientError {
  constructor(message = "Autentifikatsiya kerak") { super(message, 401); }
}

class ForbiddenError extends ClientError {
  constructor(message = "Ruxsat yo'q") { super(message, 403); }
}

class NotFoundError extends ClientError {
  constructor(resource = "Resurs") { super(`${resource} topilmadi`, 404); }
}

// instanceof bilan tekshirish
try {
  throw new NotFoundError("Foydalanuvchi");
} catch (error) {
  console.log(error instanceof NotFoundError);  // true
  console.log(error instanceof ClientError);    // true
  console.log(error instanceof AppError);       // true
  console.log(error instanceof Error);          // true
  console.log(error.statusCode);                // 404
  console.log(error.isOperational);             // true
}
```

### Under the Hood

```
Error Hierarchy:

Error
 └── AppError (isOperational, statusCode)
      ├── ClientError (4xx)
      │    ├── BadRequestError (400)
      │    ├── UnauthorizedError (401)
      │    ├── ForbiddenError (403)
      │    └── NotFoundError (404)
      │
      └── ServerError (5xx, isOperational: false)
           ├── DatabaseError
           └── ExternalServiceError
```

---

## Error Propagation

### Nazariya

Error propagation — xato funksiya zanjiri bo'ylab yuqoriga ko'tarilishi. Agar funksiya ichida try/catch bo'lmasa, xato chaqiruvchi funksiyaga qaytadi — to'xtaguncha yoki ushlanguncha.

### Kod Misollari

```javascript
function step3() {
  throw new Error("step3 da xato!"); // ① Xato sodir bo'ldi
}

function step2() {
  step3(); // ② try/catch yo'q — xato yuqoriga ko'tariladi
}

function step1() {
  step2(); // ③ Bu yerda ham yo'q — yana yuqoriga
}

try {
  step1(); // ④ Eng yuqori — shu yerda ushlanadi
} catch (error) {
  console.log(error.message); // "step3 da xato!"
  console.log(error.stack);
  // Error: step3 da xato!
  //   at step3 (...)
  //   at step2 (...)
  //   at step1 (...)
}
```

### ASCII Diagram

```
  Call Stack               Error Propagation
  
  ┌──────────┐            ┌──────────┐
  │  step3   │ ──throw──▶ │  Error!  │
  ├──────────┤            └────┬─────┘
  │  step2   │ ◀──────────────┘ catch yo'q → yuqoriga
  ├──────────┤            ┌────┴─────┐
  │  step1   │ ◀──────────┘ catch yo'q → yuqoriga
  ├──────────┤            ┌────┴─────┐
  │ try/catch│ ◀──────────┘ ✅ USHLANDI!
  └──────────┘
```

### Re-throwing Pattern

```javascript
// Kerakli xatolarni ushlash, qolganlarini qayta throw
function processData(data) {
  try {
    return JSON.parse(data);
  } catch (error) {
    if (error instanceof SyntaxError) {
      // JSON parse xatosini boshqarish
      console.warn("Noto'g'ri JSON:", data);
      return null;
    }
    throw error; // boshqa xatolarni qayta throw — yuqoriga ko'tariladi
  }
}
```

### Error Translation Pattern

```javascript
// Past darajadagi xatoni yuqori darajadagi xatoga aylantirish
async function getUserProfile(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    // HTTP xatoni business logic xatoga aylantirish
    if (error.message.includes("404")) {
      throw new NotFoundError("Foydalanuvchi");
    }
    if (error.message.includes("403")) {
      throw new ForbiddenError();
    }
    throw new ServerError("Profil yuklanmadi", error);
  }
}
```

---

## Async Error Handling

### Nazariya

Asinxron koddagi xatolarni boshqarish sinxron koddan farq qiladi. Callbacks, Promises va async/await har birida o'z usuli bor.

### Callback Error Handling (Node.js pattern)

```javascript
// Error-first callback pattern
function readFile(path, callback) {
  // ... asinxron operatsiya
  if (error) {
    callback(new Error("Fayl o'qib bo'lmadi"), null); // ① xato birinchi
  } else {
    callback(null, data); // ② xato yo'q = null
  }
}

readFile("data.txt", (error, data) => {
  if (error) {
    console.error("Xato:", error.message);
    return;
  }
  console.log("Data:", data);
});
```

### Promise Error Handling

```javascript
// === .catch() bilan ===
fetch("/api/users")
  .then(response => {
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  })
  .then(data => {
    console.log("Users:", data);
  })
  .catch(error => {
    console.error("Xato:", error.message);
  });

// === catch pozitsiyasi muhim! ===
fetch("/api/data")
  .then(response => response.json())
  .catch(error => {
    console.log("fetch yoki json xatosi");
    return { fallback: true }; // ✅ recovery — keyingi then ga o'tadi
  })
  .then(data => {
    console.log("Data:", data); // fallback data bilan ishlaydi
  })
  .catch(error => {
    console.log("Oxirgi xato handler");
  });

// === Promise.all — bitta xato BARCHASINI reject qiladi ===
Promise.all([
  fetch("/api/users"),
  fetch("/api/posts"),
  fetch("/api/comments")
])
.then(([users, posts, comments]) => {
  // barchasi muvaffaqiyatli
})
.catch(error => {
  // BITTA xato bo'lsa ham shu yerga tushadi
  // boshqa promise'lar natijasi yo'qoladi!
});

// === Promise.allSettled — barcha natijalarni olish ===
const results = await Promise.allSettled([
  fetch("/api/users"),
  fetch("/api/posts"),
  fetch("/api/comments")
]);

results.forEach((result, i) => {
  if (result.status === "fulfilled") {
    console.log(`#${i} muvaffaqiyatli:`, result.value);
  } else {
    console.log(`#${i} xato:`, result.reason);
  }
});
```

### Async/Await Error Handling

```javascript
// === try/catch bilan — eng qulay ===
async function loadDashboard() {
  try {
    const user = await fetchUser();
    const posts = await fetchPosts(user.id);
    const comments = await fetchComments(posts[0].id);
    
    return { user, posts, comments };
  } catch (error) {
    console.error("Dashboard yuklanmadi:", error);
    return null; // fallback
  }
}

// === Har bir await uchun alohida xato boshqarish ===
async function loadData() {
  let user, posts;

  try {
    user = await fetchUser();
  } catch (error) {
    throw new Error("Foydalanuvchi yuklanmadi", { cause: error });
  }

  try {
    posts = await fetchPosts(user.id);
  } catch (error) {
    console.warn("Postlar yuklanmadi, bo'sh array qaytariladi");
    posts = []; // recovery
  }

  return { user, posts };
}

// === Parallel async — Promise.all + try/catch ===
async function loadAll() {
  try {
    const [users, posts] = await Promise.all([
      fetch("/api/users").then(r => r.json()),
      fetch("/api/posts").then(r => r.json())
    ]);
    return { users, posts };
  } catch (error) {
    console.error("Parallel yuklashda xato:", error);
  }
}
```

### Unhandled Promise Rejection

```javascript
// ⚠️ XAVFLI — catch bo'lmasa:
async function forgotCatch() {
  const data = await fetch("/api/404"); // reject bo'lishi mumkin
  // catch yo'q!
}

forgotCatch(); // UnhandledPromiseRejection warning!

// ✅ Doim catch qo'ying:
forgotCatch().catch(console.error);
// Yoki: try/catch ichida chaqiring
```

---

## Global Error Handlers

### Nazariya

Global error handler'lar — ushlanmagan xatolar uchun **oxirgi himoya qatlami**. Production dasturlarda logging va monitoring uchun juda muhim.

### Browser

```javascript
// === Sinxron xatolar ===
window.onerror = function(message, source, lineno, colno, error) {
  console.log("Global xato:", {
    message,           // xato xabari
    source,            // fayl nomi
    lineno,            // qaysi qator
    colno,             // qaysi ustun
    error              // Error obyekti
  });
  
  // Xatoni logging servisga yuborish
  sendToLogging({ message, source, lineno, colno, stack: error?.stack });
  
  return true; // true — brauzer console da ko'rsatmaydi
  // return false yoki undefined — default xulq (console da ko'rsatadi)
};

// Yoki addEventListener bilan:
window.addEventListener("error", (event) => {
  console.log("Error event:", event.error);
  // event.preventDefault(); // default console log ni to'xtatish
});

// === Ushlanmagan Promise rejection ===
window.addEventListener("unhandledrejection", (event) => {
  console.log("Ushlanmagan rejection:", event.reason);
  
  // Logging
  sendToLogging({
    type: "unhandledRejection",
    reason: event.reason?.message || String(event.reason),
    stack: event.reason?.stack
  });

  event.preventDefault(); // default console warning ni to'xtatish
});

// === Rejection ushlangandan keyin ===
window.addEventListener("rejectionhandled", (event) => {
  console.log("Rejection keyin ushlandi:", event.reason);
});
```

### Node.js

```javascript
// === Ushlanmagan exception ===
process.on("uncaughtException", (error) => {
  console.error("Ushlanmagan xato:", error);
  // Logging
  // ⚠️ Keyin process ni to'xtatish TAVSIYA QILINADI
  process.exit(1);
});

// === Ushlanmagan Promise rejection ===
process.on("unhandledRejection", (reason, promise) => {
  console.error("Ushlanmagan rejection:", reason);
  // Logging
});
```

---

## Error Handling Patterns

### Pattern 1: Result Type (Go-style)

```javascript
// Xato throw qilish o'rniga — natija obyekti qaytarish
function safeDivide(a, b) {
  if (b === 0) {
    return { ok: false, error: "Nolga bo'lish mumkin emas" };
  }
  return { ok: true, value: a / b };
}

const result = safeDivide(10, 0);
if (!result.ok) {
  console.log("Xato:", result.error);
} else {
  console.log("Natija:", result.value);
}

// Asinxron versiya
async function safeAsync(fn) {
  try {
    const data = await fn();
    return { ok: true, data };
  } catch (error) {
    return { ok: false, error };
  }
}

const { ok, data, error } = await safeAsync(() => fetch("/api").then(r => r.json()));
if (!ok) console.error(error);
```

### Pattern 2: Retry Pattern

```javascript
// Xato bo'lganda qayta urinish
async function withRetry(fn, maxRetries = 3, delay = 1000) {
  let lastError;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      console.warn(`Urinish ${attempt}/${maxRetries} muvaffaqiyatsiz:`, error.message);
      
      if (attempt < maxRetries) {
        // Exponential backoff
        const waitTime = delay * Math.pow(2, attempt - 1);
        await new Promise(r => setTimeout(r, waitTime));
      }
    }
  }
  
  throw new Error(`${maxRetries} urinishdan keyin ham xato`, { cause: lastError });
}

// Ishlatish
const data = await withRetry(
  () => fetch("/api/unstable").then(r => r.json()),
  3,    // 3 marta urinish
  1000  // 1s, 2s, 4s kutish
);
```

### Pattern 3: Circuit Breaker

```javascript
class CircuitBreaker {
  constructor(fn, { threshold = 5, timeout = 30000 } = {}) {
    this.fn = fn;
    this.threshold = threshold;
    this.timeout = timeout;
    this.failures = 0;
    this.state = "CLOSED"; // CLOSED → OPEN → HALF_OPEN
    this.nextAttempt = 0;
  }

  async execute(...args) {
    if (this.state === "OPEN") {
      if (Date.now() < this.nextAttempt) {
        throw new Error("Circuit OPEN — so'rov rad etildi");
      }
      this.state = "HALF_OPEN";
    }

    try {
      const result = await this.fn(...args);
      this.reset();
      return result;
    } catch (error) {
      this.failures++;
      if (this.failures >= this.threshold) {
        this.state = "OPEN";
        this.nextAttempt = Date.now() + this.timeout;
        console.warn("Circuit OCHILDI — so'rovlar to'xtatildi");
      }
      throw error;
    }
  }

  reset() {
    this.failures = 0;
    this.state = "CLOSED";
  }
}

// Ishlatish
const apiBreaker = new CircuitBreaker(
  (url) => fetch(url).then(r => r.json()),
  { threshold: 3, timeout: 10000 }
);

try {
  const data = await apiBreaker.execute("/api/data");
} catch (error) {
  console.log("API xatosi yoki circuit ochiq");
}
```

### Pattern 4: Graceful Degradation

```javascript
// Asosiy funksiya ishlamasa — fallback
async function getUserData(userId) {
  // 1. API dan olishga harakat
  try {
    return await fetchFromAPI(userId);
  } catch (apiError) {
    console.warn("API ishlamadi, cache tekshirilmoqda...");
  }

  // 2. Cache dan olishga harakat
  try {
    return await getFromCache(userId);
  } catch (cacheError) {
    console.warn("Cache ham ishlamadi, default qaytarilmoqda...");
  }

  // 3. Default qiymat
  return {
    id: userId,
    name: "Noma'lum",
    avatar: "/default-avatar.png"
  };
}
```

---

## Common Mistakes

### ❌ Xato 1: Barcha xatolarni yutib yuborish

```javascript
// ❌ Noto'g'ri — xato haqida hech narsa qilinmaydi
try {
  criticalOperation();
} catch (error) {
  // bo'sh catch — "swallowing errors"
  // xato yo'qoldi — debugging mumkin emas!
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Kamida log qilish
try {
  criticalOperation();
} catch (error) {
  console.error("Critical operation failed:", error);
  // yoki logging service ga yuborish
  // yoki foydalanuvchiga xabar berish
}
```

**Nima uchun:** Bo'sh catch blok xatolarni "yutib yuboradi" — dastur noto'g'ri ishlaydi, lekin siz hech qachon bilmaysiz nima uchun. Eng kamida `console.error` qiling.

---

### ❌ Xato 2: Error o'rniga string throw qilish

```javascript
// ❌ Noto'g'ri — string throw
throw "Nimadir noto'g'ri ketdi!";
// Stack trace yo'q — debugging qiyin
```

### ✅ To'g'ri usul:

```javascript
// ✅ Error obyekti throw
throw new Error("Nimadir noto'g'ri ketdi!");
// Stack trace bor — qayerda sodir bo'lgani aniq
```

**Nima uchun:** `Error` obyekti `stack` property beradi — xato qayerda, qaysi funksiyada sodir bo'lganini ko'rsatadi. String da bu ma'lumot yo'q.

---

### ❌ Xato 3: async funksiyaning xatosini ushlamaslik

```javascript
// ❌ Noto'g'ri — async funksiya natijasini kutmaslik
async function riskyAsync() {
  throw new Error("Async xato!");
}
riskyAsync(); // UnhandledPromiseRejection! catch yo'q
```

### ✅ To'g'ri usul:

```javascript
// ✅ catch bilan
riskyAsync().catch(console.error);

// ✅ Yoki try/catch ichida
try {
  await riskyAsync();
} catch (error) {
  console.error(error);
}
```

**Nima uchun:** `async` funksiya **Promise** qaytaradi. Agar Promise reject bo'lsa va hech qayerda catch bo'lmasa — `UnhandledPromiseRejection` warning chiqadi. Node.js da bu process crash qilishi mumkin.

---

### ❌ Xato 4: finally ichida return

```javascript
// ❌ Noto'g'ri — finally ichida return
function getData() {
  try {
    throw new Error("muhim xato!");
  } finally {
    return "default"; // ❌ Error yo'qoldi!
  }
}
console.log(getData()); // "default" — xato hech qachon ko'rinmaydi
```

### ✅ To'g'ri usul:

```javascript
// ✅ finally da faqat cleanup — return yo'q
function getData() {
  try {
    throw new Error("muhim xato!");
  } catch (error) {
    console.error(error);
    return "default"; // catch da return — xavfsiz
  } finally {
    // faqat cleanup: file.close(), connection.end(), etc.
  }
}
```

**Nima uchun:** `finally` ichidagi `return` try/catch dagi **barcha** return va throw larni override qiladi. Xatolar "yo'qoladi" — debugging deyarli mumkin emas.

---

## Amaliy Mashqlar

### Mashq 1: Safe JSON Parse (Oson)

**Savol:** `safeJsonParse(str, fallback)` funksiyasi yarating. JSON parse muvaffaqiyatli bo'lsa natijani, bo'lmasa fallback ni qaytarsin.

<details>
<summary>Javob</summary>

```javascript
function safeJsonParse(str, fallback = null) {
  try {
    return JSON.parse(str);
  } catch (error) {
    if (error instanceof SyntaxError) {
      console.warn("JSON parse xatosi:", error.message);
      return fallback;
    }
    throw error; // boshqa xatolarni qayta throw
  }
}

// Test:
console.log(safeJsonParse('{"name":"Ali"}')); // { name: "Ali" }
console.log(safeJsonParse("invalid", {}));     // {}
console.log(safeJsonParse("null"));            // null (valid JSON)
console.log(safeJsonParse(undefined, []));     // []
```

</details>

---

### Mashq 2: Custom Error Hierarchy (O'rta)

**Savol:** API dasturingiz uchun quyidagi error hierarchy yarating:
- `AppError` (base) — `statusCode`, `isOperational`
- `ValidationError` — `field`, statusCode: 400
- `NotFoundError` — `resource`, statusCode: 404
- `DatabaseError` — statusCode: 500, isOperational: false

<details>
<summary>Javob</summary>

```javascript
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

class ValidationError extends AppError {
  constructor(message, field) {
    super(message, 400);
    this.field = field;
  }
}

class NotFoundError extends AppError {
  constructor(resource = "Resurs") {
    super(`${resource} topilmadi`, 404);
    this.resource = resource;
  }
}

class DatabaseError extends AppError {
  constructor(message, cause) {
    super(message, 500);
    this.isOperational = false; // critical!
    this.cause = cause;
  }
}

// Error handler middleware
function handleError(error) {
  if (error instanceof ValidationError) {
    return { status: 400, body: { error: error.message, field: error.field } };
  }
  if (error instanceof NotFoundError) {
    return { status: 404, body: { error: error.message } };
  }
  if (!error.isOperational) {
    console.error("CRITICAL:", error);
    // Alert yuborish, process restart
  }
  return { status: error.statusCode || 500, body: { error: "Server xatosi" } };
}

// Test:
console.log(handleError(new ValidationError("Email noto'g'ri", "email")));
// { status: 400, body: { error: "Email noto'g'ri", field: "email" } }
console.log(handleError(new NotFoundError("Foydalanuvchi")));
// { status: 404, body: { error: "Foydalanuvchi topilmadi" } }
```

</details>

---

### Mashq 3: Retry with Backoff (O'rta)

**Savol:** `retry(fn, options)` funksiyasi yarating:
- `maxRetries` — necha marta urinish (default: 3)
- `delay` — boshlang'ich kutish ms (default: 1000)
- `backoff` — har safar delay ni ko'paytirish koeffitsienti (default: 2)
- `onRetry(attempt, error)` — callback

<details>
<summary>Javob</summary>

```javascript
async function retry(fn, options = {}) {
  const {
    maxRetries = 3,
    delay = 1000,
    backoff = 2,
    onRetry = () => {}
  } = options;

  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn(attempt);
    } catch (error) {
      lastError = error;

      if (attempt === maxRetries) break;

      const waitTime = delay * Math.pow(backoff, attempt);
      onRetry(attempt + 1, error, waitTime);

      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }

  throw new Error(
    `${maxRetries + 1} urinishdan keyin muvaffaqiyatsiz`,
    { cause: lastError }
  );
}

// Test:
let callCount = 0;
const result = await retry(
  async () => {
    callCount++;
    if (callCount < 3) throw new Error(`Urinish ${callCount} muvaffaqiyatsiz`);
    return "Muvaffaqiyat!";
  },
  {
    maxRetries: 5,
    delay: 100,
    onRetry: (attempt, error, wait) => {
      console.log(`Urinish ${attempt} muvaffaqiyatsiz, ${wait}ms kutilmoqda...`);
    }
  }
);
console.log(result); // "Muvaffaqiyat!" (3-chi urinishda)
```

</details>

---

### Mashq 4: Error Boundary (Qiyin)

**Savol:** `ErrorBoundary` class yarating:
- `wrap(fn)` — funksiyani wrap qilib, xatolarni ushlaydi
- `onError(callback)` — xato handler ro'yxatga olish
- `getErrors()` — barcha xatolar ro'yxatini olish
- Async funksiyalarni ham qo'llab-quvvatlash

<details>
<summary>Javob</summary>

```javascript
class ErrorBoundary {
  constructor() {
    this.errors = [];
    this.handlers = [];
  }

  onError(callback) {
    this.handlers.push(callback);
    return this; // chaining
  }

  _handleError(error, context) {
    const entry = {
      error,
      context,
      timestamp: new Date().toISOString()
    };
    this.errors.push(entry);
    this.handlers.forEach(handler => {
      try {
        handler(entry);
      } catch (handlerError) {
        console.error("Error handler da xato:", handlerError);
      }
    });
  }

  wrap(fn, context = fn.name || "anonymous") {
    const self = this;

    return function(...args) {
      try {
        const result = fn.apply(this, args);

        // Agar Promise qaytarsa — async error ham ushlash
        if (result && typeof result.catch === "function") {
          return result.catch(error => {
            self._handleError(error, context);
            return undefined;
          });
        }

        return result;
      } catch (error) {
        self._handleError(error, context);
        return undefined;
      }
    };
  }

  getErrors() {
    return [...this.errors];
  }

  clear() {
    this.errors = [];
  }
}

// Test:
const boundary = new ErrorBoundary();
boundary.onError(({ error, context, timestamp }) => {
  console.log(`[${timestamp}] ${context}: ${error.message}`);
});

// Sinxron funksiya
const safeParse = boundary.wrap(JSON.parse, "JSON.parse");
safeParse("valid json olmas");  // xato ushlandi, undefined qaytardi
safeParse('{"ok": true}');      // { ok: true }

// Async funksiya
const safeFetch = boundary.wrap(async (url) => {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}, "safeFetch");

await safeFetch("/api/404"); // xato ushlandi

console.log(boundary.getErrors().length); // 2
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Error Types** | SyntaxError, TypeError, ReferenceError, RangeError — har biri alohida muammo |
| **try/catch/finally** | finally DOIM ishlaydi. finally da return yozmang! |
| **Custom Errors** | Error dan extend qilish — `instanceof` bilan tekshirish mumkin |
| **Error Propagation** | Xato catch bo'lmaguncha yuqoriga ko'tariladi |
| **Async Errors** | `.catch()`, `try/catch` + `await`, `Promise.allSettled` |
| **Global Handlers** | `window.onerror`, `unhandledrejection` — oxirgi himoya |
| **Result Type** | `{ ok, data, error }` — throw o'rniga natija qaytarish |
| **Retry** | Exponential backoff — vaqtinchalik xatolar uchun |
| **Circuit Breaker** | Ko'p xatodan keyin — so'rovlarni to'xtatish |
| **Error cause** | `new Error("msg", { cause })` — xato zanjirini saqlash (ES2022) |

> **Keyingi bo'lim:** [21-modern-js.md](21-modern-js.md) — Destructuring, spread/rest, optional chaining, nullish coalescing va boshqa zamonaviy xususiyatlar.

> **Cross-references:** [12-async.md](12-async.md) (Promise, async/await — async error handling), [08-classes.md](08-classes.md) (class, extends — Custom Error hierarchy), [13-event-loop.md](13-event-loop.md) (microtask — unhandled rejection), [19.5-browser-apis.md](19.5-browser-apis.md) (Fetch — network errors, AbortController)
