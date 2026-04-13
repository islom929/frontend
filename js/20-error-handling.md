# Bo'lim 20: Error Handling

> Error handling — dastur bajarilishi vaqtida yuzaga keladigan kutilgan va kutilmagan xatolarni aniqlash, ushlash va boshqarish mexanizmi.

---

## Mundarija

- [Error Types](#error-types)
- [Error Object](#error-object)
- [Error cause (ES2022)](#error-cause-es2022)
- [try/catch/finally](#trycatchfinally)
- [throw Statement](#throw-statement)
- [Custom Error Classes](#custom-error-classes)
- [Error Propagation](#error-propagation)
- [Async Error Handling](#async-error-handling)
- [Global Error Handlers](#global-error-handlers)
- [Error Handling Patterns](#error-handling-patterns)
- [Stack Traces va Debugging](#stack-traces-va-debugging)
- [Console API](#console-api)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Error Types

### Nazariya

JavaScript da barcha xatolar `Error` base class'idan meros oladigan built-in error turlari bilan ifodalanadi. Har bir tur muayyan xato kategoriyasini bildiradi — bu `catch` blokda xato turini `instanceof` bilan aniqlash va har bir turga mos javob berish imkonini beradi. Engine ma'lum xatolarni avtomatik throw qiladi (runtime errors), ba'zilarini esa dasturchi o'zi throw qiladi (programmatic errors).

**SyntaxError** — kod sintaktik jihatdan noto'g'ri yozilganda. Bu xato **parse vaqtida** sodir bo'ladi — ya'ni kod hali bajarilmagan, engine kodni o'qiyotgan paytda. `try/catch` bilan ushlash mumkin emas (chunki kod umuman ishga tushmaydi), lekin `eval()` yoki `JSON.parse()` ichidagi syntax xatolarni ushlash mumkin — chunki ular runtime da parse qiladi.

**ReferenceError** — mavjud bo'lmagan o'zgaruvchiga murojaat qilinganda. Engine scope chain bo'ylab o'zgaruvchini qidirib, global scope'gacha yetib topmaganda bu xatoni throw qiladi. `typeof` operatori istisno — u mavjud bo'lmagan o'zgaruvchi uchun `"undefined"` qaytaradi, xato bermaydi.

**TypeError** — qiymat kutilgan type'da bo'lmaganda yoki noto'g'ri type'da operatsiya bajarilganda: `null`/`undefined` ning property'sini o'qish, funksiya bo'lmagan narsani chaqirish, `const` ga qayta assign qilish, read-only property'ga yozish.

**RangeError** — qiymat ruxsat etilgan oraliqdan tashqariga chiqqanda: manfiy array uzunlik, `toFixed(200)` (0-100 orasida bo'lishi kerak), cheksiz rekursiya (Maximum call stack size exceeded).

**URIError** — `decodeURIComponent()`, `encodeURI()` kabi URI funksiyalari noto'g'ri formatdagi string bilan chaqirilganda.

**EvalError** — `eval()` bilan bog'liq xato. Hozirgi spec'da amalda throw qilinmaydi, lekin backwards compatibility uchun mavjud.

**AggregateError** (ES2021) — bir nechta xatoni bitta obyektda saqlash uchun. `errors` property'si xatolar array'ini saqlaydi. `Promise.any()` barcha promise'lar reject bo'lganda shu xatoni throw qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Barcha error turlari prototype chain orqali `Error.prototype`'dan meros oladi. Bu shuni anglatadi: har qanday custom yoki built-in xato `instanceof Error` tekshiruvidan `true` qaytaradi.

```
Error.prototype
 ├── SyntaxError.prototype   → SyntaxError instance'lari
 ├── ReferenceError.prototype → ReferenceError instance'lari
 ├── TypeError.prototype     → TypeError instance'lari
 ├── RangeError.prototype    → RangeError instance'lari
 ├── URIError.prototype      → URIError instance'lari
 ├── EvalError.prototype     → EvalError instance'lari
 └── AggregateError.prototype → AggregateError instance'lari

Prototype chain:
TypeError instance → TypeError.prototype → Error.prototype → Object.prototype → null
```

Engine xato throw qilganda ichki jarayon: (1) mos Error subclass'ining yangi instance'ini yaratadi, (2) `message` property'ga xato xabarini yozadi, (3) `stack` property'ga joriy call stack snapshot'ini yozadi (non-standard, lekin barcha zamonaviy engine'lar qo'llaydi), (4) bu obyektni throw qiladi — execution to'xtaydi va eng yaqin catch blok qidiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === SyntaxError — parse vaqtida ===
// eval("if(");           // SyntaxError: Unexpected end of input
// JSON.parse("{invalid}"); // SyntaxError: Unexpected token i
JSON.parse("{'key': 'val'}"); // SyntaxError: Expected property name or '}'
// JSON faqat double quotes qabul qiladi

// === ReferenceError — o'zgaruvchi topilmadi ===
// console.log(notDefined); // ReferenceError: notDefined is not defined
typeof notDefined; // "undefined" — typeof xato bermaydi, scope tekshiruv uchun qulay

// === TypeError — noto'g'ri type da operatsiya ===
// null.toString();        // TypeError: Cannot read properties of null
// undefined.name;         // TypeError: Cannot read properties of undefined
// "string"();             // TypeError: "string" is not a function
// const x = 5; x = 10;   // TypeError: Assignment to constant variable

// === RangeError — chegaradan tashqari ===
// new Array(-1);          // RangeError: Invalid array length
// (1).toFixed(200);       // RangeError: toFixed() digits argument must be between 0 and 100
// function f() { f(); } f(); // RangeError: Maximum call stack size exceeded

// === URIError ===
// decodeURIComponent("%"); // URIError: URI malformed

// === AggregateError (ES2021) ===
const errors = [
  new TypeError("type xato"),
  new RangeError("range xato")
];
const aggError = new AggregateError(errors, "Bir necha xato yuz berdi");
console.log(aggError.message); // "Bir necha xato yuz berdi"
console.log(aggError.errors);  // [TypeError, RangeError]
console.log(aggError.errors[0] instanceof TypeError); // true

// Promise.any() — barcha reject bo'lganda AggregateError
// const result = await Promise.any([
//   Promise.reject(new Error("1")),
//   Promise.reject(new Error("2"))
// ]);
// AggregateError: All promises were rejected
```

```javascript
// instanceof bilan xato turini aniqlash
const err = new TypeError("noto'g'ri type");

console.log(err instanceof TypeError); // true
console.log(err instanceof Error);     // true — barcha xatolar Error dan meros
console.log(err instanceof RangeError); // false — boshqa subclass

// Prototype chain tekshiruv:
console.log(Object.getPrototypeOf(TypeError.prototype) === Error.prototype); // true
```

</details>

---

## Error Object

### Nazariya

`Error` obyekti xato haqida diagnostik ma'lumot saqlaydi. 3 ta asosiy property:

1. **`name`** — xato turi nomi (string). Built-in xatolar uchun constructor nomi bilan bir xil: `"TypeError"`, `"SyntaxError"`. Custom xatolarda o'zgartirish mumkin va kerak.

2. **`message`** — xato haqida odam tushunadigan xabar (string). `new Error("xabar")` constructor'ga berilgan birinchi argument.

3. **`stack`** — xato sodir bo'lgan joydagi call stack snapshot'i (string). ECMAScript spec'da rasman yo'q (non-standard), lekin barcha zamonaviy engine'lar (V8, SpiderMonkey, JavaScriptCore) qo'llaydi. Debugging uchun eng muhim ma'lumot — qaysi fayl, qaysi qator, qaysi funksiya zanjirida xato sodir bo'lganini ko'rsatadi.

`Error` obyekti `new Error(message)` yoki `Error(message)` (new'siz ham ishlaydi) bilan yaratiladi. Lekin doim `new` bilan yaratish tavsiya qilinadi — aniqlik uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine'da `Error` obyekti yaratilganda `stack` property **lazy** hisoblanadi — ya'ni darhol string sifatida hisoblanmaydi. Stack trace aslida `Error.captureStackTrace()` (V8-specific) orqali capture qilinadi va faqat `.stack` property'ga birinchi marta murojaat qilinganda string'ga aylantiriladi. Bu performance uchun muhim — agar error catch qilinib, stack hech qachon o'qilmasa, string formatlash sarflanmaydi.

```
Stack trace formati (V8):
Error: Xato xabari
    at functionName (fileName:lineNumber:columnNumber)
    at callerFunction (fileName:lineNumber:columnNumber)
    at main (fileName:lineNumber:columnNumber)
```

`Error.stackTraceLimit` (V8-specific) — stack trace da nechta frame saqlanishini belgilaydi. Default: 10. Production da ko'proq qilish mumkin: `Error.stackTraceLimit = 50`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Error yaratish va xususiyatlarini o'qish
const error = new Error("Foydalanuvchi topilmadi");

console.log(error.name);    // "Error"
console.log(error.message); // "Foydalanuvchi topilmadi"
console.log(error.stack);
// Error: Foydalanuvchi topilmadi
//     at Object.<anonymous> (app.js:1:15)
//     at Module._compile (node:internal/modules/cjs/loader:1241:14)
//     ...

// Error obyekti plain object emas — toString() override qilingan
console.log(String(error));  // "Error: Foydalanuvchi topilmadi"
console.log(`${error}`);     // "Error: Foydalanuvchi topilmadi"

// Subclass'lar name ni avtomatik o'rnatadi
const typeErr = new TypeError("noto'g'ri tip");
console.log(typeErr.name);    // "TypeError"
console.log(typeErr.message); // "noto'g'ri tip"
```

</details>

---

## Error cause (ES2022)

### Nazariya

`cause` option — xatoning asl sababini saqlash mexanizmi. `new Error(message, { cause })` ikkinchi argument sifatida options obyekti qabul qiladi. Bu **error chaining** imkonini beradi — yuqori darajadagi xato ichida past darajadagi asl xato saqlanadi.

Bu nima uchun kerak: ko'p qatlamli dasturlarda past darajadagi xato (network timeout, JSON parse, database connection) yuqori darajadagi xatoga aylantiriladi (foydalanuvchi yuklanmadi, buyurtma saqlanmadi). `cause` yo'q bo'lganda asl xato yo'qoladi — debugging qiyinlashadi. `cause` bilan xatolar zanjiri saqlanadi.

`cause` property `Error.prototype` da emas — har bir instance'ning own property'si sifatida saqlanadi. Har qanday qiymat bo'lishi mumkin (Error, string, number, object), lekin Error instance berish tavsiya qilinadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Past darajadagi xatoni yuqori darajadagi xatoga o'rash
async function fetchUserProfile(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (networkError) {
    // Asl xatoni cause sifatida saqlash
    throw new Error(`Foydalanuvchi #${userId} profili yuklanmadi`, {
      cause: networkError // ← asl xato saqlanadi
    });
  }
}

// Xatolar zanjirini o'qish
try {
  await fetchUserProfile(42);
} catch (error) {
  console.log(error.message);       // "Foydalanuvchi #42 profili yuklanmadi"
  console.log(error.cause);          // Error: HTTP 404
  console.log(error.cause.message);  // "HTTP 404"
  // Zanjirni chuqurroq kuzatish:
  // error.cause.cause — agar yana o'ralgan bo'lsa
}

// Ko'p qatlamli error chaining
function parseConfig(rawData) {
  try {
    return JSON.parse(rawData);
  } catch (parseError) {
    throw new Error("Konfiguratsiya fayli noto'g'ri formatda", {
      cause: parseError
    });
  }
}

function loadApp() {
  try {
    const config = parseConfig("invalid json");
  } catch (configError) {
    throw new Error("Dastur ishga tushmadi", {
      cause: configError // zanjir: App → Config → JSON.parse
    });
  }
}

try {
  loadApp();
} catch (error) {
  // To'liq zanjir:
  console.log(error.message);             // "Dastur ishga tushmadi"
  console.log(error.cause.message);        // "Konfiguratsiya fayli noto'g'ri formatda"
  console.log(error.cause.cause.message);  // "Unexpected token i in JSON..."
}
```

</details>

---

## try/catch/finally

### Nazariya

`try/catch/finally` — sinxron koddagi xatolarni ushlash konstruksiyasi. 3 blokdan iborat:

1. **`try`** — xato bo'lishi mumkin bo'lgan kod. Engine bu blok ichida throw sodir bo'lsa, qolgan kodni to'xtatib, catch blokga o'tadi.

2. **`catch (error)`** — faqat try blokda xato bo'lganda ishlaydi. `error` parametri throw qilingan qiymatni oladi. ES2019 dan boshlab `catch` parametrsiz ham bo'lishi mumkin: `catch { }` — agar error obyekti kerak bo'lmasa.

3. **`finally`** — **doim** ishlaydi: xato bo'lsa ham, bo'lmasa ham, `return` bo'lsa ham, `throw` bo'lsa ham. Resurs tozalash (file close, connection release, timer clear) uchun ideal. **MUHIM**: `finally` ichida `return` yozish catch/try dagi `return` va `throw`'ni override qiladi — bu juda xavfli va hech qachon qilinmasligi kerak.

Kombinatsiyalar: `try/catch`, `try/finally` (catch'siz — xato propagation davom etadi, lekin cleanup ishlaydi), `try/catch/finally` (to'liq).

<details>
<summary><strong>Under the Hood</strong></summary>

Engine try blokga kirganda joriy execution state'ni saqlaydi (stack pointer, scope chain). Agar throw sodir bo'lsa: (1) execution to'xtatiladi, (2) engine call stack bo'ylab eng yaqin catch blokni qidiradi, (3) catch topilsa — throw qilingan qiymat catch parametriga bind qilinadi va catch blok bajariladi, (4) topilmasa — xato global handler'ga yetadi yoki dastur crash qiladi.

`finally` implementation: engine try/catch bajarilgandan keyin (yoki throw bo'lgandan keyin) completion record'ni vaqtincha saqlaydi, finally blokni bajaradi, keyin saqlangan completion'ni qaytaradi. Shuning uchun finally ichida `return` yozsangiz — saqlangan completion (asl return yoki throw) yo'qoladi.

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
              normal         re-throw
             davom etadi    yuqoriga
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Asosiy try/catch ===
try {
  const data = JSON.parse("noto'g'ri json");
} catch (error) {
  console.log(error.name);    // "SyntaxError"
  console.log(error.message); // "Unexpected token n in JSON at position 0"
}
console.log("Dastur davom etadi"); // ✅ try/catch tufayli dastur to'xtamadi

// === finally — doim ishlaydi ===
function readConfig(path) {
  const connection = openConnection();
  try {
    const data = connection.read(path);
    return data; // return bo'lsa ham finally ishlaydi
  } catch (error) {
    console.error("O'qishda xato:", error.message);
    return null;
  } finally {
    connection.close(); // ✅ DOIM yopiladi — xato bo'lsa ham, return bo'lsa ham
  }
}

// === catch'siz try/finally — cleanup bilan re-throw ===
function processWithCleanup(resource) {
  const lock = resource.acquire();
  try {
    return resource.process(); // xato bo'lsa throw davom etadi
  } finally {
    lock.release(); // lekin cleanup albatta ishlaydi
  }
  // catch yo'q — xato yuqoriga propagate qiladi, lekin lock release bo'ldi
}

// === catch parametrsiz (ES2019) ===
function isValidJSON(str) {
  try {
    JSON.parse(str);
    return true;
  } catch { // ← error parametri kerak emas
    return false;
  }
}

// === Error turini tekshirish — instanceof bilan ===
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
    throw error; // ← kutilmagan xatolarni qayta throw qilish — MUHIM!
  }
}
```

```javascript
// ⚠️ finally ichida return — XAVFLI
function dangerousFinally() {
  try {
    return "try natijasi";
  } finally {
    return "finally natijasi"; // ⚠️ try dagi return ni OVERRIDE qiladi!
  }
}
console.log(dangerousFinally()); // "finally natijasi" — try dagi return yo'qoldi!

// ⚠️ finally ichida return — throw ni ham yo'qotadi
function swallowedError() {
  try {
    throw new Error("muhim xato!");
  } finally {
    return "finally"; // ⚠️ Error yo'qoldi — hech qachon tashqariga chiqmaydi!
  }
}
console.log(swallowedError()); // "finally" — error yutildi, debugging mumkin emas

// ✅ QOIDA: finally ichida HECH QACHON return yozmang!
// finally — faqat cleanup uchun: close, release, clear
```

</details>

---

## throw Statement

### Nazariya

`throw` statement joriy execution'ni to'xtatib, berilgan qiymatni xato sifatida tashqariga chiqaradi. Texnik jihatdan JavaScript da **har qanday qiymat** throw qilish mumkin — string, number, boolean, object, `null`, `undefined`. Lekin doim `Error` yoki uning subclass'ini throw qilish kerak. Sabab: faqat `Error` instance'larida `stack` property bor — xato qayerda sodir bo'lganini ko'rsatuvchi stack trace. String throw qilsangiz — `catch` da `error.stack` `undefined` bo'ladi, debugging juda qiyinlashadi.

`throw` expression emas, statement — ya'ni ternary ichida, yoki arrow function'da `=>` dan keyin to'g'ridan-to'g'ri ishlatib bo'lmaydi. Lekin IIFE yoki helper function orqali shu effektga erishish mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ✅ Error obyekti throw qilish — DOIM shu usulda
throw new Error("Nimadir noto'g'ri ketdi");
throw new TypeError("Argument string bo'lishi kerak");
throw new RangeError("Yosh 0-150 orasida bo'lishi kerak");

// ❌ String throw qilish — stack trace yo'qoladi
// throw "xato!";           // ❌ typeof error === "string", stack yo'q
// throw 42;                // ❌ number, stack yo'q
// throw { code: 500 };     // ❌ plain object, stack yo'q
// throw null;              // ❌ catch da error === null, message/stack yo'q

// Farqni ko'rish:
try {
  throw "string xato";
} catch (error) {
  console.log(typeof error);  // "string" — Error obyekti emas
  console.log(error.stack);   // undefined — stack trace yo'q!
  console.log(error.message); // undefined — message property yo'q!
}

try {
  throw new Error("Error obyekti");
} catch (error) {
  console.log(typeof error);  // "object"
  console.log(error.stack);   // ✅ to'liq stack trace
  console.log(error.message); // ✅ "Error obyekti"
}
```

```javascript
// throw ni validatsiyada ishlatish
function createUser(name, age) {
  if (typeof name !== "string" || name.trim().length === 0) {
    throw new TypeError("name string bo'lishi va bo'sh bo'lmasligi kerak");
  }
  if (typeof age !== "number" || age < 0 || age > 150) {
    throw new RangeError("age 0-150 orasidagi son bo'lishi kerak");
  }
  return { name: name.trim(), age };
}

try {
  const user = createUser("", 25);
} catch (error) {
  if (error instanceof TypeError) {
    console.log("Validatsiya:", error.message);
    // "name string bo'lishi va bo'sh bo'lmasligi kerak"
  }
}
```

</details>

---

## Custom Error Classes

### Nazariya

Katta dasturlarda built-in error turlaridan tashqari o'z xato class'larimiz kerak bo'ladi — bu xatolarni kategoriyalash, `instanceof` bilan aniq tekshirish, qo'shimcha ma'lumot (field, statusCode, resource) biriktirish imkonini beradi. Custom error class yaratish: `extends Error` bilan meros olish, `super(message)` chaqirish, `this.name` ni o'rnatish (debugging da stack trace'da ko'rinadi). `this.name = this.constructor.name` pattern barcha subclass'lar uchun avtomatik to'g'ri nom beradi.

Custom error'lar ikki muhim kategoriyaga bo'linadi:
- **Operational errors** — kutilgan, boshqarish mumkin: validatsiya xatosi, 404 not found, network timeout. Dastur davom etishi mumkin.
- **Programmer errors** — koddagi bug: undefined property o'qish, noto'g'ri argument. Dasturni to'xtatish va tuzatish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

`extends Error` bilan class yaratilganda prototype chain to'g'ri quriladi:

```
CustomError instance
  → CustomError.prototype (name, custom methods)
    → Error.prototype (toString, stack capture)
      → Object.prototype
        → null
```

`super(message)` chaqirilganda `Error` constructor ichida `this.message` o'rnatiladi va V8 engine **avtomatik ravishda** joriy call stack snapshot'ini saqlaydi (`.stack` property shu snapshot'dan lazy formatlanadi). Agar custom error class uchun stack trace'dan konstruktor frame'ini olib tashlashni xohlasangiz — alohida `Error.captureStackTrace(this, CustomError)` static helper'ini constructor ichida **qo'lda** chaqirasiz. Ya'ni: stack trace capture avtomatik, lekin "custom class konstruktorini stack'dan hide qilish" uchun `Error.captureStackTrace()` manual chaqirish kerak.

**Prototype chain muammosi tarixi:** ES5 (transpile) davrida `extends Error` to'g'ri ishlamasdi — `Error` konstruktori `new.target` ga qarab yangi obyekt qaytarardi, bu esa `this` ni bypass qilib meros chainni buzardi. **ES6 class syntax (2015)** bu muammoni hal qildi — ES6 `class` va `super()` spec'da aniq belgilangan tarzda `this` ni to'g'ri o'rnatadi va prototype chain avtomatik quriladi. Babel va boshqa ES5 transpiler'larda `Error` subclass'lari uchun maxsus workaround (`Object.setPrototypeOf(this, new.target.prototype)`) ishlatilardi — native ES6 syntax'da bu kerak emas. ES2022 `Error.cause` option qo'shdi (error chaining uchun), lekin subclass hierarchy bilan bog'liq emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Oddiy custom error ===
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError"; // stack trace da ko'rinadi
    this.field = field;            // qaysi field xato ekani
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

// Ishlatish
function findUser(id) {
  const user = database.get(id);
  if (!user) {
    throw new NotFoundError("Foydalanuvchi", id);
  }
  return user;
}

try {
  findUser(42);
} catch (error) {
  if (error instanceof NotFoundError) {
    console.log(error.message);    // "Foydalanuvchi #42 topilmadi"
    console.log(error.resource);   // "Foydalanuvchi"
    console.log(error.statusCode); // 404
  }
}
```

```javascript
// === Error Hierarchy — katta loyihalar uchun ===
class AppError extends Error {
  constructor(message, statusCode = 500, cause) {
    super(message, { cause });
    this.name = this.constructor.name; // ← subclass nomi avtomatik
    this.statusCode = statusCode;
    this.isOperational = true; // operational error — dastur davom etishi mumkin
  }
}

// Client tomonidan kelgan xatolar (4xx)
class ClientError extends AppError {
  constructor(message, statusCode = 400, cause) {
    super(message, statusCode, cause);
  }
}

// Server ichki xatolari (5xx)
class ServerError extends AppError {
  constructor(message, cause) {
    super(message, 500, cause);
    this.isOperational = false; // programmer error — critical, restart kerak bo'lishi mumkin
  }
}

// Aniq xato turlari
class BadRequestError extends ClientError {
  constructor(message, field) {
    super(message, 400);
    this.field = field;
  }
}

class UnauthorizedError extends ClientError {
  constructor(message = "Autentifikatsiya kerak") {
    super(message, 401);
  }
}

class ForbiddenError extends ClientError {
  constructor(message = "Ruxsat yo'q") {
    super(message, 403);
  }
}

class NotFoundError extends ClientError {
  constructor(resource = "Resurs") {
    super(`${resource} topilmadi`, 404);
  }
}

// instanceof zanjir tekshiruv
const error = new NotFoundError("Foydalanuvchi");
console.log(error instanceof NotFoundError); // true
console.log(error instanceof ClientError);   // true
console.log(error instanceof AppError);      // true
console.log(error instanceof Error);         // true
console.log(error.statusCode);               // 404
console.log(error.isOperational);            // true
console.log(error.name);                     // "NotFoundError" — constructor.name tufayli
```

```
Error Hierarchy diagramma:

Error
 └── AppError (statusCode, isOperational)
      ├── ClientError (4xx)
      │    ├── BadRequestError (400, field)
      │    ├── UnauthorizedError (401)
      │    ├── ForbiddenError (403)
      │    └── NotFoundError (404, resource)
      │
      └── ServerError (5xx, isOperational: false)
           ├── DatabaseError
           └── ExternalServiceError
```

```javascript
// Error handler — Express middleware pattern
function errorHandler(error, req, res, next) {
  // Logging
  if (!error.isOperational) {
    console.error("CRITICAL ERROR:", error); // alert yuborish kerak
  }

  // Client ga javob
  if (error instanceof BadRequestError) {
    return res.status(400).json({
      error: error.message,
      field: error.field
    });
  }
  if (error instanceof NotFoundError) {
    return res.status(404).json({ error: error.message });
  }
  if (error instanceof UnauthorizedError) {
    return res.status(401).json({ error: error.message });
  }

  // Default: 500
  res.status(error.statusCode || 500).json({
    error: error.isOperational ? error.message : "Server xatosi"
  });
}
```

</details>

---

## Error Propagation

### Nazariya

Error propagation — xatoning call stack bo'ylab yuqoriga ko'tarilish mexanizmi. Agar funksiya ichida throw sodir bo'lsa va o'sha funksiyada `try/catch` bo'lmasa — xato chaqiruvchi (caller) funksiyaga qaytadi. Chaqiruvchida ham catch bo'lmasa — uning chaqiruvchisiga qaytadi. Bu jarayon eng yuqori darajadagi `try/catch` topilguncha yoki call stack'ning oxirigacha davom etadi. Agar hech qayerda catch topilmasa — dastur crash qiladi (browser'da console error, Node.js da process exit).

Bu mexanizmning 2 ta muhim pattern'i bor:

1. **Re-throwing** — catch'da faqat bilgan xatolarni boshqarish, qolganlarini qayta `throw` qilish. Bu eng yaxshi amaliyot — faqat boshqara oladigan xatolarni ushlash, boshqalarini yuqoriga uzatish.

2. **Error Translation** — past darajadagi texnik xatoni (HTTP 404, ECONNREFUSED) yuqori darajadagi business logic xatoga aylantirish (NotFoundError, ServiceUnavailableError). Bu abstraction darajalarini saqlaydi — controller HTTP xato haqida o'ylamasligi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

```
  Call Stack               Error Propagation yo'nalishi

  ┌──────────┐
  │  step3() │ ──throw──▶ Error yaratildi
  ├──────────┤            │
  │  step2() │ ◀──────────┘ catch yo'q → yuqoriga
  ├──────────┤            │
  │  step1() │ ◀──────────┘ catch yo'q → yuqoriga
  ├──────────┤            │
  │ try/catch│ ◀──────────┘ ✅ USHLANDI!
  └──────────┘
```

Engine xato throw bo'lganda har bir stack frame'ni tekshiradi — agar frame'da exception handler (catch) ro'yxatga olingan bo'lsa, execution o'sha catch blokdan davom etadi. Agar yo'q bo'lsa — frame stack'dan chiqariladi (unwinding) va keyingi frame tekshiriladi. Bu "stack unwinding" deyiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Propagation misoli ===
function step3() {
  throw new Error("step3 da xato!"); // ① xato yaratildi
}

function step2() {
  step3(); // ② catch yo'q — xato yuqoriga
}

function step1() {
  step2(); // ③ catch yo'q — yana yuqoriga
}

try {
  step1(); // ④ shu yerda ushlanadi
} catch (error) {
  console.log(error.message); // "step3 da xato!"
  // Stack trace to'liq zanjirni ko'rsatadi:
  // Error: step3 da xato!
  //   at step3 (...)
  //   at step2 (...)
  //   at step1 (...)
}
```

```javascript
// === Re-throwing pattern ===
function processData(data) {
  try {
    return JSON.parse(data);
  } catch (error) {
    if (error instanceof SyntaxError) {
      // JSON xatosini boshqaramiz — fallback qaytaramiz
      console.warn("Noto'g'ri JSON:", data);
      return null;
    }
    // Boshqa barcha xatolarni qayta throw — bu bizning mas'uliyatimiz emas
    throw error;
  }
}
```

```javascript
// === Error Translation pattern ===
async function getUserProfile(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    // Past darajadagi HTTP xatoni yuqori darajadagi xatoga aylantirish
    if (error.message.includes("404")) {
      throw new NotFoundError("Foydalanuvchi");
    }
    if (error.message.includes("403")) {
      throw new ForbiddenError();
    }
    // Barcha boshqa xatolar uchun umumiy server xatosi
    throw new ServerError("Profil yuklanmadi", error); // cause sifatida asl xato
  }
}
```

</details>

---

## Async Error Handling

### Nazariya

Asinxron koddagi xatolar sinxron koddan tubdan farq qiladi — `try/catch` faqat sinxron throw'ni ushlaydi, asinxron callback ichidagi throw'ni ushlamaydi. Asinxron xatolarni boshqarishning 3 ta usuli bor:

1. **Callback'lar** — error-first pattern (Node.js convention): callback'ning birinchi argumenti `error` (xato bo'lsa Error, bo'lmasa `null`), ikkinchi argumenti `data`. Bu pattern Promise'lardan oldingi standart edi.

2. **Promise** — `.catch()` methodi rejected promise'ni ushlaydi. `.catch()` pozitsiyasi muhim — u faqat o'zidan **oldingi** `.then()` larni ushlaydi. `Promise.allSettled()` barcha natijalarni (fulfilled va rejected) qaytaradi — `Promise.all()`'dan farqli birinchi rejection'da to'xtamaydi.

3. **async/await** — `try/catch` bilan eng qulay usul. `await` expression reject bo'lgan promise'ni sinxron `throw` ga aylantiradi — shuning uchun oddiy `try/catch` ishlaydi. **MUHIM**: har bir `async` funksiya chaqiruvi natijasida `.catch()` qo'yish yoki `try/catch` ichida chaqirish kerak — aks holda `UnhandledPromiseRejection` bo'ladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === 1. Error-first callback (Node.js) ===
const fs = require("fs");

fs.readFile("data.txt", "utf8", (error, data) => {
  if (error) {
    // ← birinchi argument error
    console.error("Fayl o'qib bo'lmadi:", error.message);
    return; // ← early return — data bilan ishlamaslik uchun
  }
  console.log("Fayl:", data); // ← error null bo'lganda
});
```

```javascript
// === 2. Promise — .catch() bilan ===
fetch("/api/users")
  .then(response => {
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  })
  .then(users => {
    console.log("Foydalanuvchilar:", users);
  })
  .catch(error => {
    // fetch xatosi YOKI json parse xatosi YOKI HTTP xatosi — barchasi shu yerga
    console.error("Xato:", error.message);
  });

// catch pozitsiyasi — recovery pattern
fetch("/api/data")
  .then(response => response.json())
  .catch(error => {
    console.warn("API xatosi, fallback ishlatiladi");
    return { items: [], fallback: true }; // ✅ recovery — keyingi then'ga o'tadi
  })
  .then(data => {
    // data — yoki API natijasi, yoki fallback obyekti
    console.log("Data:", data);
  });

// Promise.all — birinchi rejection da BARCHA natija yo'qoladi
Promise.all([
  fetch("/api/users").then(r => r.json()),
  fetch("/api/posts").then(r => r.json()),
  fetch("/api/comments").then(r => r.json())
])
.then(([users, posts, comments]) => {
  // faqat barcha muvaffaqiyatli bo'lganda
})
.catch(error => {
  // BITTA xato bo'lsa ham shu yerga — boshqalarning natijasi yo'qoldi
});

// Promise.allSettled — barcha natijalarni olish, hech biri yo'qolmaydi
const results = await Promise.allSettled([
  fetch("/api/users").then(r => r.json()),
  fetch("/api/posts").then(r => r.json()),
  fetch("/api/comments").then(r => r.json())
]);

results.forEach((result, index) => {
  if (result.status === "fulfilled") {
    console.log(`#${index} muvaffaqiyatli:`, result.value);
  } else {
    console.error(`#${index} xato:`, result.reason.message);
  }
});
```

```javascript
// === 3. async/await — try/catch bilan ===
async function loadDashboard() {
  try {
    const user = await fetchUser();
    const posts = await fetchPosts(user.id);
    return { user, posts };
  } catch (error) {
    console.error("Dashboard yuklanmadi:", error.message);
    return null; // fallback
  }
}

// Har bir await uchun alohida error handling — qaysi qadam xato berganini aniqlash
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
    posts = []; // partial recovery — foydalanuvchi bor, postlar yo'q
  }

  return { user, posts };
}

// Parallel async — Promise.all + try/catch
async function loadAll() {
  try {
    const [users, posts] = await Promise.all([
      fetch("/api/users").then(r => r.json()),
      fetch("/api/posts").then(r => r.json())
    ]);
    return { users, posts };
  } catch (error) {
    console.error("Parallel yuklashda xato:", error.message);
  }
}
```

```javascript
// ⚠️ try/catch SINXRON throw'ni ushlaydi — asinxron callback'ni EMAS!
try {
  setTimeout(() => {
    throw new Error("Async xato!"); // ❌ try/catch buni ushlamaydi!
  }, 1000);
} catch (error) {
  // Bu yerga HECH QACHON tushmaydi — setTimeout callback
  // boshqa execution context'da ishlaydi
  console.log("Ushlandimi?", error); // ❌ ishlamaydi
}

// ⚠️ async funksiyani catch'siz chaqirish
async function riskyAsync() {
  throw new Error("Async xato!");
}
riskyAsync(); // ❌ UnhandledPromiseRejection! catch yo'q

// ✅ To'g'ri usullar:
riskyAsync().catch(console.error);
// Yoki:
try { await riskyAsync(); } catch (error) { console.error(error); }
```

</details>

---

## Global Error Handlers

### Nazariya

Global error handler'lar — ushlanmagan xatolar uchun **oxirgi himoya qatlami**. Ular asosiy error handling mexanizmi o'rniga emas, qo'shimcha sifatida ishlatiladi. Asosiy vazifasi: (1) xatoni log qilish, (2) monitoring servisiga (Sentry, LogRocket, Datadog) yuborish, (3) foydalanuvchiga umumiy xato xabari ko'rsatish.

**Browser'da:**
- `window.onerror` — ushlanmagan JavaScript runtime xatolar (TypeError, ReferenceError va b.); resurs yuklash xatolarini ushlamaydi
- `window.addEventListener("error", ...)` — runtime xatolar + resurs yuklash xatolari (img, script, link)
- `window.addEventListener("unhandledrejection", ...)` — ushlanmagan Promise rejection'lar

**Node.js'da:**
- `process.on("uncaughtException", ...)` — ushlanmagan sinxron xatolar
- `process.on("unhandledRejection", ...)` — ushlanmagan Promise rejection'lar

`uncaughtException` dan keyin Node.js process'ni to'xtatish (exit) tavsiya qilinadi — chunki application state noaniq holatda bo'lishi mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Browser — sinxron xatolar ===
window.onerror = function(message, source, lineno, colno, error) {
  // error — Error obyekti (yoki null)
  sendToMonitoring({
    type: "runtime_error",
    message,
    source,        // fayl URL
    lineno,        // qaysi qator
    colno,         // qaysi ustun
    stack: error?.stack
  });

  return true; // true — brauzer console da error ko'rsatmaydi
  // false/undefined — default xulq (console da ko'rsatadi)
};

// addEventListener versiya — ko'proq imkoniyat beradi
window.addEventListener("error", (event) => {
  // event.error — Error obyekti
  // event.message, event.filename, event.lineno, event.colno
  console.log("Global error:", event.error);

  // Resurs yuklash xatolarini aniqlash (img, script):
  if (event.target !== window) {
    console.log("Resurs yuklanmadi:", event.target.src || event.target.href);
  }
});

// === Browser — ushlanmagan Promise rejection ===
window.addEventListener("unhandledrejection", (event) => {
  // event.reason — reject qilingan qiymat (odatda Error)
  // event.promise — reject bo'lgan Promise
  sendToMonitoring({
    type: "unhandled_rejection",
    reason: event.reason?.message || String(event.reason),
    stack: event.reason?.stack
  });

  event.preventDefault(); // default console warning'ni to'xtatish
});

// Rejection keyinroq ushlanganda — monitoring uchun foydali
window.addEventListener("rejectionhandled", (event) => {
  console.log("Rejection keyin ushlandi:", event.reason);
});
```

```javascript
// === Node.js ===

// Ushlanmagan sinxron xatolar
process.on("uncaughtException", (error, origin) => {
  console.error(`Ushlanmagan xato [${origin}]:`, error);
  // Logging service ga yuborish
  sendToMonitoring({ type: "uncaught_exception", error: error.stack });

  // ⚠️ Process ni to'xtatish TAVSIYA QILINADI
  // Application state noaniq — davom etish xavfli
  process.exit(1);
});

// Ushlanmagan Promise rejection
process.on("unhandledRejection", (reason, promise) => {
  console.error("Ushlanmagan rejection:", reason);
  // Logging
  sendToMonitoring({ type: "unhandled_rejection", reason: String(reason) });
  // Node.js 15+ da default behavior: process exit
});
```

</details>

---

## Error Handling Patterns

### Pattern 1: Result Type (Go-style)

### Nazariya

Error throw qilish o'rniga natija obyekti qaytarish — `{ ok: true, data }` yoki `{ ok: false, error }`. Bu pattern `try/catch` kerak qilmaydi va xato boshqarishni explicit qiladi. Go tilida standart pattern, JavaScript da kutilgan xatolar uchun foydali — ayniqsa xato "exception" emas, "normal control flow" bo'lganda (validatsiya, tekshiruv).

<details>
<summary><strong>Under the Hood</strong></summary>

Result Type pattern'da `try/catch` ga nisbatan boshqa control flow mexanizmi ishlaydi. `throw` statement engine'da stack unwinding boshlaydi — call stack bo'ylab yuqoriga qarab `catch` blok qidiriladi, har bir frame'dan environment record tozalanadi. Result Type'da esa bunday unwinding yo'q — funksiya oddiy `return` statement orqali natija qaytaradi, call stack normal tarzda pop bo'ladi. Bu ikki farq performance'ga ta'sir qiladi: V8 da `try/catch` blok ichidagi kod avval optimizatsiya qilinmas edi (TurboFan dan oldingi JIT compilerlarda), hozir bu muammo hal qilingan, lekin `throw` ning o'zi hali ham qimmat operatsiya — Error obyekt yaratiladi, stack trace capture qilinadi. Result Type'da Error obyekti faqat kerak bo'lganda yaratiladi. Discriminated union pattern `{ ok, data, error }` — engine bu obyektlarni inline allocation bilan yaratadi, hidden class bir xil bo'lgani uchun monomorphic property access ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
function safeDivide(a, b) {
  if (b === 0) {
    return { ok: false, error: "Nolga bo'lish mumkin emas" };
  }
  return { ok: true, data: a / b };
}

const result = safeDivide(10, 0);
if (!result.ok) {
  console.log("Xato:", result.error); // explicit error handling
} else {
  console.log("Natija:", result.data);
}

// Asinxron versiya — wrapper funksiya
async function safeAsync(fn) {
  try {
    const data = await fn();
    return { ok: true, data };
  } catch (error) {
    return { ok: false, error };
  }
}

const { ok, data, error } = await safeAsync(() =>
  fetch("/api/users").then(r => r.json())
);
if (!ok) {
  console.error("API xatosi:", error.message);
}
```

</details>

### Pattern 2: Retry with Exponential Backoff

### Nazariya

Vaqtinchalik xatolar (network timeout, rate limiting, server overload) uchun qayta urinish. Har safar kutish vaqti eksponensial oshadi (1s → 2s → 4s → 8s) — bu server'ni haddan tashqari yuklashni oldini oladi. Doimiy xatolar uchun emas — faqat vaqtinchalik (transient) xatolar uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

`setTimeout(resolve, waitTime)` chaqirilganda browser/Node.js timer'ni Web API (yoki libuv) ga topshiradi. Timer tugagach callback task queue (macrotask queue) ga qo'shiladi — lekin bu aniq `waitTime` ms dan keyin emas, balki **kamida** `waitTime` ms dan keyin. Agar call stack band bo'lsa yoki queue'da oldingi task'lar bo'lsa, haqiqiy delay ko'proq bo'ladi. `Math.pow(backoff, attempt)` bilan hisoblangan delay — bu minimum delay. HTML spec bo'yicha nested `setTimeout` (4+ darajadan keyin) kamida 4ms clamp qilinadi, lekin retry pattern'da bu ahamiyatsiz chunki delay'lar katta. `await new Promise(resolve => setTimeout(resolve, waitTime))` — bu Promise microtask va setTimeout macrotask'ni birlashtiradi: `setTimeout` callback macrotask sifatida ishlaydi, `resolve()` chaqiradi, keyin `await` davomidagi kod microtask sifatida ishga tushadi. Jitter qo'shish (`delay * (1 + Math.random() * 0.1)`) server thundering herd muammosini kamaytiradi — barcha client'lar bir vaqtda retry qilmasligini ta'minlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
async function withRetry(fn, { maxRetries = 3, delay = 1000, backoff = 2 } = {}) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn(attempt);
    } catch (error) {
      lastError = error;

      if (attempt === maxRetries) break; // oxirgi urinish — loop'dan chiqish

      const waitTime = delay * Math.pow(backoff, attempt);
      console.warn(`Urinish ${attempt + 1}/${maxRetries + 1} muvaffaqiyatsiz, ${waitTime}ms kutilmoqda...`);

      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }

  throw new Error(`${maxRetries + 1} urinishdan keyin muvaffaqiyatsiz`, {
    cause: lastError
  });
}

// Ishlatish
const data = await withRetry(
  () => fetch("/api/unstable-endpoint").then(r => {
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return r.json();
  }),
  { maxRetries: 3, delay: 1000, backoff: 2 }
  // Kutish: 1s, 2s, 4s
);
```

</details>

### Pattern 3: Circuit Breaker

### Nazariya

Tashqi servisga ko'p marta muvaffaqiyatsiz so'rov yuborilgandan keyin — yangi so'rovlarni butunlay to'xtatish pattern'i. 3 ta holat: **CLOSED** (normal ishlaydi), **OPEN** (so'rovlar rad etiladi — servisga yuklanish bermaydi), **HALF_OPEN** (timeout o'tgandan keyin bitta sinov so'rov yuboriladi). Muvaffaqiyatli bo'lsa CLOSED ga qaytadi, xato bo'lsa yana OPEN.

<details>
<summary><strong>Under the Hood</strong></summary>

Circuit Breaker — explicit state machine: 3 ta state (`CLOSED`, `OPEN`, `HALF_OPEN`) va transition'lar. `Date.now()` bilan vaqt taqqoslanadi — u `performance.now()` dan farqli o'laroq Unix epoch'dan millisecond qaytaradi, odatda 1ms precision bilan. Timing attack mitigation kerak bo'lgan sirtlarda (`performance.now()` cross-origin isolated contexts'da, Firefox'ning Fingerprinting Resistance rejimi, Tor Browser) browser'lar vaqt precision'ini coarsen qiladi — lekin bu `Date.now()` ga emas, asosan `performance.now()` ga taalluqli (5μs → 100μs yoki undan ko'proq). Circuit Breaker use case'da bu muhim emas — millisecond precision yetarli. State transition'lar atomik emas — concurrent request'lar `HALF_OPEN` state'da race condition yaratishi mumkin: ikkita request bir vaqtda `Date.now() >= this.nextAttempt` tekshiruvidan o'tib, ikkalasi ham sinov so'rov yuboradi. Production'da mutex yoki single-flight pattern kerak. `this.failures` counter oddiy property — V8 da Smi (Small Integer) sifatida saqlanadi (31-bit, pointer tagging), heap allocation yo'q. `threshold` bilan taqqoslash Smi comparison — eng tez operatsiyalardan biri.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class CircuitBreaker {
  constructor(fn, { threshold = 5, timeout = 30000 } = {}) {
    this.fn = fn;
    this.threshold = threshold; // nechta xatodan keyin circuit ochiladi
    this.timeout = timeout;     // OPEN dan HALF_OPEN ga o'tish vaqti
    this.failures = 0;
    this.state = "CLOSED";
    this.nextAttempt = 0;       // HALF_OPEN ga o'tish vaqti (timestamp)
  }

  async execute(...args) {
    if (this.state === "OPEN") {
      if (Date.now() < this.nextAttempt) {
        throw new Error("Circuit OPEN — so'rov rad etildi");
      }
      this.state = "HALF_OPEN"; // timeout o'tdi — sinov so'rov
    }

    try {
      const result = await this.fn(...args);
      this.reset(); // muvaffaqiyatli — CLOSED ga qaytish
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

  reset() {
    this.failures = 0;
    this.state = "CLOSED";
  }
}

// Ishlatish
const apiBreaker = new CircuitBreaker(
  (url) => fetch(url).then(r => {
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return r.json();
  }),
  { threshold: 3, timeout: 10000 } // 3 xatodan keyin 10s to'xtatish
);

try {
  const data = await apiBreaker.execute("/api/external-service");
} catch (error) {
  // "Circuit OPEN" yoki haqiqiy API xatosi
  console.error(error.message);
}
```

</details>

### Pattern 4: Graceful Degradation

### Nazariya

Asosiy funksiya ishlamasa — bir necha bosqichda fallback'larga o'tish. Foydalanuvchi xech narsa sezmaydi yoki kamida qisman natija oladi. API ishlamasa → cache'dan, cache ishlamasa → default qiymat. Bu UX uchun juda muhim — foydalanuvchi "500 Server Error" ko'rmasligi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

Har bir `try/catch` bloki engine'da completion record hosil qiladi — `{ [[Type]]: throw, [[Value]]: error }`. Catch blok bu record'ni ushlaydi va yangi normal completion bilan davom etadi. Ketma-ket `try/catch` bloklar (fallback chain) — har biri mustaqil stack unwinding qiladi. Birinchi `try` da `fetch` reject bo'lganda, engine `await` orqali microtask'dan throw qiladi, catch ushlaydi, keyin ikkinchi `try` ga o'tiladi. Bu yerda muhim nuance: `catch` ichida `console.warn` chaqirilganda xato log qilinadi, lekin execution normal davom etadi — chunki `catch` blok xatoni "handled" deb belgilaydi. Agar barcha `try/catch` bloklar muvaffaqiyatsiz bo'lsa, oxirgi `return` default object — bu heap'da yangi object yaratadi. V8 shu object literal'ni har safar yangi hidden class bilan yaratadi, lekin inline cache tufayli keyingi chaqiruvlarda tezlashadi. Fallback chain'da har bir bosqich sinxron yoki asinxron bo'lishi mumkin — `localStorage.getItem` sinxron (main thread'ni bloklaydi), `fetch` esa asinxron.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
async function getUserData(userId) {
  // 1-bosqich: API dan olish
  try {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (apiError) {
    console.warn("API ishlamadi:", apiError.message);
  }

  // 2-bosqich: Cache dan olish
  try {
    const cached = localStorage.getItem(`user_${userId}`);
    if (cached) return JSON.parse(cached);
  } catch (cacheError) {
    console.warn("Cache ham ishlamadi:", cacheError.message);
  }

  // 3-bosqich: Default qiymat
  return {
    id: userId,
    name: "Noma'lum foydalanuvchi",
    avatar: "/images/default-avatar.png"
  };
}
```

</details>

### Pattern 5: Fail Fast

### Nazariya

Xatolarni imkon qadar erta aniqlash — funksiya boshida argumentlarni tekshirish, noto'g'ri bo'lsa darhol throw qilish. Bu chuqur call stack ichida g'alati xato xabarlar o'rniga, aniq va tushunarli xabar beradi. "Guard clauses" deb ham ataladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`new TypeError("message")` yoki `new RangeError("message")` chaqirilganda engine: (1) mos prototype chain'li yangi object yaratadi, (2) `message` property'ni own property sifatida yozadi, (3) `Error.captureStackTrace()` (V8-specific) orqali stack trace capture qiladi. Stack trace capture — bu eng qimmat qism: engine hozirgi call stack'dagi har bir frame uchun function name, script URL, line/column number'ni saqlaydi. V8 da stack trace lazy formatted — `.stack` property faqat birinchi o'qilganda string'ga aylanadi. Guard clause'lar funksiya boshida throw qilganda stack trace qisqa bo'ladi (chuqur nesting'ga yetib bormagan). Agar tekshiruvsiz chuqurga kirsa — stack trace uzun, xato xabari noaniq, debug qiyin. `Error.captureStackTrace(this, Constructor)` custom error class'da constructor frame'ni stack'dan olib tashlaydi — foydalanuvchi faqat `throw` qilgan joyni ko'radi, Error constructor ichini emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
function processOrder(order) {
  // Guard clauses — boshida tekshirish, xato bo'lsa darhol throw
  if (!order) {
    throw new TypeError("order argument kerak");
  }
  if (!Array.isArray(order.items) || order.items.length === 0) {
    throw new ValidationError("Buyurtmada kamida 1 ta mahsulot bo'lishi kerak", "items");
  }
  if (typeof order.total !== "number" || order.total <= 0) {
    throw new RangeError("Buyurtma summasi musbat son bo'lishi kerak");
  }

  // Asosiy logika — faqat validatsiya o'tgandan keyin
  return submitOrder(order);
}
```

</details>

---

## Stack Traces va Debugging

### Nazariya

**Error.stack** — xato sodir bo'lgan joydan boshlab, chaqiruvlar zanjirini ko'rsatadigan string. Bu xato **yaratilgan** joydagi call stack snapshot'ini saqlaydi — `new Error()` yoki `throw` paytida hisoblab chiqariladi. Stack property non-enumerable bo'lib, runtime'da error obyektiga lazy biriktiriladi.

**Error.stackTraceLimit** — V8 engine'da stack trace'da nechta frame saqlanishini belgilaydi. Default 10. Chuqur rekursiya yoki complex call chain'larda 10 frame yetarli bo'lmasligi mumkin — `Error.stackTraceLimit = 50` yoki `Infinity` qilib ko'paytirish mumkin. Firefox va Safari da bu property mavjud emas.

**Source maps** — production'da minified/bundled kod xato berganda, stack trace'dagi fayl nomlari va qator raqamlari original kodga mos kelmaydi. Source map (`.map` fayl) minified kodning har bir qismini original source code'ga map qiladi. Browser DevTools source map'ni avtomatik aniqlaydi va stack trace'ni original fayllarga tarjima qiladi.

**Error serialization muammosi** — `JSON.stringify(error)` natijasi `"{}"` — bo'sh obyekt. Sababi: `message`, `stack`, `name` property'lari **non-enumerable**. Yechim: custom replacer yoki qo'lda serialize qilish kerak.

**Async stack traces** — `await` bilan chaqirilgan asinxron funksiya xato berganda, oddiy stack trace faqat `async` funksiya ichidagi frame'larni ko'rsatadi — chaqirgan joyni ko'rsatmaydi. Chrome DevTools "Async" stack trace'ni yoqsa (default yoqilgan), `await` qilgan joy ham ko'rinadi. Node.js da `--async-stack-traces` flag (v12+).

<details>
<summary><strong>Under the Hood</strong></summary>

**`Error.stack` ECMAScript spec'da yo'q** — bu **de facto** standart. Har engine o'z formatida:
- **V8** (Chrome/Node.js): `    at functionName (file:line:col)`
- **Firefox**: `functionName@file:line:col`
- **Safari**: `functionName@file:line:col` (shortened)

Bu farqlar Sentry/LogRocket kabi tool'lar har engine uchun alohida parser yozishiga sabab.

**V8 lazy stack trace**: `new Error()` paytida stack darhol hisoblanmaydi — faqat internal pointer saqlanadi. `error.stack` birinchi marta o'qilganda format qilinadi va cache'lanadi. Bu performance optimizatsiyasi: agar stack o'qilmasa, formatting overhead yo'q.

**`Error.captureStackTrace(target, constructorOpt)`** — V8-only API. `constructorOpt` funksiya va undan keyingi frame'larni stack'dan chiqarib tashlaydi. Custom error class'lar yoki assert utility'lar uchun foydali — foydalanuvchi stack'da utility frame'larini ko'rmaydi:
```
at MyAssert (lib.js:5)      ← removed
at validateUser (app.js:42)  at validateUser (app.js:42)
```

**Source maps**: `.map` fayl JSON format, asosiy field `mappings` — VLQ encoded. Browser DevTools `//# sourceMappingURL=` comment'dan topib avtomatik yuklaydi. **Production'da publik qilmaslik kerak** — original kodni ochib qo'yadi.

**Async stack traces (V8 8.6+)**: Klassik stack trace microtask boundary'da frame'larni yo'qotadi. V8 `zero-cost async stack traces` — async chain'larni stitch qiladi, `at async` qatorlarida ko'rinadi. Chrome DevTools'da va **Node.js 12+ da default yoqilgan** — hech qanday flag kerak emas. Disable qilish uchun `--no-async-stack-traces` flag ishlatiladi. Eski Node.js versiyalarida (v10 va undan avval) bu feature mavjud emas edi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Error serialization — JSON.stringify muammosi va yechimi ===

const err = new TypeError("Noto'g'ri tur");

// ❌ Muammo — bo'sh obyekt
console.log(JSON.stringify(err)); // "{}"

// Sabab: message, stack, name — non-enumerable
console.log(Object.keys(err)); // [] — bo'sh!
console.log(Object.getOwnPropertyDescriptor(Error.prototype, "message"));
// { writable: true, enumerable: false, configurable: true }

// ✅ Yechim 1: Custom serializer
function serializeError(error) {
  const serialized = {
    name: error.name,
    message: error.message,
    stack: error.stack,
  };

  // Custom property'lar ham bo'lishi mumkin
  if (error.code) serialized.code = error.code;
  if (error.cause) serialized.cause = serializeError(error.cause);

  // Enumerable property'larni ham olish
  for (const key of Object.keys(error)) {
    serialized[key] = error[key];
  }

  return serialized;
}

const serialized = JSON.stringify(serializeError(err), null, 2);
// {
//   "name": "TypeError",
//   "message": "Noto'g'ri tur",
//   "stack": "TypeError: Noto'g'ri tur\n    at ..."
// }

// ✅ Yechim 2: JSON.stringify replacer bilan
JSON.stringify(err, ["name", "message", "stack", "cause"]);
// '{"name":"TypeError","message":"Noto'g'ri tur","stack":"..."}'
```

```javascript
// === Error.captureStackTrace — V8 specific utility ===

// Custom error yaratishda stack trace'dan keraksiz frame'larni olib tashlash
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = "AppError";
    this.statusCode = statusCode;

    // Stack trace'dan AppError constructor'ni olib tashlash
    // Stack trace faqat AppError yaratilgan joydan boshlanadi
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, AppError);
    }
  }
}

// Custom assertion utility
function assert(condition, message) {
  if (!condition) {
    const error = new Error(message || "Assertion failed");
    // Stack trace'dan assert funksiyasini olib tashlash
    // Foydalanuvchi stack trace'da assert() emas, uni CHAQIRGAN joyni ko'radi
    if (Error.captureStackTrace) {
      Error.captureStackTrace(error, assert);
    }
    throw error;
  }
}

// assert(false, "User topilmadi");
// Error: User topilmadi
//     at processUser (app.js:42:3)    ← assert ni chaqirgan joy
//     at main (app.js:10:1)
// "at assert" frame yo'q — toza stack trace
```

```javascript
// === Stack trace formatting va stackTraceLimit ===

// V8 da stack trace limit o'zgartirish
Error.stackTraceLimit = 50; // Default 10 dan ko'proq

// Stack trace formatlash — logga yozish uchun
function formatStackTrace(error) {
  const lines = error.stack.split("\n");
  return {
    error: lines[0],                      // "TypeError: ..."
    frames: lines.slice(1).map(line => {  // "    at func (file:line:col)"
      const match = line.match(/at\s+(.+?)\s+\((.+):(\d+):(\d+)\)/);
      if (match) {
        return {
          function: match[1],
          file: match[2],
          line: parseInt(match[3]),
          column: parseInt(match[4])
        };
      }
      return { raw: line.trim() };
    })
  };
}

try {
  undefinedFunction();
} catch (e) {
  const formatted = formatStackTrace(e);
  console.log(formatted.error);
  // "ReferenceError: undefinedFunction is not defined"
  console.log(formatted.frames[0]);
  // { function: "Object.<anonymous>", file: "app.js", line: 1, column: 3 }
}
```

</details>

---

## Console API

### Nazariya

Console API — debugging va development uchun browser va Node.js da mavjud bo'lgan xabar chiqarish interfeysi. `console` global obyekt bo'lib, turli xil log levels, formatting, performance o'lchash va guruhlash imkoniyatlarini taqdim etadi.

**Log levels:**
- `console.log()` — umumiy xabar, info darajasida
- `console.info()` — informatsion xabar (ko'p browser'larda `log` bilan bir xil)
- `console.warn()` — ogohlantirish, sariq rang bilan ko'rsatiladi
- `console.error()` — xato, qizil rang bilan, stack trace bilan ko'rsatiladi

**console.table()** — array yoki object'ni jadval (table) ko'rinishida chop etadi. Array of objects bilan ishlashda juda qulay — har bir property alohida ustun bo'ladi. Ikkinchi argument sifatida ko'rsatmoqchi bo'lgan ustunlar ro'yxatini berish mumkin.

**console.time() / console.timeEnd()** — ikki nuqta orasidagi vaqtni o'lchash. `time("label")` timer'ni boshlaydi, `timeEnd("label")` to'xtatib natijani chop etadi. `timeLog("label")` oraliq natijani ko'rsatadi, timer'ni to'xtatmaydi.

**console.group() / console.groupEnd()** — log xabarlarni vizual guruhlash. `groupCollapsed()` — default yopiq holda guruh yaratadi. Nested group'lar qo'llab-quvvatlanadi.

**console.count() / console.countReset()** — berilgan label necha marta chaqirilganini sanaydi. Default label `"default"`. Debug paytida funksiya necha marta ishlaganini tekshirish uchun foydali.

**console.assert()** — birinchi argument `false` (yoki falsy) bo'lgandagina xabar chop etadi. `true` bo'lsa — hech narsa qilmaydi. Shartli debugging uchun qulay — `if` yozish shart emas.

**console.trace()** — joriy nuqtadan stack trace'ni chop etadi, xato throw qilmasdan. Funksiya qayerdan chaqirilganini aniqlash uchun foydali.

**console.dir()** — obyektni interactive (kengaytiriladigan) ko'rinishda ko'rsatadi. DOM element bilan `console.log()` HTML ko'rsatadi, `console.dir()` esa JavaScript obyekt sifatida ko'rsatadi.

**console.clear()** — konsolni tozalaydi. Node.js da terminal'ni tozalaydi.

**%c — CSS styling** — faqat browser'da ishlaydi. `console.log("%cMatn", "color: red; font-size: 20px")` — birinchi argument'dagi `%c` dan keyingi matn ikkinchi argument'dagi CSS bilan stillanadi.

**Production da console:** `console.log` chaqiruvlari production'da performance'ga ta'sir qiladi (serialization cost), xotira leak'ga sabab bo'lishi mumkin (DevTools ochiq bo'lganda logged object'lar GC qilinmaydi — console'da reference saqlanadi), va maxfiy ma'lumotlar ko'rinib qolishi mumkin. Build vaqtida `terser` yoki `babel-plugin-transform-remove-console` bilan olib tashlash kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === console.table — array va object'larni jadval ko'rinishda ===

const users = [
  { name: "Ali", age: 25, city: "Toshkent" },
  { name: "Vali", age: 30, city: "Samarqand" },
  { name: "Guli", age: 22, city: "Buxoro" }
];

console.table(users);
// ┌─────────┬────────┬─────┬─────────────┐
// │ (index) │  name  │ age │    city      │
// ├─────────┼────────┼─────┼─────────────┤
// │    0    │ 'Ali'  │ 25  │ 'Toshkent'  │
// │    1    │ 'Vali' │ 30  │ 'Samarqand' │
// │    2    │ 'Guli' │ 22  │ 'Buxoro'    │
// └─────────┴────────┴─────┴─────────────┘

// Faqat kerakli ustunlarni ko'rsatish
console.table(users, ["name", "city"]);
// Faqat name va city ustunlari ko'rinadi
```

```javascript
// === console.time — performance o'lchash ===

console.time("array-sort");
const arr = Array.from({ length: 1_000_000 }, () => Math.random());
arr.sort((a, b) => a - b);
console.timeEnd("array-sort");
// array-sort: 234.567ms

// timeLog — oraliq natijani ko'rish (timer davom etadi)
console.time("process");
// ... birinchi bosqich
console.timeLog("process", "1-bosqich tugadi");
// process: 45.123ms 1-bosqich tugadi
// ... ikkinchi bosqich
console.timeEnd("process");
// process: 120.456ms
```

```javascript
// === CSS styling bilan log (faqat browser) ===

console.log(
  "%c✅ SUCCESS %cUser yaratildi",
  "background: green; color: white; padding: 2px 6px; border-radius: 3px; font-weight: bold;",
  "color: green; font-weight: bold;"
);

console.log(
  "%c⚠️ WARNING %cSession 5 daqiqada tugaydi",
  "background: orange; color: white; padding: 2px 6px; border-radius: 3px;",
  "color: orange;"
);

// Katta banner
console.log(
  "%cMening Ilovam v2.0",
  "font-size: 24px; font-weight: bold; color: #3498db; text-shadow: 2px 2px #eee;"
);
```

```javascript
// === Custom logger wrapper — production-ready ===

const Logger = {
  _levels: { debug: 0, info: 1, warn: 2, error: 3, silent: 4 },
  _currentLevel: "info", // production da "error" yoki "silent"

  _shouldLog(level) {
    return this._levels[level] >= this._levels[this._currentLevel];
  },

  setLevel(level) {
    if (level in this._levels) this._currentLevel = level;
  },

  debug(...args) {
    if (this._shouldLog("debug")) {
      console.log("%c[DEBUG]", "color: gray;", ...args);
    }
  },

  info(...args) {
    if (this._shouldLog("info")) {
      console.info("%c[INFO]", "color: blue;", ...args);
    }
  },

  warn(...args) {
    if (this._shouldLog("warn")) {
      console.warn("%c[WARN]", "color: orange;", ...args);
    }
  },

  error(...args) {
    if (this._shouldLog("error")) {
      console.error("%c[ERROR]", "color: red;", ...args);
    }
  },

  // Performance o'lchash
  time(label) { console.time(label); },
  timeEnd(label) { console.timeEnd(label); },

  // Guruhlash
  group(label) { console.group(label); },
  groupEnd() { console.groupEnd(); },

  // Jadval
  table(data, columns) { console.table(data, columns); }
};

// Ishlatish
Logger.setLevel("debug"); // Development da
Logger.debug("User data:", { id: 1, name: "Ali" });
Logger.info("Server ishga tushdi", { port: 3000 });
Logger.warn("Deprecated API ishlatilmoqda");
Logger.error("Database ulanish xatosi", new Error("ECONNREFUSED"));

Logger.setLevel("error"); // Production da — faqat error ko'rinadi
```

```javascript
// === console.group, console.count, console.assert ===

// Guruhlash — nested log'lar
console.group("API So'rov");
  console.log("URL: /api/users");
  console.log("Method: GET");
  console.group("Headers");
    console.log("Content-Type: application/json");
    console.log("Authorization: Bearer ...");
  console.groupEnd();
  console.log("Status: 200 OK");
console.groupEnd();

// Count — necha marta chaqirildi
function handleClick(button) {
  console.count(button); // har bir button uchun alohida hisoblagich
}
handleClick("save");   // save: 1
handleClick("cancel"); // cancel: 1
handleClick("save");   // save: 2
handleClick("save");   // save: 3
console.countReset("save"); // hisoblagichni nolga qaytarish

// Assert — shart bajarilmasa log
const age = 15;
console.assert(age >= 18, "Foydalanuvchi voyaga yetmagan:", { age });
// Assertion failed: Foydalanuvchi voyaga yetmagan: { age: 15 }

const validAge = 25;
console.assert(validAge >= 18, "Bu xabar chop etilMAYDI");
// Hech narsa — shart true, assert jim

// Trace — stack trace (xato throw qilmasdan)
function a() { b(); }
function b() { c(); }
function c() { console.trace("Qayerdan chaqirildi?"); }
a();
// Trace: Qayerdan chaqirildi?
//     at c (app.js:3)
//     at b (app.js:2)
//     at a (app.js:1)
```

</details>

---

## Edge Cases va Gotchas

Error handling bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha.

### Gotcha 1: `return somePromise` vs `return await somePromise` — try/catch bilan farq

`try/catch` ichidan Promise qaytarish — ko'p dasturchilar bilmaydigan nuance. `return somePromise` **catch'ni by-pass qiladi** — Promise rejection caller'ga "oqib" o'tadi, current try/catch uni ushlamaydi. `return await somePromise` — `await` funksiya ichida Promise'ni "consume" qiladi, rejection sync throw'ga aylanadi, va catch uni ushlaydi.

```javascript
async function fetchUser() {
  throw new Error("fetch failed");
}

// ❌ return — catch ISHLAMAYDI!
async function getUser1() {
  try {
    return fetchUser(); // ← await YO'Q
  } catch (error) {
    // Hech qachon bu yerga tushmaydi!
    console.log("Caught:", error); // ishlamaydi
    return null;
  }
}

const result1 = await getUser1(); // UnhandledPromiseRejection yoki throw caller'ga
// Xato caller'ga propagate bo'ldi — catch block by-pass qilindi

// ✅ return await — catch ISHLAYDI
async function getUser2() {
  try {
    return await fetchUser(); // ← await BOR
  } catch (error) {
    console.log("Caught:", error.message); // "fetch failed"
    return null;
  }
}

const result2 = await getUser2(); // null — catch handled xatoni
```

**Nima uchun:** `return somePromise` — funksiya darhol qaytadi, Promise resolution/rejection caller'ga o'tadi. `async` funksiya ichida `try/catch` faqat **joriy execution context**'dagi throw'larni ushlaydi — qaytarilgan Promise allaqachon funksiyadan tashqarida. `return await` esa `await` expression'i ichida rejection'ni throw'ga aylantiradi, bu throw hali funksiya ichida sodir bo'ladi va catch'ga tushadi.

**Yechim:** Try/catch ichida Promise qaytarayotgan bo'lsangiz — **har doim `return await`** ishlating. Kichik performance cost bor (extra microtask), lekin aniq error handling muhim. Linter'lar (ESLint `no-return-await` eskirgan, `@typescript-eslint/return-await` yangilari `return-await: "in-try-catch"` pattern'ni tavsiya qiladi).

### Gotcha 2: Wrap + rethrow → original stack trace yo'qolishi

Error'ni yangi Error'ga "wrap" qilish keng tarqalgan pattern, lekin `new Error("wrapped")` **o'z stack trace'ini** yaratadi — wrap bo'lgan original xatoning stack'i yo'qoladi. Debugging paytida "qaysi funksiyada boshlandi?" savoliga javob yo'qoladi.

```javascript
async function lowLevel() {
  throw new Error("Connection refused");
  // Stack: at lowLevel (db.js:5)
  //        at ...
}

async function highLevel() {
  try {
    await lowLevel();
  } catch (error) {
    // ❌ Bad — original stack trace yo'qoladi:
    throw new Error("High level operation failed");
    // Stack: at highLevel (service.js:10) ← FAQAT bu ko'rinadi
    //        lowLevel frame'i yo'q — original sabab noma'lum
  }
}

// ✅ Good — { cause } bilan original'ni saqlash (ES2022):
async function highLevelFixed() {
  try {
    await lowLevel();
  } catch (error) {
    throw new Error("High level operation failed", { cause: error });
    // error.cause → original Error (stack bilan)
    // error.cause.stack → lowLevel stack saqlanadi
  }
}

// ✅ Error chain'ni to'liq log qilish:
function logErrorChain(error) {
  let current = error;
  while (current) {
    console.error(`[${current.name}] ${current.message}`);
    console.error(current.stack);
    current = current.cause;
    if (current) console.error("--- Caused by: ---");
  }
}

try {
  await highLevelFixed();
} catch (e) {
  logErrorChain(e);
  // [Error] High level operation failed
  //     at highLevelFixed (service.js:10)
  // --- Caused by: ---
  // [Error] Connection refused
  //     at lowLevel (db.js:5)
}
```

**Nima uchun:** Har `new Error()` chaqiruv V8 tomonidan o'z stack trace'ini capture qiladi — **hozirgi call stack**'dan, originaldan emas. Ikki Error'ni "birlashtirish" uchun explicit reference kerak. ES2022 gacha dasturchilar `error.originalError = e` yoki `error.stack += "\nCaused by: " + e.stack` kabi manual pattern'lar ishlatgan. `{ cause }` standart yechim sifatida qo'shildi.

**Yechim:** Har wrap + rethrow paytida `{ cause: originalError }` ishlating. Stack trace formatting kerak bo'lsa — custom helper bilan `error.cause`'ni recursively log qiling. Sentry, Datadog kabi monitoring tool'lar `cause` chain'ni avtomatik qo'llab-quvvatlaydi.

### Gotcha 3: `async function` ichida `throw` — sync emas, rejected Promise

`async function f() { throw new Error(); }` — bu **sync throw** emas. `async` funksiya **har doim** `Promise` qaytaradi, shuning uchun throw rejected promise'ga aylanadi. Caller `.catch()` yoki `try/await` ishlatishi shart — fire-and-forget chaqiruv xatoni silent yutadi yoki `unhandledrejection` beradi.

```javascript
async function validate(data) {
  if (!data.email) {
    throw new Error("Email kerak"); // ← bu sync throw emas!
  }
  return true;
}

// ❌ Sync throw kutish:
try {
  validate({}); // Promise qaytadi — throw BO'LMAYDI
  console.log("No error"); // bu chiqadi (validate resolve bo'lishini kutmaymiz)
} catch (error) {
  // Bu yerga hech qachon tushmaydi — async throw catch qilmaydi sync try/catch'da
  console.log("Caught:", error.message);
}
// Console'da: "No error"
// Va asinxron: UnhandledPromiseRejection warning/error

// ❌ Fire-and-forget — eng xavfli:
validate({}); // ← Promise silent reject
// Hech qanday warning bo'lmasa ham — dastur state inconsistent

// ✅ try/catch + await:
try {
  await validate({}); // await rejected promise'ni sync throw'ga aylantiradi
  console.log("Valid");
} catch (error) {
  console.log("Caught:", error.message); // "Email kerak"
}

// ✅ .catch() chaining:
validate({}).catch(error => {
  console.log("Caught:", error.message);
});

// ─── Validator function — sync vs async semantikasi ───
// Agar siz validator funksiyasi yozayotgan bo'lsangiz va real asinxron ish
// yo'q bo'lsa (masalan, faqat type tekshirish) — sync function yozing:
function validateSync(data) {
  if (!data.email) {
    throw new Error("Email kerak"); // ← sync throw, caller'da oddiy try/catch
  }
  return true;
}

try {
  validateSync({}); // sync throw — darhol catch'ga o'tadi
} catch (error) {
  console.log("Caught sync:", error.message); // ishlaydi ✅
}
```

**Nima uchun:** `async function` spec bo'yicha har doim `Promise` wrap qiladi — hatto `return 42` ham `Promise.resolve(42)` ga aylanadi. `throw` esa `Promise.reject(error)` ga aylanadi. Bu semantikani tushunmaslik — "mening funksiyam sync ishlaydi, async kalit so'zi shunchaki syntax sugar" deb o'ylash — eng ko'p uchraydigan async bug manbai.

**Yechim:** `async` kalit so'zini **faqat** real asynchronous ish bo'lganda (fetch, file I/O, timer) ishlating. Pure validation, synchronous computation — sync function yozing. Async funksiya chaqirganda har doim `await` yoki `.catch()` — fire-and-forget'dan saqlaning.

### Gotcha 4: Promise constructor'da sync throw ushlanadi, async callback'da yo'q

Promise constructor'ga berilgan executor funksiya ichida **sync throw** — Promise reject bo'ladi (constructor internally catches). Lekin shu executor ichidagi **async callback'da** (setTimeout, event listener) throw — constructor uni ushlay olmaydi, chunki unga yetgunicha constructor allaqachon tugagan. Bu kutilmagan unhandled rejection'ga olib keladi.

```javascript
// ✅ Sync throw executor ichida — ushlanadi, Promise reject bo'ladi
const p1 = new Promise((resolve, reject) => {
  throw new Error("sync throw"); // ← constructor ichida, ushlanadi
});

p1.catch(e => console.log("p1 caught:", e.message)); // "p1 caught: sync throw" ✅

// ❌ Async callback ichida throw — constructor ushlamaydi
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    throw new Error("async throw"); // ← constructor tugagan, throw yetmaydi
  }, 100);
});

p2.catch(e => console.log("p2 caught:", e.message));
// ❌ Hech qachon ushlanmaydi!
// Console'da: Uncaught (in promise) — lekin bu global handler uchun,
// p2.catch() trigger bo'lmaydi chunki p2 hali pending holatda
// Haqiqatan: UnhandledException (setTimeout callback'dan)

// ✅ Yechim: async callback ichida try/catch + reject
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    try {
      riskyOperation();
      resolve();
    } catch (error) {
      reject(error); // ← explicit reject
    }
  }, 100);
});

p3.catch(e => console.log("p3 caught:", e.message)); // ishlaydi ✅

// ─── Real example: fs.readFile callback ───
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) {
        reject(err); // ✅ error-first callback → reject
        return;
      }
      try {
        const parsed = JSON.parse(data); // parse throw bo'lishi mumkin
        resolve(parsed);
      } catch (parseError) {
        reject(parseError); // ✅ sync throw'ni ham reject qilish
      }
    });
  });
}
```

**Nima uchun:** Promise constructor executor funksiyasi **sync** chaqiriladi — constructor `new Promise(...)` ichida ishlaydi va uning throw'ini try/catch qiladi. Lekin setTimeout, addEventListener, fs callback kabi **asynchronous** context'dagi throw — constructor'dan tashqarida bo'ladi, execution stack allaqachon boshqa. JavaScript'da cross-async-context throw yo'q — har async boundary o'zining error boundary'si bo'lishi kerak.

**Yechim:** Promise ichida async callback ishlatganda — har doim `try/catch` bilan o'rang va xatoni explicit `reject()` orqali propagate qiling. Yoki zamonaviy `util.promisify` yoki promise-based API'lar (fs/promises, `fetch`) — ular allaqachon to'g'ri error handling bilan yozilgan.

### Gotcha 5: `JSON.stringify(error)` → `"{}"`, `cause` chain ham yo'qoladi

`Error.prototype.message`, `name`, `stack`, `cause` — barchasi **non-enumerable** property'lar. `JSON.stringify` faqat enumerable own property'larni serialize qiladi, shuning uchun `JSON.stringify(new Error("msg"))` → `"{}"`. Logger, monitoring, API'ga error yuborganda bu juda ko'p ma'lumot yo'qotadi.

```javascript
const error = new Error("Top error", {
  cause: new TypeError("Nested type error", {
    cause: new RangeError("Deep range error")
  })
});

// ❌ JSON.stringify — bo'sh obyekt:
console.log(JSON.stringify(error));
// "{}"
// message/stack/name/cause — hammasi non-enumerable, yo'qoldi

// ❌ `Object.keys(error)` ham bo'sh:
console.log(Object.keys(error)); // [] — enumerable property yo'q
// Lekin:
console.log(error.message); // "Top error" — property mavjud, lekin non-enumerable

// ✅ Yechim 1: Custom replacer — recursive cause bilan
function serializeError(error) {
  if (!error) return null;

  const result = {
    name: error.name,
    message: error.message,
    stack: error.stack,
  };

  // Custom enumerable property'lar (masalan status code):
  for (const key of Object.keys(error)) {
    result[key] = error[key];
  }

  // Cause chain (ES2022) — recursive
  if (error.cause !== undefined) {
    result.cause = serializeError(error.cause);
  }

  return result;
}

console.log(JSON.stringify(serializeError(error), null, 2));
// {
//   "name": "Error",
//   "message": "Top error",
//   "stack": "Error: Top error\n    at ...",
//   "cause": {
//     "name": "TypeError",
//     "message": "Nested type error",
//     "stack": "TypeError: ...",
//     "cause": {
//       "name": "RangeError",
//       "message": "Deep range error",
//       "stack": "RangeError: ..."
//     }
//   }
// }

// ✅ Yechim 2: JSON.stringify replacer function
const json = JSON.stringify(error, (key, value) => {
  if (value instanceof Error) {
    return {
      name: value.name,
      message: value.message,
      stack: value.stack,
      ...(value.cause && { cause: value.cause })
    };
  }
  return value;
}, 2);

// ✅ Yechim 3: Node.js `util.inspect(error, { showHidden: true })` —
// human-readable format, serialize uchun emas
const util = require("util");
console.log(util.inspect(error, { depth: null, colors: false }));
```

**Nima uchun:** ECMAScript spec `Error.prototype` property'larini (`message`, `name`) non-enumerable qiladi — bu pattern "internal" property'larni `for...in` va `Object.keys()` dan yashirish uchun. `JSON.stringify` algoritmi `EnumerableOwnPropertyNames` ishlatadi, non-enumerable'larni o'tkazib yuboradi. Bu design by convention — sizning Error'ga qo'shgan enumerable property'laringiz (masalan `error.statusCode = 500`) saqlanadi, lekin built-in property'lar yo'qoladi.

**Yechim:** Error'ni log/serialize qilishdan oldin **doim custom serializer** orqali o'tkazing. Monitoring tool'lari (Sentry, Datadog) bu muammoni o'z tomonlarida hal qilgan — Error object'ni tanib, to'g'ri ishlaydi. Lekin o'z logging'ingizda `JSON.stringify` ga to'g'ridan-to'g'ri Error bermang.

---

## Common Mistakes

### ❌ Xato 1: Barcha xatolarni yutib yuborish (empty catch)

```javascript
// ❌ Noto'g'ri — xato haqida hech narsa qilinmaydi
try {
  criticalOperation();
} catch (error) {
  // Bo'sh catch — "swallowing errors"
  // Xato yo'qoldi — dastur noto'g'ri ishlaydi, lekin sababi noma'lum
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Kamida log qilish
try {
  criticalOperation();
} catch (error) {
  console.error("Critical operation failed:", error);
  // Yoki monitoring service ga yuborish
  // Yoki foydalanuvchiga xabar berish
}
```

**Nima uchun:** Bo'sh catch blok xatolarni "yutib yuboradi" — dastur kutilmagan holatda ishlaydi, lekin siz hech qachon bilmaysiz nima uchun. Eng kamida `console.error` qiling — production da bu log'lar debugging uchun yagona manba bo'lishi mumkin.

---

### ❌ Xato 2: Error o'rniga string throw qilish

```javascript
// ❌ Noto'g'ri — string throw
throw "Nimadir noto'g'ri ketdi!";
// catch da: typeof error === "string", error.stack === undefined
```

### ✅ To'g'ri usul:

```javascript
// ✅ Error obyekti throw
throw new Error("Nimadir noto'g'ri ketdi!");
// catch da: error.stack — to'liq stack trace, error.message — xabar
```

**Nima uchun:** `Error` obyekti `stack` property beradi — xato qayerda, qaysi funksiyalar zanjirida sodir bo'lganini ko'rsatadi. String'da bu ma'lumot yo'q — debugging deyarli mumkin emas. `instanceof` tekshiruvi ham ishlamaydi.

---

### ❌ Xato 3: async funksiya xatosini ushlamaslik

```javascript
// ❌ Noto'g'ri — async funksiya natijasi catch'siz
async function riskyAsync() {
  throw new Error("Async xato!");
}
riskyAsync(); // ❌ UnhandledPromiseRejection warning/error!
```

### ✅ To'g'ri usul:

```javascript
// ✅ .catch() bilan
riskyAsync().catch(error => console.error(error));

// ✅ Yoki try/catch + await
try {
  await riskyAsync();
} catch (error) {
  console.error(error);
}
```

**Nima uchun:** `async` funksiya **Promise** qaytaradi. Promise reject bo'lganda catch bo'lmasa — `UnhandledPromiseRejection` sodir bo'ladi. Node.js 15+ da bu process'ni crash qiladi. Browser'da console warning chiqadi va xato yo'qoladi.

---

### ❌ Xato 4: finally ichida return

```javascript
// ❌ finally ichida return — throw ni yo'qotadi
function getData() {
  try {
    throw new Error("muhim xato!");
  } finally {
    return "default"; // ❌ Error yo'qoldi — hech qachon tashqariga chiqmaydi!
  }
}
console.log(getData()); // "default" — xato sezilmaydi
```

### ✅ To'g'ri usul:

```javascript
// ✅ finally da faqat cleanup, return yo'q
function getData() {
  try {
    throw new Error("muhim xato!");
  } catch (error) {
    console.error(error);
    return "default"; // catch da return — xavfsiz
  } finally {
    // Faqat cleanup: file.close(), connection.end(), timer clear
  }
}
```

**Nima uchun:** `finally` ichidagi `return` try/catch'dagi barcha `return` va `throw`'ni override qiladi. Xatolar "yo'qoladi" — dastur noto'g'ri ishlaydi va sababi topilmaydi.

---

### ❌ Xato 5: try/catch'ni asinxron callback ichida kutish

```javascript
// ❌ try/catch asinxron callback ni ushlamaydi
try {
  setTimeout(() => {
    throw new Error("Bu ushlanmaydi!"); // ❌ boshqa execution context
  }, 0);
} catch (error) {
  console.log("Ushlandimi?"); // ❌ Bu yerga hech qachon tushmaydi
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Callback ichida o'zining try/catch'ini qo'yish
setTimeout(() => {
  try {
    throw new Error("Bu ushlanadi!");
  } catch (error) {
    console.error("Ushlandi:", error.message); // ✅ ishlaydi
  }
}, 0);

// ✅ Yoki Promise + async/await ishlatish
await new Promise((resolve, reject) => {
  setTimeout(() => {
    try {
      riskyOperation();
      resolve();
    } catch (error) {
      reject(error); // ← Promise reject orqali xatoni uzatish
    }
  }, 0);
});
```

**Nima uchun:** `setTimeout` callback'i boshqa execution context'da (keyingi event loop tick'da) bajariladi. O'sha paytda try/catch allaqachon tugagan. Har bir asinxron callback o'zining xato boshqaruv mexanizmiga ega bo'lishi kerak.

---

## Amaliy Mashqlar

### Mashq 1: Safe JSON Parse (Oson)

**Savol:** `safeJsonParse(str, fallback)` funksiyasi yarating. JSON parse muvaffaqiyatli bo'lsa natijani, xato bo'lsa `fallback` ni qaytarsin. Faqat `SyntaxError` ni ushlang — boshqa xatolarni qayta throw qiling.

<details>
<summary>Javob</summary>

```javascript
function safeJsonParse(str, fallback = null) {
  try {
    return JSON.parse(str);
  } catch (error) {
    if (error instanceof SyntaxError) {
      return fallback;
    }
    throw error; // SyntaxError bo'lmasa — qayta throw
  }
}

// Test:
console.log(safeJsonParse('{"name":"Ali"}'));   // { name: "Ali" }
console.log(safeJsonParse("invalid", {}));      // {}
console.log(safeJsonParse("null"));             // null (valid JSON)
console.log(safeJsonParse(undefined, []));      // [] (undefined parse → SyntaxError)
console.log(safeJsonParse('"hello"'));           // "hello" (valid JSON string)
```

**Tushuntirish:** `JSON.parse` faqat `SyntaxError` throw qiladi — shuning uchun boshqa xatolarni qayta throw qilish to'g'ri. `null`, `"hello"`, `42` — valid JSON qiymatlari, xato bermaydi.

</details>

---

### Mashq 2: Custom Error Hierarchy (O'rta)

**Savol:** Quyidagi error hierarchy yarating va `handleError(error)` funksiyasi bilan HTTP response qaytaring:
- `AppError` — base: `statusCode`, `isOperational`
- `ValidationError` — `field` property, statusCode: 400
- `NotFoundError` — `resource` property, statusCode: 404
- `DatabaseError` — statusCode: 500, `isOperational: false`

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
    this.isOperational = false; // critical — restart kerak bo'lishi mumkin
    if (cause) this.cause = cause;
  }
}

function handleError(error) {
  if (error instanceof ValidationError) {
    return { status: 400, body: { error: error.message, field: error.field } };
  }
  if (error instanceof NotFoundError) {
    return { status: 404, body: { error: error.message } };
  }
  if (!error.isOperational) {
    console.error("CRITICAL:", error);
    // Alert yuborish, process restart ko'rib chiqish
  }
  return {
    status: error.statusCode || 500,
    body: { error: error.isOperational ? error.message : "Server xatosi" }
  };
}

// Test:
console.log(handleError(new ValidationError("Email noto'g'ri", "email")));
// { status: 400, body: { error: "Email noto'g'ri", field: "email" } }

console.log(handleError(new NotFoundError("Foydalanuvchi")));
// { status: 404, body: { error: "Foydalanuvchi topilmadi" } }

console.log(handleError(new DatabaseError("Connection lost")));
// CRITICAL: DatabaseError: Connection lost
// { status: 500, body: { error: "Server xatosi" } }
```

</details>

---

### Mashq 3: Retry with Backoff (O'rta)

**Savol:** `retry(fn, options)` funksiyasi yarating: `maxRetries` (default: 3), `delay` (default: 1000ms), `backoff` koeffitsienti (default: 2), `onRetry(attempt, error, waitTime)` callback.

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

  throw new Error(`${maxRetries + 1} urinishdan keyin muvaffaqiyatsiz`, {
    cause: lastError
  });
}

// Test:
let callCount = 0;
const result = await retry(
  async () => {
    callCount++;
    if (callCount < 3) throw new Error(`Urinish ${callCount} xato`);
    return "Muvaffaqiyat!";
  },
  {
    maxRetries: 5,
    delay: 100,
    onRetry: (attempt, error, wait) => {
      console.log(`#${attempt} xato: ${error.message}, ${wait}ms kutilmoqda`);
    }
  }
);
console.log(result); // "Muvaffaqiyat!" — 3-chi urinishda
// Output:
// #1 xato: Urinish 1 xato, 100ms kutilmoqda
// #2 xato: Urinish 2 xato, 200ms kutilmoqda
// Muvaffaqiyat!
```

**Tushuntirish:** `attempt` 0 dan boshlanadi, `maxRetries` marta qayta urinadi. Exponential backoff: `delay * 2^attempt` — 100ms, 200ms, 400ms, 800ms, ... Server'ni haddan tashqari yuklamaslik uchun.

</details>

---

### Mashq 4: Error Boundary Wrapper (Qiyin)

**Savol:** `ErrorBoundary` class yarating: `wrap(fn)` — funksiyani wrap qilib xatolarni ushlaydi (sync va async), `onError(callback)` — xato handler, `getErrors()` — barcha xatolar ro'yxati.

<details>
<summary>Javob</summary>

```javascript
class ErrorBoundary {
  #errors = [];
  #handlers = [];

  onError(callback) {
    this.#handlers.push(callback);
    return this; // chaining uchun
  }

  #handleError(error, context) {
    const entry = {
      error,
      context,
      timestamp: new Date().toISOString()
    };
    this.#errors.push(entry);

    for (const handler of this.#handlers) {
      try {
        handler(entry);
      } catch (handlerError) {
        console.error("Error handler da xato:", handlerError);
      }
    }
  }

  wrap(fn, context = fn.name || "anonymous") {
    return (...args) => {
      try {
        const result = fn.apply(null, args);

        // Agar Promise qaytarsa — async xatolarni ham ushlash
        if (result && typeof result.catch === "function") {
          return result.catch(error => {
            this.#handleError(error, context);
            return undefined;
          });
        }

        return result;
      } catch (error) {
        this.#handleError(error, context);
        return undefined;
      }
    };
  }

  getErrors() {
    return [...this.#errors]; // copy qaytarish — tashqi o'zgartirish oldini olish
  }

  clear() {
    this.#errors = [];
  }
}

// Test:
const boundary = new ErrorBoundary();
boundary.onError(({ error, context, timestamp }) => {
  console.log(`[${timestamp}] ${context}: ${error.message}`);
});

// Sinxron
const safeParse = boundary.wrap(JSON.parse, "JSON.parse");
safeParse("noto'g'ri json"); // xato ushlandi, undefined qaytardi
safeParse('{"ok": true}');    // { ok: true }

// Async
const safeFetch = boundary.wrap(async (url) => {
  const r = await fetch(url);
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}, "fetchData");

await safeFetch("/api/404"); // xato ushlandi

console.log(boundary.getErrors().length); // 2
```

**Tushuntirish:** `wrap()` return qilgan funksiya sync va async xatolarni ushlaydi. Sync — try/catch bilan, async — `.catch()` bilan (Promise duck typing: `typeof result.catch === "function"`). Private `#errors` va `#handlers` — tashqi koddan himoyalangan.

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Error Types** | SyntaxError, TypeError, ReferenceError, RangeError — har biri aniq xato kategoriyasi, `instanceof` bilan tekshiriladi |
| **Error Object** | `name`, `message`, `stack` — stack trace debugging uchun eng muhim ma'lumot |
| **Error cause** | `new Error("msg", { cause })` — xato zanjirini saqlash, debugging osonlashtirish (ES2022) |
| **try/catch/finally** | finally DOIM ishlaydi, finally da return YOZMANG — throw/return ni override qiladi |
| **throw** | Doim `Error` yoki subclass throw qiling — string throw qilish stack trace yo'qotadi |
| **Custom Errors** | `extends Error` — `instanceof` zanjir tekshiruv, operational vs programmer error ajratish |
| **Propagation** | Xato catch bo'lmaguncha call stack bo'ylab yuqoriga ko'tariladi (stack unwinding) |
| **Async Errors** | `.catch()`, `try/catch` + `await`, `Promise.allSettled` — har bir async operatsiyada error handling |
| **Global Handlers** | `window.onerror`, `unhandledrejection` — oxirgi himoya qatlami, monitoring uchun |
| **Result Type** | `{ ok, data, error }` — throw o'rniga explicit natija, control flow xatolari uchun |
| **Retry** | Exponential backoff — vaqtinchalik (transient) xatolar uchun qayta urinish |
| **Circuit Breaker** | Ko'p xatodan keyin so'rovlarni to'xtatish — tashqi servisni himoya qilish |
| **Graceful Degradation** | API → cache → default — foydalanuvchi xato ko'rmasligi uchun |
| **Fail Fast** | Guard clauses — argumentlarni boshida tekshirish, erta va aniq xato xabari |

> **Keyingi bo'lim:** [21-modern-js.md](21-modern-js.md) — Destructuring, spread/rest, template literals, optional chaining, nullish coalescing, RegExp, JSON va boshqa zamonaviy ES6+ xususiyatlar.

> **Cross-references:** [12-promises.md](12-promises.md) (Promise rejection, async error flow), [13-async-await.md](13-async-await.md) (try/catch + await, retry pattern), [08-classes.md](08-classes.md) (class extends — custom error hierarchy), [11-event-loop.md](11-event-loop.md) (microtask — unhandledrejection timing), [19.5-browser-apis.md](19.5-browser-apis.md) (Fetch — network errors, AbortController)