# Bo'lim 20: Reflect Metadata va Dependency Injection

> Reflect Metadata API — TypeScript da decorator lar bilan birgalikda metadata saqlash va o'qish mexanizmi. `reflect-metadata` package orqali class, method, property va parameter larga **runtime da** qo'shimcha ma'lumot biriktirish mumkin. `emitDecoratorMetadata` compiler option yoqilganda, TypeScript o'zi ham type ma'lumotlarini metadata sifatida emit qiladi (`design:type`, `design:paramtypes`, `design:returntype`). Bu metadata Dependency Injection (DI) pattern ning asosiy qismini tashkil etadi. Bu bo'limda Reflect Metadata API va DI pattern ning TypeScript dagi core tushunchalari o'rganiladi.

---

## Mundarija

- [Reflect Metadata API](#reflect-metadata-api)
- [`emitDecoratorMetadata` — Compiler Metadata Emit](#emitdecoratormetadata--compiler-metadata-emit)
- [Dependency Injection Nima — IoC](#dependency-injection-nima--ioc)
- [DI Patterns — Constructor, Property, Method Injection](#di-patterns--constructor-property-method-injection)
- [DI Container Yaratish — Step by Step](#di-container-yaratish--step-by-step)
- [DI va Testing](#di-va-testing)
- [Decorator-based vs Function-based DI](#decorator-based-vs-function-based-di)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Reflect Metadata API

### Nazariya

Reflect Metadata API — `reflect-metadata` polyfill package orqali TypeScript/JavaScript da **arbitrary metadata** ni object larga biriktirish imkonini beruvchi API. Bu API ECMAScript proposal sifatida taklif qilingan, lekin hali standart emas — shuning uchun polyfill kerak.

Metadata — object haqida qo'shimcha ma'lumot: "bu method qaysi HTTP method ga to'g'ri keladi", "bu class singleton mi", "bu property qaysi type da". Bu ma'lumotlar runtime da o'qiladi.

```bash
npm install reflect-metadata
```

Asosiy API:

| Method | Vazifasi |
|--------|---------|
| `Reflect.defineMetadata(key, value, target)` | Target ga metadata yozish |
| `Reflect.defineMetadata(key, value, target, prop)` | Property ga metadata yozish |
| `Reflect.getMetadata(key, target)` | Metadata o'qish |
| `Reflect.hasMetadata(key, target)` | Metadata borligini tekshirish |
| `Reflect.getOwnMetadata(key, target)` | Faqat o'z metadata (inherited emas) |

Ichki implementatsiyasi **WeakMap** asosida ishlaydi — har bir target uchun alohida metadata Map saqlanadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
import "reflect-metadata";

// === Metadata yozish va o'qish ===
class UserService {
  getUsers() { return []; }
}

Reflect.defineMetadata("role", "admin", UserService);
Reflect.defineMetadata("route", "/users", UserService, "getUsers");

console.log(Reflect.getMetadata("role", UserService)); // "admin"
console.log(Reflect.getMetadata("route", UserService, "getUsers")); // "/users"

// === Decorator bilan metadata ===
function httpGet(path: string) {
  return function (target: any, propertyKey: string) {
    Reflect.defineMetadata("http:method", "GET", target, propertyKey);
    Reflect.defineMetadata("http:path", path, target, propertyKey);
  };
}

function controller(prefix: string) {
  return function (constructor: new (...args: any[]) => any) {
    Reflect.defineMetadata("prefix", prefix, constructor);
  };
}

@controller("/api/users")
class UserController {
  @httpGet("/")
  getAll() { return []; }

  @httpGet("/:id")
  getById() { return {}; }
}

// Metadata o'qish:
const prefix = Reflect.getMetadata("prefix", UserController); // "/api/users"
const keys = Object.getOwnPropertyNames(UserController.prototype)
  .filter(k => k !== "constructor");

for (const key of keys) {
  const method = Reflect.getMetadata("http:method", UserController.prototype, key);
  const path = Reflect.getMetadata("http:path", UserController.prototype, key);
  console.log(`${method} ${prefix}${path} → ${key}`);
}
// GET /api/users/ → getAll
// GET /api/users/:id → getById
```

</details>

---

## `emitDecoratorMetadata` — Compiler Metadata Emit

### Nazariya

`emitDecoratorMetadata: true` (`experimentalDecorators: true` bilan birga) — kompilator **avtomatik type metadata** emit qiladi:

| Key | Nima saqlaydi | Qayerda |
|-----|---------------|---------|
| `"design:type"` | Property/parameter type | Property decorator |
| `"design:paramtypes"` | Constructor/method parameter type lar (array) | Class/method decorator |
| `"design:returntype"` | Method return type | Method decorator |

**Muhim:** Faqat **legacy** decorator lar bilan ishlaydi. TC39 da `Symbol.metadata` ishlatiladi ([Bo'lim 19](19-decorators.md#decorator-metadata--symbolmetadata)).

**Cheklov:** Interface lar compile-time da o'chiriladi — `design:paramtypes` da interface o'rniga `Object` yoziladi. Shuning uchun **token** system kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `emitDecoratorMetadata: true` bo'lganda har decorated element uchun `__metadata` chaqiruvlarini emit qiladi:

```typescript
// TypeScript source:
@injectable
class UserService {
  constructor(private logger: Logger, private db: Database) {}
}
```

```javascript
// Compiled JS:
var UserService = __decorate([
  injectable,
  __metadata("design:paramtypes", [Logger, Database])
], UserService);
```

`__metadata("design:paramtypes", [Logger, Database])` — constructor parameter type larini class reference array sifatida saqlaydi. DI container bu array ni `Reflect.getMetadata("design:paramtypes", UserService)` bilan o'qiydi.

**Interface muammosi:**

```typescript
constructor(private logger: ILogger) {}
// Compiled: __metadata("design:paramtypes", [Object])
// ILogger — yo'qoldi! Object qoldi.
// Yechim: token system (@inject(LOGGER_TOKEN))
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
import "reflect-metadata";

function injectable(constructor: new (...args: any[]) => any) {}

class Logger { log(msg: string) { console.log(msg); } }
class Database { query(sql: string) { return []; } }

@injectable
class UserService {
  constructor(private logger: Logger, private db: Database) {}
}

const paramTypes = Reflect.getMetadata("design:paramtypes", UserService);
console.log(paramTypes); // [class Logger, class Database]
```

</details>

---

## Dependency Injection Nima — IoC

### Nazariya

**Dependency Injection (DI)** — class o'z dependency larini **o'zi yaratmaydi**, balki **tashqaridan oladi**. Bu **Inversion of Control (IoC)** prinsipining implementatsiyasi.

```typescript
// ❌ DI siz — tight coupling
class UserService {
  private logger = new ConsoleLogger(); // O'zi yaratdi
  private db = new PostgresDB();        // O'zi yaratdi
}

// ✅ DI bilan — loose coupling
class UserService {
  constructor(
    private logger: ILogger,  // Tashqaridan berildi
    private db: IDatabase     // Tashqaridan berildi
  ) {}
}
```

**DI ning afzalliklari:**

1. **Loose coupling** — class aniq implementatsiyaga bog'lanmaydi
2. **Testability** — test da mock inject qilish oson
3. **Configurability** — runtime da implementatsiyani almashtirish mumkin
4. **Single Responsibility** — class faqat o'z ishini qiladi

**DI Container** — dependency larni register, resolve, va lifecycle boshqaradigan markaziy object.

---

## DI Patterns — Constructor, Property, Method Injection

### Nazariya

DI ning uchta asosiy pattern i:

1. **Constructor Injection** (eng ko'p ishlatiladi, tavsiya) — dependency lar constructor orqali
2. **Property Injection** — dependency class property ga assign
3. **Method Injection** — dependency method parameter orqali

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === 1. Constructor Injection (tavsiya) ===
class UserService {
  constructor(
    private readonly logger: ILogger,
    private readonly db: IDatabase
  ) {}
  getUsers() {
    this.logger.log("Getting users");
    return this.db.query("SELECT * FROM users");
  }
}

// === 2. Property Injection ===
class UserService {
  logger!: ILogger;   // Decorator yoki container orqali set
  db!: IDatabase;
}

// === 3. Method Injection ===
class ReportGenerator {
  generate(formatter: IFormatter, data: Data[]) {
    return formatter.format(data);
  }
}
```

**Constructor injection nima uchun yaxshiroq:**
- Dependency lar **immutable** (`readonly`)
- Class yaratilganda **barcha** dependency lar mavjud
- Test da mock berish oson
- `!` assertion kerak emas

</details>

---

## DI Container Yaratish — Step by Step

### Nazariya

DI Container — dependency larni boshqaradigan markaziy class:

1. **Register** — class ni container ga ro'yxatga olish
2. **Resolve** — class yaratish + dependency inject
3. **Scope** — singleton (bitta), transient (har safar yangi)

Bu container `reflect-metadata` va `emitDecoratorMetadata` ga asoslangan — `design:paramtypes` orqali constructor parameter type larni avtomatik o'qiydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

**Oddiy auto-resolve container:**

```typescript
import "reflect-metadata";

function injectable(constructor: new (...args: any[]) => any) {
  Reflect.defineMetadata("injectable", true, constructor);
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

@injectable class Logger { log(msg: string) { console.log(`[LOG] ${msg}`); } }
@injectable class Database {
  constructor(private logger: Logger) {}
  query(sql: string) { this.logger.log(`SQL: ${sql}`); return []; }
}
@injectable class UserService {
  constructor(private db: Database, private logger: Logger) {}
  getUsers() {
    this.logger.log("Getting users");
    return this.db.query("SELECT * FROM users");
  }
}

const container = new Container();
const userService = container.resolve(UserService);
userService.getUsers();
// [LOG] Getting users
// [LOG] SQL: SELECT * FROM users
```

**Token-based container (interface lar uchun):**

```typescript
import "reflect-metadata";

const INJECT_KEY = Symbol("inject");

function inject(token: any) {
  return function (target: any, _: any, index: number) {
    const tokens = Reflect.getOwnMetadata(INJECT_KEY, target) || new Map();
    tokens.set(index, token);
    Reflect.defineMetadata(INJECT_KEY, tokens, target);
  };
}

class TokenContainer {
  private bindings = new Map<any, { cls: new (...args: any[]) => any; scope: string }>();
  private singletons = new Map<any, any>();

  bind(token: any) {
    return {
      to: (cls: new (...args: any[]) => any) => ({
        asSingleton: () => { this.bindings.set(token, { cls, scope: "singleton" }); },
        asTransient: () => { this.bindings.set(token, { cls, scope: "transient" }); },
      }),
    };
  }

  resolve<T>(token: any): T {
    const binding = this.bindings.get(token);
    if (!binding) throw new Error(`No binding: ${String(token)}`);

    if (binding.scope === "singleton" && this.singletons.has(token)) {
      return this.singletons.get(token);
    }

    const paramTypes: any[] = Reflect.getMetadata("design:paramtypes", binding.cls) || [];
    const injectedTokens: Map<number, any> = Reflect.getOwnMetadata(INJECT_KEY, binding.cls) || new Map();

    const deps = paramTypes.map((type: any, i: number) => {
      return this.resolve(injectedTokens.get(i) || type);
    });

    const instance = new binding.cls(...deps) as T;
    if (binding.scope === "singleton") this.singletons.set(token, instance);
    return instance;
  }
}
```

</details>

---

## DI va Testing

### Nazariya

DI ning eng katta afzalligi — **testability**. Real dependency o'rniga mock inject qilib, tez va izolyatsiyalangan test yozish mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface IEmailService { send(to: string, body: string): Promise<void>; }
interface IUserRepo { save(user: User): Promise<void>; findByEmail(email: string): Promise<User | null>; }

class UserService {
  constructor(private email: IEmailService, private repo: IUserRepo) {}

  async register(name: string, email: string): Promise<User> {
    const existing = await this.repo.findByEmail(email);
    if (existing) throw new Error("Email already exists");
    const user = { id: crypto.randomUUID(), name, email };
    await this.repo.save(user);
    await this.email.send(email, `Welcome, ${name}!`);
    return user;
  }
}

// Test — mock inject:
describe("UserService", () => {
  let service: UserService;
  let mockEmail: jest.Mocked<IEmailService>;
  let mockRepo: jest.Mocked<IUserRepo>;

  beforeEach(() => {
    mockEmail = { send: jest.fn().mockResolvedValue(undefined) };
    mockRepo = { save: jest.fn().mockResolvedValue(undefined), findByEmail: jest.fn().mockResolvedValue(null) };
    service = new UserService(mockEmail, mockRepo);
  });

  it("should register new user", async () => {
    const user = await service.register("Ali", "ali@test.com");
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
    expect(mockEmail.send).toHaveBeenCalledWith("ali@test.com", "Welcome, Ali!");
  });

  it("should reject duplicate email", async () => {
    mockRepo.findByEmail.mockResolvedValue({ id: "1", name: "X", email: "ali@test.com" });
    await expect(service.register("Ali", "ali@test.com")).rejects.toThrow("Email already exists");
    expect(mockRepo.save).not.toHaveBeenCalled();
  });
});
```

</details>

---

## Decorator-based vs Function-based DI

### Nazariya

**Decorator-based DI** — `@injectable`, `@inject`, `reflect-metadata`. Legacy decorator larga bog'liq.

**Function-based DI** — decorator va metadata kerak emas. Dependency lar explicit funksiya parameter lari orqali beriladi.

| Xususiyat | Decorator-based | Function-based |
|-----------|----------------|----------------|
| Runtime overhead | Metadata + polyfill | Yo'q |
| TC39 mos | ❌ (legacy kerak) | ✅ |
| Type safety | Token system kerak | Natural |
| Tree-shaking | Qiyinroq | Osonroq |

TC39 decorator larda `emitDecoratorMetadata` **ishlamaydi** va parameter decorator **yo'q**. Function-based DI TC39 bilan to'liq mos.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Decorator-based (legacy) ===
@injectable()
class UserService {
  constructor(@inject("DB") private db: IDatabase) {}
}
// Kerak: reflect-metadata, experimentalDecorators, emitDecoratorMetadata

// === Function-based (zamonaviy) ===
function createUserService(db: IDatabase): UserService {
  return { getUsers: () => db.query("SELECT * FROM users") };
}
// Kerak: hech narsa — oddiy funksiya

// === Factory pattern (DI siz ham ishlaydi) ===
function createApp(config: AppConfig) {
  const logger = new ConsoleLogger(config.logLevel);
  const db = new PostgresDB(config.dbUrl);
  const userService = new UserService(logger, db);
  return { userService };
}
```

</details>

---

## Edge Cases va Gotchas

### 1. `design:paramtypes` da interface `Object` ga aylanadi

```typescript
interface ILogger { log(msg: string): void; }

@injectable
class Service { constructor(private logger: ILogger) {} }

const types = Reflect.getMetadata("design:paramtypes", Service);
console.log(types); // [Object] — ILogger yo'q!
// Yechim: @inject(LOGGER_TOKEN) bilan token
```

### 2. Circular dependency — DI container da runtime error

```typescript
@injectable class A { constructor(private b: B) {} }
@injectable class B { constructor(private a: A) {} }

container.resolve(A); // ❌ Infinite loop yoki stack overflow!
// Yechim: lazy injection yoki dependency ni ajratish
```

### 3. `emitDecoratorMetadata` faqat legacy bilan ishlaydi

```typescript
// TC39 decorator lar bilan (experimentalDecorators: false)
// emitDecoratorMetadata: true HECH NARSA QILMAYDI
// design:paramtypes metadata YO'Q
// Yechim: Symbol.metadata yoki function-based DI
```

### 4. `reflect-metadata` import tartibi muhim

```typescript
// ❌ — metadata API mavjud emas
@injectable class MyClass {}
import "reflect-metadata"; // Juda kech!

// ✅ — app entry point boshida
import "reflect-metadata"; // BIRINCHI
```

### 5. Singleton scope — test larda state leak

```typescript
container.bind(Logger).asSingleton();
// Test 1: logger.count = 5;
// Test 2: logger.count hali 5! — oldingi test state i

// Yechim: har test da yangi container
beforeEach(() => { container = new Container(); });
```

---

## Common Mistakes

### ❌ Xato 1: `@injectable` ni unutish

```typescript
class UserService { constructor(private logger: Logger) {} }
Reflect.getMetadata("design:paramtypes", UserService);
// undefined — decorator yo'q → metadata emit bo'lmaydi!
```

### ❌ Xato 2: Interface uchun token ishlatmaslik

```typescript
// ❌ — interface Object ga aylanadi
constructor(private logger: ILogger) {}
// design:paramtypes → [Object]

// ✅ — token
constructor(@inject(LOGGER_TOKEN) private logger: ILogger) {}
```

### ❌ Xato 3: `reflect-metadata` import qilmaslik

```typescript
// ❌
Reflect.getMetadata("design:paramtypes", MyClass);
// TypeError: Reflect.getMetadata is not a function

// ✅
import "reflect-metadata"; // app.ts boshida
```

### ❌ Xato 4: Circular dependency ni handle qilmaslik

```typescript
// ❌ — stack overflow
@injectable class A { constructor(private b: B) {} }
@injectable class B { constructor(private a: A) {} }

// ✅ — dependency ni ajratish yoki lazy injection
```

### ❌ Xato 5: Singleton ni test larda reset qilmaslik

```typescript
// ❌ — test lar bir-birining state ini ko'radi
// ✅ — har test da yangi container
beforeEach(() => { container = new Container(); });
```

---

## Amaliy Mashqlar

### Mashq 1: Oddiy DI Container (Oson)

**Savol:** `reflect-metadata` bilan auto-resolve container yozing.

<details>
<summary>Javob</summary>

```typescript
import "reflect-metadata";

function injectable(constructor: new (...args: any[]) => any) {}

class Container {
  private instances = new Map<any, any>();

  resolve<T>(target: new (...args: any[]) => T): T {
    if (this.instances.has(target)) return this.instances.get(target);
    const deps = (Reflect.getMetadata("design:paramtypes", target) || [])
      .map((dep: any) => this.resolve(dep));
    const instance = new target(...deps);
    this.instances.set(target, instance);
    return instance;
  }
}
```

</details>

---

### Mashq 2: Scope Support (O'rta)

**Savol:** Container ga singleton va transient scope qo'shing.

<details>
<summary>Javob</summary>

```typescript
type Scope = "singleton" | "transient";

class ScopedContainer {
  private bindings = new Map<any, { cls: new (...args: any[]) => any; scope: Scope }>();
  private singletons = new Map<any, any>();

  register(token: any, cls: new (...args: any[]) => any, scope: Scope = "transient") {
    this.bindings.set(token, { cls, scope });
  }

  resolve<T>(token: any): T {
    const binding = this.bindings.get(token);
    if (!binding) throw new Error(`No binding for ${String(token)}`);
    if (binding.scope === "singleton" && this.singletons.has(token)) return this.singletons.get(token);

    const deps = (Reflect.getMetadata("design:paramtypes", binding.cls) || [])
      .map((dep: any) => this.resolve(dep));
    const instance = new binding.cls(...deps) as T;
    if (binding.scope === "singleton") this.singletons.set(token, instance);
    return instance;
  }
}
```

</details>

---

### Mashq 3: Token-based @inject (O'rta)

**Savol:** `@inject(token)` parameter decorator yozing.

<details>
<summary>Javob</summary>

```typescript
const INJECT_KEY = Symbol("inject");

function inject(token: any) {
  return function (target: any, _: any, index: number) {
    const tokens = Reflect.getOwnMetadata(INJECT_KEY, target) || new Map();
    tokens.set(index, token);
    Reflect.defineMetadata(INJECT_KEY, tokens, target);
  };
}
```

</details>

---

### Mashq 4: useFactory Provider (Qiyin)

**Savol:** Container ga `useFactory` support qo'shing.

<details>
<summary>Javob</summary>

```typescript
type Provider =
  | { useClass: new (...args: any[]) => any; scope?: Scope }
  | { useValue: any }
  | { useFactory: (container: FactoryContainer) => any; scope?: Scope };

class FactoryContainer {
  private providers = new Map<any, Provider>();
  private singletons = new Map<any, any>();

  register(token: any, provider: Provider) { this.providers.set(token, provider); }

  resolve<T>(token: any): T {
    const p = this.providers.get(token);
    if (!p) throw new Error(`No provider: ${String(token)}`);
    if ("useValue" in p) return p.useValue;

    const scope = "scope" in p ? p.scope : "transient";
    if (scope === "singleton" && this.singletons.has(token)) return this.singletons.get(token);

    const instance = "useFactory" in p
      ? p.useFactory(this)
      : (() => {
          const deps = (Reflect.getMetadata("design:paramtypes", p.useClass) || [])
            .map((d: any) => this.resolve(d));
          return new p.useClass(...deps);
        })();

    if (scope === "singleton") this.singletons.set(token, instance as T);
    return instance as T;
  }
}
```

</details>

---

### Mashq 5: DI + Testing (O'rta)

**Savol:** `UserService` uchun DI bilan unit test yozing — mock email va mock repo.

<details>
<summary>Javob</summary>

```typescript
interface IEmailService { send(to: string, body: string): Promise<void>; }
interface IUserRepo { save(user: any): Promise<void>; findByEmail(email: string): Promise<any>; }

class UserService {
  constructor(private email: IEmailService, private repo: IUserRepo) {}
  async register(name: string, email: string) {
    const existing = await this.repo.findByEmail(email);
    if (existing) throw new Error("Email exists");
    const user = { name, email };
    await this.repo.save(user);
    await this.email.send(email, `Welcome, ${name}!`);
    return user;
  }
}

// Test:
const mockEmail: IEmailService = { send: jest.fn().mockResolvedValue(undefined) };
const mockRepo: IUserRepo = { save: jest.fn().mockResolvedValue(undefined), findByEmail: jest.fn().mockResolvedValue(null) };
const service = new UserService(mockEmail, mockRepo);

await service.register("Ali", "ali@test.com");
expect(mockRepo.save).toHaveBeenCalled();
expect(mockEmail.send).toHaveBeenCalledWith("ali@test.com", "Welcome, Ali!");
```

</details>

---

## Xulosa

**Reflect Metadata:**
- `reflect-metadata` polyfill — WeakMap asosida metadata saqlash
- `emitDecoratorMetadata` — kompilator `design:type`, `design:paramtypes`, `design:returntype` emit qiladi
- **Faqat legacy** decorator lar bilan ishlaydi. TC39 da `Symbol.metadata`

**DI asoslari:**
- **IoC** — class dependency larini o'zi yaratmaydi, tashqaridan oladi
- **Constructor injection** — eng yaxshi pattern
- **DI Container** — register, resolve, scope (singleton/transient)
- **Token system** — interface lar uchun (interface `Object` ga aylanadi)

**Testing:** DI ning eng katta afzalligi — mock inject qilish osonligi

**Trend:** Decorator-based DI → function-based DI (TC39 bilan mos, tree-shakeable)

**Bog'liq bo'limlar:**
- [Bo'lim 19: Decorators](19-decorators.md) — decorator syntax va mexanizm
- [Bo'lim 21: Design Patterns](21-design-patterns.md) — IoC, Factory, Strategy patterns

---

**Keyingi bo'lim:** [21-design-patterns.md](21-design-patterns.md) — Design Patterns TypeScript da — Creational, Structural, Behavioral pattern lar, har birining type-safe implementatsiyasi.
