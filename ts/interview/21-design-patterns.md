# Interview: Design Patterns TypeScript'da

> Factory, Abstract Factory, Singleton, Builder, Adapter, Facade, Proxy, Observer, Strategy, Command, State Machine, Repository, Result/Either, Middleware Pipeline — TypeScript'ning generics, discriminated unions, conditional types va branded types yordamida type-safe implementation'lari bo'yicha interview savollari. Junior+ dan Senior darajagacha.

---

## Mundarija

### Nazariy savollar
1. [Factory pattern — generic factory qanday yoziladi?](#savol-1-factory-pattern--generic-factory-qanday-yoziladi-middle) [Middle]
2. [Abstract Factory pattern qachon kerak?](#savol-2-abstract-factory-pattern-qachon-kerak-middle) [Middle+]
3. [Singleton TypeScript'da: qachon va qanday?](#savol-3-singleton-typescript-da-qachon-va-qanday-middle) [Middle]
4. [Builder pattern: fluent vs step builder?](#savol-4-builder-pattern-fluent-vs-step-builder-middle) [Middle+]
5. [Strategy pattern nima va qachon ishlatiladi?](#savol-5-strategy-pattern-nima-va-qachon-ishlatiladi-middle) [Middle]
6. [Adapter pattern — real-world misol](#savol-6-adapter-pattern--real-world-misol-junior) [Junior+]
7. [Facade va Adapter farqi?](#savol-7-facade-va-adapter-farqi-middle) [Middle]
8. [Proxy pattern — qaysi use case'lar?](#savol-8-proxy-pattern--qaysi-use-case-lar-middle) [Middle+]
9. [Observer pattern — typed EventEmitter](#savol-9-observer-pattern--typed-eventemitter-middle) [Middle]
10. [Command pattern: undo/redo qanday ishlaydi?](#savol-10-command-pattern-undoredo-qanday-ishlaydi-middle) [Middle+]
11. [State Machine discriminated union bilan](#savol-11-state-machine-discriminated-union-bilan-senior) [Senior]
12. [Result/Either pattern — exception o'rniga](#savol-12-resulteither-pattern--exception-orniga-middle) [Middle+]
13. [Repository pattern + generics?](#savol-13-repository-pattern--generics-middle) [Middle+]
14. [Middleware pipeline — onion model nima?](#savol-14-middleware-pipeline--onion-model-nima-middle) [Middle+]

### Output savollar
15. [Decorator composition + factory output](#savol-15-decorator-composition--factory-output-middle) [Middle+]
16. [Strategy + `this` context output](#savol-16-strategy--this-context-output-middle) [Middle+]
17. [Middleware execution order](#savol-17-middleware-execution-order-middle) [Middle+]

### Coding savollar
18. [Type-safe `EventEmitter<Events>`](#savol-18-type-safe-eventemitterevents-senior) [Senior]
19. [Generic `Result<T, E>` utility'lar](#savol-19-generic-resultt-e-utility-lar-middle) [Middle+]
20. [Generic `Repository<T>` + in-memory](#savol-20-generic-repositoryt--in-memory-middle) [Middle+]
21. [Type-safe Builder phantom types](#savol-21-type-safe-builder-phantom-types-senior) [Senior]
22. [Async middleware pipeline](#savol-22-async-middleware-pipeline-senior) [Senior]
23. [Command pattern undo/redo](#savol-23-command-pattern-undoredo-middle) [Middle+]

### Bug fix savollar
24. [Singleton + test state leak](#savol-24-singleton--test-state-leak-middle) [Middle+]
25. [Observer memory leak](#savol-25-observer-memory-leak-middle) [Middle+]

---

## Nazariy savollar

### Savol 1: Factory pattern — generic factory qanday yoziladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Factory — object yaratish logikasini bir joyga to'plab, client koddan yashiradi. TypeScript'da generic + type map orqali factory qaytaradigan type avtomatik aniqlanadi — type assertion kerak emas.

### To'liq tushuntirish

**Factory'ning maqsadlari:**
1. **Encapsulation** — object yaratish detail'ni yashirish
2. **Conditional creation** — environment yoki config'ga ko'ra turli implementation
3. **Centralization** — yaratish logikasi bitta joyda
4. **Type narrowing** — generic factory chaqiruvchiga aniq type qaytaradi

**Generic Factory anatomy (type map pattern):**

```typescript
// 1. Type map — key (literal) → type
interface Map {
  key1: Type1;
  key2: Type2;
}

// 2. Factory funksiya — generic K constrained to keyof Map
function create<K extends keyof Map>(key: K): Map[K]
```

**TypeScript'ning kuchi:**
- `K extends keyof Map` — faqat mavjud key qabul qiladi (compile-time)
- `Map[K]` — return type avtomatik
- Type assertion (`as`) kerak emas

**Factory turlari:**

1. **Simple factory** — funksiya parameter'ga ko'ra object qaytaradi
2. **Factory method** — class method, subclass override qila oladi
3. **Abstract factory** — bir-biriga bog'liq object oilasi yaratadi

### Kod misol

```typescript
// === Generic Factory — type map pattern ===
interface EmailNotification { type: "email"; to: string; subject: string; }
interface SMSNotification { type: "sms"; phone: string; message: string; }
interface PushNotification { type: "push"; token: string; title: string; }

interface NotificationMap {
  email: EmailNotification;
  sms: SMSNotification;
  push: PushNotification;
}

function createNotification<K extends keyof NotificationMap>(
  type: K,
  props: Omit<NotificationMap[K], "type">
): NotificationMap[K] {
  return { type, ...props } as NotificationMap[K];
}

const email = createNotification("email", {
  to: "ali@example.com",
  subject: "Welcome",
});
// type: EmailNotification — auto-inferred

email.subject; // ✅ — faqat email da

const sms = createNotification("sms", {
  phone: "+998901234567",
  message: "Code: 12345",
});
sms.phone; // ✅
// sms.subject; // ❌ — sms da yo'q

// createNotification("telegram", {}); // ❌ — "telegram" not in NotificationMap


// === Factory method pattern ===
abstract class TransportLogger {
  log(msg: string) {
    const transport = this.createTransport();
    transport.send(msg);
  }

  protected abstract createTransport(): { send(msg: string): void };
}

class FileLogger extends TransportLogger {
  protected createTransport() {
    return { send: (msg: string) => console.log(`[FILE] ${msg}`) };
  }
}

class HttpLogger extends TransportLogger {
  protected createTransport() {
    return { send: (msg: string) => fetch("/log", { method: "POST", body: msg }) };
  }
}
```

### Edge Cases

- **Discriminated union return:** generic factory + `type` literal — type narrowing oson (`if (n.type === "email") n.subject`)
- **Generic class factory:** `class Repository<T> {}` ni factory qilish — `function createRepo<T>(): Repository<T>` — caller `<T>` ko'rsatishi kerak
- **Lazy creation:** factory har chaqiruvda yangi instance yaratadi — costly bo'lsa, caching qo'shish kerak
- **Async factory:** factory `Promise` qaytarishi mumkin — caller `await`
- **Default config:** factory `Partial<Config>` qabul qiladi, default'lar merge qilinadi

### Follow-up savollar

1. **"Factory va constructor o'rtasidagi farq?"** — Constructor — class instance yaratadi. Factory — qanday class instantiate qilish detail'ni yashiradi (subclass, config bo'yicha)
2. **"Type map va discriminated union farqi?"** — Type map: factory input → output. Discriminated union: variant data structure. Bir-birini to'ldiradi (factory discriminated union qaytaradi)

</details>

---

### Savol 2: Abstract Factory pattern qachon kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Abstract Factory — bir-biriga bog'liq object oilasini yaratadi. Client concrete class'larni bilmaydi, faqat interface orqali ishlaydi. Theme switching, cross-platform UI, database driver'larida ishlatiladi.

### To'liq tushuntirish

**Factory vs Abstract Factory:**

| | Factory | Abstract Factory |
|---|---------|------------------|
| Yaratadi | Bir turdagi object | Object oilasi (bir nechta tur) |
| Konkretlik | Bir kelajak | Bir nechta — strategy |
| Misol | `createButton()` | `createUIKit()` — Button + Input + Modal |
| Murakkablik | Past | Yuqori |

**Strukturasi:**

```
AbstractFactory (interface)
├── createA(): ProductA
└── createB(): ProductB

ConcreteFactory1
├── createA(): ProductA1
└── createB(): ProductB1

ConcreteFactory2
├── createA(): ProductA2
└── createB(): ProductB2
```

Client `AbstractFactory` ga bog'lanadi, runtime'da concrete factory tanlanadi.

**Use case'lar:**
1. **Theme system** — Material/Bootstrap/Tailwind theme uchun Button, Input, Modal
2. **Cross-platform** — Web/Mobile/Desktop UI komponentlari
3. **Database driver** — Postgres/MySQL/SQLite uchun connection, query, transaction
4. **Cloud provider** — AWS/GCP/Azure uchun storage, compute, queue

### Kod misol

```typescript
// === Abstract Factory — Theme system ===
interface Button {
  render(): string;
  onClick(handler: () => void): void;
}

interface Input {
  render(): string;
  getValue(): string;
}

interface Modal {
  render(): string;
  open(): void;
  close(): void;
}

interface UIFactory {
  createButton(label: string): Button;
  createInput(placeholder: string): Input;
  createModal(title: string): Modal;
}


// === Concrete factory 1: Material Design ===
class MaterialFactory implements UIFactory {
  createButton(label: string): Button {
    return {
      render: () => `<md-button>${label}</md-button>`,
      onClick: (h) => { /* Material click handler */ },
    };
  }

  createInput(placeholder: string): Input {
    return {
      render: () => `<md-input placeholder="${placeholder}"/>`,
      getValue: () => "",
    };
  }

  createModal(title: string): Modal {
    return {
      render: () => `<md-dialog>${title}</md-dialog>`,
      open: () => { /* ... */ },
      close: () => { /* ... */ },
    };
  }
}


// === Concrete factory 2: Bootstrap ===
class BootstrapFactory implements UIFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="btn btn-primary">${label}</button>`,
      onClick: (h) => { /* Bootstrap click */ },
    };
  }

  createInput(placeholder: string): Input {
    return {
      render: () => `<input class="form-control" placeholder="${placeholder}"/>`,
      getValue: () => "",
    };
  }

  createModal(title: string): Modal {
    return {
      render: () => `<div class="modal">${title}</div>`,
      open: () => { /* ... */ },
      close: () => { /* ... */ },
    };
  }
}


// === Client — factory ga bog'liq ===
function buildLoginForm(factory: UIFactory) {
  const usernameInput = factory.createInput("Username");
  const passwordInput = factory.createInput("Password");
  const submitButton = factory.createButton("Login");
  const errorModal = factory.createModal("Error");

  return {
    render: () =>
      [usernameInput, passwordInput, submitButton, errorModal]
        .map((c) => c.render())
        .join("\n"),
  };
}


// === Runtime switching ===
const theme = process.env.UI_THEME === "bootstrap"
  ? new BootstrapFactory()
  : new MaterialFactory();

const form = buildLoginForm(theme);
console.log(form.render());
```

### Edge Cases

- **Family consistency:** abstract factory product'lar bir-biriga mos kelishi kerak (Material Button + Material Input — to'g'ri, Material Button + Bootstrap Input — chaos)
- **Adding new product:** har concrete factory'ni yangilash kerak (open-closed principle buzilishi)
- **Adding new family:** oson — yangi concrete factory
- **TypeScript benefit:** factory interface har method return type'ni belgilaydi — concrete factory'lar majburiy implement
- **Generic abstract factory:** `interface UIFactory<T>` — type-safe customization

### Follow-up savollar

1. **"Factory pattern'dan qachon Abstract Factory'ga o'tish kerak?"** — Bir tur object'dan bir nechta tur object kerak bo'lganda. Family consistency muhim bo'lsa
2. **"DI Container abstract factory'ni almashtira oladimi?"** — Ha, ko'p case'da. Container token bo'yicha implementation tanlaydi. Lekin abstract factory — declarative, type-safe

</details>

---

### Savol 3: Singleton TypeScript'da: qachon va qanday? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Singleton — class'dan faqat bitta instance yaratilishini kafolatlaydi. TypeScript'da `private constructor` + `static getInstance()` orqali. Module-level variable bilan ham singleton — ko'pincha yaxshiroq variant. Anti-pattern hisoblanishi mumkin (global state).

### To'liq tushuntirish

**Singleton'ning ikkita asosiy implementation:**

**1. Class-based Singleton:**

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection | null = null;
  private constructor(private connectionString: string) {}

  static getInstance(connectionString = "default"): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection(connectionString);
    }
    return DatabaseConnection.instance;
  }
}

// new DatabaseConnection("..."); // ❌ private constructor
const db = DatabaseConnection.getInstance(); // ✅
```

**2. Module-level Singleton (yaxshiroq):**

```typescript
// db.ts
const connection = new SomeDBDriver("postgres://localhost/db");
export default connection;

// Har joyda import bir xil instance
```

**Module-level'ning afzalligi:**
- Oddiy syntax
- Native JS module caching
- Test'da easier — module mock
- Lazy initialization tabiiy

**Singleton'ning kamchiliklari:**
1. **Global state** — test isolation muammosi
2. **Tight coupling** — har joyda `Class.getInstance()` chaqiruvi
3. **DI bilan ziddiyat** — singleton dependency inject qilishni qiyinlashtiradi
4. **Multi-threading** (Node worker) — har worker o'z singleton instance

**Qachon kerak:**
- **Costly resource** — database connection pool, file handler
- **Shared mutable state** — application config, cache
- **Logger** — har joyda bir xil logger kerak

**Singleton vs DI:**
- DI container singleton scope — controlled singleton
- Pure singleton (`Class.getInstance()`) — global, test qiyin

### Kod misol

```typescript
// === 1. Class-based Singleton ===
class ConfigService {
  private static instance: ConfigService | null = null;
  private config: Record<string, any> = {};

  private constructor() {
    this.config = this.loadConfig();
  }

  static getInstance(): ConfigService {
    if (!ConfigService.instance) {
      ConfigService.instance = new ConfigService();
    }
    return ConfigService.instance;
  }

  private loadConfig() {
    return { apiUrl: process.env.API_URL, dbUrl: process.env.DB_URL };
  }

  get<T>(key: string): T { return this.config[key]; }

  // Test uchun reset (ehtiyot bo'ling — production da xavfli)
  static reset(): void { ConfigService.instance = null; }
}

const config = ConfigService.getInstance();
console.log(config.get<string>("apiUrl"));


// === 2. Module-level Singleton (afzal) ===
// config.ts
class Config {
  private config: Record<string, any>;
  constructor() {
    this.config = { apiUrl: process.env.API_URL };
  }
  get<T>(key: string): T { return this.config[key]; }
}

export const config = new Config();
// Har import bir xil instance — JS module caching


// === 3. Lazy module-level ===
// db.ts
let _connection: DatabaseConnection | null = null;

export function getConnection(): DatabaseConnection {
  if (!_connection) {
    _connection = new DatabaseConnection(process.env.DB_URL!);
  }
  return _connection;
}

// Test uchun
export function resetConnection(): void {
  _connection = null;
}


// === 4. DI Container Singleton (zamonaviy) ===
// Container.bind(LoggerToken).to(ConsoleLogger).inSingletonScope()
// const logger = container.resolve(LoggerToken); // Har resolve bir xil instance
```

### Edge Cases

- **Inheritance:** singleton class'ni extend qilish murakkab — child class o'zining singleton bo'lishi kerakmi yoki parent'ni share qiladimi?
- **Multi-process (cluster, worker_threads):** har worker o'z singleton — process-level emas, worker-level
- **Test isolation:** singleton state leak — `Class.reset()` yoki yangi container har test'da
- **`Object.freeze`:** `Object.freeze(singleton)` — mutation oldini olish
- **TC39 `@singleton` decorator:** class decorator orqali singleton — declarative

### Follow-up savollar

1. **"Module-level va class-based singleton orasidagi farq?"** — Module-level: oddiy, lekin lazy yo'q (bir paytda yaratiladi). Class-based: lazy, lekin verbose
2. **"Singleton anti-pattern — nima uchun?"** — Global state, test coupling, DI bilan ziddiyat. DI container singleton scope — controlled alternative

</details>

---

### Savol 4: Builder pattern: fluent vs step builder? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Fluent Builder** — method chaining bilan oddiy syntax, lekin `build()` da runtime error (kerak field yo'q). **Step Builder** — phantom types orqali compile-time enforcement: `build()` faqat barcha required field bo'lganda available.

### To'liq tushuntirish

**Builder pattern maqsadi:**

Murakkab object'ni bosqichma-bosqich yaratish — ko'p parameter li constructor o'rniga. Telescoping constructor anti-pattern'dan qutqaradi.

**Fluent Builder:**

```typescript
class Builder {
  private data: Partial<Request> = {};
  url(u: string): this { this.data.url = u; return this; }
  method(m: string): this { this.data.method = m; return this; }
  build(): Request {
    if (!this.data.url) throw new Error("url required");
    return this.data as Request;
  }
}

new Builder().url("/api").build(); // ❌ Runtime error
```

**Step Builder (compile-time enforcement):**

Phantom types — generic type parameter har setter'da yangilanadi. `build()` faqat to'liq state'da ishlaydi (TypeScript `this` parameter'da check).

```typescript
class Builder<S = {}> {
  url(u: string): Builder<S & { url: true }> { /* ... */ }
  method(m: string): Builder<S & { method: true }> { /* ... */ }
  build(this: Builder<{ url: true; method: true }>): Request { /* ... */ }
}

new Builder().url("/api").build(); // ❌ Compile error — method missing
new Builder().url("/api").method("GET").build(); // ✅
```

**Phantom types — runtime overhead yo'q:** type-level ma'lumot, JS'ga compile bo'lganda yo'qoladi.

**Trade-off:**

| | Fluent | Step |
|---|--------|------|
| Compile-time safety | Yo'q | Bor |
| Runtime safety | `build()` da | Ortiqcha emas |
| Type complexity | Past | Yuqori |
| IDE autocomplete | Barcha method | Faqat next step |
| Verbose | Kam | Ko'proq |
| Use case | Optional fields ko'p | Required field strict |

### Kod misol

```typescript
// === 1. Fluent Builder (sodda) ===
interface HttpRequest {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  headers: Record<string, string>;
  body?: unknown;
  timeout: number;
}

class FluentRequestBuilder {
  private request: Partial<HttpRequest> = { headers: {}, timeout: 5000 };

  url(url: string): this { this.request.url = url; return this; }
  method(m: HttpRequest["method"]): this { this.request.method = m; return this; }
  header(k: string, v: string): this {
    this.request.headers![k] = v;
    return this;
  }
  body(data: unknown): this { this.request.body = data; return this; }
  timeout(ms: number): this { this.request.timeout = ms; return this; }

  build(): HttpRequest {
    if (!this.request.url || !this.request.method) {
      throw new Error("url and method required"); // Runtime check
    }
    return this.request as HttpRequest;
  }
}

const req1 = new FluentRequestBuilder()
  .url("/api/users")
  .method("POST")
  .header("Content-Type", "application/json")
  .body({ name: "Ali" })
  .build();


// === 2. Step Builder (compile-time enforcement) ===
type BuilderState = Partial<Record<"url" | "method", true>>;

class StepRequestBuilder<State extends BuilderState = {}> {
  private data: Partial<HttpRequest> = { headers: {} };

  url(url: string): StepRequestBuilder<State & { url: true }> {
    this.data.url = url;
    return this as any;
  }

  method(m: HttpRequest["method"]): StepRequestBuilder<State & { method: true }> {
    this.data.method = m;
    return this as any;
  }

  header(k: string, v: string): StepRequestBuilder<State> {
    this.data.headers = { ...this.data.headers, [k]: v };
    return this as any;
  }

  body(data: unknown): StepRequestBuilder<State> {
    this.data.body = data;
    return this as any;
  }

  // build() faqat url VA method bor bo'lganda
  build(this: StepRequestBuilder<{ url: true; method: true }>): HttpRequest {
    return {
      url: this.data.url!,
      method: this.data.method!,
      headers: this.data.headers ?? {},
      body: this.data.body,
      timeout: 5000,
    };
  }
}

const req2 = new StepRequestBuilder()
  .url("/api/users")
  .method("POST")
  .build(); // ✅

// new StepRequestBuilder().url("/api").build();
// ❌ Compile error:
// Type 'StepRequestBuilder<{ url: true }>' is not assignable to
// 'StepRequestBuilder<{ url: true; method: true }>'


// === 3. Step Builder ordered (har bosqich faqat keyingi method ko'rinadi) ===
interface NeedsUrl {
  url(u: string): NeedsMethod;
}

interface NeedsMethod {
  method(m: HttpRequest["method"]): OptionalSteps;
}

interface OptionalSteps {
  header(k: string, v: string): OptionalSteps;
  body(data: unknown): OptionalSteps;
  timeout(ms: number): OptionalSteps;
  build(): HttpRequest;
}

class OrderedRequestBuilder implements NeedsUrl, NeedsMethod, OptionalSteps {
  private req: Partial<HttpRequest> = { headers: {}, timeout: 5000 };

  url(u: string): NeedsMethod { this.req.url = u; return this; }
  method(m: HttpRequest["method"]): OptionalSteps { this.req.method = m; return this; }
  header(k: string, v: string): OptionalSteps {
    this.req.headers![k] = v;
    return this;
  }
  body(data: unknown): OptionalSteps { this.req.body = data; return this; }
  timeout(ms: number): OptionalSteps { this.req.timeout = ms; return this; }

  build(): HttpRequest { return this.req as HttpRequest; }
}

function request(): NeedsUrl { return new OrderedRequestBuilder(); }

const req3 = request().url("/api").method("GET").build(); // ✅
// request().method("GET").url("/api").build(); // ❌ — method() NeedsUrl da yo'q
// request().url("/api").build(); // ❌ — build() NeedsMethod da yo'q
```

### Edge Cases

- **Immutability:** har setter `this` qaytaradi (mutation). Immutable variant — yangi builder instance qaytarish (memory cost)
- **Reuse:** bir builder'ni qayta ishlatish (`builder.build()` ikki marta) — internal state shared. Yangi builder afzal har build uchun
- **Type inference limits:** complex phantom type — IDE tooltip "ugly". `interface` based step builder readable
- **Performance:** runtime overhead yo'q (mutation). Phantom types compile-time only
- **Inheritance:** step builder generic extend qilish murakkab — type parameter chain manage qilish kerak

### Follow-up savollar

1. **"Fluent va Step builder qaysi yaxshiroq?"** — Optional fields ko'p — Fluent. Required field strict — Step. Library API — Step (developer experience yuqori)
2. **"`this` parameter type — qanday ishlaydi?"** — TypeScript phantom — `this: StepBuilder<{ url: true; method: true }>` — call site'da `this` type mos kelishi kerak
3. **"Ordered step (NeedsUrl → NeedsMethod) qanday foydali?"** — IDE autocomplete har bosqichda faqat valid method ko'rsatadi. Discoverability yuqori

</details>

---

### Savol 5: Strategy pattern nima va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Strategy — algoritmni interface orqali abstract qilib, runtime'da almashtirishga imkon beradi. `if/else` yoki `switch` ning OOP alternative. Open/Closed Principle'ga rioya qiladi — yangi strategy qo'shilganda mavjud kod o'zgarmaydi.

### To'liq tushuntirish

**Strategy strukturasi:**

```
Context
└── strategy: IStrategy

IStrategy (interface)
└── execute(): Result

ConcreteStrategy1 implements IStrategy
ConcreteStrategy2 implements IStrategy
```

**Use case'lar:**
1. **Sorting** — bubble/quick/merge sort
2. **Pricing** — regular/bulk/seasonal pricing
3. **Validation** — email/phone/credit card validation
4. **Formatting** — CSV/JSON/XML output
5. **Payment** — Stripe/PayPal/crypto

**`switch` o'rniga Strategy:**

```typescript
// ❌ Switch — yangi turi qo'shilganda kod o'zgaradi
function calculatePrice(type: string, price: number) {
  switch (type) {
    case "regular": return price;
    case "bulk": return price * 0.8;
    case "seasonal": return price * 1.5;
  }
}

// ✅ Strategy — yangi turi qo'shilganda mavjud kod o'zgarmaydi
interface PricingStrategy {
  calculate(basePrice: number): number;
}

class RegularPricing implements PricingStrategy { /* ... */ }
class BulkPricing implements PricingStrategy { /* ... */ }
class SeasonalPricing implements PricingStrategy { /* ... */ }
```

**TypeScript generic Strategy:**

```typescript
interface SortStrategy<T> {
  sort(data: T[]): T[];
}
```

Type-safe — har strategy bir xil element type uchun.

### Kod misol

```typescript
// === Strategy pattern — pricing ===
interface PricingStrategy {
  calculate(basePrice: number, quantity: number): number;
  describe(): string;
}

class RegularPricing implements PricingStrategy {
  calculate(basePrice: number, quantity: number): number {
    return basePrice * quantity;
  }
  describe() { return "Regular pricing"; }
}

class BulkPricing implements PricingStrategy {
  calculate(basePrice: number, quantity: number): number {
    const discount = quantity >= 100 ? 0.20 : quantity >= 50 ? 0.10 : 0;
    return basePrice * quantity * (1 - discount);
  }
  describe() { return "Bulk discount (10-20%)"; }
}

class SeasonalPricing implements PricingStrategy {
  constructor(private multiplier: number, private season: string) {}
  calculate(basePrice: number, quantity: number): number {
    return basePrice * quantity * this.multiplier;
  }
  describe() { return `Seasonal pricing (${this.season} × ${this.multiplier})`; }
}


// === Context — strategy ni runtime da almashtirish ===
class OrderCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  getTotal(basePrice: number, quantity: number): number {
    return this.strategy.calculate(basePrice, quantity);
  }

  getDescription(): string {
    return this.strategy.describe();
  }
}

const calc = new OrderCalculator(new RegularPricing());
console.log(calc.getTotal(10, 5));   // 50 — regular

calc.setStrategy(new BulkPricing());
console.log(calc.getTotal(10, 100));  // 800 — 20% discount

calc.setStrategy(new SeasonalPricing(1.5, "Holiday"));
console.log(calc.getTotal(10, 5));    // 75 — 1.5x


// === Generic Strategy — sorting ===
interface SortStrategy<T> {
  sort(data: T[]): T[];
}

class BubbleSort<T> implements SortStrategy<T> {
  constructor(private compare: (a: T, b: T) => number) {}
  sort(data: T[]): T[] {
    const arr = [...data];
    for (let i = 0; i < arr.length; i++) {
      for (let j = 0; j < arr.length - i - 1; j++) {
        if (this.compare(arr[j], arr[j + 1]) > 0) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    return arr;
  }
}

class QuickSort<T> implements SortStrategy<T> {
  constructor(private compare: (a: T, b: T) => number) {}
  sort(data: T[]): T[] {
    if (data.length <= 1) return data;
    const pivot = data[0];
    const left = data.slice(1).filter((x) => this.compare(x, pivot) < 0);
    const right = data.slice(1).filter((x) => this.compare(x, pivot) >= 0);
    return [...this.sort(left), pivot, ...this.sort(right)];
  }
}

const numCompare = (a: number, b: number) => a - b;
const sorter: SortStrategy<number> = new QuickSort(numCompare);
console.log(sorter.sort([5, 2, 8, 1, 9])); // [1, 2, 5, 8, 9]
```

### Edge Cases

- **State sharing:** strategy stateless bo'lishi afzal — bir strategy ko'p context'da ishlatilishi mumkin
- **Strategy bilan parameter:** har strategy turli parameter'ga muhtoj bo'lsa — interface universalligi pasayadi. Generic constraint yoki object parameter
- **Default strategy:** context constructor'da default strategy berish — bo'sh state oldini olish
- **Strategy chain:** bir nechta strategy ketma-ket — chain of responsibility'ga yaqinroq
- **Function as strategy:** simple case'da interface o'rniga function type yetarli — `type Validator = (input: string) => boolean`

### Follow-up savollar

1. **"Strategy va Template Method farqi?"** — Strategy — runtime composition. Template Method — inheritance, hook method override
2. **"Strategy pattern Function type bilan almashtirsa bo'ladimi?"** — Single-method interface — function type oddiyroq. Lekin strategy state yoki bir necha method bo'lsa — interface afzal
3. **"`switch` qachon Strategy'ga teng?"** — `switch` 3-4 case'dan kam bo'lsa, va yangi case kam qo'shiladigan bo'lsa — `switch` yetarli. Aks holda Strategy

</details>

---

### Savol 6: Adapter pattern — real-world misol [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Adapter — mos kelmaydigan interface'larni moslashtiradi. Legacy API yoki tashqi library'ni ichki system interface'ga moslash uchun ishlatiladi. Yangi kodda `target interface` bilan ishlanadi, adapter eski API'ga chaqiruvni o'zgartiradi.

### To'liq tushuntirish

**Adapter'ning maqsadi:**

Bir-biriga to'g'ri kelmaydigan ikki interface'ni ulash. Client kod faqat o'z interface'ini biladi — implementation detail (eski API, tashqi library) yashirin.

**Strukturasi:**

```
Target (interface bizning system kutadi)
└── method(): T

Adaptee (eski yoki tashqi class)
└── oldMethod(): T

Adapter implements Target
└── method(): T  → adaptee.oldMethod()
```

**Use case'lar:**
1. **Legacy API integration** — eski payment gateway API'ni yangi `PaymentProcessor` interface'ga
2. **Third-party library** — boshqa shape'ga ega kutubxonani bizning interface'ga
3. **Migration** — yangi va eski API ikkalasi production'da, adapter ko'prik
4. **Multiple providers** — Stripe, PayPal, Square — har biri uchun adapter, client bir interface

**Object Adapter vs Class Adapter:**
- **Object Adapter** (composition) — adaptee'ni constructor'da olib saqlash. JS/TS afzal
- **Class Adapter** (multiple inheritance) — JS'da yo'q. Mixin orqali emulate qilish mumkin

### Kod misol

```typescript
// === Real-world: Payment Processor Adapter ===

// Target — bizning system interface
interface PaymentProcessor {
  charge(amount: number, currency: string): Promise<{ transactionId: string }>;
  refund(transactionId: string): Promise<boolean>;
}


// === Adaptee 1: Legacy PayPal SDK ===
class LegacyPayPalSDK {
  async makePayment(cents: number, curr: string): Promise<string> {
    // Returns transaction ID
    return `pp-${Date.now()}`;
  }

  async reversePayment(id: string): Promise<{ success: boolean; reason?: string }> {
    return { success: true };
  }
}

// Adapter — LegacyPayPalSDK ni PaymentProcessor ga moslashtiradi
class PayPalAdapter implements PaymentProcessor {
  constructor(private paypal: LegacyPayPalSDK) {}

  async charge(amount: number, currency: string) {
    // Dollars to cents
    const cents = Math.round(amount * 100);
    const id = await this.paypal.makePayment(cents, currency);
    return { transactionId: id };
  }

  async refund(transactionId: string) {
    const result = await this.paypal.reversePayment(transactionId);
    return result.success;
  }
}


// === Adaptee 2: Stripe SDK (boshqa interface) ===
class StripeSDK {
  async createCharge(opts: { amount: number; currency: string }): Promise<{ id: string; status: string }> {
    return { id: `ch-${Date.now()}`, status: "succeeded" };
  }

  async createRefund(chargeId: string): Promise<{ status: "succeeded" | "failed" }> {
    return { status: "succeeded" };
  }
}

class StripeAdapter implements PaymentProcessor {
  constructor(private stripe: StripeSDK) {}

  async charge(amount: number, currency: string) {
    const charge = await this.stripe.createCharge({
      amount: Math.round(amount * 100),
      currency: currency.toLowerCase(),
    });
    return { transactionId: charge.id };
  }

  async refund(transactionId: string) {
    const result = await this.stripe.createRefund(transactionId);
    return result.status === "succeeded";
  }
}


// === Client — faqat PaymentProcessor bilan ishlaydi ===
async function processOrder(payment: PaymentProcessor, amount: number) {
  try {
    const result = await payment.charge(amount, "USD");
    console.log(`Charged: ${result.transactionId}`);
    return result;
  } catch (e) {
    console.error("Payment failed:", e);
    throw e;
  }
}

// Runtime tanlov
const provider = process.env.PAYMENT === "stripe"
  ? new StripeAdapter(new StripeSDK())
  : new PayPalAdapter(new LegacyPayPalSDK());

await processOrder(provider, 99.99);
// Client kod — provider detail bilmaydi
```

### Edge Cases

- **Two-way adapter:** ikki tomonlama moslash kerak bo'lganda (har ikkala interface ham bizniki) — kamdan-kam
- **Performance overhead:** adapter har chaqiruvda data transformation — `amount * 100` kabi. Hot path'da minimal saqlash kerak
- **Error mapping:** adaptee error'ni target interface error type'ga aylantirish (`StripeError` → `PaymentError`)
- **Side effect leak:** adaptee'ning side effect'lari adapter orqali tashqariga chiqishi mumkin (logging, telemetry)
- **Async vs sync mismatch:** sync adaptee → async target — Promise wrap. Async adaptee → sync target — qiyin (callback hell)

### Follow-up savollar

1. **"Adapter va Facade farqi?"** — Adapter: interface mismatch (1-to-1 method mapping). Facade: complexity hiding (subsystem'ga sodda entry point)
2. **"Adapter qachon over-engineering?"** — Adaptee va target deyarli bir xil — adapter qatlam keraksiz. Direct usage afzal
3. **"Class Adapter JS'da qanday?"** — JS multiple inheritance qo'llab-quvvatlamaydi. Mixin pattern bilan emulate, lekin Object Adapter ko'pincha afzal

</details>

---

### Savol 7: Facade va Adapter farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Adapter** — bir interface'ni boshqa interface'ga moslaydi (1-to-1 method mapping). **Facade** — murakkab subsystem'ni sodda interface bilan o'rab oladi (1-to-many). Adapter — compatibility, Facade — simplicity.

### To'liq tushuntirish

**Tafovutlar:**

| | Adapter | Facade |
|---|---------|--------|
| Maqsad | Interface compatibility | Complexity hiding |
| Strukturasi | Bir adaptee'ga moslash | Bir nechta subsystem class'ni birlashtirish |
| Method mapping | 1-to-1 | 1-to-many |
| Use case | Legacy/third-party integratsiya | API simplification |
| Misol | `StripeAdapter` | `AuthFacade` |

**Adapter strukturasi:**

```
Target ← Adapter → Adaptee
```

**Facade strukturasi:**

```
Client → Facade → [Subsystem A, Subsystem B, Subsystem C]
```

**Facade qachon kerak:**
1. **Complex subsystem** — ko'p step kerak bo'lgan operatsiya
2. **API entry point** — public API yashirin internal class'lar bilan
3. **Module isolation** — qaysi class'lar tashqariga chiqishini cheklash
4. **Workflow encapsulation** — business process'ni bir method'ga to'plash

### Kod misol

```typescript
// === Facade: Authentication workflow ===

// Subsystem class lar
class AuthService {
  async login(email: string, password: string): Promise<{ token: string }> {
    return { token: "jwt-token-" + Math.random() };
  }

  async verifyToken(token: string): Promise<{ valid: boolean; userId: string }> {
    return { valid: true, userId: "user-1" };
  }
}

class UserService {
  async getProfile(userId: string) {
    return { id: userId, name: "Ali", email: "ali@example.com" };
  }

  async updateLastLogin(userId: string) {
    console.log(`Updated last login for ${userId}`);
  }
}

class NotificationService {
  async sendWelcomeBack(email: string, name: string) {
    console.log(`Welcome back email to ${email}`);
  }
}

class AnalyticsService {
  trackEvent(event: string, data: Record<string, any>) {
    console.log(`[Analytics] ${event}`, data);
  }
}


// === Facade — barcha murakkablikni yashiradi ===
interface LoginResult {
  token: string;
  profile: { id: string; name: string; email: string };
}

class AuthFacade {
  constructor(
    private auth: AuthService,
    private users: UserService,
    private notifications: NotificationService,
    private analytics: AnalyticsService
  ) {}

  async loginAndSetup(email: string, password: string): Promise<LoginResult> {
    // Subsystem orchestration — bir method da
    const { token } = await this.auth.login(email, password);
    const { userId } = await this.auth.verifyToken(token);
    const profile = await this.users.getProfile(userId);

    await this.users.updateLastLogin(userId);
    await this.notifications.sendWelcomeBack(profile.email, profile.name);

    this.analytics.trackEvent("user.login", {
      userId: profile.id,
      email: profile.email,
    });

    return { token, profile };
  }

  async logout(token: string): Promise<void> {
    const { userId } = await this.auth.verifyToken(token);
    this.analytics.trackEvent("user.logout", { userId });
    // Token invalidation, cleanup
  }
}


// === Client — faqat facade bilan ishlaydi ===
const facade = new AuthFacade(
  new AuthService(),
  new UserService(),
  new NotificationService(),
  new AnalyticsService()
);

const result = await facade.loginAndSetup("ali@example.com", "secret");
console.log(`Logged in: ${result.profile.name}`);
// Client AuthService, UserService, NotificationService, AnalyticsService — bilmaydi


// === Comparison: Adapter — 1-to-1 mapping ===
interface ModernLogger { info(msg: string): void; warn(msg: string): void; }
interface LegacyLogger { writeLog(level: number, msg: string): void; }

class LoggerAdapter implements ModernLogger {
  constructor(private legacy: LegacyLogger) {}
  info(msg: string) { this.legacy.writeLog(0, msg); }
  warn(msg: string) { this.legacy.writeLog(1, msg); }
}
// 1-to-1: info → writeLog(0, ...), warn → writeLog(1, ...)
```

### Edge Cases

- **Facade monolith object:** facade juda ko'p method bo'lsa — bir nechta facade'ga ajratish kerak (har subsystem area uchun)
- **Hidden subsystem evolution:** facade orqali ishlovchi client subsystem o'zgarishini sezmaydi — backward compatibility yaxshilanadi
- **Performance:** facade additional layer — har chaqiruv bir method call extra. Mostly negligible
- **Testing:** facade test qilganda 4 ta subsystem'ni mock qilish kerak. Integration test bilan birgalikda
- **Adapter chain:** bir nechta adapter ketma-ket (`AdapterB(AdapterA(legacy))`) — debugging qiyin

### Follow-up savollar

1. **"Facade qachon Service Layer'ga aylanadi?"** — Facade business logic qo'shsa — service. Pure orchestration — facade
2. **"Adapter va Bridge farqi?"** — Adapter — mavjud interface compatibility. Bridge — abstraction va implementation'ni alohida hierarchy'da ajratish (proactive design)
3. **"`AuthFacade` SOLID buzadimi?"** — Single Responsibility — agar facade faqat orchestration qilsa (business logic yo'q), SRP'ni qoniqtiradi

</details>

---

### Savol 8: Proxy pattern — qaysi use case'lar? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Proxy — object'ga qo'shimcha behavior qo'shadi (caching, access control, logging, lazy loading) — original object'ni o'zgartirmasdan. Client object'ga to'g'ridan-to'g'ri emas, proxy orqali murojaat qiladi.

### To'liq tushuntirish

**Proxy turlari:**

1. **Caching Proxy** — query natijasini cache qiladi
2. **Lazy Proxy (Virtual)** — costly object'ni faqat kerak bo'lganda yaratadi
3. **Protection Proxy** — access control (role/permission)
4. **Remote Proxy** — remote service uchun local placeholder
5. **Logging Proxy** — chaqiruvlarni log qiladi
6. **Smart Reference** — reference counting, garbage collection

**Struktura:**

```
Subject (interface)
├── method(): T

RealSubject implements Subject

Proxy implements Subject
├── private realSubject: RealSubject
└── method(): T  → realSubject.method() + extra behavior
```

**Proxy vs Decorator:**
- **Proxy** — access control, lifecycle (kim, qachon access qila oladi)
- **Decorator** — behavior enhancement (nima ish qiladi)
- Texnik o'xshash, intent farq

**ES6 `Proxy` object:**

JavaScript native `Proxy` — handler orqali har operation'ni intercept. Pattern'dan farq — JS spec konstruksiyasi.

### Kod misol

```typescript
// === 1. Caching Proxy ===
interface UserApi {
  getUser(id: number): Promise<{ id: number; name: string }>;
  getUsers(): Promise<{ id: number; name: string }[]>;
}

class UserApiImpl implements UserApi {
  async getUser(id: number) {
    console.log(`[DB] Query user ${id}`);
    return { id, name: `User ${id}` };
  }

  async getUsers() {
    console.log("[DB] Query all users");
    return [{ id: 1, name: "Ali" }, { id: 2, name: "Vali" }];
  }
}

class CachingUserApi implements UserApi {
  private cache = new Map<string, { data: any; expires: number }>();

  constructor(
    private inner: UserApi,
    private ttlMs: number = 60_000
  ) {}

  async getUser(id: number) {
    return this.cached(`user:${id}`, () => this.inner.getUser(id));
  }

  async getUsers() {
    return this.cached("users:all", () => this.inner.getUsers());
  }

  private async cached<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const entry = this.cache.get(key);
    if (entry && entry.expires > Date.now()) {
      return entry.data;
    }
    const data = await fn();
    this.cache.set(key, { data, expires: Date.now() + this.ttlMs });
    return data;
  }
}

const api: UserApi = new CachingUserApi(new UserApiImpl(), 60_000);
await api.getUser(1); // [DB] Query user 1
await api.getUser(1); // (cache dan)


// === 2. Protection Proxy (access control) ===
interface AdminPanel {
  deleteUser(id: number): Promise<void>;
  banUser(id: number): Promise<void>;
}

class AdminPanelImpl implements AdminPanel {
  async deleteUser(id: number) { console.log(`Deleted ${id}`); }
  async banUser(id: number) { console.log(`Banned ${id}`); }
}

class ProtectedAdminPanel implements AdminPanel {
  constructor(
    private inner: AdminPanel,
    private currentUserRole: string
  ) {}

  private requireRole(required: string) {
    if (this.currentUserRole !== required) {
      throw new Error(`Forbidden: requires ${required}, got ${this.currentUserRole}`);
    }
  }

  async deleteUser(id: number) {
    this.requireRole("admin");
    return this.inner.deleteUser(id);
  }

  async banUser(id: number) {
    if (!["admin", "moderator"].includes(this.currentUserRole)) {
      throw new Error("Forbidden");
    }
    return this.inner.banUser(id);
  }
}


// === 3. Lazy (Virtual) Proxy ===
interface HeavyResource {
  process(data: string): Promise<string>;
}

class HeavyResourceImpl implements HeavyResource {
  constructor() {
    console.log("Loading heavy ML model...");
    // Simulating expensive initialization
  }
  async process(data: string) { return `processed: ${data}`; }
}

class LazyResourceProxy implements HeavyResource {
  private instance: HeavyResource | null = null;

  async process(data: string) {
    if (!this.instance) {
      this.instance = new HeavyResourceImpl(); // Lazy init
    }
    return this.instance.process(data);
  }
}

const resource: HeavyResource = new LazyResourceProxy();
// Hech qanday log — instance hali yaratilmagan
await resource.process("input"); // "Loading heavy ML model..." → "processed: input"


// === 4. Native ES6 Proxy — logging ===
function createLoggingProxy<T extends object>(target: T): T {
  return new Proxy(target, {
    get(obj, prop) {
      console.log(`[GET] ${String(prop)}`);
      return Reflect.get(obj, prop);
    },
    set(obj, prop, value) {
      console.log(`[SET] ${String(prop)} = ${value}`);
      return Reflect.set(obj, prop, value);
    },
  });
}

const user = { name: "Ali", age: 25 };
const logged = createLoggingProxy(user);
logged.name;          // [GET] name
logged.age = 26;       // [SET] age = 26
```

### Edge Cases

- **Cache invalidation:** stale data muammosi. TTL, write-through, event-based invalidation strategy'lar
- **Proxy chain:** `LoggingProxy(CachingProxy(LazyProxy(Real)))` — har layer overhead, debugging qiyin
- **`this` binding:** proxy method ichida `this` proxy. Inner subject'ga proxy'ni emas, inner'ni o'zini berish kerak ba'zi case'da
- **TypeScript native Proxy type:** `Proxy<T>` `T` ga aylanadi (transparent). Compile-time inference yaxshi
- **Memory leak (cache proxy):** cache size cheksiz → LRU yoki TTL bilan cheklash

### Follow-up savollar

1. **"Proxy va Decorator farqi?"** — Proxy: access control/lifecycle. Decorator: behavior extension. Texnik o'xshash, intent farq
2. **"ES6 `Proxy` qaysi case'da pattern'dan afzal?"** — Dynamic intercept — `get`/`set` har property uchun universal. Pattern'da har method explicit
3. **"Proxy pattern va reverse proxy farqi?"** — Reverse proxy — network/HTTP konsepsiya (Nginx). Pattern — OOP konsepsiya. Maqsad bir xil — intermediate layer

</details>

---

### Savol 9: Observer pattern — typed EventEmitter [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Observer — bir object (Subject) holati o'zgarganda, barcha kuzatuvchilar (Observer) avtomatik xabar oladi. TypeScript'da typed EventEmitter — `Events[K]` tuple/function type bilan har event uchun argument type'lar compile-time'da tekshiriladi.

### To'liq tushuntirish

**Klassik Observer:**

```
Subject
├── observers: Observer[]
├── attach(o)
├── detach(o)
└── notify()  → har observer.update() chaqiriladi

Observer (interface)
└── update(subject)
```

**Modern EventEmitter (pub/sub):**

Event name + payload. Subject `emit(event, payload)`, listener `on(event, callback)`.

**TypeScript type-safety:**

```typescript
interface AppEvents {
  userLogin: [userId: string, timestamp: Date];
  orderPlaced: [orderId: string, amount: number];
}

emitter.on("userLogin", (userId, timestamp) => { /* type-safe */ });
emitter.emit("userLogin", "user-1", new Date()); // ✅
emitter.emit("userLogin", 42, "wrong"); // ❌ type error
```

**`Events[K]` tuple pattern:**
- Key — event name (literal)
- Value — argument tuple
- `emit<K extends keyof Events>(event: K, ...args: Events[K])` — compiler tekshiradi

**Observer use case'lar:**
1. **Event-driven systems** — UI events, lifecycle hooks
2. **Reactive state** — state change → UI update
3. **Pub/Sub** — microservice communication
4. **Logging/Telemetry** — har action'ga listener

### Kod misol

```typescript
// === Type-safe EventEmitter ===
type EventMap = Record<string, unknown[]>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners = new Map<keyof Events, Set<(...args: any[]) => void>>();

  on<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);
    return () => { this.listeners.get(event)?.delete(handler); };
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    this.listeners.get(event)?.forEach((h) => h(...args));
  }

  once<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): () => void {
    const unsub = this.on(event, (...args) => {
      handler(...args);
      unsub();
    });
    return unsub;
  }

  removeAllListeners(event?: keyof Events): void {
    if (event) this.listeners.delete(event);
    else this.listeners.clear();
  }

  listenerCount<K extends keyof Events>(event: K): number {
    return this.listeners.get(event)?.size ?? 0;
  }
}


// === Usage ===
interface UserEvents {
  login: [userId: string, timestamp: Date];
  logout: [userId: string];
  purchase: [userId: string, productId: string, amount: number];
}

const emitter = new TypedEventEmitter<UserEvents>();

// Type-safe listener
const unsubLogin = emitter.on("login", (userId, timestamp) => {
  console.log(`${userId} at ${timestamp.toISOString()}`); // ✅ string, Date
});

emitter.on("purchase", (userId, productId, amount) => {
  console.log(`${userId} bought ${productId} for $${amount}`);
});

// Emit — type-safe
emitter.emit("login", "user-1", new Date()); // ✅
emitter.emit("purchase", "user-1", "laptop-x1", 999.99); // ✅

// emitter.emit("login", "user-1"); // ❌ timestamp kerak
// emitter.emit("unknownEvent"); // ❌ event yo'q


// === Cleanup ===
unsubLogin(); // listener olib tashlandi
emitter.removeAllListeners("purchase"); // event uchun barcha listener
```

### Edge Cases

- **Memory leak:** listener register qilingan, lekin cleanup chaqirilmagan → listener (closure) GC bo'lmaydi. `unsub` return — yechim
- **Async listener:** listener async bo'lsa — `emit` o'tib ketadi, javob kutilmaydi. Async event uchun alohida `emitAsync`
- **Error in listener:** bir listener throw qilsa — boshqalari ham fail bo'lishi mumkin. `try/catch` har listener uchun
- **Re-entrant emit:** listener ichida `emit` chaqirsa — recursive call stack
- **Listener order:** `Set` insertion order — register qilingan tartibda chaqiriladi. Priority kerak bo'lsa — ranked list
- **once leak:** `once` ichida wrapper saqlanadi. Listener never fires bo'lsa, leak. Cleanup mexanizmi kerak

### Follow-up savollar

1. **"Native `EventTarget` vs Class EventEmitter?"** — Native: standart, DOM bilan integratsiya, lekin type-safety zaif. Class: full TypeScript control
2. **"RxJS Observable bilan farqi?"** — Observable: backpressure, operators, hot/cold streams. EventEmitter: simple pub/sub
3. **"`Events[K]` tuple vs function signature?"** — Tuple ko'proq fleksible, lekin function signature (`(userId: string) => void`) ham ishlaydi. Tuple `...args` spread bilan natural

</details>

---

### Savol 10: Command pattern: undo/redo qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Command — operatsiyani object sifatida encapsulate qiladi: `execute()` + `undo()`. History stack'da command'lar saqlanadi — undo `pop` qilib `undo()` chaqiradi, redo `execute()` qaytadan. Text editor, drawing app, transaction'lar uchun ideal.

### To'liq tushuntirish

**Command struktura:**

```typescript
interface Command<T = void> {
  execute(): T;
  undo(): void;
}
```

**History strukturasi:**

```
[done: Command[]]  ←→  current  ←→  [undone: Command[]]

undo(): done.pop() → cmd.undo() → undone.push(cmd)
redo(): undone.pop() → cmd.execute() → done.push(cmd)
new command: done.push(cmd); undone = []  // redo stack tozalanadi
```

**Use case'lar:**
1. **Text editor** — type, delete, format → undo/redo
2. **Drawing app** — draw, erase, move
3. **Database transaction** — execute statement, rollback
4. **Macro recording** — command sequence saqlash va replay
5. **CQRS** — command (write) va query (read) ajratish

**Memento pattern bilan birga:** undo kerak bo'lganda state snapshot saqlash variant.

### Kod misol

```typescript
// === Command interface ===
interface Command<T = void> {
  execute(): T;
  undo(): void;
  description?: string;
}


// === History (Invoker) ===
class CommandHistory {
  private done: Command[] = [];
  private undone: Command[] = [];

  execute<T>(command: Command<T>): T {
    const result = command.execute();
    this.done.push(command);
    this.undone = []; // Redo stack tozalanadi
    return result;
  }

  undo(): boolean {
    const cmd = this.done.pop();
    if (!cmd) return false;
    cmd.undo();
    this.undone.push(cmd);
    return true;
  }

  redo(): boolean {
    const cmd = this.undone.pop();
    if (!cmd) return false;
    cmd.execute();
    this.done.push(cmd);
    return true;
  }

  canUndo(): boolean { return this.done.length > 0; }
  canRedo(): boolean { return this.undone.length > 0; }

  clear(): void {
    this.done = [];
    this.undone = [];
  }
}


// === Concrete commands (TODO list) ===
interface TodoItem { id: number; text: string; done: boolean; }

class TodoList {
  items: TodoItem[] = [];
  private nextId = 1;

  getNextId(): number { return this.nextId++; }
}


class AddTodoCommand implements Command<number> {
  private id?: number;
  description = "Add todo";

  constructor(private list: TodoList, private text: string) {}

  execute(): number {
    if (this.id === undefined) {
      this.id = this.list.getNextId();
    }
    this.list.items.push({ id: this.id, text: this.text, done: false });
    return this.id;
  }

  undo(): void {
    if (this.id !== undefined) {
      this.list.items = this.list.items.filter((t) => t.id !== this.id);
    }
  }
}


class CompleteTodoCommand implements Command<void> {
  private previousState?: boolean;
  description = "Complete todo";

  constructor(private list: TodoList, private id: number) {}

  execute(): void {
    const todo = this.list.items.find((t) => t.id === this.id);
    if (todo) {
      this.previousState = todo.done;
      todo.done = true;
    }
  }

  undo(): void {
    const todo = this.list.items.find((t) => t.id === this.id);
    if (todo && this.previousState !== undefined) {
      todo.done = this.previousState;
    }
  }
}


class DeleteTodoCommand implements Command<void> {
  private deletedItem?: TodoItem;
  private deletedIndex?: number;
  description = "Delete todo";

  constructor(private list: TodoList, private id: number) {}

  execute(): void {
    const index = this.list.items.findIndex((t) => t.id === this.id);
    if (index >= 0) {
      this.deletedItem = this.list.items[index];
      this.deletedIndex = index;
      this.list.items.splice(index, 1);
    }
  }

  undo(): void {
    if (this.deletedItem && this.deletedIndex !== undefined) {
      this.list.items.splice(this.deletedIndex, 0, this.deletedItem);
    }
  }
}


// === Usage ===
const list = new TodoList();
const history = new CommandHistory();

const id1 = history.execute(new AddTodoCommand(list, "Buy milk"));
const id2 = history.execute(new AddTodoCommand(list, "Walk dog"));
history.execute(new CompleteTodoCommand(list, id1));

console.log(list.items);
// [{ id: 1, text: "Buy milk", done: true }, { id: 2, text: "Walk dog", done: false }]

history.undo(); // CompleteTodo undone
console.log(list.items[0].done); // false

history.undo(); // AddTodo "Walk dog" undone
console.log(list.items.length); // 1

history.redo(); // AddTodo "Walk dog" re-executed
console.log(list.items.length); // 2

// Yangi command — undone stack tozalanadi
history.execute(new DeleteTodoCommand(list, id1));
console.log(history.canRedo()); // false
```

### Edge Cases

- **Composite command (Macro):** bir nechta command bitta atomic operation sifatida — `MacroCommand implements Command` ichida `subCommands: Command[]`
- **Memory cost:** undo state saqlash — har command instance memory egallaydi. Big data — snapshot expensive
- **Idempotency:** `redo` paytida command effect takrorlanmasligi kerak. Internal state (`this.id`) saqlash
- **External state mutation:** command tashqi state'ga ham ta'sir qilsa (DB, network) — undo qiyin yoki imkonsiz
- **History size:** cheksiz history → memory issue. LRU yoki history limit

### Follow-up savollar

1. **"Command va Event farqi?"** — Command: imperative (do this), reversible. Event: notification (something happened), past tense, immutable
2. **"CQRS bilan bog'liqligi?"** — CQRS — Command (write, side effect) va Query (read) ajratish. Command pattern — CQRS dagi Command implementation pattern
3. **"Memento vs Command farqi (undo uchun)?"** — Memento: state snapshot saqlash (har step). Command: operation va inverse operation. Memento — memory heavy lekin universal. Command — granular, lekin har operatsiya uchun inverse logic kerak

</details>

---

### Savol 11: State Machine discriminated union bilan [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

State Machine — object mumkin bo'lgan state'lari va o'tishlarini boshqaradi. TypeScript discriminated union state'ni type-safe qiladi: har state o'z data shape'iga ega, transition function compile-time'da invalid transition'larni ushlaydi.

### To'liq tushuntirish

**State Machine komponentlari:**

1. **States** — discriminated union (har biri `kind` literal bilan)
2. **Events/Actions** — state'ni o'zgartirish triggerlari
3. **Transitions** — `(state, event) → newState`
4. **Side effects** — transition vaqtidagi action (callback)

**Discriminated union state safety:**

```typescript
type OrderState =
  | { status: "draft"; items: string[] }
  | { status: "placed"; items: string[]; orderId: string }
  | { status: "paid"; items: string[]; orderId: string; paymentId: string };
```

Har state o'z data shape'iga ega. `status` discriminant — narrowing uchun.

**Transition function type-safety:**

```typescript
function placeOrder(state: Extract<OrderState, { status: "draft" }>): Extract<OrderState, { status: "placed" }> {}
```

Compiler `state.status === "draft"` ekanini kafolatlaydi. Boshqa state'dan chaqirib bo'lmaydi.

**Use case'lar:**
- **Order processing** — draft → placed → paid → shipped → delivered
- **Form wizard** — step1 → step2 → step3
- **Authentication** — unauthenticated → loggingIn → authenticated → loggingOut
- **Async data** — idle → loading → success/error
- **Connection** — disconnected → connecting → connected → reconnecting

**XState library:** advanced state machine + statecharts (hierarchical, parallel states).

### Kod misol

```typescript
// === Order state machine ===
type OrderState =
  | { status: "draft"; items: string[] }
  | { status: "placed"; items: string[]; orderId: string; placedAt: Date }
  | { status: "paid"; items: string[]; orderId: string; placedAt: Date; paymentId: string }
  | { status: "shipped"; items: string[]; orderId: string; paymentId: string; trackingNumber: string }
  | { status: "delivered"; items: string[]; orderId: string; deliveredAt: Date }
  | { status: "cancelled"; reason: string };


// === Transition functions — type-safe ===
function placeOrder(
  state: Extract<OrderState, { status: "draft" }>
): Extract<OrderState, { status: "placed" }> {
  if (state.items.length === 0) {
    throw new Error("Cannot place empty order");
  }
  return {
    status: "placed",
    items: state.items,
    orderId: crypto.randomUUID(),
    placedAt: new Date(),
  };
}

function payOrder(
  state: Extract<OrderState, { status: "placed" }>,
  paymentId: string
): Extract<OrderState, { status: "paid" }> {
  return { ...state, status: "paid", paymentId };
}

function shipOrder(
  state: Extract<OrderState, { status: "paid" }>,
  trackingNumber: string
): Extract<OrderState, { status: "shipped" }> {
  return {
    status: "shipped",
    items: state.items,
    orderId: state.orderId,
    paymentId: state.paymentId,
    trackingNumber,
  };
}

function deliverOrder(
  state: Extract<OrderState, { status: "shipped" }>
): Extract<OrderState, { status: "delivered" }> {
  return {
    status: "delivered",
    items: state.items,
    orderId: state.orderId,
    deliveredAt: new Date(),
  };
}

function cancelOrder(
  state: Extract<OrderState, { status: "draft" } | { status: "placed" }>,
  reason: string
): Extract<OrderState, { status: "cancelled" }> {
  return { status: "cancelled", reason };
}


// === Exhaustive state handling ===
function renderOrder(order: OrderState): string {
  switch (order.status) {
    case "draft":
      return `Draft (${order.items.length} items)`;
    case "placed":
      return `Placed: ${order.orderId}`;
    case "paid":
      return `Paid: ${order.orderId}, payment ${order.paymentId}`;
    case "shipped":
      return `Shipped: tracking ${order.trackingNumber}`;
    case "delivered":
      return `Delivered at ${order.deliveredAt.toISOString()}`;
    case "cancelled":
      return `Cancelled: ${order.reason}`;
    default: {
      const _exhaustive: never = order;
      return _exhaustive;
    }
  }
}


// === Usage ===
let order: OrderState = { status: "draft", items: ["Laptop", "Mouse"] };
console.log(renderOrder(order));
// "Draft (2 items)"

// Compile-time check: faqat draft state qabul qilinadi
order = placeOrder(order as Extract<OrderState, { status: "draft" }>);
console.log(renderOrder(order));
// "Placed: ..."

order = payOrder(order as Extract<OrderState, { status: "placed" }>, "pay-123");
console.log(renderOrder(order));
// "Paid: ..., payment pay-123"

// cancelOrder(order, "reason"); // ❌ — paid state dan cancel qilib bo'lmaydi


// === State Machine class (centralized) ===
type StateMachineConfig<S, E> = {
  initial: S;
  transitions: Map<string, (state: S, event: E) => S>;
};

class OrderStateMachine {
  private state: OrderState;
  private listeners: Set<(state: OrderState) => void> = new Set();

  constructor(initial: OrderState) {
    this.state = initial;
  }

  getCurrentState(): Readonly<OrderState> { return this.state; }

  place(): boolean {
    if (this.state.status !== "draft") return false;
    this.transition(placeOrder(this.state));
    return true;
  }

  pay(paymentId: string): boolean {
    if (this.state.status !== "placed") return false;
    this.transition(payOrder(this.state, paymentId));
    return true;
  }

  ship(trackingNumber: string): boolean {
    if (this.state.status !== "paid") return false;
    this.transition(shipOrder(this.state, trackingNumber));
    return true;
  }

  cancel(reason: string): boolean {
    if (this.state.status !== "draft" && this.state.status !== "placed") {
      return false;
    }
    this.transition(cancelOrder(this.state, reason));
    return true;
  }

  subscribe(listener: (state: OrderState) => void): () => void {
    this.listeners.add(listener);
    return () => { this.listeners.delete(listener); };
  }

  private transition(newState: OrderState): void {
    this.state = newState;
    this.listeners.forEach((l) => l(newState));
  }
}

const machine = new OrderStateMachine({ status: "draft", items: ["Book"] });
machine.subscribe((state) => console.log(renderOrder(state)));

machine.place();          // "Placed: ..."
machine.pay("pay-456");   // "Paid: ..."
machine.ship("track-789"); // "Shipped: ..."
```

### Edge Cases

- **Exhaustive check:** `never` type — kompilator yangi state qo'shilganda `default` da xato beradi (forgot to handle)
- **State sharing:** state object reference — mutation xavfli. `Readonly<State>` yoki immutable update
- **Side effects in transitions:** transition function pure bo'lishi afzal. Side effect (DB write) listener orqali
- **Parallel/hierarchical states:** XState statecharts. Custom discriminated union — flat states uchun
- **Async transitions:** transition `Promise<NewState>` — intermediate `loading` state kerak
- **Replay/time travel:** state log saqlash — debugging va undo

### Follow-up savollar

1. **"XState bilan farqi?"** — XState: full state machine library — guards, actions, services, visualization. Discriminated union: lightweight, manual, type-safe core
2. **"`never` type exhaustive check'da qanday ishlaydi?"** — TS narrowing — switch barcha case'ni qoplagandan keyin remaining type `never`. Yangi state qo'shilsa, narrowing fail → compile error
3. **"Side effect transition vaqtida — qachon va qaerda?"** — Subscribe listener orqali (state machine class). Yoki XState `entry`/`exit` actions

<details>
<summary><strong>Deep Dive</strong></summary>

**Discriminated union narrowing — TypeScript internal:**

Compiler `switch (state.status)` ni ko'rganda, har `case` da `state` ning type'ni narrow qiladi. `case "draft"` ichida `state: Extract<OrderState, { status: "draft" }>` — `items` mavjud, `orderId` yo'q. Bu Control Flow Analysis (CFA) algoritmi orqali — TS 2.0'dan beri mavjud.

**`never` exhaustive check mexanizmi:**

```typescript
default: {
  const _exhaustive: never = order;
  return _exhaustive;
}
```

Bu yerda `_exhaustive: never` — compiler `order` ning type'i `never` ekanini tekshiradi. Agar barcha case qoplangan bo'lsa, `default` ga yetib kelganda `order` type'i `never` ga narrowed (mantiqiy: barcha boshqa state'lar already returned). Yangi state `cancelled` qo'shilsa, `default` da `order: { status: "cancelled"; reason: string }` — `never` ga assign qilib bo'lmaydi → compile error.

**Spec reference:** TypeScript handbook — Narrowing → Discriminated Unions, "exhaustiveness checking with the `never` type".

**State machine memory layout:**

Har transition yangi object qaytaradi (`return { status: "placed", ... }`) — immutable update pattern. Eski state object — garbage collected (no references). Bu `Object.freeze` bilan birga immutable state guarantee beradi.

**Performance — V8 hidden class optimization:**

V8 har object shape uchun hidden class yaratadi. Discriminated union — har variant boshqa shape (draft vs placed) → transition'da kelgan object hidden class'i o'zgaradi. State o'qiydigan kod bir nechta shape ko'rsa, property access polymorphic bo'ladi — monomorphic inline cache emas, JIT optimization cheklangan. Lekin state machine — domain logic, hot path emas. Performance ta'siri amalda sezilmaydi.

**Statecharts (Harel statecharts) vs flat state machine:**

Flat (discriminated union) — har state independent. Statecharts (XState) — hierarchical (parent-child states), parallel regions, history states. Misol: `editing.draft` `editing.validating` — har biri `editing` parent'i ichida. Discriminated union — flat. Hierarchical kerak bo'lsa — XState yoki manual nested unions.

**Async state transitions:**

```typescript
type AsyncState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: Error };
```

`loading` intermediate state — UI loading indicator ko'rsatadi. Transition: `idle` → `loading` → `success`/`error`. Race condition: ikki `fetch()` bir vaqtda — keyingisi oldingisini bekor qilishi kerak (`AbortController`).

</details>

</details>

---

### Savol 12: Result/Either pattern — exception o'rniga [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Result/Either — discriminated union `Ok<T> | Err<E>`. Exception o'rniga — compiler ikkala holatni handle qilishni majburlaydi. Functional error handling — Rust, Haskell, Scala'da standart pattern.

### To'liq tushuntirish

**Exception muammolari:**
1. **Hidden control flow** — function signature exception'ni ko'rsatmaydi
2. **Forgotten handling** — `try/catch` ni unutish mumkin
3. **Stack unwinding cost** — performance overhead
4. **Untyped:** `catch (e)` da `e: unknown`

**Result/Either afzalliklari:**
1. **Type-safe** — signature'da `Result<T, E>` ko'rinadi
2. **Compile-time enforcement** — kompilator handle qilishni majburlaydi
3. **Composable** — `map`, `flatMap`, `chain`
4. **No hidden control flow** — explicit success/failure path

**Naming convention:**
- **Result** — Rust style (`Ok<T> | Err<E>`)
- **Either** — Functional style (`Left<E> | Right<T>`)

**Strukturasi:**

```typescript
type Result<T, E> =
  | { readonly _tag: "Ok"; readonly value: T }
  | { readonly _tag: "Err"; readonly error: E };
```

**Utility funksiyalar:**
- `ok(value)` — Ok wrapper
- `err(error)` — Err wrapper
- `isOk(r)` — type guard
- `map(r, fn)` — Ok'da fn apply
- `flatMap(r, fn)` — Ok'da fn (Result qaytaruvchi) apply, flatten
- `unwrapOr(r, default)` — Ok bo'lsa value, Err bo'lsa default

**Railway-oriented programming:**

Functions Result qaytaradi → `flatMap` orqali chain. Birinchi `Err` butun chain'ni to'xtatadi.

### Kod misol

```typescript
// === Result type va constructors ===
type Ok<T> = { readonly _tag: "Ok"; readonly value: T };
type Err<E> = { readonly _tag: "Err"; readonly error: E };
type Result<T, E> = Ok<T> | Err<E>;

function ok<T>(value: T): Ok<T> { return { _tag: "Ok", value }; }
function err<E>(error: E): Err<E> { return { _tag: "Err", error }; }

function isOk<T, E>(r: Result<T, E>): r is Ok<T> { return r._tag === "Ok"; }
function isErr<T, E>(r: Result<T, E>): r is Err<E> { return r._tag === "Err"; }


// === Utility funksiyalar ===
function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return isOk(r) ? ok(fn(r.value)) : r;
}

function flatMap<T, U, E>(
  r: Result<T, E>,
  fn: (v: T) => Result<U, E>
): Result<U, E> {
  return isOk(r) ? fn(r.value) : r;
}

function mapErr<T, E, F>(r: Result<T, E>, fn: (e: E) => F): Result<T, F> {
  return isErr(r) ? err(fn(r.error)) : r;
}

function unwrapOr<T, E>(r: Result<T, E>, defaultValue: T): T {
  return isOk(r) ? r.value : defaultValue;
}

function unwrap<T, E>(r: Result<T, E>): T {
  if (isErr(r)) throw new Error(`unwrap Err: ${String(r.error)}`);
  return r.value;
}


// === Real-world example: User registration ===
interface User { id: number; email: string; age: number; name: string; }

type AppError =
  | { type: "ValidationError"; field: string; message: string }
  | { type: "DuplicateEmail"; email: string }
  | { type: "DatabaseError"; details: string };

function validateEmail(email: string): Result<string, AppError> {
  if (!email.includes("@")) {
    return err({ type: "ValidationError", field: "email", message: "Invalid format" });
  }
  return ok(email);
}

function validateAge(age: number): Result<number, AppError> {
  if (age < 0 || age > 150) {
    return err({ type: "ValidationError", field: "age", message: "Out of range" });
  }
  if (age < 18) {
    return err({ type: "ValidationError", field: "age", message: "Must be 18+" });
  }
  return ok(age);
}

async function checkEmailNotTaken(email: string): Promise<Result<string, AppError>> {
  const taken = false; // simulated DB check
  if (taken) {
    return err({ type: "DuplicateEmail", email });
  }
  return ok(email);
}

async function saveUser(data: Omit<User, "id">): Promise<Result<User, AppError>> {
  try {
    return ok({ id: 1, ...data });
  } catch (e) {
    return err({
      type: "DatabaseError",
      details: e instanceof Error ? e.message : String(e),
    });
  }
}


// === Railway-oriented composition ===
async function registerUser(
  input: { email: string; age: number; name: string }
): Promise<Result<User, AppError>> {
  const emailResult = validateEmail(input.email);
  if (isErr(emailResult)) return emailResult;

  const ageResult = validateAge(input.age);
  if (isErr(ageResult)) return ageResult;

  const availableResult = await checkEmailNotTaken(emailResult.value);
  if (isErr(availableResult)) return availableResult;

  return saveUser({
    email: emailResult.value,
    age: ageResult.value,
    name: input.name,
  });
}


// === Usage ===
const result = await registerUser({
  email: "ali@example.com",
  age: 25,
  name: "Ali",
});

if (isOk(result)) {
  console.log(`User created: ${result.value.id}`);
} else {
  switch (result.error.type) {
    case "ValidationError":
      console.log(`Validation: ${result.error.field} — ${result.error.message}`);
      break;
    case "DuplicateEmail":
      console.log(`Email ${result.error.email} taken`);
      break;
    case "DatabaseError":
      console.log(`DB: ${result.error.details}`);
      break;
  }
}
```

### Edge Cases

- **Mix with exception:** Result code ichida `throw` qilsa — caught emas, raw exception propagates. Result va throw'ni aralashtirish ikkala error path'ni bir vaqtda saqlaydi — bitta yondashuvni tanlash kerak
- **Nested Result:** `Result<Result<T, E1>, E2>` — `flatMap` bilan flatten qilish kerak
- **Async Result:** `Promise<Result<T, E>>` — `async/await` + `if (isErr(r)) return r` pattern
- **Performance:** Result object — `try/catch` dan tezroq (stack unwinding yo'q)
- **Compatibility:** Library exception yoritadi — `try { ... } catch (e) { return err(e) }` adapter
- **Error type union:** ko'p error type — discriminated union (`type AppError = ...`)

### Follow-up savollar

1. **"Rust `Result` bilan TypeScript `Result` farqi?"** — Semantik bir xil. Rust'da `?` operator — TypeScript'da bu sintaxis yo'q (`if (isErr) return` manual)
2. **"Effect/fp-ts'da nima farq?"** — Effect/fp-ts: Result + IO + dependency injection + async — full monad ecosystem. Result alone: simple, pragmatic
3. **"Exception qachon afzal?"** — Truly exceptional case'lar (out-of-memory, programming bug). Business error — Result. Boundary (input validation) — Result; deep internal — exception

</details>

---

### Savol 13: Repository pattern + generics? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Repository — data access logic'ni business logic'dan ajratadi. TypeScript generic `Repository<T extends Entity>` har entity uchun type-safe CRUD interface beradi. `Omit` bilan auto-managed field (id, createdAt) compile-time'da exclude qilinadi.

### To'liq tushuntirish

**Repository'ning maqsadlari:**
1. **Persistence abstraction** — DB/cache/API detail'ni yashirish
2. **Testability** — InMemoryRepository bilan unit test
3. **Single source of truth** — har entity uchun bir repository
4. **Type safety** — Entity shape compile-time'da

**Generic constraint:**

```typescript
interface Entity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T>;
  update(id: string, data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>): Promise<T>;
  delete(id: string): Promise<boolean>;
}
```

**`Omit<T, K>` significance:**
- `create` — `id`, `createdAt`, `updatedAt` repository tomonidan generate qilinadi. Caller berishi kerak emas
- `update` — `id` URL/method'da, `createdAt` immutable. Yangilanadigan field'lar `Partial<Omit<...>>`

**Repository va DDD (Domain-Driven Design):**

DDD'da Repository — domain layer va infrastructure layer ko'prigi. Domain object'lar Repository orqali persistence'dan ajratiladi.

### Kod misol

```typescript
// === Base Entity va Repository ===
interface Entity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T>;
  update(
    id: string,
    data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>
  ): Promise<T>;
  delete(id: string): Promise<boolean>;
  count(filter?: Partial<T>): Promise<number>;
}


// === InMemory Repository — test/development ===
class InMemoryRepository<T extends Entity> implements Repository<T> {
  protected items = new Map<string, T>();

  async findById(id: string): Promise<T | null> {
    return this.items.get(id) ?? null;
  }

  async findAll(filter?: Partial<T>): Promise<T[]> {
    let results = Array.from(this.items.values());
    if (filter) {
      results = results.filter((item) =>
        Object.entries(filter).every(
          ([key, value]) => item[key as keyof T] === value
        )
      );
    }
    return results;
  }

  async create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T> {
    const now = new Date();
    const entity = {
      ...data,
      id: crypto.randomUUID(),
      createdAt: now,
      updatedAt: now,
    } as T;
    this.items.set(entity.id, entity);
    return entity;
  }

  async update(
    id: string,
    data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>
  ): Promise<T> {
    const existing = this.items.get(id);
    if (!existing) throw new Error(`Entity not found: ${id}`);
    const updated = {
      ...existing,
      ...data,
      updatedAt: new Date(),
    } as T;
    this.items.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.items.delete(id);
  }

  async count(filter?: Partial<T>): Promise<number> {
    return (await this.findAll(filter)).length;
  }
}


// === Concrete entity + repository ===
interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "user";
  age: number;
}

const userRepo: Repository<User> = new InMemoryRepository<User>();

// Create — id, createdAt, updatedAt avtomatik
const user = await userRepo.create({
  name: "Ali",
  email: "ali@example.com",
  role: "admin",
  age: 25,
});
console.log(user.id);         // uuid
console.log(user.createdAt);   // Date

// userRepo.create({ name: "Ali" }); // ❌ — email, role, age kerak
// userRepo.create({ name: "Ali", id: "x" }); // ❌ — id Omit qilingan

// Find by filter
const admins = await userRepo.findAll({ role: "admin" });

// Update
const updated = await userRepo.update(user.id, { age: 26 });

// Delete
await userRepo.delete(user.id);


// === Specialized repository — domain-specific method ===
class UserRepository extends InMemoryRepository<User> {
  async findByEmail(email: string): Promise<User | null> {
    const users = await this.findAll();
    return users.find((u) => u.email === email) ?? null;
  }

  async findAdmins(): Promise<User[]> {
    return this.findAll({ role: "admin" });
  }
}


// === Production: Postgres adapter ===
interface PgClient { query: (sql: string, params?: unknown[]) => Promise<{ rows: any[] }>; }

class PostgresUserRepository implements Repository<User> {
  constructor(private db: PgClient) {}

  async findById(id: string): Promise<User | null> {
    const { rows } = await this.db.query(
      "SELECT * FROM users WHERE id = $1",
      [id]
    );
    return rows[0] ?? null;
  }

  async findAll(filter?: Partial<User>): Promise<User[]> {
    if (!filter || Object.keys(filter).length === 0) {
      const { rows } = await this.db.query("SELECT * FROM users");
      return rows;
    }
    const conditions = Object.keys(filter)
      .map((k, i) => `${k} = $${i + 1}`)
      .join(" AND ");
    const { rows } = await this.db.query(
      `SELECT * FROM users WHERE ${conditions}`,
      Object.values(filter)
    );
    return rows;
  }

  async create(data: Omit<User, "id" | "createdAt" | "updatedAt">): Promise<User> {
    const id = crypto.randomUUID();
    const now = new Date();
    const { rows } = await this.db.query(
      "INSERT INTO users (id, name, email, role, age, created_at, updated_at) VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *",
      [id, data.name, data.email, data.role, data.age, now, now]
    );
    return rows[0];
  }

  async update(
    id: string,
    data: Partial<Omit<User, "id" | "createdAt" | "updatedAt">>
  ): Promise<User> {
    const updates = Object.keys(data)
      .map((k, i) => `${k} = $${i + 1}`)
      .join(", ");
    const { rows } = await this.db.query(
      `UPDATE users SET ${updates}, updated_at = NOW() WHERE id = $${Object.keys(data).length + 1} RETURNING *`,
      [...Object.values(data), id]
    );
    return rows[0];
  }

  async delete(id: string): Promise<boolean> {
    const { rows } = await this.db.query(
      "DELETE FROM users WHERE id = $1 RETURNING id",
      [id]
    );
    return rows.length > 0;
  }

  async count(filter?: Partial<User>): Promise<number> {
    return (await this.findAll(filter)).length;
  }
}
```

### Edge Cases

- **Composite key:** `Entity { id: string }` — single key assumes. Composite key kerak bo'lsa — generic `Entity<TKey>`
- **Soft delete:** `delete` actually `deletedAt` field set. `findAll` deleted'ni exclude qilishi kerak
- **Pagination:** `findAll({ offset, limit })` parameter — `Partial<T>` bilan mix. Yaxshi yondashuv: alohida `FindOptions`
- **Transaction:** repository ko'p method bir transaction'da — `repository.inTransaction(async (repo) => ...)` pattern
- **Specification pattern:** complex filter — `Partial<T>` cheklangan. `findAll(spec: Specification<T>)` — kompozit query
- **Generic constraint chain:** `extends Entity` — repository har entity'ga bir xil base field talab qiladi
- **Aggregation:** repository pure CRUD. Aggregation (sum, avg) — alternativa: query object yoki domain service

### Follow-up savollar

1. **"Repository va Active Record farqi?"** — Repository: external collection, entity'ga method yo'q. Active Record: entity o'zining `save()`, `delete()` method'lariga ega
2. **"DDD'da Repository qaerda joylashadi?"** — Domain layer'da interface, infrastructure layer'da implementation. Dependency Inversion
3. **"ORM va Repository overlap qiladimi?"** — Yes — Prisma/TypeORM'da `prisma.user.findMany()` repository pattern instantiation. Custom repository — domain logic + abstraction

</details>

---

### Savol 14: Middleware pipeline — onion model nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Middleware pipeline — funksiyalar ichma-ich (onion) ishlaydi. `next()` chaqirilgandan oldingi kod "kirish" qatlami, keyin "chiqish" qatlami. Express, Koa, Connect framework'lari shu pattern asosida. Chain of Responsibility pattern'ning composable versiyasi.

### To'liq tushuntirish

**Onion model:**

```
Request →  A before  → B before  → C handler → B after → A after → Response
            \_________ next() ___________/  \_____ after _____/
```

Har middleware ikki fazaga ega:
1. **Before next()** — request fazasi (auth check, logging, request transform)
2. **After next()** — response fazasi (response time, error handling, response transform)

**Implementation:**

```typescript
type Middleware = (ctx: Context, next: () => Promise<void>) => Promise<void>;

class Pipeline {
  middlewares: Middleware[] = [];

  async execute(ctx: Context) {
    let i = 0;
    const next = async () => {
      if (i < this.middlewares.length) {
        await this.middlewares[i++](ctx, next);
      }
    };
    await next();
  }
}
```

**`next()` semantikasi:**
- Chaqirilsa — keyingi middleware ishlaydi
- Chaqirilmasa — zanjir to'xtaydi (early return)
- Ikki marta chaqirilsa — bug (har middleware bir marta)

**Use case'lar:**
1. **HTTP framework** — Express, Koa, Hono, NestJS interceptors
2. **Redux middleware** — action interception
3. **GraphQL middleware** — resolver wrapping
4. **Build pipeline** — Webpack loaders, Babel plugins
5. **Logging chain** — log enrichment

### Kod misol

```typescript
// === Synchronous pipeline (onion order demonstration) ===
type SyncMiddleware = (ctx: { value: number }, next: () => void) => void;

class SyncPipeline {
  private stack: SyncMiddleware[] = [];

  use(mw: SyncMiddleware): this { this.stack.push(mw); return this; }

  execute(ctx: { value: number }): void {
    let i = 0;
    const next = () => {
      if (i < this.stack.length) {
        this.stack[i++](ctx, next);
      }
    };
    next();
  }
}

const pipeline = new SyncPipeline()
  .use((ctx, next) => {
    console.log("A before:", ctx.value); // 1 — kirish
    ctx.value *= 2;
    next();
    console.log("A after:", ctx.value);  // 5 — chiqish
  })
  .use((ctx, next) => {
    console.log("B before:", ctx.value); // 2
    ctx.value += 10;
    next();
    console.log("B after:", ctx.value);  // 4
  })
  .use((ctx) => {
    console.log("C handler:", ctx.value); // 3 — markaz
  });

pipeline.execute({ value: 5 });
// A before: 5
// B before: 10
// C handler: 20
// B after: 20
// A after: 20


// === Async HTTP middleware pipeline ===
interface Context {
  request: { path: string; method: string; headers: Record<string, string> };
  response: { status: number; body?: unknown };
  state: Record<string, unknown>;
}

type Middleware = (ctx: Context, next: () => Promise<void>) => Promise<void>;

class AsyncPipeline {
  private stack: Middleware[] = [];

  use(mw: Middleware): this { this.stack.push(mw); return this; }

  async execute(ctx: Context): Promise<Context> {
    let i = 0;
    const next = async (): Promise<void> => {
      if (i < this.stack.length) {
        await this.stack[i++](ctx, next);
      }
    };
    await next();
    return ctx;
  }
}


// === Production-ready middleware lar ===

// 1. Logger
const logger: Middleware = async (ctx, next) => {
  const start = Date.now();
  console.log(`-> ${ctx.request.method} ${ctx.request.path}`);
  try {
    await next();
    console.log(`<- ${ctx.response.status} (${Date.now() - start}ms)`);
  } catch (e) {
    console.error(`<- 500 ERROR (${Date.now() - start}ms)`, e);
    throw e;
  }
};

// 2. Auth
const auth: Middleware = async (ctx, next) => {
  const token = ctx.request.headers["authorization"];
  if (!token) {
    ctx.response = { status: 401, body: { error: "Unauthorized" } };
    return; // next() chaqirilmaydi — zanjir to'xtaydi
  }
  ctx.state["userId"] = "user-from-token";
  await next();
};

// 3. Error handler
const errorHandler: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (e) {
    ctx.response = {
      status: 500,
      body: { error: e instanceof Error ? e.message : "Unknown error" },
    };
  }
};

// 4. Body parser
const bodyParser: Middleware = async (ctx, next) => {
  // Parse request body (simulated)
  ctx.state["body"] = { name: "Ali" };
  await next();
};

// 5. Route handler
const handler: Middleware = async (ctx) => {
  ctx.response = {
    status: 200,
    body: { userId: ctx.state["userId"], data: ctx.state["body"] },
  };
};


// === Compose va run ===
const app = new AsyncPipeline()
  .use(errorHandler)    // Outermost — catches errors
  .use(logger)
  .use(auth)            // Auth — return early bo'lsa, handler ishlamaydi
  .use(bodyParser)
  .use(handler);        // Innermost

const result = await app.execute({
  request: {
    path: "/api/users",
    method: "POST",
    headers: { authorization: "Bearer xxx" },
  },
  response: { status: 200 },
  state: {},
});

console.log(result.response);
// { status: 200, body: { userId: "user-from-token", data: { name: "Ali" } } }
```

### Edge Cases

- **`next()` not called:** zanjir to'xtaydi — auth fail case'da to'g'ri. Lekin accidental skip — silent bug
- **`next()` called twice:** undefined behavior — har middleware bir marta chaqirilishi shart
- **Error propagation:** `await next()` da exception throw bo'lsa — chaqiruvchi middleware'da `try/catch` bo'lmasa, propagates outer
- **Concurrent execution:** har middleware ketma-ket (await). Parallel kerak bo'lsa — `Promise.all` ichida
- **Mutation vs immutability:** ctx mutate qilinadi — shared mutable state. Parallel request'lar uchun har request o'z ctx
- **Timeout:** middleware o'rab `Promise.race([next(), timeout])` — slow middleware'ni cut qilish
- **Order matters:** errorHandler outermost (boshqalar throw qilsa, catch qiladi). Auth bodyParser'dan oldin (unauthorized request body parse qilinmasin)

### Follow-up savollar

1. **"Express vs Koa middleware farqi?"** — Express: callback-based, error handling alohida. Koa: async/await native, error handling natural `try/catch`
2. **"NestJS interceptor middleware bilan farqi?"** — Interceptor: Rx Observable bilan, response transformation. Middleware: lower level
3. **"Chain of Responsibility pattern bilan farqi?"** — Middleware pipeline — chain of responsibility'ning composable variant. Har handler explicit `next()` chaqiradi, classical chain'da link manual

</details>

---

## Output savollar

### Savol 15: Decorator composition + factory output [Middle+]

**Savol:** Output'ni ayting:

```typescript
function pricingStrategy(name: string) {
  console.log(`factory: ${name}`);
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    console.log(`apply: ${name}`);
    return function (this: any, ...args: any[]) {
      console.log(`run: ${name}`);
      return originalMethod.call(this, ...args);
    };
  };
}

class Order {
  @pricingStrategy("discount")
  @pricingStrategy("tax")
  calculate() { return 100; }
}

console.log("--- runtime ---");
new Order().calculate();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
factory: discount
factory: tax
apply: tax
apply: discount
--- runtime ---
run: discount
run: tax
```

### To'liq tushuntirish

**Decorator composition uchun ikki bosqich:**

1. **Class definition phase:**
   - **Factory evaluate** — yuqoridan pastga: `pricingStrategy("discount")` → `pricingStrategy("tax")`
   - **Decorator apply** — pastdan yuqoriga: `tax` (yaqinroq) → `discount` (tashqaridagi)

2. **Runtime phase (`new Order().calculate()`):**
   - Wrapped funksiya outer → inner ketma-ketligida ishlaydi
   - `discount` wrapper birinchi (eng tashqaridagi), `tax` ichida
   - Inside-out call: `discount` log → `tax` log → original method

**Mantiqiy composition:**

`@pricingStrategy("discount") @pricingStrategy("tax") method`
== `discount(tax(method))`

Call order: discount wrapper → tax wrapper → original
Wrapper enter order: discount → tax (outside in)

### Kod misol

```typescript
// Anatomic understanding
function visualize(name: string) {
  console.log(`factory(${name})`);
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    console.log(`apply(${name})`);
    return function (this: any, ...args: any[]) {
      console.log(`enter(${name})`);
      const result = originalMethod.call(this, ...args);
      console.log(`exit(${name})`);
      return result;
    };
  };
}

class Service {
  @visualize("outer")
  @visualize("inner")
  method() { console.log("original"); }
}

console.log("--- runtime ---");
new Service().method();

// factory(outer)         <- class definition
// factory(inner)
// apply(inner)
// apply(outer)
// --- runtime ---
// enter(outer)           <- runtime: onion model
// enter(inner)
// original
// exit(inner)
// exit(outer)
```

### Edge Cases

- **Factory side effect:** har class definition'da factory chaqiriladi (har faylda class evaluate bo'lganda)
- **Apply order:** matematik composition tartibida — innermost qabul qilingan original method
- **Runtime call:** wrapped funksiya outer'dan boshlanadi — onion model (kirish-chiqish)
- **Same decorator name:** ikki marta `@same @same method()` — wrap ikki marta, har enter/exit takrorlanadi
- **Async method:** wrapper o'zi sync, lekin `await originalMethod.call(this)` async chain'ni saqlaydi

### Follow-up savollar

1. **"Nima uchun apply order teskari?"** — Function composition semantikasi: `a(b(c(x)))` — birinchi `c` apply, keyin `b`, keyin `a`. Outermost decorator eng oxirgi qo'llaniladi
2. **"Runtime enter/exit order qanday foydali?"** — Logging (before/after), timing (start/end), error handling — outer wrapper inner xatosini tutadi

</details>

---

### Savol 16: Strategy + `this` context output [Middle+]

**Savol:** Output'ni ayting:

```typescript
interface PricingStrategy {
  calculate(base: number): number;
}

class BulkPricing implements PricingStrategy {
  constructor(private discount: number) {}
  calculate(base: number) { return base * (1 - this.discount); }
}

class Calculator {
  constructor(private strategy: PricingStrategy) {}
  getTotal(price: number) { return this.strategy.calculate(price); }
}

const calc = new Calculator(new BulkPricing(0.2));
console.log(calc.getTotal(100));

const fn = calc.getTotal;
try {
  console.log(fn(100));
} catch (e) {
  console.log("Error:", (e as Error).message);
}

const bound = calc.getTotal.bind(calc);
console.log(bound(100));
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
80
Error: Cannot read properties of undefined (reading 'strategy')
80
```

### To'liq tushuntirish

**Birinchi `calc.getTotal(100)`:**

Method chaqirilganda `this` — `calc` instance. `this.strategy.calculate(100)` = `BulkPricing(0.2).calculate(100)` = `100 * 0.8` = **80**.

**Ikkinchi `fn(100)`:**

`const fn = calc.getTotal` — method'ni reference qilib oldik. Lekin `this` binding yo'q (free function chaqiruvi).

Class body har doim strict mode — `getTotal` ichida `this` `undefined`. `this.strategy` o'qishning o'zi (`.calculate` ga yetmasdan) `TypeError` beradi:

```typescript
fn(100)
// → this = undefined (class body — strict mode)
// → this.strategy  → TypeError: Cannot read properties of undefined (reading 'strategy')
```

**Uchinchi `bound(100)`:**

`calc.getTotal.bind(calc)` — yangi function yaratiladi, `this` doim `calc`. Method to'g'ri ishlaydi → **80**.

### Kod misol

```typescript
// === `this` muammosi va yechimlar ===

class Calculator {
  constructor(private strategy: PricingStrategy) {}

  // 1. Oddiy method — this dinamik
  getTotal(price: number) {
    return this.strategy.calculate(price);
  }

  // 2. Arrow method (auto-bind) — this lexical
  getTotalBound = (price: number) => {
    return this.strategy.calculate(price);
  };
}

const calc = new Calculator(new BulkPricing(0.2));

// Oddiy method
const fn1 = calc.getTotal;
// fn1(100); // ❌ TypeError

// Arrow method — auto-bound
const fn2 = calc.getTotalBound;
console.log(fn2(100)); // 80 ✅

// Manual bind
const fn3 = calc.getTotal.bind(calc);
console.log(fn3(100)); // 80 ✅

// Inline
console.log(calc.getTotal.bind(calc)(100)); // 80


// === React event handler pattern ===
class Button {
  constructor(private label: string) {}

  // Class field arrow — auto-bound (har instance da yangi function)
  onClick = () => {
    console.log(`Clicked: ${this.label}`);
  };
}

const btn = new Button("Submit");
const handler = btn.onClick;
handler(); // "Clicked: Submit" ✅
```

### Edge Cases

- **Strict mode:** `this = undefined`. Non-strict (sloppy) mode: `this = globalThis` — silent bug ehtimoli
- **Arrow method memory:** har instance uchun yangi arrow function — class va arrow ko'p instance bo'lganda memory cost
- **`super` arrow'da:** `super` lexical — arrow ichida `super` parent method'ga ishora qiladi. Lekin arrow method override qilib bo'lmaydi (instance property, prototype emas)
- **Decorator `@bound` auto-bind:** `addInitializer` bilan har instance'da method'ni bind qilish
- **`.call(otherInstance)`:** oddiy method'da ishlaydi, arrow'da ishlamaydi (`this` immutable)

### Follow-up savollar

1. **"Class field arrow va prototype method qaysi yaxshiroq?"** — Prototype: memory efficient (single function). Arrow: auto-bind, lekin har instance uchun yangi. Frequent event handler — arrow afzal
2. **"React class component'da bu muammo qanday hal qilingan?"** — Class field arrow (`onClick = () => {}`) yoki constructor'da `this.onClick = this.onClick.bind(this)`. Hook orqali functional component bilan muammo yo'q

</details>

---

### Savol 17: Middleware execution order [Middle+]

**Savol:** Output'ni ayting:

```typescript
type Middleware = (ctx: { value: number }, next: () => Promise<void>) => Promise<void>;

class Pipeline {
  private stack: Middleware[] = [];
  use(mw: Middleware) { this.stack.push(mw); return this; }
  async execute(ctx: { value: number }) {
    let i = 0;
    const next = async () => {
      if (i < this.stack.length) await this.stack[i++](ctx, next);
    };
    await next();
  }
}

const pipeline = new Pipeline()
  .use(async (ctx, next) => {
    console.log("A in", ctx.value);
    ctx.value *= 2;
    await next();
    ctx.value -= 1;
    console.log("A out", ctx.value);
  })
  .use(async (ctx, next) => {
    console.log("B in", ctx.value);
    if (ctx.value > 10) {
      ctx.value = 0;
      return; // next() chaqirilmaydi
    }
    await next();
    console.log("B out", ctx.value);
  })
  .use(async (ctx) => {
    console.log("C", ctx.value);
    ctx.value += 100;
  });

await pipeline.execute({ value: 3 });
await pipeline.execute({ value: 10 });
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
A in 3
B in 6
C 6
B out 106
A out 105
A in 10
B in 20
A out -1
```

### To'liq tushuntirish

**Birinchi execute `{ value: 3 }`:**

1. **A in 3** — A middleware boshlanadi
2. A: `value *= 2` → 6
3. A: `next()` → B
4. **B in 6** — B boshlanadi
5. B: 6 > 10 — yo'q, davom etadi
6. B: `next()` → C
7. **C 6** — C ishlaydi
8. C: `value += 100` → 106
9. C returns
10. **B out 106** — B `next()` dan keyin
11. B returns
12. A: `value -= 1` → 105
13. **A out 105** — A `next()` dan keyin

**Ikkinchi execute `{ value: 10 }`:**

1. **A in 10**
2. A: `value *= 2` → 20
3. A: `next()` → B
4. **B in 20**
5. B: 20 > 10 — `value = 0`, `return` (next() chaqirilmaydi)
6. C ishlamaydi! Zanjir to'xtadi
7. A: `next()` finished (B returned)
8. A: `value -= 1` → 0 - 1 = -1
9. **A out -1**

**Zanjir to'xtatish:** B `next()` chaqirmasa — C va keyingilar ishlamaydi. Lekin A'ning `after next()` qismi hali ishlaydi.

### Kod misol

```typescript
// === Real middleware order ===
const app = new Pipeline()
  .use(async (ctx, next) => {
    console.log("1. Request started");
    await next();
    console.log("5. Response sent");
  })
  .use(async (ctx, next) => {
    console.log("2. Auth check");
    if (!ctx.value) {
      console.log("AUTH FAILED");
      return;
    }
    await next();
    console.log("4. Cleanup");
  })
  .use(async (ctx) => {
    console.log("3. Handler");
  });

// Valid request
await app.execute({ value: 1 });
// 1. Request started
// 2. Auth check
// 3. Handler
// 4. Cleanup
// 5. Response sent

console.log("---");

// Failed auth
await app.execute({ value: 0 });
// 1. Request started
// 2. Auth check
// AUTH FAILED
// 5. Response sent       <- A "after" hali ishlaydi
```

### Edge Cases

- **`next()` skip:** zanjir to'xtaydi, lekin awaiter middleware'larning "after next()" code'i hali ishlaydi
- **`next()` ikki marta:** undefined behavior — har middleware bir marta chaqirishi shart
- **Throw `next()` dan oldin:** outer `try/catch`'da catch qilinadi yoki propagates outer
- **Sync mutation visible:** ctx mutate qilinishi — har middleware ko'radi (shared reference)
- **`await` order:** `await next()` block qiladi — keyin "after" code. `next().then(...)` — non-blocking

### Follow-up savollar

1. **"`next()` ni chaqirmaslik foydalimi?"** — Ha, early return — auth fail, validation fail, cache hit case'lar
2. **"Pipeline parallel middleware?"** — `Promise.all([fetchUser(), fetchOrder()])` middleware ichida — concurrent I/O. Lekin middleware'lar o'rtasida concurrency yo'q (await chain)

</details>

---

## Coding savollar

### Savol 18: Type-safe `EventEmitter<Events>` [Senior]

**Savol:** `on`, `off`, `emit`, `once`, `removeAllListeners` — barcha event name va argument type'lari compile-time'da tekshirilsin:

```typescript
interface UserEvents {
  login: [userId: string, timestamp: Date];
  logout: [userId: string];
  purchase: [userId: string, productId: string, amount: number];
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic `TypedEventEmitter<Events extends Record<string, unknown[]>>`. Har method `K extends keyof Events` constraint bilan event name'ni cheklaydi. Argument'lar `...args: Events[K]` tuple spread orqali type-safe.

### To'liq tushuntirish

**Type design:**

```typescript
type EventMap = Record<string, unknown[]>;
// Key: event name (string literal)
// Value: tuple — argument lar list
```

**Method signature:**

```typescript
on<K extends keyof Events>(
  event: K,
  handler: (...args: Events[K]) => void
): () => void
```

- `K extends keyof Events` — faqat declared event'lar
- `(...args: Events[K]) => void` — tuple → function signature
- Return `() => void` — unsubscribe function (cleanup)

**`emit` type-safety:**

```typescript
emit<K extends keyof Events>(event: K, ...args: Events[K]): void
```

- `event: K` — valid event name
- `...args: Events[K]` — tuple spread, type-checked

### Kod misol

```typescript
type EventMap = Record<string, unknown[]>;

class TypedEventEmitter<Events extends EventMap> {
  private listeners = new Map<keyof Events, Set<(...args: any[]) => void>>();

  on<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);
    return () => { this.listeners.get(event)?.delete(handler); };
  }

  off<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): void {
    this.listeners.get(event)?.delete(handler);
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    const handlers = this.listeners.get(event);
    if (!handlers) return;
    handlers.forEach((h) => {
      try {
        h(...args);
      } catch (e) {
        console.error(`Error in handler for ${String(event)}:`, e);
      }
    });
  }

  once<K extends keyof Events>(
    event: K,
    handler: (...args: Events[K]) => void
  ): () => void {
    const unsub = this.on(event, (...args) => {
      handler(...args);
      unsub();
    });
    return unsub;
  }

  removeAllListeners<K extends keyof Events>(event?: K): void {
    if (event) {
      this.listeners.delete(event);
    } else {
      this.listeners.clear();
    }
  }

  listenerCount<K extends keyof Events>(event: K): number {
    return this.listeners.get(event)?.size ?? 0;
  }
}


// === Usage — full type safety ===
interface UserEvents {
  login: [userId: string, timestamp: Date];
  logout: [userId: string];
  purchase: [userId: string, productId: string, amount: number];
}

const emitter = new TypedEventEmitter<UserEvents>();

// Listener — typed parameters
emitter.on("login", (userId, timestamp) => {
  // userId: string, timestamp: Date — auto-inferred
  console.log(`${userId} logged in at ${timestamp.toISOString()}`);
});

emitter.on("purchase", (userId, productId, amount) => {
  // amount: number
  console.log(`${userId} bought ${productId} for $${amount.toFixed(2)}`);
});

// Emit — typed arguments
emitter.emit("login", "user-1", new Date());          // ✅
emitter.emit("purchase", "user-1", "laptop", 1299);    // ✅
emitter.emit("logout", "user-1");                       // ✅

// emitter.emit("login", "user-1");                   // ❌ timestamp kerak
// emitter.emit("login", 42, new Date());             // ❌ userId string kerak
// emitter.emit("unknown");                            // ❌ event yo'q


// === Once + cleanup ===
const unsub = emitter.once("login", (userId, timestamp) => {
  console.log(`First login: ${userId}`);
});

emitter.emit("login", "user-1", new Date()); // "First login: user-1"
emitter.emit("login", "user-2", new Date()); // (nothing — once)


// === Cleanup pattern ===
const unsubscribers: (() => void)[] = [];

unsubscribers.push(
  emitter.on("login", (userId) => console.log(`Login: ${userId}`))
);
unsubscribers.push(
  emitter.on("logout", (userId) => console.log(`Logout: ${userId}`))
);

// Cleanup all
unsubscribers.forEach((unsub) => unsub());
```

### Edge Cases

- **`Set` insertion order:** listener'lar register tartibida chaqiriladi
- **Error in listener:** har handler `try/catch` da — bir handler fail bo'lsa, boshqalar davom etadi
- **Re-entrant emit:** listener ichida `emit` chaqirsa — recursion. Stack overflow xavfi
- **Listener modify during emit:** `Set` o'zgartirish iteration paytida — copy qilish (`[...handlers]`) afzal
- **Cleanup uchun unsub return:** har `on` `unsub` return qiladi — cleanup pattern qulay
- **Weak reference:** listener garbage collect bo'lishi kerak bo'lsa — `WeakRef` (rare, advanced)

### Follow-up savollar

1. **"Native `EventTarget` ga ko'chiriladimi?"** — `EventTarget`/`CustomEvent` ham mavjud, lekin TypeScript'da type-safety zaif (manual cast)
2. **"`once` cleanup memory leak xavfi?"** — Once never fires bo'lsa — wrapper saqlanadi. Manual cleanup yoki timeout
3. **"Async listener qanday handle qilinadi?"** — Listener `async` bo'lsa, `emit` await qilmaydi (fire and forget). Async kerak bo'lsa `emitAsync` — `Promise.all(handlers.map(h => h(...args)))`

<details>
<summary><strong>Deep Dive</strong></summary>

**Rest parameter as tuple type — TS 3.0+:**

`(...args: Events[K]) => void` — rest parameter type'i tuple bo'lsa, compiler uni discrete parameter ketma-ketligiga yoyadi (`[userId: string, timestamp: Date]` → `(userId: string, timestamp: Date)`). Bu "rest parameters with tuple types" feature'i — TS 3.0'da kelgan, TS-specific (ECMAScript proposal yo'q). Bu TS 4.0'ning "variadic tuple types" feature'i bilan adashtirmaslik kerak — variadic tuple types tuple type ta'rifi *ichida* generic spread element (`[string, ...T]`) ga oid, boshqa narsa.

**`keyof` cheklash mexanizmi:**

`K extends keyof Events` — `K` faqat `Events` da declared key bo'lishi mumkin. Compiler `keyof Events` ni union of literal types'ga aylantiradi (`"login" | "logout" | "purchase"`). `K extends "login" | "logout" | "purchase"` — narrowing. Boshqa string literal — assignability fail.

**`Map<keyof Events, Set<...>>` — runtime data structure:**

`Map` va `Set` insertion order'ni saqlaydi — bu ECMAScript spec kafolati (engine implementation detail emas). Listener'lar register tartibida iterate qilinadi. Performance: `Map.get(key)` — O(1) average, `Set.has` — O(1), `Set.forEach` — O(n).

**Type erasure issue:**

`new Map<keyof Events, Set<(...args: any[]) => void>>` — `any[]` runtime type ma'lumotini yo'qotadi. Compile-time `Events[K]` type-safe, lekin runtime'da `Set` har xil handler'larni saqlaydi. Type system bu trade-off'ni majburlaydi (TypeScript erased type system).

**Higher-Kinded Types simulation:**

`Events` parameter — type-level Higher-Kinded Type approximation. Per-event constraint — generic constraint chain. fp-ts kabi kutubxonalar HKT'ni encode qiladi (kind1, kind2) — verbose. Bu yondashuv pragmatik — TypeScript native HKT qo'llab-quvvatlamaydi.

**Cross-process Observer (Node Worker, MessagePort):**

`postMessage` orqali message — listener subscribe `MessagePort.on("message", handler)`. Serialization (structured clone) — Date, Map, Set, ArrayBuffer OK. Function, class instance — fail. Type-safety yo'qoladi (`event.data: any`). Manual type guard: `if (typeof data === "object" && "type" in data) { ... }`.

**`WeakRef` listener pattern (advanced):**

```typescript
const ref = new WeakRef(handler);
this.listeners.add(ref);

emit() {
  for (const ref of this.listeners) {
    const handler = ref.deref();
    if (handler) handler(...args);
    else this.listeners.delete(ref); // GC'd
  }
}
```

`WeakRef` — JS engine GC handler'ni listener tarafida tutmaydi. Lekin `WeakRef.deref` non-deterministic — GC qachon ishlashi noma'lum. Production'da kamdan-kam — explicit cleanup afzal.

</details>

</details>

---

### Savol 19: Generic `Result<T, E>` utility'lar [Middle+]

**Savol:** `Result<T, E>` type va `ok`/`err` constructors, `map`, `flatMap`, `unwrapOr`, `mapErr` yozing:

```typescript
// ok(5) → { _tag: "Ok", value: 5 }
// map(ok(5), n => n * 2) → ok(10)
// map(err("fail"), n => n * 2) → err("fail")
// flatMap(ok(5), n => n > 0 ? ok(n) : err("negative")) → ok(5)
// unwrapOr(err("fail"), 0) → 0
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Result<T, E>` — discriminated union `Ok<T> | Err<E>` with `_tag` discriminant. Utility'lar — `map` Ok'da fn apply, `flatMap` Ok'da Result-returning fn apply, `unwrapOr` Err'da default qaytaradi, `mapErr` Err transform.

### To'liq tushuntirish

**Type design:**
- `Ok<T>` va `Err<E>` — alohida type, `_tag` literal discriminant
- `readonly` — immutability, accidental mutation oldini olish
- `Result<T, E>` — union of `Ok<T> | Err<E>`

**Constructors (`ok`, `err`):** Generic factory — argument'dan type inference. `ok(5)` → `Ok<number>`, `err("fail")` → `Err<string>`.

**Type guards (`isOk`, `isErr`):** Return type `r is Ok<T>` — type predicate. Compiler `if (isOk(r))` ichida `r` ni `Ok<T>` ga narrow qiladi.

**`map` vs `flatMap`:**
- `map(r, fn: T → U)` — Ok'da fn apply, result `Ok<U>`. Funktor operation
- `flatMap(r, fn: T → Result<U, E>)` — Ok'da fn apply, lekin natija allaqachon Result — flatten. Monad operation

**`mapErr` — error transformation:** xato turini boshqasiga aylantirish (logging uchun, yoki domain error'ga normalize qilish).

**`unwrapOr` vs `unwrap`:** `unwrapOr` — Err'da default. `unwrap` — Err'da throw (defensive, faqat known-Ok holatda).

### Kod misol

```typescript
// === Result type ===
type Ok<T> = { readonly _tag: "Ok"; readonly value: T };
type Err<E> = { readonly _tag: "Err"; readonly error: E };
type Result<T, E> = Ok<T> | Err<E>;


// === Constructors ===
function ok<T>(value: T): Ok<T> {
  return { _tag: "Ok", value };
}

function err<E>(error: E): Err<E> {
  return { _tag: "Err", error };
}


// === Type guards ===
function isOk<T, E>(r: Result<T, E>): r is Ok<T> {
  return r._tag === "Ok";
}

function isErr<T, E>(r: Result<T, E>): r is Err<E> {
  return r._tag === "Err";
}


// === Utilities ===
function map<T, U, E>(r: Result<T, E>, fn: (value: T) => U): Result<U, E> {
  return isOk(r) ? ok(fn(r.value)) : r;
}

function flatMap<T, U, E>(
  r: Result<T, E>,
  fn: (value: T) => Result<U, E>
): Result<U, E> {
  return isOk(r) ? fn(r.value) : r;
}

function mapErr<T, E, F>(r: Result<T, E>, fn: (error: E) => F): Result<T, F> {
  return isErr(r) ? err(fn(r.error)) : r;
}

function unwrapOr<T, E>(r: Result<T, E>, defaultValue: T): T {
  return isOk(r) ? r.value : defaultValue;
}

function unwrap<T, E>(r: Result<T, E>): T {
  if (isErr(r)) {
    throw new Error(`Unwrap failed: ${JSON.stringify(r.error)}`);
  }
  return r.value;
}

function match<T, E, R>(
  r: Result<T, E>,
  patterns: { ok: (value: T) => R; err: (error: E) => R }
): R {
  return isOk(r) ? patterns.ok(r.value) : patterns.err(r.error);
}

// Combine — bir nechta Result ni AND
function combine<T, E>(results: Result<T, E>[]): Result<T[], E> {
  const values: T[] = [];
  for (const r of results) {
    if (isErr(r)) return r;
    values.push(r.value);
  }
  return ok(values);
}


// === Usage ===
// Result<number, string> deb annotate — aks holda err("fail") da T = unknown
// bo'lib qoladi va (n) => n * 2 da n: unknown type error beradi
const r1: Result<number, string> = ok(5);
const r2: Result<number, string> = err("fail");

// map
console.log(map(r1, (n) => n * 2));   // { _tag: "Ok", value: 10 }
console.log(map(r2, (n) => n * 2));    // { _tag: "Err", error: "fail" }

// flatMap — chaining
function validate(n: number): Result<number, string> {
  return n > 0 ? ok(n) : err("negative");
}
console.log(flatMap(ok(5), validate));   // { _tag: "Ok", value: 5 }
console.log(flatMap(ok(-3), validate));   // { _tag: "Err", error: "negative" }

// unwrapOr
console.log(unwrapOr(ok(5), 0));    // 5
console.log(unwrapOr(err("x"), 0));  // 0

// mapErr — error transform
const original = err({ code: 404, msg: "Not found" });
const transformed = mapErr(original, (e) => `${e.code}: ${e.msg}`);
console.log(transformed); // { _tag: "Err", error: "404: Not found" }


// === Railway-oriented programming ===
function parseInteger(s: string): Result<number, string> {
  const n = Number(s);
  return Number.isInteger(n) ? ok(n) : err(`Not an integer: ${s}`);
}

function checkPositive(n: number): Result<number, string> {
  return n > 0 ? ok(n) : err(`Not positive: ${n}`);
}

function squareRoot(n: number): Result<number, string> {
  return n >= 0 ? ok(Math.sqrt(n)) : err(`Negative: ${n}`);
}

// Chain
const result = match(
  flatMap(
    flatMap(parseInteger("16"), checkPositive),
    squareRoot
  ),
  {
    ok: (v) => `Result: ${v}`,
    err: (e) => `Error: ${e}`,
  }
);
console.log(result); // "Result: 4"


// === Combine multiple ===
const multi = combine([ok(1), ok(2), ok(3)]);
console.log(multi); // { _tag: "Ok", value: [1, 2, 3] }

const failed = combine([ok(1), err("oops"), ok(3)]);
console.log(failed); // { _tag: "Err", error: "oops" }
```

### Edge Cases

- **Inference of `Result<never, E>`:** `err("x")` returns `Err<string>` — generic inferred from argument
- **Variance:** `Result<Cat, E>` assignable to `Result<Animal, E>` — covariance in T
- **`unknown` error type:** `Result<T, unknown>` — caller narrowing kerak
- **Async Result:** `Promise<Result<T, E>>` — `async/await` natural
- **`unwrap` xavfi:** Err bo'lsa throw — defensive code emas. Faqat known-Ok holatda
- **Compatibility with fp-ts:** fp-ts `Either<E, T>` (left-error). Migration kerak bo'lsa adapter funksiya

### Follow-up savollar

1. **"Fluent API qanday qo'shiladi?"** — `Result` class qilish — `result.map(fn).flatMap(fn2).unwrapOr(0)`. Lekin discriminated union immutable
2. **"Async chaining qanday?"** — `flatMapAsync(r, async (v) => ...): Promise<Result<U, E>>` — har qadam `await` qilinadi. Rust'ning `?` operatori yo'q, shuning uchun har Err'ni manual `if (isErr(r)) return r` bilan propagate qilinadi

</details>

---

### Savol 20: Generic `Repository<T>` + in-memory [Middle+]

**Savol:** Generic Repository interface va `InMemoryRepository<T>` implementation yozing. `create` da `id`/dates avtomatik bo'lsin.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Repository<T extends Entity>` interface — CRUD method'lar `Omit<T, "id" | "createdAt" | "updatedAt">` bilan auto-managed field'lar exclude qilinadi. `InMemoryRepository<T>` — `Map<string, T>` orqali.

### To'liq tushuntirish

**Generic constraint (`T extends Entity`):** har T uchun `id`, `createdAt`, `updatedAt` field'lar majburiy. Bu base shape — repository implementation'lar generic logic ishlatishi mumkin.

**`Omit<T, K>` significance — `create` da:**
- Caller `id`, `createdAt`, `updatedAt` bermaydi (repository generate qiladi)
- Compile-time enforcement: `repo.create({ id: "manual" })` — type error
- Real-world: id — UUID/auto-increment, dates — server clock

**`Partial<Omit<T, ...>>` — `update` da:**
- `Partial` — har field optional (partial update)
- `Omit` — auto-managed field'lar yangilanmaydi
- Result: caller faqat domain field'larni ozgina/ko'p berishi mumkin

**`Map<string, T>`:** insertion order saqlaydi, `get`/`set`/`delete` — O(1) average. Plain object'dan afzal (prototype pollution yo'q, integer-key support).

**Filter (`Partial<T>`):** equality match — `Object.entries(filter).every(...)`. Limitation: range, regex yo'q. Production'da Specification pattern yoki query builder.

**Specialized repository:** `UserRepository extends InMemoryRepository<User>` — domain-specific method qo'shish (`findByEmail`). Generic base + specific extension.

### Kod misol

```typescript
interface Entity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T>;
  update(
    id: string,
    data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>
  ): Promise<T>;
  delete(id: string): Promise<boolean>;
  count(filter?: Partial<T>): Promise<number>;
}


class InMemoryRepository<T extends Entity> implements Repository<T> {
  protected items = new Map<string, T>();

  async findById(id: string): Promise<T | null> {
    return this.items.get(id) ?? null;
  }

  async findAll(filter?: Partial<T>): Promise<T[]> {
    let results = Array.from(this.items.values());
    if (filter) {
      results = results.filter((item) =>
        Object.entries(filter).every(
          ([key, value]) => item[key as keyof T] === value
        )
      );
    }
    return results;
  }

  async create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T> {
    const now = new Date();
    const entity = {
      ...data,
      id: crypto.randomUUID(),
      createdAt: now,
      updatedAt: now,
    } as T;
    this.items.set(entity.id, entity);
    return entity;
  }

  async update(
    id: string,
    data: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>
  ): Promise<T> {
    const existing = this.items.get(id);
    if (!existing) throw new Error(`Not found: ${id}`);
    const updated = { ...existing, ...data, updatedAt: new Date() } as T;
    this.items.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.items.delete(id);
  }

  async count(filter?: Partial<T>): Promise<number> {
    return (await this.findAll(filter)).length;
  }

  clear(): void {
    this.items.clear();
  }
}


// === Usage ===
interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "user";
}

const repo: Repository<User> = new InMemoryRepository<User>();

// Create — id, createdAt, updatedAt avtomatik
const user = await repo.create({
  name: "Ali",
  email: "ali@example.com",
  role: "admin",
});

console.log(user);
// { id: "uuid-...", name: "Ali", email: "ali@example.com",
//   role: "admin", createdAt: Date, updatedAt: Date }

// repo.create({ name: "X" }); // ❌ — email, role kerak
// repo.create({ id: "manual" }); // ❌ — id Omit qilingan

// Filter
const admins = await repo.findAll({ role: "admin" });

// Update
const updated = await repo.update(user.id, { name: "Aliyev" });
console.log(updated.updatedAt > user.updatedAt); // true

// Delete
const deleted = await repo.delete(user.id);
console.log(deleted); // true

// Specialized — UserRepository
class UserRepository extends InMemoryRepository<User> {
  async findByEmail(email: string): Promise<User | null> {
    const all = await this.findAll();
    return all.find((u) => u.email === email) ?? null;
  }
}
```

### Edge Cases

- **Generic constraint:** `T extends Entity` — har repository bir xil base field talab qiladi. Eski code (composite key) — separate interface
- **Filter limitation:** `Partial<T>` — equality match only. Range, regex, OR — qo'shimcha query object
- **Pagination:** `findAll({ offset, limit })` `Partial<T>` bilan ziddiyat. Yaxshi: alohida `FindOptions { filter, offset, limit, orderBy }`
- **Update audit:** `updatedAt` auto, `updatedBy` manual — extend qilish kerak
- **Soft delete:** `deletedAt: Date | null` field qo'shish, `findAll` deleted exclude

### Follow-up savollar

1. **"`Partial<T>` filter cheklangan — alternative?"** — Specification pattern, query object, yoki ORM-style chain (`repo.where(...).andWhere(...).get()`)
2. **"Transaction support qanday?"** — Repository `inTransaction(async (txRepo) => ...)` method. Postgres `BEGIN/COMMIT/ROLLBACK`

</details>

---

### Savol 21: Type-safe Builder phantom types [Senior]

**Savol:** `build()` faqat `url` VA `method` set bo'lganda chaqirilsin. Bo'lmasa compile error:

```typescript
// new Builder().setUrl("/api").setMethod("POST").build() → ✅
// new Builder().setUrl("/api").build() → ❌
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Phantom types — generic `State` parameter har setter'da yangilanadi (`State & { url: true }`). `build()` ning `this` parameter `Builder<{ url: true; method: true }>` ga cheklanadi. Runtime overhead yo'q.

### To'liq tushuntirish

**Phantom type** — type-level only ma'lumot, runtime'da hech qaerda saqlanmaydi. Faqat compiler graph'ida `State` parameter tracked.

**Setter chain mexanizmi:**
1. Initial state: `RequestBuilder<{}>`
2. `setUrl("/api")` → `RequestBuilder<{} & { url: true }>` = `RequestBuilder<{ url: true }>`
3. `setMethod("GET")` → `RequestBuilder<{ url: true } & { method: true }>` = `RequestBuilder<{ url: true; method: true }>`

Har setter return type — yangi intersection. Type system "ladder" — pastdan yuqoriga.

**`this` parameter cheklash:** TypeScript-specific feature. `build(this: RequestBuilder<{ url: true; method: true }>)` — call site'da `this` actual type bu signature'ga mos kelishi shart. Mos kelmasa — "method's 'this' is not assignable" error.

**`as any` cast — keraklimi?** Ha, sababi: runtime'da `return this` — bir xil instance. Lekin type-level'da `RequestBuilder<State>` va `RequestBuilder<State & { url: true }>` farqli (compiler perspective). `as any` — type system'ga "trust me" signal. Alternativa: `as unknown as RequestBuilder<...>` — verbose lekin explicit.

**Alternative: ordered step interface** — har bosqich alohida interface (`NeedsUrl` → `NeedsMethod` → `OptionalSteps`). IDE autocomplete faqat next step ko'rsatadi (discoverability yuqori). Trade-off: order flexibility yo'q.

### Kod misol

```typescript
interface HttpRequest {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  headers: Record<string, string>;
  body?: unknown;
  timeout: number;
}

type BuilderState = Partial<Record<"url" | "method", true>>;

class RequestBuilder<State extends BuilderState = {}> {
  private data: Partial<HttpRequest> = { headers: {}, timeout: 5000 };

  setUrl(url: string): RequestBuilder<State & { url: true }> {
    this.data.url = url;
    return this as any;
  }

  setMethod(m: HttpRequest["method"]): RequestBuilder<State & { method: true }> {
    this.data.method = m;
    return this as any;
  }

  addHeader(key: string, value: string): RequestBuilder<State> {
    this.data.headers = { ...this.data.headers, [key]: value };
    return this as any;
  }

  setBody(data: unknown): RequestBuilder<State> {
    this.data.body = data;
    return this as any;
  }

  setTimeout(ms: number): RequestBuilder<State> {
    this.data.timeout = ms;
    return this as any;
  }

  // build() faqat url VA method bor bo'lganda chaqirilishi mumkin
  build(this: RequestBuilder<{ url: true; method: true }>): HttpRequest {
    return {
      url: this.data.url!,
      method: this.data.method!,
      headers: this.data.headers ?? {},
      body: this.data.body,
      timeout: this.data.timeout ?? 5000,
    };
  }
}


// === Usage ===
new RequestBuilder()
  .setUrl("/api/users")
  .setMethod("POST")
  .addHeader("Content-Type", "application/json")
  .setBody({ name: "Ali" })
  .build(); // ✅ OK

new RequestBuilder()
  .setMethod("POST")
  .setUrl("/api/users")
  .build(); // ✅ — order matter qilmaydi

// new RequestBuilder().setUrl("/api").build();
// ❌ Type error:
// The 'this' context of type 'RequestBuilder<{ url: true; }>' is not
// assignable to method's 'this' of type 'RequestBuilder<{ url: true; method: true; }>'

// new RequestBuilder().build();
// ❌ Type error: similar


// === Ordered step builder (alternative) ===
interface NeedsUrl {
  setUrl(u: string): NeedsMethod;
}

interface NeedsMethod {
  setMethod(m: HttpRequest["method"]): OptionalSteps;
}

interface OptionalSteps {
  addHeader(k: string, v: string): OptionalSteps;
  setBody(data: unknown): OptionalSteps;
  setTimeout(ms: number): OptionalSteps;
  build(): HttpRequest;
}

class OrderedBuilder implements NeedsUrl, NeedsMethod, OptionalSteps {
  private data: Partial<HttpRequest> = { headers: {}, timeout: 5000 };

  setUrl(u: string): NeedsMethod {
    this.data.url = u;
    return this;
  }

  setMethod(m: HttpRequest["method"]): OptionalSteps {
    this.data.method = m;
    return this;
  }

  addHeader(k: string, v: string): OptionalSteps {
    this.data.headers![k] = v;
    return this;
  }

  setBody(data: unknown): OptionalSteps {
    this.data.body = data;
    return this;
  }

  setTimeout(ms: number): OptionalSteps {
    this.data.timeout = ms;
    return this;
  }

  build(): HttpRequest {
    return this.data as HttpRequest;
  }
}

function buildRequest(): NeedsUrl {
  return new OrderedBuilder();
}

buildRequest().setUrl("/api").setMethod("POST").build(); // ✅
// buildRequest().setMethod("POST"); // ❌ — setMethod NeedsUrl da yo'q
// buildRequest().setUrl("/api").build(); // ❌ — build NeedsMethod da yo'q
```

### Edge Cases

- **`as any` cast:** runtime ish phantom type'ga aloqasi yo'q — bir builder. Type system tomonidan yangi builder ko'rinadi
- **Builder reuse:** ikki marta `build()` chaqirish — bir xil instance, lekin type-level OK. Mutation problem
- **IDE autocomplete:** ordered builder'ga qaraganda — phantom type'da barcha method ko'rinadi (state allows). Ordered — faqat keyingi step
- **Type complexity:** ko'p required field — `State & { url: true } & { method: true } & ...` chain. Readable lekin verbose
- **Generic builder for any type:** `Builder<T, Required extends keyof T>` — har T uchun general

### Follow-up savollar

1. **"Phantom types vs ordered interface qaysi yaxshiroq?"** — Ordered: discoverability yuqori (next step ko'rsatadi). Phantom: order flexibility. Library API — ordered afzal
2. **"Runtime safety qaysi bo'lsa yaxshi?"** — Compile-time prevention. Runtime check (`if (!url) throw`) — fallback. Defensive code Production uchun
3. **"Builder fluent API toxic chain bo'ladimi?"** — Endless chain — readability past. 5-7 method'dan ko'p bo'lsa — multiple builder (sub-builder)

<details>
<summary><strong>Deep Dive</strong></summary>

**Phantom types — type-level encoding:**

Phantom type — runtime'da hech qanday ma'lumot saqlamaydi (compile-time only). `State extends BuilderState` — generic parameter, lekin `private data: Partial<HttpRequest>` da hech qanday `State` ga bog'liq field yo'q. `as any` cast — type system'ga "trust me" signal. JS output: `state` parameter ham, `State` type ham hech qaerda. Faqat compiler graph'ida.

**`this` parameter type — TS-specific feature:**

ECMAScript spec'da yo'q. TypeScript: function signature'da birinchi parameter `this: Type` — call site'da `this` type'ni majburlaydi. Misol: `build(this: RequestBuilder<{ url: true; method: true }>)` — `this` actual type mos kelmasa, "method call mode" da compile error.

**Type assignability — structural typing:**

`RequestBuilder<{ url: true }>` is NOT assignable to `RequestBuilder<{ url: true; method: true }>` — chunki `{ url: true }` da `method` property yo'q. Structural subtyping: `{ url: true; method: true }` is subtype of `{ url: true }` (more specific). Reverse — type error.

**Intersection types in builder state:**

`State & { url: true }` — type intersection. Compiler `{ a: number } & { b: string }` ni `{ a: number; b: string }` ga aylantiradi (object intersection = merge). Duplicate key — `{ a: number } & { a: string }` = `{ a: never }` (incompatible).

**HKT-like pattern (Higher-Kinded Type approximation):**

`Builder<T, Required extends keyof T>` — generic over both entity and required fields. Type-level: `Required` track edi qaysi field hali set qilinmagan. Implementation: TypeScript HKT yo'q — bu pattern manual encoding (verbose).

**Alternative: "subtype refinement" pattern (Scala-inspired):**

```typescript
interface UnsetBuilder { setUrl(u: string): SetUrlBuilder; }
interface SetUrlBuilder extends UnsetBuilder { setMethod(m: string): ReadyBuilder; }
interface ReadyBuilder extends SetUrlBuilder { build(): HttpRequest; }
```

Har "state" alohida interface — narrowing explicit. Phantom — generic-based, har state computed. Subtype refinement — declarative, IDE-friendly.

**Runtime cost analysis:**

Phantom types — zero runtime overhead. `as any` cast — JS output'da yo'q. `State` parameter — JS output'da yo'q. Compile-time only. Runtime: `data: Partial<HttpRequest>` — bir bayt ham extra emas.

**TypeScript spec — Generic Type Parameter Constraint:**

`State extends BuilderState = {}` — default constraint. Initial type `{}` — bo'sh object literal. Subsequent setter `State & { url: true }` — type grow. Compile-time graph: linear chain.

</details>

</details>

---

### Savol 22: Async middleware pipeline [Senior]

**Savol:** Async middleware pipeline — `use()` bilan middleware qo'shish, `execute()` bilan ishga tushirish. Onion model, error handling, type-safe context.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Pipeline` class — middleware stack saqlaydi. `execute` da recursive `next()` funksiya — har middleware o'z `next()` ni chaqirib keyingisiga o'tadi. Async/await orqali sequential execution. Error throw outer middleware'ga propagates.

### To'liq tushuntirish

**Pipeline anatomy:**
1. `middlewares: Middleware[]` — stack (insertion order)
2. `use(mw)` — middleware qo'shish, `this` qaytaradi (chaining)
3. `execute(ctx)` — pipeline ishga tushirish, modified context return

**Recursive `next()` — kalit mexanizm:**

```typescript
let index = 0;
const next = async () => {
  if (index < this.middlewares.length) {
    await this.middlewares[index++](ctx, next);
  }
};
```

- `index` closure ichida — shared mutable state
- Har middleware `await next()` — keyingisini chaqiradi
- `next()` chaqirilmasa — zanjir to'xtaydi (early return)
- Innermost middleware `next()` chaqirmasa ham OK (oxirgi)

**Onion model — execution order:**

```
errorHandler in → logger in → auth in → handler → auth out → logger out → errorHandler out
```

Har middleware ikki fazaga ega: "before next()" (request) va "after next()" (response).

**Error propagation:** Inner middleware throw → `await next()` reject → outer `try/catch` qabul qiladi. `errorHandler` outermost — barcha xatolarni tutadi. Async stack trace V8'da chain'ni saqlaydi.

**Middleware factory pattern:** `rateLimit(max, windowMs)` — closure'da config saqlaydi, `Middleware` qaytaradi. Customization uchun standart pattern.

**Order matters:** `errorHandler` birinchi (outermost), `auth` early (unauthorized request kirishini bloklash), `handler` oxirgi (innermost).

### Kod misol

```typescript
// === Context type ===
interface Context {
  request: {
    path: string;
    method: string;
    headers: Record<string, string>;
    body?: unknown;
  };
  response: {
    status: number;
    headers: Record<string, string>;
    body?: unknown;
  };
  state: Record<string, unknown>;
}

type NextFn = () => Promise<void>;
type Middleware = (ctx: Context, next: NextFn) => Promise<void>;


// === Pipeline implementation ===
class Pipeline {
  private middlewares: Middleware[] = [];

  use(mw: Middleware): this {
    this.middlewares.push(mw);
    return this;
  }

  async execute(ctx: Context): Promise<Context> {
    let index = 0;
    const next = async (): Promise<void> => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        await middleware(ctx, next);
      }
    };
    await next();
    return ctx;
  }
}


// === Production middleware lar ===

// 1. Error handler (outermost)
const errorHandler: Middleware = async (ctx, next) => {
  try {
    await next();
  } catch (e) {
    console.error("Pipeline error:", e);
    ctx.response = {
      status: 500,
      headers: { "Content-Type": "application/json" },
      body: { error: e instanceof Error ? e.message : "Internal error" },
    };
  }
};

// 2. Logger
const logger: Middleware = async (ctx, next) => {
  const start = Date.now();
  console.log(`-> ${ctx.request.method} ${ctx.request.path}`);
  await next();
  console.log(`<- ${ctx.response.status} (${Date.now() - start}ms)`);
};

// 3. CORS
const cors: Middleware = async (ctx, next) => {
  ctx.response.headers["Access-Control-Allow-Origin"] = "*";
  await next();
};

// 4. Body parser
const bodyParser: Middleware = async (ctx, next) => {
  if (ctx.request.method === "POST" && !ctx.request.body) {
    // Parse body (simulated)
    ctx.request.body = { name: "Ali" };
  }
  await next();
};

// 5. Auth
const auth: Middleware = async (ctx, next) => {
  const token = ctx.request.headers["authorization"];
  if (!token || !token.startsWith("Bearer ")) {
    ctx.response = {
      status: 401,
      headers: { "Content-Type": "application/json" },
      body: { error: "Unauthorized" },
    };
    return; // Zanjir to'xtaydi
  }
  ctx.state["userId"] = "user-from-token";
  await next();
};

// 6. Rate limiting
const rateLimit = (max: number, windowMs: number): Middleware => {
  const counts = new Map<string, { count: number; reset: number }>();
  return async (ctx, next) => {
    const ip = (ctx.request.headers["x-forwarded-for"] as string) || "unknown";
    const now = Date.now();
    const entry = counts.get(ip);

    if (!entry || entry.reset < now) {
      counts.set(ip, { count: 1, reset: now + windowMs });
    } else {
      entry.count++;
      if (entry.count > max) {
        ctx.response = {
          status: 429,
          headers: { "Content-Type": "application/json" },
          body: { error: "Too many requests" },
        };
        return;
      }
    }
    await next();
  };
};

// 7. Handler (innermost)
const handler: Middleware = async (ctx) => {
  ctx.response = {
    status: 200,
    headers: { "Content-Type": "application/json" },
    body: {
      message: "Success",
      userId: ctx.state["userId"],
      request: ctx.request.body,
    },
  };
};


// === Compose ===
const app = new Pipeline()
  .use(errorHandler)
  .use(logger)
  .use(cors)
  .use(rateLimit(100, 60_000))
  .use(auth)
  .use(bodyParser)
  .use(handler);


// === Execute ===
const validRequest: Context = {
  request: {
    path: "/api/users",
    method: "POST",
    headers: { authorization: "Bearer xxx", "x-forwarded-for": "1.2.3.4" },
  },
  response: { status: 0, headers: {} },
  state: {},
};

const result = await app.execute(validRequest);
console.log(result.response);
// {
//   status: 200,
//   headers: { ... },
//   body: { message: "Success", userId: "...", request: { name: "Ali" } }
// }


// === Failed auth ===
const unauthRequest: Context = {
  request: { path: "/api/users", method: "POST", headers: {} },
  response: { status: 0, headers: {} },
  state: {},
};

const unauthResult = await app.execute(unauthRequest);
console.log(unauthResult.response.status); // 401
```

### Edge Cases

- **`next()` not awaited:** `next()` ni `await` qilmasa — "after" code immediately ishlaydi, hali inner middleware tugamagan. Bug source
- **`next()` called twice:** undefined behavior — index ikki marta increment, race
- **Error in error handler:** outermost error handler o'zi throw qilsa — unhandled. Outermost try/catch yoki global handler kerak
- **Async generator middleware:** alternative pattern — yet uncommon
- **Type-safe context extension:** middleware state'ga property qo'shadi — TypeScript generic kerak (`Pipeline<State>`). Murakkab pattern
- **Cancellation:** uzoq middleware'ni to'xtatish — `AbortSignal` ctx'da. Built-in support — manual cancel logic

### Follow-up savollar

1. **"Express vs Koa vs Hono middleware?"** — Express: callback-based, err handling alohida. Koa: async/await native. Hono: TypeScript-first, eng modern
2. **"Concurrent middleware execution?"** — Pipeline ketma-ket. Concurrent kerak bo'lsa — `Promise.all` middleware ichida (parallel I/O)
3. **"Type-safe context evolution?"** — `Pipeline<S>` generic — har `use` `S & NewState` qaytaradi. Advanced — bir necha boundary kerak

<details>
<summary><strong>Deep Dive</strong></summary>

**Recursive `next()` — call stack analysis:**

```
execute()
└─ next() (i=0)
   └─ middleware[0](ctx, next)
      └─ await next() (i=1)
         └─ middleware[1](ctx, next)
            └─ await next() (i=2)
               └─ middleware[2](ctx, next)
                  └─ (no next() — innermost)
                  ← return
               ← middleware[2] done
            ← await complete
            (middleware[1] "after" code)
         ← middleware[1] done
      ← await complete
      (middleware[0] "after" code)
   ← middleware[0] done
← all done
```

Har `await next()` — async frame suspended (microtask queue). Resume tartibi LIFO (last-in-first-out) — bu onion model'ning bevosita oqibati.

**Closure capture — `next` variable:**

`next` funksiya `index` ga closure orqali bog'lanadi. Har chaqiruvda `index++` — shared mutable state. Bu `next()` ikki marta chaqirilsa, `index` ikki marta increment — duplicate middleware execution. Defensive: `let called = false; if (called) throw; called = true;` har middleware uchun.

**Promise microtask scheduling:**

`await next()` — Promise resolution kutib turadi. Event loop:
1. `middleware[0]` execute boshlanadi (sync code)
2. `await next()` — current task suspended, microtask queue'ga `next()` resolve callback push
3. `middleware[1]` execute boshlanadi (sync code, ichidagi `await`)
4. ... va h.k.

Innermost middleware tugagandan keyin, microtask queue'dan reverse order'da resume. Har `await` — bir microtask hop.

**Error propagation — async stack trace:**

Async function'da `throw` — Promise reject. `await next()` ichida throw bo'lsa, current `await` reject, exception "up" propagates `try/catch` block'ga. V8 — async stack trace (Node 12+, --async-stack-traces) — full chain debug uchun.

**Memory model — heap allocation:**

Har `Pipeline.execute` da yangi closure (`next`, `index`) — heap allocated. Har `await` da async function o'z state'ini saqlash uchun frame allocate qiladi (suspend/resume state machine). High-throughput server'da bu allocation cost bo'lishi mumkin — profiling kerak. V8 generational GC (Young Gen / Scavenger) — short-lived async frame'larni samarali tozalaydi.

**Backpressure va flow control:**

Pipeline yo'q built-in backpressure. Slow middleware — barchasi kutadi. Koa-like middleware yo'q backpressure'ni handle qilmaydi — explicit `AbortController` yoki timeout middleware kerak. Streaming kerak bo'lsa — Node.js `stream.Transform` afzal.

**Comparison: Koa source code reference:**

Koa `compose` function — bir xil pattern (`koa-compose` package, ~30 line). `dispatch(i)` recursive. Reference: https://github.com/koajs/compose/blob/master/index.js — production-grade implementation.

**Type-safe context evolution — advanced:**

```typescript
class TypedPipeline<S extends object> {
  use<T extends object>(
    mw: (ctx: S) => T
  ): TypedPipeline<S & T> {
    // mw ctx ni yangi field lar bilan kengaytiradi (S & T)
    // ...
    return this as unknown as TypedPipeline<S & T>;
  }
}
```

Har `use` middleware ctx'ga yangi field qo'shadi → pipeline type'i `S & T` ga o'sadi. Keyingi middleware avvalgilar qo'shgan field'larni type-safe ko'radi. Bunday context evolution inference og'ir — har `use` ning generic chaqiruvi orqali kechadi. Library-grade implementation: Hono framework — full type-safe context evolution.

</details>

</details>

---

### Savol 23: Command pattern undo/redo [Middle+]

**Savol:** TODO list uchun Command pattern bilan undo/redo. AddTodoCommand, CompleteTodoCommand, DeleteTodoCommand.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Command` interface — `execute()` + `undo()`. `CommandHistory` — done stack va undone stack. Har action `execute(command)` — done'ga push, undone tozalanadi. `undo()` — done'dan pop, undo chaqiriladi, undone'ga push.

### To'liq tushuntirish

**Command interface:** `execute(): T` (operation), `undo(): void` (inverse). Generic `T` — return type (e.g., new entity id). `description` — UI/logging uchun.

**Two-stack history mexanizmi:**

```
done: [add1, add2, complete1]  ← top
undone: []

undo() → done: [add1, add2], undone: [complete1]
undo() → done: [add1], undone: [complete1, add2]
redo() → done: [add1, add2], undone: [complete1]

new execute(delete1) → done: [add1, add2, delete1], undone: [] (tozalanadi)
```

**Undo logic — inverse operation:**
- `AddTodoCommand.undo` — items'dan filter (id bo'yicha)
- `CompleteTodoCommand.undo` — `previousState` ga qaytarish
- `DeleteTodoCommand.undo` — saqlangan `deletedItem` va `deletedIndex` ga splice insert

**Idempotency — redo uchun:** `AddTodoCommand` ichida `id` saqlanadi (`this.id`). Redo'da `getNextId()` qaytadan chaqirilmaydi — bir xil `id` qaytadi. Aks holda redo yangi id beradi (history inconsistency).

**CompositeCommand (Macro):** bir nechta command — bitta atomic operation. Undo reverse order'da (LIFO) — har subcommand undo qilinishi kerak teskari tartibda (chunki effect bir-biriga bog'liq).

**Stack tozalash:** Yangi command execute → `undone = []`. Sabab: undone — alternative future. Yangi action — bu future'ni invalidate qiladi (git checkout-like).

### Kod misol

```typescript
interface Command<T = void> {
  execute(): T;
  undo(): void;
  description: string;
}


class CommandHistory {
  private done: Command[] = [];
  private undone: Command[] = [];

  execute<T>(command: Command<T>): T {
    const result = command.execute();
    this.done.push(command);
    this.undone = []; // Redo stack tozalanadi
    return result;
  }

  undo(): boolean {
    const cmd = this.done.pop();
    if (!cmd) return false;
    cmd.undo();
    this.undone.push(cmd);
    return true;
  }

  redo(): boolean {
    const cmd = this.undone.pop();
    if (!cmd) return false;
    cmd.execute();
    this.done.push(cmd);
    return true;
  }

  canUndo() { return this.done.length > 0; }
  canRedo() { return this.undone.length > 0; }

  getHistory(): string[] {
    return this.done.map((c) => c.description);
  }
}


// === Domain ===
interface TodoItem {
  id: number;
  text: string;
  done: boolean;
}

class TodoList {
  items: TodoItem[] = [];
  private nextId = 1;

  getNextId(): number { return this.nextId++; }

  display(): string {
    return this.items
      .map((t) => `[${t.done ? "x" : " "}] #${t.id} ${t.text}`)
      .join("\n");
  }
}


// === Commands ===
class AddTodoCommand implements Command<number> {
  private id?: number;
  description: string;

  constructor(private list: TodoList, private text: string) {
    this.description = `Add "${text}"`;
  }

  execute(): number {
    if (this.id === undefined) {
      this.id = this.list.getNextId();
    }
    this.list.items.push({ id: this.id, text: this.text, done: false });
    return this.id;
  }

  undo(): void {
    if (this.id !== undefined) {
      this.list.items = this.list.items.filter((t) => t.id !== this.id);
    }
  }
}


class CompleteTodoCommand implements Command<void> {
  private previousState?: boolean;
  description: string;

  constructor(private list: TodoList, private id: number) {
    this.description = `Complete #${id}`;
  }

  execute(): void {
    const todo = this.list.items.find((t) => t.id === this.id);
    if (todo) {
      this.previousState = todo.done;
      todo.done = true;
    }
  }

  undo(): void {
    const todo = this.list.items.find((t) => t.id === this.id);
    if (todo && this.previousState !== undefined) {
      todo.done = this.previousState;
    }
  }
}


class DeleteTodoCommand implements Command<void> {
  private deletedItem?: TodoItem;
  private deletedIndex?: number;
  description: string;

  constructor(private list: TodoList, private id: number) {
    this.description = `Delete #${id}`;
  }

  execute(): void {
    const index = this.list.items.findIndex((t) => t.id === this.id);
    if (index >= 0) {
      this.deletedItem = this.list.items[index];
      this.deletedIndex = index;
      this.list.items.splice(index, 1);
    }
  }

  undo(): void {
    if (this.deletedItem && this.deletedIndex !== undefined) {
      this.list.items.splice(this.deletedIndex, 0, this.deletedItem);
    }
  }
}


// === Composite command (macro) ===
class CompositeCommand implements Command<void> {
  description: string;

  constructor(private commands: Command[], description: string) {
    this.description = description;
  }

  execute(): void {
    this.commands.forEach((c) => c.execute());
  }

  undo(): void {
    [...this.commands].reverse().forEach((c) => c.undo());
  }
}


// === Usage ===
const list = new TodoList();
const history = new CommandHistory();

const id1 = history.execute(new AddTodoCommand(list, "Buy milk"));
const id2 = history.execute(new AddTodoCommand(list, "Walk dog"));
const id3 = history.execute(new AddTodoCommand(list, "Read book"));

history.execute(new CompleteTodoCommand(list, id1));
history.execute(new CompleteTodoCommand(list, id2));

console.log(list.display());
// [x] #1 Buy milk
// [x] #2 Walk dog
// [ ] #3 Read book

console.log("History:", history.getHistory());
// ["Add \"Buy milk\"", "Add \"Walk dog\"", "Add \"Read book\"", "Complete #1", "Complete #2"]

// Undo 2 marta
history.undo();
history.undo();
console.log(list.display());
// [ ] #1 Buy milk
// [ ] #2 Walk dog
// [ ] #3 Read book

// Redo
history.redo();
console.log(list.display());
// [x] #1 Buy milk         <- Complete #1 qaytarildi
// [ ] #2 Walk dog
// [ ] #3 Read book

// Composite — undo bir vaqtda
const clearAll = new CompositeCommand(
  list.items.map((t) => new DeleteTodoCommand(list, t.id)),
  "Clear all"
);
history.execute(clearAll);
console.log(list.items.length); // 0

history.undo(); // Hammasini qaytarish
console.log(list.items.length); // 3
```

### Edge Cases

- **Idempotency:** redo execute'ni qayta chaqiradi — internal state (`id`) saqlangan bo'lishi kerak
- **External side effect:** command DB'ga yozsa — undo qiyin yoki imkonsiz (rollback transaction kerak)
- **History size:** cheksiz history → memory leak. LRU yoki limit
- **Composite undo order:** subcommand'lar reverse order'da undone — depending logic
- **New command after undo:** undone stack tozalanadi (standard semantics). Branch history — git-like, complex
- **Async command:** `execute(): Promise<T>` — `await` history'da

### Follow-up savollar

1. **"Memento pattern bilan farqi?"** — Command: operation + inverse. Memento: state snapshot — memory heavy, lekin universal undo (har state saqlanadi)
2. **"Distributed undo (collaborative editing)?"** — Operational Transformation yoki CRDT. Command pattern + transform — yaqin, lekin conflict resolution kerak

</details>

---

## Bug fix savollar

### Savol 24: Singleton + test state leak [Middle+]

**Savol:** Bu test ikkinchi marta ishga tushganda fail bo'ladi. Toping va tuzating:

```typescript
class Config {
  private static instance: Config | null = null;
  public debug = false;

  private constructor() {}

  static getInstance(): Config {
    if (!Config.instance) Config.instance = new Config();
    return Config.instance;
  }
}

describe("Config", () => {
  it("starts with debug = false", () => {
    const config = Config.getInstance();
    expect(config.debug).toBe(false);
  });

  it("allows enabling debug", () => {
    const config = Config.getInstance();
    config.debug = true;
    expect(config.debug).toBe(true);
  });

  it("starts fresh", () => {
    const config = Config.getInstance();
    expect(config.debug).toBe(false); // ❌ FAILS — hali true
  });
});
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Singleton `Config.instance` static property — test'lar orasida shared state. Ikkinchi test `debug = true` qildi, uchinchi test bir xil instance'ni oladi — state leak. Yechim: `beforeEach` da singleton'ni reset qilish yoki DI ishlatish.

### To'liq tushuntirish

**Muammo:**

Singleton — bitta instance app lifetime davomida. Test'lar uchun bu antagonistik: har test independent bo'lishi kerak, lekin singleton state previous test'dan kelaveradi.

```
Test 1: getInstance() → instance, debug = false ✅
Test 2: getInstance() → same instance, debug = true (set)
Test 3: getInstance() → same instance, debug hali true ❌
```

**Yechimlar:**

**1. Reset method (quick fix):**

```typescript
class Config {
  private static instance: Config | null = null;
  public debug = false;

  static getInstance() {
    if (!Config.instance) Config.instance = new Config();
    return Config.instance;
  }

  // Test uchun (production da ehtiyot)
  static reset(): void {
    Config.instance = null;
  }
}

beforeEach(() => {
  Config.reset();
});
```

**2. DI bilan almashtirish (afzal):**

```typescript
class Config {
  constructor(public debug: boolean = false) {}
}

// Test
beforeEach(() => {
  const config = new Config(); // Har test da yangi
});
```

**3. Test scope module:**

```typescript
// Module reload har test da (Jest)
jest.resetModules();
```

### Kod misol

```typescript
// ❌ XATO: Singleton leak
class Config {
  private static instance: Config | null = null;
  public debug = false;

  private constructor() {}

  static getInstance(): Config {
    if (!Config.instance) Config.instance = new Config();
    return Config.instance;
  }
}


// ✅ YECHIM 1: Reset method
class ConfigWithReset {
  private static instance: ConfigWithReset | null = null;
  public debug = false;

  private constructor() {}

  static getInstance(): ConfigWithReset {
    if (!ConfigWithReset.instance) {
      ConfigWithReset.instance = new ConfigWithReset();
    }
    return ConfigWithReset.instance;
  }

  static reset(): void {
    ConfigWithReset.instance = null;
  }
}

describe("ConfigWithReset", () => {
  beforeEach(() => {
    ConfigWithReset.reset(); // Yangi instance har test
  });

  it("starts with debug = false", () => {
    expect(ConfigWithReset.getInstance().debug).toBe(false);
  });

  it("allows enabling debug", () => {
    const config = ConfigWithReset.getInstance();
    config.debug = true;
    expect(config.debug).toBe(true);
  });

  it("starts fresh", () => {
    expect(ConfigWithReset.getInstance().debug).toBe(false); // ✅
  });
});


// ✅ YECHIM 2: DI (afzal)
class ConfigDI {
  constructor(public debug: boolean = false) {}
}

describe("ConfigDI", () => {
  let config: ConfigDI;

  beforeEach(() => {
    config = new ConfigDI(); // Yangi instance har test
  });

  it("starts with debug = false", () => {
    expect(config.debug).toBe(false);
  });

  it("allows enabling debug", () => {
    config.debug = true;
    expect(config.debug).toBe(true);
  });

  it("starts fresh", () => {
    expect(config.debug).toBe(false); // ✅
  });
});


// ✅ YECHIM 3: DI Container (production)
class Container {
  private bindings = new Map<any, any>();

  bind<T>(token: any, value: T): void {
    this.bindings.set(token, value);
  }

  get<T>(token: any): T {
    return this.bindings.get(token);
  }

  reset(): void {
    this.bindings.clear();
  }
}

const CONFIG_TOKEN = Symbol("Config");

describe("With container", () => {
  let container: Container;

  beforeEach(() => {
    container = new Container();
    container.bind(CONFIG_TOKEN, new ConfigDI());
  });

  it("starts fresh", () => {
    const config = container.get<ConfigDI>(CONFIG_TOKEN);
    expect(config.debug).toBe(false);
  });
});
```

### Edge Cases

- **Module-level singleton:** `export const config = new Config()` — module cached. Jest `jest.resetModules()` yoki test setup
- **`Object.freeze` singleton:** mutation oldini olish, lekin test'da kerak bo'lsa problematic
- **Async singleton:** `getInstance()` async — Promise cache leak
- **Inheritance:** child class o'zining singleton ham resetlash kerak
- **Cross-test pollution:** ESM module top-level code bir marta ishlaydi — test framework'lar buni bypass qilishi kerak

### Follow-up savollar

1. **"Production'da `reset()` method xavfsizmi?"** — Xavfli — anyone reset chaqirib state buzishi mumkin. Test-only build flag: `if (NODE_ENV !== "production")`
2. **"Module-level singleton uchun nima?"** — Jest `jest.resetModules()` har test'da. Yoki module-level singleton'dan voz kechib — factory function

</details>

---

### Savol 25: Observer memory leak [Middle+]

**Savol:** Bu kodda memory leak bor. Toping va tuzating:

```typescript
class App {
  constructor(private emitter: TypedEventEmitter<UserEvents>) {
    this.emitter.on("login", this.handleLogin.bind(this));
    this.emitter.on("logout", this.handleLogout.bind(this));
  }

  handleLogin(userId: string, timestamp: Date) {
    console.log(`${userId} logged in`);
  }

  handleLogout(userId: string) {
    console.log(`${userId} logged out`);
  }

  destroy() {
    // Cleanup yo'q!
  }
}

const emitter = new TypedEventEmitter<UserEvents>();
for (let i = 0; i < 1000; i++) {
  const app = new App(emitter);
  // app garbage collect bo'lmaydi — emitter bound handler ni saqlaydi
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`App` instance `emitter.on()` orqali handler register qiladi. Handler — `this.handleLogin.bind(this)` — `app` instance'ni `this` orqali ushlab turadi. Emitter listener'larni Set'da saqlaydi → `app` instance'lar garbage collect bo'lmaydi. Yechim: `destroy()` da `off()` chaqirish yoki `on()` qaytargan unsubscribe function'ni saqlash.

### To'liq tushuntirish

**Memory leak mexanizmi:**

```
emitter.listeners.get("login") = Set([handler1, handler2, ...])
handler1 = app1.handleLogin (bound to app1)
handler2 = app2.handleLogin (bound to app2)
...

Even if app1 reference lost, emitter saqlaydi handler1
handler1 → app1 (closure)
→ app1 GC'd qilinmaydi
```

**Bound method muammosi:** `this.handleLogin.bind(this)` — har instance uchun yangi function yaratadi va u `app` instance'ni `this` sifatida ushlab turadi. Bu bound function emitter Set'ida saqlanadi, shu sababli `app` ga reference uzilmaydi. (Bare `this.handleLogin` reference — prototype dagi yagona function, instance'ni ushlamasdi va leak bermasdi; lekin u holda chaqirilganda `this` `undefined` bo'lib, `console.log` ishlasa-da, instance'ga bog'lanmas edi.)

**Yechimlar:**

**1. Unsubscribe save va destroy'da off:**

```typescript
class App {
  private unsubs: (() => void)[] = [];

  constructor(private emitter: TypedEventEmitter<UserEvents>) {
    this.unsubs.push(
      this.emitter.on("login", this.handleLogin.bind(this))
    );
    this.unsubs.push(
      this.emitter.on("logout", this.handleLogout.bind(this))
    );
  }

  handleLogin = (userId: string, timestamp: Date) => {
    console.log(`${userId} logged in`);
  };

  handleLogout = (userId: string) => {
    console.log(`${userId} logged out`);
  };

  destroy(): void {
    this.unsubs.forEach((unsub) => unsub());
    this.unsubs = [];
  }
}
```

**2. AbortSignal pattern:**

```typescript
class AppWithAbort {
  private controller = new AbortController();

  constructor(private emitter: TypedEventEmitter<UserEvents>) {
    const { signal } = this.controller;

    const loginHandler = (userId: string, timestamp: Date) => { /* ... */ };
    const unsub = this.emitter.on("login", loginHandler);
    signal.addEventListener("abort", unsub);

    // ... boshqa listener lar
  }

  destroy() {
    this.controller.abort(); // Barcha listener lar unsubscribed
  }
}
```

**3. WeakRef (advanced, kamdan-kam):**

```typescript
class WeakObserver {
  // Emitter handler ni WeakRef bilan saqlaydi
  // App instance reference yo'qolsa, handler ham GC bo'ladi
}
```

### Kod misol

```typescript
// ❌ XATO: Memory leak
class LeakyApp {
  constructor(private emitter: TypedEventEmitter<UserEvents>) {
    this.emitter.on("login", this.handleLogin.bind(this));
    this.emitter.on("logout", this.handleLogout.bind(this));
  }

  handleLogin(userId: string, timestamp: Date) { /* ... */ }
  handleLogout(userId: string) { /* ... */ }

  destroy() {
    // Cleanup yo'q — leak!
  }
}


// ✅ YECHIM: Unsubscribe saqlash
class CleanApp {
  private unsubscribers: (() => void)[] = [];

  constructor(private emitter: TypedEventEmitter<UserEvents>) {
    this.unsubscribers.push(
      this.emitter.on("login", (userId, timestamp) => {
        this.handleLogin(userId, timestamp);
      })
    );

    this.unsubscribers.push(
      this.emitter.on("logout", (userId) => {
        this.handleLogout(userId);
      })
    );
  }

  private handleLogin(userId: string, timestamp: Date) {
    console.log(`${userId} logged in`);
  }

  private handleLogout(userId: string) {
    console.log(`${userId} logged out`);
  }

  destroy(): void {
    this.unsubscribers.forEach((unsub) => unsub());
    this.unsubscribers = [];
  }
}


// === AbortController pattern (modern) ===
class ModernApp {
  private controller = new AbortController();

  constructor(private emitter: TypedEventEmitter<UserEvents>) {
    const handleLogin = (userId: string, timestamp: Date) => {
      console.log(`${userId} logged in`);
    };

    const unsubLogin = this.emitter.on("login", handleLogin);
    this.controller.signal.addEventListener("abort", unsubLogin);

    const handleLogout = (userId: string) => {
      console.log(`${userId} logged out`);
    };

    const unsubLogout = this.emitter.on("logout", handleLogout);
    this.controller.signal.addEventListener("abort", unsubLogout);
  }

  destroy(): void {
    this.controller.abort(); // Barcha cleanup
  }
}


// === Test no-leak ===
const emitter = new TypedEventEmitter<UserEvents>();

for (let i = 0; i < 1000; i++) {
  const app = new CleanApp(emitter);
  // ... use app
  app.destroy(); // ✅ Cleanup — listener olib tashlandi
}

console.log(emitter.listenerCount("login")); // 0
```

### Edge Cases

- **Forget destroy:** caller `destroy()` chaqirmasa — leak hali bor. `WeakRef` yoki framework-level cleanup
- **Async leak:** `setTimeout` ichidagi listener — timeout cleanup ham kerak
- **Circular reference:** app → emitter → app — modern engine GC handle qila oladi, lekin pattern problematic
- **DOM listener:** `element.addEventListener` ham bir xil pattern — `removeEventListener` kerak
- **React useEffect cleanup:** `return () => unsub()` — automatic cleanup pattern

### Follow-up savollar

1. **"`bind(this)` vs arrow method qaysi yaxshiroq?"** — Arrow class field — har instance'da yangi function (memory). `bind(this)` — har constructor call'da yangi reference. Functionally bir xil
2. **"WeakRef qachon ishlatiladi?"** — Cache, library internal — caller cleanup qilmaydi. Advanced. Production'da kamdan-kam
3. **"React component bilan o'xshashlik?"** — useEffect cleanup pattern — har subscription uchun return cleanup. Same principle

</details>

---

## Xulosa

- **Factory** — generic + type map orqali type-safe object creation
- **Abstract Factory** — bir-biriga bog'liq object oilasi (theme, platform)
- **Singleton** — bitta instance. Module-level yoki DI scope yaxshiroq pure singleton'dan
- **Builder** — Fluent (runtime check) vs Step (phantom types compile-time enforcement)
- **Strategy** — runtime'da algoritm almashtirish, Open/Closed Principle
- **Adapter** — interface compatibility (1-to-1 mapping), legacy/third-party integration
- **Facade** — complexity hiding (1-to-many), subsystem orchestration
- **Proxy** — qo'shimcha behavior (caching, access control, lazy, logging)
- **Observer** — typed EventEmitter, `Events[K]` tuple bilan compile-time type-safety
- **Command** — operation + inverse, undo/redo, transaction
- **State Machine** — discriminated union, exhaustive check, type-safe transitions
- **Result/Either** — exception o'rniga `Ok<T> | Err<E>`, compile-time error handling
- **Repository** — generic CRUD, `Omit` bilan auto-managed field, persistence abstraction
- **Middleware Pipeline** — onion model, async chain of responsibility, `next()` semantics
- **Singleton + test:** state leak — reset yoki DI
- **Observer + cleanup:** listener `unsub` return saqlash yoki AbortController