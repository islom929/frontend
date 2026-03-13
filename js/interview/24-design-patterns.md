# Design Patterns — Interview Savollari

> **Bo'lim 24** | Factory, Singleton, Builder, Module, Decorator, Adapter, Facade, Proxy, Observer, Pub/Sub, Strategy, Command, Mediator, State, Chain of Responsibility, Iterator

---

## Savol 1: Design pattern nima va nima uchun kerak? [Junior+]

**Javob:**

Design pattern — dasturiy ta'minotda takroriy uchraydigan muammolarning isbotlangan, qayta ishlatilishi mumkin bo'lgan arxitekturaviy yechimlari. Pattern'lar uchta kategoriyaga bo'linadi:

| Kategoriya | Nima haqida | Misollar |
|-----------|------------|---------|
| **Creational** | Object yaratishni boshqarish | Factory, Singleton, Builder |
| **Structural** | Object/class tuzilishi, birlashtirish | Module, Decorator, Adapter, Facade, Proxy |
| **Behavioral** | Object'lar arasi aloqa, javobgarlik | Observer, Strategy, Command, State, Iterator |

Pattern'lar kodni tuzilmali, qayta ishlatiluvchi va boshqa developer'larga tushunarli qiladi. "Biz Pub/Sub pattern ishlatamiz" desangiz — jamoadagi har kim nima nazarda tutilganini tushunadi.

---

## Savol 2: Factory pattern nima? Qachon ishlatiladi? [Junior+]

**Javob:**

Factory — object yaratish logikasini markazlashtiradi. Client kod qaysi class ishlatilayotganini bilmaydi — faqat factory'ga "nima kerak" deydi.

```javascript
function createNotification(type, message) {
  switch (type) {
    case "email": return { type, message, send() { console.log(`Email: ${message}`); } };
    case "sms":   return { type, message, send() { console.log(`SMS: ${message}`); } };
    case "push":  return { type, message, send() { console.log(`Push: ${message}`); } };
    default: throw new Error(`Noma'lum tur: ${type}`);
  }
}

const n = createNotification("email", "Salom!");
n.send(); // Email: Salom!
```

**Qachon ishlatiladi:**
- Object yaratish murakkab yoki conditional
- Yangi turlar tez-tez qo'shiladigan bo'lsa
- Client kodni concrete class'lardan ajratish kerak bo'lganda

**Real-world:** `document.createElement()`, React `createElement()`, ORM model factory'lari.

---

## Savol 3: Singleton pattern nima? JavaScript da qanday implement qilasiz? [Middle]

**Javob:**

Singleton — faqat **bitta instance** yaratilishini kafolatlaydi va unga global access beradi.

JavaScript da **eng oddiy** usul — ES Module. Module bir marta bajariladi va cache'lanadi:

```javascript
// config.js
class AppConfig {
  constructor() {
    this.settings = { apiUrl: "https://api.example.com" };
  }
  get(key) { return this.settings[key]; }
  set(key, value) { this.settings[key] = value; }
}
export const config = new AppConfig(); // singleton — module cache qiladi
```

Class-based usul:

```javascript
class Database {
  static #instance = null;

  static getInstance(connStr) {
    if (!Database.#instance) {
      Database.#instance = new Database(connStr);
    }
    return Database.#instance;
  }

  constructor(connStr) {
    if (Database.#instance) return Database.#instance;
    this.connStr = connStr;
    Database.#instance = this;
  }
}

const db1 = Database.getInstance("postgres://...");
const db2 = Database.getInstance("mysql://...");
console.log(db1 === db2); // true
```

**Ogohlantirish:** Singleton global state yaratadi — unit test'larda muammo. Dependency injection afzalroq.

---

## Savol 4: Observer va Pub/Sub pattern farqi nima? [Middle+]

**Javob:**

| | Observer | Pub/Sub |
|--|---------|---------|
| **Coupling** | Tight — observer subject'ni biladi | Loose — publisher/subscriber bir-birini bilmaydi |
| **Mediator** | Yo'q — to'g'ridan-to'g'ri aloqa | Bor — event bus / message broker |
| **Aloqa** | Subject.subscribe(observer) | bus.subscribe(channel, handler) |
| **Scalability** | Kichik tizimlar | Katta, distributed tizimlar |

```javascript
// Observer — to'g'ridan-to'g'ri bog'langan
subject.subscribe(observer); // observer subject ni biladi

// Pub/Sub — event bus orqali
bus.subscribe("user:login", handler); // handler publisher ni bilmaydi
bus.publish("user:login", data);      // publisher subscriber ni bilmaydi
```

**Observer** — DOM `addEventListener`, Node.js `EventEmitter`.
**Pub/Sub** — Redis Pub/Sub, RabbitMQ, Kafka, microservice'lar arasi aloqa.

---

## Savol 5: EventEmitter implement qiling [Middle+]

**Javob:**

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
      this.off(event, wrapper); // o'zini o'chiradi
    };
    wrapper._original = listener;
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const listeners = this.events.get(event);
    if (!listeners?.length) return false;
    // copy qilish — once emit paytida listeners o'zgarmasligi uchun
    [...listeners].forEach(l => l(...args));
    return true;
  }
}
```

Asosiy nuqtalar:
- `once` — wrapper funksiya, birinchi chaqiruvda o'zini `off` qiladi
- `_original` — `off` bilan once listener'ni original reference orqali ham o'chirish uchun
- `[...listeners]` copy — `once` emit paytida array'ni mutate qilganda index buzilmasligi uchun
- `Map` — object key'lardan farqli, har qanday string/symbol event nomi bilan ishlaydi

---

## Savol 6: Strategy pattern qachon ishlatiladi? Misol bering [Middle]

**Javob:**

Strategy — bir xil vazifani bajarishning bir nechta usuli bo'lganda, ularni runtime da almashtirish kerak bo'lganda ishlatiladi. if/else zanjirini object map bilan almashtiradi.

```javascript
// ❌ if/else — yangi format = barcha if'larni o'zgartirish
function format(data, type) {
  if (type === "json") return JSON.stringify(data);
  else if (type === "csv") return toCSV(data);
  // har safar yangi if...
}

// ✅ Strategy — yangi format = 1 qator qo'shish
const formatters = {
  json: (data) => JSON.stringify(data),
  csv: (data) => toCSV(data),
  xml: (data) => toXML(data)
};

function format(data, type) {
  const fn = formatters[type];
  if (!fn) throw new Error(`${type} qo'llab-quvvatlanmaydi`);
  return fn(data);
}

// Yangi strategy qo'shish
formatters.yaml = (data) => toYAML(data); // ✅ mavjud kodni o'zgartirmaydi
```

**Real-world:** To'lov usullari (karta, PayPal, crypto), sorting algoritm'lari, authentication (JWT, OAuth), compression (gzip, brotli).

---

## Savol 7: Decorator pattern nima? JavaScript da qanday ishlaydi? [Middle]

**Javob:**

Decorator — funksiya yoki object'ning **asl kodini o'zgartirmay** yangi funksionallik qo'shish. Wrapper funksiya original'ni ichiga oladi.

```javascript
// Logging decorator
function withLogging(fn, label) {
  return function(...args) {
    console.log(`[${label}] args:`, args);
    const result = fn.apply(this, args); // this va args to'g'ri uzatish
    console.log(`[${label}] result:`, result);
    return result;
  };
}

// Timing decorator
function withTiming(fn) {
  return function(...args) {
    const start = performance.now();
    const result = fn.apply(this, args);
    console.log(`${(performance.now() - start).toFixed(2)}ms`);
    return result;
  };
}

// Composition — decorator'larni ustma-ust qo'yish
function sum(a, b) { return a + b; }
const trackedSum = withLogging(withTiming(sum), "sum");
trackedSum(3, 5);
// [sum] args: [3, 5]
// 0.01ms
// [sum] result: 8
```

**Muhim:** `fn.apply(this, args)` — `this` kontekstini saqlash. Oddiy `fn(...args)` method decorator'da `this` yo'qoladi.

**Real-world:** Express middleware, NestJS/Angular decorator'lar, MobX `@observable`, TC39 Decorator proposal.

---

## Savol 8: Module pattern qanday ishlaydi? IIFE vs ES Module [Middle]

**Javob:**

Module pattern — private/public farqlash orqali encapsulation. Ikki usul:

```javascript
// 1. IIFE-based (eski) — closure orqali private
const Counter = (function() {
  let count = 0; // private
  return {
    increment() { return ++count; }, // public
    getCount() { return count; }
  };
})();

Counter.increment(); // 1
// Counter.count — undefined (private)

// 2. ES Module (zamonaviy) — module scope = private
// counter.js
let count = 0; // module scope — private
export function increment() { return ++count; }
export function getCount() { return count; }
```

| | IIFE Module | ES Module |
|--|-------------|-----------|
| Private | Closure orqali | Module scope |
| Export | return object | `export` keyword |
| Singleton | IIFE bir marta bajariladi | Module cache'lanadi |
| Dynamic | Yo'q | `import()` dynamic |
| Tree shaking | Yo'q | Ha |

ES Modules — standart yechim. IIFE module pattern hozir kamdan-kam ishlatiladi, lekin tushunchani bilish kerak.

---

## Savol 9: Middleware pipeline implement qiling [Senior]

**Javob:**

Express.js pattern — har bir middleware `(ctx, next)`. `next()` chaqirilmasa zanjir to'xtaydi:

```javascript
class Pipeline {
  constructor() {
    this.stack = [];
  }

  use(fn) {
    this.stack.push(fn);
    return this;
  }

  async execute(ctx) {
    let i = 0;
    const stack = this.stack;

    async function next() {
      if (i >= stack.length) return;
      const fn = stack[i++];
      await fn(ctx, next);
    }

    await next();
    return ctx;
  }
}

// Test
const app = new Pipeline();

app.use(async (ctx, next) => {
  console.log("→ Logger");
  await next();
  console.log("← Logger");
});

app.use(async (ctx, next) => {
  if (!ctx.token) {
    ctx.error = "401 Unauthorized";
    return; // next() chaqirilmaydi — zanjir to'xtaydi
  }
  ctx.user = { name: "Ali" };
  await next();
});

app.use(async (ctx, next) => {
  ctx.body = { data: "OK", user: ctx.user };
  await next();
});

// Auth bor
await app.execute({ token: "abc" });
// → Logger → auth pass → handler → ← Logger

// Auth yo'q
const result = await app.execute({});
// → Logger → auth fail (ctx.error = "401") → ← Logger
```

**Asosiy nuqtalar:**
- `next()` — closure orqali `i` index'ni ushlab turadi
- Har bir middleware `next()` chaqirmasa — qolgan middleware'lar bajarilmaydi
- **Onion model** — birinchi middleware oxirida ham kod bajariladi (`await next()` dan keyin)
- `async/await` — har bir middleware async bo'lishi mumkin

---

## Savol 10: Command pattern nima? Undo/redo qanday ishlaydi? [Middle+]

**Javob:**

Command — har bir amalni object sifatida encapsulate qiladi (`execute()` va `undo()` metodlari bilan). Bu undo/redo, command history va queue qilishni osonlashtiradi.

```javascript
// Har bir command — execute() va undo()
class SetValueCommand {
  constructor(obj, key, newValue) {
    this.obj = obj;
    this.key = key;
    this.newValue = newValue;
    this.oldValue = obj[key]; // undo uchun
  }
  execute() { this.obj[this.key] = this.newValue; }
  undo() { this.obj[this.key] = this.oldValue; }
}

// History manager
class CommandHistory {
  #undoStack = [];
  #redoStack = [];

  execute(cmd) {
    cmd.execute();
    this.#undoStack.push(cmd);
    this.#redoStack = []; // yangi amal — redo tozalanadi
  }

  undo() {
    const cmd = this.#undoStack.pop();
    if (cmd) { cmd.undo(); this.#redoStack.push(cmd); }
  }

  redo() {
    const cmd = this.#redoStack.pop();
    if (cmd) { cmd.execute(); this.#undoStack.push(cmd); }
  }
}

const data = { name: "Ali", age: 25 };
const history = new CommandHistory();

history.execute(new SetValueCommand(data, "name", "Vali"));
console.log(data.name); // "Vali"

history.undo();
console.log(data.name); // "Ali"

history.redo();
console.log(data.name); // "Vali"
```

**Real-world:** VS Code undo/redo, Git commit'lar, Redux action'lar, Photoshop history, task queue'lar.

---

## Savol 11: Facade pattern nima? Real-world misol bering [Junior+]

**Javob:**

Facade — murakkab subsystem'ga oddiy, yagona interface beradi. Client kod subsystem'ning ichki tuzilishini bilmaydi.

```javascript
// Murakkab — 3 ta API bilan ishlash
class UserAPI { getUser(id) { /* ... */ } }
class OrderAPI { getOrders(userId) { /* ... */ } }
class NotificationAPI { send(userId, msg) { /* ... */ } }

// Facade — bitta oddiy method
class DashboardFacade {
  constructor() {
    this.userAPI = new UserAPI();
    this.orderAPI = new OrderAPI();
    this.notifAPI = new NotificationAPI();
  }

  async getUserDashboard(userId) {
    const [user, orders] = await Promise.all([
      this.userAPI.getUser(userId),
      this.orderAPI.getOrders(userId)
    ]);
    return { user, orders, totalOrders: orders.length };
  }
}

// Client kod — murakkablikni bilmaydi
const dashboard = new DashboardFacade();
const data = await dashboard.getUserDashboard(42);
```

**Real-world:** jQuery (`$`), `fetch` API (XMLHttpRequest ustidan), React `ReactDOM.render()`, Node.js `fs.readFile`.

---

## Savol 12: State pattern va if/else farqi nima? [Middle+]

**Javob:**

```javascript
// ❌ if/else state machine — yangi holat = barcha if'larni o'zgartirish
class Order {
  process(action) {
    if (this.state === "pending") {
      if (action === "pay") this.state = "paid";
      else if (action === "cancel") this.state = "cancelled";
    } else if (this.state === "paid") {
      if (action === "ship") this.state = "shipped";
      // yangi holat? Har joyga if qo'shish kerak...
    }
  }
}

// ✅ State pattern — har bir holat alohida class
class PendingState {
  pay(order) { order.setState(new PaidState()); }
  cancel(order) { order.setState(new CancelledState()); }
  ship() { throw new Error("Bu holatda mumkin emas"); }
}
class PaidState {
  pay() { throw new Error("Allaqachon to'langan"); }
  ship(order) { order.setState(new ShippedState()); }
  cancel(order) { order.setState(new CancelledState()); }
}
```

| | if/else | State pattern |
|--|---------|--------------|
| Yangi holat | Barcha if'larni o'zgartirish | Yangi class qo'shish |
| Noto'g'ri transition | if shartida yodda tutish | Class'da throw |
| O'qilishi | Murakkab conditional | Har bir holat aniq |

---

## Savol 13: Builder pattern qachon ishlatiladi? [Middle]

**Javob:**

Builder — constructor'ga ko'p parameter berish o'rniga, qadam-baqadam object yaratish. Method chaining bilan o'qilishi oson.

```javascript
// ❌ 8 ta parameter — qaysi biri nima?
new HttpRequest("GET", "/api/users", null, { "Auth": "Bearer ..." },
                30000, 3, true, "json");

// ✅ Builder — har bir qadam aniq
const request = new RequestBuilder()
  .method("GET")
  .url("/api/users")
  .header("Authorization", "Bearer ...")
  .timeout(30000)
  .retries(3)
  .responseType("json")
  .build();
```

**Qachon ishlatiladi:** 4+ parameter, ko'p optional konfiguratsiya, method chaining kerak bo'lganda.

**Real-world:** Knex.js query builder, Joi validation schema, Jest matchers, supertest API testing.

---

## Savol 14: Adapter pattern real-world misol bering [Middle]

**Javob:**

Adapter — mos kelmaydigan ikki interface'ni birga ishlashga majburlaydi. Eski API'ni yangi formatga moslashtirish uchun ishlatiladi.

```javascript
// Eski logger — o'z API'si bor
class LegacyLogger {
  writeLog(level, timestamp, message) {
    console.log(`[${level}] ${timestamp}: ${message}`);
  }
}

// Yangi loyiha — boshqa interface kutadi
// interface Logger { info(msg), error(msg), warn(msg) }

// Adapter — eski logger'ni yangi interface'ga moslashtiradi
class LoggerAdapter {
  constructor() {
    this.legacy = new LegacyLogger();
  }

  info(msg)  { this.legacy.writeLog("INFO",  new Date().toISOString(), msg); }
  error(msg) { this.legacy.writeLog("ERROR", new Date().toISOString(), msg); }
  warn(msg)  { this.legacy.writeLog("WARN",  new Date().toISOString(), msg); }
}

// Client kod — yangi interface bilan ishlaydi
const logger = new LoggerAdapter();
logger.info("Server started");
```

**Real-world:** Database driver adapter'lar, fetch polyfill (XMLHttpRequest → fetch interface), framework migration (jQuery → vanilla JS), third-party library wrapper'lar.

---

## Savol 15: Chain of Responsibility va Middleware farqi bormi? [Senior]

**Javob:**

Chain of Responsibility (CoR) va Middleware — asosan **bir xil pattern**. Middleware — CoR ning JavaScript/Node.js ekosistemadagi nomi. Farqlar juda kichik:

| | Classic CoR | Middleware |
|--|------------|-----------|
| Handler | Class bilan | Funksiya bilan |
| Next chaqirish | `this.successor.handle()` | `next()` callback |
| Async | Odatda sync | Ko'pincha async |
| Onion model | Yo'q | Ha (`await next()` dan keyin kod) |
| Real-world | Java/C# enterprise | Express, Koa, NestJS |

Middleware'ning **onion model**i — birinchi middleware `next()` dan keyin ham kod bajaradi:

```javascript
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();        // keyingi middleware'lar bajariladi
  const ms = Date.now() - start;
  console.log(`${ms}ms`); // barcha middleware tugagandan KEYIN ishlaydi
});
```

Bu classic CoR da odatda yo'q — handler so'rovni qayta ishlab keyingisiga uzatadi va qaytib kelmaydi.

---

## Savol 16: Qaysi pattern'ni qachon ishlatish kerak? [Senior]

**Javob:**

| Muammo | Pattern | Misol |
|--------|---------|-------|
| Object yaratish murakkab | **Factory** | createUser("admin") |
| Faqat bitta instance kerak | **Singleton** | DB connection pool |
| Constructor parametrlari ko'p | **Builder** | QueryBuilder().from().where() |
| Private/public farqlash | **Module** | ES Module export |
| Funksiyaga yangi behavior | **Decorator** | withLogging(fn) |
| Mos kelmaydigan API | **Adapter** | OldAPI → NewInterface |
| Murakkab subsystem | **Facade** | dashboard.getData() |
| Object kirishni nazorat | **Proxy** | JS Proxy, lazy loading |
| Holat o'zgarishini xabarlash | **Observer** | EventEmitter |
| Loose coupling xabarlash | **Pub/Sub** | EventBus, Redis |
| Algoritm almashtirish | **Strategy** | payment.setStrategy() |
| Undo/redo kerak | **Command** | editor.execute(cmd) |
| N*N aloqani kamaytirish | **Mediator** | ChatRoom |
| Holat asosidagi behavior | **State** | Order state machine |
| So'rov zanjiri | **Chain of Resp.** | Middleware pipeline |
| Collection traverse | **Iterator** | Symbol.iterator |

**Qoida:** Pattern — maqsad uchun vosita. Muammosiz pattern qo'llash — over-engineering. Avval muammoni aniqlang, keyin tegishli pattern'ni tanlang.
