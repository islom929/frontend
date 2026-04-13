# Interview: Classes TypeScript da

> Access modifiers, parameter properties, abstract classes, implements, override, this type, TS private vs ES # private bo'yicha interview savollari.

---

## Nazariy savollar

### 1. TypeScript class da `public`, `private`, `protected` farqi nima? Runtime da nima bo'ladi?

<details>
<summary>Javob</summary>

Uchta access modifier — class member larining visibility darajasini belgilaydi:

| Modifier | Class ichida | Subclass da | Tashqarida |
|----------|:----------:|:-----------:|:----------:|
| `public` (default) | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ❌ |
| `private` | ✅ | ❌ | ❌ |

**Muhim:** Bular faqat **compile-time** tekshiruvi. JS ga compile bo'lganda **butunlay o'chiriladi**:

```typescript
class Secret {
  private key: string = "abc123";
}
// Compiled JS: class Secret { constructor() { this.key = "abc123"; } }

const s = new Secret();
(s as any).key; // "abc123" — runtime da ochiq!
```

Haqiqiy runtime privacy uchun ES `#` private fields kerak:

```typescript
class Secret {
  #key: string = "abc123";
}
(new Secret() as any).#key; // SyntaxError — runtime da ham yopiq
```

</details>

### 2. Parameter properties nima?

<details>
<summary>Javob</summary>

Constructor parametrida access modifier yoki `readonly` yozish orqali **avtomatik** class property yaratish shorthand:

```typescript
// Verbose
class User {
  name: string;
  age: number;
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}

// Parameter properties — bir xil natija
class User {
  constructor(public name: string, public age: number) {}
}
```

**Qoida:** Modifier **SHART** — `public`, `private`, `protected`, yoki `readonly` biri bo'lishi kerak. Modifier siz parameter oddiy constructor argument:

```typescript
class Example {
  constructor(
    public name: string,  // ✅ Property bo'ladi
    age: number           // ❌ Property emas, faqat constructor da
  ) {}
}
new Example("Ali", 25).age; // ❌ Property 'age' does not exist
```

</details>

### 3. Abstract class nima? Interface dan qanday farqi bor?

<details>
<summary>Javob</summary>

Abstract class — `new` bilan to'g'ridan-to'g'ri instance yaratib bo'lmaydigan class. Ikki turdagi member bor:

1. **Abstract members** — faqat signature, subclass MAJBURIY implement qiladi
2. **Concrete members** — tayyor implementation, subclass meros oladi

```typescript
abstract class Shape {
  abstract area(): number;
  describe(): string { return `Area: ${this.area()}`; }
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  area(): number { return Math.PI * this.radius ** 2; }
}

new Shape();   // ❌ abstract class dan instance yaratib bo'lmaydi
new Circle(5); // ✅
```

| Xususiyat | Abstract Class | Interface |
|-----------|:-----------:|:---------:|
| Concrete method | ✅ | ❌ |
| Constructor | ✅ | ❌ |
| Access modifiers | ✅ | ❌ |
| Runtime da mavjud | ✅ | ❌ (o'chiriladi) |
| Multiple inherit | ❌ Faqat 1 | ✅ Bir nechta |
| `instanceof` | ✅ | ❌ |

**Qachon nima:** Interface — faqat contract. Abstract class — shared implementation + contract (template method pattern).

</details>

### 4. `implements` nima qiladi? Nima uchun type inference bo'lmaydi?

<details>
<summary>Javob</summary>

`implements` — class ning ma'lum interface ga mos kelishini compile-time da tekshiradi. Lekin faqat **tekshiradi** — type bermaydi:

```typescript
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message) { // ❌ message: any — inference yo'q!
    console.log(message);
  }
}
```

Nima uchun inference yo'q?
1. Explicit annotation — kodni o'qigan developer type ni darhol ko'radi
2. Agar inference bo'lsa — xato interface da paydo bo'ladi (chalkash)
3. `implements` — "tekshiruv" vositasi, "type injection" emas

Compiled JS da `implements` butunlay o'chiriladi. `instanceof` bilan interface tekshirib bo'lmaydi:

```typescript
doc instanceof Printable; // ❌ 'Printable' only refers to a type
```

</details>

### 5. `override` keyword nima? `noImplicitOverride` bilan qanday ishlaydi?

<details>
<summary>Javob</summary>

`override` (TS 4.3+) — subclass da parent method ni qayta yozayotganingizni aniq belgilash. Ikki xil xatoni ushlaydi:

**1. Typo:**

```typescript
class Animal { speak(): string { return "..."; } }

class Dog extends Animal {
  override speck(): string { return "Woof"; }
  // ❌ 'speck' is not declared in base class — typo!
}
```

**2. Parent method o'chirilganda** — barcha `override` lar xato beradi.

`noImplicitOverride: true` bilan override qilish **MAJBURIY**:

```typescript
class Dog extends Animal {
  speak(): string { return "Woof"; }
  // ❌ Must have 'override' modifier

  override speak(): string { return "Woof"; } // ✅
}
```

Compiled JS da `override` o'chiriladi — faqat compile-time safety.

</details>

### 6. TS `private` vs ES `#` private — qachon qaysi birini ishlatish kerak?

<details>
<summary>Javob</summary>

Ikki farqli mexanizm — compile-time vs runtime privacy:

| Xususiyat | TS `private` | ES `#` |
|-----------|:-----------:|:------:|
| Runtime privacy | ❌ | ✅ |
| `as any` bypass | Mumkin | Mumkin emas |
| Compiled (ES2022+) | Oddiy property | `#` private |
| Compiled (ES2015) | Oddiy property | WeakMap |
| JSON.stringify | Chiqadi | Chiqmaydi |

```typescript
class A { private x = 1; }
(new A() as any).x; // 1 — runtime da ochiq

class B { #x = 1; }
(new B() as any).#x; // SyntaxError — haqiqiy private
```

**Qachon nima:**
- **TS `private`** — aksariyat hollarda yetarli, sodda
- **ES `#`** — library yozayotganda (consumer JS dan foydalanishi mumkin), yoki haqiqiy encapsulation kerak bo'lganda

`target: ES2015` da `#` har bir field uchun WeakMap yaratadi — performance ta'siri bor. `ES2022+` da native `#` — overhead minimal.

</details>

### 7. TS class yaratganingizda nechta type hosil bo'ladi?

<details>
<summary>Javob</summary>

**Ikkita type** hosil bo'ladi:

1. **Instance type** — `User` — `new User()` bilan yaratilgan object type
2. **Constructor type** — `typeof User` — class ning o'zi (constructor function)

```typescript
class User {
  constructor(public name: string) {}
}

const user: User = new User("Ali");         // Instance type
const UserClass: typeof User = User;         // Constructor type
const user2 = new UserClass("Vali");         // ✅

// Farq muhim — class ni argument sifatida berishda:
function logClass(cls: User): void { }        // ❌ Instance kutadi
function logClass(cls: typeof User): void { }  // ✅ Constructor kutadi
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Constructor + field initialization order (Daraja: Middle)

**Savol:** Output ni ayting va nima uchun ekanini tushuntiring:

```typescript
class A {
  x: number = 1;
  constructor() {
    console.log("A constructor, x =", this.x);
    this.init();
  }
  init(): void {
    this.x = 2;
    console.log("A init, x =", this.x);
  }
}

class B extends A {
  y: number = 10;
  override init(): void {
    console.log("B init, y =", this.y);
    this.x = 3;
  }
}

const b = new B();
console.log("b.x =", b.x, "b.y =", b.y);
```

<details>
<summary>Yechim</summary>

```
A constructor, x = 1
B init, y = undefined
b.x = 3 b.y = 10
```

Execution order:

1. `new B()` → B da constructor yo'q, auto `super()` chaqiradi
2. A constructor: avval `this.x = 1` (field initializer) → `"A constructor, x = 1"`
3. `this.init()` chaqiriladi — **lekin** `this` B instance → **B.init()** ishlaydi
4. B.init da `this.y` → **undefined** — B ning field initializer lari hali ishlamagan!
5. `this.x = 3` assign bo'ladi
6. A constructor tugadi → B ga qaytadi → B field initializer: `this.y = 10`
7. Natija: `b.x = 3`, `b.y = 10`

**Dars:** Constructor da virtual method chaqirish **xavfli** — subclass field lari hali tayyor bo'lmagan bo'lishi mumkin. ES2022 field initializer lar `super()` qaytgandan **keyin** ishlaydi.

</details>

### 2. Structural typing — class bilan (Daraja: Middle)

**Savol:** Bu kodda xato bormi? `Dog` `Animal` ni extend qilmagan:

```typescript
class Animal {
  constructor(public name: string) {}
}

class Dog {
  constructor(public name: string) {}
  bark(): void { console.log("Woof!"); }
}

function printAnimal(animal: Animal): void {
  console.log(animal.name);
}

const dog = new Dog("Rex");
printAnimal(dog); // Ishlaydi mi?
```

<details>
<summary>Yechim</summary>

**Xato yo'q** — ishlaydi. TypeScript **structural typing** — `Dog` ning `Animal` ni extend qilmasligi muhim emas. `printAnimal` `{ name: string }` shape kutadi — `Dog` da `name: string` bor → mos keladi.

```typescript
printAnimal(dog);           // ✅ Dog shape mos
printAnimal({ name: "Cat" }); // ✅ Oddiy object ham mos
```

`bark()` qo'shimcha method — structural typing da extra member lar muammo emas.

Nominal typing (Java, C#) da `Dog extends Animal` yozilishi shart edi. TS da class nom va inheritance emas, **shape** muhim. Nominal kerak bo'lsa — branded types ishlatiladi.

</details>

### 3. Abstract class + generics — xatoni toping (Daraja: Middle+)

**Savol:** Bu kodda ikki mantiqiy xato bor. Toping va tuzating:

```typescript
abstract class Cache<T> {
  abstract get(key: string): T;
  abstract set(key: string, value: T): void;

  getOrSet(key: string, factory: () => T): T {
    const existing = this.get(key);
    if (existing) return existing;
    const value = factory();
    this.set(key, value);
    return value;
  }
}
```

<details>
<summary>Yechim</summary>

**Xato 1:** `if (existing)` — falsy value muammosi. `0`, `""`, `false`, `null` cache da bo'lsa — falsy tufayli "miss" deb hisoblanadi:

```typescript
const cache = new MemoryCache<number>();
cache.set("count", 0);
cache.getOrSet("count", () => 99); // 99 qaytaradi — 0 falsy!
```

**Xato 2:** `get` return type `T` — lekin key mavjud bo'lmaganda `undefined` qaytishi kerak.

```typescript
// ✅ Tuzatilgan
abstract class Cache<T> {
  abstract get(key: string): T | undefined;
  abstract set(key: string, value: T): void;
  abstract has(key: string): boolean;

  getOrSet(key: string, factory: () => T): T {
    if (this.has(key)) return this.get(key)!;
    const value = factory();
    this.set(key, value);
    return value;
  }
}

class MemoryCache<T> extends Cache<T> {
  private store = new Map<string, T>();
  get(key: string): T | undefined { return this.store.get(key); }
  set(key: string, value: T): void { this.store.set(key, value); }
  has(key: string): boolean { return this.store.has(key); }
}
```

`has()` qo'shish — falsy muammoni hal qiladi. `Map` — `Record` dan yaxshiroq (key mavjudligini aniq tekshiradi).

</details>

### 4. `this` type — fluent API (Daraja: Middle+)

**Savol:** `QueryBuilder` ning fluent chain'i buzilgan. Muammoni toping va tuzating:

```typescript
class QueryBuilder {
  private query: string[] = [];

  select(fields: string): QueryBuilder {
    this.query.push(`SELECT ${fields}`);
    return this;
  }

  from(table: string): QueryBuilder {
    this.query.push(`FROM ${table}`);
    return this;
  }

  build(): string { return this.query.join(" "); }
}

class UserQueryBuilder extends QueryBuilder {
  activeOnly(): UserQueryBuilder {
    this.query.push("WHERE active = true"); // ❌ private!
    return this;
  }
}

// Muammo: chain buziladi
new UserQueryBuilder()
  .select("*")     // → QueryBuilder (UserQueryBuilder emas!)
  .activeOnly();   // ❌ Property 'activeOnly' does not exist on type 'QueryBuilder'
```

<details>
<summary>Yechim</summary>

**Muammo:** `select()` va `from()` return type `QueryBuilder` — subclass type yo'qoladi. Chain dan keyin `UserQueryBuilder` method lariga kirish mumkin emas.

```typescript
class QueryBuilder {
  protected query: string[] = []; // protected — subclass kirishiga ruxsat

  select(fields: string): this { // ← this type
    this.query.push(`SELECT ${fields}`);
    return this;
  }

  from(table: string): this {
    this.query.push(`FROM ${table}`);
    return this;
  }

  build(): string { return this.query.join(" "); }
}

class UserQueryBuilder extends QueryBuilder {
  activeOnly(): this {
    this.query.push("WHERE active = true");
    return this;
  }
}

// ✅ Chain ishlaydi
new UserQueryBuilder()
  .select("*")     // → UserQueryBuilder (this)
  .from("users")   // → UserQueryBuilder (this)
  .activeOnly()    // → UserQueryBuilder ✅
  .build();        // "SELECT * FROM users WHERE active = true"
```

**Tushuntirish:**

- `this` return type — `QueryBuilder` da `QueryBuilder`, `UserQueryBuilder` da `UserQueryBuilder` ga resolve bo'ladi
- `private query` → `protected query` — subclass da ishlatish uchun
- Polymorphic `this` — compile-time feature, Builder, Fluent API pattern larda muhim

</details>

### 5. Singleton pattern (Daraja: Middle+)

**Savol:** `Database` singleton class yozing — faqat bitta instance bo'lishi mumkin. `private constructor`, `static getInstance()` ishlatilsin:

```typescript
// Implement qiling:
// const db1 = Database.getInstance("postgres://localhost/mydb");
// const db2 = Database.getInstance();
// db1 === db2 → true
// new Database("...") → ❌ compile error
```

<details>
<summary>Yechim</summary>

```typescript
class Database {
  private static instance: Database | null = null;

  private constructor(
    private readonly connectionString: string
  ) {}

  static getInstance(connectionString?: string): Database {
    if (!Database.instance) {
      if (!connectionString) {
        throw new Error("Connection string required for first init");
      }
      Database.instance = new Database(connectionString);
    }
    return Database.instance;
  }

  query(sql: string): void {
    console.log(`Executing on ${this.connectionString}: ${sql}`);
  }
}

const db1 = Database.getInstance("postgres://localhost/mydb");
const db2 = Database.getInstance();
console.log(db1 === db2); // true

// new Database("..."); // ❌ Constructor is private
```

**Tushuntirish:**

- `private constructor` — tashqaridan `new` ni compile-time da to'xtatadi
- `static instance` — bitta shared instance
- `getInstance()` — lazy initialization, birinchi chaqiruvda yaratadi
- TS `private` JS da o'chiriladi — runtime da `new` to'xtatish uchun qo'shimcha tekshiruv kerak bo'lishi mumkin

</details>

---

## Xulosa

- Access modifiers (`public`/`private`/`protected`) — faqat compile-time, JS da o'chiriladi
- ES `#` private — haqiqiy runtime privacy, library yozayotganda afzal
- Parameter properties — constructor da modifier bilan avtomatik property
- Abstract class — shared implementation + contract. Interface — faqat contract
- `implements` — tekshiruv, type injection emas. Inference bermaydi
- `override` — typo va parent o'zgarganda xatoni ushlaydi
- `this` return type — fluent API / builder pattern uchun muhim
- Class = 2 ta type: instance type (`User`) va constructor type (`typeof User`)
- Generic Repository — batafsil [interview/21 — Design Patterns](21-design-patterns.md) da

[Asosiy bo'limga qaytish →](../10-classes.md)
