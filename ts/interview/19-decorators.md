# Interview: Decorators

> Decorator nima, Legacy vs TC39 standartlari, decorator factories, accessor keyword, Symbol.metadata, composition order, addInitializer, compiled output va real-world patterns bo'yicha interview savollari. Junior+ dan Senior darajagacha.

---

## Mundarija

### Nazariy savollar
1. [Decorator nima va qanday ishlaydi?](#savol-1-decorator-nima-va-qanday-ishlaydi-junior) [Junior+]
2. [Legacy va TC39 decorator'lar farqi?](#savol-2-legacy-va-tc39-decoratorlar-farqi-middle) [Middle]
3. [Decorator factory nima? Oddiy decorator'dan farqi?](#savol-3-decorator-factory-nima-oddiy-decoratordan-farqi-junior) [Junior+]
4. [Decorator turlari va signature'lari?](#savol-4-decorator-turlari-va-signaturelari-middle) [Middle]
5. [TC39 da `accessor` keyword nima?](#savol-5-tc39-da-accessor-keyword-nima-middle) [Middle+]
6. [`Symbol.metadata` va `reflect-metadata` farqi?](#savol-6-symbolmetadata-va-reflect-metadata-farqi-middle) [Middle+]
7. [TC39 da parameter decorator yo'q — qanday yechim?](#savol-7-tc39-da-parameter-decorator-yoq--qanday-yechim-senior) [Senior]
8. [`addInitializer` nima?](#savol-8-addinitializer-nima-middle) [Middle+]
9. [Legacy method decorator `PropertyDescriptor` qanday ishlatiladi?](#savol-9-legacy-method-decorator-propertydescriptor-qanday-ishlatiladi-middle) [Middle]
10. [Compiled output: `__decorate` vs `__esDecorate`?](#savol-10-compiled-output-__decorate-vs-__esdecorate-senior) [Senior]

### Output savollar
11. [Decorator composition order](#savol-11-decorator-composition-order-middle) [Middle+]
12. [Field + method + class order](#savol-12-field--method--class-order-middle) [Middle+]
13. [Factory + decorator execution timing](#savol-13-factory--decorator-execution-timing-middle) [Middle+]

### Coding savollar
14. [`@memoize` decorator yozing](#savol-14-memoize-decorator-yozing-middle) [Middle+]
15. [`@singleton` class decorator yozing](#savol-15-singleton-class-decorator-yozing-middle) [Middle+]
16. [`@retry` decorator factory yozing](#savol-16-retry-decorator-factory-yozing-middle) [Middle+]
17. [`@authorize` decorator factory yozing](#savol-17-authorize-decorator-factory-yozing-middle) [Middle+]

### Bug fix savollar
18. [Arrow function `this` xato — toping va tuzating](#savol-18-arrow-function-this-xato--toping-va-tuzating-middle) [Middle+]
19. [Field decorator getter/setter olishga urinish](#savol-19-field-decorator-gettersetter-olishga-urinish-middle) [Middle+]

---

## Nazariy savollar

### Savol 1: Decorator nima va qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Decorator — class yoki class a'zolariga qo'shimcha behavior qo'shadigan funksiya, `@expression` syntax bilan qo'llanadi. Type annotation'lardan farqli o'laroq runtime da ishlaydi — JS ga compile qilingandan keyin ham qoladi.

### To'liq tushuntirish

Decorator — meta-programming mexanizmi. Class declaration, method, property, accessor yoki parameter dan oldin `@decoratorName` deb yoziladi va shu element'ga avtomatik ravishda qo'llaniladi.

**Nima uchun kerak:**
1. **Cross-cutting concerns** — logging, validation, caching, authorization. Business logic dan ajratib infrastructure logic ni decorator orqali qo'shish
2. **Separation of Concerns** — har class faqat o'z asosiy ishiga e'tibor qaratadi
3. **Declarative programming** — imperative kod o'rniga "nima kerak" deb belgilash
4. **Code reuse** — bir behavior ni ko'p joyda qayta ishlatish

**Mexanizm:** Decorator — oddiy funksiya. Compiler decorate qilingan element'ni o'rab, decorator funksiyasiga argument sifatida uzatadi. Decorator yangi versiyani qaytarishi yoki original ni o'zgartirishi mumkin.

**Type annotation'dan farqi:** Interface va type alias compile-time da o'chiriladi (type erasure). Decorator — runtime artifact. Compiled JS da `@log method() {}` o'rniga `__decorate([log], ...)` chaqiruvi qoladi.

### Kod misol

```typescript
// TC39 method decorator
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  const methodName = String(context.name);
  return function (this: any, ...args: any[]) {
    console.log(`-> ${methodName}(${args.join(", ")})`);
    const result = originalMethod.call(this, ...args);
    console.log(`<- ${methodName} =`, result);
    return result;
  };
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

new Calculator().add(2, 3);
// -> add(2, 3)
// <- add = 5
```

### Edge Cases

- Decorator faqat **class va class a'zolariga** qo'llanadi — standalone funksiyaga, arrow function'ga, plain object'ga emas
- Decorator order muhim — composition tartibida (`@a @b method` → `a(b(method))`)
- Method decorator yangi funksiya qaytarsa — original almashtiriladi; `void` qaytarsa — original qoladi
- Decorator runtime ish bajarganda type information yo'qoladi — TS interface va generic decorator'ga ko'rinmaydi

### Follow-up savollar

1. **"Decorator type annotation'dan qanday farqi bor compiled output da?"** — Type annotation o'chiriladi, decorator esa `__decorate`/`__esDecorate` chaqiruvi sifatida qoladi
2. **"Standalone funksiyaga decorator qo'yish mumkinmi?"** — Yo'q, faqat class context'da. Funksiya uchun higher-order function pattern ishlatiladi: `const fn = log(myFunction)`

</details>

---

### Savol 2: Legacy va TC39 decorator'lar farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Legacy — `experimentalDecorators: true` bilan ishlovchi eski TypeScript implementation. TC39 — TS 5.0+ da default, ECMAScript Stage 3 standart. API, signature, va imkoniyatlar bir-biridan tubdan farq qiladi.

### To'liq tushuntirish

| Xususiyat | Legacy | TC39 (TS 5.0+) |
|-----------|--------|-----------------|
| Yoqish | `experimentalDecorators: true` | Default (config kerak emas) |
| Method args | `target, propertyKey, descriptor` | `value, context` |
| Class args | `constructor` | `value, context` |
| Property decorator | `target, propertyKey` | Field: `value, context` (initializer return) |
| Parameter decorator | Mavjud | Yo'q (standartlashtirilmagan) |
| `accessor` keyword | Yo'q | Mavjud |
| Metadata | `reflect-metadata` polyfill | `Symbol.metadata` native (TS 5.2+) |
| `emitDecoratorMetadata` | Ishlaydi | Ishlamaydi |
| `addInitializer` | Yo'q | Mavjud |
| Compiled helper | `__decorate` | `__esDecorate`, `__runInitializers` |
| ECMAScript standart | Proposal-based | Stage 3 |

**Asosiy farqlar tahlili:**

1. **Context API** — TC39 da har decorator yagona `context` object oladi (`kind`, `name`, `static`, `private`, `access`, `addInitializer`, `metadata`). Legacy da har turi uchun alohida argument'lar.

2. **Parameter decorator yo'qligi** — TC39 da bu chiqarib tashlangan. Bu NestJS va Angular ning legacy da qolishining asosiy sababi — DI ularga `@inject()` parameter decorator zarur.

3. **`accessor` keyword** — TC39 da auto getter/setter avtomatik yaratiladi, decorator getter va setter'ni bir vaqtda intercept qila oladi.

4. **Native metadata** — `Symbol.metadata` orqali. Class-scoped, prototype chain orqali inherited. `reflect-metadata` global WeakMap edi.

**Qachon qaysi biri:**
- **Legacy** — Angular, NestJS, TypeORM, MikroORM, mavjud production loyihalar
- **TC39** — yangi loyihalar, library author'lar, standartga moslik kerak bo'lganda

### Kod misol

```typescript
// === Legacy method decorator ===
function logLegacy(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`${propertyKey} called`);
    return original.apply(this, args);
  };
  return descriptor;
}

// === TC39 method decorator ===
function logTC39(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: any[]) {
    console.log(`${String(context.name)} called`);
    return originalMethod.call(this, ...args);
  };
}

// Ishlatish bir xil:
class UserService {
  @logLegacy  // experimentalDecorators: true bilan
  getUserLegacy() { return "user"; }

  @logTC39    // default (TS 5.0+)
  getUserTC39() { return "user"; }
}
```

### Edge Cases

- **Aralash ishlatib bo'lmaydi:** `experimentalDecorators: true` yoqilganda barcha decorator'lar legacy mode da interpret qilinadi
- **`emitDecoratorMetadata` TC39 da ta'siri yo'q** — DI container yozish uchun TC39 da boshqa yo'l (token + `Symbol.metadata`)
- **Class decorator return type** — Legacy: `void | typeof constructor`. TC39: `void | new (...args: any[]) => any`
- **Static method:** Legacy da `target` constructor, TC39 da `context.static === true`

### Follow-up savollar

1. **"NestJS nima uchun hali legacy da?"** — Parameter decorator yo'qligi sababli. `@Param`, `@Body`, `@Query` qayta yozish katta breaking change
2. **"TC39 ga migration qanday?"** — Tsconfig'dan `experimentalDecorators` olib tashlash, decorator signature'larni yangilash, `reflect-metadata` o'rniga `Symbol.metadata`

</details>

---

### Savol 3: Decorator factory nima? Oddiy decorator'dan farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Decorator factory — parametr qabul qilib, decorator qaytaradigan funksiya. Oddiy decorator `@log` deb yoziladi, factory `@log("debug")` deb chaqiriladi.

### To'liq tushuntirish

Oddiy decorator — to'g'ridan-to'g'ri qo'llaniladigan funksiya. Factory — yana bir qatlam — uni avval chaqirib, qaytgan funksiya decorator sifatida ishlatiladi. Closure orqali ichki decorator outer parametr'larni capture qiladi.

**Nima uchun kerak:** decorator'ni configuration qilish. Masalan, `@retry(3)` — 3 marta urinish, `@cache(60000)` — 60 sekund cache. Bir kodda turli xulq.

**Mexanizm:**
1. Compiler `@log("debug")` ko'radi → `log("debug")` ni evaluate qiladi (factory chaqiriladi)
2. Qaytgan funksiya decorator sifatida target'ga qo'llaniladi
3. Inner decorator closure orqali factory parametr'larini eslab qoladi

**Composition timing:**
- **Evaluate** (factory chaqiruvi) — yuqoridan pastga
- **Apply** (decorator qo'llanishi) — pastdan yuqoriga

### Kod misol

```typescript
// === Oddiy decorator — parametrsiz ===
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: any[]) {
    console.log(`${String(context.name)} called`);
    return originalMethod.call(this, ...args);
  };
}

// === Decorator factory — parametr oladi ===
function logWithLevel(level: "info" | "warn" | "error") {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return function (this: any, ...args: any[]) {
      console[level](`[${level.toUpperCase()}] ${String(context.name)}`);
      return originalMethod.call(this, ...args);
    };
  };
}

class PaymentService {
  @log                          // Oddiy
  processPayment() { return "ok"; }

  @logWithLevel("warn")         // Factory — parametrli
  refundPayment() { return "refunded"; }
}

new PaymentService().processPayment();
// "processPayment called"

new PaymentService().refundPayment();
// [WARN] refundPayment
```

### Edge Cases

- Factory **har class definition da** chaqiriladi — agar factory `console.log` qilsa, har class uchun ko'rinadi
- Factory parametri **closure orqali** ichki decorator'da yashaydi — har decorate qilingan element o'z parametri bilan
- **Composition order:** factory yuqoridan pastga evaluate, decorator pastdan yuqoriga apply. `@a() @b() method` → `a()` evaluate → `b()` evaluate → `b` apply → `a` apply
- Factory **side effect** beruvchi kod yozmang — class definition vaqtida ishlaydi, kod startup performance ga ta'sir qiladi

### Follow-up savollar

1. **"Oddiy decorator'ni factory'ga aylantirish kerakmi?"** — Faqat parametr kerak bo'lsa. Sabab: factory qo'shimcha bir level closure, syntax `@log()` (qavsli) — code reader uchun signal: "configuration bor"
2. **"Factory ichida yana factory bo'lishi mumkinmi?"** — Mumkin, lekin kamdan-kam. Curried decorators — `@auth("admin")("write")` ko'rinishidagi konstruksiyalar over-engineering

</details>

---

### Savol 4: Decorator turlari va signature'lari? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TC39 da 5 ta decorator turi: class, method, getter, setter, field, accessor (auto). Legacy da qo'shimcha parameter decorator. Har turi o'z context object i bilan keladi.

### To'liq tushuntirish

**TC39 decorator signature'lari (TS 5.0+):**

```typescript
// 1. Class decorator
type ClassDecorator = (
  value: new (...args: any[]) => any,
  context: ClassDecoratorContext
) => (new (...args: any[]) => any) | void;

// 2. Method decorator
type MethodDecorator = (
  value: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) => ((...args: any[]) => any) | void;

// 3. Getter decorator
type GetterDecorator = (
  value: () => any,
  context: ClassGetterDecoratorContext
) => (() => any) | void;

// 4. Setter decorator
type SetterDecorator = (
  value: (v: any) => void,
  context: ClassSetterDecoratorContext
) => ((v: any) => void) | void;

// 5. Field decorator (yangi initializer qaytaradi)
type FieldDecorator = (
  value: undefined,
  context: ClassFieldDecoratorContext
) => ((initialValue: any) => any) | void;

// 6. Accessor (auto) decorator
type AccessorDecorator = (
  value: { get: () => any; set: (v: any) => void },
  context: ClassAccessorDecoratorContext
) => { get?: () => any; set?: (v: any) => void; init?: (v: any) => any } | void;
```

**Context object umumiy maydonlari (`ClassMemberDecoratorContext`):**
- `kind` — "class" | "method" | "getter" | "setter" | "field" | "accessor"
- `name` — `string | symbol`
- `static` — `boolean`
- `private` — `boolean`
- `access` — `{ get?: (obj) => value; set?: (obj, v) => void; has?: (obj) => boolean }`
- `addInitializer(fn)` — class yoki instance initializer
- `metadata` — `Symbol.metadata` ga yoziladigan object

**Return value semantikasi:**
- `void` qaytarsa — original o'zgarmaydi
- Yangi value qaytarsa — original almashtiriladi
- Field decorator — initializer funksiya qaytaradi (initial value transform uchun)
- Accessor — `{ get, set, init }` object qaytaradi (qisman replace)

### Kod misol

```typescript
// Class decorator
function sealed(
  value: new (...args: any[]) => any,
  context: ClassDecoratorContext
) {
  Object.seal(value);
  Object.seal(value.prototype);
}

// Method decorator
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: any[]) {
    console.log(`${String(context.name)}(${args.join(",")})`);
    return originalMethod.call(this, ...args);
  };
}

// Getter decorator
function logGetter(
  originalGetter: () => any,
  context: ClassGetterDecoratorContext
) {
  return function (this: any) {
    const value = originalGetter.call(this);
    console.log(`get ${String(context.name)} =`, value);
    return value;
  };
}

// Field decorator — initial value transform
function uppercase(
  value: undefined,
  context: ClassFieldDecoratorContext
) {
  return function (initialValue: string) {
    return initialValue.toUpperCase();
  };
}

// Accessor decorator
function tracked(
  value: { get: () => any; set: (v: any) => void },
  context: ClassAccessorDecoratorContext
) {
  return {
    get() {
      console.log(`reading ${String(context.name)}`);
      return value.get.call(this);
    },
    set(this: any, newValue: any) {
      console.log(`setting ${String(context.name)}`);
      value.set.call(this, newValue);
    },
  };
}

@sealed
class UserProfile {
  @uppercase
  username: string = "ali";

  @tracked
  accessor email: string = "";

  @logGetter
  get displayName(): string { return this.username; }

  @log
  save(): void { /* ... */ }
}
```

### Edge Cases

- **`kind` discriminant** — context type'ni narrow qilish uchun: `if (context.kind === "method") { ... }`
- **Private member** — `context.private === true`. Decorator orqali private field'ga access — `context.access.get(obj)`
- **Static member** — `context.static === true`. Class definition vaqtida ishlaydi, instance'da emas
- **Field decorator field value ololmaydi** — `value: undefined`. Faqat initializer transform mumkin. Getter/setter kerak bo'lsa `accessor` keyword ishlatiladi

### Follow-up savollar

1. **"Field va accessor decorator orasidagi farq?"** — Field: faqat initial value transform, get/set yo'q. Accessor: get/set funksiyalari beriladi, read/write intercept qilish mumkin
2. **"`context.access` qanday holda kerak?"** — Decorator orqali boshqa decorator'ga access berish: registry pattern (`addInitializer` ichida `context.access.get(this)` orqali instance value olish)

</details>

---

### Savol 5: TC39 da `accessor` keyword nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`accessor` — class field oldidan yoziladigan keyword. Compiler avtomatik private storage + getter + setter pair yaratadi. Decorator bilan ishlatilganda getter va setter'ni bir vaqtda intercept qilish imkonini beradi.

### To'liq tushuntirish

`accessor name = "default"` quyidagicha desugar qilinadi:

```typescript
#name = "default";
get name() { return this.#name; }
set name(v) { this.#name = v; }
```

**Nima uchun kerak:**

Field decorator (oddiy `@logged name = ""`) faqat `initialValue` transform qila oladi — getter/setter yo'q. Lekin runtime da read/write track qilish kerak bo'lsa (reactivity, validation, logging) — getter/setter zarur. `accessor` keyword bu pair ni avtomatik yaratadi va decorator'ga `{ get, set }` object beradi.

**Decorator API:** Accessor decorator `{ get, set, init }` object qaytaradi:
- `get` — getter'ni almashtiradi
- `set` — setter'ni almashtiradi
- `init` — initial value'ni transform qiladi

**Reactivity uchun ideal:** MobX, SolidJS, Lit reactive framework'lari accessor decorator orqali avtomatik signal yoki observable wrap qiladi.

### Kod misol

```typescript
function tracked(
  value: { get: () => any; set: (v: any) => void },
  context: ClassAccessorDecoratorContext
) {
  const fieldName = String(context.name);

  return {
    get() {
      console.log(`reading ${fieldName}`);
      return value.get.call(this);
    },
    set(this: any, newValue: any) {
      console.log(`writing ${fieldName} =`, newValue);
      value.set.call(this, newValue);
    },
    init(initialValue: any) {
      console.log(`init ${fieldName} =`, initialValue);
      return initialValue;
    },
  };
}

class CartState {
  @tracked accessor itemCount: number = 0;
  @tracked accessor total: number = 0;
}

const cart = new CartState();
// init itemCount = 0
// init total = 0

cart.itemCount = 3;
// writing itemCount = 3

console.log(cart.itemCount);
// reading itemCount
// 3
```

### Edge Cases

- **`accessor` siz field decorator getter/setter olmaydi** — faqat initializer transform. Read/write intercept kerak bo'lsa accessor majburiy
- **Private accessor:** `accessor #count = 0` — public accessor o'rniga private getter/setter pair, backing storage ham private field (TS uni `#count_accessor_storage` ko'rinishida nomlaydi)
- **`init` field initializer transform qiladi** — accessor decorator ham field decorator vazifasini bajaradi
- **`return undefined` ruxsat** — har property optional (`{ get: ..., set: undefined, init: ... }` — faqat getter'ni intercept)
- **TS 4.9+ kerak** — `accessor` syntax o'zi (decorator-free) shu versiyadan
- **Performance:** auto-accessor — private field'ga access. Manual getter/setter'dan farqsiz, lekin code less verbose

### Follow-up savollar

1. **"`accessor` `prop` `getter/setter` qoldirish dan qanday farqi bor?"** — Manual `#name` + `get/set` — boilerplate. `accessor` — bir qator, decorator bilan API yaxshi
2. **"Reactivity uchun nima uchun ideal?"** — Set chaqirilganda dependent computation invalidate qilish oson — accessor decorator orqali har set ga callback ulanadi

</details>

---

### Savol 6: `Symbol.metadata` va `reflect-metadata` farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`reflect-metadata` — legacy decorators bilan ishlovchi prototype polyfill (rasmiy TC39 proposal bo'lib ulgurmagan, hech qanday Stage'dan o'tmagan). `Symbol.metadata` — TC39 decorators'ning native mexanizmi, Stage 3 ECMAScript, class-scoped, prototype chain orqali inherited.

### To'liq tushuntirish

| Xususiyat | `reflect-metadata` | `Symbol.metadata` |
|-----------|-------------------|-------------------|
| Import kerak | `import "reflect-metadata"` | Yo'q — native |
| API | `Reflect.defineMetadata`, `Reflect.getMetadata` | `context.metadata`, `Class[Symbol.metadata]` |
| Storage | Global WeakMap | Class object'da `[Symbol.metadata]` property |
| Scope | Global key-value | Class-scoped |
| Inheritance | Manual (`getOwnMetadata` vs `getMetadata`) | Prototype chain orqali avtomatik |
| Decorator bilan | Legacy faqat | TC39 faqat |
| `emitDecoratorMetadata` | Ishlaydi | Ishlamaydi |
| Standart holati | Rasmiy proposal emas (prototype) | Stage 3 (ECMAScript) |
| Bundle | Polyfill kerak (runtime ga qo'shiladi) | Native bo'lsa 0 |
| Polyfill | Browser/Node uchun kerak | Aksariyat runtime hali polyfill talab qiladi |

**`reflect-metadata` (legacy) misol:**

```typescript
import "reflect-metadata";

function inject(token: any) {
  return function (target: any, _key: any, paramIndex: number) {
    const tokens = Reflect.getOwnMetadata("inject:tokens", target) || new Map();
    tokens.set(paramIndex, token);
    Reflect.defineMetadata("inject:tokens", tokens, target);
  };
}
```

**`Symbol.metadata` (TC39) misol:**

```typescript
const ROLES_KEY = Symbol("roles");

function role(name: string) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    if (!context.metadata[ROLES_KEY]) {
      context.metadata[ROLES_KEY] = {};
    }
    (context.metadata[ROLES_KEY] as Record<string, string>)[String(context.name)] = name;
  };
}

class AdminController {
  @role("admin") deleteUser() {}
  @role("moderator") banUser() {}
}

const meta = AdminController[Symbol.metadata];
console.log(meta?.[ROLES_KEY]);
// { deleteUser: "admin", banUser: "moderator" }
```

**Inheritance:**

```typescript
class Base {
  @role("user") view() {}
}

class Admin extends Base {
  @role("admin") delete() {}
}

const adminMeta = Admin[Symbol.metadata]?.[ROLES_KEY];
console.log(adminMeta);
// { view: "user", delete: "admin" }
// Prototype chain orqali Base metadata avtomatik inherited
```

### Kod misol

```typescript
// === TC39 routing system Symbol.metadata bilan ===
const ROUTES_KEY = Symbol("routes");

interface RouteInfo { method: string; path: string; handler: string; }

function route(method: string, path: string) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    if (!context.metadata[ROUTES_KEY]) {
      context.metadata[ROUTES_KEY] = [];
    }
    (context.metadata[ROUTES_KEY] as RouteInfo[]).push({
      method,
      path,
      handler: String(context.name),
    });
  };
}

class UserController {
  @route("GET", "/users") list() { return []; }
  @route("GET", "/users/:id") get() { return {}; }
  @route("POST", "/users") create() { return {}; }
}

const routes = UserController[Symbol.metadata]?.[ROUTES_KEY] as RouteInfo[];
console.log(routes);
// [
//   { method: "GET", path: "/users", handler: "list" },
//   { method: "GET", path: "/users/:id", handler: "get" },
//   { method: "POST", path: "/users", handler: "create" },
// ]
```

### Edge Cases

- **`context.metadata` qaytadan o'qish:** decorator ichida yozilgan metadata `Class[Symbol.metadata]` orqali tashqaridan o'qiladi
- **Prototype chain:** child class metadata parent dan inherit qilinadi — lekin child override qilsa, parent ko'rilmaydi
- **Runtime support:** `context.metadata` API TypeScript 5.2+ da emit qilinadi, lekin native `Symbol.metadata` aksariyat runtime'da hali yo'q — polyfill kerak: `(Symbol as { metadata?: symbol }).metadata ??= Symbol("Symbol.metadata")`
- **Legacy aralash:** bir loyihada `reflect-metadata` va `Symbol.metadata` ikkalasi mavjud bo'lsa — ikki turli metadata storage. Decorator standarti bilan bog'liq

### Follow-up savollar

1. **"NestJS Symbol.metadata'ga ko'chadimi?"** — Hozircha yo'q — parameter decorator yo'qligi blocking issue. Hammasi qayta yozilishi kerak
2. **"Performance farq bormi?"** — `Symbol.metadata` — class object'da to'g'ridan-to'g'ri property access. `reflect-metadata` — global WeakMap lookup. Native tezroq, lekin farq ko'pchilik holatda sezilmaydi

</details>

---

### Savol 7: TC39 da parameter decorator yo'q — qanday yechim? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TC39 proposal ga parameter decorator kiritilmagan. Yechimlar: method decorator + metadata, object parameter pattern, yoki Symbol-based DI token. NestJS va Angular ning legacy da qolishining asosiy sababi shu.

### To'liq tushuntirish

**Muammo:** Legacy da `@inject(token) param: Type` yozish mumkin edi:

```typescript
// Legacy — ishlaydi
class UserService {
  getUser(@Param("id") id: string, @Query("limit") limit: number) {}
}
```

TC39 da bunday syntax mavjud emas — compiler ham qabul qilmaydi.

**Yechim 1: Method decorator + metadata (parametr ma'lumotini method'ga yozish)**

```typescript
const PARAMS_KEY = Symbol("params");

interface ParamMeta {
  index: number;
  source: "param" | "query" | "body" | "header";
  key: string;
}

function param(meta: Omit<ParamMeta, "index">[]) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    if (!context.metadata[PARAMS_KEY]) {
      context.metadata[PARAMS_KEY] = {};
    }
    (context.metadata[PARAMS_KEY] as Record<string, ParamMeta[]>)[String(context.name)] =
      meta.map((m, i) => ({ ...m, index: i }));
  };
}

class UserController {
  @param([
    { source: "param", key: "id" },
    { source: "query", key: "limit" },
  ])
  getUser(id: string, limit: number) {
    return { id, limit };
  }
}
```

**Yechim 2: Object parameter pattern (single object'da barcha parametr)**

```typescript
class UserController {
  @route("GET", "/users/:id")
  getUser(ctx: {
    params: { id: string };
    query: { limit?: number };
    body?: unknown;
  }) {
    return { id: ctx.params.id, limit: ctx.query.limit };
  }
}
```

Framework adapter request'dan to'g'ri object yaratadi. Type safety to'liq, parameter decorator kerak emas.

**Yechim 3: Symbol-based DI token + constructor**

```typescript
const TOKENS = {
  Logger: Symbol("Logger"),
  Database: Symbol("Database"),
} as const;

// Class-level decorator metadata orqali constructor parametr tokenlarini saqlash
function injectable(tokens: symbol[]) {
  return function <T extends new (...args: any[]) => any>(
    value: T,
    context: ClassDecoratorContext
  ) {
    context.metadata["di:tokens"] = tokens;
    return value;
  };
}

@injectable([TOKENS.Logger, TOKENS.Database])
class UserService {
  constructor(private logger: any, private db: any) {}
}

// Container constructor tokenlarini class metadata'dan o'qiydi
```

**Yechim 4: `addInitializer` orqali class init time da parametr metadata**

```typescript
function paramMeta(index: number, source: string) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    context.addInitializer(function (this: any) {
      // Class init time — instance'ga metadata biriktirish
    });
  };
}
```

### Kod misol

```typescript
// === To'liq routing example TC39 da ===
const ROUTES_KEY = Symbol("routes");
const PARAMS_KEY = Symbol("params");

interface ParamConfig {
  index: number;
  type: "param" | "query" | "body";
  name: string;
}

interface RouteConfig {
  method: string;
  path: string;
  handler: string;
  params: ParamConfig[];
}

function route(method: string, path: string, params: Omit<ParamConfig, "index">[] = []) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    if (!context.metadata[ROUTES_KEY]) {
      context.metadata[ROUTES_KEY] = [];
    }
    (context.metadata[ROUTES_KEY] as RouteConfig[]).push({
      method,
      path,
      handler: String(context.name),
      params: params.map((p, i) => ({ ...p, index: i })),
    });
  };
}

class ArticleController {
  @route("GET", "/articles/:id", [
    { type: "param", name: "id" },
  ])
  getArticle(id: string) {
    return { id, title: "Sample" };
  }

  @route("POST", "/articles", [
    { type: "body", name: "data" },
  ])
  createArticle(data: { title: string }) {
    return { id: "new", ...data };
  }
}

// Router framework metadata'ni o'qib request'larni map qiladi
const routes = ArticleController[Symbol.metadata]?.[ROUTES_KEY] as RouteConfig[];
console.log(routes);
```

### Edge Cases

- **Type safety yo'qoladi qisman** — method decorator parametr metadata'da string-based, compiler tekshirmaydi `key === parametr nomi` ekanini
- **Refactoring xavfi** — parametr nomini o'zgartirsangiz, metadata ham o'zgarishi kerak (avtomatik emas)
- **IDE autocomplete cheklangan** — string literal'lar bilan ishlaydi, parametr nomi bilan emas
- **Compile-time error yo'q** — agar metadata `params: ["id", "name"]` deylsa va method `getUser(id, role)` bo'lsa — runtime da topiladi

### Follow-up savollar

1. **"Nima uchun TC39 parameter decorator olib tashlandi?"** — Standartlashtirish murakkabligi: positional argument'lar shape ni o'zgartiradi, lexical scope ga kira oladi, type system bilan mosligi muammoli
2. **"NestJS qachon TC39 ga ko'chadi?"** — Parameter decorator proposal qayta tiklanmaguncha emas. Hozir community alohida proposal ustida ishlamoqda (`@decorator on parameter`)
3. **"DI uchun eng yaxshi pattern qaysi?"** — Constructor injection + class-level token list. Function-based DI (factory function) — TC39 bilan eng mos

<details>
<summary><strong>Deep Dive</strong></summary>

**TC39 parameter decorator proposal — to'xtab qolish sabablari:**

TC39 da parameter decorator alohida proposal bo'lib bordi (Stage 1), keyin asosiy decorator proposal'dan ajratildi. Asosiy sabablar:

1. **Parameter shape immutable emas** — parameter decorator parameter type'ni o'zgartira oladi: `@inject(Logger) log: Logger` syntax type-system'da paradox. Compiler `Logger` ni constructor argument deb biladi, lekin decorator runtime'da `LoggerImpl` yuborishi mumkin. Type information va runtime behavior nomos.

2. **Lexical scope access** — Legacy parameter decorator parametr lexical environment'ga kira olardi (closure capture). TC39 strict spec'da bu mumkin emas — parametr scope hali yaratilmagan.

3. **Compositions confusion** — `@a @b method(@c param)` — `@c` qaysi tartibda ishlaydi? Method decorator'dan oldinmi, keyinmi? Spec'da aniq javob yo'q.

**Hozirgi DI implementation — Symbol token + class metadata:**

```typescript
// === To'liq DI Container TC39 ga moslab ===
const TOKENS_KEY = Symbol("di:tokens");

interface Token<T> { readonly description: string; readonly _type?: T }
function createToken<T>(description: string): Token<T> {
  return { description } as Token<T>;
}

// Service tokenlari
const LoggerToken = createToken<Logger>("Logger");
const DatabaseToken = createToken<Database>("Database");

// Class-level decorator — constructor parametrlari uchun tokenlar
function injectable(...tokens: Token<unknown>[]) {
  return function <T extends new (...args: any[]) => any>(
    value: T,
    context: ClassDecoratorContext
  ) {
    context.metadata[TOKENS_KEY] = tokens;
    return value;
  };
}

interface Logger { log(msg: string): void }
interface Database { query(sql: string): Promise<unknown> }

@injectable(LoggerToken, DatabaseToken)
class OrderService {
  constructor(private logger: Logger, private db: Database) {}
  async createOrder(amount: number) {
    this.logger.log(`Creating order: ${amount}`);
    return this.db.query(`INSERT INTO orders ...`);
  }
}

// Container — service'larni token bo'yicha resolve
class Container {
  private providers = new Map<Token<any>, () => any>();

  register<T>(token: Token<T>, factory: () => T): void {
    this.providers.set(token, factory);
  }

  resolve<T extends new (...args: any[]) => any>(Class: T): InstanceType<T> {
    const tokens = (Class as any)[Symbol.metadata]?.[TOKENS_KEY] as Token<unknown>[] | undefined;
    if (!tokens) return new Class();
    const args = tokens.map(token => {
      const factory = this.providers.get(token);
      if (!factory) throw new Error(`Provider not registered: ${token.description}`);
      return factory();
    });
    return new Class(...args);
  }
}
```

**TC39 proposal restart hozirgi holati:** community alohida `parameter-decorators` proposal'ni qayta jonlantirgan (Stage 1, 2024). Hozircha decorator proposal stable bo'lganidan keyin parameter decorator qayta ko'rib chiqilishi taxmin qilinmoqda.

</details>

</details>

---

### Savol 8: `addInitializer` nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`addInitializer` — TC39 decorator context method. Class definition tugagandan keyin (class decorator) yoki har instance yaratilganda (member decorator) chaqiriladigan callback ro'yxatga olish uchun ishlatiladi.

### To'liq tushuntirish

`context.addInitializer(fn)` — class lifecycle ning aniq nuqtasiga callback ulaydi. Decorator return value (yangi method) ga qaraganda alohida mexanizm — side effect kerak bo'lsa.

**Qaerda chaqiriladi:**

| Decorator kind | Initializer chaqiriladi |
|----------------|------------------------|
| Class decorator | Class definition tugagach (bir marta) |
| Static method/field | Class definition vaqtida (bir marta) |
| Instance method | Har instance constructor'da |
| Instance field | Har instance constructor'da, field set bo'lgandan keyin |
| Accessor | Har instance constructor'da |

**`this` context:**
- Class decorator: `this` = class constructor
- Static member: `this` = class constructor
- Instance member: `this` = instance

**Use cases:**
1. **Method binding** — `this.method = this.method.bind(this)`
2. **Event listener** — instance yaratilganda avtomatik subscribe
3. **Validation** — barcha decorator'lar qo'llanganidan keyin invariant check
4. **Registry pattern** — class definition tugagach global ga ro'yxat qo'shish
5. **`Object.defineProperty`** — descriptor manipulation kerak bo'lganda

**Decorator return va `addInitializer` farqi:**
- Return — funksiya **structure** ni o'zgartiradi (method replace, getter/setter)
- `addInitializer` — **side effect** beradi, lifecycle hook

### Kod misol

```typescript
// === @bound — method'ni instance'ga bind qilish ===
function bound(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  context.addInitializer(function (this: any) {
    this[context.name] = this[context.name].bind(this);
  });
}

class UserEventHandler {
  username = "Ali";

  @bound
  onClick() {
    console.log(`Click by ${this.username}`);
  }
}

const handler = new UserEventHandler();
const fn = handler.onClick;
fn(); // "Click by Ali" — this saqlanadi


// === @autoRegister — class'ni global registry'ga qo'shish ===
const componentRegistry = new Map<string, new (...args: any[]) => any>();

function autoRegister(
  value: new (...args: any[]) => any,
  context: ClassDecoratorContext
) {
  context.addInitializer(function (this: any) {
    componentRegistry.set(String(context.name), this);
  });
}

@autoRegister
class HeaderComponent {}

@autoRegister
class FooterComponent {}

console.log(componentRegistry.size);  // 2
console.log(componentRegistry.get("HeaderComponent")); // [class HeaderComponent]


// === @readonly — method'ni writable: false qilish ===
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
  getDbUrl(): string { return "postgresql://localhost/app"; }
}

const cfg = new Config();
// (cfg as any).getDbUrl = () => "hacked";  // TypeError — writable: false
```

### Edge Cases

- **Order:** decorator return birinchi qo'llaniladi, keyin `addInitializer` chaqiriladi. Initializer ichida `this[context.name]` allaqachon decorated version
- **`this` static method'da** — class constructor, instance emas. Static initializer'da `this[fieldName]` static field'ga murojaat
- **Initializer return value** ignored — `void`. Side effect uchun ishlatish kerak
- **Initializer order** — bir element uchun bir nechta initializer registered bo'lsa, evaluation order da bajariladi (top-to-bottom decorator factory order)
- **Private member uchun** — `this[context.name]` ishlamaydi (private member dinamik access yo'q). `context.access.get(this)` ishlatish kerak

### Follow-up savollar

1. **"`addInitializer` `constructor` ichidagi `bind` dan qanday farqi?"** — Decorator deklarativ, har class'da `constructor` da `this.method = this.method.bind(this)` yozish boilerplate
2. **"`return` va `addInitializer` ikkalasini birga ishlatish mumkinmi?"** — Mumkin va keng tarqalgan. `return` method'ni replace qiladi, `addInitializer` instance'ga side effect

</details>

---

### Savol 9: Legacy method decorator `PropertyDescriptor` qanday ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Legacy method decorator 3 ta argument oladi: `target` (prototype yoki constructor), `propertyKey` (method nomi), `descriptor` (`PropertyDescriptor` — value, writable, enumerable, configurable). Decorator descriptor'ni o'zgartirib yoki yangi descriptor qaytarib method'ni boshqaradi.

### To'liq tushuntirish

**PropertyDescriptor anatomy:**

```typescript
interface PropertyDescriptor {
  value?: any;              // Method funksiyasi
  writable?: boolean;       // O'zgartirish mumkinmi
  enumerable?: boolean;     // for..in da ko'rinadimi
  configurable?: boolean;   // Qayta define qilish mumkinmi
  get?: () => any;          // Getter (accessor uchun)
  set?: (v: any) => void;   // Setter
}
```

**Method decorator signature:**

```typescript
function decorator(
  target: any,              // Instance method: prototype. Static: constructor
  propertyKey: string,      // Method nomi
  descriptor: PropertyDescriptor  // value: method funksiyasi
): PropertyDescriptor | void;
```

**Asosiy pattern'lar:**

1. **Original method'ni saqlab, wrap qilish** (`descriptor.value` ni almashtirish)
2. **Yangi descriptor qaytarish** (Object.defineProperty natijasiga ta'sir)
3. **Faqat metadata yozish** (descriptor o'zgarmaydi)

**`apply` vs `call`:** Wrap qilingan method ichida `originalMethod.apply(this, args)` — array bilan, `originalMethod.call(this, ...args)` — spread bilan. Funksional jihatdan bir xil.

### Kod misol

```typescript
// === @log — chaqiruv va natijani console ga ===
function log(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const original = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`-> ${propertyKey}(${args.join(", ")})`);
    const result = original.apply(this, args);
    console.log(`<- ${propertyKey} =`, result);
    return result;
  };

  return descriptor;
}

class OrderService {
  @log
  calculateTotal(price: number, quantity: number): number {
    return price * quantity;
  }
}

new OrderService().calculateTotal(100, 3);
// -> calculateTotal(100, 3)
// <- calculateTotal = 300


// === @readonly — method'ni o'zgartirib bo'lmaydi ===
function readonly(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  descriptor.writable = false;
  return descriptor;
}

class Config {
  @readonly
  getDbUrl(): string { return "postgresql://localhost/app"; }
}

const cfg = new Config();
// (cfg as any).getDbUrl = () => "hacked";
// TypeError — Cannot assign to read only property 'getDbUrl'


// === @retry(n) — xato bo'lganda qayta urinish ===
function retry(attempts: number) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      let lastError: unknown;
      for (let i = 0; i < attempts; i++) {
        try {
          return await original.apply(this, args);
        } catch (e) {
          lastError = e;
          if (i < attempts - 1) {
            console.warn(`Retry ${i + 1}/${attempts}: ${propertyKey}`);
          }
        }
      }
      throw lastError;
    };

    return descriptor;
  };
}

class HttpClient {
  @retry(3)
  async fetchUser(id: number) {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  }
}
```

### Edge Cases

- **Static method decorator:** `target` = constructor, instance method'da `target` = prototype. Bu instance va class metadata farqi
- **Descriptor return value:** `void` qaytarsa — original descriptor saqlanadi. Yangi descriptor qaytarsa — almashtiriladi. Reference'da o'zgartirish (`descriptor.value = ...`) ham ishlaydi
- **Arrow function `this` muammosi:** `descriptor.value = (...args) => original.apply(this, args)` — `this` decorator scope. To'g'risi: `function (this: any, ...args)`
- **`writable: false` + qayta `defineProperty`:** Configurable bo'lsa, `defineProperty` bilan qayta yozish mumkin. `configurable: false` esa to'liq lock
- **Async method:** decorator wrap funksiyasi `async` deb declare qilinmasligi mumkin — promise return value automatic await beradi

### Follow-up savollar

1. **"Descriptor return qilish kerakmi yoki to'g'ridan-to'g'ri mutate?"** — Ikkalasi ham ishlaydi. Return — explicit, mutate — implicit. Konvensiya: return
2. **"TC39 da `PropertyDescriptor` qaerda?"** — Yo'q. TC39 da method funksiyasining o'zi value sifatida keladi, descriptor abstraction yo'q. `writable`, `configurable` kerak bo'lsa `addInitializer` + `Object.defineProperty`

</details>

---

### Savol 10: Compiled output: `__decorate` vs `__esDecorate`? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`__decorate` — legacy decorator helper, har decorator'ni target'ga ketma-ket qo'llaydi. `__esDecorate` — TC39 helper, context object yaratadi va decorator return value'lar bilan class structure ni qayta quradi. `__runInitializers` — `addInitializer` callback'larni triggerlash uchun.

### To'liq tushuntirish

**Legacy compiled output:**

```typescript
// TypeScript source:
@sealed
class UserService {
  @log greet(name: string) { return `Hello, ${name}`; }
}
```

```javascript
// Compiled JS (legacy):
var UserService = /** @class */ (function () {
  function UserService() {}
  UserService.prototype.greet = function (name) { return "Hello, " + name; };
  return UserService;
}());

__decorate([log], UserService.prototype, "greet", null);
UserService = __decorate([sealed], UserService);
```

`__decorate` implementation:

```javascript
var __decorate = function (decorators, target, key, desc) {
  var c = arguments.length;
  var r = c < 3 ? target : desc === null ? Object.getOwnPropertyDescriptor(target, key) : desc;
  var d;
  if (typeof Reflect === "object" && typeof Reflect.decorate === "function") {
    r = Reflect.decorate(decorators, target, key, desc);
  } else {
    for (var i = decorators.length - 1; i >= 0; i--) {
      if ((d = decorators[i])) {
        r = (c < 3 ? d(r) : c > 3 ? d(target, key, r) : d(target, key)) || r;
      }
    }
  }
  return c > 3 && r && Object.defineProperty(target, key, r), r;
};
```

**TC39 compiled output:**

```typescript
// TypeScript source (TC39):
@sealed
class UserService {
  @log greet(name: string) { return `Hello, ${name}`; }
}
```

```javascript
// Compiled JS (TC39):
let UserService = (() => {
  let _classDecorators = [sealed];
  let _classDescriptor;
  let _classExtraInitializers = [];
  let _classThis;
  let _greet_decorators;
  let _instanceExtraInitializers = [];

  var UserService = class {
    static { _classThis = this; }
    static {
      _greet_decorators = [log];
      __esDecorate(this, null, _greet_decorators, {
        kind: "method",
        name: "greet",
        static: false,
        private: false,
        access: { has: obj => "greet" in obj, get: obj => obj.greet }
      }, null, _instanceExtraInitializers);
      __esDecorate(null, _classDescriptor = { value: _classThis },
        _classDecorators, { kind: "class", name: "UserService" },
        null, _classExtraInitializers);
      UserService = _classThis = _classDescriptor.value;
      __runInitializers(_classThis, _classExtraInitializers);
    }

    constructor() {
      __runInitializers(this, _instanceExtraInitializers);
    }

    greet(name) { return `Hello, ${name}`; }
  };

  return UserService = _classThis;
})();
```

**Asosiy farqlar:**

| | Legacy `__decorate` | TC39 `__esDecorate` |
|---|---|---|
| Argument | `decorators[], target, key, descriptor` | `target, descriptor, decorators, context, initializers, extraInitializers` |
| Context | Yo'q | Object: `kind`, `name`, `static`, `private`, `access`, `addInitializer`, `metadata` |
| Decorator return | Method/class replacement | Method/class replacement + initializer callbacks |
| Class wrap | IIFE da bir oddiy class | IIFE + static block + `_classThis` reference |
| Initializer | Yo'q | `__runInitializers` chaqiriladi `addInitializer` callback'lar uchun |

**`__runInitializers`:**

```javascript
var __runInitializers = function (thisArg, initializers, value) {
  var useValue = arguments.length > 2;
  for (var i = 0; i < initializers.length; i++) {
    value = useValue ? initializers[i].call(thisArg, value) : initializers[i].call(thisArg);
  }
  return useValue ? value : void 0;
};
```

Har `addInitializer(fn)` callback shu helper orqali chaqiriladi.

### Kod misol

```typescript
// === TypeScript source ===
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  context.addInitializer(function (this: any) {
    console.log(`init ${String(context.name)}`);
  });
  return function (this: any, ...args: any[]) {
    console.log(`call ${String(context.name)}`);
    return originalMethod.call(this, ...args);
  };
}

class Service {
  @log fetch() { return "data"; }
}

new Service().fetch();
// init fetch       <-- addInitializer ishlaydi har new da
// call fetch       <-- decorator return ishlaydi
// "data"
```

### Edge Cases

- **Bundle size:** TC39 helper'lar (`__esDecorate`, `__runInitializers`) — legacy `__decorate` dan kattaroq. Production bundle uchun TS `importHelpers: true` + `tslib` orqali helper'lar share qilinadi
- **`static { }` block:** TC39 da decorator setup static block ichida — class definition vaqtida bir marta ishlaydi
- **Recursive `_classThis`:** `static` decorator chaqiruvi paytida class hali fully constructed emas. `_classThis` reference orqali muhokama qilinadi
- **`Reflect.decorate` polyfill:** Legacy `__decorate` `Reflect.decorate` mavjud bo'lsa, undan foydalanadi (`reflect-metadata` polyfill bilan keladi)
- **Source map:** decorator-heavy class debug qilish qiyin — compiled output original kod dan farqli

### Follow-up savollar

1. **"Nima uchun TC39 da `static { }` block ishlatiladi?"** — Class body ichida one-time setup — class initialization order ni control qilish uchun
2. **"`__esDecorate` ni o'zi yozish mumkinmi?"** — Mumkin — TS-source da minimal version. Lekin spec to'liq implementation murakkab (private access, accessor handling, error path)

<details>
<summary><strong>Deep Dive</strong></summary>

**`__esDecorate` to'liq spec — argument'lar:**

```typescript
function __esDecorate(
  ctor: Function | null,           // Class constructor (member decorator uchun)
  descriptorIn: PropertyDescriptor | null,  // Class decorator uchun { value: class }
  decorators: Function[],           // Decorator array (top-to-bottom order)
  contextIn: {                      // Initial context shape
    kind: string;
    name: string | symbol;
    static?: boolean;
    private?: boolean;
    access?: { get?: Function; set?: Function; has?: Function };
    metadata?: object;
  },
  initializers: Function[] | null,  // Static initializers
  extraInitializers: Function[]     // Instance initializers
): void;
```

**Helper internal algorithm:**

```javascript
var __esDecorate = function (ctor, descriptorIn, decorators, contextIn, initializers, extraInitializers) {
  function accept(f) {
    if (f !== void 0 && typeof f !== "function") throw new TypeError("Function expected");
    return f;
  }
  var kind = contextIn.kind;
  var key = kind === "getter" ? "get" : kind === "setter" ? "set" : "value";
  var target = !descriptorIn && ctor ? (contextIn.static ? ctor : ctor.prototype) : null;
  var descriptor = descriptorIn || (target ? Object.getOwnPropertyDescriptor(target, contextIn.name) : {});
  var _;
  var done = false;

  // Decorator'larni pastdan yuqoriga apply
  for (var i = decorators.length - 1; i >= 0; i--) {
    var context = {};
    for (var p in contextIn) context[p] = p === "access" ? {} : contextIn[p];
    for (var p in contextIn.access) context.access[p] = contextIn.access[p];
    context.addInitializer = function (f) {
      if (done) throw new TypeError("Cannot add initializers after decoration has completed");
      extraInitializers.push(accept(f || null));
    };

    var result = (0, decorators[i])(
      kind === "accessor"
        ? { get: descriptor.get, set: descriptor.set }
        : descriptor[key],
      context
    );

    if (kind === "accessor") {
      if (result === void 0) continue;
      if (result === null || typeof result !== "object") throw new TypeError("Object expected");
      if (_ = accept(result.get)) descriptor.get = _;
      if (_ = accept(result.set)) descriptor.set = _;
      if (_ = accept(result.init)) initializers.unshift(_);
    } else if (_ = accept(result)) {
      if (kind === "field") initializers.unshift(_);
      else descriptor[key] = _;
    }
  }

  if (target) Object.defineProperty(target, contextIn.name, descriptor);
  done = true;
};
```

**Helper xususiyatlar:**

- **`done` flag** — barcha decorator'lar apply bo'lgandan keyin `addInitializer` bloklanadi. Spec invariant: initializer'lar faqat decoration vaqtida ro'yxatga olinishi mumkin.
- **`unshift` initializer'da** — pastdan yuqoriga apply ketma-ketligi tufayli initializer'lar **boshiga** qo'shiladi (final order top-to-bottom).
- **Accessor uchun maxsus** — `{ get, set, init }` object qaytarilsa, get/set descriptor'da, init initializer'da.
- **Field decorator** — return funksiya initializer sifatida ishlatiladi.

**`__runInitializers` to'liq:**

```javascript
var __runInitializers = function (thisArg, initializers, value) {
  var useValue = arguments.length > 2;
  for (var i = 0; i < initializers.length; i++) {
    value = useValue
      ? initializers[i].call(thisArg, value)
      : initializers[i].call(thisArg);
  }
  return useValue ? value : void 0;
};
```

`useValue` flag — field/accessor initializer chain (har biri oldingi qiymatni transform qiladi). Method decorator extraInitializer'lar uchun `value` argument yo'q.

**Bundle optimization — `importHelpers`:**

```json
// tsconfig.json
{
  "compilerOptions": {
    "importHelpers": true,
    "target": "es2022"
  }
}
```

`importHelpers: true` + `tslib` dependency'da — har fayl helper'larni `tslib` modulidan import qiladi (inline emas). Bundle size kichrayadi, lekin `tslib` runtime dependency.

```typescript
// Compile output (importHelpers: true)
import { __esDecorate as _esDecorate, __runInitializers as _runInitializers } from "tslib";
// ... decorator setup _esDecorate ni ishlatadi
```

</details>

</details>

---

## Output savollar

### Savol 11: Decorator composition order [Middle+]

**Savol:** Quyidagi kodning console output ini ayting:

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
a evaluated
b evaluated
b applied
a applied
c applied
```

### To'liq tushuntirish

Decorator composition ikki bosqichli jarayon:

1. **Evaluate phase (yuqoridan pastga)** — factory funksiyalar chaqiriladi. Bu inner decorator funksiyani qaytaradi. `a()` → `b()` ketma-ketligida.

2. **Apply phase (pastdan yuqoriga)** — qaytgan inner decorator'lar target'ga qo'llaniladi. Function composition tartibida: `a(b(method))` — eng yaqin decorator (`@b`) birinchi qo'llaniladi, natijani `@a` o'rab oladi.

3. **Class decorator eng oxirida** — barcha member (method, field, accessor) decorator'lar tugagandan keyin. Sabab: class decorator class'ni butunligicha (member'lar bilan birga) oladi va replace qila oladi.

**Member tartibi:** Class ichida member decorator'lar **declaration tartibida** ishlaydi (top-to-bottom class body bo'yicha).

### Kod misol

```typescript
// Mantiqiy composition:
// @a() @b() method  ==  method = a(b(method))

// Misol — b method natijasini *2, a esa +10 qiladi
function plus10(originalMethod: () => number, _: ClassMethodDecoratorContext) {
  return function (this: any) {
    return originalMethod.call(this) + 10;
  };
}

function times2(originalMethod: () => number, _: ClassMethodDecoratorContext) {
  return function (this: any) {
    return originalMethod.call(this) * 2;
  };
}

class Calc {
  @plus10   // tashqarida — natijaga +10
  @times2   // ichkarida — original ga *2
  value(): number { return 5; }
}

new Calc().value(); // (5 * 2) + 10 = 20
// times2 inner — birinchi qo'llaniladi
// plus10 outer — natijani o'rab oladi
```

### Edge Cases

- **Factory side effect:** `console.log` factory ichida class definition vaqtida ishlaydi — har class load bo'lganda
- **Asinxron decorator:** decorator funksiya `async` bo'lishi mumkin emas — class definition synchronous
- **Same decorator ikki marta:** `@log @log method()` — ikki bor wrap, har chaqiruvda 2 log
- **Factory chaqiruvini unutish:** `@a` vs `@a()` — birinchisi factory ni decorator deb interpret qiladi (xato), ikkinchisi factory ni evaluate qilib decorator oladi

### Follow-up savollar

1. **"Class decorator nima uchun oxirida?"** — Class decorator class'ni butunlay almashtirishi mumkin (constructor extend). Avval member structure tayyor bo'lishi kerak
2. **"Parameter decorator (legacy) tartibi qaerda?"** — Method decorator'dan oldin, har parametr uchun

</details>

---

### Savol 12: Field + method + class order [Middle+]

**Savol:** Output ni ayting:

```typescript
function field(value: undefined, context: ClassFieldDecoratorContext) {
  console.log(`field decorator: ${String(context.name)}`);
  return function (initialValue: number) {
    console.log(`init: ${String(context.name)} = ${initialValue}`);
    return initialValue * 2;
  };
}

function method(value: (...args: any[]) => any, context: ClassMethodDecoratorContext) {
  console.log(`method decorator: ${String(context.name)}`);
}

function cls(value: new (...args: any[]) => any, context: ClassDecoratorContext) {
  console.log(`class decorator: ${String(context.name)}`);
}

@cls
class Example {
  @field x = 10;
  @method greet() {}
  @field y = 20;
}

console.log("--- new Example() ---");
const e = new Example();
console.log("x =", (e as any).x, "y =", (e as any).y);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
field decorator: x
method decorator: greet
field decorator: y
class decorator: Example
--- new Example() ---
init: x = 10
init: y = 20
x = 20 y = 40
```

### To'liq tushuntirish

**Class definition phase (instance yaratishdan oldin):**

1. Member decorator'lar **declaration tartibida** chaqiriladi — body order: `x` → `greet` → `y`
2. Class decorator **eng oxirida** — barcha member decorator'lar tugagach
3. Field decorator faqat metadata yozadi yoki initializer qaytaradi — qiymat **hali transform qilinmaydi**

**Instance creation phase (`new Example()`):**

1. Field initializer'lar har instance'da chaqiriladi
2. Field decorator qaytargan funksiya (`initializer`) `initialValue` ni argument qilib oladi
3. Return qilingan qiymat instance field'ga yoziladi

**Field decorator timing:** decorator value sifatida `undefined` oladi (field hali set bo'lmagan). Lekin instance constructor'da har field uchun initializer (decorator return) chaqiriladi va `initialValue * 2` natijasi field'ga yoziladi.

### Kod misol

```typescript
// === Praktik misol — @range field decorator ===
function range(min: number, max: number) {
  return function (value: undefined, context: ClassFieldDecoratorContext) {
    return function (initialValue: number) {
      return Math.min(max, Math.max(min, initialValue));
    };
  };
}

class Player {
  @range(0, 100)
  health: number = 150;  // 100 ga cheklanadi

  @range(0, 100)
  mana: number = -10;    // 0 ga cheklanadi
}

const p = new Player();
console.log(p.health, p.mana); // 100 0
```

### Edge Cases

- **Field initial value:** decorator chaqirilganda hali set bo'lmagan — `value: undefined`. Initializer return da `initialValue` keladi
- **Static field:** decorator class definition vaqtida ishlaydi (`new` kerak emas)
- **Field decorator return optional:** `void` qaytarsa — original initialValue saqlanadi
- **Initializer `this` context:** instance — `function (this: any, initialValue) { return ... }` orqali olish mumkin

### Follow-up savollar

1. **"`@field x` va `@field y` initializer alohida-alohida ishlaydimi?"** — Ha, har field uchun har instance'da
2. **"Order'ni o'zgartirish mumkinmi?"** — Yo'q. Spec'da declaration order qat'iy

</details>

---

### Savol 13: Factory + decorator execution timing [Middle+]

**Savol:** Output ni ayting:

```typescript
function dec(label: string) {
  console.log(`factory: ${label}`);
  return function (v: any, ctx: any) {
    console.log(`apply: ${label}`);
  };
}

class Service {
  @dec("A")
  @dec("B")
  method() {
    console.log("method called");
  }
}

console.log("--- runtime ---");
new Service().method();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Output:

```
factory: A
factory: B
apply: B
apply: A
--- runtime ---
method called
```

### To'liq tushuntirish

**Class definition vaqtida (`new` chaqirishdan oldin):**

1. **Factory phase yuqoridan pastga** — `dec("A")` evaluate, keyin `dec("B")` evaluate. Har factory `console.log("factory: ...")` qiladi.

2. **Apply phase pastdan yuqoriga** — qaytgan inner decorator'lar applied. `B` apply (yaqinroq), keyin `A` apply (tashqaridan o'rab).

**Method body class definition vaqtida ishlamaydi** — faqat `new Service().method()` chaqirilganda. Bu sabab `--- runtime ---` dan keyin `method called`.

**Key insight:** Decorator faqat **class definition time** da ishlaydi. Har `new` da decorator qayta chaqirilmaydi (lekin `addInitializer` callback har instance'da chaqiriladi).

### Kod misol

```typescript
// === Class va decorator timing ===
console.log("1. before class");

function timer(label: string) {
  console.log(`2. factory ${label}`);
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    console.log(`3. apply ${label}`);
    context.addInitializer(function () {
      console.log(`4. initializer ${label}`);
    });
    return originalMethod;
  };
}

class App {
  @timer("X")
  run() {}
}

console.log("5. after class");

console.log("6. before new");
new App();
console.log("7. after new");

new App();

// Output:
// 1. before class
// 2. factory X         <- class definition
// 3. apply X
// 5. after class
// 6. before new
// 4. initializer X     <- har new() da
// 7. after new
// 4. initializer X     <- ikkinchi new() da yana
```

### Edge Cases

- **Factory faqat bir marta** — class definition vaqtida. Ko'p instance bo'lsa ham factory takrorlanmaydi
- **`addInitializer` har instance** — har `new` da chaqiriladi
- **Module load order:** import qilish ham class definition triggerlaydi (top-level class'lar)
- **Class expression:** `const C = class { @log method() {} }` — definition vaqti har `new C()` da emas, **`class` expression evaluate bo'lganda**

### Follow-up savollar

1. **"Lazy initialization mumkinmi?"** — Decorator apply lazy emas, lekin `addInitializer` har instance'da. Side effect lazy bo'lishi mumkin
2. **"Tree-shaking ga ta'siri?"** — Decorator side effect beradi → tree-shaking qiyinroq. Bundler decorator chaqiruvini side-effect-free deb assume qila olmaydi

</details>

---

## Coding savollar

### Savol 14: `@memoize` decorator yozing [Middle+]

**Savol:** TC39 standartida method natijasini argument'lariga ko'ra cache qiladigan `@memoize` decorator yozing:

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
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`@memoize` — method around `Map` cache wrap. Argument'lardan `JSON.stringify` orqali key yaratiladi. Birinchi chaqiruvda hisoblab cache qiladi, keyingilarida cache dan oladi.

### To'liq tushuntirish

**Asosiy g'oya:** method natijasini argument'lar bo'yicha cache qilish. Sof funksiya (pure function) uchun ishlaydi — bir xil argument'lar bir xil natija beradi.

**Cache scope tanlash:**
- **Class-level cache** (decorator scope) — barcha instance'lar bir cache ni share qiladi. Eng oddiy, lekin instance state bog'liq method uchun noto'g'ri natija
- **Instance-level cache** (`WeakMap<instance, Map>`) — har instance o'z cache'i. To'g'ri, lekin biroz murakkab

**Key strategy:**
- `JSON.stringify(args)` — sodda, primitive argument'lar uchun ishlaydi
- Object argument'lar uchun unstable (key tartibi farq qilishi mumkin) — alternativa: `WeakMap` argument identity asosida
- Symbol, function argument'lar — JSON da ko'rinmaydi

### Kod misol

```typescript
// === Class-level cache (sodda) ===
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

class MathService {
  callCount = 0;

  @memoize
  fibonacci(n: number): number {
    this.callCount++;
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}

const math = new MathService();
math.fibonacci(10); // 55, callCount = 11 (har n bir marta hisoblanadi)
math.fibonacci(10); // 55, callCount hali 11 (cache dan)


// === Instance-level cache (to'g'riroq) ===
function memoizePerInstance(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  const caches = new WeakMap<object, Map<string, any>>();

  return function (this: any, ...args: any[]) {
    let cache = caches.get(this);
    if (!cache) {
      cache = new Map();
      caches.set(this, cache);
    }
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);

    const result = originalMethod.call(this, ...args);
    cache.set(key, result);
    return result;
  };
}
```

### Edge Cases

- **Async method:** `Promise` cache qilinadi. Reject bo'lgan promise cache da qoladi → keyingi chaqiruv ham reject. Yechim: catch da cache.delete()
- **State-dependent method:** `this.balance` ga bog'liq method memoize — eski natija qaytadi. Memoize sof funksiya uchun
- **Memory leak:** class-level cache — instance garbage collect bo'lsa ham cache da qoladi. `WeakMap` (instance key) bilan auto-cleanup
- **Object argument:** `{a:1, b:2}` va `{b:2, a:1}` — JSON.stringify har xil natija beradi (key order). Yechim: deterministic stringify yoki canonical hash
- **TTL kerak bo'lsa:** `{ value, expires }` saqlash kerak. `expires < Date.now()` bo'lsa cache invalidate

### Follow-up savollar

1. **"Async method uchun nima qilish kerak?"** — Promise cache qiling, lekin reject da cache.delete(). Ikkala ham hisoblash takrorlanmasligi uchun loading state saqlanadi
2. **"Cache size cheksiz — muammomi?"** — Ha, LRU cache (max size + eviction) ishlatish kerak production da

<details>
<summary><strong>Deep Dive</strong></summary>

**TTL + LRU memoize:**

```typescript
interface CacheEntry<T> { value: T; expires: number; }

function memoizeAdvanced(options: { ttl?: number; maxSize?: number } = {}) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    const cache = new Map<string, CacheEntry<any>>();
    const { ttl = Infinity, maxSize = Infinity } = options;

    return function (this: any, ...args: any[]) {
      const key = JSON.stringify(args);
      const entry = cache.get(key);

      if (entry && entry.expires > Date.now()) {
        // LRU — refresh order
        cache.delete(key);
        cache.set(key, entry);
        return entry.value;
      }

      const value = originalMethod.call(this, ...args);
      cache.set(key, { value, expires: Date.now() + ttl });

      // LRU eviction
      if (cache.size > maxSize) {
        const firstKey = cache.keys().next().value;
        cache.delete(firstKey);
      }

      return value;
    };
  };
}

class ApiClient {
  @memoizeAdvanced({ ttl: 60_000, maxSize: 100 })
  fetchUser(id: number) { /* ... */ }
}
```

**Spec-level note:** `Map` insertion order ni saqlaydi — LRU implementation uchun ideal. `Map.prototype.delete` + `set` — element'ni oxiriga ko'chiradi.

</details>

</details>

---

### Savol 15: `@singleton` class decorator yozing [Middle+]

**Savol:** TC39 standartida — faqat bitta instance yaratiladigan `@singleton` class decorator:

```typescript
@singleton
class DatabaseConnection {
  constructor(public url: string) { console.log(`Connected to ${url}`); }
}
// new DatabaseConnection("postgres://a") → "Connected to postgres://a"
// new DatabaseConnection("postgres://b") → console.log YO'Q
// db1 === db2 → true
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Class ni `extends` qiladigan yangi class qaytarish. Birinchi `new` da super constructor ishlaydi va instance saqlanadi. Keyingi `new` larda saqlangan instance qaytariladi — JS spec'da constructor `return object` qilsa, `new` shu object'ni qaytaradi.

### To'liq tushuntirish

**Mexanizm:** JavaScript spec'da `new SomeClass()` — agar constructor object qaytarsa, yangi yaratilgan `this` o'rniga shu object qaytariladi. Bu xususiyatdan foydalanib singleton implement qilinadi.

**Step by step:**
1. Decorator class'ni argument sifatida oladi
2. Yangi class qaytaradi — original class'ni extends qiladi
3. Yangi class constructor:
   - Agar instance saqlangan bo'lsa — `return instance` (yangi `this` ignored)
   - Aks holda — `super(...args)` orqali parent constructor ishlaydi, natija saqlanadi

**TypeScript generic constraint:** `<T extends new (...args: any[]) => any>` — har qanday constructor'ni qabul qilish uchun.

### Kod misol

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

@singleton
class ConfigService {
  constructor(public env: string) {
    console.log(`Config initialized: ${env}`);
  }
}

const c1 = new ConfigService("production"); // "Config initialized: production"
const c2 = new ConfigService("development"); // console.log YO'Q
console.log(c1 === c2);   // true
console.log(c2.env);       // "production" (birinchi instance'ning env'i)
```

### Edge Cases

- **Argument farqi:** ikkinchi `new` ning argument'lari **ignored**. `c2.env === "production"` — birinchi argument
- **Inheritance:** `class Sub extends ConfigService {}` — Sub o'zining singleton emas. Lekin Sub instance yaratish murakkabroq (parent singleton constructor return qiladi)
- **Static method bilan singleton (alternativa):** `Class.getInstance()` pattern — decorator'ga muhtoj emas. Lekin `new` syntax beradi singleton decorator
- **Type info:** `as T` cast — qaytarilgan subclass'ni TypeScript original class type'i sifatida ko'radi. Compile-time'da type bir xil qoladi, faqat runtime'da subclass
- **Module-level singleton (afzal):** `export const config = new Config(...)` — har import bir xil. Decorator singleton dan oddiy
- **Reset uchun nima qilish?:** decorator inner `instance` ga tashqaridan access yo'q. Test da reset kerak bo'lsa — DI yaxshiroq

### Follow-up savollar

1. **"Singleton anti-pattern deyilishining sababi?"** — Global state, test isolation muammosi, dependency injection bilan ziddiyat. Module-level instance + DI yaxshiroq
2. **"`InstanceType<T>` nima?"** — Utility type: constructor type'dan instance type'ni oladi. `class Foo {}` → `InstanceType<typeof Foo> === Foo`

</details>

---

### Savol 16: `@retry` decorator factory yozing [Middle+]

**Savol:** TC39 standartida async method'ni xato bo'lganda qayta urinadigan `@retry(n)` decorator factory:

```typescript
class ApiClient {
  @retry(3)
  async fetchUser(id: number): Promise<{ id: number; name: string }> {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Factory `attempts` parametr oladi, inner decorator async method'ni wrap qiladi. `try/catch` orqali xatoni tutib, `attempts - 1` martagacha qayta urinadi. Oxirgi urinishda ham xato bo'lsa — throw qiladi.

### To'liq tushuntirish

**Asosiy elementlar:**
1. **Factory funksiya** — `attempts: number` qabul qiladi
2. **Inner decorator** — method'ni wrap qiladi
3. **Loop** — `attempts` marta urinish
4. **Catch + last error** — har retry da xato saqlanadi, oxirgi urinishda throw

**Production'da hisobga olinadigan jihatlar:**
- **Backoff strategy:** har retry orasida `delay` (linear, exponential). Sabab — server'ga bir vaqtning o'zida ko'p so'rov yubormaslik
- **Retryable errors:** har xato emas retry qilinadi (404 retry ma'nosiz, 500 retry mantiqli). `shouldRetry: (e) => boolean` callback
- **Jitter:** delay ga random offset — thundering herd oldini olish
- **Total timeout:** retry yig'indisi belgilangan vaqtdan oshmasin

### Kod misol

```typescript
// === Sodda @retry ===
function retry(attempts: number) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return async function (this: any, ...args: any[]) {
      let lastError: unknown;
      for (let i = 0; i < attempts; i++) {
        try {
          return await originalMethod.call(this, ...args);
        } catch (e) {
          lastError = e;
          if (i < attempts - 1) {
            console.warn(`Retry ${i + 1}/${attempts} for ${String(context.name)}`);
          }
        }
      }
      throw lastError;
    };
  };
}

class PaymentApi {
  @retry(3)
  async processPayment(amount: number): Promise<{ id: string }> {
    if (Math.random() < 0.7) throw new Error("Network error");
    return { id: "pay-" + Math.random() };
  }
}


// === Production-grade @retry — exponential backoff + selective retry ===
interface RetryOptions {
  attempts: number;
  baseDelay?: number;
  maxDelay?: number;
  shouldRetry?: (error: unknown) => boolean;
}

function retryAdvanced(options: RetryOptions) {
  const {
    attempts,
    baseDelay = 100,
    maxDelay = 10_000,
    shouldRetry = () => true,
  } = options;

  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return async function (this: any, ...args: any[]) {
      let lastError: unknown;
      for (let i = 0; i < attempts; i++) {
        try {
          return await originalMethod.call(this, ...args);
        } catch (e) {
          lastError = e;
          if (i === attempts - 1 || !shouldRetry(e)) throw e;

          const delay = Math.min(baseDelay * 2 ** i, maxDelay);
          const jitter = Math.random() * delay * 0.3;
          await new Promise(r => setTimeout(r, delay + jitter));
        }
      }
      throw lastError;
    };
  };
}

class ResilientApi {
  @retryAdvanced({
    attempts: 5,
    baseDelay: 200,
    shouldRetry: (e) => {
      if (e instanceof TypeError) return true;  // Network error
      if (e instanceof Error && e.message.includes("5")) return true;  // 5xx
      return false;
    },
  })
  async fetchOrder(id: string) { /* ... */ }
}
```

### Edge Cases

- **Sync method'da:** `for` loop sync ishlaydi, `await` no-op. Lekin natija qaytarilmaydi to'g'ri, retry semantikasi sync uchun foydasiz
- **`this` arrow function bilan:** arrow function decorator scope `this` ni oladi — class instance emas. `function (this: any, ...)` ishlatish
- **Cancellation:** uzoq retry chain ni to'xtatish kerak bo'lsa — `AbortSignal` parametr sifatida
- **Side effect retry:** `POST /charge` 5 marta retry — to'lov 5 marta o'tishi mumkin. Idempotency key kerak
- **Error swallow:** intermediate xatolar log qilinmasa, debugging qiyin

### Follow-up savollar

1. **"Exponential backoff sababi nima?"** — Linear delay server ga uzluksiz yuk beradi. Exponential — har retry da 2x kutish, server ga vaqt beradi recover qilish
2. **"`retry` va `circuit breaker` farqi?"** — Retry — individual chaqiruv darajasida. Circuit breaker — service darajasida, ko'p xatoga uchragan service'ga chaqiruvni vaqtincha to'xtatadi

</details>

---

### Savol 17: `@authorize` decorator factory yozing [Middle+]

**Savol:** TC39 standartida method chaqirilishidan oldin foydalanuvchi role ni tekshiradigan `@authorize(role)`:

```typescript
class AdminPanel {
  currentUserRole: string = "user";

  @authorize("admin")
  deleteUser(id: number) { return `User ${id} deleted`; }

  @authorize("moderator")
  banUser(id: number) { return `User ${id} banned`; }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Factory `role` qabul qiladi. Inner decorator method'ni wrap qiladi — chaqiruvdan oldin `this.currentUserRole` ni tekshiradi. Role mos kelmasa — `throw new Error("Unauthorized")`.

### To'liq tushuntirish

**Asosiy pattern:**
1. Factory roles'ni capture qiladi (closure)
2. Inner decorator method'ni replace qiladi
3. Wrapped funksiya `this` orqali current user state'ga access
4. Authorization check, fail bo'lsa throw

**Real-world enhancement:**
- **Multiple roles:** `@authorize("admin", "owner")` — har qaysi role yetarli
- **Permission hierarchy:** admin > moderator > user — yuqori role pastni ham qoplaydi
- **Async authorization:** role ni tashqi service'dan olish kerak bo'lsa, decorator async
- **Audit logging:** har unauthorized urinish log qilinadi

### Kod misol

```typescript
// === Sodda @authorize ===
function authorize(...roles: string[]) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return function (this: any, ...args: any[]) {
      const userRole = this.currentUserRole;
      if (!roles.includes(userRole)) {
        throw new Error(
          `Unauthorized: ${String(context.name)} requires [${roles.join(", ")}], got "${userRole}"`
        );
      }
      return originalMethod.call(this, ...args);
    };
  };
}

class AdminPanel {
  currentUserRole = "user";

  @authorize("admin")
  deleteUser(id: number) { return `User ${id} deleted`; }

  @authorize("admin", "moderator")
  banUser(id: number) { return `User ${id} banned`; }
}

const panel = new AdminPanel();
panel.currentUserRole = "moderator";
panel.banUser(1);        // "User 1 banned"
// panel.deleteUser(1);  // ❌ Error: Unauthorized


// === Permission hierarchy + audit log ===
const ROLE_HIERARCHY: Record<string, number> = {
  guest: 0,
  user: 1,
  moderator: 2,
  admin: 3,
  superadmin: 4,
};

interface AuditLog {
  timestamp: Date;
  method: string;
  role: string;
  allowed: boolean;
}

const auditLog: AuditLog[] = [];

function requireRole(minRole: string) {
  return function (
    originalMethod: (...args: any[]) => any,
    context: ClassMethodDecoratorContext
  ) {
    return function (this: any, ...args: any[]) {
      const userLevel = ROLE_HIERARCHY[this.currentUserRole] ?? -1;
      const requiredLevel = ROLE_HIERARCHY[minRole] ?? 999;
      const allowed = userLevel >= requiredLevel;

      auditLog.push({
        timestamp: new Date(),
        method: String(context.name),
        role: this.currentUserRole,
        allowed,
      });

      if (!allowed) {
        throw new Error(
          `Forbidden: ${String(context.name)} requires ${minRole}+, got ${this.currentUserRole}`
        );
      }
      return originalMethod.call(this, ...args);
    };
  };
}

class UserManagement {
  currentUserRole = "moderator";

  @requireRole("user")
  viewUsers() { return ["user1", "user2"]; }       // ✅

  @requireRole("moderator")
  banUser(id: number) { return `Banned ${id}`; }    // ✅

  @requireRole("admin")
  deleteAccount(id: number) { return `Deleted ${id}`; } // ❌
}
```

### Edge Cases

- **`this.currentUserRole` undefined:** decorator chaqirilganda `this` instance — agar property yo'q bo'lsa `undefined`, mos role bilan tekshirish noto'g'ri. Initial value berilishi kerak
- **Static method bilan:** `this` constructor. Decorator instance state'ga emas, class-level config'ga ishonadi
- **Async method:** wrapped funksiya `async` bo'lishi kerak — `return await originalMethod.call(this, ...args)`. Yoki promise tabiiy chain
- **Throw vs return error:** throw — exception flow. Alternative: `Result<T, AuthError>` qaytarish (functional style)
- **JWT-based:** `this.currentUserRole` o'rniga JWT token decode — async, side effect heavy

### Follow-up savollar

1. **"Role decorator-da hardcode bo'lsa, runtime config qanday?"** — Decorator factory parametri runtime da o'zgartirib bo'lmaydi (class definition vaqtida fix). Yechim: roles'ni context yoki external config'dan o'qish
2. **"Frontend va backend authorization farqi?"** — Frontend — UX (button hide), backend — security (har request'da check). Decorator faqat client tomon mexanizm, asosiy authorization backend da

</details>

---

## Bug fix savollar

### Savol 18: Arrow function `this` xato — toping va tuzating [Middle+]

**Savol:** Bu decorator'da xato bor. Toping va tuzating:

```typescript
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
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
<summary><strong>Javob</strong></summary>

### Qisqa javob

Arrow function o'z `this` ga ega emas — lexical scope dan oladi. Decorator funksiyasi class instance emas, decorator scope da chaqirilgan → `this` notog'ri. Yechim: `function (this: any, ...args)` ishlatish.

### To'liq tushuntirish

**Muammo:** Arrow function — leksik `this` ni capture qiladi (yaratilgan scope dan). `log` decorator chaqirilganda `this` modul scope (yoki strict mode da `undefined`). Wrapped funksiya `originalMethod.call(this, ...args)` da `this` instance emas.

**Method chaqiruvi:** `instance.method()` — JS engine `method` ni `instance` ga bind qiladi (dynamic `this`). Lekin arrow function — bu binding ni qabul qilmaydi. Engine arrow ga `this` bera olmaydi.

**Yechim:** **Regular function** ishlatish. Regular function dynamic `this` ga ega — chaqiruv vaqtida `this` bind bo'ladi.

### Kod misol

```typescript
// ❌ XATO: Arrow function — this lexical
function logBroken(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  return (...args: any[]) => {
    // this = decorator scope (undefined yoki module)
    console.log(`${String(context.name)} called`);
    return originalMethod.call(this, ...args); // ❌ this noto'g'ri
  };
}

// ✅ TO'G'RI: Regular function — this dinamik
function log(
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: any[]) {
    // this = chaqiruv paytidagi instance
    console.log(`${String(context.name)} called`);
    return originalMethod.call(this, ...args); // ✅ this = instance
  };
}

class UserService {
  name = "UserService";

  @log
  getUsers() { return `${this.name}: fetching`; }
}

console.log(new UserService().getUsers());
// "getUsers called"
// "UserService: fetching"
```

### Edge Cases

- **TypeScript `this: any` parametri:** function signature'ga `this` parametr type qo'shish kerak — TS unga aniq type beradi. Aks holda noaniq `any`
- **Arrow ichida arrow:** nested arrow ham lexical chain. Outer regular function `this` bind bo'lsa, ichidagi arrow uni ko'ra oladi
- **`call`/`apply`/`bind` arrow uchun:** ishlamaydi. Arrow function `this` o'zgarmas
- **Class arrow method:** `name = () => this.something` — yaratilganda lexical scope. Bu instance constructor'da bind bo'ladi. Class body da method'da — boshqacha story
- **Test:** `new Class().method.call(otherInstance, ...)` arrow bo'lsa — yangi `this` ishlamaydi

### Follow-up savollar

1. **"Arrow va regular function'da `arguments` ham farq qiladimi?"** — Ha, arrow `arguments` ham yo'q. Regular `arguments` array-like
2. **"Class method arrow sifatida yozish (auto-bind) — yaxshimi?"** — Trade-off: bind muammo yo'q, lekin har instance'da yangi function (memory cost), super orqali chaqirib bo'lmaydi

</details>

---

### Savol 19: Field decorator getter/setter olishga urinish [Middle+]

**Savol:** Bu kodda dasturchi field qiymatini set bo'lganda log qilmoqchi. Lekin ishlamayapti. Toping va tuzating:

```typescript
function logged(
  value: any,
  context: ClassFieldDecoratorContext
) {
  return function (initialValue: any) {
    return new Proxy({ value: initialValue }, {
      get(target) { console.log(`read ${String(context.name)}`); return target.value; },
      set(target, _, v) { console.log(`write ${String(context.name)}`); target.value = v; return true; },
    });
  };
}

class CartState {
  @logged itemCount: number = 0;
}

const cart = new CartState();
cart.itemCount = 5;        // Console da nima ko'rinadi?
console.log(cart.itemCount);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Field decorator faqat initial value'ni transform qila oladi — getter/setter intercept qila olmaydi. Wrapping `Proxy` initial value'ni almashtirsa ham, keyingi `cart.itemCount = 5` — bevosita field'ga assign — proxy bypass qiladi. Yechim: `accessor` keyword ishlatish va accessor decorator yozish.

### To'liq tushuntirish

**Muammo tahlili:**

1. **Field decorator API:** `value: undefined` (field hali yaratilmagan), return funksiya `(initialValue) => transformedValue`
2. **Proxy initial value sifatida** — `new Proxy(...)` field'ga assign bo'ladi
3. **`cart.itemCount = 5`** — field'ni butunlay almashtiradi, proxy yo'qoladi
4. **Read** — endi field oddiy number (5), proxy emas

**Field decorator nima qila oladi:**
- Initial value transform (`return initialValue * 2`)
- Metadata yozish (`context.metadata[KEY] = ...`)
- `addInitializer` orqali side effect

**Field decorator nima qila olmaydi:**
- Getter/setter intercept (read/write track)
- Field assignment override
- Private storage yaratish

**Yechim:** TC39 da `accessor` keyword + accessor decorator. Compiler avtomatik private storage + getter/setter pair yaratadi, decorator esa get/set funksiyalarni intercept qiladi.

### Kod misol

```typescript
// ❌ XATO YONDASHUV: Field decorator + Proxy
// (initial proxy yo'qoladi keyingi assignment da)

// ✅ TO'G'RI YONDASHUV: accessor + accessor decorator
function logged(
  value: { get: () => any; set: (v: any) => void },
  context: ClassAccessorDecoratorContext
) {
  return {
    get() {
      console.log(`read ${String(context.name)}`);
      return value.get.call(this);
    },
    set(this: any, newValue: any) {
      console.log(`write ${String(context.name)} = ${newValue}`);
      value.set.call(this, newValue);
    },
    init(initialValue: any) {
      console.log(`init ${String(context.name)} = ${initialValue}`);
      return initialValue;
    },
  };
}

class CartState {
  @logged accessor itemCount: number = 0;
}

const cart = new CartState();
// init itemCount = 0

cart.itemCount = 5;
// write itemCount = 5

console.log(cart.itemCount);
// read itemCount
// 5
```

### Edge Cases

- **`accessor` keyword TS 4.9+ kerak** — eski versiyada syntax error
- **Private accessor:** `@logged accessor #count = 0` — getter/setter pair ham private, backing storage TS tomonidan `#count_accessor_storage` ko'rinishida nomlanadi
- **`init` optional** — qaytarmasa initial value saqlanadi
- **Eski yondashuv (manual getter/setter):** `accessor` siz manual `#name + get/set` yozish mumkin, lekin decorator API harakat qilmaydi (oddiy method/accessor decorator alohida-alohida bo'ladi)
- **Reactivity framework:** MobX `@observable`, SolidJS — accessor decorator orqali implement qilingan

### Follow-up savollar

1. **"Field decorator nima uchun get/set bermaydi?"** — Spec qaror: field — `Object.defineProperty` orqali class init time da set bo'ladi. Decorator faqat initializer transform qila oladi
2. **"Reactivity uchun ideal decorator qaysi?"** — Accessor — read/write track qiladi. Field — faqat initial value
3. **"Legacy da bu muammo bo'lardimi?"** — Legacy property decorator `target` (prototype) va `propertyKey` ni oladi, descriptor yo'q. Property decorator faqat metadata uchun

</details>

---

## Xulosa

- **Decorator** — runtime da ishlovchi funksiya, type erasure ga uchramaydi. Class va class a'zolariga qo'llaniladi
- **Legacy vs TC39** — bir-biriga mos kelmaydigan ikki standart. TC39 — ECMAScript Stage 3, TS 5.0+ da default
- **TC39 API** — `context` object: `kind`, `name`, `static`, `private`, `access`, `addInitializer`, `metadata`
- **Decorator turlari** — class, method, getter, setter, field, accessor. Legacy da parameter
- **`accessor` keyword** — auto getter/setter, decorator bilan read/write intercept
- **`Symbol.metadata`** — native metadata (TC39), class-scoped, prototype chain inheritance
- **`reflect-metadata`** — legacy polyfill, global WeakMap
- **`addInitializer`** — class/instance lifecycle hook
- **Parameter decorator TC39 da yo'q** — NestJS/Angular legacy da qolish sababi
- **Composition** — factory yuqoridan pastga evaluate, decorator pastdan yuqoriga apply
- **Compiled output** — legacy `__decorate`, TC39 `__esDecorate` + `__runInitializers`
- **Arrow function decorator'da TAQIQ** — `this` yo'qoladi, regular function ishlatish
- **Field decorator faqat initializer transform** — getter/setter kerak bo'lsa `accessor` keyword