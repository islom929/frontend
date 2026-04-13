# Interview: Advanced OOP Patterns

> Mixins, composition vs inheritance, class expressions, abstract factory va advanced OOP patterns bo'yicha interview savollari. Access modifiers, this type, satisfies — oldingi bo'limlarda (06, 10).

---

## Nazariy savollar

### 1. Mixin nima? TypeScript da mixin pattern qanday ishlaydi?

<details>
<summary>Javob</summary>

Mixin — class ga **mustaqil behavior** qo'shadigan funksiya. JS da class faqat bitta class dan `extends` qila oladi (single inheritance). Mixin shu cheklovni chetlab o'tadi:

```typescript
type GConstructor<T = {}> = new (...args: any[]) => T;

function Timestamped<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();
    touch(): void { this.updatedAt = new Date(); }
  };
}

function SoftDeletable<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    deletedAt: Date | null = null;
    softDelete(): void { this.deletedAt = new Date(); }
    restore(): void { this.deletedAt = null; }
  };
}

class BaseEntity {
  constructor(public id: number) {}
}

const FullEntity = SoftDeletable(Timestamped(BaseEntity));

const entity = new FullEntity(1);
entity.id;           // number — BaseEntity dan
entity.createdAt;    // Date — Timestamped dan
entity.softDelete(); // void — SoftDeletable dan
```

Prototype chain: `instance → SoftDeletable → Timestamped → BaseEntity → Object`.

**Constrained mixin** — base class dan ma'lum shape talab qilish:

```typescript
function Printable<TBase extends GConstructor<{ name: string }>>(Base: TBase) {
  return class extends Base {
    print(): void { console.log(this.name); }
  };
}

const PrintableUser = Printable(User);     // ✅ name bor
// const PrintableProduct = Printable(Product); // ❌ name yo'q
```

</details>

### 2. Composition vs Inheritance — qachon qaysi birini ishlatish kerak?

<details>
<summary>Javob</summary>

**Inheritance** — "X is a Y". Subclass parent ning barcha behavior ini meros oladi.
**Composition** — "X has a Y". Object boshqa object lardan behavior ni delegatsiya orqali oladi.

```typescript
// Inheritance
class Animal { move(): void { console.log("Moving"); } }
class Dog extends Animal { bark(): void { console.log("Woof"); } }

// Composition
class Car {
  constructor(
    private engine: Engine,  // Car HAS an Engine
    private logger: Logger   // Car HAS a Logger
  ) {}
  start(): void {
    this.engine.start();
    this.logger.log("Car started");
  }
}
```

| Holat | Inheritance | Composition |
|-------|:-----------:|:-----------:|
| "is a" munosabat | ✅ | |
| "has a" munosabat | | ✅ |
| Ko'p behavior kerak | ❌ Single | ✅ Multiple |
| Runtime da o'zgartirish | ❌ | ✅ |
| Testing | Qiyin | Oson (DI) |

```typescript
// ❌ God class — barcha base da
class BaseService {
  log(msg: string): void { }
  cache(key: string): void { }
  validate(data: unknown): boolean { return true; }
}
class UserService extends BaseService { } // Barcha meros — kerak bo'lmasa ham

// ✅ Composition — faqat kerakli dependency lar
class UserService {
  constructor(
    private logger: Logger,
    private cache: CacheService
  ) {}
}
```

**Qoida:** "Favor composition over inheritance" (GOF). Inheritance faqat haqiqiy "is a" va ko'p shared implementation bo'lganda.

</details>

### 3. Class expression nima? Mixin larda nima uchun ishlatiladi?

<details>
<summary>Javob</summary>

Class expression — class ni **qiymat** sifatida yaratish:

```typescript
// Class declaration
class User { name = "Ali"; }

// Class expression — anonymous
const User2 = class { name = "Ali"; };

// Named — nom faqat class ichida
const User3 = class InternalName {
  whoAmI(): string { return InternalName.name; }
};
// InternalName; // ❌ Tashqarida accessible emas
```

**Mixin larda kerak** — mixin funksiya class expression **qaytaradi**:

```typescript
function Printable<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base { // ← class expression
    print(): void { console.log(String(this)); }
  };
}
```

Bu yerda `class extends Base` runtime da yaratiladi va base class dan extends qiladi. Class declaration bilan buni qilish mumkin emas.

**Factory pattern:**

```typescript
function createValidator<T>(check: (v: T) => boolean, msg: string) {
  return class {
    errorMessage = msg;
    validate(value: T): boolean { return check(value); }
  };
}
const NonEmpty = createValidator<string>(s => s.length > 0, "Bo'sh bo'lmasin");
new NonEmpty().validate("test"); // true
```

</details>

### 4. Nima uchun class faqat bitta class dan extends qila oladi, lekin bir nechta interface implement qila oladi?

<details>
<summary>Javob</summary>

JavaScript ning fundamental cheklovi — faqat **single prototype chain** bor:

```
instance.__proto__ → Child.prototype → Parent.prototype → Object.prototype
```

Ikki parent bo'lishi mumkin emas — prototype chain bitta zanjir.

**Lekin interface lar** — faqat type-level contract. Compile-time da tekshiriladi, JS da **butunlay o'chiriladi**:

```typescript
interface Printable { print(): string; }
interface Loggable { log(msg: string): void; }
interface Cacheable { cache(): void; }

class Service implements Printable, Loggable, Cacheable {
  print(): string { return "Service"; }
  log(msg: string): void { console.log(msg); }
  cache(): void { }
}

// Compiled JS — implements butunlay yo'q:
// class Service { print() { } log(msg) { } cache() { } }
```

Runtime da ko'p behavior kerak bo'lsa — **mixin pattern** ishlatiladi.

</details>

### 5. Abstract Factory pattern — TypeScript da qanday implement qilinadi?

<details>
<summary>Javob</summary>

Abstract Factory — tegishli object lar oilasini yaratish uchun interface beradi:

```typescript
interface Button { render(): string; }
interface Input { render(): string; getValue(): string; }

// Abstract Factory
interface UIFactory {
  createButton(label: string): Button;
  createInput(placeholder: string): Input;
}

// Concrete Factory — Material
class MaterialFactory implements UIFactory {
  createButton(label: string): Button {
    return { render: () => `<button class="md-btn">${label}</button>` };
  }
  createInput(placeholder: string): Input {
    return {
      render: () => `<input class="md-input" placeholder="${placeholder}">`,
      getValue: () => "",
    };
  }
}

// Client code — factory ga bog'liq emas
function buildForm(factory: UIFactory): string {
  const input = factory.createInput("Ismingiz");
  const btn = factory.createButton("Yuborish");
  return `${input.render()} ${btn.render()}`;
}

const factory: UIFactory = isMobile() ? new IOSFactory() : new MaterialFactory();
buildForm(factory); // Platform ga mos UI
```

TS bu pattern uchun ideal — interface lar compile-time type safety, runtime da overhead yo'q.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Mixin output — prototype chain (Daraja: Middle+)

**Savol:** Output ni ayting:

```typescript
function Loggable<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    log(msg: string): void { console.log(`[${this.constructor.name}] ${msg}`); }
  };
}

function Serializable<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    serialize(): string { return JSON.stringify(this); }
  };
}

class User { constructor(public name: string, public age: number) {} }

const SmartUser = Serializable(Loggable(User));
const user = new SmartUser("Ali", 25);

console.log(user.name);
console.log(user.serialize());
console.log(user instanceof User);
```

<details>
<summary>Yechim</summary>

```
Ali
{"name":"Ali","age":25}
true
```

- `user.name` → `"Ali"` — User constructor ishlaydi
- `user.serialize()` → `{"name":"Ali","age":25}` — `JSON.stringify(this)` enumerable own property larni oladi
- `user instanceof User` → `true` — prototype chain: instance → Serializable → Loggable → User

**Muhim nuance:** `this.constructor.name` mixin class larda kutilmagan natija berishi mumkin — anonymous class lar uchun engine ga bog'liq (V8 da ko'pincha bo'sh string).

</details>

### 2. Mixin field initialization order — tricky (Daraja: Senior)

**Savol:** Output ni ayting:

```typescript
class Base {
  constructor() {
    console.log("Base constructor");
    this.setup();
  }
  setup(): void { console.log("Base setup"); }
}

function Timestamped<T extends new (...args: any[]) => any>(Cls: T) {
  return class extends Cls {
    createdAt = new Date();
    setup(): void {
      console.log("Timestamped setup, createdAt:", typeof this.createdAt);
      super.setup();
    }
  };
}

function Loggable<T extends new (...args: any[]) => any>(Cls: T) {
  return class extends Cls {
    logs: string[] = [];
    setup(): void {
      console.log("Loggable setup, logs:", typeof this.logs);
      super.setup();
    }
  };
}

const Enhanced = Loggable(Timestamped(Base));
new Enhanced();
```

<details>
<summary>Yechim</summary>

```
Base constructor
Loggable setup, logs: undefined
Timestamped setup, createdAt: undefined
Base setup
```

Execution order:

1. `new Enhanced()` → constructor chaining → Base constructor ga yetadi
2. Base constructor da `this.setup()` — **virtual dispatch**: eng yuqoridagi override (Loggable.setup) chaqiriladi
3. Loggable.setup: `this.logs` → **undefined** — field initializer hali ishlamagan!
4. `super.setup()` → Timestamped.setup: `this.createdAt` → **undefined**
5. `super.setup()` → Base.setup: `"Base setup"`
6. Base constructor tugadi → Timestamped field init → Loggable field init

**Dars:** Constructor da virtual method chaqirish **xavfli** — subclass/mixin field lari hali tayyor emas. ES2022 field init `super()` qaytgandan **keyin** ishlaydi.

</details>

### 3. Mixin deserialize — xatoni toping (Daraja: Middle+)

**Savol:** Bu kodda runtime xato bor. Toping va tuzating:

```typescript
function Serializable<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    serialize(): string { return JSON.stringify(this); }
    static deserialize(json: string): InstanceType<T> {
      const data = JSON.parse(json);
      return Object.assign(new Base(), data);
    }
  };
}

class User {
  constructor(public name: string, public age: number) {}
  greet(): string { return `Hi, ${this.name}`; }
}

const SmartUser = Serializable(User);
const json = new SmartUser("Ali", 25).serialize();
const restored = SmartUser.deserialize(json);
console.log(restored.greet());
```

<details>
<summary>Yechim</summary>

**Xato:** `new Base()` — User constructor `name` va `age` **argument talab qiladi**. Argument siz chaqirilganda `name` va `age` `undefined` bo'ladi. `Object.assign` data qo'shadi, lekin constructor side effect lari (agar bo'lsa) ishlamaydi.

```typescript
// ✅ Tuzatish: Object.create bilan — constructor chaqirmasdan
function Serializable<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    serialize(): string { return JSON.stringify(this); }
    static deserialize(json: string): InstanceType<T> {
      const data = JSON.parse(json);
      const instance = Object.create(Base.prototype);
      return Object.assign(instance, data);
    }
  };
}
```

`Object.create(Base.prototype)` — constructor chaqirmasdan to'g'ri prototype chain ga ega object yaratadi. Production da **class-transformer** yoki **zod** kabi library lar ishlatiladi.

</details>

### 4. Constrained mixin yozing (Daraja: Middle+)

**Savol:** `Validatable` mixin yozing — faqat `validate(): boolean` method ga ega class larga qo'llansin. `isValid` property va `assertValid()` method qo'shsin:

```typescript
// Implement qiling:
// const ValidUser = Validatable(User);
// const user = new ValidUser("Ali", 25);
// user.isValid → true/false (validate() natijasi)
// user.assertValid() → throw agar valid emas

class User {
  constructor(public name: string, public age: number) {}
  validate(): boolean { return this.name.length > 0 && this.age > 0; }
}

class Product {
  constructor(public title: string) {}
  // validate() YO'Q
}

// Validatable(User) → ✅
// Validatable(Product) → ❌ compile error
```

<details>
<summary>Yechim</summary>

```typescript
type GConstructor<T = {}> = new (...args: any[]) => T;
type HasValidate = GConstructor<{ validate(): boolean }>;

function Validatable<TBase extends HasValidate>(Base: TBase) {
  return class extends Base {
    get isValid(): boolean {
      return this.validate();
    }

    assertValid(): void {
      if (!this.validate()) {
        throw new Error(`Validation failed for ${this.constructor.name}`);
      }
    }
  };
}

class User {
  constructor(public name: string, public age: number) {}
  validate(): boolean { return this.name.length > 0 && this.age > 0; }
}

class Product {
  constructor(public title: string) {}
}

const ValidUser = Validatable(User);     // ✅ validate() bor
// const ValidProduct = Validatable(Product); // ❌ validate() yo'q

const user = new ValidUser("Ali", 25);
console.log(user.isValid);  // true
user.assertValid();          // OK

const bad = new ValidUser("", -1);
console.log(bad.isValid);   // false
bad.assertValid();           // ❌ Error: Validation failed
```

**Tushuntirish:**

- `HasValidate` = `new (...) => { validate(): boolean }` — constraint
- `TBase extends HasValidate` — faqat `validate()` bor class lar qabul qilinadi
- `get isValid()` — computed property, `validate()` natijasini qaytaradi
- Mixin base class ning method ini ishlatadi — bu constrained mixin ning kuchi

</details>

### 5. Composition pattern — DI bilan (Daraja: Senior)

**Savol:** Inheritance o'rniga composition ishlatib, `NotificationService` yozing. Logger, Validator, Sender — alohida dependency lar sifatida inject bo'lsin:

```typescript
// Implement qiling:
// - Logger interface: log(msg: string): void
// - Validator interface: validate(to: string, body: string): boolean
// - Sender interface: send(to: string, body: string): Promise<boolean>
// - NotificationService: constructor da DI, notify() method

// notify("ali@test.com", "Hello") →
//   1. validate
//   2. send
//   3. log
//   Agar validate false → log error, send qilmaslik
```

<details>
<summary>Yechim</summary>

```typescript
interface Logger {
  log(msg: string): void;
}

interface Validator {
  validate(to: string, body: string): boolean;
}

interface Sender {
  send(to: string, body: string): Promise<boolean>;
}

class NotificationService {
  constructor(
    private logger: Logger,
    private validator: Validator,
    private sender: Sender
  ) {}

  async notify(to: string, body: string): Promise<boolean> {
    if (!this.validator.validate(to, body)) {
      this.logger.log(`Validation failed for ${to}`);
      return false;
    }

    const sent = await this.sender.send(to, body);
    this.logger.log(sent ? `Sent to ${to}` : `Failed to send to ${to}`);
    return sent;
  }
}

// Concrete implementations
const consoleLogger: Logger = { log: (msg) => console.log(`[LOG] ${msg}`) };
const emailValidator: Validator = {
  validate: (to, body) => to.includes("@") && body.length > 0,
};
const emailSender: Sender = {
  send: async (to, body) => { /* API call */ return true; },
};

// DI — composition
const service = new NotificationService(
  consoleLogger,
  emailValidator,
  emailSender
);

await service.notify("ali@test.com", "Hello"); // ✅ validate → send → log

// Test da — mock inject
const mockSender: Sender = { send: async () => true };
const testService = new NotificationService(consoleLogger, emailValidator, mockSender);
```

**Tushuntirish:**

- **Composition** — `NotificationService` hech narsani extend qilmaydi, dependency lar inject
- **Interface** lar — contract belgilaydi, concrete class emas
- **DI** — test da mock larni osongina inject qilish mumkin
- Inheritance da `BaseNotificationService` bo'lsa — mock qilish qiyin, tight coupling

</details>

---

## Xulosa

- Mixin — single inheritance cheklovini chetlab o'tish, mustaqil behavior qo'shish
- Constrained mixin — base class dan ma'lum shape talab qilish (`GConstructor<{...}>`)
- Composition > Inheritance — "has a" munosabat, DI, testing osonligi
- Class expression — mixin funksiya ichida anonymous class yaratish va qaytarish
- Multiple interface — compile-time contract, JS da o'chiriladi
- Mixin + constructor field init — xavfli, field lar `super()` qaytgandan keyin init bo'ladi
- `this` type, `#` vs `private`, `satisfies` — batafsil [interview/10](10-classes.md), [interview/06](06-type-narrowing.md)
- Type-safe EventEmitter — batafsil [interview/21](21-design-patterns.md)

[Asosiy bo'limga qaytish →](../11-advanced-oop.md)
