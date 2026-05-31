# Bo'lim 4: Object Types va Interfaces

> Object type'lar va interface'lar — TypeScript'da eng ko'p ishlatiladigan tip tuzilmalari. Interface declaration, extending, merging va type alias'dan farqi — TS type system'ining asosiy qurilish bloklari. `readonly`, optional property'lar, index signature'lar, `Record<K,V>`, excess property checking, recursive interface'lar va `PropertyKey` bu bobda.

---

## Mundarija

- [Object Type Annotations](#object-type-annotations)
- [Optional Properties](#optional-properties)
- [Readonly Properties](#readonly-properties)
- [Index Signatures](#index-signatures)
- [Record vs Index Signature](#record-vs-index-signature)
- [Interface Declaration](#interface-declaration)
- [Interface Extending](#interface-extending)
- [Multiple Interface Extending](#multiple-interface-extending)
- [Interface Merging — Declaration Merging](#interface-merging--declaration-merging)
- [Interface vs Type Alias](#interface-vs-type-alias)
- [Excess Property Checking](#excess-property-checking)
- [Recursive Interfaces](#recursive-interfaces)
- [PropertyKey Type](#propertykey-type)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Object Type Annotations

### Nazariya

Object type — TypeScript'da object'ning **shape**'ini (shaklini) belgilaydi. Qaysi property'lar bor, har birining tipi nima — shu ma'lumotni beradi. Bu TypeScript'ning structural typing'ning asosidir: object'ning **nomi** emas, **shakli** muhim.

```typescript
// Inline object type — to'g'ridan-to'g'ri yozish
let user: { name: string; age: number; email: string } = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
};

// Property separator — `;` yoki `,` ikkalasi ham ishlaydi
let point: { x: number, y: number } = { x: 10, y: 20 };
let point2: { x: number; y: number } = { x: 10, y: 20 };
```

**Convention:** interface/type alias'larda `;` (semicolon), inline type'larda esa `,` (comma) ko'proq ishlatiladi — lekin ikkalasi ham to'g'ri.

Structural typing tufayli funksiya parameter'iga undan **ko'proq** property'ga ega object ham uzatilishi mumkin — target shape qoplangani yetarli:

```typescript
function greet(person: { name: string }): string {
  return `Hello, ${person.name}`;
}

greet({ name: "Ali" });              // ✅
greet({ name: "Vali", age: 30 });    // ⚠️ Excess property — quyida batafsil
```

Inline object type tezda yozish uchun qulay, lekin **takrorlanadigan** shape'lar uchun **interface** yoki **type alias** tavsiya etiladi — qayta ishlatish va o'qish osonlashadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler object type'ni `TypeLiteral` AST node sifatida parse qiladi. Har property — `PropertySignature` node bo'lib, quyidagi field'larga ega:
- `name` (identifier)
- `type` (TypeNode)
- `questionToken` (optional bo'lsa)
- `modifiers` (readonly bo'lsa)

`checker.ts`'dagi `checkObjectLiteral()` funksiyasi object literal'ni type'ga tekshiradi:

1. **Target property'larni iteratsiya:** Har target type'dagi property uchun — source object'da shu nomli property bormi?
2. **Type assignability:** Bor bo'lsa — property type'lari mos keladimi? (`isTypeAssignableTo()`)
3. **Excess check:** Source'da target'da yo'q property bor bo'lsa — excess property check ishlaydi (faqat literal'da)

Structural compatibility tekshiruvi `isRelatedTo()` funksiyasida amalga oshadi. Bu funksiya har target property'ni source'da qidiradi va type assignability'ni rekursiv tekshiradi.

```
Object Type Checking Flow:

Source Object           Target Type
{ name: "Ali",    →    { name: string;
  age: 25,              age: number }
  email: "..." }
       │
       ▼
checker.checkObjectLiteral()
  ├── name: "Ali" → string ✅
  ├── age: 25 → number ✅
  └── email: "..." → TARGET'DA YO'Q
       └── Excess Property Check → ❌ (literal'da)
```

**Runtime:** object type compile vaqtida to'liq yo'qoladi. JavaScript'da oddiy object literal qoladi — hech qanday shape validation yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Inline object type
let config: { host: string; port: number; debug: boolean } = {
  host: "localhost",
  port: 3000,
  debug: true,
};

// Inline funksiya parameter
function createUser(data: { name: string; age: number }) {
  return { id: Math.random(), ...data };
}

createUser({ name: "Ali", age: 25 });

// Return type
function getPoint(): { x: number; y: number } {
  return { x: 10, y: 20 };
}

// Nested object
let order: {
  id: number;
  customer: { name: string; email: string };
  items: { sku: string; quantity: number }[];
} = {
  id: 1,
  customer: { name: "Ali", email: "ali@mail.com" },
  items: [
    { sku: "A-100", quantity: 2 },
    { sku: "B-200", quantity: 1 },
  ],
};

// Method yozish — ikki sintaksis
let calculator: {
  add(a: number, b: number): number;            // method shorthand
  subtract: (a: number, b: number) => number;   // property function
} = {
  add(a, b) { return a + b; },
  subtract: (a, b) => a - b,
};
```

Inline'dan type alias'ga o'tkazish (refactoring):

```typescript
// Oldingi — inline takrorlangan
function createUser(data: { name: string; age: number }) { /* ... */ }
function updateUser(id: number, data: { name: string; age: number }) { /* ... */ }

// Keyingi — interface bilan DRY
interface UserData {
  name: string;
  age: number;
}

function createUser(data: UserData) { /* ... */ }
function updateUser(id: number, data: UserData) { /* ... */ }
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
let user: { name: string; age: number } = { name: "Ali", age: 25 };

function getPoint(): { x: number; y: number } {
  return { x: 10, y: 20 };
}
```

```javascript
// Compiled JS — type annotation to'liq o'chirildi
let user = { name: "Ali", age: 25 };

function getPoint() {
  return { x: 10, y: 20 };
}
// Object type faqat compile-time — runtime'da iz yo'q
// Shape validation JS'da yo'q
```

</details>

---

## Optional Properties

### Nazariya

Optional property — `?` bilan belgilanadi. Bu property object'da **bo'lishi ham, bo'lmasligi ham** mumkin. Optional property'ning tipi avtomatik `T | undefined` bilan kengaytiriladi.

```typescript
interface User {
  name: string;        // majburiy
  age: number;         // majburiy
  email?: string;      // ixtiyoriy — string | undefined
  phone?: string;      // ixtiyoriy
}

// ✅ email va phone yo'q — OK
let user1: User = { name: "Ali", age: 25 };

// ✅ email bor, phone yo'q
let user2: User = { name: "Vali", age: 30, email: "vali@mail.com" };

// ✅ ikkalasi ham bor
let user3: User = {
  name: "Soli", age: 35,
  email: "s@mail.com", phone: "+998"
};
```

**Muhim farq:** `email?: string` va `email: string | undefined` — ikki xil narsa:
- **`email?: string`** — property **bo'lmasligi** mumkin (`{}` valid)
- **`email: string | undefined`** — property **bor**, lekin qiymati `undefined` bo'lishi mumkin (`{ email: undefined }` kerak, `{}` xato)

`exactOptionalPropertyTypes: true` flag bu farqni yanada kuchaytiradi — optional property'ga `undefined` assign qilish taqiqlanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Optional property checker'da `PropertySignature` AST node'da `questionToken` bo'lishi bilan belgilanadi. `Symbol.flags`'da `SymbolFlags.Optional` bit o'rnatiladi.

**`email?: string` semantikasi:**

```
Interface property uchun:
  - Symbol flags: Optional
  - Type: string (not string | undefined!)
  - Access: property bor yoki yo'q — "in" operator bilan tekshirish mumkin
```

Lekin **default xulq** (`exactOptionalPropertyTypes: false`) — checker optional property'ni `string | undefined`'ga kengaytiradi va `email: undefined`'ni qabul qiladi:

```typescript
interface User { email?: string; }

let withoutEmail: User = {};                  // ✅ property yo'q
let undefinedEmail: User = { email: undefined };  // ✅ default xulq — undefined qabul qilinadi
let stringEmail: User = { email: "test" };    // ✅ property bor, qiymati string
```

**`exactOptionalPropertyTypes: true` yoqilsa**, optional `T?` va `T | undefined` ajratiladi:

```typescript
let undefinedEmail: User = { email: undefined };  // ❌ Type 'undefined' is not assignable to type 'string'
// Faqat property'ni butunlay tushirib qoldirish mumkin: { }
```

Bu flag TS 4.4'da kiritilgan va `strict` meta-flag'ga kirmaydi (ataylab — backward compatibility). Production kodda yoqilishi tavsiya etiladi, lekin migration mehnatli.

**`in` operator bilan farqi:**

```typescript
interface OptionalName { name?: string; }
interface NullableName { name: string | undefined; }

const optional: OptionalName = {};              // name yo'q
const nullable: NullableName = { name: undefined }; // name bor, undefined

"name" in optional; // false — property yo'q
"name" in nullable; // true — property bor
```

Bu farq runtime'da `Object.keys()`, `JSON.stringify()`, spread operator va boshqa JS operatsiyalarida sezilarli.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Optional property bilan ishlash — narrowing shart:

```typescript
interface User {
  name: string;
  email?: string;
}

function sendEmail(user: User): void {
  // user.email type: string | undefined
  // user.email.toUpperCase(); // ❌ 'user.email' is possibly 'undefined'

  // ✅ Narrowing bilan
  if (user.email !== undefined) {
    console.log(user.email.toUpperCase());
  }

  // ✅ Optional chaining
  console.log(user.email?.toUpperCase()); // string | undefined

  // ✅ Nullish coalescing — default qiymat
  const email = user.email ?? "no-email@default.com";
}
```

Real-world: Form data — majburiy va optional field'lar:

```typescript
interface UserForm {
  username: string;        // majburiy
  password: string;        // majburiy
  firstName?: string;      // ixtiyoriy
  lastName?: string;       // ixtiyoriy
  age?: number;            // ixtiyoriy
  newsletter?: boolean;    // ixtiyoriy (default: false)
}

function register(form: UserForm) {
  // Majburiylar
  console.log(`Username: ${form.username}`);

  // Optional bilan default
  const firstName = form.firstName ?? "";
  const lastName = form.lastName ?? "";
  const fullName = `${firstName} ${lastName}`.trim() || "Anonymous";

  const age = form.age ?? 0;
  const newsletter = form.newsletter ?? false;

  return { username: form.username, fullName, age, newsletter };
}

register({ username: "ali", password: "pass123" });
// Optional'lar tushib qoldirildi — OK

register({
  username: "vali",
  password: "pass456",
  firstName: "Vali",
  newsletter: true,
});
```

Object property'larda optional va required tartibi cheklanmaydi — funksiya parameter'laridan farqli:

```typescript
// ✅ Object property'larda required optional'dan keyin kelishi mumkin
interface User {
  firstName?: string;
  age: number;        // required after optional — object'da ruxsat etiladi
  email?: string;
}

// ❌ Funksiya parameter'larida required optional'dan keyin kela olmaydi
function register(firstName?: string, age: number) {} // TS1016: A required parameter cannot follow an optional parameter

// ✅ To'g'ri tartib — optional parameter'lar oxirida
function registerOk(age: number, firstName?: string) {}
```

Object property'larda bu cheklov yo'q, chunki property'lar nom orqali ochiladi (`user.age`), parameter'lar esa pozitsiya orqali (`register(arg1, arg2)`) — pozitsion argument'da optional o'rtada turishi qaysi argument qaysi parameter'ga tegishli ekanini noaniq qiladi.

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface User {
  name: string;
  email?: string;
}

function greet(user: User): string {
  return `Hello, ${user.name} (${user.email ?? "no email"})`;
}

const user: User = { name: "Ali" };
```

```javascript
// Compiled JS — optional marker to'liq o'chirildi
function greet(user) {
  return `Hello, ${user.name} (${user.email ?? "no email"})`;
}

const user = { name: "Ali" };
// Runtime'da: user.email === undefined (property yo'q)
// Optional belgilash faqat compile-time check uchun
```

</details>

---

## Readonly Properties

### Nazariya

`readonly` modifier — property'ni faqat o'qishga cheklaydi. Object yaratilgandan keyin o'zgartirish mumkin emas (compile-time). Constructor yoki initialization paytida set qilinadi, keyin o'zgarmas.

```typescript
interface Point {
  readonly x: number;
  readonly y: number;
}

let point: Point = { x: 10, y: 20 };
console.log(point.x); // ✅ o'qish mumkin
// point.x = 30;     // ❌ Cannot assign to 'x' because it is a read-only property

// Butun object qayta assign — bu boshqa narsa
point = { x: 30, y: 40 }; // ✅ binding qayta assign (let)
// readonly faqat individual property'ni himoyalaydi
```

**`readonly` — shallow (yuzaki)**. Faqat birinchi daraja property'ni himoyalaydi, nested object'larni emas:

```typescript
interface Config {
  readonly settings: {
    theme: string;
    fontSize: number;
  };
}

const config: Config = {
  settings: { theme: "dark", fontSize: 14 },
};

// config.settings = { ... };        // ❌ settings qayta assign mumkin emas
config.settings.theme = "light";     // ⚠️ KUTILMAGAN: ichki property o'zgaradi
// readonly settings faqat settings reference'ni muzlatadi
// ichki property'lar mutable qoladi
```

Bu behavior ko'pincha **kutilmagandek** bo'ladi — "readonly" so'zi deep immutability'ni anglatadi deb o'ylaymiz. Lekin TypeScript'ning `readonly`, JavaScript'ning `Object.freeze()` singari, shallow himoya beradi — faqat birinchi daraja property reference'ni qotiradi.

**Deep readonly** kerak bo'lsa — `Readonly<T>` utility yetarli emas (u ham shallow). Custom `DeepReadonly<T>` type yozish kerak (`16-custom-utility-types.md` da).

<details>
<summary><strong>Under the Hood</strong></summary>

`readonly` modifier checker'da `PropertySignature.modifiers` array'ida `ReadonlyKeyword` token sifatida saqlanadi. `Symbol.flags`'da `SymbolFlags.Readonly` bit o'rnatiladi.

**Assignment checking:**

Assignment expression'ni (`expr = value`) tekshirayotganda checker target property'ga yozishga ruxsat borligini aniqlaydi:

1. Assignment node'ni aniqlash (`expr = value`)
2. Target property'ning Symbol'ini olish
3. `SymbolFlags.Readonly` bitini tekshirish
4. Agar readonly va bu initialization konteksti (`constructor`, object literal) **emas** bo'lsa — `Cannot assign to 'X' because it is a read-only property` diagnostikasi

**Shallow nature sababi:**

`config.settings.theme = "light"` ifodasi:
1. `config.settings` — **o'qish** operatsiyasi (read, assignment emas)
2. `.theme` — `settings` object'idan property olish
3. `= "light"` — `theme`'ga assign

Checker birinchi qadamni ko'rganida `settings` readonly ekanini aniqlaydi, lekin bu **read** — OK. Ikkinchi assignment (`theme`) checkida esa `settings` object'ining ichki `theme` property'si **readonly emas** (intersect qilinmagan), shuning uchun OK.

```
readonly Checking Flow:

config.settings = { ... }       config.settings.theme = "light"
      │                                │
      ▼                                ▼
checker: write to `settings`     checker: read `config.settings` (OK)
  → readonly? YES                        .theme = "light" (write)
  → error ❌                       checker: write to `theme`
                                         → theme readonly? NO
                                         → OK ✅ (shallow readonly)
```

**Constructor context:**

Class'da `readonly` property'lar constructor ichida **birinchi marta** assign qilinishi mumkin:

```typescript
class User {
  readonly id: number;
  constructor(id: number) {
    this.id = id; // ✅ constructor'da birinchi init OK
  }
  updateId(newId: number) {
    this.id = newId; // ❌ constructor'dan tashqarida readonly
  }
}
```

Checker assignment'ning enclosing context (constructor body yoki property initializer) ichidaligini aniqlaydi va shu yerdagi birinchi initialization'ga ruxsat beradi.

**Runtime:** `readonly` JS'da butunlay yo'qoladi. Compile'dan keyin property freely mutable. Haqiqiy runtime immutability uchun `Object.freeze()` yoki immutable library kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Interface'da readonly
interface User {
  readonly id: number;
  readonly createdAt: Date;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  createdAt: new Date(),
  name: "Ali",
  email: "ali@mail.com",
};

// user.id = 2;          // ❌ readonly
// user.createdAt = ...; // ❌ readonly
user.name = "Vali";      // ✅ mutable
user.email = "v@m.com";  // ✅ mutable
```

Class'da readonly:

```typescript
class Account {
  readonly id: string;
  readonly createdAt: Date;
  balance: number;

  constructor(id: string, initialBalance: number) {
    this.id = id;                 // ✅ constructor'da init
    this.createdAt = new Date();  // ✅ constructor'da init
    this.balance = initialBalance;
  }

  deposit(amount: number) {
    this.balance += amount;        // ✅ balance readonly emas
    // this.id = "new-id";         // ❌ readonly, constructor'dan tashqarida
  }
}
```

Shallow nature gotcha:

```typescript
interface AppConfig {
  readonly database: {
    host: string;
    port: number;
  };
  readonly features: string[];
}

const config: AppConfig = {
  database: { host: "localhost", port: 5432 },
  features: ["auth", "billing"],
};

// config.database = { ... };        // ❌ readonly
config.database.host = "remote";     // ⚠️ Ichki property mutable!
config.features.push("analytics");   // ⚠️ Array mutable! (readonly object bo'lsa ham)

// Haqiqiy runtime immutability
const frozen = Object.freeze({
  ...config,
  database: Object.freeze({ ...config.database }),
  features: Object.freeze([...config.features]),
});

// frozen.database.host = "x"; // Runtime error (strict mode)
```

`Readonly<T>` utility va cheklovi:

```typescript
interface MutableConfig {
  host: string;
  settings: { theme: string };
}

// Readonly<T> — har property'ni readonly qiladi (shallow)
type FrozenConfig = Readonly<MutableConfig>;
// {
//   readonly host: string;
//   readonly settings: { theme: string }; // settings readonly, lekin theme MUTABLE
// }

const frozen: FrozenConfig = { host: "localhost", settings: { theme: "dark" } };
// frozen.host = "x";        // ❌
// frozen.settings = { ... }; // ❌
frozen.settings.theme = "light"; // ⚠️ SHALLOW — OK!

// Deep readonly — custom type
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
// Batafsil: 16-custom-utility-types.md
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface Point {
  readonly x: number;
  readonly y: number;
}

const point: Point = { x: 10, y: 20 };

class User {
  readonly id: number;
  constructor(id: number) {
    this.id = id;
  }
}
```

```javascript
// Compiled JS — readonly marker to'liq o'chiriladi
const point = { x: 10, y: 20 };

class User {
  constructor(id) {
    this.id = id;
  }
}

// Runtime'da himoya YO'Q:
// point.x = 30;   // ✅ JS'da ishlaydi
// user.id = "x";  // ✅ JS'da ishlaydi
// readonly faqat compile-time — TS tekshiruvi
```

Runtime immutability `Object.freeze()` bilan:

```javascript
const point = Object.freeze({ x: 10, y: 20 });
point.x = 30; // TypeError (strict mode) yoki sukut (sloppy mode)
```

</details>

---

## Index Signatures

### Nazariya

Index signature — object'da **dinamik** (oldindan noma'lum) kalitlar uchun tip belgilash. Qachon property nomlari oldindan ma'lum emas — index signature ishlatiladi.

```typescript
// Index signature: [key: KeyType]: ValueType
interface StringMap {
  [key: string]: number;
}

let prices: StringMap = {
  apple: 100,
  banana: 200,
  cherry: 150,
};

prices["grape"] = 300;       // ✅ har qanday string key, number value
prices.melon = 250;           // ✅ dot notation ham ishlaydi
// prices["pear"] = "expensive"; // ❌ string value emas, number kerak
```

Index signature'ning kalit tipi `string`, `number`, `symbol` (bular JavaScript'ning valid property key tiplari) yoki **template literal type** (TS 4.4+) bo'lishi mumkin:

```typescript
// String key
interface Dictionary { [key: string]: string; }

// Number key — array-like object
interface NumberMap { [index: number]: string; }

// Symbol key (TS 4.4+)
interface SymbolMap { [key: symbol]: string; }

// Template literal pattern (TS 4.4+)
interface DataAttrs { [key: `data-${string}`]: string }
```

**Muhim:** Index signature va aniq (explicit) property'lar **birga** ishlatilganda, aniq property'ning tipi index signature'ga **mos kelishi** kerak. Sabab: runtime'da har qanday property access index signature orqali o'tadi.

```typescript
interface UserMap {
  [key: string]: string | number;
  name: string;    // ✅ string ⊂ (string | number)
  age: number;     // ✅ number ⊂ (string | number)
  // active: boolean; // ❌ boolean ⊄ (string | number)
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Index signature checker'da `IndexSignatureDeclaration` AST node sifatida parse qilinadi. Interface/type'ga `SignatureKind.Index` bilan biriktiriladi.

**Qoida: explicit property → index signature subtype:**

```typescript
interface UserMap {
  [key: string]: string | number;
  name: string;
  age: number;
}
```

Bu yerda ichki mantiq:
1. `obj.name` runtime'da `obj["name"]` — ya'ni index access
2. Index signature: `[string]: string | number`
3. `obj["name"]` tipi → `string | number`
4. Agar `name` explicit type `string` bo'lsa, `string`'ni `string | number`'ga assign qilish OK
5. Agar `name` explicit `boolean` bo'lsa, `boolean`'ni `string | number`'ga assign qilish — xato

Sabab: **runtime consistency**. Agar explicit property index signature'dan kengroq tipga ega bo'lsa, index access orqali olish va explicit property orqali olish farqli tiplar qaytarardi — bu silent type unsoundness.

**Number index va string index bilan birga:**

```typescript
interface MixedIndex {
  [index: number]: string;
  [key: string]: string | number;
  // QOIDA: number index tipi string index tipining SUBTYPE'i bo'lishi shart
  // string ⊂ (string | number) ✅
}
```

Sabab: JavaScript'da `obj[0]` aslida `obj["0"]` — number key runtime'da string'ga coerce bo'ladi. Shuning uchun number index tipi string index tipidan keng bo'la olmaydi (aksincha bo'lsa, `obj["0"]` va `obj[0]` farqli tiplar qaytarardi).

**`noUncheckedIndexedAccess`:**

Bu flag index access natijasini `T | undefined`'ga kengaytiradi — xavfsizroq:

```typescript
// noUncheckedIndexedAccess: false (default)
interface SafeMap { [key: string]: string; }
const map: SafeMap = { a: "hello" };
const val = map["nonexistent"]; // type: string (runtime'da undefined — silent!)

// noUncheckedIndexedAccess: true
const val2 = map["nonexistent"]; // type: string | undefined
if (val2 !== undefined) {
  console.log(val2.toUpperCase()); // ✅ narrowed
}
```

Production kodda bu flag juda tavsiya etiladi — index access xavfsizligi sezilarli oshadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Simple dictionary
interface Prices {
  [item: string]: number;
}

const prices: Prices = {
  apple: 100,
  banana: 200,
};

prices["cherry"] = 300;
prices.grape = 400;

// prices.orange = "expensive"; // ❌ number kerak
```

Explicit property + index signature:

```typescript
interface ApiResponse {
  [key: string]: unknown;  // dinamik metadata
  status: number;           // explicit — unknown'ga mos (har tip unknown'ga assign bo'ladi)
  message: string;          // explicit
}

const response: ApiResponse = {
  status: 200,
  message: "OK",
  // Dinamik field'lar — server'dan har xil kelishi mumkin
  requestId: "abc-123",
  timestamp: Date.now(),
  debug: { traceId: "xyz" },
};

response.status;      // number
response.message;     // string
response.requestId;   // unknown — narrowing kerak
```

Number index — array-like:

```typescript
interface NumberMap {
  [index: number]: string;
}

const items: NumberMap = {
  0: "first",
  1: "second",
  2: "third",
};

items[0]; // "first"
items[1]; // "second"
// items["label"]; // ❌ number index — string key emas
```

`noUncheckedIndexedAccess` xavfsizlik:

```typescript
// tsconfig.json: "noUncheckedIndexedAccess": true

interface CacheMap {
  [key: string]: string;
}

const cache: CacheMap = { hello: "world" };

const value = cache["missing"];
// type: string | undefined (noUncheckedIndexedAccess bilan)

// value.toUpperCase(); // ❌ 'value' is possibly 'undefined'

if (value !== undefined) {
  value.toUpperCase(); // ✅ narrowed
}

// Yoki default bilan
const safe = cache["missing"] ?? "default";
```

Real-world: HTTP headers:

```typescript
interface Headers {
  [header: string]: string | string[];
}

function parseHeaders(headers: Headers): void {
  const contentType = headers["content-type"];
  // string | string[]

  if (typeof contentType === "string") {
    console.log(contentType.split(";")[0]);
  }
}

parseHeaders({
  "content-type": "application/json",
  "set-cookie": ["session=abc", "theme=dark"],
});
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface PriceMap {
  [item: string]: number;
}

const prices: PriceMap = {
  apple: 100,
  banana: 200,
};

prices["cherry"] = 300;
```

```javascript
// Compiled JS — index signature to'liq o'chiriladi
const prices = {
  apple: 100,
  banana: 200,
};

prices["cherry"] = 300;

// Runtime'da hech qanday key type tekshiruv yo'q
// JS'da har qanday string/number/symbol key ishlatish mumkin
```

</details>

---

## Record vs Index Signature

### Nazariya

`Record<K, V>` utility type va index signature o'xshash vazifa bajaradi — lekin muhim farqlari bor. Qaysi birini ishlatish kontekstga bog'liq.

```typescript
// Index signature
interface StringNumberMap {
  [key: string]: number;
}

// Record — soddaroq sintaksis
type StringNumberRecord = Record<string, number>;

// Ikkalasi ham bir xil — ixtiyoriy string key, number value
```

**Asosiy farq:** `Record` **cheklangan key**'lar bilan ishlay oladi — union literal type bilan:

```typescript
// Cheklangan key'lar — Record afzalroq
type Fruit = "apple" | "banana" | "cherry";
type FruitPrices = Record<Fruit, number>;
// { apple: number; banana: number; cherry: number }
// BARCHA key'lar majburiy

const prices: FruitPrices = {
  apple: 100,
  banana: 200,
  cherry: 150,
  // orange: 50, // ❌ Record da yo'q
};

// Index signature bilan buni qilish mumkin emas
interface FruitMap {
  [key: string]: number;
  // Bu HAR QANDAY string'ni qabul qiladi, cheklangan emas
}
```

| Xususiyat | Index Signature | `Record<K, V>` |
|-----------|----------------|----------------|
| Sintaksis | `[key: string]: V` | `Record<K, V>` |
| Cheklangan key'lar | ❌ Mumkin emas | ✅ Union literal bilan |
| Explicit property bilan birga | ✅ | ⚠️ Intersection kerak |
| Method qo'shish | ✅ Interface ichida | ⚠️ Intersection kerak |
| Optional key'lar | ❌ | ✅ `Partial<Record<K, V>>` |
| Readonly | ✅ `readonly [key]` | ✅ `Readonly<Record<K, V>>` |

**Method qo'shish — gotcha:** `Record<K, V>` oddiy mapped type — qo'shimcha property/method yozib bo'lmaydi. Bunga harakat qilish odatda muvaffaqiyatsiz:

```typescript
// ❌ Record body'siga qo'shimcha property yozib bo'lmaydi — bu mapped type, interface emas
type ScoreMap = Record<string, number>; // faqat key-value, body yo'q

// ⚠️ Intersection bilan qo'shsa — `size` ikki cheklovga tushadi:
type ScoreMapWithSize = Record<string, number> & { size(): number };
// Record<string, number> uchun `size` — string key, demak `number` bo'lishi kerak
// { size(): number } uchun `size` — `() => number`
// Natija: size tipi `number & (() => number)` — bu tipga mos qiymat YO'Q
const scores: ScoreMapWithSize = {
  math: 90,
  size: () => 1, // ❌ Type '() => number' is not assignable to type 'number & (() => number)'
};
// Tip o'zi `never` emas, lekin `size` property'ni qoniqtirib bo'lmaydi

// ✅ Interface — index signature value tipini method'ni ham qamrab oladigan qilish kerak
interface ScoreRegistry {
  [key: string]: number | (() => number); // value tipi method'ni ham qamrab oladi
  size(): number; // ✅ () => number index signature value'siga mos
  math: number;   // ✅ number index signature value'siga mos
}
// Interface'da har bir named property index signature value tipiga assignable bo'lishi shart
```

<details>
<summary><strong>Under the Hood</strong></summary>

`Record<K, V>` — TypeScript'ning built-in mapped type:

```typescript
// lib.es5.d.ts'da:
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

Bu mapped type'da:
- `K extends keyof any` — `K` `PropertyKey` (`string | number | symbol`) bo'lishi kerak
- `[P in K]: T` — union'dagi har a'zo uchun property

**Evaluation:**

```typescript
type Fruit = "apple" | "banana";
type Prices = Record<Fruit, number>;

// Compiler evaluation:
// Record<"apple" | "banana", number>
// → { [P in "apple" | "banana"]: number }
// → { apple: number; banana: number }
```

Checker mapped type'ni **literal object type**'ga "expand" qiladi — har union a'zosi uchun alohida property yaratadi. Bu "distribution over union" xususiyati.

**Index signature farqi:**

```typescript
interface FruitMap {
  [key: string]: number;
}
```

Bu `IndexSignatureDeclaration` node — compiler uni `TypeFlags.Object | ObjectFlags.Interface` bilan track qiladi va type'ning string index info'si sifatida saqlaydi. Index access tekshiruvida har qanday string key ga `number` tipi tayinlanadi.

Farq: `Record` **aniq property'lar** ro'yxati, index signature esa **umumiy qoida**. Shu sababli:
- `Record<"a" | "b", number>` — object'da `a` va `b` majburiy, boshqalar yo'q
- `{ [key: string]: number }` — istalgan key mumkin, majburiy emas

**Mapped type'larning boshqa variantlari:**

```typescript
// Partial — hammasi optional
type PartialRecord<K extends keyof any, V> = { [P in K]?: V };

// Readonly
type ReadonlyRecord<K extends keyof any, V> = { readonly [P in K]: V };

// Kombinatsiya
type ImmutablePartial<K extends keyof any, V> = {
  readonly [P in K]?: V;
};
```

`Record` — eng oddiy mapped type variant. TypeScript'da mapped type'lar juda kuchli vosita — `13-mapped-types.md`'da batafsil.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Record cheklangan key'lar bilan:

```typescript
// Theme colors
type Theme = "light" | "dark";
type ColorScheme = Record<Theme, { bg: string; text: string }>;

const colors: ColorScheme = {
  light: { bg: "#fff", text: "#000" },
  dark: { bg: "#000", text: "#fff" },
  // agar biri tushirib qoldirilsa:
  // ❌ Property 'light' is missing in type
};

// Status handlers
type Status = "idle" | "loading" | "success" | "error";
type StatusHandlers = Record<Status, () => void>;

const handlers: StatusHandlers = {
  idle: () => console.log("idle"),
  loading: () => showSpinner(),
  success: () => hideSpinner(),
  error: () => showError(),
};

handlers.loading(); // ✅ type-safe
```

Index signature — dinamik kalitlar:

```typescript
// User-generated keys (masalan, form data)
interface FormData {
  [fieldName: string]: string | number | boolean;
}

const form: FormData = {
  username: "ali",
  age: 25,
  newsletter: true,
  // Har qanday field dinamik
  "referral-code": "SUMMER25",
  phone_number: "123-456",
};
```

Record va index signature'ni birga ishlatish:

```typescript
// Fixed keys + dynamic metadata
type BaseConfig = Record<"host" | "port", string | number>;
type Config = BaseConfig & {
  [key: string]: unknown;  // metadata
};

const cfg: Config = {
  host: "localhost",
  port: 3000,
  // Dinamik metadata
  ssl: true,
  timeout: 5000,
  customFlag: "enabled",
};
```

Optional Record (partial):

```typescript
type Language = "en" | "uz" | "ru";
type Translations = Partial<Record<Language, string>>;
// { en?: string; uz?: string; ru?: string }

const greetings: Translations = {
  en: "Hello",
  uz: "Salom",
  // ru — optional, tushirilishi mumkin
};
```

Record'dan Object.keys() bilan iterate:

```typescript
const scores: Record<string, number> = {
  math: 90,
  science: 85,
  history: 95,
};

Object.keys(scores).forEach(subject => {
  console.log(`${subject}: ${scores[subject]}`);
});

// Literal Record — iterate aniq tiplar bilan
type Subject = "math" | "science";
const typedScores: Record<Subject, number> = {
  math: 90,
  science: 85,
};

// Object.keys tipi — string[] (ichidagi key'lar aniq bilinsa ham)
// Aniqroq iterate:
(Object.keys(typedScores) as Subject[]).forEach(subject => {
  console.log(subject, typedScores[subject]);
});
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
type Theme = "light" | "dark";
type ColorScheme = Record<Theme, string>;

const colors: ColorScheme = {
  light: "#fff",
  dark: "#000",
};

interface DynamicMap {
  [key: string]: number;
}
const scores: DynamicMap = { math: 90, science: 85 };
```

```javascript
// Compiled JS — barcha type'lar o'chiriladi
const colors = {
  light: "#fff",
  dark: "#000",
};

const scores = { math: 90, science: 85 };

// Record va index signature — faqat compile-time
// Runtime'da oddiy object literal, hech qanday validation yo'q
```

</details>

---

## Interface Declaration

### Nazariya

`interface` — object'ning shape'ini e'lon qilish uchun TypeScript'ning asosiy mexanizmi. Interface faqat **tip metadata** beradi — implementation yo'q. U class implementation'iga o'xshamaydi — faqat "shape contract".

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;                // optional
  readonly createdAt: Date;    // readonly
}

function createUser(data: Omit<User, "id" | "createdAt">): User {
  return {
    id: Math.random(),
    createdAt: new Date(),
    ...data,
  };
}

const user = createUser({ name: "Ali", email: "ali@mail.com" });
```

Interface'da method'lar — **ikki xil sintaksis** bilan yoziladi:

```typescript
interface Calculator {
  // 1. Method shorthand
  add(a: number, b: number): number;

  // 2. Property function (arrow-like)
  subtract: (a: number, b: number) => number;
}
```

**Muhim farq:** Method shorthand **bivariant** parameter checking qoladi (`strictFunctionTypes: true`'da ham), property function esa **contravariant** bo'ladi. Bu backward compatibility uchun — `Array.prototype.sort` kabi method'lar bivariance'siz ishlamas edi. Batafsil `25-type-compatibility.md` da.

Umuman olganda **method shorthand** tavsiya etiladi — qisqa va idiomatic.

<details>
<summary><strong>Under the Hood</strong></summary>

`interface` declaration checker'da `InterfaceDeclaration` AST node va `InterfaceType` object sifatida track qilinadi.

**Compiler ichki saqlash:**

Interface'ning har bir property'si `Symbol` bo'lib, `SymbolTable` (hash map) ichida saqlanadi:

```
Interface User:
┌────────────────────────────────┐
│ InterfaceType                   │
│ ┌─────────────────────────────┐ │
│ │ SymbolTable (Map)           │ │
│ │ "id"    → Symbol(number)    │ │
│ │ "name"  → Symbol(string)    │ │
│ │ "email" → Symbol(string)    │ │
│ │ "age"   → Symbol(optional)  │ │
│ └─────────────────────────────┘ │
└────────────────────────────────┘
```

Property lookup `O(1)` hash map orqali — tez. Interface type relations `isRelatedTo()` funksiyasida structural compatibility bilan tekshiriladi.

**Binder bosqichi:** `binder.ts` interface declaration'ni ko'rganida `Symbol` yaratadi va uni scope'ga qo'shadi. Symbol `SymbolFlags.Interface | SymbolFlags.Type` flag'larga ega bo'ladi.

**Checker bosqichi:** Property access (`user.name`) ifodasini ko'rganida, checker interface Symbol'dan property'ni topadi va uning tipini qaytaradi. Bu lookup **cached** — bir marta hisoblangach, qayta ishlatiladi.

**Method shorthand vs property function:**

```typescript
interface Animal { name: string; }
interface Dog extends Animal { bark(): void; }

// Variance'ni ko'rsatish uchun interface'lar KENGROQ tip (Animal) kutadi
interface MethodStyle {
  handle(animal: Animal): void;       // method shorthand
}

interface PropertyStyle {
  handle: (animal: Animal) => void;   // property function
}
```

`strictFunctionTypes: true` bo'lganda:
- **Method shorthand (`MethodStyle.handle`):** parameter contravariance tekshirilmaydi — **bivariant** (eski xulq)
- **Property function (`PropertyStyle.handle`):** parameter contravariance tekshiriladi — **contravariant** (zamonaviy strict)

```typescript
type DogHandler = (animal: Dog) => void;

const dogHandler: DogHandler = (dog) => dog.bark();
// dogHandler faqat Dog'ni qabul qiladi — kengroq Animal uchun noxavfsiz

let withMethod: MethodStyle = { handle: dogHandler };
// ✅ strictFunctionTypes bilan ham OK — method shorthand BIVARIANT
// (texnik jihatdan unsound, lekin backward-compat uchun saqlangan)

let withProperty: PropertyStyle = { handle: dogHandler };
// ❌ Type '(animal: Dog) => void' is not assignable to type '(animal: Animal) => void'
// Property function CONTRAVARIANT: Animal param kutilgan, Dog param berilgan
```

Nima uchun method shorthand bivariance saqlandi: `Array<Dog>.sort((a, b) => ...)` kabi method'lar `Array<Animal>`'ga assign bo'lishi uchun. Bu JavaScript'ning inheritance pattern'i uchun muhim, lekin texnik jihatdan unsound.

Batafsil variance haqida: `25-type-compatibility.md`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Simple interface:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  name: "Ali",
  email: "ali@mail.com",
};
```

Interface'da method'lar:

```typescript
interface Logger {
  log(message: string): void;
  error(message: string, code?: number): void;
  warn(message: string): void;
}

// Implementation — object
const consoleLogger: Logger = {
  log(msg) { console.log(msg); },
  error(msg, code) { console.error(`[${code ?? "?"}] ${msg}`); },
  warn(msg) { console.warn(msg); },
};

// Implementation — class
class FileLogger implements Logger {
  log(message: string): void { /* write to file */ }
  error(message: string, code?: number): void { /* ... */ }
  warn(message: string): void { /* ... */ }
}
```

Interface bilan funksiya signature:

```typescript
// Funksiya tipi — callable interface
interface Comparator {
  (a: number, b: number): number;
}

const ascending: Comparator = (a, b) => a - b;
const descending: Comparator = (a, b) => b - a;

[3, 1, 2].sort(ascending);  // [1, 2, 3]
[3, 1, 2].sort(descending); // [3, 2, 1]

// Mixed — call signature + property
interface HttpClient {
  (url: string): Promise<Response>;
  baseURL: string;
  timeout: number;
}

const client = ((url: string) => fetch(url)) as HttpClient;
client.baseURL = "https://api.example.com";
client.timeout = 5000;
client("/users");
```

Interface bilan generics:

```typescript
interface Repository<T> {
  find(id: number): T | null;
  findAll(): T[];
  save(item: T): void;
  delete(id: number): boolean;
}

interface User { id: number; name: string; }

class UserRepo implements Repository<User> {
  private users: User[] = [];

  find(id: number) { return this.users.find(u => u.id === id) ?? null; }
  findAll() { return [...this.users]; }
  save(user: User) { this.users.push(user); }
  delete(id: number) {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) return false;
    this.users.splice(index, 1);
    return true;
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface User {
  id: number;
  name: string;
}

interface Logger {
  log(msg: string): void;
}

const user: User = { id: 1, name: "Ali" };

const logger: Logger = {
  log(msg) { console.log(msg); }
};
```

```javascript
// Compiled JS — interface TO'LIQ o'chirildi (type erasure)
const user = { id: 1, name: "Ali" };

const logger = {
  log(msg) { console.log(msg); }
};

// Interface JS'da hech qanday iz qoldirmaydi
// Type checking faqat compile-time
```

</details>

---

## Interface Extending

### Nazariya

`extends` keyword bilan bir interface boshqa interface'dan **meros** oladi. Child interface parent'ning barcha property'larini avtomatik oladi va yangilarini qo'shadi.

```typescript
// Base interface
interface Person {
  name: string;
  age: number;
}

// Extended — Person'ning barcha property'lari + yangi property'lar
interface Employee extends Person {
  company: string;
  position: string;
  salary: number;
}

const emp: Employee = {
  name: "Ali",
  age: 25,
  company: "TechCorp",
  position: "Developer",
  salary: 5000,
};
```

**Property override** — child'da parent bilan bir xil nomli property bo'lsa, tip **compatible** bo'lishi kerak (subtype'ga toraytirish mumkin, supertype'ga kengaytirish mumkin emas):

```typescript
interface Animal {
  name: string;
  sound: string;
}

interface Dog extends Animal {
  breed: string;
  sound: "bark"; // ✅ "bark" ⊂ string — subtype, mumkin
  // sound: number; // ❌ number ⊄ string — mumkin emas
}
```

Bu **covariance** qoidasi — child interface parent'ning "tor" versiyasi bo'lishi mumkin, lekin "keng" versiyasi emas. Liskov substitution principle.

<details>
<summary><strong>Under the Hood</strong></summary>

`extends` clause checker'da `HeritageClause` AST node sifatida saqlanadi. Interface declaration'da `heritageClauses` array'i bor — har bir `extends T` uchun bitta clause.

**Property merge process:**

Binder va Checker bosqichlarida compiler parent va child property'larni birlashtiradi:

1. **Parent property'larni olish** — `resolveBaseTypesOfInterface()` parent'ning to'liq property ro'yxatini yig'adi
2. **Child property'larni qo'shish** — child'ning shaxsiy property'lari parent property'lariga qo'shiladi
3. **Override check** — agar child'da parent bilan bir xil nomli property bor bo'lsa, `isTypeAssignableTo(child.type, parent.type)` tekshiradi
4. **Yangi SymbolTable** — to'liq merged property'lar bilan child interface'ning Symbol table'i quriladi

```
Interface Employee extends Person:

Person props:           Employee props:
┌──────────┐           ┌──────────────┐
│ name     │───────────┤ name         │
│ age      │───────────┤ age          │
└──────────┘           │ company      │ ← yangi
                       │ position     │ ← yangi
                       │ salary       │ ← yangi
                       └──────────────┘

Checker:
  resolveBaseTypesOfInterface(Employee)
  → base: [Person]
  → Person properties copied to Employee
  → Employee own properties added
  → Final: { name, age, company, position, salary }
```

**Override qoidasi — Liskov substitution:**

Agar child'da `sound: "bark"` va parent'da `sound: string`, checker quyidagi assignability'ni tekshiradi:

```
isTypeAssignableTo(childType: "bark", parentType: string)
→ "bark" ⊂ string → OK
```

Ya'ni child tipi parent tipiga assign bo'lishi kerak — bu "narrower than" qoidasi. Teskarisi (`sound: string` → `sound: "bark"`) ishlamaydi, chunki `string`'ni `"bark"`'ga assign qilib bo'lmaydi.

**Runtime'da** `extends` butunlay yo'qoladi. Child interface runtime'da alohida entity emas — faqat compile-time'da parent + child property'larning to'plami.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Simple extending:

```typescript
interface Base {
  id: number;
  createdAt: Date;
}

interface Post extends Base {
  title: string;
  content: string;
}

interface Comment extends Base {
  text: string;
  postId: number;
}

const post: Post = {
  id: 1,
  createdAt: new Date(),
  title: "Hello",
  content: "World",
};
```

Chain extending:

```typescript
interface Entity {
  id: string;
}

interface Timestamped extends Entity {
  createdAt: number;
}

interface Article extends Timestamped {
  published: boolean;
}

const article: Article = {
  id: "a-1",
  createdAt: 1700000000,
  published: true,
};
// Article'da: id, createdAt, published — hammasi bor
```

Property narrowing (subtype override):

```typescript
interface Response {
  status: number;
  data: string | number;
}

interface SuccessResponse extends Response {
  status: 200; // literal subtype (number)
  data: string; // string ⊂ (string | number)
}

interface ErrorResponse extends Response {
  status: 400 | 404 | 500; // literal union
  data: number; // number ⊂ (string | number)
}
```

Interface extending type alias:

```typescript
type BaseType = {
  id: number;
  name: string;
};

// Interface type alias'dan extend qilishi mumkin — intersection bilan ekvivalent
interface ExtendedUser extends BaseType {
  email: string;
}

const extendedUser: ExtendedUser = {
  id: 1,
  name: "Ali",
  email: "ali@mail.com",
};
```

Method override:

```typescript
interface Animal {
  name: string;
  makeSound(): string;
}

interface Dog extends Animal {
  // method override — signature mos kelishi shart
  makeSound(): string; // ✅ bir xil signature
}

// Return type override — covariant (subtype)
interface Cat extends Animal {
  makeSound(): "meow"; // ✅ "meow" ⊂ string
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface Person {
  name: string;
  age: number;
}

interface Employee extends Person {
  company: string;
}

const emp: Employee = {
  name: "Ali",
  age: 25,
  company: "TechCorp",
};
```

```javascript
// Compiled JS — barcha interface va extends o'chirildi
const emp = {
  name: "Ali",
  age: 25,
  company: "TechCorp",
};

// Interface hierarchy runtime'da mavjud emas
// Compile vaqtida parent + child property'lar to'plami sifatida tekshirilgan
```

</details>

---

## Multiple Interface Extending

### Nazariya

TypeScript'da interface **bir nechta** interface'dan meros olishi mumkin — `extends A, B, C`. Bu "multiple inheritance" JavaScript class'larida yo'q, lekin TypeScript interface'larida **to'liq qo'llab-quvvatlanadi** (chunki interface faqat shape, implementation emas).

```typescript
interface HasName {
  name: string;
}

interface HasAge {
  age: number;
}

interface HasEmail {
  email: string;
}

// Ko'p merosxo'rlik — barcha interface'larning property'lari birlashadi
interface User extends HasName, HasAge, HasEmail {
  id: number;
}

const user: User = {
  id: 1,
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
};
```

**Conflict holati:** ikki parent bir xil nomli property'ga ega bo'lsa, tiplar **identik** bo'lishi kerak (subtype yetarli emas):

```typescript
interface HasName { value: string; }
interface HasLabel { value: string; } // ✅ bir xil tip — OK

interface Field extends HasName, HasLabel {
  // value: string — conflictsiz birlashadi
}

// ❌ Tiplar farqli
interface StringField { value: string; }
interface NumberField { value: number; }

interface MixedField extends StringField, NumberField {
  // ❌ Interface 'MixedField' cannot simultaneously extend types 'StringField' and 'NumberField'
  // Named property 'value' of types 'StringField' and 'NumberField' are not identical
}
```

Multiple extending ayniqsa **mixin pattern**'lar uchun foydali — bir nechta "xususiyat" (capability) interface'larini bitta entity'ga jamlash.

<details>
<summary><strong>Under the Hood</strong></summary>

Multiple extending checker'da `HeritageClause.types` array orqali ifodalanadi. Har `extends T` clause uchun bitta type reference, ammo bitta heritage clause ichida bir nechta reference bo'lishi mumkin: `extends A, B, C`.

**Property merging:**

Compiler har parent'dan property'larni **chiziqli tartibda** yig'adi:

```
interface User extends HasName, HasAge, HasEmail:

1. HasName property'larini olish:  { name: string }
2. HasAge property'larini qo'shish: { name: string, age: number }
3. HasEmail property'larini qo'shish: { name, age, email: string }
4. User'ning shaxsiy property'lari: { name, age, email, id: number }
```

`binder.ts`'da `resolveBaseTypesOfInterface()` bu jarayon'ni amalga oshiradi. Har parent uchun ularning to'liq property ro'yxati (ularning o'zi ham extend qilgan bo'lishi mumkin) rekursiv olinadi.

**Conflict check:**

Agar ikki parent bir xil nomli property'ga ega bo'lsa, checker bu property'larning tiplarini solishtiradi va ular **identik** emas bo'lsa diagnostika beradi: `Interface 'X' cannot simultaneously extend types 'A' and 'B'. Named property 'name' of types 'A' and 'B' are not identical.`

Muhim nuans: bu tekshiruv **type identity** (aynan bir xil tip) talab qiladi — **assignability** (subtype) emas. Ya'ni parent'lar bir xil nomli property'ga ega bo'lsa, tiplar aynan bir xil bo'lishi kerak. `string` va `"bark"` — `"bark"` `string`'ga assign bo'lsa ham, identik emas, demak xato.

**Child override**'i esa subtype'ga toraytirishga ruxsat beradi. Sabab: child o'z declaration'i bilan ikkala parent property'ni qoplaydi, demak conflict o'rniga bitta aniq tip qoladi:

```typescript
interface HasName { label: string; }
interface HasLabel { label: string; }

// ✅ ikki parent identik tip — conflict yo'q
interface Tag extends HasName, HasLabel {
  label: "primary" | "secondary"; // ✅ child override subtype — ruxsat
}
```

**Diamond problem yo'q:** Interface'lar faqat shape, implementation yo'q — shuning uchun diamond problem (bir nechta parent orqali bir xil method kelishi) mavjud emas. Har interface faqat type contract, method body esa class'da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Mixin pattern:

```typescript
interface Serializable {
  serialize(): string;
}

interface Cloneable<T> {
  clone(): T;
}

interface Comparable<T> {
  compareTo(other: T): number;
}

// Uchta capability'ni jamlash
interface DataModel extends Serializable, Cloneable<DataModel>, Comparable<DataModel> {
  id: number;
  name: string;
}

class User implements DataModel {
  constructor(public id: number, public name: string) {}

  serialize(): string {
    return JSON.stringify({ id: this.id, name: this.name });
  }

  clone(): DataModel {
    return new User(this.id, this.name);
  }

  compareTo(other: DataModel): number {
    return this.id - other.id;
  }
}
```

React props — multiple interface composition:

```typescript
interface HasId { id: string; }
interface HasClassName { className?: string; }
interface HasStyle { style?: React.CSSProperties; }
interface HasChildren { children?: React.ReactNode; }

interface ButtonProps extends HasId, HasClassName, HasStyle, HasChildren {
  onClick: () => void;
  disabled?: boolean;
}

function Button({ id, className, style, children, onClick, disabled }: ButtonProps) {
  return (
    <button id={id} className={className} style={style} onClick={onClick} disabled={disabled}>
      {children}
    </button>
  );
}
```

Conflict hal qilish:

```typescript
interface HasName { name: string; }
interface HasLabel { name: string; } // xato emas — bir xil tip

interface Item extends HasName, HasLabel {
  // name — conflictsiz, ikkala parent'dan bir xil tip
  value: number;
}

// Agar tiplar farq qilsa
interface NumericVersion { version: number; }
interface SemverVersion { version: string; }

// ❌ interface Package extends NumericVersion, SemverVersion {} — xato
// ✅ Yechim: dizaynni qayta ko'rib chiqish
interface Package extends NumericVersion {
  // Faqat NumericVersion'dan version: number
  // SemverVersion'ni extend qilmaslik yoki property nomini o'zgartirish
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface HasName { name: string; }
interface HasAge { age: number; }

interface User extends HasName, HasAge {
  id: number;
}

const user: User = {
  id: 1,
  name: "Ali",
  age: 25,
};
```

```javascript
// Compiled JS — barcha interface'lar va extends chain o'chiriladi
const user = {
  id: 1,
  name: "Ali",
  age: 25,
};

// Multiple extends — faqat compile-time type checking
// Runtime'da oddiy object
```

</details>

---

## Interface Merging — Declaration Merging

### Nazariya

Declaration merging — TypeScript'ning **faqat interface'ga xos** xususiyati. Bir xil nomli ikki interface avtomatik **birlashadi** — barcha property'lar bitta interface'ga to'planadi. Type alias'da bu mumkin emas (Duplicate identifier xatosi).

```typescript
// Birinchi e'lon
interface User {
  name: string;
  age: number;
}

// Ikkinchi e'lon — bir xil nom
interface User {
  email: string;
  phone?: string;
}

// Natija — avtomatik birlashdi:
// interface User {
//   name: string;
//   age: number;
//   email: string;
//   phone?: string;
// }

const user: User = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
  // phone optional
};
```

**Merge qoidalari:**

1. **Farqli property'lar** — oddiy birlashadi
2. **Bir xil nomli property** — tipi **aynan** bir xil bo'lishi kerak (subtype yetarli emas)
3. **Method'lar** — overload sifatida birlashadi, **keyingi declaration birinchi bo'ladi**

```typescript
// ❌ Bir xil property, farqli tip
interface Config { port: number; }
interface Config { port: string; }
// ❌ Subsequent property declarations must have the same type

// ✅ Method overload merging
interface EventEmitter {
  on(event: "click", handler: () => void): void;
}

interface EventEmitter {
  on(event: "hover", handler: () => void): void;
}

// Natija: ikkita overload, lekin TARTIB qaytariladi
// interface EventEmitter {
//   on(event: "hover", handler: () => void): void; // keyingi — birinchi
//   on(event: "click", handler: () => void): void; // oldingi — ikkinchi
// }
```

Declaration merging'ning **asosiy foydasi** — **library augmentation**. Mavjud library tiplariga yangi property yoki method qo'shish.

<details>
<summary><strong>Under the Hood</strong></summary>

Declaration merging `binder.ts` bosqichida amalga oshadi. Parser bir xil nomli bir nechta `InterfaceDeclaration` ko'rsa, ular bitta `Symbol`'ga merge bo'ladi.

**Binder process:**

```
1. Birinchi `interface User {...}` → Symbol("User") yaratiladi
   flags: SymbolFlags.Interface | SymbolFlags.Type

2. Ikkinchi `interface User {...}` uchraydi
   - binder: existing Symbol topildi
   - existing Symbol.declarations array'iga yangi InterfaceDeclaration qo'shiladi
   - Symbol.members (SymbolTable) yangi property'lar bilan merge qilinadi

3. Checker `InterfaceType`'ni quradi
   - Symbol.declarations'dagi barcha declaration'larni iteratsiya
   - Har declaration'dan property'larni yig'adi
   - Bir xil nomli property'lar uchun `isTypeIdenticalTo` tekshiruvi
```

**Method overload tartibining sababi:**

Keyingi declaration'dagi method overload'lar **birinchi** ro'yxatga qo'shiladi (reverse order). Bitta declaration ichidagi overload'larning o'zaro tartibi saqlanadi, lekin guruhlar darajasida keyingi declaration oldingisidan oldinroq turadi. Bu TypeScript'ning overload resolution algoritmi bilan bog'liq — compiler overload'larni yuqoridan pastga iteratsiya qiladi va birinchi mos keluvchini tanlaydi.

**Istisno — string literal parameter:** agar overload'ning parameter'i bitta string literal type bo'lsa (`on(event: "click")`), bu signature merged overload ro'yxatining yuqorisiga ko'tariladi. Shuning uchun barcha overload'lari string literal bo'lgan misolda resolution baribir aniq mos keluvchini topadi — tartib amaliy natijaga ta'sir qilmaydi. Tartib faqat bir nechta overload bir argument'ga mos kelganda muhim bo'ladi.

**Library augmentation mexanizmi:**

```typescript
// Express.js Request interface'iga custom field qo'shish
import "express"; // original module

declare global {
  namespace Express {
    interface Request {
      userId?: string;
      role?: "admin" | "user";
    }
  }
}
```

Bu `declare global { ... }` ambient declaration — compiler bu block'ni global scope'ga qo'shadi. Global namespace'dagi `Express.Request` interface'i merge bo'ladi — Express'ning original definition'i + sizning custom field'lar.

**Type alias merging yo'qligi:**

Type alias uchun merging yo'q, chunki type alias `TypeAliasDeclaration` — u o'zi bir butun **type expression**'ni ifodalaydi. Ikki marta e'lon qilish — ikki xil type expression bo'lar edi, qaysi birini tanlash aniq emas.

Interface esa `InterfaceType` — property'lar to'plami. Property qo'shish/merge qilish semantik jihatdan aniq — `SymbolTable`'ga yangi entry qo'shish.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Simple merging:

```typescript
// Alohida fayllarda yoki bitta faylda — bir xil nom
interface User {
  name: string;
}

interface User {
  age: number;
}

interface User {
  email: string;
}

// Final: { name, age, email }
const user: User = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
};
```

Express.js augmentation:

```typescript
// types/express.d.ts
declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        role: "admin" | "user";
      };
      requestId: string;
    }
  }
}

// Fayl module bo'lishi uchun export shart, aks holda declare global ishlamaydi:
export {};
```

Endi har route handler'da:

```typescript
import express from "express";

const app = express();

// Middleware — user qo'shish
app.use((req, res, next) => {
  req.requestId = generateId();
  req.user = { id: "123", role: "user" };
  next();
});

// Handler'da — TS augmented type'ni taniydi
app.get("/profile", (req, res) => {
  console.log(req.user?.id);    // ✅ TS biladi
  console.log(req.requestId);    // ✅ TS biladi
});
```

Window object augmentation:

```typescript
// types/global.d.ts
declare global {
  interface Window {
    analytics: {
      track(event: string, data?: object): void;
    };
    __DEV__: boolean;
  }
}

export {};
```

```typescript
// app.ts
window.analytics.track("page_view"); // ✅ TS biladi

if (window.__DEV__) {
  console.log("Development mode");
}
```

Method overload merging:

```typescript
interface Handler {
  on(event: "click"): void;
}

interface Handler {
  on(event: "hover"): void;
}

interface Handler {
  on(event: "scroll"): void;
}

// Merged:
// interface Handler {
//   on(event: "scroll"): void;  // keyingi — birinchi
//   on(event: "hover"): void;
//   on(event: "click"): void;   // birinchi — oxirgi
// }

const handler: Handler = {
  on(event) { /* ... */ }
};
handler.on("click");   // ✅
handler.on("hover");   // ✅
handler.on("scroll");  // ✅
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface User { name: string; }
interface User { email: string; }
interface User { age: number; }

const user: User = {
  name: "Ali",
  email: "ali@mail.com",
  age: 25,
};
```

```javascript
// Compiled JS — barcha interface declaration'lar TO'LIQ o'chiriladi
const user = {
  name: "Ali",
  email: "ali@mail.com",
  age: 25,
};

// Declaration merging — faqat compile-time
// Runtime'da oddiy object, hech qanday merge metadata yo'q
```

</details>

---

## Interface vs Type Alias

### Nazariya

Interface va type alias — ikkalasi ham object shape'ni tavsiflaydi. Farqlar bor va vaziyatga qarab biri afzal bo'ladi.

```typescript
// Interface
interface User {
  name: string;
  age: number;
}

// Type alias
type UserType = {
  name: string;
  age: number;
};

// Object shape uchun ikkalasi ham ishlaydi
const userViaInterface: User = { name: "Ali", age: 25 };
const userViaType: UserType = { name: "Ali", age: 25 };
```

### Farqlar jadvali

| Xususiyat | Interface | Type Alias |
|-----------|-----------|------------|
| Object shape | ✅ | ✅ |
| Extend (meros) | `extends` keyword | `&` (intersection) |
| Declaration merging | ✅ Mumkin | ❌ Mumkin emas |
| `implements` (class) | ✅ | ✅ (lekin union emas) |
| Union types | ❌ | ✅ `type A = string \| number` |
| Intersection types | ❌ (faqat extends) | ✅ `type A = B & C` |
| Mapped types | ❌ | ✅ `{ [K in keyof T]: ... }` |
| Conditional types | ❌ | ✅ `T extends U ? X : Y` |
| Primitive alias | ❌ | ✅ `type ID = string` |
| Tuple types | ❌ | ✅ `type Pair = [string, number]` |
| Template literal | ❌ | ✅ `` type Key = `prefix-${string}` `` |
| Error message da nom | ✅ Doim ko'rinadi | ⚠️ Ba'zan inline expand |

### Extend vs Intersection — muhim farq

```typescript
// Interface extending
interface Animal { name: string; }
interface Dog extends Animal { breed: string; }

// Type intersection — o'xshash natija
type AnimalType = { name: string; };
type DogType = AnimalType & { breed: string; };
```

**Conflict holati farqi:**

```typescript
// Interface — conflict aniqlanadi, xato
interface Product { price: number; }
interface DiscountedProduct extends Product {
  price: string; // ❌ compile error: string ≠ number
}

// Type — "silent never"
type ProductType = { price: number; };
type DiscountedProductType = ProductType & { price: string };
// DiscountedProductType["price"] type: never (number & string = never)
// Diqqat: compile xato yo'q, lekin property foydasiz — qiymat berib bo'lmaydi
```

### Qachon qaysi birini ishlatish

**Interface ishlatish:**
1. Object shape uchun (asosiy holat)
2. Class bilan `implements` kerak bo'lsa
3. Declaration merging kerak bo'lsa (library augmentation)
4. `extends` bilan ko'p merosxo'rlik

**Type alias ishlatish:**
1. Union — `type ID = string | number`
2. Tuple — `type Pair = [string, number]`
3. Mapped type — `type P<T> = { [K in keyof T]?: T[K] }`
4. Conditional — `type IsString<T> = T extends string ? true : false`
5. Primitive alias — `type UserId = string`
6. Funksiya tip — `type Handler = (e: Event) => void`
7. Template literal — `` type Path = `/${string}` ``

**Umumiy qoida:** Object shape uchun — **interface** (eng ko'p holat). Boshqa type'lar uchun — **type alias**. Agar ikkalasi ham mos bo'lsa — **interface** (convention).

<details>
<summary><strong>Under the Hood</strong></summary>

Compiler interface va type alias'ni turli AST node va data structure'larda saqlaydi:

- **Interface** → `InterfaceDeclaration` + `InterfaceType` (+ `SymbolTable`)
- **Type alias** → `TypeAliasDeclaration` + lazy evaluated type expression

**Performance farqi:**

```
Interface:                           Type Alias:
┌──────────────────────┐            ┌──────────────────────┐
│ InterfaceType         │            │ TypeAliasDeclaration  │
│ ┌──────────────────┐ │            │                       │
│ │ SymbolTable      │ │            │ type expression       │
│ │ (hash map)       │ │            │ (AST tree)            │
│ │ name → Symbol    │ │            │                       │
│ │ age  → Symbol    │ │            │ Cached on first       │
│ └──────────────────┘ │            │ resolution            │
│                       │            │                       │
│ Eager resolution      │            │ Lazy but cached       │
└──────────────────────┘            └──────────────────────┘

Property lookup:
  Interface → O(1) hash map lookup
  Type alias → expression traversal (ba'zi holatlarda tezroq cached)
```

**Eski kitoblarda** "interface eager, type alias lazy" aytishadi, lekin zamonaviy TypeScript'da farq **juda kam**. Ikkalasi ham cached. Asosiy farq — **declaration merging** va **semantic expressiveness** (type alias kengroq expressions — union, mapped, conditional).

**Error message nuance:**

TS 4.2'gacha type alias'lar error message'larda ba'zan **inline expand** qilinardi (alias nomi yo'qolar edi). TS 4.2+ da bu yaxshilandi — alias nomi saqlanadi **asosan**, lekin ba'zi holatlarda hanuz expand bo'ladi:

- **Saqlanadi:** oddiy type alias (`type User = { ... }`)
- **Expand bo'lishi mumkin:** intersection (`type X = A & B`), mapped type (`type X = { [K in keyof T]: ... }`), conditional type natijalari

Interface esa **doim** error'da nomi bilan ko'rinadi — chunki interface type'i `InterfaceType.symbol.name` orqali saqlanadi.

**Class implements farqi:**

```typescript
// Interface — implements mumkin
interface ILogger {
  log(msg: string): void;
}

class ConsoleLogger implements ILogger {
  log(msg: string) { console.log(msg); }
}

// Type alias (object shape) — implements mumkin
type LoggerType = {
  log(msg: string): void;
};

class FileLogger implements LoggerType {
  log(msg: string) { /* ... */ }
}

// Type alias (union) — implements MUMKIN EMAS
type StringOrNumber = string | number;
// class Foo implements StringOrNumber {} // ❌
```

Class `implements` clause faqat **object shape**'ni qabul qiladi — union, primitive, tuple bo'lsa xato. Shuning uchun type alias (object shape) ham ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Object shape — ikkalasi ham:

```typescript
// Interface
interface User {
  id: number;
  name: string;
}

// Type alias — bir xil
type UserType = {
  id: number;
  name: string;
};

// Class implements — ikkalasi ham
class UserImpl implements User { // yoki `implements UserType`
  constructor(public id: number, public name: string) {}
}
```

Faqat type alias — union, tuple, primitive:

```typescript
// Union — interface'da mumkin emas
type Status = "idle" | "loading" | "success" | "error";
type StringOrNumber = string | number;

// Tuple — interface'da mumkin emas
type Coordinate = [number, number];
type Entry<T> = [string, T];

// Primitive alias — interface'da mumkin emas
type UserId = string;
type Age = number;

// Function type — ikkalasi ham mumkin, type oddiyroq
type Handler = (event: Event) => void;
// yoki:
interface HandlerIface {
  (event: Event): void;
}

// Mapped type — interface'da mumkin emas
type Partial<T> = { [K in keyof T]?: T[K] };
type Readonly<T> = { readonly [K in keyof T]: T[K] };

// Conditional type
type IsArray<T> = T extends any[] ? true : false;
```

Faqat interface — declaration merging:

```typescript
// Library augmentation
declare module "express" {
  interface Request {
    userId?: string;
  }
}

// Global augmentation
declare global {
  interface Window {
    dataLayer: unknown[];
  }
}

// Type alias bilan mumkin emas:
// declare module "express" {
//   type RequestExtension = { userId?: string };
// } // bunday augmentation yo'q
```

Refactoring — interface dan type'ga (kerak bo'lsa):

```typescript
// Interface
interface User {
  id: number;
  name: string;
}

// Type alias'ga o'tkazish — aniq foyda bo'lganda
type User = {
  id: number;
  name: string;
};

// Masalan, union'ga kengaytirish uchun:
type UserOrGuest =
  | { type: "user"; id: number; name: string }
  | { type: "guest" };
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface User {
  name: string;
}

type UserType = {
  name: string;
};

const userViaInterface: User = { name: "Ali" };
const userViaType: UserType = { name: "Ali" };
```

```javascript
// Compiled JS — interface VA type alias HAM to'liq o'chiriladi
const userViaInterface = { name: "Ali" };
const userViaType = { name: "Ali" };

// Ikkalasi ham faqat compile-time construct'lar
// Runtime'da farq yo'q — oddiy JS object
```

</details>

---

## Excess Property Checking

### Nazariya

Excess property checking — TypeScript'ning maxsus mexanizmi. **Object literal** (to'g'ridan-to'g'ri yozilgan object) kutilgan tipda **mavjud bo'lmagan** property bor bo'lsa, TS xato beradi. Bu **faqat literal**'da ishlaydi — variable orqali berilganda ishlamaydi.

```typescript
interface User {
  name: string;
  age: number;
}

// ❌ Object literal — excess property check
const user: User = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com", // ❌ Object literal may only specify known properties
};

// ✅ Variable orqali — excess property check YO'Q
const data = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
};
const user2: User = data; // ✅ Hech qanday xato
// Structural typing: name va age bor — User shape'iga mos
// email qo'shimcha — variable orqali tekshirilmaydi
```

**Nima uchun bu farq bor:** TypeScript'ning structural typing g'oyasi — "kerakli property'lar bormi?" — bor bo'lsa mos. Qo'shimcha property'lar muammo emas. Lekin **object literal**'da qo'shimcha property ko'pincha **typo** yoki xato — shuning uchun TS faqat literal'da tekshiradi. Bu "typo catcher" sifatida ishlaydi.

```typescript
interface Config {
  host: string;
  port: number;
}

// ❌ Typo — "hoost" o'rniga "host"
const config: Config = {
  hoost: "localhost", // ❌ TS ushlaydi
  port: 3000,
  // host property yo'q — majburiy
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

Excess property checking `checker.ts`'dagi `checkObjectLiteral()` funksiyasida amalga oshadi. Bu tekshiruv **faqat object literal** kontekstida ishga tushadi — variable reference'da yoki funksiya argumentida object literal bo'lsa.

**Algoritm:**

1. Source object literal'dan barcha property'larni olish
2. Target type'dan barcha property'larni olish
3. Source'ning har property'si uchun — target'da mavjudmi?
4. Agar **mavjud emas** va target'ning index signature'i ham yo'q bo'lsa — excess property error

```
checkObjectLiteralProperties(source, target):
  for prop in source.properties:
    if prop.name not in target.properties:
      if not target.hasIndexSignature:
        error("Object literal may only specify known properties")
```

**Literal vs variable farqi:**

```typescript
const user: User = {...}; // Literal — excess check ishlaydi
const data = {...};       // Variable — type inferred
const user: User = data;  // Variable reference — excess check YO'Q
```

Compiler literal va non-literal'ni "freshness" tushunchasi orqali ajratadi. To'g'ridan-to'g'ri yozilgan object literal — **fresh** type, o'zgaruvchiga assign qilingach esa reference **regular** type'ga aylanadi (freshness yo'qoladi). Fresh type'da excess check, regular type'da faqat structural compatibility tekshiriladi.

**Bypass mexanizmlari:**

1. **Variable orqali** — literal fresh'likni yo'qotadi
2. **Type assertion** — `as T` bilan kompileri aldash
3. **Index signature** — target type'da `[key: string]: unknown` bo'lsa

```typescript
interface Flexible {
  name: string;
  [key: string]: unknown; // index signature
}

const profile: Flexible = {
  name: "Ali",
  bio: "developer", // ✅ index signature orqali
};
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Excess property — typo catcher:

```typescript
interface Config {
  host: string;
  port: number;
  timeout?: number;
}

// ❌ Typo
const config: Config = {
  host: "localhost",
  port: 3000,
  timout: 5000, // ❌ "timeout" o'rniga "timout" — typo ushlanadi
};
```

Funksiya argumentida:

```typescript
function greet(user: { name: string }) {
  return `Hello, ${user.name}`;
}

// ❌ Object literal argument — excess check
greet({ name: "Ali", age: 25 });
// ❌ Object literal may only specify known properties

// ✅ Variable orqali — check yo'q
const person = { name: "Ali", age: 25 };
greet(person); // ✅
```

Workaround'lar:

```typescript
interface Point {
  x: number;
  y: number;
}

// Yechim 1: Variable ga avval assign
const rawPoint = { x: 10, y: 20, z: 30 };
const fromVariable: Point = rawPoint; // ✅

// Yechim 2: Type assertion (xavfli)
const fromAssertion = { x: 10, y: 20, z: 30 } as Point; // ✅ lekin xavfli
// ⚠️ Boshqa xatolarni ham yashiradi

// Yechim 3: Index signature — ishonchli
interface FlexPoint {
  x: number;
  y: number;
  [key: string]: number; // qo'shimcha number property'lar ruxsat
}
const flexPoint: FlexPoint = { x: 10, y: 20, z: 30 }; // ✅
```

Real-world: React props:

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
}

function Button(props: ButtonProps) { /* ... */ }

// ❌ data-* attribute yozish uchun
// Button({ label: "Click", onClick: () => {}, "data-testid": "btn" });
// ❌ 'data-testid' does not exist

// ✅ Yechim — index signature qo'shish
interface ButtonProps2 {
  label: string;
  onClick: () => void;
  [key: `data-${string}`]: string | undefined; // template literal key
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface User { name: string; age: number; }

// Compile-time: excess property check
const user: User = {
  name: "Ali",
  age: 25,
  // email: "ali@mail.com", // ❌ TS xato
};

// Variable bypass
const data = { name: "Ali", age: 25, email: "ali@mail.com" };
const user2: User = data; // ✅
```

```javascript
// Compiled JS — excess property check faqat compile-time
const user = { name: "Ali", age: 25 };

const data = { name: "Ali", age: 25, email: "ali@mail.com" };
const user2 = data;
// email data'da saqlanadi — runtime'da mavjud
// TS'da tekshirilmagan — silent
```

Muhim: variable bypass paytida qo'shimcha property **JavaScript'da qoladi**. TS tekshirmagan bo'lsa ham, runtime'da `user2.email` mavjud. Bu siz xohlagan narsami yoki yo'qmi — kontekstga bog'liq.

</details>

---

## Recursive Interfaces

### Nazariya

Recursive interface — interface o'ziga o'zi havola qiladi. Bu daraxt (tree), ro'yxat (linked list), yoki har qanday nested (ichma-ich) struktura uchun kerak.

```typescript
// Fayl tizimi — papka ichida papka
interface FileSystemNode {
  name: string;
  type: "file" | "directory";
  size?: number;
  children?: FileSystemNode[]; // ← recursive
}

const root: FileSystemNode = {
  name: "src",
  type: "directory",
  children: [
    { name: "index.ts", type: "file", size: 1024 },
    {
      name: "components",
      type: "directory",
      children: [
        { name: "Button.tsx", type: "file", size: 512 },
      ],
    },
  ],
};
```

Recursive pattern'lar — daraxt, linked list, JSON:

```typescript
// Linked list
interface LinkedListNode<T> {
  value: T;
  next: LinkedListNode<T> | null;
}

// JSON — recursive type
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };
```

Interface vs type alias recursive reference'da: **interface** afzalroq, chunki compiler circular reference'ni aniqroq handle qiladi. Type alias'da ba'zi recursive pattern'lar `Type instantiation is excessively deep` xatosiga olib kelishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript compiler recursive type'larni tekshirganda cheksiz loop'ga tushmaslik uchun **depth counter** va **lazy resolution** ishlatadi.

**Lazy resolution:**

```typescript
interface TreeNode {
  children: TreeNode[]; // ← o'ziga reference
}
```

Compiler bu reference'ni darhol resolve qilmaydi. Interface `TreeNode`'ni Symbol sifatida saqlaydi, `children` property'ning tipi — "type reference" (deferred). Faqat qiymat assign qilinganda yoki property access bo'lganda actual tekshirish boshlanadi.

```
Resolution flow:

interface TreeNode {
  children: TreeNode[]
}
        │
        ▼
  Symbol("TreeNode") — placeholder
        │
  Property "children": TypeReference("TreeNode", array)
  (lazy — hozircha resolve qilinmagan)
        │
        ▼
  Faqat ishlatilganda resolve
```

**Depth limit:**

Compiler recursive type instantiation'ni belgilangan depth'gacha ruxsat beradi. Bu `checker.ts`'dagi `instantiationDepth` va `instantiationCount` counter'lari orqali boshqariladi. Agar chegara oshib ketsa — `Type instantiation is excessively deep and possibly infinite` xatosi.

Bu xato ayniqsa conditional type bilan recursive type alias'larda uchraydi:

```typescript
// Chuqurlik xavfi bor
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>  // ← har nesting +1 depth
    : T[K];
};

// Juda chuqur type'larda limit'ga yetishi mumkin
```

**Interface vs type alias:**

Interface'lar recursive reference'da **afzalroq** — compiler interface'ni "named type" sifatida track qiladi va circular reference'ni identity orqali taniydi (chunki interface — declaration, type expression emas). Shu sababli interface o'ziga havola qilganda qayta-qayta expand qilinmaydi.

Type alias'lar esa inline expand qilinish moyil — chuqur recursion'da compiler `instantiationDepth`'ga tezroq yetadi.

**Runtime:** Interface va type recursive reference'lar JS'ga compile bo'lganda butunlay yo'qoladi. Recursive struktura JavaScript'da oddiy nested object sifatida ishlaydi — hech qanday type metadata qolmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tree struktura:

```typescript
interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[];
}

function countNodes<T>(node: TreeNode<T>): number {
  return 1 + node.children.reduce((sum, child) => sum + countNodes(child), 0);
}

const tree: TreeNode<string> = {
  value: "root",
  children: [
    { value: "a", children: [] },
    {
      value: "b",
      children: [
        { value: "b1", children: [] },
        { value: "b2", children: [] },
      ],
    },
  ],
};

console.log(countNodes(tree)); // 5
```

Linked list:

```typescript
interface ListNode<T> {
  value: T;
  next: ListNode<T> | null;
}

function fromArray<T>(arr: T[]): ListNode<T> | null {
  if (arr.length === 0) return null;
  return {
    value: arr[0],
    next: fromArray(arr.slice(1)),
  };
}

function toArray<T>(list: ListNode<T> | null): T[] {
  const result: T[] = [];
  let current = list;
  while (current !== null) {
    result.push(current.value);
    current = current.next;
  }
  return result;
}

const list = fromArray([1, 2, 3, 4]);
// { value: 1, next: { value: 2, next: { value: 3, next: { value: 4, next: null } } } }

console.log(toArray(list)); // [1, 2, 3, 4]
```

JSON type:

```typescript
type JsonPrimitive = string | number | boolean | null;
type JsonArray = JsonValue[];
type JsonObject = { [key: string]: JsonValue };
type JsonValue = JsonPrimitive | JsonArray | JsonObject;

function parseJson(text: string): JsonValue {
  return JSON.parse(text);
}

function stringify(value: JsonValue): string {
  return JSON.stringify(value);
}

const data: JsonValue = {
  name: "Ali",
  age: 25,
  friends: ["Vali", "Soli"],
  address: {
    city: "Tashkent",
    zip: 100000,
  },
  active: true,
  deleted: null,
};
```

Comment tree (real-world — threaded comments):

```typescript
interface Comment {
  id: number;
  author: string;
  text: string;
  timestamp: Date;
  replies: Comment[];
}

function findComment(comments: Comment[], id: number): Comment | null {
  for (const comment of comments) {
    if (comment.id === id) return comment;
    const found = findComment(comment.replies, id);
    if (found !== null) return found;
  }
  return null;
}

function countAll(comments: Comment[]): number {
  return comments.reduce(
    (total, comment) => total + 1 + countAll(comment.replies),
    0
  );
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
interface TreeNode {
  value: string;
  children: TreeNode[];
}

const tree: TreeNode = {
  value: "root",
  children: [
    { value: "child1", children: [] },
    { value: "child2", children: [{ value: "grand", children: [] }] },
  ],
};

function countNodes(node: TreeNode): number {
  return 1 + node.children.reduce((sum, child) => sum + countNodes(child), 0);
}
```

```javascript
// Compiled JS — interface to'liq o'chiriladi
const tree = {
  value: "root",
  children: [
    { value: "child1", children: [] },
    { value: "child2", children: [{ value: "grand", children: [] }] },
  ],
};

function countNodes(node) {
  return 1 + node.children.reduce((sum, child) => sum + countNodes(child), 0);
}

// Recursive struktura — oddiy nested JS object
// Type metadata yo'q, faqat compile-time'da shape tekshirilgan
```

</details>

---

## PropertyKey Type

### Nazariya

`PropertyKey` — TypeScript'ning built-in type alias. Object property key sifatida ishlatilishi mumkin bo'lgan barcha tiplarning union'i:

```typescript
// lib.es5.d.ts'da:
type PropertyKey = string | number | symbol;
```

Bu JavaScript'ning cheklovidir — object key faqat shu 3 tip bo'lishi mumkin. Boshqa tip (boolean, object, function) key sifatida ishlatilsa — runtime'da `String()` orqali string'ga convert bo'ladi.

```javascript
// JavaScript runtime — TS tip cheklovisiz
const store = {};
store[true] = 1;          // runtime: store["true"] — boolean string'ga coerce
store[42] = 2;            // runtime: store["42"] — number string'ga coerce
store[Symbol("id")] = 3;  // symbol — alohida, coerce qilinmaydi
```

`PropertyKey` qaerda ishlatiladi:

```typescript
// 1. Generic constraint — key tipini cheklash
function setProperty<K extends PropertyKey, V>(
  obj: Record<K, V>,
  key: K,
  value: V
) {
  obj[key] = value;
}

// 2. Record'da
type AnyKeyMap = Record<PropertyKey, unknown>;
// { [x: string]: unknown; [x: number]: unknown; [x: symbol]: unknown }
```

**Muhim cheklov:** `PropertyKey` union type, va TypeScript **index signature parameter'da union type'ni qabul qilmaydi**:

```typescript
// ❌ Index signature'da union — mumkin emas
interface Wrong {
  [key: PropertyKey]: unknown;
  // Error: An index signature parameter type cannot be a union type.
  //        Consider using a mapped object type instead.
}

// ✅ Har tip uchun alohida index signature
interface Right {
  [key: string]: unknown;
  [key: number]: unknown;
  [key: symbol]: unknown; // symbol index signature ham alohida yoziladi (TS 4.4+)
}

// ✅ Record ishlaydi (mapped type)
type AnyMap = Record<PropertyKey, unknown>;
```

`keyof any` ifodasi `PropertyKey`'ga teng:

```typescript
type Any = keyof any; // string | number | symbol === PropertyKey
```

Bu generic constraint'da ko'p ishlatiladi — `K extends keyof any` va `K extends PropertyKey` aynan bir xil ma'no.

<details>
<summary><strong>Under the Hood</strong></summary>

`PropertyKey` — `lib.es5.d.ts` faylida aniqlangan global type alias:

```typescript
type PropertyKey = string | number | symbol;
```

Bu ECMAScript spetsifikatsiyasiga mos keladi — `Object.getOwnPropertyNames()` faqat string qaytaradi, `Object.getOwnPropertySymbols()` faqat symbol, `Reflect.ownKeys()` esa barcha 3 tip'ni qaytaradi.

**Compiler index type checking:**

```typescript
obj[key] access uchun:

1. key type'ni aniqlash
2. obj type'ning index signature'ni tekshirish
3. key PropertyKey'ga assignable bo'lishi kerak
4. Natija: index signature value tipi (yoki undefined)
```

Checker `checkElementAccessExpression()`'da har property access'da key'ning `PropertyKey` subtype ekanligini tekshiradi:

```typescript
obj[someNumber];     // OK — number ⊂ PropertyKey
obj[someSymbol];     // OK — symbol ⊂ PropertyKey
obj[someString];     // OK — string ⊂ PropertyKey
// obj[someObject];  // ❌ object ⊄ PropertyKey
// obj[someBoolean]; // ❌ boolean ⊄ PropertyKey
```

**Nima uchun index signature'da union ishlamaydi:**

```typescript
interface Bad {
  [key: string | number]: unknown; // ❌
}
```

Sabab: compiler har index signature'ni alohida `IndexInfo` sifatida saqlaydi — har birining o'z key type'i (`string`, `number`, `symbol` yoki template literal pattern) bilan. Union key type bitta `IndexInfo`'ga to'g'ri kelmaydi, shuning uchun union o'rniga har tip uchun alohida signature yozish kerak. Bu cheklov mapped type bilan chetlanadi — `Record<PropertyKey, V>` union key'ni har a'zoga distribute qiladi.

**Number index va string coercion:**

JavaScript'da `obj[0]` aslida `obj["0"]` — number key runtime'da string'ga coerce bo'ladi. Lekin TypeScript compiler `number` index signature'ni alohida track qiladi, bu Array va Tuple type'lar uchun aniqroq type inference beradi:

```typescript
// Array: number index → element type
const arr: string[] = ["a", "b"];
arr[0]; // string

// Record: string index
const record: Record<string, number> = { a: 1 };
record.a; // number
```

**`keyof any` va `PropertyKey`:**

```typescript
type KeyofAny = keyof any;     // string | number | symbol
type PropKey = PropertyKey;    // string | number | symbol
type AreEqual = KeyofAny extends PropKey ? true : false; // true (same)
```

Bular bir xil tip. Ko'p kodbazalarda `K extends PropertyKey` yoziladi — nom aniqroq. `K extends keyof any` — eski stil (TS 2.x), hozir ham ishlaydi.

**Runtime:** `PropertyKey` type alias JS'ga compile bo'lganda butunlay yo'qoladi. Bu faqat compile-time type checking uchun mavjud.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Generic constraint — key ob'ektning haqiqiy kalitlaridan biri (keyof T ⊂ PropertyKey)
function pick<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;
  for (const key of keys) {
    result[key] = obj[key];
  }
  return result;
}

const user = { id: 1, name: "Ali", email: "ali@mail.com", age: 25 };
const subset = pick(user, ["id", "name"]);
// subset type: { id: number; name: string }
// (user literal'da id qiymati number'ga widen bo'ladi, `1` literal emas)
```

Record bilan:

```typescript
// Har xil key tipi — Record<PropertyKey, V>
type UniversalMap = Record<PropertyKey, unknown>;

const cache: UniversalMap = {};
cache["stringKey"] = "value1";
cache[42] = "value2";
cache[Symbol("id")] = "value3";
```

`keyof any` va `PropertyKey`:

```typescript
// Teng tip'lar
type KeyofAny = keyof any;     // string | number | symbol
type PropKey = PropertyKey;    // string | number | symbol

// Generic constraint — ikkala sintaksis ishlaydi
function identityViaKeyofAny<K extends keyof any>(key: K): K { return key; }
function identityViaPropertyKey<K extends PropertyKey>(key: K): K { return key; }

// Record'da `keyof any` ichki ishlatiladi
type Record<K extends keyof any, V> = {
  [P in K]: V;
};
```

Object key runtime coercion:

```typescript
const store: Record<PropertyKey, string> = {};

store["hello"] = "world";   // string key
store[42] = "forty-two";    // number key (runtime: "42")
store[Symbol("id")] = "symbol-value";

// Access
store["hello"];             // "world"
store[42];                   // "forty-two" (aslida store["42"])
store["42"];                // "forty-two" (bir xil — coerced)

// Symbol alohida
const idKey = Symbol("id");
store[idKey] = "symbolValue";
store[idKey];                 // "symbolValue" — faqat shu symbol bilan kirish mumkin
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source — PropertyKey built-in, qayta declare qilish shart emas
function getKey<K extends PropertyKey>(
  obj: Record<K, unknown>,
  key: K
): unknown {
  return obj[key];
}

const profile = { name: "Ali", email: "ali@mail.com" };
const result = getKey(profile, "name");
```

```javascript
// Compiled JS — PropertyKey va generic constraint o'chiriladi
function getKey(obj, key) {
  return obj[key];
}

const profile = { name: "Ali", email: "ali@mail.com" };
const result = getKey(profile, "name");

// Runtime'da hech qanday key type tekshiruvi yo'q
// JS'da har qanday qiymat key sifatida ishlatish mumkin
// (lekin runtime'da string yoki symbol'ga coerce bo'ladi)
```

</details>

---

## Edge Cases va Gotchas

Object va interface bilan ishlashdagi nozik holatlar. Ular ko'pincha kutilmagandek xulq sifatida namoyon bo'ladi.

### Gotcha 1: `readonly` shallow — nested mutable

```typescript
interface Config {
  readonly database: {
    host: string;
    port: number;
  };
}

const cfg: Config = {
  database: { host: "localhost", port: 5432 },
};

// cfg.database = { ... };       // ❌ readonly — OK
cfg.database.host = "remote";     // ⚠️ Mutable! readonly faqat shallow
cfg.database.port = 3306;         // ⚠️ Mutable!
```

**Sabab:** `readonly` modifier faqat **o'sha property'ni** himoyalaydi — ichki object'larga ta'sir qilmaydi. `cfg.database` reference qotirilgan (qayta assign mumkin emas), lekin `database` object'ning ichki property'lari alohida — ular `readonly` emas. Deep readonly uchun `DeepReadonly<T>` custom utility kerak.

---

### Gotcha 2: Excess property checking bypass variable orqali

```typescript
interface Point { x: number; y: number; }

// ❌ Literal — excess property check
const literalPoint: Point = { x: 10, y: 20, z: 30 };
// Error: Object literal may only specify known properties

// ✅ Variable orqali — bypass
const rawPoint = { x: 10, y: 20, z: 30 };
const fromVariable: Point = rawPoint; // ✅ Hech qanday xato!
// z property runtime'da QOLADI — TS tekshirmagan
```

**Sabab:** Excess property checking faqat **fresh object literal**'da ishlaydi. Variable orqali berilganda — bu "non-fresh" type, TS structural compatibility'ni tekshiradi (kerakli property'lar bormi?) va qo'shimcha property'lar muammo emas. Runtime'da qo'shimcha property object'da qoladi — silent bug manbai bo'lishi mumkin.

---

### Gotcha 3: Interface merging va ambient modules

```typescript
// Bir xil scope ichida (bitta fayl, yoki ambient declarations)
interface User { name: string; }

interface User {
  age: number;
  // Siz faqat "age" bilan ishlatmoqchisiz, lekin TS merged interface'ni ko'radi: { name, age }
}

const user: User = { age: 25 };
// ❌ Property 'name' is missing
// Sabab: yuqorida e'lon qilingan name property
```

**Sabab:** Interface merging **avtomatik** ishlaydi **bir xil scope** ichida — bitta fayl yoki ambient (`*.d.ts`) global declaration'larda. Modul fayllar (har biri `import`/`export` bilan) o'zaro merge qilinmaydi, lekin `declare global { interface User { ... } }` yoki `declare module "..."` bilan boshqa modullar tip'lariga qo'shilish mumkin. Library augmentation uchun bu foydali, lekin `*.d.ts` fayllarda tasodifiy merging katta kodbazalarda silent bug'larga olib keladi.

---

### Gotcha 4: `noUncheckedIndexedAccess` o'chiq — silent undefined

```typescript
// tsconfig.json: "noUncheckedIndexedAccess": false (default)

interface StringDictionary { [key: string]: string; }

const dictionary: StringDictionary = { hello: "world" };
const value = dictionary["nonexistent"]; // type: string — LEKIN runtime: undefined!

value.toUpperCase(); // ❌ Runtime: TypeError: Cannot read properties of undefined
// TS xato bermadi — silent crash

// ✅ Yechim: "noUncheckedIndexedAccess": true
// value: string | undefined — narrowing majburiy
```

**Sabab:** Default'da TS index access natijasini `string` deb ko'radi, chunki user o'zi `[key: string]: string` deb yozgan. Lekin runtime'da mavjud bo'lmagan key `undefined` qaytaradi. Bu silent gap — production kodda bug'larning asosiy manbasi. `noUncheckedIndexedAccess: true` bu xavfni compile-time'da ushlaydi.

---

### Gotcha 5: Interface default generic parameter

```typescript
interface Container<T = string> {
  value: T;
}

const stringContainer: Container = { value: "hello" };       // T = string (default)
const numberContainer: Container<number> = { value: 42 };    // T = number (explicit)

// Gotcha: explicit undefined bilan default ishlamaydi
// const undefinedContainer: Container<undefined> = { value: undefined };
// T = undefined (default emas, explicit undefined)

// Gotcha 2: default'ni o'zgartirish — backward compatibility muammosi
interface NumericContainer<T = number> { // default string → number bo'lib o'zgartirildi
  value: T;
}
// Oldingi kod: const legacyContainer: NumericContainer = { value: "hello" }; // ❌ endi xato
// Default o'zgartirish — breaking change
```

**Sabab:** Interface generic default parameter faqat type argument **butunlay tushirilganda** ishlaydi (`Container` — T = string). `Container<undefined>` — explicit, default emas. Default'ni o'zgartirish backward incompatible — barcha argumentsiz ishlatilgan joylarda tiplar o'zgaradi. Library'larda default parameter'lar deyarli hech qachon o'zgartirilmaydi.

---

## Common Mistakes

### ❌ Xato 1: Interface va type alias'ni noto'g'ri tanlash

```typescript
// ❌ Union uchun interface — mumkin emas
// interface StringOrNumber = string | number; // Syntax error

// ✅ Union uchun type alias
type StringOrNumber = string | number;

// ❌ Declaration merging uchun type alias — ishlamaydi
// type Config = { host: string };
// type Config = { port: number }; // Duplicate identifier

// ✅ Merging uchun interface
interface Config { host: string; }
interface Config { port: number; } // ✅ birlashadi
```

**Nima uchun:** Interface va type alias turli maqsadlar uchun. Object shape — interface (convention). Union, tuple, mapped, conditional — type alias. Ularni aralashtirish compile xatolariga olib keladi.

---

### ❌ Xato 2: `readonly`'ni deep deb o'ylash

```typescript
interface User {
  readonly profile: {
    name: string;
    age: number;
  };
}

const user: User = { profile: { name: "Ali", age: 25 } };

// ❌ "readonly profile — hammasi immutable" deb o'ylash
user.profile.name = "Vali"; // ⚠️ ISHLAYDI! readonly faqat shallow
user.profile.age = 30;       // ⚠️ ISHLAYDI!

// user.profile = { ... };  // ❌ Bu xato — profile reference qayta assign
```

**Nima uchun:** `readonly` — shallow immutability. Faqat o'sha property'ni (birinchi daraja) himoyalaydi, ichki object'larga ta'sir qilmaydi. JavaScript'ning `Object.freeze()` ham shallow. Deep immutability — custom `DeepReadonly<T>` type yoki Immer/Immutable.js library'lar kerak.

---

### ❌ Xato 3: Excess property checking qoidasini bilmaslik

```typescript
interface Point { x: number; y: number; }

// ❌ Literal — excess check
// const literalPoint: Point = { x: 1, y: 2, z: 3 }; // Error

// ✅ Variable — bypass
const rawPoint = { x: 1, y: 2, z: 3 };
const fromVariable: Point = rawPoint; // ✅ Hech qanday xato

// Bu inconsistency bilmaslik debug vaqtini oshiradi
```

**Nima uchun:** Excess property checking faqat fresh literal'da ishlaydi — typo ushlash uchun mo'ljallangan. Variable orqali berilganda structural typing ishlaydi — qo'shimcha property'lar "bonus", xato emas. Bu dizayn qarori: literal'da typo ushlash va variable orqali flexibility berish. Bilmasangiz — silent bug'lar ehtimoli ortadi.

---

### ❌ Xato 4: Optional property va `| undefined`'ni aralashtirish

```typescript
interface OptionalEmail { email?: string; }           // property bo'lmasligi mumkin
interface NullableEmail { email: string | undefined; } // property BOR, qiymati undefined bo'lishi mumkin

const withoutEmail: OptionalEmail = {};       // ✅ email yo'q
const missingEmail: NullableEmail = {};       // ❌ email kerak, hatto undefined bo'lsa ham

// `in` operator bilan farq
const optional: OptionalEmail = {};
"email" in optional; // false — property yo'q

const nullable: NullableEmail = { email: undefined };
"email" in nullable; // true — property bor, qiymati undefined
```

**Nima uchun:** `x?: string` — "x bo'lmasligi mumkin" (`{}` valid). `x: string | undefined` — "x majburiy, qiymati undefined bo'lishi mumkin" (`{ x: undefined }` kerak). `exactOptionalPropertyTypes: true` flag bu farqni yanada kuchaytiradi. API response'larda `undefined` va missing property'ni ajratish muhim — har ikkala variant farqli semantikaga ega.

---

### ❌ Xato 5: Index signature'da sababini bilmasdan cheklangan tiplar

```typescript
// ❌ Mustaqil index signature'da barcha qiymat tiplari bir xil bo'lishi kerak
interface Bad {
  [key: string]: number;
  name: string;    // ❌ string ⊄ number
  age: number;
}

// ✅ Index signature tipi kengroq union
interface Good {
  [key: string]: string | number;
  name: string;    // ✅ string ⊂ (string | number)
  age: number;     // ✅ number ⊂ (string | number)
}

// ✅ Yoki aniq property'lar + metadata uchun index
interface Mixed {
  id: number;
  name: string;
  [key: string]: unknown; // qo'shimcha metadata — har qanday qiymat
}
```

**Nima uchun:** Runtime'da `obj.name` ham `obj["name"]` ham **bitta** operatsiya — index access. Index signature har qanday string key uchun tipni belgilaydi. Agar explicit property tipi index signature tipidan kengroq bo'lsa, index access orqali olish va explicit access orqali olish farqli tiplar qaytarardi — bu unsound. TypeScript bu inconsistency'ni oldini olish uchun cheklov qo'yadi.

---

## Amaliy Mashqlar

### Mashq 1: Interface Extending (Oson)

**Savol:** Quyidagi interface'larni yarating:
- `Person` — `name: string`, `age: number`
- `User extends Person` — `email: string`, `isActive: boolean`
- `AdminUser extends User` — `permissions: string[]`, `level: 1 | 2 | 3`

<details>
<summary>Javob</summary>

```typescript
interface Person {
  name: string;
  age: number;
}

interface User extends Person {
  email: string;
  isActive: boolean;
}

interface AdminUser extends User {
  permissions: string[];
  level: 1 | 2 | 3;
}

const admin: AdminUser = {
  name: "Ali",
  age: 30,
  email: "ali@admin.com",
  isActive: true,
  permissions: ["read", "write", "delete"],
  level: 3,
};

// AdminUser'da: name, age (Person'dan) + email, isActive (User'dan) + permissions, level
```

**Tushuntirish:** Interface extending chain — har keyingi interface oldingisining barcha property'larini oladi va yangilarini qo'shadi. Child interface avtomatik ravishda parent'ning barcha constraint'larini meros oladi.

</details>

---

### Mashq 2: Excess Property Checking (O'rta)

**Savol:** Quyidagi 4 misoldan qaysilari xato beradi, qaysilari bermaydi? Nima uchun?

```typescript
interface Config { host: string; port: number; }

// Holat A
const directConfig: Config = { host: "localhost", port: 3000, debug: true };

// Holat B
const rawConfig = { host: "localhost", port: 3000, debug: true };
const indirectConfig: Config = rawConfig;

// Holat C
function startServer(config: Config) {}
startServer({ host: "localhost", port: 3000, debug: true });

// Holat D
function startServerFromVar(config: Config) {}
const serverOptions = { host: "localhost", port: 3000, debug: true };
startServerFromVar(serverOptions);
```

<details>
<summary>Javob</summary>

```typescript
// Holat A — ❌ XATO
const directConfig: Config = { host: "localhost", port: 3000, debug: true };
// Object LITERAL — excess property checking ishlaydi
// debug Config'da yo'q → xato

// Holat B — ✅ XATO YO'Q
const rawConfig = { host: "localhost", port: 3000, debug: true };
const indirectConfig: Config = rawConfig;
// Variable orqali — fresh emas, excess check yo'q
// Structural: host ✅, port ✅ — yetarli
// debug runtime'da qoladi — TS tekshirmagan

// Holat C — ❌ XATO
startServer({ host: "localhost", port: 3000, debug: true });
// Funksiya argumenti — object literal (fresh)
// Excess check ishlaydi

// Holat D — ✅ XATO YO'Q
const serverOptions = { host: "localhost", port: 3000, debug: true };
startServerFromVar(serverOptions);
// Variable orqali — fresh emas, excess check yo'q
```

**Qoida:** Excess property checking faqat **fresh object literal**'da (to'g'ridan-to'g'ri yozilgan, assign qilinmagan) ishlaydi. Variable, funksiya natijasi, yoki boshqa expression orqali berilganda — ishlamaydi.

</details>

---

### Mashq 3: Recursive Interface (Qiyin)

**Savol:** `Comment` interface'ini yozing — har comment'da ichki `replies` (javoblar) bo'lishi mumkin. `countAllComments` funksiyasini yozing — barcha comment'lar sonini (nested, deep) hisoblaydi.

<details>
<summary>Javob</summary>

```typescript
interface Comment {
  id: number;
  author: string;
  text: string;
  replies: Comment[]; // ← recursive
}

function countAllComments(comments: Comment[]): number {
  let count = comments.length;
  for (const comment of comments) {
    count += countAllComments(comment.replies);
  }
  return count;
}

// Yoki reduce bilan
function countAll(comments: Comment[]): number {
  return comments.reduce(
    (total, comment) => total + 1 + countAll(comment.replies),
    0
  );
}

// Test
const thread: Comment[] = [
  {
    id: 1,
    author: "Ali",
    text: "Great article!",
    replies: [
      {
        id: 2,
        author: "Vali",
        text: "Thanks!",
        replies: [
          {
            id: 3,
            author: "Ali",
            text: "You're welcome",
            replies: [],
          },
        ],
      },
    ],
  },
  {
    id: 4,
    author: "Soli",
    text: "Nice post",
    replies: [],
  },
];

console.log(countAllComments(thread)); // 4
// 1 (Ali) + 1 (Vali) + 1 (Ali) + 1 (Soli) = 4
```

**Tushuntirish:** Recursive interface o'ziga o'zi havola qiladi — compiler uni lazy resolve qiladi (darhol expand qilmaydi). Recursive funksiya esa har comment uchun o'zini qayta chaqiradi va ichki nesting'larni hisoblaydi. Bu pattern tree, linked list, JSON kabi nested strukturalar uchun universal.

</details>

---

## Xulosa

Bu bo'limda TypeScript'ning object va interface tip tizimi bilan tanishdik:

- **Object Type Annotations** — `{ key: Type }` bilan shape belgilash, structural typing asosi
- **Optional Properties** — `key?: Type` property yo'q bo'lishi mumkin, `| undefined`'dan farqi
- **Readonly Properties** — `readonly` shallow himoya, nested mutation mumkin
- **Index Signatures** — `[key: string]: V` dinamik key'lar, `noUncheckedIndexedAccess` bilan xavfsiz
- **`Record<K, V>`** — cheklangan key'lar bilan mapped type, index signature'dan afzal cheklangan holatda
- **Interface Declaration** — object shape e'lon qilish, method shorthand va property function farqi
- **Interface Extending** — `extends` bilan property meros, subtype override
- **Multiple Extending** — bir nechta parent'dan, mixin pattern, conflict rules
- **Declaration Merging** — faqat interface'da, library augmentation va global augmentation uchun
- **Interface vs Type Alias** — object shape → interface, union/tuple/mapped/conditional → type alias
- **Excess Property Checking** — faqat fresh literal'da, variable orqali bypass
- **Recursive Interfaces** — self-reference, daraxt va JSON strukturalar, lazy resolution
- **PropertyKey** — `string | number | symbol`, valid object key tiplari, `keyof any` bilan ekvivalent
- **Edge Cases** — shallow readonly, excess check bypass, interface merging gotcha, `noUncheckedIndexedAccess`, generic default

Keyingi bo'limda TypeScript'ning **union va intersection type'lari** — `|` va `&` operatorlari, discriminated unions, type guards, exhaustive checking, `assertNever` pattern va narrowing bilan tanishamiz.

---

**Keyingi bo'lim:** [05-unions-intersections.md](05-unions-intersections.md) — Type Aliases va Union/Intersection: union types, intersection types, discriminated unions, type guards (`typeof`, `instanceof`, `in`), exhaustive checking, `assertNever` pattern, control flow analysis, union vs overload.
