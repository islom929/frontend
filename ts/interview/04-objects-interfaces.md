# Interview: Object Types va Interfaces

> Object types, interface, type alias, excess property checking, declaration merging, readonly, index signatures bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Interface va Type Alias farqi nima? Qachon qaysi birini ishlatish kerak?

<details>
<summary>Javob</summary>

Ikkalasi ham object shape ni tavsiflaydi, lekin muhim farqlari bor:

| Xususiyat | Interface | Type Alias |
|-----------|-----------|------------|
| Object shape | ✅ | ✅ |
| Extend (meros) | `extends` | `&` (intersection) |
| Declaration merging | ✅ | ❌ |
| Union types | ❌ | ✅ |
| Mapped/Conditional types | ❌ | ✅ |
| Primitive alias | ❌ | ✅ |

```typescript
// Interface — object shape + extend + merge
interface User { name: string }
interface User { age: number } // ✅ merge bo'ladi
interface Admin extends User { role: string }

// Type alias — union, tuple, mapped, conditional
type ID = string | number;
type Pair = [string, number];
```

**Qoida:** Object shape uchun → interface. Union, tuple, mapped, conditional uchun → type alias.

</details>

### 2. Declaration merging nima? Qachon foydali?

<details>
<summary>Javob</summary>

Declaration merging — bir xil nomli ikki interface avtomatik birlashadi. Faqat interface da ishlaydi, type alias da **mumkin emas**.

```typescript
interface Config { host: string; }
interface Config { port: number; }
// Natija: { host: string; port: number }

type Config2 = { host: string };
type Config2 = { port: number }; // ❌ Duplicate identifier
```

Foydali holatlar:

1. **Library augmentation** — mavjud library tiplarini kengaytirish:

```typescript
declare module "express" {
  interface Request {
    userId?: string;
  }
}
```

2. **Global type kengaytirish:**

```typescript
declare global {
  interface Window {
    __APP_VERSION__: string;
  }
}
```

</details>

### 3. Excess property checking nima? Qanday ishlaydi?

<details>
<summary>Javob</summary>

Excess property checking — TypeScript **object literal** da kutilgan tipda **mavjud bo'lmagan** property bor bo'lsa xato beradi. Bu faqat literal da ishlaydi — variable orqali berilganda ishlamaydi.

```typescript
interface User { name: string; age: number }

// ❌ Literal — excess property check ishlaydi
const user: User = { name: "Ali", age: 25, email: "x" };
// ❌ 'email' does not exist in type 'User'

// ✅ Variable orqali — check ISHLAMAYDI
const data = { name: "Ali", age: 25, email: "x" };
const user2: User = data; // ✅ Structural typing — name, age bor, yetarli
```

Nima uchun bu farq bor: Object literal da qo'shimcha property ko'pincha **typo** yoki **xato** — TS buni ushlaydi. Variable orqali berilganda — structural typing ishlaydi, qo'shimcha property lar "bonus".

Bypass usullari: (1) variable ga oldin assign, (2) type assertion `as User`, (3) index signature `[key: string]: unknown`.

</details>

### 4. `readonly` property haqida nima bilasiz? Shallow ekanini tushuntiring.

<details>
<summary>Javob</summary>

`readonly` property faqat o'qish uchun — qayta assign mumkin emas. Lekin **shallow** (yuzaki) — ichki object larni cheklamaydi.

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

// config.settings = { ... }; // ❌ readonly — qayta assign mumkin emas
config.settings.theme = "light"; // ✅ Ichki property O'ZGARADI!
```

`readonly` faqat birinchi daraja. Deep readonly uchun custom utility type kerak — batafsil [interview/16 — Custom Utility Types](16-custom-utility-types.md) da.

Muhim: `readonly` faqat **compile-time** — JS ga compile bo'lganda o'chiriladi. Runtime himoya uchun `Object.freeze()` kerak.

</details>

### 5. Index signature nima? `Record` bilan farqi?

<details>
<summary>Javob</summary>

Index signature — object da **dinamik** (oldindan noma'lum) kalitlar uchun tip belgilash:

```typescript
interface StringMap {
  [key: string]: number;
}

let prices: StringMap = {};
prices["apple"] = 100; // ✅ har qanday string key
```

`Record<K, V>` o'xshash — lekin cheklangan key lar bilan ham ishlaydi:

```typescript
type Fruit = "apple" | "banana" | "cherry";
type FruitPrices = Record<Fruit, number>;
// Barcha 3 key MAJBURIY

// Index signature bilan buni qilish mumkin emas
interface FruitMap {
  [key: string]: number; // har qanday string — cheklanmagan
}
```

| Xususiyat | Index Signature | Record |
|-----------|----------------|--------|
| Cheklangan key lar | ❌ | ✅ Union bilan |
| Explicit property bilan | ✅ | ❌ |
| Method qo'shish | ✅ | ❌ |

</details>

### 6. Optional property (`?`) va `| undefined` farqi nima?

<details>
<summary>Javob</summary>

```typescript
interface A { x?: string }          // property bo'lmasligi mumkin
interface B { x: string | undefined } // property BOR, qiymati undefined

const a: A = {};              // ✅ x yo'q
const b: B = {};              // ❌ x kerak
const b2: B = { x: undefined }; // ✅

console.log("x" in a); // false — property yo'q
console.log("x" in b2); // true — property bor
Object.keys(a); // []
Object.keys(b2); // ["x"]
```

`exactOptionalPropertyTypes: true` (tsconfig) bilan farq yanada kuchli:

```typescript
interface C { x?: string }
const c: C = { x: undefined };
// ❌ Type 'undefined' is not assignable to type 'string'
// Optional property ga undefined BERIB BO'LMAYDI
// Faqat property ni tushirib qoldirish mumkin
```

</details>

### 7. Method shorthand va property function farqi nima? `strictFunctionTypes` bilan qanday bog'liq?

<details>
<summary>Javob</summary>

Interface da method ikki xil yoziladi — va ular `strictFunctionTypes` bilan farqli behavior ko'rsatadi:

```typescript
interface Handler {
  // Method shorthand — bivariant (kam qat'iy)
  handle(event: MouseEvent): void;

  // Property function — contravariant (qat'iy, xavfsizroq)
  process: (event: MouseEvent) => void;
}
```

Farq `strictFunctionTypes: true` da ko'rinadi:

```typescript
interface A {
  method(x: string | number): void;    // method shorthand
  func: (x: string | number) => void;  // property function
}

interface B extends A {
  method(x: string): void;
  // ✅ Method shorthand — bivariant, tor parametr qabul qilinadi
  // Bu XAVFLI — lekin TS ruxsat beradi (tarixiy sabab)

  func: (x: string) => void;
  // ❌ Property function — contravariant
  // TS to'xtatadi — xavfsiz
}
```

**Nima uchun:** Method shorthand bivariant qoldirilgan — DOM event handler lari uchun kerak (tarixiy sabab, `lib.dom.d.ts` compatibility). Property function contravariant — type-safe.

**Maslahat:** Yangi kodda **property function** sintaksisi afzal — xavfsizroq. ESLint `@typescript-eslint/method-signature-style` qoidasi buni majbur qiladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. readonly, optional, excess — output savoli (Daraja: Junior+)

**Savol:** Quyidagi kodning har bir satrida nima sodir bo'ladi? Qaysilari ✅, qaysilari ❌?

```typescript
interface Config {
  readonly host: string;
  port: number;
  debug?: boolean;
}

const config: Config = { host: "localhost", port: 3000 };

config.port = 4000;
config.host = "example.com";
config.debug = true;

const config2: Config = { host: "localhost", port: 3000, ssl: true };

const data = { host: "localhost", port: 3000, ssl: true };
const config3: Config = data;
```

<details>
<summary>Yechim</summary>

```typescript
const config: Config = { host: "localhost", port: 3000 };
// ✅ debug optional — yo'q bo'lsa ham OK

config.port = 4000;
// ✅ port readonly emas — o'zgartirish mumkin

config.host = "example.com";
// ❌ Cannot assign to 'host' because it is a read-only property

config.debug = true;
// ✅ optional property ga qiymat assign qilish mumkin

const config2: Config = { host: "localhost", port: 3000, ssl: true };
// ❌ Object literal may only specify known properties — 'ssl' yo'q

const data = { host: "localhost", port: 3000, ssl: true };
const config3: Config = data;
// ✅ Variable orqali — excess property check ISHLAMAYDI
// Structural typing: host ✅, port ✅ — yetarli
```

</details>

### 2. Interface extends — xatoni toping (Daraja: Middle)

**Savol:** Bu kodda compile-time xato bor. Xatoni toping va ikki xil usulda tuzating:

```typescript
interface Animal {
  name: string;
  sound: string;
}

interface Dog extends Animal {
  breed: string;
  sound: number;
}
```

<details>
<summary>Yechim</summary>

**Xato:** `extends` da child property tipi parent ning **subtype** i bo'lishi kerak. `number` ≠ `string` — mos kelmaydi.

```typescript
// ❌ Interface 'Dog' incorrectly extends interface 'Animal'
// Types of property 'sound' are incompatible

// ✅ Yechim 1 — literal subtype
interface Dog extends Animal {
  breed: string;
  sound: "bark" | "woof"; // ✅ string literal ⊂ string
}

// ✅ Yechim 2 — parent ni o'zgartirish
interface Animal {
  name: string;
  sound: string | number;
}
interface Dog extends Animal {
  breed: string;
  sound: number; // ✅ number ⊂ (string | number)
}
```

**Muhim farq:** Intersection `&` bilan xato bermaydi, lekin `never` bo'ladi:

```typescript
type AnimalType = { sound: string };
type DogType = AnimalType & { sound: number };
// DogType.sound = string & number = never — hech narsa assign mumkin emas
```

</details>

### 3. Interface extends vs Intersection — conflict farqi (Daraja: Middle+)

**Savol:** `extends` va `&` (intersection) bilan bir xil tipni birlashtiring. Conflict qayerda ko'rinadi, qayerda yashirinadi?

```typescript
// Setup
interface Base {
  id: number;
  name: string;
  meta: { version: number };
}

// 1. extends bilan — nima bo'ladi?
interface ExtendedA extends Base {
  name: number; // ?
}

// 2. intersection bilan — nima bo'ladi?
type IntersectedB = Base & {
  name: number; // ?
};

// 3. Nested conflict — extends bilan
interface ExtendedC extends Base {
  meta: { version: string }; // ?
}

// 4. Nested conflict — intersection bilan
type IntersectedD = Base & {
  meta: { version: string }; // ?
};
```

<details>
<summary>Yechim</summary>

```typescript
// 1. ❌ COMPILE ERROR — extends tezda xato beradi
interface ExtendedA extends Base {
  name: number; // ❌ Type 'number' is not assignable to type 'string'
}

// 2. ⚠️ XATO BERMAYDI, lekin name: never
type IntersectedB = Base & { name: number };
// name: string & number = never
// const b: IntersectedB = { id: 1, name: ???, meta: { version: 1 } };
// Hech qanday qiymat assign mumkin emas

// 3. ❌ COMPILE ERROR — nested da ham extends xato beradi
interface ExtendedC extends Base {
  meta: { version: string }; // ❌ Types of property 'meta' are incompatible
}

// 4. ⚠️ XATO BERMAYDI, lekin meta.version: never
type IntersectedD = Base & { meta: { version: string } };
// meta: { version: number } & { version: string }
// meta.version: number & string = never
```

**Xulosa:**

- **`extends`** — conflict darhol ko'rinadi, aniq xato xabari. **Afzal** variant.
- **`&` intersection** — conflict yashirin, `never` ga aylanadi. Xato faqat ishlatishda chiqadi.
- Performance: `extends` cached, `&` har safar qayta hisoblanadi (katta tip larda sekinroq).

</details>

### 4. Recursive interface — fayl tizimi (Daraja: Middle+)

**Savol:** `FileNode` recursive interface va `countFiles` funksiyasini yozing:

```typescript
// FileNode interface yozing:
// - name: string
// - type: "file" | "directory"
// - size: number (faqat file da)
// - children: FileNode[] (faqat directory da)

// countFiles funksiyasini yozing:
// - faqat "file" type li node larni sanaydi
// - directory ni o'zini sanmaydi

// Test:
const root: FileNode = {
  name: "src", type: "directory", children: [
    { name: "index.ts", type: "file", size: 1024 },
    { name: "utils", type: "directory", children: [
      { name: "helper.ts", type: "file", size: 512 },
      { name: "math.ts", type: "file", size: 256 },
    ]},
  ],
};

countFiles(root); // 3
```

<details>
<summary>Yechim</summary>

```typescript
// Discriminated union bilan — eng type-safe
type FileNode =
  | { name: string; type: "file"; size: number }
  | { name: string; type: "directory"; children: FileNode[] };

function countFiles(node: FileNode): number {
  if (node.type === "file") {
    return 1;
  }
  return node.children.reduce((sum, child) => sum + countFiles(child), 0);
}
```

**Tushuntirish:**

- **Discriminated union** — `type` field bo'yicha TS avtomatik narrow qiladi
- `node.type === "file"` bo'lganda — TS `size` mavjudligini biladi, `children` yo'q
- `node.type === "directory"` bo'lganda — TS `children` mavjudligini biladi, `size` yo'q
- Oddiy interface bilan `size?` va `children?` bo'lardi — kamroq type-safe

**Alternativ — oddiy interface:**

```typescript
interface FileNode {
  name: string;
  type: "file" | "directory";
  size?: number;
  children?: FileNode[];
}

function countFiles(node: FileNode): number {
  if (node.type === "file") return 1;
  if (!node.children) return 0;
  return node.children.reduce((sum, child) => sum + countFiles(child), 0);
}
// Ishlaydi, lekin null check lar kerak — kamroq xavfsiz
```

</details>

### 5. Type-safe dictionary yozing (Daraja: Senior)

**Savol:** Index signature va generics ishlatib, type-safe `TypedMap` class yozing. `Map` ga o'xshash, lekin key lar union type bilan cheklangan:

```typescript
// TypedMap ni implement qiling:
// const config = new TypedMap<"host" | "port" | "debug">()
// config.set("host", "localhost") → ✅
// config.set("invalid", "x")     → ❌ compile error
// config.get("host")             → string | undefined
// config.has("port")             → boolean
// config.keys()                  → ("host" | "port" | "debug")[]
```

<details>
<summary>Yechim</summary>

```typescript
class TypedMap<K extends string> {
  private data: Partial<Record<K, string>> = {};

  set(key: K, value: string): void {
    this.data[key] = value;
  }

  get(key: K): string | undefined {
    return this.data[key];
  }

  has(key: K): boolean {
    return this.data[key] !== undefined;
  }

  keys(): K[] {
    return Object.keys(this.data) as K[];
  }

  delete(key: K): void {
    delete this.data[key];
  }

  entries(): [K, string][] {
    return Object.entries(this.data) as [K, string][];
  }
}

// Ishlatish:
const config = new TypedMap<"host" | "port" | "debug">();
config.set("host", "localhost"); // ✅
config.set("port", "3000");      // ✅
// config.set("invalid", "x");   // ❌ Argument not assignable

const host = config.get("host"); // string | undefined
config.has("port");               // boolean
config.keys();                    // ("host" | "port" | "debug")[]
```

**Tushuntirish:**

- `K extends string` — key tipi string subtype bo'lishi kerak
- `Partial<Record<K, string>>` — barcha key lar optional, qiymati string
- `Object.keys()` `string[]` qaytaradi — `as K[]` kerak (TS cheklovi)
- Bu pattern config, feature flags, translation dictionary larda ishlatiladi

</details>

---

## Xulosa

- Interface — object shape, `extends`, declaration merging. Type alias — union, tuple, mapped/conditional
- Declaration merging faqat interface da — library augmentation uchun muhim
- Excess property checking faqat object literal da — variable orqali bypass bo'ladi
- `readonly` — shallow, faqat compile-time. Deep readonly uchun custom utility kerak
- `?` (optional) va `| undefined` — farq bor: optional property bo'lmasligi mumkin
- Method shorthand bivariant, property function contravariant — yangi kodda property function afzal
- Interface `extends` — conflict darhol ko'rinadi. Intersection `&` — yashirin `never`

[Asosiy bo'limga qaytish →](../04-objects-interfaces.md)
