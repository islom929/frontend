# Bo'lim 24: Design Patterns

> Design pattern — dasturiy ta'minotda takroriy uchraydigan muammolarning isbotlangan, qayta ishlatilishi mumkin bo'lgan arxitekturaviy yechimlari. JavaScript da pattern'lar uchta kategoriyaga bo'linadi: Creational (yaratish), Structural (tuzilish), Behavioral (xatti-harakat).

---

## Mundarija

### Creational Patterns
- [Factory Pattern](#factory-pattern)
- [Singleton Pattern](#singleton-pattern)
- [Builder Pattern](#builder-pattern)

### Structural Patterns
- [Module Pattern](#module-pattern)
- [Decorator Pattern](#decorator-pattern)
- [Adapter Pattern](#adapter-pattern)
- [Facade Pattern](#facade-pattern)
- [Proxy Pattern](#proxy-pattern)

### Behavioral Patterns
- [Observer Pattern](#observer-pattern)
- [Pub/Sub Pattern](#pubsub-pattern)
- [Strategy Pattern](#strategy-pattern)
- [Command Pattern](#command-pattern)
- [Mediator Pattern](#mediator-pattern)
- [State Pattern](#state-pattern)
- [Chain of Responsibility](#chain-of-responsibility)
- [Iterator Pattern](#iterator-pattern)

### Yakuniy
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

# CREATIONAL PATTERNS

Creational pattern'lar — object yaratish jarayonini boshqaradi. To'g'ridan-to'g'ri `new` yoki literal bilan yaratish o'rniga, yaratish logikasini markazlashtiradi, moslashuvchan qiladi va code coupling'ni kamaytiradi.

---

## Factory Pattern

### Nazariya

Factory pattern — object yaratish logikasini bitta joyga markazlashtiradi. Client kod qaysi class yoki constructor ishlatilayotganini bilmasligi kerak — faqat factory'ga "nima kerak" deydi, factory tegishli object'ni yaratib beradi. Bu code coupling'ni kamaytiradi: yangi object turi qo'shish uchun faqat factory'ni o'zgartirish yetarli, client kodga tegmaslik mumkin.

**Muammo:** Object yaratish murakkablashganda — turli constructor'lar, turli konfiguratsiyalar, conditional logic — yaratish kodi tarqalib ketadi va har joyda takrorlanadi.

**Yechim:** Bitta funksiya yoki method barcha yaratish logikasini o'z ichiga oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Factory funksiya har safar chaqirilganda yangi object literal yaratadi. V8 bu object'larni **Hidden Class** (V8 source kodida `Map` deb nomlanadi — JavaScript'dagi `Map` data structure'dan butunlay boshqa tushuncha) orqali optimize qiladi — agar factory doim bir xil property tartibida object qaytarsa, barcha natijalar bir xil hidden class'ga ega bo'ladi va property access monomorphic IC (inline cache) bilan tezlashadi. Lekin `switch` ichida har xil shape'dagi object'lar qaytarilsa (masalan, admin'da `canManageUsers` bor, viewer'da yo'q), har bir tur alohida hidden class oladi — bu megamorphic access'ga olib keladi.

Class-based factory'da `new` operator constructor'ni chaqirganda V8 oldindan hidden class tayyor qiladi. Har bir subclass (`EmailNotification`, `SMSNotification`) o'zining prototype chain'iga ega — factory client koddan bu prototype tafsilotlarini yashiradi. Factory funksiya closure bo'lsa, ichki class reference'lar closure'ning `[[Environment]]` record'ida saqlanadi — bu GC tomonidan factory reference mavjud ekan tozalanmaydi.

Factory pattern'da memory perspective — har bir chaqiruv yangi object allocate qiladi. Agar method'lar object literal ichida aniqlansa, har bir object uchun yangi function object yaratiladi. Class-based yondashuvda method'lar prototype'da bitta marta yaratiladi va barcha instance'lar share qiladi — bu memory efficient.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Oddiy factory — turga qarab object yaratish:

```javascript
// ❌ Client kod har xil class'larni bilishi kerak
const admin = new AdminUser("Ali", "admin@mail.com");
const regular = new RegularUser("Vali", "vali@mail.com");
const guest = new GuestUser();

// ✅ Factory — client faqat turini aytadi
function createUser(type, data = {}) {
  switch (type) {
    case "admin":
      return {
        ...data,
        role: "admin",
        permissions: ["read", "write", "delete", "manage"],
        canManageUsers: true
      };
    case "editor":
      return {
        ...data,
        role: "editor",
        permissions: ["read", "write"],
        canManageUsers: false
      };
    case "viewer":
      return {
        ...data,
        role: "viewer",
        permissions: ["read"],
        canManageUsers: false
      };
    default:
      throw new Error(`Noma'lum user turi: ${type}`);
  }
}

const admin = createUser("admin", { name: "Ali", email: "ali@mail.com" });
const editor = createUser("editor", { name: "Vali", email: "vali@mail.com" });
// Yangi tur qo'shish — faqat factory'ga case qo'shish
```

Class-based factory — murakkab object'lar uchun:

```javascript
class Notification {
  constructor(message) { this.message = message; }
  send() { throw new Error("send() implement qilinmagan"); }
}

class EmailNotification extends Notification {
  constructor(message, to) {
    super(message);
    this.to = to;
    this.type = "email";
  }
  send() {
    console.log(`Email → ${this.to}: ${this.message}`);
  }
}

class SMSNotification extends Notification {
  constructor(message, phone) {
    super(message);
    this.phone = phone;
    this.type = "sms";
  }
  send() {
    console.log(`SMS → ${this.phone}: ${this.message}`);
  }
}

class PushNotification extends Notification {
  constructor(message, deviceId) {
    super(message);
    this.deviceId = deviceId;
    this.type = "push";
  }
  send() {
    console.log(`Push → ${this.deviceId}: ${this.message}`);
  }
}

// Factory
class NotificationFactory {
  static create(type, message, target) {
    switch (type) {
      case "email": return new EmailNotification(message, target);
      case "sms":   return new SMSNotification(message, target);
      case "push":  return new PushNotification(message, target);
      default: throw new Error(`Noma'lum notification turi: ${type}`);
    }
  }
}

// Client kod — qaysi class ishlatilganini bilmaydi
const notification = NotificationFactory.create("email", "Salom!", "ali@mail.com");
notification.send(); // Email → ali@mail.com: Salom!
```

**Real-world:** React'da `createElement()`, document'da `createElement()`, Express'da middleware yaratish — bular factory pattern.

**Qachon ishlatish:** Object yaratish logikasi murakkab yoki conditional bo'lganda, yangi turlar tez-tez qo'shiladigan bo'lsa, client kodning concrete class'lardan ajratilishi kerak bo'lganda.

**Qachon ishlatmaslik:** Bitta oddiy constructor yetarli bo'lganda — keraksiz abstraction qo'shmaslik.

</details>

---

## Singleton Pattern

### Nazariya

Singleton — bitta class'dan faqat **bitta instance** yaratilishini kafolatlaydi va bu instance'ga global access point beradi. Database connection, configuration manager, logger, cache — bular singleton'ga mos tushadi, chunki bir nechta instance yaratish resurs isrof qiladi yoki nomuvofiqlik keltirib chiqaradi.

**Muammo:** Ba'zi resurslarga (DB connection pool, app config) dasturning turli joylaridan kirish kerak, lekin bir nechta instance yaratish xavfli — holat sinxronlashmasligi, resurs isrofi.

**Yechim:** Class o'zi faqat bitta instance yaratilishini nazorat qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

ES Module singleton'ning asosi — **Module Map** cache. Node.js yoki browser module loader `import` qilganda module specifier'ni Module Map'da qidiradi. Agar topilsa, qayta execute qilmaydi — cached module'ning `[[Namespace]]` object'ini qaytaradi. Shuning uchun `export const config = new AppConfig()` faqat bir marta bajariladi — bu singleton'ning eng ishonchli mexanizmi.

`static #instance` approach'da `constructor` ichidagi `if (Database.#instance) return Database.#instance` — bu `new` operator'ning natijasini override qiladi. `new` odatda yangi object allocate qiladi, lekin constructor explicit object qaytarsa, `new` shu qaytarilgan object'ni beradi (spec: `[[Construct]]` internal method). Private static field `#instance` class body scope'da yashaydi — tashqaridan mutatsiya qilish imkonsiz.

`Object.freeze(instance)` qo'shilsa, object'ning property descriptor'lari `writable: false`, `configurable: false` ga o'zgaradi. V8 frozen object'larni ichki **constant** sifatida belgilaydi — bu property access'ni yanada tezlashtiradi, chunki engine property o'zgarmasligini biladi va deoptimization trigger bo'lmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

ES Modules bilan singleton — **eng oddiy va tavsiya qilinadigan** usul. ES Module o'zi singleton — bir marta bajariladi va cache'lanadi:

```javascript
// config.js — module scope = singleton
class AppConfig {
  constructor() {
    this.settings = {
      apiUrl: "https://api.example.com",
      timeout: 5000,
      retries: 3
    };
  }

  get(key) {
    return this.settings[key];
  }

  set(key, value) {
    this.settings[key] = value;
  }
}

// Module scope'da bitta instance yaratiladi
// Boshqa fayllar import qilganda — doim shu instance
export const config = new AppConfig();
```

```javascript
// app.js
import { config } from "./config.js";
config.set("apiUrl", "https://staging.api.com");

// server.js — boshqa faylda import — AYNAN SHU instance
import { config } from "./config.js";
console.log(config.get("apiUrl")); // "https://staging.api.com" — o'zgangan!
```

Class-based singleton — instance'ni class ichida nazorat qilish:

```javascript
class Database {
  static #instance = null;

  constructor(connectionString) {
    if (Database.#instance) {
      return Database.#instance; // mavjud instance qaytarish
    }
    this.connectionString = connectionString;
    this.pool = [];
    Database.#instance = this;
  }

  static getInstance(connectionString) {
    if (!Database.#instance) {
      Database.#instance = new Database(connectionString);
    }
    return Database.#instance;
  }

  query(sql) {
    console.log(`[${this.connectionString}] ${sql}`);
  }
}

const db1 = Database.getInstance("postgres://localhost/mydb");
const db2 = Database.getInstance("postgres://localhost/other"); // ignorlanadi
console.log(db1 === db2); // true — bitta instance
db1.query("SELECT * FROM users");
```

**Qachon ishlatish:** Database connection pool, app configuration, logger, cache store, event bus.

**Qachon ishlatmaslik:** Unit test'larda muammo qiladi (global state), dependency injection qiyinlashadi. Keraksiz singleton — code smell. ES Modules bilan oddiy export ko'p holatlarda yetarli.

</details>

---

## Builder Pattern

### Nazariya

Builder pattern — murakkab object'ni **qadam-baqadam** (step-by-step) yaratish imkonini beradi. Constructor'ga 10+ parameter berish o'rniga, har bir konfiguratsiyani alohida method bilan belgilash mumkin. Method chaining bilan oson o'qiladi.

**Muammo:** Constructor parametrlari ko'payib ketganda — qaysi argument nima ekanini tushunish qiyin bo'ladi, optional parametrlar uchun `null`/`undefined` uzatish kerak bo'ladi.

**Yechim:** Builder object har bir konfiguratsiyani alohida method bilan qabul qiladi va oxirida `build()` bilan final object'ni qaytaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Method chaining'ning asosi — har bir method `return this` qiladi. `this` reference shu object'ga ishora qiladi, yangi object yaratilmaydi. V8 bu chained call'larni optimize qiladi: `from()`, `select()`, `where()` ketma-ket chaqirilganda, har safar bir xil receiver (`this`) bo'lgani uchun inline cache (IC) monomorphic holatda qoladi va method lookup tez ishlaydi.

Private field'lar (`#table`, `#fields`) V8 ichida class instance'ning internal slot'larida saqlanadi. Bu property'lar hidden class'da ko'rinmaydi — ular alohida `PrivateName` identifier orqali access qilinadi. Shuning uchun builder'ning ichki holati tashqaridan `Object.keys()` yoki `for...in` bilan ko'rinmaydi.

`build()` chaqirilganda yangi string concatenation bo'ladi. V8 string concatenation'ni `ConsString` (rope structure) bilan lazy bajaradi — haqiqiy birlashma faqat string o'qilganda sodir bo'ladi. Builder pattern'da intermediate object'lar (builder o'zi) `build()` dan keyin reference bo'lmasa GC tomonidan tozalanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Muammo — 8 ta parameter, qaysi biri nima?
const query = new Query("users", ["name", "email"], "age > 18", "name",
                         "ASC", 10, 0, true);

// ✅ Builder — har bir qadam aniq
class QueryBuilder {
  #table = "";
  #fields = ["*"];
  #conditions = [];
  #orderBy = null;
  #orderDir = "ASC";
  #limitVal = null;
  #offsetVal = 0;

  from(table) {
    this.#table = table;
    return this; // method chaining uchun
  }

  select(...fields) {
    this.#fields = fields.length ? fields : ["*"];
    return this;
  }

  where(condition) {
    this.#conditions.push(condition);
    return this;
  }

  orderBy(field, direction = "ASC") {
    this.#orderBy = field;
    this.#orderDir = direction;
    return this;
  }

  limit(count) {
    this.#limitVal = count;
    return this;
  }

  offset(count) {
    this.#offsetVal = count;
    return this;
  }

  build() {
    if (!this.#table) throw new Error("Table nomi kerak (from)");

    let sql = `SELECT ${this.#fields.join(", ")} FROM ${this.#table}`;
    if (this.#conditions.length) {
      sql += ` WHERE ${this.#conditions.join(" AND ")}`;
    }
    if (this.#orderBy) {
      sql += ` ORDER BY ${this.#orderBy} ${this.#orderDir}`;
    }
    if (this.#limitVal !== null) {
      sql += ` LIMIT ${this.#limitVal}`;
    }
    if (this.#offsetVal > 0) {
      sql += ` OFFSET ${this.#offsetVal}`;
    }
    return sql;
  }
}

const query = new QueryBuilder()
  .from("users")
  .select("name", "email", "age")
  .where("age > 18")
  .where("active = true")
  .orderBy("name", "ASC")
  .limit(10)
  .offset(20)
  .build();

console.log(query);
// SELECT name, email, age FROM users WHERE age > 18 AND active = true ORDER BY name ASC LIMIT 10 OFFSET 20
```

**Real-world:** Knex.js query builder, Joi schema builder, Jest test matchers, DOM API (`document.createElement` + setAttribute + appendChild kabi chain).

**Qachon ishlatish:** Object yaratish 4+ parameter talab qilganda, optional konfiguratsiyalar ko'p bo'lganda, method chaining orqali oson o'qiladigan API kerak bo'lganda.

</details>

---

# STRUCTURAL PATTERNS

Structural pattern'lar — object va class'larni qanday tuzish va birlashtirish haqida. Mavjud object'larga yangi funksionallik qo'shish, interface'larni moslashtirish va murakkab tizimlarni soddalashtirish uchun ishlatiladi.

---

## Module Pattern

### Nazariya

Module pattern — ma'lumotlarni **encapsulate** qilish (yashirish) va faqat kerakli qismini ochiq qoldirish. JavaScript da ES Modules standart bo'lishidan oldin, IIFE (Immediately Invoked Function Expression) bilan closure orqali module hosil qilish asosiy usul edi. Hozir ES Modules (`import`/`export`) standart yechim.

**Muammo:** Global scope pollution — barcha o'zgaruvchilar global scope'da bo'lsa, nom to'qnashuvi (collision) va ma'lumot ochiq qolib ketishi.

**Yechim:** IIFE + closure yoki ES Module scope bilan private/public farqlash.

<details>
<summary><strong>Under the Hood</strong></summary>

IIFE `(function() { ... })()` bajarilganda engine yangi **Execution Context** yaratadi va uning **Environment Record**'ida barcha ichki variable'lar (`count`, `MAX`, `validate`) saqlanadi. Qaytarilgan object'ning method'lari (`increment`, `decrement`) bu Environment Record'ga closure orqali reference saqlaydi. Tashqi kod bu record'ga to'g'ridan-to'g'ri kira olmaydi — bu encapsulation'ning engine-level mexanizmi.

IIFE tugagandan keyin uning Execution Context stack'dan chiqadi, lekin Environment Record GC tomonidan tozalanmaydi — chunki qaytarilgan object'ning funksiyalari `[[Environment]]` internal slot orqali unga reference saqlaydi. Bu closure'ning memory implication'i — private variable'lar module reference mavjud ekan hayotda qoladi.

ES Module'da engine boshqacha ishlaydi — module body'si `Module Environment Record`'da bajariladi. Bu record'dagi `export` qilinmagan binding'lar (`let count`) module tashqarisidan spec bo'yicha accessible emas. `import { increment }` qilganda — bu reference copy emas, **live binding** — eksport qiluvchi module'dagi o'zgaruvchiga to'g'ridan-to'g'ri bog'langan.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

IIFE-based module (eski usul, lekin tushunchani bilish kerak):

```javascript
const Counter = (function() {
  // Private — tashqaridan kirib bo'lmaydi
  let count = 0;
  const MAX = 100;

  function validate(n) {
    if (n > MAX) throw new RangeError(`Maximum ${MAX}`);
    if (n < 0) throw new RangeError("Manfiy bo'lishi mumkin emas");
  }

  // Public API — qaytarilgan object
  return {
    increment() {
      validate(count + 1);
      return ++count;
    },
    decrement() {
      validate(count - 1);
      return --count;
    },
    getCount() {
      return count;
    },
    reset() {
      count = 0;
      return count;
    }
  };
})();

Counter.increment(); // 1
Counter.increment(); // 2
Counter.getCount();  // 2
// Counter.count;    // undefined — private
// Counter.MAX;      // undefined — private
```

ES Module (zamonaviy usul):

```javascript
// counter.js
let count = 0; // module scope — private
const MAX = 100;

function validate(n) { /* ... */ }

export function increment() { return ++count; }
export function decrement() { return --count; }
export function getCount() { return count; }
export function reset() { count = 0; return count; }
```

**Revealing Module Pattern** — barcha funksiyalarni private yozib, faqat keraklisini export/return qilish. Bu nom o'zgartirishni osonlashtiradi va public API bir joyda ko'rinadi.

</details>

---

## Decorator Pattern

### Nazariya

Decorator — mavjud object yoki funksiyaning **asl kodini o'zgartirmay** unga yangi funksionallik qo'shish. Object yoki funksiyani boshqa funksiya bilan "o'rab olish" (wrapping). Bu Open/Closed printsipiga (SOLID) mos — kengaytirish uchun ochiq, o'zgartirish uchun yopiq.

**Muammo:** Har bir yangi funksionallik uchun original kodni o'zgartirish — kod murakkablashadi, regression riski ortadi.

**Yechim:** Wrapper funksiya yoki class — original'ni ichiga oladi, yangi behavior qo'shadi, original operatsiyani ham chaqiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Funksiya decorator pattern'da `withLogging(fn)` yangi funksiya qaytaradi — bu funksiya closure orqali original `fn` reference'ni `[[Environment]]` record'ida saqlaydi. Har bir decorator qatlami bitta qo'shimcha closure frame va bitta yangi function object yaratadi. `withLogging(withTiming(fn))` — bu 2 ta nested closure, 2 ta function object allocation.

`fn.apply(this, args)` chaqiruvi muhim — bu `this` binding'ni to'g'ri uzatadi. `apply` ichida engine `[[Call]]` internal method'ni `this` va `args` bilan chaqiradi. Agar `this` uzatilmasa, decorator wrapped funksiyaning context'ini yo'qotadi. Spread operator (`...args`) rest parameter orqali arguments'ni Array ga yig'adi — bu Arguments object yaratishdan ko'ra V8 da samaraliroq.

Class-level decorator (TC39 Stage 3) prototype chain'ni extend qiladi yoki property descriptor'larni o'zgartiradi. `Object.defineProperty` orqali method'ning `value`, `writable`, `configurable` attribute'lari qayta yoziladi. V8 bu o'zgarishni hidden class transition sifatida qayd qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Funksiya decorator — asl funksiyani o'zgartirmay yangi xatti-harakat qo'shish:

```javascript
// Logging decorator — har bir chaqiruvni loglash
function withLogging(fn, label) {
  return function(...args) {
    console.log(`[${label}] Chaqirildi:`, args);
    const result = fn.apply(this, args);
    console.log(`[${label}] Natija:`, result);
    return result;
  };
}

// Timing decorator — bajarilish vaqtini o'lchash
function withTiming(fn, label) {
  return function(...args) {
    const start = performance.now();
    const result = fn.apply(this, args);
    const duration = (performance.now() - start).toFixed(2);
    console.log(`[${label}] ${duration}ms`);
    return result;
  };
}

// Retry decorator — xato bo'lganda qayta urinish
function withRetry(fn, maxRetries = 3) {
  return async function(...args) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await fn.apply(this, args);
      } catch (error) {
        if (attempt === maxRetries) throw error;
        console.log(`Urinish ${attempt}/${maxRetries} muvaffaqiyatsiz, qayta urinish...`);
        await new Promise(r => setTimeout(r, attempt * 1000));
      }
    }
  };
}

// Asl funksiya o'zgarmaydi — decorator'lar ustiga qo'shiladi
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price * item.qty, 0);
}

const trackedCalc = withLogging(withTiming(calculateTotal, "Total"), "Calc");
trackedCalc([{ price: 100, qty: 2 }, { price: 50, qty: 3 }]);
// [Calc] Chaqirildi: [[{price:100,qty:2}, {price:50,qty:3}]]
// [Total] 0.05ms
// [Calc] Natija: 350
```

**Real-world:** Express middleware, NestJS `@Controller()`, `@Get()`, Angular decorator'lari, MobX `@observable`, TypeScript decorator'lar.

</details>

---

## Adapter Pattern

### Nazariya

Adapter — mos kelmaydigan ikki interface'ni birga ishlashga majburlaydi. Eski API'ni yangi format'ga yoki uchinchi tomon library'ni loyiha standartiga moslashtirish uchun ishlatiladi. Adapter asl object'ni wrap qiladi va kerakli interface'ni beradi.

**Muammo:** Ikki component farqli API bilan ishlaydi — birini boshqasiga moslashtirish kerak, lekin ikkalasini ham o'zgartirish imkoni yo'q (yoki o'zgartirish istalmaydi).

**Yechim:** Oraliq adapter object — bir interface'dan ikkinchisiga tarjima qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Adapter object composition pattern'ga asoslangan — adapter instance ichida adaptee (eski object) reference saqlanadi (`this.oldAPI`). Bu `constructor`'da property sifatida yoziladi va V8 hidden class'ga qo'shiladi. Property forwarding'da adapter method chaqirilganda, engine birinchi adapter'ning hidden class'idan method'ni topadi, keyin adapter ichida `this.oldAPI.processXMLPayment()` uchun yana property lookup sodir bo'ladi — bu 2 ta IC (inline cache) check.

Adapter qo'shimcha abstraction layer — har bir method chaqiruvi adapter orqali o'tganda bitta qo'shimcha function call frame qo'shiladi. Lekin V8 TurboFan optimizing compiler bu kichik method'larni **inline** qilishi mumkin — adapter method body'sini chaqiruvchi funksiya ichiga joylashtiradi, bu call overhead'ni yo'qotadi.

Memory jihatdan adapter bitta qo'shimcha object — adaptee reference va adapter method'lari uchun. Agar adapter ko'p instance yaratilmasa (odatda bitta), memory ta'siri minimal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eski API'ni yangi format'ga moslashtirish:

```javascript
// Eski API — XML format qaytaradi
class OldPaymentAPI {
  processXMLPayment(xmlData) {
    console.log("XML payment processed:", xmlData);
    return "<response><status>success</status><id>PAY-123</id></response>";
  }
}

// Yangi API — JSON format kutadi
// Adapter — eski API'ni yangi formatga moslashtiradi
class PaymentAdapter {
  constructor() {
    this.oldAPI = new OldPaymentAPI();
  }

  processPayment(jsonData) {
    // JSON → XML ga aylantirish
    const xml = `<payment><amount>${jsonData.amount}</amount><currency>${jsonData.currency}</currency></payment>`;
    const xmlResponse = this.oldAPI.processXMLPayment(xml);

    // XML → JSON ga aylantirish
    const statusMatch = xmlResponse.match(/<status>(\w+)<\/status>/);
    const idMatch = xmlResponse.match(/<id>([\w-]+)<\/id>/);
    return {
      success: statusMatch?.[1] === "success",
      transactionId: idMatch?.[1] || null
    };
  }
}

// Client kod — faqat JSON bilan ishlaydi
const payment = new PaymentAdapter();
const result = payment.processPayment({ amount: 100, currency: "USD" });
console.log(result); // { success: true, transactionId: "PAY-123" }
```

Third-party library adapter:

```javascript
// Turli log library'lar — har xil API
class WinstonLogger {
  info(msg) { console.log(`[Winston INFO] ${msg}`); }
  error(msg) { console.error(`[Winston ERROR] ${msg}`); }
  warn(msg) { console.warn(`[Winston WARN] ${msg}`); }
}

class PinoLogger {
  log(level, msg) { console.log(`[Pino ${level}] ${msg}`); }
}

// Unified adapter — barcha logger'lar uchun bitta interface
class LoggerAdapter {
  constructor(logger) {
    this.logger = logger;
  }

  info(msg) {
    if (this.logger.info) return this.logger.info(msg);
    if (this.logger.log) return this.logger.log("info", msg);
  }

  error(msg) {
    if (this.logger.error) return this.logger.error(msg);
    if (this.logger.log) return this.logger.log("error", msg);
  }

  warn(msg) {
    if (this.logger.warn) return this.logger.warn(msg);
    if (this.logger.log) return this.logger.log("warn", msg);
  }
}

// Client kod — qaysi logger ishlatilayotganini bilmaydi
const logger = new LoggerAdapter(new PinoLogger());
logger.info("Server started");  // [Pino info] Server started
logger.error("DB connection failed"); // [Pino error] DB connection failed
```

**Real-world:** Database ORM'lar (turli DB driver'larni yagona API orqali), `fetch` polyfill'lari, framework migration (jQuery → vanilla JS).

</details>

---

## Facade Pattern

### Nazariya

Facade — murakkab subsystem'ga **oddiy, yagona interface** beradi. Client kod subsystem'ning ichki tuzilishini bilmasligi kerak — faqat facade bilan ishlaydi. Bu coupling'ni kamaytiradi va API'ni sodda qiladi.

**Muammo:** Murakkab tizimda 5-10 ta turli component bilan ishlash kerak — har birining API'si boshqacha.

**Yechim:** Bitta facade class/funksiya barcha murakkablikni yashiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Facade engine darajasida maxsus hech narsa qilmaydi — bu sof architectural pattern. Facade constructor'da bir nechta subsystem object'larini yaratadi (`this.audio`, `this.video`), har biri facade instance'ning property'si sifatida hidden class'ga qo'shiladi. Method chaqirilganda facade ichidagi subsystem method'lariga delegation sodir bo'ladi.

Facade'ning memory cost'i — facade object + barcha subsystem instance'lari. Agar subsystem'lar og'ir bo'lsa (ko'p property, method), facade constructor'da barchasini eager yaratish costly bo'lishi mumkin. Lazy initialization (faqat kerak bo'lganda yaratish) bu muammoni hal qiladi.

V8 perspective'dan facade method'lari odatda bir nechta subsystem method'ni ketma-ket chaqiradi. TurboFan bu ketma-ket call'larni optimize qiladi — har bir subsystem property access uchun IC monomorphic bo'lsa (doim bir xil type), tezlashuv katta. Facade pattern'ning asosiy qiymati runtime performance'da emas, code organization'da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Murakkab subsystem — audio, video, subtitle, analytics
class AudioPlayer {
  load(src) { console.log(`Audio yuklandi: ${src}`); }
  play() { console.log("Audio ijro"); }
  pause() { console.log("Audio pauza"); }
  setVolume(level) { console.log(`Ovoz: ${level}%`); }
}

class VideoRenderer {
  load(src) { console.log(`Video yuklandi: ${src}`); }
  render() { console.log("Video render"); }
  stop() { console.log("Video to'xtadi"); }
  setQuality(q) { console.log(`Sifat: ${q}`); }
}

class SubtitleEngine {
  load(src) { console.log(`Subtitle yuklandi: ${src}`); }
  show() { console.log("Subtitle ko'rsatildi"); }
  hide() { console.log("Subtitle yashirildi"); }
}

class AnalyticsTracker {
  track(event, data) { console.log(`[Analytics] ${event}:`, data); }
}

// Facade — oddiy API
class MediaPlayer {
  constructor() {
    this.audio = new AudioPlayer();
    this.video = new VideoRenderer();
    this.subtitles = new SubtitleEngine();
    this.analytics = new AnalyticsTracker();
  }

  play(videoSrc, audioSrc, subtitleSrc) {
    this.video.load(videoSrc);
    this.audio.load(audioSrc);
    if (subtitleSrc) this.subtitles.load(subtitleSrc);
    this.video.setQuality("1080p");
    this.audio.setVolume(75);
    this.video.render();
    this.audio.play();
    if (subtitleSrc) this.subtitles.show();
    this.analytics.track("play", { video: videoSrc });
  }

  stop() {
    this.video.stop();
    this.audio.pause();
    this.subtitles.hide();
    this.analytics.track("stop", {});
  }
}

// Client kod — murakkab subsystem'ni bilmaydi
const player = new MediaPlayer();
player.play("movie.mp4", "movie.aac", "movie.srt");
player.stop();
```

**Real-world:** jQuery (`$` — DOM, AJAX, animation, event'larni bitta API ga birlashtiradi), `fetch` (XMLHttpRequest ustidan facade), React'da `ReactDOM.render()`, Node.js `fs.readFile` (low-level syscall'lar ustidan).

</details>

---

## Proxy Pattern

### Nazariya

Proxy pattern — object'ga kirishni nazorat qilish uchun oraliq qatlam. JS Proxy (23-bo'lim) bilan to'g'ridan-to'g'ri implement qilish mumkin, lekin pattern sifatida tushuncha kengroq — lazy initialization, access control, caching va logging. Proxy asl object o'rniga turadi va client kodga shaffof bo'ladi.

**Muammo:** Object'ga har qanday kirishni nazorat qilish, qo'shimcha logic qo'shish (cache, auth, log) kerak, lekin asl object'ni o'zgartirish istalmaydi.

**Yechim:** Proxy object asl object'ning interface'ini saqlaydi, lekin kirish oldidan/keyin qo'shimcha logic bajaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

JavaScript `Proxy` — spec'da **exotic object** deb ataladi. Oddiy object'lardan farqi — `[[Get]]`, `[[Set]]`, `[[HasProperty]]` kabi internal method'lar trap handler'lar orqali intercept qilinadi. V8 Proxy object uchun oddiy hidden class optimization'larni to'liq qo'llay olmaydi — har bir property access trap funksiyani chaqiradi, inline cache ko'p hollarda ishlamaydi. Shuning uchun Proxy ordinary object'dan sezilarli darajada sekinroq — aniq farq V8 versiyasi, trap complexity va operation turiga bog'liq, hot path'da seziladigan darajada bo'lishi mumkin.

`new Proxy(target, handler)` yaratilganda engine ichki `[[ProxyHandler]]` va `[[ProxyTarget]]` internal slot'larni saqlaydi. `get` trap chaqirilganda `Reflect.get(target, prop, receiver)` spec bo'yicha target object'ning `[[Get]]` internal method'ini to'g'ridan-to'g'ri chaqiradi — bu trap ichida target'ning asl behavior'ini saqlash uchun ishlatiladi.

Virtual proxy (class-based, `Proxy` ishlatmagan) oddiy object — V8 hidden class optimization to'liq ishlaydi. `this.realImage = null` dan `this.realImage = new HeavyImage()` ga o'zgarganda hidden class transition sodir bo'ladi. Lazy initialization memory-efficient — og'ir object faqat kerak bo'lganda heap'da allocate qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Virtual proxy — lazy initialization (og'ir resursni faqat kerak bo'lganda yaratish):

```javascript
class HeavyImage {
  constructor(url) {
    this.url = url;
    // Og'ir operatsiya — haqiqiy yuklash
    console.log(`[HeavyImage] ${url} yuklanmoqda... (5MB)`);
    this.data = `image-data-of-${url}`;
  }
  display() {
    console.log(`[HeavyImage] ${this.url} ko'rsatilmoqda`);
  }
}

class ImageProxy {
  constructor(url) {
    this.url = url;
    this.realImage = null; // hali yuklanmagan
  }
  display() {
    // faqat birinchi marta kerak bo'lganda yuklanadi (lazy)
    if (!this.realImage) {
      this.realImage = new HeavyImage(this.url);
    }
    this.realImage.display();
  }
}

const images = [
  new ImageProxy("photo1.jpg"),
  new ImageProxy("photo2.jpg"),
  new ImageProxy("photo3.jpg")
];
// Hech biri yuklanmagan — faqat proxy yaratilgan

images[0].display();
// [HeavyImage] photo1.jpg yuklanmoqda... (5MB)
// [HeavyImage] photo1.jpg ko'rsatilmoqda

images[0].display();
// [HeavyImage] photo1.jpg ko'rsatilmoqda  (qayta yuklanmaydi — cached)

// images[1] va images[2] hali yuklanmagan — resurs tejaldi
```

JS `Proxy` bilan caching proxy:

```javascript
function createCachingProxy(apiClient, ttlMs = 60000) {
  const cache = new Map();

  return new Proxy(apiClient, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      if (typeof value !== "function") return value;

      return async function(...args) {
        const key = `${prop}:${JSON.stringify(args)}`;
        const cached = cache.get(key);

        if (cached && Date.now() - cached.timestamp < ttlMs) {
          return cached.data; // cache'dan
        }

        const result = await value.apply(target, args);
        cache.set(key, { data: result, timestamp: Date.now() });
        return result;
      };
    }
  });
}
```

</details>

---

# BEHAVIORAL PATTERNS

Behavioral pattern'lar — object'lar orasidagi aloqa va javobgarlik taqsimoti haqida. Qaysi object qanday xabar yuboradi, kim qabul qiladi, javobgarlik qanday uzatiladi — shu muammolarni hal qiladi.

---

## Observer Pattern

### Nazariya

Observer pattern — **bitta subject** (kuzatiluvchi) holati o'zgarganda, unga **obuna bo'lgan barcha observer'lar** (kuzatuvchilar) avtomatik xabar oladi. Bu **bir-ko'p** (one-to-many) aloqa. Subject observer'lar ro'yxatini boshqaradi — `subscribe`, `unsubscribe`, `notify`. Observer'lar subject bilan **tight coupling** — to'g'ridan-to'g'ri obuna bo'ladi.

**Muammo:** Bitta object holati o'zgarganda bir nechta boshqa object'larni xabardor qilish kerak, lekin ular orasida qattiq bog'liqlik bo'lmasligi kerak.

**Yechim:** Subject observer'lar ro'yxatini saqlaydi, holat o'zganganda barchasiga `notify` qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Observer'lar `Map<string, Function[]>` strukturasida saqlanadi. Har bir `on()` chaqiruvida callback funksiya array'ga push qilinadi — bu funksiya reference sifatida saqlanadi, copy emas. Subject bu callback'larga **strong reference** saqlaydi, shuning uchun observer object boshqa joyda reference yo'qolsa ham GC tomonidan tozalanmaydi — bu memory leak'ning asosiy sababi.

`emit()` da `[...listeners].forEach()` — spread operator bilan copy olish muhim. Agar iterate paytida `once` listener o'zini arraydan o'chirsa, original array'ning length'i o'zgaradi va boshqa listener'lar skip bo'lishi mumkin. Shallow copy bu muammoni hal qiladi, lekin har bir emit'da yangi array allocate qiladi.

`once()` implementatsiyasi closure pattern — wrapper funksiya original listener'ni closure'da saqlaydi. `wrapper._original = listener` — bu qo'shimcha property off() da original listener bo'yicha qidirish uchun. V8 funksiyaga dynamic property qo'shganda hidden class transition sodir bo'ladi. Ko'p listener'lar ro'yxatdan o'tganda `indexOf()` O(n) — katta listener array'larda `Set` ishlatish samaraliroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

EventEmitter — JavaScript'dagi Observer pattern'ning standart implementatsiyasi:

```javascript
class EventEmitter {
  #events = new Map();

  on(event, listener) {
    if (!this.#events.has(event)) {
      this.#events.set(event, []);
    }
    this.#events.get(event).push(listener);
    return this; // chaining uchun
  }

  off(event, listener) {
    const listeners = this.#events.get(event);
    if (!listeners) return this;
    const index = listeners.indexOf(listener);
    if (index > -1) listeners.splice(index, 1);
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      listener(...args);
      this.off(event, wrapper);
    };
    wrapper._original = listener; // off bilan o'chirish uchun
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.#events.get(event);
    if (!listeners || listeners.length === 0) return false;
    // listeners ni copy qilish — iterate paytida o'zgarmasligi uchun (once)
    [...listeners].forEach(listener => listener(...args));
    return true;
  }

  listenerCount(event) {
    return this.#events.get(event)?.length || 0;
  }

  removeAllListeners(event) {
    if (event) {
      this.#events.delete(event);
    } else {
      this.#events.clear();
    }
    return this;
  }
}

// Ishlatish
const store = new EventEmitter();

function onUserLogin(user) {
  console.log(`${user.name} tizimga kirdi`);
}

store.on("login", onUserLogin);
store.on("login", (user) => {
  console.log(`Analytics: login event — ${user.email}`);
});
store.once("login", () => {
  console.log("Birinchi login — welcome email yuborildi");
});

store.emit("login", { name: "Ali", email: "ali@mail.com" });
// Ali tizimga kirdi
// Analytics: login event — ali@mail.com
// Birinchi login — welcome email yuborildi

store.emit("login", { name: "Vali", email: "vali@mail.com" });
// Vali tizimga kirdi
// Analytics: login event — vali@mail.com
// (once listener yo'q — bir marta ishladi)
```

**Real-world:** Node.js `EventEmitter`, DOM `addEventListener`, RxJS Observable, React state management (setState → re-render).

</details>

---

## Pub/Sub Pattern

### Nazariya

Pub/Sub (Publish/Subscribe) — Observer pattern'ga o'xshash, lekin **asosiy farq**: publisher va subscriber bir-birini **bilmaydi**. Ular orasida **event bus** (message broker, channel) turadi. Publisher event bus'ga event yuboradi, subscriber event bus'dan event'larni eshitadi. Bu **loose coupling** — component'lar bir-biriga bog'liq emas.

**Observer vs Pub/Sub:**

| | Observer | Pub/Sub |
|--|---------|---------|
| Coupling | Tight — observer subject'ni biladi | Loose — publisher/subscriber bir-birini bilmaydi |
| Mediator | Yo'q — to'g'ridan-to'g'ri aloqa | Bor — event bus / message broker |
| Scalability | Kichik tizimlar uchun mos | Katta, distributed tizimlar uchun |

<details>
<summary><strong>Under the Hood</strong></summary>

EventBus `Map<string, Set<Function>>` ishlatadi — `Set` Observer'dagi `Array`'dan farqli ravishda duplicate handler'larni oldini oladi va `delete()` O(1) — `indexOf()` + `splice()` ning O(n) dan tezroq. Lekin `Set` iteration tartibi insertion order bo'yicha — bu spec kafolati (ES2015+).

Pub/Sub'da memory leak muammosi Observer'dan ham kuchliroq — chunki event bus odatda global (module scope'da), u barcha subscriber funksiyalarga strong reference saqlaydi. Agar subscriber component destroy bo'lsa lekin unsubscribe qilinmasa, bus orqali butun component va uning closure chain'i memory'da qoladi. `WeakRef` bilan subscriber saqlash bu muammoni hal qiladi — `WeakRef` GC'ga object'ni tozalashga ruxsat beradi, lekin har safar `deref()` qilish va null check kerak.

`subscribe()` unsubscribe funksiyani qaytaradi — bu closure orqali `channel` va `handler` reference'larni saqlaydi. Bu pattern memory-efficient: alohida "subscription ID" yoki lookup structure kerak emas — closure o'zi yetarli. `try/catch` har bir handler atrofida muhim — bitta handler'ning xatosi boshqalarni bloklamasligi uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class EventBus {
  #channels = new Map();

  subscribe(channel, handler) {
    if (!this.#channels.has(channel)) {
      this.#channels.set(channel, new Set());
    }
    this.#channels.get(channel).add(handler);

    // unsubscribe funksiyasi
    return () => {
      this.#channels.get(channel)?.delete(handler);
    };
  }

  publish(channel, data) {
    const handlers = this.#channels.get(channel);
    if (!handlers) return;
    handlers.forEach(handler => {
      try {
        handler(data);
      } catch (error) {
        console.error(`[EventBus] "${channel}" handler xatosi:`, error);
      }
    });
  }
}

// Global event bus
const bus = new EventBus();

// Module A — foydalanuvchi tizimga kirdi
// Module B ni bilmaydi
function loginModule() {
  // ... login logic ...
  bus.publish("user:login", { id: 1, name: "Ali" });
}

// Module B — login event'ga react qiladi
// Module A ni bilmaydi
const unsub = bus.subscribe("user:login", (user) => {
  console.log(`Salom, ${user.name}!`);
});

bus.subscribe("user:login", (user) => {
  console.log(`[Analytics] User #${user.id} logged in`);
});

loginModule();
// Salom, Ali!
// [Analytics] User #1 logged in

unsub(); // birinchi subscriber olib tashlandi
```

**Real-world:** Redis Pub/Sub, WebSocket event handling, React Context + useReducer, Vuex/Pinia actions, microservice'lar arasi communication, RabbitMQ, Kafka.

</details>

---

## Strategy Pattern

### Nazariya

Strategy — bir xil vazifani bajarishning **bir nechta usuli** (algoritm, strategiya) bor bo'lganda, ularni almashtirish (swap) mumkin qiladi. Har bir algoritm alohida strategy object sifatida ajratiladi. Client kod runtime da qaysi strategy ishlatilishini tanlaydi.

**Muammo:** if/else yoki switch bilan algoritm tanlash — yangi algoritm qo'shishda barcha conditional'larni o'zgartirish kerak bo'ladi (Open/Closed principle buziladi).

**Yechim:** Har bir algoritm alohida strategy, context object kerakli strategy'ni ishlatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

JavaScript'da strategy'lar odatda object literal yoki funksiya sifatida ifodalanadi. `strategies` object'dagi har bir property (`creditCard`, `paypal`) o'z hidden class'iga ega object'ga ishora qiladi. `strategies[strategyName]` — bu computed property access, V8 buni megamorphic IC sifatida qayd qiladi (chunki key runtime'da o'zgaradi).

`this.#strategy = strategy` — bu reference almashish, copy emas. Eski strategy object'ga boshqa reference bo'lmasa GC tozalaydi. Strategy funksiya sifatida berilganda (`#strategy = fn`), context object closure capture qilmaydi — faqat funksiya reference saqlanadi. Bu memory-efficient: har bir strategy almashtiruvda yangi allocation yo'q.

Strategy pattern `if/else` dan farqi engine level'da ham bor: `if/else` branch'lari uchun V8 branch prediction ishlatadi va ko'p branch'larda prediction miss ko'payadi. Object property lookup esa IC (inline cache) orqali optimize qilinadi. Lekin amalda bu farq micro-optimization — pattern'ning asosiy qiymati code maintainability'da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Strategy'lar — har biri alohida
const strategies = {
  creditCard: {
    name: "Kredit karta",
    validate(data) {
      return data.cardNumber?.length === 16 && data.cvv?.length === 3;
    },
    process(amount, data) {
      console.log(`Karta ${data.cardNumber.slice(-4)} dan $${amount} yechildi`);
      return { success: true, method: "credit_card" };
    }
  },

  paypal: {
    name: "PayPal",
    validate(data) {
      return Boolean(data.email && data.email.includes("@"));
    },
    process(amount, data) {
      console.log(`PayPal ${data.email} dan $${amount} yechildi`);
      return { success: true, method: "paypal" };
    }
  },

  crypto: {
    name: "Cryptocurrency",
    validate(data) {
      return Boolean(data.walletAddress && data.walletAddress.startsWith("0x"));
    },
    process(amount, data) {
      console.log(`Crypto ${data.walletAddress.slice(0, 10)}... ga $${amount} yuborildi`);
      return { success: true, method: "crypto" };
    }
  }
};

// Context — strategy bilan ishlaydi
class PaymentProcessor {
  #strategy = null;

  setStrategy(strategyName) {
    const strategy = strategies[strategyName];
    if (!strategy) throw new Error(`"${strategyName}" strategy topilmadi`);
    this.#strategy = strategy;
    return this;
  }

  pay(amount, data) {
    if (!this.#strategy) throw new Error("Strategy tanlanmagan");
    if (!this.#strategy.validate(data)) {
      throw new Error(`${this.#strategy.name} validatsiyadan o'tmadi`);
    }
    return this.#strategy.process(amount, data);
  }
}

const processor = new PaymentProcessor();

// Runtime da strategy almashtirish
processor.setStrategy("creditCard");
processor.pay(100, { cardNumber: "4111111111111111", cvv: "123" });

processor.setStrategy("paypal");
processor.pay(50, { email: "ali@mail.com" });

processor.setStrategy("crypto");
processor.pay(200, { walletAddress: "0xabc123def456" });
```

**Real-world:** Sorting algoritm'lari, authentication strategiyalari (JWT, OAuth, API key), validation, compression (gzip, brotli), caching (memory, Redis, file).

</details>

---

## Command Pattern

### Nazariya

Command — har bir amalni (action) **object** sifatida encapsulate qiladi. Bu undo/redo, command history, queue, va macro command'larni implement qilishni osonlashtiradi. Har bir command object'da `execute()` va ixtiyoriy `undo()` metodi bor.

**Muammo:** Amallarni kechiktirish (queue), qaytarish (undo), qayta bajarish (redo) yoki loglash kerak.

**Yechim:** Har bir amal alohida command object — execute() va undo() bilan.

<details>
<summary><strong>Under the Hood</strong></summary>

Har bir command — alohida class instance. `new AddTextCommand(editor, text, position)` chaqirilganda V8 yangi object allocate qiladi va constructor argument'larni instance property sifatida saqlaydi. Barcha `AddTextCommand` instance'lari bir xil hidden class'ga ega — `execute()` va `undo()` prototype'da share qilinadi, har bir instance uchun takrorlanmaydi.

Undo stack (`#history = []`) va redo stack (`#redoStack = []`) command object reference'larni saqlaydi. Har bir command o'z holat ma'lumotlarini (`this.text`, `this.position`, `this.deletedText`) saqlaydi — bu undo imkonini beradi. Memory perspective: uzoq editing session'da history stack cheksiz o'sishi mumkin. Real-world implementatsiyalarda stack size limit qo'yiladi — eski command'lar dequeue qilinadi va GC tozalaydi.

`this.#redoStack = []` — yangi amal bajarilganda redo stack butunlay yangi empty array bilan almashtiriladi. Eski array va undagi barcha command object'lar reference yo'qolgani uchun GC candidate bo'ladi. `splice` o'rniga yangi array assign qilish V8 da tezroq — eski array'ni tozalash GC'ga qoldiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Command interface — har bir command execute() va undo() ga ega
class AddTextCommand {
  constructor(editor, text, position) {
    this.editor = editor;
    this.text = text;
    this.position = position;
  }
  execute() {
    this.editor.insertAt(this.position, this.text);
  }
  undo() {
    this.editor.deleteAt(this.position, this.text.length);
  }
}

class DeleteTextCommand {
  constructor(editor, position, length) {
    this.editor = editor;
    this.position = position;
    this.length = length;
    this.deletedText = ""; // undo uchun saqlanadi
  }
  execute() {
    this.deletedText = this.editor.getTextAt(this.position, this.length);
    this.editor.deleteAt(this.position, this.length);
  }
  undo() {
    this.editor.insertAt(this.position, this.deletedText);
  }
}

// Text Editor
class TextEditor {
  #content = "";
  #history = [];     // undo stack
  #redoStack = [];   // redo stack

  insertAt(pos, text) {
    this.#content = this.#content.slice(0, pos) + text + this.#content.slice(pos);
  }

  deleteAt(pos, length) {
    this.#content = this.#content.slice(0, pos) + this.#content.slice(pos + length);
  }

  getTextAt(pos, length) {
    return this.#content.slice(pos, pos + length);
  }

  execute(command) {
    command.execute();
    this.#history.push(command);
    this.#redoStack = []; // yangi amal → redo stack tozalanadi
  }

  undo() {
    const command = this.#history.pop();
    if (!command) return;
    command.undo();
    this.#redoStack.push(command);
  }

  redo() {
    const command = this.#redoStack.pop();
    if (!command) return;
    command.execute();
    this.#history.push(command);
  }

  getContent() { return this.#content; }
}

const editor = new TextEditor();
editor.execute(new AddTextCommand(editor, "Salom ", 0));
console.log(editor.getContent()); // "Salom "

editor.execute(new AddTextCommand(editor, "dunyo!", 6));
console.log(editor.getContent()); // "Salom dunyo!"

editor.undo();
console.log(editor.getContent()); // "Salom "

editor.redo();
console.log(editor.getContent()); // "Salom dunyo!"

editor.execute(new DeleteTextCommand(editor, 0, 6));
console.log(editor.getContent()); // "dunyo!"

editor.undo();
console.log(editor.getContent()); // "Salom dunyo!"
```

**Real-world:** Text editor'lar (VS Code undo/redo), Git (commit = command), Redux action'lari, Task queue'lar.

</details>

---

## Mediator Pattern

### Nazariya

Mediator — bir nechta component orasidagi aloqani **markazlashtiradi**. Component'lar bir-biri bilan to'g'ridan-to'g'ri gaplashmaydi — barcha xabarlar mediator orqali o'tadi. Bu `n*(n-1)` bog'lanishni `n` ga kamaytiradi (har bir component faqat mediator bilan bog'liq).

**Muammo:** N ta component'ning har biri boshqa N-1 component bilan to'g'ridan-to'g'ri aloqa qiladi — bog'liqliklar eksponensial o'sadi.

**Yechim:** Mediator barcha aloqani boshqaradi — component'lar faqat mediator'ni biladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Mediator pattern'da har bir component faqat mediator'ga reference saqlaydi (`user.chatRoom = this`). Mediator esa barcha component'larga reference saqlaydi (`#users = new Map()`). Bu **star topology** — N ta component uchun faqat N ta reference (mediator → component) + N ta reverse reference (component → mediator), jami 2N. Mediator'siz to'g'ridan-to'g'ri aloqada N*(N-1) reference kerak bo'lardi.

`Map` ichida `name → user` sifatida saqlash O(1) lookup beradi. `broadcast()` da `this.#users.forEach()` Map iterator protocol ishlatadi — Map insertion order bo'yicha iterate qiladi. `forEach` callback har bir entry uchun chaqiriladi — V8 bu callback'ni inline qilishi mumkin agar body kichik bo'lsa.

Circular reference mavjud: mediator → user → mediator. V8'ning GC'si mark-and-sweep algorithm ishlatadi — bu circular reference'larni to'g'ri tozalaydi (reference counting'dan farqli). Agar mediator va barcha user'lar unreachable bo'lsa, butun graph GC tomonidan tozalanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
class ChatRoom {
  #users = new Map();

  register(user) {
    this.#users.set(user.name, user);
    user.chatRoom = this;
    this.broadcast(`${user.name} chatga qo'shildi`, user);
  }

  send(message, from, to) {
    const recipient = this.#users.get(to);
    if (!recipient) {
      console.log(`"${to}" topilmadi`);
      return;
    }
    recipient.receive(message, from.name);
  }

  broadcast(message, from) {
    this.#users.forEach((user, name) => {
      if (name !== from.name) {
        user.receive(message, from.name);
      }
    });
  }
}

class User {
  constructor(name) {
    this.name = name;
    this.chatRoom = null;
  }

  send(message, to) {
    console.log(`${this.name} → ${to}: ${message}`);
    this.chatRoom.send(message, this, to);
  }

  broadcast(message) {
    console.log(`${this.name} → all: ${message}`);
    this.chatRoom.broadcast(message, this);
  }

  receive(message, from) {
    console.log(`  [${this.name}] ${from}: ${message}`);
  }
}

const chatRoom = new ChatRoom();
const ali = new User("Ali");
const vali = new User("Vali");
const soli = new User("Soli");

chatRoom.register(ali);
chatRoom.register(vali);
chatRoom.register(soli);

ali.send("Salom!", "Vali");
// Ali → Vali: Salom!
//   [Vali] Ali: Salom!

vali.broadcast("Hammaga salom!");
// Vali → all: Hammaga salom!
//   [Ali] Vali: Hammaga salom!
//   [Soli] Vali: Hammaga salom!
```

**Real-world:** Chat application'lar, Air Traffic Control, Express middleware (app = mediator), React Context, Redux store.

</details>

---

## State Pattern

### Nazariya

State — object'ning **xatti-harakati holat (state) ga qarab o'zgaradi**. Har bir holat alohida object sifatida ajratiladi. If/else yoki switch bilan holat boshqarish o'rniga — holat o'zgarishi bilan object'ning behavior'i avtomatik almashadi.

**Muammo:** Murakkab if/else yoki switch holat mashinalari — yangi holat qo'shishda barcha conditional'larni o'zgartirish kerak.

**Yechim:** Har bir holat alohida class — context object joriy holat orqali method'larni chaqiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

State pattern'da `this.#state` property turli class instance'larini saqlaydi — `PendingState`, `PaidState`, `ShippedState`. Har safar `setState(new PaidState(this))` chaqirilganda yangi state object allocate qilinadi va eski state object reference yo'qoladi (GC tozalaydi). V8 uchun `this.#state` ning type'i har safar o'zgaradi — bu **polymorphic** property access, monomorphic IC'dan sekinroq.

`this.#state.pay()` chaqirilganda engine birinchi `#state` property'ni o'qiydi, keyin state object'ning prototype chain'idan `pay()` method'ni qidiradi. Har bir state class o'z prototype'iga ega — `PendingState.prototype.pay`, `PaidState.prototype.ship`. Agar `#state` doim turli type bo'lsa, V8 bu call site'ni megamorphic deb belgilaydi va slow path ishlatadi.

State object constructor'da `this.order = order` — bu context'ga back-reference. Circular reference hosil bo'ladi: order → state → order. Mark-and-sweep GC buni muammosiz hal qiladi. Har bir state transition'da yangi state object yaratish o'rniga, state object'larni cache qilish (flyweight pattern) memory allocation'ni kamaytiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Holat interface
class OrderState {
  constructor(order) { this.order = order; }
  pay()     { throw new Error("Bu holatda to'lov mumkin emas"); }
  ship()    { throw new Error("Bu holatda yetkazish mumkin emas"); }
  deliver() { throw new Error("Bu holatda yetkazib berish mumkin emas"); }
  cancel()  { throw new Error("Bu holatda bekor qilish mumkin emas"); }
}

class PendingState extends OrderState {
  pay() {
    console.log("To'lov qabul qilindi");
    this.order.setState(new PaidState(this.order));
  }
  cancel() {
    console.log("Buyurtma bekor qilindi");
    this.order.setState(new CancelledState(this.order));
  }
}

class PaidState extends OrderState {
  ship() {
    console.log("Buyurtma yetkazishga berildi");
    this.order.setState(new ShippedState(this.order));
  }
  cancel() {
    console.log("To'lov qaytarildi, buyurtma bekor qilindi");
    this.order.setState(new CancelledState(this.order));
  }
}

class ShippedState extends OrderState {
  deliver() {
    console.log("Buyurtma yetkazib berildi");
    this.order.setState(new DeliveredState(this.order));
  }
}

class DeliveredState extends OrderState {}
class CancelledState extends OrderState {}

// Context
class Order {
  #state;
  constructor(id) {
    this.id = id;
    this.#state = new PendingState(this);
  }
  setState(state) { this.#state = state; }
  getStateName() { return this.#state.constructor.name; }

  pay()     { this.#state.pay(); }
  ship()    { this.#state.ship(); }
  deliver() { this.#state.deliver(); }
  cancel()  { this.#state.cancel(); }
}

const order = new Order("ORD-001");
console.log(order.getStateName()); // PendingState

order.pay();     // "To'lov qabul qilindi"
console.log(order.getStateName()); // PaidState

order.ship();    // "Buyurtma yetkazishga berildi"
order.deliver(); // "Buyurtma yetkazib berildi"

// order.cancel(); // ❌ Error: Bu holatda bekor qilish mumkin emas
```

**Real-world:** E-commerce order flow, TCP connection states, UI component state'lari, game character state'lari, workflow engine'lar.

</details>

---

## Chain of Responsibility

### Nazariya

Chain of Responsibility — so'rov (request) handler'lar zanjiri bo'ylab uzatiladi. Har bir handler so'rovni qayta ishlashi yoki keyingisiga uzatishi mumkin. Yuboruvchi qaysi handler qayta ishlashini bilmaydi. Bu **middleware pipeline** uchun asos — Express.js, Koa, NestJS middleware'lari shu pattern'ga asoslangan.

**Muammo:** So'rovni qayta ishlash uchun bir nechta qadam kerak — auth, validation, logging, error handling. Hammasini bitta funksiyaga yozish murakkab.

**Yechim:** Har bir qadam alohida handler — ular zanjir bo'lib, so'rovni ketma-ket qayta ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Middleware pipeline'ning `next()` funksiyasi closure orqali `index` counter va `middlewares` array'ga reference saqlaydi. Har bir `next()` chaqiruvida `index++` orqali keyingi middleware'ga o'tiladi. Bu `[[Environment]]` record'da `index` mutable binding sifatida saqlanadi — har bir chaqiruvda yangi closure yaratilmaydi, bitta closure ichidagi counter o'zgaradi.

`async/await` bilan middleware pattern "onion model" hosil qiladi — `await next()` keyingi middleware'ga o'tadi, u tugagandan keyin joriy middleware'ning qolgan qismi bajariladi. Engine har bir `await` da microtask queue'ga Promise resolution'ni qo'yadi. N ta middleware uchun N ta Promise object va N ta microtask yaratiladi — bu memory overhead, lekin amalda middleware soni kam (10-20) bo'lgani uchun sezilarli emas.

`next()` chaqirilmasa zanjir to'xtaydi — keyingi middleware'lar execute bo'lmaydi. Bu function composition pattern — har bir middleware keyingisini chaqirish yoki chaqirmaslik huquqiga ega. Express.js'da `next` funksiyaga `error` argument berish error-handling middleware'ga o'tish imkonini beradi — bu Chain of Responsibility'ning error branch'i.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Middleware pipeline — Express.js pattern:

```javascript
class MiddlewarePipeline {
  #middlewares = [];

  use(middleware) {
    this.#middlewares.push(middleware);
    return this; // chaining
  }

  async execute(context) {
    let index = 0;

    const next = async () => {
      if (index >= this.#middlewares.length) return;
      const middleware = this.#middlewares[index++];
      await middleware(context, next);
    };

    await next();
    return context;
  }
}

// Middleware'lar — har biri alohida concern
function logger(ctx, next) {
  console.log(`[${new Date().toISOString()}] ${ctx.method} ${ctx.url}`);
  return next();
}

function auth(ctx, next) {
  if (!ctx.headers?.authorization) {
    ctx.status = 401;
    ctx.body = { error: "Token kerak" };
    return; // zanjirni to'xtatadi — next() chaqirilmaydi
  }
  ctx.user = { id: 1, name: "Ali" };
  return next();
}

function validate(ctx, next) {
  if (ctx.method === "POST" && !ctx.body?.data) {
    ctx.status = 400;
    ctx.body = { error: "data maydoni kerak" };
    return;
  }
  return next();
}

function handler(ctx, next) {
  ctx.status = 200;
  ctx.body = { message: "Muvaffaqiyatli", user: ctx.user };
  return next();
}

// Pipeline qurilishi
const pipeline = new MiddlewarePipeline();
pipeline
  .use(logger)
  .use(auth)
  .use(validate)
  .use(handler);

// Authorized request
await pipeline.execute({
  method: "GET",
  url: "/api/users",
  headers: { authorization: "Bearer token123" }
});
// ctx.status = 200, ctx.body = { message: "Muvaffaqiyatli", user: {...} }
```

**Real-world:** Express.js/Koa middleware (`app.use()`), DOM event bubbling, Promise `.then()` chain, Node.js streams pipe.

</details>

---

## Iterator Pattern

### Nazariya

Iterator — collection elementlariga **ketma-ket kirish** uchun standart interface beradi. JavaScript da Iterator Protocol built-in — `Symbol.iterator` metodi bilan har qanday object'ni iterable qilish mumkin. `for...of`, spread operator, destructuring — barchasi iterator protocol ishlatadi.

**Muammo:** Turli data structure'lar (array, linked list, tree) — har birining traverse usuli boshqacha. Client kod data structure'ning ichki tuzilishini bilishi kerak.

**Yechim:** Iterator — yagona interface (`next()` metodi, `{ value, done }` qaytaradi) orqali har qanday collection'ni traverse qilish.

<details>
<summary><strong>Under the Hood</strong></summary>

`Symbol.iterator` — well-known Symbol. Engine `for...of` chaqirilganda object'dan `[Symbol.iterator]()` method'ni qidiradi. Bu method iterator object qaytaradi — `{ next() }` interface bilan. Har bir `next()` chaqiruvida `{ value, done }` object yaratiladi. V8 bu kichik object'larni **allocation elimination** orqali optimize qiladi — agar `value` va `done` faqat loop ichida ishlatilsa, object heap'da yaratilmaydi, register'larda saqlanadi.

Generator funksiya (`*[Symbol.iterator]()`) ichki **state machine** sifatida compile qilinadi. Har bir `yield` nuqtasi alohida state. V8 generator'ni suspend/resume qilganda, generator'ning execution context'i (local variable'lar, stack frame) **generator object** ichida saqlanadi — bu oddiy funksiyadan farqli ravishda context GC tomonidan tozalanmaydi, generator complete bo'lguncha.

`yield*` delegation'da ichki iterable'ning iterator'i yaratiladi va har bir `next()` chaqiruvida ichki iterator'ga forward qilinadi. Tree traversal'da rekursiv `yield*` har bir node uchun yangi generator object yaratadi — chuqur tree'larda bu sezilarli memory overhead. `for...of` loop tugaganda yoki `break` qilinganda engine `return()` method'ni chaqiradi (agar mavjud bo'lsa) — bu generator'ni tozalash imkonini beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Custom iterable — `Symbol.iterator` bilan:

```javascript
class NumberRange {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    const step = this.step;

    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

const range = new NumberRange(1, 10, 2);

for (const n of range) {
  console.log(n); // 1, 3, 5, 7, 9
}

console.log([...range]); // [1, 3, 5, 7, 9]

const [first, second, third] = range;
console.log(first, second, third); // 1, 3, 5
```

Tree traversal — generator bilan:

```javascript
class TreeNode {
  constructor(value, children = []) {
    this.value = value;
    this.children = children;
  }

  // Depth-first iterator
  *[Symbol.iterator]() {
    yield this.value;
    for (const child of this.children) {
      yield* child; // recursive delegation
    }
  }
}

const tree = new TreeNode("root", [
  new TreeNode("A", [
    new TreeNode("A1"),
    new TreeNode("A2")
  ]),
  new TreeNode("B", [
    new TreeNode("B1")
  ])
]);

console.log([...tree]); // ["root", "A", "A1", "A2", "B", "B1"]
```

**Real-world:** JavaScript built-in iterable'lar (Array, String, Map, Set), generator'lar (14-bo'lim), database cursor'lar, pagination, lazy evaluation.

</details>

---

## Edge Cases va Gotchas

Design pattern'lar bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha. Har biri pattern-specific va Common Mistakes bilan overlap qilmaydi.

### Gotcha 1: Singleton cross-module-system — bitta modul **ikki marta** load qilinishi mumkin

Module bundler'lar (Webpack, Vite, Rollup), dual CommonJS/ESM package'lar, yoki har xil path'lar orqali load qilish — bitta "singleton" modul aslida **ikki instance** yaratishi mumkin. Node.js'da `require("./config.js")` va `require("./Config.js")` (katta-kichik harf farqi) alohida cache entry bo'lishi mumkin. Webpack bir xil modul'ni turli chunk'larga join qilsa — ikki copy. Bu "dual package hazard" deb ataladi.

```javascript
// config.js — "singleton"
class AppConfig {
  constructor() {
    this.instanceId = Math.random();
    console.log("AppConfig yaratildi:", this.instanceId);
  }
}
export const config = new AppConfig();
```

```javascript
// app.js
import { config } from "./config.js";
console.log(config.instanceId); // 0.123456

// legacy.js (boshqa path)
import { config } from "./Config.js"; // ← katta C (Windows filesystem case-insensitive)
console.log(config.instanceId); // 0.789012 — ❌ BOSHQA instance!

// CommonJS vs ESM dual package hazard:
// main: "./dist/index.mjs" (ESM)
// require: "./dist/index.cjs" (CommonJS)
// Node.js ularni alohida cache qiladi — ikki alohida instance!
```

**Debug belgilari:**
- "Singleton" state noma'lum sabablarga ko'ra reset bo'ladi
- Library ichida bir versiya, app ichida boshqa versiya
- `instanceof` check'lar fail bo'ladi (turli instance)
- `console.log(a === b)` — false, lekin ikkalasi ham "singleton" deb hisoblangan

**Yechim:**
```javascript
// ✅ globalThis orqali cross-context singleton
function getAppConfig() {
  const key = Symbol.for("app.config"); // Symbol.for — global registry
  if (!globalThis[key]) {
    globalThis[key] = new AppConfig();
  }
  return globalThis[key];
}

// Har qanday module system dan chaqirilsa — aynan bir instance
// Symbol.for("app.config") — global registry'da saqlanadi
// globalThis — Node.js/browser/Worker har joyda mavjud
```

**Nima uchun:** JavaScript module system'lari (CommonJS, ESM, AMD, Webpack, Vite) har biri o'z module cache'iga ega. Modul resolution path'ga bog'liq — har yangi path yangi cache entry. `Symbol.for("key")` esa **global Symbol Registry** ishlatadi — bu cross-realm va cross-module-system shared state uchun yagona standart mexanizm.

**Yechim:** Critical singleton'lar uchun `Symbol.for()` + `globalThis` pattern. Library publisher'lar uchun "peerDependency" qo'llash — bitta versiya kafolatlanishi. Monorepo'da dedupe sozlamalari (Yarn `nohoist`, pnpm `public-hoist-pattern`).

### Gotcha 2: Decorator composition order — outer → inner execution

Decorator'lar composition'da order muhim, lekin intuitive emas. `withLogging(withTiming(fn))` — birinchi o'qilganda "timing ichida logging" deb tuyuladi, lekin aslida **outer (logging)** birinchi ishlaydi, inner (timing) ichkarida. Execution onion model — decorator'lar layered.

```javascript
function withLogging(fn, label) {
  return function(...args) {
    console.log(`[${label}] IN`);
    const result = fn.apply(this, args);
    console.log(`[${label}] OUT`);
    return result;
  };
}

function withTiming(fn, label) {
  return function(...args) {
    console.time(label);
    const result = fn.apply(this, args);
    console.timeEnd(label);
    return result;
  };
}

function calculate() {
  return 42;
}

// Variant 1: withLogging(withTiming(calculate))
const v1 = withLogging(withTiming(calculate, "calc"), "log");
v1();
// [log] IN           ← outer logging birinchi
// calc: 0.05ms       ← inner timing
// [log] OUT          ← outer logging oxirgi

// Variant 2: withTiming(withLogging(calculate)) — ORDER TESKARI!
const v2 = withTiming(withLogging(calculate, "log"), "calc");
v2();
// calc: (start)      ← outer timing birinchi
// [log] IN           ← inner logging
// [log] OUT
// calc: 0.08ms       ← outer timing (log overhead ham hisoblangan)

// ⚠️ Variant 2 da timing LOG overhead'ni ham o'lchaydi — noto'g'ri benchmark!
```

**Real-world trap — HTTP middleware:**
```javascript
// ❌ Xato order — auth logging'dan oldin
app.use(auth);      // 1-chi ishlaydi
app.use(logger);    // 2-chi ishlaydi
// Logger request'ni ko'radi, lekin auth qilingan
// Auth fail bo'lgan request'lar log'lanmaydi!

// ✅ To'g'ri order — logger eng tashqarida
app.use(logger);    // 1-chi — har request log'lanadi
app.use(auth);      // 2-chi — log'dan keyin auth
// Endi unauthorized request'lar ham log'ga tushadi (muhim debugging uchun)
```

**Nima uchun:** Function composition math: `f(g(x))` — `g` avval, keyin `f`. Lekin decorator pattern'da `f = withLogging(g)` — `f` **yangi funksiya**, uning body'sida `g` chaqiriladi. Chaqirish paytida: `f()` → logging IN → `g()` execute → logging OUT. Ya'ni outer wrapper birinchi va oxirgi kodda, inner o'rtada — "onion" model.

**Yechim:** Decorator composition'da har doim **outer → middle → inner → middle → outer** sekvensini tasavvur qiling. Test yozayotganda har layer'ning effektini alohida tekshiring. Timing/benchmark decorator'lari eng **tashqi** yoki eng **ichki** bo'lishi kerakligini ongli tanlang (tashqi = butun chain'ni o'lchaydi, ichki = faqat original fn'ni).

### Gotcha 3: Command pattern unbounded history — memory leak

Undo/redo stack'lari cheksiz o'sishi mumkin. Uzun editing session (text editor, graphic editor, form builder) — har amal stack'ga push qilinadi va hech qachon tozalanmaydi. Command object'larning ichida og'ir data (image buffer, parsed AST, clone'lar) bo'lsa — memory tez tugaydi.

```javascript
// ❌ Cheksiz history — memory leak
class Editor {
  #history = [];

  execute(command) {
    command.execute();
    this.#history.push(command); // ← cheksiz o'sish
  }

  undo() {
    const cmd = this.#history.pop();
    cmd?.undo();
  }
}

// 8 soatlik editing session'da 10000+ command
// Har command ichida image snapshot saqlasa → gigabayt memory!

// ✅ Sliding window — fixed size history
class BoundedEditor {
  #history = [];
  #redoStack = [];
  #maxSize = 100; // faqat oxirgi 100 amal

  execute(command) {
    command.execute();
    this.#history.push(command);

    // ✅ Eski command'larni o'chirish
    if (this.#history.length > this.#maxSize) {
      this.#history.shift(); // FIFO: eng eski o'chadi
    }

    this.#redoStack = []; // yangi amal — redo tozalanadi
  }

  undo() {
    const cmd = this.#history.pop();
    if (!cmd) return;
    cmd.undo();
    this.#redoStack.push(cmd);
  }
}

// ✅ Yaxshiroq: Snapshot pattern + compaction
class SnapshotEditor {
  #history = [];
  #snapshotInterval = 50; // har 50 amalda snapshot
  #snapshots = [];

  execute(command) {
    command.execute();
    this.#history.push(command);

    if (this.#history.length % this.#snapshotInterval === 0) {
      // Har 50 amalda "baseline snapshot" — eski command'lar keraksiz
      this.#snapshots.push(this.getCurrentState());
      this.#history = []; // command history reset
    }
  }

  // Restore faqat latest snapshot + command'lar orqali
}
```

**Nima uchun:** JavaScript GC mark-and-sweep — array ichidagi reference'lar "reachable" deb hisoblanadi, GC tozalay olmaydi. Command pattern'ning "temporal undo" ning asosiy idiomasi — har amalning teskarisini saqlash. Lekin tarix **cheksiz** bo'lsa, RAM xuddi stack trace'ning saqlanishi kabi ortadi. Production editor'lar (VS Code, Figma) snapshot + differential history bilan ishlaydi.

**Yechim:** Har doim `maxHistorySize` limit qo'ying (50-500 oralig'ida odatda yetarli). Og'ir command'lar uchun snapshot pattern — har N amalda baseline saqlab, oraliq command'larni tozalash. Memory profiling bilan command size'ni nazorat qiling.

### Gotcha 4: Factory return type inconsistency — `instanceof` check'lar buziladi

Factory funksiya gohi object literal, gohi class instance qaytarsa — `instanceof` check'lar ishonchsiz bo'ladi. Modern kodbase'da TypeScript type check'lar runtime'da mavjud emas, shuning uchun runtime `instanceof` tekshiruvlari cross-factory ishlatiladi.

```javascript
class AdminUser {
  constructor(data) {
    Object.assign(this, data);
    this.role = "admin";
  }
  hasPermission(p) { return true; }
}

// ❌ Noto'g'ri — factory qachon class, qachon literal qaytaradi
function createUser(type, data) {
  if (type === "admin") {
    return new AdminUser(data); // class instance
  }
  // Boshqalar — POJO
  return { ...data, role: type };
}

const admin = createUser("admin", { name: "Ali" });
const user = createUser("viewer", { name: "Vali" });

console.log(admin instanceof AdminUser);  // true ✅
console.log(user instanceof AdminUser);   // false ❌ — lekin user.role === "viewer"

// ❌ Runtime type check — adminUser uchun sababsiz if
function processUser(u) {
  if (u instanceof AdminUser) {
    return u.hasPermission("delete"); // faqat admin uchun ishlaydi
  }
  // viewer: u.hasPermission yo'q — method undefined!
  return false;
}

// ❌ Yana: JSON.parse keyin instanceof false
const json = JSON.stringify(admin);
const restored = JSON.parse(json);
console.log(restored instanceof AdminUser); // false ❌ — faqat POJO bo'ldi

// ✅ Consistent — hamma doim class instance
class User {
  constructor(data) { Object.assign(this, data); }
  hasPermission(p) { return false; }
}

class AdminUser extends User {
  hasPermission(p) { return true; }
}

class ViewerUser extends User {
  hasPermission(p) { return p === "read"; }
}

function createUser(type, data) {
  switch (type) {
    case "admin":  return new AdminUser(data);
    case "viewer": return new ViewerUser(data);
    default:       return new User(data);
  }
}

// Endi barcha user'lar User instance, method'lar guaranteed
const u = createUser("viewer", { name: "Vali" });
console.log(u instanceof User);           // true ✅
console.log(u.hasPermission("read"));     // true — method mavjud

// ✅ Yoki discriminated union pattern — POJO + type field
function createUserPojo(type, data) {
  return { ...data, __type: type, role: type };
}

function isAdmin(user) {
  return user?.__type === "admin"; // duck typing, instanceof kerak emas
}
```

**Nima uchun:** `instanceof` prototype chain tekshiradi — faqat class instance'larda ishlaydi. Object literal'ning prototype'i `Object.prototype` — custom class'larga mos kelmaydi. `JSON.parse` doim POJO qaytaradi — serialize/deserialize cycle class identity'ni yo'qotadi. Type guard uchun `instanceof` ishlatilsa, factory inconsistency silent bug'larga olib keladi.

**Yechim:** Factory'da **consistent return type** tanlang: (1) **hamma class** — base class + subclass'lar, method'lar guaranteed, `instanceof` ishonchli; yoki (2) **hamma POJO** + discriminated union (`__type` field) — serialization-friendly, duck typing bilan tekshirish. Aralashtirib ishlatmang.

### Gotcha 5: State pattern transition bypass — validation chetlab o'tiladi

State machine'ning asosiy qiymati — **allowed transitions**ni enforcement qilish. Lekin `context.setState(new PaidState(ctx))` ni **tashqaridan** public method sifatida ochsangiz, validation chetlab o'tiladi. Transitions faqat state class'lar ichidan (`pay() → setState(new PaidState)`) bo'lishi kerak.

```javascript
// ❌ Xato: setState public
class Order {
  #state;
  constructor() { this.#state = new PendingState(this); }

  setState(state) { this.#state = state; } // ⚠️ public — har qanday transition mumkin

  pay() { this.#state.pay(); }
  ship() { this.#state.ship(); }
}

class PendingState {
  constructor(order) { this.order = order; }
  pay() {
    // Validation: to'lov kerak
    this.order.setState(new PaidState(this.order));
  }
}

class PaidState {
  constructor(order) { this.order = order; }
  ship() {
    this.order.setState(new ShippedState(this.order));
  }
}

const order = new Order();

// ❌ Legitimate flow
order.pay();
order.ship(); // OK — pay → ship

// ❌ Illegitimate flow — validation chetlab o'tiladi!
const order2 = new Order();
order2.setState(new ShippedState(order2)); // ⚠️ Direct transition!
order2.ship(); // ishlaydi — lekin to'lov yo'q edi!

// ─── Xatoni qanday topish ───
// Test'lar faqat legitimate flow ishlatsa — bug topilmaydi
// Production'da race condition yoki noto'g'ri API call setState ni ochiq state'ga o'tkazadi

// ✅ To'g'ri: setState private yoki internal
class OrderSafe {
  #state;

  constructor() {
    this.#state = new PendingState(this);
  }

  // Public API — faqat action'lar, transitions yo'q
  pay(paymentData) { this.#state.pay(paymentData); }
  ship(address)    { this.#state.ship(address); }
  deliver()        { this.#state.deliver(); }
  cancel()         { this.#state.cancel(); }

  // Internal — state class'lar chaqiradi (private method, tashqaridan kirish yo'q)
  // State class'larga OrderSafe instance beriladi, ular #internalSetState chaqiradi

  // Yoki Symbol orqali "private" API
  [Symbol.for("OrderSafe.internal.setState")](state) {
    // Validation: agar kerak bo'lsa
    this.#state = state;
  }
}

// State class'lar Symbol key orqali transition qiladi:
class PendingStateSafe {
  constructor(order) { this.order = order; }
  pay(data) {
    if (!data?.amount) throw new Error("Amount kerak");
    const transitionKey = Symbol.for("OrderSafe.internal.setState");
    this.order[transitionKey](new PaidStateSafe(this.order));
  }
}

// ✅ Yaxshiroq: state machine library (xstate)
// XState — declarative state machine, visual debugging, transition validation built-in
```

**Nima uchun:** State pattern'ning invariant'i: **ma'lum state'da faqat ruxsat etilgan transitions bajariladi**. Agar state o'zgartirish public API bo'lsa, bu invariant buziladi. `PaidState` ga o'tish faqat `pay()` muvaffaqiyatli tugagandan keyin bo'lishi kerak — direct `setState(PaidState)` bu qoidani yutadi. Xuddi `private` field'lar kabi, state transition internal bo'lishi kerak.

**Yechim:** State transition method'ini **private** (`#setState`) yoki `Symbol.for()` orqali "hidden" qiling. State class'lar shu symbol key orqali transition qiladi, tashqi kod esa faqat action method'lari (`pay`, `ship`) orqali interact qiladi. Production uchun **XState** kabi library'lar — visual state chart, transition guards, history, parallel states bilan to'liq state machine support.

---

## Common Mistakes

### ❌ Xato 1: Keraksiz Singleton — testing qiyinlashadi

```javascript
// ❌ Hamma narsani singleton qilish
class UserRepository {
  static #instance = null;
  static getInstance() {
    if (!this.#instance) this.#instance = new UserRepository();
    return this.#instance;
  }
  getUsers() { /* DB query */ }
}

// Test'da muammo — mock qilish qiyin, global state
```

### ✅ To'g'ri usul:

```javascript
// ✅ Dependency injection — test'da mock berish oson
class UserRepository {
  constructor(db) { this.db = db; }
  getUsers() { return this.db.query("SELECT * FROM users"); }
}

// Production: new UserRepository(realDb)
// Test: new UserRepository(mockDb) ✅
```

**Nima uchun:** Singleton global state yaratadi — test'lar bir-biriga ta'sir qiladi. Dependency injection yechim.

---

### ❌ Xato 2: Observer'da memory leak — listener o'chirilmaydi

```javascript
// ❌ Listener qo'shib, hech qachon olib tashlamaslik
class Component {
  mount() {
    eventBus.on("data:update", this.handleUpdate.bind(this));
    // component unmount bo'lganda listener qoladi — memory leak!
  }
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Unsubscribe saqlash va unmount'da chaqirish
class Component {
  #unsubscribers = [];

  mount() {
    const unsub = eventBus.on("data:update", this.handleUpdate.bind(this));
    this.#unsubscribers.push(unsub);
  }

  unmount() {
    this.#unsubscribers.forEach(unsub => unsub());
    this.#unsubscribers = [];
  }
}
```

**Nima uchun:** Har bir `on()` uchun `off()` yoki unsubscribe bo'lishi shart. Aks holda listener GC tomonidan tozalanmaydi.

---

### ❌ Xato 3: Strategy pattern o'rniga if/else zanjiri

```javascript
// ❌ Yangi format qo'shish — barcha if/else'larni o'zgartirish kerak
function formatData(data, format) {
  if (format === "json") return JSON.stringify(data);
  else if (format === "csv") return convertToCSV(data);
  else if (format === "xml") return convertToXML(data);
  else if (format === "yaml") return convertToYAML(data);
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Strategy — yangi format = faqat object'ga qo'shish
const formatters = {
  json: (data) => JSON.stringify(data),
  csv: (data) => convertToCSV(data),
  xml: (data) => convertToXML(data),
  yaml: (data) => convertToYAML(data)
};

function formatData(data, format) {
  const formatter = formatters[format];
  if (!formatter) throw new Error(`"${format}" qo'llab-quvvatlanmaydi`);
  return formatter(data);
}

// Yangi format qo'shish — faqat 1 qator
formatters.protobuf = (data) => convertToProtobuf(data);
```

**Nima uchun:** Open/Closed principle — kengaytirish uchun ochiq, o'zgartirish uchun yopiq.

---

### ❌ Xato 4: Ulkan Mediator — hamma narsani bitta joyga yig'ish

```javascript
// ❌ Mediator juda ko'p mas'uliyat oladi
class AppMediator {
  handleUserLogin() { /* ... */ }
  handlePayment() { /* ... */ }
  handleNotification() { /* ... */ }
  handleLogging() { /* ... */ }
  // 50+ method...
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Domain-specific mediator'lar
class AuthMediator { /* faqat auth */ }
class PaymentMediator { /* faqat to'lov */ }
class NotificationMediator { /* faqat notification */ }
```

**Nima uchun:** Bitta mediator juda ko'p mas'uliyat olsa — Single Responsibility principle buziladi.

---

## Amaliy Mashqlar

### Mashq 1: EventEmitter implement qiling (O'rta)

**Savol:** `on(event, listener)`, `off(event, listener)`, `once(event, listener)`, `emit(event, ...args)` metodlari bilan `EventEmitter` class yarating.

<details>
<summary>Javob</summary>

```javascript
class EventEmitter {
  constructor() {
    this.events = new Map();
  }

  on(event, listener) {
    if (!this.events.has(event)) this.events.set(event, []);
    this.events.get(event).push(listener);
    return this;
  }

  off(event, listener) {
    const listeners = this.events.get(event);
    if (!listeners) return this;
    this.events.set(event, listeners.filter(l =>
      l !== listener && l._original !== listener
    ));
    return this;
  }

  once(event, listener) {
    const wrapper = (...args) => {
      listener(...args);
      this.off(event, wrapper);
    };
    wrapper._original = listener;
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.events.get(event);
    if (!listeners || listeners.length === 0) return false;
    [...listeners].forEach(l => l(...args));
    return true;
  }
}

// Test
const emitter = new EventEmitter();
const handler = (msg) => console.log(`Got: ${msg}`);

emitter.on("msg", handler);
emitter.once("msg", (msg) => console.log(`Once: ${msg}`));

emitter.emit("msg", "salom");
// Got: salom
// Once: salom

emitter.emit("msg", "dunyo");
// Got: dunyo

emitter.off("msg", handler);
emitter.emit("msg", "test"); // false
```

</details>

---

### Mashq 2: Middleware Pipeline implement qiling (Qiyin)

**Savol:** Express.js ga o'xshash middleware pipeline yarating. `use(fn)` bilan middleware qo'shish, `execute(ctx)` bilan bajarish. Har bir middleware `(ctx, next)` signature. `next()` chaqirilmasa zanjir to'xtaydi.

<details>
<summary>Javob</summary>

```javascript
class Pipeline {
  constructor() {
    this.middlewares = [];
  }

  use(fn) {
    this.middlewares.push(fn);
    return this;
  }

  async execute(context) {
    let index = 0;
    const middlewares = this.middlewares;

    async function next() {
      if (index >= middlewares.length) return;
      const fn = middlewares[index++];
      await fn(context, next);
    }

    await next();
    return context;
  }
}

// Test
const pipeline = new Pipeline();

pipeline.use(async (ctx, next) => {
  console.log("1: Kirish");
  await next();
  console.log("1: Chiqish");
});

pipeline.use(async (ctx, next) => {
  if (!ctx.authorized) {
    ctx.error = "Unauthorized";
    return; // zanjir to'xtaydi
  }
  await next();
});

pipeline.use(async (ctx, next) => {
  ctx.result = "OK";
  console.log("3: Handler");
  await next();
});

await pipeline.execute({ authorized: true });
// 1: Kirish → 3: Handler → 1: Chiqish
```

**Tushuntirish:** `next()` closure orqali `index`ni ushlab turadi. Har bir `next()` chaqiruvida keyingi middleware'ga o'tadi. Async/await bilan onion model ishlaydi.

</details>

---

### Mashq 3: State Machine implement qiling (Qiyin)

**Savol:** `createStateMachine(config)` funksiyasi yarating. Config: `{ initial, states: { [state]: { on: { [event]: nextState } } } }`. `send(event)` bilan holat o'zgartirilsin.

<details>
<summary>Javob</summary>

```javascript
function createStateMachine(config) {
  let currentState = config.initial;
  const listeners = [];

  return {
    get state() { return currentState; },

    send(event) {
      const stateConfig = config.states[currentState];
      if (!stateConfig?.on?.[event]) {
        throw new Error(`"${currentState}" holatida "${event}" eventi mavjud emas`);
      }
      const prev = currentState;
      currentState = stateConfig.on[event];
      listeners.forEach(fn => fn(currentState, prev, event));
      return currentState;
    },

    onChange(listener) {
      listeners.push(listener);
      return () => {
        const i = listeners.indexOf(listener);
        if (i > -1) listeners.splice(i, 1);
      };
    },

    can(event) {
      return Boolean(config.states[currentState]?.on?.[event]);
    }
  };
}

// Test
const light = createStateMachine({
  initial: "green",
  states: {
    green:  { on: { TIMER: "yellow" } },
    yellow: { on: { TIMER: "red" } },
    red:    { on: { TIMER: "green" } }
  }
});

light.onChange((to, from) => console.log(`${from} → ${to}`));
light.send("TIMER"); // green → yellow
light.send("TIMER"); // yellow → red
light.send("TIMER"); // red → green
```

</details>

---

## Xulosa

| Pattern | Kategoriya | Muammo | Yechim |
|---------|-----------|--------|--------|
| **Factory** | Creational | Object yaratish logikasi tarqalib ketadi | Yaratish bir joyga markazlashtiriladi |
| **Singleton** | Creational | Bir nechta instance — resurs isrofi | Faqat bitta instance kafolati |
| **Builder** | Creational | Constructor'ga 10+ parameter | Qadam-baqadam, method chaining |
| **Module** | Structural | Global scope pollution | IIFE/ES Module bilan encapsulation |
| **Decorator** | Structural | Har safar original kodni o'zgartirish | Wrapper bilan yangi funksionallik |
| **Adapter** | Structural | Mos kelmaydigan interface'lar | Oraliq tarjima qatlam |
| **Facade** | Structural | Murakkab subsystem | Oddiy yagona interface |
| **Proxy** | Structural | Object kirishni nazorat qilish | Oraliq qatlam — lazy init, cache, auth |
| **Observer** | Behavioral | Holat o'zgarishini xabarlash | Subject → Observer'lar (one-to-many) |
| **Pub/Sub** | Behavioral | Observer — tight coupling | Event bus (loose coupling) |
| **Strategy** | Behavioral | if/else bilan algoritm tanlash | Almashtiriladigan strategy object'lar |
| **Command** | Behavioral | Amallarni kechiktirish, qaytarish | Action = object (execute/undo) |
| **Mediator** | Behavioral | N*N component aloqasi | Markaziy hub — N aloqa |
| **State** | Behavioral | if/else holat mashinasi | Har bir holat — alohida class |
| **Chain of Resp.** | Behavioral | Murakkab so'rov qayta ishlash | Handler'lar zanjiri (middleware) |
| **Iterator** | Behavioral | Turli collection traverse usullari | Yagona next() interface |

> **Keyingi bo'lim:** [25-string-methods.md](25-string-methods.md) — String methods to'liq: charAt, indexOf, includes, slice, replace, split, padStart, normalize, Unicode handling, template literals, performance.

> **Cross-references:** [05-closures.md](05-closures.md) (Module pattern — closure bilan encapsulation), [09-functions.md](09-functions.md) (HOF, currying — Decorator va Strategy pattern'larda), [14-iterators-generators.md](14-iterators-generators.md) (Iterator protocol — Iterator pattern), [19-events.md](19-events.md) (addEventListener — Observer pattern), [23-proxy-reflect.md](23-proxy-reflect.md) (JS Proxy — Proxy pattern)
