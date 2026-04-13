# Interview: Custom Utility Types

> DeepPartial, DeepReadonly, DeepRequired, Mutable, PathKeys, Prettify, ValueOf, recursive type optimization va type-level programming bo'yicha interview savollari.

---

## Nazariy savollar

### 1. `DeepPartial<T>` vs `Partial<T>` — nima farq?

<details>
<summary>Javob</summary>

`Partial<T>` faqat **top-level** optional qiladi. Nested object lar to'liq (required) qoladi. `DeepPartial<T>` **barcha darajada** recursive:

```typescript
interface Config {
  db: { host: string; port: number; credentials: { user: string; pass: string } };
}

// Partial — shallow
type P = Partial<Config>;
const p: P = { db: { host: "x" } }; // ❌ port, credentials kerak!

// DeepPartial — recursive
type DP = DeepPartial<Config>;
const dp: DP = { db: { credentials: { user: "admin" } } }; // ✅
```

**Real-world:** Config update, API patch request, deep merge — barchasida `DeepPartial` kerak.

</details>

### 2. Recursive type da special case lar — Function, Date, Array nima uchun alohida handle?

<details>
<summary>Javob</summary>

`Function`, `Date`, `Array` — barchasi `object` ga extends qiladi. Special case qilmasak — ularning method/property lari ham transform bo'ladi:

```typescript
// ❌ Yomon — Date method lari optional bo'lib qoladi
type Bad<T> = { [K in keyof T]?: T[K] extends object ? Bad<T[K]> : T[K] };

// ✅ Yaxshi — special case lar avval
type DeepPartial<T> = T extends Function ? T :
  T extends Date ? T :
  T extends Array<infer U> ? Array<DeepPartial<U>> :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;
```

**Qoida:** Recursive type larda **avval special case** (Function, Date, Array, Map, Set), **keyin** umumiy object case.

</details>

### 3. `Prettify<T>` nima? Nima uchun kerak?

<details>
<summary>Javob</summary>

`Prettify<T>` — intersection type ni bitta flat object ga aylantiradi. IDE da hover osonlashadi:

```typescript
type Prettify<T> = { [K in keyof T]: T[K] } & {};

type Combined = { a: string } & { b: number } & { c: boolean };
// IDE: { a: string } & { b: number } & { c: boolean } — o'qish qiyin

type Pretty = Prettify<Combined>;
// IDE: { a: string; b: number; c: boolean } — toza
```

`& {}` — TS ga "to'liq evaluate qil" degan signal. `Pick` + `Omit` natijasi, generic type parameter — intersection bo'lganda foydali.

</details>

### 4. `ValueOf<T>` nima? `keyof` dan farqi?

<details>
<summary>Javob</summary>

`ValueOf<T>` — barcha **value type** larning union i. `keyof T` — barcha **key** larning union i:

```typescript
type ValueOf<T> = T[keyof T];

interface StatusMap { ok: 200; created: 201; notFound: 404; error: 500; }

type Keys = keyof StatusMap;     // "ok" | "created" | "notFound" | "error"
type Values = ValueOf<StatusMap>; // 200 | 201 | 404 | 500
```

**Qanday ishlaydi:** `T[keyof T]` — distributive index access. `T["ok" | "created" | ...]` = `T["ok"] | T["created"] | ...` = `200 | 201 | ...`

**Real-world — enum alternative:**

```typescript
const HttpMethod = { GET: "GET", POST: "POST", PUT: "PUT" } as const;
type HttpMethod = ValueOf<typeof HttpMethod>; // "GET" | "POST" | "PUT"
```

</details>

### 5. Recursive type da tail-call optimization nima?

<details>
<summary>Javob</summary>

TS 4.5+ da recursive conditional type lar uchun tail-call optimization bor. Agar recursive chaqiruv **to'g'ridan-to'g'ri** qaytarilsa — compiler stack depth oshirmasdan evaluate qiladi:

```typescript
// ❌ Tail-call EMAS — natija spread ichida
type BadReverse<T extends any[]> = T extends [infer H, ...infer R]
  ? [...BadReverse<R>, H] // ← spread ichida — optimize bo'lmaydi
  : [];
// 100+ element da "excessively deep" error

// ✅ Tail-call — accumulator pattern
type GoodReverse<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? GoodReverse<R, [H, ...Acc]> // ← to'g'ridan-to'g'ri qaytadi
  : Acc;
// 1000+ element da ham ishlaydi
```

**Qoida:** Recursive type larda **accumulator parameter** qo'shing. Natijani accumulator ga yig'ing, oxirida qaytaring.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. `Mutable<T>` implement qiling (Daraja: Middle)

**Savol:** `Readonly<T>` ning teskari operatsiyasini yozing — barcha `readonly` modifier larni olib tashlang. Deep versiyasini ham yozing:

```typescript
interface FrozenUser {
  readonly id: number;
  readonly name: string;
  readonly profile: { readonly bio: string };
}
// Mutable<FrozenUser> → { id: number; name: string; profile: { readonly bio: string } }
// DeepMutable<FrozenUser> → { id: number; name: string; profile: { bio: string } }
```

<details>
<summary>Yechim</summary>

```typescript
// Shallow Mutable
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Deep Mutable
type DeepMutable<T> = T extends Function ? T :
  T extends ReadonlyArray<infer U> ? Array<DeepMutable<U>> :
  T extends object ? { -readonly [K in keyof T]: DeepMutable<T[K]> } :
  T;

type M = Mutable<FrozenUser>;
// { id: number; name: string; profile: { readonly bio: string } }
// ❗ nested readonly qoldi!

type DM = DeepMutable<FrozenUser>;
// { id: number; name: string; profile: { bio: string } }
// ✅ recursive — barcha readonly olib tashlandi
```

`-readonly` — mapped type modifier removal. `ReadonlyArray<infer U>` → `Array<DeepMutable<U>>` — readonly array ni mutable ga aylantirish.

</details>

### 2. `DeepReadonly<T>` — output savoli (Daraja: Middle+)

**Savol:** Qaysi satr xato beradi?

```typescript
type DeepReadonly<T> = T extends Function ? T :
  T extends Array<infer U> ? ReadonlyArray<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

interface State {
  user: { name: string; scores: number[] };
}

declare const state: DeepReadonly<State>;

state.user.name = "Ali";          // A
state.user.scores.push(100);      // B
state.user.scores[0];             // C
const n: string = state.user.name; // D
```

<details>
<summary>Yechim</summary>

```
A — ❌ Error. user.name readonly — yozish mumkin emas
B — ❌ Error. scores ReadonlyArray — push method yo'q
C — ✅ OK. ReadonlyArray dan o'qish mumkin
D — ✅ OK. readonly faqat yozishni taqiqlaydi, o'qish ishlaydi
```

`ReadonlyArray<T>` — mutating method lar (`push`, `pop`, `splice`, `sort`) olib tashlangan. O'qish method lari (`map`, `filter`, `find`) mavjud. Runtime da farq yo'q — faqat compile-time.

</details>

### 3. `DeepPartial` — xatoni toping (Daraja: Middle+)

**Savol:** Bu `DeepPartial` da muammo bor. Toping va tuzating:

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface Event {
  name: string;
  date: Date;
  handlers: (() => void)[];
}

type PartialEvent = DeepPartial<Event>;
// date va handlers qanday bo'ladi?
```

<details>
<summary>Yechim</summary>

**Muammo:** `Date` va `Array` ikkalasi `object` ga extends qiladi. `Date` ning `getTime`, `toISOString` kabi method lari optional bo'lib qoladi. `Array` ning `push`, `pop` ham optional bo'ladi.

```typescript
// ❌ date: DeepPartial<Date> — Date method lari optional!
// ❌ handlers: DeepPartial<(() => void)[]> — array method lari optional!

// ✅ Tuzatish — special case lar
type DeepPartial<T> = T extends Function ? T :
  T extends Date ? T :
  T extends Array<infer U> ? Array<DeepPartial<U>> :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;

type PartialEvent = DeepPartial<Event>;
// { name?: string; date?: Date; handlers?: (() => void)[] } ✅
```

</details>

### 4. `DeepRequired<T>` + `NonNullable` (Daraja: Senior)

**Savol:** `DeepRequired<T>` yozing. `| undefined` muammosini ham hal qiling:

```typescript
interface Config {
  db?: { host?: string; port?: number; };
  logging?: { level?: string; };
}

// DeepRequired<Config> → barcha property lar majburiy
// Tricky: `value: string | undefined` da Required | undefined ni olib tashlamaydi
```

<details>
<summary>Yechim</summary>

```typescript
// Oddiy DeepRequired — | undefined qoladi
type DeepRequired<T> = T extends Function ? T :
  T extends Array<infer U> ? Array<DeepRequired<U>> :
  T extends object ? { [K in keyof T]-?: DeepRequired<T[K]> } :
  T;

// To'liq strict versiya — NonNullable bilan
type DeepStrictRequired<T> = T extends Function ? T :
  T extends Array<infer U> ? Array<DeepStrictRequired<U>> :
  T extends object ? { [K in keyof T]-?: DeepStrictRequired<NonNullable<T[K]>> } :
  NonNullable<T>;

type StrictConfig = DeepStrictRequired<Config>;
// { db: { host: string; port: number }; logging: { level: string } }
```

**Tricky part:**

```typescript
interface Tricky {
  name?: string;           // -? olib tashlaydi + undefined ham ketdi ✅
  value: string | undefined; // -? ta'sir qilmaydi — ? yo'q edi
}
type R = DeepRequired<Tricky>;
// { name: string; value: string | undefined } — ❗ value da undefined qoldi

type SR = DeepStrictRequired<Tricky>;
// { name: string; value: string } — ✅ NonNullable ham olib tashladi
```

`-?` faqat optional marker olib tashlaydi. `NonNullable<T[K]>` union dan `null`/`undefined` ham olib tashlaydi.

</details>

### 5. `PathKeys<T>` — xatoni toping va tuzating (Daraja: Senior)

**Savol:** Bu `PathKeys` da muammo bor — Array va Date method lari path ga tushadi:

```typescript
type PathKeys<T> = {
  [K in keyof T]: T[K] extends object
    ? `${K & string}.${PathKeys<T[K]>}` | K
    : K;
}[keyof T];

interface User { name: string; scores: number[]; createdAt: Date; }
type Keys = PathKeys<User>;
// "scores.length" | "scores.push" | "createdAt.getTime" | ... 😱
```

<details>
<summary>Yechim</summary>

```typescript
type PathKeys<T, Depth extends unknown[] = []> =
  Depth["length"] extends 4 ? never :  // depth limit
  T extends object
    ? T extends Array<any> | Date | Function
      ? never  // Array, Date, Function — chuqurroq bormaslik
      : {
          [K in keyof T & string]: T[K] extends object
            ? T[K] extends Array<any> | Date | Function
              ? K  // leaf — faqat key
              : K | `${K}.${PathKeys<T[K], [...Depth, unknown]>}`
            : K;
        }[keyof T & string]
    : never;

type Keys = PathKeys<User>;
// "name" | "scores" | "createdAt" ✅ — method lar yo'q
```

**3 ta tuzatish:**
1. **Array/Date/Function** — `extends` bilan skip qilish
2. **Depth limit** — tuple counter bilan recursion cheklash
3. **`keyof T & string`** — symbol/number key larni filter

</details>

### 6. `PathKeys<T>` to'liq implement + `PathValue` (Daraja: Senior)

**Savol:** To'liq `PathKeys<T>` va `PathValue<T, P>` yozing va type-safe `get` funksiya qiling:

```typescript
interface Config {
  db: { host: string; port: number; };
  cache: { enabled: boolean; ttl: number; };
}
// PathKeys<Config> → "db" | "db.host" | "db.port" | "cache" | "cache.enabled" | "cache.ttl"
// PathValue<Config, "db.host"> → string
// get(config, "db.host") → string
```

<details>
<summary>Yechim</summary>

```typescript
type PathKeys<T, Depth extends unknown[] = []> =
  Depth["length"] extends 4 ? never :
  T extends object
    ? T extends Array<any> | Date | Function ? never
    : {
        [K in keyof T & string]: T[K] extends object
          ? T[K] extends Array<any> | Date | Function
            ? K
            : K | `${K}.${PathKeys<T[K], [...Depth, unknown]>}`
          : K;
      }[keyof T & string]
    : never;

type PathValue<T, P extends string> =
  P extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T ? PathValue<T[Key], Rest> : never
    : P extends keyof T ? T[P] : never;

function get<T, P extends PathKeys<T> & string>(
  obj: T, path: P
): PathValue<T, P> {
  return path.split(".").reduce((acc: any, key) => acc[key], obj) as any;
}

const config: Config = {
  db: { host: "localhost", port: 5432 },
  cache: { enabled: true, ttl: 3600 },
};

const host = get(config, "db.host");       // string ✅
const ttl = get(config, "cache.ttl");       // number ✅
// get(config, "db.xyz");                   // ❌ invalid path
```

**`PathValue` ishlash jarayoni** (`"db.host"`):
1. `"db.host"` → `Key = "db"`, `Rest = "host"`
2. `"db" extends keyof Config` → ✅ → `PathValue<Config["db"], "host">`
3. `"host"` da `.` yo'q → `"host" extends keyof { host: string; port: number }` → `string`

</details>

---

## Xulosa

- `DeepPartial`/`DeepReadonly`/`DeepRequired` — recursive, special case lar majburiy (Function, Date, Array)
- `Mutable<T>` — `-readonly` modifier, `Readonly` ning teskari
- `Required` `-?` — optional olib tashlaydi, `| undefined` qoladi. `NonNullable` bilan birga ishlatish
- `Prettify<T>` — intersection ni flat object ga aylantiradi (IDE uchun)
- `ValueOf<T>` = `T[keyof T]` — barcha value type larning union i
- `PathKeys`/`PathValue` — depth limit majburiy, Array/Date skip qilish
- Tail-call optimization — accumulator pattern, 1000+ element ishlaydi
- Branded types — batafsil [interview/23 — Type-Safe Patterns](23-type-safe-patterns.md)

[Asosiy bo'limga qaytish →](../16-custom-utility-types.md)
