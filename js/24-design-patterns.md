# Bo'lim 24: Design Patterns

> Design Pattern'lar — ko'p marta sinab ko'rilgan muammolarga tayyor yechimlar. Ular kodni tuzilmali, qayta ishlatiluvchi va tushunarliroq qiladi.

---

## Mundarija

- [Module Pattern](#module-pattern)
- [Singleton Pattern](#singleton-pattern)
- [Factory Pattern](#factory-pattern)
- [Observer Pattern (Pub/Sub)](#observer-pattern)
- [Strategy Pattern](#strategy-pattern)
- [Decorator Pattern](#decorator-pattern)
- [Command Pattern](#command-pattern)
- [Middleware Pattern](#middleware-pattern)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Module Pattern

### Nazariya

Module pattern — kodni inkapsulatsiya qilib, faqat kerakli qismlarni (public API) ochish, qolganini yashirish (private). IIFE (Immediately Invoked Function Expression) bilan closure orqali amalga oshirilgan — ES6 modullardan oldingi asosiy pattern. Revealing Module Pattern — barcha funksiyalarni private yozib, oxirida qaysilari public ekanligini aniq ko'rsatish. Zamonaviy JavaScript da `export`/`import` bilan modul scope avtomatik ta'minlanadi.

### Kod Misollari

```javascript
// === IIFE Module (klassik) ===
const Counter = (function() {
  // Private
  let count = 0;
  const MAX = 100;

  function validate(n) {
    if (n > MAX) throw new RangeError(`${n} > ${MAX}`);
  }

  // Public API
  return {
    increment() { validate(count + 1); return ++count; },
    decrement() { return --count; },
    getCount() { return count; },
    reset() { count = 0; }
  };
})();

Counter.increment(); // 1
Counter.increment(); // 2
Counter.getCount();  // 2
// Counter.count — ❌ undefined (private)

// === Revealing Module Pattern ===
const Calculator = (function() {
  let history = [];

  function add(a, b) { 
    const result = a + b;
    history.push(`${a} + ${b} = ${result}`);
    return result;
  }

  function getHistory() { return [...history]; }
  function clear() { history = []; }

  // "Ochish" — qaysi funksiyalar public ekanligini aniq ko'rsatish
  return { add, getHistory, clear };
})();

// === ES6 Module (zamonaviy) ===
// counter.js
let count = 0;
export const increment = () => ++count;
export const getCount = () => count;
// count tashqaridan ko'rinmaydi — modul scope
```

---

## Singleton Pattern

### Nazariya

Singleton — faqat **bitta instance** (namuna) mavjud bo'lishini ta'minlash. Global state, config, database connection, logger kabi resurslar uchun ishlatiladi. Class'da static private field + `getInstance()` metodi, yoki IIFE + closure bilan amalga oshiriladi. ES Module'larda har bir modul birinchi import'da execute bo'lib, keyin cache'lanadi — shuning uchun `export default {}` eng oddiy singleton hisoblanadi.

### Kod Misollari

```javascript
// === Closure bilan ===
const Database = (function() {
  let instance = null;

  class DB {
    constructor() {
      this.connection = "mongodb://localhost:27017";
      this.queries = [];
    }
    query(sql) {
      this.queries.push(sql);
      return `Result: ${sql}`;
    }
  }

  return {
    getInstance() {
      if (!instance) {
        instance = new DB();
        console.log("DB yaratildi");
      }
      return instance;
    }
  };
})();

const db1 = Database.getInstance(); // "DB yaratildi"
const db2 = Database.getInstance(); // (hech narsa — allaqachon bor)
console.log(db1 === db2); // true — BIR XIL instance

// === Class bilan ===
class AppConfig {
  static #instance = null;

  constructor() {
    if (AppConfig.#instance) {
      return AppConfig.#instance;
    }
    this.settings = { theme: "light", lang: "uz" };
    AppConfig.#instance = this;
  }

  get(key) { return this.settings[key]; }
  set(key, value) { this.settings[key] = value; }

  static getInstance() {
    return new AppConfig();
  }
}

const c1 = new AppConfig();
const c2 = new AppConfig();
console.log(c1 === c2); // true

// === ES Module Singleton (eng oddiy!) ===
// config.js — modul birinchi import da execute bo'ladi, keyin cache'lanadi
const config = {
  theme: "light",
  lang: "uz"
};
export default config; // har doim BIR XIL object
```

---

## Factory Pattern

### Nazariya

Factory — object yaratish logikasini bitta joyga markazlashtirish. Client qaysi aniq class yoki konfiguratsiya yaratilishini bilmasligi kerak — faqat turni aytadi, factory tegishli object qaytaradi. Simple Factory (switch/if bilan), Abstract Factory (bir necha bog'liq object'larni bir xil tema/style'da yaratish), va Factory Method (subclass'larda override qilish) variantlari bor.

### Kod Misollari

```javascript
// === Simple Factory ===
function createUser(type, data) {
  switch (type) {
    case "admin":
      return {
        ...data,
        role: "admin",
        permissions: ["read", "write", "delete", "manage"],
        dashboard: "/admin"
      };
    case "editor":
      return {
        ...data,
        role: "editor",
        permissions: ["read", "write"],
        dashboard: "/editor"
      };
    case "viewer":
      return {
        ...data,
        role: "viewer",
        permissions: ["read"],
        dashboard: "/home"
      };
    default:
      throw new Error(`Unknown user type: ${type}`);
  }
}

const admin = createUser("admin", { name: "Ali" });
const viewer = createUser("viewer", { name: "Vali" });

// === Abstract Factory ===
class UIFactory {
  createButton() { throw new Error("implement me"); }
  createInput() { throw new Error("implement me"); }
  createModal() { throw new Error("implement me"); }
}

class DarkUIFactory extends UIFactory {
  createButton(text) {
    return { type: "button", text, theme: "dark", bg: "#333", color: "#fff" };
  }
  createInput(placeholder) {
    return { type: "input", placeholder, theme: "dark", bg: "#222", border: "#555" };
  }
}

class LightUIFactory extends UIFactory {
  createButton(text) {
    return { type: "button", text, theme: "light", bg: "#fff", color: "#333" };
  }
  createInput(placeholder) {
    return { type: "input", placeholder, theme: "light", bg: "#fafafa", border: "#ddd" };
  }
}

function getUIFactory(theme) {
  return theme === "dark" ? new DarkUIFactory() : new LightUIFactory();
}

const factory = getUIFactory("dark");
const btn = factory.createButton("Yuborish");
// { type: "button", text: "Yuborish", theme: "dark", bg: "#333", color: "#fff" }
```

---

## Observer Pattern

### Nazariya

Observer (Pub/Sub) — bir object (publisher/emitter) o'zgarganda yoki event emit qilganda, barcha bog'liq object'lar (subscribers/listeners) avtomatik xabardor bo'ladi. EventEmitter class: `on()` (subscribe + unsubscribe funksiya qaytarish), `once()` (bir marta ishlash), `off()` (unsubscribe), `emit()` (event trigger). State management, DOM events, WebSocket, va komponentlar arasi aloqa uchun asosiy pattern.

### Kod Misollari

```javascript
// === EventEmitter implementatsiyasi ===
class EventEmitter {
  #events = new Map();

  on(event, callback) {
    if (!this.#events.has(event)) {
      this.#events.set(event, new Set());
    }
    this.#events.get(event).add(callback);
    // Unsubscribe funksiyasi
    return () => this.off(event, callback);
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(event, wrapper);
    };
    return this.on(event, wrapper);
  }

  off(event, callback) {
    const listeners = this.#events.get(event);
    if (listeners) {
      listeners.delete(callback);
      if (listeners.size === 0) this.#events.delete(event);
    }
  }

  emit(event, ...args) {
    const listeners = this.#events.get(event);
    if (listeners) {
      listeners.forEach(cb => cb(...args));
    }
  }

  removeAllListeners(event) {
    if (event) {
      this.#events.delete(event);
    } else {
      this.#events.clear();
    }
  }
}

// === Ishlatish ===
const emitter = new EventEmitter();

// Subscribe
const unsub = emitter.on("userLogin", (user) => {
  console.log(`${user.name} tizimga kirdi`);
});

emitter.once("userLogin", (user) => {
  console.log(`Birinchi login: ${user.name}`);
});

emitter.emit("userLogin", { name: "Ali" });
// "Ali tizimga kirdi"
// "Birinchi login: Ali"

emitter.emit("userLogin", { name: "Vali" });
// "Vali tizimga kirdi"
// (once — ishlamas)

unsub(); // Unsubscribe

// === State Management bilan ===
class Store extends EventEmitter {
  #state;

  constructor(initialState) {
    super();
    this.#state = initialState;
  }

  getState() { return { ...this.#state }; }

  setState(updater) {
    const oldState = this.#state;
    this.#state = typeof updater === "function" 
      ? { ...this.#state, ...updater(this.#state) }
      : { ...this.#state, ...updater };
    this.emit("change", this.#state, oldState);
  }
}

const store = new Store({ count: 0, user: null });

store.on("change", (newState, oldState) => {
  console.log("State o'zgardi:", newState);
});

store.setState({ count: 1 });        // State o'zgardi: { count: 1, user: null }
store.setState(s => ({ count: s.count + 1 })); // State o'zgardi: { count: 2, ... }
```

---

## Strategy Pattern

### Nazariya

Strategy — algoritmni alohida funksiya/class sifatida ifodalab, **runtime** da almashtirish imkonini beradi. if/else zanjiri o'rniga object'da strategiyalarni saqlash va key bo'yicha tanlash. Yangi strategiya qo'shish uchun mavjud kodni o'zgartirish shart emas (Open/Closed Principle). Validatsiya, narx hisoblash, tartiblash, autentifikatsiya usullari kabi holatlarda ishlatiladi.

### Kod Misollari

```javascript
// === Validatsiya strategiyalari ===
const validationStrategies = {
  required: (value) => ({
    valid: value !== "" && value !== null && value !== undefined,
    message: "Majburiy maydon"
  }),
  email: (value) => ({
    valid: /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
    message: "Noto'g'ri email format"
  }),
  minLength: (min) => (value) => ({
    valid: String(value).length >= min,
    message: `Kamida ${min} ta belgi kerak`
  }),
  phone: (value) => ({
    valid: /^\+998[0-9]{9}$/.test(value),
    message: "Noto'g'ri telefon raqam"
  })
};

function validate(data, rules) {
  const errors = {};
  for (const [field, fieldRules] of Object.entries(rules)) {
    const value = data[field];
    for (const rule of fieldRules) {
      const strategy = typeof rule === "function" ? rule : validationStrategies[rule];
      const result = strategy(value);
      if (!result.valid) {
        errors[field] = errors[field] || [];
        errors[field].push(result.message);
      }
    }
  }
  return { valid: Object.keys(errors).length === 0, errors };
}

// Ishlatish
const result = validate(
  { name: "", email: "invalid", phone: "123" },
  {
    name: ["required"],
    email: ["required", "email"],
    phone: ["required", "phone"]
  }
);
// { valid: false, errors: { name: [...], email: [...], phone: [...] } }

// === Narx hisoblash strategiyalari ===
const pricingStrategies = {
  regular: (price) => price,
  premium: (price) => price * 0.9,     // 10% chegirma
  vip: (price) => price * 0.8,         // 20% chegirma
  wholesale: (price, qty) => qty > 100 ? price * 0.7 : price * 0.85
};

function calculatePrice(price, qty, customerType = "regular") {
  const strategy = pricingStrategies[customerType];
  if (!strategy) throw new Error(`Unknown customer type: ${customerType}`);
  return strategy(price, qty) * qty;
}

calculatePrice(100, 10, "regular"); // 1000
calculatePrice(100, 10, "premium"); // 900
calculatePrice(100, 150, "wholesale"); // 10500
```

---

## Decorator Pattern

### Nazariya

Decorator — mavjud object yoki funksiyaga yangi xulq-atvor (logging, timing, caching, retry) qo'shish, **original kodni o'zgartirmasdan**. JavaScript da higher-order function sifatida amalga oshiriladi: original funksiyani oladi, yangi funksiya qaytaradi. Bir necha decorator'ni compose qilish mumkin (nesting yoki pipe). TC39 Decorator Proposal (Stage 3) class method'lar uchun `@decorator` sintaksisini taklif qiladi.

### Kod Misollari

```javascript
// === Funksiya Decorator'lari ===

// Logging decorator
function withLogging(fn, label) {
  return function(...args) {
    console.log(`[${label}] Chaqirildi:`, args);
    const result = fn.apply(this, args);
    console.log(`[${label}] Natija:`, result);
    return result;
  };
}

// Timing decorator
function withTiming(fn, label) {
  return function(...args) {
    const start = performance.now();
    const result = fn.apply(this, args);
    const time = (performance.now() - start).toFixed(2);
    console.log(`[${label}] ${time}ms`);
    return result;
  };
}

// Retry decorator
function withRetry(fn, maxRetries = 3, delay = 1000) {
  return async function(...args) {
    for (let i = 0; i <= maxRetries; i++) {
      try {
        return await fn.apply(this, args);
      } catch (error) {
        if (i === maxRetries) throw error;
        console.log(`Retry ${i + 1}/${maxRetries}...`);
        await new Promise(r => setTimeout(r, delay * (i + 1)));
      }
    }
  };
}

// Cache decorator
function withCache(fn, ttl = 60000) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    const cached = cache.get(key);
    if (cached && Date.now() - cached.time < ttl) {
      return cached.value;
    }
    const result = fn.apply(this, args);
    cache.set(key, { value: result, time: Date.now() });
    return result;
  };
}

// === Ishlatish — Compose ===
const fetchUser = withRetry(
  withLogging(
    withTiming(
      async (id) => {
        const res = await fetch(`/api/users/${id}`);
        return res.json();
      },
      "fetchUser"
    ),
    "fetchUser"
  ),
  3
);

// === TC39 Decorator Proposal (Stage 3) ===
// class UserService {
//   @logged
//   @timed
//   @cached(60000)
//   async getUser(id) { ... }
// }
```

---

## Command Pattern

### Nazariya

Command — operatsiyani (action) object sifatida ifodalash. Har bir command `execute()` va `undo()` metodlariga ega. Bu Undo/Redo, transaction queue, history, macro recording kabi imkoniyatlarni beradi. CommandManager command'lar stack'ini boshqaradi: execute (bajarish + history ga qo'shish), undo (oxirgi command'ni bekor qilish), redo (bekor qilingan command'ni qayta bajarish).

### Kod Misollari

```javascript
class CommandManager {
  #history = [];
  #undone = [];

  execute(command) {
    command.execute();
    this.#history.push(command);
    this.#undone = []; // redo tarixini tozalash
  }

  undo() {
    const command = this.#history.pop();
    if (command) {
      command.undo();
      this.#undone.push(command);
    }
  }

  redo() {
    const command = this.#undone.pop();
    if (command) {
      command.execute();
      this.#history.push(command);
    }
  }
}

// Text editor command'lari
class InsertTextCommand {
  constructor(editor, text, position) {
    this.editor = editor;
    this.text = text;
    this.position = position;
  }
  execute() { this.editor.insert(this.text, this.position); }
  undo() { this.editor.delete(this.position, this.text.length); }
}

class DeleteTextCommand {
  constructor(editor, position, length) {
    this.editor = editor;
    this.position = position;
    this.length = length;
    this.deleted = "";
  }
  execute() { this.deleted = this.editor.delete(this.position, this.length); }
  undo() { this.editor.insert(this.deleted, this.position); }
}

// Simple editor
class TextEditor {
  #content = "";
  get content() { return this.#content; }
  insert(text, pos) { 
    this.#content = this.#content.slice(0, pos) + text + this.#content.slice(pos);
  }
  delete(pos, len) {
    const deleted = this.#content.slice(pos, pos + len);
    this.#content = this.#content.slice(0, pos) + this.#content.slice(pos + len);
    return deleted;
  }
}

// Ishlatish
const editor = new TextEditor();
const manager = new CommandManager();

manager.execute(new InsertTextCommand(editor, "Salom", 0));
console.log(editor.content); // "Salom"

manager.execute(new InsertTextCommand(editor, " Dunyo", 5));
console.log(editor.content); // "Salom Dunyo"

manager.undo();
console.log(editor.content); // "Salom"

manager.redo();
console.log(editor.content); // "Salom Dunyo"
```

---

## Middleware Pattern

### Nazariya

Middleware — so'rov/ma'lumotni funksiyalar zanjiri bo'ylab ketma-ket qayta ishlash. Har bir middleware context va `next()` funksiyasini oladi — `next()` chaqirilsa keyingi middleware ishlaydi, chaqirilmasa zanjir to'xtaydi. Express.js, Koa, Redux ning asosiy arxitekturasi. Logging, autentifikatsiya, validatsiya, error handling kabi cross-cutting concern'larni ajratish uchun ideal.

### Kod Misollari

```javascript
class Pipeline {
  #middlewares = [];

  use(middleware) {
    this.#middlewares.push(middleware);
    return this; // chaining uchun
  }

  async execute(context) {
    let index = 0;
    const next = async () => {
      if (index < this.#middlewares.length) {
        const middleware = this.#middlewares[index++];
        await middleware(context, next);
      }
    };
    await next();
    return context;
  }
}

// === HTTP so'rov pipeline ===
const api = new Pipeline();

// Logger middleware
api.use(async (ctx, next) => {
  console.log(`→ ${ctx.method} ${ctx.url}`);
  const start = Date.now();
  await next();
  console.log(`← ${ctx.status} (${Date.now() - start}ms)`);
});

// Auth middleware
api.use(async (ctx, next) => {
  if (!ctx.headers?.authorization) {
    ctx.status = 401;
    ctx.body = { error: "Unauthorized" };
    return; // next() chaqirmaslik — zanjirni to'xtatish
  }
  ctx.user = { id: 1, name: "Ali" }; // token dan
  await next();
});

// Handler
api.use(async (ctx, next) => {
  ctx.status = 200;
  ctx.body = { message: `Salom, ${ctx.user.name}!` };
  await next();
});

// Ishlatish
api.execute({
  method: "GET",
  url: "/api/profile",
  headers: { authorization: "Bearer token123" }
});
// → GET /api/profile
// ← 200 (2ms)
```

---

## Common Mistakes

### ❌ Xato 1: Singleton ni noto'g'ri ishlatish

```javascript
// ❌ Global state hamma joyda — test qilish qiyin
const globalState = {
  user: null,
  theme: "light",
  data: []
};
// Har qanday fayl bu state ni o'zgartirishi mumkin
```

### ✅ To'g'ri usul:

```javascript
// ✅ Singleton aniq API bilan
class AppState {
  static #instance = null;
  #state = {};

  static getInstance() {
    if (!this.#instance) this.#instance = new AppState();
    return this.#instance;
  }

  get(key) { return this.#state[key]; }
  set(key, value) { this.#state[key] = value; }
  
  // Test uchun reset
  static reset() { this.#instance = null; }
}
```

**Nima uchun:** Ochiq global object modullar o'rtasida kutilmagan bog'lanishlarga olib keladi. Singleton — kontrollangan access va aniq API beradi.

---

### ❌ Xato 2: Observer da memory leak

```javascript
// ❌ Listener'larni tozalamaslik
class Component {
  init() {
    eventBus.on("update", this.handleUpdate.bind(this));
    // Component destroy bo'lganda — listener hali bor!
  }
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Cleanup bilan
class Component {
  #unsub = null;

  init() {
    this.#unsub = eventBus.on("update", this.handleUpdate.bind(this));
  }

  destroy() {
    this.#unsub?.(); // Listener tozalash
  }
}
```

**Nima uchun:** Event listener'lar GC ni bloklaydi. Component olib tashlanganda, uning listener'larini ham olib tashlash **majburiy**. `on()` dan unsubscribe funksiya qaytarish — best practice.

---

## Amaliy Mashqlar

### Mashq 1: EventBus implementatsiyasi (O'rta)

**Savol:** `createEventBus()` funksiyasi yarating. `on`, `once`, `off`, `emit` metodlari bo'lsin. Wildcard `*` event ham qo'llab-quvvatlansin.

<details>
<summary>Javob</summary>

```javascript
function createEventBus() {
  const events = new Map();

  return {
    on(event, callback) {
      if (!events.has(event)) events.set(event, new Set());
      events.get(event).add(callback);
      return () => this.off(event, callback);
    },

    once(event, callback) {
      const wrapper = (...args) => {
        this.off(event, wrapper);
        callback(...args);
      };
      return this.on(event, wrapper);
    },

    off(event, callback) {
      events.get(event)?.delete(callback);
    },

    emit(event, ...args) {
      events.get(event)?.forEach(cb => cb(...args));
      if (event !== "*") {
        events.get("*")?.forEach(cb => cb(event, ...args));
      }
    }
  };
}

// Test:
const bus = createEventBus();
bus.on("*", (event, ...args) => console.log(`[${event}]`, ...args));
bus.on("greet", (name) => console.log(`Salom, ${name}!`));
bus.once("greet", (name) => console.log(`Birinchi: ${name}`));

bus.emit("greet", "Ali");
// "[greet] Ali"
// "Salom, Ali!"
// "Birinchi: Ali"

bus.emit("greet", "Vali");
// "[greet] Vali"
// "Salom, Vali!"
```

</details>

---

### Mashq 2: Undo/Redo Manager (Qiyin)

**Savol:** `createUndoManager()` yarating. `perform(doFn, undoFn)`, `undo()`, `redo()`, `canUndo`, `canRedo`, `history` metodlari bo'lsin.

<details>
<summary>Javob</summary>

```javascript
function createUndoManager() {
  const done = [];
  const undone = [];

  return {
    perform(doFn, undoFn, label = "action") {
      doFn();
      done.push({ doFn, undoFn, label });
      undone.length = 0;
    },

    undo() {
      const action = done.pop();
      if (action) {
        action.undoFn();
        undone.push(action);
      }
      return action?.label;
    },

    redo() {
      const action = undone.pop();
      if (action) {
        action.doFn();
        done.push(action);
      }
      return action?.label;
    },

    get canUndo() { return done.length > 0; },
    get canRedo() { return undone.length > 0; },
    get history() { return done.map(a => a.label); }
  };
}

// Test:
const manager = createUndoManager();
let count = 0;

manager.perform(
  () => count += 10,
  () => count -= 10,
  "add 10"
);
console.log(count); // 10

manager.perform(
  () => count *= 2,
  () => count /= 2,
  "double"
);
console.log(count); // 20

manager.undo(); // "double"
console.log(count); // 10

manager.undo(); // "add 10"
console.log(count); // 0

manager.redo(); // "add 10"
console.log(count); // 10

console.log(manager.history); // ["add 10"]
console.log(manager.canUndo); // true
console.log(manager.canRedo); // true
```

</details>

---

## Xulosa

| Pattern | Maqsad | Kalit So'z |
|---------|--------|-----------|
| **Module** | Inkapsulatsiya, public/private | IIFE, ES modules, closure |
| **Singleton** | Faqat bitta instance | static instance, lazy init |
| **Factory** | Object yaratishni abstraksiya | switch/map, polymorphism |
| **Observer** | Event-driven aloqa | on/off/emit, pub/sub |
| **Strategy** | Algoritmni almashtirish | Object map, runtime tanlash |
| **Decorator** | Xulq-atvor qo'shish | Higher-order function, wrapper |
| **Command** | Operatsiyani object qilish | execute/undo, history |
| **Middleware** | Zanjirli qayta ishlash | use, next, pipeline |

> **Cross-references:** [10-closures.md](10-closures.md) (closure — Module, Singleton uchun asos), [08-classes.md](08-classes.md) (class, private fields — OOP patterns), [23-proxy-reflect.md](23-proxy-reflect.md) (Proxy — decorator, observer reactive), [19-events.md](19-events.md) (addEventListener — Observer DOM versiyasi), [09-functions.md](09-functions.md) (higher-order functions — decorator, strategy)
