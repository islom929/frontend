# Interview: Type Narrowing

> Type narrowing, control flow analysis, type guards, assertion functions, satisfies operator bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Type narrowing nima? Nima uchun kerak?

<details>
<summary>Javob</summary>

Type narrowing — union type ni aniqroq (specific) type ga toraytirish jarayoni. TS compiler runtime tekshiruvlarni (`typeof`, `instanceof`, `in`, equality) tahlil qilib, har bir code branch da type ni avtomatik yangilaydi.

```typescript
function format(value: string | number): string {
  // value: string | number
  // value.toUpperCase(); // ❌ number da yo'q

  if (typeof value === "string") {
    return value.toUpperCase(); // ✅ value: string
  }
  return value.toFixed(2); // ✅ value: number
}
```

Nima uchun kerak: union type da faqat **barcha member larga umumiy** operatsiyalar ishlaydi. Specific property yoki metod uchun avval narrowing qilish shart.

</details>

### 2. Control flow analysis qanday ishlaydi?

<details>
<summary>Javob</summary>

TS compiler kodni **yuqoridan pastga** tahlil qiladi va har bir nuqtada o'zgaruvchining aniq type ini aniqlaydi. `if`, `switch`, `return`, `throw` kabi statement lar type ni o'zgartiradi.

```typescript
function process(x: string | number | null): void {
  // x: string | number | null

  if (x === null) {
    return; // Bu branch dan chiqildi
  }
  // x: string | number — null olib tashlandi (return tufayli)

  if (typeof x === "string") {
    x.toUpperCase(); // x: string
    return;
  }
  // x: number — string ham olib tashlandi

  x.toFixed(2); // x: number
}
```

Har bir `return`, `throw`, `break` dan keyin shu branch dagi type "olib tashlanadi". Bu "flow-sensitive typing" — type statik emas, kodni oqish bo'yicha o'zgaradi. TS ichki control flow graph qurib, har bir node da type holatini saqlaydi.

</details>

### 3. `typeof` qaysi qiymatlarni qaytaradi? TS bularni qanday narrowing qiladi?

<details>
<summary>Javob</summary>

`typeof` 8 ta mumkin bo'lgan qiymatni qaytaradi:

| `typeof` natijasi | Qaysi qiymatlar | TS Narrowing |
|---|---|---|
| `"string"` | string primitive | ✅ |
| `"number"` | number, NaN, Infinity | ✅ |
| `"bigint"` | bigint primitive | ✅ |
| `"boolean"` | true, false | ✅ |
| `"symbol"` | symbol | ✅ |
| `"undefined"` | undefined | ✅ |
| `"object"` | object, array, **null** | ⚠️ null ham kiradi |
| `"function"` | function | ✅ |

**Muhim:** `typeof null === "object"` — JavaScript ning tarixiy bug i (JS 1.0 dan beri). TS buni biladi: `typeof === "object"` tekshiruvida null olib tashlanMAYDI. Shuning uchun `null` ni alohida tekshirish kerak:

```typescript
if (typeof x === "object" && x !== null) {
  // x: object — null xavfsiz olib tashlandi
}
```

</details>

### 4. `satisfies` nima? `:` annotation dan qanday farq qiladi?

<details>
<summary>Javob</summary>

`satisfies` (TS 4.9+) — type tekshiruvini qiladi lekin qiymatning **inferred type** ini saqlab qoladi. `:` annotation esa inferred type ni **kengaytiradi**.

```typescript
type Config = Record<string, string | number>;

// ❌ : annotation — aniq type yo'qoladi
const config1: Config = { port: 3000, host: "localhost" };
config1.port; // string | number — TS aniq type ni bilmaydi

// ✅ satisfies — inferred type saqlanadi
const config2 = { port: 3000, host: "localhost" } satisfies Config;
config2.port; // number — aniq!
config2.host; // string — aniq!
```

| | `:` Annotation | `satisfies` |
|---|---|---|
| Type tekshiruv | ✅ | ✅ |
| Inferred type | ❌ kengayadi | ✅ saqlanadi |
| Excess property | ✅ | ✅ |
| Use case | type ni kengaytirish kerak | tekshirish + aniq type saqlash |

</details>

### 5. Type guard (`is`) va assertion function (`asserts`) farqi nima?

<details>
<summary>Javob</summary>

Ikkalasi ham type ni toraytiradi, lekin mexanizmi farq qiladi:

```typescript
// Type guard — boolean qaytaradi, if ichida ishlatiladi
function isString(value: unknown): value is string {
  return typeof value === "string";
}

if (isString(input)) {
  input.toUpperCase(); // ✅ true branch — string
}

// Assertion function — void, throw qiladi yoki davom etadi
function assertString(value: unknown): asserts value is string {
  if (typeof value !== "string") throw new TypeError("Not a string");
}

assertString(input);
input.toUpperCase(); // ✅ Keyingi kodda — string
```

| | Type Guard (`is`) | Assertion (`asserts`) |
|---|---|---|
| Return type | `boolean` | `void` |
| Ishlatilish | `if (isX(val))` | `assertX(val);` (mustaqil) |
| false holat | else branch davom etadi | throw — kod to'xtaydi |
| Use case | branching | preconditions, validation |

Type guard — "tekshir va davom et". Assertion — "tekshir, xato bo'lsa to'xtat".

</details>

### 6. Closure ichida narrowing nima uchun yo'qoladi?

<details>
<summary>Javob</summary>

`let` bilan e'lon qilingan o'zgaruvchi narrowing qilinganidan keyin closure (callback) ichida ishlatilsa — TS narrowing ni yo'qotishi mumkin. Sabab: TS callback ning **qachon** chaqirilishini bilmaydi.

```typescript
function example(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    setTimeout(() => {
      // TS 5.3-: value: string | number — narrowing yo'qoldi
      // TS 5.4+: value: string — agar reassign bo'lmasa saqlaydi
    }, 100);

    value = 42; // ← Shuning uchun TS haqligi — value o'zgardi
  }
}
```

Yechimlar:
1. `const captured = value;` — const ga capture qilish
2. Function parameter sifatida ishlatish
3. TS 5.4+ — agar `let` qayta assign qilinmasa, closure da ham narrowing saqlanadi

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Control flow narrowing — har nuqtada type (Daraja: Junior+)

**Savol:** Har bir `console.log` nuqtasida `value` ning type ini ayting:

```typescript
function mystery(value: string | number | boolean | null | undefined): void {
  console.log(value);              // A
  if (value == null) return;
  console.log(value);              // B
  if (typeof value === "boolean") {
    console.log(value);            // C
    return;
  }
  console.log(value);              // D
  if (typeof value !== "number") {
    console.log(value);            // E
  }
}
```

<details>
<summary>Yechim</summary>

```
A: string | number | boolean | null | undefined  — hali narrowing yo'q
B: string | number | boolean  — null va undefined olib tashlandi (== null ikkalasini oladi)
C: boolean  — typeof "boolean" toraytirdi
D: string | number  — boolean olib tashlandi (return tufayli)
E: string  — number olib tashlandi (typeof !== "number")
```

**Kalit tushunchalar:**

- `== null` (loose equality) — `null` va `undefined` ni **bir vaqtda** olib tashlaydi (`=== null` faqat null ni oladi)
- `return` dan keyin shu branch dagi type eliminatsiya qilinadi
- `typeof !== "number"` — **negation** ham narrowing qiladi (number olib tashlanadi)

</details>

### 2. `typeof null` — xavfli pattern (Daraja: Middle)

**Savol:** Bu kod ishlaydi, lekin xavfli pattern bor. Muammoni tushuntiring va xavfsiz variantini yozing:

```typescript
function getLength(value: string | null): number {
  if (typeof value === "object") {
    return 0; // null holat
  }
  return value.length;
}

// Agar kelajakda type o'zgartilsa:
function getLength2(value: string | object | null): number {
  if (typeof value === "object") {
    return 0; // null holat... ?
  }
  return value.length;
}
```

<details>
<summary>Yechim</summary>

**Muammo:** `typeof null === "object"` — JS tarixiy bug. Birinchi funktsiyada `value: string | null`, shuning uchun `typeof === "object"` faqat `null` ni oladi — ishlaydi. Lekin:

- Agar union ga `object` qo'shilsa (ikkinchi variant) — `typeof === "object"` ham `null` ni, ham `object` ni oladi. `null` uchun `return 0` noto'g'ri bo'lishi mumkin.

```typescript
// ✅ Xavfsiz — aniq null tekshiruv
function getLength(value: string | null): number {
  if (value === null) {
    return 0;
  }
  return value.length;
}

// ✅ Qisqa variant
function getLength2(value: string | null): number {
  return value?.length ?? 0;
}

// ✅ Kengaytirilgan union uchun ham xavfsiz
function getLength3(value: string | object | null): number {
  if (value === null) return 0;
  if (typeof value === "string") return value.length;
  return JSON.stringify(value).length; // object
}
```

**Qoida:** `typeof === "object"` ga null uchun tayanmang — `=== null` aniq va xavfsiz.

</details>

### 3. `satisfies` amaliy — type-safe routing (Daraja: Middle+)

**Savol:** `satisfies` ishlatib, route configuration yozing. Har bir route ning `handler` return type ini TS aniq bilishi kerak:

```typescript
type RouteConfig = Record<string, {
  method: "GET" | "POST" | "PUT" | "DELETE";
  handler: (...args: any[]) => unknown;
}>;

// routes object ni yarating:
// - /users (GET) → string[] qaytaradi
// - /users (POST) → { id: number } qaytaradi
// - /health (GET) → { status: string } qaytaradi
// TS har bir handler ning return type ini aniq bilishi kerak
```

<details>
<summary>Yechim</summary>

```typescript
type RouteConfig = Record<string, {
  method: "GET" | "POST" | "PUT" | "DELETE";
  handler: (...args: any[]) => unknown;
}>;

// ✅ satisfies bilan — type tekshiriladi + inferred type saqlanadi
const routes = {
  listUsers: {
    method: "GET" as const,
    handler: () => ["Ali", "Vali", "Soli"],
  },
  createUser: {
    method: "POST" as const,
    handler: (name: string) => ({ id: Math.random() }),
  },
  health: {
    method: "GET" as const,
    handler: () => ({ status: "ok" }),
  },
} satisfies RouteConfig;

// TS har bir handler ning return type ini aniq biladi:
const users = routes.listUsers.handler();   // string[]
const newUser = routes.createUser.handler("Ali"); // { id: number }
const health = routes.health.handler();     // { status: string }

// ❌ : annotation bilan — aniq type yo'qoladi:
const routesBad: RouteConfig = { /* ... */ };
routesBad.listUsers.handler(); // unknown — TS bilmaydi
```

**Tushuntirish:**

- `satisfies RouteConfig` — har bir route `RouteConfig` ga mos ekanini tekshiradi
- Lekin `handler` return type ni **kengaytirmaydi** — `string[]`, `{ id: number }` aniq qoladi
- `:` annotation bilan `handler` return type `unknown` ga kengayadi — foydasiz
- `as const` method property uchun kerak — aks holda `"GET"` → `string` ga widen bo'ladi

</details>

### 4. Assertion function yozing (Daraja: Middle+)

**Savol:** API response ni validate qiladigan assertion function yozing. Noto'g'ri format da `Error` throw qilsin:

```typescript
interface ApiResponse {
  status: number;
  data: { users: { id: number; name: string }[] };
}

function assertValidResponse(value: unknown): asserts value is ApiResponse {
  // Implement qiling
}

// Ishlatish:
const raw: unknown = await fetch("/api").then(r => r.json());
assertValidResponse(raw);
// Bu nuqtadan keyin raw: ApiResponse — TS biladi
console.log(raw.data.users[0].name); // ✅ type-safe
```

<details>
<summary>Yechim</summary>

```typescript
function assertValidResponse(value: unknown): asserts value is ApiResponse {
  if (typeof value !== "object" || value === null) {
    throw new Error("Response must be an object");
  }

  const obj = value as Record<string, unknown>;

  if (typeof obj.status !== "number") {
    throw new Error("Response.status must be a number");
  }

  if (typeof obj.data !== "object" || obj.data === null) {
    throw new Error("Response.data must be an object");
  }

  const data = obj.data as Record<string, unknown>;

  if (!Array.isArray(data.users)) {
    throw new Error("Response.data.users must be an array");
  }

  for (const user of data.users) {
    if (
      typeof user !== "object" || user === null ||
      typeof (user as Record<string, unknown>).id !== "number" ||
      typeof (user as Record<string, unknown>).name !== "string"
    ) {
      throw new Error("Invalid user format");
    }
  }
}
```

**Tushuntirish:**

- `asserts value is ApiResponse` — throw bo'lmasa, keyingi kodda `value` `ApiResponse` deb hisoblanadi
- Type guard dan farqi — `if` kerak emas, throw bo'lmasa narrowing ishlaydi
- Har bir daraja alohida tekshiriladi — xato xabarlari aniq
- Real loyihalarda `zod` kabi library shu ishni kamroq boilerplate bilan qiladi

</details>

### 5. Narrowing cheklovlari — 3 muammo va yechim (Daraja: Senior)

**Savol:** Quyidagi 3 ta kodda narrowing kutilganday ishlamaydi. Har birining sababini tushuntiring va tuzating:

```typescript
// Muammo 1: typeof null
function process1(val: object | null): string {
  if (typeof val === "object") {
    return val.toString(); // Xavfsiz mi?
  }
  return "not object";
}

// Muammo 2: Closure
function process2(): void {
  let x: string | number = "hello";
  if (typeof x === "string") {
    setTimeout(() => {
      console.log(x.toUpperCase()); // Xavfsiz mi?
    }, 100);
    x = 42;
  }
}

// Muammo 3: Object property + function call
function process3(obj: { name: string | null }): void {
  if (obj.name !== null) {
    externalMutate(obj);
    console.log(obj.name.toUpperCase()); // Xavfsiz mi?
  }
}
declare function externalMutate(o: any): void;
```

<details>
<summary>Yechim</summary>

**Muammo 1:** `typeof null === "object"` — `val: object | null` da `typeof === "object"` ham `null` ni oladi. `val.toString()` runtime crash.

```typescript
// ✅ Tuzatish
function process1(val: object | null): string {
  if (val !== null && typeof val === "object") {
    return val.toString(); // ✅ null olib tashlandi
  }
  return "not object";
}
```

**Muammo 2:** Closure ichida `x` narrowing yo'qoladi (TS 5.3 gacha). Callback qachon chaqirilishi noma'lum — `x = 42` dan keyin `x.toUpperCase()` crash.

```typescript
// ✅ Tuzatish — const ga capture
function process2(): void {
  let x: string | number = "hello";
  if (typeof x === "string") {
    const captured = x; // captured: string — o'zgarmaydi
    setTimeout(() => {
      console.log(captured.toUpperCase()); // ✅
    }, 100);
    x = 42;
  }
}
```

**Muammo 3:** Object property narrowing — external funksiya chaqiruvidan keyin property o'zgargan bo'lishi mumkin. TS buni to'xtatmaydi (performance trade-off).

```typescript
// ✅ Tuzatish — local o'zgaruvchiga saqlash
function process3(obj: { name: string | null }): void {
  const name = obj.name; // snapshot
  if (name !== null) {
    externalMutate(obj);
    console.log(name.toUpperCase()); // ✅ local o'zgaruvchi — xavfsiz
  }
}
```

**Umumiy qoida:** Narrowing cheklovlariga duch kelsangiz — **local `const` ga capture qilish** eng universal yechim.

</details>

---

## Xulosa

- Type narrowing — union type ni `typeof`, `instanceof`, `in`, equality bilan toraytirish
- Control flow analysis — TS kodni yuqoridan pastga tahlil qilib, har nuqtada type ni aniqlaydi
- `typeof null === "object"` — JS tarixiy bug, alohida tekshirish kerak
- `satisfies` — type tekshiruv + inferred type saqlash (`:` annotation kengaytiradi)
- Type guard (`is`) — branching uchun. Assertion (`asserts`) — precondition uchun
- Closure va external mutation — narrowing yo'qolishi mumkin, `const` ga capture yechim

[Asosiy bo'limga qaytish →](../06-type-narrowing.md)
