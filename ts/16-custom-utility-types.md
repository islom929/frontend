# Bo'lim 16: Custom Utility Types Yaratish

> Custom utility types — TypeScript ning built-in utility type lariga qo'shimcha ravishda, loyihaga xos type transformatsiyalarni yaratish. Bular [mapped types](13-mapped-types.md), [conditional types](12-conditional-types.md), [template literal types](14-template-literal-types.md), va recursive types ning kombinatsiyasi orqali quriladi. Bu bo'limda [Bo'lim 15](15-utility-types.md) dagi built-in utility type lar yetmaydigan joylarda o'z utility type larimizni yozamiz — `DeepPartial`, `DeepReadonly`, `Brand`, `PathKeys`, va type-level programming patterns.

---

## Mundarija

- [Nima Uchun Custom Utility Types Kerak](#nima-uchun-custom-utility-types-kerak)
- [`DeepPartial<T>`](#deeppartialt)
- [`DeepReadonly<T>`](#deepreadonlyt)
- [`DeepRequired<T>`](#deeprequiredt)
- [`Mutable<T>` va `DeepMutable<T>`](#mutablet-va-deepmutablet)
- [`Nullable<T>` va `NonNullableDeep<T>`](#nullablet-va-nonnullabledeept)
- [`ValueOf<T>`](#valueoft)
- [`Flatten<T>` va `Prettify<T>`](#flattent-va-prettifyt)
- [`PathKeys<T>` va `PathValue<T, P>`](#pathkeyst-va-pathvaluet-p)
- [`Brand<T, B>` — Branded/Nominal Types](#brandt-b--brandednominal-types)
- [Type-Level Programming Patterns](#type-level-programming-patterns)
- [Performance va Recursive Type Optimization](#performance-va-recursive-type-optimization)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Nima Uchun Custom Utility Types Kerak

### Nazariya

TypeScript ning built-in utility type lari ([Bo'lim 15](15-utility-types.md)) — `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record` — barchasi **shallow** (faqat top-level property larga ta'sir qiladi). Real-world da esa ko'p hollarda **deep** (recursive) transformatsiyalar kerak:

1. **Deep operations** — `Partial<T>` faqat birinchi daraja property larni optional qiladi. Nested object lar to'liq (required) qoladi. API response lardan keladigan chuqur nested data uchun `DeepPartial` kerak.
2. **Nominal typing** — TypeScript structural typing ishlatadi. `type UserId = number` va `type PostId = number` — bir xil type. Ularni aralashtirib yuborishni oldini olish uchun `Brand<T, B>` pattern kerak.
3. **Dotted path access** — `"user.profile.address.city"` kabi string path orqali type-safe qiymat olish. Form library larda (React Hook Form), state management da (Zustand), va ORM larda ko'p ishlatiladi.
4. **Type-level computation** — type darajasida string parsing, tuple manipulation, yoki state machine kerak bo'lgan holatlar.

Custom utility type lar ikki asosiy tushuncha ustiga quriladi:
- **Recursive types** — type o'ziga reference qiladi
- **Conditional + infer** — type ichidan ma'lumot ajratib olish ([Bo'lim 12](12-conditional-types.md))

<details>
<summary><strong>Kod Misollari</strong></summary>

Built-in `Partial` ning cheklanganligi:

```typescript
interface Config {
  database: {
    host: string;
    port: number;
    credentials: { username: string; password: string };
  };
  logging: { level: "debug" | "info" | "warn" | "error"; file: string };
}

// Built-in Partial — faqat top-level
type PartialConfig = Partial<Config>;

const config: PartialConfig = {
  database: {
    // ❌ Error! host, port, credentials — barchasi kerak
    // Partial faqat database va logging ni optional qildi,
    // lekin database ichidagi field lar hali required
    host: "localhost",
  },
};

// Bizga kerak: DeepPartial — barcha darajada optional
```

</details>

---

## `DeepPartial<T>`

### Nazariya

`DeepPartial<T>` — `T` ning **barcha darajadagi** property larini recursive ravishda optional (`?`) qiladi. Built-in `Partial<T>` faqat birinchi daraja property larni optional qiladi, `DeepPartial` esa nested object lar ichiga ham kirib boradi.

Asosiy qiyinchilik — `T[K]` ning qaysi type ekanini to'g'ri aniqlash. Agar `object` deb tekshirsak — `Date`, `Map`, `Set`, `Array` ham object. Bularni alohida handle qilish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator `DeepPartial<T>` ni evaluate qilganda recursive type instantiation amalga oshadi:

```
DeepPartial<Config>:

1. { database?: DeepPartial<{host, port, credentials}>; logging?: DeepPartial<{level, file}> }
2. database → { host?: string; port?: number; credentials?: DeepPartial<{username, password}> }
3. credentials → { username?: string; password?: string }
   (string — primitive, recursion to'xtaydi)
4. logging → { level?: "debug"|"info"|"warn"|"error"; file?: string }
```

TS 4.5+ da recursive type lar uchun **tail-call optimization** qo'llaniladi — agar recursive type to'g'ridan-to'g'ri return qilinsa, kompilator stack depth ni oshirmasdan evaluate qiladi. Real data odatda 5-10 level dan chuqurroq bo'lmaydi.

Union type larga distributive ishlaydi: `DeepPartial<A | B>` = `DeepPartial<A> | DeepPartial<B>`. Bu odatda to'g'ri xatti-harakat.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Sodda versiya** — faqat plain object lar uchun:

```typescript
type SimpleDeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? SimpleDeepPartial<T[K]> : T[K];
};
// ⚠️ Muammo: Date, Map, Set, Array ham object ga extends qiladi
```

**To'liq versiya** — built-in class larni alohida handle qiladi:

```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepPartial<T> = T extends BuiltIn
  ? T
  : T extends Map<infer K, infer V>
    ? Map<DeepPartial<K>, DeepPartial<V>>
    : T extends ReadonlyMap<infer K, infer V>
      ? ReadonlyMap<DeepPartial<K>, DeepPartial<V>>
      : T extends Set<infer U>
        ? Set<DeepPartial<U>>
        : T extends ReadonlySet<infer U>
          ? ReadonlySet<DeepPartial<U>>
          : T extends Array<infer U>
            ? Array<DeepPartial<U>>
            : T extends ReadonlyArray<infer U>
              ? ReadonlyArray<DeepPartial<U>>
              : { [K in keyof T]?: DeepPartial<T[K]> };
```

**Real-world ishlatish — config update:**

```typescript
interface AppConfig {
  database: {
    host: string;
    port: number;
    credentials: { username: string; password: string };
  };
  logging: { level: "debug" | "info" | "warn" | "error"; file: string };
}

function mergeConfig(base: AppConfig, override: DeepPartial<AppConfig>): AppConfig {
  return deepMerge(base, override);
}

mergeConfig(defaultConfig, {
  database: {
    credentials: {
      password: "new-secret", // ✅ faqat password — username kerak emas
    },
  },
});
```

</details>

---

## `DeepReadonly<T>`

### Nazariya

`DeepReadonly<T>` — `T` ning barcha darajadagi property lariga recursive ravishda `readonly` modifier qo'yadi. Built-in `Readonly<T>` faqat top-level da ishlaydi — nested object lar hali mutable qoladi.

Bu type immutable data structures yaratishda muhim — state management da (Redux, Zustand) state object ni mutatsiya qilmaslik uchun ishlatiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type DeepReadonly<T> = T extends BuiltIn
  ? T
  : T extends Array<infer U>
    ? ReadonlyArray<DeepReadonly<U>>
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T;
```

Ishlatish — state management da:

```typescript
interface AppState {
  user: {
    name: string;
    preferences: { theme: "light" | "dark"; language: string };
  };
  items: { id: number; title: string }[];
}

type ImmutableState = DeepReadonly<AppState>;

const state: ImmutableState = getState();

// ❌ Compile-time error — top-level readonly
// state.user = { ... };

// ❌ Compile-time error — nested readonly (built-in Readonly bilan bu xato BERMAYDI)
// state.user.preferences.theme = "dark";

// ❌ Compile-time error — array readonly
// state.items.push({ id: 3, title: "new" });

// ✅ Faqat o'qish mumkin
console.log(state.user.preferences.theme);
const firstItem = state.items[0];
```

**`ReadonlyArray` farqi:**

```typescript
// Ikkalasi bir xil
type A = readonly number[];      // shorthand
type B = ReadonlyArray<number>;  // full form

// ReadonlyArray da mutating method lar yo'q:
// push, pop, shift, unshift, splice, sort, reverse — barchasi ❌
// map, filter, reduce, find, forEach — barchasi ✅ (yangi array qaytaradi)
```

</details>

---

## `DeepRequired<T>`

### Nazariya

`DeepRequired<T>` — `T` ning barcha darajadagi optional property laridan `?` ni olib tashlaydi. Built-in `Required<T>` faqat top-level da ishlaydi. `-?` modifier ni recursive qo'llaydi.

API validation dan keyin to'liq data bor ekanligini type system da ifodalash uchun ishlatiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type DeepRequired<T> = T extends BuiltIn
  ? T
  : T extends Array<infer U>
    ? Array<DeepRequired<U>>
    : T extends object
      ? { [K in keyof T]-?: DeepRequired<T[K]> }
      : T;
```

Ishlatish — validation dan keyin:

```typescript
interface UserInput {
  name?: string;
  email?: string;
  profile?: {
    bio?: string;
    avatar?: string;
    social?: { twitter?: string; github?: string };
  };
}

type ValidatedUser = DeepRequired<UserInput>;
// Barcha field lar — barcha darajada — required

function validateUser(input: UserInput): ValidatedUser {
  if (!input.name) throw new Error("name required");
  if (!input.email) throw new Error("email required");
  if (!input.profile?.bio) throw new Error("bio required");
  // ... qolgan tekshiruvlar

  return input as ValidatedUser;
}

const user = validateUser(rawInput);
console.log(user.profile.social.github); // ✅ type xato yo'q — barchasi required
```

</details>

---

## `Mutable<T>` va `DeepMutable<T>`

### Nazariya

`Mutable<T>` — `Readonly<T>` ning teskari operatsiyasi. `readonly` modifier ni barcha property lardan olib tashlaydi. TypeScript da built-in `Mutable` utility type yo'q — faqat `Readonly` bor.

`-readonly` modifier — mapped type da `readonly` ni olib tashlaydi. Bu `Required` dagi `-?` ga o'xshash syntax.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Shallow versiya
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// Deep versiya
type DeepMutable<T> = T extends BuiltIn
  ? T
  : T extends ReadonlyArray<infer U>
    ? Array<DeepMutable<U>>
    : T extends object
      ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
      : T;
```

Ishlatish — `as const` object ni mutable qilish:

```typescript
const frozenConfig = {
  host: "localhost",
  port: 3000,
  options: { debug: true, verbose: false },
} as const;

type WritableConfig = DeepMutable<typeof frozenConfig>;
// {
//   host: "localhost";   ← readonly yo'q
//   port: 3000;
//   options: { debug: true; verbose: false };
// }

// ⚠️ Literal type lar (true, "localhost", 3000) saqlanadi
// Mutable faqat readonly ni olib tashlaydi, literal type larni kengaytirmaydi
const mutableConfig: Mutable<typeof frozenConfig> = { ...frozenConfig };
mutableConfig.host = "0.0.0.0"; // ❌ — literal type "localhost" saqlanadi
```

</details>

---

## `Nullable<T>` va `NonNullableDeep<T>`

### Nazariya

`Nullable<T>` — sodda utility, `T` ga `null` qo'shadi. `NonNullableDeep<T>` — recursive ravishda barcha `null` va `undefined` larni olib tashlaydi (built-in `NonNullable` faqat top-level union dan olib tashlaydi).

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// Oddiy utilities
type Nullable<T> = T | null;
type Undefinable<T> = T | undefined;
type Maybe<T> = T | null | undefined;

// Deep versiya — nested null/undefined ni tozalash
type NonNullableDeep<T> = T extends BuiltIn
  ? NonNullable<T>
  : T extends Array<infer U>
    ? Array<NonNullableDeep<U>>
    : T extends object
      ? { [K in keyof T]: NonNullableDeep<NonNullable<T[K]>> }
      : NonNullable<T>;
```

API response dan null larni tozalash:

```typescript
interface ApiUser {
  id: number;
  name: string | null;
  profile: {
    bio: string | null;
    avatar: string | null;
    settings: { theme: string | null } | null;
  } | null;
}

type CleanUser = NonNullableDeep<ApiUser>;
// {
//   id: number;
//   name: string;           ← null olib tashlandi
//   profile: {
//     bio: string;
//     avatar: string;
//     settings: { theme: string };  ← recursive
//   };
// }
```

</details>

---

## `ValueOf<T>`

### Nazariya

`ValueOf<T>` — object type ning barcha **value type larining union** ini oladi. Bu `keyof T` ning teskari operatsiyasi: `keyof T` key larning union ini beradi, `ValueOf<T>` esa value larning union ini. TypeScript da built-in `ValueOf` yo'q, lekin uni index access type orqali yaratish mumkin: `T[keyof T]`.

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type ValueOf<T> = T[keyof T];
```

```typescript
interface StatusCodes {
  ok: 200;
  created: 201;
  badRequest: 400;
  notFound: 404;
  serverError: 500;
}

type StatusCode = ValueOf<StatusCodes>;
// 200 | 201 | 400 | 404 | 500

// Taqqoslash:
type StatusName = keyof StatusCodes;
// "ok" | "created" | "badRequest" | "notFound" | "serverError"

// Real-world — enum alternative:
const HttpMethod = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
  DELETE: "DELETE",
} as const;

type HttpMethod = ValueOf<typeof HttpMethod>;
// "GET" | "POST" | "PUT" | "DELETE"

function request(method: HttpMethod, url: string): void { /* ... */ }
request(HttpMethod.GET, "/api/users"); // ✅
request("PATCH", "/api/users");        // ❌ — union da yo'q
```

</details>

---

## `Flatten<T>` va `Prettify<T>`

### Nazariya

`Flatten<T>` / `Prettify<T>` — nested type ni "tekislash". Ikki xil ma'noda ishlatiladi:

1. **Array flatten** — `Array<Array<T>>` → `Array<T>`
2. **Intersection flatten** — `A & B & C` → bitta flat object type (IDE hover da yoyilib ko'rinadi)

`Prettify<T>` — community da keng tarqalgan nom. Identity mapped type + `& {}` trick — kompilatorga intersection ni to'liq evaluate qilishga signal beradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

**Array element type olish:**

```typescript
type FlattenArray<T> = T extends Array<infer U> ? U : T;

type A = FlattenArray<string[]>;     // string
type B = FlattenArray<number[][]>;   // number[] (faqat bitta daraja)

// Deep array flatten:
type DeepFlattenArray<T> = T extends Array<infer U>
  ? DeepFlattenArray<U>
  : T;

type D = DeepFlattenArray<number[][][]>; // number
```

**Intersection ni flat object ga aylantirish:**

```typescript
type Prettify<T> = {
  [K in keyof T]: T[K];
} & {};

type Combined = { a: string } & { b: number } & { c: boolean };
// Hover da: { a: string } & { b: number } & { c: boolean } — o'qish qiyin

type Flat = Prettify<Combined>;
// Hover da: { a: string; b: number; c: boolean } — toza

// Deep versiya:
type DeepPrettify<T> = T extends BuiltIn
  ? T
  : T extends object
    ? { [K in keyof T]: DeepPrettify<T[K]> } & {}
    : T;
```

</details>

---

## `PathKeys<T>` va `PathValue<T, P>`

### Nazariya

`PathKeys<T>` — nested object type ning barcha mumkin bo'lgan dotted path key larini union type sifatida qaytaradi. `PathValue<T, P>` — berilgan path bo'yicha aniq value type ni qaytaradi. Bu ikki utility birgalikda type-safe `get`/`set` funksiyalar yaratish imkonini beradi.

Bu type form library larda juda keng ishlatiladi:
- **React Hook Form** — `register("user.profile.name")` — type-safe path
- **Lodash `_.get`** — type-safe natija
- **Zustand/Redux selectors** — type-safe selector

Bu eng murakkab custom utility type lardan biri — template literal types ([Bo'lim 14](14-template-literal-types.md)) + recursive conditional types + mapped types barchasini birlashtiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`PathKeys` ni kompilator qanday evaluate qiladi:

```
PathKeys<{user: {name: string; age: number}}>:

K = "user":
  T["user"] = {name: string; age: number} — object!
  → "user" | `user.${PathKeys<{name: string; age: number}>}`
  → "user" | `user.${"name" | "age"}`
  → "user" | "user.name" | "user.age"

Natija: "user" | "user.name" | "user.age"
```

`PathValue` esa string ni template literal pattern matching bilan split qiladi — har safar birinchi `.` gacha bo'lgan key ni oladi va `T[Key]` ga tushadi. Bu linear recursion — path uzunligiga proporsional.

**Performance ogohlantirish:** Chuqur nested type lar uchun `PathKeys` juda ko'p type instantiation yaratadi. 5-6 darajadan chuqurroq bo'lsa, kompilator sekinlashadi. Depth limit qo'shish tavsiya etiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**PathKeys:**

```typescript
type PathKeys<T> = T extends object
  ? T extends Array<any>
    ? never
    : {
        [K in keyof T & string]: T[K] extends object
          ? T[K] extends Array<any>
            ? K
            : K | `${K}.${PathKeys<T[K]>}`
          : K;
      }[keyof T & string]
  : never;
```

**PathValue:**

```typescript
type PathValue<T, P extends string> = P extends `${infer First}.${infer Rest}`
  ? First extends keyof T
    ? PathValue<T[First], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;
```

**Type-safe get funksiya:**

```typescript
function get<T, P extends PathKeys<T> & string>(
  obj: T,
  path: P,
): PathValue<T, P> {
  const keys = path.split(".");
  let result: unknown = obj;
  for (const key of keys) {
    result = (result as Record<string, unknown>)[key];
  }
  return result as PathValue<T, P>;
}

interface User {
  id: number;
  name: string;
  profile: {
    bio: string;
    avatar: string;
    address: { city: string; country: string };
  };
}

const user: User = {
  id: 1,
  name: "Ali",
  profile: { bio: "Dev", avatar: "img.png", address: { city: "Toshkent", country: "UZ" } },
};

const city = get(user, "profile.address.city"); // string ✅
// get(user, "profile.phone");                  // ❌ — PathKeys da yo'q
```

</details>

---

## `Brand<T, B>` — Branded/Nominal Types

### Nazariya

TypeScript **structural typing** ishlatadi — type ning nomi emas, **shape** muhim. Bu degani `type UserId = number` va `type PostId = number` — bir xil type. **Branded types** — structural typing ni "sindirib", type larga **unique identity** beradigan pattern.

Asosiy g'oya: base type ga **hech qachon runtime da mavjud bo'lmaydigan** phantom property qo'shish:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
// Ikkalasi number, lekin bir-birining o'rniga ishlatib bo'lmaydi
```

Bu real-world da juda muhim: `UserId` va `PostId` ni aralashtirib yuborish — production bug ga olib keladi. Branded types buni **compile-time** da oldini oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Brand tag yaratishning usullari:

```typescript
// Usul 1: intersection bilan
type Brand<T, B extends string> = T & { readonly __brand: B };

// Usul 2: unique symbol bilan (type collision oldini oladi)
declare const __brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [__brand]: B };
```

Asosiy farq — **intersection** (`&`) orqali base type ga phantom property qo'shiladi. Bu property hech qachon runtime da yaratilmaydi — faqat type checker ko'radi.

**`as` assertion kerak** — branded type yaratish uchun `as Brand<...>` ishlatish shart, chunki runtime da phantom property yo'q. Shuning uchun constructor function ichida qilinadi — validation dan keyin.

**Arithmetic va string operatsiyalar** — branded `number` bilan matematik operatsiya qilganda brand yo'qoladi: `userId + 1` natijasi `number` (brand yo'q). Chunki `+` operatsiya natijasi oddiy `number`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
type Email = Brand<string, "Email">;

// Constructor functions — runtime validation + type branding
function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("UserId must be positive");
  return id as UserId;
}

function createPostId(id: number): PostId {
  if (id <= 0) throw new Error("PostId must be positive");
  return id as PostId;
}

function createEmail(value: string): Email {
  if (!value.includes("@")) throw new Error("Invalid email");
  return value as Email;
}

// Type safety:
function getUser(id: UserId): void { /* ... */ }
function getPost(id: PostId): void { /* ... */ }

const userId = createUserId(1);
const postId = createPostId(1);

getUser(userId); // ✅
getPost(postId); // ✅

getUser(postId); // ❌ — Type 'PostId' is not assignable to type 'UserId'
getPost(userId); // ❌ — Type 'UserId' is not assignable to type 'PostId'
getUser(42);     // ❌ — number → UserId to'g'ridan-to'g'ri emas

// Zod bilan branded types:
import { z } from "zod";

const UserIdSchema = z.number().positive().brand<"UserId">();
type ZodUserId = z.infer<typeof UserIdSchema>;

const id = UserIdSchema.parse(42); // ✅ runtime validated + branded
```

</details>

---

## Type-Level Programming Patterns

### Nazariya

TypeScript ning type system i **Turing-complete** — nazariy jihatdan istalgan hisob-kitobni type darajasida bajarish mumkin. Amaliy jihatdan bu quyidagi pattern larda ishlatiladi:

1. **Tuple manipulation** — tuple type larni qirqish, birlashtirish, teskari aylantirish
2. **String parsing** — template literal type lar bilan string ni parse qilish
3. **Type-level arithmetic** — recursive tuple bilan son hisoblash
4. **Type-level state machines** — compile-time da state transition larni tekshirish

Bu pattern larning barchasi recursive conditional types ga asoslangan ([Bo'lim 12](12-conditional-types.md)).

<details>
<summary><strong>Kod Misollari</strong></summary>

**Tuple Manipulation:**

```typescript
// Head — tuple ning birinchi elementi
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type H1 = Head<[string, number, boolean]>; // string

// Tail — birinchi elementdan keyingi qism
type Tail<T extends any[]> = T extends [any, ...infer R] ? R : [];
type T1 = Tail<[string, number, boolean]>; // [number, boolean]

// Last — oxirgi element
type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;
type L1 = Last<[string, number, boolean]>; // boolean

// Concat — ikki tuple ni birlashtirish
type Concat<A extends any[], B extends any[]> = [...A, ...B];
type C1 = Concat<[1, 2], [3, 4]>; // [1, 2, 3, 4]

// Reverse — tuple ni teskari aylantirish
type Reverse<T extends any[]> = T extends [infer H, ...infer R]
  ? [...Reverse<R>, H]
  : [];
type R1 = Reverse<[1, 2, 3]>; // [3, 2, 1]

// Length — tuple uzunligi
type Length<T extends any[]> = T["length"];
type Len = Length<[string, number, boolean]>; // 3
```

**String Parsing:**

```typescript
// Split — string ni separator bo'yicha tuple ga ajratish
type Split<S extends string, D extends string> = S extends `${infer Head}${D}${infer Tail}`
  ? [Head, ...Split<Tail, D>]
  : S extends "" ? [] : [S];

type S1 = Split<"a.b.c", ".">; // ["a", "b", "c"]

// Join — tuple ni string ga birlashtirish
type Join<T extends string[], D extends string> = T extends []
  ? ""
  : T extends [infer H extends string]
    ? H
    : T extends [infer H extends string, ...infer R extends string[]]
      ? `${H}${D}${Join<R, D>}`
      : never;

type J1 = Join<["a", "b", "c"], ".">; // "a.b.c"
```

**Type-Level Arithmetic:**

```typescript
// BuildTuple — berilgan uzunlikdagi tuple yaratish
type BuildTuple<N extends number, T extends any[] = []> = T["length"] extends N
  ? T
  : BuildTuple<N, [...T, unknown]>;

// Add — ikki sonni qo'shish
type Add<A extends number, B extends number> = [
  ...BuildTuple<A>,
  ...BuildTuple<B>,
]["length"] & number;

type Sum = Add<3, 4>; // 7

// Subtract — ayirish (A >= B bo'lishi kerak)
type Subtract<A extends number, B extends number> = BuildTuple<A> extends [
  ...BuildTuple<B>,
  ...infer R,
] ? R["length"] & number : never;

type Diff = Subtract<7, 3>; // 4
```

⚠️ Type-level arithmetic faqat kichik sonlar uchun ishlaydi (tail-call bo'lmasa taxminan 50, tail-call bilan taxminan 1000).

**Type-Level State Machine:**

```typescript
type State = "idle" | "loading" | "success" | "error";

type Transitions = {
  idle: "loading";
  loading: "success" | "error";
  success: "idle";
  error: "idle" | "loading";
};

type ValidTransition<From extends State, To extends State> =
  To extends Transitions[From] ? To : never;

function transition<From extends State, To extends State>(
  _from: From,
  to: ValidTransition<From, To>,
): To {
  return to;
}

transition("idle", "loading");    // ✅
transition("loading", "success"); // ✅
transition("idle", "success");    // ❌ — never
transition("success", "error");   // ❌ — never
```

</details>

---

## Performance va Recursive Type Optimization

### Nazariya

Recursive type lar kompilator uchun qimmat operatsiya. Har bir recursion level da yangi type instantiation yaratiladi. Chuqur yoki keng recursive type lar kompilatorni sekinlashtiradi va IDE IntelliSense ni yomonlashtiradi.

**Asosiy limitlar:**

| Limit | Taxminiy qiymat | Izoh |
|-------|----------------|------|
| Conditional type recursion depth | ~50 (tail-call: ~1000) | `Type instantiation is excessively deep` xatosi |
| Union type members | Cheklangan | Katta union lar sekin |

<details>
<summary><strong>Under the Hood</strong></summary>

**Tail-call optimization (TS 4.5+):** Agar recursive conditional type natijasi to'g'ridan-to'g'ri qaytarilsa (boshqa type operatsiyalar o'ralmagan bo'lsa), kompilator stack frame ni qayta ishlatadi:

```typescript
// ❌ Tail-call EMAS — recursive natija boshqa type ichida
type BadReverse<T extends any[]> = T extends [infer H, ...infer R]
  ? [...BadReverse<R>, H] // ← natija spread ichida
  : [];

// ✅ Tail-call — accumulator pattern
type GoodReverse<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? GoodReverse<R, [H, ...Acc]> // ← to'g'ridan-to'g'ri recursive chaqiruv
  : Acc;
```

**Depth limit qo'shish:**

```typescript
type DeepPartialWithDepth<T, Depth extends number[] = []> = Depth["length"] extends 5
  ? T
  : T extends object
    ? { [K in keyof T]?: DeepPartialWithDepth<T[K], [...Depth, 0]> }
    : T;
```

**Cache-friendly patterns — intermediate type alias:**

```typescript
// ❌ Har safar qayta hisoblaydi
type BadPathKeys<T> = T extends object
  ? { [K in keyof T & string]: T[K] extends object ? `${K}.${BadPathKeys<T[K]>}` | K : K }[keyof T & string]
  : never;

// ✅ Helper type bilan — kompilator cache qilishi mumkin
type PathKeysHelper<T, K extends string> = T extends object
  ? T extends Array<any> ? K : K | `${K}.${PathKeys<T>}`
  : K;

type PathKeys2<T> = T extends object
  ? { [K in keyof T & string]: PathKeysHelper<T[K], K> }[keyof T & string]
  : never;
```

**Profiling:** `tsc --generateTrace traceDir` buyrug'i bilan qaysi recursive type eng ko'p vaqt sarflayotganini aniqlash mumkin.

</details>

---

## Edge Cases va Gotchas

### 1. `DeepPartial<Date>` — Date ning method lari optional bo'lib qoladi

```typescript
// Sodda DeepPartial da:
type SimpleDeep<T> = { [K in keyof T]?: T[K] extends object ? SimpleDeep<T[K]> : T[K] };

type PartialDate = SimpleDeep<Date>;
// Date ning getTime, toJSON, valueOf kabi method lari ham optional!
// Bu mantiqiy emas — Date ni butunlay qoldirish kerak

// ✅ To'g'ri — BuiltIn check:
type DeepPartial<T> = T extends BuiltIn ? T : /* ... recursive ... */;
```

### 2. Branded type arifmetikada brand yo'qoladi

```typescript
type UserId = number & { readonly __brand: "UserId" };

const a = 1 as UserId;
const b = 2 as UserId;
const c = a + b;
// c: number — brand YO'QOLDI!
// + operatsiya natijasi oddiy number qaytaradi

// JSON serialization da ham brand yo'qoladi — deserialize da qayta validate kerak
```

### 3. `PathKeys` circular reference da cheksiz recursion

```typescript
interface TreeNode {
  value: string;
  children: TreeNode[];
}

type TreePaths = PathKeys<TreeNode>;
// ❌ Type instantiation is excessively deep and possibly infinite

// ✅ Depth limit bilan:
type BoundedPathKeys<T, Depth extends number[] = []> = Depth["length"] extends 4
  ? never
  : T extends object
    ? T extends Array<any> ? never
      : { [K in keyof T & string]: T[K] extends object
          ? K | `${K}.${BoundedPathKeys<T[K], [...Depth, 0]>}`
          : K
        }[keyof T & string]
    : never;
```

### 4. `ValueOf` ni primitive type bilan ishlatganda

```typescript
type V = ValueOf<string>;
// string ning barcha method/property type lari union!
// charAt, includes, length, ... — bu kutilmagan natija

// ✅ Constraint qo'shish:
type SafeValueOf<T extends Record<string, unknown>> = T[keyof T];
```

### 5. `Prettify<T>` faqat top-level da ishlaydi

```typescript
type A = { x: string } & { y: { z: number } & { w: boolean } };

type Pretty = Prettify<A>;
// { x: string; y: { z: number } & { w: boolean } }
// ❗ y ICHIDAGI intersection hali flat emas!

// ✅ DeepPrettify kerak:
type DeepPretty = DeepPrettify<A>;
// { x: string; y: { z: number; w: boolean } }
```

---

## Common Mistakes

### ❌ Xato 1: `Date`, `Map`, `Set` ni handle qilmaslik

```typescript
// ❌ Date ning ichki method larini ham optional qiladi
type BadDeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? BadDeepPartial<T[K]> : T[K];
};

// ✅ BuiltIn larni alohida tekshirish
type DeepPartial<T> = T extends BuiltIn
  ? T
  : T extends Array<infer U>
    ? Array<DeepPartial<U>>
    : T extends object
      ? { [K in keyof T]?: DeepPartial<T[K]> }
      : T;
```

**Nima uchun:** `Date`, `Map`, `Set`, `RegExp` — bular `object` ga extends qiladi, lekin ichki tuzilishini o'zgartirish mantiqiy emas.

### ❌ Xato 2: Branded type phantom property ni runtime da ishlatish

```typescript
type UserId = number & { __brand: "UserId" };
const id: UserId = 42 as UserId;
console.log(id.__brand); // ❌ Runtime da undefined!
```

**Nima uchun:** Phantom property faqat type-level da — runtime da mavjud emas.

### ❌ Xato 3: Recursive type da depth limit qo'ymaslik

```typescript
// ❌ Circular reference bilan cheksiz recursion
interface TreeNode { value: string; children: TreeNode[] }
type DeepKeys = PathKeys<TreeNode>; // ❌ — cheksiz

// ✅ Depth limit bilan
type SafePathKeys<T, D extends number[] = []> = D["length"] extends 4
  ? never : /* ... */;
```

### ❌ Xato 4: Accumulator pattern o'rniga non-tail-call recursion

```typescript
// ❌ Non-tail-call — ~50 depth limit
type BadReverse<T extends any[]> = T extends [infer H, ...infer R]
  ? [...BadReverse<R>, H] : [];

// ✅ Tail-call — ~1000 depth limit
type GoodReverse<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? GoodReverse<R, [H, ...Acc]> : Acc;
```

**Nima uchun:** Tail-call optimization faqat recursive chaqiruv to'g'ridan-to'g'ri natija bo'lganda ishlaydi. Spread ichida o'ralgan recursive call — tail-call emas.

### ❌ Xato 5: `Prettify/Flatten` ni recursive qilmaslik

```typescript
type Prettify<T> = { [K in keyof T]: T[K] } & {};
// Faqat top-level! Nested intersection lar flat bo'lmaydi

// ✅ Deep versiya kerak:
type DeepPrettify<T> = T extends BuiltIn
  ? T
  : T extends object
    ? { [K in keyof T]: DeepPrettify<T[K]> } & {}
    : T;
```

---

## Amaliy Mashqlar

### Mashq 1: `PickByValue<T, V>` (Oson)

**Savol:** Object type dan faqat value type i `V` ga mos keladigan property larni oladigan utility type yozing.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  isActive: boolean;
}

type StringProps = PickByValue<User, string>;
// { name: string; email: string }
```

<details>
<summary>Javob</summary>

```typescript
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};
```

**Tushuntirish:** Key remapping (`as`) bilan har bir key ni tekshiramiz — agar `T[K]` `V` ga extends qilsa, key saqlanadi, aks holda `never` (filtrlash).

</details>

---

### Mashq 2: `OmitByValue<T, V>` (Oson)

**Savol:** `PickByValue` ning teskari — value type i `V` ga mos keladigan property larni **chiqarib** tashlaydigan utility type yozing.

```typescript
type NonStringProps = OmitByValue<User, string>;
// { id: number; age: number; isActive: boolean }
```

<details>
<summary>Javob</summary>

```typescript
type OmitByValue<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};
```

**Tushuntirish:** `PickByValue` ning aksi — ternary ni teskari qilish: `T[K] extends V` bo'lganda `never`, aks holda `K`.

</details>

---

### Mashq 3: `DeepPick<T, P>` (Qiyin)

**Savol:** Dotted path bo'yicha nested property ni type-safe tanlash.

```typescript
interface User {
  id: number;
  name: string;
  profile: {
    bio: string;
    avatar: string;
    address: { city: string; country: string };
  };
}

type Result = DeepPick<User, "profile.address.city" | "name">;
// { name: string; profile: { address: { city: string } } }
```

<details>
<summary>Javob</summary>

```typescript
type DeepPick<T, P extends string> = (
  P extends `${infer First}.${infer Rest}`
    ? First extends keyof T
      ? { [K in First]: DeepPick<T[First], Rest> }
      : never
    : P extends keyof T
      ? { [K in P]: T[P] }
      : never
) extends infer O
  ? { [K in keyof O]: O[K] }
  : never;
```

**Tushuntirish:**
1. Dotted path ni `First.Rest` ga ajratish
2. `First` bo'yicha object yaratish, value sifatida recursive `DeepPick`
3. Nuqta yo'q — oddiy property ni olish
4. Union path lar uchun distributive conditional type ishlaydi

</details>

---

### Mashq 4: `Entries<T>` (O'rta)

**Savol:** `Object.entries()` qaytaradigan type ni ifodalovchi utility type yozing.

```typescript
interface Colors {
  red: "#ff0000";
  green: "#00ff00";
  blue: "#0000ff";
}

type ColorEntries = Entries<Colors>;
// ["red", "#ff0000"] | ["green", "#00ff00"] | ["blue", "#0000ff"]
```

<details>
<summary>Javob</summary>

```typescript
type Entries<T> = {
  [K in keyof T]: [K, T[K]];
}[keyof T];
```

**Tushuntirish:** Mapped type har bir key-value juftini `[K, T[K]]` tuple ga aylantiradi. `[keyof T]` — value union ga aylantiradi. Bu `ValueOf` pattern ining kengaytirilgan shakli.

</details>

---

### Mashq 5: `ExactKeys<T, K>` (O'rta)

**Savol:** Berilgan key lar **aynan** `T` ning key lariga mos kelishini tekshiradigan type yozing.

```typescript
interface User { id: number; name: string; email: string }

type Check1 = ExactKeys<User, "id" | "name" | "email">; // ✅ User
type Check2 = ExactKeys<User, "id" | "name">;            // ❌ never
type Check3 = ExactKeys<User, "id" | "name" | "email" | "age">; // ❌ never
```

<details>
<summary>Javob</summary>

```typescript
type ExactKeys<T, K extends string> = [keyof T] extends [K]
  ? [K] extends [keyof T]
    ? T
    : never
  : never;
```

**Tushuntirish:** Ikki yo'nalishli tekshiruv: `keyof T` `K` ga extends qiladi **VA** `K` `keyof T` ga extends qiladi. `[T] extends [K]` — non-distributive check, union distribution ni oldini oladi.

</details>

---

## Xulosa

Bu bo'limda TypeScript ning type system ining to'liq kuchini ishlatib, o'z utility type larimizni yaratdik:

**Deep recursive types:**
- **`DeepPartial<T>`** — barcha darajada optional. Gotcha: `Date`, `Map`, `Set` ni alohida handle qilish kerak.
- **`DeepReadonly<T>`** — barcha darajada readonly. Array lar `ReadonlyArray` ga aylanadi.
- **`DeepRequired<T>`** — barcha darajada required (`-?` modifier).
- **`Mutable<T>`** / **`DeepMutable<T>`** — `Readonly` ning teskari (`-readonly` modifier).
- **`NonNullableDeep<T>`** — recursive `null`/`undefined` olib tashlash.

**Object va value utilities:**
- **`ValueOf<T>`** — `T[keyof T]` — barcha value type lar union i.
- **`Flatten<T>`** / **`Prettify<T>`** — intersection ni flat object ga aylantirish (`& {}` trick).

**Path-based types:**
- **`PathKeys<T>`** — dotted path key lar union i. Template literal + recursive + mapped types.
- **`PathValue<T, P>`** — path bo'yicha aniq value type.

**Nominal typing:**
- **`Brand<T, B>`** — structural typing ni "sindirib" unique identity beradi. Phantom property + constructor function pattern.

**Type-level programming:**
- Tuple manipulation: `Head`, `Tail`, `Last`, `Reverse`, `Concat`
- String parsing: `Split`, `Join`, `Replace`, `ReplaceAll`
- Type-level arithmetic: `Add`, `Subtract` (recursive tuple bilan)
- State machines: compile-time transition validation

**Performance:** Tail-call optimization (accumulator pattern), depth limit, cache-friendly intermediate type alias lar.

**Bog'liq bo'limlar:**
- [Bo'lim 12: Conditional Types](12-conditional-types.md) — `infer`, recursive conditional types
- [Bo'lim 13: Mapped Types](13-mapped-types.md) — modifier lar (`-?`, `-readonly`), key remapping
- [Bo'lim 14: Template Literal Types](14-template-literal-types.md) — dotted path, string parsing
- [Bo'lim 15: Utility Types](15-utility-types.md) — built-in utility type lar (shallow versiyalar)
- [Bo'lim 25: Type Compatibility](25-type-compatibility.md) — structural vs nominal typing

---

**Keyingi bo'lim:** [17-modules.md](17-modules.md) — TypeScript da module system: `import type` va type-only imports, `verbatimModuleSyntax`, module resolution strategiyalari, path aliases, barrel exports, ambient modules, module augmentation, va dynamic imports.
