# Interview: Primitive Types va Type Annotations

> Type annotations, type inference, literal types, as const, type assertions, never/void va primitive tiplar bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Type annotation va type inference farqi nima? Qachon annotation yozish kerak?

<details>
<summary>Javob</summary>

**Type annotation** — developer tipni aniq belgilaydi: `let name: string = "Ali"`. **Type inference** — TypeScript tipni kontekstdan avtomatik aniqlaydi: `let name = "Ali"` — TS o'zi `string` deb tushunadi.

Annotation yozish **zarur** bo'lgan holatlar:

1. **Funksiya parametrlari** — TS parametr tipini inference qila olmaydi
2. **Delayed initialization** — o'zgaruvchi keyinroq assign bo'lganda (`let id: string;`)
3. **Inference noto'g'ri natija berganda** — masalan `let arr = []` → `any[]` (kerakli tip emas)

Annotation **shart emas** bo'lgan holatlar:

1. O'zgaruvchi initialize qilinganda — `let x = 5` → `number`
2. Callback parametrlari — `names.map(name => ...)` — kontekstdan aniq
3. Return type — funksiya tanasidan inference qilinadi

```typescript
// ✅ Annotation KERAK — parametrlar
function add(a: number, b: number): number {
  return a + b;
}

// ✅ Annotation SHART EMAS — inference ishlaydi
let count = 10;                // number
const names = ["Ali", "Vali"]; // string[]
```

Best practice: funksiya parametrlari va public API return type larida doim annotation yozing. Local o'zgaruvchilarda inference ga ishoning.

</details>

### 2. `never` va `void` farqi nima? Qachon qaysi birini ishlatish kerak?

<details>
<summary>Javob</summary>

**`void`** — funksiya normal tugaydi, lekin qiymat qaytarmaydi. Aslida `undefined` qaytaradi.

**`never`** — funksiya **hech qachon** normal tugamaydi. Yoki `throw` qiladi, yoki cheksiz loop da qoladi.

```typescript
// void — normal return, qiymat yo'q
function log(msg: string): void {
  console.log(msg);
}

// never — hech qachon return qilmaydi
function throwError(msg: string): never {
  throw new Error(msg);
}
```

`never` ning eng muhim use case — **exhaustive check**:

```typescript
type Shape = "circle" | "square" | "triangle";

function area(shape: Shape): number {
  switch (shape) {
    case "circle": return Math.PI * 100;
    case "square": return 100;
    case "triangle": return 50;
    default:
      const _exhaustive: never = shape;
      // Yangi shape qo'shilsa va case yozilmasa — compile error!
      return _exhaustive;
  }
}
```

**Muhim:** agar funksiya ba'zan throw, ba'zan normal return qilsa — return type `never` emas:

```typescript
function divide(a: number, b: number): number {
  if (b === 0) throw new Error("Division by zero");
  return a / b; // Return type: number (never emas)
}
```

</details>

### 3. `strictNullChecks` nima va nima uchun yoqish kerak?

<details>
<summary>Javob</summary>

`strictNullChecks` — `strict: true` ichiga kiruvchi flag. Yoqilganda `null` va `undefined` ni alohida tip sifatida ko'radi — boshqa tiplarga avtomatik assign mumkin bo'lmaydi.

```typescript
// strictNullChecks: false (xavfli)
let name: string = null;      // ✅ Hech qanday xato
name.toUpperCase();            // 💥 Runtime crash

// strictNullChecks: true (xavfsiz)
let name: string = null;      // ❌ Type 'null' is not assignable to 'string'
let name2: string | null = null; // ✅ Aniq belgilangan

function getLength(str: string | null): number {
  if (str !== null) {
    return str.length; // ✅ null tekshirilgandan keyin xavfsiz
  }
  return 0;
}
```

`null`/`undefined` bilan bog'liq runtime xatolar JavaScript da eng ko'p uchraydigan xatolar. `strictNullChecks` ularni compile-time da ushlaydi.

</details>

### 4. Type assertion (`as`) va type guard farqi nima?

<details>
<summary>Javob</summary>

**Type assertion** — developer compiler ga "men bilaman, bu shu tip" deydi. Runtime da hech narsa o'zgarmaydi (type erasure). Noto'g'ri bo'lsa — runtime crash.

**Type guard** — runtime tekshiruv orqali compiler ga tipni isbotlaydi. Xavfsiz.

```typescript
// ❌ Type assertion — xavfli
const el = document.getElementById("name") as HTMLInputElement;
console.log(el.value); // Element topilmasa → runtime crash

// ✅ Type guard — xavfsiz
const el = document.getElementById("name");
if (el instanceof HTMLInputElement) {
  console.log(el.value); // Runtime da tekshirilgan
}
```

| Xususiyat | Type Assertion (`as`) | Type Guard |
|-----------|----------------------|------------|
| Runtime tekshiruv | ❌ Yo'q | ✅ Bor |
| Xavfsizlik | ❌ Xavfli | ✅ Xavfsiz |
| Qachon ishlatish | 100% ishonchli bo'lganda | Default holat |

Double assertion (`"hello" as unknown as number`) normal assertion cheklovini chetlab o'tadi — **yanada xavfliroq**. Faqat test mock larda ishlatilsin.

</details>

### 5. Literal types nima? Qanday ishlatiladi?

<details>
<summary>Javob</summary>

Literal type — **aniq bitta qiymat** tipga aylanadi. `string` o'rniga `"hello"`, `number` o'rniga `42`.

Asosan **union** bilan birga — cheklangan qiymatlar to'plami yaratish uchun ishlatiladi:

```typescript
// String literal union — enum alternativasi
type Status = "active" | "inactive" | "pending";
let userStatus: Status = "active"; // ✅
// userStatus = "deleted"; // ❌

// Discriminated union — eng kuchli pattern
type Result =
  | { ok: true; data: string }
  | { ok: false; error: string };

function handle(result: Result) {
  if (result.ok) {
    console.log(result.data);   // TS biladi: data mavjud
  } else {
    console.log(result.error);  // TS biladi: error mavjud
  }
}
```

Literal type va widening:

- `const x = "hello"` → type: `"hello"` (literal — const qayta assign mumkin emas)
- `let x = "hello"` → type: `string` (widened — let qayta assign mumkin)
- `let x = "hello" as const` → type: `"hello"` (widening to'xtatilgan)

</details>

### 6. `as const` qanday ishlaydi? Enum bilan qanday farq qiladi?

<details>
<summary>Javob</summary>

`as const` expression ning barcha qiymatlarini **literal type** va **readonly** qiladi. Widening to'xtaydi.

```typescript
// as const bilan enum alternativasi
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

type Direction = typeof Direction[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT"
```

| Xususiyat | `as const` object | `enum` |
|-----------|-------------------|--------|
| JS da | Oddiy object (o'zgarishsiz) | IIFE ga compile |
| Bundle size | Kichik | Kattaroq |
| Tree shaking | ✅ Yaxshi | ❌ Qiyin |
| `isolatedModules` | ✅ Ishlaydi | ⚠️ `const enum` muammoli |
| Reverse mapping | ❌ Yo'q | ✅ Bor (numeric) |

```typescript
// as const — array bilan union type olish
const ROLES = ["admin", "user", "guest"] as const;
type Role = typeof ROLES[number]; // "admin" | "user" | "guest"
```

Zamonaviy TypeScript da `as const` object/array — `enum` ga qaraganda afzalroq: bundle size kichik, tree shaking yaxshi, `isolatedModules` bilan muammosiz.

</details>

### 7. Non-null assertion (`!`) nima? Nima uchun xavfli?

<details>
<summary>Javob</summary>

`!` postfix operator — TypeScript ga "bu qiymat `null`/`undefined` emas" deyish. Runtime da hech narsa o'zgarmaydi — `!` JS ga tushmaydi (type erasure). Qiymat aslida `null` bo'lsa — runtime crash.

```typescript
// ! — xavfli
const el = document.getElementById("app")!;
// el: HTMLElement (HTMLElement | null emas)
// Agar topilmasa — 💥 runtime crash

// ✅ Xavfsiz alternativa
const el = document.getElementById("app");
if (el !== null) {
  // el: HTMLElement — xavfsiz
}
```

`!` faqat **100% ishonchli** bo'lganda ishlatilsin:

```typescript
// ✅ Map dan olish — o'zimiz qo'yganimiz uchun ishonchli
const map = new Map<string, number>();
map.set("key", 42);
const value = map.get("key")!; // biz qo'ydik, 100% bor

// ❌ Xavfli — tashqi data
const user = getUser(id)!; // user null bo'lishi mumkin!
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Literal types va widening — output savoli (Daraja: Junior+)

**Savol:** Quyidagi kodda har bir o'zgaruvchining **TypeScript type** ini (runtime `typeof` emas) ayting:

```typescript
const a = "hello";
let b = "hello";
let c = "hello" as const;

const obj = { x: 10, y: "hi" };
const obj2 = { x: 10, y: "hi" } as const;

obj.x = 20;
// obj2.x = 20; // Bu qatorni uncomment qilsak nima bo'ladi?
```

<details>
<summary>Yechim</summary>

```typescript
const a = "hello";
// TS type: "hello" (literal — const qayta assign mumkin emas)

let b = "hello";
// TS type: string (widened — let qayta assign mumkin)

let c = "hello" as const;
// TS type: "hello" (literal — as const widening ni to'xtatdi)

const obj = { x: 10, y: "hi" };
// TS type: { x: number; y: string } — property lar widened

const obj2 = { x: 10, y: "hi" } as const;
// TS type: { readonly x: 10; readonly y: "hi" } — literal + readonly

obj.x = 20;    // ✅ Ishlaydi — property mutable
// obj2.x = 20; // ❌ Cannot assign to 'x' because it is a read-only property
```

**Muhim:** `as const` va `readonly` faqat **compile-time** tushunchalar. Runtime da `typeof` hammasi uchun bir xil (`string`, `number`). `obj2.x = 20` ni JS sifatida ishlatsa ishlaydi — TypeScript buni faqat compile-time da to'sadi.

</details>

### 2. Type inference quiz — har satrda type nima? (Daraja: Middle)

**Savol:** `strictNullChecks: true` va `noImplicitAny: true` rejimida har bir o'zgaruvchining type ini ayting:

```typescript
let a = [];
const b = [1, 2, 3];
const c = [1, "two", true];
const d = [1, 2, 3] as const;
const f = null;
let g = null;
```

<details>
<summary>Yechim</summary>

```typescript
let a = [];
// type: any[] — "evolving array". TS element tipini bilmaydi.
// noImplicitAny ham buni to'xtatmaydi — bu maxsus holat.
// Amalda: let a: number[] = [] deb annotation yozish kerak

const b = [1, 2, 3];
// type: number[]

const c = [1, "two", true];
// type: (string | number | boolean)[] — union array, tuple emas!
// Tuple kerak bo'lsa: const c = [1, "two", true] as const

const d = [1, 2, 3] as const;
// type: readonly [1, 2, 3] — readonly tuple, har element literal

const f = null;
// type: null

let g = null;
// type: any — "evolving any". null widening literal bo'lgani uchun
// TS dastlab any qo'yadi, keyingi assign larga qarab tipni evolve qiladi.
// g = "hello" → string, g = 42 → number
```

**Kalit tushuncha:** `let a = []` va `let g = null` — ikkalasi "evolving" pattern. TS dastlab `any`/`any[]` qo'yadi, lekin keyingi assign larga qarab tipni dinamik kengaytiradi. Bu faqat `let` + initialization bilan ishlaydi.

</details>

### 3. `fetch` kodida xatolarni toping (Daraja: Middle)

**Savol:** Bu kodda bir nechta xato bor. Hammasini toping va tuzating:

```typescript
function fetchUser(id: number) {
  const response = fetch(`/api/users/${id}`);
  const user = response.json();
  return user.name.toUpperCase();
}
```

<details>
<summary>Yechim</summary>

**Xatolar:**

1. `fetch` `Promise` qaytaradi — `await` yo'q. `response` aslida `Promise<Response>`
2. `.json()` ham `Promise` qaytaradi — `await` kerak
3. `user` tipi `any` — `.json()` `any` qaytaradi, type safety yo'q
4. `response.ok` tekshirilmagan — HTTP error handling yo'q

```typescript
interface User {
  name: string;
}

async function fetchUser(id: number): Promise<string> {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`);
  }

  const user: unknown = await response.json();

  // Runtime validation
  if (
    typeof user !== "object" || user === null ||
    !("name" in user) ||
    typeof (user as Record<string, unknown>).name !== "string"
  ) {
    throw new Error("Invalid user format");
  }

  return (user as User).name.toUpperCase();
}
```

**Tushuntirish:** `.json()` `any` qaytaradi — `unknown` deb treat qilish va runtime validation qilish xavfsizroq. Real loyihalarda `zod` kabi library ishlatish tavsiya etiladi.

</details>

### 4. `as const` bilan type-safe config yozing (Daraja: Middle+)

**Savol:** `as const` va `typeof`/`keyof` ishlatib, type-safe configuration yarating. `getConfig` funksiyasi faqat mavjud kalitlarni qabul qilsin va to'g'ri tipni qaytarsin:

```typescript
// Config object ni as const bilan yarating
// getConfig funksiyasini yozing:
// getConfig("apiUrl") → string qaytarsin
// getConfig("port") → number qaytarsin
// getConfig("debug") → boolean qaytarsin
// getConfig("invalid") → ❌ compile error

// Sizning kodingiz:
```

<details>
<summary>Yechim</summary>

```typescript
const CONFIG = {
  apiUrl: "https://api.example.com",
  port: 3000,
  debug: false,
  maxRetries: 5,
} as const;

type ConfigKey = keyof typeof CONFIG;

function getConfig<K extends ConfigKey>(key: K): typeof CONFIG[K] {
  return CONFIG[key];
}

const url = getConfig("apiUrl");     // type: "https://api.example.com"
const port = getConfig("port");       // type: 3000
const debug = getConfig("debug");     // type: false
// getConfig("invalid");              // ❌ Argument of type '"invalid"' is not assignable
```

**Tushuntirish:**

- `as const` — barcha qiymatlar literal type va readonly bo'ladi
- `keyof typeof CONFIG` — faqat mavjud kalitlar union i (`"apiUrl" | "port" | "debug" | "maxRetries"`)
- Generic `K extends ConfigKey` — TS har chaqiruvda aniq kalitni biladi
- `typeof CONFIG[K]` — return type kalitga qarab **aniq** (string, number, boolean emas — literal qiymat!)

</details>

### 5. Double assertion — test mock yozing (Daraja: Senior)

**Savol:** `UserService` interface ni test uchun mock qiling. Faqat `getUser` method kerak — qolganlarini yozmasdan TypeScript ni qondiring:

```typescript
interface UserService {
  getUser(id: string): Promise<{ name: string; email: string }>;
  updateUser(id: string, data: Partial<{ name: string; email: string }>): Promise<void>;
  deleteUser(id: string): Promise<void>;
  listUsers(page: number): Promise<{ name: string; email: string }[]>;
}

// Mock yarating — faqat getUser kerak, qolgan method larni yozmang
// const mockService: UserService = ???
```

<details>
<summary>Yechim</summary>

```typescript
// Variant 1: Double assertion (test larda eng ko'p ishlatiladigan)
const mockService = {
  getUser: async (id: string) => ({ name: "Test", email: "test@test.com" }),
} as unknown as UserService;

// Variant 2: Partial + assertion (aniqroq niyat)
const mockService = {
  getUser: async (id: string) => ({ name: "Test", email: "test@test.com" }),
} as Pick<UserService, "getUser"> as unknown as UserService;

// Variant 3: Jest/Vitest bilan
import { vi } from "vitest";
const mockService: UserService = {
  getUser: vi.fn().mockResolvedValue({ name: "Test", email: "test@test.com" }),
  updateUser: vi.fn(),
  deleteUser: vi.fn(),
  listUsers: vi.fn(),
};
```

**Tushuntirish:**

- **Double assertion** (`as unknown as UserService`) — normal assertion ruxsat bermaydi chunki object ning shape i `UserService` ga mos kelmaydi (3 ta method yo'q). `unknown` orqali bu cheklov chetlab o'tiladi
- Bu faqat **test** larda maqbul — production kodda double assertion ishlatmang
- `as` operator runtime da butunlay o'chiriladi (type erasure) — hech qanday tekshiruv yo'q
- Variant 3 (jest/vitest) eng xavfsiz — barcha method lar mock qilingan, lekin ko'proq boilerplate

</details>

---

## Xulosa

- Type inference ishonchli bo'lganda annotation yozish shart emas — parametrlar va public API bundan mustasno
- `never` — hech qachon tugamaydigan funksiya yoki exhaustive check uchun; `void` — qiymat qaytarmaydigan funksiya
- `strictNullChecks` — doim yoqish kerak, `null`/`undefined` xatolarni compile-time da ushlaydi
- Type guard > type assertion — default holat type guard, assertion faqat 100% ishonchli bo'lganda
- `as const` — literal type + readonly, zamonaviy enum alternativasi
- Non-null assertion (`!`) — xavfli, runtime da tekshirmasdan `null` ni olib tashlaydi

[Asosiy bo'limga qaytish →](../02-primitive-types.md)
