# Interview: Reflect Metadata va Dependency Injection

> Reflect Metadata API, `emitDecoratorMetadata`, DI/IoC tushunchalari, injection turlari, scope lar, DI Container noldan yozish va DI bilan testing bo'yicha interview savollari. Core TypeScript — framework-specific emas.

---

## Nazariy savollar

### 1. Reflect Metadata API nima? Nima uchun kerak?

<details>
<summary>Javob</summary>

Reflect Metadata — `reflect-metadata` polyfill orqali runtime da object larga **metadata** biriktirish va o'qish API. WeakMap asosida ishlaydi:

```typescript
import "reflect-metadata";

class UserService {
  findAll(): string[] { return ["user1"]; }
}

// Metadata yozish
Reflect.defineMetadata("role", "service", UserService);
Reflect.defineMetadata("http:method", "GET", UserService.prototype, "findAll");

// Metadata o'qish
Reflect.getMetadata("role", UserService);                        // "service"
Reflect.getMetadata("http:method", UserService.prototype, "findAll"); // "GET"
```

**Holat:** Tarixan TC39 Stage 2 proposal edi, lekin **to'xtab qoldi**. Hozir **TC39 Decorator Metadata** (`Symbol.metadata`) o'rnini bosmoqda. Legacy framework lar (Angular, NestJS) hali `reflect-metadata` ishlatadi.

</details>

### 2. `emitDecoratorMetadata` nima? Compiler qanday type metadata emit qiladi?

<details>
<summary>Javob</summary>

`emitDecoratorMetadata: true` — compiler har decorate qilingan element uchun avtomatik **type metadata** emit qiladi. Faqat **legacy decorators** (`experimentalDecorators: true`) bilan ishlaydi.

3 ta metadata key:

| Key | Nima saqlaydi |
|-----|---------------|
| `"design:type"` | Property/method type |
| `"design:paramtypes"` | Constructor/method parametr type lari (array) |
| `"design:returntype"` | Method return type |

```typescript
import "reflect-metadata";

function injectable(constructor: Function) {}

@injectable
class UserService {
  constructor(private repo: UserRepository, private logger: Logger) {}
}

Reflect.getMetadata("design:paramtypes", UserService);
// [UserRepository, Logger] — constructor parameter type lari
```

**Cheklovlar:** interface → `Object` (JS da yo'q), union → `Object`, generic → base type (`Promise<User>` → `Promise`), array element type yo'qoladi (`string[]` → `Array`).

</details>

### 3. Dependency Injection (DI) nima? IoC nima? Nima uchun kerak?

<details>
<summary>Javob</summary>

**DI** — class dependency larini **o'zi yaratmaydi**, **tashqaridan oladi**. **IoC** — control oqimi teskari: framework class ni yaratadi va dependency larni beradi.

```typescript
// ❌ DI siz — tight coupling
class UserService {
  private repo = new PostgresRepo("postgres://localhost/db");
  private logger = new ConsoleLogger();
}

// ✅ DI bilan — loose coupling
class UserService {
  constructor(
    private repo: IUserRepository,   // Interface
    private logger: ILogger           // Tashqaridan
  ) {}
}

// Production:
new UserService(new PostgresRepo("prod-url"), new WinstonLogger());
// Test:
new UserService(new InMemoryRepo(), new SilentLogger());
```

**Afzalliklari:**
1. **Loose coupling** — interface ga bog'liq, concrete class ga emas
2. **Testability** — mock inject oson
3. **Single Responsibility** — dependency yaratmaydi, faqat o'z ishi
4. **Configurability** — turli environment larda turli implementation

SOLID **D** (Dependency Inversion) — high-level module abstraction ga bog'liq.

</details>

### 4. Constructor, Property, Method Injection farqlari?

<details>
<summary>Javob</summary>

```typescript
// 1. Constructor Injection — eng xavfsiz
class UserService {
  constructor(
    private readonly repo: IUserRepository,  // Majburiy
    private readonly logger: ILogger          // Majburiy
  ) {}
  // Class HECH QACHON incomplete state da bo'lmaydi
}

// 2. Property Injection — optional dependency lar
class NotificationService {
  cache?: ICacheService;    // Bo'lmasa ham ishlaydi
  notify(userId: number) {
    if (this.cache) { /* ... */ }
  }
}

// 3. Method Injection — per-call dependency
class ReportGenerator {
  generate(data: any[], formatter: IFormatter): string {
    return formatter.format(data);
  }
}
generator.generate(data, new CsvFormatter());
generator.generate(data, new JsonFormatter());
```

| Pattern | Qachon | Tradeoff |
|---------|--------|----------|
| Constructor | Majburiy dep lar | Barcha dep yaratishda kerak |
| Property | Optional dep lar | Incomplete state xavfi |
| Method | Per-call dep lar | Har safar dep berish kerak |

Constructor injection **eng xavfsiz** — class yaratilganda barcha dependency lar tayyor.

</details>

### 5. Singleton va Transient scope farqi?

<details>
<summary>Javob</summary>

DI Container da **scope** — dependency lifecycle:

```typescript
// Singleton — bitta instance, doim bir xil
container.register("Logger", { useClass: ConsoleLogger, scope: "singleton" });
const a = container.resolve("Logger");
const b = container.resolve("Logger");
console.log(a === b); // true

// Transient — har safar yangi instance
container.register("Logger", { useClass: ConsoleLogger, scope: "transient" });
const c = container.resolve("Logger");
const d = container.resolve("Logger");
console.log(c === d); // false
```

| Scope | Hayot davri | Ishlatish |
|-------|------------|-----------|
| **Singleton** | App lifetime | DB connection, config, cache |
| **Transient** | Har resolve da yangi | Stateless service, lightweight |

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `design:type` metadata — output savol (Daraja: Middle)

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
  @track callback: Function = () => {};
}
```

<details>
<summary>Yechim</summary>

```
name: String
age: Number
active: Boolean
tags: Array          ← element type yo'qoldi!
data: Object
callback: Function
```

`emitDecoratorMetadata` primitive type larni constructor function ga aylantiradi: `string` → `String`, `number` → `Number`. Murakkab type lar soddalashtiriladi: `string[]` → `Array`. Bu cheklov sababi interface larni DI da avtomatik resolve qilib bo'lmaydi.

</details>

### 2. Interface DI muammo — token kerak (Daraja: Middle)

**Savol:** Bu kodda nima xato? Interface DI da nima uchun muammo?

```typescript
import "reflect-metadata";
function injectable(target: Function) {}

interface ILogger { log(msg: string): void; }

@injectable
class AppService {
  constructor(private logger: ILogger) {}
}

const paramTypes = Reflect.getMetadata("design:paramtypes", AppService);
console.log(paramTypes); // ?
```

<details>
<summary>Yechim</summary>

```
[Object] — ILogger emas!
```

**Muammo:** `ILogger` interface — JS da **mavjud emas** (type erasure). `emitDecoratorMetadata` uni `Object` qiladi. DI container qaysi class inject qilishni bilmaydi.

**Yechim — token:**

```typescript
const TOKENS = { Logger: Symbol("ILogger") } as const;

function inject(token: symbol) {
  return function (target: any, _key: string | undefined, paramIndex: number) {
    const tokens: Map<number, symbol> =
      Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    tokens.set(paramIndex, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}

@injectable
class AppService {
  constructor(@inject(TOKENS.Logger) private logger: ILogger) {}
}
// Container TOKENS.Logger orqali to'g'ri implementation ni topadi
```

</details>

### 3. `emitDecoratorMetadata` — auto-resolve output (Daraja: Middle+)

**Savol:** Output ni ayting:

```typescript
import "reflect-metadata";
function injectable(target: Function) {
  console.log(`Registering: ${target.name}`);
}

class Logger { log(msg: string) { console.log(msg); } }
class UserRepo { findAll() { return ["Ali"]; } }

@injectable
class UserService {
  constructor(private repo: UserRepo, private logger: Logger) {}
}

const paramTypes = Reflect.getMetadata("design:paramtypes", UserService);
console.log(paramTypes.map((t: any) => t.name));
```

<details>
<summary>Yechim</summary>

```
Registering: UserService
["UserRepo", "Logger"]
```

1. `@injectable` — class define da chaqiriladi → `"Registering: UserService"`
2. `emitDecoratorMetadata` — compiler `"design:paramtypes"` ga `[UserRepo, Logger]` yozadi
3. `paramTypes.map(t => t.name)` — constructor function nomlarini oladi

Bu DI Container ning asosi — `design:paramtypes` dan dependency type larni o'qib, avtomatik instance yaratish.

</details>

### 4. DI bilan testing — mock inject (Daraja: Middle+)

**Savol:** DI bilan `UserService` ni test qiling — real database va email kerak emas:

```typescript
interface IUserRepository {
  findById(id: number): Promise<User | null>;
  save(user: Omit<User, "id">): Promise<User>;
}
interface IEmailService {
  sendWelcome(email: string, name: string): Promise<void>;
}

class UserService {
  constructor(private repo: IUserRepository, private email: IEmailService) {}
  async createUser(name: string, email: string): Promise<User> {
    const user = await this.repo.save({ name, email });
    await this.email.sendWelcome(email, name);
    return user;
  }
}
```

<details>
<summary>Yechim</summary>

```typescript
import { describe, it, expect, vi } from "vitest";

describe("UserService", () => {
  it("should create user and send welcome email", async () => {
    const mockRepo: IUserRepository = {
      findById: async () => null,
      save: async (data) => ({ id: 1, ...data }),
    };

    const sendWelcomeSpy = vi.fn().mockResolvedValue(undefined);
    const mockEmail: IEmailService = { sendWelcome: sendWelcomeSpy };

    // DI — mock lar inject
    const service = new UserService(mockRepo, mockEmail);
    const user = await service.createUser("Ali", "ali@mail.com");

    expect(user.name).toBe("Ali");
    expect(sendWelcomeSpy).toHaveBeenCalledWith("ali@mail.com", "Ali");
  });

  it("should handle repo error", async () => {
    const mockRepo: IUserRepository = {
      findById: async () => null,
      save: async () => { throw new Error("DB down"); },
    };
    const mockEmail: IEmailService = { sendWelcome: vi.fn() };

    const service = new UserService(mockRepo, mockEmail);
    await expect(service.createUser("Ali", "a@b.com")).rejects.toThrow("DB down");
  });
});
```

DI siz real database va email kerak — test sekin va flaky. DI bilan mock inject → test millisekundlarda, deterministik.

</details>

### 5. Sodda DI Container yozing (Daraja: Senior)

**Savol:** `register` va `resolve` method li DI Container yozing. `emitDecoratorMetadata` dan dependency larni o'qib avtomatik resolve qilsin:

```typescript
// container.register("Logger", { useClass: ConsoleLogger, scope: "singleton" });
// container.register("UserService", { useClass: UserService });
// const svc = container.resolve<UserService>("UserService");
```

<details>
<summary>Yechim</summary>

```typescript
import "reflect-metadata";

type Scope = "singleton" | "transient";

interface Registration {
  useClass?: new (...args: any[]) => any;
  useValue?: any;
  useFactory?: () => any;
  scope?: Scope;
}

class Container {
  private registrations = new Map<any, Registration>();
  private singletons = new Map<any, any>();

  register(token: any, options: Registration): void {
    this.registrations.set(token, { scope: "transient", ...options });
  }

  resolve<T>(token: any): T {
    const reg = this.registrations.get(token);
    if (!reg) throw new Error(`No registration: ${String(token)}`);

    if (reg.useValue !== undefined) return reg.useValue;

    if (reg.useFactory) {
      if (reg.scope === "singleton") {
        if (!this.singletons.has(token))
          this.singletons.set(token, reg.useFactory());
        return this.singletons.get(token);
      }
      return reg.useFactory();
    }

    if (reg.useClass) {
      if (reg.scope === "singleton" && this.singletons.has(token))
        return this.singletons.get(token);

      // Constructor parametr type lar — emitDecoratorMetadata dan
      const paramTypes: any[] =
        Reflect.getMetadata("design:paramtypes", reg.useClass) || [];

      // Har dep ni recursive resolve
      const deps = paramTypes.map((type) => this.resolve(type));
      const instance = new reg.useClass(...deps);

      if (reg.scope === "singleton") this.singletons.set(token, instance);
      return instance;
    }

    throw new Error(`Invalid registration: ${String(token)}`);
  }
}
```

**Asosiy ishi:** `design:paramtypes` dan constructor dependency larni o'qib, **recursive** resolve qilish. Singleton scope — bir marta yaratib cache. Transient — har safar yangi.

**Cheklov:** Interface dependency lar `Object` bo'ladi — token system kerak (Savol 2 ga qarang).

</details>

---

## Xulosa

- Reflect Metadata — runtime metadata API, WeakMap asosida. `Symbol.metadata` kelajak
- `emitDecoratorMetadata` — faqat legacy decorators. `design:paramtypes` emit. Interface → `Object`
- DI/IoC — dependency tashqaridan olish, loose coupling, testability
- Constructor injection eng xavfsiz — class incomplete state da bo'lmaydi
- Singleton — bitta instance, transient — har safar yangi
- Interface DI da muammo — token system kerak (`Symbol`, `@inject`)
- `emitDecoratorMetadata` TC39 da **ishlamaydi** — `Symbol.metadata` muqobil

[Asosiy bo'limga qaytish →](../20-reflect-metadata-di.md)
