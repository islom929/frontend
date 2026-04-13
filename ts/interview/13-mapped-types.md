# Interview: Mapped Types

> Mapped type syntax, property modifiers, key remapping, homomorphic vs non-homomorphic, Record va custom utility types bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Mapped type nima? Qanday ishlaydi?

<details>
<summary>Javob</summary>

Mapped type — mavjud object type ning har bir property sini **transform** qilish. `{ [K in keyof T]: NewType }` — T ning barcha key larini iterate qiladi. Bu `Array.map()` ning type-level versiyasi.

```typescript
interface User { name: string; age: number; email: string; }

// Partial implementatsiyasi
type MyPartial<T> = { [K in keyof T]?: T[K]; };
type PartialUser = MyPartial<User>;
// { name?: string; age?: number; email?: string }

// Readonly implementatsiyasi
type MyReadonly<T> = { readonly [K in keyof T]: T[K]; };

// Barcha value ni Promise ga wrap
type Promisified<T> = { [K in keyof T]: Promise<T[K]>; };
type PromisifiedUser = Promisified<User>;
// { name: Promise<string>; age: Promise<number>; email: Promise<string> }
```

Compiler jarayoni: `keyof T` → key lar union → har bir K uchun property yaratish → natijalarni birlashtirish.

</details>

### 2. Homomorphic va non-homomorphic mapped type farqi nima?

<details>
<summary>Javob</summary>

**Homomorphic** — `keyof T` dan key oladigan mapped type. Original modifier larni (optional, readonly) **saqlab qoladi**:

```typescript
interface Config {
  readonly host: string;
  readonly port: number;
  timeout?: number;
}

// Homomorphic — modifier lar saqlanadi
type PartialConfig = Partial<Config>;
// { readonly host?: string; readonly port?: number; timeout?: number }

// Non-homomorphic — modifier lar yo'qoladi
type RecordConfig = Record<keyof Config, string>;
// { host: string; port: string; timeout: string }
// ❗ readonly VA optional yo'qoldi!
```

**Qoida:** Agar original modifier larni saqlash kerak — `Record` emas, `{ [K in keyof T]: ... }` ishlatish kerak.

</details>

### 3. Property modifier lar — `+?`, `-?`, `+readonly`, `-readonly` ni tushuntiring.

<details>
<summary>Javob</summary>

Mapped type da modifier larni **qo'shish** (`+`) yoki **olib tashlash** (`-`) mumkin:

| Modifier | Ma'nosi |
|----------|---------|
| `?` yoki `+?` | Optional qilish |
| `-?` | Required qilish |
| `readonly` yoki `+readonly` | Readonly qilish |
| `-readonly` | Mutable qilish |

```typescript
// Required — optional olib tashlash
type Required<T> = { [K in keyof T]-?: T[K]; };

// Mutable — readonly olib tashlash
type Mutable<T> = { -readonly [K in keyof T]: T[K]; };

// Strict — hammasi required VA mutable
type Strict<T> = { -readonly [K in keyof T]-?: T[K]; };
```

**`-?` va `| undefined` farqi — tricky:**

```typescript
interface Example {
  a: string;
  b?: string;           // optional = string | undefined
  c: string | undefined; // required, lekin undefined bo'lishi mumkin
}

type Req = Required<Example>;
// { a: string; b: string; c: string | undefined }
// ❗ b dan ? HAM, undefined HAM olib tashlandi
// ❗ c da ? yo'q edi → -? hech narsa qilmadi, | undefined qoladi
```

</details>

### 4. Key remapping (`as`) nima?

<details>
<summary>Javob</summary>

Key remapping (TS 4.1+) — mapped type ichida key ni qayta nomlash, filtrlash yoki transform qilish:

**1. Key qayta nomlash:**

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
interface User { name: string; age: number; }
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

**2. Key filtrlash — `never` bilan:**

```typescript
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
interface Mixed { name: string; age: number; email: string; }
type StringProps = OnlyStrings<Mixed>;
// { name: string; email: string }
```

**`string & K` nima uchun kerak?** `keyof T` = `string | number | symbol`. `Capitalize` faqat `string` oladi. `string & K` — faqat string key lar, number/symbol skip bo'ladi.

</details>

### 5. `Record<K, V>` ning implementatsiyasi va mapped type dan farqi.

<details>
<summary>Javob</summary>

```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
// keyof any = string | number | symbol
```

**Farq — Record vs mapped type:**

```typescript
interface Config {
  readonly host: string;
  port?: number;
}

// Record — non-homomorphic
type A = Record<keyof Config, boolean>;
// { host: boolean; port: boolean }
// ❗ readonly va optional yo'qoldi!

// Mapped type — homomorphic
type B = { [K in keyof Config]: boolean };
// { readonly host: boolean; port?: boolean }
// ✅ modifier lar saqlanadi
```

**Qachon nima:**
- `Record<K, V>` — yangi type yaratish, modifier saqlamaslik: `Record<"home" | "about", PageConfig>`
- `{ [K in keyof T]: ... }` — existing type transform, modifier saqlanishi kerak

</details>

### 6. `keyof` ning turli type lar bilan nuanslari.

<details>
<summary>Javob</summary>

```typescript
keyof { a: 1; b: 2 }         // "a" | "b"
keyof { [k: string]: any }   // string | number (JS da obj[0] === obj["0"])
keyof { [k: number]: any }   // number
keyof string[]                // number | "length" | "push" | ...
keyof never                   // string | number | symbol (PropertyKey)
keyof unknown                 // never (hech qanday key ga access mumkin emas)
keyof any                     // string | number | symbol
```

**Index signature nuance:**

```typescript
type WithIndex = { [key: string]: any; name: string };
type Keys = keyof WithIndex;
// string | number ❗ — "name" emas!
// Index signature bo'lganda keyof = index type

// Faqat declared key lar kerak bo'lsa:
type DeclaredKeys<T> = {
  [K in keyof T]: string extends K ? never : number extends K ? never : K;
}[keyof T];
type OnlyDeclared = DeclaredKeys<WithIndex>; // "name"
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `Pick` va `Omit` qo'lda implement qiling (Daraja: Middle)

**Savol:** Mapped type ishlatib `MyPick` va `MyOmit` yozing. `Omit` ni ikki xil usulda implement qiling:

```typescript
// MyPick<User, "name" | "email"> → { name: string; email: string }
// MyOmit<User, "email"> → { id: number; name: string; age: number }
```

<details>
<summary>Yechim</summary>

```typescript
// Pick — tanlangan key larni olish
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit — variant 1: Pick + Exclude
type MyOmit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// Omit — variant 2: key remapping (TS 4.1+)
type MyOmit2<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
```

**Farqlar:**
- `Pick` da `K extends keyof T` — faqat T da mavjud key lar
- `Omit` da `K extends keyof any` — mavjud bo'lmagan key ham qabul qilinadi (xato bermaydi)
- `MyOmit2` — `as P extends K ? never : P` bilan key filtrlash

</details>

### 2. Modifier lar — output savoli (Daraja: Middle+)

**Savol:** Har bir type ning natijasini ayting — modifier lar qanday o'zgaradi:

```typescript
interface Data {
  readonly id: number;
  name: string;
  tags?: string[];
}

type A = { [K in keyof Data]: Data[K] };
type B = { [K in keyof Data]?: Data[K] };
type C = { -readonly [K in keyof Data]: Data[K] };
type D = { [K in keyof Data]-?: Data[K] };
type E = { [K in keyof Data as `${K & string}Info`]: Data[K] };
type F = Record<keyof Data, boolean>;
```

<details>
<summary>Yechim</summary>

```typescript
type A = { readonly id: number; name: string; tags?: string[] };
// Homomorphic — modifier lar SAQLANADI

type B = { readonly id?: number; name?: string; tags?: string[] };
// +? qo'shildi, readonly SAQLANADI

type C = { id: number; name: string; tags?: string[] };
// -readonly olib tashlandi, optional SAQLANADI

type D = { readonly id: number; name: string; tags: string[] };
// -? olib tashlandi, readonly SAQLANADI
// ❗ tags endi string[] — -? optional HAM undefined HAM olib tashlaydi

type E = { readonly idInfo: number; nameInfo: string; tagsInfo?: string[] | undefined };
// Key remapping — modifier lar SAQLANADI (hali ham keyof Data dan — homomorphic)

type F = { id: boolean; name: boolean; tags: boolean };
// Record — NON-HOMOMORPHIC
// ❗ readonly VA optional yo'qoldi
```

**F eng tricky** — `Record` non-homomorphic bo'lgani uchun modifier lar yo'qoladi.

</details>

### 3. `PickByType` va `OmitByType` implement qiling (Daraja: Middle+)

**Savol:** Value type ga qarab property larni tanlash/olib tashlash:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  active: boolean;
}

// PickByType<User, string> → { name: string; email: string }
// OmitByType<User, string> → { id: number; age: number; active: boolean }
```

<details>
<summary>Yechim</summary>

```typescript
type PickByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? K : never]: T[K];
};

type OmitByType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? never : K]: T[K];
};

type StringProps = PickByType<User, string>;       // { name: string; email: string }
type NumberProps = PickByType<User, number>;         // { id: number; age: number }
type NonStringProps = OmitByType<User, string>;      // { id: number; age: number; active: boolean }
type NonPrimitive = OmitByType<User, string | number | boolean>; // {} (hammasi primitive)
```

**Mexanizm:** `as T[K] extends ValueType ? K : never` — key remapping. Value type mos kelsa → key saqlanadi. Mos kelmasa → `never` → property skip.

</details>

### 4. `Capitalize` key xato — toping va tuzating (Daraja: Middle+)

**Savol:** Bu kodda compile error bor. Xatoni toping va uchta usulda tuzating:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<K>}`]: () => T[K];
};
```

<details>
<summary>Yechim</summary>

**Xato:** `K` tipi `keyof T` = `string | number | symbol`. `Capitalize` faqat `string` qabul qiladi:

```
Type 'K' does not satisfy the constraint 'string'.
```

**3 ta tuzatish:**

```typescript
// 1. string & K — intersection bilan
type Getters1<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
// string & number = never → skip, string & symbol = never → skip

// 2. Extract bilan
type Getters2<T> = {
  [K in Extract<keyof T, string> as `get${Capitalize<K>}`]: () => T[K];
};

// 3. keyof T & string
type Getters3<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
};
```

Hammasi bir xil natija — faqat `string` key lar qoladi, `number`/`symbol` key lar `never` orqali skip bo'ladi.

</details>

### 5. Object Diff — ikki type orasidagi farq (Daraja: Senior)

**Savol:** `Diff<T, U>` va `Common<T, U>` utility type larni yozing. API versioning uchun `MigrationPlan` yarating:

```typescript
interface V1 { id: number; name: string; email: string; }
interface V2 { id: number; name: string; phone: string; avatar: string; }

// Diff<V2, V1> → { phone: string; avatar: string } (yangi field lar)
// Diff<V1, V2> → { email: string } (o'chirilgan field lar)
// Common<V1, V2> → { id: number; name: string } (umumiy)
```

<details>
<summary>Yechim</summary>

```typescript
// T da bor lekin U da yo'q property lar
type Diff<T, U> = {
  [K in Exclude<keyof T, keyof U>]: T[K];
};

// T va U da umumiy property lar
type Common<T, U> = {
  [K in Extract<keyof T, keyof U>]: T[K];
};

type Added = Diff<V2, V1>;    // { phone: string; avatar: string }
type Removed = Diff<V1, V2>;  // { email: string }
type Shared = Common<V1, V2>; // { id: number; name: string }

// API versioning
type MigrationPlan<Old, New> = {
  added: Diff<New, Old>;
  removed: Diff<Old, New>;
  shared: Common<Old, New>;
};

type Plan = MigrationPlan<V1, V2>;
// {
//   added: { phone: string; avatar: string };
//   removed: { email: string };
//   shared: { id: number; name: string };
// }
```

**Mexanizm:** `Exclude<keyof T, keyof U>` — T key laridan U key larini olib tashlaydi. Mapped type shu qolgan key lar ustida iterate qiladi.

</details>

---

## Xulosa

- Mapped type — `{ [K in keyof T]: NewType }` — object type transform
- Homomorphic — `keyof T` dan key oladi, modifier lar saqlanadi. `Record` non-homomorphic — modifier yo'qoladi
- `-?` optional olib tashlaydi (undefined ham), `-readonly` readonly olib tashlaydi
- Key remapping `as` — key qayta nomlash, `never` bilan filtrlash
- `string & K` — template literal da number/symbol key larni skip qilish
- `keyof` + index signature = `string | number` (declared key lar yo'qoladi)
- DeepReadonly/DeepPartial — batafsil [interview/16 — Custom Utility Types](16-custom-utility-types.md)

[Asosiy bo'limga qaytish →](../13-mapped-types.md)
