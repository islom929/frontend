# Bo'lim 11: Advanced OOP Patterns

> TypeScript'da advanced OOP — mixins bilan ko'p merosni amalga oshirish, `#` private field'larning brand check va security pattern'lari, class expressions, fluent API uchun `this` return type'ning advanced use case'lari, type-state Builder pattern, Composition vs Inheritance tanlash strategiyasi, intersection types bilan class type'ni kengaytirish, va `satisfies` bilan class property validation. Bu pattern'lar TypeScript'ning type system kuchini OOP dizaynda to'liq ishlatishga qaratilgan.
>
> **[10-classes.md](10-classes.md)'dan farqi:** 10 class asoslarini (access modifier'lar, inheritance, `this` method chaining, `private` vs `#`) yoritadi. Bu bo'lim **advanced use case'lar**ga fokus qiladi — brand check, type-state pattern, CRTP-style fluent API, mixin pattern'lari, va real-world design pattern'lar.

---

## Mundarija

- [Mixins — TypeScript'da Ko'p Meros](#mixins--typescriptda-kop-meros)
- [`#` Private Fields — Advanced Patterns](#-private-fields--advanced-patterns)
- [Class Expressions](#class-expressions)
- [Multiple Interfaces — Bir Nechta Interface Implement](#multiple-interfaces--bir-nechta-interface-implement)
- [Abstract Factory Pattern](#abstract-factory-pattern)
- [`this` Type — Advanced Fluent Patterns](#this-type--advanced-fluent-patterns)
- [Builder Pattern — Type-State](#builder-pattern--type-state)
- [Composition vs Inheritance](#composition-vs-inheritance)
- [Intersection Types va Classes](#intersection-types-va-classes)
- [`satisfies` bilan Class Property Validation](#satisfies-bilan-class-property-validation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Mixins — TypeScript'da Ko'p Meros

### Nazariya

JavaScript va TypeScript'da class faqat **bitta** class'dan `extends` qila oladi — bu single inheritance. Lekin ko'p hollarda class'ga bir nechta mustaqil behavior qo'shish kerak bo'ladi. Masalan, bir class ham `Serializable`, ham `EventEmitter`, ham `Loggable` bo'lishi kerak. Interface bilan faqat "shape" belgilash mumkin, lekin **implementation** bera olmaydi.

**Mixin** — bu muammoning yechimi. Mixin — class'ga behavior (method va property'lar) qo'shadigan **funksiya**. Mixin funksiya base class'ni argument sifatida qabul qiladi va yangi class qaytaradi — bu class base class'dan `extends` qiladi va ustiga yangi method'lar qo'shadi.

**TypeScript'da mixin pattern'ning asosiy qoidalari:**

1. **Constructor signature** — `new (...args: any[]) => T` — kerakli pattern, aks holda base class argumentlari yo'qoladi
2. **Mustaqillik** — mixin mustaqil behavior qo'shadi, boshqa mixin'larga bog'liq bo'lmasligi kerak
3. **Composition order right-to-left** — `A(B(C))` natija: `C → B → A` prototype chain

**Mixin vs Composition farqi:**

- **Mixin** — inheritance kengaytirish: `class NewClass extends BaseClass` chain'ni uzaytiradi
- **Composition** — delegation: `class NewClass { helper: Helper }` — object'ni property sifatida saqlaydi

Mixin "IS-A" munosabat saqlaydi (`instanceof` ishlaydi), composition "HAS-A" munosabat yaratadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Mixin function pattern — aslida **class expression** qaytaruvchi higher-order function:

```
mixinFunction(BaseClass) → NewClass extends BaseClass
```

TypeScript buni type system'da quyidagicha ko'radi:

```
1. Base class type = Constructor<T>
   (new bilan chaqirsa bo'ladigan har qanday function)

2. Mixin function takes Constructor<T>
   returns Constructor<T & MixinInterface>

3. Kompilator type'larni stack qiladi:
   Base                     → { baseMethod() }
   + Mixin1(Base)           → { baseMethod(), mixin1Method() }
   + Mixin2(Mixin1(Base))   → { baseMethod(), mixin1Method(), mixin2Method() }
```

**Runtime'da prototype chain:**

```
instance
  └── Mixin2.prototype    ← mixin2Method
        └── Mixin1.prototype    ← mixin1Method
              └── Base.prototype      ← baseMethod
                    └── Object.prototype
```

Har mixin funksiya yangi anonymous class yaratadi — bu class oldingi class'dan `extends` qiladi. Natijada prototype chain uzayib boradi. Bu normal inheritance chain — hech qanday magic yo'q.

**`new (...args: any[]) => T` nima uchun kerak:** Agar siz `new () => T` yozsangiz, faqat **no-argument constructor'li** class'lar qabul qilinadi. `any[]` bilan har qanday argument count'ni qo'llab-quvvatlaydi. `any[]`'ning sabab'i — mixin base class'ning parameter type'larini bilmaydi, shuning uchun umumiy pattern kerak. Bu TypeScript handbook'da tavsiya qilingan standart pattern.

**Mixin order right-to-left:** `Cacheable(Eventful(Base))` — eng ichki `Base` prototype chain'da eng past, eng tashqi `Cacheable` eng yuqori. Har mixin oldingi natijaning prototype'ini extend qiladi:

```
Base.prototype                    ← eng past
  ↑ extends
Eventful(Base).prototype          ← ortasi
  ↑ extends
Cacheable(Eventful(Base)).prototype ← eng yuqori
```

Chaqirilganda method resolution yuqoridan pastga: `instance.method()` → `Cacheable.method()` → agar yo'q → `Eventful.method()` → agar yo'q → `Base.method()`.

**Constrained mixin:** Mixin faqat ma'lum shape'ga ega class'larga qo'llanishini talab qilish mumkin:

```typescript
// Base class'da {name: string} bo'lishi shart
function Printable<TBase extends GConstructor<{ name: string }>>(Base: TBase) {
  return class extends Base {
    print(): void { console.log(this.name); }
  };
}
```

`GConstructor<{ name: string }>` — constraint. `name` bo'lmagan class'ga qo'llasa compile error.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Constructor type helper
type GConstructor<T = {}> = new (...args: any[]) => T;

// 2. Base class
class BaseEntity {
  id: number;
  constructor(id: number) {
    this.id = id;
  }
}

// 3. Timestamped mixin — createdAt, updatedAt qo'shadi
function Timestamped<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();

    touch(): void {
      this.updatedAt = new Date();
    }
  };
}

// 4. SoftDeletable mixin
function SoftDeletable<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    deletedAt: Date | null = null;

    softDelete(): void {
      this.deletedAt = new Date();
    }

    restore(): void {
      this.deletedAt = null;
    }

    get isDeleted(): boolean {
      return this.deletedAt !== null;
    }
  };
}

// 5. Validatable mixin
function Validatable<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    errors: string[] = [];

    validate(): boolean {
      this.errors = [];
      return this.errors.length === 0;
    }
  };
}

// 6. Mixin application — right-to-left composition
const FullEntity = Validatable(SoftDeletable(Timestamped(BaseEntity)));

const entity = new FullEntity(1);
entity.id;           // BaseEntity dan
entity.createdAt;    // Timestamped dan
entity.softDelete(); // SoftDeletable dan
entity.validate();   // Validatable dan

// 7. Constrained mixin — base class shape talab qiladi
function Printable<TBase extends GConstructor<{ name: string }>>(Base: TBase) {
  return class extends Base {
    print(): void {
      console.log(`[${this.name}]`);
    }
  };
}

class User {
  constructor(public name: string, public age: number) {}
}

class Product {
  constructor(public sku: string) {} // name yo'q
}

const PrintableUser = Printable(User);       // ✅ name bor
// const PrintableProduct = Printable(Product); // ❌ name yo'q

const pUser = new PrintableUser("Ali", 25);
pUser.print(); // "[Ali]"

// 8. Generic mixin bilan type inference
function Observable<TBase extends GConstructor, T>(Base: TBase) {
  return class extends Base {
    private observers: ((value: T) => void)[] = [];

    subscribe(observer: (value: T) => void): () => void {
      this.observers.push(observer);
      return () => {
        this.observers = this.observers.filter(o => o !== observer);
      };
    }

    notify(value: T): void {
      this.observers.forEach(o => o(value));
    }
  };
}

// 9. Mixin bilan method override
function Loggable<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    log(message: string): void {
      console.log(`[${new Date().toISOString()}] ${message}`);
    }
  };
}

class Service extends Loggable(class {}) {
  start(): void {
    this.log("Service started"); // Loggable'dan
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function Timestamped<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
  };
}

class User {
  constructor(public name: string) {}
}

const TimestampedUser = Timestamped(User);
```

```javascript
// Compiled JS — generic va type annotation'lar o'chiriladi
function Timestamped(Base) {
  return class extends Base {
    constructor(...args) {
      super(...args);
      this.createdAt = new Date();
    }
  };
}

class User {
  constructor(name) {
    this.name = name;
  }
}

const TimestampedUser = Timestamped(User);
// Oddiy class inheritance — hech qanday TypeScript magic yo'q
```

Mixin — sof TypeScript type system pattern. Runtime'da oddiy class inheritance chain. Kompilator har class'ga avtomatik `constructor(...args) { super(...args); }` qo'shadi.

</details>

---

## `#` Private Fields — Advanced Patterns

### Nazariya

[10-classes.md](10-classes.md)'da `#` private field va TypeScript `private` farqining **asoslari** yoritilgan. Bu yerda advanced use case'larga fokus qilamiz — **brand check**, **security pattern'lar**, va **library design**.

**`#` private field'larning noyob xususiyati — brand check:** `#field in obj` operatori orqali object'ning haqiqiy class instance ekanligini tekshirish mumkin. Bu `instanceof`'dan kuchliroq, chunki:

- `instanceof` prototype chain'ga asoslangan — prototype manipulation bilan aldash mumkin
- `#field in obj` real private field mavjudligini tekshiradi — class tashqarisida bunday field yaratish mumkin emas

```typescript
class Color {
  #red: number;
  #green: number;
  #blue: number;

  constructor(r: number, g: number, b: number) {
    this.#red = r;
    this.#green = g;
    this.#blue = b;
  }

  static isColor(obj: unknown): obj is Color {
    return typeof obj === "object" && obj !== null && #red in obj;
  }

  equals(other: unknown): boolean {
    if (!Color.isColor(other)) return false;
    // other endi Color ekanligi kafolatlangan — #field'larga access mumkin
    return this.#red === other.#red
      && this.#green === other.#green
      && this.#blue === other.#blue;
  }
}

const c1 = new Color(255, 0, 0);
const fake = { red: 255, green: 0, blue: 0 };

Color.isColor(c1);   // true
Color.isColor(fake); // false — #red yo'q, structural similarity yetarli emas
```

**Security use case'lar:** Library/SDK yozganda `#` ishlatish muhim — foydalanuvchilar sensitive data'ga access qila olmaydi. TypeScript `private` compile-time niyat, `#` runtime kafolat.

<details>
<summary><strong>Under the Hood</strong></summary>

**`#field in obj` semantikasi:** ECMAScript specification'ga ko'ra, `#field in obj` operator class scope ichida (method body'da, static block'da, getter/setter'da) ishlaydi. Class tashqarisida `#field` tokenining o'zi sintaksik xato — parser bloklaydi.

```
Class scope:                  | Class tashqarisi:
─────────────────             |─────────────────
#red in obj  ✅                | #red in obj  ❌ SyntaxError
this.#red = 5 ✅               | obj.#red  ❌ SyntaxError
```

Bu **class-scoped operator**. Static method ham class scope ichida sanaladi — shuning uchun `static isColor()` ichida `#field in obj` ishlaydi. Top-level funksiya class scope'dan tashqarida — parser xato beradi.

**Nima uchun kuchli:** Prototype chain'ni o'zgartirib `instanceof` natijasini aldash mumkin:

```typescript
const fake = Object.create(Color.prototype);
fake instanceof Color; // true — prototype trick
```

Lekin `#red in fake` — false, chunki `fake` constructor'dan o'tmagan, `#red` init bo'lmagan. Brand check qat'iy.

**Bracket notation farqi:** `obj["prop"]` — string key bilan access. `#field` string emas — u private name (WeakMap kabi key). Shuning uchun `obj["#red"]` bu oddiy property lookup, natija `undefined`. `#red in obj` esa private name check.

**Library design pattern:** Library yozganda bu pattern'lar keng qo'llaniladi:

1. **Opaque object'lar** — foydalanuvchi internal state'ni ko'ra olmaydi
2. **Nominal typing simulation** — brand check bilan "bu faqat bizning class tomonidan yaratilgan" kafolati
3. **Token/credential storage** — sensitive data `#` bilan, accidental serialization'dan himoyalangan
4. **Immutable value objects** — `equals()` method'da brand check

**Serialization:** `JSON.stringify()` `#` field'larni **ko'rmaydi** — chunki ular `Object.keys()`'da emas. Bu security + data protection. TypeScript `private` field'lar esa oddiy property, JSON'da ko'rinadi.

```typescript
class A { private x = 1; }
class B { #x = 1; }

JSON.stringify(new A()); // '{"x":1}' — TS private JSON'da bor
JSON.stringify(new B()); // '{}' — # field JSON'da yo'q
```

**`in` operator va compatibility:** `#field in obj` — TC39 Stage 4, ES2022. Node.js 16+, zamonaviy browser'lar qo'llab-quvvatlaydi. Eski target'larda TypeScript buni `WeakMap.has(obj)` pattern'iga compile qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Brand check — class identity
class Point {
  #x: number;
  #y: number;

  constructor(x: number, y: number) {
    this.#x = x;
    this.#y = y;
  }

  get x(): number { return this.#x; }
  get y(): number { return this.#y; }

  static isPoint(obj: unknown): obj is Point {
    return typeof obj === "object" && obj !== null && #x in obj;
  }

  distanceTo(other: unknown): number {
    if (!Point.isPoint(other)) {
      throw new Error("Expected Point instance");
    }
    return Math.sqrt((other.#x - this.#x) ** 2 + (other.#y - this.#y) ** 2);
  }
}

const p1 = new Point(0, 0);
const p2 = new Point(3, 4);
const fake = { x: 3, y: 4 }; // Structural similarity

Point.isPoint(p2);    // true
Point.isPoint(fake);  // false — # brand yo'q

p1.distanceTo(p2);    // 5
// p1.distanceTo(fake); // Runtime error: Expected Point instance

// 2. Security — API client with token
class ApiClient {
  #token: string;
  #refreshToken: string;

  constructor(token: string, refreshToken: string) {
    this.#token = token;
    this.#refreshToken = refreshToken;
  }

  async request(endpoint: string): Promise<Response> {
    return fetch(`https://api.example.com${endpoint}`, {
      headers: { Authorization: `Bearer ${this.#token}` },
    });
  }

  // Brand check bilan type narrowing
  static isApiClient(obj: unknown): obj is ApiClient {
    return typeof obj === "object" && obj !== null && #token in obj;
  }
}

const client = new ApiClient("secret", "refresh");
JSON.stringify(client); // '{}' — # field'lar yo'q, token himoyalangan

// 3. Nominal typing simulation — Branded value
class UserId {
  #value: string;
  #brand: true = true;

  constructor(value: string) {
    if (!/^\d+$/.test(value)) {
      throw new Error("UserId must be numeric string");
    }
    this.#value = value;
  }

  toString(): string {
    return this.#value;
  }

  static isUserId(obj: unknown): obj is UserId {
    return typeof obj === "object" && obj !== null && #brand in obj;
  }
}

function fetchUser(id: UserId): void {
  if (!UserId.isUserId(id)) {
    throw new Error("Expected UserId");
  }
  console.log(`Fetching user ${id}`);
}

fetchUser(new UserId("123"));                  // ✅
// fetchUser("123" as any);                     // Runtime fail
// fetchUser({ toString: () => "123" } as any); // Runtime fail — brand yo'q

// 4. Immutable value object with equality
class Money {
  #amount: number;
  #currency: string;

  constructor(amount: number, currency: string) {
    if (amount < 0) throw new Error("Amount cannot be negative");
    this.#amount = amount;
    this.#currency = currency;
  }

  get amount(): number { return this.#amount; }
  get currency(): string { return this.#currency; }

  static isMoney(obj: unknown): obj is Money {
    return typeof obj === "object" && obj !== null && #amount in obj;
  }

  add(other: Money): Money {
    if (!Money.isMoney(other)) {
      throw new Error("Can only add Money instances");
    }
    if (this.#currency !== other.#currency) {
      throw new Error(`Currency mismatch: ${this.#currency} vs ${other.#currency}`);
    }
    return new Money(this.#amount + other.#amount, this.#currency);
  }

  equals(other: unknown): boolean {
    if (!Money.isMoney(other)) return false;
    return this.#amount === other.#amount && this.#currency === other.#currency;
  }
}

const usd1 = new Money(100, "USD");
const usd2 = new Money(100, "USD");
const eur = new Money(100, "EUR");

usd1.equals(usd2);              // true
usd1.equals({ amount: 100, currency: "USD" }); // false — brand yo'q
// usd1.add(eur);               // Runtime error: Currency mismatch

// 5. Combined TypeScript private + # private
class DatabaseConnection {
  // # — security critical (runtime private)
  #password: string;

  // TS private — API design niyat (compile-time)
  private connectionString: string;

  constructor(host: string, password: string) {
    this.#password = password;
    this.connectionString = `postgres://${host}`;
  }

  async connect(): Promise<void> {
    // Internal implementation uses both
  }
}
```

</details>

---

## Class Expressions

### Nazariya

JavaScript va TypeScript'da class yaratishning ikki yo'li bor:

1. **Class declaration** — `class User { ... }` — nomli, hoisted emas (TDZ'da)
2. **Class expression** — `const User = class { ... }` — o'zgaruvchiga assign qilinadigan

Class expression class'ni **qiymat sifatida** ishlatish imkonini beradi — funksiya expression'ga o'xshash. Class ham first-class value.

```typescript
// Anonymous class expression
const UserClass = class {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
};

// Named class expression — nom faqat class body ichida accessible
const AnotherClass = class MyInternalName {
  whoAmI(): string {
    return MyInternalName.name; // ✅ Class ichida accessible
  }
};
// MyInternalName; // ❌ Tashqarida accessible emas
```

**Nima uchun class expression kerak:**

- **Factory pattern** — funksiya ichida dynamic class yaratish
- **Mixin pattern** — mixin funksiyalar class expression qaytaradi
- **Conditional class creation** — shart bo'yicha turli class'lar
- **Inline class** — bir marta ishlatish uchun

<details>
<summary><strong>Under the Hood</strong></summary>

Class expression va class declaration JavaScript engine'da bir xil prototype-based object yaratadi. Farq faqat binding'da:

```
Class Declaration: class Foo {}
  → Foo nomi scope'da accessible (TDZ bilan)
  → Foo.name === "Foo"

Class Expression: const Bar = class {}
  → Bar nomi scope'da accessible (const sifatida)
  → Bar.name === "Bar" (variable nomidan inferred)

Named Class Expression: const Logger = class InternalLogger {}
  → Logger scope'da accessible
  → InternalLogger faqat class body ichida
  → Logger.name === "InternalLogger" (explicit name ustun)
```

**`Class.name` inference:** JavaScript'da class nomi variable name'dan inferred — lekin faqat variable declaration contextida. Function argument sifatida berilgan class nomi bo'sh:

```typescript
const Foo = class {};
Foo.name; // "Foo"

function register(cls: any) {
  return cls.name;
}
register(class {}); // "" — bo'sh
```

**Type parameter va class expression:** Class expression generic ham bo'lishi mumkin:

```typescript
const Container = class<T> {
  constructor(public value: T) {}
};

const num = new Container(42);  // Container<number>
```

Mixin funksiyalarda bu pattern asos — har mixin class expression qaytaradi.

**Factory pattern'da class expression:** Har call'da yangi class yaratish mumkin — lekin runtime cost bor. Har yangi class — yangi constructor function + prototype object. Performance critical kodlarda factory pattern o'rniga singleton class yoki composition yaxshiroq.

**Inline class immutability:** Class expression'ni `const` bilan bog'laganingizda, variable immutable, lekin class'ning o'zi o'zgaruvchan (prototype'ga property qo'shish mumkin). Haqiqiy immutability uchun `Object.freeze(Class.prototype)`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Anonymous class expression
const Counter = class {
  private count = 0;

  increment(): void { this.count++; }
  get value(): number { return this.count; }
};

const c = new Counter();
c.increment();
c.value; // 1

// 2. Named class expression
const Logger = class InternalLogger {
  static version = "1.0";

  log(msg: string): void {
    console.log(`[${InternalLogger.version}] ${msg}`);
  }
};

// InternalLogger; // ❌ tashqarida yo'q
Logger.version; // "1.0"

// 3. Factory pattern — dynamic class
interface Validator<T> {
  validate(value: T): boolean;
  errorMessage: string;
}

function createValidator<T>(
  predicate: (value: T) => boolean,
  message: string
): new () => Validator<T> {
  return class implements Validator<T> {
    errorMessage = message;
    validate(value: T): boolean {
      return predicate(value);
    }
  };
}

const NonEmpty = createValidator<string>(
  (s) => s.trim().length > 0,
  "String bo'sh bo'lmasligi kerak"
);

const Positive = createValidator<number>(
  (n) => n > 0,
  "Son musbat bo'lishi kerak"
);

const nonEmpty = new NonEmpty();
nonEmpty.validate(""); // false

// 4. Conditional class — environment-based
interface LoggerInterface {
  log(msg: string): void;
  error(msg: string): void;
}

const AppLogger: new () => LoggerInterface =
  process.env.NODE_ENV === "production"
    ? class implements LoggerInterface {
        log(_msg: string): void { /* noop */ }
        error(msg: string): void {
          // External error tracking
        }
      }
    : class implements LoggerInterface {
        log(msg: string): void {
          console.log(`[LOG] ${msg}`);
        }
        error(msg: string): void {
          console.error(`[ERROR] ${msg}`);
        }
      };

// 5. Generic class expression
const Box = class<T> {
  constructor(public value: T) {}
  unwrap(): T { return this.value; }
};

const strBox = new Box("hello");
const numBox = new Box(42);

// 6. Mixin pattern — class expression qaytaradi
function withTimestamp<T extends new (...args: any[]) => {}>(Base: T) {
  return class extends Base {
    createdAt = new Date();
  };
}

// 7. IIFE bilan module pattern (eski approach)
const Singleton = (() => {
  let instance: unknown = null;

  return class SingletonImpl {
    private constructor() {}

    static getInstance(): SingletonImpl {
      if (!instance) {
        instance = new SingletonImpl();
      }
      return instance as SingletonImpl;
    }
  };
})();
```

</details>

---

## Multiple Interfaces — Bir Nechta Interface Implement

### Nazariya

TypeScript'da class faqat bitta class'dan `extends` qila oladi (single inheritance), lekin **bir nechta** interface'ni `implements` qila oladi. Bu class'ning bir nechta contract (shape)'ga mos kelishini ta'minlaydi.

```typescript
interface Serializable {
  serialize(): string;
}

interface Comparable<T> {
  compareTo(other: T): number;
}

interface Printable {
  toString(): string;
}

// Bitta class — uchta interface
class Product implements Serializable, Comparable<Product>, Printable {
  constructor(public name: string, public price: number) {}

  serialize(): string {
    return JSON.stringify({ name: this.name, price: this.price });
  }

  compareTo(other: Product): number {
    return this.price - other.price;
  }

  toString(): string {
    return `${this.name}: $${this.price}`;
  }
}
```

**Muhim qoidalar:**

1. **Method conflict** — agar ikki interface'da bir xil nomli method turli signature bilan bo'lsa, class har ikki signature'ga mos kelishi kerak. Aks holda conflict.
2. **`implements` shape check** — faqat compile-time. Method parameter type'lariga type bermaydi.
3. **Structural typing** — `implements`'siz ham interface'ga mos kelish mumkin (shape yetarli)

<details>
<summary><strong>Under the Hood</strong></summary>

**Method signature compatibility:** Class bir nechta interface'ni implement qilganda, kompilator har method'ni barcha interface'larga teksharadi. Bir xil method turli interface'larda farqli return type bilan bo'lsa, class return type'i ikkala interface'ga mos kelishi kerak:

```typescript
interface A { get(): string | number; }
interface B { get(): string; }

// class Impl implements A, B — agar class `get(): string` bo'lsa, mos keladi
// Chunki string — string | number subtype
class Impl implements A, B {
  get(): string { return "hello"; } // ✅ ikkalasiga ham mos
}
```

**Return type covariance:** Class method'ning return type'i interface method'ning return type'iga **subtype** bo'lishi kerak (covariant position). Ikki interface'ning return type'lari incompatible bo'lsa — class topish mumkin bo'lgan common subtype kerak.

**Parameter type contravariance:** Class method'ning parameter type'i interface method'ning parameter type'iga **supertype** bo'lishi kerak (contravariant position). Lekin amaliyotda class bir xil parameter type ishlatadi.

**Method overload resolution:** Agar class method overload qilingan bo'lsa, har interface method'ining signature'si class overload'ga mos kelishi kerak:

```typescript
interface A { fn(x: string): string; }
interface B { fn(x: number): number; }

class Impl implements A, B {
  fn(x: string): string;
  fn(x: number): number;
  fn(x: string | number): string | number {
    return typeof x === "string" ? x.toUpperCase() : x * 2;
  }
}
```

**`implements` va `instanceof`:** `implements` compile-time contract, `instanceof` runtime prototype check. Ikkisi bog'liq emas:

```typescript
interface Printable { print(): void; }
class Document implements Printable { print() {} }

const doc = new Document();
// doc instanceof Printable; // ❌ Printable runtime'da mavjud emas
doc instanceof Document; // ✅
```

Interface runtime'da o'chiriladi — faqat class va oddiy prototype-based type'lar `instanceof`'da ishlaydi.

**Declaration merging va interface:** Bir xil nomli interface'lar merge qilinadi. Bu class'ga qo'shimcha property/method qo'shishning bir usuli:

```typescript
interface User {
  name: string;
}

interface User {
  email: string; // merged
}

class UserImpl implements User {
  name = "";
  email = ""; // ikkalasi ham kerak
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy multiple implements
interface Serializable {
  serialize(): string;
}

interface Loggable {
  log(message: string): void;
}

interface Cacheable {
  cacheKey(): string;
  cacheTtl(): number;
}

class UserDto implements Serializable, Loggable, Cacheable {
  constructor(
    public readonly id: number,
    public name: string,
    public email: string
  ) {}

  serialize(): string {
    return JSON.stringify({ id: this.id, name: this.name, email: this.email });
  }

  log(message: string): void {
    console.log(`[User:${this.name}] ${message}`);
  }

  cacheKey(): string {
    return `user:${this.id}`;
  }

  cacheTtl(): number {
    return 3600;
  }
}

// Interface type sifatida ishlatish
function storeInCache(item: Cacheable): void {
  const key = item.cacheKey();
  const ttl = item.cacheTtl();
  // cache.set(key, item, ttl);
}

storeInCache(new UserDto(1, "Ali", "ali@test.com"));

// 2. Method signature compatibility
interface Reader {
  read(): string | null;
}

interface Writer {
  read(): string;  // Stricter return type
  write(data: string): void;
}

// Class must satisfy BOTH interfaces
class File implements Reader, Writer {
  private content: string = "";

  read(): string {  // Stricter type — mos keladi
    return this.content;
  }

  write(data: string): void {
    this.content = data;
  }
}

// 3. Generic interface with class
interface Repository<T> {
  findById(id: string): T | undefined;
  save(entity: T): void;
  delete(id: string): boolean;
}

interface Auditable {
  getLastAccess(): Date;
}

class UserRepo implements Repository<User>, Auditable {
  private items = new Map<string, User>();
  private lastAccess = new Date();

  findById(id: string): User | undefined {
    this.lastAccess = new Date();
    return this.items.get(id);
  }

  save(entity: User): void {
    this.items.set(entity.id, entity);
  }

  delete(id: string): boolean {
    return this.items.delete(id);
  }

  getLastAccess(): Date {
    return this.lastAccess;
  }
}

interface User {
  id: string;
  name: string;
}

// 4. Interface conflict va resolution
interface Speaker {
  speak(volume: "low" | "normal" | "high"): string;
}

interface Singer {
  speak(volume: "low" | "normal" | "high"): string;
}

// Ikkalasi bir xil signature — class bitta implementation bilan mos keladi
class Performer implements Speaker, Singer {
  speak(volume: "low" | "normal" | "high"): string {
    return `Speaking at ${volume}`;
  }
}

// 5. Declaration merging — class'ga qo'shish
interface Logger {
  log(msg: string): void;
}

interface Logger {
  error(msg: string): void; // merged
}

class ConsoleLogger implements Logger {
  log(msg: string): void {
    console.log(msg);
  }

  error(msg: string): void {
    console.error(msg);
  }
}
```

</details>

---

## Abstract Factory Pattern

### Nazariya

Abstract factory — bir nechta tegishli (related) object'larni yaratishni abstract qiluvchi pattern. TypeScript'da generic va abstract class'lar kombinatsiyasi bilan type-safe qilish mumkin.

**Oddiy factory** — bitta turdagi object yaratadi.
**Abstract factory** — **oilaga tegishli** (family of related) object'larni yaratadi.

**Qachon kerak:**

- **Cross-platform UI** — iOS/Android/Web uchun bir xil interface, lekin turli implementation
- **Theme switching** — Light/Dark theme uchun Button, Input, Modal birga
- **Test doubles** — production factory + test factory (mock komponentlar)
- **Environment-based** — development/production/staging konfiguratsiyasi

Masalan: UI component factory har xil theme (Light, Dark) uchun Button, Input, Modal yaratadi — lekin barcha component'lar o'zaro mos bo'lishi kerak (Light Button Light Input bilan ishlashi kerak, aralash bo'lmasligi kerak).

<details>
<summary><strong>Under the Hood</strong></summary>

Abstract factory runtime'da oddiy class hierarchy va polymorphism. Generic type parameter'lar va abstract method'lar compile-time'da o'chiriladi:

```
AbstractFactory (abstract class)
  ├── createButton(): abstract
  ├── createInput(): abstract
  │
  ├── LightFactory extends AbstractFactory
  │     ├── createButton() → new LightButton()
  │     └── createInput()  → new LightInput()
  │
  └── DarkFactory extends AbstractFactory
        ├── createButton() → new DarkButton()
        └── createInput()  → new DarkInput()
```

**Type safety kafolati:** Abstract factory pattern type system orqali **aralash komponentlar**'ni oldini oladi. Agar foydalanuvchi `LightFactory.createButton()` chaqirsa, natija `LightButton` bo'ladi — `DarkButton` bilan aralashtirilmaydi. Kompilator bu muvofiqlikni tekshiradi.

**Runtime cost:** Har method chaqiruv — yangi instance. Lekin kompilator buni optimize qila olmaydi (runtime state). Katta masshtablarda object pooling yoki cache ishlatiladi.

**Generic factory alternative:** Simpler case'lar uchun generic function yetarli:

```typescript
function createInstance<T>(Ctor: new () => T): T {
  return new Ctor();
}
```

Abstract factory murakkab — **bir nechta related type'lar** va **invariant'lar** saqlash kerak bo'lganda ishlatiladi.

**Dependency injection bilan aloqasi:** Abstract factory DI pattern'ning oldingi avlodi. Zamonaviy DI framework'lar (NestJS, Angular) abstract factory logikasini avtomatlashtiradi — provider'lar va injection token'lar orqali. Lekin abstract factory patterndagi "family of products" tushunchasi DI'da ham foydali.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Product interface'lar
interface Button {
  render(): string;
  onClick(handler: () => void): void;
}

interface Input {
  render(): string;
  getValue(): string;
}

// 2. Abstract factory
abstract class UIFactory {
  abstract createButton(label: string): Button;
  abstract createInput(placeholder: string): Input;

  // Concrete method — umumiy
  createForm(buttonLabel: string, inputPlaceholder: string): { button: Button; input: Input } {
    return {
      button: this.createButton(buttonLabel),
      input: this.createInput(inputPlaceholder),
    };
  }
}

// 3. Light theme products
class LightButton implements Button {
  constructor(private label: string) {}
  render(): string { return `<button class="light">${this.label}</button>`; }
  onClick(handler: () => void): void { handler(); }
}

class LightInput implements Input {
  private value = "";
  constructor(private placeholder: string) {}
  render(): string { return `<input class="light" placeholder="${this.placeholder}">`; }
  getValue(): string { return this.value; }
}

// 4. Dark theme products
class DarkButton implements Button {
  constructor(private label: string) {}
  render(): string { return `<button class="dark">${this.label}</button>`; }
  onClick(handler: () => void): void { handler(); }
}

class DarkInput implements Input {
  private value = "";
  constructor(private placeholder: string) {}
  render(): string { return `<input class="dark" placeholder="${this.placeholder}">`; }
  getValue(): string { return this.value; }
}

// 5. Concrete factory'lar
class LightUIFactory extends UIFactory {
  createButton(label: string): Button { return new LightButton(label); }
  createInput(placeholder: string): Input { return new LightInput(placeholder); }
}

class DarkUIFactory extends UIFactory {
  createButton(label: string): Button { return new DarkButton(label); }
  createInput(placeholder: string): Input { return new DarkInput(placeholder); }
}

// 6. Client code — factory'ni almashtirish oson
function buildUI(factory: UIFactory): void {
  const { button, input } = factory.createForm("Submit", "Enter name...");
  console.log(button.render());
  console.log(input.render());
}

buildUI(new LightUIFactory());
buildUI(new DarkUIFactory());

// 7. Test factory pattern
class MockButton implements Button {
  constructor(private label: string) {}
  render(): string { return `[Mock: ${this.label}]`; }
  onClick(handler: () => void): void { /* no-op */ }
}

class MockInput implements Input {
  constructor(private placeholder: string) {}
  render(): string { return `[Mock Input: ${this.placeholder}]`; }
  getValue(): string { return ""; }
}

class MockUIFactory extends UIFactory {
  createButton(label: string): Button { return new MockButton(label); }
  createInput(placeholder: string): Input { return new MockInput(placeholder); }
}

// Test kontekstida — real rendering'ni almashtirish
function testUI(): void {
  const factory = new MockUIFactory();
  const ui = factory.createForm("Test", "Enter");
  // Assertion'lar
}

// 8. Generic abstract factory
abstract class Repository<T extends { id: string }> {
  protected items: T[] = [];

  abstract create(data: Omit<T, "id">): T;

  findAll(): T[] {
    return [...this.items];
  }

  findById(id: string): T | undefined {
    return this.items.find(item => item.id === id);
  }
}

interface UserEntity {
  id: string;
  name: string;
  email: string;
}

class UserRepository extends Repository<UserEntity> {
  private nextId = 1;

  create(data: Omit<UserEntity, "id">): UserEntity {
    const user: UserEntity = { id: `u${this.nextId++}`, ...data };
    this.items.push(user);
    return user;
  }
}

const repo = new UserRepository();
const user = repo.create({ name: "Ali", email: "ali@test.com" });
```

</details>

---

## `this` Type — Advanced Fluent Patterns

### Nazariya

[10-classes.md](10-classes.md)'da `this` type'ning **asoslari** (method chaining, subclass compatibility) yoritilgan. Bu yerda **advanced pattern'larga** fokus qilamiz:

- **Type-state pattern** — builder step'larini compile-time'da kuzatish
- **CRTP simulation** — "curiously recurring template pattern" TypeScript'da
- **Multi-level fluent chain** — parent → subclass → sub-subclass zanjir

**Type-state pattern asosi:** Har method call yangi type qaytaradi — shu yangi type faqat mumkin bo'lgan keyingi method'larni expose qiladi. Bu "compile-time state machine":

```
Initial: Builder<Empty>
  .withName() → Builder<{ name: true }>
    .withEmail() → Builder<{ name: true; email: true }>
      .build() → User  ← faqat ikkala required set bo'lganda
```

Kompilator har step'ni tekshiradi va faqat required field'lar set bo'lganda `build()` method'ni expose qiladi.

**CRTP (Curiously Recurring Template Pattern):** C++'dagi pattern — base class subclass type'ni parameter sifatida oladi. TypeScript'da bu to'g'ridan-to'g'ri yo'q, lekin `this` type bilan simulate qilinadi:

```typescript
class Fluent {
  // this subclass'da subclass type'iga resolve bo'ladi
  setValue(val: string): this { return this; }
}

class FluentChild extends Fluent {
  setName(name: string): this { return this; }
}

new FluentChild()
  .setValue("x")  // FluentChild
  .setName("y")   // FluentChild — method chain saqlanadi
  .setValue("z"); // FluentChild
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`this` type resolution mexanizmi:** Kompilator `this` return type'ni **chaqiruv kontekstida** resolve qiladi. Har class uchun alohida `this` type mavjud, va shu class'da method chaqirilsa, `this` o'sha class'ga aylanadi.

```
class A { method(): this }   // A'da chaqirilsa — A qaytaradi
class B extends A            // B'da chaqirilsa — B qaytaradi
class C extends B            // C'da chaqirilsa — C qaytaradi

Har chaqiriqda:
  aInstance.method() → A
  bInstance.method() → B
  cInstance.method() → C
```

**Type-state pattern compile-time simulation:** Type-state pattern'da har method yangi type qaytaradi — `this` emas. Bu `this` type'ning cheklovini chetlab o'tish uchun:

```typescript
class Builder<State extends Record<string, boolean>> {
  private _state: Partial<Record<string, unknown>> = {};

  withName(name: string): Builder<State & { name: true }> {
    this._state.name = name;
    return this as any; // Type cast — runtime bitta object
  }
}
```

Runtime'da bitta object — type-level'da har step yangi type. Bu "phantom types" pattern — type parameter runtime'da mavjud emas, faqat compile-time kuzatuv.

**Type-state va `build()` gating:** `build()` method faqat "hammasi set" state'da expose qilinadi. Bu conditional type yoki overload bilan amalga oshadi:

```typescript
interface RequiredState {
  name: true;
  email: true;
}

class Builder<State> {
  // build() faqat State RequiredState'ga mos kelsa ishlaydi
  build(
    this: State extends RequiredState ? this : never
  ): User {
    // ...
  }
}
```

`this:` parameter — method "this" type'ini cheklaydi. Agar `this` type constraint'ga mos kelmasa, chaqiruv compile error.

**Method chaining va immutability:** Ba'zan fluent API immutable bo'ladi — har method yangi instance qaytaradi:

```typescript
class ImmutableBuilder {
  constructor(private readonly state: Record<string, unknown> = {}) {}

  with(key: string, value: unknown): this {
    return new (this.constructor as new (state: Record<string, unknown>) => this)({
      ...this.state,
      [key]: value,
    });
  }
}
```

`this.constructor` — instance'ning constructor'iga ishora qiladi. Subclass'da bu subclass constructor'iga resolve bo'ladi — shuning uchun `new` chaqiriqda to'g'ri instance yaratadi.

**Shallow clone gotcha:** `this` return type bilan clone pattern yozganda, `Object.create(Object.getPrototypeOf(this))` shallow — nested object'lar shared bo'ladi. Deep clone uchun `structuredClone()` (modern) yoki custom deep copy kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Basic fluent — 10-classes.md asoslari
class QueryBuilder {
  protected conditions: string[] = [];

  where(condition: string): this {
    this.conditions.push(condition);
    return this;
  }

  build(): string {
    return this.conditions.join(" AND ");
  }
}

class AdvancedQueryBuilder extends QueryBuilder {
  private orderByClause = "";

  orderBy(field: string): this {
    this.orderByClause = ` ORDER BY ${field}`;
    return this;
  }

  override build(): string {
    return super.build() + this.orderByClause;
  }
}

const q = new AdvancedQueryBuilder()
  .where("age > 18")
  .where("active = true")
  .orderBy("name")
  .build();

// 2. Type-state pattern — compile-time state tracking
interface NeedsName {
  withName(name: string): NeedsEmail;
}

interface NeedsEmail {
  withEmail(email: string): OptionalAge;
}

interface OptionalAge {
  withAge(age: number): OptionalAge;
  build(): User;
}

interface User {
  name: string;
  email: string;
  age?: number;
}

class UserBuilder implements NeedsName {
  private data: Partial<User> = {};

  withName(name: string): NeedsEmail {
    this.data.name = name;
    return this as unknown as NeedsEmail;
  }

  withEmail(email: string): OptionalAge {
    this.data.email = email;
    return this as unknown as OptionalAge;
  }

  withAge(age: number): OptionalAge {
    this.data.age = age;
    return this as unknown as OptionalAge;
  }

  build(): User {
    return {
      name: this.data.name!,
      email: this.data.email!,
      age: this.data.age,
    };
  }
}

// Ishlatish — tartib majburiy
const user = new UserBuilder()
  .withName("Ali")    // NeedsName → NeedsEmail
  .withEmail("ali@test.com") // NeedsEmail → OptionalAge
  .withAge(25)        // OptionalAge
  .build();           // User

// new UserBuilder().build();            // ❌ NeedsName'da build yo'q
// new UserBuilder().withName("Ali").build(); // ❌ NeedsEmail'da build yo'q

// 3. CRTP simulation — subclass fluent
class BaseRequest {
  protected url = "";
  protected method: "GET" | "POST" = "GET";

  setUrl(url: string): this {
    this.url = url;
    return this;
  }

  setMethod(method: "GET" | "POST"): this {
    this.method = method;
    return this;
  }
}

class HttpRequest extends BaseRequest {
  private headers: Map<string, string> = new Map();

  setHeader(key: string, value: string): this {
    this.headers.set(key, value);
    return this;
  }
}

class JsonRequest extends HttpRequest {
  private body: unknown = null;

  setBody(data: unknown): this {
    this.body = data;
    return this.setHeader("Content-Type", "application/json");
  }
}

// Uch level inheritance, method chain ishlaydi
const req = new JsonRequest()
  .setUrl("/api/users")   // BaseRequest → JsonRequest (this)
  .setMethod("POST")       // BaseRequest → JsonRequest
  .setHeader("Auth", "Bearer xyz") // HttpRequest → JsonRequest
  .setBody({ name: "Ali" }); // JsonRequest

// 4. Immutable fluent builder
class ImmutableConfig {
  constructor(private readonly data: Readonly<Record<string, unknown>> = {}) {}

  with(key: string, value: unknown): ImmutableConfig {
    return new ImmutableConfig({ ...this.data, [key]: value });
  }

  get<T>(key: string): T | undefined {
    return this.data[key] as T | undefined;
  }
}

const cfg1 = new ImmutableConfig();
const cfg2 = cfg1.with("host", "localhost");
const cfg3 = cfg2.with("port", 3000);

cfg1.get("host"); // undefined
cfg2.get("host"); // "localhost"
cfg3.get("port"); // 3000
// cfg1 va cfg2 alohida immutable instance'lar

// 5. Validation DSL — fluent error collection
class StringValidator {
  private rules: Array<(val: string) => string | null> = [];

  minLength(n: number): this {
    this.rules.push(val =>
      val.length < n ? `Must be at least ${n} chars` : null
    );
    return this;
  }

  maxLength(n: number): this {
    this.rules.push(val =>
      val.length > n ? `Must be at most ${n} chars` : null
    );
    return this;
  }

  pattern(regex: RegExp, message: string): this {
    this.rules.push(val => regex.test(val) ? null : message);
    return this;
  }

  validate(value: string): string[] {
    return this.rules
      .map(rule => rule(value))
      .filter((msg): msg is string => msg !== null);
  }
}

const passwordValidator = new StringValidator()
  .minLength(8)
  .maxLength(64)
  .pattern(/[A-Z]/, "Must contain uppercase")
  .pattern(/[0-9]/, "Must contain digit");

passwordValidator.validate("abc");
// ["Must be at least 8 chars", "Must contain uppercase", "Must contain digit"]
```

</details>

---

## Builder Pattern — Type-State

### Nazariya

Builder pattern — complex object'ni bosqichma-bosqich qurishga mo'ljallangan pattern. TypeScript'da Builder'ning kuchi — **type system bilan integratsiya**. Har step yangi type qaytaradi, natijada `build()` faqat barcha required field'lar set bo'lgandan keyin chaqirilishi mumkin.

**Builder vs Constructor:**

```typescript
// ❌ Constructor — ko'p parameter, noaniqlik
const config = new ServerConfig(8080, "localhost", true, false, 30000, "/api", true, 100);
// Qaysi parameter nima — bilish qiyin

// ✅ Builder — har step'da nima set qilayotganingiz aniq
const config = ServerConfig.builder()
  .port(8080)
  .host("localhost")
  .enableCors(true)
  .timeout(30000)
  .build();
```

**Type-state Builder** — phantom types pattern bilan compile-time'da required field'larni kuzatish. Agar required field set bo'lmasa, `build()` method expose qilinmaydi.

**Batafsil Builder pattern implementation va Design Pattern fundamentals → [21-design-patterns.md](21-design-patterns.md)'da.**

<details>
<summary><strong>Under the Hood</strong></summary>

**Phantom types va type-state:** Builder'ning type parameter'i runtime'da mavjud emas — faqat compile-time tracking. Bu "phantom types" deb ataladi:

```
Initial: Builder<Empty>
  state: {}

.port(8080) → Builder<{ port: true }>
  state: { port: 8080 }  // runtime
  type: { port: true }    // phantom — compile-time only

.host("localhost") → Builder<{ port: true; host: true }>
  state: { port: 8080, host: "localhost" }
  type: { port: true; host: true }

.build() — faqat type { port: true; host: true } bo'lganda expose qilinadi
```

**Runtime'da** bitta object mutation bor. **Type-level'da** har step yangi interface. Kompilator state progression'ni kuzatadi va `build()` method'ni faqat to'g'ri state'da expose qiladi.

**Implementation strategiyalari:**

1. **Step interfaces** — har step uchun alohida interface (`NeedsPort`, `NeedsHost`, `OptionalConfig`)
2. **Generic state tracking** — `Builder<State>` — State type parameter'da state'ni kuzatish
3. **Conditional build** — `build(this: State extends Required ? this : never)`

Step interface'lar oddiy, generic state ancha flexible. Real library'lar (Zod, Prisma) generic state tracking ishlatadi.

**`this as unknown as NewType` cast:** Step interface pattern'da `return this as unknown as NewType` ishlatiladi. Bu type-level cast — runtime'da hech narsa o'zgarmaydi, faqat TypeScript type'ni keyingi interface sifatida ko'radi. Bu "safe" cast chunki runtime object bir xil, faqat type perspective o'zgaradi.

**Immutable vs mutable builder:** Ikki variant:

- **Mutable** — bir xil object mutatsiya qilinadi, har `return this`
- **Immutable** — har step yangi object qaytaradi, `return new Builder(...)`

Mutable — tezroq, kam memory. Immutable — xavfsizroq (partial builder'larni saqlab qolish mumkin), konkurrent safe.

**Type-state va runtime mismatch:** Type-state pattern'ning nozik joyi — type compile-time check, runtime'da yo'q. Agar type cast orqali bypass qilinsa, runtime'da `build()` noto'g'ri state'da chaqirilishi mumkin. Shuning uchun type-state faqat honest usage uchun — type assertion bilan bypass qilsa, runtime xato kutish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy Builder (mutable)
interface ServerConfig {
  readonly port: number;
  readonly host: string;
  readonly enableCors: boolean;
  readonly maxConnections: number;
  readonly timeout: number;
}

class ServerConfigBuilder {
  private config: Partial<ServerConfig> = {};

  port(port: number): this {
    this.config = { ...this.config, port };
    return this;
  }

  host(host: string): this {
    this.config = { ...this.config, host };
    return this;
  }

  enableCors(enable: boolean): this {
    this.config = { ...this.config, enableCors: enable };
    return this;
  }

  maxConnections(max: number): this {
    this.config = { ...this.config, maxConnections: max };
    return this;
  }

  timeout(ms: number): this {
    this.config = { ...this.config, timeout: ms };
    return this;
  }

  build(): ServerConfig {
    return {
      port: this.config.port!,
      host: this.config.host!,
      enableCors: this.config.enableCors ?? false,
      maxConnections: this.config.maxConnections ?? 100,
      timeout: this.config.timeout ?? 30000,
    };
  }
}

// 2. Type-State Builder — step interface'lar
interface NeedsPort {
  port(port: number): NeedsHost;
}

interface NeedsHost {
  host(host: string): OptionalConfig;
}

interface OptionalConfig {
  enableCors(enable: boolean): OptionalConfig;
  timeout(ms: number): OptionalConfig;
  maxConnections(max: number): OptionalConfig;
  build(): ServerConfig;
}

class StrictServerConfigBuilder implements NeedsPort {
  private config: Partial<ServerConfig> = {};

  port(port: number): NeedsHost {
    this.config.port = port;
    return this as unknown as NeedsHost;
  }

  host(host: string): OptionalConfig {
    this.config.host = host;
    return this as unknown as OptionalConfig;
  }

  enableCors(enable: boolean): OptionalConfig {
    this.config.enableCors = enable;
    return this as unknown as OptionalConfig;
  }

  timeout(ms: number): OptionalConfig {
    this.config.timeout = ms;
    return this as unknown as OptionalConfig;
  }

  maxConnections(max: number): OptionalConfig {
    this.config.maxConnections = max;
    return this as unknown as OptionalConfig;
  }

  build(): ServerConfig {
    return {
      port: this.config.port!,
      host: this.config.host!,
      enableCors: this.config.enableCors ?? false,
      maxConnections: this.config.maxConnections ?? 100,
      timeout: this.config.timeout ?? 30000,
    };
  }
}

// Ishlatish — tartib majburiy
const config = new StrictServerConfigBuilder()
  .port(3000)        // ✅ NeedsPort → NeedsHost
  .host("localhost") // ✅ NeedsHost → OptionalConfig
  .enableCors(true)  // ✅ OptionalConfig
  .build();          // ✅ build() faqat OptionalConfig'da

// new StrictServerConfigBuilder().build();
// ❌ Property 'build' does not exist on type 'NeedsPort'

// 3. Generic state tracking Builder
type BuilderState = "empty" | "hasName" | "hasEmail" | "complete";

class GenericBuilder<State extends BuilderState = "empty"> {
  private name?: string;
  private email?: string;

  withName(name: string): GenericBuilder<"hasName"> {
    this.name = name;
    return this as unknown as GenericBuilder<"hasName">;
  }

  withEmail(
    this: GenericBuilder<"hasName">,
    email: string
  ): GenericBuilder<"complete"> {
    (this as GenericBuilder<BuilderState> & { email?: string }).email = email;
    return this as unknown as GenericBuilder<"complete">;
  }

  build(this: GenericBuilder<"complete">): { name: string; email: string } {
    return {
      name: (this as GenericBuilder<BuilderState> & { name: string }).name,
      email: (this as GenericBuilder<BuilderState> & { email: string }).email,
    };
  }
}

// const b1 = new GenericBuilder().build();
// ❌ this: GenericBuilder<"complete"> expected, got GenericBuilder<"empty">

// 4. Immutable Builder
class ImmutableBuilder<T> {
  constructor(private readonly data: Readonly<Partial<T>> = {}) {}

  with<K extends keyof T>(key: K, value: T[K]): ImmutableBuilder<T> {
    return new ImmutableBuilder({ ...this.data, [key]: value });
  }

  build(): T {
    return this.data as T;
  }
}

interface AppConfig {
  apiUrl: string;
  timeout: number;
}

const builder = new ImmutableBuilder<AppConfig>();
const b1 = builder.with("apiUrl", "https://api.example.com");
const b2 = b1.with("timeout", 5000);
// builder, b1, b2 — alohida immutable instance'lar
```

</details>

---

## Composition vs Inheritance

### Nazariya

OOP'da code reuse'ning ikki asosiy yo'li — **inheritance** (`extends`) va **composition** (object ichida boshqa object'larni ishlatish).

**Inheritance (Is-A):**

```
Dog extends Animal  → "Dog IS AN Animal"
```

Class parent'dan method va property'lar meros oladi. Cheklovlar:

- Faqat **bitta** class'dan extends
- **Tight coupling** — parent o'zgarsa, child buziladi
- **Fragile base class problem** — parent internal implementation'ga bog'liqlik

**Composition (Has-A):**

```
Car has Engine    → "Car HAS AN Engine"
Car has Wheels    → "Car HAS Wheels"
```

Class boshqa object'larni property sifatida saqlaydi va ularning method'larini chaqiradi. Afzalliklari:

- **Flexible** — runtime'da component almashtirish mumkin (strategy pattern)
- **Loose coupling** — component interface orqali bog'lanadi
- **Multiple behavior** — bir nechta component birlashtiriladi

**"Favor composition over inheritance":** OOP'ning klassik qoidasi. Amalda aksariyat hollarda composition yaxshiroq.

**Qachon inheritance kerak:**

- Aniq "Is-A" munosabat bor (Dog **is** Animal)
- Shared implementation kerak (template method pattern)
- Runtime polymorphism kerak (base class type'da qabul qilish, subclass method chaqirish)

**Qachon composition kerak:**

- Behavior runtime'da o'zgarishi kerak (strategy)
- Multiple behavior birlashtirish kerak
- Deep hierarchy (3+ level) — inheritance bilan murakkab, composition flat
- Test'lar uchun mock kerak (DI bilan)

**Dependency Injection (DI) va composition:** DI — composition pattern'ning formal versiyasi. Object'lar o'z dependency'larini yaratmaydi — ular constructor yoki setter orqali inject qilinadi. Bu test doubles, environment-specific config, va modularity uchun muhim.

<details>
<summary><strong>Under the Hood</strong></summary>

**Runtime farqi:**

```
INHERITANCE:
Dog instance
  ├── dog.name         ← Dog prototype chain
  ├── dog.speak()      ← Dog.prototype (override)
  └── dog.move()       ← Animal.prototype (meros)
        prototype chain: Dog → Animal → Object

COMPOSITION:
Car instance
  ├── car.engine       ← Engine instance (alohida object)
  ├── car.transmission ← Transmission instance
  ├── car.start()      ← Car method: this.engine.ignite() chaqiradi
  └── prototype chain: Car → Object (flat)
```

Inheritance'da method'lar prototype chain orqali topiladi. Composition'da method chaqiruvlari delegation orqali — `this.component.method()`. Performance farqi amalda sezilmas.

**Favor composition pragmatic guide:**

1. **Boshlang'ich paytda** — composition tanlang (flexibility)
2. **Agar aniq Is-A bo'lsa** — inheritance ham OK
3. **Agar parent class uchta darajadan chuqur bo'lsa** — composition'ga o'ting
4. **Agar runtime polymorphism kerak bo'lsa** — inheritance yoki interface + composition
5. **Test kerak bo'lsa** — DI + composition (mock'lar oson)

**Fragile base class problem:** Inheritance'da parent class o'zgarishi child'larni buzishi mumkin. Masalan, parent `save()` method'ining internal logikasi o'zgarsa, child'larning `override save()` kutilmagan holatda ishlashi mumkin. Bu inheritance'ning klassik muammosi.

Composition bu muammodan xalos — component interface orqali bog'lanadi, implementation o'zgarishi interface saqlansa, consumer'lar ishlaydi.

**DI container'lar:** Zamonaviy framework'lar (NestJS, Angular, Spring) DI container'lar ishlatadi. Container'lar object lifecycle, dependency resolution, scope management'ni avtomatlashtiradi. Container'siz ham DI ishlaydi — oddiy constructor injection yetarli.

**Composition + interface:** Composition pattern'ning to'liq foydasi interface'lar bilan namoyon bo'ladi. Component interface'ga bog'langan, concrete implementation'ga emas. Runtime'da interface'ga mos har qanday implementation inject qilish mumkin:

```typescript
interface Logger { log(msg: string): void; }

class App {
  constructor(private logger: Logger) {}
  // App qaysi logger bo'lishini bilmaydi — ConsoleLogger, FileLogger, MockLogger
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Inheritance yomon ishlatilganda
class Animal {
  constructor(public name: string) {}
  move(): string { return `${this.name} moves`; }
}

class FlyingAnimal extends Animal {
  fly(): string { return `${this.name} flies`; }
}

class SwimmingAnimal extends Animal {
  swim(): string { return `${this.name} swims`; }
}

// ❌ Duck uchadi va suzadi — ikkalasidan extends mumkin emas
// class Duck extends FlyingAnimal, SwimmingAnimal {} // ❌

// 2. Composition bilan yechim
interface CanFly {
  fly(): string;
}

interface CanSwim {
  swim(): string;
}

interface CanWalk {
  walk(): string;
}

class FlightAbility implements CanFly {
  constructor(private ownerName: string) {}
  fly(): string { return `${this.ownerName} flies`; }
}

class SwimAbility implements CanSwim {
  constructor(private ownerName: string) {}
  swim(): string { return `${this.ownerName} swims`; }
}

class WalkAbility implements CanWalk {
  constructor(private ownerName: string) {}
  walk(): string { return `${this.ownerName} walks`; }
}

class Duck implements CanFly, CanSwim, CanWalk {
  private flightAbility: FlightAbility;
  private swimAbility: SwimAbility;
  private walkAbility: WalkAbility;

  constructor(public name: string) {
    this.flightAbility = new FlightAbility(name);
    this.swimAbility = new SwimAbility(name);
    this.walkAbility = new WalkAbility(name);
  }

  fly(): string { return this.flightAbility.fly(); }
  swim(): string { return this.swimAbility.swim(); }
  walk(): string { return this.walkAbility.walk(); }
}

class Penguin implements CanSwim, CanWalk {
  private swimAbility: SwimAbility;
  private walkAbility: WalkAbility;

  constructor(public name: string) {
    this.swimAbility = new SwimAbility(name);
    this.walkAbility = new WalkAbility(name);
  }

  swim(): string { return this.swimAbility.swim(); }
  walk(): string { return this.walkAbility.walk(); }
}

// 3. Strategy pattern — runtime behavior switching
interface SortStrategy<T> {
  sort(data: T[]): T[];
  readonly name: string;
}

class BubbleSort<T> implements SortStrategy<T> {
  name = "BubbleSort";
  sort(data: T[]): T[] {
    const arr = [...data];
    for (let i = 0; i < arr.length; i++) {
      for (let j = 0; j < arr.length - i - 1; j++) {
        if (arr[j] > arr[j + 1]) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    return arr;
  }
}

class QuickSort<T> implements SortStrategy<T> {
  name = "QuickSort";
  sort(data: T[]): T[] {
    const arr = [...data];
    if (arr.length <= 1) return arr;
    const pivot = arr[0];
    const left = arr.slice(1).filter(x => x <= pivot);
    const right = arr.slice(1).filter(x => x > pivot);
    return [...this.sort(left), pivot, ...this.sort(right)];
  }
}

class Sorter<T> {
  constructor(private strategy: SortStrategy<T>) {}

  setStrategy(strategy: SortStrategy<T>): void {
    this.strategy = strategy;
  }

  sort(data: T[]): T[] {
    return this.strategy.sort(data);
  }
}

const sorter = new Sorter<number>(new BubbleSort());
sorter.sort([3, 1, 2]); // BubbleSort

sorter.setStrategy(new QuickSort());
sorter.sort([3, 1, 2]); // QuickSort — runtime switch

// 4. Dependency Injection pattern
interface ILogger {
  log(msg: string): void;
}

interface IUserRepo {
  findById(id: string): Promise<User | null>;
}

interface User {
  id: string;
  name: string;
}

class ConsoleLogger implements ILogger {
  log(msg: string): void { console.log(msg); }
}

class MockLogger implements ILogger {
  logs: string[] = [];
  log(msg: string): void { this.logs.push(msg); }
}

class DbUserRepo implements IUserRepo {
  async findById(id: string): Promise<User | null> {
    // DB query
    return { id, name: "Ali" };
  }
}

class MockUserRepo implements IUserRepo {
  async findById(id: string): Promise<User | null> {
    return { id, name: "Test" };
  }
}

class UserService {
  constructor(
    private readonly repo: IUserRepo,
    private readonly logger: ILogger
  ) {}

  async getUser(id: string): Promise<User | null> {
    this.logger.log(`Fetching user ${id}`);
    return this.repo.findById(id);
  }
}

// Production
const prodService = new UserService(new DbUserRepo(), new ConsoleLogger());

// Test
const mockLogger = new MockLogger();
const testService = new UserService(new MockUserRepo(), mockLogger);
// mockLogger.logs — assert qilish mumkin

// 5. Simple composition with delegation
class User {
  constructor(public name: string, public email: string) {}
}

class EmailService {
  send(to: string, subject: string): void {
    console.log(`Email to ${to}: ${subject}`);
  }
}

class NotificationService {
  constructor(private emailService: EmailService) {}

  notifyUser(user: User, message: string): void {
    this.emailService.send(user.email, message);
  }
}

const notifier = new NotificationService(new EmailService());
notifier.notifyUser(new User("Ali", "ali@test.com"), "Welcome!");
```

</details>

---

## Intersection Types va Classes

### Nazariya

Intersection type (`&`) — ikki yoki undan ko'p type'ni birlashtirib, barcha property'larga ega bo'lgan yangi type yaratadi. Class'lar kontekstida intersection type class instance type'ni kengaytirish uchun ishlatiladi — `extends` yoki `implements` ishlatmasdan.

**Qachon kerak:**

- Mavjud class type'ga qo'shimcha property qo'shish
- Ikki class instance type'ni birlashtirish
- Mixin result type'ni ifodalash
- Third-party class type'ni kengaytirish (source code'ga tegmasdan)

```typescript
class User {
  constructor(public name: string, public email: string) {}
}

type AdminUser = User & {
  role: "admin";
  permissions: string[];
};

const admin: AdminUser = {
  name: "Ali",
  email: "ali@test.com",
  role: "admin",
  permissions: ["read", "write"],
};
// ❗ admin instanceof User === false — object literal, new User() emas
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Intersection type — faqat compile-time:** Intersection type JavaScript'ga compile bo'lganda butunlay o'chiriladi. Runtime'da intersection type'dagi object — oddiy JavaScript object.

```
Type A = { name: string; age: number }
Type B = { email: string; role: string }

A & B = { name: string; age: number; email: string; role: string }
```

**Property conflict:** Agar ikki type'da bir xil nomli property turli type bilan bo'lsa, intersection `never`'ga aylanadi:

```typescript
type A = { value: string };
type B = { value: number };
type C = A & B;
// C["value"] = string & number = never
```

`string & number` = hech qanday qiymat ham string, ham number bo'la olmaydi → `never`.

**Class type + property intersection:** Class type'ni intersection bilan kengaytirish mumkin, lekin natija **class instance emas**:

```typescript
class User { constructor(public name: string) {} }
type ExtendedUser = User & { role: string };

const u: ExtendedUser = { name: "Ali", role: "admin" };
u instanceof User; // false — oddiy object
```

`instanceof` saqlanishi uchun `Object.assign(new User(), extras)` pattern kerak — lekin bu method binding muammolar keltiradi (quyida).

**`Object.assign` va class method'lar — nozik holat:** Class method'lar **prototype'da** yashaydi, instance'da emas. `Object.assign({}, instance)` faqat own property'larni ko'chiradi — prototype method'lar ko'chmaydi:

```typescript
class Logger {
  log(msg: string): void { console.log(msg); }
}

const instance = new Logger();
const copy = Object.assign({}, instance);
// copy.log — undefined! Method yo'q
```

**Yechim'lar:**

1. **Object.assign bilan `new` instance saqlash** — prototype chain saqlanadi:
   ```typescript
   const extended = Object.assign(new User("Ali"), { role: "admin" });
   extended instanceof User; // true
   ```

2. **Method'larni manual bind qilish:**
   ```typescript
   const copy = Object.assign({}, instance);
   copy.log = instance.log.bind(instance);
   ```

3. **Proxy pattern — dynamic delegation:**
   ```typescript
   const proxy = new Proxy({} as Logger, {
     get(_, prop) { return (instance as any)[prop]?.bind(instance); }
   });
   ```

Proxy eng elegant, lekin har method chaqiruq'da bind cost bor.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Class type + metadata intersection
class BaseEntity {
  constructor(public id: number) {}
}

type WithTimestamps = {
  createdAt: Date;
  updatedAt: Date;
};

type WithSoftDelete = {
  deletedAt: Date | null;
  isDeleted: boolean;
};

type FullEntity = BaseEntity & WithTimestamps & WithSoftDelete;

function processEntity(entity: FullEntity): void {
  console.log(entity.id);        // BaseEntity
  console.log(entity.createdAt); // WithTimestamps
  console.log(entity.isDeleted); // WithSoftDelete
}

// Factory — instanceof saqlash uchun new bilan
function createFullEntity(id: number): FullEntity {
  const base = new BaseEntity(id);
  return Object.assign(base, {
    createdAt: new Date(),
    updatedAt: new Date(),
    deletedAt: null,
    isDeleted: false,
  });
}

const entity = createFullEntity(1);
entity instanceof BaseEntity; // true — prototype saqlangan

// 2. Ikki class instance birlashtirish (Proxy)
class Serializer {
  serialize(): string {
    return JSON.stringify(this);
  }
}

class Logger {
  log(message: string): void {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}

type SerializableLogger = Serializer & Logger;

// Proxy pattern — dynamic delegation
function createSerializableLogger(): SerializableLogger {
  const serializer = new Serializer();
  const logger = new Logger();

  return new Proxy({} as SerializableLogger, {
    get(_target, prop) {
      if (prop in logger) return (logger as any)[prop].bind(logger);
      if (prop in serializer) return (serializer as any)[prop].bind(serializer);
      return undefined;
    },
  });
}

const sl = createSerializableLogger();
sl.log("Hello");      // Logger'dan
sl.serialize();        // Serializer'dan

// 3. Intersection method conflict — never
type Getter1 = { get(): string };
type Getter2 = { get(): number };

type Combined = Getter1 & Getter2;
// Combined.get: (() => string) & (() => number)
// Implement qilish mumkin emas — return type conflict

// ✅ Yechim: nom'larni farqli qilish
type SafeGetter1 = { getString(): string };
type SafeGetter2 = { getNumber(): number };
type SafeCombined = SafeGetter1 & SafeGetter2;

class Impl implements SafeCombined {
  getString(): string { return "hello"; }
  getNumber(): number { return 42; }
}

// 4. Mixin result type — intersection bilan
class BaseClass {
  baseMethod(): string { return "base"; }
}

interface TimestampMixin {
  createdAt: Date;
  touch(): void;
}

// Mixin qaytaradigan type — intersection
type WithTimestamp<T> = T & TimestampMixin;

function addTimestamp<T extends BaseClass>(instance: T): WithTimestamp<T> {
  const timestamped = instance as WithTimestamp<T>;
  timestamped.createdAt = new Date();
  timestamped.touch = function() {
    this.createdAt = new Date();
  };
  return timestamped;
}

const base = new BaseClass();
const withTime = addTimestamp(base);
withTime.baseMethod();  // BaseClass'dan
withTime.touch();        // TimestampMixin'dan

// 5. Third-party class extension
// Third-party class (source code'ga tegmasdan)
class ExternalAPI {
  constructor(public endpoint: string) {}
  fetch(): Promise<unknown> { return Promise.resolve({}); }
}

// Extend qilish — intersection bilan
type ExtendedAPI = ExternalAPI & {
  cache: Map<string, unknown>;
  cachedFetch(): Promise<unknown>;
};

function enhance(api: ExternalAPI): ExtendedAPI {
  const cache = new Map<string, unknown>();
  return Object.assign(api, {
    cache,
    async cachedFetch(this: ExtendedAPI) {
      if (this.cache.has(this.endpoint)) {
        return this.cache.get(this.endpoint);
      }
      const result = await this.fetch();
      this.cache.set(this.endpoint, result);
      return result;
    },
  });
}
```

</details>

---

## `satisfies` bilan Class Property Validation

### Nazariya

`satisfies` operator (TypeScript 4.9+) — qiymatning ma'lum type'ga mos kelishini tekshiradi, lekin qiymatning **aniq (literal) type'ini** saqlab qoladi. Class kontekstida bu operator static property'lar yoki class bilan bog'liq config object'larni validatsiya qilish uchun ishlatiladi.

**`satisfies` vs type annotation farqi:**

```typescript
interface ThemeColors {
  primary: string;
  secondary: string;
  danger: string;
}

// Type annotation — literal type yo'qoladi
const colors1: ThemeColors = {
  primary: "#007bff",
  secondary: "#6c757d",
  danger: "#dc3545",
};
// colors1.primary: string (keng type)

// satisfies — literal saqlanadi
const colors2 = {
  primary: "#007bff",
  secondary: "#6c757d",
  danger: "#dc3545",
} satisfies ThemeColors;
// colors2.primary: "#007bff" (aniq literal)
```

Class'da `satisfies` asosan static property'lar, route table'lar, enum-like constant'lar uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Compile-time processing:** `satisfies` — butunlay compile-time construct. JavaScript'ga compile bo'lganda iz qolmaydi:

```typescript
// TS
const config = { port: 3000, host: "localhost" } satisfies ServerConfig;

// Compiled JS
const config = { port: 3000, host: "localhost" };
// satisfies — yo'q
```

**Ikki bosqichli qayta ishlash:**

1. **Validation** — object shape type'ga mos kelishini tekshiradi (ortiqcha property = xato, yetishmaydigan = xato)
2. **Type preservation** — object'ning literal type'ini saqlaydi (type annotation'dagi kabi kengaytirmaydi)

**`as const` bilan kombinatsiya:** `as const satisfies Type` — eng qudratli pattern. Object'ni `readonly` qiladi va literal type'larni saqlaydi, shu bilan birga type'ga mos kelishini tekshiradi:

```typescript
const routes = [
  { path: "/users", method: "GET" },
  { path: "/users", method: "POST" },
] as const satisfies readonly RouteConfig[];

// routes[0].path: "/users" (literal)
// routes[0].method: "GET" (literal)
// readonly array
```

**Class static property'lar:** `satisfies` class static property'larga qo'llanganda, literal type'lar autocomplete va type narrowing uchun juda foydali:

```typescript
class Status {
  static readonly VALUES = {
    pending: 1,
    active: 2,
    closed: 3,
  } satisfies Record<string, number>;
  // Status.VALUES.pending: 1 (literal, number emas)
}
```

Bu pattern enum'ga alternative — runtime'da oddiy object, compile-time'da literal type'lar.

**Excess property check:** `satisfies` fresh object literal'larda excess property'ni tekshiradi (type annotation kabi). Agar target type'da yo'q property qo'shilsa — compile error:

```typescript
type Config = { port: number };
// const c = { port: 3000, extra: true } satisfies Config; // ❌ 'extra' yo'q
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Class static config bilan satisfies
interface RouteConfig {
  path: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  handler: string;
}

class ApiRouter {
  static readonly routes = [
    { path: "/users", method: "GET", handler: "getUsers" },
    { path: "/users", method: "POST", handler: "createUser" },
    { path: "/users/:id", method: "PUT", handler: "updateUser" },
    { path: "/users/:id", method: "DELETE", handler: "deleteUser" },
  ] as const satisfies readonly RouteConfig[];

  // routes[0].method: "GET" (literal)
  static findRoute(path: string, method: RouteConfig["method"]): string | undefined {
    const route = ApiRouter.routes.find(r => r.path === path && r.method === method);
    return route?.handler;
  }
}

type FirstMethod = (typeof ApiRouter.routes)[0]["method"]; // "GET"

// 2. Command handler mapping
interface CommandHandler {
  execute(): void;
  undo(): void;
}

type CommandName = "copy" | "paste" | "cut" | "delete";

class Editor {
  private handlers = {
    copy: {
      execute: () => console.log("Copied"),
      undo: () => console.log("Undo copy"),
    },
    paste: {
      execute: () => console.log("Pasted"),
      undo: () => console.log("Undo paste"),
    },
    cut: {
      execute: () => console.log("Cut"),
      undo: () => console.log("Undo cut"),
    },
    delete: {
      execute: () => console.log("Deleted"),
      undo: () => console.log("Undo delete"),
    },
  } satisfies Record<CommandName, CommandHandler>;

  executeCommand(name: CommandName): void {
    this.handlers[name].execute();
  }
}

// 3. Enum-like constant
interface StatusInfo {
  label: string;
  color: string;
  priority: number;
}

class TaskManager {
  static readonly STATUS = {
    pending: { label: "Pending", color: "#ffc107", priority: 1 },
    active: { label: "Active", color: "#007bff", priority: 2 },
    completed: { label: "Completed", color: "#28a745", priority: 3 },
    failed: { label: "Failed", color: "#dc3545", priority: 4 },
  } satisfies Record<string, StatusInfo>;

  static getStatus(key: keyof typeof TaskManager.STATUS): StatusInfo {
    return TaskManager.STATUS[key];
  }
}

// TaskManager.STATUS.pending.color: "#ffc107" (literal)
type PendingColor = (typeof TaskManager.STATUS)["pending"]["color"];
// "#ffc107"

// 4. API endpoint definition
interface Endpoint {
  url: string;
  method: "GET" | "POST";
  body?: unknown;
}

class ApiClient {
  static readonly endpoints = {
    listUsers: { url: "/users", method: "GET" },
    createUser: { url: "/users", method: "POST" },
    getUser: { url: "/users/:id", method: "GET" },
    deleteUser: { url: "/users/:id", method: "POST" },
  } as const satisfies Record<string, Endpoint>;

  // Har endpoint literal type bilan saqlanadi
  static async call<K extends keyof typeof ApiClient.endpoints>(
    name: K
  ): Promise<unknown> {
    const ep = ApiClient.endpoints[name];
    return fetch(ep.url, { method: ep.method });
  }
}

// 5. Complex validation — nested structure
interface FormField {
  name: string;
  type: "text" | "email" | "number" | "password";
  required: boolean;
  validation?: {
    minLength?: number;
    maxLength?: number;
    pattern?: string;
  };
}

class UserForm {
  static readonly fields = [
    {
      name: "username",
      type: "text",
      required: true,
      validation: { minLength: 3, maxLength: 20 },
    },
    {
      name: "email",
      type: "email",
      required: true,
      validation: { pattern: "^[^@]+@[^@]+$" },
    },
    {
      name: "password",
      type: "password",
      required: true,
      validation: { minLength: 8 },
    },
    {
      name: "age",
      type: "number",
      required: false,
    },
  ] as const satisfies readonly FormField[];

  // fields[0].name: "username" (literal)
  // fields[0].type: "text" (literal)
}
```

</details>

---

## Edge Cases va Gotchas

### 1. `#field in obj` Class Scope Majburiy

`#field in obj` operator faqat class scope ichida ishlaydi — method body, static method, getter/setter, static block. Class tashqarisidagi function'da `#field` parse xato beradi:

```typescript
class Color {
  #red: number;
  #green: number;
  #blue: number;

  constructor(r: number, g: number, b: number) {
    this.#red = r;
    this.#green = g;
    this.#blue = b;
  }

  // ✅ Class scope — ishlaydi
  static isColor(obj: unknown): obj is Color {
    return typeof obj === "object" && obj !== null && #red in obj;
  }
}

// ❌ Class tashqarisida — SyntaxError
// function checkColor(obj: unknown): boolean {
//   return #red in obj; // Parser error
// }
```

**Sabab:** `#red` token — private name. Private name'lar class'ga bog'langan — ularni faqat class scope ichida reference qilish mumkin. Parser class scope'dan tashqari `#identifier` ko'rsa, sintaksik xato beradi.

**Yechim:** Brand check static method sifatida class ichida yozing va tashqi funksiyadan chaqiring:

```typescript
function validateColor(obj: unknown): void {
  if (!Color.isColor(obj)) {
    throw new Error("Not a Color");
  }
}
```

### 2. `this` Return Type va Shallow Clone Gotcha

`this` return type bilan clone method yozganda, `Object.create(Object.getPrototypeOf(this))` shallow — nested object'lar shared bo'ladi:

```typescript
class Entity {
  data: Record<string, unknown> = {};

  clone(): this {
    const Ctor = this.constructor as new () => this;
    const copy = new Ctor();
    copy.data = this.data; // ❌ Shallow — nested shared
    return copy;
  }
}

const e1 = new Entity();
e1.data.name = "Ali";

const e2 = e1.clone();
e2.data.name = "Vali"; // e1.data.name ham o'zgardi!
```

**Deep clone yechimi — `structuredClone` (modern):**

```typescript
class Entity {
  data: Record<string, unknown> = {};

  clone(): this {
    const Ctor = this.constructor as new () => this;
    const copy = new Ctor();
    copy.data = structuredClone(this.data); // ✅ Deep
    return copy;
  }
}
```

`structuredClone` — Node 17+, modern browser'lar. Function'lar, DOM element'lar, class instance'larni clone qilmaydi (structured data, DOM, collection'larni qo'llab-quvvatlaydi).

### 3. Mixin Order — Right-to-Left Prototype Chain

Mixin composition `Cacheable(Eventful(Base))` — right-to-left, ichkidan tashqariga. Natija prototype chain:

```
instance
  └── Cacheable.prototype    ← eng yuqori (oxirgi mixin)
        └── Eventful.prototype    ← o'rta
              └── Base.prototype      ← eng past (birinchi)
```

**Gotcha:** Agar ikki mixin bir xil method nomini qo'shsa, **eng tashqi** (oxirgi chaqirilgan) mixin yutadi:

```typescript
function A<T extends GConstructor>(Base: T) {
  return class extends Base {
    greet(): string { return "A"; }
  };
}

function B<T extends GConstructor>(Base: T) {
  return class extends Base {
    greet(): string { return "B"; }
  };
}

class Base {}

const AB = A(B(Base));  // A ustida
const instance = new AB();
instance.greet(); // "A" — eng tashqi yutadi

const BA = B(A(Base));  // B ustida
const instance2 = new BA();
instance2.greet(); // "B"
```

**Sabab:** Har mixin oldingi class'dan extends qiladi. Method resolution prototype chain orqali — eng yuqori prototype eng oldin topiladi.

**Yechim:** Mixin'lar mustaqil bo'lishi kerak — method nomi konfliktlari bo'lmasligi kerak. Agar override kerak bo'lsa, `super.greet()` bilan parent'ni chaqirish mumkin.

### 4. `Object.assign` Class Instance va Prototype Method'lar

Class method'lar **prototype'da** yashaydi, **instance'da** emas. `Object.assign({}, instance)` faqat own property'larni ko'chiradi — prototype method'lar ko'chmaydi:

```typescript
class Logger {
  level: string = "info";
  log(msg: string): void { console.log(`[${this.level}] ${msg}`); }
}

const original = new Logger();
const copy = Object.assign({}, original);

copy.level; // "info" — own property ko'chdi
// copy.log("test"); // ❌ TypeError: copy.log is not a function
```

**Yechim'lar:**

1. **`Object.assign` bilan class instance'ga:**
   ```typescript
   const copy = Object.assign(new Logger(), original);
   // copy.log — ishlaydi (new Logger() prototype saqlangan)
   ```

2. **`Object.create` bilan prototype saqlash:**
   ```typescript
   const copy = Object.create(
     Object.getPrototypeOf(original),
     Object.getOwnPropertyDescriptors(original)
   );
   ```

3. **Manual bind:**
   ```typescript
   const copy = Object.assign({}, original) as Logger;
   copy.log = original.log.bind(original);
   ```

Bu gotcha'ning sababi — JavaScript class syntax'ning prototype chain'ga asoslanganligi. Method'lar `Class.prototype`'da, instance'da faqat property'lar. Clone qilish uchun prototype saqlash kerak.

### 5. Intersection Method Signature Conflict → `never`

Ikki interface'da bir xil nomli method turli signature bilan bo'lsa, intersection method return type `string & number = never` ga aylanadi:

```typescript
type A = { getValue(): string };
type B = { getValue(): number };

type C = A & B;
// C.getValue: (() => string) & (() => number)
// Return type: string & number = never
// Implement qilib bo'lmaydi — return qiymat yo'q mumkin
```

**Nima uchun `never`:** `string & number` "string VA number bo'lgan qiymat" ma'nosini beradi. Lekin hech qaysi qiymat bir vaqtda ham string, ham number bo'la olmaydi → `never`.

**Yechim'lar:**

1. **Method nom'larini farqlash:**
   ```typescript
   type A = { getStringValue(): string };
   type B = { getNumberValue(): number };
   type C = A & B; // Conflict yo'q
   ```

2. **Union return type:**
   ```typescript
   type A = { getValue(): string | number };
   type B = { getValue(): string | number };
   // Ikkisi ham bir xil — mos keladi
   ```

3. **Overload'lar bilan:**
   ```typescript
   class Impl {
     getValue(): string;
     getValue(type: "number"): number;
     getValue(type?: "number"): string | number {
       return type === "number" ? 42 : "hello";
     }
   }
   ```

Intersection method conflict — structural typing'ning nozik chegarasi. Amaliyotda interface dizaynda bundan qochish kerak — ikki interface bir xil method nomini ishlatsa, signature'lar mos kelishi kerak.

---

## Common Mistakes

### ❌ Xato 1: Mixin constructor signature ni yo'qotish

```typescript
// ❌ Faqat no-arg constructor qabul qiladi
function Broken<T extends new () => any>(Base: T) {
  return class extends Base {
    extra = true;
  };
}

class User {
  constructor(public name: string) {}
}

// const Mixed = Broken(User); // ❌ User'da argument kerak
```

**✅ To'g'ri usul:**

```typescript
function Fixed<T extends new (...args: any[]) => any>(Base: T) {
  return class extends Base {
    extra = true;
  };
}

const Mixed = Fixed(User);
const u = new Mixed("Ali"); // ✅ Constructor argument ishlaydi
```

**Nima uchun:** `new (...args: any[])` har qanday argument count'ni qo'llab-quvvatlaydi. Mixin base class'ning parameter type'larini bilmaydi, shuning uchun umumiy pattern kerak.

---

### ❌ Xato 2: `private` ni runtime privacy deb o'ylash

```typescript
class Secret {
  private password: string = "qwerty";
}

const s = new Secret();
// TS compile error:
// s.password; // ❌

// Lekin runtime'da ochiq:
(s as any).password;  // "qwerty"
s["password"];         // "qwerty" (eski TS yoki JS fayldan)
```

**✅ To'g'ri usul:**

```typescript
class RealSecret {
  #password: string;
  constructor(pw: string) {
    this.#password = pw;
  }
}
```

**Nima uchun:** TS `private` faqat compile-time. Runtime privacy kerak bo'lsa — `#` ECMAScript native.

---

### ❌ Xato 3: `this` return type'ni yo'qotish

```typescript
class Base {
  // ❌ Aniq class nomi — subclass'da buziladi
  clone(): Base {
    return Object.assign(Object.create(this), this);
  }
}

class Child extends Base {
  extra = "data";
}

const child = new Child();
const cloned = child.clone();
// cloned: Base — Child emas!
// cloned.extra; // ❌ Property 'extra' does not exist on type 'Base'
```

**✅ To'g'ri usul:**

```typescript
class BaseFixed {
  clone(): this {
    return Object.assign(Object.create(Object.getPrototypeOf(this)), this);
  }
}

class ChildFixed extends BaseFixed {
  extra = "data";
}

const child = new ChildFixed();
const cloned = child.clone();
// cloned: ChildFixed ✅
cloned.extra; // ✅ accessible
```

**Nima uchun:** `this` return type subclass'da o'sha subclass'ga resolve bo'ladi. Aniq class nomi parent type'ga lock qiladi.

---

### ❌ Xato 4: Composition'da delegation method'ni yozmaslik

```typescript
interface CanFly {
  fly(): string;
}

class FlyAbility implements CanFly {
  fly(): string { return "flying"; }
}

// ❌ implements qo'yib, delegation yozmaslik
class Bird implements CanFly {
  private flyAbility = new FlyAbility();
  // ❌ Error: Property 'fly' is missing
}
```

**✅ To'g'ri usul:**

```typescript
class Bird implements CanFly {
  private flyAbility = new FlyAbility();

  fly(): string {
    return this.flyAbility.fly(); // ✅ Delegation
  }
}
```

**Nima uchun:** `implements` faqat contract tekshiradi — avtomatik delegation qo'shmaydi. Har method'ni manual yozish kerak.

---

### ❌ Xato 5: Intersection type method conflict

```typescript
type A = { getValue(): string };
type B = { getValue(): number };

type C = A & B;
// C["getValue"] = (() => string) & (() => number)
// Implement qilib bo'lmaydi

// class Impl implements C {
//   getValue(): ??? // Return type: string & number = never
// }
```

**✅ To'g'ri usullar:**

```typescript
// 1. Nom farqi
type SafeA = { getStringValue(): string };
type SafeB = { getNumberValue(): number };
type SafeC = SafeA & SafeB;

// 2. Overload
class Flexible {
  getValue(): string;
  getValue(type: "number"): number;
  getValue(type?: "number"): string | number {
    return type === "number" ? 42 : "hello";
  }
}
```

**Nima uchun:** `string & number = never` — disjoint type'lar. Return type conflict implement qilib bo'lmaydi.

---

## Amaliy Mashqlar

### Mashq 1: Mixin — Eventful + Cacheable (O'rta)

Ikki mixin yozing:
1. `Eventful` — `on(event, handler)`, `emit(event, ...args)` method'lari
2. `Cacheable` — `getCached(key)`, `setCache(key, value, ttlMs)` method'lari

Ikkalasini bitta `ApiService` class'ga qo'llang.

<details>
<summary>Javob</summary>

```typescript
type GConstructor<T = {}> = new (...args: any[]) => T;
type Handler = (...args: unknown[]) => void;

function Eventful<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    private listeners = new Map<string, Handler[]>();

    on(event: string, handler: Handler): void {
      const handlers = this.listeners.get(event) ?? [];
      handlers.push(handler);
      this.listeners.set(event, handlers);
    }

    emit(event: string, ...args: unknown[]): void {
      const handlers = this.listeners.get(event) ?? [];
      handlers.forEach(h => h(...args));
    }
  };
}

function Cacheable<TBase extends GConstructor>(Base: TBase) {
  return class extends Base {
    private cache = new Map<string, { value: unknown; expiresAt: number }>();

    getCached<T>(key: string): T | undefined {
      const entry = this.cache.get(key);
      if (!entry) return undefined;
      if (Date.now() > entry.expiresAt) {
        this.cache.delete(key);
        return undefined;
      }
      return entry.value as T;
    }

    setCache(key: string, value: unknown, ttlMs: number): void {
      this.cache.set(key, { value, expiresAt: Date.now() + ttlMs });
    }
  };
}

class BaseApiService {
  constructor(public baseUrl: string) {}
}

const EnhancedApiService = Cacheable(Eventful(BaseApiService));
const api = new EnhancedApiService("https://api.example.com");

api.on("request", (url) => console.log(`Requesting: ${url}`));
api.setCache("users", [{ id: 1, name: "Ali" }], 60000);
api.emit("request", "/users");
console.log(api.getCached("users"));
```

**Tushuntirish:**
- `Handler = (...args: unknown[]) => void` — callable signature, `Function` type emas
- `Cacheable(Eventful(BaseApiService))` — right-to-left, prototype chain: Cacheable → Eventful → BaseApiService
- Har mixin mustaqil behavior qo'shadi, bir-biriga bog'liq emas

</details>

---

### Mashq 2: Composition + DI — Logger Aggregator (O'rta)

`FileLogger`, `ConsoleLogger`, `HttpLogger` — uchta logger implementation yozing. `Application` class composition va dependency injection bilan bir nechta logger'ni birga ishlatsin.

<details>
<summary>Javob</summary>

```typescript
interface Logger {
  log(level: "info" | "warn" | "error", message: string): void;
}

class ConsoleLogger implements Logger {
  log(level: "info" | "warn" | "error", message: string): void {
    const prefix = `[${level.toUpperCase()}] ${new Date().toISOString()}`;
    if (level === "error") console.error(`${prefix} ${message}`);
    else if (level === "warn") console.warn(`${prefix} ${message}`);
    else console.log(`${prefix} ${message}`);
  }
}

class FileLogger implements Logger {
  constructor(private filePath: string) {}

  log(level: "info" | "warn" | "error", message: string): void {
    const line = `${new Date().toISOString()} [${level}] ${message}\n`;
    // fs.appendFileSync(this.filePath, line);
    console.log(`[FILE:${this.filePath}] ${line.trim()}`);
  }
}

class HttpLogger implements Logger {
  constructor(private endpoint: string) {}

  log(level: "info" | "warn" | "error", message: string): void {
    // fetch(this.endpoint, { method: "POST", body: JSON.stringify({ level, message }) });
    console.log(`[HTTP:${this.endpoint}] ${level}: ${message}`);
  }
}

class Application {
  constructor(private loggers: Logger[]) {}

  log(level: "info" | "warn" | "error", message: string): void {
    for (const logger of this.loggers) {
      logger.log(level, message);
    }
  }

  start(): void {
    this.log("info", "Application started");
  }

  handleError(err: Error): void {
    this.log("error", err.message);
  }
}

// Production DI
const app = new Application([
  new ConsoleLogger(),
  new FileLogger("/var/log/app.log"),
  new HttpLogger("https://logs.example.com/ingest"),
]);

app.start();
```

**Tushuntirish:**
- `Application` `Logger` interface'ga bog'langan, concrete implementation'ga emas
- Runtime'da logger qo'shish/olib tashlash oson (inheritance bilan murakkab)
- Test uchun `MockLogger` inject qilish mumkin — DI'ning asosiy foydasi

</details>

---

### Mashq 3: Type-Safe Email Builder (Qiyin)

`EmailMessage` uchun type-state builder yozing. `to` va `subject` — required, `cc`, `bcc`, `body`, `attachments` — optional. `build()` faqat `to` va `subject` set bo'lgandan keyin chaqirilishi mumkin.

<details>
<summary>Javob</summary>

```typescript
interface EmailMessage {
  to: string;
  subject: string;
  cc?: string[];
  bcc?: string[];
  body?: string;
  attachments?: string[];
}

interface NeedsTo {
  to(address: string): NeedsSubject;
}

interface NeedsSubject {
  subject(text: string): EmailOptionals;
}

interface EmailOptionals {
  cc(addresses: string[]): EmailOptionals;
  bcc(addresses: string[]): EmailOptionals;
  body(text: string): EmailOptionals;
  attachments(files: string[]): EmailOptionals;
  build(): EmailMessage;
}

class EmailBuilder implements NeedsTo {
  private data: Partial<EmailMessage> = {};

  to(address: string): NeedsSubject {
    this.data.to = address;
    return this as unknown as NeedsSubject;
  }

  subject(text: string): EmailOptionals {
    this.data.subject = text;
    return this as unknown as EmailOptionals;
  }

  cc(addresses: string[]): EmailOptionals {
    this.data.cc = addresses;
    return this as unknown as EmailOptionals;
  }

  bcc(addresses: string[]): EmailOptionals {
    this.data.bcc = addresses;
    return this as unknown as EmailOptionals;
  }

  body(text: string): EmailOptionals {
    this.data.body = text;
    return this as unknown as EmailOptionals;
  }

  attachments(files: string[]): EmailOptionals {
    this.data.attachments = files;
    return this as unknown as EmailOptionals;
  }

  build(): EmailMessage {
    return {
      to: this.data.to!,
      subject: this.data.subject!,
      ...(this.data.cc && { cc: this.data.cc }),
      ...(this.data.bcc && { bcc: this.data.bcc }),
      ...(this.data.body && { body: this.data.body }),
      ...(this.data.attachments && { attachments: this.data.attachments }),
    };
  }
}

const email = new EmailBuilder()
  .to("ali@test.com")
  .subject("Hello")
  .body("Hi Ali!")
  .cc(["bob@test.com"])
  .build();

// new EmailBuilder().build(); // ❌ NeedsTo'da build yo'q
// new EmailBuilder().to("a@b.com").build(); // ❌ NeedsSubject'da build yo'q
```

**Tushuntirish:**
- `NeedsTo` → `NeedsSubject` → `EmailOptionals` — step interfaces
- `build()` faqat `EmailOptionals`'da — `to` va `subject` set bo'lgandan keyin
- Runtime'da bitta object, type system har step'da turli interface ko'rsatadi
- Phantom types pattern — compile-time state machine

</details>

---

### Mashq 4: `#` Brand Check — Money Class (Qiyin)

`Money` class yozing — `#amount` va `#currency` private field'lar bilan. `add()` method faqat bir xil currency'dagi `Money` larni qo'shsin. Brand check bilan struktural impostor'larni rad eting.

<details>
<summary>Javob</summary>

```typescript
class Money {
  #amount: number;
  #currency: string;

  constructor(amount: number, currency: string) {
    if (amount < 0) throw new Error("Amount cannot be negative");
    this.#amount = amount;
    this.#currency = currency;
  }

  get amount(): number { return this.#amount; }
  get currency(): string { return this.#currency; }

  static isMoney(obj: unknown): obj is Money {
    return typeof obj === "object" && obj !== null && #amount in obj;
  }

  add(other: Money): Money {
    if (!Money.isMoney(other)) {
      throw new Error("Can only add Money instances");
    }
    if (this.#currency !== other.#currency) {
      throw new Error(`Currency mismatch: ${this.#currency} vs ${other.#currency}`);
    }
    return new Money(this.#amount + other.#amount, this.#currency);
  }

  subtract(other: Money): Money {
    if (!Money.isMoney(other)) {
      throw new Error("Can only subtract Money instances");
    }
    if (this.#currency !== other.#currency) {
      throw new Error(`Currency mismatch: ${this.#currency} vs ${other.#currency}`);
    }
    if (this.#amount < other.#amount) {
      throw new Error("Result cannot be negative");
    }
    return new Money(this.#amount - other.#amount, this.#currency);
  }

  equals(other: unknown): boolean {
    if (!Money.isMoney(other)) return false;
    return this.#amount === other.#amount && this.#currency === other.#currency;
  }

  toString(): string {
    return `${this.#amount} ${this.#currency}`;
  }
}

const usd1 = new Money(100, "USD");
const usd2 = new Money(50, "USD");
const eur = new Money(75, "EUR");

const sum = usd1.add(usd2);
console.log(sum.toString()); // "150 USD"

console.log(usd1.equals(new Money(100, "USD")));               // true
console.log(Money.isMoney(usd1));                                // true
console.log(Money.isMoney({ amount: 100, currency: "USD" }));    // false — brand yo'q

// usd1.add(eur); // Error: Currency mismatch
```

**Tushuntirish:**
- `#amount` va `#currency` — runtime private, himoyalangan
- `#amount in obj` — brand check, faqat haqiqiy Money instance'lar o'tadi
- Structural similarity (`{ amount, currency }`) yetarli emas
- `Money.isMoney()` — `instanceof`'dan kuchli (prototype manipulation bilan aldash mumkin emas)

</details>

---

### Mashq 5: Strategy Pattern — Pricing (Qiyin)

`Product` class, `PricingStrategy` interface, va uchta strategy (`RegularPricing`, `DiscountPricing`, `BulkPricing`) yozing. Runtime'da strategy'ni almashtirish mumkin bo'lsin.

<details>
<summary>Javob</summary>

```typescript
interface PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number;
  readonly name: string;
}

class RegularPricing implements PricingStrategy {
  name = "Regular";
  calculatePrice(basePrice: number, quantity: number): number {
    return basePrice * quantity;
  }
}

class DiscountPricing implements PricingStrategy {
  name = "Discount";
  constructor(private discountPercent: number) {
    if (discountPercent < 0 || discountPercent > 100) {
      throw new Error("Discount must be 0-100");
    }
  }

  calculatePrice(basePrice: number, quantity: number): number {
    const total = basePrice * quantity;
    return total * (1 - this.discountPercent / 100);
  }
}

class BulkPricing implements PricingStrategy {
  name = "Bulk";
  constructor(
    private threshold: number,
    private bulkDiscountPercent: number
  ) {}

  calculatePrice(basePrice: number, quantity: number): number {
    const total = basePrice * quantity;
    if (quantity >= this.threshold) {
      return total * (1 - this.bulkDiscountPercent / 100);
    }
    return total;
  }
}

class Product {
  constructor(
    public readonly name: string,
    public readonly basePrice: number,
    private strategy: PricingStrategy = new RegularPricing()
  ) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  getPrice(quantity: number): number {
    return this.strategy.calculatePrice(this.basePrice, quantity);
  }

  describe(): string {
    return `${this.name} ($${this.basePrice}) — Strategy: ${this.strategy.name}`;
  }
}

const product = new Product("Laptop", 1000);

// Regular pricing
console.log(product.getPrice(2)); // 2000

// Discount strategy
product.setStrategy(new DiscountPricing(10));
console.log(product.getPrice(2)); // 1800 (10% off)

// Bulk strategy
product.setStrategy(new BulkPricing(5, 20));
console.log(product.getPrice(3)); // 3000 (below threshold)
console.log(product.getPrice(6)); // 4800 (6 * 1000 * 0.8)
```

**Tushuntirish:**
- `Product` — `PricingStrategy` interface'ga bog'langan, concrete'ga emas
- Runtime'da `setStrategy()` bilan behavior almashtirish
- Har strategy mustaqil — alohida test qilish mumkin
- Inheritance bilan bu flexibility'ni olish murakkab — subclass compile-time'da belgilanadi

</details>

---

## Xulosa

Bu bo'limda TypeScript'dagi advanced OOP pattern'larni o'rgandik:

**Asosiy tushunchalar:**

- **Mixins** — single inheritance cheklovini chetlab o'tish, mixin function pattern bilan type-safe ko'p behavior. `GConstructor<T>` bilan constrained mixin'lar, right-to-left composition order
- **`#` Private Advanced** — brand check (`#field in obj`), class identity, security pattern'lar, library design. `instanceof`'dan kuchli — prototype manipulation bilan aldash mumkin emas
- **Class Expressions** — class qiymat sifatida: factory pattern, mixin pattern, conditional class creation
- **Multiple Interfaces** — class bir nechta contract'ni bajarish, method signature compatibility
- **Abstract Factory** — related object'lar oilasini yaratish, cross-platform/theme/environment pattern'lar
- **`this` Type Advanced** — type-state pattern, CRTP simulation, multi-level fluent chain
- **Builder Pattern Type-State** — phantom types bilan compile-time required field tracking
- **Composition vs Inheritance** — "Favor composition over inheritance", DI pattern, strategy pattern
- **Intersection Types va Classes** — class type'ni kengaytirish, method conflict → `never`, Proxy delegation
- **`satisfies`** — class property validation + literal type preservation

**Umumiy takeaway'lar:**

1. **Mixin constructor `new (...args: any[])`.** Majburiy pattern — base class parameter'larini qo'llab-quvvatlash uchun.
2. **`#` brand check class scope.** Only inside class, static methods included. Outside class — SyntaxError.
3. **Composition > Inheritance.** Flexibility, testability, DI — composition'ning asosiy afzalliklari.
4. **Type-state pattern**. Phantom types bilan compile-time state machine — builder'lar va DSL'lar uchun.
5. **`Object.assign` class instance.** Method'lar prototype'da, clone uchun `new Ctor()` yoki `Object.create(prototype)`.
6. **Intersection method conflict → `never`.** Signature mos kelmaslik implement qilib bo'lmaydi.
7. **`satisfies` + `as const`.** Literal type preservation + type check — config va enum-like constant'lar uchun.

**Cross-references:**

- **[06-type-narrowing.md](06-type-narrowing.md)** — Type guards (`is`, `asserts`), brand check narrowing
- **[07-functions.md](07-functions.md)** — Function type, `this` parameter, variance
- **[08-generics.md](08-generics.md)** — Generic class'lar, `keyof`, constraint
- **[09-advanced-generics.md](09-advanced-generics.md)** — Conditional types, phantom types
- **[10-classes.md](10-classes.md)** — Class asoslari: access modifier'lar, inheritance, method chaining
- **[15-utility-types.md](15-utility-types.md)** — `Pick`, `Omit`, `Partial` — builder'larda ishlatiladi
- **[19-decorators.md](19-decorators.md)** — TC39 decorators, accessor
- **[21-design-patterns.md](21-design-patterns.md)** — Design pattern'lar chuqur: Builder, Factory, Strategy, Observer
- **[23-type-safe-patterns.md](23-type-safe-patterns.md)** — Branded types, nominal typing, type-state
- **[25-type-compatibility.md](25-type-compatibility.md)** — Class assignability, variance

---

**Keyingi bo'lim:** [12-conditional-types.md](12-conditional-types.md) — Conditional Types chuqur: `T extends U ? X : Y` syntax, `infer` keyword bilan type extraction, distributive conditional types, recursive conditional types — TypeScript type system'ning eng qudratli va flexible qismi.
