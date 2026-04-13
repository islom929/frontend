# Bo'lim 21: Design Patterns TypeScript da

> Design pattern — takrorlanadigan muammolarga isbot qilingan yechim. TypeScript ning type system i (generics, interfaces, discriminated unions, conditional types) pattern larni **compile-time type safety** bilan kuchaytirib, runtime xatolarni deyarli yo'q qiladi. Har bir pattern TS-specific implementation bilan — type inference, generic constraints, va branded types yordamida — ancha kuchli va xavfsiz bo'ladi.

---

## Mundarija

- [Creational: Factory Pattern](#creational-factory-pattern)
- [Creational: Abstract Factory](#creational-abstract-factory)
- [Creational: Singleton Pattern](#creational-singleton-pattern)
- [Creational: Builder Pattern](#creational-builder-pattern)
- [Structural: Adapter Pattern](#structural-adapter-pattern)
- [Structural: Facade Pattern](#structural-facade-pattern)
- [Structural: Proxy Pattern](#structural-proxy-pattern)
- [Behavioral: Observer Pattern](#behavioral-observer-pattern)
- [Behavioral: Strategy Pattern](#behavioral-strategy-pattern)
- [Behavioral: Command Pattern](#behavioral-command-pattern)
- [Behavioral: State Machine](#behavioral-state-machine)
- [Cross-cutting: Repository Pattern](#cross-cutting-repository-pattern)
- [Cross-cutting: Result/Either Pattern](#cross-cutting-resulteither-pattern)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Creational: Factory Pattern

### Nazariya

Factory pattern — object yaratish logikasini bir joyga to'plab, client koddan yashiradi. TypeScript da generic factory juda kuchli — type inference orqali har bir factory chaqiruvida to'g'ri type qaytariladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Generic Map-based Factory ===
interface NotificationMap {
  email: EmailNotification;
  sms: SMSNotification;
  push: PushNotification;
}

interface EmailNotification { type: "email"; to: string; subject: string; body: string; }
interface SMSNotification { type: "sms"; phone: string; message: string; }
interface PushNotification { type: "push"; token: string; title: string; }

const factories: { [K in keyof NotificationMap]: () => NotificationMap[K] } = {
  email: () => ({ type: "email", to: "", subject: "", body: "" }),
  sms: () => ({ type: "sms", phone: "", message: "" }),
  push: () => ({ type: "push", token: "", title: "" }),
};

function createNotification<K extends keyof NotificationMap>(type: K): NotificationMap[K] {
  return factories[type]();
}

const email = createNotification("email");
// type: EmailNotification — TS avtomatik aniqladi
email.subject; // ✅ — faqat email da bor

const sms = createNotification("sms");
sms.phone; // ✅
// sms.subject; // ❌ — sms da yo'q
```

</details>

---

## Creational: Abstract Factory

### Nazariya

Abstract Factory — o'zaro bog'liq object larning **oilasini** yaratadi. Client concrete class larni bilmaydi — faqat interface orqali ishlaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface Button { render(): string; onClick(handler: () => void): void; }
interface Input { render(): string; getValue(): string; }

interface UIFactory {
  createButton(label: string): Button;
  createInput(placeholder: string): Input;
}

class MaterialFactory implements UIFactory {
  createButton(label: string): Button {
    return {
      render: () => `<md-button>${label}</md-button>`,
      onClick: (h) => { /* Material click */ },
    };
  }
  createInput(placeholder: string): Input {
    return {
      render: () => `<md-input placeholder="${placeholder}"/>`,
      getValue: () => "",
    };
  }
}

class BootstrapFactory implements UIFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="btn">${label}</button>`,
      onClick: (h) => { /* Bootstrap click */ },
    };
  }
  createInput(placeholder: string): Input {
    return {
      render: () => `<input class="form-control" placeholder="${placeholder}"/>`,
      getValue: () => "",
    };
  }
}

function createForm(factory: UIFactory) {
  const button = factory.createButton("Submit");
  const input = factory.createInput("Enter name");
  return { button, input };
}

const materialForm = createForm(new MaterialFactory());
const bootstrapForm = createForm(new BootstrapFactory());
```

</details>

---

## Creational: Singleton Pattern

### Nazariya

Singleton — class dan faqat **bitta instance** yaratilishini kafolatlaydi. TypeScript da `private constructor` + `static getInstance()` orqali. Module scope da oddiy variable bilan ham singleton yaratish mumkin (va bu ko'pincha yaxshiroq).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Class-based Singleton ===
class DatabaseConnection {
  private static instance: DatabaseConnection | null = null;

  private constructor(private connectionString: string) {}

  static getInstance(connectionString?: string): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection(
        connectionString || "postgres://localhost/db"
      );
    }
    return DatabaseConnection.instance;
  }

  query(sql: string) { console.log(`[DB] ${sql}`); return []; }
}

const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();
console.log(db1 === db2); // true

// new DatabaseConnection("..."); // ❌ — private constructor

// === Module-based Singleton (oddiyroq) ===
// db.ts
const connection = new SomeDBDriver("postgres://localhost/db");
export default connection;

// Har joyda import qilganda BIR XIL instance qaytadi
```

</details>

---

## Creational: Builder Pattern

### Nazariya

Builder pattern — murakkab object ni **bosqichma-bosqich** yaratish. TypeScript da **fluent API** (method chaining) va **step builder** (compile-time enforcement) bilan. Step builder — har bosqichda faqat ruxsat etilgan method lar ko'rinadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Fluent Builder ===
interface HttpRequest {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  headers: Record<string, string>;
  body?: unknown;
  timeout: number;
}

class RequestBuilder {
  private request: Partial<HttpRequest> = { headers: {}, timeout: 5000 };

  url(url: string): this { this.request.url = url; return this; }
  method(m: HttpRequest["method"]): this { this.request.method = m; return this; }
  header(key: string, value: string): this { this.request.headers![key] = value; return this; }
  body(data: unknown): this { this.request.body = data; return this; }
  timeout(ms: number): this { this.request.timeout = ms; return this; }

  build(): HttpRequest {
    if (!this.request.url || !this.request.method) {
      throw new Error("url and method required");
    }
    return this.request as HttpRequest;
  }
}

const req = new RequestBuilder()
  .url("/api/users")
  .method("POST")
  .header("Content-Type", "application/json")
  .body({ name: "Ali" })
  .timeout(10000)
  .build();

// === Step Builder — compile-time enforcement ===
interface NeedsUrl { url(u: string): NeedsMethod; }
interface NeedsMethod { method(m: "GET" | "POST" | "PUT" | "DELETE"): OptionalSteps; }
interface OptionalSteps {
  header(k: string, v: string): OptionalSteps;
  body(data: unknown): OptionalSteps;
  build(): HttpRequest;
}

class StepRequestBuilder implements NeedsUrl, NeedsMethod, OptionalSteps {
  private req: Partial<HttpRequest> = { headers: {} };

  url(u: string): NeedsMethod { this.req.url = u; return this; }
  method(m: HttpRequest["method"]): OptionalSteps { this.req.method = m; return this; }
  header(k: string, v: string): OptionalSteps { this.req.headers![k] = v; return this; }
  body(data: unknown): OptionalSteps { this.req.body = data; return this; }
  build(): HttpRequest { return this.req as HttpRequest; }
}

function request(): NeedsUrl { return new StepRequestBuilder(); }

request().url("/api").method("GET").build(); // ✅
// request().method("GET").build(); // ❌ — url() kerak avval
// request().url("/api").build();   // ❌ — method() kerak
```

</details>

---

## Structural: Adapter Pattern

### Nazariya

Adapter — **mos kelmaydigan interface** larni bir-biriga moslashtiradi. Old code va new code, yoki tashqi library va ichki system orasidagi ko'prik.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Old logger (tashqi kutubxona)
interface OldLogger {
  writeLog(severity: number, msg: string): void;
}

// New logger interface (bizning system)
interface Logger {
  info(msg: string): void;
  warn(msg: string): void;
  error(msg: string): void;
}

// Adapter — old interface ni new ga moslashtiradi
class LoggerAdapter implements Logger {
  constructor(private oldLogger: OldLogger) {}

  info(msg: string) { this.oldLogger.writeLog(0, msg); }
  warn(msg: string) { this.oldLogger.writeLog(1, msg); }
  error(msg: string) { this.oldLogger.writeLog(2, msg); }
}

// Ishlatish:
const legacy: OldLogger = { writeLog: (s, m) => console.log(`[${s}] ${m}`) };
const logger: Logger = new LoggerAdapter(legacy);
logger.info("Application started"); // [0] Application started
```

</details>

---

## Structural: Facade Pattern

### Nazariya

Facade — murakkab subsystem ga **sodda interface** beradi. Client murakkab internal API larni bilmaydi — facade orqali ishlaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Murakkab subsystem lar
class AuthService { login(email: string, password: string) { return { token: "jwt-token" }; } }
class UserService { getProfile(token: string) { return { name: "Ali", email: "ali@test.com" }; } }
class NotificationService { sendWelcome(email: string) { console.log(`Welcome email to ${email}`); } }
class AnalyticsService { trackLogin(userId: string) { console.log(`Login tracked: ${userId}`); } }

// Facade — barcha murakkablikni yashiradi
class AuthFacade {
  constructor(
    private auth: AuthService,
    private users: UserService,
    private notifications: NotificationService,
    private analytics: AnalyticsService
  ) {}

  async loginAndSetup(email: string, password: string) {
    const { token } = this.auth.login(email, password);
    const profile = this.users.getProfile(token);
    this.notifications.sendWelcome(profile.email);
    this.analytics.trackLogin(profile.name);
    return { token, profile };
  }
}

// Client — faqat facade bilan ishlaydi
const facade = new AuthFacade(new AuthService(), new UserService(), new NotificationService(), new AnalyticsService());
facade.loginAndSetup("ali@test.com", "secret");
```

</details>

---

## Structural: Proxy Pattern

### Nazariya

Proxy — object ga **qo'shimcha behavior** qo'shadi (caching, logging, access control, lazy loading) — original object ni o'zgartirmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface IUserService {
  getUser(id: number): Promise<{ id: number; name: string }>;
  getUsers(): Promise<{ id: number; name: string }[]>;
}

class UserServiceImpl implements IUserService {
  async getUser(id: number) { console.log(`DB query: user ${id}`); return { id, name: "Ali" }; }
  async getUsers() { console.log("DB query: all users"); return [{ id: 1, name: "Ali" }]; }
}

// Caching Proxy
class CachingUserProxy implements IUserService {
  private cache = new Map<string, { data: any; expires: number }>();

  constructor(private service: IUserService, private ttlMs: number = 60000) {}

  async getUser(id: number) {
    const key = `user:${id}`;
    const cached = this.cache.get(key);
    if (cached && cached.expires > Date.now()) return cached.data;

    const data = await this.service.getUser(id);
    this.cache.set(key, { data, expires: Date.now() + this.ttlMs });
    return data;
  }

  async getUsers() {
    const key = "users:all";
    const cached = this.cache.get(key);
    if (cached && cached.expires > Date.now()) return cached.data;

    const data = await this.service.getUsers();
    this.cache.set(key, { data, expires: Date.now() + this.ttlMs });
    return data;
  }
}

const service: IUserService = new CachingUserProxy(new UserServiceImpl());
await service.getUser(1); // "DB query: user 1"
await service.getUser(1); // (cache dan — DB query yo'q)
```

</details>

---

## Behavioral: Observer Pattern

### Nazariya

Observer — bir object (subject) holati o'zgarganda, barcha kuzatuvchilar (observer) **avtomatik xabar oladi**. TypeScript da **typed EventEmitter** pattern bilan type-safe qilinadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type EventMap = Record<string, (...args: any[]) => void>;

class TypedEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<T[keyof T]>>();

  on<K extends keyof T>(event: K, listener: T[K]): this {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(listener as any);
    return this;
  }

  off<K extends keyof T>(event: K, listener: T[K]): this {
    this.listeners.get(event)?.delete(listener as any);
    return this;
  }

  emit<K extends keyof T>(event: K, ...args: Parameters<T[K]>): void {
    this.listeners.get(event)?.forEach(fn => (fn as any)(...args));
  }
}

// Type-safe events:
interface AppEvents {
  userLoggedIn: (userId: string, timestamp: Date) => void;
  orderPlaced: (orderId: string, total: number) => void;
  error: (error: Error) => void;
}

const emitter = new TypedEmitter<AppEvents>();

emitter.on("userLoggedIn", (userId, timestamp) => {
  // userId: string, timestamp: Date — ✅ type-safe
  console.log(`${userId} logged in at ${timestamp}`);
});

emitter.emit("userLoggedIn", "user-1", new Date()); // ✅
// emitter.emit("userLoggedIn", 42, "wrong");        // ❌ — type error
// emitter.emit("unknownEvent");                      // ❌ — event yo'q
```

</details>

---

## Behavioral: Strategy Pattern

### Nazariya

Strategy — algoritmni **almashtiriladigan** qiladi. Interface orqali turli strategiya lar yaratiladi, runtime da keraklisi tanlanadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface PricingStrategy {
  calculate(basePrice: number, quantity: number): number;
}

class RegularPricing implements PricingStrategy {
  calculate(basePrice: number, quantity: number) { return basePrice * quantity; }
}

class BulkPricing implements PricingStrategy {
  calculate(basePrice: number, quantity: number) {
    const discount = quantity >= 100 ? 0.20 : quantity >= 50 ? 0.10 : 0;
    return basePrice * quantity * (1 - discount);
  }
}

class SeasonalPricing implements PricingStrategy {
  constructor(private multiplier: number) {}
  calculate(basePrice: number, quantity: number) {
    return basePrice * quantity * this.multiplier;
  }
}

class OrderCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy) { this.strategy = strategy; }

  getTotal(basePrice: number, quantity: number): number {
    return this.strategy.calculate(basePrice, quantity);
  }
}

const calc = new OrderCalculator(new RegularPricing());
console.log(calc.getTotal(10, 5)); // 50

calc.setStrategy(new BulkPricing());
console.log(calc.getTotal(10, 100)); // 800 (20% discount)

calc.setStrategy(new SeasonalPricing(1.5));
console.log(calc.getTotal(10, 5)); // 75 (1.5x)
```

</details>

---

## Behavioral: Command Pattern

### Nazariya

Command — operatsiyani **object sifatida** encapsulate qiladi. Bu undo/redo, queue, va logging uchun ishlatiladi. TypeScript da generic command interface + typed command bus.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface Command<T = void> {
  execute(): T;
  undo(): void;
}

class CommandHistory {
  private history: Command[] = [];

  execute<T>(command: Command<T>): T {
    const result = command.execute();
    this.history.push(command);
    return result;
  }

  undo() {
    const command = this.history.pop();
    command?.undo();
  }
}

// Typed commands
class AddItemCommand implements Command {
  constructor(private items: string[], private item: string) {}
  execute() { this.items.push(this.item); }
  undo() { this.items.pop(); }
}

class RemoveItemCommand implements Command<string | undefined> {
  private removed?: string;
  constructor(private items: string[], private index: number) {}
  execute() { this.removed = this.items.splice(this.index, 1)[0]; return this.removed; }
  undo() { if (this.removed) this.items.splice(this.index, 0, this.removed); }
}

const items: string[] = [];
const history = new CommandHistory();

history.execute(new AddItemCommand(items, "Apple"));
history.execute(new AddItemCommand(items, "Banana"));
console.log(items); // ["Apple", "Banana"]

history.undo();
console.log(items); // ["Apple"]

history.undo();
console.log(items); // []
```

</details>

---

## Behavioral: State Machine

### Nazariya

State Machine — object ning mumkin bo'lgan holatlarini va ular orasidagi o'tishlarni boshqaradi. TypeScript da **discriminated unions** bilan compile-time da noto'g'ri transition larni oldini olish mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Discriminated union bilan typed states
type OrderState =
  | { status: "draft"; items: string[] }
  | { status: "placed"; items: string[]; orderId: string }
  | { status: "paid"; items: string[]; orderId: string; paymentId: string }
  | { status: "shipped"; items: string[]; orderId: string; trackingNumber: string }
  | { status: "cancelled"; reason: string };

// Type-safe transitions
function placeOrder(state: Extract<OrderState, { status: "draft" }>): Extract<OrderState, { status: "placed" }> {
  if (state.items.length === 0) throw new Error("Cannot place empty order");
  return { status: "placed", items: state.items, orderId: crypto.randomUUID() };
}

function payOrder(state: Extract<OrderState, { status: "placed" }>): Extract<OrderState, { status: "paid" }> {
  return { ...state, status: "paid", paymentId: `pay-${Date.now()}` };
}

function cancelOrder(
  state: Extract<OrderState, { status: "draft" }> | Extract<OrderState, { status: "placed" }>,
  reason: string
): Extract<OrderState, { status: "cancelled" }> {
  return { status: "cancelled", reason };
}

// Ishlatish:
let order: OrderState = { status: "draft", items: ["Laptop"] };
order = placeOrder(order as Extract<OrderState, { status: "draft" }>);
order = payOrder(order as Extract<OrderState, { status: "placed" }>);

// cancelOrder(order, "reason"); // ❌ — "paid" state dan cancel mumkin emas
```

</details>

---

## Cross-cutting: Repository Pattern

### Nazariya

Repository — data access logic ni business logic dan ajratadi. TypeScript da **generic repository** — har qanday entity type bilan ishlaydigan type-safe CRUD interface.

<details>
<summary><strong>Kod Misollari</strong></summary>

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
  update(id: string, data: Partial<Omit<T, "id">>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

class InMemoryRepo<T extends Entity> implements Repository<T> {
  private store = new Map<string, T>();

  async findById(id: string) { return this.store.get(id) || null; }

  async findAll(filter?: Partial<T>) {
    const all = Array.from(this.store.values());
    if (!filter) return all;
    return all.filter(item =>
      Object.entries(filter).every(([k, v]) => (item as any)[k] === v)
    );
  }

  async create(data: Omit<T, "id" | "createdAt" | "updatedAt">) {
    const entity = {
      ...data,
      id: crypto.randomUUID(),
      createdAt: new Date(),
      updatedAt: new Date(),
    } as T;
    this.store.set(entity.id, entity);
    return entity;
  }

  async update(id: string, data: Partial<Omit<T, "id">>) {
    const existing = this.store.get(id);
    if (!existing) throw new Error(`Not found: ${id}`);
    const updated = { ...existing, ...data, updatedAt: new Date() } as T;
    this.store.set(id, updated);
    return updated;
  }

  async delete(id: string) { return this.store.delete(id); }
}

// Type-safe CRUD:
interface Product extends Entity { name: string; price: number; }
const repo = new InMemoryRepo<Product>();
const p = await repo.create({ name: "Laptop", price: 999 }); // ✅
// repo.create({ name: "Laptop" }); // ❌ — price kerak
```

</details>

---

## Cross-cutting: Result/Either Pattern

### Nazariya

Result/Either — **error handling** ning type-safe usuli. `throw` o'rniga `Ok<T> | Err<E>` qaytarish — caller xatoni handle qilishga **majbur bo'ladi** (compile-time).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> { return { ok: true, value }; }
function err<E>(error: E): Result<never, E> { return { ok: false, error }; }

// Type-safe error handling:
function parseJSON<T>(json: string): Result<T, SyntaxError> {
  try {
    return ok(JSON.parse(json));
  } catch (e) {
    return err(e as SyntaxError);
  }
}

function divideNumbers(a: number, b: number): Result<number, string> {
  if (b === 0) return err("Division by zero");
  return ok(a / b);
}

// Ishlatish — caller MAJBUR handle qiladi
const result = parseJSON<{ name: string }>('{"name":"Ali"}');

if (result.ok) {
  console.log(result.value.name); // ✅ — value: { name: string }
} else {
  console.error(result.error.message); // ✅ — error: SyntaxError
}

// Chaining:
function processUser(json: string): Result<string, string> {
  const parsed = parseJSON<{ name: string; age: number }>(json);
  if (!parsed.ok) return err(`Parse error: ${parsed.error.message}`);

  if (parsed.value.age < 18) return err("User must be 18+");
  return ok(`Welcome, ${parsed.value.name}!`);
}
```

</details>

---

## Edge Cases va Gotchas

### 1. Singleton va testing — global state muammosi

```typescript
class Config {
  private static instance: Config;
  private constructor(public debug: boolean) {}
  static getInstance() { return this.instance ??= new Config(false); }
}

// Test 1 — debug = true qiladi
Config.getInstance().debug = true;

// Test 2 — BIR XIL INSTANCE! debug hali true
// ❌ Test lar bir-birini buzadi

// Yechim: DI ishlatish yoki test da reset qilish
```

### 2. Builder pattern — `build()` da runtime error vs compile-time error

```typescript
// Fluent builder — runtime error
class Builder {
  build() {
    if (!this.url) throw new Error("url required"); // Runtime ❌
  }
}

// Step builder — compile-time error
interface NeedsUrl { url(u: string): NeedsMethod; }
// build() NeedsUrl da YO'Q — compile error ✅
```

### 3. Observer — memory leak (listener tozalanmasa)

```typescript
const emitter = new TypedEmitter<AppEvents>();

function setupPage() {
  emitter.on("userLoggedIn", handler); // Listener qo'shildi
  // Sahifa yopilganda off() chaqirilmasa — leak!
}

// Yechim: cleanup/dispose pattern
function setupPage() {
  emitter.on("userLoggedIn", handler);
  return () => emitter.off("userLoggedIn", handler); // Cleanup
}
```

### 4. Strategy pattern — `this` kontekst yo'qolishi

```typescript
class Calculator {
  constructor(private strategy: PricingStrategy) {}
  getTotal(price: number, qty: number) {
    return this.strategy.calculate(price, qty);
  }
}

// ❌ — method ni callback sifatida berilganda this yo'qoladi
const calc = new Calculator(new RegularPricing());
const fn = calc.getTotal;
fn(10, 5); // ❌ TypeError — this undefined

// Yechim: arrow function yoki .bind()
const fn2 = calc.getTotal.bind(calc); // ✅
```

### 5. Result pattern — `throw` bilan aralashtirganda type safety yo'qoladi

```typescript
function riskyFn(): Result<string, Error> {
  const result = someOperation(); // ❌ Agar someOperation throw qilsa?
  // Result pattern ichida throw bo'lsa — catch qilinmaydi
  // Result va throw — ikkalasini birga ishlatmang!
}
```

---

## Common Mistakes

### ❌ Xato 1: Factory da type narrowing ni unutish

```typescript
// ❌ — return type keng — client qo'shimcha narrowing kerak
function createNotification(type: string): Notification { /* ... */ }

// ✅ — generic + mapped type — return type aniq
function createNotification<K extends keyof NotificationMap>(type: K): NotificationMap[K] { /* ... */ }
```

### ❌ Xato 2: Singleton ni DI o'rniga global state sifatida ishlatish

```typescript
// ❌ — test qilish qiyin, global state
const db = DatabaseConnection.getInstance();

// ✅ — DI orqali inject
class UserService {
  constructor(private db: IDatabaseConnection) {} // Test da mock berish mumkin
}
```

### ❌ Xato 3: Observer da event name typo tutilmasligi

```typescript
// ❌ — string literal, typo tutilmaydi
emitter.on("userLogedIn", handler); // Typo! "Logged" emas "Loged"

// ✅ — TypedEmitter — faqat mavjud event lar
emitter.on("userLoggedIn", handler); // ✅ — autocomplete bor
// emitter.on("userLogedIn", handler); // ❌ — compile error
```

### ❌ Xato 4: Builder da immutable qoidani buzish

```typescript
// ❌ — bitta builder instance ni qayta ishlatish
const builder = new RequestBuilder().url("/api");
const req1 = builder.method("GET").build();
const req2 = builder.method("POST").build();
// ❌ req1 va req2 bir xil internal state ni share qiladi!

// ✅ — har safar yangi builder
const req1 = new RequestBuilder().url("/api").method("GET").build();
const req2 = new RequestBuilder().url("/api").method("POST").build();
```

### ❌ Xato 5: State machine da exhaustive check qilmaslik

```typescript
// ❌ — yangi status qo'shilganda switch da handle qilmaslik
function render(order: OrderState) {
  switch (order.status) {
    case "draft": return "Draft";
    case "placed": return "Placed";
    // "paid", "shipped", "cancelled" UNUTILDI!
  }
}

// ✅ — exhaustive check
function render(order: OrderState): string {
  switch (order.status) {
    case "draft": return "Draft";
    case "placed": return "Placed";
    case "paid": return "Paid";
    case "shipped": return "Shipped";
    case "cancelled": return "Cancelled";
    default: {
      const _exhaustive: never = order;
      return _exhaustive;
    }
  }
}
```

---

## Amaliy Mashqlar

### Mashq 1: Generic Factory (Oson)

**Savol:** `Shape` factory yarating — "circle", "square", "triangle" turlarini qabul qilib, type-safe shape qaytarsin.

<details>
<summary>Javob</summary>

```typescript
interface Circle { kind: "circle"; radius: number; }
interface Square { kind: "square"; side: number; }
interface Triangle { kind: "triangle"; base: number; height: number; }

interface ShapeMap { circle: Circle; square: Square; triangle: Triangle; }

function createShape<K extends keyof ShapeMap>(kind: K, props: Omit<ShapeMap[K], "kind">): ShapeMap[K] {
  return { kind, ...props } as ShapeMap[K];
}

const c = createShape("circle", { radius: 5 }); // Circle
const s = createShape("square", { side: 10 });   // Square
```

</details>

---

### Mashq 2: Typed EventEmitter (O'rta)

**Savol:** `on`, `off`, `emit` bilan typed EventEmitter yozing.

<details>
<summary>Javob</summary>

```typescript
type EventMap = Record<string, (...args: any[]) => void>;

class TypedEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<(...args: any[]) => void>>();

  on<K extends keyof T>(event: K, fn: T[K]): this {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(fn);
    return this;
  }

  off<K extends keyof T>(event: K, fn: T[K]): this {
    this.listeners.get(event)?.delete(fn);
    return this;
  }

  emit<K extends keyof T>(event: K, ...args: Parameters<T[K]>): void {
    this.listeners.get(event)?.forEach(fn => fn(...args));
  }
}
```

</details>

---

### Mashq 3: Result Type bilan Error Handling (O'rta)

**Savol:** `Result<T, E>` type va `map`, `flatMap` method larini yozing.

<details>
<summary>Javob</summary>

```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> { return { ok: true, value }; }
function err<E>(error: E): Result<never, E> { return { ok: false, error }; }

function map<T, U, E>(result: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return result.ok ? ok(fn(result.value)) : result;
}

function flatMap<T, U, E>(result: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return result.ok ? fn(result.value) : result;
}

// Chaining:
const result = flatMap(
  parseJSON<{ age: number }>('{"age": 25}'),
  (user) => user.age >= 18 ? ok(user) : err("Too young")
);
```

</details>

---

### Mashq 4: Step Builder (Qiyin)

**Savol:** `DatabaseConfig` uchun step builder — `host()`, `port()`, `database()` required, `username()`, `password()`, `ssl()` optional. `build()` faqat required tugaganda.

<details>
<summary>Javob</summary>

```typescript
interface DbConfig { host: string; port: number; database: string; username?: string; password?: string; ssl?: boolean; }

interface NeedsHost { host(h: string): NeedsPort; }
interface NeedsPort { port(p: number): NeedsDb; }
interface NeedsDb { database(db: string): OptionalSteps; }
interface OptionalSteps {
  username(u: string): OptionalSteps;
  password(p: string): OptionalSteps;
  ssl(v: boolean): OptionalSteps;
  build(): DbConfig;
}

class DbBuilder implements NeedsHost, NeedsPort, NeedsDb, OptionalSteps {
  private cfg: Partial<DbConfig> = {};
  host(h: string): NeedsPort { this.cfg.host = h; return this; }
  port(p: number): NeedsDb { this.cfg.port = p; return this; }
  database(db: string): OptionalSteps { this.cfg.database = db; return this; }
  username(u: string): OptionalSteps { this.cfg.username = u; return this; }
  password(p: string): OptionalSteps { this.cfg.password = p; return this; }
  ssl(v: boolean): OptionalSteps { this.cfg.ssl = v; return this; }
  build(): DbConfig { return this.cfg as DbConfig; }
}

function dbConfig(): NeedsHost { return new DbBuilder(); }

dbConfig().host("localhost").port(5432).database("myapp").build(); // ✅
// dbConfig().host("localhost").port(5432).build(); // ❌ — database kerak
```

</details>

---

### Mashq 5: Middleware Pipeline (Qiyin)

**Savol:** Generic middleware pipeline — `use()` bilan middleware qo'shish, `execute()` bilan ishga tushirish. `next()` chaqirilmasa zanjir to'xtaydi.

<details>
<summary>Javob</summary>

```typescript
interface Context {
  request: { path: string; method: string; headers: Record<string, string> };
  response: { status: number; body: unknown };
  state: Map<string, unknown>;
}

type Next = () => Promise<void>;
type Middleware = (ctx: Context, next: Next) => Promise<void>;

class Pipeline {
  private stack: Middleware[] = [];

  use(mw: Middleware): this { this.stack.push(mw); return this; }

  async execute(ctx: Context): Promise<Context> {
    let i = 0;
    const next = async (): Promise<void> => {
      if (i < this.stack.length) await this.stack[i++](ctx, next);
    };
    await next();
    return ctx;
  }
}

const app = new Pipeline()
  .use(async (ctx, next) => {
    console.log(`${ctx.request.method} ${ctx.request.path}`);
    await next();
  })
  .use(async (ctx, next) => {
    if (!ctx.request.headers["auth"]) {
      ctx.response = { status: 401, body: "Unauthorized" };
      return;
    }
    await next();
  })
  .use(async (ctx) => {
    ctx.response = { status: 200, body: { ok: true } };
  });
```

</details>

---

## Xulosa

Bu bo'limda TypeScript da design pattern larning type-safe implementatsiyalarini o'rgandik:

- **Creational** — Factory (generic map), Abstract Factory (product families), Singleton (private constructor), Builder (fluent API, step builder)
- **Structural** — Adapter (interface conversion), Facade (complexity yashirish), Proxy (caching, access control)
- **Behavioral** — Observer (typed EventEmitter), Strategy (interface + DI), Command (undo/redo), State Machine (discriminated unions)
- **Cross-cutting** — Repository (generic CRUD), Result/Either (`Ok<T> | Err<E>`)

TypeScript ning type system i — generics, discriminated unions, conditional types — design pattern larni **compile-time type safety** darajasiga ko'taradi.

**Bog'liq bo'limlar:**
- [Bo'lim 5: Union/Intersection](05-unions-intersections.md) — discriminated unions (state machine)
- [Bo'lim 8: Generics](08-generics.md) — generic patterns
- [Bo'lim 20: Reflect Metadata va DI](20-reflect-metadata-di.md) — IoC, DI container

---

**Keyingi bo'lim:** [22-tsconfig.md](22-tsconfig.md) — tsconfig.json Mastery: compiler options, strict mode, module resolution, project references, va barcha configuration imkoniyatlari.
