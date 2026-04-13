# Interview: Built-in Utility Types

> Partial, Required, Readonly, Record, Pick, Omit, Awaited, NoInfer — practical usage, gotchas, combinations va classification bo'yicha interview savollari. Implement qilish — [interview/12](12-conditional-types.md) (Exclude, Extract, ReturnType, Parameters) va [interview/13](13-mapped-types.md) (Pick, Omit, Record).

---

## Nazariy savollar

### 1. `Partial<T>` va `Required<T>` — real-world use case.

<details>
<summary>Javob</summary>

`Partial<T>` — barcha property larni optional. `Required<T>` — barcha optional ni required qiladi:

```typescript
type Partial<T>  = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
```

**Eng ko'p ishlatilish — update funksiyalar:**

```typescript
interface User { id: number; name: string; email: string; }

function updateUser(id: number, updates: Partial<User>): User {
  const current = getUserById(id);
  return { ...current, ...updates };
}
updateUser(1, { name: "Ali" }); // ✅ faqat name
```

**Muhim:** `Partial` va `Readonly` **shallow** — faqat top-level. Nested object lar o'zgarishsiz qoladi. Deep variant kerak bo'lsa — [interview/16 — Custom Utility Types](16-custom-utility-types.md).

</details>

### 2. `Omit<T, K>` da qanday gotcha bor?

<details>
<summary>Javob</summary>

`Omit` typo ni **tutmaydi** — `K extends keyof any` (loose):

```typescript
interface User { id: number; name: string; password: string; }

// Pick — strict: K extends keyof T
// type Bad = Pick<User, "pasword">; // ❌ Error — typo topildi

// Omit — loose: K extends keyof any
type Bad = Omit<User, "pasword">; // ⚠️ ✅ xato BERMAYDI!
// password qoladi! Typo tufayli olib tashlanmadi
```

**Yechim — StrictOmit:**

```typescript
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
// type Bad = StrictOmit<User, "pasword">; // ❌ Error ✅
```

Nima uchun Omit loose? Generic code da moslashuvchanlik uchun. Lekin production da `StrictOmit` xavfsizroq.

</details>

### 3. `Awaited<T>` nima? Nima uchun `ReturnType` async funksiyalar uchun yetarli emas?

<details>
<summary>Javob</summary>

`ReturnType` async funksiya uchun `Promise<T>` qaytaradi — unwrap **qilmaydi**:

```typescript
async function getUser(): Promise<User> { return {} as User; }

type R1 = ReturnType<typeof getUser>;           // Promise<User> — ❗ Promise qoldi
type R2 = Awaited<ReturnType<typeof getUser>>;  // User — ✅ unwrap
```

`Awaited<T>` — recursive Promise unwrap:

```typescript
type A = Awaited<Promise<string>>;                // string
type B = Awaited<Promise<Promise<number>>>;       // number — nested ham
type C = Awaited<boolean | Promise<string>>;      // boolean | string — union da ham
type D = Awaited<string>;                         // string — Promise emas, o'zgarishsiz
```

Implementation thenable object larni ham qo'llab-quvvatlaydi:

```typescript
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends ((value: infer V, ...args: infer _) => any) ?
      Awaited<V> :
      never :
  T;
```

</details>

### 4. `NoInfer<T>` (TS 5.4+) nima? Qachon kerak?

<details>
<summary>Javob</summary>

`NoInfer<T>` — ma'lum pozitsiyada **type inference ni to'xtatadi**:

```typescript
// ❌ Muammo — T ikkala argument dan infer
function createStore<T>(initial: T, defaultValue: T): T {
  return initial ?? defaultValue;
}
const store = createStore(42, "fallback");
// T = string | number — noto'g'ri!

// ✅ Yechim — NoInfer
function createStore<T>(initial: T, defaultValue: NoInfer<T>): T {
  return initial ?? defaultValue;
}
const store = createStore(42, "fallback");
// ❌ Error: '"fallback"' is not assignable to 'number'
// T faqat birinchi argument dan infer → number

const store2 = createStore(42, 0); // ✅ T = number
```

`NoInfer` — `intrinsic` type. Compiler inference candidate larni yig'ayotganda shu pozitsiyani **skip** qiladi. Runtime da hech ta'siri yo'q (type erasure).

</details>

### 5. Built-in utility type lar qaysi mexanizmlar asosida qurilgan?

<details>
<summary>Javob</summary>

**3 ta asosiy mexanizm:**

**1. Mapped Types (Object transformation):**

```typescript
Partial<T>  = { [P in keyof T]?: T[P] };      // homomorphic
Required<T> = { [P in keyof T]-?: T[P] };
Readonly<T> = { readonly [P in keyof T]: T[P] };
Pick<T, K>  = { [P in K]: T[P] };
Record<K, T> = { [P in K]: T };               // non-homomorphic
Omit<T, K>  = Pick<T, Exclude<keyof T, K>>;
```

**2. Conditional Types + `infer` (Type extraction):**

```typescript
Exclude<T, U>           = T extends U ? never : T;
Extract<T, U>           = T extends U ? T : never;
NonNullable<T>          = T & {};
ReturnType<T>           = T extends (...args: any) => infer R ? R : any;
Parameters<T>           = T extends (...args: infer P) => any ? P : never;
ConstructorParameters<T> = T extends abstract new (...args: infer P) => any ? P : never;
InstanceType<T>         = T extends abstract new (...args: any) => infer R ? R : any;
Awaited<T>              = /* recursive conditional + infer */;
```

**3. Intrinsic (Compiler ichida):**

```typescript
Uppercase<S>, Lowercase<S>, Capitalize<S>, Uncapitalize<S>, NoInfer<T>
```

| Kategoriya | Mexanizm | Utility types |
|-----------|---------|---------------|
| Object transform | Mapped types | Partial, Required, Readonly, Record, Pick, Omit |
| Union filter | Distributive conditional | Exclude, Extract, NonNullable |
| Function extract | Conditional + infer | ReturnType, Parameters, ConstructorParameters, InstanceType |
| Async | Recursive conditional | Awaited |
| String | Intrinsic | Uppercase, Lowercase, Capitalize, Uncapitalize |
| Inference | Intrinsic | NoInfer |

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Utility composition — output savoli (Daraja: Middle)

**Savol:** Har bir type ning natijasini ayting:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
}

type A = Pick<User, "name" | "email">;
type B = Omit<User, "id" | "active">;
type C = Partial<Pick<User, "name" | "age">>;
type D = Required<C>;
type E = Readonly<Omit<User, "active">> & { active: boolean };
```

<details>
<summary>Yechim</summary>

```typescript
type A = { name: string; email: string };

type B = { name: string; email: string; age: number };

type C = { name?: string; age?: number };

type D = { name: string; age: number };
// Required Partial ni qaytaradi — faqat Pick qilingan field lar

type E = {
  readonly id: number;
  readonly name: string;
  readonly email: string;
  readonly age: number;
  active: boolean;  // ❗ readonly EMAS — intersection da alohida berildi
};
```

**E tricky** — `Readonly<Omit<...>>` 4 ta field ni readonly qildi, `& { active: boolean }` alohida qo'shildi (readonly emas).

</details>

### 2. `typeof` gotcha — ReturnType<getUser> (Daraja: Middle)

**Savol:** Bu kodda xato bor. Toping va tuzating:

```typescript
function getUser() {
  return { id: 1, name: "Ali", email: "ali@test.com" };
}

type UserType = ReturnType<getUser>;
```

<details>
<summary>Yechim</summary>

**Xato:** `ReturnType<getUser>` — `getUser` runtime **value**, type emas. `typeof` kerak:

```typescript
// ❌ 'getUser' refers to a value, but is being used as a type here
type UserType = ReturnType<getUser>;

// ✅ typeof bilan
type UserType = ReturnType<typeof getUser>;
// { id: number; name: string; email: string }
```

**Qoida:** `ReturnType`, `Parameters`, `InstanceType`, `ConstructorParameters` — hammasi **TYPE** qabul qiladi. Runtime value dan type olish uchun `typeof` kerak:

```typescript
type R = ReturnType<typeof someFunction>;
type P = Parameters<typeof someFunction>;
type I = InstanceType<typeof SomeClass>;
type C = ConstructorParameters<typeof SomeClass>;
```

</details>

### 3. ConstructorParameters + InstanceType — factory (Daraja: Middle+)

**Savol:** Generic factory funksiya yozing — class va uning constructor argument larini qabul qilib, type-safe instance qaytarsin:

```typescript
class UserService {
  constructor(private db: Database, private logger: Logger) {}
}
class PostService {
  constructor(private db: Database) {}
}

// createService(UserService, db, logger) → UserService
// createService(PostService, db) → PostService
```

<details>
<summary>Yechim</summary>

```typescript
function createService<T extends new (...args: any) => any>(
  ServiceClass: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new ServiceClass(...args);
}

const userService = createService(UserService, db, logger);
// userService: UserService ✅

const postService = createService(PostService, db);
// postService: PostService ✅

// createService(PostService, db, logger);
// ❌ Expected 2 arguments, got 3
```

**Tushuntirish:**

- `ConstructorParameters<T>` — constructor parametr type larini tuple sifatida oladi
- `InstanceType<T>` — `new` bilan yaratilgan instance type
- `...args: ConstructorParameters<T>` — spread bilan aniq argumentlar talab qilinadi
- `abstract new (...)` qo'llab-quvvatlash abstract class larni ham ishlashini ta'minlaydi

</details>

### 4. `Required<T>` va `| undefined` — tricky (Daraja: Middle+)

**Savol:** Nima uchun `Required<Form>` `phone` dan `undefined` ni olib tashlamaydi? Qanday tuzatiladi?

```typescript
interface Form {
  name?: string;
  phone: string | undefined;
}

type RequiredForm = Required<Form>;
// phone hali ham string | undefined — nima uchun?
```

<details>
<summary>Yechim</summary>

```typescript
type RequiredForm = Required<Form>;
// { name: string; phone: string | undefined }
// ❗ phone da | undefined QOLDI!
```

**Sabab:** `Required` ning `-?` faqat **optional marker** (?) ni olib tashlaydi. `phone: string | undefined` da `?` yo'q — `undefined` union type ning qismi. `-?` union ichidagi `undefined` ni olib tashlamaydi.

```typescript
// name?: string → -? → name: string (? ham, undefined ham ketdi)
// phone: string | undefined → -? → string | undefined (? yo'q edi, hech narsa o'zgarmadi)
```

**Yechim — StrictRequired:**

```typescript
type StrictRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type StrictForm = StrictRequired<Form>;
// { name: string; phone: string } — undefined yo'q ✅
```

`NonNullable<T[K]>` — union dan `null` va `undefined` ni olib tashlaydi. `-?` bilan birgalikda — to'liq "required" natija.

</details>

### 5. Complex DTO patterns — utility combination (Daraja: Senior)

**Savol:** `User` interface dan quyidagi DTO type larni yarating:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// CreateUserDto — id, createdAt, updatedAt yo'q (server yaratadi)
// UpdateUserDto — id majburiy, qolganlar optional, server field larsiz
// PublicUser — password yo'q, readonly
// RequireFields<T, K> — faqat K field larni required qilish
// Merge<T, U> — ikki type birlashtirish, U ustun
```

<details>
<summary>Yechim</summary>

```typescript
// 1. Create DTO
type CreateUserDto = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; password: string; role: "admin" | "user" }

// 2. Update DTO
type UpdateUserDto = Pick<User, "id"> &
  Partial<Omit<User, "id" | "createdAt" | "updatedAt">>;
// { id: number; name?: string; email?: string; password?: string; role?: ... }

// 3. Public response
type PublicUser = Readonly<Omit<User, "password">>;
// { readonly id: number; readonly name: string; ... }

// 4. RequireFields — universal utility
type RequireFields<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

interface Options { debug?: boolean; timeout?: number; retries?: number; }
type WithTimeout = RequireFields<Options, "timeout">;
// { debug?: boolean; retries?: number; timeout: number }

// 5. Merge — U field lari T ni override qiladi
type Merge<T, U> = Omit<T, keyof U> & U;

type Updated = Merge<User, { email: string[]; verified: boolean }>;
// User + email string[] ga o'zgardi + verified qo'shildi
```

**Qoida:** Intermediate type alias lar ishlatish — o'qilishi oson, debug qilish oson. `Partial<Required<Readonly<Omit<...>>>>` o'rniga bosqichma-bosqich.

</details>

---

## Xulosa

- `Partial`/`Required`/`Readonly` — **shallow**, nested ga ta'sir qilmaydi
- `Omit` — `keyof any` (loose), typo tutmaydi. `StrictOmit` ishlatish xavfsizroq
- `Required` `-?` — optional marker olib tashlaydi, `| undefined` **qoladi**. `NonNullable` bilan birga ishlatish
- `Awaited<T>` — recursive Promise unwrap, `ReturnType` async uchun yetarli emas
- `NoInfer<T>` — inference ni to'xtatish, bitta argument dan infer qilish uchun
- `typeof` — utility type larga runtime value emas, type berish kerak
- 3 ta mexanizm: mapped types, conditional + infer, intrinsic
- Barcha utility type lar zero cost — compile da o'chiriladi

[Asosiy bo'limga qaytish →](../15-utility-types.md)
