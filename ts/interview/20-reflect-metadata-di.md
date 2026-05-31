# Interview: Reflect Metadata va Dependency Injection

> Reflect Metadata API, `emitDecoratorMetadata`, `design:type`/`design:paramtypes`/`design:returntype`, Dependency Injection (DI), IoC, injection turlari (constructor/property/method), Singleton/Transient scope, DI Container noldan yozish, circular dependency, testing va decorator-based vs function-based DI bo'yicha interview savollari.

---

## Mundarija

### Nazariy savollar
1. [Reflect Metadata API nima va nima uchun kerak?](#savol-1-reflect-metadata-api-nima-va-nima-uchun-kerak-middle) [Middle]
2. [`emitDecoratorMetadata` qanday ishlaydi?](#savol-2-emitdecoratormetadata-qanday-ishlaydi-middle) [Middle]
3. [Dependency Injection va IoC nima?](#savol-3-dependency-injection-va-ioc-nima-junior) [Junior+]
4. [Constructor vs Property vs Method Injection?](#savol-4-constructor-vs-property-vs-method-injection-middle) [Middle]
5. [Singleton va Transient scope farqi?](#savol-5-singleton-va-transient-scope-farqi-middle) [Middle]
6. [Interface DI muammo: nima uchun token kerak?](#savol-6-interface-di-muammo-nima-uchun-token-kerak-middle) [Middle+]
7. [Decorator-based vs Function-based DI?](#savol-7-decorator-based-vs-function-based-di-middle) [Middle+]
8. [Circular dependency qanday hal qilinadi?](#savol-8-circular-dependency-qanday-hal-qilinadi-senior) [Senior]
9. [`useFactory` provider qachon kerak?](#savol-9-usefactory-provider-qachon-kerak-middle) [Middle+]

### Output savollar
10. [`design:type` metadata output](#savol-10-designtype-metadata-output-middle) [Middle]
11. [`design:paramtypes` interface bilan output](#savol-11-designparamtypes-interface-bilan-output-middle) [Middle]
12. [Auto-resolve container output](#savol-12-auto-resolve-container-output-middle) [Middle+]

### Coding savollar
13. [Sodda DI Container yozing](#savol-13-sodda-di-container-yozing-senior) [Senior]
14. [Scope support (singleton/transient)](#savol-14-scope-support-singletontransient-middle) [Middle+]
15. [Token-based `@inject` decorator](#savol-15-token-based-inject-decorator-middle) [Middle+]
16. [DI bilan testing — mock inject](#savol-16-di-bilan-testing--mock-inject-middle) [Middle+]

### Bug fix savollar
17. [`reflect-metadata` import order xato](#savol-17-reflect-metadata-import-order-xato-middle) [Middle]
18. [Singleton bo'lib qolgan transient](#savol-18-singleton-bolib-qolgan-transient--bug-toping-middle) [Middle+]
19. [`@inject` decorator metadata yo'qotilmoqda](#savol-19-inject-decorator-metadata-yoqotilmoqda--bug-toping-senior) [Senior]

---

## Nazariy savollar

### Savol 1: Reflect Metadata API nima va nima uchun kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Reflect Metadata API — `reflect-metadata` polyfill orqali object lar va class member larga **runtime metadata** biriktirish va o'qish mexanizmi. WeakMap asosida ishlaydi. Decorator-based framework lar (NestJS, TypeORM, Angular) ning asosiy fundamenti.

### To'liq tushuntirish

**Nima uchun kerak:**

TypeScript type information compile-time da o'chiriladi (type erasure). Lekin DI container, ORM, validation library lar runtime da type ma'lumotini bilishi kerak — masalan, `UserService` constructor `Logger` va `Database` ga muhtoj ekanini. Reflect Metadata bu ma'lumotni runtime ga ko'chirish imkonini beradi.

**Tarixiy holat:**
- `reflect-metadata` (Ron Buckton) — Metadata Reflection API ning prototype i. Bu API hech qachon TC39 stage jarayonidan o'tmagan va standartlashtirish uchun ko'rib chiqilmaydi
- O'rniga TC39 Decorator Metadata (`Symbol.metadata`, Stage 3) keldi — metadata decorator ning `context.metadata` object i orqali yoziladi
- `reflect-metadata` package — legacy decorator (`experimentalDecorators`) + `emitDecoratorMetadata` uchun de-facto polyfill bo'lib qoladi

**Asosiy API:**

| Method | Vazifasi |
|--------|---------|
| `Reflect.defineMetadata(key, value, target)` | Class ga metadata yozish |
| `Reflect.defineMetadata(key, value, target, propertyKey)` | Class member ga metadata yozish |
| `Reflect.getMetadata(key, target)` | Metadata o'qish (prototype chain bo'ylab) |
| `Reflect.getOwnMetadata(key, target)` | Faqat o'z metadata (inherited emas) |
| `Reflect.hasMetadata(key, target)` | Metadata mavjudligini tekshirish |
| `Reflect.deleteMetadata(key, target)` | Metadata o'chirish |
| `Reflect.getMetadataKeys(target)` | Barcha metadata key larni olish |

**Ichki implementation:**
Global `WeakMap<target, Map<propertyKey, Map<metadataKey, value>>>` strukturasi. Har target uchun alohida metadata Map saqlanadi. WeakMap — class garbage collect bo'lganda metadata avtomatik tozalanadi.

### Kod misol

```typescript
import "reflect-metadata";

// === Manual metadata ===
class UserService {
  findAll(): string[] { return ["user1"]; }
}

Reflect.defineMetadata("role", "service", UserService);
Reflect.defineMetadata("http:method", "GET", UserService.prototype, "findAll");

console.log(Reflect.getMetadata("role", UserService));
// "service"

console.log(Reflect.getMetadata("http:method", UserService.prototype, "findAll"));
// "GET"


// === Decorator orqali metadata — routing system ===
function controller(prefix: string) {
  return function (constructor: new (...args: any[]) => any) {
    Reflect.defineMetadata("prefix", prefix, constructor);
  };
}

function httpGet(path: string) {
  return function (target: any, propertyKey: string) {
    Reflect.defineMetadata("http:method", "GET", target, propertyKey);
    Reflect.defineMetadata("http:path", path, target, propertyKey);
  };
}

@controller("/api/users")
class UserController {
  @httpGet("/")
  list() { return []; }

  @httpGet("/:id")
  get() { return {}; }
}

// Router framework metadata ni o'qib endpoint larni register qiladi
const prefix = Reflect.getMetadata("prefix", UserController);
console.log(prefix); // "/api/users"
```

### Edge Cases

- **`getMetadata` vs `getOwnMetadata`** — birinchi prototype chain ni ko'radi (inherited), ikkinchi faqat target ning o'z metadata si
- **`reflect-metadata` import — app entry da bir marta** — har faylda import qilish kerak emas, lekin barcha kod dan oldin chaqirilishi shart
- **Inheritance:** child class ga metadata yozsa, parent metadata override qilinmaydi (alohida storage). Lekin `getMetadata` parent dan ham o'qiy oladi
- **WeakMap auto-cleanup** — class reference yo'qolsa, metadata garbage collect. Lekin `target.prototype` reference bo'lsa, qoladi
- **Browser ga deployment:** `reflect-metadata` polyfill bundle ga qo'shiladi — production size impact

### Follow-up savollar

1. **"`Symbol.metadata` `reflect-metadata` ni almashtirib bo'ladimi?"** — TC39 decorator bilan — ha. Legacy decorator + `emitDecoratorMetadata` ishlatadigan loyihalar hali `reflect-metadata` ga muhtoj
2. **"Babel orqali metadata emit qilish mumkinmi?"** — `@babel/plugin-transform-typescript` `emitDecoratorMetadata` flag ni qo'llab-quvvatlamaydi (faqat type annotation larni strip qiladi). Metadata emit uchun `babel-plugin-transform-typescript-metadata` yoki TypeScript compiler ishlatish kerak

</details>

---

### Savol 2: `emitDecoratorMetadata` qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`emitDecoratorMetadata: true` — TypeScript compiler decorate qilingan element lar uchun avtomatik ravishda 3 ta type metadata key emit qiladi: `design:type`, `design:paramtypes`, `design:returntype`. Faqat **legacy decorators** (`experimentalDecorators: true`) bilan ishlaydi.

### To'liq tushuntirish

**3 ta avtomatik metadata key:**

| Key | Nima saqlaydi | Qayerda emit qilinadi |
|-----|---------------|----------------------|
| `"design:type"` | Property yoki parameter type (single constructor ref) | Property decorator |
| `"design:paramtypes"` | Constructor yoki method parameter type lari (array) | Class yoki method decorator |
| `"design:returntype"` | Method return type | Method decorator |

**Muhim shart:** element decorate qilingan bo'lishi kerak. `@injectable()` (yoki har qanday decorator) bo'lmasa — compiler metadata emit qilmaydi.

**Compiler nima qiladi:**

```typescript
// Source:
@injectable
class UserService {
  constructor(private repo: UserRepository, private logger: Logger) {}
}

// Compiled JS:
var UserService = __decorate([
  injectable,
  __metadata("design:paramtypes", [UserRepository, Logger])
], UserService);
```

`__metadata("design:paramtypes", [...])` — `Reflect.defineMetadata("design:paramtypes", [UserRepository, Logger], target)` ni chaqiradi.

**Cheklovlar (type → runtime value mapping):**

| TypeScript type | Runtime metadata |
|----------------|------------------|
| `string` | `String` |
| `number` | `Number` |
| `boolean` | `Boolean` |
| `Array<T>`, `T[]` | `Array` (element type yo'qoladi) |
| `Date` | `Date` |
| `Promise<T>` | `Promise` (generic argument yo'qoladi) |
| `Map<K, V>`, `Set<T>` | `Map`, `Set` (generic argument yo'qoladi) |
| `interface I {}` | `Object` (interface JS da yo'q) |
| `A \| B` (union) | `Object` |
| `any`, `unknown` | `Object` |
| `void`, `undefined` | `undefined` |
| Class reference | Class constructor |

**Class reference faqat haqiqiy class lar uchun ishlaydi** — interface, type alias, union, generic argument lar yo'qoladi.

### Kod misol

```typescript
import "reflect-metadata";

// Class decorator — design:paramtypes emit qiladi
function injectable(constructor: new (...args: any[]) => any) {}

class Logger { log(msg: string) { console.log(msg); } }
class Database { query(sql: string) { return []; } }

@injectable
class UserService {
  constructor(private logger: Logger, private db: Database) {}
}

// Metadata o'qish:
const paramTypes = Reflect.getMetadata("design:paramtypes", UserService);
console.log(paramTypes);
// [class Logger, class Database]

console.log(paramTypes.map((t: any) => t.name));
// ["Logger", "Database"]


// === Method-level metadata ===
// Method decorator bo'lishi shart — class decorator method ga emit qilmaydi
function traceMethod(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {}

class ApiController {
  @traceMethod
  getUser(id: number): Promise<{ name: string }> {
    return Promise.resolve({ name: "Ali" });
  }
}

Reflect.getMetadata("design:type", ApiController.prototype, "getUser");
// Function — method ning o'zi Function constructor ga map qilinadi

Reflect.getMetadata("design:paramtypes", ApiController.prototype, "getUser");
// [Number] — bitta parameter type li array (id: number)

Reflect.getMetadata("design:returntype", ApiController.prototype, "getUser");
// Promise — Promise<T> dan generic argument yo'qoladi


// === Property metadata ===
// Property decorator bo'lishi shart
function traceProp(target: any, propertyKey: string) {}

class Config {
  @traceProp
  apiUrl: string = "";
}

Reflect.getMetadata("design:type", Config.prototype, "apiUrl");
// String — apiUrl: string ning single constructor reference i
```

### Edge Cases

- **Decorator bo'lmasa metadata yo'q:** `@injectable` ni olib tashlash → `paramTypes` undefined
- **Interface → `Object`:** DI container interface dan haqiqiy class bilmaydi — token system kerak
- **Generic argument yo'qoladi:** `Promise<User>` → `Promise`. Runtime da `User` ni topib bo'lmaydi
- **Circular type:** `class A { constructor(b: B) {} }` va `class B { constructor(a: A) {} }` — birinchi yuklanganda ikkinchisi hali `undefined`. Forward reference muammosi
- **`emitDecoratorMetadata` TC39 da:** **ishlamaydi**. TC39 da `Symbol.metadata` ishlatiladi va u compiler tomonidan automatic emit qilinmaydi — qo'lda yozish kerak
- **`design:paramtypes` array bo'sh:** parameter siz constructor → `[]`

### Follow-up savollar

1. **"Class reference circular bo'lsa nima?"** — Forward declaration: `@Inject(forwardRef(() => User))` pattern (NestJS). Ikki class ni alohida import kerak
2. **"Build size ga qanday ta'sir qiladi?"** — Har decorated class uchun `__metadata` chaqiruvi qo'shiladi. Production bundle da minimal, lekin tree-shaking qiyinroq

</details>

---

### Savol 3: Dependency Injection va IoC nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**DI (Dependency Injection)** — class o'z dependency larini **o'zi yaratmaydi**, **tashqaridan oladi**. **IoC (Inversion of Control)** — control oqimi teskari: object lifecycle ni class emas, framework yoki container boshqaradi. DI — IoC ning implementation laridan biri.

### To'liq tushuntirish

**Muammo DI siz:**

```typescript
class UserService {
  private repo = new PostgresUserRepository("postgres://localhost/db");
  private logger = new FileLogger("/var/log/app.log");

  async findUser(id: number) {
    this.logger.log(`Finding user ${id}`);
    return this.repo.findById(id);
  }
}
```

**Muammolar:**
1. **Tight coupling** — `PostgresUserRepository` va `FileLogger` ga qattiq bog'langan
2. **Testability yo'q** — test da real database va file kerak
3. **Configurability yo'q** — production va development da har xil implementation qilib bo'lmaydi
4. **Single Responsibility buzilishi** — class business logic + dependency creation

**Yechim DI bilan:**

```typescript
class UserService {
  constructor(
    private readonly repo: IUserRepository,
    private readonly logger: ILogger
  ) {}

  async findUser(id: number) {
    this.logger.log(`Finding user ${id}`);
    return this.repo.findById(id);
  }
}

// Production:
new UserService(new PostgresUserRepository(url), new FileLogger(path));

// Test:
new UserService(new InMemoryUserRepository(), new SilentLogger());
```

**DI ning afzalliklari:**

1. **Loose coupling** — interface ga bog'langan, concrete class ga emas
2. **Testability** — mock inject oson, real I/O kerak emas
3. **Single Responsibility** — class faqat o'z logic ini bajaradi
4. **Configurability** — environment ga ko'ra turli implementation
5. **Dependency Inversion (SOLID D)** — high-level module abstraction ga bog'liq, low-level detail ga emas

**IoC tushunchasi:**

Klassik dasturlash: application kodi library function larini imperative chaqiradi (`library.call()`).
IoC: control oqimi teskari — framework user code ni callback sifatida chaqiradi (Hollywood Principle: "don't call us, we'll call you").

DI — IoC ning bir implementation i. Object creation va dependency wiring ni framework (DI container) bajaradi, application kod faqat dependency ni declare qiladi.

### Kod misol

```typescript
// === Interface va abstraction ===
interface IEmailService {
  sendWelcome(to: string, name: string): Promise<void>;
}

interface IUserRepository {
  save(user: { name: string; email: string }): Promise<{ id: number }>;
  findByEmail(email: string): Promise<{ id: number } | null>;
}

// === Concrete implementations ===
class SmtpEmailService implements IEmailService {
  async sendWelcome(to: string, name: string) {
    console.log(`SMTP: Welcome to ${to}, ${name}`);
  }
}

class PostgresUserRepository implements IUserRepository {
  async save(user: { name: string; email: string }) {
    console.log(`INSERT INTO users...`);
    return { id: 1 };
  }
  async findByEmail(email: string) {
    return null;
  }
}

// === Service — abstraction ga bog'liq ===
class RegistrationService {
  constructor(
    private readonly email: IEmailService,
    private readonly users: IUserRepository
  ) {}

  async register(name: string, email: string) {
    const existing = await this.users.findByEmail(email);
    if (existing) throw new Error("Email already registered");

    const user = await this.users.save({ name, email });
    await this.email.sendWelcome(email, name);
    return user;
  }
}

// === Composition root — bitta joyda wire qilish ===
const service = new RegistrationService(
  new SmtpEmailService(),
  new PostgresUserRepository()
);

await service.register("Ali", "ali@example.com");
```

### Edge Cases

- **Service Locator anti-pattern:** `container.get(...)` orqali har joyda dependency olish — DI emas, hidden dependency. Constructor injection afzal
- **Optional dependency:** `cache?: ICache` — null check kerak, lekin partial state imkoniyat
- **Too many dependencies:** constructor 10+ argument — class juda ko'p ish qiladi (SRP buzilgan). Refactor kerak
- **DI Framework overkill:** kichik app uchun manual wiring (composition root) yetarli — framework keraksiz overhead

### Follow-up savollar

1. **"DI va Service Locator farqi?"** — DI: dependency declarative (constructor parameter). Service Locator: imperative (`locator.get(Type)`) — hidden, test qiyin
2. **"SOLID dan qaysi tamoyilga DI bog'liq?"** — Dependency Inversion (D). Abstraction (interface) ga bog'liq, concrete implementation ga emas
3. **"DI Container kerakmi har loyihada?"** — Yo'q. Kichik loyihada manual wiring yetarli. Katta loyihada (50+ service) container faydali

</details>

---

### Savol 4: Constructor vs Property vs Method Injection? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Constructor Injection** — eng xavfsiz, dependency lar majburiy va immutable. **Property Injection** — optional dependency lar uchun, incomplete state xavfi bor. **Method Injection** — per-call dependency, har chaqiruvda turli implementation.

### To'liq tushuntirish

**1. Constructor Injection (afzal):**

Dependency lar constructor parametri orqali beriladi. Class instance yaratilganda barcha dependency lar majburiy mavjud.

**Afzalliklari:**
- Class **hech qachon incomplete state** da bo'lmaydi
- Dependency lar `readonly` qilinishi mumkin (immutability)
- Test da mock berish oson
- Type system to'liq tekshiradi (compile-time error)

**Kamchiliklari:**
- Constructor argument soni o'sib ketishi mumkin (ko'p dependency bo'lsa)
- Inheritance da `super(...)` kerak

**2. Property Injection:**

Dependency class property ga assign qilinadi (constructor da emas, keyin).

**Afzalliklari:**
- Optional dependency lar qulay
- Constructor signature ni shishirmaydi

**Kamchiliklari:**
- Class **incomplete state** da bo'lishi mumkin (dependency hali set bo'lmagan)
- `!` (definite assignment) yoki `?` (optional) ishlatish kerak
- DI framework dependency set qilishini kafolatlamaydi

**3. Method Injection:**

Dependency method parametri orqali har chaqiruvda beriladi.

**Afzalliklari:**
- Per-call dependency (har chaqiruvda turli implementation)
- Strategy pattern uchun ideal

**Kamchiliklari:**
- Har chaqiruvda dependency yaratish/berish kerak
- Repeat code

### Kod misol

```typescript
// === 1. Constructor Injection (eng xavfsiz) ===
interface IUserRepository { findById(id: number): Promise<{ name: string } | null>; }
interface ILogger { log(msg: string): void; }

class UserService {
  constructor(
    private readonly repo: IUserRepository,
    private readonly logger: ILogger
  ) {}

  async getUser(id: number) {
    this.logger.log(`Get user ${id}`);
    return this.repo.findById(id);
  }
}

const repo: IUserRepository = { findById: async (id) => ({ name: "Ali" }) };
const logger: ILogger = { log: console.log };
new UserService(repo, logger); // ✅ Barcha dep tayyor


// === 2. Property Injection (optional dep lar uchun) ===
interface ICacheService { get(key: string): unknown; set(key: string, v: unknown): void; }

class NotificationService {
  cache?: ICacheService;  // Optional — bo'lmasa ham ishlaydi

  notify(userId: number) {
    if (this.cache) {
      const cached = this.cache.get(`user:${userId}`);
      if (cached) return cached;
    }
    return { userId, message: "notification" };
  }
}

const notif = new NotificationService();
notif.cache = { get: () => null, set: () => {} }; // Keyinroq inject


// === 3. Method Injection (per-call dep) ===
interface IFormatter { format(data: unknown[]): string; }

class ReportGenerator {
  generate(data: unknown[], formatter: IFormatter): string {
    return formatter.format(data);
  }
}

const csvFormatter: IFormatter = { format: (d) => d.map(String).join(",") };
const jsonFormatter: IFormatter = { format: (d) => JSON.stringify(d) };

const gen = new ReportGenerator();
gen.generate([1, 2, 3], csvFormatter);  // "1,2,3"
gen.generate([1, 2, 3], jsonFormatter); // "[1,2,3]"
```

### Edge Cases

- **Constructor + property mix:** ko'p framework optional dep ni property orqali, majburiy constructor orqali (NestJS `@Inject` parameter + property `@Inject`)
- **`!` assertion xavfi:** `private logger!: ILogger` — TS ga "menga ishon" deyish, lekin runtime da set bo'lmagan bo'lishi mumkin → undefined error
- **Circular dependency va injection:** constructor injection da circular dep stack overflow. Property/method injection bilan lazy load mumkin
- **Inheritance:** child class constructor parent dep lar uchun `super(...)` chaqirishi kerak
- **Method injection callback hell:** har method chaqiruvida dep berish — boilerplate. Class darajasidagi dep larni constructor ga ko'chiring

### Follow-up savollar

1. **"NestJS qaysi injection turini ishlatadi?"** — Constructor injection — default. Property injection ham bor (`@Inject()` decorator)
2. **"Optional dependency qanday berish?"** — Constructor da `private cache?: ICache = undefined`, yoki property + `?`
3. **"Setter injection nima?"** — Property injection ning variant — setter method orqali (`setLogger(logger: ILogger)`). Lekin oddiy property afzal

</details>

---

### Savol 5: Singleton va Transient scope farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Singleton scope** — bir instance yaratiladi va har `resolve` da bir xil instance qaytariladi. **Transient scope** — har `resolve` da yangi instance. Stateful service uchun singleton, stateless utility uchun transient.

### To'liq tushuntirish

**Scope — dependency lifecycle:**

| Scope | Hayot davri | Memory | Use case |
|-------|------------|--------|----------|
| **Singleton** | App lifetime | 1 instance | DB connection, config, cache, logger |
| **Transient** | Per-resolve | N instance | Lightweight DTO, request context, stateless utility |
| **Scoped** (advanced) | Per-request (HTTP) | 1 per request | DB transaction, request-scoped context |

**Singleton — qachon kerak:**
- **Costly resource** — database connection, file handler, network socket
- **Shared state** — application config, cache, in-memory store
- **Logger** — bir instance ko'p service tomonidan ishlatiladi

**Transient — qachon kerak:**
- **Stateless service** — pure function lar joylashgan (formatter, validator)
- **Mutation kerak** — har user uchun alohida state
- **Lightweight** — instance yaratish arzon (no I/O)

**Singleton xavf lari:**
- **Global state** — testlar bir-birini buzadi (state leak)
- **Thread safety** (Node.js da single-threaded, lekin worker thread/cluster da muammo)
- **Lifecycle management** — qachon dispose qilish noaniq

### Kod misol

```typescript
type Scope = "singleton" | "transient";

interface Registration {
  cls: new (...args: any[]) => any;
  scope: Scope;
}

class Container {
  private registrations = new Map<any, Registration>();
  private singletons = new Map<any, any>();

  register<T>(
    token: any,
    cls: new (...args: any[]) => T,
    scope: Scope = "transient"
  ): void {
    this.registrations.set(token, { cls, scope });
  }

  resolve<T>(token: any): T {
    const reg = this.registrations.get(token);
    if (!reg) throw new Error(`No registration: ${String(token)}`);

    if (reg.scope === "singleton" && this.singletons.has(token)) {
      return this.singletons.get(token);
    }

    const instance = new reg.cls();

    if (reg.scope === "singleton") {
      this.singletons.set(token, instance);
    }
    return instance;
  }
}

// === Foydalanish ===
class Logger { id = Math.random(); log(msg: string) { console.log(`[${this.id}] ${msg}`); } }
class Formatter { format(d: unknown) { return JSON.stringify(d); } }

const container = new Container();
container.register("Logger", Logger, "singleton");
container.register("Formatter", Formatter, "transient");

const a = container.resolve<Logger>("Logger");
const b = container.resolve<Logger>("Logger");
console.log(a === b); // true — singleton

const x = container.resolve<Formatter>("Formatter");
const y = container.resolve<Formatter>("Formatter");
console.log(x === y); // false — transient
```

### Edge Cases

- **Singleton + test isolation:** har test da yangi container — singleton state previous test dan kelmasligi uchun
- **Transient ichida singleton dep:** transient service singleton dep ga muhtoj bo'lsa — singleton bir xil, transient har xil
- **Singleton da mutable state:** concurrent request lar shared state ni buzishi mumkin. Read-only state afzal
- **Scoped (request) scope:** HTTP request lifecycle ga bog'langan. NestJS `@Injectable({ scope: Scope.REQUEST })`
- **Memory leak:** singleton uzoq yashaydi — agar listener register qilsa, dispose kerak

### Follow-up savollar

1. **"Nima uchun singleton anti-pattern deb aytiladi?"** — Global state, test coupling. Lekin DI container ichidagi singleton — controlled (container instance ichida cheklangan), anti-pattern emas
2. **"Per-request scope NestJS da qanday ishlaydi?"** — Har HTTP request uchun yangi scope, request tugagach dispose
3. **"Singleton va static class farqi?"** — Singleton instance — DI bilan inject mumkin, interface implement qila oladi. Static class — global, polymorphism yo'q

</details>

---

### Savol 6: Interface DI muammo: nima uchun token kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript interface compile-time da o'chiriladi — runtime da mavjud emas. `emitDecoratorMetadata` interface ni `Object` deb yozadi. DI container qaysi concrete class inject qilishni bilmaydi. Yechim: explicit token (Symbol yoki string) + `@Inject(TOKEN)`.

### To'liq tushuntirish

**Muammo:**

```typescript
interface ILogger { log(msg: string): void; }

@injectable
class AppService {
  constructor(private logger: ILogger) {}
}

const paramTypes = Reflect.getMetadata("design:paramtypes", AppService);
console.log(paramTypes); // [Object] — ILogger emas!
```

`ILogger` JS ga compile bo'lganda o'chiriladi. `design:paramtypes` da uning o'rniga `Object` yoziladi — compiler runtime da topib bo'lmaydigan type uchun `Object` ni fallback sifatida emit qiladi. DI container `Object` ga qarab qaysi implementation kerakligini bilmaydi.

**Yechim 1: Class ishlatish (interface o'rniga abstract class):**

```typescript
abstract class Logger {
  abstract log(msg: string): void;
}

class ConsoleLogger extends Logger {
  log(msg: string) { console.log(msg); }
}

@injectable
class AppService {
  constructor(private logger: Logger) {}  // Class reference — runtime da mavjud
}

container.register(Logger, ConsoleLogger);
// design:paramtypes → [Logger] — to'g'ri
```

**Yechim 2: Token system (Symbol + `@Inject`):**

```typescript
const LOGGER_TOKEN = Symbol("Logger");

function inject(token: any) {
  return function (target: any, _key: string | undefined, paramIndex: number) {
    const tokens: Map<number, any> =
      Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    tokens.set(paramIndex, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}

@injectable
class AppService {
  constructor(@inject(LOGGER_TOKEN) private logger: ILogger) {}
}

container.bind(LOGGER_TOKEN).to(ConsoleLogger);
// Container — LOGGER_TOKEN orqali ConsoleLogger ni topadi
```

**Yechim 3: String token (kamroq afzal — typo xavfi):**

```typescript
@injectable
class AppService {
  constructor(@inject("ILogger") private logger: ILogger) {}
}

container.register("ILogger", ConsoleLogger);
```

**Yechim 4: TC39 + `Symbol.metadata` (zamonaviy):**

```typescript
const tokens = { Logger: Symbol("Logger") } as const;

function inject(token: symbol) {
  return function (
    originalMethod: any,
    context: ClassMethodDecoratorContext | ClassFieldDecoratorContext
  ) {
    // Metadata yozish
  };
}
```

### Kod misol

```typescript
import "reflect-metadata";

// === Token-based DI ===
interface IUserRepository {
  findById(id: number): Promise<{ name: string } | null>;
}

interface IEmailService {
  send(to: string, body: string): Promise<void>;
}

const TOKENS = {
  UserRepository: Symbol("UserRepository"),
  EmailService: Symbol("EmailService"),
} as const;

// Parameter decorator (legacy) — token ni metadata ga
function inject(token: any) {
  return function (target: any, _: any, index: number) {
    const tokens: Map<number, any> =
      Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    tokens.set(index, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}

function injectable(constructor: new (...args: any[]) => any) {}

// Class — interface ga bog'liq, lekin token bilan resolve qilinadi
@injectable
class UserService {
  constructor(
    @inject(TOKENS.UserRepository) private repo: IUserRepository,
    @inject(TOKENS.EmailService) private email: IEmailService
  ) {}

  async welcome(id: number) {
    const user = await this.repo.findById(id);
    if (user) await this.email.send("ali@x.com", `Welcome, ${user.name}`);
  }
}

// Concrete implementations
class PostgresUserRepo implements IUserRepository {
  async findById(id: number) { return { name: "Ali" }; }
}

class SmtpEmailService implements IEmailService {
  async send(to: string, body: string) { console.log(`Send to ${to}`); }
}

// Container — token → implementation
class Container {
  private bindings = new Map<any, any>();

  bind(token: any) {
    return {
      to: (cls: new (...args: any[]) => any) => {
        this.bindings.set(token, cls);
      },
    };
  }

  resolve<T>(target: new (...args: any[]) => T): T {
    const tokens: Map<number, any> =
      Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", target) || [];

    const deps = paramTypes.map((type, i) => {
      const token = tokens.get(i) ?? type;
      const ImplCls = this.bindings.get(token);
      if (!ImplCls) throw new Error(`No binding: ${String(token)}`);
      return this.resolve(ImplCls);
    });

    return new target(...deps);
  }
}

const container = new Container();
container.bind(TOKENS.UserRepository).to(PostgresUserRepo);
container.bind(TOKENS.EmailService).to(SmtpEmailService);
container.bind(UserService).to(UserService);

const service = container.resolve(UserService);
await service.welcome(1);
```

### Edge Cases

- **Token uniqueness:** Symbol unique — collision yo'q. String token — typo va naming collision xavfi
- **Const assertion:** `TOKENS = { ... } as const` — `keyof typeof TOKENS` orqali type-safe
- **Token va class bir vaqtda:** abstract class — runtime da class reference, lekin abstract method orqali interface contract beradi. Ikki vazifani bajaradi
- **`@Inject` decorator bo'lmasa:** container `design:paramtypes` ga qaraydi — class reference bo'lsa ishlaydi, interface bo'lsa `Object` — error
- **TC39 da parameter decorator yo'q:** token tizimini class-level metadata yoki object parameter pattern bilan implement qilish kerak

### Follow-up savollar

1. **"InversifyJS qanday ishlaydi?"** — Symbol token + `@injectable()` + `@inject(token)` parameter decorator. NestJS, type-safe DI library lar shu pattern asosida
2. **"Type safety nima bo'ladi?"** — Token va class type alohida — runtime check, compile-time emas. TypeScript `keyof typeof TOKENS` bilan partial safety
3. **"Generic interface qanday inject qilinadi?"** — `IRepository<User>` — generic argument yo'qoladi. Bir token per generic instantiation kerak (`USER_REPO_TOKEN`, `ORDER_REPO_TOKEN`)

</details>

---

### Savol 7: Decorator-based vs Function-based DI? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Decorator-based DI** — `@injectable`, `@inject`, `reflect-metadata` ga bog'liq. NestJS, Angular standartlari. **Function-based DI** — oddiy funksiya/factory orqali dependency wiring, decorator va metadata kerak emas. TC39 mos, tree-shakeable, modern trend.

### To'liq tushuntirish

| Xususiyat | Decorator-based | Function-based |
|-----------|----------------|----------------|
| Decorator kerak | `@injectable`, `@inject` | Yo'q |
| Runtime overhead | `reflect-metadata` polyfill | Yo'q |
| TC39 mos | Yo'q (legacy faqat) | Ha |
| Type safety | Token-based partial | Natural (TS type system) |
| Tree-shaking | Qiyinroq | Osonroq |
| Boilerplate | Decorator + token | Factory funksiya |
| Configuration | Container API | Composition root |
| Use case | Enterprise (NestJS, Angular) | Modern (tRPC, Hono, Effect) |
| Test setup | Mock container | Manual wiring |
| Learning curve | Steep | Flat |

**Decorator-based misol:**

```typescript
import "reflect-metadata";

@injectable()
class UserService {
  constructor(
    @inject(LOGGER_TOKEN) private logger: ILogger,
    @inject(DB_TOKEN) private db: IDatabase
  ) {}
}

container.bind(LOGGER_TOKEN).to(ConsoleLogger);
container.bind(DB_TOKEN).to(PostgresDB);

const service = container.resolve(UserService);
```

**Function-based misol:**

```typescript
// Factory function — explicit dependency
function createUserService(logger: ILogger, db: IDatabase) {
  return {
    async getUser(id: number) {
      logger.log(`Get user ${id}`);
      return db.query(`SELECT * FROM users WHERE id = ${id}`);
    },
  };
}

// Composition root
const logger = createConsoleLogger();
const db = createPostgresDB({ url: "..." });
const userService = createUserService(logger, db);
```

**Function-based — Effect/fp-ts pattern:**

```typescript
import { Effect, Context, Layer } from "effect";

class Logger extends Context.Tag("Logger")<Logger, { log: (msg: string) => void }>() {}
class Database extends Context.Tag("Database")<Database, { query: (sql: string) => unknown }>() {}

const program = Effect.gen(function* () {
  const logger = yield* Logger;
  const db = yield* Database;
  logger.log("Querying...");
  return yield* Effect.promise(() => db.query("SELECT 1"));
});

const ConsoleLogger = Layer.succeed(Logger, { log: console.log });
const RealDb = Layer.succeed(Database, { query: () => [] });

Effect.runPromise(program.pipe(Effect.provide(Layer.merge(ConsoleLogger, RealDb))));
```

**Trade-off analizi:**

**Decorator-based afzalliklari:**
- Declarative — class ichida declaration
- Framework integration (NestJS, Angular) bilan mukammal
- Metadata orqali advanced feature lar (auto-routing, validation)

**Decorator-based kamchiliklari:**
- `reflect-metadata` polyfill majburiy (production size)
- Tree-shaking qiyinroq (decorator side-effect heavy)
- TC39 ga migration complex
- Magic — debugging qiyinroq

**Function-based afzalliklari:**
- Zero runtime overhead
- TC39 fully compatible
- TypeScript type system to'liq ishlaydi
- Test setup oddiyroq

**Function-based kamchiliklari:**
- Composition root da boilerplate
- Large app uchun manual wiring repetitive — composition root o'sib ketadi
- IDE autocomplete kichikroq qulaylik

### Kod misol

```typescript
// === To'liq function-based DI tizim ===

// 1. Domain — interface lar
interface ILogger { log(msg: string): void; }
interface IUserRepo { findById(id: number): Promise<{ name: string } | null>; }
interface IEmailSender { send(to: string, body: string): Promise<void>; }

// 2. Implementations — factory funksiya lar
function createConsoleLogger(): ILogger {
  return { log: (msg) => console.log(`[LOG] ${msg}`) };
}

function createPostgresUserRepo(connectionUrl: string): IUserRepo {
  return {
    findById: async (id) => ({ name: `User ${id}` }),
  };
}

function createSmtpEmail(host: string): IEmailSender {
  return {
    send: async (to, body) => console.log(`SMTP ${host}: ${to}`),
  };
}

// 3. Service — factory + injected dependencies
function createUserService(deps: {
  logger: ILogger;
  repo: IUserRepo;
  email: IEmailSender;
}) {
  return {
    async welcome(id: number) {
      const user = await deps.repo.findById(id);
      if (!user) throw new Error("Not found");
      deps.logger.log(`Welcoming ${user.name}`);
      await deps.email.send("ali@x.com", `Hello ${user.name}`);
    },
  };
}

// 4. Composition root — single wiring point
function createApp(config: { dbUrl: string; smtpHost: string }) {
  const logger = createConsoleLogger();
  const repo = createPostgresUserRepo(config.dbUrl);
  const email = createSmtpEmail(config.smtpHost);
  const userService = createUserService({ logger, repo, email });
  return { userService };
}

const app = createApp({ dbUrl: "postgres://...", smtpHost: "smtp.example.com" });
await app.userService.welcome(1);


// 5. Test — easy mocking
const testApp = {
  userService: createUserService({
    logger: { log: () => {} },
    repo: { findById: async () => ({ name: "Test User" }) },
    email: { send: async () => {} },
  }),
};

await testApp.userService.welcome(1);
```

### Edge Cases

- **Function-based + class:** ikkalasi aralash bo'lishi mumkin — class instance ni factory dan qaytarish
- **Lazy initialization:** function-based da Lazy evaluator (`() => expensive()`) bilan natural
- **Cyclic deps:** function-based da explicit — bir factory boshqaning natijasiga muhtoj bo'lsa, oddiy issue ko'rinadi
- **Plugin architecture:** decorator-based ko'pincha qulayroq — `@Module({ imports: [...], providers: [...] })` declarative
- **Effect/IO monad:** functional pure DI (Effect, fp-ts) — pure function lar uchun ideal, lekin learning curve katta

### Follow-up savollar

1. **"Modern framework lar qaysi yo'lda?"** — Vercel/Next, Hono, Bun, tRPC — function-based. NestJS, Angular hali decorator-based
2. **"TypeScript ga eng yaxshi DI?"** — Function-based + TypeScript type inference — eng natural. Decorator system metadata orqali type information manual maintain qilish kerak
3. **"InversifyJS yaxshimi yoki Effect?"** — InversifyJS — classical OOP DI, NestJS-like. Effect — functional, advanced (effects, errors, dependencies birgalikda)

</details>

---

### Savol 8: Circular dependency qanday hal qilinadi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Circular dependency — A class B ga, B class A ga muhtoj. DI container resolve da infinite loop yoki stack overflow. Yechim: forward reference, lazy injection (factory yoki getter), yoki dependency ni ajratish (refactor).

### To'liq tushuntirish

**Muammo:**

```typescript
@injectable
class A {
  constructor(private b: B) {}
}

@injectable
class B {
  constructor(private a: A) {}
}

container.resolve(A);
// 1. A ni resolve qilish — B kerak
// 2. B ni resolve qilish — A kerak
// 3. A ni resolve qilish — B kerak
// → Stack overflow
```

**Yechimlar:**

**1. Refactoring (eng yaxshi) — dependency ni ajratish:**

Circular dep — odatda **design problem**. Class lar bir-biri bilan juda chambarchas bog'langan. Yechim — uchinchi class yaratish va ikkalasi unga bog'lanadi.

```typescript
// ❌ Circular
class UserService { constructor(private orderService: OrderService) {} }
class OrderService { constructor(private userService: UserService) {} }

// ✅ Refactor — common dep
class UserOrderHelper {
  async getUserOrders(userId: number) { /* ... */ }
}

class UserService { constructor(private helper: UserOrderHelper) {} }
class OrderService { constructor(private helper: UserOrderHelper) {} }
```

**2. Forward Reference (framework-specific):**

NestJS, Angular `forwardRef(() => Class)` orqali. Late binding — class hali undefined bo'lsa ham reference saqlanadi.

```typescript
class A {
  constructor(
    @Inject(forwardRef(() => B)) private b: B
  ) {}
}

class B {
  constructor(
    @Inject(forwardRef(() => A)) private a: A
  ) {}
}
```

**3. Lazy Injection (factory):**

Constructor da to'g'ridan-to'g'ri inject qilmaslik. Buning o'rniga factory yoki getter saqlash — kerak bo'lganda resolve.

```typescript
class A {
  constructor(private getB: () => B) {}

  doSomething() {
    const b = this.getB(); // Lazy resolve
    b.action();
  }
}

class B {
  constructor(private getA: () => A) {}

  action() {
    const a = this.getA();
    // ...
  }
}

// Container — factory bind
container.bind("A").toFactory(() => new A(() => container.resolve("B")));
container.bind("B").toFactory(() => new B(() => container.resolve("A")));
```

**4. Property Injection (lazy by nature):**

Constructor da emas, property orqali. Setter yoki direct assign.

```typescript
class A {
  b!: B; // Late inject

  doSomething() { this.b.action(); }
}

class B {
  a!: A;

  action() { /* ... */ }
}

const a = new A();
const b = new B();
a.b = b;
b.a = a; // Manual wiring
```

**5. Subject/Observable pattern:**

Class ikkinchisiga emas, event/subject ga bog'lansin.

```typescript
class EventBus {
  emit(event: string, data: unknown) { /* ... */ }
  on(event: string, handler: (data: unknown) => void) { /* ... */ }
}

class A {
  constructor(private bus: EventBus) {
    bus.on("from-b", (data) => { /* react */ });
  }
  notify() { this.bus.emit("from-a", { /* ... */ }); }
}

class B {
  constructor(private bus: EventBus) {
    bus.on("from-a", (data) => { /* react */ });
  }
}
```

### Kod misol

```typescript
// === Real circular dependency detection ===
class Container {
  private bindings = new Map<any, new (...args: any[]) => any>();
  private resolving = new Set<any>(); // Circular detection

  bind(token: any, cls: new (...args: any[]) => any) {
    this.bindings.set(token, cls);
  }

  resolve<T>(token: any): T {
    if (this.resolving.has(token)) {
      throw new Error(
        `Circular dependency detected: ${[...this.resolving, token].map(String).join(" -> ")}`
      );
    }

    const cls = this.bindings.get(token);
    if (!cls) throw new Error(`No binding: ${String(token)}`);

    this.resolving.add(token);
    try {
      const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", cls) || [];
      const deps = paramTypes.map((type) => this.resolve(type));
      return new cls(...deps);
    } finally {
      this.resolving.delete(token);
    }
  }
}

// === Lazy resolve container ===
class LazyContainer {
  private bindings = new Map<any, new (...args: any[]) => any>();

  bind<T>(token: any, cls: new (...args: any[]) => T) {
    this.bindings.set(token, cls);
  }

  resolveLazy<T>(token: any): () => T {
    return () => this.resolve(token);
  }

  resolve<T>(token: any): T {
    const cls = this.bindings.get(token);
    if (!cls) throw new Error(`No binding: ${String(token)}`);
    return new cls() as T;
  }
}

// Usage with lazy
class UserService {
  constructor(private getOrderService: () => OrderService) {}

  getOrders(userId: number) {
    const orderService = this.getOrderService();
    return orderService.findByUser(userId);
  }
}

class OrderService {
  constructor(private getUserService: () => UserService) {}

  findByUser(userId: number) {
    return [{ id: 1, userId }];
  }
}
```

### Edge Cases

- **Detection:** container `resolving` Set bilan track qiladi. Stack overflow oldini olib explicit error
- **Singleton + circular:** birinchi resolve da instance qisman tuzilgan bo'lishi mumkin (partial state) — bug source
- **Property injection avoids constructor cycle:** lekin runtime da dependency hali set bo'lmagan instance ishlatilishi xavfli
- **Module-level cycle (import):** `import { B } from "./b"` — ESM module cycle ham circular dep. `b` hali export qilinmagan bo'lishi mumkin
- **Refactor signal:** circular dep — code smell. SOLID Single Responsibility buzilgan bo'lishi ehtimoli yuqori

### Follow-up savollar

1. **"NestJS qanday handle qiladi?"** — `forwardRef(() => Class)` — placeholder reference. Resolve vaqtida actual class ga aylanadi
2. **"Module-level circular import muammosi?"** — ESM da partial undefined export bo'lishi mumkin. Yechim: type-only import yoki refactor
3. **"Mediator pattern circular ni hal qiladimi?"** — Ha, mediator (yoki event bus) — direct dependency ni indirect ga aylantiradi

<details>
<summary><strong>Deep Dive</strong></summary>

**Topological sort approach:**

Dependency graph qurish va topological sort qilish — agar valid order topilsa, cycle yo'q. Topology sort cycle bo'lsa fail beradi (cycle detection algorithm).

```typescript
function topoSort(deps: Map<string, string[]>): string[] | null {
  const visited = new Set<string>();
  const visiting = new Set<string>();
  const order: string[] = [];

  function visit(node: string): boolean {
    if (visiting.has(node)) return false; // cycle
    if (visited.has(node)) return true;

    visiting.add(node);
    for (const dep of deps.get(node) || []) {
      if (!visit(dep)) return false;
    }
    visiting.delete(node);
    visited.add(node);
    order.push(node);
    return true;
  }

  for (const node of deps.keys()) {
    if (!visit(node)) return null; // cycle detected
  }
  return order;
}

const graph = new Map([
  ["A", ["B"]],
  ["B", ["C"]],
  ["C", []],
]);
console.log(topoSort(graph)); // ["C", "B", "A"]

const cyclic = new Map([
  ["A", ["B"]],
  ["B", ["A"]],
]);
console.log(topoSort(cyclic)); // null — cycle
```

Container bu algorithm orqali resolve order topadi yoki cycle aniqlaydi.

</details>

</details>

---

### Savol 9: `useFactory` provider qachon kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useFactory` — instance yaratish logikasi murakkab bo'lganda (config kerak, async initialization, conditional implementation). Constructor injection o'rniga factory funksiya orqali dependency yaratiladi.

### To'liq tushuntirish

**Provider turlari (NestJS/InversifyJS convention):**

| Provider | Vazifasi | Use case |
|----------|---------|----------|
| `useClass` | Class ni resolve qiladi (constructor injection) | Default — ko'p case |
| `useValue` | Ready-made value qaytaradi | Config object, constants |
| `useFactory` | Factory funksiya chaqiradi | Murakkab init, conditional |
| `useExisting` | Boshqa token ni reuse qiladi | Alias |

**`useFactory` qachon kerak:**

1. **Configuration-dependent:** env variable ga ko'ra turli implementation
2. **Async initialization:** DB connection, network setup
3. **Computed dependency:** runtime info ga asoslangan
4. **Optional dependency:** ba'zi holatda dep mavjud, ba'zida null
5. **Conditional logic:** test/prod farq

### Kod misol

```typescript
// === Provider turlari ===
type Provider =
  | { token: any; useClass: new (...args: any[]) => any; scope?: Scope }
  | { token: any; useValue: any }
  | { token: any; useFactory: (container: Container) => any; scope?: Scope }
  | { token: any; useExisting: any };

type Scope = "singleton" | "transient";

class Container {
  private providers = new Map<any, Provider>();
  private singletons = new Map<any, any>();

  register(provider: Provider) {
    this.providers.set(provider.token, provider);
  }

  resolve<T>(token: any): T {
    const p = this.providers.get(token);
    if (!p) throw new Error(`No provider: ${String(token)}`);

    // useValue
    if ("useValue" in p) return p.useValue;

    // useExisting (alias)
    if ("useExisting" in p) return this.resolve(p.useExisting);

    // Singleton cache
    const scope = "scope" in p ? p.scope : "transient";
    if (scope === "singleton" && this.singletons.has(token)) {
      return this.singletons.get(token);
    }

    let instance: any;

    // useFactory
    if ("useFactory" in p) {
      instance = p.useFactory(this);
    }
    // useClass
    else if ("useClass" in p) {
      const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", p.useClass) || [];
      const deps = paramTypes.map((t) => this.resolve(t));
      instance = new p.useClass(...deps);
    }

    if (scope === "singleton") this.singletons.set(token, instance);
    return instance;
  }
}


// === Use cases ===
const container = new Container();
const CONFIG_TOKEN = Symbol("Config");
const DB_TOKEN = Symbol("Database");
const LOGGER_TOKEN = Symbol("Logger");

// 1. useValue — config object
container.register({
  token: CONFIG_TOKEN,
  useValue: {
    dbUrl: process.env.DATABASE_URL || "postgres://localhost",
    debug: process.env.NODE_ENV !== "production",
  },
});

// 2. useFactory — config dan database yaratadi
container.register({
  token: DB_TOKEN,
  useFactory: (c) => {
    const config = c.resolve<{ dbUrl: string }>(CONFIG_TOKEN);
    return createDatabaseConnection(config.dbUrl);
  },
  scope: "singleton",
});

// 3. useFactory — env ga ko'ra turli logger
container.register({
  token: LOGGER_TOKEN,
  useFactory: (c) => {
    const config = c.resolve<{ debug: boolean }>(CONFIG_TOKEN);
    return config.debug ? new VerboseLogger() : new ProductionLogger();
  },
  scope: "singleton",
});


// 4. useFactory — async initialization (Promise return)
interface IEmailService { send(to: string, body: string): Promise<void>; }

async function createEmailService(): Promise<IEmailService> {
  const transporter = await connectSmtpServer({ host: "smtp.example.com" });
  return {
    send: async (to, body) => transporter.send({ to, body }),
  };
}

const EMAIL_TOKEN = Symbol("Email");
container.register({
  token: EMAIL_TOKEN,
  useFactory: () => createEmailService(), // Returns Promise<IEmailService>
  scope: "singleton",
});

// Resolve async — caller `await` kerak
const emailPromise = container.resolve<Promise<IEmailService>>(EMAIL_TOKEN);
const email = await emailPromise;


function createDatabaseConnection(url: string) {
  return { query: (sql: string) => console.log(`[DB ${url}] ${sql}`) };
}

async function connectSmtpServer(opts: { host: string }) {
  return { send: async (msg: any) => {} };
}

class VerboseLogger { log(msg: string) { console.log(`[VERBOSE] ${msg}`); } }
class ProductionLogger { log(msg: string) { console.log(msg); } }
```

### Edge Cases

- **Async factory:** factory `Promise` qaytarishi mumkin — caller resolve da `await` kerak. NestJS `useFactory` async supported
- **Factory ichida circular dependency:** factory `container.resolve(...)` chaqirsa va o'sha token o'zi bo'lsa — infinite loop
- **Factory side effect:** factory side effect berishi xavfli — transient da har resolve da takrorlanadi. Bir martalik side effect uchun `singleton` scope ishlatib, factory ni faqat bir marta chaqirtirish kerak
- **Factory return type:** runtime tekshirish yo'q — yashirin bug source. Type assertion kerak
- **`useExisting` alias:** ikkita token bir xil instance — yangi instance yaratmaydi

### Follow-up savollar

1. **"NestJS asyncProvider qanday ishlaydi?"** — `useFactory` da Promise qaytarsa — NestJS init da await qiladi. Module ready bo'lguncha kutiladi
2. **"Factory ichidagi dependency lar qanday inject qilinadi?"** — Factory parameter sifatida `container` keladi, undan boshqa dep larni resolve qiladi. Yoki `inject: [TOKEN1, TOKEN2]` array (NestJS pattern)

</details>

---

## Output savollar

### Savol 10: `design:type` metadata output [Middle]

**Savol:** Har bir property uchun `"design:type"` metadata nima bo'ladi?

```typescript
import "reflect-metadata";
function track(target: any, key: string) {}

class Demo {
  @track name: string = "";
  @track age: number = 0;
  @track active: boolean = true;
  @track tags: string[] = [];
  @track data: object = {};
  @track callback: () => void = () => {};
  @track createdAt: Date = new Date();
  @track items: Map<string, number> = new Map();
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
name:      String
age:       Number
active:    Boolean
tags:      Array       (element type yo'qoladi)
data:      Object
callback:  Function
createdAt: Date
items:     Map         (generic argument yo'qoladi)
```

### To'liq tushuntirish

`emitDecoratorMetadata: true` paytida compiler har decorate qilingan property uchun runtime constructor function reference saqlaydi.

**Type → Runtime mapping:**

| TypeScript type | Runtime metadata | Tushuntirish |
|----------------|------------------|--------------|
| `string` | `String` | Primitive → wrapper constructor |
| `number` | `Number` | Primitive → wrapper constructor |
| `boolean` | `Boolean` | Primitive → wrapper constructor |
| `bigint` | `BigInt` | Primitive → wrapper constructor |
| `symbol` | `Symbol` | Primitive → wrapper constructor |
| `Array<T>`, `T[]` | `Array` | Generic argument yo'qoladi |
| `object` | `Object` | Generic object |
| `Function`, `() => void` | `Function` | Function constructor |
| `Date` | `Date` | Class reference (runtime mavjud) |
| `Map<K,V>`, `Set<T>` | `Map`, `Set` | Runtime class, generic argument yo'qoladi |
| `Promise<T>` | `Promise` | Class reference, generic yo'qoladi |
| `RegExp` | `RegExp` | Runtime class |
| `interface I` | `Object` | Interface JS da yo'q |
| `type T = ...` | `Object` (yoki primitive) | Type alias erasure |
| Union `A \| B` | `Object` | Common parent |
| Intersection `A & B` | `Object` | Common parent |
| `any`, `unknown` | `Object` | Generic placeholder |
| `void`, `undefined`, `never` | `undefined` | Constructor reference yo'q — metadata qiymati `undefined` |

**Generic erasure:** `string[]` → `Array` (element type qoldirilmaydi). DI uchun bu cheklov — `T[]` dan `T` ni topib bo'lmaydi.

### Kod misol

```typescript
import "reflect-metadata";

function track(target: any, key: string) {}

class Demo {
  @track name: string = "";
  @track age: number = 0;
  @track active: boolean = true;
  @track tags: string[] = [];
  @track data: object = {};
  @track callback: () => void = () => {};
}

const props = ["name", "age", "active", "tags", "data", "callback"];
props.forEach(p => {
  const type = Reflect.getMetadata("design:type", Demo.prototype, p);
  console.log(`${p}: ${type?.name}`);
});

// name: String
// age: Number
// active: Boolean
// tags: Array
// data: Object
// callback: Function
```

### Edge Cases

- **Initial value irrelevant:** `name: string = ""` va `name: string` bir xil metadata
- **Optional property:** `name?: string` → `String` (optional erasure)
- **Union erasure:** `string | number` → `Object` (common parent)
- **Generic class:** `class Container<T> { @track item: T }` → `Object` (T runtime da yo'q)
- **Method type:** `@track method() {}` → `Function`. Method body relevant emas

### Follow-up savollar

1. **"Nima uchun `string[]` da element type yo'qoladi?"** — `Array` JS class. Generic argument compile-time only — runtime da array element type ma'lum emas
2. **"Class self-reference qanday?"** — `class A { @track parent: A }` — runtime da A class reference. Lekin forward reference muammosi bo'lishi mumkin

</details>

---

### Savol 11: `design:paramtypes` interface bilan output [Middle]

**Savol:** Bu kodda `paramTypes` nima bo'ladi?

```typescript
import "reflect-metadata";
function injectable(target: new (...args: any[]) => any) {}

interface ILogger { log(msg: string): void; }
abstract class BaseRepo { abstract find(): unknown; }
class ConcreteEmail { send() {} }

@injectable
class AppService {
  constructor(
    private logger: ILogger,
    private repo: BaseRepo,
    private email: ConcreteEmail,
    private count: number,
    private tags: string[]
  ) {}
}

const paramTypes = Reflect.getMetadata("design:paramtypes", AppService);
console.log(paramTypes.map((t: any) => t.name));
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
["Object", "BaseRepo", "ConcreteEmail", "Number", "Array"]
```

### To'liq tushuntirish

**Har parametr tahlili:**

1. **`logger: ILogger`** → `Object`
   - Interface — JS da yo'q (compile-time erasure)
   - Compiler `Object` ni metadata sifatida emit qiladi
   - DI container `Object` orqali qaysi logger ekanini bilmaydi → **token kerak**

2. **`repo: BaseRepo`** → `BaseRepo`
   - Abstract class — JS da **class declaration sifatida saqlanadi** (abstract method lar runtime check yo'q)
   - Class reference metadata da saqlanadi
   - DI container `BaseRepo` token sifatida ishlatishi mumkin

3. **`email: ConcreteEmail`** → `ConcreteEmail`
   - Oddiy class — runtime da mavjud
   - Class reference metadata da

4. **`count: number`** → `Number`
   - Primitive → wrapper constructor

5. **`tags: string[]`** → `Array`
   - Array — class reference, element type yo'qoladi

### Kod misol

```typescript
import "reflect-metadata";

function injectable(target: new (...args: any[]) => any) {}

interface ILogger { log(msg: string): void; }

abstract class BaseRepo { abstract find(): unknown; }

class ConcreteEmail { send() {} }

@injectable
class AppService {
  constructor(
    private logger: ILogger,
    private repo: BaseRepo,
    private email: ConcreteEmail,
    private count: number,
    private tags: string[]
  ) {}
}

const paramTypes = Reflect.getMetadata("design:paramtypes", AppService);
console.log(paramTypes.map((t: any) => t.name));
// ["Object", "BaseRepo", "ConcreteEmail", "Number", "Array"]


// === DI container nima qila oladi ===
// 1. logger — Object — implementation noaniq, token kerak
// 2. repo — BaseRepo — container.bind(BaseRepo).to(PostgresRepo) ishlaydi
// 3. email — ConcreteEmail — container.bind(ConcreteEmail).to(ConcreteEmail) ishlaydi
// 4. count — Number — primitive, useValue bilan resolve
// 5. tags — Array — element type noaniq, useValue bilan resolve
```

### Edge Cases

- **Abstract class — ham class ham interface:** runtime da class reference saqlanadi, lekin `new BaseRepo()` mumkin emas (abstract method)
- **Generic class:** `class Repo<T> {}` — runtime da `Repo` (T erasure). DI da generic argument yo'qoladi
- **Type alias:** `type Logger = { log: (msg: string) => void }` → `Object` (interface bilan bir xil erasure)
- **Tuple:** `[string, number]` → `Array` (tuple element type lar yo'qoladi)
- **Function type:** `(msg: string) => void` → `Function` (parameter va return yo'qoladi)

### Follow-up savollar

1. **"Nima uchun abstract class metadata da saqlanadi, interface esa yo'q?"** — Abstract class — `class` declaration syntax bilan, JS ga compile qilinganda constructor function qoldiradi (`new` qila olmasak ham reference mavjud). Interface — type-only construct, JS ga compile qilinmaydi, runtime da artifact yo'q
2. **"Token o'rniga abstract class ishlatish yaxshimi?"** — Ba'zi holatda — abstract class ham class reference, ham contract berib, qo'shimcha token kerak emas. Lekin runtime da `new AbstractClass()` xavfli (abstract method check yo'q) va class darajasidagi state imkonsiz

</details>

---

### Savol 12: Auto-resolve container output [Middle+]

**Savol:** Output ni ayting:

```typescript
import "reflect-metadata";

function injectable(target: new (...args: any[]) => any) {
  console.log(`Registering: ${target.name}`);
}

class Logger {
  log(msg: string) { console.log(`[LOG] ${msg}`); }
}

class Database {
  constructor(private logger: Logger) {
    console.log(`Database created`);
  }
  query(sql: string) { this.logger.log(`SQL: ${sql}`); return []; }
}

@injectable
class UserService {
  constructor(private db: Database, private logger: Logger) {
    console.log(`UserService created`);
  }
  getUsers() {
    this.logger.log("Getting users");
    return this.db.query("SELECT * FROM users");
  }
}

class Container {
  private singletons = new Map<any, any>();
  resolve<T>(target: new (...args: any[]) => T): T {
    if (this.singletons.has(target)) return this.singletons.get(target);
    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", target) || [];
    const deps = paramTypes.map(dep => this.resolve(dep));
    const instance = new target(...deps);
    this.singletons.set(target, instance);
    return instance;
  }
}

console.log("--- before resolve ---");
const container = new Container();
const service = container.resolve(UserService);
console.log("--- after resolve ---");
service.getUsers();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
Registering: UserService
--- before resolve ---
Database created
UserService created
--- after resolve ---
[LOG] Getting users
TypeError: Cannot read properties of undefined (reading 'log')
```

`Logger` va `Database` da `@injectable` yo'q — `emitDecoratorMetadata` ular uchun `design:paramtypes` emit qilmaydi. Container `Database` ni argumentsiz yaratadi va `Database.logger` undefined bo'lib qoladi.

### To'liq tushuntirish

**Class definition phase (decorator larning sync chaqiruvi):**

1. Compiler `@injectable` decorator ni `UserService` ga apply qiladi → `"Registering: UserService"` console ga chiqadi
2. `Logger` va `Database` da decorator yo'q → `injectable` function chaqirilmaydi va metadata emit qilinmaydi

**Muhim qoida:** TypeScript `emitDecoratorMetadata` faqat **decorate qilingan element lar uchun** `design:paramtypes` emit qiladi. Decorator yo'q bo'lsa — metadata yo'q.

**Resolve trace:**

```
container.resolve(UserService)
  paramTypes = [Database, Logger]    // UserService decorated
  resolve(Database)
    paramTypes = []                  // Database decorate qilinmagan → metadata yo'q
    new Database()                   // logger argumentisiz
    Database.logger = undefined
    console.log("Database created")
  resolve(Logger)
    paramTypes = []
    new Logger()
  new UserService(dbInstance, loggerInstance)
  console.log("UserService created")
service.getUsers()
  this.logger.log("Getting users")   // UserService.logger valid
  this.db.query("SELECT * FROM users")
    this.logger.log(...)             // Database.logger undefined → TypeError
```

**Yechim:** `Database` va `Logger` ga ham `@injectable` decorator qo'shilishi shart. Decorator side-effect ahamiyatsiz — `emitDecoratorMetadata` faqat decorate qilingan element lar uchun metadata emit qiladi.

### Kod misol

```typescript
import "reflect-metadata";

function injectable(target: new (...args: any[]) => any) {
  console.log(`Registering: ${target.name}`);
}

@injectable
class Logger {
  log(msg: string) { console.log(`[LOG] ${msg}`); }
}

@injectable
class Database {
  constructor(private logger: Logger) {
    console.log(`Database created`);
  }
  query(sql: string) { this.logger.log(`SQL: ${sql}`); return []; }
}

@injectable
class UserService {
  constructor(private db: Database, private logger: Logger) {
    console.log(`UserService created`);
  }
  getUsers() {
    this.logger.log("Getting users");
    return this.db.query("SELECT * FROM users");
  }
}

class Container {
  private singletons = new Map<any, any>();
  resolve<T>(target: new (...args: any[]) => T): T {
    if (this.singletons.has(target)) return this.singletons.get(target);
    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", target) || [];
    const deps = paramTypes.map(dep => this.resolve(dep));
    const instance = new target(...deps);
    this.singletons.set(target, instance);
    return instance;
  }
}

const container = new Container();
const service = container.resolve(UserService);
service.getUsers();

// Registering: Logger
// Registering: Database
// Registering: UserService
// Database created
// UserService created
// [LOG] Getting users
// [LOG] SQL: SELECT * FROM users
```

### Edge Cases

- **`@injectable` ni unutish:** decorator yo'q class — `paramTypes` undefined, constructor argument lar undefined → runtime error
- **Singleton cache:** keyingi `resolve(Logger)` cache dan oladi — `Database created` faqat bir marta
- **Resolve order:** dependency tree leaf birinchi (`Logger`), root oxirgi (`UserService`). Topological order
- **Constructor side effect:** `console.log("created")` constructor da — har resolve da bir marta singleton uchun

### Follow-up savollar

1. **"Decorator bo'lmaganda nima emit qilinadi?"** — Hech nima. `Reflect.getMetadata("design:paramtypes", ...)` `undefined`
2. **"Singleton yo'q bo'lsa nima bo'ladi?"** — Har resolve da yangi instance — `Database created`, `UserService created` har bir resolve da

</details>

---

## Coding savollar

### Savol 13: Sodda DI Container yozing [Senior]

**Savol:** `register` va `resolve` method li DI Container yozing. `emitDecoratorMetadata` dan dependency larni o'qib avtomatik resolve qilsin. Singleton va transient scope qo'llab-quvvatlasin. `useValue`, `useClass`, `useFactory` provider turlari bo'lsin.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Map` da provider lar saqlanadi (token → registration). `resolve` recursive ishlaydi: `design:paramtypes` dan parameter type larni o'qib, har dep ni resolve qiladi. Singleton scope — cache `Map`. `useFactory` factory chaqiradi.

### To'liq tushuntirish

**Container komponentlari:**

1. **Registration map** — token → `{ useClass/useValue/useFactory, scope }`
2. **Singleton cache** — singleton instance lar
3. **Resolve algorithm:**
   - Token bo'yicha registration topish
   - `useValue` bo'lsa — to'g'ridan-to'g'ri qaytarish
   - `useFactory` bo'lsa — factory chaqirib instance olish
   - `useClass` bo'lsa — `design:paramtypes` dan dependency type larni o'qish, har dep ni recursive resolve qilish, `new useClass(...deps)`
   - Singleton bo'lsa — cache ga saqlash

### Kod misol

```typescript
import "reflect-metadata";

type Scope = "singleton" | "transient";

interface Registration {
  useClass?: new (...args: any[]) => any;
  useValue?: any;
  useFactory?: (container: Container) => any;
  scope?: Scope;
}

class Container {
  private registrations = new Map<any, Registration>();
  private singletons = new Map<any, any>();
  private resolving = new Set<any>(); // Circular detection

  register(token: any, options: Registration): void {
    this.registrations.set(token, { scope: "transient", ...options });
  }

  resolve<T>(token: any): T {
    // Circular dependency detection
    if (this.resolving.has(token)) {
      const chain = [...this.resolving, token].map((t) => t.name || String(t));
      throw new Error(`Circular dependency: ${chain.join(" -> ")}`);
    }

    // Direct resolve (class reference passed)
    let reg = this.registrations.get(token);
    if (!reg && typeof token === "function") {
      reg = { useClass: token };
    }
    if (!reg) {
      throw new Error(`No registration for: ${token?.name || String(token)}`);
    }

    // useValue — direct return
    if (reg.useValue !== undefined) {
      return reg.useValue;
    }

    // Singleton cache check
    const scope = reg.scope || "transient";
    if (scope === "singleton" && this.singletons.has(token)) {
      return this.singletons.get(token);
    }

    this.resolving.add(token);
    let instance: T;

    try {
      // useFactory
      if (reg.useFactory) {
        instance = reg.useFactory(this);
      }
      // useClass — auto-resolve
      else if (reg.useClass) {
        const paramTypes: any[] =
          Reflect.getMetadata("design:paramtypes", reg.useClass) || [];

        // Token overrides (from @Inject decorator)
        const tokens: Map<number, any> =
          Reflect.getOwnMetadata("inject:tokens", reg.useClass) || new Map();

        const deps = paramTypes.map((type, i) => {
          const t = tokens.get(i) ?? type;
          return this.resolve(t);
        });

        instance = new reg.useClass(...deps);
      } else {
        throw new Error(`Invalid registration for: ${String(token)}`);
      }

      // Cache singleton
      if (scope === "singleton") {
        this.singletons.set(token, instance);
      }

      return instance;
    } finally {
      this.resolving.delete(token);
    }
  }
}


// === Usage ===
function injectable(target: new (...args: any[]) => any) {}

@injectable
class Logger {
  log(msg: string) { console.log(`[LOG] ${msg}`); }
}

@injectable
class Database {
  constructor(private logger: Logger) {}
  query(sql: string) {
    this.logger.log(`SQL: ${sql}`);
    return [];
  }
}

@injectable
class UserService {
  constructor(private db: Database, private logger: Logger) {}
  getUsers() {
    this.logger.log("Getting users");
    return this.db.query("SELECT * FROM users");
  }
}

const container = new Container();
container.register(Logger, { useClass: Logger, scope: "singleton" });
container.register(Database, { useClass: Database, scope: "singleton" });
container.register(UserService, { useClass: UserService });

const service = container.resolve<UserService>(UserService);
service.getUsers();
// [LOG] Getting users
// [LOG] SQL: SELECT * FROM users


// === useValue ===
const CONFIG = Symbol("Config");
container.register(CONFIG, { useValue: { dbUrl: "postgres://localhost" } });

const config = container.resolve<{ dbUrl: string }>(CONFIG);
console.log(config.dbUrl); // "postgres://localhost"


// === useFactory ===
const DB_CONN = Symbol("DbConn");
container.register(DB_CONN, {
  useFactory: (c) => {
    const cfg = c.resolve<{ dbUrl: string }>(CONFIG);
    return { connect: () => `Connected to ${cfg.dbUrl}` };
  },
  scope: "singleton",
});

const conn = container.resolve<{ connect: () => string }>(DB_CONN);
console.log(conn.connect()); // "Connected to postgres://localhost"
```

### Edge Cases

- **Interface (`design:paramtypes` da `Object`):** auto-resolve ishlamaydi. `@Inject(TOKEN)` parameter decorator bilan token kerak
- **Generic type:** `Repo<User>` — generic erasure. Har generic instantiation uchun alohida token
- **Circular dependency:** `resolving` Set bilan track qilinadi → explicit error
- **Singleton va test:** har test da yangi container kerak — eski singleton state ko'chmasin
- **Async factory:** `useFactory` Promise qaytarsa — caller `await` qilishi shart, container Promise saqlaydi (lekin singleton cache da Promise saqlanadi — har resolve await qiluvchi unwrap qiladi)

### Follow-up savollar

1. **"Async useFactory qanday handle qilinadi?"** — Container `resolveAsync` method qo'shish, yoki Promise ni cache qilish (await bir marta — keyingi resolve cached Promise dan resolve qiladi)
2. **"Lifecycle hooks (onInit, onDestroy)?"** — Singleton uchun ham `onDestroy` kerak — container.dispose() chaqirilganda. NestJS module lifecycle pattern

<details>
<summary><strong>Deep Dive</strong></summary>

**Production-grade container — advanced features:**

```typescript
interface IDisposable { dispose(): Promise<void> | void; }

class AdvancedContainer extends Container {
  private requestScope = new WeakMap<object, Map<any, any>>(); // Per-request
  private modules = new Map<string, Container>(); // Modular DI

  async dispose() {
    for (const instance of this.singletons.values()) {
      if (typeof instance?.dispose === "function") {
        await (instance as IDisposable).dispose();
      }
    }
    this.singletons.clear();
  }

  resolveRequest<T>(token: any, requestContext: object): T {
    let scope = this.requestScope.get(requestContext);
    if (!scope) {
      scope = new Map();
      this.requestScope.set(requestContext, scope);
    }
    if (scope.has(token)) return scope.get(token);

    const instance = this.resolve<T>(token);
    scope.set(token, instance);
    return instance;
  }
}
```

**Lifecycle phases:**
- **Registration** — bind tokens to providers
- **Initialization** — `onInit` callback lar (factory call qilinganda)
- **Resolution** — dependency graph traversal
- **Disposal** — `onDestroy` callback lar, singleton tozalash

</details>

</details>

---

### Savol 14: Scope support (singleton/transient) [Middle+]

**Savol:** Container ga singleton va transient scope qo'shing. Behavior:
- **Singleton** — har `resolve` da bir xil instance
- **Transient** — har `resolve` da yangi instance

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Registration` ga `scope: "singleton" | "transient"` qo'shilsin. Resolve da singleton bo'lsa — cache check. Yangi instance yaratilganda — singleton bo'lsa cache, transient bo'lsa cache yo'q.

### To'liq tushuntirish

**Scope flow:**

1. Register vaqtida `scope` saqlash
2. Resolve da:
   - Singleton + cached → cache dan
   - Singleton + cached emas → yarat va cache ga
   - Transient → har safar yarat (cache yo'q)

**Default scope tanlash:** ko'p framework — singleton (NestJS default), boshqalari — transient. Container API da `register` da default value: convention bo'yicha `singleton` ko'proq xavfsiz (resource sharing).

### Kod misol

```typescript
import "reflect-metadata";

type Scope = "singleton" | "transient";

interface Registration {
  cls: new (...args: any[]) => any;
  scope: Scope;
}

class ScopedContainer {
  private registrations = new Map<any, Registration>();
  private singletons = new Map<any, any>();

  register<T>(
    token: any,
    cls: new (...args: any[]) => T,
    scope: Scope = "transient"
  ): void {
    this.registrations.set(token, { cls, scope });
  }

  resolve<T>(token: any): T {
    const reg = this.registrations.get(token);
    if (!reg) throw new Error(`No registration: ${String(token)}`);

    // Singleton — return cached or create + cache
    if (reg.scope === "singleton") {
      if (this.singletons.has(token)) {
        return this.singletons.get(token);
      }
      const instance = this.create<T>(reg.cls);
      this.singletons.set(token, instance);
      return instance;
    }

    // Transient — always new
    return this.create<T>(reg.cls);
  }

  private create<T>(cls: new (...args: any[]) => any): T {
    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", cls) || [];
    const deps = paramTypes.map((type) => this.resolve(type));
    return new cls(...deps);
  }

  reset(): void {
    this.singletons.clear();
  }
}


// === Usage ===
function injectable(target: new (...args: any[]) => any) {}

@injectable
class Counter {
  count = 0;
  increment() { this.count++; }
}

@injectable
class RequestContext {
  id = Math.random();
}

const container = new ScopedContainer();
container.register(Counter, Counter, "singleton");
container.register(RequestContext, RequestContext, "transient");

// Singleton — same instance
const c1 = container.resolve<Counter>(Counter);
const c2 = container.resolve<Counter>(Counter);
c1.increment();
c1.increment();
console.log(c1 === c2);     // true
console.log(c2.count);       // 2 — shared state

// Transient — new instance
const r1 = container.resolve<RequestContext>(RequestContext);
const r2 = container.resolve<RequestContext>(RequestContext);
console.log(r1 === r2);      // false
console.log(r1.id !== r2.id); // true


// === Test isolation ===
beforeEach(() => {
  container.reset(); // Singleton state ni clear
});
```

### Edge Cases

- **Singleton + transient mix:** transient service singleton dep ga muhtoj — transient har xil, lekin dep bir xil
- **Singleton state leak (test):** test lar bir-birini ko'radi → `beforeEach(() => container.reset())`
- **Multi-process:** Node.js cluster da har worker — alohida container instance. Singleton process-level, machine-level emas
- **Constructor side effect:** transient da har resolve — yangi side effect (`console.log("created")`). Performance-critical kod uchun e'tibor
- **Scoped scope:** request lifecycle bilan bog'langan (HTTP). Container request context bilan resolve qiladi

### Follow-up savollar

1. **"Default scope qaysi yaxshiroq?"** — Singleton — kam memory, lekin shared mutable state xavfli. Transient — xavfsiz, lekin garbage collection load
2. **"Scope test da qanday isolate qilinadi?"** — `beforeEach` da yangi container yoki `container.reset()`
3. **"Per-request scope qanday implement qilinadi?"** — Request context object orqali — `WeakMap<context, Map<token, instance>>`

</details>

---

### Savol 15: Token-based `@inject` decorator [Middle+]

**Savol:** Interface DI muammosini hal qilish uchun `@inject(token)` parameter decorator (legacy) yozing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Parameter decorator paramIndex va token ni `Reflect` metadata ga yozadi. Container resolve da `design:paramtypes` bilan birga `inject:tokens` metadata ni o'qiydi va token bo'lsa uni ishlatadi.

### To'liq tushuntirish

**Token metadata storage:**

Map struktura: `paramIndex → token`. Class-level metadata sifatida saqlanadi, har class uchun alohida.

**Container resolve algorithm:**

```
For each parameter:
  if token override exists at index i:
    resolve(token_at_i)
  else:
    resolve(paramTypes[i])
```

### Kod misol

```typescript
import "reflect-metadata";

const INJECT_TOKENS_KEY = "di:inject_tokens";

// === Parameter decorator (legacy) ===
function inject(token: any) {
  return function (
    target: any,
    propertyKey: string | symbol | undefined,
    parameterIndex: number
  ) {
    // Constructor parameter — propertyKey undefined
    // Method parameter — propertyKey method name
    const key = propertyKey ?? "constructor";
    const tokensMap: Map<string | symbol, Map<number, any>> =
      Reflect.getOwnMetadata(INJECT_TOKENS_KEY, target) || new Map();

    const paramTokens: Map<number, any> = tokensMap.get(key) || new Map();
    paramTokens.set(parameterIndex, token);
    tokensMap.set(key, paramTokens);

    Reflect.defineMetadata(INJECT_TOKENS_KEY, tokensMap, target);
  };
}

// === Class decorator ===
function injectable(target: new (...args: any[]) => any) {}


// === Container ===
interface Registration {
  useClass?: new (...args: any[]) => any;
  useValue?: any;
  useFactory?: () => any;
}

class TokenContainer {
  private bindings = new Map<any, Registration>();

  bind(token: any) {
    return {
      to: (cls: new (...args: any[]) => any) =>
        this.bindings.set(token, { useClass: cls }),
      toValue: (value: any) =>
        this.bindings.set(token, { useValue: value }),
      toFactory: (factory: () => any) =>
        this.bindings.set(token, { useFactory: factory }),
    };
  }

  resolve<T>(token: any): T {
    const reg = this.bindings.get(token);
    if (!reg) throw new Error(`No binding: ${String(token)}`);

    if (reg.useValue !== undefined) return reg.useValue;
    if (reg.useFactory) return reg.useFactory();

    if (reg.useClass) {
      const paramTypes: any[] =
        Reflect.getMetadata("design:paramtypes", reg.useClass) || [];

      const tokensMap: Map<string | symbol, Map<number, any>> =
        Reflect.getOwnMetadata(INJECT_TOKENS_KEY, reg.useClass) || new Map();
      const constructorTokens = tokensMap.get("constructor") || new Map();

      const deps = paramTypes.map((type, i) => {
        const overrideToken = constructorTokens.get(i);
        return this.resolve(overrideToken ?? type);
      });

      return new reg.useClass(...deps) as T;
    }

    throw new Error("Invalid registration");
  }
}


// === Usage ===
interface ILogger { log(msg: string): void; }
interface IUserRepo { findAll(): Promise<{ id: number; name: string }[]>; }

const TOKENS = {
  Logger: Symbol("Logger"),
  UserRepo: Symbol("UserRepo"),
} as const;

class ConsoleLogger implements ILogger {
  log(msg: string) { console.log(msg); }
}

class InMemoryUserRepo implements IUserRepo {
  async findAll() { return [{ id: 1, name: "Ali" }]; }
}

@injectable
class UserService {
  constructor(
    @inject(TOKENS.Logger) private logger: ILogger,
    @inject(TOKENS.UserRepo) private repo: IUserRepo
  ) {}

  async getAll() {
    this.logger.log("Getting users");
    return this.repo.findAll();
  }
}


const container = new TokenContainer();
container.bind(TOKENS.Logger).to(ConsoleLogger);
container.bind(TOKENS.UserRepo).to(InMemoryUserRepo);
container.bind(UserService).to(UserService);

const service = container.resolve<UserService>(UserService);
await service.getAll();
// "Getting users"
// [{ id: 1, name: "Ali" }]
```

### Edge Cases

- **TC39 da parameter decorator yo'q** — bu legacy faqat. TC39 da class-level metadata yoki object parameter pattern
- **Method parameter:** `getUser(@inject(TOKEN) param: Type)` — method-level metadata. Container method invocation da resolve qila olishi kerak (DI for methods — rare)
- **Token + concrete class:** `@inject(LOGGER_TOKEN) private logger: ILogger` — type annotation faqat compile-time. Runtime — token aniqlaydi
- **Inheritance:** child class parent decorator larini inherit qiladi `getMetadata` orqali (lekin `getOwnMetadata` qilmaydi)
- **Symbol token vs string token:** Symbol — collision-free, IDE refactor xavfsiz. String — readable, lekin typo xavfi

### Follow-up savollar

1. **"TC39 da bu pattern qanday yozish kerak?"** — Class-level decorator + token list yoki object parameter pattern. Parameter decorator hozircha yo'q
2. **"InversifyJS pattern bilan farqi?"** — InversifyJS ham shu pattern, lekin `Container` API kengroq (binding chain, scope, conditional binding)

</details>

---

### Savol 16: DI bilan testing — mock inject [Middle+]

**Savol:** DI bilan `UserService` ni test qiling — real database va email kerak emas, mock lar inject qiling.

```typescript
interface IUserRepository {
  findById(id: number): Promise<User | null>;
  save(user: Omit<User, "id">): Promise<User>;
}
interface IEmailService {
  sendWelcome(email: string, name: string): Promise<void>;
}
interface User { id: number; name: string; email: string; }

class UserService {
  constructor(private repo: IUserRepository, private email: IEmailService) {}
  async createUser(name: string, email: string): Promise<User> {
    const existing = await this.repo.findById(0); // demo
    if (existing) throw new Error("Already exists");
    const user = await this.repo.save({ name, email });
    await this.email.sendWelcome(email, name);
    return user;
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Test da `IUserRepository` va `IEmailService` implementations ni stub/mock object lar bilan almashtirish. Constructor injection orqali mock lar `UserService` ga beriladi — real I/O kerak emas, test deterministik va tez.

### To'liq tushuntirish

**Test pyramid:**

| Level | Mock strategy | Speed |
|-------|---------------|-------|
| Unit | Barcha dep mock | Eng tez |
| Integration | Real DB, mock external | O'rta |
| E2E | Hammasi real | Sekin |

**Mock turlari:**

1. **Stub** — qaytaradigan natija oldindan belgilangan
2. **Spy** — chaqiruvlarni track qiladi (parametrlar, count)
3. **Mock** — stub + spy + expectation
4. **Fake** — soddalashtirilgan implementation (in-memory DB)

**DI bilan testing afzalliklari:**
- Real database, network, file system kerak emas
- Test millisekundlarda
- Deterministik (random faktor yo'q)
- Edge case osongina sinash (DB error, network timeout)

### Kod misol

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";

interface User { id: number; name: string; email: string; }

interface IUserRepository {
  findById(id: number): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: Omit<User, "id">): Promise<User>;
}

interface IEmailService {
  sendWelcome(email: string, name: string): Promise<void>;
}

class UserService {
  constructor(
    private repo: IUserRepository,
    private email: IEmailService
  ) {}

  async createUser(name: string, email: string): Promise<User> {
    const existing = await this.repo.findByEmail(email);
    if (existing) throw new Error("Email already exists");

    const user = await this.repo.save({ name, email });
    await this.email.sendWelcome(email, name);
    return user;
  }
}


// === Test setup ===
describe("UserService", () => {
  let service: UserService;
  let mockRepo: IUserRepository;
  let mockEmail: IEmailService;
  let saveSpy: ReturnType<typeof vi.fn>;
  let sendWelcomeSpy: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    saveSpy = vi.fn(async (data: Omit<User, "id">) => ({ id: 1, ...data }));
    sendWelcomeSpy = vi.fn(async () => undefined);

    mockRepo = {
      findById: vi.fn(async () => null),
      findByEmail: vi.fn(async () => null),
      save: saveSpy,
    };

    mockEmail = {
      sendWelcome: sendWelcomeSpy,
    };

    // DI — mock lar inject
    service = new UserService(mockRepo, mockEmail);
  });

  it("creates user and sends welcome email", async () => {
    const user = await service.createUser("Ali", "ali@example.com");

    expect(user).toEqual({
      id: 1,
      name: "Ali",
      email: "ali@example.com",
    });

    expect(saveSpy).toHaveBeenCalledWith({
      name: "Ali",
      email: "ali@example.com",
    });
    expect(saveSpy).toHaveBeenCalledTimes(1);

    expect(sendWelcomeSpy).toHaveBeenCalledWith(
      "ali@example.com",
      "Ali"
    );
  });

  it("rejects duplicate email", async () => {
    mockRepo.findByEmail = vi.fn(async () => ({
      id: 99,
      name: "Existing",
      email: "ali@example.com",
    }));

    await expect(
      service.createUser("Ali", "ali@example.com")
    ).rejects.toThrow("Email already exists");

    expect(saveSpy).not.toHaveBeenCalled();
    expect(sendWelcomeSpy).not.toHaveBeenCalled();
  });

  it("propagates repository error", async () => {
    mockRepo.save = vi.fn(async () => {
      throw new Error("DB connection failed");
    });

    await expect(
      service.createUser("Ali", "ali@example.com")
    ).rejects.toThrow("DB connection failed");

    expect(sendWelcomeSpy).not.toHaveBeenCalled();
  });

  it("propagates email error after user is saved", async () => {
    sendWelcomeSpy.mockRejectedValue(new Error("SMTP down"));

    await expect(
      service.createUser("Ali", "ali@example.com")
    ).rejects.toThrow("SMTP down");

    // save sendWelcome dan oldin chaqirilgan — user yozilgan, lekin email xatosi propagate bo'ladi
    expect(saveSpy).toHaveBeenCalled();
  });
});


// === Fake (in-memory) repository — integration-like test ===
class FakeUserRepository implements IUserRepository {
  private users = new Map<number, User>();
  private nextId = 1;

  async findById(id: number) { return this.users.get(id) ?? null; }

  async findByEmail(email: string) {
    return [...this.users.values()].find((u) => u.email === email) ?? null;
  }

  async save(data: Omit<User, "id">) {
    const user = { id: this.nextId++, ...data };
    this.users.set(user.id, user);
    return user;
  }
}

describe("UserService with fake repo", () => {
  it("creates multiple users", async () => {
    const repo = new FakeUserRepository();
    const email: IEmailService = { sendWelcome: async () => {} };
    const service = new UserService(repo, email);

    await service.createUser("Ali", "ali@test.com");
    await service.createUser("Vali", "vali@test.com");

    const ali = await repo.findByEmail("ali@test.com");
    expect(ali?.id).toBe(1);

    const vali = await repo.findByEmail("vali@test.com");
    expect(vali?.id).toBe(2);
  });
});
```

### Edge Cases

- **Spy verification:** parametrlar `toHaveBeenCalledWith`, count `toHaveBeenCalledTimes`. Strict comparison
- **Async mock:** `vi.fn(async () => ...)` yoki `mockResolvedValue`. Reject — `mockRejectedValue`
- **Partial mock:** kerakli method larni mock, qolganlarini stub. Lekin TS strict mode — to'liq interface implement kerak
- **Mock leak:** test orasida mock state leak. `beforeEach` da reset, yoki `vi.restoreAllMocks()`
- **Real timer:** `setTimeout` ishlatadigan service — `vi.useFakeTimers()` bilan vaqtni boshqarish

### Follow-up savollar

1. **"`vi.fn()` va manual mock orasida qaysi yaxshi?"** — `vi.fn()` — spy + reset, har test uchun fresh. Manual mock — stable behavior kerak bo'lganda
2. **"Test pyramid da DI qanday joy egallaydi?"** — Unit test layer da DI critical. Integration da real implementation, lekin external (network, file) mock
3. **"Interface dan ko'p method bo'lsa nima qilish?"** — Helper factory function — `createMockUserRepo(overrides)` — default mock + override

</details>

---

## Bug fix savollar

### Savol 17: `reflect-metadata` import order xato [Middle]

**Savol:** Bu kodda nima xato? Toping va tuzating:

```typescript
// service.ts
function injectable(target: any) {
  Reflect.defineMetadata("injectable", true, target);
}

@injectable
class UserService {
  constructor(private logger: any) {}
}

// app.ts
import "./service";
import "reflect-metadata";

console.log(Reflect.getMetadata("injectable", UserService));
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`reflect-metadata` polyfill `Reflect.defineMetadata` va `Reflect.getMetadata` ni global `Reflect` object ga qo'shadi. Import qilinmasdan oldin shu method lar mavjud emas. Yechim: `reflect-metadata` ni app entry point ning **eng birinchi import** sifatida qo'shish.

### To'liq tushuntirish

**Muammo tahlili:**

```typescript
// app.ts
import "./service";          // ❌ service.ts birinchi import
                              //    service.ts da @injectable ishlaydi
                              //    Reflect.defineMetadata — undefined function
                              //    → TypeError yoki silent fail

import "reflect-metadata";   // Kech — service.ts allaqachon yuklangan
```

**ES module yuklanish tartibi:**

1. `import "./service"` — `service.ts` parse va execute qilinadi
2. `service.ts` ichida `@injectable` decorator runtime da chaqiriladi
3. `Reflect.defineMetadata` — global `Reflect` object da yo'q (polyfill yuklanmagan)
4. → `TypeError: Reflect.defineMetadata is not a function`

`reflect-metadata` side-effect import — global `Reflect` ni patch qiladi. Qaysi tartibda import bo'lsa, shunda patch qo'llaniladi.

**To'g'ri tartib:**

```typescript
// app.ts — eng birinchi qatorda
import "reflect-metadata";

// Keyin barcha boshqalari
import "./service";
import { UserService } from "./service";

console.log(Reflect.getMetadata("injectable", UserService)); // true
```

**Production setup:**

NestJS, Angular, TypeORM loyihalarda `reflect-metadata` `main.ts` yoki `bootstrap.ts` ning birinchi qatorida.

### Kod misol

```typescript
// ❌ XATO TARTIB
// app.ts
import "./service";              // service.ts da Reflect ishlatiladi, lekin polyfill yo'q
import "reflect-metadata";       // Kech
import { UserService } from "./service";

// service.ts
function injectable(target: any) {
  Reflect.defineMetadata("injectable", true, target); // TypeError!
}

@injectable
class UserService {}


// ✅ TO'G'RI TARTIB
// app.ts — eng birinchi qator
import "reflect-metadata";       // Polyfill birinchi
import "./service";              // Endi Reflect.defineMetadata mavjud
import { UserService } from "./service";

const meta = Reflect.getMetadata("injectable", UserService);
console.log(meta); // true


// === main.ts (NestJS-style) ===
import "reflect-metadata";       // ALWAYS FIRST

import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

### Edge Cases

- **TypeScript bundler:** bundler import order ni preserve qiladi, lekin tree-shaking bo'sh import lar ni olib tashlashi mumkin. `reflect-metadata` ni explicit ishlatish (`import * as ... from "reflect-metadata"` keraksiz — side-effect import yetarli)
- **Test setup:** `vitest.config` yoki `jest.setup.ts` da `reflect-metadata` global import qilinadi
- **Multiple imports:** `reflect-metadata` ni har faylda import qilish kerakmi — yo'q, bir marta yetarli (global `Reflect` ga patch)
- **Decorator usage bo'lmasa:** `reflect-metadata` import qilmaslik mumkin. Lekin decorator + `emitDecoratorMetadata` true bo'lsa — `__metadata` chaqiruvi compiled code da bo'ladi → kerak
- **Browser bundle:** `reflect-metadata` polyfill production bundle ga qo'shiladi — size impact

### Follow-up savollar

1. **"Nima uchun `Symbol.metadata` da bu muammo yo'q?"** — Native ECMAScript spec — polyfill kerak emas (modern env), yoki minimal polyfill (`Symbol.metadata ??= Symbol("Symbol.metadata")`)
2. **"Babel orqali metadata emit qilish?"** — `@babel/plugin-transform-typescript` — to'liq `emitDecoratorMetadata` qo'llab-quvvatlamaydi. TypeScript compiler afzal
3. **"Cyclic import ham bu muammoga sabab bo'la oladimi?"** — Cyclic import partial undefined export beradi — decorator chaqirilganda dep undefined bo'lishi mumkin

</details>

---

### Savol 18: Singleton bo'lib qolgan transient — bug toping [Middle+]

**Savol:** `RequestContext` transient bo'lishi kerak (har request uchun yangi), lekin har resolve da bir xil instance qaytadi. Nima xato?

```typescript
import "reflect-metadata";

type Scope = "singleton" | "transient";

interface Registration {
  cls: new (...args: any[]) => any;
  scope: Scope;
}

class Container {
  private registrations = new Map<any, Registration>();
  private cache = new Map<any, any>();

  register<T>(token: any, cls: new (...args: any[]) => T, scope: Scope) {
    this.registrations.set(token, { cls, scope });
  }

  resolve<T>(token: any): T {
    const reg = this.registrations.get(token);
    if (!reg) throw new Error(`No registration: ${String(token)}`);

    if (this.cache.has(token)) {
      return this.cache.get(token);
    }

    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", reg.cls) || [];
    const deps = paramTypes.map((t) => this.resolve(t));
    const instance = new reg.cls(...deps);

    this.cache.set(token, instance);
    return instance;
  }
}

class RequestContext { id = Math.random(); }

const container = new Container();
container.register(RequestContext, RequestContext, "transient");

const a = container.resolve<RequestContext>(RequestContext);
const b = container.resolve<RequestContext>(RequestContext);
console.log(a === b); // true — XATO, transient bo'lishi kerak edi
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`resolve` cache check ni `scope` ni tekshirmasdan har holatda bajaradi. Birinchi `resolve` da instance cache ga yoziladi, keyingi resolve cache dan qaytaradi — scope `transient` bo'lsa ham. Yechim: cache check va cache write `scope === "singleton"` shart ostida bo'lishi kerak.

### To'liq tushuntirish

**Bug tahlili:**

```typescript
if (this.cache.has(token)) {
  return this.cache.get(token);   // ❌ Scope check yo'q
}
// ...
this.cache.set(token, instance);  // ❌ Scope check yo'q — transient ham cache ga
```

Cache ikkita scope uchun ham ishlatilmoqda. Transient ning ta'rifi — **har `resolve` da yangi instance** — buzilgan.

**To'g'ri implementation:**

```typescript
resolve<T>(token: any): T {
  const reg = this.registrations.get(token);
  if (!reg) throw new Error(`No registration: ${String(token)}`);

  // ✅ Singleton bo'lsa cache check
  if (reg.scope === "singleton" && this.cache.has(token)) {
    return this.cache.get(token);
  }

  const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", reg.cls) || [];
  const deps = paramTypes.map((t) => this.resolve(t));
  const instance = new reg.cls(...deps);

  // ✅ Faqat singleton ni cache qilish
  if (reg.scope === "singleton") {
    this.cache.set(token, instance);
  }
  return instance;
}
```

### Kod misol

```typescript
import "reflect-metadata";

type Scope = "singleton" | "transient";

interface Registration {
  cls: new (...args: any[]) => any;
  scope: Scope;
}

class Container {
  private registrations = new Map<any, Registration>();
  private singletons = new Map<any, any>();

  register<T>(token: any, cls: new (...args: any[]) => T, scope: Scope = "transient") {
    this.registrations.set(token, { cls, scope });
  }

  resolve<T>(token: any): T {
    const reg = this.registrations.get(token);
    if (!reg) throw new Error(`No registration: ${String(token)}`);

    if (reg.scope === "singleton" && this.singletons.has(token)) {
      return this.singletons.get(token);
    }

    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", reg.cls) || [];
    const deps = paramTypes.map((t) => this.resolve(t));
    const instance = new reg.cls(...deps);

    if (reg.scope === "singleton") {
      this.singletons.set(token, instance);
    }
    return instance;
  }
}

class RequestContext { id = Math.random(); }
class Logger { id = Math.random(); }

const container = new Container();
container.register(RequestContext, RequestContext, "transient");
container.register(Logger, Logger, "singleton");

const r1 = container.resolve<RequestContext>(RequestContext);
const r2 = container.resolve<RequestContext>(RequestContext);
console.log(r1 === r2); // false — transient, har resolve yangi

const l1 = container.resolve<Logger>(Logger);
const l2 = container.resolve<Logger>(Logger);
console.log(l1 === l2); // true — singleton, bir xil instance
```

### Edge Cases

- **Cache nomi yanglish:** `cache` o'rniga `singletons` deb nomlash semantic to'g'riroq — faqat singleton uchun ishlatilishini ko'rsatadi
- **Default scope:** spec da default `transient` bo'lishi xavfsizroq (no shared state). NestJS default singleton — opt-out
- **Scope inheritance:** transient class singleton ga muhtoj bo'lsa — transient yangi, lekin dep bir xil. Bu odatda kutilgan xatti-harakat
- **Race condition:** singleton birinchi resolve da yaratiladi — concurrent resolve da double-create xavfi (Node.js single-thread bo'lgani uchun resolve sync ekan — yo'q)

### Follow-up savollar

1. **"Nima uchun cache bug to'g'ridan-to'g'ri ko'rinmaydi?"** — Birinchi `console.log(a === b)` chiqishidan keyin ko'rinadi, lekin runtime exception bermaydi — silent bug. Test yoki manual check kerak
2. **"Test bilan bu bug ni qanday topish?"** — Har scope uchun separate test: `expect(container.resolve(X)).not.toBe(container.resolve(X))` for transient
3. **"NestJS bu bug dan qanday himoyalanadi?"** — NestJS Module level scope tracking — `Scope.DEFAULT` (singleton), `Scope.REQUEST`, `Scope.TRANSIENT` enum. Resolve algorithm explicit branch

</details>

---

### Savol 19: `@inject` decorator metadata yo'qotilmoqda — bug toping [Senior]

**Savol:** `@inject(TOKEN)` qo'llanilgan bo'lsa ham container `design:paramtypes` ni ishlatadi va token override qilinmaydi. Nima xato?

```typescript
import "reflect-metadata";

interface ILogger { log(msg: string): void; }

const LOGGER_TOKEN = Symbol("Logger");

function inject(token: any) {
  return function (target: any, _: any, index: number) {
    // ❌ Bug shu yerda
    const tokens = new Map<number, any>();
    tokens.set(index, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}

function injectable(target: any) {}

class ConsoleLogger implements ILogger {
  log(msg: string) { console.log(msg); }
}

@injectable
class UserService {
  constructor(
    @inject("CONFIG") private config: { debug: boolean },
    @inject(LOGGER_TOKEN) private logger: ILogger
  ) {}
}

const tokens = Reflect.getMetadata("inject:tokens", UserService);
console.log(tokens);
// Map(1) { 0 => 'CONFIG' }  — XATO! LOGGER_TOKEN yo'qolgan
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har `@inject` chaqiruvida yangi `Map` yaratiladi va eski metadata ustiga yoziladi. Decorator larning evaluation order — **o'ngdan chapga** (last parameter first). Birinchi `LOGGER_TOKEN` yoziladi, keyin `CONFIG` ustiga yoziladi va `LOGGER_TOKEN` yo'qoladi. Yechim: mavjud metadata ni o'qib, yangi entry qo'shish.

### To'liq tushuntirish

**Decorator evaluation order:**

Parameter decorator lar **teskari tartibda** (right-to-left, last-to-first) bajariladi:

```typescript
constructor(
  @inject("CONFIG") private config: ...,    // 2. Ikkinchi bajariladi
  @inject(LOGGER_TOKEN) private logger: ... // 1. Birinchi bajariladi
)
```

**Buggy code trace:**

```
1. @inject(LOGGER_TOKEN) param 1 ga apply:
   tokens = new Map()         // yangi Map
   tokens.set(1, LOGGER_TOKEN)
   defineMetadata("inject:tokens", { 1: LOGGER_TOKEN }, target)

2. @inject("CONFIG") param 0 ga apply:
   tokens = new Map()         // ❌ yangi Map (avvalgisini olmaydi)
   tokens.set(0, "CONFIG")
   defineMetadata("inject:tokens", { 0: "CONFIG" }, target)  // ❌ override!

Final: { 0: "CONFIG" } — LOGGER_TOKEN yo'qoldi
```

**To'g'ri implementation:**

```typescript
function inject(token: any) {
  return function (target: any, _: any, index: number) {
    // ✅ Mavjud metadata ni o'qish (yoki yangi Map)
    const tokens: Map<number, any> =
      Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    tokens.set(index, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}
```

**Muhim nuance — `getOwnMetadata` vs `getMetadata`:**

- `getMetadata` — prototype chain ni ko'radi (parent metadata ni ham qaytaradi)
- `getOwnMetadata` — faqat shu target ning o'z metadata

Inheritance da agar `getMetadata` ishlatilsa, parent class tokens ni inherit qilamiz va mutate qilamiz — parent class metadata ham buziladi. `getOwnMetadata` xavfsiz.

### Kod misol

```typescript
import "reflect-metadata";

interface ILogger { log(msg: string): void; }

const TOKENS = {
  Config: Symbol("Config"),
  Logger: Symbol("Logger"),
  Database: Symbol("Database"),
} as const;

// ✅ To'g'ri implementation
function inject(token: any) {
  return function (target: any, _: any, index: number) {
    const tokens: Map<number, any> =
      Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    tokens.set(index, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}

function injectable(target: any) {}

@injectable
class OrderService {
  constructor(
    @inject(TOKENS.Config) private config: { debug: boolean },
    @inject(TOKENS.Logger) private logger: ILogger,
    @inject(TOKENS.Database) private db: { query: (sql: string) => unknown }
  ) {}
}

const tokens: Map<number, any> = Reflect.getOwnMetadata("inject:tokens", OrderService);
console.log(tokens.size);                  // 3
console.log(tokens.get(0) === TOKENS.Config);    // true
console.log(tokens.get(1) === TOKENS.Logger);    // true
console.log(tokens.get(2) === TOKENS.Database);  // true
```

### Edge Cases

- **Inheritance va `getOwnMetadata`:** child class o'z `inject:tokens` ga ega — parent ga tegmaydi. Container child + parent metadata ni alohida o'qishi kerak
- **Partial injection:** ba'zi parameter decorate qilinmagan — `tokens.get(i)` undefined, container `design:paramtypes` dan oladi
- **Method-level injection:** method parameter ga `@inject` — `propertyKey` mavjud. Constructor da `propertyKey === undefined`
- **Mutation across instances:** `Map` reference metadata da — agar inherit qilingan tokens ni mutate qilsangiz, parent ham o'zgaradi. Har class o'z `Map` ini saqlashi kerak

### Follow-up savollar

1. **"Decorator evaluation order spec da qayerda?"** — TC39 legacy decorator proposal. Parameter — right-to-left, method/property — top-to-bottom, class — class decorator oxirgi
2. **"`Reflect.defineMetadata` ni avtomatik append qiladigan API bormi?"** — Yo'q. Har `defineMetadata` chaqiruvi override qiladi. Append logic ni qo'lda yozish kerak
3. **"`Symbol.metadata` da bu pattern qanday?"** — TC39 da `context.metadata` object — har decorator metadata key ga qo'shadi. API explicit, override xavfi kamroq

<details>
<summary><strong>Deep Dive</strong></summary>

**TypeScript compiler — decorator evaluation order (spec):**

Compiler `__decorate` helper ni generate qiladi:

```javascript
// Source:
class UserService {
  constructor(
    @inject("CONFIG") private config: any,
    @inject(LOGGER_TOKEN) private logger: any
  ) {}
}

// Compiled (legacy):
var UserService = /** @class */ (function () {
  function UserService(config, logger) {
    this.config = config;
    this.logger = logger;
  }
  UserService = __decorate([
    __param(0, inject("CONFIG")),
    __param(1, inject(LOGGER_TOKEN)),
    __metadata("design:paramtypes", [Object, Object])
  ], UserService);
  return UserService;
}());
```

`__decorate` decorator larni **teskari tartibda** apply qiladi (array oxiridan boshigacha):

```javascript
function __decorate(decorators, target, key, desc) {
  // ...
  for (var i = decorators.length - 1; i >= 0; i--) {
    if (d = decorators[i]) r = ... d(target, key, paramIndex) ... ;
  }
  // ...
}
```

`__param` parameter decorator ni paramIndex bilan o'rab beradi:

```javascript
function __param(paramIndex, decorator) {
  return function (target, key) { decorator(target, key, paramIndex); };
}
```

**Natija:** `__param(1, inject(LOGGER_TOKEN))` birinchi, `__param(0, inject("CONFIG"))` ikkinchi chaqiriladi. Birinchi metadata write `LOGGER_TOKEN` ni saqlaydi, ikkinchi `CONFIG` ni — agar `getOwnMetadata` bilan merge qilinmasa, ikkinchisi birinchini override qiladi.

**Memory tahlili:**

`Reflect.defineMetadata` global `WeakMap<target, Map<propertyKey, Map<metadataKey, value>>>` ni mutate qiladi. Har `defineMetadata` chaqiruvi — Map.set, key bo'yicha override.

**Merge pattern xavfsizligi:**

```javascript
// Race-free (sync decorator evaluation):
const existing = Reflect.getOwnMetadata(KEY, target) || new Map();
existing.set(index, value);
Reflect.defineMetadata(KEY, existing, target);
```

JavaScript decorator evaluation sync — race condition yo'q. Lekin parent class metadata ga mutate qilmaslik uchun `getOwnMetadata` (not `getMetadata`) ishlatish shart.

</details>

</details>

---

## Xulosa

- **Reflect Metadata** — runtime metadata API. `reflect-metadata` polyfill (legacy), `Symbol.metadata` native (TC39)
- **`emitDecoratorMetadata`** — compiler `design:type`, `design:paramtypes`, `design:returntype` emit qiladi. **Faqat legacy** decorator lar bilan
- **Interface erasure** — DI da interface `Object` ga aylanadi → **token system** kerak
- **DI/IoC** — class dependency larini o'zi yaratmaydi, tashqaridan oladi. Loose coupling, testability, SRP
- **Injection turlari** — Constructor (afzal), Property (optional dep), Method (per-call)
- **Scope** — Singleton (shared instance), Transient (yangi instance), Scoped (request lifecycle)
- **DI Container** — `register` + `resolve`, `useClass`/`useValue`/`useFactory` provider lar
- **Circular dependency** — refactor afzal, forward reference yoki lazy injection alternative
- **Decorator-based vs Function-based DI** — Decorator: NestJS/Angular. Function: modern (Hono, Effect)
- **Testing** — DI ning eng katta afzalligi: mock inject oson, real I/O kerak emas
- **`reflect-metadata` import order** — entry point ning **eng birinchi qatori** bo'lishi shart
- **Scope bug** — cache check va write `scope` ga bog'liq bo'lishi shart, transient cache ga yozilmasin
- **Decorator merge** — parameter decorator metadata ni mavjudini o'qib qo'shish (`getOwnMetadata` + `set`), aks holda override