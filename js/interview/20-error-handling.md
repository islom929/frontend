# Error Handling — Interview Savollari

> **Bo'lim 20** | Error types, try/catch/finally, custom errors, async error handling, error patterns

---

## Nazariy savollar

### 1. JavaScript dagi asosiy Error turlari qaysilar va farqlari nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

JavaScript da barcha xatolar `Error` base class'idan meros oladi. Har bir tur muayyan xato kategoriyasini bildiradi:

| Error Turi | Qachon Sodir Bo'ladi | Misol |
|------------|----------------------|-------|
| `SyntaxError` | Kod sintaktik noto'g'ri — **parse vaqtida** | `JSON.parse("{invalid}")` |
| `ReferenceError` | Mavjud bo'lmagan o'zgaruvchiga murojaat | `console.log(x)` — x declare qilinmagan |
| `TypeError` | Noto'g'ri type da operatsiya | `null.toString()`, `"str"()` |
| `RangeError` | Qiymat chegaradan tashqariga | `new Array(-1)`, cheksiz rekursiya |
| `URIError` | Noto'g'ri URI format | `decodeURIComponent("%")` |
| `AggregateError` | Bir nechta xato birgalikda (ES2021) | `Promise.any()` barcha reject bo'lganda |

```javascript
// Prototype chain — barcha error'lar Error dan meros oladi
const err = new TypeError("xato");
console.log(err instanceof TypeError); // true
console.log(err instanceof Error);     // true — barcha subclass'lar

// SyntaxError — try/catch bilan ushlash mumkin emas (parse vaqtida)
// Lekin JSON.parse() ichidagisini ushlash mumkin:
try {
  JSON.parse("invalid");
} catch (e) {
  console.log(e instanceof SyntaxError); // true — runtime da parse qilgani uchun
}
```

**Deep Dive:**

`SyntaxError` boshqalardan tubdan farq qiladi — u parse vaqtida sodir bo'ladi (kod bajarilmagan). Shuning uchun global try/catch bilan ushlash mumkin emas. Faqat `JSON.parse()` kabi runtime da parse qiladigan funksiyalar ichidagi syntax xatolarni ushlash mumkin.


</details>

### 2. Nima uchun Error throw qilish kerak, string emas? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
// ❌ String throw — stack trace yo'q
try {
  throw "xato!";
} catch (e) {
  console.log(typeof e);    // "string"
  console.log(e.stack);     // undefined — qayerda sodir bo'lgani noma'lum!
  console.log(e.message);   // undefined
  console.log(e instanceof Error); // false
}

// ✅ Error throw — to'liq diagnostika
try {
  throw new Error("xato!");
} catch (e) {
  console.log(typeof e);    // "object"
  console.log(e.stack);     // ✅ Error: xato! at file.js:2:9 ...
  console.log(e.message);   // ✅ "xato!"
  console.log(e instanceof Error); // ✅ true
}
```

`Error` obyektida `stack` property bor — xato qaysi fayl, qaysi qator, qaysi funksiyalar zanjirida sodir bo'lganini ko'rsatadi. String'da bu ma'lumot yo'q — production da debugging deyarli mumkin emas. `instanceof` tekshiruvi ham faqat Error obyektlarida ishlaydi.


</details>

### 3. Custom Error class qanday yaratiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`extends Error` bilan meros olish, `super(message)` chaqirish, `this.name` ni o'rnatish:

```javascript
class ValidationError extends Error {
  constructor(message, field) {
    super(message);                      // ① Error constructor — message, stack
    this.name = this.constructor.name;   // ② "ValidationError" — stack trace da ko'rinadi
    this.field = field;                  // ③ Qo'shimcha ma'lumot
  }
}

class NotFoundError extends Error {
  constructor(resource, id) {
    super(`${resource} #${id} topilmadi`);
    this.name = this.constructor.name;
    this.statusCode = 404;
    this.resource = resource;
  }
}

// instanceof zanjir tekshiruv
const err = new ValidationError("Email noto'g'ri", "email");
console.log(err instanceof ValidationError); // true
console.log(err instanceof Error);           // true
console.log(err.name);    // "ValidationError"
console.log(err.field);   // "email"
console.log(err.stack);   // ✅ to'liq stack trace — Error dan meros
```

`this.name = this.constructor.name` pattern har qanday subclass uchun avtomatik to'g'ri nom beradi. `super(message)` chaqirilganda `Error.captureStackTrace()` (V8) ham ishlaydi — stack trace to'g'ri joyni ko'rsatadi.

**Deep Dive:**

ES6 dan oldin `Error` subclass qilish muammoli edi — `Object.setPrototypeOf` kerak bo'lardi. ES6 `class`/`extends` prototype chain ni avtomatik to'g'ri quradi:

```
ValidationError instance → ValidationError.prototype → Error.prototype → Object.prototype
```


</details>

### 4. Error.cause nima va qanday ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`Error.cause` (ES2022) — xatoning asl sababini saqlash. Ikkinchi argument sifatida `{ cause }` options obyekti beriladi:

```javascript
async function loadConfig() {
  try {
    const data = await fs.readFile("config.json", "utf8");
    return JSON.parse(data);
  } catch (error) {
    // Asl xatoni cause sifatida saqlash
    throw new Error("Konfiguratsiya yuklanmadi", { cause: error });
  }
}

try {
  await loadConfig();
} catch (error) {
  console.log(error.message);       // "Konfiguratsiya yuklanmadi"
  console.log(error.cause.message); // "ENOENT: no such file" yoki "Unexpected token..."
  // Zanjir: yuqori darajadagi xato → asl sabab
}
```

Bu nima uchun kerak: ko'p qatlamli dasturlarda past darajadagi texnik xato (ENOENT, JSON parse, ECONNREFUSED) yuqori darajadagi xatoga aylantiriladi. `cause` yo'q bo'lganda asl sabab yo'qoladi. `cause` bilan xatolar zanjiri to'liq saqlanadi — debugging osonlashadi.


</details>

### 5. unhandledrejection eventi nima va qanday ishlatiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

`unhandledrejection` — Promise reject bo'lib, hech qayerda `.catch()` yoki `try/catch` bilan ushlanmaganda browser/Node.js tomonidan trigger qilinadigan event. Bu xatolarni global darajada ushlashning oxirgi himoya qatlami.

```javascript
// Browser
window.addEventListener("unhandledrejection", (event) => {
  console.error("Ushlanmagan rejection:", event.reason);
  // event.reason — reject qilingan qiymat (odatda Error)
  // event.promise — reject bo'lgan Promise

  // Monitoring service ga yuborish
  sendToSentry({
    error: event.reason,
    stack: event.reason?.stack
  });

  event.preventDefault(); // default console warning'ni to'xtatish
});

// Node.js
process.on("unhandledRejection", (reason, promise) => {
  console.error("Ushlanmagan rejection:", reason);
  // Node.js 15+ da default: process exit
});
```

Bu **asosiy** error handling o'rniga emas, qo'shimcha sifatida ishlatiladi. Har bir Promise uchun `.catch()` yoki `try/catch` yozish — asosiy qoida. `unhandledrejection` faqat o'tkazib yuborilgan xatolarni tutish uchun.

**Deep Dive:**

Event loop'da microtask queue'dagi barcha microtask'lar tugagandan keyin, agar biror Promise rejected holatda va hech qanday rejection handler yo'q bo'lsa — engine `unhandledrejection` event'ini dispatch qiladi. Agar keyinroq `.catch()` qo'shilsa — `rejectionhandled` eventi trigger bo'ladi.


</details>

### 6. Re-throwing pattern nima? Qachon ishlatiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Re-throwing — catch blokda faqat boshqara oladigan xatolarni ushlash, qolganlarini qayta `throw` qilish. Bu eng muhim error handling pattern'laridan biri.

```javascript
function processInput(data) {
  try {
    return JSON.parse(data);
  } catch (error) {
    if (error instanceof SyntaxError) {
      // ✅ Faqat JSON syntax xatosini boshqaramiz — biz bilgan xato
      console.warn("Noto'g'ri JSON format:", data);
      return null;
    }
    // ❌ Boshqa barcha xatolarni qayta throw — bu bizning mas'uliyatimiz emas
    throw error;
  }
}
```

**Nima uchun:** Agar barcha xatolarni catch qilib, hammasini bir xil boshqarsangiz — kutilmagan xatolar (TypeError, RangeError) yashirinib qoladi. Faqat bilgan va boshqara oladigan xatolarni ushlash — qolganlarini yuqoriga uzatish kerak. Bu "catch what you can handle, re-throw what you can't" prinsipi.


</details>

### 7. Result pattern nima? throw dan qanday farq qiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Result pattern — `try/catch` va `throw` o'rniga natija obyekti qaytarish: `{ ok: true, data }` yoki `{ ok: false, error }`. Go tilida standart, JavaScript da kutilgan xatolar uchun ishlatiladi.

```javascript
// throw pattern — exception-based
function divide(a, b) {
  if (b === 0) throw new RangeError("Nolga bo'lish mumkin emas");
  return a / b;
}
try {
  const result = divide(10, 0);
} catch (error) { /* ... */ }

// Result pattern — explicit
function safeDivide(a, b) {
  if (b === 0) return { ok: false, error: "Nolga bo'lish mumkin emas" };
  return { ok: true, data: a / b };
}
const { ok, data, error } = safeDivide(10, 0);
if (!ok) console.log(error);
```

| | `throw` | Result Pattern |
|---|---|---|
| Control flow | Implicit — catch ga sakraydi | Explicit — if/else |
| Performance | try/catch overhead (minimal) | Oddiy object qaytarish |
| Composability | try/catch nesting | Oddiy destructuring |
| Qachon | Kutilmagan xatolar (bug, crash) | Kutilgan xatolar (validatsiya, lookup) |
| TypeScript | Type narrowing qiyin | Discriminated union — type-safe |

**Deep Dive:**

Performance: zamonaviy engine'larda try/catch overhead minimal (V8 TurboFan optimized). Lekin `throw` stack trace yaratadi — bu costly operatsiya. Agar xato **tez-tez** sodir bo'lsa (validatsiya loop ichida) — Result pattern tezroq. Agar xato **kamdan-kam** sodir bo'lsa — `throw` yaxshi.


</details>

### 8. Operational vs Programmer errors farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

**Operational errors** — dastur to'g'ri yozilgan, lekin tashqi omillar sababli xato: network timeout, fayl topilmadi, user input noto'g'ri, rate limit, disk to'ldi. Bular **kutilgan** va **boshqarish mumkin** — retry, fallback, foydalanuvchiga xabar.

**Programmer errors** — koddagi bug: `undefined` ning property'si o'qish, noto'g'ri argument tipi, logic error, off-by-one. Bular **kutilmagan** — dasturni to'xtatish va kodni tuzatish kerak.

```javascript
class AppError extends Error {
  constructor(message, statusCode, isOperational = true) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = isOperational;
  }
}

// Operational — dastur davom etishi mumkin
throw new AppError("Foydalanuvchi topilmadi", 404, true);

// Programmer — critical bug
const err = new AppError("null reference", 500, false);
// isOperational: false → dasturni restart qilish kerak bo'lishi mumkin
```

| | Operational | Programmer |
|---|---|---|
| Sabab | Tashqi omil | Kod xatosi (bug) |
| Kutilganmi | Ha | Yo'q |
| Yechim | Retry, fallback, xabar | Kodni tuzatish |
| Dastur | Davom etishi mumkin | To'xtatish tavsiya qilinadi |
| Misol | 404, timeout, validatsiya | TypeError, null reference |

**Deep Dive:**

Node.js da `uncaughtException` dan keyin `process.exit(1)` tavsiya qilinadi — chunki programmer error application state'ni noaniq holatga olib kelishi mumkin. PM2, Docker, Kubernetes kabi process manager'lar avtomatik restart qiladi. Operational error'lar uchun esa dastur normal davom etishi kerak.


</details>

### 9. Circuit Breaker pattern'ini tushuntiring va implement qiling [Senior]

<details>
<summary><strong>Javob</strong></summary>

Circuit Breaker — tashqi servisga ko'p marta muvaffaqiyatsiz so'rov yuborilgandan keyin yangi so'rovlarni **vaqtincha to'xtatish** pattern'i. Servisni haddan tashqari yuklashni va cascading failure'ni oldini oladi.

3 ta holat:
- **CLOSED** — normal ishlaydi, so'rovlar o'tadi
- **OPEN** — xatolar threshold'dan oshdi, barcha so'rovlar darhol rad etiladi
- **HALF_OPEN** — timeout o'tdi, bitta sinov so'rov yuboriladi

```
  ┌─────────┐   xatolar < threshold   ┌─────────┐
  │ CLOSED  │◄────────────────────────│HALF_OPEN│
  │(normal) │                         │(sinov)  │
  └────┬────┘                         └────┬────┘
       │ xatolar >= threshold              │ xato
       ▼                                   ▼
  ┌─────────┐   timeout o'tdi        ┌─────────┐
  │  OPEN   │────────────────────────►│HALF_OPEN│
  │(bloklangan)│                      └─────────┘
  └─────────┘
```

```javascript
class CircuitBreaker {
  constructor(fn, { threshold = 5, timeout = 30000 } = {}) {
    this.fn = fn;
    this.threshold = threshold;
    this.timeout = timeout;
    this.failures = 0;
    this.state = "CLOSED";
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
      this.failures = 0;
      this.state = "CLOSED";
      return result;
    } catch (error) {
      this.failures++;
      if (this.failures >= this.threshold) {
        this.state = "OPEN";
        this.nextAttempt = Date.now() + this.timeout;
      }
      throw error;
    }
  }
}

// Ishlatish: 3 xatodan keyin 10s to'xtatish
const breaker = new CircuitBreaker(
  (url) => fetch(url).then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); }),
  { threshold: 3, timeout: 10000 }
);
```

**Deep Dive:**

Real production da qo'shimcha narsalar kerak: health check endpoint, sliding window (so'nggi N sekunddagi xatolar), half-open da faqat bitta so'rov o'tkazish (semaphore), logging/metrics. Netflix Hystrix (Java) va opossum (Node.js) kutubxonalari to'liq implementatsiya beradi.


</details>

### 10. Graceful degradation pattern'ini tushuntiring [Senior]

<details>
<summary><strong>Javob</strong></summary>

Graceful degradation — asosiy data manba ishlamasa bir necha bosqichda fallback'larga o'tish. Foydalanuvchi har doim natija oladi — to'liq yoki qisman.

```javascript
async function getUserData(userId) {
  // 1-bosqich: API
  try {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const data = await response.json();
    // Muvaffaqiyatli — cache'ni yangilash
    localStorage.setItem(`user_${userId}`, JSON.stringify(data));
    return data;
  } catch (apiError) {
    console.warn("API:", apiError.message);
  }

  // 2-bosqich: Cache
  try {
    const cached = localStorage.getItem(`user_${userId}`);
    if (cached) {
      const data = JSON.parse(cached);
      data._fromCache = true; // UI da "eskirgan ma'lumot" ko'rsatish uchun
      return data;
    }
  } catch (cacheError) {
    console.warn("Cache:", cacheError.message);
  }

  // 3-bosqich: Default
  return {
    id: userId,
    name: "Noma'lum foydalanuvchi",
    avatar: "/images/default-avatar.png",
    _fallback: true
  };
}
```

Pattern'ning 3 ta prinsipi: (1) Har bir bosqich o'z try/catch'ida — bitta bosqich xatosi boshqasiga ta'sir qilmaydi. (2) Muvaffaqiyatli natijada cache yangilanadi — keyingi xatoda yangi cache ishlatiladi. (3) Fallback natija har doim qaytadi — foydalanuvchi "500 error" ko'rmaydi.

**Deep Dive:** Resilience engineering'da bu pattern "bulkhead" prinsipi bilan bog'liq — har bir bosqich izolyatsiyalangan, bitta bosqich xatosi boshqalarni cascading failure'ga olib kelmaydi. Netflix va Amazon kabi kompaniyalar multi-tier fallback (primary API → replica → cache → static default) ishlatadi. Browser'da Service Worker cache layer ham shu pattern'ning infratuzilma darajasidagi implementatsiyasi.

</details>

### 11. window.onerror vs addEventListener("error") farqi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

| | `window.onerror` | `addEventListener("error")` |
|---|---|---|
| Syntax | `window.onerror = fn` | `window.addEventListener("error", fn)` |
| Ko'p handler | Yo'q (overwrite) | Ha (additive) |
| Resurs xatolari | Yo'q | Ha (img, script yuklanmadi) |
| Parameters | `(message, source, lineno, colno, error)` | `ErrorEvent` obyekti |
| Default to'xtatish | `return true` | `event.preventDefault()` |

```javascript
// onerror — faqat runtime xatolar
window.onerror = (message, source, lineno, colno, error) => {
  console.log("onerror:", message, "at", source, lineno);
  return true; // default console error ko'rsatmaydi
};

// addEventListener — runtime + resurs xatolari
window.addEventListener("error", (event) => {
  if (event.target !== window) {
    // Resurs yuklash xatosi (img, script, link)
    console.log("Resurs yuklanmadi:", event.target.src);
  } else {
    // Runtime xato
    console.log("Runtime xato:", event.error?.message);
  }
}, true); // ← true (capture) — resurs xatolarini ushash uchun
```

Resurs xatolari (img, script) bubble qilmaydi — shuning uchun **capture phase** da (`true` argument bilan) ushlash kerak.


</details>

### 12. Promise.allSettled vs Promise.all — error handling farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

| | `Promise.all` | `Promise.allSettled` |
|---|---|---|
| Xato bo'lganda | **Darhol** reject — qolgan natijalar yo'qoladi | Hammasi tugashini kutadi |
| Natija | `[val1, val2, ...]` | `[{status, value/reason}, ...]` |
| Qachon | Barcha natija kerak — bitta xato = fail | Har bir natija alohida kerak |

```javascript
const promises = [
  fetch("/api/users").then(r => r.json()),
  fetch("/api/INVALID").then(r => { if (!r.ok) throw new Error("404"); return r.json(); }),
  fetch("/api/posts").then(r => r.json())
];

// ❌ Promise.all — bitta xato = hammasi yo'qoladi
try {
  const [users, _, posts] = await Promise.all(promises);
} catch (error) {
  // "404" — faqat birinchi xato. users va posts natijasi YO'QOLDI!
}

// ✅ Promise.allSettled — barcha natijalar saqlanadi
const results = await Promise.allSettled(promises);
// [
//   { status: "fulfilled", value: [...users] },
//   { status: "rejected", reason: Error("404") },
//   { status: "fulfilled", value: [...posts] }
// ]

const succeeded = results.filter(r => r.status === "fulfilled").map(r => r.value);
const failed = results.filter(r => r.status === "rejected").map(r => r.reason);
```


</details>

### 13. Error handling best practices ro'yxatini ayting [Senior]

<details>
<summary><strong>Javob</strong></summary>

1. **Doim `Error` throw qiling** — string, number emas. Stack trace uchun.

2. **Faqat boshqara oladigan xatolarni ushlang** — qolganlarini re-throw qiling.

3. **Bo'sh catch yozmang** — kamida `console.error`. Xato yutish — debugging ni o'ldiradi.

4. **finally da return yozmang** — throw/return override bo'ladi.

5. **Har bir async operatsiyada catch bo'lsin** — `.catch()` yoki `try/catch` + `await`.

6. **Error.cause ishlating** — xato zanjirini saqlash uchun (ES2022).

7. **Custom error class'lar yarating** — `instanceof` bilan aniq tekshirish, statusCode, qo'shimcha field'lar.

8. **Fail fast** — argumentlarni boshida tekshiring (guard clauses).

9. **Operational vs Programmer error ajrating** — operational → recovery, programmer → fix code.

10. **Global handler qo'ying** — `unhandledrejection`, `window.onerror` — monitoring uchun.

```javascript
// Amalda — Express error handler
app.use((error, req, res, next) => {
  // 1. Logging
  console.error(error.stack);

  // 2. Monitoring
  sendToSentry(error);

  // 3. Operational vs Programmer
  if (!error.isOperational) {
    // Critical — alert + graceful shutdown
    process.exit(1);
  }

  // 4. Client ga javob
  res.status(error.statusCode || 500).json({
    error: error.isOperational ? error.message : "Server xatosi"
  });
});
```

**Deep Dive:** Express'da error-handling middleware'ni aniqlash uchun 4 ta parameter bilan `(error, req, res, next)` imzo ishlatiladi — Express `function.length` ni tekshirib error middleware'ni oddiy middleware'dan ajratadi. Node.js 15+ da `unhandledRejection` default `throw` qiladi — jarayon tushadi. `Error.captureStackTrace(this, this.constructor)` (V8-specific) custom error'larda stack trace'ni constructor frame'dan boshlaydi — debug uchun toza stack trace beradi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Output nima? try/catch/finally execution order [Middle]


```javascript
function test() {
  try {
    console.log("A");
    throw new Error("xato");
    console.log("B");
  } catch (error) {
    console.log("C");
    return "catch";
  } finally {
    console.log("D");
  }
  console.log("E");
}

console.log(test());
```

<details>
<summary><strong>Javob</strong></summary>

```
A
C
D
catch
```

`"A"` — try boshi. throw sodir bo'ladi — `"B"` o'tkaziladi. `"C"` — catch ishlaydi. `return "catch"` — **lekin avval** finally ishlaydi. `"D"` — finally DOIM ishlaydi, hatto return bo'lsa ham. Keyin catch'dagi `return "catch"` qaytadi. `"E"` — hech qachon ishlamaydi, chunki catch'da `return` bor.


</details>

### 2. Output nima? finally da return [Middle+]


```javascript
function tricky() {
  try {
    throw new Error("xato!");
  } catch (error) {
    console.log("catch:", error.message);
    return 1;
  } finally {
    return 2;
  }
}

console.log(tricky());
```

<details>
<summary><strong>Javob</strong></summary>

```
catch: xato!
2
```

`finally` ichidagi `return` catch'dagi `return 1` ni **override** qiladi. finally completion har doim ustun — try/catch'dagi return va hatto throw ham yo'qoladi. **QOIDA: finally ichida hech qachon return yozmang!**

```javascript
// Yana bir xavfli holat — throw ham yo'qoladi:
function swallow() {
  try {
    throw new Error("muhim!");
  } finally {
    return "ok"; // Error yo'qoldi — tashqariga hech qachon chiqmaydi!
  }
}
console.log(swallow()); // "ok" — xato yutildi
```


</details>

### 3. Output nima? try/catch va async [Middle+]


```javascript
function test() {
  try {
    setTimeout(() => {
      throw new Error("async xato");
    }, 0);
    console.log("A");
  } catch (error) {
    console.log("B:", error.message);
  }
  console.log("C");
}

test();
```

<details>
<summary><strong>Javob</strong></summary>

```
A
C
// Keyin: Uncaught Error: async xato (console error)
```

`try/catch` faqat **sinxron** throw'ni ushlaydi. `setTimeout` callback boshqa execution context'da (keyingi event loop tick'da) bajariladi — o'sha paytda try/catch allaqachon tugagan. Shuning uchun `"A"` va `"C"` chiqadi, keyin callback ishlaganda xato ushlanmaydi va global error bo'ladi.

```javascript
// ✅ To'g'ri usul — callback ichida try/catch
setTimeout(() => {
  try {
    throw new Error("async xato");
  } catch (error) {
    console.log("Ushlandi:", error.message); // ✅
  }
}, 0);

// ✅ Yoki Promise + await
try {
  await new Promise((_, reject) => {
    setTimeout(() => reject(new Error("async xato")), 0);
  });
} catch (error) {
  console.log("Ushlandi:", error.message); // ✅
}
```


</details>

### 4. Bu kodda nima xato va qanday tuzatiladi? [Middle]

```javascript
async function fetchData() {
  const response = await fetch("/api/data");
  const data = await response.json();
  return data;
}

// Chaqirish
fetchData();
console.log("Done");
```

<details>
<summary><strong>Javob</strong></summary>

**3 ta xato bor:**

1. **HTTP error tekshiruvi yo'q** — `fetch()` faqat network error da reject bo'ladi, 404/500 HTTP status'lar uchun reject qilmaydi.

2. **Error handling yo'q** — `async` funksiya Promise qaytaradi, lekin `.catch()` yoki `try/catch` yo'q. Agar xato bo'lsa — `UnhandledPromiseRejection`.

3. **Natijani ishlatish mumkin emas** — `await` yo'q, `console.log("Done")` darhol ishlaydi, data hali kelmagan.

```javascript
// ✅ Tuzatilgan versiya
async function fetchData() {
  const response = await fetch("/api/data");

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return response.json();
}

try {
  const data = await fetchData();
  console.log("Data:", data);
  console.log("Done");
} catch (error) {
  console.error("Xato:", error.message);
}
```


</details>

### 5. Output nima? Error propagation [Middle+]


```javascript
function a() {
  console.log("a start");
  b();
  console.log("a end");
}

function b() {
  console.log("b start");
  c();
  console.log("b end");
}

function c() {
  console.log("c start");
  throw new Error("c xato");
  console.log("c end");
}

try {
  a();
} catch (error) {
  console.log("caught:", error.message);
}
console.log("done");
```

<details>
<summary><strong>Javob</strong></summary>

```
a start
b start
c start
caught: c xato
done
```

`c()` da throw sodir bo'ladi — `"c end"` ishlamaydi. `b()` da catch yo'q — `"b end"` ishlamaydi, xato yuqoriga. `a()` da catch yo'q — `"a end"` ishlamaydi, xato yana yuqoriga. Global try/catch'da ushlanadi — `"caught: c xato"`. Keyin normal execution davom etadi — `"done"`. Bu **stack unwinding** — throw dan keyin eng yaqin catch'gacha barcha frame'lar skip qilinadi.


</details>

### 6. retry funksiyasini exponential backoff bilan implement qiling [Middle+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
async function retry(fn, { maxRetries = 3, delay = 1000, backoff = 2 } = {}) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn(attempt);
    } catch (error) {
      lastError = error;
      if (attempt === maxRetries) break;

      const waitTime = delay * Math.pow(backoff, attempt);
      // Jitter qo'shish — bir vaqtda ko'p client qayta urinmasligi uchun
      const jitter = waitTime * Math.random() * 0.1;
      await new Promise(r => setTimeout(r, waitTime + jitter));
    }
  }

  throw new Error(`${maxRetries + 1} urinish muvaffaqiyatsiz`, { cause: lastError });
}

// Kutish vaqtlari (delay=1000, backoff=2):
// Urinish 0: darhol
// Urinish 1: ~1000ms kutish
// Urinish 2: ~2000ms kutish
// Urinish 3: ~4000ms kutish
```

Exponential backoff server'ni haddan tashqari yuklashni oldini oladi. Jitter (tasodifiy kechikish) qo'shish muhim — agar 1000 ta client bir vaqtda xato olsa va 1s keyin hammasi qayta urinsa — server yana overload bo'ladi. Jitter bilan so'rovlar tarqaladi.


</details>

### 7. Output nima? Promise error handling [Middle+]


```javascript
Promise.resolve(1)
  .then(val => {
    console.log("A:", val);
    throw new Error("xato");
  })
  .then(val => {
    console.log("B:", val);
  })
  .catch(err => {
    console.log("C:", err.message);
    return 42;
  })
  .then(val => {
    console.log("D:", val);
  });
```

<details>
<summary><strong>Javob</strong></summary>

```
A: 1
C: xato
D: 42
```

`"A: 1"` — birinchi then, val = 1. throw sodir bo'ladi. `"B"` — **o'tkaziladi**, chunki xato bo'ldi — eng yaqin `.catch()`'ga boradi. `"C: xato"` — catch xatoni ushladi va `42` qaytardi. `"D: 42"` — catch'dan keyin zanjir davom etadi, catch'ning return qiymati keyingi then'ga o'tadi.

**MUHIM**: `.catch()` xatoni **recovery** qiladi — catch'dan keyin zanjir normal davom etadi. Agar catch ichida yana throw qilsangiz — keyingi catch ga o'tadi.


</details>

### 8. Output nima? `return await` vs `return` try/catch ichida [Senior]


```javascript
async function fetchUser() {
  throw new Error("network");
}

async function getUserNoAwait() {
  try {
    return fetchUser();
  } catch (error) {
    return "caught-no-await";
  }
}

async function getUserWithAwait() {
  try {
    return await fetchUser();
  } catch (error) {
    return "caught-with-await";
  }
}

getUserNoAwait().then(r => console.log("A:", r), e => console.log("A reject:", e.message));
getUserWithAwait().then(r => console.log("B:", r));
```

<details>
<summary><strong>Javob</strong></summary>

```
A reject: network
B: caught-with-await
```

`return fetchUser()` — rejected Promise to'g'ridan-to'g'ri caller'ga qaytadi. Funksiya ichidagi `try/catch` faqat **joriy execution context**'dagi sync throw'ni ushlaydi — qaytarilgan Promise allaqachon funksiyadan tashqarida bo'ladi. `return await fetchUser()` — `await` rejected promise'ni sync throw'ga aylantiradi, throw hali funksiya ichida sodir bo'ladi va catch ushlaydi.

**Qoida:** try/catch ichidan Promise qaytarayotgan bo'lsangiz — **doim `return await`** ishlating. Kichik performance cost (qo'shimcha microtask) bor, lekin aniq error handling muhim. ESLint'da `@typescript-eslint/return-await: "in-try-catch"` rule shu pattern'ni tekshiradi.


</details>

### 9. AggregateError nima va qachon throw bo'ladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

`AggregateError` (ES2021) — bir nechta xatoni bitta obyektda jamlash uchun mo'ljallangan Error subclass'i. `errors` property'sida xatolar array'ini saqlaydi. Asosan `Promise.any()` tomonidan throw qilinadi — agar barcha promise'lar reject bo'lsa.

```javascript
// Promise.any — kamida bittasi resolve bo'lishi kutiladi
try {
  const result = await Promise.any([
    Promise.reject(new TypeError("type xato")),
    Promise.reject(new RangeError("range xato")),
    Promise.reject(new Error("oddiy xato"))
  ]);
} catch (error) {
  console.log(error instanceof AggregateError); // true
  console.log(error.errors.length); // 3
  console.log(error.errors[0] instanceof TypeError); // true
  console.log(error.message); // "All promises were rejected"
}

// Manual yaratish — bir nechta validation xatolarini birga qaytarish
function validateForm(data) {
  const errors = [];
  if (!data.email) errors.push(new ValidationError("Email kerak", "email"));
  if (!data.password) errors.push(new ValidationError("Parol kerak", "password"));
  if (errors.length > 0) {
    throw new AggregateError(errors, "Forma validatsiyadan o'tmadi");
  }
}
```

**Farqi `Promise.all()` dan:** `Promise.all` birinchi rejection'da darhol reject qiladi va xatoni `error` sifatida beradi (bitta). `Promise.any` **barcha** rejection'larni jamlab `AggregateError` qaytaradi — har birining sababi `errors` array'da.

**Deep Dive:** `AggregateError` constructor signature: `new AggregateError(errors, message?, options?)`. `options.cause` ES2022 dan keyin qo'llanadi — `Error` constructor bilan bir xil. `errors` argument iterable bo'lishi kerak (array, Set, generator). Promise.any spec algoritmi (Promise Combinators) bir-bir resolve/reject'ni kuzatadi, oxirgisi reject bo'lganda yangi AggregateError yaratib reject qiladi.


</details>

### 10. Error.stack qanday capture qilinadi va Error.captureStackTrace nima qiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

`Error.stack` — ECMAScript spec'da rasman yo'q (non-standard), lekin V8, SpiderMonkey, JavaScriptCore — barcha zamonaviy engine'lar qo'llaydi. `new Error()` paytida engine joriy call stack'ning snapshot'ini saqlaydi va `.stack` property orqali string sifatida ochib beradi.

```javascript
function a() { return b(); }
function b() { return c(); }
function c() { return new Error("test"); }

const err = a();
console.log(err.stack);
// Error: test
//     at c (file.js:3:24)
//     at b (file.js:2:24)
//     at a (file.js:1:24)
//     at <anonymous> ...
```

**V8 lazy stack trace** — performance optimizatsiyasi: `new Error()` paytida stack darhol format qilinmaydi, faqat internal pointer saqlanadi. `.stack` property birinchi marta o'qilganda string'ga aylantiriladi va cache qilinadi. Agar error catch qilinib stack hech qachon o'qilmasa — formatting overhead butunlay yo'q.

`Error.captureStackTrace(target, constructorOpt)` — **V8-only** API. `constructorOpt` argument'iga berilgan funksiya va undan keyingi frame'lar stack'dan **olib tashlanadi**. Custom error class'lar va assert utility'lar uchun foydali — foydalanuvchi stack'da utility frame'larini ko'rmaydi:

```javascript
class AppError extends Error {
  constructor(message) {
    super(message);
    this.name = "AppError";
    // Stack trace'dan AppError constructor'ni olib tashlash
    Error.captureStackTrace?.(this, AppError);
  }
}

function assert(condition, message) {
  if (!condition) {
    const error = new Error(message);
    // assert frame'ni stack'dan olib tashlash
    Error.captureStackTrace?.(error, assert);
    throw error;
  }
}
```

`Error.stackTraceLimit` (V8) — default 10 frame. `Infinity` qilib o'zgartirish mumkin, lekin performance ta'siri bor. Firefox/Safari'da bu property yo'q.

**Deep Dive:** Stack format engine'ga bog'liq: V8 — `    at functionName (file:line:col)`, Firefox — `functionName@file:line:col`. Sentry/LogRocket'lar har engine uchun alohida parser ishlatadi. Source maps (`.map` fayl) production'da minified kod stack trace'ni original kodga map qiladi — DevTools avtomatik bog'laydi, lekin production'da source map'ni publik qilmaslik kerak (kod ochiladi).


</details>

### 11. Error obyektini JSON.stringify qilganda nima uchun bo'sh obyekt? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

`Error.prototype.message`, `name`, `stack` — barchasi **non-enumerable** property'lar. `JSON.stringify` faqat enumerable own property'larni serialize qiladi (`EnumerableOwnPropertyNames` abstract operation) — shuning uchun Error obyekti uchun natija `"{}"` bo'ladi:

```javascript
const error = new Error("Yuklash xatosi");
console.log(JSON.stringify(error)); // "{}" — bo'sh!

// Sabab: built-in property'lar non-enumerable
console.log(Object.keys(error)); // [] — bo'sh
console.log(Object.getOwnPropertyDescriptor(Error.prototype, "message"));
// { writable: true, enumerable: false, configurable: true }

// ✅ Yechim 1: Custom serializer
function serializeError(error) {
  if (!error) return null;
  const result = {
    name: error.name,
    message: error.message,
    stack: error.stack,
  };
  // Custom enumerable property'lar (statusCode, code, ...)
  for (const key of Object.keys(error)) {
    result[key] = error[key];
  }
  // ES2022 cause chain — recursive
  if (error.cause !== undefined) {
    result.cause = serializeError(error.cause);
  }
  return result;
}

console.log(JSON.stringify(serializeError(error), null, 2));
// {
//   "name": "Error",
//   "message": "Yuklash xatosi",
//   "stack": "Error: Yuklash xatosi\n    at ..."
// }

// ✅ Yechim 2: JSON.stringify replacer
JSON.stringify(error, ["name", "message", "stack", "cause"]);
// '{"name":"Error","message":"Yuklash xatosi","stack":"..."}'
```

**Production muhim:** Logger, monitoring servislar (Sentry, Datadog) Error obyektni tanib avtomatik to'g'ri serialize qiladi. Lekin o'z logging kodingizda Error'ni to'g'ridan-to'g'ri `JSON.stringify` ga bermang — custom serializer orqali o'tkazing. Aks holda log'larda bo'sh obyekt'lar paydo bo'ladi va incident debug qiyinlashadi.


</details>

### 12. Bu kodda nima xato? finally va resource leak [Middle]

```javascript
async function processFile(path) {
  const connection = await openConnection();
  const data = await connection.read(path);
  connection.close();
  return JSON.parse(data);
}
```

<details>
<summary><strong>Javob</strong></summary>

**Xato:** Agar `connection.read()` yoki `JSON.parse()` throw qilsa — `connection.close()` **chaqirilmaydi**. Bu **resource leak** — connection pool exhaust bo'lishi va serverni crash qilishi mumkin.

```javascript
// ✅ Tuzatilgan: try/finally bilan cleanup garantiyalanadi
async function processFile(path) {
  const connection = await openConnection();
  try {
    const data = await connection.read(path);
    return JSON.parse(data);
  } finally {
    connection.close(); // ✅ DOIM ishlaydi — xato bo'lsa ham
  }
}
```

`finally` blok **doim** ishlaydi: try muvaffaqiyatli tugasa ham, throw bo'lsa ham, `return` ishlasa ham. Cleanup (file close, connection release, timer clear, mutex release) uchun ideal — `catch` shart emas (xato propagate bo'lib davom etadi).

**Boshqa pattern'lar:**
- **using statement** (ES2024+ Stage 3, hozircha barcha runtime'larda yo'q): `using connection = openConnection();` — block tugaganda avtomatik `[Symbol.dispose]()` chaqiriladi.
- **Wrapper pattern**: `withConnection(async (conn) => {...})` — connection ochish/yopish wrapper ichida.

**Anti-pattern — finally ichida throw qilish:** Agar `connection.close()` o'zi throw qilsa — bu xato `try` ichidagi original xatoni **override** qiladi. Production-ready kod:

```javascript
try {
  // ...
} finally {
  try { connection.close(); } catch (closeError) {
    console.error("Close failed:", closeError);
    // Original xatoni saqlash — close xatosini faqat log qilish
  }
}
```


</details>
