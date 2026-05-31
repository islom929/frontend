# Bo'lim 21: Design Patterns TypeScript'da

> Design pattern — takrorlanadigan dasturlash muammolariga vaqt sinovidan o'tgan strukturali yechim (Gang of Four 1994 va keyingi evolution'lar). TypeScript'ning type system'i (generics, discriminated unions, conditional types, mapped types) pattern'larni runtime'dan compile-time'ga ko'taradi — noto'g'ri foydalanish compiler bosqichida tutiladi. Bu bo'lim Creational, Structural, Behavioral va Cross-cutting kategoriyalar bo'yicha pattern'larning idiomatik TS implementation'ini ko'rib chiqadi.

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

Factory pattern — object yaratish logikasini alohida funksiya yoki method'ga to'plab, client koddan konkret class'ni yashiradi. Client `new ConcreteClass()` yozish o'rniga `factory.create(type)` chaqiradi — bu qaror'ni almashtirish (boshqa implementation'ga o'tish, configuration'ga qarab tanlash) faqat factory ichida amalga oshiriladi, client kod o'zgarmaydi.

TypeScript'da factory ikki darajada kuchayadi:
1. **Generic + key parameter** — `K extends keyof Map` orqali factory chaqirig'ida string literal'dan return type avtomatik tortib olinadi (type inference).
2. **Discriminated union** — return type literal'lar bilan markalanadi (`type: "email"`), keyin downstream kod `switch`/narrowing bilan exhaustive ishlay oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`createNotification<K extends keyof NotificationMap>(type: K): NotificationMap[K]` ifodasida nima sodir bo'ladi:

```text
1. Call site: createNotification("email")
2. TS type checker K ni inference qiladi:
     K = "email" (string literal type, kengaytirilmaydi)
3. Return type evaluate qilinadi:
     NotificationMap[K] = NotificationMap["email"] = EmailNotification
4. Result type — EmailNotification (string emas, Notification umumiy ham emas)
```

Agar `K extends keyof NotificationMap` constraint'i bo'lmasa, `K` literal'dan `string`'ga widen bo'lardi va `NotificationMap[K]` indekslash xato berardi (`string` `keyof NotificationMap`'ga assignable emas). `keyof` cheklov `K`'ni faqat ruxsat etilgan literal'lar bilan cheklab, indexed access type'ning to'g'ri ishlashini ta'minlaydi.

Runtime'da `factories[type]()` — oddiy object property access + call. Hech qanday metadata, decorator, polyfill kerak emas — type safety to'liq compile-time'da, runtime kod minimum.

</details>

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
email.subject; // ✅ — faqat email'da bor

const sms = createNotification("sms");
sms.phone; // ✅
// sms.subject; // ❌ — sms da yo'q
```

</details>

---

## Creational: Abstract Factory

### Nazariya

Abstract Factory — bir-biriga bog'liq (oilaga tegishli) object'larni yaratish uchun interface. Oddiy Factory bitta product turini yaratsa, Abstract Factory **bir butun product family**'sini (`Button` + `Input` + `Modal` + ...) yaratadi. Client `UIFactory` interface'ga bog'lanadi, lekin `MaterialFactory` yoki `BootstrapFactory` instance'ini almashtirsa, butun UI uslubi consistent o'zgaradi — `Material Button` bilan `Bootstrap Input`'ni aralashtirib qo'yish ehtimoli yo'q.

Nima muammoni hal qiladi: oilaviy mos kelmaslik. Agar client har bir komponentni alohida `new` orqali yaratsa, dasturchi tasodifan turli oilalardan komponent qo'shib yuborishi mumkin (Material + Bootstrap aralash UI). Abstract Factory butun oilani bitta source'dan oladi — mos kelish compiler darajasida kafolatlanadi.

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

Singleton — class'dan butun dastur davomida faqat bitta instance mavjud bo'lishini kafolatlaydi va shu instance'ga global access nuqtasini beradi. Klassik implementation: `private constructor` orqali tashqi `new` taqiqlanadi, `static getInstance()` lazy yaratish va keshlashni boshqaradi.

JavaScript/TypeScript ekosistemasida Singleton'ning **module-based** muqobili ko'p hollarda yaxshiroq: ES module bir marta yuklanadi va export qilingan qiymat module cache'da saqlanadi — har `import` BIR XIL reference qaytaradi. Bu Singleton'ning class boilerplate'isiz tabiiy implementation'i. Class-based Singleton qaerda kerak: framework integration (Angular service), interface implementation talab qilinganda yoki polymorphism orqali test'da almashtirish kerak bo'lganda.

**Anti-pattern ogohlantirishi:** Singleton — global mutable state demakdir. Test'lar orasida state leak, hidden dependency (qaysi modullar Singleton'ga bog'liq, kod'dan ko'rinmaydi) va concurrency muammolari uning kamchiligi. DI container bilan singleton scope ishlatish (lifecycle container boshqaradi, class o'zi emas) sof Singleton'dan xavfsizroq.

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
class PostgresDriver {
  constructor(private url: string) {}
  query(sql: string) { /* ... */ return []; }
}

const connection = new PostgresDriver("postgres://localhost/db");
export default connection;

// app.ts
// import db from "./db";   ← har import BIR XIL instance qaytaradi
// import db from "./db";   ← module cache sababli ikkinchi `new` yo'q
```

</details>

---

## Creational: Builder Pattern

### Nazariya

Builder — ko'p parametrli murakkab object'ni bosqichma-bosqich quradigan pattern. Klassik muammo: constructor 8 ta parametr olsa (yarmi optional) — `new User(name, email, undefined, undefined, true, undefined, 18, null)` o'qib bo'lmaydigan kod. Builder har parametr uchun nomli method beradi (`b.email(...)`/`b.age(...)`), keyin oxirida `build()` chaqirilganda yakuniy object qaytadi.

TypeScript'da builder ikki variantga ajraladi:

1. **Fluent Builder** — har setter `this` qaytaradi, chaqiruv zanjir bo'ladi. Validation `build()` ichida runtime'da. Soddaroq, lekin majburiy maydonni unutgan o'rinda compile error yo'q.

2. **Step Builder (Type-State pattern)** — har setter qaysi method'larni keyingisida chaqirish mumkinligini type bilan cheklaydi. Majburiy ketma-ketlik compiler darajasida tasdiqlanadi: `request().method(...)` chaqirig'i mumkin emas, chunki `request()` `NeedsUrl` qaytaradi, unda `method` method'i yo'q. Bu — Rust'dagi typestate pattern'ning TS variant'i.

Step Builder'ning kuchi: kod compilatsiyaga ulgursa, builder grammar'i to'g'ri ishlatilgan demakdir. Runtime check'lar yo'q.

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
  // headers timeout har doim mavjud — qolgan maydonlar optional
  private request: Partial<HttpRequest> & Pick<HttpRequest, "headers" | "timeout"> = {
    headers: {},
    timeout: 5000,
  };

  url(url: string): this { this.request.url = url; return this; }
  method(m: HttpRequest["method"]): this { this.request.method = m; return this; }
  header(key: string, value: string): this { this.request.headers[key] = value; return this; }
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
  private req: Partial<HttpRequest> & Pick<HttpRequest, "headers"> = { headers: {} };

  url(u: string): NeedsMethod { this.req.url = u; return this; }
  method(m: HttpRequest["method"]): OptionalSteps { this.req.method = m; return this; }
  header(k: string, v: string): OptionalSteps { this.req.headers[k] = v; return this; }
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

Adapter — bir-biriga mos kelmaydigan interface'larni bog'lab beradigan oraliq qatlam. Maqsad: client kod yangi/idiomatik interface bilan ishlasin, tashqi (eski yoki third-party) implementation esa o'zining boshqacha shaklida qolaversin. Adapter ichida transformation bo'ladi: client chaqirig'i tashqi API call'iga aylantiriladi (parametr'lar qayta tuziladi, payload format'i o'zgartiriladi, error model'i normallashtiriladi).

Adapter ikki yondashuvda yoziladi:
- **Object Adapter** (compozitsiya) — adapter tashqi object'ni `private` field'da saqlaydi. JS/TS'da deyarli har doim shu (multiple inheritance yo'q).
- **Class Adapter** (inheritance) — adapter tashqi class'dan extend qiladi. JS'da kamroq ishlatiladi.

Adapter'ning Facade va Proxy'dan farqi: Adapter **interface'ni o'zgartiradi** (signatura mos kelmaydi → moslashadi); Facade interface'ni o'zgartirmaydi, balki murakkablikni yashirib soddalashtiradi; Proxy bir xil interface'ni saqlab, behavior qo'shadi (caching, access control).

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

// Adapter — old interface'ni new'ga moslashtiradi
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

Facade — bir nechta murakkab subsystem'larni birlashtirib, client uchun yagona soddalashtirilgan interface beradigan pattern. Client ko'plab class/method'larni va ularning to'g'ri ketma-ketligini bilishi shart emas — facade'ning bitta high-level method'ini chaqiradi (`loginAndSetup`), facade ichida esa kerakli barcha kichik chaqiruv'lar to'g'ri tartibda amalga oshiriladi.

Facade Adapter'dan farqi: Facade interface'ni transformation qilmaydi, balki **sath darajasini** o'zgartiradi — ko'p kichik chaqiruvlar bittaga jamlanadi. Adapter mos kelmaslikni hal qiladi (signature mismatch), Facade murakkablikni yashiradi (low-level → high-level).

Real misol: HTTP client kutubxonalari (`fetch` o'rniga `axios`/`ky` — bularning ko'pi facade — XHR/fetch ustidan retry/interceptor/timeout boshqaruvini soddalashtirgan). NestJS'dagi `JwtService.signAsync()` — `node-jsonwebtoken`, kalit configuration'i va promisify ustidan facade.

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

Proxy — original object bilan **bir xil interface**'ni saqlab, har chaqiruvni boshqarish (intercept) imkonini beradigan o'rin egasi. Client proxy'ni real object'dan ajratib bilmaydi — chaqiriqlar shaklan bir xil, lekin proxy ichida qo'shimcha behavior (caching, access control, lazy initialization, logging, remote call) bajariladi.

Proxy variantlari:
- **Caching Proxy** — bir xil so'rov natijasini eslab qoladi, takror chaqiriqda real service'ga bormaydi (memoization).
- **Lazy Proxy** — haqiqiy object qimmat (DB connection, heavy resource) — proxy faqat birinchi murojaatda real object yaratadi.
- **Protection Proxy** — chaqiriq'ni amalga oshirishdan oldin permission tekshiradi.
- **Remote Proxy** — local object ko'rinishida tashqi xizmatga (gRPC, RPC, microservice) chaqiriq yuboradi.

JavaScript'ning built-in `Proxy` global'i (ES2015) — pattern'ning til darajasidagi implementation'i. Trap'lar (`get`, `set`, `apply`, `has`, ...) orqali har operation intercept qilinadi. Lekin pattern sifatida proxy shart emas — oddiy class wrapper ham yetadi va type safety yaxshi (Proxy global'ida type narrowing qiyinroq).

Adapter'dan farqi: Adapter interface'ni o'zgartiradi, Proxy aynan o'sha interface'ni saqlaydi — client kod o'zgarmaydi.

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
  private cache = new Map<string, { data: unknown; expires: number }>();

  constructor(private service: IUserService, private ttlMs: number = 60000) {}

  // Generic helper — har method o'z return type'ini saqlaydi, any kerak emas
  private async memoize<T>(key: string, fetch: () => Promise<T>): Promise<T> {
    const cached = this.cache.get(key);
    if (cached && cached.expires > Date.now()) return cached.data as T;

    const data = await fetch();
    this.cache.set(key, { data, expires: Date.now() + this.ttlMs });
    return data;
  }

  getUser(id: number) {
    return this.memoize(`user:${id}`, () => this.service.getUser(id));
  }

  getUsers() {
    return this.memoize("users:all", () => this.service.getUsers());
  }
}

const service: IUserService = new CachingUserProxy(new UserServiceImpl());
await service.getUser(1); // "DB query: user 1"
await service.getUser(1); // (cache'dan — DB query yo'q)
```

</details>

---

## Behavioral: Observer Pattern

### Nazariya

Observer — bir object (subject/publisher) holati o'zgarganda, unga obuna bo'lgan barcha kuzatuvchilar (observer/subscriber) avtomatik xabar oladigan pattern. Publisher subscriber'lar haqida hech narsa bilmaydi (faqat ularning interface'ini biladi) — loose coupling. Bu — event-driven architecture'ning poydevori.

JavaScript ekosistemasida Observer turli ko'rinishlarda mavjud:
- **Node.js `EventEmitter`** — built-in class, string event nomlar bilan ishlaydi (type-safe emas).
- **DOM `addEventListener`** — browser API darajasidagi observer.
- **RxJS `Observable`** — push-based async stream, operator pipeline bilan boyitilgan kengaytmasi.
- **Vue/MobX reactivity** — observer Proxy/getter ostida yashiringan (avtomatik dependency tracking).

TypeScript'da idiomatik yondashuv — **typed EventEmitter**: event nomlar va payload type'lari interface bilan markalaydi (`EventMap`). `on`/`emit` generic method'lar `keyof T` cheklov orqali noma'lum event yoki noto'g'ri argument'larni compile-time'da rad etadi. Bu Node `EventEmitter` API'sining type-safe variant'i.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type EventMap = Record<string, (...args: any[]) => void>;

class TypedEmitter<T extends EventMap> {
  // Set value type — umumiy callable signature (har event uchun T[K] generic cast emas)
  private listeners = new Map<keyof T, Set<(...args: any[]) => void>>();

  on<K extends keyof T>(event: K, listener: T[K]): this {
    let set = this.listeners.get(event);
    if (!set) {
      set = new Set();
      this.listeners.set(event, set);
    }
    set.add(listener);
    return this;
  }

  off<K extends keyof T>(event: K, listener: T[K]): this {
    this.listeners.get(event)?.delete(listener);
    return this;
  }

  emit<K extends keyof T>(event: K, ...args: Parameters<T[K]>): void {
    this.listeners.get(event)?.forEach(fn => fn(...args));
  }
}

// Type-safe events — `type` (interface emas):
// interface'da implicit index signature yo'q, shuning uchun u Record<string, ...>
// constraint'ini qanoatlantirmaydi. `type` alias esa qanoatlantiradi.
type AppEvents = {
  userLoggedIn: (userId: string, timestamp: Date) => void;
  orderPlaced: (orderId: string, total: number) => void;
  error: (error: Error) => void;
};

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

Strategy — algoritmni alohida object'ga (yoki funksiyaga) encapsulate qilib, runtime'da almashtirish imkonini beradigan pattern. Context class strategy interface'iga bog'lanadi, konkret strategy'ni faqat constructor yoki setter orqali oladi. Algoritm o'zgartirilganda context kod o'zgarmaydi — bu Open/Closed prinsipining amaliy ko'rinishi.

Class-based vs Function-based Strategy:
- **Class-based** — strategy interface implement qiluvchi class'lar (`RegularPricing`, `BulkPricing`). State (config, dependency) saqlash, polymorphism, OOP framework integration uchun mos.
- **Function-based** — strategy oddiy funksiya signature'i. State'siz pure algoritm uchun yetarli, kamroq boilerplate, tree-shake oson. `type PricingStrategy = (price: number, qty: number) => number`.

JavaScript ekosistemasida Strategy ko'pincha **higher-order function** sifatida tushuniladi: `array.sort(comparator)`'dagi `comparator` — strategy; `array.filter(predicate)`'dagi `predicate` — strategy. Class kerak emas, funksiya yetadi.

Strategy va polymorphism farqi: oddiy polymorphism'da behavior class type'iga bog'liq (subclassing). Strategy'da behavior **inject qilingan object**'ga bog'liq — bir xil class instance'i ham runtime'da strategy'ni almashtira oladi (`setStrategy()`).

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

Command — bajariladigan operation'ni object shaklida reify qiladigan pattern. Operation (action) bevosita funksiya chaqirig'i bo'lmasdan, `execute()` method'iga ega object'ga aylanadi — buni saqlash, navbatga qo'yish, log qilish, qaytarish (`undo`) yoki tarmoq orqali yuborish mumkin.

Asosiy foydalanish holatlari:
- **Undo/Redo** — har bajarilgan command stack'da saqlanadi; `undo()` orqali aksini bajarish (text editor, graphic tools, transaction managers).
- **Command queue / job scheduling** — task'larni queue'ga qo'shib keyinroq bajarish (background worker, retry logic).
- **Macro / batching** — bir nechta command'ni kompozit Command'ga birlashtirib bitta `execute()` chaqirig'i bilan amalga oshirish.
- **Audit log / event sourcing** — har command serialize qilinib saqlanadi; system state qayta qurishda command'lar replay qilinadi.
- **Remote execution** — command serialize bo'lib server'ga yuboriladi (RPC).

TypeScript'da `Command<T>` generic interface (`execute(): T`, `undo(): void`) command'ning return type'ini saqlaydi — `CommandHistory.execute()` to'g'ri type qaytaradi. Bu CQRS architecture'sidagi command bus'ning poydevori (NestJS `@nestjs/cqrs` module'i ham aynan shu modelda).

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

Finite State Machine (FSM) — object'ning aniq sanab bo'lingan holatlari (`states`) va ular orasidagi ruxsat etilgan o'tishlar (`transitions`) to'plamini formal ravishda tasvirlovchi model. Har bir paytda object aniq bitta state'da bo'ladi, transition esa "joriy state + event" juftligi orqali keyingi state'ni aniqlaydi.

TypeScript'da idiomatik implementation — **discriminated union**: har state alohida shape (faqat shu state'da mavjud field'lar bilan), `status` literal field discriminant rolida. Bu noto'g'ri combination'larni compile-time'da rad etadi: `placed` state'ida `paymentId` mavjud emas, compiler ushbu field'ga murojaatni xato deb belgilaydi.

Transition funksiyalari `Extract<State, { status: "draft" }>` orqali input state'ni cheklab oladi — `payOrder` faqat `placed` state'ni qabul qiladi, `draft` state'ni berishga urinish compilation bosqichida tutiladi. Bu **type-state pattern** — invariant'lar runtime check o'rniga type system tomonidan kafolatlanadi.

Alternativalar: `XState` kutubxonasi — JavaScript'da product-grade FSM/statecharts implementation'i (hierarchical states, parallel states, history). Manual implementation o'rganish va kichik holatlar uchun mos, lekin murakkab biznes flow'lar (ko'p state, guard'lar, side effect'lar) `XState` darajasini talab qiladi.

> **Eslatma:** Quyidagi misolda `as Extract<...>` cast'lar ishlatilgan, chunki `order` `let` bilan `OrderState` deb e'lon qilingan — qayta tayinlashda type narrowing yo'qoladi va `order` yana keng `OrderState`'ga widen bo'ladi. Real kod'da har transition'dan keyingi qiymatni alohida `const`'ga olish (yoki state machine'ni class ichida encapsulate qilish) cast'larni yo'qotadi va type narrowing tabiiy bo'ladi.

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

// cancelOrder(order, "reason"); // ❌ — "paid" state'dan cancel mumkin emas
```

</details>

---

## Cross-cutting: Repository Pattern

### Nazariya

Repository — data access logic'ni business logic'dan ajratadigan abstraction qatlami. Domain qatlami (entity'lar, use case'lar) `IUserRepository` interface'ga bog'lanadi; konkret implementation (`PostgresUserRepo`, `InMemoryUserRepo`, `MongoUserRepo`) infrastructure qatlamida joylashadi. Bu Domain-Driven Design (Eric Evans) va Clean Architecture (Robert Martin)'ning markaziy konseptlaridan biri.

Repository'ning Data Access Object (DAO)'dan farqi: DAO ko'pincha jadval/dokument bilan 1:1 mos keladi (CRUD'ning yupqa qatlami). Repository esa **aggregate**'lar bilan ishlaydi — bir nechta jadval/kolleksiyani birlashtirgan domain object'ni butun holida saqlaydi va o'qiydi. Repository client uchun "in-memory kolleksiya" illyuziyasini beradi (`repo.findById(id)` — DB query yashirin).

TypeScript'da generic `Repository<T extends Entity>` har qanday entity type bilan ishlaydigan type-safe CRUD beradi. `Omit<T, "id" | "createdAt" | "updatedAt">` — `create()`'ga repository tomonidan boshqariladigan field'larni o'tkazishni taqiqlaydi; `Partial<Omit<T, "id">>` — `update()`'da id o'zgartirishni man qiladi. Bu invariant'lar interface darajasida kafolatlanadi.

Testability nuqtai nazaridan: business logic `InMemoryRepo` bilan unit test'lanadi (DB kerakmas, tez), `PostgresRepo` esa alohida integration test bilan tekshiriladi.

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
      Object.entries(filter).every(
        ([key, value]) => (item as Record<string, unknown>)[key] === value
      )
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

Result (Rust'da `Result<T, E>`, Haskell/F#'da `Either<L, R>`) — xatolarni return value sifatida explicit qaytaradigan pattern. Funksiya `throw` qilish o'rniga `{ ok: true, value: T } | { ok: false, error: E }` discriminated union qaytaradi. Caller `ok` flag'ni tekshirmasdan `value`'ga murojaat qila olmaydi — bu compile-time darajasida kafolatlanadi (TypeScript narrowing).

`throw`'ga nisbatan asosiy farq'lar:
- **Error type signature'da** — `function parseJSON<T>(json: string): Result<T, SyntaxError>` — qaysi xato turi bo'lishi mumkinligi return type'da ko'rinadi. `throw`'da bu signature'da yo'q (TS'da `throws` annotation yo'q), faqat hujjat orqali bilinadi.
- **Forgetting impossible** — `ok` tekshirsiz `value`'ga kirib bo'lmaydi (compile error). `try/catch`'ni unutsa kod silently davom etadi.
- **Performance** — `throw` exception ko'tarilganda stack trace yig'ish va stack unwind xarajati bor; Result oddiy object allocation. Bu farq hot path'da (sekundiga ko'p marta ishlaydigan kod) seziladi, kamdan-kam yo'lda ahamiyatsiz.
- **Composability** — `map`/`flatMap` (monad operation'lari) bilan chaining qulay.

Kamchiliklari: JavaScript runtime'i va kutubxonalari `throw` ga asoslangan — `JSON.parse` exception qaytaradi, `fetch` reject Promise qaytaradi. Result pattern'ni qabul qilish — chegara qatlamida (`try/catch` ↔ `Result` conversion) wrapper yozish kerak. Aralash kod (bir joyda `throw`, boshqa joyda `Result`) chalkashlik beradi — bitta yondashuvga rioya qilish muhim.

TypeScript ekosistemasida `neverthrow`, `ts-results`, `fp-ts` kutubxonalari product-grade Result/Either implementation'larini taqdim etadi (monad method'lar, async variant, pipe utility'lar bilan).

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
  private static instance: Config | undefined;
  private constructor(public debug: boolean) {}
  static getInstance() { return this.instance ??= new Config(false); }
}

// Test 1 — debug = true qiladi
Config.getInstance().debug = true;

// Test 2 — BIR XIL INSTANCE! debug hali true
// ❌ Test'lar bir-birini buzadi

// Yechim: DI ishlatish yoki test'da reset qilish
```

### 2. Builder pattern — `build()`'da runtime error vs compile-time error

```typescript
// Fluent builder — majburiy maydon unutilsa xato faqat runtime'da chiqadi
class Builder {
  private url?: string;
  build(): string {
    if (!this.url) throw new Error("url required"); // Runtime ❌
    return this.url;
  }
}
new Builder().build(); // tsc qabul qiladi, lekin runtime'da throw

// Step builder — url() chaqirilmasa build() umuman mavjud emas
interface NeedsUrl { url(u: string): { build(): string }; }
function request(): NeedsUrl { /* ... */ return null as unknown as NeedsUrl; }
// request().build(); // ❌ — Property 'build' does not exist on type 'NeedsUrl' (compile error)
```

### 3. Observer — memory leak (listener tozalanmasa)

```typescript
const emitter = new TypedEmitter<AppEvents>();
const handler = (userId: string) => console.log(userId);

// ❌ — listener qo'shiladi, lekin hech qachon olib tashlanmaydi
function setupPageLeaky() {
  emitter.on("userLoggedIn", handler);
  // Sahifa yopilganda off() chaqirilmasa — leak
}

// ✅ — cleanup/dispose pattern: setup cleanup funksiyasini qaytaradi
function setupPage() {
  emitter.on("userLoggedIn", handler);
  return () => emitter.off("userLoggedIn", handler);
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
declare function readConfig(): string; // throw qilishi mumkin

function riskyFn(): Result<string, Error> {
  const config = readConfig(); // ❌ readConfig throw qilsa — Result emas, exception ko'tariladi
  // Signature Result<string, Error> va'da qiladi, lekin throw bu va'dani buzadi.
  // Caller `if (result.ok)` bilan ishlaydi, exception'ni kutmaydi — silently buziladi.
  return ok(config);
}
```

---

## Common Mistakes

### ❌ Xato 1: Factory'da type narrowing'ni unutish

```typescript
type AnyNotification = NotificationMap[keyof NotificationMap];

// ❌ — return type keng (union) — client qo'shimcha narrowing yozishi kerak
declare function createNotificationWide(type: string): AnyNotification;

// ✅ — generic + indexed access — return type aniq bitta turga toraytiriladi
declare function createNotification<K extends keyof NotificationMap>(type: K): NotificationMap[K];
```

### ❌ Xato 2: Singleton'ni DI o'rniga global state sifatida ishlatish

```typescript
// ❌ — test qilish qiyin, global state
const db = DatabaseConnection.getInstance();

// ✅ — DI orqali inject
class UserService {
  constructor(private db: IDatabaseConnection) {} // Test'da mock berish mumkin
}
```

### ❌ Xato 3: Observer'da event name typo tutilmasligi

```typescript
// ❌ — string literal, typo tutilmaydi
emitter.on("userLogedIn", handler); // Typo! "Logged" emas "Loged"

// ✅ — TypedEmitter — faqat mavjud event'lar
emitter.on("userLoggedIn", handler); // ✅ — autocomplete bor
// emitter.on("userLogedIn", handler); // ❌ — compile error
```

### ❌ Xato 4: Builder'da immutable qoidani buzish

```typescript
// ❌ — bitta builder instance'ni qayta ishlatish
const shared = new RequestBuilder().url("/api");
const badGet = shared.method("GET").build();
const badPost = shared.method("POST").build();
// badGet va badPost bir xil internal request object'ini share qiladi —
// ikkinchi method() birinchisining holatini ustiga yozadi

// ✅ — har safar yangi builder
const getReq = new RequestBuilder().url("/api").method("GET").build();
const postReq = new RequestBuilder().url("/api").method("POST").build();
```

### ❌ Xato 5: State machine'da exhaustive check qilmaslik

```typescript
// ❌ — yangi status qo'shilganda switch'da handle qilmaslik
function renderBad(order: OrderState) {
  switch (order.status) {
    case "draft": return "Draft";
    case "placed": return "Placed";
    // "paid", "shipped", "cancelled" UNUTILDI — tsc indamaydi, return type string | undefined
  }
}

// ✅ — exhaustive check: yangi status qo'shilsa never'ga tayinlash compile error beradi
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

const circle = createShape("circle", { radius: 5 }); // Circle
const square = createShape("square", { side: 10 });  // Square
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
    let set = this.listeners.get(event);
    if (!set) {
      set = new Set();
      this.listeners.set(event, set);
    }
    set.add(fn);
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

**Savol:** `Result<T, E>` type va `map`, `flatMap` method'larini yozing.

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

// Chaining — flatMap ikkala tarafda bir xil E talab qiladi.
// parseJSON E = SyntaxError beradi, shuning uchun avval error'ni string'ga map qilamiz:
const parsed = parseJSON<{ age: number }>('{"age": 25}');
const stringErr: Result<{ age: number }, string> =
  parsed.ok ? parsed : err(parsed.error.message);

const result = flatMap(
  stringErr,
  (user) => user.age >= 18 ? ok(user) : err("Too young")
); // Result<{ age: number }, string>
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

Bu bo'limda TypeScript'da design pattern'larning type-safe implementation'larini o'rgandik:

- **Creational** — Factory (generic map), Abstract Factory (product families), Singleton (private constructor), Builder (fluent API, step builder)
- **Structural** — Adapter (interface conversion), Facade (complexity yashirish), Proxy (caching, access control)
- **Behavioral** — Observer (typed EventEmitter), Strategy (interface + DI), Command (undo/redo), State Machine (discriminated unions)
- **Cross-cutting** — Repository (generic CRUD), Result/Either (`{ ok: true; value: T } | { ok: false; error: E }`)

TypeScript'ning type system'i — generics, discriminated unions, conditional types — design pattern'larni **compile-time type safety** darajasiga ko'taradi.

**Bog'liq bo'limlar:**
- [Bo'lim 5: Union/Intersection](05-unions-intersections.md) — discriminated unions (state machine)
- [Bo'lim 8: Generics](08-generics.md) — generic patterns
- [Bo'lim 20: Reflect Metadata va DI](20-reflect-metadata-di.md) — IoC, DI container

---

**Keyingi bo'lim:** [22-tsconfig.md](22-tsconfig.md) — tsconfig.json Mastery: compiler options, strict mode, module resolution, project references, va barcha configuration imkoniyatlari.
