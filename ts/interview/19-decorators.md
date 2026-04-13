# Interview: Decorators

> Decorator nima, Legacy vs TC39, decorator factories, accessor keyword, Symbol.metadata, composition order va real-world patterns bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Decorator nima? TypeScript da qanday ishlaydi?

<details>
<summary>Javob</summary>

Decorator — class yoki class a'zolariga **qo'shimcha behavior qo'shadigan funksiya**. `@expression` syntax bilan. Type annotation kabi o'chirilmaydi — **runtime da ishlaydi**:

```typescript
function log(originalMethod: Function, context: ClassMethodDecoratorContext) {
  return function (this: any, ...args: any[]) {
    console.log(`${String(context.name)} called with`, args);
    return originalMethod.call(this, ...args);
  };
}

class Calculator {
  @log
  add(a: number, b: number): number { return a + b; }
}

new Calculator().add(2, 3);
// "add called with [2, 3]" → 5
```

Decorator lar **meta-programming** — cross-cutting concern lar (logging, validation, caching) ni business logic dan ajratish. Separation of Concerns.

</details>

### 2. Legacy va TC39 decorator lar farqi?

<details>
<summary>Javob</summary>

| Xususiyat | Legacy | TC39 (TS 5.0+) |
|-----------|--------|-----------------|
| Yoqish | `experimentalDecorators: true` | Flag kerak emas |
| Method args | `target, key, descriptor` | `value, context` |
| Parameter decorator | ✅ | ❌ Yo'q |
| `accessor` keyword | ❌ | ✅ |
| Metadata | `reflect-metadata` package | `Symbol.metadata` native |
| ECMAScript standart | ❌ | ✅ Stage 3 |

```typescript
// Legacy:
function log(target: any, key: string, descriptor: PropertyDescriptor) { }

// TC39:
function log(originalMethod: Function, context: ClassMethodDecoratorContext) { }
```

**Eng muhim farq:** TC39 da **parameter decorator yo'q** — NestJS/Angular shu sabab hali legacy da.

</details>

### 3. Decorator factory nima? Oddiy decorator dan farqi?

<details>
<summary>Javob</summary>

Factory — **parametr qabul qilib, decorator qaytaradigan** funksiya. Oddiy: `@log`, factory: `@log("debug")`.

```typescript
// Oddiy decorator — parametrsiz
function log(originalMethod: Function, context: ClassMethodDecoratorContext) {
  return function (this: any, ...args: any[]) {
    console.log(`${String(context.name)} called`);
    return originalMethod.call(this, ...args);
  };
}

// Factory — parametr oladi
function logWithLevel(level: "info" | "warn" | "error") {
  return function (originalMethod: Function, context: ClassMethodDecoratorContext) {
    return function (this: any, ...args: any[]) {
      console[level](`[${level}] ${String(context.name)} called`);
      return originalMethod.call(this, ...args);
    };
  };
}

class Service {
  @log                     // oddiy
  method1() {}
  @logWithLevel("warn")    // factory
  method2() {}
}
```

Factory **closure** orqali ishlaydi — tashqi funksiya parametrlarini ichki decorator capture qiladi.

</details>

### 4. TC39 da `accessor` keyword nima?

<details>
<summary>Javob</summary>

`accessor` — avtomatik **private storage + getter + setter** yaratadi:

```typescript
class User {
  accessor name: string = "Guest";
  // → #name = "Guest"; get name() { return this.#name; } set name(v) { this.#name = v; }
}
```

Decorator bilan eng kuchli — getter **va** setter ni bir vaqtda intercept:

```typescript
function tracked(
  value: { get: () => any; set: (v: any) => void },
  context: ClassAccessorDecoratorContext
) {
  return {
    get() {
      console.log(`Reading ${String(context.name)}`);
      return value.get.call(this);
    },
    set(newValue: any) {
      console.log(`Setting ${String(context.name)} to`, newValue);
      value.set.call(this, newValue);
    },
    init(initialValue: any) {
      console.log(`Init ${String(context.name)} =`, initialValue);
      return initialValue;
    },
  };
}

class AppState {
  @tracked accessor count: number = 0;
}
// Init count = 0
// state.count = 5 → "Setting count to 5"
// state.count → "Reading count" → 5
```

</details>

### 5. `Symbol.metadata` nima? `reflect-metadata` dan farqi?

<details>
<summary>Javob</summary>

TC39 decorator da metadata saqlash uchun native mexanizm:

```typescript
const ROLES = Symbol("roles");

function role(roleName: string) {
  return function (originalMethod: Function, context: ClassMethodDecoratorContext) {
    if (!context.metadata[ROLES]) context.metadata[ROLES] = {};
    (context.metadata[ROLES] as any)[String(context.name)] = roleName;
  };
}

class AdminController {
  @role("admin") deleteUser() {}
  @role("user") viewProfile() {}
}

const meta = AdminController[Symbol.metadata];
// meta[ROLES] → { deleteUser: "admin", viewProfile: "user" }
```

| | `reflect-metadata` | `Symbol.metadata` |
|---|---|---|
| Import kerak | ✅ | ❌ Native |
| API | `Reflect.defineMetadata` | `context.metadata` |
| Scope | Global | Class-scoped |
| Inheritance | Manual | Prototype chain (auto) |
| Standart | ❌ Third-party | ✅ ECMAScript |

</details>

### 6. TC39 da parameter decorator yo'q — qanday yechim bor?

<details>
<summary>Javob</summary>

TC39 proposal ga parameter decorator **kiritilmagan** — NestJS/Angular uchun katta muammo:

```typescript
// NestJS (legacy) — parameter decorator
getUser(@Param("id") id: string, @Query("fields") fields: string) {}
// TC39 da bu mumkin EMAS
```

**Yechimlar:**

```typescript
// 1. Method decorator + metadata
@params({ index: 0, source: "param", key: "id" },
        { index: 1, source: "query", key: "fields" })
getUser(id: string, fields: string) {}

// 2. Object parameter pattern
@Get(":id")
getUser(ctx: { params: { id: string }; query: { fields: string } }) {}
```

NestJS va Angular legacy da qolish sababi shu — parameter decorator larsiz DI va routing qayta yozish kerak. `addInitializer` + `Symbol.metadata` bilan partial workaround mavjud.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Decorator composition order — output (Daraja: Middle+)

**Savol:** Output ni ayting:

```typescript
function a() {
  console.log("a evaluated");
  return function (v: any, ctx: ClassMethodDecoratorContext) {
    console.log("a applied");
  };
}

function b() {
  console.log("b evaluated");
  return function (v: any, ctx: ClassMethodDecoratorContext) {
    console.log("b applied");
  };
}

function c(v: any, ctx: ClassDecoratorContext) {
  console.log("c applied");
}

@c
class Demo {
  @a()
  @b()
  method() {}
}
```

<details>
<summary>Yechim</summary>

```
a evaluated
b evaluated
b applied
a applied
c applied
```

Ikki bosqich:

1. **Evaluate** (yuqoridan pastga) — factory lar chaqiriladi: `a()` → `b()`
2. **Apply** (pastdan yuqoriga) — decorator funksiyalar qo'llanadi: `b` → `a`
   - Matematikadagi `a(b(method))` — funksiya kompozitsiyasi
3. **Class decorator** eng oxirida — barcha member decorator tugagandan keyin

</details>

### 2. Arrow function `this` xato — toping (Daraja: Middle+)

**Savol:** Bu decorator da xato bor. Toping va tuzating:

```typescript
function log(originalMethod: Function, context: ClassMethodDecoratorContext) {
  return (...args: any[]) => {
    console.log(`${String(context.name)} called`);
    return originalMethod.call(this, ...args);
  };
}

class UserService {
  name = "UserService";
  @log
  getUsers() { return `${this.name}: fetching`; }
}

console.log(new UserService().getUsers());
```

<details>
<summary>Yechim</summary>

**Xato:** Arrow function `this` ni lexical scope dan oladi (decorator scope) — class instance dan **emas**:

```typescript
// ❌ Arrow function — this decorator scope
return (...args: any[]) => {
  originalMethod.call(this, ...args); // this = undefined yoki decorator scope
};

// ✅ Regular function — this class instance
return function (this: any, ...args: any[]) {
  originalMethod.call(this, ...args); // this = class instance
};
```

Arrow function bilan `this.name` `undefined` bo'ladi. Regular function da `this` method chaqiruvi orqali class instance ga bog'lanadi.

</details>

### 3. `@memoize` decorator yozing (Daraja: Middle+)

**Savol:** TC39 standartida method natijasini cache qiladigan `@memoize` decorator yozing:

```typescript
class MathService {
  callCount = 0;
  @memoize
  fibonacci(n: number): number {
    this.callCount++;
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}
// math.fibonacci(10) → 55 (hisoblaydi)
// math.fibonacci(10) → 55 (cache dan, callCount oshmaydi)
```

<details>
<summary>Yechim</summary>

```typescript
function memoize(
  originalMethod: Function,
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
```

**Nuanslar:**
1. `cache` decorator scope da — **barcha instance** lar bir cache. Instance-level kerak bo'lsa `WeakMap<instance, Map>` ishlatish
2. `JSON.stringify(args)` — sodda key. Object argument lar uchun identity-based cache yaxshiroq
3. Async uchun `Promise` cache qilinadi — reject bo'lsa o'chirish kerak

</details>

### 4. `@singleton` class decorator yozing (Daraja: Middle+)

**Savol:** TC39 standartida — faqat bitta instance yaratilsin:

```typescript
@singleton
class Database {
  constructor(public url: string) { console.log(`Connected to ${url}`); }
}
// new Database("postgres://a") → "Connected to postgres://a"
// new Database("postgres://b") → constructor chaqirilMAYDI
// db1 === db2 → true
```

<details>
<summary>Yechim</summary>

```typescript
function singleton<T extends new (...args: any[]) => any>(
  value: T,
  context: ClassDecoratorContext
) {
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

**Qanday ishlaydi:** JS da constructor `return object` qilsa — `new` shu object ni qaytaradi (yangi yaratilgan o'rniga). Birinchi `new` da instance yaratiladi va saqlanadi, keyingilarida `return instance` — yangi constructor chaqirilmaydi.

</details>

### 5. Field + method + class order — output (Daraja: Middle+)

**Savol:** Output ni ayting:

```typescript
function field(value: undefined, context: ClassFieldDecoratorContext) {
  console.log(`field: ${String(context.name)}`);
  return function (initialValue: any) {
    console.log(`init: ${String(context.name)} = ${initialValue}`);
    return initialValue * 2;
  };
}

function method(value: Function, context: ClassMethodDecoratorContext) {
  console.log(`method: ${String(context.name)}`);
}

function cls(value: Function, context: ClassDecoratorContext) {
  console.log(`class: ${String(context.name)}`);
}

@cls
class Example {
  @field x = 10;
  @method greet() {}
  @field y = 20;
}

new Example();
```

<details>
<summary>Yechim</summary>

```
field: x
method: greet
field: y
class: Example
init: x = 10
init: y = 20
```

**Tartib:**

1. **Class define paytida** — member decorator lar body tartibida: `x` → `greet` → `y`
2. **Class decorator eng oxirida** — barcha member decorator tugagandan keyin
3. **`new Example()` da** — field initializer lar ishlaydi (har instance uchun)
4. Field decorator qaytargan funksiya `initialValue` ni oladi → `x = 20`, `y = 40` (`* 2`)

</details>

---

## Xulosa

- Decorator — runtime da ishlaydigan funksiya, type erasure ga uchramaydi
- Legacy (`experimentalDecorators`) vs TC39 — farqli API, TC39 standat
- Composition: evaluate yuqoridan pastga, apply pastdan yuqoriga (`a(b(x))`)
- Factory — parametr oluvchi decorator. Closure orqali konfiguratsiya
- `accessor` — auto getter/setter, decorator bilan read/write intercept
- `Symbol.metadata` — native metadata, `reflect-metadata` o'rniga
- TC39 da parameter decorator yo'q — NestJS/Angular legacy da qolish sababi
- Arrow function decorator da ishlatmang — `this` noto'g'ri bo'ladi

[Asosiy bo'limga qaytish →](../19-decorators.md)
