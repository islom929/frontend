# Interview: Type-Safe Patterns

> Branded/opaque types, runtime validation (Zod), exhaustive object maps, `as const satisfies` va validation library tanlov bo'yicha interview savollari. Builder — [interview/21](21-design-patterns.md), Result — [interview/21](21-design-patterns.md), exhaustive switch — [interview/05](05-unions-intersections.md).

---

## Nazariy savollar

### 1. Branded (opaque) type nima va nima uchun kerak?

<details>
<summary>Javob</summary>

TypeScript structural typing — shape bir xil bo'lsa, type lar mos. Bu xavfli: `UserId` va `PostId` ikkalasi `number`. Branded type — phantom property bilan **nominal** typing beradi:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;

function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("UserId must be positive");
  return id as UserId; // constructor funksiya ichida brand "beriladi"
}

function getPost(id: PostId): void { /* ... */ }

const userId = createUserId(1);
const postId = createPostId(1);

getPost(postId); // ✅
// getPost(userId); // ❌ UserId ≠ PostId
// getPost(42);     // ❌ number ≠ PostId
```

**`unique symbol` bilan to'liq nominal:**

```typescript
declare const UserIdTag: unique symbol;
type UserId = number & { readonly [UserIdTag]: void };
```

**Key points:** `__brand` runtime da **yo'q** (phantom). Brand yaratish faqat constructor function ichida `as` bilan. Arithmetic brand ni yo'qotadi: `userId + 1` → `number`.

</details>

### 2. Zod bilan runtime validation — `z.infer` qanday ishlaydi?

<details>
<summary>Javob</summary>

TS type lari compile-time da o'chiriladi. API response, `JSON.parse` — runtime da untyped. Zod **schema** dan validate + type chiqaradi — single source of truth:

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(2),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

type User = z.infer<typeof UserSchema>;
// { id: number; name: string; email: string; role: "admin" | "user" }

// Runtime validation
const user = UserSchema.parse(data);       // throws on invalid
const result = UserSchema.safeParse(data); // { success, data/error }
```

Zod `.brand()` bilan branded type:

```typescript
const EmailSchema = z.string().email().brand<"Email">();
type Email = z.infer<typeof EmailSchema>; // string & { __brand: "Email" }
```

</details>

### 3. Exhaustive object map — `Record` bilan barcha case majburiy.

<details>
<summary>Javob</summary>

`switch` ga alternativa — `Record` bilan exhaustive mapping:

```typescript
type StatusCode = 200 | 201 | 400 | 401 | 404 | 500;

const statusMessages: Record<StatusCode, string> = {
  200: "OK",
  201: "Created",
  400: "Bad Request",
  401: "Unauthorized",
  404: "Not Found",
  500: "Internal Server Error",
  // Biror code qoldirilsa — TS Error!
};

function getMessage(code: StatusCode): string {
  return statusMessages[code]; // type-safe, undefined mumkin emas
}
```

Yangi `StatusCode` qo'shilganda `statusMessages` da ham qo'shish **majburiy** — aks holda compile error. `switch` dan ixcham va xavfsiz.

</details>

### 4. `as const satisfies` — ikkalasi birgalikda nima beradi?

<details>
<summary>Javob</summary>

`as const` — literal type + readonly. `satisfies` — type validation. Birgalikda: **validate + literal saqla**:

```typescript
type Theme = Record<string, string | number>;

// ❌ Type annotation — literal yo'qoladi
const t1: Theme = { primary: "#007bff", size: 16 };
t1.primary; // string | number — aniq emas

// ✅ as const satisfies — validate + literal
const t2 = { primary: "#007bff", size: 16 } as const satisfies Theme;
t2.primary; // "#007bff" — literal!
t2.size;    // 16 — literal!
// Noto'g'ri qiymat compile error beradi
```

`as const` → narrow type. `satisfies` → constraint tekshiruv. Birgalikda eng kuchli pattern.

</details>

### 5. Zod vs Valibot — qachon qaysi biri?

<details>
<summary>Javob</summary>

| Xususiyat | Zod | Valibot |
|-----------|-----|---------|
| API | Method chaining | Pipe / functional |
| Bundle | ~13 KB | ~1-2 KB (tree-shakeable) |
| Tree-shaking | Cheklangan | Ajoyib |
| Ecosystem | Eng katta (tRPC, RHF) | O'sib bormoqda |
| `.brand()` | ✅ | ❌ |

```typescript
// Zod
const User = z.object({ name: z.string().min(2) });

// Valibot — tree-shakeable
const User = v.object({ name: v.pipe(v.string(), v.minLength(2)) });
```

**Zod** — aksariyat holatlarda, katta ecosystem. **Valibot** — bundle size muhim (frontend). **ArkType** — TS type syntax ga yaqin, type-first.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Branded types — output savol (Daraja: Middle)

**Savol:** Output va `typeof` ni ayting:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type USD = Brand<number, "USD">;

function usd(amount: number): USD { return amount as USD; }

function addUSD(a: USD, b: USD): USD {
  return usd((a as number) + (b as number));
}

const total = addUSD(usd(29.99), usd(2.40));
console.log(total);
console.log(typeof total);
```

<details>
<summary>Yechim</summary>

```
32.39
number
```

Branded type faqat **compile-time**. Runtime da `USD` oddiy `number`. `__brand` property **mavjud emas** (phantom). `as USD` assertion runtime da hech narsa o'zgartirmaydi.

</details>

### 2. Reducer xato — exhaustive (Daraja: Middle)

**Savol:** Xatoni toping va `assertNever` bilan tuzating:

```typescript
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
  }
}
```

<details>
<summary>Yechim</summary>

**Xato:** `"RESET"` case qoldirilgan + return type `number` lekin `undefined` qaytishi mumkin.

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected: ${JSON.stringify(value)}`);
}

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
    case "RESET":     return 0;
    default:          return assertNever(action);
  }
}
```

`assertNever` — yangi action qo'shilsa compile error ogohlantiradi.

</details>

### 3. DeepValues — output savol (Daraja: Middle+)

**Savol:** `RoutePath` type nima? `"/about"` compile bo'ladimi?

```typescript
const ROUTES = {
  home: "/",
  users: { list: "/users", detail: "/users/:id" },
} as const;

type DeepValues<T> = T extends object
  ? { [K in keyof T]: DeepValues<T[K]> }[keyof T]
  : T;

type RoutePath = DeepValues<typeof ROUTES>;

const x: RoutePath = "/users/:id"; // ?
const y: RoutePath = "/about";     // ?
```

<details>
<summary>Yechim</summary>

```typescript
type RoutePath = "/" | "/users" | "/users/:id";
```

- `x = "/users/:id"` → ✅ union da bor
- `y = "/about"` → ❌ compile error — `"/about"` union da yo'q

`DeepValues` recursive — nested object ning barcha **leaf** qiymatlarini literal union sifatida yig'adi. `as const` bo'lgani uchun literal type saqlanadi.

</details>

### 4. Zod xato — validation yo'q (Daraja: Middle+)

**Savol:** Bu kodda xato bor — Zod schema bor lekin ishlatilmayapti:

```typescript
import { z } from "zod";

const ProductSchema = z.object({
  name: z.string(),
  price: z.number().positive(),
});
type Product = z.infer<typeof ProductSchema>;

async function fetchProduct(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  const data = await response.json();
  return data; // ← Xavfsiz mi?
}
```

<details>
<summary>Yechim</summary>

**Xato:** `data` — `any`, **runtime validation yo'q**. Schema bor lekin ishlatilmayapti:

```typescript
async function fetchProduct(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  const data: unknown = await response.json();
  return ProductSchema.parse(data); // ✅ runtime validation
}

// Yoki safeParse:
async function fetchProduct(id: number): Promise<Product | null> {
  const data: unknown = await (await fetch(`/api/products/${id}`)).json();
  const result = ProductSchema.safeParse(data);
  return result.success ? result.data : null;
}
```

</details>

### 5. Brand\<T,B\> + unique symbol implement (Daraja: Senior)

**Savol:** `Brand<T, B>` ning ikki variantini yozing — `__brand` string va `unique symbol` bilan. `Email` branded type yarating va validate qiling:

```typescript
// createEmail("invalid") → throw
// createEmail("ali@test.com") → Email branded type
// sendEmail(email: Email) — faqat validated email qabul qilsin
```

<details>
<summary>Yechim</summary>

```typescript
// Variant 1: string brand
type Brand<T, B extends string> = T & { readonly __brand: B };
type Email = Brand<string, "Email">;

function createEmail(value: string): Email {
  if (!value.includes("@")) throw new Error("Invalid email");
  return value as Email;
}

// Variant 2: unique symbol — to'liq nominal
declare const EmailTag: unique symbol;
type Email2 = string & { readonly [EmailTag]: void };

function createEmail2(value: string): Email2 {
  if (!value.includes("@")) throw new Error("Invalid email");
  return value as Email2;
}

// Ishlatish
function sendEmail(to: Email): void {
  console.log(`Sending to ${to}`);
}

const email = createEmail("ali@test.com"); // ✅
sendEmail(email);     // ✅
// sendEmail("raw");  // ❌ string ≠ Email
// sendEmail(createUserId(1) as any as Email); // ❌ runtime da ham xato (agar validate bo'lsa)
```

**unique symbol farqi:** `__brand: "Email"` va `__brand: "Phone"` — agar developer xato qilib bir xil string yozsa, TS tutmaydi. `unique symbol` har declaration uchun **unique** — hech qachon to'qnashmaydi.

**Zod bilan integration:**

```typescript
const EmailSchema = z.string().email().brand<"Email">();
type Email = z.infer<typeof EmailSchema>;
// Compile-time brand + runtime validation = to'liq safety
```

</details>

---

## Xulosa

- Branded types — phantom property bilan nominal typing. `unique symbol` to'liq nominal
- Zod — runtime validation + `z.infer` type chiqarish. Single source of truth
- `as const satisfies` — validate + literal type saqlash
- Exhaustive object map — `Record` bilan barcha case majburiy
- Brand yaratish faqat constructor function da (`as` assertion)
- Zod vs Valibot — ecosystem vs bundle size
- Result/Either — batafsil [interview/21](21-design-patterns.md)
- Exhaustive switch — batafsil [interview/05](05-unions-intersections.md)

[Asosiy bo'limga qaytish →](../23-type-safe-patterns.md)
