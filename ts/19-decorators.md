# Bo'lim 19: Decorators

> Decorator — TypeScript va JavaScript da meta-programming mexanizmi bo'lib, class, method, property, yoki accessor ga qo'shimcha behavior qo'shish imkonini beradi. Decorator lar **runtime da ishlaydigan funksiyalar** — ular compile-time da o'chirilmaydi, balki JS ga compile bo'lganda haqiqiy kod sifatida qoladi. TypeScript da ikki xil decorator standarti mavjud: legacy (`experimentalDecorators`) va TC39 Stage 3 (TS 5.0+). Bu bo'limda har ikkala standart chuqur o'rganiladi — syntax, mexanizm, compiled output, va real-world patterns.

---

## Mundarija

- [Decorators Nima](#decorators-nima)
- [Ikki Standart — Legacy vs TC39](#ikki-standart--legacy-vs-tc39)
- [Legacy Class Decorator](#legacy-class-decorator)
- [Legacy Method Decorator](#legacy-method-decorator)
- [Legacy Property va Parameter Decorators](#legacy-property-va-parameter-decorators)
- [Legacy Accessor Decorator](#legacy-accessor-decorator)
- [TC39 Decorators — Yangi Standart](#tc39-decorators--yangi-standart)
- [TC39 Class Decorator](#tc39-class-decorator)
- [TC39 Method Decorator](#tc39-method-decorator)
- [TC39 Field va Accessor Decorators](#tc39-field-va-accessor-decorators)
- [Decorator Factories — Parametrli Decoratorlar](#decorator-factories--parametrli-decoratorlar)
- [Decorator Composition va Execution Order](#decorator-composition-va-execution-order)
- [Decorator Metadata — `Symbol.metadata`](#decorator-metadata--symbolmetadata)
- [Real-World Decorator Patterns](#real-world-decorator-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Decorators Nima

### Nazariya

Decorator — class yoki class a'zolariga **qo'shimcha behavior qo'shadigan funksiya**. Decorator `@expression` syntax bilan class declaration oldidan yoziladi. Decorator lar **meta-programming** ning bir turi — kod o'zi haqida ma'lumot oladi va o'z behavior ini o'zgartiradi.

Decorator lar quyidagi muammolarni yechadi:

1. **Cross-cutting concerns** — logging, validation, caching, authorization kabi logic ni decorator orqali tashqaridan qo'shish
2. **Separation of concerns** — business logic va infrastructure logic ni ajratish
3. **Code reuse** — bir xil behavior ni ko'p joylarda qayta ishlatish
4. **Declarative programming** — imperative emas, declarative tarzda ifodalash

Decorator lar **runtime da ishlaydigan oddiy funksiyalar**. Ular `type erasure` ga uchramaydi — JS ga compile bo'lganda qoladi va haqiqiy runtime ta'sir ko'rsatadi. Bu interface va type alias lardan tubdan farq qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Decorator — oddiy funksiya
function sealed(constructor: new (...args: any[]) => any) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class BankAccount {
  balance: number = 0;
  deposit(amount: number) { this.balance += amount; }
}

// Object.seal tufayli yangi property qo'shib bo'lmaydi
// BankAccount.prototype.newMethod = ... // ❌ Runtime da xato
```

</details>

---

## Ikki Standart — Legacy vs TC39

### Nazariya

TypeScript da decorator lar **ikki xil standart** bo'yicha ishlaydi. Bu ikki standart bir-biridan **tubdan farq qiladi** — syntax o'xshash bo'lsa-da, ichki mexanizm va API boshqacha.

| Xususiyat | Legacy | TC39 (TS 5.0+) |
|-----------|--------|----------------|
| tsconfig | `experimentalDecorators: true` | Hech narsa kerak emas |
| API | `target, key, descriptor` | `value, context` |
| Parameter decorator | ✅ Bor | ❌ Yo'q |
| `accessor` keyword | ❌ Yo'q | ✅ Bor |
| `Symbol.metadata` | ❌ (`reflect-metadata`) | ✅ Native |
| Angular/NestJS | ✅ Ishlatadi | Migrating |
| Compiled output | `__decorate` helper | `__esDecorate` + `__runInitializers` |

**Qachon qaysi biri:**
- **Legacy** — Angular, NestJS, TypeORM, mavjud loyihalar
- **TC39** — yangi loyihalar, standartga mos (kelajak)

<details>
<summary><strong>Under the Hood</strong></summary>

**Legacy compiled output:**

```typescript
// TypeScript source:
@sealed
class User { @log greet() {} }

// Compiled JS:
var User = /** @class */ (function () { /* ... */ }());
__decorate([log], User.prototype, "greet", null);
User = __decorate([sealed], User);
```

Legacy da `__decorate` helper funksiyasi har bir decorator ni target ga qo'llaydi.

**TC39 compiled output:**

```typescript
// TypeScript source (TC39):
@sealed
class User { @log greet() {} }

// Compiled JS:
let User = (() => {
  let _instanceExtraInitializers = [];
  let _greet_decorators = [log];
  return class User {
    constructor() { __runInitializers(this, _instanceExtraInitializers); }
    greet() { /* ... */ }
  };
})();
User = __esDecorate(User, null, [sealed], { kind: "class", name: "User" });
```

TC39 da `__esDecorate` va `__runInitializers` helper lar ishlatiladi — decorator `context` object oladi.

</details>

---

## Legacy Class Decorator

### Nazariya

Legacy class decorator — class constructor ni argument sifatida oladi. Agar yangi constructor qaytarsa — class constructor almashtiriladi.

**Signature:** `(constructor: new (...args: any[]) => any) => void | typeof constructor`

`experimentalDecorators: true` kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Sealed class ===
function sealed(constructor: new (...args: any[]) => any) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class User {
  name: string;
  constructor(name: string) { this.name = name; }
}

// === Registry pattern ===
const registry = new Map<string, new (...args: any[]) => any>();

function register(constructor: new (...args: any[]) => any) {
  registry.set(constructor.name, constructor);
}

@register
class UserService { /* ... */ }

@register
class OrderService { /* ... */ }

console.log(registry.get("UserService")); // [class UserService]

// === Constructor override ===
function withTimestamp<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    createdAt = new Date();
  };
}

@withTimestamp
class Post {
  title: string;
  constructor(title: string) { this.title = title; }
}

const post = new Post("Hello");
// (post as any).createdAt — Date object mavjud
```

</details>

---

## Legacy Method Decorator

### Nazariya

Legacy method decorator — method ni o'zgartirish yoki kuzatish uchun. Uchta argument oladi:

1. **`target`** — static method uchun constructor, instance method uchun prototype
2. **`propertyKey`** — method nomi (string yoki symbol)
3. **`descriptor`** — `PropertyDescriptor` (value, writable, enumerable, configurable)

Decorator `descriptor` ni o'zgartirishi yoki yangi descriptor qaytarishi mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === @log — method chaqiruvlarni log qilish ===
function log(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const original = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${propertyKey} with:`, args);
    const result = original.apply(this, args);
    console.log(`${propertyKey} returned:`, result);
    return result;
  };

  return descriptor;
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// Calling add with: [2, 3]
// add returned: 5

// === @memoize — natijani cache qilish ===
function memoize(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const original = descriptor.value;
  const cache = new Map<string, any>();

  descriptor.value = function (...args: any[]) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = original.apply(this, args);
    cache.set(key, result);
    return result;
  };

  return descriptor;
}

class ApiClient {
  @memoize
  fetchUser(id: number) {
    console.log(`Fetching user ${id}...`);
    return { id, name: "User" };
  }
}

const client = new ApiClient();
client.fetchUser(1); // "Fetching user 1..." — actual call
client.fetchUser(1); // (silence) — cache dan

// === @retry — xato bo'lganda qayta urinish ===
function retry(attempts: number) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      for (let i = 0; i < attempts; i++) {
        try {
          return await original.apply(this, args);
        } catch (e) {
          if (i === attempts - 1) throw e;
          console.log(`Retry ${i + 1}/${attempts} for ${propertyKey}`);
        }
      }
    };

    return descriptor;
  };
}

class HttpClient {
  @retry(3)
  async fetch(url: string) {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  }
}
```

</details>

---

## Legacy Property va Parameter Decorators

### Nazariya

**Property decorator** — ikki argument oladi: `target` (prototype) va `propertyKey` (property nomi). `PropertyDescriptor` **olmaydi** — chunki class property lar instance da yaratiladi, prototype da emas. Property decorator faqat **metadata saqlash** uchun ishlatiladi.

**Parameter decorator** — uchta argument oladi: `target`, `methodName`, `parameterIndex` (0-based). Bu ham faqat metadata saqlash uchun. **TC39 da parameter decorator mavjud emas** — bu legacy ga xos.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Property decorator — @required ===
const requiredProps = new Map<new (...args: any[]) => any, string[]>();

function required(target: any, propertyKey: string) {
  const constructor = target.constructor;
  const existing = requiredProps.get(constructor) || [];
  requiredProps.set(constructor, [...existing, propertyKey]);
}

class UserForm {
  @required name: string = "";
  @required email: string = "";
  bio: string = "";
}

// Validation function
function validate(obj: any): boolean {
  const props = requiredProps.get(obj.constructor) || [];
  return props.every(prop => obj[prop] !== "" && obj[prop] !== undefined);
}

// === Parameter decorator — @inject ===
const injectMetadata = new Map<new (...args: any[]) => any, Map<string, number[]>>();

function inject(target: any, methodName: string, paramIndex: number) {
  const constructor = target.constructor;
  const existing = injectMetadata.get(constructor) || new Map();
  const indices = existing.get(methodName) || [];
  existing.set(methodName, [...indices, paramIndex]);
  injectMetadata.set(constructor, existing);
}

class UserService {
  getUser(@inject id: number, @inject role: string) {
    return { id, role };
  }
}
// DI framework lar (NestJS) shu metadata ni o'qib parameter inject qiladi
```

</details>

---

## Legacy Accessor Decorator

### Nazariya

Accessor decorator — getter/setter uchun. Method decorator bilan **bir xil signature** — farqi shundaki, descriptor da `value` o'rniga `get` va `set` funksiyalari bo'ladi.

**Muhim:** Bitta property uchun getter va setter bitta descriptor ni share qiladi — faqat birinchi e'lon qilingan accessor ga decorator qo'yish mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
function configurable(value: boolean) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    descriptor.configurable = value;
  };
}

class Point {
  private _x: number;
  private _y: number;

  constructor(x: number, y: number) { this._x = x; this._y = y; }

  @configurable(false)
  get x() { return this._x; }
  set x(value: number) { this._x = value; }

  @configurable(false)
  get y() { return this._y; }
  set y(value: number) { this._y = value; }
}
```

</details>

---

## TC39 Decorators — Yangi Standart

### Nazariya

TC39 Stage 3 decorators — ECMAScript standart decorator proposal. TS 5.0 dan boshlab qo'llab-quvvatlanadi. **`experimentalDecorators` kerak emas** — default holatda ishlaydi.

Legacy dan asosiy farqlar:

1. **`context` API** — barcha decorator turlari yagona `context` object oladi
2. **`addInitializer`** — class yoki instance yaratilganda chaqiriladigan callback
3. **`Symbol.metadata`** — native metadata mexanizm (reflect-metadata o'rniga)
4. **`accessor` keyword** — auto-accessor decorators
5. **Parameter decorator YO'Q** — TC39 da standartlashtirilmagan

**Context object:**

```typescript
interface ClassMethodDecoratorContext {
  kind: "method";
  name: string | symbol;
  static: boolean;
  private: boolean;
  access: { get(obj: any): any };
  addInitializer(initializer: () => void): void;
  metadata: Record<string | symbol, any>;
}
```

Har bir decorator turi uchun o'z context interface bor — `kind` property farqli.

---

## TC39 Class Decorator

### Nazariya

TC39 class decorator — class constructor ni va context ni oladi. Yangi class qaytarsa — constructor almashtiriladi.

**Signature:** `(value: new (...args: any[]) => any, context: ClassDecoratorContext) => new (...args: any[]) => any | void`

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === @sealed (TC39) ===
function sealed(
  value: new (...args: any[]) => any,
  context: ClassDecoratorContext
) {
  Object.seal(value);
  Object.seal(value.prototype);
}

@sealed
class User {
  name: string;
  constructor(name: string) { this.name = name; }
}

// === @register (TC39) ===
const globalRegistry = new Map<string, new (...args: any[]) => any>();

function register(
  value: new (...args: any[]) => any,
  context: ClassDecoratorContext
) {
  globalRegistry.set(String(context.name), value);
}

@register
class OrderService { /* ... */ }

// === @withTimestamp (TC39) ===
function withTimestamp<T extends new (...args: any[]) => any>(
  value: T,
  context: ClassDecoratorContext
) {
  return class extends value {
    createdAt = new Date();
  } as T;
}

@withTimestamp
class Post {
  title: string;
  constructor(title: string) { this.title = title; }
}
```

</details>

---

## TC39 Method Decorator

### Nazariya

TC39 method decorator — method funksiyasini va context ni oladi. Legacy dan farqi — `PropertyDescriptor` emas, **method funksiyasining o'zi** birinchi argument.

**Signature:** `(value: (...args: any[]) => any, context: ClassMethodDecoratorContext) => ((...args: any[]) => any) | void`

Yangi funksiya qaytarsa — method almashtiriladi. `void` qaytarsa — original qoladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === @log (TC39) ===
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  const methodName = String(context.name);

  return function (this: any, ...args: any[]) {
    console.log(`→ ${methodName}(${args.join(", ")})`);
    const result = originalMethod.call(this, args);
    console.log(`← ${methodName} =`, result);
    return result;
  };
}

class Calculator {
  @log
  add(a: number, b: number): number { return a + b; }
}

// === @memoize (TC39) ===
function memoize(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  const cache = new Map<string, any>();

  return function (this: any, ...args: any[]) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = originalMethod.call(this, ...args);
    cache.set(key, result);
    return result;
  };
}

// === @authorize (TC39) ===
function authorize(role: string) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return function (this: any, ...args: any[]) {
      if ((this as any).currentUserRole !== role) {
        throw new Error(`Unauthorized: requires ${role}`);
      }
      return originalMethod.call(this, ...args);
    };
  };
}

class AdminPanel {
  currentUserRole = "user";

  @authorize("admin")
  deleteUser(id: number) { /* ... */ }
}
```

</details>

---

## TC39 Field va Accessor Decorators

### Nazariya

**Field decorator** — class property ni decorate qiladi. Property **value sini** oladi (yoki `undefined` agar initializer yo'q bo'lsa). Return value — initializer funksiya bo'lib, property ning boshlang'ich qiymatini o'zgartiradi.

**Accessor decorator** — `accessor` keyword bilan declare qilingan auto-accessor lar uchun. `accessor` keyword — TS 4.9+ da qo'shilgan — avtomatik getter/setter pair yaratadi. Decorator `get` va `set` funksiyalarni o'z ichiga olgan object oladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

**Field decorator:**

```typescript
function uppercase(
  value: undefined,
  context: ClassFieldDecoratorContext
) {
  return function (initialValue: string) {
    return initialValue.toUpperCase();
  };
}

class Config {
  @uppercase
  hostname: string = "localhost";
}

const cfg = new Config();
console.log(cfg.hostname); // "LOCALHOST"

// === @range — value ni cheklash ===
function range(min: number, max: number) {
  return function (
    value: undefined,
    context: ClassFieldDecoratorContext
  ) {
    return function (initialValue: number) {
      return Math.min(max, Math.max(min, initialValue));
    };
  };
}

class Player {
  @range(0, 100)
  health: number = 150;
}

const player = new Player();
console.log(player.health); // 100 (max ga cheklandi)
```

**Accessor decorator:**

```typescript
// accessor keyword — auto getter/setter yaratadi
class User {
  accessor name: string = "Ali";
  // Bu aslida:
  // #name = "Ali";
  // get name() { return this.#name; }
  // set name(v) { this.#name = v; }
}

// Accessor decorator — get/set ni o'zgartirish
function logged(
  value: { get: () => any; set: (v: any) => void },
  context: ClassAccessorDecoratorContext
) {
  return {
    get(this: any) {
      console.log(`Get ${String(context.name)}`);
      return value.get.call(this);
    },
    set(this: any, newValue: any) {
      console.log(`Set ${String(context.name)} = ${newValue}`);
      value.set.call(this, newValue);
    },
  };
}

class Settings {
  @logged
  accessor theme: string = "light";
}

const s = new Settings();
s.theme;           // "Get theme"
s.theme = "dark";  // "Set theme = dark"
```

</details>

---

## Decorator Factories — Parametrli Decoratorlar

### Nazariya

Decorator factory — **parametr qabul qilib decorator qaytaradigan funksiya**. Bu `@decorator` o'rniga `@decorator(args)` syntax bilan ishlatiladi. Factory pattern decorator ni konfiguratsiya qilish imkonini beradi.

```typescript
// Oddiy decorator — parametrsiz
function log(target: any, ctx: ClassMethodDecoratorContext) { /* ... */ }

// Decorator factory — parametrli
function log(level: string) {
  return function (target: any, ctx: ClassMethodDecoratorContext) { /* ... */ };
}
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === @retry(attempts) ===
function retry(attempts: number) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return async function (this: any, ...args: any[]) {
      for (let i = 0; i < attempts; i++) {
        try {
          return await originalMethod.call(this, ...args);
        } catch (e) {
          if (i === attempts - 1) throw e;
          console.log(`Retry ${i + 1}/${attempts}`);
        }
      }
    };
  };
}

class ApiClient {
  @retry(3)
  async fetchData(url: string) { /* ... */ }
}

// === @debounce(ms) ===
function debounce(ms: number) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    let timer: ReturnType<typeof setTimeout>;
    return function (this: any, ...args: any[]) {
      clearTimeout(timer);
      timer = setTimeout(() => originalMethod.call(this, ...args), ms);
    };
  };
}

class SearchInput {
  @debounce(300)
  onInput(value: string) { console.log("Searching:", value); }
}
```

</details>

---

## Decorator Composition va Execution Order

### Nazariya

Bir nechta decorator bitta target ga qo'yilganda — ikki bosqichli jarayon:

1. **Evaluate** (yuqoridan pastga) — decorator factory lar chaqiriladi
2. **Apply** (pastdan yuqoriga) — decorator lar target ga qo'llanadi

Bu matematikdagi **funksiya kompozitsiyasi** — `@a @b @c method` → `a(b(c(method)))`.

**Class ichidagi umumiy tartib:**
1. Instance method/property/accessor decorator lar (tartibda)
2. Static method/property/accessor decorator lar (tartibda)
3. Constructor parameter decorator lar (legacy faqat)
4. Class decorator lar

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
function a() {
  console.log("a factory");
  return function (v: any, ctx: ClassMethodDecoratorContext) {
    console.log("a decorator");
  };
}

function b() {
  console.log("b factory");
  return function (v: any, ctx: ClassMethodDecoratorContext) {
    console.log("b decorator");
  };
}

function c(v: any, ctx: ClassDecoratorContext) {
  console.log("c decorator");
}

@c
class Test {
  @a()
  @b()
  method() {}
}

// Output:
// a factory       ← factory lar yuqoridan pastga
// b factory
// b decorator     ← decorator lar pastdan yuqoriga
// a decorator
// c decorator     ← class decorator eng oxirida
```

</details>

---

## Decorator Metadata — `Symbol.metadata`

### Nazariya

TC39 decorators `Symbol.metadata` orqali **native metadata** mexanizm beradi. Bu legacy dagi `reflect-metadata` polyfill ning standart versiyasi. Har bir decorator `context.metadata` object ga ma'lumot yozishi mumkin — bu metadata keyinchalik `Class[Symbol.metadata]` orqali o'qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
const ROLES_KEY = Symbol("roles");

function requiredRole(role: string) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    if (!context.metadata[ROLES_KEY]) {
      context.metadata[ROLES_KEY] = {};
    }
    (context.metadata[ROLES_KEY] as Record<string, string>)[String(context.name)] = role;

    return function (this: any, ...args: any[]) {
      return originalMethod.call(this, ...args);
    };
  };
}

class AdminController {
  @requiredRole("admin")
  deleteUser(id: number) { /* ... */ }

  @requiredRole("moderator")
  banUser(id: number) { /* ... */ }
}

// Metadata o'qish:
const meta = AdminController[Symbol.metadata];
console.log(meta?.[ROLES_KEY]);
// { deleteUser: "admin", banUser: "moderator" }
```

</details>

---

## Real-World Decorator Patterns

### Nazariya

Decorator lar real-world da quyidagi pattern larda keng ishlatiladi. Barchasi **cross-cutting concerns** — business logic dan ajratilgan infrastructure logic.

<details>
<summary><strong>Kod Misollari</strong></summary>

**@sealed — class ni extend qilish mumkin emas:**

```typescript
function sealed(value: new (...args: any[]) => any, context: ClassDecoratorContext) {
  Object.seal(value);
  Object.seal(value.prototype);
}
```

**@singleton — faqat bitta instance:**

```typescript
function singleton<T extends new (...args: any[]) => any>(value: T, context: ClassDecoratorContext) {
  let instance: InstanceType<T> | null = null;
  return class extends value {
    constructor(...args: any[]) {
      if (instance) return instance;
      super(...args);
      instance = this as InstanceType<T>;
    }
  } as T;
}
```

**@validate — input validation:**

```typescript
function validate(schema: Record<number, string>) {
  return function (originalMethod: (...args: any[]) => any, context: ClassMethodDecoratorContext) {
    return function (this: any, ...args: any[]) {
      for (const [index, expectedType] of Object.entries(schema)) {
        if (typeof args[Number(index)] !== expectedType) {
          throw new TypeError(`${String(context.name)}: arg[${index}] expected ${expectedType}`);
        }
      }
      return originalMethod.call(this, ...args);
    };
  };
}
```

**@deprecated — ogohlantirish chiqarish:**

```typescript
function deprecated(message: string) {
  return function (originalMethod: (...args: any[]) => any, context: ClassMethodDecoratorContext) {
    return function (this: any, ...args: any[]) {
      console.warn(`DEPRECATED: ${String(context.name)} — ${message}`);
      return originalMethod.call(this, ...args);
    };
  };
}

class OldApi {
  @deprecated("Use fetchUsers() instead")
  getUsers() { return []; }
}
```

</details>

---

## Edge Cases va Gotchas

### 1. Legacy va TC39 aralash bo'lmaydi

```json
// ❌ — ikkalasi birga ishlamaydi
{ "compilerOptions": { "experimentalDecorators": true } }
// Bu yoqilganda BARCHA decorator lar legacy mode da ishlaydi
// TC39 decorator syntax ni legacy deb interpret qiladi → kutilmagan xatolar

// ✅ — bitta tanlang: yoki legacy, yoki TC39
// TC39 uchun: experimentalDecorators NI O'CHIRING (yoki qo'ymang)
```

### 2. TC39 da parameter decorator YO'Q

```typescript
// Legacy — ✅ ishlaydi
class Service {
  method(@inject param: string) {} // ✅ experimentalDecorators bilan
}

// TC39 — ❌ syntax error
class Service {
  method(@inject param: string) {} // ❌ — TC39 da parameter decorator yo'q
}

// Yechim: NestJS va Angular bular uchun boshqa pattern ishlatmoqda (inject token, constructor injection)
```

### 3. Decorator lar faqat class a'zolariga qo'llanadi

```typescript
// ❌ — oddiy funksiya ga decorator qo'yib bo'lmaydi
@log
function standalone() {} // ❌ Syntax error

// ❌ — arrow function ga qo'yib bo'lmaydi
const fn = @log () => {}; // ❌

// ✅ — faqat class va class a'zolari
class Service {
  @log method() {} // ✅
}
```

### 4. `accessor` keyword siz field decorator getter/setter bermaydi

```typescript
// TC39 field decorator — get/set olish MUMKIN EMAS
function logged(value: undefined, context: ClassFieldDecoratorContext) {
  // value = undefined — field ning getter/setter sini BERMAYDI
  // Faqat initializer qaytarish mumkin
}

// accessor keyword KERAK — getter/setter olish uchun
class User {
  @logged accessor name = "Ali"; // ✅ — get/set decorator ga beriladi
  @logged name = "Ali";          // ⚠️ — faqat initializer o'zgaradi
}
```

### 5. Decorator return type muhim — `void` vs yangi qiymat

```typescript
// Method decorator — void qaytarsa original qoladi
function noOp(method: (...args: any[]) => any, ctx: ClassMethodDecoratorContext) {
  // void — method O'ZGARMAYDI
}

// Method decorator — yangi function qaytarsa ALMASHTIRILADI
function replace(method: (...args: any[]) => any, ctx: ClassMethodDecoratorContext) {
  return function () { return "replaced"; }; // original method YO'QOLDI
}
```

---

## Common Mistakes

### ❌ Xato 1: Legacy va TC39 ni aralashtirish

```typescript
// ❌ — tsconfig da experimentalDecorators: true + TC39 syntax
// Barcha decorator lar legacy mode da ishlaydi — TC39 context API ISHLAMAYDI

// ✅ — bitta standart tanlang
// Legacy: experimentalDecorators: true + emitDecoratorMetadata: true
// TC39: experimentalDecorators ni O'CHIRING
```

### ❌ Xato 2: Decorator dan `this` ni yo'qotish

```typescript
// ❌ — arrow function decorator da this yo'qoladi
function log(method: (...args: any[]) => any, ctx: ClassMethodDecoratorContext) {
  return (...args: any[]) => {
    // this = undefined — arrow function o'z this i yo'q!
    return method.call(this, ...args); // ❌ this noto'g'ri
  };
}

// ✅ — regular function ishlatish
function log(method: (...args: any[]) => any, ctx: ClassMethodDecoratorContext) {
  return function (this: any, ...args: any[]) {
    return method.call(this, ...args); // ✅ this to'g'ri
  };
}
```

### ❌ Xato 3: Decorator tartibini bilmaslik

```typescript
// @a @b method — b BIRINCHI apply bo'ladi (pastdan yuqoriga)
// Lekin factory lar YUQORIDAN PASTGA evaluate bo'ladi

// ❌ — tartibni teskari o'ylash
@first   // 2-chi apply bo'ladi
@second  // 1-chi apply bo'ladi
method() {}
```

### ❌ Xato 4: Property decorator da value o'zgartirishga urinish (legacy)

```typescript
// ❌ — legacy property decorator descriptor OLMAYDI
function setDefault(target: any, key: string) {
  // descriptor yo'q — property value ni bu yerda o'zgartirib bo'lmaydi
  target[key] = "default"; // ❌ — prototype ga yoziladi, instance ga emas
}

// ✅ — TC39 da initializer qaytarish
function setDefault(value: undefined, context: ClassFieldDecoratorContext) {
  return () => "default"; // ✅ — instance yaratilganda chaqiriladi
}
```

### ❌ Xato 5: `emitDecoratorMetadata` ni TC39 bilan ishlatish

```typescript
// ❌ — emitDecoratorMetadata faqat legacy bilan ishlaydi
{
  // "experimentalDecorators": false (yoki yo'q),
  "emitDecoratorMetadata": true  // ❌ — ta'siri yo'q TC39 da
}

// TC39 da metadata uchun Symbol.metadata ishlatiladi
// emitDecoratorMetadata va reflect-metadata kerak EMAS
```

---

## Amaliy Mashqlar

### Mashq 1: `@readonly` Method Decorator (Oson)

**Savol:** TC39 method decorator yozing — method ni `writable: false` qilsin (qayta assign qilish mumkin bo'lmasin).

<details>
<summary>Javob</summary>

```typescript
function readonly(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  context.addInitializer(function (this: any) {
    Object.defineProperty(this, context.name, {
      value: originalMethod,
      writable: false,
      configurable: false,
    });
  });
}

class Config {
  @readonly
  getDbUrl(): string { return "postgresql://localhost:5432/db"; }
}
```

</details>

---

### Mashq 2: `@validate` Decorator Factory (O'rta)

**Savol:** TC39 method decorator factory — `@validate(schema)` — method argumentlarni tekshirsin.

<details>
<summary>Javob</summary>

```typescript
function validate(schema: Record<number, string>) {
  return function (originalMethod: (...args: any[]) => any, context: ClassMethodDecoratorContext) {
    return function (this: any, ...args: any[]) {
      for (const [index, expectedType] of Object.entries(schema)) {
        if (typeof args[Number(index)] !== expectedType) {
          throw new TypeError(`${String(context.name)}: arg[${index}] expected ${expectedType}`);
        }
      }
      return originalMethod.call(this, ...args);
    };
  };
}

class UserService {
  @validate({ 0: "string", 1: "number" })
  createUser(name: string, age: number) { return { name, age }; }
}
```

</details>

---

### Mashq 3: `@singleton` Class Decorator (O'rta)

**Savol:** TC39 class decorator — faqat bitta instance yaratilsin.

<details>
<summary>Javob</summary>

```typescript
function singleton<T extends new (...args: any[]) => any>(value: T, context: ClassDecoratorContext) {
  let instance: InstanceType<T> | null = null;
  return class extends value {
    constructor(...args: any[]) {
      if (instance) return instance;
      super(...args);
      instance = this as InstanceType<T>;
    }
  } as T;
}

@singleton
class Database {
  constructor(public url: string) { console.log("Created"); }
}

const db1 = new Database("postgres://localhost/app"); // "Created"
const db2 = new Database("other");                     // (silence)
console.log(db1 === db2); // true
```

</details>

---

### Mashq 4: Decorator Composition Output (O'rta)

**Savol:** Quyidagi kodning console output ini ayting:

```typescript
function a() { console.log("a factory"); return (v: any, ctx: ClassMethodDecoratorContext) => { console.log("a apply"); }; }
function b() { console.log("b factory"); return (v: any, ctx: ClassMethodDecoratorContext) => { console.log("b apply"); }; }
function c(v: any, ctx: ClassDecoratorContext) { console.log("c apply"); }

@c class Test { @a() @b() method() {} }
```

<details>
<summary>Javob</summary>

```
a factory
b factory
b apply
a apply
c apply
```

Factory lar yuqoridan pastga, decorator lar pastdan yuqoriga, class decorator eng oxirida.

</details>

---

### Mashq 5: `@trace` Full System (Qiyin)

**Savol:** TC39 da `@trace` method decorator + `@traceable` class decorator yozing. `Symbol.metadata` ishlatilsin.

<details>
<summary>Javob</summary>

```typescript
const TRACED = Symbol("traced");

function trace(originalMethod: (...args: any[]) => any, context: ClassMethodDecoratorContext) {
  const name = String(context.name);
  if (!context.metadata[TRACED]) context.metadata[TRACED] = [];
  (context.metadata[TRACED] as string[]).push(name);

  return function (this: any, ...args: any[]) {
    const id = Math.random().toString(36).slice(2, 8);
    console.log(`[${id}] → ${name}(${args.join(", ")})`);
    try {
      const result = originalMethod.call(this, ...args);
      console.log(`[${id}] ← ${name} =`, result);
      return result;
    } catch (e) {
      console.log(`[${id}] ✗ ${name}:`, e);
      throw e;
    }
  };
}

function traceable(value: new (...args: any[]) => any, context: ClassDecoratorContext) {
  context.addInitializer(function () {
    const traced = context.metadata[TRACED] as string[] || [];
    console.log(`${String(context.name)} traced: [${traced.join(", ")}]`);
  });
}

@traceable
class OrderService {
  @trace createOrder(userId: number) { return { id: 1, userId }; }
  @trace cancelOrder(id: number) { if (id <= 0) throw new Error("Invalid"); return true; }
}
```

</details>

---

## Xulosa

Bu bo'limda TypeScript/JavaScript decorator system i o'rganildi:

**Ikki standart:**
- **Legacy** (`experimentalDecorators`) — Angular, NestJS, TypeORM. `target/key/descriptor` API. Parameter decorator bor.
- **TC39** (TS 5.0+) — yangi standart. `value/context` API. `Symbol.metadata`, `addInitializer`, `accessor` keyword. Parameter decorator yo'q.

**Decorator turlari:**
- **Class** — constructor ni oladi/almashtiradi
- **Method** — method funksiyasini oladi/almashtiradi
- **Field** — initializer qaytaradi (TC39), metadata saqlaydi (legacy)
- **Accessor** — getter/setter ni oladi/almashtiradi
- **Parameter** — faqat legacy, metadata saqlash uchun

**Composition:** Factory lar yuqoridan pastga evaluate, decorator lar pastdan yuqoriga apply — `a(b(c(target)))`.

**Runtime funksiyalar:** Decorator lar type erasure ga uchramaydi — JS ga compile bo'lganda qoladi.

**Real-world patterns:** `@log`, `@memoize`, `@validate`, `@authorize`, `@singleton`, `@retry`, `@deprecated`.

**Bog'liq bo'limlar:**
- [Bo'lim 20: Reflect Metadata va DI](20-reflect-metadata-di.md) — `reflect-metadata`, DI pattern
- [Bo'lim 21: Design Patterns](21-design-patterns.md) — Singleton, Observer, Strategy patterns

---

**Keyingi bo'lim:** [20-reflect-metadata-di.md](20-reflect-metadata-di.md) — Reflect Metadata API, `emitDecoratorMetadata`, Dependency Injection pattern, va DI container yaratish.
