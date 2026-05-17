# Interview: Advanced OOP Patterns

> Mixins, composition vs inheritance, class expressions, abstract factory, type-state builder, intersection types, `satisfies`, advanced OOP patterns bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Amaliy savollar (Coding Challenges)](#amaliy-savollar-coding-challenges)

---

## Nazariy savollar

### Savol 1: Mixin nima? TypeScript da mixin pattern qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Mixin — class'ga **mustaqil behavior** qo'shadigan funksiya. JavaScript single inheritance — class faqat bitta class'dan `extends` qila oladi. Mixin shu cheklovni constructor signature generic'i orqali chetlab o'tadi.

### To'liq tushuntirish

Mixin function `Base` parameter qabul qiladi va `Base` ni extend qiladigan yangi class expression qaytaradi. TypeScript'da mixin pattern uchun standart constructor signature type:

```typescript
type GConstructor<T = {}> = new (...args: any[]) => T;
```

Mixin'ni chain qilish mumkin: `Mixin2(Mixin1(Base))`. Prototype chain har mixin uchun yangi level qo'shadi.

**Constrained mixin** — base class'dan ma'lum shape talab qilish: `GConstructor<{ name: string }>`. Constraint mos kelmasa compile error.

### Kod misol

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

console.log(entity.id);           // → 1
console.log(entity.createdAt);    // → Date instance
entity.softDelete();
console.log(entity.deletedAt);    // → Date instance

// Constrained mixin — base'dan shape talab qilish
function Printable<TBase extends GConstructor<{ name: string }>>(Base: TBase) {
  return class extends Base {
    print(): void { console.log(this.name); }
  };
}

class UserAccount {
  constructor(public name: string) {}
}

const PrintableUser = Printable(UserAccount); // ✅ name bor

class Anonymous {
  constructor(public id: number) {}
}
// const PrintableAnonymous = Printable(Anonymous);
// ❌ Property 'name' is missing in type 'Anonymous'
```

### Edge Cases

- **Anonymous class name:** mixin'lar anonymous class qaytaradi — `this.constructor.name` ko'pincha bo'sh string
- **`instanceof` chain:** `instance instanceof BaseEntity` → `true` (prototype chain orqali)
- **Method override:** subclass mixin parent mixin method'ini override qilishi mumkin — `super.method()` ishlaydi
- **Type parameter forwarding:** generic mixin'da `<TBase extends GConstructor>` — `Base` ning extra generic'larini saqlamaydi
- **Decorator alternativa:** TS 5.0+ ECMAScript decorators ba'zi mixin use case'larni almashtiradi

### Follow-up savollar

1. "Mixin ichida private field'larga kira olamanmi?" — TS `private` — yo'q (cross-class). `#private` — yo'q (lexical scope). Mixin o'z field'lariga kiradi
2. "Multiple inheritance simulyatsiyasi to'liqmi?" — Yo'q. Diamond problem — mixin chain'da bir method ikki marta override bo'lsa, oxirgisi yutadi

<details>
<summary><strong>Deep Dive</strong></summary>

Mixin pattern V8 hidden class optimization'iga zid kelishi mumkin — har mixin level yangi prototype hosil qiladi. Inline cache har mixin level uchun alohida tracking qiladi. Cold path'da bu ahamiyatsiz, hot loop'da `Mixin1(Mixin2(Mixin3(Base)))` chain monomorphic property access'ni murakkablashtiradi.

TC39 mixin'ni native qilish proposal'i (Class Friend, Class Brand) bor edi — Stage 1'da to'xtagan. Decorators (Stage 3) — alternative mexanizm: behavior'ni decorator orqali qo'shish.

</details>

</details>

---

### Savol 2: Composition vs Inheritance — qachon qaysi birini ishlatish kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Inheritance** — "X is a Y" munosabati. Subclass parent'ning butun shape va behavior'ini meros oladi. **Composition** — "X has a Y". Object boshqa object'lardan behavior'ni delegatsiya orqali oladi. "Favor composition over inheritance" (Gang of Four) — aksariyat hollarda composition afzal.

### To'liq tushuntirish

| Kriteriya | Inheritance | Composition |
|-----------|:-----------:|:-----------:|
| "is a" munosabat | Mos | Mos emas |
| "has a" munosabat | Mos emas | Mos |
| Ko'p behavior kerak | Single — yo'q | Multiple — ha |
| Runtime'da o'zgartirish | Yo'q | Ha (swap dependency) |
| Testing | Qiyin (parent mock) | Oson (DI bilan mock) |
| Coupling | Tight | Loose |
| Refactoring | Yorilish risk | Safe |

Inheritance hech bo'lmasa quyidagi shartlar hammasi bajarilganda:
1. Haqiqiy "is a" munosabati
2. Subclass parent ning butun API'sini meros olishi mantiqli
3. Liskov Substitution Principle buzilmaydi

Aksariyat real-world OOP'da composition + interface afzal — DI test'da mock'ni osongina inject qilish imkonini beradi.

### Kod misol

```typescript
// ❌ Inheritance — ulkan base class
class BaseService {
  log(msg: string): void { console.log(msg); }
  cache(key: string): unknown { return null; }
  validate(data: unknown): boolean { return true; }
  sendNotification(to: string, body: string): void { /* ... */ }
}

class UserService extends BaseService {
  createUser(name: string): void {
    if (!this.validate(name)) return;
    this.log(`Creating ${name}`);
    // sendNotification kerak emas, lekin meros olindi
  }
}

// ✅ Composition — faqat kerakli dependency'lar
interface Logger { log(msg: string): void; }
interface Validator { validate(data: unknown): boolean; }

class UserServiceComposed {
  constructor(
    private readonly logger: Logger,
    private readonly validator: Validator,
  ) {}

  createUser(name: string): void {
    if (!this.validator.validate(name)) return;
    this.logger.log(`Creating ${name}`);
  }
}

// Test'da mock'larni oson inject qilish
const mockLogger: Logger = { log: () => {} };
const mockValidator: Validator = { validate: () => true };
const userService = new UserServiceComposed(mockLogger, mockValidator);
```

### Edge Cases

- **Inheritance fragility:** parent method'ini o'zgartirish — barcha subclass'lar buzilishi mumkin
- **Diamond problem:** TypeScript single inheritance — diamond yo'q, lekin mixin'larda paydo bo'lishi mumkin
- **Composition runtime swap:** dependency container — runtime'da dependency'ni o'zgartirish (NestJS, Angular)
- **Mix qilish:** abstract class + composition — abstract'da shared logic, ichida composed dependency'lar

### Follow-up savollar

1. "DI framework qachon kerak?" — Katta app'da (50+ service). Kichik app'da manual DI yetarli
2. "Inheritance qachon shart?" — Template Method pattern (algorithm skeleton + override hooks), DOM Node, React Component (legacy)

</details>

---

### Savol 3: Class expression nima? Mixin'da nima uchun ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Class expression — class'ni **qiymat** sifatida yaratish (declaration emas). Mixin function class expression qaytaradi — `Base` parameter runtime'da ma'lum bo'ladi, declaration vaqtida emas.

### To'liq tushuntirish

Class declaration faqat top-level yoki function body'da yoziladi va statik nom oladi. Class expression — har joyda qiymat sifatida — variable'ga assign, function'dan return, conditional ichida.

Mixin pattern'da `class extends Base` — `Base` runtime parameter, declaration vaqtida noma'lum. Class declaration bilan buni qilish mumkin emas:

```typescript
function Mixin(Base) {
  class Result extends Base { }  // ✅ ham declaration ham ishlaydi (function scope)
  return Result;
}
```

Class declaration ham mumkin (function scope'da), lekin idiomatic — anonymous class expression.

### Kod misol

```typescript
// Class declaration
class UserAccount {
  name = "Ali";
}

// Class expression — anonymous
const UserAccount2 = class {
  name = "Ali";
};

// Named class expression — nom faqat class ichida
const UserAccount3 = class InternalName {
  whoAmI(): string { return InternalName.name; }
};
console.log(new UserAccount3().whoAmI()); // → "InternalName"
// InternalName; // ❌ tashqarida accessible emas

// Mixin uchun kritik
function Printable<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    print(): void { console.log(String(this)); }
  };
}

// Factory pattern
function createValidator<T>(
  check: (v: T) => boolean,
  errorMessage: string,
) {
  return class {
    static readonly errorMessage = errorMessage;
    validate(value: T): boolean { return check(value); }
  };
}

const NonEmptyValidator = createValidator<string>(
  (s) => s.length > 0,
  "String bo'sh bo'lmasin",
);
const validator = new NonEmptyValidator();
console.log(validator.validate("test")); // → true

// IIFE class expression
const Singleton = (class {
  private static instance: any = null;
  static get(): any {
    if (!this.instance) this.instance = new this();
    return this.instance;
  }
});
```

### Edge Cases

- **Named expression scope:** internal nom faqat class body'da accessible — recursive reference yoki `Class.name` uchun
- **Hoisting:** class declaration hoist bo'lmaydi (TDZ), class expression assignment qilingan vaqtdan keyin
- **Class as type vs value:** declaration ikkalasini ham qo'shadi, anonymous expression faqat value (alias kerak type uchun)
- **Decorator class expression'ga:** Stage 3 decorators class expression'ni ham decorate qiladi (TS 5.0+)

### Follow-up savollar

1. "Class declaration'ni mixin function ichida ishlatib bo'ladimi?" — Ha, lekin idiomatic anonymous expression — natija bir xil
2. "`class.toString()` namesi nima?" — Anonymous class — engine'ga bog'liq (V8: variable nomi inferred)

</details>

---

### Savol 4: Class faqat 1 ta class'dan extends qiladi, lekin ko'p interface implement qiladi. Sababi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

JavaScript'da har object'ning yagona `[[Prototype]]` chain'i — bitta zanjir, ikki parent yo'q. Interface'lar esa compile-time type-level contract — JS emit'da butunlay o'chiriladi. Runtime'da hech qanday chain emas, faqat shape tekshiruvi.

### To'liq tushuntirish

JS prototype model:

```
instance.__proto__ → Child.prototype → Parent.prototype → Object.prototype → null
```

`[[Prototype]]` har object uchun yagona — diamond problem'ning oldini olish (qaysi parent method ishlatiladi?). Bu ECMAScript spec'ining fundamental cheklovi.

Interface esa TypeScript abstraction. Compile-time'da class shape interface contract'iga rioya qilishi tekshiriladi. JS emit'da `implements` o'chiriladi — class oddiy JS class.

### Kod misol

```typescript
interface Printable { print(): string; }
interface Loggable { log(msg: string): void; }
interface Cacheable { invalidate(): void; }

class ServiceManager implements Printable, Loggable, Cacheable {
  print(): string { return "ServiceManager"; }
  log(msg: string): void { console.log(msg); }
  invalidate(): void { /* cache clear */ }
}

// Compiled JS — implements yo'q:
// class ServiceManager {
//   print() { return "ServiceManager"; }
//   log(msg) { console.log(msg); }
//   invalidate() { /* ... */ }
// }

const sm = new ServiceManager();
// sm instanceof Printable; // ❌ 'Printable' only refers to a type
// Runtime tekshiruv: structural type guard
function isPrintable(x: unknown): x is Printable {
  return typeof (x as Printable).print === "function";
}

// Runtime'da multiple behavior kerak bo'lsa — mixin
function Cached<T extends new (...args: any[]) => {}>(Base: T) {
  return class extends Base { invalidate(): void {} };
}
```

### Edge Cases

- **Implements + extends:** `class X extends Base implements I1, I2` — bittada inheritance, ko'p interface contract
- **Conflicting interface:** ikki interface'da bir xil method nomi farqli signature bilan — class implement qila olmaydi
- **Interface inheritance:** `interface I3 extends I1, I2` — interface'lar multiple inherit qila oladi (compile-time)
- **Declaration merging:** bir xil nom bilan interface — merge bo'ladi, class bilan emas

### Follow-up savollar

1. "Trait/protocol simulyatsiyasi qanday?" — Mixin + interface kombinatsiyasi
2. "Python multiple inheritance bilan farq?" — Python C3 linearization MRO — JS yo'q

</details>

---

### Savol 5: Abstract Factory pattern — TypeScript'da qanday implement qilinadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Abstract Factory — tegishli object'lar oilasini yaratish uchun interface beradi. Client concrete factory turini bilmasdan ishlaydi — UI theme, database driver, payment gateway kabi use case'lar uchun mos.

### To'liq tushuntirish

Pattern komponentlari:
1. **Abstract Factory** — har product oilasini yaratish method'lari (interface)
2. **Concrete Factory** — bitta oilani yaratadigan implementation
3. **Abstract Product** — har product turi uchun interface
4. **Concrete Product** — har factory'ning specific implementation'i

TypeScript bu pattern uchun ideal — interface'lar compile-time type safety beradi, runtime'da overhead yo'q (interface emit'da yo'q). Polymorphism strukturaviy typing orqali — concrete factory har doim abstract factory'ga assignable.

### Kod misol

```typescript
// Abstract Products
interface Button {
  render(): string;
  onClick(handler: () => void): void;
}

interface InputField {
  render(): string;
  getValue(): string;
}

// Abstract Factory
interface UIComponentFactory {
  createButton(label: string): Button;
  createInput(placeholder: string): InputField;
}

// Concrete Factory — Material Design
class MaterialUIFactory implements UIComponentFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="md-btn">${label}</button>`,
      onClick: (handler) => { /* attach */ },
    };
  }
  createInput(placeholder: string): InputField {
    return {
      render: () => `<input class="md-input" placeholder="${placeholder}">`,
      getValue: () => "",
    };
  }
}

// Concrete Factory — iOS
class IosUIFactory implements UIComponentFactory {
  createButton(label: string): Button {
    return {
      render: () => `<button class="ios-btn">${label}</button>`,
      onClick: (handler) => { /* attach */ },
    };
  }
  createInput(placeholder: string): InputField {
    return {
      render: () => `<input class="ios-input" placeholder="${placeholder}">`,
      getValue: () => "",
    };
  }
}

// Client — factory turini bilmaydi
function buildLoginForm(factory: UIComponentFactory): string {
  const input = factory.createInput("Ismingiz");
  const submitBtn = factory.createButton("Kirish");
  return `${input.render()} ${submitBtn.render()}`;
}

const platform: "ios" | "material" = "ios";
const factory: UIComponentFactory =
  platform === "ios" ? new IosUIFactory() : new MaterialUIFactory();
console.log(buildLoginForm(factory));
```

### Edge Cases

- **Yangi family qo'shish:** yangi `ConcreteFactory` osongina qo'shiladi — Open/Closed Principle
- **Yangi product qo'shish:** har Factory'ga yangi method qo'shilishi kerak — qiyin, lekin compile error orqali ushlanadi
- **Generic abstract factory:** `interface Factory<P extends Product>` — type-safe product family
- **DI integratsiya:** factory'ni DI container orqali inject qilish — runtime swap qulayligi

### Follow-up savollar

1. "Builder pattern bilan farq?" — Abstract Factory — product oilasi yaratadi. Builder — bitta murakkab object'ni qadam-baqadam quradi
2. "Static factory method'lar yetarli emasmi?" — Bitta factory uchun ha. Ko'p family uchun Abstract Factory polymorphism beradi

</details>

---

### Savol 6: Type-state Builder pattern — qanday implement qilinadi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type-state builder — har step'da builder type'i progressiv o'zgaradi, faqat kerakli method'larni allow qiladi. Foydalanuvchi noto'g'ri tartibda chaqirsa compile error — runtime tekshiruvga muhtoj emas.

### To'liq tushuntirish

Standard builder mutable `this` — har method `this` qaytaradi. Type-state builder esa har step yangi type qaytaradi. Bu generic state parameter orqali — har step'da state union'dan ma'lum state olib tashlanadi yoki qo'shiladi.

Faydasi: required field'lar to'ldirilmasdan `build()` chaqirilsa compile error. Real-world use case: HTTP request builder, query builder, configuration builder.

### Kod misol

```typescript
type RequestState = "unset" | "set";

interface RequestBuilder<
  Url extends RequestState,
  Method extends RequestState,
> {
  setUrl(url: string): RequestBuilder<"set", Method>;
  setMethod(method: "GET" | "POST" | "PUT"): RequestBuilder<Url, "set">;
  setHeader(key: string, value: string): RequestBuilder<Url, Method>;
}

interface CompleteRequestBuilder extends RequestBuilder<"set", "set"> {
  build(): { url: string; method: string; headers: Record<string, string> };
}

class HttpRequestBuilder<Url extends RequestState, Method extends RequestState>
  implements RequestBuilder<Url, Method>
{
  private url?: string;
  private method?: string;
  private headers: Record<string, string> = {};

  setUrl(url: string): RequestBuilder<"set", Method> {
    this.url = url;
    return this as unknown as RequestBuilder<"set", Method>;
  }

  setMethod(method: "GET" | "POST" | "PUT"): RequestBuilder<Url, "set"> {
    this.method = method;
    return this as unknown as RequestBuilder<Url, "set">;
  }

  setHeader(key: string, value: string): RequestBuilder<Url, Method> {
    this.headers[key] = value;
    return this;
  }

  build(this: CompleteRequestBuilder): {
    url: string;
    method: string;
    headers: Record<string, string>;
  } {
    return {
      url: (this as any).url,
      method: (this as any).method,
      headers: (this as any).headers,
    };
  }
}

const start = new HttpRequestBuilder<"unset", "unset">();

// ❌ build() — Url va Method "unset"
// start.build(); // Error: 'this' context not compatible

// ✅ to'liq chain
const request = start
  .setUrl("https://api.example.com/users")
  .setMethod("POST")
  .setHeader("Content-Type", "application/json")
  .build();

console.log(request);
// → { url: "...", method: "POST", headers: { "Content-Type": "..." } }
```

### Edge Cases

- **Cast cost:** type assertion (`as unknown as ...`) — type system bilan kurashish. Library author yaqasiga olishi mumkin
- **`this` parameter pattern:** alternativa — `build` method'ni faqat to'liq state'da accessible qilish
- **Phantom type:** generic parameter haqiqiy runtime value bermaydi — pure compile-time mexanizm
- **Builder reuse:** har method yangi type qaytaradi, lekin runtime'da bir xil instance — clone strategy kerak bo'lishi mumkin

### Follow-up savollar

1. "Branded type bilan bog'lanishi bormi?" — Ha, har ikkalasi phantom type — type-level invariant'lar
2. "Rust state pattern bilan farq?" — Rust har state alohida struct (zero-cost). TypeScript faqat type-level

<details>
<summary><strong>Deep Dive</strong></summary>

Type-state pattern — type system'da progress tracking. Rust state pattern'ining direct ekvivalenti yo'q (TS runtime instance bitta). Cast'lar bilan bypass mumkin — bu pattern guideline, mutlaq xavfsizlik emas.

Alternative API: discriminated union'lar va exhaustive check. Builder API ergonomic, lekin type system murakkab bo'lib ketadi. Industry'da: Effect.ts, fp-ts builder'lari, Zod schema builder.

TC39'da type-state'ni native qilish proposal yo'q — bu TypeScript-specific pattern. Run-time'da hech qanday Foydasi yo'q.

</details>

</details>

---

### Savol 7: Intersection type vs `extends` — class hierarchy bilan farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Intersection type `A & B` — strukturaviy birlashma (har ikki shape'ni mos). `extends` — class inheritance, prototype chain. Intersection — type-level only, runtime'da hech qanday class hosil qilmaydi.

### To'liq tushuntirish

Intersection use case:
- Function parameter — har ikki shape'ni mos qiymat qabul qilish
- Mixin natijasi — har mixin'dan shape'larni birlashtirish
- Plain object'larni `extends` qilish o'rniga

Class inheritance use case:
- Runtime'da `instanceof`
- Method override
- `super` orqali parent chaqirish

Intersection'da konfliktli property type'lar — `never` ga aylanadi (`string & number = never`). Class extends'da subclass parent property type'ini compatible bo'lishi shart.

### Kod misol

```typescript
// Intersection — strukturaviy
type Loggable = { log(msg: string): void };
type Cacheable = { cache(): void };
type Service = Loggable & Cacheable;

const service: Service = {
  log: (msg) => console.log(msg),
  cache: () => { /* ... */ },
};

// `extends` — class inheritance
class BaseLogger {
  log(msg: string): void { console.log(msg); }
}

class CachedLogger extends BaseLogger {
  cache(): void { /* ... */ }
}

const cl = new CachedLogger();
console.log(cl instanceof BaseLogger); // → true

// Intersection runtime'da hech narsa qilmaydi
const s: Service = { log: () => {}, cache: () => {} };
// s instanceof Loggable; // ❌ Loggable type, value emas

// Konfliktli property
type A = { x: string };
type B = { x: number };
type Conflict = A & B; // { x: never } — assign qilib bo'lmaydigan
// const c: Conflict = { x: ??? }; // hech qaysi qiymat mos kelmaydi

// Mixin natijasi — intersection orqali ifoda etiladi
type GConstructor<T = {}> = new (...args: any[]) => T;
function WithTimestamp<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
  };
}
function WithLogger<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    log(msg: string): void {}
  };
}

class BaseEntity { constructor(public id: number) {} }
const Enhanced = WithLogger(WithTimestamp(BaseEntity));
type EnhancedShape = InstanceType<typeof Enhanced>;
// shape: { id: number; createdAt: Date; log: (msg: string) => void }
```

### Edge Cases

- **Method signature intersection:** `(x: string) => void` & `(x: number) => void` — bivariant funksiya `(x: string & number) => void` = `never`
- **`unknown` intersection:** `T & unknown = T` (unknown identity)
- **Conflicting return:** `() => string & () => number` — `() => string & number = () => never`
- **Index signature merge:** intersection — har bir index signature merge bo'ladi

### Follow-up savollar

1. "Intersection vs union qachon?" — Intersection — "ham A ham B". Union — "A yoki B"
2. "Intersection runtime'da check'lash mumkinmi?" — Bir nechta type guard combine: `isLoggable(x) && isCacheable(x)`

</details>

---

### Savol 8: `satisfies` operator class property validation'i bilan qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`satisfies` (TS 4.9+) — qiymatning ma'lum type'ga rioya qilishini tekshiradi, lekin literal type'ni saqlaydi. Class property'da `as Type` o'rniga `satisfies Type` — type widening'ga yo'l qo'ymaydi.

### To'liq tushuntirish

`as Type` — explicit assertion, qiymat type'ga assignable deb assumption qiladi (xato bo'lsa ham TS oxirgi gapni assertion'ga qoldiradi). `satisfies Type` — bidirectional check: qiymat type'ga mos kelishi shart, lekin TS literal type'ni saqlaydi.

Class context'da `satisfies` foydali — `Record<K, V>` shape'iga rioya qilish, lekin har specific key uchun aniq literal type saqlash.

### Kod misol

```typescript
type StatusConfig = {
  [K in "pending" | "approved" | "rejected"]: {
    label: string;
    color: string;
  };
};

class OrderStatus {
  // ❌ `as` — type widening, literal yo'qoladi
  static configAs = {
    pending: { label: "Kutilmoqda", color: "yellow" },
    approved: { label: "Tasdiqlangan", color: "green" },
    rejected: { label: "Rad etilgan", color: "red" },
  } as StatusConfig;
  // configAs.pending.color: string (literal yo'qoldi)

  // ✅ `satisfies` — literal type saqlanadi
  static config = {
    pending: { label: "Kutilmoqda", color: "yellow" },
    approved: { label: "Tasdiqlangan", color: "green" },
    rejected: { label: "Rad etilgan", color: "red" },
  } satisfies StatusConfig;
  // config.pending.color: "yellow" (literal!)
}

const pendingColor = OrderStatus.config.pending.color; // → "yellow" literal
const approvedColor = OrderStatus.config.approved.color; // → "green" literal

// Xato ushlash
class ProductCategory {
  static categories = {
    electronics: 1,
    books: 2,
    // clothes: "3", // ❌ satisfies bilan — number kutilgan
  } satisfies Record<string, number>;
}

// Class field bilan
class FormConfig {
  fields = {
    name: { type: "text", required: true },
    age: { type: "number", required: false },
  } satisfies Record<string, { type: "text" | "number"; required: boolean }>;
}

const form = new FormConfig();
form.fields.name.type; // → "text" literal, not "text" | "number"
```

### Edge Cases

- **`as const` bilan:** `satisfies T as const` — har ikkala feature kombinatsiyasi. `const T = {...} as const satisfies SomeType`
- **Generic constraint:** `function f<T>(x: T satisfies BaseType)` — generic'da `extends` afzal
- **Class property modifier:** `static readonly config = {...} satisfies T` — readonly + literal preservation
- **`satisfies` after expression:** `obj.method() satisfies ReturnType` — runtime overhead yo'q, faqat compile check

### Follow-up savollar

1. "`as` vs `satisfies` qachon `as`?" — Type assertion sifatida (`unknown as Type`). Satisfies type widening yo'q
2. "`satisfies` bilan widening qachon kerak?" — Boshqa code'ga generic shape sifatida berish. Aksariyat use case'da widening kerak emas

<details>
<summary><strong>Deep Dive</strong></summary>

TS 4.9 release notes: `satisfies` literal preservation muammosini hal qilish uchun joriy qilingan. Eski yondashuv:

```typescript
const config: StatusConfig = { ... }; // widening
const config = { ... } as StatusConfig; // assertion, type check yo'q
const config: StatusConfig & typeof literalConfig = ...; // qiyin
```

TypeScript implementation: AST'da `SatisfiesExpression` node yaratiladi, type checker har ikki yo'nalishda assignability check qiladi. Emit'da hech narsa qoldirilmaydi.

</details>

</details>

---

### Savol 9: `#private` field static brand check va `in` operator qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`#field in obj` — `obj` ushbu class'ning instance'imi tekshiradi. ECMAScript Stage 4 brand check pattern — `instanceof` o'rniga, sub-class'larni ham qamraydi.

### To'liq tushuntirish

`#field in obj` — agar `obj` `[[PrivateBrand]]` slot'iga ega bo'lsa va shu class'da declared bo'lsa true qaytaradi. `instanceof` prototype chain orqali ishlaydi, brand check esa privacy lexical scope orqali.

Foydasi:
1. Subclass'lar uchun ham true qaytaradi (private brand inherit qilinadi)
2. `instanceof` realm chegaralarida buzilishi mumkin, brand check ishonchli
3. Type narrowing: TS bilan `is` predicate'da ishlatilishi mumkin

### Kod misol

```typescript
class PaymentProcessor {
  #transactionId: string;

  constructor(txId: string) {
    this.#transactionId = txId;
  }

  static isPaymentProcessor(obj: unknown): obj is PaymentProcessor {
    return obj !== null && typeof obj === "object" && #transactionId in obj;
  }
}

class StripeProcessor extends PaymentProcessor {
  charge(amount: number): void { /* ... */ }
}

const stripe = new StripeProcessor("tx_123");
console.log(PaymentProcessor.isPaymentProcessor(stripe)); // → true
console.log(PaymentProcessor.isPaymentProcessor({})); // → false

// Cross-realm ishonchli (Node.js vm modul misol)
import vm from "node:vm";
const otherRealm = vm.runInNewContext("({})"); // boshqa V8 context
console.log(PaymentProcessor.isPaymentProcessor(otherRealm)); // → false

// instanceof — cross-realm muammo
console.log(otherRealm instanceof Object); // → false (boshqa realm Object)
```

### Edge Cases

- **Subclass brand:** subclass parent'ning `#field` ni meros oladi — brand check parent class'da true qaytaradi
- **Conflicting `#name`:** subclass parent'da bor `#name` ni qayta declare qila olmaydi (lexical conflict)
- **`in` operator standard:** `"prop" in obj` har qanday property uchun, `#field in obj` faqat private field uchun
- **Memory inspection:** `#field` heap snapshot'da ko'rinmaydi — debugger uchun maxsus DevTools support

### Follow-up savollar

1. "`instanceof` o'rniga brand check qachon ishlatish kerak?" — Library yozayotganda (cross-realm) yoki private state mavjudligini tekshirayotganda
2. "TypeScript `#field in obj` bilan type narrow qiladimi?" — Ha, type guard sifatida ishlatilsa

<details>
<summary><strong>Deep Dive</strong></summary>

V8 implementation: `[[PrivateBrand]]` — har class declaration uchun yashirin internal slot. `#field in obj` bytecode'da `LdaPrivateName` + brand check sifatida emit qilinadi. `instanceof` esa `OrdinaryHasInstance` abstract operation orqali prototype chain'ni traverse qiladi — `[[GetPrototypeOf]]` har step uchun.

Realm cheklovi: har `vm.runInNewContext`, iframe, Worker — alohida V8 isolate yoki realm. Har realm o'zining `Object`, `Array`, `Map` global'lariga ega. `instanceof` faqat bir xil realm ichida ishonchli — cross-realm tekshiruv (`x instanceof Object`) `false` qaytarishi mumkin agar `x` boshqa realm'da yaratilgan bo'lsa.

Brand check esa class lexical scope'iga bog'liq, realm emas. `#field in obj` faqat shu class declaration'dan kelgan instance uchun `true` qaytaradi — realm'dan qat'iy nazar.

Spec referensi: ECMA-262 §7.3.30 `PrivateElementFind`, §13.10.1 `RelationalExpression` `in` operator semantics.

</details>

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 10: Mixin output — prototype chain [Middle+]

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

class UserAccount {
  constructor(public name: string, public age: number) {}
}

const SmartUser = Serializable(Loggable(UserAccount));
const user = new SmartUser("Ali", 25);

console.log(user.name);
console.log(user.serialize());
console.log(user instanceof UserAccount);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
Ali
{"name":"Ali","age":25}
true
```

### To'liq tushuntirish

1. `user.name` → `"Ali"` — `UserAccount` constructor ishlaydi, parameter property orqali assign
2. `user.serialize()` → `JSON.stringify(this)` enumerable own property'larni oladi. `name` va `age` parameter property — enumerable
3. `user instanceof UserAccount` → `true` — prototype chain: `user → Serializable → Loggable → UserAccount → Object`

### Edge Cases

- **`this.constructor.name`:** mixin anonymous class qaytaradi — V8'da ko'pincha bo'sh string yoki inferred name
- **Method order:** mixin chain'da method resolution — eng tashqi mixin priority

### Follow-up savollar

1. "`user.log("hi")` ishlaydimi?" — Ha, `Loggable` mixin'dan
2. "Prototype chain'ni qanday tekshiraman?" — `Object.getPrototypeOf(obj)` ketma-ket

</details>

---

### Savol 11: Mixin field initialization order — tricky [Senior]

**Savol:** Output ni ayting:

```typescript
class BaseEntity {
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

const Enhanced = Loggable(Timestamped(BaseEntity));
new Enhanced();
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
Base constructor
Loggable setup, logs: undefined
Timestamped setup, createdAt: undefined
Base setup
```

### To'liq tushuntirish

Execution order:

1. `new Enhanced()` — constructor chain orqali Base constructor'iga yetadi
2. Base constructor'da `this.setup()` — virtual dispatch: eng tashqi override (`Loggable.setup`) chaqiriladi
3. Loggable.setup: `this.logs` → `undefined` — field initializer hali ishlamagan (super() qaytmagan)
4. `super.setup()` → Timestamped.setup: `this.createdAt` → `undefined` — bir xil sabab
5. `super.setup()` → Base.setup: `"Base setup"`
6. Base constructor tugadi → Timestamped field init → Loggable field init

ES2022 `useDefineForClassFields: true` semantics — field initializer'lar `super()` qaytgandan keyin ishlaydi.

### Kod misol

```typescript
// ✅ Pattern: explicit init method
class BaseEntity2 {
  init(): void { this.setup(); }
  setup(): void { /* ... */ }
}
const Enhanced2 = Loggable(Timestamped(BaseEntity2));
const e = new Enhanced2();
e.init(); // ✅ field'lar tayyor
```

### Edge Cases

- **Constructor'da abstract chaqirish:** abstract method chaqirilganda bir xil muammo
- **Parameter property:** `constructor(public x: number)` — body'da explicit init, super()'dan keyin

### Follow-up savollar

1. "Bu xatti-harakatdan qanday qochish mumkin?" — Constructor'da virtual method chaqirmaslik. Factory method yoki explicit `init()`
2. "Bu Java/C# da ham bir xilmi?" — Bir xil — fundamental OOP gotcha "calling virtual methods from constructor"

<details>
<summary><strong>Deep Dive</strong></summary>

ES2022 class field semantics (`useDefineForClassFields: true`, TS 4.6+ default):
1. `super()` chaqiriladi — parent constructor butun body bajariladi
2. Parent constructor tugagandan keyin, joriy class field initializer'lari **`[[Define]]` semantics** bilan ishlaydi (har biri `Object.defineProperty` ekvivalenti — accessor'larni trigger qilmaydi)
3. Constructor body qolgan qismi bajariladi

Eski semantics (`useDefineForClassFields: false`) — field initializer'lar `[[Set]]` semantics bilan ishlaydi (parent setter'larini trigger qilishi mumkin) — bu legacy behavior `Object.defineProperty` o'rniga oddiy assignment.

V8'da har field initializer alohida bytecode block sifatida emit qilinadi. Mixin chain'da har class o'z field initializer'iga ega — chain ichidagi har class qadam-baqadam initialize qilinadi.

Spec referensi: ECMA-262 §15.7.14 `InitializeInstanceElements` — class field'larni `[[Construct]]` paytida tartibli init qilish algoritmi. TC39 proposal "Class Fields" (Stage 4, ES2022) bu order'ni mustahkamladi.

</details>

</details>

---

### Savol 12: Mixin deserialize — xatoni toping [Middle+]

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

class UserAccount {
  constructor(public name: string, public age: number) {}
  greet(): string { return `Hi, ${this.name}`; }
}

const SmartUser = Serializable(UserAccount);
const json = new SmartUser("Ali", 25).serialize();
const restored = SmartUser.deserialize(json);
console.log(restored.greet());
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`new Base()` — `UserAccount` constructor `name` va `age` argument'larni talab qiladi. Argument'siz chaqirilganda `undefined`. `Object.assign` data qo'shadi, lekin constructor side effect'lar ishlamaydi.

### To'liq tushuntirish

`new Base()` bilan constructor invariant'lar buziladi. Yechim: `Object.create(Base.prototype)` — constructor chaqirmasdan to'g'ri prototype chain'ga ega object yaratish.

### Kod misol

```typescript
// ✅ Object.create bilan — constructor chaqirilmaydi
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

class UserAccount {
  constructor(public name: string, public age: number) {}
  greet(): string { return `Hi, ${this.name}`; }
}

const SmartUser = Serializable(UserAccount);
const json = new SmartUser("Ali", 25).serialize();
const restored = SmartUser.deserialize(json);
console.log(restored.greet()); // → "Hi, Ali"
```

### Edge Cases

- **Non-enumerable property'lar:** `JSON.stringify` enumerable'larni oladi, non-enumerable'lar yo'qoladi
- **Methods on instance:** arrow function field'lar (`onClick = () => {}`) — own property, JSON'da ko'rinadi, deserialize'da ham
- **Date va custom class:** `JSON.stringify(new Date())` → string. `JSON.parse` string'ni Date'ga qaytarmaydi
- **Production library:** `class-transformer`, `zod` schema validation — tip-top serializatsiya

### Follow-up savollar

1. "Validation kerak bo'lsa-chi?" — `zod`/`io-ts` schema bilan parse: invalid data'ni reject
2. "Polymorphic serialize?" — Discriminator field qo'shish: `{ __type: "UserAccount", ... }` keyin factory'da branch

</details>

---

### Savol 13: Constrained mixin yozing [Middle+]

**Savol:** `Validatable` mixin yozing — faqat `validate(): boolean` method'ga ega class'larga qo'llansin. `isValid` getter va `assertValid()` method qo'shsin.

```typescript
// const ValidUser = Validatable(UserAccount);
// const user = new ValidUser("Ali", 25);
// user.isValid → true/false
// user.assertValid() → throw agar valid emas

class UserAccount {
  constructor(public name: string, public age: number) {}
  validate(): boolean { return this.name.length > 0 && this.age > 0; }
}

class Product {
  constructor(public title: string) {}
  // validate() YO'Q
}
// Validatable(UserAccount) → ✅
// Validatable(Product) → ❌ compile error
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`GConstructor<{ validate(): boolean }>` constraint — `validate` method'i bor class'larni qabul qilish.

### Kod misol

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

class UserAccount {
  constructor(public name: string, public age: number) {}
  validate(): boolean { return this.name.length > 0 && this.age > 0; }
}

class Product {
  constructor(public title: string) {}
}

const ValidUser = Validatable(UserAccount);       // ✅
// const ValidProduct = Validatable(Product);     // ❌ validate yo'q

const user = new ValidUser("Ali", 25);
console.log(user.isValid); // → true
user.assertValid();         // OK

const invalid = new ValidUser("", -1);
console.log(invalid.isValid); // → false
// invalid.assertValid(); // ❌ Error: Validation failed for...
```

### Edge Cases

- **`get isValid()` getter:** har access'da `validate()` chaqiriladi — heavy validation hot path'da cache kerak
- **Constructor check yo'q:** mixin constructor'ni o'zgartirmaydi — invalid data hali ham yaratilishi mumkin
- **Async validate?** `validate(): Promise<boolean>` — async getter qo'llab-quvvatlanmaydi, async method'ga aylantirish kerak

### Follow-up savollar

1. "Decorator bilan o'rniga?" — Stage 3 decorators bilan ham mumkin, lekin runtime metadata kerak
2. "Validation rule'lar kompozitsiyasi?" — Strategy pattern: validate'ga array of rules berish

</details>

---

### Savol 14: Composition pattern — DI bilan [Senior]

**Savol:** Inheritance o'rniga composition ishlatib, `NotificationService` yozing. Logger, Validator, Sender — alohida dependency'lar sifatida inject bo'lsin:

```typescript
// notify("ali@test.com", "Hello") →
//   1. validate
//   2. send
//   3. log
//   Agar validate false → log error, send qilmaslik
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Interface contract'lar + constructor DI. `NotificationService` hech narsani extend qilmaydi — har dependency interface orqali inject bo'ladi.

### Kod misol

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
    private readonly logger: Logger,
    private readonly validator: Validator,
    private readonly sender: Sender,
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
const consoleLogger: Logger = {
  log: (msg) => console.log(`[LOG] ${msg}`),
};

const emailValidator: Validator = {
  validate: (to, body) => to.includes("@") && body.length > 0,
};

const emailSender: Sender = {
  send: async (to, body) => true,
};

// Production
const service = new NotificationService(consoleLogger, emailValidator, emailSender);
await service.notify("ali@test.com", "Hello");

// Test — mock inject
const mockSender: Sender = { send: async () => true };
const testService = new NotificationService(consoleLogger, emailValidator, mockSender);
```

### Edge Cases

- **Optional dependency:** `private readonly metrics?: MetricsCollector` — optional injection
- **Lazy initialization:** factory function'lar — `() => Logger` — circular dependency'ni hal qilish
- **DI container:** ko'p service'lar bo'lsa — NestJS/Inversify avtomatik resolution

### Follow-up savollar

1. "Inheritance versiyasi qaysi muammoga olib keladi?" — `BaseNotificationService` — har subclass barcha utility'larni meros oladi, mock qilish qiyin
2. "Function dependency'lar afzal emasmi?" — Functional yondashuv ham mumkin: `notify(deps, to, body)` — currying yoki object DI

<details>
<summary><strong>Deep Dive</strong></summary>

Dependency Injection mexanizm: constructor signature DI container uchun "resolution recipe". TypeScript `reflect-metadata` polyfill (`emitDecoratorMetadata: true`) bilan constructor parameter type'lar runtime'da mavjud — NestJS, Inversify, TypeDI shu metadata'dan foydalanadi.

Composition Root pattern: app entry point'da barcha dependency'lar bir joyda yig'iladi (`main.ts`). Service constructor'lar faqat o'z dependency'larini deklaratsiya qiladi, instantiation strategiyasini bilmaydi. Bu Inversion of Control (IoC) printsipining amaliy ifodasi.

SOLID printsiplari bilan bog'liq:
- **D — Dependency Inversion:** yuqori darajadagi modul (NotificationService) past darajadagi modulga (concrete Sender) emas, abstractsiyaga (Sender interface) bog'lanadi
- **L — Liskov Substitution:** har Sender implementation NotificationService kontekstida bir xil tarzda almashtirilishi mumkin
- **I — Interface Segregation:** har dependency alohida interface — Logger faqat `log`, Validator faqat `validate`

TypeScript'da DI container'lar runtime metadata'siz ham ishlaydi (TypeDI manual registration) yoki reflect-metadata bilan avtomatik resolve (NestJS, InversifyJS).

</details>

</details>

---

### Savol 15: `this` type — fluent chain'da subclass-aware return [Middle+]

**Savol:** Output ni ayting va sababini tushuntiring:

```typescript
class StringBuilder {
  protected parts: string[] = [];

  append(s: string): this {
    this.parts.push(s);
    return this;
  }

  build(): string {
    return this.parts.join("");
  }
}

class CapitalizedStringBuilder extends StringBuilder {
  capitalize(): this {
    const last = this.parts.pop() ?? "";
    this.parts.push(last.toUpperCase());
    return this;
  }
}

const result = new CapitalizedStringBuilder()
  .append("hello ")
  .append("world")
  .capitalize()
  .build();

console.log(result);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
hello WORLD
```

### To'liq tushuntirish

1. `new CapitalizedStringBuilder()` — `parts: []`
2. `.append("hello ")` — `parts: ["hello "]`, `this` type → `CapitalizedStringBuilder`
3. `.append("world")` — `parts: ["hello ", "world"]`, `this` type saqlangan
4. `.capitalize()` — oxirgi element pop, uppercase qilib qaytarish: `parts: ["hello ", "WORLD"]`
5. `.build()` → `"hello WORLD"`

`this` polymorphic return type — parent class method `this` qaytaradi, subclass instance'da chaqirilganda `this` type subclass'ga resolve bo'ladi.

### Edge Cases

- **`protected parts`:** `private` bo'lsa subclass kira olmaydi
- **`return this` shart:** method `void` qaytarsa chain buziladi
- **Generic class bilan:** `class GenericBuilder<T>` da `this` type generic'ni saqlaydi

### Follow-up savollar

1. "Builder result'ini boshqa subclass'ga aylantirib bo'ladimi?" — Yo'q — `this` polymorphic, lekin bitta type'da qoladi
2. "Immutable builder bilan qanday farq?" — Immutable — har step yangi instance qaytaradi, type-state pattern ishlatadi

</details>

---

### Savol 16: Class expression bilan factory — type extraction [Senior]

**Savol:** Quyidagi factory'dan instance type'ni qanday olamiz?

```typescript
function createEntity<T extends string>(tableName: T) {
  return class {
    static readonly table: T = tableName;
    id: number = 0;
    createdAt: Date = new Date();
  };
}

const UserEntity = createEntity("users");
const OrderEntity = createEntity("orders");
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`InstanceType<typeof ClassExpression>` — instance type oladi. Constructor type uchun `typeof UserEntity` ishlatish kerak.

### Kod misol

```typescript
function createEntity<T extends string>(tableName: T) {
  return class {
    static readonly table: T = tableName;
    id: number = 0;
    createdAt: Date = new Date();
  };
}

const UserEntity = createEntity("users");
const OrderEntity = createEntity("orders");

// Instance type
type UserInstance = InstanceType<typeof UserEntity>;
// = { id: number; createdAt: Date }

// Constructor type
type UserConstructor = typeof UserEntity;

const u: UserInstance = new UserEntity();
console.log(UserEntity.table); // → "users"
console.log(OrderEntity.table); // → "orders"

// Generic factory function bilan
function makeRepository<T extends new () => { id: number }>(EntityClass: T) {
  return class {
    private items: InstanceType<T>[] = [];
    add(e: InstanceType<T>): void { this.items.push(e); }
    findById(id: number): InstanceType<T> | undefined {
      return this.items.find(e => e.id === id);
    }
  };
}

const UserRepo = makeRepository(UserEntity);
const repo = new UserRepo();
repo.add(new UserEntity());
```

### Edge Cases

- **Generic class expression:** anonymous class generic parameter saqlash uchun outer function'da generic
- **Class expression hoisting:** class expression — value, assignment vaqtidan keyin accessible
- **Type alias:** har createEntity chaqirig'i yangi anonymous class — type'lar nominal emas, lekin distinct

### Follow-up savollar

1. "Generic class returnida narrowing yo'qoladi-chi?" — Yo'q — TS class expression'ning generic context'ini saqlaydi
2. "Static method generic bilan?" — Static metod o'z generic'iga ega, lekin class T'ga kira olmaydi

<details>
<summary><strong>Deep Dive</strong></summary>

`InstanceType<T>` utility'ning ichki implementatsiyasi:

```typescript
type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

Conditional type + `infer` orqali constructor return type'ni "extract" qiladi. `abstract new` signature (TS 4.2+) — abstract class'larni ham qamrash uchun.

Class expression generic context: `createEntity<T extends string>(tableName: T)` — har chaqiriqda yangi anonymous class type yaratiladi. TypeScript bu class type'ni "generic instantiation"ning oqibati sifatida saqlaydi — `typeof UserEntity` va `typeof OrderEntity` distinct type'lar, lekin shape strukturaviy bir xil.

Bu pattern Rust'dagi monomorphization'ga o'xshash, lekin TypeScript erasure model — runtime'da generic farq yo'q. Compile-time'da har instantiation type system'da unique tracking olinadi.

Generic factory'da `InstanceType<T>` ishlatish — Higher-Kinded Type (HKT) simulyatsiyasiga yaqin: `T` constructor type, `InstanceType<T>` shu T'dan kelib chiqadigan instance type. Real HKT TypeScript'da yo'q, lekin `InstanceType`/`ConstructorParameters`/`ReturnType` utility'lar shu cheklovni qisman yengillashtiradi.

</details>

</details>

---

## Xulosa

- Mixin — single inheritance cheklovini chetlab o'tish, constructor signature generic orqali
- Constrained mixin — `GConstructor<{...}>` bilan base'dan shape talab qilish
- Composition > Inheritance — DI, testing, loose coupling. Inheritance — haqiqiy "is a" + shared implementation
- Class expression — runtime'da class yaratish, mixin va factory uchun zaruriy
- Multiple interface implement — compile-time contract, JS'da o'chiriladi
- Abstract Factory pattern — product family'lar, polymorphic factory
- Type-state Builder — har step'da type progressiv o'zgaradi, required field'lar compile-time tekshiriladi
- Intersection type — strukturaviy birlashma, `extends`ni almashtirmaydi
- `satisfies` (TS 4.9+) — class property'da literal type'ni saqlash + type check
- `#field in obj` brand check — runtime hard private + sub-class qamrov
- Mixin + constructor field init — xavfli, field'lar `super()` qaytgandan keyin init
