# Interview: Template Literal Types

> Template literal type syntax, string manipulation types, union distribution, `infer` bilan pattern matching, route params extraction, recursive template literal va CamelCase conversion bo'yicha interview savollari.

---

## Nazariy savollar

### 1. Template literal type nima? Oddiy string type dan farqi?

<details>
<summary>Javob</summary>

Template literal type — backtick ichida type interpolation. JavaScript da runtime string yaratish uchun, TypeScript da **compile-time string type** yaratish uchun ishlatiladi:

```typescript
type Greeting = `Hello ${string}`;
const a: Greeting = "Hello World"; // ✅
const b: Greeting = "Hi World";    // ❌ — "Hello " bilan boshlanmaydi

type Port = `port:${number}`;
const p: Port = "port:3000"; // ✅
const q: Port = "port:abc";  // ❌ — "abc" number emas
```

| Xususiyat | Oddiy `string` | Template literal |
|-----------|---------------|-----------------|
| Qabul qiladi | Har qanday string | Faqat pattern ga mos |
| Compile-time check | ❌ | ✅ |
| Autocomplete | ❌ | ✅ (literal union da) |
| Runtime cost | — | Zero — type erasure |

Template literal faqat `string`, `number`, `bigint`, `boolean`, `null`, `undefined` va ularning literal type larini qabul qiladi.

</details>

### 2. String manipulation types — `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`.

<details>
<summary>Javob</summary>

4 ta built-in **intrinsic** string manipulation type — compiler ichida C++ da implement qilingan:

```typescript
type U = Uppercase<"hello">;     // "HELLO"
type L = Lowercase<"HELLO">;     // "hello"
type C = Capitalize<"hello">;    // "Hello" — faqat 1-harf
type UN = Uncapitalize<"Hello">; // "hello" — faqat 1-harf

// Union bilan — distributive
type Events = "click" | "scroll";
type Handlers = `on${Capitalize<Events>}`;
// "onClick" | "onScroll"
```

`lib.es5.d.ts` da `intrinsic` keyword bilan declare qilingan — user code da qayta yozib bo'lmaydi. Widened type berilsa o'zgarishsiz: `Uppercase<string>` → `string`.

</details>

### 3. Template literal da union distribution — kartezian ko'paytma qanday ishlaydi?

<details>
<summary>Javob</summary>

Template literal ichida union bo'lganda — **kartezian ko'paytma** hosil bo'ladi:

```typescript
type Verb = "get" | "set";
type Entity = "User" | "Post";
type Method = `${Verb}${Entity}`;
// "getUser" | "getPost" | "setUser" | "setPost" — 2 × 2 = 4 ta

// Uch pozitsiya — 3D
type Color = "red" | "blue";
type Size = "sm" | "lg";
type Variant = "primary" | "secondary";
type ClassName = `${Color}-${Size}-${Variant}`;
// 2 × 2 × 2 = 8 ta member
```

**Performance ogohlantirish:** `N₁ × N₂ × ... × Nₖ` ta member. TS 100,000 gacha ruxsat beradi:

```typescript
type Digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";
type TwoDigit = `${Digit}${Digit}`;          // 100 — OK
type ThreeDigit = `${Digit}${Digit}${Digit}`; // 1,000 — sekin
```

</details>

### 4. Template literal da `infer` qanday ishlaydi? Greedy matching nima?

<details>
<summary>Javob</summary>

`infer` bilan template literal type larni **parse** qilish mumkin:

```typescript
type Before<S extends string> =
  S extends `${infer A}.${infer B}` ? A : S;

type T1 = Before<"user.name">;  // "user"
type T2 = Before<"a.b.c.d">;   // "a" — birinchi "." gacha

type After<S extends string> =
  S extends `${infer A}.${infer B}` ? B : S;

type T3 = After<"a.b.c.d">;    // "b.c.d" — B greedy!
```

**Greedy matching:** Birinchi `infer` **minimal** match (separator gacha), oxirgi `infer` **greedy** (qolgan hammasi).

To'g'ri split uchun recursive kerak:

```typescript
type Split<S extends string, D extends string> =
  S extends `${infer F}${D}${infer R}`
    ? [F, ...Split<R, D>]
    : [S];

type Parts = Split<"a.b.c", ".">;  // ["a", "b", "c"]
```

</details>

### 5. Template literal type lar compile bo'lganda nima bo'ladi?

<details>
<summary>Javob</summary>

**Butunlay o'chiriladi** — pure type-level construct. Runtime da hech qanday iz qolmaydi:

```typescript
// TypeScript
type EventName = `on${Capitalize<"click" | "scroll">}`;
const name: `Hello ${string}` = "Hello World";
function handle(event: EventName) { console.log(event); }

// Compiled JavaScript
const name = "Hello World";
function handle(event) { console.log(event); }
// Type lar o'chirildi — zero cost abstraction
```

Bundle size ga ta'sir qilmaydi. Compile-time da kuchli type safety, runtime da overhead yo'q.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Distribution output savoli (Daraja: Middle)

**Savol:** Har bir type ning natijasini ayting:

```typescript
type Direction = "left" | "right" | "top" | "bottom";

type A = `margin${Capitalize<Direction>}`;
type B = `${Direction}-${Direction}`;
type C = `${Uppercase<"hello">}_${boolean}`;
```

<details>
<summary>Yechim</summary>

```typescript
type A = "marginLeft" | "marginRight" | "marginTop" | "marginBottom";
// Capitalize distributive: 4 ta member

type B = "left-left" | "left-right" | "left-top" | "left-bottom" |
         "right-left" | "right-right" | ... | "bottom-bottom";
// Kartezian: 4 × 4 = 16 ta member

type C = "HELLO_true" | "HELLO_false";
// Uppercase<"hello"> = "HELLO"
// boolean = true | false → "true" | "false"
// 1 × 2 = 2 ta member
```

**Muhim:** `boolean` template literal da `"true" | "false"` ga aylanadi (2 ta). `number` va `string` esa widened qoladi — aniq enum hosil qilmaydi.

</details>

### 2. Route params extraction (Daraja: Middle+)

**Savol:** `/users/:userId/posts/:postId` kabi route string dan `{ userId: string; postId: string }` type yarating:

```typescript
// ExtractRouteParams va RouteParams type larini yozing:
// RouteParams<"/users/:userId/posts/:postId"> → { userId: string; postId: string }
// RouteParams<"/about"> → {}
```

<details>
<summary>Yechim</summary>

```typescript
type ExtractRouteParams<S extends string> =
  S extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<Rest>
    : S extends `${string}:${infer Param}`
      ? Param
      : never;

type RouteParams<S extends string> = {
  [K in ExtractRouteParams<S>]: string;
};

type P1 = RouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }

type P2 = RouteParams<"/about">;
// {} — parametr yo'q (never → bo'sh mapped type)

// Type-safe handler
function get<R extends string>(
  route: R,
  handler: (params: RouteParams<R>) => void
): void { /* ... */ }

get("/users/:id/posts/:postId", (params) => {
  params.id;     // ✅ string
  params.postId; // ✅ string
  // params.name; // ❌ yo'q
});
```

**Ishlash:** Recursive — `:` dan keyingi nomni `infer`, `/` bo'lsa qolganini recursive qayta ishlash. Oxirgi segmentda `/` yo'q → ikkinchi branch.

</details>

### 3. `Replace` va `ReplaceAll` implement qiling (Daraja: Middle+)

**Savol:** Template literal + infer bilan string replacement type yozing:

```typescript
// Replace<"hello world", "world", "TS"> → "hello TS" (faqat birinchi)
// ReplaceAll<"a-b-c", "-", "_"> → "a_b_c" (barchasi)
```

<details>
<summary>Yechim</summary>

```typescript
// Replace — faqat birinchi occurrence
type Replace<S extends string, From extends string, To extends string> =
  S extends `${infer Before}${From}${infer After}`
    ? `${Before}${To}${After}`
    : S;

type R1 = Replace<"hello world", "world", "TS">;        // "hello TS"
type R2 = Replace<"home page home", "home", "main">;     // "main page home"

// ReplaceAll — barcha occurrence lar (recursive)
type ReplaceAll<S extends string, From extends string, To extends string> =
  From extends "" ? S :  // guard: bo'sh string → cheksiz recursion
  S extends `${infer Before}${From}${infer After}`
    ? ReplaceAll<`${Before}${To}${After}`, From, To>
    : S;

type RA1 = ReplaceAll<"a-b-c-d", "-", "_">;             // "a_b_c_d"
type RA2 = ReplaceAll<"user.profile.name", ".", "/">;    // "user/profile/name"
```

**Farq:** `Replace` — bitta match, recursion yo'q. `ReplaceAll` — natijani qaytadan o'ziga beradi (recursive), match topilmaguncha davom etadi. `From extends ""` guard — bo'sh string cheksiz recursion oldini oladi.

</details>

### 4. CamelCase — output savoli (Daraja: Middle+)

**Savol:** Har bir type ning natijasini ayting:

```typescript
type CamelCase<S extends string> =
  S extends `${infer F}_${infer R}`
    ? `${F}${CamelCase<Capitalize<R>>}`
    : S;

type A = CamelCase<"hello_world">;
type B = CamelCase<"get_all_users">;
type C = CamelCase<"no_underscore_at_all">;
type D = CamelCase<"already">;
```

<details>
<summary>Yechim</summary>

```typescript
type A = "helloWorld";
// "hello_world" → F="hello", R="world"
// `hello${CamelCase<"World">}` → "World" da _ yo'q → "helloWorld"

type B = "getAllUsers";
// "get_all_users" → F="get", R="all_users"
// `get${CamelCase<"All_users">}`
// "All_users" → F="All", R="users" → `All${"Users"}` → "AllUsers"
// Natija: "getAllUsers"

type C = "noUnderscoreAtAll";
// Recursive: no → Underscore → At → All

type D = "already";
// _ yo'q → o'zgarishsiz
```

**Kalit:** `Capitalize` birinchi harfni katta qiladi. Recursive — har `_` da split bo'ladi va keyingi qism capitalize bo'ladi.

</details>

### 5. Type-safe event system — template literal inference (Daraja: Senior)

**Savol:** `PropEventSource<T>` type yozing — `"nameChanged"` event bersa TS `K = "name"` deb infer qilsin va callback type `T["name"]` bo'lsin:

```typescript
// person.on("nameChanged", (val) => { val: string })
// person.on("ageChanged", (val) => { val: number })
// person.on("xyzChanged", ...) → ❌ "xyz" keyof T da yo'q
```

<details>
<summary>Yechim</summary>

```typescript
type PropEventSource<T> = {
  on<K extends string & keyof T>(
    eventName: `${K}Changed`,
    callback: (newValue: T[K]) => void
  ): void;
};

declare function makeWatchedObject<T>(obj: T): T & PropEventSource<T>;

const person = makeWatchedObject({
  name: "Ali",
  age: 25,
  active: true,
});

person.on("nameChanged", (val) => {
  // val: string — K = "name", T["name"] = string
  console.log(val.toUpperCase()); // ✅
});

person.on("ageChanged", (val) => {
  // val: number — K = "age", T["age"] = number
  console.log(val + 1); // ✅
});

// person.on("nameChange", () => {}); // ❌ pattern ga mos emas
```

**Qanday ishlaydi:**
1. `on` generic — `K extends string & keyof T`
2. `eventName: \`${K}Changed\`` — TS `"nameChanged"` dan `K = "name"` **infer** qiladi
3. `callback: (newValue: T[K]) => void` — `T["name"]` = `string` → `val: string`

Bu **contextual typing** + **template literal inference** birgalikda ishlashi.

</details>

### 6. Dotted path type — nested object access (Daraja: Senior)

**Savol:** `DottedPaths<T>` — barcha mumkin dot-notation path larni olish va `PathValue<T, P>` — path bo'yicha value type:

```typescript
interface Config {
  db: { host: string; port: number; };
  cache: { enabled: boolean; };
}

// DottedPaths<Config> → "db" | "db.host" | "db.port" | "cache" | "cache.enabled"
// PathValue<Config, "db.host"> → string
```

<details>
<summary>Yechim</summary>

```typescript
type DottedPaths<T, Prefix extends string = "", Depth extends unknown[] = []> =
  Depth["length"] extends 4 ? never :  // max 4 daraja
  T extends object
    ? {
        [K in keyof T & string]:
          | `${Prefix}${K}`
          | DottedPaths<T[K], `${Prefix}${K}.`, [...Depth, unknown]>
      }[keyof T & string]
    : never;

type PathValue<T, P extends string> =
  P extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? PathValue<T[Key], Rest>
      : never
    : P extends keyof T
      ? T[P]
      : never;

type Paths = DottedPaths<Config>;
// "db" | "db.host" | "db.port" | "cache" | "cache.enabled"

type HostType = PathValue<Config, "db.host">; // string

// Type-safe get funksiyasi
function get<T, P extends DottedPaths<T> & string>(
  obj: T, path: P
): PathValue<T, P> {
  return path.split(".").reduce((acc: any, key) => acc[key], obj) as any;
}

const config: Config = {
  db: { host: "localhost", port: 5432 },
  cache: { enabled: true },
};

const host = get(config, "db.host"); // string ✅
// get(config, "db.xyz");            // ❌ invalid path
```

**Performance:** `DottedPaths` har nesting da exponential o'sadi. Depth limit **majburiy** — aks holda compiler sekinlashadi. Real-world da `lodash.get`, `react-hook-form`, `vue-i18n` shu pattern ishlatadi.

</details>

---

## Xulosa

- Template literal type — compile-time string pattern, zero cost (type erasure)
- String manipulation: `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — intrinsic
- Union distribution — kartezian ko'paytma (`N₁ × N₂` member). 100,000 limit
- `infer` + template literal — string parsing. Birinchi infer minimal, oxirgi greedy
- Route params extraction — recursive `infer` bilan `:param` olish
- CamelCase, Replace, Split — recursive template literal patterns
- DottedPaths — depth limit majburiy, exponential o'sish

[Asosiy bo'limga qaytish →](../14-template-literal-types.md)
