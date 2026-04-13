# Bo'lim 10: Classes TypeScript'da

> TypeScript class — JavaScript class sintaksisi ustiga compile-time type annotation'lar, access modifier'lar (`public`, `private`, `protected`), `readonly`, parameter properties, abstract classes, `implements`, va `override` kabi type-safety mexanizmlarini qo'shadi. Runtime'da TypeScript'ning aksariyat class qo'shimchalari o'chiriladi (type erasure) — faqat ECMAScript native `#` private fields va decorator'lar runtime'da saqlanadi.
>
> Bu bo'limda TypeScript class'larning **pedagogik asoslarini** ko'ramiz: sintaksis va semantika, compile-time va runtime farqi, `private` vs `#` private, `implements` va structural typing, `this` type va method chaining. Mixins, class expressions, va advanced OOP pattern'lar — [11-advanced-oop.md](11-advanced-oop.md)'da.

---

## Mundarija

- [Class Anatomy — JS Class + TS Qo'shimchalar](#class-anatomy--js-class--ts-qoshimchalar)
- [Type Annotations — Properties, Constructor, Methods](#type-annotations--properties-constructor-methods)
- [Access Modifiers — public, private, protected](#access-modifiers--public-private-protected)
- [readonly Modifier](#readonly-modifier)
- [Parameter Properties — Constructor Shorthand](#parameter-properties--constructor-shorthand)
- [Static Members](#static-members)
- [Static Initialization Blocks](#static-initialization-blocks)
- [Abstract Classes](#abstract-classes)
- [Class Implements Interface](#class-implements-interface)
- [Class Extends Class — Inheritance](#class-extends-class--inheritance)
- [override Keyword](#override-keyword)
- [this Type — Method Chaining](#this-type--method-chaining)
- [Generics bilan Classes](#generics-bilan-classes)
- [accessor Keyword (TS 4.9+)](#accessor-keyword-ts-49)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Class Anatomy — JS Class + TS Qo'shimchalar

### Nazariya

TypeScript class — JavaScript ES6 class sintaksisining to'liq superset'i. JavaScript'da class yaratganingizda constructor, method'lar, getter/setter'lar, static member'lar ishlatiladi. TypeScript shu barcha narsalarni qabul qiladi va ustiga quyidagi type-safety qatlamlarini qo'shadi:

1. **Property declarations** — JavaScript'da class property'larini oldindan e'lon qilish shart emas, TypeScript'da barcha instance property'lar class body'da type bilan e'lon qilinishi kerak (`strictPropertyInitialization` yoqilganda)
2. **Type annotations** — har property, method parameter va return type'ga annotation yozish mumkin
3. **Access modifiers** — `public`, `private`, `protected` — compile-time visibility control
4. **`readonly`** — property'ni faqat constructor'da set qilinadigan qiladi
5. **Parameter properties** — constructor parametrida modifier yozish orqali avtomatik property yaratish
6. **`abstract`** — abstract class va abstract method'lar
7. **`implements`** — interface'ni implement qilish
8. **`override`** — parent method'ni qayta yozishni aniq belgilash

TypeScript class yaratganingizda **ikki** narsa hosil bo'ladi:

- **Instance type** — `new ClassName()` bilan yaratilgan object'ning type'i
- **Constructor function type** — class'ning o'zi (ya'ni `typeof ClassName`)

Bu ikki type'ni farqlash muhim — `User` deganda instance type tushuniladi, `typeof User` deganda constructor function.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori class'ni ikki bosqichda qayta ishlaydi:

1. **Type checking** — barcha type annotation'lar, access modifier'lar, `implements`, `override` tekshiriladi. Bu faqat compile-time'da sodir bo'ladi.
2. **Emit (JS generation)** — type annotation'lar va aksariyat modifier'lar o'chiriladi. Natija — oddiy JavaScript class.

```
TS Class
  │
  ├── Type Annotations     → O'chiriladi (type erasure)
  ├── Access Modifiers     → O'chiriladi (private, protected → JS'da yo'q)
  ├── readonly             → O'chiriladi (JS'da constraint yo'q)
  ├── Parameter Properties → Constructor ichiga assignment qo'shiladi
  ├── abstract             → O'chiriladi
  ├── implements           → O'chiriladi (faqat type check)
  ├── override             → O'chiriladi
  │
  └── JS Class             → Runtime'da ishlaydi
      ├── constructor()
      ├── methods
      ├── getters/setters
      ├── static members
      └── # private fields  → Runtime'da ham private (saqlanadi!)
```

**Instance type vs constructor type:** TypeScript class e'lon qilganingizda, kompilator avtomatik ravishda ikki narsani yaratadi:

```typescript
class User {
  constructor(public name: string) {}
}

// User — instance type
const u: User = new User("Ali");

// typeof User — constructor function type
const UserCtor: typeof User = User;
const u2 = new UserCtor("Vali");
```

`User` — instance (`{ name: string }`), `typeof User` — constructor (`new (name: string) => User`). Factory pattern, dependency injection, va generic constraint'larda bu farq muhim.

**Runtime'da qoladi:** Class structure, constructor, method'lar, static member'lar, `#` private field'lar, decorator'lar (target qo'llab-quvvatlasa), va property initializer'lar. Boshqa hamma narsa type erasure orqali o'chiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy class — TS qo'shimchalari bilan
class User {
  readonly id: number;
  name: string;
  private email: string;
  protected role: string;

  constructor(id: number, name: string, email: string) {
    this.id = id;          // readonly — faqat constructor'da
    this.name = name;
    this.email = email;
    this.role = "user";
  }

  getEmail(): string {
    return this.email;
  }

  updateName(newName: string): void {
    this.name = newName;
  }
}

// 2. Instance type vs constructor type
function createUser(Ctor: typeof User, id: number, name: string, email: string): User {
  return new Ctor(id, name, email);
}

const user = createUser(User, 1, "Ali", "ali@test.com");
// user: User (instance type)

// 3. Class nomi ham type, ham value
const u1: User = new User(2, "Vali", "vali@test.com");  // User — type
const Cls = User;                                        // User — value (constructor)
const u2 = new Cls(3, "Soli", "soli@test.com");

// 4. Method signature va return type
class Calculator {
  add(a: number, b: number): number {
    return a + b;
  }

  // Method chaining uchun this return
  log(message: string): this {
    console.log(message);
    return this;
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class User {
  readonly id: number;
  name: string;
  private email: string;
  protected role: string;

  constructor(id: number, name: string, email: string) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.role = "user";
  }

  getEmail(): string {
    return this.email;
  }
}
```

```javascript
// Compiled JS (ES2015+) — barcha type qo'shimchalari o'chirilgan
class User {
  constructor(id, name, email) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.role = "user";
  }

  getEmail() {
    return this.email;
  }
}
// ❗ private, protected, readonly, type annotations — hammasi yo'q
// Runtime'da this.email'ga tashqaridan kirishni hech narsa to'xtatmaydi
```

Kompile bo'lganda access modifier'lar va type annotation'lar butunlay yo'qoladi. Faqat native JavaScript class qoladi.

</details>

---

## Type Annotations — Properties, Constructor, Methods

### Nazariya

TypeScript class'da uch joyda type annotation yoziladi:

**1. Property declarations** — class body'da property'ni type bilan e'lon qilish. `strictPropertyInitialization: true` (strict mode'da default) bo'lganda, har non-optional property constructor'da initialize bo'lishi **shart**.

```typescript
class Product {
  name: string;           // MAJBURIY — constructor'da initialize
  description?: string;   // Optional
  price: number = 0;      // Default qiymat
}
```

**Initialization'ning yo'llari:**

- **Constructor'da assign** — eng to'g'ri yo'l
- **Default qiymat berish** — `name: string = ""`
- **Optional qilish** — `name?: string`
- **Definite assignment assertion** — `name!: string` — TypeScript'ga "ishon, bor" deyish (faqat framework'lar uchun, xavfli)

**2. Constructor parameters** — har parameter'ga type:

```typescript
class Order {
  items: string[];
  total: number;

  constructor(items: string[], total: number) {
    this.items = items;
    this.total = total;
  }
}
```

**3. Method signatures** — parameter va return type:

```typescript
class Service {
  fetch(url: string): Promise<unknown> {
    return fetch(url).then(r => r.json());
  }

  log(message: string, level: "info" | "warn" | "error"): void {
    console.log(`[${level}] ${message}`);
  }
}
```

**Definite assignment assertion (`!`) ishlatish** — faqat framework tomonidan (Angular, NestJS) initialize qilinadigan property'lar uchun. Boshqa joylarda ishlatish xavfli, chunki TypeScript'ga yolg'on gapirish.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript class'ning type'i — aslida **interface**'ga juda yaqin structure. Class yaratganingizda, kompilator avtomatik ravishda shu class'ga mos **instance type** yaratadi:

```typescript
class User {
  name: string;
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  greet(): string {
    return `Hi, ${this.name}`;
  }
}

// TypeScript ichida User type quyidagiga ekvivalent:
// interface User {
//   name: string;
//   age: number;
//   greet(): string;
// }
```

Shuning uchun TypeScript'da **structural typing** ishlaydi — class instance va oddiy object bir xil type'ga mos kelishi mumkin:

```typescript
function printName(obj: { name: string }) {
  console.log(obj.name);
}

const user = new User("Ali", 25);
printName(user);                    // ✅ User'da name: string bor
printName({ name: "Vali", age: 30 }); // ✅ oddiy object ham mos keladi
```

**`strictPropertyInitialization` mexanizmi:** Kompilator class'ning `constructor` body'sini scan qiladi va har non-optional property uchun quyidagi shartlardan birini tekshiradi:

1. Property declaration'da default qiymat bor
2. Constructor ichida `this.prop = ...` assignment bor
3. Property `?:` bilan optional qilingan
4. Property `!:` bilan definite assignment assertion qo'yilgan

Agar hech qaysi shart bajarilmasa — compile error.

**Definite assignment assertion xavfi:** `!` — "TypeScript'ga yolg'on gapirish". Runtime'da property `undefined` bo'lishi mumkin, lekin type `string` deb belgilangan. Agar assignment haqiqatan bo'lmasa, runtime'da `TypeError: Cannot read property of undefined` ko'rasiz.

```typescript
// ❌ Xavfli — unit test'da bu property init bo'lmaydi
class UserService {
  db!: Database;
}

const svc = new UserService();
svc.db.query("..."); // Runtime crash — db undefined
```

Framework'lar (Angular `@Input`, NestJS `@Inject`) property'larni constructor'dan keyin init qiladi — shuning uchun `!` ishlatilishi maqbul. Boshqa joylarda — constructor'da assign qiling.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. strictPropertyInitialization — barcha property'lar initialize bo'lishi kerak
class Product {
  name: string;
  price: number;
  description?: string;  // optional — init shart emas
  inStock: boolean = true; // default

  constructor(name: string, price: number) {
    this.name = name;
    this.price = price;
  }
}

// 2. Constructor overload (function overload pattern class'da)
class Greeting {
  message: string;

  constructor(name: string);
  constructor(name: string, greeting: string);
  constructor(name: string, greeting?: string) {
    this.message = greeting ? `${greeting}, ${name}` : `Hello, ${name}`;
  }
}

new Greeting("Ali");              // "Hello, Ali"
new Greeting("Ali", "Salom");      // "Salom, Ali"

// 3. Method return type inference vs explicit
class MathHelper {
  // Inference — return type number
  square(x: number) {
    return x * x;
  }

  // Explicit — public API uchun yaxshiroq
  distance(x1: number, y1: number, x2: number, y2: number): number {
    return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
  }
}

// 4. Definite assignment — framework context
class ApiService {
  // Framework (Angular, NestJS) inject qiladi
  baseUrl!: string;
}

// 5. ❌ Xavfli — boshqa joylarda ! ishlatish
class BadService {
  data!: string; // Runtime'da init bo'lmasa — crash
}

// ✅ Yaxshi — constructor'da init
class GoodService {
  data: string;
  constructor(data: string) {
    this.data = data;
  }
}

// 6. Optional va default parameter kombinatsiyasi
class Config {
  host: string;
  port: number;
  secure: boolean;

  constructor(host: string, port: number = 3000, secure: boolean = false) {
    this.host = host;
    this.port = port;
    this.secure = secure;
  }
}

new Config("localhost");              // port=3000, secure=false
new Config("localhost", 8080);        // port=8080, secure=false
new Config("localhost", 443, true);   // hammasi
```

</details>

---

## Access Modifiers — public, private, protected

### Nazariya

TypeScript'da uchta access modifier bor. Bular **faqat compile-time**'da ishlaydi — runtime (JavaScript)'da hech qanday ta'sir qilmaydi:

| Modifier | Class ichida | Subclass'da | Tashqarida |
|----------|--------------|-------------|------------|
| `public` (default) | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ❌ |
| `private` | ✅ | ❌ | ❌ |

**`public`** — default modifier. Yozmasangiz ham `public` bo'ladi. Hamma joydan kirish mumkin.

**`protected`** — faqat class ichida va uni extend qilgan subclass'larda kirish mumkin. Tashqaridan kirish compile error.

**`private`** — faqat o'sha class ichida. Subclass'da ham ko'rinmaydi.

```typescript
class BankAccount {
  public owner: string;
  protected accountNumber: string;
  private balance: number;

  constructor(owner: string, accountNumber: string, balance: number) {
    this.owner = owner;
    this.accountNumber = accountNumber;
    this.balance = balance;
  }

  deposit(amount: number): void {
    this.balance += amount;  // ✅ class ichida
  }
}

class SavingsAccount extends BankAccount {
  addInterest(rate: number): void {
    // this.balance *= (1 + rate); // ❌ private — subclass'da ham yo'q
    this.accountNumber;             // ✅ protected — subclass'ga ruxsat
  }
}

const account = new BankAccount("Ali", "ACC-1", 1000);
account.owner;         // ✅ public
// account.accountNumber; // ❌ protected
// account.balance;       // ❌ private
```

**Muhim:** TypeScript `private` **faqat compile-time** tekshiruv. Runtime'da property oddiy JavaScript property bo'lib qoladi. Haqiqiy runtime privacy uchun **ECMAScript `#` private field'lar** ishlatish kerak (pastda).

### `private` vs `#` Private Fields

TypeScript'da ikki turdagi privacy mavjud:

| Xususiyat | TS `private` | ECMAScript `#` |
|-----------|--------------|-----------------|
| Compile-time check | ✅ | ✅ |
| Runtime privacy | ❌ | ✅ |
| Bracket notation (`obj["x"]`) | Compile-time'da bloklangan (TS 4.3+) | Butunlay ishlamaydi |
| `as any` cast bypass | Mumkin | Mumkin emas |
| Subclass'dan ko'rinish | Ko'rinmaydi (type-level) | Ko'rinmaydi (runtime) |
| JavaScript output | Oddiy property | `#field` (ES2022+) yoki WeakMap (eski target) |

```typescript
class Secret {
  private tsPrivate: string = "ts-secret";     // TS-only
  #jsPrivate: string = "js-secret";             // Runtime private

  revealTs(): string { return this.tsPrivate; }
  revealJs(): string { return this.#jsPrivate; }
}

const s = new Secret();

// TS private:
// s.tsPrivate;              // ❌ Compile error
// s["tsPrivate"];           // ❌ Compile error (TS 4.3+)
(s as any).tsPrivate;        // ✅ "ts-secret" — runtime'da ochiq
// Compiled JS'da bu oddiy property

// ECMAScript #:
// s.#jsPrivate;             // ❌ SyntaxError (class tashqarisida)
// s["#jsPrivate"];          // undefined — boshqa property
// (s as any).#jsPrivate;    // ❌ SyntaxError
// ECMAScript tomonidan enforce qilinadi
```

**Qachon qaysi biri?**

- **`#` private** — security kerak bo'lganda (token'lar, secrets, internal state)
- **TS `private`** — API design niyati bilan, performance muhim bo'lsa (`#` biroz sekinroq)

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript `private` va `protected` **faqat type-checker**'da ishlaydi. Compiled JavaScript'da bu modifier'lar butunlay yo'qoladi:

```typescript
// TS
class Secret {
  private password: string = "12345";
}

// Compiled JS — private yo'q
class Secret {
  constructor() {
    this.password = "12345";
  }
}
```

**Bracket notation bloklash (TS 4.3+):** TypeScript 4.3'dan boshlab `obj["privateField"]` kabi bracket notation orqali `private` field'ga kirish **compile-time'da** error beradi:

```typescript
const s = new Secret();
// s["password"]; // ❌ Error: Property 'password' is private and only accessible within class 'Secret'
```

Lekin `as any` cast orqali bypass qilish hali ham mumkin:

```typescript
(s as any).password; // ✅ Compile OK, runtime "12345"
```

JavaScript fayldan TypeScript class'ga kirilsa, hech qanday tekshiruv yo'q — JS type checker'ni bilmaydi.

**ECMAScript `#` private mexanizmi:** `#field` sintaksisi ES2022'da standardlashtirilgan. ECMAScript engine runtime'da enforce qiladi — class tashqarisida `#field`'ga kirish **sintaksik xato** (parser bloklidi). Bracket notation ham ishlamaydi, chunki `#` oddiy identifier emas, balki private name.

**Compiled output (target'ga qarab):**

- **ES2022+ target** — `#field` native saqlanadi
- **Eski target (ES2015-ES2021)** — TypeScript `WeakMap` bilan emulate qiladi (chunki `#` ES2015'da yo'q)

```typescript
// TS
class Counter {
  #count = 0;
  increment() { this.#count++; }
}

// ES2022+ output — native
class Counter {
  #count = 0;
  increment() { this.#count++; }
}

// ES2015 output — WeakMap emulation (approximate)
// Real TypeScript emit biroz murakkab
```

**Performance farqi:** `#` private bilan property access biroz sekinroq bo'lishi mumkin — chunki runtime check qiladi. Lekin modern engine'larda optimization bor, aksariyat use case'larda farq sezilmaydi.

**Encapsulation niyati:** TypeScript `private` — "public API'da bu property'ni ko'rsatma" degan signal. `#` — "runtime'da ham bu property'ga kirib bo'lmaydi" degan kafolat. Agar siz library yozayotgan bo'lsangiz va foydalanuvchilar JavaScript'dan sizning class'lardan foydalansa — `#` kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Access modifier'lar bilan real service
class AuthService {
  private secretKey: string;
  private tokenStore: Map<string, string> = new Map();
  protected maxAttempts: number = 5;

  constructor(secretKey: string) {
    this.secretKey = secretKey;
  }

  // Public API — tashqaridan foydalanish
  login(username: string, password: string): string | null {
    if (!this.validateCredentials(username, password)) {
      return null;
    }
    const token = this.generateToken(username);
    this.tokenStore.set(username, token);
    return token;
  }

  // Private — implementation detail
  private validateCredentials(username: string, password: string): boolean {
    return username.length > 0 && password.length > 0;
  }

  private generateToken(username: string): string {
    return `${username}_${this.secretKey}_${Date.now()}`;
  }

  // Protected — subclass'lar o'zgartirishi mumkin
  protected onLoginSuccess(username: string): void {
    console.log(`Login: ${username}`);
  }
}

class AuditedAuth extends AuthService {
  protected onLoginSuccess(username: string): void {
    super.onLoginSuccess(username);
    // Audit log'ga qo'shish
  }
}

// 2. # private fields — haqiqiy runtime privacy
class TokenVault {
  #tokens = new Map<string, string>();

  store(key: string, token: string): void {
    this.#tokens.set(key, token);
  }

  retrieve(key: string): string | undefined {
    return this.#tokens.get(key);
  }
}

const vault = new TokenVault();
vault.store("api", "secret-token");
// vault.#tokens;           // ❌ SyntaxError
// (vault as any).#tokens;  // ❌ SyntaxError
// vault["#tokens"];        // undefined (boshqa property)

// 3. Kombinatsiya — TS private + # private
class UserService {
  #internalState = { lastAccess: 0 };  // Runtime private
  private logger: Logger;                // TS private — API design

  constructor(logger: Logger) {
    this.logger = logger;
  }

  getLastAccess(): number {
    this.#internalState.lastAccess = Date.now();
    return this.#internalState.lastAccess;
  }
}

interface Logger {
  log(msg: string): void;
}

// 4. Tashqaridan kirish farqi
class Example {
  public a = 1;
  private b = 2;
  #c = 3;
}

const e = new Example();
e.a;              // ✅ 1
// e.b;           // ❌ Compile error
(e as any).b;     // ✅ 2 (runtime'da ochiq)
// e.#c;          // ❌ SyntaxError (class tashqarisida)
// e["#c"];       // undefined (runtime'da enforce)

// 5. Private constructor — singleton pattern
class Database {
  private static instance: Database | null = null;

  private constructor(public readonly url: string) {}

  static getInstance(url: string): Database {
    if (!Database.instance) {
      Database.instance = new Database(url);
    }
    return Database.instance;
  }
}

// new Database("postgres://..."); // ❌ private constructor
Database.getInstance("postgres://..."); // ✅ singleton
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class Secret {
  private tsPrivate = "a";
  #jsPrivate = "b";

  reveal(): string {
    return `${this.tsPrivate}-${this.#jsPrivate}`;
  }
}
```

```javascript
// Compiled JS (ES2022+ target) — # native
class Secret {
  tsPrivate = "a";    // private yo'q
  #jsPrivate = "b";   // # saqlanadi

  reveal() {
    return `${this.tsPrivate}-${this.#jsPrivate}`;
  }
}
```

`tsPrivate` oddiy property — tashqaridan kirib bo'ladi. `#jsPrivate` runtime'da enforce qilinadi — class tashqarisida kirish SyntaxError.

</details>

---

## readonly Modifier

### Nazariya

`readonly` modifier — property'ni faqat **declaration** yoki **constructor**'da assign qilish mumkin bo'lgan holga keltiradi. Keyinchalik qayta assign qilishga urinish compile error beradi.

```typescript
class Config {
  readonly apiUrl: string;
  readonly maxRetries: number = 3;  // declaration default

  constructor(apiUrl: string) {
    this.apiUrl = apiUrl;            // ✅ constructor'da assign
  }

  updateUrl(newUrl: string): void {
    // this.apiUrl = newUrl;          // ❌ Error: Cannot assign to 'apiUrl'
  }
}

const config = new Config("https://api.example.com");
config.apiUrl;            // ✅ o'qish mumkin
// config.apiUrl = "new"; // ❌ tashqaridan ham assign mumkin emas
```

**`readonly` — shallow.** Agar property object yoki array bo'lsa — ichidagi qiymatlarni o'zgartirish mumkin:

```typescript
class Team {
  readonly members: string[] = [];

  addMember(name: string): void {
    this.members.push(name);   // ✅ ichki mutatsiya
    // this.members = ["new"]; // ❌ qayta assign mumkin emas
  }
}
```

Deep immutability uchun `Readonly<T>` utility type yoki `as const` ishlatish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

`readonly` modifier **faqat compile-time**'da ishlaydi. Compiled JavaScript'da butunlay o'chiriladi — runtime'da hech qanday himoya yo'q:

```typescript
// TS
class Point {
  readonly x: number;
  readonly y: number;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }
}

// Compiled JS
class Point {
  constructor(x, y) {
    this.x = x;  // readonly yo'q
    this.y = y;
  }
}
```

Runtime'da `(p as any).x = 100` yoki JavaScript fayldan o'zgartirish mumkin.

**`readonly` vs `const`:** `const` — o'zgaruvchi binding'ni qayta assign qila olmaydi. `readonly` — property'ni qayta assign qila olmaydi. Ikkalasi ham **shallow** — ichidagi object/array mutable.

```typescript
const arr = [1, 2, 3];
arr.push(4);     // ✅ — array mutable
// arr = [];     // ❌ — const

class X { readonly list = [1, 2, 3]; }
const x = new X();
x.list.push(4);  // ✅ — array mutable
// x.list = [];  // ❌ — readonly
```

**Deep immutability usullari:**

1. **`Readonly<T>` utility** — type-level shallow readonly
2. **`as const`** — literal types + readonly recursively
3. **`Object.freeze()`** — runtime shallow freeze
4. **`DeepReadonly<T>` custom** — recursive type ([09-advanced-generics.md](09-advanced-generics.md))

Runtime immutability kerak bo'lsa, `Object.freeze()` ishlatish mumkin, lekin u ham shallow. Deep freeze custom recursive funksiya kerak.

**`readonly` va `ReadonlyArray<T>`:** Array property'larda `readonly T[]` yoki `ReadonlyArray<T>` ishlatib, ichki mutatsiya'ni bloklash mumkin:

```typescript
class Team {
  readonly members: readonly string[] = [];

  addMember(name: string): Team {
    // this.members.push(name); // ❌ ReadonlyArray'da push yo'q
    return new Team(); // yangi instance
  }
}
```

Bu pattern immutable data structure'lar uchun ishlatiladi — har mutation yangi instance yaratadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Immutable config with readonly + access modifier
class DatabaseConfig {
  constructor(
    public readonly host: string,
    public readonly port: number,
    private readonly password: string,
    public readonly maxConnections: number = 10
  ) {}

  getConnectionString(): string {
    return `${this.host}:${this.port}`;
  }
}

const config = new DatabaseConfig("localhost", 5432, "secret");
config.host;            // ✅ "localhost"
// config.host = "new"; // ❌ Error

// 2. readonly array — shallow
class Registry {
  readonly tags: string[] = [];

  addTag(tag: string): void {
    this.tags.push(tag);   // ✅ ichki mutatsiya OK
    // this.tags = [];      // ❌ qayta assign mumkin emas
  }
}

// 3. readonly T[] — ichki mutation ham bloklangan
class ImmutableTeam {
  readonly members: readonly string[] = [];

  addMember(name: string): ImmutableTeam {
    const team = new ImmutableTeam();
    // team.members = [...this.members, name]; // readonly
    return team;
  }
}

// 4. readonly bilan computed property
class Rectangle {
  constructor(
    public readonly width: number,
    public readonly height: number
  ) {}

  // Getter — computed, har chaqiriqda hisoblanadi
  get area(): number {
    return this.width * this.height;
  }
}

const r = new Rectangle(10, 20);
r.area; // 200
// r.width = 15; // ❌ readonly

// 5. Deep readonly bilan
class Settings {
  readonly theme: Readonly<{ primary: string; secondary: string }> = {
    primary: "#000",
    secondary: "#fff",
  };
}

const s = new Settings();
// s.theme.primary = "#111"; // ❌ Readonly<T> ichki property'larni ham readonly qiladi
// s.theme = {};              // ❌ outer readonly

// 6. Object.freeze bilan runtime immutability
class FrozenConfig {
  readonly data: { name: string };

  constructor(name: string) {
    this.data = Object.freeze({ name });
  }
}

const fc = new FrozenConfig("Ali");
// fc.data.name = "Vali"; // strict mode: TypeError; non-strict: silently ignored
```

</details>

---

## Parameter Properties — Constructor Shorthand

### Nazariya

Parameter properties — TypeScript'da constructor parametrida **access modifier** yoki **`readonly`** yozish orqali avtomatik ravishda class property yaratish va assign qilishning qisqartmasi. Bu boilerplate kod'ni sezilarli kamaytiradi.

**Oddiy yo'l (batafsil):**

```typescript
class User {
  name: string;
  age: number;
  email: string;

  constructor(name: string, age: number, email: string) {
    this.name = name;
    this.age = age;
    this.email = email;
  }
}
```

**Parameter properties bilan (qisqa):**

```typescript
class User {
  constructor(
    public name: string,
    public age: number,
    private email: string
  ) {
    // TypeScript o'zi property yaratadi va assign qiladi
  }
}
```

Bu ikki kod **bir xil** JavaScript'ga compile bo'ladi. `public name: string`'ni ko'rganda TypeScript:

1. `name: string` property'ni class'da e'lon qiladi
2. `this.name = name` assignment'ni constructor ichiga qo'shadi

**Modifier shart:** Parameter property bo'lishi uchun `public`, `private`, `protected`, yoki `readonly` (yoki ularning kombinatsiyasi) yozilishi kerak. Modifier'siz parameter — oddiy constructor parametri, property bo'lmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator parameter property'ni quyidagicha transform qiladi:

```typescript
// TS — parameter properties
class Product {
  constructor(
    public readonly id: number,
    public name: string,
    private price: number
  ) {}
}

// Compiled JS
class Product {
  constructor(id, name, price) {
    this.id = id;
    this.name = name;
    this.price = price;
  }
}
```

**Jarayon:**

1. Har modifier'li parameter uchun — class body'da property declaration qo'shadi (type check uchun)
2. Constructor body'ning **boshida** (boshqa kod'dan oldin) `this.param = param` assignment qo'shadi
3. Modifier va type annotation'larni o'chiradi

**Muhim — tartib:** Parameter property assignment'lar constructor body'ning eng boshida bajariladi. Agar super call bo'lsa, super'dan keyin. Bu nozik chekiniq — qo'shimcha logic parameter property'larga tayanishi mumkin:

```typescript
class Logger {
  constructor(
    private prefix: string,
    private level: number
  ) {
    // Bu qator this.prefix va this.level assign bo'lgandan keyin ishlaydi
    console.log(`Logger created: ${this.prefix}`); // ✅ this.prefix mavjud
  }
}
```

**Declaration + assignment kombinatsiyasi:** Parameter property'larni oddiy property'lar bilan kombinatsiya qilish mumkin:

```typescript
class Service {
  private count: number = 0;  // oddiy property + default

  constructor(
    public readonly name: string,  // parameter property
    private config: Config          // parameter property
  ) {
    this.count = config.initialCount; // constructor'da extra logic
  }
}
```

**NestJS va dependency injection:** Parameter properties modern framework'larda juda keng tarqalgan. NestJS service'larda, Angular component'larda — deyarli har yerda:

```typescript
class UserController {
  constructor(
    private readonly userService: UserService,
    private readonly logger: Logger,
    private readonly cache: Cache
  ) {}
  // Barcha dependencies constructor injection + property bir kadrda
}
```

Bu pattern'ning foydasi — boilerplate kamayadi, dependency injection toza ko'rinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy parameter properties
class User {
  constructor(
    public name: string,
    public age: number,
    private email: string
  ) {}

  getEmail(): string {
    return this.email;
  }
}

const u = new User("Ali", 25, "ali@test.com");
u.name;  // ✅ "Ali"
// u.email; // ❌ private

// 2. readonly bilan immutable
class Point {
  constructor(
    public readonly x: number,
    public readonly y: number
  ) {}

  distanceTo(other: Point): number {
    return Math.sqrt((other.x - this.x) ** 2 + (other.y - this.y) ** 2);
  }
}

// 3. Aralash — modifier'siz parameter + parameter properties
class Validator {
  constructor(
    private readonly rules: ValidationRule[],
    defaultMessage: string // property emas — faqat constructor'da ishlatiladi
  ) {
    if (!defaultMessage) {
      throw new Error("Default message required");
    }
  }

  validate(value: unknown): boolean {
    return this.rules.every(rule => rule.check(value));
  }
}

interface ValidationRule {
  check(value: unknown): boolean;
}

// 4. NestJS style dependency injection
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService,
    private readonly logger: LoggerService
  ) {}

  async createUser(data: CreateUserDto): Promise<User> {
    const user = await this.userRepo.save(data);
    await this.emailService.sendWelcome(user.email);
    this.logger.info(`User created: ${user.name}`);
    return user;
  }
}

interface UserRepository { save(data: CreateUserDto): Promise<User>; }
interface EmailService { sendWelcome(email: string): Promise<void>; }
interface LoggerService { info(msg: string): void; }
interface CreateUserDto { name: string; email: string; }

// 5. Default qiymatli parameter property
class Config {
  constructor(
    public readonly host: string,
    public readonly port: number = 3000,
    public readonly secure: boolean = false
  ) {}
}

new Config("localhost");       // port=3000, secure=false
new Config("localhost", 443);  // secure=false
new Config("localhost", 443, true); // hammasi
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS — parameter properties
class Product {
  constructor(
    public readonly id: number,
    public name: string,
    private price: number
  ) {}
}
```

```javascript
// Compiled JS — parameter properties expanded
class Product {
  constructor(id, name, price) {
    this.id = id;
    this.name = name;
    this.price = price;
  }
}
// readonly va access modifier'lar yo'q
// TypeScript avtomatik assignment qo'shadi
```

</details>

---

## Static Members

### Nazariya

`static` keyword — property yoki method'ni instance'ga emas, class'ning **o'ziga** biriktiradi. Static member'lar `new` bilan yaratilgan object'da emas, class constructor function'da saqlanadi.

```typescript
class MathUtils {
  static readonly PI: number = 3.14159;

  static square(x: number): number {
    return x * x;
  }

  static clamp(value: number, min: number, max: number): number {
    return Math.max(min, Math.min(max, value));
  }
}

// Class nomi orqali chaqiriladi
MathUtils.PI;               // 3.14159
MathUtils.square(5);        // 25
MathUtils.clamp(15, 0, 10); // 10

// Instance orqali chaqirib bo'lmaydi
const m = new MathUtils();
// m.PI;     // ❌ Error — static member instance'da yo'q
// m.square; // ❌ Error
```

Static member'larda ham **access modifier** va `readonly` ishlatiladi — singleton pattern, config class'lar, utility class'lar uchun keng qo'llaniladi.

**Static member va `this`:** Static method ichida `this` class'ning **o'ziga** ishora qiladi (instance'ga emas):

```typescript
class Counter {
  static count: number = 0;

  static increment(): void {
    this.count++;  // this = Counter class
    // Counter.count++ bilan bir xil
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Static member'lar JavaScript'da class constructor function'ning property'si sifatida saqlanadi. ES2022'dan boshlab JavaScript'da ham static field'lar to'g'ridan-to'g'ri class body'da yoziladi.

```
Class Constructor Function (Counter)
  ├── prototype
  │   ├── constructor → Counter
  │   └── instanceMethod()    ← Instance methods (shared)
  │
  ├── count: 0                ← Static property (on class itself)
  └── increment()             ← Static method

Instance (new Counter())
  └── __proto__ → Counter.prototype (instance methods)
```

**Static member'lar prototype chain'da yo'q** — shuning uchun instance orqali kirib bo'lmaydi. Ular class constructor function'ga to'g'ridan-to'g'ri biriktirilgan.

**Static va inheritance:** Subclass parent'ning static member'larini meros oladi — kompilator buni prototype chain emas, constructor function chain orqali qiladi:

```typescript
class Base {
  static id: string = "base";
}

class Child extends Base {}

Child.id; // "base" — meros olindi
```

Lekin static method ichida `this` — chaqirilgan class'ga ishora qiladi (subclass bo'lsa subclass'ga), Base'ga emas. Bu "static polymorphism" pattern:

```typescript
class Base {
  static create(): Base {
    return new this(); // this — chaqirilgan class
  }
}

class Child extends Base {}

Base.create();  // Base instance
Child.create(); // Child instance — this = Child
```

**Static member'larda type parameter ishlatib bo'lmaydi:** Class'ning generic type parameter'i instance-level concept. Static method'lar class'ning o'ziga tegishli, shuning uchun generic T'ga kirolmaydi. Agar generic kerak bo'lsa, method-level generic yozish kerak:

```typescript
class Container<T> {
  // static default: T;  // ❌ Error
  // static create(value: T): Container<T> {}  // ❌ Error

  // Method-level generic ✅
  static create<U>(value: U): Container<U> {
    return new Container(value);
  }

  constructor(public value: T) {}
}
```

**Static va private constructor:** Singleton pattern uchun `private constructor` + static factory method ishlatiladi — tashqaridan `new` bilan instance yaratishni bloklaydi:

```typescript
class Database {
  private static instance: Database | null = null;
  private constructor(public readonly url: string) {}

  static getInstance(url: string): Database {
    if (!Database.instance) {
      Database.instance = new Database(url);
    }
    return Database.instance;
  }
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Utility class — faqat static method'lar
class DateUtils {
  private constructor() {} // instance yaratishni bloklash

  static formatDate(date: Date): string {
    return date.toISOString().split("T")[0];
  }

  static isWeekend(date: Date): boolean {
    const day = date.getDay();
    return day === 0 || day === 6;
  }

  static daysBetween(a: Date, b: Date): number {
    const diff = Math.abs(a.getTime() - b.getTime());
    return Math.floor(diff / (1000 * 60 * 60 * 24));
  }
}

DateUtils.formatDate(new Date()); // "YYYY-MM-DD"
DateUtils.isWeekend(new Date());  // true/false

// 2. Singleton pattern
class ConfigService {
  private static instance: ConfigService | null = null;
  static readonly VERSION: string = "2.0.0";

  private constructor(public readonly dbUrl: string) {}

  static getInstance(dbUrl: string): ConfigService {
    if (!ConfigService.instance) {
      ConfigService.instance = new ConfigService(dbUrl);
    }
    return ConfigService.instance;
  }
}

ConfigService.VERSION;                      // ✅
ConfigService.getInstance("postgres://.."); // ✅

// 3. Static counter — class-level state
class Entity {
  private static nextId: number = 1;
  public readonly id: number;

  constructor(public name: string) {
    this.id = Entity.nextId++;
  }
}

const e1 = new Entity("A"); // id = 1
const e2 = new Entity("B"); // id = 2
const e3 = new Entity("C"); // id = 3

// 4. Static factory with polymorphism
class Shape {
  constructor(public readonly type: string) {}

  static create(type: string): Shape {
    return new this(type); // this = qaysi class chaqirilsa
  }
}

class Circle extends Shape {
  constructor() {
    super("circle");
  }
}

const s = Shape.create("square");   // Shape instance
const c = Circle.create("circle");   // Circle instance (this = Circle)

// 5. Static method with class generics
class Container<T> {
  constructor(public value: T) {}

  // Method-level generic — class T'dan mustaqil
  static from<U>(value: U): Container<U> {
    return new Container(value);
  }
}

const strContainer = Container.from("hello");  // Container<string>
const numContainer = Container.from(42);       // Container<number>

// 6. Static initialization sequence
class AppRegistry {
  static readonly DEFAULT_TIMEOUT = 5000;
  static readonly DEFAULT_RETRIES = 3;

  private static services: Map<string, unknown> = new Map();

  static register(name: string, service: unknown): void {
    AppRegistry.services.set(name, service);
  }

  static get<T>(name: string): T | undefined {
    return AppRegistry.services.get(name) as T | undefined;
  }
}

AppRegistry.register("logger", console);
const logger = AppRegistry.get<typeof console>("logger");
```

</details>

---

## Static Initialization Blocks

### Nazariya

Static initialization block (ES2022, TS 4.4+) — class body ichida `static { ... }` blok. Bu blok class yuklanganda **bir marta** bajariladi. Murakkab static property'larni initialize qilish uchun ishlatiladi — oddiy assignment yetarli bo'lmagan hollarda.

```typescript
class Database {
  static connection: Connection;
  static isReady: boolean;

  static {
    try {
      Database.connection = createConnection(process.env.DB_URL!);
      Database.isReady = true;
    } catch {
      Database.connection = createFallbackConnection();
      Database.isReady = false;
    }
  }
}

interface Connection {}
declare function createConnection(url: string): Connection;
declare function createFallbackConnection(): Connection;
```

**Static block'ning afzalliklari:**

1. **Murakkab logic** — try/catch, if/else, loop ishlatish mumkin (oddiy field declaration'da mumkin emas)
2. **Private member'larga kirish** — static block class scope'da ishlaydi, `private` va `#` member'larga kirish mumkin
3. **Bir nechta block** — class'da bir nechta static block bo'lishi mumkin, tartib bo'yicha bajariladi

<details>
<summary><strong>Under the Hood</strong></summary>

Static block JavaScript'ga **native** ES2022 feature sifatida qo'shilgan. TypeScript'ning emit strategiyasi target'ga bog'liq:

- **ES2022+ target** — static block native saqlanadi
- **Eski target** — TypeScript IIFE yoki oddiy assignment pattern'ga compile qiladi (aniq emit target'ga bog'liq)

**Scope va access:** Static block class scope ichida ishlaydi — shuning uchun `private`, `protected`, `#` private member'larga kirish mumkin:

```typescript
class Config {
  static #secret: string;  // ES private static

  static {
    // Static block — #secret'ga kira oladi
    Config.#secret = process.env.SECRET ?? "default-secret";
  }

  static getSecret(): string {
    return Config.#secret;
  }
}
```

**Bir nechta static block tartibi:** Class'da bir nechta static block bo'lsa, ular **deklaratsiya tartibida** bajariladi. Har block'ning ichidagi logic previous block'lar state'iga bog'liq bo'lishi mumkin:

```typescript
class Registry {
  static items: string[];
  static version: number;

  static {
    // Birinchi block
    Registry.items = ["default"];
  }

  static {
    // Ikkinchi block — birinchi block'dan keyin
    Registry.version = Registry.items.length;
  }
}
```

**Qachon ishlaydi:** Static block class deklaratsiya bajarilganda — class file birinchi marta load qilinganda. `new Class()` chaqirmay ham static block bajariladi. Agar class hech qachon ishlatilmasa (tree-shaking), static block ham bajarilmaydi.

**`this` static block'da:** Static block ichida `this` — class'ning o'ziga ishora qiladi (`typeof ClassName`). Lekin static block'da `this` o'rniga class nomi ishlatish aniqlik uchun yaxshiroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy static block
class Registry {
  static items: Map<string, unknown>;

  static {
    Registry.items = new Map();
    Registry.items.set("default", {});
  }
}

// 2. Try/catch bilan initialization
class ApiClient {
  static baseUrl: string;
  static token: string | null;

  static {
    try {
      ApiClient.baseUrl = process.env.API_URL ?? "http://localhost:3000";
      ApiClient.token = process.env.API_TOKEN ?? null;
    } catch {
      ApiClient.baseUrl = "http://localhost:3000";
      ApiClient.token = null;
    }
  }
}

// 3. # private fields bilan
class SecureConfig {
  static #secret: string;
  static #apiKey: string;

  static {
    // # private fields — static block'da kirish mumkin
    SecureConfig.#secret = process.env.SECRET ?? "fallback";
    SecureConfig.#apiKey = SecureConfig.#secret.substring(0, 8);
  }

  static getApiKey(): string {
    return SecureConfig.#apiKey;
  }
}

// 4. Bir nechta static block — tartib muhim
class Pipeline {
  static stages: string[];
  static config: { totalStages: number };

  static {
    // 1-block
    Pipeline.stages = ["validate", "transform", "save"];
  }

  static {
    // 2-block — 1-block'dan keyin
    Pipeline.config = { totalStages: Pipeline.stages.length };
  }
}

// 5. Connection pool with complex initialization
class ConnectionPool {
  static #pool: Map<string, Connection> = new Map();
  static #maxSize: number;

  static {
    const env = process.env.NODE_ENV ?? "development";
    ConnectionPool.#maxSize = env === "production" ? 20 : 5;
  }

  static getConnection(name: string): Connection | undefined {
    return ConnectionPool.#pool.get(name);
  }

  static addConnection(name: string, conn: Connection): void {
    if (ConnectionPool.#pool.size >= ConnectionPool.#maxSize) {
      throw new Error("Pool limit reached");
    }
    ConnectionPool.#pool.set(name, conn);
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class Registry {
  static items: Map<string, unknown>;

  static {
    Registry.items = new Map();
    Registry.items.set("default", {});
  }
}
```

```javascript
// Compiled JS (target: ES2022+) — native static block saqlanadi
class Registry {
  static items;
  static {
    Registry.items = new Map();
    Registry.items.set("default", {});
  }
}
```

Eski target'larda TypeScript static block'ni IIFE yoki oddiy assignment pattern'ga transform qiladi — aniq emit version'ga bog'liq, lekin semantika saqlanadi.

</details>

---

## Abstract Classes

### Nazariya

Abstract class — to'g'ridan-to'g'ri `new` bilan instance yaratib bo'lmaydigan, faqat extend qilish uchun mo'ljallangan class. Abstract class ikki turdagi member'ga ega bo'lishi mumkin:

1. **Abstract member'lar** — faqat signature (implementation yo'q). Subclass **majburiy** implement qiladi
2. **Concrete member'lar** — oddiy method/property (implementation bor). Subclass meros oladi

```typescript
abstract class Shape {
  abstract area(): number;
  abstract perimeter(): number;
  abstract readonly name: string;

  // Concrete — implementation bor, subclass meros oladi
  describe(): string {
    return `${this.name}: area=${this.area()}, perimeter=${this.perimeter()}`;
  }
}

// Abstract class'dan instance yaratib bo'lmaydi
// new Shape(); // ❌ Error

class Circle extends Shape {
  readonly name = "Circle";

  constructor(private radius: number) {
    super();
  }

  area(): number {
    return Math.PI * this.radius ** 2;
  }

  perimeter(): number {
    return 2 * Math.PI * this.radius;
  }
}
```

**Abstract class vs Interface:**

| Xususiyat | Abstract Class | Interface |
|-----------|---------------|-----------|
| Concrete method | ✅ Bor | ❌ Yo'q |
| Constructor | ✅ Bor | ❌ Yo'q |
| Access modifiers | ✅ Bor | ❌ Yo'q |
| Runtime'da mavjud | ✅ JS class | ❌ O'chiriladi |
| Multiple inheritance | ❌ Faqat bitta | ✅ Bir nechta |
| `instanceof` | ✅ Ishlaydi | ❌ Mumkin emas |

Abstract class'ning asosiy afzalligi — **shared implementation** berishi mumkin (template method pattern).

<details>
<summary><strong>Under the Hood</strong></summary>

Abstract class JavaScript'ga **oddiy class** sifatida compile bo'ladi — `abstract` keyword butunlay o'chiriladi:

```typescript
// TS
abstract class Animal {
  abstract speak(): string;

  move(distance: number): void {
    console.log(`Moving ${distance}m`);
  }
}

// Compiled JS
class Animal {
  // speak() — yo'q (abstract edi, implementation ham yo'q edi)
  move(distance) {
    console.log(`Moving ${distance}m`);
  }
}
```

**Runtime'da `new Animal()` bloklanmaydi** — `abstract` faqat compile-time constraint. Agar runtime'da ham bloklanishi kerak bo'lsa, constructor'da shart tekshirish yoki factory pattern bilan ishlash kerak.

```typescript
abstract class Animal {
  constructor() {
    if (new.target === Animal) {
      throw new Error("Cannot instantiate abstract class");
    }
  }
}
```

Lekin TypeScript bunday bilan kam ishlatiladi — compile-time check ko'p holatlarda yetarli.

**Template method pattern:** Abstract class'ning eng kuchli use case — algoritm'ning umumiy qismini yozib, o'zgartirish mumkin bo'lgan joylarini abstract qilish:

```typescript
abstract class DataProcessor<T, R> {
  // Template method — umumiy algoritm
  process(input: T): R {
    const validated = this.validate(input);
    const transformed = this.transform(validated);
    return this.format(transformed);
  }

  // Abstract — subclass o'zi implement qiladi
  protected abstract validate(input: T): T;
  protected abstract transform(input: T): R;

  // Default implementation — override mumkin, lekin shart emas
  protected format(result: R): R {
    return result;
  }
}
```

Subclass'lar faqat abstract method'larni implement qiladi, umumiy algoritm parent'dan keladi. Bu kod takrorlanishini kamaytiradi va consistency ta'minlaydi.

**Abstract constructor pattern:** Abstract class'ni factory'da ishlatish uchun `new (...args: any[]) => T` pattern'i:

```typescript
function createShape<T extends Shape>(
  Ctor: new (...args: any[]) => T,
  ...args: any[]
): T {
  return new Ctor(...args);
}

// createShape(Shape); // ❌ Shape abstract
createShape(Circle, 5); // ✅ Circle concrete
```

Kompilator abstract class'ni `new (...args) => T` signature'ga mos kelmaydi deb belgilaydi — faqat concrete class'lar constructable.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy abstract class
abstract class Shape {
  abstract area(): number;
  abstract readonly name: string;

  describe(): string {
    return `${this.name}: area=${this.area()}`;
  }
}

class Circle extends Shape {
  readonly name = "Circle";
  constructor(private radius: number) { super(); }
  area(): number { return Math.PI * this.radius ** 2; }
}

class Square extends Shape {
  readonly name = "Square";
  constructor(private side: number) { super(); }
  area(): number { return this.side ** 2; }
}

// 2. Template method pattern
abstract class DataProcessor<T, R> {
  process(input: T): R {
    const validated = this.validate(input);
    const transformed = this.transform(validated);
    return this.format(transformed);
  }

  protected abstract validate(input: T): T;
  protected abstract transform(input: T): R;

  protected format(result: R): R {
    return result; // default, override optional
  }
}

class CsvProcessor extends DataProcessor<string, string[][]> {
  protected validate(input: string): string {
    if (!input.trim()) throw new Error("Empty CSV");
    return input.trim();
  }

  protected transform(input: string): string[][] {
    return input.split("\n").map(row => row.split(","));
  }
}

const processor = new CsvProcessor();
processor.process("a,b\nc,d"); // [["a","b"],["c","d"]]

// 3. Abstract class with constructor
abstract class Entity {
  constructor(
    public readonly id: string,
    public readonly createdAt: Date
  ) {}

  abstract save(): Promise<void>;
  abstract toJSON(): unknown;

  describe(): string {
    return `${this.constructor.name}(${this.id}) at ${this.createdAt.toISOString()}`;
  }
}

class User extends Entity {
  constructor(id: string, public name: string, public email: string) {
    super(id, new Date());
  }

  async save(): Promise<void> {
    // DB save
  }

  toJSON(): { id: string; name: string; email: string } {
    return { id: this.id, name: this.name, email: this.email };
  }
}

// 4. Abstract + generic
abstract class Repository<T extends { id: string }> {
  protected abstract store: Map<string, T>;

  save(item: T): void {
    this.store.set(item.id, item);
  }

  findById(id: string): T | undefined {
    return this.store.get(id);
  }

  abstract findByField<K extends keyof T>(key: K, value: T[K]): T[];
}

class UserRepo extends Repository<User> {
  protected store = new Map<string, User>();

  findByField<K extends keyof User>(key: K, value: User[K]): User[] {
    return Array.from(this.store.values()).filter(u => u[key] === value);
  }
}

// 5. Factory with abstract constructor
function createShape<T extends Shape>(
  Ctor: new (...args: any[]) => T,
  ...args: any[]
): T {
  return new Ctor(...args);
}

// createShape(Shape, 5);       // ❌ Shape abstract
createShape(Circle, 5);          // ✅ Circle concrete
createShape(Square, 10);         // ✅
```

</details>

---

## Class Implements Interface

### Nazariya

`implements` keyword — class'ning ma'lum interface (shape)'ga mos kelishini compile-time'da tekshiradi. Bu "contract" — class shu interface'dagi barcha property va method'larga ega bo'lishi **shart**.

```typescript
interface Serializable {
  serialize(): string;
  deserialize(data: string): void;
}

interface Loggable {
  log(message: string): void;
}

// Bir nechta interface implement qilish mumkin
class User implements Serializable, Loggable {
  constructor(public name: string, public age: number) {}

  serialize(): string {
    return JSON.stringify({ name: this.name, age: this.age });
  }

  deserialize(data: string): void {
    const parsed = JSON.parse(data);
    this.name = parsed.name;
    this.age = parsed.age;
  }

  log(message: string): void {
    console.log(`[User:${this.name}] ${message}`);
  }
}
```

**Muhim nuance — `implements` type inference bermaydi:** `implements` faqat class'ning **shape**'ini tekshiradi. Method parameter'lariga type'ni avtomatik bermaydi. `noImplicitAny: true` (strict mode'da default) bo'lsa, parameter'siz class method compile error beradi:

```typescript
interface Checkable {
  check(name: string): boolean;
}

class NameChecker implements Checkable {
  check(name) {  // ❌ noImplicitAny: Parameter 'name' implicitly has an 'any' type
    return name.toLowerCase() === "valid";
  }
}

// ✅ To'g'ri — explicit type
class NameCheckerFixed implements Checkable {
  check(name: string): boolean {
    return name.toLowerCase() === "valid";
  }
}
```

**Nima uchun TypeScript shunday:** Dizayn qarori. Class method'larning parameter'lari class'ning boshqa constraint'lari bilan bog'liq bo'lishi mumkin — `implements` ishonchli source emas. Explicit annotation yaxshiroq — method body ichida parameter'lar aniq bo'ladi.

### Structural Typing — `implements`'siz ham mos keladi

TypeScript **structural typing** ishlatadi — class `implements` yozmasdan ham interface'ga mos kelishi mumkin:

```typescript
interface HasName {
  name: string;
}

class Person {
  constructor(public name: string) {}
}

function greet(obj: HasName): void {
  console.log(obj.name);
}

greet(new Person("Ali")); // ✅ Structural — shape mos keladi
```

**`implements` yozishning foydalari:**

1. **Aniq contract** — developer kodni o'qiganda tushunadi
2. **Erta xato** — interface'ga mos kelmasa, darhol xato
3. **Refactoring safety** — interface o'zgarganda implement qilgan class'lar xato beradi

**Nominal typing:** TypeScript structural, lekin ba'zi paytlar nominal (nom bo'yicha) typing kerak bo'ladi — masalan, branded type'lar bilan. Bu [23-type-safe-patterns.md](23-type-safe-patterns.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

`implements` compiled JavaScript'da **butunlay o'chiriladi** — runtime'da hech qanday iz yo'q:

```typescript
// TS
interface Printable {
  print(): void;
}

class Document implements Printable {
  print(): void {
    console.log("Printing...");
  }
}

// Compiled JS
class Document {
  print() {
    console.log("Printing...");
  }
}
// implements Printable — butunlay yo'q
```

Shuning uchun `instanceof` bilan interface tekshirib bo'lmaydi:

```typescript
const doc = new Document();
// doc instanceof Printable; // ❌ 'Printable' only refers to a type
```

Interface faqat compile-time structural check.

**`implements` dizayn qarori:** Nima uchun `implements` type inference bermaydi? Ikki sabab:

1. **Bir nechta `implements`** — class bir nechta interface implement qilsa, qaysi birining type'ini olishi aniq emas
2. **Class'ning o'z constraint'lari** — class method'i interface'dan tashqari `override`, abstract, yoki boshqa constraint'lar bilan bog'liq bo'lishi mumkin

Dizayn qarori: explicit annotation har doim ishonchli. Developer interface'ga tayanmasdan, method'ga aniq type yozadi.

**Structural vs nominal typing:** TypeScript dizayni JavaScript'ning duck typing xususiyatiga asoslangan — "agar u ducklike ko'rinsa va ducklike tovushlasa, u duck". Interface'lar faqat shape'ni belgilaydi, nom emas. `implements` yozmasdan ham moslik ishlaydi.

Nominal typing kerak bo'lgan holatlarda branded type'lar ishlatiladi:

```typescript
type UserId = string & { __brand: "UserId" };
type ProductId = string & { __brand: "ProductId" };

// Ikkalasi string, lekin brand tufayli almashtirib bo'lmaydi
```

Bu trick structural typing'da nominal'ga yaqin xatti-harakat'ni simulate qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy implements
interface Serializable {
  serialize(): string;
}

interface Loggable {
  log(message: string): void;
}

class User implements Serializable, Loggable {
  constructor(public name: string, public age: number) {}

  serialize(): string {
    return JSON.stringify({ name: this.name, age: this.age });
  }

  log(message: string): void {
    console.log(`[User:${this.name}] ${message}`);
  }
}

// 2. implements + noImplicitAny
interface Handler {
  handle(request: Request): Response;
}

// ✅ To'g'ri — explicit type
class UserHandler implements Handler {
  handle(request: Request): Response {
    return new Response("OK");
  }
}

// 3. Structural typing — implements'siz moslik
interface HasName {
  name: string;
}

class Product {
  constructor(public name: string, public price: number) {}
}

function printName(obj: HasName): void {
  console.log(obj.name);
}

printName(new Product("Phone", 999));  // ✅ Structural
printName({ name: "Ali" });             // ✅ Oddiy object

// 4. Interface bilan decorator-style pattern
interface Cacheable {
  cacheKey(): string;
  cacheTtl(): number;
}

interface Validatable {
  validate(): boolean;
  errors(): string[];
}

class UserDto implements Cacheable, Validatable {
  constructor(
    public readonly id: number,
    public name: string,
    public email: string
  ) {}

  cacheKey(): string {
    return `user:${this.id}`;
  }

  cacheTtl(): number {
    return 3600;
  }

  validate(): boolean {
    return this.errors().length === 0;
  }

  errors(): string[] {
    const errs: string[] = [];
    if (!this.name.trim()) errs.push("Name is required");
    if (!this.email.includes("@")) errs.push("Invalid email");
    return errs;
  }
}

// 5. Interface type sifatida ishlatish
function cacheItem(item: Cacheable): void {
  const key = item.cacheKey();
  const ttl = item.cacheTtl();
  // cache.set(key, item, ttl);
}

cacheItem(new UserDto(1, "Ali", "ali@test.com"));

// 6. Class va interface merge (declaration merging)
interface User {
  // Qo'shimcha property'lar interface'dan
  extraField?: string;
}

class User {
  constructor(public name: string) {}
}

// User endi: { name: string; extraField?: string }
```

</details>

---

## Class Extends Class — Inheritance

### Nazariya

`extends` keyword — bitta class'dan boshqasiga meros olish (inheritance). Subclass parent class'ning barcha public va protected member'larini meros oladi.

```typescript
class Animal {
  constructor(
    public name: string,
    protected speed: number
  ) {}

  move(): string {
    return `${this.name} moves at ${this.speed}km/h`;
  }
}

class Bird extends Animal {
  constructor(
    name: string,
    speed: number,
    public canFly: boolean
  ) {
    super(name, speed); // Parent constructor MAJBURIY
  }

  fly(): string {
    if (!this.canFly) return `${this.name} can't fly`;
    return `${this.name} flies at ${this.speed}km/h`; // protected speed
  }
}
```

**`super` qoidalari:**

1. Subclass constructor'da `super()` chaqirish **majburiy** — parent constructor'ni ishga tushiradi
2. `super()` — `this`'dan **oldin** chaqirilishi kerak
3. `super.method()` — parent class'ning method'ini chaqirish (override bo'lganda)

**Method override:** Subclass parent method'ni qayta yozishi mumkin — bu polymorphism:

```typescript
class Animal {
  speak(): string {
    return "...";
  }
}

class Cat extends Animal {
  speak(): string {
    return "Meow";  // Override — parent'ning speak o'rniga
  }
}

const animals: Animal[] = [new Animal(), new Cat()];
animals.map(a => a.speak()); // ["...", "Meow"] — runtime polymorphism
```

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript inheritance JavaScript inheritance bilan bir xil — prototype chain orqali ishlaydi. TypeScript faqat type checking qo'shadi:

```
Bird extends Animal

Bird.prototype
  ├── fly()                ← own method
  └── __proto__ → Animal.prototype
                    ├── move()     ← inherited method
                    └── __proto__ → Object.prototype
```

`instanceof` ishlaydi — runtime'da prototype chain tekshiriladi:

```typescript
const eagle = new Bird("Eagle", 100, true);
eagle instanceof Bird;   // true
eagle instanceof Animal; // true (prototype chain)
eagle instanceof Object; // true
```

**`super` chaqiruv mexanizmi:** `super()` parent constructor'ni chaqiradi. JavaScript va TypeScript'da `super` `this`'dan oldin chaqirilishi kerak — bu ECMAScript qoidasi, TypeScript dizayn emas. Sabab: parent constructor parent property'larni init qiladi, child bu property'larga keyin tayanishi mumkin.

```typescript
class Base {
  constructor(public x: number) {}
}

class Derived extends Base {
  constructor(x: number, public y: number) {
    // this.y = y; // ❌ super'dan oldin this'ga kirib bo'lmaydi
    super(x);      // ✅ avval super
    this.y = y;    // ✅ endi mumkin (parameter property avtomatik qo'shadi)
  }
}
```

**`super.method()` dispatch:** `super.method()` parent class'ning method'ini to'g'ridan-to'g'ri chaqiradi — prototype chain'ni o'tkazib yuboradi. Bu override bo'lgan method'ning parent versiyasiga kirish uchun kerak:

```typescript
class Base {
  greet(): string { return "Hello from Base"; }
}

class Child extends Base {
  greet(): string {
    const baseGreeting = super.greet();  // Parent method
    return `${baseGreeting}, enhanced by Child`;
  }
}
```

**Multiple inheritance yo'q:** JavaScript va TypeScript **single inheritance** bilan cheklangan — class faqat bitta parent'ni extend qila oladi. Multiple inheritance kerak bo'lsa, **mixin pattern** ishlatiladi ([11-advanced-oop.md](11-advanced-oop.md)).

**Constructor chain:** `new DerivedClass()` chaqirilganda JavaScript/TypeScript quyidagi tartibda ishlaydi:

1. `Derived` constructor boshlanadi
2. `super(...)` — `Base` constructor chaqiriladi
3. `Base` constructor tugaydi, `this` bilan base property'lar init bo'ladi
4. `Derived` constructor davom etadi, `this`'ga kirish mumkin
5. `Derived` constructor tugaydi

Parameter property'lar bilan aralash holatda, avtomatik assignment'lar super'dan keyin qo'shiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy inheritance
class Animal {
  constructor(
    public name: string,
    protected speed: number
  ) {}

  move(): string {
    return `${this.name} moves at ${this.speed}km/h`;
  }
}

class Dog extends Animal {
  constructor(name: string, speed: number, public breed: string) {
    super(name, speed);
  }

  bark(): string {
    return `${this.name} barks!`;
  }
}

const rex = new Dog("Rex", 30, "Labrador");
rex.move();  // "Rex moves at 30km/h" — parent method
rex.bark();  // "Rex barks!" — own method

// 2. Method override + super
class Base {
  greet(): string {
    return "Hello from Base";
  }
}

class Child extends Base {
  override greet(): string {
    const base = super.greet();
    return `${base}, enhanced by Child`;
  }
}

// 3. EventEmitter chain
class EventEmitter {
  protected listeners: Map<string, Function[]> = new Map();

  on(event: string, callback: Function): void {
    const list = this.listeners.get(event) ?? [];
    list.push(callback);
    this.listeners.set(event, list);
  }

  protected emit(event: string, ...args: unknown[]): void {
    this.listeners.get(event)?.forEach(fn => fn(...args));
  }
}

class LoggingEmitter extends EventEmitter {
  private eventLog: string[] = [];

  override emit(event: string, ...args: unknown[]): void {
    this.eventLog.push(`${event} at ${Date.now()}`);
    super.emit(event, ...args); // Parent'ni chaqirish
  }

  getLog(): string[] {
    return [...this.eventLog];
  }
}

// 4. Polymorphism misoli
class Shape {
  area(): number {
    return 0;
  }
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  override area(): number {
    return Math.PI * this.radius ** 2;
  }
}

class Square extends Shape {
  constructor(private side: number) { super(); }
  override area(): number {
    return this.side ** 2;
  }
}

const shapes: Shape[] = [new Circle(5), new Square(10)];
const totalArea = shapes.reduce((sum, s) => sum + s.area(), 0);
// Runtime polymorphism — har shape'da o'z area() ishlaydi

// 5. Constructor chain — complex
class User {
  constructor(public name: string, public email: string) {
    console.log(`User constructor: ${name}`);
  }
}

class Admin extends User {
  constructor(
    name: string,
    email: string,
    public permissions: string[]
  ) {
    super(name, email); // Avval parent
    console.log(`Admin constructor: ${permissions.length} permissions`);
  }
}

class SuperAdmin extends Admin {
  constructor(
    name: string,
    email: string,
    permissions: string[],
    public isGod: boolean
  ) {
    super(name, email, permissions);
    console.log(`SuperAdmin constructor: isGod=${isGod}`);
  }
}

// new SuperAdmin("Ali", "ali@test.com", ["read"], true);
// Output:
// User constructor: Ali
// Admin constructor: 1 permissions
// SuperAdmin constructor: isGod=true

// 6. instanceof chain
class A {}
class B extends A {}
class C extends B {}

const c = new C();
c instanceof A; // true
c instanceof B; // true
c instanceof C; // true
// prototype chain: c -> C.prototype -> B.prototype -> A.prototype
```

</details>

---

## override Keyword

### Nazariya

`override` keyword (TS 4.3+) — subclass'da parent method'ni qayta yozayotganingizni **aniq belgilash**. Bu keyword `noImplicitOverride: true` tsconfig option bilan birgalikda kuchli ishlaydi.

**Muammo:** `override` siz yozilgan subclass method'larda typo'lar ko'rinmaydi:

```typescript
class Animal {
  speak(): string { return "..."; }
}

class Dog extends Animal {
  // Override qilmoqchi edik, lekin typo
  speck(): string { return "Woof"; }
  // TypeScript hech qanday xato bermaydi — yangi method yaratilgan deb hisoblaydi
}
```

**Yechim — `override` + `noImplicitOverride`:**

```typescript
// tsconfig: "noImplicitOverride": true

class Dog extends Animal {
  // override speck(): string { return "Woof"; }
  // ❌ Error: This member cannot have an 'override' modifier because
  // it is not declared in the base class 'Animal'
  // Typo tufayli xato topildi!

  override speak(): string { return "Woof"; }
  // ✅ To'g'ri — parent'da speak() bor
}
```

`noImplicitOverride: true` bo'lganda:

- Parent method'ni override qilsangiz — `override` yozish **majburiy**
- `override` yozgan method parent'da yo'q bo'lsa — **xato**

Bu ikki tomonlama himoya — typo ham, noto'g'ri override ham ushlanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`override` keyword **faqat compile-time**'da ishlaydi. Compiled JavaScript'da butunlay o'chiriladi:

```typescript
// TS
class Dog extends Animal {
  override speak(): string { return "Woof"; }
}

// Compiled JS
class Dog extends Animal {
  speak() { return "Woof"; }
}
```

**Nima uchun keyword kerak:** JavaScript'da method override "invisible" — subclass'da bir xil nomli method yozish avtomatik override. Typo hech qanday ogohlantirishsiz parent method'ni "saqlab qoladi" (subclass'ning yangi method'i bilan birga). Bu **silent bug** manbai.

`override` keyword'ning ikki foydasi:

1. **Typo detection** — method nomida xato bo'lsa compile error
2. **Intent communication** — boshqa developer'larga "bu method override" signalini beradi

**`noImplicitOverride` va refactoring:** Katta code base'da parent class'da method o'zgarsa (nom yoki signature), `override` bilan belgilangan subclass method'lar darhol xato beradi. `override` siz holatda — silent breakage.

```typescript
// Parent o'zgardi
class Animal {
  // speak -> sayHello ga o'zgardi
  sayHello(): string { return "..."; }
}

// Subclass — yo'q bo'lib ketgan parent method'ni "override" qilmoqchi
class Dog extends Animal {
  override speak(): string { // ❌ Error: 'speak' yo'q parent'da
    return "Woof";
  }
}
```

Bu erta detection — refactoring bug'larini kompilator topadi.

**Tavsiya:** Katta TypeScript loyihalarda `noImplicitOverride: true` yoqish tavsiya etiladi — bu "strict" mode'ning qismi emas, alohida option. Qisqa muddatli "friction" bor (har override'ga keyword qo'shish kerak), lekin uzoq muddatli foyda katta.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy override
abstract class HttpHandler {
  abstract handle(req: Request): Promise<Response>;

  handleError(error: Error): Response {
    return new Response(error.message, { status: 500 });
  }
}

class UserHandler extends HttpHandler {
  override async handle(req: Request): Promise<Response> {
    return new Response("User data");
  }

  override handleError(error: Error): Response {
    return new Response(
      JSON.stringify({ error: error.message, code: "USER_ERROR" }),
      { status: 400 }
    );
  }
}

// 2. override + super chaqiruv
class Logger {
  log(message: string): void {
    console.log(message);
  }
}

class TimestampedLogger extends Logger {
  override log(message: string): void {
    super.log(`[${new Date().toISOString()}] ${message}`);
  }
}

// 3. override typo detection
class Base {
  sendMessage(msg: string): void { console.log(msg); }
}

class Derived extends Base {
  // ❌ noImplicitOverride: true bilan
  // override sendMesage(msg: string): void { ... }
  // Error: 'sendMesage' is not declared in base class 'Base'

  // ✅ To'g'ri
  override sendMessage(msg: string): void {
    console.log(`Derived: ${msg}`);
  }
}

// 4. override chain — uch daraja
class A {
  greet(): string { return "A"; }
}

class B extends A {
  override greet(): string {
    return `B > ${super.greet()}`;
  }
}

class C extends B {
  override greet(): string {
    return `C > ${super.greet()}`;
  }
}

new C().greet(); // "C > B > A"

// 5. override bilan generic
abstract class Collection<T> {
  abstract add(item: T): void;
  abstract toArray(): T[];

  size(): number {
    return this.toArray().length;
  }
}

class UniqueCollection<T> extends Collection<T> {
  private items: Set<T> = new Set();

  override add(item: T): void {
    this.items.add(item);
  }

  override toArray(): T[] {
    return Array.from(this.items);
  }

  // size() — meros olinadi, override shart emas
}
```

</details>

---

## this Type — Method Chaining

### Nazariya

TypeScript'da `this`'ni return type sifatida ishlatish mumkin — bu **polymorphic `this`** deb ataladi. Subclass'larda method chaining (fluent API)'ni to'g'ri type'lash uchun juda foydali.

**Muammo — `this` type'siz:**

```typescript
class Builder {
  private data: Record<string, unknown> = {};

  set(key: string, value: unknown): Builder {
    this.data[key] = value;
    return this;
  }
}

class UserBuilder extends Builder {
  setName(name: string): UserBuilder {
    return this.set("name", name) as UserBuilder;
    // ❌ Yomon — this.set() Builder qaytaradi, UserBuilder emas
    // Har chaqiriqda cast kerak
  }
}
```

**Yechim — `this` return type:**

```typescript
class Builder {
  private data: Record<string, unknown> = {};

  set(key: string, value: unknown): this {
    this.data[key] = value;
    return this; // this — subclass'da subclass type bo'ladi
  }
}

class UserBuilder extends Builder {
  setName(name: string): this {
    this.set("name", name);
    return this;
  }
}

// Fluent chaining — barcha method'lar UserBuilder qaytaradi
const user = new UserBuilder()
  .set("age", 25)       // this → UserBuilder
  .setName("Ali")        // this → UserBuilder
  .set("role", "admin"); // this → UserBuilder
```

`this` type — har subclass'da o'sha subclass'ning type'iga **resolve** bo'ladi. `Builder`'da `this = Builder`, `UserBuilder`'da `this = UserBuilder`.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `this` type'ni **substitution** orqali amalga oshiradi. Har class uchun alohida `this` type mavjud bo'lib, chaqiruv joyidagi receiver type'iga aylantiriladi:

```
class Builder { set(): this }
class UserBuilder extends Builder { setName(): this }

Kompilator this type'ni resolve qiladi:

Builder context:
  this → Builder (Builder o'zi)
  set() return type → Builder

UserBuilder context:
  this → UserBuilder (UserBuilder o'zi)
  set() return type → UserBuilder (parent method ham subclass type)
  setName() return type → UserBuilder

Chaqiruv joyida:
  1. Method'da `this` return type uchraydi
  2. Call site'da receiver'ning actual type aniqlanadi
  3. this → receiver type'ga almashtiriladi
  4. Natija: subclass method'lari subclass type qaytaradi
```

Bu oddiy substitution mexanizmi — har class scope'ida `this` shu class'ning placeholder'i. `extends` qilinganda kompilator yangi `this`'ni subclass bilan bog'laydi.

**Compiled JS'da iz yo'q:** `this` type faqat compile-time type narrowing. Runtime'da `return this` oddiy object reference qaytaradi — JavaScript engine `this`'ning type'ini bilmaydi.

**Fluent API pattern:** Method chaining — `this` return type'ning asosiy use case. Builder pattern, query builder, DOM manipulation library'larda (jQuery, anime.js) keng qo'llaniladi.

**Contrast with `ClassName` return type:** `Builder` deb yozganingizda, kompilator har doim `Builder` qaytaradi — subclass'da ham. `this` yozganingizda, har subclass'da o'sha subclass'ning type'i qaytariladi. Bu fluent API'ni subclass'larda ham ishlashini ta'minlaydi.

**`this` va static method'lar:** Static method'da `this` — class constructor function'ga ishora qiladi. Bu `this` type'i sifatida ishlatilganda, subclass'da subclass constructor'ga resolve bo'ladi (static polymorphism):

```typescript
class Model {
  static create<T extends Model>(this: new () => T): T {
    return new this();
  }
}

class User extends Model {}

Model.create();  // Model instance
User.create();   // User instance — this = User
```

Bu pattern "static factory with subclass type"'ni ifodalaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy fluent API
class QueryBuilder {
  protected parts: string[] = [];

  from(table: string): this {
    this.parts.push(`FROM ${table}`);
    return this;
  }

  where(condition: string): this {
    this.parts.push(`WHERE ${condition}`);
    return this;
  }

  build(): string {
    return `SELECT * ${this.parts.join(" ")}`;
  }
}

class PaginatedQueryBuilder extends QueryBuilder {
  limit(n: number): this {
    this.parts.push(`LIMIT ${n}`);
    return this;
  }

  offset(n: number): this {
    this.parts.push(`OFFSET ${n}`);
    return this;
  }
}

// Chain — hech qanday cast kerak emas
const query = new PaginatedQueryBuilder()
  .from("users")          // this → PaginatedQueryBuilder
  .where("age > 18")
  .limit(10)
  .offset(20)
  .build();

// 2. HTTP request builder
class HttpRequest {
  private _method: string = "GET";
  private _url: string = "";
  private _headers: Map<string, string> = new Map();

  method(method: string): this {
    this._method = method;
    return this;
  }

  url(url: string): this {
    this._url = url;
    return this;
  }

  header(key: string, value: string): this {
    this._headers.set(key, value);
    return this;
  }

  async send(): Promise<Response> {
    const headers: Record<string, string> = {};
    this._headers.forEach((v, k) => { headers[k] = v; });
    return fetch(this._url, { method: this._method, headers });
  }
}

// 3. Validation DSL
class Validator<T> {
  private rules: Array<(value: T) => string | null> = [];

  rule(check: (value: T) => string | null): this {
    this.rules.push(check);
    return this;
  }

  validate(value: T): string[] {
    return this.rules
      .map(rule => rule(value))
      .filter((msg): msg is string => msg !== null);
  }
}

class StringValidator extends Validator<string> {
  minLength(n: number): this {
    return this.rule(value =>
      value.length < n ? `Must be at least ${n} characters` : null
    );
  }

  maxLength(n: number): this {
    return this.rule(value =>
      value.length > n ? `Must be at most ${n} characters` : null
    );
  }

  pattern(regex: RegExp, message: string): this {
    return this.rule(value => regex.test(value) ? null : message);
  }
}

const passwordValidator = new StringValidator()
  .minLength(8)
  .maxLength(64)
  .pattern(/[A-Z]/, "Must contain uppercase")
  .pattern(/[0-9]/, "Must contain digit");

const errors = passwordValidator.validate("abc");
// ["Must be at least 8 characters", "Must contain uppercase", "Must contain digit"]

// 4. Static this type
class Model {
  static create<T extends Model>(this: new () => T): T {
    return new this();
  }
}

class Product extends Model {
  constructor() { super(); }
}

class User extends Model {
  constructor() { super(); }
}

const p = Product.create(); // Product instance
const u = User.create();    // User instance
```

</details>

---

## Generics bilan Classes

### Nazariya

Generic class — type parameter'li class. Type parameter class body'dagi property'lar, method'lar, constructor'da ishlatiladi. Bu class'ni turli type'lar bilan qayta ishlatish imkonini beradi.

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numbers = new Stack<number>();
numbers.push(1);
numbers.push(2);
// numbers.push("3"); // ❌ Error
```

**Generic constraint bilan class:**

```typescript
interface HasId {
  id: string | number;
}

class Repository<T extends HasId> {
  private items: Map<string | number, T> = new Map();

  save(item: T): void {
    this.items.set(item.id, item);
  }

  findById(id: T["id"]): T | undefined {
    return this.items.get(id);
  }
}
```

Generic class'lar haqida chuqur [08-generics.md](08-generics.md)'da. Bu yerda class context'da generic'larni qanday ishlatilishi ko'rsatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Generic class'lar JavaScript'ga compile bo'lganda type parameter'lar butunlay o'chiriladi:

```typescript
// TS
class Box<T> {
  constructor(public value: T) {}
  unwrap(): T { return this.value; }
}

// Compiled JS
class Box {
  constructor(value) {
    this.value = value;
  }
  unwrap() { return this.value; }
}
```

**Bitta class emit:** Kompilator `Box<number>` va `Box<string>` uchun **bitta class** emit qiladi. Runtime'da ikki instance orasida hech qanday farq yo'q:

```typescript
const numBox = new Box<number>(42);
const strBox = new Box<string>("hello");

numBox instanceof Box; // true
strBox instanceof Box; // true
// Runtime'da farq yo'q
```

**Static member va generic:** Static member'larda class'ning generic type parameter'i **ishlatib bo'lmaydi** — static member'lar instance-level emas:

```typescript
class Container<T> {
  // static default: T; // ❌ Error
  // static create(value: T): Container<T> {} // ❌ Error

  static create<U>(value: U): Container<U> {
    return new Container(value);
  }
}
```

Yechim: method-level generic.

**Constructor type parameter inference:** `new Box(42)` chaqirilganda kompilator argument'dan `T = number` deb infer qiladi — explicit `<number>` shart emas. Lekin argument yo'q bo'lsa explicit kerak:

```typescript
const box1 = new Box(42);        // T = number (inferred)
const box2 = new Box<string>("hello"); // explicit
const empty = new Stack();        // T = unknown — explicit yoki push orqali infer
```

**Instantiation expression** (TS 4.7+) — generic class'ni pre-bind qilish:

```typescript
const NumStack = Stack<number>; // Pre-bound type
const s = new NumStack();        // Stack<number>
```

Bu factory pattern va dependency injection'da foydali.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy generic class
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}

const numbers = new Stack<number>();
numbers.push(1);
numbers.push(2);

// 2. Generic class with constraint
interface Entity {
  id: string;
  createdAt: Date;
}

class Repository<T extends Entity> {
  private store: Map<string, T> = new Map();

  save(entity: T): void {
    this.store.set(entity.id, entity);
  }

  findById(id: string): T | undefined {
    return this.store.get(id);
  }

  findAll(): T[] {
    return Array.from(this.store.values());
  }
}

interface User extends Entity {
  name: string;
  email: string;
}

const userRepo = new Repository<User>();

// 3. Method-level generic inside class
class Mapper<T> {
  constructor(private items: T[]) {}

  // Method-level U — class T'dan mustaqil
  map<U>(fn: (item: T) => U): U[] {
    return this.items.map(fn);
  }

  filter(predicate: (item: T) => boolean): T[] {
    return this.items.filter(predicate);
  }
}

const mapper = new Mapper([1, 2, 3, 4, 5]);
const strings = mapper.map(n => n.toString()); // string[]

// 4. Multiple type parameters
class KeyValueStore<K, V> {
  private map = new Map<K, V>();

  set(key: K, value: V): this {
    this.map.set(key, value);
    return this;
  }

  get(key: K): V | undefined {
    return this.map.get(key);
  }

  has(key: K): boolean {
    return this.map.has(key);
  }
}

const config = new KeyValueStore<string, number>();
config.set("port", 3000).set("timeout", 5000);

// 5. Generic class + inheritance
interface Timestamped {
  createdAt: Date;
  updatedAt: Date;
}

class TimestampedStore<T extends Timestamped> {
  private items: T[] = [];

  add(item: T): void {
    item.createdAt = new Date();
    item.updatedAt = new Date();
    this.items.push(item);
  }

  getRecent(since: Date): T[] {
    return this.items.filter(item => item.updatedAt >= since);
  }
}

interface Article extends Timestamped {
  title: string;
  content: string;
}

const store = new TimestampedStore<Article>();

// 6. Static method with method-level generic
class Container<T> {
  constructor(public value: T) {}

  static from<U>(value: U): Container<U> {
    return new Container(value);
  }
}

const strCnt = Container.from("hello");  // Container<string>
const numCnt = Container.from(42);       // Container<number>

// 7. Instantiation expression (TS 4.7+)
const NumStack = Stack<number>;
const stack = new NumStack();
stack.push(1);
```

</details>

---

## accessor Keyword (TS 4.9+)

### Nazariya

`accessor` keyword — class property'ga avtomatik getter va setter yaratadi. Bu TC39 decorators'ning bir qismi bo'lib, TypeScript 4.9'da qo'shilgan. TS 5.0+'dan boshlab TC39 decorators bilan to'liq ishlaydi.

```typescript
class Person {
  accessor name: string;

  constructor(name: string) {
    this.name = name; // setter chaqiriladi
  }
}

const p = new Person("Ali");
console.log(p.name); // getter chaqiriladi → "Ali"
p.name = "Vali";     // setter chaqiriladi
```

`accessor` yozganingizda TypeScript ichida quyidagilar sodir bo'ladi:

1. `#name` — private storage field yaratiladi (ES `#` private)
2. `get name()` — getter yaratiladi
3. `set name(value)` — setter yaratiladi

**Nima uchun kerak?** Asosan **decorator'lar** bilan birga ishlatish uchun — decorator getter/setter'ni intercept qila oladi. Bu validation, logging, observation pattern'lari uchun foydali.

**Oddiy property bilan farq:** `accessor` property'siz, siz manual tarzda getter/setter yozishingiz yoki oddiy property ishlatishingiz kerak edi. `accessor` shu boilerplate'ni kamaytiradi va decorator API'ga bog'lanish imkonini beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`accessor` property JavaScript'ga quyidagicha compile bo'ladi (target ES2022+):

```typescript
// TS
class Person {
  accessor name: string = "Ali";
}

// Compiled JS
class Person {
  #name = "Ali";  // Private storage

  get name() {
    return this.#name;
  }

  set name(value) {
    this.#name = value;
  }
}
```

Kompilator `#name` private field'ni avtomatik yaratadi — foydalanuvchi `#` yozmasdan ham runtime privacy oladi.

**Decorator bilan integration:** `accessor` property'ning asosiy foydasi — TC39 decorator'lar bilan. Decorator getter yoki setter'ni intercept qilishi mumkin:

```typescript
function logged(
  target: ClassAccessorDecoratorTarget<unknown, string>,
  context: ClassAccessorDecoratorContext
) {
  return {
    get(this: unknown) {
      console.log(`Getting ${String(context.name)}`);
      return target.get.call(this);
    },
    set(this: unknown, value: string) {
      console.log(`Setting ${String(context.name)} to ${value}`);
      target.set.call(this, value);
    },
  };
}

class User {
  @logged accessor name: string = "default";
}
```

Decorator'ga `ClassAccessorDecoratorTarget` beriladi — original `get` va `set` method'larga referens. Decorator yangi `{ get, set }` object qaytaradi va shu asl'ning o'rnini oladi.

**`accessor` vs oddiy property:**

| Xususiyat | Oddiy property | `accessor` |
|-----------|----------------|-------------|
| Storage | Instance property | Private field (`#`) |
| Runtime privacy | ❌ | ✅ |
| Decorator intercept | Faqat `@observable` kabi eski API | Native TC39 |
| Performance | Biroz tez | Biroz sekin (getter/setter overhead) |
| Bekor qilish | Oddiy property — hech narsa | Hech narsa (built-in) |

**Qachon ishlatish:** `accessor` — agar siz decorator ishlatishingiz kerak bo'lsa (validation, reactivity, logging). Oddiy property — boshqa hamma holatlarda.

**Decorator'lar chuqur [19-decorators.md](19-decorators.md)'da.**

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy accessor
class Person {
  accessor name: string = "default";
  accessor age: number = 0;
}

const p = new Person();
p.name; // "default" — getter
p.name = "Ali"; // setter

// 2. Decorator bilan validation
function range(min: number, max: number) {
  return function (
    target: ClassAccessorDecoratorTarget<unknown, number>,
    context: ClassAccessorDecoratorContext
  ) {
    return {
      set(this: unknown, value: number) {
        if (value < min || value > max) {
          throw new RangeError(`${String(context.name)} must be ${min}-${max}`);
        }
        target.set.call(this, value);
      },
      get(this: unknown) {
        return target.get.call(this);
      },
    };
  };
}

class Slider {
  @range(0, 100) accessor value: number = 50;
  @range(1, 10) accessor step: number = 1;
}

const slider = new Slider();
slider.value = 75;  // ✅
// slider.value = 150; // ❌ RangeError

// 3. Decorator bilan logging
function logged(
  target: ClassAccessorDecoratorTarget<unknown, unknown>,
  context: ClassAccessorDecoratorContext
) {
  return {
    get(this: unknown) {
      const value = target.get.call(this);
      console.log(`GET ${String(context.name)}:`, value);
      return value;
    },
    set(this: unknown, value: unknown) {
      console.log(`SET ${String(context.name)}:`, value);
      target.set.call(this, value);
    },
  };
}

class Config {
  @logged accessor apiUrl: string = "http://localhost";
}

const config = new Config();
config.apiUrl;              // Log: GET apiUrl: "http://localhost"
config.apiUrl = "https://"; // Log: SET apiUrl: "https://"
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class Person {
  accessor name: string = "Ali";
}
```

```javascript
// Compiled JS (target: ES2022+) — # private storage + getter/setter
class Person {
  #name = "Ali";

  get name() {
    return this.#name;
  }

  set name(value) {
    this.#name = value;
  }
}
```

`accessor` keyword — TypeScript'ning syntactic sugar'i. Compiled output'da native JavaScript getter/setter + `#` private field qoladi.

</details>

---

## Edge Cases va Gotchas

### 1. TS `private` Runtime'da Himoyasiz

TypeScript `private` modifier faqat compile-time tekshiruv. Runtime'da property oddiy JavaScript property bo'lib qoladi — bracket notation compile-time'da bloklangan, lekin `as any` cast yoki JavaScript fayldan access ochiq:

```typescript
class Secret {
  private password: string = "12345";
}

const s = new Secret();

// TS 4.3+ — compile-time bloklangan:
// s["password"];           // ❌ Compile error

// Lekin runtime'da ochiq:
(s as any).password;         // ✅ "12345"
// JavaScript fayldan: s.password — mumkin
```

**Yechim:** Runtime privacy kerak bo'lsa — `#` ECMAScript private fields ishlatish. `#` ECMAScript engine tomonidan enforce qilinadi — class tashqarisida syntax xato, `as any` cast ham bypass qila olmaydi.

**Pedagogik xulosa:** `private` — API niyati. `#` — runtime kafolat. Security sensitive data uchun `#` kerak.

### 2. `strictPropertyInitialization` va `!` Xavfsizligi

Definite assignment assertion (`!`) — TypeScript'ga "bu property init bo'lgan" deb yolg'on gapirish. Xavfli, chunki runtime'da crash:

```typescript
class UserService {
  db!: Database;  // Framework init qiladi deb

  constructor() {
    // db init yo'q — xato
  }

  query(): void {
    this.db.execute("SELECT ..."); // Runtime TypeError
  }
}

interface Database {
  execute(query: string): void;
}
```

`!` faqat **framework context'da** xavfsiz — Angular `@Input`, NestJS `@Inject`, test framework'lar `beforeEach`. Boshqa joylarda constructor'da init qiling.

**Xavfli misollar:**

```typescript
// ❌ Test'da setUp bilan init, lekin unit test'da ishlamasa
class Service {
  data!: string;
}

// ❌ Conditional init
class Config {
  apiKey!: string;

  load(): void {
    if (process.env.API_KEY) this.apiKey = process.env.API_KEY;
    // else — apiKey undefined, runtime'da xato bo'lishi mumkin
  }
}
```

### 3. Abstract Class Runtime'da Oddiy Class

Abstract class TypeScript compile-time constraint. Compiled JavaScript'da `abstract` keyword butunlay o'chiriladi — runtime'da `new AbstractClass()` bloklanmaydi:

```typescript
abstract class Animal {
  abstract speak(): string;
}

// TS: new Animal(); // ❌ Compile error

// Compiled JS:
// class Animal {}
// new Animal(); // ✅ Ishlaydi! Runtime'da bloklash yo'q
```

JavaScript fayldan yoki `any` cast'dan abstract class'ni instantiate qilish mumkin. Agar instance'da abstract method chaqirilsa, `undefined is not a function` xatosi chiqadi.

**Yechim — runtime protection kerak bo'lsa:**

```typescript
abstract class Animal {
  constructor() {
    if (new.target === Animal) {
      throw new Error("Cannot instantiate abstract class");
    }
  }
}
```

`new.target` — constructor chaqirilgan class'ga ishora qiladi. Agar `Animal`'ning o'zi chaqirilsa — throw. Subclass chaqirsa — `new.target` subclass bo'ladi, OK.

### 4. `implements` va Type Inference Yo'qligi

`implements` keyword class'ning shape'ini tekshiradi, lekin method parameter'lariga type'ni avtomatik bermaydi:

```typescript
interface Handler {
  handle(event: { type: string; data: unknown }): void;
}

class MyHandler implements Handler {
  handle(event) {  // ❌ noImplicitAny: event: any
    console.log(event.type);
  }
}
```

**Yechim:** Method parameter'larga explicit type yozish:

```typescript
class MyHandler implements Handler {
  handle(event: { type: string; data: unknown }): void {
    console.log(event.type);
  }
}
```

Yoki interface'ni alias sifatida ishlatish:

```typescript
type HandlerEvent = { type: string; data: unknown };

interface Handler {
  handle(event: HandlerEvent): void;
}

class MyHandler implements Handler {
  handle(event: HandlerEvent): void {  // Type alias reuse
    console.log(event.type);
  }
}
```

**Nima uchun TypeScript shunday:** Dizayn qarori. Class method'ning parameter'lari interface'dan qat'iy bog'liq emas — class boshqa method'lar yoki constraint'larda ham ishlatiladi. Explicit annotation ishonchli.

### 5. `this` Type Base Method'ida Subclass Type

`this` return type base class'da yozilganda, subclass method chain'larida avtomatik subclass type'iga resolve bo'ladi. Bu qulay, lekin nozik holat — base class method body'da subclass-specific kod yozib bo'lmaydi:

```typescript
class Builder {
  protected data: Record<string, unknown> = {};

  set(key: string, value: unknown): this {
    this.data[key] = value;
    return this;
  }
}

class UserBuilder extends Builder {
  private _name?: string;

  setName(name: string): this {
    this._name = name;
    return this;
  }
}

const user = new UserBuilder()
  .set("age", 25)    // this: UserBuilder ✅
  .setName("Ali")     // ✅ subclass method
  .set("role", "admin"); // ✅
```

**Gotcha:** Base `set` method'i `this` qaytaradi — lekin bu method base'da yozilgan, base kontekstida `this = Builder`. Kompilator shu method'ni subclass'da chaqirsa, `this = UserBuilder` deb resolve qiladi. Lekin base method body'da `this._name`'ga kira olmaydi (chunki `Builder`'da yo'q):

```typescript
class Builder {
  set(key: string, value: unknown): this {
    // this._name; // ❌ Error: _name yo'q Builder'da
    return this;
  }
}
```

**Xulosa:** `this` type caller context'da resolve bo'ladi, method definition context'da emas. Bu subclass'larning method chaining'ini qulay qiladi, lekin base class method'lari subclass-specific property'larga kira olmaydi.

---

## Common Mistakes

### ❌ Xato 1: `strictPropertyInitialization`'ni tushunmaslik

```typescript
class User {
  name: string;  // ❌ no initializer
  age: number;   // ❌ not assigned in constructor

  loadFromDb(id: number): void {
    this.name = "Ali";
    this.age = 25;
  }
}
```

**✅ To'g'ri usul:**

```typescript
class User {
  constructor(public name: string, public age: number) {}
}
// yoki:
class User {
  name: string;
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}
```

**Nima uchun:** `strictPropertyInitialization` property'lar `undefined` bo'lmasligini kafolatlaydi. `!` (definite assignment) bilan yashirish — faqat framework tomonidan init qilinadigan property'lar uchun xavfsiz.

---

### ❌ Xato 2: `private`'ni runtime privacy deb o'ylash

```typescript
class Secret {
  private password: string = "12345";
}

const s = new Secret();
console.log((s as any).password); // "12345" — runtime'da ochiq!
```

**✅ To'g'ri usul:**

```typescript
class Secret {
  #password: string = "12345"; // ES private — runtime'da ham private

  getPassword(auth: string): string {
    if (auth === "valid") return this.#password;
    throw new Error("Unauthorized");
  }
}
```

**Nima uchun:** TS `private` faqat compile-time. Haqiqiy privacy kerak bo'lsa — `#` ishlatish kerak.

---

### ❌ Xato 3: `implements`'dan type inference kutish

```typescript
interface Logger {
  log(message: string, level: number): void;
}

class ConsoleLogger implements Logger {
  log(message, level) {  // ❌ noImplicitAny error
    console.log(message);
  }
}
```

**✅ To'g'ri usul:**

```typescript
class ConsoleLogger implements Logger {
  log(message: string, level: number): void {
    console.log(`[${level}] ${message}`);
  }
}
```

**Nima uchun:** `implements` faqat class'ning shape'ini tekshiradi — method parameter'lariga type bermaydi. Explicit type annotation majburiy.

---

### ❌ Xato 4: `super()`'ni unutish

```typescript
class Animal {
  constructor(public name: string) {}
}

class Dog extends Animal {
  constructor(name: string, public breed: string) {
    // ❌ Error: 'super' must be called before accessing 'this'
    this.breed = breed;
  }
}
```

**✅ To'g'ri usul:**

```typescript
class Dog extends Animal {
  constructor(name: string, public breed: string) {
    super(name); // avval super
    // parameter property — avtomatik qo'shadi
  }
}
```

**Nima uchun:** `super()` parent class constructor'ni chaqiradi — parent property'larni init qiladi. `this`'ga `super()`'dan oldin kirib bo'lmaydi.

---

### ❌ Xato 5: `readonly`'ni deep immutability deb o'ylash

```typescript
class Config {
  readonly settings: { theme: string; lang: string } = {
    theme: "dark",
    lang: "en",
  };
}

const c = new Config();
c.settings.theme = "light"; // ✅ Ishlaydi! Ichidagi property o'zgaradi
// c.settings = {};          // ❌ qayta assign mumkin emas
```

**✅ To'g'ri usul:**

```typescript
class Config {
  readonly settings: Readonly<{ theme: string; lang: string }> = {
    theme: "dark",
    lang: "en",
  };
}

const c = new Config();
// c.settings.theme = "light"; // ❌ Readonly<T> tufayli
```

**Nima uchun:** `readonly` faqat top-level assignment'ni to'xtatadi. Ichidagi property'lar uchun `Readonly<T>` yoki `DeepReadonly<T>` kerak.

---

## Amaliy Mashqlar

### Mashq 1: BankAccount Class (Oson)

**Savol:** `BankAccount` class yarating — `owner` (string, readonly), `balance` (getter orqali o'qiladi, private storage), `deposit(amount)` va `withdraw(amount)` method'lari bilan. Withdraw balance'dan oshib ketmasin.

<details>
<summary>Javob</summary>

```typescript
class BankAccount {
  constructor(
    public readonly owner: string,
    private _balance: number
  ) {}

  get balance(): number {
    return this._balance;
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error("Amount must be positive");
    this._balance += amount;
  }

  withdraw(amount: number): void {
    if (amount <= 0) throw new Error("Amount must be positive");
    if (amount > this._balance) throw new Error("Insufficient funds");
    this._balance -= amount;
  }
}

const account = new BankAccount("Ali", 1000);
account.deposit(500);    // balance: 1500
account.withdraw(200);   // balance: 1300
account.balance;         // 1300 (getter)
// account.balance = 0;  // ❌ no setter
```

**Tushuntirish:** `_balance` private — tashqaridan faqat `get balance()` orqali o'qiladi. Setter yo'q — tashqaridan o'zgartirish mumkin emas. `deposit` va `withdraw` method'lari validation bilan ishlaydi.

</details>

---

### Mashq 2: Abstract Vehicle (O'rta)

**Savol:** `abstract class Vehicle` yarating — `abstract fuelEfficiency(): number` method bilan. `Car` va `Truck` subclass'lari yarating, har birida turli fuel efficiency hisoblash formulasi. `describe()` concrete method barcha vehicle'lar uchun umumiy.

<details>
<summary>Javob</summary>

```typescript
abstract class Vehicle {
  constructor(
    public readonly brand: string,
    public readonly year: number,
    protected mileage: number
  ) {}

  abstract fuelEfficiency(): number;

  describe(): string {
    return `${this.brand} (${this.year}) — ${this.fuelEfficiency().toFixed(1)} L/100km`;
  }

  addMileage(km: number): void {
    if (km > 0) this.mileage += km;
  }
}

class Car extends Vehicle {
  constructor(
    brand: string,
    year: number,
    mileage: number,
    private engineSize: number
  ) {
    super(brand, year, mileage);
  }

  override fuelEfficiency(): number {
    const base = this.engineSize * 3;
    const degradation = this.mileage / 100000;
    return base + degradation;
  }
}

class Truck extends Vehicle {
  constructor(
    brand: string,
    year: number,
    mileage: number,
    private loadCapacity: number
  ) {
    super(brand, year, mileage);
  }

  override fuelEfficiency(): number {
    const base = 15;
    const loadFactor = this.loadCapacity * 2;
    const degradation = this.mileage / 50000;
    return base + loadFactor + degradation;
  }
}

const vehicles: Vehicle[] = [
  new Car("Toyota", 2023, 50000, 1.6),
  new Truck("Volvo", 2021, 200000, 10),
];

vehicles.forEach(v => console.log(v.describe()));
```

**Tushuntirish:** `Vehicle` abstract — to'g'ridan-to'g'ri instance yaratib bo'lmaydi. Har subclass `fuelEfficiency()`'ni o'zi implement qiladi (polymorphism). `describe()` — shared implementation.

</details>

---

### Mashq 3: Generic Repository (Qiyin)

**Savol:** `Repository<T extends { id: string }>` generic class yarating — `save`, `findById`, `findAll`, `update`, `delete` method'lari bilan. `UserRepository extends Repository<User>` yaratib, `findByEmail` method qo'shing.

<details>
<summary>Javob</summary>

```typescript
interface Entity {
  id: string;
}

class Repository<T extends Entity> {
  protected items: Map<string, T> = new Map();

  save(item: T): T {
    this.items.set(item.id, { ...item });
    return item;
  }

  findById(id: string): T | undefined {
    const item = this.items.get(id);
    return item ? { ...item } : undefined;
  }

  findAll(): T[] {
    return Array.from(this.items.values()).map(item => ({ ...item }));
  }

  update(id: string, data: Partial<Omit<T, "id">>): T | undefined {
    const existing = this.items.get(id);
    if (!existing) return undefined;
    const updated = { ...existing, ...data } as T;
    this.items.set(id, updated);
    return { ...updated };
  }

  delete(id: string): boolean {
    return this.items.delete(id);
  }
}

interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "user";
}

class UserRepository extends Repository<User> {
  findByEmail(email: string): User | undefined {
    for (const user of this.items.values()) {
      if (user.email === email) return { ...user };
    }
    return undefined;
  }

  findByRole(role: User["role"]): User[] {
    return Array.from(this.items.values())
      .filter(u => u.role === role)
      .map(u => ({ ...u }));
  }
}

const repo = new UserRepository();
repo.save({ id: "1", name: "Ali", email: "ali@test.com", role: "admin" });
repo.findByEmail("ali@test.com");
repo.findByRole("admin");
```

**Tushuntirish:** `T extends Entity` — T'ning `id: string` property'ga ega bo'lishini constraint bilan ta'minlaydi. `UserRepository` extends orqali CRUD method'larni meros oladi va `findByEmail`/`findByRole` qo'shadi. `protected items` — subclass'da kirish mumkin.

</details>

---

### Mashq 4: Fluent HttpRequest Builder (Qiyin)

**Savol:** `HttpRequest` class yarating — fluent API bilan. `method()`, `url()`, `header()`, `body()`, `send()` method'lari chaining bilan ishlasin.

<details>
<summary>Javob</summary>

```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

class HttpRequest {
  private _method: HttpMethod = "GET";
  private _url: string = "";
  private _headers: Map<string, string> = new Map();
  private _body: unknown = undefined;

  method(method: HttpMethod): this {
    this._method = method;
    return this;
  }

  url(url: string): this {
    this._url = url;
    return this;
  }

  header(key: string, value: string): this {
    this._headers.set(key, value);
    return this;
  }

  body(data: unknown): this {
    this._body = data;
    return this;
  }

  async send(): Promise<Response> {
    const headers: Record<string, string> = {};
    this._headers.forEach((v, k) => { headers[k] = v; });

    return fetch(this._url, {
      method: this._method,
      headers,
      body: this._body ? JSON.stringify(this._body) : undefined,
    });
  }

  toString(): string {
    return `${this._method} ${this._url}`;
  }
}

// Ishlatish
const request = new HttpRequest()
  .method("POST")
  .url("https://api.example.com/users")
  .header("Content-Type", "application/json")
  .header("Authorization", "Bearer token123")
  .body({ name: "Ali", email: "ali@test.com" });
```

**Tushuntirish:** Har method `this` qaytaradi — chaining imkoni. `this` type'i tufayli subclass'lar ham chaining'ni to'g'ri ishlata oladi.

</details>

---

### Mashq 5: Constructor Execution Order (O'rta)

**Savol:** Quyidagi kodning output'ini ayting:

```typescript
class Animal {
  constructor(public name: string) {
    console.log(`Animal: ${this.name}`);
  }

  speak(): string {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  constructor(name: string, public breed: string) {
    super(name);
    console.log(`Dog: ${this.breed}`);
  }

  override speak(): string {
    return `${this.name} barks`;
  }
}

const dog = new Dog("Rex", "Labrador");
console.log(dog.speak());
console.log(dog instanceof Animal);
```

<details>
<summary>Javob</summary>

```
Animal: Rex
Dog: Labrador
Rex barks
true
```

**Tushuntirish:**

1. `new Dog("Rex", "Labrador")` chaqiriladi
2. Dog constructor ichida `super("Rex")` — Animal constructor chaqiriladi → `"Animal: Rex"`
3. super tugagach, Dog constructor davom etadi → `"Dog: Labrador"`
4. `dog.speak()` — Dog'da override qilingan → `"Rex barks"` (Animal'dagi emas — runtime polymorphism)
5. `dog instanceof Animal` — prototype chain'da `Animal.prototype` bor → `true`

**Muhim:** `super()` har doim `this`'dan oldin chaqirilishi kerak. Parameter properties avtomatik assignment constructor body'ning boshida (super'dan keyin) qo'shiladi.

</details>

---

## Xulosa

Bu bo'limda TypeScript class'larning barcha asosiy tushunchalarini o'rgandik:

**Asosiy tushunchalar:**

- **Class anatomy** — JavaScript class + TypeScript qo'shimchalari, instance type vs constructor type
- **Type annotations** — property, constructor, method'lar uchun type safety
- **Access modifiers** — `public`, `private`, `protected` (faqat compile-time)
- **`#` ECMAScript private** — runtime privacy, bracket notation bypass yo'q
- **`readonly`** — qayta assign mumkin emas (shallow)
- **Parameter properties** — constructor'da shorthand property yaratish
- **Static members** — class'ga birikkan, instance'ga emas
- **Static initialization blocks** (ES2022, TS 4.4+) — murakkab static init logic
- **Abstract classes** — shared implementation + abstract contract
- **`implements`** — interface contract tekshiruvi (type inference bermaydi)
- **`extends`** — inheritance + super + method override
- **`override` keyword** (TS 4.3+) — parent method'ni qayta yozishni aniq belgilash
- **`this` type** — polymorphic return type, fluent API
- **Generics** — type-safe reusable class'lar
- **`accessor` keyword** (TS 4.9+) — TC39 decorator'lar bilan auto-accessor

**Umumiy takeaway'lar:**

1. **TS `private` vs `#` private.** `private` — compile-time niyat, `#` — runtime kafolat.
2. **`strictPropertyInitialization` majburiy.** Har property constructor'da init bo'lishi kerak — `!` faqat framework context'da xavfsiz.
3. **`implements` shape check, type inference emas.** Method parameter'lariga explicit type yozish kerak.
4. **`super()` `this`'dan oldin.** Parent constructor avval ishlaydi.
5. **`readonly` shallow.** Ichki object/array mutable, deep immutability uchun `Readonly<T>` yoki `DeepReadonly<T>`.
6. **`this` type fluent API uchun.** Subclass'larda chaining'ni to'g'ri ishlatish uchun.
7. **Type erasure.** Abstract, implements, override, private, readonly — barchasi compile-time. Runtime'da oddiy JavaScript class qoladi.

**Cross-references:**

- **[06-type-narrowing.md](06-type-narrowing.md)** — `instanceof` narrowing, `this` parameter
- **[07-functions.md](07-functions.md)** — `this` parameter vs `this` type farqi, function types
- **[08-generics.md](08-generics.md)** — Generic class'lar chuqur, `keyof`, constraint
- **[11-advanced-oop.md](11-advanced-oop.md)** — Mixins, class expressions, composition
- **[19-decorators.md](19-decorators.md)** — TC39 decorators, `accessor` keyword bilan
- **[22-tsconfig.md](22-tsconfig.md)** — `strictPropertyInitialization`, `noImplicitOverride`, `noImplicitAny`
- **[23-type-safe-patterns.md](23-type-safe-patterns.md)** — Branded types, nominal typing
- **[25-type-compatibility.md](25-type-compatibility.md)** — Class assignability, variance

---

**Keyingi bo'lim:** [11-advanced-oop.md](11-advanced-oop.md) — Advanced OOP Patterns: mixins (multiple inheritance simulation), class expressions, composition vs inheritance, `this` as return type chuqur, builder pattern intro, va dependency injection pattern'lari.
