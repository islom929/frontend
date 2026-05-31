# Interview: Classes TypeScript da

> Access modifiers, parameter properties, abstract classes, implements, override, this type, readonly, static members, generics, accessor, TS private vs ES `#` private bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar)
- [Amaliy savollar (Coding Challenges)](#amaliy-savollar-coding-challenges)

---

## Nazariy savollar

### Savol 1: TypeScript class'da `public`, `private`, `protected` farqi nima? Runtime'da nima bo'ladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Uchta access modifier compile-time'da member visibility'ni cheklaydi. JS'ga compile bo'lganda **butunlay o'chiriladi** — runtime'da hech qanday tekshiruv qolmaydi.

### To'liq tushuntirish

| Modifier | Class ichida | Subclass'da | Tashqarida |
|----------|:----------:|:-----------:|:----------:|
| `public` (default) | Ruxsat | Ruxsat | Ruxsat |
| `protected` | Ruxsat | Ruxsat | Taqiq |
| `private` | Ruxsat | Taqiq | Taqiq |

Access modifier'lar TypeScript'ning **type system** xususiyati. Compiler tekshiradi, lekin `tsc` `.js` faylga yozganda barcha modifier'lar o'chiriladi. Bu sabab — JavaScript spec'ida bunday modifier'lar yo'q, compiler emit qila olmaydi.

Haqiqiy runtime privacy uchun ECMAScript `#` private fields (Stage 4, ES2022) ishlatiladi. `#field` lexical scoping orqali implement qilingan — class declaration tashqarisidan parse-time'da reject bo'ladi.

### Kod misol

```typescript
class PaymentProcessor {
  private apiKey: string = "sk_live_abc123";
  protected merchantId: number = 42;
  public name: string = "Stripe";
}

const processor = new PaymentProcessor();
// (processor as any).apiKey — runtime'da hech qanday cheklov yo'q
console.log((processor as any).apiKey); // → "sk_live_abc123" — ochiq!

// ES `#` private — runtime hard privacy
class SecurePaymentProcessor {
  #apiKey: string = "sk_live_abc123";

  charge(amount: number): void {
    console.log(`Charging ${amount} with ${this.#apiKey}`); // ✅ class ichida
  }
}

const secure = new SecurePaymentProcessor();
// (secure as any).#apiKey — parse-time SyntaxError:
// `#apiKey` declarator scope tashqarisida — `as any` cast bu lexical
// cheklovni chetlab o'tolmaydi
```

### Edge Cases

- **Soft private bypass:** `(obj as any).private` har doim ishlaydi — TS modifier compile-time emit'dan keyin yo'q
- **Subclass override:** subclass parent'ning `private` member'ini override qila olmaydi — compile error, lekin JS'da hech qanday cheklov yo'q
- **Cross-instance access:** TS `private` ham, `#private` ham bir xil class'ning boshqa instance'iga kirishga ruxsat beradi (`this.other.#privateField`). Cheklov — declaring class lexical scope: **boshqa** class'dan kirish ikkalasida ham taqiq. `#privateField` ni declaring class'dan tashqarida yozishning iloji yo'q — `#name` syntax faqat shu nomni e'lon qilgan class body ichida lexical scope'da ko'rinadi, tashqarida parse-time SyntaxError. TS `private` esa compile error
- **JSON.stringify:** TS `private` enumerable — JSON'da chiqadi. `#private` har doim chiqmaydi
- **DevTools:** TS `private` browser DevTools'da oddiy property sifatida ko'rinadi

### Follow-up savollar

1. "Production secret'ni saqlash kerak bo'lsa `private` yetarlimi?" — Yo'q. Server-side env variable yoki Vault kabi secret manager. `private` faqat IDE'da accidental misuse'ni oldini oladi
2. "Library ishlab chiqayotgan bo'lsam qaysi birini ishlataman?" — `#private` — consumer JS'dan import qilishi mumkin, TS `private` JS'da yo'q

</details>

---

### Savol 2: Parameter properties nima? Modifier siz parameter qaysi farqi bor? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Constructor parametrida access modifier (`public`/`private`/`protected`) yoki `readonly` yozish — TypeScript shorthand. Class property avtomatik yaratiladi va assign bo'ladi.

### To'liq tushuntirish

Modifier **SHART** — `public`, `private`, `protected`, `readonly` biri. Aralash kombinatsiya ham mumkin (`public readonly`, `private readonly`). Modifier siz parameter oddiy constructor argument — class instance'da property bo'lmaydi.

Compiled JS'da TypeScript constructor body'ga `this.x = x` qatorlarini avtomatik qo'shadi. Modifier emit'dan keyin yo'qoladi.

### Kod misol

```typescript
// Verbose syntax
class OrderProcessor {
  userId: number;
  amount: number;
  readonly currency: string;

  constructor(userId: number, amount: number, currency: string) {
    this.userId = userId;
    this.amount = amount;
    this.currency = currency;
  }
}

// Parameter properties — bir xil natija
class OrderProcessor2 {
  constructor(
    public userId: number,
    public amount: number,
    public readonly currency: string,
  ) {}
}

const order = new OrderProcessor2(1, 100, "USD");
console.log(order.userId);   // → 1
console.log(order.currency); // → "USD"
// order.currency = "EUR"; // ❌ readonly

// Modifier siz parameter — property emas
class InvoiceBuilder {
  public total: number;
  constructor(items: number[]) {
    // `items` faqat constructor scope'ida — instance property emas
    this.total = items.reduce((sum, x) => sum + x, 0);
  }
}
const inv = new InvoiceBuilder([10, 20, 30]);
console.log(inv.total); // → 60
// console.log((inv as any).items); // → undefined
```

### Edge Cases

- **`useDefineForClassFields: true`** (TS 3.7+, target ES2022+ da default `true`) — field'lar `Object.defineProperty` (`[[Define]]`) semantics bilan emit bo'ladi. Parameter property esa constructor body'da `this.x = x` orqali assign bo'ladi, shu sabab oddiy own property bo'lib qoladi
- **Inherit holatda:** parent'ning parameter property'sini subclass `super()` orqali init qiladi — subclass'da qayta yozish kerak emas
- **Mix qilish:** parameter property va oddiy parameter aralash bo'lishi mumkin: `constructor(public id: number, options: Options)` — `id` property, `options` lokal
- **Decorator bilan:** Stage 3 ECMAScript decorators (TS 5.0+) parameter decorator'ni umuman qo'llab-quvvatlamaydi. Parameter decorator faqat legacy `experimentalDecorators: true` rejimida mavjud — u yerda parameter property'ga ham qo'llanadi

### Follow-up savollar

1. "`readonly` parameter property mutate bo'ladimi?" — Class ichida constructor'dan tashqari joyda yo'q. Constructor ichida ruxsat
2. "Inheritance'da parent parameter property override bo'ladimi?" — Field reassignment shaklida `super()` chaqirilgandan keyin

</details>

---

### Savol 3: Abstract class nima? Interface'dan qanday farqi bor? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Abstract class — `new` bilan to'g'ridan-to'g'ri instantiate qilib bo'lmaydigan class. Abstract members (faqat signature, subclass implement qilishi shart) va concrete members (tayyor implementation) aralash bo'lishi mumkin. Interface — faqat type-level contract, runtime'da yo'q.

### To'liq tushuntirish

Abstract class runtime'da real JS class sifatida mavjud — `instanceof` ishlaydi. Interface compile-time only — JS emit'da butunlay yo'q.

| Xususiyat | Abstract Class | Interface |
|-----------|:-----------:|:---------:|
| Concrete method | Ruxsat | Taqiq |
| Constructor | Ruxsat | Taqiq |
| Access modifiers | Ruxsat | Taqiq |
| Runtime'da mavjud | Ha | Yo'q (o'chiriladi) |
| Multiple inheritance | Faqat 1 | Bir nechta |
| `instanceof` ishlaydi | Ha | Yo'q |
| Declaration merging | Yo'q | Ha |

**Qachon abstract class:** shared implementation kerak (Template Method pattern), instantiate qilinmasligi shart, `protected` member'lar bilan subclass'ga API berish. **Qachon interface:** faqat contract, multiple inheritance kerak, structural typing yetarli.

### Kod misol

```typescript
abstract class PaymentGateway {
  constructor(protected readonly apiKey: string) {}

  // Abstract — subclass implement qilishi SHART
  abstract charge(amount: number, currency: string): Promise<string>;

  // Concrete — shared logic
  async chargeWithLog(amount: number, currency: string): Promise<string> {
    console.log(`[${this.constructor.name}] charging ${amount} ${currency}`);
    const txId = await this.charge(amount, currency);
    console.log(`[${this.constructor.name}] tx: ${txId}`);
    return txId;
  }
}

class StripeGateway extends PaymentGateway {
  async charge(amount: number, currency: string): Promise<string> {
    return `stripe_tx_${Date.now()}`;
  }
}

// new PaymentGateway("..."); // ❌ Cannot create an instance of an abstract class
const gateway = new StripeGateway("sk_test");
await gateway.chargeWithLog(100, "USD");
// → [StripeGateway] charging 100 USD
// → [StripeGateway] tx: stripe_tx_...

// Abstract construct signature (TS 4.2+)
type GatewayCtor = abstract new (apiKey: string) => PaymentGateway;
function registerGateway(Ctor: GatewayCtor): void { /* ... */ }
```

### Edge Cases

- **Abstract construct signature** (TS 4.2+): `abstract new (...) => T` — abstract class'ni constructor parameter sifatida qabul qilish
- **Abstract member visibility:** abstract member `public`/`protected` bo'lishi mumkin, `private` taqiq (subclass implement qila olmaydi)
- **Abstract static taqiq:** `abstract static` member TypeScript'da ruxsat etilmaydi — `'static' modifier cannot be used with 'abstract' modifier` (TS1243). Bu hali ham ochiq feature request
- **Empty abstract class:** abstract member yo'q bo'lsa ham `new` taqiq — declaration intent muhim

### Follow-up savollar

1. "Abstract class va `protected constructor` farqi?" — `protected constructor` instantiate qilinishini cheklaydi, lekin tasodifan subclass'da `new` mumkin. Abstract — har qanday `new` taqiq
2. "Interface ham `instanceof` orqali tekshirilsa-chi?" — Yo'q — interface JS'da yo'q. Type guard (`is`) yoki branded type kerak

<details>
<summary><strong>Deep Dive</strong></summary>

Abstract class emit'da har qanday boshqa class kabi yoziladi — `abstract` keyword tashlanadi. Yagona farq — TypeScript constructor call'ni compile-time'da bloklaydi. Runtime'da `Reflect.construct(AbstractClass, [])` muvaffaqiyatli ishlaydi (TS himoyasidan tashqarida).

`abstract` ECMAScript spec'ida mavjud emas — bu sof TypeScript type-system construct'i. Emit qilingan JS'da abstract class oddiy class, abstract method esa umuman yozilmaydi (faqat signature). Shu sabab `new`-ni bloklash kafolati faqat `tsc` compile bosqichida — JS-darajadagi hech qanday native cheklov yo'q.

</details>

</details>

---

### Savol 4: `implements` nima qiladi? Nima uchun type inference bo'lmaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`implements` — class'ning interface contract'iga mosligini compile-time'da tekshiradi. Faqat tekshiruv — type bermaydi. Method signature'lari `any` bo'lib qoladi agar explicit annotate qilinmasa.

### To'liq tushuntirish

TypeScript dizayn qarori: `implements` "tekshiruv", "type injection" emas. Inference bo'lsa, xato message interface'da paydo bo'lar va developer chalkash bo'lar edi. Explicit type — kodni o'qigan developer interface'ga qarab signature'ni topish o'rniga method'da darhol ko'radi.

Compiled JS'da `implements` butunlay o'chiriladi. `instanceof` bilan interface tekshirib bo'lmaydi — interface JS'da yo'q.

### Kod misol

```typescript
interface CacheStore {
  get(key: string): string | null;
  set(key: string, value: string, ttl: number): void;
}

// ❌ implements faqat tekshiradi, type bermaydi
class RedisCacheBad implements CacheStore {
  get(key) { // ❌ Parameter 'key' implicitly has 'any' type (strict mode)
    return null;
  }
  set(key, value, ttl) { /* any, any, any */ }
}

// ✅ Explicit annotation
class RedisCache implements CacheStore {
  private store = new Map<string, { value: string; expiresAt: number }>();

  get(key: string): string | null {
    const entry = this.store.get(key);
    if (!entry || entry.expiresAt < Date.now()) return null;
    return entry.value;
  }

  set(key: string, value: string, ttl: number): void {
    this.store.set(key, { value, expiresAt: Date.now() + ttl });
  }
}

// implements emit'da yo'q
// const cache = new RedisCache();
// cache instanceof CacheStore; // ❌ 'CacheStore' only refers to a type, not a value
```

### Edge Cases

- **Multiple implements:** `class X implements A, B, C` — har bir interface contract'iga rioya qilish kerak
- **`implements` va inheritance:** `class X extends Y implements Z` — Y'dan member meros oladi, Z contract'iga rioya qilishi shart
- **Method signature variance:** method shaklida (`get(key: string): T`) yozilgan signature parameter'lari `strictFunctionTypes` yoqilgan bo'lsa ham **bivariant** tekshiriladi — `strictFunctionTypes` faqat function-type property'lar (`get: (key: string) => T`) ga ta'sir qiladi, ularni contravariant qiladi. Return type har doim covariant
- **Optional member:** interface'da `prop?: T` — class'da yozilmasa ham OK
- **Index signature:** interface'da `[key: string]: T` — class'da har bir property `T` ga mos bo'lishi shart

### Follow-up savollar

1. "Class extends class + implements interface ishlaydimi?" — Ha. `class X extends Base implements I1, I2`. Inheritance bittada, interface ko'p
2. "Interface'ni class sifatida ham implement qilish mumkinmi?" — TS interface'lar nominal emas, structural — har qanday matching shape OK

</details>

---

### Savol 5: `override` keyword nima? `noImplicitOverride` qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`override` (TS 4.3+) — subclass'da parent method'ni qayta yozayotganligini explicit belgilash. Typo (`speack` o'rniga `speak`) va parent method olib tashlanganda barcha override'larni xato bilan ushlaydi.

### To'liq tushuntirish

`noImplicitOverride: true` (tsconfig) bilan parent method'ni qayta yozish'da `override` keyword MAJBURIY bo'ladi. Default `false`. Production codebase'larda har doim yoqilishi tavsiya — refactoring xatolarini compile-time'da ushlaydi.

`override` JS emit'da yo'q — faqat compile-time safety.

### Kod misol

```typescript
class PaymentGateway {
  charge(amount: number): string {
    return `charged ${amount}`;
  }

  refund(transactionId: string): boolean {
    return true;
  }
}

class StripeGateway extends PaymentGateway {
  // ✅ To'g'ri override
  override charge(amount: number): string {
    return `stripe: charged ${amount}`;
  }

  // ❌ Typo — parent'da bunday method yo'q
  override refnud(txId: string): boolean { // ❌ Error TS4117
    return false;
  }
}

// noImplicitOverride: true bilan
class PaypalGateway extends PaymentGateway {
  charge(amount: number): string { // ❌ Error: This member must have an 'override' modifier
    return `paypal: charged ${amount}`;
  }

  override charge(amount: number): string { // ✅
    return `paypal: charged ${amount}`;
  }
}

// Parent method olib tashlanganda
// PaymentGateway'dan `refund` o'chirilsa →
// barcha subclass'lardagi `override refund` xato beradi (refactoring safety)
```

### Edge Cases

- **Method va property override:** `override` har ikkalasi uchun ishlaydi
- **Static method:** static method ham `override` qabul qiladi (TS 4.3+)
- **Abstract member:** abstract'ni implement qilganda `override` shart emas, lekin yozish ruxsat
- **Constructor:** constructor'da `override` mavjud emas — har subclass o'z constructor'iga ega
- **Accessor (getter/setter):** override qo'llaniladi

### Follow-up savollar

1. "Parent va child da signature farqli bo'lsa?" — Compile error. Bivariant param/covariant return constraint
2. "Multiple inheritance bo'lsa-chi?" — JS'da yo'q. Mixin pattern bilan oxirgi mixin'dagi method override hisoblanadi

</details>

---

### Savol 6: TS `private` vs ES `#` private — qachon qaysi birini ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TS `private` — compile-time soft privacy, JS emit'da o'chiriladi. ES `#` private — runtime hard privacy, lexical scoping orqali implement qilingan. Library yozayotganda yoki haqiqiy encapsulation kerak bo'lsa `#`. Aksariyat application kodlarida TS `private` yetarli.

### To'liq tushuntirish

| Xususiyat | TS `private` | ES `#` |
|-----------|:-----------:|:------:|
| Runtime privacy | Yo'q | Ha |
| `(obj as any).x` bypass | Mumkin | Mumkin emas |
| Compiled (target ES2022+) | Oddiy property | Native `#` |
| Compiled (target < ES2022) | Oddiy property | WeakMap shim |
| `JSON.stringify` | Chiqadi | Chiqmaydi |
| `Object.keys` | Chiqadi | Chiqmaydi |
| Cross-instance access | Ruxsat | Ruxsat (same class) |
| `in` operator | Default ko'rinadi | `#x in obj` syntax bilan |

`target: ES2015` da `#` har bir field uchun WeakMap shim'ga compile bo'ladi — har access qo'shimcha WeakMap lookup. `ES2022+` native `#` esa engine'ning o'z private field mexanizmiga emit bo'ladi, WeakMap shim qatlamisiz.

### Kod misol

```typescript
// TS private — soft
class TsPaymentProcessor {
  private apiKey: string = "sk_live_test";
  charge(): void { console.log(this.apiKey); }
}

const tsProc = new TsPaymentProcessor();
console.log((tsProc as any).apiKey); // → "sk_live_test" — ochiq
console.log(JSON.stringify(tsProc));  // → {"apiKey":"sk_live_test"}

// ES # private — hard
class EsPaymentProcessor {
  #apiKey: string = "sk_live_test";
  charge(): void { console.log(this.#apiKey); }

  static brandCheck(obj: unknown): boolean {
    return #apiKey in (obj as object); // Stage 4 brand check
  }
}

const esProc = new EsPaymentProcessor();
console.log(JSON.stringify(esProc)); // → {} — bo'sh
console.log(Object.keys(esProc));    // → [] — bo'sh
// (esProc as any).#apiKey;          // ❌ Parse-time SyntaxError

console.log(EsPaymentProcessor.brandCheck(esProc));      // → true
console.log(EsPaymentProcessor.brandCheck({}));          // → false
```

### Edge Cases

- **Mix qilish:** bitta class'da `private` va `#private` ikkalasi ishlatilishi mumkin, lekin chalkash
- **WeakMap shim performance:** `target: ES2015` da `#` access har biri WeakMap lookup — hot loop'da sezilarli overhead
- **Subclass `#name` conflict:** subclass parent'ning `#name` ga kira olmaydi (lexical scope cheklovi)
- **Reflection cheklovi:** `Object.getOwnPropertyNames`, `Reflect.ownKeys` va `Object.getOwnPropertySymbols` `#` field'ni qaytarmaydi
- **DevTools:** `#field` Chrome DevTools'da ko'rsatiladi (debugger uchun), lekin user code'dan kira olmaydi

### Follow-up savollar

1. "Performance critical kodda qaysi?" — `target: ES2022+` bo'lsa `#` overhead minimal. Eski target — TS `private` (oddiy property)
2. "Class'imni serialize qilishni xohlayman, lekin secret saqlash kerak — qaysi?" — `#` — `JSON.stringify` avtomatik secret'ni qoldiradi

<details>
<summary><strong>Deep Dive</strong></summary>

ECMAScript spec'da `#field` Private Name sifatida modellanadi — bu nom faqat class lexical scope ichida ko'rinadi va object'ning oddiy property key'lari (string/symbol) ro'yxatiga kirmaydi. Shu sabab `Object.keys`, `JSON.stringify`, `Reflect.ownKeys` uni qaytarmaydi. Class instance yaratilganda unga private brand biriktiriladi; private name'ga har access shu brand mavjudligini tekshiradi va yo'q bo'lsa runtime'da `TypeError` beradi.

TS `private` esa type-system modifier — emit phase'da butunlay olib tashlanadi va JS'da hech qanday iz qoldirmaydi. `tsc --target es2022` `#` ni native sifatida emit qiladi, `--target es2015` esa downlevel WeakMap shim ishlatadi:

```javascript
// target: es2015 uchun `#field` emit (soddalashtirilgan):
var _ApiKey = new WeakMap();
class EsPaymentProcessor {
  constructor() { _ApiKey.set(this, "sk_live_test"); }
  charge() { console.log(_ApiKey.get(this)); }
}
```

Brand check (`#x in obj`) shu private brand mavjudligini xavfsiz, exception'siz tekshiradi — `instanceof` o'rniga ishlatilishi mumkin (lekin asosiy maqsadi bu emas).

</details>

</details>

---

### Savol 7: TS class yaratganingizda nechta type hosil bo'ladi? Farqi qanaqa? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Class declaration ikkita type hosil qiladi: **instance type** (`OrderEntity` — `new OrderEntity()` natijasi) va **constructor type** (`typeof OrderEntity` — class'ning o'zi). Function parametrida class'ni qabul qilganda farq muhim.

### To'liq tushuntirish

Class declaration ham value (constructor function), ham type space'ga qo'shiladi. Type space'da:

- `OrderEntity` — instance shape
- `typeof OrderEntity` — constructor signature + static member'lar

Generic factory function'larda class'ni argument sifatida qabul qilganda `typeof OrderEntity` (yoki `new (...) => OrderEntity`) kerak.

### Kod misol

```typescript
class OrderEntity {
  static tableName = "orders";
  constructor(public id: number, public total: number) {}

  toJSON(): object {
    return { id: this.id, total: this.total };
  }
}

// Instance type — OrderEntity
const order: OrderEntity = new OrderEntity(1, 100);
order.toJSON(); // ✅

// Constructor type — typeof OrderEntity
const EntityClass: typeof OrderEntity = OrderEntity;
console.log(EntityClass.tableName);   // → "orders" (static access)
const order2 = new EntityClass(2, 200); // ✅

// Function parameter farqi
function logInstance(entity: OrderEntity): void {
  console.log(entity.id);
}
logInstance(new OrderEntity(1, 100)); // ✅ instance kutadi

function registerEntity<T extends new (...args: any[]) => any>(Ctor: T): void {
  // Constructor type — `new` qilish mumkin, static'ga kirish mumkin
  console.log((Ctor as any).tableName);
}
registerEntity(OrderEntity); // ✅

// Abstract construct signature (TS 4.2+)
abstract class BaseRepo {
  abstract save(): void;
}
type AbstractCtor = abstract new (...args: any[]) => BaseRepo;
function registerRepo(Repo: AbstractCtor): void { /* ... */ }
```

### Edge Cases

- **`InstanceType<typeof Class>`** — constructor type'dan instance type olish
- **`ConstructorParameters<typeof Class>`** — constructor parametrlar tuple'i
- **Abstract class** — `new` qilib bo'lmaydi, lekin `typeof AbstractClass` constructor type — `abstract new` signature kerak
- **Class expression:** `const X = class { ... }` — `X` ham value, ham type. `typeof X` constructor

### Follow-up savollar

1. "Factory'da `new Ctor()` xato bersa nima qilish kerak?" — Ctor signature'ga rioya qilish: `new (...args: any[]) => T`
2. "`InstanceType` va `T extends new (...) => infer R` farqi?" — `InstanceType` built-in utility, ichida `infer` ishlatadi — bir xil natija

</details>

---

### Savol 8: `readonly` class property nima qiladi? `const` bilan farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`readonly` — property faqat declaration yoki constructor ichida assign bo'ladi. Constructor'dan keyin reassignment compile error. Compile-time only — runtime'da `Object.defineProperty` semantics qo'llanmaydi.

### To'liq tushuntirish

`const` variable binding'ga, `readonly` object property'ga taalluqli. `const obj = { x: 1 }` — `obj` reassign qilib bo'lmaydi, lekin `obj.x = 2` mumkin. `readonly` esa property mutation'ni bloklaydi.

`readonly` modifier'i:
- Class field
- Interface property
- Type alias property
- Tuple element (`readonly [number, string]`)
- Array (`readonly T[]` yoki `ReadonlyArray<T>`)

`readonly` shallow — nested object mutate qilinishi mumkin. Deep readonly uchun `DeepReadonly<T>` utility kerak.

### Kod misol

```typescript
class UserProfile {
  readonly id: number;
  readonly createdAt: Date;
  name: string;

  constructor(id: number, name: string) {
    this.id = id;
    this.createdAt = new Date(); // ✅ constructor ichida ruxsat
    this.name = name;
  }

  rename(newName: string): void {
    this.name = newName;       // ✅ mutable
    // this.id = 999;          // ❌ Cannot assign to 'id' (readonly)
  }
}

const user = new UserProfile(1, "Ali");
// user.id = 2;                 // ❌ Cannot assign to 'id'
user.createdAt.setFullYear(2020); // ✅ shallow — Date mutate bo'ladi!

// Deep readonly utility
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

class ImmutableProfile {
  readonly settings: DeepReadonly<{ theme: { color: string } }>;
  constructor(settings: { theme: { color: string } }) {
    this.settings = settings;
  }
}
const prof = new ImmutableProfile({ theme: { color: "dark" } });
// prof.settings.theme.color = "light"; // ❌ Cannot assign (deep readonly)
```

### Edge Cases

- **`as const`** — literal type lock + readonly. `[1, 2, 3] as const` → `readonly [1, 2, 3]`
- **Mutation via aliasing:** `function f(p: { x: number })` — `readonly` property'ni mutable parameter'ga bersa, mutation mumkin (TS strictreadonly emas)
- **`readonly` static field:** `static readonly` — class darajasida constant, declaration vaqtida init
- **Parameter property:** `constructor(public readonly id: number)` — shorthand
- **Compile-time only:** runtime'da `obj.readonlyField = 1` ishlaydi (TS strip qilingan)

### Follow-up savollar

1. "Runtime'da haqiqiy immutability kerak bo'lsa?" — `Object.freeze` (shallow) yoki Immer/Immutable.js library
2. "`readonly` array va `Array` farqi?" — `readonly T[]` `push`/`pop` taqiq, `T[]` mumkin

</details>

---

### Savol 9: Static member nima? Static initialization block (TS 4.4+) qachon kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Static member — class darajasiga biriktirilgan (instance'larga emas) property/method. `ClassName.staticMember` orqali kirilgan. Static initialization block (TS 4.4+, ES2022) — `static { ... }` ichida static field'larni init qilish uchun maxsus blok (asinxron emas, `this` static class'ga ishora qiladi).

### To'liq tushuntirish

Static member'lar shared state yoki utility method'lar uchun: `Math.PI`, `Array.isArray`. Constructor function ning property'lariga bog'lanadi (prototype'ga emas).

Static initialization block:
- Class declaration vaqtida bir marta ishlaydi
- Murakkab static init logic uchun (try/catch, conditional, private static field'larga kirish)
- `super` parent class'ga reference

### Kod misol

```typescript
class HttpClient {
  static baseUrl = "https://api.example.com";
  static defaultHeaders: Record<string, string>;
  static #instanceCounter = 0;

  // Static initialization block — TS 4.4+ / ES2022
  static {
    try {
      const env = process.env.API_BASE_URL;
      HttpClient.defaultHeaders = {
        "Content-Type": "application/json",
        "X-Client-Version": HttpClient.#computeVersion(),
      };
      if (env) HttpClient.baseUrl = env;
    } catch (e) {
      HttpClient.defaultHeaders = { "Content-Type": "application/json" };
    }
  }

  static #computeVersion(): string {
    return `v1.0.${HttpClient.#instanceCounter}`;
  }

  static request(path: string): Promise<Response> {
    return fetch(`${HttpClient.baseUrl}${path}`, {
      headers: HttpClient.defaultHeaders,
    });
  }
}

console.log(HttpClient.baseUrl); // → "https://api.example.com" yoki env

// Static method instance'siz chaqiriladi
HttpClient.request("/users");
```

### Edge Cases

- **`this` static method ichida:** static method ichida `this` — class'ning o'zi (`typeof Class`), instance emas
- **Static + inheritance:** subclass parent static member'ni meros oladi (`Child.parentStatic` ishlaydi)
- **`static override`** (TS 4.3+): static method override qilinganda `override` keyword ishlatish mumkin
- **Multiple static blocks:** bir class'da bir nechta static block — declaration tartibida ishlaydi
- **`#private` static:** static private field — `static #counter`. Lexical scope cheklovi qo'llaniladi

### Follow-up savollar

1. "Static field generic bo'lishi mumkinmi?" — Yo'q. Static class darajasida, generic T instance darajasida — TS bunga ruxsat bermaydi
2. "Singleton pattern'da static qanday ishlatiladi?" — `static instance: Class | null`, `static getInstance()`

</details>

---

### Savol 10: `accessor` keyword (TS 4.9+) nima qiladi? Oddiy field bilan farqi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`accessor` keyword (TS 4.9+, Stage 3 ECMAScript decorators uchun) — field'ni auto-accessor (getter/setter pair) sifatida e'lon qiladi. Decorator metadata'da getter/setter sifatida ko'rinishi kerak bo'lganda ishlatiladi.

### To'liq tushuntirish

Stage 3 ECMAScript decorators field'ga apply qilinganda decorator getter/setter pair olib, getter/setter pair qaytaradi. Oddiy field'ni decorate qilish mumkin emas — `accessor` keyword field'ni avtomatik getter/setter'ga aylantiradi.

Compile'da `accessor x` → private storage + getter + setter:

```javascript
// accessor x = 1; emit:
#__x = 1;
get x() { return this.#__x; }
set x(v) { this.#__x = v; }
```

### Kod misol

```typescript
function logged<T, V>(
  target: ClassAccessorDecoratorTarget<T, V>,
  context: ClassAccessorDecoratorContext<T, V>,
): ClassAccessorDecoratorResult<T, V> {
  return {
    get(this: T): V {
      const v = target.get.call(this);
      console.log(`Read ${String(context.name)}: ${String(v)}`);
      return v;
    },
    set(this: T, value: V): void {
      console.log(`Write ${String(context.name)}: ${String(value)}`);
      target.set.call(this, value);
    },
  };
}

class UserAccount {
  @logged accessor balance: number = 0;
}

const acc = new UserAccount();
acc.balance = 100;
// → Write balance: 100
console.log(acc.balance);
// → Read balance: 100
// → 100
```

### Edge Cases

- **`accessor` static:** `static accessor x = 1` — class darajasida auto-accessor
- **Inheritance:** subclass `override accessor` bilan parent accessor'ni qayta yozadi
- **`readonly accessor`** taqiq — getter/setter pair'ni half-readonly qilish mantiqsiz
- **Compatibility:** legacy decorators (`experimentalDecorators: true`) bilan ishlamaydi — faqat Stage 3
- **Performance:** har access getter/setter call — hot loop'da oddiy field'dan sekinroq

### Follow-up savollar

1. "Legacy decorators bilan `accessor` ishlaydimi?" — Yo'q. `experimentalDecorators: false` bo'lishi kerak
2. "Reflect metadata bilan integratsiya?" — Stage 3 decorator context'da `metadata` slot mavjud (TS 5.2+)

<details>
<summary><strong>Deep Dive</strong></summary>

Stage 3 decorators (TC39 proposal) accessor uchun maxsus shape belgilaydi. Decorator'ning **birinchi argumenti** — `ClassAccessorDecoratorTarget` — backing storage'ning `get`/`set` funksiyalarini beradi. **Ikkinchi argument** — `ClassAccessorDecoratorContext` — `name`, `kind: "accessor"`, `static`, `private` va `access` (`{ get, set, has }`) maydonlarini, hamda `addInitializer` ni ushlaydi. Decorator yangi `{ get?, set?, init? }` qaytarib backing accessor'ni o'rab oladi yoki initial value'ni transform qiladi.

Emit'da `accessor x = 1` private backing field + getter + setter'ga aylanadi, decorator esa shu pair ustida bir marta (class definition vaqtida) chaqiriladi. Aniq emit `tsc` versiyasi va helper'larga (`__esDecorate`, `__runInitializers`) bog'liq — versiyalar orasida o'zgaradi.

`accessor` field decorator uchun unified API beradi — old-style getter/setter manual yozishni almashtiradi va decorator'ga read/write ikkalasini ham intercept qilish imkonini beradi.

</details>

</details>

---

### Savol 11: Generic class qanday yoziladi? Constraint'lar bilan misol [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Generic class — class declaration'ga type parameter'lar qo'shish. `class Repository<T> { ... }`. Type parameter constructor, method, property type'larida ishlatilishi mumkin. Constraint (`T extends BaseEntity`) — parameter'ga shape talab qiladi.

### To'liq tushuntirish

Type parameter class darajasida e'lon qilinganda — instance method'lar va property'lar uchun foydalanish mumkin. Static member'larda type parameter ishlatib bo'lmaydi — static class darajasida, T instance darajasida.

Generic constraint'lar `extends` orqali — parameter'ga minimal shape majburiyati. Default type parameter (`T = string`) bo'sh argument'da fallback.

### Kod misol

```typescript
interface Entity {
  id: number;
}

class Repository<T extends Entity> {
  private items: Map<number, T> = new Map();

  save(entity: T): void {
    this.items.set(entity.id, entity);
  }

  findById(id: number): T | undefined {
    return this.items.get(id);
  }

  findAll(): T[] {
    return Array.from(this.items.values());
  }

  // Method-level generic
  filter<K extends keyof T>(key: K, value: T[K]): T[] {
    return this.findAll().filter(item => item[key] === value);
  }
}

interface UserEntity extends Entity {
  name: string;
  email: string;
}

const userRepo = new Repository<UserEntity>();
userRepo.save({ id: 1, name: "Ali", email: "ali@example.com" });
const found = userRepo.findById(1); // UserEntity | undefined
const byName = userRepo.filter("name", "Ali"); // UserEntity[]

// const bad = new Repository<{ name: string }>(); // ❌ id yo'q
```

### Edge Cases

- **Default type parameter:** `class Box<T = unknown> { ... }` — argument'siz `Box` → `Box<unknown>`
- **Multiple parameters:** `class Cache<K, V> { ... }` — har biri uchun constraint mumkin
- **Type parameter va `this`:** method `T` qaytarganda `this.value as T` bilan narrow
- **Static method generic:** static method o'z type parameter'iga ega bo'lishi mumkin, lekin class T'ga kira olmaydi
- **Variance:** generic class member'lar variance'ni belgilaydi (covariant return, contravariant param)

### Follow-up savollar

1. "Generic class abstract bo'lishi mumkinmi?" — Ha. `abstract class Repo<T> { abstract save(x: T): void; }`
2. "Constructor type'da generic'ni qanday saqlash mumkin?" — `typeof Class` generic argument'lariga ega bo'lmaydi — `new <T>(...) => Class<T>` shaklida ifoda

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 12: Constructor + field initialization order [Middle]

**Savol:** Output ni ayting va sababini tushuntiring:

```typescript
class Base {
  x: number = 1;
  constructor() {
    console.log("Base ctor, x =", this.x);
    this.init();
  }
  init(): void {
    this.x = 2;
    console.log("Base init, x =", this.x);
  }
}

class Derived extends Base {
  y: number = 10;
  override init(): void {
    console.log("Derived init, y =", this.y);
    this.x = 3;
  }
}

const d = new Derived();
console.log("d.x =", d.x, "d.y =", d.y);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
Base ctor, x = 1
Derived init, y = undefined
d.x = 3 d.y = 10
```

Constructor'da virtual method chaqirish xavfli — subclass field'lari hali initialize bo'lmagan.

### To'liq tushuntirish

Execution order:

1. `new Derived()` — Derived'da explicit constructor yo'q, auto `super()` chaqiradi
2. Base constructor: avval field initializer (`x = 1`) → `"Base ctor, x = 1"`
3. `this.init()` chaqiriladi — **virtual dispatch**: `this` Derived instance, `Derived.init()` ishlaydi
4. Derived.init: `this.y` → **undefined** — Derived'ning field initializer'lari hali ishlamagan (`super()` qaytmagan)
5. `this.x = 3` assign bo'ladi
6. Base constructor tugadi → Derived field initializer'lar: `this.y = 10`
7. Natija: `d.x = 3`, `d.y = 10`

### Kod misol

Target ES2022+ da (`useDefineForClassFields` default `true`) subclass field initializer'lar `super()` qaytgandan **keyin** ishlaydi. Bu sabab subclass field'lari Base constructor ichida `undefined` ko'rinadi.

### Edge Cases

- **`useDefineForClassFields: false`** (target ES2021 va past da default) — field assignment `[[Set]]` semantics bilan, parent'dagi accessor'larni trigger qiladi (`[[Define]]` esa qilmaydi)
- **Parameter property:** `constructor(public x: number)` — body'da explicit init kabi ishlaydi
- **Abstract method call:** abstract class constructor'da abstract method chaqirish ham xavfli

### Follow-up savollar

1. "Bu xatti-harakatdan qanday qochish mumkin?" — Constructor'da template method chaqirmaslik. Factory pattern yoki `init()` method'ni constructor'dan tashqarida explicit chaqirish
2. "Java/C# da ham bir xilmi?" — Ha — bu fundamental OOP gotcha "calling virtual methods from constructor"

</details>

---

### Savol 13: Structural typing — class bilan [Middle]

**Savol:** `Dog` `Animal` ni extend qilmagan. Bu kod ishlaydimi?

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
printAnimal(dog);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Xato yo'q — ishlaydi. TypeScript structural typing — class nominal emas, shape muhim. `Dog` `{ name: string }` shape'iga mos.

### To'liq tushuntirish

`printAnimal` `Animal` ni qabul qiladi, lekin TS strukturani solishtirgan: `Animal` shape — `{ name: string }`. `Dog` da `name: string` bor → mos keladi. Qo'shimcha `bark()` method strukturaviy typing'da muammo emas.

Nominal typing (Java, C#) — `Dog extends Animal` yozilishi shart edi. TS — `Dog` va `Animal` mustaqil class'lar, lekin shape mos kelganda interchangeable.

### Kod misol

```typescript
printAnimal(dog);                   // ✅ Dog shape mos
printAnimal({ name: "Cat" });       // ✅ Oddiy object ham mos

// Fresh object literal'da excess property check (EPC) ishlaydi:
// printAnimal({ name: "X", age: 3 }); // ❌ TS2353 — 'age' Animal'da yo'q

// Lekin oldin variable'ga assign qilinsa — EPC faqat fresh literal'ga
// qo'llanadi, shu sabab keng shape o'tib ketadi:
const extra = { name: "X", age: 3 };
printAnimal(extra);                 // ✅ structural mos (EPC fresh literal emas)

// Private/protected member'lar nominal qiladi
class SecureAnimal {
  private secret: string = "x";
  constructor(public name: string) {}
}

class OtherSecure {
  private secret: string = "y";
  constructor(public name: string) {}
}

function f(a: SecureAnimal): void {}
// f(new OtherSecure("x")); // ❌ private member'lar bir xil declaration'dan emas
```

### Edge Cases

- **Private member nominal effect:** `private` yoki `#private` bo'lsa — har class unique brand, faqat bir xil declaration'dan kelgan instance mos
- **Branded type:** `type Brand<T, B> = T & { __brand: B }` — nominal simulation
- **Class hierarchy:** structural typing'da `Animal | Dog` — bir xil shape bo'lsa subset qarash

### Follow-up savollar

1. "Nominal typing kerak bo'lsa?" — Branded type pattern: `type UserId = number & { __brand: "UserId" }`
2. "`instanceof` ishlaydimi?" — Ha — instanceof prototype chain tekshiradi, structural emas

</details>

---

### Savol 14: Abstract class + generics — xatoni toping [Middle+]

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
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Xato 1:** `if (existing)` falsy value muammosi — `0`, `""`, `false`, `null` cache'da bo'lsa "miss" deb hisoblanadi.
**Xato 2:** `get` return type `T` — key topilmasa `undefined` qaytishi kerak, lekin `T` `undefined` ni qamramaydi.

### To'liq tushuntirish

`abstract get(key: string): T` — implementation `T` qaytarishi shart, lekin Map'da key mavjud bo'lmasa `undefined` qaytadi. Return type `T | undefined` bo'lishi kerak.

Falsy check `if (existing)` — cache'da `0` saqlangan bo'lsa, har chaqiriqda factory chaqiriladi (regression bug). To'g'ri yondashuv: `has(key)` method qo'shish.

### Kod misol

```typescript
// ✅ Tuzatilgan
abstract class Cache<T> {
  abstract get(key: string): T | undefined;
  abstract set(key: string, value: T): void;
  abstract has(key: string): boolean;

  getOrSet(key: string, factory: () => T): T {
    if (this.has(key)) {
      const existing = this.get(key);
      if (existing !== undefined) return existing;
    }
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

const cache = new MemoryCache<number>();
cache.set("count", 0);
console.log(cache.getOrSet("count", () => 99)); // → 0 (xato yo'q!)
```

`has(key)` gate `0`/`""`/`false` kabi falsy qiymatlarni "miss" deb hisoblamaydi. `get` ning `T | undefined` natijasini `existing !== undefined` bilan narrow qilish `!` non-null assertion'siz `T` ga olib keladi.

### Edge Cases

- **`T` ichida `undefined`:** agar `T` o'zi `undefined` ni qamrasa, `existing !== undefined` narrowing aralashadi — bunday holatda `has` natijasini sentinel sifatida ishlatish kerak
- **Async cache:** `async get(key): Promise<T | undefined>` — `getOrSet` ham async, race protection kerak
- **Eviction policy:** LRU, TTL — `has` true qaytarishidan oldin TTL tekshirilishi kerak

### Follow-up savollar

1. "Concurrent `getOrSet` chaqiruvlar bo'lsa-chi?" — Promise saqlash: `Map<string, Promise<T>>` — duplicate factory'ni oldini olish
2. "`null` ni cache'lash kerak bo'lsa?" — `T | null | undefined` aralash — `has` yagona to'g'ri yo'l

</details>

---

### Savol 15: `this` type — fluent API [Middle+]

**Savol:** `QueryBuilder` chain'i buzilgan. Muammoni toping va tuzating:

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

new UserQueryBuilder()
  .select("*")     // → QueryBuilder
  .activeOnly();   // ❌ does not exist on type 'QueryBuilder'
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`select()` va `from()` return type `QueryBuilder` — subclass type yo'qoladi. Yechim: `this` polymorphic return type + `protected query` (subclass uchun access).

### To'liq tushuntirish

`this` return type — polymorphic. `QueryBuilder` da `QueryBuilder`, `UserQueryBuilder` da `UserQueryBuilder` ga resolve bo'ladi. Chain'da subclass method'lari accessible bo'lib qoladi.

`private` → `protected` — subclass'ga `query` field'ga kirishga ruxsat berish.

### Kod misol

```typescript
class QueryBuilder {
  protected query: string[] = [];

  select(fields: string): this {
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

const sql = new UserQueryBuilder()
  .select("*")     // this → UserQueryBuilder
  .from("users")   // this → UserQueryBuilder
  .activeOnly()    // ✅
  .build();        // → "SELECT * FROM users WHERE active = true"
```

### Edge Cases

- **`this` type narrowing:** `this is SubType` type predicate orqali narrow qilish mumkin
- **Method nashr qilish:** `this` polymorphic — subclass har doim o'z type'iga ega bo'lib qoladi
- **`return this`** SHART — method `this` qaytarishga e'lon qilinib `this` ni qaytarmasa, return type mos kelmaydi va compile error beradi (chain'ning keyingi bo'g'ini `undefined` ustida ishlar edi)

### Follow-up savollar

1. "`this` type qachon foydali emas?" — Subclass'da method explicit boshqa subclass type qaytarsa, polymorphic break
2. "Builder pattern bilan farq?" — `this` chaining (mutable). Type-state builder — har chaqiriq yangi type qaytaradi (immutable type progression)

</details>

---

### Savol 16: Singleton pattern [Middle+]

**Savol:** `DatabaseConnection` singleton class yozing — faqat bitta instance bo'lishi mumkin. `private constructor`, `static getInstance()` ishlatilsin.

```typescript
// const db1 = DatabaseConnection.getInstance("postgres://localhost/mydb");
// const db2 = DatabaseConnection.getInstance();
// db1 === db2 → true
// new DatabaseConnection("...") → ❌ compile error
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`private constructor` — tashqaridan `new` ni compile-time'da bloklaydi. `static instance` — shared instance, `getInstance()` lazy initialization.

### To'liq tushuntirish

`private constructor` faqat compile-time himoya. Haqiqiy runtime privacy uchun `#private` static field + factory function ham kerak (chunki TS `private` JS'da o'chiriladi va `Reflect.construct` ishlaydi).

### Kod misol

```typescript
class DatabaseConnection {
  static #instance: DatabaseConnection | null = null;

  private constructor(
    private readonly connectionString: string,
  ) {}

  static getInstance(connectionString?: string): DatabaseConnection {
    if (!DatabaseConnection.#instance) {
      if (!connectionString) {
        throw new Error("Connection string required for first init");
      }
      DatabaseConnection.#instance = new DatabaseConnection(connectionString);
    }
    return DatabaseConnection.#instance;
  }

  query(sql: string): void {
    console.log(`Executing on ${this.connectionString}: ${sql}`);
  }
}

const db1 = DatabaseConnection.getInstance("postgres://localhost/mydb");
const db2 = DatabaseConnection.getInstance();
console.log(db1 === db2); // → true

db1.query("SELECT * FROM users");

// new DatabaseConnection("..."); // ❌ Constructor of class 'DatabaseConnection' is private
```

### Edge Cases

- **Module-level singleton:** ES module — module bir marta evaluate qilinadi. `export const db = new DB()` ham singleton (alternativa)
- **Testability:** singleton mock qilish qiyin — DI yoki `resetInstance()` method qo'shish
- **Inheritance:** `private constructor` subclass'ni bloklaydi — `protected` qilish mumkin
- **Thread safety:** Node.js single-threaded — race condition yo'q. Worker threads alohida memory

### Follow-up savollar

1. "Multiple parameter bilan getInstance() — keyingi chaqiruvlarda nima bo'ladi?" — Birinchi argument bilan init, keyingi argument'lar ignore. Strict yondashuv: idempotency tekshiruvi
2. "Anti-pattern emasmi?" — Ko'p hollarda DI afzal. Singleton — connection pool, config, logger uchun mos

</details>

---

### Savol 17: Class implements interface — index signature bilan bug [Senior]

**Savol:** Quyidagi kod compile xato beradi. Sababini tushuntiring va tuzating:

```typescript
interface JsonSerializable {
  [key: string]: string | number | boolean;
}

class UserRecord implements JsonSerializable {
  name: string;
  age: number;
  isActive: boolean;
  createdAt: Date;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
    this.isActive = true;
    this.createdAt = new Date();
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`JsonSerializable` index signature `string | number | boolean` ga cheklaydi. `createdAt: Date` index signature'ga mos emas — compile error. Method va class'lar ham index signature talabidan istisno emas.

### To'liq tushuntirish

Interface'da index signature `[key: string]: T` bo'lsa — barcha property'lar `T` ga assignable bo'lishi shart. Class `implements` qilganda har bir property index signature'ga rioya qiladi.

`Date` `string | number | boolean` ga assignable emas. Yechim:
1. Index signature'ga `Date` qo'shish: `[key: string]: string | number | boolean | Date`
2. `createdAt` ni `string` (ISO) qilib saqlash
3. Index signature'ni olib tashlash, explicit property list ishlatish

### Kod misol

```typescript
// ✅ Variant 1: Index signature'ni kengaytirish
interface JsonSerializable {
  [key: string]: string | number | boolean | Date;
}

class UserRecord implements JsonSerializable {
  name: string;
  age: number;
  isActive: boolean;
  createdAt: Date;
  // Index signature qoladi: har property string|number|boolean|Date bo'lishi shart

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
    this.isActive = true;
    this.createdAt = new Date();
  }
}

// ✅ Variant 2: ISO string saqlash
interface JsonSerializable2 {
  [key: string]: string | number | boolean;
}

class UserRecord2 implements JsonSerializable2 {
  name: string;
  age: number;
  isActive: boolean;
  createdAt: string; // ISO

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
    this.isActive = true;
    this.createdAt = new Date().toISOString();
  }
}
```

### Edge Cases

- **Method va index signature:** method `(...args) => any` — `string | number | boolean` ga mos emas, xato
- **Optional property:** `prop?: T` — index signature'ga `T | undefined` bo'lib mos kelishi shart
- **Symbol key:** index signature `[key: string]` symbol property'larni qamramaydi
- **`Record<string, T>` alternativa:** `class X implements Record<string, T>` ham aynan shu cheklov

### Follow-up savollar

1. "Index signature'ni keng `unknown` qilsa-chi?" — Type safety yo'qoladi — har property accept qilinadi
2. "Class instance'ni `JSON.stringify` qilganda?" — Method'lar va `undefined` qiymatlar avtomatik ignore qilinadi, lekin compile-time muammosi qoladi

<details>
<summary><strong>Deep Dive</strong></summary>

Index signature ECMAScript spec'da mavjud emas — bu sof TypeScript type-system xususiyati. `class X implements I` tekshirilganda, agar `I` da `[key: string]: V` index signature bo'lsa, compiler `X` ning har bir e'lon qilingan property type'ini `V` ga assignable ekanligini tekshiradi (method type'lar ham, ular `V` ga mos kelmasa xato). Index signature key'i `string`, `number` yoki (TS 4.4+) `symbol`/template literal bo'lishi mumkin.

Runtime'da index signature hech qanday iz qoldirmaydi — emit qilingan JS'da har property oddiy own property bo'lib qoladi.

</details>

</details>

---

### Savol 18: `override` + `noImplicitOverride` — bug fix [Middle]

**Savol:** Bu kod `noImplicitOverride: true` bilan ishlamaydi. Toping va tuzating:

```typescript
class BaseLogger {
  log(message: string): void {
    console.log(`[INFO] ${message}`);
  }
  error(message: string): void {
    console.error(`[ERROR] ${message}`);
  }
}

class JsonLogger extends BaseLogger {
  log(message: string): void {
    console.log(JSON.stringify({ level: "info", message }));
  }
  errror(message: string): void { // typo
    console.error(JSON.stringify({ level: "error", message }));
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ikkita muammo: `log()` `override` keyword'siz (`noImplicitOverride: true` bilan xato), va `errror()` typo — `override` qo'shilsa bu typo ushlanadi.

### To'liq tushuntirish

`noImplicitOverride: true` parent method'ni qayta yozayotgan har bir method'ga `override` keyword'ni MAJBURIY qiladi. `errror` — typo, lekin `override` siz oddiy yangi method sifatida qabul qilinadi. `override` qo'yilsa — TS parent'da `errror` topa olmaydi va xato beradi.

### Kod misol

```typescript
class BaseLogger {
  log(message: string): void {
    console.log(`[INFO] ${message}`);
  }
  error(message: string): void {
    console.error(`[ERROR] ${message}`);
  }
}

class JsonLogger extends BaseLogger {
  override log(message: string): void { // ✅ override qo'shildi
    console.log(JSON.stringify({ level: "info", message }));
  }

  override error(message: string): void { // ✅ typo tuzatildi
    console.error(JSON.stringify({ level: "error", message }));
  }
}

const logger = new JsonLogger();
logger.log("user created");
// → {"level":"info","message":"user created"}
```

### Edge Cases

- **Yangi method qo'shish:** subclass'da parent'da bo'lmagan method `override` siz yoziladi
- **Refactoring safety:** parent'dan method olib tashlansa — barcha `override` xato beradi (intent: refactoring)
- **Static method override:** TS 4.3+ static method ham `override` qabul qiladi

### Follow-up savollar

1. "Eski codebase'da `noImplicitOverride: true` yoqsam — qancha xato bo'ladi?" — Har subclass method uchun bir xato. `--fixOverrides` flag yo'q, codemod kerak
2. "Abstract method implement qilishda `override` shartmi?" — Yo'q, lekin yozish ruxsat. TS dizayn qarori — abstract implement va override semantically farqli

</details>

---

### Savol 19: Parameter property + readonly — output [Middle]

**Savol:** Output ni ayting:

```typescript
class Account {
  constructor(
    public readonly id: number,
    public balance: number,
    private readonly _createdAt: Date = new Date(),
  ) {}

  deposit(amount: number): this {
    this.balance += amount;
    return this;
  }
}

const acc = new Account(1, 100);
acc.deposit(50).deposit(25);
console.log(acc.balance);
console.log((acc as any)._createdAt instanceof Date);
console.log("id" in acc);
console.log(Object.keys(acc).length);
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
175
true
true
3
```

### To'liq tushuntirish

1. `acc.deposit(50)` → `balance = 150`, `this` qaytaradi. `.deposit(25)` → `balance = 175`
2. `_createdAt` `private` — runtime'da oddiy property, `Date` instance
3. `"id" in acc` → `true` — parameter property avtomatik property yaratadi
4. `Object.keys` — barcha enumerable own property'lar: `id`, `balance`, `_createdAt`

Parameter property TypeScript shorthand — JS emit'da constructor body'ga `this.x = x` qo'shadi. TS `private` runtime'da yo'q — property oddiy enumerable.

### Edge Cases

- **`readonly` runtime'da:** runtime'da reassignment ishlaydi (TS strip qilingan)
- **Default value:** `_createdAt: Date = new Date()` — argument berilmagani uchun har instance uchun yangi `Date`
- **Method `Object.keys`'da yo'q:** method'lar prototype'da, own property emas

### Follow-up savollar

1. "Methodlar ham `Object.keys`'da ko'rinishi uchun?" — `useDefineForClassFields: false` (eski emit), yoki arrow function field
2. "`#private` qilsam `Object.keys`?" — `#` static private'lar `Object.keys`'da yo'q

</details>

---

## Xulosa

- Access modifiers (`public`/`private`/`protected`) — compile-time only, JS'da o'chiriladi
- ES `#` private — runtime hard privacy, lexical scope orqali brand check
- Parameter properties — constructor'da modifier'lar bilan avtomatik property
- `readonly` — compile-time mutation protection, shallow
- Abstract class — instantiate qilib bo'lmaydi, shared implementation + contract
- `implements` — tekshiruv, type injection emas. Inference bermaydi
- `override` — typo va refactoring xatolarini ushlaydi (`noImplicitOverride`)
- `this` polymorphic return type — fluent API / builder pattern uchun
- Class = 2 ta type: instance type (`Class`) va constructor type (`typeof Class`)
- Static initialization block (TS 4.4+, ES2022) — murakkab static init uchun
- `accessor` keyword (TS 4.9+) — auto-accessor, Stage 3 decorators uchun
- Generic class — type parameter constructor/method/property'da, static'da emas
