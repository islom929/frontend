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

Decorator — class yoki class a'zolariga **qo'shimcha behavior qo'shadigan funksiya**. Decorator `@expression` syntax bilan class declaration oldidan yoziladi. Decorator'lar **meta-programming** ning bir turi — kod o'zi haqida ma'lumot oladi va o'z behavior'ini o'zgartiradi.

Decorator'lar quyidagi muammolarni yechadi:

1. **Cross-cutting concerns** — logging, validation, caching, authorization kabi logic'ni decorator orqali tashqaridan qo'shish
2. **Separation of concerns** — business logic va infrastructure logic'ni ajratish
3. **Code reuse** — bir xil behavior'ni ko'p joylarda qayta ishlatish
4. **Declarative programming** — imperative emas, declarative tarzda ifodalash

Decorator'lar **runtime'da ishlaydigan oddiy funksiyalar**. Ular type erasure'ga uchramaydi — JS'ga compile bo'lganda qoladi va haqiqiy runtime ta'sir ko'rsatadi. Bu interface va type alias'lardan tubdan farq qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Decorator'lar **runtime construct** — TypeScript'ning type erasure qoidasidan istisno. `tsc` emitter decorator'larni JavaScript'ga compile qiladi va target'ga `__decorate` (legacy) yoki `__esDecorate` (TC39) helper'lar orqali qo'llaydi.

ECMAScript decorator proposal taraqqiyot bosqichlari:
- **Stage 0** (2014, aprel) — Yehuda Katz va Ron Buckton tomonidan dastlabki proposal
- **Stage 1** (2015, mart) — proposal Stage 1 ga ko'tarildi
- **"Static decorators"** (2018) — alohida dizayn varianti, keyinroq tashlab yuborilgan
- **Stage 3** (2022, mart) — Daniel Ehrenberg va Kristen Hewell Garrett championligida qayta dizayn, hozirgi `context` API

TS evolution:
- **TS 1.5** (2015) — eksperimental decorator support (`experimentalDecorators`), eski proposal draft asosida
- **TS 5.0** (2023, mart) — TC39 Stage 3 decorator'lar nativ qo'llab-quvvatlanadi, `experimentalDecorators` flag'siz

Runtime qo'llab-quvvatlash: Stage 3 ga yetganidan keyin ham decorator'lar V8'da nativ implement qilinmagan (V8 issue 12763 hali ochiq) — Chrome, Node.js'da kod transpilation orqali ishlaydi. Deno 1.40 (2024, yanvar) `.ts`/`.tsx` fayllarda decorator'larni transpilation orqali qo'llaydi, nativ JS engine darajasida emas.

Decorator funksiya call signature decorator turiga qarab farqlanadi (class, method, field, accessor) va ikki standart (legacy vs TC39) bir-biridan parameter ro'yxati bilan ajraladi — bu farqlar keyingi section'larda batafsil ochiladi.

</details>

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
// BankAccount.prototype.newMethod = ... // ❌ Runtime'da xato
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

Legacy class decorator — class constructor'ni argument sifatida oladi. Agar yangi constructor qaytarsa — class constructor almashtiriladi.

**Signature:** `(constructor: new (...args: any[]) => any) => void | typeof constructor`

`experimentalDecorators: true` kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

Legacy decorator'lar `tsc` emitter tomonidan `__decorate` helper funksiyasiga compile qilinadi. Bu helper TypeScript runtime'ining ichki qismi — har `.ts` faylga avtomatik inject qilinadi yoki `importHelpers: true` + `tslib` orqali umumiy bo'ladi.

```javascript
// __decorate helper (soddalashtirilgan)
var __decorate = function (decorators, target, key, desc) {
  var c = arguments.length;
  var r = c < 3 ? target : desc === null ? desc = Object.getOwnPropertyDescriptor(target, key) : desc;
  for (var i = decorators.length - 1; i >= 0; i--) {
    if (d = decorators[i]) r = (c < 3 ? d(r) : c > 3 ? d(target, key, r) : d(target, key)) || r;
  }
  return c > 3 && r && Object.defineProperty(target, key, r), r;
};
```

Class decorator chaqirilganda `__decorate([decoratorFn], TargetClass)` formatda — `c < 3` shart `true`, `target` (class constructor) decorator'ga uzatiladi, decorator return qiymati (yangi class yoki `undefined`) `target`'ni almashtiradi.

`Object.getOwnPropertyDescriptor` chaqiruvi class decorator'da sodir bo'lmaydi (faqat method decorator'da `c > 3` da). Class decorator return qiymati `void` bo'lsa — `r || target` orqali original class qoladi.

</details>

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

Legacy method decorator — method'ni o'zgartirish yoki kuzatish uchun. Uchta argument oladi:

1. **`target`** — static method uchun constructor, instance method uchun prototype
2. **`propertyKey`** — method nomi (string yoki symbol)
3. **`descriptor`** — `PropertyDescriptor` (`value`, `writable`, `enumerable`, `configurable`)

Decorator `descriptor`'ni o'zgartirishi yoki yangi descriptor qaytarishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Method decorator chaqirig'i `__decorate([log], User.prototype, "greet", null)` formatda emit qilinadi (`c === 4`, `desc === null`). `__decorate` helper'da quyidagi bosqichlar:

1. **Descriptor olish** — `desc === null` bo'lsa, `r = Object.getOwnPropertyDescriptor(target, key)` orqali mavjud method'ning `PropertyDescriptor` olinadi. Bu object'da `value: originalMethod, writable: true, enumerable: false, configurable: true` bo'ladi (prototype method default'lari).

2. **Decorator chaqiruv** — `r = d(target, key, r) || r`. Decorator descriptor'ni o'zgartirib qaytarsa — yangi r ishlatiladi. `void` qaytarsa — original `r` qoladi (mutating mumkin: `descriptor.value = newFn` original objectni o'zgartiradi).

3. **Re-define** — `c > 3 && r` shart bajarilgani uchun `Object.defineProperty(target, key, r)` — yangi descriptor prototype'ga yoziladi. Bu eski method'ni almashtiradi.

**Muhim:** Decorator'lar **teskari tartibda** (`for i = decorators.length - 1; i >= 0; i--`) qo'llanadi — pastdan yuqoriga. `@a @b method` bo'lsa: avval `b`, keyin `a` qo'llanadi — function composition `a(b(method))` natijasini beradi.

Static method uchun `target` constructor function'ning o'zi (prototype emas). Instance method uchun — `Constructor.prototype`.

</details>

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

**Property decorator** — ikki argument oladi: `target` (prototype) va `propertyKey` (property nomi). `PropertyDescriptor` **olmaydi** — chunki class property'lar instance'da yaratiladi, prototype'da emas. Property decorator faqat **metadata saqlash** uchun ishlatiladi.

**Parameter decorator** — uchta argument oladi: `target`, `methodName`, `parameterIndex` (0-based). Bu ham faqat metadata saqlash uchun. **TC39 da parameter decorator mavjud emas** — bu legacy'ga xos.

<details>
<summary><strong>Under the Hood</strong></summary>

**Property decorator** chaqirig'i `__decorate([required], User.prototype, "name", void 0)` formatda — `desc === void 0` (`undefined`, `null` emas). Bu farq muhim: `c < 3` shart `false`, `desc === null` shart ham `false` — `r = desc = undefined` bo'ladi va `c > 3 && r` shart `false` bo'lib `Object.defineProperty` chaqirilmaydi.

Decorator faqat `target` (prototype) va `key` ni oladi — descriptor yo'qligi sababli value'ni o'zgartirish mumkin emas. Class property'lar `tsc` emitter tomonidan constructor body'da `this.propName = initializer` shaklida yoziladi — prototype'da hech qanday descriptor yo'q, faqat instance'da yaratiladi.

Bu sababdan property decorator faqat metadata saqlash uchun ishlatiladi: `Reflect.defineMetadata(key, value, target, propertyKey)` (reflect-metadata polyfill bilan) yoki o'z `Map<Constructor, ...>` bilan.

**Parameter decorator** chaqirig'i: `__param(0, inject)` helper orqali method/constructor decorator'ga "wrapper" sifatida qo'shiladi:

```javascript
__decorate([
  __param(0, inject),
  __param(1, inject)
], UserService.prototype, "getUser", null);
```

`__param(index, decorator)` curry pattern — `(target, key) => decorator(target, key, index)` qaytaradi. Parameter decorator return qiymati `__decorate`'da inkor qilinadi — faqat side effect (metadata yozish) muhim.

TC39 Stage 3 proposal'iga parameter decorator kiritilmadi — proposal'ni minimal va consensus'ga olib chiqish uchun parameter decorator alohida follow-on proposal'ga qoldirildi. NestJS va Angular DI hozircha legacy decorator + `reflect-metadata` ishlatadi.

</details>

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

Accessor decorator — getter/setter uchun. Method decorator bilan **bir xil signature** — farqi shundaki, descriptor'da `value` o'rniga `get` va `set` funksiyalari bo'ladi.

**Muhim:** Bitta property uchun getter va setter bitta descriptor'ni share qiladi — faqat birinchi e'lon qilingan accessor'ga decorator qo'yish mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Accessor descriptor method descriptor'dan farqli — `value`/`writable` o'rniga `get`/`set` funksiyalari:

```javascript
// Method descriptor
{ value: fn, writable: true, enumerable: false, configurable: true }

// Accessor descriptor
{ get: fn, set: fn, enumerable: false, configurable: true }
```

`__decorate` helper accessor uchun ham method bilan bir xil yo'l bo'yicha ishlaydi — `Object.getOwnPropertyDescriptor(target, key)` getter/setter pair'ni qaytaradi. Decorator ikkalasini o'zgartirishi mumkin (faqat birini yoki ikkalasini).

**Cheklov:** class body ichida bitta property uchun `get` va `set` alohida declaration bo'lsa-da, runtime'da ular bitta property'ning ikki tomoni (bitta descriptor). Shu sababdan decorator ikkala accessor'ga emas, faqat **document order bo'yicha birinchi** kelganiga qo'yiladi va u butun descriptor'ga (get + set) ta'sir qiladi. `get` va `set` ikkalasiga decorator yozsangiz TypeScript `TS1207: Decorators cannot be applied to multiple get/set accessors of the same name` compile xatosini beradi.

TC39 standartida bu cheklovni `accessor` keyword hal qiladi — auto-accessor bitta declaration, bitta decorator chaqirig'i `{ get, set }` pair'ni atomic uzatadi.

</details>

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

TC39 Stage 3 decorators — ECMAScript standart decorator proposal. TS 5.0'dan boshlab qo'llab-quvvatlanadi. **`experimentalDecorators` kerak emas** — default holatda ishlaydi.

Legacy'dan asosiy farqlar:

1. **`context` API** — barcha decorator turlari yagona `context` object oladi
2. **`addInitializer`** — class yoki instance yaratilganda chaqiriladigan callback
3. **`Symbol.metadata`** — native metadata mexanizm (reflect-metadata o'rniga)
4. **`accessor` keyword** — auto-accessor decorators
5. **Parameter decorator YO'Q** — TC39'da standartlashtirilmagan

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

<details>
<summary><strong>Under the Hood</strong></summary>

TC39 decorator proposal Stage 3'ga 2022-mart oyida o'tdi. Avvalgi Stage 2 "static decorators" proposal'i (2018-2021) butunlay tashlab yuborildi va dizayn noldan qayta ishlandi.

**Asosiy dizayn printsiplari (yangi proposal):**

1. **Composability** — har decorator turi yagona shape'ga ega: `(value, context) => newValue | void`
2. **Type-safety friendly** — `context.kind` discriminated union orqali decorator turini compile-time'da tekshirish mumkin
3. **No `target` argument** — `target` (prototype yoki constructor) o'rniga `context.access` orqali field/method'ga reference olinadi
4. **Initialization separation** — `addInitializer` orqali class/instance lifecycle'ga ulanish

**TS implementation:**

- type checker decorator signature'ni tip bo'yicha tekshiradi — `kind: "method"` bilan tagged context'ni method decorator'ga uzatadi
- legacy mode'da emitter `__decorate` helper'ini chiqaradi
- TC39 mode'da (default, 5.0+) emitter `__esDecorate` + `__runInitializers` helper'larini chiqaradi

**`accessor` keyword (TS 5.0):**

Auto-accessor `accessor` keyword Stage 3 decorators bilan birga TS 5.0'da yetkazildi. `accessor userName = "Ali"` syntax avtomatik quyidagiga compile bo'ladi:

```javascript
#userName = "Ali";  // private storage
get userName() { return this.#userName; }
set userName(value) { this.#userName = value; }
```

Bu pattern Object getter/setter'larni explicit yozishdan saqlaydi va decorator'lar uchun atomik `{ get, set }` pair'ni ta'minlaydi.

</details>

---

## TC39 Class Decorator

### Nazariya

TC39 class decorator — class constructor'ni va context'ni oladi. Yangi class qaytarsa — constructor almashtiriladi.

**Signature:** `(value: new (...args: any[]) => any, context: ClassDecoratorContext) => new (...args: any[]) => any | void`

<details>
<summary><strong>Under the Hood</strong></summary>

TC39 decorator'lar `__esDecorate` helper'iga compile qilinadi. Bu helper TC39 Stage 3 proposal'idagi abstract operation'larni implement qiladi.

`__esDecorate` signature:

```javascript
__esDecorate(
  ctor,           // class constructor yoki null (initialization fazasida)
  descriptorIn,   // property descriptor yoki null (class decorator uchun null)
  decorators,     // decorator funksiyalar array'i
  contextIn,      // { kind, name, static, private, access, ... }
  initializers,   // collected initializer functions
  extraInitializers
)
```

Class decorator chaqirig'ida `context` object:

```typescript
interface ClassDecoratorContext {
  kind: "class";
  name: string | undefined;       // anonymous class uchun undefined
  addInitializer(fn: () => void): void;
  metadata: Record<PropertyKey, unknown>;
}
```

`addInitializer` — class to'liq qurilgandan keyin chaqiriladigan callback'larni register qiladi. Bu `Symbol.metadata` ga ma'lumot yozish, registry'ga class qo'shish kabi side-effect operatsiyalar uchun.

Decorator return qiymati:
- **Yangi class constructor** — original'ni almashtiradi (mixin, subclass yaratish uchun)
- **`undefined` / `void`** — original class qoladi

`__esDecorate` chaqiruv tartibi:
1. A'zo (field/accessor/method) decorator'lar avval chaqiriladi — **document order** bo'yicha, static va instance ajratilmasdan (kod tartibi belgilaydi)
2. Class decorator eng oxirida — to'liq qurilgan class'ga
3. Bitta a'zoga bir nechta decorator bo'lsa, ular shu a'zoda **teskari tartibda** apply qilinadi (pastdan yuqoriga)

</details>

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

TC39 method decorator — method funksiyasini va context'ni oladi. Legacy'dan farqi — `PropertyDescriptor` emas, **method funksiyasining o'zi** birinchi argument.

**Signature:** `(value: (...args: any[]) => any, context: ClassMethodDecoratorContext) => ((...args: any[]) => any) | void`

Yangi funksiya qaytarsa — method almashtiriladi. `void` qaytarsa — original qoladi.

<details>
<summary><strong>Under the Hood</strong></summary>

TC39 method decorator `context` object'i quyidagi shape'ga ega:

```typescript
interface ClassMethodDecoratorContext<This = unknown, Value = (this: This, ...args: any[]) => any> {
  kind: "method";
  name: string | symbol;
  static: boolean;
  private: boolean;
  access: {
    has(object: This): boolean;
    get(object: This): Value;
  };
  addInitializer(initializer: (this: This) => void): void;
  metadata: DecoratorMetadataObject;
}
```

`access.get` — method'ga reference olish (private method'lar uchun ham ishlaydi). `access.has` — instance'da method mavjudligini tekshirish.

`__esDecorate` ichida method decorator quyidagi qadamlarda ishlanadi:

1. **Initial value extract** — class body'da `method() {...}` parse qilinganda, `value` = original method function
2. **Decorator chaqirig'i** — `result = decorator(value, context)`
3. **Result tekshirish:**
   - `result` function bo'lsa — `value = result` (method almashtiriladi)
   - `result` undefined/null bo'lsa — original `value` qoladi
   - Boshqa tip qaytarsa — `TypeError: Function expected`
4. **`addInitializer` callbacks** — class qurilganda chaqiriladi (instance initialization'dan oldin)
5. **Define on prototype** — yakuniy `value` `Object.defineProperty(prototype, name, { value, writable: true, enumerable: false, configurable: true })` orqali yoziladi

**Muhim farq legacy'dan:** TC39'da decorator descriptor olmaydi — faqat funksiya. `writable`, `enumerable`, `configurable` o'zgartirilmaydi. Agar bu kerak bo'lsa — `addInitializer` ichida `Object.defineProperty` chaqirish kerak.

Method'ga return qilingan funksiya `this` context'ni saqlashi shart — odatda regular `function (this: This, ...args)` ishlatiladi, arrow function emas (arrow `this` yo'q, qo'ng'iroq paytida instance'ga bog'lanmaydi).

</details>

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
    const result = originalMethod.call(this, ...args);
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
    return function (this: { currentUserRole: string }, ...args: any[]) {
      if (this.currentUserRole !== role) {
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

**Field decorator** — class property'ni decorate qiladi. Property **value'sini** oladi (yoki `undefined` agar initializer yo'q bo'lsa). Return value — initializer funksiya bo'lib, property'ning boshlang'ich qiymatini o'zgartiradi.

**Accessor decorator** — `accessor` keyword bilan declare qilingan auto-accessor'lar uchun. `accessor` keyword TS 5.0'da Stage 3 decorators bilan birga qo'shilgan — avtomatik getter/setter pair yaratadi. Decorator `get` va `set` funksiyalarni o'z ichiga olgan object oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Field decorator** `value` argument doim `undefined` — sababi field initialization instance qurilganda sodir bo'ladi, decorator chaqirig'i esa class definition vaqtida. Decorator return qiymati `(initialValue: T) => T` shaklidagi **initializer transformer**:

```javascript
// __runInitializers ichida
for (const initFn of fieldInitializers) {
  initialValue = initFn.call(this, initialValue);
}
this[fieldName] = initialValue;
```

`__runInitializers` constructor body'da `__runInitializers(this, _fieldName_initializers, initialValue)` chaqiruvi bilan invoke qilinadi. Initializer'lar zanjir tarzda chaqiriladi — har biri oldingisining natijasini oladi.

**Accessor decorator** auto-accessor'lar uchun. `accessor` keyword TS 5.0'da Stage 3 decorators bilan birga kiritildi. Compile natijasi:

```javascript
// accessor userName = "Ali"  →
#userName = "Ali";              // private field
get userName() { return this.#userName; }
set userName(value) { this.#userName = value; }
```

Accessor decorator `context` quyidagi shape'da:

```typescript
interface ClassAccessorDecoratorContext<This, Value> {
  kind: "accessor";
  // ... boshqa standart fieldlar
  access: {
    has(object: This): boolean;
    get(object: This): Value;
    set(object: This, value: Value): void;
  };
}
```

Decorator argument'i `{ get, set }` object — original auto-accessor'ning getter va setter funksiyalari. Return qiymati `{ get?, set?, init? }` object — har biri ixtiyoriy. `init` field decorator initializer kabi ishlaydi — boshlang'ich qiymatni transform qiladi.

</details>

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

Decorator factory — **parametr qabul qilib decorator qaytaradigan funksiya**. Bu `@decorator` o'rniga `@decorator(args)` syntax bilan ishlatiladi. Factory pattern decorator'ni configuration qilish imkonini beradi.

```typescript
// Oddiy decorator — parametrsiz
function log(originalMethod: (...args: any[]) => any, context: ClassMethodDecoratorContext) { /* ... */ }

// Decorator factory — parametrli
function logLevel(level: string) {
  return function (originalMethod: (...args: any[]) => any, context: ClassMethodDecoratorContext) { /* ... */ };
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Decorator factory closure mexanizmi bilan ishlaydi — outer funksiya parametr'larni capture qiladi, inner funksiya esa actual decorator vazifasini bajaradi.

**Source code emission:**

```typescript
@retry(3)
async fetchData() {}
```

`tsc` emitter `@retry(3)` ifodasini ikki bosqichli expression sifatida talqin qiladi:
1. **Factory chaqiruv** — `retry(3)` darhol evaluate qilinadi, decorator function qaytariladi
2. **Decorator apply** — qaytarilgan funksiya `__decorate` array'iga qo'yiladi

```javascript
// Compiled (legacy)
__decorate([retry(3)], ApiClient.prototype, "fetchData", null);
```

Diqqat: `retry(3)` array yaratishdan oldin chaqiriladi — factory'lar har class evaluation paytida bir marta evaluate qilinadi. Bu performance jihatdan muhim: `@retry(3)` har instance yaratilganda emas, class definition vaqtida bir marta `retry(3)` chaqiriladi.

**Type inference:** Factory return type decorator slot signature'iga mos kelishi kerak. Argument'i ixtiyoriy bo'lgan factory'ni overload bilan ifodalash mumkin — qaytarilgan funksiya method decorator signature'iga to'g'ri keladi:

```typescript
type MethodDecorator = (
  originalMethod: (...args: any[]) => any,
  context: ClassMethodDecoratorContext,
) => ((...args: any[]) => any) | void;

function memoize(): MethodDecorator;
function memoize(maxSize: number): MethodDecorator;
function memoize(maxSize?: number): MethodDecorator {
  return (originalMethod, context) => originalMethod;
}
```

Bu pattern Angular `@Injectable()` va NestJS `@Module()` kabi factory'larda keng ishlatiladi — argument'lar ixtiyoriy bo'lib, default behavior'ga ega.

</details>

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

Bir nechta decorator bitta target'ga qo'yilganda — ikki bosqichli jarayon:

1. **Evaluate** (yuqoridan pastga) — decorator factory'lar chaqiriladi
2. **Apply** (pastdan yuqoriga) — decorator'lar target'ga qo'llanadi

Bu matematik **function composition** — `@a @b @c method` → `a(b(c(method)))`.

**Class a'zolari decorator'lari tartibi standartga bog'liq:**

- **Legacy (TypeScript):** avval barcha instance a'zo decorator'lari (member tartibida), keyin barcha static a'zo decorator'lari, keyin constructor parameter decorator'lari, eng oxirida class decorator'lar.
- **TC39:** a'zo decorator'lari **document order**'da (static/instance aralash, kod tartibi bo'yicha) chaqiriladi; static va instance ajratilmaydi. Class decorator a'zolardan keyin chaqiriladi.

Ikkala standartda ham class decorator class a'zolari decorator'laridan **keyin** ishlanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Decorator composition tartibining sababi — JavaScript expression evaluation va composition semantikasidan kelib chiqadi.

**Factory evaluation (yuqoridan pastga):**

```typescript
@a()          // 1-chi: a() chaqiriladi → decoratorA qaytaradi
@b()          // 2-chi: b() chaqiriladi → decoratorB qaytaradi
method() {}
```

`@a()`, `@b()` — bu **expression**'lar. JavaScript expression'larni source order'da evaluate qiladi. Decorator factory'lar bu yerda chaqiriladi va decorator funksiyalar qaytariladi.

**Decorator application (pastdan yuqoriga):**

`__decorate` helper'da `for (var i = decorators.length - 1; i >= 0; i--)` loop'i — array oxiridan boshiga. Decorator'lar source code'da `[a, b]` tartibda joylashgan (TS emitter top-to-bottom yozadi), shuning uchun loop avval `b`, keyin `a` chaqiradi.

Bu function composition semantikasiga mos: `f ∘ g (x) = f(g(x))` — `g` avval `x`'ga qo'llanadi, keyin `f`. Decorator'larda `@a @b method` → `a(b(method))` — `b` avval method'ga qo'llanadi (eng yaqindagi decorator birinchi).

**Class member tartibi spec'da:**

TC39 spec'iga ko'ra a'zo decorator'lari **document order**'da chaqiriladi — static va instance a'zolar ajratilmaydi, kod tartibi (computed property nomlari bilan birga, chapdan o'ngga, yuqoridan pastga) belgilaydi. TypeScript legacy esa boshqacha: avval barcha instance a'zolar, keyin barcha static a'zolar. A'zo decorator'lar chaqirilgach, class decorator'lar ishlanadi; decorator'larning haqiqiy ta'siri (prototype va constructor'ni mutate qilish) barcha decorator chaqirilgandan keyin bir vaqtda qo'llanadi.

`addInitializer` callback'lar collected bo'ladi va alohida fazada chaqiriladi:
- Class addInitializer'lar — class decoration tugagandan keyin
- Instance addInitializer'lar — har `new` chaqirig'ida constructor body boshida (`__runInitializers`)
- Static addInitializer'lar — class definition'dan keyin darhol

</details>

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

TC39 decorators `Symbol.metadata` orqali **native metadata** mexanizm beradi. Bu legacy'dagi `reflect-metadata` polyfill'ning standart versiyasi. Har bir decorator `context.metadata` object'ga ma'lumot yozishi mumkin — bu metadata keyinchalik `Class[Symbol.metadata]` orqali o'qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`Symbol.metadata` ECMAScript Decorator Metadata proposal (Stage 3) tarkibida. TS 5.2'dan qo'llab-quvvatlanadi — `lib`'ga `ESNext.Decorators` qo'shilishi kerak (`Function[Symbol.metadata]` declaration shu lib faylida). Hozircha JS engine'lar (V8 ham) decorator'ni nativ implement qilmagan, shu sababli `Symbol.metadata` ham transpilation orqali polyfill qilinadi.

**Metadata lifecycle:**

1. **Class evaluation paytida** — `__esDecorate` har decorator chaqirig'iga `context.metadata` object'ni uzatadi. Bu object class darajasida shared — har decorator unga yozishi mumkin.

2. **Class qurilgandan keyin** — `Class[Symbol.metadata] = collectedMetadata` orqali metadata class'ga biriktiriladi. Subclass'lar metadata'ni inherit qiladi:

```javascript
// Spec algoritmi (soddalashtirilgan)
const metadata = Object.create(parentClass[Symbol.metadata] ?? null);
// ... decorator'lar metadata ga yozadi ...
Class[Symbol.metadata] = metadata;
```

Prototype chain orqali subclass parent metadata'ni avtomatik ko'radi, lekin o'z metadata'sini yozsa — parent'ga ta'sir qilmaydi (Object.create + prototypal inheritance).

**Legacy `reflect-metadata` bilan farq:**

| Xususiyat | reflect-metadata (legacy) | Symbol.metadata (TC39) |
|-----------|---------------------------|------------------------|
| Standartlashtirilgan | Yo'q (npm polyfill) | Ha (TC39 Stage 3) |
| Storage | Global WeakMap | Class property |
| Inheritance | Manual `Reflect.getMetadata` | Native prototype chain |
| Type information | `emitDecoratorMetadata: true` bilan auto | Faqat decorator o'zi yozsa |
| Bundle size | polyfill kerak | qo'shimcha polyfill yo'q |

`emitDecoratorMetadata: true` — faqat legacy decorators bilan ishlaydi. TC39'da bunday flag yo'q — type information'ni decorator'da qo'lda yozish kerak (yoki Zod/io-ts kabi runtime schema library ishlatish).

</details>

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

Decorator'lar real-world'da quyidagi pattern'larda keng ishlatiladi. Barchasi **cross-cutting concerns** — business logic'dan ajratilgan infrastructure logic.

**Production framework'lar:**

- **Angular** — `@Component`, `@Injectable`, `@NgModule`, `@Input`, `@Output` — DI, template binding, change detection
- **NestJS** — `@Controller`, `@Get/Post/...`, `@Injectable`, `@Module`, `@UseGuards` — REST routing, DI, middleware
- **TypeORM** — `@Entity`, `@Column`, `@OneToMany` — ORM mapping
- **MobX** — `@observable`, `@action`, `@computed` — reactive state management
- **class-validator** — `@IsString`, `@Min`, `@IsEmail` — runtime validation

Bularning aksariyati hozircha **legacy decorators** ishlatadi (`experimentalDecorators: true`) — TC39'ga migration sekin kechmoqda. Sababi: legacy parameter decorator + `reflect-metadata` + `emitDecoratorMetadata` kombinatsiyasi DI uchun zarur, TC39 esa parameter decorator'siz alternative DI pattern'lar talab qiladi.

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

// TC39 — ❌ compile error
class Service {
  method(@inject param: string) {} // ❌ TS1206: Decorators are not valid here.
}

// Yechim: NestJS va Angular bular uchun boshqa pattern ishlatmoqda (inject token, constructor injection)
```

### 3. Decorator lar faqat class a'zolariga qo'llanadi

```typescript
// ❌ — oddiy funksiya ga decorator qo'yib bo'lmaydi
@log
function standalone() {} // ❌ TS1206: Decorators are not valid here.

// ❌ — arrow function ga qo'yib bo'lmaydi
const fn = @log () => {}; // ❌ TS1109: Expression expected (parse error)

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
  @logged name = "Ali";          // Diqqat: faqat initializer o'zgaradi
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
      console.log(`[${id}] threw ${name}:`, e);
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
