# Interview: TypeScript Asoslari

> TypeScript nima, compiler pipeline, type erasure, structural typing, strict mode va asosiy tushunchalar bo'yicha interview savollari.

---

## Nazariy savollar

### 1. TypeScript nima va JavaScript dan qanday farq qiladi?

<details>
<summary>Javob</summary>

TypeScript — Microsoft tomonidan yaratilgan, JavaScript ustiga qurilgan statik tiplangan til. Har qanday to'g'ri JavaScript kodi TypeScript kodi ham hisoblanadi (typed superset).

Asosiy farqlar:

| Xususiyat | JavaScript | TypeScript |
|-----------|-----------|------------|
| Type checking | Runtime (dynamic) | Compile-time (static) |
| Xato aniqlash | Faqat runtime da crash | Compile-time da xato |
| IDE support | Cheklangan | To'liq autocomplete, refactoring |
| Compile step | Kerak emas | Kerak (tsc, esbuild, SWC) |
| Runtime da | O'zi ishlaydi | JS ga compile bo'ladi (type erasure) |

TypeScript ning asosiy qiymati ikki narsa: (1) xatolarni runtime dan oldin ushlash, (2) IDE tooling — autocomplete, go-to-definition, rename symbol. Katta loyihalarda developer productivity ni keskin oshiradi.

</details>

### 2. TypeScript Compiler (tsc) qanday ishlaydi? Pipeline bosqichlarini tushuntiring.

<details>
<summary>Javob</summary>

tsc TypeScript source code ni JavaScript ga aylantiradi. Bu jarayon 5 bosqichdan iborat:

1. **Scanner (Lexer)** — source code matnini tokenlar ga ajratadi (`let`, `age`, `:`, `number`, `=`, `25`, `;`)
2. **Parser** — tokenlar oqimidan AST (Abstract Syntax Tree) qurilmasini hosil qiladi
3. **Binder** — AST bo'ylab yurib, har bir declaration uchun Symbol yaratadi va scope larni aniqlaydi
4. **Checker** — tsc ning eng katta qismi. Barcha type tekshiruvlarni amalga oshiradi: inference, assignability, generic resolution, narrowing
5. **Emitter** — AST dan JavaScript, `.d.ts`, va source map fayllarni hosil qiladi. Type annotation lar o'chiriladi

```
Source (.ts) → Scanner → Parser → AST → Binder → Checker → Emitter → .js + .d.ts + .map
```

Checker lazy evaluation ishlatadi — barcha type larni oldindan hisoblamasdan, kerak bo'lganda hisoblaydi. Bu katta loyihalarda performance uchun muhim.

</details>

### 3. Type erasure nima? Nima uchun muhim?

<details>
<summary>Javob</summary>

Type erasure — compile jarayonida barcha type ma'lumotlari to'liq o'chiriladi. Interface lar, type alias lar, generic lar, type annotation lar — bularning hech biri JavaScript ga tushmaydi. Runtime da TypeScript haqida hech qanday iz qolmaydi.

Bu shunday dizayn qilingan chunki TypeScript mavjud JavaScript runtime larni o'zgartirmaydi — brauzerlar va Node.js faqat JavaScript ni bajaradi.

```typescript
// TypeScript:
interface User { name: string; age: number; }
function greet(user: User): string { return `Hi ${user.name}`; }

// Compiled JavaScript — interface butunlay yo'q:
function greet(user) { return `Hi ${user.name}`; }
```

Type erasure ning muhim oqibati — runtime da `instanceof` bilan interface tekshirish **mumkin emas**. Interface runtime da mavjud emas. Runtime type tekshirish kerak bo'lsa — `typeof`, `in` operator, yoki user-defined type guard ishlatiladi.

</details>

### 4. Structural typing va Nominal typing farqi nima?

<details>
<summary>Javob</summary>

**Nominal typing** (Java, C#) — ikki tip mos kelishi uchun ular **bir xil nom** ga yoki aniq inheritance bog'lanishiga ega bo'lishi kerak. Tuzilma bir xil bo'lsa ham, nom farq qilsa — mos kelmaydi.

**Structural typing** (TypeScript) — ikki tip mos kelishi uchun ularning **tuzilmasi (shape)** mos bo'lishi yetarli. Nomlar ahamiyatsiz.

TypeScript structural typing ishlatadi — bu JavaScript ning tabiatiga mos. JavaScript da object ning qaysi class dan yaratilgani emas, qanday property lari borligi muhim (duck typing).

```typescript
interface HasLength { length: number; }

function logLength(item: HasLength) {
  console.log(item.length);
}

logLength("hello");         // ✅ string da length bor
logLength([1, 2, 3]);       // ✅ array da length bor
logLength({ length: 42 });  // ✅ object da length bor
```

Kamchiligi — semantik jihatdan farqli tiplar aralashib ketishi mumkin:

```typescript
type Meters = number;
type Seconds = number;

function speed(distance: Meters, time: Seconds) { return distance / time; }

const time: Seconds = 10;
const distance: Meters = 100;
speed(time, distance); // ✅ Xato bermaydi! Parametrlar almashgan
// Yechim: Branded types (23-type-safe-patterns.md da)
```

</details>

### 5. `strict: true` nima qiladi? Qaysi flaglarni yoqadi?

<details>
<summary>Javob</summary>

`strict: true` — tsconfig.json da barcha strict type-checking flaglarni birdan yoqadigan meta-flag. Yangi loyihada doim yoqish kerak.

Yoqadigan flaglar:

| Flag | Nima qiladi |
|------|-------------|
| `strictNullChecks` | `null`/`undefined` har type dan ajratiladi |
| `noImplicitAny` | Type aniqlanmasa `any` deb qabul qilmaslik |
| `strictFunctionTypes` | Funksiya parametrlarini contravariant tekshirish |
| `strictBindCallApply` | `bind`, `call`, `apply` parametrlarini tekshirish |
| `strictPropertyInitialization` | Class property lari constructor da init bo'lishi shart |
| `noImplicitThis` | `this` type noma'lum bo'lsa xato |
| `alwaysStrict` | Har faylga `"use strict"` qo'shish |
| `useUnknownInCatchVariables` | `catch(e)` da `e` type i `unknown` |

`strict: true` yozib, keyin alohida flagni `false` qilish mumkin — masalan migration paytida `"strictPropertyInitialization": false`.

</details>

### 6. `any` va `unknown` farqi nima? Qachon qaysi birini ishlatish kerak?

<details>
<summary>Javob</summary>

`any` — type checking ni **to'liq o'chiradi**. Har qanday operatsiya qilish mumkin, TypeScript hech narsa demaydi.

`unknown` — "type noma'lum, avval tekshir". Hech qanday operatsiya qilish mumkin emas — faqat narrowing dan keyin.

| Xususiyat | `any` | `unknown` |
|-----------|-------|-----------|
| Boshqa type ga assign | ✅ | ❌ (faqat `any`/`unknown` ga) |
| Property/method call | ✅ (tekshirmaydi) | ❌ (narrowing kerak) |
| Xavfsizlik | ❌ | ✅ |
| Ishlatish | JS → TS migration | API boundary, catch, tashqi data |

```typescript
// any — xavfli
function processAny(data: any) {
  data.user.profile.getName(); // Compile ok, runtime crash xavfi
}

// unknown — xavfsiz
function processUnknown(data: unknown) {
  if (typeof data === "object" && data !== null && "name" in data) {
    console.log(data.name); // ✅ narrowing dan keyin xavfsiz
  }
}
```

**Qoida:** `any` faqat migration da vaqtincha. Yangi kodda doim `unknown` yoki aniq type.

</details>

### 7. `.ts`, `.tsx`, `.d.ts`, `.mts`, `.cts` farqi nima?

<details>
<summary>Javob</summary>

| Kengaytma | Maqsad | Compile natijasi |
|-----------|--------|-----------------|
| `.ts` | TypeScript source fayl | `.js` |
| `.tsx` | TypeScript + JSX (React) | `.js` |
| `.d.ts` | Declaration fayl (faqat type, kod yo'q) | Compile qilinmaydi |
| `.mts` | TypeScript ES Module | `.mjs` |
| `.cts` | TypeScript CommonJS | `.cjs` |

`.tsx` da muhim cheklov — angle bracket assertion ishlamaydi:

```typescript
// .ts faylda:
const value = <string>someValue;    // ✅ angle bracket assertion

// .tsx faylda:
// const value = <string>someValue; // ❌ JSX deb tushunadi
const value = someValue as string;  // ✅ faqat as ishlaydi
```

`.d.ts` — faqat type information, implementation yo'q. JS kutubxonalar uchun type beradi (`@types/node`, `@types/react`). `.mts`/`.cts` — Node.js da ESM/CJS formatini aniq belgilash uchun, `moduleResolution: "Node16"` bilan ishlatiladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Structural typing — output savoli (Daraja: Junior+)

**Savol:** Quyidagi kodning output ini ayting va nima uchun ekanini tushuntiring:

```typescript
interface Printable {
  print(): string;
}

class Document {
  constructor(public title: string) {}
  print() { return `Doc: ${this.title}`; }
}

class Logger {
  title = "SystemLog";
  print() { return `Log: ${this.title}`; }
  clear() { console.log("cleared"); }
}

function output(item: Printable): string {
  return item.print();
}

const doc = new Document("Report");
const logger = new Logger();
const plain = { title: "Plain", print() { return `Plain: ${this.title}`; } };

console.log(output(doc));
console.log(output(logger));
console.log(output(plain));
```

<details>
<summary>Yechim</summary>

```
Doc: Report
Log: SystemLog
Plain: Plain
```

Uchalasi ham ishlaydi. TypeScript **structural typing** ishlatadi — `Document`, `Logger`, va oddiy object hech biri `Printable` ni aniq `implements` qilmagan, lekin ularning shape i mos keladi (`print(): string` method bor).

- `Logger` da qo'shimcha `clear()` va `title` bor — structural typing da qo'shimcha member lar to'sqinlik qilmaydi
- `plain` variable ga assign qilinib keyin berilgan, shuning uchun **excess property checking** ishlamaydi
- Agar to'g'ridan-to'g'ri `output({ title: "X", print() { ... } })` deb yozsak — `title` uchun excess property error beradi (inline object literal da TS qo'shimcha property larni tekshiradi)

</details>

### 2. Interface va instanceof — xatoni toping (Daraja: Middle)

**Savol:** Bu kodda compile-time xato bor. Xatoni toping, sababini tushuntiring va to'g'ri variantini yozing:

```typescript
interface Animal {
  name: string;
  sound(): string;
}

function makeSound(animal: Animal) {
  if (animal instanceof Animal) {
    return animal.sound();
  }
  return "unknown";
}
```

<details>
<summary>Yechim</summary>

**Xato:** `instanceof Animal` — compile-time xato. `instanceof` faqat class (constructor function) bilan ishlaydi. `Animal` bu interface — type erasure tufayli runtime da mavjud emas.

```typescript
// ❌ 'Animal' only refers to a type, but is being used as a value here
if (animal instanceof Animal) { ... }

// ✅ Variant 1: property tekshirish
function makeSound(animal: unknown): string {
  if (
    typeof animal === "object" && animal !== null &&
    "name" in animal && "sound" in animal &&
    typeof (animal as Record<string, unknown>).sound === "function"
  ) {
    return (animal as Animal).sound();
  }
  return "unknown";
}

// ✅ Variant 2: class ishlatish (class runtime da mavjud)
class Dog implements Animal {
  constructor(public name: string) {}
  sound() { return "Woof!"; }
}

const dog = new Dog("Rex");
console.log(dog instanceof Dog); // ✅ true — class bilan ishlaydi
```

**Sabab:** Interface va type — compile-time tushunchalar. JS ga compile bo'lganda butunlay o'chiriladi (type erasure). `instanceof` runtime operator — faqat class bilan ishlaydi.

</details>

### 3. User-defined type guard yozing (Daraja: Middle)

**Savol:** `isUser` type guard funksiyasini yozing. `unknown` tipdan `User` tipga xavfsiz tekshirish qilsin:

```typescript
interface User {
  name: string;
  email: string;
  age: number;
}

function isUser(value: unknown): /* return type? */ {
  // Implement qiling
}

// Ishlatish:
const data: unknown = JSON.parse('{"name":"Ali","email":"ali@test.com","age":25}');

if (isUser(data)) {
  console.log(data.name);  // TS buni xavfsiz deb bilishi kerak
  console.log(data.email);
}
```

<details>
<summary>Yechim</summary>

```typescript
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    typeof (value as Record<string, unknown>).name === "string" &&
    "email" in value &&
    typeof (value as Record<string, unknown>).email === "string" &&
    "age" in value &&
    typeof (value as Record<string, unknown>).age === "number"
  );
}
```

**Tushuntirish:**

- `value is User` — **type predicate**. TypeScript ga aytadi: agar funksiya `true` qaytarsa, `value` ni `User` tipida deb hisoblash kerak
- `typeof value === "object" && value !== null` — avval object ekanini tekshirish (`typeof null === "object"` — JS legacy bug)
- `"name" in value` — property mavjudligini tekshirish
- `typeof ... === "string"` — property tipini tekshirish
- `Record<string, unknown>` — `as any` dan xavfsizroq intermediate assertion

</details>

### 4. `unknown` tipdan xavfsiz data extraction (Daraja: Middle+)

**Savol:** API dan kelgan `unknown` response dan xavfsiz tarzda `users` ro'yxatini chiqaring:

```typescript
interface ApiUser {
  id: number;
  name: string;
  active: boolean;
}

function extractUsers(response: unknown): ApiUser[] {
  // Implement qiling:
  // - unknown dan xavfsiz tarzda ApiUser[] qaytaring
  // - Agar format noto'g'ri bo'lsa bo'sh array qaytaring
  // - Har bir element individual tekshirilsin
}
```

<details>
<summary>Yechim</summary>

```typescript
function extractUsers(response: unknown): ApiUser[] {
  if (
    typeof response !== "object" || response === null ||
    !("data" in response)
  ) {
    return [];
  }

  const data = (response as Record<string, unknown>).data;

  if (typeof data !== "object" || data === null || !("users" in data)) {
    return [];
  }

  const users = (data as Record<string, unknown>).users;

  if (!Array.isArray(users)) {
    return [];
  }

  return users.filter((u): u is ApiUser =>
    typeof u === "object" && u !== null &&
    "id" in u && typeof (u as Record<string, unknown>).id === "number" &&
    "name" in u && typeof (u as Record<string, unknown>).name === "string" &&
    "active" in u && typeof (u as Record<string, unknown>).active === "boolean"
  );
}
```

**Tushuntirish:**

- Har qadamda `unknown` dan bitta daraja chuqurroqqa narrowing qilinadi
- `Record<string, unknown>` — `as any` dan xavfsizroq intermediate assertion
- `.filter` ichida type predicate — array element larini individual tekshirish
- Agar biror qadamda format mos kelmasa — darhol `[]` qaytaradi (defensive)
- Real loyihalarda `zod` yoki `valibot` kabi validation kutubxona ishlatish tavsiya etiladi

</details>

### 5. Enum compile output ini yozing (Daraja: Middle+)

**Savol:** Quyidagi TypeScript kodi JavaScript ga qanday compile bo'ladi? Ikki variantning farqini yozing:

```typescript
// Variant A: oddiy enum
enum Color {
  Red,
  Green,
  Blue
}
const c1 = Color.Green;
const name1 = Color[1];

// Variant B: const enum
const enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT"
}
const d1 = Direction.Up;
```

<details>
<summary>Yechim</summary>

**Variant A — oddiy enum:**

```javascript
var Color;
(function (Color) {
    Color[Color["Red"] = 0] = "Red";
    Color[Color["Green"] = 1] = "Green";
    Color[Color["Blue"] = 2] = "Blue";
})(Color || (Color = {}));
const c1 = Color.Green;   // 1
const name1 = Color[1];   // "Green" (reverse mapping)
```

**Variant B — const enum:**

```javascript
// enum butunlay yo'q — qiymat inline bo'ldi
const d1 = "UP";
```

| Xususiyat | `enum` | `const enum` |
|-----------|--------|-------------|
| Runtime object | ✅ IIFE yaratadi | ❌ Yo'q (inline) |
| Reverse mapping | ✅ (numeric uchun) | ❌ |
| Bundle size | Kattaroq | Kichikroq |
| Dynamic access | ✅ `Color[value]` | ❌ Mumkin emas |

**Farq:** Oddiy enum runtime da object yaratadi (IIFE + reverse mapping). `const enum` compile vaqtida to'liq inline bo'ladi — runtime da hech narsa qolmaydi. `const enum` `isolatedModules` rejimda `.d.ts` dan re-export qilib bo'lmaydi.

</details>

---

## Xulosa

- TypeScript — JavaScript ning typed superset i. Compile-time type safety va IDE tooling beradi
- tsc pipeline: Scanner → Parser → AST → Binder → Checker → Emitter
- Type erasure — barcha type ma'lumotlari compile vaqtida o'chiriladi, runtime da TypeScript yo'q
- Structural typing — shape mos bo'lsa yetarli, nom muhim emas
- `strict: true` — doim yoqish kerak, 8 ta strict flagni birdan yoqadi
- `unknown` > `any` — yangi kodda doim `unknown` ishlatish

[Asosiy bo'limga qaytish →](../01-ts-intro.md)
