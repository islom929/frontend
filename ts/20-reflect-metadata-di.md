# Bo'lim 20: Reflect Metadata va Dependency Injection

> Reflect Metadata API — TypeScript'da decorator'lar bilan birgalikda metadata saqlash va o'qish mexanizmi. `reflect-metadata` package orqali class, method, property va parameter'larga runtime'da qo'shimcha ma'lumot biriktirish mumkin. `emitDecoratorMetadata` compiler option yoqilganda, TypeScript o'zi ham type ma'lumotlarini metadata sifatida emit qiladi (`design:type`, `design:paramtypes`, `design:returntype`). Bu metadata Dependency Injection (DI) pattern'ning asosiy qismini tashkil etadi (Angular, NestJS, TypeORM).

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

Reflect Metadata API — `reflect-metadata` polyfill package orqali JavaScript runtime'ga `Reflect.defineMetadata`/`Reflect.getMetadata` method'larini qo'shadigan API. Object'larga (class, prototype, method, property) arbitrary key-value metadata biriktirib, keyin runtime'da o'qish imkonini beradi.

Metadata — object haqida qo'shimcha (associative) ma'lumot. Misol: "bu method qaysi HTTP method'ga to'g'ri keladi", "bu class qaysi DI scope'da", "bu constructor parameter qaysi class type'da". Bu ma'lumotlar **runtime'da** o'qiladi, shu sababli compilation'dan keyin ham yo'qolmaydi.

Tarixiy kontekst: `reflect-metadata` paketdagi Metadata Reflection API rasmiy TC39 proposal sifatida hech qachon stage'lardan o'tmagan — bu prototip taklif edi. Decorators va Decorator Metadata Stage 3 ga yetgach, paket README'sida ko'rsatilganidek bu API "endi standartlashtirish uchun ko'rib chiqilmaydi" deb belgilangan; uning o'rnini Stage 3 decorators bilan keladigan `Symbol.metadata` (TS 5.2+) egalladi. Lekin Angular/NestJS/TypeORM ekosistemasi hali `experimentalDecorators` + `reflect-metadata` polyfill'iga asoslangan — shu sababli amaliyotda ikkala mexanizm parallel mavjud.

```bash
npm install reflect-metadata
```

Asosiy API:

| Method | Vazifasi |
|--------|---------|
| `Reflect.defineMetadata(key, value, target)` | Target ga metadata yozish |
| `Reflect.defineMetadata(key, value, target, propertyKey)` | Property/method'ga metadata yozish |
| `Reflect.getMetadata(key, target)` | Metadata o'qish (prototype chain bo'ylab) |
| `Reflect.getOwnMetadata(key, target)` | Faqat o'z metadata (prototype chain ga qaramaydi) |
| `Reflect.hasMetadata(key, target)` | Metadata borligini tekshirish (chain bo'ylab) |
| `Reflect.hasOwnMetadata(key, target)` | Faqat o'z metadata borligi |
| `Reflect.deleteMetadata(key, target)` | Metadata o'chirish |
| `Reflect.getMetadataKeys(target)` | Barcha metadata key'lar (chain bilan) |

`getMetadata` vs `getOwnMetadata` — birinchisi prototype chain bo'ylab qidiradi (parent class metadata'sini ham topadi), ikkinchisi faqat o'z target'ni tekshiradi. DI container'da `getOwnMetadata` xavfsizroq — inheritance'dan kelgan metadata bilan adashish bo'lmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Polyfill ichida WeakMap'lar hierarxiyasi:

```text
Metadata Storage:
  WeakMap<target, Map<propertyKey | undefined, Map<metadataKey, value>>>
   │           │                                  │
   │           │                                  └─ key=value pair
   │           └─ class/prototype/method aniqlovchisi
   └─ class/prototype reference (GC qilinsa metadata ham GC bo'ladi)
```

**Nima uchun WeakMap:** target reference ushlanmaydi — target garbage collect bo'lganda metadata avtomatik bo'shaydi (memory leak yo'q).

**`defineMetadata` algoritmi (soddalashtirilgan):**

```text
1. target uchun outer Map'ni ol (yoki yarat)
2. propertyKey uchun inner Map'ni ol (yoki yarat); class darajasi uchun key = undefined
3. inner Map'ga (metadataKey → value) yoz
```

**`getMetadata` prototype walk:**

```text
1. target'ning OwnMetadata'sini tekshir → topilsa qaytar
2. Object.getPrototypeOf(target) → null bo'lsa undefined
3. parent target uchun 1-qadamga qayt (rekursiv yuqoriga)
```

Bu sababli inheritance chain'da parent class'ga qo'yilgan `@injectable` metadata child class'da ham `getMetadata` orqali ko'rinadi — DI container avtomatik resolve qila oladi.

</details>

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

`emitDecoratorMetadata: true` (`experimentalDecorators: true` bilan birga) — compiler har decorated element uchun **avtomatik type metadata** emit qiladi (`reflect-metadata` polyfill API'siga `__metadata(...)` chaqiruvi sifatida):

| Key | Nima saqlaydi | Qachon emit qilinadi |
|-----|---------------|---------|
| `"design:type"` | Property/parameter type (class reference) | Property/accessor da decorator bo'lsa |
| `"design:paramtypes"` | Constructor/method parameter type'lar massivi | Class yoki method'da decorator bo'lsa |
| `"design:returntype"` | Method return type | Method'da decorator bo'lsa |

**Cheklov 1 — faqat legacy decorator:** `emitDecoratorMetadata` faqat `experimentalDecorators: true` (legacy Stage 2 decorator) bilan ishlaydi. TC39 Stage 3 decorator'lar bilan flag hech narsa qilmaydi — compiler metadata emit qilmaydi. Zamonaviy yo'l — Stage 3 `Symbol.metadata` (TS 5.2+, [Bo'lim 19](19-decorators.md)).

**Cheklov 2 — interface erasure:** Interface'lar compile paytida o'chiriladi (type-only) — `design:paramtypes` array'iga interface o'rniga `Object` constructor yoziladi. Misol: `constructor(private logger: ILogger)` compilation'dan keyin `__metadata("design:paramtypes", [Object])` bo'ladi. Yechim — `@inject(LOGGER_TOKEN)` parameter decorator bilan explicit token berish (token system).

**Cheklov 3 — kamida bitta decorator kerak:** Compiler `__metadata` chaqiruvini faqat class/method/property'da kamida bitta decorator bo'lganda emit qiladi. Decorator'siz class'da `design:paramtypes` yo'q — DI container `undefined` oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler `emitDecoratorMetadata: true` bo'lganda har decorated element uchun `__metadata` chaqiruvlarini emit qiladi:

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

`__metadata("design:paramtypes", [Logger, Database])` — constructor parameter type'larini class reference array sifatida saqlaydi. DI container bu array'ni `Reflect.getMetadata("design:paramtypes", UserService)` bilan o'qiydi.

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

**Dependency Injection (DI)** — class o'z dependency'larini **o'zi yaratmaydi**, balki tashqaridan oladi (constructor, property yoki method orqali). DI — **Inversion of Control (IoC)** prinsipining konkret implementation'i: dependency yaratish va ulash mas'uliyati class'dan tashqi composition kodiga ko'chiriladi.

**IoC ma'nosi:** an'anaviy oqimda yuqori darajadagi module pastki module'ni `new` orqali yaratadi (control yuqoridan pastga). IoC oqimida control teskari — pastki abstraction'lar tashqaridan inject qilinadi, yuqori module faqat interface'larga bog'liq bo'ladi.

```typescript
// ❌ DI siz — tight coupling, hard-coded dependency
class UserService {
  private logger = new ConsoleLogger();   // Konkret class
  private db = new PostgresDB("...");     // Konkret class + config
  // Test da mock berish mumkin emas; PostgresDB ni MySQLDB ga almashtirish — kod o'zgaradi
}

// ✅ DI bilan — loose coupling, abstraction'ga bog'lanish
class UserService {
  constructor(
    private logger: ILogger,    // Interface — har qanday implementation
    private db: IDatabase       // Interface — runtime'da almashtirish mumkin
  ) {}
}
```

**DI ning amaliy afzalliklari:**

1. **Loose coupling** — class abstraction'ga (interface) bog'lanadi, konkret class'ga emas
2. **Testability** — unit test'da mock/stub inject qilish oson (real DB/HTTP kerakmas)
3. **Configurability** — production'da `PostgresDB`, dev'da `InMemoryDB` — kod o'zgarmaydi, faqat container binding
4. **Single Responsibility** — class faqat business logic'ni bajaradi, dependency lifecycle'ni emas
5. **Composability** — dependency graph'i bitta joyda (composition root) tasvirlangan

**DI Container** — dependency'larni register, resolve, va lifecycle (singleton/transient/request scope) boshqaradigan markaziy object. Container metadata'dan foydalanib (yoki explicit configuration'dan) dependency graph'ni avtomatik quradi.

---

## DI Patterns — Constructor, Property, Method Injection

### Nazariya

DI'da dependency'larni class'ga "yetkazib berish"ning uchta usuli mavjud — har birining trade-off'lari farqli:

1. **Constructor Injection** — dependency'lar constructor parameter orqali. Class instantiate qilinganda barcha dependency'lar majburiy. Eng tavsiya etilgan.
2. **Property Injection** (Setter Injection) — dependency'lar public property/setter orqali, instantiate'dan keyin set qilinadi. Optional dependency yoki circular dependency'ni hal qilish uchun.
3. **Method Injection** — dependency method'ning bir martalik parametri sifatida. Class'da saqlanmaydi, faqat shu method chaqirig'ida ishlatiladi.

**Constructor injection nima uchun default tanlov:**
- Dependency'lar `readonly` qilib markalanadi — immutability kafolati
- Class yaratilgan paytda barcha dependency'lar mavjudligi compile-time'da tasdiqlanadi (no null state)
- TypeScript `strictPropertyInitialization`'da `!` assertion kerak emas
- Test'da mock berish — bitta `new Service(mock1, mock2)` chaqirig'i

**Property injection kamchiligi:** instantiate va dependency set'i orasida class "yarim ishchi" holatda bo'ladi — bu vaqtda method chaqirilsa `undefined` access bo'lishi mumkin. `!` definite-assignment assertion compiler'ni jim qiladi, lekin runtime safety bermaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === 1. Constructor Injection (default tanlov) ===
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

// Instantiate — barcha dependency'lar majburiy
const service = new UserService(new ConsoleLogger(), new PostgresDB());

// === 2. Property Injection ===
class UserServiceProp {
  // ! — definite assignment assertion (initializer bermayman, lekin set qilinadi)
  logger!: ILogger;
  db!: IDatabase;

  getUsers() {
    this.logger.log("Getting users");
    return this.db.query("SELECT * FROM users");
  }
}

const propService = new UserServiceProp();
propService.logger = new ConsoleLogger();   // Set qilish — alohida qadam
propService.db = new PostgresDB();
// Agar set qilishni unutsa — runtime'da `Cannot read property 'log' of undefined`

// === 3. Method Injection ===
class ReportGenerator {
  // Dependency saqlanmaydi — har chaqiriqda berish
  generate(formatter: IFormatter, data: ReportData[]) {
    return formatter.format(data);
  }
}

const generator = new ReportGenerator();
generator.generate(new PdfFormatter(), data);    // Bir martalik formatter
generator.generate(new HtmlFormatter(), data);   // Boshqa chaqiriqda boshqa
```

**Qachon qaysi pattern:**

| Pattern | Qachon |
|---------|--------|
| Constructor | Default tanlov — required, instance lifetime davomida o'zgarmas dependency |
| Property | Circular dependency, optional dependency, framework-managed (Angular `@Input`) |
| Method | Per-call strategy (formatter, validator) — har chaqiriqda boshqa implementation |

</details>

---

## DI Container Yaratish — Step by Step

### Nazariya

DI Container — dependency'larni register qilib, resolve paytida graph'ni avtomatik quradigan markaziy class. Asosiy mas'uliyatlari:

1. **Register** — token (class yoki Symbol) ga implementation bog'lash
2. **Resolve** — token bo'yicha instance qaytarish, kerakli barcha dependency'larni rekursiv tarzda yaratish
3. **Scope (lifecycle)** — `singleton` (bitta instance umrga), `transient` (har resolve'da yangi), `scoped` (request/session davomida bitta)

Container `reflect-metadata` va `emitDecoratorMetadata`'ga asoslanadi — TypeScript compiler har decorated class uchun `design:paramtypes` metadata'ni emit qiladi, bunda constructor parameter'larining class reference'lari array tarzida saqlanadi. Container shu array'ni o'qib, har bir parameter uchun rekursiv `resolve` chaqiradi va constructor argumentlarini quradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Resolve algoritmi (rekursiv graph build):**

```text
resolve(Target):
  1. Singleton cache da Target bormi? → bor bo'lsa qaytar
  2. Reflect.getMetadata("design:paramtypes", Target) → [LoggerDep, DbDep, ...]
     (emitDecoratorMetadata sababli compiler bu array'ni emit qilgan)
  3. Har bir Dep uchun rekursiv resolve(Dep)
  4. new Target(...resolvedDeps) — instance yaratish
  5. Scope = singleton bo'lsa, cache'ga saqlash
  6. Instance'ni qaytarish
```

**Circular dependency muammosi:** A → B → A bo'lsa, 3-qadamda cheksiz rekursiya. Yechim — "resolving" set bilan tracking:

```text
resolving = Set<Target>
if (resolving.has(Target)) throw CircularDependencyError
resolving.add(Target)
... resolve dependencies ...
resolving.delete(Target)
```

NestJS, InversifyJS, tsyringe — bu detection'ni implement qilgan. Quyidagi misol oddiy variant, detection'siz.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Oddiy auto-resolve container:**

```typescript
import "reflect-metadata";

// @injectable — emitDecoratorMetadata ishlashi uchun class'ga decorator MAJBUR
// (kamida bitta decorator bo'lmasa, compiler design:paramtypes emit qilmaydi)
function injectable<T extends new (...args: any[]) => any>(constructor: T): T {
  Reflect.defineMetadata("injectable", true, constructor);
  return constructor;
}

class Container {
  private singletons = new Map<unknown, unknown>();

  resolve<T>(target: new (...args: any[]) => T): T {
    if (this.singletons.has(target)) return this.singletons.get(target) as T;

    // @injectable bilan markalanganligini tekshirish (xato erta tutiladi)
    if (!Reflect.getMetadata("injectable", target)) {
      throw new Error(`${target.name} is not @injectable`);
    }

    const paramTypes: Array<new (...args: any[]) => unknown> =
      Reflect.getMetadata("design:paramtypes", target) || [];
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

**Token-based container (interface'lar uchun):**

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

DI'ning eng katta amaliy afzalligi — **testability**. Class real dependency'larga emas, abstraction'larga (interface) bog'langanligi sababli, unit test'da real DB/HTTP/file system o'rniga mock implementation inject qilinadi. Bu uch foyda beradi: testlar **tez** (I/O yo'q), **deterministik** (tashqi state'ga bog'liq emas) va **isolated** (faqat shu class logic'i test'da).

Mocking strategiyalari:
- **Manual mock** — interface'ni qo'lda implement qilish (jest yoki test framework'siz)
- **`jest.Mocked<T>` / `vi.Mocked<T>`** — har method'ni avtomatik `jest.fn()`/`vi.fn()` qiladi, type-safe (interface'dan tashqari method qo'shilmaydi)
- **Spy on real implementation** — real instance'da bir method'ni override qilish (kamroq DI, ko'proq overkill)

Constructor injection bilan mock berish — bitta `new Service(mockA, mockB)` chaqirig'i. Container kerak emas, faqat type-compatible object yetarli.

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

**Decorator-based DI** — `@injectable`/`@inject` decorator'lar va `reflect-metadata` polyfill'iga asoslangan yondashuv. Container `design:paramtypes` metadata'ni o'qib, constructor parameter'larini avtomatik resolve qiladi. Angular, NestJS, TypeORM, InversifyJS, tsyringe — barchasi shu modelda. Cheklov: `experimentalDecorators: true` + `emitDecoratorMetadata: true` (legacy decorator) shart, TC39 Stage 3 decorator'larda `emitDecoratorMetadata` umuman ishlamaydi va parameter decorator'lar Stage 3'ga kiritilmagan.

**Function-based DI** — decorator/metadata umuman ishlatilmaydi. Dependency'lar oddiy function parameter'lari yoki factory'lar orqali explicit beriladi. Composition root (masalan `createApp` function'i) qo'lda yoziladi — kim kimga muhtoj ekanligi kodda ko'rinib turadi. Type system natural ishlaydi: interface'lar to'g'ridan-to'g'ri parameter type'i sifatida, token kerakmas.

| Xususiyat | Decorator-based | Function-based |
|-----------|----------------|----------------|
| Setup | `reflect-metadata` + tsconfig flag'lar | Hech narsa |
| Runtime overhead | Polyfill + metadata lookup | Yo'q |
| TC39 Stage 3 mos | Ishlamaydi (legacy kerak) | To'liq mos |
| Interface support | Token system shart (interface'lar erase bo'ladi) | Natural — parameter type sifatida |
| Bundle size | reflect-metadata polyfill qo'shiladi | Yo'q qo'shimcha |
| Tree-shaking | Qiyin (decorator side effect) | Tabiiy |
| Magic darajasi | Yuqori (avtomatik graph) | Past (explicit) |
| Framework mos | Angular/NestJS standart | React, plain TS |

TypeScript 5.0 (2023) standart Stage 3 decorator'larni `experimentalDecorators` flag'isiz qo'llab-quvvatlaydi; lekin `emitDecoratorMetadata` Stage 3 decorator'lar bilan **ishlamaydi** — `design:paramtypes` emit qilinmaydi. Parameter decorator (`@inject(token)` ko'rinishida) ham Stage 3'ga kiritilmagan. Shu sababli zamonaviy DI uchun `Symbol.metadata` (TS 5.2+) yoki function-based yondashuvga o'tish trend bo'lib boryapti.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
interface IDatabase { query(sql: string): unknown[]; }
interface ILogger { log(msg: string): void; }

// === Decorator-based (legacy) ===
@injectable()
class UserServiceDecorated {
  constructor(@inject("DB") private db: IDatabase) {}
  getUsers() { return this.db.query("SELECT * FROM users"); }
}
// Kerak: reflect-metadata, experimentalDecorators, emitDecoratorMetadata, @inject token

// === Function-based (zamonaviy) ===
interface IUserService {
  getUsers(): unknown[];
}

function createUserService(db: IDatabase): IUserService {
  return { getUsers: () => db.query("SELECT * FROM users") };
}
// Kerak: hech narsa — oddiy function, hech qanday metadata yo'q

// === Composition root (DI container'siz ham ishlaydi) ===
interface AppConfig { logLevel: "info" | "debug"; dbUrl: string; }

class ConsoleLogger implements ILogger {
  constructor(private level: AppConfig["logLevel"]) {}
  log(msg: string) { console.log(`[${this.level}] ${msg}`); }
}

class PostgresDB implements IDatabase {
  constructor(private url: string) {}
  query(sql: string): unknown[] { /* real client */ return []; }
}

function createApp(config: AppConfig) {
  // Dependency graph qo'lda quriladi — hech qanday magic yo'q
  const logger = new ConsoleLogger(config.logLevel);
  const db = new PostgresDB(config.dbUrl);
  const userService = createUserService(db);
  return { userService, logger };
}

const app = createApp({ logLevel: "info", dbUrl: "postgres://localhost/app" });
app.userService.getUsers();
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

### 2. Circular dependency — DI container'da runtime error

```typescript
@injectable class OrderService { constructor(private invoice: InvoiceService) {} }
@injectable class InvoiceService { constructor(private order: OrderService) {} }

container.resolve(OrderService); // ❌ Stack overflow — resolve cheksiz rekursiya
// Yechim: lazy injection yoki dependency'ni ajratish
```

### 3. `emitDecoratorMetadata` faqat legacy bilan ishlaydi

```typescript
// TC39 decorator'lar bilan (experimentalDecorators: false)
// emitDecoratorMetadata: true HECH NARSA QILMAYDI
// design:paramtypes metadata YO'Q
// Yechim: Symbol.metadata yoki function-based DI
```

### 4. `reflect-metadata` import tartibi muhim

```typescript
// entry.ts — boshqa hamma narsadan oldin polyfill yuklanishi shart
import "reflect-metadata";   // ✅ BIRINCHI import — Reflect.defineMetadata global'da o'rnatiladi
import { Container } from "./container";
import { UserService } from "./user-service";

// ❌ — boshqa modul "reflect-metadata"'dan oldin yuklansa,
//      o'sha modul evaluate bo'lganda Reflect.defineMetadata hali yo'q
```

Module darajasida `import` declaration'lar evaluate'dan oldin tepaga hoist qilinadi, shu sababli bitta fayl ichida `import` qatori qayerda turishi farq qilmaydi. Muammo modullar orasida: polyfill'ni import qiluvchi modul, decorator metadata'ga tayanadigan modullardan **oldin** evaluate bo'lishi kerak. Yagona kafolatli yo'l — entry point'ning birinchi statement'i `import "reflect-metadata"`.

### 5. Singleton scope — test'larda state leak

```typescript
container.bind(LOGGER_TOKEN).to(Logger).asSingleton();
const logger = container.resolve<Logger>(LOGGER_TOKEN);
// Test 1: logger.count = 5;
// Test 2: bir xil container — resolve o'sha singleton'ni qaytaradi, count hali 5

// Yechim: har test'da yangi container
beforeEach(() => { container = new TokenContainer(); });
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

### ❌ Xato 4: Circular dependency'ni handle qilmaslik

```typescript
// ❌ — stack overflow
@injectable class OrderService { constructor(private invoice: InvoiceService) {} }
@injectable class InvoiceService { constructor(private order: OrderService) {} }

// ✅ — dependency'ni ajratish yoki lazy injection
```

### ❌ Xato 5: Singleton'ni test'larda reset qilmaslik

```typescript
// ❌ — test'lar bir-birining state'ini ko'radi
// ✅ — har test'da yangi container
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
interface User { id: string; name: string; email: string; }
interface IEmailService { send(to: string, body: string): Promise<void>; }
interface IUserRepo { save(user: User): Promise<void>; findByEmail(email: string): Promise<User | null>; }

class UserService {
  constructor(private email: IEmailService, private repo: IUserRepo) {}
  async register(name: string, email: string): Promise<User> {
    const existing = await this.repo.findByEmail(email);
    if (existing) throw new Error("Email exists");
    const user: User = { id: crypto.randomUUID(), name, email };
    await this.repo.save(user);
    await this.email.send(email, `Welcome, ${name}!`);
    return user;
  }
}

// Test:
const mockEmail: IEmailService = { send: jest.fn().mockResolvedValue(undefined) };
const mockRepo: IUserRepo = {
  save: jest.fn().mockResolvedValue(undefined),
  findByEmail: jest.fn().mockResolvedValue(null),
};
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
- `emitDecoratorMetadata` — compiler `design:type`, `design:paramtypes`, `design:returntype` emit qiladi
- **Faqat legacy** decorator'lar bilan ishlaydi. TC39'da `Symbol.metadata`

**DI asoslari:**
- **IoC** — class dependency'larini o'zi yaratmaydi, tashqaridan oladi
- **Constructor injection** — eng yaxshi pattern
- **DI Container** — register, resolve, scope (singleton/transient)
- **Token system** — interface'lar uchun (interface `Object`'ga aylanadi)

**Testing:** DI ning eng katta afzalligi — mock inject qilish osonligi

**Trend:** Decorator-based DI → function-based DI (TC39 bilan mos, tree-shakeable)

**Bog'liq bo'limlar:**
- [Bo'lim 19: Decorators](19-decorators.md) — decorator syntax va mexanizm
- [Bo'lim 21: Design Patterns](21-design-patterns.md) — IoC, Factory, Strategy patterns

---

**Keyingi bo'lim:** [21-design-patterns.md](21-design-patterns.md) — Design Patterns TypeScript'da — Creational, Structural, Behavioral pattern'lar, har birining type-safe implementation'i.
